# basis-schemas — Planning and Charter

**Status:** Planning and governance document. Defines the charter and scope of the future `basis-schemas` repository. It does **not** create the repository, define any schema, or resolve any open contract question.
**Scope:** Why `basis-schemas` should exist, what belongs in it, what does not, how it should be governed and versioned, and what should move first — grounded in the contracts that already exist in practice.
**Foundation:** [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md) is the inventory of contracts this document plans to formalize. See also [`action-vocabulary.md`](action-vocabulary.md), [`action-vocabulary-reconciliation.md`](action-vocabulary-reconciliation.md), [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md), [`compatibility-philosophy.md`](compatibility-philosophy.md), [`../security/threat-model.md`](../security/threat-model.md), and [`../glossary.md`](../glossary.md).

---

## 1. Purpose

```text
basis-schemas exists to formalize contracts
that already exist across the ecosystem.
```

`basis-schemas` is not a place to invent new architecture. It is a place to **publish shared contracts** that the implementation repositories already depend on, so they are defined once and consumed everywhere rather than re-derived independently in each repository.

The need is concrete, not speculative. The [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md) records contracts that are already implemented and exercised by tests across `basis-core`, `basis-gateway`, `basis-adapters`, and `basis-console` — an action vocabulary, an action string format, a resource-identifier format, the adapter normalized-request shape, the gateway composition rules, the decision request/response contracts, the audit event shape, and a reserved gateway evidence namespace. Several of these are currently mirrored or re-declared in more than one repository, and the reconciliation reports document the drift that follows when the same contract lives in several places at once.

`basis-schemas` is the intended resolution: a single, neutral home for the contracts every component must agree on. This document is its charter. It states the repository's role before any schema is written, so that a future contributor understands *why* the repository exists and *what* it is for before authoring a single definition.

This document deliberately stops short of design. It names candidates, not schemas; it records boundaries, not field lists; and it preserves — rather than resolves — the open questions carried from the inventory and the reconciliation reports.

---

## 2. Why A Separate Repository

The contracts in question are **shared**: no single implementation repository is their natural owner. Placing them inside any one component makes that component the de-facto authority for a contract its peers must also honor, which is the exact failure mode the reconciliation work catalogued. A separate repository exists to remove that accidental authority.

Four reasons motivate the separation:

- **Shared ownership.** The decision request/response, the action vocabulary, the audit event, and the request shapes are consumed by multiple components. If they live in `basis-core`, the gateway and adapters depend on the kernel for a contract that is not purely the kernel's; if they live in `basis-adapters`, the kernel depends on an adapter. A neutral repository lets every component depend on the contract without depending on a peer's implementation.

- **Contract neutrality.** A schema repository carries definitions and compatibility rules, not behavior. Keeping it free of policy logic, authentication, protocol code, or UI means the contract can be read, reviewed, and tested as a contract — without pulling in a component's runtime concerns. This neutrality is what makes the contract citable as a shared source of truth (the threat model already references "the canonical schema in `basis-schemas`").

- **Compatibility governance.** Cross-component compatibility is a property of the *contract*, not of any one consumer. A breaking change to the audit event shape affects the kernel, the gateway, and every audit consumer simultaneously. Governing compatibility in one place — where the change is visible as a change to the shared definition — is more reliable than discovering it through downstream breakage.

- **Independent versioning.** A shared contract should be able to version on its own cadence, decoupled from any single component's release. The kernel, the gateway, and the adapters each release on their own schedule; the contract they share should not be forced to ride one of those schedules, nor should a contract revision require a coordinated lockstep release of every consumer.

The dependency direction stays consistent with the rest of the distribution: `basis-schemas` sits **below** the implementation repositories in the strict, downward-only dependency graph described in [`../security/threat-model.md`](../security/threat-model.md). Components depend on the schemas; the schemas depend on nothing in the distribution.

---

## 3. What Belongs In basis-schemas

The following are the candidate contracts, drawn directly from the [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md). They are listed here to define scope, not to design them. Each already exists in implementation today, which is precisely why it is a candidate for a single shared definition.

**Action vocabulary.** The five canonical verbs:

```text
read
write
execute
browse
subscribe
```

