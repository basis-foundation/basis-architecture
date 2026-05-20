# Diagram Specification: BASIS PoC Architecture Mapping

**Diagram file:** `basis-poc-architecture-mapping.mmd`
**Diagram type:** Conceptual architecture mapping
**Associated section:** `sections/06-basis-poc.md`
**Primary reference:** `basis-poc` repository — `docs/architecture/overview.md`

---

## Diagram Purpose

This diagram maps the implementation components of the BASIS proof-of-concept to the conceptual architecture roles described in Sections 04 and 05 of this paper. It answers the question: when the paper describes an identity provider, an enforcement point, a protocol adapter, or an audit pipeline, what did the PoC actually build to explore each of those roles?

The diagram is a conceptual mapping diagram, not a deployment diagram. It shows what each component does and how the key flows connect the layers. It does not show container topology, Docker networking, port assignments, volume mounts, or any of the operational infrastructure concerns that would appear in a deployment architecture diagram.

The intended reader is someone who has read Sections 04 and 05 and wants to understand how a concrete research implementation explored the architectural ideas described there. The diagram should be read alongside Section 06 prose, not as a standalone reference.

---

## What This Diagram Is and Is Not

**This diagram is:**

- A conceptual mapping from PoC implementation components to the authorization architecture roles defined in Sections 04 and 05
- A simplified representation of the key flows: authentication, authorization, command dispatch, telemetry, and audit
- A tool for situating the PoC within the paper's architectural framework
- Accurate to the actual PoC implementation in the `basis-poc` repository

**This diagram is not:**

- An infrastructure or deployment diagram
- A container orchestration diagram
- A complete picture of the PoC's internal module structure
- A guide for deploying the PoC in any environment
- Evidence that the PoC is production-ready or production-representative
- A claim that the specific technology choices (Keycloak, FastAPI, Mosquitto, SQLite) are required for the conceptual architecture

---

## Component Reference

The following table maps each diagram component to its implementation details and the conceptual architecture role it demonstrates.

| Diagram Node | Implementation | Conceptual Role |
|---|---|---|
| Operator / Technician | Browser or API client | Subject |
| Keycloak | Keycloak 23.0 (OIDC provider, RS256 JWT, realm roles: viewer / operator / admin) | Identity Provider |
| JWT Validation + subject_from_jwt() | `auth.py` — JWKS fetch (cached 5 min), RS256 signature validation, `Subject` model resolution | Operator Identity |
| PolicyEngine + RoleBasedPolicy | `policy/engine.py` + `policy/rbac.py` — chain-of-responsibility evaluation; action → role mapping; fail-closed default deny | Authorization Logic |
| require_action() / FastAPI routers | `auth.py::require_action()` FastAPI dependency — enforced on every protected endpoint before handler executes | Enforcement Point |
| MQTT Adapter (AdapterBase) | `adapters/mqtt/subscriber.py` + `adapters/mqtt/publisher.py` — subscribes to MQTT telemetry, publishes MQTT commands, normalizes to `TelemetryEvent` / `CommandEvent` | Protocol Adapter |
| Modbus TCP Adapter (AdapterBase) | `adapters/modbus/adapter.py` — in-memory register bank simulating chiller and pump; implements same `AdapterBase` interface as MQTT adapter | Protocol Adapter |
| Mosquitto | Eclipse Mosquitto 2.0, per-service credential auth, no per-topic authorization | Telemetry Transport |
| WebSocket Broadcaster | `ws_manager.py` — `TelemetrySession` model; JWT token required per connection (Stage 9); broadcasts `TelemetryEvent` to authenticated clients | Real-Time Telemetry View |
| HVAC Simulator | `services/simulator/simulator.py::HVACSimulator` — single-zone HVAC, publishes every 3s, accepts setpoint commands via MQTT | Operational Resource |
| CO₂ / Occupancy Sensors | `services/simulator/simulator.py::CO2Simulator` + `OccupancySimulator` — publishes every 6s / 12s | Operational Resource |
| Data Center (dc-boise-01) | `services/simulator/simulator.py::DataCenterSimulator` — rack inlet temps, thermal aisle, CRAC cooling, PDU, UPS, environmental sensors | Operational Resource |
| Chiller / Pump (Modbus registers) | `adapters/modbus/adapter.py` — in-memory register bank within the API process; no separate simulator process | Operational Resource |
| audit.db / DualAuditStore | `audit/store.py::DualAuditStore` — writes every `AuditEvent` to SQLite (append-only, WAL mode, indexed) and to structured stdout; `AuditLogger.record()` never raises | Audit Pipeline |

---

## Key Flows

### Authentication and Authorization Chain

