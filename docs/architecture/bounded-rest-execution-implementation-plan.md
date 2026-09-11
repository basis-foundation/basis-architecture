# Bounded REST Execution Implementation Plan

**Status:** Approved for bounded implementation. This document translates the coupled Accepted ADR-0011, ADR-0012, ADR-0013, ADR-0014, ADR-0015, and ADR-0016 execution-readiness milestone into a bounded, repository-aware implementation sequence for `basis-producer`. It does not implement runtime behavior in any repository and does not add or modify a schema. It does not alter any Accepted ADR. Work Item 1 was additionally gated on a `basis-gateway`-side contract prerequisite, made explicit in §1's **Prerequisite** subsection below: the operation-aware evaluation response did not, at plan-authoring time, expose the authoritative kernel `AuditEvidence.evidence_id` to the caller. **That prerequisite is now satisfied** — see §1 — and Work Item 1 is the next, unblocked implementation step.

**Authority:** All implementation obligations in this plan are traceable to Accepted ADRs (ADR-0011 through ADR-0016, plus the foundational Accepted ADRs they build on) and the normative invariants those ADRs restate from companion documents. Where a choice is not fixed by architecture, it is labeled IMPLEMENTATION FREEDOM. Where a question is left open by architecture, it is labeled with its exact architectural posture (per `docs/architecture-discovery.md` gate-state vocabulary) and is not resolved by this plan.

**Companion documents:** [ADR-0011](../adr/0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](../adr/0012-authorization-to-execution-binding.md), [ADR-0013](../adr/0013-execution-lifecycle-semantics.md), [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md), [ADR-0015](../adr/0015-first-bounded-execution-target.md), [ADR-0016](../adr/0016-bounded-target-replay-freshness-posture.md), [ADR-0007](../adr/0007-adapter-evidence-construction.md), [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md), [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md), [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md), [`bounded-operation-producer-reference-implementation-plan.md`](bounded-operation-producer-reference-implementation-plan.md) (authorization-side historical record; this plan is not a continuation of it), [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`adapter-evidence-construction-semantics.md`](adapter-evidence-construction-semantics.md).

---

## 1. Purpose and Scope

This document is the implementation plan for the first bounded REST `authorize → execute → evidence` reference slice in the BASIS ecosystem. Its purpose is to translate governed, accepted execution-readiness architecture into a concrete, bounded PR sequence that an implementer can follow without inventing security-critical behavior, without making new architectural decisions, and without widening the approved topology or scope.

### Where the Prior Slice Left Off

The authorization side of the boundary is implemented: `basis-producer`'s Phases 2A through 5 — evidence retention, reference lifecycle, authenticated gateway client, REST adapter composition, and conformance/demonstration — are merged and complete. `basis_producer.rest_composition`'s own module docstring states the scope boundary explicitly, as a governed architectural boundary rather than an incidental omission: the module stops at the authoritative authorization disposition and does not execute any protocol operation under any disposition, including `ALLOW`.

This plan begins exactly where that implementation ends: at the point an `ALLOW` disposition is received.

### What This Plan Is Not

This plan is not a continuation of the Phase 2A–5 numbering. ADR-0011 states directly: "This is not '`basis-producer` Phase 6' — the completed five-phase producer authorization implementation plan remains complete, and any later execution implementation plan is a new architecture chapter and a new implementation plan, not an extension of the old Phase 2A–5 numbering." Work items in this plan are the first items in a new, distinct implementation chapter.

This document is not an ADR and does not reopen any decision the Accepted ADRs have settled. This document is the implementation-planning source of truth for the first bounded execution slice until that slice is complete. Implementation PRs should refer back to it rather than re-deciding architecture.

### Prerequisite: Gateway Evidence-ID Exposure — SATISFIED

**Current state:** This prerequisite is satisfied. `basis-gateway` (revision `1e08348483944d1859739cdf1d1cc16d31ce518a`) exposes `evidence_id` on the operation-aware evaluation response (`OperationAwareEvaluateResponse`, `src/basis_gateway/api/operation_aware_schemas.py`, returned by `POST /v1/evaluate/operation-aware`), copied verbatim from `result.audit_evidence.evidence_id` — never regenerated or derived — and absent only when `result.audit_evidence is None`. The gateway establishes the cross-artifact invariant `response.evidence_id == AuditEvidence.evidence_id == GatewayAuditEvent.audit_evidence_id` for evaluations that produce authoritative `AuditEvidence`. Work Item 1 is unblocked by this prerequisite and is the next implementation step (it has not started).

The historical discovery below, recorded at plan-authoring time, is retained for provenance and is no longer the current state.

**Historical discovery (plan-authoring time; superseded above).** ADR-0012's binding record carries, by reference, "the gateway-generated correlation ID and the kernel's `trace_id`/`evidence_id`" once the authorization round trip completes (ADR-0012, "The binding record"). ADR-0014 correspondingly lists `kernel_evidence_id` — `AuditEvidence.evidence_id` — as an **always-required** minimum evidence fact for both Case 1 and Case 2 execution evidence (ADR-0014 Minimum Evidence Facts; Work Item 3 §B below). Both ADRs assume the operation-producer role can obtain this identifier from the authorization round trip, alongside `correlation_id` and `trace_id`.

Direct inspection of the then-current, released `basis-gateway` operation-aware evaluation response and of the response model `basis-producer`'s gateway client already parsed (`gateway_client_response.py`) confirmed that the then-released, public gateway contract returned `request_id`, `correlation_id`, `bundle_id`, `bundle_version`, `trace_id`, `reason_code`, `explanation`, `disposition`, and an optional `evaluation_trace` — but did not return `evidence_id`. `AuditEvidence.evidence_id` was generated by the same gateway request and was used only to construct `basis-gateway`'s own internal, durably-recorded `GatewayAuditEvent` (`api/routes.py`'s post-classification audit-emission step); it was never serialized into the HTTP response body returned to the caller. This matched the published `operation-aware-decision-response` contract (`basis-schemas`) as it stood at that time, whose constraints explicitly excluded any audit-event field (`event_id` or similar) from this response shape, reserving it for "a future PR F `AuditEvidence`/`GatewayAuditEvent` contract."

**This was an implementation-state gap the accepted architecture already anticipated the shape of, not an open architectural question.** ADR-0012 and ADR-0014 already decided that the binding record and execution evidence correlate to the authoritative kernel evidence identifier; what was missing was the gateway actually exposing that already-governed identifier on the response it returns to `basis-producer`. That gateway-side change has since been made — see **Current state**, above.

**Provenance constraint (continues to apply now that the prerequisite is satisfied).** `kernel_evidence_id` must be preserved by `basis-producer` exactly as the gateway returns it. `basis-producer` must not: synthesize or locally generate an `evidence_id`-shaped value; derive one from `trace_id` or `correlation_id`; infer one from any other field; or substitute a placeholder when the gateway response omits it (`audit_evidence` was `None` for that evaluation). This constraint is unconditional and does not lapse now that the prerequisite is satisfied — a `basis-producer` implementation that fabricates or infers a stand-in value violates this constraint rather than satisfies it.

---

## 2. Governing Architecture

The following Accepted ADRs are the architectural authority for this plan. ADRs 0001 through 0006 carry `Status: Proposed` and are not governing sources.

| ADR | Status | Governing role in this plan |
| --- | --- | --- |
| ADR-0007 | Accepted | Canonicalization/digest pattern reused for the binding record; adapter evidence facts this plan must not repurpose |
| ADR-0008 | Accepted | Producer workload authentication topology; producer/authorization-subject identity separation — both inherited unchanged |
| ADR-0009 | Accepted | Trusted NGINX ingress topology — inherited unchanged |
| ADR-0010 | Accepted | `basis-producer` as the permanent operation-producer-runtime repository — this plan adds to that repository |
| ADR-0011 | Accepted | Protocol-executor role definition and invariants; same-process default topology; governing execution invariants; follow-on gate structure; "no Phase 6" instruction |
| ADR-0012 | Accepted | Authorization-to-execution binding mechanism: binding record structure, creation ownership (operation-producer role), verification ownership (protocol-executor role), failure semantics, process-local single-consumption freshness narrowing |
| ADR-0013 | Accepted | Execution-lifecycle semantics: three independent axes, attempt-outcome vocabulary, pre-dispatch record ordering, multi-stage semantics (single-stage for REST), retry definition, broader replay/freshness open-question delineation |
| ADR-0014 | Accepted | Minimum execution-evidence semantics: evidence definition, two production cases (Case C and dispatched), construction/retention ownership, minimum evidence facts, correlation model, evidence-durability axis |
| ADR-0015 | Accepted | First bounded execution target: same-process REST, credential-free, deterministic local test service, one bounded operation shape, required demonstrable outcomes |
| ADR-0016 | Accepted | Authorization replay/freshness posture for the ADR-0015 target: NON-BLOCKING FOR THIS TARGET under stated topology constraints; operational duplicate-execution/idempotency explicitly left OPEN |

**Normative companion documents** (authority through their citing Accepted ADRs, not independently):

- `docs/architecture/operation-producer-and-execution-boundary.md` §5 lifecycle invariants — restated normatively by ADR-0011 Decision 11, ADR-0013 Decision 11, ADR-0014
- `docs/architecture/operation-producer-and-execution-boundary.md` §6 correlation model — restated normatively by ADR-0014 Correlation Model
- `docs/architecture/adapter-evidence-construction-semantics.md` canonicalization profile — reused by ADR-0012 for the binding digest
- `docs/architecture/ecosystem-contract-inventory.md` "implementation proves a stable shape" principle — adopted normatively by ADR-0014 Schema Publication Boundary

---

## 3. Current Implementation State

Based on direct inspection of `basis-producer` at HEAD, the following modules are merged and complete:

| Module | Authorization-slice phase | Responsibility |
| --- | --- | --- |
| `basis_producer.evidence_store` | 2A | Durable digest-addressed evidence retention |
| `basis_producer.evidence_reference_lifecycle` | 2B | Reference lifecycle and retain-before-mint ordering |
| `basis_producer.gateway_client` | 3 | Authenticated gateway submission (mTLS producer + bearer authorization subject) |
| `basis_producer.rest_composition` | 4 | REST adapter composition — terminates at authorization disposition |

The Phase 5 conformance workflow covers eight categories including `no-execution`, which verifies the scope boundary explicitly. Protocol execution is entirely unimplemented. The current pipeline's terminal state is: receive authoritative disposition → STOP.

**What the new implementation adds inside `basis-producer`'s existing process boundary** (per ADR-0011 Decision 3's same-process default topology):

