# Identity-to-Operation Contract and Interoperability Roadmap

This document defines the long-term architectural direction for BASIS as an open identity-to-operation contract and interoperability layer for fragmented operational technology (OT) environments. It is an architecture-planning document, not an implementation plan. It does not implement runtime behavior, modify any implementation repository, publish new schemas, define final JSON Schema files, choose a signing mechanism, choose a message broker, choose a transport, or commit to new repository names. It records long-term intent so that the idea is not lost while the ecosystem's present implementation effort continues, and it defines the architectural boundaries, decision gates, and completion criteria that later implementation work — whenever it begins — must satisfy.

**Status:** Planned — implementation is deferred until the current operation-aware gateway, audit-evidence, readiness, conformance, and release-hardening program reaches its intended completion point. No phase described in this document is in progress. This roadmap does not authorize implementation work in `basis-core`, `basis-gateway`, `basis-identity`, `basis-adapters`, `basis-console`, or `basis-schemas`. The `basis-gateway` operation-aware integration program described in [`ROADMAP.md`](../../ROADMAP.md) — and the audit-evidence, readiness, conformance, and release-hardening work that surrounds it across the distribution — remains the ecosystem's active implementation priority, and nothing in this document changes that sequencing or implies that priority is paused. Contract publication in `basis-schemas` is deferred until this roadmap's architecture and at least one reference implementation stabilize, following the same readiness discipline ADR-0005 established for the operation-aware contract suite. Repository names used in this document — including names such as a hypothetical relationship, evidence, or distribution service — are conceptual references to future architectural roles unless the repository already exists in the BASIS Core Services Distribution today (`basis-core`, `basis-gateway`, `basis-console`, `basis-adapters`, `basis-identity`, `basis-schemas`, `basis-architecture`; `basis-deploy` remains defined in architecture but not yet established as a repository, consistent with [`ROADMAP.md`](../../ROADMAP.md)).

This roadmap does not implement anything. It does not modify `basis-core`, `basis-identity`, `basis-gateway`, `basis-console`, `basis-adapters`, or `basis-schemas`. It does not add dependencies, choose a message-integrity or signing standard, select a transport or message-broker technology, or define a final machine-readable schema. Where a decision has not been made, this document says so explicitly and records what evidence would be needed to make it, consistent with how [`ROADMAP.md`](../../ROADMAP.md) distinguishes "In architecture" and "Research direction" items from released work.

A note on terminology: this roadmap introduces working vocabulary — identity-to-operation contract, trusted operation producer, enforcement disposition, execution result, interoperability conformance — that does not yet have entries in [`docs/glossary.md`](../glossary.md). Consistent with the glossary's role as a canonical reference for decided terminology rather than speculative vocabulary, formal glossary entries for these terms should be added only as each phase's architecture stabilizes through its own ADR, not as part of this roadmap. No term introduced here is promoted to canonical status by its appearance in this document.

---

## Purpose

The BASIS Core Services Distribution answers a bounded question well today: given a normalized subject, action, and resource, does policy permit the request, and is that decision auditable? [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) records the contracts that have converged across `basis-core`, `basis-gateway`, `basis-adapters`, and `basis-console` to answer it — an action vocabulary, action and resource composition, a decision request and response shape, and an audit event contract — and the questions that remain open even within that bounded scope, including the resource taxonomy, the action-domain-versus-resource-type distinction, and audit persistence and correlation.

This roadmap looks past that bounded question toward a harder one that a production OT authorization platform eventually has to answer: not merely whether one request is permitted by one policy engine, but how a fragmented collection of identity providers, remote-access systems, privileged-access-management tooling, gateways, protocol adapters, field devices, SIEMs, asset inventories, and human and automated actors can agree on a single, inspectable, extensible account of who acted, whose authority was exercised, what was requested, what was decided, what was enforced, whether execution occurred, and what evidence proves the sequence. No single OT vendor, identity provider, or protocol standard currently owns that account end to end. Each system in a typical OT environment holds part of the truth — an identity provider knows who authenticated; a PAM system knows who checked out a credential; a gateway knows what was authorized; a protocol adapter knows what a device was asked to do; a SIEM knows what looked anomalous — and today those fragments are not connected by any shared, open semantic model.

