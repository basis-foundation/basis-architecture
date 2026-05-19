# Why Existing Approaches Become Insufficient

The authorization mechanisms described in the previous section — network segmentation, VLAN-based trust, controller-local credentials, shared operator accounts — were not designed in a vacuum. They reflect coherent engineering decisions made under specific constraints: closed networks, bounded operator populations, limited integration requirements, and protocol ecosystems that predated networked threat models. Within those constraints, they function.

The challenge is not that these approaches are wrong. The challenge is that the operational context they were designed for has changed in ways that expose structural limitations they were never built to handle. This section examines those limitations — not as indictments of the existing architecture, but as the specific conditions under which it begins to strain and the particular capabilities it cannot provide as requirements evolve.

---

## The Expanding Operational Context

Understanding why existing approaches become insufficient requires first understanding what has changed in the operational environment.

Building automation systems that once operated as isolated networks are now expected to participate in a much larger operational ecosystem. Energy management platforms aggregate consumption data from BAS subsystems across multiple buildings. Predictive maintenance systems ingest equipment telemetry to detect early failure indicators. Facility management software integrates with access control, HVAC, and lighting systems to respond to occupancy changes. Remote monitoring centers managed by facility operators or vendor support teams observe and respond to equipment conditions across geographically distributed sites.

Each of these integrations introduces a communication path that crosses what was formerly a closed boundary. The integration itself is often well-justified: the operational and efficiency value of connected building systems is real. But each path introduces a channel through which the trust assumptions of the BAS network are extended in ways the original architecture was not designed to accommodate.

Remote access has followed a similar trajectory. Direct physical access by on-site technicians has been supplemented by — and in some cases largely replaced by — remote access over VPN for vendor support, firmware updates, configuration management, and incident response. The operational benefit is clear: faster response times, access to specialized expertise without on-site travel, and reduced support costs. The architectural consequence is that the population of entities interacting with OT systems is no longer bounded by physical presence.

These changes do not make the existing architecture wrong. They make it insufficient for a set of requirements it was not designed to meet.

---

## Where Network-Centric Trust Reaches Its Limits

Network segmentation and VLAN-based trust provide a single enforcement surface: the network boundary. Traffic that originates within the trusted segment is permitted; traffic from outside is filtered. This is a binary model applied at a coarse granularity — the unit of trust is the network segment, not the device, the operator, or the action.

This model functions well when the network segment accurately reflects a coherent trust domain: a fixed set of known devices, accessed only by authorized personnel through controlled physical means, performing a well-understood set of operations. The model begins to strain when any of these conditions is relaxed.

When a VPN connection provides network-layer access to the BAS segment, the VPN user inherits the trust of the segment. If the BAS segment trusts all traffic from within it, then the VPN user is trusted to communicate with any device on that segment, issue any valid protocol command, and receive any data available to segment members — without the authorization model being involved at all. The question of *what* the authenticated VPN user should be permitted to do is not answered by the network architecture; it is implicitly answered as "anything a segment member can do."

This coarseness becomes a meaningful limitation as the population of remote users grows and as the diversity of legitimate use cases expands. A vendor supporting a specific chiller system should not have implicit protocol-level access to the access control subsystem. An analytics platform ingesting energy consumption data should not have the network-level ability to issue setpoint commands to HVAC controllers. An operator monitoring environmental conditions should not have the same effective access as a technician performing configuration changes. Network segmentation cannot express these distinctions; it was not designed to.

The limitation is not that network segmentation provides no value. It provides significant value as a defense-in-depth layer and as a mechanism for limiting blast radius. The limitation is that it cannot be the *only* mechanism for expressing access policy when the policy needs to be more fine-grained than "permitted on this segment" or "not."

---

## Authentication Is Not the Same as Authorization

A distinction that is frequently collapsed in OT access control discussions is the difference between authentication — verifying who or what is making a request — and authorization — determining what that principal is permitted to do. Network-centric trust models skip authentication at the device level entirely: a device on the trusted segment is presumed to be a legitimate segment member, and no further identity verification is performed. VPN and jump host patterns introduce authentication at the network access boundary — a user must authenticate to the VPN before gaining segment access — but do not extend authorization reasoning into the OT layer.

