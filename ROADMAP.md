# Roadmap

This document describes the architectural and implementation trajectory of the BASIS ecosystem across a set of development phases. It is an engineering reference, not a product release plan.

**Important:** Inclusion of an item in this roadmap is not a commitment to build it, a timeline for when it will be built, or a guarantee that it will be built in the form described. The roadmap reflects current architectural intent and research direction. Items that are not yet implemented are architectural specifications or research questions — not announced features.

Status markers used throughout this document:

- **Implemented** — exists and has been validated at research or production scope
- **Completed** — architecture work (a document, ADR, or specification) is finished
- **Released** — an implementation repository, or a set of published contracts, is versioned and publicly available
- **In progress** — active implementation work is underway in a separate repository, not yet released in the form described
- **Planned** — an implementation program is defined (a roadmap or plan document exists) but the work itself has not started or is only beginning
- **In architecture** — specified in architecture documents, not yet implemented as a separate component
- **Research direction** — an identified area of work with open engineering questions
- **Open question** — a known problem without a current architectural answer
- **Deferred** — intentionally not scheduled at this time
- **Architecture decision required** — implementation is blocked pending a decision that must be made in `basis-architecture`, not resolved unilaterally by an implementation repository

---

## Current State

The BASIS ecosystem has moved from an architecture-only research project into a set of separately maintained implementation repositories that consume published, versioned contracts. The architecture remains the authority for what those repositories build; the repositories are now doing the building.

**Completed or established:**

- `basis-core` exists as a separate public repository; `v0.1.0` is released.
- `basis-gateway` exists as a separate public repository and is released (`v0.1.0`); active development continues on top of that release.
- `basis-adapters` exists as a separate public repository, is released (`v0.1.0`), and supports nine OT protocols and platforms (REST, BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, and Niagara), each normalization-complete.
- `basis-console` exists as a separate public repository and is released (`v0.1.1`).
- `basis-schemas` exists as a separate public repository and is released (`v0.2.0`), publishing 20 contracts — the six first-wave contracts from `v0.1.0` plus fourteen operation-aware contracts published across ADR-0005's schema readiness plan — and five canonical compatibility scenarios connecting them.
- The operation-aware architecture is defined: ADR-0001 through ADR-0005 (authorization model, evaluation semantics, trace/audit evidence, policy bundle/rule model, schema readiness plan) are complete.
- The operation-aware contract suite is published, fulfilling the schema readiness plan above.
- Canonical compatibility scenarios exist, connecting the operation-aware request, policy, trace, response, and audit contracts under one executable set of examples.
- Architecture and compatibility governance are established: this repository's ADR process and compatibility-philosophy documentation, plus repository-level compatibility policies now in place in `basis-schemas` (`docs/contract-governance.md`) and `basis-core` (`docs/breaking-change-discipline.md`).
- The `basis-core` v0.2.0 implementation roadmap is defined: a 44-PR, 15-milestone plan describing how the kernel will consume the published operation-aware contracts without breaking `v0.1.0` compatibility.

**In progress:**

- Incremental `basis-core` v0.2.0 implementation, against the roadmap above. No operation-aware evaluation behavior has shipped yet; the `v0.1.0` kernel remains the released, supported surface, unchanged.
- `basis-identity`: active implementation of the identity engine and federation boundary (OIDC discovery, JWKS, token verification, login/callback composition, sessions). Release preparation is underway, but `basis-identity` has not yet been tagged or published.

**Not yet implemented:**

- `basis-deploy`: deployment and distribution tooling. No repository exists yet.
- The production engineering and ecosystem-maturity work described in Phases 4 and 5 below.
- Condition-operator semantics for the operation-aware policy model — a clarification proposal now exists (see the **In architecture** item in Phase 2); it is not yet reviewed or approved, and `basis-core` v0.2.0 condition-evaluation code (Milestone 7, PRs 22-23) remains blocked until it is.

This roadmap does not claim that runtime operation-aware authorization support exists merely because the contracts are published. `basis-schemas` v0.2.0 publishes the shapes; `basis-core` v0.2.0 has not yet implemented evaluation against them.

---

## Phase 1 — Architecture Definition and Proof-of-Concept

This phase is substantially complete. Its purpose was to define the architecture with enough precision to evaluate it and to validate that core mechanisms are implementable.

| Item | Status |
| - | - |
| White paper: identity-aware authorization for OT environments (Sections 01–10) | **Implemented** |
| Architecture principles (15 principles across authorization, identity, resilience, and OT constraints) | **Implemented** |
| Trust boundary model and zone taxonomy | **Implemented** |
| Authorization flow model (subject-resource-action, enforcement points, policy engine, audit pipeline) | **Implemented** |
| Threat model (nine categories with architectural mitigations and residual risk assessment) | **Implemented** |
| basis-poc: proof-of-concept validating identity propagation, policy evaluation, enforcement, protocol adapters, audit | **Implemented** |
| MQTT and Modbus TCP adapter implementations (research scope, simulated resources) | **Implemented** |
| Dual-record audit semantics (authorization event + dispatch event) | **Implemented** |
| Diagram standards, writing standards, terminology guidelines | **Implemented** |
| Glossary with OT, IAM, and ecosystem terminology | **Implemented** |
| Ecosystem structure document (Foundation, distribution, BASAuth, component dependencies) | **Implemented** |

