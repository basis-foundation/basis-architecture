# Identity-Aware OT Architecture

The previous section concluded that authorization in operational technology environments is increasingly a system-level concern rather than a protocol-level one. This section gives that claim architectural substance. It describes a generalized model for identity-aware authorization — one that is not specific to any product, protocol stack, or deployment topology — and explains the reasoning behind each structural element.

The architecture is organized around a diagram that appears in the preceding section of this paper (`diagrams/identity-aware-authorization-flow.mmd`). References to specific components and flows in that diagram appear throughout this section. The diagram should be read alongside the prose; the two are designed to reinforce each other, not to duplicate each other.

---

## Architectural Overview

The architecture model separates three functions that protocol-centric security conflates into a single mechanism — network membership — that performs all of them coarsely and simultaneously.

**Identity verification** is the determination that a subject is who or what it claims to be. In a network-centric model, this function is implicit: a device or user on the trusted network segment is presumed to be a legitimate member of that segment. In an identity-aware model, verification is explicit: subjects present verifiable credentials, and those credentials are validated by a dedicated identity layer before any authorization decision is made.

**Policy evaluation** is the determination of whether a verified subject is permitted to perform a requested action on a specific resource. In a network-centric model, this function collapses into network membership: permitted traffic is whatever a segment member can send. In an identity-aware model, evaluation is explicit: a policy engine receives a structured request — subject, resource, action, context — and applies defined policy to produce a decision.

**Policy enforcement** is the application of a policy decision to operational traffic. Enforcement is necessarily distributed: it must occur at the points where operational traffic crosses trust boundaries, not at the location of the policy engine. In a network-centric model, enforcement is performed by firewalls and VLAN configurations at network boundaries. In an identity-aware model, it is performed by enforcement points positioned at trust boundary crossings, applying decisions produced by the policy engine.

The diagram depicts these three functions across four trust zones. The Central Services Zone contains the identity verification and policy evaluation infrastructure. The Operations Zone and Edge/Gateway Zone contain enforcement points. The Field Device Zone is downstream of enforcement — traffic that reaches it has already been authorized at the edge zone boundary.

---

## Architecture Model and Implementation Scope

The architecture described in this section is a full system model. It defines the complete set of components, zones, and flows required for identity-aware authorization in OT environments: the identity provider, the policy engine, the enforcement points, the protocol adapters, the local policy cache, and the audit pipeline — each in its defined position relative to the others.

This model is intentionally independent of any specific implementation. It describes what must exist and how those things must relate. It does not prescribe which software component implements which role, or how the roles are packaged and distributed.

Within this architecture, the authorization kernel — the component responsible for policy evaluation semantics, enforcement contracts, and audit event definitions — represents the stable, isolated core that all other components depend on. In the BASIS ecosystem, this role corresponds to basis-core.

**What basis-core owns within this architecture:**

- The normalized decision semantics: the canonical meaning of `allow`, `deny`, and `not applicable` decisions and the conditions under which each applies
- The policy evaluation behavior: the logic that takes subject identity, resource descriptor, action, and applicable policy as inputs and produces a deterministic authorization decision
- The enforcement semantics: the contracts that define what enforcement points must evaluate and how they must behave when evaluation cannot complete
- The failure mode contracts: the defined behavior for conditions where evaluation cannot proceed normally
- The audit event contracts: the canonical schema for authorization decision events, including required fields, their types, and the conditions under which events must be emitted

**What basis-core does not own:**

- Identity provider operation — authenticating subjects and issuing credentials is the responsibility of the identity layer above the kernel
- Protocol adapter implementation — normalizing BACnet, Modbus, MQTT, or other field-protocol messages is the responsibility of the adapter layer (basis-adapters in the ecosystem)
- Runtime API hosting — receiving authorization requests over a network, managing sessions, and returning decisions to callers is the responsibility of the gateway layer (basis-gateway)
- User interfaces — operator consoles, administrative dashboards, and policy review tools belong in a separate console component
- Telemetry pipelines — OT telemetry collection and routing are operational data concerns that sit outside the authorization kernel
- Deployment and cloud infrastructure — basis-core must not depend on cloud platform SDKs, specific database runtimes, deployment orchestrators, or cloud-specific service integrations

