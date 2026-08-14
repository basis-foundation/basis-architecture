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

The BASIS ecosystem has moved from an architecture-only research project into a set of separately maintained implementation repositories that consume published, versioned contracts. The architecture remains the authority for what those repositories build; the repositories are now doing the building. `basis-core`'s operation-aware implementation program is complete and released; downstream adoption of that released kernel surface is underway and has already reached three components — `basis-gateway` and `basis-console` are both released with operation-aware integration, and `basis-adapters` `v0.2.0` is released with adapter-side deterministic evidence-material construction (ADR-0007 Stage 1). See the **Downstream Rollout Sequence** below for what remains.

**Completed or established:**

- `basis-core` exists as a separate public repository; `v0.1.0`, `v0.2.0`, and `v0.2.1` are all released. `v0.2.0` is additive to `v0.1.0`; `v0.2.1` is an additive correction to `v0.2.0` (a public-factory construction path, `OperationAwareEnforcementPoint.for_bundle()`, so downstream consumers no longer import `basis_core.evaluation.*` directly — no policy, evaluation, aggregation, trace, evidence, disposition, or failure semantics changed). The `v0.1.0` public surface remains fully supported and unchanged throughout.
- `basis-gateway` exists as a separate public repository; `v0.1.0` and `v0.2.0` are both released. `v0.2.0` adds an operation-aware enforcement path (`POST /v1/evaluate/operation-aware`, feature-flagged via `OPERATION_AWARE_ENABLED` and disabled by default) alongside the unchanged, always-registered `v0.1` path (`POST /v1/evaluate`). See **Completed Operation-Aware Evaluation and Enforcement Boundary** below.
- `basis-adapters` exists as a separate public repository; `v0.1.0` and `v0.2.0` are both released, and support nine OT protocols and platforms (REST, BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, and Niagara), each normalization-complete. `v0.2.0` adds adapter-side deterministic evidence-material construction, RFC 8785 canonicalization, and digest generation (ADR-0007 Stage 1) alongside the unchanged nine-protocol normalization surface. Adapters still do not authenticate a producer, persist evidence, mint a final evidence reference, submit to the gateway, or execute protocol operations — the remaining gap is operation-producer identity and integration, not adapter evidence-material construction. See **Next Producer and Execution-Evidence Boundary** below.
- `basis-console` exists as a separate public repository; `v0.1.0`, `v0.1.1`, and `v0.2.0` are all released. `v0.2.0` adds an operation-aware evaluation contract to the Decision Simulator, consuming `basis-gateway`'s operation-aware path through a typed client with Operator/Training runtime parity; the legacy `POST /v1/evaluate` workflow is unchanged and remains the default. See **Completed Operation-Aware Evaluation and Enforcement Boundary** below and [`docs/roadmaps/operator-and-training-experience.md`](docs/roadmaps/operator-and-training-experience.md) for the deferred direction beyond this release.
- `basis-schemas` exists as a separate public repository and is released (`v0.2.0`), publishing 20 contracts — the six first-wave contracts from `v0.1.0` plus fourteen operation-aware contracts published across ADR-0005's schema readiness plan — and five canonical compatibility scenarios connecting them. `v0.2.0` introduced the operation-aware contract suite; `v0.2.1` and `v0.2.2` are subsequent corrective releases that fix compatibility-fixture defects without changing any schema, field, reason-code vocabulary, or authorization outcome. `basis-core` conforms to the corrected `v0.2.2` snapshot.
- The operation-aware architecture is defined: ADR-0001 through ADR-0006 (authorization model, evaluation semantics, trace/audit evidence, policy bundle/rule model, schema readiness plan, and the evaluation orchestration layer) are complete.
- The operation-aware contract suite is published, fulfilling the schema readiness plan above.
- Canonical compatibility scenarios exist, connecting the operation-aware request, policy, trace, response, and audit contracts under one executable set of examples.
- Architecture and compatibility governance are established: this repository's ADR process and compatibility-philosophy documentation, plus repository-level compatibility policies now in place in `basis-schemas` (`docs/contract-governance.md`) and `basis-core` (`docs/breaking-change-discipline.md`).
- The `basis-core` v0.2.0 operation-aware implementation program — a 44-PR, 15-milestone plan — is complete. Operation-aware typed models; deterministic policy validation and evaluation; bundle applicability and candidate selection; selector and condition evaluation; deny precedence; default deny; a `NOT_APPLICABLE` outcome kept distinct from both; deterministic trace assembly; bounded `AuditEvidence`; and the separate, fail-closed `OperationAwareEnforcementPoint` are all implemented. Canonical conformance passes through the real `OperationAwareEnforcementPoint.evaluate()` path against the vendored `basis-schemas` `v0.2.2` snapshot. `v0.1.0` compatibility is preserved and unmodified. Release-readiness review and packaging validation are complete.
- Condition-operator semantics — what a `policy-condition`'s `operator` field evaluates at runtime — are approved and implemented: the closed, ten-operator registry defined in [`docs/architecture/condition-operator-semantics.md`](docs/architecture/condition-operator-semantics.md) is the operator set `basis-core`'s condition evaluator implements.
- The pure evaluation orchestration layer described by ADR-0006 is implemented: `basis-core`'s `evaluation` kernel subpackage (`trace_assembly`, `engine`, `response_assembly`, at `src/basis_core/evaluation/`) sequences policy-owned evaluation facts and audit-owned trace models without weakening `policy` ↔ `audit` isolation, per [`docs/architecture/operation-aware-evaluation-orchestration.md`](docs/architecture/operation-aware-evaluation-orchestration.md). ADR-0006 itself remains recorded as `Proposed` — implementation is not, by itself, formal ADR acceptance under this repository's governance process; see [`docs/adr/0006-evaluation-orchestration-layer.md`](docs/adr/0006-evaluation-orchestration-layer.md) and [`docs/adr/README.md`](docs/adr/README.md#lifecycle-states).
- The `basis-gateway` v0.2.0 operation-aware integration is complete for its approved, feature-flagged scope: operation-aware request ingestion and composition; operation-producer trust classification (`OPERATION_PRODUCER_SUBJECT_IDS`, exact-match, empty by default); provenance-gated rejection of producer-only fields from untrusted callers; invocation of the released `OperationAwareEnforcementPoint` (public `basis-core` integration only); a startup semantic preflight; exhaustive kernel-outcome-to-HTTP classification; `GatewayAuditEvent` recorded beside the kernel's unmodified `AuditEvidence`; and dedicated readiness components. The `v0.1` path is unaffected. Production adoption (enabling the feature flag) has not been established as a deployment default and should be preceded by environment-specific validation, per `basis-gateway`'s own documentation.
- The `basis-console` v0.2.0 operation-aware integration is complete for its approved scope: a second, explicit evaluation contract on the Decision Simulator; strict typed response parsing distinguishing governed results from generic/client errors; correlation-ID integrity checking; a shared, provenance-labeled presentation model; and identical rendering across Operator and Training modes, with Training mode adding only explanatory content. The legacy evaluation workflow is unchanged and remains the default.
- `basis-identity` exists as a separate public repository and is released (`v0.1.0`): the identity engine and federation boundary (OIDC discovery, JWKS, token verification, login/callback composition, sessions, BASIS-local token issuance and signing) is implemented and covered by tests for its `v0.1.0` scope. It has not yet integrated against the operation-aware surface — see **Next Producer and Execution-Evidence Boundary** below.

**Next implementation phase:**

- Trusted operation-producer identity and integration — Planned, not yet started. `basis-adapters` `v0.2.0` has already implemented the adapter-owned half of adapter evidence alignment (deterministic evidence-material construction, canonicalization, and digest generation, per ADR-0007 Stage 1); what remains not yet started is the operation-producer runtime itself — producer authentication, `reference_id` minting, final `AdapterEvidenceReference` assembly, and gateway submission — none of which any component in the ecosystem performs today. See **Next Producer and Execution-Evidence Boundary** below.
- `basis-identity` evidence alignment against the operation-aware surface — Planned, not yet started, independent of `basis-identity`'s own `v0.1.0` release.

**Not yet implemented:**

- `basis-deploy`: deployment and distribution tooling. No repository exists yet.
- The production engineering and ecosystem-maturity work described in Phases 4 and 5 below.
- The operation-producer runtime (authentication, `reference_id` minting, final `AdapterEvidenceReference` assembly, and gateway submission), `basis-identity` evidence alignment, `basis-deploy` packaging, and end-to-end validation — see the **Downstream Rollout Sequence** below. None of this remaining downstream work has begun; `basis-adapters`' own evidence-material construction (ADR-0007 Stage 1) is the one piece of the adapter-evidence gap that has begun and is complete, per the bullet above.

This roadmap does not claim that ecosystem-wide operation-aware authorization support is complete merely because `basis-core`, `basis-gateway`, `basis-console`, and `basis-adapters` have each released operation-aware integration. `basis-gateway`'s operation-aware path is feature-flagged and disabled by default; `basis-adapters`' `v0.2.0` integration is limited to adapter-owned evidence-material construction (ADR-0007 Stage 1) — it does not authenticate a producer, submit to the gateway, or execute protocol operations; `basis-identity` has not yet integrated against the operation-aware surface at all; and `basis-deploy` does not yet exist.

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
| Policy format specification | **Released** (`policy-condition`, `policy-rule`, `policy-bundle` contracts published) — condition-operator evaluation semantics are addressed separately below |
| Enforcement point contract definition | **Released** (`basis-core` `EnforcementPoint`) |
| Failure mode contract specification (fail-closed / fail-open semantics, defined conditions) | **Released** |
| basis-core dependency rules enforced structurally (no upward dependencies) | **Released** |
| ADR process for kernel boundary decisions | **Completed** |
| Operation-aware authorization model: conceptual expansion of DecisionRequest/DecisionResponse beyond subject/action/resource (ADR-0001) | **Completed** |
| Operation-aware evaluation semantics: default deny, `NOT_APPLICABLE`, deny precedence, conflict resolution, missing context, and safe error handling (ADR-0002) | **Completed** |
| Operation-aware trace and audit evidence model: trace vs. audit distinction, evidence lifecycle, redaction rules, reason codes, and evidence assembly ownership (ADR-0003) | **Completed** |
| Operation-aware policy bundle and rule model: bundle scope, rule effects and match criteria, conditions, combining semantics, validation, and reason codes (ADR-0004) | **Completed** |
| Operation-aware schema readiness and migration plan: contract surfaces, publication order, dependency relationships, compatibility rules, and ownership for the basis-schemas expansion (ADR-0005) | **Completed** |
| Pure evaluation orchestration layer: `evaluation` kernel subpackage resolving the policy-owned-facts / audit-owned-trace composition conflict without weakening `policy` ↔ `audit` isolation (ADR-0006) | **Implemented** — implemented by `basis-core` `v0.2.0` at `src/basis_core/evaluation/` (`trace_assembly`, `engine`, `response_assembly`) — see [`docs/architecture/operation-aware-evaluation-orchestration.md`](docs/architecture/operation-aware-evaluation-orchestration.md); ADR-0006 itself remains `Proposed` (not formally accepted) per [`docs/adr/0006-evaluation-orchestration-layer.md`](docs/adr/0006-evaluation-orchestration-layer.md) |
| Compatibility versioning strategy for basis-schemas | **Released** (`basis-schemas` `docs/contract-governance.md`: experimental/stable lifecycle states) |
| Operation-aware contract publication: fourteen contracts published across ADR-0005's plan | **Released** (`basis-schemas` `v0.2.0`) |
| Five canonical compatibility scenarios connecting operation-aware request, policy, trace, response, and audit contracts | **Released** (`basis-schemas` `v0.2.0`; corrected by `v0.2.1` and `v0.2.2`) |
| `basis-core` v0.2.0 operation-aware implementation program (44-PR, 15-milestone plan) | **Completed** — all 44 PRs and 15 milestones landed; see `basis-core`'s `docs/v0.2-readiness-review.md` |
| `basis-core` v0.2.0 operation-aware evaluation, implemented against the published contracts | **Released** (`v0.2.0`) — canonical conformance passes through the real `OperationAwareEnforcementPoint.evaluate()` path against the vendored `basis-schemas` `v0.2.2` snapshot |
| Condition-operator semantics: what a `policy-condition`'s `operator` field evaluates at runtime | **Completed and implemented** — the clarification is approved, and its ten-operator registry (`equals`, `not_equals`, `in`, `not_in`, `greater_than`, `greater_than_or_equal`, `less_than`, `less_than_or_equal`, `exists`, `not_exists`) is implemented by `basis-core` `v0.2.0` — see [`docs/architecture/condition-operator-semantics.md`](docs/architecture/condition-operator-semantics.md) |

**Design constraint:** basis-core must not acquire dependencies on basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, cloud SDKs, identity providers, database runtimes, UI frameworks, or protocol stacks. This constraint is a governance requirement, not a coding convention. See [`GOVERNANCE.md`](GOVERNANCE.md) for the basis-core boundary protection policy.

---

## Downstream Rollout Sequence

`basis-core` v0.2.0's release completes the kernel implementation phase described above. What follows is a governed, incremental adoption sequence, not a simultaneous rewrite of every repository in the distribution:

```text
basis-schemas operation-aware contracts            Released
        ↓
basis-core operation-aware evaluation              Released
        ↓
basis-gateway operation-aware enforcement          Released (feature-flagged, disabled by default)
        ↓
basis-console operation-aware consumption          Released
        ↓
basis-adapters evidence-material construction      Released (ADR-0007 Stage 1)
        ↓
trusted operation-producer identity and runtime    Next remaining integration boundary
        ↓
basis-identity evidence alignment                  Future
        ↓
basis-deploy packaging                             Future
        ↓
basis-demo end-to-end validation                   Future
        ↓
real OT integration validation                     Future
```

The first five stages are released, including `basis-adapters`' evidence-material construction (ADR-0007 Stage 1). Trusted operation-producer identity and the operation-producer runtime are named as the next remaining integration boundary rather than a committed sequence position — architecture has not yet fixed whether the producer runtime and `basis-identity` evidence alignment must proceed in strict order, or can proceed in parallel once each has its own implementation plan. See **Next Producer and Execution-Evidence Boundary** below.

Governing principles for this sequence:

- Downstream repositories adopt the released kernel surface incrementally, one repository at a time, rather than all repositories changing at once.
- `v0.1.0` compatibility is preserved wherever a downstream repository's own compatibility commitments require it — and has been, in both the `basis-gateway` and `basis-console` `v0.2.0` releases.
- Gateway integration precedes console UI expansion: the console had nothing operation-aware to explain to operators until the gateway could produce operation-aware decisions and evidence. This has now occurred — both are released.
- `basis-adapters` and `basis-identity` should be changed based on real integration needs that gateway work surfaces, not speculative taxonomy work performed ahead of it.
- `basis-schemas` changes only when downstream integration finds a genuine contract defect or a missing governed concept, not to anticipate hypothetical future needs.
- Implementation repositories must not invent architecture independently; changes to established contracts or component boundaries still require the ADR process described in [`GOVERNANCE.md`](GOVERNANCE.md).

This sequence does not assign PR counts, milestones, or delivery dates to any remaining downstream phase. Those belong to each repository's own implementation plan, established when that phase of work actually begins.

### Completed Operation-Aware Evaluation and Enforcement Boundary

With `basis-core`, `basis-gateway`, and `basis-console` each released with operation-aware integration, the established, now-implemented responsibility split is:

- **`basis-core` decides.** It evaluates operation-aware `DecisionRequest`s and produces `OperationAwareDecisionResponse`, `EvaluationTrace`, and bounded `AuditEvidence` through the released `OperationAwareEnforcementPoint`.
- **`basis-gateway` authenticates, composes, invokes, enforces, and records enforcement-boundary facts.** It classifies trusted operation producers, composes governed operations and resources, invokes the kernel, classifies the kernel's outcome to an HTTP response, and combines kernel `AuditEvidence` with gateway-owned authentication, route, transport, correlation, response, timeout, and enforcement facts into `GatewayAuditEvent`, recorded beside — never nested inside — the kernel's unmodified evidence.
- **`basis-console` consumes and presents.** It submits through the gateway, renders the typed result the gateway returns, and never re-evaluates, reclassifies, or fabricates evidence; Operator and Training modes render the identical result.

This is a restatement of the existing ownership model established in ADR-0003 and [`docs/architecture/operation-aware-trace-audit-evidence.md`](docs/architecture/operation-aware-trace-audit-evidence.md), now implemented rather than planned. None of the three components rewrites a kernel decision, converts a kernel `NOT_APPLICABLE` outcome into a kernel `DENY`, invents kernel trace evidence, attributes gateway-produced facts to the kernel, or treats `GatewayAuditEvent` as a kernel-owned artifact. This boundary is released and implemented for its approved scope; it is not a claim that the `basis-gateway` operation-aware path is enabled by default in any given deployment, or that production adoption has been validated — see `basis-gateway`'s own documentation for its feature-flag and current-limitations guidance.

### Next Producer and Execution-Evidence Boundary

The remaining architectural direction, not yet implemented, includes: trusted operation-producer identity; adapter-to-gateway normalized operation intent; protocol-specific provenance; safe request handoff; execution-result evidence; correlation between authorization and execution; and clear producer, gateway, kernel, and adapter ownership of each of these facts. Future schema work in `basis-schemas` should follow only after architecture defines the governed contracts these capabilities require — it is not scheduled here.

[`docs/architecture/operation-producer-and-execution-boundary.md`](docs/architecture/operation-producer-and-execution-boundary.md) is the first architecture document addressing this boundary directly: it names the logical roles between adapter normalization and gateway ingestion (the operation-producer runtime) and between gateway enforcement and OT protocol execution (the protocol executor, the execution-evidence producer), states the trust-establishment rules that must govern them, and proposes a ten-stage, decision-gated implementation sequence. It is architecture-planning only — it does not implement anything, does not select a producer-authentication mechanism, does not add a schema, and does not decide whether the operation-producer runtime requires a new repository. That decision, and the authentication-mechanism decision, remain open gates recorded in that document's own **Open Questions Deferred** section.

That document's Stage 2 decision gate — whether `basis-adapters` needs additive surface to construct adapter evidence — is answered: `basis-adapters`' merged handoff-alignment plan found the answer mixed by field, and [`docs/architecture/adapter-evidence-construction-semantics.md`](docs/architecture/adapter-evidence-construction-semantics.md), adopted by [ADR-0007](docs/adr/0007-adapter-evidence-construction.md), now defines what adapter evidence material is (a fixed `basis-adapter-evidence-v1` profile), how it is canonicalized (RFC 8785, adopted exactly, not left open), which component computes its digest, which component mints the evidence reference identifier, and which component assembles the final `AdapterEvidenceReference`. It assigns material construction, canonicalization, and digest computation to `basis-adapters`, and reference assembly to the still-unimplemented operation-producer runtime. The architecture document itself is architecture-planning only — it does not implement anything and does not modify `basis-schemas` — but the assignment it records has since been implemented: `basis-adapters` `v0.2.0` released `construct_adapter_evidence()`, completing ADR-0007 Stage 1. Reference assembly, producer authentication, and gateway submission remain unimplemented, per the operation-producer-runtime gap described above.

This roadmap does not define final schemas for this boundary, does not choose a transport-signing technology for producer trust, does not implement adapter behavior, and does not claim execution is currently supported anywhere in the ecosystem. `basis-gateway`'s existing operation-producer trust classification (`OPERATION_PRODUCER_SUBJECT_IDS`) is a configuration-driven subject-ID allowlist for the gateway's own callers; it is not the adapter-to-gateway trusted-producer contract this section describes, which remains open. Existing architecture does not yet establish enough authority to call this a single, ordered "next phase" — it is the next major boundary requiring architecture-first planning, per the roadmaps referenced under **Identity-to-Operation Contract and Interoperability** and **Post-Authentication Identity Activity, Correlation, and Detection** below.

**Producer workload authentication and gateway admission — architecture proposed, not implemented.** [ADR-0008](docs/adr/0008-producer-workload-authentication-and-admission.md) answers the authentication-mechanism decision gate named above: it selects mutual TLS as the first normative producer-to-gateway workload-authentication profile, distinguishes authentication from gateway admission, distinguishes producer workload identity from authorization subject identity, and authorizes — without implementing — a first bounded reference slice (REST adapter, producer-minted `AdapterEvidenceReference`, mTLS submission to `basis-gateway`, stopping at the authorization disposition, no protocol execution). ADR-0008 is recorded `Proposed`, consistent with this repository's ADR governance convention that merging or drafting an ADR does not itself constitute acceptance. `basis-gateway`, `basis-identity`, `basis-adapters`, and `basis-core` are unchanged by ADR-0008. No mTLS validation, certificate handling, or gateway admission configuration exists in any repository yet. The operation-producer runtime itself remains unimplemented and its permanent repository placement remains an explicit open question (per the boundary document's §11). Protocol execution remains future work, out of scope for the bounded reference slice ADR-0008 authorizes.

