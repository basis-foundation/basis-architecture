# Operation-Aware Policy Bundle and Rule Model

This document defines the conceptual policy bundle and rule model for the operation-aware authorization model described in [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md), the evaluation semantics defined in [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md), and the trace and audit evidence model defined in [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md). [Section 5 of the authorization model](operation-aware-authorization-model.md#5-policy-model-direction) named the capabilities a future policy model would need to support and deferred the design of that model to a later architecture PR. This is that PR.

This document defines policy model concepts, not final policy syntax. It is an architecture document, not an implementation specification. It defines conceptual bundle structure, rule structure, match criteria, conditions, combining semantics, validation, metadata, and compatibility expectations — not JSON Schema, Python types, database schemas, implementation-specific parser behavior, or a complete policy language. The corresponding decision to define this model before publishing policy contracts is recorded in [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md). Cross-references: [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md), [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md), [`docs/architecture/action-vocabulary.md`](action-vocabulary.md), [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md), [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md), [`docs/architecture/operation-aware-evaluation-orchestration.md`](operation-aware-evaluation-orchestration.md), [`docs/glossary.md`](../glossary.md).

See [`operation-aware-evidence-provenance-semantics.md`](operation-aware-evidence-provenance-semantics.md) for the narrower clarification governing when authored rule `reason_code` and `explanation` are projected into rule evidence, and how `matched` rules are treated differently from `not_matched`, `skipped`, and `error` rules.

---

## 1. Purpose

A richer `DecisionRequest` and deterministic evaluation semantics are not enough by themselves. [Section 3 of the authorization model](operation-aware-authorization-model.md#3-rich-decisionrequest-conceptual-fields) defined what a request can carry, and [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md) defined how the kernel reasons about that context once it has policy to evaluate it against. Neither document defines what that policy *is* — how it is packaged, identified, validated, and evaluated as a coherent whole rather than an unstructured pile of rules. Without that model, `basis-schemas` would have to invent policy structure at the same time it is trying to formalize contracts, and `basis-core` v0.2.0 would have to make policy-shape decisions under implementation pressure — exactly the risk ADR-0002 and ADR-0003 were written to avoid for evaluation semantics and evidence.

The purpose of this document is to define:

- What is a policy bundle?
- What is a rule?
- What can a rule match?
- What effect can a rule produce?
- What metadata must policies carry?
- How should policies be validated before evaluation?
- What compatibility surfaces must be protected?

**This document defines policy model concepts, not final policy syntax.** It does not choose a policy language such as Rego, Cedar, CEL, YAML-only, Python, or a custom DSL, and it does not define final JSON Schemas. Those are `basis-schemas` decisions, made once this conceptual model has stabilized, consistent with the same architecture-before-schema ordering [ADR-0001](../adr/0001-operation-aware-ot-authorization.md) and [ADR-0002](../adr/0002-operation-aware-evaluation-semantics.md) already established.

---

## 2. Policy Bundle Concept

A **policy bundle** is the unit of policy distribution, versioning, validation, and compatibility. It is the thing `basis-core` evaluates a `DecisionRequest` against, the thing `basis-gateway` loads and selects at runtime, and the thing compatibility tests are written against as a stable target.

Conceptual bundle categories:

- **Bundle identifier** — a stable identifier for the bundle, distinct from its version, so that successive versions of "the same bundle" can be recognized as such.
- **Bundle version** — a version identifier for this specific revision of the bundle's rule content.
- **Schema version** — the version of the policy bundle format itself, independent of the bundle's own content version, per [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md).
- **Policy owner / authority** — a reference to who authored or is accountable for the bundle's content, for provenance and review purposes.
- **Target scope** — the conceptual scope the bundle declares it applies to, defined fully in [Section 3](#3-policy-scope).
- **Rule set** — the collection of rules the bundle contains, defined in [Section 4](#4-rule-concept).
- **Metadata** — the descriptive and provenance information defined in [Section 12](#12-policy-metadata-and-provenance).
- **Compatibility metadata** — the version and compatibility-target information defined in [Section 15](#15-compatibility-expectations).
- **Validation status** — whether the bundle has passed the validation described in [Section 11](#11-policy-validation), and against what schema version.
- **Provenance** — where the bundle came from and how it reached the evaluating kernel, distinct from policy authorship.
- **Optional deprecation/replacement metadata** — whether the bundle is deprecated and, if so, what bundle identifier replaces it.

A policy bundle is **not necessarily a file format**. It is a conceptual unit — the boundary within which rules are versioned and validated together. A future `basis-schemas` release may define one or more concrete serializations of a bundle (a single file, a directory of files, a database-backed representation, a bundle assembled from multiple sources at load time); this document does not choose among those options, and nothing here should be read as assuming any one of them.

---

## 3. Policy Scope

**Policy scope** is what a bundle declares itself applicable to. A bundle should be able to declare scope conceptually along these dimensions:

- **Domain** — the operational domain the bundle governs (for example, a building-automation domain versus an industrial-control domain), where a deployment chooses to distinguish them.
- **Action vocabulary scope** — which actions, per [`docs/architecture/action-vocabulary.md`](action-vocabulary.md), the bundle's rules are meant to cover.
- **Resource type scope** — which resource types the bundle governs.
- **Site/building/zone scope** — the physical or logical location scope the bundle applies to, consistent with the site/zone context described in [Section 3 of the authorization model](operation-aware-authorization-model.md#3-rich-decisionrequest-conceptual-fields).
- **Device class scope** — which device classes the bundle's rules are meant to cover.
- **Environment/deployment scope** — whether the bundle applies to a specific environment (for example, production versus a staging or training deployment).
- **Identity authority mode scope** — whether the bundle assumes a particular identity authority mode, per [`docs/architecture/identity-authority-modes.md`](identity-authority-modes.md).
- **Protocol/evidence scope, if supported carefully** — whether the bundle conditions on protocol evidence, subject to the same protocol-agnostic caution the authorization model already applies to the kernel itself; a bundle may reference protocol evidence carried in the request, but that does not make the kernel protocol-aware, consistent with [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership).

Scope matters because it is what lets `NOT_APPLICABLE` (per [Section 5 of the evaluation semantics document](operation-aware-evaluation-semantics.md#5-not_applicable-semantics)) mean something precise rather than being inferred informally from the absence of a matching rule. The recommended principle:

```text
A bundle outside its declared scope should not pretend to deny or allow a request.
It should be semantically not applicable.
```

A bundle that governs building-automation zone access should not have an opinion, allow or deny, about a request scoped to a domain it does not declare coverage for. Whether "no bundle declares coverage" resolves to `NOT_APPLICABLE` or to default deny at a higher level is an evaluation-semantics question already settled in Sections 4 and 5 of the evaluation semantics document; this section only establishes that scope declaration is what makes that determination possible in the first place, rather than something the kernel has to guess from rule content.

---

## 4. Rule Concept

A **rule** is a deterministic unit of policy evaluation — the thing that a request is matched against, and the thing whose identifier appears in trace and audit evidence per [Section 5 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence).

Conceptual rule categories:

- **Rule identifier** — a stable identifier for the rule, unique at least within its owning bundle.
- **Effect** — the outcome the rule produces when it matches, defined in [Section 5](#5-rule-effects).
- **Target / match criteria** — what the rule matches against, defined in [Section 6](#6-match-criteria).
- **Conditions** — the deterministic tests the rule applies, defined in [Section 7](#7-conditions).
- **Reason code** — the stable, machine-readable code the rule emits when it matches, per [Section 13](#13-reason-codes-in-policy).
- **Explanation template or safe explanation reference** — a reference the rule carries so that a human-readable explanation can be derived from it later, per [Section 14](#14-explanation-text-and-templates).
- **Priority or ordering metadata, if supported** — optional metadata a future policy format may use for deterministic ordering, per [Section 10](#10-rule-ordering-and-priority).
- **Lifecycle metadata** — whether the rule is active, deprecated, or scheduled, and since/until information if the policy format supports it.
- **Source/provenance reference** — which bundle, and which authoring source within that bundle, the rule came from.

**Rule identifiers must be stable enough for trace, audit, compatibility tests, and operator explanations.** A rule identifier that changes across bundle revisions without an explicit rename breaks every downstream consumer that referenced it — a compatibility test asserting a specific rule matched, an audit record pointing back to the rule that produced a decision, and an operator explanation citing the rule by name all depend on the identifier meaning the same thing over time. This is the same stability requirement [Section 12 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#12-reason-codes) already places on reason codes, applied here to rule identifiers.

---

## 5. Rule Effects

Recommended rule effects, conceptually:

```text
ALLOW
DENY
```

Future effects such as advisory, warn, or log-only are out of scope unless separately governed by their own architecture work. Introducing an effect beyond ALLOW/DENY changes the combining semantics this document defines in [Section 9](#9-bundle-combining-and-rule-combining) and should not be added informally by a policy format implementation.

Important constraints on effects:

- **`DENY` must participate in deny precedence**, consistent with [Section 6 of the evaluation semantics document](operation-aware-evaluation-semantics.md#6-deny-precedence). A matching deny rule overrides any matching allow rule, unconditionally.
- **`ALLOW` grants only when no applicable deny rule matches.** An allow rule's effect is conditional on the absence of a matching deny, not an independent grant.
- **Rules should not encode gateway enforcement behavior.** A rule's effect describes an authorization outcome, not an instruction for how `basis-gateway` should carry out enforcement — the same caution the authorization model already places on obligations and enforcement hints in [Section 4](operation-aware-authorization-model.md#4-rich-decisionresponse-conceptual-fields).
- **Rules should not directly execute actions.** A rule is evaluated, not executed; it has no mechanism to act on the world, consistent with the kernel's evaluation-only boundary.
- **Rules should not mutate request context.** Evaluating a rule against a request must not change the context that other rules, or subsequent evaluation phases, see — consistent with the deterministic, side-effect-free evaluation already required by [Section 9 of the evaluation semantics document](operation-aware-evaluation-semantics.md#9-condition-evaluation).

---

## 6. Match Criteria

Match criteria describe what a rule can match against, conceptually. Final field names, types, and encodings belong to `basis-schemas`; this section names categories only.

- Subject identity
- Subject attributes
- Identity source / authority mode
- Action
- Resource identifier
- Resource type
- Site/building/zone/area
- Device identity
- Device class
- Protocol
- Protocol operation
- Operation intent
- Safety context
- Environment context
- Time context
- Risk context

This list mirrors the `DecisionRequest` categories in [Section 3 of the authorization model](operation-aware-authorization-model.md#3-rich-decisionrequest-conceptual-fields) deliberately: a rule can only match what a request can carry, and a policy model that could match against something a request cannot represent would be meaningless. The reverse is not required — a rule is not obligated to match against every category a request can carry, only to be capable of matching against any of them where policy authoring needs it.

---

## 7. Conditions

A **condition** is a deterministic test over request context — the mechanism by which a rule's match criteria are actually evaluated against a specific request.

Recommended principles:

- **Conditions must be explicit.** A condition's test must be stated, not inferred from the shape of surrounding policy.
- **Conditions must be deterministic.** The same condition evaluated against the same request and policy bundle version must produce the same result every time, consistent with [Section 9 of the evaluation semantics document](operation-aware-evaluation-semantics.md#9-condition-evaluation).
- **Conditions must not fetch external data at evaluation time.** A condition evaluates the context it is given; it does not reach out to another system to gather more, consistent with the kernel's boundary in [Section 2 of the evaluation semantics document](operation-aware-evaluation-semantics.md#2-evaluation-inputs).
- **Conditions must not have side effects.** Evaluating a condition must not change anything, in the request, the bundle, or any external system.
- **Conditions must distinguish match / no match / error**, the same three-way outcome already required by [Section 9 of the evaluation semantics document](operation-aware-evaluation-semantics.md#9-condition-evaluation).
- **Conditions must not silently coerce incompatible types.** A comparison between incompatible types is an evaluation error or a defined no-match, not an implicit coercion.
- **Conditions must not treat missing context as truthy.** A condition referencing context the request does not carry does not default to matching, consistent with [Section 10 of the evaluation semantics document](operation-aware-evaluation-semantics.md#10-missing-context-behavior).
- **Conditions must be traceable.** A condition's outcome must be representable in rule-level trace evidence, per [Section 5 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence).

This document does not define a full operator language — no enumeration of comparison operators, set operators, or boolean combinators is specified here. That is deferred to the eventual policy format design, informed by the semantic requirements above.

---

## 8. Operation-Aware Policy Examples

The following examples are conceptual and non-normative. They illustrate the kind of policy the model in this document must be able to express; they are not proposed syntax, and no future policy format is obligated to resemble their phrasing.

- Allow operators to read AHU telemetry in assigned zones.
- Deny writes to safety-critical setpoints outside maintenance windows.
- Deny vendor access to control-affecting operations unless time-boxed and scoped to a site.
- Allow auditors to browse audit evidence but not execute control actions.
- Deny protocol operations normalized as state-changing when safety mode is active.

Each example combines match criteria from [Section 6](#6-match-criteria) (role, resource type, zone, time context, safety context, operation intent) with an effect from [Section 5](#5-rule-effects). None of them presuppose a specific policy language, and none should be read as a commitment to specific field names.

---

## 9. Bundle Combining and Rule Combining

Recommended combining direction:

- **Bundle scope determines applicability.** A bundle whose declared scope does not cover a request does not contribute candidate rules for that request, consistent with [Section 3](#3-policy-scope).
- **Candidate rules are selected from applicable bundles.** Only rules from bundles whose scope covers the request are considered.
- **Matching deny rules override matching allow rules.** This restates [Section 6 of the evaluation semantics document](operation-aware-evaluation-semantics.md#6-deny-precedence) at the policy-model level: deny precedence is unconditional.
- **If one or more allow rules match and no deny rules match, the result is `ALLOW`.**
- **If no allow rules match inside an applicable policy scope, the result is `DENY`.** This restates default deny ([Section 4 of the evaluation semantics document](operation-aware-evaluation-semantics.md#4-default-deny)) at the policy-model level.
- **If no policy bundle applies, the result is `NOT_APPLICABLE`.** This restates [Section 5 of the evaluation semantics document](operation-aware-evaluation-semantics.md#5-not_applicable-semantics) at the policy-model level.
- **An invalid policy bundle produces an evaluation failure, not an `ALLOW`/`DENY` decision**, consistent with [Section 14 of the evaluation semantics document](operation-aware-evaluation-semantics.md#14-safe-error-handling).
- **Ambiguous conflicts must be surfaced in trace metadata**, consistent with [Section 7 of the evaluation semantics document](operation-aware-evaluation-semantics.md#7-conflict-resolution). A policy model that produces an ambiguous combining case must not resolve it silently.

This section does not design final algorithm details beyond this semantic contract. It restates, at the policy-model level, combining behavior already established at the evaluation-semantics level, so that a future policy format's combining logic has a defined target rather than having to re-derive these rules from the evaluation semantics document independently.

Combining semantics remain owned by this document and by `policy`, not by the `evaluation` kernel subpackage [ADR-0006](../adr/0006-evaluation-orchestration-layer.md) introduces. `evaluation`'s future `engine` module orchestrates *invoking* this semantics — sequencing candidate identification, condition evaluation, and outcome combination as phases — without defining a competing combining behavior. See [Section 6 of the evaluation orchestration document](operation-aware-evaluation-orchestration.md#6-layer-ownership-clarification) for the full distinction between policy-owned semantics and evaluation-owned orchestration.

---

## 10. Rule Ordering and Priority

Recommended direction, consistent with [Section 8 of the evaluation semantics document](operation-aware-evaluation-semantics.md#8-rule-ordering):

- **Evaluation must not depend on incidental file order, map order, or runtime iteration order.** A rule set that produces different outcomes depending on how it happened to be loaded or serialized is not deterministic in the sense this model requires.
- **If priority is supported later, priority semantics must be explicit.** A future policy format that introduces rule priority must define what priority means and how it interacts with deny precedence — priority does not override deny precedence by default, and any format that lets it do so must say so explicitly and treat that as its own architectural decision.
- **Stable rule identifiers may be used as a deterministic tie-breaker.** Where no other ordering signal resolves a tie, the rule identifier itself — already required to be stable per [Section 4](#4-rule-concept) — is an acceptable deterministic fallback.
- **Declaration order should only matter if the future policy format explicitly defines it as stable.** A policy format may choose to treat the order rules appear in a bundle as meaningful, but only if it says so as part of its own specification, not as an incidental consequence of how a parser happens to iterate.
- **Trace output should report ordering deterministically**, consistent with the deterministic trace output requirements in [Section 13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output) and the rule evidence requirements in [Section 5 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence).

---

## 11. Policy Validation

Validation is a required pre-evaluation step. A policy bundle must be validated before it is used to evaluate any request. Conceptual validation categories:

- Schema/version validation
- Duplicate rule identifier detection
- Invalid effect detection
- Invalid action format detection
- Invalid resource/match field detection
- Unsupported condition/operator detection
- Invalid scope declaration detection
- Incompatible version detection
- Unsafe explanation/redaction detection
- Reason-code validation, if reason codes are governed

```text
Invalid policy bundles must not produce ALLOW decisions.
```

This restates, at the policy-model level, the same principle [Section 14 of the evaluation semantics document](operation-aware-evaluation-semantics.md#14-safe-error-handling) already establishes: an evaluation failure is a distinct outcome category from `ALLOW`, `DENY`, and `NOT_APPLICABLE`, and a bundle that fails validation must produce that failure category, never a default or fallback `ALLOW`. Validation happening before evaluation begins is what makes this guarantee possible — a bundle that fails validation should never reach the rule-evaluation phase described in [Section 3 of the evaluation semantics document](operation-aware-evaluation-semantics.md#3-evaluation-phases) in the first place.

---

## 12. Policy Metadata and Provenance

Conceptual policy metadata categories:

- Bundle ID
- Bundle version
- Schema version
- Author/source reference
- Approval/review reference, if available
- Created/updated timestamp
- Compatibility target
- Deprecation status
- Replacement bundle reference
- Policy owner
- Environment/site scope

This document does not define a release process or governance process for policy authoring. It identifies the metadata a bundle needs to carry so that auditability and compatibility are possible — who owns a bundle, what it is compatible with, and whether it has been superseded — without specifying how that metadata gets populated, reviewed, or approved. That process work, if and when it is needed, is a separate concern from the metadata categories themselves.

---

## 13. Reason Codes in Policy

Recommended direction, consistent with [Section 12 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#12-reason-codes):

- Rules may emit stable reason codes.
- Reason codes should be machine-readable.
- Reason codes should be governed as compatibility surfaces, analogous to the action vocabulary's governance model in [`docs/architecture/action-vocabulary.md`](action-vocabulary.md).
- Reason codes should be usable by trace, audit, tests, and console explanations.
- The final reason-code vocabulary is deferred.

Examples only, not final:

```text
ROLE_ALLOWED_OPERATION
ZONE_SCOPE_MISMATCH
SAFETY_MODE_DENY
MAINTENANCE_WINDOW_REQUIRED
VENDOR_SCOPE_EXPIRED
NO_MATCHING_ALLOW_RULE
```

These examples are illustrative of the kind of distinction (why a rule matched or did not) the eventual vocabulary needs to preserve; they are not a commitment to these specific codes, and they overlap deliberately with — rather than replace — the reason-code examples already given in [Section 12 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#12-reason-codes).

---

## 14. Explanation Text and Templates

Recommended direction:

- **Explanations must be derived from rule metadata, reason codes, and trace evidence.** An explanation renders what already happened; it does not independently reason about the request.
- **Explanations must not invent facts**, consistent with [Section 11 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#11-human-readable-explanation).
- **Explanations must be safe after redaction.** Whatever a rule's explanation template references must respect the redaction tiers defined in [Section 10 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#10-redaction-and-secret-handling).
- **Policy-authored explanation templates must not expose secrets.** A rule author writing an explanation template must not be able to embed credential material, tokens, or other secrets into the resulting explanation text.
- **Human-readable explanations are not authoritative.** The machine-readable decision and trace are authoritative; explanation text is a rendering, not a second source of truth, consistent with the trace-versus-explanation distinction in [Section 11 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#11-human-readable-explanation).

This document does not design a template language. It establishes the constraints a future template mechanism must satisfy, not its syntax.

---

## 15. Compatibility Expectations

Recommended direction, consistent with [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md):

- Policy bundle schema versions must be explicit.
- Additive request context should not change old decisions unless policy uses it, consistent with [Section 12 of the evaluation semantics document](operation-aware-evaluation-semantics.md#12-schema-version-compatibility).
- Existing policy bundles should continue to evaluate consistently under compatible kernel versions.
- Rule IDs and reason codes should be stable once published, consistent with [Section 4](#4-rule-concept) and [Section 13](#13-reason-codes-in-policy).
- Deprecation should be explicit, using the deprecation/replacement metadata described in [Section 12](#12-policy-metadata-and-provenance).
- Compatibility snapshots should cover both decisions and trace behavior, consistent with the compatibility testing expectations already established for evaluation semantics and trace.
- Breaking changes must be governed through architecture/ADR and schema versioning, not introduced informally through a policy format's own evolution.

---

## 16. Ownership Boundaries

```text
basis-architecture:
  defines policy model semantics and governance direction

basis-schemas:
  later publishes machine-readable policy contracts

basis-core:
  validates and evaluates policy bundles deterministically

basis-gateway:
  loads/selects/configures policy bundles as runtime input but does not define policy semantics

basis-adapters:
  do not author, interpret, or enforce policy

basis-identity:
  does not author or evaluate policy

basis-console:
  may visualize policy, explanations, and traces; does not evaluate policy
```

None of these assignments are new. They are the same boundary ownership already established in [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership) and [Section 15 of the evaluation semantics document](operation-aware-evaluation-semantics.md#15-boundary-preservation), restated against the policy-model concepts this document introduces. Policy loading, distribution, storage, and deployment are separate concerns from policy semantics: this document defines what a policy bundle and rule *mean*, not how a bundle is fetched, stored, or deployed by `basis-gateway` or any future policy-distribution tooling.

---

## 17. Security Considerations

- **Policies are sensitive configuration.** A policy bundle determines what is allowed and denied across an OT environment; it should be handled with the same care as any other security-critical configuration artifact.
- **Policy changes are high-impact operational changes.** Changing a bundle's rules can change access outcomes across every request it governs, which is a materially larger blast radius than most configuration changes.
- **Unsafe policy examples/explanations can leak operational details.** An explanation template or authored example that embeds specific site, device, or subject detail carelessly can leak operational information to anyone who can view an explanation, consistent with the redaction discipline in [Section 10 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#10-redaction-and-secret-handling).
- **Invalid policy must not default to allow**, restating [Section 11](#11-policy-validation) as a security property, not only a correctness property.
- **Policy provenance matters.** Knowing who authored and approved a bundle, per [Section 12](#12-policy-metadata-and-provenance), is a precondition for trusting it.
- **Policy bundle tampering must be detectable in future deployment work.** This document does not design tamper-evidence or signing; it notes that a bundle whose integrity cannot be verified is a future risk this model does not yet close.
- **Policy validation failures should be visible.** A bundle that fails validation silently, with no observable signal, creates the same kind of gap [Section 15 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#15-audit-failure-behavior) already flags for audit failure.
- **Policy must not contain secrets.** A rule, condition, or explanation template must not embed credential material as part of its authored content.
- **Policy should not embed raw credentials, tokens, passwords, keys, or sensitive operational payloads.** This restates the no-secrets principle in concrete terms, consistent with [Section 10 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#10-redaction-and-secret-handling).

This document does not design signing or tamper-evidence. That is explicitly deferred (see [Section 18](#18-open-questions-deferred)).

---

## 18. Open Questions Deferred

This document does not settle:

- Final policy language
- Final policy JSON Schema
- Rule condition operator set
- Final reason-code vocabulary
- Policy signing/tamper-evident packaging
- Policy distribution model
- Policy storage model
- Policy authoring UI
- Policy approval workflow
- Policy migration tooling
- Policy simulation UX
- Safety certification implications
- Multi-bundle hierarchy/federation
- Tenant/site-level policy delegation

Naming these here is intentional, for the same reason [Section 17 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#17-open-questions-deferred) named the topics that document did not settle: it establishes that this PR is aware of what remains open and is not silently deciding those questions by omission. Each should be addressed in its own later architecture work, informed by the policy bundle and rule model this document defines.
