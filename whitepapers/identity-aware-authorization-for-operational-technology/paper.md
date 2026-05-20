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

| # | Section | Status |
|---|---|---|
| 01 | [Introduction and Motivation](#introduction-and-motivation) | Draft |
| 01b | [Non-Goals](sections/01b-non-goals.md) | Complete |
| 02 | [Current State of OT Authorization](sections/02-current-state-of-ot-authorization.md) | Complete |
| 03 | [Why Existing Approaches Become Insufficient](sections/03-why-existing-approaches-are-incomplete.md) | Complete |
| 04 | [Identity-Aware OT Architecture](sections/04-identity-aware-ot-architecture.md) | Complete |
| 05 | [Identity-Aware OT Architecture](sections/05-identity-aware-ot-architecture.md) | Complete |
| 06 | The BASIS Proof-of-Concept *(stub)* | Not started |
| 07 | Production Considerations and Operational Constraints *(stub)* | Not started |
| 08 | Future Directions *(stub)* | Not started |
| 09 | [Threat Modeling and Security Considerations](sections/09-threat-modeling-and-security-considerations.md) | Complete |
| — | [References](references/sources.md) | In progress |

---

## Introduction and Motivation

Authorization in operational technology environments is not a solved problem. It is not even, in most deployments, a well-posed one. The access control mechanisms in widespread use — VLAN membership, network segmentation, controller-local passwords, shared operator credentials — were not designed as authorization systems. They evolved as practical constraints on communication within bounded physical networks, and they function reasonably well within those bounds.

The problem is that the bounds themselves have changed. A building automation system that was once operated by a single team of technicians with physical access to a dedicated network is now expected to integrate with enterprise energy management platforms, expose operational data to remote analytics services, support vendor remote access for maintenance, and provide audit records sufficient for compliance reporting. None of these requirements were contemplated in the design of BACnet or Modbus. None of them are well-served by VLAN-based trust models.

Identity-aware authorization offers a different approach: rather than treating network location as a proxy for trust, verify the identity of every principal — operator, device, service — and evaluate every access request against explicitly defined policy. This approach is well-established in enterprise IT and cloud environments. Its application to OT is less mature, and for good reason: OT environments carry constraints that do not exist in IT contexts.

Field devices may have no computational capacity for authentication protocol participation. Field protocols may have no message structure that supports attaching identity context. Maintenance windows are narrow and infrequent. Controllers that have operated without interruption for years cannot be taken offline for software upgrades. Operational continuity is a safety requirement, not merely an operational preference. Any authorization architecture for OT environments must be designed with these realities as first-order constraints, not as edge cases to be addressed later.

This paper describes what identity-aware authorization looks like when designed with those constraints in mind. It does not claim to have resolved all of them. It documents where they bite, where they require architectural accommodations, and where they remain open engineering problems.

---

## Section Summaries

### 01b — Non-Goals

*[sections/01b-non-goals.md](sections/01b-non-goals.md)*

Establishes explicit scope boundaries for the paper. Clarifies that the work does not propose replacing OT protocols, does not argue for eliminating network segmentation, does not claim BASIS is production-ready, does not assume cloud-only deployment, and does not claim identity-aware authorization alone resolves OT security. These are not aspirational limitations — they are deliberate scope decisions.

---

### 02 — Current State of OT Authorization

*[sections/02-current-state-of-ot-authorization.md](sections/02-current-state-of-ot-authorization.md)*

Describes the authorization landscape in OT environments as it exists today: network segmentation, VLAN-based trust, VPN and jump host access patterns, controller-local credentials, shared operator accounts, vendor-specific trust models, and protocol-centric security assumptions. The section is analytical rather than prescriptive — it explains how current approaches evolved historically and identifies the operational and architectural conditions under which they become insufficient, without characterizing the current state as negligent or broken.

---

### 03 — Why Existing Approaches Become Insufficient

*[sections/03-why-existing-approaches-are-incomplete.md](sections/03-why-existing-approaches-are-incomplete.md)*

Examines the structural limitations of network-centric trust, controller-local authorization, and distributed authorization logic as OT environments become more interconnected, remotely managed, and subject to audit and accountability requirements. Topics include the coarse granularity of network-based access control, the distinction between authentication and authorization, the operational scaling challenge of maintaining fragmented policy, inconsistent and incomplete auditability, and the inability to centrally reason about access decisions across heterogeneous systems. Closes by introducing the framing that authorization is increasingly a system-level concern rather than a protocol-level one — the architectural premise that the following section builds on.

---

### 04 — Identity-Aware OT Architecture

*[sections/04-identity-aware-ot-architecture.md](sections/04-identity-aware-ot-architecture.md)*

Defines the generalized conceptual architecture for identity-aware authorization in OT environments. Introduces and explains the core model — subjects, resources, and actions as the authorization primitives; policy evaluation as a shared layer distinct from enforcement; enforcement points at trust boundary crossings; protocol adapters that normalize field-protocol messages into the shared representation; identity propagation across multi-hop request paths; centralized auditability through a shared audit pipeline; control plane and data plane separation; local policy caching for offline resilience; and context-aware authorization decisions. Closes with an honest accounting of architectural tradeoffs — latency, operational complexity, legacy device coverage gaps, partial deployment realities, and new failure modes introduced by the authorization infrastructure itself. Sets up the following sections on trust boundaries and the BASIS proof-of-concept.

---

### 05 — Identity-Aware OT Architecture

*[sections/05-identity-aware-ot-architecture.md](sections/05-identity-aware-ot-architecture.md)*

The conceptual center of the paper. Establishes authorization as a system-level concern, builds the full architecture from first principles, and integrates closely with the authorization flow diagram. Covers: the three-function separation (identity verification, policy evaluation, enforcement); subjects, resources, and actions as authorization primitives; identity propagation across trust boundaries and what breaks it; how protocol adapters normalize field-protocol semantics into the shared authorization model; why distributed enforcement can coexist with centralized policy through local policy caching; control plane and data plane as distinct architectural concerns with distinct traffic paths; centralized auditability as an architectural first-class property; offline resilience design and the policy cache staleness tradeoff; context-aware authorization and its complexity costs. Closes with an honest accounting of constraints — latency, legacy coverage, operational complexity, partial deployment, and high availability requirements for the authorization infrastructure itself.

---

### 06 — The BASIS Proof-of-Concept *(stub)*

*[sections/06-basis-poc.md](sections/06-basis-poc.md)*

*Planned content:* Description of the BASIS PoC implementation — its scope, components, design decisions, and what it demonstrates. Will cover Keycloak integration for identity, OPA-based policy evaluation, MQTT telemetry pipeline, audit logging design, and protocol adapter implementation. Will be explicit about what was validated and what remains to be validated.

---

### 07 — Production Considerations and Operational Constraints *(stub)*

*[sections/07-production-considerations.md](sections/07-production-considerations.md)*

*Planned content:* Analysis of the gap between PoC implementation and production deployment. Topics include: high availability requirements for the policy engine and identity provider; latency constraints in time-sensitive control loops; air-gapped and intermittently connected deployments; fail-safe behavior under policy engine unavailability; credential lifecycle management in OT maintenance workflows; and vendor interoperability challenges across heterogeneous BAS deployments.

---

### 08 — Future Directions *(stub)*

*[sections/08-future-directions.md](sections/08-future-directions.md)*

*Planned content:* Open research and engineering questions in identity-aware OT authorization. Topics may include: lightweight authentication protocols for resource-constrained field devices; hardware attestation at the device level; standardization of identity context in OT protocols; anomaly detection based on authorization decision patterns; and the long-term trajectory of OT IAM as protocol ecosystems evolve.

---

### 09 — Threat Modeling and Security Considerations

*[sections/09-threat-modeling-and-security-considerations.md](sections/09-threat-modeling-and-security-considerations.md)*

Identifies and analyzes threats that are specifically relevant to, or materially affected by, the authorization architecture described in this paper. Threat categories include unauthorized command issuance, rogue gateways, telemetry spoofing, credential reuse, audit log tampering, policy misconfiguration, offline authorization failure modes, fail-open conditions, and operator impersonation. Each threat is described in terms of mechanism, architectural mitigation, and residual risk. The analysis is practical and systems-oriented rather than theoretical.

---

## Diagram Index

Diagrams associated with this paper are maintained in the [`diagrams/`](diagrams/) subdirectory.

| File | Description | Status |
|---|---|---|
| [`ot-trust-boundary-overview.mmd`](diagrams/ot-trust-boundary-overview.mmd) | Mermaid source: trust zones, enforcement points, data and control plane flows | Complete |
| [`ot-trust-boundary-overview-spec.md`](diagrams/ot-trust-boundary-overview-spec.md) | Full diagram specification including color palette, Draw.io guidance, and checklist | Complete |

---

## Editorial Notes

This paper is in active development. Sections marked as stubs represent planned content that has not yet been drafted. Sections marked as complete have been reviewed for tone and content consistency against the writing guidelines in [`docs/standards/writing-guidelines.md`](../../docs/standards/writing-guidelines.md).

Contributions to this paper should follow the writing guidelines document. The diagram standards in [`docs/diagrams/README.md`](../../docs/diagrams/README.md) apply to all diagrams added to this paper.

The non-goals section (01b) and threat modeling section (09) should be reviewed whenever a new section is drafted to ensure the scope boundaries remain accurate and the threat analysis remains consistent with the architecture described.
