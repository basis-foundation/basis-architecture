# ADR-0006: Introduce a Pure Evaluation Orchestration Layer

## Status

Proposed

## Context

Preparing `basis-core`'s operation-aware implementation work surfaced a dependency conflict the existing kernel boundary rules cannot resolve as written. [ADR-0003](0003-operation-aware-trace-audit-evidence.md) assigns `EvaluationTrace` and `TraceRuleEvidence` to `audit`. [ADR-0004](0004-operation-aware-policy-rule-model.md) assigns rule-level and bundle-level evaluation facts — matched rules, skipped rules, condition outcomes, combining results — to `policy`. Constructing a complete, audit-shaped trace requires both: the trace must be built from policy-owned evaluation facts, using the audit-owned contracts trace consumers depend on.

[`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md#import-boundary-summary) permits `policy` to import only `domain` and `decisions`, and permits `audit` to import only `domain` and `decisions`. Neither may import the other, and this ADR does not propose relaxing either rule. No kernel subpackage that existed prior to this decision may legally perform the composition trace assembly requires — `policy` cannot construct audit-owned trace contracts without an import it isn't allowed to make, and `audit` cannot derive trace content from policy-owned facts without an import it isn't allowed to make either. This is a missing architectural responsibility, not a misplaced file: no prior document assigned ownership of policy-fact-to-trace-evidence composition to any kernel subpackage, because no kernel subpackage was the legal owner of that composition.

The same composition problem recurs for operation-aware response assembly, which must maintain agreement between the `DecisionResponse` the kernel returns and the trace that explains it — again requiring facts that originate in both `policy` and `audit`. Deferring this decision further would recreate the risk [ADR-0002](0002-operation-aware-evaluation-semantics.md) and [ADR-0003](0003-operation-aware-trace-audit-evidence.md) were each written to avoid: `basis-core` making composition-boundary decisions under implementation pressure, with less architectural scrutiny than a new kernel subpackage warrants.

## Decision

BASIS will introduce `evaluation` as a first-class `basis-core` kernel subpackage, implemented in the future at `src/basis_core/evaluation/`. Its responsibility is pure, deterministic orchestration of already-defined `domain`, `decisions`, `policy`, and `audit` contracts into complete kernel evaluation behavior — the legal composition boundary [Section 2 of the companion document](../architecture/operation-aware-evaluation-orchestration.md#2-the-composition-problem) shows is currently missing.

The conceptual model for this decision is defined in [`docs/architecture/operation-aware-evaluation-orchestration.md`](../architecture/operation-aware-evaluation-orchestration.md). That document defines the composition problem in full; the evaluation layer's dependency rules (`evaluation` may import `domain`, `decisions`, `policy`, and `audit`; it must not import `adapters` or `enforcement`); the three future modules under `evaluation/operation_aware/` — `trace_assembly`, `engine`, `response_assembly` — and their conceptual responsibilities; a layer ownership clarification confirming that effect-combining semantics remain `policy`-owned while `evaluation` owns only orchestration of those semantics; the kernel purity requirements `evaluation` must satisfy (synchronous, side-effect free, deterministic, no I/O); the preserved distinction between `EvaluationTrace`, `AuditEvidence`, and `GatewayAuditEvent`; rejected alternatives; compatibility; security and determinism consequences; and implementation guidance for `basis-core`. It is the companion to this ADR, in the same relationship the authorization model, evaluation semantics, trace/audit evidence, and policy/rule model documents have with ADR-0001 through ADR-0004.

`evaluation` sits above `policy` and `audit` and below `enforcement` in the kernel's import direction. It composes without being composed into: nothing in `policy`, `audit`, `domain`, or `decisions` may import `evaluation`, and `evaluation` may not import `adapters` or `enforcement`. `enforcement` may import `evaluation`, consistent with `enforcement`'s existing position as the top layer that already imports `domain`, `decisions`, `policy`, `audit`, and `adapters`.

This ADR records the decision to introduce this layer now, ahead of `basis-core` v0.2.0's operation-aware implementation. It does not itself change `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`, and it does not reorder or schedule `basis-core`'s existing 44-PR, 15-milestone v0.2.0 implementation roadmap.

## Consequences

**Positive:**

- `basis-core` has a legal composition boundary for trace assembly and response assembly that did not previously exist.
- `policy` and `audit` remain mutually isolated — this decision does not create, directly or transitively, an import path between them.
- Import-boundary violations at this new boundary become testable, the same way `policy` → `audit` isolation is already testable.
- `basis-core`'s v0.2.0 roadmap-alignment work has a defined target subpackage for trace and response assembly, rather than an open composition question.
- The kernel/runtime boundary stays sharp: pure orchestration remains distinct from `enforcement`'s side-effect-bearing runtime behavior.

**Tradeoffs:**

- A new kernel subpackage increases the conceptual surface a contributor must learn.
- `evaluation`'s ownership must be actively kept narrow — it is permitted to import from every other kernel subpackage below `enforcement`, which makes it an easy place to misplace unrelated orchestration if reviewers do not hold the line established in [Section 5 of the companion document](../architecture/operation-aware-evaluation-orchestration.md#5-evaluation-layer-responsibilities).
- `basis-core` must add recursive import-boundary tests for the new subpackage before implementation begins, which is additional test-infrastructure work ahead of feature work.
- Any future expansion of `evaluation`'s responsibilities beyond the three modules named here requires its own ADR.

## Alternatives Considered

**Grant `policy` an exception to import `audit`.** Rejected. Violates the established dependency direction on a one-off basis and sets a precedent that the next composition problem — and there will be one, given response assembly's identical shape — would reasonably invoke again. The aggregate effect of accepting such exceptions is a kernel whose isolation is nominal rather than real, which is the exact erosion [`CONTRIBUTING.md`](../../CONTRIBUTING.md#basis-core-boundary-rules) already warns against.

**Move trace models into `domain`.** Rejected. `EvaluationTrace` and `TraceRuleEvidence` are explanation- and audit-oriented contracts; relocating them into the foundational, protocol-neutral `domain` layer to solve an import problem would not cleanly solve the later response-and-audit composition problem, and it would move already-established ownership for a reason unconnected to what the contracts are for. This is not a judgment that the trace contracts are poorly designed — the issue is ownership composition, not contract quality.

**Move orchestration into `enforcement`.** Rejected. Conflates pure, deterministic kernel evaluation with runtime enforcement, weakens the test isolation that makes a dedicated evaluation subpackage valuable, and mixes side-effect-free computation with side-effect-bearing orchestration in a way that obscures the kernel/runtime boundary [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) exists to keep sharp.

**Let `policy` produce parallel, non-canonical trace-shaped structures that `audit` never has to import.** Rejected. Duplicates the contracts `basis-schemas` publishes, creates conversion debt, weakens type safety at the boundary, and risks drift from the canonical `TraceRuleEvidence`/`EvaluationTrace` shape over time.

**Resolve the conflict with local imports, function-scoped imports, or type-checking-only imports.** Rejected. These hide the dependency from static import-boundary tooling rather than resolving it — the import still exists; only its visibility to governance tooling changes. This preserves the architectural violation under a technicality and weakens static governance.

## Non-goals

This ADR does not:

- Implement the operation-aware evaluation engine (`trace_assembly`, `engine`, or `response_assembly`)
- Define Python function signatures, class hierarchies, or data structures for any `evaluation` module
- Implement executable policy semantics (deny precedence, default deny, effect aggregation, selector matching, condition evaluation, or any other operation ADR-0004 assigns to `policy`)
- Change `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`
- Reorder or schedule `basis-core`'s existing v0.2.0 implementation roadmap
- Change any published schema contract, request field, response field, trace field, audit-evidence field, or semantic already established by ADR-0001 through ADR-0005
- Relax the `policy` ↔ `audit` isolation rule
- Define audit storage or deployment behavior

## References

- [`docs/architecture/operation-aware-evaluation-orchestration.md`](../architecture/operation-aware-evaluation-orchestration.md) — the conceptual evaluation orchestration layer this ADR adopts
- [ADR-0002](0002-operation-aware-evaluation-semantics.md) — the evaluation semantics `evaluation`'s `engine` module orchestrates without redefining
- [ADR-0003](0003-operation-aware-trace-audit-evidence.md) — the trace and audit evidence model `evaluation`'s `trace_assembly` module composes; not weakened by this decision
- [ADR-0004](0004-operation-aware-policy-rule-model.md) — the policy bundle/rule model and combining semantics that remain policy-owned under this decision
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules and import boundary summary this decision extends
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction
- [`CONTRIBUTING.md`](../../CONTRIBUTING.md#basis-core-boundary-rules) — the basis-core boundary protection policy this ADR's alternatives section applies
- [`docs/glossary.md`](../glossary.md) — terminology used throughout this ADR and its companion document