### Operator and Training Experience Direction

`basis-console`'s current operation-aware integration (Operator and Training presentation modes, gateway-first operation-aware submission, typed evidence handling), released as `v0.2.0`, is functionally and semantically complete for its approved scope and constitutes the correct foundation for the console's next phase of maturity. [`docs/roadmaps/operator-and-training-experience.md`](docs/roadmaps/operator-and-training-experience.md) records the deferred doctrine and maturity roadmap for that next phase: a professional, dense, responsive, investigation-oriented Operator experience with rapid access to trustworthy evidence, and a deeper, playbook-driven Training experience, both built over the same gateway-mediated runtime and evidence the current console already respects.

That document is deliberately doctrine and roadmap, not an implementation plan. Broad implementation of the future Operator workspace and Training playbook framework it describes is intentionally deferred until real investigation workflows, trusted-producer integration (see **Next Producer and Execution-Evidence Boundary** above), execution-result evidence, and identity-activity correlation are further along. Smaller, incremental Operator- and Training-mode improvements consistent with that doctrine may proceed earlier. This is not the next scheduled implementation phase in the **Downstream Rollout Sequence** above; it is the standing doctrine that future console (and eventual CLI) work should be evaluated against whenever that work begins.

---

## Phase 3 — Core Services Distribution

