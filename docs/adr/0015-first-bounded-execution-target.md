# ADR-0015: First Bounded Execution Target

## Status

Accepted

## Context

[ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — Accepted — established a distinct logical protocol-executor role, architecturally separate from the operation-producer role, and selected same-process colocation with the existing `basis-producer` reference implementation as the default topology for the first bounded execution reference slice. Its own **First Bounded Reference Scope** section named REST as the intended first protocol, for the same "REST first" reasoning [ADR-0008](0008-producer-workload-authentication-and-admission.md) already used successfully for the authorization side, and stated directly what it deliberately did not do: *"This ADR does not define the REST execution implementation. It does not select a specific HTTP library. It does not select a target service. It does not define request/response mappings. It does not authorize contacting real production endpoints... The first bounded implementation should eventually demonstrate the execution architecture with the smallest realistic target needed to validate the architectural guarantees; this ADR does not design that target."* This ADR records that target selection. It is narrower than ADR-0011 itself: it names one target, not an execution architecture.

[ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) also named a fourth, conditional follow-on gate: **Gate 4 — Credential Custody, If Required by the First Slice**, stating plainly that "this gate does not apply if the first synthetic REST execution target can avoid endpoint credentials entirely." Whether Gate 4 applies at all has therefore been an open, target-dependent question since ADR-0011 was accepted — a question only a target *selection* can settle, because Gate 4's applicability is a property of the target, not of the execution architecture in the abstract.

[ADR-0012](0012-authorization-to-execution-binding.md) — Accepted — resolved Gate 1 for the same-process first-reference topology: a same-process authorization-to-execution binding record, created by the operation-producer role and verified by the protocol-executor role immediately before dispatch, with Gate 1's broader replay/freshness requirement narrowed to a process-local single-consumption guarantee and left open beyond that. [ADR-0013](0013-execution-lifecycle-semantics.md) — Accepted — resolves the Gate 2 execution-lifecycle vocabulary: three independently tracked facts (attempt outcome, resulting-state verification, evidence durability) rather than one flat status enum. [ADR-0014](0014-minimum-execution-evidence-semantics.md) — Accepted — resolves the Gate 3 minimum execution-evidence semantics: construction assigned to a distinct execution-evidence-producer role, retention colocated with `basis-producer` for the first bounded slice, and no schema publication authorized. ADR-0013 and ADR-0014 were accepted together with this ADR and ADR-0016, as one coupled execution-readiness milestone.

A milestone review of ADR-0013 through ADR-0015 together, conducted after this ADR was first proposed, surfaced a further, narrower ambiguity this ADR was not itself scoped to resolve: whether the target proposed here depends on the broader authorization replay/freshness properties ADR-0012 leaves open, or whether this target's own constraints mean that question does not arise for it specifically. [ADR-0016](0016-bounded-target-replay-freshness-posture.md) — Accepted — closes that gap for this target specifically, without resolving broader replay/freshness for any other topology and without resolving the separate operational duplicate-execution/idempotency question; see **Replay/Freshness Boundary**, below. This ADR does not itself resolve that ambiguity — ADR-0016 does.

None of ADR-0011 through ADR-0014 names a concrete target: what host the executor dispatches against, whether that host requires a credential, or what single operation shape the first slice should exercise. That is the specific, narrow gap this ADR closes — a target, not an implementation, and not a resolution of any of Gates 1 through 3.

Consistent with this repository's ADR-acceptance governance convention (ADR-0007 through ADR-0014 were each first merged with `Status: Proposed`, and a separate, dedicated follow-up PR later changed the status to `Accepted` for those that have since been accepted — see [`docs/adr/README.md`](README.md#lifecycle-states)), this ADR was submitted as `Proposed` and has since undergone that same separate, dedicated formal-acceptance review, together with ADR-0013, ADR-0014, and ADR-0016 as one coupled execution-readiness milestone; its status now records `Accepted`. Acceptance establishes the target selected here as the governed first bounded execution target — it does not itself authorize implementation; see **Validation / Implementation Gate**, below.

The current implemented state remains exactly what ADR-0011 through ADR-0014 all describe: `basis-producer`'s completed bounded authorization slice (Phase 2A through Phase 5, merged) stops at the authoritative disposition; `basis_producer.rest_composition`'s own module docstring states, as a scope boundary, that the module will not "execute any protocol operation under any disposition, including `ALLOW`." No protocol operation is executed anywhere in the ecosystem today, and this ADR's acceptance does not change that.

## Problem

Given ADR-0011's same-process default topology, ADR-0012's accepted same-process binding mechanism, and ADR-0013's and ADR-0014's Accepted lifecycle and evidence vocabularies, what specific first bounded execution target should a future reference implementation dispatch against? The answer must be concrete enough to guide implementation planning now that Gates 1 through 3 are resolved (and Gate 4, if applicable), without itself constituting that implementation planning, and it must state, for the target it names, whether Gate 4 applies. It must also state why the target is appropriate for a first slice specifically — as distinct from a mature or general-purpose execution target — and why real OT protocols, physical devices, and simulators requiring device credentials are not needed to validate the execution architecture this target exists to exercise.

## Decision

**ADR-0015 selects, as the first bounded execution target for the eventual reference implementation ADR-0011 anticipates: a same-process REST reference executor, hosted within `basis-producer`'s existing process boundary per ADR-0011 Decision 3, dispatching against a deterministic local test service or equivalent controlled reference endpoint, requiring no protocol/device credential, scoped to one tightly bounded REST operation shape, required to demonstrate explicit accept/reject/timeout-shaped outcomes, permitted but not required to perform a safe read-back state verification, and required to retain whatever execution evidence it produces internally — for implementation learning — rather than through a published `basis-schemas` contract.** This ADR selects a target. It does not design, plan, schedule, or authorize the implementation that would dispatch against it, and it does not itself resolve Gates 1 through 3.

### Selected Target

- **Topology.** Same-process, colocated with `basis-producer`, exactly as ADR-0011 Decision 3 already selected as the default topology for the first bounded reference slice. This ADR does not create `basis-executor` or any other new repository, and does not reopen the topology decision.
- **Protocol.** REST — the protocol ADR-0011's own "First Bounded Reference Scope" section already named as the intended first target, and the protocol the completed producer-authorization slice already uses end to end.
- **Endpoint.** A deterministic local test service, or an equivalent controlled reference endpoint under BASIS's own control — never a real production endpoint, a physical device, a PLC, a gateway appliance, or a cloud service. The endpoint's defining property is that its behavior is reproducible and does not depend on external infrastructure BASIS does not control.
- **Credential posture.** No protocol/device credential of any kind. The endpoint must be reachable without a bearer token, an API key, an mTLS client certificate, or any other endpoint-specific secret. This is the fact **Gate 4 Assessment**, below, turns on, and it is a requirement of the target, not an incidental property of whichever test service a future implementation happens to pick.
- **Operation shape.** Exactly one tightly bounded REST operation shape. This ADR does not name a specific HTTP verb, path, or payload schema — that is implementation-planning detail left to whatever future implementation plan actually builds against this target. It requires only that the shape be singular and narrow enough to validate the execution architecture without incidentally validating a general-purpose REST client or a multi-operation surface.

### Execution Outcomes That Must Be Demonstrable

A reference implementation built against this target must be capable of demonstrating, at minimum, outcomes corresponding to the following — stated here using [ADR-0013](0013-execution-lifecycle-semantics.md)'s Accepted vocabulary, normative because ADR-0013 itself is Accepted, not because this ADR makes it so:

| Demonstrable outcome | ADR-0013's vocabulary (Accepted) |
| - | - |
| A dispatch that never occurs because a pre-dispatch fail-closed condition holds (per ADR-0011 Decision 9 and ADR-0012's Failure Semantics) | `NOT_ATTEMPTED` |
| A dispatch the target endpoint positively accepts | `PROTOCOL_ACCEPTED` |
| A dispatch the target endpoint explicitly rejects | `PROTOCOL_REJECTED` |
| A dispatch for which no definitive answer arrives within the executor's observation window | `OUTCOME_UNKNOWN` |
| A safe, optional read-back confirming resulting state, where the target endpoint supports one | `STATE_VERIFIED` (else `STATE_NOT_VERIFIED` or `VERIFICATION_NOT_APPLICABLE`) |

This ADR borrows this vocabulary as a working target for what the selected endpoint and operation shape must be capable of exercising; it does not redefine it and does not make it normative in place of ADR-0013 — ADR-0013 itself, now Accepted, is normative. This ADR does not resolve Gate 2; Gate 2 is resolved by Accepted ADR-0013. ADR-0013's vocabulary did not change materially between this ADR's original proposal and the coupled acceptance review, so this table did not require reconciliation — the same outcome ADR-0014 records for the same reason.

## Target Selection Criteria

This target is appropriate for the first bounded execution slice, specifically, for the following reasons:

- **Minimizes unrelated protocol complexity.** REST is already the protocol the completed producer-authorization slice uses end to end, and ADR-0011's own reasoning for selecting REST first (avoiding unrelated protocol complexity) applies with equal force to selecting the execution-side target.
- **Avoids Gate 4 by construction, not by argument.** A credential-free endpoint removes the conditional gate entirely for this target, rather than requiring a case-by-case justification for why a given credential's custody is acceptable.
- **Deterministic and reproducible.** A local test service or controlled reference endpoint behaves predictably under CI and architecture-validation conditions, which a real production endpoint or physical device cannot guarantee.
- **Isolates execution-architecture concerns from protocol-specific complexity.** Validating binding verification (ADR-0012), lifecycle observation (Accepted ADR-0013's vocabulary), and evidence construction (Accepted ADR-0014's vocabulary) does not require a specific real-world OT protocol's semantics; REST's synchronous request/response shape is sufficient to exercise all three without simultaneously exercising multi-stage select/operate sequencing, fire-and-forget acknowledgement absence, or bus-level telegram semantics.
- **No safety exposure.** A controlled reference endpoint carries no risk of an unintended physical-world side effect, which a real OT protocol target aimed at even a bench-scale physical device would.

**Real OT protocols are explicitly out of scope for this first slice** — Modbus, BACnet, OPC UA, MQTT, DNP3, IEC 61850, KNX, and Niagara are not selected by this ADR, for reasons distinct from, and additional to, the reasons above:

- A real OT protocol target may introduce protocol-specific security profiles, deployment assumptions, simulator or device behavior, protocol/device credentials, or physical-world consequences this first slice does not need to take on. Selecting a deliberately credential-free target instead guarantees none of those concerns is a dependency of the first slice, and avoids Gate 4 by the target's own construction (**Gate 4 Assessment**, below) rather than by arguing that every OT protocol necessarily or typically requires a credential — Gate 4 remains conditional, and would still apply to any future target, OT or otherwise, that actually requires a protocol/device credential.
- DNP3 and IEC 61850's select/operate model is exactly the multi-stage complexity ADR-0013 Decision 7 exists to handle; validating that the simplest end-to-end execution path works at all does not require starting with the hardest protocol case, and this ADR deliberately keeps those two questions — "does the architecture work" and "does the architecture handle the hardest protocol" — separate rather than conflating them.
- MQTT and KNX's fire-and-forget, no-acknowledgement semantics are a meaningfully different case from REST's synchronous acknowledgement shape; exercising them well depends on ADR-0013's `OUTCOME_UNKNOWN` no-ack-primitive handling, which was not yet accepted architecture at the time this target was selected.
- None of the nine protocols' own adapters claim readiness for live protocol communication — every one of `basis-adapters`' module docstrings states the opposite — and this ADR does not create a reason to reconsider that.

## Gate 4 Assessment

**ADR-0015 selects a credential-free target.** ADR-0011 states Gate 4's applicability directly: "Gate 4 — Credential Custody, If Required by the First Slice... This gate does not apply if the first synthetic REST execution target can avoid endpoint credentials entirely." The target selected here — a deterministic local test service or equivalent controlled reference endpoint, reachable without any protocol/device credential — is precisely the credential-free case ADR-0011 anticipated.

**Because ADR-0015 is accepted, Gate 4 is not triggered for that first bounded target.** Acceptance is what satisfies ADR-0011's own credential-free condition for the first bounded slice. This section states the assessment the formal-acceptance review confirmed.

This assessment is scoped to the target selected here. It does not resolve Gate 4 in general, and it does not foreclose a future target — a real OT protocol target selected by later architecture — from triggering Gate 4 when that later selection is made. **Gate 4 remains conditional for any future target that requires protocol/device credentials**, exactly as ADR-0011 already states; this ADR establishes only that, for the target named here, that condition is not met.

## Relationship to ADR-0011

This ADR exercises ADR-0011 Decision 3 (same-process default topology) directly, by naming a target hosted within that topology, and answers the specific question ADR-0011's own "First Bounded Reference Scope" section declined to answer — naming a target that section said a future implementation "should eventually demonstrate the execution architecture with." It does not reopen ADR-0011's protocol-executor role establishment, its topology selection, or any of its governing invariants (Decisions 7 through 19), and it does not itself satisfy Gates 1 through 3 — those are governed by ADR-0012 (Accepted), ADR-0013 (Accepted), and ADR-0014 (Accepted), respectively. This ADR resolves Gate 4's applicability for the target selected here, per **Gate 4 Assessment**, above.

## Relationship to ADR-0012

The selected target is exactly the kind of target ADR-0012's same-process binding record was designed for: binding-record creation (operation-producer role) and verification (protocol-executor role) both occur within the same `basis-producer` process boundary this ADR's target colocates with, and neither the digest composition nor the process-local single-consumption freshness narrowing ADR-0012 defines changes for this target. This ADR does not modify ADR-0012's binding model or failure semantics. A future implementation against this target would still need to satisfy every one of ADR-0012's fail-closed conditions before any dispatch.

## Relationship to ADR-0013

This ADR borrows, without redefining, ADR-0013's Accepted three-axis vocabulary (attempt outcome, resulting-state verification, evidence durability) as the working shape a future implementation against this target should aim to demonstrate — see **Execution Outcomes That Must Be Demonstrable**, above. ADR-0013 is Accepted, having been accepted together with this ADR as part of the same coupled execution-readiness milestone; this ADR's reliance on its vocabulary does not itself constitute the "dedicated future architecture decision" ADR-0012 and ADR-0013 both reserve for broader replay/freshness properties. ADR-0013's vocabulary did not change materially between this ADR's proposal and the coupled acceptance review, so this ADR's Execution Outcomes table did not require reconciliation.

## Relationship to ADR-0014

This ADR borrows, without redefining, ADR-0014's Accepted minimum-evidence semantics — construction assigned to a distinct execution-evidence-producer role, retention colocated with `basis-producer` for the first bounded slice — as the shape execution evidence produced against this target should follow. ADR-0014 is Accepted, having been accepted together with this ADR as part of the same coupled execution-readiness milestone; this ADR does not itself resolve Gate 3, which is resolved by Accepted ADR-0014 jointly with Accepted ADR-0013. Retaining execution evidence "internally, for implementation learning," as this ADR's Decision states, is exactly ADR-0014's own **Schema Publication Boundary**: no `basis-schemas` contract is authorized until a bounded implementation proves a stable shape, and this ADR does not accelerate that sequence or claim that a stable shape already exists.

## Relationship to ADR-0016

[ADR-0016](0016-bounded-target-replay-freshness-posture.md) — Accepted — evaluates exactly the target this ADR selects and no other; **it does not select that target** — this ADR does, in **Target Selection Criteria**, above — and ADR-0016 does not reopen or restate that selection. ADR-0016 concludes that this target does not depend on broader authorization replay/freshness properties before a bounded reference implementation may be planned against it, under the constraints ADR-0016's own "The Boundary of This Conclusion" section states.

**The dependency between the two ADRs ran one way, not both.** This ADR's target-selection rationale did not depend on ADR-0016: this ADR does not reopen, restate, or depend on ADR-0016's reasoning for its own target selection or **Gate 4 Assessment**, above, and the reasons stated in **Target Selection Criteria** stand regardless of ADR-0016. The dependency ran the other way for implementation-readiness purposes: ADR-0016's replay/freshness posture is a conclusion about *this specific target*, under this target's exact bounded constraints — not a conclusion about bounded execution targets in general — and a complete bounded implementation-readiness conclusion required the target ADR-0016 evaluates to itself also be governed, which required this ADR's own acceptance. Both ADRs were accepted together, at the same coupled milestone review — the intended path this asymmetry was designed to support. ADR-0016 does not resolve operational duplicate-execution/idempotency for this target, and neither does this ADR; that question remains open for whatever future implementation is eventually planned against this target.

## Schema Publication Boundary

**This ADR does not authorize any `basis-schemas` publication.** Consistent with `ecosystem-contract-inventory.md`'s governing "implementation proves a stable shape" principle, already adopted without weakening by ADR-0014, execution evidence a future implementation produces against the target selected here is retained internally — colocated with `basis-producer`, per Accepted ADR-0014's retention model — for implementation learning. It does not obligate, schedule, or bring forward a future schema-publication step, and this ADR does not shorten the sequence discovery-assessment §20 already recommends (discovery assessment → execution-model ADR → normative specifications → a bounded reference implementation → conformance, demonstration, and review → shared schema publication).

## Replay/Freshness Boundary

Broader replay/freshness properties — wall-clock expiry, restart-surviving replay prevention, policy-reload invalidation, retry/idempotency behavior, and any replay database — remain exactly as open, globally, as ADR-0012 leaves them and as Accepted ADR-0013 narrows them for attempt correlation specifically. **This ADR's target selection does not itself narrow, resolve, or presuppose an answer to any of them.** The narrow attempt-correlation resolution Accepted ADR-0013 provides applies unchanged to any implementation built against this target.

[ADR-0016](0016-bounded-target-replay-freshness-posture.md) — Accepted — answers a narrower, related question this ADR did not originally address: whether the target proposed here depends on those broader, still-open *authorization* replay/freshness properties before a bounded reference implementation may be planned against it. ADR-0016 concludes that, under this target's exact bounded constraints (same-process; no persisted or transferable binding artifact; no protocol/device credential; no architecture-defined decision-to-dispatch window), broader authorization replay/freshness is not a prerequisite for planning a bounded reference implementation against it. ADR-0016 does not, by the same token, resolve operational duplicate-execution/idempotency — whether a freshly and validly authorized retry can still duplicate an unconfirmed prior attempt's effect — which remains open regardless, for this target and every other. This ADR does not restate ADR-0016's reasoning here; it points to it rather than duplicating it (**Relationship to ADR-0016**, above, states the full, one-directional dependency: this ADR's target selection does not depend on ADR-0016, but ADR-0016's implementation-readiness value depends on this ADR's own acceptance). This ADR's own status is unaffected by ADR-0016's status either way.

**This ADR itself establishes the one-dispatch, no-automatic-retry constraint, as part of the selected target's bounded scope — it is not a constraint ADR-0016 imposes:** a reference implementation built against this target is expected to perform at most one dispatch attempt per authorization-to-execution cycle — authorize → bind → dispatch → observe → record lifecycle/evidence → stop — and must not include automatic retry, automatic redispatch, or caller-transparent retry orchestration deciding on its own whether an uncertain prior attempt should be dispatched again. **Because this ADR is accepted, that constraint is part of the governed target it defines.** ADR-0016 relies on, and reinforces, this constraint for its own operational-idempotency non-blocking reasoning (its **Authorization Replay Versus Operational Idempotency** section) — it does not independently establish the constraint, and ADR-0016's acceptance does not make the constraint authoritative in place of this ADR's own acceptance. Neither this ADR nor ADR-0016 designs retry behavior beyond that constraint; both treat the absence of automatic retry as a bounding constraint on the first slice, not as a policy either ADR itself defines. Operational duplicate-execution/idempotency remains open regardless: the absence of automatic retry, automatic redispatch, and caller-transparent retry orchestration does not by itself resolve whether a freshly and validly authorized retry can duplicate an unconfirmed prior attempt's effect.

## Alternatives Considered

**A — Select a real building-automation protocol (BACnet or Modbus) target now.** Rejected. Selecting either introduces protocol-specific semantics this first slice does not need in order to validate the execution architecture's mechanics; it introduces simulator, device, and deployment assumptions — which simulator, which device profile, which deployment topology — that this ADR would then have to justify in place of the architecture itself; depending on the specific target eventually selected, it may also introduce security-profile or credential concerns this first slice has no architectural need to take on; and all of this is unnecessary complexity for proving the first bounded execution architecture specifically, as distinct from a mature or general-purpose execution target (**Target Selection Criteria**, above). The deterministic, credential-free REST target this ADR selects instead avoids these concerns by construction, not by an argument about what BACnet or Modbus specifically require in their typical deployment shape.

**B — Select a select/operate protocol (DNP3 or IEC 61850) as the first target, to stress-test ADR-0013's multi-stage model early.** Rejected. Multi-stage select/operate handling is exactly the highest-complexity case ADR-0013 Decision 7 was designed for; validating that the simplest end-to-end execution path (binding verification, dispatch, observation, evidence) works at all does not require starting with the hardest protocol case, and doing so would conflate two questions this ADR keeps separate — whether the execution architecture works, and whether it handles the hardest protocol.

**C — Select a real, production REST endpoint (a live third-party API) as the target.** Rejected. A production third-party endpoint would likely introduce credential or security-profile concerns most such APIs require in practice, working against the Gate-4-avoidance goal even though this ADR does not assert that every conceivable production endpoint necessarily requires one; it introduces external trust and deployment assumptions — whose API, whose account, whose rate limits and terms of service — a controlled architecture-validation slice should not depend on; it would introduce non-deterministic external state such a slice should not depend on either; and it risks an unintended side effect against a real system before the authorization-to-execution binding is implementation-proven. The selected credential-free, deterministic local target avoids all of these concerns by construction, not by an argument about what a production endpoint would specifically require.

**D — Select a separated-process executor target now, ahead of ADR-0011's own future-work framing.** Rejected. ADR-0011 Decision 3 explicitly selects same-process colocation as the default topology for the first bounded reference slice; a separated-process target would introduce a new authenticated trust boundary and its own follow-on architecture that ADR-0011 Decision 5 already defers to later work this ADR does not reopen.

**E — Defer target selection entirely until Gates 2 and 3 are accepted.** Considered, not selected. Naming a credential-free, protocol-neutral-compatible REST target does not require Gates 2 or 3 to already be accepted — this ADR only names a target a future implementation would eventually dispatch against, it does not authorize dispatch, and naming the target now gives Gate 2 and Gate 3 review a concrete, bounded scope to reason against rather than an abstract one.

---

## Consequences

### Positive

- This ADR names a concrete, credential-free first bounded execution target, closing the specific gap ADR-0011's "First Bounded Reference Scope" section left open.
- This ADR resolves Gate 4's applicability for the first slice: it is not triggered.
- This ADR gave Gate 2 (ADR-0013) and Gate 3 (ADR-0014) review a concrete target to reason against without requiring either to reach acceptance first.
- This ADR keeps real OT protocol integration, physical devices, and schema publication explicitly out of scope for the first slice, consistent with every other accepted document's restraint on implementation specifics.

### Negative / Tradeoff

- A REST-only, credential-free, local-test-service target validates the execution architecture's mechanics but does not validate protocol-specific complexity — multi-stage select/operate, no-acknowledgement protocols, device-credential handling — that a later target will eventually need to address.
- Selecting a target ahead of Gate 2 and Gate 3 acceptance meant a material change to ADR-0013's or ADR-0014's vocabulary before the coupled acceptance review could have required this ADR's Execution Outcomes table to be reconciled; in the event, no such change occurred (see **Relationship to ADR-0013**, above).

---

## Security Consequences

### Prevented by the target this ADR selects

- Introduction of a protocol/device credential into the first bounded execution slice.
- Dispatch against a real production endpoint, a physical device, or a PLC/gateway appliance before the execution architecture is implementation-proven.
- Premature commitment to a specific real OT protocol's semantics ahead of ADR-0013's protocol-neutral vocabulary, which has since reached acceptance.

### Still requiring follow-on architecture

- Gate 1's broader replay/freshness properties (Gate 1's same-process mechanism is resolved by Accepted ADR-0012; the broader properties remain open globally, though Accepted ADR-0016 establishes they are not a dependency of this target).
- A future target for real OT protocol validation, once the first bounded slice proves the architecture, remains a separate, later decision this ADR does not make.
- Gate 4 still applies to any future target that does require a protocol/device credential — this ADR resolves Gate 4's applicability only for the target selected here.

