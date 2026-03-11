## Development Fund Proposal

**Author:**  Zhe Li <zhe@bitdynamics.me>, Srikanth Yeleswarapu <srikanth@bitdynamics.me>

**Status:** Submitted  

**Created:** 2026-02-22

---

## Abstract

**Canton DevKit** is a proposed **one-stop LocalNet toolkit** for the Canton network that makes local development faster, more discoverable, and more approachable for new and experienced developers alike.

The proposal is grounded in the 2026 Canton developer experience survey: 41% of respondents identified environment setup and node operations as the hardest part of getting started, observability was ranked the highest-priority tooling improvement area, and most respondents were new to Canton but already familiar with mature Ethereum-style developer workflows. DevKit addresses that gap with a unified CLI and web UI for LocalNet lifecycle management, monitoring, debugging-oriented workflows, and CIP-56 token experimentation so developers can focus on building applications instead of managing infrastructure.

---

## Specification

### 1. Objective

The current official LocalNet stack creates significant friction for onboarding, workshops, hackathons, and AI-assisted coding because it requires users to manually manage:

* Docker containers
* Configuration files
* Environment variables
* Observability setup
* Ad-hoc scripts and tools for inspection and token operations

This proposal directly responds to the pain points surfaced in the 2026 developer survey:

* **41% of developers** said environment setup and node operations were the hardest part of starting development.
* Developers explicitly asked for a unified development framework comparable to **Hardhat, Foundry, or Anchor**.
* **Observability** was ranked as the most important tooling improvement area.
* Developers also highlighted weak debugging and transaction analysis workflows, with several referencing a **Tenderly-like** experience as the ideal benchmark.

The goal is to deliver a unified DevKit that reduces infrastructure complexity and standardizes the local development workflow. This maintained tooling will enable any developer or AI agent to manage the complete lifecycle of one or more LocalNets through simple commands or a UI, monitor and explore activity, and experiment with CantonCoin and CIP-56 tokens without assembling a custom toolchain first.

### 2. Implementation Mechanics
(Explain how the solution will be implemented. Include technologies, components, workflows, and operational approach.)

The solution is delivered as a **standalone CLI application** (`canton-devkit`) with a web UI. It uses Docker containers and runs LocalNet using Splice nodes, but packages the developer experience into a single installable tool that requires no git clone, no Makefile knowledge, and no manual environment variable setup. Optional helper services can be enabled or disabled as needed.

#### Relationship to Existing Tooling

Canton already ships several developer tools. The DevKit is designed to complement—not replace—them:

| Existing Tool | What It Does | DevKit Relationship |
|---|---|---|
| **DPM** (`dpm`) | SDK management, Daml build/test/codegen, lightweight single-process sandbox (`dpm sandbox`), Daml Shell | DevKit does **not** replicate DPM functionality. Developers continue to use `dpm` for Daml compilation, testing, and codegen. DevKit targets the higher-level **multi-node LocalNet** lifecycle instead of the single-node sandbox that `dpm sandbox` does not cover. |
| **Existing LocalNet setup in Splice codebase** | Raw Docker Compose with 3 validators, PostgreSQL, wallet/SV/scan UIs, Canton Console | DevKit simplifies the process and abstracts the configuration and management of the container-based nodes. And it also provides additional features instead of just start, stop and cleanup operations. |

#### Canton DevKit Features

##### LocalNet Management

The existing LocalNet setup requires manually downloading Splice bundles, exporting environment variables, composing multi-flag Docker commands, and understanding Docker Compose profiles. Survey feedback shows that this setup burden is the single largest onboarding blocker. DevKit collapses this into a workflow that feels closer to modern Web3 developer tooling:

###### CLI Commands
* `canton-devkit localnet up` / `down` / `restart` / `clean` / `status` / `logs` — a single installable CLI that downloads the latest Splice LocalNet bundle, generates configs, keys, and identities, starts the Docker stack, runs health checks, and prints ready-to-use endpoints (Ledger API, Admin API, JSON API, wallet UIs) and token credentials.
* `canton-devkit localnet [up|down|...etc] --version <version>` — will manage a specific version of the LocalNet bundle.
* `canton-devkit localnet [up|down|...etc] --name <localnet-name>` — will manage a named LocalNet instance. Which is useful for running multiple LocalNet instances in parallel.
* `canton-devkit localnet snapshot/restore [--name <localnet-name>]` — snapshot management of the LocalNet instance.
* `canton-devkit localnet logs [service]` — stream or tail aggregated logs of the specified service.
* `canton-devkit localnet restart [service]` — restart the specified service.

###### Web UI Features
All features covered by CLI commands but with a user-friendly interface.

###### AI Coding Agent Integration
AI skill and command to work with the LocalNet using canton-devkit that supports Claude, Codex.

##### Observability and Monitoring

Observability was ranked as the highest-priority tooling improvement area in the survey. DevKit does **not** rebuild the observability stack from scratch. Instead, it bundles and configures a Prometheus/Grafana/Loki/Tempo stack tailored for LocalNet:

* Ensures the observability profile is enabled by default when starting LocalNet.
* Ships **Canton-specific Grafana dashboard presets** focused on DApp developers (as opposed to operator-level dashboards): transactions/sec, command completion latency, active contract counts, and per-template throughput.
* Adds a `canton-devkit metrics` subcommand that prints Grafana dashboard URLs and a concise text summary of key metrics (throughput, latency p50/p99, resource usage) for quick terminal-based checks.
* Adds health checks and service status views so developers can quickly identify connectivity issues across validators, sequencers, mediators, and supporting services.
* Documents how teams can extend or customize dashboards for their own services.
* Introduces an experimental **"cost projection" view** that estimates how an application's observed transaction patterns would translate to traffic costs on Mainnet, helping developers understand running costs and project margins before deployment.

##### Debugging and Transaction Inspection

Developers reported that current debugging relies too heavily on logs and manual inspection. To improve that experience, DevKit will provide a first layer of debugging-oriented workflows around LocalNet:

* Service-aware log aggregation and filtering via `canton-devkit localnet logs [service]`.
* Status and health reporting that helps pinpoint failures in node startup, connectivity, and transaction submission flows.
* Metrics and dashboard views that make transaction lifecycle and performance issues visible without requiring developers to assemble bespoke monitoring stacks.
* A foundation for future simulation and trace-style capabilities as the surrounding Canton tooling surface evolves.

##### Local Token Faucets & CIP-56 Developer Toolkit

This is the most differentiated feature. LocalNet already ships wallet UIs and a Registry API for token transfers, but developers still lack a CLI-driven faucet, a guided token-creation flow, and reusable Daml templates for everyday token operations. DevKit closes those gaps:

* `canton-devkit token create` — interactive "token wizard" to define new CIP-56 tokens (name, symbol, decimals, initial supply) and mint to test wallets.
* `canton-devkit token [mint | transfer | burn | balance] {token-name} {amount} [--to wallet]` — convenience commands wrapping the Ledger API / Registry API for common token operations.
* Example Daml templates and scripts for issuance, transfer, burn, and escrow flows that developers can use as starting points for their own token applications.

### 3. Architectural Alignment

The Canton DevKit removes the friction of managing local test environments so developers can focus on building their applications. This directly addresses the survey's core insight: the primary barrier to building on Canton is not protocol capability, but developer tooling maturity. The proposal aligns with the Development Fund's remit to support developer tooling and critical infrastructure as common goods, and is consistent with the milestone-based, CC-denominated funding and governance model formalized under CIP-100. Token tooling is designed to follow the Canton token standard CIP-56, making it easier for developers to test tokenized applications and integrations in a way that reflects Mainnet patterns.

### 4. Backward Compatibility

The Canton DevKit primarily targets LocalNet developer environments and does not change Canton protocol behavior, DAML semantics, or existing production deployments. Developers can continue using the Splice LocalNet Docker stack.