**Validated by the proof-of-concept:** Identity propagation from authentication through policy evaluation to audit record; structural enforcement at an API boundary; protocol-agnostic authorization through the adapter abstraction; dual-event audit structure; authenticated telemetry delivery.

**Not validated by the proof-of-concept:** Distributed policy evaluation, local policy cache operation, production-scale OT resource coverage, operational latency under realistic command volumes, high availability, real industrial protocol handling, safety constraint interaction, or production security hardening. Section 06 of the white paper documents these gaps in detail.

---

## Phase 2 — Kernel Extraction and Contract Stabilization

This phase addresses the step from validated research implementation to a separately maintained, stable authorization kernel. The primary work is extracting the evaluation semantics from the monolithic PoC into a component that can be independently versioned, tested, and depended upon.

| Item | Status |
| - | - |
| basis-core: isolated authorization kernel as a separate repository | **Released** (`v0.1.0`) |
| basis-schemas: shared schema and contract definitions as a separate repository | **Released** (`v0.2.0`) |
| Formal authorization request and response schema | **Released** (first-wave `decision-request`/`decision-response`; second-wave `operation-aware-decision-request`/`operation-aware-decision-response`) |
| Audit event schema (canonical fields, semantic definitions, emission conditions) | **Released** (first-wave `audit-event`; second-wave `audit-evidence`, `gateway-audit-event`) |
| Policy format specification | **Released** (`policy-condition`, `policy-rule`, `policy-bundle` contracts published) — condition-operator evaluation semantics are a separate, still-open item below |
| Enforcement point contract definition | **Released** (`basis-core` `EnforcementPoint`) |
| Failure mode contract specification (fail-closed / fail-open semantics, defined conditions) | **Released** |
| basis-core dependency rules enforced structurally (no upward dependencies) | **Released** |
| ADR process for kernel boundary decisions | **Completed** |
| Operation-aware authorization model: conceptual expansion of DecisionRequest/DecisionResponse beyond subject/action/resource (ADR-0001) | **Completed** |
| Operation-aware evaluation semantics: default deny, `NOT_APPLICABLE`, deny precedence, conflict resolution, missing context, and safe error handling (ADR-0002) | **Completed** |
| Operation-aware trace and audit evidence model: trace vs. audit distinction, evidence lifecycle, redaction rules, reason codes, and evidence assembly ownership (ADR-0003) | **Completed** |
| Operation-aware policy bundle and rule model: bundle scope, rule effects and match criteria, conditions, combining semantics, validation, and reason codes (ADR-0004) | **Completed** |
| Operation-aware schema readiness and migration plan: contract surfaces, publication order, dependency relationships, compatibility rules, and ownership for the basis-schemas expansion (ADR-0005) | **Completed** |
| Compatibility versioning strategy for basis-schemas | **Released** (`basis-schemas` `docs/contract-governance.md`: experimental/stable lifecycle states) |
| Operation-aware contract publication: fourteen contracts published across ADR-0005's plan | **Released** (`basis-schemas` `v0.2.0`) |
| Five canonical compatibility scenarios connecting operation-aware request, policy, trace, response, and audit contracts | **Released** (`basis-schemas` `v0.2.0`) |
| `basis-core` v0.2.0 operation-aware implementation program (44-PR, 15-milestone plan) | **Planned** |
| `basis-core` v0.2.0 operation-aware evaluation, implemented against the published contracts | **In progress** |
| Condition-operator semantics: what a `policy-condition`'s `operator` field evaluates at runtime | **In architecture** (clarification proposed, not yet reviewed or approved — see [`docs/architecture/condition-operator-semantics.md`](docs/architecture/condition-operator-semantics.md)) |

**Design constraint:** basis-core must not acquire dependencies on basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, cloud SDKs, identity providers, database runtimes, UI frameworks, or protocol stacks. This constraint is a governance requirement, not a coding convention. See [`GOVERNANCE.md`](GOVERNANCE.md) for the basis-core boundary protection policy.

---

## Phase 3 — Core Services Distribution

This phase assembles the full BASIS Core Services Distribution: the set of components that together provide a complete, deployable identity-aware authorization system.

