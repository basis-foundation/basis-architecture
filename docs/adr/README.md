# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for the BASIS ecosystem. ADRs document significant architectural choices: what was decided, what alternatives were considered, why the chosen approach was selected, and what consequences the decision carries.

For the governance rationale — why ADRs exist, when they are required, and how they relate to the basis-core boundary protection policy — see [`GOVERNANCE.md`](../../GOVERNANCE.md). This document covers the mechanics: how ADRs are numbered, named, structured, and maintained.

---

## What This Directory Contains

At this stage of the project, no ADRs have yet been formally recorded. This README establishes the process so that when ADRs are created, they are consistent and useful.

---

## Numbering

ADRs are numbered sequentially, starting at 0001. The number is assigned at the time the ADR is accepted — not when it is drafted. Draft ADRs in review carry a working title without a number until they are accepted.

The number is permanent. ADRs that are superseded retain their number; a new ADR that supersedes them gets a new number and references the superseded ADR by number.

Format: `NNNN-{short-title}.md` where `NNNN` is the zero-padded sequence number.

Examples:

- `0001-basis-core-isolation-boundary.md`
- `0002-audit-schema-version-strategy.md`
- `0003-action-vocabulary-naming-structure.md`

---

## Lifecycle States

Each ADR carries a status that reflects where it is in its lifecycle:

- **Draft** — under active development; not yet submitted for review
- **Proposed** — submitted for Foundation review; open for comment
- **Accepted** — accepted by Foundation review; represents a current architectural decision
- **Superseded** — replaced by a later ADR; the superseding ADR number is recorded in the status field
- **Rejected** — reviewed and not accepted; the rejection rationale is recorded in the ADR body

ADRs are never deleted. A rejected or superseded ADR remains in the directory as a record of what was considered and why it was not chosen or why it was replaced.

---

## Naming Conventions

ADR file names are lowercase, hyphen-separated, and describe the decision subject concisely:

- Prefer noun phrases that name the thing being decided: `basis-core-isolation-boundary`, `audit-schema-version-strategy`
- Avoid verb phrases that describe the outcome: not `isolate-basis-core`, not `define-audit-schema-versioning`
- Avoid generic titles: not `architecture-decision`, not `important-change`
- The name should be stable. Once an ADR is accepted, its file name does not change even if later understanding refines the framing.

---

## Supersession

When a later decision replaces an earlier one, the superseding ADR must:

1. Reference the superseded ADR number in its own status and context sections.
2. Explain what changed and why the earlier decision is no longer applicable.

The superseded ADR must be updated to:

1. Change its status to "Superseded."
2. Record the number of the superseding ADR in its status field.

The body of the superseded ADR is not edited beyond the status line. The historical record of what was decided and why — including the reasoning that is now outdated — is preserved as written.

---

## Expected ADR Sections

Each ADR should contain the following sections. Sections may be brief when the content is genuinely limited; they should not be padded to appear thorough.

**Status** — one of: Draft, Proposed, Accepted, Superseded (by ADR-NNNN), Rejected.

**Context** — the situation that made a decision necessary. What was the problem, constraint, or architectural question being addressed? What options existed? What information was available at the time? This section should be written so that a reader who was not present during the discussion can understand why a decision was needed.

**Decision** — what was decided, stated directly. This section should be specific enough that a reviewer can determine whether a proposed change is consistent with the decision. Vague or ambiguous decisions are not useful.

**Alternatives considered** — what other approaches were evaluated and why they were rejected. An ADR that does not record alternatives does not explain why the chosen approach was selected; it only records what was selected.

**Consequences** — what follows from the decision. What constraints does it impose on future decisions? What does it enable? What does it prevent? What implementation, governance, or operational work does it require?

**References** — links to relevant white paper sections, architecture principles, glossary terms, or other ADRs that provide context.

---

## When an ADR Is Required

An ADR is required when a decision:

- Establishes or changes a component boundary or dependency rule
- Changes the behavior of a compatibility surface (audit schema, action vocabulary, policy format, kernel evaluation semantics)
- Proposes adding a dependency to basis-core (always requires an ADR and Foundation review; see [`GOVERNANCE.md`](../../GOVERNANCE.md))
- Rejects an architectural approach that was seriously considered — the rejection should be recorded so it is not re-evaluated with incomplete information
- Changes the interpretation of an established architecture principle in a specific context
- Would reasonably prompt a future implementer to ask "why was this done this way?" without an obvious answer

An ADR is not required for:

- Corrections to documentation that do not change architectural intent
- Changes that are clearly consistent with an already-accepted ADR
- Implementation decisions within a component repository that do not affect cross-component contracts or kernel boundaries
- Operational and deployment decisions that do not change architectural semantics

The line between architecture and implementation is examined in [`docs/architecture/reference-vs-implementation.md`](../architecture/reference-vs-implementation.md). When it is unclear whether a decision requires an ADR, err toward creating one — an unnecessary ADR is a minor overhead; an absent ADR for a consequential decision creates technical debt in the form of undocumented reasoning.

---

## Architecture Principle vs. ADR vs. Implementation Detail

These three document types cover different kinds of knowledge and should not be confused.

**An architecture principle** (in [`docs/architecture-principles.md`](../architecture-principles.md)) states a general design commitment that applies across the full scope of the architecture. Principles are stable, broadly applicable, and do not reference specific components or decisions. "Policy Evaluation Independent of Protocol" is an architecture principle: it applies everywhere protocol adapters and policy engines interact, not to a specific decision at a specific time.

**An ADR** records a specific decision made at a specific point in time, for specific reasons, with specific consequences. ADRs are narrower than principles and more precise. "Use a three-part verb:domain:object naming structure for action names" is an ADR-level decision: it is specific, it has alternatives that were considered, and it has consequences that constrain future vocabulary additions.

**An implementation detail** is a decision within a component repository about how to realize an architectural requirement in a specific programming language, framework, or runtime environment. "Use Python dataclasses for DecisionRequest and DecisionResponse" is an implementation detail. It belongs in the basis-core repository, not in an ADR in basis-architecture. Implementation details that do not affect cross-component contracts or kernel boundaries do not belong in this directory.

When an implementation decision surfaces an architectural ambiguity — a case where the architecture documentation does not answer the design question — the correct response is to clarify the architecture documentation first (possibly through an ADR), then carry that clarification into the implementation. Implementation decisions should not silently resolve architectural ambiguities without documenting them here.

---

## Relationship to White Papers

White papers describe and analyze the architecture. ADRs record specific decisions within that architecture.

White paper content should not include ADR-style decision records. A white paper section that explains the tradeoffs between two approaches should not also record which approach was chosen and why — that belongs in an ADR that can be superseded if the decision changes. Mixing them makes white papers harder to maintain and makes it unclear which content reflects permanent architectural analysis and which reflects a specific decision that may be revisited.

When an ADR is accepted that changes architectural intent, the affected white paper sections should be reviewed for consistency. The white paper is updated to reflect the new architectural understanding; the ADR records why the change was made.

---

## Relationship to Implementation Repositories

ADRs in basis-architecture record architectural decisions — decisions that affect the design constraints, component contracts, and dependency rules that implementation repositories must satisfy. They do not record implementation decisions within those repositories.

Implementation repositories may maintain their own decision records for component-level decisions. Those records are local to the component. When a component-level decision has cross-component implications — when it affects a compatibility surface, a dependency boundary, or a contract that other components depend on — it must be elevated to an architectural ADR in basis-architecture before it is acted upon.