This phase assembles the full BASIS Core Services Distribution: the set of components that together provide a complete, deployable identity-aware authorization system.

| Item | Status |
| - | - |
| basis-gateway: API and runtime wrapper around basis-core | **Released** (`v0.1.0`, `v0.2.0`); `v0.2.0` adds the feature-flagged operation-aware enforcement path, disabled by default; active development continues |
| basis-console: operator and administrator UI | **Released** (`v0.1.0`, `v0.1.1`, `v0.2.0`); `v0.2.0` adds operation-aware Decision Simulator consumption with Operator/Training parity |
| basis-adapters: BACnet adapter | **Released** |
| basis-adapters: Modbus TCP adapter (building on PoC implementation) | **Released** |
| basis-adapters: MQTT adapter (building on PoC implementation) | **Released** |
| basis-adapters: shared AdapterBase interface and normalization contracts | **Released** |
| basis-adapters: additional protocol/platform coverage beyond the original three (REST, OPC UA, DNP3, IEC 61850, KNX, Niagara) | **Released** — nine protocols/platforms normalization-complete in total; per `basis-adapters`' own roadmap, no further protocols are currently planned |
| basis-identity: identity engine and federation boundary | **Released** (`v0.1.0`) |
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

## Identity and Fine-Grained Authorization Expansion

