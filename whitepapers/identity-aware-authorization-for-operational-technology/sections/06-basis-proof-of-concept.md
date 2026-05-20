# BASIS Proof-of-Concept

Sections 04 and 05 described identity-aware authorization in architectural terms: a policy engine that evaluates requests, enforcement points that apply decisions, protocol adapters that normalize field-protocol messages, trust boundaries where those components are positioned, and a local policy cache that supports operation when connectivity to central services is interrupted. These concepts are useful as design targets, but architectural descriptions do not confirm that the described components can be assembled into a coherent whole, that the interactions between them behave as expected, or that the abstractions hold up against implementation details.

The BASIS proof-of-concept was built to perform that confirmation at a limited but concrete scope. Its purpose was not to produce a deployable system, but to validate that the core mechanisms of the architecture — identity propagation, explicit policy evaluation, enforcement at a defined boundary, protocol normalization, and audit recording — are coherent enough to implement in a connected form. The implementation revealed where the abstractions require additional specification, where assumptions embedded in the conceptual architecture do not map cleanly to available tooling, and where gaps between a development implementation and a production-grade one are larger than the architectural description suggests.

The analysis that follows treats the BASIS PoC as a research artifact. It describes what was built, why specific choices were made, what the implementation demonstrated, and what it explicitly did not demonstrate. It does not represent BASIS as a production architecture, a finished platform, or evidence that identity-aware OT authorization is a solved problem.

---

## Purpose and Scope of the PoC

The architecture described in this paper requires demonstrating that several distinct concerns — identity verification, policy evaluation, enforcement, protocol normalization, and audit — can be realized as connected components that interact as the model predicts. Demonstrating each concern in isolation is insufficient; the architecture's value depends on them functioning as a system.

The scoping decision for the PoC followed from this requirement. Rather than building a single component in depth, the implementation was intentionally breadth-first: cover the full path from operator authentication through policy evaluation, enforcement, protocol dispatch, and audit recording, using simulated rather than real OT resources. This produced a smaller, more tractable system than a deployment-oriented implementation would require, while still exercising the interactions between components that matter most for architectural validation.

The PoC was also scoped with reproducibility as an explicit requirement. An architectural exploration that cannot be independently reproduced is harder to evaluate critically. By targeting a self-contained, containerized development environment accessible via GitHub Codespaces, the implementation could be shared and examined without requiring access to physical OT infrastructure or specialized networking equipment. This decision shaped several implementation choices, including the selection of simulated OT resources over real ones and the use of local containerized infrastructure over distributed components.

---

## Why a Proof-of-Concept Was Necessary

The conceptual architecture in Section 04 makes several claims about component interaction that cannot be validated by reasoning alone. It claims that identity context can be propagated from an authentication event at the operator boundary to policy evaluation at an enforcement point positioned downstream of that event. It claims that a protocol adapter can normalize field-protocol messages into the subject-resource-action representation required by the authorization model, and that this normalization can proceed in a way that is transparent to the policy engine. It claims that authorization decisions and their audit records can be generated consistently and independently enough to produce a meaningful dual-record for each evaluated request.

These claims involve coordination between components built with different assumptions — an identity provider that issues tokens using OIDC conventions, an API framework that validates those tokens and applies policy, adapter code that understands field-protocol semantics, and an audit store that receives events from multiple sources. Whether these components can be wired together in a way that faithfully reflects the architecture's design is an empirical question. The answer requires building something and observing what happens.

The PoC also served a secondary purpose: surfacing the implementation-level questions that the architectural description defers. The conceptual architecture describes a local policy cache that holds policy applicable to the traffic a local enforcement point governs. Implementing that concept requires decisions about cache structure, invalidation semantics, policy distribution format, and enforcement-point behavior when cached policy is absent or expired. These questions are not answerable at the level of architectural description; they require implementation to make them concrete. The PoC did not fully resolve all such questions, but it made them visible.

---

## PoC Architectural Overview

The BASIS PoC architecture maps its components to the conceptual architecture roles defined in Sections 04 and 05. The mapping diagram (`diagrams/basis-poc-architecture-mapping.mmd`) presents this correspondence explicitly, using `[Conceptual Role]` annotations to mark each component's architectural function. The following description should be read alongside that diagram; it explains the implementation choices rather than repeating what the diagram shows.

The PoC is organized into five layers: an operator layer, an identity and access layer, an authorization and API layer, a protocol adapter layer, and a simulated OT resource layer. An audit layer spans the authorization and adapter layers, recording events generated by both.

