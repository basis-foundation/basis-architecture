# OT Trust Boundaries

Section 04 established the conceptual architecture: a policy engine that evaluates authorization requests, enforcement points that apply decisions, protocol adapters that normalize field-protocol messages, and a local policy cache that supports offline operation. That architecture is deliberately topology-independent — it describes what the components do, not where they belong or why.

This section supplies the topology. It examines where trust actually changes in OT deployments, what those changes mean operationally, and why enforcement placement follows from topology rather than from security preference alone. The primary reference diagram for this analysis is the trust zone and boundary crossing diagram (`diagrams/ot-trust-boundary-overview.mmd`), which depicts five operational zones, the enforcement points at their boundaries, and the control-plane and data-plane flows between them. The authorization flow diagram (`diagrams/identity-aware-authorization-flow.mmd`) provides supporting detail on individual component relationships within zones.

---

## Trust Boundaries Are Not Firewall Locations

A trust boundary is not a firewall rule. Firewalls are one mechanism for enforcing a boundary — but the boundary itself is an operational and architectural fact, not a configuration artifact. Trust boundaries exist wherever two connected domains operate under different assumptions about who or what is present, what they are permitted to do, and how that determination is made.

In OT deployments, these differences accumulate at the same physical and logical points, which is why the zones in the reference architecture correspond simultaneously to administrative divisions, device capability differences, and operational latency domains. The operations zone is administered by facility operations staff, hosts human-operated workstations, and requires authentication mechanisms suited to human principals. The field device zone is administered — in many deployments, only loosely — by whoever commissioned the original controller installation; it contains hardware with no authentication capability and operates at latency tolerances incompatible with real-time policy queries. These are not distinctions imposed by security architecture. They were true before any authorization infrastructure was introduced, and they determine where enforcement can be placed.

Enforcement placement is constrained by three simultaneous requirements: traffic must be observable at the enforcement point, identity context must be available to attach to authorization requests, and the evaluation latency must be tolerable for the operational role of the traffic being governed. Firewalls enforce traffic reachability; enforcement points enforce authorization semantics. These are different functions, and an enforcement point cannot exist where identity context is absent, regardless of what the surrounding network topology permits or denies.

This also means that trust boundaries are not uniform in character. Some boundaries are defined primarily by administrative ownership — who controls the systems on each side. Some are defined primarily by device capability — whether the systems on the far side can participate in identity verification. Some are defined by operational latency tolerance — whether the control loop on the far side can absorb the additional latency that authorization evaluation introduces. The analysis that follows identifies which of these characteristics dominate at each zone boundary in the reference architecture.

---

## Zone Analysis

The trust zone diagram identifies five operational zones. Each zone carries distinct trust assumptions, identity characteristics, latency tolerances, autonomy requirements, enforcement implications, and audit characteristics. The following analysis proceeds from the infrastructure zone that governs all others toward the field devices that receive, but cannot verify, the decisions those zones produce.

### Central Services Zone

The Central Services Zone contains the identity provider, the policy engine, the audit log store, and the telemetry pipeline. No operational equipment operates here and no data-plane traffic traverses it — commands and telemetry do not pass through the policy engine or the identity provider. What flows through this zone is exclusively control-plane traffic: policy distributions to enforcement points, identity context for policy evaluation, authorization requests from enforcement points, and authorization decisions in return.

The trust model for this zone differs from all others: it is the zone of highest administrative privilege in the architecture. Policy is authored here, identities are provisioned here, and all audit records ultimately persist here. The compromise of this zone would not directly inject unauthorized commands into the field device layer — the data plane does not traverse it — but it would undermine the policy state and identity context that all enforcement decisions across the deployment depend on.

The Central Services Zone is also the zone with the most complete authorization visibility. Every enforcement point in the deployment reports its decisions to the audit store here. Every policy evaluation is recorded here. This completeness is contingent on connectivity: enforcement points that are disconnected from the central audit store cannot deliver events in real time, and events buffered locally during connectivity gaps may arrive out of order or not at all if local buffer capacity is exhausted.