These boundaries are not implementation details. They are architectural constraints. A kernel that imports from protocol stacks, cloud SDKs, or UI frameworks cannot be deployed portably in the range of OT environments the architecture is designed to serve. A kernel with narrow, stable dependencies can be tested independently, versioned conservatively, and deployed in environments where the surrounding infrastructure changes frequently.

Higher-level services — the gateway, the console, the adapter implementations, the deployment tooling — sit above the kernel and depend on it. They provide the operational surface that makes the architecture usable. The kernel provides the stable evaluation semantics that they all share.

---

## Authorization as a System-Level Concern

The core architectural claim of this paper is that authorization cannot be effectively managed as a local property of individual protocols and devices when the environment in which those protocols and devices operate has become interconnected, remotely accessible, and subject to audit and accountability requirements.

A system where authorization logic is embedded independently in each component — each controller managing its own access levels, each system maintaining its own credentials, each vendor implementing its own trust model — will produce the outcomes described in the preceding section: inconsistent policy, incomplete audit records, credential management overhead that scales poorly, and an inability to reason centrally about who has access to what.

Treating authorization as a system-level concern means building dedicated infrastructure for it: a shared identity layer, a shared policy evaluation layer, shared enforcement points at trust boundaries, and a shared audit pipeline. These components are not features added to existing OT systems — they are a separate architectural layer that operates alongside and above the existing field infrastructure, without requiring that field infrastructure to change.

The diagram makes this layering visible. The field device zone at the bottom of the diagram contains BAS controllers and field hardware that are unchanged by the authorization architecture. The edge zone above them contains protocol adapters and enforcement infrastructure that mediate access to those devices. The central services zone contains the identity, policy, and audit infrastructure that the enforcement layer depends on. None of these zones eliminate or replace each other — they layer.

This layering is also what makes incremental adoption possible. An environment that deploys enforcement at the operations-zone boundary — the jump host — without yet deploying enforcement at the edge zone has improved its authorization posture for remote access without requiring any change to field devices or to the protocol adapters. Each layer added improves the coverage and granularity of the authorization model without requiring that the full model be in place before any value is delivered.

---

## Subjects, Resources, and Actions

Every authorization request is defined by three primitives. These definitions are canonical to the authorization model and must be used consistently throughout the deployment.

A **subject** is the entity whose request is being authorized: an operator, a technician, a vendor service account, a device acting on behalf of another device, or an automated process consuming OT data. The subject must present verifiable credentials — credentials that the identity layer can authenticate and that carry the attributes required for policy evaluation. In the diagram, the human subjects are the operator and technician workstations in the Operations Zone; their identity is established at the jump host before any access to the OT environment is permitted.

A **resource** is the target of the request: a BAS controller point, a device configuration interface, a telemetry data stream, a configuration parameter, or any other system object to which access is controlled. Resources must be described with structured, consistent identifiers — the resource descriptor must be the same regardless of the protocol or path through which the request arrived. A setpoint on a BACnet controller and a setpoint on a Modbus controller are different resources with different descriptors, but the descriptor format must be consistent so that a single policy can express rules about both.

An **action** is the specific operation the subject is requesting on the resource: read a sensor value, write a setpoint, change a configuration parameter, issue an override command. The granularity of action definition directly determines the expressiveness of the policy model. A model that distinguishes only between read and write cannot express that an operator is permitted to write setpoints within a defined range but not to write controller configuration. The action vocabulary should be designed to match the operational distinctions that the policy model needs to express.

These three primitives — subject, resource, action — form the authorization request that the policy engine evaluates. Everything else in the architecture exists to produce this request accurately and to enforce the resulting decision reliably.

---

## Identity Propagation Across Trust Boundaries

When an operator command passes through multiple system components — from the workstation through the jump host, across the OT gateway, through the local enforcement point, to the protocol adapter, and finally to the controller — the identity of the operator must remain attributable at each step. Without identity propagation, enforcement points downstream of the initial authentication event can only observe the identity of the immediate upstream component, not the identity of the human who initiated the request.

