# ADR-0022: Workload Credential Lifecycle and Scope Boundary

## Status

Proposed

Consistent with this repository's established practice (see ADR-0018's Context and ADR-0021's Status), this ADR is numbered when first proposed and submitted as `Proposed`. Merging it does not accept it. Acceptance requires a separate, dedicated formal-acceptance review and PR. Publication of this proposal authorizes no implementation, selects no credential technology, does not close cross-ecosystem gap ARCH-GAP-011, and makes no upstream platform's contract canonical.

## Context

Accepted architecture authenticates two kinds of workload on the governed path to an OT target, and it keeps both separate from every other identity on that path.

- **Producer workload identity.** [ADR-0008](0008-producer-workload-authentication-and-admission.md) (Accepted) establishes the operation-producer workload's identity toward `basis-gateway` through mutual TLS. The identity is the single eligible URI SAN of a validated client certificate, taken verbatim. Admission is a separate, exact-match, deployment-controlled decision. [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) (Accepted) places certificate validation at a trusted ingress and keeps identity derivation and admission in `basis-gateway`. [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) (Accepted) makes producer workload credential custody a permanent responsibility of the operation-producer runtime.
- **Upstream workload identity.** [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md) (Accepted) requires the producer-intake boundary to establish an authenticated upstream workload identity before it admits any intake request. It selects no mechanism. [ADR-0021](0021-upstream-context-assertion-trust-boundary.md) (Accepted) uses that identity as the *originating source* `U` of upstream-originated context, and grants category authority to `U` and to the relaying producer `P` independently.

Accepted architecture also fixes the separations this ADR must keep:

```text
authorization subject
  != upstream workload
  != operation-producer workload
  != protocol-executor role
  != OT device identity / device credential
```

ADR-0011 Decision 14 distinguishes the authorization-subject credential, the producer workload credential, a possible future executor workload credential, and the protocol/device credential. ADR-0018 Decision 8 adds upstream workload authentication material as a fifth, separate class. None may be exchanged for, converted into, or substituted for another.

**What accepted architecture leaves open.** Every accepted decision treats a workload identity as a stable thing that is authenticated and admitted. None says what keeps it stable, or what a credential for it is allowed to do over time.

- ADR-0008 records revocation paths (admission removal, certificate expiry, trust-anchor removal, optional local revocation lists) and a naming discipline ("ensuring certificate renewal reissues the same URI SAN"). It defers "full credential lifecycle automation" and "whether admission entries require an environment/operational scope field." It describes gateway HA only as a configuration-consistency concern.
- ADR-0018 defers "producer and upstream workload credential issuance, rotation, revocation, HA identity sharing, bootstrap, compromise handling, and disconnected operation" as cross-ecosystem Workstream 3E.
- ADR-0020 and ADR-0021 list upstream and producer credential lifecycle as open (Workstream 3E, ARCH-GAP-011). ADR-0021 Decision 13 states that it relies only on abstract authenticated identities `U` and `P`.
- `ROADMAP.md` Phase 4 tracks "Certificate and credential lifecycle management tooling" and "Credential revocation propagation" as research directions.

The gap has a concrete security effect. Admission, category trust (ADR-0021), and attribution are all expressed against a workload identity. If rotation, replacement, replication, failover, or disconnection can change which identity a workload presents, or keep a credential usable after it should not be, then each of those accepted decisions can be broadened or bypassed without anyone changing its configuration. Examples: a rotated credential that arrives with a different identity string; a replacement instance that inherits a predecessor's key from disk; a replica fleet sharing one private key, so that one compromised replica forces revocation of all of them; a disconnected enforcement point that keeps accepting a revoked credential indefinitely.

**Current implementation (read-only evidence; architecture governs).** At the revisions inspected — `basis-gateway` `1e08348`, `basis-producer` `8c7410a`:

- `basis-gateway` admits mTLS producers by exact, case-sensitive match of the certificate-derived URI SAN against the static `OPERATION_PRODUCER_MTLS_ADMITTED_URIS` configuration. The reference NGINX ingress template validates the client certificate chain and validity period against a configured producer CA bundle. It configures no revocation list. Gateway audit events record the derived producer workload identity (`operation_producer_workload_identity`). They record nothing that distinguishes one certificate for that identity from another.
- `basis-producer` loads one client certificate and private key from configured file paths when its gateway client is constructed. It has no enrollment, renewal, or rotation behavior.
- No producer-intake realization exists, so no upstream workload credential exists in any implementation.
- No BASIS component issues workload credentials, distributes revocation state, or records credential lifecycle events. `basis-identity` has no workload-identity pipeline (ADR-0008 Context).

**Motivating consumer.** Ipotio, a supervisory platform, tracks the cross-ecosystem form of this question in its own architecture repository as **ARCH-GAP-011**. It is referenced here by name, not by hyperlink, per this repository's cross-repository citation convention, and only as the motivating integration. The decision below is stated for any upstream system and any deployment.

**Terminology.** This ADR uses *BASIS*, not the proposed *Basitra* ([ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md), Proposed). *Upstream workload identity* and *producer-intake boundary* are ADR-0018's working vocabulary. *Logical workload identity*, *credential instance*, *workload role*, *intended scope*, *compromise domain*, *replica*, *credential-instance revocation*, *identity retirement*, *principal compromise*, and *residual authority window* are introduced below as working vocabulary under the same convention and are not promoted to the glossary.

### Sufficiency of existing architecture

Before deciding, this ADR checked whether any unresolved decision blocks a durable lifecycle decision. None does.

- The identity model the lifecycle must preserve is fixed: ADR-0008 for the producer, ADR-0018 for the upstream workload, ADR-0021 for how the upstream identity is used as a source.
- The enforcement points are fixed: `basis-gateway` (with the ADR-0009 ingress) for producers, and the producer-intake boundary for upstream workloads.
- The authentication *mechanism* for intake is not selected (ADR-0018). The lifecycle semantics below are stated against an abstract authenticated identity, the same way ADR-0021 is. They do not need that mechanism.
- Deployment topology, HA, and site decomposition are not decided (`ROADMAP.md` Phase 4). The decision below needs only the identity consequences of replication and disconnection, not a topology.

The remaining open questions are mechanisms. They are recorded under **Deferred Decisions** and do not change the semantics decided here.

