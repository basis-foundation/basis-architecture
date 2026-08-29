# ADR-0013: Execution Lifecycle Semantics

## Status

Accepted

## Context

[ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — Accepted — established a distinct logical protocol-executor role, architecturally separate from the operation-producer role, and selected same-process colocation with the existing `basis-producer` reference implementation as the default topology for the first bounded execution reference slice. It fixed the invariants execution must preserve regardless of topology, and it named three unconditional prerequisites, plus a fourth conditional one, before any bounded execution implementation may dispatch a protocol operation: Gate 1 (authorization-to-execution binding), Gate 2 (execution-lifecycle semantics), Gate 3 (minimum execution-evidence semantics), and Gate 4 (protocol/device credential custody, only if the eventually selected bounded target requires one). ADR-0011's own Decision 12 already states a truthfulness principle Gate 2 must satisfy — "execution reporting must preserve uncertainty rather than infer non-occurrence from missing confirmation" — and lists, without making normative, the distinctions a future execution-lifecycle specification must preserve: not attempted; attempted; remote acknowledgement observed; remote rejection observed; outcome unknown; resulting state verified; resulting state not verified; and evidence persistence success/failure. ADR-0011's own Gate 2 description is narrower and more direct: "A normative architecture specification must define truthful execution-state semantics sufficient for the first bounded protocol. It must preserve uncertainty (Decision 12) and avoid assuming all protocols have HTTP-style acknowledgement behavior. This ADR does not define the final enum."

[ADR-0012](0012-authorization-to-execution-binding.md) — Accepted — resolved Gate 1 for the same-process first-reference topology: a same-process authorization-to-execution binding record, created by the operation-producer role before gateway submission and verified by the protocol-executor role immediately before dispatch, establishes by content-based digest comparison that the operation about to be dispatched is unchanged from the operation whose normalized semantics received the authoritative permitting disposition. ADR-0012 also narrowed Gate 1's replay/freshness requirement to a **process-local single-consumption guarantee** — a binding record may be consumed by at most one dispatch attempt within the process's own running lifetime — while stating explicitly, and repeatedly, that broader replay/freshness properties (wall-clock expiry, policy-reload invalidation, restart-surviving replay prevention, retry/idempotency behavior, a replay database) remain open, and that whether those broader properties "are eventually resolved by a Gate 2 execution-lifecycle specification, a dedicated Gate 1 addendum, or another future architecture decision is not decided here." This ADR is the first point at which that open question must be answered one way or the other; see **Replay and Freshness — Bounded Resolution**, below.

The merged, non-normative [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) is the evidentiary basis for this ADR, exactly as it was for ADR-0011 and ADR-0012. Its §8 ("Execution Lifecycle and Failure Semantics") is the primary source: it evaluates the interoperability roadmap's five-state sketch (`authorized-but-not-attempted`, `attempted-and-completed`, `attempted-and-failed`, `partially-applied`, `execution-status-unavailable`) — already documented elsewhere as "roadmap language, not a governed contract" — against the nine adapters' own accepted protocol-semantics evidence in §9, and finds the five-state sketch insufficient in at least two ways: it conflates a local pre-dispatch rejection with a remote protocol-level rejection, and it conflates "no acknowledgement primitive exists for this protocol" with "an acknowledgement primitive exists but did not arrive in time." §8 states two governing distinctions as its load-bearing finding: `failed` and `not executed` are not equivalent, and `timeout` is evidence of uncertainty, not evidence of non-occurrence. §9's protocol-by-protocol table is direct, repository-anchored evidence that no single request/response, HTTP-shaped lifecycle can honestly describe DNP3 and IEC 61850's stateless select/operate model, MQTT's fire-and-forget publish semantics, or KNX's bus-level telegrams without either lying about confirmation the protocol cannot provide, or discarding a real distinction (protocol-transaction progress versus physical device-state confirmation) the architecture has never previously permitted itself to discard.

[`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §5 and §8 are restated here, not re-derived: §5's authorization-to-execution lifecycle invariants (no dispatch before authoritative disposition; authorization evidence and execution evidence are separate artifacts; execution failure does not retroactively invalidate a correct authorization decision; execution success cannot retroactively legitimize an unauthorized dispatch) and §8's five preserved distinctions among failure and degraded conditions (`not executed ≠ execution failed`; `execution failed ≠ execution status unknown`; `execution status unknown ≠ authorization denied`; `authorization denied ≠ authorization evaluation failed`; and `not executed` as a family of causes, not one condition) are the foundation this ADR's lifecycle model is built on top of, not a foundation it revisits.

Consistent with this repository's ADR-acceptance governance convention (ADR-0007 through ADR-0012 were each first merged with `Status: Proposed`, and a separate, dedicated follow-up PR later changed the status to `Accepted` after independent architecture and governance review — see [`docs/adr/README.md`](README.md#lifecycle-states)), this ADR was submitted as `Proposed` and has since undergone that same separate, dedicated formal-acceptance review, together with ADR-0014, ADR-0015, and ADR-0016 as one coupled execution-readiness milestone; its status now records `Accepted`. Acceptance establishes the execution-lifecycle vocabulary defined here as governed semantics a future bounded execution implementation must satisfy — it does not itself authorize that implementation; see **Validation / Implementation Gate**, below.

The current implemented state remains exactly what ADR-0011 and ADR-0012 both describe: `basis-producer`'s completed bounded authorization slice (Phase 2A through Phase 5, merged) stops at the authoritative disposition; `basis_producer.rest_composition`'s own module docstring states, as a scope boundary, that the module will not "execute any protocol operation under any disposition, including `ALLOW`." Nothing in this ADR changes that. No protocol operation is executed anywhere in the ecosystem today, and this ADR's acceptance does not change that either.

## Problem

Once a protocol dispatch has been authorized (`ALLOW`, per ADR-0011 Decision 9) and bound to the exact authorized operation (ADR-0012's binding record, verified immediately before dispatch), what lifecycle states can an execution attempt truthfully occupy, in what order may it move between them, and how must partial protocol progression, missing confirmation, and evidence-persistence failure each be represented — without any state ever implying stronger truth than the protocol, or the executor's own observation of it, can actually support?

The answer must, at minimum: preserve every distinction ADR-0011 Decision 12 and `operation-producer-and-execution-boundary.md` §8 already require; avoid presupposing that every protocol offers HTTP-style synchronous acknowledgement (ADR-0011 Decision 13, the discovery assessment §9's own protocol table); avoid collapsing "the protocol has no confirmation primitive" into "confirmation didn't arrive" (discovery assessment §8); avoid inventing a single `partially-applied` state that conflates a protocol transaction stopping partway through (a structural, frequent outcome for DNP3 and IEC 61850's select/operate model) with a claim about physical device state the protocol evidence does not support (discovery assessment §9); and state plainly which logical role's observation is authoritative for a lifecycle-state assertion, without deciding who durably records it (Gate 3). The answer must also decide — because ADR-0012 explicitly left the question open rather than resolved or reassigned it — whether any part of Gate 1's outstanding broader replay/freshness requirement is inseparable from defining what an execution attempt is, and if so, exactly how much of it this ADR resolves.

## Decision

**ADR-0013 defines the minimum truthful execution-lifecycle semantics required before BASIS can plan a bounded protocol-execution implementation.** It fixes what counts as an execution attempt and when one begins; establishes a lifecycle model built from three independently tracked, orthogonal facts — attempt outcome, resulting-state verification, and evidence durability — rather than one flat enum; fixes the ordering among authorization, binding verification, dispatch, observation, and (future) evidence construction; scopes truth-owning observation of lifecycle state so that upstream, pre-boundary facts remain owned by whichever upstream role or component actually observed them, while execution-boundary lifecycle facts are owned by the protocol-executor role ADR-0011 established, once that role has actually entered the boundary (Decision 14, below); states what must fail closed; and resolves the narrow, lifecycle-representation portion of Gate 1's outstanding replay/freshness question while explicitly declining to resolve the rest of it. It does not define an execution-evidence schema, does not select field names, enum value names, or a serialization, does not create an implementation plan, and does not authorize protocol execution.

### 1. What Counts as an Execution Attempt

An **execution attempt** is the logical unit of one protocol-executor dispatch action directed at the preserved, bound protocol operation (ADR-0011 Decision 7; ADR-0012's binding record), together with whatever the protocol-executor role subsequently observes about that specific dispatch before it stops observing. An execution attempt is scoped to exactly one authorization-to-execution binding record (ADR-0012); it is never shared across two different bindings, and a binding record, once consumed, authorizes at most one execution attempt (restating, not extending, ADR-0012's single-consumption guarantee). A multi-stage protocol operation (DNP3 and IEC 61850's select/operate model) is one execution attempt composed of an ordered sequence of stage-attempts — see **Multi-Stage Protocol Operations**, below — not several independent execution attempts.

### 2. Pre-Dispatch Record Versus Execution Attempt — Dispatch Is the Boundary

**Normative requirement:** the semantic content of an execution attempt — the fact that a protocol dispatch happened — cannot exist before dispatch, because nothing has yet occurred for it to describe. This ADR rejects the inference that authorization (`ALLOW`) or binding verification alone constitutes an execution attempt; an `ALLOW` disposition that is never dispatched is not an attempt of any kind, it is simply an unexecuted authorization, exactly as `operation-producer-and-execution-boundary.md` §8 already classifies it (`not executed`).

**A separate, narrower requirement governs what exists before dispatch, and fixes when it comes into existence relative to binding verification:** once an authoritative permitting disposition (`ALLOW`) reaches the protocol-execution boundary, the protocol-executor role enters that boundary and must create a local **pre-dispatch record** (equivalently, a pre-dispatch attempt context) *before* performing ADR-0012 binding verification — not after it, and not conditioned on verification succeeding. Binding verification is then carried out using that already-created record, as the first thing the protocol executor does once inside the pre-dispatch boundary, still strictly before the irreversible dispatch action. The pre-dispatch record is not itself an execution attempt, and does not carry an attempt-outcome value from the axis defined in Decision 4 while dispatch remains pending — it is the precondition an execution attempt is created from, at the instant dispatch is actually committed to, and it is also the object a binding-verification failure truthfully resolves (see the three cases below). Creating the record before verification, rather than after it succeeds, is what makes it possible for a binding-verification failure to be represented as the resolution of a record that already existed when the failure occurred, rather than as a fact invented retroactively for a boundary the executor never actually entered. This ordering also preserves the discovery assessment §14's crash-consistency analysis and its §8 finding that "process crash before dispatch" is only classifiable as `not attempted` "if durably recorded before the crash." This ADR requires the *ordering* (pre-dispatch record; then binding verification; then, only if verification succeeds, dispatch); it does not require the record to be durable across a process restart — whether and how that record must survive a crash, and whether it becomes part of a future execution-evidence artifact, is a Gate 3 question this ADR forwards rather than answers. A protocol executor that cannot create this record for whatever reason must not attempt binding verification or dispatch anyway (see **Fail-Closed Requirements**, below).

**Three cases produce `NOT_ATTEMPTED`, distinguished by whether the protocol-execution boundary was ever entered and, if it was, whether the pre-dispatch record itself was successfully created:**

- **Case A — Pre-boundary failure.** `DENY`, authorization-evaluation failure, producer authentication or admission failure, request-composition failure, a malformed or contradictory gateway response, or gateway unavailability (ADR-0011 Decision 9's existing list) prevent the operation from ever reaching the protocol-execution boundary at all. The protocol executor is never invoked, no pre-dispatch record is ever created, and `NOT_ATTEMPTED` for these conditions is a fact about the authorization chain's own outcome (`operation-producer-and-execution-boundary.md` §8's `not executed`), not a pre-dispatch record's resolution.
- **Case B — Boundary entered, but creation of the required pre-dispatch record itself fails.** The protocol executor has entered the pre-dispatch boundary, but the record-creation step required by this Decision does not succeed. Because the record never came into existence, nothing exists for anything to "resolve" — this ADR does not describe a nonexistent record as resolving to `NOT_ATTEMPTED`, because a record that was never created cannot occupy or leave any value. Instead, record-creation failure is itself a fail-closed condition (Decision 16): the operation is `NOT_ATTEMPTED`, fail-closed, and no pre-dispatch record exists — for Gate 3 or anything else — to correlate against.
- **Case C — Pre-dispatch record successfully created, then a boundary-local condition prevents dispatch.** Only once a pre-dispatch record already exists can a boundary-local condition cause *it* to resolve to `NOT_ATTEMPTED`. This is the case ADR-0012 binding-verification failures fall under (no binding record found; the recomputed binding digest not matching the stored one; a non-permitting correlated disposition; an already-consumed binding; binding-verification error), and it also covers another protocol-executor-local pre-dispatch failure that arises after record creation (for example, an endpoint unreachable, or a transport-level connection timeout, before any bytes are sent). Here, and only here, is it accurate to say the pre-dispatch record itself resolves to `NOT_ATTEMPTED`, because a record actually existed at the moment the condition arose.

All three cases resolve to the identical terminal attempt-outcome value (Decision 4); the attempt-outcome vocabulary does not need, and this ADR does not define, separate enum values to distinguish them. That does not mean nothing downstream depends on which case produced a given `NOT_ATTEMPTED`: Gate 3's evidence-construction question — specifically, whether a pre-dispatch record exists for an execution-evidence candidate to correlate against at all — depends directly on it (see ADR-0014's **Execution Evidence Definition**). Case A produces no pre-dispatch record and no protocol-executor-observed fact for Gate 3 to record. Case B produces no pre-dispatch record either, because record creation is exactly what failed, and Gate 3 must not invent one to correlate evidence against. Case C is the only one of the three in which a pre-dispatch record actually exists, and correspondingly the only case in which an execution-evidence candidate may correlate to one. The three cases also differ in where truthful responsibility for the record sits: Case A was never the protocol executor's own record to keep, because the protocol executor never ran; Case B is the protocol executor's own fact, but about a record-creation attempt that failed, not a record it holds; Case C is the protocol executor's own pre-dispatch record, truthfully resolved.

**The pre-dispatch record's resolution (where one exists) and an execution attempt's existence are mutually exclusive, not sequential:**

- **If dispatch never occurs** — because the operation never reached the protocol-execution boundary (Case A), because the pre-dispatch record could not be created (Case B), because binding verification failed or another boundary-local condition arose after the record was created (Case C), or otherwise (Decision 16) — the operation resolves to `NOT_ATTEMPTED`: a terminal fact about the operation, stated using the attempt-outcome vocabulary (Decision 4), even though no execution attempt ever came into existence to hold that value, and, for Cases A and B, even though no pre-dispatch record ever came into existence either. `NOT_ATTEMPTED` is therefore a value in the attempt-outcome vocabulary that describes the *absence* of an attempt, not a state an attempt occupies and later leaves, and this ADR does not require a record to exist merely so `NOT_ATTEMPTED` can be produced.
- **If binding verification succeeds and dispatch is committed to,** an execution attempt begins to exist at that instant, and its attempt-outcome value begins directly at `DISPATCHED` (Decision 4) — never at `NOT_ATTEMPTED` first. The pre-dispatch record does not "become" an execution attempt already sitting in `NOT_ATTEMPTED` and then transition out of it; the pre-dispatch record (Case C's precondition) is superseded by the execution attempt the moment dispatch is committed to, and `NOT_ATTEMPTED` never applies to an attempt that reached this branch.

This is a semantic distinction, not an implementation requirement: this ADR does not mandate two separate stored objects, two identifiers, or two data structures for the pre-dispatch record and the eventual execution attempt — a future implementation may represent them as one evolving in-process structure. What this ADR requires is that the *concept* of "dispatch was never attempted" — a terminal fact about the operation — never be modeled as a state an execution attempt transitions out of, and that no execution attempt be described as having passed through `NOT_ATTEMPTED` on its way to `DISPATCHED`.

### 3. Three Independent Axes, Not One Enum

**Normative decision:** an execution attempt's truthful description requires three independently tracked facts, not one combined status field. Collapsing them was exactly the discovery assessment's finding against the five-state sketch — "operation accepted but state change unverified... a distinction the five states do not currently carry a field for," and "evidence persistence failure after device action... is a *distinct* failure from any execution-status value." The three axes:

1. **Attempt outcome** — what happened to the dispatch, across the full `NOT_ATTEMPTED` / `DISPATCHED` / `PROTOCOL_ACCEPTED` / `PROTOCOL_REJECTED` / `OUTCOME_UNKNOWN` vocabulary (Decision 4). Not uniformly owned by one role: a pre-boundary `NOT_ATTEMPTED` (Decision 2's Case A) is owned by whichever upstream role or component actually observed the condition, because the protocol executor was never invoked and observed nothing. Every other attempt-outcome value — a boundary-local `NOT_ATTEMPTED` (Cases B and C), `DISPATCHED`, `PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, and `OUTCOME_UNKNOWN` — is owned and asserted by the protocol-executor role, because it is the only role with direct access to the protocol-execution boundary and channel (Decision 14, below).
2. **Resulting-state verification** — whether the physical or logical state the operation was meant to change was independently confirmed, distinct from whether the protocol accepted the dispatch. Owned and asserted by the protocol-executor role (Decision 14, below), for the same reason. Meaningful only once attempt outcome has reached a non-`NOT_ATTEMPTED` value; often unavailable for protocols or operations that offer no read-back.
3. **Evidence durability** — whether the record of the above two facts was itself durably persisted. This axis describes the record, not the event the record describes, and a failure on this axis never changes the value of either of the other two axes.

No axis may be inferred from another. A `NOT_VERIFIED` result on axis 2 does not imply anything negative about axis 1, and an `EVIDENCE_PERSISTENCE_FAILED` result on axis 3 does not imply the operation did not execute — it implies only that the durable record of what happened, whatever it was, does not yet reliably exist.

### 4. Attempt-Outcome Axis — Definitions

| Value | Meaning | Terminal for this attempt? |
| - | - | - |
| `NOT_ATTEMPTED` | No protocol dispatch occurred, and no execution attempt exists to describe. A family of causes, not one condition, spanning the three cases Decision 2 distinguishes: Case A, pre-boundary (denied, evaluation failed, producer authentication or admission failed, producer-only context rejected, gateway unavailable, malformed or contradictory gateway response — restating `operation-producer-and-execution-boundary.md` §8's existing non-exhaustive set), never reaches the protocol executor and never produces a pre-dispatch record, so this value is a fact about the authorization chain's outcome; Case B, the protocol executor enters the pre-dispatch boundary but fails to create the required pre-dispatch record itself, so no record exists to resolve — this value is the record-creation failure's own fail-closed outcome, not a record's resolution; Case C, a pre-dispatch record was successfully created and a boundary-local condition then prevents dispatch (no binding record found; digest mismatch; disposition not permitting; binding already consumed; verification error; or another protocol-executor-local pre-dispatch failure, such as an endpoint unreachable before any bytes are sent), and here the pre-dispatch record itself resolves to this value. All three are equally valid members of the same `NOT_ATTEMPTED` family and share one enum value, but they are not interchangeable for evidence purposes — see Decision 2's discussion of what each case leaves for Gate 3 to correlate against. | Yes — a terminal fact about the operation; no execution attempt ever came into existence (Decision 2). |
| `DISPATCHED` | The protocol-executor role has committed to and completed the irreversible act of sending the protocol-shaped operation toward its endpoint. The execution attempt begins to exist at this value (Decision 2); it is never preceded by `NOT_ATTEMPTED` for the same attempt. Transient: an attempt does not remain in this state once an outcome is known or the executor stops waiting for one. | No — a resolving value below is always eventually assigned, even if that value is `OUTCOME_UNKNOWN`. |
| `PROTOCOL_ACCEPTED` | A positive protocol-level acknowledgement, response, or status was observed. This states only that the protocol layer accepted or positively responded to the dispatch — see **Protocol Acknowledgement Is Not Completion**, below, for what this value deliberately does not claim. | Yes, for this stage/attempt. |
| `PROTOCOL_REJECTED` | A negative protocol-level acknowledgement, error response, or explicit rejection was observed (a Modbus exception response, a BACnet `Error-PDU`, a DNP3 or IEC 61850 negative internal indication). Distinct from `NOT_ATTEMPTED`: dispatch occurred, and the protocol endpoint responded negatively to it — this is not the same fact as a local pre-dispatch rejection, and this ADR requires that the two never be represented with the same value. | Yes, for this stage/attempt. |
| `OUTCOME_UNKNOWN` | Dispatch occurred and the protocol-executor role obtained no definitive protocol-level answer, for either of two structurally different reasons the executor must be able to distinguish where the protocol makes the distinction knowable: the protocol offers no acknowledgement primitive at all for this operation (a QoS-0 MQTT publish; a KNX group-communication telegram; DNP3's stateless SBO, which the adapter's own docstring states plainly has no persistent select-state to confirm against), or an acknowledgement primitive exists but none arrived within the executor's observation window (a timeout, a lost response, a transport failure mid-operation). This ADR requires the distinction be preserved conceptually wherever the protocol makes it knowable; it does not mandate a specific field name or force a distinction where the protocol genuinely gives the executor no way to tell the two apart. | Yes, for this stage/attempt — closing the observation window does not mean the operation is known to have not occurred; see **Failure and Uncertainty Semantics**, below. |

This ADR retires the interoperability roadmap's `attempted-and-completed`, `attempted-and-failed`, and `partially-applied` labels as governed vocabulary. `attempted-and-completed` is replaced by `PROTOCOL_ACCEPTED` specifically to stop implying physical completion (Decision 12, below). `attempted-and-failed` is replaced by the `PROTOCOL_REJECTED` / `NOT_ATTEMPTED`(local-rejection-family) split. `partially-applied` is replaced by stage-sequence composition under **Multi-Stage Protocol Operations**, below, rather than by a fifth flat value.

### 5. Resulting-State-Verification Axis — Definitions

| Value | Meaning |
| - | - |
| `VERIFICATION_NOT_APPLICABLE` | The operation, or the protocol, offers no independent mechanism to confirm resulting state (KNX group communication, per its adapter's own explicit "payload values are never coerced into operational meaning" scope; most fire-and-forget publishes). |
| `STATE_VERIFIED` | An independent observation (a separate read-back, a distinct read operation, an out-of-band confirmation) confirmed that the resulting state matches what the operation intended. |
| `STATE_NOT_VERIFIED` | A verification mechanism exists in principle for this protocol or operation, but was not obtained — either because it was not attempted, or because it was attempted and was inconclusive. |

This axis is meaningless while attempt outcome is `NOT_ATTEMPTED` and carries `VERIFICATION_NOT_APPLICABLE` by definition in that case. It remains independently tracked once attempt outcome reaches `PROTOCOL_ACCEPTED`, because — per the discovery assessment §9's KNX evidence, the single clearest statement in the protocol table of the authorization/execution gap this whole ADR exists to describe truthfully — "a write's format" being validated is never evidence that "the physical actuator actually moved."

### 6. Evidence-Durability Axis — Provisional, Owned by Gate 3

This ADR names the axis and states the invariant that it must never be conflated with the other two (Decision 3, above; restating `operation-producer-and-execution-boundary.md` §8's "execution evidence write failure" row). It does not define the axis's values, its ownership, or its persistence mechanism beyond that requirement — which logical role constructs and retains execution evidence, and what the durable record looks like, is Gate 3's question. See **Relationship to Gate 3**, below.

### 7. Multi-Stage Protocol Operations

**Normative requirement:** a multi-stage protocol operation (DNP3 and IEC 61850's select/operate control model, or any future protocol with an equivalent structure) is represented as an ordered sequence of stage-attempts, each independently carrying its own attempt-outcome value from the axis in Decision 4. The execution attempt's overall attempt-outcome value is derived from its stages, never assigned independently of them:

- If every defined stage reaches `PROTOCOL_ACCEPTED`, the overall attempt-outcome is `PROTOCOL_ACCEPTED`.
- If any stage reaches `PROTOCOL_REJECTED` or `OUTCOME_UNKNOWN`, the overall attempt-outcome equals that stage's value, and no later stage is dispatched (see **Fail-Closed Requirements**, below) — a multi-stage attempt is only ever as confident as its weakest stage, and this ADR never allows a stage sequence to "round up" past the point where certainty was lost.

This directly answers the discovery assessment §9's finding that a SELECT that succeeds followed by an OPERATE that fails or times out "is direct evidence only that the protocol exchange stopped there — it is not, by itself, evidence of what fraction of the physical operation the device actually performed." This ADR does not attempt to state what fraction of a physical operation occurred; it states only where, in the protocol transaction, an attempt stopped. Whether the device physically changed state at all during a partially progressed multi-stage attempt is governed entirely by the resulting-state-verification axis (Decision 5), independently, and is very often `VERIFICATION_NOT_APPLICABLE` or `STATE_NOT_VERIFIED` for exactly the protocols where multi-stage attempts are most common — this ADR does not claim otherwise.

### 8. State Transitions

```text
authoritative permitting disposition reaches the protocol-execution boundary
        │
        ▼
protocol executor enters pre-dispatch boundary
        │
        ▼
create pre-dispatch record
        │
        ├── record creation fails (Case B; fail-closed; Decision 16)
        │       ▼
        │   NOT_ATTEMPTED   (terminal; no execution attempt ever existed; no pre-dispatch record exists —
        │                    a fail-closed fact about the failed creation attempt, not a record's own resolution)
        │
        └── record created — not yet an execution attempt
                │
                ▼
            verify ADR-0012 authorization-to-execution binding, using that record
                │
                ├── verification fails, or another boundary-local condition arises (Case C; Decision 2, Decision 16)
                │       ▼
                │   NOT_ATTEMPTED   (terminal; no execution attempt ever existed — the pre-dispatch record's own resolution)
                │
                └── verification succeeds, dispatch committed
                        ▼
                    execution attempt begins: DISPATCHED
                        │
                        ├─▶ PROTOCOL_ACCEPTED   (positive protocol-level response)
                        ├─▶ PROTOCOL_REJECTED   (negative protocol-level response)
                        └─▶ OUTCOME_UNKNOWN     (no definitive response within the executor's observation window)
```

A Case A condition (Decision 2's pre-boundary case — `DENY`, evaluation failure, producer authentication/admission failure, gateway unavailability, and similar) never reaches this diagram at all: the protocol executor is never invoked, so no pre-dispatch record is ever created, and `NOT_ATTEMPTED` is reached without any of the steps above. Case B (record-creation failure, the diagram's first branch) and Case C (a boundary-local condition after a record already exists, the diagram's second branch) both reach this diagram, but only Case C's `NOT_ATTEMPTED` is a pre-dispatch record's own resolution — Case B's is not, because no record was ever created for it to resolve.

Governing rules:

- `NOT_ATTEMPTED` is terminal, and it is not a state an execution attempt occupies and later leaves — for Case C, it is the pre-dispatch record's own resolution when binding verification fails or another boundary-local condition arises (Decision 2), reached after the record was created but before any execution attempt exists; for Case A, it is a fact about the operation for which no pre-dispatch record was ever created at all, because the protocol executor was never invoked; for Case B, it is the fail-closed fact that the protocol executor entered the boundary but failed to create the required record, and it is not a record's resolution either, because no record came into existence. There is consequently no `NOT_ATTEMPTED → DISPATCHED` transition: `DISPATCHED` marks the moment an execution attempt begins to exist, not a change of state within an attempt that already existed as `NOT_ATTEMPTED`. A later dispatch of the same authorized operation is, by Decision 1 and Decision 9 below, a distinct attempt with its own binding record, not a continuation of a `NOT_ATTEMPTED` pre-dispatch record.
- `DISPATCHED` is transient, and is the value an execution attempt's attempt-outcome holds from the instant the attempt begins to exist. This ADR does not permit an attempt to remain indefinitely in `DISPATCHED` as a reported terminal value; an executor that stops waiting without a definitive answer must resolve the attempt to `OUTCOME_UNKNOWN`, never leave it unresolved and never silently resolve it to `PROTOCOL_ACCEPTED`.
- There is no transition from `DISPATCHED` back to `NOT_ATTEMPTED`. Dispatch is a one-way transition once committed to (Decision 2), and `NOT_ATTEMPTED` describes only the case where dispatch never occurred at all. A process that crashes after dispatch and never learns the outcome must resolve, on recovery if it can determine an attempt was made, to `OUTCOME_UNKNOWN` — never to `NOT_ATTEMPTED` — because bytes may have already reached the endpoint. Whether a recovering process even *can* determine an attempt was made depends on the durability of the pre-dispatch record (Decision 2), which this ADR forwards to Gate 3 rather than resolves.
- `PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, and `OUTCOME_UNKNOWN` are each terminal for the stage/attempt they describe. None of the three transitions into another without a new dispatch action, which constitutes either the next stage of the same multi-stage attempt (Decision 7) or a new attempt entirely (Decision 9).

### 9. Retry and Attempt Correlation — Bounded Replay Resolution

**Normative requirement:** any dispatch action following a prior attempt's terminal outcome — for the same underlying authorized operation — is a new, distinct execution attempt, correlated to the prior one, never represented as a mutation of the prior attempt's record. This is inseparable from Decision 1's definition of an execution attempt as scoped to exactly one binding record: because ADR-0012's binding record is single-consumption, no second dispatch can occur under the *same* binding once the first has consumed it — a retry, if one occurs at all, necessarily requires the operation to be re-authorized and re-bound (a new `ALLOW`, a new ADR-0012 binding record), not a re-use of the exhausted one.

**This ADR requires retry lineage as a semantic fact, not as a record-shape requirement.** What Gate 2 requires: a sequence of retries for the same logical operation must remain explicitly reconstructible, and a new attempt must be correlated by reference to the prior dispatched attempt it follows. What Gate 2 does not require: that a particular "execution attempt record" itself physically carry that reference, or any specific field, identifier, or data structure for carrying it — this ADR does not mandate that shape, and does not introduce a new execution-attempt identifier to hold it. Which concrete artifact carries the correlation, and in what shape, was a Gate 3 question this ADR forwarded rather than answered. Accepted ADR-0014 satisfies this requirement through the execution-evidence record it defines, by having that record reference the prior attempt's own execution-evidence record (ADR-0014's Correlation Model); this ADR does not require that particular mechanism, only the semantic fact that ADR-0014's mechanism is one way of satisfying it, and a future reconciliation remains free to satisfy it by another concrete means without reopening this Decision. The fact that a new attempt exists is itself evidence that a new authorization-to-execution binding was created and consumed for it, per ADR-0012.

**This requirement, and the single-consumption inference it rests on, apply only once a prior binding record has actually been consumed by dispatch.** ADR-0012's Binding Model defines a binding record's consumption state as distinguishing not-yet-dispatched from dispatched; consumption happens at dispatch, not at binding-record creation or verification. Where a pre-dispatch `NOT_ATTEMPTED` condition (Decision 2) occurs because dispatch was never committed to — including a binding-verification failure under ADR-0012's Failure Semantics — the binding record involved, if one existed at all, was never consumed, and ADR-0012's single-consumption guarantee alone does not by itself prove what, if anything, must happen next for that operation. This ADR does not decide, for that pre-dispatch case: who may request another attempt; whether an unconsumed authorization or an unconsumed binding record may be resumed or reused; a maximum retry count; retry triggers; automatic retry; idempotency keys; or duplicate-effect handling. All of that remains open, exactly as **Replay and Freshness — Bounded Resolution**, below, already leaves it open — this ADR resolves attempt correlation and fresh-binding necessity only for the case where a prior attempt actually reached dispatch and consumed its binding, not for a pre-dispatch `NOT_ATTEMPTED` condition in which nothing was ever consumed. Where this ADR uses "retry" without qualification elsewhere, it means a dispatch action following a prior attempt that reached dispatch (this Decision); it does not mean, and must not be read as deciding, whatever may or may not follow a pre-dispatch `NOT_ATTEMPTED` condition — that is authorization-replay and operational-retry territory this ADR does not enter.

### 10. Replay and Freshness — Bounded Resolution

ADR-0012 left open whether Gate 1's broader replay/freshness properties — wall-clock expiry, policy-reload invalidation of a pending disposition, restart-surviving replay prevention, retry/idempotency behavior, and any replay database — would be resolved by a Gate 2 execution-lifecycle specification, a dedicated Gate 1 addendum, or another future architecture decision, and stated plainly that implementation must not proceed as though they were already answered.

**This ADR resolves only the portion of that question inseparable from defining what an execution attempt is** (Decision 9, above): attempt multiplicity is represented as distinct, explicitly correlated attempts — correlated by reference to the prior dispatched attempt, per Decision 9's semantic requirement, without this ADR mandating any particular record shape for carrying that correlation (Gate 3's question) — and any attempt after the first requires its own fresh authorization-to-execution binding under ADR-0012's existing single-consumption rule. This is a lifecycle-representation consequence of ADR-0012, not a new replay-prevention mechanism, and this ADR does not present it as resolving Gate 1's broader requirement.

**This ADR explicitly does not resolve, and defers to a dedicated future architecture decision:** whether a pending or already-consumed disposition expires on a wall clock; whether a policy-bundle reload between authorization and dispatch invalidates a disposition that has not yet been dispatched; whether a replay record must survive a process restart; whether or how a caller may request a retry, or how many retries are permitted; and whether a replay database of any kind is required. This ADR does not authorize implementation to treat any of those questions as answered, and does not itself constitute the "dedicated future architecture decision" ADR-0012 anticipated for the properties it leaves open.

### 11. Authorization Success Differs From Execution Success

**Normative restatement, not a new rule:** `ALLOW` means `basis-core` and `basis-gateway` determined the operation was permitted to be dispatched; it is a fact about the authorization decision, complete and immutable the moment it is produced (`operation-aware-trace-audit-evidence.md`; `operation-producer-and-execution-boundary.md` §5). It is never proof that dispatch occurred, that the protocol accepted it, or that any resulting state changed. This ADR's attempt-outcome axis (Decision 4) is not one fact with a single precondition: its `DISPATCHED` value, and every value reachable from it, can only begin to exist after `ALLOW` and after Gate 1 binding verification succeed — the two facts are produced by different roles, at different times, and this ADR does not permit either to stand in for the other in a governed record. Its `NOT_ATTEMPTED` value carries no such precondition: it is exactly the value this axis takes when that chain breaks at any point — including before `ALLOW` is ever produced (Decision 2's Case A, a pre-boundary condition) or after `ALLOW` but before dispatch is committed to (Decision 2's Cases B and C, boundary conditions — the pre-dispatch record itself failing to be created, or, once created, a boundary-local condition such as binding-verification failure preventing dispatch) — per Decision 2's three cases.

### 12. Dispatch Differs From Completion

`DISPATCHED` (Decision 4) marks only that the protocol-executor role has sent the operation; it is not synonymous with, and must never be reported as equivalent to, a resolved outcome. "Completion," in this ADR's vocabulary, means an attempt or stage has reached a terminal attempt-outcome value (`PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, or `OUTCOME_UNKNOWN`) — not that the underlying real-world action is confirmed to have finished, which is a separate question the resulting-state-verification axis (Decision 5) governs independently.

### 13. Protocol Acknowledgement Is Not Completion

**Normative restatement of a finding already established, applied here as a binding constraint on the vocabulary:** `PROTOCOL_ACCEPTED` states only that the protocol layer accepted, positively responded to, or returned a success status for the dispatched operation. It never states that an asynchronously scheduled operation later completed, that any downstream physical action occurred, or that device state changed. This applies without exception across every protocol in discovery assessment §9's table, including REST — a 2xx HTTP response is application/protocol acknowledgement, in exactly the same sense a KNX bus telegram's transmission is, and this ADR forbids treating REST's synchronous shape as evidence that its acknowledgement carries a stronger guarantee than any other protocol's.

### 14. Lifecycle-State Ownership

**Normative decision, scoped to the facts this ADR actually puts the protocol executor in a position to observe:** once the protocol-execution boundary has been entered (Decision 2), the protocol-executor role (ADR-0011 Decision 1) is the sole truth-owning observer of every execution-boundary lifecycle fact this ADR defines — a boundary-local `NOT_ATTEMPTED` (Decision 2's Cases B and C), `DISPATCHED`, `PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, `OUTCOME_UNKNOWN`, and resulting-state verification — because it is the only role with direct access to the protocol channel and its responses, or, for Case B specifically, the only role that attempted the record-creation step that failed.

**This does not extend to a pre-boundary `NOT_ATTEMPTED` (Decision 2's Case A).** The protocol executor is never invoked for `DENY`, evaluation failure, producer authentication or admission failure, request-composition failure, a malformed or contradictory gateway response, or gateway unavailability, and it therefore neither observes nor may assert a fact it had no access to. Truth-owning observation of a Case A `NOT_ATTEMPTED` belongs to whichever upstream role or component actually produced or detected that condition — `basis-core`'s evaluation, `basis-gateway`'s enforcement or admission handling, or the operation-producer role's own request composition and authentication, per each condition's own cause — restating `operation-producer-and-execution-boundary.md`'s existing provenance-and-fact-ownership assignment for those facts rather than assigning the protocol executor an observation it never made.

This does not decide, and does not need to decide, whether the same component also constructs or durably retains the record of the values it owns — that division of labor, mirroring the authorization side's `basis-adapters`/`basis-producer` construction/retention split under ADR-0007, is Gate 3's question (ADR-0011's own Gate 3 text: "Architecture must establish which logical role constructs execution evidence"). This ADR decides only whose *observation* is authoritative, not who *writes it down*.

### 15. Ordering

```text
authorize (basis-core evaluates; basis-gateway enforces)
    → ALLOW
        → authoritative permitting disposition reaches the protocol-execution boundary
            → protocol executor enters pre-dispatch boundary
                → create pre-dispatch record
                    → record creation fails → NOT_ATTEMPTED (terminal; Case B; fail-closed, Decision 16 —
                                               no record exists to resolve; not a record's resolution)
                    → record created (Decision 2) — not yet an execution attempt
                        → verify authorization-to-execution binding (ADR-0012, Gate 1), using that record
                            → verification fails, or another boundary-local condition arises →
                                pre-dispatch record resolves: NOT_ATTEMPTED (terminal; Case C; Decision 2)
                            → verification succeeds → dispatch committed → execution attempt begins: attempt-outcome = DISPATCHED
                                → observe protocol result: attempt-outcome resolves to a terminal value (Decision 4)
                                    → [Gate 3] construct and retain execution evidence
```

A Case A condition (`DENY`, evaluation failure, producer authentication/admission failure, gateway unavailability, and similar — Decision 2's pre-boundary case) never reaches `ALLOW`, or reaches it but never reaches the protocol-execution boundary; for these, the chain above never starts, the protocol executor is never invoked, and no pre-dispatch record is ever created — `NOT_ATTEMPTED` is reached directly, as a fact about the authorization chain's outcome.

This ordering is normative for every link this ADR governs. It does not permit dispatch to precede binding verification (already established by ADR-0012); it does not permit binding verification to precede creation of the pre-dispatch record (Decision 2) — verification is performed using a record that already exists, never used as the gate that decides whether a record gets created; it does not permit a record-creation failure (Case B) to be represented as though a record existed and resolved, when none was ever created; it does not permit the pre-dispatch record to be created after dispatch (Decision 2); it does not permit the pre-dispatch record to be conflated with, or represented as, an execution attempt before dispatch is committed to (Decision 2); and it does not permit evidence construction (Gate 3, wherever it is eventually placed in the pipeline) to gate, block, retroactively alter, or be conflated with any attempt-outcome or resulting-state-verification value already reached — a failure at the evidence-construction step is an evidence-durability-axis fact (Decision 6), never a reason to revise what actually happened at dispatch or observation time.

### 16. Fail-Closed Requirements

The operation resolves to `NOT_ATTEMPTED` when any of the conditions ADR-0011 Decision 9 and ADR-0012's Failure Semantics already establish hold, and the three cases this holds for (Decision 2) are not identical in what "fail-closed" requires of the protocol executor, or in whether a pre-dispatch record exists afterward. For Case A (`DENY`; failed evaluation; failed authentication; failed producer admission; failed request composition; malformed or contradictory gateway response), the operation never reaches the protocol executor at all — there is no dispatch decision for it to make, and no pre-dispatch record is ever created. For Case B, the protocol executor has entered the pre-dispatch boundary but fails to create the required pre-dispatch record; it must not dispatch, and the operation resolves directly to `NOT_ATTEMPTED` — not as an outcome a record holds, because no record exists to hold one. For Case C (no binding record; binding digest mismatch; non-permitting correlated disposition; already-consumed binding; binding-verification error; or another protocol-executor-local pre-dispatch failure arising after the record was created), the protocol executor has already created a pre-dispatch record; it must not dispatch, and must resolve that record to `NOT_ATTEMPTED`. This ADR adds three requirements specific to lifecycle representation:

- **Inability to create the pre-dispatch record is itself a fail-closed condition (Case B).** An executor that cannot represent, before dispatching, that a dispatch is about to be attempted must not dispatch "blind" — the inability to create the record is treated exactly as any other pre-dispatch failure, and resolves the operation to `NOT_ATTEMPTED` directly, never toward dispatch (Decision 2). This is a fact about the failed creation attempt itself, not an outcome recorded against a pre-dispatch record — no record exists for it to be recorded against.
- **Observation failure must resolve toward uncertainty, never toward a positive result.** If the mechanism used to observe a protocol response itself malfunctions, the attempt-outcome must resolve to `OUTCOME_UNKNOWN`, never be defaulted to `PROTOCOL_ACCEPTED`. An executor must never treat "I could not tell what happened" as though it were "it succeeded."
- **A later stage of a multi-stage attempt must not dispatch once an earlier stage has resolved to anything other than `PROTOCOL_ACCEPTED`.** This generalizes ADR-0011 Decision 9's "no dispatch before authoritative disposition" principle to inter-stage dispatch within one multi-stage protocol operation — a failed or unconfirmed SELECT must not be followed by an OPERATE (Decision 7).

### 17. Preserve Protocol Neutrality

Restated from ADR-0011 Decision 13 and applied specifically to this ADR's vocabulary: the attempt-outcome and resulting-state-verification axes are deliberately abstract enough to accommodate synchronous request/response (REST, most BACnet and Modbus services), asynchronous publish (MQTT, KNX), acknowledgement/no-acknowledgement variation within a single protocol (MQTT's QoS levels), multi-stage select/operate sequences (DNP3, IEC 61850), and protocols with no state-readback capability at all. This ADR does not define a REST-shaped or HTTP-shaped default that other protocols must contort themselves to fit; the discovery assessment §9 table's own finding — that REST's 2xx response deserves no more evidentiary weight than any other protocol's positive acknowledgement — is carried forward as a binding constraint (Decision 13), not a stylistic preference.

---

## Failure and Uncertainty Semantics

Extending, without weakening, `operation-producer-and-execution-boundary.md` §8's five preserved distinctions with the refinements this ADR's evidence requires:

```text
not executed               ≠  execution attempted
attempt outcome unknown    ≠  attempt outcome failed          (uncertainty is not negative evidence)
attempt outcome unknown    ≠  attempt not attempted            (uncertainty is not absence)
protocol rejected          ≠  authorization denied             (a remote negative response is not a local one)
protocol accepted          ≠  resulting state verified         (acknowledgement is not confirmation)
resulting state verified   ≠  resulting state unverifiable     (confirmation available is not the same as confirmation impossible)
evidence persistence failed ≠  execution failed                (the record's durability is not the event's outcome)
```

Selected scenarios, resolved into this ADR's vocabulary (restating and resolving discovery assessment §8's own scenario table):

| Scenario | Attempt outcome | State verification |
| - | - | - |
| Authorization denied, or evaluation/authentication/admission failed (Case A — never reaches the protocol executor) | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Protocol executor enters the boundary but cannot create the required pre-dispatch record (Case B — fail-closed; no record ever exists) | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Gate 1 binding verification fails against an already-created pre-dispatch record (Case C) | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Endpoint unreachable before any bytes sent, after the pre-dispatch record already exists (Case C) | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Timeout establishing a connection, before dispatch is committed to, after the pre-dispatch record already exists (Case C) | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Timeout after dispatch, no response | `OUTCOME_UNKNOWN` (ack-timeout reason) | `STATE_NOT_VERIFIED` unless independently confirmed |
| Fire-and-forget publish with no acknowledgement primitive (MQTT QoS 0, KNX telegram) | `OUTCOME_UNKNOWN` (no-ack-primitive reason) | `VERIFICATION_NOT_APPLICABLE` unless a separate read-back exists |
| Protocol-level negative response (Modbus exception, BACnet `Error-PDU`) | `PROTOCOL_REJECTED` | `VERIFICATION_NOT_APPLICABLE` |
| Positive protocol-level acknowledgement, no independent state confirmation | `PROTOCOL_ACCEPTED` | `STATE_NOT_VERIFIED` |
| Positive protocol-level acknowledgement, independent read-back confirms state | `PROTOCOL_ACCEPTED` | `STATE_VERIFIED` |
| SELECT accepted, OPERATE times out (DNP3/IEC 61850) | Overall `OUTCOME_UNKNOWN` (from the OPERATE stage, per Decision 7) | `STATE_NOT_VERIFIED` |
| Process crash after dispatch, before observation completes | `OUTCOME_UNKNOWN` on recovery, if the attempt can be recovered at all (Decision 2) | `STATE_NOT_VERIFIED` |
| Durable evidence write fails after a `PROTOCOL_ACCEPTED` observation | `PROTOCOL_ACCEPTED` (unchanged) | whatever verification value already applied (unchanged); evidence-durability axis records the failure separately (Decision 6) |
| Retry after `OUTCOME_UNKNOWN` | A new, correlated attempt (Decision 9), with its own fresh binding | Independent of the prior attempt's value |

---

## Lifecycle Model

```text
  create pre-dispatch record
        │
        ├── record creation fails ──▶  NOT_ATTEMPTED  (terminal; Case B; fail-closed; no record ever existed to resolve)
        │
        └── record created (Decision 2) — not yet an execution attempt
                │
                ▼
          verify ADR-0012 authorization-to-execution binding, using that record
                │
                ├── verification fails, or another boundary-local condition ──▶  NOT_ATTEMPTED  (terminal; Case C; the record's own resolution)
                │
                └── verification succeeds, dispatch committed, execution attempt begins
                 ┌───────────────────────────────────────────────┐
                 │              Attempt-outcome axis               │
                 │                                                 │
                 │   DISPATCHED ─┬─▶ PROTOCOL_ACCEPTED               │
                 │  (transient)  ├─▶ PROTOCOL_REJECTED               │
                 │               └─▶ OUTCOME_UNKNOWN                 │
                 └───────────────────────────────────────────────┘
                                    │
                                    ▼ (only once outcome is known)
                 ┌───────────────────────────────────────────────┐
                 │         Resulting-state-verification axis      │
                 │  VERIFICATION_NOT_APPLICABLE / STATE_VERIFIED /  │
                 │  STATE_NOT_VERIFIED                              │
                 └───────────────────────────────────────────────┘
                                    │
                                    ▼ (independent of the above)
                 ┌───────────────────────────────────────────────┐
                 │             Evidence-durability axis            │
                 │        (values and ownership: Gate 3)           │
                 └───────────────────────────────────────────────┘
```

A multi-stage attempt (Decision 7) runs this attempt-outcome sub-diagram once per stage, in sequence, with the overall attempt's outcome equal to its weakest stage's outcome.

---

## Alternatives Considered

**A — Retain the interoperability roadmap's five-state sketch unchanged.** Rejected. The discovery assessment §8 found it insufficient on repository-anchored evidence: it conflates local and remote rejection, and it conflates "no acknowledgement primitive" with "acknowledgement absent." Adopting it as governed vocabulary despite that finding would encode a known truthfulness gap into architecture.

**B — A single, larger flat enum (ten to twelve values) instead of independent axes.** Considered and rejected. Enumerating every combination of attempt outcome and state verification as separate flat values (for example, `PROTOCOL_ACCEPTED_STATE_VERIFIED`, `PROTOCOL_ACCEPTED_STATE_NOT_VERIFIED`, and so on) reproduces the same conflation risk the discovery assessment already found in the five-state sketch, just with more states — a `partially-applied`-shaped catch-all becomes likely again the moment a new combination doesn't map cleanly onto an existing flat value. Independent axes compose without needing to be pre-enumerated (Decision 3).

**C — Defer lifecycle semantics entirely until execution-evidence schema work (Gate 3) settles the question implicitly.** Rejected. ADR-0011 names Gate 2 as its own unconditional prerequisite, separate from and prior to Gate 3, specifically because a schema cannot honestly represent states that have not yet been defined, and Gate 3's own text (ADR-0011) presupposes a lifecycle vocabulary to serialize.

**D — Resolve broader replay/freshness in full within this ADR.** Rejected. ADR-0012 deliberately left wall-clock expiry, policy-reload invalidation, restart-surviving replay prevention, and a replay database open for a dedicated future decision; resolving all of it here would preempt architecture ADR-0012 explicitly reserved, without the evidentiary basis that decision requires (no wall-clock or restart-survival evidence was gathered for this ADR).

**E — Leave replay/freshness completely untouched by this ADR, deferring all of it, including attempt correlation.** Considered and rejected as too narrow. Defining what an execution attempt is (Decision 1) and how a retry relates to a prior attempt (Decision 9) cannot be done without touching *some* part of replay's shape — the alternative would leave "what happens on a second dispatch attempt" as an unstated gap in this ADR's own lifecycle model, which is the ambiguity Gate 2 exists to close. This ADR resolves only the lifecycle-representation portion (Decision 9, Decision 10) and states plainly what remains open.

**F — Assign lifecycle-state ownership to the execution-evidence producer role rather than the protocol executor.** Rejected. The execution-evidence producer role, wherever Gate 3 eventually places it, does not have direct access to the protocol channel; assigning it truth-ownership over facts it cannot itself observe would repeat, for execution, the exact "normalization drift" risk ADR-0007 already identified and avoided for authorization evidence by keeping construction and retention separate but neither one detached from direct observation.

---

## Consequences

### Positive

- BASIS now has a normative execution-lifecycle vocabulary that preserves every distinction the authorization side of the architecture has always required, extended honestly to execution.
- The vocabulary is protocol-neutral by construction, verified against all nine adapters' own accepted protocol-semantics evidence rather than assumed from HTTP.
- Multi-stage select/operate protocols (DNP3, IEC 61850) have a truthful representation that does not invent an unsupported physical-state claim.
- A future execution-evidence schema (Gate 3) has a stable vocabulary to serialize, rather than needing to invent lifecycle semantics as a side effect of schema design.
- The narrow retry/correlation resolution (Decision 9) closes an otherwise-unstated gap in this ADR's own model without preempting the broader replay/freshness decision ADR-0012 reserved.

### Negative / Tradeoff

- Three independently tracked axes are more complex to implement correctly than one flat status field, and that complexity is deliberate, not incidental.
- Some protocol-specific nuance (exactly which no-ack reasons are distinguishable for which protocol) is left to Gate 3's field-level design rather than fixed here, meaning this ADR's vocabulary is necessarily coarser than a full execution-evidence contract will eventually need.
- Broader replay/freshness remains unresolved, and this ADR's narrow attempt-correlation resolution should not be mistaken for a full answer — a future implementer must still wait for the dedicated decision this ADR does not make.
- Gate 3 now inherits a specific vocabulary it must be consistent with, rather than having a blank slate — a constraint accepted deliberately so evidence work does not silently redefine lifecycle semantics as a side effect of schema convenience.

---

## Security Consequences

### Prevented by this architecture

- Reporting an uncertain outcome as though it were a known negative or a known positive result.
- Treating a protocol-level acknowledgement as proof of physical state change.
- Treating a partially progressed multi-stage protocol transaction as evidence of a specific fraction of physical completion.
- Silently resolving an observation failure toward a positive result.
- Dispatching a later stage of a multi-stage operation after an earlier stage failed or resolved to unknown.
- Reusing a consumed authorization-to-execution binding for a second dispatch attempt (restating, not extending, ADR-0012).
- Conflating evidence-persistence failure with execution failure.

### Still requiring follow-on architecture

- Execution-evidence schema, field names, serialization, and ownership (Gate 3).
- Broader replay/freshness: wall-clock expiry, policy-reload invalidation, restart-surviving replay prevention, retry/idempotency behavior, and any replay database (deferred, per Decision 10).
- Protocol/device credential custody, if the eventually selected bounded target requires one (Gate 4).
- Provenance classification for protocol-dispatch-result and resulting-state-confirmation facts, if none of the existing six `ProvenanceClassification` values fit (a Gate 3 question the discovery assessment §7 already names).

### Residual risks

- A compromised protocol-executor role can fabricate any execution-boundary lifecycle fact it truth-owns once the boundary is entered (Decision 14) — a boundary-local `NOT_ATTEMPTED` (Decision 2's Cases B and C), `DISPATCHED`, `PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, `OUTCOME_UNKNOWN`, or resulting-state verification — because it is the sole authoritative observer of those facts, the same residual-risk structure ADR-0011 Decision 19 and ADR-0012 already accept for a compromised same-process component, restated here rather than newly introduced. **It does not extend to a pre-boundary `NOT_ATTEMPTED` (Decision 2's Case A): the protocol executor was never invoked and never observed that fact, so a compromised executor has nothing of its own to fabricate there — fabrication risk for a Case A fact belongs to whichever upstream role or component actually produced it (Decision 14), and is that role's own residual risk, not this ADR's to restate.**
- This ADR's vocabulary constrains what a conforming implementation may honestly report; it does not, by itself, verify that an implementation is conforming.

---

## Relationship to ADR-0011

This ADR resolves the Gate 2 execution-lifecycle specification exactly as ADR-0011 named it: a normative specification defining truthful execution-state semantics, preserving Decision 12's uncertainty requirement, and not assuming HTTP-style acknowledgement across every protocol (Decision 13). This ADR resolves Gate 2 as ADR-0011 described it; acceptance does not authorize implementation. It does not reopen ADR-0011's protocol-executor role establishment, its same-process default-topology selection, or its governing invariants — it operates entirely within the space ADR-0011 left open for Gate 2, and it treats ADR-0011's Decision 9 (no dispatch before authoritative permission), Decision 11 (authorization and execution evidence remain separate), and Decision 12 (preserve uncertainty) as binding constraints this ADR must satisfy rather than revisit. This ADR's fail-closed requirements (Decision 16) are additive to, and do not weaken, ADR-0011 Decision 9's existing list.

## Relationship to ADR-0012

This ADR builds directly on ADR-0012's binding model: an execution attempt (Decision 1) is scoped to exactly one ADR-0012 binding record, the pre-dispatch record (Decision 2) — not itself an execution attempt — is created *before* binding verification is performed, not after it succeeds (Decision 15's ordering), so that a binding-verification failure resolves an already-existing record rather than requiring one to be invented retroactively; and a retry attempt following dispatch (Decision 9) is only possible because ADR-0012's single-consumption guarantee forces a new binding cycle rather than reuse of a consumed one; ADR-0012's single-consumption guarantee does not, by itself, decide what may follow a pre-dispatch `NOT_ATTEMPTED` condition in which no binding was ever consumed (Decision 9). This ADR narrows the disposition of ADR-0012's open replay/freshness question (Decision 10, above): a portion — attempt multiplicity and its binding-record consequence — is resolved here, upon this ADR's own acceptance; the rest is explicitly not addressed here, and this ADR does not claim to be the "dedicated future architecture decision" ADR-0012 anticipated for the remainder. This ADR does not reopen ADR-0012's binding-model decision, its trust assumptions, or its failure semantics; it treats ADR-0012's Failure Semantics section as the authoritative source for pre-dispatch fail-closed conditions and restates it by reference (Decision 16) rather than by re-derivation.

## Relationship to Gate 3

This ADR is a prerequisite for Gate 3, not an overlap with it. It fixes the vocabulary — the attempt-outcome and resulting-state-verification axes, their values, and their transition rules — that a future execution-evidence specification must serialize; it does not itself define that specification. Specifically left to Gate 3, and explicitly not decided here: which logical role constructs execution evidence and which retains it, and whether they are the same component (Decision 14 decides only observation ownership, not construction/retention ownership); the evidence-durability axis's values, field names, and ownership (Decision 6); the exact reasons distinguishable within `OUTCOME_UNKNOWN` for a given protocol, as governed fields (Decision 4's note); minimum correlation-identifier requirements for a future execution-evidence record, including the concrete evidence carrier and reference shape for the retry-lineage correlation this ADR already requires semantically, but does not itself carry in any particular record shape (Decision 9), and any correlation requirements beyond it; whether the existing six-value `ProvenanceClassification` enum suffices for protocol-dispatch-result and resulting-state-confirmation facts, or a new value is required; and persistence/failure ordering for the evidence-construction step itself, beyond the requirement that it never gate or retroactively alter an already-reached attempt-outcome or verification value (Decision 15). Gate 3 was outstanding after this ADR alone, exactly as ADR-0011 named it. Gate 3 is now resolved by Accepted ADR-0014, accepted together with this ADR, ADR-0015, and ADR-0016 in the same coupled execution-readiness milestone — this ADR's own Decision does not itself resolve Gate 3.

---

## Non-Goals

This ADR does not: implement execution; add protocol dispatch code for REST, BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, or Niagara; modify `basis-producer`, `basis-gateway`, `basis-adapters`, `basis-core`, `basis-identity`, or `basis-schemas`; create `basis-executor` or any other new repository; define an execution-evidence schema or any `basis-schemas` contract; select field names, enum value identifiers, or a serialization format for any axis defined here; define who constructs or retains execution evidence (Gate 3); resolve broader replay/freshness beyond the narrow attempt-correlation consequence stated in Decision 9 and Decision 10; define device/protocol credential custody (Gate 4); create an implementation plan, a PR sequence, or a milestone count; imply a `basis-producer` "Phase 6"; change ADR-0011's or ADR-0012's status; or mark itself `Accepted`. Any class, module, function, or API name that appears anywhere in this document is a non-normative illustration of a concept, never a specification of an interface.

---

## Validation / Implementation Gate

This ADR's merging did not itself constitute acceptance, consistent with this repository's established convention (see ADR-0011's and ADR-0012's own Validation / Implementation Gate sections and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself change its status to `Accepted`. After this ADR merged, a separate, dedicated formal architecture and governance review accepted it together with ADR-0014, ADR-0015, and ADR-0016, as one coupled execution-readiness milestone; this ADR's status now records `Accepted`. Acceptance establishes the execution-lifecycle vocabulary defined here as the governed semantics a future bounded execution implementation must satisfy — it does not, by itself, authorize that implementation. Per ADR-0011's own Follow-On Decision Gates and Validation / Implementation Gate sections, Gates 1 through 3 are unconditional prerequisites, and Gate 4 is conditional; acceptance of this ADR resolves Gate 2. Gate 3 (minimum execution-evidence semantics) is resolved together with this ADR by Accepted ADR-0014. Gate 4 (device/protocol credential custody) is not triggered by the credential-free target Accepted ADR-0015 establishes; it remains conditionally outstanding only for a future target that requires a protocol/device credential. No implementation is authorized by this ADR's acceptance.

## References

- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — establishes the protocol-executor role, the same-process first-reference topology, Decision 12's uncertainty-preservation requirement, and Gate 2 as the unconditional prerequisite this ADR resolves
- [ADR-0012](0012-authorization-to-execution-binding.md) — resolves Gate 1's same-process binding mechanism and narrows Gate 1's replay/freshness requirement to process-local single consumption; leaves open whether a Gate 2 specification would resolve the remainder, which this ADR partially and explicitly resolves (Decision 10)
- [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) §8 (execution lifecycle and failure semantics — the primary evidentiary source for this ADR's vocabulary), §9 (protocol-specific execution semantics — the per-protocol evidence this ADR's neutrality requirement is verified against), §7 (replay and freshness), §14 (side-effect ordering — the crash-consistency analysis behind Decision 2), §19 (decision inventory naming the execution-lifecycle vocabulary as required before implementation)
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §5 (authorization-to-execution lifecycle invariants this ADR builds on without revising), §8 (the five preserved failure/degraded-condition distinctions this ADR extends)
- [ADR-0007](0007-adapter-evidence-construction.md) — the construction/retention separation precedent this ADR's Decision 14 reasons from without extending to execution evidence itself
- [`docs/architecture/adapter-evidence-construction-semantics.md`](../architecture/adapter-evidence-construction-semantics.md) — the "implementation proves a stable shape" schema-publication discipline this ADR's deferral to Gate 3 follows
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — referenced by ADR-0012 for the same publication discipline
- [`docs/glossary.md`](../glossary.md) — terminology this ADR's vocabulary is reconciled against
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR proposal and acceptance process this ADR follows
- [`docs/adr/README.md`](README.md) — ADR lifecycle states and required sections
