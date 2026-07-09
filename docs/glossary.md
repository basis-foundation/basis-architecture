# Glossary

This glossary defines terminology used throughout the BASIS architecture documentation. Terms are drawn from the domains of operational technology, identity and access management, distributed systems security, and the BASIS ecosystem itself. Definitions are intended as engineering reference material — precise enough to support design decisions and implementation work, and consistent with the specific usage of each term within this project.

---

## Authorization Kernel

The isolated, minimal component responsible for policy evaluation semantics, enforcement contracts, and audit event definitions in the BASIS ecosystem. The authorization kernel — implemented as basis-core — owns the stable logic that all other components in the distribution depend on for determining whether a request is permitted. It does not host a runtime API, operate protocol adapters, provide a user interface, or depend on cloud infrastructure or identity providers. See also: **BASIS Core Services Distribution**, **basis-core**.

---

## BASIS Core Services Distribution

The open-source, deployable set of components maintained under Basis Foundation governance. The distribution includes basis-core (the authorization kernel), basis-gateway (the API and runtime wrapper), basis-console (the operator and administrative UI), basis-adapters (protocol normalization adapters), basis-identity (the identity engine and federation boundary), basis-deploy (deployment and distribution tooling), and basis-schemas (shared schemas and compatibility definitions). The distribution is designed to be complete enough for real deployments without requiring commercial services from BASAuth. See also: **Basis Foundation**, **BASAuth**.

---

## Basis Foundation

The nonprofit open-source body responsible for governing the BASIS Core Services Distribution, maintaining the architectural standards and design principles of the BASIS ecosystem, and stewarding the open-source repositories under the BASIS namespace. The Foundation is not a commercial entity and does not operate managed services. See also: **BASIS Core Services Distribution**, **BASAuth**.

---

## BASAuth

The future for-profit commercial company that builds enterprise products and managed services on top of the BASIS Core Services Distribution. BASAuth commercial offerings address managed hosting, enterprise support, fleet management, advanced policy workflows, audit and compliance tooling, adapter certification, enterprise integrations, and related operational services. BASAuth does not own or modify the open-source kernel or the core distribution components. See also: **BASIS Core Services Distribution**, **Basis Foundation**.

---

## Administrative Interface

A human-facing surface for submitting configuration changes, reviewing system state, and initiating operational requests through established enforcement boundaries. In the BASIS ecosystem, `basis-console` is the administrative interface for the authorization system. An administrative interface must not bypass the enforcement boundaries it interacts through — it submits requests; those boundaries authenticate and enforce them. See also: **Console**, **Operator Workflow**, **basis-console**.

---

## basis-core

The isolated authorization kernel in the BASIS Core Services Distribution. basis-core implements the policy evaluation logic, enforcement semantics, failure mode contracts, and audit event schema that all other distribution components depend on. basis-core must not depend on basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, identity providers, cloud platform SDKs, or UI frameworks. See also: **Authorization Kernel**, **BASIS Core Services Distribution**, **basis-identity**.

---

## basis-identity

The identity engine and federation boundary in the BASIS Core Services Distribution. basis-identity integrates external, authoritative identity providers — Keycloak, Okta, Entra ID, Auth0, Ping, ADFS, LDAP, and other SAML/OIDC sources — without replacing them, and brokers their identities into the ecosystem. Depending on deployment configuration it may act as a SAML Service Provider or OIDC relying party, own login/logout/callback flows, manage BASIS-local sessions, exchange tokens, map provider claims, resolve subjects, produce canonical BASIS identity context, and issue BASIS-local tokens for downstream components. Its architectural role mirrors an identity broker (such as Keycloak) deployed in front of applications: external IdPs remain authoritative; basis-identity presents a single normalized identity context downstream. A deployment chooses one primary identity authority mode — federated, synchronized registry, or standalone / air-gapped local authority. basis-identity is not an authoritative enterprise identity provider by default and does not own the enterprise user directory, except in the explicitly configured standalone / air-gapped mode where it acts as the deployment-local identity authority; in no mode does it evaluate authorization, store policy, or normalize OT protocols. See also: **Identity Engine**, **Identity Federation**, **Identity Authority Mode**, **Canonical Identity Context**, **BASIS Core Services Distribution**.

