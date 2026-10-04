# ADR-0023: Supervisory-Platform and Administrative-Interface Boundary

## Status

Proposed

Consistent with this repository's established practice (see ADR-0018's Context and ADR-0019's Status), this ADR is numbered when first proposed and submitted as `Proposed`. Merging it does not accept it. Acceptance requires a separate, dedicated review. Until then, the clarifications below are a proposal, and the accepted decisions they compose remain authoritative on their own terms.

## Context

Accepted architecture already separates the parties on the governed path from operator intent to an OT target:

- [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §2 defines the *operation initiator*. It states that "an operator using a supervisory HMI is an operation initiator whose request is carried, but not authenticated as, the operation producer that actually submits to the gateway."
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) (Accepted) makes the producer workload identity and the authorization subject structurally separate. "A producer does not become authoritative for subject identity merely because its mTLS connection is trusted."
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) (Accepted) establishes the operation-producer runtime. [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) through [ADR-0016](0016-bounded-target-replay-freshness-posture.md) (Accepted) define the protocol-executor role, authorization-to-execution binding, execution lifecycle, execution evidence, and the first bounded target.
- [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md) (Accepted) defines the producer-intake boundary through which an upstream supervisory system submits supervisory intent. It keeps the authorization subject, the upstream workload identity, and the producer workload identity distinct (Decision 3). It states that intake admission is neither gateway admission nor authorization (Decision 2).
- [ADR-0020](0020-operation-to-authorization-mapping-and-composition-boundary.md) (Accepted) Decision 6 states that a disposition obtained on the direct, non-producer path, "such as `basis-console`'s direct evaluation," is not bound to any preserved operation and cannot support dispatch.
- [ADR-0021](0021-upstream-context-assertion-trust-boundary.md) (Accepted) makes context-assertion authority category-scoped and states that context never establishes or substitutes for the authorization subject. [ADR-0022](0022-workload-credential-lifecycle-and-scope-boundary.md) (Accepted) governs workload credential lifecycle.

Taken together, that architecture already places the origin of operator-driven OT intent outside BASIS, in an upstream supervisory system. It places authorization, governed execution, and evidence inside BASIS. No document states the resulting product boundary directly, and no document states how a human's interactive access to a BASIS user interface relates to that boundary. Several passages can be read the other way:

- [`basis-console.md`](../architecture/basis-console.md) uses *operator* throughout to mean a human who operates the authorization system. Its **Administrative Interaction** section lists "Initiate requests — triggering gateway-authenticated operations appropriate for the operator's role." Without a stated boundary, a reader can take "operations" to include OT device operations and *operator* to mean an OT operator. The same document's **Device Management** non-responsibility says otherwise, but the two are not connected.
- ADR-0010 (Accepted) states that "`basis-console` owns the operator experience." In context, that line restates existing component ownership without change, so it inherits `basis-console.md`'s meaning. Read alone, it could be taken to mean the OT operator's experience.
- [`basis-ecosystem.md`](../architecture/basis-ecosystem.md) and [`basis-gateway.md`](../architecture/basis-gateway.md) describe the console as supporting "basic operational management" and "operational management" without saying what is being managed.
- The glossary's **Administrative Interface** entry describes "initiating operational requests through established enforcement boundaries."
- [`operator-and-training-experience.md`](../roadmaps/operator-and-training-experience.md) refers to "whichever future phase introduces consequential, execution-capable operator actions" and to "a mature operational platform." It does not say whether such actions would originate in the console or through what path.
- `basis-console` includes a Decision Simulator that constructs and submits evaluation requests to `basis-gateway`. Nothing states, at the component-architecture level, that this submission is not an OT-control path.

None of these passages contradicts an accepted decision. Each leaves room for a reasonable reader to conclude that `basis-console` is meant to be the normal supervisory or control interface for OT devices, or that logging in to a BASIS interface is how a human becomes an authorization subject. The upstream-integration work that produced ADR-0018 and ADR-0020 through ADR-0022 makes that reading costly. If it took hold, it would create a second, BASIS-native origin of OT intent with an implied privileged relationship to the gateway and executor. It would also make an interactive BASIS session a precondition for authorizing operations initiated elsewhere.

