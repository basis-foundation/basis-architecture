# Glossary

This glossary defines terminology used throughout the BASIS (Building Automation Secure Identity Service) architecture documentation. Terms are drawn from the domains of operational technology, identity and access management, and distributed systems security. Definitions are intended as engineering reference material — precise enough to support design decisions and implementation work, and consistent with the specific usage of each term within this project.

---

## Action

An operation that a subject requests to perform against a resource. In the context of authorization, actions are discrete, enumerable operations (e.g., `read`, `write`, `command`, `subscribe`) that a policy engine evaluates against applicable policies. The set of permitted actions for a given subject-resource pair is determined at authorization decision time.

---

## Attribute-Based Access Control (ABAC)

An authorization model in which access decisions are computed from structured attributes associated with subjects, resources, actions, and environmental context. ABAC enables fine-grained policy expression beyond what static role assignments can represent — for example, permitting a command only if the requesting device is within a specific floor zone and the current time falls within a maintenance window. ABAC policies are typically evaluated dynamically at request time.

---

## Audit Pipeline

The infrastructure responsible for capturing, transporting, and persisting authorization decisions and related system events for subsequent review or analysis. An audit pipeline typically includes structured event emission at enforcement points, a transport layer that delivers events reliably to storage, and a persistence layer that maintains records in a tamper-resistant or append-only form. The audit pipeline is distinct from the telemetry pipeline, though both may share transport infrastructure.

---

## Authorization Decision

The outcome produced by a policy engine in response to a policy evaluation request. An authorization decision is a discrete result — typically `allow`, `deny`, or `not applicable` — paired with the context in which it was reached (subject, resource, action, applicable policies, timestamp). Authorization decisions are the authoritative record of what was permitted or refused by the system and must be logged for auditability.

---

## Building Automation System (BAS)

A networked control system used to manage and monitor mechanical, electrical, and environmental infrastructure within a building or campus. Typical BAS domains include HVAC, lighting, access control, fire suppression, and power distribution. BAS components operate across a range of hardware generations and communication protocols, and are frequently deployed in environments where availability and operational continuity take precedence over frequent system updates.

---

## Centralized Authorization

An authorization architecture in which policy evaluation is performed by a dedicated, shared service rather than embedded independently within each individual component. Centralized authorization enables consistent policy enforcement across heterogeneous systems, simplifies auditing, and allows policies to be updated without modifying individual enforcement points. It introduces a dependency on the availability and performance of the authorization service and requires careful design for resilience and failover behavior.

---

## Control Plane

The logical layer of a system responsible for configuration, coordination, and management operations — as distinguished from the data plane, which carries operational workloads. In an IAM context, the control plane encompasses policy distribution, identity provisioning, certificate issuance, and system configuration. Control plane communications are typically lower-volume, higher-privilege operations and warrant stricter access controls and audit coverage than data plane traffic.

---

## Data Plane

The logical layer of a system responsible for carrying operational workloads — sensor readings, control commands, state updates, and similar runtime traffic. In a BAS context, the data plane includes the communications between field devices, controllers, and supervisory systems. Data plane interactions are subject to authorization enforcement but are generally higher-volume and more latency-sensitive than control plane operations.

---

## Device Identity

A verifiable credential or certificate that uniquely identifies a physical or virtual device within a system. Device identity is foundational to zero trust architectures: rather than trusting devices by virtue of their network location, systems authenticate device identity on each interaction. Device identities must be provisioned, rotated, and revoked through a defined lifecycle, and are typically bound to hardware where feasible to reduce credential portability risk.

---

## Edge System

A computing node deployed close to operational infrastructure — at the field level of a BAS, for example — that performs local processing, protocol translation, or enforcement functions. Edge systems may operate with intermittent connectivity to central services and must therefore support defined behavior during periods of isolation, including local policy caching and audit buffering. Edge systems represent a distinct trust zone from central infrastructure.

---

## Gateway

A system component that mediates communication between two or more network segments or protocol domains. In OT environments, gateways commonly bridge field-level device protocols (e.g., BACnet, Modbus) to higher-level infrastructure. A gateway may also serve as a policy enforcement point, applying authorization decisions to traffic passing through it. The gateway's position at a network boundary makes it a critical control point and a natural site for trust boundary enforcement.

---

## Identity-Aware Authorization

An authorization approach in which access decisions are conditioned on verified identity — of users, devices, or services — rather than on implicit trust derived from network location or protocol membership. Identity-aware authorization requires that subjects present authenticated credentials at the time of each request, and that the policy engine evaluate those credentials against applicable policy. This model is a prerequisite for consistent enforcement across heterogeneous, multi-protocol OT environments.

---

## Identity Propagation

The mechanism by which a subject's verified identity is carried through a multi-component request chain, so that intermediate and downstream services can make authorization decisions based on the original requester's identity rather than assuming trust from the previous hop. Identity propagation may be implemented through signed tokens, attested headers, or mutual TLS with forwarding semantics. Maintaining identity context across system boundaries is necessary for end-to-end auditability.

---

## Immutable Audit Logging

A logging discipline in which records of system events — particularly authorization decisions — are written in a form that cannot be modified or deleted after the fact. Immutability may be achieved through append-only storage, cryptographic chaining, or write-once media. Immutable audit logs provide a trustworthy record for forensic investigation, compliance verification, and incident response, and their integrity must be protected as a security property of the system.

