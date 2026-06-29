# BASIS Ecosystem

This document describes the structure of the BASIS ecosystem: the organizational bodies, software components, and commercial layers that make up the broader project. It is intended as a canonical reference for contributors and readers who need to understand how the pieces relate to each other, what each component is responsible for, and where boundaries between them are drawn.

---

## The Three Layers

The BASIS ecosystem consists of three distinct layers: the Basis Foundation, the BASIS Core Services Distribution, and BASAuth. These are not names for the same thing at different stages — they represent different kinds of entities with different roles, different scopes, and different governing authorities.

---

## Basis Foundation

The Basis Foundation is the nonprofit open-source body responsible for the architectural standards, governance model, and long-term stewardship of the BASIS project. It is the organizational home for the open-source work.

**What the Foundation does:**

- Defines and maintains the architectural standards and design principles that govern the BASIS ecosystem
- Governs the schema, protocol, and compatibility definitions that allow components to interoperate
- Stewards the open-source repositories under the BASIS namespace
- Manages the contribution and review process for core components
- Maintains the reference documentation, white papers, and architecture decision records that define the intended system behavior
- Defines what belongs in the open-source distribution and what belongs outside it

The Foundation does not build commercial products and does not operate managed services. Its work is architectural, definitional, and organizational.

---

## BASIS Core Services Distribution

The BASIS Core Services Distribution is the open-source, deployable set of components maintained under Foundation governance. Together, these components provide everything needed to run a basic, functional identity-aware authorization system for OT environments. The distribution is intentionally complete enough to be genuinely useful — it is not artificially limited to force commercial dependency.

### Components

**basis-core** — the isolated authorization kernel

The authorization kernel is the stable, minimal core of the distribution. It implements the policy evaluation engine, the canonical authorization decision semantics, enforcement behavior definitions, failure mode contracts, and the audit event schema. basis-core is the component that other services in the distribution depend on.

basis-core is defined by what it must not contain as much as by what it does contain. It does not operate an identity provider, run a protocol adapter, host an API runtime, provide a user interface, manage deployment infrastructure, or depend on cloud platform services. These boundaries are not incidental — they are the reason basis-core can remain stable, testable, and portable across the deployment contexts that OT environments require.

**basis-gateway** — API and runtime wrapper

The API gateway wraps the authorization kernel and exposes it as a runtime service. It handles the request lifecycle: receiving authorization requests, invoking basis-core for policy evaluation, returning decisions to callers, and emitting audit events. basis-gateway is the component that enforcement points and external services interact with directly. It depends on basis-core.

**basis-console** — operator and administrator UI

The console provides an operator and administrator interface for the authorization system. It supports policy inspection, authorization decision review, audit log querying, and basic operational management. It depends on basis-gateway. It does not contain authorization logic of its own. See [`docs/architecture/basis-console.md`](basis-console.md) for the canonical console architecture reference.

**basis-adapters** — protocol and integration adapters

The adapter library provides the normalization layer between field-level OT protocols and the subject-resource-action representation that basis-core evaluates. Each adapter handles a specific protocol family (BACnet, Modbus, MQTT, and others) and is responsible for translating protocol-specific messages into the shared authorization vocabulary and for delivering authorized commands back to the protocol layer. basis-adapters depends on basis-core for its authorization contracts and event schemas. See [`docs/architecture/basis-adapters.md`](basis-adapters.md) for the canonical adapter architecture reference.

**basis-identity** — identity engine and federation boundary

The identity engine is the federation and normalization boundary between external identity systems and the BASIS authorization runtime. It integrates external, authoritative identity providers — Keycloak, Okta, Entra ID, Auth0, Ping, ADFS, LDAP, and other SAML/OIDC sources — without replacing them, and brokers their identities into the ecosystem. Depending on deployment configuration it may act as a SAML Service Provider or OIDC relying party, own the login/logout/callback flows, manage BASIS-local sessions, normalize provider claims into canonical BASIS identity context, and issue BASIS-local tokens that downstream components can trust. Its architectural role mirrors how an identity broker such as Keycloak is deployed in front of applications: external enterprise IdPs remain authoritative, while basis-identity presents downstream BASIS components — principally basis-gateway — with a single normalized identity context. A deployment chooses one primary identity authority mode — federated, synchronized registry, or standalone / air-gapped local authority — and all modes converge into the same canonical identity context downstream; see [`docs/architecture/identity-authority-modes.md`](identity-authority-modes.md). basis-identity is not an authoritative enterprise identity provider by default and does not own the enterprise user directory, except in the explicitly configured standalone / air-gapped mode where no external IdP is reachable and it acts as the deployment-local identity authority; in no mode does it evaluate authorization. See [`docs/architecture/basis-identity.md`](basis-identity.md) for the canonical identity engine architecture reference.

