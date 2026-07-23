# Operation-Aware Evidence Provenance Semantics

**Status:** Proposed. Architecture clarification, narrower than an ADR, opened to resolve three repeatable disagreements `basis-core`'s roadmap PR 37 ("Wire the five canonical scenarios end-to-end," `docs/implementation/basis-core-v0.2-operation-aware-plan.md`, Milestone 12, a document in the separate `basis-core` repository, not this one — referenced by path, not hyperlinked, per this repository's cross-repository citation convention) found between the merged `basis-core` implementation and the vendored `basis-schemas` v0.2.1 canonical compatibility fixtures. This document does not claim any of the three disagreements is already resolved in either repository. It settles the governing semantics; correcting the fixtures and correcting `basis-core`'s rule-evidence projection are separate, later pull requests in their own repositories, sequenced after this clarification per `basis-core`'s reconciliation runbook.

**Scope:** Three narrow evidence-provenance questions that ADR-0002, ADR-0003, and ADR-0004 (and their companion documents) named categories for but did not settle at the level of specificity PR 37's field-by-field equality check requires: (1) whether `basis-core` should synthesize aggregate, human-readable `explanation` text when no rule or governed stage supplies one; (2) exactly which per-rule `reason_code`/`explanation` fields a `TraceRuleEvidence` entry carries, keyed by that rule's match result; (3) whether policy bundle identity (`bundle_id`/`bundle_version`) is preserved on a response/trace/audit-evidence artifact when the evaluation outcome is `NOT_APPLICABLE` or a typed semantic policy-validation failure. It does not design a template language, does not finalize the reason-code vocabulary, does not change authorization outcomes, deny precedence, default deny, or `NOT_APPLICABLE` semantics, and does not add new schema fields.

