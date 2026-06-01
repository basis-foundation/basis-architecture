# basis-core Kernel Boundary Rules

This document defines the enforceable rules that protect basis-core as an isolated authorization kernel. It is the review reference for any proposed change to basis-core: what may enter the kernel, what must stay outside it, and how boundary questions are evaluated.

For the conceptual rationale behind these rules, see [`docs/architecture/basis-ecosystem.md`](architecture/basis-ecosystem.md). For governance consequences of boundary violations, see [`GOVERNANCE.md`](../GOVERNANCE.md). For the architectural principles that motivate kernel isolation, see [`docs/architecture-principles.md`](architecture-principles.md).

---

## Non-Negotiable Rules

These rules are not defaults or preferences. They are invariants. A proposed change that violates any of them requires an Architecture Decision Record and Foundation review before it can be accepted — not afterward.

### Protocol and transport

- basis-core must remain protocol-agnostic. It must have no knowledge of BACnet, Modbus, MQTT, OPC-UA, or any other field or application protocol.
- basis-core must remain transport-agnostic. It must contain no HTTP, WebSocket, gRPC, AMQP, or other transport-layer code.
- basis-core must not perform network I/O during policy evaluation.

### Identity and runtime services

- basis-core must not operate or integrate directly with an identity provider. It must not fetch JWKS endpoints, issue tokens, manage LDAP connections, or call Keycloak, Entra ID, Okta, Auth0, or equivalent services.
- basis-core must not include database clients, ORMs, or runtime persistence backends. SQLAlchemy, PostgreSQL drivers, SQLite runtime connections, and equivalent libraries must not appear in kernel dependencies.
- basis-core must not include cloud platform SDKs, Kubernetes or Docker client libraries, or deployment tooling of any kind.
- basis-core must not include UI or console logic.

### Evaluation semantics

- Policy rules must be deterministic and stateless at evaluation time. A given subject, resource, action, and policy set must produce the same decision on every evaluation, with no dependence on mutable external state.
- Actions not covered by any policy rule must default to deny. Absence of a matching policy is not permission.
- Raw exceptions and internal stack traces must not propagate to callers. Callers receive structured decision responses or defined failure reasons.

### Audit

- Audit is evidence, not enforcement. An authorization decision must be returned to the caller before — and independently of — audit write success.
- An audit write failure must not reverse or delay an authorization decision already reached.

### Import discipline

- Lower layers must not import higher layers. The dependency direction within the kernel package is strictly downward: `domain` → `decisions` → `policy` → `audit` → `adapters` → `enforcement`.
- Nothing outside the application or runtime layer may import from `basis_core.enforcement`. Lower-layer kernel subpackages must not import from `enforcement`.

### The separation principle

- Adapters normalize; basis-core evaluates. basis-core never normalizes raw field-protocol data. basis-core never dispatches commands to field devices.

---

## Allowed Kernel Responsibilities

basis-core owns exactly the following concerns:

- Domain primitives: `Subject`, `Resource`, `Action`, `IdentityContext`
- Decision contracts: `DecisionRequest`, `DecisionResponse`, `FailureReason`
- Policy evaluation semantics: the logic that maps a request to an authorization decision
- Policy rule contracts and the policy format that the evaluation engine consumes
- Enforcement boundary semantics: the contracts that enforcement points must satisfy
- Failure mode contracts: defined behavior for evaluation failures, including fail-closed and fail-open semantics and the conditions under which each applies
- Audit event contracts: the canonical schema for authorization events, required fields, semantic definitions, and emission conditions
- The audit writer protocol: the interface that audit writers must implement, not the writers themselves
- Adapter contracts: the interfaces that protocol adapters must implement when producing `DecisionRequest` objects and applying `DecisionResponse` outcomes — not the adapter implementations
- Schema examples and test fixtures that validate kernel evaluation behavior in isolation

Anything not in this list is not a kernel responsibility. When in doubt, apply the boundary decision test below.

---

## Disallowed Concerns

The following concerns must not appear in basis-core, regardless of how the proposal is framed:

**basis-gateway concerns:** API hosting, HTTP request routing, authentication middleware, session management, policy distribution service, request queueing, rate limiting, health check endpoints.

**basis-console concerns:** UI rendering, policy management screens, audit dashboards, operator workflows, configuration wizards.

**basis-adapters concerns:** BACnet normalization, Modbus normalization, MQTT normalization, OPC-UA normalization, vendor-specific protocol handling, device command dispatch, adapter lifecycle management.

**basis-deploy concerns:** Docker image configuration, Kubernetes manifests, Helm charts, cloud deployment scripts, installer tooling, environment bootstrapping.

**BASAuth concerns:** managed service coordination, fleet management, enterprise integration connectors, commercial support tooling, multi-tenancy management. BASAuth must not be a required dependency for basis-core to function.

**Identity provider operation:** Keycloak administration, Entra ID configuration, Okta/Auth0 clients, LDAP directory clients, token issuance, JWKS endpoint fetching, certificate authority clients.

**Persistence implementations:** SQLAlchemy models, PostgreSQL runtime connections, SQLite runtime persistence, S3 or object storage clients, SIEM writer implementations.