No backward compatibility impact.

---

## Milestones and Deliverables

### Milestone 1: LocalNet Management — CLI

- **Estimated Delivery:** Month 3  
- **Focus:** One-click LocalNet lifecycle management via CLI.  
- **Deliverables / Value Metrics:**  
  - `canton-devkit localnet up/down/restart/clean/status/logs` CLI commands with auto-generated configs, keys, identities, and printed endpoints and credentials.  
  - Version pinning (`--version`) and named instances (`--name`) for running multiple LocalNets in parallel.  
  - Snapshot and restore (`canton-devkit localnet snapshot/restore`) for saving and replaying LocalNet state.  
  - Installation and "Getting Started" guide for macOS and Linux.  
  - Internal testing plus at least one external tester validating that a new developer can go from zero to running LocalNet in under 10 minutes.
  - Measurable reduction in manual setup steps compared with the existing Splice LocalNet workflow.

### Milestone 2: Web UI, Observability, Monitoring & AI Agent Integration

- **Estimated Delivery:** Month 6  
- **Focus:** Web UI for LocalNet management, integrated observability, and AI coding agent support.  
- **Deliverables / Value Metrics:**  
  - Web UI covering all CLI features from Milestone 1 with a user-friendly interface.  
  - Bundled Prometheus/Grafana/Loki/Tempo stack enabled by default when starting LocalNet.  
  - Canton-specific Grafana dashboard presets focused on DApp developers: transactions/sec, command completion latency, active contract counts, and per-template throughput.  
  - `canton-devkit metrics` subcommand printing Grafana dashboard URLs and a concise text summary of key metrics (throughput, latency p50/p99, resource usage).  
  - Health checks and service status views covering validator, sequencer, mediator, and supporting services.  
  - AI coding agent skills and commands for Claude and Codex to interact with LocalNet via `canton-devkit`.  
  - Documentation on recommended usage, dashboard customization, and AI agent integration.  
  - (Experimental) Cost projection view estimating how observed transaction patterns translate to traffic costs on Mainnet.
  - Demonstrated improvement in issue diagnosis speed for common LocalNet failures versus log-only workflows.

### Milestone 3: Token Faucets & CIP-56 Token Tooling

- **Estimated Delivery:** Month 9  
- **Focus:** CantonCoin/CIP-56 tooling and UX polish.  
- **Deliverables / Value Metrics:**  
  - `canton-devkit token mint` CLI and Web UI minting for CIP-56 tokens on LocalNet.  
  - `canton-devkit token create` interactive token wizard to define new CIP-56 tokens (name, symbol, decimals, initial supply).  
  - `canton-devkit token transfer / burn / balance` convenience commands wrapping the Ledger API / Registry API.  
  - Cross-platform testing, UX polish across CLI and Web UI, and consolidated documentation, FAQs, and troubleshooting guides.

### Milestone 4: Maintenance and Marketing

- **Estimated Delivery:** Month 12  
- **Focus:** Stability, compatibility maintenance, and ecosystem outreach.  
- **Deliverables / Value Metrics:**  
  - Patch bugs and maintain compatibility with **newer Splice releases**.  
  - Host 2 online/offline workshops about the Canton DevKit.  
  - Publish 1 case study or blog post.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

* Delivery of the Canton DevKit project as specified for each milestone.  
* Demonstrated functionality via scripts, demos, and documentation showing:  
  * One-command LocalNet startup and teardown, including multi-instance and snapshot/restore workflows.  
  * Web UI covering the same LocalNet management features as the CLI.  
  * Working Grafana dashboards for throughput, latency, and resource usage on a sample DApp.  
  * Health and status visibility across validators, sequencers, mediators, and supporting services.  
  * AI coding agent skills successfully managing LocalNet via `canton-devkit`.  
  * CIP-56 token creation wizard, and CIP-56 token flows (mint, transfer, burn, balance) on LocalNet.  
