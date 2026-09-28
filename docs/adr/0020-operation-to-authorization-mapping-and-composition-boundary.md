# ADR-0020: Upstream Operation-to-Authorization Mapping and Composition Boundary

## Status

Proposed

Consistent with this repository's established practice (see ADR-0018's Context and ADR-0019's Status), this ADR is numbered when first proposed and submitted as `Proposed`. Merging it does not accept it. Acceptance requires a separate, dedicated review. Until then, nothing below is a current architectural position, and every statement elsewhere in this repository that describes action or resource composition ownership as undecided remains accurate.

## Context

[ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md) (Accepted) defines the producer-intake boundary through which an upstream supervisory system submits supervisory intent to the operation-producer role. It deliberately left one question to a later decision (ADR-0018 **Deferred Decisions**, "Resource and action mapping"): how upstream operation and target references become the protocol-shaped operation BASIS preserves for execution, and the authorization representation `basis-core` evaluates. This ADR answers that question on the BASIS side.

The question spans several things the repository currently holds in different states. This ADR records them as found on `main` before deciding anything.

**What adapter normalization produces.** `basis-adapters` normalizes a `ProtocolOperation` into a `NormalizedAuthorizationRequest` carrying a bare verb in `action`, a `resource_type`, and a *local* `resource_id`, plus verbatim `protocol_evidence` ([`ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) §3.6; [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §2). The `resource_type` and the `resource_id_template` from which the local identifier is rendered come from operator-supplied mapping configuration, which the adapter's caller supplies. Adapters do not compose kernel-canonical values.

**What the operation-producer role does with it.** The operation-producer role invokes adapter normalization, retains evidence (ADR-0007), creates the ADR-0012 binding record, and submits the normalized fields to `basis-gateway` "unchanged, inventing no canonical action/resource semantics" (`operation-producer-and-execution-boundary.md` §4). The `basis-producer` reference implementation does exactly this. Its REST reference route mapping is defined in code, and it records no identity for the mapping configuration it used.

**What `basis-gateway` does.** `basis-gateway` implements composition of both the canonical action (`read` + `ahu` → `read:ahu`) and the canonical resource identifier (`ahu` + `rooftop-1` → `ahu:rooftop-1`). The same `resource_type` value drives both. It also passes already-canonical values through unchanged and rejects ambiguous combinations. The operation-aware path reuses the same dual-accept logic. It classifies a composed resource identifier as gateway-derived and a passed-through one as untrusted-caller-asserted. [`ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) §3.3 and §3.5 record both compositions as implemented, "emerging," and "not yet ratified by ADR."

**What `basis-core` requires.** A composite action in `{verb}:{domain}[:{object}]` form, and either no resource identifier or a single typed identifier in `{type}:{qualifier}` form. The kernel derives the resource type from that prefix. It has no separate resource-type input and performs no composition ([`resource-identifier-reconciliation.md`](../architecture/resource-identifier-reconciliation.md) §2.1).

**What accepted architecture says about composition ownership.** The Accepted state is not settled, and it is not consistent either:

- [ADR-0017](0017-action-vocabulary-naming-structure.md) (Accepted) ratifies the five canonical verbs. It states that the composition rule and its owner are "explicitly out of scope ... and remain unresolved," and it assigns them to no component. [`action-vocabulary.md`](../architecture/action-vocabulary.md) repeats this: "Neither the rule nor its owner has been decided."
- [`resource-identifier-reconciliation.md`](../architecture/resource-identifier-reconciliation.md) *recommends* gateway-owned resource composition and calls for an ADR to ratify it. No such ADR exists.
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md), [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md) Decision 18, and [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md) Decision 8 (all Accepted) list "composition" among `basis-gateway`'s responsibilities. They describe the role but do not ratify a rule.
- [ADR-0012](0012-authorization-to-execution-binding.md) (Accepted) binds the preserved protocol-shaped operation and the *normalized* request the producer submitted. It explicitly does not bind the canonical form the gateway composes, and it treats gateway composition as "already governed."

The result is that implementation made a composition choice that accepted role descriptions assume and ADR-0017 declares undecided. The binding treats that choice as governed, but no governed decision states the rule, its sole owner, or its determinism.

**What no document addresses.** No document says who owns resolution from an upstream target to a protocol-native target. None governs, versions, or attributes adapter mapping configuration. None defines the provenance needed to trace an upstream request through authorization to the exact operation bound for dispatch.

**Motivating consumer.** Ipotio, a supervisory platform, tracks the cross-ecosystem form of this question in its own architecture repository as **ARCH-GAP-002**. It is referenced here by name, not by hyperlink, per this repository's cross-repository citation convention, and only as the motivating integration. Its current architecture proposes that the upstream system resolve its own durable asset identity to a protocol-native target and hand off the protocol-shaped operation, and that it never hand off a pre-composed canonical action or resource. That proposal is evidence of one consumer's needs. It is not assumed correct here. The models below are evaluated on BASIS's own terms, for any upstream supervisory system.

**Terminology.** The cross-ecosystem workstream refers to this ecosystem as *Basitra*. That name is proposed by [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Proposed) and is not yet canonical, so this ADR uses *BASIS*. *Upstream supervisory system*, *producer-intake boundary*, and *upstream workload identity* are the working vocabulary ADR-0018 introduced. *Mapping configuration*, *effective mapping identity*, and *mapping provenance* are introduced below as working vocabulary under the same convention and are not promoted to the glossary.

## Problem

How does an admitted upstream request become (a) the protocol-shaped operation BASIS preserves and may later dispatch, and (b) the canonical action and resource `basis-core` evaluates? The answer must name exactly one owner for each step and must not let:

- an upstream system choose its own BASIS authorization identity;
- two conforming producers derive different authorization identities for the same operation under the same configuration;
- a mapping change silently alter an operation that is already bound;
- the executor reconstruct a device command from the authorization representation;
- BASIS acquire a dependency on any one upstream platform's identifiers?

## Decision

**This decision is defined per concrete target operation: one protocol-shaped operation against one protocol-native target. The upstream supervisory system resolves its own targets and supplies each concrete operation in protocol-shaped form. BASIS never resolves an upstream reference into a protocol-native target. The operation-producer role instantiates that operation losslessly and preserves it. It derives authorization semantics from it solely through `basis-adapters` normalization, under an identified snapshot of BASIS-governed mapping configuration. On the governed admitted-producer path, `basis-gateway` is the single composition boundary for the canonical action and canonical resource identifier. It composes them from the normalized fields by a fixed rule that deployments cannot configure. On that path, canonical values are always derived there and never accepted from a caller. If the operation is authorized and later dispatched, the protocol executor dispatches only the preserved operation. How several concrete operations may be grouped under one parent intent is left to the separate one-to-many/batch contract (Decision 11).**

### 1. Four representations stay distinct

Four representations of "what is being done to what" participate. None is interchangeable with another, even where their values coincide:

| Representation | Example (illustrative) | Owned by |
| - | - | - |
| **Upstream source reference**: the upstream system's durable target identity and operation intent | an upstream asset identifier and "change setpoint" | Upstream system |
| **Protocol-shaped operation**: the protocol-native target and operation BASIS preserves and may dispatch | BACnet WriteProperty on device 1201, object analog-value 3, property present-value | Authored upstream (Decision 2); preserved by the operation-producer role (Decision 4) |
| **Normalized authorization semantics**: the adapter output | `action=write`, `resource_type=ahu`, `resource_id=rooftop-1` | `basis-adapters`, under BASIS mapping configuration (Decisions 5, 8) |
| **Canonical authorization representation**: the kernel input | `write:ahu` on `ahu:rooftop-1` | `basis-gateway` composition (Decision 6) |

Upstream durable identity is never a BASIS canonical resource identifier. The canonical resource identifier is never the protocol-native target. The local resource identifier and the canonical resource identifier are distinct concepts: the first is a normalization input and the second is the only form the kernel evaluates.

### 2. What enters BASIS production

For each concrete target operation, an admitted intake request supplies:

- **a protocol-shaped operation.** This is an execution-ready description of one operation on one protocol-native target, expressed in the terms of a protocol `basis-adapters` supports. It is the only operation/target content in the intake request from which BASIS derives the normalized and canonical authorization action and resource, and the concrete operation that may later be dispatched. It is not the only input to authorization. Whether the operation is authorized still depends on the authenticated authorization subject, policy, and whatever context is eligible under the governing trust rules;
- optionally, **upstream source references** (the upstream system's durable target identity, operation intent, and request identifier). These are carried as upstream-owned provenance (ADR-0018 Decision 5). BASIS does not interpret them, does not validate their truth, and does not derive authorization semantics or a target from them.

Whether one intake request carries one concrete operation or several is not decided here (Decision 11). An intake request does **not** supply normalized authorization semantics, a canonical action, or a canonical resource identifier, and no intake field has that meaning. A request that asserts any of them is malformed or contradictory and fails closed under ADR-0018 Decision 6. This ADR does not choose the representation or schema of the protocol-shaped operation at intake. That remains with the intake transport and envelope decision ADR-0018 defers.

### 3. Protocol-native target resolution is upstream-owned; BASIS performs none

Resolving an upstream durable identity to a protocol-native target (an asset to a BACnet object, a Modbus unit and register, an OPC UA node, an MQTT topic, or a REST resource) belongs to the upstream system. BASIS holds no registry of upstream identifiers, performs no lookup from an upstream reference to a protocol target, and never substitutes, completes, or corrects the protocol-native target it received.

This is a split by semantic layer, and each layer has one owner:

- **Upstream system:** upstream durable identity → protocol-native target. BASIS does not govern this step and cannot verify it. An upstream resolution error is an upstream error. BASIS guarantees that if the operation is authorized and later dispatched, the dispatched operation and protocol-native target are the preserved operation and target received through intake. It records the upstream reference beside them (Decision 10). BASIS does not guarantee an authorization outcome.
- **BASIS:** protocol-native target → normalized authorization semantics (Decision 5) → canonical authorization representation (Decision 6). Both steps are deterministic and BASIS-governed.

The protocol executor may resolve *transport reachability* for the preserved protocol-native target, such as a network endpoint for a device instance, from its own deployment configuration. That resolution operates on the preserved target, never on the canonical representation. It must not change which protocol-native target is addressed, and a failure to resolve fails closed.

### 4. Construction and preservation of the governed operation

The operation-producer role turns the admitted protocol-shaped operation into the `ProtocolOperation` it preserves and binds. The upstream system authors the content. The operation-producer role instantiates it, and it does so **losslessly**: no target, operation, parameter, or value is added, removed, defaulted, or changed. If the operation cannot be represented losslessly as a `ProtocolOperation` for a configured adapter, production fails closed.

From instantiation onward, that `ProtocolOperation` is the preserved original operation required by ADR-0011 Decision 7 and bound by ADR-0012. Normalization, composition, authorization, and mapping-configuration changes never modify it.

### 5. Adapter normalization is unchanged and is the only path to authorization semantics

The canonical model remains:

```text
operation-producer role  --invokes-->  basis-adapters  -->  normalized authorization semantics
```

Normalized authorization semantics are derived only by `basis-adapters` normalization of the preserved `ProtocolOperation`, under the effective mapping configuration snapshot (Decision 9). No other source may supply or alter them, including the upstream system, the intake boundary, and the producer's own logic. `basis-adapters` remains a deterministic normalization library. It executes mapping configuration but neither authors nor governs it, and it holds no authorization authority. Adapter normalization is deterministic: the same `ProtocolOperation`, adapter release, and mapping configuration always produce the same normalized result.

On the governed operation path, the resource identity (`resource_type` and local `resource_id`) must come from an explicit entry in the effective deployment mapping configuration. A built-in verb default shipped in a versioned adapter release (ADR-0017) is part of the effective mapping. A resource identity derived from any other source, such as a raw protocol address used because no entry matched, is a silent fallback and fails closed.

### 6. Canonical composition on the governed producer path has exactly one owner: `basis-gateway`

For the governed admitted-producer path (the ADR-0018 intake → operation-producer role → `basis-gateway` → `basis-core` → binding and protocol-executor path), this ADR resolves the composition question ADR-0017 left open (reconciliation finding I-1). It ratifies the recommendation in `resource-identifier-reconciliation.md` §5 for that path:

1. **Sole owner on the governed path.** For admitted operation-producer submissions governed by this ADR, `basis-gateway` is the only component that composes the canonical action and the canonical resource identifier for evaluation. Within that flow, upstream systems, the intake boundary, the operation-producer role, adapters, the protocol executor, and `basis-core` do not compose. The pass-through form is not a second composition path for admitted producers (point 4). `basis-core` continues to validate the canonical formats and to derive resource type from the identifier prefix. Its contract does not change.
2. **The rule.** For normalized input `action = v`, `resource_type = t`, and local `resource_id = l`:
   - canonical action = `v:t`;
   - canonical resource identifier = `t:l`.

   One `resource_type` value supplies both the action's `{domain}` segment and the resource identifier's `{type}` prefix. The verb is composed as supplied. Composition does not translate deprecated aliases, consistent with ADR-0017's refusal of any evaluation-time rewriting mechanism. This is the rule `basis-gateway` implements today, stated here as the governed rule.
3. **Fixed and deterministic.** The rule is a pure function of the normalized fields. It is not deployment-configurable and has no per-producer, per-upstream, or per-adapter variants. Changing it, for example to separate action domain from resource type or to add an `{object}` segment, is a change to this decision that requires a superseding or amending ADR. It remains gateway-owned. This is why two conforming producers cannot derive different canonical identities from the same operation under the same mapping configuration: everything upstream of composition is fixed by the preserved operation and the mapping snapshot, and composition itself has no inputs beyond the normalized fields.
4. **Derivation required on the governed operation path.** When `basis-gateway` has classified the caller as an admitted operation producer (ADR-0008, ADR-0009), it must require the adapter-normalized form (a single-segment verb, a `resource_type`, and a local `resource_id`) and compose. It must reject a submission that carries an already-composite action, an already-typed resource identifier, or no local resource identifier. On this path, the pass-through form is not a second composition path.
5. **Ambiguity is rejected, never repaired.** Composition rejects a local identifier that already contains the canonical separator (`:`) when a `resource_type` is present, and any segment that is not a valid single segment. It also rejects any composed value that would fail the kernel's format. The gateway does not sanitize, escape, truncate, or guess.
6. **Direct path, outside execution.** For callers the gateway has *not* classified as admitted operation producers, such as `basis-console`'s direct evaluation, the existing dual-accept behavior may continue. An already-canonical value is passed through and classified untrusted-caller-asserted, and ambiguous combinations are rejected. Within `basis-gateway`, only on this path may a resource type exist without a resource instance (domain-level evaluation with no resource identifier). A disposition obtained on the direct path is not bound to any preserved operation and cannot support dispatch (ADR-0011 Decision 9, ADR-0012). Every operation that may be dispatched carries a resource instance.
7. **Composition is evidenced.** On the governed operation path, the gateway records its existing reserved-namespace composition evidence for every evaluation: the original verb, `resource_type`, and local identifier, and the composed action and identifier. That evidence is normative here, not optional.

**Why composition sits at the gateway, and why the binding still reaches it.** ADR-0012 binds the preserved operation and the normalized request, not the canonical form. Point 3 makes the canonical form a fixed function of the bound normalized request, and point 7 records that function's input and output for every governed evaluation. The canonical identity evaluated for a bound operation is therefore fully determined by bound content and reconstructible from evidence, with no new binding and no change to ADR-0012.

**Scope: embedded deployments are not governed or superseded by this decision.** Existing architecture also recognizes an embedded deployment model in which an adapter or enforcement boundary invokes `basis-core` directly, without `basis-gateway` ([`basis-adapters.md`](../architecture/basis-adapters.md), *Relationship to `basis-gateway`*). This decision does not remove or supersede that model. The model lies outside ADR-0018's admitted-producer path and outside the composition-ownership decision made here. This ADR does not define, revise, or ratify composition ownership for embedded direct-kernel deployments. Nothing here makes such a deployment execution-eligible under the ADR-0011 and ADR-0012 producer/executor flow.

### 7. Caller-supplied authorization identity

| Where | Caller-supplied normalized or canonical identity is... |
| - | - |
| Producer-intake boundary | **Rejected.** No intake field carries it (Decision 2). |
| Operation-producer role | **Never forwarded.** The `action`, `resource_type`, and `resource_id` it submits come only from adapter output for the preserved operation. It does not pre-compose, substitute, or edit them. |
| `basis-gateway`, admitted-producer submissions | **Rejected** in composite or typed form. Composition is required (Decision 6.4). |
| `basis-gateway`, direct callers | **Accepted as caller-asserted** and classified untrusted-caller-asserted. The kernel evaluates it against the authenticated subject's policy. It is never execution-eligible (Decision 6.6). |

An upstream system therefore cannot select a privileged-looking BASIS canonical resource or action by sending a string. It can influence authorization identity only by the protocol-native target it names, whose normalization BASIS governs, and by proposing mapping configuration that only a BASIS configuration authority can adopt (Decision 8).

### 8. Mapping configuration governance

*Mapping configuration* means the deployment-specific normalization tables adapter normalization executes (routes or object, register, node, and topic matches; `resource_type`; `resource_id_template`; explicit verb mappings) together with the built-in defaults of the adapter release in use. Mapping *execution* and mapping *governance* are separate:

- **Execution:** `basis-adapters`, invoked by the operation-producer role, with no discretion.
- **Governance:** a **BASIS configuration authority** designated by the deployment. It is an administrative role, attributable to an authenticated administrator or change process, and the only party that can create, change, or retire effective mapping configuration.

The following requirements apply:

- **Mapping configuration is BASIS deployment configuration.** It is deployment-specific. It remains BASIS configuration even when its content is derived from an upstream system's topology.
- **Upstream influence is declarative only.** An upstream system may supply proposed mapping content, for example derived from its own asset or topology model, to the BASIS configuration authority. A proposal has no effect until that authority adopts it. No intake request, runtime interaction, or upstream credential can create, activate, or modify effective mapping configuration.
- **Who cannot change it.** The upstream system, the producer runtime (at run time), `basis-adapters`, `basis-gateway`, and the protocol executor cannot change effective mapping configuration.
- **Validation at adoption.** Configuration must be validated before it becomes effective. At minimum, any protocol-native target the deployment governs matches at most one entry. Every entry produces segment-valid, separator-free normalized values. Two distinct protocol-native targets never produce the same `(resource_type, local resource_id)` unless the configuration declares that equivalence explicitly. Configuration that fails validation does not become effective.
- **Missing, ambiguous, or conflicting mappings fail closed at run time as well** (see **Failure Behavior**). Silent fallback is never permitted.
- **Change is compatibility-sensitive.** Changing the mapping for an established operation changes the authorization identity it evaluates under. [`compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) already treats that as a breaking change for adapter normalization, and ADR-0017 relies on the same classification.

This ADR does not define an administrative API, a storage technology, a configuration format, or an approval workflow.

### 9. Effective mapping identity and versioning

1. **Identity.** Every effective mapping state has a stable **effective mapping identity** covering both the adapter release identity and the deployment mapping-configuration state. That identity determines the complete normalization mapping. Its form, whether version, digest, or otherwise, is not chosen here.
2. **One snapshot per operation.** The operation-producer role resolves the effective mapping identity once per governed operation, normalizes under exactly that snapshot, and records the identity. Normalization never mixes states. If the identity cannot be determined, production fails closed.
3. **Normalize once.** A governed operation is normalized exactly once. Any re-normalization is a new production, with a new normalized request, a new binding record, and a new authorization.
4. **After binding, nothing moves.** A mapping change after the binding record is created never mutates, retargets, or re-resolves the bound operation. The executor dispatches the preserved operation and never consults mapping configuration. A future lifecycle or freshness rule may decide that a mapping change makes a pending, not-yet-dispatched operation ineligible. That would be a *not executed* outcome, in the same open family as ADR-0012's policy-reload invalidation question, and it is not decided here. Re-resolution under the new mapping is never permitted.
5. **Later requests.** A request produced after a change is normalized under the then-effective mapping. The same protocol-native target may therefore evaluate under a different canonical identity after a governed change. That is visible, attributable, and distinguishable in evidence by effective mapping identity. It is never silent.
6. **Reconstructibility.** Every mapping state referenced by recorded provenance must remain reconstructible for at least as long as any evidence that references it is retained.

### 10. Mapping provenance and traceability

For each governed operation, the operation-producer role records **mapping provenance** sufficient, together with records that already exist, to answer the following question. *Which upstream request and reference became which canonical action and resource, and which exact protocol operation was bound and dispatched?* At minimum, the record links:

- the upstream workload identity and the upstream request identifier (ADR-0018 Decision 5);
- the upstream source references as received, labeled upstream-originated and unverified;
- the preserved protocol-shaped operation, through the ADR-0012 binding record;
- the effective mapping identity used (Decision 9);
- the normalized request submitted, which is bound under ADR-0012;
- the gateway `correlation_id`, kernel `trace_id`, and kernel `evidence_id` the round trip returns.

The gateway's composition evidence (Decision 6.7) and the kernel's audit evidence then record the canonical action and resource. Execution evidence (ADR-0014) records what was attempted against the preserved operation. The complete chain is:

```text
upstream refs  --(mapping provenance)-->  preserved operation + effective mapping identity
               --(ADR-0012 binding)-->    normalized request
               --(gateway composition evidence, correlation_id)-->  canonical action/resource
               --(evidence_id / trace_id)-->  kernel AuditEvidence
preserved operation  --(binding verification)-->  dispatch  -->  execution evidence (ADR-0014)
```

This ADR defines no record schema. It does not define the complete cross-ecosystem evidence-correlation contract (Workstream 4), and it adds no wire field to the gateway request or response. Upstream source references carried as provenance are not authorization context. Whether any upstream-originated value may reach the evaluation context is the Workstream 3D question (see **Deferred Decisions**), and ADR-0018 Decision 4's interim posture stands.

### 11. One concrete target, one governed mapping unit; no reinterpretation

This ADR specifies the mapping chain for one concrete protocol-shaped operation against one concrete protocol-native target. That is the **governed mapping unit**. For each unit there is one preserved `ProtocolOperation`, one effective mapping identity, one normalized request, and one canonical action and resource. Each unit is subject to the ADR-0012 binding and has its own mapping provenance.

**The operation-producer role never fans out, broadens, narrows, splits, merges, substitutes, or reinterprets a concrete operation after it crosses intake.** For each concrete operation it received, it either produces exactly that governed mapping unit or fails closed. It never derives additional targets or operations from one it received.

Every concrete target that takes part in a future one-to-many operation must satisfy the mapping, binding, provenance, and traceability rules of this ADR on its own. Every concrete execution target must be independently traceable to the authorization context that actually permitted that target's execution.

This ADR does **not** decide whether a future intake or batch contract:

- carries one concrete operation per intake request, or groups several;
- uses one authorization context valid across several targets, or target-specific authorization contexts;
- represents parent and child requests, several authorization decisions associated with one parent intent, or mixed per-target outcomes.

Those questions remain with the existing one-to-many/batch operation decision gate. One valid realization is for an upstream system to fan a parent intent out into several single-target intake requests. This ADR does not make that the only permissible architecture.

## Architectural Mapping Chain

The chain below is drawn for one governed mapping unit (Decision 11).

```text
UPSTREAM (outside BASIS)
  supervisory intent, durable target identity, operation intent
        │  upstream-owned target resolution (Decision 3)
        ▼
  protocol-shaped operation for one protocol-native target  (+ upstream source refs as provenance)
        │  intake request  (grouping of several concrete operations: not decided here)
════════╪═══════════════════════ producer-intake boundary (ADR-0018) ═══════════════════════
        ▼
  operation-producer role
    ├─ lossless instantiation → preserved ProtocolOperation ──────────────────────┐ (Decision 4)
    ├─ resolve effective mapping identity (one snapshot)                          │ (Decision 9)
    ├─ invoke basis-adapters normalization under that snapshot                    │ (Decision 5)
    │     → normalized: verb, resource_type, local resource_id                    │
    ├─ create ADR-0012 binding record over {preserved operation, normalized request}
    └─ record mapping provenance; authenticated submission (ADR-0008/0009)        │ (Decision 10)
        ▼                                                                         │
  basis-gateway  (sole composition boundary on this path; derivation required)     │ (Decision 6)
        │  canonical action  v:t      canonical resource  t:l   (+ composition evidence)
        ▼                                                                         │
  basis-core evaluation  →  authoritative disposition                             │
        ▼                                                                         │
  protocol-executor role: verify binding; dispatch the PRESERVED operation  ◄─────┘
        ▼
  OT target

Never:  canonical action/resource ──reverse-map──► protocol command
Never:  upstream-supplied canonical/normalized identity ──► kernel (governed path)
Never:  mapping change ──► bound operation
```

## Ownership

| Semantic concern | Owner | Notes |
| - | - | - |
| Supervisory intent, durable target identity, operation intent | Upstream system | Outside BASIS authority; carried only as provenance |
| Upstream identity → protocol-native target resolution | Upstream system | BASIS performs none (Decision 3) |
| Protocol-shaped operation content | Upstream system (authors) | Per concrete target operation (Decisions 2, 11) |
| Grouping of concrete operations (one-to-many/batch) | Not decided here | Existing one-to-many/batch decision gate |
| Intake admission | Producer-intake boundary | ADR-0018, unchanged |
| Instantiation and preservation of the `ProtocolOperation` | Operation-producer role | Lossless; immutable thereafter (Decision 4) |
| Adapter normalization (execution) | `basis-adapters`, invoked by the operation-producer role | Deterministic; no discretion (Decision 5) |
| Mapping configuration authorship and governance | Deployment-designated BASIS configuration authority | Upstream may only propose (Decision 8) |
| Effective mapping identity: capture and recording | Operation-producer role | One snapshot per operation (Decision 9) |
| Mapping provenance | Operation-producer role | Decision 10 |
| Composition rule definition | BASIS architecture (this ADR) | Machine-readable publication is a future `basis-schemas` candidate; publication does not move ownership |
| Canonical action composition on the governed admitted-producer path | `basis-gateway`, only | Decision 6; embedded direct-kernel deployments are out of scope |
| Canonical resource composition on the governed admitted-producer path | `basis-gateway`, only | Decision 6; embedded direct-kernel deployments are out of scope |
| Composition evidence | `basis-gateway` | Decision 6.7 |
| Canonical format validation; resource type derivation | `basis-core` | Unchanged |
| Authorization | `basis-core`, through `basis-gateway` | Unchanged |
| Binding creation / verification | Operation-producer role / protocol-executor role | ADR-0012, unchanged |
| Transport reachability for the preserved target | Protocol-executor role | Must not change the target (Decision 3) |
| Dispatch | Protocol-executor role | Preserved operation only; no reverse mapping (ADR-0011 Decision 7) |

## Security Invariants

These invariants apply to the governed admitted-producer path: one governed producer operation, one deterministic normalization path, one gateway composition boundary, and one kernel authorization identity. They do not claim that every BASIS deployment contains `basis-gateway`.

- **Target substitution.** BASIS performs no target resolution, and instantiation is lossless. BASIS therefore cannot resolve an intended target A into a dispatched target B. An upstream-side resolution error is outside BASIS's control, but it is recorded: the upstream reference and the protocol-native target actually authorized and dispatched are linked in mapping provenance.
- **Namespace collision.** Two distinct protocol-native targets cannot silently share a canonical resource. Adoption-time validation rejects unintended collisions, so equivalence exists only where declared. Composition rejects separator-bearing local identifiers, so different `(t, l)` pairs cannot compose to the same identifier.
- **Mapping drift.** An effective mapping identity is captured once per operation. Bound operations are never re-resolved. Every change is attributable and distinguishable in evidence.
- **Authorization/execution divergence.** The canonical identity is a fixed function (Decision 6.3) of the normalized request, and that request is bound with the preserved operation under ADR-0012. The executor dispatches only that preserved operation.
- **Caller-controlled authorization identity.** There is no alternate caller-controlled composition route within the governed producer path. Such values are rejected at intake, never forwarded by the producer, and rejected by the gateway for admitted producers. On the direct path they are caller-asserted and never execution-eligible (Decision 7).
- **Reverse mapping.** The executor has no input from which to reverse-map. It dispatches the preserved operation and consults neither the canonical representation nor mapping configuration.
- **Confused deputy.** After intake, the producer never fans out, broadens, substitutes, or reinterprets a concrete operation (Decision 11). Its only derivation path is deterministic normalization under governed configuration, and the kernel decides under the authenticated subject's authority, never the upstream workload's (ADR-0018 Decision 3).

## Failure Behavior

Each outcome below is semantic. This ADR defines no error codes or wire representations. *Production fails closed* means that no governed operation is produced, no gateway submission or binding record is made, and nothing is dispatched. The outcome is recorded attributably and belongs to the "not executed" family (`operation-producer-and-execution-boundary.md` §8), upstream of ADR-0013's Case A.

| Condition | Detected at | Outcome |
| - | - | - |
| Malformed or unsupported protocol-shaped operation, or malformed upstream target reference | Intake boundary; or producer instantiation | Intake rejected (ADR-0018 Decision 6), or production fails closed |
| Operation cannot be instantiated losslessly | Operation-producer role | Production fails closed |
| Unsupported operation (no configured adapter, or the adapter cannot normalize the operation kind) | Normalization | Production fails closed |
| Missing mapping (no effective entry covers the target) | Normalization | Production fails closed; no fallback |
| Ambiguous mapping (more than one entry matches) | Adoption validation; or normalization | Not adopted; or production fails closed |
| Conflicting mapping (undeclared canonical collision, or contradictory entries) | Adoption validation; or normalization | Not adopted; or production fails closed |
| Operation inconsistent with its mapping entry (for example, the operation kind or parameters are outside what the entry covers) | Normalization | Production fails closed |
| Caller-supplied normalized or canonical identity where derivation is required | Intake; producer; gateway (admitted producer) | Intake rejected; never forwarded; composition rejected |
| Effective mapping identity unavailable or indeterminate | Operation-producer role | Production fails closed |
| Mapping changes before normalization | Operation-producer role | The new snapshot applies in full; states are never mixed |
| Mapping changes after binding | — | Bound operation unchanged. It may be *not executed* under a future rule, and it is never re-resolved |
| Normalization output cannot form a valid canonical representation (invalid segment, separator in local identifier) | Gateway composition | Composition rejected; no permitting disposition ("request composition failed," ADR-0011 Decision 9) |
| Composed value rejected by the kernel | `basis-core` | Evaluation failed; not executed |

## Relationship to ADR-0018

This ADR answers the "Resource and action mapping" item ADR-0018 deferred. It does not reopen ADR-0018 and does not modify its body. The following stay separate, each with its own owner and failure mode:

```text
intake admission  !=  mapping / normalization  !=  producer submission / gateway admission
                  !=  composition / authorization  !=  execution
```

It narrows what an intake request may carry (Decision 2). That is a constraint on the intake transport and envelope decision ADR-0018 defers, not a selection of that transport or envelope. It preserves ADR-0018 Decision 4's interim context posture and ADR-0018 Decision 5's correlation-ownership rule without change.

## Relationship to Other Decisions

- **ADR-0017.** For the governed admitted-producer path, this ADR decides the composition rule and owner that ADR-0017 declared out of scope. It does not change the verb set, the aliases, or the migration posture.
- **ADR-0011 / ADR-0012.** This ADR preserves ADR-0011 Decision 7 (no reverse mapping) and ADR-0012's binding content unchanged. It supplies the governed composition that ADR-0012 treated as already governed, and it explains why the binding determines the canonical identity (Decision 6).
- **ADR-0010, ADR-0011 Decision 18, ADR-0018 Decision 8.** Their description of composition as a `basis-gateway` responsibility is now backed by a governed rule on the admitted-producer path.
- **Embedded deployment model.** Not superseded; see the scope paragraph in Decision 6.
- **[`resource-identifier-reconciliation.md`](../architecture/resource-identifier-reconciliation.md).** For the governed admitted-producer path, this ADR ratifies §5 (Model C through Model D), with one refinement: on the admitted-producer path, derivation is required and pass-through is not accepted. It answers open questions 3 and 5. Local identifiers must not contain the separator, and composition evidence plus mapping provenance preserve the original values. Open questions 1 and 2 remain deferred. The report's §8 three-segment example (`resource_type=sensor`, `resource_id=co2:lobby`) is rejected under Decision 6.5, consistent with current gateway behavior.

## Relationship to ARCH-GAP-002

ARCH-GAP-002 is a joint cross-ecosystem gap tracked in Ipotio's architecture repository. On acceptance, this ADR establishes the **BASIS-side prerequisite** for closing it. It does **not** close ARCH-GAP-002, which remains open until Ipotio reconciles its asset and operation model against this decision in its own repository.

That reconciliation will need to confirm, at least:

- **Asset influence without assertion.** Ipotio's durable asset identity influences BASIS only through (a) Ipotio's own resolution to a protocol-native target and (b) mapping content Ipotio may *propose* to a BASIS configuration authority. It is never asserted as a BASIS canonical resource (Decisions 2, 3, 7, 8).
- **Target resolution and the intake representation.** Ipotio's decision to resolve targets and hand off the protocol-shaped operation for each resolved target conforms to Decisions 2 and 3. Ipotio's fan-out of one intent into per-target handoffs is one valid realization under Decision 11. It is not required by this ADR, and the one-to-many/batch contract stays with its own gate.
- **Operation semantics without self-authorization.** Ipotio's operation-class semantics reach BASIS only as provenance. The normalized and canonical action and resource derive from normalization of the protocol-shaped operation alone. Subject, policy, and eligible context still determine the outcome. Ipotio's statement that asset identity reaches the BASIS decision as producer-asserted context (for example `device` or `location`) must be reconciled with ADR-0018 Decision 4's interim posture and left to Workstream 3D. Under this ADR, asset identity is mapping provenance only.
- **Cross-ecosystem change governance.** An Ipotio topology change affects later Ipotio target resolution. A BASIS mapping change is governed by a BASIS configuration authority (Decision 8). Keeping the two consistent, including through Ipotio-originated proposals, is an operating concern that both ecosystems must describe.
- **Traceability.** The Ipotio request identifier and asset reference link, through BASIS mapping provenance, to the effective mapping identity, the binding, the canonical action and resource, and execution evidence (Decision 10). The full correlation contract remains Workstream 4.

Nothing here requires a BASIS type, field, or vocabulary specific to Ipotio. Every upstream supervisory system uses the same decision.

## Alternatives Considered

### What enters BASIS production

**Upstream supplies references; BASIS resolves them into a protocol-shaped operation.** In this model the operation-producer role would hold a BASIS-side registry from upstream target references to protocol-native targets. Rejected. BASIS would need its own asset and topology model, duplicating the upstream system's model and introducing a second place where the two can drift apart. BASIS would become the component that decides which device an upstream intent means, which is exactly the target-substitution surface this ADR removes. It would also need per-upstream identifier namespaces, eroding platform neutrality. The one real advantage, usability by an upstream system with no protocol knowledge, does not require BASIS to own resolution. Such an upstream can resolve targets through its own integration, and a future decision may revisit a BASIS-side resolver if it satisfies the invariants above.

**Upstream supplies both source semantics and an execution-ready operation, and BASIS checks them for consistency.** Rejected in its consistency-checking form. BASIS can check the two for consistency only if it models the upstream's identity-to-target relationship, which is the registry rejected above. The provenance half of this model is adopted: source references travel with the operation, unverified and uninterpreted (Decision 2).

**Upstream supplies the protocol-shaped operation, and BASIS derives the authorization action and resource from it.** Selected. The upstream owns what it already knows, and BASIS owns what it must govern. The design reuses the chain that already exists (ADR-0010, ADR-0011 Decision 7, ADR-0012) without a new resolution step, and it works identically for any upstream system.

### Where canonical composition happens

**Model A — Upstream canonicalizes.** Rejected. It would transfer control of BASIS's authorization namespace to an external application. The identity the kernel evaluates would be whatever string an admitted upstream sent, detached from the operation dispatched, and every upstream would need to track BASIS's composition rules.

**Model B — Adapters or the producer canonicalize.** Rejected. It couples nine protocol adapters, and every conforming producer, to kernel formats, so the rule would be duplicated across independently released components. It contradicts the adapter/kernel separation the resource reconciliation (Model A there) and the action reconciliation both rejected. It also makes "two producers, same operation, different identity" possible whenever implementations or versions disagree.

**Model C — Normalization, then centralized composition at `basis-gateway`.** Selected. It preserves the existing layering. It ratifies what is implemented, what accepted role descriptions already assume, and what the resource reconciliation recommended. It puts composition at the one component every governed operation already crosses, beside the evidence that records it.

**Model D — Dual-accept everywhere, including the admitted-producer path.** Rejected for the governed operation path. Accepting pre-composed values from admitted producers would create a second path to an authorization identity that bypasses adapter normalization. Dual-accept is retained only on the direct path, which is not execution-eligible (Decision 6.6).

**Kernel composes from a separate `resource_type` input.** Rejected. For the reasons in `resource-identifier-reconciliation.md` §4 Model B, it changes the kernel contract and creates two sources of truth for resource type.

**`basis-schemas` owns composition.** Rejected as an *owner of execution*: `basis-schemas` publishes contracts and runs nothing. It remains the candidate home for the machine-readable form of the rule this ADR defines.

## Consequences

### Positive

- On the governed admitted-producer path, every step from upstream intent to the kernel input, and from the kernel input back to dispatch, now has exactly one owner.
- The composition gap ADR-0017 left open (I-1) and the resource reconciliation's recommendation are resolved together at one boundary for the governed admitted-producer path, consistent with implementation and with accepted role descriptions.
- The canonical identity evaluated for a bound operation is determined by bound content, without changing ADR-0012.
- Mapping configuration becomes governed, identified, attributable BASIS configuration, instead of unversioned tables supplied by whichever process invokes an adapter.
- Any upstream supervisory system can integrate with no BASIS change and no BASIS knowledge of its identifiers.

### Negative / Tradeoff

- This ADR governs individual concrete operations only. A one-to-many operation cannot be realized until the separate batch contract is decided, beyond single-target requests the upstream fans out itself.
- An upstream system must be able to produce a protocol-shaped operation, which requires protocol-level knowledge of its targets. An upstream without that knowledge needs its own resolution capability.
- BASIS cannot detect an upstream target-resolution error. It can only record it.
- Deployments gain an administrative obligation: a designated configuration authority, adoption-time validation, and retention of past mapping states.
- One `resource_type` still drives both the action domain and the resource type (M-5). Deployments whose action domains and resource types diverge cannot express that divergence on the governed path until a later decision changes the rule.
- Colon-bearing local identifiers are no longer expressible. Multi-part local identifiers must use another permitted character.
- When implemented, admitted producers lose the gateway's pass-through path. That is a deliberate narrowing of current gateway behavior.

### Security Consequences

**Prevented:** the attacks and failure modes listed under **Security Invariants**.

**Still requiring follow-on architecture:** trust in upstream-originated context (3D); upstream and producer credential lifecycle (3E); the intake transport and envelope; the invalidation of pending operations on mapping change; the complete evidence-correlation contract (Workstream 4).

**Residual risks:** a compromised or mistaken upstream system can name a wrong but mapped protocol-native target. The operation is still evaluated under the authenticated subject's authority against the correct canonical identity *for the target named*, and the discrepancy is recorded. A compromised BASIS configuration authority can adopt harmful mappings, just as a compromised policy author can write harmful policy. A compromised producer process can defeat these guarantees along with the binding (ADR-0012 residual risks).

## Deferred Decisions

- Whether the action `{domain}` and resource `{type}` become separate inputs (resource reconciliation M-5 and open question 1), and three-part `{verb}:{domain}:{object}` composition.
- Resource-type and domain vocabulary alignment (M-4; `basis-schemas`).
- A BASIS-side resolver from upstream references to protocol-native targets, if a future need justifies one.
- Mapping-configuration administrative surface, storage, format, approval workflow, and the form of the effective mapping identity.
- Whether, and when, a mapping change makes a pending bound operation ineligible (with ADR-0012's broader freshness questions).
- Value transformation (units, scaling) between upstream intent and protocol values. Under this ADR, values are carried as supplied.
- Context-assertion trust for upstream-originated values (ARCH-GAP-004, Workstream 3D).
- Upstream and producer workload credential lifecycle (ARCH-GAP-011, Workstream 3E).
- Intake transport, envelope, and schema (ADR-0018).
- The one-to-many/batch operation contract. This includes whether an intake request carries one or several concrete operations, whether one authorization context may cover several targets or each target gets its own, and how parent/child requests and mixed per-target outcomes are represented. It remains with the existing one-to-many/batch decision gate, subject to Decision 11's per-target rules.
- Complete cross-ecosystem evidence correlation (Workstream 4).
- Machine-readable publication of the composition rule (`basis-schemas`).
- Composition ownership for embedded direct-kernel deployments, which remain governed by their existing architecture.

## Non-Goals

This ADR does not: implement anything; modify any implementation repository or schema; add an endpoint or wire field; define an intake transport or format; define the one-to-many/batch operation contract; remove, supersede, or define composition for the embedded adapter-to-`basis-core` deployment model; redesign the policy language, the kernel, or the canonical verb set; define a resource taxonomy; change the ADR-0012 binding; define executor behavior beyond dispatching the preserved operation; reopen ADR-0018 or the role placement behind ARCH-GAP-001; modify Ipotio's repository; or close ARCH-GAP-002.

## Validation / Implementation Gate

Acceptance requires the same separate, dedicated review every ADR since ADR-0007 has received. Acceptance authorizes no implementation. Once it is accepted, the following gaps between current implementation and this decision become inputs to a bounded implementation plan:

- `basis-gateway`: require derivation, and reject composite or typed values, for callers classified as admitted operation producers (Decision 6.4). Today the operation-aware path reuses dual-accept.
- `basis-producer`: capture and record the effective mapping identity and mapping provenance (Decisions 9, 10). Today the REST reference mapping is fixed in code and has no identity.
- `basis-adapters`: expose, or allow a caller to obtain, an identity for the adapter release plus its mapping configuration. Confirm that no adapter supplies resource identity by silent fallback (Decision 5).
- Deployment tooling: adoption-time mapping validation (Decision 8).

An intake realization additionally remains gated on every ADR-0018 prerequisite.

## References

- [ADR-0007](0007-adapter-evidence-construction.md): adapter evidence construction
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) / [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md): producer authentication and admission
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md): operation-producer runtime
- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md): Decisions 7, 9, 10, 18
- [ADR-0012](0012-authorization-to-execution-binding.md): authorization-to-execution binding
- [ADR-0014](0014-minimum-execution-evidence-semantics.md): execution evidence
- [ADR-0017](0017-action-vocabulary-naming-structure.md): canonical verbs; composition declared out of scope
- [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md): producer-intake boundary
- [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Proposed): Basitra terminology
- [`action-vocabulary.md`](../architecture/action-vocabulary.md), [`action-vocabulary-reconciliation.md`](../architecture/action-vocabulary-reconciliation.md) (I-1, §5.2)
- [`resource-identifier-reconciliation.md`](../architecture/resource-identifier-reconciliation.md)
- [`ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md): §3.3–§3.6, §3.11
- [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md): §2, §4, §8
- [`compatibility-philosophy.md`](../architecture/compatibility-philosophy.md): adapter-normalization compatibility
- [`threat-model.md`](../security/threat-model.md)
- Ipotio architecture repository: ARCH-GAP-002 and the execution-role and context boundary (cross-repository reference, not hyperlinked)
