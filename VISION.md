# Vision

This document describes the architectural direction of the BASIS ecosystem — why it exists, what problems it is built to address, and what principles shape its design. It is an engineering document, not a product statement. It makes no predictions about adoption, market position, or future capabilities beyond what the current architecture supports and the ongoing research justifies.

For the complete architectural analysis that underlies this vision, see the white paper at [`whitepapers/identity-aware-authorization-for-operational-technology/`](whitepapers/identity-aware-authorization-for-operational-technology/).

---

## The Authorization Problem in OT Environments

Operational technology environments — building automation systems, industrial control networks, campus infrastructure, data center facilities — have historically managed access through network-centric mechanisms: VLAN separation, firewall rules, and the assumption that devices on a trusted network segment are legitimate participants. These mechanisms were appropriate for the environments in which they were designed: physically bounded, protocol-homogeneous networks with limited external connectivity.

Those conditions have changed. OT networks are increasingly connected to enterprise IT infrastructure, cloud-hosted management platforms, and remote access services. Vendor technicians connect remotely. Building management systems exchange data with energy management and facility systems. IT operations tooling reaches into operational infrastructure. The implicit trust assumptions built into field protocols and segment-based access control are no longer adequate.

The structural problem is not that network segmentation is a poor security control. It is that network segmentation was never designed to answer the authorization question — "is this specific subject permitted to perform this specific action on this specific resource?" — in a way that can be audited, governed, or updated without reconfiguring network topology. When the relevant question is about identity and permission rather than about network membership, the answer requires different infrastructure.

This is the problem that identity-aware authorization addresses. The architectural analysis in Section 02 of the white paper documents the current state of OT authorization in detail; Section 03 explains why existing approaches become structurally insufficient as connectivity requirements grow. The vision for BASIS is grounded in that analysis — it is a response to a specific, documented structural gap, not to a general aspiration for better security.

---

## Why an Ecosystem, Not a Single Tool

The earliest version of this work — the BASIS proof-of-concept — demonstrated that the core mechanisms of identity-aware authorization are coherent: identity propagation, explicit policy evaluation, enforcement at a defined boundary, protocol normalization, and audit recording can be assembled into a functioning system. The PoC was a monolithic, research-scope implementation in a single containerized environment. Its purpose was validation, not deployment.

The step from a validated research implementation to a deployable, operable authorization system requires separating concerns that the PoC deliberately collapsed for tractability. The policy evaluation semantics need to be stable and independently versionable. The protocol adapters need to be separately maintainable as device populations change. The deployment tooling needs to address the specific connectivity and maintenance-window constraints of OT environments. The API surface needs to be operable independently of the kernel.

These are not implementation details — they are architectural boundaries. An authorization system that entangles kernel semantics with deployment configuration, or that couples the protocol adapter layer to the policy evaluation logic, cannot be maintained reliably over the multi-decade operational life of the OT systems it governs. The ecosystem structure — basis-core, basis-gateway, basis-adapters, basis-console, basis-schemas, basis-deploy — reflects these architectural boundaries as component boundaries.

The ecosystem model also reflects a distinction between what can be provided as open-source infrastructure and what requires commercial operational services. The BASIS Core Services Distribution, governed by the Basis Foundation, is designed to provide a complete functional authorization system for organizations that can operate it themselves. Commercial services from BASAuth address the operational layer above that: managed hosting, fleet coordination, enterprise integrations, and support at scale. The ecosystem boundary between open-source and commercial is architectural, not artificial — it follows the natural line between authorization semantics (which belong to shared infrastructure) and operational services (which require ongoing human engagement and commercial accountability).

The complete ecosystem structure is described in [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md).

---

## Why the Kernel Must Be Isolated

The authorization kernel — basis-core — owns the stable evaluation semantics that all other components depend on. Its isolation is not a design preference; it is a consequence of the operational requirements of the environments the ecosystem serves.

OT environments impose constraints that make a large, entangled authorization kernel untenable:

**Long asset lifecycles.** A building automation controller installed today may remain in service for fifteen to twenty years. The software components that govern access to it must be maintainable across that timeframe — which requires that the components with stable semantics (the kernel) be separable from the components that change more frequently (adapters, gateways, deployment tooling).

**Constrained maintenance windows.** Authorization infrastructure updates in OT environments must fit within maintenance windows measured in hours per month. A component that can be updated, tested, and validated in isolation is far more tractable to maintain than one that requires coordinated updates across a monolithic system.

**Deployment heterogeneity.** OT deployments vary enormously in their connectivity, infrastructure, and operational constraints. An air-gapped industrial facility and a cloud-adjacent commercial building management system have fundamentally different deployment characteristics. A kernel without cloud SDK dependencies, database runtime dependencies, or identity provider dependencies can be deployed across this range. A kernel with such dependencies cannot.

**Testability and auditability.** Authorization behavior in OT environments has safety implications. A kernel that can be tested independently — with its evaluation semantics verified in isolation from network conditions, adapter behavior, and deployment configuration — provides a stronger basis for trust than one whose behavior is entangled with its operational context.

The principle that basis-core must not depend on higher-level services is stated as a governance constraint in [`GOVERNANCE.md`](GOVERNANCE.md) and as a component boundary rule in [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md). It is enforced as a governance concern because its value degrades invisibly when dependencies accumulate incrementally. Architecture Principle 10 in [`docs/architecture-principles.md`](docs/architecture-principles.md) — "Protocol Abstraction Where Feasible" — reflects the same reasoning applied to the protocol layer.

---

## Why Protocol Normalization Matters