The purpose of this roadmap is not to select the technologies that would connect those fragments. It is to define the architectural shape of an identity-to-operation contract precise enough that a future implementation effort can build toward it without inventing a parallel semantic model, without prematurely publishing schema that has not stabilized, and without eroding the boundaries — kernel isolation, identity/authorization separation, evidence discipline — that already make the current architecture legible and defensible. Where the identity and fine-grained authorization expansion roadmap ([`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md)) develops how BASIS scales as an identity and authorization platform — multi-tenancy, delegation, workload identity, relationship-based authorization — this roadmap develops how BASIS's identity-to-operation account becomes legible and interoperable *across* systems that are not BASIS components at all. The two are complementary, not competing, and their intersection is addressed explicitly in **Relationship to Existing Roadmaps** below.

---

## Central Thesis

BASIS aims to become the open identity-to-operation contract for operational technology.

BASIS should provide a common, inspectable, extensible semantic model through which fragmented OT systems can agree on who acted, whose authority was exercised, whether authority was delegated, which trusted producer submitted the operation, which operation was requested, which OT resource was targeted, what operational context existed, what authorization decision was made, what enforcement action was taken, whether execution was attempted, whether execution completed, failed, or partially applied, and what evidence proves the sequence.

This is a larger ambition than authorization alone, but it remains built around deterministic authorization as its center of gravity. The contract does not exist to describe OT operations in the abstract; it exists to carry a request from a verified identity through an authorization decision to an enforcement outcome and a durable evidentiary record, in a form that other systems can consume, extend, and verify without adopting BASIS's implementation. The identity-to-operation contract is connective tissue, not a replacement for the systems it connects. It is not a replacement for identity providers, VPNs, privileged-access-management systems, jump hosts, Niagara, BACnet, Modbus, OPC UA, MQTT, SIEMs, network detection systems, asset inventories, or native OT protocols. Each of those systems continues to do what it already does. What the contract adds is a consistent way for those systems to exchange a shared identity-to-operation story through trusted integration points, rather than each pair of systems inventing its own bilateral translation.

### Preferred framing

OAuth standardized delegated authorization flows between IT systems. BASIS could standardize how identity-driven OT operations are described, authorized, enforced, executed, and recorded.

This is an analogy of architectural role, not a claim that BASIS is literally OAuth for OT. OAuth's contribution was not a specific technology; it was a shared vocabulary and flow structure — client, resource owner, authorization server, resource server, scope, grant — that let independently built systems interoperate around delegated authorization without each pair negotiating its own protocol. The parallel this roadmap draws is structural: a shared vocabulary and sequencing for identity-driven OT operations could let independently built OT identity, gateway, adapter, and evidence systems interoperate the same way. The parallel does not extend to OAuth's specific flows, token formats, or grant types, none of which are assumed to transfer to OT operation authorization, and it does not imply that BASIS should be described as "OAuth for OT" without this qualification.

### Open-glue framing

OT environments are fragmented across external identity providers, local identity authorities, VPNs and remote-access systems, PAM systems, shared engineering accounts, jump hosts, supervisors and controllers, gateways, protocol adapters, field devices, SIEM and logging systems, asset inventories, network detection systems, orchestration systems, human operators, and workloads and agents. Each of these systems may possess part of the truth about a given operation, and none of them is positioned to hold all of it. BASIS could provide the open semantic chain that connects those fragments without requiring any of them to be replaced or subordinated to BASIS.

The conceptual flow this chain follows, at the level this roadmap is written, is:

```text
verified identity
    ↓
authority and delegation
    ↓
trusted operation producer
    ↓
normalized operation
    ↓
canonical OT resource
    ↓
operational context
    ↓
authorization decision
    ↓
enforcement result
    ↓
execution result
    ↓
durable evidence
```

This flow is conceptual. It names the semantic stages the contract must eventually be able to represent; it is not a final schema, a fixed message envelope, or an implementation sequence diagram, and no phase of this roadmap should be read as freezing it into one.

---

## JSON and Contract Semantics

Several distinct kinds of work are easy to conflate when people say "the contract," and this roadmap depends on keeping them separate:

```text
JSON
    represents data
JSON Schema
    validates structure
BASIS architecture and contracts
    define meaning
TLS or mTLS
    protects data in transit
producer authentication
    establishes who submitted the message
signatures, digests, or message-integrity mechanisms
    provide integrity and origin evidence
basis-core
    evaluates authorization deterministically
gateway and enforcement boundaries
    enforce the decision
adapters or protocol-native enforcement points
    translate and execute native OT behavior
evidence
    records what occurred
```

JSON is a serialization format. It carries no trust, security, authorization, or integrity property on its own — a well-formed JSON document proves nothing about who sent it, whether it was tampered with in transit, or whether the operation it describes was actually authorized. JSON Schema validates that a document has the expected shape; it does not validate that the document is truthful, that its sender is who it claims to be, or that the operation it describes was permitted. Meaning — what a field is for, what values are legitimate, how the document relates to an authorization decision — is defined by BASIS architecture and contracts, not by the serialization or its structural validation. Transport security (TLS or mTLS) protects data in transit between two endpoints; it says nothing about whether the endpoint sending the data is a producer BASIS should trust with that specific claim. Producer authentication establishes which registered, trusted producer submitted a message; it is a separate concern from transport security and from the content of the message itself. Signatures, digests, or other message-integrity mechanisms provide integrity and origin evidence for a specific message, potentially outliving the transport session that carried it — a property transport security alone does not provide once a message has been received and stored. `basis-core` evaluates authorization deterministically against normalized input; it does not itself provide transport security, producer authentication, or message integrity, and it must not be asked to. Gateway and enforcement boundaries enforce the decision `basis-core` returns; they do not reinterpret it. Adapters, plugins, and proxies translate a decision into native OT protocol behavior and execute it, or report why they could not. Evidence records what occurred at each of these layers, durably and attributably enough to reconstruct the sequence after the fact.

No one of these mechanisms substitutes for another, and this roadmap does not propose that any single mechanism — a signature scheme, a schema validator, a transport protocol — could stand in for the others. This is restated as the Security Invariant below because it is easy to design toward a single artifact ("the signed JSON payload") that appears to satisfy all of these concerns and in fact satisfies none of them completely.

### The translation boundary

Legacy OT devices will not consume BASIS JSON directly, and this roadmap does not anticipate that they ever should. The expected translation boundary is:

```text
human / workload / machine / agent
    ↓
authenticated identity
    ↓
trusted producer
    ↓
BASIS identity-to-operation contract
    ↓
gateway authorization and enforcement
    ↓
adapter / plugin / proxy / native enforcement point
    ↓
BACnet / Modbus / OPC UA / Niagara / other native protocol
    ↓
device or operational system
    ↓
execution result and evidence
```

A human, workload, machine, or AI agent initiates an action under an authenticated identity. That identity, and whatever authority or delegation it carries, is expressed through a trusted producer — the adapter, gateway, plugin, proxy, or orchestrator that speaks the BASIS identity-to-operation contract on the initiator's behalf. The contract carries the request through gateway authorization and enforcement. An adapter, plugin, proxy, or other native enforcement point translates the enforced decision into the native protocol the target device or system actually understands — BACnet, Modbus, OPC UA, Niagara, or whatever else applies — and the device executes, or fails to execute, in its own native terms. The execution result and any evidence the native system can report are translated back across the same boundary. The device itself remains unchanged throughout; it never needs to know the contract exists.

---

## Architectural Invariants

The following invariants are preconditions for every phase in this roadmap, and they restate and extend boundaries already established in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/security/threat-model.md`](../security/threat-model.md), and the identity and fine-grained authorization expansion roadmap. Nothing in this section supersedes those documents; where language differs, this section is additive. A phase whose implementation would require relaxing one of these invariants should be treated as evidence that the phase has been scoped to the wrong component, not as grounds for an exception.

### Kernel invariant

`basis-core` remains deterministic, synchronous, side-effect free during evaluation, protocol neutral, transport neutral, and free from historical activity storage, analytics, session correlation, anomaly detection, automated response, and network or persistence dependencies during evaluation. This roadmap's contract may carry inputs to the kernel and carry outputs from it, but the kernel does not become the interoperability bus, and no phase below proposes that it should. Everything this roadmap adds sits around the kernel — in producers, in the gateway, in evidence pipelines — never inside it.

### Identity invariant

Authentication identity, authority, delegation, and producer identity remain distinguishable, and this roadmap must not collapse them into one ambiguous `user` field. Preserved throughout every phase are the distinctions among the authenticated subject, the authority issuer, the delegated subject, the acting identity or workload, the approving identity, the trusted producer, the enforcement point, and the execution target. This is a restatement, in cross-system terms, of the same discipline the identity and fine-grained authorization expansion roadmap applies within the BASIS ecosystem itself.

### Gateway invariant

`basis-gateway` authenticates, composes, invokes, enforces, and emits evidence. It does not invent authorization semantics, reinterpret kernel outcomes, claim that native execution succeeded merely because authorization allowed it, or become the durable historical activity store, the topology graph, or the detection engine. Nothing this roadmap adds to the gateway's future responsibilities — composing broader identity-to-operation context, invoking a producer-trust check, carrying execution-result evidence — changes what the gateway is authoritative for.

### Adapter invariant

Adapters and plugins normalize protocol operations and may report execution telemetry. They do not decide authorization, fabricate identity, convert an `ALLOW` decision into proof of execution, redefine canonical action or resource semantics independently, or create incompatible protocol-specific audit models. This restates, for the interoperability contract specifically, the trusted-adapter-boundary principle already established in [`docs/glossary.md`](../glossary.md) and [`docs/security/threat-model.md`](../security/threat-model.md): an adapter is trusted to normalize and report faithfully, never to authorize.

### Execution invariant

Authorization and execution are separate facts, and the contract must eventually be capable of distinguishing at least: authorized but not attempted; attempted and completed; attempted and failed; partially applied; and execution status unavailable or unconfirmed. This roadmap does not define a final status vocabulary for these states unless one is already governed elsewhere — it defines the architectural need for the distinction and names the decision gate that must resolve the vocabulary before it is published. An `ALLOW` decision is never, on its own, evidence that an operation occurred.

### Schema invariant

`basis-schemas` publishes stable contracts; it does not invent them. This roadmap may identify future schema families — an identity-to-operation envelope, a producer-trust assertion, an execution-result record — but it must not define their final machine-readable content. Any future contract this roadmap anticipates is proposed to `basis-schemas` only after its architecture and a reference implementation exist, consistent with the readiness discipline ADR-0005 established.

### Security invariant

Transport security, origin authentication, integrity, authorization, enforcement, and evidence are separate layers, each named in **JSON and Contract Semantics** above. No one mechanism substitutes for all the others, and no phase of this roadmap should be read as proposing a single artifact that collapses them.

### Open-contract invariant

The open contract remains the foundation even if commercial services eventually provide certified integrations, hosted analytics, enterprise support, managed deployment, policy packs, compliance reporting, advanced detection content, or upgrade assurance. This mirrors the BASAuth boundary already established in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md): commercial capabilities may build on the contract; they must not make the foundational contract proprietary, and no phase of this roadmap should be read as creating a commercial dependency for basic interoperability.

### Console invariant

`basis-console` observes and explains interoperability state. It does not become the identity authority, the contract authority, the enforcement authority, the execution authority, or the activity store. Training mode may explain the identity-to-operation chain this roadmap defines; it must not change the chain, consistent with the console invariant already established in [`docs/architecture/basis-console.md`](../architecture/basis-console.md) and restated in the identity and fine-grained authorization expansion roadmap's training-mode constitutional requirement.

---

## Training-Mode Constitutional Requirement

Consistent with the constitutional requirement the identity and fine-grained authorization expansion roadmap already establishes, every externally observable capability this roadmap introduces must have a corresponding operator-facing representation and training-mode explanation in `basis-console` from the outset of its design, not as work added after a capability ships. Training mode may explain producer trust, delegation and provenance, protocol-to-operation normalization, enforcement disposition, the distinction between authorization and execution, transport-versus-message integrity, and contract-version compatibility. Training mode must not change backend behavior, bypass authentication or authorization, fabricate live data, or expose secrets or unredacted credentials. Each phase below states what operator mode needs to show, what training mode needs to explain, what evidence proves the behavior occurred, and what must be redacted; a **Console Education Matrix** near the end of this document summarizes the pattern across phases.

---

## Contract Responsibility Model

The following are the conceptual semantic areas a future identity-to-operation contract may eventually need to represent. This is not a final JSON object, a field list, or a schema tree — it is a map of the questions the contract must be able to answer, organized the way the flow in **Central Thesis** above suggests, so that later architecture work has a shared starting vocabulary rather than a blank page. Nothing named here is frozen; several of these areas overlap with concepts the identity and fine-grained authorization expansion roadmap is independently developing, and where they do, that roadmap's phases are the authoritative source (see **Relationship to Existing Roadmaps** below).