## Problem

What lifecycle and scope properties must an upstream workload identity and an operation-producer workload identity satisfy so that admission, replacement, rotation, revocation, compromise recovery, replication, and disconnected operation do not silently broaden authority, or preserve it beyond the intended workload, while:

- keeping every identity separation accepted architecture already requires;
- selecting no credential technology, PKI, secret store, or enrollment protocol;
- not claiming revocation guarantees a distributed or disconnected deployment cannot deliver;
- working for any upstream system and any deployment topology?

## Decision

**A workload's security identity is a governed, stable, role-specific logical workload identity. It is never a credential, a runtime instance, a host, a name, or a network position. Credentials are bounded, individually attributable authenticators, each bound to exactly one logical identity and usable only at the boundary that identity's role belongs to. Admission and every authority grant attach to the logical identity. No lifecycle event, including issuance, rotation, replication, replacement, failover, or disconnection, can create, transfer, widen, or extend that authority. Only an explicit, attributable governance action can do so. Revocation of a credential and retirement of an identity are distinct operations. Each takes effect at an enforcement boundary no later than a bounded, governed time, and not necessarily instantly. Where an enforcement boundary cannot establish that its trust state is within its bounds, it fails closed.**

### 1. Logical workload identity is not a credential instance

```text
logical workload identity  1 ── *  credential instance
credential instance        * ── 1  logical workload identity
```

- **Logical workload identity.** The stable security principal a workload is authenticated as. Admission, ADR-0021 category grants, attribution, and evidence attach to it. For the producer under the ADR-0008 profile, it is exactly the URI SAN identity string the gateway derives and matches. For the upstream workload, it is the admission identity the producer-intake boundary establishes under whatever mechanism a later decision selects.
- **Credential instance.** One concrete authenticator that proves control of exactly one logical workload identity. Under the ADR-0008 profile, one issued certificate and its private key is one credential instance.

The following rules apply:

- **One identity per credential.** A credential instance proves exactly one logical workload identity. A credential that could prove more than one is not conforming. ADR-0008's exactly-one-eligible-URI-SAN rule already enforces this for the producer profile. Any other profile must enforce it too.
- **Many credentials per identity.** A logical identity may have several credential instances over time (rotation) and at the same time (replicas, Decision 4; bounded rotation overlap, Decision 8).
- **Rotating a credential never changes identity.** A new credential instance for an existing logical identity presents exactly the same identity value. A credential that presents a different identity value is, by definition, a credential for a different logical identity. It is never treated as a renewal, however similar the value looks. ADR-0008's exact-match admission rejects it until that different identity is itself explicitly admitted.
- **Identity is assigned, never derived from runtime facts.** A logical identity comes into existence only by a governed action (Decision 6). It is never derived from hostname, IP address, process name, container or VM identifier, scheduler slot, filesystem location, software image, or a self-asserted name. Restarting, rescheduling, upgrading, moving, or reimaging a workload therefore neither creates a new logical identity nor, by itself, carries an existing one to a new instance (Decision 11).

### 2. Upstream and producer identities stay independent through every lifecycle event

Each logical workload identity has exactly one **workload role**: *upstream workload* (authenticated at the producer-intake boundary, ADR-0018) or *operation producer* (authenticated at `basis-gateway`, ADR-0008). The role is fixed when the identity is established and never changes. A workload that needs a different role gets a different logical identity.

- **Disjoint admission.** A logical identity admitted at the producer-intake boundary must not be admitted at `basis-gateway` as an operation producer, and the reverse also holds. Being admitted at one boundary never counts as being admitted at the other (ADR-0018 Decision 2).
- **No dual-use credential.** One credential instance never serves as both an upstream-intake credential and a producer-gateway credential. This follows from Decision 1 (one identity per credential) together with disjoint admission. It holds even when both workloads run on one host or in one process, are operated by one team, or obtain credentials from one trust root.
- **No borrowing across the boundary.** An upstream workload never presents, holds, or is delegated the producer's credential. A producer never authenticates an upstream workload on the strength of its own admission. ADR-0018 Decision 3 already prohibits exposing or delegating the producer's gateway-facing credential across the intake boundary. This ADR extends that prohibition to every lifecycle event. Rotation, failover, recovery, or colocation never creates an exception.
- **Source attribution is stable.** For ADR-0021, the originating source `U` of an upstream-originated value is the upstream *logical* identity that authenticated at intake. It is not a credential instance, replica, or host. Rotation, replica failover, and identity-preserving replacement of `U` therefore never change which source a value is attributed to. They also never change which ADR-0021 category grants apply. The authentication identity that intake admission uses and the originating source identity that ADR-0021 uses are one identity for a given logical principal. No second, synthetic identity is created for context assertions.
- **Relay and origin stay separate.** The relaying producer `P` and the originating source `U` stay distinct principals through every lifecycle event. No lifecycle event can make `P` the origin of a value `U` supplied, or make `U` a producer.

### 3. A credential proves one identity and grants nothing by itself

A credential instance is scoped to its logical workload identity. It is not a general deployment credential.

- **Possession proves identity only.** Possession of a valid credential instance establishes that the presenter controls that logical identity. Every authority the workload has comes from governed configuration that names that identity: intake admission (ADR-0018), producer admission (ADR-0008), and category grants (ADR-0021).
- **Credential possession never implies:**
  - authorization-subject authority (ADR-0008 "Producer vs. authorization subject"; ADR-0018 Decision 3);
  - any other producer's or any other upstream source's identity;
  - protocol-executor authority beyond what the workload's role and the ADR-0012 binding permit;
  - device or protocol authentication authority (ADR-0011 Decision 14);
  - any authority at a boundary other than the one its role belongs to.
- **No shared credentials across principals.** A single credential that several distinct logical workloads use (for example, one credential used by every workload at a site) is not conforming. It erases attribution and makes every one of those workloads revocable only together.
- **Distinguishable from other credential classes.** A workload credential must be distinguishable from, and never acceptable as, an authorization-subject credential or a device/protocol credential, and the reverse also holds. This ADR does not decide how that is achieved.

### 4. Replicas share a logical identity and never share a credential instance

A deployment may run several instances of one logical workload at the same time, for availability or capacity. The default security semantics are:

