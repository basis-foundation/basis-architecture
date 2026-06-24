# Ecosystem Contract Inventory

**Status:** Inventory. Records contracts that exist in practice today; does not ratify, formalize, or change them.
**Scope:** A cross-repository inventory of the contracts now implemented across `basis-core`, `basis-gateway`, `basis-adapters`, and `basis-console`, recorded ahead of the future `basis-schemas` repository.
**Companion documents:** [`action-vocabulary.md`](action-vocabulary.md), [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md), [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md), [`compatibility-philosophy.md`](compatibility-philosophy.md), [`../glossary.md`](../glossary.md).

---

## 1. Purpose

BASIS now has real contracts implemented across multiple repositories. This inventory records those contracts before they are formalized in `basis-schemas`.

The ecosystem's division of responsibility has stabilized through implementation rather than declaration:

```text
Adapters normalize.
Gateway composes and enforces.
Core evaluates.
Console operates.
```

Several cross-repository contracts that follow from that division are now implemented and exercised by tests in their owning repositories — an action vocabulary, action composition, resource-identifier composition, gateway request shapes, the adapter normalized-request shape, the console's normalized submission, audit evidence, and a reserved gateway evidence namespace. They are currently expressed in whichever repository implements them, and in several cases mirrored or re-derived in more than one place.

This document inventories those contracts so that the future `basis-schemas` repository can be designed from what has actually been proven in implementation, rather than speculatively. For each contract it records the **current owner**, the **repositories that implement it**, its **current stability**, whether it is a likely **future `basis-schemas` candidate**, and the **open questions** still attached to it. It does not create `basis-schemas`, define any JSON Schema, or resolve any of the open questions it records.

This is an inventory, not a roadmap. Where a contract is unsettled, the entry says so and defers it; it does not propose a resolution.

---

## 2. Contract Inventory Table

Stability labels used below:

- **stable** — implemented, covered by contract tests, and not expected to change shape in the near term.
- **emerging** — implemented and converging across components, but still settling; some details remain in flux.
- **provisional** — implemented as a deliberate placeholder or bridge, explicitly expected to be replaced.
- **open** — a contract whose shape is recorded but not yet agreed across components.

| Contract | Current Owner | Implemented In | Current Stability | Future `basis-schemas` Candidate | Open Questions |
| - | - | - | - | - | - |
| Action vocabulary (canonical verbs) | `basis-architecture` | `basis-core`, `basis-gateway`, `basis-adapters`, `basis-console` | emerging | Yes | Which verb-set drift remains across adapter schemas vs. governance doc; deprecation of `control`/`discover` aliases. |
| Action string format (`{verb}:{domain}[:{object}]`) | `basis-core` | `basis-core`, `basis-gateway`, `basis-console` | stable | Yes | Whether the `{domain}` segment is the same token as the resource `{type}`. |
| Gateway-owned action composition | `basis-gateway` (architectural owner: `basis-architecture`) | `basis-gateway` | emerging | Yes | Source of the `{domain}`/`{object}` segments when they must differ from `resource_type`. |
| Resource identifier format (`{type}:{qualifier}`) | `basis-core` | `basis-core`, `basis-gateway` | stable | Yes | Resource-type vocabulary not unified across components. |
| Gateway-owned resource identifier composition | `basis-gateway` | `basis-gateway` | emerging | Yes | Already-typed local identifiers; qualifier charset vs. composition; `resource_type` dual role. |
| Adapter normalized request shape | `basis-adapters` | `basis-adapters` | stable | Yes | Verb enum still carries `control`/`discover`; `resource_type` vocabulary. |
| Console gateway request shape | `basis-console`, `basis-gateway` | `basis-console`, `basis-gateway` | emerging | Yes | Dual-purpose `resource_type` (shared with the composition entries); none specific to the console's own submission. |
| Decision request contract | `basis-core` | `basis-core`, `basis-gateway` | stable | Yes | — |
| Decision response contract | `basis-core` | `basis-core`, `basis-gateway` | stable | Yes | — |
| Audit event contract | `basis-core`, `basis-gateway` | `basis-core`, `basis-gateway` | emerging | Yes | Persistence; tamper resistance; gateway/kernel audit correlation. |
| Reserved gateway evidence namespace (`basis_gateway.*`) | `basis-gateway` | `basis-gateway` | emerging | Likely | Whether namespace rules should be schema-enforced. |
| Readiness contract | `basis-gateway` | `basis-gateway` | provisional | Maybe | Whether a standardized cross-component shape is warranted. |
| Policy loading contract | `basis-gateway`, `basis-core` | `basis-gateway`, `basis-core` | provisional | Maybe | Policy provenance and lifecycle (see §4). |