**Identity.** The human, workload, machine, device (where a device can meaningfully hold identity), or agent that initiated or is otherwise party to an operation.

**Authority.** The issuer, credential source, roles, groups or attributes, authority mode, and trust domain that establish what an identity is entitled to assert.

**Delegation.** The acting-on-behalf-of relationship, the approving identity where one exists, the delegated scope, the workload or agent-run instance carrying out a delegated action, the delegation chain, and its attenuation.

**Producer.** The adapter, gateway, plugin, proxy, orchestrator, or workload that submitted the operation into the contract, and that producer's trust classification.

**Operation.** The action, the operation's intent, the requested state change, an idempotency or replay identity, and a request or correlation identity.

**Resource.** The site, facility, building, system, controller, device, point, process, or safety function the operation targets, expressed through a canonical resource identifier.

**Operational context.** The protocol in use, location, device state, safety state, environmental state, risk context, and temporal context surrounding the operation.

**Decision.** The evaluation status, the authorization outcome, any governed failure reason, the policy or bundle identity that produced the outcome, and a rule-evidence or trace reference.

**Enforcement.** The enforced disposition, the enforcement point, the enforcement timestamp, and any recorded inability to enforce.

**Execution.** Whether the operation was attempted, completed, failed, partially applied, or left unavailable or unconfirmed — the states named in the Execution invariant above, held as conceptual requirements rather than a frozen vocabulary.

**Evidence.** The trace, provenance, event and ingest timestamps, evidence references, integrity metadata, signatures or digests, redaction classification, source-system identity, and contract version associated with the operation.

---

## Roadmap Phases

The following ten phases are architecture phases, not a predetermined pull-request schedule. Each phase uses a consistent structure: purpose, primary repositories, prerequisites, architectural outcome, key capabilities, security and abuse cases, distributed-systems concerns, decision gates, completion criteria, operator-mode representation, training-mode explanation, evidence and audit requirements, schema and documentation impact, explicitly deferred work, and engineering experience developed.

### Phase 1 — Existing Contract and Evidence Inventory

**Purpose.** Inventory the contracts and evidence already present across `basis-identity`, `basis-gateway`, `basis-core`, `basis-adapters`, `basis-schemas`, `basis-console`, and `basis-architecture`; distinguish released contracts, implementation-local shapes, and future gaps; identify overlap with the operation-aware contract suite; and prevent this roadmap from creating a parallel semantic model where a governed one already exists.

**Primary repositories.** `basis-architecture` (the inventory and its resulting gap analysis); no implementation repository is modified.

**Prerequisites.** None. This phase is discovery work against what already exists and can begin, as architecture work, independently of every other phase below.

**Architectural outcome.** A documented current-state chain: what identity, operation, decision, enforcement, and evidence shapes already exist in the ecosystem today, drawn from [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md), the operation-aware contract suite first published in `basis-schemas` v0.2.0 and subsequently corrected through the current v0.2.2 release, `basis-identity`'s canonical identity context and session model, and the gateway's reserved evidence namespace — together with an explicit gap analysis of what a cross-system identity-to-operation contract would still need that none of these currently provide, such as producer-trust representation, delegation-chain representation crossing organizational boundaries, and execution-result evidence distinct from authorization evidence.

**Key capabilities.** A cross-repository contract table extending the existing ecosystem contract inventory with an interoperability lens; an explicit mapping of which existing contracts (action vocabulary, resource identifier, decision request/response, audit event, evaluation trace, audit evidence, gateway audit event, reason code, policy bundle/rule/condition, canonical identity context) already satisfy part of the Contract Responsibility Model above, and which parts remain unaddressed; and a recorded list of overlaps with the operation-aware contract suite so that later phases build on it rather than beside it.

**Security and abuse cases.** None directly — this is a documentation phase. Its principal risk is a documentation risk: an inventory that overstates what exists, or that silently promotes an implementation-local shape to contract status, would mislead every phase that follows it.

**Distributed-systems concerns.** None; this phase produces no runtime behavior.

**Decision gates.** None resolved by this phase; its output is the input the later decision gates in this roadmap depend on.

**Completion criteria.** A reviewed inventory document, consistent in method with [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md), that a later phase can cite without re-deriving the current state of the ecosystem's contracts.

**Operator-mode representation.** Not applicable; this is an architecture-discovery phase with no operator-facing surface.

**Training-mode explanation.** Not applicable at this phase; the inventory informs later phases' training-mode content.

**Evidence and audit requirements.** The inventory itself should record its sources and the date each was current as of, the same discipline [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) already follows, so that later readers can tell whether it has gone stale.

**Schema and documentation impact.** None; this phase proposes no schema.

**Explicitly deferred work.** Any resolution of the open questions the existing inventory already records (resource taxonomy, action-domain-versus-resource-type distinction, audit persistence and correlation) — those remain owned by the documents that already track them, not duplicated here.

**Engineering experience developed.** Cross-repository architectural discovery discipline: reading what is actually implemented before proposing what should exist next.

### Phase 2 — Identity, Authority, Delegation, and Producer Model

**Purpose.** Define the conceptual distinction, at cross-system scope, among authenticated identity, authority, delegation, actor, approver, and trusted producer, extending the same distinctions the identity and fine-grained authorization expansion roadmap develops within the BASIS ecosystem to the systems that sit outside it — external PAM tooling, shared engineering accounts, jump hosts, remote vendors, and other producers this roadmap's contract must eventually recognize.

**Primary repositories.** `basis-architecture` (the conceptual model); no implementation repository is modified. Eventual implementation would primarily involve `basis-identity` and `basis-gateway`, per the existing ecosystem boundaries.

**Prerequisites.** Phase 1's inventory; coordination with the identity and fine-grained authorization expansion roadmap's Phase 3 (token exchange and delegation) and Phase 4 (workload and non-human identity), so this phase extends that work outward rather than re-deriving it.

**Architectural outcome.** A conceptual model for how a subject's authenticated identity, the authority it carries, any delegation chain it participates in, and the producer that ultimately submits the resulting operation into the BASIS contract are represented as distinct, attributable facts — accounting for humans, workloads, machines, agents, shared engineering accounts, service accounts, remote vendors, and delegated automation, none of which collapse into a single ambiguous field.

**Key capabilities.** A producer-trust classification model distinguishing a producer's registration and enrollment status from the identity of whoever is acting through it; a conceptual representation of delegation chains that cross an organizational or trust-domain boundary, not only chains internal to one BASIS deployment; and an explicit treatment of shared or ambiguous accounts (a shared engineering login, a jump-host session shared across a shift) as a named category the contract must be able to represent honestly rather than silently attribute to a single identity.

**Security and abuse cases.** Producer impersonation, where a system that is not a registered, trusted producer submits operations as though it were one; confused-deputy scenarios, where a trusted producer is induced to submit an operation on behalf of an identity that never authorized it; delegation escalation across an organizational boundary, which is a harder case than the single-deployment escalation the expansion roadmap's Phase 3 already addresses; and shared-account ambiguity being exploited to defeat attribution.

**Distributed-systems concerns.** Consistency between the authority and delegation state a producer relies on at submission time and the state that was actually current — the same class of staleness concern the expansion roadmap's Phase 2 and Phase 5 already name for BASIS-internal caches, now extended to producers outside BASIS's direct control.

**Decision gates.** How producer identity and producer trust are represented; how authority differs from authenticated identity at the contract level; how delegation chains are represented when they cross a trust-domain boundary the expansion roadmap's internal delegation model does not reach. None of these are resolved by this roadmap.

**Completion criteria.** A conceptual model, reviewed and reconciled explicitly against the expansion roadmap's Phase 3 and Phase 4, that a future ADR can adopt without contradiction.

**Operator-mode representation.** The identity, authority, and delegation chain behind a given operation, and the producer that submitted it.

**Training-mode explanation.** Why a given operation's identity chain looks the way it does — which credential authenticated, which authority it carried, whether it was delegated, and which producer ultimately submitted it into the contract.

**Evidence and audit requirements.** Sufficient provenance to reconstruct the identity, authority, and delegation chain behind any operation after the fact, extending the same evidentiary standard the expansion roadmap's Phase 3 already requires for BASIS-internal delegation.

**Schema and documentation impact.** A future producer-trust and delegation-chain contract extension is a candidate for `basis-schemas`, deferred until this phase's architecture and a reference implementation exist.

**Explicitly deferred work.** Final producer-registration mechanism; final representation of cross-trust-domain delegation; whether producer trust is itself distributed via a mechanism resembling Phase 5's signed-artifact distribution below.