> **Replicas of one logical workload share that logical workload identity. Each replica holds its own, independently issued credential instance. No two instances share the secret authentication material of one credential instance.**

A **replica** is an instance that implements the same logical workload with the same role, the same intended scope (Decision 5), the same admission, and the same authority grants. Instances that differ in any of those are not replicas of one workload. They are distinct logical workloads and need distinct logical identities.

Three options were compared.

| Property | **A. Shared logical identity, distinct credential instances** (selected) | B. Each replica a separate logical identity | C. Either, declared per admission |
| - | - | - | - |
| Attribution | Logical identity plus credential instance | Per-replica identity | Depends on declaration |
| Blast radius of one replica's credential | That credential instance only; it can be revoked alone (Decision 9) | That replica's identity only | Depends on declaration |
| Authority exposed by one compromised replica | The logical identity's authority, which replicas by definition share | The same authority, duplicated across N identities | Same |
| Replaceability and HA | Admission and grants stated once; failover preserves source attribution (Decision 2) | Every grant repeated per replica; upstream failover changes the ADR-0021 originating source | Adds a mode that every enforcement point must interpret |
| Least privilege | Equal to B for true replicas | Equal to A | Equal |
| Auditability | Requires credential instances to be distinguishable at the enforcement boundary and in evidence (Decision 12) | Native | Requires both |

**Why A.** Replicas have the same authority by definition, so B does not reduce what one compromise exposes. It only multiplies the grants a deployment must keep consistent. Under ADR-0021 it also changes the originating source identity on every upstream failover, which makes source grants and attribution unstable for what is operationally one source. A keeps admission and grants expressed once. It keeps attribution stable across failover. Because credential instances are distinct, it also keeps per-replica attribution and per-replica revocation.

**Why not C as a declared mode.** Nothing in this architecture treats two distinct logical identities as equivalent. Exact-match admission (ADR-0008) and identity-scoped grants (ADR-0021) already forbid accidental equivalence, so a declaration flag would add configuration without adding a guarantee. A deployment that wants per-instance principals may still establish each instance as its own logical workload. Each such identity is then a separate principal with its own admission, its own grants, and, under ADR-0021, its own standing as a separate source.

**Why distinct credential instances are required.** Copying one credential instance's secret material to several replicas makes their authentications indistinguishable. Compromise of any one replica then forces revocation of the credential all of them depend on. That turns a single-replica credential incident into a whole-workload outage and removes per-replica attribution. It is not conforming.

This ADR does not decide load balancing, cluster membership, leader election, or how replicas coordinate work.

### 5. Identity scope is explicit and bounded to one compromise domain

The authority exposed by one compromised credential instance is, at most:

```text
the authority governed configuration grants its logical identity
  (admission, category grants, and any future identity-scoped grant)
for as long as that instance can still be admitted
  (its residual authority window, Decisions 9 and 13)
```

The architecture makes that exposure explicit and governable through three rules. It does not prescribe a topology.

- **Intended scope is recorded.** When a logical workload identity is established, its role and its *intended scope* are recorded with it as governed, attributable configuration. Intended scope means what the identity is meant to serve, such as which sites, execution boundaries, runtime instances, protocol families, or target classes. The record is the reviewable statement of that identity's blast radius. This settles, at the semantic level, the question ADR-0008 left open about whether admission needs an operational scope. It selects no field, format, or storage.
- **One identity per compromise domain.** A *compromise domain* is the set of sites, execution boundaries, and targets that a deployment has explicitly decided to accept losing together to one compromise. One logical workload identity must not span more than one compromise domain. A deployment decides how large its compromise domains are. It may decide that a site, a zone, a protocol family, a device class, or one execution boundary is its own domain. It may also decide, explicitly and on the record, that several sites form one domain. What it must not do is let one identity span independent domains by default, by convenience, or because instances happen to share infrastructure.
- **Scope never changes through a lifecycle event.** Rotation, replacement, replication, and failover never widen intended scope. Widening scope is a governed configuration change on its own, attributable to an administrative authority. It is never a side effect of issuing a credential.

Recording scope does not, by itself, enforce it. Whether `basis-gateway` should restrict a producer identity to the targets in its intended scope is a separate authorization question, recorded under **Deferred Decisions**. Until such enforcement exists, what bounds a compromised producer is: the need for an independently authenticated authorization subject (ADR-0008); `basis-core` evaluation of that subject; the producer's ADR-0021 category grants; and deployment controls. Where a producer is colocated with a protocol executor that holds device credentials, compromise of that runtime also yields device reach for its dispatch scope. ADR-0011 Decision 19 records that concentration as a residual risk no application architecture closes alone. Custody of those device credentials belongs to ADR-0011 Gate 4, not to this ADR.

### 6. Bootstrap: a credential is usable only after an explicit, governed binding

A new workload is not admitted merely because:

- it can reach the producer-intake boundary or `basis-gateway`;
- it presents a credential that chains to a generally trusted, or deployment-trusted, root;
- it is colocated with, or deployed by, an already trusted workload;
- it inherited a predecessor's filesystem, volume, image, secrets, or runtime environment;
- it claims the same name, hostname, or address as an admitted workload.

Before any credential instance is usable, all of the following must be established, each as an attributable governed act:

1. **The logical workload identity exists.** An authenticated administrative authority has established it, with its role and intended scope (Decisions 2, 5).
2. **The credential instance is bound to that identity.** It was issued for that identity and no other (Decision 7).
3. **The receiving instance was governed into receiving it.** The instance that holds the credential was enrolled through a governed act that vouches for it. That act may be operator action or verified attestation. It is never reachability, name, colocation, or inherited environment.
4. **The enforcement boundary admits the identity.** The boundary for the identity's role has explicit admission configuration naming it (ADR-0008 for producers; ADR-0018 Decision 2 for upstream workloads).

Any of these may happen before the others. A credential is not usable for admission until all four hold. This ADR does not choose the enrollment protocol, bootstrap-secret format, attestation technology, or who performs each step.

**Inherited credential material is not bootstrap.** An instance that holds another instance's credential material because it inherited that material is using a credential that was not governed into its hands. If a deployment moves credential material to a new runtime deliberately, for example to restore failed hardware, that move is issuance by other means. It must satisfy the same requirements as issuance: it is attributable, recorded, and bound to the same identity and scope.

