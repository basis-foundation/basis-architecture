# ADR-0003: Define Operation-Aware Trace and Audit Evidence Model

## Status

Proposed

## Context

[ADR-0001](0001-operation-aware-ot-authorization.md) recorded the decision to evolve `basis-core` toward an operation-aware authorization model, and its companion document, [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md), named the conceptual categories a future audit and trace evidence model would need to cover — decision request hash, policy bundle version, matched rule IDs, reason codes, evaluation trace, normalized adapter evidence reference, identity source reference, correlation ID propagation, safe redaction rules, and no secrets in audit — while explicitly deferring the design of that model to later architecture work.

[ADR-0002](0002-operation-aware-evaluation-semantics.md) recorded the decision to define deterministic evaluation semantics, and its companion document, [`docs/architecture/operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md), added trace-specific expectations in its own deferred scope: trace output must be deterministic, stable enough for tests, safe to expose in audit and operator views after redaction, clear about matched rules, skipped rules, missing context, and reason codes, and free of secrets. That document deferred the full trace schema, along with the audit event schema and reason-code vocabulary, to a dedicated audit/trace architecture PR.

Both prior documents converge on the same open question: what evidence must exist, conceptually, to make an operation-aware decision explainable, auditable, testable, and safe to visualize — before any of `basis-schemas`, `basis-core`, `basis-gateway`, or `basis-console` commit to a specific trace or audit schema. Leaving this open any longer risks the same problem ADR-0002 was written to avoid: each downstream component inventing its own notion of what "audit evidence" means, with no shared architectural target, and no agreed boundary between what the kernel explains and what the gateway records about enforcement.

## Decision

BASIS will define trace and audit evidence as separate but related architecture surfaces:

- Kernel-produced evaluation trace
- Kernel-side decision/audit evidence
- Adapter evidence references
- Identity evidence references
- Gateway enforcement audit event
- Redacted console explanations

The conceptual model for this decision is defined in [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md). That document defines the distinction between evaluation trace, audit evidence, and gateway audit events; the evidence lifecycle from identity context through console visualization; conceptual categories for evaluation trace, rule-level evidence, request evidence, adapter evidence references, and identity evidence references; gateway enforcement evidence; redaction and secret-handling rules; the distinction between trace and human-readable explanation; the role of reason codes; determinism and compatibility expectations; audit event assembly ownership; audit failure behavior; and console/training-mode use of evidence. It is the companion to this ADR, in the same relationship the authorization model and evaluation semantics documents have with ADR-0001 and ADR-0002.

This ADR records the decision to define this evidence model now, ahead of `basis-schemas` and `basis-core` v0.2.0. It does not itself change `basis-core`, `basis-schemas`, `basis-gateway`, `basis-adapters`, `basis-console`, or `basis-identity`.

## Alternatives Considered

**Leave trace and audit evidence design to `basis-schemas` or `basis-core` v0.2.0 implementation.** Rejected, for the same reason ADR-0002 rejected the equivalent alternative for evaluation semantics: redaction rules, evidence ownership, and the trace/audit distinction are exactly the kind of decisions that should not be made under implementation pressure or reviewed with less scrutiny than a compatibility-surface decision requires.

**Design a single unified audit schema now, covering kernel trace, adapter evidence, identity evidence, and gateway enforcement facts in one structure.** Rejected. Collapsing these into one schema at this stage would either force `basis-core` to know about enforcement facts it structurally cannot know, or force `basis-gateway`'s audit event to depend on an internal kernel trace shape before either is stabilized independently. Keeping trace, kernel-side audit evidence, adapter evidence, identity evidence, and gateway audit events as related-but-separate concepts, each owned by the component that can actually produce it, preserves the same isolation boundary [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) already protects.

**Finalize the reason-code vocabulary as part of this PR.** Rejected. Reason codes are a governed compatibility surface consumed by tests, console, audit, and troubleshooting simultaneously; finalizing that vocabulary requires input from `basis-schemas` and from implementation experience this architecture work does not yet have. This PR records that reason codes exist and are governed, not what the final list contains.

**Design audit storage, retention, and tamper-evidence now, alongside the evidence categories.** Rejected. Storage, retention, and tamper-evidence are runtime and deployment concerns that depend on choices (database technology, deployment topology, compliance regime) this architecture repository does not make. Defining the evidence categories without prescribing how they are stored keeps this document useful across deployment models rather than tied to one.

## Consequences

**Positive:**

- Decisions become explainable — the categories in [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) give `basis-schemas` concrete evidence requirements to formalize rather than an open design problem.
- Audits become evidence-backed — kernel trace, adapter evidence, identity evidence, and gateway enforcement facts each have a defined conceptual home before any of them are implemented.
- Console/training mode has a stable evidence source — `basis-console` can be built against a defined evidence model rather than against whatever a specific kernel or gateway implementation happens to expose.
- Future schemas have a concrete target — `basis-schemas` inherits categories, not an open design question, for trace, audit, adapter evidence, and identity evidence schemas.
- Compatibility tests can verify trace behavior — the determinism and compatibility expectations in Section 13 of the companion document give future compatibility tests a defined target.

**Tradeoffs:**

- More contract surface area — trace, kernel-side audit evidence, adapter evidence references, identity evidence references, and gateway audit events are now five related surfaces to keep consistent, rather than one undifferentiated "audit" concept.
- Redaction becomes a first-class requirement — every evidence category this document defines must be classified into a redaction tier before it can be exposed, which is additional design work later schemas must carry.
- Reason-code governance becomes necessary — treating reason codes as a compatibility surface means future changes to that vocabulary require the same care as changes to the action vocabulary.
- Audit storage remains a future unresolved runtime concern — this ADR intentionally leaves storage, retention, and audit-failure deployment policy open, which means that work still has to happen before a production deployment can rely on this model end to end.

## Non-goals

This ADR does not:

- Define final schemas
- Implement tracing
- Implement audit storage
- Define SIEM/export behavior
- Define retention policy
- Define compliance mappings
- Define final reason-code vocabulary
- Change any implementation repo
- Define gateway fail-open/fail-closed behavior

## References

- [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) — the conceptual evidence model this ADR adopts
- [ADR-0001](0001-operation-aware-ot-authorization.md) — the decision that named the audit/trace categories this ADR resolves
- [ADR-0002](0002-operation-aware-evaluation-semantics.md) — the decision that named the trace expectations this ADR resolves
- [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md) — Section 7's audit and trace direction, the starting point for this document
- [`docs/architecture/operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md) — Section 13's deterministic trace output expectations
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules this evidence model preserves
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction
- [`docs/architecture/compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) — the compatibility commitments the determinism and compatibility expectations build on
- [`docs/architecture/action-vocabulary.md`](../architecture/action-vocabulary.md) — the governance model reason codes are analogous to
- [`docs/glossary.md`](../glossary.md) — terminology used throughout this ADR and its companion document
