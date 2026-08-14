# Post-Authentication Identity Activity, Correlation, and Detection Roadmap

This document defines the long-term architectural direction for extending BASIS from deterministic authorization and trustworthy evidence into durable identity activity, correlation, deterministic detection, investigation, behavioral analytics, and bounded response for operational technology (OT). It is an architecture-planning document, not an implementation plan. It does not implement storage, implement a graph, implement detections, create new repositories, publish schemas, add dependencies, choose a database, choose a stream processor, choose a graph technology, choose a machine-learning platform, modify the current `basis-gateway` implementation, or interrupt the active operation-aware gateway roadmap. It records long-term intent so the idea is not lost while the ecosystem's present implementation effort continues, and it defines the architectural boundaries, decision gates, and completion criteria that later implementation work — whenever it begins — must satisfy.

**Status:** Planned only. No implementation is authorized by this document. Deterministic authorization and trustworthy evidence, as established by `basis-core`'s operation-aware kernel and the identity-to-operation contract this roadmap builds on, remain prerequisites, and this roadmap does not begin until the identity-to-operation contract and evidence foundations it depends on are sufficiently stable. The current `basis-gateway` operation-aware rollout — including the operation-aware gateway audit-evidence phase, currently planned as PR 7, and the readiness, conformance, and release-hardening work that surrounds it — continues uninterrupted and is not reordered, paused, or deprioritized by anything in this document; no phase described below is in progress. Conceptual component names introduced in this roadmap (`basis-activity`, `basis-topology`, `basis-detect`) are conceptual references to future architectural roles, not committed repository names, consistent with how [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) already treats its own conceptual repository references. Deterministic detection comes before behavioral analytics or machine learning throughout this roadmap's sequencing, and automated response is bounded, human-governed, and late in the sequence, per **Architectural Invariants** below.

A note on terminology: this roadmap introduces working vocabulary — normalized identity-activity, operation-chain correlation, deterministic OT identity detection, bounded response, investigation graph — that does not yet have entries in [`docs/glossary.md`](../glossary.md). Consistent with the glossary's role as a canonical reference for decided terminology rather than speculative vocabulary, formal glossary entries for these terms should be added only as each phase's architecture stabilizes through its own ADR, not as part of this roadmap. No term introduced here is promoted to canonical status by its appearance in this document.

---

## Purpose

[`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) already anticipated this roadmap without writing it: its **Relationship to Existing Roadmaps** section notes that "a future identity-activity roadmap — if the ecosystem pursues one, covering the durable historical record of identity and operation activity over time, as distinct from the point-in-time contract this roadmap defines — would depend on the evidence and execution-lifecycle architecture this roadmap establishes," and records that dependency precisely so a later roadmap would not be designed independently of the evidence model it develops. This document is that later roadmap, written against the dependency the identity-to-operation roadmap already named.

OT environments fragment identity and activity across identity providers, VPNs, jump hosts, PAM systems, shared engineering accounts, supervisors, gateways, adapters, native OT protocols, controllers and devices, asset inventories, network sensors, SIEMs, ticketing systems, and operator workflows. Each of these systems holds part of the truth about who did what — an identity provider knows who authenticated; a PAM system knows who checked out a credential; a gateway knows what was authorized; a protocol adapter knows what a device was asked to do; a SIEM knows what looked anomalous — and none of them, individually or in combination today, can reliably answer "who did what, under what authority, with what result" across the full chain from authentication to native execution. A detection platform built on top of that fragmentation inherits its ambiguity: it cannot reliably distinguish an authorized operator from an impersonated one, or a completed command from a failed one, if the systems it draws from do not first agree on what an identity, an operation, a decision, and an execution result even are.

The identity-to-operation contract this roadmap builds on exists precisely to remove that ambiguity at the point where an operation is authorized, enforced, and (where confirmable) executed. This roadmap's purpose is to define, at the same architecture-planning rigor, how that contract's output — durable, trustworthy, attributable evidence — could eventually be normalized into activity history, related across identities and infrastructure, correlated into meaningful operation chains, screened by deterministic detections, investigated by human operators, refined by tightly bounded behavioral analytics, and — only late, and only under human governance — acted on through bounded response. Therefore:

> Post-authentication identity activity must be built on top of the BASIS identity-to-operation contract, not beside it.

The potential strategic outcome this roadmap identifies is an OT-specific identity activity and investigation layer that follows authority from authentication through producer, operation, authorization, enforcement, execution, and subsequent activity — answering, over time and across systems, the questions a single point-in-time authorization decision cannot answer on its own. This roadmap does not claim BASIS already provides this. It claims only that the identity-to-operation contract is the correct foundation for it, and that building it on any other foundation would recreate the same fragmentation this roadmap exists to resolve.

Cloud identity threat-detection products (identity threat detection and response platforms, cloud security posture and behavior analytics tooling, and SIEM-adjacent identity analytics products) provide conceptual precedent for the general shape of this problem — normalize activity, correlate it, detect deviations, support investigation, enable bounded response — and this roadmap draws on that precedent as a shape, not as a specification. BASIS must adapt the problem to OT's producer, protocol, device, safety, execution, air-gap, and operational-continuity realities, which differ from the cloud-identity setting in ways that matter architecturally: OT operations have physical consequences that cloud API calls do not; OT execution is frequently unconfirmable within any bounded window; OT deployments are frequently air-gapped or intermittently connected; OT identity is frequently shared, delegated through jump hosts and PAM systems, or exercised by protocol adapters on a human's behalf rather than directly; and OT environments cannot treat automated response the way a cloud environment might treat automatically disabling a compromised account, because bounded physical and safety consequences follow from OT actuation in a way they do not follow from revoking a cloud session. This roadmap is not a plan to clone a cloud product for OT; it is a plan to solve OT's version of the problem those products address, using BASIS's own evidence foundation.

---

## Central Dependency

This roadmap's phases are ordered around one dependency chain, and no phase below is intended to be read independently of it:

```text
identity-to-operation contract
    ↓
trustworthy evidence
    ↓
durable normalized activity
    ↓
correlation
    ↓
detection
    ↓
investigation
    ↓
bounded response
```

Each stage in this chain depends on the stage above it being architecturally sound before it can be trusted. Durable activity that is not built on trustworthy evidence just durably stores untrustworthy data. Correlation over activity that is not normalized correlates noise. Detection built on unreliable correlation produces unreliable findings. Investigation of unreliable findings wastes operator attention. Response to unreliable investigation risks operational harm. This roadmap's sequencing exists specifically to prevent any later stage from being built ahead of the stage it depends on.

---

## Required Status Restated

To avoid any ambiguity beyond the Status paragraph above: nothing in this document authorizes implementation in `basis-core`, `basis-gateway`, `basis-identity`, `basis-adapters`, `basis-console`, or `basis-schemas`. Nothing in this document selects a database, stream processor, graph technology, or machine-learning platform. Nothing in this document creates a new repository; the conceptual component names used throughout (`basis-activity`, `basis-topology`, `basis-detect`) name architectural roles for reasoning purposes only. Nothing in this document implies any phase below is currently underway. The current `basis-gateway` operation-aware integration program remains the ecosystem's active implementation priority, exactly as [`ROADMAP.md`](../../ROADMAP.md) and [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) already state, and this roadmap changes none of that sequencing.

---

## Architectural Invariants

The following invariants are preconditions for every phase in this roadmap. They restate and extend boundaries already established in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/security/threat-model.md`](../security/threat-model.md), [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md), and [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md). Nothing in this section supersedes those documents; where language differs, this section is additive. A phase whose implementation would require relaxing one of these invariants should be treated as evidence that the phase has been scoped to the wrong component, not as grounds for an exception.

### Kernel boundary

`basis-core` must not absorb durable historical activity, session storage, event streaming, event correlation, graph traversal, baselining, anomaly scoring, machine learning, behavioral analytics, investigation workflow, alert lifecycle, or automated response. The kernel continues to produce deterministic decisions and trustworthy bounded evidence, exactly as the Kernel invariant in [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) already states. Nothing in this roadmap asks the kernel to become the interoperability bus, the activity store, or the detection engine; everything this roadmap adds sits downstream of the kernel's evidence output.

