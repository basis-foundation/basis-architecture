# Basitra Target-Architecture Discovery Assessment

> **Status: non-normative discovery assessment.** This assessment informs the future OD-8 architecture decision. It does not itself define the long-term scope of BASIS, rename any component, or authorize migration.
>
> It inventories the architectural capabilities, roles, layers, and boundaries that already exist in accepted or implemented architecture, independent of current repository names. It creates no ADR, resolves neither OD-8 nor OD-9, supersedes no accepted decision, modifies no implementation repository or contract, and creates no new runtime responsibility. Where it reaches a judgment, the judgment is labeled as an inference or a recommendation to the future OD-8 ADR, not as accepted architecture.

**Scope of this pass:** read-only inspection of `basis-architecture` only. Implementation status is taken from what this repository records ([`README.md`](../../README.md), [`ROADMAP.md`](../../ROADMAP.md), the ADRs, and the component architecture documents). Implementation repositories were not re-inspected for this assessment, so every "Implemented" label below means "recorded as implemented in this repository."

**Governing decision:** [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Accepted). Its Decision 3 leaves the long-term scope of BASIS open as OD-8, and its Decision 7 sets the principle this assessment applies: *historical continuity informs migration but does not constrain the target architecture.*

**Companion documents:** [`basitra-ecosystem-and-boundary-aware-security.md`](basitra-ecosystem-and-boundary-aware-security.md) (the "strategy document"; this assessment is input to its [Stage 4](basitra-ecosystem-and-boundary-aware-security.md#124-stage-4--basitra-target-architecture-and-repository-naming-reconciliation), step 1), [`basis-ecosystem.md`](basis-ecosystem.md), [`reference-vs-implementation.md`](reference-vs-implementation.md), [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md), [`kernel-boundary-rules.md`](../kernel-boundary-rules.md), and [ADR-0007](../adr/0007-adapter-evidence-construction.md) through [ADR-0023](../adr/0023-supervisory-platform-and-administrative-interface-boundary.md).

**Terminology in this document.** *BASIS* is used in its current sense: the component family whose long-term scope OD-8 will decide. *Boundary-Aware Security* is always written in full, because this document also refers to building automation systems ([strategy document §5.5](basitra-ecosystem-and-boundary-aware-security.md#55-the-bas-abbreviation-disambiguation-convention)). *The governed path* means the accepted chain from an upstream supervisory system through producer intake, operation production, gateway enforcement, kernel evaluation, authorization-to-execution binding, and protocol execution to an OT target ([ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md); [ADR-0023](../adr/0023-supervisory-platform-and-administrative-interface-boundary.md)). Capability identifiers (`CAP-nn`), layer identifiers (`L1`–`L6`), and candidate letters (A–D) are local to this assessment.

---

## Contents

1. [Purpose and Question](#1-purpose-and-question)
2. [Status Vocabulary and Evidence Rules](#2-status-vocabulary-and-evidence-rules)
3. [Analysis Lens](#3-analysis-lens)
4. [Inventory of Durable Architectural Capabilities](#4-inventory-of-durable-architectural-capabilities)
5. [Roles Versus Implementations](#5-roles-versus-implementations)
6. [Candidate Architectural Layers](#6-candidate-architectural-layers)
7. [Basitra-Level and BASIS-Candidate Responsibilities](#7-basitra-level-and-basis-candidate-responsibilities)
8. [The Ipotio Boundary Is Preserved](#8-the-ipotio-boundary-is-preserved)
9. [Cohesion and Seams](#9-cohesion-and-seams)
10. [OD-8 Candidate Shapes](#10-od-8-candidate-shapes)
11. [The "Identity" in the BASIS Expansion](#11-the-identity-in-the-basis-expansion)
12. [Accepted-ADR and Canonical-Document Constraints](#12-accepted-adr-and-canonical-document-constraints)
13. [Questions the OD-8 ADR Must Answer](#13-questions-the-od-8-adr-must-answer)
14. [Recommendation to the Future OD-8 ADR (Non-Normative)](#14-recommendation-to-the-future-od-8-adr-non-normative)
15. [What This Assessment Does Not Decide](#15-what-this-assessment-does-not-decide)
16. [References](#16-references)

---

## 1. Purpose and Question

The strategy document's Stage 4 is a decision gate that works "architecture first, names second." Its first step is to inventory each component "by what it owns and does not own," with current names explicitly excluded as an input ([strategy document §12.4](basitra-ecosystem-and-boundary-aware-security.md#124-stage-4--basitra-target-architecture-and-repository-naming-reconciliation)). OD-8 then decides what BASIS names inside Basitra, and only after that does OD-9 choose repository and component names.

This assessment performs that first step. Its question is:

> If Basitra is the ecosystem identity, what architectural capabilities, roles, components, and boundaries exist underneath it today, independent of their current repository names?

It then uses that inventory to describe the OD-8 candidate shapes the architecture supports, and the evidence for and against each. It does not choose among them.

---

## 2. Status Vocabulary and Evidence Rules

Every capability, role, and claim carries one of these statuses. They follow the architecture-status vocabulary in [strategy document §2](basitra-ecosystem-and-boundary-aware-security.md#2-status-vocabulary-used-in-this-document), with two additions needed for a capability inventory.

| Status | Meaning in this assessment |
| - | - |
| **Implemented** | Recorded in this repository as realized in a released or merged implementation. A qualifier such as "(bounded)" restates the scope the record gives. |
| **Accepted, not implemented** | Decided by an `Accepted` ADR or approved architecture document; no implementation is recorded, or only part of one. |
| **Established in architecture / analysis** | Described in a canonical architecture document, the white paper, the threat model, or the architecture principles, but not realized as an ADR decision, contract, or implementation. |
| **Planned** | Recorded as a roadmap direction with a `Planned` status. Not decided. |
| **Open** | Explicitly named as an unresolved decision in an accepted document. |
| **Outside Basitra/BASIS responsibility** | Owned by an external system under accepted architecture. Listed only so its boundary is visible. |

Evidence rules applied throughout:

- **Accepted is not implemented.** ADR-0018, ADR-0020, ADR-0021, ADR-0022, and ADR-0023 are accepted and record no implementation of their decisions. ADR-0011 through ADR-0016 are accepted, and protocol execution remains entirely unimplemented ([`README.md`](../../README.md)).
- **Implemented is not accepted.** ADR-0001 through ADR-0006, which govern the operation-aware kernel model, are still recorded `Proposed`, although `basis-core` implements the structure they describe ([ADR-0006](../adr/0006-evaluation-orchestration-layer.md) implementation note). Kernel isolation is governed independently by [`kernel-boundary-rules.md`](../kernel-boundary-rules.md). This assessment does not treat ADR-0001 through ADR-0006 as accepted constraints.
- **A repository is evidence of an implementation boundary, not of a role boundary.** [`reference-vs-implementation.md`](reference-vs-implementation.md) states that the distribution is a reference realization of the conceptual architecture and "not the only possible realization."
- **"BASIS" in accepted ADRs means the current component family.** Accepted ADRs were written with BASIS as the family name. Their use of "BASIS" is therefore evidence that a responsibility belongs to the governed security system, not evidence of what BASIS should name after OD-8. [ADR-0023](../adr/0023-supervisory-platform-and-administrative-interface-boundary.md) states this directly about itself: "The decision is about architectural roles, not names. It applies unchanged under whatever terminology ADR-0019 settles."

---

## 3. Analysis Lens

The analysis organizes the architecture by seven inputs:

| Input | Question asked of each capability |
| - | - |
| **Architectural responsibility** | What does it own, and what is it explicitly barred from owning? |
| **Trust boundary** | Which boundary does it establish, verify, or cross? |
| **Role ownership** | Which logical role owns it under accepted architecture? |
| **Dependency direction** | What does it depend on, and what may depend on it? |
| **Security authority** | What can it decide, admit, permit, or dispatch? |
| **Runtime responsibility** | Does it act on the governed path at runtime, or support the system outside runtime? |
| **Compatibility surface** | Which published contracts, namespaces, or identifiers does it expose? |

Current repository names are not an organizing input. A repository is used as evidence of where a first-party implementation currently lives. Repository existence alone does not establish the correct long-term Basitra architecture.

Two dependency directions are kept apart, because they answer different questions:

- **Build and package dependency** runs toward shared contracts: services depend on `basis-core`, which depends on `basis-schemas` ([`basis-ecosystem.md`](basis-ecosystem.md#component-dependency-direction)).
- **Governed-operation flow** runs from an upstream supervisory system toward the OT target. Each arrow on that path "is a separate boundary with its own admission or verification rule. None of them inherits trust from the one above it" ([ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md) Decision).

A component can sit low in one direction and early in the other. `basis-adapters`, for example, depends on `basis-core` contracts at build time but acts before the gateway on the governed path.

---

## 4. Inventory of Durable Architectural Capabilities

The inventory lists 21 capabilities, grouped by the area they act in. [Section 4.1](#41-responsibility-authority-and-status) records ownership, authority, and status. [Section 4.2](#42-inputs-outputs-boundaries-and-dependencies) records inputs, outputs, boundaries, and dependencies for the same identifiers. [Section 4.3](#43-findings-from-the-inventory) states what the inventory shows.

### 4.1 Responsibility, authority, and status

| ID | Capability | Architectural owner (role) | Current implementation | Security authority held | Status | Source |
| - | - | - | - | - | - | - |
| **Trust establishment** | | | | | | |
| CAP-01 | Identity federation and canonical identity establishment | Identity-engine role (federation and normalization boundary) | `basis-identity` v0.1.0 (OIDC, JWKS, session, BASIS-local token) | Integration authority over canonical identity context. Directory authority only in explicitly configured standalone or air-gapped mode. No authorization authority. | **Implemented** (v0.1.0). Alignment with the operation-aware surface **not begun**. | [`basis-identity.md`](basis-identity.md); [`identity-authority-modes.md`](identity-authority-modes.md); strategy §5.3, §6 |
| CAP-02 | Authorization-subject establishment | Gateway/enforcement role (subject authentication), optionally consuming identity-engine output | `basis-gateway` bearer (JWT/OIDC) verification | Decides which identity is evaluated as the subject. Never derived from producer or upstream workload authentication. | **Implemented** for bearer subjects. Subject-credential conveyance across intake **Open**. | [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) "Producer vs. authorization subject"; ADR-0018 Decision 3; ADR-0023 Decisions 3–4 |
| CAP-03 | Workload identity and admission | Gateway/enforcement role with trusted ingress (producer admission); producer-intake boundary (upstream workload admission) | `basis-gateway` Phase 1B with the ADR-0009 trusted ingress | Admits a logical workload at one boundary. Confers no authorization and no admission at any other boundary. | Producer admission **Implemented** (bounded). Upstream workload admission **Accepted, not implemented**. Lifecycle semantics **Accepted, not implemented**; lifecycle mechanism **Open**. | ADR-0008; [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md); ADR-0018 Decision 2; [ADR-0022](../adr/0022-workload-credential-lifecycle-and-scope-boundary.md) |
| **Authorization** | | | | | | |
| CAP-04 | Mapping and canonical composition | Deployment-designated BASIS configuration authority (mapping configuration); operation-producer role (effective mapping identity); gateway (sole canonical composer on the governed admitted-producer path) | Gateway operation-aware composition exists in dual-accept form; producer REST mapping is fixed in code | Determines the canonical action and resource the kernel evaluates | Composition **Implemented** in its pre-ADR-0020 form. ADR-0020 **Accepted, not implemented**. Mapping administrative surface **Open**. | [ADR-0017](../adr/0017-action-vocabulary-naming-structure.md); [ADR-0020](../adr/0020-operation-to-authorization-mapping-and-composition-boundary.md) **Ownership** and **Validation** |
| CAP-05 | Context-assertion admission | Gateway (final enforcement); intake boundary (earlier rejection); producer (local category scope); governed category trust policy | All-or-nothing producer-only context gate in `basis-gateway` | Decides which context values reach policy evaluation | Category-scoped trust **Accepted, not implemented**. ADR-0018 Decision 4 interim posture in force. | [ADR-0021](../adr/0021-upstream-context-assertion-trust-boundary.md) Decision 11 |
| CAP-06 | Policy evaluation | Authorization-kernel role | `basis-core` v0.2.x | Sole source of the authorization outcome | **Implemented**. Governing model ADRs 0001–0006 recorded `Proposed`. | [`kernel-boundary-rules.md`](../kernel-boundary-rules.md); [`operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md) |
| CAP-07 | Policy enforcement | Gateway/enforcement role; in the embedded model, the adapter host | `basis-gateway` v0.2.0 (operation-aware path feature-flagged) | Enforces the kernel's disposition and fails closed. Adds no permit logic. | **Implemented** for authorization enforcement | [`basis-gateway.md`](basis-gateway.md); [`basis-adapters.md`](basis-adapters.md) **Embedded Model**; [ADR-0011](../adr/0011-protocol-execution-role-and-bounded-reference-topology.md) Decision 18 |
| CAP-08 | Authorization evidence | Distributed: kernel (decision evidence), gateway (`GatewayAuditEvent`, composition evidence), adapter role (evidence material and digest), operation-producer role (retention and reference lifecycle) | `basis-core`, `basis-gateway`, `basis-adapters`, `basis-producer` | None over decisions. Audit is evidence, not enforcement. | **Implemented** | [`operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md); ADR-0007; [`kernel-boundary-rules.md`](../kernel-boundary-rules.md#audit) |
| **Operation and execution governance** | | | | | | |
| CAP-09 | Producer intake | Producer-intake boundary, the ingress responsibility of the operation-producer role | None | Admits an intake request for production only. Not authorization, not gateway admission. | **Accepted, not implemented**. Transport, integrity, and bounded-validity mechanism **Open**. Repository placement deferred. | ADR-0018 Decisions 2, 6, 7, **Deferred Decisions** |
| CAP-10 | Operation production and orchestration | Operation-producer role | `basis-producer` bounded authorization slice (Phases 2A–5) | Holds the producer workload credential; conveys a subject credential independently; will create the binding record. No authorization authority. | **Implemented** (bounded, authorization-only; not a complete production runtime) | [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md); [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §2 |
| CAP-11 | Authorization-to-execution binding | Operation-producer role creates; protocol-executor role verifies | None | Gates dispatch on a content-based match between the dispatched and the authorized operation | **Accepted, not implemented** (same-process). Bounded REST execution plan approved. Separated-topology binding **Open**. | [ADR-0012](../adr/0012-authorization-to-execution-binding.md); [`bounded-rest-execution-implementation-plan.md`](bounded-rest-execution-implementation-plan.md) |
| CAP-12 | Protocol execution | Protocol-executor role | None. First slice colocated in `basis-producer`'s process; no `basis-executor` | Dispatches only on a bound, permitting disposition. Holds device credentials where a target requires them (Gate 4). | **Accepted, not implemented**. First bounded target selected (credential-free REST). | ADR-0011 Decisions 1–4; [ADR-0015](../adr/0015-first-bounded-execution-target.md) |
| CAP-13 | Execution lifecycle observation | Protocol-executor role | None | Sole truth-owning observer of execution-boundary facts | **Accepted, not implemented** | [ADR-0013](../adr/0013-execution-lifecycle-semantics.md) Decision 14 |
| CAP-14 | Execution evidence | Execution-evidence-producer role (construction and retention) | None. First-slice retention may be colocated in `basis-producer` | None over decisions or dispatch | **Accepted, not implemented**. Schema publication declined for now. | [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md) **Observation, Construction, and Retention Ownership** |
| **Protocol integration** | | | | | | |
| CAP-15 | Protocol normalization and adapter evidence construction | Protocol-adapter role (trusted normalization library) | `basis-adapters` v0.2.0 (nine protocol families) | No decision authority. Semantic trust: authorization semantics derive only from its normalization. In the embedded model, the adapter host becomes an enforcement boundary. | **Implemented** (normalization and evidence construction). Execution excluded. | [`basis-adapters.md`](basis-adapters.md); ADR-0007; ADR-0011 Decision 16; ADR-0020 Decision 5 |
| **Administration** | | | | | | |
| CAP-16 | Administrative and operator interface | Administrative-interface role | `basis-console` v0.2.0 | A BASIS administrative context only. No OT operation-initiation, execution, or subject authority. Its actions are enforced at the gateway. | **Implemented** (v0.2.0). Conformance to ADR-0023 not reviewed, per `basis-console.md`. Optional in every deployment. | [`basis-console.md`](basis-console.md) Design Invariants 10–15; ADR-0023 Decisions 5–7 |
| **Ecosystem support** | | | | | | |
| CAP-17 | Shared contracts and compatibility definitions | Contract-publication role ("Architecture proposes. Schemas publish. Implementations consume.") | `basis-schemas` v0.2.x | None at runtime. Governs compatibility of shared shapes. | **Implemented** for published contracts. Several candidates unpublished, including execution evidence (declined) and the composition rule (future candidate). | [`basis-schemas.md`](basis-schemas.md); [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md); [`compatibility-philosophy.md`](compatibility-philosophy.md) |
| CAP-18 | Deployment and distribution | Deployment-tooling role | None; `basis-deploy` not established | None over semantics. Shapes the deployment boundary (placement, configuration, secrets). | Tooling **Established in architecture**, not implemented. Deployment boundary **Established in analysis**. | [`basis-ecosystem.md`](basis-ecosystem.md); [`kernel-boundary-rules.md`](../kernel-boundary-rules.md); strategy §5.3 |
| CAP-19 | Architecture and governance | Architecture authority; Basis Foundation stewardship | `basis-architecture`; [`GOVERNANCE.md`](../../GOVERNANCE.md) | Decides architecture. No runtime authority. | **Implemented** as practice | [`basis-ecosystem.md`](basis-ecosystem.md#basis-foundation); [`docs/adr/README.md`](../adr/README.md) |
| **External** | | | | | | |
| CAP-20 | Supervisory intent | Upstream supervisory system (for example, Ipotio or Niagara; neither is a special case) | External | Originates operator-driven OT intent. No producer, authorization, execution, or credential authority in BASIS. | **Outside Basitra/BASIS responsibility** | ADR-0018 Decision 1; ADR-0023 Decisions 1–2 |
| CAP-21 | Topology and operational-state ownership (asset model, supervisory state, alarms, scheduling, operator workflows, target resolution) | Upstream supervisory system | External | None in BASIS. BASIS performs no upstream-reference-to-target resolution. | **Outside Basitra/BASIS responsibility** | ADR-0023 Decision 1; ADR-0020 Decision 3 |

### 4.2 Inputs, outputs, boundaries, and dependencies

| ID | Inputs | Outputs | Trust boundary crossed or established | Upstream dependencies | Downstream dependents |
| - | - | - | - | - | - |
| CAP-01 | External IdP assertions and tokens; local credentials in standalone mode | Canonical identity context; BASIS-local token where configured | Identity boundary (external IdP → canonical identity) | Enterprise IdPs; `basis-schemas` (identity context shape) | CAP-02. Does not depend on `basis-core`. |
| CAP-02 | Subject credential; canonical identity context | `Subject` and `IdentityContext` for the decision request | Subject authentication at the gateway | CAP-01 (where deployed); CAP-10 conveys the credential | CAP-06, CAP-07 |
| CAP-03 | Producer client certificate (URI SAN); upstream admission material (mechanism Open) | Admitted logical workload identity | Producer admission boundary; producer-intake boundary | Deployment-owned PKI and issuance (mechanism Open) | CAP-07 (producer admission); CAP-09 (intake admission) |
| CAP-04 | Normalized fields; identified mapping snapshot | Canonical action and resource identifier; composition evidence | Pre-kernel, inside the gateway | CAP-15, CAP-10 | CAP-06 |
| CAP-05 | Context values with origin, source, and provenance | Admitted context with provenance classification, or fail-closed rejection | Intake boundary; producer → gateway | CAP-20 (supply), CAP-10 (relay) | CAP-06 |
| CAP-06 | Decision request (subject, action, resource, context); policy bundle | Decision response; decision evidence | Authorization/policy boundary (gateway → kernel) | CAP-02, CAP-04, CAP-05, CAP-07; `basis-schemas` only at build time | CAP-07, CAP-08 |
| CAP-07 | Authenticated request | Authoritative disposition; `GatewayAuditEvent` | Runtime API boundary | Producers, the console, and other enforcement-point callers | CAP-06; disposition to CAP-10 |
| CAP-08 | Decision, composition, and protocol facts | Durable authorization records | Evidence/audit boundary | CAP-06, CAP-07, CAP-15, CAP-10 | CAP-16; external evidence consumers |
| CAP-09 | Intake request from an upstream supervisory system | Admitted intake request, or an explicit rejection or failure | Producer-intake boundary | CAP-20; CAP-03 | CAP-10 |
| CAP-10 | Preserved `ProtocolOperation`; adapter result | Authenticated operation-aware submission; retained evidence and reference; binding record (accepted, not implemented) | Producer admission (as client) | CAP-09, CAP-15 (library dependency) | CAP-07 (network client only); CAP-11. Nothing depends on it. |
| CAP-11 | Preserved operation; submitted normalized request; disposition | Verified, single-consumption binding | Authorization-to-execution boundary | CAP-10, CAP-07 | CAP-12 |
| CAP-12 | Verified binding; preserved operation | Protocol dispatch; observations | BASIS → OT target | CAP-11 | OT target; CAP-13 |
| CAP-13 | Protocol-channel results | Attempt outcome; resulting-state verification | Execution boundary | CAP-12 | CAP-14 |
| CAP-14 | Executor observations; correlation facts | Execution-evidence record; durability outcome | Evidence boundary, kept separate from authorization evidence | CAP-13 | Evidence consumers; never mutates CAP-08 |
| CAP-15 | `ProtocolOperation` | Normalized request; evidence material and digest | Protocol boundary (semantic trust boundary) | `basis-core` adapter contracts at build time | CAP-10; CAP-04 (via normalized fields) |
| CAP-16 | Administrator actions | Gateway API calls for administration, inspection, diagnostics, labeled simulation | Administrative path to the gateway; no path to intake, binding, or executor | CAP-07 | None on the governed path |
| CAP-17 | Decided architecture | Machine-readable contracts; compatibility fixtures | None at runtime | CAP-19 | Every implementation component |
| CAP-18 | Released components; configuration | Packaged, configured deployments | Deployment boundary | All runtime components | Deployments. No component depends on it. |
| CAP-19 | Evidence, proposals, review | ADRs, standards, architecture documents | None at runtime | — | All capabilities |
| CAP-20 | Operator and automation intent | Intake requests | Producer-intake boundary (as submitter) | External identity and operational systems | CAP-09 |
| CAP-21 | Upstream operational state | Protocol-shaped operations it authors | None in BASIS | — | CAP-20 |

### 4.3 Findings from the inventory

1. **Roles outnumber repositories, and one repository hosts several roles.** Under accepted architecture, `basis-producer`'s process boundary hosts the operation-producer role, and for the first bounded slice also the protocol-executor and execution-evidence-producer roles (ADR-0011 Decision 3; ADR-0014). The producer-intake boundary is assigned to the operation-producer role as its ingress responsibility, while its repository placement is deferred (ADR-0018 Decision 2, **Deferred Decisions**). The current repository name describes one of these roles.
2. **Some accepted authorities have no assigned component.** ADR-0020 assigns mapping configuration to a "deployment-designated BASIS configuration authority," and ADR-0021 requires a "BASIS-governed category trust policy." Both defer the administrative surface, storage, and approval workflow. No component owns these administrative surfaces today.
3. **Evidence is a distributed obligation, not a component.** Authorization evidence is produced by four owners (CAP-08), and execution evidence by a further distinct role (CAP-14). ADR-0023 Decision 4 places evidence with "owning components." No architecture assigns evidence to a single component.
4. **Implementation is concentrated in authorization.** Identity federation, subject and producer authentication, normalization, composition, evaluation, enforcement, and authorization evidence are implemented. Intake, binding, execution, lifecycle observation, and execution evidence are accepted and unimplemented.
5. **Several documents still describe pre-ADR-0011 responsibilities.** [`basis-console.md`](basis-console.md) Design Invariant 5 says protocol execution belongs to `basis-adapters`, and [`kernel-boundary-rules.md`](../kernel-boundary-rules.md#relationship-to-surrounding-basis-components) says `basis-adapters` applies outcomes "to field-level command delivery." ADR-0011 Decision 16 and Alternative C place execution outside adapters. This assessment records the inconsistency and does not correct it. It is relevant to any later per-repository documentation reconciliation.

---

## 5. Roles Versus Implementations

Accepted architecture repeatedly separates a durable logical role from the first-party component that currently implements it. The glossary states the general rule: "a deployment may substitute a conforming alternative for a given role where the architecture permits one" ([`glossary.md`](../glossary.md#basis-core-services-distribution)).

| Logical role | Defined by | Current first-party implementation | Conforming alternatives permitted? | Role equals repository? |
| - | - | - | - | - |
| Authorization-kernel role (policy engine) | [`kernel-boundary-rules.md`](../kernel-boundary-rules.md); [`reference-vs-implementation.md`](reference-vs-implementation.md) ("the logical role (policy engine, authorization kernel)" versus "the specific component that fulfills that role") | `basis-core` | Yes, at the conceptual level ([`reference-vs-implementation.md`](reference-vs-implementation.md)). Kernel isolation rules bind whatever fulfills the role. | No |
| Gateway/enforcement role | [`basis-gateway.md`](basis-gateway.md) ("the reference enforcement API"); [`basis-adapters.md`](basis-adapters.md) (adapter host as enforcement boundary in the embedded model) | `basis-gateway` | Yes. The embedded model already places enforcement in an adapter host. | No |
| Identity-engine role | [`basis-identity.md`](basis-identity.md) ("This analogy describes an architectural role, not an implementation mandate") | `basis-identity` | Yes. The gateway may verify identity directly in deployments without `basis-identity`. | No |
| Protocol-adapter role | [`basis-adapters.md`](basis-adapters.md); ADR-0007 | `basis-adapters` | Yes. Protocol adapters are a named community extension area (strategy §11.3, P-1). | No |
| Operation-producer role | [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §2; ADR-0010 | `basis-producer` | Yes: `basis-producer` is the Foundation-maintained implementation, "not a claim that it is the only possible one" (ADR-0010 **Distribution Membership**) | ADR-0010 fixes the permanent repository for the Foundation-maintained implementation, and keeps the role distinct from it |
| Producer-intake boundary | ADR-0018 Decision 2 | None | Mechanism-neutral; may be realized differently in networked and air-gapped deployments | No; placement deferred |
| Protocol-executor role | ADR-0011 Decisions 1–5 | None (first slice colocated in `basis-producer`) | Yes; a separated executor is a valid future topology with additional architecture | No; ADR-0011 Decision 4 reserves no repository name |
| Execution-evidence-producer role | ADR-0014 | None (first slice colocated) | Logical separation required; deployment separation optional | No |
| Administrative-interface role | ADR-0023 Decisions 5–7 ("any BASIS user interface, including `basis-console`") | `basis-console` | Yes; the console "must remain optional within the ecosystem" (Design Invariant 10) | No |
| Contract-publication role | [`basis-schemas.md`](basis-schemas.md) §5 | `basis-schemas` | Not addressed as a role. Its purpose is to be the single neutral definition point. | Effectively yes today, by design of the single-source-of-truth model |
| Deployment-tooling role | [`basis-ecosystem.md`](basis-ecosystem.md) | None (`basis-deploy` not established) | Commercial deployment services also exist in the model | No |
| Architecture authority | [`basis-ecosystem.md`](basis-ecosystem.md); [`GOVERNANCE.md`](../../GOVERNANCE.md) | `basis-architecture` | No | Effectively yes today |
| Upstream supervisory-system role | ADR-0018 Decision 1; ADR-0023 Decision 2 | External (examples: Ipotio, Niagara) | Any platform in the category | Not a Basitra or BASIS repository |

**Finding.** Only one accepted ADR, ADR-0010, fixes a role-to-repository identity, and even it keeps the role distinct from the implementation. Every other role-to-repository mapping is established by canonical architecture documents, chiefly the repository table in [`basis-ecosystem.md`](basis-ecosystem.md#repository-and-component-boundaries). The architecture does not equate *role* with *repository* anywhere it defines a runtime role.

This matters for OD-8 in one specific way: if BASIS names a set of **roles**, a conforming third-party producer or adapter would fall inside BASIS. If BASIS names a set of **first-party implementations**, it would not. Accepted architecture treats the roles as the durable unit (ADR-0010, ADR-0011, ADR-0018 Decision 8, ADR-0023 Decision 7). See [question Q-5](#13-questions-the-od-8-adr-must-answer).

---

## 6. Candidate Architectural Layers

The layers below are derived from responsibility and authority, not from repository names. The aim was the smallest model that keeps every accepted boundary visible without inventing new ones.

```text
L1  Stewardship and architecture governance        non-runtime    ecosystem-level
L2  Interoperability contracts                     non-runtime    shared by all layers
L3  Governed security path                         runtime        cohesive security subsystem
      L3a  trust establishment
      L3b  authorization decision
      L3c  operation and execution governance
      (evidence: an obligation of each L3 owner, not a separate layer)
L4  Protocol integration edge                      runtime/library   security-relevant edge
L5  Administration and observability surfaces      runtime        consumer of L3
L6  Deployment and distribution                    non-runtime    packages L3–L5

External: upstream supervisory platforms, enterprise IdPs, OT targets,
          evidence consumers, BASAuth
```

### L1 — Stewardship and architecture governance

- **Belongs:** the ADR process, architecture standards, principles, kernel boundary rules, white papers (CAP-19).
- **Does not belong:** any runtime behavior, contract publication, or implementation decision ([`reference-vs-implementation.md`](reference-vs-implementation.md)).
- **Trust responsibility:** none at runtime; it decides what the runtime layers must guarantee.
- **Dependency direction:** every layer conforms to it; it depends on none.
- **Classification:** an ecosystem-level capability. ADR-0019 makes Basitra the identity of "the open-source project, its community, its ecosystem," which is where stewardship sits.

### L2 — Interoperability contracts

- **Belongs:** published shared shapes such as the decision request and response, audit event, action vocabulary, resource identifier, normalized request, and compatibility fixtures (CAP-17).
- **Does not belong:** policy logic, authentication, protocol translation, deployment topology, UI workflows, or architecture reasoning ([`basis-schemas.md`](basis-schemas.md) §4).
- **Trust responsibility:** none at runtime. The contracts are security-relevant, because they define what crosses each boundary, and a breaking change affects every consumer at once.
- **Dependency direction:** the dependency sink. "Components depend on the schemas; the schemas depend on nothing in the distribution."
- **Classification:** ambiguous. The contents are almost entirely security contracts of L3. The role is ecosystem interoperability: a "single, neutral home," so that "no component is the de-facto authority by accident," and the reference against which conforming alternative implementations are tested. See [§9.4](#94-are-schemas-an-implementation-concern-of-basis-or-an-ecosystem-interoperability-concern).

### L3 — Governed security path

- **Belongs:**
  - **L3a, trust establishment:** identity federation and canonical identity (CAP-01), subject establishment (CAP-02), workload admission and credential lifecycle (CAP-03).
  - **L3b, authorization decision:** composition (CAP-04), context admission (CAP-05), evaluation (CAP-06), enforcement (CAP-07), authorization evidence (CAP-08).
  - **L3c, operation and execution governance:** intake (CAP-09), production (CAP-10), binding (CAP-11), execution dispatch gating (CAP-12), lifecycle observation (CAP-13), execution evidence (CAP-14).
- **Does not belong:** supervisory intent or operational state (CAP-20, CAP-21); administration as an origin of OT operations (ADR-0023 Decision 5); packaging (L6).
- **Trust responsibility:** the security boundaries on the governed path whose responsibilities L3 roles own: identity, subject and workload admission, producer intake, the authorization/policy boundary, the authorization-to-execution boundary, and the evidence boundary ([strategy document §5.3](basitra-ecosystem-and-boundary-aware-security.md#53-boundary-categories-and-their-current-status)). This is the set ADR-0023 Decision 4 lists as what BASIS "independently establishes": upstream workload admission, subject establishment, producer admission, authorization, binding and governed execution, and evidence. L3 is not the complete set of implemented or accepted Boundary-Aware Security boundaries. The catalog also includes the protocol boundary, whose normalization and evidence construction sit in L4.
- **Dependency direction:** governed-operation flow runs L3c intake → L3c production → L3b → L3c binding and dispatch. At build time, L3 components depend on L2 and on the kernel; the kernel depends only on L2. The identity engine depends on L2 and not on the kernel.
- **Classification:** a cohesive security subsystem. The accepted cross-cutting trust rules in [§9.7](#97-which-responsibilities-share-one-trust-model) collectively couple L3a, L3b, and L3c. The accepted architecture therefore provides substantial evidence for treating them as one cohesive subsystem. OD-8 remains the decision that determines whether that cohesion should define the BASIS boundary.

### L4 — Protocol integration edge

- **Belongs:** protocol-specific normalization and adapter evidence construction (CAP-15). When execution is implemented, protocol-specific dispatch mechanics behind the protocol-executor role may also sit here; ADR-0011 Decision 6 leaves that taxonomy open.
- **Does not belong:** authorization, credentials, live protocol communication from the normalization library, gateway submission, execution, or execution policy (ADR-0010 non-responsibilities; ADR-0011 Decision 16).
- **Trust responsibility:** semantic. "The authorization decision is only as good as the normalization in front of it" (strategy §7). Under ADR-0020 Decision 5, adapter normalization is the only path to authorization semantics on the governed producer path. In the embedded model, the adapter host carries enforcement responsibility.
- **Dependency direction:** depends on kernel contracts at build time; acts before L3b at runtime; nothing in L3b depends on any particular adapter.
- **Classification:** a security-relevant edge. It participates in Boundary-Aware Security (the protocol boundary is in the catalog, recorded as implemented for normalization and evidence construction) and is also the main extension area. Participating in Boundary-Aware Security does not by itself place L4 inside BASIS. Whether it belongs there is an explicit OD-8 edge-placement decision. See [§9.3](#93-do-adapters-belong-inside-a-secure-identity-and-authorization-service-or-at-the-ecosystem-integration-edge).

### L5 — Administration and observability surfaces

- **Belongs:** administrative, inspection, diagnostic, and labeled simulation interfaces (CAP-16), and the not-yet-assigned administrative surfaces for mapping configuration and category trust policy ([§4.3](#43-findings-from-the-inventory), finding 2).
- **Does not belong:** authorization evaluation, independent authentication, protocol handling, device management, or any path into intake, binding, or execution ([`basis-console.md`](basis-console.md) Design Invariants; ADR-0023 Decision 7).
- **Trust responsibility:** administrative actions are themselves authenticated at the gateway and evaluated by policy. Policy-administration integrity is a residual risk the architecture names (ADR-0023 **Residual risks**).
- **Dependency direction:** consumes L3 through the gateway only. L3 does not depend on L5; the console "must remain optional."
- **Classification:** a consumer of the security subsystem. It administers L3 and holds no authority on the governed path. The strategy document classifies console presentation modes as "Unrelated" to Boundary-Aware Security as a model (strategy §6).

### L6 — Deployment and distribution

- **Belongs:** packaging, configuration management, deployment validation (CAP-18).
- **Does not belong:** runtime semantics, runtime credentials, or protocol dispatch ([`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §10; ADR-0010 alternatives).
- **Trust responsibility:** the deployment boundary (placement, configuration, secrets) is in the threat model, and ADR-0011 Decision 19 states that some bypass resistance depends on deployment controls. The tooling itself owns no security semantics.
- **Dependency direction:** packages L3–L5; no component depends on it; `basis-core` "must be deployable independently of basis-deploy's packaging choices."
- **Classification:** ecosystem-level support. [`basis-ecosystem.md`](basis-ecosystem.md) states that it "is not part of the authorization runtime — it is the tooling used to stand one up."

---

## 7. Basitra-Level and BASIS-Candidate Responsibilities

The matrix asks, for each capability, whether it is intrinsically part of a security, identity, and authorization service that BASIS could reasonably name, or better understood as a broader Basitra ecosystem capability. It answers from responsibility, not from current names.

*Natural BASIS candidate* and *Natural Basitra-level candidate* are analytical classifications only. They are not decisions, and a capability can be a candidate for both. "Y" means yes, "P" means partially, "N" means no. Confidence is in the classification, not in any OD-8 outcome.

| Capability / role | Security-critical | Identity-related | Authorization-related | Execution-governance | Protocol-specific | Administrative | Deployment / infra | Natural BASIS candidate | Natural Basitra-level candidate | Evidence / rationale | Confidence |
| - | - | - | - | - | - | - | - | - | - | - | - |
| CAP-01 Identity federation | Y | Y | N | N | N | N | N | Strong | Weak | Establishes the canonical identity every boundary relies on. Upstream of evaluation, but shares the canonical identity context with the gateway. | High |
| CAP-02 Subject establishment | Y | Y | Y | N | N | N | N | Strong | Weak | ADR-0023 Decision 4, item 2 | High |
| CAP-03 Workload identity and admission | Y | Y | P | N | N | N | N | Strong | Weak | ADR-0008, ADR-0018, ADR-0022; three-identity separation | High |
| CAP-04 Mapping and composition | Y | N | Y | N | P | P | N | Strong | Weak | Determines what the kernel evaluates; ADR-0020 makes the gateway sole composer | High |
| CAP-05 Context admission | Y | P | Y | N | N | N | N | Strong | Weak | ADR-0021; context never establishes the subject | High |
| CAP-06 Policy evaluation | Y | N | Y | N | N | N | N | Strong | Weak | Sole authorization authority | High |
| CAP-07 Policy enforcement | Y | P | Y | P | N | N | N | Strong | Weak | Authoritative disposition; fail-closed | High |
| CAP-08 Authorization evidence | Y | P | Y | N | P | N | N | Strong | Weak | Attributability of each crossing is part of the Boundary-Aware Security definition | High |
| CAP-09 Producer intake | Y | Y | N | P | N | N | N | Strong | Weak | ADR-0018: "The boundary belongs to the BASIS side" | High |
| CAP-10 Operation production | Y | Y | N | Y | P | N | N | Strong | Weak | Holds producer credential; creates binding; ADR-0018 Decision 8 and ADR-0023 Decision 4 place it inside BASIS | Medium-high |
| CAP-11 Binding | Y | N | Y | Y | N | N | N | Strong | Weak | Spans producer and executor; ADR-0023 Decision 1 lists binding as BASIS-owned | High |
| CAP-12 Execution governance (binding verification, no dispatch before permission) | Y | N | P | Y | N | N | N | Strong | Weak | ADR-0011 Decisions 8–10; ADR-0023 Decision 1 ("governing dispatch") | High |
| CAP-12 Protocol-specific dispatch mechanics | Y | N | N | Y | Y | N | N | Moderate | Moderate | ADR-0011 Decisions 5–6 leave implementation taxonomy and separation open | Low |
| CAP-13 Lifecycle observation | Y | N | N | Y | P | N | N | Strong | Weak | Truthfulness of execution facts | Medium-high |
| CAP-14 Execution evidence | Y | N | N | Y | N | N | N | Strong | Weak | ADR-0023 Decision 4, item 6 | Medium-high |
| CAP-15 Protocol normalization | Y | N | P | N | Y | N | N | Moderate | Moderate | Semantic trust boundary, but no authority, no credential, and a named extension area | Low |
| CAP-16 Administrative interface | P | P | P | N | N | Y | N | Moderate | Moderate | ADR-0023 calls it a "BASIS administrative interface" for "BASIS-owned responsibilities"; optional; holds no path authority | Medium-low |
| CAP-17 Shared contracts | P | P | P | P | P | N | N | Moderate | Moderate-strong | Security shapes; neutral publication role; conformance reference | Medium |
| CAP-18 Deployment and distribution | P | N | N | N | N | N | Y | Weak | Strong | "Not part of the authorization runtime" | High |
| CAP-19 Architecture and governance | N | N | N | N | N | Y | N | Weak | Strong | Stewardship of the whole project | High |
| CAP-20 Supervisory intent | — | — | — | — | — | — | — | Neither | Neither | Outside: ADR-0018, ADR-0023 | High |
| CAP-21 Topology and operational state | — | — | — | — | — | — | — | Neither | Neither | Outside: ADR-0023 Decision 1 | High |

**Reading the matrix.** Every L3 capability classifies as a strong BASIS candidate with medium-high or high confidence. The one exception is the separate protocol-specific dispatch mechanics row, discussed below. Every capability classified as a strong Basitra-level candidate is non-runtime (L1, L6). The low- and medium-confidence rows are the L2, L4, and L5 rows, plus protocol-specific dispatch mechanics, which [Section 6](#6-candidate-architectural-layers) notes may sit at the L4 edge. The evidence converges on the core and does not converge on the adjacent layers.

The strong BASIS candidates include responsibilities that hold no decision or dispatch authority: authorization evidence (CAP-08), lifecycle observation (CAP-13), and execution evidence (CAP-14). Their owners preserve, observe, or evidence what crossed the governed path. Any membership rule that matches this column must therefore cover those responsibilities, and cannot be "holds authority" alone. [Section 10.1](#101-the-candidates) states the rule this assessment proposes for Candidate C.

---

## 8. The Ipotio Boundary Is Preserved

The Ipotio/Basitra relationship is resolved at durable semantics and is not reopened here:

```text
Ipotio          = supervisory / operational platform
Basitra/BASIS   = independent authorization and security substrate
```

The following invariants hold under every candidate shape in [Section 10](#10-od-8-candidate-shapes). None of the candidates is viable unless it preserves them.

```text
authentication                   !=  authorization
Ipotio request                   !=  authorization
Ipotio operational eligibility   !=  BASIS authorization
BASIS ALLOW                      !=  dispatch
authorization                    !=  execution
```

They restate accepted decisions: ADR-0023 Decision 1 ("BASIS is an authorization and security substrate for OT operations. It is not a supervisory platform."), ADR-0018 Decisions 1–2 (intake admission is neither gateway admission nor authorization), ADR-0011 Decision 9, and the consolidated invariants in ADR-0023 Decision 9. Ipotio appears in this assessment only as an example of the upstream supervisory-system category (CAP-20). It receives no special trust, type, or vocabulary (ADR-0018 Decision 9; ADR-0023 Decision 11). Ipotio's informal use of "Basitra" for BASIS components remains a reconciliation item in Ipotio's own repository that follows OD-8 and OD-9 (strategy C-6, OD-3).

One consequence for OD-8: whatever BASIS comes to name, the accepted text "BASIS is an authorization and security substrate" must continue to identify the governed security system. If OD-8 narrowed BASIS so that it no longer covered that whole substrate, the OD-8 ADR would need to state which term now carries ADR-0023's meaning. See [Q-9](#13-questions-the-od-8-adr-must-answer).

---

## 9. Cohesion and Seams

### 9.1 Is identity distinct enough from authorization to sit beside BASIS rather than inside it?

**Evidence for a seam.** [`basis-identity.md`](basis-identity.md) separates three questions: `basis-identity` answers "who is this," `basis-gateway` answers "is this request permitted and enforced," and `basis-core` answers "does policy permit this." The identity engine does not depend on the kernel ([`basis-ecosystem.md`](basis-ecosystem.md#component-dependency-direction)), never evaluates authorization, and is optional: the gateway may verify identity directly.

**Evidence against placing identity outside the security subsystem.** Identity is not confined to the identity engine. Subject establishment is a gateway responsibility (ADR-0008). Producer workload identity, upstream workload identity, and their lifecycle are governed by ADR-0008, ADR-0018, and ADR-0022, none of which assigns them to `basis-identity`; ADR-0008 explicitly does not. The three-identity separation (ADR-0018 Decision 3) and the administrative-versus-subject separation (ADR-0023 Decision 5) are enforced across intake, producer, and gateway. The canonical identity context that `basis-identity` produces is the input the gateway validates.

**Inference.** *Identity federation* is a separable component with a clean dependency boundary. *Identity establishment at boundaries* is not separable from authorization; it is distributed across L3. The architecture supports identity federation as a distinct component inside the security subsystem. It provides no evidence for identity federation as a subsystem outside it.

### 9.2 Is execution governance part of the same security subsystem as authorization?

**Evidence that it is.** The binding record is created by the operation-producer role before gateway submission and verified by the protocol-executor role before dispatch, over the operation and the normalized request that was authorized (ADR-0012). ADR-0023 Decision 1 lists "binding a permitted decision to the exact operation authorized" and "governing dispatch through the protocol-executor role" among the responsibilities BASIS owns, next to evaluation. Boundary-Aware Security names the authorization-to-execution crossing as one of its three defining properties (strategy §5.2, item 3).

**Evidence of a separation.** *Authorization ≠ execution* is load-bearing (ADR-0011 Context). Execution evidence must never mutate authorization evidence (ADR-0011 Decision 11). The executor never calls the kernel (ADR-0011 Decision 10).

**Inference.** The authorization/execution split is a **lifecycle boundary inside the governed path**, enforced by binding. Accepted architecture does not treat it as a boundary between subsystems. A subsystem boundary drawn between authorization and execution governance, as in Candidate B ([Section 10](#10-od-8-candidate-shapes)), would **not** split binding creation from binding verification. Under B, the operation producer, binding creation, binding verification, governed dispatch, lifecycle observation, and execution evidence would all sit together outside BASIS. What B does is raise the lifecycle boundary into a subsystem boundary placed immediately after the authorization decision:

```text
BASIS
  identity + authorization decision
          │
          │  authoritative disposition crosses the subsystem boundary
          ▼
separately classified operation and execution governance
  producer · binding creation · binding verification ·
  governed dispatch · lifecycle observation · execution evidence
```

The coupling across that boundary would remain strong. The outside subsystem exists to preserve and enforce the exact relationship among four things: the preserved operation, the normalized authorization request submitted for it, the authoritative disposition produced inside BASIS, and the operation dispatched (ADR-0012). The governed path would also cross the boundary twice. Intake and production come before authorization in the flow, so the operation would enter BASIS when the producer submits it for admission and authorization, and leave again with the disposition.

The architecture does not prohibit this. *Authorization ≠ execution* already makes the disposition a defined handoff, and ADR-0011 Decision 5 contemplates separated executors. The question for OD-8 is whether this lifecycle boundary should also be a subsystem boundary. The accepted record leans against it without ruling it out: ADR-0023 Decision 1 lists binding and governing dispatch among BASIS's own responsibilities, and ADR-0018 places intake, the producer, and the executor on the BASIS side.

Protocol-specific dispatch *mechanics* are more separable than execution *governance*: ADR-0011 Decisions 5 and 6 allow a separated executor and protocol-specific implementations, so the low-confidence row in [Section 7](#7-basitra-level-and-basis-candidate-responsibilities) concerns mechanics, not governance.

### 9.3 Do adapters belong inside a secure identity and authorization service, or at the ecosystem integration edge?

**Evidence for inside.** The protocol boundary is a semantic trust boundary ([trusted adapter boundary](../glossary.md#trusted-adapter-boundary)). Authorization semantics on the governed producer path derive only from adapter normalization (ADR-0020 Decision 5). Adapters construct the canonical evidence material and digest the binding and evidence chain depends on (ADR-0007). In the embedded model, the adapter host becomes the enforcement boundary. Adapter interface contracts are a kernel responsibility ([`kernel-boundary-rules.md`](../kernel-boundary-rules.md#allowed-kernel-responsibilities)).

**Evidence for the edge.** Adapters hold no decision authority, no credential, and no network client, and must never gain them (ADR-0010; ADR-0011 Decision 16). They are libraries. Protocol adapters are the first-listed community extension area (strategy §11.3, P-1), and adapter certification is assigned to commercial services ([`basis-ecosystem.md`](basis-ecosystem.md#what-must-stay-outside-basis-core)).

**Inference.** The evidence supports a split that current naming hides: the **adapter contract and normalization semantics** (what any adapter must guarantee) behave like part of the security subsystem, and the **per-protocol implementations** behave like an integration edge where conforming alternatives are expected. The evidence does not settle which side of OD-8's boundary the first-party adapter library belongs on. This is a genuine open seam.

### 9.4 Are schemas an implementation concern of BASIS, or an ecosystem interoperability concern?

**Evidence for BASIS.** Nearly every published or candidate contract is a security contract of L3: decision request and response, audit event, action vocabulary, resource identifier, normalized request, identity context ([`basis-schemas.md`](basis-schemas.md) §3).

**Evidence for Basitra-level.** The repository exists to be neutral and to remove "accidental authority" from any one component. Its ownership model ("Architecture proposes. Schemas publish. Implementations consume.") ties it to L1 governance rather than to a runtime role. Conformance of alternative implementations (strategy §9.3, gap 8; P-1) needs a reference that does not belong to one implementation. Contract identifiers are a compatibility-governed surface that branding cannot change (strategy §12.2).

**Inference.** Schemas are the interoperability surface **of** the security subsystem. Whether that surface is classified inside BASIS or as a Basitra-level interoperability layer depends on whether OD-8 defines BASIS as a runtime subsystem or as an architecture-plus-contracts family. Evidence does not force either answer.

### 9.5 Is the console a BASIS administrative interface, or a broader Basitra administrative surface?

ADR-0023 calls it a "BASIS administrative interface" that exists "for BASIS responsibilities," and lists only BASIS-owned administrative areas (Decision 5). No architecture describes any Basitra component outside the current BASIS family that would need administration. The question of a broader Basitra administrative surface is therefore **hypothetical** on present evidence. It becomes concrete only if OD-8 places some administered capability, such as mapping configuration or category trust policy administration, outside BASIS. The console's status as an optional consumer of the gateway (Design Invariant 10) holds under either answer.

### 9.6 Does deployment package BASIS only, or the broader Basitra ecosystem?

[`basis-ecosystem.md`](basis-ecosystem.md) describes `basis-deploy` as tooling for "packaging, configuring, and distributing the BASIS Core Services Distribution," and as "not part of the authorization runtime." It owns no semantics. Its scope is whatever the distribution contains. If OD-8 narrowed BASIS, deployment tooling would package more than BASIS, which by itself suggests deployment is a distribution-level (Basitra-level) concern. No `basis-deploy` repository exists, so no implementation constrains this.

### 9.7 Which responsibilities share one trust model?

The following trust-model elements are defined once and enforced across several capabilities. Each one crosses at least two of L3a, L3b, and L3c. Not every rule spans all three, and one also reaches outside L3:

| Shared trust-model element | Governing decisions | Capabilities that must apply it |
| - | - | - |
| Three-identity separation (subject, upstream workload, producer workload) and administrative-versus-subject separation | ADR-0008; ADR-0018 Decision 3; ADR-0023 Decisions 3, 5 | CAP-02, CAP-03, CAP-09, CAP-10, CAP-16 |
| Role-specific logical workload identity and credential lifecycle | ADR-0022 | CAP-03, CAP-09, CAP-10 |
| Origin-preserving, category-scoped context trust | ADR-0021 | CAP-05, CAP-09, CAP-10, CAP-07 |
| Identifier ownership and correlation ("no component may overwrite an identifier owned by another component") | ADR-0014; ADR-0018 Decision 5 | CAP-08, CAP-09, CAP-10, CAP-11, CAP-14 |
| Fail-closed at every boundary; silence is not admission | ADR-0011 Decision 9; ADR-0018 Decision 6; ADR-0021 Decision 8 | CAP-03 through CAP-14 |
| Preserved original operation; no reverse mapping | ADR-0011 Decision 7; ADR-0020 Decision 4 | CAP-04, CAP-10, CAP-11, CAP-12 |
| Credential-class separation | ADR-0011 Decision 14; ADR-0018 Decision 8; ADR-0022 Decision 14 | CAP-02, CAP-03, CAP-10, CAP-12 |

This table is the strongest structural evidence in the assessment. Individual rules have different spans:

- workload credential lifecycle and credential-class separation span L3a and L3c;
- identity separation spans L3a and L3c, and also reaches CAP-16 in L5;
- the preserved original operation spans L3b and L3c;
- context trust and identifier ownership span L3b and L3c;
- fail-closed behavior spans all three.

**Collectively**, the rule set couples trust establishment, authorization, and operation and execution governance. No one subdivision can be applied or understood in isolation from the others. That is the evidence on which [Section 6](#6-candidate-architectural-layers) treats L3 as one cohesive subsystem. It is evidence of cohesion, not proof that a subsystem separation is architecturally impossible ([§9.2](#92-is-execution-governance-part-of-the-same-security-subsystem-as-authorization)).

Every capability the table lists is in L3 except CAP-16, the administrative interface (L5). CAP-16 applies the administrative-versus-subject separation as a consumer: it must keep a BASIS administrative context distinct from subject context, but it owns no responsibility on the governed path (ADR-0023 Decision 5).

L4 is not untouched by these rules either. Adapter normalization supplies the semantics and evidence material the binding and evidence rules depend on, and adapters fail closed on normalization failure ([`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §2). But L4 holds none of the identities, credentials, grants, or owned identifiers these rules govern. The cohesion the table shows is the cohesion of the governed-path core, not of every participant in Boundary-Aware Security or every component that touches one of its rules.

### 9.8 Participants in Boundary-Aware Security versus supporters and consumers

| Relationship | Capabilities | Basis |
| - | - | - |
| **Participates:** establishes or verifies a boundary crossing, or records it | CAP-01 through CAP-15 | Strategy §5.3 and §6 (implemented or accepted boundaries) |
| **Supports:** shapes the model without acting on the governed path | CAP-17, CAP-18, CAP-19 | Non-runtime; no authority |
| **Consumes:** uses the model's results or administers it | CAP-16; external evidence consumers | ADR-0023 Decisions 5–7; strategy §6 ("Unrelated" for console presentation) |
| **Submits into the model from outside** | CAP-20, CAP-21 | ADR-0018; ADR-0023 |

The participating set (CAP-01 through CAP-15) is larger than L3 (CAP-01 through CAP-14). It includes CAP-15, protocol normalization and adapter evidence construction, which participates through the protocol boundary while holding no decision authority. Two distinctions therefore apply throughout this assessment:

```text
participates in Boundary-Aware Security   !=  must therefore be inside BASIS
L3 is the cohesive governed-path core     !=  L3 is the complete Boundary-Aware Security model
```

### 9.9 Where the seams actually are

The strategy document frames OD-8 broadly as "the whole component family" versus "a narrower identity and security subsystem" (OD-8). The evidence locates the natural seams somewhat differently:

- **Least natural seam:** between identity/authorization (L3a–L3b) and operation and execution governance (L3c). A subsystem boundary here would sit inside the governed authorization-to-execution lifecycle. The authoritative disposition would cross it to reach the roles that bind and enforce it. Several shared trust rules also span it, and ADR-0023 Decision 1 assigns both sides to BASIS (§9.2, §9.7).
- **Natural seams:** between L3 and non-runtime ecosystem support (L1 governance, L6 deployment). Less sharply, there are seams between L3 and the adjacent layers whose placement the evidence does not settle: L2 interoperability contracts, L4 protocol integration, and L5 administration.

---

## 10. OD-8 Candidate Shapes

### 10.1 The candidates

**Candidate A — BASIS remains the broad component family.** BASIS continues to name every first-party component, including future roles.

```text
Basitra
└── BASIS (component family)
    ├── identity, subject and workload admission
    ├── authorization (kernel, enforcement, composition, context)
    ├── operation and execution governance (intake, producer, binding, executor, evidence)
    ├── protocol adapters
    ├── administrative interface
    ├── shared contracts
    └── deployment tooling
(architecture governance: Basitra / Foundation)
```

**Candidate B — BASIS narrows to the identity and authorization decision subsystem.** BASIS names L3a and L3b. Operation and execution governance, adapters, administration, contracts, and deployment become direct Basitra components.

```text
Basitra
├── BASIS: identity, subject and workload admission, composition,
│          context admission, evaluation, enforcement, authorization evidence
├── operation and execution governance (intake, producer, binding, executor, execution evidence)
├── protocol adapters
├── administrative interface
├── shared contracts
└── deployment tooling
```

**Candidate C — BASIS names the governed security path.** C's fixed center is L3 as a whole: trust establishment, authorization, and operation and execution governance, with evidence as an obligation of each owner. Two layers are Basitra-level on strong evidence: L1 architecture and governance, and L6 deployment and distribution. Neither is part of the runtime, and neither owns security semantics (§6; §7, High confidence). C does **not** decide the three adjacent layers. L2 interoperability contracts (§9.4), L4 protocol integration (§9.3), and L5 administrative and observability surfaces (§9.5) each need an explicit OD-8 placement.

```text
Basitra
├── BASIS: governed security path (L3a + L3b + L3c, with their evidence)   fixed center
│
├── placement to decide in OD-8 (C does not decide these):
│   ├── interoperability contracts (L2)        §9.4, Q-13
│   ├── protocol integration edge (L4)         §9.3, Q-11
│   └── administrative and observability (L5)  §9.5, Q-12
│
├── architecture and governance (L1)            Basitra-level
└── deployment and distribution (L6)            Basitra-level
```

**Candidate C's proposed organizing principle (non-normative).** C needs a membership rule that matches L3 without pulling in its neighbors by default. "Holds or enforces authority" is too narrow: L3 includes authorization evidence, lifecycle observation, and execution evidence, whose owners hold no decision or dispatch authority (§7). The inventory supports this broader formulation:

> BASIS contains the roles that own a security-critical responsibility for a governed operation as it crosses the governed path. That means the roles that establish identity and admission, decide or enforce authorization, preserve and bind the operation, govern its dispatch, or observe and evidence what crossed. They bear those responsibilities directly, under the accepted trust rules of §9.7.

Applied to the inventory, the principle gives these results:

| Capabilities | Result | Why |
| - | - | - |
| L3a–L3c, including evidence and observation (CAP-08, CAP-13, CAP-14) | Included | They own the responsibilities the principle names |
| Supervisory intent and operational state (CAP-20, CAP-21) | Excluded | They originate the operation; they do not govern it |
| Generic packaging and governance (L1, L6) | Excluded | They shape or package the system but own no runtime responsibility for any operation |
| L2, L4, L5 | **Not settled** | Contracts define the shapes that cross boundaries but own no runtime responsibility. Adapters determine the semantics authorization receives and construct evidence material, which the principle could be read to cover, but they are libraries holding no identity, credential, grant, or owned identifier. The administrative interface administers the path but owns no responsibility on it. |

Because the principle leaves L2, L4, and L5 open, it does not pull them into BASIS automatically. That is intended: their placement remains an explicit OD-8 decision. The principle itself is a proposal for OD-8 to accept, refine, or reject ([Q-15](#13-questions-the-od-8-adr-must-answer)).

**Candidate D — BASIS names the first-party reference distribution, not an architectural subsystem.** BASIS is a stewardship and packaging label for the Foundation-maintained implementations. Architectural subsystems are named by responsibility directly under Basitra. This is close to the current canonical wording in [`writing-guidelines.md`](../standards/writing-guidelines.md) §4.1 ("BASIS refers to the open-source core services distribution").

**Considered and weakly supported — BASIS narrows to the identity engine.** The literal expansion "Secure Identity Service" fits `basis-identity` closely. The architecture gives little support for it: the identity engine is optional on the governed path, so the name of the security substrate would attach to a component a conforming deployment may omit; and every accepted statement about what "BASIS" owns (ADR-0018, ADR-0023) would then describe something other than BASIS. It is not analyzed further.

### 10.2 Comparison

| Criterion | A — broad family | B — identity and authorization decision | C — governed security path | D — reference distribution label |
| - | - | - | - | - |
| **Architectural coherence** | Moderate. Coherent as a family; groups runtime security roles with non-runtime support. | Weak-moderate. Cohesive decision core, but it places a subsystem boundary inside the governed authorization-to-execution lifecycle (§9.2, §9.9). | Strong for L3. L2, L4, and L5 placement unresolved. | Moderate. Coherent as stewardship; not an architectural unit. |
| **Security-boundary clarity** | Moderate. "Inside BASIS" does not distinguish authority-holding roles from tooling. | Weak-moderate. Binding creation and verification stay together, outside BASIS. But the authoritative disposition must cross a subsystem boundary to reach the roles that bind and enforce it. Intake, production, binding, and governed dispatch would also sit outside the named security subsystem, although ADR-0018 and ADR-0023 describe them as BASIS-side responsibilities. | Strong for L3. Membership follows the proposed responsibility principle (§10.1), which covers evidence and observation roles as well as decision and enforcement roles. The protocol boundary (L4), which participates without owning a governed-path responsibility in that sense, remains an explicit edge decision, as do L2 and L5. | Weak. Membership is by implementer, not by boundary. |
| **Dependency direction** | Unchanged from today. | Requires stating new inter-subsystem directions. The governed path enters BASIS at producer submission and leaves with the disposition (producer → BASIS gateway; BASIS disposition → binding verification and dispatch). Expressible, but adds a subsystem boundary, crossed twice, on the governed path. | Unchanged inside L3; L2 remains the sink; L6 packages everything; L5 consumes L3. | Unchanged. |
| **Fit with Boundary-Aware Security** | Moderate. Includes the model plus non-participants. | Weak-moderate. Leaves intake, binding, and execution crossings, three of the catalog's accepted boundaries, outside BASIS. | Strong for the governed-path core. Covers every implemented or accepted catalog boundary whose responsibilities L3 roles own (strategy §5.3). The catalog's protocol boundary (L4) is covered only if OD-8 places L4 inside BASIS. | Weak. Not defined by the model. |
| **Fit with the forward BASIS expansion** | Weak-moderate (see [Section 11](#11-the-identity-in-the-basis-expansion)). | Strong under a narrow reading of "Identity". | Moderate-strong under the broader, identity-propagation reading. | Weak. A distribution is not a "service." |
| **Compatibility implications** | Lowest. No accepted ADR text changes meaning. | Highest. ADR-0010's "BASIS Producer" and distribution membership, ADR-0018's "BASIS-side" roles, and ADR-0023 Decisions 1 and 4 would describe a substrate larger than BASIS. | Low-moderate. Accepted ADR statements about "BASIS" remain true of L3. Changes concentrate at the adjacent layers (L2, L4, L5, depending on placement) and in [`basis-ecosystem.md`](basis-ecosystem.md). | Moderate. ADR-0018 and ADR-0023 use BASIS for roles any conforming implementation may fill (§5). |
| **Effect on accepted ADRs** | None required. | Likely a terminology statement or supersession touching ADR-0010, ADR-0018, ADR-0023. | None required for L3. ADR-0010's distribution-membership sentence may need interpretation if the distribution concept changes. | Interpretation needed wherever ADRs use BASIS for roles. |
| **Conceptual simplicity** | Simple to state; least informative. | Simple to state; hard to apply on the governed path. | Moderate: one responsibility principle (§10.1) plus explicit placements for L2, L4, and L5. | Simple; conflates role and implementation. |
| **Future extensibility** | New roles join BASIS by default, whatever they are. | Execution-side growth (separated executor, intake realization) happens outside BASIS. | New governed-path roles join BASIS by the same principle; new packaging and governance tooling joins Basitra; new contracts, adapters, or administrative surfaces follow whatever placement OD-8 decides for L2, L4, and L5. | Third-party conforming implementations are never BASIS. |
| **Risk of misleading names** | Moderate: "Service" and "Identity" describe non-runtime members poorly. | Moderate: BASIS would not name the system ADR-0023 calls BASIS. | Low-moderate: residual question of whether "Identity" reads as narrow. | Moderate-high: "Service" for a distribution; role/implementation conflation. |

### 10.3 Observations

- **A and C differ mainly outside L3.** C fixes L3 inside BASIS and L1 and L6 outside it, and leaves L2, L4, and L5 open. Depending on how OD-8 places those three, C's membership ranges from L3 alone to A's membership minus L1 and L6. A and C differ in the support and adjacent layers. B and C differ inside the governed path itself.
- **B's difficulty is structural, not terminological.** B does not split binding creation from verification. It turns the *authorization ≠ execution* lifecycle boundary into a subsystem boundary placed immediately after the authorization decision, so the authoritative disposition crosses from BASIS into separately classified operation and execution governance (§9.2). B remains viable. It is less favored because the accepted record assigns binding and governed dispatch to BASIS (ADR-0023 Decision 1; ADR-0018), and because several shared trust rules span the boundary B would draw (§9.7). Adopting B would require the OD-8 ADR to justify that subsystem boundary and to state how those accepted statements are read afterward.
- **D is coherent but answers a different question.** It names who maintains what, not what the architecture is. It could coexist with C if OD-8 chose to keep a distribution term alongside an architectural one. See [Q-2](#13-questions-the-od-8-adr-must-answer).
- **The evidence does not support any particular placement of L2, L4, or L5.** Whichever candidate OD-8 adopts must state those placements explicitly and justify them.

---

## 11. The "Identity" in the BASIS Expansion

ADR-0019 Decision 3 adopts *Boundary-Aware Secure Identity Service* as the forward expansion. This section asks whether "Identity" (and "Service") strains under a broad BASIS. It assumes neither answer.

**Two readings of "Identity" exist in the record.**

- **Narrow reading: identity as a service type.** An "identity service" is an identity provider or broker. This reading fits CAP-01 well and most other capabilities poorly. Policy evaluation, protocol normalization, execution, and deployment are not identity services in this sense.
- **Broad reading: identity as the organizing principle of the security model.** The repository's established framing is [identity-aware authorization](../glossary.md#identity-aware-authorization): decisions conditioned on verified identity rather than network location. [Principle 2](../architecture-principles.md#2-identity-propagation-over-network-origin-trust) requires identity propagation across intermediaries. The accepted ADR chain keeps distinct identities distinct across every boundary (§9.7). The `Planned` [identity-to-operation contract roadmap](../roadmaps/identity-to-operation-contract-and-interoperability.md) describes the intended direction as an account of "who acted, whose authority was exercised, what was requested, what was decided, what was enforced, whether execution occurred." That roadmap is planned, not decided; it is cited only as evidence that the broad reading already exists in the repository's own vocabulary.

**Assessment by capability, under each reading.**

| If BASIS includes… | Narrow reading | Broad reading |
| - | - | - |
| Identity federation, subject and workload admission (L3a) | Accurate | Accurate |
| Evaluation, enforcement, composition, context admission (L3b) | Strained | Accurate: authorization conditioned on established identity |
| Intake, production, binding, governed execution, evidence (L3c) | Strained | Mostly accurate: the binding ties an identity-attributed authorization to the exact dispatched operation; evidence attributes each crossing |
| Protocol-specific dispatch mechanics | Inaccurate | Strained: dispatch itself has no identity function |
| Protocol adapters | Inaccurate | Strained: normalization is semantic, not identity-related, although it feeds attributed evidence |
| Administrative interface | Inaccurate | Strained: administers the identity-aware system, holds administrative context only |
| Shared contracts | Inaccurate | Strained: includes identity contracts and much else |
| Deployment tooling | Inaccurate | Inaccurate |
| Architecture governance | Inaccurate | Inaccurate |

**"Service" is the second word under strain.** A *service* suggests a runtime system. Under any reading, it describes L3 better than it describes libraries, contracts, tooling, or documentation.

**The tension is not new.** *Building Automation Secure Identity Service* already named a family that included adapters, a console, schemas, and planned deployment tooling. The acronym has always been looser than its expansion. A broad BASIS under the new expansion would continue that, not introduce it.

**Inference.** Under the broad reading, the forward expansion describes L3, the governed security path, with reasonable accuracy, and describes non-runtime support poorly. Under the narrow reading, it describes only L3a. Whether "Boundary-Aware Secure Identity Service" should be read narrowly or broadly is itself a question the OD-8 ADR should answer explicitly ([Q-8](#13-questions-the-od-8-adr-must-answer)), because the answer changes which candidate the expansion supports.

---

## 12. Accepted-ADR and Canonical-Document Constraints

### 12.1 Accepted ADRs

Classifications: **semantic** (fixes meaning that any outcome must preserve), **component-boundary** (fixes which role owns what), **repository-placement** (fixes where an implementation lives), **name** (fixes a name), **compatibility** (fixes a published surface), **historical only** (no forward constraint on OD-8). Nothing here is superseded.

| ADR | What it fixes that bears on OD-8 or OD-9 | Classification | Effect on candidate outcomes |
| - | - | - | - |
| [ADR-0007](../adr/0007-adapter-evidence-construction.md) | Adapters construct and digest evidence; the producer retains and mints `reference_id`; retain-before-mint | Semantic; component-boundary | Unaffected by any candidate. If L4 sits outside BASIS, the adapter-to-producer dependency becomes cross-subsystem but unchanged. |
| [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) | mTLS producer profile; producer ≠ subject; gateway-derived URI SAN identity; no BASIS-specific URI namespace ("this ADR does not invent one") | Semantic; component-boundary; compatibility | Unaffected. Because the URI SAN namespace is deployment-defined, a component rename does not alter producer identity strings by architecture. |
| [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) | Trusted ingress plus gateway certificate handoff | Component-boundary; compatibility | Unaffected. |
| [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md) | Permanent component name `basis-producer`, repository `basis-foundation/basis-producer`, package `basis_producer`, architectural name "BASIS Producer"; distribution membership ("a component of the BASIS Core Services Distribution"); ownership list; role distinct from implementation | **Name**; repository-placement; component-boundary; compatibility (package namespace) | Any rename needs a superseding ADR (ADR-0019 **Consequences**). Under B, "BASIS Producer" would name a non-BASIS component. Under D, the distribution sentence is central. Under A and C, unaffected by OD-8 itself; OD-9 may still revisit the name. |
| [ADR-0011](../adr/0011-protocol-execution-role-and-bounded-reference-topology.md) | Distinct executor role; not in adapters, gateway, or kernel; first slice same-process in `basis-producer`; no `basis-executor` and no reserved name | Semantic; component-boundary; repository-placement (first slice) | Under B, the executor sits outside BASIS while its invariants depend on a BASIS disposition. Under A and C, unaffected. |
| [ADR-0012](../adr/0012-authorization-to-execution-binding.md) | Producer creates and executor verifies a same-process binding record | Semantic; component-boundary | Under B, creation and verification both fall outside BASIS and the bound disposition inside it (§9.2). |
| [ADR-0013](../adr/0013-execution-lifecycle-semantics.md) | Lifecycle model; executor owns boundary-entered observation | Semantic | Unaffected by naming; relevant to where L3c is placed. |
| [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md) | Distinct execution-evidence-producer role; first-slice retention colocated in `basis-producer`; "no existing repository gains implementation responsibility" | Semantic; component-boundary; repository-placement (first slice) | As ADR-0013. |
| [ADR-0015](../adr/0015-first-bounded-execution-target.md) | First target hosted within `basis-producer`'s process | Repository-placement (bounded) | Unaffected by OD-8; bounded to the first slice. |
| [ADR-0016](../adr/0016-bounded-target-replay-freshness-posture.md) | Replay/freshness posture for the first target only | Semantic (scoped) | No naming relevance. |
| [ADR-0017](../adr/0017-action-vocabulary-naming-structure.md) | Five canonical verbs; deprecated aliases remain accepted; credential-management operations belong to "a separate identity/administration vocabulary" | Semantic; compatibility | Unaffected. The identity/administration vocabulary remark is relevant if OD-8 separates identity administration. |
| [ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md) | Intake boundary "belongs to the BASIS side"; three identities; producer and executor "remain BASIS-side roles"; platform neutrality | Semantic; component-boundary | Any outcome must keep intake, producer, and executor on the governed side. Under B, "BASIS side" would need a terminology statement identifying the governed side. |
| [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) | Basitra names the project, not a component; BASIS ≠ `basis-core`; BASIS is a standalone acronym; OD-8 precedes OD-9; history is not rewritten; renames need migration plans | Semantic; name | Binds every candidate. Rules out redefining BASIS as `basis-core` and naming a component "Basitra." |
| [ADR-0020](../adr/0020-operation-to-authorization-mapping-and-composition-boundary.md) | Gateway is sole composer on the governed producer path; producer captures mapping identity; deployment-designated configuration authority; composition rule a future `basis-schemas` candidate | Component-boundary; semantic; compatibility (future) | Unaffected for L3. The configuration authority's administrative surface is an unplaced capability (§4.3, finding 2). |
| [ADR-0021](../adr/0021-upstream-context-assertion-trust-boundary.md) | Enforcement placement across intake, producer, gateway, kernel; kernel stays neutral | Component-boundary; semantic | Spans L3a–L3c. Under B, part of its placement table would sit outside BASIS. |
| [ADR-0022](../adr/0022-workload-credential-lifecycle-and-scope-boundary.md) | Logical workload identity is role-specific; implementation upgrade or conforming replacement does not by itself change identity (Decision 11) | Semantic; compatibility | Inference: a repository or component rename that leaves role and scope unchanged would not by itself require re-admission. The ADR addresses replacement and upgrade, not renaming as such; a migration plan should confirm this. |
| [ADR-0023](../adr/0023-supervisory-platform-and-administrative-interface-boundary.md) | "BASIS is an authorization and security substrate… not a supervisory platform"; what BASIS independently establishes (Decision 4); administrative context only; BASIS-native tools consume, never bypass; states it is "about architectural roles, not names" | Semantic; component-boundary | Not a name constraint by its own terms. Its Decisions 1 and 4 are the closest accepted description of BASIS by responsibility, and their listed responsibilities fall within L3. Any outcome must say which term carries its meaning. |

ADR-0001 through ADR-0006 are `Proposed` and are not listed as accepted constraints. The kernel isolation they assume is independently governed by [`kernel-boundary-rules.md`](../kernel-boundary-rules.md).

### 12.2 Canonical documents that are not ADRs

These do not require supersession to change, but they encode current naming and placement and would need bounded follow-on updates under some outcomes.

| Document | What it currently fixes | Classification |
| - | - | - |
| [`basis-ecosystem.md`](basis-ecosystem.md) | Three-layer ecosystem; repository table ("Each component… expected to be maintained in its own repository"); dependency rules | Component-boundary; repository-placement |
| [`kernel-boundary-rules.md`](../kernel-boundary-rules.md) | Kernel isolation invariants; import rules naming `basis_core.enforcement` | Component-boundary; compatibility |
| [`compatibility-philosophy.md`](compatibility-philosophy.md) | Import-namespace and normalization changes are breaking | Compatibility |
| [`basis-schemas.md`](basis-schemas.md); [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md) | Reserved `basis_gateway.*` evidence namespace carries a current component name | Compatibility (OD-9/OD-10 concern, not OD-8) |
| [`writing-guidelines.md`](../standards/writing-guidelines.md) §4.1 | "BASIS refers to the open-source core services distribution" | Current canonical text, eligible for an ADR-0019 Decision 8 follow-on |
| [`glossary.md`](../glossary.md#basis-core-services-distribution) | Distribution membership; role substitution | Semantic (substitution); name (membership list) |

---

## 13. Questions the OD-8 ADR Must Answer

The future OD-8 ADR should answer at least the following. Q-1 through Q-10 restate the questions this assessment was asked to prepare; Q-11 onward arise from the evidence above.

| ID | Question |
| - | - |
| Q-1 | What exactly does BASIS name inside Basitra? |
| Q-2 | Is BASIS the whole security subsystem, a narrower identity and authorization subsystem, another bounded subsystem, or a distribution label? Does it name an **architectural subsystem**, a **first-party distribution**, or both, and if both, do they coincide? |
| Q-3 | Which capabilities ([Section 4](#4-inventory-of-durable-architectural-capabilities)) are inside BASIS? |
| Q-4 | Which capabilities are outside BASIS but inside Basitra? |
| Q-5 | Does BASIS membership attach to **roles** or to **first-party implementations**? Is a conforming third-party producer, adapter, or executor "BASIS"? |
| Q-6 | Where are the subsystem boundaries, and is the authorization-to-execution binding inside one subsystem or across two? |
| Q-7 | What dependency direction applies across those boundaries, separately for build dependency and governed-operation flow? |
| Q-8 | What does "Boundary-Aware Secure Identity Service" accurately describe, and is "Identity" to be read narrowly (identity service) or broadly (identity-aware security model)? |
| Q-9 | Which accepted ADRs are affected by the chosen boundary, and does the ADR state how their uses of "BASIS" are read afterward? In particular: ADR-0010 ("BASIS Producer," distribution membership), ADR-0018 ("BASIS side"), ADR-0023 Decisions 1 and 4. |
| Q-10 | Which questions are explicitly deferred to OD-9, OD-10, OD-1, and OD-7? |
| Q-11 | Where does the protocol-integration edge sit: is the adapter **contract** treated differently from per-protocol **implementations**? |
| Q-12 | Where does the administrative interface sit, and where do the currently unplaced administrative surfaces (mapping configuration, category trust policy) sit? |
| Q-13 | Are shared contracts part of BASIS or a Basitra-level interoperability layer, and does that change who governs contract compatibility? |
| Q-14 | Does the term "BASIS Core Services Distribution" survive, and what is its relationship to BASIS and to any Basitra distribution? |
| Q-15 | Do future roles default into BASIS by a responsibility rule? One example is the principle proposed for Candidate C (§10.1): BASIS contains the roles that own a security-critical responsibility for a governed operation, namely establishing identity and admission, deciding or enforcing authorization, preserving and binding the operation, governing its dispatch, or observing and evidencing what crossed. If such a rule is adopted, how does it treat responsibilities that hold no decision authority, and how does it treat L2, L4, and L5? The question covers an intake realization, a separated executor, and protocol-specific executors. |
| Q-16 | Does the decision preserve, unchanged, the Ipotio boundary and the invariants in [Section 8](#8-the-ipotio-boundary-is-preserved), and ADR-0023's statement that BASIS is not a supervisory platform? |
| Q-17 | Does the decision avoid giving BASac any meaning, consistent with ADR-0019 Decision 5? |
| Q-18 | Does the decision need any supersession, or only a terminology-continuity statement? |

**Deferred to OD-9 and later, not OD-8:** repository names; whether `basis-producer`'s name still answers the Stage 4 question once its hosted roles are considered (§4.3, finding 1); package and import namespaces (OD-10); the `basis_gateway.*` evidence namespace; the GitHub organization (OD-7); the Foundation relationship (OD-1); Ipotio terminology reconciliation (OD-3).

---

## 14. Recommendation to the Future OD-8 ADR (Non-Normative)

This section is a recommendation to the future OD-8 ADR. It is not a decision, and nothing in it defines BASIS.

**The evidence in this assessment suggests that Candidate C — BASIS names the governed security path (L3), with the placement of L2, L4, and L5 left to explicit OD-8 decisions — should be the preferred starting point for the OD-8 decision.**

The reasons, in order of weight:

1. **The accepted architecture already describes BASIS by responsibility, and that description closely matches L3.** ADR-0023 Decision 1 lists what BASIS owns, and Decision 4 lists what BASIS independently establishes. Both lists fall within L3a–L3c and include no deployment tooling, contract publication, or governance. Neither list names protocol normalization, contract publication, or the administrative interface as a responsibility BASIS owns on the governed path. ADR-0023 therefore leaves the placement of L4, L2, and L5 open: it neither includes nor excludes them. ADR-0023 also states that it is about roles, not names, which is what makes its description usable as evidence for OD-8.
2. **The accepted cross-cutting trust rules collectively couple L3** (§9.7). Each rule spans at least two of L3a, L3b, and L3c, though not all span all three. Taken together, they couple trust establishment, authorization, and operation and execution governance. None reaches into L1, L2, or L6. One reaches CAP-16 (L5) as a consumer. L4 supplies inputs the rules depend on without holding the identities, credentials, or grants they govern.
3. **L3 contains every implemented or accepted Boundary-Aware Security boundary whose responsibilities a governed-path role owns** (strategy §5.3). It does not contain all of the catalog's boundaries: the protocol boundary is also implemented, and its normalization and evidence construction sit in L4. Candidate C's proposed organizing principle (§10.1) would make "inside BASIS" mean "owns a security-critical responsibility for a governed operation on the governed path." That covers evidence and observation roles as well as decision and enforcement roles. L4 participates in Boundary-Aware Security, but participation alone does not decide BASIS membership. That remains an explicit OD-8 edge-placement decision (Q-11), as do L2 (Q-13) and L5 (Q-12).
4. **C preserves accepted ADR meaning with the least reinterpretation.** Accepted statements about what "BASIS" owns remain true. ADR-0018's "BASIS side" and ADR-0023's "substrate" still identify the same thing.
5. **C fits the forward expansion under its broad reading** (§11) better than A does, and avoids B's structural difficulty (§10.3).

The evidence is **not** strong enough to settle the following, and the OD-8 ADR should decide each explicitly rather than inherit an answer from this recommendation:

- placement of the protocol-integration edge (§9.3; Q-11);
- placement of the administrative interface and the unplaced administrative surfaces (§9.5; Q-12);
- placement of shared contracts (§9.4; Q-13);
- whether a distribution term in the sense of Candidate D is kept alongside C (Q-2, Q-14).

**Candidate A remains viable** and has the lowest compatibility cost. The case against it as the starting point is that its membership is derived from which first-party repositories exist rather than from responsibility, which ADR-0019 Decision 7 cautions against, and that "Service" and "Identity" describe its non-runtime members poorly. **Candidate B remains viable but is not recommended** as a starting point. It does not split binding creation from verification. It does place a subsystem boundary inside the governed authorization-to-execution lifecycle, immediately after the authorization decision. That separates the authoritative disposition from the roles that bind and enforce it, contrary to ADR-0023 Decision 1 and ADR-0018's placement of those roles on the BASIS side (§9.2). **Candidate D** answers a stewardship question rather than an architectural one.

**Confidence.** High that the accepted architecture provides substantial evidence of L3's cohesion. Moderate that BASIS should name L3, and that the responsibility principle in §10.1 is the right membership test. Low on the placement of L2, L4, and L5. One further architecture clarification is not required to begin drafting the OD-8 ADR from this evidence, but the ADR will need to reach its own conclusions on Q-11 through Q-14.

---

## 15. What This Assessment Does Not Decide

This assessment does not:

- resolve OD-8 or OD-9, or create the OD-8 ADR;
- define the long-term scope of BASIS;
- choose, rename, or reserve any repository, component, package, import namespace, GitHub organization, foundation name, or website;
- supersede, amend, or reinterpret any accepted ADR;
- modify any implementation repository, schema, or contract;
- create any new runtime responsibility, role, or boundary; the capabilities in [Section 4](#4-inventory-of-durable-architectural-capabilities) are inventoried from existing architecture, and the layers in [Section 6](#6-candidate-architectural-layers) are an analytical grouping, not accepted architecture;
- give BASac any meaning;
- reopen, redesign, or extend the Ipotio boundary, or give any supervisory platform special trust;
- correct the documentation inconsistencies it records (§4.3, finding 5);
- authorize implementation or migration.

---

## 16. References

- [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md): Basitra ecosystem identity and terminology hierarchy (Accepted)
- [`basitra-ecosystem-and-boundary-aware-security.md`](basitra-ecosystem-and-boundary-aware-security.md): strategy document; §5.3, §6, §9.3, §11.3, §12.4, §13
- [`basis-ecosystem.md`](basis-ecosystem.md), [`reference-vs-implementation.md`](reference-vs-implementation.md), [`compatibility-philosophy.md`](compatibility-philosophy.md)
- [`basis-identity.md`](basis-identity.md), [`identity-authority-modes.md`](identity-authority-modes.md), [`basis-gateway.md`](basis-gateway.md), [`basis-adapters.md`](basis-adapters.md), [`basis-console.md`](basis-console.md), [`basis-schemas.md`](basis-schemas.md), [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md)
- [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md), [`execution-boundary-discovery-assessment.md`](execution-boundary-discovery-assessment.md) (methodological precedent), [`bounded-rest-execution-implementation-plan.md`](bounded-rest-execution-implementation-plan.md)
- [`kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`architecture-principles.md`](../architecture-principles.md), [`glossary.md`](../glossary.md), [`writing-guidelines.md`](../standards/writing-guidelines.md)
- [ADR-0007](../adr/0007-adapter-evidence-construction.md), [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md), [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md), [ADR-0011](../adr/0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](../adr/0012-authorization-to-execution-binding.md), [ADR-0013](../adr/0013-execution-lifecycle-semantics.md), [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md), [ADR-0015](../adr/0015-first-bounded-execution-target.md), [ADR-0016](../adr/0016-bounded-target-replay-freshness-posture.md), [ADR-0017](../adr/0017-action-vocabulary-naming-structure.md), [ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md), [ADR-0020](../adr/0020-operation-to-authorization-mapping-and-composition-boundary.md), [ADR-0021](../adr/0021-upstream-context-assertion-trust-boundary.md), [ADR-0022](../adr/0022-workload-credential-lifecycle-and-scope-boundary.md), [ADR-0023](../adr/0023-supervisory-platform-and-administrative-interface-boundary.md)
- [ADR-0001](../adr/0001-operation-aware-ot-authorization.md) through [ADR-0006](../adr/0006-evaluation-orchestration-layer.md) (`Proposed`; implemented kernel model)
- [`identity-to-operation-contract-and-interoperability.md`](../roadmaps/identity-to-operation-contract-and-interoperability.md) (Planned; cited only for vocabulary)
- [`README.md`](../../README.md), [`ROADMAP.md`](../../ROADMAP.md), [`GOVERNANCE.md`](../../GOVERNANCE.md)
- Ipotio architecture repository: upstream supervisory platform (cross-repository reference, not hyperlinked)
