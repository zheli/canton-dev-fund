## Development Fund Proposal

**Author:**  Zhe Li <zhe@bitdynamics.me>, Srikanth Yeleswarapu <srikanth@bitdynamics.me>

**Status:** Submitted  

**Created:** 2026-02-22

---

## Abstract

**Canton DevKit** is the proposed **default, one-stop local developer environment** for the Canton network, making building on the network fast, discoverable, and approachable for any developer.

It is essentially a unified toolkit designed to simplify the development, testing, and debugging of Canton Applications and DAML smart contracts. It is archived by providing developers with simple CLI commands and/or UI to interact with LocalNet, directly advancing the developer experience on the Canton ecosystem.

---

## Specification

### 1. Objective

The current official LocalNet stack creates significant friction for onboarding, workshops, hackathons, and AI-assisted coding because it requires users to manually manage:

* Docker containers
* Configuration files
* Environment variables
* Observability setup
* Ad-hoc scripts and tools for inspection and token operations

The goal is to deliver a unified DevKit. This open-source, maintained tooling will enable any developer or AI agent to manage the complete lifecycle of one or more LocalNets—using simple commands or a UI—monitor and explore activity, and easily experiment with CantonCoin and CIP-56 tokens.

### 2. Implementation Mechanics
(Explain how the solution will be implemented. Include technologies, components, workflows, and operational approach.)

The solution is delivered as an **open-source, standalone CLI application** (`canton-devkit`) with optional web UI and remote on-demand LocalNet services. It utilize docker containers and runs LocalNet using Splice node, but packages the developer experience into a single binary that requires no git clone, no Makefile knowledge, and no manual environment variable setup. It will also include other optional helper services that developers can enable or disable as needed.

#### Relationship to Existing Tooling

Canton already ships several developer tools. The DevKit is designed to complement—not replace—them:

| Existing Tool | What It Does | DevKit Relationship |
|---|---|---|
| **DPM** (`dpm`) | SDK management, Daml build/test/codegen, lightweight single-process sandbox (`dpm sandbox`), Daml Shell | DevKit does **not** replicate DPM functionality. Developers continue to use `dpm` for Daml compilation, testing, and codegen. DevKit targets the higher-level **multi-node LocalNet** lifecycle instead of the single-node sandbox that `dpm sandbox` does not cover. |
| **Existing LocalNet setup in Splice codebase** | Raw Docker Compose with 3 validators, PostgreSQL, wallet/SV/scan UIs, Canton Console | DevKit simplifies the process and abstracts the configuration and management of the container-based nodes. And it also provides additional features instead of just start, stop and cleanup operations. |

#### Canton DevKit Features

##### LocalNet Management

The existing LocalNet setup requires manually downloading Splice bundles, exporting environment variables, composing multi-flag Docker commands, and understanding Docker Compose profiles. DevKit collapses this into:

###### CLI Commands
* `canton-dev localnet up` / `down` / `restart` / `clean` / `status` / `logs` — a single installable CLI that downloads the latest Splice LocalNet bundle, generates configs, keys, and identities, starts the Docker stack, runs health checks, and prints ready-to-use endpoints (Ledger API, Admin API, JSON API, wallet UIs) and token credentials.
* `canton-dev localnet [up|down|...etc] --version <version>` — will manage a specific version of the LocalNet bundle.
* `canton-dev localnet [up|down|...etc] --name <localnet-name>` — will manage a named LocalNet instance. Which is useful for running multiple LocalNet instances in parallel.
* `canton-dev localnet snapshot/restore [--name <localnet-name>]` — snapshot management of the LocalNet instance.
* `canton-dev localnet logs [service]` — stream or tail aggregated logs of the specified service.
* `canton-dev localnet restart [service]` — restart the specified service.

###### Web UI Features
All features covered by CLI commands but with a user-friendly interface.

###### AI Coding Agent Integration
AI skill and command to work with the LocalNet using canton-devkit that supports Claude, Codex.


##### 2. Observability Convenience Layer

DevKit does **not** rebuild the observability stack from scratch. Instead, it bundles and configures a Prometheus/Grafana/Loki/Tempo stack tailored for LocalNet:

