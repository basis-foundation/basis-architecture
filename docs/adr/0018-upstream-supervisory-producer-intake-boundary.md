# ADR-0018: Upstream Supervisory-System Producer-Intake Boundary

## Status

Accepted

## Context

[`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §2 defines the *operation initiator* as "the human, machine, workload, service, automation rule, or upstream system requesting an operation," and states that an initiator's identity "may or may not be the same as the operation producer's authenticated workload identity." [ADR-0008](0008-producer-workload-authentication-and-admission.md) narrows that distinction for the producer-to-gateway hop: the operation producer's workload identity and the authorization subject's identity are structurally separate, and "a producer does not become authoritative for subject identity merely because its mTLS connection is trusted." [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) establishes `basis-producer` as the operation-producer runtime, and [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) through [ADR-0016](0016-bounded-target-replay-freshness-posture.md) define the protocol-executor role, authorization-to-execution binding, execution lifecycle, execution evidence, the first bounded execution target, and that target's replay/freshness posture.

That architecture fully specifies what happens from the moment the operation-producer role holds an operation — adapter normalization, evidence retention, authenticated gateway submission, binding, dispatch, and evidence. It does not specify how the operation-producer role *receives* an operation request from a system outside BASIS. Every accepted document to date either begins at a `ProtocolOperation` already in the producer's hands or treats the initiator as a label carried alongside the request. None defines the boundary an external initiator's request crosses to reach the producer, what BASIS must establish before accepting that request, or what the initiator does and does not gain by submitting it.

That gap has become concrete. A supervisory platform — an application that owns operator workflows, asset models, and supervisory intent, and that wants governed operations performed through BASIS rather than directly against OT targets — has made a corresponding architectural decision on its own side of the boundary. That platform's decision, recorded in its own architecture repository and tracked there as cross-ecosystem gap **ARCH-GAP-001**, classifies it as an upstream supervisory platform and operation initiator. It explicitly does not classify it as a BASIS operation producer or a BASIS protocol executor. The platform is Ipotio. It is referenced here by name, not by hyperlink, per this repository's cross-repository citation convention, and only as the motivating integration. That decision is necessary but not sufficient. It says what the upstream platform will not claim. It cannot say what BASIS will accept, because only BASIS architecture can govern BASIS's own boundary. ARCH-GAP-001 therefore cannot close from the upstream side alone.

Without a BASIS-side decision, the path of least resistance is to collapse the boundary. One way is to admit the upstream platform as a gateway-trusted operation producer directly. Another is to embed it as a trusted in-process caller of the producer runtime. A third is to let it address the protocol executor. Each alternative is analyzed under **Alternatives Considered** below. Each would make supervisory intent into producer trust or execution authority by integration convenience rather than by architecture.

Consistent with this repository's ADR-acceptance convention (each of ADR-0007 through ADR-0016 was first merged `Proposed` and accepted by a separate, dedicated PR — see [`README.md`](README.md#lifecycle-states)), this ADR was submitted as `Proposed`, and its merging did not constitute acceptance. It has since undergone that same separate, dedicated formal-acceptance review, and its status now records `Accepted`. Acceptance changes no part of the decision below and authorizes no implementation.

## Problem

How does BASIS accept a governed operation request from an upstream supervisory system, and turn that admitted request into BASIS-owned operation-producer behavior, without:

- making the upstream system a BASIS operation producer, protocol executor, authorization authority, or execution authority;
- conflating the upstream system's workload identity with either the authorization subject or the operation producer's workload identity;
- letting an unauthenticated, unverifiable, malformed, or stale request become an authorized or executed operation;
- binding BASIS architecture to any one supervisory platform?

## Decision

**BASIS recognizes an explicit, governed *producer-intake boundary* on the ingress side of the operation-producer role. An upstream supervisory system may submit supervisory intent into BASIS only across that boundary. Crossing it confers no producer, authorization, execution, or credential authority on the upstream system. What emerges from the boundary is an admitted intake request. It is not an operation, not an authorization, and not a command: the operation-producer role alone turns an admitted intake request into a governed operation. That operation then proceeds through the existing, unchanged authorization and execution path.**

```text
upstream supervisory system          (outside BASIS; owns supervisory intent)
        │  intake request
        ▼
producer-intake boundary             (BASIS-governed; admit or reject, fail closed)
        │  admitted intake request
        ▼
operation-producer role              (BASIS; produces the governed operation — ADR-0010)
        │  authenticated operation-aware submission (ADR-0008, ADR-0009)
        ▼
basis-gateway → basis-core           (authorization and enforcement — unchanged)
        │  authoritative disposition + authorization-to-execution binding (ADR-0012)
        ▼
protocol-executor role               (BASIS; bounded dispatch — ADR-0011, ADR-0013)
        │
        ▼
OT target
```

Every arrow in this diagram is a separate boundary with its own admission or verification rule. None of them inherits trust from the one above it. The diagram shows logical roles, not a deployment prescription. Per `operation-producer-and-execution-boundary.md` §9, the intake boundary and the operation-producer role may share a process, but sharing a process does not merge their obligations. The same rule governs the producer/executor colocation in ADR-0011 Decision 2.

### 1. Upstream supervisory system

An **upstream supervisory system** is a software system outside BASIS that originates supervisory intent and requests that BASIS perform a governed operation on its behalf. It is a specific kind of *operation initiator* as defined in `operation-producer-and-execution-boundary.md` §2: a workload-type initiator that submits requests into BASIS through the producer-intake boundary. The category is generic. A building-management supervisory platform, an orchestration or scheduling system, a maintenance-workflow system, or another BASIS-integrating application may each occupy it. Ipotio is one valid instance and receives no special treatment.

Across the intake boundary, an upstream supervisory system **may**:

- originate supervisory intent;
- select or reference a requested operation;
- identify or reference an intended target or resource;
- carry subject-related authorization context, where a governed conveyance permits it (Decision 3);
- carry request-scoped contextual facts, where a governed conveyance permits it (Decision 4);
- attach the provenance needed for BASIS to interpret the request downstream;
- submit the request and receive the intake outcome (Decision 6).

Submitting a request does **not**, by itself or in combination with any upstream-side configuration:

- make the upstream system a BASIS operation producer, or give it producer workload credentials or gateway producer admission (ADR-0008, ADR-0009);
- make it a protocol executor, or give it any path to the protocol-executor role that bypasses the operation producer, `basis-gateway`, and the ADR-0012 binding;
- give it protocol or device credentials (ADR-0011 Decision 14);
- give it authorization authority, or let it reinterpret, pre-empt, or override a `basis-core` outcome or a `basis-gateway` disposition;
- let it bypass the producer/executor lifecycle ADR-0011 through ADR-0014 define;
- let it directly actuate protocol or device behavior through this architectural relationship.

The upstream supervisory system keeps ownership of its own application semantics: its operator workflows, asset model, supervisory state, and whatever it decides to request. BASIS does not absorb them. The upstream system likewise does not absorb any BASIS role.

### 2. The producer-intake boundary

The **producer-intake boundary** is the trust and responsibility transition between *upstream supervisory intent*, which is external and upstream-owned, and *BASIS-governed operation-producer behavior*. It is a governed transition. The upstream system is never treated as a trusted in-process producer, and its request is never treated as already admitted because of where it came from.

The boundary belongs to the BASIS side. It is the ingress responsibility of the operation-producer role. Before the operation-producer role acts on an intake request, the boundary must establish all of the following. If any one of them cannot be established, the request is rejected (Decision 6):

| Requirement | What the boundary must establish |
| - | - |
| **Admission identity** | Which upstream workload submitted the request: an authenticated *upstream workload identity* or equivalent admission identity, not a self-asserted name in the request body. Network location alone is insufficient, as in `operation-producer-and-execution-boundary.md` §3. |
| **Authorization to use the intake** | That this authenticated upstream workload is explicitly permitted to submit intake requests. The grant is deployment-owned and explicit, and never inferred from role membership, attribute values, or reachability. This mirrors how producer admission at the gateway binds an authenticated identity to an explicit grant (ADR-0008 "Authentication vs. admission"). |
| **Request integrity** | That the request the operation-producer role acts on is the request the upstream workload submitted, unaltered in transit and between admission and production. |
| **Correlation and traceability** | That the request carries, or is assigned at admission, identity sufficient to correlate it with everything BASIS later produces for it (Decision 5). |
| **Provenance** | That every fact in the request is attributable to its origin, and that the origin survives the handoff (Decision 5). |
| **Bounded validity** | That the request is still eligible for processing (Decision 7). |
| **Well-formedness** | That the request has a supported shape and is internally consistent. Malformed, unsupported, contradictory, or unverifiable requests are rejected, not repaired or guessed at. |
| **Explicit outcome** | That every intake request resolves to an explicit, attributable admission outcome (Decision 6). |

This ADR fixes these semantics. It does **not** select a transport or mechanism: no HTTP endpoint, RPC, queue, message bus, file drop, or embedding API is prescribed. No accepted architecture requires one, and the semantics above must hold under whichever mechanism a later decision selects. The same boundary may be realized differently in a networked deployment and in a constrained or air-gapped one, as long as every requirement above still holds.

**Intake admission is not gateway admission, and neither substitutes for the other.** Gateway producer admission (ADR-0008, ADR-0009) answers whether a *producer workload* may submit operation-aware requests to `basis-gateway`. Intake admission answers whether an *upstream workload* may submit supervisory intent to the operation-producer role. A request admitted at intake still passes gateway producer admission, authorization-subject authentication, and `basis-core` evaluation in full. Being admitted at one boundary never counts as being admitted at the other.

**Intake admission is not authorization.** An admitted intake request has been accepted for *production*. It has not been evaluated, decided, or permitted for execution. `basis-core`, through `basis-gateway`, remains the only source of the authorization outcome.

### 3. Subject identity, upstream workload identity, and producer workload identity remain distinct

Three identities meet at the intake boundary. They stay distinct even when, in a given deployment, two of them happen to name the same underlying entity:

| Identity | Meaning | Established by |
| - | - | - |
| **Authorization subject** | The identity whose authority `basis-core` evaluates, which may be human or non-human (ADR-0008 "Producer vs. authorization subject"). | The existing identity and authentication chain for that subject (today `basis-gateway`'s `authenticate()` dispatch, and potentially a `basis-identity` canonical identity context). Never established by the upstream workload's own authentication. |
| **Upstream workload identity** | The upstream supervisory system's workload submitting intent into BASIS. | Intake admission (Decision 2). |
| **Producer workload identity** | The BASIS operation-producer role's own workload, which produces and submits the governed operation. | ADR-0008 mTLS profile and gateway admission, unchanged. |

The following rules apply:

- **The upstream workload is not the subject.** An upstream system may submit a request *on behalf of*, or *associated with*, a human or non-human subject. Being authenticated as the upstream workload does not authenticate that subject. Authority evaluated for the subject is never inferred from the upstream workload's admission.
- **The producer is not the subject.** The producer workload identity must never silently substitute for the authorization subject. This restates ADR-0008 and ADR-0010's "never deriving one from the other," extended to requests that arrive through intake.
- **The upstream workload is not the producer.** The upstream workload identity never becomes, is never mapped onto, and never borrows the producer workload identity. The producer's gateway-facing credential is not exposed or delegated across the intake boundary.
- **Subject context crossing intake keeps its provenance.** Where a request conveys subject-related context (a subject reference, a subject credential, or an identity assertion), the intake boundary preserves which upstream workload conveyed it and in what form. Context that is not cryptographically or contractually bound to a verifiable subject authentication stays exactly what `operation-producer-and-execution-boundary.md` §7 already calls an unverified hint. It is never upgraded because it arrived through an admitted upstream workload.

How a subject credential is conveyed from an upstream system to the operation-producer role is not decided here. ADR-0010 already leaves open "*how* `basis-producer` obtains" the authorization-subject credential. This ADR adds only the constraint that any such mechanism must satisfy the rules above when the credential arrives through intake. Producer credential lifecycle is likewise not decided here (see **Deferred Decisions**).

### 4. Request-scoped context crossing intake

An upstream supervisory system may carry request-scoped contextual facts into BASIS. This ADR decides only that such facts **keep their upstream origin** across the boundary. The intake boundary and the operation-producer role must not erase, relabel, or launder that origin.

This ADR does **not** decide which context categories an upstream system, or an operation producer relaying upstream-originated context, may authoritatively assert. That is the category-scoped trust question `operation-producer-and-execution-boundary.md` §3 already names as open.

Until that question is decided, the following fail-closed interim posture applies. The operation-producer role must not present upstream-originated context to `basis-gateway` as producer-only context merely because an admitted upstream system supplied it. Doing so would cause the gateway to classify the fact `trusted_producer_asserted` when it was actually asserted upstream, and that would erase exactly the provenance this Decision requires. A producer may still assert producer-only context it independently has a basis to assert, under existing rules. This posture withholds trust. It does not grant any, so it pre-decides nothing the later category-scoped trust decision might conclude.

### 5. Request integrity, provenance, and correlation

A request that crosses the intake boundary must carry, or be assigned at admission, enough integrity and provenance for BASIS to determine:

- who or what submitted it (the upstream workload identity);
- which subject it concerns, if any, and in what form that subject was conveyed (Decision 3);
- what operation was requested;
- what target or resource was intended;
- which contextual facts originated upstream (Decision 4);
- what correlation identity follows the request;
- whether the request is still eligible for processing (Decision 7).

"What operation was requested" and "what target was intended" mean exactly what the upstream system referenced. How those references map to BASIS actions, resources, and a protocol-shaped operation is not decided here. Whatever mapping a later decision defines, the operation-producer role must still produce the operation through the existing chain: `basis-adapters` normalization, retained evidence (ADR-0007), the ADR-0012 binding over the preserved original operation (ADR-0011 Decision 7), and authenticated gateway submission. Intake introduces no alternate path around that chain.

**Correlation follows the existing ownership rule.** `operation-producer-and-execution-boundary.md` §6 and ADR-0014 require that "no component may overwrite an identifier owned by another component." An upstream request identifier is an upstream-owned identifier. It is carried and linked, never adopted as, or substituted for, a BASIS-owned identifier. It does not replace the operation producer's own `request_id`. It never becomes the gateway-generated `correlation_id`, since the gateway already ignores caller-supplied correlation headers by explicit policy. It does not replace the ADR-0012 binding identifier or any kernel or execution-evidence identifier. The chain gains one upstream link. No existing link changes owner.

**Correlation is not authenticity.** ADR-0014's caution applies unchanged: an upstream correlation identifier establishes deterministic linkage, not proof that the upstream system's claims are true.

This ADR does not define a signing protocol, message-authentication scheme, or envelope format. The requirement is that integrity and provenance be *established*. The mechanism that establishes them is deferred to whatever decision selects the intake transport.

### 6. Admission outcomes and fail-closed rejection

Every intake request resolves to exactly one explicit outcome:

- **Admitted.** Every Decision 2 requirement was established. The request may be handed to the operation-producer role. This outcome says nothing about authorization or execution.
- **Rejected.** A Decision 2 requirement was affirmatively not met.
- **Failed.** Admission could not be determined, for example because an intake dependency was unavailable. This outcome is treated exactly like rejection for every downstream purpose.

Admission can fail before any BASIS operation exists. Conditions include, without being limited to:

- unauthenticated upstream workload;
- upstream workload not authorized to use the intake;
- malformed, contradictory, or unsupported request shape;
- unverifiable provenance or integrity;
- invalid or unverifiable subject-context conveyance;
- expired, replay-ineligible, or otherwise ineligible request (Decision 7);
- intake dependency unavailable.

For every rejected or failed intake:

- **it does not become an authorized operation**, because the operation-producer role does not produce an operation from it;
- **it does not proceed to execution**, because no gateway submission, no ADR-0012 binding record, and no protocol-executor invocation occurs;
- **silence is not admission.** An upstream system that receives no outcome must not treat the request as admitted, and BASIS must not treat an undetermined admission as admitted.

A rejected or failed intake belongs to the "not executed" family in `operation-producer-and-execution-boundary.md` §8, with a cause that comes before every existing member of that family. It occurs before ADR-0013's Case A and produces no execution-evidence record under ADR-0014, which already excludes causes upstream of the protocol-executor role. The intake outcome must still be attributable and recorded in a form that preserves which Decision 2 requirement failed. This ADR does not define error codes, a status vocabulary, or a record schema for that purpose. None exists in accepted architecture, and none is invented here.

An upstream system may later learn BASIS-owned outcomes for an admitted request, such as the gateway disposition or execution lifecycle facts. Any such return path must report those facts as BASIS produced them, without altering or re-deciding them. The return-path contract is not defined here.

### 7. Bounded validity

An intake request is not an indefinitely reusable authorization or execution request. Admission applies to one request, for one production cycle. It is not a standing grant: admitting one request confers nothing on a resubmission, a duplicate, or any later request, and each of those must be admitted on its own.

Intake therefore requires **bounded validity**. That may eventually be expressed through request lifetime, freshness, nonce or replay handling, binding to authorization or execution state, correlation identity, or a combination of these. This ADR selects none of them. It does not reopen or extend the broader authorization replay/freshness question that ADR-0012 and ADR-0013 reserve and ADR-0016 leaves open globally, and it does not bear on operational duplicate-execution or idempotency.

This ADR fixes only the posture: **a stale, invalid, replay-ineligible, or otherwise non-admissible intake request fails closed under Decision 6**, according to whichever adjacent or future lifecycle rule governs eligibility. Until such a rule exists for a given intake realization, that realization cannot establish Decision 2's bounded-validity requirement and must not admit requests.

### 8. Role separation is preserved

| Role | Owns | Does not own |
| - | - | - |
| **Upstream supervisory system** | Supervisory intent and its own application semantics; submission of intake requests | Producer behavior, producer credentials, gateway admission, authorization, protocol dispatch, device credentials |
| **Producer-intake boundary** (ingress of the operation-producer role) | Admission of intake requests under Decision 2; explicit outcomes; provenance and correlation preservation | Authorization; production of the governed operation; execution |
| **Operation-producer role** (ADR-0010) | Producing the governed operation from an admitted intake request through the existing chain; binding record creation (ADR-0012); authenticated gateway submission | Admitting itself at the gateway; authorization; protocol dispatch (ADR-0011 Decision 2) |
| **`basis-gateway` / `basis-core`** | Producer admission, subject authentication, composition, evaluation, and enforcement, all unchanged | Intake admission; production; execution |
| **Protocol-executor role** (ADR-0011) | Binding verification and bounded dispatch against the OT target; device credentials where required | Reinterpreting authorization; accepting commands from anything other than the governed path |

ADR-0011 Decision 14 distinguishes four credential classes: the authorization-subject credential, the producer workload credential, a possible future executor workload credential, and the protocol/device credential. Any **upstream workload authentication material or credential**, where the selected intake mechanism uses one, stays distinct from all four. It cannot be exchanged for, converted into, borrowed as, or silently substituted for any of them, and none of them can be substituted for it. This distinction is normative now. It does not define a new credential mechanism: the intake authentication mechanism, credential technology, issuance model, and lifecycle remain deferred (see **Deferred Decisions**, Workstream 3E).

**Device credentials remain outside the upstream supervisory boundary.** This ADR moves no device-credential custody and changes nothing about protocol-executor credential handling.

### 9. Platform neutrality

This ADR defines no BASIS type, field, role, or vocabulary named after, or shaped specifically for, any supervisory platform. Every requirement above is stated for the upstream-supervisory-system category. A platform-specific integration is a conforming *use* of this boundary, never a modification of it. BASIS remains application- and platform-neutral.

## Relationship to ARCH-GAP-001

ARCH-GAP-001 records a cross-ecosystem gap: the upstream platform's architecture defined what it is and is not relative to BASIS, but nothing on the BASIS side defined how BASIS receives its requests. This ADR supplies the BASIS-side half of that closure condition. With this ADR accepted, once it is reconciled with the upstream platform's already-merged decision, both ecosystems' architecture will jointly support the following statements:

- Ipotio is an upstream supervisory operation initiator (upstream decision; consistent with Decision 1 here).
- BASIS accepts upstream supervisory requests only through a governed producer-intake boundary (Decisions 2, 6, 7).
- The BASIS operation producer and protocol executor remain BASIS-side roles (Decision 8; ADR-0010, ADR-0011).
- Neither ecosystem absorbs the other's responsibilities (Decisions 1, 8, 9).

This ADR does not itself mark ARCH-GAP-001 closed. The gap is tracked in the upstream platform's repository, and marking it jointly closed is a separate reconciliation step there, after this ADR's acceptance. Workstreams 3C, 3D, and 3E (below) are follow-on work of the same cross-ecosystem effort. They build on this boundary and are not preconditions for recording it.

## Alternatives Considered

**Admit the upstream supervisory system as a BASIS operation producer.** Issue it a producer mTLS identity and add it to gateway admission (ADR-0008, ADR-0009). Rejected. The objection is not that such a system would necessarily skip producer obligations: a conforming implementation of the operation-producer role could, in principle, perform adapter normalization, evidence retention (ADR-0007), and the ADR-0012 binding correctly. The objection is that it collapses two roles this architecture deliberately keeps distinct. Upstream supervisory intent and BASIS operation-producer behavior are different responsibilities. Making the upstream platform the producer would make an external supervisory application directly responsible for BASIS producer obligations, and would transfer producer trust to it. That includes producer-only-context trust, which is currently all-or-nothing across nine fields (`operation-producer-and-execution-boundary.md` §3), and, under ADR-0011's same-process topology, colocation with the protocol-executor role. It would contradict the matched upstream-side decision that the upstream platform remains an operation initiator and requesting workload, not a BASIS producer. It would also couple each supervisory application to BASIS producer semantics, so that every such platform would have to implement and maintain them. The selected intake boundary avoids all of this: it preserves platform neutrality and lets BASIS govern producer behavior independently of whichever upstream application requests an operation.

**Treat the upstream system as a trusted in-process caller of the producer runtime.** Embed it as a library, or trust it because of co-location or network position. Rejected. Trust from reachability or co-location contradicts the zero-trust rule in `operation-producer-and-execution-boundary.md` §3 and gives no admission identity, integrity, provenance, or explicit outcome. A deployment may still *co-locate* the two, but the Decision 2 requirements apply unchanged (`operation-producer-and-execution-boundary.md` §9).

**Let the upstream system call `basis-gateway` directly and act on the disposition.** Rejected as an execution path. A disposition returned to a non-producer caller is not bound to any preserved operation (ADR-0012) and confers no execution authority on anyone. The upstream system would then need either to execute itself (rejected below) or to hand the disposition to something that would. That recreates an unbound, unattributed authorization-to-execution handoff.

**Let the upstream system address the protocol executor.** Rejected. ADR-0011 Decisions 9 and 10 and ADR-0012 require that dispatch follow only an authoritative disposition bound to the exact operation that was authorized. An upstream-to-executor path has no such binding.

**Define an Ipotio-specific intake contract.** Rejected. It would couple BASIS architecture to one application's model, violate the platform neutrality BASIS maintains across adapters, identity providers, and protocols, and force a redesign for the next supervisory platform.

**Select the intake transport and message format now.** Rejected as premature. No accepted architecture requires a particular transport. Choosing one before 3C (what a request references), 3D (what context it may carry), and 3E (how the participating workloads are credentialed) are settled would freeze a wire shape around unresolved semantics. ADR-0008 set the same order: semantics first, then the profile.

## Consequences

### Positive

- The receiving side of the upstream-to-BASIS relationship is governed architecture rather than integration convention, and the BASIS-side half of ARCH-GAP-001's closure condition exists.
- Every boundary in the chain from supervisory intent to OT target now has its own admission or verification rule. No boundary inherits trust from another.
- Subject, upstream workload, and producer workload identities stay distinct on the new boundary, extending ADR-0008's distinction rather than weakening it.
- Upstream-originated context cannot be laundered into producer-asserted trust while the category-scoped trust question is open.
- The boundary is reusable by any supervisory platform without BASIS changes.

### Negative / Tradeoff

- An upstream integration now has two admission steps before authorization: intake admission, then gateway producer admission. That adds configuration and failure surface compared with a direct collapse.
- Decision 4's interim posture means upstream-originated context cannot reach producer-only fields until the category-scoped trust question is resolved. Early integrations may therefore authorize with less context than the upstream system could supply.
- Decision 7 means no intake realization may admit requests until a governing eligibility rule exists for it. That places a bounded-validity decision on the critical path of any intake implementation.
- Intake adds one more place where outcomes, provenance, and correlation must be recorded, with no schema yet defined for them.

### Security Consequences

**Prevented by this architecture:** an upstream system gaining producer trust, gateway admission, execution capability, or device credentials by submitting requests; upstream workload identity substituting for the authorization subject (threat model §7.2, forged identity context; §7.5, privilege escalation); a rejected, malformed, or unverifiable request reaching authorization or execution; admission of one request serving as a standing grant (threat model §7.6, replay, at the level of posture, not mechanism); and upstream identifiers overwriting BASIS-owned correlation identifiers.

**Still requiring follow-on architecture:** the intake transport and its integrity and authentication mechanism; the bounded-validity rule; the upstream-to-BASIS operation/resource mapping; category-scoped context trust; producer and upstream workload credential lifecycle; subject-credential conveyance; and the intake-outcome record and return path.

**Residual risks:** a compromised but admitted upstream workload can still submit requests that are admissible yet unwanted. Those requests are still subject to full subject authentication and `basis-core` evaluation, which is exactly why intake admission is not authorization. A compromised intake realization could admit requests it should reject. Like the gateway's own admission boundary, intake's correctness rests on its implementation and deployment configuration.

## Deferred Decisions

The following are deliberately not decided here, so that later work does not interpret silence as authorization:

- **Resource and action mapping (cross-ecosystem Workstream 3C).** How upstream operation and target references map to BASIS actions (ADR-0017 verbs), canonical resource identifiers ([`resource-identifier-reconciliation.md`](../architecture/resource-identifier-reconciliation.md)), and a protocol-shaped operation. This includes canonical resource composition, mapping-version semantics, conversion of upstream durable asset identifiers, protocol target conversion, and full action mapping.
- **Context-assertion trust (cross-ecosystem Workstream 3D).** Which context categories an upstream system, or a producer relaying upstream-originated context, may authoritatively assert, and how such context is provenance-classified at the gateway. This is the category-scoped trust question `operation-producer-and-execution-boundary.md` §3 already holds open. Decision 4's interim posture stands until then.
- **Producer credential lifecycle (cross-ecosystem Workstream 3E).** Producer and upstream workload credential issuance, rotation, revocation, HA identity sharing, bootstrap, compromise handling, and disconnected operation, extending ADR-0008's own deferral of "full credential lifecycle automation."
- **Subject-credential conveyance** from an upstream system to the operation-producer role (ADR-0010 already leaves the mechanism open).
- **Intake transport, envelope, and integrity mechanism.**
- **Bounded-validity mechanism** (lifetime, freshness, nonce/replay handling), and broader authorization replay/freshness generally (ADR-0012, ADR-0013, ADR-0016).
- **Intake-outcome record, status vocabulary, and error codes.**
- **Return-path contract** for reporting BASIS-owned authorization or execution facts to an upstream system.
- **Repository and deployment placement** of an intake realization, and any `basis-producer` implementation phase for it.
- **Device-credential custody** and any change to protocol-executor responsibilities. None is intended.

## Non-Goals

This ADR does not modify any implementation repository or schema; does not define an endpoint, API, queue, or message format; does not change `basis-gateway` producer admission, `basis-core` evaluation, the ADR-0012 binding, or the protocol-executor role; does not modify any Accepted ADR's body; does not update the upstream platform's repository; and does not promote *upstream supervisory system*, *producer-intake boundary*, or *upstream workload identity* to canonical glossary terms. They are working vocabulary, following the convention `operation-producer-and-execution-boundary.md` uses for its own introduced terms.

## Validation / Implementation Gate

Acceptance of this ADR establishes the producer-intake boundary as governed architecture and supplies the BASIS-side decision needed for ARCH-GAP-001's joint closure. It does **not** authorize implementing an intake realization. At minimum, an intake implementation additionally requires decisions on: the intake transport and integrity mechanism; the bounded-validity rule (Decision 7); the Workstream 3C mapping for the operations the realization will accept; and the Workstream 3E credential posture for the workloads it involves. A bounded implementation plan must then derive from those accepted decisions.

## References

- [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) — §2 (operation initiator; operation-producer runtime), §3 (trust establishment; category-scoped trust open question), §6 (correlation ownership), §7 (provenance), §8 (failure conditions), §9 (topologies)
- [ADR-0007](0007-adapter-evidence-construction.md) — adapter evidence construction
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) — producer workload authentication and admission; producer vs. authorization subject
- [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) — trusted producer mTLS ingress
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) — `basis-producer` as operation-producer runtime
- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) — protocol-executor role; credential classes (Decision 14)
- [ADR-0012](0012-authorization-to-execution-binding.md) — authorization-to-execution binding
- [ADR-0013](0013-execution-lifecycle-semantics.md) — execution lifecycle; pre-boundary `NOT_ATTEMPTED` cases
- [ADR-0014](0014-minimum-execution-evidence-semantics.md) — execution evidence; identifier ownership
- [ADR-0016](0016-bounded-target-replay-freshness-posture.md) — bounded-target replay/freshness posture
- [ADR-0017](0017-action-vocabulary-naming-structure.md) — canonical operational action verbs
- [`resource-identifier-reconciliation.md`](../architecture/resource-identifier-reconciliation.md) — resource identifier composition
- [`threat-model.md`](../security/threat-model.md) — §7.2 forged identity context, §7.5 privilege escalation, §7.6 replay
- Ipotio architecture repository — upstream supervisory platform / operation-initiator decision; ARCH-GAP-001 (cross-repository reference, not hyperlinked)