The consequence is that authentication and authorization are decoupled in a way that creates a structural gap. A user who authenticates successfully to a jump host is typically granted segment access, and from that point forward, the OT layer itself applies no further access control. The authentication event at the jump host boundary does not propagate into the OT systems. Controllers do not know who is issuing commands to them; they know only that the commands arrived from an address on their segment.

This gap has two distinct operational implications. The first is access control: the absence of per-action authorization means that any authenticated segment member can, in principle, issue any valid command. The second is auditability: because command sources are not attributed to authenticated identities, the audit record does not answer the question "who did what." It answers only "what happened," and even that only to the extent that individual controller logs are collected and retained — which is inconsistent across deployments.

Neither of these implications is an inherent deficiency in OT design. They are reasonable properties of a system that was not designed to attribute fine-grained actions to individual human principals. The deficiency emerges when fine-grained attribution and authorization are required — by operational governance requirements, compliance frameworks, or incident investigation needs — and the architecture cannot provide them.

---

## The Coarse Granularity of Controller-Local Access Control

Controller-local access control mechanisms — PIN-protected panels, password-protected configuration interfaces, proprietary access level schemes — provide a layer of protection against casual unauthorized access. They do not provide a foundation for a consistent authorization policy across a deployment.

The granularity of controller-local access control is typically coarse: a small number of access levels (view, operator, technician, administrator) that are fixed in the controller's firmware and applied uniformly to all users of that level. There is no mechanism for expressing that a user should have operator-level access to HVAC subsystems but only view-level access to the fire suppression controller. There is no mechanism for time-bounded access grants — a technician performing a maintenance activity cannot be given elevated access only for the duration of that activity. There is no mechanism for contextual conditions — an operator should be able to issue override commands during a declared emergency but not during normal operations.

These limitations are not arbitrary. Implementing fine-grained, context-aware access control at the controller level would require computational resources, configuration management tooling, and administrative infrastructure that most BAS controllers are not designed to support and most OT operational teams are not staffed to maintain. The coarse-grained model is the model that is actually operable in the environments where these controllers are deployed.

The limitation surfaces when operational requirements demand finer granularity than the controller can provide. Organizations that need to demonstrate to an auditor that a specific technician had access only to specific subsystems on specific dates cannot derive that assurance from controller-local access logs — the logs, where they exist, record accesses by access level, not by individual identity.

---

## Fragmented Authorization Enforcement

In a multi-vendor BAS deployment — which is the norm rather than the exception — authorization logic is distributed across a set of independent systems that do not share policy, do not share identity, and cannot be administered consistently as a whole. Each vendor's system enforces access controls according to its own model. Credentials for one system do not apply to another. Policy changes in one system have no effect on others. The operator who is a "technician" on the HVAC controller firmware is an entirely separate entity from the "operator" account on the lighting management system, even if both accounts belong to the same human.

This fragmentation creates several practical problems. The first is administrative overhead: maintaining access policies across independent systems requires maintaining separate credentials, separate role definitions, and separate lifecycle processes for each system. This is operationally feasible when the scale is small; it becomes increasingly difficult as the number of systems and operators grows. The de facto response to this overhead — creating a small number of shared service accounts used across multiple systems — reduces administrative burden at the cost of the access attribution that individual accounts would have provided.

The second problem is policy consistency. When authorization logic is distributed across independent systems, there is no mechanism to verify that the access policies across those systems are coherent with each other or with an overall organizational intent. A policy change intended to restrict a vendor's access during an off-boarding process must be executed separately on each system the vendor had access to. The probability of a gap in that execution increases with the number of systems involved.

The third problem is reasoning about authorization decisions. When asked whether a specific principal has access to a specific resource, there is no single place to look. The answer requires querying each system independently, interpreting the results against each system's own authorization model, and synthesizing an answer from heterogeneous data. This is tractable when the scope of inquiry is narrow; it becomes impractical when the scope spans an entire site.

---

## Inconsistent and Incomplete Auditability

The audit trail available from a typical OT deployment is partial, heterogeneous, and difficult to correlate. Controller-local logs, where they exist, record events at the granularity of the controller's own access level model — which, as noted, does not include individual operator identity in shared-credential environments. Supervisory system logs record what the supervisory system observed but do not capture events that bypassed it. Jump host logs record what sessions were established but do not record what was done within those sessions unless session recording is explicitly configured and maintained.

