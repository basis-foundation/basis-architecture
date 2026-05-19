# BASIS Architecture

This repository contains architecture documentation, white papers, threat models, architecture decision records (ADRs), and diagramming standards for the BASIS (Building Automation Secure Identity Service) research project.

BASIS explores identity-aware authorization, centralized policy enforcement, and auditability for operational technology (OT) environments. The work is focused on building automation systems (BAS) as a primary domain, with architectural patterns intended to be broadly applicable across OT contexts including data centers, hospitals, campuses, commercial buildings, and industrial facilities.

---

## Repository Purpose

This repository is the canonical source of architectural documentation for the BASIS initiative. It is organized to support long-term research continuity: each document type has a defined location, each white paper is self-contained within its own directory, and shared standards apply across all content.

The repository does not contain application code, deployment configuration, or operational tooling. Those artifacts are maintained in the BASIS implementation repositories. What is here is architecture: the reasoning behind design decisions, the threat analysis that informs them, the standards that keep documentation consistent, and the diagrams that make structural relationships visible.

---

## Relationship to BASIS PoC

The BASIS proof-of-concept is a research implementation developed to validate the architectural patterns described in the white papers. It is not a production system and should not be treated as one. References to the PoC in this repository are illustrative — they demonstrate that a specific architectural pattern is implementable, not that the implementation is production-ready or that the specific technology choices are prescriptive.

The PoC implementation repositories are separate from this architecture repository. This repository documents the *why* and *what* of the architecture; the implementation repositories contain the *how*.

---

## Scope of Research

The primary research questions addressed by this repository are:

1. What does identity-aware authorization look like when applied to OT environments, given the operational constraints — legacy protocols, long asset lifecycles, availability requirements, limited device compute — that those environments impose?

2. Where does centralized policy evaluation become difficult to implement, and what architectural accommodations are required to address those difficulties without undermining the authorization model?

3. What threats does an identity-aware authorization layer address, what threats does it not address, and what residual risks remain after the authorization layer is in place?

4. What are the production engineering challenges — high availability, latency, offline operation, credential lifecycle — that would need to be resolved before patterns like those explored here could be deployed at scale?

These questions are examined through a combination of architectural analysis and proof-of-concept implementation. The research is ongoing and the answers are not complete.

---

## Repository Structure

```
basis-architecture/
├── README.md                          # This file
├── LICENSE
│
├── docs/
│   ├── glossary.md                    # Canonical terminology definitions
│   ├── architecture-principles.md    # Guiding architectural principles
│   ├── diagrams/
│   │   └── README.md                 # Diagram standards and conventions
│   └── standards/
│       └── writing-guidelines.md     # Writing tone, terminology, and style guidance
│
└── whitepapers/
    └── identity-aware-authorization-for-operational-technology/
        ├── paper.md                   # Master document: abstract, TOC, section summaries
        ├── README.md                  # Paper overview and status
        ├── sections/                  # Individual paper sections
        │   ├── 01b-non-goals.md
        │   ├── 02-current-state-of-ot-authorization.md
        │   └── 09-threat-modeling-and-security-considerations.md
        ├── diagrams/                  # Diagrams specific to this paper
        │   ├── ot-trust-boundary-overview.mmd
        │   └── ot-trust-boundary-overview-spec.md
        ├── references/
        │   └── sources.md
        └── exported/                  # Exported diagram images for publication
```

---

## White Papers

### Identity-Aware Authorization for Operational Technology

**Location:** [`whitepapers/identity-aware-authorization-for-operational-technology/`](whitepapers/identity-aware-authorization-for-operational-technology/)

**Status:** Draft — active development

This paper examines how identity-aware authorization can be applied to operational technology environments, with building automation systems as the primary domain. It covers the current state of OT authorization, the architectural patterns required for identity-aware models, the operational constraints that shape implementation, threat analysis, and open engineering questions.

The master document at [`paper.md`](whitepapers/identity-aware-authorization-for-operational-technology/paper.md) provides the abstract, table of contents, section status, and editorial notes.

**Completed sections:**
- [Non-Goals](whitepapers/identity-aware-authorization-for-operational-technology/sections/01b-non-goals.md) — scope boundaries for the paper
- [Current State of OT Authorization](whitepapers/identity-aware-authorization-for-operational-technology/sections/02-current-state-of-ot-authorization.md) — how authorization is handled in OT environments today
- [Threat Modeling and Security Considerations](whitepapers/identity-aware-authorization-for-operational-technology/sections/09-threat-modeling-and-security-considerations.md) — threat analysis specific to the authorization architecture

---

## Architecture Standards

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

[`docs/diagrams/README.md`](docs/diagrams/README.md)

Establishes standards for all diagrams committed to this repository. Covers diagram categories, visual style, trust boundary conventions, color usage, labeling standards, simplification rules, recommended diagram types, Mermaid vs Draw.io guidance, file naming conventions, versioning, and accessibility. All diagrams should conform to these standards.

### Whitepaper Diagrams

Diagrams specific to individual white papers are maintained within each paper's `diagrams/` subdirectory. The trust boundary overview diagram for the identity-aware authorization paper is at:

- [`whitepapers/identity-aware-authorization-for-operational-technology/diagrams/ot-trust-boundary-overview.mmd`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/ot-trust-boundary-overview.mmd) — Mermaid source
- [`whitepapers/identity-aware-authorization-for-operational-technology/diagrams/ot-trust-boundary-overview-spec.md`](whitepapers/identity-aware-authorization-for-operational-technology/diagrams/ot-trust-boundary-overview-spec.md) — Full diagram specification including Draw.io guidance

---

## Threat Modeling

Threat analysis is maintained as a section within each white paper rather than as standalone threat model documents. This keeps threat analysis adjacent to the architecture it describes and makes it easier to keep the two in sync as the architecture evolves.

The threat modeling section for the identity-aware authorization paper is at:
[`whitepapers/identity-aware-authorization-for-operational-technology/sections/09-threat-modeling-and-security-considerations.md`](whitepapers/identity-aware-authorization-for-operational-technology/sections/09-threat-modeling-and-security-considerations.md)

It covers nine threat categories with mechanism analysis, architectural mitigations, and residual risk assessment for each.

As the repository grows to include ADRs and additional research areas, threat models for specific components may be maintained separately. This README will be updated to reflect that structure when it is introduced.

---

## Repository Philosophy

This repository is a technical reference, not a product presentation. The documentation here is intended to be useful to engineers working on similar problems, researchers studying OT identity and access management, and practitioners evaluating whether identity-aware authorization is appropriate for their OT environments.

The work is honest about what is not known, what has not been validated, and what the current proof-of-concept does not demonstrate. The architecture described here addresses specific, well-defined problems in OT authorization — it does not claim to solve OT security broadly.

Contributions to this repository should preserve the analytical, restrained tone established in the existing documentation. The writing guidelines document describes this in more detail. The goal is documentation that is useful to a careful technical reader, not documentation that is persuasive to a general audience.