In the diagram, identity propagation begins at the jump host. The identity provider validates the operator's credentials and returns verified identity context — a structured, integrity-protected representation of the authenticated subject's attributes. This identity context is carried forward into the session that the jump host establishes into the edge zone. The local enforcement point, when it evaluates a command arriving from that session, uses the identity context to construct the authorization request it submits to the policy engine. The policy engine receives the original operator's identity, not the identity of the gateway that routed the session.

This propagation is what makes end-to-end attribution possible. An audit event generated by the local enforcement point records the identity of the operator whose session produced the command — not the identity of the gateway that forwarded it. Without propagation, the audit record at the edge zone boundary would show only that an authenticated gateway submitted a command; it would not show which operator's action produced that command.

The mechanism for propagating identity depends on what the intermediate components support. Where modern, authenticated protocols are in use between components, identity context can be carried in signed tokens. Where legacy protocols or components cannot carry signed tokens, the gateway or adapter that receives the authenticated session is responsible for attaching the identity context it received at authentication time. Intermediaries must not substitute their own identity for the original subject's; doing so breaks the propagation chain in a way that the audit record cannot detect.

Identity propagation is not achievable for all paths in all environments. A legacy device that initiates communication autonomously — without having authenticated to any subject — cannot propagate operator identity because no operator initiated the request. Device-originated communications require device identity rather than operator identity: a verifiable credential that identifies the device, not a human subject. Device identity and operator identity are distinct concepts in this architecture, and the policy model must be able to reason about both.

---

## Policy Evaluation and Enforcement

The policy engine is the component responsible for evaluating authorization requests against defined policy and producing authorization decisions. It is, along with the identity provider, the core of the Central Services Zone in the diagram.

The policy engine receives a structured request — subject identity and attributes, resource descriptor, requested action, and relevant contextual information — and applies the applicable policies to determine the outcome. The outcome is an authorization decision: `allow`, `deny`, or `not applicable` when no policy covers the request. The decision is returned to the enforcement point that submitted the request.

Several properties of policy evaluation are architecturally significant.

**Determinism.** Given the same inputs, the policy engine must always produce the same output. Non-deterministic evaluation produces unpredictable authorization outcomes, which operators cannot reason about and auditors cannot interpret. Determinism requires that policies be versioned, that the policy version in effect at evaluation time be recorded in the decision, and that evaluation logic not incorporate non-deterministic state.

**Independence.** The policy engine evaluates requests; it does not carry operational traffic. In the diagram, there are no solid data-plane arrows into the Central Services Zone — only dashed control-plane arrows. An operator command does not pass through the policy engine. The enforcement point at the edge zone boundary submits a structured representation of the command to the policy engine, receives a decision, and then — if the decision is `allow` — permits the command to proceed to the protocol adapter and controller. The policy engine's role is evaluative, not intermediary.

**Shared policy.** Because policy evaluation is performed by a single shared component rather than independently by each enforcement point, the same policy applies consistently at every boundary. A policy change takes effect everywhere simultaneously; no enforcement point can diverge from the current policy state except temporarily, during the window between policy updates.

Enforcement points are the components that apply policy decisions to operational traffic. Their role is distinct from the policy engine's: they do not determine policy; they apply decisions that the policy engine has produced. In the diagram, two enforcement points are identified with the `[AuthZ]` marker: the jump host in the Operations Zone and the local enforcement point in the Edge/Gateway Zone.

The jump host enforces at the operations-zone boundary. Its authorization decision concerns whether to admit the operator's session into the OT environment at all — a coarse decision that applies before any specific command is submitted. The local enforcement point enforces at the edge-zone boundary. Its authorization decision concerns whether a specific command to a specific resource is permitted for the authenticated operator — a fine-grained decision that applies per command.

These two enforcement points address different aspects of the same authorization model using the same policy engine. The coexistence of multiple enforcement points covering different boundary crossings is not redundancy — it is coverage. The jump host cannot evaluate field-device-level commands because they have not yet been specified; the local enforcement point cannot evaluate session-level admittance because the session has already been established. Each enforcement point addresses the authorization question that is visible at its position in the request flow.

---

## Protocol Adapters and Semantic Normalization