### Residual risks

- None new beyond those ADR-0011 Decision 19, ADR-0012, ADR-0013, and ADR-0014 already state for a same-process executor generally; this ADR does not change the same-process compromise/blast-radius analysis those documents already accept.

---

## What This ADR Deliberately Leaves Open

This ADR does not decide: the exact HTTP verb, path, payload shape, or fixture-service technology for the deterministic local test service; whether a future reference implementation's optional read-back verification is exercised in the first slice or deferred; an implementation plan, PR sequence, class/module/API shape, or milestone count for building against this target; Gate 1's broader replay/freshness properties for any topology other than the one this ADR selects; whether or when a future target for real OT protocol validation will be selected; or any `basis-schemas` contract for execution evidence.

---

## Non-Goals

This ADR does not: implement execution; add protocol dispatch code for REST or any other protocol; select a specific HTTP library, fixture-service technology, path, verb, or payload schema; create `basis-executor` or any other new repository; modify `basis-producer`, `basis-gateway`, `basis-adapters`, `basis-core`, `basis-identity`, or `basis-schemas`; authorize `basis-schemas` publication; select a real OT protocol, a physical device, a simulator requiring device credentials, a cloud service, or a production endpoint as the target; resolve Gate 1's broader replay/freshness properties, Gate 2, or Gate 3; change ADR-0011's, ADR-0012's, ADR-0013's, or ADR-0014's status; mark itself `Accepted`; create an implementation plan, a PR sequence, or a milestone count; imply a `basis-producer` "Phase 6"; or authorize protocol execution of any kind. Any class, module, function, or API name that might appear in a future implementation against this target is not specified by this ADR.