**Companion documents:** [`docs/adr/0002-operation-aware-evaluation-semantics.md`](../adr/0002-operation-aware-evaluation-semantics.md) and [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md) (§4–§5, §14 — default deny, `NOT_APPLICABLE`, and safe error handling, the evaluation-state categories this document's provenance rules attach to), [`docs/adr/0003-operation-aware-trace-audit-evidence.md`](../adr/0003-operation-aware-trace-audit-evidence.md) and [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md) (§5, §11, §12 — rule evaluation evidence, human-readable explanation, and reason codes, the categories this document narrows), [`docs/adr/0004-operation-aware-policy-rule-model.md`](../adr/0004-operation-aware-policy-rule-model.md) and [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md) (§2–§3, §11, §14 — policy bundle identity, policy scope, policy validation, and explanation templates), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Purpose

`basis-core`'s canonical end-to-end conformance test (roadmap PR 37) is the first point in either repository's history where the merged operation-aware evaluation pipeline was compared, field by field, against the vendored `basis-schemas` v0.2.1 canonical compatibility fixtures. Running it surfaced three repeatable disagreements, present across all five canonical scenarios (`allow-basic`, `deny-precedence`, `default-deny`, `not-applicable`, `invalid-policy-bundle`) in at least one form:

1. Every fixture's top-level `explanation` (on the expected `OperationAwareDecisionResponse`, `EvaluationTrace`, and `AuditEvidence`) is a non-null, synthesized sentence; `basis-core`'s pipeline always produces `explanation: null` at that level, because no governed stage supplies one.
2. Each fixture's per-rule `TraceRuleEvidence.reason_code`/`.explanation` disagree with `basis-core`'s current unconditional-copy behavior in scenario-specific ways: a `not_matched` rule's authored text is copied through where the fixture expects it omitted; a non-decisive `matched` rule's authored text is copied through where one fixture expects it omitted; a decisive `matched` rule's authored text is copied through where the fixture expects a different, synthesized sentence.
3. `bundle_id`/`bundle_version` are populated by `basis-core` with the configured bundle's real identity for a `NOT_APPLICABLE` outcome and for a typed semantic policy-validation failure; the `not-applicable` and `invalid-policy-bundle` fixtures expect `null` in both cases.

None of ADR-0002 through ADR-0004 or their companion documents settle these three questions at this level of specificity. [Trace and audit evidence §5](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence) lists "reason code emitted by the rule" and "redacted explanation" as rule-evidence categories without stating whether a non-matching rule emits anything; [§11](operation-aware-trace-audit-evidence.md#11-human-readable-explanation) states that explanation "must not invent facts" and "must be derived from trace/evidence, not generated independently of it" without stating what a top-level explanation is when no such derivation exists; [Policy rule model §3](operation-aware-policy-rule-model.md#3-policy-scope) states that "a bundle outside its declared scope should not pretend to deny or allow a request" without stating whether the bundle's own identity remains known and reportable. This document closes exactly that gap — narrowly, and without redesigning any of the three governing documents' existing conclusions.

---

## 2. Top-Level Explanation Provenance

### Governing direction

```text
top-level explanation (OperationAwareDecisionResponse.explanation,
EvaluationTrace.explanation, AuditEvidence.explanation) is optional
and non-authoritative.

reason_code remains the authoritative machine-readable explanation of
a completed or failed evaluation.

basis-core does not synthesize aggregate, human-readable prose merely
to populate this field.

When no evaluation stage supplies a governed top-level explanation,
the value is null. Null is a valid, complete, expected value — not a
placeholder for a future feature, and not evidence of a defect.
```

This follows directly from [Trace and audit evidence §11](operation-aware-trace-audit-evidence.md#11-human-readable-explanation): "Explanation must not invent facts," "Explanation must be derived from trace/evidence, not generated independently of it." No governed stage in the current architecture — not policy-owned aggregation, not evaluation-owned orchestration, not audit-owned trace assembly — produces an aggregate sentence describing *why* an evaluation reached its outcome as a whole; each stage produces `reason_code` (a stable, machine-readable value) and, where a rule authored one, a per-rule `explanation` (§3 below). Synthesizing a top-level sentence from those parts, merely so that a field is non-null, would be exactly the "generated independently of it" case §11 already forbids — the synthesized sentence would not be a rendering of an evidence category this document or its companions define; it would be new prose invented at assembly time.

### What this does not decide

A future, separately governed explanation-generation feature remains possible. It would need its own reviewed contract defining: provenance (which evidence categories a generated sentence may draw from and how), template or generation semantics, determinism (the same evaluation must always generate the same sentence), localization implications, compatibility expectations (can generated text change without a version bump), and redaction requirements (a generated sentence must not leak evidence a structured field would otherwise redact). This document does not define that feature. It only settles that its absence today is not a defect.

### Fixture implication

A canonical fixture must not require `basis-core` to populate a top-level `explanation` from a synthesis mechanism this architecture does not define. Where a canonical fixture currently carries synthesized top-level prose not derivable from an existing evidence category, that fixture is the artifact that needs correction, not `basis-core`.

---

## 3. Rule-Evidence Projection

### Governing direction

A `TraceRuleEvidence` entry's `reason_code`/`explanation` are the rule's own authored fields, projected according to that rule's `rule_result` — never copied unconditionally, and never synthesized into different text:

| Rule result   | Authored `reason_code`     | Authored `explanation`     |
| ------------- | --------------------------: | --------------------------: |
| `matched`     | preserve verbatim            | preserve verbatim            |
| `not_matched` | omit (`null`)                 | omit (`null`)                 |
| `skipped`     | omit (`null`)                 | omit (`null`)                 |
| `error`       | governed error reason code, never the rule's authored success/deny reason code | governed error explanation, or `null` if none exists — never the rule's authored success/deny explanation |

Three clarifications follow directly from [Trace and audit evidence §5](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence) and [Policy rule model §14](operation-aware-policy-rule-model.md#14-explanation-text-and-templates):

- **A rule that did not match did not emit anything.** [§5](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence) lists "reason code emitted by the rule" as a rule-evidence category — a rule that did not match, or was never reached (`skipped`), emitted no reason code and authored no explanation for *this* evaluation; its evidence entry must not present its authored success/deny rationale as though it had.
- **A matched rule remains matched evidence even when another rule determines the final outcome.** Deny precedence ([Evaluation semantics §6](operation-aware-evaluation-semantics.md#6-deny-precedence)) governs the final authorization outcome, not whether an earlier matched rule actually matched. In a deny-precedence evaluation, a matched `ALLOW` rule and a matched `DENY` rule are both genuine matched evidence; "non-decisive" describes the matched rule's effect on the final outcome, not its evidentiary status. Both preserve their own authored `reason_code`/`explanation`.
- **Authored evidence is not rewritten into synthesized prose.** [Policy rule model §14](operation-aware-policy-rule-model.md#14-explanation-text-and-templates) requires explanations to be "derived from rule metadata, reason codes, and trace evidence" and states that "explanations must not invent facts." A matched rule's own authored `explanation` already satisfies that requirement directly — replacing it with a different, independently-composed sentence at assembly time is not derivation, it is invention, for the same reason discussed in §2 above.

Error-result projection is scoped narrowly here: this document does not design the governed error-evidence shape (that remains [Evaluation semantics §9](operation-aware-evaluation-semantics.md#9-condition-evaluation)'s and a later architecture surface's concern). It states only the negative constraint already implied by the "must not invent facts" principle: an errored condition's evidence entry must never present the rule's authored success or deny explanation as though the rule had actually reached that outcome.

### Fixture implication

A canonical fixture must project rule evidence by the table above, consistently, for every rule in every scenario. Where an existing canonical fixture's per-rule evidence disagrees with this table — including any wording inconsistency between a rule's authored `explanation` in its policy bundle and that same text as it appears in the fixture's rule evidence — the fixture is the artifact that needs correction, using the policy bundle's own authored text as the source of truth for a matched rule's `explanation`.

---

## 4. Bundle Identity Provenance

### Governing direction

Bundle identity (`bundle_id`, `bundle_version`) is preserved on the response, trace, and audit-evidence artifacts whenever a trustworthy typed policy bundle exists for the evaluation, independent of whether that bundle's scope covered the request or whether it passed semantic validation:

| Evaluation state                                          | Trustworthy typed bundle exists? | Bundle identity |
| ----------------------------------------------------------- | ------------------------------: | ---------------: |
| Completed, applicable evaluation                            |                              yes |           present |
| Completed, `NOT_APPLICABLE` evaluation                       |                              yes |           present |
| Typed semantic policy-validation failure                     |                              yes |           present |
| Structural bundle-parsing/model-construction failure         |                               no |    null permitted |
| Evaluation failure reached before a typed bundle can be trusted |                            no |    null permitted |

This follows from [Policy rule model §3](operation-aware-policy-rule-model.md#3-policy-scope): "A bundle outside its declared scope should not pretend to deny or allow a request. It should be semantically not applicable." `NOT_APPLICABLE` describes the *outcome* of comparing the request against the bundle's declared scope — it does not describe an evaluation in which no bundle was ever considered. The bundle whose scope was checked and found not to cover the request is a known, specific, typed bundle; reporting its identity is provenance for *which* bundle was checked, not a claim that the bundle applied, matched, or granted anything. The same reasoning extends to a typed semantic policy-validation failure ([Policy rule model §11](operation-aware-policy-rule-model.md#11-policy-validation)): a bundle that constructs successfully as a well-formed, identified typed object and is then rejected by a cross-rule invariant (for example, a duplicate rule identifier) is still the specific policy artifact that was evaluated and rejected — its identity is exactly the fact an auditor needs to know which bundle failed, and omitting it would remove provenance the architecture elsewhere treats as a first-class audit requirement ([Trace and audit evidence §6](operation-aware-trace-audit-evidence.md#6-request-evidence), on preferring identifiable references over silent omission).

This is distinct from a failure reached *before* a typed bundle can be trusted at all — a structural parsing or model-construction failure, where no well-formed bundle object with a validated identity was ever produced. In that case there is no trustworthy identity to report, and `null` remains the correct, honest value; this document does not require `basis-core` to invent or partially trust an unvalidated identifier merely to populate the field.

### What this does not decide

This document does not decide how a future multi-bundle evaluation model (if one is ever introduced) reports identity when several bundles are considered and none applies — that scenario does not exist in the current single-configured-bundle enforcement model this clarification was written against, and extending this table to it is a separate, later architecture question.

### Fixture implication

A canonical fixture must retain bundle identity for a `NOT_APPLICABLE` outcome and for a typed semantic policy-validation failure, using the identity of the bundle the paired request fixture and policy-bundle fixture actually describe. Where an existing canonical fixture publishes `null` bundle identity for either state, the fixture is the artifact that needs correction, not `basis-core`.

---

## 5. Canonical Fixture Implications (Summary)

Restating §§2–4's fixture implications together, for `basis-schemas`' later correction work: a canonical compatibility-vector fixture governed by this document must:

- avoid requiring invented top-level `explanation` prose not derivable from an existing evidence category;
- preserve a matched rule's authored `reason_code`/`explanation` verbatim, including a matched-but-non-decisive rule in a deny-precedence scenario;
- omit authored `reason_code`/`explanation` for a `not_matched` or `skipped` rule;
- represent an `error`-result rule using governed error evidence, never the rule's authored success/deny text;
- retain trustworthy bundle identity for a `NOT_APPLICABLE` outcome;
- retain trustworthy bundle identity for a typed semantic policy-validation failure;
- maintain agreement among response, trace, audit evidence, and (where a shared field repeats) gateway-derived artifacts, consistent with the existing response/trace authority expectations these fixtures already document.

---

## 6. Non-Goals

This document does not:

- change authorization outcomes, deny precedence, default deny, or `NOT_APPLICABLE` semantics;
- change the failure-reason vocabulary;
- add an explanation-generation feature or design its provenance/template/determinism/localization/redaction contract;
- introduce localization;
- introduce a new schema field, model, or contract;
- change gateway artifact ownership or the kernel/gateway evidence boundary;
- change audit persistence ownership;
- design the governed error-evidence shape referenced in §3;
- decide bundle-identity reporting for a future multi-bundle evaluation model.

---

## 7. Open Questions Deferred

This document does not settle:

- a future governed aggregate-explanation-generation feature, if one is ever proposed;
- the final reason-code vocabulary;
- the governed error-evidence shape for a rule whose condition evaluation errors;
- bundle-identity reporting under a future multi-bundle evaluation model;
- whether `basis-schemas` publishes a formal cross-artifact agreement schema constraint for the categories this document narrows, beyond the prose expectations its contracts already carry.

Naming these here is intentional, for the same reason every other operation-aware architecture document in this repository names what it defers: it establishes that this document is aware of what remains open and is not silently deciding those questions by omission.