Field-level OT protocols were not designed to carry the subject, resource, and action primitives that the authorization model requires. A BACnet WriteProperty service request carries a device address, an object identifier, a property identifier, and a value. It does not carry an authenticated subject identity, a structured resource descriptor in the format the policy engine expects, or an action label that maps to the authorization model's vocabulary.

Protocol adapters bridge this gap. Each adapter is responsible for two functions: translating field-protocol messages into the shared semantic representation used by the authorization and audit infrastructure, and translating authorized commands back into field-protocol format for delivery to the controller.

In the normalization direction, the adapter receives a field-protocol message, extracts the operationally meaningful content — what device, what point, what operation, what value — and maps it to the authorization model's primitives: resource descriptor for the specific controller point, action label for the specific operation type, and identity context from the authenticated session in which the message arrived. The result is a structured authorization request that the enforcement point can submit to the policy engine without any knowledge of BACnet or Modbus semantics.

In the delivery direction, the adapter receives an authorized command from the enforcement point — specified in the shared representation — and translates it into the appropriate field-protocol message format for the target device.

This separation has a significant architectural consequence: the policy engine is protocol-agnostic. It evaluates authorization requests expressed in the shared representation, regardless of whether those requests originated from a BACnet session, a Modbus session, or any other protocol. A policy that grants a specific operator access to a specific class of controller points applies consistently, regardless of which protocol the controller speaks. Adding a new protocol requires implementing a new adapter; it does not require changing the policy model, the enforcement points, or the audit pipeline.

In the diagram, two protocol adapters appear in the Edge/Gateway Zone: one for BACnet and one for Modbus. Both connect to the local enforcement point for authorization evaluation. Both connect to the telemetry pipeline for normalized telemetry output. The policy engine interacts with neither adapter directly — it interacts with the enforcement point, which interacts with the adapters. This layering keeps protocol-specific logic contained in the adapter layer and out of the shared authorization infrastructure.

Not all field-device communication is amenable to adapter-based normalization. Maintenance interfaces accessed outside the normal communication path, vendor diagnostic tools with proprietary protocols, and legacy devices with no network interface cannot be intermediated by software adapters. These gaps require compensating controls — physical access restrictions, manual audit procedures, out-of-band session recording — rather than architectural workarounds. The adapter model improves coverage for the majority of field-device communication; it does not provide universal coverage.

---

## Distributed Enforcement and Centralized Policy

A common concern about centralized policy evaluation is that it introduces a single point of failure: if the policy engine is unavailable, enforcement fails across the deployment. A related concern is that centralized evaluation introduces latency that is unacceptable in time-sensitive OT operations.

The architecture addresses both concerns through a design that separates the location of policy evaluation from the location of policy enforcement. Policy is evaluated centrally — at the policy engine, where policy consistency is maintained — and applied locally — at enforcement points, where operational traffic is present and latency requirements are immediate.

The mechanism is the local policy cache, shown in the diagram as a component in the Edge/Gateway Zone connected to the local enforcement point. The policy engine periodically distributes the current applicable policy to the local cache. The local enforcement point evaluates authorization requests against the local cache rather than submitting real-time queries to the policy engine for every request. The policy engine is consulted for requests that require evaluation against current policy state — such as initial session establishment — but routine per-command enforcement uses the local cache.

This design has several properties. First, the enforcement point continues to make authorization decisions when the policy engine is temporarily unreachable, because the local cache retains a copy of applicable policy. The duration for which this works is bounded by the staleness limit of the cache — after which cached policy may no longer reflect the current authorization state. Second, the latency of per-command authorization evaluation is bounded by the time to evaluate against the local cache, not by the round-trip time to the policy engine. For deployments with constrained network connectivity to central services, this can be a significant operational improvement. Third, policy updates distributed by the policy engine reach all local caches through the control plane, keeping all enforcement points consistent.

The diagram reflects this design explicitly. The dashed control-plane arrow from the policy engine to the local policy cache represents policy distribution. The dashed arrow from the local policy cache to the local enforcement point represents the enforcement point's use of cached policy for authorization evaluation. The dashed arrows between the enforcement points and the policy engine represent real-time queries for the cases where cached evaluation is insufficient.

