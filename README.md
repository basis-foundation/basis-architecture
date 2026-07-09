# BASIS Architecture

This repository contains architecture documentation, white papers, threat models, architecture decision records (ADRs), and diagramming standards for the BASIS ecosystem.

BASIS is an open-source core services distribution for identity-aware authorization in operational technology (OT) environments, governed by the Basis Foundation. The work addresses identity-aware authorization, centralized policy enforcement, and auditability for OT environments, with building automation systems (BAS) as the primary domain. Architectural patterns are intended to be broadly applicable across OT contexts including data centers, hospitals, campuses, commercial buildings, and industrial facilities.

The BASIS ecosystem consists of three distinct layers:

- **Basis Foundation** — the nonprofit body that governs the open-source work, maintains architectural standards, and stewards the core services distribution
- **BASIS Core Services Distribution** — the open-source, deployable set of components (basis-core, basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, basis-schemas) that together provide a functional identity-aware authorization system
- **BASAuth** — the future for-profit commercial company that builds enterprise products and managed services on top of the open-source distribution

The architectural center of the distribution is **basis-core**, the isolated authorization kernel. basis-core owns the stable policy evaluation semantics, enforcement contracts, and audit event definitions that all other components in the distribution depend on. It does not depend on higher-level services, cloud infrastructure, identity providers, protocol stacks, or UI components.

For a complete description of the ecosystem structure, component responsibilities, and dependency rules, see [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md).

---

## Repository Purpose

This repository is the canonical source of architectural documentation for the BASIS initiative. It is organized to support long-term research continuity: each document type has a defined location, each white paper is self-contained within its own directory, and shared standards apply across all content.

The repository does not contain application code, deployment configuration, or operational tooling. Those artifacts are maintained in the BASIS implementation repositories. What is here is architecture: the reasoning behind design decisions, the threat analysis that informs them, the standards that keep documentation consistent, and the diagrams that make structural relationships visible.

---

## How to Use This Repository

### Start Here

If you are new to this repository, start with the white paper abstract and the ecosystem structure document:

- [`whitepapers/identity-aware-authorization-for-operational-technology/paper.md`](whitepapers/identity-aware-authorization-for-operational-technology/paper.md) — the master document for the identity-aware authorization white paper: abstract, table of contents, section summaries, and diagram index
- [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md) — the three layers of the BASIS ecosystem, component responsibilities, and dependency rules
- [`docs/glossary.md`](docs/glossary.md) — canonical definitions for all terms used across the documentation

### For Architecture Reviewers

- [`docs/architecture-principles.md`](docs/architecture-principles.md) — the fifteen guiding principles
- [`docs/kernel-boundary-rules.md`](docs/kernel-boundary-rules.md) — the enforceable rules for the basis-core isolation boundary
- [`docs/architecture/compatibility-philosophy.md`](docs/architecture/compatibility-philosophy.md) — compatibility and stability commitments for shared contracts
- [`docs/architecture/reference-vs-implementation.md`](docs/architecture/reference-vs-implementation.md) — the distinction between conceptual architecture, reference architecture, and implementation
- [`docs/architecture/operation-aware-authorization-model.md`](docs/architecture/operation-aware-authorization-model.md) — the conceptual model for expanding basis-core into an operation-aware OT authorization kernel
- [`docs/architecture/operation-aware-evaluation-semantics.md`](docs/architecture/operation-aware-evaluation-semantics.md) — deterministic evaluation semantics for the operation-aware model: default deny, `NOT_APPLICABLE`, deny precedence, conflict resolution, missing context, and safe error handling
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](docs/architecture/operation-aware-trace-audit-evidence.md) — the conceptual trace and audit evidence model: trace vs. audit, evidence lifecycle, redaction, reason codes, and evidence assembly ownership
- [`docs/architecture/operation-aware-policy-rule-model.md`](docs/architecture/operation-aware-policy-rule-model.md) — the conceptual policy bundle and rule model: bundle scope, rule effects and match criteria, conditions, combining semantics, validation, and reason codes
- [`docs/architecture/operation-aware-schema-readiness-plan.md`](docs/architecture/operation-aware-schema-readiness-plan.md) — the schema readiness and migration plan for moving the operation-aware architecture into basis-schemas: contract surfaces, publication order, compatibility rules, and ownership
- [`docs/adr/README.md`](docs/adr/README.md) — the ADR process and when an ADR is required

