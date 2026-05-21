# Identity-Aware Authorization for Operational Technology

**Status:** Draft — active development
**Repository:** `basis-architecture`
**Last structural revision:** See git log

---

## Abstract

Operational technology environments — building automation systems, industrial control networks, campus infrastructure — have historically relied on network segmentation and protocol membership as the primary mechanisms for access control. These mechanisms were appropriate for the environments in which they were designed: physically bounded, protocol-homogeneous networks with limited external connectivity.

The conditions that made those mechanisms sufficient have changed. OT networks are increasingly connected to enterprise IT infrastructure, cloud-hosted management platforms, and remote access services. The implicit trust assumptions built into field protocols and VLAN-based access models are no longer adequate for the access control and accountability requirements of modern OT deployments.

This paper examines how identity-aware authorization — the use of verified identity context and centralized policy evaluation to make access decisions — can be applied to OT environments. It describes the architectural patterns required to do so, the operational constraints that complicate implementation, the threats that the authorization layer addresses and the ones it does not, and the directions in which this problem space is likely to develop.

The paper is grounded in the BASIS (Building Automation Secure Identity Service) proof-of-concept, which was developed to validate the feasibility of these patterns in a realistic BAS context. BASIS is a research artifact, not a production system. The architectural patterns described here are intended to be applicable beyond the specific implementation choices made in the PoC.

---

## Table of Contents

| # | Section | File | Status |
|---|---|---|---|
| 01 | [Non-Goals](sections/01-non-goals.md) | `sections/01-non-goals.md` | Complete |
| 02 | [Current State of OT Authorization](sections/02-current-state-of-ot-authorization.md) | `sections/02-current-state-of-ot-authorization.md` | Complete |
| 03 | [Why Existing Approaches Become Insufficient](sections/03-why-existing-approaches-are-incomplete.md) | `sections/03-why-existing-approaches-are-incomplete.md` | Complete |
| 04 | [Identity-Aware OT Architecture](sections/04-identity-aware-ot-architecture.md) | `sections/04-identity-aware-ot-architecture.md` | Complete |
| 05 | [OT Trust Boundaries](sections/05-ot-trust-boundaries.md) | `sections/05-ot-trust-boundaries.md` | Complete |
| 06 | [The BASIS Proof-of-Concept](sections/06-basis-proof-of-concept.md) | `sections/06-basis-proof-of-concept.md` | Mature Draft |
| 07 | [Production Realities and Constraints](sections/07-production-realities-and-constraints.md) | `sections/07-production-realities-and-constraints.md` | Mature Draft |
| 08 | [Threat Modeling and Security Considerations](sections/08-threat-modeling-and-security-considerations.md) | `sections/08-threat-modeling-and-security-considerations.md` | Mature Draft |
| 09 | Future Direction | `sections/09-future-directions.md` | Not started |
| 10 | Conclusion | — | Not started |
| — | [References](references/sources.md) | `references/sources.md` | In progress |

---

## Introduction and Motivation

Authorization in operational technology environments is not a solved problem. It is not even, in most deployments, a well-posed one. The access control mechanisms in widespread use — VLAN membership, network segmentation, controller-local passwords, shared operator credentials — were not designed as authorization systems. They evolved as practical constraints on communication within bounded physical networks, and they function reasonably well within those bounds.

The problem is that the bounds themselves have changed. A building automation system that was once operated by a single team of technicians with physical access to a dedicated network is now expected to integrate with enterprise energy management platforms, expose operational data to remote analytics services, support vendor remote access for maintenance, and provide audit records sufficient for compliance reporting. None of these requirements were contemplated in the design of BACnet or Modbus. None of them are well-served by VLAN-based trust models.

Identity-aware authorization offers a different approach: rather than treating network location as a proxy for trust, verify the identity of every principal — operator, device, service — and evaluate every access request against explicitly defined policy. This approach is well-established in enterprise IT and cloud environments. Its application to OT is less mature, and for good reason: OT environments carry constraints that do not exist in IT contexts.

