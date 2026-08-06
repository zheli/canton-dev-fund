## PQS Relational Backend and Query API - Technical Review

| Field | Value |
| :---- | :---- |
| Author | Srikanth |
| Org | Bit Dynamics |
| Status | Technical review draft |
| Created | 2026-08-05 |
| Label | canton-apis |
| Source | Extracted from the Development Fund proposal. Milestones, acceptance criteria, funding, and adoption material are omitted. |

## Abstract

Digital Asset built PQS around a pluggable `Datastore` interface. Ledger ingestion and payload
representation can vary independently. The document (JSONB) representation exists. The relational
representation does not.

This work implements the relational representation. It then builds a versioned HTTP query API over
that representation. An application can query ledger data by contract attribute. It does not need
direct database access. It does not need to build an indexing service.

## Specification

### 1. Objective

**Give PQS a relational payload representation and a query API over it. A Canton application can
query ledger state and history by contract attribute over HTTP. It does not build its own indexing
layer.**

The two halves form one delivery chain. The query API needs a typed, indexable representation.
Attribute filters on JSONB are the problem this work solves. A relational store without an API still
forces each application to hold a PostgreSQL credential and write SQL by hand. Storage and serving
are the two ends of one read path. The API follows the backend.

At the end of this work a validator operator can run:

```
# relational projection - a PQS datastore backend
scribe pipeline ledger postgres-relational --pipeline-filter-contracts '...'

# query API - a standalone service against the same database
pqs-query-api serve --port 8080 --auth-issuer https://...
```

An application can then issue attribute queries over HTTP against typed, indexed columns. Results
are scoped to the parties its token authorizes.

### 2. Implementation Mechanics

#### 2.1 The extension point already exists

Digital Asset already defines and ships the interface that bounds this work. At commit `8c4749f`
(2026-07-29):

| Location | Current state |
| :---- | :---- |
| `apps/backend/src/com/digitalasset/scribe/backend/Datastore.scala:21` | `trait Datastore`, documented as *"common interface for pluggable data stores"* - `processAcs`, `processTransactions`, `getFirstCheckpoint`/`getLastCheckpoint`, active-writer registration |
| `apps/postgres/document/` | The JSONB implementation: ~1,340 lines of Scala (`DocumentPostgres`, `SqlSchema`, `Prune`, `model`, `specific`) plus its SQL surface - `R__functions.sql` and `R__views_and_triggers.sql`, 1,403 lines live, over 41 accumulated Flyway migrations |
| `apps/postgres/relational/` | Module declared at `apps/build.sc:390`, linked into the `scribe` assembly at `apps/build.sc:137`, containing a 16-line `Main` that prints `"posgtres-relational not yet unimplemented"` and exits with failure |
| `apps/scribe/.../Main.scala:21` | `- document.Main.app /* | relational.Main.app */` - the relational subcommand is commented out, under a `TODO` |
| `apps/pipeline/.../Main.scala:43-45` | `destinationPostgresRelational` is defined as `ZLayer.fail(Exception("Unimplemented"))` and is not offered by `destination` |

The last three rows are the gaps this design fills. The first two are the base it uses. The design
adds a second implementation of the existing `Datastore` interface. Ingestion stays unchanged and
shared. The design does not add a second pipeline.

#### 2.2 Component A - Relational schema generator

The traversal engine belongs to the library. `DescriptorSchemaProcessor.process` drives any
`SchemaVisitor` over the Daml-LF descriptors. PQS already runs two visitors through it:
`document.SqlSchema` for schema setup and `JsonCodec` for payload encoding. `SqlSchema` is a
`SchemaVisitor.Unit`. It walks the entity list to emit partition DDL. It drops payload type
structure, because JSONB needs no per-field mapping. This work writes the first *structural*
visitor: the DDL generator. It also writes a `Codec` that turns `DynamicValue` into typed column
tuples. Both plug into the existing traversal on the `com.daml::transcode-schema` /
`transcode-daml-lf` libraries already on the build path. Neither exists today.