**Terminology.** This ADR uses *BASIS*, not the proposed *Basitra* ([ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md), Proposed). The decision is about architectural roles, not names. It applies unchanged under whatever terminology ADR-0019 settles. *Upstream supervisory system*, *producer-intake boundary*, and *upstream workload identity* are ADR-0018's working vocabulary. This ADR uses *supervisory platform* as a descriptive synonym for ADR-0018's upstream supervisory system when describing the human-facing operational application. *BASIS administrative interface*, *administrative context*, and *OT operation-initiation authority* are introduced below as working vocabulary under the same convention, and are not promoted to the glossary. In this ADR, *OT operator* means a human who operates OT equipment through a supervisory platform. *BASIS administrator* means a human who administers, inspects, or diagnoses BASIS itself. Existing documents that use *operator* for the second meaning are not rewritten.

### Sufficiency of existing architecture

Before deciding, this ADR checked whether any unresolved decision blocks stating the boundary. None does.

- The origin of upstream intent, the intake boundary, and the three-identity separation are fixed by ADR-0018.
- Subject authentication at the gateway and its separation from producer authentication are fixed by ADR-0008. The identity engine's role is fixed by [`basis-identity.md`](../architecture/basis-identity.md).
- That a direct-path disposition cannot support dispatch is fixed by ADR-0011 Decision 9, ADR-0012, and ADR-0020 Decision 6.
- The console's non-responsibilities (no authorization, no independent authentication, no protocol handling, no device management) are fixed by `basis-console.md`.

The open questions are mechanisms: subject-credential conveyance, intake transport, and federation profile. They are recorded under **Deferred Decisions** and do not change the boundary decided here.

## Problem

How does BASIS state, durably and precisely, that:

- it is an authorization and security substrate for OT operations, not the supervisory or operator platform through which humans normally operate OT devices;
- a human can be the authorization subject of an OT operation initiated through a supervisory platform without holding an interactive BASIS session;
- interactive access to a BASIS user interface creates a BASIS administrative context only, never OT operation-initiation or execution authority;
- no BASIS-native tool has a privileged path around the governed producer, authorization, binding, execution, and evidence chain;

without weakening any accepted separation, selecting any mechanism, or making any one supervisory platform a special case?

## Decision

**BASIS is an authorization and security substrate for OT operations. It is not a supervisory platform. Operator-driven OT intent originates in an upstream supervisory platform. BASIS independently authenticates, authorizes, governs, enforces, and evidences that intent through the accepted chain, without requiring the subject to hold an interactive BASIS session. Interactive access to any BASIS user interface, including `basis-console`, establishes a BASIS administrative context only. It confers no OT operation-initiation authority, no execution authority, and no authorization-subject standing on any OT operation. No BASIS-native tool has a path around BASIS's own security architecture. BASIS may consume its own security architecture. It may not bypass it.**

```text
supervisory platform  =  originates operator-driven OT intent
BASIS                 =  authenticates, authorizes, governs, enforces, and evidences that intent
```

```text
Human or non-human subject
        │  authenticates through the supervisory platform's
        │  legitimate identity path (possibly a shared enterprise IdP — Decision 8)
        ▼
Upstream supervisory platform          (outside BASIS; owns the operator experience
        │                               and supervisory intent — ADR-0018 Decision 1)
        │  governed supervisory request, carrying subject provenance
        ▼
Producer-intake boundary               (BASIS; authenticate and admit the upstream workload;
        │                               preserve subject provenance — ADR-0018 Decisions 2, 3)
        ▼
Operation-producer role                (BASIS; produce the governed operation;
        │                               authenticate as the producer workload — ADR-0008, ADR-0010)
        │  independently verifiable subject context
        ▼
basis-gateway                          (independently establish the authorization subject;
        │                               compose; invoke authorization — ADR-0008, ADR-0020, ADR-0021)
        ▼
basis-core                             (authorization outcome)
        │  authoritative disposition + binding (ADR-0012)
        ▼
Protocol-executor role                 (bounded dispatch; execution evidence — ADR-0011, ADR-0013, ADR-0014)
        ▼
OT target


BASIS administrator ──► basis-console ──► basis-gateway      (administrative, inspection,
                                                              diagnostic, and simulation APIs only;
                                                              no path to the producer-intake boundary,
                                                              the binding, or the protocol executor)
```