Field devices may have no computational capacity for authentication protocol participation. Field protocols may have no message structure that supports attaching identity context. Maintenance windows are narrow and infrequent. Controllers that have operated without interruption for years cannot be taken offline for software upgrades. Operational continuity is a safety requirement, not merely an operational preference. Any authorization architecture for OT environments must be designed with these realities as first-order constraints, not as edge cases to be addressed later.

This paper describes what identity-aware authorization looks like when designed with those constraints in mind. It does not claim to have resolved all of them. It documents where they bite, where they require architectural accommodations, and where they remain open engineering problems.

---

## Section Summaries

### 01 — Non-Goals

*[sections/01-non-goals.md](sections/01-non-goals.md)*

Establishes explicit scope boundaries for the paper. Clarifies that the work does not propose replacing OT protocols, does not argue for eliminating network segmentation, does not claim BASIS is production-ready, does not assume cloud-only deployment, and does not claim identity-aware authorization alone resolves OT security. These are not aspirational limitations — they are deliberate scope decisions.

---

### 02 — Current State of OT Authorization

*[sections/02-current-state-of-ot-authorization.md](sections/02-current-state-of-ot-authorization.md)*

Describes the authorization landscape in OT environments as it exists today: network segmentation, VLAN-based trust, VPN and jump host access patterns, controller-local credentials, shared operator accounts, vendor-specific trust models, and protocol-centric security assumptions. The section is analytical rather than prescriptive — it explains how current approaches evolved historically and identifies the operational and architectural conditions under which they become insufficient, without characterizing the current state as negligent or broken.

---

### 03 — Why Existing Approaches Become Insufficient

*[sections/03-why-existing-approaches-are-incomplete.md](sections/03-why-existing-approaches-are-incomplete.md)*

Examines the structural limitations of network-centric trust, controller-local authorization, and distributed authorization logic as OT environments become more interconnected, remotely managed, and subject to audit and accountability requirements. Topics include the coarse granularity of network-based access control, the distinction between authentication and authorization, the operational scaling challenge of maintaining fragmented policy, inconsistent and incomplete auditability, and the inability to centrally reason about access decisions across heterogeneous systems. Closes by introducing the framing that authorization is increasingly a system-level concern rather than a protocol-level one — the architectural premise that Section 04 builds on.

---

### 04 — Identity-Aware OT Architecture

*[sections/04-identity-aware-ot-architecture.md](sections/04-identity-aware-ot-architecture.md)*

The conceptual center of the paper. Establishes authorization as a system-level concern, then builds the architecture from first principles. Covers the three-function separation of identity verification, policy evaluation, and enforcement; subjects, resources, and actions as authorization primitives; identity propagation across trust boundaries; how protocol adapters normalize field-protocol semantics into the shared authorization model; why distributed enforcement can coexist with centralized policy through local policy caching; control plane and data plane as distinct architectural concerns with distinct traffic paths; centralized auditability as a first-class property; offline resilience design and the policy cache staleness tradeoff; and context-aware authorization with its associated complexity costs. Integrates directly with the authorization flow diagram. Closes with an accounting of architectural constraints — latency, legacy device coverage gaps, operational complexity, partial deployment realities, and new failure modes introduced by the authorization infrastructure itself. Sets up Section 05's analysis of where and how trust boundaries are enforced in practice.

---

### 05 — OT Trust Boundaries

*[sections/05-ot-trust-boundaries.md](sections/05-ot-trust-boundaries.md)*

Extends the conceptual architecture from Section 04 with analysis of where trust boundaries exist in OT environments, why they matter operationally, and what enforcement at each boundary requires in practice. Covers the trust boundary taxonomy for BAS and broader OT deployments; enforcement at the operations zone boundary (jump host as ingress enforcement point); enforcement at the edge zone boundary (local enforcement point, protocol normalization, local policy cache); the field device zone as an enforcement-downstream zone; vendor access and cross-boundary flows; and asymmetric trust requirements for inbound commands versus outbound telemetry.

---

### 06 — The BASIS Proof-of-Concept

*[sections/06-basis-proof-of-concept.md](sections/06-basis-proof-of-concept.md)*