### For Contributors

- [`CONTRIBUTING.md`](CONTRIBUTING.md) — contribution scope, writing expectations, and review criteria
- [`docs/standards/writing-guidelines.md`](docs/standards/writing-guidelines.md) — tone, prohibited language, and style
- [`docs/standards/terminology-rules.md`](docs/standards/terminology-rules.md) — canonical terminology and rules for introducing new terms
- [`docs/standards/diagram-standards.md`](docs/standards/diagram-standards.md) — diagram conventions and lifecycle rules
- [`GOVERNANCE.md`](GOVERNANCE.md) — governance model, ADR requirements, and basis-core boundary protection policy

### For Implementation Repositories

- [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md) — component boundaries, dependency direction, and what belongs in each component
- [`docs/kernel-boundary-rules.md`](docs/kernel-boundary-rules.md) — the non-negotiable rules for basis-core, including the boundary decision test and import hierarchy
- [`docs/architecture/action-vocabulary.md`](docs/architecture/action-vocabulary.md) — action naming structure, conventions, and stability expectations
- [`docs/architecture/compatibility-philosophy.md`](docs/architecture/compatibility-philosophy.md) — breaking change expectations and schema evolution philosophy
- [`docs/architecture/operation-aware-authorization-model.md`](docs/architecture/operation-aware-authorization-model.md) — conceptual direction for the richer DecisionRequest/DecisionResponse context basis-core v0.2.0 is expected to support
- [`docs/architecture/operation-aware-evaluation-semantics.md`](docs/architecture/operation-aware-evaluation-semantics.md) — the evaluation semantics basis-core v0.2.0 is expected to implement: default deny, deny precedence, conflict resolution, and safe error handling
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](docs/architecture/operation-aware-trace-audit-evidence.md) — the trace and audit evidence categories basis-core, basis-gateway, basis-adapters, and basis-identity are each expected to contribute
- [`docs/architecture/operation-aware-policy-rule-model.md`](docs/architecture/operation-aware-policy-rule-model.md) — the conceptual policy bundle and rule structure basis-schemas is expected to formalize and basis-core v0.2.0 is expected to validate and evaluate
- [`docs/architecture/operation-aware-schema-readiness-plan.md`](docs/architecture/operation-aware-schema-readiness-plan.md) — the ordered publication sequence and compatibility expectations the next basis-schemas expansion is expected to follow
- [`docs/adr/README.md`](docs/adr/README.md) — when implementation decisions require an ADR in this repository

---

## Relationship to BASIS PoC and basis-core

The BASIS proof-of-concept (basis-poc) is a research implementation developed to validate the architectural patterns described in the white papers. It is not a production system and should not be treated as one. References to the PoC in this repository are illustrative — they demonstrate that specific architectural patterns are implementable, not that the implementation is production-ready or that the specific technology choices are prescriptive.

The PoC is also distinct from basis-core. Where the PoC is a breadth-first research artifact that validated core mechanisms in a monolithic, containerized form, basis-core is the isolated authorization kernel that represents the architectural direction that emerged from the lessons the PoC surfaced. The PoC demonstrated that identity propagation, policy evaluation, and protocol-agnostic enforcement are logically coherent and implementable. basis-core defines those mechanisms as stable, separately deployable evaluation semantics that higher-level services depend on.

Section 06 of the white paper ("The BASIS Proof-of-Concept") addresses this distinction directly, including a dedicated subsection titled "From PoC to Core Services Distribution."

The PoC implementation repositories are separate from this architecture repository. This repository documents the *why* and *what* of the architecture; the implementation repositories contain the *how*.

---