1. An **authorization-to-execution binding record** sub-responsibility, assigned to the existing operation-producer role
2. A **protocol-executor logical role**, logically distinct from but colocated with the operation-producer role
3. An **execution-evidence-producer logical role**, logically distinct from both

No new repository is created, and Work Items 1–4 themselves modify no existing repository other than `basis-producer`. (This is distinct from the Gateway Evidence-ID Exposure prerequisite in §1: a separate, preceding `basis-gateway` change, now satisfied, that was not itself one of Work Items 1–4 and was not performed by this plan.)

---

## 4. Topology Invariants

The following invariants govern every work item in this plan. Each is derived from governing architecture and must not be violated by any implementation choice. They are stated here together rather than repeated in full in each work item; each work item references back to the applicable invariants.

| Invariant | Governing source |
| --- | --- |
| Same-process colocation: executor and evidence-producer in `basis-producer`'s process | ADR-0011 Decision 3 |
| No new repository, daemon, queue, message bus, or deployment unit | ADR-0011 Decision 4 |
| No dispatch before authoritative `ALLOW` disposition is received and validated | ADR-0011 Decision 9; `operation-producer-and-execution-boundary.md` §5 |
| No dispatch if binding verification fails (any ADR-0012 Failure Semantics condition) | ADR-0012 Failure Semantics |
| No reuse of a consumed binding record across multiple dispatch attempts | ADR-0012 single-consumption guarantee |
| No queue, scheduler, or mechanism that holds a disposition or binding record between authorization and dispatch | ADR-0016 Boundary of Conclusion |
| Exactly one dispatch attempt per authorization-to-execution cycle; no automatic retry, redispatch, or caller-transparent retry orchestration | ADR-0015 Replay/Freshness Boundary; ADR-0016 Boundary of Conclusion |
| No protocol/device credential of any kind | ADR-0015 Gate 4 Assessment |
| Authorization evidence and execution evidence are separate artifacts — never merged, nested, or mutually revised | ADR-0011 Decision 11; ADR-0014 Execution Evidence Definition |
| Execution evidence must not mutate, nest inside, or complete `AuditEvidence` or `GatewayAuditEvent` | ADR-0014 Relationship to AuditEvidence and GatewayAuditEvent |
| Execution does not add an `enforcement_status` / `enforcement_result` field to `GatewayAuditEvent` | ADR-0014 |
| `basis-adapters` remains a network-free normalization library with no protocol dispatch | ADR-0011 Decision 16 |
| `basis-core` remains deterministic and side-effect-free with no execution responsibility | ADR-0011 Decision 17 |
| `basis-gateway` remains the authorization enforcement boundary with no protocol dispatch | ADR-0011 Decision 18 |
| No `basis-schemas` publication | ADR-0014 Schema Publication Boundary |

---

## 5. Authorization Boundaries

The following describe the implementation boundaries authorized by this plan. The "Not Authorized" prohibitions reflect constraints imposed by Accepted architecture.

### Authorized

- Adding authorization-to-execution binding record creation and verification to `basis-producer` (ADR-0012)
- Adding a protocol-executor logical role to `basis-producer` with REST dispatch to a deterministic local test service (ADR-0011, ADR-0015)
- Implementing execution-lifecycle observation per ADR-0013's three-axis model
- Adding an execution-evidence-producer logical role that constructs and retains evidence per ADR-0014's minimum semantics
- Conformance tests and a demonstration of the complete `authorize → execute → evidence` path

### Not Authorized

The following are prohibited **for Work Items 1–4 of this `basis-producer` implementation plan** by Accepted architecture and must not be introduced by those work items regardless of this plan's approval status. If implementation of any work item appears to require one of these, it is a signal that a STOP condition has been triggered and architectural review is needed before proceeding. This scoping is not a weakening of any cross-repository boundary: it clarifies that the prohibitions below bind what Work Items 1–4 themselves may touch, not whether some other, separate implementation effort in another repository may ever proceed — see the Gateway Evidence-ID Exposure prerequisite (§1), a `basis-gateway`-owned change that was neither authorized nor performed by this plan's own work items, and that is now satisfied (§1).