All components below the operator layer run within a single Docker Compose stack, deployable locally or in GitHub Codespaces. This is a significant structural difference from the multi-zone topology described in Section 05, where components are distributed across geographically or administratively distinct zones. In the PoC, the identity provider, policy engine, enforcement point, protocol adapters, and simulated resources all share the same containerized environment. The PoC's value is not topological fidelity — it makes no claim to accurately represent the physical distribution of a production deployment — but functional fidelity: the logical relationships between components, the authorization flow, and the enforcement behavior are the things under investigation.

The diagram uses dashed arrows for identity and authorization flows, which correspond to the control-plane flows in the conceptual architecture, and solid arrows for operational and telemetry flows, which correspond to the data-plane flows. This visual convention was adopted directly from the conceptual architecture diagrams to make the correspondence explicit.

---

## Identity and Authorization Model

Keycloak serves as the identity provider in the BASIS PoC, occupying the identity provider role described in the conceptual architecture. It issues RS256-signed OIDC tokens through a PKCE-based login flow. Operators authenticate to Keycloak directly; no intermediary resolves their identity. The resulting JWT carries the operator's role assignments as Keycloak realm roles, which the API uses during authorization evaluation.

The choice of Keycloak reflects its availability as an open-source, locally deployable OIDC provider suitable for a containerized development environment. It does not reflect a recommendation for Keycloak as the correct identity provider for production OT deployments. The conceptual architecture is identity-provider-agnostic; any system capable of issuing verifiable, attribute-carrying tokens and supporting a JWKS endpoint for token validation would satisfy the architectural role. The PoC happens to use Keycloak because it is well-documented, operationally self-contained, and straightforward to configure for the role assignments the implementation required.

The API layer — implemented as a FastAPI modular monolith — performs the work of the operator identity component, the policy engine, and the enforcement point as described in the architecture. When an operator presents a Bearer token to an API endpoint, the `subject_from_jwt()` function validates the token's signature against Keycloak's JWKS endpoint and resolves the token claims into a typed subject representing the authenticated operator. This step is the identity propagation mechanism in the PoC: the verified subject identity produced at token validation is available to every subsequent step in the request evaluation chain.

Policy evaluation is performed by a `PolicyEngine` component implementing `RoleBasedPolicy`. For each request, the policy engine receives the resolved subject and the requested action, evaluates the subject's role assignments against the policy applicable to that action, and returns a policy result. The policy model in the PoC is RBAC, using Keycloak-issued realm roles as the role assignment mechanism. This is a simplified version of the authorization model described in the conceptual architecture, which anticipates context-aware policy extending beyond static role assignments. The PoC's RBAC implementation validates the structural connection between subject resolution and policy evaluation; it does not validate the more complex policy expressions that a production deployment would require.

Enforcement is implemented through the `require_action()` dependency, which is attached to every API endpoint that governs access to operational resources. No command reaches a protocol adapter without passing through this enforcement gate. This placement reflects the enforcement-point concept in the conceptual architecture: the enforcement point applies policy decisions at the boundary before operational traffic proceeds to the protocol layer. In the PoC, the enforcement point and the policy engine reside in the same process, which simplifies implementation but does not accurately represent the deployment topology where enforcement points are distributed across trust zones and query a shared, remote policy engine.

The collapsed co-location of identity resolution, policy evaluation, and enforcement in a single process is the most significant structural difference between the PoC's authorization model and the distributed model described in the conceptual architecture. In a production deployment, these concerns would be operationally separated: the identity provider would be a distinct, HA-capable service; the policy engine would be separately administered and its policy independently versioned; the enforcement point would be a component that queries the policy engine remotely and operates against a local cache when the engine is unreachable. The PoC's monolithic implementation cannot validate those operational separation properties, even though it validates the logical relationships between them.

---

## Protocol Adapter Design

The protocol adapter layer in the PoC implements the adapter abstraction described in Section 04. Both adapters — the MQTT adapter and the Modbus TCP adapter — implement a shared `AdapterBase` interface, which establishes a common contract for how adapters receive authorized commands from the enforcement point and produce telemetry events that the downstream pipeline consumes.