* Ensures the observability profile is enabled by default when starting LocalNet.
* Ships **Canton-specific Grafana dashboard presets** focused on DApp developers (as opposed to operator-level dashboards): transactions/sec, command completion latency, active contract counts, and per-template throughput.
* Adds a `canton-dev metrics` subcommand that prints Grafana dashboard URLs and a concise text summary of key metrics (throughput, latency p50/p99, resource usage) for quick terminal-based checks.
* Introduces a **"cost projection" view** that estimates how a DApp's observed transaction patterns would translate to CantonCoin costs on Mainnet, helping developers understand running costs and project margins before deployment. This has no existing equivalent in current tooling.
* Documents how teams can extend or customize dashboards for their own services.

##### 3. Token Faucets & CIP-56 Token Tooling (LocalNet)

This is the most differentiated feature. While LocalNet already provides wallet UIs and the Scan app exposes a Registry API for token transfers, there is currently no CLI-based faucet, no token creation wizard, and no ready-made Daml templates for common token operations. DevKit adds:

* `canton-dev faucet drip [--amount] [--to wallet]` — CLI faucet for CantonCoin and CIP-56 tokens on LocalNet, plus a lightweight web UI for the same.
* `canton-dev token create` — interactive "token wizard" to define new CIP-56 tokens (name, symbol, decimals, initial supply) and mint to test wallets.
* `canton-dev token transfer / burn / balance` — convenience commands wrapping the Ledger API / Registry API for common token operations.
* Example Daml templates and scripts for issuance, transfer, burn, and escrow flows that developers can use as starting points for their own token applications.

### 3. Architectural Alignment

The DevKit builds directly on the official Canton Quickstart and LocalNet architecture, wrapping the existing Dockerized services and profiles rather than introducing a parallel stack. It aligns with the Development Fund's remit to support developer tooling and critical infrastructure as common goods, and is consistent with the milestone‑based, CC‑denominated funding and governance model formalized under CIP‑100. Token tooling is designed to follow the emerging Canton token standard (CIP‑56) and related CIPs, making it easier for developers to test tokenized applications and integrations in a way that reflects mainnet patterns.

### 4. Backward Compatibility

The DevKit primarily targets LocalNet developer environments and does not change Canton protocol behavior, mainnet, or existing production deployments. It wraps and automates existing Quickstart/LocalNet components, so the impact on existing systems is limited to optional local tooling adoption.

If required, developers can continue using the underlying Quickstart make targets and Docker stack without DevKit.

No backward compatibility impact.

---

## Milestones and Deliverables

### Milestone 1: LocalNet CLI Foundation

- **Estimated Delivery:** Month 3  
- **Focus:** One‑click LocalNet lifecycle management.  
- **Deliverables / Value Metrics:**  
  - `canton-dev localnet up/down/restart/clean` wrapping the Quickstart Docker stack, with auto‑generated configs, keys, identities, and printed endpoints/credentials.  
  - Installation and "Getting Started" guide for macOS and Linux.  
  - Internal testing plus at least one external tester validating that a new developer can go from zero to running LocalNet in under 10 minutes.  
  - **Optionally expose an API for ease of use, contingent on user payment for infrastructure costs.**

### Milestone 2: Metrics and Monitoring

- **Estimated Delivery:** Month 6  
- **Focus:** Integrated observability for LocalNet.  
- **Deliverables / Value Metrics:**  
  - Prometheus/Grafana dashboards bundled as an optional or default DevKit profile showing throughput, latency, and resource usage for sample DApps.  
  - `canton-dev metrics` subcommand exposing dashboard URLs and a concise text summary of key metrics.  
  - Documentation on recommended usage and customization, plus example dashboards.

### Milestone 3: Token Faucets & Polish

- **Estimated Delivery:** Month 9  
- **Focus:** CantonCoin/CIP‑56 tooling and UX polish.  
- **Deliverables / Value Metrics:**  
  - LocalNet faucets for CantonCoin and CIP‑56 tokens exposed via CLI and web UI.  
  - Token creation wizard for defining new CIP‑56 tokens and minting to local wallets, plus example token flows (e.g., payment, escrow).  
  - Cross‑platform testing, UX polish across CLI and web components, and consolidated documentation, FAQs, and troubleshooting guides.

