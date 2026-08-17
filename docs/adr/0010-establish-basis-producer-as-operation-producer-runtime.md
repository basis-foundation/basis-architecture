# ADR-0010: Establish `basis-producer` as the Operation-Producer Runtime

## Status

Accepted

## Context

[`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) named the *operation-producer runtime* as working vocabulary for the authenticated software workload that sits between `basis-adapters` normalization and `basis-gateway` trusted-producer ingestion — a logical role, not yet assigned to any repository. That document's §11 evaluated four alternatives for where the role should live, settled on "a separate logical adapter-host or producer-runtime component" as the likely direction, and explicitly held permanent repository placement open as a decision gate: "Whether that role requires a *new* repository... is not decided here." [ADR-0008](0008-producer-workload-authentication-and-admission.md) subsequently selected mutual TLS as the first normative producer-to-gateway workload-authentication profile and authorized, without implementing, a bounded operation-producer reference slice — but its own "Provisional implementation location" section was explicit that it "does not create or name a permanent `basis-producer` repository," and its "Deferred decisions" section lists "permanent producer repository placement; creation of a new repository" as unresolved. [`docs/architecture/bounded-operation-producer-reference-implementation-plan.md`](../architecture/bounded-operation-producer-reference-implementation-plan.md) (§5) then selected a *provisional* repository — the working name `basis-operation-producer-reference` — for the bounded reference slice specifically because no permanent name had been decided, and its §30 ("Permanent Repository Decision Gate") states plainly that the plan "does not name a permanent repository and does not pre-judge that review's outcome."

That deferral was deliberate and correctly scoped at the time: the producer role had not yet been proven to require a distinct, permanent component boundary, and naming a permanent repository ahead of that proof risked freezing a decision the ecosystem did not yet have evidence for. The evidence has since materially changed. `basis-gateway`'s Phase 1A mTLS-termination spike reached Outcome B (direct in-process certificate termination is not viable), and [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) — Accepted — defined the trusted NGINX ingress and certificate-handoff topology required to carry an authenticated producer identity into `basis-gateway`. `basis-gateway` Phase 1B has since merged in full: Phase 1B.1 (configuration surface) and Phase 1B.2 (certificate-identity extraction, trusted-ingress boundary) landed first, and Phase 1B.3 (exact producer admission and dual-authentication wiring, independent of the bearer authorization subject) has since merged as well, proven in CI against the trusted NGINX ingress. Producer admission at the gateway is no longer a hypothetical integration point — it is an independently authenticated, implemented workload boundary with a real, proven trust topology on the other side of it. The operation-producer runtime that will consume that boundary — the subject of this ADR — does not yet exist in any repository.

What this establishes is that the operation-producer runtime's eventual responsibilities — invoking `basis-adapters`, retaining adapter-constructed evidence material and digest, enforcing retain-before-mint ordering, minting opaque `reference_id` values, persisting reference-to-evidence bindings, assembling the final `AdapterEvidenceReference`, holding a producer mTLS client certificate and private key, independently obtaining and presenting a credential for the authorization subject, and submitting the authenticated operation-aware request to `basis-gateway` — do not belong cleanly inside any existing component. They are materially different in kind from `basis-adapters` (a normalization library that must never hold a network credential, per its own non-negotiable non-responsibilities and per the bounded plan's §5 Option 2 rejection), from `basis-gateway` (the receiving authentication/enforcement boundary, which must not also hold the client credential of the caller it authenticates, per the bounded plan's §5 Option 1 rejection), from `basis-core` (the isolated, protocol- and transport-independent kernel), from `basis-identity` (the identity/federation boundary, which has no workload-credential-holding pipeline today), from `basis-console` (the operator-facing surface, whose own simulator is explicitly preview-only), and from `basis-deploy` (packaging/deployment tooling, not a runtime-semantics owner). The bounded plan's own §5 already reached this conclusion for the *reference* implementation and selected a new, separate repository for it, provisionally named. This ADR is the follow-on decision the plan's §30 and the boundary document's §11 both name in advance: whether that already-separate repository is also the *permanent* one.

Consistent with this repository's ADR-acceptance governance convention (confirmed by the acceptance history of ADR-0007, ADR-0008, and ADR-0009: each was first merged with `Status: Proposed`, and a separate, dedicated follow-up PR later changed the status to `Accepted` after independent architecture and governance review — see [`docs/adr/README.md`](README.md#lifecycle-states) and [ADR-0006](0006-evaluation-orchestration-layer.md)'s implementation note), this ADR is submitted as `Proposed`. Merging this ADR does not itself constitute acceptance, and no implementation work — including creation of the `basis-foundation/basis-producer` repository — is authorized until a separate formal-acceptance PR records `Status: Accepted`.

## Problem

Where should the BASIS operation-producer runtime permanently live, under what name, at what visibility, and as part of which distribution — and what does answering that question now, rather than continuing to defer it, actually commit the ecosystem to?

## Decision

**`basis-producer` is established as the permanent repository and component responsible for the BASIS operation-producer runtime.**

### Permanent component name

`basis-producer`. Referred to architecturally as **BASIS Producer**, or, where precision about the logical role versus the implementation matters, **the BASIS operation-producer runtime**. Canonical description:

> The BASIS Producer is the runtime component that turns normalized adapter operations and their evidence into durably referenced, authenticated operation-aware submissions to `basis-gateway`.

`basis-producer` is not defined primarily as "the reference producer." The repository is permanent; its first implementation is deliberately bounded and reference-oriented, but boundedness is a statement about initial scope, not about the repository's disposability.

### Permanent repository

`basis-foundation/basis-producer`. Public visibility (see **Public repository decision**, below). No credentials, deployment secrets, real certificates, or private environment material belong in the public repository; synthetic test PKI and fixtures are permitted, consistent with `basis-gateway`'s own existing test-fixture conventions for producer mTLS.

### Python package

The Python import namespace `basis_producer` is reserved for the future implementation. It is not created, and no `pyproject.toml` or packaging artifact is added, by this ADR.

### Core role and responsibilities

`basis-producer` owns the operation-producer runtime boundary named, but not assigned to a repository, by [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §2 and refined by [ADR-0007](0007-adapter-evidence-construction.md) and [ADR-0008](0008-producer-workload-authentication-and-admission.md). Its long-term responsibilities include:

- orchestrating invocation of `basis-adapters` through its public normalization interface;
- accepting the `NormalizedAuthorizationRequest` / `AdapterResult` a normalization call returns;
- receiving the canonical evidence material and digest `basis-adapters`' `construct_adapter_evidence()` produces;
- durably retaining that evidence material;
- enforcing retain-before-mint ordering — evidence must be confirmed retained before a reference identifier may be minted (ADR-0008, ADR-0007);
- minting the opaque `reference_id`;
- durably persisting the `reference_id`-to-evidence binding;
- assembling the final `AdapterEvidenceReference` (`adapter_source`, `redaction_classification`, request/correlation linkage, per ADR-0007's ownership split);
- holding and using a producer mTLS client certificate and private key (ADR-0008's workload-authentication profile) — this is a real, permanent producer workload responsibility;
- obtaining, accepting, or otherwise receiving an authorization-subject credential through a governed mechanism, and presenting it independently of its own producer workload credential, never deriving one from the other (ADR-0008's "Producer vs. authorization subject" distinction). This ADR fixes the invariant — producer workload identity and authorization-subject identity/credential must remain independent — without permanently deciding *how* `basis-producer` obtains that credential (bearer token, a future delegated-credential mechanism, or another governed form); that is an implementation-phase decision, not a permanent charter commitment;
- assembling and submitting the operation-aware request to `basis-gateway`;
- handling the gateway's authorization disposition;
- stopping before protocol execution, in the bounded implementation this ADR does not expand (see **Preserve the bounded initial implementation**, below);
- failing closed on its own internal errors, consistent with the fail-closed discipline `operation-producer-and-execution-boundary.md` §5 already requires of this role.

This ADR does not claim any of the above is implemented. It assigns permanent ownership of the responsibility to a named, permanent component; implementation remains separately authorized, phase by phase (see **Initial implementation authorization**, below).

## Non-Responsibilities

`basis-producer` does not own:

- policy evaluation or authorization semantics;
- authorization-outcome aggregation;
- gateway enforcement, HTTP disposition classification, or audit-event recording;
- producer-certificate admission (the gateway derives and admits producer identity; the producer only presents its certificate);
- bearer-token verification of the authorization subject (the gateway's existing `authenticate()` dispatch owns this unchanged);
- identity federation or identity-provider integration;
- shared schema/contract definitions;
- adapter normalization semantics (owned entirely by `basis-adapters`);
- adapter evidence canonicalization semantics or digest computation (owned entirely by `basis-adapters`, per ADR-0007);
- the operator-facing UI;
- deployment packaging or configuration distribution;
- protocol execution, in the current bounded, authorization-only slice;
- execution-result evidence, which does not exist as a concept anywhere in the ecosystem today.

Ownership by component, restated without change from existing accepted architecture:

- `basis-adapters` owns protocol normalization and canonical adapter evidence construction, canonicalization, and digest computation.
- `basis-producer` owns evidence retention after construction, reference-lifecycle management, and authenticated gateway submission.
- `basis-gateway` owns producer admission, authorization-subject authentication, request composition, kernel invocation, enforcement, and audit recording.
- `basis-core` owns deterministic authorization semantics, isolated from every component above it.
- `basis-schemas` owns shared, published contracts.
- `basis-identity` owns identity/federation capability.
- `basis-console` owns the operator experience.
- `basis-deploy` owns packaging and deployment tooling.

## Dependency Direction

```text
protocol integration
    ↓
basis-adapters
    ↓
basis-producer
    ↓  HTTPS + mTLS + Bearer
basis-gateway
    ↓
basis-core
```

**`basis-producer` → `basis-adapters`.** A normal package/runtime dependency is permitted: the producer orchestrates adapters and must consume them through their existing public interface. This is not a new dependency direction — it restates, for a now-permanent repository, the same relationship the bounded plan's §5–§7 already established for the reference implementation.

**`basis-producer` → `basis-gateway`.** No Python package dependency on gateway internals. `basis-producer` is a network client of `basis-gateway`, crossing the boundary over the published HTTP/operation-aware contract, mTLS, and bearer authentication — the same boundary ADR-0008 and ADR-0009 already specify. `basis-producer` must not import `basis_gateway` internal modules.

**No reverse dependency.** `basis-adapters`, `basis-gateway`, and `basis-core` must not depend on `basis-producer`. `basis-gateway` does not import producer runtime code; `basis-core` has no knowledge of producer implementation; `basis-adapters` remains an independently usable normalization library regardless of whether `basis-producer`, an alternative producer, or no producer at all is deployed alongside it. This is a primary justification for the separate-repository decision this ADR makes permanent: a same-repository placement inside either `basis-adapters` or `basis-gateway` would have made this direction harder to hold, not easier (bounded plan §5, Options 1 and 2).

## Distribution Membership

**`basis-producer` is a component of the BASIS Core Services Distribution.** The Foundation maintains it as part of the open-source distribution because it owns a first-class responsibility — durable evidence/reference lifecycle and authenticated gateway submission — required to complete the adapter-to-authorization lifecycle end to end.

This does not mean every BASIS deployment must run `basis-producer`. Deployment topology remains composable: a deployment may run `basis-producer`, a different conforming producer implementation, or a future integration that speaks the same governed gateway contract, subject to whatever compatibility requirements that contract carries. `basis-producer` is the Foundation-maintained implementation of the operation-producer role, not a claim that it is the only possible one. This mirrors how `basis-ecosystem.md` already describes the other distribution components: membership in the distribution is about Foundation stewardship and availability, not about mandatory deployment.

## Public Repository Decision

The permanent implementation repository, `basis-foundation/basis-producer`, is intended to be **public**. Rationale:

- BASIS is an open-source authorization ecosystem, and the producer is the component that turns normalized operations into authenticated authorization submissions — exactly the kind of security-relevant behavior the rest of the open-source distribution is already reviewable end to end.
- Producer evidence/reference handling and producer credential use are part of the trust model ADR-0008 and ADR-0009 already establish; keeping the code that implements them private would make that trust model unreviewable in exactly the place it matters most.
- The complete open-source authorization path — adapter normalization through gateway enforcement — should be inspectable without requiring access to a private repository.

**Public repository visibility does not mean production maturity.** The first `basis-producer` implementation remains bounded, reference-oriented, pre-release, and explicitly non-executing (see below). Visibility and maturity are separate concerns; this ADR does not declare a release version and does not define release policy beyond stating that the repository follows normal BASIS versioning/release discipline once implementation reaches a releasable state.

## Preserve the Bounded Initial Implementation

This ADR is a repository/component-boundary decision. It does not expand, and does not authorize expanding, the bounded producer slice ADR-0008 already authorized:

```text
local REST ProtocolOperation
    → basis-adapters normalization
    → canonical evidence + SHA-256 digest
    → durable evidence retention
    → opaque reference_id
    → durable reference binding
    → AdapterEvidenceReference
    → producer mTLS
      + separate bearer authorization subject
    → basis-gateway
    → basis-core
    → authorization disposition
    → STOP
```

No protocol execution — REST, BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, or Niagara — is authorized by this decision, and no generic executor architecture is authorized by it. Execution remains a later, separately governed architectural boundary, unaffected by where the producer runtime's permanent repository lives.

## Naming Alternatives Considered

**Keep `basis-operation-producer-reference` permanently.** Rejected. The `-reference` suffix incorrectly signals that a permanent security/runtime boundary — one that will eventually hold a private key and durable evidence — is disposable. Retaining it would likely force a disruptive rename later, after the repository has accumulated history, exactly when renaming is costliest. The component's responsibility is now understood well enough (per the Context above) to name permanently.

**Put the producer runtime in `basis-adapters`.** Rejected, restating the bounded plan's §5 Option 2 finding: adapters are libraries; they own normalization and evidence-material construction; they must not become credential-holding daemons; they must not own evidence retention or reference lifecycle; they must not submit authenticated gateway requests. This is a currently released, non-negotiable non-responsibility of `basis-adapters`, not a new finding.

**Put the producer runtime in `basis-gateway`.** Rejected, restating the bounded plan's §5 Option 1 finding: the producer is a client across the gateway's own trust boundary; combining caller and receiver in one repository would obscure the authentication/admission boundary ADR-0008 and ADR-0009 establish; the gateway must not hold the producer's client credential; doing so creates undesirable internal coupling between the two sides of an intentionally external trust boundary.

**Put the producer runtime in `basis-deploy`.** Rejected. `basis-deploy` owns packaging, configuration, and deployment tooling; it does not own runtime behavior or runtime credentials, and `operation-producer-and-execution-boundary.md` §10 already states this explicitly for any future runtime role.

**`basis-agent`.** Rejected. Ambiguous with AI agents, host agents, monitoring agents, and endpoint agents — a naming collision this ecosystem's own documentation would otherwise have to disambiguate on every use.

**`basis-edge`.** Rejected. Names a deployment location, not a component responsibility; the producer runtime is not defined by where it is deployed.

**`basis-operation-producer`.** Considered and rejected in favor of the shorter `basis-producer`. The longer form is not materially clearer once "operation producer" is already established as the logical-role vocabulary in `operation-producer-and-execution-boundary.md`, and the shorter name is more consistent with this ecosystem's existing single-word component-name convention (`basis-core`, `basis-gateway`, `basis-adapters`, `basis-console`, `basis-identity`, `basis-deploy`, `basis-schemas`).

## Public Contract Assessment

This ADR does not require a `basis-schemas` change. It names and places a runtime component; it does not create, modify, or propose a new public contract. The future `basis-producer` implementation is expected to consume already-published, shared operation-aware and evidence-reference contracts (`adapter-evidence-reference`, `operation-aware-decision-request`, and related contracts already inventoried in `operation-producer-and-execution-boundary.md` §12). If implementation later discovers a missing shared contract, that finding is a separate, later architecture and schema decision — consistent with `ecosystem-contract-inventory.md`'s governing principle that a contract becomes schema-ready when "implementation proves a stable shape," not before. No `basis-schemas` file is modified by this ADR.

## Initial Implementation Authorization

Once this ADR is formally **Accepted** — not merely merged as `Proposed` — it authorizes creation of the public `basis-foundation/basis-producer` repository. It does not create that repository itself.

The first authorized implementation phase is narrowly scoped:

**Phase 2A — evidence-retention foundation.** Expected first branch: `feature/phase-2a-evidence-retention-foundation`. Scope: canonical evidence bytes plus the expected SHA-256 digest, digest-addressed durable blob storage, and verified retrieval. `reference_id` minting is intentionally deferred to Phase 2B if the implementing team preserves that split (consistent with the bounded plan's own Phase 2 sequencing, §26).

The first implementation PR must not include: `reference_id` minting (if split into Phase 2B); a gateway client; an mTLS client; bearer-credential handling; adapter orchestration; REST operation composition; or protocol execution. None of this is implemented by this architecture PR.

**Expected subsequent sequence**, restated from the bounded plan and not redesigned here:

- **Phase 2B — reference lifecycle.** Retained blob → opaque `reference_id` → durable binding → final `AdapterEvidenceReference`.
- **Phase 3 — producer gateway client.** mTLS producer credential; independent bearer authorization-subject credential; operation-aware submission; layered failure handling.
- **Phase 4 — REST adapter composition.** `ProtocolOperation` → adapter normalization → evidence creation → retention/reference → gateway request assembly.
- **Phase 5 — end-to-end bounded conformance/demo.** Adapter → producer → gateway → Core, authorization only, no execution.

These phase descriptions are not implementation specifications; they restate, without redesigning, the bounded plan's existing §26 sequence for a now-permanently-named repository.

## Alternatives Considered

See **Naming Alternatives Considered**, above, for repository-name alternatives. Two structural alternatives were also considered for this decision as a whole:

**Continue deferring permanent placement until Phase 5 completes.** This is what the bounded plan's own §30 originally called for. Rejected as still the *default* posture but no longer the *necessary* one: §30 sets a review checklist (code cohesion, dependency-graph cleanliness, credential-boundary integrity, persistence-boundary integrity, release-cadence need) that assumes the reference implementation exists to review. The Context above shows the relevant boundary question — does this responsibility belong in a separate, permanent component at all — is now answerable independent of that checklist, because ADR-0008 and ADR-0009 have already fixed the shape of the trust boundary the producer sits behind. Continuing to defer naming while every neighboring boundary is already fixed would leave the most consequential open question in the bounded plan unresolved for no remaining architectural reason, only administrative caution. This ADR accordingly resolves the *placement/naming* question now, while explicitly not resolving the *maturity* question — the bounded plan's §30 checklist remains the right tool for judging when `basis-producer` is ready for anything beyond reference status, and this ADR does not shortcut that judgment.

**Name the repository now but keep it private until Phase 5 review.** Rejected. Producer credential handling and evidence/reference lifecycle are exactly the security-relevant behavior this ecosystem's open-source posture exists to make reviewable; keeping the code private during the phase where its trust-boundary discipline is actually being built and tested would defeat that purpose more than it would protect anything, since the ADR itself, the certificate profile, and the trust topology are already public.

## Consequences

**Positive:**

- A clean, permanent credential boundary: the component that will hold a producer private key has a name, a repository, and an ownership boundary that will not need to change later.
- A clean, permanent persistence boundary: evidence retention and reference-lifecycle responsibility has a stable home, distinct from both the library that constructs evidence and the gateway that receives references.
- `basis-adapters` remains a pure, reusable normalization library, unaffected by this decision.
- `basis-gateway` remains the receiver/enforcement boundary, unaffected by this decision, with no new coupling to producer implementation.
- `basis-core` remains fully isolated.
- The evidence/reference lifecycle becomes independently testable and independently releasable once implementation warrants it.
- The complete open-source authorization path, from adapter normalization through gateway enforcement, becomes reviewable without a private repository.
- Future alternative producer implementations remain architecturally possible; `basis-producer` is the Foundation-maintained implementation, not an exclusivity claim.

**Tradeoffs:**

- One additional repository and component for the Foundation to maintain, with its own versioning and release lifecycle once it reaches releasable maturity.
- An additional deployment process once `basis-deploy` exists to package it.
- Producer credential management becomes a real operational responsibility of a named, permanent component rather than an unallocated future concern: mTLS client certificate/private-key custody is a permanent producer workload responsibility, and independently obtaining and presenting an authorization-subject credential (mechanism not fixed permanently by this ADR — see **Core role and responsibilities**, above) is a permanent producer obligation even though its concrete form may evolve.
- Additional cross-repository integration testing is required to keep `basis-producer`'s consumption of `basis-adapters` and its submission to `basis-gateway` aligned with both repositories' own compatibility commitments as they evolve independently.
- Future `basis-deploy` work must package and configure `basis-producer` once that repository exists.

These costs are accepted because the responsibility boundary is real — established by ADR-0008 and ADR-0009's own trust-topology decisions — not because adding another repository is inherently desirable. This ADR does not minimize them.

## Validation / Implementation Gate

Formal acceptance of this ADR, through this repository's ADR governance process ([`docs/adr/README.md`](README.md#lifecycle-states)), is the gate before: the public `basis-foundation/basis-producer` repository is created; any Phase 2A implementation work begins; and any `basis-producer`-related `basis-schemas` proposal is considered (none is anticipated — see **Public Contract Assessment**, above). This ADR, once merged, does not itself constitute acceptance, consistent with this repository's established convention (see [ADR-0006](0006-evaluation-orchestration-layer.md)'s implementation note and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself change its status to `Accepted`.

## References

- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) — §2 (logical roles), §10–§11 (repository ownership and the deferred repository-placement decision gate this ADR resolves), §13 (implementation sequence)
- [ADR-0007](0007-adapter-evidence-construction.md) — adapter evidence construction ownership, preserved unchanged by this ADR
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) — producer workload authentication and admission; "Provisional implementation location" and "Deferred decisions" sections this ADR resolves the repository-placement portion of
- [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) — trusted mTLS ingress and certificate-handoff topology; the trust-boundary decision whose completion is this ADR's primary Context justification
- [`docs/architecture/bounded-operation-producer-reference-implementation-plan.md`](../architecture/bounded-operation-producer-reference-implementation-plan.md) — §5 (provisional component placement and its rejected alternatives, restated here as permanent), §30 (the permanent repository decision gate this ADR answers), §26 (the implementation phase sequence this ADR does not redesign)
- [`docs/architecture/adapter-evidence-construction-semantics.md`](../architecture/adapter-evidence-construction-semantics.md) — evidence construction and reference-assembly ownership split between `basis-adapters` and the operation-producer runtime, preserved unchanged by this ADR
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — the component-boundary and dependency-direction model this ADR adds `basis-producer` to
- [`docs/glossary.md`](../glossary.md) — terminology this ADR's naming decision is reconciled against
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR acceptance process this ADR's `Proposed` status observes