### 7. Issuance is governed, attributable, bounded, and role-specific

Every credential instance must be:

- **issued for exactly one logical workload identity** (Decision 1), and for that identity's role only (Decision 2);
- **issued within the identity's intended scope.** Issuing a credential never creates or widens scope (Decision 5);
- **issued by an issuer the enforcement boundary for that role is configured to accept.** Accepting an issuer is governed configuration of that boundary. It is never admission. ADR-0008's "Trust anchor is not admission" applies to every workload role;
- **attributable.** It must be possible to establish which authority issued it, for which identity, and when;
- **validity-bounded.** Every credential instance has a validity bound that the enforcement boundary can evaluate using only locally held state, without contacting the issuer. A credential with no such bound is not conforming. This is what keeps disconnection from creating indefinite authority (Decision 13);
- **distinguishable.** It is never acceptable as an authorization-subject or device/protocol credential, and the reverse also holds (Decision 3).

This ADR does not select a CA hierarchy, certificate profile, issuer product, workload-identity system, secret manager, hardware key store, or validity duration. It does not require separate issuers per role. Where a deployment uses one trust root for several roles, the separation in Decision 2 must be enforced by admission and issuance, not by the trust root alone.

### 8. Rotation replaces a credential and changes nothing else

Rotation issues a new credential instance for an existing logical workload identity and then retires the old instance. Rotation must not:

- change the logical workload identity (Decision 1);
- widen admission, intended scope, or any grant (Decision 5);
- reset or re-attribute provenance. ADR-0021 sources and relays remain the same logical identities (Decision 2);
- lose attribution. Evidence stays attributable to the logical identity and, through the credential-instance distinguisher, to the specific instance (Decision 12);
- create indefinite dual validity.

**Overlap is bounded, explicit, and auditable.** A deployment may let old and new credential instances be valid at the same time so that rotation does not interrupt service. If it does, the overlap:

- is **bounded**, ending no later than the old instance's validity bound or its revocation;
- is **explicit**, meaning the old instance is expected to retire, not merely left to be forgotten;
- is **auditable**, meaning the issuance of the new instance and the retirement of the old one are recorded events.

**Rotation does not recover a compromised principal.** Rotating a credential leaves the logical identity, and every grant, in place. That is correct after a credential incident. It is wrong after a principal compromise (Decision 10).

This ADR selects no rotation interval, overlap duration, renewal protocol, or automation.

### 9. Revocation reduces authority within a bounded time, never instantly by assumption

```text
revoked credential != indefinitely usable authority
```

Two distinct operations exist:

| Operation | Target | Effect | Identity and grants |
| - | - | - | - |
| **Credential-instance revocation** | One credential instance | That instance stops being admissible | Logical identity, admission, and grants unchanged; other instances unaffected |
| **Identity retirement or revocation** | A logical workload identity | Every credential instance for it stops being admissible, and its admission is removed at its boundary | Identity no longer admitted; its grants no longer usable |

ADR-0008 already makes removal of an identity from admission configuration always available and sufficient by itself to deny a producer identity, whatever its certificates' state. This ADR extends the same property to upstream identities at intake. Credential-instance revocation is different. It requires the enforcement boundary to tell one instance of an identity from another. Decision 12 makes that capability a requirement of every conforming workload-authentication mechanism. How the distinction is derived and how revocation state refers to it are mechanisms this ADR does not select.

**When revocation takes effect.** After an identity is retired or a credential instance is revoked, admission using it fails closed at an enforcement boundary no later than the earlier of:

- the moment that boundary has incorporated the revocation into the trust state it enforces; and
- the end of the credential instance's own validity bound (Decision 7).

The architecture does not claim instantaneous global revocation. In a distributed deployment, or one with several enforcement instances (for example, gateway replicas), each boundary instance incorporates revocation state on its own schedule. The interval between a revocation decision and the point at which a given boundary stops admitting the credential is that boundary's **residual authority window**. The architecture requires that window to be:

- **finite.** At least one of the two bounds above must always apply;
- **governed.** The deployment knows and records what bounds it;
- **fail-closed at its limit.** A boundary never extends the window because it cannot learn whether a revocation occurred (Decision 13).

**Retired identities are not silently reused.** Re-admitting a retired identity is an explicit governed act. After a principal compromise, the retired identity value is never reused for the replacement principal (Decision 10).

### 10. Compromise recovery distinguishes the credential from the principal

| | Credential compromise | Principal compromise |
| - | - | - |
| Meaning | One or more credential instances may be controlled by someone else. The binding between the logical identity and its legitimate workload still holds. | The binding itself can no longer be trusted. Examples: an attacker can obtain new credentials for the identity through issuance or enrollment; the workload's implementation, configuration, or governing record is untrustworthy across its instances; the extent of compromise is unknown. |
| Required response | Revoke the affected instances (Decision 9). Issue replacements only through governed issuance (Decisions 6, 7). Preserve the logical identity. | Retire the logical identity at every boundary that admits it. Establish a **new** logical identity through full bootstrap (Decision 6). Re-admit it explicitly. Re-grant its authority explicitly. |
| Identity afterward | Same | New; the old value is never reused for the replacement |
| Grants afterward | Unchanged | Not inherited. Each admission and each ADR-0021 category grant is re-established by a governed act. |

The following rules apply:

- **Undetermined means principal.** If a deployment cannot establish that a compromise is limited to specific credential instances, it treats the incident as a principal compromise.
- **Stop accepting first.** Recovery begins by making the compromised credential or identity inadmissible, within the bounds of Decision 9. Issuing a replacement is never a substitute for revoking what was compromised.
- **Fail closed while uncertain.** While a boundary cannot establish whether a credential remains admissible, it does not admit it.
- **Attribution survives the transition.** Evidence recorded under the old identity or instance stays attributed to it. It is never re-attributed to the replacement. The governed record of the transition links old and new, so that later review can connect them. The replacement does not inherit the old identity's history as its own.
- **Reusing the value would revive the old credentials.** Reusing a compromised identity value for its replacement would make any unexpired credential issued under that value admissible again, including at boundaries that have not yet incorporated revocation. That is why the value is retired, not reused.

This ADR defines no incident-response workflow, tooling, or operator interface.

### 11. Implementation replaceability without accidental identity replacement

