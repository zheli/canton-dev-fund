## PQS Relational Backend and Query API — Technical Review

| Field | Value |
| :---- | :---- |
| Author | Srikanth |
| Org | Bit Dynamics |
| Status | Technical review draft |
| Created | 2026-08-05 |
| Label | canton-apis |
| Source | Extracted from the Development Fund proposal; milestones, acceptance criteria, funding, and adoption material omitted |

---

## Abstract

 Digital Asset built PQS around a pluggable`Datastore` interface so that ledger ingestion and 
 payload representation could vary
independently. The document (JSONB) representation was implemented; the relational one was not.

This work implements it, then builds the read API the relational representation makes possible: a
versioned HTTP query API that lets an application query ledger data by contract attribute without
direct database access and without building an indexing service.

The backend is developed in a public, continuously-rebased fork of
[`digital-asset/participant-query-store`](https://github.com/digital-asset/participant-query-store)
(Apache-2.0) as additive modules behind the existing `Datastore` interface, and offered upstream
after the backend and API are complete. The query API needs no fork: it reads the PQS database over
SQL and ships as a standalone service in its own repository. `postgres-document` remains the
default and is not modified.

---

## Specification

### 1. Objective

**Give PQS a relational payload representation and a query API over it, so that a Canton
application can query ledger state and history by contract attribute over HTTP without building its
own indexing layer.**

The two halves are one delivery chain. The query API is only viable on top of a typed, indexable
representation; serving arbitrary attribute filters from JSONB is the problem it exists to solve. A
relational representation without an API still requires every application to hold a PostgreSQL
credential and hand-roll SQL. They are the storage and serving ends of one read path, delivered in
sequence, with the API following the backend.

At the end of this work a validator operator can run:

```
# relational projection — a PQS datastore backend
scribe pipeline ledger postgres-relational --pipeline-filter-contracts '...'

# query API — a standalone service against the same database
pqs-query-api serve --port 8080 --auth-issuer https://...
```

and an application can issue attribute queries over HTTP against typed, indexed columns, scoped to
the parties its token authorises.

### 2. Implementation Mechanics

#### 2.1 The extension point already exists

The work is bounded by an interface Digital Asset already defines and ships. At commit `8c4749f`
(2026-07-29):

| Location | Current state |
| :---- | :---- |
| `apps/backend/src/com/digitalasset/scribe/backend/Datastore.scala:21` | `trait Datastore`, documented as *"common interface for pluggable data stores"* — `processAcs`, `processTransactions`, `getFirstCheckpoint`/`getLastCheckpoint`, active-writer registration |
| `apps/postgres/document/` | The JSONB implementation: ~1,340 lines of Scala (`DocumentPostgres`, `SqlSchema`, `Prune`, `model`, `specific`) plus its SQL surface — `R__functions.sql` and `R__views_and_triggers.sql`, 1,403 lines live, over 41 accumulated Flyway migrations |
| `apps/postgres/relational/` | Module declared at `apps/build.sc:390`, linked into the `scribe` assembly at `apps/build.sc:137`, containing a 16-line `Main` that prints `"posgtres-relational not yet unimplemented"` and exits with failure |
| `apps/scribe/.../Main.scala:21` | `- document.Main.app /* | relational.Main.app */` — the relational subcommand is commented out, under a `TODO` |
| `apps/pipeline/.../Main.scala:43-45` | `destinationPostgresRelational` is defined as `ZLayer.fail(Exception("Unimplemented"))` and is not offered by `destination` |

The last three rows are the gaps this design fills; the first two are the infrastructure it builds
on. It adds a second implementation of the existing `Datastore` interface. Ingestion is unchanged
and shared, and no second pipeline is introduced.

#### 2.2 Component A — Relational schema generator

The traversal engine is the library's. `DescriptorSchemaProcessor.process` drives any
`SchemaVisitor` over the Daml-LF descriptors, and PQS already runs two through it:
`document.SqlSchema` for schema initialisation and `JsonCodec` for payload encoding. `SqlSchema` is
a `SchemaVisitor.Unit`: it walks the entity list to emit partition DDL but discards payload type
structure, because JSONB needs no per-field mapping. This work writes the first *structural*
visitor, the DDL generator, plus a `Codec` that materialises `DynamicValue` into typed column
tuples. Both plug into the existing traversal on the `com.daml::transcode-schema` /
`transcode-daml-lf` libraries already on the build path. Neither exists today.

For each template and interface view the generator emits a typed table, keyed by package-name so a
single logical table spans an upgrade line.

| Daml-LF type | PostgreSQL mapping | Notes |
| :---- | :---- | :---- |
| `Text` | `text` | |
| `Int64` | `bigint` | |
| `Numeric n` | `numeric(38, n)` | scale from the type |
| `Bool` | `boolean` | |
| `Party` | `text` | indexed |
| `Date` | `date` | |
| `Timestamp` | `timestamptz` | microsecond precision, UTC |
| `ContractId a` | `text` | indexed; FK candidate to `a`'s table |
| `Optional a` | nullable column | JSONB fallback where `a` is itself nested or optional |
| `Enum` | `text` | discriminator value |
| `Record` | flattened, prefixed columns | deep or recursive records fall back to a child table or JSONB |
| `Variant` | discriminator column plus per-arm nullable columns, or child table | |
| `List a` / `TextMap a` / `GenMap k v` | child table, or JSONB array | configurable per field |

Flattened column names use a deterministic rule with a stable hash suffix past PostgreSQL's 63-byte
identifier limit, since silent truncation would collide sibling fields; the full Daml field path is
recorded in the projection-contract metadata (§2.3), and collision cases are covered by golden
tests. A template exceeding the 1,600-column ceiling is reported, never silently truncated.

**System columns.** Each relational table carries the per-contract metadata the document store
records:

| Column | Type | Origin in the document schema |
| :---- | :---- | :---- |
| `signatories`, `observers` | `text[]` | `__contracts`, added in `V011__Remove_witnesses_add_stakeholders.sql`; the `stakeholders()` helper is `distinct(signatories \|\| observers)` |
| `witnesses` | `text[]` | `__contracts` and `__exercises`, added in `V033__Implement_contracts_divulgence.sql` |
| `divulged_only` | `boolean` | `V033`; distinguishes a contract merely divulged to the participant from one properly disclosed to a stakeholder |
| `redaction_id` | `uuid` | stamped by `redact_contract`/`redact_exercise` (`R__functions.sql:765, 811`) |
| `creation_package_id` | `text` | `V037__Add_creation_package_id.sql`; nullable, no backfill. Read as `coalesce(c.creation_package_id, p.id)` against `__packages` (`R__functions.sql:118`) — left NULL both for pre-V037 contracts and whenever the creating package equals the representative package, which upstream notes is the common case. The projection materializes the coalesced value, not the raw column |
| `contract_id`, `created_at_ix`, `archived_at_ix`, `life_ix` | as today | lifetime tracking. `created_at_ix`/`archived_at_ix` are handles into `__transactions.ix`, a PQS-internal surrogate, not ledger offsets; `life_ix` is a generated `int8range` over the two (`V001:133-136`). The ledger offset is exposed on the `contract` type as `created_at_offset`/`archived_at_offset`, so the projection carries those alongside and rows are pinnable to an offset without a join |

`signatories`/`observers`/`witnesses` are what the authorization model in §2.5 filters on;
`divulged_only` governs visibility and pruning and is already the predicate on the document store's
partial lifetime index (`... using gist (life_ix) ... where not divulged_only`); `redaction_id`
carries erasure state; `creation_package_id` preserves the concrete creating package where it
differs from the representative package, which the projection needs for upgrade-correct decoding.
Package-name keying itself is resolved by `__contract_tpe.package_name`/`module_name`/`entity_name`
and the generated `template_fqn` (`V010__Add_package_name_support.sql:40-54`), which ships today.

The scope is contract-level facts. Transaction-level facts (`transaction_id`, `effective_at`,
`domain_id`, `workflow_id`, `trace_context`, `external_transaction_hash`, `paid_traffic_cost`) stay
in the document store's transaction tables, reachable by join on the transaction index. Reassignment
events are ingested by neither backend today and are out of scope.

**Hybrid mode.** Commonly-queried scalars are promoted to typed, indexed top-level columns;
deeply-nested or rarely-filtered structure stays JSONB. A fully-normalized mode with child tables
for all nested and repeated structure is available for teams wanting complete relational fidelity.

The promotion set is per-template and operator-declared. "Commonly queried" is not a property the
generator can know, and the promotion set *is* the API's query vocabulary. Changing it after tables
exist is a defined maintenance operation: add the column, backfill from the JSONB remainder,
`CREATE INDEX CONCURRENTLY`, bump the projection-contract version, and the API rejects predicates on
the new column until it observes that version. Demoting a field dropped from the remainder requires
re-ingestion, bounded by participant pruning. The procedure and a measured runtime on a partner
dataset are part of the datastore deliverable.

Which templates are projected is controlled by the existing `ContractFilter`/`IdentifierFilter`
configuration: `--pipeline-filter-contracts` on the `pipeline` path, `--filter-contracts` on the
`datastore … schema` path. The filter is not the sole determinant.
`DamlSchema.processFromDescriptors` passes `id => (contractFilter.filter(id) ||
entitiesToAdd.contains(id))`, so interface-implementation integrity pulls excluded templates into
the schema, and in relational mode each becomes a table. Write cost also rises, since `life_ix` is a
stored generated column and is indexed, making every archive a non-HOT update that rewrites every
index on the table. Hybrid mode bounds promoted columns per table but not table count independently
of interface fan-out. Both are measured as part of the datastore work.

#### 2.3 Component B — `postgres-relational` datastore

A `RelationalPostgres` implementation of `Datastore`, mirroring `DocumentPostgres.live`, wired into
`destinationPostgresRelational` and the `scribe datastore` command tree. It reuses the shared
`postgres.backend` module for connection pooling, `InstanceId` and `SchemaConfig`.

`registerActiveWriterAndCleanupTransactions` is a `Datastore` trait method, not something
`postgres.backend` provides. The mechanism behind it (`__watermark`,
`__cleanup_transactions_after_watermark()` (`R__functions.sql:973`), the `__ensure_writer_valid()`
guard and the `__update_watermark_trg` trigger in `R__views_and_triggers.sql`) lives in the document
backend's own migrations, alongside the `ix`↔offset mapping, the checkpoint functions and the
deferred-archive path. The relational backend implements the equivalents against its own schema,
following the same design.

- **Offset consistency.** Writes are driven by the checkpoint mechanism the interface defines, so
  relational rows are pinnable to a ledger offset.
- **Pruning parity.** A relational `Prune` equivalent to `document.Prune`, honouring the same
  offset-based pruning.
- **Redaction parity.** `redact_contract` and `redact_exercise` (`R__functions.sql:765, 811`) null
  the payload/key and argument/result and stamp a `redaction_id`. Relational tables carry
  `redaction_id` and the backend provides equivalents, so redacting in the document store redacts
  the relational projection on the same participant. A document-store backfill refuses rows with
  `redaction_id is not null` and reports them rather than importing all-NULL payload columns; a
  Ledger API re-pipe can resurrect redacted payloads and must be run against a redaction manifest.
  The API excludes redacted rows. Covered by conformance tests.
- **Schema application.** Flyway versions the fixed relational scaffolding: watermark, checkpoints,
  writer fencing, projection-contract metadata. Ledger-derived DDL (one table per template and
  interface view, with its columns and indexes) is generated and applied idempotently at startup
  and on DAR ingestion, outside Flyway, following the `__initialize_contract_tpe` precedent
  (`R__functions.sql:195-229`). Index creation on non-empty tables uses `CREATE INDEX CONCURRENTLY`,
  which runs outside a transaction block.
- **Projection contract.** The backend writes the projection contract (promoted columns, their
  types, indexes and admitted query shapes) into a versioned metadata table in the PQS database at
  schema-apply time, alongside the existing `__contract_tpe`/`__packages` metadata. The API reads it
  at startup and on change, and refuses to start if the contract version is unknown to it.
- **Scope boundary.** The relational backend needs its own DDL, Flyway baseline, prune,
  reset-to-offset and redaction equivalents. It does not reproduce the document store's SQL read
  functions (`creates`/`archives`/`active`/`summary_*`), because the read surface for the relational
  representation is the HTTP API in §2.4.
- **Backfill.** Enabling the backend on an existing deployment runs a bounded backfill from either
  the existing document store or a Ledger API re-pipe at a chosen offset. A document-store backfill
  cannot recover what PQS has already pruned; a re-pipe cannot recover what the participant has
  pruned.

  **Event-class coverage.** `creates()` defaults `from_offset` to `oldest_offset()`, while
  `archives()` and `exercises()` default to `coalesce(pruned_offset(), oldest_offset())`. Only
  `active()`, `summary_active()` and `summary_transients()` carry an explicit `and not
  c.divulged_only` predicate (`R__functions.sql:1090, 1109, 1201`). Divulged-only rows are absent
  from `archives()` incidentally: such a contract never receives an explicit archive event, so its
  `archived_at_ix` stays null and it falls out of the range predicate (`V033`; asserted in
  `DivulgedContractsSpec`).

  The document store's defaults are self-consistent, because pruning deletes a contract's create and
  archive together (`R__functions.sql:548-555`). Two hazards apply to a backfill. An explicit
  `from_offset` below `pruned_offset()` is not rejected: `__nearest_ix_ceil` clamps it to the oldest
  surviving transaction, so the range read is not the range requested. And
  `__delete_transactions_before` sets `created_at_ix = cutoff_ix` for every surviving contract
  created before the cutoff (`R__functions.sql:960-966`), making creation offsets at the boundary
  lower bounds rather than actual values.

  The backfill therefore drives every event class from an explicit common offset range, records the
  source store's pruning watermark, marks imported contracts whose `created_at_ix` equals it, and
  reports per-event-class coverage. The API surfaces the lower-bound marker on offset-pinned and
  history responses.

#### 2.4 Component C — Query API

An HTTP server exposing the relational schema through a versioned contract rather than raw SQL.

**Deployment.** The API reads the PQS database over SQL and never touches the `Datastore`
interface, so it ships as a standalone service in its own public repository, deployable against any
PQS instance. Against a stock document-store deployment it serves the system-column surface
(template, contract id, offset range, active-at, stakeholder scope), so it is usable before the
relational backend. Attribute filtering requires the relational backend: the document store has no
typed attribute columns, and the only alternative, hand-created `payload->>` expression indexes via
`create_index_for_contract`, is the mechanism the Rationale rejects. Witness-scoped history is also
narrower in document mode, because `archives()` blanks the witness set.

It is built on the same ZIO stack PQS uses. `zio-http` is already a declared dependency
(`apps/mill-build/src/millbuild/package.scala:102`) and PQS already serves HTTP with it: the
pipeline's `/livez` and `/readyz` endpoints run on a `zio.http.Server`
(`apps/pipeline/.../health/Health.scala`), exercised by the existing functional tests. The serving
runtime, config shape and functest pattern are in-tree, so the maintainers could later absorb the
API into PQS as a subcommand without a rewrite or a new runtime dependency.

**Filter syntax.** The API adopts PostgREST-compatible filter and ordering syntax
(`?owner=eq.alice&amount=gte.100&order=created_at.desc`), documented prior art with existing client
libraries, mapping one-to-one onto SQL predicates without exposing SQL. OData `$filter` is the
alternative: more expressive and more widely known in enterprise tooling, but substantially larger
to implement correctly and harder to reconcile with the admission control below. If design partners
prefer OData, the same admission model applies.

**Query admission control.** A read API over a large ledger dataset is dangerous to the degree that
it lets callers issue unbounded scans. Column-level allowlisting does not bound cost: with every
named column indexed, `?name=like.*acme*`, `?status=not.eq.done`, `?or=(a.eq.1,b.eq.2)` and a
low-selectivity equality over 100M rows all sit inside a column allowlist and all are expensive. Two
predicates are also mandatory on every query and neither is btree equality: `life_ix @> ix` (GiST
containment) and `text[]` overlap on the party columns, for which the document schema has no index
and installs neither `btree_gist` nor `btree_gin`.

The admission unit is therefore a declared **query shape**, a tuple of (filter columns, operators,
order key, direction), each backed by a named composite index that includes the mandatory `life_ix`
containment and the party-overlap term.

1. **The shape set is declared** in the projection contract, which the backend writes and the API
   reads.
2. **Requests outside the shape set are rejected** with `400` naming the shape and the index that
   would need to exist, so a caller cannot issue an unindexed query and a user-facing UI can be
   pointed at the API directly. A request inside the shape set is bounded by `statement_timeout`
   and per-token concurrency: the worst case is a rejected or timed-out request, not a stalled
   database.
3. **Operator allowlist:** `eq`, `in`, `gt`/`gte`/`lt`/`lte`, prefix-only `like`. Denied: `not.`,
   `or=()`, leading-wildcard `like`/`ilike`, `match`, `fts`.
4. **Keyset pagination is mandatory**, with a configurable maximum page size.
5. **Optional strict mode** runs `EXPLAIN` ahead of execution and rejects plans above a cost
   ceiling.
6. **Per-query-shape metrics**, so operators see which shapes are expensive before they become an
   incident.

The relational schema requires the `btree_gist` extension. Document-store mode has neither that
extension nor party-column indexes, so its shape set is correspondingly smaller.

The API surface:

- **Reads:** active-contract queries and create/exercise/archive history, per template and
  interface view.
- **Temporal semantics:** latest state, state pinned at an offset or time, and events across a
  range.
- **Tail mode:** push-driven where the backend owns the schema: `NOTIFY` on watermark advance, one
  `LISTEN` per API instance, fan-out to SSE subscribers. In document-store mode, a single shared
  watermark poll at a configurable interval. The active mode and interval are reported on the
  readiness endpoint, so an operator can see which one is in effect.
- **Offset watermark:** every response carries the ledger offset it reflects.
- The contract is decoupled from physical table layout, so hybrid-vs-normalized changes do not break
  clients.

#### 2.5 Component D — Authentication and authorization

- **Transport:** TLS required.
- **Authentication:** bearer tokens validated against a configurable OIDC/JWKS issuer. The existing
  `apps/auth` module (`Auth`, `TokenService`, `auth.Config.OAuth`) authenticates PQS *outbound* to
  the Ledger API; inbound validation is new code that reuses the same configuration shape.
- **Authorization:** two layers. PQS contents are already bounded by the ingestion party and
  contract filter `scribe` is configured with. The API applies a narrower scoping: token →
  authorized parties → row-level predicate, default-deny.

  Canton visibility is not the stakeholder set. `witnesses` is recorded separately from
  `signatories`/`observers` because a party can witness an event without being a stakeholder, and
  create, exercise and archive events do not share one visibility rule. The enforced predicate is
  derived from the witness and stakeholder sets as PQS records them per event, and validated in
  conformance tests against what the Ledger API discloses to the same parties. Where the two
  disagree, the API is the more restrictive.

  **There are two witness sets, not three.** PQS records `__contracts.witnesses` (create-time) and
  `__exercises.witnesses`. There is no archive-event witness set: `__events.witnesses` was dropped
  in `V011` and not restored by `V033`, the canonical `Event.Archived` carries no party field, and
  `archives()` returns `'{}'::text[]` with the comment "prevent propagation of witnesses
  information" (`R__functions.sql:1151`). Where an archive arises from a consuming exercise it
  shares the exercise's event pk and the exercise witness set is joinable; where it arrives as a
  bare archived event it is unrecoverable. The relational backend therefore captures the
  archive-visibility set at ingestion instead of reading it out of the document store, and
  materializes exercise stakeholders onto the exercise row so the predicate is NULL-safe and
  default-deny on a missing set. Authorizing archive history on inherited create-time witnesses
  would widen an input upstream deliberately narrowed; that divergence is recorded in §4 and covered
  by the conformance suite.

  **The witness set is deployment-relative.** `signatories`/`observers` are absolute properties of
  the contract. `witnesses` is `evt.witnessParties`, computed relative to
  `--pipeline-filter-parties` and varying with the datasource (`TransactionStream` vs
  `TransactionTreeStream`) and with whether the contract arrived in the ACS seed, so two PQS
  instances over the same ledger can disagree. The backend records the deployment's ingestion party
  set, contract filter and datasource; the API intersects the token's parties with the recorded
  ingestion set and denies outside it, which is what "bounded by the ingestion party" means
  operationally. The conformance test asserts equality with Ledger API disclosure only under
  `AsAnyParty` with an all-templates filter, and subset otherwise.

  **Divulged-only contracts.** `divulged_only` marks a contract whose creation was shared with the
  participant but which will never receive an explicit archive event, and whose stakeholder set need
  not intersect the parties a token authorizes. Participant-level visibility is not party-level
  entitlement. The API excludes `divulged_only` rows from all query results by default, matching
  what `active()`, `summary_active()` and `summary_transients()` enforce explicitly, and making
  explicit on the history path what the document store achieves only incidentally: `creates()` and
  `exercises()` carry no divulgence predicate and do return divulged-only rows under
  `TransactionTreeStream`, so those two paths are today a side channel disclosing contracts the
  active-contract path withholds. Because `__exercises` carries no `divulged_only` column, the
  relational exercise table propagates the flag from the contract at write time, and exercise
  responses do not project the contract's `signatories`/`observers` to a caller authorized only via
  `witnesses`. Divulged-only rows are reachable only through an explicit, separately-authorized
  scope, and never satisfy a stakeholder-scoped query.
- **Auditing:** structured access logs for queries and subscriptions, on the same ZIO/OpenTelemetry
  stack PQS uses. The `app-blocks/o11y` and `app-blocks/logging` sources are Apache-2.0 and light,
  so the API repository mirrors them and the modules line up if the API is later absorbed upstream.

#### 2.6 Configuration

Backend selection, hybrid vs normalized mode, the promotion set, and per-field collection-handling
overrides are driven by `scribe` configuration in the existing style. The query API requires no
`scribe` configuration file: it takes its database connection, listen port, auth issuer and
admission limits as its own service configuration, and reads the projection contract from the
database.

#### 2.7 Where the work lands, and how it reaches upstream

Two repositories, because the halves have different constraints:

- **Relational backend — a public fork of `digital-asset/participant-query-store`.** PQS publishes
  no consumable library artifacts: its release outputs are an executable assembly JAR, a Docker
  image, a DPM component and a Helm chart, and the only `pom` task in `apps/build.sc` exists to feed
  dependency scanning rather than publication. There is no `PublishModule` anywhere in the build.
  The `Datastore` interface is therefore only implementable from inside the codebase, so a fork is
  the only way to build this at all. The fork is public and Apache-2.0 from the first commit.
- **Query API — its own public repository.** No fork, no dependency on PQS internals.

**Fork discipline.** The fork exists to be merged. Changes to shared files are additive and
minimal; the only non-module change is a fork-local CI wrapper, which is not part of the upstream
contribution. `postgres-document` and the existing test suite are untouched, the branch is rebased
onto upstream `main` on a regular cadence, and upstream release-line changes are tracked. An
operator can run the fork's build against a current PQS at any point.

**Upstreaming.** The Digital Asset CLA is signed as soon as there is a PR to sign it against, and
the fork carries Apache-2.0 headers and `make copyright-check` compliance from the first commit.
Once the backend is complete, the modules are offered upstream: PRs opened against
`digital-asset/participant-query-store` and maintained through review. Whether they are merged is
the maintainers' decision.

The repository's `cla.yaml` comment flow is the signing mechanism; its `pull_request` path is
currently inert, because the job's `if` gate tests for `pull_request_target`, an event it does not
subscribe to, so CLA status is confirmed by the maintainers rather than by a passing check.

**Maintainer availability.** Every commit to date is from Digital Asset engineers and the issue
tracker is empty, so there is no established external-contribution flow to rely on. The plan asks
for maintainer time in bounded, scheduled reviews (an early design review and the upstream
contribution) rather than continuously, and no deliverable is blocked on a response between those
points.

**CODEOWNERS.** `**/build.sc`, `**/Makefile`, `.github/**` and `apps/mill-build/**` are CODEOWNERS
paths owned by `@digital-asset/sdk-tools-admin`. Un-stubbing the relational module necessarily
touches `apps/build.sc`, and packaging may touch the Makefiles. `apps/mill-build/**` changes only if
a new dependency coordinate is required, which the plan avoids: `transcode-schema`,
`transcode-daml-lf`, `flyway-core` and `zio-jdbc` are already declared. Those files are the parts of
the upstream contribution most likely to need iteration.

**Upstream naming.** PR #3 (`Rename Scribe to PQS`, open since 2026-07-17) touches 197 files,
including the `postgres-relational` stub itself, `apps/build.sc`, and the repeatable SQL migrations,
and it renames the writer-fencing GUC `scribe.instance` to `pqs.instance`. This document uses
current names for precision. The fork takes the fencing GUC from `postgres.backend.instanceIdProp`
instead of hardcoding it, and whichever phase is in flight when #3 merges budgets a one-time
mechanical re-baselining: package and import rename across the fork's new modules, plus conflict
resolution in the files the fork also edits. Delivery is not gated on #3 merging.

**Governance.** Grant #67 provides for transferring the repository to the `canton-foundation`
namespace *if and when* the Daml SDK transitions; its M1 acceptance criterion requires only that the
repository be prepared for that move. The contribution follows the repository wherever it is
governed.

### 3. Architectural Alignment

- **canton-apis SIG.** Fills a gap in the Canton read-API surface: a first-party, attribute-queryable
  HTTP read API.
- **App Building and Developer Experience.** Removes a component every application team currently
  builds for itself, and lowers total cost of ownership by one service to run, monitor, back up and
  secure.
- **Consistent with Digital Asset's design.** The `Datastore` abstraction, the reserved
  `postgres-relational` module and the commented-out CLI wiring are DA's, not this proposal's.
- **CIP-0100 / CIP-0082.** Everything is Apache-2.0 and public from the first commit: the relational
  backend in a public fork of PQS, offered upstream when complete; the query API in its own public
  repository.

**Relationship to the approved PQS open-sourcing grant (PR
[#67](https://github.com/canton-foundation/canton-dev-fund/pull/67)).** #67 M2 delivers
zero-downtime DAR ingestion, orphaned-package decoding and single-writer concurrency control; #67 M3
delivers Smart Contract Upgrade and interface-lineage mapping, input-contract support, and validator
Helm chart integration. **No part of that work is re-done here.** This design consumes it.
Package-name keying and representative-package resolution already ship at `8c4749f`
(`V010__Add_package_name_support.sql`; `coalesce(creation_package_id, package_id)` in
`__contracts()`), and the relational backend consumes them as they stand. It reuses the existing
`registerActiveWriterAndCleanupTransactions` hook rather than adding a second concurrency-control
mechanism, so #67 M2's single-writer work applies unchanged. Schema generation interoperates with
#67 M2's dynamic DAR ingestion but does not require it: without it, relational DDL generation
follows the same restart-on-new-DAR behaviour PQS has today. #67 M3 demonstrates that upgrades and
acquired interfaces are resolved correctly in the document store; this work verifies that the
relational *projection* stays correct under that resolution. Different artifacts under test. If
#67 M3 slips, upgrade-line verification slips with it; earlier phases do not depend on it.

### 4. Backward Compatibility

No backward compatibility impact for existing deployments.

- `postgres-document` is not modified. It remains the default and the only backend selected unless
  an operator explicitly selects otherwise.
- The relational backend is a separate module behind a separate CLI target, and the query API is a
  separate service in a separate repository. Neither sits on any existing code path.
- No change to the Ledger API, the Canton protocol, or transaction submission.
- No change to the existing SQL/JDBC surface. Applications querying the document store continue to
  work unmodified, including alongside a relational projection on the same participant.
- Relational schema evolution within an upgrade line is additive (nullable columns).
- One deliberate divergence: archive-history visibility is captured at ingestion rather than
  inherited from create-time witnesses, so relational archive history can be narrower than a naive
  join over document-store columns would produce. This is documented and tested (§2.5).

---

## Rationale

**Why extend PQS.** The default approach should be to extend what exists, and here the component
exists, is adopted, is Apache-2.0, and already contains a named, reserved, unimplemented extension
point for this. Building the same capability elsewhere would mean re-implementing offset tracking,
reconnection, backpressure, crash recovery, single-writer concurrency control, Daml-LF decoding,
package resolution, pruning, redaction and schema migration, all of which `scribe` already does.

Alternatives considered:

- **Views over the existing JSONB tables.** The cheapest option, and useful for some access
  patterns. A view over JSONB does not give the planner typed columns, native btree indexes, or
  column statistics; selectivity estimates on `payload->>'field'` are poor, every row must be
  deserialized to be filtered, and expression indexes must be created and maintained by hand per
  query shape. That last point is disqualifying for an API whose safety model depends on knowing in
  advance which query shapes are indexed.
- **A separate indexing service reading from PQS.** Workable, and what teams do today. It costs a
  second datastore, a second copy of the data, a second trust boundary, a second set of pruning and
  redaction rules to keep in sync, and it turns a streamed pipeline into a polled one.
- **Exposing PostgREST or a GraphQL gateway directly over the tables.** Worth offering later as a
  power-user path. Not suitable as the primary surface: it couples clients to physical table layout,
  pushes authorization down to database roles rather than party-scoped predicates, and has no notion
  of ledger offsets or temporal perspectives. Adopting PostgREST's filter syntax while owning the
  serving layer takes the ergonomics without the coupling.
- **A new query language.** Rejected: a custom grammar is a permanent maintenance and documentation
  burden, and unbounded expressiveness conflicts with the admission control in §2.4.

**Why a fork.** PQS publishes no consumable library artifacts, so the `Datastore` interface is only
implementable from inside the codebase. Making the fork public, additive and continuously rebased
keeps the eventual merge cheap, lets operators run the work throughout development, and decouples
delivery from another organisation's review calendar without giving up the upstream outcome.
The query API, which needs no fork, is kept out of it entirely.

**Why the API is designed with the backend.** The admission control in §2.4 is implementable only
because the projection contract declares the indexed query shapes; the two are designed against
each other. They ship as separate artifacts and are delivered in sequence.

---

### Notes for reviewers

Every claim about the current PQS codebase is against commit `8c4749f` (2026-07-29). Extension
points (§2.1) are checkable in `Datastore.scala`, `apps/build.sc` and the two `Main.scala` files;
column and index claims (§2.2) in the versioned migrations under
`apps/postgres/document/resources/db/migration/`; read-function semantics (§2.3, §2.5) in
`R__functions.sql` in that same directory, which since `V036__Extract_functions_to_repeatable.sql`
is the sole source of truth for function definitions; the `V0xx` copies are superseded and their
signatures no longer reflect current behaviour.