Documents the BASIS PoC as a research artifact: its scope, implementation choices, validated claims, and explicitly unvalidated properties. The PoC was scoped breadth-first — designed to exercise the full path from operator authentication through policy evaluation, enforcement, protocol dispatch, and audit recording at limited scope, using simulated OT resources rather than physical hardware, with reproducibility as an explicit design constraint.

The implementation is organized into five layers — operator, identity and access, authorization and API, protocol adapter, and simulated OT resources — all running within a single Docker Compose stack. Keycloak serves as the identity provider issuing OIDC tokens. A FastAPI application combines the enforcement point and policy engine, implementing RBAC through a `PolicyEngine` with `RoleBasedPolicy`. Two protocol adapters — MQTT and Modbus TCP — implement a shared `AdapterBase` interface, each translating the authorization model's resource-action vocabulary into protocol-specific operations while keeping the policy evaluation path protocol-agnostic. A `DualAuditStore` records authorization decisions and dispatch outcomes to both SQLite and structured stdout.

The section identifies three key architectural observations embedded in the implementation: that adapter normalization is itself a trust boundary whose correctness the policy engine cannot independently verify; that the asymmetric treatment of inbound commands and outbound telemetry is structurally distinct and must be represented as such; and that the PoC's monolithic collapse of identity resolution, policy evaluation, and enforcement into a single process validates the logical relationships between these concerns without validating the operational separation that the conceptual architecture's resilience properties require.

**What the PoC demonstrated:** Identity propagation from authentication through policy evaluation to audit record as a continuous chain; structural enforcement at an API boundary that cannot be bypassed by new endpoint additions; protocol-agnostic authorization through the adapter abstraction (adding the Modbus adapter required no changes to authorization or audit logic); dual-event audit structure without request-path failure modes; authenticated real-time telemetry delivery.

**What the PoC did not demonstrate:** Distributed policy evaluation and local policy cache operation; production-scale OT resource coverage; operational latency under realistic command volumes; high availability; real industrial protocol handling; safety constraint interaction; or production security hardening.

The section closes with lessons from the implementation: that the enforcement point's correctness depends on semantic accuracy of upstream subject and resource resolution, not only structural gate behavior; that audit schema design requires operational input that a research implementation cannot supply; and that reproducibility as a design constraint surfaces assumptions that would otherwise remain implicit.

---

### 07 — Production Realities and Constraints

*[sections/07-production-realities-and-constraints.md](sections/07-production-realities-and-constraints.md)*

Takes up the operational engineering challenges that the conceptual architecture defers, arguing that the distance between a coherent architecture and a deployable production system is primarily a gap in operational discipline rather than technical sophistication. The section proceeds through a series of interconnected challenge domains, each of which surfaces tradeoffs the architecture cannot resolve and that each deployment must negotiate explicitly.

**High availability and operational continuity.** The identity provider and policy engine are the components that enforcement points depend on for session establishment and policy synchronization. HA design for these components is non-trivial: active-active configurations introduce distributed coordination problems, and the local policy cache addresses only one of several availability dependencies. Availability requirements for the authorization layer must be developed in coordination with the operational continuity requirements of the systems it governs, not derived independently from IT infrastructure norms.

**Offline authorization and policy distribution.** The local policy cache requires a defined distribution format, delivery mechanism, and consistency model. Edge sites may connect over narrow, unreliable, or metered links that enterprise-oriented policy distribution tooling does not anticipate. Partial policy delivery without transactional guarantees can produce enforcement behavior consistent with neither the old policy nor the new policy as coherent wholes. The staleness limit — the maximum age at which cached policy is considered current — must be set with explicit awareness of its implications in both directions: too short causes enforcement points to reach their limit during normal operations; too long extends the window during which revoked credentials remain honored.

**Policy cache invalidation and synchronization.** Credential revocation propagation is the most operationally significant invalidation scenario: a revoked credential may remain valid at cached enforcement points until the next successful synchronization. Forcing immediate invalidation in a partially disconnected environment is not straightforward and requires architectural mechanisms that must be designed and tested before they are needed. At fleet scale, the deployment will persistently operate in a state of bounded inconsistency across enforcement points — different sites enforcing different policy vintages simultaneously. This is not a failure condition; it is the normal operational state of a distributed authorization system with imperfect connectivity, and policy changes must be designed to be safe during the propagation window, not only after full propagation has completed.

