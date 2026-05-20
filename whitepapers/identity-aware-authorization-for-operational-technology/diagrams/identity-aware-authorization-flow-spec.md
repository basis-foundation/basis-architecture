# Diagram Specification: Identity-Aware Authorization Flow

**Proposed file names:**
- Mermaid source: `identity-aware-authorization-flow.mmd`
- Draw.io source: `identity-aware-authorization-flow.drawio` *(to be produced)*
- Exported image: `../exported/identity-aware-authorization-flow.png` *(to be produced)*

**Referenced in:** Section 04 — Identity-Aware OT Architecture

---

## 1. Diagram Purpose and Scope

This diagram is the primary architectural illustration for the white paper. Its purpose is to make the relationships between the authorization model's components visible in a single view — showing how identity, policy evaluation, enforcement, operational action, and auditability interact across the trust boundaries of an OT environment.

The diagram is conceptual, not implementation-specific. It depicts a generalized architecture, not the BASIS proof-of-concept implementation specifically. Component names correspond to architectural roles, not specific software products.

The diagram should answer the following questions for a reader approaching it without having read the full section:

- Where does authorization happen, and how many times?
- What is the path of an operator command from initiation to execution?
- What separates the control plane from the data plane?
- Where do audit events originate, and where do they go?
- What is the role of the protocol adapter layer?
- How does the architecture support operation when central services are unreachable?

A diagram that answers all of these questions without accompanying text has succeeded. A diagram that requires extensive prose to interpret has not.

---

## 2. Trust Boundaries

Four trust zones are represented in the diagram. Each is enclosed in a dashed border with a label. The dashed border is the visual convention for trust boundaries throughout this paper (see `docs/diagrams/README.md`).

### Operations Zone

Contains the human-facing access layer: operator workstations, technician workstations, and the jump host. This zone is where authenticated sessions originate. It is adjacent to but distinct from the OT environment — operators work in this zone; they do not work in the edge or field zones during normal operations.

**Why this boundary exists:** The operations zone is administratively controlled separately from the OT field infrastructure. It is subject to IT-facing security controls (endpoint management, MFA enforcement, session recording) that do not apply inside the edge zone. The trust boundary marks the point at which session identity is established and at which the first authorization decision is made (at the jump host).

### Edge / Gateway Zone

Contains the OT gateway, protocol adapters, the local enforcement point, and the local policy cache. This zone mediates all traffic between the operations zone and the field device zone. It is the architectural location for protocol normalization and the second authorization enforcement point.

**Why this boundary exists:** The edge zone operates at a different trust level than the operations zone. Systems in the edge zone have direct protocol-level access to field devices; systems in the operations zone do not. The trust boundary marks where field-protocol semantics become relevant and where the enforcement point closest to field devices is positioned. It also marks where the local policy cache makes offline authorization possible — the edge zone can continue to enforce authorization decisions for field-device traffic even when central services are unreachable.

### Field Device Zone

Contains BAS controllers and their connected field hardware. Devices in this zone communicate over field protocols (BACnet, Modbus) that have no native authentication or identity mechanism. They respond to commands that arrive from the edge zone after authorization has been applied at the edge boundary.

**Why this boundary exists:** Field devices operate under fundamentally different trust assumptions than the edge and operations zones. They cannot participate in the identity model directly. The trust boundary marks the limit of the enforcement architecture — authorization of traffic destined for field devices is enforced at the edge zone boundary, not at the devices themselves. What enters this zone has already been authorized.

### Central Services Zone

Contains the identity provider, policy engine, audit log store, and telemetry pipeline. This zone is control-plane infrastructure — it does not carry operational workloads and is not in the path of real-time device commands. It is the authoritative source of identity verification and policy decisions, and the central sink for audit events.

**Why this boundary exists:** Separating identity and policy infrastructure into its own zone — with its own administrative access controls, its own network position, and its own failure characteristics — is a prerequisite for meaningful security governance. If the policy engine and the OT operational systems share the same administrative domain, the ability to audit and constrain that domain is limited. Administrative separation is itself a security control.

---

## 3. Component Responsibilities

### Identity Provider

Issues and validates credentials for all principals in the system — operators, technicians, devices enrolled in the identity model, and service accounts. The identity provider is the root of the authentication chain. It does not make authorization decisions; it verifies identity and provides verified identity context to the policy engine.

### Policy Engine

Receives structured authorization requests — subject identity, resource descriptor, requested action, relevant context — and evaluates them against policy to produce decisions. The policy engine is the authoritative source of authorization outcomes. It distributes policy to local policy caches at enforcement points and receives authorization requests from enforcement points in real time or near-real time. It generates decision records that are sent to the audit log.

### Local Policy Cache

