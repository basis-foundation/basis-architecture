# Reference Architecture vs. Implementation

A recurring source of confusion in architecture documentation is the conflation of three distinct kinds of knowledge: what the architecture requires conceptually, what a reference realization looks like, and what specific implementations have actually built. These are different things, and treating them as interchangeable produces misleading documentation and incorrect implementation expectations.

This document distinguishes the three levels, explains what each one expresses, and identifies the conflations that are most likely to occur in the BASIS ecosystem given its current stage of development.

---

## The Three Levels

**Conceptual architecture** describes the structural requirements of the system: what components must exist, what responsibilities they must own, how they must relate to each other, and what constraints govern their interactions. Conceptual architecture does not specify technology, programming language, deployment topology, or implementation approach. It establishes the *what* and *why* — the constraints and semantics that any implementation must satisfy.

The white papers in this repository operate at the conceptual architecture level. The authorization flow, the trust zone model, the policy evaluation model, the audit pipeline, and the enforcement point contracts are conceptual descriptions. They express what must be true, not how any specific system achieves it.

**Reference architecture** describes one specific realization of the conceptual architecture: a coherent set of component choices, deployment patterns, and integration approaches that satisfies the conceptual requirements. A reference architecture is not the only valid realization — it is a concrete example that implementation teams can use as a starting point. It is more specific than conceptual architecture and less authoritative: a conceptual requirement is binding; a reference architecture pattern is a suggestion.

The BASIS Core Services Distribution (basis-core, basis-gateway, basis-console, basis-adapters, basis-deploy, basis-schemas) represents the reference architecture for the BASIS ecosystem — a specific, deployable realization of the conceptual model described in the white papers. It is not the only possible realization, but it is the one maintained under Foundation governance.

**Implementation** describes what has actually been built in a specific repository, deployment, or production context. An implementation is specific, observed, and operational. It may satisfy the conceptual requirements in ways that differ from the reference architecture patterns without violating the architecture, as long as the contracts and constraints are preserved.

The basis-poc (the BASIS proof-of-concept) is an implementation — a specific, monolithic, containerized research implementation that validated selected aspects of the conceptual architecture. It is neither a reference architecture pattern nor a production-ready system.

---

## What Architecture Documents Express

Architecture documents in this repository express conceptual requirements and design constraints. They establish what must be true — not as engineering aspirations, but as architectural requirements that any compliant implementation must satisfy.

When a white paper section states that "the policy engine receives a structured request and produces a deterministic authorization decision," that is a conceptual requirement. It constrains every implementation that calls itself an authorization kernel: the evaluation must be deterministic, the input must be structured, and the output must be an authorization decision. It does not specify the data format, the evaluation algorithm, the programming language, or the deployment topology.

When `docs/architecture/kernel-boundary-rules.md` states that "basis-core must not include database clients, ORMs, or runtime persistence backends," that is a design constraint. It is enforceable: a proposed implementation of basis-core that imports SQLAlchemy violates this constraint, regardless of how the import is framed.

When `docs/architecture-principles.md` states that "the authorization infrastructure must be designed to support continued safe operation during partial failures," that is an architecture principle. It constrains the design choices of any component that participates in the authorization infrastructure.

Architecture documents do not express implementation decisions. How the kernel stores policy during evaluation, which Python version the gateway targets, or how the deployment tooling handles container orchestration are implementation decisions that belong in component repositories, not in architecture documentation.

---

## What Implementation Repositories Own

Implementation repositories (basis-core, basis-gateway, basis-adapters, and others) own the decisions about how to realize the architectural requirements in a specific technology context. This includes:

- Programming language and runtime environment choices
- Library and framework selections (subject to the dependency constraints in the architecture)
- Internal data structure and algorithm choices
- Deployment configuration and packaging format
- Test strategy and tooling

These decisions are constrained by the architecture but are not prescribed by it. An architecture that prescribes implementation technology becomes tightly coupled to the technology choices of one period and loses the adaptability that OT environments require over long operational lifetimes.

When an implementation decision reveals that an architectural constraint is impractical — that satisfying the constraint in the available technology context is genuinely impossible or produces unacceptable operational tradeoffs — the correct response is to surface that conflict as an architectural discussion in basis-architecture, not to silently resolve it in the implementation repository. The architecture may need to be refined. That refinement should be visible and deliberate.

---

## Diagrams: Conceptual Unless Labeled Otherwise

Diagrams in this repository are conceptual unless explicitly labeled as implementation-specific. A conceptual diagram shows architectural structure, trust zones, component responsibilities, and data flows at the level of abstraction appropriate to the conceptual architecture. It does not show specific technology choices, specific deployment topology, or specific implementation decisions.

The diagrams in the white paper (`identity-aware-authorization-flow.mmd`, `ot-trust-boundary-overview.mmd`, and others) are conceptual diagrams. They show an "identity provider," not "Keycloak." They show a "policy engine," not "basis-core at version 1.0." They show "protocol adapters," not "the specific BACnet adapter implementation in basis-adapters."

Reading a conceptual diagram as if it prescribes a specific implementation produces incorrect conclusions:

- The diagram shows a single policy engine in the Central Services Zone. This does not mean the architecture requires exactly one policy engine instance. It means there is a logical policy evaluation authority. The implementation may run multiple instances for availability.
- The diagram shows protocol adapters in the Edge/Gateway Zone. This does not mean adapters must be co-located with the gateway on the same host. It means adapters occupy the architectural position of the edge zone. The deployment topology may differ.
- The diagram shows the audit log store in the Central Services Zone. This does not mean the audit store must be a specific database. It means audit records must be collected in a component that is administratively independent of the components being audited.

When a diagram is implementation-specific — when it shows specific technology choices, version numbers, or deployment configurations — it must be clearly labeled as such, and it should appear in an implementation repository or in an ADR that documents the specific decision, not in the conceptual white paper.

---

## Deployment Realities May Vary While Preserving Contracts

Deployments that implement the BASIS architecture may differ from each other — and from the reference architecture — in ways that are consistent with the conceptual requirements. The architecture does not prescribe a single deployment topology. It prescribes a set of contracts and constraints that every compliant deployment must satisfy.

A deployment that uses a different identity provider than the one used in the reference implementation is compliant, as long as the identity provider produces verified identity context in the form the policy engine can evaluate.

A deployment that places enforcement points at different trust boundary locations than the reference topology is compliant, as long as enforcement covers the boundaries identified by the trust zone analysis and applies the authorization contracts correctly.

A deployment that uses a different audit store technology than the reference implementation is compliant, as long as the audit store satisfies the immutability, schema consistency, and administrative independence requirements.

The architecture is a constraint set, not a blueprint. Deployments that satisfy the constraint set are correct regardless of how they differ from the reference implementation.

The corollary is important: deployments that satisfy the reference implementation pattern but violate a conceptual constraint are not compliant. Using the basis-gateway component in the deployment does not guarantee compliance if the gateway is configured in a way that bypasses enforcement contracts or breaks identity propagation.

---

## basis-poc: Validation, Not Canonical Deployment Model

The basis-poc is a monolithic, containerized research implementation. It was developed to validate that the core architectural mechanisms — identity propagation, policy evaluation, protocol-agnostic enforcement, audit pipeline, protocol adapters — are coherent and implementable. It succeeded at that goal.

The basis-poc is not:

- A production-ready system
- A canonical deployment model
- A reference for how basis-core should be implemented
- A reference for how the distribution components should be structured
- An indication of the technology choices that the distribution endorses

References to the basis-poc in architecture documentation are illustrative. They demonstrate that specific patterns are implementable, not that the PoC implementation is the correct or preferred implementation approach.

Section 06 of the white paper — "The BASIS Proof-of-Concept" — explicitly addresses what the PoC validated, what it did not validate, and the distinction between the PoC and the BASIS Core Services Distribution direction that emerged from its lessons. Architecture documents and implementation repositories should be consistent with that characterization.

---

## basis-core: Stable Semantics, Not Operational Topology

basis-core defines the stable evaluation semantics that all other components depend on. It is the authorization kernel — the component that implements policy evaluation logic, enforcement contracts, failure mode contracts, and audit event schema.

basis-core is not a deployed network service. It does not host an API. It does not occupy a position in the deployment topology in the way that basis-gateway does. basis-core is a library — a kernel that gateway and adapter components call into. basis-gateway is the component that exposes basis-core evaluation as a runtime service accessible to enforcement points and external callers.

This distinction has concrete consequences for how architecture documents should describe the system:

- "The policy engine evaluates the request" describes the conceptual architecture. It is correct.
- "basis-gateway invokes basis-core to evaluate the request" describes the implementation. It is correct for the reference implementation.
- "basis-core sits at the trust boundary and enforces the decision" is incorrect. basis-core does not sit at trust boundaries; enforcement points implemented by basis-gateway and basis-adapters sit at trust boundaries and call into basis-core for evaluation.

Architecture documents should maintain the distinction between the logical role (policy engine, authorization kernel) and the specific component that fulfills that role (basis-core), and between the component that hosts the evaluation (basis-core) and the component that hosts the runtime (basis-gateway).

---

## Common Conflations to Avoid

The following conflations appear frequently in documentation and should be corrected when encountered:

**Conflating basis-poc with the BASIS Core Services Distribution.** The PoC demonstrated architectural feasibility; the distribution is the separated, maintained ecosystem of components. They are related but distinct. Do not describe PoC implementation choices as if they are distribution design decisions.

**Conflating basis-core with basis-gateway.** basis-core is the evaluation kernel; basis-gateway is the runtime wrapper that hosts the API and calls into the kernel. "basis-core receives authorization requests over HTTP" is incorrect — basis-gateway receives them; basis-core evaluates them.

**Treating conceptual diagrams as deployment prescriptions.** A conceptual diagram that shows one policy engine does not require exactly one policy engine instance in every deployment. Read diagrams at their intended level of abstraction.

**Conflating the reference architecture with the only valid implementation.** The BASIS Core Services Distribution is a reference realization of the conceptual architecture. Deployments that realize the same conceptual architecture through different component choices are not non-compliant by virtue of differing from the reference.

**Describing the PoC as a production deployment or as representative of production behavior.** The PoC was operated at research scope, with simulated resources, in a controlled environment. Its latency, throughput, availability, and operational characteristics are not representative of production OT deployments.

**Describing architectural constraints as implementation prescriptions.** "basis-core must not include database clients" is an architectural constraint that applies to the basis-core component. It does not prescribe how the full system handles persistence — basis-gateway and basis-deploy may use databases as appropriate for their functions.