The upper chain is ADR-0018's diagram with its human origin made explicit. Every arrow remains a separate boundary with its own admission or verification rule, and none inherits trust from the one above it. The lower line is separate. It never joins the upper chain except as Decision 7 describes. The diagram shows logical roles, not a deployment prescription.

### 1. BASIS is an authorization and security substrate, not a supervisory platform

BASIS owns authorization and the governed security enforcement around OT operations:

- authenticating and admitting the workloads on the governed path;
- establishing the authorization subject;
- evaluating authorization in `basis-core`;
- binding a permitted decision to the exact operation authorized;
- governing dispatch through the protocol-executor role;
- producing authorization and execution evidence.

BASIS does not own the operational experience through which humans monitor and operate OT equipment. That means the asset model, supervisory state, alarm handling, scheduling, operator workflows, and the decision to request an operation all stay with the supervisory platform. This restates, from the BASIS side, ADR-0018 Decision 1: "The upstream supervisory system keeps ownership of its own application semantics... BASIS does not absorb them."

BASIS does not become a supervisory platform because it has a user interface. A BASIS user interface exists for BASIS responsibilities (Decision 5). Its existence does not move the origin of OT intent into BASIS.

### 2. Normal OT operations originate in an upstream supervisory platform

A human or non-human subject who wants to operate OT equipment does so through a supervisory platform. That platform originates a governed request on the subject's behalf and submits it across the producer-intake boundary (ADR-0018). The category is generic. It includes:

- a building-management or building-automation supervisory platform;
- an industrial supervisory, HMI, or SCADA application;
- an orchestration, scheduling, or maintenance-workflow system;
- another admitted operational application.

Ipotio and Niagara are examples. Neither receives special treatment, and the rule applies identically to every platform in the category (Decision 11).

```text
OT operator
    ↓
supervisory platform (for example, Ipotio or Niagara)
    ↓
governed request across the producer-intake boundary
    ↓
BASIS: subject establishment, authorization, binding
    ↓
governed execution
```

The supervisory platform remains responsible for the normal operational experience. BASIS remains responsible for whether the requested operation is authorized and how a permitted operation is governed to its target.

### 3. The authorization subject does not require an interactive BASIS session

A human or non-human subject may be the authorization subject of an OT operation without ever opening a BASIS user interface. An interactive BASIS session is not a prerequisite for being an authorization subject.

```text
subject authentication  !=  interactive BASIS login
authorization subject   !=  BASIS console user
```

The subject is established through the governed identity path for that subject. That path is today `basis-gateway`'s `authenticate()` dispatch and, potentially, a `basis-identity` canonical identity context (ADR-0008 "Producer vs. authorization subject"; ADR-0018 Decision 3). Identity proof for the subject may originate from, or federate with, an external authority (Decision 8). Where a deployment's identity chain uses a BASIS-local session or token issued by `basis-identity` ([`basis-identity.md`](../architecture/basis-identity.md)), that artifact is identity establishment for the subject. It is not a `basis-console` session, and it confers no authorization. This ADR does not require, and does not prohibit, `basis-identity` participating in establishing the subject of an upstream-initiated operation. It requires only that the subject not need to operate a BASIS user interface to be established.

How subject-related material is conveyed from a supervisory platform to the operation-producer role remains the subject-credential conveyance decision that ADR-0010 and ADR-0018 defer.

### 4. What BASIS independently establishes

For an operation initiated through a supervisory platform, BASIS establishes each of the following itself. None is inherited from the supervisory platform, and none substitutes for another:

| # | BASIS establishes | Where | Governing decision |
| - | - | - | - |
| 1 | The upstream workload is authenticated and admitted to submit intake requests | Producer-intake boundary | ADR-0018 Decision 2; ADR-0022 |
| 2 | The authorization subject is established through the governed identity path, from independently verifiable subject context, never from the upstream workload's authentication or assertion | `basis-gateway`, with `basis-identity` where deployed | ADR-0008; ADR-0018 Decision 3 |
| 3 | The operation-producer workload is authenticated and admitted | `basis-gateway` (with the ADR-0009 ingress) | ADR-0008; ADR-0009; ADR-0022 |
| 4 | Authorization is evaluated | `basis-core`, invoked by `basis-gateway` | ADR-0020 (composition); ADR-0021 (context admissibility) |
| 5 | A permitted operation is bound and governed to its target | Operation-producer and protocol-executor roles | ADR-0011; ADR-0012; ADR-0013 |
| 6 | Authorization and execution evidence are recorded | Owning components | ADR-0007; ADR-0014; ADR-0018 Decision 6 |