---

## 3. Contracts To Include

Each entry below records what is implemented today, who owns it, and what remains open. The entries do not restate the full reconciliation analyses; they point to them.

### 3.1 Action vocabulary

**Contract.** The canonical operational verbs:

```text
read
write
execute
browse
subscribe
```

These five cover every operation emitted by all nine protocol adapters, confirmed by the ecosystem-wide inventory in [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md).

**Current owner:** `basis-architecture` ([`action-vocabulary.md`](action-vocabulary.md) is the authoritative source of truth for the vocabulary today).

**Implemented in:** `basis-core` (documentary verb taxonomy in `domain/action.py`), `basis-gateway` (forwards composite actions built on these verbs), `basis-adapters` (verb enum in the normalized-request schema and per-adapter mapping tables), `basis-console` (provisional `vocabulary.py`).

**Current stability:** emerging. The governance document settles the five canonical verbs, but the `basis-adapters` schema enum still carries the deprecated aliases `control` and `discover`, and the console copy is provisional. Convergence is in progress, not complete.

**Future `basis-schemas` candidate:** Yes.

**Open questions:** the migration of `control`/`discover` to their canonical replacements; whether the vocabulary becomes a machine-readable enum in `basis-schemas` with this document as its human-readable companion.

### 3.2 Action string format

**Contract.** The kernel-compatible composite action name:

```text
{verb}:{domain}
{verb}:{domain}:{object}
```

Two or more lowercase colon-separated segments. Implemented as `_ACTION_RE` in `basis-core` (`decisions/models.py`). The regex constrains shape only; it does not enforce the verb whitelist (that is the vocabulary contract, §3.1).

**Current owner:** `basis-core`.

**Implemented in:** `basis-core` (the format regex and policy matching), `basis-gateway` (produces and forwards values in this form), `basis-console` (mirrors the form so it never emits a bare verb).

**Current stability:** stable.

**Future `basis-schemas` candidate:** Yes.

**Open questions:** whether the `{domain}` segment and the resource `{type}` prefix are the same token — recorded in §7 and in [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) §9.

### 3.3 Gateway-owned action composition

**Contract.** The gateway combines a bare adapter verb with a resource type into a kernel-compatible composite action:

```text
input:  action + resource_type
output: action:resource_type      (e.g. read + ahu → read:ahu)
```

Implemented as `compose_action()` in `basis-gateway` (`core/actions.py`). When composition occurs the gateway records reserved-namespace evidence (§3.11). A request that already carries a composite action plus a `resource_type` is rejected as ambiguous, mirroring the resource-side rule (§3.5).

**Current owner:** `basis-gateway`.
**Architectural owner:** `basis-architecture` (the boundary decision is documented in [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md)).

**Implemented in:** `basis-gateway`.

**Current stability:** emerging. The mechanism is implemented and tested; the boundary decision has not yet been ratified by ADR.

**Future `basis-schemas` candidate:** Yes — the composition rule is named in [`action-vocabulary.md`](action-vocabulary.md) as a `basis-schemas`-owned contract.

**Open questions:** where the `{domain}`/`{object}` segments come from when they must differ from `resource_type`.

### 3.4 Resource identifier format

**Contract.** The canonical typed resource identifier:

```text
{type}:{qualifier}
```

Example:

```text
ahu:rooftop-1
```

A type-prefix segment, a colon, and at least one qualifier. Implemented as `_RESOURCE_ID_RE` in `basis-core` (`decisions/models.py`). The kernel derives the resource type from the prefix; there is no separate `resource_type` input to evaluation. `None` is permitted for resource-independent requests.

**Current owner:** `basis-core`.

**Implemented in:** `basis-core` (format regex, `parse_resource_id()`/`build_resource_id()`, prefix-derived typing in policy rules), `basis-gateway` (produces values in this form via composition).