| Item | Status |
| - | - |
| basis-gateway: API and runtime wrapper around basis-core | **Released** (`v0.1.0`); active development continues |
| basis-console: operator and administrator UI | **Released** (`v0.1.1`) |
| basis-adapters: BACnet adapter | **Released** |
| basis-adapters: Modbus TCP adapter (building on PoC implementation) | **Released** |
| basis-adapters: MQTT adapter (building on PoC implementation) | **Released** |
| basis-adapters: shared AdapterBase interface and normalization contracts | **Released** |
| basis-adapters: additional protocol/platform coverage beyond the original three (REST, OPC UA, DNP3, IEC 61850, KNX, Niagara) | **Released** — nine protocols/platforms normalization-complete in total; per `basis-adapters`' own roadmap, no further protocols are currently planned |
| basis-identity: identity engine and federation boundary | **In progress** — implementation active, not yet tagged or released |
| basis-deploy: container definitions and configuration tooling | **In architecture** |
| basis-deploy: deployment validation | **In architecture** |
| End-to-end deployment: basis-core + gateway + console + adapters + deploy from a single basis-deploy configuration | **In architecture** |
| Policy authoring interface (basic) | **In architecture** |
| Audit query interface (basic) | **In architecture** |
| Policy distribution from gateway to enforcement points | **Research direction** |
| Adapter correctness testing framework | **Released** (`basis-adapters` cross-protocol contract tests validate the shared normalized-request shape) |
| Local policy cache in gateway/enforcement points | **Research direction** |

**Architectural requirement for this phase:** Each component must conform to the dependency rules established in Phase 2. basis-gateway depends on basis-core. basis-adapters depends on basis-core for contracts. basis-console depends on basis-gateway. No component introduces upward dependencies into basis-core.

---

## Phase 4 — Production Engineering and Resilience

This phase addresses the operational engineering challenges documented in Section 07 of the white paper. These are the hardest problems in the architecture — not because they are technically novel, but because they require operational validation under realistic OT conditions.

| Item | Status |
| - | - |
| Local policy cache: time-bounded cached policy at enforcement points | **Research direction** |
| Offline enforcement: defined behavior when basis-gateway cannot reach basis-core | **Research direction** |
| Policy distribution: defined format, delivery mechanism, and consistency model | **Research direction** |
| Cache invalidation: defined semantics and behavior on staleness limit expiration | **Research direction** |
| Credential revocation propagation: mechanism for forced cache invalidation | **Research direction** |
| High-availability topology for basis-core and basis-gateway | **Research direction** |
| Compatibility and versioning strategy for stable interface surfaces at production/high-availability scale (distinct from the contract-level versioning already established for `basis-schemas`, tracked in Phase 2) | **Research direction** |
| Enforcement point health monitoring and policy synchronization monitoring | **Research direction** |
| Audit delivery guarantees: local buffering, overflow policy, delivery ordering | **Research direction** |
| Degraded operation mode definition (fail-open, fail-closed, read-only) | **Research direction** |
| Certificate and credential lifecycle management tooling | **Research direction** |
| Staleness limit configuration and operational guidance | **Open question** |
| Fail behavior governance: who owns the fail-open/fail-closed decision per site | **Open question** |
| Safety-authorization interaction: how authorization infrastructure is accounted for in safety analysis | **Open question** |

**Note:** The production challenges in this phase are not engineering problems with obvious technical solutions. Many of them involve operational tradeoffs that must be negotiated per-deployment — between security and continuity, between policy currency and operational independence, between audit completeness and delivery reliability. The architecture provides a framework for making those tradeoffs; it does not resolve them. Section 07 of the white paper examines each of these areas in detail.

---

## Phase 5 — Ecosystem Maturity and Operational Validation

This phase addresses the longer-term engineering and organizational questions that emerge after the distribution is deployable and early operational experience has been accumulated.

| Item | Status |
| - | - |
| Production reference deployment patterns (per environment class) | **Open question** |
| Operational validation of authorization latency under realistic OT command volumes | **Open question** |
| Identity federation: cross-organizational identity, revocation coordination | **Open question** |
| Distributed policy coordination: hierarchical or federated policy distribution topologies | **Open question** |
| Policy validation and explainability tooling | **Open question** |
| Adapter certification process for specific device manufacturers and firmware versions | **Open question** |
| Device identity enrollment and lifecycle tooling for OT hardware | **Open question** |
| Ecosystem interoperability: standards engagement, cross-vendor normalization | **Open question** |
| Open governance maturity: formal Foundation structure, membership, RFC process | **Open question** |
| Cross-sector applicability: industrial process control, power utilities, water treatment | **Open question** |

**Note on the open questions in this phase:** Several of these items represent fundamental tensions that the architecture cannot resolve and that future engineering work will need to address. Identity federation, distributed policy coordination, and safety-authorization interaction are well-characterized as problems and underspecified as solutions. Section 09 of the white paper ("Future Direction") analyzes these open engineering problems and the organizational and standards-level constraints that make them difficult. The expectation that a clean architectural solution exists for all of them is probably not well-founded.

---

## What This Roadmap Does Not Say

This roadmap does not specify dates, release schedules, or version numbers. It does not commit to building any item in Phase 3, 4, or 5. It does not imply that items listed as "In architecture" or "Research direction" will be implemented in the form or sequence described here.

The roadmap reflects current architectural intent. That intent will be updated as implementation experience, operational feedback, and engineering analysis surface requirements or constraints that the current architecture does not fully anticipate. Changes to architectural intent that affect stable component boundaries or established contracts should be documented through the ADR process described in [`GOVERNANCE.md`](GOVERNANCE.md).

The architecture is a starting point for the implementation work ahead, not a finished specification of it.