For each template and interface view the generator emits a typed table. The table key is
package-name, so one logical table spans an upgrade line.

| Daml-LF type | PostgreSQL mapping | Notes |
| :---- | :---- | :---- |
| `Text` | `text` | |
| `Int64` | `bigint` | |
| `Numeric n` | `numeric(38, n)` | scale from the type |
| `Bool` | `boolean` | |
| `Party` | `text` | indexed |
| `Date` | `date` | |
| `Timestamp` | `timestamptz` | microsecond precision, UTC |
| `ContractId a` | `text` | indexed. FK candidate to `a`'s table |
| `Optional a` | nullable column | JSONB fallback where `a` is itself nested or optional |
| `Enum` | `text` | discriminator value |
| `Record` | flattened, prefixed columns | deep or recursive records fall back to a child table or JSONB |
| `Variant` | discriminator column plus per-arm nullable columns, or child table | |
| `List a` / `TextMap a` / `GenMap k v` | child table, or JSONB array | configurable per field |

Flattened column names use a fixed rule. Past the 63-byte identifier limit in PostgreSQL, names get a
stable hash suffix. Silent truncation would collide sibling fields. The full Daml field path is
recorded in the projection-contract metadata (§2.3). Golden tests cover collision cases. If a
template exceeds the 1,600-column ceiling, the generator reports it. It never truncates in silence.

**System columns.** Each relational table carries the per-contract metadata the document store
records:

| Column | Type | Origin in the document schema |
| :---- | :---- | :---- |
| `signatories`, `observers` | `text[]` | `__contracts`, added in `V011__Remove_witnesses_add_stakeholders.sql`. The `stakeholders()` helper is `distinct(signatories \|\| observers)` |
| `witnesses` | `text[]` | `__contracts` and `__exercises`, added in `V033__Implement_contracts_divulgence.sql` |
| `divulged_only` | `boolean` | `V033`. Marks a contract only divulged to the participant, not properly disclosed to a stakeholder |
| `redaction_id` | `uuid` | stamped by `redact_contract`/`redact_exercise` (`R__functions.sql:765, 811`) |
| `creation_package_id` | `text` | `V037__Add_creation_package_id.sql`. Nullable, no backfill. Read as `coalesce(c.creation_package_id, p.id)` against `__packages` (`R__functions.sql:118`). Left NULL both for pre-V037 contracts and when the creating package equals the representative package. Upstream notes that common case. The projection stores the coalesced value, not the raw column |
| `contract_id`, `created_at_ix`, `archived_at_ix`, `life_ix` | as today | lifetime tracking. `created_at_ix`/`archived_at_ix` are handles into `__transactions.ix`, a PQS-internal surrogate, not ledger offsets. `life_ix` is a generated `int8range` over the two (`V001:133-136`). The ledger offset is exposed on the `contract` type as `created_at_offset`/`archived_at_offset`. The projection carries those too, so rows pin to an offset without a join |

The authorization model in §2.5 filters on `signatories`/`observers`/`witnesses`.
`divulged_only` controls visibility and pruning. It is already the predicate on the partial lifetime index in the document
store (`... using gist (life_ix) ... where not divulged_only`). `redaction_id`
carries erasure state. `creation_package_id` keeps the concrete creating package when it differs
from the representative package. The projection needs that for upgrade-correct decoding.
Package-name keying is resolved by `__contract_tpe.package_name`/`module_name`/`entity_name` and
the generated `template_fqn` (`V010__Add_package_name_support.sql:40-54`). That ships today.

The scope is contract-level facts. Transaction-level facts (`transaction_id`, `effective_at`,
`domain_id`, `workflow_id`, `trace_context`, `external_transaction_hash`, `paid_traffic_cost`) stay
in the transaction tables in the document store. They are reachable by join on the transaction index.
Neither backend ingests reassignment events today. Reassignment events are out of scope.

**Hybrid mode.** Operators promote commonly queried scalars to typed, indexed top-level columns.
Deep or rarely filtered structure stays JSONB. A fully normalized mode with child tables for all
nested and repeated structure is also available.