### Milestone 4: Maintenance and Marketing

- **Estimated Delivery:** Month 12  
- **Focus:** Stability, compatibility maintenance, and ecosystem outreach.  
- **Deliverables / Value Metrics:**  
  - Patch bugs and make it compatible with **newer Splice releases**.  
  - Host 2 online/offline workshop about the devkit.  
  - Publish 1 case study or blog post.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

* Delivery of the DevKit components as specified for each milestone  
* Demonstrated functionality via scripts, demos, and documentation showing:  
  * One‑command LocalNet startup and teardown.  
  * Working dashboards for throughput, latency, and resource usage on a sample DApp.  
  * Token faucet and CIP‑56 token flows on LocalNet, visible in the explorer and dashboards.  
* Documentation and knowledge transfer sufficient for developers to install, run, and extend DevKit.  
* Alignment with the stated value metrics: reduced onboarding time, standardized LocalNet tooling, and improved ability to experiment with tokens and observability.

---

## Funding

**Total Funding Request:**

Total: **1,665,900 CC** over **12 months**.

### Payment Breakdown by Milestone

* Milestone 1 (LocalNet CLI Foundation): 555,300 CC upon committee acceptance.  
* Milestone 2 (Metrics and Monitoring): 555,300 CC upon committee acceptance.  
* Milestone 3 (Token Faucets & Polish): 555,300 CC upon final release and acceptance.  
* Milestone 4 (Maintenance and Marketing): 0 CC upon completion (costs covered by Milestones 1-3 payments through month 12).

Funding is requested in Canton Coin, consistent with the Development Fund's CC‑denominated, milestone‑based grants model under CIP‑100.

### Volatility Stipulation

The proposed project duration is 9 months (core development, followed by 3 months of maintenance).

* The grant is denominated in a fixed amount of Canton Coin (**1,665,900 CC**) with milestone allocations as above, and will be subject to re‑evaluation at the 6‑month mark to account for material CC/USD volatility, in line with the Fund's governance guidelines.  
* If scope changes or delays requested by the Committee extend timelines beyond the original plan, remaining milestones and CC amounts can be renegotiated by mutual agreement.

---

## Co-Marketing

Upon release of major components (e.g., first public DevKit release, explorer, token tooling), the implementing entity will collaborate with the Canton Foundation on:

* Coordinated announcements highlighting DevKit as shared developer tooling for the ecosystem.  
* A case study or technical blog post explaining how DevKit simplifies LocalNet workflows and token experimentation.  
* Participation in developer‑focused promotion such as workshops, hackathons, office hours, or webinars showcasing DevKit usage.

---

## Motivation

Canton's Quickstart and LocalNet already provide a powerful local environment with a Super Validator and CantonCoin wallet, but developers must manually manage Docker, configs, and observability and often build ad‑hoc tools for exploring transactions, contract state, and token operations. This slows down onboarding for new teams, workshops, and hackathons, and leads to fragmented, privately maintained tooling rather than shared public goods.

By consolidating one‑click LocalNet lifecycle management, observability and token tooling into a single DevKit, the proposal significantly lowers the barrier to entry for building on Canton. It directly supports the Fund's aim to back developer tooling and critical infrastructure that act as common goods and deliver long‑term value across the ecosystem.

---

## Rationale

This approach reuses and wraps the existing Quickstart/LocalNet stack instead of reinventing it, minimizing risk and aligning DevKit with the canonical way to run Canton locally. Delivering functionality in three incremental, self‑contained milestones enables early value (one‑click LocalNet) and iterative refinement (metrics, tokens) with clear checkpoints for the Committee.

Alternative approaches—such as separate, uncoordinated tools for observability, explorers, and token faucets—would increase maintenance burden and fragment the developer experience. A unified DevKit offers a single, opinionated path that can become the de facto standard for local development, while remaining open‑source and extensible so the community can adapt it to evolving needs and future CIPs.