A dedicated roadmap document, [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](docs/roadmaps/identity-and-fine-grained-authorization-expansion.md), elaborates selected work already represented, at a summary level, across Phases 4 and 5 above — multi-tenant identity and trust isolation, tenant-isolated caching, token exchange and delegation, workload and non-human identity, distributed session revocation, SCIM-synchronized registries, relationship-based authorization, fine-grained authorization query APIs, gateway/core runtime integration of relationship sub-decisions, external authorization-technology evaluation, signed policy and configuration distribution, and performance/failure/isolation validation — into a named, twelve-phase capability program with a more detailed dependency sequence than the phase tables above provide.

| Item | Status |
| - | - |
| Identity and fine-grained authorization expansion roadmap (twelve capability phases, architectural invariants, cross-phase security concerns, console education matrix) | **Planned** — its original gate, completion of the `basis-core` operation-aware roadmap, has been satisfied; implementation has not begun |

This expansion is architecture only at this stage. No implementation work in `basis-identity`, `basis-core`, `basis-gateway`, or `basis-console` has begun against it, and the twelve phases it defines are not a committed schedule. The roadmap's original gate — completion of the `basis-core` operation-aware roadmap — has been satisfied: `basis-core`'s operation-aware implementation program is complete and released (`v0.2.0`, additively corrected by `v0.2.1`). Satisfying that gate does not, by itself, start implementation; beginning this program still requires a separate architecture and implementation decision, and should account for the broader ecosystem foundations described under **Next Producer and Execution-Evidence Boundary** above (trusted operation-producer identity, adapter context/evidence alignment, execution-result evidence, `basis-identity` evidence alignment, correlation, deployment, and end-to-end validation), several of which this expansion's own phases depend on. It does not require every unrelated Phase 4 or Phase 5 item above to be completed first, and it does not replace or supersede Phases 4 and 5 — several of their open questions (for example, identity federation and distributed policy coordination in Phase 5) are exactly what the dedicated roadmap develops in more architectural detail. See the roadmap document itself for phase-by-phase prerequisites, decision gates, and deferred decisions.