**Current stability:** stable.

**Future `basis-schemas` candidate:** Yes.

**Open questions:** the resource-type vocabulary is not unified across components (see §4 and [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) M-4).

### 3.5 Gateway-owned resource identifier composition

**Contract.** The gateway combines a resource type with a local resource identifier into the canonical typed identifier:

```text
input:  resource_type + local resource_id
output: resource_type:local_resource_id
```

Example:

```text
ahu + rooftop-1 → ahu:rooftop-1
```

Implemented as `compose_resource_id()` in `basis-gateway` (`core/resources.py`), at the same boundary and from the same `resource_type` field as action composition, so the composed action and composed identifier are consistent by construction. The boundary detects already-typed identifiers and rejects ambiguous or inconsistent combinations; composition records reserved-namespace evidence (§3.11).

**Current owner:** `basis-gateway`.

**Implemented in:** `basis-gateway`.

**Current stability:** emerging. Implemented and tested; boundary decision not yet ratified by ADR.

**Future `basis-schemas` candidate:** Yes.

**Important note (recorded, not resolved):**

```text
resource_type currently remains dual-purpose: action domain and resource type.
```

The gateway derives both the action's `{domain}` segment and the resource identifier's `{type}` prefix from one `resource_type` field. The kernel's own examples treat these as the same token, but its action-domain vocabulary and its resource-type set are not identical lists. This inventory records that as an open schema question (§7), not a resolved one.

### 3.6 Adapter normalized request shape

**Contract.** Adapters emit a normalized authorization request carrying:

```text
action            (bare verb)
resource_type     (local category, e.g. point / device / schedule)
resource_id       (local, untyped identifier rendered from a template)
context/evidence
```

Adapters do **not** compose kernel-compatible composite actions or canonical resource identifiers. Their output is a normalization input, not a kernel-canonical artifact. Implemented as `NormalizedAuthorizationRequest` in `basis-adapters` (`models.py`) and `schemas/normalized-authorization-request.schema.json`.

**Current owner:** `basis-adapters`.

**Implemented in:** `basis-adapters` (across all nine adapters).

**Current stability:** stable. The separate-field emission model is endorsed by the resource reconciliation as the correct normalization input.

**Future `basis-schemas` candidate:** Yes.

**Open questions:** the verb enum still carries the deprecated `control`/`discover` aliases (§3.1); the adapter `resource_type` vocabulary differs from the kernel's (§4).

### 3.7 Console gateway request shape

**Contract.** The console submits **normalized** gateway requests and lets the gateway compose; it no longer pre-composes. The preferred shape carries a bare verb, a `resource_type`, and a **local** resource identifier:

```json
{
  "action": "read",
  "resource_type": "ahu",
  "resource_id": "rooftop-1"
}
```

The direct typed shape — an already-composite action and an already-typed identifier, with **`resource_type` omitted** — remains valid:

```json
{
  "action": "read:ahu",
  "resource_id": "ahu:rooftop-1"
}
```

Two further cases follow from the same rules and are implemented:

- **`resource_type` without `resource_id`** is valid for a domain-level request: the gateway composes the action (`read:ahu`) and the kernel evaluates it with no resource identifier.
- A `resource_type` must **not** be sent alongside an already-typed `resource_id`; `resource_type` is omitted precisely when the identifier is already typed. The console refuses that combination before any call, and the gateway rejects it at the composition boundary — so the two shapes above are never mixed for one evaluation.

**Current owner:** `basis-console` and `basis-gateway` jointly (the console produces the request; the gateway defines what it accepts).

**Implemented in:** `basis-console` (the request builder `build_gateway_request` and gateway-backed simulation), `basis-gateway` (`EvaluateRequest` and the composition boundary).

**Current stability:** emerging. The console's submission is **decided**: it has aligned with gateway-owned composition and no longer pre-composes. The entry stays "emerging" only because the shared `resource_type` field is still dual-purpose ecosystem-wide (§3.5), not because the console's own behavior is unsettled.

**Future `basis-schemas` candidate:** Yes.

**Open questions:** none specific to the console's submission. The only carried question is the ecosystem-wide dual-purpose `resource_type` (action domain vs. resource type), tracked in §7 and [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) §9, and not resolved here.