**Engineering experience developed.** Cross-organizational identity and delegation modeling, extending single-deployment delegation discipline to a setting where the deployment does not control every participant.

### Phase 3 — Canonical Operation and Resource Semantics

**Purpose.** Define how the existing action vocabulary, canonical resource identifier, operation intent, protocol context, safety context, and evidence references already governed elsewhere in the ecosystem form one interoperable operation description, without redesigning any of them.

**Primary repositories.** `basis-architecture`; no implementation repository is modified.

**Prerequisites.** Phase 1's inventory, since this phase explicitly builds on what it records rather than proposing new vocabulary.

**Architectural outcome.** A conceptual description of how an operation — expressed today through the governed action vocabulary, the canonical resource identifier, the operation-aware context objects, and the evidence-reference contracts of the operation-aware contract suite first published in `basis-schemas` v0.2.0 and subsequently corrected through the current v0.2.2 release — composes into the single operation description a cross-system contract would carry, together with an explicit, honest list of what remains absent: primarily the unresolved resource taxonomy and the action-domain-versus-resource-type distinction that [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) already records as open.

**Key capabilities.** A composition map showing how the existing five-verb action vocabulary, the canonical resource identifier format, the operation-aware context objects (location, device, protocol, safety, environment, risk), and the existing evidence-reference contracts (identity evidence reference, adapter evidence reference, evidence digest) already cover most of the Contract Responsibility Model's operation and resource categories; and an explicit statement of the remaining gaps rather than a proposal to close them prematurely.

**Security and abuse cases.** Resource or operation misrepresentation carried across a system boundary — the cross-system analogue of the adapter-normalization threat already analyzed in [`docs/security/threat-model.md`](../security/threat-model.md) §6.3, now considered at the scope of a producer outside BASIS's own adapter trust boundary.

**Distributed-systems concerns.** None beyond what the existing operation-aware architecture already addresses; this phase is a composition exercise over governed vocabulary, not new distributed behavior.

**Decision gates.** Whether the resource taxonomy and the action-domain-versus-resource-type distinction should be resolved before or independently of this roadmap's later phases — this roadmap does not resolve either question, and explicitly defers to whatever process [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) §7 eventually resolves them through.

**Completion criteria.** A composition document, reviewed for consistency with every operation-aware architecture document it draws from, that names its gaps rather than glossing over them.

**Operator-mode representation.** The composed operation description for a given cross-system exchange: action, resource, and context, in terms an operator already recognizes from the existing gateway and console surfaces.

**Training-mode explanation.** How a cross-system operation description maps onto the governed BASIS action and resource vocabulary, and where that mapping is still incomplete.

**Evidence and audit requirements.** None beyond what the existing operation-aware evidence contracts already require; this phase does not add a new evidence category.

**Schema and documentation impact.** None; this phase composes existing, already-published contracts rather than proposing new ones.

**Explicitly deferred work.** Resolution of the resource taxonomy and the action-domain-versus-resource-type distinction, both explicitly out of scope and owned elsewhere.

**Engineering experience developed.** Composition discipline: building on governed vocabulary precisely, without redefining it under a different name because a new context makes redefinition tempting.

### Phase 4 — Enforcement and Execution Lifecycle

**Purpose.** Define the semantic distinction among request, decision, disposition, enforcement, execution, execution result, and post-execution evidence, so the contract can eventually represent an operation's full lifecycle rather than stopping at the authorization decision.

**Primary repositories.** `basis-architecture`; eventual implementation would touch `basis-gateway` and `basis-adapters`, per the existing ecosystem boundaries.

**Prerequisites.** Phase 3's composed operation description, since execution and enforcement are described relative to the operation they act on.

**Architectural outcome.** A conceptual lifecycle model distinguishing the moment a request is received, the moment `basis-core` reaches a decision, the disposition the gateway derives from that decision, the enforcement action the gateway or an enforcement point actually takes, the attempt to execute against the native OT system, the result of that attempt, and any evidence produced after execution — with the Execution invariant's five states (authorized but not attempted; attempted and completed; attempted and failed; partially applied; execution status unavailable) as the conceptual minimum the model must support.

**Key capabilities.** A lifecycle-stage model precise enough to distinguish "authorized" from "executed" in every representation the contract produces; explicit conceptual handling of asynchronous execution, where the native OT system does not confirm completion within the request's own timeframe; partial execution, where an operation affecting multiple points or a batch of devices succeeds for some and fails for others; timeouts and uncertain execution state, where the contract must represent "unknown" honestly rather than defaulting to an assumed success or failure; retries and duplicate commands, and how the contract distinguishes a legitimate retry from a duplicate submission; rollback or compensating action, where reversing a partially applied operation is itself a distinct operation the model must be able to represent; and acknowledgement versus proof of execution, since a device or protocol acknowledging receipt of a command is not the same fact as the command having taken effect.

**Security and abuse cases.** False execution success, where a disposition or acknowledgement is misread or misrepresented as proof of execution; missing execution evidence, where the absence of a result is not distinguishable from a suppressed one; and duplicate command execution resulting from ambiguous retry semantics.

**Distributed-systems concerns.** Asynchronous execution and delayed confirmation are the central concern of this phase: a native OT system may not confirm within any bounded window the contract can rely on, and the model must represent that honestly rather than forcing a premature synchronous conclusion. Multiple enforcement points acting on the same operation, and reconciling their potentially divergent execution reports, is a related concern this phase names without resolving.

**Decision gates.** How execution result vocabulary is eventually governed — a closed enum, an open extensible set, or something else; how retries and idempotency identifiers interact with the operation description from Phase 3; and how a rollback or compensating action is represented relative to the operation it reverses. None of these are resolved by this roadmap.

**Completion criteria.** A lifecycle model, reviewed against at least the asynchronous, partial-execution, and unconfirmed-execution scenarios named above, that a future ADR can adopt as the basis for an execution-result contract.

**Operator-mode representation.** The current lifecycle stage of a given operation — authorized, enforced, execution attempted, execution result — in terms an operator can act on.

**Training-mode explanation.** Why an authorization `ALLOW` is not proof that anything happened, walked through the same lifecycle stages named above.

**Evidence and audit requirements.** Post-execution evidence, where a native system can provide it, is recorded distinctly from authorization and enforcement evidence, never merged into a single record that obscures which fact came from which stage.

**Schema and documentation impact.** A future execution-result contract is a candidate for `basis-schemas`, deferred until this phase's architecture and a reference implementation exist.

**Explicitly deferred work.** Final execution-result vocabulary; final retry and idempotency-identifier format; final rollback/compensating-action representation.

**Engineering experience developed.** Lifecycle modeling under asynchronous, partial, and uncertain execution — a materially harder problem than modeling the authorization decision alone.

### Phase 5 — Producer Authentication, Integrity, and Transport Profiles

**Purpose.** Define the architectural requirements for securely transporting contract messages between trusted producers and BASIS, without choosing a specific technology for any of them.

**Primary repositories.** `basis-architecture`; eventual implementation would touch `basis-gateway` and, for producer enrollment, `basis-identity`.

**Prerequisites.** Phase 2's producer-trust model, since transport and integrity requirements are meaningless without a defined notion of which producers are being authenticated.

