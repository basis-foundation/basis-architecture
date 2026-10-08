# Basitra Ecosystem Identity and Boundary-Aware Security

**Status:** Architecture strategy governed by [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md), which is `Accepted`. Where ADR-0019's Decision cites a section of this document (the definition in §5.1, the status mapping in §5.3 and §6, the convention in §5.5, the BASac gaps in §9.3, and the naming-root criteria in §10.2), that section is the referenced content of the accepted decision. The rest of the document, including the conflict recommendations, the adoption roadmap, the community principles, and the open decisions in [Section 13](#13-open-decisions), remains strategy and planning, not accepted architecture. Nothing here authorizes a change to any repository, package, organization setting, or published artifact. Each follow-on change described in [Section 12](#12-cross-repository-adoption-roadmap) needs its own bounded PR, and until those PRs are made, the repository's canonical reference documents (glossary, writing guidelines, terminology rules, `SECURITY.md`) keep their current text. [ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md), which is `Accepted`, resolves [OD-8](#13-open-decisions). Where text below describes BASIS's long-term scope within Basitra as undecided (for example C-5, §8, §10.1, and §12.5), that text predates the acceptance and ADR-0024 governs. Its reconciliation is bounded follow-on work ([Section 12.4](#124-stage-4--basitra-target-architecture-and-repository-naming-reconciliation)). [ADR-0025](../adr/0025-basitra-repository-and-component-naming.md), which is `Accepted`, resolves [OD-9](#13-open-decisions) by fixing target repository and component names. It renames nothing: current names used below, such as `basis-architecture`, `basis-deploy`, and the BASIS Core Services Distribution, remain current until bounded migration and reconciliation work changes them.

This document does four things:

1. It records the current, historical, and ADR-0019-adopted terminology for the project's identity.
2. It defines **Boundary-Aware Security** as an architectural concept and checks that concept against architecture that already exists.
3. It states a long-term community objective for the open-source project.
4. It sets out a staged, cross-repository plan for adopting the ADR-0019 terminology without rewriting history.

The main constraint throughout is that branding must not outrun architecture. Every claim below about what BASIS does is traced to an existing document, ADR, or released contract. Where no such evidence exists, the claim is labeled future work.

The document's naming and migration principle is:

> **Historical continuity informs migration but does not constrain the target architecture.** Basitra's future terminology, repository organization, component boundaries, and public identity should be chosen according to the project's intended long-term architecture. Existing BASIS names are preserved where they remain useful and migrated, narrowed, or retired where they do not.

Existing names, including the BASIS naming structure, the `basis-foundation` GitHub organization, and the `basis-*` repository names, are therefore the *starting point* of a migration, not permanent defaults. Nothing in this document renames or migrates anything; it only records the decisions that would have to be made first ([Section 13](#13-open-decisions)).

---

## Contents

1. [Scope and Non-Goals](#1-scope-and-non-goals)
2. [Status Vocabulary Used in This Document](#2-status-vocabulary-used-in-this-document)
3. [Terminology Register: Current, Historical, and Adopted](#3-terminology-register-current-historical-and-adopted)
4. [Terminology Conflicts and Recommended Reconciliation](#4-terminology-conflicts-and-recommended-reconciliation)
5. [Boundary-Aware Security](#5-boundary-aware-security)
6. [Relationship to Existing Architecture](#6-relationship-to-existing-architecture)
7. [Why Operational Technology Is a Strong Initial Domain](#7-why-operational-technology-is-a-strong-initial-domain)
8. [Ecosystem Vocabulary](#8-ecosystem-vocabulary)
9. [BASac: Prospective Term and Formalization Gaps](#9-basac-prospective-term-and-formalization-gaps)
10. [Naming and Branding Principles](#10-naming-and-branding-principles)
11. [Community-Driven Objective and Principles](#11-community-driven-objective-and-principles)
12. [Cross-Repository Adoption Roadmap](#12-cross-repository-adoption-roadmap)
13. [Open Decisions](#13-open-decisions)
14. [Deferred Work](#14-deferred-work)
15. [References](#15-references)

---

## 1. Scope and Non-Goals

**In scope:**

- recording what the repository currently says about the project's name, acronyms, and organizational entities;
- defining Basitra, Boundary-Aware Security, and the forward re-expansion of BASIS, which ADR-0019 adopts, and reserving BASac as a prospective term;
- mapping Boundary-Aware Security onto existing architecture, with a status for each mapping;
- community principles and a staged adoption roadmap.

**Non-goals:**

- It authorizes no rename or migration of any repository, the `basis-foundation` GitHub organization, any Python distribution, or any import namespace. It does not decide whether any of them *should* eventually be renamed; that is future work ([Section 12.4](#124-stage-4--basitra-target-architecture-and-repository-naming-reconciliation)).
- It does not decide the long-term architectural scope of BASIS within Basitra ([OD-8](#13-open-decisions)).
- It does not change any existing canonical acronym table, glossary definition, `SECURITY.md`, historical ADR, or release note. See [Section 3](#3-terminology-register-current-historical-and-adopted) for which documents ADR-0019's acceptance makes eligible for bounded follow-on updates.
- It adds no kernel semantics, no evaluation behavior, and no `basis-schemas` contract. Boundary-Aware Security, as defined here, describes and organizes the architecture. It is not a new enforcement mechanism.
- It does not define BASac as a normative model or specification.
- It does not decide the organizational relationship between Basitra, the Basis Foundation, and BASAuth. That decision is recorded as open in [Section 13](#13-open-decisions).
- It does not create governance bodies, working groups, or a proposal process.
- It does not redesign or constrain Ipotio.

---

## 2. Status Vocabulary Used in This Document

Two status vocabularies are used, one for terminology and one for architecture.

**Terminology status:**

| Label | Meaning |
| - | - |
| **Current canonical** | Defined in this repository's glossary, writing guidelines, terminology rules, or ecosystem document today. It governs contributions now. |
| **Historical** | Used in earlier artifacts. It stays accurate as a record and is not rewritten. |
| **Accepted (ADR-0019)** | Introduced by this document and adopted by ADR-0019, which is `Accepted`. The canonical reference documents reflect it through the bounded follow-on PRs in ADR-0019 Decision 8. |
| **Prospective** | Named so that it can be discussed, but carrying no normative meaning even though ADR-0019 is accepted. A separate decision must give it one. |
| **Future migration** | A change that ADR-0019's acceptance makes eligible for a later, bounded PR. |

**Architecture status** (used in [Sections 5](#5-boundary-aware-security), [6](#6-relationship-to-existing-architecture), and [9](#9-basac-prospective-term-and-formalization-gaps)):

| Label | Meaning |
| - | - |
| **Implemented** | Realized in a released or merged implementation, as recorded in [`README.md`](../../README.md) and [`ROADMAP.md`](../../ROADMAP.md). |
| **Accepted, not implemented** | Decided by an `Accepted` ADR or an approved architecture document. No implementation exists yet, or it exists only in part. |
| **Established in analysis** | Analyzed in the white paper, the architecture principles, or the threat model, but not realized as a contract, ADR decision, or implementation. |
| **Future** | Not yet architecturally decided. It needs explicit architecture work. |
| **Unrelated** | Not part of Boundary-Aware Security as defined here. It is listed only to prevent a false association. |

---

## 3. Terminology Register: Current, Historical, and Adopted

### 3.1 Current canonical terminology

| Term | Current canonical meaning | Source |
| - | - | - |
| **BASIS** | Acronym. The cited reference documents state *Building Automation Secure Identity Service*, which is now the historical expansion ([Section 3.2](#32-historical-terminology)); the forward expansion adopted by ADR-0019 ([Section 3.3](#33-terminology-adopted-by-adr-0019)) reaches them through a bounded follow-on PR. It refers to the open-source core services distribution governed by the Basis Foundation. | [`writing-guidelines.md`](../standards/writing-guidelines.md) §3.3 and §4.1; [`SECURITY.md`](../../SECURITY.md) |
| **BAS** | Acronym: *Building Automation System*. The white paper's primary domain; this repository uses it in that sense throughout. | [`writing-guidelines.md`](../standards/writing-guidelines.md) §3.3; [`glossary.md`](../glossary.md#building-automation-system-bas); [`README.md`](../../README.md) |
| **Basis Foundation** | The nonprofit open-source governance body. It stewards the architecture, the distribution, and "the open-source repositories under the BASIS namespace." | [`basis-ecosystem.md`](basis-ecosystem.md#basis-foundation); [`glossary.md`](../glossary.md#basis-foundation); [`GOVERNANCE.md`](../../GOVERNANCE.md) |
| **BASIS Core Services Distribution** | The set of open-source, deployable `basis-*` components. | [`basis-ecosystem.md`](basis-ecosystem.md#basis-core-services-distribution); [`glossary.md`](../glossary.md#basis-core-services-distribution) |
| **BASAuth** | The future for-profit commercial company that builds enterprise products and managed services on top of the distribution. It does not govern the open-source work. | [`basis-ecosystem.md`](basis-ecosystem.md#basauth); [`glossary.md`](../glossary.md#basauth); [`GOVERNANCE.md`](../../GOVERNANCE.md#commercial-ownership-and-open-source-governance) |
| **BASIS ecosystem** | The umbrella term for all three layers together: Basis Foundation, BASIS Core Services Distribution, and BASAuth. | [`basis-ecosystem.md`](basis-ecosystem.md#the-three-layers); [`terminology-rules.md`](../standards/terminology-rules.md#the-five-ecosystem-entities) |
| **`basis-*` component names** | Lowercase, hyphenated repository and component names such as `basis-core` and `basis-gateway`. The GitHub organization is `basis-foundation`. | [`terminology-rules.md`](../standards/terminology-rules.md#component-naming-conventions); [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md) |

### 3.2 Historical terminology

| Term | Where it appears | Treatment |
| - | - | - |
| *Building Automation Secure Identity Service* as the BASIS expansion | This repository (`SECURITY.md`, `writing-guidelines.md`), the `basis-poc` README, the BASIS website source, and all prior ADRs, release notes, and white-paper text written under it | Was the current canonical expansion until ADR-0019's acceptance. It remains the accurate historical expansion for everything written under it and is never retroactively replaced. The current documents that still state it as current (`SECURITY.md`, `writing-guidelines.md`) are updated through ADR-0019 Decision 8 follow-on PRs. |
| *Building Automation Systems Identity Shield (BASis)* | The README of the `basis-foundation/basis` repository, an early 2025 predecessor artifact | Historical only. It is not a current term and is not revived. It is recorded so the inventory in [Section 12.2](#122-stage-2--ecosystem-inventory) is complete. |

### 3.3 Terminology adopted by ADR-0019

| Term | Meaning | Status |
| - | - | - |
| **Basitra** | A proper noun, not an acronym. The intended long-term canonical identity of the open-source project, its community, its ecosystem, and eventually its organizational and GitHub namespace. | Accepted (ADR-0019) |
| **Boundary-Aware Security (BAS)** | The architectural security approach defined in [Section 5](#5-boundary-aware-security). The abbreviation is context-qualified and never universal; see [Section 5.5](#55-the-bas-abbreviation-disambiguation-convention). | Accepted (ADR-0019) |
| **BASIS** re-expansion: *Boundary-Aware Secure Identity Service* | Forward-only re-expansion of the existing acronym. It changes the expansion only. It neither fixes BASIS's long-term scope within Basitra ([OD-8](#13-open-decisions)) nor renames anything. | Accepted (ADR-0019) |
| **BASac**: *Boundary-Aware Secure Access Control* | A name reserved for a possible future formal access-control model. | Prospective |

### 3.4 Why the BASIS re-expansion was adopted

BASIS began in building automation. The white paper takes building automation systems as its primary domain, and the name *Building Automation Secure Identity Service* described that starting point accurately.

The project's scope has since broadened, and the record shows it:

- [`README.md`](../../README.md) already describes the architecture as "intended to be broadly applicable across OT contexts including data centers, hospitals, campuses, commercial buildings, and industrial facilities."
- `basis-adapters` normalizes nine protocol families, including OPC UA, DNP3, and IEC 61850, which are not building-automation-specific ([`basis-ecosystem.md`](basis-ecosystem.md#components)).
- [`ROADMAP.md`](../../ROADMAP.md) Phase 5 names cross-sector applicability in industrial process control, power utilities, and water treatment as an open question.

The accepted architecture has also converged on a specific security model. Authorization and trust are evaluated at explicit, separately verified boundaries: gateway ingress, producer admission, producer intake, authorization-to-execution binding, and evidence. They are not inherited from network position or from the previous hop ([Section 6](#6-relationship-to-existing-architecture)).

*Boundary-Aware Secure Identity Service* keeps the acronym and its historical continuity, and replaces a domain label that is now too narrow with a description of the security model the architecture is formalizing. The re-expansion does not presuppose what BASIS will name in the long term. BASIS may remain a component family, become a narrower identity and security subsystem, or take another role; that scope is derived from the Basitra target architecture ([OD-8](#13-open-decisions)), not from current repository names.

The re-expansion is forward-only. Documents written under the original expansion stay accurate as written.

---

## 4. Terminology Conflicts and Recommended Reconciliation

The terminology adopted by ADR-0019 overlaps with existing terms in several places. None of these overlaps is reconciled silently by this document. Each is listed with its current state and a recommended controlled follow-on.

### C-1. "BAS" already means Building Automation System, inside this repository

**Current state.** The writing guidelines' capitalization table defines BAS as *Building Automation System*. The repository uses bare "BAS" in that sense in dozens of places, including the root README's statement of the primary domain, the architecture principles, the white paper, and diagram labels such as `BAS Controller (BACnet)`. The overlap therefore exists within this repository's own vocabulary, not only with industry usage.

**Risk.** Under the ADR-0019 hierarchy, a reader could take an existing bare "BAS" to mean Boundary-Aware Security. That would change the meaning of historical text such as "a BAS controller point" or "the BAS VLAN."

**Recommendation.** Keep *building automation system* as the default meaning of bare "BAS" in this repository, and apply the convention in [Section 5.5](#55-the-bas-abbreviation-disambiguation-convention). With ADR-0019 accepted, a bounded follow-on PR adds a context-qualified second row to the capitalization table. Do not replace the existing row. Do not edit any existing use of "BAS."

### C-2. The BASIS acronym's "BAS" and the canonical "BAS" would no longer align

**Current state.** Under the historical expansion, the "BAS" inside "BASIS" loosely echoes *Building Automation*. Under the forward re-expansion adopted by ADR-0019, it reads as *Boundary-Aware Secure*, while bare "BAS" in this repository still means building automation system (C-1).

**Risk.** Readers may assume that BASIS is "BAS + IS" in the building-automation sense, or in the Boundary-Aware sense, depending on which document they read first.

**Recommendation.** Do not describe BASIS as derived from either expansion of "BAS." Treat BASIS as a standalone acronym with a single canonical expansion at any point in time. The acronym table records which expansion is current, and historical documents keep theirs.

### C-3. Basis Foundation and Basitra's "eventual organization" role

**Current state.** The Basis Foundation is the canonical stewardship body. It governs the open-source work and owns the `basis-foundation` GitHub organization. ADR-0019 describes Basitra as the project and community identity *and eventually an organization*.

**Risk.** Two organizational names could both claim stewardship of the same open-source work. Contributors could be unsure which body reviews or accepts a proposal.

**Recommendation.** Basitra is the intended long-term canonical identity, including eventually at the organizational level. Today:

- The Basis Foundation remains the governing and stewardship body described in [`GOVERNANCE.md`](../../GOVERNANCE.md).
- Basitra does not yet name a governance body.

The long-term relationship is an explicit future decision, not an assumption of permanent coexistence. Two decisions are recorded:

- [OD-1](#13-open-decisions): whether the Basis Foundation is renamed to, succeeded by, or coexists with a Basitra organization;
- [OD-7](#13-open-decisions): whether and how the `basis-foundation` GitHub organization migrates toward a Basitra identity.

Neither this document nor ADR-0019 decides them.

### C-4. BASAuth and the BAS naming root

**Current state.** BASAuth is the future commercial entity. The architecture states that it does not govern the open-source work.

**Risk.** BASAuth begins with "BAS." If BAS becomes a productive naming root for Basitra ecosystem terms ([Section 10.2](#102-bas-as-a-productive-naming-root)), readers could infer that BASAuth is an open-source Basitra component or a Boundary-Aware Security specification. It is neither.

**Recommendation.** State explicitly that BASAuth is not a member of the BAS-derived open-source vocabulary and is not governed through Basitra community processes. Existing commercial and open-source separation rules continue to apply unchanged. Any further BASAuth naming question belongs to BASAuth and is out of scope ([OD-2](#13-open-decisions)).

### C-5. "BASIS ecosystem" as the existing umbrella term

**Current state.** "The BASIS ecosystem" is the canonical umbrella for Foundation, distribution, and BASAuth together.

**Risk.** A "Basitra ecosystem" would overlap with "BASIS ecosystem," and the two would have different membership, since the current umbrella includes a commercial entity.

**Recommendation.** With ADR-0019 accepted, a bounded follow-on PR to the terminology rules should define the relationship explicitly:

- The **Basitra ecosystem** is the open-source project and community, plus independent community work that identifies with it.
- **BASIS** is, today, the technical component family inside it. Its long-term scope within Basitra is [OD-8](#13-open-decisions).
- **BASAuth** remains an external commercial participant that builds on the open-source work.

Until then, "BASIS ecosystem" keeps its current meaning. Existing uses are historical and are not rewritten.

### C-6. Ipotio's existing use of "Basitra"

**Current state.** Ipotio's architecture documents already use "Basitra" to name the authorization system Ipotio consumes. For example: "Ipotio depends on Basitra," "Basitra operation-producer role," and "Basitra gateway / kernel." They also describe ADR-0018 as "Basitra's half" of a matched decision.

**Risk.** That usage treats Basitra as a component or runtime name. ADR-0019 adopts Basitra as a project and community identity, with BASIS as the component family. Role and component references across the two ecosystems could drift.

**Recommendation.** Record a cross-ecosystem reconciliation item: with ADR-0019 accepted, Ipotio's architecture should refer to Basitra for the project and to the component and role names for components and roles. Today those are BASIS names, such as "BASIS operation-producer role" or "`basis-gateway`." Because those names may change under [OD-8 and OD-9](#13-open-decisions), the Ipotio reconciliation should follow those decisions rather than precede them. The dependency direction Ipotio records, Ipotio depending on Basitra/BASIS and not the reverse, is consistent with [ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md) and needs no change. This is a change to the Ipotio repository and is not made here.

### C-7. The "five ecosystem entities" list predates current components

**Current state.** [`terminology-rules.md`](../standards/terminology-rules.md#the-five-ecosystem-entities) lists the distribution components without `basis-producer`, which [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md) established. The component-naming list omits it too.

**Relevance.** This drift already exists and is independent of Basitra. It matters here because Stage 3 ([Section 12.3](#123-stage-3--current-documentation-reconciliation)) would edit the same section. It is noted so that one follow-on PR can reconcile both, not so that it is fixed here.

### C-8. `docs/standards/terminology-guidelines.md` is empty

**Current state.** The README's repository-structure tree describes `terminology-guidelines.md` as "controlled terminology and usage rules," but the file is empty. The substantive rules live in `terminology-rules.md`.

**Recommendation.** A follow-on PR should either remove the empty file and its README reference, or give it a defined purpose. The new terminology in this document follows `terminology-rules.md`.

---

## 5. Boundary-Aware Security

### 5.1 Definition

> **Boundary-Aware Security** is a security approach in which authorization, enforcement, and evidence explicitly account for the security-relevant boundaries a requested operation must cross between the party that initiates it and the resource it affects. A boundary-aware system evaluates not only whether a subject may perform an action on a resource, but whether a specific operation — carried by specific intermediaries, under a specific security context and policy — may cross each boundary on its path. Each crossing is verified on its own terms rather than inherited from the crossing before it, and the outcome of each crossing is attributable in evidence.

The definition extends existing architecture. It does not replace it:

- "Verified on its own terms rather than inherited" restates, in general form, the rule in ADR-0018: "every arrow … is a separate boundary with its own admission or verification rule. None of them inherits trust from the one above it."
- It also restates Principle 2's requirement that intermediaries "must not silently substitute their identity for the original requester's" ([`architecture-principles.md`](../architecture-principles.md#2-identity-propagation-over-network-origin-trust)).
- "Boundaries" means trust boundaries in the sense of [`glossary.md`](../glossary.md#trust-boundary) and white paper [Section 05](../../whitepapers/identity-aware-authorization-for-operational-technology/sections/05-ot-trust-boundaries.md): architectural facts, not firewall locations.

Boundary-Aware Security is **not** a component, a product, a policy language, or a contract. It adds no evaluation semantics to `basis-core`. It is a way of describing, reviewing, and extending the architecture.

### 5.2 What Boundary-Aware Security adds to identity-aware authorization

The repository's existing framing is [identity-aware authorization](../glossary.md#identity-aware-authorization): decisions conditioned on verified identity, not on network location. Boundary-Aware Security keeps that framing and makes three further properties explicit. The accepted architecture already exhibits all three, though only in parts:

1. **The unit of evaluation is an operation along a path, not only a subject–resource pair.** The operation-aware model already evaluates richer operation context than subject, action, and resource ([`operation-aware-authorization-model.md`](operation-aware-authorization-model.md)). The producer and execution architecture already treats the path from initiator to OT target as a sequence of distinct roles ([`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md)).
2. **Distinct identities stay distinct across boundaries.** Authorization-subject identity, producer workload identity, and upstream workload identity are kept separate ([ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md)).
3. **Authorization does not implicitly become execution.** Crossing from an authorization disposition into protocol dispatch is a boundary of its own, with its own binding and evidence ([ADR-0011](../adr/0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](../adr/0012-authorization-to-execution-binding.md), [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md)).

### 5.3 Boundary categories and their current status

The table lists the boundary categories that Boundary-Aware Security may account for, each with its status in this architecture. It is a catalog, not a claim that every category is modeled or enforced.

| Boundary category | What crosses it | Status | Evidence |
| - | - | - | - |
| **Identity boundary** (external identity provider → canonical identity context) | Externally authenticated identity normalized into BASIS identity context | **Implemented** | `basis-identity` v0.1.0 (OIDC, JWKS, session, BASIS-local token); gateway token verification ([threat model §3.1, §3.5](../security/threat-model.md#3-trust-boundaries)); [`identity-authority-modes.md`](identity-authority-modes.md) |
| **Subject / workload / operator distinctions** | Authorization subject vs. producer workload | **Implemented** for producer-to-gateway | [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) and [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md); `basis-gateway` Phase 1B and `basis-producer` Phase 3 merged |
| | Upstream supervisory workload vs. producer workload vs. subject | **Accepted, not implemented** | [ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md) |
| | Human operator vs. system principal (for example, a BMS acting autonomously) | **Established in analysis** | White paper [Section 05](../../whitepapers/identity-aware-authorization-for-operational-technology/sections/05-ot-trust-boundaries.md) (Supervisory Zone) |
| **Resource boundary** | The canonical resource an operation targets | **Implemented** | Canonical resource identifiers and resource type in `basis-schemas` v0.2.0 and `basis-core` v0.2.0; [`resource-identifier-reconciliation.md`](resource-identifier-reconciliation.md) |
| **Authorization / policy boundary** | Normalized request into deterministic kernel evaluation | **Implemented** | Gateway → core boundary ([threat model §3.2](../security/threat-model.md#32-gateway--core)); default deny and deny precedence ([`operation-aware-evaluation-semantics.md`](operation-aware-evaluation-semantics.md)); [`kernel-boundary-rules.md`](../kernel-boundary-rules.md) |
| **Protocol boundary** (protocol-native operation → normalized authorization request) | Protocol intent, carried as evidence | **Implemented** for normalization and evidence construction; protocol execution not implemented | `basis-adapters` v0.2.0 (nine protocol families); [trusted adapter boundary](../glossary.md#trusted-adapter-boundary); [ADR-0007](../adr/0007-adapter-evidence-construction.md) Stage 1 |
| **Producer admission boundary** (producer workload → gateway) | Authenticated operation-aware submission | **Implemented** for its bounded scope | ADR-0008, ADR-0009; [`producer-mtls-proxy-trust-boundary.md`](producer-mtls-proxy-trust-boundary.md) |
| **Producer-intake boundary** (upstream system → operation-producer role) | Supervisory intent | **Accepted, not implemented** | ADR-0018 |
| **Network / security-zone boundary** | Traffic between operational zones | **Established in analysis.** Zone and site identifiers are **implemented** as optional request context. | Five-zone analysis in white paper [Section 05](../../whitepapers/identity-aware-authorization-for-operational-technology/sections/05-ot-trust-boundaries.md). `location` (site, building, zone, area) is an optional field of the published `operation-aware-decision-request` contract. The architecture does not yet model an operation's *crossing* between zones as evaluable context. |
| **Site / physical boundary** | Operations with physical effect at a location | **Implemented** as optional `location`, `device`, and `safety_context` request context. Device identity at scale: **Future**. | `operation-aware-decision-request` contract; device identity enrollment is an open question in [`ROADMAP.md`](../../ROADMAP.md) Phase 5 |
| **Operational boundary** (maintenance mode, safety mode, time windows) | Operations under changing operational state | **Implemented** as optional `safety_context`, `environment_context`, and time context, evaluable through condition operators. Break-glass: **Established in analysis.** | [`condition-operator-semantics.md`](condition-operator-semantics.md); [Principle 11](../architecture-principles.md#11-human-operators-remain-part-of-the-system); [`identity-authority-modes.md`](identity-authority-modes.md) (local break-glass login) |
| **Administrative boundary** | Operations crossing administrative ownership | **Established in analysis** | White paper [Section 05](../../whitepapers/identity-aware-authorization-for-operational-technology/sections/05-ot-trust-boundaries.md) ("Audit Visibility Across Administrative Boundaries") and Section 09. No request-context category in [`operation-aware-authorization-model.md`](operation-aware-authorization-model.md) §3 represents administrative domain. |
| **Organizational / tenant boundary** | Operations across tenants or organizations | **Future** | Multi-tenant identity and trust isolation in the [identity and fine-grained authorization expansion roadmap](../roadmaps/identity-and-fine-grained-authorization-expansion.md) (Planned; not begun) |
| **Authorization-to-execution boundary** | An authorized disposition becoming protocol dispatch | **Accepted, not implemented** | ADR-0011, ADR-0012, ADR-0013, ADR-0015, ADR-0016; [bounded REST execution implementation plan](bounded-rest-execution-implementation-plan.md) approved; protocol execution entirely unimplemented |
| **Evidence / audit boundary** | Decision and execution facts into durable records | Authorization audit evidence: **Implemented**. Execution evidence: **Accepted, not implemented**. Tamper resistance: deployment property. | `basis-core` v0.2.0 trace and audit evidence ([`operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md)); [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md); [threat model §3.7](../security/threat-model.md#37-audit-boundary); [Principle 14](../architecture-principles.md#14-immutable-security-relevant-event-logging) |
| **Deployment boundary** | Runtime placement, configuration, and secrets | **Established in analysis.** `basis-deploy` is not established. | [Threat model §3.6](../security/threat-model.md#36-deployment-boundary) |

Two observations follow from the table:

- The boundaries that are implemented or accepted are almost all on the **request path**: identity, admission, normalization, evaluation, binding, and evidence.
- The boundaries that stay in analysis or future work are **environmental**: zone crossing, administrative domain, tenant, and physical device identity.

Boundary-Aware Security should therefore not be described as covering environmental boundaries until architecture work makes them evaluable.

### 5.4 What Boundary-Aware Security does not claim

- It does not replace network segmentation, zones and conduits, physical security, or safety engineering. Consistent with [`SECURITY.md`](../../SECURITY.md#security-philosophy), identity-aware authorization "does not eliminate the need for network segmentation, physical security, operational governance, … or safety engineering." The same holds for Boundary-Aware Security.
- It does not claim that devices downstream of the last enforcement point can verify authorization. White paper Section 05 states that the field device zone is enforcement-downstream.
- It does not move boundary interpretation into the kernel. Any future boundary-context semantics must still reach `basis-core` as fully assembled, normalized input, per [`kernel-boundary-rules.md`](../kernel-boundary-rules.md) and [`operation-aware-authorization-model.md`](operation-aware-authorization-model.md) §8.

### 5.5 The BAS abbreviation disambiguation convention

*Building automation system (BAS)* is an established industry term. It is also this repository's current canonical meaning of "BAS" ([C-1](#c-1-bas-already-means-building-automation-system-inside-this-repository)). The ecosystem does not redefine the abbreviation. The convention, in effect since ADR-0019's acceptance (Decision 4):

1. **Industry meaning is the default in this repository.** Bare "BAS" in `basis-architecture` continues to mean *building automation system*. Existing uses are not edited.
2. **Boundary-Aware Security is written in full.** Use "Boundary-Aware Security (BAS)" at first use in any document that uses the abbreviation. In any document that also discusses building automation systems, spell out *Boundary-Aware Security* at every use and do not use bare "BAS" for it at all.
3. **Building automation is written in full wherever both meanings could be present.** Use "building automation system (BAS)" at first use.
4. **Diagrams and tables** that label building automation equipment, such as `BAS Controller`, keep their existing labels. New diagrams that refer to Boundary-Aware Security spell it out.
5. **External communication** outside this repository, such as a project website or talk abstract, expands Boundary-Aware Security at first use and does not present BAS as a universal abbreviation for it.

This document follows the convention: after this section, the concept is written as *Boundary-Aware Security*, and building automation is written as *building automation system*.

---

## 6. Relationship to Existing Architecture

This section checks Boundary-Aware Security against the architecture already in the repository. For each concept it states whether the concept supports Boundary-Aware Security, and at what status.

| Existing concept | Relationship to Boundary-Aware Security | Status |
| - | - | - |
| Canonical identity context and identity propagation ([Principle 2](../architecture-principles.md#2-identity-propagation-over-network-origin-trust); [glossary](../glossary.md#canonical-identity-context)) | Identity boundary; the non-substitution rule for intermediaries | **Implemented** (gateway verification and `basis-identity` v0.1.0); `basis-identity` evidence alignment with the operation-aware surface **not begun** |
| Operation-aware authorization model ([ADR-0001](../adr/0001-operation-aware-ot-authorization.md), [model](operation-aware-authorization-model.md)) | Evaluates an operation with context, not only a subject–resource pair | **Implemented** (`basis-schemas` v0.2.0, `basis-core` v0.2.0) |
| Deterministic evaluation semantics (default deny, deny precedence, `NOT_APPLICABLE`, fail-closed errors) | Decision semantics at the authorization boundary | **Implemented** |
| Policy bundle and rule model; condition operators | Policy can condition on published context (location, device, safety, environment, time) | **Implemented**; environmental boundary *crossing* is not modeled (**Future**) |
| Canonical action vocabulary ([ADR-0017](../adr/0017-action-vocabulary-naming-structure.md)) | A normalized operation verb across protocol boundaries | **Accepted.** Migration of the two built-in default drifts it names was not verified in this review. [ADR-0020](../adr/0020-operation-to-authorization-mapping-and-composition-boundary.md) (Accepted) now resolves composite-action composition ownership (I-1) for the governed admitted-producer path; embedded direct-kernel composition remains outside that decision. |
| Protocol normalization and the trusted adapter boundary | The protocol boundary is a semantic trust boundary | **Implemented** (normalization and evidence construction) |
| Adapter evidence construction ([ADR-0007](../adr/0007-adapter-evidence-construction.md)) | Evidence that attributes a decision to the protocol operation that produced it | **Implemented** (Stage 1 in `basis-adapters`; retention and reference lifecycle in `basis-producer`) |
| Producer workload authentication and admission ([ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md)) | Producer identity kept separate from subject identity at the admission boundary | **Implemented** for its bounded scope |
| Operation-producer role ([ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md)) | A defined role on the request path; role distinct from implementation | **Implemented** (bounded authorization slice) |
| Producer-intake boundary ([ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md)) | Explicit per-crossing verification; no trust inherited from upstream | **Accepted, not implemented** |
| Protocol-executor role and authorization-to-execution binding ([ADR-0011](../adr/0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](../adr/0012-authorization-to-execution-binding.md)) | The authorization-to-execution boundary | **Accepted, not implemented.** The bounded plan is approved; Work Item 1 is the next step. |
| Execution lifecycle and execution evidence ([ADR-0013](../adr/0013-execution-lifecycle-semantics.md), [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md)) | Truthful evidence of what crossed into execution | **Accepted, not implemented** |
| Replay and freshness ([ADR-0016](../adr/0016-bounded-target-replay-freshness-posture.md)) | Whether a prior crossing can be reused | Process-local single consumption **accepted** for the first target; broader replay and freshness **open** |
| Architectural decision gates (Gates 1–4 in [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md)) | The practice of not crossing an implementation boundary until its semantics are decided | **Accepted** practice; this is how the Boundary-Aware Security areas in [Section 9](#9-basac-prospective-term-and-formalization-gaps) should also be advanced |
| Kernel isolation ([`kernel-boundary-rules.md`](../kernel-boundary-rules.md)) | The small trusted core that boundary semantics must not erode | **Implemented** and governance-protected |
| Trust boundaries in the threat model and white paper | The conceptual basis of the boundary catalog | **Established in analysis** |
| Operator and Training presentation modes (`basis-console`) | Presentation of decisions. It is not a security boundary. | **Unrelated** to Boundary-Aware Security as a model; it may later *explain* boundary crossings |
| Telemetry ingestion | Separate pipeline with different trust properties ([`terminology-rules.md`](../standards/terminology-rules.md#audit-vs-telemetry)) | **Unrelated.** Boundary-Aware Security as defined here governs operations, not telemetry. |

**Conclusion.** Boundary-Aware Security is an accurate description of the request-path architecture that is implemented or accepted today. It is not yet an accurate description of environmental boundary evaluation. Using the term must not imply otherwise.

---

## 7. Why Operational Technology Is a Strong Initial Domain

Boundary-Aware Security is not inherently limited to operational technology. Access or execution crosses meaningful trust, resource, administrative, or operational boundaries in many other settings: delegated automation acting on behalf of users, multi-party service integrations, or software agents invoking tools with real-world effects. The definition in [Section 5.1](#51-definition) assumes nothing specific to OT. This document makes no claim about whether the approach suits any particular non-OT domain, which is an open question.

OT is a strong initial domain because several of its characteristics make boundaries both numerous and consequential:

- **Heterogeneous and legacy protocols.** Protocol translation is a semantic trust boundary. The authorization decision is only as good as the normalization in front of it (white paper Section 05, "Identity at Protocol Translation Boundaries").
- **Weak or absent native device identity.** Many field devices cannot authenticate, so identity must be established and enforced at the boundaries around them, not at them.
- **Long equipment lifecycles.** Boundaries persist for decades, so incremental adoption at boundaries ([Principle 13](../architecture-principles.md#13-incremental-adoption-over-full-replacement)) is more realistic than replacing devices.
- **Mixed trust and administrative domains.** Facilities, OT engineering, IT security, and vendors administer different sides of the same operational path.
- **Physical consequences of digital actions.** The authorization-to-execution boundary matters more when execution moves physical equipment.
- **Distinct operational roles.** Operators, system principals, maintenance technicians, and remote vendors warrant different treatment at the same boundary.
- **Segmented environments.** Zones already exist operationally, and enforcement placement follows them (white paper Section 05).
- **Safety and availability.** Fail-safe, predictable behavior at each boundary is a requirement, not a preference ([Principles 6 and 7](../architecture-principles.md#6-operational-resilience-first)).

**Relationship to established OT terminology.** The ISA/IEC 62443 series defines the zone and conduit model for segmenting industrial automation and control systems. The white paper already positions its trust-boundary taxonomy against it ([`references/sources.md`](../../whitepapers/identity-aware-authorization-for-operational-technology/references/sources.md)). Boundary-Aware Security is complementary to that model:

- *Zones and conduits* describe how an environment is segmented and how communication between segments is constrained.
- *Boundary-Aware Security* concerns how an individual operation's authorization, enforcement, and evidence account for the boundaries it crosses.

Basitra terminology does not replace, reinterpret, or claim conformance with ISA/IEC 62443 or any other standard.

---

## 8. Ecosystem Vocabulary

The definitions below are adopted by ADR-0019, which is `Accepted`, except BASac, which remains prospective. The corresponding glossary entries are promoted through a bounded follow-on PR (ADR-0019 Decision 8).

### Basitra

The intended long-term canonical identity of the open-source project, its community, and its ecosystem, including independent community work that identifies with it, and eventually of its organizational and GitHub namespace.

- Basitra is a proper noun and does not expand to anything.
- It is deliberately independent of implementation technology and of any single domain.
- It names the project and community. It does not name a component, runtime, or service ([C-6](#c-6-ipotios-existing-use-of-basitra)).
- It does not *yet* name a legal entity or governance body. The Basis Foundation governs today; the long-term relationship, including possible rename or succession and GitHub organization migration, is [OD-1 and OD-7](#13-open-decisions) ([C-3](#c-3-basis-foundation-and-basitras-eventual-organization-role)).

### Boundary-Aware Security (BAS)

The architectural security approach defined in [Section 5.1](#51-definition). It is the conceptual root of the Basitra vocabulary. It is not a software component and does not correspond to a repository.

### BASac: Boundary-Aware Secure Access Control

A **prospective** term for a possible future formal access-control model derived from Boundary-Aware Security. No BASac specification exists. [Section 9](#9-basac-prospective-term-and-formalization-gaps) states what must happen before the term can carry normative meaning.

### BASIS

Today, the technical identity of the component family and its architecture: the `basis-*` components of the BASIS Core Services Distribution and the architecture in this repository.

- **Forward canonical expansion (ADR-0019 Decision 3):** *Boundary-Aware Secure Identity Service* ([Section 3.4](#34-why-the-basis-re-expansion-was-adopted)).
- **Historical expansion:** *Building Automation Secure Identity Service*. It remains accurate for everything written under it.

BASIS is not a synonym for `basis-core`. This document does not assume that BASIS permanently remains the umbrella name for every current `basis-*` component. Its long-term scope within Basitra is an explicit future decision ([OD-8](#13-open-decisions)): it may remain a component family, become a narrower identity and security subsystem, or take another role justified by the Basitra target architecture. No repository rename is authorized by this document.

### Ipotio

A separate operational technology platform ecosystem with its own architecture and governance. Under ADR-0018, Ipotio is one valid *upstream supervisory system*: it may submit supervisory intent to BASIS across the producer-intake boundary. It is not a BASIS operation producer, protocol executor, or authorization authority. BASIS defines no Ipotio-specific contract (ADR-0018 Decision 9, platform neutrality). Ipotio is not part of Basitra governance, and this document does not describe Ipotio's internal architecture.

### Entities whose roles ADR-0019 does not change

- **Basis Foundation** remains the governing and stewardship body today. Its long-term relationship to Basitra is [OD-1](#13-open-decisions) ([C-3](#c-3-basis-foundation-and-basitras-eventual-organization-role)).
- **BASAuth** remains the future commercial entity. It is external to the open-source vocabulary and to Basitra community processes ([C-4](#c-4-basauth-and-the-bas-naming-root)).

### 8.1 Relationship diagram

The diagram shows conceptual and naming relationships, not software dependencies or deployment topology.

```text
Basitra
  intended long-term project / community / ecosystem identity   (Accepted, ADR-0019)
  stewarded today by the Basis Foundation                  (Current canonical; see OD-1, OD-7)
        │
        │  conceptual root
        ▼
Boundary-Aware Security                                    (Architectural concept; ADR-0019)
        │
        ├── BASIS                                          (Current component family;
        │                                                   long-term scope: OD-8)
        │   Boundary-Aware Secure Identity Service         (forward canonical expansion)
        │   Building Automation Secure Identity Service    (historical expansion)
        │   today: basis-core, basis-gateway, basis-identity, basis-adapters,
        │   basis-producer, basis-console, basis-schemas, (basis-deploy: not established)
        │   component names subject to Basitra-first review (Stage 4, OD-9)
        │
        └── BASac                                          (Prospective term only)
            Boundary-Aware Secure Access Control
            no specification exists; see Section 9
```

Ipotio and BASAuth sit outside the diagram above. They are drawn separately to avoid implying membership or governance:

```text
Ipotio                                  BASAuth
  separate OT platform ecosystem          future commercial entity
        │                                       │
        │ may submit supervisory intent         │ may build products and services
        │ across the BASIS producer-intake      │ on the open-source distribution
        │ boundary (ADR-0018)                   │ (basis-ecosystem.md)
        ▼                                       ▼
   BASIS (within Basitra)                  BASIS (within Basitra)

   Neither governs, nor is governed by, Basitra community processes.
   BASIS does not depend on either.
```

---

## 9. BASac: Prospective Term and Formalization Gaps

### 9.1 Why BASac is not yet a formal term

[`terminology-rules.md`](../standards/terminology-rules.md#introducing-new-terms) prohibits synonyms: "Inventing a synonym creates two terms for one concept." If BASac were adopted now, the only existing concept it could name is the **operation-aware authorization model**, which already has a canonical name and an ADR. BASac would then be a synonym.

BASac therefore earns a normative meaning only if it names something the operation-aware model does not: an access-control model covering the whole boundary-crossing path, not only the kernel's evaluation.

### 9.2 What already exists that a BASac model would build on

| Element | Existing basis | Status |
| - | - | - |
| Subject context | Canonical identity context; subject attributes; authority mode reference | **Implemented** |
| Operation | Canonical action verbs (ADR-0017); operation intent; protocol operation as evidence | **Implemented** (composition ownership accepted for the governed admitted-producer path by ADR-0020; embedded direct-kernel composition remains outside that decision) |
| Target resource | Canonical resource identifier and resource type | **Implemented** |
| Boundary context | Optional `location`, `device`, `protocol_context`, `safety_context`, `environment_context` request context | **Implemented** as individual context fields; no model of boundary *crossings* |
| Decision semantics | Default deny, deny precedence, `NOT_APPLICABLE`, fail-closed failures | **Implemented** |
| Enforcement | Gateway enforcement and pre-kernel validation; enforcement contracts | **Implemented** for authorization; **Accepted, not implemented** for execution |
| Authorization-to-execution binding | ADR-0012 same-process binding record | **Accepted, not implemented** |
| Evidence | Authorization trace and audit evidence; adapter evidence; execution evidence (ADR-0014) | Authorization and adapter evidence **implemented**; execution evidence **accepted, not implemented** |

### 9.3 Gaps to resolve before BASac can become a formal model

Each item needs architecture work and, where it changes a compatibility surface or boundary rule, an ADR. None is resolved here.

1. **Scope decision.** Decide whether BASac is (a) a name for the composed, end-to-end model spanning intake, admission, evaluation, binding, execution, and evidence, or (b) unnecessary, because the operation-aware model plus the existing ADR chain already covers it. If (b), BASac stays unused.
2. **Boundary-crossing representation.** Decide how, if at all, an operation's path of crossings is represented to policy (for example, source and destination zone, or administrative domain), and which component assembles it. Kernel purity requires that it arrive fully assembled.
3. **Context-assertion trust.** Decide who may assert boundary context and how its authority, provenance, and freshness are governed for admission. ADR-0018 explicitly defers context-assertion trust. [ADR-0021](../adr/0021-upstream-context-assertion-trust-boundary.md) (`Status: Accepted`) establishes category-scoped, origin-preserving assertion trust for the existing context categories; it is not implemented, and it does not address boundary-crossing representation (item 2).
4. **Multi-crossing decision semantics.** Decide whether each crossing yields its own decision or whether one decision covers the path, and how partial failures compose. This must fit the fail-closed rules already accepted.
5. **Broader replay and freshness.** This is open globally (ADR-0012, ADR-0016). A path-level model cannot be normative while crossings can be replayed outside the first bounded target.
6. **Execution evidence publication.** ADR-0014 declines schema publication for now. A normative model that relies on execution evidence needs a published contract.
7. **Administrative and tenant boundaries.** These depend on the identity and fine-grained authorization expansion roadmap.
8. **Conformance criteria.** Define what an implementation must demonstrate to claim conformance, consistent with [`compatibility-philosophy.md`](compatibility-philosophy.md).
9. **Implementation evidence.** At minimum, the bounded REST execution slice (Work Items 1–4) should be implemented before the authorization-to-execution portion of any BASac model is treated as validated.

---

## 10. Naming and Branding Principles

### 10.1 Principles

0. **Historical continuity informs migration but does not constrain the target architecture.** Names are chosen from the intended long-term architecture. Existing BASIS names are preserved where they remain useful and migrated, narrowed, or retired where they do not. Historical existence alone is not a justification for keeping a name.
1. **Public identity and technical names can differ.** Basitra is the intended long-term canonical identity of the project. Component names are chosen by architectural responsibility and need not repeat the project name.
2. **Basitra is the umbrella identity.** It names the project, community, and ecosystem, and eventually the organizational namespace. It does not name a component.
3. **BASIS is the current technical identity, with its long-term scope to be decided.** It is the component family's name today. Whether it remains that, narrows to an identity and security subsystem, or takes another role is derived from the Basitra target architecture ([OD-8](#13-open-decisions)), not from current repository names.
4. **Boundary-Aware Security is a concept, and "BAS" is a naming root, not a replacement string.** "Boundary-Aware Security" does not replace "BASIS," and the abbreviation does not replace *building automation system*.
5. **BASac must earn formal meaning** ([Section 9](#9-basac-prospective-term-and-formalization-gaps)).
6. **Repository names are evaluated Basitra-first, and renamed only by explicit decision.** No repository rename is authorized by this document. Each repository is evaluated in [Stage 4](#124-stage-4--basitra-target-architecture-and-repository-naming-reconciliation) by asking: *if Basitra had been the project identity from the beginning, what should this component and repository be called based on its architectural responsibility?* A current name may remain if it still answers that question well.
7. **History stays accurate; active surfaces may migrate.** Historical ADRs, release notes, tags, white-paper text, and released artifacts written under earlier terminology remain accurate and are not rewritten. Supersession follows the [ADR supersession rule](../adr/README.md#supersession), and only the status line of a superseded ADR changes. Current, active project surfaces, such as READMEs, current-state documentation, repository descriptions, package metadata, and the public site, may be deliberately migrated to Basitra terminology over time.
8. **No mass replacement; migrations are planned.** Terminology migration happens one bounded PR at a time. Repository, package, organization, and public-site migrations may be appropriate where justified, each with its own compatibility and migration plan (for example, redirects, deprecation periods, and dual publication where needed).
9. **Terms follow architecture.** New terminology must correspond to a real architectural concept that exists in, or is decided by, this repository.
10. **No acronym-driven names.** A name whose main purpose is to fit an acronym is rejected.
11. **Durable across technology.** Basitra should not depend on any implementation language, deployment model, or single domain.
12. **Room for independent work.** Community extensions should be able to identify with the Basitra ecosystem without living in first-party repositories. Rules for how they may describe that relationship are future work ([Section 12.7](#127-stage-7--community-readiness)).

### 10.2 BAS as a productive naming root

Mature ecosystems often develop a recognizable vocabulary around a shared root. "BAS" may serve that purpose for Basitra. BAS-derived names are optional vocabulary, not a requirement imposed on components: nothing needs to begin with "BAS," and a plain descriptive name is preferred wherever a BAS-derived one would not correspond to a genuine architectural concept. This document lists no speculative names.

The current meaningful members are **Basitra**, **Boundary-Aware Security**, **BASac** (prospective), and **BASIS**. BASAuth shares the letters but is not a member ([C-4](#c-4-basauth-and-the-bas-naming-root)).

A future BAS-derived name should be adopted only when all of the following hold:

1. The underlying architectural concept is real, meaning decided or implemented in this repository.
2. The name improves clarity over the plain descriptive term.
3. Its expansion is accurate and not contrived.
4. It does not collide with established industry terminology, or the collision is explicitly disambiguated as in [Section 5.5](#55-the-bas-abbreviation-disambiguation-convention).
5. It does not cause renaming churn that the target architecture does not justify.

---

## 11. Community-Driven Objective and Principles

### 11.1 Objective

A long-term objective of Basitra is to become a community-driven open-source ecosystem: one whose use, extension, documentation, and eventual stewardship do not depend permanently on a single founder or a small team.

This is a design and governance objective, not a prediction. Consistent with [`writing-guidelines.md`](../standards/writing-guidelines.md#72-what-speculation-is-not-appropriate-in-this-repository) §7.2, this document makes no claim about adoption, contributor numbers, or scale.

Python's open-source community is used here only as an **inspiration for the community model**, not as a comparison of scale. The relevant lesson is that a durable ecosystem lets people other than its original authors:

- use the technology in environments its authors did not foresee;
- contribute implementations, integrations, and adapters;
- improve documentation and create educational material;
- identify new use cases and propose architectural improvements;
- maintain parts of the ecosystem and build complementary tooling;
- form groups around specialized domains;
- eventually take a meaningful part in governance.

### 11.2 Community-driven is not architecture by popularity

Basitra is a security project. Some areas remain governed by rigorous architecture review, validation, and security processes no matter how broad participation becomes:

- the `basis-core` isolation boundary;
- authorization and evaluation semantics;
- evidence integrity;
- execution boundaries;
- compatibility surfaces.

This is the existing position of [`GOVERNANCE.md`](../../GOVERNANCE.md): "Architectural Consistency," "Compatibility Protection," and "basis-core Boundary Protection" apply "regardless of who submits the contribution."

The intended shape is **broad extensibility and participation around a deliberately small and trustworthy security core**.

### 11.3 Principles

These principles restate and organize commitments that already exist. They do not add architecture. Their adoption as governance text is follow-on work ([Section 12.7](#127-stage-7--community-readiness)).

**P-1. Stable extension boundaries.** Prefer contribution paths that add capability without modifying the trusted authorization kernel. The architecture already provides the basis for this:

- Kernel isolation is protected by governance ([`kernel-boundary-rules.md`](../kernel-boundary-rules.md)).
- Higher-level services "may … extend its behavior at defined extension points" ([`basis-ecosystem.md`](basis-ecosystem.md#component-dependency-direction)).
- Roles are distinct from their implementations: "other conforming implementations remain possible" for the operation-producer role (ADR-0010), and a deployment "may substitute a conforming alternative for a given role where the architecture permits one" ([glossary](../glossary.md#basis-core-services-distribution)).
- ADR-0018 keeps upstream integrations as conforming *uses* of a generic boundary.

Candidate extension areas, **where architecture already supports them**, are:

- protocol adapters;
- identity-provider integrations, through `basis-identity`'s provider-neutral federation;
- conforming role implementations, such as producer or executor, subject to their ADRs;
- evidence consumers downstream of published contracts;
- deployment, observability, and developer tooling.

This document declares **no plugin API, extension SDK, or registration mechanism**. None exists, and defining one would be a compatibility-surface decision requiring an ADR.

**P-2. Contributor accessibility.** Architecture and documentation should be usable by several kinds of contributor:

- software engineers;
- OT and controls engineers;
- security engineers;
- integrators;
- facilities professionals;
- researchers;
- documentation contributors and educators;
- operators.

Valuable contributions include field knowledge, protocol behavior reports, operational constraints, and architecture feedback, not only code. [`CONTRIBUTING.md`](../../CONTRIBUTING.md) already lists "additional operational constraints or tradeoff analysis" and "threat-model refinements" as in scope.

**P-3. Governance evolution.** Governance should be able to move from founder-led stewardship toward distributed maintainership when project maturity and sustained participation justify it. No governance structure is created now. [`GOVERNANCE.md`](../../GOVERNANCE.md#current-governance-maturity) already anticipates "formal contribution roles, an RFC or proposal process for significant changes, … and organizational membership structures" as future additions.

**P-4. Transparent architectural evolution.** The ADR process remains the mechanism for architectural decisions. A cross-ecosystem proposal mechanism, similar in purpose to Python's PEP process, may become useful once proposals routinely span several repositories or come from outside contributors. "Basitra Enhancement Proposal" is one possible name, given only as an example. The existing open question is recorded in [`ROADMAP.md`](../../ROADMAP.md) Phase 5 ("Open governance maturity: … RFC process"), and this document does not resolve it.

---

## 12. Cross-Repository Adoption Roadmap

The steps below are called **Stages**, to avoid confusion with the Phases in [`ROADMAP.md`](../../ROADMAP.md), `basis-producer`'s implementation phases, and roadmap-document phases. Each stage after Stage 1 requires ADR-0019 to be `Accepted`, which it now is, and proceeds through bounded PRs. Stages are not a schedule.

Stage 4 is a **decision gate**. No final repository, package, or organization naming is assumed before it is passed, and Stage 5 depends on it.

### 12.1 Stage 1 — Architecture authority

Establish the strategy and terminology in `basis-architecture` without implementation changes. This document, [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md), a clearly marked proposed-terminology block in the [glossary](../glossary.md), and README discoverability links made up this stage. Stage 1 is complete: ADR-0019 has been accepted through its own review.

### 12.2 Stage 2 — Ecosystem inventory

This inventory is based on local checkouts inspected on 2026-09-27. Those checkouts may lag the remote default branches, so each row should be re-verified at the start of its Stage 3 PR. Nothing listed here has been changed. The "current naming" column is a starting point for Stage 4, not a statement that the current name is the target name.

| Repository / surface | Role | Current naming observed | Likely change categories (after acceptance) |
| - | - | - | - |
| `basis-architecture` | Architecture authority | BASIS expansion in `writing-guidelines.md` §3.3 and `SECURITY.md`; "BASIS ecosystem" umbrella; five-entity terminology rules | Acronym table (add, do not replace), `SECURITY.md` scope line, glossary promotion, terminology-rules entity section (with C-7), README introduction, `GOVERNANCE.md` relationship statement (after OD-1), empty `terminology-guidelines.md` (C-8); repository name reviewed in Stage 4 |
| `basis-core` | Authorization kernel | Distribution `basis-core` 0.2.1; import `basis_core`; has CONTRIBUTING | README project description, architecture links, contributor documentation, security policy (none observed), package metadata; repository, distribution, and namespace names reviewed in Stage 4 |
| `basis-gateway` | API and runtime wrapper | `basis-gateway` 0.2.0; `basis_gateway`; has CONTRIBUTING and SECURITY | README, documentation navigation, security policy wording, package metadata; names reviewed in Stage 4 |
| `basis-identity` | Identity engine | `basis-identity` 0.1.0; `basis_identity`; has CONTRIBUTING, SECURITY, CODE_OF_CONDUCT | README, security and conduct documents, package metadata; names reviewed in Stage 4 (closely tied to OD-8) |
| `basis-adapters` | Protocol normalization | `basis-adapters` 0.2.0; `basis_adapters`; has CONTRIBUTING, SECURITY, CODE_OF_CONDUCT | README, adapter contributor guide (extension-oriented), package metadata; names reviewed in Stage 4 |
| `basis-console` | Operator and admin UI | `basis-console` 0.2.0; `basis_console`; has CONTRIBUTING, SECURITY, CODE_OF_CONDUCT | README, in-product text that names the project (if any), package metadata; names reviewed in Stage 4 |
| `basis-schemas` | Shared contracts | `basis-schemas` 0.2.2; `basis_schemas`; has CONTRIBUTING | README, contract-governance wording, package metadata; names reviewed in Stage 4. Contract identifiers are a separate, compatibility-governed surface: any change follows the versioning and deprecation process, never branding alone. |
| `basis-producer` | Operation-producer runtime | `basis-producer` (local `pyproject` version 0.1.0); `basis_producer`; has CONTRIBUTING and SECURITY | README, package metadata; names reviewed in Stage 4. [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md) (Accepted) fixes this repository's name, so a rename needs a superseding ADR. |
| `basis-deploy` | Deployment tooling | Not established as a repository | Named under the Stage 4 outcome when created |
| `basis-poc` | Research proof of concept | README title "BASIS — Building Automation Secure Identity Service" | Historical artifact: at most, a pointer to current project identity; the title is **not** rewritten |
| `basis-foundation/basis` | Early predecessor artifact | "Building Automation Systems Identity Shield (BASis)" | Historical: an optional archival note only |
| `basis-lab` | Lab repository (initial commit only) | No project description observed | Named and described under the Stage 4 outcome when populated |
| BASIS website source | Public website | "BASIS (Building Automation Secure Identity Service) … under Basis Foundation" | Public identity text, navigation, and expansion, following the Section 5.5 convention; migration to a Basitra public surface is expected over time, subject to OD-1 and OD-7 |
| GitHub organization `basis-foundation` | Repository namespace | Organization name and repository descriptions | Repository description text in Stage 3; migration of the organization toward a Basitra identity is OD-7 and is **not authorized** by this document |
| Ipotio architecture (external) | Consumer ecosystem | Uses "Basitra" for BASIS components and roles | Cross-ecosystem reconciliation (C-6), in Ipotio's repository, after OD-8 and OD-9 |

### 12.3 Stage 3 — Current-documentation reconciliation

Plan one bounded PR per repository, in this order:

1. `basis-architecture`
2. `basis-schemas`
3. `basis-core`
4. the remaining components
5. public surfaces

Each PR updates only **current-state descriptive text**. It does not touch historical records, release notes, tags, changelogs, or accepted ADR bodies. Wording is set per repository role and must not presuppose the outcome of OD-8 or OD-9. An illustrative pattern, not mandated text:

> `basis-core` is the authorization kernel of the Basitra open-source ecosystem for Boundary-Aware Security, currently developed as part of the BASIS component family.

Each PR applies the [Section 5.5](#55-the-bas-abbreviation-disambiguation-convention) convention and does not introduce bare "BAS" for Boundary-Aware Security.

### 12.4 Stage 4 — Basitra target architecture and repository naming reconciliation

This stage is a decision gate. It determines the target names before any repository, package, or organization naming is treated as final. It works **architecture first, names second**:

1. **Inventory by architectural responsibility.** List each component and repository by what it owns and does not own, using [`basis-ecosystem.md`](basis-ecosystem.md), the accepted ADRs, and the kernel boundary rules. Current names are not an input to this step.
2. **Decide the long-term scope of BASIS within Basitra (OD-8).** BASIS may remain the name of the whole component family, become a narrower identity and security subsystem, or take another architecturally justified role.
3. **Choose names (OD-9).** For each component, answer: *If Basitra had been the project identity from the beginning, what should this component and repository be called based on its architectural responsibility?* A current name is kept if it still answers that question well; historical existence alone is not sufficient.
4. **Plan migrations.** For each name that changes, produce a bounded migration plan: repository redirects, documentation links, import-namespace compatibility and deprecation under [`compatibility-philosophy.md`](compatibility-philosophy.md), and any superseding ADR (for example, for [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md), which fixes the `basis-producer` name).
5. **Coordinate with OD-1 and OD-7.** Keep the organization-level identity and the repository namespace consistent.

The outcome is recorded by ADR. This stage renames nothing; renames are executed afterward as separate, planned PRs.

**Current state.** Step 1 is complete. [`basitra-target-architecture-discovery-assessment.md`](basitra-target-architecture-discovery-assessment.md) is the non-normative discovery assessment that inventories existing capabilities and roles by architectural responsibility and analyzes candidate shapes for OD-8. It remains the evidentiary record and is not revised to match the decision.

Step 2 is complete. [ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) (`Accepted`) resolves OD-8: BASIS names an architectural subsystem, the Boundary-Aware Security subsystem of Basitra that governs operations along the governed path, defined by roles rather than repositories. Under it:

- trust establishment, authorization (including the protocol-adapter role), operation and execution governance, and the evidence each owning role produces are inside BASIS;
- contract semantics are inside BASIS, and contract publication is a Basitra-level role;
- administrative interfaces are Basitra-level consumers of BASIS;
- architecture governance and deployment tooling are Basitra level;
- the BASIS Core Services Distribution is a broader, Foundation-maintained distribution, not a synonym for BASIS.

This document's descriptions of BASIS as the current component family, and its statements that BASIS's long-term scope is undecided, predate that acceptance. They are reconciled through the bounded current-documentation follow-on work that ADR-0024 lists, not here.

Step 3 is complete. The OD-9 naming assessment, [`basitra-repository-and-component-naming-assessment.md`](basitra-repository-and-component-naming-assessment.md), remains the non-normative evidentiary record and is not revised to match the decision. [ADR-0025](../adr/0025-basitra-repository-and-component-naming.md) (`Accepted`) is the normative decision and resolves OD-9. Repository and component target naming is settled: names record scope of concern (`basis-` for BASIS, `basitra-` for Basitra as a whole) without deciding BASIS membership, which ADR-0024 alone defines. No repository has been renamed, and no name is changed by the acceptance.

| Step | State |
| - | - |
| 1. Inventory by architectural responsibility | Complete |
| 2. OD-8, scope of BASIS | Complete ([ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md), `Accepted`) |
| 3. OD-9, repository and component naming | Complete ([ADR-0025](../adr/0025-basitra-repository-and-component-naming.md), `Accepted`) |
| 4. Migration planning | Next; follow-on work, not begun |
| 5. Coordination with OD-1 and OD-7 | Open, as applicable |

ADR-0025 acceptance closes the Stage 4 decision gate for repository and component naming. OD-10 and repository migration planning are now eligible as separate bounded workstreams; neither is performed here.

### 12.5 Stage 5 — Public artifact naming

After Stage 4, and before any package is published to a global registry, decide a consistent provenance and namespace convention (OD-10). Three naming surfaces must be kept distinct:

| Surface | Current | Examples only | Notes |
| - | - | - | - |
| Source repository | `basis-foundation/basis-core` and similar | Determined by Stage 4 (OD-7, OD-9) | No rename is authorized by this document |
| Distribution / registry name | `basis-core` (in `pyproject.toml`) | For illustration: `basitra-basis-core`, or a Basitra-prefixed name derived from the Stage 4 component name | Illustrations only, **not** a preferred convention. A `basitra-basis-*` pattern presupposes that BASIS remains the family name, which OD-8 has not decided. Registry availability has not been checked, and publication is not authorized. |
| Import / module namespace | `basis_core`, `basis_gateway`, and so on | Determined by Stage 4 and OD-10 | Renaming an import namespace is a breaking API change under [`compatibility-philosophy.md`](compatibility-philosophy.md). It may be justified by the target architecture, but it needs its own ADR, a deprecation path, and a compatibility plan. |

The decision should also cover non-Python artifacts, such as container images and schema bundles, when they exist.

### 12.6 Stage 6 — BASac formalization

Work through the gaps in [Section 9.3](#93-gaps-to-resolve-before-basac-can-become-a-formal-model). The expected sequence is:

1. the scope decision (gap 1), which may conclude that BASac is not needed;
2. then boundary-crossing representation and context-assertion trust;
3. then decision semantics, evidence, and conformance.

A normative BASac specification requires its own ADR and must not be introduced in a naming PR. This stage is independent of Stages 4 and 5 and may proceed in parallel.

### 12.7 Stage 7 — Community readiness

The checklist below lists future adoption targets. The "observed" column records what exists in local checkouts today.

| Artifact | Observed today | Target |
| - | - | - |
| `CONTRIBUTING.md` | Present in `basis-architecture` and all seven `basis-*` component repositories inspected | Ecosystem-level contributor guide linking per-repository guides |
| Code of Conduct | Present in `basis-identity`, `basis-adapters`, `basis-console`; absent in `basis-architecture`, `basis-core`, `basis-gateway`, `basis-schemas`, `basis-producer` | Consistent Code of Conduct across repositories |
| Security reporting policy | Present in `basis-architecture`, `basis-gateway`, `basis-identity`, `basis-adapters`, `basis-console`, `basis-producer`, `basis-poc`; absent in `basis-core`, `basis-schemas` | Consistent private reporting channel and scope statement |
| License copyright attribution | Open issue: unfilled placeholder ([`GOVERNANCE.md`](../../GOVERNANCE.md#license)) | Resolved before broader outside contribution, consistent with OD-1 |
| Extension-development guides | None | Per extension area, only where architecture supports it (P-1) |
| Contributor and maintainer expectations | Not defined | Defined when there are maintainers beyond the founder |
| Governance principles | `GOVERNANCE.md` (Foundation, lightweight) | Principle-level statement of P-3 after OD-1 |
| Roadmap visibility | `ROADMAP.md` and roadmap documents | Linked from the public project surface |
| Issue-label conventions, including beginner-friendly issues | Not defined | Consistent labels across repositories |
| Architecture proposal process | ADRs; RFC process is an open question | Decided when cross-repository proposals warrant it (P-4) |
| Community communication channels | Not defined | Chosen when there is a community to serve |
| Documentation for OT contributors who are not software engineers | Not present | Guide to contributing field knowledge, protocol behavior, and constraints (P-2) |

### 12.8 Stage 8 — Distributed ecosystem maturity

This stage is a long-term aspiration, not a present organizational claim. If participation grows and is sustained, it may eventually justify:

- specialized maintainers for components or protocol families;
- protocol or domain working groups;
- security or architecture working groups;
- broader project governance;
- independent, community-created extensions;
- stewardship structures that can survive changes in the original maintainers.

Each would be introduced by a governance decision when it is warranted, not in advance.

---

## 13. Open Decisions

| ID | Decision needed | Owner | Blocking? |
| - | - | - | - |
| **OD-1** | The eventual relationship between Basitra and the Basis Foundation: rename, succession, or coexistence. Basitra is the intended long-term canonical identity; permanent coexistence is not assumed. | Basis Foundation governance | Blocks the `GOVERNANCE.md` item in Stage 3 and informs OD-7. Does not block ADR-0019, which leaves the Foundation's current governing role unchanged. |
| **OD-2** | Whether BASAuth naming needs any adjustment given the BAS naming root | BASAuth (outside open-source governance) | Non-blocking; C-4's disclaimer suffices for the open-source record |
| **OD-3** | Cross-ecosystem terminology reconciliation with Ipotio (C-6) | Ipotio architecture, coordinated with this repository | Non-blocking for ADR-0019; should follow OD-8 and OD-9, both now resolved. Eligible, in Ipotio's repository, when sequencing permits; not yet performed. |
| **OD-4** | Whether BASac is needed at all (Section 9.3, gap 1) | Architecture review | Blocks any normative use of BASac |
| **OD-5** | ADR numbering convention: [`docs/adr/README.md`](../adr/README.md#numbering) says numbers are assigned at acceptance, but ADR-0007 through ADR-0018 were numbered when first proposed (as recorded in ADR-0018's Context). ADR-0019 follows the established practice. | Architecture maintainers | Non-blocking; a documentation reconciliation |
| **OD-6** | Whether the community principles in Section 11.3 should become governance text in `GOVERNANCE.md` and what form it takes | Basis Foundation governance | Non-blocking |
| **OD-7** | Whether, when, and how the `basis-foundation` GitHub organization migrates toward a Basitra identity, including redirects and the effect on repository URLs cited in accepted ADRs | Basis Foundation governance, with architecture review | Blocks any organization-level rename; depends on OD-1 |
| **OD-8** | The long-term architectural scope of BASIS within Basitra: the whole component family, a narrower identity and security subsystem, or another justified role | Architecture review (ADR) | **Resolved** in Stage 4 by [ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) (`Accepted`): BASIS is the Boundary-Aware Security subsystem of Basitra, defined by architectural roles rather than repositories. No longer blocks OD-9 or OD-10. |
| **OD-9** | Repository and component naming under the Basitra target architecture, per the Stage 4 evaluation question | Architecture review (ADR), with a superseding ADR where an accepted ADR fixes a name (for example, ADR-0010) | **Resolved** in Stage 4 by [ADR-0025](../adr/0025-basitra-repository-and-component-naming.md) (`Accepted`): a hybrid scope-of-concern convention; target `basitra-architecture` and, for the planned deployment repository, `basitra-deploy`; current BASIS-scoped names retained where accurate; and "the Basitra distribution" as the target human-facing distribution term. No repository is renamed by the acceptance; each rename needs a bounded migration plan. |
| **OD-10** | Package, distribution, and import-namespace conventions, including whether a `basitra-` prefix is used | Architecture review | Blocks registry publication naming. Open. Its architecture and repository-naming prerequisites are resolved (OD-8 by ADR-0024, OD-9 by ADR-0025), so it is now eligible for its own architecture work. Organization- and governance-dependent technical naming choices may still require coordination with OD-1 and OD-7, which remain open. |

---

## 14. Deferred Work

The following are deliberately **not** done by this document or ADR-0019:

- editing the writing-guidelines acronym table, `SECURITY.md`, the terminology-rules entity list, or any existing glossary entry;
- editing any existing use of "BAS," "BASIS ecosystem," or the BASIS expansion;
- renaming or migrating any repository, organization, distribution, import namespace, or public site;
- deciding the long-term scope of BASIS (OD-8), target repository names (OD-9), or package and namespace conventions (OD-10);
- publishing any package, or choosing final distribution names;
- creating `CODE_OF_CONDUCT.md`, governance bodies, working groups, or a proposal process;
- defining BASac, boundary-crossing context, or any plugin or extension API;
- changing any document or setting in Ipotio or any other repository.

---

## 15. References

- [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md): Basitra ecosystem identity and terminology hierarchy (Accepted)
- [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md)
- [`docs/standards/terminology-rules.md`](../standards/terminology-rules.md)
- [`docs/standards/writing-guidelines.md`](../standards/writing-guidelines.md)
- [`docs/glossary.md`](../glossary.md)
- [`docs/architecture-principles.md`](../architecture-principles.md)
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md)
- [`docs/security/threat-model.md`](../security/threat-model.md)
- [`docs/architecture/operation-aware-authorization-model.md`](operation-aware-authorization-model.md)
- [`docs/architecture/operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md)
- [`docs/architecture/compatibility-philosophy.md`](compatibility-philosophy.md)
- [ADR-0007](../adr/0007-adapter-evidence-construction.md) through [ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md)
- White paper [Section 05 — OT Trust Boundaries](../../whitepapers/identity-aware-authorization-for-operational-technology/sections/05-ot-trust-boundaries.md) and [Section 09 — Future Direction](../../whitepapers/identity-aware-authorization-for-operational-technology/sections/09-future-direction.md)
- [`GOVERNANCE.md`](../../GOVERNANCE.md), [`CONTRIBUTING.md`](../../CONTRIBUTING.md), [`ROADMAP.md`](../../ROADMAP.md), [`SECURITY.md`](../../SECURITY.md)
- ISA/IEC 62443-1-1 (zones and conduits), as cited in [`references/sources.md`](../../whitepapers/identity-aware-authorization-for-operational-technology/references/sources.md)