The promotion set is per-template and operator-declared. The generator cannot know which fields are
"commonly queried". The promotion set *is* the query vocabulary of the API. Changing it after tables
exist is a defined maintenance operation:

1. Add the column.
2. Backfill from the JSONB remainder.
3. Run `CREATE INDEX CONCURRENTLY`.
4. Bump the projection-contract version.

The API rejects predicates on the new column until it sees that version. Demoting a field that left
the remainder needs re-ingestion. Participant pruning bounds that work. The procedure and a measured
runtime on a partner dataset are part of the datastore deliverable.

Existing `ContractFilter`/`IdentifierFilter` configuration controls which templates are projected:
`--pipeline-filter-contracts` on the `pipeline` path, `--filter-contracts` on the
`datastore … schema` path. The filter is not the only factor.
`DamlSchema.processFromDescriptors` passes `id => (contractFilter.filter(id) ||
entitiesToAdd.contains(id))`. Interface-implementation integrity pulls excluded templates into the
schema. In relational mode each becomes a table. Write cost also rises. `life_ix` is a stored
generated column and is indexed. Every archive is a non-HOT update that rewrites every index on the
table. Hybrid mode bounds promoted columns per table. It does not bound table count apart from
interface fan-out. Both effects are measured as part of the datastore work.

#### 2.3 Component B - `postgres-relational` datastore

`RelationalPostgres` implements `Datastore`. It mirrors `DocumentPostgres.live`. It wires into
`destinationPostgresRelational` and the `scribe datastore` command tree. It reuses the shared
`postgres.backend` module for connection pooling, `InstanceId`, and `SchemaConfig`.

`registerActiveWriterAndCleanupTransactions` is a `Datastore` trait method.
`postgres.backend` does not provide it. The mechanism behind it lives in the migrations owned by the document
backend: `__watermark`, `__cleanup_transactions_after_watermark()` (`R__functions.sql:973`), the
`__ensure_writer_valid()` guard, and the `__update_watermark_trg` trigger in
`R__views_and_triggers.sql`. Those migrations also hold the `ix` to offset mapping, the checkpoint
functions, and the deferred-archive path. The relational backend implements the equivalents against
its own schema. It follows the same design.

- **Offset consistency.** Writes use the checkpoint mechanism the interface defines. Relational rows
  pin to a ledger offset.
- **Pruning parity.** A relational `Prune` matches `document.Prune`. It honors the same offset-based
  pruning.
- **Redaction parity.** `redact_contract` and `redact_exercise` (`R__functions.sql:765, 811`) null
  the payload/key and argument/result and stamp a `redaction_id`. Relational tables carry
  `redaction_id`. The backend provides equivalents. A redaction in the document store redacts the
  relational projection on the same participant. A document-store backfill refuses rows with
  `redaction_id is not null`. It reports them. It does not import all-NULL payload columns. A Ledger
  API re-pipe can restore redacted payloads. It must run against a redaction manifest. The API
  excludes redacted rows. Conformance tests cover this.
- **Schema application.** Flyway versions the fixed relational scaffolding: watermark, checkpoints,
  writer fencing, projection-contract metadata. Ledger-derived DDL (one table per template and
  interface view, with its columns and indexes) is generated and applied at startup and on DAR
  ingestion. That path is idempotent and outside Flyway. It follows the `__initialize_contract_tpe`
  precedent (`R__functions.sql:195-229`). Index creation on non-empty tables uses
  `CREATE INDEX CONCURRENTLY`. That runs outside a transaction block.
- **Projection contract.** At schema-apply time the backend writes the projection contract
  (promoted columns, types, indexes, and admitted query shapes) into a versioned metadata table in
  the PQS database. It sits beside the existing `__contract_tpe`/`__packages` metadata. The API
  reads it at startup and on change. The API refuses to start if the contract version is unknown.
