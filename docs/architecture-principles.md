# Architecture Principles

## Introduction

Architectural principles are not a substitute for design work. They do not resolve hard tradeoffs, and they cannot anticipate every operational condition an implementation will encounter. What they do is establish a shared reference point — a set of commitments about what matters most when tradeoffs arise and constraints compete.

The principles in this document reflect the specific challenges of building identity-aware authorization infrastructure for operational technology environments. OT systems carry constraints that differ meaningfully from enterprise IT: devices with decade-long lifecycles, protocols designed without authentication, environments where a misconfigured access control decision can have physical consequences, and operators who depend on system availability under adverse conditions.

These principles are intended to be stable enough to guide design decisions across implementation phases, and specific enough that they can be used to evaluate competing approaches. They are not aspirational statements about what the system will eventually become — they describe what the system is built to be.

---

## Principles

### 1. Explicit Authorization Over Implicit Trust

Access decisions must be based on verified identity and evaluated policy, not on assumptions derived from network location, protocol membership, or prior connectivity.

In traditional BAS environments, devices on the same network segment or communication bus are treated as mutually trusted. This model is a product of constrained hardware and protocols that predate networked threat models, not a deliberate security design. As BAS systems are increasingly interconnected — with enterprise networks, cloud platforms, and third-party service providers — the boundaries that made implicit trust workable have eroded.

Explicit authorization means that every access request carries verifiable identity context, and that access is granted only when a policy engine has evaluated that context against applicable policy and produced an `allow` decision. Silence from the policy engine, or the absence of an applicable policy, is not permission.

---

### 2. Identity Propagation Over Network-Origin Trust

When a request passes through multiple system components, the original requester's identity must be carried and verified at each hop rather than replaced by the identity of the intermediary.

In multi-tier architectures — where field devices communicate through protocol adapters, gateways, and supervisory systems before reaching a target — the downstream component typically sees only the identity of the immediate upstream caller. If that upstream caller is trusted by virtue of its network position rather than its identity, then the end-to-end authorization chain is only as strong as the weakest segment.

Identity propagation requires that authenticated identity context — in the form of signed tokens, attested headers, or verified session credentials — be carried through the request chain in a way that downstream components can verify. Intermediaries may add their own identity to the chain, but they must not silently substitute their identity for the original requester's.

---

### 3. Centralized Auditability

Authorization decisions, policy changes, and identity lifecycle events must be recorded in a way that is centrally accessible, consistently structured, and independent of the individual components that generated them.

Distributed systems produce distributed logs. When authorization events are recorded only at the component that evaluated them — without aggregation, normalization, or central retention — reconstructing a sequence of events across a multi-component request becomes impractical. Forensic investigation, compliance review, and incident response all depend on the ability to query across components and time ranges with confidence in record completeness.

Centralized auditability does not require a single monolithic log store. It requires that all security-relevant events be emitted in a consistent schema, transported through a reliable audit pipeline, and retained in a form that survives individual component failure. The audit record is a system property, not a component property.

---

### 4. Separation of Control Plane and Data Plane

Policy distribution, identity provisioning, and configuration management must be handled through infrastructure that is logically and, where feasible, physically separate from the infrastructure carrying operational workloads.

Conflating control plane and data plane functions creates coupling that degrades both security and operational clarity. A system that handles device telemetry, operator commands, and policy updates over the same communication path makes it harder to apply differentiated access controls, harder to audit control-plane activity distinctly from operational traffic, and harder to contain the impact of a data-plane compromise.

In practice, strict physical separation may not be achievable in all BAS deployments. The principle requires logical separation as a minimum: control plane communications must be identifiable, subject to distinct access controls, and audited independently of data plane traffic. Where physical separation is possible, it should be pursued.

---

### 5. Policy Evaluation Independent of Protocol

Authorization logic must be expressed and evaluated in terms of subjects, resources, and actions — not in terms of the underlying communication protocol through which a request arrived.

OT environments are protocol-heterogeneous. A single building may carry BACnet, Modbus, MQTT, and proprietary vendor protocols across its various subsystems. If authorization logic is embedded within each protocol adapter or controller, the result is a fragmented set of per-protocol access control implementations that cannot be audited consistently, cannot share policy, and will drift from each other over time.