---

## Validation / Implementation Gate

This ADR's merging did not itself constitute acceptance, consistent with this repository's established convention (see ADR-0011's, ADR-0012's, ADR-0013's, and ADR-0014's own Validation / Implementation Gate sections and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself change its status to `Accepted`. After this ADR merged, a separate, dedicated formal architecture and governance review accepted it together with ADR-0013, ADR-0014, and ADR-0016, as one coupled execution-readiness milestone; this ADR's status now records `Accepted`. Acceptance establishes the target selected here as the governed first bounded execution target a future implementation should build against; it does not, by itself, authorize that implementation. Per ADR-0011's own Follow-On Decision Gates and Validation / Implementation Gate sections, Gates 1 through 3 were unconditional prerequisites: Gate 1's same-process mechanism is resolved by Accepted ADR-0012, though its broader replay/freshness properties remain open globally; Gate 2 is resolved by Accepted ADR-0013; Gate 3 is resolved by Accepted ADR-0014, jointly with Accepted ADR-0013. Because this ADR is accepted, Gate 4 is not triggered by the target selected here, per **Gate 4 Assessment**, above. [ADR-0016](0016-bounded-target-replay-freshness-posture.md) — Accepted — establishes that this target does not depend on broader authorization replay/freshness properties either, per **Replay/Freshness Boundary**, above; it does not resolve operational duplicate-execution/idempotency, which remains open. No implementation is authorized by this ADR's acceptance. Protocol execution remains entirely unimplemented and unauthorized.

## References

- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — establishes the protocol-executor role, the same-process first-reference topology, and the "First Bounded Reference Scope" section this ADR directly answers; also the source of Gate 4's conditional framing this ADR resolves for the selected target
- [ADR-0012](0012-authorization-to-execution-binding.md) — Accepted; the same-process binding mechanism a future implementation against this target must satisfy before dispatch
- [ADR-0013](0013-execution-lifecycle-semantics.md) — Accepted; the lifecycle vocabulary this ADR's Execution Outcomes table borrows without redefining
- [ADR-0014](0014-minimum-execution-evidence-semantics.md) — Accepted; the minimum-evidence semantics and Schema Publication Boundary this ADR's internal-retention requirement follows
- [ADR-0016](0016-bounded-target-replay-freshness-posture.md) — Accepted; evaluates the target this ADR selects and concludes that it does not depend on broader authorization replay/freshness properties; referenced without duplication in **Replay/Freshness Boundary** and **Relationship to ADR-0016**, above
- [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) §9 (protocol-specific execution semantics — the evidentiary basis for excluding real OT protocols from this first target), §20 (ADR recommendation and schema-publication sequencing)
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §13 (Recommended Implementation Sequence — Stage 6, which this ADR does not itself advance beyond naming a target)
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) — the "REST first" precedent this ADR's protocol selection follows
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the "implementation proves a stable shape" schema-readiness principle this ADR's Schema Publication Boundary adopts without weakening
- [`docs/glossary.md`](../glossary.md) — terminology this ADR's vocabulary is reconciled against
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR proposal and acceptance process this ADR follows
- [`docs/adr/README.md`](README.md) — ADR lifecycle states and required sections