High availability requirements for this zone differ in character from those of operational zones. A five-minute outage of the policy engine does not interrupt field operations — local policy caches at edge enforcement points continue to function during the gap. But it does prevent new session establishment at the operations zone boundary, prevents policy updates from reaching enforcement points, and creates a gap in the real-time audit record. Recovery time requirements must be defined against these specific operational impacts, not against the availability standards of the field infrastructure the zone governs.

### Operations Zone

The Operations Zone is where human operators and technicians initiate access to the OT environment. It contains operator and technician workstations and the jump host, which is the first enforcement point in the inbound access path.

The defining characteristic of this zone is that it is where human identity is established. Operators present credentials through the jump host, those credentials are verified by the identity provider, and the resulting verified identity context is attached to the session before any access to the OT environment is admitted. The trust assumption in the Operations Zone is that subjects are authenticated individuals whose identity can be verified through mechanisms appropriate for human principals — multi-factor authentication, smartcard-backed credentials, or equivalent.

From a latency perspective, the Operations Zone operates in the domain of human-perceptible time. An authorization evaluation that adds 200 milliseconds to a setpoint command submitted through a workstation interface is not operationally significant for a human operator. This latency tolerance makes it feasible to submit session-level authorization requests from the jump host to the policy engine in real time, rather than relying on cached policy alone. The local policy cache at this boundary is a reliability measure; it is not the primary evaluation mechanism in the way it is at the edge zone.

The enforcement decision at the jump host is session-level, not command-level. The question it answers is whether this authenticated identity may establish a session that routes toward the OT environment, given the current policy and context. It does not evaluate the specific commands the operator will issue — those commands are not yet known at session establishment. Session admission at the operations zone boundary is a necessary but not sufficient authorization event; per-command authorization at the edge enforcement point is required in addition to it.

Operational failures at the jump host produce session denial. An operator whose session is rejected has no authorized path into the OT environment. The risk this creates — that an authorized operator is denied access during an emergency — is the primary driver for jump host redundancy and for defined break-glass procedures that produce audit records even when the normal authorization path cannot complete.

### Supervisory Zone

The Supervisory Zone occupies the layer between the Operations Zone and the Edge/Gateway Zone. It contains the building management system or SCADA supervisor — itself an enforcement point — and the HMI workstations that provide local operational interfaces to on-site personnel.

The trust transition at the supervisory zone boundary differs in character from the jump host crossing. The Operations Zone hosts remote principals: individuals who may be accessing the system from a different site or through a managed remote access service. The Supervisory Zone hosts local operational systems: the BMS that aggregates data from and issues commands to the building infrastructure, and the HMI terminals through which on-site technicians interact with the supervisory layer directly. The shift from remote human identity to local system identity — and to human-at-console identity — changes the authentication mechanisms that are available and appropriate at this boundary.

The BMS supervisor carries an enforcement point (`[AuthZ]` in the trust zone diagram) because commands arriving from both remote operators and the local system must be authorized before they proceed toward the field device layer. The BMS occupies a dual role: it is a subject — it generates autonomous commands as part of its scheduled and reactive control logic — and it is an enforcement intermediary, routing operator-initiated commands toward the gateway and edge. This dual role means the enforcement point at the BMS boundary must distinguish between commands the BMS generates itself and commands it is forwarding on behalf of an authenticated operator, and the identity context attached to each must reflect the actual originating subject.

Autonomous commands generated by the BMS — scheduled setpoint adjustments, reactive responses to sensor readings, load-shedding actions triggered by building conditions — carry system identity rather than operator identity. A policy that governs what the BMS can do autonomously is a different kind of policy than one that governs what an operator can do through the BMS. Both must be expressible in the shared policy model and both must produce audit records that accurately attribute the command to its actual source: the BMS system principal for automated actions, and the propagated operator identity for operator-initiated commands relayed through the supervisory layer.