Protocol-independent policy evaluation requires that protocol adapters normalize requests into a common representation before submitting them to the policy engine. The policy engine reasons about normalized subjects, resources, and actions — it does not interpret raw protocol frames. This separation allows the policy layer to remain stable as new protocols are introduced and allows protocol adapters to be replaced or updated without affecting the authorization model.

---

### 6. Operational Resilience First

The authorization infrastructure must be designed to support continued safe operation of controlled systems during partial failures, degraded connectivity, and planned or unplanned downtime of authorization services.

Authorization infrastructure that becomes a hard dependency for operational continuity introduces risk in an environment where availability is a safety requirement. A policy engine that is unreachable should not prevent an authorized operator from responding to a building emergency, nor should it cause a previously permitted device to lose access to a function it was actively using.

Resilience requires explicit design: local policy caches with defined staleness limits, credential validation that can proceed offline for a bounded period, and defined failsafe behaviors that specify what is permitted when central services are unavailable. Resilience mechanisms must themselves be tested under realistic failure conditions, not assumed to work because caching is present.

---

### 7. Fail-Safe and Operationally Predictable Behavior

When the system encounters an error, ambiguity, or condition outside its designed operating envelope, it must behave in a manner that is safe, consistent, and predictable to operators.

"Fail closed" — denying all access when a decision cannot be made — is the correct default for most security systems. In OT environments, this default must be qualified by operational context: a building access control system that fails closed during a fire evacuation is not a safe failure. The system must distinguish between security-relevant failures, where denial is correct, and operational emergencies, where defined break-glass or fail-safe modes must engage.

Predictable behavior means that operators and integrators can determine, from documented system behavior, what will happen under specific failure conditions before those conditions occur. Surprises in OT environments carry operational and safety risk. Every failure mode that affects authorization decisions must be documented and tested.

---

### 8. Least Privilege for Operational Actions

Subjects must be granted only the permissions necessary to perform their defined functions, scoped as narrowly as the operational model supports.

Least privilege is a well-established security principle, but its application in OT environments requires care. Operational roles in building systems often involve broad access to large sets of devices across a site, because physical work does not always decompose into the neat permission boundaries that software systems assume. Over-restricting access creates friction that operators work around through informal means, which is worse than a slightly broader permission scope applied consistently.

The practical goal is not the minimum theoretically expressible permission set — it is the minimum permission set that supports the actual operational workflow without requiring operators to circumvent access controls. This requires working from real operational patterns, not from abstract least-privilege ideals. Permissions should be reviewed as roles and workflows evolve.

---

### 9. Observable Authorization Decisions

Authorization decisions must be visible to operators and administrators in a form that supports understanding, troubleshooting, and verification without requiring access to internal system state.

A policy engine that produces authorization decisions without explaining them is difficult to operate and impossible to audit meaningfully. When an operator is denied access to a device they expect to reach, the system must provide enough information to determine whether the denial was correct — and if not, what needs to change. When an unexpected access pattern appears in the audit record, it must be possible to reconstruct the policy state that produced it.

Observability requires structured decision records that include the subject, resource, action, applicable policies, and the outcome. It also requires that policy evaluation logic be inspectable — that an administrator can determine, from the policy configuration, what outcome a given request will produce before submitting it. Opaque authorization decisions undermine operator confidence and complicate incident response.

---

### 10. Protocol Abstraction Where Feasible

The core identity and authorization infrastructure should interact with field-level device protocols through adapter components, rather than incorporating protocol-specific logic into shared services.

This principle is a practical consequence of OT protocol diversity. Protocols like BACnet, Modbus, LonWorks, and their variants each have distinct message structures, addressing schemes, and operational semantics. Encoding knowledge of these protocols into the policy engine, the audit pipeline, or the identity service creates maintenance burdens and limits portability.

Protocol adapters serve as the interface between the field protocol layer and the normalized representation used by shared services. Each adapter is responsible for message translation, identity attachment, and protocol lifecycle management. Shared services operate on normalized data. This boundary allows new protocols to be supported by adding adapters without modifying the authorization or audit infrastructure.

