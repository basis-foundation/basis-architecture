# Roadmap

This document describes the architectural and implementation trajectory of the BASIS ecosystem across a set of development phases. It is an engineering reference, not a product release plan.

**Important:** Inclusion of an item in this roadmap is not a commitment to build it, a timeline for when it will be built, or a guarantee that it will be built in the form described. The roadmap reflects current architectural intent and research direction. Items that are not yet implemented are architectural specifications or research questions — not announced features.

Status markers used throughout this document:

- **Implemented** — exists and has been validated at research or production scope
- **In architecture** — specified in architecture documents, not yet implemented as a separate component
- **Research direction** — an identified area of work with open engineering questions
- **Open question** — a known problem without a current architectural answer

---

## Current State

The BASIS ecosystem is in an early architectural maturity stage. The architecture is substantially defined. A proof-of-concept has validated core mechanisms at limited scope. Component boundaries and dependency rules are specified. The governance model is lightweight and appropriate for the current stage of development.

What does not yet exist as separately implemented and maintained components: basis-core as a standalone repository, basis-gateway, basis-console, basis-adapters as a library, basis-deploy, and basis-schemas. The architecture describes where these components belong and what they should contain. Building them is the work ahead.

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
| basis-core: isolated authorization kernel as a separate repository | **In architecture** |
| basis-schemas: shared schema and contract definitions as a separate repository | **In architecture** |
| Formal authorization request and response schema | **In architecture** |
| Audit event schema (canonical fields, semantic definitions, emission conditions) | **In architecture** |
| Policy format specification | **In architecture** |
| Enforcement point contract definition | **In architecture** |
| Failure mode contract specification (fail-closed / fail-open semantics, defined conditions) | **In architecture** |
| basis-core dependency rules enforced structurally (no upward dependencies) | **In architecture** |
| ADR process for kernel boundary decisions | **In architecture** |
| Operation-aware authorization model: conceptual expansion of DecisionRequest/DecisionResponse beyond subject/action/resource (ADR-0001) | **In architecture** |
| Operation-aware evaluation semantics: default deny, `NOT_APPLICABLE`, deny precedence, conflict resolution, missing context, and safe error handling (ADR-0002) | **In architecture** |
| Operation-aware trace and audit evidence model: trace vs. audit distinction, evidence lifecycle, redaction rules, reason codes, and evidence assembly ownership (ADR-0003) | **In architecture** |
| Operation-aware policy bundle and rule model: bundle scope, rule effects and match criteria, conditions, combining semantics, validation, and reason codes (ADR-0004) | **In architecture** |
| Compatibility versioning strategy for basis-schemas | **Research direction** |

**Design constraint:** basis-core must not acquire dependencies on basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, cloud SDKs, identity providers, database runtimes, UI frameworks, or protocol stacks. This constraint is a governance requirement, not a coding convention. See [`GOVERNANCE.md`](GOVERNANCE.md) for the basis-core boundary protection policy.

---

## Phase 3 — Core Services Distribution

This phase assembles the full BASIS Core Services Distribution: the set of components that together provide a complete, deployable identity-aware authorization system.

| Item | Status |
| - | - |
| basis-gateway: API and runtime wrapper around basis-core | **In architecture** |
| basis-console: operator and administrator UI | **In architecture** |
| basis-adapters: BACnet adapter | **In architecture** |
| basis-adapters: Modbus TCP adapter (building on PoC implementation) | **In architecture** |
| basis-adapters: MQTT adapter (building on PoC implementation) | **In architecture** |
| basis-adapters: shared AdapterBase interface and normalization contracts | **In architecture** |
| basis-deploy: container definitions and configuration tooling | **In architecture** |
| basis-deploy: deployment validation | **In architecture** |
| End-to-end deployment: basis-core + gateway + console + adapters + deploy from a single basis-deploy configuration | **In architecture** |
| Policy authoring interface (basic) | **In architecture** |
| Audit query interface (basic) | **In architecture** |
| Policy distribution from gateway to enforcement points | **Research direction** |
| Adapter correctness testing framework | **Research direction** |
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
| Compatibility and versioning strategy for stable interface surfaces | **Research direction** |
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
