# Operation-Aware Evaluation Semantics

This document defines the evaluation semantics for the operation-aware authorization model described in [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md) and adopted in [ADR-0001](../adr/0001-operation-aware-ot-authorization.md). That prior work defined the conceptual categories a richer `DecisionRequest`/`DecisionResponse` needs to carry and explicitly deferred the question of how the kernel reasons about that richer context to a later architecture PR. This is that PR.

This is an architecture document, not an implementation specification. It defines conceptual evaluation semantics — decision principles, precedence rules, and behavior categories — not Python types, JSON Schema, or a policy language. The corresponding decision to define these semantics before publishing richer contracts is recorded in [ADR-0002](../adr/0002-operation-aware-evaluation-semantics.md). Cross-references: [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md), [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md), [`docs/architecture/action-vocabulary.md`](action-vocabulary.md), [`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Purpose

Operation-aware authorization requires more than defining richer request fields. A `DecisionRequest` that can carry subject attributes, resource type, site/zone context, device identity and class, protocol operation evidence, operation intent, and safety/risk/environment/time context (per [Section 3 of the authorization model](operation-aware-authorization-model.md#3-rich-decisionrequest-conceptual-fields)) is only useful if there are deterministic rules for how those fields influence the outcome. Without those rules, two deployments — or two versions of the same kernel — could evaluate the same request differently, and no amount of schema precision would fix that, because the ambiguity is in the evaluation logic, not the data shape.

This document addresses the topics that [Section 6 of the authorization model](operation-aware-authorization-model.md#6-evaluation-semantics-to-define-in-later-prs) named and deliberately deferred: default deny, `NOT_APPLICABLE` behavior, deny precedence, conflict resolution, rule ordering, condition evaluation, missing context behavior, unknown action/resource behavior, schema version compatibility, deterministic trace output, and safe error handling. It addresses them conceptually, so that future `basis-schemas` and `basis-core` work has a stable semantic target rather than an open design question.

The boundary this document operates within is unchanged from the authorization model and from [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md):

- The kernel evaluates.
- The gateway enforces.
- Adapters normalize.
- Identity establishes identity context.
- Schemas publish contracts after semantics stabilize.

---

## 2. Evaluation Inputs

Evaluation, conceptually, takes the following inputs:

- **Operation-aware `DecisionRequest`** — the structured, normalized request described in [Section 3 of the authorization model](operation-aware-authorization-model.md#3-rich-decisionrequest-conceptual-fields), assembled by `basis-gateway` before it reaches the kernel.
- **Policy bundle** — the set of policy rules the request is evaluated against, however the eventual policy format represents that set.
- **Schema/version metadata** — the version identifiers attached to the request and to the policy bundle, needed to determine compatibility before evaluation proceeds.
- **Request context** — the categories described in Section 3 of the authorization model: site/zone, device identity and class, protocol operation evidence, operation intent, safety/risk/environment/time context, and correlation metadata.
- **Policy metadata** — versioning and provenance information carried with the policy bundle, per [Section 5 of the authorization model](operation-aware-authorization-model.md#5-policy-model-direction).
- **Rule identifiers** — the identifiers needed to reference which specific rules matched or were skipped, for trace and audit purposes.
- **Evaluation configuration, if applicable** — any deployment-level evaluation parameters a future policy format might define (for example, a default-outcome override for a specific policy scope). This document does not assume such configuration exists; it only notes that if it does, it must be resolved deterministically alongside everything else in this document.

`basis-core` receives all of these as structured input. It does not fetch identity, protocol, topology, risk, or safety context from external systems. That context arrives fully assembled through the gateway, exactly as [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership) already establishes. Nothing in this document changes that: evaluation semantics describe how the kernel reasons about the context it is given, not how it might go get more of it.

---

## 3. Evaluation Phases

The following is a conceptual evaluation sequence, not an implementation algorithm. It describes the order in which evaluation concerns must be resolved so that later phases can assume earlier ones are already settled — it does not prescribe function boundaries, data structures, or control flow.

```text
1. Validate request shape and schema version
2. Validate policy bundle shape and version
3. Establish evaluation context
4. Identify applicable candidate rules
5. Evaluate rule conditions deterministically
6. Combine matching rule outcomes
7. Produce final decision outcome
8. Produce reason codes and trace metadata
9. Return response for gateway enforcement
```

Two properties of this sequence matter more than its specific steps. First, validation (steps 1–2) happens before any policy logic runs — a request or policy bundle that fails validation never reaches rule evaluation, which is what makes the evaluation-failure category in [Section 14](#14-safe-error-handling) distinct from a substantive decision. Second, the sequence is one-directional: later steps do not reach back into earlier ones (rule evaluation, for instance, does not re-validate the request). This keeps the sequence something a reader can reason about statically, which is the point of stating it here rather than leaving it implicit in whatever the eventual implementation happens to do.

---

## 4. Default Deny

If a request is valid but no policy rule grants access, the final result is `DENY`.

This is the safe baseline for any valid authorization request. It applies regardless of how much or how little of the richer operation-aware context in Section 2 the request carries: a valid request that matches no allow rule is denied, not passed through, not left `NOT_APPLICABLE`, and not treated as an error.

Default deny is distinct from two other conditions this document also defines:

- **`NOT_APPLICABLE`** ([Section 5](#5-not_applicable-semantics)) — the policy bundle, rule set, or enforcement scope does not apply to this request at all. Default deny presumes a policy bundle that does apply but contains no matching allow rule; `NOT_APPLICABLE` presumes no applicable bundle or scope in the first place.
- **Evaluation failure** ([Section 14](#14-safe-error-handling)) — the request or policy bundle could not be evaluated at all, due to invalid shape, unsupported schema version, or an internal evaluation error. Default deny presumes evaluation completed; evaluation failure means it did not.

Keeping these three outcomes distinct is what lets a policy author, an auditor, or an operator reconstruct which of "nothing granted access," "this didn't apply," and "evaluation couldn't complete" actually happened.

---

## 5. NOT_APPLICABLE Semantics

`NOT_APPLICABLE` should mean: the policy bundle, rule set, or enforcement scope is not applicable to this request.

It should not mean: the subject lacks permission. That is `DENY`.

This distinction matters because the two conditions have different causes and imply different remediation. A `DENY` outcome means policy was evaluated and did not grant access — the remediation, if any, is a policy change. A `NOT_APPLICABLE` outcome means no policy bundle covers the request's domain, site, or scope at all — the remediation, if any, is closing a coverage gap, which is a materially different operational signal than "this subject was refused."

Representative examples:

- No policy bundle applies to the requested domain, site, or scope → `NOT_APPLICABLE`.
- A policy bundle applies, but no allow rule matches → `DENY` (default deny, per [Section 4](#4-default-deny)).
- A deny rule matches → `DENY` (deny precedence, per [Section 6](#6-deny-precedence)).
- The request cannot be evaluated because required input is invalid → evaluation failure, not `NOT_APPLICABLE` (per [Section 14](#14-safe-error-handling)).

Why this matters: gateways often collapse `NOT_APPLICABLE` into deny or fail-closed behavior at the enforcement boundary, and that collapse is a reasonable operational choice for `basis-gateway` to make. But the kernel itself should not perform that collapse internally. Preserving the distinction in the kernel's output means the gateway's fail-closed mapping is a visible, documented enforcement decision rather than something baked silently into evaluation, and it means audit and trace records retain the information needed to tell a coverage gap apart from a refusal after the fact.

---

## 6. Deny Precedence

If any applicable deny rule matches, the final decision is `DENY`, even if one or more allow rules also match.

Deny precedence is unconditional within this model: an allow rule cannot override a matching deny rule, regardless of specificity, priority, or ordering. This supports safety and least privilege in OT environments, where the cost of an incorrectly permitted control-affecting operation is generally much higher than the cost of an incorrectly refused one. A model that allowed a sufficiently specific or high-priority allow rule to override a deny rule would make the presence of a deny rule advisory rather than binding, which undermines the reason deny rules exist in an OT authorization context in the first place.

Deny precedence composes with default deny: the two together mean that access is granted only when at least one allow rule matches and no deny rule matches. Nothing else yields `ALLOW`.

---

## 7. Conflict Resolution

Conflict resolution in this model is intentionally narrow. It governs how multiple matching rules combine into a single outcome — it does not design a full policy language, and future policy-language work should not read this section as having settled anything beyond what is stated here.

- Explicit deny beats allow. This restates [Section 6](#6-deny-precedence) as the governing conflict rule.
- Explicit allow only succeeds when no deny matches.
- Multiple allows are acceptable if they converge on the same final outcome. The model does not require a single, uniquely identified "winning" allow rule — it requires that the set of matching allow rules, in the absence of a matching deny, produce `ALLOW`.
- Conflicting obligations or enforcement hints must not create enforcement behavior inside the kernel. If two matching rules attach different obligations or hints, resolving that conflict is not the kernel's responsibility to act on — the kernel surfaces what matched; it does not adjudicate between competing hints as though they were commands. This is consistent with the caution on obligations and enforcement hints in [Section 4 of the authorization model](operation-aware-authorization-model.md#4-rich-decisionresponse-conceptual-fields).
- Ambiguous rule conflicts — cases the policy format cannot resolve through deny precedence alone — must be handled deterministically and surfaced in trace/reason metadata, not silently resolved by incidental evaluation order. What "handled deterministically" means in a specific case is a policy-format design question left to later work; what this document commits to is that no such case may be resolved by accident.

---

## 8. Rule Ordering

Policy evaluation must not depend on incidental file order, dictionary order, runtime iteration order, or interpreter behavior. A rule set that produces a different outcome depending on how it happened to be loaded, iterated, or serialized is not deterministic in the sense this document and [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) require.

If a future policy format supports rule priority, the architecture requires a stable ordering model — for example, one built from:

```text
priority
then rule identifier
then stable declaration order only if explicitly defined by the policy format
```

This document does not commit to those as final schema fields. It commits to the requirement that whatever ordering model the policy format eventually defines must be stable and fully specified, not left to depend on the incidental behavior of a particular loader or runtime.

---

## 9. Condition Evaluation

Conditions must evaluate deterministically. A condition evaluated twice against the same request and the same policy bundle version must produce the same result every time.

A condition should produce one of three outcomes:

- **Match**
- **No match**
- **Evaluation error**

Two behaviors are required to keep this predictable for policy authors:

- Missing context should not be silently treated as truthy. A condition that references a context field the request does not carry does not default to matching; it is handled per the missing-context behavior in [Section 10](#10-missing-context-behavior).
- Type mismatches should not be silently coerced. A condition comparing incompatible types should not be quietly forced into a comparable form — it should be treated as an evaluation error or a defined no-match, not as an implicit coercion that changes meaning depending on the values involved.

The underlying goal is that a policy author should be able to predict, from the condition and the categories of context a request may or may not carry, whether a given rule will apply. This document does not design every operator a future condition language might support — it defines the semantics those operators must respect, not their syntax.

---

## 10. Missing Context Behavior

Missing context behavior is one of the more consequential distinctions in this document, because OT authorization will often involve partial context — during early integration, during degraded adapter connectivity, or when a given site has not yet onboarded a particular context category. The kernel's behavior in each case must be explicit rather than incidental:

- **Missing optional context used by a rule condition** — that condition does not match, unless the policy explicitly defines another behavior. A rule conditioning on device class, for example, simply does not match a request that carries no device class, rather than erroring or defaulting to a match.
- **Missing required request context needed for schema validity** — invalid request, and therefore an evaluation failure (per [Section 14](#14-safe-error-handling)), not a substantive decision.
- **Missing context required by an applicable policy bundle** — deterministic denial or evaluation failure, depending on whether the policy marks the context as required. If the policy bundle declares a context category as required for a scope it governs, and the request omits it, the outcome must be one of those two — never a silent fallback to some other behavior.

This three-way distinction exists so that "the request didn't have this field" cannot quietly turn into "the field didn't matter" or "the field was assumed present." Both would undermine the auditability this model exists to support.

---

## 11. Unknown Action / Resource / Type Behavior

- **Unknown action format** — invalid request, and therefore an evaluation failure, not a substantive decision. An action that does not conform to the canonical vocabulary in [`docs/architecture/action-vocabulary.md`](action-vocabulary.md) is a shape problem, not a policy-coverage problem.
- **Known action format but no policy coverage** — `DENY` or `NOT_APPLICABLE`, depending on policy scope, per the distinction in [Section 5](#5-not_applicable-semantics).
- **Unknown resource type with applicable policy scope** — `DENY` unless explicitly allowed. An unrecognized resource type inside a scope that policy does govern is treated the same way missing coverage for a known type would be: it does not get access by default.
- **Unknown protocol evidence** — should not make the kernel protocol-aware. The kernel evaluates only structured, normalized fields; unrecognized protocol-level evidence is not something the kernel interprets or special-cases, consistent with the protocol-agnostic boundary restated in [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership).

Fail-safe behavior matters more than convenience throughout this section. An unrecognized value should never resolve to `ALLOW` by default under any of these conditions.

---

## 12. Schema Version Compatibility

- v0.1-era request behavior must remain stable. Requests shaped according to the current subject/action/resource `DecisionRequest` must continue to evaluate to the same outcomes they do today as the model expands.
- Additive fields should not change old decisions unless policy explicitly uses the new fields. A policy bundle that does not reference a new context category must not change its outcome for a request that happens to carry that category.
- Unknown future fields should not silently alter evaluation. A kernel version evaluating a request that carries fields introduced after that kernel version was built should ignore those fields for evaluation purposes rather than fail unpredictably or, worse, alter the outcome based on data it cannot interpret.
- Version incompatibility should produce a deterministic failure mode — a defined evaluation failure, not undefined behavior — when a request or policy bundle declares a schema version the kernel cannot support.
- Compatibility tests should protect expected outcomes across version increments, consistent with the compatibility testing expectations already established in [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md).

---

## 13. Deterministic Trace Output

This section defines high-level trace expectations. It does not design the full trace schema — that is deferred to a later audit/trace architecture PR, consistent with [Section 7 of the authorization model](operation-aware-authorization-model.md#7-audit-and-trace-direction).

Trace output should be:

- **Deterministic** — the same request evaluated against the same policy bundle version produces the same trace, not merely the same decision outcome.
- **Stable enough for tests** — trace structure and content should be reliable enough to serve as the basis for compatibility tests, not something that changes shape incidentally between runs or kernel builds.
- **Safe to expose in audit and operator views after redaction** — trace content is intended to eventually be visible to auditors and, through `basis-console`, to operators, subject to whatever redaction rules the eventual audit schema defines.
- **Clear about matched rules, skipped rules, missing context, and reason codes** — a trace should be able to answer not only "what was the outcome" but "which rules were considered, which of those matched, which were skipped and why, and what context, if any, was missing."
- **Free of secrets** — trace output must never carry credentials, tokens, or key material, consistent with the no-secrets-in-audit principle already established for `basis-identity` and restated in [Section 7 of the authorization model](operation-aware-authorization-model.md#7-audit-and-trace-direction).

---

## 14. Safe Error Handling

Evaluation failure is a distinct outcome category from `ALLOW`, `DENY`, and `NOT_APPLICABLE`. Representative evaluation failure categories:

- **Invalid request** — the `DecisionRequest` does not conform to the required shape for its declared schema version.
- **Unsupported schema version** — the request or policy bundle declares a schema version the kernel does not support.
- **Invalid policy bundle** — the policy bundle does not conform to the required shape.
- **Policy validation failure** — the policy bundle is shaped correctly but fails internal consistency validation.
- **Condition evaluation error** — a rule condition could not be evaluated (per [Section 9](#9-condition-evaluation)), as distinct from evaluating to no-match.
- **Internal evaluation error** — an unexpected failure within the evaluation process itself, not attributable to the request or policy bundle.

Evaluation errors are not authorization allows. A kernel that cannot produce a valid decision must not default to `ALLOW` under any of the categories above, and it must not disguise an evaluation failure as a `DENY` or `NOT_APPLICABLE` decision, because that would misrepresent an evaluation problem as a policy outcome. Gateways should fail closed when the kernel cannot produce a valid allow decision — this is a `basis-gateway` enforcement responsibility, not something this document assigns to the kernel, but it depends on the kernel surfacing evaluation failures as a distinct, recognizable category rather than collapsing them into a decision outcome.

---

## 15. Boundary Preservation

These semantics do not move responsibilities. They are restated here explicitly, in the same way [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership) restates them, because evaluation semantics touch the same territory that boundary ownership governs and it would be easy to blur the two while defining them.

- **`basis-core`** evaluates policy only.
- **`basis-gateway`** enforces decisions and handles runtime failure behavior.
- **`basis-adapters`** normalize protocol evidence and never authorize.
- **`basis-identity`** establishes identity context.
- **`basis-schemas`** will later publish the concrete contracts these semantics are written against.
- **`basis-console`** may visualize and explain semantics but does not evaluate.

None of these assignments are new. They are the same assignments in [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md) and [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), restated against the evaluation semantics this document defines so that defining how the kernel reasons about richer context does not quietly redraw who reasons about it.

---

## 16. Open Questions Deferred

This document does not settle:

- the final policy language
- the final JSON Schema
- the final trace schema
- the final reason-code vocabulary
- policy authoring UI
- gateway runtime mapping of errors to HTTP responses
- adapter evidence schema
- safety certification or safety-system behavior

Naming these here is intentional, for the same reason [Section 6 of the authorization model](operation-aware-authorization-model.md#6-evaluation-semantics-to-define-in-later-prs) named the topics this document addresses: it establishes that this PR is aware of what remains open and is not silently deciding those questions by omission. Each should be addressed in its own later architecture work, informed by the semantics this document defines.