| Event | Logical identity | Credential instance | Re-admission |
| - | - | - | - |
| Restart of the same instance | Continues | May continue | No |
| Upgrade of the same workload, with unchanged role and scope | Continues | May continue, or be rotated | No |
| Reschedule, move, or reimage | Continues | New instance obtains its own credential through governed issuance (Decision 6) | No |
| New replica added | Continues (Decision 4) | New, independently issued instance | No |
| Replacement by another conforming implementation of the same logical workload | Continues only if a governed act confirms the replacement is that workload, with the same role and intended scope | New, through governed issuance | No, if continuity is confirmed |
| Change of role, intended scope beyond a governed widening, or owning authority | New | New | Yes |
| Principal compromise | New (Decision 10) | New | Yes |

Identity continuity is a governed fact. It is never inferred from a runtime slot. An arbitrary replacement has no credential for the identity unless governed issuance gives it one, and nothing it inherits counts as governed issuance (Decision 6). A legitimate replacement keeps its identity, admission, and grants without re-admission, because the identity it presents is the same governed principal.

### 12. Credential instances stay distinguishable; minimum evidence consequences

**Credential-instance distinguishability is required.** Whenever more than one credential instance can prove one logical workload identity, whether concurrently (replicas, Decision 4; rotation overlap, Decision 8) or over time, a conforming workload-authentication mechanism must make a **stable, non-secret credential-instance distinguisher** available at the enforcement boundary for every authentication. The distinguisher:

- identifies exactly one credential instance among all instances of that logical identity;
- stays the same for that instance for its whole validity, so that revocation state and evidence can refer to it;
- is not secret, and possessing or observing it confers no ability to authenticate;
- is established by the authentication itself at the enforcement boundary, never supplied by the caller in a request body or header outside the authentication mechanism (ADR-0008 "Identity derivation is gateway-derived, never caller-supplied").

This is what makes per-instance admissibility, per-instance revocation (Decision 9), and per-replica attribution (Decision 4) possible. A mechanism that cannot provide it, while allowing several instances per identity, is not conforming. That requirement does not depend on this ADR naming the distinguisher's form. Its derivation, representation, transport to the deciding component, storage, and schema are deferred mechanisms (**Deferred Decisions**).

**Minimum evidence.** For every admission decision at the producer-intake boundary and at `basis-gateway`, evidence must eventually be able to establish:

- which logical workload identity authenticated;
- which credential instance proved it, by its credential-instance distinguisher;
- whether that identity was admitted, and whether that instance was admissible, at the time of the decision;
- whether a lifecycle state, such as expiry, credential-instance revocation, identity retirement, or bounded trust state exceeded, determined the outcome.

Issuance, rotation, credential-instance revocation, identity retirement, principal replacement, and scope changes must each be recorded as attributable governed events. Each record identifies the authority that performed it.

For replicas (Decision 4), the credential-instance distinguisher is what keeps per-replica attribution when the logical identity is shared. Evidence must never record secret credential material, and the distinguisher must never be secret material.

This ADR states requirements only. The distinguisher's concrete form, the record shape, storage, correlation vocabulary, and operator representation belong to Workstream 4 evidence correlation and to later schema decisions.

### 13. Disconnected operation continues only on bounded, previously established trust state

BASIS targets deployments with intermittent or no connectivity to central services. Two outcomes are unsafe:

```text
network disconnected -> every credential immediately invalid          (rejected: availability failure by design)
network disconnected -> existing credentials remain valid indefinitely (rejected: authority without bound)
```

The rule is:

> **An enforcement boundary may keep admitting workloads while disconnected only on trust state it already holds locally — trust anchors, admission configuration, and revocation state — and only while that state is within explicit, governed bounds. Loss of connectivity never extends a credential's validity, never makes revocation state count as current, and never widens admission.**

The following rules apply:

- **Local validity is always evaluated.** Every credential's validity bound (Decision 7) is evaluated locally and enforced during disconnection. A credential that expires while its boundary is disconnected stops being admissible.
- **Local revocation always works.** Removing an identity from local admission configuration takes effect without connectivity, as ADR-0008 already requires for producers.
- **Remotely sourced revocation state has a bounded age.** Where revocation decisions originate outside the enforcement boundary, the boundary's locally held revocation state has a governed maximum age. Past that age, the boundary stops admitting credentials whose admissibility depends on that state, unless a credential's own validity bound already ends within the governed residual authority window (Decision 9).
- **Renewal needs governed issuance.** A disconnected workload can renew a credential only through governed issuance available to it locally. Otherwise its credential expires and the workload stops being admissible. That is an availability consequence, not a security failure, which is consistent with ADR-0008's treatment of expiry.
- **No component extends a bound.** No workload, ingress, gateway, or intake realization may extend a credential's validity, refresh revocation state it did not receive, or treat "could not check" as "not revoked."

A deployment chooses how to meet these bounds, for example with short credential validity, a local issuer, locally distributed revocation state, or a combination. That choice sets the length of its residual authority window. This ADR selects none of these mechanisms and no durations. Credential validity evaluation depends on a trustworthy enforcement-boundary clock, as ADR-0008 already records. Clock mechanisms are not decided here.

Credential validity is not request freshness. The bounded validity of an intake request (ADR-0018 Decision 7), context freshness (ADR-0021 Decision 7), and the broader authorization replay/freshness family (ADR-0012, ADR-0013, ADR-0016) are separate conditions. Satisfying one never satisfies another.

### 14. Roles outside this decision

- **Protocol executor.** Accepted architecture defines no independent executor workload credential. ADR-0011 names only a "possible future executor workload credential," needed if a separated producer/executor topology is ever adopted. In the accepted same-process topology, the executor role holds no workload credential of its own, and this ADR creates none. If a separated topology later defines an executor authentication boundary, that architecture decides whether and how these semantics apply to it (ADR-0011 Decision 5).
- **Device and protocol credentials.** Out of scope. This ADR preserves only the separation: producer workload credential ≠ device credential; upstream workload credential ≠ device credential; authorization-subject credential ≠ device credential. Device-credential custody and scoping remain with ADR-0011 Gate 4.
- **Authorization-subject credentials.** Out of scope. Subject authentication and subject-credential conveyance across intake remain governed by ADR-0008, ADR-0010, and ADR-0018 Decision 3. This ADR selects no delegation, token exchange, impersonation, grant type, or format.
- **Gateway server credentials and ingress-to-gateway trust.** Out of scope. ADR-0008 and ADR-0009 treat them as separate roles.