**basis-deploy** — deployment and distribution tooling

The deployment component provides tooling for packaging, configuring, and distributing the BASIS Core Services Distribution in OT environments. This includes container definitions, configuration management tooling, and deployment validation scripts. basis-deploy is not part of the authorization runtime — it is the tooling used to stand one up.

**basis-schemas** — shared schemas, contracts, and compatibility definitions

The schemas component defines the shared data contracts used across the distribution: the authorization request and response schemas, audit event schemas, policy format definitions, and compatibility specifications for interoperability between components. basis-schemas is a foundational dependency — all other components reference it for their shared data definitions.

### What the distribution is designed to support

A deployment using only the BASIS Core Services Distribution should be able to:

- Evaluate authorization requests against a defined policy using the kernel
- Receive and process those requests through the API gateway
- Normalize field-protocol messages into the authorization vocabulary through protocol adapters
- Record authorization decisions and audit events in a structured, queryable form
- Provide operators with basic visibility into policy state and authorization activity through the console
- Package and deploy the above components into an OT environment using the deployment tooling

This is a complete authorization system for deployments that do not require commercial managed services, advanced fleet operations, or enterprise-specific integrations.

---

## BASAuth

BASAuth is the future for-profit commercial company that builds enterprise-grade products and services on top of the BASIS Core Services Distribution. BASAuth is not part of the Foundation and does not govern the open-source components — it is a commercial entity that depends on the open-source work as its foundation.

### What BASAuth provides

BASAuth's commercial offerings address the production engineering challenges that open-source components intentionally leave to deployment operators. These include:

- **Managed hosting** — operated, SLA-backed BASIS infrastructure for organizations that cannot or do not want to self-host
- **Enterprise support and SLA** — support contracts, incident response, and upgrade assistance
- **Fleet management** — tooling and services for managing large, distributed deployments of BASIS components across many sites
- **Advanced policy workflows** — policy authoring, review, approval, and rollback tooling appropriate for enterprise governance processes
- **Advanced audit and compliance tooling** — audit aggregation, retention management, compliance reporting, and forensic query interfaces beyond what the open-source console provides
- **Managed identity federation** — operated, SLA-backed federation services that build on the open-source basis-identity engine: hosted federation for connecting enterprise identity providers and vendor identity systems to BASIS-governed enforcement infrastructure, beyond what self-hosted basis-identity provides
- **Adapter certification** — tested, validated, and supported adapter implementations for specific device manufacturers and firmware versions
- **Enterprise integrations** — integrations with enterprise asset management systems, ITSM platforms, CMDB systems, and operational data historians
- **Multi-site operations** — services for managing policy distribution, consistency monitoring, and synchronization state across large numbers of geographically distributed sites
- **Upgrade orchestration** — managed upgrade tooling for OT environments with constrained maintenance windows and change management requirements
- **Observability and analytics** — authorization traffic analytics, anomaly detection, and operational dashboards beyond the standard audit pipeline

BASAuth does not own or modify the open-source kernel or the core distribution components. Its commercial services are layered above them.

---

## Component Dependency Direction

The dependency relationships between components in the ecosystem follow a single direction: higher-level services depend on lower-level ones; lower-level components do not depend on higher-level ones.

```text
Commercial services (BASAuth)
    depends on
Open-source services (basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy)
    depends on
Authorization kernel (basis-core)
    depends on
Shared schemas (basis-schemas)
```

`basis-identity` is an exception to the "depends on basis-core" arrow: it sits upstream of evaluation and does not depend on the kernel. It produces the canonical identity context that `basis-gateway` validates before invoking `basis-core`; identity integration and normalization happen before, not inside, authorization. It depends on `basis-schemas` for the shape of that canonical identity context.

The rules that enforce this structure:

1. **Commercial services may depend on open-source BASIS services.** BASAuth products may integrate with basis-core, basis-gateway, basis-adapters, and other open-source components. They must not be required for the open-source distribution to function.

2. **Open-source services may depend on basis-core.** basis-gateway, basis-console, basis-adapters, and basis-deploy may call into basis-core, reference its contracts, and extend its behavior at defined extension points. They must not modify core evaluation semantics or bypass its enforcement contracts. basis-identity is upstream of evaluation and does not depend on basis-core; it produces canonical identity context that basis-gateway consumes.

