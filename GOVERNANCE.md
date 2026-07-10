# Governance

This document describes the governance model for the BASIS ecosystem at its current stage of development. The model is intentionally lightweight — it reflects the actual organizational state of an early-stage project rather than projecting governance processes that do not yet exist.

Governance will evolve as the ecosystem matures. This document will be updated to reflect that evolution.

---

## Governing Body

The **Basis Foundation** is the stewardship body for the open-source BASIS ecosystem. It is responsible for architectural consistency, compatibility protection, review standards, repository stewardship, and documentation quality across the BASIS Core Services Distribution and this architecture repository.

The Foundation's current role is primarily architectural: defining and maintaining the standards, principles, and decision records that determine what belongs in the open-source ecosystem and how components should relate to each other. The organizational form of the Foundation and its formal membership processes are matters for future determination as the ecosystem grows.

At this stage, Foundation governance operates through:

- the architecture documentation and white papers in `basis-architecture`
- the architectural standards defined in [`docs/standards/`](docs/standards/)
- the ecosystem structure defined in [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md)
- the guiding principles in [`docs/architecture-principles.md`](docs/architecture-principles.md)
- Architecture Decision Records (ADRs), which document significant architectural choices and their rationale

---

## What Governance Covers

### Architectural Consistency

Contributions to the BASIS ecosystem must be consistent with the architectural model established in the white papers and architecture documents. The architecture is not a suggestion — it is the design framework that components must conform to. Changes that introduce inconsistency with the established model require explicit architectural review and, where significant, an ADR that documents the reasoning.

The canonical sources for architectural consistency are:

- [`docs/architecture-principles.md`](docs/architecture-principles.md) — the fifteen principles that guide design decisions
- [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md) — the component structure and dependency rules
- [`docs/glossary.md`](docs/glossary.md) — the shared terminology that must be used consistently
- [`whitepapers/`](whitepapers/) — the detailed architectural analysis and design rationale

### Compatibility Protection

The schemas, contracts, and interfaces defined in `basis-schemas` are compatibility surfaces — they determine whether components built against one version of the distribution can interoperate with other components built against the same version. Changes to these surfaces require review that considers not only functional correctness but the impact on deployed systems and on components that implement the contracts. As of `basis-schemas` v0.2.0, twenty such contracts are published and versioned, each carrying an explicit lifecycle state; the review requirement in this section applies to all of them.

Changes to `basis-core` evaluation semantics carry the same protection. The kernel's behavior is a contract: enforcement points and gateways depend on it. A change to evaluation behavior that is not explicitly versioned and documented breaks that contract in ways that may not be immediately visible.

### Review Standards

All contributions to this architecture repository are reviewed against the standards in [`docs/standards/writing-guidelines.md`](docs/standards/writing-guidelines.md) and [`docs/standards/diagram-standards.md`](docs/standards/diagram-standards.md). Reviews assess:

- analytical accuracy and operational realism
- terminology consistency with the glossary
- appropriate epistemic restraint (distinguishing what is known, inferred, or uncertain)
- architectural consistency with existing documentation
- absence of marketing language, unsupported claims, or forward-looking promises

These criteria apply regardless of who submits the contribution.

### Repository Stewardship

The `basis-architecture` repository is the canonical source of architectural decisions, design principles, and system documentation for the BASIS ecosystem. It is not an implementation repository, and it does not contain application code. Its purpose is to document the *why* and *what* of the architecture with enough precision and analytical depth to guide implementation work and to serve as a reference for engineers evaluating or operating the system.

Repository stewardship means maintaining that purpose against pressure to add content that does not serve it: product marketing, speculative roadmap language, implementation details that belong in implementation repositories, or generic OT security content unrelated to the architecture under analysis.

### Documentation Quality

Documentation quality is a first-order concern, not an afterthought. The writing standards in this repository are strict because the documentation's value depends on its precision and its honesty about limitations. Documentation that overstates what the architecture provides, understates the operational challenges it introduces, or obscures residual risk is worse than no documentation.

---

## Architecture Decision Records

Architecture Decision Records (ADRs) are the formal mechanism for documenting significant architectural choices. An ADR records what was decided, what alternatives were considered, why the chosen approach was selected, and what consequences the decision carries. ADRs are the right place for arguments about design direction — not white paper prose, not pull request comments.

ADRs should be created when:

- a significant component boundary or dependency rule is being established
- a design alternative was seriously considered and rejected
- an implementation choice requires understanding the architectural reasoning to be evaluated correctly
- a future implementer would reasonably ask "why was this done this way?" without an obvious answer