HMI workstations in the Supervisory Zone operate under a different authentication posture than remote operator workstations. Physical presence at an HMI terminal implies that the individual has already passed whatever physical access controls protect the space containing the terminal. This contextual difference is a legitimate input to policy evaluation: a command issued from a local HMI during a declared maintenance window may warrant a different authorization outcome than the same command issued remotely from an operator workstation at an off-site location. The policy model must be able to express this distinction without treating physical presence as unconditional trust.

The Supervisory Zone also introduces the possibility of local operational autonomy that is independent of remote operator involvement. A BMS that is disconnected from the Central Services Zone and from the Operations Zone may continue managing the building according to its local programming and stored schedules. Whether this autonomous operation is authorized during a connectivity gap, and under what conditions, must be explicitly defined. It cannot be assumed to be permitted or denied by default.

### Edge/Gateway Zone

The Edge/Gateway Zone is the operational boundary between the supervisory infrastructure and the field device layer. It contains the OT gateway, the local enforcement point, the local policy cache, and the protocol adapters for each field protocol in use. The enforcement point here applies the last authorization check before commands reach field devices — and it is the enforcement point operating under the tightest latency and connectivity constraints in the deployment.

The trust transition at the edge zone boundary involves three simultaneous shifts: from session-level traffic to per-request field-protocol commands, from subjects with established identity context to traffic that will be delivered to devices with no identity participation capability, and from human-perceptible latency tolerance to the tighter timing constraints of supervisory control loops.

The latency shift drives the local policy cache design. The enforcement point at this boundary cannot submit a real-time query to the policy engine for each field-protocol command and remain within the latency budget of the control loops it governs. The local policy cache holds a time-bounded copy of applicable policy, against which the enforcement point evaluates requests locally. The staleness limit — the maximum age at which cached policy is considered current enough to enforce — is a deployment-specific parameter that trades resilience against the risk of enforcing outdated policy. A credential revocation that has not yet propagated to the cache will not prevent a previously authorized session from issuing commands during the cache validity window. This is an accepted tradeoff, not a defect, but it must be understood by operators who set the staleness limit.

Protocol adapters in this zone represent a trust boundary of a different type: a semantic trust boundary. When the adapter receives a BACnet WriteProperty request and normalizes it into the authorization model's subject-resource-action representation, it is performing a translation that the enforcement point and policy engine trust unconditionally. Neither has the protocol knowledge to verify the adapter's normalization independently. The accuracy of the adapter's normalization is therefore a precondition for the correctness of every authorization decision that follows from it. This is addressed through adapter implementation testing and version management, not through runtime verification at the enforcement point.

Audit at the edge zone is subject to connectivity constraints that do not apply to the Operations Zone. When the edge site is connected to central services, audit events flow promptly to the central audit store. During connectivity gaps, events buffer locally. The buffer size, the delivery guarantee, and the behavior when the buffer fills must be explicitly defined and tested. An audit gap caused by a disconnected edge site is a different operational condition from a gap caused by events that were never generated — but both produce the same symptom in the central audit record: absence of events for a time window.

### Field Device Zone

The Field Device Zone contains BAS controllers, sensors, and actuators — the equipment that directly monitors and controls physical building processes. Its defining characteristic for the authorization architecture is that it is enforcement-downstream: field devices cannot participate in the authorization model in any capacity.

A BAS controller has no mechanism for verifying that an incoming BACnet WriteProperty request was authorized by a policy engine. It can confirm that the request arrived on the expected communication channel and is correctly formatted. It cannot confirm that the subject who initiated the request had permission to do so, or that the request passed through any enforcement point. The security coverage of the field device zone is entirely a function of the integrity of the enforcement applied at the edge zone boundary above it.