**Latency and deterministic operations.** The architecture's enforcement placement strategy accommodates the spectrum of OT latency requirements by positioning enforcement at trust boundary crossings rather than inside control loops. However, policy complexity growth over time may push evaluation latency beyond initial budgets, and context-aware policy that produces different decisions based on operational state introduces non-determinism that OT operators are not accustomed to from the field systems they govern.

**Fail-open vs fail-closed tradeoffs.** Fail behavior under enforcement point failure is a site-specific tradeoff between security risk and operational risk, not a security question with a universal correct answer. Intermediate degraded operation modes — permitting reads while denying writes, for example — may better reflect operational reality than binary all-or-nothing behavior, but they push semantic classification logic into the failure path. Critically, the organizational question of who has authority to approve a fail-open exception must be resolved before deployment, not during an incident.

**Emergency override and supervisory control.** Emergency procedures that effectively disable authorization undermine audit coverage precisely when the system's behavior is most operationally significant. Autonomous BMS operations — scheduled setpoint adjustments, load-shedding logic, sensor-triggered responses — are subjects in the authorization model that must be addressed explicitly, particularly during connectivity gaps when they may operate without authorization evaluation.

**Credential and certificate lifecycle management.** Certificate rotation in OT environments is substantially less tractable than in enterprise IT. Field gateways and enforcement points at edge sites may not be reachable through standard management channels. Certificate expiration that is not caught before the expiration date leaves enforcement points unable to authenticate to the policy distribution channel — a degraded state that may be difficult to detect remotely and operationally problematic to resolve.

**Fleet-scale resource and identity management.** Resource descriptor management — maintaining the identifiers that map physical controller points to the authorization model's resource vocabulary — requires a process that keeps descriptors current as device configurations change. A mismatch between the resource model and the actual device configuration produces well-formed policy decisions about the wrong resources, with no indication at the policy level that the evaluation is semantically incorrect. Manual identity lifecycle management at fleet scale produces the operational shortcuts — shared credentials, deferred deprovisioning — that identity-aware authorization is designed to replace.

**Vendor interoperability and protocol diversity.** OT device manufacturers commonly implement proprietary extensions and vendor-specific behavioral variations that deviate from protocol standard base behavior. Firmware updates introduce adapter compatibility concerns with no parallel in IT software systems, and no natural owner typically exists for the cross-domain dependency between device firmware state and adapter normalization correctness in most organizational structures.

**Audit durability and delivery semantics.** Edge enforcement points must buffer audit events locally when central store delivery fails. Buffer exhaustion must produce an observable alert rather than silent event loss. Reordered delivery from buffered events requires that analysis tools are robust to non-temporal ordering. Cryptographic chaining of audit records must be maintained consistently through normal operation, local buffering, and bulk delivery after connectivity gaps.

**Administrative ownership and governance complexity.** The authorization layer spans organizational and administrative boundaries that no existing OT infrastructure component does — identity administration, policy governance, enforcement point operations, and adapter configuration each belonging to different teams under different organizational structures. These cross-team dependencies are not unique to OT authorization, but their failure consequences are more significant in environments with high operational sensitivity and limited tolerance for disruption. Ownership fragmentation is acutely visible during failure scenarios: degraded mode behavior, policy rollback under an active incident, and synchronization failure response each require coordinated input from teams with no established coordination path.

The section closes by naming the fundamental tradeoffs that each deployment must negotiate — security vs. continuity, policy currency vs. operational independence, audit completeness vs. delivery reliability, policy expressiveness vs. evaluation determinism — and identifying these as properties of distributed systems operating under the constraints of OT network connectivity, not engineering defects that architectural refinement resolves. The architecture provides a framework within which these tradeoffs can be made deliberately; it does not eliminate them.

---

### 08 — Threat Modeling and Security Considerations

*[sections/08-threat-modeling-and-security-considerations.md](sections/08-threat-modeling-and-security-considerations.md)*