---

## Operational Resilience

The capacity of a system to continue providing its defined functions — including authorization enforcement — in the presence of component failures, degraded connectivity, or partial service unavailability. In OT environments, operational resilience is a primary design constraint: field devices and controllers must remain functional even when central services are unreachable. Resilience mechanisms in an IAM system include local policy caching, cached credential validation, and defined failsafe modes that preserve safety without bypassing all access controls.

---

## Operational Technology (OT)

Hardware and software systems that monitor and control physical processes, devices, and infrastructure. OT encompasses industrial control systems, building automation systems, programmable logic controllers (PLCs), remote terminal units (RTUs), and related field equipment. OT environments are characterized by long asset lifecycles, constrained compute resources, protocol heterogeneity, and operational requirements that prioritize availability and safety over features typical of enterprise IT systems.

---

## Operator Identity

A verifiable credential that uniquely identifies a human operator interacting with a system. Operator identity is distinct from device identity and typically involves stronger authentication mechanisms (e.g., multi-factor authentication) appropriate for human principals. In an OT context, operator identities must account for shift-based access patterns, break-glass emergency procedures, and role assignments that may vary by site, system, or operational state.

---

## Policy Engine

The component responsible for evaluating authorization requests against a defined set of policies and producing authorization decisions. A policy engine receives a structured request — containing subject identity, resource descriptor, requested action, and relevant context — and applies applicable policy rules to determine the outcome. Policy engines may support multiple policy languages or models (e.g., RBAC, ABAC) and must produce consistent, auditable decisions.

---

## Policy Evaluation

The process by which a policy engine applies relevant policies to a specific authorization request. Policy evaluation takes a structured input (subject, resource, action, context), identifies which policies are applicable, executes the policy logic, and returns an authorization decision. The evaluation process must be deterministic, auditable, and performant enough for the latency requirements of the systems it serves.

---

## Protocol Adapter

A software component that translates between a field-level device protocol (such as BACnet/IP, Modbus TCP, or MQTT) and the internal representation used by the authorization or telemetry infrastructure. Protocol adapters decouple the policy enforcement and audit systems from the specifics of device communication protocols, allowing the core identity and authorization logic to remain protocol-agnostic. Each adapter is responsible for normalizing device messages and, where applicable, attaching or verifying identity context.

---

## Protocol-Centric Security

A security model in which access control and trust decisions are based primarily on membership in a shared communication protocol or network segment, rather than on verified identity. Protocol-centric security is characteristic of legacy OT environments, where devices on the same BACnet or Modbus segment are implicitly trusted. This model is inadequate for environments with heterogeneous protocols, cross-segment communication, or threats that originate from within the network perimeter.

---

## Resource

The target of an authorization request — the system object, device point, data stream, configuration parameter, or service endpoint to which a subject is requesting access. Resources are described by structured identifiers that capture the information necessary for policy evaluation, such as the resource type, owning system, physical location, or sensitivity classification. Resource descriptors must be consistent and unambiguous across the systems that reference them.

---

## Role-Based Access Control (RBAC)

An authorization model in which permissions are assigned to roles, and subjects are granted access by virtue of their role memberships. RBAC simplifies permission administration in environments where access patterns map cleanly onto defined job functions or organizational responsibilities. In OT contexts, RBAC provides a practical baseline for operator access control, though it is often extended with attribute-based conditions to handle the operational complexity of building environments.

---

## Subject

The entity — user, device, service, or automated process — that initiates an authorization request. Subjects must present verifiable identity credentials as part of each request. The subject is a fundamental input to policy evaluation: policies specify what actions subjects with given characteristics are permitted to perform on given resources.

---

## Telemetry

Structured data collected from devices and system components that describes operational state, behavior, and events over time. Telemetry in a BAS context includes sensor readings, device health indicators, communication statistics, and control state. Telemetry data is distinct from audit log data, though both may flow through shared transport infrastructure. Telemetry supports situational awareness, anomaly detection, and operational monitoring.

---

## Telemetry Ingestion

The process of receiving, parsing, normalizing, and routing telemetry data from source devices or adapters into storage or processing systems. Telemetry ingestion infrastructure must handle the volume, variability, and protocol diversity characteristic of OT environments. Ingestion pipelines are responsible for attaching metadata (such as device identity and timestamp) and routing data to appropriate consumers, including monitoring systems and anomaly detection components.

---

## Trust Boundary

A defined demarcation between two system domains with different trust levels, where identity verification and authorization enforcement are applied to traffic crossing the boundary. Trust boundaries are explicit design constructs, not implicit properties of network topology. In a BAS environment, trust boundaries may exist between the field device layer and the supervisory layer, between building networks and enterprise networks, or between a local edge system and central cloud infrastructure. Each trust boundary requires defined enforcement mechanisms.

---

## Zero Trust

A security model premised on the assumption that no subject, device, or network segment should be implicitly trusted based on its location or prior access. Under zero trust, every access request is authenticated and authorized independently, regardless of whether the requester is inside or outside a conventional network perimeter. Zero trust is an architectural principle rather than a specific technology; its application to OT environments requires careful adaptation to account for constrained devices, legacy protocols, and availability requirements.