OT environments are protocol-heterogeneous. A single building may carry BACnet, Modbus, MQTT, and proprietary vendor protocols across its various subsystems. Authorization logic that is expressed in terms of protocol-specific details — BACnet object identifiers, Modbus register addresses — cannot be shared, cannot be audited consistently, and will diverge across protocols over time.

The protocol adapter layer exists to solve this problem by translating field-protocol messages into the shared subject-resource-action representation that the authorization kernel evaluates. Once a message has been normalized, the kernel evaluates it without any knowledge of which protocol the underlying device speaks. A policy that governs access to a class of HVAC controller points applies consistently regardless of whether those controllers speak BACnet or a proprietary protocol, as long as the adapters for both produce equivalent normalized representations.

The architectural consequence is that the policy model is stable across protocol changes. Adding support for a new protocol requires a new adapter, not a change to the kernel or to the policies that govern access to the resources the new adapter serves.

This is not a novel architectural insight — Section 04 of the white paper develops it in detail, and Architecture Principle 5 ("Policy Evaluation Independent of Protocol") and Principle 10 ("Protocol Abstraction Where Feasible") articulate it as a design commitment. What the ecosystem adds to the architectural insight is an organizational expression of it: basis-adapters is a separately maintained component precisely because protocol support changes at a different rate than authorization semantics.

The adapter layer also carries a trust implication: the kernel trusts adapter normalizations unconditionally. This makes adapter correctness a security requirement, not merely a functional one. Section 05 of the white paper examines this trust boundary in detail. The governance consequences are that adapter implementations must be tested against the full object model of the devices they serve, and that device configuration changes must trigger adapter review as part of the change management process.

---

## Why Incremental Adoption Is First-Order

The architecture is designed to be adopted incrementally, without requiring full replacement of existing OT infrastructure. This is not a concession to practical constraints — it is a design requirement that reflects how OT environments actually work.

BAS deployments are not greenfield projects. Real environments contain controllers, field devices, and protocol infrastructure with operational lifespans measured in decades. A security architecture that requires full replacement of existing systems as a prerequisite for adoption will not be adopted. The relevant design question is not "what does a fully ideal system look like?" but "how does a system improve security in an environment with significant existing constraints?"

Architecture Principle 13 ("Incremental Adoption Over Full Replacement") captures this directly. Incremental adoption requires that new components interoperate with legacy systems, that the architecture supports partial deployment where some systems are identity-aware and others are not, and that the security posture of a partially deployed system is meaningfully better than the baseline it replaces.

The enforcement placement model — described in detail in Section 05 of the white paper — reflects this requirement. Enforcement at the operations-zone boundary (the jump host) provides session-level accountability for remote access without requiring any changes to field devices or protocol adapters. Each additional layer of enforcement added improves coverage without requiring full deployment as a prerequisite. The architecture delivers value incrementally, not only after complete deployment.

This design choice has a consequence that must be stated plainly: a partially deployed system provides partial coverage. An enforcement point at the jump host cannot govern field-device-level command semantics. An adapter that covers BACnet devices does not cover Modbus devices until the Modbus adapter is also deployed. Incremental adoption is not a workaround for architectural incompleteness — it is the realistic adoption path for technology in operational environments where disruption is costly and change management is constrained.

---

## Why Operational Constraints Shape the Architecture

The architecture described in the white papers and implemented in the ecosystem components is designed around operational constraints, not around idealized security properties. This is intentional, and it is what distinguishes this architectural approach from security frameworks that treat OT environments as edge cases.

The constraints are real and consequential:

- Field devices often cannot participate in authentication protocols. The authorization model must work around this constraint, not assume it away.
- Connectivity between edge sites and central services is frequently intermittent. The local policy cache exists because the alternative — a hard dependency on real-time central policy evaluation — would make the authorization infrastructure an operational liability.
- Maintenance windows are narrow. Authorization infrastructure updates must compete with field device maintenance, firmware updates, and physical infrastructure work for limited operational downtime.
- Latency tolerances at the field device level preclude real-time policy queries per control loop. Enforcement is positioned at trust boundaries where latency can be tolerated, not inside time-critical control loops.
- The teams that operate OT systems and the teams that govern security policy are often organizationally separate. The architecture must accommodate this separation, not require organizational restructuring as a prerequisite for deployment.

Architecture Principle 12 ("Security Must Respect Operational Constraints") and Principle 6 ("Operational Resilience First") articulate these design commitments. The production analysis in Section 07 of the white paper examines the operational engineering consequences in detail. The vision for BASIS is not a system that overcomes OT constraints — it is a system that works within them, that acknowledges them as first-order design inputs, and that makes the resulting tradeoffs explicit rather than hiding them behind architectural abstractions.

---

## What This Vision Is Not

This vision document does not describe a finished system. The BASIS ecosystem is in an early architectural maturity stage. The white paper is the most developed artifact. The proof-of-concept validated core mechanisms at limited scope. The component architecture described in [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md) is specified but not yet fully implemented. The governance model described in [`GOVERNANCE.md`](GOVERNANCE.md) is lightweight and will evolve.

This document does not make predictions about adoption, deployment scale, or industry impact. The architectural analysis is honest about where the approach is strong and where significant engineering work remains. The production realities examined in Section 07 of the white paper are not problems to be dismissed — they are the dominant engineering challenges that any deployment of this architecture must address.

The goal is an authorization architecture that engineers can evaluate honestly, deploy incrementally, operate within the constraints of OT environments, and maintain over long timeframes without accumulating hidden dependencies or governance debt. Whether that goal is achievable at any given deployment scale and organizational context is a judgment that belongs to the teams doing the deployment, not to the architecture.