---

## Identity-to-Operation Contract and Interoperability

A second dedicated roadmap document, [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](docs/roadmaps/identity-to-operation-contract-and-interoperability.md), records long-term architectural intent for BASIS as an open identity-to-operation contract and interoperability layer connecting fragmented OT identity providers, remote-access and PAM systems, gateways, protocol adapters, field devices, SIEMs, and asset inventories around a shared account of identity, authority, delegation, operation, decision, enforcement, execution, and evidence.

| Item | Status |
| - | - |
| Identity-to-operation contract and interoperability roadmap (ten architecture phases, architectural invariants, contract responsibility model, decision gates) | **Planned** — its original gate, the `basis-gateway` operation-aware audit-evidence, readiness, conformance, and release-hardening program, has been satisfied; implementation has not begun |

This roadmap is architecture only. It does not authorize implementation work in any repository and does not define final schemas. The roadmap's original gate has been satisfied: `basis-gateway`'s operation-aware integration, including its audit-evidence, readiness, conformance, and release-hardening work, is complete and released (`v0.2.0`, feature-flagged), per **Completed Operation-Aware Evaluation and Enforcement Boundary** above. Satisfying that gate does not, by itself, start this roadmap's implementation; a separate architecture and implementation decision is still required, and the remaining foundations this roadmap depends on — trusted operation-producer identity, adapter context/evidence alignment, execution-result evidence, `basis-identity` evidence alignment, correlation, deployment, and end-to-end validation — are described under **Next Producer and Execution-Evidence Boundary** above and remain open. Its relationship to the identity and fine-grained authorization expansion roadmap above — where the two intersect and where each is authoritative — is addressed explicitly in that document's own "Relationship to Existing Roadmaps" section.

