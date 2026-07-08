# Operation-Aware Authorization Model

This document defines the conceptual model for the next phase of `basis-core`: an expansion from a basic subject/action/resource authorization kernel into an operation-aware OT authorization kernel. It describes why the model is expanding, the categories of context a richer decision needs to carry, and the boundaries that keep that expansion from compromising kernel isolation.

This is an architecture document, not an implementation specification. It defines conceptual categories, not JSON Schemas, Python types, or a policy language. The corresponding decision to adopt this direction is recorded in [ADR-0001](../adr/0001-operation-aware-ot-authorization.md). Cross-references: [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md), [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md), [`docs/architecture/action-vocabulary.md`](action-vocabulary.md), [`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Why the Model Is Expanding

`basis-core` v0.1.0 established a deterministic authorization kernel evaluating a simple `DecisionRequest` — subject, action, resource — against policy and producing a `DecisionResponse`. That kernel is intentionally small: [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) defines the isolation rules that keep it protocol-agnostic, identity-provider-agnostic, and enforcement-agnostic, and those rules are not being revisited here.

What v0.1.0 does not yet represent is the operational context that real OT authorization decisions depend on. Whether a request should be allowed frequently depends on more than which subject is asking to perform which action on which resource. It depends on where the resource sits (site, building, zone), what kind of device is involved, which protocol operation produced the request, whether the operation reads a value or changes physical state, and whether a safety or risk context should change the outcome. A kernel that evaluates only subject/action/resource cannot express policies that condition on any of this, which means that context either goes unenforced or gets encoded informally outside the kernel — in adapter logic, in gateway middleware, or in operator judgment — where it is neither deterministic nor auditable.

The expansion described here is additive: it enriches what a `DecisionRequest` can carry and what a `DecisionResponse` can explain, without changing what the kernel is for. `basis-core` still evaluates policy deterministically against structured input and returns a structured, auditable decision. It does not become aware of OIDC, JWTs, sessions, cookies, or OT protocols in the process — those remain the concerns of `basis-identity`, `basis-gateway`, and `basis-adapters` respectively, exactly as they are today.

---

## 2. Conceptual Evaluation Path

The evaluation path does not introduce new components or change which component owns which responsibility. It describes how context accumulates as a request moves through the existing component chain before reaching the kernel, and what happens after evaluation.

```text
Verified identity context
        ↓
Gateway request assembly
        ↓
Adapter-normalized OT operation context
        ↓
Operation-aware DecisionRequest
        ↓
basis-core deterministic evaluation
        ↓
DecisionResponse + trace + audit evidence
        ↓
Gateway enforcement
```

`basis-identity` produces verified identity context upstream of evaluation, as it does today. `basis-adapters` normalizes the protocol-specific operation into protocol-independent evidence, as it does today. `basis-gateway` is the boundary that assembles these inputs — identity context, normalized adapter evidence, and any additional request-level context — into the richer `DecisionRequest` that `basis-core` evaluates. `basis-core` remains the sole evaluator: it receives a structured request, applies policy, and returns a structured response. `basis-gateway` then enforces the outcome and emits its own audit evidence, as it does today.

The only conceptual change on this path is the width of the object that crosses the gateway-to-kernel boundary and the object that crosses back. Component responsibilities and dependency direction are unchanged from [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md).

---

## 3. Rich DecisionRequest Conceptual Fields

The categories below describe what an operation-aware `DecisionRequest` should be able to represent conceptually. They are categories, not a schema — field names, types, optionality, and validation rules are deliberately not specified here. That work belongs to `basis-schemas`, once this architecture has stabilized (see [Section 9](#9-migration-path)).

- **Subject identity** — the verified identifier of the requesting subject, produced upstream by `basis-identity` and carried unchanged through the gateway.
- **Subject attributes** — additional attributes about the subject relevant to policy (role, group membership, clearance level) needed for attribute-based rules.
- **Authority mode / identity source** — a reference to which identity authority mode (see [`docs/architecture/identity-authority-modes.md`](identity-authority-modes.md)) or provider produced the identity, carried as a reference rather than provider-specific detail.
- **Action** — the operational verb being requested, per the canonical vocabulary in [`docs/architecture/action-vocabulary.md`](action-vocabulary.md).
- **Resource** — the canonical resource identifier being acted upon.
- **Resource type** — the operational classification of the resource, per [`docs/glossary.md`](../glossary.md#resource-type).
- **Site / building / zone / area** — the physical or logical location context in which the resource sits, needed for site- and zone-scoped policy.
- **Device identity** — a reference to the specific device involved, distinct from the resource identifier when a resource is exposed through a device.
- **Device class** — the operational category of device (controller, sensor, actuator, gateway) independent of the specific device identity.
- **Protocol** — which field protocol produced the underlying operation, carried as evidence rather than as something the kernel interprets.
- **Protocol operation** — the protocol-native operation that the adapter normalized, preserved as evidence alongside the normalized action.
- **Normalized adapter evidence** — the adapter's normalization output more broadly, carried through so that a decision's rationale can be traced back to the protocol-level request that produced it.
- **Operation intent** — whether the operation is read-only, state-changing, or command/control-affecting, as a distinct concept from the action verb itself.
- **Safety context** — whether a safety-relevant mode or constraint applies to the resource or site at evaluation time.
- **Environment context** — operational environment conditions relevant to policy (e.g., maintenance mode, degraded connectivity) distinct from safety context.
- **Time context** — the time at which the request is evaluated, needed for time-window constraints.
- **Risk context** — a risk classification or score associated with the request, resource, or subject, where a deployment chooses to define one.
- **Correlation ID** — an identifier that ties this decision to the broader request chain across `basis-identity`, `basis-gateway`, and `basis-adapters`.
- **Request ID** — an identifier unique to this specific evaluation request, distinct from the correlation ID that spans the chain.
- **Policy version** — the policy bundle version the request expects to be evaluated against, for compatibility and audit purposes.

Several of these categories overlap with concepts already tracked informally in [`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md) (action, resource type, normalized adapter evidence). This document does not resolve those open questions; it records that an operation-aware `DecisionRequest` needs to represent all of these categories, however the final field boundaries are drawn.