`basis-schemas` would publish these as the machine-readable companion to the governance rules in [`action-vocabulary.md`](action-vocabulary.md), which remains the human-readable source of *why*.

**Action string schema.** The composite action-name format `{verb}:{domain}[:{object}]`:

```text
read:ahu
write:setpoint
execute:command
```

The shape is owned today by `basis-core`'s format regex and mirrored by the console; `basis-schemas` would hold the single definition.

**Resource identifier schema.** The canonical typed identifier `{type}:{qualifier}`:

```text
ahu:rooftop-1
sensor:co2:lobby
```

**Normalized request schema.** The adapter/console normalization shape:

```text
action
resource_type
resource_id
context/evidence
```

Today this is `basis-adapters/schemas/normalized-authorization-request.schema.json`. The composition rules that turn this normalized shape into canonical kernel inputs are a gateway concern, but the *shape* itself is a shared contract.

**Decision request schema.** The kernel input — subject, composite action, optional canonical resource identifier, and context. Owned today by `basis-core`'s `DecisionRequest`.

**Decision response schema.** The kernel output — outcome, reason, evaluating policy, policy version, optional failure reason, timestamp. Owned today by `basis-core`'s `DecisionResponse`.

**Audit event schema.** The canonical audit structure, including its schema version and the action-vocabulary version field that lets consumers interpret historical records across vocabulary evolution. Owned today by `basis-core`'s `AuditEvent`, with the gateway as a producer.

**Reserved evidence namespace rules.** The rule that the gateway's composition and audit metadata namespace is reserved and callers must not write to it:

```text
basis_gateway.*
```

`basis-schemas` is a candidate home for the *rule* (which keys are reserved and that callers must not supply them), not for the gateway's enforcement of it.

**Schema versioning metadata.** The version and compatibility identifiers a consumer reads to know which revision of a contract it is handling.

**Compatibility snapshots.** Cross-repository validation fixtures — example payloads and expected results that every consumer can test against, so drift surfaces as a failing shared fixture rather than a production incident.

This is a candidate list. Which of these is defined first, and in what form, is decided when the repository is created (see §9), not here.

---

## 4. What Does Not Belong In basis-schemas

This boundary is as important as the inclusion list. `basis-schemas` holds **contracts**, not **behavior**. The following remain owned by their existing repositories, and must not migrate into `basis-schemas` because they are implementations, governance, or topology — not shared data contracts:

- **Policy logic** — how rules are matched and evaluated — belongs in `basis-core`. `basis-schemas` may define the *shape* of a decision request; it must not define how that request is evaluated.
- **Authentication** — token verification, subject derivation, identity at the trust boundary — belongs in `basis-gateway`. `basis-schemas` defines no credential handling.
- **Authorization evaluation** — the decision semantics themselves — belongs in `basis-core`. The contract is the request and response shape; the *decision* is the kernel's.
- **Deployment topology** — how components are arranged, networked, and trusted across sites — belongs in `basis-deploy`.
- **UI workflows** — operator-facing flows, forms, and presentation — belong in `basis-console`.
- **Protocol translation** — the protocol-specific normalization that turns BACnet, Modbus, OPC UA, and the rest into a normalized request — belongs in `basis-adapters`. `basis-schemas` defines the normalized *output* shape; it must not contain protocol logic.
- **Architecture governance** — the reasoning, principles, reconciliations, and ADRs that decide *why* a contract takes the shape it does — belongs in `basis-architecture`. `basis-schemas` publishes the *what*; this repository continues to explain the *why*.

The line is: if it is a shape that multiple components must agree on, it is a candidate for `basis-schemas`; if it is a decision, a behavior, a workflow, or a deployment arrangement, it stays where it is.

---

## 5. Ownership Model

```text
Architecture proposes.
Schemas publish.
Implementations consume.
```

The model keeps responsibility where the existing repositories already place it:

- **`basis-architecture` proposes.** A new or changed shared contract is reasoned about and decided here — in an architecture document or an ADR — before it becomes a schema. `basis-schemas` is not where contracts are *invented*; it is where decided contracts are *published*.
- **`basis-schemas` publishes.** Once a contract is decided, its machine-readable definition, version, and compatibility fixtures live in `basis-schemas` as the single source of truth.
- **Implementations consume.** `basis-core`, `basis-gateway`, `basis-adapters`, and `basis-console` import the published contract rather than re-declaring it. No component re-declares a contract it does not own.