The distinction between distributed enforcement and centralized policy is important to state plainly: having enforcement points at multiple locations in the deployment does not mean having multiple policy authorities. Policy is authored, versioned, and maintained in one place. It is distributed to enforcement points for local application. Enforcement points do not independently determine what is permitted; they apply what the policy engine has determined. An enforcement point that diverges from the policy engine — whether through misconfiguration, stale cache, or compromise — is an operational and security problem, not a designed mode of operation.

---

## Control Plane and Data Plane

The diagram uses two distinct arrow styles to distinguish two distinct categories of traffic: solid arrows for data-plane flows and dashed arrows for control-plane flows. This visual distinction reflects a structural architectural distinction that has operational consequences.

**Data-plane flows** carry operational workloads: operator commands propagating toward field devices, device telemetry propagating toward the telemetry pipeline, sensor readings flowing between sensors and controllers. These are the flows that the system exists to support. They are high-volume relative to control-plane flows, latency-sensitive, and directly relevant to the physical operation of the building or facility.

**Control-plane flows** carry authorization infrastructure: policy distribution from the policy engine to the local cache, authorization requests and decisions between enforcement points and the policy engine, credential validation between the identity provider and the jump host, and identity context from the identity provider to the policy engine. These flows are lower-volume, higher-privilege, and concerned with the state of the authorization model rather than with the operational state of the field devices.

The structural consequence of this distinction is that the two categories of traffic should be treated differently: different access controls, different latency tolerance, different audit coverage, and — where feasible — different network paths. A change to a policy is a control-plane event that should be reviewed, approved, and audited as such. A setpoint adjustment is a data-plane event subject to authorization enforcement. Conflating these categories makes both harder to manage.

The diagram makes one architectural constraint visible through the absence of connections: there are no solid data-plane arrows entering the Central Services Zone. Operational traffic does not traverse the policy engine or the identity provider. Operator commands arrive at the jump host, are authorized by the enforcement point at the jump host boundary, and proceed through the OT gateway and local enforcement point to the protocol adapters and controllers — all in the data plane. The policy engine participates through control-plane flows: it distributes policy, receives authorization requests, and returns decisions. It does not carry the commands themselves.

This separation is important for two reasons. First, it means that the availability and latency of the policy engine does not directly constrain the availability and latency of operational command execution — enforcement operates against the local cache, not against the policy engine in the path of each command. Second, it means that the compromise of the data plane — a compromised gateway, a manipulated protocol adapter — does not automatically compromise the control plane. The policy engine and identity provider are administratively separate from the OT operational infrastructure.

In practice, logical separation of control and data planes is the minimum requirement. Physical separation — distinct networks for control-plane and data-plane traffic — should be pursued where the deployment environment permits. Many OT deployments will not achieve physical separation immediately, particularly when retrofitting identity-aware authorization onto existing network infrastructure. Logical separation, implemented through access controls, distinct credentials, and separate audit treatment for control-plane activity, is the achievable baseline.

---

## Centralized Auditability

The audit pipeline is not a logging feature appended to the authorization system — it is an architectural first-class component with specific properties required for the audit record to be trustworthy.

Authorization decisions must be recorded at the point where they are made. The enforcement point that applies a decision — permit or deny — emits a structured audit event containing the authenticated subject identity, the resource descriptor, the requested action, the policy version in effect, the authorization decision, and a timestamp. This event is emitted regardless of whether the underlying operational action succeeds or fails; the authorization decision is the event, not the operational outcome.

In the diagram, audit events flow from both enforcement points — the jump host and the local enforcement point — to the audit log store. The policy engine separately records every evaluation it performs, as a decision record, in the same store. The two audit streams are independent: if an enforcement point fails to emit an audit event, the policy engine's decision record remains. If the policy engine fails to record an evaluation, the enforcement point's audit event remains. Neither alone is sufficient for a complete audit trail, but together they provide redundant coverage.

For the audit record to be useful, it must be structured consistently across all enforcement points. An audit event from the jump host and an audit event from the local enforcement point must use the same schema — the same field names, the same identifier formats, the same timestamp precision — so that events from both can be queried and correlated against each other. A single authorization request that is evaluated at the jump host boundary and then again at the edge boundary should produce two related audit events that can be joined by a common session or request identifier.

