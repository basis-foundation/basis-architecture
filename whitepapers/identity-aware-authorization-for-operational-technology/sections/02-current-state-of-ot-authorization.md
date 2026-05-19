# Current State of OT Authorization

Understanding how authorization is handled in operational technology environments today requires understanding how those environments were built. Most OT deployments are not the product of a single design decision made at a single point in time. They are the accumulated result of incremental expansion, vendor-specific integration, protocol standardization efforts that spanned decades, and operational constraints that shaped every architectural choice along the way. Any credible assessment of where OT authorization stands must begin with that context.

---

## Network Segmentation as the Primary Defense

The most widely deployed authorization model in OT environments is not, strictly speaking, an authorization model at all. It is a network architecture: the separation of OT systems from corporate IT networks and external connectivity through VLANs, firewalls, and in some deployments, physical air gaps.

This approach is grounded in a coherent principle. If OT devices communicate only with other OT devices on the same segment, and that segment has no external connectivity, then unauthorized access from outside the segment is structurally constrained. The attack surface is defined by physical access and by the limited number of permitted crossing points. For environments where those conditions hold, network segmentation provides meaningful protection without requiring that individual devices support any authentication or authorization capability.

The challenge is that these conditions are increasingly difficult to maintain. Building automation systems that once operated in isolation are now expected to share data with energy management platforms, facility maintenance systems, and cloud-hosted analytics services. Remote access for vendor support, previously handled through dedicated modems or physical on-site visits, is now routed through shared VPN infrastructure. Integration points that were once exceptions have become routine operational requirements. The perimeter that network segmentation was designed to defend has expanded significantly, and with it, the assumptions on which the model was built.

Network segmentation remains a necessary component of OT security architecture. The question is not whether to segment, but whether segmentation alone is sufficient as the primary mechanism for expressing and enforcing access decisions.

---

## VLAN-Based Trust and Its Operational Logic

VLAN-based trust models assign authorization implicitly based on network membership. A device on the building automation VLAN is trusted to communicate with other devices on the same VLAN; traffic from outside that VLAN is filtered at the network boundary. No per-device authentication is required; no individual action is evaluated against a policy. Trust is a property of network position.

This model has genuine operational advantages. It is operationally simple to administer, requires no capability from the end devices, and can be applied uniformly regardless of whether individual devices support any authentication mechanism. For environments dominated by legacy hardware with no authentication support — which describes a significant portion of the installed BAS base — VLAN-based trust may be the only access control mechanism that is practically deployable across the full device population.

The limitations of VLAN-based trust emerge most clearly when the implicit assumption of uniform device trust within a segment is violated. If a compromised or misconfigured device gains access to the BAS VLAN, the network model offers no mechanism to constrain what that device can do once inside. Similarly, VLAN membership provides no basis for differentiating between a legitimate controller issuing a routine command and the same type of device issuing an anomalous one. The model provides a boundary at the segment edge, but no discrimination within the segment.

---

## VPN and Jump Host Access Patterns

Remote access to OT environments is most commonly implemented through VPN concentrators and jump hosts — intermediary systems through which remote operators or vendors connect before accessing OT components. The jump host serves as a controlled access point: authentication occurs at the jump host boundary, and the sessions that originate from it inherit whatever trust the jump host carries on the OT network.

This pattern reflects a practical tradeoff. Jump hosts can be administered, patched, and monitored more readily than OT endpoints. Concentrating remote access through a small number of controlled intermediaries reduces the attack surface compared to allowing direct external connections to OT devices. Multi-factor authentication can be enforced at the jump host boundary even when OT devices themselves cannot participate in modern authentication schemes.

What the jump host model does not address is the granularity of access once a session is established. A vendor authenticated to a jump host typically receives the same network access as any other authenticated user, regardless of which specific device or function they need to reach. Session-level audit coverage depends on what is recorded at the jump host itself, and the recording of actions taken within an OT session varies considerably between deployments. The model provides a meaningful perimeter control, while leaving questions of per-action authorization and comprehensive auditability largely unresolved.

---

## Controller-Local Authorization

Many OT controllers implement some form of local access control. Typical mechanisms include password-protected configuration interfaces, PIN-based operator panels, and proprietary access level schemes built into controller firmware. These controls are intended to prevent unauthorized configuration changes and to differentiate between read-only monitoring access and write access to control points.

Controller-local authorization functions independently of any centralized identity system. Each controller manages its own credentials, its own access levels, and its own audit log — if an audit log exists at all. In a large building automation deployment, this means that access controls for hundreds or thousands of individual devices are administered separately, with no shared policy, no common credential store, and no unified view of who has access to what.

This fragmentation is not a design failure — it reflects the era in which these systems were built and the operational model they were designed to support. Controllers were historically administered by specialist technicians with direct physical access, not by IT administrators managing access policy at scale. The assumption was that physical access to the controller was itself a meaningful access control, and that the credential mechanisms on the device served as a second layer rather than the primary one.

The operational consequence is that credential management across a large OT environment requires either a sustained administrative discipline that many organizations cannot maintain, or a de facto collapse to a small number of shared credentials that are rarely changed. Neither outcome is the result of negligence; both are predictable responses to the administrative overhead of managing distributed, uncoordinated authorization systems.

---

## Shared Credentials and Operational Workflow

Shared operator credentials are common in OT environments. A single username and password may be used by multiple technicians, across multiple shifts, for access to a given controller type or system. This is frequently not a policy failure but a reflection of how operational workflows are structured.