- **Scope boundary.** The relational backend needs its own DDL, Flyway baseline, prune,
  reset-to-offset, and redaction equivalents. It does not reproduce the SQL read
  functions of the document store (`creates`/`archives`/`active`/`summary_*`). The HTTP API in §2.4 is the read surface for
  the relational representation.
- **Backfill.** Enabling the backend on an existing deployment runs a bounded backfill from either
  the existing document store or a Ledger API re-pipe at a chosen offset. A document-store backfill
  cannot recover what PQS has already pruned. A re-pipe cannot recover what the participant has
  pruned.

  **Event-class coverage.** `creates()` defaults `from_offset` to `oldest_offset()`.
  `archives()` and `exercises()` default to `coalesce(pruned_offset(), oldest_offset())`. Only
  `active()`, `summary_active()`, and `summary_transients()` carry an explicit `and not
  c.divulged_only` predicate (`R__functions.sql:1090, 1109, 1201`). Divulged-only rows are absent
  from `archives()` by chance. Such a contract never receives an explicit archive event. Its
  `archived_at_ix` stays null. It falls out of the range predicate (`V033`. Asserted in
  `DivulgedContractsSpec`).

  The defaults of the document store are self-consistent. Pruning deletes the create and archive of a contract
  together (`R__functions.sql:548-555`). Two hazards apply to a backfill. An explicit `from_offset`
  below `pruned_offset()` is not rejected. `__nearest_ix_ceil` clamps it to the oldest surviving
  transaction. The range read is not the range requested.
  `__delete_transactions_before` sets `created_at_ix = cutoff_ix` for every surviving contract
  created before the cutoff (`R__functions.sql:960-966`). Creation offsets at the boundary become
  lower bounds, not actual values.

  The backfill therefore drives every event class from one explicit common offset range. It records
  the pruning watermark of the source store. It marks imported contracts whose `created_at_ix` equals
  that watermark. It reports per-event-class coverage. The API shows the lower-bound marker on
  offset-pinned and history responses.

#### 2.4 Component C - Query API

An HTTP server exposes the relational schema through a versioned contract. It does not expose raw
SQL.

**Deployment.** The API reads the PQS database over SQL. It never touches the `Datastore`
interface. It ships as a standalone service in its own public repository. It can deploy against any
PQS instance. Against a stock document-store deployment it serves the system-column surface
(template, contract id, offset range, active-at, stakeholder scope). That mode works before the
relational backend exists. Attribute filtering needs the relational backend. The document store has
no typed attribute columns. The only alternative is hand-created `payload->>` expression indexes via
`create_index_for_contract`. The Rationale rejects that path. Witness-scoped history is also
narrower in document mode, because `archives()` blanks the witness set.

The API uses the same ZIO stack as PQS. `zio-http` is already a declared dependency
(`apps/mill-build/src/millbuild/package.scala:102`). PQS already serves HTTP with it. The
`/livez` and `/readyz` endpoints of the pipeline run on a `zio.http.Server`
(`apps/pipeline/.../health/Health.scala`). Existing functional tests exercise them. The serving
runtime, config shape, and functest pattern are in-tree. Maintainers could later absorb the API into
PQS as a subcommand. That would need no rewrite and no new runtime dependency.

**Filter syntax.** The API uses PostgREST-compatible filter and ordering syntax
(`?owner=eq.alice&amount=gte.100&order=created_at.desc`). That syntax is documented prior art with
existing client libraries. It maps one-to-one onto SQL predicates. It does not expose SQL. OData
`$filter` is the alternative. It is more expressive and more widely known in enterprise tooling. It
is also larger to implement correctly. It is harder to align with the admission control below. If
design partners prefer OData, the same admission model applies.

**Query admission control.** A read API over a large ledger dataset is dangerous when callers can
issue unbounded scans. Column-level allowlisting does not bound cost. With every named column
indexed, `?name=like.*acme*`, `?status=not.eq.done`, `?or=(a.eq.1,b.eq.2)`, and a low-selectivity
equality over 100M rows all sit inside a column allowlist. All are expensive. Two predicates are
also mandatory on every query. Neither is btree equality: `life_ix @> ix` (GiST containment) and
`text[]` overlap on the party columns. The document schema has no index for those. It installs
neither `btree_gist` nor `btree_gin`.