**Observability stacks:** Datadog clients, OpenTelemetry SDK exporters, Prometheus metric servers, structured logging frameworks that introduce external dependencies.

The presence of any of these in a pull request to basis-core is a boundary violation. The question is not whether the specific addition seems minor — it is whether the rule holds.

---

## Boundary Decision Test

For any proposed addition to basis-core, answer these questions:

1. Does this change alter or extend authorization evaluation semantics, or define a stable contract that multiple higher-level components must implement?
2. Can it be tested in-process, in isolation, without a database, network service, identity provider, protocol stack, or running container?
3. Would basis-gateway, basis-console, basis-adapters, or basis-deploy be a more appropriate owner?
4. Does this introduce network I/O, file I/O, runtime orchestration, or deployment assumptions?
5. Does this make basis-core harder to embed in a constrained environment, air-gapped deployment, or test harness?
6. Does this require a specific OT protocol, cloud provider, or identity system to be useful?
7. Does this change evaluation behavior in a way that could break existing enforcement points without an explicit version increment?

If question 1 is yes and questions 3–7 are all no, the change may belong in basis-core. If questions 3, 4, 5, or 6 are yes, the change does not belong in basis-core — it belongs in a higher-level component.

If a design problem appears to require relaxing a kernel boundary, the correct response is to reconsider whether the concern belongs in the kernel at all. Proposing a specific exception is the wrong starting point.

---

## Import Boundary Summary

The intended subpackage structure for basis-core is:

- `domain` — primitives: Subject, Resource, Action, IdentityContext
- `decisions` — contracts: DecisionRequest, DecisionResponse, FailureReason
- `policy` — evaluation semantics and rule contracts
- `audit` — event contracts, emission conditions, audit writer protocol
- `adapters` — adapter interface contracts (not implementations)
- `enforcement` — top-level orchestration: coordinates evaluation, enforces decisions, emits audit events

Import rules:

- `enforcement` is the top layer. Lower layers must not import from it.
- `adapters` may import from `domain`, `decisions`, and `policy` contracts.
- `audit` may import from `domain` and `decisions`.
- `policy` may import from `domain` and `decisions`.
- `decisions` may import from `domain`.
- `domain` imports nothing from within basis-core.

The full import boundary specification, including enforcement via static analysis tooling, belongs in the basis-core implementation repository at `docs/import-boundaries.md`. That document is the authoritative detail reference. This summary establishes the architectural intent.

---

## Relationship to Surrounding BASIS Components

**basis-gateway** wraps basis-core as a runtime API service. It hosts the HTTP interface, manages request lifecycle, invokes basis-core for evaluation, and returns decisions to callers. basis-gateway depends on basis-core; basis-core does not depend on basis-gateway.

**basis-console** depends on basis-gateway and must not implement authorization logic directly. It provides operator visibility into the system; it does not evaluate policy.

**basis-adapters** produces normalized `DecisionRequest` objects from field-protocol messages and applies `DecisionResponse` outcomes to field-level command delivery. basis-adapters depends on basis-core for its contract definitions; basis-core does not depend on basis-adapters for anything. See [`docs/architecture/basis-adapters.md`](architecture/basis-adapters.md) for the canonical adapter architecture reference.

**basis-deploy** packages the runtime — basis-gateway, basis-adapters, and their dependencies — for deployment into OT environments. It is not part of the authorization runtime. basis-core must be deployable independently of basis-deploy's packaging choices.

**BASAuth** may build commercial services on top of the BASIS Core Services Distribution, including services that call into basis-core or basis-gateway. BASAuth must not be a required dependency for basis-core to function. A deployment that does not use BASAuth must be a complete, functional authorization system.

---

## Required Implementation Checks

The following checks must pass on any change to the basis-core implementation. They are recorded here as an architectural requirement; the implementation repository owns the tooling configuration.

```text
pytest
ruff check src tests
ruff format --check src tests
mypy src
```

Import boundary enforcement should be covered by dedicated tests that verify the subpackage dependency graph matches the rules stated in this document. If a proposed change to basis-core requires expanding import boundary test coverage to reflect new rules, that test expansion should be proposed and reviewed separately before the implementation change that depends on it is merged. Import boundary tests are governance infrastructure, not incidental test coverage.

---

## Cross-References

- [`docs/architecture/basis-ecosystem.md`](architecture/basis-ecosystem.md) — component responsibilities, what belongs in basis-core, what must stay outside, dependency direction
- [`GOVERNANCE.md`](../GOVERNANCE.md) — basis-core boundary protection as a governance concern; what a boundary exception requires
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — contribution scope and terminology requirements for this repository
- [`docs/architecture-principles.md`](architecture-principles.md) — Principles 5, 6, 10, and 12, which motivate protocol-agnosticism, operational resilience, protocol abstraction, and operational constraint respect
- `docs/kernel-boundary.md` in basis-core — conceptual explanation of the boundary and the reasoning behind each rule (to be created in the basis-core implementation repository)
- `docs/import-boundaries.md` in basis-core — authoritative import rule specification and enforcement configuration (to be created in the basis-core implementation repository)