---

## Post-Authentication Identity Activity, Correlation, and Detection

A third dedicated roadmap document, [`docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`](docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md), records long-term architectural intent for extending BASIS from deterministic authorization and trustworthy evidence into durable identity activity, correlation, deterministic detection, investigation, behavioral analytics, and bounded response for OT — the roadmap the identity-to-operation contract roadmap above already anticipated in its own "Relationship to Existing Roadmaps" section.

| Item | Status |
| - | - |
| Post-authentication identity activity, correlation, and detection roadmap (ten architecture phases, architectural invariants, activity record conceptual requirements, decision gates) | **Planned** — the `basis-gateway` operation-aware integration program its original gate depended on has been satisfied (released as `v0.2.0`); implementation remains deferred because the identity-to-operation contract and evidence foundations this roadmap depends on are not yet stable |

This roadmap is architecture only. It does not authorize implementation work in any repository, does not implement storage, a graph, or detections, and does not select a database, stream processor, graph technology, or machine-learning platform. The `basis-gateway` operation-aware integration program this roadmap's original gate referred to has been satisfied — see **Completed Operation-Aware Evaluation and Enforcement Boundary** above — but that alone does not start this roadmap's implementation: it remains gated on the identity-to-operation contract roadmap's own foundations (trusted operation-producer identity, adapter context/evidence alignment, execution-result evidence, correlation, and the other items under **Next Producer and Execution-Evidence Boundary** above) stabilizing first, and on a separate architecture and implementation decision once they do. It builds on the identity-to-operation contract roadmap's evidence and execution-lifecycle architecture and does not define a competing event model beside it; deterministic detection precedes behavioral analytics throughout, and automated response remains bounded and human-governed. See the roadmap document itself for phase-by-phase prerequisites, decision gates, and its explicit distinction between the activity-investigation graph it develops and the ReBAC/FGA authorization graph the identity and fine-grained authorization expansion roadmap develops.