## Scope of Research

The primary research questions addressed by this repository are:

1. What does identity-aware authorization look like when applied to OT environments, given the operational constraints — legacy protocols, long asset lifecycles, availability requirements, limited device compute — that those environments impose?

2. Where does centralized policy evaluation become difficult to implement, and what architectural accommodations are required to address those difficulties without undermining the authorization model?

3. What threats does an identity-aware authorization layer address, what threats does it not address, and what residual risks remain after the authorization layer is in place?

4. What are the production engineering challenges — high availability, policy distribution, cache synchronization, credential lifecycle, governance ownership — that must be addressed before patterns like those explored here can be deployed and operated at scale?

These questions are examined through a combination of architectural analysis and proof-of-concept implementation. The PoC validates the core mechanisms at limited scope; the production analysis identifies where the distance between a coherent architecture and a deployable system is larger than the conceptual description implies.

---

## Repository Structure

```text
basis-architecture/
├── README.md                          # This file
├── LICENSE
│
├── docs/
│   ├── glossary.md                    # Canonical terminology definitions
│   ├── architecture-principles.md    # Guiding architectural principles
│   ├── architecture/
│   │   └── basis-ecosystem.md        # Ecosystem structure: Foundation, distribution, BASAuth, component boundaries
│   └── standards/
│       ├── diagram-standards.md      # Diagram conventions, categories, and visual style
│       ├── terminology-guidelines.md # Controlled terminology and usage rules
│       └── writing-guidelines.md     # Writing tone, prohibited language, and style guidance
│
└── whitepapers/
    └── identity-aware-authorization-for-operational-technology/
        ├── paper.md                   # Master document: abstract, TOC, section summaries
        ├── sections/                  # Individual paper sections
        │   ├── 01-non-goals.md
        │   ├── 02-current-state-of-ot-authorization.md
        │   ├── 03-why-existing-approaches-are-incomplete.md
        │   ├── 04-identity-aware-ot-architecture.md
        │   ├── 05-ot-trust-boundaries.md
        │   ├── 06-basis-proof-of-concept.md
        │   ├── 07-production-realities-and-constraints.md
        │   ├── 08-threat-modeling-and-security-considerations.md
        │   ├── 09-future-direction.md
        │   └── 10-conclusion.md
        ├── diagrams/                  # Architectural analysis diagrams
        │   ├── README.md              # Diagram index and conventions
        │   ├── identity-aware-authorization-flow.mmd
        │   ├── identity-aware-authorization-flow-spec.md
        │   ├── ot-trust-boundary-overview.mmd
        │   ├── ot-trust-boundary-overview-spec.md
        │   ├── basis-poc-architecture-mapping.mmd
        │   └── basis-poc-architecture-mapping-spec.md
        ├── references/
        │   └── sources.md
        └── exported/                  # Exported diagram images for publication
```

---

## White Papers

### Identity-Aware Authorization for Operational Technology

**Location:** [`whitepapers/identity-aware-authorization-for-operational-technology/`](whitepapers/identity-aware-authorization-for-operational-technology/)

**Status:** Mature Draft — all ten sections complete

This paper examines how identity-aware authorization can be applied to operational technology environments, with building automation systems as the primary domain. It covers the current state of OT authorization, the architectural patterns required for identity-aware models, trust boundary analysis, a proof-of-concept implementation against those patterns, the production engineering realities that govern what deployment actually requires, architectural threat analysis specific to the authorization layer, open engineering problems in identity-aware OT authorization, and a concluding synthesis of the paper's architectural and operational findings.

The paper's analysis emphasizes operational realism throughout. It treats OT-specific constraints — long asset lifecycles, availability requirements, intermittent connectivity, legacy protocol stacks, narrow maintenance windows — as first-order design inputs rather than edge cases. Sections on production deployment address synchronization complexity, degraded operation, governance ownership, and distributed coordination tradeoffs that architectural descriptions typically abstract away. The paper's consistent position is that identity-aware authorization redistributes operational complexity rather than eliminating it, and that deployment realities shape the architecture more than conceptual purity can.

