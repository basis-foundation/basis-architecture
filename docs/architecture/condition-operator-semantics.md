# Condition Operator Semantics

**Status:** Proposed. Architecture clarification, narrower than an ADR, opened to satisfy the implementation-blocking gate `basis-core`'s v0.2.0 roadmap names in Section 8, "Condition-operator decision gate," of `basis-core`'s `docs/implementation/basis-core-v0.2-operation-aware-plan.md` (a document in the separate `basis-core` repository, not this one — referenced by path, not hyperlinked, per this repository's cross-repository citation convention). This document does not claim that condition evaluation is implemented anywhere in `basis-core`, does not claim runtime conformance against the table below exists, and does not claim `basis-schemas` publishes a closed operator vocabulary. It proposes the first implementable operator subset for `basis-core` v0.2.0 and is subject to `basis-architecture` review before Milestone 7's PR 22 may begin.

**Scope:** The initial, `basis-core`-v0.2.0-scoped operator subset and its complete evaluation semantics: which operators are supported, what each means, field-path resolution, absent/null handling, type compatibility, coercion, ordering, condition-array evaluation order, and the match/no-match/error outcome for every case PR 22 and PR 23 must implement. It does not design a general policy language, does not change the published `policy-condition` shape, and does not resolve the operator vocabulary as a permanently closed, ecosystem-wide enum.

**Companion documents:** [`docs/adr/0004-operation-aware-policy-rule-model.md`](../adr/0004-operation-aware-policy-rule-model.md) (§7, "Conditions" — the semantic constraints this document narrows), [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md) (§7, §9), [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md) (§9–§11, §14, the general missing-context/type-mismatch/error-handling rules this document applies operator-by-operator), [`docs/adr/0002-operation-aware-evaluation-semantics.md`](../adr/0002-operation-aware-evaluation-semantics.md), [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Purpose

`basis-schemas` v0.2.0 has published the `policy-condition` contract: `condition_id`, `field_path`, `operator`, `expected_value`. That contract validates condition *shape* — it confirms `field_path` is a well-formed dotted identifier path and `operator` is a well-formed lowercase snake_case identifier — and deliberately does not close the operator vocabulary or define what any operator does. [ADR-0004 §7](../adr/0004-operation-aware-policy-rule-model.md) requires conditions to be deterministic, side-effect-free, three-outcome (match/no-match/error), non-coercing, and to treat missing context as non-truthy, but likewise stops short of naming an operator set. This is a deliberate, repeatedly-restated gap: ADR-0001 §6, ADR-0002 §16, ADR-0004 §18, and ADR-0005 §14 each name "condition operator language" as explicitly deferred.

The boundary today is:

```text
PolicyCondition shape:              published (basis-schemas v0.2.0)
condition execution semantics:      intentionally unresolved until this clarification
basis-core condition implementation: blocked (Milestone 7, PRs 22-23)
```

This document establishes the minimum deterministic operator semantics required by the first operation-aware kernel implementation. It is scoped as the "first implementable subset for `basis-core` v0.2.0," per the roadmap's own recommendation, not a permanent, ecosystem-wide closed vocabulary decision. A structurally valid `operator` string that is not in the table below remains structurally valid under the published schema; it is simply not implemented by this kernel version, and this document defines exactly what that produces (§6).

---

## 2. Scope

**In scope:**

- The initial supported operator subset for `basis-core` v0.2.0.
- Operator-by-operator behavior: meaning, accepted actual-value types, accepted expected-value types, absent/null handling, type-mismatch handling.
- Field-path resolution: root object, syntax, nested traversal, absent parents, absent leaves, unknown paths, mapping traversal, evidence-reference traversal.
- Coercion rules (there are none; this section makes that concrete per type pair).
- Deterministic condition-array evaluation order.
- The match / no_match / error classification for every case named in Section 8 of `basis-core`'s `docs/implementation/basis-core-v0.2-operation-aware-plan.md`.

**Out of scope:**

A general policy language; arbitrary expressions; code execution; scripts; templates; CEL; Rego; JSONPath or JMESPath as a policy language; external data lookup; network access; protocol access; identity-provider lookup; topology lookup; custom user-defined operators; dynamic plugins; regular-expression support (not justified by any named operator below); location-hierarchy inference; device-taxonomy inference; policy effects; deny precedence; final decision assembly; trace schema design beyond what is needed to state the condition-level outcome; gateway enforcement.

---

## 3. Governing Principles

- Condition evaluation is deterministic: the same condition, actual request, and policy bundle version produce the same outcome every time.
- Condition evaluation is side-effect free.
- Condition evaluation performs no network I/O and no external data retrieval.
- Condition evaluation executes no arbitrary code, script, template, or expression.
- Condition evaluation uses only the already-validated, already-typed `OperationAwareDecisionRequest` supplied to the kernel — nothing else.
- No operator silently coerces incompatible types.
- Unsupported operators produce a defined error, never a silent no-match or silent match.
- Unknown field paths produce a defined result (§7.5), never a silent null and never a silently ignored condition.
- Every evaluation outcome is exactly one of: `match`, `no_match`, `error`. Implementation may internally raise and catch exceptions; those are not an observable semantic outcome — the three-value result above is what PR 23 must produce.

---

## 4. Initial Operator Registry

Ten operators, drawn directly from `policy-condition.yaml`'s `illustrative_operators` (`equals`, `not_equals`, `in`, `greater_than`, `less_than`, `exists`) plus the four operators Section 8 of `basis-core`'s roadmap plan names but the contract's illustrative list omits (`not_in`, `not_exists`, and the `>=`/`<=` complements of `greater_than`/`less_than`). Using the schema's own illustrative names — rather than a different abbreviation such as `eq`/`gt`/`gte` — avoids introducing operator-name churn against the one set of names `basis-schemas` has already published as examples.

| Operator | Meaning | Actual value types | Expected value types | Expected value required | Absent actual value | Explicit null | Type mismatch | Result |
| --- | --- | --- | --- | ---: | --- | --- | --- | --- |
| `equals` | Actual value is exactly equal to `expected_value`. | string, number, boolean | string, number, boolean, null (never matches; see §8) | yes | `no_match` | actual never observably null (§8); `expected_value: null` always `no_match` | `error` | `match` / `no_match` |
| `not_equals` | Actual value is present and not equal to `expected_value`. | string, number, boolean | string, number, boolean, null | yes | `no_match` (absence is not inequality; §8) | `expected_value: null` + present actual → `error` (family mismatch); absent actual → `no_match` | `error` | `match` / `no_match` |
| `in` | Actual scalar value is a member of `expected_value` (must be array). | string, number, boolean | homogeneous array of string, number, or boolean | yes | `no_match` | `no_match` | `error` | `match` / `no_match` |
| `not_in` | Actual scalar value is present and is not a member of `expected_value` (must be array). | string, number, boolean | homogeneous array of string, number, or boolean | yes | `no_match` | `no_match` | `error` | `match` / `no_match` |
| `greater_than` | Actual numeric value is strictly greater than `expected_value`. | number (int/float; never boolean) | number | yes | `no_match` | `no_match` | `error` | `match` / `no_match` |
| `greater_than_or_equal` | Actual numeric value is greater than or equal to `expected_value`. | number | number | yes | `no_match` | `no_match` | `error` | `match` / `no_match` |
| `less_than` | Actual numeric value is strictly less than `expected_value`. | number | number | yes | `no_match` | `no_match` | `error` | `match` / `no_match` |
| `less_than_or_equal` | Actual numeric value is less than or equal to `expected_value`. | number | number | yes | `no_match` | `no_match` | `error` | `match` / `no_match` |
| `exists` | The field path resolves to a present (non-absent, non-null) value. | any | required by schema; value not interpreted | yes (ignored) | `no_match` | `no_match` | never (no type check performed) | `match` / `no_match` |
| `not_exists` | The field path resolves to an absent or explicit-null value. | any | required by schema; value not interpreted | yes (ignored) | `match` | `match` | never (no type check performed) | `match` / `no_match` |

No cell above is `TBD`, `implementation-defined`, or `probably`. Where a behavior is deliberately narrow (for example, `equals` against `expected_value: null` never matching), the reasoning is given in §8 and is intentional, not an oversight.

---

## 5. Operator Vocabulary Boundary

```text
policy-condition.operator   remains an open normalized identifier in the shared contract.
basis-core v0.2.0            implements a finite, approved ten-operator subset (§4).
```

An `operator` value that is structurally valid under `policy-condition.yaml`'s `operator_pattern` but is **not** one of the ten names in §4 must produce `error`. It must never silently become `no_match`, and must never silently become `match`. This is the same rule the roadmap plan already states for unsupported operators, restated here as the binding architecture decision. The shared contract's `operator` field is not redefined as a closed enum by this document, and no `basis-schemas` change is required or implied — adding a future `basis-core`-supported operator is an additive implementation capability that requires its own architecture review (§22), not a `basis-schemas` contract change.

---

## 6. Field-Path Resolution

### 6.1 Root object

Field-path resolution begins at exactly one root: the `OperationAwareDecisionRequest` instance supplied to the evaluator for this decision. No other object — process state, environment variables, files, services, policy metadata, or any other request in the same batch — may ever become a path root.

### 6.2 Path syntax

`field_path` is the dotted identifier path already validated by `policy-condition.yaml`'s `field_path_pattern` (`^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`). Resolution rules:

- **Exact field-name matching.** Each dot-separated segment must match, exactly, a declared field name on the current typed object (`OperationAwareDecisionRequest` at the root, or one of its six nested context objects — `OperationAwareLocation`, `OperationAwareDevice`, `OperationAwareProtocolContext`, `OperationAwareSafetyContext`, `OperationAwareEnvironmentContext`, `OperationAwareRiskContext`). "Exact" includes case: every field name on the request model is already lowercase snake_case, and the pattern itself admits no uppercase, so case sensitivity is not a practical ambiguity, but resolution never performs case-insensitive matching.
- **No aliases.** A segment must be the field's own declared name. No alternate or historical name is accepted.
- **No list indexing.** `subject_roles[0]` is rejected by the schema's own pattern before it reaches resolution (no `[`/`]` characters are permitted), and no bracketed or numeric-index traversal is implemented if a future contract revision relaxed the pattern.
- **No wildcard segments.** No `*`, `**`, or similar traversal-all syntax exists in the pattern or in resolution.
- **Empty segments, leading dots, trailing dots, repeated dots** are all already rejected by `field_path_pattern` at the schema layer (the pattern requires a leading identifier character and forbids `..`, a leading `.`, and a trailing `.`); resolution does not need to re-detect these because a `PolicyCondition` carrying such a path could not have been constructed.
- **Typed field traversal only.** Resolution walks typed model attributes exclusively — never a `getattr`-style dynamic lookup against arbitrary Python object state, never a dict-of-everything lookup, with the single named exception of `subject_attrs` (§6.6).

This is the conservative boundary the roadmap plan anticipates: typed field traversal only, no indexing, no wildcards, no expression syntax.

### 6.3 Optional parent absent

```text
field_path:       location.site_id
request.location: absent (None)
```

Resolution short-circuits at the first absent segment. If `request.location` is `None`, the path `location.site_id` resolves to **ABSENT** without attempting to read `.site_id` off `None`. This is explicit, not an accident of Python attribute-access semantics: the resolution algorithm checks for `None` at every intermediate step and returns ABSENT immediately, rather than raising.

### 6.4 Child field absent

```text
request.location:          present (an OperationAwareLocation instance)
request.location.site_id:  absent (None)
```

This resolves to the same **ABSENT** result as §6.3. An absent intermediate object and a present intermediate object with an absent leaf are semantically identical outcomes for condition evaluation: both produce ABSENT, and every operator in §4 treats ABSENT uniformly (§8). The distinction between "the object was never carried" and "the object was carried but this one field was not populated" is preserved in whatever request-level audit evidence exists elsewhere; it is not preserved as a distinct condition-evaluation outcome, because no operator in the initial subset needs that distinction to behave correctly.

### 6.5 Unknown path

```text
field_path: location.campus_id
```

`OperationAwareLocation` has no `campus_id` field. This path is structurally valid (it matches `field_path_pattern`) but does not resolve to any field this kernel version's request model defines.

**Decision: `error`.** An unknown field path is a policy-authoring defect (a typo, a reference to a field from a different schema version, or a reference to a field this kernel version does not yet implement), not a legitimate "no data available" case. Returning `no_match` would let a deny rule silently fail to fire because of a misspelled path, or let an allow rule's condition silently degrade to "never matches" without any visible signal — exactly the kind of silent failure [Section 11 of the evaluation semantics document](operation-aware-evaluation-semantics.md#11-unknown-action--resource--type-behavior) already rejects for unknown resource types ("An unrecognized value should never resolve to `ALLOW` by default"), and the same reasoning applies here: an unrecognized path must never resolve to a decision-relevant `no_match` that looks, from the outside, like a legitimate evaluated comparison. `error` is loud, is distinct in trace output from a genuine `no_match`, and is exactly the outcome the roadmap plan's own inventory anticipates ("must be a defined evaluation error, never a silent no-match or silent match" — stated there for unsupported operators, and applied here to unsupported paths by the same governing principle).

This is distinct from ABSENT (§6.3–6.4): ABSENT means "a known field on the request model, for which this particular request happened to supply no value." Unknown path means "not a field this kernel version's request model has at all." The two must never be conflated.

### 6.6 Mapping traversal — `subject_attrs`

`OperationAwareDecisionRequest.subject_attrs` is the one open `dict[str, str]` field on the request, used for ABAC-style subject attributes. `policy-condition.yaml`'s own published examples already demonstrate `field_path: subject_attrs.clearance` as a valid, illustrative condition (Example 1 in the vendored contract). Given that precedent, this document makes mapping-key traversal explicit rather than silently assumed:

- `subject_attrs.<key>` is a supported field path, exactly one segment beyond `subject_attrs`. Resolution performs a dictionary lookup of `<key>` against `request.subject_attrs`.
- If `<key>` is present in the mapping, resolution yields that string value (family: string).
- If `<key>` is not present in the mapping, resolution yields **ABSENT** — the same outcome as an absent typed field (§6.4), not an error. An unknown *attribute key* is expected, routine ABAC behavior (not every subject carries every attribute); an unknown *typed field name* (§6.5) is a policy-authoring defect. These are different failure classes and this document deliberately gives them different outcomes.
- Traversal beyond one segment past `subject_attrs` (for example `subject_attrs.department.region`) is **not supported**: `subject_attrs` values are always plain strings, never nested structures, so a second-level segment cannot resolve to anything. This is treated as an unknown path (§6.5) → `error`, because it references a shape the mapping cannot have, not a routinely-absent value.
- `subject_attrs` itself (the bare field path, with no key segment) resolves to the mapping object, not a scalar or array — see §9 for how a structured-object resolution behaves under each operator (only `exists`/`not_exists` are meaningful against it; every comparison operator errors).

No other mapping field exists on `OperationAwareDecisionRequest` or its nested context objects in this contract version, so this rule is `subject_attrs`-specific, not a general "any dict field supports one-segment key lookup" rule — a future mapping-typed field would require its own explicit decision, not an inferred extension of this one.

### 6.7 Evidence-reference traversal

`identity_evidence_reference` (`IdentityEvidenceReference`) and `adapter_evidence_reference` (`AdapterEvidenceReference`) are structurally present, typed fields on `OperationAwareDecisionRequest`. They are **not** part of the addressable condition field-path surface in this initial subset.

**Decision: excluded, not supported in v0.2.0.** Any `field_path` beginning with `identity_evidence_reference` or `adapter_evidence_reference` — whether the bare field or any nested sub-field such as `identity_evidence_reference.identity_source` or `adapter_evidence_reference.protocol` — resolves per §6.5 ("unknown path" treatment) → `error`. This is a deliberate scope boundary, not a resolution-algorithm accident: these two types exist to carry a bounded *reference* to evidence produced by `basis-identity`/`basis-adapters` — a reference identifier, a digest, a redaction classification, and provenance labels — never raw evidence content (`docs/domain/evidence.py`'s own "reference, not proof" boundary). Allowing arbitrary policy conditions to match against evidence-reference internals (digest algorithms, redaction classifications, normalization/mapping versions) before this kernel version has any concrete operational need for it would widen the addressable surface of policy authoring into metadata whose consumption implications have not been reviewed, and would do so through the same clarification meant only to unblock the ten scalar/array operators in §4. If a real, reviewed need to condition on evidence-reference metadata emerges, it should be proposed as its own, separately-scoped architecture change — not inferred from this document's general field-path rule.

### 6.8 Summary table

| Case | Resolution result |
| --- | --- |
| Root object | `OperationAwareDecisionRequest` — no other root permitted |
| Path syntax | Typed field traversal only; no indexing, wildcards, aliases, or expression syntax |
| Case sensitivity | Exact match (all model field names are lowercase; no case-insensitive fallback) |
| Optional parent absent | ABSENT (§6.3) |
| Child field absent | ABSENT (§6.4), identical to parent-absent |
| Unknown typed field path | `error` (§6.5) |
| `subject_attrs.<key>`, key present | resolves to the string value |
| `subject_attrs.<key>`, key absent | ABSENT (§6.6) |
| `subject_attrs` traversal beyond one key segment | `error` (unknown path) |
| `identity_evidence_reference` / `adapter_evidence_reference` (any depth) | `error` (excluded, §6.7) |

---

## 7. Absent and Null Semantics

Four distinct states are named by the roadmap plan: path absent, parent absent, field present with null, field present with a concrete value. This document collapses the first three into two resolution states, and explains exactly why.

- **ABSENT** — the field-path resolution algorithm reached a `None` at some point before the final segment (parent absent, §6.3), or reached `None` at the final segment (leaf absent, §6.4), or resolved a `subject_attrs` key that is not present in the mapping (§6.6). All three collapse to one resolution state because every optional field on `OperationAwareDecisionRequest` and its nested context objects is typed `T | None = None` — Pydantic, and the JSON the contract describes, cannot distinguish "the producer omitted this field" from "the producer explicitly sent `null` for this field." Both deserialize to the same Python `None`. **`basis-core` therefore cannot observe, and this document does not invent, a distinction between "absent" and "explicit null" for any actual request field.** This is stated here as a deliberate architectural reading of the request contract, not an oversight: a future request-model revision that wanted to represent "explicitly null, as distinct from omitted" would need its own field-shape change (for example a sentinel value or an explicit presence flag), which this document does not propose.
- **PRESENT** — resolution reached a concrete, non-`None` value at the final segment: a string, a number, a boolean, an array, or (for `subject_attrs`) a mapping-value string.

Per-operator behavior against ABSENT is given in §4 and restated in §14. The short answer, stated once here as the governing default (consistent with [§10 of the evaluation semantics document](operation-aware-evaluation-semantics.md#10-missing-context-behavior), "missing optional context used by a rule condition — that condition does not match"): **every comparison operator (`equals`, `not_equals`, `in`, `not_in`, and all four ordering operators) treats ABSENT as `no_match`**, never as `error` and never as an implicit `match`. `not_equals` in particular does **not** treat ABSENT as satisfying "not equal" — absence is not automatically inequality, per the roadmap plan's own explicit warning. `exists` treats ABSENT as `no_match`; `not_exists` treats ABSENT as `match`. Those two are the only operators for which ABSENT is a positive signal rather than a uniform non-match.

`expected_value: null` is a distinct, meaningful authored value on the *policy* side (the schema explicitly allows it, and a condition author may write it deliberately — for example, an illustrative "exists"-style condition that does not otherwise need a value, per the vendored contract's own commentary). Because no actual request field can ever be observed as explicitly null (see above), a condition comparing to `expected_value: null` cannot ever match a present, typed value under `equals` (different family — see §8, §16) and cannot ever match an ABSENT actual value either (ABSENT is `no_match` under `equals` regardless of what `expected_value` is). The practical consequence: `equals`/`not_equals` against `expected_value: null` are accepted by the schema and by this evaluator, but they are not the mechanism for testing presence — `exists`/`not_exists` are. This is stated explicitly so a policy author is not surprised that `equals(x, null)` never matches anything in this request model.

---

## 8. Equality Semantics (`equals`, `not_equals`)

- **Exact type requirements.** `equals`/`not_equals` require the actual value's family and the expected value's family to be the same one of: string, number (int and float unified — see §9), boolean. `expected_value: null` is its own family and is addressed separately below.
- **String comparison** is exact, byte-for-byte string equality. No case-folding, no whitespace normalization, no locale-aware comparison.
- **Boolean comparison** is exact boolean equality (`true` only equals `true`; `false` only equals `false`).
- **Numeric comparison** is exact numeric equality across the unified int/float "number" family (`1` equals `1.0`); see §9 and §16 for why this is not considered coercion.
- **Array actual values are not supported by `equals`/`not_equals` in this initial subset** — see §12 for why array-typed actual values (`subject_roles`, `safety_context.constraint_ids`, `environment_context.condition_ids`) are deferred from every operator except `exists`/`not_exists`.
- **Array expected values are not supported by `equals`/`not_equals`** — an authored condition using `equals` with an array `expected_value` is a malformed semantic operand for this operator (arrays are `in`/`not_in`'s expected shape, not `equals`'s) → `error`.
- **Null behavior:** see §7. `equals` against `expected_value: null` never matches (ABSENT actual → `no_match` per the general absence rule; PRESENT actual → different family from `null` → `error` under the type-mismatch rule in §16, since a present, typed value is never itself the null family). `not_equals` against `expected_value: null`: ABSENT actual → `no_match` (absence does not satisfy inequality, per §7); PRESENT actual → `error` (family mismatch, same reasoning as `equals`).
- **Absent behavior:** `no_match` for both `equals` and `not_equals` (§7). This is the specific case the roadmap plan calls out: `not_equals` must not silently match merely because the field is absent, and it does not.

---

## 9. Type System

The condition evaluator's actual-value type families, derived from the published request contract and this document's own decisions above:

| Family | Populated by | Notes |
| --- | --- | --- |
| `string` | `subject_id`, `action`, `resource`, `resource_type`, `location.*_id`, `device.device_id`, `device.device_class`, `protocol_context.protocol`, `protocol_context.operation`, `safety_context.mode`/`classification`, `environment_context.mode`, `risk_context.classification`, `subject_attrs.<key>`, `operation_intent` (resolved to its `.value`, §9.1), and others | Never conflated with `number` or `boolean` |
| `number` | `risk_context.score` (float) | int and float share one family (§9.2); never conflated with `boolean` |
| `boolean` | none on the current request model as a leaf scalar | reserved family; no current field populates it directly, kept distinct for forward compatibility and because `expected_value` itself may be boolean |
| `array` | `subject_roles`, `safety_context.constraint_ids`, `environment_context.condition_ids` | homogeneous string arrays; deferred as an actual-value family from every operator except `exists`/`not_exists` (§12) |
| `timestamp` | `evaluation_time` | distinct from `string`, even though its wire representation is an ISO 8601 string; see §15 |
| `structured object` | `location`, `device`, `protocol_context`, `safety_context`, `environment_context`, `risk_context`, `subject_attrs` (bare, no key), `identity_evidence_reference`, `adapter_evidence_reference` (excluded, §6.7) | only `exists`/`not_exists` are meaningful against this family; every comparison operator errors |
| `ABSENT` | any optional field/segment not populated | §7 |

### 9.1 Enum-backed values

`operation_intent` is the one closed-enum field on the request (`OperationIntent`: `read_only` / `state_changing` / `control_affecting`). Field-path resolution against `operation_intent` yields the enum member's canonical `.value` string (for example `"read_only"`), not the enum object itself — the same treatment `evaluate_rule_selectors`'s own selector-to-request mapping already applies (`operation_intents → request.operation_intent.value`). This keeps `operation_intent` in the `string` family for condition purposes, directly comparable to a plain string `expected_value` such as `"read_only"`, with no separate "enum" family and no special-case operator.

### 9.2 Boolean is not numeric

Python's `bool` is a subclass of `int`; this evaluator does not inherit that behavior. `true` is never equal to `1`, and `false` is never equal to `0`, under `equals` or any ordering operator. `PolicyCondition`'s own `expected_value` validation already classifies booleans into a distinct family from numbers (`_FAMILY_BOOLEAN` vs. `_FAMILY_NUMBER` in the published condition model), and `OperationAwareRiskContext.score` already rejects a `bool` at construction time for the same reason. This document extends that same family separation to the actual-value side of every comparison: a `boolean`-family actual value never participates in `equals`/`not_equals`/ordering against a `number`-family expected value, or vice versa — that combination is a type mismatch → `error`.

### 9.3 int/float unification

`int` and `float` are one `number` family, per `PolicyCondition.expected_value`'s own published treatment (the contract publishes a single `number` type, not separate `integer`/`number` types). `1` and `1.0` are the same family and compare equal under `equals`; this is not coercion (§16) because no type conversion occurs — both are already members of the same declared family before comparison.

### 9.4 Non-finite numbers

`PolicyCondition.expected_value` already rejects non-finite numbers (`NaN`, `Infinity`) at construction time; `OperationAwareRiskContext.score` does the same for the one numeric actual-value field this contract currently defines. No operator in §4 needs to define behavior for a non-finite actual or expected value, because the typed models this evaluator consumes cannot carry one.

---

## 10. Coercion

**Global rule: no silent coercion, anywhere, for any operator.** Made concrete:

| Comparison | Outcome |
| --- | --- |
| `"3"` (string) vs. `3` (number) | `error` — different families, never coerced |
| `"true"` (string) vs. `true` (boolean) | `error` — different families, never coerced |
| `1` vs. `1.0` | not coercion — both are already `number` family (§9.3); compares equal |
| enum-backed value (`operation_intent`) vs. serialized string | not coercion — resolution already yields the `.value` string (§9.1), so both sides are already `string` family |
| timestamp text (a `string`-typed `expected_value`) vs. `evaluation_time` (`timestamp`-family actual) | `error` for every operator except `exists`/`not_exists` — timestamp is its own family, deliberately never treated as `string` (§15) |
| single scalar actual vs. one-element array `expected_value` | `error` under `equals`/`not_equals` (array `expected_value` is not a valid operand for those operators) — never implicitly unwrapped or implicitly wrapped |

---

## 11. Equality Semantics — see §8

(Retained as its own numbered section above per the required document structure; not duplicated here.)

---

## 12. Membership Semantics (`in`, `not_in`)

- **Direction.** `in`/`not_in` test whether the **actual scalar value is a member of the `expected_value` array** — never the reverse. This is the direction `policy-condition.yaml`'s own illustrative example uses (`location.site_id in [west-campus, east-campus]`): a scalar actual field tested against an authored set of alternatives. "Expected scalar is a member of an actual array" (the reverse direction) is a different, unimplemented semantic in this initial subset — see §12.1.
- **Actual operand shape:** a scalar in the `string`, `number`, or `boolean` family. Array-typed actual values are not supported (§12.1).
- **Expected operand shape:** a homogeneous array of `string`, `number`, or `boolean` scalars, per `policy-condition.yaml`'s own `expected_value_array_item_types` (heterogeneous arrays are already rejected at the schema layer, so this evaluator never observes one).
- **Membership rule:** exact equality (same rules as §8) against any one element of the array. `in` matches if the actual value equals at least one array element in the same family; `not_in` matches if the actual value is present and equals none of the array elements, and the actual value's family matches the array's family (a family mismatch between actual and the array is `error`, not `no_match` — see §16).
- **Duplicate elements** in `expected_value` have no effect on the result (membership is a set test, not a positional one); duplicates are not rejected by this evaluator, though a future policy validator may choose to flag them as an authoring quality issue — that is out of scope here.
- **Ordering** of `expected_value`'s elements has no effect on the result.
- **Empty array `expected_value`:** `in` against an empty array never matches (no element can equal the actual value) → `no_match` for any present, well-typed actual value. `not_in` against an empty array always matches for any present, well-typed actual value (vacuously "not a member of nothing") → `match`.
- **Missing (ABSENT) actual value:** `no_match` for both `in` and `not_in` (§7) — an absent value is not a member of anything, and is not treated as satisfying "not a member" either, for the same non-auto-match-on-absence reasoning as `not_equals`.
- **Type mismatch:** actual family different from the array's element family → `error` (§16).

### 12.1 Deferred: array-typed actual values

Whether a condition can test list-shaped *request* context (`subject_roles`, `safety_context.constraint_ids`, `environment_context.condition_ids`) against a scalar or array `expected_value`, and whether that would mean "any of" or "all of," is **explicitly deferred** from this initial subset. `subject_roles` in particular defaults to an empty list rather than `None` (`Field(default_factory=list)`), so it is never ABSENT in the sense §7 defines — it is a caveat worth naming here rather than silently discovering later: `exists` against `subject_roles` will always `match` (the field is always present, even when empty), and `not_exists` will never match it. No operator in this initial subset tests array *emptiness* or array *membership-in-reverse*; a policy author who needs "does this subject have any of these roles" cannot express it with the ten operators in §4. This is a named gap for future work (§23), not an oversight.

---

## 13. Ordering and Numeric Comparison (`greater_than`, `greater_than_or_equal`, `less_than`, `less_than_or_equal`)

- **Supported numeric types:** the unified `number` family only (int and float, per §9.3). `expected_value` must also be `number`-family; a non-numeric `expected_value` (string, boolean, array, null) is a malformed semantic operand for an ordering operator → `error`.
- **Integer/number compatibility:** an int actual value compares directly against a float `expected_value` (and vice versa) using ordinary numeric comparison — same family, not coercion (§9.3).
- **Boolean exclusion:** a `boolean`-family actual or expected value is never accepted by any ordering operator, even though Python would happily compare `True > 0` — this evaluator rejects that combination as a type mismatch → `error` (§9.2).
- **String ordering: not supported.** Lexical string comparison (`"a" < "b"`) is deliberately excluded from the initial subset to avoid locale-dependent or surprising ordering behavior for OT identifiers that are not intended to carry ordering semantics (site IDs, device classes, and so on). A `string`-family actual or expected value under any ordering operator → `error`.
- **Timestamp ordering:** deferred — see §15.
- **Type mismatch:** any actual/expected combination outside "both `number`-family" → `error`, never a silent `no_match`.
- **Absent actual value:** `no_match` (§7) — an ordering comparison against a value that is not there cannot be evaluated as true or false; it is treated the same as every other comparison operator's absence behavior, not as an error, consistent with "missing optional context used by a rule condition — that condition does not match."
- **Non-finite values:** cannot occur (§9.4).

---

## 14. Time Comparison

**Decision: time comparison is deliberately deferred from this initial subset.**

`evaluation_time` is a `timestamp`-family actual value (§9), distinct from `string` even though its JSON representation is an ISO 8601 string. None of the ten operators in §4 accepts a `timestamp`-family actual value for comparison purposes:

- `equals`, `not_equals`, `in`, `not_in`, and all four ordering operators against `evaluation_time` → `error` (type mismatch: no `expected_value` family this contract can carry — string, number, boolean, null, or an array of those — is ever the same family as `timestamp`).
- `exists`, `not_exists` against `evaluation_time` → behave normally (§4), because presence-testing does not require comparing timestamp values to anything.

This is a deliberate boundary, not an accidental gap: allowing `equals`/ordering operators to compare `evaluation_time` against a `string`-typed `expected_value` would silently create lexical, non-chronological string comparison (`"2026-07-14T10:00:00Z" < "2026-07-14T9:00:00Z"` is lexically true and chronologically false) — exactly the "accidentally create time semantics through generic lexical string comparison" trap this document must not fall into. A future, separately-reviewed clarification may add explicit timestamp-aware operators (with a stated accepted format, a UTC-normalization rule, and explicit rejection of timezone-naive input); this document does not design that operator now.

---

## 15. Missing-Value Semantics

Per-operator absent-value behavior (restating §4's table as the required standalone view):

| Operator | Path absent (ABSENT) |
| --- | --- |
| `equals` | `no_match` |
| `not_equals` | `no_match` (absence is not inequality) |
| `in` | `no_match` |
| `not_in` | `no_match` (absence is not "not a member") |
| `greater_than` / `greater_than_or_equal` / `less_than` / `less_than_or_equal` | `no_match` |
| `exists` | `no_match` |
| `not_exists` | `match` |

No operator in this table returns `error` for an ABSENT actual value where the path itself is a known, structurally valid field (§6.5's `error` is reserved for genuinely unknown paths, a different case). This distinction — known-but-unpopulated vs. genuinely-unrecognized — is the single most consequential missing-value distinction this document makes, per [§10 of the evaluation semantics document](operation-aware-evaluation-semantics.md#10-missing-context-behavior)'s own three-way framing (missing optional context used by a condition does not match; missing required request context is invalid; missing context required by an applicable policy bundle is deterministic denial or evaluation failure, a bundle-level concern this document does not decide).

---

## 16. Type-Mismatch Semantics

**One governing rule:** any comparison operator (`equals`, `not_equals`, `in`, `not_in`, and the four ordering operators) presented with a PRESENT actual value whose family does not match what the operator requires from `expected_value` produces `error` — never a silent `no_match`, and never an implicit coercion. `exists`/`not_exists` never type-check (they only test presence) and therefore never produce a type-mismatch `error`.

This single rule replaces sixteen independent per-case judgment calls with one applied uniformly:

- `equals`/`not_equals`: actual family ≠ expected family (including the `null` family, §8) → `error`.
- `in`/`not_in`: actual family ≠ the array's element family → `error`.
- Ordering operators: actual family ≠ `number`, or expected family ≠ `number` (including `boolean`, which is excluded even though numeric-shaped, §9.2, §13) → `error`.

A `no_match` from any of these operators therefore always means "the values were comparable and were found to differ (or not intersect)" — never "the values could not be compared." This distinction matters for trace and audit explainability: a policy author or auditor reading `no_match` can trust that a real comparison happened.

---

## 17. Condition Evaluation Order

Conditions are evaluated in **authored array order** — the order `PolicyCondition` entries appear in `OperationAwarePolicyRule.conditions` (a `list[PolicyCondition]`, never a set or a map with incidental iteration order).

**Full evaluation, no short-circuiting.** Every condition in a rule's `conditions` array is evaluated, in order, regardless of whether an earlier condition already produced `no_match` or `error`. This is a deliberate choice, not the only defensible one, made for trace completeness: [§13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output) requires trace output to be "clear about matched rules, skipped rules, missing context, and reason codes," and the roadmap plan's own PR 23 completion criterion is that `TraceRuleEvidence.condition_results` is "populated correctly" — read here as populated for every authored condition, not only up to the first failure. A policy author or auditor reviewing a rule that did not match should be able to see every condition's individual result, not just the first one that failed.

- **Later conditions are never recorded as "skipped."** Every condition produces its own `match`/`no_match`/`error` result, independent of any other condition's result.
- **Condition order is evaluation order, not effect precedence.** The order conditions appear in the array has no bearing on rule effect, deny precedence, or any other authorization-outcome concept — those are governed entirely by [ADR-0002 §6](operation-aware-evaluation-semantics.md#6-deny-precedence) and are unaffected by this document.
- **This decision remains compatible with future `condition_results`:** an ordered, per-condition result list is exactly what a `condition_results` trace field (Milestone 8, PR 24 onward) needs to represent; nothing here presupposes a specific trace schema shape beyond "one result per authored condition, in authored order."

---

## 18. Rule-Level Aggregation Boundary

Scoped narrowly, only enough to unblock PR 23; this document does not design general rule/bundle combining semantics (that is [ADR-0002](operation-aware-evaluation-semantics.md) and [ADR-0004 §9](operation-aware-policy-rule-model.md#9-bundle-combining-and-rule-combining)).

Given the full set of per-condition results for one rule (§17):

```text
any condition result is `error`   → rule_result = error
else, any condition result is `no_match` → rule_result = not_matched
else (every condition matched, or there were zero conditions) → rule_result = matched
```

`error` takes precedence over `not_matched`: a rule with one erroring condition and one non-matching condition is `error`, not `not_matched`, because an evaluation failure inside a rule means that rule's outcome could not be reliably determined at all — reporting it as an ordinary non-match would misrepresent an evaluation problem as a policy outcome, the same distinction [§14 of the evaluation semantics document](operation-aware-evaluation-semantics.md#14-safe-error-handling) draws between evaluation failure and a substantive `DENY`. This matches the roadmap plan's own stated aggregation ("`matched` requires all conditions to match; any condition `error` makes the rule `error`") and is restated here as the condition-level contribution to that rule-level rule.

This section does not redefine deny precedence, does not implement bundle-level combining, and does not decide what happens to the overall decision when a rule's `rule_result` is `error` — that is downstream aggregation work, out of scope here.

---

## 19. Error Semantics

The three-outcome model, with a representative example of each case this document defines:

- **`match`** — a supported operator, a resolvable field path, compatible types, and the comparison holds. Example: `equals` on `safety_context.mode = "interlock-engaged"` against `expected_value: "interlock-engaged"`.
- **`no_match`** — a supported operator, a resolvable field path, compatible types (or the actual value is ABSENT, per §15), and the comparison does not hold. Example: `equals` on `safety_context.mode = "normal"` against `expected_value: "interlock-engaged"`.
- **`error`** — any of: an unsupported operator (§5); an unknown field path (§6.5); a type mismatch (§16); a malformed semantic operand reaching evaluation (for example, an array `expected_value` supplied to `equals`, or a non-numeric `expected_value` supplied to an ordering operator); an excluded evidence-reference path (§6.7). Structural policy validation (a future Milestone 4/PR 15 concern) may catch some of these earlier, at bundle-load time, before a request is ever evaluated against the offending condition — this document defines the required condition-evaluation-time outcome regardless of which pipeline stage first detects the problem, so that PR 23's evaluator is correct even before a validator exists to pre-empt it.

`malformed expected value that somehow reaches evaluation` — the schema's own `expected_value` validation (scalar or homogeneous array of string/number/boolean, non-finite numbers rejected) already prevents most malformed shapes from existing as a constructed `PolicyCondition` at all. The residual case this document must still cover is an authored value that is *structurally* valid per the schema but *semantically* wrong for its operator (an array given to `equals`, a boolean given to an ordering operator) — those are the "malformed semantic operand" `error` cases named above, not schema violations.

---

## 20. Security Boundaries

Condition evaluation, as scoped by this document, must never perform: network access; database access; filesystem lookup; cloud API calls; identity-provider lookup; a gateway callback; an adapter callback; a live protocol read; a topology lookup; a live device-state fetch; arbitrary Python execution; user-defined code; dynamic imports; templates; scripts; shell commands; CEL; Rego; embedded JavaScript; unbounded (or any) regular-expression evaluation; or arbitrary expression evaluation of any kind. Condition evaluation consumes only the bounded, already-validated `OperationAwareDecisionRequest` fields named in §9 — nothing this document defines requires, or is satisfiable by, anything outside that object.

---

## 21. Determinism and Boundedness

Identical inputs (the same `PolicyCondition`, the same `OperationAwareDecisionRequest`, the same policy bundle version) must produce identical outputs, every time, forever. Condition evaluation as scoped here must not depend on: current wall-clock time; local timezone; locale; environment variables; process-global mutable state; random values; hash-iteration order (relevant to `subject_attrs` dictionary traversal — key lookup by name, per §6.6, never iteration order); network availability; or external service state. Any future timestamp comparison this evaluator implements (§14, deferred) must use the request's own `evaluation_time` field — a value the producer supplied — never a live clock call inside the evaluator.

---

## 22. Compatibility and Vocabulary Governance

```text
The shared operator field remains open (basis-schemas, policy-condition.operator).
The basis-core v0.2.0 supported set is finite: the ten operators in §4.
Unsupported operator:            deterministic error (§5).
Adding a future supported operator: additive implementation capability,
                                     but requires architecture review, because
                                     operator semantics are security-relevant
                                     (they determine what ALLOW/DENY conditions
                                     on).
```

**Is a future `basis-schemas` operator-vocabulary contract needed?** Not for this document. The conservative posture is: not required now; may be considered later, once `basis-core` v0.2.0 has real implementation and deployment experience with the ten-operator subset above, following the same "experimental → candidate → stable" path every other operation-aware contract already follows (per `basis-schemas`' own `docs/contract-governance.md`). This document does not create a `basis-schemas` contract and does not update the released `policy-condition` shape.

---

## 23. Examples

Synthetic OT examples only; no real identities, organizations, device addresses, credentials, or customer data. Each example gives one `match`, one `no_match`, and one `error` case where applicable (existence operators have no meaningful "type mismatch" error case, per §19, so their error example uses an unsupported field path instead).

### `equals`

```yaml
condition_id: cond-safety-mode-interlock
field_path: safety_context.mode
operator: equals
expected_value: interlock-engaged
```
- match: `safety_context.mode = "interlock-engaged"`
- no_match: `safety_context.mode = "normal"`
- error: `safety_context.mode` absent → `no_match`, not error (§8); a genuine error example instead: `expected_value: 1` compared against `safety_context.mode = "interlock-engaged"` (string vs. number family) → `error`

### `not_equals`

```yaml
condition_id: cond-environment-mode-not-maintenance
field_path: environment_context.mode
operator: not_equals
expected_value: maintenance_mode
```
- match: `environment_context.mode = "degraded_connectivity"`
- no_match: `environment_context.mode = "maintenance_mode"`
- error: `environment_context.mode = "maintenance_mode"` (string) compared with `expected_value: true` (boolean family) → `error`; note `environment_context.mode` absent → `no_match`, not `match` (absence is not inequality, §7–§8)

### `in`

```yaml
condition_id: cond-site-in-allowed-set
field_path: location.site_id
operator: in
expected_value:
  - west-campus
  - east-campus
```
- match: `location.site_id = "west-campus"`
- no_match: `location.site_id = "north-annex"`
- error: `location.site_id = 42` cannot occur (the field is string-typed on the request model); a genuine error example instead: `field_path: risk_context.score`, `expected_value: [west-campus, east-campus]` — `risk_context.score` is `number`-family, the array is `string`-family → `error`

### `not_in`

```yaml
condition_id: cond-device-class-not-restricted
field_path: device.device_class
operator: not_in
expected_value:
  - legacy-controller
  - unmanaged-sensor
```
- match: `device.device_class = "gateway"`
- no_match: `device.device_class = "legacy-controller"`
- error: `device.device_class` present as `"gateway"` compared against `expected_value: [1, 2, 3]` (number-family array vs. string-family actual) → `error`

### `greater_than`

```yaml
condition_id: cond-risk-score-above-threshold
field_path: risk_context.score
operator: greater_than
expected_value: 0.5
```
- match: `risk_context.score = 0.72`
- no_match: `risk_context.score = 0.30`
- error: `risk_context.classification = "elevated"` (string) used with `operator: greater_than` → `error` (string not accepted for ordering, §13)

### `greater_than_or_equal`

```yaml
condition_id: cond-risk-score-at-or-above-threshold
field_path: risk_context.score
operator: greater_than_or_equal
expected_value: 0.5
```
- match: `risk_context.score = 0.5`
- no_match: `risk_context.score = 0.49`
- error: `expected_value: true` (boolean) → `error` (booleans excluded from ordering, §9.2, §13)

### `less_than`

```yaml
condition_id: cond-risk-score-below-threshold
field_path: risk_context.score
operator: less_than
expected_value: 0.2
```
- match: `risk_context.score = 0.1`
- no_match: `risk_context.score = 0.5`
- error: `risk_context.score` absent → `no_match`, not error; a genuine error example instead: `field_path: evaluation_time`, `operator: less_than`, `expected_value: "2026-07-14T00:00:00Z"` → `error` (timestamp family never accepted by ordering, §14)

### `less_than_or_equal`

```yaml
condition_id: cond-risk-score-at-or-below-threshold
field_path: risk_context.score
operator: less_than_or_equal
expected_value: 0.2
```
- match: `risk_context.score = 0.2`
- no_match: `risk_context.score = 0.9`
- error: `expected_value: [0.1, 0.2]` (array) → `error` (ordering operators require a scalar `number` expected value)

### `exists`

```yaml
condition_id: cond-evaluation-time-present
field_path: evaluation_time
operator: exists
expected_value: true
```
- match: `evaluation_time = "2026-07-14T14:30:00Z"` (any present, non-null value)
- no_match: `evaluation_time` absent
- error: `field_path: location.campus_id` (unknown field, §6.5) → `error`

### `not_exists`

```yaml
condition_id: cond-device-id-not-supplied
field_path: device.device_id
operator: not_exists
expected_value: true
```
- match: `device` absent entirely, or `device.device_id` absent
- no_match: `device.device_id = "ctrl-042"`
- error: `field_path: adapter_evidence_reference.protocol` (excluded evidence-reference path, §6.7) → `error`

---

## 24. Decision Summary

Implementation-grade table PR 22 can consume directly:

| Operator | Actual type | Expected type | Missing actual | Null actual | Type mismatch | Match rule |
| --- | --- | --- | --- | --- | --- | --- |
| `equals` | string / number / boolean | same family as actual, or null | `no_match` | not observable (§7); `expected_value: null` → `no_match` | `error` | exact equality, same family |
| `not_equals` | string / number / boolean | same family as actual, or null | `no_match` | not observable; `expected_value: null` + present actual → `error` | `error` | present, same family, not equal |
| `in` | string / number / boolean | homogeneous array of string/number/boolean | `no_match` | not observable | `error` | actual equals ≥1 array element, same family |
| `not_in` | string / number / boolean | homogeneous array of string/number/boolean | `no_match` | not observable | `error` | present, same family, equals 0 array elements |
| `greater_than` | number | number | `no_match` | not observable | `error` | actual > expected |
| `greater_than_or_equal` | number | number | `no_match` | not observable | `error` | actual ≥ expected |
| `less_than` | number | number | `no_match` | not observable | `error` | actual < expected |
| `less_than_or_equal` | number | number | `no_match` | not observable | `error` | actual ≤ expected |
| `exists` | any | required, not interpreted | `no_match` | `no_match` (collapsed with absent, §7) | never | actual is PRESENT |
| `not_exists` | any | required, not interpreted | `match` | `match` (collapsed with absent, §7) | never | actual is ABSENT |

Separate case summary:

| Case | Required result |
| --- | --- |
| Unsupported operator | `error` (§5) |
| Unknown field path | `error` (§6.5) |
| Missing intermediate object | ABSENT → per-operator table above (typically `no_match`; `not_exists` → `match`) (§6.3) |
| Missing leaf field | ABSENT → per-operator table above (§6.4) |
| Explicit null | Not observable on any actual request field (§7); treated identically to absent |
| Incompatible types | `error` (§16) |
| Invalid semantic operand (e.g., array to `equals`, non-numeric to ordering) | `error` (§19) |
| Repeated identical evaluation | Identical result every time (§21) |

No cell in either table is left as `TBD`, `implementation-defined`, or "as appropriate."

---

## 25. Documentation Integration

This document is added to `docs/architecture/`, following the lightweight `**Status:**`/`**Scope:**`/`**Companion documents:**` header convention already used by [`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md) and the reconciliation reports, rather than the heavier ADR-companion prose pattern used by the four `operation-aware-*.md` documents each paired 1:1 with ADR-0001 through ADR-0004. That heavier pattern is reserved for documents with a dedicated ADR; this document, per the roadmap plan's own recommendation (§8 of `basis-core-v0.2-operation-aware-plan.md`), is intentionally narrower — an implementation-enabling clarification of a subset of ADR-0004 §7's already-correct deferral, not a new architectural decision requiring its own ADR. Nothing in ADR-0004 is amended or contradicted: ADR-0004 §7 remains the correct statement that a full operator language is out of scope; this document narrows only "what is the first implementable subset," exactly the question ADR-0004 left open on purpose.

---

## 26. Open Questions Deferred

This document does not settle:

- Array-typed actual values (`subject_roles`, `safety_context.constraint_ids`, `environment_context.condition_ids`) as operands to any operator beyond `exists`/`not_exists` — "any of"/"all of" semantics remain unimplemented (§12.1).
- Timestamp-aware comparison operators for `evaluation_time` (§14) — deliberately deferred, not designed here.
- String ordering operators (§13) — deliberately excluded from the initial subset.
- Mapping-key traversal for any field other than `subject_attrs` (§6.6) — no other mapping-typed field exists on the current contract, so this has not been tested against a second case.
- Evidence-reference field-path traversal (§6.7) — excluded pending a separately-reviewed need.
- Case-insensitive string comparison, whitespace normalization, and value aliasing — explicitly not introduced (§8).
- Regular-expression-based operators — not justified by any operator in §4; would require its own security review before proposal.
- A closed, versioned `basis-schemas` operator-vocabulary contract — not created here; may be revisited after real implementation experience with the ten-operator subset (§22).
- Whether duplicate `expected_value` array elements should be flagged by a future policy validator as an authoring-quality issue (§12) — noted, not decided.

Each should be addressed in its own later, separately-scoped architecture work, informed by the semantics this document defines. Naming them here follows the same convention every other operation-aware architecture document in this repository already uses: the absence of a decision on these points is intentional and visible, not an oversight silently discovered later.
