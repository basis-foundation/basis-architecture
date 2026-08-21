# ADR-0011: Protocol Execution Role and Bounded Reference Topology

## Status

Proposed

## Context

The bounded operation-producer authorization reference slice is complete. [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) — Accepted — established `basis-producer` as the permanent operation-producer-runtime repository and component, and, per [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) §1's direct inspection of that repository at commit `b8271de` on `main`, all five phases of its bounded implementation are merged: Phase 2A evidence retention, Phase 2B reference lifecycle, Phase 3 the authenticated gateway client, Phase 4 REST adapter composition, and Phase 5 cross-repository conformance and demonstration. `basis-gateway`'s Phase 1B (1B.1–1B.3) is merged, implementing trusted-ingress mTLS producer authentication and admission alongside the retained legacy bearer-subject allowlist. That reference slice proves protocol normalization, deterministic adapter evidence construction, evidence retention, reference lifecycle, producer workload authentication, exact producer admission, independent bearer authorization-subject authentication, authenticated producer-to-gateway submission, gateway composition and enforcement, `basis-core` authorization evaluation, and deterministic `ALLOW`/`DENY` — and it also deliberately proves **no execution**: `basis_producer.rest_composition`'s own module docstring states, as a scope boundary rather than an omission, that the module will not "execute any protocol operation under any disposition, including `ALLOW`."