### 15. Platform neutrality

This ADR defines no type, field, identity namespace, topology, or rule named after, or shaped for, any upstream platform or deployment product. The same semantics apply to a supervisory platform, an automation service, an agent, a SCADA supervisory workload, a custom orchestrator, or any other upstream operational system, and to deployments with no upstream system at all.

## Security Invariants

When this decision is accepted and implemented, the following hold on the governed path:

```text
logical workload identity                 != credential instance
authorization subject identity            != upstream workload identity
                                          != producer workload identity
                                          != device identity
upstream workload identity                != producer workload identity
one credential instance                   -> exactly one logical identity, one role
rotation                                  != new authority
rotation                                  != new identity
replacement                               != automatic identity inheritance
implementation replacement                != implicit re-admission
colocation                                != identity equivalence
replica                                   != new logical principal (by default)
replicas                                  -> distinct credential instances
credential instances of one identity      -- distinguishable at the enforcement boundary
credential-instance distinguisher         != secret material
credential possession                     != authorization-subject authority
credential possession                     != device authority
credential possession                     != admission
shared trust root                         != admission
network location                          != identity
runtime name / host / slot                != identity
inherited credential material             != governed issuance
revocation                                != indefinite residual authority
disconnection                             != indefinite credential validity
could not check revocation                != not revoked
credential compromise                     != principal compromise
principal compromise                      -> new identity, explicit re-admission, no inherited grants
retired identity value                    -- never reused for a replacement principal
identity scope                            -- explicit, recorded, within one compromise domain
```

All ADR-0018 and ADR-0021 invariants are preserved, including upstream admission ≠ producer admission, producer relay ≠ producer origin, and admitted upstream ≠ trusted for all context.

## Threat Cases

**Rotated credential with a new identity value.** A renewal presents `…/producer-02` where `…/producer-01` was admitted. It is a different logical identity (Decision 1). Exact-match admission rejects it until that identity is explicitly admitted with its own scope and grants.

**Replacement inherits a key from disk.** A reimaged instance finds its predecessor's private key on a reused volume and authenticates with it. That is not governed issuance (Decision 6). A conforming deployment revokes the inherited instance or never lets material persist across reimaging, and the replacement obtains its own credential. The residual risk, where inherited material is still valid, is bounded by Decision 9.

**Upstream workload presents the producer's credential.** The producer identity is not admitted at intake (Decision 2), so intake rejects it. If the producer's credential ever reached the upstream workload, that is a custody failure. Treated as credential compromise of the producer (Decision 10), it does not make the upstream workload a producer.

**Shared key across replicas.** One replica of five is compromised. With a shared key, the only remedy is revoking the credential all five use. Under Decision 4, only the compromised replica's instance is revoked, and the other four continue.

**Compromised replica can enroll new instances.** The attacker can obtain fresh credentials for the identity. That is principal compromise (Decision 10). Revoking one instance is insufficient. The identity is retired and replaced, and its grants are re-established explicitly.

**Upstream failover.** An upstream source fails over to a replica with a different credential instance. ADR-0021 attribution and category grants continue unchanged, because the source is the logical identity (Decision 2).

**Disconnected gateway, revoked producer.** A producer credential instance is revoked centrally while a site's gateway is disconnected. The gateway stops admitting that instance no later than the instance's validity bound or its governed revocation-state age, whichever comes first (Decisions 9, 13). It never admits the instance indefinitely.

**Revived identity value.** After principal compromise, an operator re-admits the same identity value for a rebuilt workload. Old, unexpired credentials for that value would become admissible again. Decision 10 prohibits the reuse.

**Site-wide producer credential.** One producer identity is used across three sites that the deployment considers independent. Decision 5 prohibits it. Each independent compromise domain needs its own identity.

## Failure Behavior

Each outcome is semantic. This ADR defines no error codes, status vocabulary, or wire representation.

| Condition | Detected at | Outcome |
| - | - | - |
| Credential chains to an accepted root, but its identity is not admitted for this boundary's role | Enforcement boundary | Rejected |
| Credential proves an identity admitted only for the other role | Enforcement boundary | Rejected |
| Credential presents zero or more than one identity | Enforcement boundary | Rejected (ADR-0008 for producers) |
| Credential expired, or not yet valid | Enforcement boundary (for producers, the ADR-0009 ingress) | Rejected |
| Credential instance revoked, and revocation incorporated | Enforcement boundary | Rejected |
| Identity retired or removed from admission | Enforcement boundary | Rejected |
| Remotely sourced revocation state older than its governed maximum age | Enforcement boundary | Rejected, for credentials that depend on it (Decision 13) |
| Admissibility cannot be determined, for example because trust state is unavailable or indeterminate | Enforcement boundary | Rejected; never treated as permissive |
| Workload credential expired and cannot be renewed through governed issuance | Workload | Workload cannot authenticate; it fails closed locally |

A rejection at intake is an ADR-0018 Decision 6 rejected or failed outcome. A rejection at `basis-gateway` happens before kernel evaluation, as ADR-0008 already requires. Each belongs to the "not executed" family (`operation-producer-and-execution-boundary.md` §8).

## Relationship to Existing ADRs

This ADR modifies no Accepted ADR's body.