3. **basis-core must not depend on commercial services or higher-level runtime services.** basis-core must not import from basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, or any BASAuth component. It must not depend on cloud platform SDKs, specific identity providers, database runtimes, UI frameworks, or protocol stacks.

This rule is not a convention — it is a structural property of the kernel. Any dependency that crosses upward from basis-core toward higher-level services violates the isolation that makes the kernel stable, testable, and portable.

---

## What Belongs in basis-core

basis-core owns the stable, protocol-agnostic authorization semantics that all higher-level services depend on:

- **Normalized decision semantics** — the canonical meaning of `allow`, `deny`, and `not applicable` decisions, including the conditions under which each applies
- **Policy evaluation behavior** — the evaluation logic that takes a subject, resource, action, and policy set as inputs and produces a deterministic authorization decision
- **Enforcement semantics** — the contracts that enforcement points must satisfy: what they must evaluate, what they must record, and how they must behave when evaluation cannot complete
- **Failure mode contracts** — the defined behavior for conditions where evaluation cannot proceed normally, including the distinction between fail-closed and fail-open semantics and the conditions under which each applies
- **Audit event contracts** — the canonical schema for authorization decision events: the required fields, their types, their semantic meaning, and the conditions under which each event must be emitted

---

## What Must Stay Outside basis-core

The following concerns belong in higher-level components or commercial services, not in basis-core:

- **Identity provider operation and federation** — integrating external identity providers, brokering and federating identity, owning login/logout/callback flows, managing sessions, mapping claims, and issuing BASIS-local tokens belong to basis-identity (and to the external IdPs that remain authoritative); the kernel stays identity-provider, token, and session agnostic and never participates in federation
- **Protocol adapter implementation** — the logic for normalizing BACnet, Modbus, MQTT, or any other field protocol belongs in basis-adapters, not in the kernel
- **Runtime API hosting** — the HTTP API surface, request routing, and session management belong in basis-gateway
- **User interface** — operator consoles, administrative dashboards, and policy authoring interfaces belong in basis-console or commercial tooling
- **Telemetry pipelines** — OT telemetry collection, normalization, and routing are operational data concerns that sit outside the authorization kernel
- **Deployment orchestration** — container management, configuration distribution, and upgrade tooling belong in basis-deploy or commercial deployment services
- **Cloud infrastructure** — cloud platform SDKs, managed database clients, object storage clients, and cloud-specific service integrations must not enter basis-core
- **High-availability topology** — clustering, load balancing, and failover design are deployment concerns, not kernel concerns; basis-core must be deployable in HA configurations without encoding HA behavior in the kernel itself
- **Policy distribution services** — the mechanisms for distributing policy from a central engine to remote enforcement points belong in basis-gateway or higher-level deployment services
- **Adapter certification and validation** — testing and certifying adapter implementations against specific device types is a commercial service concern

---

## Repository and Component Boundaries

Each component in the BASIS Core Services Distribution is expected to be maintained in its own repository under the Basis Foundation organization:

| Repository | Component | Role |
| - | - | - |
| `basis-core` | Authorization kernel | Policy evaluation, enforcement semantics, audit contracts |
| `basis-gateway` | API and runtime wrapper | Request handling, decision dispatch, policy distribution |
| `basis-console` | Operator/admin UI | Policy inspection, audit review, operational management |
| `basis-adapters` | Protocol adapters | Field-protocol normalization and command delivery |
| `basis-identity` | Identity engine and federation boundary | External IdP integration, federation, login/session, claim mapping, canonical identity context |
| `basis-deploy` | Deployment tooling | Packaging, configuration, deployment validation |
| `basis-schemas` | Shared schemas | Authorization contracts, audit schemas, compatibility definitions |
| `basis-architecture` | Architecture documentation | White papers, ADRs, standards, and design reference |

The `basis-architecture` repository — where this document lives — is the canonical source for architectural decisions, design principles, and system documentation. It does not contain application code.

---

## Relationship to the White Papers

The white papers in this repository describe the full system model: the complete identity-aware authorization architecture for OT environments, including all zones, enforcement points, protocol adapters, identity providers, audit pipelines, and operational machinery.

basis-core implements the authorization kernel within that model — the stable evaluation core that the full architecture depends on. The white papers describe the architecture that basis-core anchors, not the full scope of what basis-core alone provides. Readers who want to understand what basis-core specifically implements should consult both this document and the component boundaries described in each white paper section alongside the full architectural model.