The admission unit is therefore a declared **query shape**. A shape is a tuple of (filter columns,
operators, order key, direction). Each shape is backed by a named composite index. That index
includes the mandatory `life_ix` containment and the party-overlap term.

1. **The shape set is declared** in the projection contract. The backend writes it. The API reads
   it.
2. **Requests outside the shape set are rejected** with `400`. The response names the shape and the
   index that would need to exist. A caller cannot issue an unindexed query. A user-facing UI can
   point at the API directly. A request inside the shape set is bounded by `statement_timeout` and
   per-token concurrency. The worst case is a rejected or timed-out request. It is not a stalled
   database.
3. **Operator allowlist:** `eq`, `in`, `gt`/`gte`/`lt`/`lte`, prefix-only `like`. Denied: `not.`,
   `or=()`, leading-wildcard `like`/`ilike`, `match`, `fts`.
4. **Keyset pagination is mandatory**, with a configurable maximum page size.
5. **Optional strict mode** runs `EXPLAIN` ahead of execution and rejects plans above a cost
   ceiling.
6. **Per-query-shape metrics** show operators which shapes are expensive before they become an
   incident.

The relational schema requires the `btree_gist` extension. Document-store mode has neither that
extension nor party-column indexes. Its shape set is smaller.

The API surface:

- **Reads:** active-contract queries and create/exercise/archive history, per template and
  interface view.
- **Temporal semantics:** latest state, state pinned at an offset or time, and events across a
  range.
- **Tail mode:** push-driven where the backend owns the schema. `NOTIFY` on watermark advance. One
  `LISTEN` per API instance. Fan-out to SSE subscribers. In document-store mode, one shared
  watermark poll at a configurable interval. The readiness endpoint reports the active mode and
  interval. An operator can see which one is in effect.
- **Offset watermark:** every response carries the ledger offset it reflects.
- The contract is separate from physical table layout. Hybrid or normalized changes do not break
  clients.

#### 2.5 Component D - Authentication and authorization

- **Transport:** TLS required.
- **Authentication:** bearer tokens validated against a configurable OIDC/JWKS issuer. The existing
  `apps/auth` module (`Auth`, `TokenService`, `auth.Config.OAuth`) authenticates PQS *outbound* to
  the Ledger API. Inbound validation is new code. It reuses the same configuration shape.