**Architectural outcome.** A set of architectural requirements — not a selected technology — for producer enrollment and trust, transport protection, message signing or other integrity mechanisms, replay protection, idempotency, key rotation, trust-anchor distribution, and delivery under constrained or intermittent connectivity, stated precisely enough that a future technology evaluation (Phase 8 below, and the external-technology-evaluation discipline the expansion roadmap's Phase 10 already establishes) can be run against them.

**Key capabilities.** Producer enrollment and trust requirements, distinct from the trust classification Phase 2 defines conceptually; TLS and mTLS as baseline transport-protection requirements; a requirement to distinguish detached from embedded integrity metadata as an open design question rather than a foregone choice; replay-protection and idempotency requirements, coordinated with the idempotency and replay-identity concept named in the Contract Responsibility Model's Operation category; key-rotation and trust-anchor-distribution requirements; and explicit requirements for air-gapped transport and store-and-forward delivery, since OT deployments frequently cannot reach a networked distribution source continuously — the same operational reality [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 11 already names for policy distribution.

**Security and abuse cases.** Producer impersonation and signature or digest substitution, both named at the ecosystem level in **Security and Abuse Cases** below; replay of a previously valid message; and key-rotation failure leaving a producer's messages unverifiable or, worse, leaving a compromised key trusted longer than intended.

**Distributed-systems concerns.** Store-and-forward delivery under intermittent connectivity; message ordering and duplicate delivery, mirroring the same idempotency discipline the expansion roadmap's Phase 6 already requires for SCIM synchronization; and late delivery, where a message arrives after the operational context it describes has changed.

**Decision gates.** Whether integrity metadata is embedded in the message or detached from it; which signing or integrity standards should be evaluated, without pre-selecting one; how replay and idempotency identifiers interact; and how late, duplicate, and out-of-order messages are reconciled once they arrive. This roadmap does not resolve any of these; it names them as the questions Phase 8's conformance work and any future technology evaluation must answer.

**Completion criteria.** A requirements document specific enough to be falsifiable against a candidate technology, reviewed and accepted, with no technology selection made as part of accepting it.

**Operator-mode representation.** Producer trust and transport-integrity status for a given operation: verified, unverified, or unavailable.

**Training-mode explanation.** The distinction between transport security and message integrity, and why a verified TLS connection does not by itself prove a specific message's integrity or origin.

**Evidence and audit requirements.** Integrity-verification outcomes (verified, failed, unavailable) are recorded as evidence distinct from the authorization outcome, so a denial caused by a failed integrity check is never indistinguishable from a denial caused by policy.

**Schema and documentation impact.** A future producer-authentication and message-integrity contract is a candidate for `basis-schemas`, deferred until architecture and a reference implementation exist.

**Explicitly deferred work.** Specific signing or integrity standard; specific transport technology; specific key-management or trust-anchor-distribution mechanism.

**Engineering experience developed.** Requirements-driven security architecture: specifying what a mechanism must guarantee before evaluating which mechanism guarantees it.

### Phase 6 — Adapter, Plugin, Proxy, and Gateway Integration Profiles

**Purpose.** Describe how integrations participate in the identity-to-operation contract without requiring legacy OT devices to understand it, and define the responsibilities an integration must satisfy to claim BASIS compatibility.

**Primary repositories.** `basis-architecture` (the integration-profile requirements); eventual implementation would be distributed across `basis-adapters`, `basis-gateway`, and any future community-contributed integration.

**Prerequisites.** Phase 3's canonical operation and resource semantics and Phase 4's enforcement and execution lifecycle, since an integration profile is defined in terms of what it must translate and what it must report.

**Architectural outcome.** A conceptual profile — not an implementation — of what an integration must do to participate in the contract: translate a native protocol operation into the canonical operation description, translate an enforced decision back into native protocol behavior, and report execution telemetry in the lifecycle terms Phase 4 defines, for integration classes including a Niagara module, a BACnet gateway, a Modbus proxy, an OPC UA integration, an MQTT producer, a REST integration, PAM or jump-host integration, identity-provider integration, a SIEM exporter, and an asset-inventory connector. This roadmap does not implement or promise any of these; it defines what any of them would need to satisfy.

**Key capabilities.** A minimum-responsibility profile applicable across integration classes: faithful normalization consistent with the trusted-adapter-boundary invariant already established in [`docs/glossary.md`](../glossary.md); faithful translation of enforcement dispositions back into native behavior; honest execution-result reporting, including honest reporting of an integration's own inability to confirm execution; and non-participation in authorization decisions, restated from the Adapter invariant above. Class-specific notes — for example, that a PAM or jump-host integration primarily contributes producer and identity evidence rather than operation execution, or that a SIEM exporter primarily consumes evidence rather than producing operations — are recorded as profile variations, not separate architectures.

**Security and abuse cases.** An integration claiming BASIS compatibility while omitting required evidence, named explicitly in **Security and Abuse Cases** below; protocol downgrade, where an integration silently falls back to a less secure native protocol mode without reporting that it has done so; and a compromised or misbehaving integration misrepresenting either the operation it normalized or the execution result it reports, extending the adapter-normalization threat already analyzed in [`docs/security/threat-model.md`](../security/threat-model.md) §6.3 to every integration class this phase names.

**Distributed-systems concerns.** Integration classes vary widely in their connectivity and latency characteristics — a REST integration and an air-gapped Modbus proxy do not share the same operational profile — and this phase's minimum-responsibility profile must remain satisfiable across that range without assuming continuous connectivity as a baseline.

**Decision gates.** How an integration demonstrates that it satisfies the minimum-responsibility profile — self-attestation, a conformance test kit (Phase 8 below), or both; and how contract-version compatibility is negotiated between an integration and the gateway it connects to (Phase 7 below). Neither is resolved by this roadmap.

**Completion criteria.** A reviewed integration-profile document specific enough that a future contributor building any of the named integration classes could determine what they must implement, without this roadmap having implemented any of it.

**Operator-mode representation.** Which integrations are active for a given deployment, their contract-compatibility status, and their trust classification.

**Training-mode explanation.** How a native protocol operation is normalized into the canonical operation description by a given integration class, and what that integration is and is not trusted to do.

**Evidence and audit requirements.** Integration identity and version are recorded as part of the provenance of any operation or evidence record the integration produced or transported.

**Schema and documentation impact.** None directly; a future integration-conformance contract is a candidate deferred to Phase 8.

**Explicitly deferred work.** Implementation of any named integration class; a certification or compatibility-claim process, deferred to Phase 9.

**Engineering experience developed.** Integration-profile design that separates what an integration must guarantee from how any specific integration is built.

### Phase 7 — Contract Versioning and Schema Readiness

**Purpose.** Define semantic versioning expectations, compatibility rules, and the readiness gate a future identity-to-operation contract must clear before any part of it is proposed to `basis-schemas`, without publishing schemas in this phase.

**Primary repositories.** `basis-architecture`; `basis-schemas` only after this phase's readiness gate is met and a proposal is made through the existing schema-readiness process.

**Prerequisites.** Phases 1 through 6, since versioning and readiness are meaningless without the contract surfaces they would apply to.

**Architectural outcome.** A readiness discipline for this roadmap's eventual contract surfaces, modeled on — not a departure from — the discipline ADR-0005 already established for the operation-aware contract suite: semantic-versioning expectations; the distinction between additive and breaking changes; optional-versus-required field discipline; compatibility windows; producer-and-consumer version negotiation; deprecation handling; redaction and privacy expectations; extension-namespace discipline, preventing vendor-specific semantic fragmentation; minimum evidence quality required before a contract surface is considered ready; and the specific readiness gate that must be met before any part of this roadmap's contract is proposed to `basis-schemas`.

**Key capabilities.** A version-negotiation model allowing a producer built against one contract version and a gateway built against a newer compatible version to interoperate without either silently misinterpreting the other; an extension-namespace convention, analogous to the gateway's existing reserved `basis_gateway.*` evidence namespace, that lets an integration or deployment add fields without fragmenting the shared semantic model; and an explicit statement of the minimum evidence — architecture review, a working reference implementation, and demonstrated interoperability across at least two independent implementations — this roadmap considers necessary before proposing a contract to `basis-schemas`.

**Security and abuse cases.** Malicious extension fields, where an extension namespace is used to smuggle content that collides with or reinterprets a governed field; and version-negotiation downgrade, where a producer or consumer is induced to negotiate down to an older, less secure contract version.

**Distributed-systems concerns.** Compatibility windows across a fleet of producers and gateways that cannot all be upgraded simultaneously — a concern this phase should treat consistently with the staged-rollout discipline the expansion roadmap's Phase 11 already develops for policy distribution.

**Decision gates.** How privacy and redaction work jointly across identity and OT operational context, which is a materially different redaction problem than redacting a single system's evidence; and when architecture and reference-implementation maturity are considered sufficient to propose a first `basis-schemas` contract from this roadmap. Both are explicitly unresolved here.

**Completion criteria.** A readiness-discipline document, reviewed and accepted, that a future `basis-schemas` proposal for any contract this roadmap anticipates could be measured against.

**Operator-mode representation.** The active contract version a given deployment, producer, or integration is operating under.

**Training-mode explanation.** How contract versions are negotiated between a producer and BASIS, and what happens when they are incompatible.

**Evidence and audit requirements.** Contract version is recorded as part of every operation and evidence record's provenance, consistent with the Contract Responsibility Model's Evidence category.

**Schema and documentation impact.** None; no schema is published by this phase. This phase defines the gate a future schema proposal must clear.

**Explicitly deferred work.** The actual `basis-schemas` proposal for any contract this roadmap anticipates; final extension-namespace syntax.

**Engineering experience developed.** Contract-readiness discipline: distinguishing an architecture that is well-reasoned from one that has been proven across independent implementations, and treating only the latter as ready for publication.

### Phase 8 — Interoperability Conformance

**Purpose.** Define the future compatibility-vector and test-kit strategy that would let an independent implementation demonstrate it satisfies this roadmap's contract, without building that test kit in this phase.

**Primary repositories.** `basis-architecture` (the conformance strategy); a future conformance test kit would be a new artifact, conceptual until this phase's architecture and Phase 7's readiness gate are both satisfied.

**Prerequisites.** Phase 6's integration profiles and Phase 7's versioning discipline, since conformance is measured against both.

**Architectural outcome.** A conceptual strategy for validating that an implementation satisfies the identity-to-operation contract, modeled on the canonical-compatibility-scenario discipline `basis-schemas` already uses for the operation-aware contract suite, extended to scenarios that specifically exercise the interoperability boundary this roadmap adds.

**Key capabilities.** A set of conceptual compatibility scenarios a future conformance kit would need to cover, including: a human acting through a trusted adapter reaching a BACnet point; a workload acting through the gateway reaching a Modbus operation; a delegated agent action that requires approval evidence before it is enforced; an authorization `ALLOW` followed by a failed execution, exercising the Execution invariant directly; an authorization `DENY` with no execution attempted; a duplicate request handled idempotently; a partially applied operation; an untrusted or unregistered producer; an invalid signature or integrity failure; late or out-of-order execution evidence; and air-gapped store-and-forward evidence delivery. Each scenario is named as a conceptual requirement for a future test kit, not implemented here.

**Security and abuse cases.** Every abuse case named in **Security and Abuse Cases** below is, in effect, a candidate conformance scenario: this phase's purpose is precisely to ensure those threats are exercised in an interoperability test kit rather than left to be discovered in a production deployment.

**Distributed-systems concerns.** A conformance kit that only exercises the synchronous, well-connected case would validate a materially easier problem than the one this roadmap actually poses; the scenario list above deliberately includes asynchronous, partial, delayed, and disconnected cases for that reason.

**Decision gates.** Whether conformance is demonstrated through self-attestation, an automated test kit, third-party review, or some combination — this roadmap does not select among them, consistent with Phase 6's deferral of the same question.

**Completion criteria.** A reviewed conformance strategy naming the scenario set above (or a refined version of it) precisely enough that a future contributor could build a test kit from it, with no test kit built as part of this phase.

**Operator-mode representation.** Not applicable directly; conformance status of a given integration is surfaced through Phase 6's integration-status representation.

**Training-mode explanation.** What a conformance scenario demonstrates and why it matters — for example, why the "authorization allowed, execution failed" scenario exists and what it protects against.

**Evidence and audit requirements.** Conformance test results, once a kit exists, are themselves evidence supporting or undermining an integration's compatibility claim.

**Schema and documentation impact.** None; this phase defines a strategy, not a contract.

**Explicitly deferred work.** The conformance test kit itself; any certification process, deferred to Phase 9.

**Engineering experience developed.** Conformance-strategy design: identifying the scenarios that actually exercise an architecture's stated guarantees rather than only its happy path.

### Phase 9 — Reference Integrations and Adoption

**Purpose.** Define how the contract becomes usable through bounded reference implementations, without turning this roadmap into a product backlog or a commitment to build every named integration class.

**Primary repositories.** `basis-architecture` (the adoption model); reference integrations, if pursued, would be scoped individually and are not committed to by this roadmap.

**Prerequisites.** Phase 7's readiness discipline and Phase 8's conformance strategy, since reference integrations are the first implementations a conformance kit would need to validate against.

**Architectural outcome.** A model for how the contract could reach usable adoption through a bounded set of reference integrations and an open contribution path, without this roadmap committing to specific integrations, timelines, or maintainers.

**Key capabilities.** An open-source integration contribution model, describing how a community contributor could build and submit an integration against this roadmap's eventual contract; a definition of what "certification" or a "conformance claim" would mean, building on Phase 8's strategy, without operating a certification program as part of this roadmap; a notion of implementation maturity levels (for example, experimental, conformant, production-hardened) that lets a deployment reason about an integration's readiness without a binary "supported or not" judgment; a description of how sector-specific context extensions (for example, extensions specific to a given regulatory or safety domain) could be added consistent with Phase 7's extension-namespace discipline; and an explicit statement of support boundaries, distinguishing what the open contract guarantees from what any given implementation's maintainers commit to.

**Security and abuse cases.** An integration or reference implementation claiming a maturity level or conformance status it has not actually demonstrated, restated from Phase 6 and Phase 8 at the adoption-model level.

**Distributed-systems concerns.** None beyond what earlier phases already name; this phase is primarily an adoption and governance model rather than new technical architecture.

**Decision gates.** Whether reference integrations are maintained by the Basis Foundation, by commercial contributors, or by community contributors independently, and what governance applies to each case — none of this is resolved by this roadmap, and it should be informed by however Foundation governance itself matures, per [`GOVERNANCE.md`](../../GOVERNANCE.md).

**Completion criteria.** A reviewed adoption model, with no reference integration built or committed to as part of accepting it.

**Operator-mode representation.** An integration's maturity level and conformance status, extending Phase 6's integration-status representation.

**Training-mode explanation.** What a conformance claim does and does not guarantee, and how a deployment should weigh an integration's maturity level when deciding whether to depend on it.

**Evidence and audit requirements.** Maturity-level and conformance claims are themselves evidence, timestamped and attributable to whoever made them.

**Schema and documentation impact.** None directly.

**Explicitly deferred work.** Any specific reference integration; any formal certification program; commercial-services scoping, addressed at the strategy level only in **Open-Source and Commercial Strategy** below.

**Engineering experience developed.** Open-source adoption and governance design for a technical contract, distinct from designing the contract itself.

### Phase 10 — Failure, Isolation, and Security Validation

**Purpose.** Validate, empirically, the security and failure properties this roadmap's earlier phases assert architecturally — the same discipline the expansion roadmap's Phase 12 applies to its own twelve phases, applied here to this roadmap's contract and interoperability surfaces.

**Primary repositories.** `basis-architecture` (the validation strategy); actual validation work depends on reference integrations existing, per Phase 9, and is not performed as part of accepting this roadmap.

**Prerequisites.** Phases 1 through 9, since this phase validates them rather than building new capability.

**Architectural outcome.** A stated validation strategy covering replay protection, producer-impersonation resistance, tamper detection, cross-tenant isolation where a deployment's identity model has tenancy per the expansion roadmap's Phase 1, duplicate and reordered events, partial execution, evidence loss, stale configuration, key rotation, unavailable signing infrastructure, protocol integration failure, air-gapped synchronization, and privacy and redaction boundaries — named as the properties a future validation effort must test, not as properties already demonstrated by this roadmap.

**Key capabilities.** A validation strategy naming, for each property above, what a successful test would demonstrate and what evidence it would produce; explicit acknowledgment that this phase's completion criteria cannot be met until reference integrations from Phase 9 exist to validate.

**Security and abuse cases.** This entire phase is a security and abuse-case validation exercise; it should specifically re-test the cross-phase concerns named in **Security and Abuse Cases** below, under load and failure conditions rather than only under normal operation, mirroring the discipline the expansion roadmap's Phase 12 already states for its own scope.

**Distributed-systems concerns.** This phase is where every consistency, staleness, and partition-behavior assumption made in Phases 1 through 9 would be empirically verified rather than assumed, once implementation exists to verify against.

**Decision gates.** What specific service-level objectives or bounded guarantees this roadmap's eventual contract should target, and what constitutes acceptable degraded-mode behavior versus unacceptable behavior — neither is resolved here, consistent with how the expansion roadmap's Phase 12 treats the same category of question for its own scope.

**Completion criteria.** This phase has two distinct completion criteria, and the two must not be conflated. Architecture-planning completion requires a reviewed and accepted validation strategy that defines the properties, scenarios, measurements, and evidence a future validation effort must produce; accepting that document is what completes this phase as architecture. The implementation program's Phase 10, by contrast, is not complete until reference integrations exist, the validation described in the strategy has actually been executed, and empirical evidence supports every production-readiness claim — a state this roadmap does not reach and does not claim to reach by accepting the strategy document.

**Operator-mode representation.** Degraded-state status for interoperability: which producers, integrations, or contract guarantees are currently degraded.

**Training-mode explanation.** Safe failure demonstrations showing what degraded interoperability means and which guarantees remain intact during a given degradation.

**Evidence and audit requirements.** Test evidence for every claimed bound or guarantee, retained as the basis for any future claim that a contract surface is production-ready.

**Schema and documentation impact.** None directly; this phase validates prior phases' conceptual guarantees rather than introducing new contract surfaces.

**Explicitly deferred work.** All actual validation execution, which depends on reference integrations from Phase 9 that do not yet exist.

**Engineering experience developed.** Failure-mode and adversarial validation design for a cross-organizational contract, where the systems under test are not all owned by the same party running the validation.

---

## Security and Abuse Cases

The following threats are named at the scope of this roadmap's contract and are not fully addressed by any single phase above; each recurs across the phases named in parentheses and should be re-examined as each phase's architecture solidifies, consistent with how [`docs/security/threat-model.md`](../security/threat-model.md) already treats cross-cutting threats for the existing ecosystem.

Producer impersonation (Phases 2, 5, 8) is primarily Phase 5's concern but recurs anywhere a producer's identity is relied upon. Signature or digest substitution (Phase 5) and replay (Phases 5, 8) are Phase 5's central concerns, tested in Phase 8. Duplicate command execution (Phases 4, 5) spans the execution-lifecycle model and the transport-layer idempotency requirements. Confused deputy (Phase 2) and delegation escalation (Phase 2) are Phase 2's central concerns, extending the same threats the expansion roadmap's Phase 3 already names for BASIS-internal delegation to cross-organizational producers. Shared-account ambiguity (Phase 2) is named explicitly as a category the contract must represent honestly rather than resolve by assumption. False execution success (Phase 4) and missing execution evidence (Phase 4) are Phase 4's central concerns. Tampered evidence (Phases 5, 10) and reordered events (Phases 4, 5, 10) recur across the transport, lifecycle, and validation phases. Stale policy or trust configuration (Phases 2, 5, 7) is a recurring concern wherever a producer or an integration relies on configuration that could have changed since it was last refreshed. Malicious extension fields (Phase 7) are Phase 7's concern, guarded by its extension-namespace discipline. Protocol downgrade (Phase 6) is named explicitly as an integration-profile threat. Cross-environment or cross-tenant confusion (Phase 10) extends the tenant-isolation concerns the expansion roadmap's Phase 1 already names, to the case where the confusion originates from an external producer rather than from within BASIS. Sensitive operational-context leakage (Phases 3, 7) recurs wherever operational context — safety state, location, device state — is carried across a system boundary that may not share BASIS's own redaction discipline. Denial of service through oversized or recursively nested contract data (Phase 5) is named as a transport-layer concern requiring bounded validation before a message is trusted with further processing. An integration claiming compatibility while omitting required evidence (Phases 6, 8, 9) is named explicitly as the abuse case Phase 8's conformance strategy and Phase 9's maturity-level model exist specifically to catch.

This roadmap does not attempt to fully resolve any of these threats here. It identifies where each must be addressed and notes that [`docs/security/threat-model.md`](../security/threat-model.md) will require updates as each phase's architecture solidifies, in the same way the expansion roadmap already commits to for its own twelve phases.

---

## Required Decision Gates

The following decisions are recorded as unresolved unless existing architecture has already decided them. For each, this section states what evidence would be needed to resolve it, rather than proposing an answer.

**One composite contract versus a family of linked contracts.** Resolved by Phase 7's readiness work, informed by whether the Contract Responsibility Model's categories (identity, authority, delegation, producer, operation, resource, context, decision, enforcement, execution, evidence) prove separable in practice or tend to be needed together.

**Request/decision/enforcement/execution event separation.** Resolved by Phase 4's lifecycle model, informed by whether a single-record representation or a multi-record, correlated representation better serves the asynchronous and partial-execution cases that model names.

**How delegation chains are represented.** Resolved by Phase 2, informed by reconciliation with the expansion roadmap's Phase 3 delegation model and by whatever cross-trust-domain cases Phase 2's discovery work surfaces.

**How producer identity and producer trust are represented.** Resolved by Phase 2 and Phase 5 jointly, informed by the producer-enrollment and trust-classification requirements those phases develop.

**How authority differs from authenticated identity.** Resolved by Phase 2, informed by the same reconciliation with the expansion roadmap's identity model.

**How execution result vocabulary is governed.** Resolved by Phase 4, informed by whether a closed enum or an open extensible vocabulary better serves the range of native OT execution outcomes Phase 6's integration profiles surface.

**Whether integrity metadata is embedded or detached.** Resolved by Phase 5, informed by a future technology evaluation run against the requirements that phase states.

**Which signing or integrity standards should be evaluated.** Resolved by a future technology-evaluation phase, informed by Phase 5's requirements and conducted with the same evidence-based discipline the expansion roadmap's Phase 10 already establishes for authorization-technology comparison.

**How replay and idempotency identifiers interact.** Resolved by Phase 5, informed by Phase 3's operation-identity concept and Phase 8's conformance scenarios.

**How late, duplicate, and out-of-order events are reconciled.** Resolved by Phase 5 and validated by Phase 10, informed by store-and-forward and air-gapped delivery testing.

**Whether event order is global, per session, per producer, or per operation chain.** Resolved by Phase 4 and Phase 5 jointly, informed by which ordering guarantee the lifecycle model actually depends on versus which it merely finds convenient.

**How contract extensions avoid vendor-specific semantic fragmentation.** Resolved by Phase 7's extension-namespace discipline, informed by the reserved-namespace precedent already established in `basis-gateway`'s `basis_gateway.*` evidence namespace.

**How privacy and redaction work across identity and OT context.** Resolved by Phase 7, informed by the redaction-tier discipline already established for operation-aware trace and audit evidence, extended to the cross-system case.

**How air-gapped systems exchange contract data safely.** Resolved by Phase 5, informed by store-and-forward testing in Phase 10 and by consistency with the air-gapped distribution requirements the expansion roadmap's Phase 11 already names for policy distribution.

**What minimum conformance is required to call an integration BASIS-compatible.** Resolved by Phase 8 and Phase 9 jointly, informed by the conformance-scenario set Phase 8 defines.

**When architecture is stable enough to propose new `basis-schemas` contracts.** Resolved by Phase 7's readiness gate, informed by the minimum-evidence standard that phase states.

---

## Console Education Matrix

| Capability | Operator mode | Training mode |
| --- | --- | --- |
| Identity chain | Effective identity and authority for a given operation | Authentication, delegation, and provenance behind it |
| Producer trust | Producer and trust status | Why a producer's assertions are or are not trusted |
| Operation | Canonical action and resource | Protocol-to-operation normalization |
| Decision | Outcome and disposition | Kernel evaluation and gateway enforcement |
| Execution | Attempt/result state | Why authorization is not proof of execution |
| Integrity | Verified/unverified status | Transport versus message integrity |
| Contract version | Active contract version | Compatibility and version negotiation |
| Integration status | Maturity level and conformance status | What a conformance claim does and does not guarantee |
| Failure | Actionable degraded state | Which boundary failed and what remains trustworthy |

Detailed explanations of each row belong in the corresponding phase section above; this table exists to make the operator/training split scannable at a glance, not to substitute for those sections.

---

## Relationship to Existing Roadmaps

This roadmap and [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) govern different, complementary scopes, and neither supersedes the other.

The existing expansion roadmap governs identity-platform expansion within the BASIS ecosystem: multi-tenancy, token exchange and delegation, workload and non-human identity, distributed session lifecycle, SCIM synchronization, relationship-based authorization, fine-grained authorization queries, runtime gateway/core integration of relationship sub-decisions, external authorization-technology evaluation, and signed policy and configuration distribution. Its scope is how BASIS itself scales as an identity and authorization platform.

This roadmap governs cross-system identity-to-operation semantics and interoperability: how BASIS's account of an operation — identity, authority, delegation, producer, operation, resource, context, decision, enforcement, and execution — can be expressed, transported, and verified in a form that systems outside BASIS can consume and extend. Its scope is how BASIS's account of what happened becomes legible and interoperable beyond BASIS's own boundary.

The two roadmaps intersect at several points, named explicitly so a future implementer does not have to rediscover them: delegation (this roadmap's Phase 2 builds on the expansion roadmap's Phase 3), workload identity (this roadmap's Phase 2 builds on the expansion roadmap's Phase 4), relationship context (a future relationship sub-decision, per the expansion roadmap's Phase 7 through 9, is a candidate input to this roadmap's operational-context category), gateway composition (both roadmaps extend what `basis-gateway` composes into a request, and any implementation must reconcile the two rather than composing twice), policy and model versioning (this roadmap's Phase 7 should stay consistent with the versioning discipline the expansion roadmap's own contracts eventually adopt), and signed distribution (this roadmap's Phase 5 transport-and-integrity requirements and the expansion roadmap's Phase 11 signed-policy-distribution requirements should converge on compatible mechanisms rather than each inventing its own).

Overlapping decisions between the two roadmaps must be reconciled through the ADR process described in [`GOVERNANCE.md`](../../GOVERNANCE.md), not independently redefined by either roadmap or by whichever implementation phase happens to reach the overlap first.

A future identity-activity roadmap — if the ecosystem pursues one, covering the durable historical record of identity and operation activity over time, as distinct from the point-in-time contract this roadmap defines — would depend on the evidence and execution-lifecycle architecture this roadmap establishes. No such roadmap exists today, and this document does not propose one; it records the dependency so that if such a roadmap is later written, it is not designed independently of the evidence model this roadmap develops.

---

## Open-Source and Commercial Strategy

Open source can reduce licensing and roadmap barriers to adopting an identity-to-operation contract; it does not reduce deployment cost. Organizations pursuing this roadmap's eventual contract would still invest in integration work, adapter development, deployment, policy design, operations, maintenance, evidence storage, analytics, and support, in the same way they already do for the existing BASIS Core Services Distribution.

The open-source value this roadmap's contract could provide is that integrators and operators can fill unsupported OT identity gaps — a protocol without a maintained integration, a PAM system without a producer profile, a regulatory context without a sector-specific extension — without waiting for a proprietary vendor's roadmap to reach it. Potential ecosystem contributions, consistent with Phase 9's adoption model, may include Niagara modules, BACnet gateways, Modbus proxies, OPC UA normalization, identity-provider integrations, PAM and jump-host integrations, SIEM exporters, asset-inventory connectors, sector-specific context extensions, and conformance fixtures. None of these are committed to by this roadmap; they are named as the kinds of contribution the open contract is designed to make possible.

Potential commercial capabilities, consistent with the open-contract invariant above and the existing BASAuth boundary in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md), may include certified integrations, enterprise support, managed deployment, hosted evidence and analytics, compliance reporting, policy packs, upgrade assurance, and advanced detection content. Commercial capabilities may build on the open contract; they must not make the foundational contract itself proprietary, and no phase of this roadmap should be read as creating a commercial dependency for basic interoperability.

