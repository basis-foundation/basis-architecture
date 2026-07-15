# Operation-Aware Trace and Audit Evidence

This document defines the conceptual trace and audit evidence model for the operation-aware authorization model described in [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md) and the evaluation semantics defined in [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md). Section 7 of the authorization model named the audit and trace categories a future evidence model would need to cover and deferred the design of that model to a later architecture PR. Section 13 of the evaluation semantics document named the high-level trace expectations — deterministic, stable enough for tests, safe to expose after redaction, clear about matched/skipped rules and reason codes, free of secrets — and deferred the full trace schema to the same later work. This is that PR.

This is an architecture document, not an implementation specification. It defines conceptual evidence categories, lifecycle, and ownership — not JSON Schema, Python types, database schemas, storage mechanisms, or logging code. The corresponding decision to define this model before publishing trace/audit contracts is recorded in [ADR-0003](../adr/0003-operation-aware-trace-audit-evidence.md). Cross-references: [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md), [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md), [`docs/architecture/action-vocabulary.md`](action-vocabulary.md), [`docs/architecture/identity-authority-modes.md`](identity-authority-modes.md), [`docs/architecture/operation-aware-evaluation-orchestration.md`](operation-aware-evaluation-orchestration.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Purpose

Operation-aware authorization is only as trustworthy as the evidence that explains it. An `ALLOW` or `DENY` outcome, by itself, tells an operator or auditor what happened but not why, and "why" is what makes a decision reviewable, defensible, and debuggable. This document defines what evidence must exist, conceptually, so that a decision can be reconstructed and trusted after the fact — not merely observed.

The evidence model this document defines exists to answer:

- Who requested what?
- What operational context was evaluated?
- Which policy bundle and rules were considered?
- Why was the decision reached?
- What evidence supports the decision?
- What was enforced by the gateway?
- What can be safely shown to an operator, auditor, or training-mode user?

Three sentences summarize the division of labor this document builds on, and each is developed further in [Section 2](#2-trace-vs-audit):

- Trace explains evaluation.
- Audit records evidence.
- Gateway enforcement records what happened at runtime.

Without this distinction, "audit" tends to collapse into whatever the kernel happens to emit, which either overloads the kernel with runtime concerns it should not carry or leaves gateway enforcement facts undocumented and inconsistent across deployments. This document exists so that `basis-schemas`, `basis-core`, `basis-gateway`, and `basis-console` have a stable, reviewed target for evidence rather than an implicit one that each component would otherwise have to infer independently.

---

## 2. Trace vs Audit

This document distinguishes three related but non-identical concepts.

**Evaluation trace** — the kernel-produced explanation of how policy evaluation reached an outcome. It is scoped to evaluation: which rules were candidates, which matched, which were skipped, what context was missing, and what reason codes apply. It is produced by `basis-core` as part of the `DecisionResponse`, consistent with [Section 4 of the authorization model](operation-aware-authorization-model.md#4-rich-decisionresponse-conceptual-fields) and [Section 13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output).

**Audit evidence** — a durable evidence package derived from the request, response, trace, identity context, adapter evidence, and gateway enforcement result. Audit evidence is broader than trace: it includes trace as one input among several, alongside request-side references, identity evidence references, adapter evidence references, and gateway enforcement facts.

**Gateway audit event** — the runtime record `basis-gateway` emits after enforcement. It is the artifact that actually gets persisted, exported, or reviewed; it combines kernel-produced evidence with enforcement facts the kernel cannot know.

Three constraints follow directly from these definitions:

- Trace is not the same as audit. Trace is one input to audit, not a synonym for it.
- Audit may include references to trace. It does not need to duplicate trace content wholesale — a reference is often sufficient, consistent with the preference for hashes and references over raw duplication established in [Section 6](#6-request-evidence).
- Gateway audit may include enforcement facts the kernel cannot know — which route was called, what was returned to the caller, whether enforcement succeeded — none of which the kernel has visibility into.
- Kernel trace must not include runtime enforcement behavior. A kernel that reasoned about enforcement outcomes would violate the same evaluation/enforcement boundary that [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) already protects.

---

## 3. Evidence Lifecycle

The following is a conceptual lifecycle, not an implementation sequence. It describes how evidence accumulates as a request moves through the existing component chain, consistent with the evaluation path already described in [Section 2 of the authorization model](operation-aware-authorization-model.md#2-conceptual-evaluation-path).

```text
Identity context established
        ↓
Adapter evidence normalized
        ↓
Gateway assembles DecisionRequest
        ↓
Kernel evaluates and produces DecisionResponse + trace
        ↓
Gateway enforces decision
        ↓
Gateway emits audit event with request/response/trace/enforcement evidence
        ↓
Console visualizes redacted evidence
```

Two things about this lifecycle matter more than its specific steps. First, evidence accumulates — it is not manufactured at a single point. Identity evidence comes from `basis-identity`, adapter evidence comes from `basis-adapters`, trace comes from `basis-core`, and enforcement evidence comes from `basis-gateway`; no single component originates the whole audit event. Second, and consequential for [Section 14](#14-audit-event-assembly-ownership): `basis-core` does not persist audit, and `basis-console` does not own audit storage. `basis-core` produces evidence as part of its response; it does not write that evidence anywhere durable. `basis-console` reads and visualizes evidence that already exists; it does not become the system of record for it.

Within `basis-core`, "Kernel evaluates and produces DecisionResponse + trace" is itself a composition of policy-owned evaluation facts and this document's audit-owned trace contracts. [ADR-0006](../adr/0006-evaluation-orchestration-layer.md) and its companion document, [`docs/architecture/operation-aware-evaluation-orchestration.md`](operation-aware-evaluation-orchestration.md), assign that composition to the `evaluation` kernel subpackage — not to `policy` and not to `audit` — so that neither subpackage has to import the other to produce a complete trace. This does not change what this document defines: `evaluation` composes the `TraceRuleEvidence` and `EvaluationTrace` contracts this document owns; it does not redefine them.

---

## 4. Evaluation Trace Categories

The following are conceptual trace categories — what an evaluation trace should be able to represent, not a final trace schema. Field names, types, and encoding are deliberately unspecified; that work belongs to `basis-schemas` once this architecture stabilizes.

- Trace ID
- Request ID
- Correlation ID
- Schema version
- Policy bundle identifier
- Policy bundle version
- Evaluation timestamp
- Evaluated action
- Evaluated resource
- Applicable policy scope
- Candidate rule IDs
- Matched rule IDs
- Skipped rule IDs
- Deny rule matches
- Allow rule matches
- Missing context observations
- Unknown value observations
- Condition evaluation outcomes
- Final combining result
- Final decision
- Reason codes
- Safe explanation text

This list is deliberately exhaustive at the category level and deliberately silent on structure. A future trace schema may group these categories differently, nest them, or represent some as references rather than inline values — none of that is decided here. What is decided is that an evaluation trace must be able to represent all of these categories in some form, consistent with the "clear about matched rules, skipped rules, missing context, and reason codes" expectation already stated in [Section 13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output).

---

## 5. Rule Evaluation Evidence

Rule-level evidence is the finest-grained layer of the trace — the record of what happened when each candidate rule was considered. Recommended conceptual fields:

- Rule identifier
- Rule source / policy bundle reference
- Rule applicability
- Condition results
- Match / no-match / error
- Deny / allow effect
- Reason code emitted by the rule
- Redacted explanation

Constraints that apply to rule evidence regardless of eventual schema shape:

- **Rule evidence must be deterministic.** The same rule evaluated against the same request and policy bundle version must produce the same evidence, consistent with the condition-evaluation determinism already required in [Section 9 of the evaluation semantics document](operation-aware-evaluation-semantics.md#9-condition-evaluation).
- **Rule evidence must not expose secrets.** A condition that references sensitive context must not leak the raw value of that context into its evidence record merely because the condition needed to inspect it.
- **Rule evidence should support compatibility tests.** Rule-level evidence is the layer most directly useful for verifying that a kernel version change did not silently alter which rules match, consistent with the compatibility testing expectations in [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md).
- **Rule evidence should support operator explanations without exposing raw sensitive context.** An operator troubleshooting a denial needs to know that a device-class condition did not match; they do not need the raw device attributes that were compared, if those attributes are themselves sensitive.

---

## 6. Request Evidence

Request-side evidence is what audit may reference about the request that was evaluated. Conceptual categories:

- Decision request hash
- Request ID
- Correlation ID
- Subject reference
- Identity source reference
- Authority mode reference
- Action
- Resource identifier
- Resource type
- Site/zone context
- Device identity reference
- Protocol evidence reference
- Operation intent
- Safety/risk/time/environment context references

Audit should usually avoid duplicating the full raw request when a stable hash or reference is sufficient. This preference exists for two reasons: it keeps audit evidence smaller and more stable across schema changes to the request shape itself, and it avoids audit becoming a second, informally-governed copy of request data whose consistency with the original is not guaranteed. A decision request hash, paired with a request ID and correlation ID, is generally enough to let an auditor correlate an audit record back to the original request without the audit record itself needing to carry every field the request carried.

---

## 7. Adapter Evidence Reference

Adapter evidence is protocol-level context normalized by `basis-adapters`. The trace/audit model references it rather than absorbing it.

- Adapter evidence should be preserved by `basis-adapters`. It is not the kernel's or the gateway's evidence to own.
- The kernel may receive normalized adapter evidence or references to it. Which of the two a given deployment carries is a `basis-schemas`/`basis-gateway` question, not something this document resolves.
- Audit should be able to link a decision to the adapter evidence that produced it, whether by embedding a normalized summary or by carrying a reference resolvable back to `basis-adapters`.
- Adapter evidence must not make `basis-core` protocol-aware. Carrying a reference to adapter evidence does not mean the kernel interprets that evidence's protocol-specific content, consistent with the protocol-agnostic boundary restated throughout the authorization model and evaluation semantics documents.
- Raw protocol payloads should be redacted or referenced carefully. A raw BACnet or Modbus payload is not automatically safe to carry into audit; it must go through the same redaction discipline as any other evidence category (see [Section 10](#10-redaction-and-secret-handling)).

Representative adapter evidence categories, at the conceptual level only:

- Protocol
- Protocol operation
- Source endpoint reference
- Normalized action
- Normalized resource
- Normalization rule/mapping ID
- Evidence hash
- Redaction status

This document does not define a final adapter evidence schema. That is explicitly deferred (see [Section 17](#17-open-questions-deferred)).

---

## 8. Identity Evidence Reference

Identity evidence is what audit may reference about the subject and how their identity was established. Conceptual categories:

- Subject identifier
- Subject type
- Identity provider / authority mode reference
- Authentication/federation source reference
- Session/token reference, if safe and non-secret
- Claim mapping version
- Identity normalization version
- Identity evidence hash

The constraints here are stricter than for most other evidence categories, because identity evidence is the category most likely to carry credential material by accident:

- Never include raw tokens.
- Never include private keys, credentials, cookies, session secrets, refresh tokens, or bearer tokens.
- Avoid storing full claim sets when references or redacted views are enough.

A session or token *reference* — an opaque identifier that lets an operator correlate a decision to a session without exposing the session's contents — is a different thing from the token itself, and this document treats only the former as potentially safe to carry. This is consistent with the identity redaction discipline already established across `basis-identity`'s phased work (session cookie binding, login callback composition, and logout/revocation all treat session and token material as sensitive-by-default).

---

## 9. Gateway Enforcement Evidence

Some evidence exists only because the gateway, not the kernel, knows it. This section defines what the gateway contributes that the kernel structurally cannot.

- Gateway instance reference
- Route/API endpoint
- Caller network/context reference, when appropriate
- Enforcement outcome
- HTTP/status mapping, if applicable
- Fail-closed behavior
- Timeout/error behavior
- Response returned to caller
- Audit emission timestamp
- Gateway correlation ID
- Gateway policy loader state, if relevant

The governing statement for this section:

```text
basis-core decides.
basis-gateway enforces and records enforcement facts.
```

The kernel trace must not pretend to know gateway enforcement behavior. A kernel that produced a decision does not know whether the gateway successfully enforced it, whether the caller received the response, or whether a timeout occurred downstream of the decision — those facts exist only at the gateway, and the audit event that combines kernel evidence with gateway evidence is where they are first joined together. This mirrors the boundary already stated in [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership): `basis-gateway` enforces decisions and emits gateway audit evidence; `basis-core` evaluates policy and emits decision metadata and (kernel-side) audit evidence. The two evidence sets are assembled together by the gateway, not produced by the same component.

---

## 10. Redaction and Secret Handling

This section is critical, and its governing rule is unconditional:

```text
No audit, trace, or explanation artifact may contain secrets.
```

Examples that must never appear in any trace, audit, or explanation artifact:

- Bearer tokens
- Raw JWTs
- Refresh tokens
- Cookies
- Private keys
- Client secrets
- Passwords
- API keys
- Unredacted credentials
- Sensitive raw protocol payloads, unless explicitly classified safe

Redaction categories that evidence should be sorted into, conceptually:

- **Safe to expose** — evidence that can appear in trace, audit, and operator-facing explanations without redaction (for example, a matched rule ID or a reason code).
- **Safe after redaction** — evidence that carries useful signal but must have sensitive components removed or masked before it is safe to expose (for example, a claim value that identifies a subject attribute used in a condition, with the raw claim redacted but the fact that the condition matched preserved).
- **Reference only** — evidence that must never be carried in full, only as a resolvable reference to a system that holds the full value under its own access controls (for example, a session reference rather than session contents).
- **Never store** — evidence that should not be persisted into audit at all, even in redacted form, because no redaction makes it safe or useful (for example, raw protocol payload bytes with no defined redaction rule).
- **Never display** — evidence that may exist in a durable record for narrow purposes (e.g., cryptographic verification) but must never be rendered in an operator-, auditor-, or training-mode-facing view.

Redaction must be deterministic enough for tests and operator trust. A redaction rule that behaves inconsistently — sometimes masking a field, sometimes not, depending on incidental code paths — is not a redaction rule an auditor or a compatibility test can rely on. This document does not define the redaction implementation; it establishes that redaction behavior is itself a property that must be stable and testable, not incidental.

---

## 11. Human-Readable Explanation

Explanation is distinct from trace:

```text
Trace:
  structured, machine-readable evidence.

Explanation:
  human-readable rendering of selected trace/audit evidence.
```

Explanations should support:

- Operator mode
- Training mode
- Audit review
- Denial troubleshooting
- Policy debugging

Constraints on explanation:

- Explanation must not invent facts. Every statement in an explanation must be traceable to a specific trace or audit evidence category.
- Explanation must be derived from trace/evidence, not generated independently of it.
- Explanation must preserve uncertainty. If a trace records an evaluation error or a missing-context observation, the explanation must say so rather than smoothing it into a confident-sounding narrative.
- Explanation must not reveal secrets. The redaction constraints in [Section 10](#10-redaction-and-secret-handling) apply to explanation text exactly as they apply to structured evidence — arguably more so, since prose is easier to leak sensitive detail into carelessly than a structured field.
- Explanation should not become a policy engine. Rendering "why" a decision was reached is not the same as re-deriving or re-evaluating the decision; an explanation renderer that reasons about policy rather than reporting on evaluation that already happened has crossed into evaluation territory that belongs to `basis-core` alone.

---

## 12. Reason Codes

Reason codes are the stable, machine-readable vocabulary that trace, audit, console, and tests all key off of. Their role:

- Stable machine-readable values.
- Attached to decisions, matched rules, skipped rules, and errors.
- Used by tests, console, audit, and troubleshooting.
- Governed as a compatibility surface — consistent with the "stable reason codes" capability already named in [Section 5 of the authorization model](operation-aware-authorization-model.md#5-policy-model-direction).
- Eventually published through `basis-schemas`.

Examples only, not a final vocabulary:

```text
ALLOW_RULE_MATCHED
DENY_RULE_MATCHED
NO_ALLOW_RULE_MATCHED
MISSING_REQUIRED_CONTEXT
UNKNOWN_RESOURCE_TYPE
UNSUPPORTED_SCHEMA_VERSION
POLICY_BUNDLE_INVALID
EVALUATION_ERROR
```

This document does not finalize the reason-code vocabulary. It records that reason codes are a governed compatibility surface — analogous in spirit to the action vocabulary's governance model referenced in [`docs/architecture/action-vocabulary.md`](action-vocabulary.md) — and that the categories above are illustrative of the kind of distinctions (substantive denial vs. missing context vs. evaluation failure) the eventual vocabulary needs to preserve, consistent with [Section 5 of the evaluation semantics document](operation-aware-evaluation-semantics.md#5-not_applicable-semantics) and [Section 14](operation-aware-evaluation-semantics.md#14-safe-error-handling) of that document.

---

## 13. Determinism and Compatibility

Trace and audit evidence carry the same determinism and compatibility expectations already established for evaluation itself in [Section 12](operation-aware-evaluation-semantics.md#12-schema-version-compatibility) and [Section 13](operation-aware-evaluation-semantics.md#13-deterministic-trace-output) of the evaluation semantics document:

- Identical request + identical policy bundle + identical evaluation config should produce equivalent trace outcomes.
- Trace ordering must be stable — matched and skipped rule lists, in particular, must not vary based on incidental iteration order.
- Matched/skipped rule reporting must be deterministic.
- Trace shape should be versioned, independent of the `DecisionRequest`/`DecisionResponse` schema version, so that trace structure can evolve on its own compatibility timeline.
- Additive fields must not alter old decisions. A trace or audit schema that adds a new category must not change the decision outcome or the trace content for requests that do not use the new category.
- Compatibility snapshots should protect trace behavior, the same way [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md) already expects for other compatibility surfaces.
- Reason-code changes should be treated carefully, since reason codes are consumed by tests, console rendering, and operator troubleshooting simultaneously — a reason code renamed or removed without care breaks all three at once.

---

## 14. Audit Event Assembly Ownership

Evidence has many contributors; assembly is not shared. This section states, conceptually, who produces what and who assembles the final record.

```text
basis-core:
  produces decision response, reason metadata, evaluation trace, kernel-side audit evidence

basis-gateway:
  assembles gateway audit event, combines kernel evidence with enforcement facts, emits audit

basis-adapters:
  produce adapter evidence and normalization evidence

basis-identity:
  produces identity evidence and identity normalization evidence

basis-console:
  visualizes redacted evidence and explanations

basis-schemas:
  later publishes stable contracts for trace/audit evidence
```

Neither `basis-core` nor `basis-console` owns durable audit storage. `basis-core` produces the kernel-side evidence that becomes part of an audit event, but it does not persist that evidence anywhere durable — it returns it as part of a `DecisionResponse`. `basis-console` reads and visualizes audit evidence that already exists elsewhere; it is not the system of record, and building console features that assume it can retain or reconstruct audit history on its own would misplace a responsibility that belongs to the gateway (or whatever durable store the gateway writes to, which this document does not design).

---

## 15. Audit Failure Behavior

What happens when audit emission itself fails is a runtime and deployment concern, not something this architecture-level document can settle. But the architecture must require that the question be answered explicitly by later work, rather than left as undefined behavior:

- Audit failure handling is a gateway/runtime concern. It is not a kernel concern — `basis-core` produces evidence; it does not emit or persist audit, so it cannot fail to emit audit in the way the gateway can.
- The architecture should require fail-safe behavior to be explicit. Whatever a deployment chooses to do when audit emission fails, that choice must be a documented, deliberate policy — not an implicit consequence of how the emission code happens to be written.
- Some deployments may require fail-closed on audit failure. An OT environment with strict compliance requirements may need to refuse to enforce a decision if the corresponding audit event cannot be durably recorded.
- Some degraded OT modes may buffer audit locally. A deployment prioritizing availability may choose to buffer audit events during a connectivity gap and reconcile later, accepting a different risk tradeoff than fail-closed.
- The kernel must not decide runtime audit failure policy. Fail-open versus fail-closed on audit failure is a `basis-gateway`/deployment decision, consistent with the same reasoning that already keeps runtime failure behavior out of the kernel in [Section 14 of the evaluation semantics document](operation-aware-evaluation-semantics.md#14-safe-error-handling).
- Audit failure itself must become an audit/diagnostic event when possible. A deployment that silently drops audit records on failure, with no trace that the failure occurred, has created a gap that is worse than a visible one — this document requires that the failure be observable in principle, without prescribing the mechanism.

This document does not define a final deployment policy for audit failure. It requires that later work define one explicitly rather than leave it to incidental implementation behavior.

---

## 16. Console and Training-Mode Use

`basis-console` is a consumer of trace and audit evidence, not a producer of it. Its use of that evidence should be bounded as follows:

- Operator mode shows concise decision/explanation — the minimum an operator needs to understand and act on a decision.
- Training mode shows expanded flow and context — more of the trace, more of the evaluation path, intended for onboarding and understanding rather than day-to-day operation.
- Both modes use the same underlying evidence. Training mode is a different *presentation* of the same evidence, not a separate or embellished evidence source, consistent with the operator-vs-training-mode distinction already established as UX/copy-only in prior `basis-console` work.
- Console may not invent evidence. Anything training mode displays must be traceable to an actual trace or audit category from this document, not a plausible-sounding elaboration.
- Console may not evaluate policy. Rendering an explanation of an evaluation that already happened is not the same as re-running or approximating that evaluation for display purposes.
- Console must respect redaction metadata. Whatever a trace or audit record marks as redacted, reference-only, or never-display must be honored by every console rendering path, including training mode's more expansive views.
- Console should distinguish sample, live, and future capability data. A training-mode view built on illustrative sample data must not be presented in a way that could be mistaken for a live decision, and features that do not exist yet must not be rendered as though they already produce evidence.

---

## 17. Open Questions Deferred

This document does not settle:

- Final trace schema
- Final audit event schema
- Reason-code vocabulary
- Audit storage/retention model
- Tamper-evident audit design
- SIEM/export integrations
- Compliance mapping
- Privacy/data minimization policy
- Safety-system audit requirements
- Gateway fail-open/fail-closed behavior on audit failure
- Adapter evidence schema
- Identity evidence schema

Naming these here is intentional, for the same reason [Section 16 of the evaluation semantics document](operation-aware-evaluation-semantics.md#16-open-questions-deferred) named the topics that document did not settle: it establishes that this PR is aware of what remains open and is not silently deciding those questions by omission. Each should be addressed in its own later architecture work, informed by the evidence model this document defines.
