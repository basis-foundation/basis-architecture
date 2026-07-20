# Identity and Fine-Grained Authorization Expansion Roadmap

This document defines the architectural direction for the next major expansion of the BASIS ecosystem: deeper identity-platform capability, fine-grained authorization, relationship-based access control, distributed policy operation, and failure-resilient multi-tenant services. It is an architecture-planning document, not an implementation plan. It assigns capabilities to architectural boundaries, records the decision gates that must be resolved before implementation can begin, and states completion criteria in terms of testable outcomes — not in terms of chosen storage technologies, database schemas, source-file layouts, or final public APIs.

**Status:** Planned — implementation begins after the current `basis-core` operation-aware authorization roadmap (defined in `ROADMAP.md` Phase 2 and the `basis-core` v0.2.0 implementation program) reaches its intended completion point. Nothing in this document authorizes work to begin before that point, and no phase described here is in progress.

This roadmap does not implement anything. It does not modify `basis-core`, `basis-identity`, `basis-gateway`, `basis-console`, or any other implementation repository. It does not add dependencies, choose a relationship-authorization engine, or select a storage or caching technology. Where a decision has not been made, this document says so explicitly and records what evidence would be needed to make it, consistent with how `ROADMAP.md` distinguishes "In architecture" and "Research direction" items from released work.

A note on terminology: this roadmap introduces working vocabulary for concepts — tenant isolation, token exchange, workload identity, relationship-based authorization, fine-grained authorization queries, policy distribution — that do not yet have entries in [`docs/glossary.md`](../glossary.md). Consistent with the glossary's role as a canonical reference for decided terminology rather than speculative vocabulary, formal glossary entries for these terms should be added as each phase's architecture stabilizes through its own ADR, not as part of this roadmap.

---

## Purpose

The current BASIS Core Services Distribution establishes a working authorization kernel, a runtime enforcement boundary, a protocol normalization layer, an operator interface, and an identity engine that federates external providers into a canonical identity context. That architecture answers a bounded question well: given a normalized subject, resource, and action, does policy permit the request, and is that decision auditable?

The capabilities named in this roadmap extend the ecosystem toward a set of harder, adjacent questions that a production identity-aware authorization platform eventually has to answer: how does the platform serve more than one tenant without leaking trust across the tenant boundary; how does it cache identity and policy state without serving stale or poisoned data; how does it let one system act on behalf of another without inventing a confused-deputy vulnerability; how does it represent workloads, devices, and AI agents as first-class subjects alongside humans; how does it revoke a session or a credential across a distributed deployment within a bounded time; how does it keep a synchronized identity registry consistent with an authoritative source under retries and partial failures; how does it answer "can this subject reach this resource, and through what relationship" when the answer depends on organizational structure rather than a static rule; and how does it do all of this while remaining explainable to the operators who are accountable for it.

The purpose of this roadmap is not to collect the names of technologies that solve pieces of this problem. It is to build coherent, production-relevant platform capability — multi-tenant identity services, tenant-isolated trust configuration, distributed session lifecycle, relationship-based authorization, fine-grained authorization queries, signed policy distribution, and the fault-tolerance and adversarial testing that make claims about any of it credible — while preserving the component boundaries that keep `basis-core` a small, stable, portable kernel and keep `basis-identity` an identity engine rather than an authorization authority.

---

## Architectural Invariants

The following boundaries are preconditions for every phase in this roadmap. They restate and extend boundaries already established in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), and [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md). Nothing in this roadmap supersedes those documents; where language differs, this section is written to be additive, not contradictory. Any phase whose implementation would require relaxing one of these invariants should be treated as evidence that the phase has been scoped to the wrong component, not as grounds for an exception.

### Identity Boundary

`basis-identity` may federate with external identity providers, validate identity credentials, normalize identity into canonical form, manage BASIS-local sessions, exchange credentials, represent delegation, issue BASIS-local credentials, synchronize identity lifecycle data from an authoritative source, and support explicitly configured standalone or air-gapped identity authority modes.

`basis-identity` must not evaluate authorization policy, determine whether an OT operation is allowed, own relationship-based authorization semantics, replace `basis-core`, become the gateway enforcement boundary, or normalize OT protocols. Multi-tenant trust configuration, token exchange, workload identity, distributed session revocation, and SCIM synchronization — Phases 1 through 6 below — extend what `basis-identity` federates, brokers, and normalizes. They do not extend it into deciding what a subject may do.

### Authorization Boundary

`basis-core` owns deterministic authorization semantics, including the final operation-aware authorization decision. It must not authenticate users, parse external identity-provider protocols, manage sessions, fetch discovery metadata, own runtime HTTP enforcement, or become an identity directory. Fine-grained authorization queries — Phase 8 below — may resolve a bounded relationship sub-decision that a relationship authorization capability owns evaluating; `basis-core` may consume that sub-decision as relationship evaluation evidence at a defined extension point, alongside identity, policy, and operation-aware context, when it reaches the final decision. See the Relationship Data Boundary below for the full distinction between relationship-model evaluation and the final authorization decision. None of this relocates identity federation, session management, or directory synchronization into the kernel; none of it makes `basis-core` the store of record for relationship data; and none of it gives `basis-core` its own graph persistence or network-backed relationship traversal.

### Gateway Boundary

`basis-gateway` owns the runtime enforcement boundary. It may validate trusted BASIS credentials, compose authorization requests, invoke authorization capabilities, enforce returned decisions, and emit runtime evidence. It must not invent authorization semantics, become the authoritative relationship store, redefine identity contracts, or reinterpret kernel decisions. Runtime integration of identity, relationship, and policy-distribution capability — Phase 9 below — extends what the gateway composes into a request. It does not extend what the gateway is authoritative for.

### Relationship Data Boundary

Relationship tuples and authorization graph data — the facts that describe which organizations, facilities, systems, devices, users, groups, and service identities stand in what relation to one another — are authorization data, not identity data. `basis-identity` must not become the relationship store merely because relationships often reference identities.

Where this capability should live is a decision gate, not a decision made by this roadmap. Phase 7 below evaluates at least three candidate architectures — a narrowly bounded new repository such as `basis-relationships`, an extension provider consumed by `basis-core` at a defined evaluation extension point, or another architecture that later discovery justifies — and records the evidence that must be gathered before a repository decision is made. This roadmap does not assume `basis-core` becomes a relationship-based authorization (ReBAC) engine in place, and it does not assume a new repository is warranted. Both are live options pending Phase 7's discovery work.

Independent of where relationship data and evaluation eventually live, the relationship authorization capability and `basis-core` divide two distinct layers of the same problem, and this division holds regardless of which repository or extension architecture Phase 7 eventually selects:

```text
Relationship authorization capability
    owns relationship-model evaluation
    returns a bounded relationship sub-decision

basis-core
    owns the final operation-aware authorization decision
```

A relationship result may answer a bounded predicate — for example, whether `subject is_operator_for building-7` — and that predicate result is a relationship sub-decision, not a complete authorization outcome. It must not independently answer the full BASIS operation question — for example, whether a subject may write a chiller setpoint under the current operation, context, policy, and safety state. `basis-core` remains the sole authority that composes a relationship sub-decision with identity context, policy, operation-aware context, and safety/risk context to reach that final decision, and it does so without acquiring graph persistence, network-backed relationship traversal, or a dependency on a specific relationship-authorization technology such as OpenFGA or SpiceDB. A relationship sub-decision that `basis-core` relies on must be bounded, attributable, versioned, and safely explainable; an unbounded, unattributed, or unversioned result is not a legitimate evaluation input regardless of which architecture produced it. This distinction governs Phase 8's query outcomes and Phase 9's integration model below.

### Schema Boundary