- Execution against a real OT protocol endpoint, physical device, PLC, gateway appliance, or cloud service
- Modification of `basis-gateway`, `basis-adapters`, `basis-core`, `basis-schemas`, or `basis-identity` **by Work Items 1–4 of this plan.** (The Gateway Evidence-ID Exposure prerequisite in §1 was separate, preceding `basis-gateway` implementation work outside this plan's work-item scope — this plan neither authorized nor performed it, and Work Items 1–4 must not perform it either; that separate `basis-gateway` implementation effort has since satisfied the prerequisite — see §1.)
- Publication of an execution-evidence schema or any `basis-schemas` contract
- Creation of `basis-executor` or any other new Foundation repository
- Introduction of a protocol/device credential of any kind
- Automatic retry, automatic redispatch, or caller-transparent retry orchestration
- Introduction of a queue, scheduler, or any mechanism that holds a disposition or binding record between authorization and dispatch
- Wall-clock expiry, policy-reload invalidation, restart-surviving replay prevention, retry/idempotency policy, or a replay database
- A signed execution grant, JWT, nonce, TTL, or one-time-use token (rejected for same-process topology by ADR-0012 Alternative D)
- Expanding the scope of execution beyond the single bounded REST target
- Widening the same-process topology beyond ADR-0011 Decision 3
- Resolving operational duplicate-execution/idempotency (OPEN — see §9)
- Altering any Accepted ADR
- Treating Work Item 1's binding-record correlation references, or Work Item 3's `kernel_evidence_id` evidence fact, as complete merely because the Gateway Evidence-ID Exposure prerequisite (§1) is now satisfied — the prerequisite being satisfied makes the identifier obtainable; it does not itself complete Work Item 1 or Work Item 3, which still require their own implementation
- `basis-producer` synthesizing, locally generating, or deriving (from `trace_id`, `correlation_id`, or any other field) a stand-in value for `kernel_evidence_id` in place of the authoritative gateway-returned identifier

---

## 6. Constraint Inventory

This constraint inventory is derived from the governing Accepted ADRs. It is the prerequisite gate for the work items that follow.

### MUST PRESERVE

**Architectural invariants the implementation must satisfy** (each traced to governing source):

1. Protocol-executor role is **logically distinct** from the operation-producer role even when both run in the same process. (ADR-0011 Decisions 1, 2)
2. The **binding record** covers both the preserved original `ProtocolOperation` and the normalized authorization request actually submitted. (ADR-0012 "What is bound")
3. The binding record is **created by the operation-producer role**, once, before the gateway call. (ADR-0012 "Where the binding is created")
4. Binding **verification occurs in the protocol-executor role**, immediately before dispatch. (ADR-0012 "Where the binding is verified")
5. The **pre-dispatch record is created before binding verification** begins, not after it succeeds. (ADR-0013 Decision 2)
6. A **Case C `NOT_ATTEMPTED`** (pre-dispatch boundary failure after a record exists) always has a pre-dispatch record to resolve against; a **Case B `NOT_ATTEMPTED`** (record creation failure) never does. (ADR-0013 Decision 2)
7. Attempt state tracks **three independent axes** (attempt outcome, resulting-state verification, evidence durability) — never collapsed into one flat field. (ADR-0013 Decision 3)
8. `DISPATCHED` is transient — must not persist as a terminal value. (ADR-0013 Decision 8)
9. A retry is a **new, distinct attempt** with its own fresh binding cycle. (ADR-0013 Decision 9) The first slice performs exactly one dispatch per cycle.
10. Execution evidence is produced **for Case C failures and dispatched attempts only** — not for Case A (pre-boundary) or Case B (record-creation failure). (ADR-0014 Execution Evidence Definition)
11. Construction is assigned to a **distinct execution-evidence-producer role**. (ADR-0014 Ownership)
12. Evidence records must carry the **minimum facts** enumerated in ADR-0014 (Work Item 3 §B). (ADR-0014 Minimum Evidence Facts)
13. The **pre-dispatch record/context identifier** is the lifecycle correlation anchor — never replaced by the execution-evidence-record identifier. (ADR-0014 Correlation Model)
14. The **evidence-durability value** describes retention outcome independently of attempt-outcome and resulting-state-verification — `PERSISTENCE_FAILED` does not revise either. (ADR-0014 Evidence Durability)
15. A `binding_stored_disposition` that is non-permitting must be recorded as-is — never substituted by the authoritative `ALLOW` disposition. (ADR-0014 Minimum Evidence Facts — Conditionally Required)
16. Execution evidence is retained **internally for implementation learning** — no schema publication. (ADR-0015 Schema Publication Boundary; ADR-0014 Schema Publication Boundary)

### MUST NOT INTRODUCE

**Behavior, topology, or contracts explicitly prohibited:**

1. Protocol execution in `basis-adapters` — rejected as Alternative C by ADR-0011
2. Protocol execution in `basis-gateway` — rejected as Alternative D by ADR-0011
3. Protocol execution in `basis-core` — rejected categorically as Alternative E by ADR-0011
4. A new `basis-executor` repository or other new Foundation repository — rejected as Alternative F by ADR-0011
5. A signed execution grant, JWT, nonce, TTL, or one-time-use token for the same-process topology — rejected as Alternative D by ADR-0012
6. Dispatch based on process-memory identity alone without a content-based binding digest — rejected as Alternative A by ADR-0012
7. Duplicate dispatch of the same authorized operation within one process lifetime — ADR-0012 single-consumption guarantee
8. A single flat status enum for execution state — rejected as Alternative A by ADR-0013
9. The retired vocabulary labels `attempted-and-completed`, `attempted-and-failed`, or `partially-applied` — retired by ADR-0013 Decision 4
10. A queue, scheduler, durable store, or hold mechanism between authorization and dispatch — ADR-0016 Boundary of Conclusion
11. Automatic retry, redispatch, or caller-transparent retry orchestration — ADR-0015; ADR-0016 Boundary of Conclusion
12. An execution-evidence schema or `basis-schemas` contract — ADR-0014 Schema Publication Boundary
13. An `enforcement_status` / `enforcement_result` field on `GatewayAuditEvent` — ADR-0014
14. A real OT protocol target, physical device, or cloud service — ADR-0015
15. A protocol/device credential — ADR-0015 Gate 4 Assessment
16. A `basis-producer`-synthesized, regenerated, or field-derived stand-in for `kernel_evidence_id` — the operation-producer role preserves the authoritative gateway-returned identifier only; it never mints one (see §1 Prerequisite)

### OPEN / UNRESOLVED

Items whose architectural posture is documented precisely, not normalized:

| Item | Posture | Basis |
| --- | --- | --- |
| Operational duplicate-execution/idempotency (whether a freshly re-authorized retry can duplicate a prior unconfirmed effect) | OPEN — non-blocking because first slice performs exactly one dispatch per cycle with no automatic retry | ADR-0013 Decision 10; ADR-0016 "Authorization Replay Versus Operational Idempotency" |
| Broader authorization replay/freshness (wall-clock expiry, policy-reload invalidation, restart-surviving replay prevention, replay database) | OPEN globally; NON-BLOCKING FOR THIS TARGET under ADR-0016's stated topology constraints | ADR-0016 Decision, Boundary of Conclusion |
| Whether six-value `ProvenanceClassification` suffices for execution-observed facts | OPEN | ADR-0014 What This ADR Deliberately Leaves Open |
| Exact `OUTCOME_UNKNOWN` reasons as governed, serialized schema fields | OPEN | ADR-0013 Decision 4 note |
| Crash recovery for `NOT_YET_PERSISTED` evidence candidates | OPEN — non-blocking; first slice does not require crash-surviving retention | ADR-0014 Evidence Durability |
| Whether failed evidence retention is retried, and how | OPEN — non-blocking; first slice retains once | ADR-0014 Evidence Durability |
| Eventual execution-evidence schema (field names, types, serialization) | DEFERRED until implementation proves stable shape | ADR-0014 Schema Publication Boundary |
| Separated executor topology (new trust boundary, portable binding, replay/freshness) | DEFERRED | ADR-0011 Decision 5 |
| Future OT protocol execution targets | DEFERRED | ADR-0015 What This ADR Deliberately Leaves Open |

### IMPLEMENTATION FREEDOM

Choices architecture explicitly leaves to implementation for this bounded slice:

1. Internal module names, class names, package structure, and file layout within `basis-producer` for the new logical roles
2. Whether the pre-dispatch record and execution attempt are one evolving in-process structure or two separate objects (ADR-0013 Decision 2: "does not mandate two separate stored objects")
3. The specific Python HTTP client library for REST dispatch (e.g., `httpx`, `requests`, `urllib.request`)
4. The deterministic local test service fixture technology (e.g., `pytest-httpserver`, `httpretty`, a minimal `http.server`-based fixture)
5. The specific HTTP verb, path, and payload schema for the single bounded REST operation shape
6. The exact binding-material field list, their canonical byte encoding within each representation, and the delimiter/concatenation method for combining the two representations before digestion — provided: (a) the digest is deterministic, (b) it covers the semantically meaningful fields of each representation, and (c) it follows the RFC 8785 / sha-256 canonical JSON pattern ADR-0007 established
7. The exact durable retention mechanism for execution evidence (local filesystem, SQLite, or equivalent process-external storage) — for implementation learning only; no schema required. An in-memory object or dictionary may hold the evidence candidate before or during retention, but process memory alone must not be the mechanism that produces `PERSISTED`; a `PERSISTED` outcome requires that the evidence candidate has reached durable storage external to the process. The implementation PR must demonstrate that its successful `PERSISTED` path is not process-memory-only.
8. Whether the optional resulting-state read-back (`STATE_VERIFIED`) is included in the first slice
9. Internal naming for evidence-durability transitions, including any operational logging for `PERSISTENCE_FAILED`
10. Whether the `OUTCOME_UNKNOWN` reason (no-ack-primitive vs. ack-timeout) is preserved as a distinguishable internal field — it must not be promoted to a published schema field, but may be tracked for implementation learning

---

## 7. Implementation Sequence

The following four work items define the PR sequence. Each is bounded and independently reviewable. They are ordered by dependency: each builds on the preceding one. Work Item 1 additionally depended on a prerequisite outside this PR sequence — a `basis-gateway` contract change, not a `basis-producer` work item — described in §1's Prerequisite subsection. That prerequisite is now satisfied, and Work Item 1 is the next, unblocked implementation step; it has not started.

```
Gateway Evidence-ID   →   Work Item 1  →  Work Item 2  →  Work Item 3  →  Work Item 4
Exposure (prerequisite;      (binding)        (executor)       (evidence)      (conformance)
basis-gateway, out of        NEXT /           PLANNED          PLANNED         PLANNED
this plan's scope —          UNBLOCKED
SATISFIED)
```

---

### Work Item 1: Authorization-to-Execution Binding Record

**Repository:** `basis-producer`
**Architectural authority:** ADR-0012 (binding record structure, creation, single-consumption); ADR-0007 (canonicalization pattern reused)
**Role:** Operation-producer role (existing; this work item adds a new responsibility within it)

#### Description

Adds binding record creation and single-consumption enforcement to the operation-producer role within `basis-producer`. This work item introduces no execution — the binding record is created before the gateway call and held in process memory for verification by the executor in Work Item 2.

#### Binding Record Structure

The binding record is an in-process data structure carrying the following semantic fields. IMPLEMENTATION FREEDOM on field names, class structure, and serialization (none is required, since the record is same-process-local per ADR-0012):

- **`binding_id`** — an opaque identifier minted once by the operation-producer role when the record is created, following the same deployment-local uniqueness convention ADR-0007 established for `reference_id`. IMPLEMENTATION FREEDOM on format (UUID4 or equivalent is sufficient).

- **`binding_digest`** — a deterministic content-based digest over two representations, using the ecosystem's existing RFC 8785 / sha-256 / canonical-bytes pattern (ADR-0007's `basis-adapter-evidence-v1` discipline, applied to different material per ADR-0012's express instruction to reuse rather than invent):

  - **Representation 1 — Preserved protocol-shaped operation:** a deterministic canonical encoding of the `ProtocolOperation` that entered the chain at normalization. The canonical encoding must cover the semantically meaningful fields of the operation (e.g., for REST: method, URL, and any header or body fields material to the operation's identity). IMPLEMENTATION FREEDOM on the exact field list and RFC 8785 JSON shape, provided the encoding is deterministic, covers the operation's identity, and would produce a different digest if any of those fields were mutated.

  - **Representation 2 — Normalized authorization request actually submitted:** a deterministic canonical encoding of the fields the operation-producer role composed into its `OperationAwareSubmission` for the gateway call — specifically: `action`, `resource_type`, `resource_id`, and any populated context fields submitted. This is the exact set of fields `operation-producer-and-execution-boundary.md` §4 assigns to the operation-producer role as its submission composition responsibility; the binding digest captures what was actually submitted, not a reconstruction after the fact.

  The two representations' canonical bytes are combined — concatenated with a deterministic delimiter, hashed together, or digested sequentially — and the result is sha-256 digested. IMPLEMENTATION FREEDOM on the exact combination method, provided it is deterministic, order-sensitive (Representation 1 precedes Representation 2), and not ambiguous. The digest algorithm follows ADR-0007's open-algorithm-label convention.

- **`consumption_state`** — `not_dispatched` initially. Transitions to `dispatched` exactly once, atomically within the process, at the dispatch boundary defined in Work Item 2 §D. After transitioning to `dispatched`, the binding record must refuse any further consumption attempt (fail closed).

- **Correlation references** — populated after the gateway round trip completes, by correlating the disposition received back to the binding record: the gateway-generated `correlation_id`, the kernel `trace_id`, and `AuditEvidence.evidence_id` (all three are available on the operation-aware evaluation response — the Gateway Evidence-ID Exposure prerequisite in §1 that previously blocked `evidence_id` is now satisfied). These are attached by reference, never included in the `binding_digest` (ADR-0012: "the disposition is not known at binding-creation time"). `basis-producer` must not populate this reference with a synthesized, derived, or inferred value — if a given evaluation's response omits `evidence_id` (no authoritative `AuditEvidence` was produced for it), the reference is absent, never substituted (§1 Provenance constraint).

#### Where and When Created

The operation-producer role creates the binding record once, **after** both Representation 1 and Representation 2 are fully determined, and **before** the gateway call. In `rest_composition.py`'s `RestOperationComposer.submit()`, this is immediately after `AdapterEvidenceReference` assembly (when both the preserved `ProtocolOperation` and the `OperationAwareSubmission` are known) and before the `GatewayClient.submit()` call.

IMPLEMENTATION FREEDOM on whether this is achieved by extending the existing `RestOperationComposer.submit()` method signature, extracting an executor-aware entrypoint, or another approach — provided creation order is correct.

#### What Must Not Change

- The binding record is **not submitted to `basis-gateway`** and does not appear on the operation-aware request or response (ADR-0012 "The binding record").
- No `basis-schemas` change is introduced by the binding record; it is same-process-local (ADR-0012). This is distinct from the Gateway Evidence-ID Exposure prerequisite in §1: that prerequisite was a change to the gateway's *response* (now satisfied), not to the binding record described here, which remains same-process-local either way.
- The existing mTLS producer workload authentication and bearer authorization-subject authentication topology is unchanged (ADR-0008, ADR-0009).
- `basis-adapters` is unchanged (ADR-0011 Decision 16).

#### Traceability

| Governing source | Required property | Planned implementation responsibility | Validation |
| --- | --- | --- | --- |
| ADR-0012 "What is bound" | Binding digest covers preserved ProtocolOperation + normalized submission actually evaluated | Digest computation in new binding-record module | Unit test: digest differs when either representation is mutated |
| ADR-0012 "Where the binding is created" | Created by operation-producer role, before gateway call | Wired into RestOperationComposer before GatewayClient.submit() | Unit test: binding record exists before any gateway call; not sent to gateway |
| ADR-0012 "The binding record" | Same-process-local; no wire transmission; no basis-schemas change | In-process data structure only | Code inspection; no new wire field; no schema export |
| ADR-0012 "Freshness and single use" | Single-consumption: at most one dispatch per process lifetime | consumption_state field; fail-closed on second consumption | Unit test: second consumption attempt fails closed |
| ADR-0007 canonicalization | Reuse RFC 8785 / sha-256 pattern | Binding digest follows basis-adapter-evidence-v1 canonicalization discipline | Unit test: digest is deterministic for same input |

---

### Work Item 2: Protocol-Executor Logical Role with Local Test Service and Lifecycle Observation

**Repository:** `basis-producer`
**Architectural authority:** ADR-0011 (executor role definition, governing invariants), ADR-0012 (binding verification and failure semantics), ADR-0013 (pre-dispatch record, lifecycle vocabulary, three-axis model), ADR-0015 (target definition and demonstrable outcomes), ADR-0016 (topology constraints)
**Role:** Protocol-executor logical role (new, logically distinct from operation-producer role per ADR-0011 Decision 2)

#### Description

Implements the protocol-executor logical role inside `basis-producer`'s process boundary. This work item adds: the deterministic local test service, the pre-dispatch record, binding verification, REST dispatch, and lifecycle observation. It does not add evidence construction or retention (Work Item 3).

#### A. Deterministic Local Test Service

A deterministic local HTTP test service fixture, implemented within `basis-producer`'s test infrastructure, that functions as the credential-free controlled reference endpoint ADR-0015 requires. IMPLEMENTATION FREEDOM on the fixture technology.

**Required behaviors** — the fixture must be configurable at test time to produce each of the following for the single bounded REST operation shape:

| Behavior | Exercised outcome |
| --- | --- |
| Return a 2xx HTTP response | `PROTOCOL_ACCEPTED` |
| Return a 4xx or 5xx HTTP response | `PROTOCOL_REJECTED` |
| Not respond within the executor's observation window, or close the connection | `OUTCOME_UNKNOWN` |
| Optionally: confirm resulting state via a separate read-back endpoint | `STATE_VERIFIED` (if included) |

The fixture must:
- Require **no credential** of any kind (no bearer token, API key, mTLS client certificate, or other endpoint-specific secret) — ADR-0015 Gate 4 Assessment
- Be reachable at **localhost only** (never a network-accessible or production address)
- Have **deterministic, test-controlled behavior** (no dependence on external state or infrastructure)

**Single bounded REST operation shape:** IMPLEMENTATION FREEDOM on the specific HTTP verb, path, and payload schema, subject to: (a) it is one tightly bounded shape (not a multi-operation surface), (b) it can be reproduced identically across the authorize and execute paths, and (c) it can exercise all required outcome categories through the fixture's behavioral configuration. A safe, idempotent write to a test "device register" endpoint is appropriate. The same operation shape must be used for both the normalization input (the `ProtocolOperation` ADR-0012's Representation 1 covers) and the dispatch target.

#### B. Pre-Dispatch Record

When an authoritative `ALLOW` disposition reaches the protocol-execution boundary, the protocol-executor role **enters the pre-dispatch boundary** and immediately creates a **pre-dispatch record** (ADR-0013 Decision 2). This must happen before binding verification — not after it and not conditioned on verification succeeding.

The pre-dispatch record is the **lifecycle correlation anchor** for the entire execution cycle (ADR-0014 Correlation Model): a pre-dispatch record/context identifier is minted at creation time and remains the stable correlation identifier regardless of whether the cycle resolves to Case C `NOT_ATTEMPTED` or a committed dispatch.

IMPLEMENTATION FREEDOM on whether the pre-dispatch record and the eventual execution attempt are one evolving in-process data structure or two separate objects (ADR-0013 Decision 2: "does not mandate two separate stored objects"). What this plan requires: the concept "dispatch was never attempted" is never modeled as a state an execution attempt transitions out of, and the pre-dispatch record/context identifier persists as the correlation anchor in both resolution paths.

**Fail-closed on record creation failure:** If the pre-dispatch record cannot be created for any reason, the operation resolves to `NOT_ATTEMPTED` (Case B), no binding verification occurs, no dispatch occurs, and no execution-evidence record is produced for this condition. The inability to create the record is itself a fail-closed condition (ADR-0013 Decision 16).

#### C. Binding Verification

Using the pre-dispatch record, the protocol-executor role verifies the binding record created in Work Item 1:

1. **Locate the binding record** for the operation entering the pre-dispatch boundary. If no binding record is found → fail closed.
2. **Recompute the binding digest** over: (a) the `ProtocolOperation` instance about to be dispatched (the same preserved object the operation-producer role retained), and (b) the normalized request the process holds for it — using the same canonicalization procedure established in Work Item 1.
3. **Compare** the recomputed digest to the stored `binding_digest`. If they differ → fail closed.
4. **Verify** the disposition correlated to the binding record is a permitting disposition (`ALLOW`). If not → fail closed.
5. **Verify** `consumption_state` is `not_dispatched`. If already consumed → fail closed.
6. If verification itself raises an error or cannot complete → fail closed.

Any fail-closed condition resolves the pre-dispatch record to `NOT_ATTEMPTED` (ADR-0013 Decision 2 Case C). The binding record's `consumption_state` is NOT marked consumed on a verification failure — an unconsumed binding record is not consumed merely because verification failed (ADR-0012 "Freshness and single use"). All verification failures are definitively pre-dispatch: no dispatch was initiated.

On successful verification: proceed to dispatch (§D below). The binding record is not marked consumed at verification success — consumption occurs atomically at the dispatch boundary.

#### D. REST Dispatch and Dispatch-Boundary Semantics

Using a standard Python HTTP client library (IMPLEMENTATION FREEDOM), dispatch the single bounded REST operation to the local test service endpoint.

**Dispatch boundary:** The dispatch boundary is the moment the HTTP request is actually committed to the endpoint — the irreversible act that begins the execution attempt (ADR-0013 Decision 2). At this boundary, mark the binding record's `consumption_state` as `dispatched`. From this point, `NOT_ATTEMPTED` no longer applies to this attempt; the attempt-outcome begins at `DISPATCHED` (ADR-0013 Decision 4 — transient) and proceeds to lifecycle observation (§E).

**Pre-dispatch vs. post-dispatch failure classification:** The implementation must distinguish between failures that are definitively pre-dispatch and failures that occurred at or after the dispatch boundary. These two cases have different semantics and must not be conflated:

1. **Definitely pre-dispatch failure** — The implementation can establish with certainty that no dispatch occurred and no operation bytes were committed to the endpoint. This is the case when: the selected HTTP client library raises an exception that the library guarantees occurs before request initiation (e.g., connection refused before any bytes are sent, DNS resolution failure, TLS handshake failure before HTTP send). In this case:
   - The pre-dispatch record resolves as Case C `NOT_ATTEMPTED`
   - The binding record remains unconsumed (`not_dispatched`)
   - A Case 2 execution-evidence candidate is produced with a `failure_cause` identifying the specific pre-dispatch condition

2. **Dispatch committed or may have occurred** — The HTTP request was initiated (the dispatch boundary was crossed), or the implementation cannot establish with certainty that no bytes were committed. In this case:
   - Consume the binding at the dispatch boundary
   - The attempt begins at `DISPATCHED`
   - If the result cannot be determined (no response received, connection closed after initiation, observation timeout), resolve to `OUTCOME_UNKNOWN` — never to `NOT_ATTEMPTED`

**Conservative classification required:** The implementation must not assume that every HTTP connection exception proves that no bytes were sent. HTTP client libraries differ in when they raise exceptions and whether those exceptions guarantee pre-dispatch failure. If the selected library does not guarantee that a given exception type occurs before any bytes are transmitted, the implementation must classify conservatively: treat the failure as having occurred at or after dispatch and resolve to `OUTCOME_UNKNOWN`. Claiming `NOT_ATTEMPTED` when dispatch may have occurred is a false assertion about execution state — `OUTCOME_UNKNOWN` is the truthful alternative (ADR-0013 Decision 16).

#### E. Lifecycle Observation

Observe the protocol result and assign the terminal attempt-outcome value:

| Observed protocol result | Attempt-outcome value | Notes |
| --- | --- | --- |
| 2xx HTTP response received | `PROTOCOL_ACCEPTED` | Protocol acknowledgement only — not proof of resulting physical state (ADR-0013 Decision 13) |
| 4xx or 5xx HTTP response received | `PROTOCOL_REJECTED` | Definitive negative response |
| No response within observation window, or connection closed/failure after the dispatch boundary | `OUTCOME_UNKNOWN` | For REST, this is the ack-timeout reason; the protocol always has a response primitive, so no-ack-primitive does not apply. Applies once dispatch has been committed or cannot be ruled out. |
| Observation mechanism itself fails after dispatch boundary | `OUTCOME_UNKNOWN` | Must never default to `PROTOCOL_ACCEPTED` (ADR-0013 Decision 16) |

`DISPATCHED` must not persist as a terminal value. If the executor stops waiting without a definitive answer, it must resolve to `OUTCOME_UNKNOWN`.

**Resulting-state verification axis** (ADR-0013 Decision 5):

- For Case C `NOT_ATTEMPTED`: `VERIFICATION_NOT_APPLICABLE` (semantically fixed; dispatch never occurred)
- For dispatched attempts with no read-back or no read-back capability: `STATE_NOT_VERIFIED`
- For dispatched attempts where the optional read-back endpoint confirms resulting state: `STATE_VERIFIED`
- IMPLEMENTATION FREEDOM on whether the optional read-back is included in this work item. If deferred, default to `STATE_NOT_VERIFIED` for all dispatched outcomes

#### Traceability

| Governing source | Required property | Planned implementation responsibility | Validation |
| --- | --- | --- | --- |
| ADR-0011 Decision 1 | Executor verifies binding before dispatch | Binding verification step as gate before dispatch | Test: binding failure → no dispatch |
| ADR-0011 Decision 7 | Original ProtocolOperation preserved through dispatch | Executor accesses same preserved object from operation-producer role | Code inspection: no reconstruction from gateway representation |
| ADR-0011 Decision 9 | No dispatch if DENY or other pre-boundary failure | Pre-boundary conditions never reach the executor; Case A exits before executor is invoked | Test: DENY → no executor entry |
| ADR-0012 Failure Semantics | All six binding-verification failure conditions → fail closed | Each condition verified, operation resolves to NOT_ATTEMPTED | Unit test per failure condition |
| ADR-0012 single-consumption | Binding consumed once at dispatch; not on verification failure | consumption_state marked dispatched at commit; unconsumed on verification failure | Test: second consumption fails; failed verification does not consume |
| ADR-0013 Decision 2 | Pre-dispatch record created before binding verification | Module creates record as first action on boundary entry | Code: record creation precedes verification call |
| ADR-0013 Decision 4 | Correct attempt-outcome vocabulary for each protocol result | Lifecycle observation assigns correct terminal value | Integration tests with fixture in each behavioral mode |
| ADR-0013 Decision 16 | Observation failure → OUTCOME_UNKNOWN; conservative classification when pre-dispatch cannot be established with certainty | Error handling in observation step; dispatch-boundary classification rule in §D | Test: simulated observation failure → OUTCOME_UNKNOWN; Test: ambiguous connection exception (cannot rule out dispatch) → OUTCOME_UNKNOWN, not NOT_ATTEMPTED |
| ADR-0015 Decision | Credential-free, local test service, one REST operation shape | Fixture: no credential; localhost only; one operation | Fixture inspection; no credential in any test configuration |
| ADR-0015 Demonstrable outcomes | All required outcome categories exercisable | Fixture supports accept, reject, timeout behaviors | Integration tests per category |
| ADR-0016 Boundary of Conclusion | No queue/scheduler between auth and dispatch; one dispatch per cycle | No queue introduced; no automatic retry | Code inspection; topology constraint test in Work Item 4 |

---

### Work Item 3: Execution-Evidence-Producer Logical Role with Evidence Construction and Retention

**Repository:** `basis-producer`
**Architectural authority:** ADR-0013 Decision 14 (observation ownership), ADR-0014 (evidence definition, ownership, minimum facts, correlation model, evidence-durability axis)
**Role:** Execution-evidence-producer logical role (new, logically distinct from protocol-executor role)

#### Description

Implements the execution-evidence-producer logical role inside `basis-producer`'s process boundary. This role receives the protocol-executor role's observations and constructs and retains execution-evidence candidates. It is logically distinct from the protocol-executor role (ADR-0011 Decision 2's "same-process does not mean same responsibility" principle, applied to evidence production) and does not itself dispatch or observe protocol results.

#### A. Evidence Production Scope

Execution evidence is produced for exactly two cases (ADR-0014 Execution Evidence Definition). The implementation must distinguish them and must not produce evidence for conditions outside these two:

| Case | Condition | Evidence produced? |
| --- | --- | --- |
| **Case 1** | Execution attempt reached dispatch (attempt-outcome is PROTOCOL_ACCEPTED, PROTOCOL_REJECTED, or OUTCOME_UNKNOWN) | Yes |
| **Case 2** | Pre-dispatch execution-boundary failure: a pre-dispatch record existed and a boundary-local condition (ADR-0013 Decision 2's Case C — binding-verification failure, endpoint unreachable before dispatch, or other executor-local pre-dispatch failure) prevented dispatch | Yes |
| Case A | Pre-boundary failure (DENY, evaluation failure, producer auth/admission failure, gateway unavailability) — protocol executor was never invoked | **No** — these are already recorded in GatewayAuditEvent / AuditEvidence |
| Case B | Pre-dispatch record creation failure — no record ever existed | **No** — nothing to correlate evidence against |

#### B. Minimum Evidence Facts

Each evidence candidate must carry the following facts. These are semantic minimums per ADR-0014; field names, types, and serialization format are IMPLEMENTATION FREEDOM (no schema is defined or published by this plan). Where a conditionally-required fact's source does not exist, the field must be absent — never fabricated, defaulted to a placeholder, or substituted with a different fact.

**Always required for both Case 1 and Case 2:**

- `execution_evidence_record_id` — opaque identifier minted by the execution-evidence-producer role; never replaces the `pre_dispatch_record_id`
- `evidence_case` — whether this record describes Case 1 (dispatched attempt) or Case 2 (pre-dispatch boundary failure)
- `attempt_outcome` — the ADR-0013 attempt-outcome value: for Case 1, one of `PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, `OUTCOME_UNKNOWN`; for Case 2, always `NOT_ATTEMPTED`
- `resulting_state_verification` — the ADR-0013 resulting-state-verification value: for Case 2, always `VERIFICATION_NOT_APPLICABLE`; for Case 1, whichever value the executor reached
- `evidence_durability` — the evidence-durability value (see §C below)
- `pre_dispatch_record_id` — the lifecycle correlation anchor; present for both cases because both are constructed only when a pre-dispatch record exists
- `pre_dispatch_record_creation_time` — when the pre-dispatch record was created
- `observation_resolution_time` — when the attempt outcome or boundary-failure was determined
- `authoritative_disposition` — the `ALLOW` disposition that caused boundary entry
- `gateway_correlation_id` — the gateway-generated correlation ID from the authorization round trip
- `kernel_trace_id` — the `trace_id` from the authorization round trip
- `kernel_evidence_id` — `AuditEvidence.evidence_id` from the authorization round trip. **The Gateway Evidence-ID Exposure prerequisite (§1) is now satisfied:** the released gateway response returns this identifier to the caller, so this fact can be populated as specified. This work item must still not populate it with a synthesized, derived, or inferred value (§1 Provenance constraint) — if a given evaluation's response omits `evidence_id` (no authoritative `AuditEvidence` was produced for it), this fact is absent, never substituted with another field.

**Case 2 only — additionally always required:**

- `failure_cause` — the specific boundary-local failure reason under ADR-0013 Decision 2's Case C: one of the ADR-0012 binding-verification failure conditions (no binding record found; digest mismatch; non-permitting correlated disposition; already-consumed binding; verification error) or another protocol-executor-local pre-dispatch condition (endpoint unreachable before dispatch, transport/connection timeout before dispatch). The `failure_cause` and `attempt_outcome` must be preserved as two separate facts — they must not be collapsed. `attempt_outcome = NOT_ATTEMPTED` states that dispatch did not occur; `failure_cause` states specifically why.

**Conditionally required — only when the source fact exists:**

- `binding_id` — the ADR-0012 binding record identifier, when a binding record was found at verification time. **Omitted** (not fabricated or defaulted) when the Case 2 failure reason is "no binding record found"
- `binding_stored_disposition` — the disposition stored in or associated with the binding record, when a binding record was found. **Must be recorded as the actual stored value** — including a non-permitting or mismatched value. Must never be substituted by the authoritative `ALLOW` disposition; substitution would make a security failure look like a permitting fact
- `binding_digest_stored` and `binding_digest_recomputed` — when a digest mismatch is the Case 2 failure reason and a binding record was found
- `adapter_evidence_reference_id` — `AdapterEvidenceReference.reference_id` from the authorization-side artifact, when the operation-producer role holds one for this operation

#### C. Evidence-Durability Axis

The evidence-durability axis tracks retention outcome independently of the other two axes (ADR-0014 Evidence Durability and Persistence Failure). It must never be conflated with attempt-outcome or resulting-state-verification.

| Value | Meaning | Terminal? |
| --- | --- | --- |
| `NOT_YET_PERSISTED` | Observational facts known in-process; retention not yet attempted or not yet complete | No, while process is running and retention is pending |
| `PERSISTED` | Retention attempt completed successfully | Yes |
| `PERSISTENCE_FAILED` | Retention attempt was made and did not succeed | Yes, for the observed retention attempt |

A `PERSISTENCE_FAILED` value must **never** revise, retract, or cast doubt on an already-reached `attempt_outcome` or `resulting_state_verification` value (ADR-0014). The observational facts are fixed at the moment the executor reaches them.

IMPLEMENTATION FREEDOM on whether a `PERSISTENCE_FAILED` event triggers any additional operational communication (log entry, etc.).

#### D. Retention Mechanism

The execution-evidence-producer role retains the evidence candidate using a durable retention mechanism appropriate for implementation learning. The exact durable technology is IMPLEMENTATION FREEDOM (local filesystem, SQLite, or equivalent process-external storage). An in-memory object or dictionary may hold the evidence candidate before or during retention, but process memory alone must not serve as the mechanism that produces `PERSISTED`: a `PERSISTED` outcome requires that the evidence candidate has actually reached durable storage external to the process.

`basis-producer` already contains `FilesystemEvidenceBlobStore`, a local-filesystem retention implementation used for adapter evidence (Phases 2A–5). This is noted as current-state implementation evidence only — it confirms that a durable local-filesystem approach is already present in the codebase — but does not automatically authorize reuse of that exact class or its interface for execution evidence. The implementation PR is free to choose any durable mechanism, including a new one or an extension of the existing one, provided it demonstrates that successful retention reaches durable storage rather than process memory.

The implementation PR must include a demonstration or test confirming that its successful `PERSISTED` path is not process-memory-only.

The retention discipline is: attempt retention once; record whichever of the three durability values results; do not revise observational facts based on retention outcome. IMPLEMENTATION FREEDOM on whether and how a `PERSISTENCE_FAILED` outcome is logged or communicated operationally.

ADR-0014 explicitly does not require retention-retry logic. This plan does not introduce it. The first slice retains once and reports the resulting durability value.

#### E. Evidence Isolation Requirements

The implementation must enforce:

1. Execution evidence **never nests inside, mutates, or completes** `AuditEvidence` or `GatewayAuditEvent`
2. No `enforcement_status` / `enforcement_result` field is added to `GatewayAuditEvent`
3. The execution-evidence record correlates to authorization evidence **by reference only** (`kernel_evidence_id`, `gateway_correlation_id`) — never by embedding

#### F. Correlation Model

The execution-evidence record extends the existing identifier chain by exactly one hop (ADR-0014 Correlation Model):

```
AdapterEvidenceReference.reference_id    [operation-producer role, ADR-0007]
    → gateway_correlation_id              [basis-gateway, always generated]
        → kernel_trace_id                 [basis-gateway-generated, embedded by basis-core]
            → kernel_evidence_id          [AuditEvidence.evidence_id]
                → authoritative ALLOW disposition
                    → binding_id          [operation-producer role; conditional — present only when
                                           binding record was found at verification time]
                    → pre_dispatch_record_id   [protocol-executor role; lifecycle correlation anchor;
                                                always present for both Case 1 and Case 2]
                        ├── Case 2 evidence record   (execution_evidence_record_id, from evidence-producer role)
                        └── Case 1 evidence record   (execution_evidence_record_id, from evidence-producer role)
```

Governing rule (ADR-0014, restating `operation-producer-and-execution-boundary.md` §6): **no component may overwrite an identifier owned by another component**. The `execution_evidence_record_id` is minted by the execution-evidence-producer role and appended; it does not replace the `pre_dispatch_record_id` or any upstream identifier.

#### Traceability

| Governing source | Required property | Planned implementation responsibility | Validation |
| --- | --- | --- | --- |
| ADR-0014 Execution Evidence Definition | Evidence for Case 1 and Case 2 only — not for Case A or Case B | Evidence module produces only for those two cases; Case A and Case B produce no evidence | Test: Case A → no evidence record; Case B → no evidence record |
| ADR-0014 Ownership | Construction = execution-evidence-producer role (distinct from executor) | Separate logical module; executor passes observations; evidence producer assembles | Code: evidence module does not dispatch or observe directly |
| ADR-0014 Minimum Evidence Facts | All always-required and applicable conditionally-required facts | Field population per §B above | Unit test per required field; absent-not-fabricated for conditional fields |
| ADR-0014 Minimum Evidence Facts — Conditionally Required | binding_stored_disposition recorded as actual stored value, not substituted | Conditional field populated from binding record's stored value | Test: "non-permitting correlated disposition" Case 2 records the non-permitting value |
| ADR-0014 Correlation Model | Identifier chain extended by one hop; no overwriting | evidence_record_id appended; all upstream IDs referenced | Test: all upstream IDs present in evidence; no upstream ID overwritten |
| ADR-0014 Evidence Durability | Three-value axis tracked independently from attempt-outcome | Durability field updated based on retention result; PERSISTENCE_FAILED does not revise attempt_outcome | Test: PERSISTENCE_FAILED → attempt_outcome unchanged |
| ADR-0014 Schema Publication Boundary | No schema export; retention internal | No schema file produced; no basis-schemas directory created or modified | Code inspection |
| ADR-0011 Decision 11 / ADR-0014 | Evidence never mutates AuditEvidence or GatewayAuditEvent | Evidence module reads but never writes to those artifacts | Test: AuditEvidence and GatewayAuditEvent unchanged after evidence construction |

---

### Work Item 4: End-to-End Conformance, Demonstration, and Topology Validation

**Repository:** `basis-producer`
**Architectural authority:** ADR-0011 (all invariants), ADR-0012, ADR-0013, ADR-0014, ADR-0015 (required demonstrable outcomes), ADR-0016 (topology constraints)

#### Description

Exercises the complete `authorize → bind → execute → evidence` path through conformance tests and a reproducible demonstration. Verifies that all required outcome categories are demonstrable, all minimum evidence facts are present, evidence is isolated from authorization artifacts, and the topology constraints from ADR-0016 are respected.

#### A. Conformance Test Suite

The conformance suite must cover the following categories. All must pass:

| Conformance category | What is verified | Governing source |
| --- | --- | --- |
| `not_attempted_pre_boundary` | DENY or authorization-side failure → NOT_ATTEMPTED (Case A); no execution-evidence record produced; existing GatewayAuditEvent / AuditEvidence unchanged | ADR-0013 Decision 2 Case A; ADR-0014 |
| `not_attempted_binding_failure` | Each ADR-0012 failure condition → NOT_ATTEMPTED (Case C); execution-evidence record (Case 2) produced with correct failure_cause; binding_stored_disposition recorded as actual stored value for non-permitting case | ADR-0012 Failure Semantics; ADR-0013 Decision 2 Case C; ADR-0014 |
| `dispatched_accepted` | Complete path with accept fixture → PROTOCOL_ACCEPTED; Case 1 execution-evidence record with all always-required facts present | ADR-0013 Decision 4; ADR-0014 |
| `dispatched_rejected` | Complete path with reject fixture → PROTOCOL_REJECTED; Case 1 execution-evidence record | ADR-0013 Decision 4; ADR-0014 |
| `dispatched_outcome_unknown` | Complete path with timeout fixture → OUTCOME_UNKNOWN; Case 1 execution-evidence record; OUTCOME_UNKNOWN does not imply NOT_ATTEMPTED | ADR-0013 Decision 4; ADR-0014 |
| `evidence_durability_persisted` | Successful retention → PERSISTED; attempt_outcome unchanged | ADR-0014 Evidence Durability |
| `evidence_durability_persistence_failed` | Simulated retention failure → PERSISTENCE_FAILED; attempt_outcome and resulting_state_verification unchanged | ADR-0014 Evidence Durability |
| `evidence_separation` | Execution evidence does not appear in or modify AuditEvidence or GatewayAuditEvent; authorization facts unchanged after execution | ADR-0011 Decision 11; ADR-0014 |
| `single_consumption` | A consumed binding record refuses a second dispatch (both from an explicit second attempt and from the same attempt retrying) | ADR-0012 single-consumption guarantee |
| `topology_constraints` | No queue/scheduler between authorization and dispatch; exactly one dispatch per cycle; no automatic retry triggered by the implementation | ADR-0016 Boundary of Conclusion |

If the optional `STATE_VERIFIED` read-back was included in Work Item 2:

| `state_verified` | Complete path with accept fixture and read-back → STATE_VERIFIED in Case 1 evidence | ADR-0013 Decision 5; ADR-0015 |

#### B. Demonstration

A reproducible demonstration (analogous to Phase 5's `test_bounded_slice_demo.py`) showing the complete path with inspectable, human-readable output for:

1. The `ProtocolOperation` input and its resulting `ALLOW` disposition
2. The binding record: `binding_id`, `binding_digest`, and `consumption_state` before and after dispatch
3. The pre-dispatch record and its creation timestamp
4. The dispatch action and the observed attempt-outcome and resulting-state-verification values
5. The execution-evidence candidate: all minimum facts populated; `evidence_durability` value after retention

The demonstration must show the complete correlation chain, traceable from `adapter_evidence_reference_id` through `kernel_evidence_id` through `pre_dispatch_record_id` to `execution_evidence_record_id`.

#### C. Topology Constraint Verification

Explicit checks confirming the ADR-0016 topology constraints hold for the bounded implementation:

- No binding record, disposition, or equivalent authorization artifact is persisted beyond process memory or transmitted outside the process
- No queue, scheduler, or deliberate hold mechanism exists between disposition receipt and dispatch
- The implementation performs at most one dispatch attempt per `authorize → execute → evidence` cycle; no automatic retry or redispatch is triggered by the implementation itself

#### Traceability

| Governing source | Required property | Planned implementation responsibility | Validation |
| --- | --- | --- | --- |
| ADR-0015 Demonstrable outcomes | All five required outcome categories exercisable and demonstrated | Conformance suite and demonstration cover all five | All five conformance categories pass |
| ADR-0014 Minimum Evidence Facts | All always-required facts present in produced evidence | Conformance asserts minimum fact completeness for both cases | Assertion on all required fields for each case |
| ADR-0016 Boundary of Conclusion | Topology constraints hold | Topology constraint conformance category | Explicit topology constraint assertions pass |
| ADR-0011 Decision 11; ADR-0014 | Evidence separation | Evidence separation conformance category | AuditEvidence and GatewayAuditEvent unchanged assertions |

---

## 8. What Remains Open

The following questions remain architecturally unresolved after this plan is complete and the bounded slice is implemented. This plan does not answer them and must not implicitly introduce answers through implementation choices. Where a question arises during implementation that appears to require one of these decisions, it must be surfaced for architecture review rather than resolved locally.

| Open question | Posture | Basis |
| --- | --- | --- |
| Operational duplicate-execution/idempotency (whether a freshly re-authorized retry can duplicate a prior unconfirmed effect) | OPEN — non-blocking because first slice performs exactly one dispatch per cycle with no automatic retry | ADR-0013 Decision 10; ADR-0016 "Authorization Replay Versus Operational Idempotency" |
| Broader authorization replay/freshness for topologies other than the ADR-0015 target | OPEN globally | ADR-0016 "What This Decision Does Not Claim" |
| Whether the six-value `ProvenanceClassification` enum suffices for execution-observed facts | OPEN | ADR-0014 What This ADR Deliberately Leaves Open |
| Exact `OUTCOME_UNKNOWN` distinguishable reasons as governed, serialized schema fields | OPEN | ADR-0013 Decision 4 note |
| Crash recovery for `NOT_YET_PERSISTED` evidence candidates across a process restart | OPEN — non-blocking; first slice does not require crash-surviving retention | ADR-0014 Evidence Durability |
| Whether failed evidence retention is ever retried; if so, how a retry is represented and correlated | OPEN — non-blocking; first slice retains once | ADR-0014 Evidence Durability |
| Eventual execution-evidence schema (field names, types, serialization format, profile identifier) | DEFERRED until bounded implementation proves a stable shape | ADR-0014 Schema Publication Boundary |
| Separated executor topology (authenticated trust boundary, portable binding artifact, replay/freshness for separated topology) | DEFERRED | ADR-0011 Decision 5 |
| Future OT protocol execution targets | DEFERRED | ADR-0015 What This ADR Deliberately Leaves Open |

---

## 9. Schema Publication Boundary

This plan does not authorize any `basis-schemas` publication. Consistent with `ecosystem-contract-inventory.md`'s governing "implementation proves a stable shape" principle, adopted normatively by ADR-0014, execution evidence produced by this bounded reference implementation is retained internally for implementation learning only. The semantic minimum specified in Work Item 3 (§7) is the architectural floor a future schema must satisfy — it is not a candidate schema and does not predetermine field names or serialization format.

Schema publication follows only after the bounded reference implementation provides sufficient evidence to stabilize the shape, per discovery-assessment §20's recommended sequence:

```
discovery assessment                         ← done
execution-model ADRs (0011–0016)             ← done (Accepted)
normative specifications (ADR-0012–0014)     ← done (Accepted)
this bounded implementation plan             ← this document
bounded reference implementation             ← next
conformance, demonstration, review           ← after implementation
shared schema publication                    ← only once shape is stable
```

This plan is the "implementation plan" step in that sequence. It does not shorten the sequence by skipping ahead.

---

## 10. Gate Status at Plan Authoring Time

For reference, the gate posture established by the Accepted ADR chain at the time this plan was authored:

| Gate | Posture | Governing source |
| --- | --- | --- |
| Gate 1 — Same-process authorization-to-execution binding mechanism | RESOLVED | ADR-0012 (Accepted) |
| Gate 1 — Process-local single-consumption freshness | RESOLVED | ADR-0012 (Accepted) |
| Gate 1 — Broader replay/freshness for ADR-0015 target | NON-BLOCKING FOR THIS TARGET (OPEN globally for other topologies) | ADR-0016 (Accepted) |
| Gate 2 — Execution-lifecycle semantics | RESOLVED | ADR-0013 (Accepted) |
| Gate 3 — Minimum execution-evidence semantics | RESOLVED | ADR-0014 (Accepted, together with ADR-0013) |
| Gate 4 — Credential custody for ADR-0015 target | NOT TRIGGERED (target is credential-free) | ADR-0015 Gate 4 Assessment (Accepted) |
| Operational duplicate-execution/idempotency | OPEN — non-blocking for this target because bounded implementation performs exactly one dispatch per cycle | ADR-0016 |
| Prerequisite — Gateway evidence_id exposure (blocked Work Item 1 completion at plan-authoring time) | NOT YET SATISFIED **at plan-authoring time** — the then-released `basis-gateway` operation-aware evaluation response did not expose `AuditEvidence.evidence_id` to the caller. Not a reopening of Gate 1: the same-process binding *mechanism* itself remained RESOLVED by Accepted ADR-0012 throughout | §1 Prerequisite subsection, above; ADR-0012 "The binding record"; ADR-0014 Minimum Evidence Facts |

**Current state (post plan-authoring):** The Gateway Evidence-ID Exposure prerequisite above is now **SATISFIED**. `basis-gateway` (revision `1e08348483944d1859739cdf1d1cc16d31ce518a`) exposes `AuditEvidence.evidence_id` on the operation-aware evaluation response, so Work Item 1's binding-record correlation references and Work Item 3's `kernel_evidence_id` fact can now be populated as specified. See §1 for the current-state detail. This table's other rows continue to reflect the current gate posture unchanged since plan authoring.