A supervisory platform cannot assert that a subject is authorized and thereby compel BASIS to permit an operation. Authorization authority belongs only to `basis-core` through `basis-gateway` (ADR-0018 Decision 1). A supervisory platform's claim about a subject that is not bound to a verifiable subject authentication remains an unverified hint (ADR-0018 Decision 3; `operation-producer-and-execution-boundary.md` §7). A trusted context assertion is not an established subject (ADR-0021).

A supervisory platform may convey identity and provenance that allow BASIS to establish the subject independently. Such material includes a subject reference, a subject credential, or an identity assertion (ADR-0018 Decision 3). Conveying it is permitted. What BASIS accepts as subject establishment is decided by the BASIS identity chain, not by the conveyor.

```text
upstream supervisory authentication  !=  BASIS authorization
upstream workload authentication     !=  subject authentication
producer authentication              !=  subject authentication
intake admission                     !=  gateway admission
intake admission                     !=  authorization
ALLOW                                !=  DISPATCHED
```

### 5. BASIS administrative access confers no OT operation authority

A human may authenticate interactively to a BASIS user interface for BASIS-owned responsibilities, such as:

- authorization-policy administration;
- security, identity, and trust configuration;
- producer and upstream-workload admission configuration;
- mapping and category-trust configuration;
- audit and evidence inspection;
- diagnostics, system health, and security posture;
- security analytics appropriate to BASIS;
- other BASIS-owned administration.

That login establishes a **BASIS administrative context** only. Administrative actions taken in it are themselves authenticated and enforced at `basis-gateway` and evaluated by policy, as `basis-console.md` and the [threat model](../security/threat-model.md) §3.4 already require. The administrator is the authorization subject *of those administrative actions*. The administrative context does not inherently establish:

- OT operation-initiation authority;
- execution ownership or execution authority;
- device-control authority;
- authorization-subject standing on any OT operation.

```text
BASIS console login                !=  OT control session
BASIS administrative access        !=  OT operation-initiation authority
BASIS administrator                !=  OT operator
BASIS administrative identity      !=  authorization subject of an OT operation, by virtue of administration
BASIS user interface               !=  supervisory platform
policy administration              !=  device operation initiation
security administration            !=  execution authority
```

One human may be both a BASIS administrator and an OT operator. The two roles stay separate. The person's OT operations still originate in a supervisory platform, and their administrative session is not their subject context on those operations. Administrative grants and OT operation grants are separate policy questions about separate actions. Holding one never implies the other.

Changing policy is not operating a device. An administrator able to change policy can change what a future governed request would be permitted to do. That ability gives the administrator no path to originate that request. Origination still requires an admitted supervisory platform or another admitted producer path (Decision 7), and the request still traverses the full chain. The risk that an administrator misuses policy authority is the existing malicious-insider and policy-integrity concern (threat model §4.2; §2.2, policy integrity). Policy-change controls such as review, separation of duties, and audit are not decided here.

### 6. `basis-console` is not the production OT supervisory workstation

`basis-console` is not the production supervisory or operator workstation for managing OT devices. Its role is BASIS-facing human interaction:

- security administration;
- inspection;
- diagnostics;
- policy and security visibility;
- evidence and audit review;
- identity and security visibility;
- testing and simulation where explicitly labeled.

This states directly what `basis-console.md` already implies through its **Device Management** non-responsibility ("It is not a SCADA system, a building automation system...") and its prohibition on submitting protocol commands. Where `basis-console.md`, ADR-0010, the glossary, or the operator-and-training roadmap use *operator*, *operator experience*, *operational management*, or *operational requests* for the console, those terms refer to operating, administering, and investigating BASIS itself. They do not refer to operating OT equipment.