The master document at [`paper.md`](whitepapers/identity-aware-authorization-for-operational-technology/paper.md) provides the abstract, table of contents, section status, section summaries, diagram index, and editorial notes.

**Sections:**
- [01 — Non-Goals](whitepapers/identity-aware-authorization-for-operational-technology/sections/01-non-goals.md) — explicit scope boundaries
- [02 — Current State of OT Authorization](whitepapers/identity-aware-authorization-for-operational-technology/sections/02-current-state-of-ot-authorization.md) — how authorization works in OT environments today
- [03 — Why Existing Approaches Become Insufficient](whitepapers/identity-aware-authorization-for-operational-technology/sections/03-why-existing-approaches-are-incomplete.md) — structural limitations of network-centric and controller-local models
- [04 — Identity-Aware OT Architecture](whitepapers/identity-aware-authorization-for-operational-technology/sections/04-identity-aware-ot-architecture.md) — conceptual architecture: policy evaluation, enforcement, protocol adaptation, audit
- [05 — OT Trust Boundaries](whitepapers/identity-aware-authorization-for-operational-technology/sections/05-ot-trust-boundaries.md) — trust boundary taxonomy and enforcement requirements at each zone boundary
- [06 — The BASIS Proof-of-Concept](whitepapers/identity-aware-authorization-for-operational-technology/sections/06-basis-proof-of-concept.md) — PoC scope, implementation analysis, validated claims, and unvalidated properties
- [07 — Production Realities and Constraints](whitepapers/identity-aware-authorization-for-operational-technology/sections/07-production-realities-and-constraints.md) — operational engineering challenges: HA, policy distribution, cache synchronization, lifecycle management, governance
- [08 — Threat Modeling and Security Considerations](whitepapers/identity-aware-authorization-for-operational-technology/sections/08-threat-modeling-and-security-considerations.md) — threat analysis across nine categories with architectural mitigations and residual risk assessment
- [09 — Future Direction](whitepapers/identity-aware-authorization-for-operational-technology/sections/09-future-direction.md) — open engineering problems: policy federation, device identity at scale, safety-authorization interaction, governance evolution, and the structural constraints that architectural refinement is unlikely to resolve
- [10 — Conclusion](whitepapers/identity-aware-authorization-for-operational-technology/sections/10-conclusion.md) — synthesis of the paper's architectural and operational analysis; authorization as operational infrastructure; complexity redistribution; long-term architectural uncertainty; final perspective

---

## Architecture Standards

### Ecosystem Structure