Identifies and analyzes threats that are specifically relevant to, or materially affected by, the authorization architecture described in this paper. Threat categories include unauthorized command issuance, rogue gateways, telemetry spoofing, credential reuse, audit log tampering, policy misconfiguration, offline authorization failure modes, fail-open conditions, and operator impersonation. Each threat is described in terms of mechanism, architectural mitigation, and residual risk. The analysis is practical and systems-oriented rather than theoretical.

---

### 09 — Future Direction *(stub)*

*[sections/09-future-directions.md](sections/09-future-directions.md)*

*Planned content:* Open research and engineering questions in identity-aware OT authorization. Topics may include: lightweight authentication protocols for resource-constrained field devices; hardware attestation at the device level; standardization of identity context in OT protocols; anomaly detection based on authorization decision patterns; and the long-term trajectory of OT IAM as protocol ecosystems evolve.

---

### 10 — Conclusion *(stub)*

*Planned content:* Synthesis of the paper's findings. Will summarize what the authorization architecture provides, what it does not provide, and what the BASIS PoC demonstrated and left open. Will close with the paper's core claim: that identity-aware authorization is a tractable engineering problem in OT environments, that it requires architectural accommodation of OT-specific constraints, and that the gap between current practice and this model is narrowing incrementally through deployable components rather than wholesale infrastructure replacement.

---

## Diagram Index

Diagrams associated with this paper are maintained in the [`diagrams/`](diagrams/) subdirectory.

| File | Description | Status |
|---|---|---|
| [`identity-aware-authorization-flow.mmd`](diagrams/identity-aware-authorization-flow.mmd) | Mermaid source: full architecture diagram showing trust zones, subjects, enforcement points, protocol adapters, policy engine, audit pipeline, and control/data plane flows | Complete |
| [`identity-aware-authorization-flow-spec.md`](diagrams/identity-aware-authorization-flow-spec.md) | Full diagram specification including component responsibilities, flow semantics, Draw.io guidance, and pre-commit checklist | Complete |
| [`ot-trust-boundary-overview.mmd`](diagrams/ot-trust-boundary-overview.mmd) | Mermaid source: trust boundary overview — zones, enforcement point positions, and cross-boundary flow directions | Complete |
| [`ot-trust-boundary-overview-spec.md`](diagrams/ot-trust-boundary-overview-spec.md) | Diagram specification for the trust boundary overview | Complete |
| [`basis-poc-architecture-mapping.mmd`](diagrams/basis-poc-architecture-mapping.mmd) | Mermaid source: conceptual architecture mapping diagram — maps BASIS PoC components to the authorization architecture roles described in Sections 04 and 05 | Complete |
| [`basis-poc-architecture-mapping-spec.md`](diagrams/basis-poc-architecture-mapping-spec.md) | Diagram specification including component reference table, flow semantics, intentional omissions, and guidance for use in Section 06 | Complete |

---

## Editorial Notes

Sections 01–08 are drafted. Sections 01–05 have been reviewed for tone and content consistency. Sections 06–08 are mature drafts — substantively complete but not yet through a final consistency review pass. Sections 09 (Future Direction) and 10 (Conclusion) are not yet started.

The paper has reached a stage where conceptual cohesion, architectural consistency, and stable terminology are more important than expansion. New sections and revisions should be evaluated against whether they strengthen the paper's central argument — that identity-aware authorization in OT environments requires engineering the operational, synchronization, governance, and deployment tradeoffs carefully rather than abstracting them away — rather than whether they add coverage of adjacent topics.

Sections 01 (Non-Goals) and 08 (Threat Modeling) should be reviewed whenever a new section is drafted to ensure that scope boundaries remain accurate and that the threat analysis remains consistent with the architecture as described.

The Introduction and Motivation, Section Summaries, and Diagram Index in this master document should be kept current as section content is revised. The section summaries are the primary way a reader scans the paper's coverage before reading individual sections; they should accurately reflect conceptual emphasis, not just list topics.

Contributions to this paper should follow the writing guidelines in [`docs/standards/writing-guidelines.md`](../../docs/standards/writing-guidelines.md). The diagram standards apply to all diagrams added to the paper.