**Simulation, diagnostic, and test submission is not an OT-control path.** A simulator, diagnostic, or test client, such as the Decision Simulator, may construct and submit a request to `basis-gateway` for evaluation. That submission is on the direct, non-producer path (ADR-0020 Decision 6). The disposition it returns is not bound to any preserved operation and cannot support dispatch (ADR-0011 Decision 9; ADR-0012). An `ALLOW` on that path is evaluation information, not permission to execute. Such tools remain legitimate and are not removed by this decision. They must keep simulation and diagnostic results distinguishable from governed operations, consistent with the existing requirement to "distinguish simulation from live action at all times" ([`operator-and-training-experience.md`](../roadmaps/operator-and-training-experience.md), **Safety and Confirmation Design**). This ADR does not define how that distinction is represented.

```text
simulation / test submission  !=  privileged OT-control path
```

### 7. BASIS-native tools consume, never bypass, BASIS's security architecture

No BASIS user interface, administrative session, or BASIS-native tool receives implicit OT operation-initiation or execution authority. Being part of BASIS confers no privilege on the governed path.

BASIS is not defined as permanently incapable of containing a reference producer, test harness, or emergency tool that originates a real governed operation. If any BASIS-native capability is ever permitted to do so, it must:

- be explicitly architected as a governed producer or upstream source by its own decision;
- traverse the same governed intake, authentication, admission, subject establishment, authorization, binding, execution, and evidence path as any other origin;
- receive no special trust, admission, category authority, or bypass because it is part of BASIS;
- keep its own workload identity distinct from the administrative context of any human using it, and keep that human's administrative context distinct from their subject context (Decision 5).

```text
BASIS may consume BASIS's security architecture.
BASIS may not bypass BASIS's security architecture.
```

This ADR authorizes no such capability. Break-glass and emergency-override procedures, which [`architecture-principles.md`](../architecture-principles.md) §11 requires to be "defined, constrained, and audited," remain future architecture. Any such procedure is bound by this Decision.

### 8. Shared enterprise identity is permitted; authentication authority is not authorization authority

BASIS and supervisory platforms may federate with the same enterprise identity authority, and a human may experience seamless single sign-on across them:

```text
              Enterprise IdP
             /      |       \
            /       |        \
      Ipotio    Niagara    BASIS (basis-identity / basis-gateway)
```

This is consistent with `basis-identity`'s federated authority mode, in which the external IdP remains authoritative ([`identity-authority-modes.md`](../architecture/identity-authority-modes.md)). Sharing it does not merge any of the following:

```text
shared authentication authority  !=  shared authorization authority
same external human  !=  same logical identity  !=  same grant  !=  same session
```

- **Each relying party establishes its own security context.** A supervisory platform's session, and any authentication artifact issued to it, is that platform's context. A shared IdP does not by that fact make it a BASIS subject authentication. Any subject proof BASIS accepts must satisfy ADR-0018 Decision 3 through the BASIS identity chain.
- **One human may correspond to several identities.** A supervisory-platform account, a BASIS canonical subject resolved by `basis-identity`, and a BASIS administrative identity may all refer to the same human. Correspondence among them is established by the governed identity chain (subject resolution), never by name equality or a shared login.
- **Authorization stays with each ecosystem.** BASIS authorization is decided by `basis-core` under BASIS policy. The supervisory platform's own permissions, the IdP's group memberships, and a seamless SSO experience are inputs at most, through governed identity normalization. They are never substitutes.

This ADR selects no federation protocol, token format, token exchange, delegation scheme, or SSO technology. Mechanisms already selected by accepted decisions keep exactly their current scope.

### 9. Consolidated invariants

The following hold together. Each is stated or composed above:

```text
BASIS console login                 !=  OT control session
BASIS administrative access         !=  OT operation-initiation authority
BASIS administrator                 !=  OT operator
BASIS administrative identity       !=  authorization subject, merely by being an administrator
BASIS user interface                !=  supervisory platform
BASIS authorization subject         !=  BASIS console user
upstream supervisory authentication !=  BASIS authorization
upstream workload authentication    !=  subject authentication
producer authentication             !=  subject authentication
subject authentication              !=  interactive BASIS login
policy administration               !=  device operation initiation
security administration             !=  execution authority
simulation / test submission        !=  privileged OT-control path
shared authentication authority     !=  shared authorization authority
intake admission                    !=  gateway admission
intake admission                    !=  authorization
ALLOW                               !=  DISPATCHED
```