For the audit record to be trustworthy, it must be written to a store that is administratively independent of the OT operational systems it covers. An audit record stored on the same system as the operational components — and under the same administrative control — can be modified or deleted by the same principals whose actions are being recorded. Administrative separation is a prerequisite for the audit record's integrity as evidence.

Immutability is the property that prevents modification after the fact. Append-only storage, cryptographic chaining of audit records, and replication to out-of-band systems with separate administrative access are implementation mechanisms for this property. The specific mechanism is less important than the property it provides: a credible assurance that the audit record has not been modified since the event was recorded.

---

## Offline Resilience and Local Policy Caching

OT environments cannot accept authorization infrastructure that is a hard dependency for operational continuity. A policy engine that is temporarily unreachable should not prevent an authorized operator from responding to a building emergency. A transient network partition between an edge site and central services should not cause the site to lose the ability to authorize operational commands.

The local policy cache is the primary mechanism for offline resilience. It holds a time-bounded copy of the policy applicable to the traffic the local enforcement point governs. When the policy engine is reachable, the cache is synchronized periodically; when it is not, the enforcement point evaluates against the cached copy. The bound on how long this can continue is the staleness limit — the maximum age at which cached policy is considered current enough to enforce.

Setting the staleness limit involves a tradeoff. A short limit — hours rather than days — reduces the window in which stale policy can be enforced, limiting the exposure if a policy change (such as a credential revocation) has not yet reached the cache. A long limit provides more resilience to extended connectivity outages but increases the risk that cached policy diverges from intended policy. The appropriate limit depends on the operational risk of stale authorization decisions in each deployment context — a decision that must be made by the operators of that deployment, not prescribed by the architecture.

Beyond policy caching, the local enforcement point must have defined behavior for conditions that the cache cannot address: credentials that expire while the policy engine is unreachable, revocation events that cannot propagate to the cache, and cache entries that age beyond their staleness limit without renewal. Each of these conditions requires a defined operational response — not a silent fallback to permit-all or deny-all behavior, but a documented, tested, and understood behavior that operators can predict and plan around.

The enforcement point's behavior when it encounters a failure it cannot resolve — internal crash, resource exhaustion, unrecoverable error — must also be explicitly defined. The choice between failing closed (denying all requests) and failing open (permitting all requests) is a safety and security tradeoff that is context-specific. For most enforcement points, failing closed is the safer default. For enforcement points guarding access to life-safety systems where denial under failure could prevent an emergency response, the acceptable failure behavior may be different. This tradeoff must be documented, reviewed by both security and operations stakeholders, and tested under realistic failure conditions — not assumed to be correct because the implementation compiles.

---

## Context-Aware Authorization

Role assignments establish a baseline of what a subject is permitted under ordinary conditions. Many operationally meaningful access distinctions cannot be expressed through role assignments alone.

Context-aware authorization extends the policy model to incorporate attributes of the environment at the time of the request: the current time, the declared operational mode of the building or site, the physical location of the subject or the target resource, whether a maintenance window is active, or whether an emergency condition has been declared. These contextual attributes allow policies to express distinctions that static role models cannot — for example, that maintenance technicians may write controller configuration only during a declared maintenance window, or that override commands may be issued only by operators with site-lead status during an active alarm.

The architectural requirement for context-aware authorization is that contextual attributes must be trustworthy at the time they are used in policy evaluation. An attribute that says "maintenance window active" must be set through a process that is itself access-controlled and audited; otherwise it is a mechanism for circumventing the access controls it conditions. The trustworthiness of contextual inputs is as important as the correctness of the policies that consume them.

Context-aware policies are also more complex to reason about than pure role-based policies. A policy that combines a role condition with multiple contextual conditions produces an authorization decision that depends on the intersection of those conditions — an intersection that may produce unexpected outcomes when conditions interact in ways that were not anticipated during policy authoring. Policy testing must cover not only individual conditions but combinations of conditions that could plausibly co-occur in production.