The OIDC login flow uses PKCE: the browser redirects to Keycloak, receives an authorization code, exchanges it for a signed JWT. Subsequent API requests carry the JWT as a Bearer token. The API fetches Keycloak's JWKS endpoint (public keys, RS256) and validates the token signature and claims on every request. Token validation is stateless — no session state is stored in the API.

After JWT validation, `subject_from_jwt()` resolves the JWT payload into a typed `Subject` model containing the user's ID, roles, and name. `require_action()` then calls `PolicyEngine.evaluate(subject, action)` to determine whether the subject's roles permit the requested action. The `RoleBasedPolicy` maps action constants (`write:hvac:setpoint`, `read:audit:log`, etc.) to permitted role sets. The policy chain fails closed: an unrecognized action or an absence of matching roles produces `allowed=False`.

### Command Dispatch

An authorized command follows two paths depending on the target resource:

**MQTT path:** `require_action()` approves the request → the endpoint constructs a `CommandEvent` → `MQTT Adapter` publishes to Mosquitto (QoS 1) → the OT Simulator subscribes, receives the command, and applies it to the simulated device state.

**Modbus TCP path:** `require_action()` approves the request → the endpoint constructs a `CommandEvent` → `Modbus TCP Adapter` writes to in-memory register bank (chiller setpoint or pump speed) → no external process involved.

Both paths share the same `require_action()` enforcement and the same `AuditEvent` recording. The protocol is a concern only of the adapter layer.

### Telemetry

The OT Simulator publishes simulated telemetry to Mosquitto at defined intervals (HVAC every 3s, CO₂ every 6s, occupancy every 12s, data center every 9s). The MQTT Adapter subscribes, normalizes the raw MQTT payload into a `TelemetryEvent` domain model, and fans out the event to all authenticated WebSocket connections. WebSocket connections require a valid JWT token (Stage 9) and are tracked as `TelemetrySession` objects that expire when the token expires.

### Audit

Two `AuditEvent` records are produced for each successful command:
1. **Authorization event** — produced by `require_action()`, records the subject, action, policy decision, and outcome (`allowed` or `denied`). Written for every request, including denied ones.
2. **Dispatch event** — produced by the command router, records the command delivery outcome (`allowed` or `error`).

`DualAuditStore` writes each event to SQLite and to structured stdout independently. A write failure in one store does not affect the other. `AuditLogger.record()` never raises — audit failures are logged but never fail the request being audited.

---

## Arrow Semantics

| Arrow style | Represents | Examples |
|---|---|---|
| Dashed | Identity and auth flows (control-plane analog) | OIDC login, JWT issuance, JWKS fetch, policy evaluation chain |
| Solid | Operational flows (data-plane analog) | Command dispatch, telemetry, WebSocket delivery |
| Solid (labeled) | Audit flows | AuditEvent from Enforcement Point to Audit Layer |

The distinction between dashed and solid arrows in this diagram mirrors the control-plane / data-plane separation described in Section 04. In the PoC, this separation is logical (both flow through the same process and network), not physical.

---

## Intentional Omissions

The following details are present in the PoC implementation but are omitted from this diagram because they are infrastructure concerns rather than architectural mapping concerns.

**Container and Docker internals.** The PoC runs five Docker Compose services (keycloak, mosquitto, api, frontend, simulator) plus named volumes and a bridge network. Showing this level of detail would make the diagram a deployment diagram rather than a conceptual mapping diagram — the opposite of its purpose.

**Port assignments.** Keycloak listens on 18080, the API on 8000, the frontend on 5173, Mosquitto on 1883. These are implementation details specific to the development environment and have no architectural significance.

**Frontend (React + Vite).** The browser-based UI is an implementation of the "Operator / Technician (Browser)" node. Showing it as a separate component would suggest it has architectural significance distinct from the operator interaction — it does not. The UI demonstrates operator access patterns; its implementation technology is not architecturally relevant to the authorization model.

**JWKS caching details.** The API caches Keycloak's JWKS for 5 minutes and force-refreshes on unknown key IDs. This is a reliability and performance detail, not an architectural distinction worth representing in a conceptual diagram.

**Mosquitto per-service credential authentication.** Mosquitto is configured to require MQTT username/password credentials and disallow anonymous connections. This is a correct operational configuration but is not part of the authorization architecture being mapped — it is a transport security detail.

**SQLite schema and query interface.** The audit database has defined indexes, WAL mode, and a query API (`GET /api/audit` with filters). These are implementation details that belong in Section 06 prose and in the PoC documentation, not in the conceptual mapping diagram.

**Structured stdout.** `DualAuditStore` also writes every audit event to structured stdout (grep-friendly key=value format). This is omitted because the SQLite store is the architecturally significant persistent audit pipeline. Stdout is a development and operational convenience.