---

## Action

An operation that a subject requests to perform against a resource. In the context of authorization, actions are discrete, enumerable operations that a policy engine evaluates against applicable policies. BASIS recognizes five canonical action verbs — `read`, `write`, `execute`, `browse`, `subscribe` — which compose with a domain into the `{verb}:{domain}[:{object}]` action-name form (e.g. `read:hvac:setpoint`). See [`docs/architecture/action-vocabulary.md`](architecture/action-vocabulary.md). The set of permitted actions for a given subject-resource pair is determined at authorization decision time.

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

## Console

The human-facing operational interface for observing, configuring, and interacting with BASIS components through established enforcement boundaries. In the BASIS ecosystem, `basis-console` is the console component. The console provides operators with visibility into policy state, authorization decisions, and audit activity, and provides interaction paths for submitting changes and initiating administrative operations. The console is not an authorization engine, identity provider, protocol adapter, or deployment platform. It is an interface layer: it surfaces what the authorization system has done and submits requests through channels the authorization system enforces. See also: **Administrative Interface**, **Operator Workflow**, **basis-console**.

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

## Identity Engine

The component that integrates external identity providers and normalizes externally-authenticated identity into the single canonical form a system's downstream components consume. In the BASIS ecosystem, `basis-identity` is the identity engine: it brokers external IdPs, owns the federation-related login/session surface, maps claims, resolves subjects, and produces canonical BASIS identity context. An identity engine is distinct from an authoritative identity provider — it integrates and normalizes identity rather than originating accounts or credentials. See also: **Identity Federation**, **Canonical Identity Context**, **basis-identity**.

---

## Identity Federation

The arrangement in which a system accepts identities authenticated by one or more external, authoritative identity providers rather than maintaining its own authoritative directory, typically by acting as a SAML Service Provider or OIDC relying party toward those providers. A federation boundary brokers external identities into a normalized internal representation and may own the login, logout, and callback flows federation requires. In the BASIS ecosystem, federation is owned by `basis-identity`; the external enterprise IdPs (Keycloak, Okta, Entra ID, Auth0, Ping, ADFS, LDAP, and others) remain authoritative. See also: **Identity Engine**, **basis-identity**.

---

## Canonical Identity Context

The normalized, provider-independent representation of a subject's identity that BASIS components evaluate and propagate — principally the `Subject` and `IdentityContext` inputs to `basis-core`. Canonical identity context is the single internal form of identity in the ecosystem: external provider claims and assertions are mapped into it by `basis-identity`, and `basis-gateway` validates and carries it into kernel evaluation. The kernel depends on this normalized form and never on identity-provider-specific artifacts. The shape of the canonical identity context is expected to be formalized under a future `basis-schemas`. See also: **Identity Engine**, **Identity Propagation**, **basis-identity**.

---

## Identity Authority Mode

The deployment-level choice of *where the authoritative record of identity lives* for a BASIS deployment. `basis-identity` may support multiple authority modes, but each deployment selects one primary mode: **Federated Mode**, **Synchronized Registry Mode**, or **Local Identity Authority** (standalone / air-gapped). The authority mode is distinct from the federation surface a deployment operates — it concerns who originates accounts, not how much login and session machinery `basis-identity` runs — and all modes converge into the same canonical identity context downstream. Defined in [`docs/architecture/identity-authority-modes.md`](architecture/identity-authority-modes.md). See also: **Federated Mode**, **Synchronized Registry Mode**, **Local Identity Authority**, **Canonical Identity Context**.

---

## Federated Mode

The identity authority mode in which an external IdP (Okta, Entra ID, Keycloak, Auth0, Ping, ADFS, or other SAML/OIDC source) is authoritative for accounts, credentials, MFA, and lifecycle, and `basis-identity` brokers and normalizes identity from it without originating accounts. `basis-identity` may keep shadow/profile records derived from the authoritative provider for diagnostics, mapping, access review, and audit context. This is the primary, default enterprise mode. See also: **Identity Authority Mode**, **Identity Federation**, **Synchronized Registry Mode**.

---

## Synchronized Registry Mode

