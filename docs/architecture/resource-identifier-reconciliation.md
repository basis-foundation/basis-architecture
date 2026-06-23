# Resource Identifier Reconciliation

**Status:** Investigation report and recommendation. Not yet ratified.
**Scope:** Architecture-level review of how resource identity is represented across `basis-core`, `basis-gateway`, `basis-adapters`, `basis-console`, and `basis-architecture`, and a recommendation for where canonical resource-identifier composition belongs.
**Companion work:** [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md) settled the parallel question for the `action` field. This report applies the same method to the resource fields and arrives at a structurally identical boundary decision.

This report does not assume any single repository is correct. It inventories what each component actually does with `resource_type` and `resource_id`, identifies where the components disagree, compares candidate models, and recommends a canonical contract for the ecosystem to converge on. A formal ADR (provisionally `docs/adr/0004-gateway-owned-resource-identifier-composition.md`) should be opened to ratify the recommendation in [§5](#5-recommended-canonical-model).

---

## Executive summary

Action vocabulary reconciliation established a clear action boundary: adapters emit a bare verb plus a `resource_type`, the gateway composes `verb:resource_type` into a kernel-compatible composite action, and the kernel evaluates the composite. That work resolved the `action` field. It did **not** resolve the **resource** fields, and it left a matching gap.

The remaining mismatch is narrow and concrete:

- **`basis-adapters`** emit `resource_type` and `resource_id` as **two separate fields**, where `resource_id` is a **local, untyped identifier** rendered from an operator-configured template (e.g. `resource_type = "ahu"`, `resource_id = "rooftop-1"`).
- **`basis-core`** does not carry a `resource_type` field at all. Its `DecisionRequest.resource_id` must be a **single typed/structured string** whose **type lives in the identifier itself as a prefix** (e.g. `ahu:rooftop-1`). The kernel **derives** the resource type by splitting on the first colon.
- **`basis-gateway`** already composes the *action* from `resource_type`, but **passes `resource_id` through unchanged**. A bare adapter `resource_id` (`rooftop-1`) therefore fails the kernel's resource-identifier format and the request is rejected (`validation_failed`).
- **`basis-console`** sidesteps the problem by requiring a human to type an **already-typed** `resource_id` (e.g. `hvac:zone-a`) and treats its separate `resource_type` form field as descriptive only — a field that, in current sample data, sometimes **disagrees** with the identifier's own prefix.

No component composes the adapter's `resource_type` + local `resource_id` into the kernel's typed `resource_id`. This is the same shape of gap the action work found, and the recommended resolution is the same: **the gateway is the composition boundary** — concretely, **Model C realized through Model D's dual-accept/reject boundary** ([§5](#5-recommended-canonical-model)). The gateway already holds `resource_type` at the moment it composes the action; composing the canonical `resource_id` from the same field at the same point closes the gap and keeps the two composed outputs consistent by construction.

A secondary finding, raised but not resolved here: the gateway currently reuses one `resource_type` field to fill **both** the action's `{domain}` segment **and** (under this recommendation) the resource identifier's `{type}` prefix. The kernel's own examples treat those as the same token, but its action-domain vocabulary and its `ResourceType` set are not identical lists. Whether "action domain" and "resource type" are one concept or two is the deeper contract question this reconciliation exposes; it is recorded in [§9](#9-open-questions) for the future `basis-schemas` and the follow-up ADR, and is explicitly **not** decided here.

---

## 1. Problem statement

The ecosystem agrees that an authorization decision targets a *resource*, but it does not agree on how that resource is **named on the wire** between the component that observes the operation (the adapter, or a human in the console) and the component that evaluates it (the kernel).

Two representations are in use:

```text
Separate fields (adapters, console form):     Single typed string (kernel):
  resource_type = ahu                            resource_id = ahu:rooftop-1
  resource_id   = rooftop-1
```

The kernel cannot consume the separate-field form: its `resource_id` must already carry the type as a prefix, because the kernel derives the resource type *from* that prefix during policy evaluation and audit. Nothing in the request path turns `("ahu", "rooftop-1")` into `"ahu:rooftop-1"`. The action work built exactly this bridge for the verb but not for the identifier, so an adapter-normalized request can now produce a valid composite action (`read:ahu`) while still carrying a resource identifier (`rooftop-1`) the kernel rejects.

The question this report answers is: **what is the canonical resource-identifier contract, and which component owns turning local resource fields into it?**

Four terms are kept distinct throughout this document, and conflating them is the root of the mismatch:

- **Local resource identifier** — the *untyped* identifier produced upstream of the composition boundary, e.g. the `rooftop-1` rendered by an adapter's `resource_id_template`. It is a normalization input, not a kernel-valid value on its own.
- **Canonical resource identifier** — the single *typed, structured* `resource_id` the kernel evaluates and audits: `{type}:{qualifier[:{subqualifier}...]}`, e.g. `ahu:rooftop-1`. It is the only form the kernel accepts.
- **Resource type** — the classification of the resource (e.g. `ahu`, `sensor`, `device`). In the canonical identifier it is the *prefix*, derived by the kernel; upstream it is carried as a separate `resource_type` field.
- **Action domain** — the `{domain}` segment of a composite action name (`read:`**`hvac`**`:setpoint`). It is a property of the *action*, not the resource. The gateway currently fills it from the same `resource_type` field, which is why the two can be conflated — see M-5 ([§3](#3-contract-mismatch)) and open question 1 ([§9](#9-open-questions)).

---

## 2. Current behavior by repository

### 2.1 `basis-core` — typed, structured, self-describing; type derived from the prefix

The kernel does **not** model `resource_type` as an input to evaluation. `DecisionRequest` (`src/basis_core/decisions/models.py`) carries `resource_id: str | None` only. When present it must match:

```python
_RESOURCE_ID_RE = re.compile(r"^[a-z][a-z0-9_-]*(:[a-z0-9][a-z0-9_:/-]*)$")
```

This requires a **type-prefix segment, a colon, and at least one qualifier** — i.e. the identifier must be typed and contain a colon. `None` is permitted for resource-independent requests. A bare token such as `rooftop-1` (no colon) is invalid.

The type is **derived from the identifier**, not supplied alongside it. `domain/resource.py` provides `parse_resource_id()` (`"sensor:co2:lobby" → ("sensor", ["co2","lobby"])`) and `build_resource_id()`, and `ResourceTypePolicyRule` (`policy/rules.py`) computes the type as `resource_id.split(":")[0]`. So in the kernel the resource identifier is **structured and meaningful, not opaque**: policy scoping and audit both read the type out of the prefix.

The full `Resource` domain object additionally constrains its `type` to a closed `ResourceType` enum (`hvac`, `sensor`, `zone`, `device`, `gateway`). That enum governs the rich `Resource` model, **not** `DecisionRequest.resource_id`, which is validated by the regex alone (any lowercase prefix passes). The kernel's `AuditEvent` (`audit/events.py`) records `resource_id` **and** an optional `resource_type` as separate fields — but evaluation never depends on a supplied `resource_type`; it derives it.

**Summary:** the kernel expects a single, typed, structured `resource_id`. It is globally meaningful (stable, appears in policy and audit) and self-describing (the type is the prefix). It is not opaque, and it is not a separate-field pair.

### 2.2 `basis-gateway` — composes the action, passes the identifier through

`EvaluateRequest` (`src/basis_gateway/api/schemas.py`) accepts `action`, optional `resource_type`, optional `resource_id`, and `context`. The gateway's composition boundary (`core/actions.py`) uses `resource_type` to compose the **action** — `compose_action("read", "ahu") → "read:ahu"` — and records `basis_gateway.*` evidence when it does so.

It does **not** compose the resource identifier. The evaluator (`core/evaluator.py`) forwards `resource_id` to the kernel **verbatim** (`resource_id=resource_id`). The schema docstring documents resource_type's role in *action* composition and is silent on resource_id composition.

**Consequence:** a request with `action="read"`, `resource_type="ahu"`, `resource_id="rooftop-1"` yields a valid composite action (`read:ahu`) but an unchanged, untyped `resource_id` (`rooftop-1`) that fails the kernel's `_RESOURCE_ID_RE`. Only requests whose `resource_id` is *already* typed survive. The gateway holds the field needed to fix this (`resource_type`) at the exact point it composes the action, but does not yet apply it to the identifier.

### 2.3 `basis-adapters` — separate `resource_type` and local `resource_id`

`NormalizedAuthorizationRequest` (`src/basis_adapters/models.py`) carries three independent fields: `action` (bare verb), `resource_type` (e.g. `"point"`, `"device"`, `"schedule"`), and `resource_id`. The JSON Schema (`schemas/normalized-authorization-request.schema.json`) requires all three.

Crucially, `resource_id` is **rendered from an operator-configured `resource_id_template`** (`resolve_resource_id()` in `rest/mapping.py`, and equivalents in the other adapters): `route.resource_id_template.format_map(captures)`. The template can render anything the operator configures — typically a **local** identifier such as `rooftop-1` — and is **not** guaranteed to be type-prefixed. The `resource_type` is a sibling field, not a prefix of `resource_id`.

**Summary:** adapters emit resource type and a *local* resource ID **separately**. They do not own, and do not perform, kernel-specific identifier composition. This matches the action finding exactly: adapters emit normalization inputs, not kernel-canonical artifacts.

### 2.4 `basis-console` — human types an already-typed identifier; `resource_type` is descriptive

The simulator (`ui/views.py`, `simulator.py`, `ui/templates/simulate.html`) exposes **both** `resource_id` and `resource_type` as free-text form fields. In practice the console expects the human to type an **already-composite** `resource_id`: sample data uses `hvac:zone-a`, `ahu:rooftop-1:supply-temp`, `device:restricted-controller`.

The separate `resource_type` field is **descriptive/display only** and in current sample data **sometimes disagrees** with the identifier's own prefix — e.g. `resource_id = "ahu:rooftop-1:supply-temp"` with `resource_type = "sensor"`, and `resource_id = "hvac:zone-3:setpoint"` with `resource_type = "actuator"`. When the console submits a gateway-backed evaluation it forwards **`action` and `resource_id` only** (`views.py`); it does **not** send `resource_type` and does **not** compose. It relies on the human having pre-typed a kernel-valid identifier.

**Summary:** the console constructs the typed identifier by hand (via the operator) rather than from separate type/id fields, and carries a second, non-authoritative `resource_type` label that can drift from the identifier it describes.

### 2.5 `basis-architecture` — describes "resource identifier", not the composition rule

The architecture docs consistently say the kernel "receives a subject, an action, and a **resource identifier**" and that adapters "normalize … against **resource identifiers** derived from register/topic/path mappings" ([`basis-adapters.md`](basis-adapters.md)). They describe the identifier as a normalized, protocol-independent string but do **not** state where a separate `resource_type` plus a local `resource_id` becomes that single identifier. That omission is the gap this document fills.

---

## 3. Contract mismatch

**M-1 — Field shape: separate pair vs. single typed string (most consequential).**
Adapters (and the console form) model resource identity as `(resource_type, resource_id)` with a **local** `resource_id`. The kernel models it as one **typed** `resource_id` with no separate type field. Submitted as-is, an adapter-normalized resource identifier **fails kernel validation**. No component performs the composition that would bridge the two. This is the resource-side twin of action reconciliation finding I-1.

**M-2 — Type is supplied vs. derived.** Adapters and the console *supply* a `resource_type` next to the identifier. The kernel *derives* the type from the identifier's prefix. These are different sources of truth for the same fact. When both exist independently they can disagree (see M-3).

**M-3 — The console's `resource_type` already drifts from the prefix.** Live sample data pairs `ahu:rooftop-1:supply-temp` with `resource_type = "sensor"` and `hvac:zone-3:setpoint` with `resource_type = "actuator"`. Whatever the canonical model, a representation that carries the type both *in* the identifier and *beside* it invites exactly this inconsistency.

**M-4 — Resource-type vocabularies do not align across components.** Adapter `resource_type` examples are `point` / `device` / `schedule`; the kernel `ResourceType` enum is `hvac` / `sensor` / `zone` / `device` / `gateway`; the action-vocabulary domains are `hvac` / `access` / `power` / `lighting` / `fire` / `controller` / `audit` / `policy`; console sample types include `sensor` / `actuator`. Only `device` is common to more than one list. Composition makes the *mechanism* consistent but does not, by itself, reconcile these **vocabularies** — that is a separate, larger question (see Non-goals and [§9](#9-open-questions)).

**M-5 — One field (`resource_type`) is asked to play two roles.** The gateway already uses `resource_type` as the action's `{domain}` segment. Under resource composition it would also become the resource identifier's `{type}` prefix. The kernel's examples treat these as the same token (`read:hvac:setpoint` acting on `hvac:zone-a`), but its action-domain set and `ResourceType` set differ, and a `read:hvac:sensor` action on a `sensor:co2:lobby` resource would need **two different tokens** (`hvac` for the action domain, `sensor` for the resource type). A single `resource_type` field cannot supply both when they diverge. This does not block the recommendation, but it bounds it (see [§5.3](#53-boundaries-of-the-recommendation) and [§9](#9-open-questions)).

---

## 4. Candidate models

### Model A — Typed `resource_id` only

Adapters and console emit a single, already-typed identifier (`ahu:rooftop-1`); `resource_type` is omitted entirely.

- **Pros:** Exactly the kernel's current contract — no kernel change, single source of truth for the type, self-describing identifier, no possibility of prefix/field drift (M-3 disappears).
- **Cons:** Pushes kernel-specific identifier composition **upstream into every adapter** (and onto the console's human operator). Adapters today render a *local* `resource_id` from operator templates and do not know the kernel's prefix convention; making each adapter responsible for emitting kernel-canonical identifiers spreads a kernel concern across nine protocol implementations — the same anti-pattern the action work rejected when it declined to make adapters compose the composite action. It also strands the adapter `resource_type` field (now redundant) and the gateway's symmetry with action composition is lost.

### Model B — Separate `resource_type` and `resource_id`, all the way to the kernel

The kernel is changed to accept a separate `resource_type` field and to do its own typing/composition (or to evaluate type from the field rather than the prefix).

- **Pros:** Matches the adapter/console field shape directly; no upstream composition needed.
- **Cons:** Changes the **kernel contract** to add a parallel type input and to normalize/infer from it — work the kernel deliberately does not do (it "receives normalized input", per the gateway trust-boundary model). It creates a permanent dual source of truth (field vs. prefix) and bakes M-3's drift risk into the kernel. It contradicts the already-settled action decision, which kept composition *out* of the kernel. Highest blast radius, lowest alignment with existing boundaries.

### Model C — Gateway-composed canonical `resource_id`

Adapters and console emit `resource_type` + local `resource_id`; the **gateway** composes `resource_type:resource_id → ahu:rooftop-1` before kernel evaluation, exactly as it already composes the action.

- **Pros:** Mirrors the ratified action-composition decision precisely. The gateway **already holds `resource_type`** at composition time, so the canonical action (`read:ahu`) and the canonical resource (`ahu:rooftop-1`) are produced **from the same field at the same point**, consistent by construction. No kernel change. Adapters stay free of kernel-specific identifier rules. Composition can emit `basis_gateway.*` evidence symmetric to the action evidence, preserving the original local id for audit.
- **Cons:** Concentrates another composition responsibility in the gateway (acceptable — it is the established composition boundary). Requires a rule for the case where `resource_id` is *already* typed (don't double-prefix) and for the `resource_type`-as-both-action-domain-and-resource-type tension (M-5). Does not by itself reconcile the type **vocabularies** (M-4).

### Model D — Hybrid: accept both styles, reject ambiguity at one boundary

The gateway accepts **either** a direct kernel-compatible request (already-typed `resource_id`, no `resource_type`) **or** an adapter-normalized request (`resource_type` + local `resource_id`), composing in the second case and **rejecting** ambiguous or internally inconsistent combinations (e.g. an already-typed `resource_id` *and* a `resource_type`, or a `resource_type` that contradicts an existing prefix).

- **Pros:** This is the exact generalization of what `compose_action()` **already does** for actions — it passes a composite action through untouched and rejects "composite action + resource_type" as ambiguous. Applying the same dual-accept/reject discipline to the resource fields is consistent and low-surprise. It accommodates the console (which sends a pre-typed `resource_id`) and adapters (which send the pair) without forcing either to change its emission model.
- **Cons:** Slightly more validation logic at the boundary; the ambiguity rules must be specified precisely so rejection is deterministic and its reasons are caller-safe.

---

## 5. Recommended canonical model

**Recommendation: Model C realized through Model D's dual-accept/reject boundary — both owned by `basis-gateway`.** This mirrors the action-composition decision and reuses the boundary that already exists.

### 5.1 The canonical resource identifier

The canonical authorization resource identifier — the value the policy engine evaluates and the audit record stores — remains the kernel's single **typed, structured** `resource_id` in `{type}:{qualifier[:{subqualifier}...]}` form, because policy scoping and audit forensics derive the resource type from that prefix and the kernel already enforces this shape. A separate `resource_type` field is **not** part of the canonical kernel request; it is a **normalization input**, not a canonical artifact.

### 5.2 Composition ownership and rule

`basis-gateway` is the resource-identifier composition boundary. Concretely:

- **Adapter-normalized request** (`resource_type` supplied, `resource_id` local/untyped): the gateway composes `"{resource_type}:{resource_id}"` and submits the result as the kernel `resource_id`.
- **Direct, kernel-compatible request** (`resource_id` already typed, no `resource_type`): the gateway passes `resource_id` through unchanged.
- **Ambiguous / inconsistent** (both an already-typed `resource_id` and a `resource_type`, or a `resource_type` that disagrees with an existing prefix, or a resource-independent request that nonetheless carries a stray `resource_type`): the gateway **rejects** the request at the composition boundary with a caller-safe error, exactly as `compose_action()` rejects "composite action + resource_type" today.
- When the gateway composes, it records reserved-namespace evidence (e.g. `basis_gateway.resource_composed`, `basis_gateway.original_resource_id`, `basis_gateway.resource_type`, `basis_gateway.composed_resource_id`) symmetric to the existing action-composition evidence, so the original local identifier is preserved for audit and the composition is never silently assumed.

This makes the canonical flow symmetric with action composition and consistent by construction, because the **same** `resource_type` value feeds both compositions:

```text
Adapter-normalized request:
  action        = read
  resource_type = ahu
  resource_id   = rooftop-1

Gateway composes (single boundary, single resource_type):
  action      = read:ahu
  resource_id = ahu:rooftop-1

Core evaluates:
  action      = read:ahu
  resource_id = ahu:rooftop-1     (type "ahu" derived from the prefix)
```

### 5.3 Boundaries of the recommendation

This recommendation fixes the **mechanism** (where composition happens and what the canonical identifier is). It deliberately does **not**:

- reconcile the resource-**type vocabularies** across components (M-4) — that is a separate contract, owned by the future `basis-schemas`, not this note;
- resolve whether the action's `{domain}` and the resource's `{type}` are one concept or two (M-5). The recommendation works cleanly when they coincide (the kernel's own examples), and the dual-accept boundary lets a caller send an already-typed `resource_id` when they must differ. Settling this is the main open question carried to the ADR.

Adapters must not own kernel-specific identifier composition. The kernel must not infer resource type from a separate supplied field — it continues to derive it from the prefix. The console must not bypass the gateway-owned composition when it submits gateway-backed evaluations; if it sends separate `resource_type` + local `resource_id`, it must send them to the gateway and let the gateway compose, rather than pre-composing or forwarding a descriptive `resource_type` the gateway would treat as authoritative.

---

## 6. Boundary ownership decision

```text
basis-gateway is the resource identifier composition boundary.
```

- **Adapters** emit `resource_type` and a **local** `resource_id` separately. They do not compose kernel-canonical identifiers.
- **Console** may emit `resource_type` and a **local** `resource_id` separately; when it does, it submits them to the gateway for composition rather than composing them itself. It may also send an already-typed `resource_id` directly (the direct path), but it must not do both for the same request.
- **Gateway** composes the canonical kernel `resource_id` (and rejects ambiguous combinations), at the same boundary and from the same `resource_type` field it already uses to compose the action.
- **Core** continues to receive a single typed `resource_id` and continues to derive the resource type from its prefix. The kernel contract does not change.

This is the same ownership shape ratified for actions: normalization inputs upstream, composition at the gateway, a single canonical artifact at the kernel.

---

## 7. Impact on each component

**basis-core** — No contract change. `DecisionRequest.resource_id` stays a single typed string; type continues to be derived from the prefix; `_RESOURCE_ID_RE` is unchanged. The kernel's audit `resource_type` field remains available but is not promoted to an evaluation input. This is the lowest-impact outcome and a reason to prefer Models C/D over B.

**basis-gateway** — Gains a resource-identifier composition step alongside the existing action composition, plus the dual-accept/reject rules and symmetric `basis_gateway.*` evidence. The `EvaluateRequest` docstring should be extended to describe resource composition (today it documents only action composition). No new dependency on `basis-adapters` or `basis-console` is introduced; the boundary stays self-contained, as `core/actions.py` already is.

**basis-adapters** — No change required to the emission model: adapters continue to emit `resource_type` + local `resource_id`. The reconciliation **endorses** their current separate-field output as the correct normalization input and removes any implicit expectation that an adapter should render a kernel-prefixed identifier. (If an adapter template happens to render an already-typed identifier, that becomes the direct path and the adapter should then **not** also emit a conflicting `resource_type` — a documentation note, not a code change.)

**basis-console** — Direction clarified: send separate `resource_type` + local `resource_id` to the gateway and let it compose, **or** send an already-typed `resource_id` with no `resource_type` — not both. The descriptive `resource_type` form field that currently drifts from the identifier prefix (M-3) should either be forwarded as a true composition input or dropped from the request payload, so the console stops carrying a second, non-authoritative type. No console change is mandated by this note; the design direction is recorded for when the console's gateway-backed path is next revised.

**future basis-schemas** — Owns the machine-readable home for: the canonical `resource_id` format, the resource-identifier **composition rule** (`resource_type` + local `resource_id` → canonical `resource_id`) alongside the action-composition rule it is already slated to own, and the dual-accept/ambiguity contract. It is also the right place to finally reconcile the resource-**type vocabulary** (M-4) and to decide the action-domain-vs-resource-type question (M-5). This document does **not** create `basis-schemas` and does not define schemas; it records the ownership so the boundaries are clear when the repository is created.

---

## 8. Examples

Canonical, adapter-normalized (the primary path):

```text
in:   action=read, resource_type=ahu, resource_id=rooftop-1
gw:   action=read:ahu, resource_id=ahu:rooftop-1
core: evaluates read:ahu on ahu:rooftop-1 (type "ahu" derived from prefix)
```

Three-segment local identifier (template rendered a multi-part local id):

```text
in:   action=read, resource_type=sensor, resource_id=co2:lobby
gw:   action=read:sensor, resource_id=sensor:co2:lobby
core: evaluates read:sensor on sensor:co2:lobby
```

Direct, kernel-compatible (console operator typed a full identifier):

```text
in:   action=read:hvac:setpoint, resource_id=hvac:zone-a   (no resource_type)
gw:   passed through unchanged
core: evaluates read:hvac:setpoint on hvac:zone-a
```

Resource-independent request:

```text
in:   action=read:audit:log   (no resource_type, no resource_id)
gw:   passed through unchanged
core: evaluates read:audit:log with resource_id = None
```

Rejected — ambiguous (already-typed identifier *and* a resource_type):

```text
in:   resource_id=hvac:zone-a, resource_type=hvac
gw:   rejected at composition boundary (caller-safe error): supply either a
      typed resource_id or resource_type + local resource_id, not both
```

Rejected — inconsistent (resource_type contradicts an existing prefix; the drift in M-3):

```text
in:   resource_id=ahu:rooftop-1:supply-temp, resource_type=sensor
gw:   rejected: resource_type 'sensor' disagrees with identifier prefix 'ahu'
```

---

## 9. Open questions

1. **Is the action `{domain}` the same token as the resource `{type}`?** (M-5) Today the gateway derives both from one `resource_type` field, and the kernel's examples coincide (`read:hvac:setpoint` on `hvac:zone-a`). But `read:hvac:sensor` on `sensor:co2:lobby` needs two different tokens. The ADR must decide whether one field can drive both compositions (with the direct path as the escape hatch when they differ) or whether the request model needs separate "action domain" and "resource type" inputs. This is the single most consequential unresolved item.

2. **Resource-type vocabulary.** (M-4) Adapter (`point`/`device`/`schedule`), kernel `ResourceType` (`hvac`/`sensor`/`zone`/`device`/`gateway`), and console (`sensor`/`actuator`) type sets barely overlap. Which set is canonical, and is it a closed enum (like the kernel `Resource` model) or an open prefix (like `DecisionRequest.resource_id`)? Owned by `basis-schemas`; out of scope here.

3. **Already-typed local identifiers from adapters.** If an operator configures a `resource_id_template` that already renders a typed identifier, should the gateway detect the existing prefix and skip composition, or should adapter output be constrained to *local* (untyped) identifiers so composition is unconditional? A deterministic detection rule must be specified.

4. **`resource_id` qualifier charset vs. composition.** The kernel's `_RESOURCE_ID_RE` permits `/` and additional colons in qualifiers. Composition must guarantee the joined `"{resource_type}:{resource_id}"` still matches the regex for all template-rendered local ids (e.g. ids containing `/` or `:`), and reject or sanitize those that would not.

5. **Audit of the original local identifier.** The composed identifier is what the kernel stores. The pre-composition local `resource_id` and its `resource_type` should be preserved in gateway audit evidence (as the action path already preserves the original action). Confirm this is sufficient for forensic traceability, or whether the kernel `AuditEvent.resource_type` field should carry the supplied type explicitly.

---

## 10. Recommended ADR follow-up

This note recommends opening a formal ADR to ratify the boundary decision:

```text
ADR: Gateway-Owned Resource Identifier Composition
```

The ADR should ratify the flow:

```text
resource_type + local resource_id
        ↓
gateway composition  (single boundary; dual-accept; reject ambiguity)
        ↓
canonical typed resource_id  (type derived from prefix at the kernel)
```

Following the conventions in [`docs/adr/README.md`](../adr/README.md), the ADR should:

- record the **decision** that `basis-gateway` owns resource-identifier composition, parallel to and at the same boundary as action composition;
- record the **alternatives considered** (Models A, B, C, D in [§4](#4-candidate-models)) and why Model C realized through Model D's dual-accept/reject boundary was chosen over pushing composition into adapters (A) or into the kernel (B);
- record the **consequences**: no kernel contract change, an additive gateway composition step with symmetric evidence, endorsement of the adapter separate-field output, and a clarified console direction;
- explicitly **defer** the action-domain-vs-resource-type question ([§9](#9-open-questions) item 1) and the resource-type vocabulary ([§9](#9-open-questions) item 2) to a `basis-schemas` ADR, so they are not silently resolved by implementation.

Provisional filename, consistent with the numbering and naming rules in [`docs/adr/README.md`](../adr/README.md): `docs/adr/0004-gateway-owned-resource-identifier-composition.md`. The number is assigned at acceptance, not at draft.

---

## Documents updated by this work

- **This report** — `docs/architecture/resource-identifier-reconciliation.md` (new).
- **[`glossary.md`](../glossary.md)** — added/refined definitions for *resource identifier*, *resource type*, *local resource identifier*, *canonical resource identifier*, and *resource identifier composition*; the existing **Resource** entry cross-references them.
- Minimal consistency cross-references added where existing documents discussed resource identifiers without naming the composition boundary (see the relevant entries in [`basis-gateway.md`](basis-gateway.md), [`basis-adapters.md`](basis-adapters.md), and [`action-vocabulary.md`](action-vocabulary.md)).

No code was changed, and no implementation repository was modified. A formal ADR should be opened to ratify [§5](#5-recommended-canonical-model).