Every authorization, execution, and evidence separation in ADR-0007 through ADR-0022 is preserved unchanged.

### 10. Mechanism neutrality

This ADR fixes roles and authority boundaries only. It does not select or constrain:

- the intake transport, envelope, or integrity mechanism (ADR-0018);
- the subject-credential conveyance mechanism (ADR-0010, ADR-0018);
- a federation protocol, token format, token exchange, or delegation profile;
- a console authentication flow beyond what `basis-console.md` already states;
- any user-interface design, labeling scheme, or API.

### 11. Platform neutrality

This ADR defines no BASIS type, field, role, or vocabulary named after, or shaped for, any supervisory platform. Ipotio and Niagara appear only as examples. A platform-specific integration is a conforming use of this boundary, never a modification of it (ADR-0018 Decision 9).

## Boundary Questions

| Question | Answer | Basis |
| - | - | - |
| Is BASIS an OT supervisory or control platform? | No. It is an authorization and security substrate. | Decision 1 |
| Where do normal human OT operations originate? | In an upstream supervisory platform, across the producer-intake boundary. | Decision 2; ADR-0018 |
| Must a subject log in to BASIS interactively to be an authorization subject? | No. | Decision 3 |
| What does BASIS independently establish? | Upstream workload admission, subject establishment, producer admission, authorization, binding and governed execution, and evidence. | Decision 4 |
| What authority does a BASIS administrative login create? | A BASIS administrative context only. It creates no OT operation-initiation, execution, or subject authority. | Decision 5 |
| Can a normal operator log in to `basis-console` and, by that login, command an OT device? | No. | Decisions 5, 6 |
| Can a supervisory platform assert that a user is authorized and force BASIS to trust it? | No. | Decision 4; ADR-0018 Decisions 1, 3 |
| Can a supervisory platform convey identity and provenance that let BASIS establish the subject independently? | Yes. The mechanism is deferred. | Decision 4; ADR-0018 Decision 3 |
| Can `basis-console` or a BASIS-native simulator or test client bypass `basis-gateway`, authorization, binding, or execution governance? | No. | Decisions 6, 7; ADR-0020 Decision 6 |
| How do BASIS-native simulators and test clients fit? | As direct-path evaluation tools whose results cannot support dispatch. Any future real-operation capability must be an ordinary governed producer. | Decisions 6, 7 |
| Can BASIS and a supervisory platform share an enterprise IdP or SSO? | Yes. Relying-party contexts and authorization contexts stay independent. | Decision 8 |
| Is BASIS still responsible for authorization and governed security enforcement around OT operations? | Yes. | Decisions 1, 4 |
| Does the supervisory platform remain responsible for the normal operational experience? | Yes. | Decisions 1, 2 |
| What remains mechanism-neutral? | Intake transport, subject-credential conveyance, federation and token mechanisms, and UI and API design. | Decision 10 |

## Relationship to Existing Decisions

This ADR composes accepted decisions. It invalidates none, modifies no accepted ADR's body, and changes no ADR's status.

- **ADR-0008 / ADR-0009.** Unchanged. The producer-versus-subject distinction is extended to say that a BASIS administrative session is not subject context either.
- **ADR-0010.** Unchanged. Its line "`basis-console` owns the operator experience" restates existing component ownership "without change." Read with `basis-console.md`, which excludes device management, it means the experience of operating and administering BASIS. Decision 6 states that reading explicitly. It is a clarification, not an amendment.
- **ADR-0011 through ADR-0016.** Unchanged. No new dispatch origin is created. The binding and no-dispatch-before-permission rules apply to every origin, including any future BASIS-native one (Decision 7).
- **ADR-0017.** Unchanged.
- **ADR-0018.** Unchanged and built upon. Decisions 2 through 4 here make the human origin of supervisory intent explicit and restate the three-identity separation. Decision 1's ownership split is restated from the BASIS side.
- **ADR-0019 (Proposed).** Not relied on. Terminology follows current canonical usage.
- **ADR-0020.** Unchanged. Decision 6 here relies on its Decision 6, under which a direct-path disposition, including the console's, cannot support dispatch.
- **ADR-0021.** Unchanged. Context, including category-authorized context, never establishes the subject.
- **ADR-0022.** Unchanged. Any future BASIS-native producer would hold its own logical workload identity under ADR-0022's rules.
- **`basis-console.md`, `basis-identity.md`, threat model.** Consistent. Narrow clarifying additions accompany this proposal. No existing invariant is removed or weakened.