This section is deliberately restrained. It exists to record the boundary between open contract and commercial layer this roadmap depends on, not to develop a business plan.

---

## Sequencing and Dependency Guidance

The expected high-level sequence is:

```text
Phase 1: existing contract and evidence inventory
    (architecture discovery only — may proceed independently
    or in parallel, when architecture capacity permits;
    not gated on the program below)

Complete current basis-gateway operation-aware integration,
audit-evidence, readiness, conformance, and release-hardening program
    (the ecosystem's active implementation priority; no phase
    of this roadmap is currently in progress)
    ↓
Phase 2: identity, authority, delegation, and producer model
    ↓
Phase 3: canonical operation and resource semantics
    ↓
Phase 4: enforcement and execution lifecycle
    ↓
Phase 5: producer authentication, integrity, and transport profiles
    ↓
Phase 6: adapter, plugin, proxy, and gateway integration profiles
    ↓
Phase 7: contract versioning and schema readiness
    ↓
Phase 8: interoperability conformance
    ↓
Phase 9: reference integrations and adoption
    ↓
Phase 10: failure, isolation, and security validation
```

This sequence is provisional. It reflects the dependency structure named in each phase's Prerequisites subsection above, not a committed schedule or an exact pull-request count. Phase 1 is set apart from the chain below it because it is architecture discovery only — cataloguing what already exists — and carries no dependency on the current gateway program; it is drawn this way specifically so the diagram does not read as placing discovery work behind that program's completion, which would contradict the Prerequisites stated in Phase 1 above. Phases 2 through 10 remain gated on that program reaching its intended completion point, as the Status section at the top of this document states, and nothing in this diagram authorizes any of them to begin now. Architecture discoveries made during any phase may still justify splitting a phase, combining two phases, or reordering later phases — Phase 6's integration profiles, for instance, could reasonably begin in parallel with Phase 5 once Phase 3 and Phase 4 are stable, since integration profiles depend more on the operation and lifecycle model than on the transport mechanism. Any such change to phase sequencing that affects a stable component boundary or an established contract should go through the ADR process described in [`GOVERNANCE.md`](../../GOVERNANCE.md), the same discipline [`ROADMAP.md`](../../ROADMAP.md) already applies to changes in architectural intent.