The identity authority mode in which an external IdP remains authoritative but `basis-identity` maintains a local synchronized registry — a maintained local copy of users, groups, external identifiers, mapped roles, lifecycle state, and sync metadata — kept in step with the authoritative source via SCIM push, periodic import, or controlled offline bundle import. The registry supports diagnostics, access review, mapping, and limited offline resilience, and must not silently diverge from the authoritative source. See also: **Identity Authority Mode**, **Federated Mode**, **Local Identity Authority**.

---

## Local Identity Authority

The identity authority mode (also called standalone or air-gapped mode) in which `basis-identity` is the *deployment-local* authoritative source of identity, owning local users, groups, credential verification, sessions, token issuance, and lifecycle state. It exists for OT environments where no external IdP is reachable. This authority is deployment-local only and is an explicit, constrained operating mode — not a general-purpose enterprise IdP replacement and not the default. See also: **Identity Authority Mode**, **Federated Mode**, **Synchronized Registry Mode**.

---

## Immutable Audit Logging

A logging discipline in which records of system events — particularly authorization decisions — are written in a form that cannot be modified or deleted after the fact. Immutability may be achieved through append-only storage, cryptographic chaining, or write-once media. Immutable audit logs provide a trustworthy record for forensic investigation, compliance verification, and incident response, and their integrity must be protected as a security property of the system.

---

## Operation-Aware Authorization Model

The conceptual expansion of `basis-core` from a basic subject/action/resource authorization kernel into a kernel that evaluates richer OT operation context — subject attributes, resource type, site/zone, device identity and class, protocol operation evidence, operation intent, safety/risk/environment/time context, and correlation metadata — while remaining protocol-agnostic, identity-provider-agnostic, and enforcement-agnostic. The model is defined in [`docs/architecture/operation-aware-authorization-model.md`](architecture/operation-aware-authorization-model.md) and adopted in ADR-0001. It defines conceptual categories for a future, richer `DecisionRequest`/`DecisionResponse`, not a final schema or a policy language. See also: **Authorization Kernel**, **Policy Evaluation**, **basis-core**.

---

## Operation-Aware Evaluation Semantics

The set of deterministic rules governing how `basis-core` reasons about the richer context described by the **Operation-Aware Authorization Model** — default deny, `NOT_APPLICABLE` behavior, deny precedence, conflict resolution, rule ordering, condition evaluation, missing context behavior, unknown action/resource behavior, schema version compatibility, deterministic trace output, and safe error handling. These semantics are defined in [`docs/architecture/operation-aware-evaluation-semantics.md`](architecture/operation-aware-evaluation-semantics.md) and adopted in ADR-0002. They define conceptual evaluation behavior, not a final policy language, JSON Schema, or trace schema. See also: **Operation-Aware Authorization Model**, **Policy Evaluation**, **Authorization Decision**.

---

## Operation-Aware Policy Bundle and Rule Model

The conceptual model defining how policy is packaged, identified, validated, and evaluated for the **Operation-Aware Authorization Model** — the **Policy Bundle** as the unit of policy distribution, versioning, validation, and compatibility; policy scope; the rule as a deterministic unit of evaluation with an effect, match criteria, conditions, a reason code, and traceable metadata; `ALLOW`/`DENY` rule effects and their relationship to deny precedence; bundle and rule combining semantics; rule ordering; policy validation; and policy metadata and provenance. The model is defined in [`docs/architecture/operation-aware-policy-rule-model.md`](architecture/operation-aware-policy-rule-model.md) and adopted in ADR-0004. It defines conceptual policy model categories and ownership, not a final policy language, JSON Schema, or reason-code vocabulary. See also: **Operation-Aware Authorization Model**, **Operation-Aware Evaluation Semantics**, **Policy Bundle**, **Policy Engine**, **Policy Evaluation**.

---

## Operation-Aware Trace and Audit Evidence

The conceptual model defining what evidence must exist to make an operation-aware decision explainable, auditable, testable, and safe to visualize — including the distinction between kernel-produced **evaluation trace**, durable **audit evidence**, and the runtime **gateway audit event**; the evidence lifecycle from identity context through console visualization; conceptual categories for trace, rule-level evidence, request evidence, adapter evidence references, and identity evidence references; redaction tiers; and reason codes. The model is defined in [`docs/architecture/operation-aware-trace-audit-evidence.md`](architecture/operation-aware-trace-audit-evidence.md) and adopted in ADR-0003. It defines conceptual evidence categories and ownership, not a final trace schema, audit event schema, or reason-code vocabulary. See also: **Operation-Aware Authorization Model**, **Operation-Aware Evaluation Semantics**, **Immutable Audit Logging**.

