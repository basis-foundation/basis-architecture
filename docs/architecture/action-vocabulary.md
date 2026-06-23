# Action Vocabulary

This document defines the governance model for action naming in the BASIS ecosystem. It establishes the naming structure, conventions, stability expectations, and compatibility rules that apply to action names used in policy evaluation and audit records.

Section 04 of the white paper establishes why actions are one of three fundamental primitives in the authorization model — alongside subjects and resources — and notes that "the granularity of action definition directly determines the expressiveness of the policy model." This document extends that analysis with the governance rules that keep the action vocabulary coherent, stable, and interoperable across components and deployments.

Cross-references: [`docs/architecture/action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md) — the ecosystem-wide inventory and reconciliation that established the canonical verb set recorded here. [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md) — action vocabulary stability requirements. [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md) — adapter normalization responsibilities. [`docs/glossary.md`](../glossary.md) — definitions of Action, Resource, Subject, Protocol Adapter, and Policy Evaluation.

---

## Actions as Policy Contracts

An action name is not a label. It is a contract.

When a policy author writes `write:hvac:setpoint`, they are making a binding claim about which protocol operations should be governed by that policy. When a protocol adapter normalizes a BACnet WriteProperty request to `write:hvac:setpoint`, it is claiming that this protocol operation semantically corresponds to that action. When an audit record captures `write:hvac:setpoint`, it is asserting that the recorded operation was of that type for the duration of the audit record's retention period.

These three uses — policy, normalization, audit — must be consistent, and they must remain consistent across time. A policy authored today must evaluate correctly against normalizations produced years from now. An audit record produced today must be interpretable years from now. This requires that the vocabulary behind the action name be stable: that the name refers to the same operation, with the same scope, under the same conditions, across the operational lifetime of the deployments that depend on it.

This is the justification for treating action naming as a governance concern rather than an implementation decision.

---

## Action Naming Structure

All action names in the BASIS ecosystem follow the structure:

```text
{verb}:{domain}[:{object}]
```

**`{verb}`** — the operation being requested. The verb must be a simple, lowercase English word that describes the operation at the operational level of the OT environment: the action a subject is requesting to perform, not the protocol mechanism used to perform it. The verb is required.

**`{domain}`** — the functional domain or system category that owns the resource being acted upon. The domain is required. It should correspond to a meaningful operational category — `hvac`, `access`, `power`, `lighting`, `fire`, `controller`, `audit`, `policy` — not to a protocol name, device model, or vendor category.

**`{object}`** (optional) — the specific kind of resource object within the domain being acted upon. The object qualifier is used when the domain contains multiple resource types that require distinct policy treatment. It should describe the resource kind in operational terms, not in protocol-specific terms.

Examples of well-formed action names:

| Action name | Meaning |
| - | - |
| `read:hvac:setpoint` | Read a setpoint value on an HVAC subsystem resource |
| `write:hvac:setpoint` | Write a setpoint value on an HVAC subsystem resource |
| `execute:hvac:override` | Issue an override command to an HVAC subsystem resource |
| `read:hvac:sensor` | Read a sensor value from an HVAC subsystem resource |
| `read:access:event` | Read an access control event |
| `execute:access:unlock` | Issue an unlock command to an access control resource |
| `write:controller:config` | Write a configuration parameter on a controller |
| `read:power:meter` | Read a value from a power metering resource |
| `execute:fire:suppress` | Issue a suppression command to a fire system resource |
| `browse:controller:point` | Enumerate the points exposed by a controller |
| `read:audit:log` | Read audit records (`audit` is a domain, not a verb) |
| `write:policy:rule` | Write a policy rule to the policy store |

---

## Naming Conventions

**Verbs must be chosen from a controlled set.** The action vocabulary recognizes exactly **five** canonical verbs. This set was confirmed by an ecosystem-wide inventory (see [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md)): these five cover every operation emitted by all nine protocol adapters (BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, Niagara, REST).

- `read` — retrieve the current value, state, or configuration of a resource, without modification
- `write` — modify the value, setpoint, configuration parameter, or state of a resource (this **absorbs** structural configuration changes; there is no separate `configure` verb)
- `execute` — cause the resource to perform an operation or command, as distinct from writing a value (method invocation, select-before-operate / operate, overrides, command points, acknowledgements)
- `browse` — enumerate or navigate the resource or address space without retrieving operational values
- `subscribe` — establish a subscription to ongoing telemetry or event notifications from a resource

**Deprecated and absorbed verbs.** The following earlier or protocol-local terms map onto the canonical set and must not be used in new action names:

| Deprecated verb | Canonical replacement | Notes |
| - | - | - |
| `control` | `execute` | BACnet `CommandValue` and the shared base set used `control`; the majority of adapters used `execute` for the same concept. |
| `command` | `execute` | Used in earlier governance text and console starter-domains. |
| `discover` | `browse` | REST `OPTIONS` used `discover`; OPC UA / Niagara used `browse` for the same concept. |
| `configure` | `write` | Structural configuration is a `write` to a configuration resource. |
| `audit` (as a verb) | `read` with the `audit` **domain** | The kernel models audit access as `read:audit:log`; `audit` is a domain, not a verb. |

`control` and `discover` are established in the `basis-adapters` normalized-request schema and are therefore retained as **deprecated aliases** during a defined deprecation window (see [Deprecation Expectations](#deprecation-expectations)); adapters should update their default maps to emit the canonical verbs, and the aliases should be removed only after the window closes.

**Credential-infrastructure operations are out of scope for this vocabulary.** Enrolling and revoking devices, subjects, or credentials are identity-infrastructure operations, not protocol actions on operational resources. They are deliberately **not** verbs in the operational action vocabulary; if such a contract is needed it belongs to a separate identity/administration vocabulary.

New verbs must not be introduced without Foundation review and an update to this document. Using an unrecognized verb in an action name produces an action that the policy model cannot associate with the recognized operational taxonomy, which degrades policy clarity and auditability.

**Domain names must be lowercase, singular, and operationally meaningful.** Domains should reflect operational categories that an OT operator would recognize: `hvac`, `access`, `lighting`, `power`, `fire`, `controller`, `policy`, `audit`. They must not reflect protocol names (`bacnet`, `modbus`), vendor names, or device model categories.

**Object qualifiers must be lowercase and operationally meaningful.** Object qualifiers should describe the resource type at the operational level: `setpoint`, `sensor`, `override`, `config`, `rule`, `event`. They must not describe protocol-specific object types (`BACnetObject`, `HoldingRegister`, `Topic`).

**The three-part form is preferred over two-part when meaningful distinctions exist.** If a domain contains resource types that warrant distinct policy treatment — an HVAC domain that contains both setpoints and sensors, for which separate policies are appropriate — the object qualifier makes those distinctions expressible without creating separate domains.

---

## Verbs vs. Composite Action Names

A **verb** and a **composite action name** are different things, and several components in the ecosystem currently disagree about which one the `action` field holds. This section makes the contract explicit.

- A **verb** (`read`, `write`, `execute`, `browse`, `subscribe`) names the operational intent. It is one segment.
- A **composite action name** (`read:hvac:setpoint`) is what the **policy engine evaluates** and what the **audit record stores**. It is the verb plus a domain (and optional object), in the `{verb}:{domain}[:{object}]` form — two or more colon-separated segments.

The canonical authorization action is the **composite name**. Policy scoping and audit forensics require the domain; a bare verb cannot be scoped to a resource category and is therefore not a valid authorization action.

Protocol adapters today emit only the **verb** (in `action`) and carry the domain/object separately (in `resource_type` / `resource_id`). That is a legitimate *intermediate* normalization product, but it is **not** a canonical authorization action: a bare verb does not satisfy the `{verb}:{domain}[:{object}]` form the policy engine enforces. Producing the canonical composite therefore requires a single, explicitly-owned **composition rule** — verb (from the adapter) combined with domain/object (from the resource mapping). Defining that rule and assigning its owner (the normalization/gateway boundary, ultimately specified by a future `basis-schemas`) is the open structural item tracked in [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md). Until it exists, adapter output and kernel input remain structurally incompatible even when the verbs agree.

The same `resource_type` field is also the input to **resource identifier** composition: the gateway folds it into the kernel's typed `resource_id` (`ahu` + `rooftop-1` → `ahu:rooftop-1`) at the same boundary it composes the action. The resource-side reconciliation, and the open question of whether the action's `{domain}` and the resource's `{type}` are the same token, are tracked in [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md).

---

## Protocol-Independent Semantics

Action names must represent operational semantics, not transport details. This requirement follows directly from Architecture Principle 5 — Policy Evaluation Independent of Protocol — and from the adapter normalization model.

The normalization responsibility belongs to the protocol adapter, not the action name. The adapter translates the protocol-specific operation into the action name; the action name does not contain or imply any protocol-specific information. This separation is what makes a single policy applicable across multiple protocols.

**BACnet object identifiers and property identifiers are not action names.** A WriteProperty request on BACnet object `Analog-Output 3` is not an action name. The adapter's responsibility is to determine what that write operation means operationally — which might be `write:hvac:setpoint` if the object represents an HVAC control setpoint — and emit the appropriate action name. The action name captures the operational intent; the protocol details remain in the adapter layer.

**Modbus register addresses are not action names.** Writing to Modbus holding register 40001 is not an action name. The adapter maps that register address to the operational concept it represents: `write:hvac:setpoint`, `write:controller:config`, or another appropriate action. The register address is implementation-level detail.

**MQTT topic paths are not action names.** Publishing to `facility/floor3/hvac/zone1/setpoint` is not an action name. The adapter determines the operational semantics from the topic structure and emits the appropriate action name.

Bad action name examples (all violate this rule):

| Bad action name | Problem |
| - | - |
| `BACnet.WriteProperty` | Protocol-specific operation, not operational semantics |
| `Modbus.FC06` | Protocol function code, not an operational concept |
| `mqtt.publish` | Transport mechanism, not an operational concept |
| `AnalogOutput.write` | BACnet object type, not an operational category |
| `HoldingRegister.40001.write` | Register address, not an operational concept |
| `setHvacSetpoint` | Camel-case function name, not the required structure |
| `HVAC_WRITE` | Uppercase with underscore separator, violates naming convention |

---

## Stability Expectations

An action name that has been used in at least one of the following contexts is considered established and subject to the full stability expectations described in [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md):

- A deployed policy rule
- An audit record that has been persisted
- An adapter normalization mapping that is in production use

Once established, an action name must not be removed, renamed, or semantically redefined. The appropriate path for addressing a poorly named or semantically imprecise action name is deprecation followed by the introduction of a replacement under a new name, with a defined transition period during which both names evaluate correctly.

The stability period for action names is operationally long. Audit records may be retained for years. Policies in OT deployments may be authored once and remain in effect for the operational lifetime of the controlled systems — which may be measured in decades. Action names must be designed with that timescale in mind from the point of initial authoring.

---

## Deprecation Expectations

When an action name must be phased out — because it was poorly named, because the operational concept it represented has been split into more precise categories, or because a normalization change makes the prior name inaccurate — the following process applies:

1. A replacement action name is defined that correctly represents the intended operational semantics.
2. The prior action name is marked deprecated in the action vocabulary log, with a reference to the replacement and the reason for the deprecation.
3. The policy engine continues to evaluate the deprecated name for the duration of the deprecation period.
4. Adapters that emitted the deprecated name update to emit the replacement name, with a versioned adapter release.
5. Policy authors are notified of the deprecation and given sufficient lead time to update policies.
6. After the deprecation period ends, the deprecated name may be removed from the vocabulary.

The deprecation period must be long enough to be operationally meaningful in OT contexts. See [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md) for the deprecation philosophy that governs this process.

---

## Reserved Prefixes and Namespaces

Access to the authorization infrastructure itself is governed through **reserved domains**, not through dedicated verbs. The five canonical verbs are sufficient: infrastructure resources are read, written, and acted upon with the same `read` / `write` / `execute` verbs as operational resources, distinguished by their reserved domain.

**Reserved domains:** `policy`, `audit`, `admin` — these domains refer to the authorization infrastructure's own resources and require stricter policy controls than operational domains. Policies governing access to policy management (`read:policy:rule`, `write:policy:rule`), audit records (`read:audit:log`), and administrative functions must use these domains; operational system actions must not. (Earlier drafts of this document modelled `audit` as a verb; it is a **domain**, consistent with the kernel constant `read:audit:log`.)

**Credential lifecycle operations** — enrolling and revoking devices, subjects, or credentials — are identity-infrastructure concerns and are **not** part of this operational action vocabulary. They previously appeared here as the verbs `enroll` and `revoke`; they have been removed. If a contract for them is required, it belongs to a separate identity/administration vocabulary owned alongside the future `basis-schemas` (see [Future Shared Contract Ownership](#future-shared-contract-ownership)).

Third-party adapter implementations that need to represent operations not covered by the current vocabulary should use the form `{verb}:{vendor-domain}:{object}` where `{vendor-domain}` is prefixed with a reverse-DNS-style identifier (e.g., `vendor.acme:device:point`). This prevents collision with the canonical vocabulary while the operational need is evaluated for inclusion in the standard vocabulary.

Requests to add new domains or verbs to the standard vocabulary should be submitted as proposals to the Basis Foundation for review, following the process defined in [`docs/adr/README.md`](../adr/README.md).

---

## Compatibility Implications of Renaming Actions

Renaming an established action name is never a minor change. The following consequences follow directly from any rename:

- **Policies break.** Policies that reference the old name will no longer match requests that use the new name. Depending on the default evaluation behavior (deny for unmatched actions), this produces silent denials rather than errors.
- **Audit records diverge.** Audit records before the rename use the old name; records after the rename use the new name. Cross-period audit queries must be aware of the rename to produce correct results.
- **Normalization coverage gaps.** During any period when adapters have been updated to emit the new name but policies have not been updated to reference it, requests that previously would have produced an allow decision will produce a deny.

For these reasons, renaming an action name is treated as a breaking change under the compatibility philosophy and requires the full deprecation process rather than a direct replacement.

---

## Relationship to Audit Records

Every authorization decision event in an audit record contains the action name that was evaluated. The audit record is an immutable historical record. The action name in a historical audit record cannot be updated retroactively to reflect a later rename.

This creates a permanent record of the action vocabulary state at the time each decision was made. Forensic analysis that spans a rename event must account for the rename explicitly — querying by the old name for records before the rename, and by the new name for records after it.

The audit schema includes an action vocabulary version field so that audit consumers can identify which version of the vocabulary was in effect when a record was produced. This version information is necessary for correct interpretation when the vocabulary has evolved.

---

## Relationship to the Policy Model

The action vocabulary is one half of the semantic contract that policies express. A policy rule specifies: for subjects with property X, on resources with property Y, the action Z is permitted (or denied). The correctness of the policy depends on Z being consistently interpreted: the same action name must map to the same set of protocol operations across all adapters that produce it.

The policy model specification (defined in the basis-core implementation repository) governs how action names are matched against policy rules. This document governs what action names exist and what they mean. The two must be consistent: the policy model must support the action naming structure defined here, and this document must not define action names that the policy model cannot represent.

Policy authors should treat the action vocabulary as a fixed external reference, not as a namespace they can extend locally. Locally invented action names — used in one deployment's policies without being defined in the vocabulary — create normalization mapping obligations that adapters must satisfy consistently. A local action name that a deployment relies on but that is not defined in the canonical vocabulary is a fragile operational dependency that cannot be validated by shared tooling or tested against shared conformance expectations.

---

## Future Shared Contract Ownership

This document is the **authoritative source of truth for the action vocabulary today**. It is, however, only one of several cross-component contracts that are currently defined, mirrored, or re-derived independently across repositories — `basis-core` owns the action *format* regex and the policy-model matching, `basis-adapters` owns the normalized-request schema and the verb enum, `basis-gateway` owns the evaluate request/response shape, and `basis-console` carried a provisional local vocabulary copy during its Phase 6. When the same contract is expressed in several places, the components drift apart; the inconsistencies catalogued in [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md) are the direct result of that drift.

The intended resolution is a dedicated **`basis-schemas`** repository that becomes the single shared home for the contracts every component must agree on. This document does **not** create `basis-schemas` and does not define schemas; it records the ownership model so that, when the repository is created, the boundaries are already clear.

`basis-schemas` should eventually own:

- **The action vocabulary** — the canonical verbs, the `{verb}:{domain}[:{object}]` structure, the domain conventions, reserved domains, and the deprecation register. This governance document would become the human-readable companion to the machine-readable definitions `basis-schemas` publishes.
- **Request schemas** — the normalized authorization request (today: `basis-adapters/schemas/normalized-authorization-request.schema.json`) and the gateway evaluate request (today: `basis-gateway` `EvaluateRequest`), including the **composition rule** that turns an adapter's bare verb plus its resource mapping into a canonical composite action (the open structural item in the reconciliation report).
- **Response schemas** — the decision/evaluate response shape (outcome, reason, policy version, correlation id) shared by `basis-core` and `basis-gateway`.
- **Audit and event schemas** — the audit record shape and its **action-vocabulary version field**, so audit consumers can interpret historical records across vocabulary evolution.
- **Compatibility contracts** — the cross-component versioning and deprecation rules (today described in [`compatibility-philosophy.md`](compatibility-philosophy.md)) that keep adapters, the gateway, the kernel, and the console mutually consistent over time.

Ownership principles for the transition:

1. **One definition, many consumers.** Each contract is defined once in `basis-schemas` and imported elsewhere; no component re-declares a contract it does not own.
2. **No component is the de-facto authority by accident.** In particular, neither `basis-console` nor any single adapter should be the place a vocabulary or schema is effectively decided. `basis-console.vocabulary` is explicitly provisional and should be deleted once `basis-schemas` exists.
3. **Governance stays human-readable here; definitions become machine-readable there.** This document continues to explain *why*; `basis-schemas` publishes the *what* in a form tooling and tests can consume.