- **Authorization:** two layers. PQS contents are already bounded by the ingestion party and
  contract filter configured on `scribe`. The API applies a narrower scope: token to authorized
  parties to row-level predicate. Default deny.

  Canton visibility is not the stakeholder set. `witnesses` is recorded separately from
  `signatories`/`observers`. A party can witness an event without being a stakeholder. Create,
  exercise, and archive events do not share one visibility rule. The enforced predicate comes from
  the witness and stakeholder sets as PQS records them per event. Conformance tests check it against
  what the Ledger API discloses to the same parties. Where the two disagree, the API is more
  restrictive.

  **There are two witness sets, not three.** PQS records `__contracts.witnesses` (create-time) and
  `__exercises.witnesses`. There is no archive-event witness set. `__events.witnesses` was dropped
  in `V011` and not restored by `V033`. The canonical `Event.Archived` carries no party field.
  `archives()` returns `'{}'::text[]` with the comment "prevent propagation of witnesses
  information" (`R__functions.sql:1151`). When an archive comes from a consuming exercise, it shares
  the event pk of the exercise and the exercise witness set is joinable. When it arrives as a bare
  archived event, the set is unrecoverable. The relational backend therefore captures the
  archive-visibility set at ingestion. It does not read it from the document store. It also stores
  exercise stakeholders on the exercise row. The predicate is NULL-safe and default-deny on a
  missing set. Authorizing archive history on inherited create-time witnesses would widen an input
  that upstream narrowed on purpose. §4 records that divergence. The conformance suite covers it.

  **The witness set is deployment-relative.** `signatories`/`observers` are absolute properties of
  the contract. `witnesses` is `evt.witnessParties`. It is computed relative to
  `--pipeline-filter-parties`. It varies with the datasource (`TransactionStream` vs
  `TransactionTreeStream`) and with whether the contract arrived in the ACS seed. Two PQS
  instances over the same ledger can disagree. The backend records the ingestion party of the deployment
  set, contract filter, and datasource. The API intersects the parties in the token with the recorded
  ingestion set. It denies parties outside that set. That is what "bounded by the ingestion party"
  means in operation. The conformance test asserts equality with Ledger API disclosure only under
  `AsAnyParty` with an all-templates filter. Otherwise it asserts subset.

  **Divulged-only contracts.** `divulged_only` marks a contract whose creation was shared with the
  participant. That contract will never receive an explicit archive event. Its stakeholder set need
  not intersect the parties a token authorizes. Participant-level visibility is not party-level
  entitlement. By default the API excludes `divulged_only` rows from all query results. That matches
  what `active()`, `summary_active()`, and `summary_transients()` enforce. It also makes the history
  path explicit. The document store does this only by chance: `creates()` and `exercises()` carry no
  divulgence predicate and do return divulged-only rows under `TransactionTreeStream`. Those two
  paths are a side channel today. They disclose contracts the active-contract path withholds.
  `__exercises` has no `divulged_only` column. The relational exercise table copies the flag from
  the contract at write time. Exercise responses do not project the
  `signatories` of the contract/`observers` to a caller authorized only via `witnesses`. Divulged-only rows are
  reachable only through an explicit, separately authorized scope. They never satisfy a
  stakeholder-scoped query.
- **Auditing:** structured access logs for queries and subscriptions, on the same ZIO/OpenTelemetry
  stack PQS uses. The `app-blocks/o11y` and `app-blocks/logging` sources are Apache-2.0 and light.
  The API repository mirrors them. The modules line up if the API is later absorbed upstream.

#### 2.6 Configuration

`scribe` configuration in the existing style drives backend selection, hybrid vs normalized mode,
the promotion set, and per-field collection-handling overrides. The query API needs no `scribe`
configuration file. It takes its database connection, listen port, auth issuer, and admission limits
as its own service configuration. It reads the projection contract from the database.

### 3. Architectural Alignment

- **canton-apis SIG.** Fills a gap in the Canton read-API surface: a first-party, attribute-queryable
  HTTP read API.
- **App Building and Developer Experience.** Removes a component every application team currently
  builds for itself. Lowers total cost of ownership by one service to run, monitor, back up, and
  secure.
- **Consistent with the Digital Asset design.** The `Datastore` abstraction, the reserved
  `postgres-relational` module, and the commented-out CLI wiring belong to Digital Asset. They do not belong to this
  proposal.
- **CIP-0100 / CIP-0082.** Everything is Apache-2.0 and public from the first commit. The relational
  backend lives in a public fork of PQS and is offered upstream when complete. The query API lives
  in its own public repository.