**Module import graph.** The PoC has a defined internal import graph (domain → everything; events → nothing; adapters → domain but not routers). This is an internal code quality constraint, not an architectural flow.

**Stage history.** The PoC was developed incrementally across numbered stages (1–10). The diagram represents the current state of the implementation, not the development sequence.

---

## Why Omissions Matter

The omissions are not arbitrary. Including infrastructure detail in a conceptual mapping diagram shifts the reader's attention from the architectural question — which components demonstrate which concepts — to deployment questions — how is this containerized, what ports are in use, how does the networking work.

The paper's purpose is to describe the conceptual architecture and examine how the PoC validates selected aspects of it. A diagram that looks like an infrastructure topology implies that the infrastructure is the architecture. The PoC's specific container configuration has no relevance to the question of whether identity-aware authorization is architecturally tractable in OT environments. That question is addressed by the policy evaluation model, the enforcement placement, the protocol adapter abstraction, and the audit record — not by the Docker Compose file.

---

## How to Use This Diagram in Section 06

Section 06 should introduce this diagram early and use it as a reference for the component descriptions that follow. The diagram's layer structure corresponds directly to the section's organizational structure: a description of each layer and its implementation serves the reader better than a flat list of components.

The diagram should not be presented as evidence of the architecture's completeness or production readiness. It should be presented as a map: here is what was built, here is what architectural concept each piece explores, and here is how the pieces connect. The section's prose fills in the what-was-demonstrated and what-remains-open-problems framing that the diagram cannot carry.

Specific uses in Section 06:

- **Orienting the reader** before component-by-component description: "The architecture mapping diagram (`diagrams/basis-poc-architecture-mapping.mmd`) shows the five layers of the PoC and which conceptual architecture role each component demonstrates."
- **Anchoring component descriptions** to the diagram: when describing Keycloak, reference the diagram node. When describing `require_action()`, reference the `[Enforcement Point]` annotation.
- **Marking boundaries** between what the PoC demonstrates and what it leaves open: the diagram shows the Modbus TCP adapter as a second `[Protocol Adapter]` node alongside the MQTT adapter, which is the appropriate place to note that this demonstrates protocol-agnostic adapter design at the PoC scale, not at production scale.

---

## Differences from a Production Architecture Diagram

A production architecture diagram for identity-aware authorization in an OT environment would differ from this diagram in several significant ways.

**Zone topology.** This diagram does not show trust zones (Operations Zone, Edge/Gateway Zone, Field Device Zone). The PoC does not implement the zone topology described in Section 05 — it implements a single API boundary that serves as the enforcement point for all access. A production architecture diagram would show enforcement at multiple zone boundaries as described in the trust boundary diagrams in this paper.

**Real field devices.** The PoC uses simulated devices implemented in Python. A production architecture diagram would show real BAS controllers, field hardware, and the protocol stacks they actually use. The conceptual roles are the same; the components are different.

**Distributed enforcement.** The PoC's enforcement is centralized at the FastAPI API boundary. The conceptual architecture described in Section 04 distributes enforcement across multiple enforcement points (jump host, local enforcement point at the edge). The PoC explores the authorization model at a single boundary; a production architecture diagram would show enforcement distributed across the topology.

**Local policy cache.** The PoC does not implement a local policy cache for offline operation. The conceptual architecture's offline resilience design — described in Section 04 — is not demonstrated in the current PoC. A production architecture diagram would show the local policy cache and the policy distribution path.

**High availability and redundancy.** The PoC runs as a single Docker Compose stack with no redundancy. A production architecture diagram would show redundant identity providers, policy engine replicas, and failover paths.

The diagram makes the PoC's boundary explicit through the "local development environment" label on the outer subgraph. The intent is to prevent the reader from interpreting this diagram as a production reference architecture.

---

## Pre-commit Review Checklist

Before committing changes to this diagram or its specification:

- [ ] All component labels accurately reflect the current PoC implementation in `basis-poc`
- [ ] `[Conceptual Role]` annotations match the terminology used in Sections 04 and 05
- [ ] Arrow semantics (dashed = auth/control, solid = operational) are consistent with the other diagrams in this paper
- [ ] The outer "development environment" boundary label is present and accurate
- [ ] No components imply production readiness, deployment topology, or infrastructure specifics
- [ ] Simulated device nodes are clearly labeled as simulations
- [ ] Spec document updated if diagram components or flows change

---

*This specification reflects the state of the BASIS PoC at Stage 10 (protocol-agnostic adapter PoC) + Stage 9 (authenticated WebSocket) + Stage 5b (SQLite audit) as documented in `services/api/main.py` and `docs/architecture/overview.md` in the `basis-poc` repository.*