---

## Operator Workflow

A structured sequence of human-initiated actions through which an operator interacts with the authorization system to accomplish a defined operational objective — reviewing policy state, examining audit records, submitting a configuration change, or investigating a denial. In the BASIS ecosystem, operator workflows are mediated by `basis-console` and flow through `basis-gateway`; they do not bypass enforcement boundaries or reach `basis-core` directly in production deployments. See also: **Console**, **Administrative Interface**.

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

## Policy Bundle

The conceptual unit of policy distribution, versioning, validation, and compatibility in the **Operation-Aware Policy Bundle and Rule Model** — the thing `basis-core` evaluates a `DecisionRequest` against and the thing `basis-gateway` loads and selects at runtime. A policy bundle carries a bundle identifier, bundle version, schema version, policy owner/authority, target scope, rule set, metadata, compatibility metadata, validation status, provenance, and optional deprecation/replacement metadata. A policy bundle is not necessarily a file format; concrete serializations are a future `basis-schemas` decision. Defined in [`docs/architecture/operation-aware-policy-rule-model.md`](architecture/operation-aware-policy-rule-model.md) and adopted in ADR-0004. See also: **Operation-Aware Policy Bundle and Rule Model**, **Policy Engine**, **Policy Evaluation**.

---

## Policy Engine

The component responsible for evaluating authorization requests against a defined set of policies and producing authorization decisions. A policy engine receives a structured request — containing subject identity, resource descriptor, requested action, and relevant context — and applies applicable policy rules to determine the outcome. Policy engines may support multiple policy languages or models (e.g., RBAC, ABAC) and must produce consistent, auditable decisions.

---

## Policy Evaluation

The process by which a policy engine applies relevant policies to a specific authorization request. Policy evaluation takes a structured input (subject, resource, action, context), identifies which policies are applicable, executes the policy logic, and returns an authorization decision. The evaluation process must be deterministic, auditable, and performant enough for the latency requirements of the systems it serves.

---

## Adapter

A protocol-specific translation component that converts external operational semantics into BASIS authorization semantics. An adapter parses protocol-native messages, normalizes them into `DecisionRequest` objects that `basis-core` can evaluate, and serializes authorization decisions back into protocol-native responses. Adapters are the only components in the BASIS ecosystem that contain protocol-specific logic. See also: **Protocol Normalization**, **Trusted Adapter Boundary**, **basis-adapters**.

---

## Protocol Adapter

A software component that translates between a field-level device protocol (such as BACnet/IP, Modbus TCP, or MQTT) and the internal representation used by the authorization or telemetry infrastructure. Protocol adapters decouple the policy enforcement and audit systems from the specifics of device communication protocols, allowing the core identity and authorization logic to remain protocol-agnostic. Each adapter is responsible for normalizing device messages and, where applicable, attaching or verifying identity context.

---

## Protocol Normalization

The process of converting protocol-specific operations into protocol-independent authorization semantics. Normalization is the core responsibility of a BASIS adapter: the adapter understands the protocol-native operation and translates it into the subject-resource-action vocabulary that `basis-core` evaluates. After normalization, the originating protocol is no longer visible to the authorization kernel. See also: **Adapter**, **basis-adapters**.

---

## Protocol-Centric Security

A security model in which access control and trust decisions are based primarily on membership in a shared communication protocol or network segment, rather than on verified identity. Protocol-centric security is characteristic of legacy OT environments, where devices on the same BACnet or Modbus segment are implicitly trusted. This model is inadequate for environments with heterogeneous protocols, cross-segment communication, or threats that originate from within the network perimeter.

---

## Resource

The target of an authorization request — the system object, device point, data stream, configuration parameter, or service endpoint to which a subject is requesting access. Resources are described by structured identifiers that capture the information necessary for policy evaluation, such as the resource type, owning system, physical location, or sensitivity classification. Resource descriptors must be consistent and unambiguous across the systems that reference them. See also: **Resource Identifier**, **Resource Type**, **Canonical Resource Identifier**, **Resource Identifier Composition**.