**Relationship to the approved PQS open-sourcing grant (PR
[#67](https://github.com/canton-foundation/canton-dev-fund/pull/67)).** #67 M2 delivers
zero-downtime DAR ingestion, orphaned-package decoding, and single-writer concurrency control. #67
M3 delivers Smart Contract Upgrade and interface-lineage mapping, input-contract support, and
validator Helm chart integration. **No part of that work is re-done here.** This design consumes it.
Package-name keying and representative-package resolution already ship at `8c4749f`
(`V010__Add_package_name_support.sql`. `coalesce(creation_package_id, package_id)` in
`__contracts()`). The relational backend consumes them as they stand. It reuses the existing
`registerActiveWriterAndCleanupTransactions` hook. It does not add a second concurrency-control
mechanism. the single-writer work from #67 M2 applies unchanged. Schema generation works with dynamic DAR ingestion from #67 M2. It does not require it. Without it, relational DDL generation follows the
same restart-on-new-DAR behavior PQS has today. #67 M3 shows that upgrades and acquired interfaces
are resolved correctly in the document store. This work checks that the relational *projection*
stays correct under that resolution. Different artifacts under test. If #67 M3 slips, upgrade-line
verification slips with it. Earlier phases do not depend on it.

### 4. Backward Compatibility

No backward compatibility impact for existing deployments.

- `postgres-document` is not modified. It remains the default. It is the only backend selected
  unless an operator selects otherwise.
- The relational backend is a separate module behind a separate CLI target. The query API is a
  separate service in a separate repository. Neither sits on any existing code path.
- No change to the Ledger API, the Canton protocol, or transaction submission.
- No change to the existing SQL/JDBC surface. Applications that query the document store continue to
  work unmodified. That includes use beside a relational projection on the same participant.
- Relational schema evolution within an upgrade line is additive (nullable columns).
- One deliberate divergence: archive-history visibility is captured at ingestion. It is not
  inherited from create-time witnesses. Relational archive history can be narrower than a naive join
  over document-store columns would produce. This is documented and tested (§2.5).

---

## Rationale

**Why extend PQS.** The default approach is to extend what exists. Here the component exists. It is
adopted. It is Apache-2.0. It already contains a named, reserved, unimplemented extension point for
this. Building the same capability elsewhere would mean re-implementing offset tracking,
reconnection, backpressure, crash recovery, single-writer concurrency control, Daml-LF decoding,
package resolution, pruning, redaction, and schema migration. `scribe` already does all of that.

Alternatives considered:

- **Views over the existing JSONB tables.** The cheapest option. Useful for some access patterns. A
  view over JSONB does not give the planner typed columns, native btree indexes, or column
  statistics. Selectivity estimates on `payload->>'field'` are poor. Every row must be deserialized
  to be filtered. Expression indexes must be created and maintained by hand per query shape. That
  last point disqualifies an API whose safety model depends on knowing in advance which query shapes
  are indexed.
- **A separate indexing service reading from PQS.** Workable. Teams do this today. It costs a second
  datastore, a second copy of the data, a second trust boundary, and a second set of pruning and
  redaction rules to keep in sync. It also turns a streamed pipeline into a polled one.
- **Exposing PostgREST or a GraphQL gateway directly over the tables.** Worth offering later as a
  power-user path. Not suitable as the primary surface. It couples clients to physical table layout.
  It pushes authorization down to database roles rather than party-scoped predicates. It has no
  notion of ledger offsets or temporal perspectives. Adopting the PostgREST filter syntax while owning
  the serving layer keeps the ergonomics without the coupling.
- **A new query language.** Rejected. A custom grammar is a permanent maintenance and documentation
  burden. Unbounded expressiveness conflicts with the admission control in §2.4.

**Why a fork.** PQS publishes no consumable library artifacts. The `Datastore` interface is only
implementable from inside the codebase. Making the fork public, additive, and continuously rebased
keeps the eventual merge cheap. Operators can run the work throughout development. Delivery is not
tied to the review calendar of another organization. The upstream outcome is still available. The query
API needs no fork. It is kept out of the fork entirely.

**Why the API is designed with the backend.** The admission control in §2.4 works only because the
projection contract declares the indexed query shapes. The two are designed against each other. They
ship as separate artifacts. They are delivered in sequence.

---

### Notes for reviewers

Every claim about the current PQS codebase is against commit `8c4749f` (2026-07-29). Extension
points (§2.1) are checkable in `Datastore.scala`, `apps/build.sc`, and the two `Main.scala` files.
Column and index claims (§2.2) are in the versioned migrations under
`apps/postgres/document/resources/db/migration/`. Read-function semantics (§2.3, §2.5) are in
`R__functions.sql` in that same directory. Since `V036__Extract_functions_to_repeatable.sql`, that
file is the sole source of truth for function definitions. The `V0xx` copies are superseded. Their
signatures no longer reflect current behavior.