Maintenance and operational tasks in building systems often involve teams rather than individuals. A shift change may involve multiple technicians taking over active sessions or resuming work that was begun by a previous shift. Controller access is frequently managed by the department or team that owns a system rather than by individual identity. Adding per-operator credentials to devices that were not designed for per-operator identity management introduces administrative burden that may not be feasible to sustain given available staffing and tooling.

The audit consequence is significant: where shared credentials are in use, log entries record the credential rather than the individual. Attributing an action in an audit log to a specific operator becomes impossible without out-of-band records such as physical access logs or work order documentation, which may not be maintained with sufficient granularity.

---

## Vendor-Specific Trust Models

OT environments commonly involve multiple vendors, each with their own access model for their respective equipment. A building automation deployment might include controllers from one manufacturer using a proprietary network protocol, a fire system from a second manufacturer with its own access control scheme, a lighting system from a third using a different authentication mechanism, and a supervisory platform from a fourth that aggregates data from all of them.

Each vendor implements authorization logic according to its own conventions. Credential formats, access level definitions, audit log schemas, and remote access mechanisms vary between systems. An organization managing a multi-vendor environment cannot apply consistent access policy or produce a unified audit view without additional tooling that sits above the individual vendor systems.

Vendor-specific trust models are also shaped by the practical realities of OT maintenance. Vendors typically require administrative-level access to their own systems for firmware updates, calibration, and support. The mechanisms through which this access is granted — vendor accounts, shared service credentials, remote access protocols specific to the vendor's toolchain — are often separate from whatever access controls the organization applies to its own operators.

---

## Protocol-Centric Trust and Its Origins

Several widely deployed building automation protocols — BACnet, Modbus, and LonWorks among them — were designed in an era when the assumption of a closed, physically secured network was reasonable. These protocols do not include built-in authentication. A device on a BACnet network that sends a valid BACnet message to another device is trusted by virtue of sending a syntactically correct message on the correct network segment.

This is not an oversight in the conventional sense. The protocol designers were addressing the problem in front of them, which was reliable device communication in resource-constrained, physically bounded environments. Authentication mechanisms require compute resources, add latency, and introduce complexity that is difficult to justify when the network is a dedicated closed bus with five devices on it.

The protocols remained in use — and continue to be deployed in new installations — because they work reliably for their intended purpose, because enormous amounts of installed infrastructure depend on them, and because the alternative of requiring authentication on every message on a building automation network is not straightforward to retrofit without disrupting existing deployments.

---

## Fragmented Audit Trails

Audit coverage in OT environments reflects the fragmentation of authorization systems. Where controller-local logs exist, they cover only the controller's own events. Where supervisory systems log events, coverage is limited to what the supervisory system can observe. Network-layer logging, where deployed, captures traffic metadata but typically not application-layer semantics. No unified audit record exists across the full system.

The practical consequence is that reconstructing a complete sequence of events across a multi-component BAS — in the context of an incident investigation, a compliance review, or an operational anomaly — requires collecting and correlating records from multiple systems with different schemas, different timestamps, and different levels of completeness. This is feasible when specific systems are implicated and the scope is limited; it becomes substantially more difficult when the scope of investigation is broad or the events in question are distributed across many components.

---

## Operational Uptime and the Constraints It Creates

Any discussion of OT authorization must acknowledge the asymmetry between security and availability in operational environments. In most OT contexts, unplanned downtime carries consequences that are direct and visible: physical processes are disrupted, occupants are affected, and recovery may require manual intervention that is time-consuming and costly.

This asymmetry shapes architectural decisions in ways that are rational from an operational perspective. Authorization mechanisms that introduce latency into control loops are genuinely problematic in time-sensitive systems. Changes to access control infrastructure that require controller downtime may be scheduled infrequently because the window for downtime is narrow and competing demands on that window are many. Mechanisms that could interrupt communication between a controller and its supervised devices are evaluated conservatively.

Security improvements in OT environments are not blocked by organizational indifference. They are constrained by operational realities that place a high value on stability and predictability. Incremental, non-disruptive approaches are not just preferred — they are often the only approaches that can be adopted at all.

---

## Air-Gap Assumptions and Their Erosion

The assumption that OT networks are or can be maintained as air gaps — fully isolated from external networks — has historically shaped both the design of OT systems and the assessment of their risk profile. A system that is truly air-gapped does not require the same level of device-level authentication as one with external connectivity because the population of potential actors is defined by physical access.

In practice, the air gap assumption has eroded considerably in most building automation environments. Integration with energy management systems, enterprise building management platforms, and cloud analytics services typically requires some form of data connection between OT and IT networks. Remote access for support and maintenance provides another. The assumption of isolation persists in some operational documentation and organizational security models even in environments where it no longer accurately describes the network architecture.

This gap between assumed and actual network architecture is a meaningful factor in OT security posture. It does not imply that organizations have been negligent; it reflects how integration requirements have expanded incrementally, often without corresponding updates to the security model.

---

## Summary

The authorization landscape in OT environments is the product of decades of incremental development, protocol evolution, and operational constraint. Network segmentation, VLAN-based trust, controller-local access controls, and shared credentials are not artifacts of poor security judgment — they are solutions to real problems, implemented with the tools and assumptions available at the time, in environments where operational continuity carries significant weight.

What has changed is the operational context in which these mechanisms function. Increasing connectivity, broader vendor ecosystems, expanded remote access, and more stringent audit and compliance requirements are placing demands on OT authorization infrastructure that was not designed to meet them. The challenge of addressing these demands is not primarily technical — the concepts of identity-aware authorization are well understood. The challenge is doing so in ways that are compatible with the operational realities of OT environments: the legacy devices that cannot be replaced, the protocols that cannot be changed, the maintenance windows that are narrow, and the availability requirements that are uncompromising.