---

## 4. Rich DecisionResponse Conceptual Fields

The categories below describe what an operation-aware `DecisionResponse` should be able to represent conceptually. As with Section 3, these are categories, not a schema.

- **Decision outcome** — `ALLOW`, `DENY`, or `NOT_APPLICABLE`, consistent with the existing kernel decision semantics.
- **Failure reason** — a structured reason distinguishing a substantive denial from an evaluation failure, consistent with the existing `FailureReason` contract.
- **Matched rules** — identifiers of the policy rules that determined the outcome.
- **Policy identifiers** — the policy bundle and version evaluated, distinct from the individual matched rules.
- **Evaluation trace** — a deterministic, structured record of how the evaluation proceeded, sufficient to reconstruct why the outcome was reached without exposing internal implementation detail.
- **Human-readable explanation** — a rendering of the trace and matched rules intended for an operator, distinct from the machine-readable trace itself.
- **Obligations or enforcement hints** — optional, carefully scoped signals that accompany a decision (for example, "log this decision at elevated severity") without directing how enforcement is carried out.
- **Audit evidence** — the evidence the kernel attaches to support the audit record `basis-gateway` ultimately emits.
- **Schema version** — the version of the `DecisionResponse` shape itself, for compatibility across kernel versions.

**Caution on obligations and enforcement hints.** This category is the one most likely to be misused. `basis-core` evaluates policy; it does not enforce decisions, and it must not become an enforcement engine through the back door of an increasingly prescriptive obligations field. An obligation or enforcement hint may describe *what the decision implies* (for example, a recommended audit severity, or a flag that a decision was reached under a safety-context override) but must not instruct `basis-gateway` or an adapter *how to act* in a way that encodes enforcement logic inside the kernel's output. If a future obligation category starts to resemble a command, that is a signal the design has crossed the boundary this document is trying to hold, and it should be treated as a boundary question requiring its own ADR, not resolved by adding a field.

---

## 5. Policy Model Direction

The current kernel's policy model evaluates subject/action/resource rules. An operation-aware kernel needs a policy model that can condition on the richer context described in Section 3, without yet committing to a full policy language. The capabilities a future policy model should support, conceptually:

- **Role-based rules** — rules keyed on subject role, as today.
- **Attribute-based rules** — rules keyed on subject attributes beyond role.
- **Operation-aware rules** — rules keyed on operation intent (read-only vs. state-changing vs. control-affecting), not only the action verb.
- **Resource matching** — rules that match on resource type and identifier, as today, extended to the richer resource context.
- **Site/zone constraints** — rules scoped to a site, building, zone, or area.
- **Device-class constraints** — rules scoped to a class of device rather than an individual device identity.
- **Protocol/action constraints** — rules that can, where operationally meaningful, condition on the protocol or protocol operation evidence carried in the request, without making the policy model protocol-specific in the way Section 1 rules out.
- **Time-window constraints** — rules that apply only within defined time windows.
- **Safety-mode constraints** — rules that condition on the safety context described in Section 3.
- **Deny-overrides or documented combining semantics** — an explicit, documented rule for how multiple matching rules combine into a single outcome.
- **Deterministic rule ordering** — a defined, stable ordering for rule evaluation so that outcomes do not depend on incidental rule authoring order.
- **Stable reason codes** — a controlled vocabulary of reason codes attached to outcomes, analogous in spirit to the action vocabulary's governance model.
- **Policy bundle metadata** — versioning and provenance information carried with a policy bundle.
- **Policy validation** — the ability to validate a policy bundle for internal consistency before it is loaded for evaluation.
- **Schema versioning** — version identifiers on the policy format itself, per [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md).
- **Compatibility tests** — regression tests that verify policy evaluation outcomes are stable across kernel version increments, per the compatibility testing expectations in [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md).

This document does not define a policy language. It records the capabilities the policy model must eventually support so that the `DecisionRequest`/`DecisionResponse` expansion in Sections 3 and 4 is not designed against a policy model that cannot use the context it carries.

---