---

## Resource Identifier

The string that names the **Resource** a request targets. In the BASIS authorization kernel, the canonical resource identifier is a single typed, structured value in `{type}:{qualifier[:{subqualifier}...]}` form (e.g. `hvac:zone-a`, `sensor:co2:lobby`). The kernel derives the resource type from the identifier's prefix rather than from a separate field. A resource identifier may be absent for resource-independent requests. See also: **Canonical Resource Identifier**, **Local Resource Identifier**, **Resource Type**, [`docs/architecture/resource-identifier-reconciliation.md`](architecture/resource-identifier-reconciliation.md).

---

## Resource Type

The classification of a **Resource** by operational category — for example `hvac`, `sensor`, `zone`, `device`, or `gateway`. In the canonical kernel form, the resource type is carried as the prefix of the **Canonical Resource Identifier** and is derived from it during policy evaluation and audit; it is not supplied as an independent input to kernel evaluation. Upstream components (protocol adapters, and the console form) may carry resource type as a separate `resource_type` field as a normalization input, which the composition boundary folds into the canonical identifier. The resource-type vocabulary is not yet unified across components and is expected to be governed by a future `basis-schemas`. See also: **Resource Identifier**, **Resource Identifier Composition**.

---

## Local Resource Identifier

A resource identifier as produced upstream of the composition boundary, before the resource type has been folded into it — for example the `rooftop-1` rendered by a protocol adapter's `resource_id_template`, carried alongside a separate `resource_type` of `ahu`. A local resource identifier is a normalization input, not a canonical authorization artifact; on its own it does not satisfy the kernel's resource-identifier format. See also: **Canonical Resource Identifier**, **Resource Identifier Composition**.

---

## Canonical Resource Identifier

The single typed, structured **Resource Identifier** that the policy engine evaluates and the audit record stores: `{type}:{qualifier[:{subqualifier}...]}` (e.g. `ahu:rooftop-1`). It is "canonical" in that it is the one form the kernel accepts and the form that is stable across policy and audit over time. A **Local Resource Identifier** plus a separate `resource_type` becomes a canonical resource identifier through **Resource Identifier Composition**. See also: **Resource Identifier**, **Resource Identifier Composition**.

---

## Resource Identifier Composition

The operation that combines a separate `resource_type` and a **Local Resource Identifier** into a **Canonical Resource Identifier** (`ahu` + `rooftop-1` → `ahu:rooftop-1`). The recommended owner of this operation is `basis-gateway`, the same boundary that composes the canonical action — so that the action's domain and the resource's type prefix are produced from the same input and remain consistent. Adapters do not perform kernel-specific identifier composition; the kernel does not infer resource type from a separate supplied field. See [`docs/architecture/resource-identifier-reconciliation.md`](architecture/resource-identifier-reconciliation.md). See also: **Action** (whose composite name is composed at the same boundary), **Canonical Resource Identifier**.

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

## Trusted Adapter Boundary

The trust position occupied by a BASIS adapter: trusted to parse protocol requests accurately, normalize protocol intent faithfully, preserve operational semantics, and serialize responses correctly; not trusted to authorize, grant permissions, define policy, or override kernel decisions. An adapter that accepts untrusted protocol input and invokes `basis-core` directly becomes an enforcement boundary and must protect that boundary accordingly. See also: **Adapter**, **Trust Boundary**.

---

## Trust Boundary

A defined demarcation between two system domains with different trust levels, where identity verification and authorization enforcement are applied to traffic crossing the boundary. Trust boundaries are explicit design constructs, not implicit properties of network topology. In a BAS environment, trust boundaries may exist between the field device layer and the supervisory layer, between building networks and enterprise networks, or between a local edge system and central cloud infrastructure. Each trust boundary requires defined enforcement mechanisms.

---

## Zero Trust

A security model premised on the assumption that no subject, device, or network segment should be implicitly trusted based on its location or prior access. Under zero trust, every access request is authenticated and authorized independently, regardless of whether the requester is inside or outside a conventional network perimeter. Zero trust is an architectural principle rather than a specific technology; its application to OT environments requires careful adaptation to account for constrained devices, legacy protocols, and availability requirements.