`basis-schemas` publishes contracts only after they have stabilized through architecture and implementation, following the same readiness discipline ADR-0005 established for the operation-aware contract suite. `basis-schemas` must not invent speculative contracts for the capabilities named in this roadmap. No phase in this roadmap should be read as directing `basis-schemas` to publish a contract before the corresponding architecture and reference implementation exist.

### Console Boundary

`basis-console` observes, submits, displays, and explains. It must not authenticate users, issue credentials, evaluate authorization, own relationship data, store authoritative identity state, or become an alternate implementation path for any capability described in this roadmap. Every phase's operator-mode and training-mode requirements below extend what the console renders and explains. None of them extend what the console decides or stores.

### AI Boundary

AI may explain, summarize, propose, assist with diagnostics, and prepare policy or access recommendations. AI must not become the authorization authority. Where a future phase introduces AI-assisted diagnostics, relationship-path explanation, or policy-recommendation tooling, the invariant that governs it is unchanged from the invariant that governs every other component in the ecosystem:

```text
AI suggests.
Humans approve.
Gateway enforces.
Core evaluates.
```

---

## Training-Mode Constitutional Requirement

Every externally observable identity or authorization capability introduced through this roadmap must have a corresponding operator-facing representation and training-mode explanation in `basis-console`. Training mode is a design requirement for each capability from the outset, not cosmetic UI work added after a capability ships. A phase whose implementation plan does not include its console representation is not a complete plan.

Training mode may explain trust boundaries, identity provenance, credential transformation, request composition, delegation chains, relationship traversal, cache behavior, consistency guarantees, policy revisions, synchronization, retries, failure states, and audit evidence. Training mode must not change backend behavior, bypass authentication or authorization, use alternate business logic, fabricate live data, expose secrets or unredacted credentials, or become a separate application. This extends, without altering, the console invariant already established in [`docs/architecture/basis-console.md`](../architecture/basis-console.md):

```text
Operator mode may simplify.
Training mode may educate.
Neither mode may change system behavior.
```

For every phase in this roadmap, the phase definition answers six questions: what operator mode needs to show; what training mode needs to explain; which backend component owns the behavior being explained; what evidence proves the behavior occurred; what the user sees when the capability fails; and which sensitive details must be redacted. These six questions appear as the **Operator-mode representation**, **Training-mode explanation**, and **Evidence and audit requirements** subsections of each phase below, together with failure-state and redaction notes folded into those same subsections.

---

## Roadmap Phases

The following twelve phases are architecture phases, not a predetermined pull-request schedule. Each phase uses a consistent structure: purpose, primary repositories, prerequisites, architectural outcome, key capabilities, security and abuse cases, distributed-systems concerns, decision gates, completion criteria, operator-mode representation, training-mode explanation, evidence and audit requirements, schema and documentation impact, explicitly deferred work, and engineering experience developed. Subsections vary in depth according to how much is already known; none are omitted.

### Phase 1 — Multi-Tenant Identity and Trust Model

**Purpose.** Define the architectural outcome for first-class tenant isolation in `basis-identity`, so that a single `basis-identity` deployment can serve multiple tenants without any tenant's trust configuration being reachable, inferable, or substitutable by another tenant's requests.

**Primary repositories.** `basis-identity` (tenant-scoped provider configuration, issuer/audience isolation, session-policy isolation); `basis-architecture` (this roadmap and its follow-on ADR); `basis-console` (tenant-aware diagnostics display).

**Prerequisites.** Completion of the current `basis-core` operation-aware roadmap; a stable canonical identity context contract to extend with a tenant dimension; no dependency on Phases 2 or later.

**Architectural outcome.** Tenancy becomes a trust boundary, not merely an optional string field on an identity record. Every trust decision `basis-identity` makes — which issuer is accepted, which signing keys are trusted, which session policy applies, which provider configuration revision is active — is resolved with the tenant as an input to that decision, not as metadata attached after the decision is made. A request that supplies credentials valid for one tenant must be structurally incapable of being evaluated against another tenant's trust configuration, regardless of how the request is composed.

**Key capabilities.** Tenant-scoped identity-provider configuration, so that each tenant may point at its own upstream provider or providers independently of every other tenant; issuer and audience isolation, so that a token's issuer and audience are validated against the specific tenant's configuration and never against a global set; signing-trust isolation, so that a tenant's trusted signing keys are never implicitly available to another tenant's validation path; session-policy isolation, so that idle and absolute session lifetimes, and any tenant-specific session rules, are configured and enforced per tenant; provider-configuration versioning, so that a tenant's trust configuration has an auditable history and a defined active revision; rollout and rollback expectations for configuration changes, so that a misconfigured trust change can be reversed without cross-tenant impact; and tenant-aware diagnostics, so that identity diagnostics never conflate or leak configuration details across tenant boundaries.

**Security and abuse cases.** Cross-tenant credential acceptance, where a token minted for tenant A is mistakenly accepted under tenant B's trust configuration; issuer confusion, where two tenants share an upstream IdP but are not correctly distinguished by audience or claim; configuration rollback attacks, where an attacker forces a tenant back to a previous, weaker trust configuration; and diagnostic leakage, where tenant-aware diagnostics inadvertently reveal another tenant's provider configuration or issuer details.

**Distributed-systems concerns.** Configuration propagation latency across however many `basis-identity` instances serve a tenant; the consistency model for a mid-flight trust-configuration change (a session established before a change should not silently break, but must not outlive the security intent of the change); and coordination between configuration versioning here and the caching behavior defined in Phase 2.

**Decision gates.** Whether tenant isolation is enforced primarily through configuration namespacing, structural request routing, or both; whether tenant identifiers are opaque or hierarchical; and how many tenants a single deployment is expected to support before a different scaling architecture is warranted. None of these are resolved by this roadmap.