Physical access to the field device zone — or unauthorized network access that bypasses the edge enforcement point — removes the authorization architecture's coverage entirely. A technician who connects directly to a controller's local maintenance port, bypassing the OT gateway and local enforcement point, is issuing commands that the authorization architecture cannot observe or govern. Physical access controls, maintenance procedure requirements, and out-of-band session recording are the controls that address this gap. The software authorization architecture cannot close it.

The identity model in the field device zone is path identity rather than device identity in the strong sense. The protocol adapter can verify that telemetry arrived on the expected communication path from the expected device address. It cannot verify that the device at that address is the authentic device rather than a substitute. This is not a failure of the adapter design — it is a consequence of what the field protocols and the device hardware generations present in most BAS deployments support. Hardware attestation mechanisms for field devices exist in some product categories but are not yet common in the equipment generations that most real deployments contain.

Operational failures in the field device zone — communication timeouts, controller faults, sensor dropouts — do not produce authorization events. They produce operational telemetry events and communication errors that surface through the telemetry pipeline. The local enforcement point does not interpret a failed command delivery as an authorization failure; it interprets it as an operational condition. The distinction between authorization failures and operational failures must be maintained in the audit and monitoring infrastructure, or operational noise will obscure authorization-relevant signals.

### Remote Access: Zone Entry Patterns

Remote access — vendor maintenance sessions, off-site operator access, third-party monitoring integrations — does not originate within any operational zone. It enters the architecture from outside the OT environment, crossing a trust boundary at the edge of the Operations Zone before any OT-specific authorization can be applied.

The trust transition at this entry point differs from the intra-zone transitions in one important respect: the identity context presented by a remote session may be issued by an external identity infrastructure rather than by the deployment's own identity provider. A remote operator connecting through an enterprise VPN brings enterprise directory credentials. A vendor technician connecting through a managed remote access service brings vendor-issued credentials. A third-party monitoring platform authenticates as a service account managed by neither the OT operator nor the vendor.

The enforcement model at the operations zone boundary must be able to evaluate externally issued identities against the same policy model that governs locally issued identities. This requires that the identity provider either federate directly with external identity sources or accept and verify externally issued tokens that carry the attributes required for policy evaluation. Policy applicable to remote sessions should be narrower than policy applicable to locally administered operator identities, because the OT operator's visibility into the external party's access management practices is limited.

Vendor remote access warrants specific analysis because its trust semantics differ from remote operator access in ways that have direct enforcement implications. A vendor session is granted to support a specific maintenance task on specific systems, typically within a defined time window. The authorization model should reflect these constraints: the vendor's scope should be bounded to the systems they are engaged to maintain, the session should carry a time-based expiration consistent with the maintenance window, and every command issued within the session should generate an audit event with full identity context. The policy governing vendor sessions is not simply a subset of operator policy — it is a distinct policy that reflects the distinct trust relationship with an external party.

---

## Trust-Boundary Crossings: Enforcement Placement and Its Consequences

Three crossings in the reference architecture carry the most significant operational and security consequences for enforcement placement.

### The Operations-to-Supervisory Crossing

At the boundary between the Operations Zone and the Supervisory Zone, the jump host applies session-level authorization before admitting traffic toward the BMS and the field infrastructure beyond it. The authorization question here is coarser than the per-command questions asked downstream: is this authenticated identity permitted to establish a session that routes into the OT environment, given current policy and context?

The coarseness of this decision is appropriate to what is knowable at session establishment — the operator's role, their authentication method, the declared operational context. But it means that session-level authorization at the operations zone boundary cannot substitute for command-level authorization at the edge. A session admitted through the jump host carries identity context forward, but each command issued within that session must still be evaluated at the edge enforcement point. Admission is a gate, not a grant.

Identity context established at this crossing must propagate through every downstream component the session traverses. If the BMS or the gateway substitutes its own system identity for the propagated operator identity when forwarding commands, the edge audit record will attribute commands to the forwarding system rather than to the operator who issued them. An auditor examining the edge enforcement point's audit events cannot distinguish, from those records alone, between an operator-initiated command and an autonomous BMS action, unless identity propagation was maintained through the full chain.