ADRs are not required for contributions that are consistent with already-established decisions. They are required for contributions that change established decisions or establish new ones.

The ADR directory is located at `docs/adr/` and follows the format established there. When no ADRs yet exist, the first ADR in a given area should establish the format and the decision being documented.

---

## basis-core Boundary Protection

The isolation boundary of `basis-core` is a governance concern, not merely a coding convention. This bears emphasis because the boundary will face pressure over time — from convenience, from a desire to avoid architectural complexity, and from the natural tendency to solve problems by adding capabilities to shared components.

The boundary rule is stated clearly in [`docs/architecture/basis-ecosystem.md`](docs/architecture/basis-ecosystem.md): basis-core must not depend on basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, any BASAuth component, cloud platform SDKs, specific identity providers, database runtimes, UI frameworks, or protocol stacks.

Any proposed contribution that adds such a dependency to basis-core requires:

- an explicit architectural justification for why the dependency cannot be satisfied outside the kernel
- an ADR documenting the decision and the alternatives considered
- Foundation review before the change is accepted

The same protection applies to `basis-schemas`. Schemas that are compatible surfaces for the entire ecosystem must not be changed in ways that break implemented contracts without explicit versioning and migration guidance.

The reason for treating these boundary rules as governance concerns rather than implementation details is that the value of kernel isolation degrades invisibly when dependencies accumulate incrementally. Each individual violation may appear harmless; the aggregate effect is a kernel that cannot be tested independently, cannot be deployed portably, and cannot be maintained stably. Governance is the mechanism that prevents incremental erosion.

---

## Commercial Ownership and Open-Source Governance

**BASAuth does not govern the open-source ecosystem.** BASAuth is a commercial entity that builds enterprise services on top of the BASIS Core Services Distribution. Its commercial interests and its engineering work are downstream of the open-source architecture — they depend on it and may influence proposals to it, but they do not control it.

Contributions from BASAuth personnel to this repository or to open-source distribution components are welcome and are evaluated by the same review criteria as contributions from any other source. BASAuth personnel who contribute to the open-source work do so as contributors to the open-source project, not as representatives of commercial authority over it.

The organizational boundary between Foundation governance and commercial ownership is important for the long-term health of the ecosystem. It ensures that the open-source distribution remains genuinely useful independently of any commercial relationship, and that design decisions in the open-source components are made on architectural grounds rather than commercial ones.

---

## License

The content in this repository is licensed under the Apache License, Version 2.0. See [`LICENSE`](LICENSE) for the full license text.

**Open issue:** The LICENSE file currently contains an unfilled copyright placeholder (`[yyyy] [name of copyright owner]`). The copyright attribution for this repository has not yet been formally determined. This requires a decision by the Foundation about the appropriate copyright holder — whether the Foundation itself, individual contributors, or another legal entity. Until this is resolved, contributions are made to the project under the Apache 2.0 terms, but the copyright attribution question remains open. This should be resolved before the repository is publicly released or before the Foundation is formally constituted.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for contribution terms. The Apache 2.0 license applies to the documentation and any configuration or specification artifacts in this repository. Code components in separate implementation repositories may carry the same or different licenses; those repositories will specify their own licensing.

---

## Current Governance Maturity

This governance model is appropriate for the current stage of the project: a small number of contributors, an architecture that is substantially defined and whose implementation is now underway across separate repositories, and an organizational structure that is still being determined.

Implementation repositories are now established alongside this architecture repository: `basis-core`, `basis-gateway`, `basis-adapters`, `basis-console`, `basis-identity`, and `basis-schemas` each exist as separately maintained, versioned repositories. `basis-schemas` publishes released, versioned contracts (`v0.2.0`) that the implementation repositories consume, and compatibility/versioning policies for those contracts already exist at the repository level — `basis-schemas`'s `docs/contract-governance.md` and `basis-core`'s `docs/breaking-change-discipline.md` — though they are not yet restated or cross-referenced from this document. `basis-deploy` remains the one distribution component not yet established as a repository.

The governance model is expected to continue to evolve as:

- the contributor base grows and contributions from new parties require clearer processes
- the Foundation takes formal organizational shape
- `basis-deploy` and any further ecosystem components are established

Future additions to this document may include: formal contribution roles, an RFC or proposal process for significant changes, a cross-repository index of the versioning and compatibility policies each implementation repository already maintains, and organizational membership structures for the Foundation. None of these are described here because they do not yet exist in a form that would be accurate to document at the Foundation level.