## 6. Evaluation Semantics to Define in Later PRs

The following semantics are necessary for an operation-aware kernel but are deliberately out of scope for this document. Each requires dedicated architectural analysis, in most cases because getting it wrong has compatibility or safety consequences that a single PR should not decide as a side effect:

- Default deny behavior under the expanded model
- `NOT_APPLICABLE` behavior and when it applies versus `DENY`
- Deny precedence when multiple rules match with conflicting outcomes
- Conflict resolution beyond simple deny precedence
- Rule ordering guarantees
- Condition evaluation semantics (how individual rule conditions are evaluated and combined)
- Missing context behavior — what the kernel does when a request omits a category described in Section 3
- Unknown action or resource type behavior
- Schema version compatibility across `DecisionRequest`/`DecisionResponse` versions
- Deterministic trace output format and stability guarantees
- Safe error handling for evaluation failures under the richer model

Naming these here is intentional: it establishes that this PR is aware they exist and is not silently deciding them by omission. Each should be addressed in its own architecture PR, informed by the categories this document defines.

---

## 7. Audit and Trace Direction

The future audit and evaluation evidence model should be able to represent, conceptually:

- **Decision request hash** — a stable hash of the evaluated request, for audit correlation without duplicating the full request body.
- **Policy bundle version** — the version of the policy bundle in effect at evaluation time.
- **Matched rule IDs** — the specific rules that determined the outcome.
- **Reason codes** — the stable reason code vocabulary referenced in Section 5.
- **Evaluation trace** — the same trace concept described in Section 4, as it flows into audit evidence.
- **Normalized adapter evidence reference** — a reference to the adapter evidence associated with the decision, rather than a duplicate copy, consistent with existing audit-is-evidence principles.
- **Identity source reference** — a reference to the authority mode or provider that produced the subject's identity, consistent with Section 3.
- **Correlation ID propagation** — the same correlation ID described in Section 3, carried through into the audit record.
- **Safe redaction rules** — rules for what must be redacted from audit evidence (raw credentials, tokens, key material) consistent with the redaction practice already established in `basis-identity`.
- **No secrets in audit** — an explicit constraint, not an assumption: audit evidence must not carry secrets regardless of how convenient it would be to include them for debugging.

Detailed audit schema work — field names, types, storage format, retention — belongs to later architecture and `basis-schemas` PRs. This section records the categories the future schema needs to cover, consistent with the audit principles already established in [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md) and [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md).

---

## 8. Boundary Ownership

The operation-aware model does not change the responsibility split established in [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md) and [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md). It is restated here explicitly because the richer `DecisionRequest`/`DecisionResponse` categories in Sections 3 and 4 could, if implemented carelessly, tempt a component to take on a responsibility that belongs elsewhere.

**basis-identity** establishes identity context and/or BASIS-local identity tokens. It does not evaluate authorization and does not normalize OT protocols.

**basis-gateway** verifies caller identity, assembles requests — including the richer operation-aware context described in Section 3 — enforces decisions, and emits gateway audit evidence. It composes the `DecisionRequest`; it does not evaluate it.

**basis-adapters** normalizes protocol operations and preserves protocol evidence. It never authorizes, and the operation-aware model does not create a path for it to start doing so.

**basis-core** evaluates policy deterministically and emits decision metadata and audit evidence. It never authenticates, never talks to OT protocols, and never enforces transport behavior. The richer context it evaluates arrives fully assembled; it does not reach out to any other component to gather it.

**basis-schemas** publishes stable shared contracts after this architecture stabilizes. It does not exist yet as a populated repository for these contracts; see [Section 9](#9-migration-path).

**basis-console** visualizes and explains decisions, traces, and training-mode flows. It does not evaluate policy and does not enforce decisions.

None of these assignments are new. They are the same assignments in [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md), restated against the specific categories this document introduces so that the expansion does not quietly redraw them.

---

## 9. Migration Path

```text
basis-core v0.1.0
  simple DecisionRequest / DecisionResponse

Architecture vNext
  operation-aware conceptual model  (this document)

basis-schemas
  publish vNext request/response/policy/audit contracts

basis-core v0.2.0
  additive model expansion with compatibility protections

basis-gateway/adapters/console
  consume richer contracts incrementally
```

This document is the first step: it establishes the conceptual model before any schema or code exists for it. `basis-schemas` is the next dependency — the categories in Sections 3, 4, and 7 need to become concrete, versioned contracts before `basis-core` v0.2.0 can implement them. `basis-core` v0.2.0 is expected to be an additive expansion, consistent with the compatibility philosophy in [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md): existing `DecisionRequest`/`DecisionResponse` shapes and evaluation outcomes for v0.1.0-era requests should continue to evaluate correctly, not be replaced wholesale. `basis-gateway`, `basis-adapters`, and `basis-console` are expected to consume the richer contracts incrementally as each is published, not simultaneously and not as a coordinated breaking release.

Nothing in this migration path implies a timeline. Consistent with [`ROADMAP.md`](../../ROADMAP.md), inclusion here is a statement of architectural direction, not a commitment to build on any particular schedule.