### The Supervisory-to-Edge Crossing

At the boundary between the Supervisory Zone and the Edge/Gateway Zone, the gateway routes authorized sessions toward the local enforcement point, which applies per-command authorization against the local policy cache.

This crossing is where the enforcement model shifts from session-level to command-level governance. The local enforcement point receives a command — normalized by the protocol adapter into the shared subject-resource-action representation — and evaluates whether the authenticated subject has permission to perform the requested action on the specified resource under current cached policy. The quality of this evaluation depends on two things simultaneously: the accuracy of the protocol adapter's normalization and the currency of the local policy cache.

The accuracy dependency on the adapter creates a risk at protocol translation boundaries. When a controller's object model changes — a new property is added, a register map is reconfigured, a firmware update alters the semantics of an existing object — the adapter's normalization may no longer accurately reflect the field-protocol reality. The enforcement point will continue evaluating authorization requests against the outdated normalization. Protocol change management must be synchronized with adapter configuration updates; they are not independent operational concerns.

### The Edge-to-Field Transition: Enforcement Downstream

The transition from the Edge/Gateway Zone to the Field Device Zone is not a crossing where additional enforcement is applied. It is the boundary at which enforcement coverage ends. Traffic that reaches a BAS controller has, by construction, passed through the local enforcement point. Nothing in the field device zone will verify that claim independently.

The security property this creates is contingent: the field device zone is protected to the extent that the enforcement applied at the edge is correct, complete, and not circumventable. Unauthorized access paths that bypass the OT gateway — direct connections to controller network ports, unmonitored communication channels, legacy maintenance interfaces that predate the current deployment topology — are outside the authorization architecture's coverage and must be addressed through physical and procedural controls the software architecture cannot substitute for.

---

## Asymmetric Trust: Commands Inward, Telemetry Outward

The most operationally significant asymmetry in the trust model is the difference between commands flowing toward field devices and telemetry flowing outward from them. These flows are not symmetric in their authorization requirements, and treating them as equivalent in the enforcement or audit model produces incorrect analysis.

An inbound command is an assertion of intent by a subject: this entity requests that this resource perform this action. The authorization model applies directly. There is a subject whose credentials can be verified, a resource that can be described by its structured identifier, and an action that can be evaluated against policy. The enforcement point can confirm whether the subject is authorized before the command reaches the field device.

Outbound telemetry originates from devices that cannot authenticate themselves. A temperature sensor reports a reading. A BAS controller reports its operating state. Neither has a subject identity to present, and neither is requesting permission — they are reporting observations. The authorization question that governs inbound commands does not map onto device-originated telemetry. The relevant trust question for telemetry is not "is this subject authorized to perform this action?" but "is this data from a source and path that should be trusted for the purposes it will be used for?"

Source trust for outbound telemetry is provided by the integrity of the communication path. The protocol adapter receives telemetry from devices on the expected channel with the expected addressing. It normalizes the reading and routes it to the telemetry pipeline with whatever path metadata is available. The telemetry pipeline trusts this data to the extent it trusts the adapter that produced it. A compromised or misconfigured adapter can inject false telemetry that the pipeline will accept, because the authorization model cannot verify the originating device's identity at the source.

This asymmetry has a direct consequence for the audit model. Authorization audit events for inbound commands carry verified subject identity and a recorded policy decision. Telemetry audit events carry path metadata but not verifiable device identity. The two kinds of records have different evidential weight. Treating them as equivalent in audit analysis conflates what the architecture can guarantee — that a verified subject was authorized to issue a command — with what it can only observe: that a reading arrived on a given path from a given device address.

---

## Identity at Protocol Translation Boundaries

Protocol translation boundaries — where field-protocol messages are normalized by protocol adapters — are semantic trust boundaries. The trust that the enforcement point and the policy engine place in the adapter's output is categorical: neither has the protocol knowledge to independently verify the normalization at evaluation time.

