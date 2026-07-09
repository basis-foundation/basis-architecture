# ADR-0002: Define Operation-Aware Evaluation Semantics

## Status

Proposed

## Context

[ADR-0001](0001-operation-aware-ot-authorization.md) recorded the decision to evolve `basis-core` toward an operation-aware authorization model — a kernel that evaluates richer, structured OT operation context (subject attributes, resource type, site/zone, device identity and class, protocol operation evidence, operation intent, safety/risk/environment/time context, and correlation metadata) while remaining protocol-agnostic, identity-provider-agnostic, and enforcement-agnostic. The companion document, [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md), defined the conceptual categories that richer request and response needs to carry, and it explicitly deferred a list of evaluation-semantics questions to later architecture work: default deny, `NOT_APPLICABLE` behavior, deny precedence, conflict resolution, rule ordering, condition evaluation, missing context behavior, unknown action/resource behavior, schema version compatibility, deterministic trace output, and safe error handling.

Those questions are not incidental details that can be resolved during implementation. A richer `DecisionRequest` without stable evaluation semantics would let two deployments — or two versions of the same kernel — reach different outcomes for the same request, and no amount of schema precision would correct that, because the ambiguity lives in the evaluation logic, not the data shape. In an OT authorization context, where a wrong `ALLOW` can mean an unauthorized control-affecting operation, that ambiguity is not acceptable to leave open until `basis-schemas` or `basis-core` v0.2.0 implementation work is already underway.

The evaluation-semantics questions also interact with each other in ways that are easier to reason about together than piecemeal: default deny and `NOT_APPLICABLE` need to be distinguished from each other and from evaluation failure; deny precedence and conflict resolution both govern how multiple matching rules combine into one outcome; missing context behavior and unknown action/resource behavior both govern what the kernel does when the input is incomplete or unrecognized. Addressing them as a single companion document to ADR-0001, rather than as scattered ADRs decided independently as each becomes urgent, keeps the semantics internally consistent.

## Decision

BASIS will define deterministic operation-aware evaluation semantics before publishing the richer `DecisionRequest`, `DecisionResponse`, policy, and audit contracts.

The conceptual semantics for this decision are defined in [`docs/architecture/operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md). That document defines default deny, `NOT_APPLICABLE` semantics, deny precedence, conflict resolution, rule ordering, condition evaluation, missing context behavior, unknown action/resource/type behavior, schema version compatibility, deterministic trace output, and safe error handling as conceptual semantics, not final schemas or a policy language. It is the companion to this ADR and should be read alongside it, in the same relationship [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md) has with ADR-0001.

This ADR records the decision to define these semantics now, ahead of `basis-schemas` and `basis-core` v0.2.0. It does not itself change `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`.

## Alternatives Considered

**Leave evaluation semantics to be decided during `basis-schemas` or `basis-core` v0.2.0 implementation.** Rejected. Deciding default-deny behavior, deny precedence, or missing-context handling as implementation details would mean those decisions get made under implementation pressure, without the same architectural review this repository requires for compatibility-surface and kernel-boundary decisions (see [`docs/adr/README.md`](README.md#when-an-adr-is-required)). It would also risk `basis-schemas` encoding a specific implementation's incidental behavior into a published contract rather than a reviewed semantic model.

**Design a complete policy language and evaluation engine specification now, rather than semantics alone.** Rejected. A full policy language is a larger and more implementation-adjacent undertaking than this document's scope, and committing to operator syntax now would foreclose policy-format design choices that belong to `basis-schemas` and `basis-core` v0.2.0. This ADR deliberately defines semantics — the constraints any future policy language must satisfy — without designing that language.

**Resolve each deferred topic (default deny, conflict resolution, missing context, and so on) as its own separate ADR.** Considered, but rejected as a decomposition choice. These topics are interdependent enough — default deny versus `NOT_APPLICABLE` versus evaluation failure, in particular — that reviewing them independently risks inconsistent boundaries between categories that need to compose cleanly. A single companion document, paired with one ADR, keeps the semantics coherent as a set.

## Consequences

**Positive:**

- A clearer target for `basis-schemas` — the categories in [`docs/architecture/operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md) give the schema repository concrete semantic requirements to formalize rather than an open design problem.
- A safer future `basis-core` v0.2.0 implementation — default deny, deny precedence, and safe error handling are settled before evaluation code is written against them.
- Better compatibility testing — schema version compatibility expectations (Section 12 of the companion document) give future compatibility tests a defined target rather than an implicit one.
- Less semantic drift across gateway, adapters, and core — distinguishing `NOT_APPLICABLE` from `DENY` from evaluation failure, and stating which component is responsible for what in each case, reduces the chance that different components interpret the same decision outcome differently.

**Tradeoffs:**

- More architectural work before implementation — this ADR adds a second companion document to review and stabilize before `basis-schemas` can proceed with confidence.
- More decisions to stabilize before schemas — every semantic category this document defines becomes something `basis-schemas` must account for, rather than something left flexible.
- Stricter constraints on future policy design — deny precedence and default deny, in particular, are stated as unconditional, which limits the design space for any future policy language that might otherwise want to make them configurable.

## Non-Goals

This ADR does not:

- define final schemas
- implement policy evaluation
- define policy syntax
- change any implementation repository
- define gateway HTTP behavior
- define adapter behavior
- define console behavior

## References

- [`docs/architecture/operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md) — the conceptual semantics this ADR adopts
- [ADR-0001](0001-operation-aware-ot-authorization.md) — the decision this ADR builds on
- [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md) — the conceptual model whose evaluation semantics this ADR defines
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules these semantics preserve
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction
- [`docs/architecture/compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) — the compatibility commitments the schema version compatibility semantics build on
- [`docs/architecture/action-vocabulary.md`](../architecture/action-vocabulary.md) — the canonical action vocabulary referenced in the unknown-action-format semantics
- [`docs/glossary.md`](../glossary.md) — terminology used throughout this ADR and its companion document