- **ADR-0008.** Unchanged: mTLS profile, URI SAN identity, exact-match admission, trust anchor ≠ admission, and existing revocation paths. This ADR answers, at the level of durable semantics, two ADR-0008 deferrals: "full credential lifecycle" (the automation remains deferred) and "whether admission entries require an environment/operational scope field" (intended scope is recorded, Decision 5; no field is selected). It generalizes ADR-0008's renewal naming discipline into Decision 1. Under the ADR-0008 profile, each credential instance is a distinct certificate, so the profile can satisfy Decision 12's distinguishability requirement. Which stable, non-secret value is derived from the certificate is deferred.
- **ADR-0009.** Unchanged. The trusted ingress remains where producer certificate chain and validity are checked. How credential-instance revocation state reaches the ingress or the gateway is a deferred mechanism.
- **ADR-0010.** Unchanged. Producer credential custody stays with the operation-producer runtime. Decision 6 governs how a runtime instance comes to hold a credential, and Decision 4 requires each instance to hold its own.
- **ADR-0011.** Credential classes (Decision 14) unchanged. No executor workload credential is created (Decision 14 here). Gate 4 keeps device-credential custody.
- **ADR-0012, ADR-0013, ADR-0016.** Unchanged. Credential validity is separate from authorization replay and freshness.
- **ADR-0018.** Supplies the BASIS-side semantics of the "Producer credential lifecycle (cross-ecosystem Workstream 3E)" deferral: issuance, rotation, revocation, HA identity, bootstrap, compromise handling, and disconnected operation. The intake authentication *mechanism* stays deferred with the intake transport. Decision 3's identity separation and Decision 8's credential-class separation are extended through every lifecycle event.
- **ADR-0020.** Unchanged. Governed configuration remains attributable to an administrative authority, in the same sense used here.
- **ADR-0021.** Unchanged. Decision 2 guarantees that the originating source `U` and relaying producer `P` that ADR-0021 relies on are stable logical identities across rotation, failover, and identity-preserving replacement. It also guarantees that category grants are not inherited by a replacement principal.

## Alternatives Considered

**Treat each credential as its own identity.** Rotation would then change identity, and every renewal would require re-admission and re-granting. Operators would learn to re-admit routinely, which would make re-admission an unreviewed formality. Rejected.

**Derive identity from runtime placement (host, address, scheduler slot, image).** That is network or placement trust, which ADR-0008 and ADR-0018 already reject. It would also let any workload scheduled into an admitted slot inherit authority. Rejected.

**Per-replica logical identities as the default (Option B).** Rejected as the default for the reasons in Decision 4. It remains available where a deployment explicitly wants separate principals.

**Declared per-admission replica model (Option C).** Rejected as a mode flag. It adds configuration without adding a guarantee (Decision 4).

**Shared credential across replicas.** Rejected. It destroys per-replica attribution and turns a single-replica incident into whole-workload revocation.

**Instantaneous global revocation as a requirement.** Rejected, because a deployment with disconnected or independent enforcement points cannot deliver it. Requiring it would make the architecture claim a guarantee it cannot keep. The bounded residual authority window is stated instead.

**Fail every credential closed on disconnection.** Rejected. It turns every network interruption into a total outage, contrary to [`architecture-principles.md`](../architecture-principles.md) §12, "Security Must Respect Operational Constraints," and to ADR-0008's air-gap posture.

**Keep credentials valid until reconnection.** Rejected. It gives authority with no bound.

**Rotate in place after principal compromise.** Rejected. It preserves exactly the authority that can no longer be trusted.

**Select a workload-identity technology now** (a PKI product, workload-identity framework, or secret manager). Rejected as premature, for the same reason ADR-0008 did not require SPIRE and ADR-0018 selected no intake transport. The semantics must hold under whichever mechanism a later decision selects.

## Consequences

### Positive

- Admission, ADR-0021 category grants, and attribution now attach to an identity whose stability is itself governed. Lifecycle events can no longer broaden them silently.
- HA has explicit identity semantics that preserve both logical attribution and per-replica attribution and revocation.
- The compromised-credential blast radius becomes a recorded, reviewable fact (intended scope, one compromise domain) with a finite, governed time bound.
- Credential incidents and principal incidents have different, explicit recoveries, and a principal compromise can no longer be "fixed" by rotation.
- Disconnected operation has a stated safety bound that does not depend on connectivity.

### Negative / Tradeoff

- Deployments must record intended scope and compromise domains, keep an attributable record of issuance and revocation, and issue distinct credentials per replica. That is more operational work than a shared credential or an implicit topology.
- Per-instance revocation requires a mechanism the current implementation lacks. Until one exists, revoking one replica's credential without affecting the others is possible only through expiry or trust-anchor changes.
- The residual authority window is real. Deployments that want it short must accept short credential validity, and therefore more frequent renewal, or must distribute revocation state reliably.
- Principal compromise requires re-establishing every grant for the new identity, which takes time during an incident.

### Security Consequences

**Prevented, when implemented:** identity drift through rotation; authority inheritance through placement, inherited material, or naming; dual-use upstream/producer credentials; source re-attribution through failover; indefinite authority after revocation or during disconnection; recovery by rotation from a principal compromise; revival of a retired identity; one identity silently spanning independent compromise domains.

**Still requiring follow-on architecture or mechanism:** the lifecycle mechanism (issuance, enrollment, custody, rotation automation, revocation distribution, credential-instance revocation); the intake authentication mechanism; enforcement of producer intended scope against targets; evidence representation (Workstream 4); clock trust; executor authentication for a separated topology; device-credential custody (Gate 4).

**Residual risks:** a compromised issuer can mint credentials for an admitted identity (ADR-0008 **Compromised producer CA**). Admission limits this to already-admitted identities but does not prevent it. The residual authority window after revocation is bounded, not zero. A compromised enforcement boundary can ignore any of this, as ADR-0008 already records for the gateway. Colocated producer/executor runtimes concentrate producer and device authority (ADR-0011 Decision 19).

## Deferred Decisions

These are deliberately not decided here, so that later work does not interpret silence as authorization. Where an existing gate already owns a question, it stays there.

- **Workload credential lifecycle mechanism.** Issuance and enrollment protocol, bootstrap-secret or attestation form, credential custody and storage, rotation automation, the derivation, representation, transport, and storage of the credential-instance distinguisher (its existence is required by Decision 12, not deferred), revocation-state distribution and its maximum age, local issuance for disconnected sites, and validity durations. This narrows the `ROADMAP.md` Phase 4 items "Certificate and credential lifecycle management tooling" and "Credential revocation propagation" to mechanism. It is one gate, not several.
- **Upstream workload authentication mechanism.** This stays with ADR-0018's deferred intake transport, envelope, and integrity mechanism. Whatever it selects must satisfy Decisions 1 through 13.
- **Producer intended-scope enforcement.** Whether, and how, `basis-gateway` restricts a producer identity to targets within its recorded intended scope. This is an authorization question, not a credential question.
- **Evidence representation** of identity, credential-instance distinguisher, and lifecycle facts (Workstream 4 and later schema decisions).
- **Clock trust and skew** for validity evaluation at enforcement boundaries.
- **Executor workload authentication** for a separated producer/executor topology (ADR-0011 Decision 5).
- **Device-credential custody and scoping** (ADR-0011 Gate 4).
- **Subject-credential conveyance** (ADR-0010, ADR-0018).
- **Deployment topology, HA orchestration, and site decomposition** (`ROADMAP.md` Phase 4; cross-ecosystem Workstream 5). This ADR requires only that each compromise domain has its own identity.
- **Gateway configuration surface** for admission, intended scope, and trust state (ADR-0008 deferral, unchanged).

