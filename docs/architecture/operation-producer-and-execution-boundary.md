# Operation-Producer and Execution-Evidence Boundary

**Status:** Architecture-planning. This document defines logical roles, trust rules, correlation semantics, and a decision-gated implementation sequence for the next unresolved boundary in the identity-to-operation chain. It does not implement runtime behavior in any repository, does not add or modify a schema, and does not select an authentication technology.

**Scope:** The unresolved gap between `basis-adapters` normalization and `basis-gateway` trusted-producer ingestion, and the further, distinct gap between `basis-gateway` authorization and actual OT protocol execution. This document names the logical roles across that chain, states how trust in an operation producer must be established, defines the request handoff and authorization-to-execution lifecycle, defines a correlation model across the chain, records a provenance and fact-ownership matrix, discusses failure and degraded conditions, describes deployment topologies without prescribing one, restates repository ownership, evaluates alternatives for where a new runtime role might live, inventories what existing schemas can and cannot already represent, and proposes a narrow, decision-gated implementation sequence.

**Companion documents:** [`ROADMAP.md`](../../ROADMAP.md) (**Next Producer and Execution-Evidence Boundary**), [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](../roadmaps/identity-to-operation-contract-and-interoperability.md) (architectural invariants, contract responsibility model, ten-phase program this document narrows into a first, concrete boundary), [`docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`](../roadmaps/post-authentication-identity-activity-correlation-and-detection.md) (the successor roadmap this document's evidence and execution-lifecycle work is a prerequisite for), [`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`docs/architecture/basis-adapters.md`](basis-adapters.md), [`docs/architecture/basis-gateway.md`](basis-gateway.md), [`docs/architecture/basis-identity.md`](basis-identity.md), [`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md), [`docs/architecture/operation-aware-evidence-provenance-semantics.md`](operation-aware-evidence-provenance-semantics.md), [`docs/security/threat-model.md`](../security/threat-model.md), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/glossary.md`](../glossary.md).

---

## 1. Problem Statement

`basis-adapters` normalizes a protocol-specific operation into a `NormalizedAuthorizationRequest` and preserves the originating protocol operation as evidence. It does this without calling `basis-gateway`, without authenticating a caller, and without evaluating whether the operation should be allowed — the normalization contract requires exactly that separation. `basis-gateway`, for its part, accepts an operation-aware request only from a caller it has already authenticated, and treats a narrow set of producer-only fields (`operation_intent`, `location`, `device`, `protocol_context`, `safety_context`, `environment_context`, `risk_context`, `identity_evidence_reference`, `adapter_evidence_reference`) as acceptable only from a caller it has separately classified as a trusted operation producer via `OPERATION_PRODUCER_SUBJECT_IDS`.

Between those two, well-specified components sits a gap this ecosystem has not yet defined: **how a `NormalizedAuthorizationRequest` produced by `basis-adapters` actually becomes an authenticated, trusted-producer request accepted by `basis-gateway`.** Nothing in `basis-adapters` authenticates to the gateway. Nothing in `basis-gateway` accepts an unauthenticated adapter output directly. The two contracts are compatible in shape — the `AdapterEvidenceReference` model that `basis-gateway`'s operation-aware request accepts is structurally built to reference the evidence `basis-adapters` preserves — but no component in the ecosystem today performs the handoff between them. `ROADMAP.md` names this the *next remaining integration boundary*, not yet started.

A second, distinct gap exists on the other side of authorization. `basis-gateway` enforces its own boundary: it invokes `basis-core`, classifies the kernel's outcome to an HTTP disposition, and records that disposition in a `GatewayAuditEvent`. It states this limit explicitly in its own documentation: *"No adapter execution confirmation — the gateway proves an authorization decision, an enforcement disposition, and an HTTP result; it does not prove that a physical device executed the operation."* `gateway-audit-event`'s own schema makes the same statement structurally: `enforcement_action` records the action the gateway *selected*, never a guarantee that the selected action was carried out end-to-end by a downstream device. Nothing in the ecosystem today dispatches an authorized operation to an OT protocol endpoint, and nothing records what happened when — or whether — that dispatch occurred.

Today the system can prove:

- who called the gateway (the authenticated caller's verified `subject_id`);
- what operation was requested (the composed action, resource, and, where a trusted producer supplied it, operation intent and context);
- what the gateway accepted as trusted-producer assertions (the producer-only fields, provenance-classified as `trusted_producer_asserted`, never promoted to `verified`);
- what the kernel decided (`OperationAwareDecisionResponse` / `EvaluationTrace` / `AuditEvidence`, produced deterministically and independently of enforcement);
- what disposition the gateway enforced at its HTTP boundary (`GatewayAuditEvent.enforcement_action`, recorded beside — never inside — the kernel's evidence).

It cannot yet prove:

- that an OT protocol operation was dispatched to a device or endpoint;
- that a device accepted, rejected, or even received the operation;
- that the requested state change actually occurred;
- that an observed result — if one is ever recorded — corresponds to the specific authorized request that produced it.

Both gaps share a root cause: the ecosystem has fully specified the components on either side of each gap (adapters normalize; the gateway authenticates, classifies, and enforces; the kernel decides) but has not yet specified what runs *between* normalization and gateway ingestion, or what runs *after* gateway enforcement and before a device sees the operation. This document names those missing logical roles, states the trust and provenance rules that govern them, and sequences the decision-gated work required to fill them — without assigning that work to an implementation repository prematurely and without inventing runtime behavior this document is not authorized to add.

---

## 2. Logical Roles

The chain from protocol operation to execution evidence involves more logical roles than there are repositories today. A single deployment may combine several of these roles into one process (§9); combining them operationally does not collapse them conceptually, and each role's obligations still apply to whatever process performs it.

### Operation initiator

The human, machine, workload, service, automation rule, or upstream system requesting an operation. The operation initiator's identity may or may not be the same as the operation producer's authenticated workload identity (§3) — an operator using a supervisory HMI is an operation initiator whose request is carried, but not authenticated as, the operation producer that actually submits to the gateway.

### Adapter normalization library

The existing `basis-adapters` responsibility, unchanged by this document. It translates protocol-specific intent (a `ProtocolOperation`) into normalized BASIS semantics (a `NormalizedAuthorizationRequest`, carrying `protocol_evidence` verbatim) and preserves protocol evidence. It performs no network I/O, does not authenticate to the gateway, does not authorize, does not enforce, and does not execute. It has no gateway client and no knowledge of `basis-gateway`'s existence. This is unchanged from the normalization contract `basis-adapters` released at `v0.1.0` and has preserved unmodified through its subsequent `v0.2.0` release (which added adapter-side evidence-material construction per ADR-0007 Stage 1, not a change to normalization behavior) (see [`basis-adapters.md`](basis-adapters.md) in this repository, and `docs/contracts/normalization-contract.md` and `docs/contracts/adapter-contract.md` in the separate `basis-adapters` repository — referenced by path, not hyperlinked, per this repository's cross-repository citation convention).

### Operation-producer runtime

The authenticated software workload that:

- invokes the adapter normalization library and receives its `NormalizedAuthorizationRequest` and `AdapterResult`;
- binds the normalized operation to its own authenticated workload identity (not the operation initiator's identity, and not a value it invents);
- constructs an `AdapterEvidenceReference` (or accepts one the adapter library helps construct) referencing the adapter's preserved evidence by digest, never forwarding a raw protocol payload;
- submits the resulting operation-aware request to `basis-gateway`, supplying only the producer-only fields it is authorized to assert;
- preserves the relationship between the normalized intent it received and the adapter evidence it references — it does not recompose, reinterpret, or "improve" what the adapter normalized;
- receives the gateway's disposition;
- prevents execution unless that disposition permits it, and fails closed (denies the operation) on any of its own internal errors, exactly as `basis-adapters`' own fail-closed rule already requires of normalization failures.

This role does not exist as a named repository or component today. This document calls it the *operation-producer runtime* as working vocabulary, consistent with how both `docs/roadmaps/identity-to-operation-contract-and-interoperability.md` and `docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md` introduce their own working vocabulary: naming it here does not promote it to the glossary, and does not commit the ecosystem to a particular implementation shape. It may eventually be called an adapter host, a producer runtime, or an integration runtime. §11 evaluates, but does not decide, whether it requires a new repository.

### Gateway

The existing `basis-gateway` responsibility, unchanged by this document. It authenticates the producer runtime; classifies whether the authenticated caller is a trusted operation producer (today, exclusively via `OPERATION_PRODUCER_SUBJECT_IDS`, exact-match, case-sensitive, empty by default — see §3); accepts or rejects producer-only context accordingly; records every accepted field's provenance; invokes `basis-core`; enforces the returned disposition at its own HTTP boundary; and records `GatewayAuditEvent` beside the kernel's unmodified `AuditEvidence`. It does not parse OT protocols, does not independently verify a producer's protocol claims, and — per its own documented limitation — does not claim that device execution occurred.

### Protocol executor

The component that, after the gateway's disposition permits it, actually communicates with the protocol endpoint or platform — opening the BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, or Niagara session and dispatching the operation. The protocol executor owns protocol dispatch mechanics. It does not reinterpret the authorization decision it received; a disposition that does not permit execution is not a decision the executor is positioned to override, question, or retry around. No such component exists in the ecosystem today — `basis-adapters` is explicitly a normalization library with "no sockets, no packet parsing, no protocol stacks, no live protocol communication," and no other repository has assumed protocol dispatch.

### Execution-evidence producer

The component that records what was observed after authorization permitted an attempt:

- execution was not attempted (because the disposition did not permit it, or because the operation-producer runtime failed closed before dispatch);
- dispatch was attempted;
- dispatch was rejected locally (before reaching the network or the field bus);
- the remote endpoint accepted or rejected the operation;
- the result is unknown because an acknowledgement was not available (protocol has no ack primitive, timeout, connectivity loss);
- an observable state change was, or was not, confirmed by a subsequent read or report.

This document does not finalize a machine-readable status vocabulary for these outcomes. §5 and §8 discuss the closest existing frame — the five-state sketch in `docs/roadmaps/identity-to-operation-contract-and-interoperability.md`'s Phase 4 (`authorized-but-not-attempted`, `attempted-and-completed`, `attempted-and-failed`, `partially-applied`, `execution-status-unavailable`) — but that sketch is roadmap language, not a governed contract, and this document does not promote it to one.

---

## 3. Trust Establishment

Trust in an operation producer is not one fact. This document distinguishes six separate facts that are routinely conflated when discussing "trusted producers," because conflating them is exactly the failure mode the ecosystem's threat model already names for the existing, narrower gateway↔adapter trust boundary (`docs/security/threat-model.md` §3.3, §4.2, §6.3 — rogue integrator and compromised-adapter adversaries; "an adapter receiving untrusted protocol input does not treat the protocol channel as evidence of authorization").

| Distinct fact | What it answers | Where it is established today |
| - | - | - |
| Authenticated workload identity | Which software process is making this call? | `basis-gateway`'s existing `authenticate()` dispatch (OIDC or `basis_local_token`) |
| Authorization to act as an operation producer | Is this authenticated workload allowed to submit operation-aware requests at all, and specifically to assert producer-only context? | `basis-gateway`'s `OPERATION_PRODUCER_SUBJECT_IDS` allowlist, exact-match against the already-authenticated `subject_id` |
| Authorization to assert specific context categories | Within producer trust, is this producer allowed to assert *this* category of context (for example, safety context vs. plain protocol context)? | Not established today — `OPERATION_PRODUCER_SUBJECT_IDS` grants an all-or-nothing capability across all nine producer-only fields |
| Protocol evidence | What did the adapter actually observe or receive at the protocol layer? | `basis-adapters`' `protocol_evidence` / `ProtocolOperation`, referenced (not embedded) via `AdapterEvidenceReference` |
| Execution evidence | What happened when the protocol executor attempted the operation? | Not established today — no component or contract exists (§12) |
| Human or initiating subject identity | Who or what originally asked for this operation? | `basis-identity`'s canonical identity context, when the initiator authenticated through it; not necessarily the same identity as the operation producer's workload identity |

Several rules follow directly from keeping these six facts distinct, and are already implemented, in narrower form, by `basis-gateway`'s existing producer-trust module — this document generalizes them to the operation-producer runtime the gateway does not yet see:

- **Producer trust must not be self-asserted in a request body.** `basis-gateway`'s operation-aware schema already enforces this for its own callers by rejecting `is_trusted_operation_producer` / `producer_trust_classification` as caller-supplied fields; the same rule must hold at every future point where producer trust is established, including a future adapter-to-gateway authentication mechanism this document does not select.
- **Roles alone must not silently establish producer trust.** `basis-gateway`'s own module docstring states this as its load-bearing safety property: "never infers trust from... role membership or attribute values." A role such as `service` or `device` on a `basis-identity` `CanonicalSubject` is a classification of subject *type*, not a grant of operation-producer capability.
- **Network location alone is insufficient.** Consistent with the ecosystem's zero-trust framing (`docs/glossary.md`, *Zero Trust*), being reachable from an OT network segment, or presenting from an expected IP range, must never substitute for authenticated workload identity plus an explicit producer-trust grant.
- **Deployment configuration must bind an authenticated workload identity to producer capabilities.** This is what `OPERATION_PRODUCER_SUBJECT_IDS` already does, narrowly, for gateway callers; any future adapter-to-gateway trust mechanism must bind the same way — an explicit, deployment-owned configuration artifact, not code-path inference.
- **A trusted producer's assertions remain assertions unless another component independently verifies them.** This is already stated in `basis-gateway`'s own architecture: every producer-only field is provenance-classified `trusted_producer_asserted`, never promoted to `verified`, because "the gateway cannot independently confirm the truth of a producer's claim (it has no device-state or protocol-parsing capability)." The same limit applies transitively to an `AdapterEvidenceReference`: the gateway trusts that the producer's reference accurately points to genuine adapter evidence; it does not independently verify the digest against a source it can inspect.
- **The current gateway subject-ID allowlist is a valid, bounded first implementation — not the final ecosystem trust model.** `ROADMAP.md` already states this explicitly: `OPERATION_PRODUCER_SUBJECT_IDS` "is not the adapter-to-gateway trusted-producer contract... which remains open." This document does not close that gap; it names the shape of the decision (§13, Stage 4) without selecting mTLS, SPIFFE, OAuth client-credentials, or any other transport-authentication technology, per this document's explicit non-goals. [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) is the proposed (not yet accepted or implemented) architectural answer to that decision gate; it retains the allowlist as a compatibility mechanism rather than removing it.

The category-level granularity implied by the third row above — whether a producer trusted to assert `protocol_context` should also, automatically, be trusted to assert `safety_context` or `risk_context` — is an open question this document surfaces (§7, §11's decision gates) but does not resolve. Today's all-or-nothing model may be an acceptable first implementation for the same reason the subject-ID allowlist is: bounded, explicit, and honestly documented as a placeholder rather than a final model.

---

## 4. Request Handoff

The governed handoff from normalization to gateway submission has four stages, only the first and last of which are implemented today:

```text
ProtocolOperation
    ↓  (basis-adapters: implemented)
NormalizedAuthorizationRequest  (+ protocol_evidence, preserved verbatim)
    ↓  (operation-producer runtime: not yet implemented)
producer runtime binds authenticated producer identity,
constructs AdapterEvidenceReference from adapter evidence,
composes producer-only context it is authorized to assert
    ↓  (operation-producer runtime → basis-gateway: authentication
       mechanism not yet selected)
operation-aware gateway request
    (POST /v1/evaluate/operation-aware, basis-gateway: implemented,
     feature-flagged, disabled by default)
```

The second and third stages are the gap this document names. They are not one stage: binding workload identity to a request is a distinct act from constructing a safe evidence reference, and both are distinct from actually submitting the HTTP request under gateway-recognized authentication.

Field ownership across the handoff, using `basis-gateway`'s existing operation-aware request shape as the reference:

| Field | Owner across the handoff | Notes |
| - | - | - |
| `action`, `resource_type`, `resource_id` | Adapter produces (bare verb, local resource shape); producer runtime and gateway compose into kernel-canonical form | Existing action/resource-composition contracts (`ecosystem-contract-inventory.md` §3.3, §3.5) apply unchanged; this document does not add a new composition rule |
| `protocol_evidence` | Adapter (verbatim, mandatory, never stripped) | Referenced from the gateway request via `adapter_evidence_reference`, not embedded |
| `operation_intent` | Producer runtime asserts, if it knows | Producer-only field; absent unless the producer runtime has a basis for asserting it |
| `location`, `device` | Producer runtime asserts, if known from deployment configuration or protocol evidence | Producer-only fields; not derived by the gateway |
| `protocol_context` | Producer runtime asserts, derived from adapter evidence | Producer-only field; distinct from the mandatory `protocol_evidence` reference |
| `safety_context`, `environment_context`, `risk_context` | Producer runtime asserts, if a governed upstream source exists | Producer-only fields; §3's open category-level trust question applies most acutely here |
| `adapter_evidence_reference` | Producer runtime constructs, from adapter output | References `basis-adapters`' evidence by digest; never a raw protocol payload |
| `identity_evidence_reference` | Producer runtime asserts, if `basis-identity` produced one for the initiating subject | `basis-identity` does not yet produce this contract (§12) |

Two rules govern the handoff regardless of which fields are populated:

**Not every producer must populate every optional operation-aware field.** A producer runtime integrating a protocol with no meaningful safety context, or with no location model, correctly omits those fields. This is not a lesser integration; it is an honest one.

**Absence must remain distinct from empty or unknown.** `basis-gateway`'s own composition code already enforces this — "an omitted optional field stays `None`... the gateway never manufactures an empty-but-present object" — and the operation-producer runtime must preserve the same distinction on its side of the handoff. A producer runtime that cannot determine `risk_context` must omit the field, not assert a synthetic "unknown" value into it; a governed "explicitly unknown" value, if the ecosystem ever needs one, is a schema decision for `basis-schemas`, not something a producer runtime should invent unilaterally.

---

## 5. Authorization-to-Execution Lifecycle

```text
normalize
    → authenticate producer
        → authorize (basis-core evaluates)
            → enforce disposition (basis-gateway)
                → execute only if allowed (protocol executor)
                    → record execution evidence (execution-evidence producer)
```

Only the first four links are implemented (normalize, authenticate, authorize, enforce); the last two are the gap. The following invariants govern the whole chain, and are stated so that implementing the missing links cannot silently violate a guarantee the implemented links already provide:

- **A denied or not-applicable operation must not be dispatched.** This extends `basis-gateway`'s existing enforcement boundary forward: a `deny` or `not_applicable` disposition is not a signal to retry, downgrade, or attempt a "safer" version of the operation at the protocol executor. It is a stop.
- **A failed evaluation must not be dispatched.** `evaluation_status: failed` (any `failure_reason`) is not an implicit allow. The gateway's own HTTP classification already treats every failure reason as a non-2xx outcome; the protocol executor must treat the same failure the same way.
- **Producer-runtime errors must fail closed.** Consistent with `basis-adapters`' existing fail-closed rule for normalization failures, and with the gateway's own fail-closed audit events (`gateway.evaluation_failed_closed`), an operation-producer runtime that cannot construct a valid request, cannot authenticate, or cannot interpret the gateway's response must deny the operation locally rather than guess.
- **Execution must never occur before the authoritative disposition is received.** This rules out speculative or pipelined dispatch patterns that begin protocol communication before the gateway's response arrives, even where doing so might reduce latency.
- **An execution result must not alter or rewrite the prior kernel outcome.** `basis-core`'s `OperationAwareDecisionResponse` and `AuditEvidence` are complete and immutable the moment the kernel produces them (per `operation-aware-trace-audit-evidence.md` and `operation-aware-evidence-provenance-semantics.md`'s existing provenance rules). Execution evidence is a later, separate fact appended to the same correlation chain (§6) — never a patch applied to the decision.
- **Authorization evidence and execution evidence are separate artifacts.** This is the structural reason `gateway-audit-event.yaml` deliberately does not define an `enforcement_status` / `enforcement_result` field today: mixing the two would let an execution outcome silently retcon what was actually decided.
- **Execution failure after an allow decision does not mean the authorization decision was incorrect.** A device offline, a protocol timeout, or a rejected write are facts about the world at execution time, not defects in an evaluation that correctly reasoned about the world at decision time. Treating them as evidence of a bad decision would blur exactly the boundary ADR-0002 and ADR-0003 already establish between deterministic evaluation and everything that happens afterward.
- **Execution success cannot retroactively legitimize an operation that was not authorized.** An operation-producer runtime or protocol executor that dispatches despite a `deny` — because of a bug, a race, or a deliberate bypass — and that dispatch happens to succeed at the protocol layer, has not thereby produced a legitimate operation. This is a security invariant, not a data-modeling one: execution evidence for an operation without a corresponding `allow` disposition is itself the signal of a boundary violation, to be treated with the same seriousness the threat model gives a compromised adapter (`docs/security/threat-model.md` §4.2).

---

## 6. Correlation Model

At minimum, the following identifiers must remain linkable across the chain, each generated by exactly one owning component:

| Identifier | Generated by | Notes |
| - | - | - |
| Caller/producer request ID | Operation-producer runtime (or, absent one, the caller) | Maps to `request_id` on the existing operation-aware request; `basis-gateway` never overwrites a caller-supplied value, but falls back to its own `correlation_id` when absent |
| Gateway-generated correlation ID | `basis-gateway` | Generated unconditionally per request via `CorrelationMiddleware`; caller-supplied `X-Correlation-ID` headers are ignored by explicit, documented policy — accepting them "would allow external parties to influence the audit trail" |
| Kernel trace ID | `basis-core` | `trace_id` on `OperationAwareDecisionResponse` / `EvaluationTrace`; gateway-generated per evaluation call today, per `operation-aware-endpoint.md`'s response table, and passed through unmodified into `GatewayAuditEvent` |
| Kernel audit evidence ID | `basis-core` | `AuditEvidence.evidence_id`; referenced, never embedded, from `GatewayAuditEvent.audit_evidence_id` — the linkage invariant `basis-gateway`'s audit model already states explicitly (`gateway_audit_event.audit_evidence_id == audit_evidence.evidence_id`) |
| Adapter evidence reference ID | Operation-producer runtime | `AdapterEvidenceReference.reference_id`; minted by the operation-producer runtime, never by `basis-adapters` itself — see [`docs/architecture/adapter-evidence-construction-semantics.md`](adapter-evidence-construction-semantics.md) §7–§8 for why minting is kept outside the pure, deterministic normalization call. Carries its own optional `request_id` / `correlation_id` pass-through fields today, unenforced against the gateway's own correlation ID |
| Identity evidence reference ID | `basis-identity` (via whatever composes the request) | `IdentityEvidenceReference.reference_id`; not yet produced by `basis-identity` today (§12) |
| Future execution evidence ID | Execution-evidence producer | Does not exist yet; §12 records this as a schema gap, not a decision |

Two rules, restated from the components that already implement correlation today and generalized forward:

**No component may overwrite an identifier owned by another component.** This is already `basis-gateway`'s explicit policy for the correlation ID it generates, and it must hold symmetrically in the other direction: a future execution-evidence producer must not overwrite the `request_id`, `correlation_id`, or `trace_id` it receives from the authorization chain — it appends its own identifier and references the ones that came before it, exactly as `GatewayAuditEvent` references `AuditEvidence` rather than absorbing it.

**Correlation creates deterministic linkage; it does not by itself prove authenticity or causation.** A shared correlation ID across an `AuditEvidence` record and a future execution-evidence record proves that the two records claim to describe the same operation. It does not prove that the execution-evidence record is genuine, that the component which wrote it was the component authorized to, or that the execution it describes actually corresponds to the authorized operation rather than a coincidentally similar one. This is the same caution the threat model already applies to replay protection (`docs/security/threat-model.md` §7.6): correlation identifiers "support detection of duplicates in audit," they do not themselves prevent duplication or forgery.

**Adapter evidence construction ownership**, decided in full by [`docs/architecture/adapter-evidence-construction-semantics.md`](adapter-evidence-construction-semantics.md) (adopted by [ADR-0007](../adr/0007-adapter-evidence-construction.md)) and restated here only to keep this document's own correlation-model row (above) consistent with it:

```text
basis-adapters
    constructs the governed evidence material
    canonicalizes it with RFC 8785
    computes the digest

operation-producer runtime
    mints reference_id
    assembles AdapterEvidenceReference
    supplies adapter_source
    assigns redaction classification
    attaches request linkage

basis-gateway
    authenticates and classifies the producer
    validates and admits the reference
    does not regenerate the evidence or digest
```

This document's own §2 and §4 already assign reference *assembly* to the operation-producer runtime; the correction above is narrower — it is specifically the `reference_id` value that this document's earlier correlation-model table (§6) had mis-stated as generated "by `basis-adapters`," which was inconsistent with this document's own §4 field-ownership table and with `basis-adapters`' own handoff-alignment plan, which confirms `basis-adapters` "generates no identifier of any kind" during normalization. Minting belongs to the operation-producer runtime alone.

---

## 7. Provenance and Fact Ownership Matrix

| Fact | Owner | Trust meaning |
| - | - | - |
| Producer workload identity | Authentication boundary (today: `basis-gateway`'s `authenticate()`; future: whatever authenticates the operation-producer runtime) | Verified according to the configured authentication mode; not self-asserted |
| Human initiating identity | Identity/gateway chain (`basis-identity` → `basis-gateway`) | Verified only when cryptographically or contractually bound (a `basis-identity` canonical identity context or BASIS-local token) — otherwise carried only as an unverified hint, exactly as `basis-adapters`' existing `subject_hint` field is already documented to be |
| Normalized action/resource | Adapter + producer handoff | Producer-asserted normalization result; the gateway composes it into canonical form but does not independently verify that the adapter normalized correctly (the threat model's existing "adapter is trusted to normalize accurately, not to authorize" boundary applies unchanged) |
| Protocol evidence | Adapter / producer runtime | Preserved observation or input, referenced by digest via `AdapterEvidenceReference`; not independently verified by the gateway, which "has no device-state or protocol-parsing capability" |
| Kernel outcome | `basis-core` | Authoritative authorization result — the one fact in this table produced by deterministic, testable, protocol-independent evaluation rather than by trust in an external assertion |
| Gateway disposition | `basis-core` result, enforced by `basis-gateway` | Authoritative enforcement disposition at the gateway's own boundary |
| HTTP result | `basis-gateway` | Result at the gateway boundary only; not a claim about anything downstream of it |
| Protocol dispatch result | Protocol executor | Executor-observed result; does not exist as a produced fact anywhere in the ecosystem today |
| Device-state confirmation | Protocol executor or independent observer | Observed state, with limitations explicitly recorded (acknowledgement availability varies enormously by protocol — compare DNP3's stateless SBO model, which has no persistent select-state to confirm against, with a protocol that supports a genuine read-back) |

This table uses the ecosystem's existing provenance vocabulary — `basis-gateway`'s closed `ProvenanceClassification` enum (`verified`, `gateway_derived`, `trusted_producer_asserted`, `untrusted_caller_asserted`, `configuration_derived`, `unavailable`) and `basis-schemas`' closed `redaction-classification` enum — rather than inventing a second, competing one. Every fact in the "not yet implemented" rows of this table, when it is eventually produced, must be classified using one of these existing values; this document does not propose a seventh provenance category or a sixth redaction tier. If protocol dispatch result and device-state confirmation genuinely need a provenance value none of the six existing ones fit, that is itself a finding for the architecture process — not something to resolve by extension in this document.

---

## 8. Failure and Degraded Conditions

| Condition | Distinguishing invariant |
| - | - |
| Normalization failure | `AdapterResult.success is False`; `basis-adapters`' existing rule: treat as deny, do not forward |
| Producer authentication failure | Not executed — the request never reaches a classified producer state |
| Producer not trusted | Not executed — `basis-gateway`'s existing `UNTRUSTED` classification and safe-default-empty-allowlist behavior apply unchanged |
| Producer-only context rejected | Not executed — `basis-gateway`'s existing `UntrustedOperationProducerContextError`, raised before any other composition step |
| Gateway unavailable | Execution status unknown to the caller, but not executed by definition — no disposition was ever received |
| Evaluation failure | Authorization evaluation failed — distinct from a deny; `evaluation_status: failed` with a governed `failure_reason`, never a silent allow |
| Deny | Authorization denied — not executed |
| Not applicable | Authorization denied at the enforcement boundary today (`basis-gateway` collapses `not_applicable` to a 403 HTTP outcome while preserving the distinct `outcome` value in the body); not executed |
| Timeout before protocol dispatch | Not executed — the operation-producer runtime or protocol executor must fail closed rather than assume permission expired into an allow |
| Timeout after dispatch with unknown remote result | Execution status unknown — distinct from execution failed; the operation was attempted and its outcome is unconfirmed, not known-negative |
| Local executor failure | Execution failed — the executor itself could not complete dispatch, independent of anything the remote endpoint did or didn't do |
| Remote protocol rejection | Execution failed — the remote endpoint explicitly rejected the operation; distinct from a missing acknowledgement |
| Missing acknowledgement | Execution status unknown — the protocol offers no confirmation primitive, or none arrived in time |
| State verification unavailable | Execution status unknown at the device-state layer specifically — dispatch may be confirmed while the resulting state change is not, and these are separate facts (compare KNX's DPT shape checks, which validate a write's format, against confirming the physical actuator actually moved) |
| Execution evidence write failure | A new failure mode with no analogue among the above — the operation may have executed, failed, or remained unknown, and in addition the record of which of those happened was itself not durably written. This is structurally the same problem `basis-gateway`'s existing `gateway.operation_aware_evidence_missing` fallback event solves for missing kernel evidence, generalized to a future execution-evidence writer |

Five distinctions must be preserved across every condition above, because collapsing any pair of them destroys information the ecosystem's audit and evidence model depends on:

```text
not executed              ≠  execution failed
execution failed          ≠  execution status unknown
execution status unknown  ≠  authorization denied
authorization denied      ≠  authorization evaluation failed
not executed               ⊇  {producer auth failure, producer not trusted,
                                producer-only context rejected, deny,
                                not-applicable, evaluation failure,
                                timeout before dispatch}
```

The last line is deliberate: "not executed" is not itself one condition but a family of conditions with different causes upstream of execution, all of which share the property that no protocol dispatch occurred. Preserving the specific cause (deny vs. evaluation failure vs. producer rejected) alongside the shared "not executed" fact is exactly the distinction `basis-schemas`' `operation-aware-decision-response` contract already enforces structurally between `outcome` and `evaluation_status`/`failure_reason` — a design pattern the eventual execution-evidence contract should very likely reuse rather than reinvent, without this document deciding that it must.

---

## 9. Deployment Topologies

These are logical roles (§2), not a deployment prescription. Topology changes where a role's obligations are enforced; it does not change what the obligations are, exactly as `basis-adapters`' own architecture already states for its two existing integration models ("deployment topology must not alter authorization semantics").

### Combined producer/executor process

A single authenticated runtime normalizes (via the adapter library), calls the gateway for authorization, and — if permitted — executes against the protocol endpoint, all in one process. This is the closest analogue to `basis-adapters`' existing "embedded model," extended one step further to include execution. What must be protected: the authenticated credential this single process uses toward the gateway (its compromise grants both producer trust and execution capability in one place), and the boundary inside the process between "received disposition" and "began dispatch," which must be enforced by that process's own code discipline since no separate component enforces it externally.

### Separated producer and executor

A producer runtime submits for authorization and, on an `allow`, passes an authorization-bound command to a separate executor process — potentially on different hardware, at a different network position. What must be protected: the channel between producer and executor must itself carry enough of the disposition and correlation identifiers that the executor can refuse to act on a stale, replayed, or unrelated command; a compromised or confused executor that accepts commands without checking their disposition binding reintroduces exactly the "confused deputy" abuse case the interoperability roadmap already names.

### Embedded or constrained deployment

Normalization, an authorization client, and execution all run close to an isolated OT endpoint — for example, on a single-board controller with no separate gateway reachable, mirroring `basis-adapters`' existing "embedded model" rationale (air-gapped deployments, strict resource constraints). What must be protected: in this topology the constrained device itself typically holds the producer credential, the evaluation client, and the protocol stack, which concentrates risk that a networked topology would otherwise distribute — cryptographic protection of the credential material at rest becomes proportionally more important the more roles a single constrained device combines.

Topology does not change authorization semantics in any of the three: a request that would be denied in the separated topology must be denied in the combined topology, for the same reason `basis-adapters`' existing design invariant #10 already requires this across its own two integration models.

---

## 10. Repository Ownership

```text
basis-adapters
    owns normalization libraries and protocol evidence construction
    (unchanged by this document)

basis-gateway
    owns producer authentication, trust classification, request admission,
    kernel invocation, and boundary enforcement
    (unchanged by this document)

basis-core
    owns authorization semantics only
    (unchanged by this document)

basis-identity
    owns canonical identity and identity evidence
    (unchanged by this document)

basis-schemas
    owns shared machine-readable contracts only after the architecture
    is stable
    (unchanged by this document; see §12 for what remains an inventory,
    not a schema proposal)

basis-deploy
    will package and configure runtimes but will not own their semantics
    (unchanged by this document; no repository exists yet)
```

Two additional statements this document makes explicit, because §2's new logical roles could otherwise be misread as license to expand an existing repository's scope:

- `basis-adapters` does not automatically expand into a daemon, gateway, or device controller merely because the ecosystem now names an operation-producer runtime and a protocol executor. Those roles remain either undecided (§11) or explicitly out of `basis-adapters`' scope per its own non-responsibilities.
- `basis-deploy`, once it exists, packages and configures whatever runtime performs the operation-producer or protocol-executor role; it does not itself perform protocol dispatch or own execution semantics. Live protocol execution is not assigned to `basis-deploy` by this document.

---

## 11. Alternatives and Repository Decision

Four alternatives for where the operation-producer runtime (and, by extension, the protocol executor) should live:

**1. Expand `basis-adapters` itself into a runtime host.** Would let one repository own normalization through execution. Rejected as the default direction: it directly contradicts `basis-adapters`' own, currently released, non-negotiable non-responsibilities ("no live protocol communication... adapters are libraries, not daemons") and its ten design invariants, several of which (protocol logic terminating at the adapter, no parallel enforcement paths) exist specifically to keep the library free of exactly the runtime, credential-holding, network-facing responsibilities a producer runtime requires. Reopening that boundary would also undermine the embedded-model guarantee that adapter behavior is portable and side-effect-free regardless of deployment topology.

**2. Define a separate logical adapter-host or producer-runtime component.** The direction this document's own role definitions in §2 point toward. Preserves `basis-adapters` as a pure library; gives the authenticated, stateful, network-facing responsibilities (holding a producer credential, calling the gateway, dispatching to a protocol executor) a home that is architecturally allowed to have them. Does not, by itself, require a new repository — see the decision gate below.

**3. Allow protocol-specific integrations to provide their own conforming runtime hosts.** Analogous to how `basis-adapters` already supports nine independent protocol adapters under one shared contract. Would let a BACnet-heavy deployment and an MQTT-heavy deployment each ship a runtime shaped for their operational constraints, as long as both conform to this document's trust and lifecycle invariants. Plausible as a long-term end state; premature to commit to now, since no reference implementation of even one conforming runtime exists yet to validate that the shared contract is sufficient.

**4. Embed adapters directly inside `basis-gateway`.** Would eliminate the handoff gap by eliminating the boundary. Rejected: it directly contradicts `basis-gateway`'s own stated existence rationale ("`basis-gateway` exists because `basis-core` is a library, not a network service" — its scope is authentication, composition, and enforcement, not protocol translation) and would reintroduce the exact protocol-specific logic into a shared, multi-tenant-facing component that the adapter/gateway split exists to keep out.

**Likely direction, held as a decision gate rather than a conclusion:** preserve `basis-adapters` as the normalization library, unchanged, and define a separate operation-producer runtime role (alternative 2). Whether that role requires a *new* repository — as opposed to a reference implementation living inside `basis-gateway`'s own repository as an optional client, or inside a future `basis-deploy`-packaged component — is not decided here. This document treats "does the operation-producer runtime need its own repository" as an explicit, later decision gate (§13, Stage 5), to be resolved once a bounded reference implementation exists to learn from, consistent with `ROADMAP.md`'s own stated principle that "`basis-adapters` and `basis-identity` should be changed based on real integration needs that gateway work surfaces, not speculative taxonomy work performed ahead of it."

---

## 12. Schema Impact Assessment

`basis-schemas` v0.2.2 publishes twenty contracts, all at `experimental` lifecycle. This section inventories what they can already represent for the request-handoff and execution-evidence gap, and what they cannot, without proposing new schema work — consistent with `ecosystem-contract-inventory.md`'s own convention of recording open questions rather than resolving them.

### Already representable

| Concept | Contract | Sufficiency |
| - | - | - |
| Normalized operation and its protocol evidence, referenced safely | `adapter-evidence-reference` (v0.1.0) | Sufficient as designed: digest-based reference, closed `redaction-classification`, explicit rejection of raw-payload-shaped fields. No new schema needed for a producer runtime to reference adapter evidence. |
| Operation-aware request shape carrying producer-only fields | `operation-aware-decision-request` (v0.1.0) | Sufficient as a shape; the contract itself states it "does not assemble requests... does not retrieve evidence" — assembly is the operation-producer runtime's job, not a schema gap. |
| Structural separation of outcome from failure | `operation-aware-decision-response` (v0.1.0) | Directly reusable as a design pattern for a future execution-evidence contract (§8's final paragraph). |
| Identity provenance reference shape | `identity-evidence-reference` (v0.1.0) | Sufficient as designed, but currently unproduced (see gap below). |

### Gaps

| Missing concept | Producer | Consumer | Reason current contracts are insufficient | Canonical scenario that would validate it |
| - | - | - | - | - |
| Execution-evidence record (protocol dispatch attempted/rejected/unacknowledged/confirmed) | Protocol executor / execution-evidence producer | `basis-gateway` (for correlation), `basis-console` (for operator display), a future `basis-activity` (per the post-authentication roadmap) | `gateway-audit-event.yaml`'s own header explicitly disclaims this: "enforcement-result / execution-success semantics are not defined by this PR and remain deferred to future architecture work." No contract in the current twenty defines dispatch or device-state outcomes. | An `allow` disposition on a BACnet `WriteProperty` operation, dispatched by a protocol executor, where the device returns a `BACnet-Error-PDU` — the ecosystem currently has no governed way to record that the write was attempted and rejected, only that it was authorized. |
| Producer-to-gateway authentication/trust assertion contract | Operation-producer runtime | `basis-gateway` | No contract exists because no authentication mechanism has been selected (§3, §11); `OPERATION_PRODUCER_SUBJECT_IDS` is gateway-internal configuration, not a schema. | A deployment onboarding a second, independently-operated producer runtime with no shared code to the first — today this can only be configured, not described in an interoperable, machine-readable way. |
| Category-scoped producer capability (trusted for `protocol_context` but not `safety_context`) | `basis-gateway` (would need to consult it) | `basis-gateway` | Today's all-or-nothing `OPERATION_PRODUCER_SUBJECT_IDS` model has no per-field capability concept; adding one is a schema and gateway-configuration question together, not a schema-alone question. | A deployment where a general-purpose integration runtime should be trusted for ordinary read/write context but not for asserting `safety_context`, which only a dedicated safety-system integration should be allowed to assert. |
| Workload/machine identity as a first-class identity concept | `basis-identity` | `basis-gateway`, future producer-authentication mechanism | `basis-identity`'s `CanonicalSubject` has a `SubjectType.SERVICE` value but no workload-identity establishment pipeline (no client-credentials or mTLS/SPIFFE-shaped `AuthenticationProtocol` member) and no glossary entry for "workload identity." This is a gap in `basis-identity`, not in `basis-schemas`, but it constrains what a future producer-authentication schema can assume already exists. | An operation-producer runtime that needs to present a workload identity to `basis-gateway` distinct from any human operator's identity, with its own lifecycle (issuance, rotation, revocation) — no component in the ecosystem currently issues or verifies this. |
| `basis-identity` production of `identity-evidence-reference` | `basis-identity` | `basis-gateway`, `basis-core`, `basis-console` | The contract is published and `basis-gateway`'s schema already accepts it, but `basis-identity` neither produces nor references it anywhere in its current source — an alignment gap between a published contract and its intended producer, not a missing contract. | An operation-aware request where the initiating human's identity evidence (not just the operation producer's workload identity) needs to be carried alongside the operation for audit purposes. |

This document does not conclude that a new schema is required for any of the above merely because a gap can be named. Per `ecosystem-contract-inventory.md`'s own governing principle, a contract becomes schema-ready when implementation proves a stable shape — none of the five gaps above have a reference implementation yet to prove one.

---

## 13. Recommended Implementation Sequence

1. **Architecture approval** of this document, its role definitions (§2), and its trust rules (§3) — a decision gate in `basis-architecture`, per `GOVERNANCE.md`'s ADR process for changes that affect component boundaries.
2. **`basis-adapters` operation-aware handoff alignment plan** — a `basis-adapters`-owned planning document assessing what, if anything, its own public surface (`AdapterContext`, `AdapterResult`, per-protocol evidence shapes) needs in order to make constructing an `AdapterEvidenceReference` straightforward for whatever consumes its output. Additive only; does not change `basis-adapters`' non-network, non-authenticating contract.
   **Decision gate:** does `basis-adapters` need any additive surface at all, or is its current public API already sufficient for a producer runtime to consume? **Resolved:** mixed, by field — see that plan and the architecture decision in Stage 2a below.
2a. **Adapter evidence construction and canonicalization architecture** — [`docs/architecture/adapter-evidence-construction-semantics.md`](adapter-evidence-construction-semantics.md), adopted by [ADR-0007](../adr/0007-adapter-evidence-construction.md), answers the construction questions Stage 2's plan could not resolve on its own authority: what constitutes adapter evidence material, how it is canonicalized, which component computes its digest, which component mints `reference_id`, and which component assembles the final `AdapterEvidenceReference`. It fixes a first evidence profile (`basis-adapter-evidence-v1`) canonicalized exactly under RFC 8785 (`rfc8785`) — not a candidate awaiting a future technology evaluation — and assigns material construction, canonicalization, and digest computation to `basis-adapters` through a pure deterministic helper, with reference assembly (`reference_id`, `adapter_source`, `redaction_classification`, request/correlation linkage) to the operation-producer runtime named in §2 above. It does not implement any of this and does not modify `basis-schemas`.
3. **Additive adapter-side evidence/reference construction, only if required** by Stage 2's plan and specified by Stage 2a's architecture. Not scheduled here as a certainty.
4. **Gateway producer-authentication and admission refinement** — `basis-gateway`-owned work to move beyond the subject-ID allowlist toward whatever authentication mechanism the architecture eventually selects for the operation-producer runtime, and to resolve or explicitly defer the category-scoped capability question from §3 and §12.
   **Decision gate:** which authentication/transport mechanism (mTLS, SPIFFE, OAuth client-credentials, or another) — explicitly not selected by this document. **Resolved, proposed only:** [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) selects mutual TLS as the first normative producer-to-gateway workload-authentication profile, distinguishes authentication from admission, and authorizes (without implementing) a bounded producer reference slice. ADR-0008 is recorded `Proposed`; no implementation of mTLS, gateway admission configuration, or the reference slice exists yet, and permanent operation-producer-runtime repository placement remains the deferred decision gate this document's §11 and Stage 5 below already describe.
5. **Bounded producer-runtime reference implementation** — a first, narrow implementation of the operation-producer runtime role (§2), scoped to one protocol and one deployment topology (§9), used to validate that the roles and rules this document defines are actually sufficient before generalizing them.
   **Decision gate:** does this reference implementation live in an existing repository (a `basis-gateway` client library, for instance) or does it justify a new repository, per §11's deferred decision?
6. **Execution-evidence contract publication, only after producer and consumer are stable** — `basis-schemas` work, gated on Stage 5 having produced a real protocol executor and execution-evidence producer whose output shape can inform the contract, per `ecosystem-contract-inventory.md`'s own "implementation proves a stable shape" principle.
   **Decision gate:** is the five-state sketch from the interoperability roadmap's Phase 4 sufficient, or does the reference implementation surface additional states this document did not anticipate?
7. **Gateway/executor correlation tests** — conformance tests validating that the correlation identifiers named in §6 remain linkable end-to-end through a real producer runtime, gateway, and protocol executor.
8. **Deployment packaging** — `basis-deploy` work, once that repository exists, to package whatever combination of roles Stage 5 validated, without `basis-deploy` acquiring semantic ownership of any of them (§10).
9. **End-to-end demonstration** — analogous to `basis-gateway`'s existing `demo/operation-aware/` bounded demonstration, extended to include a real (or faithfully simulated) protocol dispatch and execution-evidence record.
10. **Real or representative protocol validation** — validation against an actual OT protocol endpoint or a high-fidelity simulator, closing the loop `ROADMAP.md` already names as "real OT integration validation," the final stage of the existing Downstream Rollout Sequence.

This sequence deliberately does not assign PR counts, milestones, or delivery dates to any stage, consistent with `ROADMAP.md`'s own convention that such detail belongs to each repository's own implementation plan once that stage of work actually begins.

---

## Non-Goals

This document does not:

- modify `basis-adapters`, `basis-gateway`, `basis-schemas`, `basis-identity`, `basis-core`, or any deployment component;
- add or modify a schema;
- add Python code or any other implementation;
- add network communication;
- implement a protocol stack;
- create an adapter daemon;
- create a new repository;
- implement mTLS, SPIFFE, OAuth client-credentials, or any other producer-authentication mechanism;
- modify the gateway's `OPERATION_PRODUCER_SUBJECT_IDS` allowlist or its behavior;
- add an execution-status vocabulary as a governed contract;
- implement an audit ledger;
- add topology discovery;
- modify `basis-console`;
- begin `basis-deploy`;
- design a broad commercial control plane;
- claim production readiness for any component named in this document.

---

## Open Questions Deferred

- Whether the operation-producer runtime requires a new repository, or can live inside an existing one as a reference client (§11, §13 Stage 5).
- Which authentication and transport mechanism establishes producer workload identity toward `basis-gateway` beyond the current subject-ID allowlist (§3, §13 Stage 4) — [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) proposes mutual TLS as the first normative profile; the ADR remains `Proposed` and no implementation exists yet, so this remains open in practice until acceptance and implementation both occur.
- Whether producer trust should become category-scoped (trusted for some producer-only context fields but not others) rather than all-or-nothing (§3, §12).
- Whether `basis-identity` should introduce an explicit workload/machine identity concept, and if so, what authentication protocols it should recognize beyond its current OIDC/SAML/OAuth2/LDAP/local-dev set (§12).
- Whether and how `basis-identity` should begin producing the already-published `identity-evidence-reference` contract it does not yet produce (§12).
- Whether the interoperability roadmap's five-state execution-status sketch is the right starting vocabulary for a future execution-evidence contract, or whether a bounded reference implementation will surface different states (§13 Stage 6).
- What provenance classification a future protocol-dispatch or device-state fact should carry if none of the six existing `ProvenanceClassification` values fit (§7).
- How a future execution-evidence contract should handle protocols (like DNP3's stateless select-before-operate model) that structurally lack a persistent state to confirm against, versus protocols with genuine read-back confirmation (§8).

Naming these here is intentional, for the same reason every other operation-aware architecture document in this repository names what it defers: it establishes that this document is aware of what remains open and is not silently deciding those questions by omission.

---

## Relationship to Other Documents

This document narrows the long-term, ten-phase `docs/roadmaps/identity-to-operation-contract-and-interoperability.md` into the first concrete boundary named in `ROADMAP.md`'s **Next Producer and Execution-Evidence Boundary** section — it does not replace that roadmap's broader architectural invariants (Kernel, Identity, Gateway, Adapter, Execution, Schema, Security, Open-contract, Console) or its Contract Responsibility Model, both of which this document assumes and builds on rather than restates in full. It is a prerequisite, not yet satisfied, for `docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`'s own dependency chain ("identity-to-operation contract → trustworthy evidence → durable normalized activity"), which explicitly cannot proceed until the evidence and execution-lifecycle architecture this document begins to define is stable.

It restates, without altering, `basis-adapters`' existing normalization-only contract (`basis-adapters.md`), `basis-gateway`'s existing authentication, composition, and enforcement contract (`basis-gateway.md`), and the trusted-adapter boundary already established in the threat model (`docs/security/threat-model.md` §3.3). It uses, without redefining, the operation-aware evidence and provenance vocabulary established in `operation-aware-trace-audit-evidence.md` and narrowed by `operation-aware-evidence-provenance-semantics.md`. Where this document introduces working vocabulary not yet in `docs/glossary.md` — *operation-producer runtime*, *protocol executor*, *execution-evidence producer* — it follows the same disclaimer convention both existing roadmap documents already use: naming a term here does not promote it to canonical status, and any of these terms may be renamed, merged, or dropped as the decision gates in §11 and §13 resolve.