The practical consequence is that reconstructing a complete and authoritative picture of who did what across a BAS deployment — for an incident investigation, a compliance audit, or an operational review — requires collecting records from multiple sources with different schemas, different timestamps, and different retention policies, and attempting to correlate them against each other. The resulting picture is usually incomplete. Events that occurred through access paths not covered by any log source are invisible. Events logged by controller-local systems may not have been exported before the controller's internal log buffer was overwritten.

This is not a situation that arises from deliberate design decisions to evade accountability. It arises from the accumulation of individually reasonable decisions: each system logs what it can observe, at the granularity its model supports, for the retention period its storage allows. The problem is that these individually reasonable decisions do not aggregate into a coherent audit capability.

As audit and accountability requirements on OT environments have grown — through regulatory requirements, contractual obligations, and internal governance expectations — the gap between available audit data and required audit data has become a practical operational challenge. It is a challenge that cannot be resolved by adding more logging to individual systems; it requires a different approach to how authorization decisions are recorded in the first place.

---

## The Challenge of Scaling Distributed Authorization Logic

Authorization logic that is maintained independently in each system cannot be scaled consistently as the operational environment grows. Adding a new system means adding a new set of credentials to manage, a new set of access policies to maintain, and a new source of audit data that does not integrate with existing records. Onboarding a new operator means provisioning access in each system independently. Off-boarding an operator means revoking access in each system, with no automated mechanism to verify that revocation was complete.

This scaling challenge is not theoretical. Organizations managing large BAS deployments — multi-building campuses, hospital complexes, data center portfolios — routinely encounter situations where the overhead of maintaining consistent access policies across dozens of independent systems exceeds the capacity of their operational teams. The response is typically some combination of: broader access grants to reduce the number of systems that need to be managed per role, fewer and less specific access levels, infrequent policy review, and informal rather than formal access management processes.

None of these responses are unreasonable given the constraints. All of them represent a divergence between the access policies that operational governance requires and the access policies that are practically maintainable with the available tooling.

---

## Authorization as a System-Level Concern

The common thread through the limitations described above is this: they all arise from treating authorization as a protocol-level or component-level concern — something each device, controller, or system resolves independently using its own mechanisms — rather than as a system-level concern that requires shared infrastructure, shared policy, and shared observability.

The shift from protocol-level to system-level authorization is not primarily a technology change. It is a change in where authorization logic lives and what it has access to. In a protocol-level model, the controller that receives a command decides whether to execute it based on the access level of the credential presented to it. In a system-level model, a shared policy engine receives a structured description of the request — who is making it, what they are asking to do, which resource they are asking to act on — and evaluates it against a policy that applies consistently across the deployment.

The system-level model makes certain things possible that the protocol-level model cannot support: policies that reference organizational roles rather than per-controller access levels; authorization decisions that account for context that no individual controller can observe; audit records that capture decisions in a consistent format across all enforcement points; policy administration that happens in one place rather than in each system independently.

It also introduces requirements that the protocol-level model does not have: infrastructure to run the policy engine reliably, mechanisms to propagate identity context through multi-hop request paths, enforcement points at trust boundaries, and a clear separation between the components that evaluate policy and the components that enforce it.

These requirements are not trivial to satisfy in OT environments, and the following sections address them directly. The point of this section is to establish why satisfying them is worth the engineering effort: not because the existing approach is broken, but because the operational requirements it is being asked to meet have grown beyond what it was designed to support, and there is no path to meeting those requirements that stays entirely within the existing model.

---

## Summary

Traditional OT authorization mechanisms were designed for environments characterized by physical isolation, bounded operator populations, limited integration requirements, and protocol ecosystems that did not include authentication. They continue to function well within those constraints.

The constraints have changed. Increasing connectivity, remote access at scale, audit and accountability requirements, multi-system operational workflows, and vendor ecosystem complexity have introduced requirements that coarse-grained, network-centric, distributed authorization cannot satisfy: fine-grained access control that distinguishes between principals and actions, attribution of actions to individual identities, consistent policy across heterogeneous systems, centralized reasoning about authorization decisions, and a unified audit record that survives the fragmentation of the systems that generate it.

Addressing these requirements means treating authorization not as a property of individual protocols and devices but as a system-level concern with its own infrastructure, its own policy model, and its own observability. What that infrastructure looks like in an OT context — and what it requires from the surrounding system — is the subject of the following section.