The current path deliberately ends at `authoritative authorization disposition → STOP`. [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §5 already states the invariant that "execution must never occur before the authoritative disposition is received," and ADR-0010 restates that its own repository-placement decision "does not expand, and does not authorize expanding, the bounded producer slice ADR-0008 already authorized" to include execution for any of the nine adapter protocol families. Every accepted document treats the authorization/execution split as a load-bearing architectural boundary, not an artifact of implementation sequencing. This ADR inherits that boundary; it does not reopen whether the split itself is correct.

The merged, non-normative [`execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) — the primary evidence source for this ADR — inventoried the current authorization-to-execution boundary, analyzed the logical roles hidden inside "protocol executor" and "execution-evidence producer," evaluated same-process and separate-process deployment topologies against the completed producer slice's own evidence, and identified the decisions required before BASIS implements protocol execution. Its findings, restated here only to the extent they bear directly on the decision this ADR makes: authorization and execution are distinct lifecycle stages, and that distinction is load-bearing, not incidental (§1, §4); a logical execution role exists regardless of process placement, and decomposes into at least four sub-responsibilities — original-operation preservation and authorized-operation binding, protocol dispatch, execution-result observation, and execution-evidence construction/retention — that are not all equally separable (§5); same-process execution is measurably better-precedented by existing accepted architecture (ADR-0010's own credential-concentration reasoning, `basis-producer`'s existing pipeline-composition pattern) than separate-process execution, which has no equivalent existing precedent to build from (§5, §16, §18); a compromised executor with direct device reach and sufficient protocol credentials can bypass BASIS policy entirely, and this is not fully closable by software alone under any topology the assessment evaluated (§11, §12); and same-process topology removes a cross-process transport hop and the authenticated-handoff problem that hop would otherwise reopen, which is a reduction in how much a binding mechanism must additionally solve for, not a demonstration that any particular existing artifact already solves it (§6). What, if anything, already in the chain is sufficient to serve as that binding is exactly the question this ADR leaves to Gate 1, below, and this ADR does not anticipate that gate's answer.

The assessment's own §20 concluded there is enough evidence to write an ADR selecting a default deployment topology and stating the governing invariants execution must preserve, but not yet enough evidence to select a specific binding mechanism, freshness model, or execution-evidence contract shape — each of those needs its own normative architecture specification following this ADR, and, only where a specific implementation question cannot responsibly be settled from architecture alone, a bounded technical spike. This ADR is that first decision: it establishes the protocol-executor role and its governing invariants; it does not resolve the binding mechanism, the execution-lifecycle vocabulary, or the execution-evidence contract, each of which this ADR explicitly defers to Gates 1 through 3 below — and, if the selected bounded target requires a protocol/device credential, a fourth, conditional gate (Gate 4).

Consistent with this repository's ADR-acceptance governance convention (ADR-0007, ADR-0008, ADR-0009, and ADR-0010 were each first merged with `Status: Proposed`, and a separate, dedicated follow-up PR later changed the status to `Accepted` after independent architecture and governance review — see [`docs/adr/README.md`](README.md#lifecycle-states)), this ADR is submitted as `Proposed`. Merging this ADR does not itself constitute acceptance, and no implementation work is authorized until a separate formal-acceptance PR records `Status: Accepted` — and, per **ADR Acceptance Boundary** below, acceptance itself would not authorize protocol execution.

## Problem

Does BASIS have a distinct logical role responsible for protocol execution, is that role architecturally separate from the operation-producer role even when both are hosted in the same process, and — for the first bounded execution reference slice specifically — what is the default deployment topology for that role? The answer must preserve, as separate concerns this ADR does not collapse into a generic "execution architecture": the logical-role question (does a protocol-executor role exist and what does it own); the topology question (where does the first reference implementation of that role run); the binding-mechanism question (how is a dispatch attempt proven to correspond to the exact operation that was authorized); the lifecycle-vocabulary question (what states can an execution attempt be in); and the evidence-contract question (what a durable execution-evidence record looks like). This ADR answers the first two. It establishes that the third, fourth, and fifth require their own normative architecture work before any bounded execution implementation may dispatch a protocol operation, and it does not do that work here.

## Decision

**ADR-0011 establishes a distinct logical protocol-executor role, architecturally separate from the operation-producer role, and selects a same-process logical executor colocated with the existing `basis-producer` reference implementation as the default topology for the first bounded execution reference slice.** It fixes the invariants execution must preserve regardless of topology. It does not implement execution, does not select a binding mechanism, does not define an execution-lifecycle vocabulary, does not define execution-evidence schemas, and does not create a new repository.

### 1. A Distinct Logical Protocol Executor Role

The **protocol executor** is a normative logical architectural role, narrowly defined. It is the logical responsibility that:

- receives or retains access to the original protocol-shaped `ProtocolOperation` that entered the chain at normalization;
- acts only after `basis-gateway` has produced an authoritative disposition permitting execution;
- verifies whatever authorization-to-execution binding a future binding architecture requires (Gate 1) before dispatch;
- owns protocol dispatch mechanics;
- communicates with the protocol endpoint or platform;
- observes whatever result the protocol makes available;
- never re-evaluates policy;
- never overrides the gateway disposition;
- never converts a `DENY`, a failed evaluation, or any other non-permitting disposition into an execution attempt.

This is a logical architectural role. This ADR does not yet define its Python interface, its process API, its network API, its schema, its package, its class hierarchy, or its protocol driver abstraction.

### 2. Protocol Executor and Operation Producer Remain Distinct Roles

**The operation producer and protocol executor are distinct logical responsibilities even when one process performs both.** The operation-producer role continues to own the authorization-side responsibilities ADR-0010 already established: adapter orchestration; authorization-side evidence retention; reference lifecycle; producer workload credential custody; independent authorization-subject credential presentation; authenticated gateway submission; and receipt and interpretation of the authoritative disposition. The protocol-executor role begins at the execution side of that boundary. `basis-producer` hosting both roles in its first reference topology (Decision 3) does not collapse them conceptually, because future deployments may separate them without changing the authorization semantics either role depends on.

### 3. Same-Process as the Default First-Reference Topology

**A same-process logical executor colocated with the existing `basis-producer` reference implementation is the default topology for the first bounded execution reference slice.** For that first bounded reference implementation:

```text
basis-producer
    contains the existing operation-producer role
    +
    hosts a logically distinct protocol-executor role
    within the same process/runtime boundary
```

Conceptually:

```text
ProtocolOperation
    → producer authorization pipeline
        → authoritative ALLOW
            → authorization-to-execution binding verification
                → same-process logical protocol executor
                    → protocol endpoint
```

This selection is specifically for the first bounded reference slice. Reasons supported by the discovery assessment (§5, §16, §18): it avoids introducing a second authenticated network trust boundary before one is shown to be necessary; it avoids requiring a portable execution handoff before the architecture has evidence one is needed; it minimizes initial replay/handoff complexity and operational components; it fits constrained and air-gapped environments better, consistent with ADR-0008's own preference for mechanisms that do not require external connectivity; it reduces the number of trust relationships and failure modes Gate 1's binding specification must address, by removing the cross-process transport hop and authenticated handoff a separated topology would otherwise require (§6); it builds naturally on `basis-producer`'s existing `RestOperationComposer`-style composition pattern; and it allows BASIS to learn from one bounded execution implementation before generalizing a distributed executor architecture. Same-process execution is not claimed to be inherently more secure in all dimensions — its costs are stated directly in **Alternative A**, below, and in Decision 19.

### Same-Process Does Not Mean Same Responsibility

This ADR explicitly prevents the following interpretation: *because execution happens in `basis-producer`, execution becomes part of the operation-producer role.* That is not the decision. The correct model is:

```text
basis-producer process
    ├── operation-producer logical role
    │     normalization orchestration
    │     evidence lifecycle
    │     gateway submission
    │     disposition handling
    │
    └── protocol-executor logical role
          authorized-operation binding check
          protocol dispatch
          result observation
```

This diagram is conceptual only. The exact internal implementation structure — module layout, class boundaries, whether the two roles share a process supervisor — is not decided by this ADR.

### 4. Do Not Create `basis-executor`

Selecting a logical protocol-executor role does not create or justify `basis-executor`, a new Foundation repository, a new network service, a daemon, a queue, a message bus, or a new deployment unit. The discovery assessment found no current evidence that a separate execution repository is required for the first bounded slice (assessment §2's own governing principle: "a logical architectural role does not automatically imply a repository, process, service, package, or deployment unit"). This ADR does not reserve a repository name, does not create placeholders for one, and does not add `basis-executor` to the ecosystem component table as an existing component. Future architecture may revisit process/repository separation if concrete evidence justifies it.

### 5. Preserve Separate-Executor Topology as a Valid Future Option

This ADR does not prohibit a future separated executor. A future deployment may place the protocol executor in a separate process or service where justified by requirements such as device-credential isolation, reduced single-process compromise blast radius, isolation of protocol-stack dependencies, protocol-specific runtime requirements, network-zone placement, independent lifecycle or scaling, safety or regulatory requirements, or deployment-specific trust-boundary requirements. However: **a separated executor introduces a new authenticated trust boundary and therefore requires additional architecture before it can be considered conforming.** That future architecture would need to address, at minimum, producer/executor or gateway/executor authentication; a portable authorization-to-execution binding; replay protection; freshness; handoff integrity; failure semantics; and correlation. This ADR does not decide those mechanisms. A separated topology is allowed as future architecture, not selected for the first reference slice.

### 6. Preserve Protocol-Specific Execution Implementations

This ADR distinguishes topology from protocol implementation. A common logical execution contract may eventually have protocol-specific implementations for REST, BACnet, Modbus TCP, OPC UA, MQTT, DNP3, IEC 61850, KNX, and Niagara — compatible with either same-process or separated execution. This ADR does not decide whether there will eventually be one executor implementation, nine executor packages, protocol-family groupings, a plugin architecture, or separate protocol services. The discovery assessment (§9) found that protocol semantics vary enough that execution cannot be designed as an HTTP-only abstraction, but not enough evidence exists to establish the final implementation taxonomy.

### 7. Original Protocol Operation Must Be Preserved

The discovery assessment's corrected finding (§5, §6) is carried forward as a normative requirement: the first execution architecture must not reconstruct a protocol-native command by reversing gateway canonical action composition, Core authorization semantics, or the gateway response. The chain begins with the original `ProtocolOperation`; the authorization path derives normalized authorization semantics from that operation. A future execution path must retain access to the original protocol-shaped operation and establish that the operation about to be dispatched is the same operation whose normalized semantics were authorized.

**Normative requirement:** execution must operate on the preserved original protocol-shaped operation, or on a representation explicitly proven by future architecture to be equivalent to it; execution must not derive a new protocol command by reverse-mapping the authorization representation. This ADR does not claim the current implementation already provides a sufficient security binding for this requirement — it does not (Decision 8).

### 8. Binding Is Mandatory, Mechanism Is Deferred

**Normative requirement:** no protocol dispatch may occur unless the execution attempt is bound to the exact operation that received the authoritative authorization disposition. This protects against the threat the discovery assessment states directly: `authorize operation A → mutation/substitution → execute operation B`.

This ADR does not select the binding mechanism. It does not decide whether the adapter evidence digest is sufficient, whether another digest is required, whether gateway evidence participates, whether an execution grant exists, whether a signed artifact exists, whether a nonce exists, whether a one-time-use record exists, whether process memory is sufficient, or whether a new schema is required. Instead, this ADR establishes a decision gate: **a normative authorization-to-execution binding specification must be accepted before the bounded execution implementation may dispatch an operation** (Gate 1, below). A technical spike may be authorized later if architecture cannot determine the mechanism safely without implementation evidence; this ADR does not implement or design that spike.

### 9. No Dispatch Before Authoritative Permission

The following execution invariants are normative. A protocol executor must not dispatch when: the gateway disposition is `DENY`; evaluation failed; authentication failed; producer admission failed; request composition failed; the gateway response is malformed or contradictory; the authorization-to-execution binding cannot be validated; or any required pre-dispatch security state is unavailable. Execution must fail closed. Restated from `operation-producer-and-execution-boundary.md` §5: **execution must never begin before the authoritative disposition permitting execution has been received and validated.** This ADR does not introduce speculative execution, pre-dispatch, optimistic dispatch, or asynchronous "authorize while executing" behavior.

### 10. The Executor Must Not Reinterpret Authorization

The protocol executor does not call `basis-core`; does not reevaluate policy; does not reinterpret a denied request into a safer request; does not downgrade a requested operation; does not retry around authorization; does not infer permission from prior success; does not treat gateway failure as permission; and does not use a device response as evidence that authorization should have been granted. If execution requires a materially different operation than the one authorized, that different operation requires its own authorization.

### 11. Authorization Evidence and Execution Evidence Remain Separate

**Normative invariant, restated without weakening from `operation-producer-and-execution-boundary.md` §5:** authorization evidence and execution evidence are separate artifacts representing different facts at different times. Execution evidence must never mutate, replace, reinterpret, or "complete" the kernel's authorization evidence.

```text
ALLOW + execution failed          remains: authorization = allowed, execution = failed
                                   (never: authorization = failed)

ALLOW + execution status unknown  remains: authorization = allowed, execution = unknown

unauthorized execution succeeded  remains: authorization violation, execution happened
                                   (never a legitimate authorization record merely
                                    because the endpoint accepted the command)
```

This is the structural reason `GatewayAuditEvent` deliberately has no `enforcement_status` field today (discovery assessment §10). This ADR does not define the execution-evidence schema (Gate 3).

### 12. Preserve Uncertainty

**Normative truthfulness principle, without defining the final lifecycle vocabulary:** execution reporting must preserve uncertainty rather than infer non-occurrence from missing confirmation. At minimum, a future execution-lifecycle specification must preserve these distinctions conceptually: not attempted; attempted; remote acknowledgement observed; remote rejection observed; outcome unknown; resulting state verified; resulting state not verified; and evidence persistence success/failure. This ADR does not make these a normative enum, does not define field names, and does not publish schemas (Gate 2).

Specific invariants, carried forward from the discovery assessment's §8 governing distinctions: timeout does not prove that nothing happened; failed does not automatically mean not executed; protocol acknowledgement does not independently verify final state; execution-evidence persistence failure is not execution failure; and protocol transaction progression is not proof of equivalent physical progression. The future execution-lifecycle specification defines the governed vocabulary.

### 13. Preserve Protocol Neutrality

This ADR does not define execution as though every protocol behaves like HTTP. The protocol-executor role must accommodate different execution semantics across synchronous request/response, asynchronous publish, acknowledgement/no-acknowledgement protocols, multi-stage control sequences (DNP3 and IEC 61850's select/operate model), subscriptions, protocol-level negative responses, independent state readback, and uncertain outcomes — per the protocol-specific evidence in discovery assessment §9. The first bounded reference slice may use REST because it minimizes unrelated protocol complexity (see **First Bounded Reference Scope**, below). That does not make REST semantics the universal execution model.

### 14. Credential Classes Remain Distinct

This ADR preserves the distinctions among: authorization-subject credential; producer workload credential; possible future executor workload credential; and protocol/device credential. For the selected same-process first-reference topology: the producer workload identity and authorization-subject identity remain exactly as defined by ADR-0008; colocating the logical executor does not merge those identities; a device/protocol credential, if required by a later bounded implementation, belongs conceptually to the executor responsibility and must not become the authorization subject; device credentials must not be derived from the producer's mTLS identity; and device credentials must not be embedded in adapter evidence or authorization evidence. This ADR does not define a credential store and does not decide a secret-management product. It does not require device credentials in the first reference slice if the bounded REST target can avoid them. General device-credential custody/scoping, if it remains unresolved after the first slice's scope is fixed, is recorded as a downstream architecture decision (Gate 4).

### 15. Same-Process Does Not Eliminate the Binding Requirement

This ADR rejects the following inference: *because producer and executor share memory, no authorization-to-execution binding is required.* That is not acceptable. Same-process topology removes a cross-process transport boundary. It does not remove the requirement that execution be demonstrably tied to the operation that was authorized (Decision 8). Bugs, mutation, confused internal composition, compromised code paths, retries, and stale state can still cause `authorized A → executed B` within a single process. The future binding specification may determine that a simpler mechanism is sufficient in a same-process topology; this ADR does not pre-decide what that mechanism is.

### 16. Execution Does Not Expand `basis-adapters`

Restated as a consequence, not as a redesign: `basis-adapters` remains a normalization library, deterministic evidence-material construction, network-free with respect to OT dispatch, free of device credential custody, and free of protocol execution. This ADR does not move dispatch into adapters, does not add protocol clients to `basis-adapters`, and does not weaken its current non-responsibilities.

### 17. Execution Does Not Expand `basis-core`

`basis-core` remains deterministic, synchronous, side-effect-free, protocol-neutral, transport-independent, credential-independent, and persistence-independent. It must not execute, track execution state, perform retries, hold device credentials, perform protocol acknowledgement processing, or produce execution evidence. The kernel decision remains complete when evaluation completes.

### 18. Execution Does Not Move into `basis-gateway`

`basis-gateway` remains the authorization enforcement boundary: it authenticates, admits, composes, invokes Core, classifies, and returns the authoritative disposition, recording gateway-owned authorization/enforcement-boundary facts. It does not become a protocol dispatcher. This ADR does not add endpoint credentials, OT protocol stacks, or execution state to the gateway architecture. A future binding design may reference gateway-owned authorization facts, but that does not make the gateway the executor.

### 19. Preserve Deployment Security Reality

This ADR explicitly acknowledges a residual risk the discovery assessment states plainly (§11, §12): a process that possesses valid OT endpoint credentials and network reach to an endpoint may technically be capable of bypassing BASIS authorization if that process is compromised. Same-process execution therefore cannot, by application architecture alone, guarantee that a fully compromised executor will obey authorization. Mitigation may require deployment controls such as network segmentation, endpoint ACLs, credential scoping, least privilege, device-side controls, and independent monitoring — none of which this ADR designs. Stated directly: **BASIS software can enforce correct behavior for non-compromised conforming components; some bypass resistance against a fully compromised executor depends on deployment controls outside the authorization decision itself.**

---

## Alternatives Considered

**Alternative A — Same-process logical executor.** Selected as the default topology for the first bounded reference slice (Decision 3). *Reasons:* fewest new trust boundaries; lowest initial operational complexity; simpler authorization/execution correlation; no portable execution handoff required initially; natural continuation of the current `basis-producer` composition model; good fit for bounded, air-gapped, and constrained deployment learning; allows implementation evidence before generalizing distributed execution. *Costs, stated directly rather than minimized:* wider `basis-producer` runtime responsibility; potential device-credential concentration; larger single-process compromise blast radius; future protocol-stack dependencies may conflict; eventual extraction may be desirable.

**Alternative B — Separate executor process/service.** Not selected as the first reference default; preserved as a valid future topology (Decision 5). *Advantages:* credential isolation; protocol-stack isolation; smaller per-process blast radius; independent lifecycle; network placement flexibility. *Costs:* new authenticated trust boundary; portable authorization binding required; replay/freshness complexity; additional failure modes; additional monitoring/deployment burden; additional contract design. Future architecture may select this topology for specific deployment classes without invalidating this ADR's logical-role model.

**Alternative C — Execution inside `basis-adapters`.** Rejected. It violates adapter library purity, introduces live protocol communication, introduces credentials and state, breaks current repository responsibilities, and conflicts with existing adapter invariants — every one of the nine adapters' own module docstrings makes an explicit "no live protocol communication" claim, and DNP3 and IEC 61850's adapters go further, stating directly that "stateful control sequencing belongs to a runtime enforcement boundary... not a normalization library" (discovery assessment §5).

**Alternative D — Execution inside `basis-gateway`.** Rejected. It mixes protocol dispatch with authentication, composition, and enforcement; expands the gateway's blast radius; introduces protocol stacks and device credentials into a shared authorization service; and blurs authorization and execution — the gateway's own audit-event design (`GatewayAuditEvent` deliberately omitting an `enforcement_status` field) is itself evidence its own architects have already reasoned about, and declined, absorbing an execution-result concept.

**Alternative E — Execution inside `basis-core`.** Rejected categorically. It violates kernel isolation, introduces side effects, destroys deterministic evaluation purity, and introduces protocol, network, and state concerns the kernel's foundational property explicitly excludes.

**Alternative F — New `basis-executor` repository now.** Rejected/deferred. There is currently no evidence that the first bounded reference slice requires a separate repository or process. A logical role is not sufficient justification for repository creation (Decision 4). Future evidence may reopen the question.

---

## Consequences

### Positive

- BASIS now has a normative execution-role boundary.
- The first reference topology is no longer ambiguous.
- Implementation cannot accidentally place execution in adapters, gateway, or Core.
- The initial execution slice avoids an unnecessary second network trust boundary.
- Future separated deployments remain possible.
- The authorization/execution distinction becomes binding architecture.
- Supporting specifications have clear prerequisites and scope.

### Negative / Tradeoff

- `basis-producer` will eventually host more than authorization-side orchestration in the first reference topology.
- Compromise of the colocated producer/executor process has a larger blast radius than a separated topology would.
- Protocol/device credentials may eventually be colocated with producer credentials.
- Protocol-stack dependencies may increase runtime complexity.
- Same-process topology does not solve authorization binding (Decision 15).
- Additional architecture is still required before execution can begin (Follow-On Decision Gates, below).
- A future production topology may require separating a role that the first reference implementation colocates.

These costs are accepted for the reasons stated in Decision 3, not because they are minimal.

---

## Security Consequences

### Prevented by the architecture

- Dispatch before authorization.
- Dispatch after `DENY`.
- Execution being treated as proof that authorization occurred.
- Execution evidence overwriting authorization evidence.
- Reconstruction of arbitrary protocol commands from authorization semantics.
- Silent placement of protocol execution in adapters, gateway, or Core.

### Still requiring follow-on architecture

- The exact authorization-to-execution binding mechanism (Gate 1).
- Replay/freshness semantics (Gate 1).
- Device credential custody/scoping, where the first slice requires one (Gate 4).
- The execution-lifecycle vocabulary (Gate 2).
- Execution-evidence provenance and shape (Gate 3).
- Executor workload authentication for a future separated topology (Decision 5).

### Residual risks

- A compromised same-process executor may possess direct device reach (Decision 19).
- A compromised executor can potentially fabricate its own observations, the same residual-risk structure ADR-0007 already accepts for adapter evidence ("hashing does not make a compromised producer trustworthy").
- Protocol acknowledgement may not prove resulting physical state.
- Unknown execution outcomes remain unavoidable for some failure/protocol conditions (discovery assessment §14's crash-consistency analysis).
- Deployment/network controls remain necessary; they are not designed by this ADR.

---

## Follow-On Decision Gates

Acceptance of this ADR does not immediately authorize execution implementation. At minimum, before any bounded execution implementation can dispatch a protocol operation, architecture must resolve the following gates.

**Gate 1 — Authorization-to-Execution Binding.** A normative architecture specification or ADR must define how the exact preserved operation is bound to the authoritative authorization result. It must answer enough of: what is bound; where the binding is created; where it is verified; what mutation is prohibited; what identifier/digest/artifact participates; what same-process guarantees are sufficient; and what must differ for a separated topology. This ADR does not solve it.

**Gate 2 — Execution Lifecycle Semantics.** A normative architecture specification must define truthful execution-state semantics sufficient for the first bounded protocol. It must preserve uncertainty (Decision 12) and avoid assuming all protocols have HTTP-style acknowledgement behavior. This ADR does not define the final enum.

**Gate 3 — Minimum Execution-Evidence Semantics.** Architecture must establish which logical role constructs execution evidence; which facts are observations versus authoritative facts; minimum correlation requirements; persistence/failure ordering; and separation from authorization evidence (Decision 11). This ADR does not publish the schema. Schema publication remains downstream of a stable, implementation-informed shape, per `ecosystem-contract-inventory.md`'s "implementation proves a stable shape" principle.

**Gate 4 — Credential Custody, If Required by the First Slice.** If the first execution target requires a protocol/device credential, architecture must define the bounded custody/scoping rule before introducing that credential. This gate does not apply if the first synthetic REST execution target can avoid endpoint credentials entirely.

**Gates 1 through 3 are unconditional prerequisites for any bounded execution implementation. Gate 4 is conditional: it applies only if the selected bounded target actually requires a protocol/device credential, and does not apply to a credential-free target.** Wherever this ADR or its governance process refers to "the decision gates" or "the follow-on gates" as a complete set, that set means Gates 1 through 3, plus Gate 4 where applicable — never four unconditional gates.

---

## What This ADR Deliberately Leaves Open

This ADR does not decide any of the following: the exact binding algorithm; the exact digest input; a new canonicalization profile; a signed execution token; a JWT execution grant; a nonce; a TTL or freshness duration; a retry count; an idempotency key; a replay database; an execution-attempt schema; an execution-evidence schema; the final lifecycle enum; a provenance-classification enum extension; long-term evidence storage; a device credential store; a secret-management technology; separate-executor authentication; separate-executor transport; a production high-availability execution topology; a protocol execution interface; a plugin architecture; a queue or message bus; a daemon or API shape; or a generalized multi-protocol implementation. These belong to later architecture decisions or implementation learning.

---

## First Bounded Reference Scope

The intended first reference execution slice should remain **REST first**, because the completed producer reference slice already uses REST and it minimizes unrelated protocol complexity, per the same "REST first" reasoning ADR-0008 already used successfully for the authorization side. This ADR does not define the REST execution implementation. It does not select a specific HTTP library. It does not select a target service. It does not define request/response mappings. It does not authorize contacting real production endpoints. It does not create tests or runtime code. The first bounded implementation should eventually demonstrate the execution architecture with the smallest realistic target needed to validate the architectural guarantees; this ADR does not design that target.

---

## ADR Acceptance Boundary

This ADR is precise about what its eventual acceptance would authorize:

> Acceptance of ADR-0011 establishes the protocol-executor role, selects same-process colocation with `basis-producer` as the default topology for the first bounded reference slice, and fixes the governing execution invariants. It does not authorize bounded execution implementation — including dispatch — until Gates 1 through 3 identified by this ADR are satisfied — and Gate 4 as well, if the selected bounded target requires a protocol/device credential.

Acceptance does not mean implementation may now begin. Implementation remains blocked by the supporting decision gates above.

---

## Relationship to ADR-0010

ADR-0010 established `basis-producer` as the permanent operation-producer-runtime implementation. This ADR does not reverse or rewrite ADR-0010. Instead, for the first bounded reference topology: the existing `basis-producer` runtime is permitted to host an additional logical protocol-executor role; the operation-producer role itself retains the responsibilities ADR-0010 gave it; and execution remains a separate logical responsibility inside the same runtime boundary. This is not "`basis-producer` Phase 6" — the completed five-phase producer authorization implementation plan remains complete, and any later execution implementation plan is a new architecture chapter and a new implementation plan, not an extension of the old Phase 2A–5 numbering.

## Relationship to ADR-0008 / ADR-0009

This ADR preserves producer mTLS workload identity; independent bearer authorization-subject identity; trusted NGINX ingress; gateway-owned URI SAN derivation; and exact producer admission. Execution does not change those semantics. A same-process executor does not acquire a new identity merely because it is logically distinct from the producer. A future separate-process executor may require its own workload identity, but that is deferred and would require separate architecture (Decision 5).

## Relationship to ADR-0007

ADR-0007 remains authoritative for adapter evidence construction. This ADR does not repurpose adapter evidence into execution evidence. The adapter evidence digest may eventually participate in authorization-to-execution binding if later architecture determines it is appropriate (Gate 1) — this ADR does not decide that it does.

---

## Non-Goals

This ADR does not: implement execution; contact an OT endpoint; add REST, BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, or Niagara dispatch; modify `basis-producer`, `basis-gateway`, `basis-adapters`, `basis-core`, `basis-identity`, or `basis-schemas`; create `basis-executor` or any other new repository; define execution-evidence schemas; define an execution lifecycle enum; define an execution grant; define a signed token; introduce cryptographic keys; define TTL or freshness values; define retry behavior; define idempotency behavior; define device credential storage; create an implementation plan; or create "`basis-producer` Phase 6."

---

## Validation / Implementation Gate

Formal acceptance of ADR-0011 establishes the execution-model direction — the protocol-executor role, the same-process default topology for the first bounded reference slice, and the governing execution invariants — but does not itself open bounded execution implementation. Except for a separately authorized bounded technical spike needed to resolve a specific architectural feasibility question that architecture cannot responsibly answer from available evidence (the same narrow precedent `basis-gateway`'s Phase 1A mTLS-termination spike set on the path to ADR-0009), Gates 1 through 3, plus Gate 4 if applicable to the selected bounded target, must be satisfied before bounded execution implementation begins; actual dispatch remains prohibited until that point regardless. Such a spike must be explicitly scoped to the specific question it is authorized to answer, authorized by architecture rather than self-initiated by an implementation team, non-production, and must not itself constitute — or silently become — the execution reference implementation. This ADR, once merged, does not itself constitute acceptance, consistent with this repository's established convention (see ADR-0010's own Validation / Implementation Gate section and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself change its status to `Accepted`. After this ADR merges, a separate formal architect acceptance review, consistent with recent ADR governance practice, determines whether ADR-0011 is ready to move from `Proposed` to `Accepted`. No implementation is authorized by this ADR, and none is authorized by this correction pass.

## References

- [`docs/architecture/execution-boundary-discovery-assessment.md`](../architecture/execution-boundary-discovery-assessment.md) — the primary, non-normative evidence source for this ADR: current-state inventory (§3–§4), logical-role decomposition (§5), authorization-to-execution binding analysis (§6), replay and freshness (§7), execution lifecycle and failure semantics (§8), protocol-specific execution semantics (§9), execution evidence (§10), credentials and trust (§11), compromise and bypass analysis (§12), correlation model (§13), side-effect ordering (§14), deployment topologies (§16), option comparison (§18), decision inventory (§19), and ADR recommendation (§20)
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) — §2 (logical roles this ADR's protocol-executor role definition inherits), §5 (the authorization-to-execution lifecycle invariants this ADR restates as normative), §9 (deployment topologies)
- [ADR-0007](0007-adapter-evidence-construction.md) — adapter evidence construction ownership; preserved unchanged by this ADR (Relationship to ADR-0007)
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) — producer workload authentication and admission; preserved unchanged by this ADR (Relationship to ADR-0008 / ADR-0009)
- [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) — trusted mTLS ingress and certificate-handoff topology; preserved unchanged by this ADR (Relationship to ADR-0008 / ADR-0009)
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) — establishes `basis-producer` as the permanent operation-producer-runtime repository and component; this ADR's own default-topology decision (Decision 3) is not a reversal or extension of ADR-0010's own bounded-implementation scope (Relationship to ADR-0010)
- [`docs/architecture/adapter-evidence-construction-semantics.md`](../architecture/adapter-evidence-construction-semantics.md) — adapter evidence material, canonicalization, and digest semantics this ADR does not repurpose as execution evidence
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component-boundary and dependency-direction model this ADR does not add `basis-executor` to
- [`docs/security/threat-model.md`](../security/threat-model.md) — the compromised-adapter, compromised-gateway, and deployment/network-guarantee framing this ADR's Security Consequences section restates for the execution boundary
- [`docs/glossary.md`](../glossary.md) — terminology this ADR's role naming is reconciled against
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR acceptance process this ADR's `Proposed` status observes