Context should be introduced into the policy model where it materially improves the precision of access decisions — where the operational distinction it enables cannot be expressed any other way. It should not be introduced reflexively as a feature of every policy, because the complexity it adds must be justified by the operational value it provides.

---

## Architectural Constraints and Tradeoffs

The architecture described in this section provides capabilities that protocol-centric models cannot: consistent policy across heterogeneous systems, end-to-end attribution of actions to authenticated identities, a unified audit record, and the ability to reason centrally about authorization decisions. These capabilities come with costs and constraints that must be understood before adoption.

**Latency.** Submitting an authorization request to the policy engine adds latency to the request flow. For supervisory-layer operations — operator commands routed through the jump host and edge zone — the added latency of a local cache evaluation is typically acceptable. For time-critical control loops at the field device level, latency introduced by authorization evaluation may not be. The architecture accommodates this by positioning enforcement at boundaries where latency can be tolerated, not inside time-critical loops. Enforcement between the edge zone and the field device zone applies to commands crossing that boundary; it does not apply to the millisecond-scale communication between a controller and its directly connected sensors.

**Legacy device coverage.** Field devices that have no network interface and no participation in authentication protocols are not directly covered by the identity-aware authorization model. They receive commands that have been authorized at the boundary level — the edge zone enforcement point — but the device itself cannot verify that an incoming command was authorized. Physical access controls and maintenance procedure discipline are required to close this gap at the device level. The architecture improves the security posture of these environments substantially without fully addressing the device-level constraint.

**Operational complexity.** An identity-aware authorization architecture requires infrastructure that must be operated, monitored, and maintained: the identity provider, the policy engine, the enforcement components, and the audit pipeline. This infrastructure has its own availability requirements, its own update and patching requirements, and its own operational failure modes. The operational cost of this infrastructure must be evaluated honestly against the operational burden of the distributed, fragmented authorization it replaces.

**Partial deployment.** The architecture does not require universal deployment to deliver value. An environment with enforcement at the operations-zone boundary — jump host — but not yet at the edge zone is meaningfully more auditable and accountable than one with no enforcement points. Each layer of enforcement added improves coverage without requiring full deployment as a prerequisite. Partial deployment is a realistic adoption path, not an incomplete architecture.

**High availability.** The identity provider and policy engine are components that all enforcement points depend on for initial session establishment and policy synchronization. Their availability requirements are not the same as those of the operational OT systems — a five-minute outage of the policy engine is not equivalent to a five-minute outage of a BAS controller — but they must be designed and operated with sufficient availability to meet the operational requirements of the deployments that depend on them. Redundancy, failover design, and recovery time objectives for these components must be explicitly planned.

---

## Summary

Identity-aware authorization in OT environments is not a replacement for network segmentation, field-protocol architecture, or controller-level access controls. It is an additional architectural layer — positioned above and alongside the existing field infrastructure — that provides capabilities the existing infrastructure cannot: verified subject identity, policy-based authorization evaluated against a consistent shared model, end-to-end identity propagation across trust boundary crossings, and a centralized audit record that covers the full request lifecycle.

The model is built on three primitives — subjects, resources, and actions — that define every authorization request. It evaluates requests through a shared policy engine that is protocol-agnostic, applying policy expressed in terms of OT concepts rather than field-protocol specifics. It enforces decisions at trust boundary crossings through distributed enforcement points that apply decisions locally, using a local policy cache to support offline operation without requiring real-time connectivity to central services. It records decisions through an audit pipeline that is independent of operational infrastructure, structured consistently across all enforcement points, and written in a form that survives individual component failure.

The architecture is constrained by the operational realities it must coexist with: latency sensitivity at the field level, limited device capability for authentication participation, partial deployment realities in heterogeneous environments, high availability requirements for the infrastructure it introduces, and the maintenance window patterns and operational workflows of the teams who will operate it. These constraints do not invalidate the architecture — they shape it. An authorization infrastructure that acknowledges operational constraints and designs around them is one that can actually be adopted. One that ignores them will be worked around.

The following sections examine the trust boundaries where enforcement decisions are made in detail, describe the BASIS proof-of-concept implementation that validates selected aspects of this architecture, and address the production engineering challenges that arise when the model moves from a controlled research context toward real operational deployments.
