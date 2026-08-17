# Identity Provider Integration and Identity Learning Environment Roadmap

This document defines the long-term architectural direction for connecting external identity providers into a future BASIS reference and learning environment (`basis-demo`), and for exposing — through BASIS Training Mode — how identity moves from an external identity provider through authentication, normalization, authorization, enforcement, execution, and evidence. It is an architecture-planning document, not an implementation plan. It does not implement runtime behavior, modify any implementation repository, publish new schemas, create `basis-demo` or any other repository, select an identity-provider vendor as a dependency, choose a fault-injection mechanism, choose a trace transport, or commit to a release schedule. It records long-term intent so the idea is not lost while the ecosystem's present implementation effort continues, and it defines the architectural boundaries, decision gates, and completion criteria that later implementation work — whenever it begins — must satisfy.

**Status:** Planned only. No implementation is authorized by this document, and no repository — including `basis-demo` — is created by it. This roadmap does not commit to a `basis-demo` creation date, a release schedule, or a specific identity-provider vendor as a runtime dependency of the BASIS Core Services Distribution. The ecosystem's active implementation priority remains what [`ROADMAP.md`](../../ROADMAP.md) already states it to be: trusted operation-producer identity and integration, under **Next Producer and Execution-Evidence Boundary**. [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) (producer workload authentication via mTLS) is recorded `Accepted`, and `basis-gateway`'s Phase 1B (1B.1, 1B.2, and 1B.3) has since merged in full — gateway-side producer mTLS authentication and admission is implemented for its bounded, approved scope. The operation-producer runtime itself — the component [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md), Accepted, permanently names `basis-producer` — has not yet begun implementation; the repository does not yet exist. Nothing in this document reorders, pauses, deprioritizes, or competes with that work, with the current `basis-gateway` operation-aware rollout, or with any other roadmap's stated active priority. This roadmap's implementation program, taken as a whole, remains downstream of that work — but not every phase depends on it equally. Architecture-discovery work (Phase 1) and provider-neutral lab architecture (Phase 2 through Phase 4) may be reasoned about, and an authentication-only Keycloak learning scenario (Phase 5) may be architected, independently of it: `basis-identity` `v0.1.0` already provides real, released OIDC, JWKS, session, and BASIS-local token capability that scenario can be grounded in. Only the full identity-to-operation learning flow (Phase 6 onward) is directly gated on the producer and execution-evidence work completing. None of this authorizes `basis-demo` implementation to begin, and none of it moves `basis-demo` ahead of the downstream rollout sequence `ROADMAP.md` already establishes; it only clarifies which parts of this roadmap's own architecture are, and are not, technically blocked on work still in progress elsewhere. Conceptual component and repository names used in this document — principally `basis-demo` — are conceptual references to a future architectural role. `basis-demo` is not a new name invented by this roadmap: [`ROADMAP.md`](../../ROADMAP.md)'s own Downstream Rollout Sequence already names "`basis-demo` end-to-end validation" as a future stage, positioned after `basis-deploy` packaging and before real OT integration validation. This roadmap elaborates the architectural shape of that already-named future stage specifically for identity-provider integration and identity-systems learning; it does not move that stage earlier in the sequence, and it does not create a second, competing demo concept.

A note on terminology: this roadmap introduces working vocabulary — identity learning environment, IdP profile, sanitized training trace, broken lab, controlled failure injection — that does not yet have entries in [`docs/glossary.md`](../glossary.md). Consistent with the glossary's role as a canonical reference for decided terminology rather than speculative vocabulary, formal glossary entries for these terms should be added only as each phase's architecture stabilizes through its own ADR, not as part of this roadmap. No term introduced here is promoted to canonical status by its appearance in this document.

---

## Purpose

BASIS today answers a bounded, deterministic question: given a normalized subject, action, resource, and operation-aware context, does policy permit the request, and is that decision auditable? Four existing roadmaps already chart how that bounded answer grows — deeper identity-platform capability, cross-system contract interoperability, durable activity and detection, and a mature Operator/Training console experience. None of them, individually or together, answers a different and narrower question this document exists to answer: once those capabilities exist, in whole or in part, how does a person actually *see* an external identity provider's protocol behavior connect to BASIS's authorization, enforcement, execution, and evidence, in a reproducible, safe, inspectable form?

That question matters for two converging reasons. First, BASIS needs a reproducible, end-to-end demonstration environment — the `basis-demo` stage [`ROADMAP.md`](../../ROADMAP.md) already anticipates — so that the ecosystem's components can be exercised together against a real external identity provider rather than only against unit and integration fixtures scoped to one repository at a time. Second, BASIS's Training Mode doctrine already commits the console to teaching real system behavior, never an alternate implementation of it; an identity-provider integration is one of the richest, most standards-grounded subjects Training Mode could teach, precisely because OAuth 2.0 and OpenID Connect are widely deployed, well-specified, and routinely misunderstood even by experienced engineers.

The central idea this roadmap develops:

> `basis-demo` creates realistic identity scenarios. BASIS Training Mode lets the user dissect them.

`basis-demo` is the reproducible environment that makes a real OIDC flow, a real token, a real signature, and a real authorization decision available to inspect. Training Mode is the safe, sanitized lens the console already provides for explaining real system behavior. Neither exists today. This roadmap defines how they could eventually fit together without either one becoming something the rest of the architecture does not already permit it to be.

---

## Core Architectural Principles

Three principles govern every phase below, restated from the doctrine already established elsewhere in this repository and applied specifically to identity-provider integration and learning:

> BASIS remains an authorization ecosystem first. `basis-demo` and Training Mode turn the running ecosystem into an observable identity-systems laboratory.

> Learn the identity standard first and the vendor implementation second.

> Training explains real system behavior. It never implements an alternate security path.

The third principle is a direct restatement of the Training Mode invariant already established in [`docs/architecture/basis-console.md`](../architecture/basis-console.md) ("Operator mode may simplify. Training mode may educate. Neither mode may change system behavior.") and in [`docs/roadmaps/operator-and-training-experience.md`](operator-and-training-experience.md)'s Mode Divergence Rules. This roadmap does not weaken it; it applies it to a new subject.

---

## Architectural Invariants