When a BACnet WriteProperty service request is mapped to the authorization model's subject-resource-action primitives, the adapter's accuracy determines whether the policy engine is evaluating the right question. If the adapter maps a write operation on a safety-critical control point to the resource descriptor of a non-critical monitoring point, the policy engine will produce a decision for the wrong question. An allow decision under that misconfigured normalization would be meaningless as authorization evidence.

The operational implication is that adapter correctness is a security requirement, not just a functional one. Adapter implementation must be tested against the full object model of the devices it serves. Adapter configuration must be version-controlled and subject to change review. Adapter updates must be reviewed for normalization correctness before deployment, because errors in normalization are not detectable by the enforcement or audit infrastructure at evaluation time.

Changes in the field device layer that affect the semantic content of protocol messages — new controller firmware, reconfigured device object models, expanded register maps — must trigger adapter configuration review as part of the change management process. A device configuration change that is not reflected in the adapter produces a semantic divergence between what the policy engine believes it is authorizing and what is actually executing in the field. The authorization and field infrastructure are not independent operational concerns; their configuration change cycles must be coordinated.

---

## Audit Visibility Across Administrative Boundaries

Audit coverage degrades predictably as distance from the Central Services Zone increases and as the administrative independence of components increases. Understanding where and why it degrades is necessary for correctly interpreting what the audit record the architecture produces can and cannot support.

In the Central Services Zone, coverage is comprehensive. Every policy evaluation, every identity lifecycle event, every authorization decision submitted by an enforcement point is recorded at the source and written to the audit store. The audit store is administratively independent of the systems it covers.

In the Operations Zone, audit events are generated at the jump host and transmitted to the central audit store. Coverage here is strong for session-level events — admission, denial, session termination — and depends on the jump host's connectivity to the audit store. The jump host is typically better connected to central services than edge enforcement points, but buffering behavior and delivery guarantees must still be explicitly defined.

At the Supervisory Zone, the BMS enforcement point generates audit events for the commands it evaluates. Commands that pass through the BMS enforcement point using locally cached policy without a real-time policy engine query may produce audit events only at the enforcement point, not at the policy engine. When both the policy engine's decision record and the enforcement point's audit event are expected for a complete audit trail, the absence of either must be detectable and interpretable.

At the Edge/Gateway Zone, connectivity constraints affect both event delivery timing and the completeness of the central audit record during connectivity gaps. Events are not necessarily lost — local buffering preserves them — but their arrival at the central store is delayed, and the central audit record will have a temporal gap during the disconnection period. Forensic analysis of activity during a connectivity gap must account for reordered or delayed events.

Vendor remote access creates a structural audit gap that connectivity cannot address. The vendor session may generate audit events in the vendor's own infrastructure before it crosses the operations zone boundary — events that the OT operator has no access to. The audit record within the OT environment captures what the session did after the jump host admitted it; it does not capture what preceded that admission or what other activity the vendor credentials were used for simultaneously. This is a fundamental consequence of federated identity: the audit record follows the trust boundary of the deployment, not the lifecycle of the credential.

---

## Centralized Policy, Local Autonomy

The local policy cache at the edge enforcement point is a reliability mechanism, but it creates an operational condition that deserves explicit analysis: enforcement points operating on cached policy are exercising local autonomy within centrally defined bounds.

When the edge enforcement point evaluates a command against cached policy without querying the policy engine, it is making an independent authorization decision. That decision is bounded by the policy the central engine distributed, and it expires when the cache staleness limit is reached — but within those bounds, the enforcement point does not require central authorization for each evaluation.

