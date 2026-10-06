# ADR-0019: Basitra Ecosystem Identity and Terminology Hierarchy

## Status

Accepted

Consistent with this repository's established practice (see ADR-0018's Context), this ADR was numbered when first proposed and submitted as `Proposed`, and its merging did not constitute acceptance. It has since undergone that separate, dedicated formal-acceptance review, and its status now records `Accepted`. Acceptance changes no part of the decision below, renames nothing, and authorizes none of the follow-on work in Decision 8, each item of which requires its own bounded PR. The mismatch between this practice and [`README.md`](README.md#numbering), which says numbers are assigned at acceptance, is recorded as open decision OD-5 in the strategy document.

## Context

The repository's canonical terminology currently says:

- **BASIS** expands to *Building Automation Secure Identity Service* ([`writing-guidelines.md`](../standards/writing-guidelines.md) §3.3; [`SECURITY.md`](../../SECURITY.md)).
- **BAS** means *Building Automation System* ([`writing-guidelines.md`](../standards/writing-guidelines.md) §3.3; [`glossary.md`](../glossary.md#building-automation-system-bas)). The repository uses it that way throughout, and building automation systems are the white paper's primary domain.
- The **BASIS ecosystem** has three layers: the Basis Foundation (the governing body), the BASIS Core Services Distribution (the `basis-*` components), and BASAuth (the future commercial entity) ([`basis-ecosystem.md`](../architecture/basis-ecosystem.md)).

Two developments make that terminology incomplete.

**Scope has outgrown the original expansion.** The architecture is described as applying across OT generally ([`README.md`](../../README.md)). `basis-adapters` normalizes non-building-automation protocols such as OPC UA, DNP3, and IEC 61850. Cross-sector applicability is an open roadmap question ([`ROADMAP.md`](../../ROADMAP.md) Phase 5).

**The accepted architecture has converged on a recognizable security model.** Trust is established separately at each boundary a governed operation crosses, never inherited from the previous hop. The boundaries in question are:

- identity verification at the gateway;
- producer admission (ADR-0008, ADR-0009);
- producer intake (ADR-0018);
- authorization-to-execution binding (ADR-0011, ADR-0012);
- evidence (ADR-0007, ADR-0013, ADR-0014).

ADR-0018 states the rule directly: each arrow on the operation path "is a separate boundary with its own admission or verification rule. None of them inherits trust from the one above it." No term currently names this model.

There is also a practical naming need:

- a public identity for the open-source project and its community that does not depend on any one component name or domain;
- a way to describe the relationship between that identity, the security model, and the established BASIS component family.

An external consumer ecosystem, Ipotio, has already begun using "Basitra" informally in its own architecture documents.

The full analysis is in [`docs/architecture/basitra-ecosystem-and-boundary-aware-security.md`](../architecture/basitra-ecosystem-and-boundary-aware-security.md) (the "strategy document"). It covers the current, historical, and proposed terminology register; eight terminology conflicts (C-1 through C-8); the definition of Boundary-Aware Security and its mapping to existing architecture, with a status for each mapping; the gaps before BASac could be formalized; and a staged adoption roadmap.

## Decision

On acceptance, the following terminology decisions take effect. None of them renames a repository, organization, distribution, import namespace, or contract, and none changes any architectural semantics.

1. **Basitra.** *Basitra* is adopted as the intended long-term canonical identity of the open-source project, its community, its ecosystem, and eventually its organizational and GitHub namespace.
   - It is a proper noun and not an acronym.
   - It names the project and community, not a component, runtime, or service.
   - It does not yet name a governance body. The Basis Foundation remains the governing body under [`GOVERNANCE.md`](../../GOVERNANCE.md) today.
   - The long-term relationship between Basitra and the Basis Foundation is an explicit future decision, not an assumption of permanent coexistence. That includes possible rename or succession (strategy document OD-1) and migration of the `basis-foundation` GitHub organization toward a Basitra identity (OD-7).

2. **Boundary-Aware Security.** *Boundary-Aware Security* is adopted as the name of the architectural security approach defined in strategy document §5.1.
   - It describes and organizes existing architecture.
   - It adds no kernel semantics, evaluation behavior, contract, or implementation obligation.
   - Using the term must follow the status mapping in strategy document §5.3 and §6. In particular, it must not imply that environmental boundaries (zone crossing, administrative domain, tenant, device identity) are modeled or enforced before architecture work makes them so.

3. **BASIS re-expansion (forward-only).** The canonical expansion of BASIS becomes *Boundary-Aware Secure Identity Service*.
   - *Building Automation Secure Identity Service* remains the accurate expansion for everything written under it: ADRs, release notes, white-paper text, the `basis-poc` artifact, and prior documentation. Those are not rewritten.
   - This decision changes the expansion only. It does not decide the long-term scope of BASIS within Basitra, which is an explicit future architecture decision (strategy document OD-8). BASIS may remain a component family, become a narrower identity and security subsystem, or take another role. That scope is derived from the Basitra target architecture, not from current repository names. It is not assumed that BASIS permanently remains the umbrella name for every current `basis-*` component.
   - BASIS is not redefined as `basis-core`, and nothing is renamed.
   - Rationale: BASIS originated in building automation, but the architecture and project scope have broadened toward operational technology generally. *Boundary-Aware* describes the security model the accepted architecture is formalizing better than a single-domain label does.
   - BASIS is treated as a standalone acronym, not as "BAS + IS" under either meaning of BAS (strategy document C-2).

4. **BAS abbreviation convention.** The abbreviation BAS is not redefined.
   - In this repository, bare "BAS" continues to mean *building automation system*.
   - Boundary-Aware Security is written in full, abbreviated only as "Boundary-Aware Security (BAS)" at first use.
   - In any document that also discusses building automation systems, Boundary-Aware Security is spelled out at every use.
   - The full convention is strategy document §5.5. The writing-guidelines capitalization table gains a context-qualified row; the existing row is not replaced.

5. **BASac is prospective only.** *BASac* (Boundary-Aware Secure Access Control) is reserved as a possible name for a future formal access-control model.
   - It carries no normative meaning.
   - It must not be used as a synonym for the operation-aware authorization model ([`terminology-rules.md`](../standards/terminology-rules.md#introducing-new-terms)).
   - Giving it normative meaning requires its own ADR, after the gaps in strategy document §9.3 are addressed, beginning with whether the term is needed at all.

6. **Hierarchy and naming root.**
   - Basitra is the umbrella identity. Boundary-Aware Security is its conceptual root. BASIS is, today, the component family, with its long-term scope to be decided (OD-8). BASac is a prospective model name.
   - "BAS" may act as a productive naming root only under the five criteria in strategy document §10.2. BAS-derived names are optional vocabulary, used only where they correspond to genuine architectural concepts, and are never a requirement imposed on components.
   - BASAuth is not a member of this open-source vocabulary and is not governed through Basitra community processes (strategy document C-4).

7. **Governing naming and migration principle.** *Historical continuity informs migration but does not constrain the target architecture.*
   - Basitra's future terminology, repository organization, component boundaries, and public identity are chosen according to the project's intended long-term architecture.
   - Existing BASIS names are preserved where they remain useful and migrated, narrowed, or retired where they do not. Historical existence alone is not a justification for keeping a name.
   - Historical ADRs, releases, tags, and artifacts remain accurate and are not rewritten. Current, active project surfaces may be migrated to Basitra terminology over time through bounded PRs.
   - Repository, package, organization, and public-site migrations may be appropriate where justified, each with its own compatibility and migration plan.
   - No repository rename is authorized by this ADR. A future Basitra target-architecture and repository-naming reconciliation (strategy document §12.4, OD-9) evaluates each repository by asking: *if Basitra had been the project identity from the beginning, what should this component and repository be called based on its architectural responsibility?* A current name may remain if it still answers that question well.

8. **Follow-on scope.** Acceptance makes the following eligible for bounded follow-on PRs. It does not perform them:
   - adding a context-qualified BAS row to the writing-guidelines capitalization table and updating the BASIS row to the forward expansion;
   - updating the `SECURITY.md` scope line;
   - promoting the glossary's proposed-terminology block to canonical entries;
   - extending the terminology-rules entity section to define Basitra and its relationship to the BASIS ecosystem (together with strategy document C-7);
   - per-repository current-documentation reconciliation under strategy document §12.

## Alternatives Considered

**Keep the current terminology unchanged.** Rejected. It leaves the project without a domain-independent public identity. It leaves the boundary-by-boundary security model the accepted ADR chain has converged on without a name. And it leaves a single-domain acronym expansion describing a scope that already extends beyond that domain.

**Keep BASIS's current expansion and add Basitra and Boundary-Aware Security above it.** Viable, and less disruptive. Not preferred: the "Building Automation" expansion would keep signaling a narrower scope than the architecture now has. It would also leave BASIS as the only name in the family that does not describe the security model. This alternative remained available had reviewers judged the re-expansion not worth the churn. Decisions 1, 2, and 4–6 do not depend on Decision 3.

**Rename repositories and packages to `basitra-*` in this ADR.** Rejected for this ADR, not as an eventual outcome. Names should follow the target architecture, so the long-term scope of BASIS (OD-8) and repository naming (OD-9) must be decided first. A mechanical `basitra-*` rename now could require a second migration. Import-namespace changes are also breaking changes under [`compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) and need their own migration planning. Package and distribution conventions are decided afterward (strategy document §12.5, OD-10).

**Treat existing `basis-*` repository names and the `basis-foundation` organization as permanent defaults.** Rejected. It would let historical accident constrain the target architecture. Current names are kept where they still fit their architectural responsibility, not because they exist.

**Make Basitra an acronym.** Rejected. It would force a contrived expansion and tie the umbrella identity to one description of its scope.

**Redefine BAS universally as Boundary-Aware Security.** Rejected. *Building automation system* is an established industry term and is this repository's own canonical meaning of BAS.

**Adopt BASac now as the name of the operation-aware authorization model.** Rejected. It would create a synonym for an existing canonical term, which [`terminology-rules.md`](../standards/terminology-rules.md) prohibits. It would also imply a normative model that does not exist.

**Declare Basitra the successor to the Basis Foundation in this ADR.** Rejected for this ADR. It is an organizational decision outside architecture terminology, and it affects the GitHub organization and legal questions (such as the open license-attribution issue in [`GOVERNANCE.md`](../../GOVERNANCE.md#license)). Basitra is the intended long-term identity (Decision 1), but the form of the transition is recorded as open decisions OD-1 and OD-7.

## Consequences

- Current canonical terminology did not change while this ADR was `Proposed`. Now that it is `Accepted`, the repository's canonical reference documents change only through the bounded follow-on PRs listed in Decision 8.
- Final repository, package, and organization naming is gated on the strategy document's Stage 4 reconciliation and on OD-1 and OD-7 through OD-10. Some outcomes may require superseding accepted ADRs that fix names. For example, [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) establishes `basis-producer` as the permanent repository for the operation-producer runtime. Its component boundary is unaffected by this ADR, but a change to its name would need a superseding ADR.
- Historical documents keep their original terminology permanently. Readers will see both BASIS expansions in the repository, and the dated register in strategy document §3 is the reference for which applies where.
- Contributors must apply the BAS disambiguation convention. Reviewers should reject bare "BAS" used for Boundary-Aware Security in documents that also discuss building automation systems.
- Claims made with the term "Boundary-Aware Security" are subject to the same epistemic-restraint review as all other content ([`writing-guidelines.md`](../standards/writing-guidelines.md) §1.3 and §4.3). The status mapping in strategy document §6 is the reference.
- Ipotio's informal use of "Basitra" for BASIS components becomes a cross-ecosystem reconciliation item (strategy document C-6, OD-3). It is resolved in Ipotio's repository, not here.
- The community-driven objective and principles in strategy document §11 are recorded as strategy, not decided by this ADR. They restate existing governance positions, such as kernel boundary protection and role-versus-implementation substitutability. Adopting them as governance text is open decision OD-6.

## References

- [`docs/architecture/basitra-ecosystem-and-boundary-aware-security.md`](../architecture/basitra-ecosystem-and-boundary-aware-security.md): strategy document
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md)
- [`docs/standards/terminology-rules.md`](../standards/terminology-rules.md)
- [`docs/standards/writing-guidelines.md`](../standards/writing-guidelines.md)
- [`docs/glossary.md`](../glossary.md)
- [`GOVERNANCE.md`](../../GOVERNANCE.md)
- [ADR-0008](0008-producer-workload-authentication-and-admission.md), [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md), [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](0012-authorization-to-execution-binding.md), [ADR-0014](0014-minimum-execution-evidence-semantics.md), [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md)
- White paper [Section 05 — OT Trust Boundaries](../../whitepapers/identity-aware-authorization-for-operational-technology/sections/05-ot-trust-boundaries.md)
