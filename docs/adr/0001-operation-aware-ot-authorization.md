# ADR-0001: Adopt Operation-Aware Authorization Model for OT Decisions

## Status

Proposed

## Context

`basis-core` v0.1.0 is intentionally small: it evaluates a `DecisionRequest` built from a subject, an action, and a resource, and returns a deterministic `DecisionResponse`. That scope was the right starting point — it let the kernel's evaluation semantics, enforcement contracts, and audit event definitions stabilize (see [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md)) without also having to solve OT-specific context modeling at the same time.

Real OT authorization decisions depend on more than a user's role and an action string. Representative examples:

- who is acting — not just their identifier, but the attributes and the identity authority mode (see [`docs/architecture/identity-authority-modes.md`](../architecture/identity-authority-modes.md)) that produced their identity
- what device or resource is affected, and what class of device it is
- which site or zone is involved
- what protocol operation is being attempted, and through which protocol
- whether the action is read-only or control-affecting, independent of the action verb itself
- whether safety or risk context should change the decision
- what audit evidence must be preserved to reconstruct why the decision was reached

A kernel limited to subject/action/resource cannot express policy that conditions on any of this. Either the additional context gets encoded informally somewhere outside the kernel — gateway middleware, adapter logic, operator judgment — where it is not deterministic or auditable, or it does not get enforced at all. Both outcomes are inconsistent with what an authorization kernel is for.

The next phase of `basis-core` needs to expand to accommodate this context. That expansion has to happen without compromising the invariants that make the kernel valuable in the first place: it must remain protocol-agnostic, identity-provider-agnostic, and enforcement-agnostic, and it must not become a second enforcement engine sitting behind `basis-gateway`.

## Decision

BASIS will evolve toward an operation-aware authorization model in which `basis-core` evaluates structured, normalized OT operation context — subject, subject attributes, resource, resource type, site/zone context, device identity and class, protocol operation evidence, operation intent, safety/risk/environment/time context, and correlation metadata — while remaining protocol-agnostic, identity-provider-agnostic, and enforcement-agnostic.

The conceptual model for this expansion is defined in [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md). That document defines categories of context, not final schemas; it is the companion to this ADR and should be read alongside it.

This ADR records the decision to pursue this direction. It does not itself change `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`.

## Alternatives Considered

**Keep `basis-core` limited to subject/action/resource indefinitely, and let `basis-gateway` encode operation-aware logic ad hoc.** Rejected. This was, in effect, the status quo this ADR is moving away from. It produces authorization logic that is not deterministic in the kernel's sense, not centrally auditable, and not portable across gateway implementations — it recreates exactly the fragmentation that centralizing evaluation in `basis-core` was meant to avoid.

**Push operation-aware context handling into `basis-adapters` instead of `basis-core`.** Rejected. Adapters normalize protocol operations; they do not make policy decisions, and giving an adapter enough context-awareness to influence authorization outcomes would blur that boundary. The invariant that adapters never authorize is not up for revision.

**Defer this decision until `basis-schemas` exists and let the schema repository decide the model.** Considered, but rejected as an ordering choice, not as a direction. `basis-schemas` needs an architectural model to formalize; designing the schema before the architecture is settled risks encoding an unreviewed model into a compatibility surface that is expensive to change afterward. The migration path in [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md#9-migration-path) puts architecture before `basis-schemas` for this reason.

## Consequences

**Positive:**

- Richer OT authorization decisions — policy can condition on the operational context that actually determines whether an action should be allowed in an OT environment.
- Better auditability — decisions can carry the evidence and trace needed to reconstruct why an outcome was reached.
- Better gateway/adapters/core separation — the model gives the gateway a defined way to assemble richer context without either the adapter or the gateway taking on evaluation responsibility.
- Clearer future schema ownership — the categories in the companion architecture document give `basis-schemas` a concrete model to formalize rather than an open-ended design problem.
- A stronger, more concrete path toward `basis-core` v0.2.0.

**Tradeoffs:**

- More request/response complexity — a wider `DecisionRequest`/`DecisionResponse` is harder to reason about than the current subject/action/resource shape.
- Stricter compatibility requirements — a richer contract has more surface area that must be versioned and protected, per [`docs/architecture/compatibility-philosophy.md`](../architecture/compatibility-philosophy.md).
- More schema governance — the ownership questions already tracked in [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) become more consequential as the contract they govern grows.
- More need for deterministic traces and redaction rules — a richer decision response is more useful but also carries more that must be handled correctly for audit safety.

## Non-Goals

This ADR does not:

- define final JSON Schemas for `DecisionRequest`, `DecisionResponse`, policy, or audit contracts
- implement `basis-core` v0.2.0
- design a complete policy language
- add behavior to `basis-gateway`
- add behavior to `basis-adapters`
- add behavior to `basis-console`
- introduce AI-driven or autonomous policy decisions
- make `basis-core` aware of OIDC, JWTs, sessions, cookies, or OT protocol libraries

## References

- [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md) — the conceptual model this ADR adopts
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules this decision preserves
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction
- [`docs/architecture/compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) — the compatibility commitments that govern how this expansion must be introduced
- [`docs/architecture/action-vocabulary.md`](../architecture/action-vocabulary.md) — the existing governance model for one of the compatibility surfaces this expansion touches
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the current state of cross-repository contracts this expansion builds on
- [`docs/architecture/identity-authority-modes.md`](../architecture/identity-authority-modes.md) — identity authority mode context referenced in the subject attributes category
- [`docs/glossary.md`](../glossary.md) — terminology used throughout this ADR and its companion document
