# ADR-0015: First Bounded Execution Target

## Status

Proposed

## Context

[ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — Accepted — established a distinct logical protocol-executor role, architecturally separate from the operation-producer role, and selected same-process colocation with the existing `basis-producer` reference implementation as the default topology for the first bounded execution reference slice. Its own **First Bounded Reference Scope** section named REST as the intended first protocol, for the same "REST first" reasoning [ADR-0008](0008-producer-workload-authentication-and-admission.md) already used successfully for the authorization side, and stated directly what it deliberately did not do: *"This ADR does not define the REST execution implementation. It does not select a specific HTTP library. It does not select a target service. It does not define request/response mappings. It does not authorize contacting real production endpoints... The first bounded implementation should eventually demonstrate the execution architecture with the smallest realistic target needed to validate the architectural guarantees; this ADR does not design that target."* This ADR proposes that target selection. It is narrower than ADR-0011 itself: it names one target, not an execution architecture.

[ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) also named a fourth, conditional follow-on gate: **Gate 4 — Credential Custody, If Required by the First Slice**, stating plainly that "this gate does not apply if the first synthetic REST execution target can avoid endpoint credentials entirely." Whether Gate 4 applies at all has therefore been an open, target-dependent question since ADR-0011 was accepted — a question only a target *selection* can settle, because Gate 4's applicability is a property of the target, not of the execution architecture in the abstract.

[ADR-0012](0012-authorization-to-execution-binding.md) — Accepted — resolved Gate 1 for the same-process first-reference topology: a same-process authorization-to-execution binding record, created by the operation-producer role and verified by the protocol-executor role immediately before dispatch, with Gate 1's broader replay/freshness requirement narrowed to a process-local single-consumption guarantee and left open beyond that. [ADR-0013](0013-execution-lifecycle-semantics.md) — **Proposed, not yet accepted** — proposes the Gate 2 execution-lifecycle vocabulary: three independently tracked facts (attempt outcome, resulting-state verification, evidence durability) rather than one flat status enum. [ADR-0014](0014-minimum-execution-evidence-semantics.md) — **Proposed, not yet accepted** — proposes the Gate 3 minimum execution-evidence semantics: construction assigned to a distinct execution-evidence-producer role, retention colocated with `basis-producer` for the first bounded slice, and no schema publication authorized. Neither ADR-0013 nor ADR-0014 has been accepted, and nothing in this ADR changes that.

None of ADR-0011 through ADR-0014 names a concrete target: what host the executor dispatches against, whether that host requires a credential, or what single operation shape the first slice should exercise. That is the specific, narrow gap this ADR proposes to close upon acceptance — a target, not an implementation, and not a resolution of any of Gates 1 through 3.

Consistent with this repository's ADR-acceptance governance convention (ADR-0007 through ADR-0014 were each first merged with `Status: Proposed`, and a separate, dedicated follow-up PR later changed the status to `Accepted` for those that have since been accepted — see [`docs/adr/README.md`](README.md#lifecycle-states)), this ADR is submitted as `Proposed`. Nothing in this ADR, and nothing in its merging, constitutes acceptance. No target selection recorded here is authoritative until a separate, dedicated formal-acceptance review changes this ADR's status to `Accepted`, and acceptance itself would not authorize implementation — see **Validation / Implementation Gate**, below.

The current implemented state remains exactly what ADR-0011 through ADR-0014 all describe: `basis-producer`'s completed bounded authorization slice (Phase 2A through Phase 5, merged) stops at the authoritative disposition; `basis_producer.rest_composition`'s own module docstring states, as a scope boundary, that the module will not "execute any protocol operation under any disposition, including `ALLOW`." No protocol operation is executed anywhere in the ecosystem today, and this ADR's proposal — let alone its eventual acceptance — does not change that.

## Problem

Given ADR-0011's same-process default topology, ADR-0012's accepted same-process binding mechanism, and ADR-0013's and ADR-0014's proposed — not accepted — lifecycle and evidence vocabularies, what specific first bounded execution target should a future reference implementation dispatch against? The answer must be concrete enough to guide implementation planning once Gates 1 through 3 (and Gate 4, if applicable) are actually resolved, without itself constituting that implementation planning, and it must propose, for the target it names, whether Gate 4 would apply at all upon acceptance. It must also state why the target is appropriate for a first slice specifically — as distinct from a mature or general-purpose execution target — and why real OT protocols, physical devices, and simulators requiring device credentials are not needed to validate the execution architecture this target exists to exercise.

## Decision

**ADR-0015 proposes to select, as the first bounded execution target for the eventual reference implementation ADR-0011 anticipates: a same-process REST reference executor, hosted within `basis-producer`'s existing process boundary per ADR-0011 Decision 3, dispatching against a deterministic local test service or equivalent controlled reference endpoint, requiring no protocol/device credential, scoped to one tightly bounded REST operation shape, required to demonstrate explicit accept/reject/timeout-shaped outcomes, permitted but not required to perform a safe read-back state verification, and required to retain whatever execution evidence it produces internally — for implementation learning — rather than through a published `basis-schemas` contract.** This ADR proposes a target selection. It does not design, plan, schedule, or authorize the implementation that would dispatch against it, and it does not resolve Gates 1 through 3. This proposal is not authoritative until a separate, dedicated formal-acceptance review changes this ADR's status from `Proposed`.

### Selected Target

- **Topology.** Same-process, colocated with `basis-producer`, exactly as ADR-0011 Decision 3 already selected as the default topology for the first bounded reference slice. This ADR does not create `basis-executor` or any other new repository, and does not reopen the topology decision.
- **Protocol.** REST — the protocol ADR-0011's own "First Bounded Reference Scope" section already named as the intended first target, and the protocol the completed producer-authorization slice already uses end to end.
- **Endpoint.** A deterministic local test service, or an equivalent controlled reference endpoint under BASIS's own control — never a real production endpoint, a physical device, a PLC, a gateway appliance, or a cloud service. The endpoint's defining property is that its behavior is reproducible and does not depend on external infrastructure BASIS does not control.
- **Credential posture.** No protocol/device credential of any kind. The endpoint must be reachable without a bearer token, an API key, an mTLS client certificate, or any other endpoint-specific secret. This is the fact **Gate 4 Assessment**, below, turns on, and it is a requirement of the target, not an incidental property of whichever test service a future implementation happens to pick.
- **Operation shape.** Exactly one tightly bounded REST operation shape. This ADR does not name a specific HTTP verb, path, or payload schema — that is implementation-planning detail left to whatever future implementation plan actually builds against this target. It requires only that the shape be singular and narrow enough to validate the execution architecture without incidentally validating a general-purpose REST client or a multi-operation surface.

### Execution Outcomes That Must Be Demonstrable

A reference implementation built against this target must be capable of demonstrating, at minimum, outcomes corresponding to the following — stated here using [ADR-0013](0013-execution-lifecycle-semantics.md)'s proposed vocabulary as the working shape to build toward, not as vocabulary this ADR itself makes normative:

| Demonstrable outcome | ADR-0013's proposed vocabulary (Proposed, not yet accepted) |
| - | - |
| A dispatch that never occurs because a pre-dispatch fail-closed condition holds (per ADR-0011 Decision 9 and ADR-0012's Failure Semantics) | `NOT_ATTEMPTED` |
| A dispatch the target endpoint positively accepts | `PROTOCOL_ACCEPTED` |
| A dispatch the target endpoint explicitly rejects | `PROTOCOL_REJECTED` |
| A dispatch for which no definitive answer arrives within the executor's observation window | `OUTCOME_UNKNOWN` |
| A safe, optional read-back confirming resulting state, where the target endpoint supports one | `STATE_VERIFIED` (else `STATE_NOT_VERIFIED` or `VERIFICATION_NOT_APPLICABLE`) |

This ADR borrows this vocabulary as a working target for what the selected endpoint and operation shape must be capable of exercising; it does not redefine it, does not make it normative in place of ADR-0013, and does not resolve Gate 2. If ADR-0013's vocabulary changes materially before its own acceptance review, this table would need reconciliation before this ADR could itself be accepted — the same dependency ADR-0014 already accepted for the same reason.

## Target Selection Criteria

This target is appropriate for the first bounded execution slice, specifically, for the following reasons:

- **Minimizes unrelated protocol complexity.** REST is already the protocol the completed producer-authorization slice uses end to end, and ADR-0011's own reasoning for selecting REST first (avoiding unrelated protocol complexity) applies with equal force to selecting the execution-side target.
- **Avoids Gate 4 by construction, not by argument.** A credential-free endpoint removes the conditional gate entirely for this target, rather than requiring a case-by-case justification for why a given credential's custody is acceptable.
- **Deterministic and reproducible.** A local test service or controlled reference endpoint behaves predictably under CI and architecture-validation conditions, which a real production endpoint or physical device cannot guarantee.
- **Isolates execution-architecture concerns from protocol-specific complexity.** Validating binding verification (ADR-0012), lifecycle observation (ADR-0013's proposed vocabulary), and evidence construction (ADR-0014's proposed vocabulary) does not require a specific real-world OT protocol's semantics; REST's synchronous request/response shape is sufficient to exercise all three without simultaneously exercising multi-stage select/operate sequencing, fire-and-forget acknowledgement absence, or bus-level telegram semantics.
- **No safety exposure.** A controlled reference endpoint carries no risk of an unintended physical-world side effect, which a real OT protocol target aimed at even a bench-scale physical device would.

**Real OT protocols are explicitly out of scope for this first slice** — Modbus, BACnet, OPC UA, MQTT, DNP3, IEC 61850, KNX, and Niagara are not selected by this ADR, for reasons distinct from, and additional to, the reasons above:

- Each requires a protocol/device credential in its typical deployment shape, which would trigger Gate 4 and add a dependency this first slice does not need.
- DNP3 and IEC 61850's select/operate model is exactly the multi-stage complexity ADR-0013 Decision 7 exists to handle; validating that the simplest end-to-end execution path works at all does not require starting with the hardest protocol case, and this ADR deliberately keeps those two questions — "does the architecture work" and "does the architecture handle the hardest protocol" — separate rather than conflating them.
- MQTT and KNX's fire-and-forget, no-acknowledgement semantics are a meaningfully different case from REST's synchronous acknowledgement shape; exercising them well depends on ADR-0013's `OUTCOME_UNKNOWN` no-ack-primitive handling, which is not yet accepted architecture.
- None of the nine protocols' own adapters claim readiness for live protocol communication — every one of `basis-adapters`' module docstrings states the opposite — and this ADR does not create a reason to reconsider that.

## Gate 4 Assessment

**ADR-0015 proposes a credential-free target.** ADR-0011 states Gate 4's applicability directly: "Gate 4 — Credential Custody, If Required by the First Slice... This gate does not apply if the first synthetic REST execution target can avoid endpoint credentials entirely." The target proposed here — a deterministic local test service or equivalent controlled reference endpoint, reachable without any protocol/device credential — is precisely the credential-free case ADR-0011 anticipated.

**If ADR-0015 is accepted, Gate 4 would not be triggered for that first bounded target.** Acceptance is what would satisfy ADR-0011's own credential-free condition for the first bounded slice; proposal alone does not.

**Until ADR-0015 is accepted, Gate 4 applicability is proposed but not authoritatively settled.** This section states the assessment a formal-acceptance review would need to confirm, not a conclusion already reached on the architecture's behalf. Nothing in this ADR's proposal changes Gate 4's status as an open, conditional gate in the interim.

This assessment is scoped to the target proposed here. It does not resolve Gate 4 in general, and it does not foreclose a future target — a real OT protocol target selected by later architecture — from triggering Gate 4 when that later selection is made. **Gate 4 remains conditional for any future target that requires protocol/device credentials**, exactly as ADR-0011 already states; this ADR proposes only that, for the target named here, that condition is not met.

## Relationship to ADR-0011

This ADR exercises ADR-0011 Decision 3 (same-process default topology) directly, by naming a target hosted within that topology, and proposes to answer the specific question ADR-0011's own "First Bounded Reference Scope" section declined to answer — naming a target that section said a future implementation "should eventually demonstrate the execution architecture with." It does not reopen ADR-0011's protocol-executor role establishment, its topology selection, or any of its governing invariants (Decisions 7 through 19), and it does not itself satisfy Gates 1 through 3 — those remain governed by ADR-0012 (Accepted), ADR-0013 (Proposed, not yet accepted), and ADR-0014 (Proposed, not yet accepted), respectively. It would resolve Gate 4's applicability for the target proposed here, upon acceptance, per **Gate 4 Assessment**, above.

## Relationship to ADR-0012

The selected target is exactly the kind of target ADR-0012's same-process binding record was designed for: binding-record creation (operation-producer role) and verification (protocol-executor role) both occur within the same `basis-producer` process boundary this ADR's target colocates with, and neither the digest composition nor the process-local single-consumption freshness narrowing ADR-0012 defines changes for this target. This ADR does not modify ADR-0012's binding model or failure semantics. A future implementation against this target would still need to satisfy every one of ADR-0012's fail-closed conditions before any dispatch.

## Relationship to ADR-0013

This ADR borrows, without redefining, ADR-0013's proposed three-axis vocabulary (attempt outcome, resulting-state verification, evidence durability) as the working shape a future implementation against this target should aim to demonstrate — see **Execution Outcomes That Must Be Demonstrable**, above. ADR-0013 remains **Proposed, not yet accepted**, and this ADR's reliance on its vocabulary does not change that status, does not resolve Gate 2, and does not itself constitute the "dedicated future architecture decision" ADR-0012 and ADR-0013 both reserve for broader replay/freshness properties. If ADR-0013's vocabulary changes materially before its own acceptance review, this ADR's Execution Outcomes table would need reconciliation before this ADR could itself be accepted.

## Relationship to ADR-0014

This ADR borrows, without redefining, ADR-0014's proposed minimum-evidence semantics — construction assigned to a distinct execution-evidence-producer role, retention colocated with `basis-producer` for the first bounded slice — as the shape execution evidence produced against this target should follow. ADR-0014 remains **Proposed, not yet accepted**, and this ADR does not resolve Gate 3. Retaining execution evidence "internally, for implementation learning," as this ADR's Decision states, is exactly ADR-0014's own **Schema Publication Boundary**: no `basis-schemas` contract is authorized until a bounded implementation proves a stable shape, and this ADR does not accelerate that sequence or claim that a stable shape already exists.

## Schema Publication Boundary

**This ADR does not authorize any `basis-schemas` publication.** Consistent with `ecosystem-contract-inventory.md`'s governing "implementation proves a stable shape" principle, already adopted without weakening by ADR-0014, execution evidence a future implementation produces against the target selected here is retained internally — colocated with `basis-producer`, per ADR-0014's proposed retention model — for implementation learning. It does not obligate, schedule, or bring forward a future schema-publication step, and this ADR does not shorten the sequence discovery-assessment §20 already recommends (discovery assessment → execution-model ADR → normative specifications → a bounded reference implementation → conformance, demonstration, and review → shared schema publication).

## Replay/Freshness Boundary

Broader replay/freshness properties — wall-clock expiry, restart-surviving replay prevention, policy-reload invalidation, retry/idempotency behavior, and any replay database — remain exactly as open as ADR-0012 leaves them and as ADR-0013 (Proposed, not yet accepted) proposes to narrow them. **This ADR's target selection does not narrow, resolve, or presuppose an answer to any of them.** Whatever narrow attempt-correlation resolution ADR-0013 proposes applies unchanged, once ADR-0013 is itself accepted, to any implementation built against this target; this ADR does not accelerate or substitute for that acceptance.

## Alternatives Considered

**A — Select a real building-automation protocol (BACnet or Modbus) target now.** Rejected. Both require a protocol/device credential in their typical deployment shape, which would trigger Gate 4 and add a dependency this first slice does not need; either would also likely require a simulator, and a simulator requiring device credentials is explicitly excluded from what this ADR may select.

**B — Select a select/operate protocol (DNP3 or IEC 61850) as the first target, to stress-test ADR-0013's multi-stage model early.** Rejected. Multi-stage select/operate handling is exactly the highest-complexity case ADR-0013 Decision 7 was designed for; validating that the simplest end-to-end execution path (binding verification, dispatch, observation, evidence) works at all does not require starting with the hardest protocol case, and doing so would conflate two questions this ADR keeps separate — whether the execution architecture works, and whether it handles the hardest protocol.

**C — Select a real, production REST endpoint (a live third-party API) as the target.** Rejected. It would require credential custody, defeating the Gate-4-avoidance goal; it would introduce non-deterministic external state a controlled architecture-validation slice should not depend on; and it risks an unintended side effect against a real system before the authorization-to-execution binding is implementation-proven.

**D — Select a separated-process executor target now, ahead of ADR-0011's own future-work framing.** Rejected. ADR-0011 Decision 3 explicitly selects same-process colocation as the default topology for the first bounded reference slice; a separated-process target would introduce a new authenticated trust boundary and its own follow-on architecture that ADR-0011 Decision 5 already defers to later work this ADR does not reopen.

**E — Defer target selection entirely until Gates 2 and 3 are accepted.** Considered, not selected. Naming a credential-free, protocol-neutral-compatible REST target does not require Gates 2 or 3 to already be accepted — this ADR only names a target a future implementation would eventually dispatch against, it does not authorize dispatch, and naming the target now gives Gate 2 and Gate 3 review a concrete, bounded scope to reason against rather than an abstract one.

---

## Consequences

### Positive

- Upon acceptance, this ADR would name a concrete, credential-free first bounded execution target, closing the specific gap ADR-0011's "First Bounded Reference Scope" section left open.
- Upon acceptance, this ADR would resolve Gate 4's applicability for the first slice: it would not be triggered. Until acceptance, Gate 4 remains open like the other gates, and this proposal does not itself remove it from the first bounded implementation's critical path.
- This proposal gives Gate 2 (ADR-0013) and Gate 3 (ADR-0014) review a concrete target to reason against without requiring either to reach acceptance first.
- This proposal keeps real OT protocol integration, physical devices, and schema publication explicitly out of scope for the first slice, consistent with every other accepted document's restraint on implementation specifics.

### Negative / Tradeoff

- A REST-only, credential-free, local-test-service target validates the execution architecture's mechanics but does not validate protocol-specific complexity — multi-stage select/operate, no-acknowledgement protocols, device-credential handling — that a later target will eventually need to address.
- Selecting a target ahead of Gate 2 and Gate 3 acceptance means a future material change to ADR-0013's or ADR-0014's vocabulary could require this ADR's Execution Outcomes table to be reconciled before this ADR could itself be accepted.

---

## Security Consequences

### Prevented by the target this ADR proposes

- Introduction of a protocol/device credential into the first bounded execution slice, upon acceptance of this proposal.
- Dispatch against a real production endpoint, a physical device, or a PLC/gateway appliance before the execution architecture is implementation-proven.
- Premature commitment to a specific real OT protocol's semantics ahead of ADR-0013's protocol-neutral vocabulary reaching acceptance.

### Still requiring follow-on architecture

- Gate 1's broader replay/freshness properties (Gate 1's same-process mechanism is resolved by Accepted ADR-0012; the broader properties are not).
- Gate 2 (ADR-0013) and Gate 3 (ADR-0014) acceptance — both remain outstanding regardless of this ADR.
- A future target for real OT protocol validation, once the first bounded slice proves the architecture, remains a separate, later decision this ADR does not make.
- Gate 4 would still apply to any future target that does require a protocol/device credential — upon acceptance, this ADR would resolve Gate 4's applicability only for the target proposed here.

### Residual risks

- None new beyond those ADR-0011 Decision 19, ADR-0012, ADR-0013, and ADR-0014 already state for a same-process executor generally; this ADR does not change the same-process compromise/blast-radius analysis those documents already accept.

---

## What This ADR Deliberately Leaves Open

This ADR does not decide: the exact HTTP verb, path, payload shape, or fixture-service technology for the deterministic local test service; whether a future reference implementation's optional read-back verification is exercised in the first slice or deferred; an implementation plan, PR sequence, class/module/API shape, or milestone count for building against this target; Gate 1's broader replay/freshness properties; Gate 2's or Gate 3's acceptance, neither of which selecting a target resolves; whether or when a future target for real OT protocol validation will be selected; or any `basis-schemas` contract for execution evidence.

---

## Non-Goals

This ADR does not: implement execution; add protocol dispatch code for REST or any other protocol; select a specific HTTP library, fixture-service technology, path, verb, or payload schema; create `basis-executor` or any other new repository; modify `basis-producer`, `basis-gateway`, `basis-adapters`, `basis-core`, `basis-identity`, or `basis-schemas`; authorize `basis-schemas` publication; select a real OT protocol, a physical device, a simulator requiring device credentials, a cloud service, or a production endpoint as the target; resolve Gate 1's broader replay/freshness properties, Gate 2, or Gate 3; change ADR-0011's, ADR-0012's, ADR-0013's, or ADR-0014's status; mark itself `Accepted`; create an implementation plan, a PR sequence, or a milestone count; imply a `basis-producer` "Phase 6"; or authorize protocol execution of any kind. Any class, module, function, or API name that might appear in a future implementation against this target is not specified by this ADR.

---

## Validation / Implementation Gate

This ADR's merging does not itself constitute acceptance, consistent with this repository's established convention (see ADR-0011's, ADR-0012's, ADR-0013's, and ADR-0014's own Validation / Implementation Gate sections and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself change its status to `Accepted`. Formal acceptance of this ADR, when and if it occurs through a separate, dedicated architecture and governance review, would establish the target selected here as the governed first bounded execution target a future implementation should build against; it would not, by itself, authorize that implementation. Per ADR-0011's own Follow-On Decision Gates and Validation / Implementation Gate sections, Gates 1 through 3 remain unconditional prerequisites regardless of this ADR's status: Gate 1's same-process mechanism is resolved by Accepted ADR-0012, but its broader replay/freshness properties remain open; Gate 2 remains open until a separate, dedicated acceptance review changes ADR-0013's status from `Proposed`; Gate 3 remains open until a separate, dedicated acceptance review changes ADR-0014's status from `Proposed` (and, per ADR-0014's own dependency, not before ADR-0013 is also accepted). If this ADR is accepted, Gate 4 would not be triggered by the target selected here, per **Gate 4 Assessment**, above; until then, Gate 4's applicability remains proposed rather than settled. No implementation is authorized by this ADR's proposal, and none would be authorized by its acceptance alone. Protocol execution remains entirely unimplemented and unauthorized.

## References

- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — establishes the protocol-executor role, the same-process first-reference topology, and the "First Bounded Reference Scope" section this ADR proposes to directly answer; also the source of Gate 4's conditional framing this ADR proposes to resolve for the selected target upon acceptance
- [ADR-0012](0012-authorization-to-execution-binding.md) — Accepted; the same-process binding mechanism a future implementation against this target must satisfy before dispatch
- [ADR-0013](0013-execution-lifecycle-semantics.md) — **Proposed, not yet accepted**; the proposed lifecycle vocabulary this ADR's Execution Outcomes table borrows without redefining
- [ADR-0014](0014-minimum-execution-evidence-semantics.md) — **Proposed, not yet accepted**; the proposed minimum-evidence semantics and Schema Publication Boundary this ADR's internal-retention requirement follows
- [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) §9 (protocol-specific execution semantics — the evidentiary basis for excluding real OT protocols from this first target), §20 (ADR recommendation and schema-publication sequencing)
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §13 (Recommended Implementation Sequence — Stage 6, which this ADR does not itself advance beyond naming a target)
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) — the "REST first" precedent this ADR's protocol selection follows
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the "implementation proves a stable shape" schema-readiness principle this ADR's Schema Publication Boundary adopts without weakening
- [`docs/glossary.md`](../glossary.md) — terminology this ADR's vocabulary is reconciled against
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR proposal and acceptance process this ADR follows
- [`docs/adr/README.md`](README.md) — ADR lifecycle states and required sections