The following invariants are preconditions for every phase in this roadmap. They restate and extend boundaries already established in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md), [`docs/architecture/basis-gateway.md`](../architecture/basis-gateway.md), [`docs/architecture/basis-console.md`](../architecture/basis-console.md), [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md), [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [`docs/security/threat-model.md`](../security/threat-model.md), and all four roadmaps referenced under **Relationship to Existing Roadmaps** below. Nothing in this section supersedes those documents; where language differs, this section is additive. A phase whose implementation would require relaxing one of these invariants should be treated as evidence that the phase has been scoped to the wrong component, not as grounds for an exception.

**External identity provider invariant.** An external IdP integrated for learning purposes (Keycloak, Auth0, or another provider) remains exactly what [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md) already says an upstream IdP is: authoritative for accounts, credentials, MFA, and its own protocol behavior. `basis-demo` does not reimplement IdP functionality to avoid depending on a real one, and this roadmap does not weaken `basis-identity`'s existing non-responsibility for enterprise directory authority.

**`basis-demo` invariant.** `basis-demo` may own reproducible lab topology, IdP profile selection, scenario fixtures, sample (non-production) identities and workloads, lab startup and reset, controlled failure injection bounded to the lab, and reproducibility metadata. It must never own authorization semantics, token validation semantics, gateway enforcement, policy evaluation, or an alternate implementation of any identity-provider or BASIS-component behavior. This restates, for a demo/lab environment specifically, the same discipline the identity-to-operation roadmap's Adapter invariant and Execution invariant already apply to adapters and producers: a component that stands near the authorization boundary is trusted to set up and observe, never to decide.

**`basis-identity` invariant.** Unchanged from [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md): `basis-identity` integrates, brokers, normalizes, sessions, and (when configured) tokenizes identity; it does not evaluate authorization, does not become the relationship or activity store, and does not acquire demo-specific or Training-Mode-specific behavior. Where this roadmap's labs exercise `basis-identity` capability, they exercise the real capability — they do not fork a parallel identity-normalization path for learning purposes.

**`basis-gateway` invariant.** Unchanged from [`docs/architecture/basis-gateway.md`](../architecture/basis-gateway.md): the gateway authenticates, composes, invokes, enforces, and emits evidence. A lab exercises the real gateway boundary — including, once it exists, real producer authentication per ADR-0008 — never a bypass of it.

**`basis-core` invariant.** Unchanged from [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md): the kernel remains deterministic, protocol-neutral, and identity-provider-agnostic. It must never acquire OAuth/OIDC semantics, demo-specific behavior, or Training-Mode-specific behavior. Where a lab needs to explain a kernel decision, it explains the real decision the kernel already produced.

**`basis-adapters` / producer invariant.** Restated from [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md): adapters normalize and preserve evidence; they do not authorize. Where this roadmap's later phases exercise a producer runtime, they exercise it under the same trust-establishment rules that document and ADR-0008 already define — a lab does not invent a lighter-weight producer-trust model merely because it is a lab.

**`basis-console` invariant.** Restated from [`docs/architecture/basis-console.md`](../architecture/basis-console.md) and the Training-Mode constitutional requirement already established in three existing roadmaps: Training Mode may render deep explanations, timelines, comparisons, and lab guidance; it must never become an alternate PDP, PEP, identity engine, or evidence authority. A sanitized training trace is a projection of real evidence, never a second evidence model.

**AI invariant.** Where AI assists a future lab — generating explanations, suggesting a next investigation step, summarizing a broken-lab scenario — the governing invariant is unchanged from every other roadmap in this repository:

```text
AI suggests.
Humans approve.
Gateway enforces.
Core evaluates.
```

AI must never select a broken-lab fault to inject, decide a lab's pass/fail outcome, or bypass deterministic BASIS authorization to make a demonstration flow more smoothly.

**Production-independence invariant.** Production BASIS operation must not depend on `basis-demo`, on any lab fixture, or on Training Mode. This is stated in full under **`basis-demo` Must Not Become a Product Runtime Dependency** below.

---

## Training-Mode Constitutional Requirement

Consistent with the constitutional requirement already established in [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) and restated in the two roadmaps that build on it, every externally observable capability this roadmap introduces must have a corresponding operator-facing representation and training-mode explanation in `basis-console` from the outset of its design, not as work added after a capability ships. Training Mode may explain identity-provider protocol mechanics, token structure, signature and key selection, subject normalization, producer authentication and admission, operation normalization, policy evaluation, enforcement, execution, and evidence — always as an explanation of what a real request actually did, never as a simulated alternative to it. Training Mode must not change backend behavior, bypass authentication or authorization, use alternate business logic, fabricate live data, expose secrets or unredacted credentials, or become a separate application. Each phase below states what operator mode needs to show, what training mode needs to explain, which component owns the behavior being explained, what evidence proves it occurred, what failure looks like, and what must be redacted.

---

## Relationship to Existing Roadmaps

This roadmap does not redefine identity federation architecture, token exchange semantics, workload identity, session lifecycle, SCIM synchronization, relationship authorization, identity-to-operation semantics, producer identity, authorization semantics, execution semantics, durable identity-activity storage, correlation architecture, or Operator/Training doctrine. Each of those capabilities already belongs to an existing roadmap or architecture document. This roadmap instead answers a question none of them ask: how do existing and future BASIS identity, gateway, authorization, producer, execution, evidence, and Training Mode capabilities assemble into a reproducible external-IdP integration and identity-learning environment?

### Identity and Fine-Grained Authorization Expansion Roadmap

[`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) owns deeper BASIS identity-platform functionality: multi-tenant trust isolation, token exchange and delegation, workload and non-human identity, distributed session revocation, SCIM-synchronized registries, relationship-based authorization, fine-grained authorization queries, and signed policy/configuration distribution. This roadmap consumes those capabilities when and if they become available; it does not redefine them merely to make a lab easier to build. A workload-identity lab (this roadmap's Phase 9) depends on that roadmap's Phase 4; a token-exchange lab depends on its Phase 3; a SCIM lab depends on its Phase 6. None of those phases has begun implementation, per that roadmap's own Status section, and this roadmap does not claim otherwise merely because a lab scenario would be pedagogically useful.

### Identity-to-Operation Contract and Interoperability Roadmap

[`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) provides the authoritative point-in-time semantics connecting subject, authority, delegation, producer, operation, resource, authorization, enforcement, execution, and evidence — the Contract Responsibility Model this roadmap's end-to-end learning flow (Phase 6) visualizes and exercises rather than redefines. The nearer, bounded application of that roadmap's Phase 2 and Phase 4 — [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) and [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) — is this roadmap's single most load-bearing prerequisite: this roadmap's later phases cannot exercise a real producer-to-gateway trust boundary, a real execution attempt, or real execution evidence until that work — now `Accepted` — is implemented. This roadmap's training traces, once they exist, follow the same identity-to-operation contract and evidence chain that roadmap defines; they do not introduce a competing representation.

### Post-Authentication Identity Activity, Correlation, and Detection Roadmap

[`docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`](post-authentication-identity-activity-correlation-and-detection.md) owns durable identity history, activity normalization, cross-event correlation, relationship graphs, deterministic detection, investigation, and bounded response. This roadmap must not create a second activity ledger simply because Training Mode wants a timeline. Early training traces this roadmap's Phase 4 defines are ephemeral or derived directly from authoritative runtime evidence (kernel `AuditEvidence`, `GatewayAuditEvent`, and, once it exists, execution evidence); they are not a durable store. Later, if and when the durable, normalized activity ledger that roadmap's Phase 2 defines exists, this roadmap's learning environment may consume it as a read-only source rather than maintaining a parallel one. This distinction is stated once, plainly, and applies throughout every phase below that touches a trace or timeline:

> Training observability is not a competing audit or identity-activity system.

### Operator and Training Experience Doctrine and Roadmap

[`docs/roadmaps/operator-and-training-experience.md`](operator-and-training-experience.md) owns Training Mode doctrine itself: mode divergence rules, playbook structure, the progression from guided learning to independent operation, Operator/Training parity, and the future playbook framework (that roadmap's Stage 4). This roadmap does not redefine Training Mode. It supplies a concrete class of identity-focused scenarios, traces, lab inputs, and observability requirements that a future playbook framework — once built — can consume. This roadmap's Phase 10 depends explicitly on that roadmap's Stage 4 existing before any identity playbook can be authored in the format that roadmap defines.

---

## Repository Responsibility Model

The exact ownership boundaries below follow directly from the architecture documents already cited; this roadmap does not relocate any existing component responsibility to make a demo easier to build.

**External identity provider.** Owns the real upstream identity behavior it already owns today: user authentication, authorization-endpoint behavior, token issuance, discovery metadata, JWKS, upstream sessions, and provider-specific federation behavior. `basis-demo` treats this as a real, external dependency for the lab — not something to reimplement.

**`basis-demo`.** May eventually own: reproducible demo topology; IdP profile selection; demo client/application configuration; sample identities and workloads (never production credentials); sample policies and scenario fixtures; lab startup and reset; controlled failure injection bounded to the lab; expected lab outcomes; local configuration generation; and reproducibility metadata. It must never own authorization semantics, token-validation semantics, identity-provider functionality invented merely to avoid depending on a real IdP, gateway enforcement, policy evaluation, or an alternate, training-only implementation of any security behavior a real component already owns.

**`basis-identity`.** Continues to own federation, login/logout/callback flows, BASIS-local sessions, claim mapping, subject resolution, canonical identity context, identity diagnostics, and BASIS-local token issuance, exactly as [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md) already states. Where it owns behavior this roadmap's labs need to explain, it should eventually expose safe observability for that behavior (Phase 4); its authority is not expanded solely because Training Mode wants to explain more of it.

**`basis-gateway`.** Continues to own whatever the current architecture assigns to caller authentication, request composition, kernel invocation, enforcement, and gateway evidence — including, once ADR-0008's now-accepted mTLS profile is implemented, producer authentication and admission. Training instrumentation describes those real actions; it does not reimplement them.

**`basis-core`.** Remains the deterministic authorization kernel. This roadmap introduces no IdP awareness, OAuth/OIDC behavior, demo-specific behavior, or Training-Mode-specific behavior into it. Where existing evaluation evidence (`OperationAwareDecisionResponse`, `EvaluationTrace`, `AuditEvidence`) is sufficient to explain a decision, Training Mode consumes that evidence rather than adding educational logic to the kernel.

**`basis-adapters` / trusted producers.** Expose or contribute the existing trustworthy evidence necessary to explain normalized operations, producer identity, producer admission, execution attempts, and execution results, once those capabilities exist per the operation-producer-and-execution-boundary work. This roadmap does not expand `basis-adapters`' non-network, non-authenticating contract to accommodate a lab.

**`basis-console`.** Remains the human-facing presentation layer. Training Mode may render deep explanations, timelines, comparisons, and lab guidance, but must never become an alternate PDP, PEP, identity engine, or evidence authority, consistent with the Console invariant restated above.

---

## Provider-Neutral IdP Profile Model

A future IdP profile concept lets a lab describe a scenario abstractly — protocol, grant/flow, client type, expected issuer, expected audience/resource, scopes, claims required by the scenario, redirect behavior, subject-mapping expectations, and expected BASIS result — without baking a specific vendor's configuration into the scenario definition itself. A provider profile then maps that abstract scenario onto vendor-specific configuration for a given IdP. This is a decision gate this roadmap names but does not resolve (see **Decision Gates** below): the scenario-descriptor format and the profile-mapping format are both left open, deferred to Phase 3.

The intended learning hierarchy, restated as the governing pedagogical principle for every lab this roadmap eventually produces:

```text
Concept
    ↓
standard/protocol
    ↓
trust/security implication
    ↓
BASIS behavior
    ↓
vendor-specific representation
```

not:

```text
Vendor dashboard
    ↓
memorized configuration
```

A lab that only teaches a Keycloak realm's specific screen layout has failed this roadmap's purpose even if it is technically accurate. A lab that teaches why a redirect URI mismatch matters, and only then shows what that looks like in Keycloak's admin console, has succeeded.

Initial profiles this roadmap anticipates, in the same provider-neutral spirit, are Keycloak and Auth0, described in the next two sections.

---

## Keycloak Reference Profile

Keycloak is the likely default deep-learning profile because it offers an open-source IdP, reproducible local deployment, controlled identities, controlled clients, controlled signing keys, controlled claims, the ability to deliberately misconfigure a scenario for a broken lab, offline and local workshop suitability, and CI/reference-testing suitability where appropriate. This makes it valuable as a baseline reference environment without making BASIS dependent on Keycloak — the same non-dependency relationship [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md) already establishes between `basis-identity` and every upstream IdP it brokers.

A future environment may conceptually contain:

```text
Keycloak
    ↓
demo client/application
    ↓
BASIS identity/gateway boundary
    ↓
basis-core
    ↓
enforcement/execution path
    ↓
evidence
    ↓
Training Mode
```

This document does not specify a Docker Compose topology, a container image, or any other deployment mechanism beyond this illustrative reference; whether and how such detail belongs in a future `basis-demo` repository is a decision left to that repository's own implementation planning once it exists, consistent with [`docs/architecture/reference-vs-implementation.md`](../architecture/reference-vs-implementation.md)'s distinction between conceptual architecture and implementation.

---

## Auth0 Comparison Profile

Auth0 is an optional, externally managed IdP profile whose role is to demonstrate that the same underlying standards remain recognizable across a different configuration model, that BASIS is not Keycloak-specific, and that managed CIAM/IAM platforms expose the same protocol concepts through different administrative surfaces. The Auth0 profile must never be required for core development, local learning, CI, offline demos, or the open-source baseline experience; any paid or trial-only Auth0 feature may remain optional and is never required for roadmap completion. This restates, for a specific vendor, the open-contract discipline already established in [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md)'s Open-Source and Commercial Strategy section: the open, provider-neutral foundation remains foundational regardless of which managed providers eventually build comparison profiles on top of it.

---

## Identity Flow Learning

A flagship initial human-flow scenario for this roadmap is **OIDC Authorization Code + PKCE**, because it is the flow `basis-identity` already implements the most of today. Conceptually, a future learning trace should eventually be able to explain each of the following steps. This roadmap does not claim every step is already implemented; the table in **Required Dependency / Lab Availability Matrix** below states, for each step, what already exists and what remains gated.

```text
 1. protected application access
 2. authorization request construction
 3. browser redirect
 4. IdP authentication
 5. redirect with authorization code
 6. state validation
 7. authorization-code exchange
 8. PKCE validation
 9. token issuance
10. resource/API request
11. token verification
12. issuer validation
13. audience validation
14. lifetime validation
15. external identity normalization
16. BASIS subject construction
17. producer/workload authentication where applicable
18. producer admission
19. operation normalization
20. authorization evaluation
21. enforcement
22. execution outcome
23. evidence retention/correlation
```

Steps 1 through 16 sit substantially within `basis-identity`'s existing, released `v0.1.0` scope — OIDC discovery, JWKS retrieval, ID-token verification, authorization-request construction with PKCE, callback validation, session establishment, and BASIS-local token issuance and signing are implemented and tested for that release, per [`ROADMAP.md`](../../ROADMAP.md). What is not yet true, and what this roadmap does not claim is true, is that this chain is integrated against the operation-aware surface — `basis-identity`'s evidence alignment against operation-aware evaluation is explicitly named in [`ROADMAP.md`](../../ROADMAP.md) as "Planned, not yet started." Steps 17 through 23 depend entirely on work that is, at best, `Accepted` and unimplemented: producer authentication and admission depend on ADR-0008 (`Accepted`, not yet implemented); operation normalization through a producer runtime depends on [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md)'s unimplemented Stages 4 and 5; execution outcome and evidence depend on a schema and runtime gap that document's §12 records as open. This roadmap's Phase 5 and Phase 6 exist specifically to separate these two halves of the trace rather than presenting the flow as a single, uniformly available capability.

---

## Required Identity Learning Topics

This roadmap should preserve a path toward teaching, over time and across labs: OAuth 2.0 versus OpenID Connect; authorization servers, identity providers, relying parties/clients, and resource servers; authorization requests, redirect URIs, `state`, `nonce`, and PKCE; front-channel versus back-channel communication; authorization-code exchange; access tokens versus ID tokens; OIDC discovery and JWKS; JWT structure, `alg`, `kid`, and signature verification; issuer, audience, and lifetime validation; scopes and claims; subject normalization and identity provenance; authentication versus authorization; policy decision versus enforcement; execution versus authorization; authorization subject versus producer identity; workload identity, mTLS, certificate chains, and URI SAN identity; producer admission; and evidence provenance and correlation. Advanced subjects — workload identity beyond the mTLS producer profile, token exchange, delegation, relationship authorization — remain gated on the existing roadmap that owns the underlying capability, per **Relationship to Existing Roadmaps** above.

---

## Training Depth

Coordinated with, and subordinate to, [`docs/roadmaps/operator-and-training-experience.md`](operator-and-training-experience.md)'s own doctrine, a future identity lab may support progressive conceptual depth without creating separate backend behavior for each level — these are different projections of the same underlying truth, exactly as that roadmap's Mode Divergence Rules already require.

**Simple.** Who authenticated; what was requested; whether authorization succeeded; whether execution occurred.

**Engineer.** Selected normalized claims; identity mapping; producer identity; operation; policy; decision; enforcement; evidence.

**Deep Dive.** Sanitized protocol and security detail: redirect parameters; discovery; JWKS; JWT header and claims; `kid`; the signature-verification path; issuer/audience/time validation; PKCE mechanics; certificate chain and SAN identity; canonicalization, digest, and provenance where applicable; and detailed correlation.

---

## Sanitized Training Observability

This is one of the most important sections of this document. The governing principle:

> Components expose purpose-built, sanitized training observability for the behavior they already own.

Whether this eventually takes the form of existing evidence projected for training, ephemeral correlated training events, a sanitized trace envelope, later consumption of the durable identity-activity ledger the post-authentication roadmap defines, or some combination, is an architecture decision this roadmap does not prematurely make (see **Decision Gates** below). What this roadmap does fix, as an invariant rather than a decision gate, is that any future training trace must reference or derive from authoritative runtime facts — `basis-identity` session and diagnostic state, `basis-gateway`'s `GatewayAuditEvent`, `basis-core`'s `AuditEvidence` and `EvaluationTrace`, and, once they exist, producer-admission and execution-evidence facts — and that this roadmap does not create a second canonical evidence model to power it. This restates, for identity-provider learning specifically, the same discipline [`docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`](post-authentication-identity-activity-correlation-and-detection.md)'s Activity boundary already states: an observability record is a durable (or ephemeral) representation of what evidence said occurred, never an independent claim about what was permitted or what actually happened beyond what the evidence supports.

---

## Sensitive Artifact Rules

Redaction discipline for a sanitized training trace must be at least as strict as the redaction-tier discipline already established for operation-aware trace and audit evidence in [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md), extended here to identity-provider protocol artifacts specifically.

Normally safe to expose in sanitized form: `client_id`; redirect URI; requested scope; whether `state` was present and validated; whether `nonce` was present and validated; the PKCE method and code challenge where appropriate; token header fields, `alg`, and `kid`; issuer and audience; token timestamps; selected non-sensitive claims; certificate subject and SAN; trust result; policy bundle and version; authorization outcome; enforcement outcome; and correlation identifiers.

Never exposed by default, because each is a reusable, replayable, or otherwise directly exploitable secret rather than a fact about how the system behaved: passwords; client secrets; private keys; session cookies; refresh tokens; reusable access tokens; raw bearer credentials; an authorization code while it remains usable; PKCE verifier material; and any other replayable credential.

A deliberately isolated local training environment may eventually support carefully controlled deeper inspection — for example, an instructor-only mode that briefly reveals otherwise-redacted material inside an air-gapped classroom deployment — but that requires an explicit, later architecture and security decision this roadmap does not make, and production observability must never be weakened to satisfy a demo or a training scenario. This restates, for identity artifacts specifically, the Sensitive-data-leakage cross-phase concern the identity/FGA expansion roadmap and identity-to-operation roadmap already name for their own evidence categories.

---

## Broken Labs and Controlled Failure Injection

Intentionally broken identity scenarios are a first-class learning mechanism this roadmap preserves, not an afterthought bolted onto a working lab. Representative examples: wrong audience; expired token; unknown signing key; issuer mismatch; redirect URI mismatch; `state` mismatch; PKCE failure; key rotation; a missing producer certificate; an invalid producer SAN; an unadmitted producer; authentication succeeding while authorization returns `DENY`; authorization succeeding while execution fails; and a policy-version change mid-scenario.

For every broken lab, a future implementation must eventually define: a controlled starting state; the expected failure; the component that owns that failure; the authoritative evidence proving it occurred; the training explanation of it; the redaction requirements that apply; a reset/recovery procedure; and, where practical, a deterministic expected result. Fault injection must be bounded to the explicit lab environment this roadmap's Phase 2 defines, and the architecture must prevent demo or fault-injection controls from becoming a production security bypass — this is the specific reason **`basis-demo` invariant** above forbids `basis-demo` from owning authorization or enforcement semantics: a component that can inject faults must never also be the component that decides whether a request is allowed.

---

## Authentication Success Must Not Imply Authorization

This is a flagship learning invariant this roadmap exists partly to teach, and it is not a pedagogical simplification — it is a direct restatement of architecture already established elsewhere in this repository, some of it implemented and some of it still proposed. [`docs/architecture/basis-gateway.md`](../architecture/basis-gateway.md) already states, as implemented and released behavior, that a `NOT_APPLICABLE` kernel outcome is treated as `DENY` at the gateway, and that the gateway must never add supplementary allow logic on top of a kernel decision. The accepted, unimplemented [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) states the same distinction explicitly for producer trust: successful mTLS authentication and admission must not, and does not, imply that "authorization will return `ALLOW`," that "an allowed operation was executed," or that "a target device accepted or completed an operation."

A future lab should be able to show a sequence such as:

```text
OIDC authentication:      success
JWT validation:           success
Subject mapping:          success
Producer authentication:  success
Producer admission:       success
Authorization:            deny
```

and teach the invariant this sequence demonstrates:

> Authentication establishes identity. Authorization determines whether that identity has authority for the requested operation.

A closely related invariant, equally central and equally already established in [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md)'s lifecycle rules:

> Authorization `ALLOW` does not prove execution occurred.

Both distinctions must appear prominently in every future lab this roadmap's Phase 5 through Phase 7 produce.

---

## Human Identity Versus Producer Identity

A second flagship learning concept, equally well grounded in architecture already established in this repository rather than invented for pedagogy. [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §2's distinction between the *operation initiator* and the *operation-producer runtime* already establishes that a single operation may involve both the authorization subject on whose authority an action is requested, and the authenticated producer workload that actually submits the operation — and that these are never required or assumed to be the same underlying identity. The accepted, unimplemented [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) restates this distinction narrowly, in its own **Producer vs. authorization subject** section, specifically for its mTLS producer-authentication mechanism.

A future lab should eventually show both without collapsing them, for example:

```text
Authorization subject:  operator:alice
Producer identity:      spiffe://example.org/basis/rest-adapter-01
```

and demonstrate, concretely, that a valid human credential does not automatically authenticate or admit the producer, and that an authenticated, admitted producer does not automatically authorize the human subject whose operation it carries. This must remain aligned with whatever the identity/FGA roadmap's Phase 3 and Phase 4 (delegation, workload identity) eventually establish, and does not anticipate or redefine that work — this roadmap's Phase 9 depends on it explicitly.

---

## Required Dependency / Lab Availability Matrix

The matrix below records, against the current repository state established during this roadmap's review, which learning capability depends on which existing or future architecture. It exists to prevent this roadmap from quietly absorbing work owned elsewhere, and to keep this roadmap honest about what is actually buildable today versus what remains gated.

| Learning capability | Dependency / owning roadmap or architecture | Current state |
| - | - | - |
| Browser-redirect OIDC authentication (steps 1–14 of the identity flow above) | `basis-identity` federation, login/callback, JWKS, token verification | Implemented and released, `basis-identity` `v0.1.0` |
| Subject normalization and BASIS-local session/token issuance (steps 15–16) | `basis-identity` canonical identity context, BASIS-local token issuance/signing | Implemented and released, `basis-identity` `v0.1.0` |
| `basis-identity` evidence alignment against the operation-aware surface | Current gateway/identity integration work | Planned, not yet started, per `ROADMAP.md` |
| Producer authentication and admission (step 17–18) | [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) | Accepted; gateway-side implemented and merged (`basis-gateway` Phase 1B: 1B.1, 1B.2, 1B.3), bounded and approved scope |
| Operation normalization via a producer runtime (step 19) | [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §13 Stages 4–5; [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md) | Permanent component (`basis-producer`) established, `Accepted`; repository not yet created and no reference implementation exists |
| Authorization decision explanation (step 20) | `basis-core` / `basis-gateway` operation-aware evaluation | Implemented and released, `basis-core` / `basis-gateway` `v0.2.0` (gateway path feature-flagged, disabled by default) |
| Enforcement disposition explanation (step 21) | `basis-gateway` `GatewayAuditEvent` | Implemented and released, `basis-gateway` `v0.2.0` |
| Execution-result explanation (step 22) | Execution-evidence architecture, per boundary document §12 | Schema and runtime gap; not yet defined |
| Evidence correlation across the full chain (step 23) | Boundary document §6 correlation model | Partially implemented (gateway/kernel correlation); producer/execution correlation undefined |
| Token exchange and delegation labs | Identity/FGA roadmap Phase 3 | Planned, not started |
| Workload/non-human identity labs beyond the mTLS producer profile | Identity/FGA roadmap Phase 4 | Planned, not started |
| Session revocation/logout labs | Identity/FGA roadmap Phase 5 | Partial session-lifecycle groundwork exists as part of `basis-identity` `v0.1.0`'s released session surface (session store, expire/revoke/touch/refresh, cookie binding, login-callback and logout composition); distributed, multi-node revocation itself remains planned, not started |
| SCIM synchronization labs | Identity/FGA roadmap Phase 6 | Planned, not started |
| Relationship authorization labs | Identity/FGA roadmap Phase 7–9 | Planned, not started |
| Durable identity timeline / cross-operation correlation labs | Post-authentication activity roadmap Phase 2–4 | Planned, gated on identity-to-operation contract stabilizing |
| Detection labs | Post-authentication activity roadmap Phase 5 | Planned, not started |
| Playbook-framework-authored labs | Operator/Training roadmap Stage 4 | Deferred; entry criteria not yet met |

This matrix should be treated as a snapshot, not a durable contract; it should be re-verified against the current repository state before any phase below begins implementation planning, consistent with how [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) treats its own inventory tables.

---

## Roadmap Phases

The following ten phases are architecture phases, not a predetermined pull-request schedule. Each phase uses a consistent structure: purpose, primary repositories, prerequisites, architectural outcome, key capabilities, security and abuse cases, distributed-systems concerns, decision gates, completion criteria, operator-mode representation, training-mode explanation, evidence and audit requirements, schema and documentation impact, explicitly deferred work, and engineering experience developed.

### Phase 1 — Existing Capability, Roadmap, and Evidence Inventory

**Purpose.** Establish, before any lab design begins, exactly what identity, authorization, evidence, and Training Mode capability already exists across the ecosystem, and exactly where this roadmap's own work begins relative to it — the same discipline the identity-to-operation roadmap's own Phase 1 already applies at ecosystem scope, narrowed here to identity-provider integration and learning specifically.

**Primary repositories.** `basis-architecture` only; no implementation repository is modified.

**Prerequisites.** None. This phase is discovery work and can proceed independently of every other phase below, and independently of the ecosystem's active implementation priority.

**Architectural outcome.** A documented current-state chain — the same chain the **Required Dependency / Lab Availability Matrix** above summarizes — distinguishing released capability (`basis-identity` `v0.1.0`'s OIDC/JWKS/session/token surface; `basis-core`/`basis-gateway`/`basis-console` `v0.2.0`'s operation-aware surface; `basis-adapters` `v0.2.0`'s evidence-material construction) from architecture that is `Accepted` but unimplemented (ADR-0008) from architecture that is planning-only with no reference implementation (the operation-producer-and-execution-boundary document) from capability that is merely planned elsewhere (the identity/FGA and post-authentication roadmaps).

**Key capabilities.** A maintained inventory table (this roadmap's own matrix above is the seed of it); explicit callouts distinguishing "implemented and released" from "architecture accepted, not implemented" from "architecture proposed, not accepted" from "not yet architected"; and a recorded list of overlaps with the other four roadmaps so later phases build on them rather than beside them.

**Security and abuse cases.** None directly; this is a documentation phase. Its principal risk is a documentation risk — an inventory that overstates ecosystem maturity would mislead every phase that depends on it, precisely the failure mode the **Required Status and Scope Language** commitments in this document's header exist to prevent.

**Distributed-systems concerns.** None; this phase produces no runtime behavior.

**Decision gates.** None resolved by this phase.

**Completion criteria.** A reviewed inventory, consistent in method with [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md), that later phases can cite without re-deriving current ecosystem state.

**Operator-mode representation.** Not applicable; architecture-discovery phase with no operator-facing surface.

**Training-mode explanation.** Not applicable at this phase; the inventory informs later phases' training content.

**Evidence and audit requirements.** The inventory records its sources and the date each was current as of, so later readers can tell whether it has gone stale.

**Schema and documentation impact.** None.

**Explicitly deferred work.** Resolution of any open question already tracked by another roadmap or architecture document.

**Engineering experience developed.** Cross-repository architectural discovery discipline: reading what is actually implemented before proposing what a learning environment should teach.

### Phase 2 — `basis-demo` Responsibility and Trust-Boundary Model

**Purpose.** Define what a future `basis-demo` owns, what it must never own, and how it is isolated from production BASIS deployments and from every real component's own authority — before any lab content is designed.

**Primary repositories.** `basis-architecture` (this document and its eventual ADR); `basis-demo` (future, not created by this phase).

**Prerequisites.** Phase 1's inventory.

**Architectural outcome.** The **Repository Responsibility Model** above, formalized: `basis-demo` as an orchestration and reference-scenario layer that consumes the real ecosystem rather than reimplementing any part of it, with an explicit, testable boundary between lab controls (topology selection, fixture loading, reset, bounded fault injection) and every authorization-relevant decision, which always remains inside `basis-core`, `basis-gateway`, and the real IdP.

**Key capabilities.** A topology model describing which real components a lab instantiates and how; an IdP-profile selection mechanism (elaborated in Phase 3); sample-identity and sample-workload provisioning that is clearly and structurally distinguishable from production credentials; scenario-fixture loading; lab startup and reset; and reproducibility metadata sufficient to reconstruct exactly which component versions, policy bundles, and IdP configuration a given lab run used.

**Security and abuse cases.** Lab fault-injection controls leaking into a production configuration surface; demo-provisioned sample credentials being reused, copied, or mistaken for production credentials; and a `basis-demo` component acquiring, even accidentally, the ability to influence an authorization or enforcement decision, which the **`basis-demo` invariant** above exists specifically to prevent.

**Distributed-systems concerns.** Reset and teardown consistency across however many real components a lab instantiates; and fixture drift, where a lab's sample policies, identities, or scenario data fall out of sync with the real contracts and component versions they are meant to exercise.

**Decision gates.** Exact `basis-demo` repository-creation timing; scenario-definition format; and whether `basis-demo` is a single repository or a thin orchestration layer over per-component demo fixtures maintained closer to each real component. None of these are resolved by this roadmap.

**Completion criteria.** A reviewed responsibility model, ideally recorded as an ADR, naming the trust boundary between lab control and authorization decision precisely enough that a future implementer cannot accidentally cross it.

**Operator-mode representation.** Not directly applicable; a lab's real components still render their real Operator-mode surfaces unchanged.

**Training-mode explanation.** That a `basis-demo` lab is running real components against real (if sample) data, and exactly which parts of what the trainee sees are lab-provisioned fixtures versus authoritative runtime output — the same "simulated versus live" honesty discipline [`docs/roadmaps/operator-and-training-experience.md`](operator-and-training-experience.md) already requires of Training Mode generally.

**Evidence and audit requirements.** None new; `basis-demo` does not own evidence. Whatever evidence a lab's real components produce remains attributable to those components exactly as it would be in a non-lab deployment.

**Schema and documentation impact.** None; a future lab/scenario descriptor is a candidate contract only, per **Schema Discipline** below.

**Explicitly deferred work.** Any `basis-demo` implementation; repository creation.

**Engineering experience developed.** Reproducible-environment design and trust-boundary discipline applied to a demonstration layer specifically — a different problem than trust-boundary design inside a production authorization path, but governed by the same rigor.

### Phase 3 — Provider-Neutral Identity Scenario and IdP Profile Model

**Purpose.** Define provider-neutral lab semantics and the relationship between an abstract scenario and a concrete IdP profile (Keycloak, Auth0, or another provider), per the **Provider-Neutral IdP Profile Model** above, without finalizing a configuration-file format.

**Primary repositories.** `basis-architecture`; `basis-demo` (future).

**Prerequisites.** Phase 2's responsibility model.

**Architectural outcome.** A scenario description is separable from a provider profile: the scenario names protocol, grant/flow, client type, expected issuer, expected audience, scopes, required claims, redirect behavior, subject-mapping expectations, and expected BASIS result; a provider profile maps that abstract scenario onto a specific IdP's concrete configuration. The Keycloak profile and the Auth0 profile, per the sections above, are two realizations of the same scenario model, not two independently authored lab suites.

**Key capabilities.** A scenario-description vocabulary sufficient to express the flagship OIDC Authorization Code + PKCE scenario abstractly; a profile-mapping concept translating that vocabulary into Keycloak realm/client configuration and, separately, Auth0 tenant/application configuration; and an explicit non-goal — this phase does not standardize a machine-readable scenario schema.

**Security and abuse cases.** A provider profile silently narrowing or widening what a scenario actually tests (for example, a Keycloak profile that happens to disable audience validation for convenience, teaching an inaccurate lesson about what BASIS actually validates); and profile configuration drifting from the abstract scenario it claims to realize.

**Distributed-systems concerns.** None beyond what Phase 2 already names for lab topology.

**Decision gates.** Scenario-definition format; IdP-profile configuration format; and whether provider-specific extensions belong in core lab definitions or in provider profiles specifically (this roadmap's default position, restated from the **Provider-Neutral IdP Profile Model** section, is that they belong in provider profiles).

**Completion criteria.** A reviewed scenario/profile model specific enough that a future contributor could express the flagship OIDC scenario abstractly and realize it under both the Keycloak and Auth0 profiles without contradiction.

**Operator-mode representation.** Not applicable; this phase defines lab-authoring concepts, not an operator-facing surface.

**Training-mode explanation.** How the same abstract scenario is realized differently by different IdP profiles, reinforcing the learning hierarchy stated above (concept before vendor).

**Evidence and audit requirements.** None new.

**Schema and documentation impact.** A future lab/scenario descriptor and a future IdP-profile contract are both candidates only, deferred per **Schema Discipline** below.

**Explicitly deferred work.** The scenario-descriptor and profile-mapping file formats themselves.

**Engineering experience developed.** Provider-neutral abstraction design — separating what a scenario teaches from how a specific vendor happens to configure it.

### Phase 4 — Sanitized Training Trace and Cross-Component Observability Model

**Purpose.** Define how real identity and security behavior becomes safely inspectable by Training Mode, per **Sanitized Training Observability** and **Sensitive Artifact Rules** above, reconciled carefully with existing evidence contracts and the future durable activity ledger.

**Primary repositories.** `basis-architecture`; eventual implementation would touch `basis-console`, and, for identity-specific observability, `basis-identity`.

**Prerequisites.** Phase 1's inventory of what evidence already exists; coordination with [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) and [`docs/architecture/operation-aware-evidence-provenance-semantics.md`](../architecture/operation-aware-evidence-provenance-semantics.md).

**Architectural outcome.** A sanitized training trace is a projection of authoritative evidence — `basis-identity` session/diagnostic state, `GatewayAuditEvent`, `AuditEvidence`, `EvaluationTrace`, and, once they exist, producer-admission and execution-evidence facts — never a second, competing evidence model. Whether it is realized as evidence projected at render time, an ephemeral correlated training event, a sanitized trace envelope, or later consumption of the durable activity ledger the post-authentication roadmap defines, is an open decision gate this phase does not resolve (see **Decision Gates** below).

**Key capabilities.** A redaction model at least as strict as the existing redaction-tier discipline, applied to identity-provider protocol artifacts per **Sensitive Artifact Rules** above; a correlation model linking a training trace back to the real correlation identifiers [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §6 already defines, without minting a competing identifier scheme; and an explicit non-goal — this phase does not define a trace transport, broker, or storage technology.

**Security and abuse cases.** Redaction bypass, where a sanitized trace inadvertently carries a reusable credential named in **Sensitive Artifact Rules** above; and trace-correlation spoofing, where a fabricated or manipulated training event is presented as though it derived from authoritative evidence it does not actually reference.

**Distributed-systems concerns.** None beyond what Phase 2 already names for lab topology; a training trace's own consistency depends entirely on the consistency of the authoritative evidence it projects.

**Decision gates.** Sanitized training-event representation; whether training traces are ephemeral, derived, or later ledger-backed; trace transport; and trace retention. None resolved by this roadmap.

**Completion criteria.** A reviewed observability model naming, for at least the flagship OIDC scenario, which authoritative evidence sources a training trace would project from and which fields are redacted by default.

**Operator-mode representation.** Not directly applicable; Operator Mode continues to render the real evidence it already renders, unchanged by this phase.

**Training-mode explanation.** How a training trace maps back to the authoritative evidence it was derived from, so a trainee learns to trust the console's explanation precisely because it is traceable to a real record, not because it is asserted.

**Evidence and audit requirements.** Every sanitized training trace element retains a reference back to the authoritative evidence it was derived from, mirroring the same discipline [`docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`](post-authentication-identity-activity-correlation-and-detection.md) Phase 2 already requires of a normalized activity record.

**Schema and documentation impact.** A future sanitized-training-trace contract is a candidate only, deferred per **Schema Discipline** below.

**Explicitly deferred work.** Trace transport and storage technology; final redaction-field list beyond the illustrative one in **Sensitive Artifact Rules** above.

**Engineering experience developed.** Safe observability design: exposing enough of a real system's behavior to teach it without exposing anything that could be replayed or misused.

### Phase 5 — Baseline Keycloak OIDC Reference Learning Flow (Authentication Scope)

**Purpose.** Define the architectural readiness criteria for a reproducible Authorization Code + PKCE reference scenario using the Keycloak profile, scoped honestly to what `basis-identity` `v0.1.0` already implements — identity flow steps 1 through 16 — without claiming readiness for the steps that remain gated.

**Primary repositories.** `basis-architecture`; `basis-demo` and `basis-identity` (future/existing, not modified by this phase itself).

**Prerequisites.** Phases 1 through 4; `basis-identity`'s released `v0.1.0` OIDC/session/token capability, which this phase depends on rather than re-derives.

**Architectural outcome.** An authentication-only reference lab is architecturally the nearest-term buildable increment this roadmap identifies: a real Keycloak instance, a real `basis-identity` deployment exercising its already-released login/callback/session/token-issuance path, and a Training Mode explanation stopping at BASIS subject construction (step 16) — explicitly not extending into producer authentication, operation normalization, authorization evaluation, enforcement, or execution, each of which Phase 6 addresses separately because each depends on architecture that is not yet accepted or implemented.

**Key capabilities.** A Keycloak realm/client profile per Phase 3's model; a reproducible `basis-identity` deployment exercising real OIDC discovery, JWKS retrieval, authorization-code exchange, PKCE validation, ID-token verification, and BASIS-local session/token issuance; and a Training Mode explanation of steps 1 through 16 using Phase 4's sanitized trace model.

**Security and abuse cases.** Presenting an authentication-only lab as though it demonstrates full identity-to-operation behavior, which would violate this document's own accuracy commitments; and any of the redirect/state/PKCE/signature abuse cases named under **Broken Labs and Controlled Failure Injection** above, scoped here to the authentication half of the flow only.

**Distributed-systems concerns.** None beyond what Phase 2 already names for lab topology and reset.

**Decision gates.** Whether this phase's lab should be built before or independently of Phase 6's fuller flow — this roadmap's own **Sequencing and Dependency Guidance** below treats this as the more tractable near-term increment precisely because its prerequisites are already released, but does not mandate that it be built first.

**Completion criteria.** A reviewed lab design demonstrating steps 1 through 16 of the identity flow against a real Keycloak instance and a real `basis-identity` deployment, honestly scoped to stop at BASIS subject construction.

**Operator-mode representation.** Session and identity-context state for a given lab-authenticated subject, rendered through whatever real `basis-identity`/`basis-console` surfaces already exist for this purpose.

**Training-mode explanation.** Steps 1 through 16 of the identity flow, using Phase 4's sanitized trace model and Simple/Engineer/Deep-Dive depth per **Training Depth** above.

**Evidence and audit requirements.** Whatever identity diagnostics and session evidence `basis-identity` already produces, projected through Phase 4's observability model.

**Schema and documentation impact.** None beyond what Phase 3 and Phase 4 already name as candidates.

**Explicitly deferred work.** Everything from producer authentication onward (identity flow steps 17–23), addressed by Phase 6.

**Engineering experience developed.** Building a reproducible, standards-accurate OIDC reference environment scoped honestly to real, released capability.

### Phase 6 — End-to-End Identity-to-Operation Learning Flow

**Purpose.** Extend Phase 5's authentication-only trace through subject normalization, producer authentication and admission, operation normalization, policy evaluation, enforcement, execution, and evidence — identity flow steps 17 through 23 — once, and only once, the architecture those steps depend on is accepted and implemented.

**Primary repositories.** `basis-architecture`; eventual implementation would touch `basis-demo`, `basis-gateway`, `basis-identity`, and a future operation-producer runtime reference implementation.

**Prerequisites.** Phase 5; [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) — now `Accepted` — being implemented; [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §13 Stage 5's bounded producer-runtime reference implementation existing; and `basis-identity`'s evidence alignment against the operation-aware surface, per [`ROADMAP.md`](../../ROADMAP.md)'s own "Planned, not yet started" status for that work. This phase is explicitly and substantially gated — it is named here as architecture, not as a near-term buildable increment.

**Architectural outcome.** A single, real, end-to-end trace connecting an external IdP through `basis-identity`, a real authenticated and admitted operation producer, `basis-gateway`'s operation-aware evaluation path, `basis-core`'s deterministic decision, enforcement, and — once execution-evidence architecture exists per the boundary document's §12 — an execution outcome and its evidence, all visualized through Phase 4's sanitized trace model.

**Key capabilities.** Everything Phase 5 provides, extended through: real mTLS-based producer authentication and admission per ADR-0008's bounded reference slice; a real operation-producer runtime submitting a real operation-aware request; real `basis-core`/`basis-gateway` evaluation and enforcement, exactly as already released in `v0.2.0`; and, only once it exists, real execution-evidence production. This phase must not simulate any of these steps in place of the real capability — a lab that fabricates a producer-authentication result to complete the trace before the real mechanism exists violates the **`basis-demo` invariant** and the Training-Mode constitutional requirement stated above, and is explicitly rejected as an interim substitute.

**Security and abuse cases.** Every abuse case already named in [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §5 and ADR-0008's **Security consequences** section applies unchanged inside a lab context; a lab does not get a lighter-weight threat model merely because it is a lab. The specific, additional risk this phase must guard against is a lab quietly substituting a simulated step for a real one to appear complete before the real architecture exists.

**Distributed-systems concerns.** Whatever the real producer-to-gateway and gateway-to-executor correlation model eventually requires, once it exists, per boundary-document §6 — this phase inherits, rather than redefines, that model.

**Decision gates.** None new beyond what ADR-0008 and the boundary document already name as open (execution-status vocabulary, category-scoped producer capability, workload-identity establishment in `basis-identity`); this phase depends on those gates resolving elsewhere, not on resolving them itself.

**Completion criteria.** A reviewed lab design for the full 23-step trace that can only be marked ready for implementation once every prerequisite above is independently satisfied — this phase's completion criteria are therefore compound and explicitly not met by this roadmap's own acceptance.

**Operator-mode representation.** The complete, safely redacted request and evidence path for a given lab operation, mirroring the same representation [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 9 already defines for its own runtime-integration operator surface.

**Training-mode explanation.** The full 23-step trace, walked step by step, at Simple/Engineer/Deep-Dive depth, showing explicitly where authentication ends and authorization begins, and where authorization ends and execution begins, per the two flagship invariants above.

**Evidence and audit requirements.** Every step's evidence traced back to its authoritative source, per Phase 4's observability model; execution evidence, once it exists, recorded distinctly from authorization and enforcement evidence, never merged into a single record.

**Schema and documentation impact.** None from this roadmap directly; this phase consumes whatever execution-evidence and producer-authentication contracts the boundary document's own Stage 6 eventually produces.

**Explicitly deferred work.** Everything gated on ADR-0008's implementation (now `Accepted` as architecture), the bounded producer-runtime reference implementation, and `basis-identity` evidence alignment — which is to say, most of this phase's own content, honestly stated.

**Engineering experience developed.** End-to-end distributed-systems tracing across a real, multi-component authorization and execution chain — the deepest single learning outcome this roadmap can offer, precisely because it is gated on the deepest remaining architectural gap in the ecosystem.

### Phase 7 — Controlled Failure and Broken-Lab Framework

**Purpose.** Define bounded failure injection, expected outcomes, reset behavior, redaction, and deterministic validation for the broken-lab scenarios named under **Broken Labs and Controlled Failure Injection** above.

**Primary repositories.** `basis-architecture`; `basis-demo` (future).

**Prerequisites.** Phase 5 (and, for producer/execution-scoped broken labs, Phase 6) providing a working baseline scenario to deliberately break.

**Architectural outcome.** A framework in which every broken lab is defined against a controlled starting state, injects exactly one bounded fault, and produces a deterministic, evidence-backed, explainable failure — never an ambiguous or silently-swallowed one — with fault injection strictly confined to the lab environment per the **`basis-demo` invariant** above.

**Key capabilities.** A fault taxonomy covering at least the examples named under **Broken Labs and Controlled Failure Injection**; a reset/recovery procedure for every broken-lab scenario; and a mechanism, not yet selected, for injecting a fault (a misconfigured redirect URI, an expired token, a revoked certificate) without that mechanism becoming reachable from outside the lab boundary.

**Security and abuse cases.** A fault-injection mechanism that is insufficiently isolated, becoming a production security bypass — the central risk this section and the **`basis-demo` invariant** exist to prevent; and a broken lab that produces an ambiguous failure, teaching a trainee to distrust or misread a real denial.

**Distributed-systems concerns.** Reset consistency after a fault has been injected and observed, so a broken lab can be re-run deterministically.

**Decision gates.** The controlled fault-injection mechanism itself — not selected by this roadmap.

**Completion criteria.** A reviewed framework naming, for at least five of the broken-lab examples above, the controlled starting state, expected failure, owning component, authoritative evidence, and reset procedure.

**Operator-mode representation.** Not directly applicable; broken labs are a Training Mode capability specifically.

**Training-mode explanation.** The two flagship invariants above (authentication ≠ authorization; authorization ≠ execution), demonstrated concretely through deliberately broken scenarios rather than only asserted in the abstract.

**Evidence and audit requirements.** Every broken-lab run's evidence traced back to the real component that produced the failure, per Phase 4's observability model — a broken lab never fabricates its own evidence of failure.

**Schema and documentation impact.** None beyond what Phase 3's scenario model already anticipates.

**Explicitly deferred work.** The fault-injection mechanism itself; the full broken-lab catalog beyond the illustrative examples above.

**Engineering experience developed.** Fault-injection design bounded by a hard security constraint — building a mechanism that can break a system on purpose without ever being usable to break it by accident.

### Phase 8 — Managed IdP Comparison Profile

**Purpose.** Define the Auth0 comparison profile per **Auth0 Comparison Profile** above, and the provider-equivalence requirements a lab must satisfy to demonstrate that BASIS's identity-provider integration is standards-based rather than Keycloak-specific.

**Primary repositories.** `basis-architecture`; `basis-demo` (future).

**Prerequisites.** Phase 3's provider-neutral scenario model; Phase 5's (and, eventually, Phase 6's) Keycloak-realized baseline scenario, which the Auth0 profile re-realizes rather than duplicates.

**Architectural outcome.** The same abstract scenario Phase 3 defines and Phase 5 realizes under Keycloak is separately realized under an Auth0 profile, demonstrating standards equivalence rather than feature-for-feature Auth0 coverage. Auth0 remains strictly optional, per the invariant already stated in **Auth0 Comparison Profile** above.

**Key capabilities.** An Auth0 tenant/application profile mapping; a side-by-side Training Mode presentation showing the same protocol concepts realized through Keycloak's and Auth0's differing administrative surfaces; and an explicit non-goal — this phase does not pursue Auth0 feature parity or Auth0-specific advanced capability.

**Security and abuse cases.** A comparison profile silently testing a materially different scenario under Auth0 than under Keycloak, undermining the equivalence claim this phase exists to demonstrate.

**Distributed-systems concerns.** None beyond what Phase 2 already names; an externally managed IdP introduces network-dependency considerations a local Keycloak lab does not have, which this phase's completion criteria should account for without this roadmap prescribing a specific mitigation.

**Decision gates.** Whether any Auth0-specific paid feature is ever exercised by a lab — this roadmap's default answer, stated above, is no.

**Completion criteria.** A reviewed Auth0 profile realizing the same scenario Phase 3 defines and Phase 5 realizes under Keycloak, with no lab capability made to depend on it.

**Operator-mode representation.** Not applicable beyond what Phase 5 already defines, realized under a different profile.

**Training-mode explanation.** How the same underlying standard (OIDC Authorization Code + PKCE) is recognizable across two differently administered providers, reinforcing the learning hierarchy in **Provider-Neutral IdP Profile Model** above.

**Evidence and audit requirements.** None beyond what Phase 4 and Phase 5 already require.

**Schema and documentation impact.** None beyond what Phase 3 already anticipates.

**Explicitly deferred work.** Any Auth0-specific advanced feature; any additional managed-IdP profile beyond Auth0.

**Engineering experience developed.** Demonstrating standards portability empirically, not just asserting it.

### Phase 9 — Workload, Machine, and Delegation Learning Scenarios

**Purpose.** Add architecture for client-credentials, workload identity, mTLS, producer identity, and delegation learning scenarios, strictly bounded to where the owning roadmap has actually made the underlying capability available.

**Primary repositories.** `basis-architecture`; `basis-demo`, `basis-identity`, `basis-gateway` (future, gated).

**Prerequisites.** [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) Phase 3 (token exchange and delegation) and Phase 4 (workload and non-human identity) reaching implementation; Phase 6's producer-authentication lab (mTLS) as the nearest-term workload-identity scenario this roadmap can already ground in the accepted, unimplemented ADR-0008 architecture, once it is implemented.

**Architectural outcome.** A workload-identity lab exercises the real mTLS producer-authentication profile ADR-0008 defines once it is implemented; a token-exchange or delegation lab exercises the real capability the identity/FGA roadmap's Phase 3 defines once it, too, is implemented. This phase invents no interim substitute for either.

**Key capabilities.** A producer/mTLS-scoped workload-identity lab, buildable once Phase 6 is; a token-exchange/delegation lab, buildable only once the identity/FGA roadmap's Phase 3 is implemented; and an explicit human-versus-producer-identity teaching scenario per **Human Identity Versus Producer Identity** above.

**Security and abuse cases.** All confused-deputy and delegation-escalation abuse cases already named in the identity/FGA roadmap's Phase 3 and Phase 4, and in ADR-0008's **Security consequences** section, apply unchanged inside a lab context.

**Distributed-systems concerns.** None beyond what the owning roadmap's own phases already name.

**Decision gates.** None new; this phase inherits every open decision gate from the roadmaps it depends on.

**Completion criteria.** A reviewed lab design for the mTLS-scoped workload-identity scenario (buildable once Phase 6 is), with the token-exchange/delegation scenario explicitly marked not ready until its own prerequisite roadmap phase is implemented.

**Operator-mode representation.** Workload identity, its owner, and its trust state, mirroring the identity/FGA roadmap's own Phase 4 operator-mode representation once that capability exists.

**Training-mode explanation.** The distinction between human, workload, and delegated identity, and how a workload's certificate-derived trust is established, reusing the identity/FGA roadmap's own Phase 4 training-mode content rather than authoring a competing explanation.

**Evidence and audit requirements.** Whatever the owning roadmap's own phases already require, projected through Phase 4's observability model.

**Schema and documentation impact.** None from this roadmap; any future contract belongs to the owning roadmap.

**Explicitly deferred work.** Every scenario gated on identity/FGA roadmap implementation that has not occurred.

**Engineering experience developed.** Recognizing, and respecting, the boundary between "this would make a good lab" and "this capability actually exists to build a lab from."

### Phase 10 — Playbook Integration, Reproducibility, Conformance, and Security Validation

**Purpose.** Define playbook/version relationships, repeatability, lab reset, provider-profile conformance, expected evidence, regression testing, secret-leak testing, offline/local behavior, and documentation/workshop readiness — the hardening phase for everything Phases 1 through 9 define.

**Primary repositories.** `basis-architecture`; all repositories touched by earlier phases; `basis-console` for playbook integration specifically.

**Prerequisites.** Phases 1 through 9; [`docs/roadmaps/operator-and-training-experience.md`](operator-and-training-experience.md)'s own Stage 4 (Training playbook framework) existing, since this phase's playbook integration cannot proceed in a format that roadmap has not yet defined.

**Architectural outcome.** Every lab this roadmap defines, once implemented, carries a stated, tested reproducibility guarantee (a lab reset returns to a known state), a conformance check against the provider-neutral scenario it claims to realize (per Phase 3), and a regression and secret-leak test suite verifying that **Sensitive Artifact Rules** above is actually enforced in the built artifact, not merely specified in this document — the same empirical-validation discipline every other roadmap in this repository applies to its own final phase.

**Key capabilities.** Lab-to-playbook mapping, once a playbook format exists; version pinning for lab component and IdP-profile combinations; automated secret-leak testing against every sanitized trace this roadmap's labs produce; offline/air-gapped lab operation testing, consistent with the OT operational realities every other roadmap in this repository already accounts for; and documentation sufficient for a workshop or classroom setting to run a lab without bespoke support.

**Security and abuse cases.** This entire phase is a security and abuse-case validation exercise; it should specifically re-test the redaction and fault-isolation concerns named in Phase 4 and Phase 7 under repeated, automated conditions, not only under a single manual walkthrough.

**Distributed-systems concerns.** None beyond what earlier phases already name; this phase validates them rather than introducing new runtime behavior.

**Decision gates.** How labs are authored, versioned, and validated within whatever playbook format the Operator/Training roadmap's Stage 4 eventually defines; and whether any future contract this roadmap's earlier phases identified as a candidate is actually mature enough for `basis-schemas`, per **Schema Discipline** below.

**Completion criteria.** This phase has two distinct completion criteria that must not be conflated, mirroring the same distinction every other roadmap in this repository draws for its own final phase: architecture-planning completion requires a reviewed and accepted validation strategy; the implementation program's own completion is not reached until reference labs exist, the validation described has actually been executed, and empirical evidence supports every reproducibility and security claim — a state this roadmap does not reach and does not claim to reach by being accepted.

**Operator-mode representation.** Not directly applicable; this phase is primarily a Training Mode and lab-infrastructure validation concern.

**Training-mode explanation.** How a trainee progresses from a guided lab walkthrough to independent lab operation, reusing the Operator/Training roadmap's own progression model (Orientation → Guided fundamentals → Guided investigations → Independent exercises → Operator transition) rather than defining a competing one.

**Evidence and audit requirements.** Test evidence for every reproducibility, conformance, and secret-leak claim, retained as the basis for any future claim that a lab is workshop-ready.

**Schema and documentation impact.** None directly; this phase validates prior phases' conceptual guarantees.

**Explicitly deferred work.** All actual validation execution, which depends on reference labs from Phases 5 through 9 that do not yet exist, and on the Operator/Training roadmap's Stage 4 playbook framework, which does not yet exist either.

**Engineering experience developed.** Reproducibility and conformance engineering applied to an educational environment — a discipline this repository has not yet needed for any of its four existing roadmaps, each of which validates production capability rather than a learning environment.

---

## Cross-Phase Security Concerns

Some threats span more than one phase above and are easy to under-address if each phase is reviewed only in isolation, mirroring the same discipline every other roadmap in this repository already applies to its own cross-phase concerns. Redaction bypass and sensitive-artifact leakage (Phases 4, 7, 10) recur wherever a sanitized trace or a broken-lab scenario is rendered, and are the specific reason Phase 10 requires automated secret-leak testing rather than trusting Phase 4's specification alone. Fault-injection escape (Phases 2, 7) — a lab control reaching production — is the single most severe risk this roadmap names, guarded architecturally by the **`basis-demo` invariant** and validated empirically by Phase 10. Premature-capability simulation (Phase 6 specifically, but relevant wherever a phase is tempted to fake a step that does not exist yet) is named explicitly in Phase 6's key capabilities as a rejected practice: this roadmap does not permit a lab to simulate producer authentication, execution, or any other unimplemented step to appear complete. Provider-profile inequivalence (Phases 3, 8) — a Keycloak-realized and an Auth0-realized version of the same scenario silently testing different things — undermines the standards-first pedagogical claim this roadmap exists to make. Sample-credential confusion (Phase 2) — a demo-provisioned identity mistaken for or reused as a production one — is named explicitly under Phase 2's security and abuse cases. This roadmap does not attempt to fully resolve any of these here; it identifies where each must be addressed and notes that [`docs/security/threat-model.md`](../security/threat-model.md) will require updates as each phase's architecture solidifies, in the same way every other roadmap in this repository already commits to for its own phases.

---

## Console Education Matrix

| Capability | Operator mode | Training mode |
| --- | --- | --- |
| Lab topology and reset | Not applicable | What real components a lab instantiates, and what "reset" actually restores |
| IdP profile (Keycloak/Auth0) | Not applicable | The same scenario realized under two providers' differing administrative surfaces |
| Authentication flow (steps 1–16) | Session and identity-context state | Redirect, PKCE, JWT, JWKS, issuer/audience/lifetime validation |
| Producer authentication/admission (steps 17–18) | Producer and trust status, once implemented | mTLS handshake, URI SAN identity, admission matching |
| Operation and authorization (steps 19–20) | Canonical action/resource and decision | Normalization, policy evaluation, deny precedence |
| Enforcement and execution (steps 21–22) | Disposition and execution state, once implemented | Why `ALLOW` is not proof of execution |
| Evidence and correlation (step 23) | Evidence references for a given operation | How every step's evidence traces back to its authoritative source |
| Broken labs | Not applicable | Deliberately induced failure, its owning component, and its evidence |
| Redaction | Not applicable | What is shown, what is redacted, and why |

Detailed explanations of each row belong in the corresponding phase section above; this table exists to make the operator/training split scannable at a glance, not to substitute for those sections.

---

## Decision Gates

The following major implementation choices are explicitly left open by this roadmap. For each, later work — not this document — resolves it.

**`basis-demo` repository creation timing.** Not resolved; Phase 2 names the responsibility model this decision must satisfy once made.

**Scenario-definition format.** Not resolved; named in Phase 3.

**IdP-profile configuration format.** Not resolved; named in Phase 3.

**Sanitized training-event representation.** Not resolved; named in Phase 4.

**Whether training traces are ephemeral, derived, or later ledger-backed.** Not resolved; named in Phase 4, explicitly deferred to the post-authentication activity roadmap's own maturity.

**Trace transport and retention.** Not resolved; named in Phase 4.

**Lab versioning.** Not resolved; named in Phase 10.

**Provider-profile conformance mechanism.** Not resolved; named in Phase 10.

**Controlled fault-injection mechanism.** Not resolved; named in Phase 7.

**How much deeper raw-protocol inspection an isolated lab may expose.** Not resolved; named in **Sensitive Artifact Rules** above as requiring a dedicated future architecture and security decision.

**Whether provider-specific extensions belong in core lab definitions or in provider profiles.** This roadmap's default position (provider profiles) is stated in **Provider-Neutral IdP Profile Model** above, but is not treated as a closed decision.

**How offline/air-gapped workshops operate.** Not resolved; named in Phase 10.

**Whether any future contract this roadmap identifies is mature enough for `basis-schemas`.** Not resolved; governed by **Schema Discipline** below.

---

## Schema Discipline

No new schema is published as part of this task or this roadmap's acceptance. Where a phase above identifies a future contract candidate — a lab/scenario descriptor, a provider profile, a sanitized training trace, playbook metadata, or an expected-outcome/conformance result — it is named only as a candidate, following the same readiness discipline ADR-0005 already established for the operation-aware contract suite: architecture and at least one reference implementation must stabilize before `basis-schemas` publication is proposed, per [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md)'s own "implementation proves a stable shape" principle.

---

## `basis-demo` Must Not Become a Product Runtime Dependency

Production BASIS operation must not depend on `basis-demo`. A production deployment must not require lab orchestration, training fixtures, fake identities, failure-injection controls, provider-comparison metadata, or demo state of any kind to authenticate, authorize, enforce, or execute. `basis-demo` consumes BASIS; BASIS must not depend on `basis-demo`. The same holds for Training Mode itself, restated from doctrine already established elsewhere in this repository:

> Production security behavior exists independently of the educational presentation layered over it.

---

## Required Long-Term Experience

The intended eventual experience this roadmap works toward, illustrative and non-committal:

```text
Choose identity provider:
    Keycloak
    Auth0

Choose lab:
    OIDC Authorization Code + PKCE
    (later additional scenarios, as their dependencies become available)

Start lab.

A real identity flow executes. Training Mode reconstructs an inspectable sequence:

Authorization request
    → redirect
    → authentication
    → authorization code
    → PKCE exchange
    → token issuance
    → signature/key selection
    → issuer validation
    → audience validation
    → subject normalization
    → producer authentication
    → producer admission
    → operation normalization
    → policy evaluation
    → decision
    → enforcement
    → execution result
    → evidence
```

Every step should eventually answer: what happened; which component owned it; what security decision was made; what evidence proves it; what failure would look like; what is safe to show; what is intentionally redacted; how it would appear in Keycloak; and how the same standard would appear in Auth0. As **Required Dependency / Lab Availability Matrix** above makes explicit, the flow through external identity validation, normalization, and BASIS subject construction is substantially supported today by `basis-identity` `v0.1.0`'s released capability. Producer authentication and admission, operation-producer integration, execution-result evidence, and full-chain correlation remain gated on the later architecture and implementation work this roadmap's Phase 6 and Phase 9 depend on.

---

## Non-Goals

This roadmap explicitly rejects: turning BASIS into Auth0, Okta, Keycloak, Entra ID, or another full enterprise identity provider; implementing an alternate Training Mode authentication stack; adding OAuth/OIDC semantics to `basis-core`; moving gateway responsibilities into the demo environment; creating a second activity or audit ledger; teaching only vendor configuration in place of the underlying standard; storing reusable credentials for educational convenience; exposing secrets in production traces; building every identity protocol immediately; requiring managed SaaS providers for the open-source learning experience; creating a certification platform; implementing AI-based identity decisions; allowing AI or Training Mode to bypass deterministic BASIS authorization; and interrupting current BASIS implementation priorities, specifically the trusted operation-producer identity and integration work [`ROADMAP.md`](../../ROADMAP.md) already names as active.

---

## Sequencing and Dependency Guidance

The expected high-level sequence is:

```text
Phase 1: existing capability, roadmap, and evidence inventory
    (architecture discovery only — may proceed independently
    or in parallel, when architecture capacity permits)
    ↓
Phase 2: basis-demo responsibility and trust-boundary model
    ↓
Phase 3: provider-neutral identity scenario and IdP profile model
    ↓
Phase 4: sanitized training trace and cross-component observability model
    ↓
Phase 5: baseline Keycloak OIDC reference learning flow (authentication scope)
    ├──→ Phase 7: controlled failure and broken-lab framework
    │        (authentication-scoped broken labs only, until Phase 6)
    └──→ Phase 8: managed IdP comparison profile
    ↓
Phase 6: end-to-end identity-to-operation learning flow
    (gated on ADR-0008 implementation — now `Accepted` as architecture —
    and on docs/architecture/operation-producer-and-execution-boundary.md
    Stage 5's bounded reference implementation — not a near-term phase)
    ↓
Phase 9: workload, machine, and delegation learning scenarios
    (gated on identity/FGA roadmap Phase 3 and Phase 4)
    ↓
Phase 10: playbook integration, reproducibility, conformance,
          and security validation
    (gated on Operator/Training roadmap Stage 4)
```

This sequence is provisional, reflecting the dependency structure named in each phase's Prerequisites subsection above, not a committed schedule. Phase 5's authentication-only scope is deliberately positioned ahead of Phase 6 in this diagram because its prerequisites — `basis-identity` `v0.1.0`'s released OIDC capability — already exist, while Phase 6's prerequisites do not; this reflects real repository state discovered during this roadmap's review, not a preference. Phase 7 and Phase 8 branch from Phase 5 rather than waiting on Phase 6, since broken labs and provider comparison are both meaningful within the authentication-only scope alone. Nothing in this diagram authorizes any phase to begin implementation; every phase remains architecture-planning only until a separate implementation decision is made, consistent with how every other roadmap in this repository treats its own sequencing diagram. This roadmap's acceptance does not reorder, pause, or deprioritize the ecosystem's active implementation priority — the trusted operation-producer identity and integration work named in [`ROADMAP.md`](../../ROADMAP.md) — which several phases above (most acutely Phase 6) depend on completing first.

---

## Deferred Decisions

The following decisions are intentionally left open until implementation planning begins for the relevant phase, beyond what **Decision Gates** above already states.

**Whether `basis-demo` is a single repository or a thin orchestration layer over per-component fixtures.** Resolved by Phase 2's implementation planning, informed by how the other four roadmaps' own future repositories (if any) end up structured.

**Final scenario-descriptor and IdP-profile file formats.** Resolved by Phase 3, following the same "implementation proves a stable shape" discipline this document applies throughout.

**Final sanitized-training-trace transport and retention.** Resolved by Phase 4, in coordination with whatever the post-authentication activity roadmap's own Phase 2 eventually decides for its durable ledger, so the two do not diverge unnecessarily.

**Whether an instructor-only deeper-inspection mode is ever built for isolated classroom deployments.** Resolved, if ever, only through a dedicated future architecture and security decision, per **Sensitive Artifact Rules** above.

**Fault-injection mechanism.** Resolved by Phase 7, informed by whatever isolation guarantee Phase 10's validation work can actually demonstrate.

**Auth0-specific feature scope, if any is ever added beyond the baseline comparison profile.** Resolved, if ever, independently of this roadmap's completion, per the strict-optionality invariant in **Auth0 Comparison Profile** above.

**Exact schema-proposal timing for any candidate contract this roadmap names.** Resolved individually, phase by phase, per **Schema Discipline** above.

---

## Summary

This roadmap defines the missing bridge between an external identity provider, a real identity flow, BASIS authorization, enforcement, execution, and evidence, and safe, repeatable Training Mode inspection of all of it — the specific, narrow gap none of the four existing roadmaps in this repository already covers, because each of them governs a different layer of the same larger system. It assigns `basis-demo` a bounded orchestration and reference-scenario role that consumes the real ecosystem rather than reimplementing any part of it, and it holds that boundary as the single most important invariant in the document. It separates what is buildable today — an authentication-only Keycloak reference lab, grounded in `basis-identity`'s already-released `v0.1.0` capability — from what remains gated on architecture that is, at best, `Accepted` and unimplemented, and it refuses to claim otherwise for pedagogical convenience. It teaches two flagship invariants — authentication is not authorization, and authorization is not execution — using the ecosystem's own already-established architecture, implemented in some cases and accepted but still unimplemented in ADR-0008's case, rather than inventing new doctrine to make the point. It reconciles explicitly with all four existing roadmaps, naming what each already owns and what this roadmap merely consumes. And it remains, throughout, exactly what its status line states: planned only, architecture-planning work, not an implementation authorization, not a commitment to build `basis-demo` on any particular timeline, and not permission to interrupt the ecosystem's actual current priority.

---

## Related Documents

- [`ROADMAP.md`](../../ROADMAP.md) — the ecosystem's current phase structure and status, including the Downstream Rollout Sequence that already names `basis-demo` end-to-end validation as a future stage this roadmap elaborates, and the **Next Producer and Execution-Evidence Boundary** section naming the ecosystem's active priority this roadmap does not interrupt
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) — the architecture-planning document this roadmap's Phase 6 depends on most directly, and the source of the trust-establishment and lifecycle rules this roadmap's two flagship learning invariants restate
- [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) — the accepted, unimplemented producer-authentication decision this roadmap's Phase 6 and Phase 9 are gated on
- [`docs/roadmaps/identity-and-fine-grained-authorization-expansion.md`](identity-and-fine-grained-authorization-expansion.md) — the companion roadmap for BASIS-internal identity and authorization platform expansion; this roadmap's Phase 9 consumes its Phase 3 and Phase 4 rather than redefining them
- [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](identity-to-operation-contract-and-interoperability.md) — the contract and evidence foundation this roadmap's end-to-end learning flow visualizes rather than redefines
- [`docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`](post-authentication-identity-activity-correlation-and-detection.md) — the durable identity-activity roadmap this roadmap's sanitized training trace must not duplicate, per **Relationship to Existing Roadmaps** above
- [`docs/roadmaps/operator-and-training-experience.md`](operator-and-training-experience.md) — the Training Mode doctrine and future playbook framework this roadmap's Phase 10 depends on and does not redefine
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction this roadmap's Repository Responsibility Model extends
- [`docs/architecture/basis-identity.md`](../architecture/basis-identity.md) — the identity engine architecture this roadmap's identity-flow phases exercise without redefining
- [`docs/architecture/basis-gateway.md`](../architecture/basis-gateway.md) and [`docs/architecture/basis-console.md`](../architecture/basis-console.md) — the gateway and console architectures this roadmap's invariants extend
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) and [`docs/architecture/operation-aware-evidence-provenance-semantics.md`](../architecture/operation-aware-evidence-provenance-semantics.md) — the redaction-tier and evidence-provenance discipline this roadmap's Sanitized Artifact Rules extend
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — the "implementation proves a stable shape" principle this roadmap's Schema Discipline applies
- [`docs/architecture/reference-vs-implementation.md`](../architecture/reference-vs-implementation.md) — the conceptual/reference/implementation distinction this roadmap's `basis-demo` sections observe
- [`docs/security/threat-model.md`](../security/threat-model.md) — the existing threat model that later phase-specific work must update
- [`GOVERNANCE.md`](../../GOVERNANCE.md) — the ADR process this roadmap's decision gates and sequencing changes must follow
- [`docs/adr/README.md`](../adr/README.md) — when a phase's architecture work requires a new ADR
- [`docs/glossary.md`](../glossary.md) — the canonical terminology reference this roadmap's working vocabulary should graduate into as each phase stabilizes