Holds a time-bounded copy of policy applicable to the traffic the local enforcement point governs. It is populated by the policy engine via periodic synchronization. When the policy engine is unreachable, the local enforcement point evaluates requests against the cache, allowing continued authorization without real-time central connectivity. The cache has a defined staleness limit beyond which its policy should not be applied without re-synchronization.

### Local Enforcement Point

Intercepts traffic crossing the edge zone boundary, constructs authorization requests from available context, submits them to the policy engine (or evaluates against the local cache), applies the resulting decision, and emits audit events recording the decision and its outcome. It is the enforcement mechanism for field-device-destined traffic. It is positioned in the edge zone because that is the last point at which an operator command can be evaluated before reaching field devices.

### OT Gateway

Receives authenticated sessions from the operations zone and routes traffic to the appropriate protocol adapter and enforcement path within the edge zone. The gateway is the ingress point for operations-zone traffic into the edge zone. It does not perform authorization itself — it routes traffic to the enforcement point.

### Protocol Adapters (BACnet, Modbus)

Normalize field-protocol messages into the shared representation used by the authorization model: structured descriptions of the resource being accessed and the action being requested. The adapter extracts the relevant information from the field-protocol frame, attaches identity context derived from the session in which the request arrived, and produces the authorization request that the enforcement point submits for evaluation. On the data-plane side, the adapter translates authorized commands into field-protocol format for delivery to the controller. It also collects telemetry from controllers and normalizes it for the telemetry pipeline.

### Jump Host

The ingress point for remote operator and technician access to the OT environment. Authentication is enforced here. The jump host is also an enforcement point: it applies authorization decisions before establishing sessions into the OT environment. It submits authorization requests to the policy engine and emits audit events for each session established or denied.

### BAS Controllers

Respond to field-protocol commands received from the protocol adapters. Controllers have no direct role in the authorization model — they receive commands that have already been authorized at the edge zone boundary and execute them. Their telemetry output is collected by the protocol adapters and routed to the telemetry pipeline.

### Audit Log Store

Receives structured authorization decision records from enforcement points (jump host, local enforcement point) and from the policy engine. Stores records in append-only form. Is administratively separate from the OT operational systems. Does not participate in real-time authorization decisions — it is a sink, not a component in the evaluation path.

### Telemetry Pipeline

Receives normalized device telemetry from protocol adapters. Routes telemetry to monitoring, analytics, and storage consumers. Is distinct from the audit pipeline: the telemetry pipeline carries operational state data; the audit pipeline carries security-relevant decision records. They may share transport infrastructure but should be treated as logically separate.

---

## 4. Flow Semantics

### Data Plane Flows (solid arrows)

Data plane flows carry operational workloads: operator commands propagating toward field devices, device telemetry propagating toward the telemetry pipeline, sensor readings flowing between sensors and controllers. These flows are the operational purpose of the system — the reason the rest of the architecture exists.

**Operator command flow (downward):**
`Operator Workstation → Jump Host → OT Gateway → Local Enforcement Point → Protocol Adapter → BAS Controller → Field Devices`

The command originates at the operator workstation, is authenticated and authorized at the jump host, passes through the OT gateway into the edge zone, is again authorized at the local enforcement point (with field-device-level context), is translated by the protocol adapter into the appropriate field protocol, and is delivered to the controller.

Authorization occurs twice: at the operations zone boundary (jump host) and at the edge zone boundary (local enforcement point). The two authorization decisions address different aspects of the request — the jump host enforces that the operator is permitted to initiate a session into the OT environment; the local enforcement point enforces that the specific command to the specific resource is permitted for the authenticated operator.

**Telemetry flow (upward):**
`Sensors / Actuators → BAS Controller → Protocol Adapter → Telemetry Pipeline`

Telemetry does not traverse the enforcement point in the authorization-enforcement sense — it is device data flowing to the telemetry pipeline, not a command requesting action on a resource. It is normalized by the protocol adapter for the telemetry pipeline. Telemetry source authentication (verifying that telemetry originates from enrolled devices) is a separate concern that is noted in the threat modeling section but not represented in the primary flow.

### Control Plane Flows (dashed arrows)

Control plane flows carry authorization infrastructure: policy distribution, authorization requests and decisions, credential validation, identity context. These flows do not carry operational workloads. They are the mechanism by which the authorization model stays consistent and by which enforcement points have the information they need to make correct decisions.

**Policy distribution:**
`Policy Engine → Local Policy Cache`

The policy engine periodically pushes policy updates to local policy caches at enforcement points. The cache holds the current applicable policy. This synchronization is what enables the local enforcement point to make authorization decisions without a round-trip to the policy engine on every request, and to continue operating if the policy engine becomes temporarily unreachable.

