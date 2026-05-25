# Contributing

This repository contains architecture documentation, white papers, diagrams, references, and research material for the BASIS ecosystem. Contributions should preserve the repository's restrained engineering tone and operationally grounded analysis.

Before contributing, it is worth understanding the distinction between this repository and the implementation repositories. `basis-architecture` is the canonical source of architectural decisions, design principles, and system documentation. It does not contain application code. Contributions here address the *why* and *what* of the architecture. Implementation work belongs in the respective component repositories (basis-core, basis-gateway, basis-adapters, and others). The relationship between this repository and those repositories is described in [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md).

## Contribution Scope

Appropriate contributions include:

- corrections to terminology, references, or cross-references
- improvements to architectural clarity or analytical precision
- diagram updates and companion specification documents
- additional operational constraints or tradeoff analysis
- threat-model refinements
- documentation consistency fixes

This repository is not the right place for:

- product marketing or positioning
- speculative roadmap language
- unsupported or exaggerated security claims
- generic OT security content unrelated to the architecture described in the white papers
- changes that characterize the BASIS PoC (basis-poc) as production-ready
- changes that conflate the proof-of-concept with basis-core or with the BASIS Core Services Distribution
- implementation details that belong in component repositories, not in architecture documentation

## Writing Expectations

Before contributing, review:

- [`docs/glossary.md`](docs/glossary.md) — canonical terminology definitions, including ecosystem terms (Basis Foundation, BASIS Core Services Distribution, BASAuth, basis-core)
- [`docs/architecture-principles.md`](docs/architecture-principles.md) — guiding architectural principles
- [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md) — ecosystem structure, component roles, and dependency rules
- [`docs/standards/writing-guidelines.md`](docs/standards/writing-guidelines.md) — tone, prohibited language, and style (including the updated BASIS positioning guidance in Section 4.1)
- [`docs/standards/diagram-standards.md`](docs/standards/diagram-standards.md) — diagram conventions and visual style

Contributions should use consistent terminology, preserve architectural restraint, acknowledge limitations and residual risk, and treat OT operational constraints as first-order concerns rather than edge cases to be addressed later.

**Terminology precision matters.** The ecosystem involves several related but distinct concepts that must not be conflated: the Basis Foundation (governance body), the BASIS Core Services Distribution (open-source components), BASAuth (future commercial entity), basis-poc (the research proof-of-concept), and basis-core (the isolated authorization kernel specified in the architecture). Using these terms correctly, as defined in the glossary and the ecosystem document, is a contribution requirement.

## Diagrams

Diagram contributions should include source files (`.mmd` or `.drawio`) and, where the diagram is used in a white paper section, a companion specification document following the pattern established in the existing `diagrams/` directories. Exported images belong in the `exported/` subdirectory and should not be committed without a corresponding source file.

## Pull Requests

Use pull requests for all changes. A useful pull request describes what changed, explains why the change improves the repository, and identifies any affected sections, diagrams, or references. Avoid mixing unrelated changes in a single pull request.

## basis-core Boundary Rules

The isolation rules for basis-core are intentionally strict and are treated as a governance concern, not merely a coding convention. Contributions that propose adding dependencies to basis-core — on basis-gateway, basis-console, basis-adapters, basis-deploy, cloud platform SDKs, identity providers, database runtimes, UI frameworks, or protocol stacks — require explicit architectural justification and an Architecture Decision Record before they will be accepted.

The strictness is deliberate: the kernel's value as a stable, portable, independently testable component depends on maintaining these boundaries against incremental erosion. Each individual proposed exception may appear harmless; the aggregate effect of accepting exceptions is a kernel whose isolation is nominal rather than real. When you encounter a design problem that seems to call for relaxing a kernel boundary, the preferred response is to reconsider whether the concern belongs in the kernel at all — not to argue for the specific exception.

See [`GOVERNANCE.md`](GOVERNANCE.md) for the full basis-core boundary protection policy, and [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md) for the list of what must stay outside basis-core.

## Architecture Discussions vs. Implementation Discussions

This repository is the appropriate place for discussions about:

- architectural intent and design rationale
- whether a proposed design is consistent with the established architecture
- how an architectural concern is documented or described
- what belongs in the architecture documentation and what belongs elsewhere

It is not the appropriate place for discussions about:

- implementation details of specific component repositories
- technology choices for implementation (programming language, framework, library selections)
- deployment configuration for specific OT environments
- operational procedures for specific deployments

Implementation discussions belong in the relevant component repositories. If an implementation question surfaces an architectural ambiguity — a case where the architecture documentation does not answer a real design question — the right response is to clarify the architecture documentation in this repository and then carry that clarification into the implementation discussion.

The line between architecture and implementation is not always sharp. When it is unclear, err on the side of clarifying the architecture documentation, because architecture clarity has lasting value across all implementations. Implementation choices that do not reflect architectural intent should surface as architecture discussions, not disappear into implementation repositories where they remain invisible.

## Review Criteria

Reviews prioritize architectural accuracy, operational realism, terminology consistency, and restraint in claims. The goal is not to make the repository more persuasive — it is to make it more accurate, analytically useful, and technically grounded.

The repository's value comes from its willingness to document complexity, residual risk, and unresolved tradeoffs honestly. Contributions that increase precision or surface legitimate constraints are more valuable than contributions that simplify the analysis.