[`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md)

Describes the three layers of the BASIS ecosystem — the Basis Foundation, the BASIS Core Services Distribution, and BASAuth — along with the component structure of the distribution (basis-core, basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, basis-schemas), the dependency rules that govern relationships between them, and the boundaries that define what belongs in basis-core and what must stay outside it.

### Kernel Boundary Rules

[`docs/kernel-boundary-rules.md`](docs/kernel-boundary-rules.md)

The enforceable rules that protect basis-core as an isolated authorization kernel. Defines what may enter the kernel, what must stay outside it, the allowed subpackage responsibilities, the disallowed concerns and named technologies, a boundary decision test for evaluating proposed changes, the import boundary summary, and the relationship of basis-core to each surrounding component. This document is the review reference for any proposed change to basis-core.

### Glossary

[`docs/glossary.md`](docs/glossary.md)

Defines the terminology used throughout the white papers, ADRs, and supporting documentation. All technical terms should be consistent with this document. New terms should be added here rather than defined inline in individual sections.

### Architecture Principles

[`docs/architecture-principles.md`](docs/architecture-principles.md)

Establishes the guiding principles for identity-aware authorization systems in OT environments. These principles are intended to be stable enough to guide design decisions across implementation phases and specific enough to evaluate competing approaches. Fifteen principles are defined, covering topics from explicit authorization and identity propagation to operational resilience and incremental adoption.

### Writing Guidelines

[`docs/standards/writing-guidelines.md`](docs/standards/writing-guidelines.md)

Internal writing standards for all content in this repository. Covers tone, prohibited language, terminology consistency, BASIS positioning, citation guidance, diagram referencing, capitalization conventions, and speculative language. All contributors should read this document before drafting new content.

---

## Diagrams

### Diagram Standards

[`docs/standards/diagram-standards.md`](docs/standards/diagram-standards.md)

Establishes standards for all diagrams committed to this repository. Covers diagram categories, visual style, trust boundary conventions, color usage, labeling standards, simplification rules, recommended diagram types, Mermaid vs Draw.io guidance, file naming conventions, versioning, and accessibility. All diagrams should conform to these standards.

### Whitepaper Diagrams

Diagrams specific to individual white papers are maintained within each paper's `diagrams/` subdirectory. These diagrams are architectural analysis artifacts — they make structural relationships, component responsibilities, and flow semantics explicit, and they are intended to be read alongside the section text rather than as standalone illustrations. Each diagram has a companion specification document that describes the rendering guidance, component annotations, and intentional omissions that the diagram source file alone does not convey.

Diagrams for the identity-aware authorization paper:

- [`identity-aware-authorization-flow.mmd`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/identity-aware-authorization-flow.mmd) / [`-spec.md`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/identity-aware-authorization-flow-spec.md) — Full architecture diagram: trust zones, subjects, enforcement points, protocol adapters, policy engine, audit pipeline, and control/data plane flows
- [`ot-trust-boundary-overview.mmd`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/ot-trust-boundary-overview.mmd) / [`-spec.md`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/ot-trust-boundary-overview-spec.md) — Trust boundary overview: zone structure, enforcement point positions, and cross-boundary flow directions
- [`basis-poc-architecture-mapping.mmd`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/basis-poc-architecture-mapping.mmd) / [`-spec.md`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/basis-poc-architecture-mapping-spec.md) — PoC architecture mapping: BASIS PoC components annotated with their conceptual architecture roles from Sections 04 and 05

---

## Threat Modeling

Threat analysis is maintained as a section within each white paper rather than as standalone threat model documents. This keeps threat analysis integrated with the architectural context it describes and makes it easier to keep the two in sync as the architecture evolves.

The threat modeling section for the identity-aware authorization paper is at:
[`whitepapers/identity-aware-authorization-for-operational-technology/sections/08-threat-modeling-and-security-considerations.md`](whitepapers/identity-aware-authorization-for-operational-technology/sections/08-threat-modeling-and-security-considerations.md)

It covers nine threat categories with mechanism analysis, architectural mitigations, and residual risk assessment for each. The analysis is positioned within the authorization architecture described in the paper rather than treating OT threat modeling in general terms.

As the repository grows to include ADRs and additional research areas, threat models for specific components may be maintained separately. This README will be updated to reflect that structure when it is introduced.

---

## Repository Philosophy

This repository is a technical reference, not a product presentation. The documentation here is intended to be useful to engineers working on similar problems, researchers studying OT identity and access management, and practitioners evaluating whether identity-aware authorization is appropriate for their OT environments.

The work is honest about what is not known, what has not been validated, and where the proof-of-concept falls short of production readiness. The architecture described here addresses specific, well-defined problems in OT authorization — it does not claim to solve OT security broadly, and it does not treat the constraints of OT environments as problems to be argued away.

A recurring theme across the documentation is that deploying identity-aware authorization in OT environments requires negotiating tradeoffs that the conceptual architecture cannot resolve on its own: between policy currency and operational independence, between security posture and availability, between audit completeness and delivery reliability under degraded connectivity, between precise policy expressiveness and deterministic evaluation behavior. The repository documents these tradeoffs explicitly rather than presenting the architecture as though they do not exist.

Contributions to this repository should preserve the analytical, restrained tone established in the existing documentation. The writing guidelines document describes this in more detail. The goal is documentation that is useful to a careful technical reader, not documentation that is persuasive to a general audience.