**Authorization request / decision:**
`Local Enforcement Point → Policy Engine → Local Enforcement Point` (and equivalently for the Jump Host)

When an enforcement point intercepts a request, it submits an authorization request to the policy engine. The policy engine evaluates the request and returns a decision. The enforcement point applies the decision. For time-sensitive requests, local cache evaluation is used instead, foregoing the round-trip.

**Identity context:**
`Identity Provider → Policy Engine`

The policy engine relies on the identity provider to validate credentials presented by subjects. The identity provider supplies verified identity context — confirmed attributes about the subject — that the policy engine incorporates into its evaluation.

**Credential validation:**
`Identity Provider → Jump Host`

The jump host validates operator credentials at the session ingress point. It queries the identity provider to verify the presented credential and receive the authenticated identity context that will be propagated into the OT session.

### Audit Flows

Audit flows originate at enforcement points and at the policy engine and terminate at the audit log store. They carry structured records of authorization decisions: who, what, which resource, which action, which policy, what decision, when.

Audit flows from enforcement points to the audit log store use the same arrow style as data plane flows in the Mermaid diagram for visual clarity. In the Draw.io version, audit flows should use a distinct style (dotted line, green color per the diagram standards) to distinguish them from operational data flows.

**Enforcement point audit events:**
`Jump Host → Audit Log Store` and `Local Enforcement Point → Audit Log Store`

Each enforcement point emits an audit event for every authorization decision it applies — both permitted and denied. The event records the authenticated identity, the resource, the action, the decision, and the policy version in effect.

**Policy engine decision records:**
`Policy Engine → Audit Log Store`

The policy engine records each evaluation it performs, independent of whether the requesting enforcement point also records the event. This provides a secondary record that can be used to detect discrepancies between what was evaluated and what was enforced.

---

## 5. Authorization Semantics

The diagram communicates three key authorization semantics that should be visible without explanation:

**Authorization occurs before operational action.** Enforcement points sit upstream of the operational components they protect. No arrow from an enforcement point to a downstream operational component appears unless an authorization decision has been applied. This visual ordering — enforcement point, then the downstream component — represents the temporal ordering of the actual system: evaluate, decide, then execute or reject.

**Enforcement is distributed; policy logic is centralized.** Enforcement points appear in multiple zones (jump host in the operations zone, local enforcement point in the edge zone). But both enforcement points connect to the same policy engine. The distribution of enforcement does not imply distribution of policy. This is a key architectural claim of the paper, and the diagram should make it visually obvious.

**Authorization is independent of operational protocol.** The protocol adapters sit between the enforcement point and the field-protocol controllers. The enforcement point evaluates authorization using the normalized representation produced by the adapters — it does not interact with BACnet or Modbus directly. This visual separation of the enforcement point from the protocol-specific components represents the protocol-independence of the authorization model.

---

## 6. Control Plane and Data Plane Separation

The diagram separates control plane and data plane flows using arrow style: solid for data plane, dashed for control plane. This visual distinction is the mechanism for communicating the architectural principle that operational traffic and authorization infrastructure traffic are distinct concerns that travel different paths.

The Central Services zone is architecturally a control-plane zone. The data plane does not traverse it. Operator commands do not pass through the policy engine or identity provider — they are evaluated *by* those components at enforcement points, but the traffic itself does not route through central services. This distinction — that the policy engine evaluates requests but does not carry traffic — is a core property of the architecture and should be visible in the diagram.

The absence of solid arrows into the Central Services zone (all flows into that zone are dashed control-plane flows or solid audit flows) is the visual representation of this property.

---

## 7. Offline Operation Visibility

The local policy cache is included as a named component in the edge zone. Its presence in the diagram serves two purposes: it explains how the local enforcement point can make authorization decisions when the policy engine is unreachable, and it makes the offline resilience property of the architecture visible without requiring prose.

The dashed arrow from the policy engine to the local policy cache (policy distribution) and the dashed arrow from the local policy cache to the local enforcement point (cached policy) together communicate the offline authorization mechanism. A reader who notices these flows and asks "why does the enforcement point have two sources of policy?" has understood the resilience design.

---

## 8. Diagram Conventions

This diagram follows the standards defined in `docs/diagrams/README.md`. The following specific conventions apply:

| Element | Mermaid convention | Visual meaning |
|---|---|---|
| Trust boundary | Subgraph with dashed border | Enclosing trust zone |
| Human principal | `(["Label"])` stadium shape | Operator or technician |
| Storage component | `[("Label")]` cylinder shape | Audit store, telemetry pipeline |
| Enforcement point | Rectangular node with `[AuthZ]` label | Site where authorization is applied |
| Data plane flow | Solid arrow `-->` | Operational traffic |
| Control plane flow | Dashed arrow `-.->` | Policy, identity, authorization |
| Audit flow | Solid arrow `-->` (Mermaid); dotted green (Draw.io) | Audit event to store |