## Alternatives Considered

**Leave the boundary implicit.** Rejected. Each needed fact is derivable from accepted documents, but only by composing several of them against component prose that, read alone, suggests the opposite. The cost of a wrong reading is a second, privileged origin of OT intent. That cost is high enough to justify a short explicit decision.

**Make `basis-console` an OT operator workstation alongside supervisory platforms.** Rejected. BASIS would absorb supervisory semantics that ADR-0018 leaves with the upstream platform. Its own interface would compete with the platforms it governs. The component that presents and administers authorization would also originate the operations it governs, which collapses the separation every boundary in the chain exists to keep. It would also contradict `basis-console.md`'s **Device Management** non-responsibility.

**Require subjects to authenticate interactively to BASIS before an upstream-initiated operation can be authorized.** Rejected. It would make every OT operation depend on a second interactive login, or on a BASIS session kept alive alongside the supervisory platform's. Neither is a security property: what matters is that BASIS establish the subject independently and verifiably, which ADR-0018 Decision 3 already requires. It would also push supervisory platforms toward caching or replaying BASIS sessions, which is worse than governed subject-credential conveyance.

**Let the supervisory platform's authentication stand as subject authentication for BASIS.** Rejected. It contradicts ADR-0018 Decision 3 ("Being authenticated as the upstream workload does not authenticate that subject") and would turn a compromised supervisory platform into an authority over every subject it serves.

**Prohibit shared enterprise SSO between BASIS and supervisory platforms.** Rejected. It adds operational friction without a security benefit. Shared authentication authority is safe so long as relying-party contexts and authorization stay separate, as Decision 8 requires.

**Prohibit any BASIS-native operation-originating tool, permanently.** Rejected. Reference producers, conformance harnesses, and constrained emergency procedures may later be justified. The durable risk is an implicit or privileged path, not the tool's existence. Decision 7 forbids the privileged path and requires any such tool to be an ordinary governed producer.

**Remove simulation and test submission from `basis-console`.** Rejected. Direct-path evaluation is already non-dispatchable (ADR-0020 Decision 6). Simulation is useful for policy verification and training, and removing it would buy no security.

## Consequences

### Positive

- A reader can determine, from one decision, that BASIS governs OT operations without originating them, and that a BASIS login carries no OT authority.
- Supervisory platforms integrate without requiring their users to hold BASIS sessions, and without any path by which they can assert authorization.
- The console's scope is anchored to BASIS administration, inspection, diagnostics, and labeled simulation. That gives future console work, including the operator-and-training roadmap, a fixed outer boundary.
- Any future BASIS-native operation-originating tool has a predetermined architectural shape: an ordinary governed producer.

### Negative / Tradeoff

- Current architecture does not provide or authorize a BASIS-native interface that combines OT operation and BASIS administration. An organization that wants a single screen today obtains it by composition, for example a supervisory platform that links to BASIS evidence. Any future proposal for such an interface would require its own architecture decision. It would remain subject to Decision 7: separate administrative, workload, and subject contexts; ordinary governed-producer status; full authorization-to-execution governance; and no privileged bypass.
- Until subject-credential conveyance is decided, an upstream-initiated operation cannot be authorized for a human subject in a conforming implementation. That gap already exists under ADR-0010 and ADR-0018. This ADR does not close it.
- Existing documents keep *operator* in the BASIS-administration sense. Readers must apply the terminology note in **Context**. Narrow clarifications in those documents reduce, but do not remove, that burden.

### Security Consequences

**Prevented by this architecture:**

- a BASIS administrative session used as an OT control path;
- an administrator acting as an authorization subject on OT operations by virtue of administration;
- a BASIS-native tool obtaining privileged admission, binding, or dispatch;
- a simulator's `ALLOW` being treated as execution permission;
- supervisory-platform authentication, or a shared IdP session, substituting for BASIS subject establishment (threat model §7.2, forged identity context; §7.5, privilege escalation).