Two ownership principles, carried forward from [`action-vocabulary.md`](action-vocabulary.md):

1. **One definition, many consumers.** Each contract is defined once and imported elsewhere. The provisional copies that exist today — for example `basis-console.vocabulary` — are retired once the shared definition exists.
2. **No component is the de-facto authority by accident.** A vocabulary or schema must not be effectively decided inside a single adapter or the console because that is where it happened to be implemented first.

On change review: a change to a shared schema is reviewed as a change with ecosystem-wide blast radius, under the compatibility rules in [`compatibility-philosophy.md`](compatibility-philosophy.md) and the Basis Foundation governance process. This document does not specify a review mechanism in detail — that is for the repository's own governance when it is created — and deliberately does not over-engineer one in advance.

---

## 6. Versioning Strategy

Principles only; mechanics are deferred to repository creation.

```text
Schemas version independently.
Compatibility must be explicit.
Breaking changes should be rare.
Consumers should know what version they support.
```

- **Schemas version independently.** A shared contract versions on its own cadence, decoupled from any single component's release schedule.
- **Compatibility must be explicit.** Whether a revision is compatible with prior consumers is stated, not inferred. This is what compatibility snapshots (§3) make testable.
- **Breaking changes should be rare and deliberate.** Given the operational timescales the ecosystem targets — audit records retained for years, policies in effect for the lifetime of OT systems — a breaking change to a shared contract is a significant event governed by the deprecation discipline in [`compatibility-philosophy.md`](compatibility-philosophy.md), not a routine revision.
- **Consumers know what they support.** A consumer can identify which contract version it handles (via the schema versioning metadata of §3), so mismatches are detectable rather than silent.

These align with the existing audit `schema_version` and action-vocabulary version fields already present in `basis-core` today; `basis-schemas` generalizes that discipline across all shared contracts rather than introducing a new one.

---

## 7. Relationship To Existing Repositories

Concise view of how each repository relates to `basis-schemas`. "Consumes" = imports shared contracts; "Publishes" = owns contracts others depend on; "Depends on" = the contract direction relative to `basis-schemas`.

- **`basis-core`** — *Consumes:* the shared definitions of the decision request/response, action string, resource identifier, and audit event once they move to `basis-schemas`. *Publishes (today):* those same contracts, as the originating owner, until they migrate. *Depends on:* `basis-schemas` only (downward); it must not depend on the gateway, adapters, console, or deploy.
- **`basis-gateway`** — *Consumes:* the action string, resource identifier, normalized request, decision request/response, and audit event schemas. *Publishes (today):* the composition rules and the reserved `basis_gateway.*` namespace rule (the namespace rule is a `basis-schemas` candidate; the composition *behavior* stays in the gateway). *Depends on:* `basis-schemas` and `basis-core`.
- **`basis-adapters`** — *Consumes:* the normalized request schema and the action vocabulary. *Publishes (today):* the normalized-authorization-request shape and the verb enum, until they migrate. *Depends on:* `basis-schemas`.
- **`basis-console`** — *Consumes:* the normalized request schema, action vocabulary, and decision response. *Publishes:* nothing canonical; its provisional vocabulary copy is retired once `basis-schemas` exists. *Depends on:* `basis-schemas` and `basis-gateway` (for the evaluate request shape).
- **`basis-architecture`** — *Consumes:* nothing at runtime. *Publishes:* the reasoning, principles, reconciliations, and ADRs that *decide* contracts before they are published as schemas. *Depends on:* nothing; it proposes, it does not import.
- **`basis-deploy`** — *Consumes:* potentially the compatibility metadata, to assemble mutually compatible component versions into a deployable distribution. *Publishes:* deployment and topology contracts, which are out of scope for `basis-schemas`. *Depends on:* `basis-schemas` only insofar as it needs to reason about cross-component compatibility.

The constant across all rows: dependencies point **downward toward `basis-schemas`**, never sideways between implementation repositories.

---

## 8. Contracts Not Yet Ready

