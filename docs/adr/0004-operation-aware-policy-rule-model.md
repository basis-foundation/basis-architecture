# ADR-0004: Define Operation-Aware Policy Bundle and Rule Model

## Status

Proposed

## Context

[ADR-0001](0001-operation-aware-ot-authorization.md) recorded the decision to evolve `basis-core` toward an operation-aware authorization model, and its companion document, [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md), named the policy-model capabilities a future operation-aware kernel would need — role-based rules, attribute-based rules, operation-aware rules, resource matching, site/zone constraints, device-class constraints, protocol/action constraints, time-window constraints, safety-mode constraints, deny-overrides or documented combining semantics, deterministic rule ordering, stable reason codes, policy bundle metadata, policy validation, schema versioning, and compatibility tests — while explicitly deferring the design of that model to later architecture work.

[ADR-0002](0002-operation-aware-evaluation-semantics.md) then defined deterministic evaluation semantics: default deny, `NOT_APPLICABLE`, deny precedence, missing context, unknown values, safe errors, and trace expectations. [ADR-0003](0003-operation-aware-trace-audit-evidence.md) defined the trace and audit evidence model those semantics produce. Both documents describe how the kernel reasons about a request and what evidence that reasoning must leave behind, but neither defines what the kernel reasons *against*: what a policy bundle is, what a rule inside it can represent, how bundles and rules combine into a single outcome, how they are validated, and what compatibility surfaces they create.

Leaving the policy model undefined any longer risks the same problem ADR-0002 and ADR-0003 were each written to avoid: `basis-schemas` would have to invent bundle and rule structure while also trying to formalize contracts, and `basis-core` v0.2.0 would have to make policy-shape decisions under implementation pressure, with less architectural scrutiny than a compatibility surface of this size warrants. The next required architecture surface is the policy model itself — policy bundles, rules, effects, conditions, scope, metadata, validation, reason codes, explanations, and compatibility — defined conceptually, without yet choosing a policy language or final schema.

## Decision

BASIS will define policy bundles as the conceptual unit of policy versioning, validation, compatibility, scope, and provenance. Rules inside bundles will be deterministic units of evaluation with explicit effects, match criteria, conditions, reason codes, and traceable metadata.

The conceptual model for this decision is defined in [`docs/architecture/operation-aware-policy-rule-model.md`](../architecture/operation-aware-policy-rule-model.md). That document defines the policy bundle concept and its metadata categories; policy scope and why it matters for `NOT_APPLICABLE` semantics; the rule concept and stable rule identifiers; rule effects (`ALLOW`/`DENY`) and their relationship to deny precedence; match criteria mirrored from the `DecisionRequest` categories in ADR-0001's companion document; conditions and their determinism requirements; non-normative operation-aware policy examples; bundle and rule combining semantics restated from ADR-0002's evaluation semantics; rule ordering and priority; policy validation as a required pre-evaluation step; policy metadata and provenance; reason codes in policy; explanation text and templates; compatibility expectations; ownership boundaries; and security considerations. It is the companion to this ADR, in the same relationship the authorization model, evaluation semantics, and trace/audit evidence documents have with ADR-0001, ADR-0002, and ADR-0003.

This ADR records the decision to define this policy model now, ahead of `basis-schemas` and `basis-core` v0.2.0. It does not itself change `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`.

## Consequences

**Positive:**

- `basis-schemas` has a stable target for policy contracts.
- `basis-core` v0.2.0 has a clearer evaluation target.
- Trace/audit evidence can reference stable bundle/rule identifiers.
- Policy compatibility testing becomes possible.
- Console explanations can be derived from policy metadata rather than invented.

**Tradeoffs:**

- Larger compatibility surface — bundles, rules, reason codes, and explanation templates are now compatibility-governed artifacts, not informal configuration.
- More metadata to govern — bundle and rule metadata (identifiers, versions, scope, provenance, deprecation) must be maintained consistently across every future bundle.
- Reason-code governance becomes important, in the same way [ADR-0003](0003-operation-aware-trace-audit-evidence.md) already made reason codes a governed surface for trace and audit.
- Policy validation becomes a first-class requirement — a bundle cannot be evaluated without passing validation first, which is additional design and implementation work later phases must carry.
- Policy syntax remains deferred — this ADR intentionally leaves the actual policy language undecided, which means `basis-schemas` still has significant design work ahead of it before `basis-core` v0.2.0 can implement evaluation against real bundles.

## Alternatives Considered

**Define policy syntax immediately.** Rejected. Choosing a policy language (Rego, Cedar, CEL, YAML-only, Python, or a custom DSL) before the conceptual model of bundles, rules, effects, and combining semantics has stabilized risks encoding an unreviewed structure into a compatibility surface that is expensive to change afterward — the same reasoning ADR-0001 already applied to deferring `basis-schemas` work until the authorization model stabilized.

**Let `basis-core` define policy shape during implementation.** Rejected. This risks implementation-first semantics and weakens architecture governance — a policy model designed under implementation pressure is more likely to encode incidental parser or runtime behavior into a surface that other components and future compatibility tests then depend on.

**Let `basis-gateway` own policy semantics.** Rejected. `basis-gateway` enforces decisions and loads/selects/configures policy bundles as runtime input, but it does not evaluate policy and should not define what a bundle or rule means. Assigning policy semantics to the gateway would blur the same evaluation/enforcement boundary [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) already protects.

**Treat policies as unstructured configuration.** Rejected. Traceability, compatibility, validation, and auditability all require stable structure — an unstructured policy blob cannot be validated deterministically, cannot produce stable rule identifiers for trace and audit, and cannot support the compatibility testing ADR-0001 already named as a required policy-model capability.

## Non-goals

This ADR does not:

- Define final policy syntax
- Define final JSON Schema
- Implement policy evaluation
- Implement policy loading/distribution/storage
- Define policy authoring UI
- Define policy approval workflow
- Finalize reason-code vocabulary
- Change any implementation repository
- Choose a policy language
- Define policy signing/tamper-evidence

## References

- [`docs/architecture/operation-aware-policy-rule-model.md`](../architecture/operation-aware-policy-rule-model.md) — the conceptual policy bundle and rule model this ADR adopts
- [ADR-0001](0001-operation-aware-ot-authorization.md) — the decision that named the policy-model capabilities this ADR resolves
- [ADR-0002](0002-operation-aware-evaluation-semantics.md) — the evaluation semantics this document's combining rules restate at the policy-model level
- [ADR-0003](0003-operation-aware-trace-audit-evidence.md) — the trace and audit evidence model rule-level evidence and reason codes connect to
- [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md) — Section 5's policy model direction, the starting point for this document
- [`docs/architecture/operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md) — the deterministic evaluation semantics this document's combining and ordering sections build on
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) — the evidence model rule identifiers, reason codes, and explanations must remain consistent with
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules this policy model preserves
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction
- [`docs/architecture/compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) — the compatibility commitments the compatibility expectations section builds on
- [`docs/architecture/action-vocabulary.md`](../architecture/action-vocabulary.md) — the governance model reason codes and rule identifiers are analogous to
- [`docs/glossary.md`](../glossary.md) — terminology used throughout this ADR and its companion document