The MQTT adapter handles communication with the HVAC simulator, CO2 and occupancy sensors, and the data center resource. It publishes authorized commands as MQTT messages to the Mosquitto broker and subscribes to telemetry topics published by simulated devices. The Modbus TCP adapter handles communication with the chiller and pump, which are represented as in-memory Modbus register sets. Each adapter contains the protocol-specific logic required to translate between the shared representation used by the API layer and the format required by its respective protocol.

This design reflects the architectural intent in Section 04: protocol-specific logic is contained in the adapter layer, and the API's authorization and audit paths are protocol-agnostic. An authorized command issued by the enforcement point is expressed in the shared representation — referencing a resource by its identifier and specifying the action to perform — and the adapter is responsible for translating that into the appropriate MQTT publish or Modbus register write. The policy engine evaluates the authorization request without any knowledge of which protocol the target resource speaks.

The adapter implementations in the PoC are, however, not full industrial protocol stacks. The MQTT adapter uses a Python MQTT client communicating with a locally containerized Mosquitto broker — a development-grade configuration rather than a production-grade OT messaging infrastructure. The Modbus TCP adapter simulates registers in memory rather than communicating with real Modbus hardware. These implementations demonstrate the adapter pattern — the structural relationship between the enforcement point, the adapter, and the downstream resource — without demonstrating the protocol handling challenges that would arise in real deployments: timing requirements, error recovery at the protocol level, handling of malformed or unexpected responses from real devices, register mapping for actual controller firmware, and the object model variations between devices from different manufacturers.

The semantic normalization that the adapters perform in the PoC maps a limited vocabulary of simulated resource identifiers and actions to the protocol-level operations required to execute them. In a real deployment, this normalization must cover the full object model of the devices being managed — a scope that expands substantially with the number of device types, firmware versions, and protocol variants in the environment. The PoC demonstrates that normalization is implementable for a controlled set of resources; it does not demonstrate that the normalization logic is maintainable at real-deployment scale.

---

## Telemetry and Real-Time Visibility

Outbound telemetry from simulated devices flows from the resource layer through the MQTT broker to the MQTT adapter, which normalizes the raw MQTT payloads into structured `TelemetryEvent` objects and forwards them to a WebSocket broadcaster. Operators with an authenticated browser session receive this telemetry stream in real time.

The WebSocket broadcaster requires authentication: operators must present a valid JWT to establish a WebSocket connection, and the broadcaster validates that token before admitting the stream. This reflects the architecture's asymmetric treatment of inbound commands and outbound telemetry discussed in Section 05. Telemetry cannot carry verified operator identity the way a command can — it originates from devices, not from authenticated subjects — but the access path for consuming telemetry can still be access-controlled. The PoC's authenticated WebSocket enforces that an operator has presented valid credentials before receiving the telemetry stream, without claiming that the stream itself carries device-originated authentication.

The telemetry design in the PoC is illustrative rather than representative of production requirements. The MQTT topics and payload structures used by the simulated devices are defined for this implementation and do not reflect real device message schemas. The MQTT broker is locally hosted, single-instance, and without the message durability, clustering, or access control configuration that production MQTT deployments require. The WebSocket connection carries telemetry to a browser; it does not represent how normalized telemetry would be integrated into a real operational data historian, alert management system, or energy management platform. The PoC demonstrates the telemetry normalization path and real-time delivery concept; it does not validate the scale, durability, or integration surface of a production telemetry pipeline.

---

## Audit Logging and Authorization Visibility

The BASIS PoC implements audit logging through a `DualAuditStore` that writes every authorization decision — whether the decision was allow or deny — to two destinations simultaneously: an SQLite database and structured stdout. The SQLite store is append-only by implementation; the audit logging path is designed so that a write failure does not fail the request being logged.

Every API endpoint that governs access to operational resources generates two audit events: one recording the authorization decision at the enforcement point, and one recording the outcome of the command dispatch (on the allow path only). This dual-event structure reflects the architectural intent described in Section 04: the authorization event and the operational event are recorded separately, so that the audit record captures both what was decided and what resulted from the decision. An authorization event without a corresponding dispatch event indicates a command that was authorized but did not complete. A dispatch event without a preceding authorization event would indicate an enforcement failure — a condition that should not occur under correct implementation but whose detectability in the audit record is an architectural property that must hold.