This local autonomy is operationally essential in OT environments where connectivity to central services is intermittent. But it means that the correctness of enforcement during a connectivity gap depends on two things simultaneously: the integrity of the enforcement point itself and the accuracy of the policy it held when it last synchronized. A policy distribution event that delivered incorrect policy — whether through a policy authoring error, a misconfiguration in the distribution path, or a compromise of the distribution mechanism — would cause the enforcement point to make well-structured decisions against wrong policy. The enforcement point cannot detect this condition independently.

This dependency is the reason that the control-plane path — from the policy engine to the local policy cache — must be protected with the same administrative care as the policy engine itself. A compromised control-plane path is not merely a reliability problem. It is an authorization integrity problem with consequences that manifest at every enforcement point the compromised policy reaches.

---

## Component Implementation of Enforcement Points

The trust boundary analysis in this section describes enforcement points in terms of their architectural role — where they are positioned, what authorization questions they answer, and what trust transitions they manage. It does not specify which software component implements that role at each boundary.

In the BASIS ecosystem, enforcement is implemented by components that depend on the authorization kernel:

- **basis-gateway** hosts the authorization request lifecycle — receiving requests from subjects or forwarding systems, invoking the kernel for policy evaluation, and returning decisions. It is the runtime that enforcement points in the Operations Zone and Supervisory Zone typically surface as.
- **basis-adapters** perform the normalization that makes field-protocol traffic legible to the authorization model. A protocol adapter translates a BACnet write or a Modbus register operation into the subject-resource-action representation that the kernel can evaluate. The adapter also applies the decision the kernel returns, permitting or blocking the protocol-level operation accordingly.

The authorization kernel itself — basis-core — does not sit at trust boundaries as a deployed network component. It is the evaluation engine that gateway and adapter components call into. The distinction matters for understanding what the architecture requires at each boundary: the gateway or adapter is the deployed enforcement point; the kernel is the shared evaluation logic those enforcement points invoke.

This division reflects a general principle in the ecosystem: **adapters normalize, the kernel evaluates, and gateway or adapter services apply decisions.** The kernel does not normalize protocol-specific inputs. The adapter does not define evaluation semantics. Each component remains responsible for the function it owns.

The trust boundary topology described in this section is therefore a map of where gateway and adapter components must be positioned — not a map of where the kernel is deployed directly. The kernel is deployed once (or in a replicated configuration for availability), and the services that depend on it are distributed to the boundary positions this section identifies.

---

## Summary

Trust boundaries in OT deployments are not locations where security controls are optionally applied — they are points where the nature of trust fundamentally changes, and where the architecture must be positioned to reflect that change. Each zone in the reference architecture carries distinct trust assumptions that determine what authorization questions can be asked and answered at its boundaries, what identity context is available to attach to authorization requests, and what the audit record can reliably capture.

The Central Services Zone holds the policy and identity infrastructure that all enforcement depends on, and carries no data-plane traffic. The Operations Zone is where human identity is established; enforcement here is session-level and designed for human-perceptible latency. The Supervisory Zone introduces both a local enforcement point and a system-principal subject — the BMS — whose autonomous actions require distinct policy treatment from operator-initiated commands relayed through the same layer. The Edge/Gateway Zone is where per-command enforcement applies against field-protocol traffic, where latency tolerance tightens, and where the local policy cache is the primary operational mechanism for resilient authorization. The Field Device Zone is enforcement-downstream: devices here receive authorized commands but cannot verify the authorization, and the coverage they receive is entirely contingent on the integrity of enforcement applied at the preceding boundary.

The asymmetry between inbound commands and outbound telemetry is not a gap in the architecture — it is a structural consequence of the difference between subject-initiated actions, which carry verifiable identity, and device-originated observations, which cannot. The audit model must reflect this difference rather than treating all events as equivalent records of authorized activity.

Enforcement placement in this architecture is topology-determined. It follows the boundaries where trust changes, where identity context is available, and where the operational latency budget accommodates authorization evaluation. Understanding why enforcement is positioned where it is — and what it therefore cannot cover — is a prerequisite for correctly assessing the authorization posture of a deployment that uses this architecture in whole or in part.
