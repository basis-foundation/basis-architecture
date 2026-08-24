# ADR-0013: Execution Lifecycle Semantics

## Status

Proposed

## Context

[ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — Accepted — established a distinct logical protocol-executor role, architecturally separate from the operation-producer role, and selected same-process colocation with the existing `basis-producer` reference implementation as the default topology for the first bounded execution reference slice. It fixed the invariants execution must preserve regardless of topology, and it named three unconditional prerequisites, plus a fourth conditional one, before any bounded execution implementation may dispatch a protocol operation: Gate 1 (authorization-to-execution binding), Gate 2 (execution-lifecycle semantics), Gate 3 (minimum execution-evidence semantics), and Gate 4 (protocol/device credential custody, only if the eventually selected bounded target requires one). ADR-0011's own Decision 12 already states a truthfulness principle Gate 2 must satisfy — "execution reporting must preserve uncertainty rather than infer non-occurrence from missing confirmation" — and lists, without making normative, the distinctions a future execution-lifecycle specification must preserve: not attempted; attempted; remote acknowledgement observed; remote rejection observed; outcome unknown; resulting state verified; resulting state not verified; and evidence persistence success/failure. ADR-0011's own Gate 2 description is narrower and more direct: "A normative architecture specification must define truthful execution-state semantics sufficient for the first bounded protocol. It must preserve uncertainty (Decision 12) and avoid assuming all protocols have HTTP-style acknowledgement behavior. This ADR does not define the final enum."

[ADR-0012](0012-authorization-to-execution-binding.md) — Accepted — resolved Gate 1 for the same-process first-reference topology: a same-process authorization-to-execution binding record, created by the operation-producer role before gateway submission and verified by the protocol-executor role immediately before dispatch, establishes by content-based digest comparison that the operation about to be dispatched is unchanged from the operation whose normalized semantics received the authoritative permitting disposition. ADR-0012 also narrowed Gate 1's replay/freshness requirement to a **process-local single-consumption guarantee** — a binding record may be consumed by at most one dispatch attempt within the process's own running lifetime — while stating explicitly, and repeatedly, that broader replay/freshness properties (wall-clock expiry, policy-reload invalidation, restart-surviving replay prevention, retry/idempotency behavior, a replay database) remain open, and that whether those broader properties "are eventually resolved by a Gate 2 execution-lifecycle specification, a dedicated Gate 1 addendum, or another future architecture decision is not decided here." This ADR is the first point at which that open question must be answered one way or the other; see **Replay and Freshness — Bounded Resolution**, below.

The merged, non-normative [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) is the evidentiary basis for this ADR, exactly as it was for ADR-0011 and ADR-0012. Its §8 ("Execution Lifecycle and Failure Semantics") is the primary source: it evaluates the interoperability roadmap's five-state sketch (`authorized-but-not-attempted`, `attempted-and-completed`, `attempted-and-failed`, `partially-applied`, `execution-status-unavailable`) — already documented elsewhere as "roadmap language, not a governed contract" — against the nine adapters' own accepted protocol-semantics evidence in §9, and finds the five-state sketch insufficient in at least two ways: it conflates a local pre-dispatch rejection with a remote protocol-level rejection, and it conflates "no acknowledgement primitive exists for this protocol" with "an acknowledgement primitive exists but did not arrive in time." §8 states two governing distinctions as its load-bearing finding: `failed` and `not executed` are not equivalent, and `timeout` is evidence of uncertainty, not evidence of non-occurrence. §9's protocol-by-protocol table is direct, repository-anchored evidence that no single request/response, HTTP-shaped lifecycle can honestly describe DNP3 and IEC 61850's stateless select/operate model, MQTT's fire-and-forget publish semantics, or KNX's bus-level telegrams without either lying about confirmation the protocol cannot provide, or discarding a real distinction (protocol-transaction progress versus physical device-state confirmation) the architecture has never previously permitted itself to discard.

[`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §5 and §8 are restated here, not re-derived: §5's authorization-to-execution lifecycle invariants (no dispatch before authoritative disposition; authorization evidence and execution evidence are separate artifacts; execution failure does not retroactively invalidate a correct authorization decision; execution success cannot retroactively legitimize an unauthorized dispatch) and §8's five preserved distinctions among failure and degraded conditions (`not executed ≠ execution failed`; `execution failed ≠ execution status unknown`; `execution status unknown ≠ authorization denied`; `authorization denied ≠ authorization evaluation failed`; and `not executed` as a family of causes, not one condition) are the foundation this ADR's lifecycle model is built on top of, not a foundation it revisits.

Consistent with this repository's ADR-acceptance governance convention (ADR-0007 through ADR-0012 were each first merged with `Status: Proposed`, and a separate, dedicated follow-up PR later changed the status to `Accepted` after independent architecture and governance review — see [`docs/adr/README.md`](README.md#lifecycle-states)), this ADR is submitted as `Proposed`. Nothing in this ADR, and nothing in its merging, constitutes acceptance. No execution lifecycle vocabulary defined here is authoritative until a separate, dedicated formal-acceptance review changes this ADR's status to `Accepted`, and acceptance itself would not authorize implementation — see **Validation / Implementation Gate**, below.

The current implemented state remains exactly what ADR-0011 and ADR-0012 both describe: `basis-producer`'s completed bounded authorization slice (Phase 2A through Phase 5, merged) stops at the authoritative disposition; `basis_producer.rest_composition`'s own module docstring states, as a scope boundary, that the module will not "execute any protocol operation under any disposition, including `ALLOW`." Nothing in this ADR changes that. No protocol operation is executed anywhere in the ecosystem today, and this ADR's proposal — let alone its eventual acceptance — does not change that either.

## Problem

Once a protocol dispatch has been authorized (`ALLOW`, per ADR-0011 Decision 9) and bound to the exact authorized operation (ADR-0012's binding record, verified immediately before dispatch), what lifecycle states can an execution attempt truthfully occupy, in what order may it move between them, and how must partial protocol progression, missing confirmation, and evidence-persistence failure each be represented — without any state ever implying stronger truth than the protocol, or the executor's own observation of it, can actually support?

The answer must, at minimum: preserve every distinction ADR-0011 Decision 12 and `operation-producer-and-execution-boundary.md` §8 already require; avoid presupposing that every protocol offers HTTP-style synchronous acknowledgement (ADR-0011 Decision 13, the discovery assessment §9's own protocol table); avoid collapsing "the protocol has no confirmation primitive" into "confirmation didn't arrive" (discovery assessment §8); avoid inventing a single `partially-applied` state that conflates a protocol transaction stopping partway through (a structural, frequent outcome for DNP3 and IEC 61850's select/operate model) with a claim about physical device state the protocol evidence does not support (discovery assessment §9); and state plainly which logical role's observation is authoritative for a lifecycle-state assertion, without deciding who durably records it (Gate 3). The answer must also decide — because ADR-0012 explicitly left the question open rather than resolved or reassigned it — whether any part of Gate 1's outstanding broader replay/freshness requirement is inseparable from defining what an execution attempt is, and if so, exactly how much of it this ADR resolves.

## Decision

**ADR-0013 defines the minimum truthful execution-lifecycle semantics required before BASIS can plan a bounded protocol-execution implementation.** It fixes what counts as an execution attempt and when one begins; establishes a lifecycle model built from three independently tracked, orthogonal facts — attempt outcome, resulting-state verification, and evidence durability — rather than one flat enum; fixes the ordering among authorization, binding verification, dispatch, observation, and (future) evidence construction; assigns truth-owning observation of lifecycle state to the protocol-executor role ADR-0011 established; states what must fail closed; and resolves the narrow, lifecycle-representation portion of Gate 1's outstanding replay/freshness question while explicitly declining to resolve the rest of it. It does not define an execution-evidence schema, does not select field names, enum value names, or a serialization, does not create an implementation plan, and does not authorize protocol execution.

### 1. What Counts as an Execution Attempt

An **execution attempt** is the logical unit of one protocol-executor dispatch action directed at the preserved, bound protocol operation (ADR-0011 Decision 7; ADR-0012's binding record), together with whatever the protocol-executor role subsequently observes about that specific dispatch before it stops observing. An execution attempt is scoped to exactly one authorization-to-execution binding record (ADR-0012); it is never shared across two different bindings, and a binding record, once consumed, authorizes at most one execution attempt (restating, not extending, ADR-0012's single-consumption guarantee). A multi-stage protocol operation (DNP3 and IEC 61850's select/operate model) is one execution attempt composed of an ordered sequence of stage-attempts — see **Multi-Stage Protocol Operations**, below — not several independent execution attempts.

### 2. An Attempt Does Not Exist Before Dispatch Is Committed To — But Its Record Must

**Normative requirement:** the semantic content of an execution attempt — the fact that a protocol dispatch happened — cannot exist before dispatch, because nothing has yet occurred for it to describe. This ADR rejects the inference that authorization (`ALLOW`) or binding verification alone constitutes an execution attempt; an `ALLOW` disposition that is never dispatched is not an attempt of any kind, it is simply an unexecuted authorization, exactly as `operation-producer-and-execution-boundary.md` §8 already classifies it (`not executed`).

**A separate, narrower requirement governs the attempt *record*, not the attempt itself:** the protocol-executor role must create a local record marking that an attempt is about to begin — in the attempt-outcome vocabulary below, this is the transition into a pre-dispatch state — immediately before the irreversible dispatch action, and never after it. This ordering exists so that a crash between record-creation and dispatch is distinguishable, in principle, from a crash between dispatch and observation, per the discovery assessment §14's crash-consistency analysis and its §8 finding that "process crash before dispatch" is only classifiable as `not attempted` "if durably recorded before the crash." This ADR requires the *ordering* (record, then dispatch); it does not require the record to be durable across a process restart — whether and how that record must survive a crash, and whether it becomes part of a future execution-evidence artifact, is a Gate 3 question this ADR forwards rather than answers. A protocol executor that cannot create this record for whatever reason must not dispatch anyway (see **Fail-Closed Requirements**, below).

### 3. Three Independent Axes, Not One Enum

**Normative decision:** an execution attempt's truthful description requires three independently tracked facts, not one combined status field. Collapsing them was exactly the discovery assessment's finding against the five-state sketch — "operation accepted but state change unverified... a distinction the five states do not currently carry a field for," and "evidence persistence failure after device action... is a *distinct* failure from any execution-status value." The three axes:

1. **Attempt outcome** — what the protocol itself reported (or failed to report) about the dispatch. Owned and asserted by the protocol-executor role (Decision 13, below).
2. **Resulting-state verification** — whether the physical or logical state the operation was meant to change was independently confirmed, distinct from whether the protocol accepted the dispatch. Meaningful only once attempt outcome has reached a non-`NOT_ATTEMPTED` value; often unavailable for protocols or operations that offer no read-back.
3. **Evidence durability** — whether the record of the above two facts was itself durably persisted. This axis describes the record, not the event the record describes, and a failure on this axis never changes the value of either of the other two axes.

No axis may be inferred from another. A `NOT_VERIFIED` result on axis 2 does not imply anything negative about axis 1, and an `EVIDENCE_PERSISTENCE_FAILED` result on axis 3 does not imply the operation did not execute — it implies only that the durable record of what happened, whatever it was, does not yet reliably exist.

### 4. Attempt-Outcome Axis — Definitions

| Value | Meaning | Terminal for this attempt? |
| - | - | - |
| `NOT_ATTEMPTED` | No protocol dispatch occurred for this attempt. A family of causes, not one condition — restates `operation-producer-and-execution-boundary.md` §8's existing non-exhaustive set (denied, evaluation failed, producer authentication or admission failed, producer-only context rejected, gateway unavailable, timeout before dispatch) and adds the ADR-0012 binding-verification failure conditions (no binding record found; digest mismatch; disposition not permitting; binding already consumed; verification error) as additional, equally valid members of the same family. | Yes — this attempt never began. |
| `DISPATCHED` | The protocol-executor role has committed to and completed the irreversible act of sending the protocol-shaped operation toward its endpoint. Transient: an attempt does not remain in this state once an outcome is known or the executor stops waiting for one. | No — a resolving value below is always eventually assigned, even if that value is `OUTCOME_UNKNOWN`. |
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
NOT_ATTEMPTED  ──dispatch committed, per Decision 2──▶  DISPATCHED

DISPATCHED  ──positive protocol-level response──▶  PROTOCOL_ACCEPTED
DISPATCHED  ──negative protocol-level response──▶  PROTOCOL_REJECTED
DISPATCHED  ──no definitive response within the executor's observation window──▶  OUTCOME_UNKNOWN
```

Governing rules:

- `NOT_ATTEMPTED` is terminal for a given attempt. This ADR does not define a transition out of `NOT_ATTEMPTED` for the same attempt; a later dispatch of the same authorized operation is, by Decision 1 and Decision 9 below, a distinct attempt with its own binding record, not a continuation of the prior one.
- `DISPATCHED` is transient. This ADR does not permit an attempt to remain indefinitely in `DISPATCHED` as a reported terminal value; an executor that stops waiting without a definitive answer must resolve the attempt to `OUTCOME_UNKNOWN`, never leave it unresolved and never silently resolve it to `PROTOCOL_ACCEPTED`.
- `DISPATCHED → NOT_ATTEMPTED` does not exist. Dispatch is a one-way transition once committed to (Decision 2). A process that crashes after dispatch and never learns the outcome must resolve, on recovery if it can determine an attempt was made, to `OUTCOME_UNKNOWN` — never back to `NOT_ATTEMPTED` — because bytes may have already reached the endpoint. Whether a recovering process even *can* determine an attempt was made depends on the durability of the pre-dispatch record (Decision 2), which this ADR forwards to Gate 3 rather than resolves.
- `PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, and `OUTCOME_UNKNOWN` are each terminal for the stage/attempt they describe. None of the three transitions into another without a new dispatch action, which constitutes either the next stage of the same multi-stage attempt (Decision 7) or a new attempt entirely (Decision 9).

### 9. Retry and Attempt Correlation — Bounded Replay Resolution

**Normative requirement:** any dispatch action following a prior attempt's terminal outcome — for the same underlying authorized operation — is a new, distinct execution attempt, correlated to the prior one by reference, never represented as a mutation of the prior attempt's record. This is inseparable from Decision 1's definition of an execution attempt as scoped to exactly one binding record: because ADR-0012's binding record is single-consumption, no second dispatch can occur under the *same* binding once the first has consumed it — a retry, if one occurs at all, necessarily requires the operation to be re-authorized and re-bound (a new `ALLOW`, a new ADR-0012 binding record), not a re-use of the exhausted one. This ADR therefore requires, as a lifecycle-representation fact rather than a security mechanism: an execution attempt record must carry a reference to the prior attempt it correlates to, when one exists, so that a sequence of retries for the same logical operation remains reconstructible; and the fact that a new attempt exists is itself evidence that a new authorization-to-execution binding was created and consumed for it, per ADR-0012.

### 10. Replay and Freshness — Bounded Resolution

ADR-0012 left open whether Gate 1's broader replay/freshness properties — wall-clock expiry, policy-reload invalidation of a pending disposition, restart-surviving replay prevention, retry/idempotency behavior, and any replay database — would be resolved by a Gate 2 execution-lifecycle specification, a dedicated Gate 1 addendum, or another future architecture decision, and stated plainly that implementation must not proceed as though they were already answered.

**This ADR resolves only the portion of that question inseparable from defining what an execution attempt is** (Decision 9, above): attempt multiplicity is represented as distinct, correlated attempt records, and any attempt after the first requires its own fresh authorization-to-execution binding under ADR-0012's existing single-consumption rule. This is a lifecycle-representation consequence of ADR-0012, not a new replay-prevention mechanism, and this ADR does not present it as resolving Gate 1's broader requirement.

**This ADR explicitly does not resolve, and defers to a dedicated future architecture decision:** whether a pending or already-consumed disposition expires on a wall clock; whether a policy-bundle reload between authorization and dispatch invalidates a disposition that has not yet been dispatched; whether a replay record must survive a process restart; whether or how a caller may request a retry, or how many retries are permitted; and whether a replay database of any kind is required. This ADR does not authorize implementation to treat any of those questions as answered, and does not itself constitute the "dedicated future architecture decision" ADR-0012 anticipated for the properties it leaves open.

### 11. Authorization Success Differs From Execution Success

**Normative restatement, not a new rule:** `ALLOW` means `basis-core` and `basis-gateway` determined the operation was permitted to be dispatched; it is a fact about the authorization decision, complete and immutable the moment it is produced (`operation-aware-trace-audit-evidence.md`; `operation-producer-and-execution-boundary.md` §5). It is never proof that dispatch occurred, that the protocol accepted it, or that any resulting state changed. This ADR's attempt-outcome axis (Decision 4) describes a fact that can only begin to exist after `ALLOW` and after Gate 1 binding verification succeed — the two facts are produced by different roles, at different times, and this ADR does not permit either to stand in for the other in a governed record.

### 12. Dispatch Differs From Completion

`DISPATCHED` (Decision 4) marks only that the protocol-executor role has sent the operation; it is not synonymous with, and must never be reported as equivalent to, a resolved outcome. "Completion," in this ADR's vocabulary, means an attempt or stage has reached a terminal attempt-outcome value (`PROTOCOL_ACCEPTED`, `PROTOCOL_REJECTED`, or `OUTCOME_UNKNOWN`) — not that the underlying real-world action is confirmed to have finished, which is a separate question the resulting-state-verification axis (Decision 5) governs independently.

### 13. Protocol Acknowledgement Is Not Completion

**Normative restatement of a finding already established, applied here as a binding constraint on the vocabulary:** `PROTOCOL_ACCEPTED` states only that the protocol layer accepted, positively responded to, or returned a success status for the dispatched operation. It never states that an asynchronously scheduled operation later completed, that any downstream physical action occurred, or that device state changed. This applies without exception across every protocol in discovery assessment §9's table, including REST — a 2xx HTTP response is application/protocol acknowledgement, in exactly the same sense a KNX bus telegram's transmission is, and this ADR forbids treating REST's synchronous shape as evidence that its acknowledgement carries a stronger guarantee than any other protocol's.

### 14. Lifecycle-State Ownership

**Normative decision:** the protocol-executor role (ADR-0011 Decision 1) is the sole truth-owning observer of attempt-outcome and resulting-state-verification values, because it is the only role with direct access to the protocol channel and its responses. This does not decide, and does not need to decide, whether the same component also constructs or durably retains the record of those values — that division of labor, mirroring the authorization side's `basis-adapters`/`basis-producer` construction/retention split under ADR-0007, is Gate 3's question (ADR-0011's own Gate 3 text: "Architecture must establish which logical role constructs execution evidence"). This ADR decides only whose *observation* is authoritative, not who *writes it down*.

### 15. Ordering

```text
authorize (basis-core evaluates; basis-gateway enforces)
    → ALLOW
        → verify authorization-to-execution binding (ADR-0012, Gate 1)
            → create pre-dispatch attempt record: attempt-outcome = NOT_ATTEMPTED (Decision 2)
                → dispatch: attempt-outcome = DISPATCHED
                    → observe protocol result: attempt-outcome resolves to a terminal value (Decision 4)
                        → [Gate 3] construct and retain execution evidence
```

This ordering is normative for every link this ADR governs. It does not permit dispatch to precede binding verification (already established by ADR-0012); it does not permit the pre-dispatch attempt record to be created after dispatch (Decision 2); and it does not permit evidence construction (Gate 3, wherever it is eventually placed in the pipeline) to gate, block, retroactively alter, or be conflated with any attempt-outcome or resulting-state-verification value already reached — a failure at the evidence-construction step is an evidence-durability-axis fact (Decision 6), never a reason to revise what actually happened at dispatch or observation time.

### 16. Fail-Closed Requirements

A protocol executor must not dispatch, and must resolve the attempt to `NOT_ATTEMPTED`, when any of the conditions ADR-0011 Decision 9 and ADR-0012's Failure Semantics already establish hold (`DENY`; failed evaluation; failed authentication; failed producer admission; failed request composition; malformed or contradictory gateway response; no binding record; binding digest mismatch; non-permitting correlated disposition; already-consumed binding; binding-verification error). This ADR adds three requirements specific to lifecycle representation:

- **Inability to create the pre-dispatch attempt record is itself a fail-closed condition.** An executor that cannot represent an attempt truthfully before dispatching must not dispatch "blind" — the inability to record is treated exactly as any other pre-dispatch failure (Decision 2).
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
| Authorization denied, or evaluation/authentication/admission failed | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Gate 1 binding verification fails | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Endpoint unreachable before any bytes sent | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
| Timeout establishing a connection, before dispatch is committed to | `NOT_ATTEMPTED` | `VERIFICATION_NOT_APPLICABLE` |
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
                 ┌───────────────────────────────────────────────┐
                 │              Attempt-outcome axis               │
                 │                                                 │
  NOT_ATTEMPTED ─┼─▶ DISPATCHED ─┬─▶ PROTOCOL_ACCEPTED               │
  (terminal;      │  (transient) ├─▶ PROTOCOL_REJECTED               │
   fail-closed)   │              └─▶ OUTCOME_UNKNOWN                 │
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

- Upon acceptance, BASIS will have a normative execution-lifecycle vocabulary that preserves every distinction the authorization side of the architecture has always required, extended honestly to execution. This proposal defines that vocabulary; it is not authoritative before a separate, dedicated acceptance review changes this ADR's status from `Proposed`.
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

- A compromised protocol-executor role can fabricate any attempt-outcome or state-verification value, because it is the sole authoritative observer (Decision 14) — the same residual-risk structure ADR-0011 Decision 19 and ADR-0012 already accept for a compromised same-process component, restated here rather than newly introduced.
- This ADR's vocabulary constrains what a conforming implementation may honestly report; it does not, by itself, verify that an implementation is conforming.

---

## Relationship to ADR-0011

This ADR proposes the Gate 2 execution-lifecycle specification exactly as ADR-0011 named it: a normative specification defining truthful execution-state semantics, preserving Decision 12's uncertainty requirement, and not assuming HTTP-style acknowledgement across every protocol (Decision 13). Upon acceptance, this ADR would resolve Gate 2 as ADR-0011 described it; until a separate, dedicated formal-acceptance review changes this ADR's status from `Proposed`, Gate 2 remains open, and this proposal does not authorize implementation. It does not reopen ADR-0011's protocol-executor role establishment, its same-process default-topology selection, or its governing invariants — it operates entirely within the space ADR-0011 left open for Gate 2, and it treats ADR-0011's Decision 9 (no dispatch before authoritative permission), Decision 11 (authorization and execution evidence remain separate), and Decision 12 (preserve uncertainty) as binding constraints this ADR must satisfy rather than revisit. This ADR's fail-closed requirements (Decision 16) are additive to, and do not weaken, ADR-0011 Decision 9's existing list.

## Relationship to ADR-0012

This ADR builds directly on ADR-0012's binding model: an execution attempt (Decision 1) is scoped to exactly one ADR-0012 binding record, the pre-dispatch attempt record (Decision 2) is created after binding verification succeeds (Decision 15's ordering), and a retry attempt (Decision 9) is only possible because ADR-0012's single-consumption guarantee forces a new binding cycle rather than reuse. This ADR narrows the disposition of ADR-0012's open replay/freshness question (Decision 10, above): a portion — attempt multiplicity and its binding-record consequence — is resolved here; the rest is explicitly not resolved here, and this ADR does not claim to be the "dedicated future architecture decision" ADR-0012 anticipated for the remainder. This ADR does not reopen ADR-0012's binding-model decision, its trust assumptions, or its failure semantics; it treats ADR-0012's Failure Semantics section as the authoritative source for pre-dispatch fail-closed conditions and restates it by reference (Decision 16) rather than by re-derivation.

## Relationship to Gate 3

This ADR is a prerequisite for Gate 3, not an overlap with it. It fixes the vocabulary — the attempt-outcome and resulting-state-verification axes, their values, and their transition rules — that a future execution-evidence specification must serialize; it does not itself define that specification. Specifically left to Gate 3, and explicitly not decided here: which logical role constructs execution evidence and which retains it, and whether they are the same component (Decision 14 decides only observation ownership, not construction/retention ownership); the evidence-durability axis's values, field names, and ownership (Decision 6); the exact reasons distinguishable within `OUTCOME_UNKNOWN` for a given protocol, as governed fields (Decision 4's note); minimum correlation-identifier requirements for a future execution-evidence record, beyond the prior-attempt correlation this ADR already requires (Decision 9); whether the existing six-value `ProvenanceClassification` enum suffices for protocol-dispatch-result and resulting-state-confirmation facts, or a new value is required; and persistence/failure ordering for the evidence-construction step itself, beyond the requirement that it never gate or retroactively alter an already-reached attempt-outcome or verification value (Decision 15). Gate 3 remains outstanding after this ADR, exactly as ADR-0011 named it, and this ADR's proposal — let alone its eventual acceptance — does not open it.

---

## Non-Goals

This ADR does not: implement execution; add protocol dispatch code for REST, BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, or Niagara; modify `basis-producer`, `basis-gateway`, `basis-adapters`, `basis-core`, `basis-identity`, or `basis-schemas`; create `basis-executor` or any other new repository; define an execution-evidence schema or any `basis-schemas` contract; select field names, enum value identifiers, or a serialization format for any axis defined here; define who constructs or retains execution evidence (Gate 3); resolve broader replay/freshness beyond the narrow attempt-correlation consequence stated in Decision 9 and Decision 10; define device/protocol credential custody (Gate 4); create an implementation plan, a PR sequence, or a milestone count; imply a `basis-producer` "Phase 6"; change ADR-0011's or ADR-0012's status; or mark itself `Accepted`. Any class, module, function, or API name that appears anywhere in this document is a non-normative illustration of a concept, never a specification of an interface.

---

## Validation / Implementation Gate

This ADR's merging does not itself constitute acceptance, consistent with this repository's established convention (see ADR-0011's and ADR-0012's own Validation / Implementation Gate sections and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself change its status to `Accepted`. Formal acceptance of this ADR, when and if it occurs through a separate, dedicated architecture and governance review, would establish the execution-lifecycle vocabulary defined here as the governed semantics a future bounded execution implementation must satisfy — it would not, by itself, authorize that implementation. Per ADR-0011's own Follow-On Decision Gates and Validation / Implementation Gate sections, Gates 1 through 3 are unconditional prerequisites, and Gate 4 is conditional; even full acceptance of this ADR resolves only Gate 2. Gate 3 (minimum execution-evidence semantics) remains outstanding regardless, and Gate 4 (device/protocol credential custody) remains conditionally outstanding if the eventually selected bounded target requires a protocol/device credential. No implementation is authorized by this ADR's proposal, and none would be authorized by its acceptance alone.

## References

- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — establishes the protocol-executor role, the same-process first-reference topology, Decision 12's uncertainty-preservation requirement, and Gate 2 as the unconditional prerequisite this ADR proposes to resolve upon acceptance
- [ADR-0012](0012-authorization-to-execution-binding.md) — resolves Gate 1's same-process binding mechanism and narrows Gate 1's replay/freshness requirement to process-local single consumption; leaves open whether a Gate 2 specification would resolve the remainder, which this ADR partially and explicitly does (Decision 10)
- [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) §8 (execution lifecycle and failure semantics — the primary evidentiary source for this ADR's vocabulary), §9 (protocol-specific execution semantics — the per-protocol evidence this ADR's neutrality requirement is verified against), §7 (replay and freshness), §14 (side-effect ordering — the crash-consistency analysis behind Decision 2), §19 (decision inventory naming the execution-lifecycle vocabulary as required before implementation)
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §5 (authorization-to-execution lifecycle invariants this ADR builds on without revising), §8 (the five preserved failure/degraded-condition distinctions this ADR extends)
- [ADR-0007](0007-adapter-evidence-construction.md) — the construction/retention separation precedent this ADR's Decision 14 reasons from without extending to execution evidence itself
- [`docs/architecture/adapter-evidence-construction-semantics.md`](../architecture/adapter-evidence-construction-semantics.md) — the "implementation proves a stable shape" schema-publication discipline this ADR's deferral to Gate 3 follows
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — referenced by ADR-0012 for the same publication discipline
- [`docs/glossary.md`](../glossary.md) — terminology this ADR's vocabulary is reconciled against
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR proposal and acceptance process this ADR follows
- [`docs/adr/README.md`](README.md) — ADR lifecycle states and required sections