The `DualAuditStore` design — writing to both a durable store and structured stdout simultaneously — reflects the dual-record concept in the conceptual architecture, where audit events flow from enforcement points and independently from the policy engine to provide redundant coverage. In the PoC, both destinations receive the same event from the same source rather than from independent sources, which is a simplification relative to the architecture. In a production deployment, the enforcement point's audit events and the policy engine's decision records would be generated by separate components and must be correlatable by a common request identifier. The PoC validates that audit events are generated at the enforcement point and include the required fields; it does not validate the independent-source, out-of-band-delivery properties that make the audit record trustworthy as evidence in a production environment.

The SQLite store is local to the containerized environment and is not replicated or independently administered. An audit store that is administratively collocated with the operational components it covers does not satisfy the administrative independence requirement described in Section 04. The PoC implementation acknowledges this: the SQLite store serves as a durable research record for the implementation's development and review cycle, not as a model for production audit storage.

---

## Simulated Operational Resources

The BASIS PoC represents OT resources through simulation rather than physical hardware. The simulated resources include an HVAC system with temperature setpoints and zone states, CO2 and occupancy sensors, a data center resource (`dc-boise-01`) publishing rack, thermal, power, and UPS telemetry, and a chiller and pump represented as Modbus register sets. These were chosen to represent two functionally distinct deployment contexts — commercial building HVAC and data center infrastructure — that exercise different telemetry patterns, different control vocabularies, and different protocol paths through the adapter layer.

The choice of simulation over real hardware was deliberate and entails known limitations. Simulation allows the PoC to be reproduced in any environment with Docker support, removes hardware dependencies from the development and review cycle, and permits the resource behavior to be defined precisely enough that the authorization model can be validated against predictable inputs and outputs. These are appropriate properties for a research implementation.

What simulation does not provide is fidelity to the operational behavior of real OT devices. Real HVAC controllers have communication timing requirements, error conditions, and state management behaviors that simulated Python processes do not reproduce. Real Modbus devices have register maps that vary between manufacturers and firmware versions, response timing that depends on physical layer characteristics, and error codes that require protocol-level handling. The authorization model operates on resource identifiers and action labels that the PoC defines; in a real deployment, those identifiers and labels must be mapped to the actual semantics of the devices being managed, a process that introduces configuration complexity the simulation bypasses entirely.

The simulated resources also operate at a scale that is not representative of real OT deployments. A commercial building automation deployment may include hundreds of BAS controllers, thousands of monitored points, and multiple protocol stacks. The PoC's five simulated resources exercise the authorization path for a small, controlled set of resource identifiers. Scaling the authorization model to a realistic resource inventory — defining resource descriptors for a real object model, managing the policy that governs access to each class of resource, and maintaining the adapter normalization for a heterogeneous device population — is a materially larger engineering problem than the PoC addresses.

---

## Development Environment and Reproducibility

The BASIS PoC runs in a Docker Compose stack configured for local development and for GitHub Codespaces. All components — Keycloak, the FastAPI application, Mosquitto, the simulated devices, and the SQLite audit store — run as containers within this stack. Port forwarding in Codespaces allows browser-based interaction with the API and telemetry stream without requiring local infrastructure beyond a browser and a GitHub account.

Reproducibility was treated as a design constraint rather than a convenience feature. An architectural exploration that requires specialized networking, licensed software, or physical OT hardware to reproduce cannot be independently evaluated. By constraining the PoC to containerized, open-source components deployable from a public repository, the implementation can be examined, critiqued, and used as a reference by anyone evaluating the architectural concepts it explores.

This constraint shaped several implementation decisions that would not be appropriate in a production context. Keycloak runs in a single-container development configuration without external database persistence — a configuration that Keycloak's own documentation warns against for production use. The MQTT broker runs without TLS or access control, relying on container network isolation rather than transport-layer security. Credentials for development identities are committed to the repository configuration. These are known, intentional compromises for a development environment; they would each require remediation before any component of this configuration was used outside that context.

The Codespaces onboarding path also reflects a prioritization of accessibility over operational realism. The goal was to make the authorization concepts explorable with minimal setup — a legitimate goal for a research implementation that is intended to be examined, not deployed.

---

## What the PoC Demonstrated

The BASIS PoC produced several concrete demonstrations that support the architectural claims in this paper.

Identity propagation from authentication through policy evaluation to enforcement is implementable as a connected flow. An operator who authenticates through Keycloak receives a JWT that carries their role assignments; that JWT is validated at the API boundary; the resolved subject identity is available to the policy engine at evaluation time and to the audit logger at recording time. The subject who issued a command is attributable at every step in the request processing chain, including in the audit record. This is the core identity propagation claim of the conceptual architecture, and the PoC implements it.

