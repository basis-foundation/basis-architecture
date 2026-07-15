# Operation-Aware Evaluation Orchestration Layer

This document defines the conceptual composition layer that resolves a dependency conflict discovered while sequencing `basis-core`'s operation-aware implementation work: operation-aware trace assembly must consume both policy-owned evaluation facts (defined in [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md) and adopted in [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md)) and audit-owned trace models (defined in [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md) and adopted in [ADR-0003](../adr/0003-operation-aware-trace-audit-evidence.md)), and no existing kernel subpackage may legally depend on both without crossing a dependency direction [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) already establishes as non-negotiable.

This is an architecture document, not an implementation specification. It defines a conceptual kernel layer, its dependency rules, and its ownership boundaries — not Python module signatures, class hierarchies, or function contracts. The corresponding decision to introduce this layer is recorded in [ADR-0006](../adr/0006-evaluation-orchestration-layer.md). Cross-references: [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md), [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md), [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md), [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Purpose

`basis-core`'s kernel subpackages are deliberately isolated from one another along a single downward dependency direction — `domain` → `decisions` → `policy` / `audit` → `enforcement` — so that policy evaluation semantics and audit evidence models can each stabilize independently, per [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#import-boundary-summary). That isolation is a feature, not an oversight: it is what lets `policy` and `audit` be reasoned about, tested, and evolved without either acquiring the other's concerns.

The operation-aware model this document's prerequisites establish requires a step that isolation alone cannot satisfy. Producing an `EvaluationTrace` and a `TraceRuleEvidence` record — audit-owned contracts, per [Section 5 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence) — requires reading `RuleConditionEvaluation` and `ConditionEvaluation` facts and rule metadata that policy evaluation produces, per [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md). Some kernel subpackage must be able to read both without either `policy` importing `audit` or `audit` importing `policy`, because both of those directions are prohibited and neither prohibition should be relaxed (see [Section 9](#9-alternatives-considered)).

This document names that subpackage — `evaluation` — and defines what it owns, what it may import, and what it must never become.

---

## 2. The Composition Problem

Stated precisely, the conflict is this:

- `policy` owns rule-level and bundle-level evaluation facts: matched rules, skipped rules, condition outcomes, and combining results, per [Section 9 of the policy/rule model](operation-aware-policy-rule-model.md#9-bundle-combining-and-rule-combining).
- `audit` owns the trace and evidence contracts that explain a decision after the fact: `EvaluationTrace`, `TraceRuleEvidence`, and the evidence categories in [Section 6 through Section 11 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md).
- Constructing a complete `EvaluationTrace` requires combining both: the trace must report which rules were candidates, matched, or skipped, and why — information that only exists as a policy-evaluation fact — using the audit-owned shape that trace consumers depend on.
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#import-boundary-summary) permits `policy` to import `domain` and `decisions` only, and permits `audit` to import `domain` and `decisions` only. Neither may import the other. This document does not relax either rule (see [Section 4](#4-dependency-rules)).

No kernel subpackage that existed prior to this decision may perform this composition legally. `policy` cannot construct `TraceRuleEvidence` without importing `audit` contracts it wasn't allowed to import. `audit` cannot derive trace content from policy facts without importing `policy` internals it wasn't allowed to import. The composition has to happen somewhere; before this decision, every somewhere was a boundary violation.

This is a missing architectural responsibility, not a misplaced file. No document prior to this one assigned ownership of policy-fact-to-trace-evidence composition to any kernel subpackage, because no kernel subpackage was the legal owner of that composition.

---

## 3. The Evaluation Layer

`basis-core` introduces `evaluation` as a first-class kernel subpackage, implemented in the future at `src/basis_core/evaluation/`.

Its responsibility is pure, deterministic orchestration of already-defined `domain`, `decisions`, `policy`, and `audit` contracts into complete kernel evaluation behavior. It is the composition boundary [Section 2](#2-the-composition-problem) shows is currently missing.

The evaluation layer is not a policy-definition layer — it does not define, and does not reimplement, rule effects, match criteria, conditions, or combining semantics; those remain owned and executed by `policy`, per [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md). Authorization semantics are executable policy-layer behavior, not merely prose that `evaluation` reimplements. The evaluation layer composes and invokes those semantics; it does not become a second policy engine. [Section 6](#6-layer-ownership-clarification) states the full ownership split, and the table in that section is the authoritative reference for which subpackage owns which concern.

The glossary's **Policy Engine** entry describes the authorization-evaluation capability at a system level — the logical evaluation authority a conceptual diagram shows, per [`docs/architecture/reference-vs-implementation.md`](reference-vs-implementation.md). Introducing `evaluation` does not add a second implementation of that capability: `policy` implements the executable semantics the system-level "policy engine" role depends on, and `evaluation` sequences and invokes them within `basis-core`. Neither subpackage is independently "the policy engine" in the glossary's sense; together, within `basis-core`, they fulfill it.

The evaluation layer is not an audit-persistence layer — it does not write audit evidence anywhere durable, and it does not decide audit failure policy; those remain gateway/runtime concerns, per [Section 15 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#15-audit-failure-behavior).

The evaluation layer is not enforcement — it does not decide whether a decision is applied, does not talk to protocols or transports, and does not emit gateway audit events; those remain owned by `enforcement` and `basis-gateway`, per [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md).

The evaluation layer is not an adapter boundary — it has no knowledge of protocol normalization and does not implement adapter contracts.

The evaluation layer performs no external I/O of any kind. [Section 7](#7-kernel-purity-requirements) states this precisely.

---

## 4. Dependency Rules

This section restates the dependency model in [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#import-boundary-summary) with `evaluation` added, and changes nothing else. The kernel's subpackage dependencies form a directed acyclic import graph, not a linear chain — the block below states each layer's complete allowed and forbidden imports; it is not a claim that every layer imports the layer immediately adjacent to it.

```text
domain
    imports nothing from within basis-core

decisions
    may import: domain

policy
    may import: domain, decisions
    must not import: audit, evaluation, enforcement

audit
    may import: domain, decisions
    must not import: policy, evaluation, enforcement

evaluation
    may import: domain, decisions, policy, audit
    must not import: adapters, enforcement

adapters
    may import: domain, decisions, policy contracts
    must not import: audit, evaluation, enforcement

enforcement
    may import: domain, decisions, policy, audit, evaluation, adapters
    (top layer — nothing may import from it)
```

Three properties of this table matter more than any individual line:

- **`policy` and `audit` remain mutually isolated.** Neither the introduction of `evaluation` nor anything else in this document creates a path — direct or transitive — from `policy` to `audit` or from `audit` to `policy`. A subpackage that needs facts from both must be a subpackage that is not `policy` and is not `audit`. `evaluation` is that subpackage; it does not make `policy` and `audit` importable from one another, because `evaluation` importing both is not the same relationship as either importing the other.
- **`evaluation` composes without being composed into.** `adapters` and `enforcement` retain their existing allowed imports; `evaluation` does not require `adapters` to import it, and nothing below `evaluation` in this table may import `evaluation`. `enforcement` may import `evaluation` because `enforcement` is the runtime boundary that invokes the deterministic kernel and needs its result; that is an existing pattern (`enforcement` already imports `domain`, `decisions`, `policy`, `audit`, and `adapters`) and not a new kind of exception.
- **No existing permission was widened.** `policy`'s and `audit`'s allowed-import lists are unchanged from [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#import-boundary-summary). `adapters`' allowed-import list is unchanged. The only new rows in the table are `evaluation`'s own allowed and forbidden imports.

---

## 5. Evaluation Layer Responsibilities

The future implementation package is `src/basis_core/evaluation/`, with an `operation_aware/` subpackage holding three future modules. These are architectural placements, not implementation in this document or in [ADR-0006](../adr/0006-evaluation-orchestration-layer.md); no function signatures, class hierarchies, or data structures are specified here.

**`trace_assembly`** — future responsibility:

- translate already-evaluated policy facts into bounded trace evidence
- construct `TraceRuleEvidence`, per [Section 5 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#5-rule-evaluation-evidence)
- construct `EvaluationTrace`, per [Section 13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output)
- preserve the deterministic ordering [Section 13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output) requires
- preserve the boundedness and redaction rules [Section 3](operation-aware-trace-audit-evidence.md) through [Section 11](operation-aware-trace-audit-evidence.md) of the trace and audit evidence model establish
- perform no policy evaluation of its own
- derive no authorization result of its own

**`engine`** — future responsibility:

- choose and preserve the documented stage sequence named in [Section 3 of the evaluation semantics document](operation-aware-evaluation-semantics.md#3-evaluation-phases) — request/policy validation, candidate rule identification, condition evaluation, outcome combination, and reason/trace production — by invoking, in that order, the policy-owned operations that implement each phase
- invoke policy-owned evaluators for selector matching, condition evaluation, and condition-result aggregation, carrying their typed results forward; it does not reimplement them
- invoke the policy-owned effect-combination operation — deny precedence, default deny, and `NOT_APPLICABLE` determination, per [Section 9 of the policy/rule model](operation-aware-policy-rule-model.md#9-bundle-combining-and-rule-combining) — and carry its typed result into subsequent evaluation stages
- propagate typed results between stages for `trace_assembly` and `response_assembly` to consume
- remain synchronous, pure, and side-effect free, per [Section 7](#7-kernel-purity-requirements)
- perform no enforcement
- perform no audit persistence
- contain no second implementation of deny precedence, default deny, effect aggregation, selector semantics, condition semantics, operator semantics, or applicability semantics — where any of these appears to be missing from `policy`, the correct response is to extend `policy`, not to implement it in `engine`

**`response_assembly`** — future responsibility:

- construct operation-aware `DecisionResponse` objects, per [Section 4 of the authorization model](operation-aware-authorization-model.md#4-rich-decisionresponse-conceptual-fields)
- maintain agreement between the assembled response and the trace `trace_assembly` produces for the same evaluation
- construct bounded kernel-side audit evidence when that contract is available, consistent with [Section 14 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#14-audit-event-assembly-ownership), which already assigns `basis-core` — not `basis-gateway` or `basis-console` — as the producer of decision response, reason metadata, evaluation trace, and kernel-side audit evidence
- perform no audit persistence
- perform no gateway enforcement

---

## 6. Layer Ownership Clarification

None of the assignments below are new except the `evaluation` row. They restate, with `evaluation` added, the same ownership model [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#allowed-kernel-responsibilities), [Section 8 of the authorization model](operation-aware-authorization-model.md#8-boundary-ownership), [Section 15 of the evaluation semantics document](operation-aware-evaluation-semantics.md#15-boundary-preservation), and [Section 16 of the policy/rule model](operation-aware-policy-rule-model.md#16-ownership-boundaries) already establish.

**Domain**

Owns: foundational value objects, shared identifiers, shared constrained vocabularies, evidence references, protocol-neutral domain contracts.

Does not own: trace assembly, policy evaluation orchestration, enforcement.

**Decisions**

Owns: request contracts, response contracts, evaluation-boundary data shapes.

Does not own: policy semantics, trace assembly, enforcement.

**Policy**

Owns and implements pure, executable authorization semantics, per [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md): bundle applicability, candidate selection, selector matching, condition evaluation, condition-result aggregation, rule-effect aggregation, deny precedence, allow determination, default deny, `NOT_APPLICABLE` determination, and machine-readable decision-reason selection where ADR-0004 already assigns it to policy semantics. These are executable operations that `policy` implements, not merely prose that a later layer reimplements. The complete concern-by-concern breakdown is in the [semantic ownership table](#semantic-ownership-table) below.

Does not own: audit models, trace models, response assembly, or orchestration of the sequence in which its own operations are invoked.

[Section 9 of the policy/rule model](operation-aware-policy-rule-model.md#9-bundle-combining-and-rule-combining) is not superseded, moved, or reinterpreted by this document.

```text
policy       defines and implements pure authorization semantics
evaluation   sequences and invokes those policy-owned semantic operations
```

**Audit**

Owns: bounded trace data models, bounded audit-evidence data models, audit contracts, per [ADR-0003](../adr/0003-operation-aware-trace-audit-evidence.md).

Does not own: policy evaluation, evaluator orchestration, enforcement.

**Evaluation**

Owns: pure orchestration of the evaluation phases in [Section 3 of the evaluation semantics document](operation-aware-evaluation-semantics.md#3-evaluation-phases) — sequencing policy-owned semantic operations, passing typed values between stages, and coordinating end-to-end kernel evaluation — trace assembly, response assembly, assembly of bounded kernel audit evidence where this architecture assigns that responsibility, and preserving agreement among the decision, trace, and audit artifacts it assembles.

Does not own: external I/O, persistence, protocol normalization, runtime enforcement, or any authorization semantic named in the Policy row above. `evaluation` must not contain a second implementation of deny precedence, default deny, effect aggregation, selector semantics, condition semantics, operator semantics, or applicability semantics.

### Semantic Ownership Table

This table is the authoritative, concern-by-concern reference for the policy/evaluation distinction stated above. It clarifies existing ownership already established by ADR-0002 and ADR-0004; it does not move or create any policy semantic.

| Concern | Semantic owner | Orchestration owner |
| --- | --- | --- |
| Bundle applicability | `policy` | `evaluation` invokes |
| Candidate selection | `policy` | `evaluation` invokes |
| Selector matching | `policy` | `evaluation` invokes |
| Condition evaluation | `policy` | `evaluation` invokes |
| Condition-result aggregation | `policy` | `evaluation` invokes |
| Rule-effect aggregation | `policy` | `evaluation` invokes |
| Deny precedence | `policy` | `evaluation` invokes |
| Default deny | `policy` | `evaluation` invokes |
| `NOT_APPLICABLE` determination | `policy` | `evaluation` invokes |
| Final authorization outcome | `policy` semantics | `evaluation` carries the result |
| Trace assembly (`TraceRuleEvidence`, `EvaluationTrace`) | `evaluation` | `evaluation` |
| Response assembly (`DecisionResponse`) | `evaluation` | `evaluation` |
| Kernel audit-evidence assembly (`AuditEvidence`) | `evaluation` | `evaluation` |
| Audit persistence | outside `evaluation` | `basis-gateway`/runtime boundary |
| Runtime enforcement | `enforcement` | `enforcement` |

**Adapters**

Unchanged from [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#allowed-kernel-responsibilities): adapter interface contracts, not adapter implementations. `evaluation` does not alter what `adapters` owns. `adapters` must not import `audit`, `evaluation`, or `enforcement`, per [Section 4](#4-dependency-rules).

**Enforcement**

Owns: the runtime authorization boundary, invoking evaluation, enforcing the returned result, fail-closed behavior, audit-write interaction, external/runtime integration.

Does not own: evaluation semantics. `enforcement` invokes `evaluation`'s deterministic result; it does not redefine what that result means.

---

## 7. Kernel Purity Requirements

`evaluation` is part of the deterministic kernel `basis-core` already commits to preserving, per [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#non-negotiable-rules). It must remain:

- synchronous
- side-effect free
- deterministic — identical inputs (request, policy bundle, evaluation config) must produce identical outputs, restating [Section 13 of the evaluation semantics document](operation-aware-evaluation-semantics.md#13-deterministic-trace-output) at the orchestration level
- transport agnostic
- protocol agnostic
- identity-provider agnostic
- offline
- free of network access
- free of filesystem access
- free of database access
- free of environment-variable lookup
- free of clocks
- free of randomness
- free of logging side effects
- free of telemetry side effects

The evaluation layer may construct bounded result objects — `DecisionResponse`, `EvaluationTrace`, kernel-side `AuditEvidence`. It may not persist them, transmit them, or otherwise cause an effect outside the value it returns to its caller.

---

## 8. Trace, Audit, and Enforcement Separation

The distinction [Section 2 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#2-trace-vs-audit) establishes is unchanged:

```text
EvaluationTrace
    explains evaluation

AuditEvidence
    records bounded kernel evidence

GatewayAuditEvent
    records enforcement-boundary facts
```

`evaluation` may assemble bounded kernel artifacts in the first two categories. It does not persist audit data — persistence remains a `basis-gateway`/deployment concern, per [Section 15 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#15-audit-failure-behavior). It does not record gateway runtime facts — those exist only at the gateway, per [Section 2 of the trace and audit evidence model](operation-aware-trace-audit-evidence.md#2-trace-vs-audit). It does not enforce decisions. [ADR-0003](../adr/0003-operation-aware-trace-audit-evidence.md) is not weakened by this document.

---

## 9. Alternatives Considered

[ADR-0006](../adr/0006-evaluation-orchestration-layer.md) records the alternatives considered and the reasons they were rejected, including a `policy`-to-`audit` import exception, relocating trace contracts into `domain`, placing pure orchestration in `enforcement`, introducing parallel trace-shaped structures, and hiding imports through local, function-scoped, or type-checking-only import workarounds. This document does not restate that analysis; ADR-0006's Alternatives Considered section is authoritative, consistent with the pattern the other operation-aware companion documents already follow — alternatives live in the ADR, not in the companion.

---

## 10. Compatibility

This decision is additive. It does not change:

- published schema contracts
- operation-aware request fields
- operation-aware response fields
- trace fields
- audit-evidence fields
- policy semantics
- condition semantics
- effect-combining semantics
- enforcement semantics
- `v0.1.0` behavior

It changes architectural ownership, legal dependency direction, and future module placement only. The ownership boundaries [ADR-0003](../adr/0003-operation-aware-trace-audit-evidence.md) and [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md) established remain valid without modification:

```text
policy       owns evaluator facts
audit        owns trace models
evaluation   composes them
```

---

## 11. Security and Determinism Consequences

**Positive:**

- Prevents circular or reverse dependencies between `policy` and `audit` that a future composition need might otherwise have introduced informally.
- Keeps policy evaluation isolated from evidence-shaping concerns.
- Prevents audit models from acquiring evaluator behavior.
- Prevents policy modules from acquiring audit concerns.
- Preserves bounded trace construction as a single, reviewable responsibility rather than a duplicated one.
- Keeps enforcement side effects outside the deterministic kernel.
- Makes import-boundary violations testable: a subpackage-dependency test can assert `evaluation` never imports `adapters` or `enforcement`, the same way it already asserts `policy` never imports `audit`.

**Risks:**

- Introducing another kernel subpackage increases the conceptual surface a new contributor has to learn.
- `evaluation`'s ownership must remain narrow, or it risks becoming a general-purpose utility package that unrelated orchestration gets added to by default.
- Future contributors may be tempted to place logic in `evaluation` because it is permitted to import from everywhere else in the kernel, not because the logic belongs there.

**Mitigations:**

- The allowed/forbidden import lists in [Section 4](#4-dependency-rules) are explicit and narrow.
- [Section 5](#5-evaluation-layer-responsibilities) names exactly three future modules and their responsibilities; anything outside that scope is not automatically an `evaluation` concern merely because `evaluation` could technically import what it needs.
- Recursive import-boundary tests, per [Section 12](#12-implementation-guidance-for-basis-core), make violations of [Section 4](#4-dependency-rules) mechanically detectable rather than dependent on reviewer memory.
- Any future expansion of `evaluation`'s responsibilities beyond [Section 5](#5-evaluation-layer-responsibilities) requires its own ADR, consistent with [`docs/adr/README.md`](../adr/README.md#when-an-adr-is-required).

---

## 12. Implementation Guidance for basis-core

This section is guidance for the future `basis-core` implementation. It is not implementation in this repository, and it does not schedule or reorder any milestone in `basis-core`'s existing v0.2.0 implementation roadmap; that sequencing decision belongs to a separate `basis-core` roadmap-alignment change.

`basis-core` should eventually create:

```text
src/basis_core/evaluation/__init__.py
src/basis_core/evaluation/operation_aware/__init__.py
src/basis_core/evaluation/operation_aware/trace_assembly.py
src/basis_core/evaluation/operation_aware/engine.py
src/basis_core/evaluation/operation_aware/response_assembly.py
```

`basis-core` should add recursive import-boundary tests covering:

```text
policy/operation_aware/
audit/operation_aware/
evaluation/operation_aware/
```

The expected `evaluation` boundary those tests should assert:

```text
allowed:    domain, decisions, policy, audit
forbidden:  adapters, enforcement
```

This is the same boundary [Section 4](#4-dependency-rules) defines; it is restated here because [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#required-implementation-checks) already treats import-boundary test coverage as governance infrastructure, not incidental test coverage, and a new kernel subpackage is exactly the kind of change that requires expanding that coverage.

---

## 13. Non-Goals

This document does not:

- Implement the operation-aware evaluation engine (`trace_assembly`, `engine`, or `response_assembly`)
- Define Python function signatures, class hierarchies, or data structures for any `evaluation` module
- Implement executable policy semantics (deny precedence, default deny, effect aggregation, selector matching, condition evaluation, or any other operation ADR-0004 assigns to `policy`)
- Change `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`
- Reorder or schedule `basis-core`'s existing v0.2.0 implementation roadmap
- Change any published schema contract, request field, response field, trace field, audit-evidence field, or semantic already established by ADR-0001 through ADR-0005
- Define audit storage or deployment behavior

---

## References

- [ADR-0006](../adr/0006-evaluation-orchestration-layer.md) — the decision this document is the companion to
- [ADR-0002](../adr/0002-operation-aware-evaluation-semantics.md) — the evaluation semantics `evaluation`'s `engine` module orchestrates
- [ADR-0003](../adr/0003-operation-aware-trace-audit-evidence.md) — the trace and audit evidence model `evaluation`'s `trace_assembly` module composes
- [ADR-0004](../adr/0004-operation-aware-policy-rule-model.md) — the policy bundle/rule model, including combining semantics, that remain policy-owned under this decision
- [`docs/architecture/operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md) — the evaluation phase sequence `engine` orchestrates
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md) — the trace/audit evidence categories and assembly ownership this document builds on
- [`docs/architecture/operation-aware-policy-rule-model.md`](operation-aware-policy-rule-model.md) — the policy bundle/rule/combining model this document does not alter
- [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md) — the boundary ownership this document restates with `evaluation` added
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules and import boundary summary this document extends
- [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md) — component responsibilities and dependency direction
- [`CONTRIBUTING.md`](../../CONTRIBUTING.md#basis-core-boundary-rules) — the basis-core boundary protection policy behind the alternatives ADR-0006 records
- [`docs/glossary.md`](../glossary.md) — terminology used throughout this document