### 3.8 Decision request contract

**Contract.** The kernel evaluation request: subject, action (composite form, §3.2), optional resource identifier (canonical form, §3.4), and context. Implemented as `DecisionRequest` in `basis-core` (`decisions/models.py`).

**Current owner:** `basis-core`.

**Implemented in:** `basis-core` (the contract), `basis-gateway` (constructs it from the evaluate request after composition).

**Current stability:** stable.

**Future `basis-schemas` candidate:** Yes.

**Open questions:** none specific to the request shape beyond the action/resource open questions above.

### 3.9 Decision response contract

**Contract.** The kernel decision response: an explicit outcome (never a bare string), a non-empty reason, the evaluating policy name, the policy version, an optional failure reason for enforcement-boundary denials, and a timezone-aware timestamp. Implemented as `DecisionResponse` in `basis-core` (`decisions/models.py`).

**Current owner:** `basis-core`.

**Implemented in:** `basis-core` (the contract), `basis-gateway` (returns it to callers).

**Current stability:** stable.

**Future `basis-schemas` candidate:** Yes.

**Open questions:** none specific to the response shape today.

### 3.10 Audit event contract

**Contract.** The canonical audit event shape: a schema version, event type and outcome, subject/action/resource fields (including an optional `resource_type`), the policy version in effect, and timing. Implemented as `AuditEvent` in `basis-core` (`audit/events.py`), with the gateway as a producer of audit records around evaluation.

**Current owner:** `basis-core` (event definition) and `basis-gateway` (gateway-side audit production).

**Implemented in:** `basis-core`, `basis-gateway`.

**Current stability:** emerging. The event shape is defined and versioned in the kernel; the surrounding pipeline is not settled.

**Future `basis-schemas` candidate:** Yes.

**Open questions:**

```text
audit persistence
tamper resistance
gateway/kernel audit correlation
```

### 3.11 Reserved gateway evidence namespace

**Contract.** A reserved evidence namespace:

```text
basis_gateway.*
```

**Purpose:**

```text
prevent caller-supplied evidence from overwriting gateway-owned composition/audit metadata
```

The gateway writes composition metadata under this prefix (e.g. the original action, the original local resource identifier, the resource type, and the composed values) and rejects caller-supplied context keys that collide with it. Implemented in `basis-gateway` (`core/actions.py` and `core/resources.py`, which share the prefix and the `resource_type` evidence key).

**Current owner:** `basis-gateway`.

**Implemented in:** `basis-gateway`.

**Current stability:** emerging.

**Future `basis-schemas` candidate:** Likely. The namespace rule is a cross-component contract (callers must not write reserved keys), which is the kind of rule `basis-schemas` could enforce; whether it should be schema-enforced is an open question (§7).

### 3.12 Readiness contract

**Contract.** A gateway readiness signal composed from per-component status: the gateway is ready only when all registered components are ready, and it can report the not-ready reason(s). Implemented as `ReadinessState` in `basis-gateway` (`readiness.py`).

**Current owner:** `basis-gateway`.

**Implemented in:** `basis-gateway`.

**Current stability:** provisional — a gateway-internal operational signal, not a published cross-component contract.

**Future `basis-schemas` candidate:** Maybe. This entry deliberately does not over-specify; a standardized readiness response across components may or may not be warranted (§7).

### 3.13 Policy loading contract

**Contract.** The gateway loads a policy engine at startup from a configured policy source, and the kernel exposes the policy version that evaluation and audit records reference. Implemented via `load_policy_engine()` in `basis-gateway` (`policy/loader.py`) and the policy-version fields in the `basis-core` decision and audit contracts.

**Current owner:** `basis-gateway` (loading) and `basis-core` (policy version semantics).

**Implemented in:** `basis-gateway`, `basis-core`.

**Current stability:** provisional. Loading works; policy provenance (where a policy came from and how its version is established) is not yet a formalized contract.

**Future `basis-schemas` candidate:** Maybe.

**Open questions:** policy provenance and the broader policy-lifecycle questions deferred in §4.

---

## 4. Contracts Not Yet Ready For basis-schemas