Explicit policy evaluation at an enforcement point can be interposed between operator intent and protocol dispatch without breaking the authorization flow or introducing prohibitive complexity in the implementation. The `require_action()` dependency pattern enforces that every endpoint governing access to operational resources passes through a consistent evaluation gate. Adding a new endpoint to the API requires explicitly declaring the action it governs; there is no path by which an endpoint can be added without making an authorization decision, because the evaluation is structural rather than optional.

Protocol-agnostic authorization is achievable through the adapter abstraction. The policy engine evaluates resource-action pairs without any knowledge of whether the underlying resource speaks MQTT or Modbus. Adding the Modbus adapter to an implementation that already had the MQTT adapter did not require changes to the authorization or audit logic — only a new adapter implementation that conforms to `AdapterBase` and handles Modbus protocol specifics. This validates the extensibility claim of the protocol adapter model.

Dual-record audit logging is implementable without blocking the request path. The `DualAuditStore` writes to SQLite and stdout on every authorization decision, authorized or denied, without adding a failure mode to the command path. The append-only write pattern and the decoupling of audit write failures from request failures are both implementable properties.

Authenticated real-time telemetry delivery to a browser client is achievable with the selected transport stack. Operators can observe simulated OT telemetry in their authenticated session, with the telemetry normalized from device-specific MQTT payloads to a shared structure before delivery.

Collectively, these demonstrations validate that the core mechanisms of the conceptual architecture are coherent: the components interact as described, the logical flows hold under implementation, and the authorization model can be applied consistently across heterogeneous protocol paths through the adapter abstraction.

---

## What the PoC Did Not Demonstrate

The PoC's scope was limited by design, and the limitations define clearly what the implementation cannot be taken as evidence for.

**Distributed policy evaluation.** The PoC collapses the identity provider, policy engine, and enforcement point into a single containerized environment. The local policy cache, offline resilience, and distributed enforcement-point design described in Section 04 were not implemented. The claim that enforcement points can operate correctly against locally cached policy when the policy engine is unreachable has not been validated. The claim that policy updates can be distributed to enforcement points at multiple sites consistently and securely has not been validated.

**Production-scale OT resource coverage.** Five simulated resources do not constitute a meaningful test of the authorization model's scalability. The policy authoring, resource descriptor management, and adapter normalization required for a realistic device population — measured in hundreds of controllers and thousands of monitored points — were not exercised.

**Operational latency validation.** No latency measurements were taken under realistic command volumes or under conditions representing real-time control loop requirements. The conceptual architecture positions enforcement at boundaries where latency can be tolerated, but whether the authorization evaluation latency introduced by the PoC's implementation would fall within acceptable bounds for supervisory-layer operations in a real deployment was not tested.

**High availability.** The PoC runs as a single Docker Compose stack with no redundancy, no failover, and no recovery testing. The identity provider, policy engine, and broker each represent single points of failure that would require explicit availability design for production use.

**Real protocol handling.** The MQTT and Modbus adapters communicate with simulated resources under controlled conditions. Connection errors, protocol-level timeouts, malformed responses, and the behavioral variations between real devices of the same protocol class were not encountered and not handled.

**Safety constraint validation.** The conceptual architecture discusses the interaction between authorization enforcement and OT safety requirements — particularly the tradeoff between failing closed and failing open at enforcement points guarding life-safety systems. The PoC does not operate in an environment with safety constraints and made no attempt to characterize this tradeoff against real operational requirements.

**Security hardening.** The PoC runs with development-grade security configuration throughout: no TLS on the broker, no database persistence for the identity provider, development credentials in the repository. The implementation has not been reviewed for production security requirements and should not be treated as a reference for secure configuration.

---

## Architectural Gaps and Remaining Challenges

Building the PoC surfaced several areas where the distance between the conceptual architecture and a production-grade implementation is larger than the architectural description suggests.

**Policy distribution and cache management.** The conceptual architecture describes a local policy cache at edge enforcement points, synchronized from the central policy engine and used for local evaluation when the engine is unreachable. The PoC did not implement this component. Implementing it correctly requires decisions about policy format, distribution protocol, cache invalidation, staleness limits, and enforcement-point behavior when the cache is expired. These are implementation-level decisions that the architectural description defers, and they carry significant operational consequence: a misconfigured staleness limit means enforcing outdated policy; an incorrectly implemented cache invalidation means that policy changes do not propagate as expected.