To be explicit about what this sequencing does and does not authorize: no phase of this roadmap is currently in progress; the `basis-gateway` operation-aware integration program — including PR 7 and the rest of that program — remains the ecosystem's active implementation priority and is not reordered, paused, or deprioritized by this diagram; Phase 1 is architecture discovery only and produces no implementation; and Phases 2 through 10 stay gated on that program's completion exactly as stated elsewhere in this document. This roadmap does not authorize implementation of any phase.

---

## Deferred Decisions

The following decisions are intentionally left open beyond what **Required Decision Gates** above already states, until implementation planning begins for the relevant phase.

**Final selection of a signing or message-integrity standard.** Resolved by a future technology evaluation conducted with the same evidence-based methodology the expansion roadmap's Phase 10 already establishes, run against the requirements Phase 5 defines.

**Final transport and message-broker technology.** Resolved once Phase 5's requirements and a future technology evaluation are both complete; this roadmap does not name or imply a preferred technology anywhere above.

**Final repository or repositories that would implement this roadmap's contract.** Not resolved by this roadmap. Any future repository name is conceptual until it is actually established, consistent with the Status stated at the top of this document.

**Exact schema-proposal timing for any contract this roadmap anticipates.** Resolved individually, phase by phase, following the same readiness discipline ADR-0005 already established: a contract is proposed to `basis-schemas` only after its architecture and a reference implementation exist, never speculatively ahead of them.