**Completion criteria.** A defined, testable set of cross-tenant isolation properties (a credential valid for tenant A is provably rejected under tenant B's configuration in every code path that performs validation); a documented configuration-versioning and rollback model; and tenant-aware diagnostics that pass a review confirming no cross-tenant leakage, before this phase is considered architecturally complete.

**Operator-mode representation.** The active tenant, the provider(s) trusted for that tenant, and the health of that tenant's trust configuration.

**Training-mode explanation.** Why a credential from another tenant was rejected, which issuer boundary applied, and which configuration revision was in effect at decision time.

**Evidence and audit requirements.** Every trust-configuration change is versioned and attributable; every rejection due to tenant-boundary mismatch is distinguishable in evidence from an ordinary authentication failure, so operators and auditors can tell tenant-isolation enforcement apart from routine credential rejection.

**Schema and documentation impact.** A future tenant dimension on the canonical identity context is a candidate `basis-schemas` contract change, to be proposed only after this phase's architecture and a reference implementation exist — not before.

**Explicitly deferred work.** Final tenant-identifier format; storage of tenant configuration; whether tenant configuration is itself distributed via the mechanism defined in Phase 11.

**Engineering experience developed.** Trust-boundary design as a first-class architectural concern, not an access-control afterthought; multi-tenant configuration versioning and rollback discipline.

### Phase 2 — Tenant-Isolated Caches

**Purpose.** Define expectations for caching in a multi-tenant `basis-identity` deployment — OIDC discovery, JWKS, and configuration caches — so that caching improves performance without becoming a cross-tenant leakage path or a source of silently stale trust decisions.

**Primary repositories.** `basis-identity` (cache implementation and invalidation); `basis-console` (cache observability).

**Prerequisites.** Phase 1's tenant trust model, since cache keys must be tenant-scoped to mean anything.

**Architectural outcome.** Every cache that participates in a trust decision — discovery metadata, JWKS, and tenant configuration — is keyed in a way that makes cross-tenant cache hits structurally impossible, has a defined expiration and refresh policy, and has a defined answer for what happens when a refresh fails: whether the cache serves last-known-good data, refuses to serve, or does something else, and for how long that behavior is permitted to continue before it becomes a security problem rather than a resilience feature.

**Key capabilities.** Tenant-scoped cache keys for discovery, JWKS, and configuration data; expiration and refresh behavior with a documented refresh trigger (time-based, event-based, or both); stale-data behavior that is an explicit, reviewed decision rather than an implementation accident; negative caching for known-bad lookups, bounded so it cannot itself become a denial-of-service vector; cache-poisoning prevention, including validation of anything fetched before it enters the cache; key rotation handling, so that a signing-key rotation at the upstream provider is reflected within a bounded time; and cache observability, so operators can see hit/miss rates, refresh events, and staleness without exposing cached secret material.

**Security and abuse cases.** Cache poisoning, where an attacker causes malicious or stale key material to be cached and subsequently trusted; cache-timing side channels that could reveal tenant existence or configuration shape; and denial of service through cache-bypass request patterns that force expensive upstream fetches.

**Distributed-systems concerns.** In-process versus shared caching in a multi-instance deployment; invalidation propagation across instances when a key or configuration changes; and the fail-open-versus-fail-closed choice when a cache cannot be refreshed and the upstream provider is unreachable — this is the same tension named as an open question in `ROADMAP.md` Phase 4 for gateway-level policy caching, and this phase should stay consistent with whatever resolution that broader question eventually reaches, rather than resolving it independently for identity caches alone.

**Decision gates.** In-process versus shared (e.g., distributed) caching; the specific invalidation mechanism (push notification, short TTL with revalidation, or a hybrid); and whether last-known-good behavior on refresh failure is fail-open or fail-closed by default, and whether that default is deployment-configurable.

**Completion criteria.** A documented cache-key isolation model verified against Phase 1's tenant boundary; a documented and tested behavior for refresh failure; and cache observability that a training-mode session can walk an operator through without exposing key material.

**Operator-mode representation.** Cache health and freshness per tenant and per cache type.

**Training-mode explanation.** What a cache hit, miss, and refresh look like; how cache keys are isolated by tenant; and what stale-state behavior means when an upstream provider cannot be reached.

**Evidence and audit requirements.** Cache refresh failures and stale-data fallback events are recorded as evidence distinguishable from ordinary trust decisions, so a denied request caused by stale key material is not indistinguishable from a denied request caused by an invalid credential.

**Schema and documentation impact.** None anticipated at the contract level; this phase is primarily an internal `basis-identity` behavior specification.

**Explicitly deferred work.** Specific cache technology; specific invalidation transport (event bus, change feed, or polling).

**Engineering experience developed.** Cache architecture and invalidation design under multi-tenant and security constraints simultaneously — a harder problem than either constraint alone.

### Phase 3 — Token Exchange and Delegation

**Purpose.** Define a future standards-aware token-exchange capability that lets one verified identity act on behalf of, or with the delegated authority of, another — the mechanism that later phases (workload identity, gateway integration) depend on to represent "this workload, acting for this user" or "this automation, acting within this scope" without inventing an ad hoc trust relationship for each case.

**Primary repositories.** `basis-identity` (token exchange, delegation representation); `basis-gateway` (consumption of the resulting BASIS-local credential); `basis-console` (delegation visualization).

**Prerequisites.** Phase 1's tenant trust model, since exchanged tokens must remain tenant-scoped; a stable canonical identity context to extend with actor and delegation fields.

**Architectural outcome.** `basis-identity` can accept a subject credential and, where a deployment's policy permits it, an actor credential, and produce a BASIS-local credential that carries both the original subject and the delegation chain that produced the final credential — with audience binding and scope attenuation applied at each step, so that a delegated credential can never carry more authority than the credential it was derived from.

**Key capabilities.** Subject and actor credential handling; audience and resource binding, so an exchanged token is only valid for the resource it was exchanged for; scope attenuation, so delegation can only narrow authority, never widen it; delegation-chain representation, bounded in depth and fully attributable back to the originating subject; expiration constraints tied to the shortest-lived credential in the chain; provenance sufficient to answer "who is this really, and who said so" at every step; and replay protection for exchanged tokens.

**Security and abuse cases.** Privilege escalation through a delegation chain that widens rather than narrows scope; confused-deputy behavior, where a workload with broad access is tricked into acting with that access on behalf of a subject who should not have it; token replay across the exchange boundary; and delegation-chain forgery, where an actor claims a delegation it was never granted.

**Distributed-systems concerns.** Token-exchange latency added to the request path; and consistency between the delegation policy evaluated at exchange time and the delegation policy in effect if it changes before the exchanged token expires.

**Decision gates.** Final API shape for the exchange endpoint; how closely the design tracks OAuth 2.0 Token Exchange (RFC 8693) concepts versus deviating where OT delegation needs differ; and maximum permitted delegation-chain depth.

**Completion criteria.** A documented delegation model with a proof that scope can only attenuate, never escalate, across every step in a chain of at least the maximum depth chosen; and a reviewed threat analysis specifically covering confused-deputy scenarios.

**Operator-mode representation.** Delegated-identity status for the current session or request: whose authority is being exercised, and on whose behalf.

**Training-mode explanation.** A visualization of the chain from external credential through trust validation, canonical subject resolution, actor and delegation chain, scope attenuation, to the resulting BASIS-local credential and its enforcement at the gateway:

```text
External credential
    ↓
Trust validation
    ↓
Canonical subject
    ↓
Actor and delegation chain
    ↓
Scope attenuation
    ↓
BASIS-local credential
    ↓
Gateway enforcement
```

**Evidence and audit requirements.** Every exchange produces evidence of the subject, the actor (if any), the requested and granted scope, and the resulting audience binding, sufficient to reconstruct the full delegation chain after the fact.

**Schema and documentation impact.** A future BASIS-local delegated-credential contract is a candidate `basis-schemas` addition once this phase's reference implementation exists.

**Explicitly deferred work.** Final token format; whether exchange is synchronous or supports a pre-authorization flow; specific standards-track conformance claims.

**Engineering experience developed.** Secure token design, delegated authority modeling, and confused-deputy threat analysis under real constraints.

### Phase 4 — Workload and Non-Human Identity

**Purpose.** Define support for services, pipelines, gateways, adapters, automation jobs, OT workloads, devices where appropriate, and AI agents as canonical, first-class BASIS subjects, distinct from but interoperable with human identity.

**Primary repositories.** `basis-identity` (workload subject issuance and lifecycle); `basis-core` (workload subject already fits the existing `Subject` domain primitive — this phase is expected to extend how subjects are populated, not to change the primitive itself).

**Prerequisites.** Phase 3's delegation model, since human-on-behalf-of-workload and workload-on-behalf-of-human chains are a form of delegation.

**Architectural outcome.** A workload, device, or AI agent has a canonical BASIS subject representation with the same rigor as a human subject: a stable identifier, an owning tenant and environment context, audience-bound credentials, and a defined rotation and revocation lifecycle — without collapsing the meaningful distinctions between a human operator, a long-lived service, a short-lived automation job, and a physical device.

**Key capabilities.** Canonical non-human subject representation; workload ownership (which team, tenant, or system owns a given workload identity); tenant and environment context (production versus staging identity must not be interchangeable); audience-bound credentials scoped to the specific resource or service a workload needs to reach; credential rotation on a defined cadence; revocation with the same bounded-delay expectations as Phase 5's session revocation; human-on-behalf-of-workload and workload-on-behalf-of-human delegation chains, built on Phase 3; and potential interoperability with SPIFFE/SPIRE as an existing workload-identity standard, without `basis-identity` replacing or reimplementing it.

**Security and abuse cases.** Workload credential theft and reuse outside its intended environment; long-lived workload credentials that were never rotated becoming a persistent attack surface; and an AI agent's credential being used to exceed the scope a human approved for it — the AI boundary invariant applies to workload identity precisely because agents are workloads.

**Distributed-systems concerns.** Credential rotation without service interruption; and revocation propagation consistent with Phase 5.

**Decision gates.** Whether `basis-identity` issues its own workload credential format or defers entirely to SPIFFE/SPIRE-style attestation where it is already present in a deployment; and how device identity in this phase relates to the device-identity concepts already used in OT adapter contexts.

**Completion criteria.** A documented, testable distinction between human, workload, device, and delegated identity in the canonical identity context; and a rotation-and-revocation lifecycle proof for at least one workload identity class.

**Operator-mode representation.** The workload, its owner, and its current trust state (active, rotating, revoked).

**Training-mode explanation.** The distinction between human identity, workload identity, device identity, and delegated identity; how a workload's attestation or trust domain is established; and what rotation and delegation look like for a non-human subject.

**Evidence and audit requirements.** Workload credential issuance, rotation, and revocation events are evidenced with the same rigor as human session events.

**Schema and documentation impact.** A future workload-subject contract extension, deferred until reference implementation exists.

**Explicitly deferred work.** Final device-identity provisioning mechanism; whether SPIFFE/SPIRE integration is in-scope for the first implementation or a later increment.

**Engineering experience developed.** Non-human identity modeling, credential rotation design, and the discipline of keeping AI-agent identity subject to the same authority model as any other workload.

### Phase 5 — Distributed Session Revocation and Logout

**Purpose.** Define expected behavior for session and refresh-token lifecycle across a distributed `basis-identity` deployment, including revocation that must reach every node within a bounded delay even during a network partition.

**Primary repositories.** `basis-identity` (session store, revocation propagation); `basis-console` (revocation and partition-state visibility).

**Prerequisites.** Phase 1's tenant isolation, since revocation must respect tenant boundaries; the existing session-lifecycle work already underway in `basis-identity` (session store, expire/revoke/touch/refresh operations, cookie binding) as the concrete foundation this phase extends toward distributed operation.

**Architectural outcome.** A revocation decision — whether initiated by a user logging out, an administrator revoking a session, or a tenant-wide revocation event — reaches every node that could accept the affected credential within a documented, bounded delay, and the system's behavior during a network partition is a stated consistency model, not an unstated assumption.

**Key capabilities.** Refresh-token rotation and reuse detection; session families, so a compromised refresh token's blast radius is bounded to the family it belongs to; idle and absolute expiration; user-wide, tenant-wide, and administrative revocation; relying-party and back-channel logout; replay prevention; and revocation propagation across however many nodes serve a deployment.

**Security and abuse cases.** Refresh-token reuse after rotation, indicating token theft; session fixation; revocation bypass during a partition, where a node that has not yet received a revocation event continues to accept a credential that should be dead; and back-channel logout spoofing.

**Distributed-systems concerns.** This phase requires an explicit consistency model. Candidate models include bounded-staleness propagation (revocation is guaranteed to reach all nodes within a stated time bound, and that bound is itself a documented guarantee to operators) and strict consistency at the cost of availability during a partition. The chosen model must be stated, not implied, and must account for partition behavior: does a partitioned node fail closed (reject credentials it cannot confirm are unrevoked) or fail open (accept them, accepting the risk) — and is that choice deployment-configurable, matching the fail-open/fail-closed governance question `ROADMAP.md` Phase 4 already identifies as unresolved for the broader ecosystem.

**Decision gates.** The specific consistency model (bounded-staleness versus strict); the revocation-propagation transport; and whether the bounded revocation delay is a fixed architectural guarantee or a deployment-tunable parameter.

**Completion criteria.** A stated, tested consistency model with a measured propagation bound under simulated partition; and a demonstrated reuse-detection path for rotated refresh tokens.

**Operator-mode representation.** Active/revoked status for a session, with revocation origin when relevant.

**Training-mode explanation.** Session state, the origin of a revocation, propagation status across nodes, cache invalidation triggered by revocation, whether a specific node is currently relying on stale session state, and the bounded revocation delay the deployment guarantees.

**Evidence and audit requirements.** Every revocation event records its origin (user-initiated, administrative, tenant-wide), its propagation status, and the time bound within which it was expected to take effect everywhere.

**Schema and documentation impact.** A future session and revocation-event contract, deferred until reference implementation exists.

**Explicitly deferred work.** Final propagation transport (event bus, change feed, or gossip); exact bound value, which is a deployment and implementation decision, not an architectural one.

**Engineering experience developed.** Distributed consistency and revocation semantics, partition-tolerant design, and the discipline of stating a consistency guarantee precisely enough that it can be tested and falsified.

### Phase 6 — SCIM Synchronized Registry

**Purpose.** Define a synchronized identity-registry capability building on `basis-identity`'s existing synchronized-registry authority mode, so that a local registry can be kept consistent with an authoritative external source under retries, duplicate delivery, conflicting updates, and upstream outages.

**Primary repositories.** `basis-identity` (registry synchronization); `basis-console` (synchronization health display).

**Prerequisites.** Phase 1's tenant isolation, since a synchronized registry must not merge records across tenants; the existing Synchronized Registry Mode already defined in [`docs/architecture/identity-authority-modes.md`](../architecture/identity-authority-modes.md).

**Architectural outcome.** Users and groups synchronized into the local registry via create, replace, patch, deactivate, and delete operations remain consistent with the authoritative source even when synchronization events arrive out of order, are delivered more than once, or cannot be delivered for a period because the upstream source is unreachable — and the registry never silently diverges from that source, consistent with the non-divergence requirement already stated for Synchronized Registry Mode.

**Key capabilities.** User and group lifecycle operations (create, replace, patch, deactivate, delete); external-identifier mapping; idempotent handling of duplicate delivery; retry behavior with bounded backoff; conflict handling when two updates to the same record race; ordering guarantees, or an explicit statement of their absence, for out-of-order delivery; tombstones for deleted records, so a delete is distinguishable from a record that was never synchronized; reconciliation, a periodic full-state comparison that catches drift the event stream missed; and upstream-outage behavior, including how long the registry may serve last-known-good state before it must be flagged as unreliable.

**Security and abuse cases.** Synchronization poisoning, where a malformed or malicious synchronization event corrupts the local registry; identity duplication, where the same external identity is represented by two local records due to a mapping failure; replayed delete events resurrecting or re-deleting records incorrectly; and tenant-boundary violations during synchronization.

**Distributed-systems concerns.** Idempotency as a first-class design requirement, since SCIM-style synchronization protocols are expected to redeliver events; reconciliation frequency versus cost; and the upstream-outage degraded-mode decision (serve last-known-good, or refuse to serve stale registry data for identity decisions that matter).

**Decision gates.** Push-based synchronization versus periodic pull versus a hybrid; the specific conflict-resolution rule (last-write-wins, source-of-truth-always-wins, or a manual reconciliation queue); and how deeply automated reconciliation is trusted to auto-correct drift versus flagging it for operator review.

**Completion criteria.** A demonstrated idempotent handling of duplicate and out-of-order delivery; a documented conflict-resolution rule; and a tested reconciliation pass that detects and corrects an intentionally introduced drift scenario.

**Operator-mode representation.** Synchronization health: last successful sync, pending conflicts, and drift status.

**Training-mode explanation.** The distinction between source-of-truth and synchronized state; lifecycle events as they arrive; retries; conflicts; tombstones; reconciliation; and what last-known-good state means during an upstream outage.

**Evidence and audit requirements.** Every synchronization event, conflict, and reconciliation action is evidenced, sufficient to answer "why does this registry record look the way it does" after the fact.

**Schema and documentation impact.** A future synchronization-event contract, deferred until reference implementation exists; no final endpoint structure or persistence schema is defined here.

**Explicitly deferred work.** Final SCIM endpoint structure; persistence schema; specific reconciliation-frequency defaults.

**Engineering experience developed.** Reliable synchronization, idempotency, retries, and reconciliation under real-world delivery guarantees rather than idealized ones.

### Phase 7 — Relationship-Based Authorization Architecture

**Purpose.** Define the architectural discovery work required before implementing relationship-based access control (ReBAC) or fine-grained authorization (FGA) — deliberately a discovery phase, not an implementation phase, because the repository-ownership question named in the Relationship Data Boundary above must be resolved with evidence before implementation begins.

**Primary repositories.** `basis-architecture` (discovery and the resulting ADR); a to-be-determined repository or extension point, per the decision gate below.

**Prerequisites.** A stable operation-aware evaluation extension point in `basis-core` (expected to exist as a consequence of the current operation-aware roadmap); Phases 1 through 6, since relationships between tenants, workloads, and synchronized registry entries are inputs to this model.

**Architectural outcome.** A defined conceptual model for relationships among organizations, facilities, buildings, systems, controllers, devices, points, users, groups, contractors, and service identities — covering authorization models, relationship tuples, inheritance, nested groups, ownership, delegated administration, recursive traversal, cycle handling, maximum traversal depth, contextual relationships, and model versioning, with tenant isolation preserved throughout — without a final choice of storage technology, query language, or repository.

**Key capabilities.** A relationship-tuple model expressive enough to represent ownership, membership, and delegated administration across the entity types named above; inheritance and nested-group semantics; recursive traversal with a defined, enforced maximum depth; cycle detection, since organizational and facility hierarchies are not guaranteed to be acyclic in practice even when they are intended to be; contextual relationships (a relationship that only holds under certain conditions, distinct from a static tuple); and model versioning, so that a relationship model can evolve without breaking evaluations made against an earlier version.

**Security and abuse cases.** Relationship injection, where an attacker introduces a tuple that grants unintended access; authorization-graph cycles used to cause unbounded traversal or denial of service; stale relationship data producing an access grant that should have been revoked when an underlying relationship changed; and cross-tenant relationship leakage.

**Distributed-systems concerns.** Consistency between the relationship store and the authorization decisions made against it — a relationship change should become effective within a bounded time, mirroring the same consistency-model discipline Phase 5 applies to session revocation; and traversal cost at scale, since recursive relationship queries can be expensive without bounded depth and caching.

**Decision gates.** Repository ownership is the central decision gate of this phase and is explicitly not resolved here. The evidence needed to resolve it includes: the expected relationship-graph size and traversal complexity in realistic BASIS deployments; whether `basis-core`'s deterministic, stateless evaluation model can accommodate relationship traversal without compromising the kernel boundary rules in [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md); and a comparison of the operational cost of a new repository against the operational cost of an extension provider. This roadmap does not assume `basis-core` should become an OpenFGA-equivalent kernel, and it does not assume a new repository is warranted — both remain open until this evidence is gathered.

**Completion criteria.** A conceptual relationship model document reviewed and accepted as an ADR; a resolved repository-ownership decision, informed by the evidence above; and a documented cycle-handling and maximum-depth policy, before any implementation work begins.

**Operator-mode representation.** Not applicable to this discovery phase directly; operator-mode requirements are inherited by Phase 8, which implements the queries against this model.

**Training-mode explanation.** Eventually, the relationship path that contributed to a decision — which tuples were traversed, in what order, to reach an allow or deny — once Phase 8 implements queryable traversal.

**Evidence and audit requirements.** The discovery process itself should be evidenced: what alternatives were evaluated, what data informed the repository decision, and why the chosen architecture was selected, in the same form ADR alternatives-considered sections already require elsewhere in this repository.

**Schema and documentation impact.** No `basis-schemas` contract is proposed by this phase. Any future relationship-tuple contract is deferred until the repository decision is made and a reference implementation exists.

**Explicitly deferred work.** Final repository choice; storage technology (SQL versus graph-oriented); consistency model implementation; and the query APIs, which Phase 8 addresses architecturally, still without implementation.

**Engineering experience developed.** Relationship-based authorization modeling and the discipline of resolving a consequential architecture question with evidence rather than preference.

### Phase 8 — Fine-Grained Authorization Query APIs

**Purpose.** Define the future capability outcomes for authorization queries analogous to Check, BatchCheck, ListObjects, ListSubjects, and Explain operations, built against the relationship model Phase 7 defines, without locking in final route names or a final transport. Each of these operations resolves a relationship sub-decision — it does not independently produce a final BASIS authorization decision.

**Primary repositories.** Dependent on Phase 7's repository decision; `basis-gateway` for request composition; `basis-console` for query-explanation display.

**Prerequisites.** Phase 7's accepted relationship model and resolved repository-ownership decision; the Relationship Data Boundary's relationship-sub-decision/final-decision distinction above.

**Architectural outcome.** A caller can ask whether a subject stands in a given relationship to a resource — a Check-equivalent query returning a relationship predicate result, such as whether a subject `is_operator_for` a facility — ask the same question in bulk, list the objects a subject can reach or the subjects that can reach an object, and ask for an explanation of why a relationship query resolved the way it did, with deterministic results, bounded traversal, and tenant isolation, regardless of which underlying architecture Phase 7 selects. Every one of these operations returns a bounded relationship sub-decision. None of them answers the complete BASIS operation-aware authorization question on its own; composing a relationship sub-decision into a final decision is Phase 9's concern, and `basis-core` remains the authority that reaches it.

**Key capabilities.** Deterministic query results for a given relationship-model revision and consistency token; bounded traversal depth and cost; pagination for list-style queries; consistency tokens that let a caller request a query be evaluated against a specific, known relationship-model snapshot rather than an unspecified "current" state; contextual tuples supplied at query time rather than only from the stored graph; partial-failure handling for batch queries; caching of query results where safe; authorization-model revisioning, so a query can be evaluated against a specific model version; decision explanations; query authorization (who may ask a fine-grained authorization query is itself subject to authorization); and tenant isolation across every query type.

**Security and abuse cases.** Unbounded traversal used as a denial-of-service vector; query authorization bypass, where an unauthorized caller extracts relationship information through the query API itself; and explanation responses that leak more relationship structure than the querying subject is entitled to see.

**Distributed-systems concerns.** Consistency tokens as the mechanism for giving callers a predictable answer to "was this evaluated against fresh or slightly stale relationship data," rather than leaving that question unstated; and caching query results without violating the same staleness and revocation-consistency requirements established in Phases 2 and 5.

**Decision gates.** Final route or RPC naming; whether queries are synchronous only or also support a batched/asynchronous mode for very large ListObjects-style requests; and how consistency tokens are represented.

**Completion criteria.** A demonstrated deterministic-result property for a fixed relationship-model snapshot; a demonstrated bounded-traversal property under an intentionally cyclic or deep test graph; and a working explanation output that a training-mode session can render without exposing unauthorized relationship structure.

**Operator-mode representation.** The decision and its explanation for a specific subject-resource query.

**Training-mode explanation.** How a subject-resource relationship was resolved, and why alternative relationship paths did not grant access.

**Evidence and audit requirements.** Query evidence sufficient to reconstruct which relationship-model revision and consistency token a decision was evaluated against.

**Schema and documentation impact.** A future fine-grained-authorization-query contract, deferred until Phase 7's repository decision and a reference implementation exist.

**Explicitly deferred work.** Final API transport; final route names; whether this becomes a `basis-core` extension point invocation or a call to a separate relationship-authorization service.

**Engineering experience developed.** Fine-grained authorization query design, deterministic explanation generation, and bounded-traversal engineering.

### Phase 9 — Gateway and Core Integration

**Purpose.** Define how identity, relationship, and policy information may participate in the runtime decision path — as bounded evaluation inputs to `basis-core`'s final decision, never as an independent authorization outcome — without breaking the responsibilities `basis-gateway` and `basis-core` already carry.

**Primary repositories.** `basis-gateway` (request composition, relationship-evaluation invocation); `basis-core` (extension-point consumption of a relationship sub-decision).

**Prerequisites.** Phases 1 through 8; a stable operation-aware evaluation extension point in `basis-core`; the Relationship Data Boundary's relationship-sub-decision/final-decision distinction above.

**Architectural outcome.** The gateway composes identity context, relevant attributes, and relationship evaluation evidence into the request it sends to `basis-core`. The relationship authorization capability owns relationship-model evaluation and returns a bounded relationship sub-decision; `basis-core` remains the sole authority for the final operation-aware authorization decision and composes that sub-decision with identity, policy, and operation-aware context to reach it:

```text
Relationship authorization capability
    owns relationship-model evaluation
    returns a bounded relationship sub-decision

basis-core
    owns the final operation-aware authorization decision
```

A relationship sub-decision may answer a bounded predicate, such as whether `subject is_operator_for building-7`. It must not independently answer the complete BASIS operation question, such as whether a subject may write a chiller setpoint under the current operation, context, policy, and safety state — that composition belongs to `basis-core` alone. A relationship sub-decision `basis-core` relies on must be bounded, attributable, versioned, and safely explainable. Latency introduced by relationship evaluation is bounded and budgeted, and a failure to obtain a required relationship sub-decision fails closed by default, consistent with the kernel's existing default-deny posture.

**Key capabilities.** Gateway request composition extended to include relationship evaluation evidence alongside existing identity and operation-aware context; a defined `basis-core` extension point for consuming a relationship sub-decision during evaluation without the kernel acquiring graph persistence, network-backed relationship traversal, or a dependency on a specific relationship-authorization technology such as OpenFGA or SpiceDB; fail-closed behavior when a required relationship sub-decision cannot be obtained; correlation identifiers connecting a final decision back to the identity, relationship-model version, and policy versions that produced it; and an authorization-latency budget that accounts for the added relationship-evaluation step.

**Security and abuse cases.** Relationship sub-decision spoofing, where a compromised or misbehaving relationship authorization capability supplies a false predicate result that `basis-core` then relies on; and latency-budget exhaustion used as a denial-of-service vector against the evaluation path. Because a relationship sub-decision is bounded and attributable rather than a final grant, its blast radius is limited to the specific predicate it answers — but `basis-core` must still treat it as an untrusted evaluation input subject to the same evidentiary scrutiny as any other context, not as a pre-authorized decision.

**Distributed-systems concerns.** Latency budget allocation across identity validation, relationship evaluation, and policy evaluation; and the fail-closed behavior's interaction with the availability of the (still architecturally undetermined) relationship-data source.

**Decision gates.** Phase 9 must evaluate at least the following integration alternatives before any is selected; this roadmap does not select between them.

Gateway-resolved relationship evaluation:

```text
gateway
    → relationship capability
    → versioned relationship result
    → basis-core final evaluation
```

Precomputed relationship context:

```text
trusted relationship context
    → gateway validation and composition
    → basis-core final evaluation
```

The following integration shape is explicitly rejected unless a later ADR changes the kernel isolation rules in [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md):

```text
basis-core
    → network/database call
    → relationship service
```

`basis-core` performing its own network or database call to a relationship service would violate the kernel's prohibition on network I/O during evaluation and on runtime persistence clients; it is out of scope for this roadmap under the current kernel boundary rules. Beyond that rejection, the specific shape of the `basis-core` extension point for consuming a relationship sub-decision, and whether that sub-decision is fetched synchronously in the request path (the gateway-resolved alternative) or supplied as an already-validated, precomputed context (the precomputed alternative), remain open and are left to the ADR that resolves this phase.

**Completion criteria.** A demonstrated end-to-end path from gateway request composition through kernel evaluation using a relationship sub-decision as bounded evaluation input, without the kernel acquiring a dependency on the relationship store, graph persistence, or network-backed traversal; a resolved choice among the integration alternatives above, recorded in an ADR; and a measured latency budget.

**Operator-mode representation.** The complete, safely redacted request and evidence path for a given decision, including any relationship sub-decision it relied on.

**Training-mode explanation.** The same complete, safely redacted request and evidence path, walked step by step for a training audience, showing where a relationship sub-decision entered the path and how `basis-core` composed it into the final decision.

**Evidence and audit requirements.** Correlation IDs and policy/model version stamps sufficient to reconstruct exactly which identity, relationship-model revision, relationship sub-decision, and policy state produced a given final decision.

**Schema and documentation impact.** Extensions to the `operation-aware-decision-request`/`operation-aware-decision-response` contract family are a candidate future change, deferred until this phase's reference implementation exists.

**Explicitly deferred work.** Final extension-point interface; exact latency budget figures, which are deployment and implementation concerns.

**Engineering experience developed.** Runtime integration design under strict boundary constraints, and latency-budget engineering across a multi-hop decision path.

### Phase 10 — External Authorization Technology Evaluation

**Purpose.** Define an evidence-based comparison phase for selected external authorization technologies, so that any eventual adoption is grounded in real experience against BASIS's own requirements rather than in the recognizability of a tool's name.

**Primary repositories.** `basis-architecture` (comparison methodology and findings).

**Prerequisites.** Phases 7 through 9, since a meaningful comparison requires BASIS's own relationship model and integration boundary to compare against.

**Architectural outcome.** A small, defensible comparison set — not every listed technology — is selected and evaluated against BASIS's actual requirements: semantic fit, typed policy support, relationship modeling, auditability, explanation quality, latency, caching, policy rollout, consistency, tenant isolation, failure behavior, deployment complexity, air-gapped suitability, and operational burden. Candidate technologies named for consideration include OpenFGA or SpiceDB for relationship-based authorization, Cedar for typed attribute-based access control, and OPA/Rego for general policy definition and distribution. No technology in this list is required to be implemented, and none is assumed to be adopted.

**Key capabilities.** A comparison methodology applied consistently across the selected technologies; representative BASIS scenarios (an operation-aware decision request, a relationship traversal, a policy rollout) run against each candidate; and a findings document stating semantic fit, latency, and operational-burden results with enough specificity to be falsifiable, consistent with the writing standard already established for this repository.

**Security and abuse cases.** Adopting an external engine's trust and consistency model without verifying it meets BASIS's own security invariants (tenant isolation, fail-closed defaults, deny precedence) is the primary risk this phase is designed to catch before it becomes a production dependency.

**Distributed-systems concerns.** Each candidate's own consistency model, caching behavior, and failure behavior must be evaluated on the same terms this roadmap applies to BASIS's own components — not assumed adequate because the technology is established elsewhere.

**Decision gates.** Which small comparison set to evaluate (this roadmap does not select it); and the specific integration value bar a technology must clear to be adopted, versus BASIS building the equivalent capability itself.

**Completion criteria.** A findings document comparing the selected technologies against the criteria above, reviewed and accepted, before any adoption decision is made.

**Operator-mode representation.** The engine and policy revision in effect, once and if any external engine reaches a runtime deployment.

**Training-mode explanation.** How the same request is represented and evaluated differently by different candidate engines, for users comparing them.

**Evidence and audit requirements.** The comparison methodology and raw findings are retained as evidence supporting whatever adoption or non-adoption decision follows.

**Schema and documentation impact.** None at the contract level; this phase produces an evaluative document, not a contract.

**Explicitly deferred work.** The adoption decision itself; any integration work with a selected technology.

**Engineering experience developed.** Rigorous, criteria-based technology evaluation — real experience with each candidate rather than documentation-only comparison.

### Phase 11 — Policy and Configuration Distribution

**Purpose.** Define future capabilities for distributing signed policy artifacts, signed authorization models, and versioned identity-provider configuration to the places that must enforce them, addressing the policy-distribution research direction already named as unresolved in `ROADMAP.md` Phase 4.

**Primary repositories.** `basis-gateway` (distribution consumption, staged rollout); `basis-identity` (configuration distribution); `basis-architecture` (control-plane/data-plane separation model).

**Prerequisites.** Phases 1 through 9, since this phase distributes the tenant configuration, relationship model, and policy artifacts those phases define.

**Architectural outcome.** Policy artifacts, authorization models, and identity-provider configuration are distributed as signed, versioned artifacts with a defined staged-rollout, validation, rollback, and status-reporting model, with control-plane concerns (deciding what the desired state is) kept separate from data-plane concerns (enforcing the currently active state) without locking in a specific deployment technology.

**Key capabilities.** Signed policy and authorization-model artifacts; versioned identity-provider configuration distribution; staged rollout; validation before activation; rollback; status reporting back to the control plane; distribution-health monitoring; stale-policy behavior (what an enforcement point does when it cannot reach the distribution source); last-known-good behavior; provenance for every distributed artifact; air-gapped distribution paths, since OT deployments frequently cannot reach a networked distribution source continuously; and trust anchors for signature verification.

**Security and abuse cases.** Policy-rollback attacks, where an attacker forces an enforcement point back to a previous, weaker policy version; unsigned or tampered configuration being accepted; and stale-policy exploitation, where an attacker times an action to a window when an enforcement point is known to be running outdated policy.

**Distributed-systems concerns.** Control-plane/data-plane separation as the organizing principle; rollout staging across a fleet of enforcement points; and the same fail-open/fail-closed and last-known-good tension named in Phases 2 and 5, now applied to policy rather than identity data.

**Decision gates.** Deployment topology for the distribution mechanism; whether distribution is push-based, pull-based, or hybrid; and hosted versus air-gapped defaults.

**Completion criteria.** A demonstrated signed-artifact rollout and rollback cycle; a documented stale-policy and last-known-good behavior; and a tested rejection of an unsigned or tampered artifact.

**Operator-mode representation.** The active policy revision and rollout state.

**Training-mode explanation.** Active version, desired version, signature validation, rollout state, rollback, stale configuration, and why a node accepted or rejected a given update.

**Evidence and audit requirements.** Every distribution event — signed, validated, rolled out, rolled back, or rejected — is evidenced with the artifact's provenance and version.

**Schema and documentation impact.** A future signed-policy-distribution contract, deferred until reference implementation exists.

**Explicitly deferred work.** Specific deployment technology for distribution; specific signing scheme; exact staged-rollout percentages or timing.

**Engineering experience developed.** Signed configuration distribution, control-plane/data-plane separation, and rollout/rollback engineering under adversarial assumptions.

### Phase 12 — Performance, Failure, and Isolation Validation

**Purpose.** Define a final hardening phase that establishes measurable service-level objectives and bounded guarantees for the capabilities built in Phases 1 through 11, before any production-readiness claim is made about them.

**Primary repositories.** All repositories touched by Phases 1 through 11; `basis-console` for failure-state visibility.

**Prerequisites.** Phases 1 through 11, since this phase validates them rather than building new capability.

**Architectural outcome.** Every capability introduced by this roadmap carries a stated, tested bound — a latency figure, a revocation-propagation delay, a cache-staleness window, a synchronization-reconciliation interval — verified under load, under dependency outage, under network partition, and under adversarial cross-tenant attack, rather than asserted without evidence.

**Key capabilities.** Load testing and latency measurement under realistic concurrency; cache-stampede testing; dependency-outage simulation; network-partition testing; stale-configuration testing; key rotation performed while traffic is live; revocation-delay measurement; synchronization retry and duplicate-event testing; relationship-cycle and deep-traversal testing; cross-tenant attack simulation; noisy-neighbor testing across tenants sharing infrastructure; data-corruption and recovery testing; rollback testing; and chaos testing more broadly.

**Security and abuse cases.** This entire phase is a security and abuse-case validation exercise. It should specifically re-test the cross-phase security concerns enumerated below under load and failure conditions, since a mitigation that holds under normal operation does not necessarily hold under a simultaneous failure and attack.

**Distributed-systems concerns.** This phase is where every consistency model, staleness bound, and partition-behavior decision made in Phases 1 through 11 is empirically verified rather than assumed.

**Decision gates.** What specific service-level objectives are targeted for each bounded guarantee; and what constitutes an acceptable degraded-mode behavior versus an unacceptable one, per capability.

**Completion criteria.** Measurable service-level objectives or explicitly bounded guarantees, backed by test evidence, for every phase this roadmap introduces, before any of them is described as production-ready.

**Operator-mode representation.** Degraded-state status: what is currently failing or degraded.

**Training-mode explanation.** Safe failure demonstrations showing what degraded operation means for a given capability and which guarantees remain intact during that degradation.

**Evidence and audit requirements.** Test evidence for every claimed bound — latency, propagation delay, staleness window — retained as the basis for any production-readiness claim.

**Schema and documentation impact.** None directly; this phase validates prior phases' contracts rather than introducing new ones.

**Explicitly deferred work.** None — this is the validation phase itself, and its scope is exactly the capabilities the prior eleven phases define.

**Engineering experience developed.** Performance engineering, failure-mode design, and adversarial and isolation testing as disciplines applied to the full breadth of what this roadmap builds.

---

## Cross-Phase Security Concerns

Some threats span more than one phase and are easy to under-address if each phase is reviewed only in isolation. This roadmap does not attempt to fully resolve any of them here; it identifies where each must be addressed and notes that later threat-model updates to [`docs/security/threat-model.md`](../security/threat-model.md) will be required as each phase's architecture solidifies.

Cross-tenant credential acceptance, issuer confusion, and audience confusion are primarily Phase 1's concern, but recur anywhere a credential crosses a trust boundary — including token exchange (Phase 3) and workload identity (Phase 4). Key substitution and stale signing keys are primarily Phase 2's concern. Token replay is relevant to Phases 3, 4, and 5. Delegation escalation and confused-deputy behavior are Phase 3's central concern but recur wherever workload identity (Phase 4) or gateway integration (Phase 9) composes a request on a subject's behalf. Session fixation and refresh-token reuse are Phase 5's concern. Synchronization poisoning and identity duplication are Phase 6's concern. Relationship injection and authorization-graph cycles are Phase 7's concern, tested empirically in Phase 12. Stale relationship data is a Phase 7/9 boundary concern. Policy-rollback attacks and unsigned configuration are Phase 11's concern. Cache poisoning is Phase 2's concern, tested again in Phase 12. Denial of service and unbounded traversal recur across Phases 2, 7, 8, and 12. Audit tampering is a concern for every phase that produces evidence, and is addressed by the same evidentiary discipline `basis-core` and `basis-gateway` already apply to audit records — the console's inability to redefine audit semantics, established in [`docs/architecture/basis-console.md`](../architecture/basis-console.md), extends to every new evidence type this roadmap introduces. Sensitive-data leakage through diagnostics or training mode recurs across every phase with a training-mode requirement, and is the specific reason each phase's Evidence and audit requirements subsection above names what must be redacted.

---

## Console Education Matrix

| Capability | Operator mode | Training mode |
| --- | --- | --- |
| Tenant trust | Provider, tenant, and health | Issuer boundary, configuration version, rejection of cross-tenant credentials |
| Identity caches | Health and freshness | Cache key isolation, hit/miss, refresh, stale-state behavior |
| Token exchange | Delegated identity status | Subject token, actor token, attenuation, resulting credential |
| Workload identity | Workload, owner, trust state | Attestation, trust domain, rotation, delegation |
| Session revocation | Active/revoked status | Propagation, invalidation, consistency, partition behavior |
| SCIM | Synchronization health | Idempotency, retry, conflict, tombstone, reconciliation |
| ReBAC/FGA | Decision and explanation | Tuples, traversal, inheritance, cycles, model version |
| External policy engine | Engine and revision | Request transformation and semantic comparison |
| Policy distribution | Active revision and rollout | Signing, validation, rollout, rollback, stale-policy behavior |
| Failure testing | Degraded-state status | Which dependency failed and which guarantees remain |

Detailed explanations of each row belong in the corresponding phase section above; this table exists to make the operator/training split scannable at a glance, not to substitute for those sections.

---

## Experience-Development Goals

Implementation of this roadmap should develop and demonstrate substantive engineering experience — building identity infrastructure rather than only integrating vendor products, multi-tenant service architecture, trust-boundary design, cache architecture and invalidation, secure token design, delegated authority, human and non-human identity modeling, distributed session lifecycle, consistency and revocation semantics, reliable synchronization with idempotency and retries and reconciliation, relationship-based authorization, fine-grained authorization queries, external policy-engine integration, signed configuration distribution, failure-mode design, performance engineering, and isolation and adversarial testing.

The purpose of naming these areas explicitly is not to accumulate résumé keywords. Each capability in this roadmap is included because it is architecturally justified by a real requirement named in the Purpose section above, and each is expected to be implemented rigorously, tested under failure — not merely under expected conditions — and made observable through evidence, consistent with the evidence requirements stated in every phase and with the training-mode constitutional requirement stated earlier in this document.

---

## Sequencing and Dependency Guidance

The expected high-level sequence is:

```text
Complete current basis-core operation-aware roadmap
    ↓
Phases 1–2: tenant trust and cache isolation
    ↓
Phases 3–4: delegation and workload identity
    ↓
Phases 5–6: lifecycle, revocation, and synchronization
    ↓
Phases 7–8: relationship architecture and FGA queries
    ↓
Phase 9: runtime integration
    ↓
Phase 10: external technology comparison
    ↓
Phase 11: distribution control plane
    ↓
Phase 12: performance and failure hardening
```

This sequence is provisional. It reflects the dependency structure named in each phase's Prerequisites subsection above, not a committed schedule or an exact pull-request count. Architecture discoveries made during any phase may justify splitting a phase into more than one, combining two phases, or reordering later phases — Phase 10's external technology comparison, for instance, could reasonably be pulled earlier if a specific integration question in Phase 7 or 8 needs it sooner. Any such change to phase sequencing that affects a stable component boundary or an established contract should go through the ADR process described in [`GOVERNANCE.md`](../../GOVERNANCE.md), the same discipline `ROADMAP.md` already applies to changes in architectural intent.

---

## Deferred Decisions

The following decisions are intentionally left open until implementation planning begins for the relevant phase. Each is listed with the evidence needed to resolve it.

**Final relationship-service repository ownership (Phase 7).** Resolved by the evidence named in Phase 7's decision-gate subsection: expected graph size and traversal complexity, kernel-boundary compatibility analysis, and an operational-cost comparison between a new repository and an extension provider.

**Storage technology for relationship data, and SQL versus graph-oriented persistence (Phase 7).** Resolved once the repository-ownership decision above is made and a reference implementation can be benchmarked against realistic traversal patterns.

**Cache technology and shared-versus-local caching (Phase 2).** Resolved by measuring cross-instance cache-consistency requirements against the tenant-isolation and staleness bounds Phase 1 and Phase 2 establish.

**Consistency model for distributed session revocation (Phase 5).** Resolved by the partition-behavior testing Phase 12 performs, informed by the fail-open/fail-closed governance question `ROADMAP.md` Phase 4 already tracks as unresolved.

**Final API transport for fine-grained authorization queries (Phase 8).** Resolved once Phase 7's repository decision is made, since the transport choice depends in part on where the relationship data and query logic live.

**OpenFGA versus SpiceDB, and Cedar versus OPA/Rego comparison scope (Phase 10).** Resolved by Phase 10's own comparison methodology; this roadmap deliberately does not pre-select the comparison set.

**Event bus or change-feed technology for cache invalidation and revocation propagation (Phases 2 and 5).** Resolved by benchmarking candidate transports against the staleness and propagation bounds those phases establish.

**Deployment topology and hosted-versus-air-gapped defaults for policy distribution (Phase 11).** Resolved in coordination with `basis-deploy`'s eventual packaging and orchestration design, once that repository exists.

**Exact schema migration timing for any contract this roadmap anticipates.** Resolved individually, phase by phase, following the same readiness discipline ADR-0005 established: a contract is proposed to `basis-schemas` only after its architecture and a reference implementation exist, never speculatively ahead of them.

**Exact service boundaries beyond what the architectural invariants above already state.** Resolved as each phase's implementation planning begins, within the boundaries this roadmap and the existing ecosystem documents already fix.

**Production availability targets.** Resolved by Phase 12's measured service-level objectives, not asserted in advance of that testing.

---

## Summary

This roadmap preserves the architectural direction for a substantial expansion of the BASIS ecosystem — multi-tenant identity, distributed session lifecycle, relationship-based authorization, fine-grained authorization queries, external policy-engine evaluation, and signed policy distribution — while the current `basis-core` operation-aware roadmap continues as the ecosystem's present priority. It assigns each capability to the architectural boundary that already governs the component most suited to own it, extending those boundaries where a genuinely new capability requires it and refusing to relax them where a design pressure suggests otherwise. It makes training-mode education a requirement for every phase from the outset, not an afterthought. It identifies the security and distributed-systems concerns each phase must address and names where later threat-model work is required. It defines completion outcomes in testable terms without prescribing implementation technology, final APIs, or an exact schedule. And it remains explicitly adaptable: the phase structure, sequencing, and open decision gates recorded here are architectural intent as understood today, subject to revision through the same ADR process that governs every other consequential decision in this repository.

---

## Related Documents

- [`ROADMAP.md`](../../ROADMAP.md) — the ecosystem's current phase structure and status; this document elaborates selected work represented across Phases 4 and 5 of that roadmap into a named capability program with a more detailed dependency sequence, gated on completion of the current `basis-core` operation-aware roadmap, not on completion of every unrelated Phase 4 or Phase 5 item
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction this roadmap's invariants extend
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules Phases 7 and 9 must not violate
- [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md) and [`docs/architecture/identity-authority-modes.md`](../architecture/identity-authority-modes.md) — the identity engine architecture and authority modes Phases 1 through 6 extend
- [`docs/architecture/basis-console.md`](../architecture/basis-console.md) — the console architecture and operator/training-mode invariants this roadmap's console requirements extend
- [`docs/security/threat-model.md`](../security/threat-model.md) — the existing threat model that later phase-specific work must update
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR process this roadmap's decision gates and sequencing changes must follow
- [`docs/adr/README.md`](../adr/README.md) — when a phase's architecture work requires a new ADR
- [`docs/glossary.md`](../glossary.md) — the canonical terminology reference this roadmap's working vocabulary should graduate into as each phase stabilizes