The `[AuthZ]` marker appears on the Jump Host and the Local Enforcement Point — the two nodes where authorization decisions are applied. It does not appear on the Policy Engine (which evaluates but does not enforce), on the OT Gateway (which routes but does not enforce), or on the Protocol Adapters (which normalize but do not enforce).

---

## 9. Rationale for Major Flows

**Why is there a jump host in addition to an OT gateway?**
The jump host and the OT gateway serve different functions. The jump host is the enforcement point for the operations-zone boundary: it authenticates operators and makes a coarse-grained authorization decision about whether to admit the session at all. The OT gateway routes authenticated sessions into the edge zone but does not itself enforce authorization. This separation reflects the real architecture of remote-access OT environments, where the session ingress point and the OT-network entry point are distinct components.

**Why does the local enforcement point appear between the OT gateway and the protocol adapters?**
The local enforcement point is positioned after the OT gateway (which provides routing) and before the protocol adapters (which provide translation). This positioning reflects the architectural principle that authorization should be evaluated after the request has been received from the authenticated session but before it is translated into field-protocol format and delivered to the controller. The enforcement point evaluates using the normalized representation that the adapter produces from the incoming request.

**Why do both the jump host and the local enforcement point connect to the policy engine?**
Both are enforcement points making distinct authorization decisions at distinct points in the request flow. The jump host enforces operations-zone-boundary access. The local enforcement point enforces field-device-level access. They both rely on the same policy engine because the policy that governs both decisions is maintained in the same place — this is the centralized policy logic property of the architecture.

**Why does the policy engine connect to the audit log store?**
The policy engine records its own evaluation decisions independent of the enforcement point records. This provides an independent audit trail that can be used to detect discrepancies — for example, if an enforcement point fails to emit an audit event for a decision it applied, or if an enforcement point's behavior diverges from the decision the policy engine reports having made.

**Why is there no direct connection between the operations zone and the field device zone?**
The absence of a direct connection is intentional. All traffic from the operations zone to the field device zone passes through the edge zone and its enforcement point. This visual constraint represents the architectural requirement that field devices are not directly accessible from the operations zone — all access is mediated by the gateway and enforcement infrastructure.

---

## 10. Draw.io Version Guidance

The Mermaid diagram is the working reference version for repository rendering. For publication-quality output suitable for PDF export, a Draw.io version should be produced with the following refinements:

**Layout:** Use a strict top-to-bottom zone stack with the Central Services zone at the top. Central Services should be visually positioned as a separate zone connected to the operational zones via control-plane flows, not as part of the operational stack. Consider placing the Central Services zone to the right of the operational stack, with control-plane flows running horizontally between them.

**Trust boundary borders:** Dashed enclosing rectangles for each zone. Apply the background fill colors from the diagram standards: light orange for the Central Services zone, light yellow for enforcement points within each zone, blue-gray for OT/field components.

**Arrow routing:** Orthogonal routing. Use distinct line weights for data plane (heavier) versus control plane (lighter, dashed). Group audit flows into a single color (green, per diagram standards) so they are visually traceable to the audit log store.

**Control plane lane:** Add a vertical lane on the right side of the diagram for control-plane flows between the policy engine and the enforcement points. Label the lane "Control Plane." This makes the separation between control plane and data plane visually explicit rather than requiring the reader to track arrow styles.

**Enforcement point badges:** Add a small `[AuthZ]` badge icon to the jump host and local enforcement point nodes. Keep the badge at 8pt or smaller to avoid visual clutter.

**Offline annotation:** Add a small callout or annotation near the local policy cache explaining its role in offline operation: "Enables continued enforcement when policy engine is unreachable."

---

## Pre-Commit Checklist

- [ ] All components are labeled
- [ ] All arrows carry descriptive labels
- [ ] All trust boundaries are marked with dashed borders and zone labels
- [ ] Jump Host and Local Enforcement Point carry `[AuthZ]` markers
- [ ] Control plane and data plane flows are visually distinguishable
- [ ] Audit event flows are traceable from both enforcement points and policy engine to the audit log store
- [ ] Central Services zone has no solid/data-plane arrow inbound (only dashed control-plane and audit flows)
- [ ] No direct connection exists between Operations Zone and Field Device Zone
- [ ] Local Policy Cache is present and its connection to the enforcement point is visible
- [ ] Diagram is legible in grayscale
- [ ] Source file is committed alongside any exported image
- [ ] Alt-text is written for any Markdown image reference