**Adapter normalization at realistic scale.** The adapter normalization in the PoC covers a fixed, small set of resource identifiers and actions defined by the simulation. Extending normalization to the full object model of a real BAS deployment requires a structured approach to resource descriptor management that the PoC does not establish. The gap between "normalization is implementable for five resources" and "normalization is maintainable for a real deployment" is primarily an engineering and tooling gap, not a conceptual one, but it is a real gap.

**Identity federation.** The PoC uses a single, locally administered Keycloak instance. The conceptual architecture anticipates that enforcement points may need to evaluate identities issued by external identity providers — enterprise directories, vendor identity services, or federated OT-specific identity infrastructure. Token validation across federation boundaries, attribute mapping from external claims to the authorization model's subject representation, and trust management for federated identity sources were not explored.

**Operational tooling.** The PoC has no tooling for policy authoring, policy review, resource descriptor management, or audit query. These are operational requirements for the architecture, not optional features. A deployment that has enforcement infrastructure but no tooling for understanding what policy says, what requests have been evaluated, and what the audit record contains cannot be operated effectively. Building this tooling is a substantial effort that the PoC entirely defers.

**Credential lifecycle management.** The PoC uses static, development-purpose credentials that do not expire, rotate, or reflect any realistic identity lifecycle. A production authorization infrastructure must handle credential issuance, expiration, rotation, and revocation — including the consequences of revocation at enforcement points operating on cached policy that predate the revocation event.

---

## Lessons Learned

Several observations from building the PoC are directly relevant to how the conceptual architecture should be understood.

The enforcement-point abstraction is load-bearing in a way the conceptual description does not fully convey. In the PoC, the enforcement point's correctness depends on the accurate resolution of the subject identity from the token, the correct mapping of the requested action to the policy model's vocabulary, and the accurate identification of the target resource. Any error in these inputs produces a policy evaluation that is structurally correct but semantically wrong — the policy engine evaluates the right question about the wrong subject, action, or resource. This is the same concern that Section 05 raises about adapter normalization, and it applies equally at the enforcement-point boundary: the enforcement point is trusted unconditionally by the policy engine, and the trustworthiness of that trust depends entirely on the correctness of the subject and resource resolution logic upstream of it.

The audit record is most useful when it is designed with specific query patterns in mind. The PoC's audit events contain subject identity, resource identifier, action, decision, timestamp, and dispatch outcome. Designing these fields for the specific questions that operators and auditors will ask — who issued commands to a specific resource over a time window, which requests were denied and why, what actions preceded an anomalous state change — requires understanding those questions before the audit schema is finalized. The PoC's schema is functional for development review; whether it is functional for operational audit is a question that requires input from the teams who will conduct those audits.

Reproducibility as a constraint improved the clarity of the implementation's assumptions. Constraining the PoC to a containerized development environment forced every external dependency to be explicitly declared, containerized, and documented. This made the implementation's actual dependency set visible in a way that an implementation built against existing shared infrastructure would not. The limitations of the development configuration — single-instance Keycloak, no TLS, no HA — are explicit rather than implicit, which is a better starting point for understanding what would need to change for a production deployment.

---

## Summary

The BASIS PoC is an exploratory research implementation built to validate that the core mechanisms of the identity-aware authorization architecture described in this paper are coherent at the level of a connected, functioning system. It demonstrated that identity propagation from authentication to audit is implementable, that explicit policy evaluation can be interposed at an enforcement point without structural gaps in the request path, that protocol-agnostic authorization is achievable through a shared adapter abstraction, and that per-decision audit logging is implementable without adding request-path failure modes.

It did not demonstrate distributed policy evaluation, local policy caching, operational latency adequacy, high availability, real industrial protocol handling, safety constraint behavior, or the operational tooling required to manage the authorization infrastructure it represents. The gap between the PoC and a production-grade system is not primarily a matter of completing partially implemented features. It is a gap in validated operational behavior across dimensions — latency, availability, protocol fidelity, policy management, credential lifecycle — that the PoC's controlled development environment cannot exercise.

The PoC's value is proportional to how precisely its scope is understood. As evidence that the conceptual architecture is implementable in a coherent form at small scale, it is useful. As evidence that identity-aware OT authorization is ready for deployment in production environments, it would be misleading. The work it represents is a starting point for the validation that production readiness actually requires — not a demonstration that the work is complete.