**Certification or conformance-claim governance.** Resolved as part of Phase 9's adoption model, informed by however Foundation governance itself matures.

**Production availability or performance targets for the contract's transport and evidence layers.** Resolved by Phase 10's validation work once reference integrations exist to validate against, not asserted in advance of that testing.

---

## Summary

This roadmap preserves the long-term architectural direction for BASIS as an open identity-to-operation contract and interoperability layer for fragmented OT environments, while the current `basis-gateway` operation-aware integration, audit-evidence, readiness, conformance, and release-hardening program continues as the ecosystem's present priority. It states a central thesis — that BASIS could provide the connective semantic chain from verified identity through authorization, enforcement, execution, and evidence that fragmented OT identity, PAM, gateway, adapter, and evidence systems currently lack — without claiming that thesis as anything more than an analogy of architectural role or that any of it is implemented today. It defines architectural invariants that extend, without relaxing, the kernel isolation, identity/authorization separation, and evidence discipline the existing architecture already establishes. It defines ten phases covering inventory, identity and producer modeling, operation and resource semantics, execution lifecycle, transport and integrity, integration profiles, versioning readiness, conformance, adoption, and validation, each stating its own security and abuse cases, distributed-systems concerns, decision gates, and operator/training-mode requirements without prescribing implementation technology or a final schema. It names its relationship to the identity and fine-grained authorization expansion roadmap explicitly, so the two do not drift into redefining the same concepts independently. And it remains explicitly provisional: the phase structure, sequencing, and open decision gates recorded here are architectural intent as understood today, subject to revision through the same ADR process that governs every other consequential decision in this repository.

---

## Related Documents

- [`ROADMAP.md`](../../ROADMAP.md) — the ecosystem's current phase structure, status, and the active `basis-gateway` operation-aware integration program this roadmap's Status section depends on
- [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) — the companion roadmap for BASIS-internal identity and authorization platform expansion; see **Relationship to Existing Roadmaps** above for the boundary between the two
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction this roadmap's invariants extend
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules the Kernel invariant above must not violate
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the existing cross-repository contract inventory Phase 1 extends
- [`docs/architecture/operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md), [`docs/architecture/operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md), [`docs/architecture/operation-aware-policy-rule-model.md`](../architecture/operation-aware-policy-rule-model.md), and [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) — the governed operation-aware vocabulary Phase 3 composes without redefining
- [`docs/architecture/operation-aware-schema-readiness-plan.md`](../architecture/operation-aware-schema-readiness-plan.md) and [ADR-0005](../adr/0005-operation-aware-schema-readiness.md) — the readiness discipline Phase 7 models its own gate on
- [`docs/architecture/basis-console.md`](../architecture/basis-console.md) — the console architecture and operator/training-mode invariants this roadmap's console requirements extend
- [`docs/security/threat-model.md`](../security/threat-model.md) — the existing threat model that Phase-specific work must update as each phase's architecture solidifies
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR process this roadmap's decision gates and sequencing changes must follow
- [`docs/adr/README.md`](../adr/README.md) — when a phase's architecture work requires a new ADR
- [`docs/glossary.md`](../glossary.md) — the canonical terminology reference this roadmap's working vocabulary should graduate into as each phase stabilizes
