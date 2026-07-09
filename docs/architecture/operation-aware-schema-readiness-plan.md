# Operation-Aware Schema Readiness and Migration Plan

This document defines the schema readiness and migration plan for the operation-aware authorization architecture described in [ADR-0001](../adr/0001-operation-aware-ot-authorization.md) through [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md). Those four ADRs and their companion documents — [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md), [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md), [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md), and [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md) — defined the operation-aware model, its evaluation semantics, its trace and audit evidence categories, and its policy bundle and rule model. None of them defined how that architecture should move into `basis-schemas`. This document is that migration plan.

This is an architecture and planning document, not an implementation specification. It does not define final JSON Schema, Python classes, database schemas, or any other implementation-specific artifact. It does not choose a policy language. It does not create, modify, or schedule work in `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`. It defines an ordered, reviewable sequence for the next `basis-schemas` expansion, and the compatibility, ownership, and security expectations that sequence must satisfy. The corresponding decision to treat this plan as the architecture handoff into `basis-schemas` is recorded in [ADR-0005](../adr/0005-operation-aware-schema-readiness.md).

Cross-references: [`docs/architecture/basis-schemas.md`](basis-schemas.md), [`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Purpose

[ADR-0001](../adr/0001-operation-aware-ot-authorization.md) through [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md) created the architecture stack for operation-aware authorization: a richer conceptual `DecisionRequest`/`DecisionResponse`, deterministic evaluation semantics, a trace and audit evidence model, and a policy bundle and rule model. Each of those documents deliberately stopped short of schema work — they named categories and constraints, not field names, types, or serializations — so that the next phase, formalizing these categories as machine-readable contracts in `basis-schemas`, would have a stable architectural target rather than an open design problem.

This document defines:

```text
Which contracts should move into basis-schemas
What order they should move in
What each contract must preserve
What compatibility rules apply
What examples/test vectors should accompany the contracts
What must remain deferred
```

It is written for whoever authors the next `basis-schemas` expansion — the operation-aware vNext contracts — so that the publication work can proceed as an ordered sequence of reviewable PRs, each with a defined scope, a defined set of things it must preserve from the prior ADRs, and a defined compatibility posture, rather than as a single undifferentiated schema effort.

Ownership across the migration is unchanged from what ADR-0001 through ADR-0004 and [`docs/architecture/basis-schemas.md`](basis-schemas.md) already establish:

```text
Architecture defines.
Schemas publish.
Core evaluates.
Gateway enforces.
Adapters normalize.
Identity establishes identity context.
Console explains.
```

This document does not change that division of labor. It sequences the work that follows from it.

---

## 2. Inputs From Prior ADRs

Each prior ADR contributes a distinct category of input to schema readiness. This section summarizes what each one hands to the next `basis-schemas` expansion, so that the contract surfaces in [Section 4](#4-contract-surfaces-to-publish) can be read as formalizations of decisions already made, not new decisions being made here.

**ADR-0001 — context categories for request/response.** [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md) named the conceptual fields a richer `DecisionRequest` and `DecisionResponse` must be able to carry: subject identity, subject attributes, identity source/authority mode, action, resource, resource type, site/building/zone/area, device identity, device class, protocol, protocol operation, normalized adapter evidence reference, operation intent, safety context, environment context, time context, risk context, correlation ID, request ID, and policy version expectation on the request side; final decision, failure reason, matched/skipped rule references, policy identifiers, and trace/evidence references on the response side. Schema work must formalize these categories without narrowing, renaming, or silently reinterpreting them.

**ADR-0002 — evaluation semantics that schemas must not obscure.** [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md) defined default deny, `NOT_APPLICABLE` semantics, deny precedence, conflict resolution, rule ordering, condition evaluation, missing-context behavior, unknown action/resource/type behavior, schema version compatibility, deterministic trace output, and safe error handling. A schema is not free to represent these outcomes in a shape that makes any of them ambiguous — for example, a response schema that cannot distinguish `DENY` from an evaluation failure, or that cannot represent `NOT_APPLICABLE` as its own outcome, would obscure semantics ADR-0002 already settled.

**ADR-0003 — trace/audit/redaction/evidence categories.** [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md) defined the distinction between kernel-produced evaluation trace, durable audit evidence, and the runtime gateway audit event; rule-level evidence; request evidence; adapter and identity evidence references; redaction tiers; human-readable explanation; and reason codes. Schema work must preserve the trace/audit distinction, the redaction tiers, and the no-secrets rule as structural properties of the contracts, not as conventions left to each publisher's discretion.

**ADR-0004 — policy bundle/rule/condition/metadata/validation categories.** [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md) defined the policy bundle as the unit of distribution, versioning, and validation; policy scope; the rule concept and stable rule identifiers; `ALLOW`/`DENY` effects and their relationship to deny precedence; match criteria mirrored from the `DecisionRequest` categories; conditions and their determinism requirements; bundle and rule combining semantics; rule ordering; validation as a required pre-evaluation step; policy metadata and provenance; and reason codes in policy. Schema work must formalize bundle and rule structure without choosing a policy language, condition operator set, or final reason-code vocabulary — those remain deferred per [Section 14](#14-what-must-not-move-to-schemas-yet).

Taken together, these four inputs mean `basis-schemas` is formalizing architecture decisions already made. Where this plan lists a required property of a future contract, that property traces back to one of these four documents; this plan does not introduce new semantics of its own.

---

## 3. What Schema Readiness Means

Schema readiness is a conceptual bar, not a process checklist. A contract is ready to move into `basis-schemas` when all of the following are true:

- **Its purpose is architecturally defined.** The contract corresponds to a category named in ADR-0001 through ADR-0004, not to a shape invented during schema authoring.
- **Its owner is clear.** The ownership boundaries already established in [Section 16 of the policy/rule model](operation-aware-policy-rule-model.md#16-ownership-boundaries), [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership), and [Section 15 of the evaluation semantics document](operation-aware-evaluation-semantics.md#15-boundary-preservation) identify who publishes it and who consumes it.
- **Its compatibility expectations are known.** The contract's stability requirements can be stated against [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md) before a single field is finalized.
- **Its required examples/test vectors are known.** The categories of example and test vector the contract needs, per [Section 8](#8-required-examples) and [Section 9](#9-required-test-vectors) of this plan, are identifiable in advance.
- **Its redaction/security implications are understood.** Whether the contract can carry sensitive material, and what redaction classification applies, follows directly from [Section 10 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#10-redaction-and-secret-handling).
- **It does not force unresolved policy-language or runtime decisions.** Publishing the contract's shape must not require deciding a policy language, a condition operator set, or an evaluation algorithm ahead of the architecture work that owns those decisions.
- **It can be versioned without breaking existing consumers unnecessarily.** The contract's initial shape leaves room for additive evolution, consistent with the schema-version-compatibility expectations in [Section 12 of the evaluation semantics document](operation-aware-evaluation-semantics.md#12-schema-version-compatibility).

A contract that fails any of these tests is not ready and belongs in [Section 14](#14-what-must-not-move-to-schemas-yet) until it is.

---

## 4. Contract Surfaces To Publish

The following operation-aware contract surfaces should eventually be published in `basis-schemas`. These are conceptual names, not filenames — final schema file layout is an open question deferred in [Section 16](#16-open-questions-deferred), consistent with the existing `basis-schemas.md` charter, which itself stops at naming candidates rather than files (see [`docs/architecture/basis-schemas.md`](basis-schemas.md#3-what-belongs-in-basis-schemas)).

```text
OperationAwareDecisionRequest
OperationAwareDecisionResponse
PolicyBundle
PolicyRule
PolicyCondition
EvaluationTrace
TraceRuleEvidence
AuditEvidence
GatewayAuditEvent
AdapterEvidenceReference
IdentityEvidenceReference
ReasonCode
RedactionClassification
SchemaVersionMetadata
CompatibilityExample / TestVector
```

Each name corresponds to a category already defined in ADR-0001 through ADR-0004: `OperationAwareDecisionRequest`/`OperationAwareDecisionResponse` to [Sections 3–4 of the authorization model](operation-aware-authorization-model.md#3-rich-decisionrequest-conceptual-fields); `PolicyBundle`/`PolicyRule`/`PolicyCondition` to [Sections 2, 4, and 7 of the policy/rule model](operation-aware-policy-rule-model.md#2-policy-bundle-concept); `EvaluationTrace`/`TraceRuleEvidence` to [Sections 4–5 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#4-evaluation-trace-categories); `AuditEvidence`/`GatewayAuditEvent` to [Sections 9 and 14 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#9-gateway-enforcement-evidence); `AdapterEvidenceReference`/`IdentityEvidenceReference` to [Sections 7–8 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#7-adapter-evidence-reference); `ReasonCode` to [Section 12 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#12-reason-codes) and [Section 13 of the policy/rule model](operation-aware-policy-rule-model.md#13-reason-codes-in-policy); `RedactionClassification` to [Section 10 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#10-redaction-and-secret-handling); and `SchemaVersionMetadata`/`CompatibilityExample`/`TestVector` to the versioning discipline already established in [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md) and [`docs/architecture/basis-schemas.md`](basis-schemas.md#6-versioning-strategy).

---

## 5. Recommended Publication Order

The following ordered sequence is recommended for future `basis-schemas` PRs. Each PR is scoped narrowly enough for independent review; later PRs depend on earlier ones per [Section 6](#6-dependency-relationships).

### PR A — Shared Metadata and Vocabulary

Publish or extend shared metadata contracts: schema version metadata; contract identifiers; redaction classification; reason-code structure (not the final exhaustive vocabulary); and compatibility/test-vector structure, if appropriate at this stage.

Purpose:

```text
Establish shared building blocks before larger contracts depend on them.
```

### PR B — Evidence Reference Contracts

Publish conceptual support contracts: identity evidence reference; adapter evidence reference; request/reference hashes; correlation/request ID references.

Purpose:

```text
Let request, response, trace, and audit contracts reference evidence safely without embedding raw secrets or protocol payloads.
```

### PR C — Operation-Aware DecisionRequest

Publish the richer request contract. Must preserve the ADR-0001 categories: subject identity; subject attributes; identity source / authority mode; action; resource; resource type; site/building/zone/area; device identity; device class; protocol; protocol operation; normalized adapter evidence reference; operation intent; safety context; environment context; time context; risk context; correlation ID; request ID; policy version expectation.

Purpose:

```text
Define what the kernel may evaluate without making the kernel fetch context.
```

### PR D — Policy Bundle and Rule Contracts

Publish policy bundle/rule contracts. Must preserve the ADR-0004 concepts: bundle ID/version/schema version; scope; rules; effects; match criteria; conditions; reason codes; explanations; metadata/provenance; validation expectations.

Purpose:

```text
Define what policy can express before core implements vNext evaluation.
```

### PR E — DecisionResponse and EvaluationTrace

Publish response and trace contracts. Must preserve the ADR-0002 and ADR-0003 concepts: final decision; failure reason; matched/skipped rules; policy identifiers; evaluation trace; reason codes; deterministic ordering; missing/unknown context observations; safe explanation metadata.

Purpose:

```text
Make evaluation results explainable and testable.
```

### PR F — Audit Evidence and GatewayAuditEvent

Publish audit/evidence contracts. Must preserve the ADR-0003 concepts: kernel-side audit evidence; gateway enforcement evidence; request/response/trace references; identity evidence reference; adapter evidence reference; redaction metadata; the no-secrets rule; audit emission metadata.

Purpose:

```text
Separate decision evidence from runtime enforcement evidence while preserving end-to-end auditability.
```

### PR G — Compatibility Examples and Test Vectors

Publish canonical examples/test vectors covering: allow rule matched; deny rule matched; deny overrides allow; no allow rule matched → deny; no applicable bundle → not applicable; missing optional context; missing required context; unknown resource type; unsupported schema version; invalid policy bundle; trace redaction; gateway enforcement evidence.

Purpose:

```text
Protect semantics before implementation changes land in basis-core.
```

---

## 6. Dependency Relationships

```text
metadata / redaction / reason-code structure
        ↓
evidence references
        ↓
DecisionRequest
        ↓
PolicyBundle / PolicyRule
        ↓
DecisionResponse / EvaluationTrace
        ↓
AuditEvidence / GatewayAuditEvent
        ↓
Compatibility examples / test vectors
```

This map mirrors the PR sequence in [Section 5](#5-recommended-publication-order): each layer references the structures defined above it, so publishing out of order would force forward references to contracts that do not yet exist. Some contracts may be developed in parallel — for example, `PolicyBundle`/`PolicyRule` work (PR D) and the response/trace work (PR E) both depend only on PR A through PR C, and could proceed concurrently once those are published — but compatibility review should respect the dependency direction shown above rather than treat every PR as independent. A PR that introduces a forward reference to a not-yet-published contract is not ready for review.

---

## 7. Compatibility Rules

The following compatibility expectations apply to the schema phase, consistent with [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md):

- v0.1-era request/response behavior must remain stable. The existing subject/action/resource `DecisionRequest`/`DecisionResponse` contract continues to evaluate consistently; operation-aware contracts are an additive expansion, not a replacement.
- Additive fields must not change old decisions unless policy explicitly uses them, consistent with [Section 12 of the evaluation semantics document](operation-aware-evaluation-semantics.md#12-schema-version-compatibility).
- Unknown future fields must not silently alter evaluation. A consumer encountering a field it does not recognize must ignore it, not fail closed or open in an undocumented way.
- Schema version incompatibility must be deterministic. A version mismatch produces a defined, observable outcome, not silent misinterpretation.
- Reason-code changes are compatibility-sensitive, consistent with [Section 12 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#12-reason-codes) and [Section 13 of the policy/rule model](operation-aware-policy-rule-model.md#13-reason-codes-in-policy).
- Trace ordering is compatibility-sensitive, consistent with [Section 13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output).
- Policy bundle versions must be explicit, consistent with [Section 15 of the policy/rule model](operation-aware-policy-rule-model.md#15-compatibility-expectations).
- Deprecation must be explicit. No contract is retired implicitly by omission from a later schema revision.
- Examples/test vectors must accompany compatibility-sensitive changes. A change to a contract in this category is not reviewable without the corresponding update to [Section 8](#8-required-examples) and [Section 9](#9-required-test-vectors) material.

---

## 8. Required Examples

The following examples should accompany future schemas. These are pseudocode-level descriptions of what each example must illustrate, not final example JSON — this is not the schema PR.

- Minimal valid operation-aware request
- Rich operation-aware request
- Request with adapter evidence reference
- Request with identity evidence reference
- Allow response
- Deny response
- Not applicable response
- Evaluation failure response
- Policy bundle with allow rule
- Policy bundle with deny rule
- Policy bundle with missing-context behavior
- Trace with matched/skipped rules
- Audit event with gateway enforcement evidence
- Redacted explanation example

---

## 9. Required Test Vectors

The following test-vector categories should be defined alongside future schemas:

- Deterministic allow
- Deterministic deny
- Deny precedence
- Default deny
- Not applicable
- Invalid request
- Invalid policy bundle
- Unsupported schema version
- Missing optional context
- Missing required context
- Unknown action/resource/type
- Trace determinism
- Redaction behavior
- Reason-code stability

These test vectors should eventually be consumed by `basis-core`, `basis-gateway`, and possibly `basis-console` to prevent semantic drift — a shared fixture that fails is a visible compatibility signal, in the same way [`docs/architecture/basis-schemas.md`](basis-schemas.md#3-what-belongs-in-basis-schemas) already identifies compatibility snapshots as a candidate contract.

---

## 10. Ownership Matrix

```text
basis-architecture:
  owns semantic authority

basis-schemas:
  publishes machine-readable contracts

basis-core:
  consumes request/policy contracts and produces response/trace/evidence

basis-gateway:
  consumes request/response/audit contracts, assembles requests, emits gateway audit

basis-adapters:
  consume adapter evidence/reference contracts, produce normalized evidence

basis-identity:
  consumes/produces identity evidence/reference contracts

basis-console:
  consumes response/trace/audit/explanation contracts for visualization

basis-deploy:
  later packages compatible versions
```

This restates, for the operation-aware contract surfaces named in [Section 4](#4-contract-surfaces-to-publish), the same ownership model already established in [Section 16 of the policy/rule model](operation-aware-policy-rule-model.md#16-ownership-boundaries), [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership), [Section 15 of the evaluation semantics document](operation-aware-evaluation-semantics.md#15-boundary-preservation), and [Section 5 of `basis-schemas.md`](basis-schemas.md#5-ownership-model). No new ownership assignment is introduced here.

---

## 11. Boundary Rules For Schema Design

Future schemas must preserve the following boundaries:

- Schemas must not make `basis-core` identity-provider-aware.
- Schemas must not make `basis-core` protocol-aware.
- Schemas must not make `basis-core` an enforcement engine.
- Schemas must not make adapters policy evaluators.
- Schemas must not make console authoritative.
- Schemas must not include secrets in trace/audit artifacts.
- Schemas must distinguish trace from audit.
- Schemas must distinguish decision from enforcement.
- Schemas must preserve gateway ownership of enforcement facts.

Each of these restates a boundary already established elsewhere: kernel isolation in [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md); the trace-versus-audit and decision-versus-enforcement distinctions in [Section 2 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#2-trace-vs-audit) and [Section 9](operation-aware-trace-audit-evidence.md#9-gateway-enforcement-evidence); and the adapter/console non-authority already stated in [Section 16 of the policy/rule model](operation-aware-policy-rule-model.md#16-ownership-boundaries). A schema PR that would require violating one of these boundaries to represent a contract correctly has not found the right shape for that contract.

---

## 12. Redaction and Security Requirements

Future schemas must support the following security requirements:

- No raw bearer tokens.
- No raw JWTs.
- No refresh tokens.
- No cookies.
- No private keys.
- No passwords.
- No API keys.
- No client secrets.
- No unredacted sensitive payloads.
- Redaction classification must be represented, per [Section 10 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#10-redaction-and-secret-handling).
- Reference/hash patterns should be preferred over raw duplication — a contract should carry a reference to evidence, not the evidence's sensitive source material, consistent with the evidence-reference categories in [Section 4](#4-contract-surfaces-to-publish).
- Examples must not include realistic secrets. Example and test-vector material described in [Section 8](#8-required-examples) and [Section 9](#9-required-test-vectors) must use clearly synthetic values.

---

## 13. Migration From Architecture To Schemas To Core

```text
basis-architecture
  ADR-0001 through ADR-0004 + this readiness plan

basis-schemas
  publish operation-aware vNext contracts in ordered PRs

basis-core
  implement additive vNext support against published schemas

basis-gateway
  assemble richer requests and consume richer responses incrementally

basis-adapters
  emit richer evidence references incrementally

basis-console
  visualize richer trace/audit/explanation contracts incrementally
```

This migration is additive and compatibility-protected at every stage. `basis-core` v0.1.0 behavior is not withdrawn while vNext support is added; `basis-gateway`, `basis-adapters`, and `basis-console` each adopt richer contracts incrementally, at their own pace, rather than as a coordinated lockstep release. This is the same incremental-adoption posture [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md) already establishes for shared contracts generally.

---

## 14. What Must Not Move To Schemas Yet

The following remain explicitly deferred:

- Final policy language
- Condition operator language
- Policy authoring workflows
- Policy approval workflows
- Policy distribution topology
- Audit storage backend
- Audit retention policy
- SIEM/export mappings
- Compliance mappings
- Safety certification rules
- Gateway runtime fail-open/fail-closed policy
- Deployment packaging
- Topology discovery model
- AI/autonomous policy suggestions

Some of these — most notably the policy language and condition operator set — may require their own later architecture PRs before `basis-schemas` can formalize them, in the same way ADR-0004 deferred policy syntax pending this readiness plan. Others, such as audit storage backend and deployment packaging, are owned by different components entirely (`basis-deploy`, operational tooling) and are named here only so their absence from this plan is not mistaken for an oversight.

---

## 15. Success Criteria For Entering basis-schemas Work

This architecture phase is complete enough to begin the next `basis-schemas` expansion when:

- ADR-0001 through ADR-0004 are merged.
- This schema readiness plan is merged.
- Contract surfaces are identified (see [Section 4](#4-contract-surfaces-to-publish)).
- Publication order is identified (see [Section 5](#5-recommended-publication-order)).
- Compatibility expectations are identified (see [Section 7](#7-compatibility-rules)).
- Examples/test-vector categories are identified (see [Sections 8](#8-required-examples) and [9](#9-required-test-vectors)).
- Redaction/security requirements are identified (see [Section 12](#12-redaction-and-security-requirements)).
- Ownership boundaries are restated (see [Section 10](#10-ownership-matrix)).
- Unresolved items are explicitly deferred (see [Section 14](#14-what-must-not-move-to-schemas-yet)).

---

## 16. Open Questions Deferred

This document does not settle:

- Final schema file layout
- Exact schema version numbers
- Exact policy language
- Exact condition operator set
- Exact reason-code vocabulary
- Exact trace schema fields
- Exact audit storage format
- Exact package release cadence
- Whether schemas are published as one vNext release or multiple incremental releases

Naming these here is intentional, for the same reason [Section 18 of the policy/rule model](operation-aware-policy-rule-model.md#18-open-questions-deferred) and [Section 17 of the trace/audit evidence model](operation-aware-trace-audit-evidence.md#17-open-questions-deferred) named the topics those documents did not settle: it establishes that this plan is aware of what remains open and is not silently deciding those questions by omission. Each should be resolved during the `basis-schemas` expansion itself, or in its own later architecture work, informed by the readiness plan this document defines.