* Documentation and knowledge transfer sufficient for developers to install, run, and extend DevKit.  
* Alignment with the stated value metrics: reduced onboarding time, streamlined LocalNet tooling, better visibility into transaction and infrastructure health, and improved ability to experiment with tokens and observability.

---

## Funding

**Total Funding Request:**

Total: **1,665,900 CC** over **12 months**.

### Payment Breakdown by Milestone

* Milestone 1 (LocalNet Management — CLI): 555,300 CC upon committee acceptance.  
* Milestone 2 (Web UI, Observability, Monitoring & AI Agent Integration): 555,300 CC upon committee acceptance.  
* Milestone 3 (Token Faucets & CIP-56 Token Tooling): 555,300 CC upon final release and acceptance.  
* Milestone 4 (Maintenance and Marketing): 0 CC upon completion (costs covered by Milestones 1–3 payments through month 12).

Funding is requested in Canton Coin, consistent with the Development Fund's CC‑denominated, milestone‑based grants model under CIP‑100.

### Volatility Stipulation

The proposed project duration is 9 months (core development, followed by 3 months of maintenance).

* The grant is denominated in a fixed amount of Canton Coin (**1,665,900 CC**) with milestone allocations as above, and will be subject to re‑evaluation at the 6‑month mark to account for material CC/USD volatility, in line with the Fund's governance guidelines.  
* If scope changes or delays requested by the Committee extend timelines beyond the original plan, remaining milestones and CC amounts can be renegotiated by mutual agreement.
* The token tooling will only support CIP-56 token standard, any other token standards released during the project will require re-negotiation of the milestone scope and funding.

---

## Co-Marketing

Upon release of major components (e.g., first public DevKit release, explorer, token tooling), the implementing entity will collaborate with the Canton Foundation on:

* Coordinated announcements highlighting DevKit as shared developer tooling for the ecosystem.  
* A case study or technical blog post explaining how DevKit simplifies LocalNet workflows and token experimentation.  
* Participation in developer‑focused promotion such as workshops, hackathons, office hours, or webinars showcasing DevKit usage.

---

## Motivation

The 2026 Canton developer experience survey makes the case for this proposal clear. Most respondents joined the ecosystem within the last 12 months, most come with prior Ethereum experience, and a large majority are building TradFi or hybrid TradFi/crypto applications. In other words, many incoming developers already expect the kind of streamlined tooling experience common in mature Web3 ecosystems.

The Splice source code for Canton already provides a LocalNet environment, but developers must manually manage Docker, configs, and observability and often build ad-hoc tools for exploring transactions, contract state, and token operations. Survey respondents identified setup and infrastructure complexity as the hardest part of getting started, and they ranked observability as the top tooling improvement area. This slows down onboarding for new teams, workshops, and hackathons, and leads to fragmented, privately maintained tooling rather than shared public goods.

By consolidating one-click LocalNet lifecycle management, observability, and token tooling into a single tool suite, the proposal significantly lowers the barrier to entry for building on Canton. It directly supports the Fund's aim to back developer tooling and critical infrastructure that act as common goods and deliver long-term value across the ecosystem.

---

## Rationale

Reducing the operational overhead of local development is a prerequisite for sustainable ecosystem growth; developer time reclaimed from infrastructure management translates directly into faster application delivery and broader adoption. The survey's central finding is that tooling maturity, not protocol capability, is the primary blocker. Delivering functionality in incremental, self-contained milestones enables early value (one-click LocalNet) and iterative refinement (metrics, debugging workflows, tokens) with clear checkpoints for the Committee.

Alternative approaches—such as separate, uncoordinated tools for observability, explorers, and token tooling—would increase maintenance burden and fragment the developer experience. A unified Canton DevKit tool suite offers a single, opinionated path that can serve as the Canton equivalent of the modern local-development frameworks developers already expect, while remaining extensible so the community can adapt it to evolving needs and future CIPs.
