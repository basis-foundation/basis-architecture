# ADR-0012: Authorization-to-Execution Binding

## Status

Accepted

## Context

[ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — Accepted — established a distinct logical protocol-executor role, architecturally separate from the operation-producer role, and selected same-process colocation with the existing `basis-producer` reference implementation as the default topology for the first bounded execution reference slice. It fixed the invariants execution must preserve regardless of topology, but it explicitly did not select a binding mechanism: its Decision 8 states that "no protocol dispatch may occur unless the execution attempt is bound to the exact operation that received the authoritative authorization disposition," and names this as **Gate 1 — Authorization-to-Execution Binding**, one of the unconditional prerequisites (with Gates 2 and 3, and Gate 4 if applicable) before any bounded execution implementation may dispatch a protocol operation.

[`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) — the evidentiary basis for ADR-0011 — analyzed this binding problem in detail (§5, §6, §7, §12, §13) and reached several findings this ADR treats as settled rather than re-derives: the binding problem is not one of *deriving* a dispatchable operation from the gateway's authorization-side representation — that framing was considered and rejected as risking a second, redundant protocol-mapping path outside `basis-adapters` (§5, §6); the producer path already begins with a real, protocol-shaped `ProtocolOperation`, and the binding question is how BASIS preserves that operation through the authorization chain and proves, immediately before dispatch, that the operation about to be dispatched is the operation whose normalized semantics were authorized; nothing in the current architecture requires inventing a new signed execution token for a same-process topology specifically, though the assessment does not conclude that no new artifact is needed at all (§6); and no existing identifier in the chain — the gateway's correlation ID, the kernel's `trace_id`/`evidence_id`, or the `AdapterEvidenceReference` digest — is integrity-checked at a trust boundary today, or was designed to serve as one (§6, §13).

`basis-producer`'s completed bounded authorization slice (Phase 2A through Phase 5, merged) proves that the process retains the adapter-normalized operation from normalization through gateway submission, but nothing in that implementation binds the specific operation object a future executor would dispatch to the specific disposition the process received for it — the bounded slice deliberately stops at the disposition (`rest_composition.py`'s own module docstring: no execution "under any disposition, including `ALLOW`"). This ADR is the normative specification Gate 1 requires. It is scoped to the same-process first-reference topology ADR-0011 selected; it does not design a binding mechanism for a future separated executor.

Consistent with this repository's ADR-acceptance governance convention (ADR-0007 through ADR-0011 were each first merged with `Status: Proposed`, and a separate, dedicated follow-up PR later changed the status to `Accepted` after independent architecture and governance review — see [`docs/adr/README.md`](README.md#lifecycle-states)), this ADR was submitted as `Proposed` and has since undergone that same separate, dedicated formal-acceptance review; its status now records `Accepted`. Acceptance resolves the same-process authorization-to-execution binding mechanism this ADR specifies, and narrows Gate 1's replay/freshness requirement to the process-local single-consumption guarantee the same-process first-reference topology needs — it does not resolve broader replay/freshness semantics, which remain open pending later architecture. Acceptance does not itself authorize bounded execution implementation: per the **Validation / Implementation Gate** section below, Gates 2 and 3 remain outstanding regardless, and Gate 4 remains conditionally outstanding if the eventually selected bounded target requires a protocol/device credential.

## Problem

Immediately before dispatch, how does BASIS prove that the exact preserved protocol-shaped operation about to be executed is the same operation whose normalized semantics received the authoritative permitting disposition? The canonical threat this ADR must make fail closed: `authorize operation A → mutation/substitution → execute operation B`. The answer must be precise enough to guide a first bounded REST implementation, must not presuppose a separated-executor topology it does not yet exist to serve, and must not claim more than the evidence supports about any existing artifact's suitability for this purpose.

## Decision

**A protocol dispatch may occur only if a same-process authorization-to-execution binding record — created by the operation-producer role before gateway submission and verified by the protocol-executor role immediately before dispatch — establishes, by content-based digest comparison rather than by process-memory identity, that the operation about to be dispatched is unchanged from the operation whose normalized semantics received the authoritative permitting disposition.** No existing artifact in the chain (adapter evidence reference, kernel trace ID, gateway correlation ID, `AuditEvidence`, `GatewayAuditEvent`, or the gateway's HTTP response) is, by itself, sufficient to serve as this binding — each is analyzed below for what it proves and what it does not. This ADR establishes the binding record as a new, same-process-local architectural concept. It does not publish a schema for it, does not select a serialization or hashing technology beyond reusing this ecosystem's existing canonicalization discipline, and does not design a binding mechanism for a future separated-executor topology.

---

## Binding Model

### What is bound

Two representations participate in the binding digest:

1. **The preserved original protocol-shaped operation** — the `ProtocolOperation` that entered the chain at normalization, retained unchanged through authorization, per ADR-0011 Decision 7's normative requirement. A canonical dispatch-intent representation derived from it before authorization submission may substitute for the original operation only where a future specification explicitly proves the two equivalent; this ADR does not make that proof and defaults to binding the preserved original operation itself.
2. **The normalized authorization request actually submitted for evaluation** — the action, resource type, resource identifier, and populated context fields the operation-producer role composed into its gateway submission (§4 of `operation-producer-and-execution-boundary.md`'s field-ownership table, unchanged by this ADR).

Binding both, rather than the original operation alone, is deliberate: the discovery assessment's compromise analysis (§12) identifies a threat specific to execution's existence — a compromised or buggy adapter-invocation path could normalize operation X for authorization while a substituted operation Y is later dispatched under that authorization. Binding only the dispatched operation to itself would not catch this; binding it to the specific normalized request that was actually evaluated does.

**A third fact participates by reference, not by digest:** the authoritative gateway disposition permitting execution. The disposition is not known at binding-creation time — it is the result of the round trip this record is created before — so it cannot be part of the digest. It is instead correlated to the binding record using the identifiers the chain already generates (below), and its being a permitting disposition (`ALLOW`, per ADR-0011 Decision 9) is checked independently at verification time.

**What this binding does not attempt to close.** The gap between the normalized request the operation-producer role submits and the canonical action/resource form `basis-gateway`'s own composition logic derives from it (`operation_aware_composition.py`) is internal to the gateway's existing, accepted authorization-enforcement boundary and is not reopened here — `basis-gateway` remains the authorization enforcement boundary, not the executor, and this ADR does not add a verification responsibility to it. Gate 1's scope, per the problem statement, is the boundary between authorization and dispatch — not a re-verification of gateway-internal composition correctness, which this ADR treats as already governed.

### The binding record

A **binding record** is created by the operation-producer role, once, at or before gateway submission, for exactly one operation. It carries, as an architectural concept rather than a published schema:

- an opaque **binding identifier**, minted by the operation-producer role (deployment-local uniqueness, following the same convention ADR-0007 already established for `reference_id`);
- a **binding digest** — a deterministic digest computed over the two representations named above, canonicalized under this ecosystem's existing canonicalization discipline (the same RFC 8785 / digest-over-canonical-bytes pattern ADR-0007 already adopted for adapter evidence, reused here rather than a new mechanism invented for this purpose — the exact field composition and profile identity are implementation-planning detail this ADR does not fix, consistent with how ADR-0011 itself deferred "the exact digest input");
- a **consumption state** (not-yet-dispatched / dispatched), used only for the single-use property described under Failure Semantics;
- references to the existing correlation identifiers already generated by the chain (the producer's own request/correlation identifiers, and, once the round trip completes, the gateway-generated correlation ID and the kernel's `trace_id`/`evidence_id`) — sufficient to tie the record to the specific disposition received for it.

The binding identifier and the binding record itself — `binding_id`, `binding_digest`, and `consumption_state` — are **same-process-local**: no new field is needed on the wire to carry any of them, they are not submitted to `basis-gateway`, they do not appear on the operation-aware request or response, and they require no `basis-schemas` change. This is a property of the binding record's own fields, not a claim that every identifier the binding record must correlate against already reaches the operation-producer role today — see the clarification immediately below for that distinct question.

Because the first bounded topology is same-process (ADR-0011 Decision 3), the operation-producer role correlates the disposition it receives back to its own binding record using the identifiers the round trip already carries — for the gateway-generated `correlation_id` and the kernel `trace_id`, this holds today (both are already returned on the operation-aware evaluation response). It does not yet hold for `evidence_id`.

**Clarification (reconciliation, post-acceptance):** `basis-gateway`'s currently released operation-aware evaluation response returns `correlation_id` and `trace_id` to the caller, but does not return `AuditEvidence.evidence_id`; that identifier is retained by `basis-gateway` today only to construct its own internal `GatewayAuditEvent`. This does not change this ADR's binding model — the binding digest never depends on `evidence_id`, and the binding record's correlation references are attached by reference, not by digest — and it does not require the binding record itself to be sent to `basis-gateway` or to acquire a portable, wire-facing form: the binding record remains exactly as same-process-local as this ADR specifies. What it means is narrower: a binding record cannot actually carry a correlation reference to `evidence_id` until the operation-aware gateway response is extended to expose that already-governed identifier. That gateway-contract exposure is an implementation prerequisite tracked in `bounded-rest-execution-implementation-plan.md`, not a reopening of this ADR's binding-record decision or its Accepted status, and not a reason to remove `evidence_id`'s (`kernel_evidence_id`'s) required place in this architecture.

### Where the binding is created

The operation-producer role creates the binding record, because it is the role that already retains the preserved original operation and already composes the normalized submission (ADR-0011 Decision 1, 2; `operation-producer-and-execution-boundary.md` §2). Creation happens once both representations are known and before the gateway call, so that the digest reflects what was actually submitted for evaluation, not a reconstruction after the fact.

### Where the binding is verified

The protocol-executor role verifies the binding record immediately before dispatch — the same requirement ADR-0011 Decision 1 already names ("verifies whatever authorization-to-execution binding a future binding architecture requires... before dispatch"). Verification is deliberately colocated with dispatch itself, consistent with the discovery assessment's finding (§5) that "verifying the binding immediately before dispatch is plausibly inseparable from dispatch itself — the check is only meaningful as the last gate before the side effect." Verification recomputes the binding digest over the operation instance about to be dispatched and the normalized request the process holds for it, and compares the result to the stored binding digest. Any mismatch fails closed (below).

### Why existing artifacts, alone, are not sufficient

Each is examined for what it proves and what it does not, per the discipline this ADR is required to apply:

- **Object identity in process memory / Python object reference equality.** Identity proves that the process holds a reference to the same object, not that the object's contents are unchanged — a mutable object can be altered in place after evaluation without its identity changing. This is precisely the mutation half of the canonical threat, and an identity check does not catch it. The binding digest is a content check, not an identity check, for exactly this reason, and is described as a digest over defined representations rather than as a property of process memory so the same concept generalizes to a future topology where identity is meaningless across a process boundary.
- **Request ID / correlation ID / kernel trace ID / kernel evidence ID.** Per the discovery assessment's own finding (§6, §13), none of these is integrity-checked at a trust boundary today — they are generated, carried, and compared for correlation, not verified against tampering. They remain diagnostic and are reused here only for correlating a binding record to the disposition received for it, never as the mutation/substitution check itself.
- **`AdapterEvidenceReference` / adapter evidence digest.** This digest is computed over normalized fields plus a governed, deliberately narrow per-protocol projection of protocol evidence (ADR-0007) — one mapping step removed from the original `ProtocolOperation`, not a digest of it, and not cryptographically bound to the gateway-composed canonical action/resource the kernel actually evaluated (discovery assessment §6). It was designed for authorization-side evidence hygiene, not for this purpose; widening its projection to serve binding would be exactly the scope creep ADR-0007's own governed-projection design was built to avoid. It may be referenced as one of the correlation identifiers tying a binding record to the operation's authorization-side evidence trail; it does not substitute for the binding digest.
- **`AuditEvidence` / `GatewayAuditEvent`.** These are the authoritative record of what was decided and enforced, and are exactly what the binding record correlates to via `evidence_id`. Neither carries a digest of the dispatch-intent operation, and neither was designed to detect a substitution that happens after they were produced.
- **The gateway's HTTP response, alone.** It carries the disposition, the outcome, and the failure-reason vocabulary — everything needed to decide *whether* to dispatch. It carries nothing that would let an executor detect that the operation about to be dispatched has been silently substituted between receipt of that response and dispatch, because nothing in the response is a digest of the dispatch-intent operation.
- **A new digest, unjustified.** This ADR does introduce one — the binding digest — but not as an invented cryptographic primitive: it reuses the same "digest over canonical bytes" pattern ADR-0007 already established, applied to different material chosen because the existing digests do not cover it (above).
- **A signed execution grant, JWT, nonce, TTL, or one-time-use token.** None of these is required for the same-process first-reference topology. Per the discovery assessment's finding (§6): cryptographic protection is necessary only when the channel carrying the bound value is not already trusted by some other means; in the same-process topology, the channel is process memory, already trusted at the level ADR-0011 already accepted when it selected this topology. A future separated topology would need to revisit this — see below — and this ADR does not design that revisit.

---

## Same-Process First Reference Topology

### Trust assumptions accepted here

This ADR accepts, for the same-process topology only: that the process's own retained data, from binding-record creation to verification, is not altered by anything other than the process's own code — an in-process memory-integrity assumption, not a network-authenticity one; and, per ADR-0011 Decision 19, that a fully compromised process defeats this binding along with everything else the process does, which is a residual risk this ADR does not attempt to close in software. Same-process trust does not eliminate the need for the binding digest itself — ADR-0011 Decision 15 already rejects the inference that shared memory alone removes the binding requirement, because bugs, confused composition, retries, and stale state can still cause `authorized A → executed B` within a single process even absent a compromise.

### Freshness and single use — narrowing Gate 1's replay/freshness requirement for this topology

ADR-0011's Follow-On Decision Gates section lists "replay/freshness semantics" as part of Gate 1. This ADR does not resolve that requirement in full. It resolves only the portion the same-process first-reference topology needs: a **process-local single-consumption guarantee** — a binding record may be consumed by at most one dispatch attempt within the process's own running lifetime. This prevents an accidental duplicate dispatch of the same authorized operation (a retry bug, a re-entrant call) from within the same process instance, using the discipline this ecosystem already has a working precedent for — `basis-producer`'s own retain-before-mint ordering (Phase 2A/2B) — extended to "consume-once" rather than invented fresh.

This guarantee is deliberately minimal, and this ADR states directly what it does not provide: it is not wall-clock expiry, requires no external time source (consistent with ADR-0008's own air-gapped-deployment preference), does not survive a process restart, and provides no protection against a deliberately compromised process replaying its own state. Whether a disposition should expire, whether a policy reload between authorization and dispatch invalidates a pending execution, restart-surviving replay prevention, retry/idempotency behavior, and any replay-database mechanism are all broader replay/freshness properties this ADR does not define and does not authorize treating as already answered.

**This ADR narrows, but does not close, Gate 1's replay/freshness requirement — and only for the same-process topology.** It does not reclassify those broader properties as Gate 2 concerns merely by naming Gate 2 near them: whether they are eventually resolved by a Gate 2 execution-lifecycle specification, a dedicated Gate 1 addendum, or another future architecture decision is not decided here. Implementation must not proceed as though they were already answered. This ADR does not define a TTL value, a retry count, or an idempotency key.

---

## Future Separated Topology Considerations

The binding digest is defined as content over specified representations, not as a property of process memory, so a future separated-executor topology can extend the same concept — the same two representations, digested the same way — into a portable artifact without redefining what is bound. What changes in a separated topology is not the binding's content but how its integrity is protected in transit: ADR-0011 Decision 5 already states that a separated executor introduces a new authenticated trust boundary requiring its own architecture — producer/executor or gateway/executor authentication, a portable authorization-to-execution binding, replay protection, freshness, handoff integrity, failure semantics, and correlation. This ADR does not design any of that. It states only that the binding-content decision made here is not itself an obstacle to that future work, because the digest concept does not presuppose same-process trust — only this ADR's freshness answer and its "no additional cryptographic protection" conclusion are specific to the same-process topology and would need to be revisited.

---

## Failure Semantics

A protocol dispatch must not occur, and the operation must be treated as not executed, if any of the following hold, in addition to the dispatch-prevention conditions ADR-0011 Decision 9 already establishes (`DENY`, failed evaluation, failed authentication, failed admission, failed composition, malformed or contradictory gateway response):

- no binding record exists for the operation the executor is about to dispatch;
- the binding digest recomputed at verification time does not match the binding record's stored digest;
- the disposition correlated to the binding record is not a permitting disposition;
- the binding record has already been consumed by a prior dispatch attempt;
- the executor cannot correlate the disposition it received to any binding record at all;
- verification itself raises an error or cannot complete.

Every one of these is a fail-closed condition: the executor does not dispatch, does not retry around the failure, does not downgrade to a "safer" operation, and does not treat the failure as informative about whether authorization should have been granted — restating, for the binding check specifically, ADR-0011 Decision 10's existing prohibition on reinterpreting authorization.

---

## Security Consequences

### Prevented by this architecture

- Dispatch of an operation that differs, in the representations the binding digest covers, from the operation whose normalized semantics were authorized — the canonical `authorize A → execute B` threat.
- Dispatch based on process-memory identity alone, which would not detect in-place mutation of a retained mutable object.
- A duplicate dispatch of the same authorized operation from within one process's own running lifetime.
- Treating an adapter evidence digest, a correlation identifier, or the gateway's HTTP response, alone, as proof that the dispatched operation is unchanged.

### Still requiring follow-on architecture

- Full replay/freshness semantics beyond the process-local single-consumption guarantee this ADR narrows Gate 1 to — TTL/expiry, policy-reload invalidation, restart-surviving replay prevention, retry/idempotency behavior, and any replay database — remain open; this ADR does not decide whether they are resolved by a Gate 2 execution-lifecycle specification, a dedicated Gate 1 addendum, or another future architecture decision, and does not authorize proceeding as though they were already answered.
- Execution-lifecycle vocabulary (Gate 2).
- Minimum execution-evidence semantics, including whether the binding record or its identifier participates in a future execution-evidence record (Gate 3).
- Protocol/device credential custody, if the eventually selected bounded target requires one (Gate 4) — this ADR's binding model does not require a protocol/device credential to participate in the binding digest or the correlation chain, and nothing in this decision creates a Gate 4 dependency where none otherwise exists.
- A portable, authenticated binding artifact and its accompanying trust-boundary architecture, if a separated executor topology is ever selected (deferred to future architecture per ADR-0011 Decision 5).

### Residual risks

- A fully compromised operation-producer/executor process can fabricate a binding record that matches its own subsequently substituted operation, because the same process both creates and verifies the record — the same residual-risk structure ADR-0007 already accepts for adapter evidence ("hashing does not make a compromised producer trustworthy") and ADR-0011 Decision 19 already accepts for the same-process topology generally.
- Deployment- and network-level controls (process integrity, memory protection, least-privilege execution) remain necessary and are not designed by this ADR.

---

## Alternatives Considered

**A — Rely on process-memory identity alone (no digest).** Rejected: identity does not detect in-place mutation of a retained mutable object, which is the mutation half of the canonical threat this ADR must close; see the analysis above.

**B — Reuse the adapter evidence digest as the binding mechanism.** Rejected as insufficient alone: the adapter evidence digest covers normalized-plus-governed-projection fields, not the preserved original operation, and is not bound to the specific normalized request the kernel evaluated; widening its projection for this purpose would repurpose a deliberately narrow, evidence-hygiene-motivated design for a security property it was not built to carry.

**C — Treat correlation identifiers (`trace_id`, `evidence_id`, correlation ID) as sufficient, without a content digest.** Rejected: every one of these is diagnostic today, generated and compared for linkage but never checked at a trust boundary; promoting one to that role without a content-based check would prove that two records claim to describe the same operation, not that the dispatched operation's content is unchanged.

**D — Require a signed execution grant or token now, ahead of a separated topology.** Rejected for the first bounded reference slice: the discovery assessment's own finding is that cryptographic protection is needed only when the channel is not already trusted by some other means, and the same-process channel is process memory, already trusted at the level ADR-0011 accepted. Introducing signing now would be exactly the "invent crypto for elegance" ADR-0007 and ADR-0008 already declined to do where the evidence did not require it.

**E — Defer Gate 1 entirely until a separated topology forces the question.** Rejected: ADR-0011 Decision 15 already rejects the inference that same-process topology removes the binding requirement; deferring further would leave the bounded reference implementation with no way to make the canonical threat fail closed, contrary to ADR-0011's own unconditional Gate 1 prerequisite.

---

## Consequences

### Positive

- Upon acceptance, this ADR resolves the same-process authorization-to-execution binding mechanism and narrows Gate 1's replay/freshness requirement to the process-local single-consumption guarantee that topology needs. It does not resolve broader replay/freshness properties, and it does not by itself unblock implementation planning, which remains gated on Gates 2 and 3, and Gate 4 if applicable, and on whatever future decision resolves those broader replay/freshness properties.
- The canonical mutation/substitution threat has a concrete, evidence-grounded, implementation-neutral answer.
- No existing component's contract, schema, or behavior changes to realize the binding record itself: `binding_id`, `binding_digest`, and `consumption_state` are same-process-local and require no `basis-gateway`, `basis-core`, `basis-adapters`, or `basis-schemas` modification. (This is distinct from populating the binding record's `evidence_id` correlation reference, which additionally depends on a `basis-gateway` response-contract prerequisite not yet satisfied — see "The binding record," above.)
- The binding concept is defined to survive a future separated topology without redesign of what is bound.

### Negative / Tradeoff

- The operation-producer and protocol-executor roles each gain a concrete new responsibility (creation; verification) beyond what ADR-0011 named in the abstract.
- A same-process-compromised producer/executor can still defeat the binding, a residual risk this ADR states rather than closes.
- The exact binding-material serialization and canonicalization profile remain implementation-planning detail, not fixed here, which later work must still resolve before a reference implementation can be built.

---

## What This Leaves Open

This ADR does not decide: the exact binding-material field list, serialization, or a named/versioned profile identifier for the binding digest; the specific digest algorithm beyond reusing the ecosystem's existing `sha-256`-recommended, open-algorithm-label convention; whether the binding record or its identifier participates in a future execution-evidence record (Gate 3); the execution-lifecycle vocabulary (Gate 2); replay/freshness semantics beyond process-local single use; a portable or signed binding artifact for a future separated topology; device credential custody (Gate 4, not triggered by this decision); an implementation plan; or any Python interface, class, module, or package shape for either role, consistent with ADR-0011's own restraint on implementation specifics.

---

## Relationship to ADR-0011

This ADR resolves the authorization-to-execution binding mechanism Gate 1 requires, and narrows Gate 1's replay/freshness requirement to the process-local single-consumption guarantee the same-process first-reference topology needs; it does not resolve broader replay/freshness properties ADR-0011 also listed under Gate 1, which remain open pending later architecture. It does not reopen ADR-0011's role establishment, its same-process default-topology selection, or its governing invariants — it operates entirely within the space ADR-0011 left open. It answers ADR-0011 Decision 1's third bullet ("verifies whatever authorization-to-execution binding a future binding architecture requires (Gate 1) before dispatch") directly, and it does not weaken ADR-0011 Decision 15's rejection of "same-process removes the binding requirement." Acceptance of this ADR, together with Gates 2 and 3 (and Gate 4 if applicable) and whatever future decision resolves the broader replay/freshness properties this ADR leaves open, is required before bounded execution implementation may dispatch, per ADR-0011's own Validation / Implementation Gate — this ADR does not itself authorize implementation.

## Relationship to ADR-0010

This ADR does not modify `basis-producer`'s established role as the permanent operation-producer-runtime repository, nor does it expand that establishment into an authorization to implement execution. Binding creation is assigned to the operation-producer *role*, which `basis-producer` hosts per ADR-0010; this ADR does not require, and does not authorize, any change to `basis-producer`'s merged, bounded authorization-only implementation.

## Relationship to ADR-0007 / ADR-0008 / ADR-0009

This ADR reuses, without altering, ADR-0007's canonicalization and digest-over-canonical-bytes discipline, applying the same pattern to different material rather than inventing a new one. It does not modify `AdapterEvidenceReference`, its digest, or its construction ownership, and it does not widen the governed protocol-evidence projection ADR-0007 deliberately narrowed. It does not modify producer workload authentication, admission, or the trusted-ingress topology ADR-0008 and ADR-0009 established; the binding record described here is unrelated to, and does not participate in, mTLS producer identity or bearer authorization-subject identity.

---

## Non-Goals

This ADR does not: implement protocol execution; modify `basis-producer`, `basis-gateway`, `basis-adapters`, `basis-core`, `basis-schemas`, or `basis-identity`; create execution schemas or a `basis-schemas` contract for the binding record; create `basis-executor` or any other new repository; create a daemon, service, queue, message bus, or API; define execution lifecycle states beyond the fail-closed binding-failure conditions stated above; define execution evidence schema; define retry behavior; define TTL or freshness values beyond the process-local single-consumption property stated above; define idempotency semantics; define protocol/device credential custody; design a separated-executor binding artifact; select a specific digest algorithm or canonicalization profile identifier beyond reusing the ecosystem's existing convention; create an implementation plan; or create a "`basis-producer` Phase 6."

---

## Validation / Implementation Gate

This ADR's merging did not itself constitute acceptance, consistent with this repository's established convention (see ADR-0011's own Validation / Implementation Gate section and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself change its status to `Accepted`. After this ADR merged, a separate formal architect acceptance review found the binding model internally consistent, correctly scoped to the same-process first-reference topology, and preserving the architecture-first boundary before implementation; this ADR's status now records `Accepted`.

Acceptance of this ADR resolves the authorization-to-execution binding mechanism for the same-process first-reference topology and narrows Gate 1's replay/freshness requirement to the process-local single-consumption guarantee that topology needs. It does not resolve broader replay/freshness properties — TTL/expiry, policy-reload invalidation, restart-surviving replay prevention, retry/idempotency behavior, or a replay database — which remain open and must be addressed by a later architecture decision before any implementation that depends on them may proceed. Acceptance does not, by itself, authorize bounded execution implementation: Gates 2 and 3 remain outstanding regardless, and Gate 4 remains conditionally outstanding if the eventually selected bounded target requires a protocol/device credential, per ADR-0011's own Validation / Implementation Gate. No implementation is authorized by this ADR's acceptance, and none was authorized by its proposal.

## References

- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — establishes the protocol-executor role, the same-process first-reference topology, and Gate 1 as an unconditional prerequisite this ADR resolves for the same-process binding mechanism and narrows for same-process replay/freshness (process-local single consumption); broader replay/freshness remains open
- [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) §5 (logical-role decomposition), §6 (authorization-to-execution binding analysis, the primary evidentiary source for this ADR's Binding Model), §7 (replay and freshness), §12 (compromise and bypass analysis), §13 (correlation model)
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §2 (role definitions this ADR's binding-creation/verification assignment inherits), §5 (lifecycle invariants), §6 (correlation model)
- [ADR-0007](0007-adapter-evidence-construction.md) — the canonicalization/digest pattern this ADR reuses for different material; adapter evidence remains unmodified and insufficient alone for this purpose, per the analysis above
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) / [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) — producer workload authentication and trusted-ingress topology, unrelated to and unmodified by this ADR's same-process binding record
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) — establishes `basis-producer` as the operation-producer-runtime repository hosting the binding-creation responsibility this ADR assigns to that role
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the "implementation proves a stable shape" publication discipline this ADR's deferral of a binding-record schema follows
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component-boundary and dependency-direction model this ADR does not alter
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR proposal and acceptance process this ADR follows
