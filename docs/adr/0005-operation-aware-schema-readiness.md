# ADR-0005: Define Operation-Aware Schema Readiness and Migration Plan

## Status

Proposed

## Context

[ADR-0001](0001-operation-aware-ot-authorization.md) through [ADR-0004](0004-operation-aware-policy-rule-model.md) now define the operation-aware model, its evaluation semantics, its trace and audit evidence categories, and its policy bundle and rule model. Each of those documents named the schema-level implications of its own decisions and deferred the actual schema work — ADR-0001 deferred `basis-schemas` entirely pending an architectural model to formalize; ADR-0004 deferred policy syntax pending the conceptual model it defined. None of the four resolved how the accumulated architecture should actually move into `basis-schemas`: in what order, under what compatibility expectations, and with what preserved from each prior document.

`basis-schemas` already exists as a repository, with its own charter in [`docs/architecture/basis-schemas.md`](../architecture/basis-schemas.md) defining what belongs in it, what does not, and an initial migration order for its first six contracts (action vocabulary, action string, resource identifier, decision request, decision response, audit event). That charter predates the operation-aware architecture stack and does not sequence the richer operation-aware contracts ADR-0001 through ADR-0004 now describe. The next phase should move into `basis-schemas` as the operation-aware vNext contracts, but doing so without a handoff plan risks the same problem each prior ADR was written to avoid: `basis-schemas` inventing structure under implementation pressure, or `basis-core` v0.2.0 making shape decisions that should have been architectural ones. Schemas must not invent semantics or encode unresolved ambiguity; they must formalize what has already been decided.

## Decision

BASIS will treat the schema readiness plan defined in [`docs/architecture/operation-aware-schema-readiness-plan.md`](../architecture/operation-aware-schema-readiness-plan.md) as the architecture handoff into `basis-schemas` for the operation-aware contract surfaces. Operation-aware contracts — `OperationAwareDecisionRequest`, `OperationAwareDecisionResponse`, `PolicyBundle`, `PolicyRule`, `PolicyCondition`, `EvaluationTrace`, `TraceRuleEvidence`, `AuditEvidence`, `GatewayAuditEvent`, `AdapterEvidenceReference`, `IdentityEvidenceReference`, `ReasonCode`, `RedactionClassification`, `SchemaVersionMetadata`, and `CompatibilityExample`/`TestVector` — should be published in the ordered sequence of PRs A through G that document defines, preserving the semantics, boundaries, and security requirements established in ADR-0001 through ADR-0004.

The companion document is [`docs/architecture/operation-aware-schema-readiness-plan.md`](../architecture/operation-aware-schema-readiness-plan.md). It defines what each prior ADR contributes to schema readiness; what schema readiness means conceptually; the contract surfaces to publish; the recommended publication order and dependency relationships between PRs; compatibility rules; required examples and test vectors; an ownership matrix; boundary rules for schema design; redaction and security requirements; the migration path from architecture through schemas to core, gateway, adapters, and console; what must not move to schemas yet; success criteria for entering `basis-schemas` work; and open questions intentionally left deferred. It is the companion to this ADR, in the same relationship the authorization model, evaluation semantics, trace/audit evidence, and policy/rule model documents have with ADR-0001 through ADR-0004.

This ADR records the decision to treat this readiness plan as the architecture handoff now, ahead of the next `basis-schemas` expansion. It does not itself publish schemas, define final JSON Schema, implement `basis-core` v0.2.0, or change any implementation repository.

## Consequences

**Positive:**

- `basis-schemas` gets a clear implementation roadmap for the operation-aware contract surfaces.
- Schema PRs can stay small and ordered, each independently reviewable against a defined scope.
- `basis-core` v0.2.0 gets stable contract targets to implement against.
- Compatibility examples and test vectors are planned before implementation begins, rather than retrofitted afterward.
- Redaction and security expectations are preserved as structural requirements on the contracts, not left to each publisher's discretion.

**Tradeoffs:**

- More architecture work before code — this is the fifth architecture PR in the operation-aware sequence before a single schema is published.
- Schema work may take several PRs (A through G) rather than a single release, which extends the calendar time before `basis-core` v0.2.0 can begin implementation against complete contracts.
- Some details remain deferred until schema design begins — this ADR does not resolve final field names, types, or serializations.
- Exact schema file layout and version numbers remain open, per [Section 16 of the readiness plan](../architecture/operation-aware-schema-readiness-plan.md#16-open-questions-deferred).

## Alternatives Considered

**Move directly to `basis-schemas` without a handoff plan.** Rejected. This risks schema-first invention and contract ambiguity — the same risk ADR-0001 avoided by deferring `basis-schemas` until the authorization model existed, and the same risk ADR-0004 avoided by deferring policy syntax until the policy/rule model existed. Four ADRs of architectural work should not be handed to schema authors as an undifferentiated pile of prior documents; it should be handed to them as a sequenced plan.

**Publish all operation-aware schemas in one large PR.** Rejected. A single PR spanning request, response, policy, trace, and audit contracts would have a review surface too large to evaluate carefully, and would make the dependency order between contracts (Section 6 of the readiness plan) hard to validate — a reviewer could not confirm that `PolicyBundle` was reviewed independently of the `DecisionResponse` contracts that reference concepts it introduces.

**Implement `basis-core` v0.2.0 before schema publication.** Rejected. Implementing against unpublished contracts would define those contracts implicitly, through whatever shape the kernel implementation happens to choose — the same implementation-first risk ADR-0004 already rejected when it declined to let `basis-core` define policy shape during implementation.

**Let each implementation repository define its own vNext contract.** Rejected. This would recreate the cross-repository drift the [`ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) reconciliation work already catalogued for the v0.1-era contracts — the exact problem `basis-schemas` exists to prevent, per [`docs/architecture/basis-schemas.md`](../architecture/basis-schemas.md#2-why-a-separate-repository).

## Non-goals

This ADR does not:

- Publish schemas
- Define final JSON Schema
- Define final policy language
- Implement `basis-core` v0.2.0
- Change any implementation repository
- Define exact schema package version numbers
- Define exact release cadence
- Define audit storage or deployment behavior

## References

- [`docs/architecture/operation-aware-schema-readiness-plan.md`](../architecture/operation-aware-schema-readiness-plan.md) — the schema readiness and migration plan this ADR adopts
- [ADR-0001](0001-operation-aware-ot-authorization.md) — the operation-aware authorization model this plan's `DecisionRequest`/`DecisionResponse` contract surfaces formalize
- [ADR-0002](0002-operation-aware-evaluation-semantics.md) — the evaluation semantics this plan's compatibility rules and response/trace contracts must not obscure
- [ADR-0003](0003-operation-aware-trace-audit-evidence.md) — the trace/audit/redaction/evidence categories this plan's evidence and audit contract surfaces formalize
- [ADR-0004](0004-operation-aware-policy-rule-model.md) — the policy bundle/rule model this plan's policy contract surfaces formalize
- [`docs/architecture/basis-schemas.md`](../architecture/basis-schemas.md) — the existing `basis-schemas` charter this plan sequences the next expansion against
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the inventory of contracts already formalized or in need of formalization
- [`docs/architecture/compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) — the compatibility commitments this plan's compatibility rules build on
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules this plan's boundary rules preserve
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction
- [`docs/glossary.md`](../glossary.md) — terminology used throughout this ADR and its companion document