**Still requiring follow-on architecture:** subject-credential conveyance; intake transport and integrity; a federation profile where BASIS accepts subject proof originating from a shared IdP; break-glass procedures; how simulation and diagnostic evidence is distinguished in presentation and records.

**Residual risks:**

- An administrator with policy authority can broaden what governed requests are permitted to do, so policy integrity remains critical (threat model §2.2, §7.3).
- A compromised supervisory platform can still originate admissible but unwanted requests for subjects it can legitimately convey. Those requests remain subject to full subject establishment and evaluation.
- A non-conforming console implementation could present a simulation result as live. The rule against that is architectural, and its enforcement is an implementation concern.

## Deferred Decisions

- **Subject-credential conveyance** from a supervisory platform to the operation-producer role (ADR-0010, ADR-0018).
- **Federation profile** for accepting subject proof that originates from an enterprise IdP shared with a supervisory platform.
- **Intake transport, envelope, and integrity mechanism** (ADR-0018).
- **Representation of simulation and diagnostic results** in console presentation and in evidence, so they remain distinguishable from governed operations.
- **Break-glass and emergency procedures** (`architecture-principles.md` §11), bound by Decision 7.
- **Any BASIS-native operation-originating capability**, which requires its own decision under Decision 7.
- **Policy-administration controls** such as review and separation of duties.

## Non-Goals

This ADR does not:

- modify any implementation repository, including `basis-console`, `basis-gateway`, `basis-identity`, `basis-producer`, `basis-core`, `basis-adapters`, or `basis-schemas`;
- remove or restrict existing console simulation or diagnostic functionality;
- define an endpoint, token, session, delegation, or federation mechanism;
- modify any accepted ADR's body or change any ADR's status;
- update any supervisory platform's repository;
- promote its working vocabulary to the glossary;
- change branding or terminology.

## Validation / Implementation Gate

Acceptance would establish the boundary as governed architecture against which console, identity, and integration work is reviewed. It authorizes no implementation. Implementation repositories may later reflect it in their own documentation through separate, bounded changes. A console change that adds any path toward the producer-intake boundary, the binding, or the protocol executor is not conforming unless a separate decision under Decision 7 authorizes it.

## References

- [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md): §2 (operation initiator), §3 (trust establishment), §7 (provenance)
- [`basis-console.md`](../architecture/basis-console.md): responsibilities, **Device Management** non-responsibility, design invariants
- [`basis-identity.md`](../architecture/basis-identity.md) and [`identity-authority-modes.md`](../architecture/identity-authority-modes.md): identity engine role; federated authority mode
- [`basis-gateway.md`](../architecture/basis-gateway.md): subject authentication and enforcement
- [`operator-and-training-experience.md`](../roadmaps/operator-and-training-experience.md): **Safety and Confirmation Design**
- [`architecture-principles.md`](../architecture-principles.md): §11 (human operators; break-glass)
- [`threat-model.md`](../security/threat-model.md): §2.2, §3.4, §4.1, §4.2, §6.4, §7.2, §7.3, §7.5
- [ADR-0008](0008-producer-workload-authentication-and-admission.md): producer vs. authorization subject
- [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md): trusted producer ingress
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md): operation-producer runtime; component ownership
- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md): protocol-executor role; Decision 9 (no dispatch before authoritative permission)
- [ADR-0012](0012-authorization-to-execution-binding.md): authorization-to-execution binding
- [ADR-0013](0013-execution-lifecycle-semantics.md) and [ADR-0014](0014-minimum-execution-evidence-semantics.md): execution lifecycle and evidence
- [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md): producer-intake boundary; three-identity separation
- [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Proposed): Basitra terminology
- [ADR-0020](0020-operation-to-authorization-mapping-and-composition-boundary.md): Decision 6 (direct path is non-dispatchable)
- [ADR-0021](0021-upstream-context-assertion-trust-boundary.md): category-scoped context trust; context never establishes subject
- [ADR-0022](0022-workload-credential-lifecycle-and-scope-boundary.md): workload credential lifecycle
- Ipotio architecture repository: upstream supervisory platform (cross-repository reference, not hyperlinked)
