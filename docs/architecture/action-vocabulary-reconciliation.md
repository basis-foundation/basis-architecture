# Action Vocabulary Reconciliation

**Status:** Investigation report and recommendation. Not yet ratified.
**Scope:** Architecture-level review of the action vocabulary as it actually exists across `basis-core`, `basis-gateway`, `basis-adapters`, `basis-console`, and `basis-architecture`.
**Companion document:** [`action-vocabulary.md`](action-vocabulary.md) — the governance document, updated to reflect the recommendation in this report.

This report does not assume any single repository is correct. It inventories what each component actually does, identifies where the components disagree, and recommends a canonical contract for the ecosystem to converge on. A formal ADR (provisionally `docs/adr/0003-action-vocabulary-naming-structure.md`) should be opened to ratify the recommendation in [§5](#5-recommended-canonical-vocabulary).

---

## Executive summary

There are **two independent disagreements** in the ecosystem today, and they are easy to conflate:

1. **A structural disagreement** about what the `action` field *is*. `basis-adapters` emit a **bare verb** (`"read"`) and carry the domain/object in *separate* `resource_type` / `resource_id` fields. `basis-core`, `basis-console`, and `basis-architecture` treat `action` as a **composite string** (`"read:hvac:setpoint"`) with two or more colon-separated segments. An adapter's normalized output therefore does **not** satisfy the kernel's action format, and no component currently performs the composition that would bridge the two.

2. **A verb-set disagreement.** Four components define four different verb sets, and two distinct operational concepts each have multiple competing names: "cause a device to act" is `control` **and** `execute` **and** `command` depending on where you look; "enumerate/navigate" is `discover` **and** `browse`.

The good news: once the two synonym pairs are unified (`control`/`command` → `execute`, `discover` → `browse`), **five verbs cover every operation all nine adapters emit**: `read`, `write`, `execute`, `browse`, `subscribe`. The hypothesis that this is the right canonical set is **supported by the evidence**.

The structural disagreement is the more consequential finding and is the one that currently breaks gateway-backed evaluation of adapter-shaped and console-shaped requests.

---

## 1. What action vocabulary actually exists today?

### 1.1 `basis-adapters` — the only components that actually emit actions

Adapters are where real protocol traffic becomes a normalized authorization request, so their output is the ground truth for "what actions exist."

**Structural fact.** `NormalizedAuthorizationRequest` (`src/basis_adapters/models.py`) carries `action`, `resource_type`, and `resource_id` as **three separate fields**. The docstring and `to_dict()` confirm `action` is a *bare verb* ("Normalized action verb, e.g. `read`, `write`, `control`, `discover`"). The domain/object lives in `resource_type` (`point`, `device`, `schedule`) and `resource_id`. **No adapter composes a `{verb}:{domain}` string.**

**Vocabulary fact.** The shared verb set is `VALID_ACTIONS` in `rest/mapping.py`:

```python
VALID_ACTIONS = frozenset({"read", "write", "control", "discover", "subscribe"})
```

Three adapters extend it additively:

- `VALID_OPCUA_ACTIONS = VALID_ACTIONS | {"execute", "browse"}`
- `VALID_DNP3_ACTIONS = VALID_ACTIONS | {"execute"}`
- `VALID_IEC61850_ACTIONS = VALID_ACTIONS | {"execute"}`
- `VALID_NIAGARA_ACTIONS = VALID_ACTIONS | {"execute", "browse"}`
- `VALID_KNX_ACTIONS = VALID_ACTIONS` (unchanged)

The authoritative adapter contract is the JSON Schema `schemas/normalized-authorization-request.schema.json`, whose `action` enum is:

```json
["read", "write", "control", "discover", "subscribe", "execute", "browse"]
```

**Per-adapter operation → verb inventory** (from each `mapping.py` default map):

| Adapter | Operation → verb (defaults) | Verbs emitted |
| - | - | - |
| **REST** | GET/HEAD/TRACE/CONNECT → `read`; OPTIONS → `discover`; POST/PUT/PATCH/DELETE → `write` | read, write, discover (control valid for explicit routes) |
| **BACnet** | ReadProperty → `read`; WriteProperty → `write`; SubscribeCOV → `subscribe`; **CommandValue → `control`** | read, write, subscribe, **control** |
| **Modbus** | ReadCoils/ReadDiscreteInputs/ReadHoldingRegisters/ReadInputRegisters → `read`; WriteSingle/Multiple Coil(s)/Register(s) → `write` | read, write |
| **OPC UA** | Read → `read`; Write → `write`; **Call → `execute`**; Subscribe → `subscribe`; **Browse → `browse`** | read, write, **execute**, subscribe, **browse** |
| **MQTT** | PUBLISH → `write`; SUBSCRIBE → `subscribe` | write, subscribe |
| **DNP3** | READ → `read`; **SELECT/OPERATE/DIRECT_OPERATE/CONTROL → `execute`**; ENABLE_UNSOLICITED → `subscribe` | read, **execute**, subscribe |
| **IEC 61850** | READ → `read`; WRITE → `write`; **SELECT/SELECT_WITH_VALUE/OPERATE/DIRECT_OPERATE/CANCEL → `execute`**; ENABLE_REPORTING/ENABLE_GOOSE/ENABLE_SAMPLED_VALUES → `subscribe` | read, write, **execute**, subscribe |
| **KNX** | GROUP_VALUE_READ → `read`; GROUP_VALUE_WRITE → `write`; GROUP_VALUE_RESPONSE → `read`; OBSERVE → `subscribe` | read, write, subscribe |
| **Niagara** | READ_* → `read`; WRITE_POINT/WRITE_SLOT/UPDATE_SCHEDULE → `write`; ACK_ALARM/INVOKE_ACTION/COMMAND_POINT/OVERRIDE_POINT/RELEASE_OVERRIDE → `execute`; BROWSE/RESOLVE_ORD/LIST_CHILDREN → `browse`; SUBSCRIBE_* → `subscribe` | read, write, **execute**, **browse**, subscribe |

**Union of verbs actually emitted:** `read`, `write`, `subscribe`, `execute`, `browse`, `control`, `discover`.
Of these, **`control` is emitted by exactly one adapter** (BACnet `CommandValue`) and **`discover` by exactly one** (REST `OPTIONS`). Every other "command/invoke" mapping across OPC UA, DNP3, IEC 61850, and Niagara uses **`execute`**, and every "navigate/enumerate" mapping uses **`browse`**.

### 1.2 `basis-core` — what the kernel will accept

`DecisionRequest.action` (`src/basis_core/decisions/models.py`) is validated by:

```python
_ACTION_RE = re.compile(r"^[a-z][a-z0-9_-]*(:[a-z][a-z0-9_-]*)+$")
```

This requires the composite `{verb}:{domain}[:{object}]` form — **two or more colon-separated lowercase segments**. The regex constrains *shape only*; it does **not** enforce a verb whitelist. The verb taxonomy is documentary: `domain/action.py` declares `verb: read | write | execute | subscribe | command` and ships constants such as `READ_SENSOR_TELEMETRY = "read:sensor:telemetry"`, `EXECUTE_DEVICE_COMMAND = "execute:device:command"`, `READ_AUDIT_LOG = "read:audit:log"`.

Note `READ_AUDIT_LOG = "read:audit:log"`: in the kernel, **`audit` is a domain**, reached with the verb `read`.

### 1.3 `basis-console` — Phase 6 simulator

`basis_console.vocabulary` (provisional, self-described as non-authoritative) composes `{verb}:{domain}` from:

- verbs: `read`, `write`, `execute`, `browse`, `subscribe`
- starter domains: `ahu`, `setpoint`, `telemetry`, `device`, `schedule`, `command`

It mirrors the kernel regex so it never emits a bare verb.

### 1.4 `basis-gateway` — pass-through

`EvaluateRequest.action` (`src/basis_gateway/api/schemas.py`) validates only non-emptiness. The evaluator (`core/evaluator.py`) forwards the string verbatim (`action=action`) into the kernel and never splits, parses, or rewrites it. Its bundled reference policy is keyed on the kernel's composite constants (`read:sensor:telemetry`, …). **The gateway interprets nothing; it forwards.**

---

## 2. What action vocabulary is documented today?

| Source | Structure documented | Verb set documented |
| - | - | - |
| `basis-architecture/docs/architecture/action-vocabulary.md` | `{verb}:{domain}[:{object}]` | read, write, **command**, subscribe, **configure**, **audit**, **enroll**, **revoke** |
| `basis-core` `domain/action.py` | `{verb}:{domain}[:{object}]` | read, write, **execute**, subscribe, **command** |
| `basis-adapters` schema + `normalization-contract.md` | **bare verb** + separate resource | read, write, **control**, **discover**, subscribe, **execute**, **browse** |
| `basis-console` `vocabulary.py` | `{verb}:{domain}` | read, write, **execute**, **browse**, subscribe |

No two components document the same verb set, and the architecture governance document is the **only** place `command`, `configure`, `enroll`, and `revoke` appear — none of which any adapter emits.

---

## 3. Where do inconsistencies exist?

**I-1 — Structural: bare verb vs. composite action (most consequential).**
Adapters emit `action="read"` with the domain in `resource_type`/`resource_id`. The kernel requires `action="read:…"` with ≥2 segments. An adapter's normalized request, submitted as-is, **fails kernel validation** (`validation_failed`). No component composes the adapter's verb + resource into a kernel-valid action. This is the same failure `basis-console` hit in Phase 4/5 and worked around locally in Phase 6. It is an ecosystem gap, not a console bug.

**I-2 — "Cause a device to act" has three names.** `control` (BACnet, REST schema), `execute` (OPC UA, DNP3, IEC 61850, Niagara, console, core docstring), `command` (architecture doc, console starter-domain). Among adapters, `execute` wins decisively (four protocols vs. one).

**I-3 — "Enumerate / navigate" has two names.** `discover` (REST OPTIONS only) vs. `browse` (OPC UA, Niagara, console). `browse` is the richer OT concept (address-space / station traversal) and is used by more components.

**I-4 — `audit` is a verb in one place and a domain in another.** The architecture doc lists `audit` as a verb (`audit:policy:read`); the kernel models it as a domain (`read:audit:log`). These cannot both be canonical.

**I-5 — Governance verbs that nothing emits.** `command`, `configure`, `enroll`, `revoke` appear only in the architecture doc. `enroll`/`revoke` describe credential-infrastructure operations, not protocol actions; `configure` overlaps `write`. Carrying them in the operational action vocabulary violates the project's own "avoid large ontologies / excessive granularity" principle.

**I-6 — `control` and `discover` are declared but barely used.** Both sit in the shared `VALID_ACTIONS` yet each is emitted by a single adapter, while their synonyms (`execute`, `browse`) are emitted by several. The shared base set is internally inconsistent with the adapters' own default maps.

---

## 4. What vocabulary best serves the ecosystem?

Evaluating against the stated design principles — simplicity, protocol neutrality, long-term stability, operator comprehension, adapter consistency; avoiding protocol/vendor-specific verbs, excessive granularity, and large ontologies:

- **Building automation** (BACnet, KNX, Niagara, MQTT): read, write, subscribe, plus command/override invocation and station browsing → covered by `read`, `write`, `subscribe`, `execute`, `browse`.
- **Industrial control** (Modbus, OPC UA, IEC 61850): read, write, method/control invocation, reporting subscriptions → `read`, `write`, `execute`, `subscribe` (+ `browse` for OPC UA address space).
- **Utilities** (DNP3, IEC 61850): read, SBO/operate control, unsolicited/reporting subscriptions → `read`, `execute`, `subscribe`.
- **Protocol neutrality:** the five verbs name *operational intent* (`execute` = "cause the resource to act") rather than any wire mechanism (CROB, Call, CommandValue, INVOKE_ACTION all normalize cleanly to `execute`).
- **Long-term stability / operator comprehension:** a five-verb set is small enough to be memorable and stable across decades of OT deployment; synonyms (`control`/`command`/`discover`) are the main threat to comprehension and are eliminated.

A minimal five-verb set maximizes adapter consistency (every existing mapping already lands on one of the five once synonyms unify) while keeping the ontology small.

---

## 5. Recommended canonical vocabulary

### 5.1 Canonical verbs (five)

| Verb | Definition | Replaces / absorbs |
| - | - | - |
| `read` | Retrieve a value, state, or configuration without modifying it. | — |
| `write` | Modify a value, setpoint, configuration parameter, or state. | absorbs `configure` (structural config is a `write` to a config resource) |
| `execute` | Cause the resource to perform an operation or command, as distinct from writing a value (method calls, SBO/operate, overrides, acknowledgements). | **`control`**, **`command`** (deprecated aliases) |
| `browse` | Enumerate or navigate the resource/address space without retrieving operational values. | **`discover`** (deprecated alias) |
| `subscribe` | Establish an ongoing subscription to telemetry or event notifications. | — |

These five cover every operation emitted by all nine adapters. **This confirms the hypothesised set `read / write / execute / browse / subscribe`.**

### 5.2 Canonical structure

The canonical **authorization action** — the value the policy engine evaluates and the audit record stores — remains the composite `{verb}:{domain}[:{object}]` (≥2 lowercase colon-separated segments), because policy scoping and audit forensics need the domain, and the kernel already enforces this shape.

The adapter's bare `action` verb is **one input to that composite**, not the composite itself. The ecosystem must define a single, explicit **composition rule** — verb (from the adapter) + domain/object (from the resource mapping) → canonical action — and assign an owner for it (recommended: the normalization/gateway boundary, ultimately specified by `basis-schemas`). Until that rule exists, adapter output and kernel input remain structurally incompatible (I-1). This report recommends closing that gap as the **highest-priority** follow-up; the verb reconciliation above is necessary but not sufficient on its own.

### 5.3 Domains and infrastructure verbs

- `audit` is a **domain**, not a verb (align with the kernel: `read:audit:log`). Remove it from the verb set.
- `policy`, `audit`, `admin` remain **reserved domains** for authorization-infrastructure resources.
- `enroll` / `revoke` are **credential-infrastructure** operations, not protocol actions. They are removed from the operational action vocabulary; if needed they belong to a separate identity/admin contract, not the protocol-action set.

### 5.4 Migration notes (non-breaking path)

`control` and `discover` are established in the adapter schema and so are subject to stability expectations. They become **deprecated aliases** of `execute` and `browse` respectively: continue to validate during a defined deprecation window, update adapter default maps to emit the canonical verbs, and remove the aliases only after the window closes. This follows the deprecation process already defined in [`action-vocabulary.md`](action-vocabulary.md) and [`compatibility-philosophy.md`](compatibility-philosophy.md).

---

## 6. Documents updated by this work

- **This report** — `docs/architecture/action-vocabulary-reconciliation.md` (new).
- **[`action-vocabulary.md`](action-vocabulary.md)** — canonical verb set reduced to the five recommended verbs with definitions; `control`/`command`/`discover`/`configure` recorded as deprecated/absorbed; `audit` reclassified as a domain; `enroll`/`revoke` removed from the operational verb set; examples updated; a "Verbs vs. composite action names" clarification and a "Future Shared Contract Ownership" section added.

No code was changed. A formal ADR should be opened to ratify §5.