The following must **not** move into `basis-schemas` yet. They are unresolved architecture questions, not settled shared contracts, and formalizing them now would bake in a decision before the constraints that should shape it are understood. They remain owned by `basis-architecture` (as open questions) until implementation proves a stable shape:

```text
resource taxonomy
action-domain vs resource-type distinction
audit persistence
deployment trust relationships
operator workflows
policy lifecycle governance
```

- **Resource taxonomy.** The canonical set of resource types — and whether it is a closed enum or an open prefix — is not unified across components (see [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) M-4).
- **Action-domain vs resource-type distinction.** Whether an action's `{domain}` segment and a resource's `{type}` prefix are one concept or two is unresolved; the gateway currently derives both from one `resource_type` field. This is recorded — not resolved — in [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) §9 and the [`../security/threat-model.md`](../security/threat-model.md) open questions. This document does **not** resolve it.
- **Audit persistence.** The audit *event shape* is a contract candidate (§3); durable storage, immutability guarantees, and gateway/kernel correlation are pipeline concerns without a settled contract.
- **Deployment trust relationships.** Cross-site trust and topology belong to `basis-deploy` and remain architecture questions.
- **Operator workflows.** What an operator does and how those flows map to enforcement boundaries is `basis-console` behavior, not a shared schema.
- **Policy lifecycle governance.** Authoring, distribution, and retirement of policies beyond a single version field are not yet contracted.

When any of these stabilizes into a shape multiple components must agree on, it may graduate from this list into §3.

---

## 9. Initial Migration Candidates

If the repository were created tomorrow, the contracts should move in roughly this order:

```text
1. Action vocabulary
2. Action string schema
3. Resource identifier schema
4. Decision request schema
5. Decision response schema
6. Audit event schema
```

The ordering follows dependency and stability, lowest-risk first:

1. **Action vocabulary** — the smallest, most-settled contract (five verbs), already governed by a mature document. Moving it first proves the publish-and-consume mechanism on low-risk content and lets the provisional console copy be retired immediately.
2. **Action string schema** — depends only on the vocabulary; it is the format that wraps the verbs. Stable in `basis-core` today.
3. **Resource identifier schema** — the parallel format contract, also stable in `basis-core`. Together with the action string it covers the two canonical identifier shapes.
4. **Decision request schema** — composes the action string and resource identifier into the kernel input. It can only follow once both of its constituent formats are published.
5. **Decision response schema** — the kernel output; pairs with the request and is independently stable.
6. **Audit event schema** — last of this set because it is the broadest (it references action, resource, and policy version) and its surrounding pipeline is still emerging; moving the *shape* is safe, but it benefits from the earlier contracts being settled first.

The normalized request schema, the reserved namespace rule, and the compatibility snapshots are strong candidates but are intentionally **not** in this first wave: the normalized request depends on the still-open action-domain/resource-type question, and the namespace rule and snapshots are better introduced once the core six establish the repository's conventions. Their sequencing is left to the repository's own planning.

This is a suggested order, not a commitment. The repository's creators decide the actual sequence in light of conditions at the time.

---

## 10. Success Criteria

```text
basis-schemas succeeds when:

- contracts are published once
- implementations consume them consistently
- compatibility becomes easier
- contract drift becomes harder
```

Concretely, the repository has achieved its purpose when:

- each shared contract has exactly one authoritative definition, and no implementation repository re-declares a contract it does not own;
- `basis-core`, `basis-gateway`, `basis-adapters`, and `basis-console` consume those definitions rather than mirroring them, and the provisional copies (notably `basis-console.vocabulary`) are gone;
- compatibility between component versions can be reasoned about and tested against shared fixtures, rather than discovered downstream;
- a contract change is visible as a change to a shared definition with explicit compatibility consequences, making silent drift between components hard rather than easy.

A future contributor should be able to read this document and understand exactly why `basis-schemas` exists, what it is and is not for, and what to move first — before writing a single schema.

---

## Non-goals of this document

This planning document does **not**: create the `basis-schemas` repository; define any schema, JSON Schema file, or OpenAPI specification; define the resource taxonomy; resolve the action-domain vs resource-type question; define audit persistence; define deployment models; or modify any implementation repository. It stops at the charter and scope, which is where a planning document should stop.
