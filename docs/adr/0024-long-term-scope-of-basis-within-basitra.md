# ADR-0024: Long-Term Scope of BASIS within Basitra

## Status

Accepted

Consistent with this repository's established practice (see ADR-0018's Context, and the Status sections of ADR-0019 and ADR-0023), this ADR was numbered when first proposed and submitted as `Proposed`, and its merging did not constitute acceptance. It has since undergone that separate, dedicated formal-acceptance review, and its status now records `Accepted`. Acceptance resolves open decision OD-8 in the [strategy document](../architecture/basitra-ecosystem-and-boundary-aware-security.md#13-open-decisions). It changes no part of the decision below, renames nothing, and performs none of the follow-on work listed under **Follow-On Work**, each item of which requires its own bounded PR. No canonical reference document changes because of this acceptance. The mismatch between this numbering practice and [`README.md`](README.md#numbering) remains open decision OD-5.

## Context

[ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Accepted) made *Basitra* the long-term identity of the project, community, and ecosystem. It named *Boundary-Aware Security* as the architectural security approach and re-expanded BASIS, forward-only, as *Boundary-Aware Secure Identity Service*. It deliberately did not decide what BASIS names inside Basitra. Its Decision 3 states that BASIS "may remain a component family, become a narrower identity and security subsystem, or take another role," and that the scope "is derived from the Basitra target architecture, not from current repository names." That question is OD-8. It blocks OD-9 (repository and component naming) and OD-10 (package and namespace conventions), because names cannot be evaluated against a responsibility that has not been fixed.

ADR-0019 Decision 7 sets the governing principle: *historical continuity informs migration but does not constrain the target architecture.* Current `basis-*` repository names are evidence of where first-party implementations live. They are not evidence of where architectural boundaries should be drawn.

The evidentiary input is [`basitra-target-architecture-discovery-assessment.md`](../architecture/basitra-target-architecture-discovery-assessment.md) (the "discovery assessment"). It is non-normative. It inventories 21 capabilities by responsibility (its §4), shows that accepted architecture defines roles and not repositories as the durable unit (§5), groups the capabilities into six analytical layers (§6), and analyzes four candidate shapes (§10). Its recommendation, Candidate C, makes the governed security path (its L3) the fixed center of BASIS. It leaves three adjacent layers explicitly undecided: interoperability contracts (L2), protocol integration (L4), and administration and observability (L5). This ADR uses that evidence. It does not adopt the recommendation as authority, and it decides the three placements the assessment left open.

Five facts from accepted architecture carry most of the weight:

- **Accepted ADRs already describe BASIS by responsibility.** [ADR-0023](0023-supervisory-platform-and-administrative-interface-boundary.md) Decision 1 states that "BASIS is an authorization and security substrate for OT operations. It is not a supervisory platform," and lists what it owns: workload admission on the governed path, subject establishment, evaluation, binding, governed dispatch, and authorization and execution evidence. Decision 4 lists what BASIS independently establishes. ADR-0023 states that it is "about architectural roles, not names."
- **Accepted ADRs place the producer, intake, and executor on the BASIS side.** [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md) Decision 2 states that the producer-intake boundary "belongs to the BASIS side," and its Relationship to ARCH-GAP-001 states that "the BASIS operation producer and protocol executor remain BASIS-side roles."
- **Accepted trust rules couple trust establishment, authorization, and operation and execution governance.** Three-identity separation, role-specific workload identity, category-scoped context trust, identifier ownership, fail-closed admission, preservation of the original operation, and credential-class separation each span at least two of those three areas (discovery assessment §9.7).
- **Roles are distinct from implementations.** ADR-0010 calls `basis-producer` "the Foundation-maintained implementation of the operation-producer role, not a claim that it is the only possible one." The glossary states that "a deployment may substitute a conforming alternative for a given role where the architecture permits one."
- **Adapter contracts and adapter implementations are already distinguished.** [`kernel-boundary-rules.md`](../kernel-boundary-rules.md#allowed-kernel-responsibilities) lists "adapter contracts: the interfaces that protocol adapters must implement … — not the adapter implementations" among the kernel's allowed responsibilities.

## Problem

What does BASIS name inside Basitra? The answer must be specific enough that:

- every existing role can be placed inside or outside BASIS for a stated reason;
- a role introduced later can be placed by a responsibility-based test rather than by its repository or its name;
- accepted statements about "BASIS" keep their meaning;
- OD-9 can evaluate every current repository against a stable boundary without another scope decision.

It must do this without renaming anything, changing any accepted separation, giving BASac meaning, or reopening the boundary with upstream supervisory platforms.

## Decision

**BASIS is the Boundary-Aware Security subsystem of Basitra that governs operations along the governed path. It consists of the architectural roles that own a security-critical responsibility for an operation as it crosses the boundaries of that path, from producer intake to the OT target: establishing identity and admission, determining what is evaluated, deciding and enforcing authorization, preserving and binding the operation, governing its dispatch, and observing and evidencing each crossing. BASIS membership attaches to roles, not to repositories, components, or first-party implementations. Basitra is the ecosystem that contains BASIS together with the capabilities that govern, publish for, package, consume, and extend it.**

```text
Basitra                          project · community · ecosystem identity (ADR-0019)
│
├── BASIS                        the governed-path security subsystem (this ADR)
│   ├── trust establishment              identity federation · subject establishment ·
│   │                                    upstream and producer workload admission
│   ├── authorization                    protocol normalization (adapter role) · composition ·
│   │                                    context admission · evaluation · enforcement
│   ├── operation and execution governance
│   │                                    producer intake · operation production · binding ·
│   │                                    governed dispatch · lifecycle observation
│   └── evidence                         an obligation of each owning role, not a separate role
│        + contract semantics            the meaning of what crosses each BASIS boundary
│        + BASIS-governed configuration  policy · mapping configuration · category trust
│                                        policy · admission grants
│
├── architecture and governance          Basitra level (decides BASIS, among other things)
├── contract publication                 Basitra level (publishes BASIS contract semantics neutrally)
├── administrative interfaces            Basitra level, consumers of BASIS through enforced interfaces
└── deployment and distribution          Basitra level (packages BASIS and non-BASIS components)

External: upstream supervisory platforms · enterprise identity providers ·
          OT targets · evidence consumers · BASAuth
```

The diagram shows architectural membership. It is not a dependency graph, a deployment topology, or a repository layout.

### 1. BASIS is an architectural subsystem

BASIS names an **architectural subsystem**: a set of roles defined by responsibility, held together by shared trust rules, and bounded by the governed path. It does not name:

- a repository prefix or a set of repositories;
- a distribution, package set, or release train (Decision 8);
- the Basis Foundation, the GitHub organization, or the project as a whole (that is Basitra);
- `basis-core` alone (ADR-0019 Decision 3);
- the Boundary-Aware Security approach itself (Decision 10);
- an access-control model or specification (Decision 13).

For this ADR, the **governed path** is the chain of separately verified boundaries defined by accepted architecture, not by membership in BASIS. An operation enters it at the producer-intake boundary (ADR-0018). It then passes operation production and producer admission (ADR-0008, ADR-0009, ADR-0010), subject establishment, composition, context admission, evaluation, and enforcement (ADR-0008, ADR-0020, ADR-0021), and authorization-to-execution binding (ADR-0012). It ends in governed dispatch to the OT target and lifecycle observation (ADR-0011, ADR-0013). Each crossing is evidenced (ADR-0007, ADR-0014). The diagrams in ADR-0018 and ADR-0023 Decision 1 are the reference. The direct, non-producer evaluation path at the enforcement role, whose dispositions cannot support dispatch (ADR-0020 Decision 6), is served by the same roles and adds none.

### 2. Membership rule

A role belongs to BASIS if, and only if, accepted architecture makes it the **owner** of at least one responsibility that meets all three conditions below.

1. **Path condition.** The responsibility is discharged at, or between, boundaries of the governed path while an operation crosses them. Alternatively, it establishes the identity, admission, or governed configuration that those boundaries apply at crossing time. A responsibility exercised only before an operation is originated, only outside runtime, or only on artifacts such as documents, published contracts, or packages does not meet this condition.
2. **Security condition.** If the role fails to discharge the responsibility, or discharges it wrongly, an operation can cross a boundary it should not cross, cross with a meaning other than the one authorized, cross under an identity or authority other than the one established, or cross without truthful, attributable evidence. Holding decision or dispatch authority is **not** required. Roles that preserve, observe, or evidence what crossed meet this condition.
3. **Ownership condition.** The role is the accountable owner of the responsibility, and no other role is assigned to establish it in its place. A role that only invokes, conveys, presents, displays, publishes, or packages a responsibility owned by another role does not meet this condition through that responsibility.

The rule is applied to roles within Basitra. External authorities that BASIS relies on remain external. Examples are an enterprise identity provider in federated mode and an upstream supervisory platform. BASIS's responsibility toward them is to verify what they supply, not to own what they do.

**Applying the rule to a new role.** An ADR that introduces a role states its placement under this rule, naming the owned responsibility and the condition each part of the test relies on. Examples of future roles the rule already places inside BASIS: an intake realization, a separated protocol executor, and protocol-specific executor implementations (ADR-0011 Decisions 5 and 6). Examples it places outside: a new administrative console, a schema-tooling project, a deployment tool, or a protocol simulator, unless the role also takes ownership of a responsibility that meets all three conditions.

**What the rule does not do.** Membership confers no authority, privilege, trust, or exemption. Every BASIS role keeps exactly the responsibilities and non-responsibilities its accepted decisions give it. Being inside BASIS does not let an adapter authorize, a producer admit itself, or an executor reinterpret a disposition. Being outside BASIS does not relax any obligation a capability already has. ADR-0023 Decision 7 ("BASIS may consume BASIS's security architecture. BASIS may not bypass it.") applies to every role and every consumer.

**Optionality is not a criterion.** A role may be optional in a deployment and still be inside BASIS, as the identity-engine role is. A role may be optional and outside BASIS, as the administrative interface is. Placement follows responsibility, not deployment frequency.

### 3. Roles inside BASIS

Each row names the responsibility that places the role inside BASIS. Evidence is listed as an obligation of the role that owns each crossing, not as a separate evidence component; no accepted architecture assigns evidence to a single component (ADR-0023 Decision 4 places it with "owning components").

| Area | Role | Owned responsibility that meets the rule | Governing decisions |
| - | - | - | - |
| Trust establishment | Identity-engine role (federation and canonical identity) | Establishes the canonical identity context subject establishment relies on. Never authorizes. Optional where the enforcement role verifies identity directly. | [`basis-identity.md`](../architecture/basis-identity.md); [`identity-authority-modes.md`](../architecture/identity-authority-modes.md) |
| | Subject establishment (enforcement role) | Decides which identity is evaluated as the subject, never from workload authentication | ADR-0008; ADR-0018 Decision 3; ADR-0023 Decisions 3–4 |
| | Workload admission (producer admission at the enforcement role; upstream workload admission at the producer-intake boundary) | Admits a role-specific logical workload at one boundary only | ADR-0008; ADR-0009; ADR-0018 Decision 2; ADR-0022 |
| Authorization | Protocol-adapter role | Determines the authorization semantics of a protocol operation by normalization, and constructs the evidence material and digest that attribute the decision to that operation. Holds no authority, credential, or network client. | ADR-0007; ADR-0010; ADR-0011 Decision 16; ADR-0020 Decision 5 |
| | Composition (enforcement role) | Composes the canonical action and resource evaluated, as sole composer on the governed admitted-producer path | ADR-0017; ADR-0020 |
| | Context admission (intake, producer, and enforcement roles, each for its part) | Decides which context reaches evaluation | ADR-0021 |
| | Authorization-kernel role | Sole source of the authorization outcome; decision evidence | [`kernel-boundary-rules.md`](../kernel-boundary-rules.md) |
| | Enforcement role | Enforces the disposition, fails closed, records authorization audit evidence. In the embedded model, the adapter host carries this role. | [`basis-gateway.md`](../architecture/basis-gateway.md); [`basis-adapters.md`](../architecture/basis-adapters.md) |
| Operation and execution governance | Producer-intake boundary (ingress of the operation-producer role) | Admits intake requests for production only; preserves provenance and correlation | ADR-0018 Decisions 2, 5–8 |
| | Operation-producer role | Produces the governed operation; holds the producer workload credential; conveys the subject credential independently; retains adapter evidence and mints references; creates the binding record | ADR-0007; ADR-0008; ADR-0010; ADR-0012 |
| | Protocol-executor role, including any protocol-specific dispatch implementation of it | Verifies the binding; dispatches only on a bound, permitting disposition; holds device credentials where required; sole truth-owning observer of execution-boundary facts | ADR-0011; ADR-0012; ADR-0013 Decision 14; ADR-0015; ADR-0016 |
| | Execution-evidence-producer role | Constructs and retains the execution-evidence record, separate from authorization evidence | ADR-0014 |

Two further things are inside BASIS without being roles of their own:

- **Contract semantics.** The meaning of every contract that crosses a BASIS boundary is part of BASIS. That includes the decision request and response, audit and evidence structures, action vocabulary, resource identifiers, normalized requests, identity context, and adapter interface contracts (Decision 4).
- **BASIS-governed configuration.** Policy, effective mapping configuration (ADR-0020), the category trust policy (ADR-0021), and producer and upstream admission grants (ADR-0008, ADR-0018, ADR-0022) are BASIS state. BASIS roles apply them at crossing time. Changes to them are adopted only through BASIS enforcement: authenticated, evaluated by policy, and evidenced (ADR-0023 Decision 5). The humans and change processes that exercise that authority, such as ADR-0020's deployment-designated "BASIS configuration authority," are principals acting through BASIS. They are not BASIS roles.

### 4. Interoperability contracts (L2): semantics inside BASIS, publication at the Basitra level

This ADR separates **contract semantics** from **contract publication authority** and places them differently.

- **Contract semantics belong to BASIS.** What a contract that crosses a BASIS boundary means, and what each party may rely on it for, is part of the subsystem whose boundaries it defines. A change to that meaning is a change to BASIS. It is decided through this repository's architecture process (an ADR where [`README.md`](README.md#when-an-adr-is-required) requires one), like any other change to BASIS.
- **Contract publication is a Basitra-level interoperability role, outside BASIS.** Publication means holding the machine-readable definitions, versions, and compatibility fixtures as the single, neutral source of truth. It acts on no operation and owns no crossing, so it fails the path and ownership conditions of Decision 2. Its purpose is neutrality: no BASIS role implementation, first-party or third-party, is the de facto authority over a contract it consumes ([`basis-schemas.md`](../architecture/basis-schemas.md) §2). A neutral reference outside the subsystem is what lets alternative implementations of BASIS roles be checked against the same definition.

```text
Architecture proposes.     Basitra governance decides BASIS contract semantics
Schemas publish.           Basitra-level publication, neutral to every implementation
Implementations consume.   implementations of BASIS roles, and BASIS consumers
```

The ownership model in [`basis-schemas.md`](../architecture/basis-schemas.md) §5 is preserved unchanged. This decision adds only which subsystem the proposed and published semantics belong to. It changes no contract, identifier, namespace, version, or compatibility rule. Contract identifiers remain a compatibility-governed surface that naming alone cannot change (OD-10). Adapter interface contracts remain an allowed kernel responsibility under [`kernel-boundary-rules.md`](../kernel-boundary-rules.md).

### 5. Protocol integration (L4): the protocol-adapter role is inside BASIS; its implementations are implementations of a BASIS role

**The protocol-adapter role is inside BASIS.** It meets all three conditions of Decision 2:

- **Path:** it acts at the protocol boundary, a boundary of the governed path, while the operation is produced.
- **Security:** wrong normalization makes an operation cross with a meaning other than the one authorized. On the governed producer path, adapter normalization is the only source of authorization semantics (ADR-0020 Decision 5), and "the authorization decision is only as good as the normalization in front of it" (strategy document §7). The evidence material and digest the role constructs are what bind the decision to the protocol operation and feed the binding (ADR-0007, ADR-0012).
- **Ownership:** normalization semantics and evidence canonicalization are "owned entirely by `basis-adapters`" in the role sense (ADR-0010 **Non-Responsibilities**; ADR-0007). No other role re-establishes them.

Placement inside BASIS changes none of the role's accepted limits. The protocol-adapter role:

- does not authorize;
- does not hold producer, subject, or device credentials;
- performs no live protocol communication and no governed protocol execution because it normalizes (ADR-0011 Decision 16);
- does not submit to the enforcement role;
- remains a library that the operation-producer role, or an adapter host in the embedded model, invokes.

**Specific protocol adapters are implementations of a BASIS role.** The same rule applies as for every other role (Decision 7). A first-party adapter library is the Foundation-maintained implementation of the protocol-adapter role for its protocol families. A conforming third-party adapter implements the same BASIS role without becoming a Foundation-maintained component. Protocol adapters remain the first-listed community extension area (strategy document §11.3, P-1). That openness comes from the role-versus-implementation distinction, not from placing the role outside BASIS.

**Protocol knowledge alone does not place a capability inside BASIS.** A protocol-specific capability that fulfills neither the protocol-adapter role nor the protocol-executor role for a governed operation is outside BASIS. Examples are protocol test endpoints, simulators, discovery tooling, and telemetry parsing. Conversely, protocol-specific dispatch mechanics that implement the protocol-executor role are inside BASIS as part of that role. ADR-0011 Decision 6 leaves their implementation taxonomy open, and this ADR does not decide it.

### 6. Administration and observability (L5): administrative interfaces are Basitra-level consumers of BASIS

**The administrative-interface role is outside BASIS.** It is a **Basitra-level consumer of BASIS**: a capability that uses BASIS responsibilities only through BASIS's enforced interfaces and owns none of them. It fails the ownership condition of Decision 2. Every responsibility it touches is owned and enforced by a BASIS role:

- a BASIS administrative context is established by BASIS enforcement;
- administrative actions are authenticated, evaluated by policy, and evidenced by BASIS roles;
- changes to governed configuration take effect only when BASIS adopts them.

It also fails the path condition: it acts on administrative actions, not on governed operations. This applies to `basis-console` and to any other administrative, inspection, diagnostic, or labeled-simulation interface.

ADR-0023 is preserved in full:

- interactive administrative login creates a BASIS administrative context only, and no authorization-subject standing on any OT operation (Decision 5);
- an administrative interface originates no OT operation and has no path to producer intake, the binding, or the protocol executor (Decisions 6 and 7);
- it does not bypass enforcement, and simulation and direct-path results cannot support dispatch (Decision 6; ADR-0020 Decision 6);
- the console is optional in every deployment ([`basis-console.md`](../architecture/basis-console.md) Design Invariant 10).

Placing the interface outside BASIS strengthens these rules. It makes explicit that an administrative surface stands in the same relationship to BASIS as any other consumer.

**The currently unassigned administrative surfaces split the same way.** The authority over effective mapping configuration (ADR-0020) and over the category trust policy (ADR-0021) is BASIS-governed configuration, and its adoption is a BASIS enforcement responsibility (Decision 3). Any surface, API client, or workflow tool through which an administrator or change process proposes, reviews, or submits that configuration is an administrative interface, and so a Basitra-level consumer. This ADR assigns neither the authority nor the surface to a component, and selects no storage, approval workflow, or API.

**Observability follows the same rule.** Producing authorization and execution evidence, and the authorization audit event, is a BASIS obligation of the owning roles. Inspecting, querying, analyzing, or presenting that evidence is consumption, whether by an administrative interface or by an external evidence consumer. Telemetry ingestion remains unrelated to Boundary-Aware Security as defined (strategy document §6).

### 7. BASIS membership attaches to roles, not to implementations

BASIS is defined by roles. Three relationships stay distinct:

| Relationship | Meaning | What it does not imply |
| - | - | - |
| **Architectural conformance** | An implementation fulfills a BASIS role's accepted responsibilities and non-responsibilities | Foundation maintenance, endorsement, certification, or any right to BASIS or Basitra names |
| **Ecosystem participation** | A person, project, or organization works within or identifies with the Basitra ecosystem | That their work implements any BASIS role, or that it is first-party |
| **First-party implementation** | The Basis Foundation maintains the implementation as part of its open-source work | That the implementation is the only conforming one, or that it receives any trust beyond its role |

Consequences:

- **A conforming third-party implementation of a BASIS role implements part of the BASIS architecture. It does not become a Foundation-maintained component.** This applies to adapters, producers, executors, intake realizations, enforcement, identity, and kernel implementations, where their governing decisions permit substitution.
- **A first-party component is not BASIS because it is first-party.** A Foundation-maintained administrative interface, contract-publication repository, or deployment tool is a Basitra component outside BASIS.
- **A component may host several roles.** It is placed role by role. For the first bounded execution slice, the `basis-producer` process hosts the operation-producer, protocol-executor, and execution-evidence-producer roles (ADR-0011 Decision 3; ADR-0014; ADR-0015), and the producer-intake boundary is the producer role's ingress (ADR-0018). All four are BASIS roles. A component that hosted both BASIS and non-BASIS roles would be described accordingly; that description is an OD-9 concern.
- **No trust follows from being first-party or from being BASIS** (ADR-0023 Decision 7). Each implementation is admitted, authenticated, and constrained by its role's rules.

This ADR defines no conformance criteria, certification, or rules for how third-party work may describe its relationship to BASIS or Basitra. Those are future work (strategy document §12.7; [`compatibility-philosophy.md`](../architecture/compatibility-philosophy.md)).

### 8. BASIS and the current distribution terminology

*BASIS Core Services Distribution* predates the Basitra target architecture. Under this decision:

- **It remains current canonical terminology with its existing meaning**: the open-source, deployable set of components maintained under Foundation governance ([`basis-ecosystem.md`](../architecture/basis-ecosystem.md#basis-core-services-distribution); [glossary](../glossary.md#basis-core-services-distribution)). This ADR does not redefine it.
- **It does not correspond to BASIS.** The distribution is a stewardship and packaging concept. BASIS is an architectural subsystem. The distribution contains Foundation-maintained implementations of BASIS roles, and it also contains Basitra-level components outside BASIS: contract publication, the administrative interface, and planned deployment tooling. A conforming third-party implementation of a BASIS role is inside BASIS architecturally and outside the distribution. Distribution membership and BASIS membership are therefore separate questions.
- **Its name is transitional.** Because it begins with "BASIS," the name now suggests that everything in the distribution is BASIS, which this decision makes inaccurate. Whether the distribution keeps, changes, or drops that name is naming work for OD-9 and OD-10. This ADR chooses no distribution name.

The same reading applies to related current terms. "The BASIS ecosystem" (strategy document C-5), "the BASIS namespace," and the `basis-*` component-name convention are current naming. They keep their meaning until later naming work changes them, and they do not define BASIS membership.

### 9. Meaning of "Identity" in *Boundary-Aware Secure Identity Service*

The expansion adopted by ADR-0019 Decision 3 is read **broadly**. *Identity* names the organizing principle of the subsystem, not a service type. BASIS establishes who is acting, through which workloads, and under whose authority, at the start of the governed path. It then keeps that identity-anchored security context distinct and attributable across every boundary the operation crosses:

- trust establishment produces the identities;
- authorization decides on the operation in the context of the established subject, never of a network position or a carrying workload (Principle 2; ADR-0008; ADR-0018 Decision 3);
- operation and execution governance ensures that what is dispatched is exactly what was authorized for that subject, through the binding;
- evidence attributes every crossing to the identities and the decision involved.

The broad reading does not claim that every BASIS role is an identity function:

- policy evaluation is authorization, not an identity-provider function;
- protocol normalization is semantic translation;
- protocol dispatch has no identity function of its own.

Their place in BASIS comes from Decision 2, not from the word *Identity*. The expansion describes what holds the subsystem together. It does not describe each member.

*Service* is read the same way. BASIS is a runtime subsystem that provides the service of governing operations. Under this decision it no longer includes contract publication, governance, deployment tooling, or administrative interfaces, which are the members *Service* described worst (discovery assessment §11).

**Residual strain, recorded rather than resolved.** The protocol-adapter role is the BASIS role the expansion describes least well. Its function is semantic, not identity-related, although it supplies the attributed evidence the identity chain depends on. This is not a conflict that requires revisiting ADR-0019. The acronym has always been looser than its expansion: the historical expansion also covered adapters. ADR-0019 Decision 3 treats BASIS as a standalone acronym. If later evidence shows the expansion misleads readers in practice, that is a terminology question for its own decision, not a reason to change BASIS's scope.

### 10. BASIS and Boundary-Aware Security are not synonyms

```text
Boundary-Aware Security  =  the architectural security approach (ADR-0019 Decision 2; strategy document §5.1)
BASIS                    =  the Basitra subsystem that realizes that approach on the governed path
```

- **Boundary-Aware Security is broader than BASIS.** It is a way of describing, reviewing, and extending architecture, and it is not limited to one subsystem or domain (strategy document §5.1, §7). Capabilities outside BASIS take part in it without becoming BASIS:
  - contract publication defines what crosses boundaries;
  - deployment shapes the deployment boundary (ADR-0011 Decision 19);
  - administrative interfaces administer BASIS-governed configuration;
  - upstream supervisory platforms and enterprise identity providers submit into, or are verified at, BASIS boundaries.
- **BASIS does not realize the whole catalog.** Environmental boundaries remain in analysis or future work (strategy document §5.3). This decision does not extend BASIS's claims to cover them.
- **Participation does not decide membership.** A capability is inside BASIS only by Decision 2. The protocol-adapter role is inside because it owns a qualifying responsibility (Decision 5), not merely because the protocol boundary is in the Boundary-Aware Security catalog.

### 11. Upstream supervisory platforms remain outside BASIS and outside Basitra

```text
Ipotio   =  one example of an upstream supervisory / operational platform (external)
BASIS    =  the authorization and security subsystem
Basitra  =  project / community / ecosystem identity
```

This decision restates ADR-0018 and ADR-0023 and changes neither. Originating supervisory intent is not governing it, so an upstream supervisory platform owns no BASIS responsibility (Decision 2). It also owns no Basitra role. The following remain in force unchanged:

```text
authentication                   !=  authorization
Ipotio request                   !=  authorization
Ipotio operational eligibility   !=  BASIS authorization
BASIS ALLOW                      !=  dispatch
authorization                    !=  execution
```

Every invariant in ADR-0023 Decision 9 also remains in force. No platform receives special trust, type, or vocabulary (ADR-0018 Decision 9; ADR-0023 Decision 11), and BASIS remains integrable with any upstream supervisory system in the category. Defining BASIS as the governed-path subsystem keeps *authorization ≠ execution* intact. That separation is a lifecycle boundary inside BASIS, enforced by the binding (ADR-0011, ADR-0012). This ADR does not convert it into a boundary between subsystems, and does not merge the two.

### 12. Continuity with accepted ADRs

This ADR supersedes no ADR, modifies no accepted ADR's body, and changes no ADR's status. Because accepted ADRs were written with BASIS as the current component family, one reading rule applies to them:

- **Where an accepted ADR uses "BASIS" for responsibilities, roles, sides, or the substrate**, it means the subsystem defined here. Its meaning is unchanged, because every such use falls inside Decision 3.
- **Where an accepted ADR uses "BASIS" as a family, distribution, namespace, or naming label**, it denotes current naming (Decision 8). That usage remains accurate and is addressed, where needed, by OD-9 and OD-10.

| ADR | Effect |
| - | - |
| [ADR-0007](0007-adapter-evidence-construction.md) | Semantically unchanged. The adapter and producer roles it divides evidence work between are both inside BASIS. |
| [ADR-0008](0008-producer-workload-authentication-and-admission.md), [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) | Semantically unchanged. Producer admission and subject establishment are inside BASIS. ADR-0008 defines no BASIS-specific URI namespace, so no workload identity string depends on a component name. |
| [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) | Semantically unchanged. "BASIS operation-producer runtime" and "BASIS Producer" describe a Foundation-maintained implementation of a BASIS role, which is accurate. "A component of the BASIS Core Services Distribution" is a distribution-membership statement with its existing meaning (Decision 8). Its ownership list is unchanged; under this decision the components it assigns to the console, schemas, and deployment tooling host Basitra-level roles outside BASIS. ADR-0010 fixes the name `basis-producer`, so any rename requires a superseding ADR. That is an OD-9 concern. |
| [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) through [ADR-0016](0016-bounded-target-replay-freshness-posture.md) | Semantically unchanged. The executor, binding, lifecycle, evidence, first-target, and replay decisions are inside BASIS. ADR-0011 Decision 4 reserves no executor repository name, and nothing here reserves one. |
| [ADR-0017](0017-action-vocabulary-naming-structure.md) | Semantically unchanged. The vocabulary's semantics are BASIS contract semantics, published at the Basitra level (Decision 4). |
| [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md) | Semantically unchanged. "The BASIS side" is the subsystem defined here. Intake, the producer, and the executor are BASIS roles. Intake's repository placement remains deferred. |
| [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) | Fulfilled, not superseded. ADR-0019 Decision 3 delegated BASIS's long-term scope to this decision. Its Decision 6 description of BASIS as, "today, the component family" was current usage while this ADR was `Proposed`. With this ADR accepted, it reads as current naming in the sense of Decision 8. The expansion, the BAS abbreviation convention, BASac's prospective status, and the naming principle are unchanged. |
| [ADR-0020](0020-operation-to-authorization-mapping-and-composition-boundary.md), [ADR-0021](0021-upstream-context-assertion-trust-boundary.md) | Semantically unchanged. Composition, context admission, mapping configuration, and the category trust policy are BASIS responsibilities or BASIS-governed configuration. Their administrative surfaces are consumers (Decision 6). |
| [ADR-0022](0022-workload-credential-lifecycle-and-scope-boundary.md) | Semantically unchanged. Logical workload identities are role-specific. Inference: a later component rename that leaves role and scope unchanged would not by itself change a logical workload identity. A migration plan should confirm this. |
| [ADR-0023](0023-supervisory-platform-and-administrative-interface-boundary.md) | Semantically unchanged; interpretive continuity for one phrase family. Decision 1's "authorization and security substrate" is the subsystem defined here, and Decisions 1 and 4 list only BASIS responsibilities. "BASIS user interface," "BASIS administrative interface," and "BASIS-native tool" name interfaces and tools that administer or consume BASIS. Under this decision those interfaces are Basitra-level consumers, outside the subsystem (Decision 6). The phrases keep their meaning, and "BASIS" in them identifies what is administered, not subsystem membership. Decision 7 applies to them unchanged. A future BASIS-native capability architected as a governed producer under Decision 7 would fulfill the operation-producer role, a BASIS role, and would still receive no privilege. |

ADR-0001 through ADR-0006 remain `Proposed`. This decision does not depend on them. Kernel isolation is governed independently by [`kernel-boundary-rules.md`](../kernel-boundary-rules.md), and the authorization-kernel role is inside BASIS under any reading.

### 13. What this decision does not decide

This ADR does not:

- choose, reserve, or change any repository, component, distribution, package, or import-namespace name (OD-9, OD-10);
- rename or migrate the `basis-foundation` GitHub organization (OD-7), or decide the Basis Foundation's relationship to Basitra (OD-1);
- reconcile Ipotio's terminology (OD-3), or change anything in Ipotio;
- give BASac any meaning (ADR-0019 Decision 5; OD-4). The subsystem defined here is not an access-control model or specification;
- decide intake transport or repository placement, a separated executor's architecture, executor implementation taxonomy, or subject-credential conveyance;
- assign the mapping-configuration or category-trust-policy administrative surfaces to a component, or select their storage, workflow, or API;
- define conformance criteria, certification, extension APIs, or rules for third-party naming;
- change any contract, schema, identifier, or compatibility rule;
- correct documentation drift recorded in discovery assessment §4.3, finding 5;
- authorize any implementation or migration.

## Placement Summary

The table places every role and capability in the discovery assessment's inventory, so that OD-9 can evaluate each repository against a fixed boundary. The "current first-party implementation" column is evidence of where code lives today. It is not a naming decision.

| Role or capability | Placement | Reason | Current first-party implementation |
| - | - | - | - |
| Identity-engine role | BASIS | Decision 3 | `basis-identity` |
| Subject establishment, producer admission, composition, context admission, enforcement, authorization audit evidence | BASIS | Decision 3 | `basis-gateway` |
| Authorization-kernel role; adapter interface contracts | BASIS | Decision 3; Decision 4 | `basis-core` |
| Protocol-adapter role | BASIS | Decision 5 | `basis-adapters` |
| Producer-intake boundary; operation-producer role; protocol-executor role; execution-evidence-producer role | BASIS | Decision 3 | `basis-producer` (producer role implemented in its bounded scope; intake, executor, and execution evidence accepted, not implemented; intake placement deferred) |
| Contract semantics | BASIS | Decision 4 | Decided in `basis-architecture`; published in `basis-schemas` |
| Contract publication | Basitra level, outside BASIS | Decision 4 | `basis-schemas` |
| Administrative-interface role, including administrative surfaces for mapping configuration and category trust policy | Basitra level, consumer of BASIS | Decision 6 | `basis-console` (other surfaces unassigned) |
| Deployment and distribution tooling | Basitra level, outside BASIS | Fails Decision 2: acts on no operation; packages BASIS and non-BASIS components | None (`basis-deploy` not established) |
| Architecture and governance | Basitra level, outside BASIS | Fails Decision 2: no runtime responsibility; decides BASIS among other things | `basis-architecture`; [`GOVERNANCE.md`](../../GOVERNANCE.md) |
| Upstream supervisory intent; topology and operational state | External | Decision 11; ADR-0018; ADR-0023 | External (for example, Ipotio or Niagara) |
| Enterprise identity providers; OT targets; evidence consumers | External | Decision 2 (external authorities and consumers) | External |

Historical artifacts (`basis-poc`, `basis-foundation/basis`) and the uninitialized `basis-lab` host no role in the inventory. Their treatment is a naming and archival matter for OD-9.

## Boundary Questions

| Question | Answer | Basis |
| - | - | - |
| What does BASIS name? | The Boundary-Aware Security subsystem of Basitra that governs operations along the governed path, defined by roles | Decision 1 |
| How is a new role placed? | By the path, security, and ownership conditions, stated in the ADR that introduces the role | Decision 2 |
| Is execution governance inside BASIS? | Yes. *Authorization ≠ execution* is a lifecycle boundary inside BASIS, enforced by the binding. | Decisions 3, 11 |
| Are shared contracts BASIS? | Their semantics are. Their publication is a Basitra-level role outside BASIS. | Decision 4 |
| Are protocol adapters BASIS? | The protocol-adapter role is. Specific adapters are implementations of that role. | Decision 5 |
| Is a conforming third-party adapter, producer, or executor BASIS? | It implements a BASIS role. It is not a Foundation-maintained component. | Decisions 5, 7 |
| Is `basis-console` BASIS? | No. It is a Basitra-level consumer of BASIS. | Decision 6 |
| Is the BASIS Core Services Distribution the same as BASIS? | No. It is a broader, Foundation-maintained distribution, and its name is transitional. | Decision 8 |
| Does BASIS membership confer authority or trust? | No | Decision 2; ADR-0023 Decision 7 |
| Is BASIS the same as Boundary-Aware Security? | No | Decision 10 |
| Does this change Ipotio's relationship to BASIS? | No | Decision 11 |
| Does this give BASac meaning? | No | Decision 13 |
| Does this choose any name? | No. That is OD-9 and OD-10. | Decision 13 |

## Alternatives Considered

**Candidate A: BASIS remains the broad component family.** BASIS would continue to name every first-party component, including contract publication, administrative interfaces, and deployment tooling. Rejected. It has the lowest compatibility cost, but its membership is defined by which first-party repositories exist, which ADR-0019 Decision 7 cautions against. It gives no test for placing a future role other than "the Foundation built it." It cannot place a conforming third-party implementation. It also groups runtime security roles with non-runtime support under a *Service* expansion that describes the latter poorly (discovery assessment §11). Everything A gets right about the governed-path core is kept by this decision.

**Candidate B: BASIS narrows to identity and the authorization decision, with operation and execution governance directly under Basitra.** B is structurally possible. It does not split binding creation from binding verification: the producer, binding creation and verification, governed dispatch, lifecycle observation, and execution evidence would all sit together outside BASIS. *Authorization ≠ execution* already makes the disposition a defined handoff, and B would raise that lifecycle boundary into a subsystem boundary placed immediately after the authorization decision. Rejected, for four reasons:

- the authoritative disposition would cross a subsystem boundary to reach the roles that bind and enforce it, and the governed path would cross that boundary twice (into BASIS at producer submission, and out with the disposition);
- several accepted trust rules span the boundary B would draw (credential-class separation, preservation of the original operation, identifier ownership, context trust);
- accepted text assigns binding and governed dispatch to BASIS (ADR-0023 Decision 1) and places intake, the producer, and the executor on "the BASIS side" (ADR-0018). Under B, those statements would describe a substrate larger than BASIS, and a terminology statement or supersession would be needed;
- three of the accepted Boundary-Aware Security boundaries (intake, authorization-to-execution, execution evidence) would sit outside the subsystem that carries the name.

**Candidate C: BASIS names the governed security path.** Selected, with refinements the discovery assessment left open:

- the responsibility rule is made precise and operational (Decision 2);
- the protocol-adapter role is placed inside BASIS on that rule (Decision 5);
- contract semantics are separated from publication (Decision 4);
- administrative interfaces are placed as consumers (Decision 6);
- roles are fixed as the unit of membership (Decision 7).

**Candidate D: BASIS names the first-party reference distribution.** BASIS would become a stewardship and packaging label, with subsystems named directly under Basitra. Rejected. It answers who maintains what, not what the architecture is. Accepted ADRs use "BASIS" for roles any conforming implementation may fill (ADR-0018, ADR-0023), so D would require interpreting those uses throughout. Under D, a conforming third-party producer could never be BASIS, even while it fulfills ADR-0018's "BASIS-side" role. A distribution is also not a "service." Decision 8 keeps a distribution concept alongside the subsystem without making BASIS that concept.

**BASIS as the identity engine only.** The literal "Secure Identity Service" fits the identity-engine role. Rejected. That role is optional on the governed path, so the name of the security substrate would attach to a component that a conforming deployment may omit. Every accepted statement about what BASIS owns would then describe something else.

**L2: all contracts, semantics and publication, inside BASIS.** Rejected. Publication acts on no operation and owns no crossing. Placing it inside BASIS would also put the neutral reference against which BASIS role implementations are checked inside the subsystem those implementations make up. That conflicts with the neutrality that is publication's purpose.

**L2: all contracts at the Basitra level.** Rejected. The meaning of what crosses a BASIS boundary would then belong to no subsystem, and a change to a decision request's or audit event's meaning, which is a change to BASIS's behavior, would not be a change to BASIS.

**L4: only the adapter contract inside BASIS; adapter implementations as Basitra extensions outside it.** Considered seriously, because the contract-versus-implementation distinction is real ([`kernel-boundary-rules.md`](../kernel-boundary-rules.md)). Rejected as the placement rule, for three reasons:

- ADR-0007 assigns evidence construction and digest computation to the adapter role itself, not to a contract;
- the decision depends on the role's normalization in practice, not only on its interface;
- separating "role" from "implementation" for adapters alone would treat adapters differently from producers and executors, which also have conforming third-party implementations. Decision 7 already provides the contract-versus-implementation distinction for every role uniformly.

**L4: the protocol-adapter role at the Basitra integration edge.** Rejected. Owning the semantics that authorization receives meets every condition of Decision 2, so excluding it would require an exception to the rule for one role. In the embedded model, the adapter host also carries enforcement, so the boundary would have to shift with topology.

**L5: administrative interfaces inside BASIS.** Rejected. An administrative interface owns no governed-path responsibility, and every responsibility it touches is owned by a BASIS role. Placing it inside would suggest that subsystem membership gives an administrative surface standing, which ADR-0023 Decisions 5 to 7 exist to deny.

**L5: a separate "administrative plane" subsystem.** Rejected as unnecessary. The authority being administered is already BASIS-governed configuration, and the interfaces add no responsibility that needs its own subsystem.

**Defer L2, L4, and L5 again.** Rejected. OD-9 cannot evaluate `basis-schemas`, `basis-adapters`, or `basis-console` without these placements, and no separate architecture question blocks them.

## Consequences

### Positive

- OD-8 is resolved. With this ADR accepted, OD-9 can begin: every current repository can be evaluated against a stable, responsibility-based boundary (see **Placement Summary**).
- A future role can be placed by a stated test, with no default inclusion and no circular reasoning.
- Accepted statements about what BASIS owns keep their meaning. No supersession is needed for OD-8.
- Conforming third-party work has a clear architectural position: it implements BASIS roles, separate from Foundation maintenance.
- Shared contracts, protocol adapters, and administrative interfaces each have an explicit placement and reason.
- *Boundary-Aware Secure Identity Service* has a stated reading that matches the chosen subsystem.

### Negative / Tradeoff

- **"BASIS" now has two current uses**: the subsystem defined here, and the existing naming in "BASIS Core Services Distribution," "BASIS ecosystem," "BASIS namespace," and `basis-*`. Readers must apply the reading rule in Decision 12 until OD-9 and OD-10 reconcile names.
- **Current-state documents are now partly inaccurate, because this ADR is accepted.** Examples: [`writing-guidelines.md`](../standards/writing-guidelines.md) §4.1 ("BASIS refers to the open-source core services distribution"), the BASIS glossary and terminology-rules entries, and the strategy document's §3.1 and §8 descriptions of BASIS as the component family. Each needs a bounded follow-on PR. None is changed by this ADR or its acceptance.
- **Some current repository names may no longer answer the Stage 4 naming question well.** For example, a `basis-*` name on a component that is now outside BASIS, or a single-role name on a component hosting several BASIS roles. This ADR records the tension and decides nothing about it.
- **The protocol-adapter role is the BASIS role the expansion describes least well** (Decision 9).

### Security Consequences

- The decision moves no authority, credential, admission, or trust. It classifies existing responsibilities.
- It closes a reading that could have mattered: that being "part of BASIS" gives a component standing on the governed path. Membership confers nothing (Decision 2), and the administrative interface is explicitly a consumer (Decision 6).
- It keeps the security-critical semantic edge, protocol normalization, under the same subsystem rules as the decision it feeds.

## Follow-On Work

Each item needs its own bounded PR. Item 1, the formal acceptance review that moved this ADR from `Proposed` to `Accepted`, is complete. Items 2 onward did not begin while this ADR was `Proposed`; with it accepted, each is now eligible to begin in its own bounded PR:

1. **Formal acceptance review** of this ADR. Complete.
2. **OD-9: repository and component naming.** Apply the Stage 4 question to each repository using the **Placement Summary**, including whether ADR-0010's name requires supersession and how the distribution term (Decision 8) is treated.
3. **OD-10: package, distribution, and import-namespace conventions**, after OD-9 and in coordination with OD-1 and OD-7.
4. **Current-documentation reconciliation**, limited to current-state text:
   - [`writing-guidelines.md`](../standards/writing-guidelines.md) §4.1;
   - the glossary's BASIS and distribution entries;
   - [`terminology-rules.md`](../standards/terminology-rules.md) (with strategy document C-5 and C-7);
   - [`basis-ecosystem.md`](../architecture/basis-ecosystem.md);
   - the strategy document's §3.1 and §8.
5. **OD-3: Ipotio terminology reconciliation**, in Ipotio's repository, after OD-9.

## Non-Goals

This ADR does not:

- rename, create, or retire any repository, package, import namespace, organization, foundation, or website;
- modify any implementation repository, schema, contract, or compatibility surface;
- modify any accepted ADR's body or change any ADR's status;
- update canonical reference documents (glossary, writing guidelines, terminology rules, `basis-ecosystem.md`);
- update Ipotio or any other repository;
- define BASac;
- implement or authorize execution or any other runtime behavior.

## References

- [`basitra-target-architecture-discovery-assessment.md`](../architecture/basitra-target-architecture-discovery-assessment.md): §4 (inventory), §5 (roles versus implementations), §6 (layers), §9 (cohesion and seams), §10 (candidates), §11 (the expansion), §12 (ADR constraints), §13 (questions)
- [`basitra-ecosystem-and-boundary-aware-security.md`](../architecture/basitra-ecosystem-and-boundary-aware-security.md): §5.1, §5.3, §6, §7, §11.3, §12.4, §13
- [`basis-ecosystem.md`](../architecture/basis-ecosystem.md), [`reference-vs-implementation.md`](../architecture/reference-vs-implementation.md), [`compatibility-philosophy.md`](../architecture/compatibility-philosophy.md)
- [`basis-schemas.md`](../architecture/basis-schemas.md) §2, §5; [`ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md)
- [`basis-adapters.md`](../architecture/basis-adapters.md), [`basis-console.md`](../architecture/basis-console.md), [`basis-gateway.md`](../architecture/basis-gateway.md), [`basis-identity.md`](../architecture/basis-identity.md), [`identity-authority-modes.md`](../architecture/identity-authority-modes.md)
- [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md)
- [`kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`architecture-principles.md`](../architecture-principles.md), [`glossary.md`](../glossary.md), [`writing-guidelines.md`](../standards/writing-guidelines.md), [`terminology-rules.md`](../standards/terminology-rules.md)
- [ADR-0007](0007-adapter-evidence-construction.md), [ADR-0008](0008-producer-workload-authentication-and-admission.md), [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md), [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md), [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](0012-authorization-to-execution-binding.md), [ADR-0013](0013-execution-lifecycle-semantics.md), [ADR-0014](0014-minimum-execution-evidence-semantics.md), [ADR-0015](0015-first-bounded-execution-target.md), [ADR-0016](0016-bounded-target-replay-freshness-posture.md), [ADR-0017](0017-action-vocabulary-naming-structure.md), [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md), [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md), [ADR-0020](0020-operation-to-authorization-mapping-and-composition-boundary.md), [ADR-0021](0021-upstream-context-assertion-trust-boundary.md), [ADR-0022](0022-workload-credential-lifecycle-and-scope-boundary.md), [ADR-0023](0023-supervisory-platform-and-administrative-interface-boundary.md)
- [`GOVERNANCE.md`](../../GOVERNANCE.md)
- Ipotio architecture repository: upstream supervisory platform (cross-repository reference, not hyperlinked)