## Non-Goals

This ADR does not: select a PKI, CA hierarchy, workload-identity framework, secret manager, hardware key store, cloud identity service, service mesh, or scheduler; define a certificate profile or SAN naming convention beyond ADR-0008; define rotation intervals, overlap durations, validity lifetimes, or revocation-state ages; define an enrollment API, bootstrap token, or revocation transport; design HA, load balancing, or site topology; define device-credential custody; define subject-credential conveyance; define the intake API or envelope; define an evidence schema or close Workstream 4; perform a full threat model (Workstream 6); modify any implementation repository or schema; modify any Accepted ADR's body; modify Ipotio's repository; close ARCH-GAP-011; accept ADR-0019 or perform any branding migration; or promote its working vocabulary to the glossary.

## Implementation Status

Architecture decision ≠ implementation. Nothing in this ADR is implemented, and acceptance would authorize no implementation. Measured against this decision, current implementation stands as follows:

| Area | Current state | Gap against this decision |
| - | - | - |
| Producer identity and admission (`basis-gateway`) | Exact URI SAN match against static `OPERATION_PRODUCER_MTLS_ADMITTED_URIS` | Consistent with Decisions 1–3. No recorded intended scope (Decision 5). |
| Producer credential validation (ADR-0009 ingress) | Chain and validity checked against a configured CA bundle; no revocation list configured in the reference template | No credential-instance revocation (Decision 9); no bounded revocation-state age (Decision 13) |
| Producer audit (`basis-gateway`) | Records the logical producer identity | No credential-instance distinguisher is derived or recorded, although the ADR-0009 handoff already carries the full leaf certificate; no lifecycle-state reason (Decision 12) |
| Producer credential custody (`basis-producer`) | One static certificate and key loaded from file paths | No governed enrollment, renewal, or rotation (Decisions 6–8); per-replica credentials not addressed (Decision 4) |
| Upstream workload credential (intake) | No intake realization exists | Entire lifecycle unimplemented; mechanism not selected (ADR-0018) |
| Issuance, revocation distribution, lifecycle records | None in any BASIS component | Decisions 6, 7, 9, 12 unimplemented |

## Relationship to Cross-Ecosystem Use

*Non-normative.* An upstream platform that integrates through the producer-intake boundary consumes this decision the same way it consumed ADR-0020 and ADR-0021: it reconciles its own architecture against these semantics in its own repository. Ipotio tracks that reconciliation as Workstream 3E-B for **ARCH-GAP-011**.

- **BASIS-side prerequisite:** proposed by this ADR. It is established only when this ADR is accepted.
- **ARCH-GAP-011:** remains open. This ADR does not close it. Closure requires (1) acceptance of this ADR and (2) the upstream platform's Workstream 3E-B reconciliation in its own repository.

That reconciliation would need to confirm, at least:

- that the upstream platform's requesting-workload identity is one role-specific logical identity, stable across its own rotation, failover, and replacement, and is the originating source identity ADR-0021 grants name (Decisions 1, 2);
- that it never holds, receives, or presents a producer credential (Decision 2);
- how its producer runtimes map onto compromise domains, and that no producer identity spans independent ones (Decision 5);
- that its HA replicas share logical identities and hold distinct credential instances (Decision 4);
- that its bootstrap, rotation, revocation, compromise recovery, and disconnected-site behavior satisfy Decisions 6 through 13, whatever mechanism it chooses;
- that device credentials remain separate and governed by Gate 4 (Decision 14).

Nothing here requires a BASIS type, field, or vocabulary specific to any upstream platform. This ADR does not write any upstream platform's architecture.

## References

- [ADR-0008](0008-producer-workload-authentication-and-admission.md): producer mTLS authentication and admission; trust anchor ≠ admission; revocation paths; lifecycle deferral
- [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md): trusted mTLS ingress
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md): producer credential custody
- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md): credential classes (Decision 14); separated topology (Decision 5); deployment security reality (Decision 19); Gate 4
- [ADR-0012](0012-authorization-to-execution-binding.md), [ADR-0013](0013-execution-lifecycle-semantics.md), [ADR-0016](0016-bounded-target-replay-freshness-posture.md): replay and freshness family, kept separate
- [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md): producer-intake boundary; identity separation (Decision 3); credential classes (Decision 8); Workstream 3E deferral
- [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Proposed): Basitra terminology
- [ADR-0020](0020-operation-to-authorization-mapping-and-composition-boundary.md): governed configuration; credential lifecycle open
- [ADR-0021](0021-upstream-context-assertion-trust-boundary.md): originating source and relay identities; credentials deferred (Decision 13)
- [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md): §3 trust establishment; §8 not-executed family; §9 topologies
- [`producer-mtls-proxy-trust-boundary.md`](../architecture/producer-mtls-proxy-trust-boundary.md): ingress trust boundary
- [`architecture-principles.md`](../architecture-principles.md): §2 identity over network-origin trust; §8 least privilege; §12 security must respect operational constraints
- [`threat-model.md`](../security/threat-model.md): §7.2, §7.5, §7.7, §11
- `ROADMAP.md` Phase 4: credential lifecycle tooling; revocation propagation; HA topology
- `basis-gateway` `1e08348`: `src/basis_gateway/config.py` (`OPERATION_PRODUCER_MTLS_ADMITTED_URIS`), `src/basis_gateway/audit/operation_aware_gateway_events.py` (`operation_producer_workload_identity`), `examples/producer-mtls/nginx.conf.template` (read-only evidence)
- `basis-producer` `8c7410a`: `src/basis_producer/gateway_client.py`, `src/basis_producer/gateway_client_config.py` (read-only evidence)
- Ipotio architecture repository: ARCH-GAP-011 and Workstream 3E-B (cross-repository reference, not hyperlinked)