---

## Identity Provider Integration and Identity Learning Environment

A fourth dedicated roadmap document, [`docs/roadmaps/identity-provider-integration-and-learning-environment.md`](docs/roadmaps/identity-provider-integration-and-learning-environment.md), records long-term architectural intent for a reproducible external-identity-provider reference and learning environment (`basis-demo`) and for the BASIS Training Mode capability that would let an operator or trainee inspect a real identity flow — authentication, normalization, authorization, enforcement, execution, and evidence — as it actually happens. It elaborates the "`basis-demo` end-to-end validation" stage the Downstream Rollout Sequence above already names, specifically for identity-provider integration and identity-systems education, rather than introducing a new or competing future stage.

| Item | Status |
| - | - |
| Identity provider integration and identity learning environment roadmap (ten architecture phases, `basis-demo` responsibility model, provider-neutral IdP profile model, sanitized training observability model, decision gates) | **Planned** — architecture only; no phase is in progress and no repository, including `basis-demo`, is created by it |

This roadmap is architecture only. It does not authorize implementation work in any repository, does not create `basis-demo` or any other repository, and does not select an identity-provider vendor as a runtime dependency of the BASIS Core Services Distribution. It depends most directly on the still-unresolved **Next Producer and Execution-Evidence Boundary** above: its own Phase 6 (the full identity-to-operation learning flow) cannot proceed until [ADR-0008](docs/adr/0008-producer-workload-authentication-and-admission.md) is accepted and implemented and a bounded operation-producer reference implementation exists, per [`docs/architecture/operation-producer-and-execution-boundary.md`](docs/architecture/operation-producer-and-execution-boundary.md). A narrower authentication-only reference lab (its Phase 5) is, by contrast, groundable in capability already released today — `basis-identity`'s `v0.1.0` OIDC, JWKS, session, and BASIS-local token surface — and the roadmap document is explicit about that distinction rather than presenting the two as equally available. It reconciles explicitly with the identity and fine-grained authorization expansion roadmap, the identity-to-operation contract and interoperability roadmap, the post-authentication identity activity roadmap, and the operator and training experience roadmap, consuming capability each of them owns rather than redefining any of it. See the roadmap document itself for phase-by-phase prerequisites, decision gates, and its `basis-demo` responsibility and trust-boundary model.

---

## What This Roadmap Does Not Say

This roadmap does not specify dates, release schedules, or version numbers. It does not commit to building any item in Phase 3, 4, or 5. It does not imply that items listed as "In architecture" or "Research direction" will be implemented in the form or sequence described here.

The roadmap reflects current architectural intent. That intent will be updated as implementation experience, operational feedback, and engineering analysis surface requirements or constraints that the current architecture does not fully anticipate. Changes to architectural intent that affect stable component boundaries or established contracts should be documented through the ADR process described in [`GOVERNANCE.md`](GOVERNANCE.md).

The architecture is a starting point for the implementation work ahead, not a finished specification of it.