The qualifier "where feasible" acknowledges that some protocols are deeply integrated into controller firmware or legacy hardware that cannot be intermediated by an adapter. These cases require documented exceptions and compensating controls rather than forcing an abstraction that cannot be implemented.

---

### 11. Human Operators Remain Part of the System

The authorization model must account for the role of human operators — including emergency overrides, manual interventions, and break-glass procedures — as first-class operational patterns rather than exceptions to be suppressed.

Automated systems in OT environments do not operate independently of human oversight. Operators intervene during system anomalies, perform maintenance that requires temporary permission elevation, and must retain the ability to act during emergencies when automated authorization decisions may not be appropriate. A system that treats operator override as a security failure, or that provides no mechanism for emergency access, will be worked around.

Break-glass and override procedures must be defined, constrained, and audited. They should require authentication, produce audit records, and carry time limits where operationally appropriate. The goal is not to prevent operator intervention but to ensure it is visible, accountable, and reviewable after the fact.

---

### 12. Security Must Respect Operational Constraints

Security controls must be designed with an understanding of the operational environment they are being applied to, including device resource constraints, maintenance windows, protocol limitations, and human workflow patterns.

Security controls that are incompatible with operational requirements will not be adopted, or will be adopted in ways that defeat their purpose. An authentication mechanism that adds unacceptable latency to time-sensitive control loops will be disabled. A certificate rotation policy that requires device downtime during operating hours will produce long-running expired certificates. A complex policy configuration interface that requires specialized security expertise to operate will result in static, unmaintained policies.

This principle is not an argument for weaker security controls. It is an argument for security controls that are designed by people who understand how OT systems actually operate. Constraints are real inputs to security design, not obstacles to be dismissed.

---

### 13. Incremental Adoption Over Full Replacement

The architecture must support deployment in environments where legacy systems cannot be replaced and where new capabilities must coexist with existing infrastructure over extended transition periods.

BAS deployments are not greenfield projects. Real environments contain controllers, field devices, and protocol infrastructure with operational lifespans measured in decades. A security architecture that requires full replacement of existing systems as a prerequisite for adoption will not be adopted. The relevant design question is not "what does a fully ideal system look like" but "how does a system improve security in an environment with significant existing constraints."

Incremental adoption requires that new components interoperate with legacy systems, that the architecture supports partial deployment where some systems are identity-aware and others are not, and that the security posture of a partially deployed system is better than the baseline it replaces. Migration paths must be designed, not assumed.

---

### 14. Immutable Security-Relevant Event Logging

Records of authorization decisions, policy changes, identity lifecycle events, and security-relevant system activity must be written in a form that cannot be modified after the fact.

The value of an audit log depends on its integrity. A log that can be modified — whether by an attacker covering their tracks, an operator attempting to conceal an unauthorized action, or an inadvertent administrative change — cannot be relied upon for forensic investigation or compliance purposes. Immutability is a property of the logging infrastructure, not of the application that writes to it.

Immutable logging may be implemented through append-only storage backends, cryptographic chaining of log entries, write-once media, or out-of-band log replication to systems with separate administrative access. The implementation mechanism is less important than the property it provides: a credible assurance that the audit record reflects what actually occurred.

---

### 15. Identity Should Be Context-Aware

Authorization decisions should be able to incorporate contextual attributes — including device state, physical location, time of day, operational mode, and environmental conditions — not just static identity and role assignments.

Static identity and role assignments are a necessary foundation for authorization, but they are not sufficient for the operational complexity of building environments. The same operator may require different permissions during normal operations, a maintenance window, and an emergency response. The same device command may be appropriate from a controller on the local floor and inappropriate from the same controller during a fire alarm condition. Policies that cannot incorporate context produce access decisions that are either too broad or too restrictive to match operational reality.

Context-awareness requires that the policy evaluation model support attribute-based conditions alongside role assignments, that relevant context attributes are reliably available to the policy engine at evaluation time, and that context sources themselves are trustworthy. Context should extend the precision of authorization decisions — not serve as a mechanism for obscuring what access is actually permitted.