The following remain **architecture questions, not schema contracts**. They are not yet implemented as agreed cross-component contracts, and formalizing them in `basis-schemas` now would be speculative. They are listed so that `basis-schemas` is scoped to what exists, not to these:

```text
resource taxonomy
action-domain vs resource-type distinction
audit persistence
deployment topology
multi-site trust
policy lifecycle management
operator workflow semantics
```

- **Resource taxonomy.** The set of valid resource types (and whether it is a closed enum or an open prefix) is not unified across adapters, kernel, and console. See [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) M-4.
- **Action-domain vs resource-type distinction.** Whether the action `{domain}` and the resource `{type}` are one concept or two is unresolved; the gateway currently derives both from one field. See [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) §9 item 1.
- **Audit persistence, tamper resistance, correlation.** The audit *event shape* exists (§3.10); the *pipeline* — durable storage, immutability guarantees, and gateway/kernel record correlation — does not yet have an agreed contract.
- **Deployment topology and multi-site trust.** How components are deployed, and how trust is established across sites, are architecture concerns with no implemented cross-repository contract.
- **Policy lifecycle management.** Authoring, distribution, versioning beyond a single version field, and retirement of policies are not yet contracted.
- **Operator workflow semantics.** What an operator can do through the console, and how those workflows map onto enforcement boundaries, are described architecturally but not as a schema.

These belong in architecture documents and ADRs until implementation proves a stable shape, at which point they may join the inventory above.

---

## 5. Candidate basis-schemas Scope

The following are the **likely** future `basis-schemas` candidates drawn from §3. This is a candidate list, not an implementation plan, and not a commitment to a particular order or shape:

```text
Action vocabulary
Action string schema
Resource identifier schema
Normalized request schema
Decision request schema
Decision response schema
Audit event schema
Reserved evidence namespace rules
Schema versioning rules
Compatibility snapshots
```

Each of these is already implemented in at least one repository today (§3), which is precisely why it is a candidate: `basis-schemas` would consolidate the single definition and let the other repositories import rather than re-derive it. Which of these `basis-schemas` should own *first*, and how versioning and compatibility snapshots are structured, are open questions (§7), not decisions made here. The charter for that future repository — its scope, non-scope, ownership model, and a suggested migration order — is developed in [`basis-schemas.md`](basis-schemas.md).

---

## 6. Non-Goals

This document does **not**:

- create `basis-schemas`;
- define final JSON Schemas or any machine-readable schema;
- change any implementation repository (`basis-core`, `basis-gateway`, `basis-adapters`, `basis-console` are referenced only);
- resolve the resource taxonomy;
- resolve the action-domain vs resource-type semantics;
- define policy authoring;
- define deployment topology;
- define commercial BASAuth behavior.

It also does not ratify any boundary decision; the action- and resource-composition boundaries are recorded here as implemented, and their ratification remains with the ADRs proposed in the reconciliation reports.

---

## 7. Open Questions

- Should the action domain and the resource type split into separate fields, rather than both being derived from one `resource_type`? (See [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) §9 item 1.)
- Which contracts should become versioned schemas first when `basis-schemas` is created?
- Should the reserved gateway evidence namespace rules (`basis_gateway.*`, §3.11) become schema-enforced, rather than enforced only by gateway code?
- Should readiness responses (§3.12) be standardized across components, or remain a gateway-internal operational signal?
- How should future schema compatibility be governed — what counts as a breaking change to a shared schema, and how are compatibility snapshots structured? (See [`compatibility-philosophy.md`](compatibility-philosophy.md).)
- What belongs in `basis-schemas` versus remaining repository-owned? Some contracts (e.g. the decision request/response) are intrinsic to the kernel; the boundary between "shared schema" and "kernel-owned contract imported by others" is itself unsettled.

---

## 8. Relationship to other documents

This inventory is descriptive: it records the contracts as they exist across repositories at the time of writing. The analyses behind two of the most consequential entries live in their own reports — [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md) (action vocabulary and action composition) and [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) (resource identifier and its composition) — and the action vocabulary's governance rules live in [`action-vocabulary.md`](action-vocabulary.md). Where this inventory and those documents differ in detail, those documents are authoritative for their respective contracts; this one is the cross-cutting index. When `basis-schemas` is created, this inventory is the list of what it has to account for.