### Gateway boundary

`basis-gateway` may emit complete enforcement-boundary evidence. It must not become the long-term activity ledger, the correlation engine, the detection engine, the investigation platform, or the response orchestrator. Gateway availability and latency must not become dependent on downstream analytics availability — evidence export from the gateway toward any future activity capability is decoupled and failure-aware, so that a downstream outage degrades analytics, not enforcement.

### Adapter boundary

Adapters or native enforcement integrations may emit execution telemetry, and must distinguish at least: command accepted, command transmitted, native acknowledgement received, state confirmed, failure, partial application, and unknown result — the same execution-lifecycle distinctions the Execution invariant in [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) already requires. Adapters must not claim an authorization decision or reinterpret one, consistent with the Adapter invariant that roadmap already states.

### Activity boundary

A future activity capability may normalize, persist, index, retrieve, and export activity. It must not authorize operations, reinterpret kernel decisions, become the source of identity truth, fabricate missing execution results, silently merge conflicting evidence, or make topology facts authoritative without provenance. An activity record is a durable representation of what evidence said occurred; it is never an independent claim about what was permitted or what actually happened beyond what the evidence supports.

### Topology and relationship boundary

A future topology or relationship capability may model links among identities, sessions, producers, gateways, adapters, devices, resources, facilities, operations, and evidence. It does not become the final authorization authority. This activity-investigation graph is explicitly distinct from the ReBAC/FGA relationship model [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 7 through Phase 9 develop for authorization purposes:

```text
activity/investigation relationships
    describe observed activity and infrastructure relationships
    support correlation, investigation, and detection

authorization/ReBAC relationships
    describe entitlement — who may act on what, through what relation
    support the fine-grained authorization query APIs that roadmap defines
```

One graph may describe observed activity and infrastructure relationships. Another may represent authorization relationships. This roadmap does not assume they share storage, ownership, or semantics merely because both are graph-shaped, and no phase below should be read as merging them.

### Detection boundary

Detections consume normalized activity and evidence. They do not alter historical evidence. Detection findings remain separately attributable to the rule, rule version, rule evaluation result, input evidence, correlation basis, evidence completeness, evaluation time, and suppression or exception state that produced them. A deterministic finding records whether a governed rule matched its inputs; it does not carry a probabilistic confidence score. This roadmap reserves probabilistic confidence, anomaly scores, baseline deviation, and model uncertainty for Phase 7 behavioral analytics, and uses the following terminology consistently everywhere else:

```text
Deterministic rule result
    whether the rule matched its governed inputs

Correlation certainty
    whether the events were explicitly linked or inferred

Evidence completeness
    whether required or expected evidence was missing

Behavioral confidence or uncertainty
    uncertainty associated with a Phase 7 analytic finding
```

A deterministic finding built on inferred correlation or incomplete evidence still honestly exposes that inherited correlation certainty and evidence completeness; the distinction above governs vocabulary, not disclosure — deterministic detections do not become falsely definitive merely because they avoid probabilistic language. Deterministic detections come first in this roadmap's sequencing, per **Architectural Invariants** above and **Roadmap Phases** below.

### Analytics boundary

Behavioral analytics may rank, summarize, or identify deviations. They must not replace deterministic policy evaluation, rewrite historical facts, autonomously authorize or execute OT operations, conceal uncertainty, or turn a probabilistic score into an unreviewable enforcement decision. A behavioral finding is an analytic hypothesis, not a fact and not a decision.

### Response boundary

Response is bounded. The default posture is:

```text
detect
    → explain
    → recommend
    → human review
    → deterministic authorization
    → gateway enforcement
    → adapter execution
    → evidence
```

No autonomous southbound actuation is authorized by this roadmap. Any future automated containment, if ever allowed, would require an explicit policy, a bounded action vocabulary, affected-resource limits, dual control where appropriate, step-up authentication, a rollback or recovery path, full evidence, operator visibility, and a dedicated ADR and threat model of its own — none of which exist today, and none of which this roadmap creates.

### Console boundary

`basis-console` may investigate, explain, and submit approved workflows. It does not become the activity store, the graph authority, the detection authority, or the response enforcement point, consistent with the console invariant already established in [`docs/architecture/basis-console.md`](../architecture/basis-console.md) and restated by both roadmaps this document builds on.

### AI boundary

```text
AI may summarize, correlate candidate evidence, and assist investigation.
AI does not decide historical truth.
AI does not authorize.
AI does not silently create detections.
AI does not actuate OT systems.
Humans approve.
Gateway enforces.
Core evaluates.
Adapters execute.
Evidence records.
```

---

## Training-Mode Constitutional Requirement

Consistent with the constitutional requirement already established in [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) and restated in [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md), every externally observable capability this roadmap introduces must have a corresponding operator-facing representation and training-mode explanation in `basis-console` from the outset of its design, not as work added after a capability ships. Training mode may explain activity normalization, relationship provenance, operation-chain correlation, deterministic detection logic, investigation evidence handling, behavioral-finding uncertainty, and response approval and enforcement. Training mode must not change backend behavior, bypass authentication or authorization, fabricate live data, or expose secrets, unredacted credentials, or sensitive asset detail beyond what the current user's role and redaction tier permit. Each phase below states what operator mode needs to show, what training mode needs to explain, what evidence proves the behavior occurred, and what must be redacted; the **Operator and Training-Mode Requirements** matrix near the end of this document summarizes the pattern across phases.

---

## Conceptual Components

The following logical names are used to reason about responsibilities in this roadmap:

```text
basis-core
    deterministic authorization

basis-gateway
    authentication, composition, enforcement, gateway evidence

basis-adapters
    protocol normalization and execution telemetry

basis-activity
    durable normalized identity-action ledger

basis-topology
    identity, producer, device, asset, and relationship graph

basis-detect
    operation-chain correlation and deterministic detections

basis-console
    investigation, explanation, approval, and bounded response workflow
```

These names are conceptual. No new repository is approved by this document. Repository ownership is a future decision gate, recorded in **Decision Gates** below, not a decision made here. Some of these logical responsibilities may ultimately share a deployment or a repository with each other, or with an existing component; architectural boundaries matter more than repository count, and nothing in this roadmap should be read as committing to `basis-activity`, `basis-topology`, or `basis-detect` becoming standalone repositories.

---

## Central Questions

This roadmap is organized around a set of questions no single existing BASIS component answers on its own, and that a post-authentication identity activity capability would eventually need to answer together: who authenticated; what type of identity it was; whose authority was exercised; whether authority was delegated; which trusted producer submitted the action; which operation was attempted; which canonical OT resource was targeted; what operational and safety context existed; what policy decided; what disposition was enforced; whether native execution was attempted; whether execution completed, failed, partially applied, or remained unknown; what happened immediately before and after; which other identities, workloads, devices, assets, and sessions were related; whether the sequence was expected; which deterministic rule identified it as suspicious; what evidence supports the detection; what action an operator took; and whether any response was approved and applied. Each of these questions is addressed by at least one phase below, and none of them is fully answerable by the identity-to-operation contract alone — the contract establishes trustworthy facts about a single operation; this roadmap is about what those facts become when accumulated, related, and examined over time.

---

## Roadmap Phases

The following ten phases are architecture phases, not a predetermined pull-request schedule. Each phase uses a consistent structure: purpose, primary conceptual components, prerequisites, architectural outcome, key capabilities, security and abuse cases, distributed-systems concerns, decision gates, completion criteria, operator-mode representation, training-mode explanation, evidence and audit requirements, schema and documentation impact, and explicitly deferred work.

### Phase 1 — Deterministic Authorization and Trustworthy Evidence

**Purpose.** State the prerequisites this roadmap depends on and that current and active roadmap work supplies, without pretending all of them are complete today.

**Primary conceptual components.** `basis-core`, `basis-gateway`, `basis-adapters`, `basis-identity`; no new conceptual component is introduced by this phase.

**Prerequisites.** None beyond what is already in progress or planned elsewhere; this phase largely points to existing work rather than defining new architecture.

**Architectural outcome.** This roadmap depends on: an authenticated subject; authority and delegation evidence; producer trust; a normalized action and resource; operation-aware context; a kernel decision; a gateway disposition; a gateway audit event; execution telemetry where available; correlation identifiers; provenance; version stamps; and governed timestamps. These are supplied, in whole or in part, by `basis-core`'s operation-aware kernel, the active `basis-gateway` operation-aware integration program (including its operation-aware gateway audit-evidence work), `basis-adapters`' execution telemetry, `basis-identity`'s canonical identity context, and the identity-to-operation contract [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) defines. Not all of these are complete today: the gateway's operation-aware integration is in progress, not released; the identity-to-operation contract is itself planned, not implemented; and execution-result vocabulary remains an open decision gate in that roadmap's Phase 4. This phase does not claim otherwise.

**Key capabilities.** No new capability is introduced by this phase. Its function is to name the readiness gate the rest of this roadmap depends on: durable activity storage (Phase 2) should not begin until the evidence it would durably store is itself trustworthy, attributable, and stable enough that redesigning the evidence model would not immediately invalidate accumulated activity history.

**Security and abuse cases.** None directly introduced by this phase; it inherits every security and abuse case already named in [`docs/security/threat-model.md`](../security/threat-model.md) and in the two roadmaps this document builds on.

**Distributed-systems concerns.** None directly introduced by this phase.

**Decision gates.** The readiness gate for moving into Phase 2 is itself a decision gate: what evidence would demonstrate that the identity-to-operation contract and the gateway's audit-evidence model are stable enough to build durable storage on top of, without that storage needing to be redesigned as soon as the evidence model changes underneath it. This roadmap does not resolve that gate; it names it as the condition Phase 2 must satisfy before implementation planning begins.

**Completion criteria.** Not applicable as an implementation milestone; this phase is complete, for this roadmap's purposes, when the identity-to-operation contract and the gateway's operation-aware audit-evidence work reach the completion points their own roadmaps define.

**Operator-mode representation.** Not applicable; this phase has no new operator-facing surface.

**Training-mode explanation.** Not applicable; this phase's evidence model is explained by the roadmaps it points to.

**Evidence and audit requirements.** None beyond what the identity-to-operation contract and the current gateway program already require.

**Schema and documentation impact.** None; this phase proposes no schema.

**Explicitly deferred work.** Everything downstream of this phase, until its readiness gate is satisfied.

### Phase 2 — Durable Normalized Identity-Activity Ledger

**Purpose.** Define the architectural outcome for append-oriented, durable storage of identity activity, without choosing a storage technology.

**Primary conceptual components.** `basis-activity`, consuming evidence from `basis-gateway` and `basis-adapters`.

**Prerequisites.** Phase 1's readiness gate.

**Architectural outcome.** A conceptual normalized-event envelope distinguishes the source event a gateway, adapter, or identity system emitted from the normalized event the ledger stores; distinguishes event time (when the underlying occurrence happened) from ingest time (when the ledger recorded it); carries identity and producer provenance; carries operation-chain identifiers usable by Phase 4's correlation; and is designed for duplicate delivery, out-of-order arrival, and air-gapped, delayed ingestion from the outset, rather than assuming continuous, ordered, exactly-once delivery. The ledger preserves facts as evidence reported them, not interpretations of those facts — interpretation is Phase 5's and Phase 7's concern, not this one's.

**Key capabilities.** Idempotent ingestion under duplicate delivery; explicit handling of late-arriving events, including events that arrive after a related event they logically precede; a defined correction model — tombstones or superseding records, not silent overwrites — for evidence that must be corrected after ingestion; retention as a governed, not merely operational, concern; redaction consistent with the redaction-tier discipline already established for operation-aware trace and audit evidence in [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md); integrity verification sufficient to detect tampering after the fact; export for downstream consumers named in Phase 9; air-gapped ingestion paths; and failure isolation from the gateway, so that a ledger outage degrades activity history, never enforcement.

**Security and abuse cases.** Evidence tampering after ingestion; event deletion; event injection of fabricated activity; replay of previously ingested events presented as new; and duplicate-delivery handling that silently double-counts an operation rather than deduplicating it.

**Distributed-systems concerns.** At-least-once delivery and idempotency as a first-class design requirement; event ordering and the explicit acceptance that global ordering is not guaranteed; late arrival and correction semantics; retention and redaction as governed policies rather than storage defaults; and the critical invariant this roadmap states once and applies throughout: downstream activity or detection failure must not silently weaken authorization enforcement, so ledger unavailability degrades this phase's own capability, never the gateway's.

**Decision gates.** Storage model; append-only versus correction/supersession semantics; source-event versus normalized-event retention periods; and the integrity and tamper-evidence model. Named in **Decision Gates** below; none resolved here.

**Completion criteria.** Not applicable as a testable outcome until implementation exists; this phase's architectural completion is a reviewed envelope model and failure-isolation design, not a running ledger.

**Operator-mode representation.** Searchable identity and operation history for a given subject, producer, or resource.

**Training-mode explanation.** The distinction between the source event and the normalized event, and between event time and ingest time.

**Evidence and audit requirements.** Every normalized activity record retains a reference back to the source evidence it was derived from, sufficient to reconstruct why the record looks the way it does.

**Schema and documentation impact.** A future normalized-activity-envelope contract is a candidate for `basis-schemas`, deferred until this phase's architecture and a reference implementation exist, following the same readiness discipline ADR-0005 established.

**Explicitly deferred work.** Storage technology; exact retention periods; final envelope field names.

### Phase 3 — Identity, Producer, Device, Asset, and Evidence Relationship Graph

**Purpose.** Define an investigation-oriented relationship model connecting the entities activity records reference, distinct from the authorization relationship model [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) develops.

**Primary conceptual components.** `basis-topology`, coordinating with `basis-activity`.

**Prerequisites.** Phase 2's normalized envelope, since relationships are described relative to the entities the envelope names.

**Architectural outcome.** A conceptual relationship model connects authenticated identities, delegated actors, sessions, credentials, producers, gateways, adapters, resources, devices, facilities, operations, decisions, execution results, evidence, and detections, for the purpose of investigation and correlation — not for the purpose of authorization. This roadmap explicitly distinguishes activity/investigation relationships from authorization/ReBAC relationships, per the Topology and relationship boundary above, and coordinates with the existing identity/FGA roadmap rather than redefining relationship semantics independently.

**Key capabilities.** A relationship model expressive enough to represent the entity types above; explicit provenance for every relationship — where it was observed, from what evidence, and when — so topology facts are never authoritative without a traceable source; and an explicit non-decision about whether this capability belongs in `basis-topology`, `basis-activity`, or a shared persistence layer, left to a future decision gate.

**Security and abuse cases.** Graph poisoning, where fabricated or manipulated evidence produces an incorrect relationship; topology poisoning, where an attacker causes the graph to misrepresent infrastructure relationships in a way that later misleads correlation or detection; and identity stitching across unrelated people, where imprecise correlation incorrectly merges two distinct identities' activity into one apparent actor.

**Distributed-systems concerns.** Consistency between the graph and current device or topology state, which changes independently of activity ingestion; and consistency between the graph and the ledger it is built from, since the two may update at different rates.

**Decision gates.** Graph storage and ownership; and the relationship between this topology graph and the ReBAC/FGA graph — whether they ever share infrastructure while remaining semantically distinct. Named in **Decision Gates** below.

**Completion criteria.** A reviewed relationship model, coordinated explicitly with the identity/FGA roadmap's Phase 7 through Phase 9, that a future ADR could adopt without contradiction.

**Operator-mode representation.** Related identities, producers, assets, and sessions for a given activity record or investigation.

**Training-mode explanation.** Why a given relationship exists and what evidence establishes its provenance.

**Evidence and audit requirements.** Every relationship carries provenance sufficient to answer why it is represented, distinguishing an observed relationship from an inferred one.

**Schema and documentation impact.** A future relationship-record contract is a candidate for `basis-schemas`, deferred until architecture and a reference implementation exist.

**Explicitly deferred work.** Graph storage technology; final ownership between `basis-topology`, `basis-activity`, or another component.

### Phase 4 — Session and Operation-Chain Correlation

**Purpose.** Define how normalized activity events become meaningful chains rather than an unordered collection of independent facts.

**Primary conceptual components.** `basis-detect`, consuming `basis-activity` and `basis-topology`.

**Prerequisites.** Phase 2's normalized ledger and Phase 3's relationship graph.

**Architectural outcome.** A conceptual correlation model connects authentication, session establishment, token exchange or delegation, producer submission, authorization request, decision, enforcement, protocol execution, state confirmation, subsequent operation, and logout or session termination into an operation chain, where such a chain is explicitly derivable from the evidence rather than assumed. The model distinguishes explicit correlation — evidence that directly links two events, such as a shared correlation identifier — from inferred correlation — a chain assembled from proximity in time, shared identity, or shared resource, which carries materially higher false-correlation risk and must be represented as inferred, never presented with the same correlation certainty as an explicit link.

**Key capabilities.** Session identifiers, request and correlation identifiers, and delegation chains as inputs to explicit correlation; handling for shared accounts, identity switching, and jump hosts, where a single apparent actor may represent more than one real actor and correlation must not silently resolve that ambiguity in either direction; handling for multi-step maintenance activity, multiple operations against one asset, fan-out from one automation run, and fan-in from multiple producers; handling for missing events, clock skew, and air-gapped delayed evidence, all of which can break a chain that otherwise would have been complete; and bounded correlation windows, so that inferred correlation does not silently extend across an unbounded time span.

**Security and abuse cases.** False correlation, where unrelated activity is incorrectly chained together, either merging distinct actors or splitting one actor's activity into apparently unrelated pieces; and correlation-window exploitation, where an attacker deliberately spaces actions to avoid being correlated within the model's bounded window.

**Distributed-systems concerns.** Bounded correlation windows as a first-class parameter, not an implementation default; reconciliation when a late-arriving event completes a chain that had already been treated as complete without it; and the same clock-skew and air-gapped-delay concerns Phase 2 already names, now applied to chain assembly rather than single-event ingestion.

**Decision gates.** Explicit versus inferred correlation representation; correlation-window limits; and the handling rule for shared accounts. Named in **Decision Gates** below.

**Completion criteria.** A reviewed correlation model that names, for at least the scenarios above, whether the resulting chain is explicit or inferred and what correlation certainty that distinction carries.

**Operator-mode representation.** An ordered activity sequence for a given identity, session, or resource.

**Training-mode explanation.** Which inputs produced a given correlation, which events are missing, and where uncertainty exists in the resulting chain.

**Evidence and audit requirements.** Every correlated chain retains a reference to the events and relationships that produced it, and an explicit correlation-certainty classification distinguishing explicit from inferred correlation.

**Schema and documentation impact.** A future operation-chain contract is a candidate for `basis-schemas`, deferred until architecture and a reference implementation exist.

**Explicitly deferred work.** Final correlation-window values; final correlation-certainty vocabulary.

### Phase 5 — Deterministic OT Identity Detections

**Purpose.** Define deterministic, explainable detection concepts, implemented before any behavioral analytics, per the Detection boundary and the Analytics boundary above.

**Primary conceptual components.** `basis-detect`, consuming Phase 4's correlated chains.

**Prerequisites.** Phase 4's correlation model.

**Architectural outcome.** A conceptual catalog of deterministic detection categories — not final detection content — that a future implementation could build from. Example categories include: an authenticated identity using an unexpected producer; a producer asserting context outside its trust classification; an operation against an asset outside its expected site or scope; a shared engineering identity followed by a privileged control operation; an after-hours write or execute operation; an unusual delegation chain; a repeated deny followed by an allow; an authorization allow followed by execution on a different resource than the one authorized; missing execution evidence after a high-impact allow; an operation sequence violating an approved maintenance workflow; rapid operations across unrelated facilities; a workload identity used interactively; activity following a revoked or expired session; break-glass use without required follow-up evidence; a policy-version mismatch across an operation chain; a new or unrecognized producer interacting with critical resources; activity inconsistent with known topology; and execution continuing after authority was revoked. Every detection built from this catalog must be deterministic, versioned, explainable, evidence-linked, testable, suppressible through governed exception handling, bounded in scope, and kept separate from the original evidence it was derived from — a finding is never written back into the evidence it evaluated.

**Key capabilities.** A rule representation precise enough to be tested and versioned; a suppression and exception model governed, not ad hoc; explicit separation between a detection finding and the evidence supporting it, so a finding can be revised or retired without altering historical evidence; and an explicit requirement that a finding honestly exposes the correlation certainty and evidence completeness it inherited from Phase 4 — a finding built on inferred correlation or incomplete evidence is never presented with the same certainty as one built on explicit correlation and complete evidence, per the Detection boundary's terminology above.

**Security and abuse cases.** Malicious detection rules, where a compromised or poorly reviewed rule produces false or misleading findings; suppressed high-severity findings, where suppression is used to hide rather than legitimately triage; alert flooding, where a rule or set of rules produces enough findings to overwhelm operator attention as a denial-of-service pattern against the investigation workflow; and detection denial of service, where an attacker deliberately triggers detection logic at a rate the system cannot sustain.

**Distributed-systems concerns.** Detection latency relative to the activity it evaluates; and the interaction between detection evaluation and correlation-window boundaries from Phase 4, since a detection that depends on a chain that has not yet fully arrived risks evaluating an incomplete picture.

**Decision gates.** Detection language or rule representation; detection deployment and distribution; and suppression and exception governance. Named in **Decision Gates** below.

**Completion criteria.** A reviewed detection catalog and rule-representation model, with the categories above (or a refined version of them) specific enough that a future contributor could implement a rule engine from it, with no rule engine built as part of this phase.

**Operator-mode representation.** A finding, its severity, the rule that produced it, and the evidence supporting it.

**Training-mode explanation.** Exact deterministic rule evaluation for a given finding — which inputs, which condition, which version.

**Evidence and audit requirements.** Every finding is attributable to its rule ID and version, its rule evaluation result, the input evidence and correlated chain it evaluated, the correlation basis (explicit or inferred) and evidence completeness inherited from that chain, any missing-event indicators, source trust or integrity status, the evaluation time, and its suppression or exception state.

**Schema and documentation impact.** A future detection-finding contract is a candidate for `basis-schemas`, deferred until architecture and a reference implementation exist.

**Explicitly deferred work.** Final detection content beyond the example categories above; final rule-representation syntax.

### Phase 6 — Investigation and Operator Workflows

**Purpose.** Define operator workflows in `basis-console` for investigating activity, correlation, and detection findings.

**Primary conceptual components.** `basis-console`, consuming `basis-activity`, `basis-topology`, and `basis-detect`.

**Prerequisites.** Phases 2 through 5, since investigation workflows are defined over their outputs.

**Architectural outcome.** A conceptual set of console workflows — identity timeline, session timeline, operation chain, resource timeline, producer timeline, decision-and-execution comparison, evidence provenance, related-activity pivoting, deterministic detection explanation, case creation, disposition of findings, false-positive handling, escalation, evidence export, redaction-aware sharing, and training-mode walkthroughs — none of which make the console the owner of evidence or detection truth, consistent with the Console boundary above.

**Key capabilities.** Timeline and pivot views over activity, correlation, and detection data; case creation and disposition as a workflow the console renders and submits, not one it independently decides; redaction-aware evidence export respecting the same redaction tiers Phase 2 establishes; and training-mode walkthroughs that explain, without altering, every workflow above.

**Security and abuse cases.** A compromised console analyst account being used to suppress findings, misclassify cases, or exfiltrate sensitive activity data through export — this is the console-specific instance of the broader compromised-analyst concern named in **Security and Abuse Cases** below.

**Distributed-systems concerns.** Console responsiveness when underlying activity, topology, or detection data is degraded or delayed, and how the console represents that degradation honestly rather than silently showing stale data as current.

**Decision gates.** Investigation case ownership — which conceptual component records case state. Named in **Decision Gates** below.

**Completion criteria.** A reviewed workflow model covering the capabilities above, consistent with the console invariant, with no workflow implemented as part of this phase.

**Operator-mode representation.** Timeline, pivots, notes, and disposition for a given investigation.

**Training-mode explanation.** How evidence is preserved and interpreted throughout an investigation, and what the console can and cannot change.

**Evidence and audit requirements.** Every disposition, escalation, and export action taken in the console is itself evidenced, attributable to the analyst and time it occurred.

**Schema and documentation impact.** None directly; a future case-and-disposition contract is a candidate deferred until reference implementation exists.

**Explicitly deferred work.** Final workflow UI design; case-management data model.

### Phase 7 — Behavioral Analytics

**Purpose.** Define tightly bounded behavioral analytics, pursued only after deterministic detection and normalized historical evidence are mature, per the Analytics boundary above.

**Primary conceptual components.** `basis-detect`, consuming mature Phase 2 through Phase 5 outputs.

**Prerequisites.** Phases 2 through 5, since a behavioral baseline requires normalized history, correlation, and a deterministic detection layer already in place to compare against.

**Architectural outcome.** A conceptual scope for baseline-relative analytics — identity baseline, producer baseline, asset baseline, operation-sequence baseline, time-of-day and maintenance-window context, and peer-group comparison — addressed as candidates for statistical, rule-based, state-machine, or other techniques, without committing to machine learning. Any resulting behavioral score is an analytic finding, never an authorization decision, per the Analytics boundary.

**Key capabilities.** Explicit handling of concept drift, cold start, and sparse OT data, all of which are more severe in OT deployments than in the higher-volume settings behavioral analytics is more commonly built for; explicit handling of seasonal operations and approved change periods, so a baseline does not misclassify expected maintenance activity as anomalous; model or algorithm versioning; explanation and uncertainty as required outputs, not optional ones; a human-feedback loop; evaluation against known scenarios; and explicit consideration of air-gapped inference, since a deployment that cannot reach a networked model-update source continuously cannot depend on one.

**Security and abuse cases.** Behavioral-model poisoning, where an attacker deliberately shapes activity over time to shift a baseline toward tolerating the attacker's eventual action; and explanation leakage, where a behavioral finding's explanation reveals more about the underlying baseline or population than the recipient is entitled to see.

**Distributed-systems concerns.** Baseline consistency across however many nodes or deployments compute it; and the same air-gapped and cold-start concerns named above, which are distributed-systems concerns as much as modeling concerns in an intermittently connected OT deployment.

**Decision gates.** Behavioral analytic techniques; model or baseline versioning; and behavioral confidence or uncertainty vocabulary, kept terminologically distinct from Phase 4 and Phase 5's deterministic vocabulary. Named in **Decision Gates** below.

**Completion criteria.** A reviewed scope document naming the baseline categories above, their known failure modes (drift, cold start, sparsity, seasonality), and the evaluation discipline a future implementation must satisfy, with no model built or technique selected as part of this phase.

**Operator-mode representation.** A deviation and its explanation for a given identity, producer, or asset.

**Training-mode explanation.** The baseline a finding was compared against, its uncertainty, its drift status, and its version.

**Evidence and audit requirements.** Every behavioral finding retains the baseline version, the input window, and the uncertainty associated with it, so a finding can be evaluated for reliability after the fact.

**Schema and documentation impact.** None; this phase defines scope, not a contract.

**Explicitly deferred work.** Technique selection; model architecture; baseline-window defaults.

### Phase 8 — Bounded Response

**Purpose.** Define safe, human-governed response workflows, kept distinct from behavioral analytics per the Response boundary above.

**Primary conceptual components.** `basis-console` for approval and submission; `basis-gateway` and `basis-adapters` for any resulting enforcement or execution, exactly as they already operate for ordinary authorized operations.

**Prerequisites.** Phase 5's deterministic detections and Phase 6's investigation workflows, since response is initiated from a finding and an investigation disposition, not independently.

**Architectural outcome.** A conceptual set of response classes — notify, create investigation, require step-up authentication, require re-approval, revoke a session, revoke a delegated credential, disable a producer trust relationship, suspend a workflow, recommend a policy change, request manual isolation, export evidence to SIEM, PAM, or ticketing systems, and apply a pre-approved bounded control — none of which authorize autonomous device control. Every response requires an initiating detection, an approving identity, policy authorization, a defined response scope, a timeout, a rollback or recovery path, a result, evidence, and failure handling.

**Key capabilities.** A response taxonomy bounded to the classes above; an approval model requiring a human identity and existing BASIS policy authorization for every response, never a detection or analytic finding acting alone; and failure handling for a response that cannot complete, cannot roll back, or produces an unexpected result.

**Security and abuse cases.** Unauthorized response, where a response is applied without the approval and policy authorization this phase requires; automated response causing operational harm, which this phase's bounded, human-governed posture exists specifically to prevent; and a response loop or repeated containment, where a poorly bounded response triggers conditions that cause it to reapply indefinitely.

**Distributed-systems concerns.** Response latency relative to the condition it addresses; and rollback consistency when a response partially applies before failing.

**Decision gates.** The response approval model in full detail, and which responses, if any, may ever be automated beyond the notify and export classes. Named in **Decision Gates** below; this roadmap does not resolve either, and defaults to no autonomous actuation.

**Completion criteria.** A reviewed response taxonomy and approval model, with every response class traceable to a governing policy and an approving identity, with no response mechanism implemented as part of this phase.

**Operator-mode representation.** Proposed, approved, or applied state for a given response.

**Training-mode explanation.** The approval, policy, enforcement, and rollback path a response would follow, walked step by step.

**Evidence and audit requirements.** Every response is evidenced with its initiating detection, approving identity, policy authorization, scope, timeout, result, and any rollback action taken.

**Schema and documentation impact.** A future response-record contract is a candidate for `basis-schemas`, deferred until architecture and a reference implementation exist.

**Explicitly deferred work.** Any future automated-response class beyond notify and export; the specific ADR and threat model any such class would require.

### Phase 9 — Integration and Export Ecosystem

**Purpose.** Define interoperability with systems outside BASIS that already participate in OT identity and security operations.

**Primary conceptual components.** `basis-activity` and `basis-detect` as exporters; external SIEM, SOAR, PAM, identity-provider, VPN, jump-host, asset-inventory, network-detection, ticketing, and data-lake systems as consumers.

**Prerequisites.** Phase 2's export requirement and the identity-to-operation contract's integration-profile work, which this phase extends toward activity and detection data specifically.

**Architectural outcome.** BASIS's activity model remains canonical inside BASIS while preserving source provenance from every external system it exports to or imports evidence from. Interoperability targets include SIEM, SOAR (with the same strict response boundaries Phase 8 establishes), PAM, identity providers, VPN and remote-access logs, jump hosts, asset inventories, network detection systems, ticketing systems, data lakes, offline evidence bundles, and sector-specific reporting formats.

**Key capabilities.** Export formats that preserve provenance rather than flattening it; import handling that keeps an external system's data attributable to its source rather than silently merging it into BASIS-native activity; and offline, air-gapped export and import paths.

**Security and abuse cases.** Cross-tenant activity leakage through an export path that does not respect tenant boundaries established elsewhere in the ecosystem; and sensitive asset-detail leakage through an export that does not apply the same redaction discipline this roadmap requires internally.

**Distributed-systems concerns.** Export and import reliability under intermittent connectivity, and the same air-gapped delivery concerns named throughout this roadmap, now applied to cross-system exchange rather than internal ingestion.

**Decision gates.** None uniquely introduced by this phase beyond what **Decision Gates** below already names for retention, redaction, and privacy.

**Completion criteria.** A reviewed integration model naming the target systems above and the provenance-preservation requirement, with no connector or exporter built as part of this phase.

**Operator-mode representation.** Which external systems a given deployment exports to or imports from, and their integration health.

**Training-mode explanation.** How an exported record preserves its BASIS provenance in an external system, and how an imported record preserves its external provenance inside BASIS.

**Evidence and audit requirements.** Every export and import is itself evidenced, attributable to the system and time involved.

**Schema and documentation impact.** A future export-format contract is a candidate for `basis-schemas`, deferred until architecture and a reference implementation exist.

**Explicitly deferred work.** Specific connector implementations; specific export-format technology.

### Phase 10 — Performance, Retention, Privacy, and Failure Validation

**Purpose.** Define the empirical validation this roadmap's earlier phases assert architecturally, mirroring the validation-phase discipline both roadmaps this document builds on already apply to their own scopes.

**Primary conceptual components.** All conceptual components introduced above; `basis-console` for degraded-state visibility.

**Prerequisites.** Phases 1 through 9, since this phase validates them rather than building new capability.

**Architectural outcome.** A stated validation strategy covering sustained event ingestion, burst ingestion, duplicate delivery, late events, clock skew, source outage, ledger outage without gateway outage, graph inconsistency, detection lag, retention deletion, legal or privacy redaction, key rotation, tamper detection, cross-tenant isolation where a deployment's identity model has tenancy, air-gapped export and import, recovery from partial corruption, degraded console behavior, false-correlation rates, detection precision and recall where measurable, and response failure and rollback — named as properties a future validation effort must test, not as properties already demonstrated by this roadmap.

**Key capabilities.** A validation strategy naming, for each property above, what a successful test would demonstrate and what evidence it would produce; and an explicit acknowledgment that this phase's completion criteria cannot be met until reference implementations of Phases 2 through 9 exist to validate.

**Security and abuse cases.** This entire phase is a security and abuse-case validation exercise; it should specifically re-test the cross-phase concerns named in **Security and Abuse Cases** below under load and failure conditions, not only under normal operation.

**Distributed-systems concerns.** This phase is where every consistency, staleness, and partition-behavior assumption made in Phases 1 through 9 would be empirically verified rather than assumed, once implementation exists to verify against.

**Decision gates.** What specific service-level objectives or bounded guarantees this roadmap's eventual capabilities should target, and what constitutes acceptable degraded-mode behavior versus unacceptable behavior — neither resolved here.

**Completion criteria.** This phase has two distinct completion criteria that must not be conflated. Architecture-planning completion requires a reviewed and accepted validation strategy defining the properties, scenarios, measurements, and evidence a future validation effort must produce; accepting that document completes this phase as architecture. The implementation program's Phase 10, by contrast, is not complete until reference implementations exist, the validation described in the strategy has actually been executed, and empirical evidence supports every production-readiness claim — a state this roadmap does not reach and does not claim to reach by accepting the strategy document.

**Operator-mode representation.** Degraded-capability status: which capability is currently failing or degraded.

**Training-mode explanation.** Safe failure demonstrations showing what degraded activity, correlation, detection, or response capability means, and which facts remain trustworthy during a given degradation.

**Evidence and audit requirements.** Test evidence for every claimed bound or guarantee, retained as the basis for any future production-readiness claim.

**Schema and documentation impact.** None directly; this phase validates prior phases' conceptual guarantees rather than introducing new contract surfaces.

**Explicitly deferred work.** All actual validation execution, which depends on reference implementations of Phases 2 through 9 that do not yet exist.

---

## Activity Record Conceptual Requirements

The following are conceptual needs a normalized activity record may eventually require references to, not a final schema or field list; final field names are not defined here unless already governed elsewhere.

**Identity and authority.** Subject; subject type; issuer; credential source; authority mode; delegated actor; approver; workload or agent-run identifier.

**Producer.** Producer identity; producer type; producer trust classification; producer configuration revision.

**Session and chain.** Session ID; authentication-event reference; token or delegation-exchange reference; operation-chain ID; parent-activity reference; request and correlation IDs.

**Operation and resource.** Canonical action; intent; canonical resource; resource type; site or facility context; protocol context.

**Authorization.** Evaluation status; outcome; failure reason; policy or bundle version; matched-rule or trace reference; disposition.

**Enforcement and execution.** Enforcement point; enforcement time; execution attempt; execution status; native acknowledgement; state confirmation; partial application; execution failure.

**Evidence and integrity.** Source-event reference; normalized-event ID; event time; ingest time; provenance; redaction classification; signature or digest status; contract version.

**Detection and correlation.** Rule ID; rule version; rule evaluation result; input evidence reference; correlation basis (explicit or inferred); evidence completeness; missing-event indicators where applicable; source trust or integrity status; evaluation time; suppression or exception state. This category records what a deterministic finding matched and on what basis, per the Detection boundary's terminology above; it does not carry a probabilistic confidence or anomaly score, which belongs to a separate future behavioral-finding record scoped to Phase 7.

This list draws directly on, and does not redefine, the Contract Responsibility Model already published in [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md); an activity record is a durable accumulation of the same categories that contract already defines for a single point-in-time operation, not a parallel model of what those categories mean.

---

## Distributed-Systems Concerns

Every phase above should be read against a common set of distributed-systems issues that recur across this roadmap: at-least-once delivery; duplicate events; idempotency; event ordering; late arrival; clock skew; source outage; temporary partitions; air-gapped delay; reconciliation; consistency between the ledger and the graph; consistency between revocation and observed activity; consistency between topology and current device state; detection latency; backpressure; replay; poison events; and partial pipeline failure, alongside the requirement, restated from the Gateway boundary above, that the gateway remain independent of downstream analytics availability. The critical invariant governing all of this:

> Downstream activity or detection failure must not silently weaken authorization enforcement.

No phase in this roadmap should be read as relaxing that invariant, and Phase 10's validation strategy exists specifically to confirm it holds under the failure conditions named above, not merely under normal operation.

---

## Decision Gates

The following decisions are recorded as unresolved. For each, this section states what evidence would be needed to resolve it, rather than proposing an answer.

**Logical component and repository ownership.** Resolved by implementation-planning discovery analogous to [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 7's repository-ownership discovery, informed by expected activity and relationship-graph scale, operational cost, and kernel-boundary compatibility.

**One activity ledger versus sector-specific ledgers.** Resolved once realistic deployment scale and sector-specific retention or compliance requirements are known.

**Event-family boundaries.** Resolved by Phase 2's implementation planning, informed by which envelope categories prove separable in practice.

**Storage model.** Resolved by a future technology evaluation run against Phase 2's requirements, following the same evidence-based discipline [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 10 already establishes for authorization-technology comparison.

**Append-only versus correction/supersession semantics.** Resolved by Phase 2, informed by which correction pattern actually occurs in practice once source systems produce corrected evidence.

**Integrity and tamper-evidence model.** Resolved by a future technology evaluation informed by Phase 2's requirements and by the transport-and-integrity requirements [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) Phase 5 already defines.

**Source event retention.** Resolved by Phase 2, informed by legal, compliance, and operational retention requirements gathered during implementation planning.

**Normalized event retention.** Resolved jointly with source-event retention, informed by the same evidence.

**Graph storage and ownership.** Resolved by Phase 3, informed by expected relationship-graph size and traversal complexity, mirroring the evidence [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 7 already requires for its own graph.

**Relationship between the topology graph and the ReBAC/FGA graph.** Resolved jointly with the identity/FGA roadmap, through the ADR process, informed by whether shared infrastructure would compromise the semantic distinction the Topology and relationship boundary above requires.

**Explicit versus inferred correlation.** Resolved by Phase 4, informed by measured false-correlation rates once a reference implementation exists.

**Correlation-window limits.** Resolved by Phase 4, informed by realistic OT maintenance-workflow durations.

**Handling shared accounts.** Resolved by Phase 4, informed by how frequently shared-account ambiguity actually occurs in representative deployments.

**Handling missing or contradictory evidence.** Resolved by Phase 2 and Phase 4 jointly, informed by which failure patterns actually occur once ingestion from real producers begins.

**Detection language or rule representation.** Resolved by Phase 5, informed by a future technology evaluation weighing expressiveness against determinism and explainability.

**Detection deployment and distribution.** Resolved by Phase 5, informed by consistency with the signed-policy-distribution model [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 11 already develops.

**Suppression and exception governance.** Resolved by Phase 5, informed by operational experience with false-positive rates once a reference implementation exists.

**Behavioral analytic techniques.** Resolved by Phase 7, informed by evaluation against known scenarios and against the cold-start and sparse-data constraints that phase names.

**Model or baseline versioning.** Resolved by Phase 7, informed by the same versioning discipline [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) Phase 7 already establishes for contract versioning.

**Correlation-certainty and evidence-completeness vocabulary.** Resolved jointly by Phase 4 and Phase 5, informed by which explicit-versus-inferred and complete-versus-incomplete classifications prove usable across correlation and deterministic-detection evidence without implying probabilistic evaluation.

**Behavioral confidence or uncertainty vocabulary.** Resolved by Phase 7, informed by the same versioning discipline [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) Phase 7 already establishes for contract versioning, and kept terminologically distinct from Phase 4 and Phase 5's deterministic vocabulary, per the Detection boundary above.

**Investigation case ownership.** Resolved by Phase 6, informed by whether case state belongs with `basis-activity`, `basis-console`, or another component.

**Response approval model.** Resolved by Phase 8, informed by the policy-authorization and human-approval requirements the Response boundary above already states as non-negotiable.

**Which responses may ever be automated.** Resolved, if ever, only through a dedicated future ADR and threat model, per the Response boundary above; this roadmap's default is none beyond notify and export.

**Air-gapped ingestion and response.** Resolved across Phase 2, Phase 9, and Phase 10, informed by store-and-forward testing consistent with the air-gapped requirements [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) Phase 5 already names.

**Privacy, retention, and redaction requirements.** Resolved by Phase 2 and Phase 9 jointly, informed by the redaction-tier discipline already established for operation-aware trace and audit evidence, extended to durable activity and cross-system export.

**When activity contracts are ready for `basis-schemas`.** Resolved individually, phase by phase, following the same readiness discipline ADR-0005 already established: a contract is proposed only after its architecture and a reference implementation exist, never speculatively ahead of them.

---

## Security and Abuse Cases

The following threats are named at the scope of this roadmap and are not fully resolved by any single phase above; each recurs across the phases named in parentheses and should be re-examined as each phase's architecture solidifies, consistent with how [`docs/security/threat-model.md`](../security/threat-model.md) already treats cross-cutting threats for the existing ecosystem.

Evidence tampering, event deletion, and event injection (Phase 2) are Phase 2's central concerns, extending the integrity requirements the identity-to-operation contract already names to durable storage specifically. Producer impersonation and replay (Phases 2, 4) recur wherever ingestion or correlation relies on producer identity established elsewhere. Duplicate execution and false execution reporting (Phase 2) extend the execution-lifecycle threats the identity-to-operation contract's Phase 4 already names, now to their durable representation. False correlation and identity stitching across unrelated people (Phases 3, 4) are Phase 4's central concern, with shared-account ambiguity (Phase 4) named explicitly as a category the model must represent honestly rather than resolve by assumption. Delegation-chain forgery (Phase 4) extends the delegation threats the identity/FGA roadmap's Phase 3 already names to the activity-chain setting. Graph poisoning and topology poisoning (Phase 3) are Phase 3's concern. Malicious detection rules and suppressed high-severity findings (Phase 5) are Phase 5's concern, addressed by its versioned, evidence-linked rule discipline. Alert flooding and detection denial of service (Phase 5) are named as capacity-adjacent threats against the investigation workflow itself. Behavioral-model poisoning and explanation leakage (Phase 7) are Phase 7's concern, guarded by that phase's uncertainty and versioning requirements. Cross-tenant activity leakage and sensitive asset-detail leakage (Phase 9) recur wherever export crosses a system or tenant boundary that may not share BASIS's own redaction discipline. A compromised console analyst (Phase 6) is named explicitly as a threat against investigation integrity, not only against evidence integrity. Unauthorized response, automated response causing operational harm, and a response loop or repeated containment (Phase 8) are Phase 8's central concerns, addressed by that phase's bounded, human-governed default posture. Ledger outage affecting enforcement (Phase 2, and the Distributed-Systems Concerns section above) is named as the specific failure mode the Gateway boundary and the critical invariant above exist to prevent. Downstream compromise attempting to alter kernel or gateway decisions (all phases) is named as the threat every boundary in **Architectural Invariants** above is collectively designed to make structurally impossible: no phase in this roadmap grants any downstream capability the ability to reinterpret, rewrite, or override an authorization decision or an enforcement action.

This roadmap does not attempt to fully resolve any of these threats here. It identifies where each must be addressed and notes that [`docs/security/threat-model.md`](../security/threat-model.md) will require updates as each phase's architecture solidifies, in the same way both roadmaps this document builds on already commit to for their own phases.

---

## Operator and Training-Mode Requirements

Each phase above defines its own operator view, training explanation, owning backend component, evidence, and failure state. The following matrix summarizes the pattern across phases; detailed explanations belong in the corresponding phase sections above, not here.

| Capability | Operator mode | Training mode |
| --- | --- | --- |
| Activity ledger | Searchable identity and operation history | Source versus normalized events, event and ingest time |
| Relationship graph | Related identities, producers, assets, sessions | Why a relationship exists and its provenance |
| Operation chain | Ordered activity sequence | Correlation inputs, missing events, uncertainty |
| Detection | Finding, severity, rule, evidence | Exact deterministic rule evaluation |
| Investigation | Timeline, pivots, notes, disposition | How evidence is preserved and interpreted |
| Behavioral finding | Deviation and explanation | Baseline, uncertainty, drift, version |
| Response | Proposed/approved/applied state | Approval, policy, enforcement, rollback |
| Failure | Degraded capability | Which facts remain trustworthy |

---

## Relationship to Other Roadmaps

### Identity-to-operation contract roadmap

This roadmap consumes the identity-to-operation contract and the evidence it produces. It does not define a second event meaning for identity, producer, operation, resource, decision, enforcement, execution, or evidence — every category in **Activity Record Conceptual Requirements** above is a durable accumulation of that contract's Contract Responsibility Model, not a redefinition of it. Where that roadmap's Phase 4 names the execution-lifecycle states this roadmap's activity records must be able to represent, this roadmap depends on that phase's outcome rather than restating it. Where that roadmap's own text anticipates "a future identity-activity roadmap," this document is written as the roadmap it anticipated, against the dependency it already named.

### Identity and fine-grained authorization expansion roadmap

This roadmap coordinates with the identity/FGA roadmap on multi-tenancy, delegation, workloads, session revocation, synchronized identity, relationship authorization, and policy distribution — each of those capabilities is a potential input to this roadmap's activity, correlation, or topology model, not a capability this roadmap redefines. The central distinction between the two roadmaps' relationship models is stated once, in the Topology and relationship boundary above, and applies throughout every phase here that touches relationships: activity and investigation relationships describe what was observed; authorization and ReBAC relationships describe what is permitted. The two may eventually share infrastructure, but that is a decision gate, not an assumption either roadmap makes on the other's behalf.

### Current operation-aware gateway roadmap

Current work establishes the trustworthy upstream evidence this roadmap depends on. The gateway's operation-aware audit-evidence work is foundational to Phase 2's ledger, and the readiness and conformance work surrounding it remains required before Phase 2 can be built on a stable foundation. This roadmap does not interrupt or expand the current gateway implementation sequence in any way; future activity work begins downstream, after that program's outputs stabilize, exactly as this document's Status section states.

---

## Open-Source and Commercial Boundaries

A potential open-source foundation for this roadmap's eventual capabilities includes normalized activity contracts, ingestion interfaces, deterministic detections, the investigation evidence model, exporters, reference connectors, conformance vectors, and local and air-gapped operation. Potential commercial capabilities, consistent with the open-contract invariant already established for the identity-to-operation contract and with the existing BASAuth boundary in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md), include certified connectors, enterprise support, managed storage, hosted analytics, advanced detection content, compliance packs, long-term retention, fleet management, and upgrade assurance. The open contract and deterministic evidence model remain foundational; hosted analytics are never made necessary to interpret the open evidence, and no phase of this roadmap should be read as creating a commercial dependency for basic activity, correlation, or deterministic detection capability.

This section is deliberately restrained. It exists to record the boundary between open contract and commercial layer this roadmap depends on, not to develop a business plan.

---

## Sequencing and Dependency Guidance

The expected high-level sequence is:

```text
Phase 1: deterministic authorization and trustworthy evidence
    (points to current and active roadmap work;
    not a new phase of implementation)
    ↓
Complete current basis-gateway operation-aware integration,
audit-evidence, readiness, conformance, and release-hardening program,
and the identity-to-operation contract roadmap's own completion point
    (the ecosystem's active implementation priority; no phase
    of this roadmap is currently in progress)
    ↓
Phase 2: durable normalized identity-activity ledger
    ↓
Phase 3: identity, producer, device, asset, and evidence relationship graph
    ↓
Phase 4: session and operation-chain correlation
    ↓
Phase 5: deterministic OT identity detections
    ↓
Phase 6: investigation and operator workflows
    ├──→ Phase 7: behavioral analytics
    └──→ Phase 8: bounded response
    ↓
Phase 9: integration and export ecosystem
    ↓
Phase 10: performance, retention, privacy, and failure validation
```

This sequence is provisional. It reflects the dependency structure named in each phase's Prerequisites subsection above, not a committed schedule or an exact pull-request count. Phase 7 and Phase 8 branch independently from Phase 6 rather than chaining through one another: Phase 8's prerequisites are Phase 5's deterministic detections and Phase 6's investigation workflows, not Phase 7's behavioral analytics, so bounded-response architecture may be developed without behavioral analytics existing first. A behavioral finding from Phase 7, once it exists, may contribute a recommendation or an investigation input to a Phase 8 response workflow, but a behavioral score or probabilistic finding never independently authorizes or triggers enforcement — every applied response still requires human approval, deterministic policy authorization, gateway enforcement, and evidence, exactly as the Response boundary above states. Phase 9's integration and export work could reasonably begin in parallel with Phase 6, Phase 7, or Phase 8, since export depends more on Phase 2's envelope stability than on investigation workflows, behavioral analytics, or response specifically; Phase 9 and Phase 10 remain downstream of the capabilities they consume regardless of how Phase 7 and Phase 8 are sequenced relative to each other. Any change to phase sequencing that affects a stable component boundary or an established contract should go through the ADR process described in [`GOVERNANCE.md`](../../GOVERNANCE.md), the same discipline both roadmaps this document builds on already apply to changes in architectural intent.

To be explicit about what this sequencing does and does not authorize: no phase of this roadmap is currently in progress; the `basis-gateway` operation-aware integration program remains the ecosystem's active implementation priority and is not reordered, paused, or deprioritized by this diagram; and Phases 2 through 10 stay gated on the identity-to-operation contract roadmap and the current gateway program reaching their respective completion points, exactly as stated elsewhere in this document. This roadmap does not authorize implementation of any phase.

---

## Deferred Decisions

The following decisions are intentionally left open beyond what **Decision Gates** above already states, until implementation planning begins for the relevant phase.

**Final selection of a storage, streaming, or graph technology.** Resolved by future technology evaluations conducted with the same evidence-based methodology [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 10 already establishes, run against each phase's own requirements.

**Final selection of a machine-learning platform or technique, if any.** Resolved by Phase 7's evaluation, informed by the cold-start and air-gapped constraints that phase names; this roadmap does not assume machine learning is the eventual technique.

**Final repository or repositories that would implement this roadmap's capabilities.** Not resolved by this roadmap. Any future repository name is conceptual until it is actually established, consistent with the Status stated at the top of this document.

**Exact schema-proposal timing for any contract this roadmap anticipates.** Resolved individually, phase by phase, following the same readiness discipline ADR-0005 already established.

**Which responses may ever be automated, and their governing ADR and threat model.** Resolved, if ever, independently of this roadmap, per the Response boundary above.

**Production availability or performance targets for any capability this roadmap introduces.** Resolved by Phase 10's validation work once reference implementations exist, not asserted in advance of that testing.

---

## Summary

This roadmap preserves the long-term architectural direction for extending BASIS from deterministic authorization and trustworthy evidence into durable identity activity, correlation, deterministic detection, investigation, behavioral analytics, and bounded response for OT — the roadmap [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) already anticipated and deferred, written now against the evidence and execution-lifecycle dependency that document named. It states a central dependency chain from the identity-to-operation contract through trustworthy evidence, durable normalized activity, correlation, detection, investigation, and bounded response, and it sequences its ten phases so that no later stage is designed ahead of the stage it depends on — including branching bounded response and behavioral analytics independently after investigation, rather than chaining one through the other, so bounded-response architecture does not wait on behavioral analytics that a deterministic detection and a human investigation can already justify. It defines architectural invariants that extend, without relaxing, the kernel isolation, gateway independence from downstream analytics, and evidence discipline the existing architecture already establishes, and it draws an explicit line between the activity-investigation graph this roadmap develops and the authorization/ReBAC graph the identity and fine-grained authorization expansion roadmap develops, so the two are never conflated merely because both are graph-shaped. It requires deterministic detection before behavioral analytics, keeps a behavioral finding's confidence or uncertainty terminologically distinct from a deterministic finding's rule result, and bounds automated response to a human-governed, late-sequence capability with no autonomous OT actuation authorized by this document. It names its relationship to both roadmaps it builds on explicitly, so the three do not drift into redefining the same concepts independently. And it remains explicitly provisional: the phase structure, sequencing, and open decision gates recorded here are architectural intent as understood today, subject to revision through the same ADR process that governs every other consequential decision in this repository.

---

## Related Documents

- [`ROADMAP.md`](../../ROADMAP.md) — the ecosystem's current phase structure, status, and the active `basis-gateway` operation-aware integration program this roadmap's Status section depends on
- [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) — the contract and evidence foundation this roadmap builds on; see that document's "Relationship to Existing Roadmaps" section for the dependency this roadmap fulfills
- [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) — the companion roadmap for BASIS-internal identity and authorization platform expansion; see **Relationship to Other Roadmaps** above for the boundary between the two, particularly the activity-graph/authorization-graph distinction
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction this roadmap's invariants extend
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the kernel isolation rules the Kernel boundary above must not violate
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the existing cross-repository contract inventory this roadmap's evidence dependencies trace back to
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) — the redaction-tier discipline Phase 2 and Phase 9 extend to durable activity and export
- [`docs/architecture/basis-console.md`](../architecture/basis-console.md) — the console architecture and operator/training-mode invariants this roadmap's console requirements extend
- [`docs/roadmaps/identity-provider-integration-and-learning-environment.md`](identity-provider-integration-and-learning-environment.md) — the `basis-demo`/Training-Mode identity-learning roadmap whose sanitized training trace must not become a second activity ledger, per its own **Relationship to Existing Roadmaps** section
- [`docs/security/threat-model.md`](../security/threat-model.md) — the existing threat model that phase-specific work must update as each phase's architecture solidifies
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR process this roadmap's decision gates and sequencing changes must follow
- [`docs/adr/README.md`](../adr/README.md) — when a phase's architecture work requires a new ADR
- [`docs/glossary.md`](../glossary.md) — the canonical terminology reference this roadmap's working vocabulary should graduate into as each phase stabilizes
