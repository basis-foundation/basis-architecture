# Basitra Repository and Component Naming Assessment

> **Status: non-normative naming assessment (OD-9).** This assessment informs the future OD-9 naming decision. It does not rename any repository, component, package, namespace, organization, distribution, website, or published artifact.
>
> Recommendations in this document are non-normative until adopted by an OD-9 decision.

It applies the Stage 4 naming question in the [strategy document](basitra-ecosystem-and-boundary-aware-security.md#124-stage-4--basitra-target-architecture-and-repository-naming-reconciliation) to every current repository and naming concept, using the boundary fixed by [ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) (`Accepted`). It creates no ADR, resolves none of OD-1, OD-7, OD-9, or OD-10, supersedes no accepted decision (including [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md)), and modifies no implementation repository, contract, or schema.

---

## Contents

1. [Purpose and Question](#1-purpose-and-question)
2. [Evidence Rules and Assessment Labels](#2-evidence-rules-and-assessment-labels)
3. [Fixed Inputs](#3-fixed-inputs)
4. [Repositories Versus Roles](#4-repositories-versus-roles)
5. [Evaluation Criteria and the Three Readings of `basis-`](#5-evaluation-criteria-and-the-three-readings-of-basis-)
6. [Repository Assessments](#6-repository-assessments)
7. [Naming Concepts Beyond Repositories](#7-naming-concepts-beyond-repositories)
8. [Current Naming Versus Target Architecture](#8-current-naming-versus-target-architecture)
9. [ADR-0010 Naming Constraint](#9-adr-0010-naming-constraint)
10. [Compatibility Surfaces](#10-compatibility-surfaces)
11. [Naming Coherence Across the Ecosystem](#11-naming-coherence-across-the-ecosystem)
12. [Candidate Convention Analysis](#12-candidate-convention-analysis)
13. [Recommendation to the Future OD-9 Decision (Non-Normative)](#13-recommendation-to-the-future-od-9-decision-non-normative)
14. [Decision Quality Checks](#14-decision-quality-checks)
15. [What This Assessment Does Not Decide](#15-what-this-assessment-does-not-decide)
16. [References](#16-references)

---

## 1. Purpose and Question

[ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) resolved OD-8. BASIS is the Boundary-Aware Security subsystem of Basitra. Its membership attaches to roles, not to repositories, and is decided by the path, security, and ownership conditions of its [Decision 2](../adr/0024-long-term-scope-of-basis-within-basitra.md#2-membership-rule). Its [Placement Summary](../adr/0024-long-term-scope-of-basis-within-basitra.md#placement-summary) places every current role. Its Follow-On Work item 2 hands the next question to OD-9:

> Given the architecture now accepted by ADR-0024, does the current name still accurately describe the architectural responsibility of this repository or component?

The assessment answers that question in two separate steps for each repository and concept:

1. **Target question.** If Basitra had been the project identity from the beginning, and ADR-0024's architecture had already existed, what would this repository or component be called based on the responsibility it owns?
2. **Retention question.** Does the current name already answer the target question well enough to keep?

Neither step treats the existence of a current name as evidence that it should remain ([ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) Decision 7). Neither treats change as desirable in itself. Historical continuity informs migration, but it does not constrain the target architecture.

The aim is completeness. A future OD-9 ADR should be able to make its naming decision from this document without another repository inventory ([Section 14](#14-decision-quality-checks), check 10).

---

## 2. Evidence Rules and Assessment Labels

### 2.1 Evidence

- **Architecture evidence** comes from accepted ADRs and canonical architecture documents in this repository. ADR-0024's Placement Summary is the normative starting point for every placement in this document. No placement is re-derived or changed here.
- **Implementation evidence** comes from local checkouts of the `basis-foundation` repositories, inspected on 2026-10-06: `pyproject.toml` names and versions, import packages, README descriptions, CI workflow files, cross-repository dependency declarations, and cross-repository links. Local checkouts may lag the remote default branches, so each observation should be re-verified before any migration PR.
- **Current names are evidence of where first-party code lives**, not of where architectural boundaries are ([ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) Context). No placement in this document follows from a name.

### 2.2 Provisional outcome labels

| Label | Meaning in this assessment |
| - | - |
| **KEEP** | The current name answers the target question well. Evidence for retention is strong. |
| **LIKELY KEEP** | The current name answers the target question adequately. A credible alternative exists, but the evidence does not justify the change. |
| **RENAME RECOMMENDED** | The current name answers the target question materially worse than an available alternative, and the benefit plausibly exceeds the cost. |
| **RENAME REQUIRED BY ARCHITECTURE** | Accepted architecture itself states or entails that the current name is inaccurate. Used only where accepted text supports it directly. |
| **RETIRE / ARCHIVE** | A historical artifact. Its name is retained as history and not migrated. Any treatment is archival, not a rename. |
| **DEFER** | Not enough evidence exists, or the answer depends on a decision outside OD-9. |

These are assessment labels, not naming decisions.

### 2.3 Confidence

| Level | Meaning |
| - | - |
| **High** | The recommendation follows directly from accepted architecture and observed evidence. A reasonable reviewer is unlikely to reach a different outcome. |
| **Medium** | The recommendation is the better-supported outcome, but it depends on a judgment, often the naming convention ([Section 12](#12-candidate-convention-analysis)), that a reviewer could reasonably make differently. |
| **Low** | The evidence is thin or the outcome depends materially on an open decision. The reason is stated in each case. |

Where the direction of a recommendation and the specific candidate name have different confidence, both are given.

---

## 3. Fixed Inputs

The following are taken as fixed. This assessment does not reopen any of them.

| Input | What it fixes | Source |
| - | - | - |
| Basitra | Project, community, and ecosystem identity. It names no runtime component. | [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) Decision 1; strategy document [§8](basitra-ecosystem-and-boundary-aware-security.md#basitra) |
| BASIS | The Boundary-Aware Security subsystem of Basitra, defined by roles. It does not name a repository prefix, a set of repositories, a distribution, the Foundation, the GitHub organization, or `basis-core` alone. | [ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) Decisions 1 and 7 |
| Membership test | Path, Security, Ownership | ADR-0024 Decision 2 |
| Role placements | Inside BASIS: identity engine, subject establishment, workload admission, protocol adapter, composition, context admission, authorization kernel, enforcement, producer intake, operation producer, binding, protocol executor, lifecycle observation, execution evidence, evidence obligations, BASIS-governed configuration, contract semantics. Basitra level, outside BASIS: architecture and governance, contract publication, administrative interfaces, deployment and distribution tooling. External: upstream supervisory platforms (such as Ipotio or Niagara), enterprise identity providers, OT targets, external evidence consumers, BASAuth. | ADR-0024 Decisions 3–6 and [Placement Summary](../adr/0024-long-term-scope-of-basis-within-basitra.md#placement-summary) |
| Distribution | *BASIS Core Services Distribution* is current canonical terminology, does not correspond to BASIS, and its name is transitional | ADR-0024 [Decision 8](../adr/0024-long-term-scope-of-basis-within-basitra.md#8-basis-and-the-current-distribution-terminology) |
| Reading rule | "BASIS" used for responsibilities, roles, or sides means the subsystem. "BASIS" used as a family, distribution, namespace, or naming label denotes current naming. | ADR-0024 [Decision 12](../adr/0024-long-term-scope-of-basis-within-basitra.md#12-continuity-with-accepted-adrs) |
| `basis-producer` | Fixed by an accepted ADR as the permanent repository and component name | [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md); [Section 9](#9-adr-0010-naming-constraint) |
| Naming principles | Responsibility-first names; no acronym-driven names; no forced prefix; technology durability; history stays accurate | strategy document [§10](basitra-ecosystem-and-boundary-aware-security.md#10-naming-and-branding-principles); ADR-0019 Decisions 6–7 |

Three decisions outside OD-9 limit what this assessment can conclude:

- **OD-1** governs the long-term relationship between Basitra and the Basis Foundation.
- **OD-7** governs migration of the `basis-foundation` GitHub organization.
- **OD-10** governs package, distribution, and import-namespace conventions, and the registry artifact names that follow from them.

Where a recommendation depends on one of them, the dependency is recorded and the decision is not made ([strategy document §13](basitra-ecosystem-and-boundary-aware-security.md#13-open-decisions)).

---

## 4. Repositories Versus Roles

ADR-0024 makes the role the durable architectural unit. A repository may host several roles, and its name may describe only some of them. The table lists, for each current repository, every role it hosts under accepted architecture and where each role sits.

| Repository | Roles hosted under accepted architecture | Membership of those roles | Does the name cover the hosted roles? |
| - | - | - | - |
| `basis-architecture` | Architecture and governance. It decides BASIS contract semantics and every Basitra-level matter, including the boundary with external platforms. | Basitra level, outside BASIS | No. The name scopes the repository to BASIS. Its governed scope is Basitra. |
| `basis-core` | Authorization-kernel role; adapter interface contracts | BASIS | Partly. "Core" describes the kernel's centrality, not its responsibility. |
| `basis-gateway` | Subject establishment; producer admission; composition; context admission (its part); enforcement; authorization audit evidence | All BASIS | Yes, as an implementation term: the network trust-boundary runtime that hosts these roles. |
| `basis-identity` | Identity-engine role (federation and canonical identity) | BASIS | Yes |
| `basis-adapters` | Protocol-adapter role, implemented for several protocol families | BASIS | Yes |
| `basis-console` | Administrative-interface role (administration, inspection, diagnostics, labeled simulation) | Basitra level, consumer of BASIS | Yes for its responsibility. Whether its prefix implies membership is a convention question ([Section 5](#5-evaluation-criteria-and-the-three-readings-of-basis-)). |
| `basis-schemas` | Contract publication. It publishes BASIS contract semantics decided in `basis-architecture`. | Publication: Basitra level. Published semantics: BASIS. | Mostly. It publishes contracts, of which schemas are one part. |
| `basis-producer` | Operation-producer role, including binding creation (ADR-0012). Producer-intake boundary, as the producer role's ingress (ADR-0018). Protocol-executor and execution-evidence-producer roles, colocated for the first bounded slice (ADR-0011 Decision 3; ADR-0014; ADR-0015). | All BASIS | Partly. It names the permanent role. It does not name the roles colocated for the first slice. |
| `basis-deploy` (planned) | Deployment and distribution tooling | Basitra level, outside BASIS | Not established. The planned name scopes the tool to BASIS. Its responsibility packages BASIS and non-BASIS components. |
| `basis-lab` | None | — | No role to describe ([Section 6.10](#610-basis-lab)) |
| `basis-poc` | None (historical research artifact) | — | Historical |
| `basis-foundation/basis` | None (historical predecessor artifact) | — | Historical |

Four cases need explicit attention, because the repository's name does not match the full set of responsibilities it hosts:

1. **`basis-producer`** hosts four BASIS roles in its first execution slice and is named for one of them ([Section 6.8](#68-basis-producer)).
2. **`basis-schemas`** hosts a Basitra-level role, publication, whose entire content is BASIS contract semantics ([Section 6.7](#67-basis-schemas)).
3. **`basis-console`** hosts a Basitra-level role whose entire object is BASIS ([Section 6.6](#66-basis-console)).
4. **`basis-architecture`** hosts a Basitra-level role whose object is all of Basitra, not only BASIS ([Section 6.1](#61-basis-architecture)).

Cases 2, 3, and 4 have the same structure. Each is a Basitra-level role outside BASIS. What separates them is the **scope of concern** of each role: whether it concerns BASIS specifically or Basitra as a whole. Section 5 explains why that distinction matters for names.

---

## 5. Evaluation Criteria and the Three Readings of `basis-`

### 5.1 Criteria

Each repository is evaluated against eight criteria:

| # | Criterion | Question |
| - | - | - |
| C1 | Architectural accuracy | Does the name describe the responsibility actually owned? |
| C2 | ADR-0024 membership accuracy | If the name begins with `basis-`, does it imply BASIS membership accurately, or does it at least not imply it falsely? |
| C3 | Role breadth | Does the name describe only one of several hosted roles? |
| C4 | Future durability | Does the name survive splitting, combining, another protocol, a third-party implementation, and topology change? |
| C5 | User comprehension | Would a new contributor infer correctly what the repository owns? |
| C6 | Ecosystem clarity | Does the name clarify or blur the Basitra/BASIS distinction? |
| C7 | Historical continuity | What recognition and compatibility benefit comes from keeping it? |
| C8 | Rename cost | How many links, artifacts, dependencies, and automation references would move? Package and import names are recorded separately as OD-10 concerns ([Section 10](#10-compatibility-surfaces)). |

### 5.2 Three readings of a `basis-` prefix

Criterion C2 cannot be applied until it is clear what a `basis-` prefix claims. Current usage supports three readings:

| Reading | A `basis-` prefix means | Status |
| - | - | - |
| **R1: Family label** | The repository belongs to the first-party `basis-*` component family | Current naming. ADR-0024 Decision 8 keeps its meaning until naming work changes it, and states that it does not define membership. |
| **R2: Scope of concern** | The repository's responsibility concerns BASIS: it implements a BASIS role, or it exists to serve, publish, or administer BASIS specifically | Supported by accepted text. ADR-0024 Decision 12 reads "BASIS administrative interface" and "BASIS user interface" (ADR-0023) as naming "what is administered, not subsystem membership." |
| **R3: Subsystem membership** | The repository implements a BASIS role, and nothing else | A possible future convention. It does not describe current usage. ADR-0024 Decision 1 states that BASIS does not name "a repository prefix or a set of repositories." |

The reading matters for three repositories:

- `basis-console` and `basis-schemas` are accurate under R1 and R2 and inaccurate under R3.
- `basis-architecture` and the planned `basis-deploy` are accurate only under R1. Their scope of concern is Basitra as a whole, so they are inaccurate under R2 and R3.
- Every other current repository is accurate under all three readings.

R1 is the legacy reading that ADR-0024 makes transitional, so OD-9 effectively chooses between R2 and R3. That choice is the main judgment in this assessment. It is analyzed in [Sections 11](#11-naming-coherence-across-the-ecosystem) and [12](#12-candidate-convention-analysis). The per-repository assessments in Section 6 state their outcome under each reading where it differs.

### 5.3 Constraints on candidate names

Candidate names follow the constraints in strategy document [§10.1](basitra-ecosystem-and-boundary-aware-security.md#101-principles) and §10.2:

- no forced `basitra-` prefix and no forced `basis-` prefix;
- no acronym-driven names (principle 10). No candidate is built to fit BAS, BASIS, or BASac;
- responsibility first;
- no technology-bound names: no Python, FastAPI, identity-provider, OT-protocol, or topology-specific terms;
- no new sub-brands. Candidates use plain descriptive words.

ADR-0019 adds one constraint: Basitra names no component. A `basitra-` prefix on a repository scopes it to the ecosystem. It does not make Basitra the name of a component, but a candidate that would read as "the Basitra" runtime is avoided.

---

## 6. Repository Assessments

Each subsection gives the evidence, the responsibility under ADR-0024, the criteria, the provisional outcome, the candidates, and the confidence.

### 6.1 `basis-architecture`

**Evidence.** This repository holds the architecture white papers, threat model, standards, every ADR, and [`GOVERNANCE.md`](../../GOVERNANCE.md). Its current content is not limited to BASIS. It includes the strategy document, which defines Basitra itself, and ADR-0019 and ADR-0024, which decide what Basitra and BASIS are. It also includes the Basitra–Ipotio roadmap ownership assessment, decisions about Basitra-level roles that ADR-0024 places outside BASIS (contract publication, administrative interfaces, deployment), and the boundary with external platforms. At least 35 files in other local `basis-*` checkouts link to it by URL. It is the most-linked repository in the ecosystem.

**Responsibility under ADR-0024.** Architecture and governance: Basitra level, outside BASIS. It "decides BASIS, among other things" (ADR-0024 decision diagram).

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Inaccurate in scope. The responsibility is Basitra's architecture, of which BASIS is the central part. |
| C2 Membership | Inaccurate under R2 and R3. Its scope of concern is the whole ecosystem, and the role is outside BASIS. Accurate only under R1. |
| C3 Breadth | The name covers one subject of the repository (BASIS) and omits the others (Basitra-level roles, ecosystem identity, governance). |
| C4 Durability | Weak. Every future Basitra-level decision, such as conformance criteria, community governance, or non-BASIS components, would land in a repository whose name says it is about BASIS. |
| C5 Comprehension | A new contributor would reasonably look elsewhere for Basitra-level architecture. |
| C6 Ecosystem clarity | It blurs the distinction most visibly, because the authority that defines the Basitra/BASIS boundary is itself named for one side of it. |
| C7 Continuity | High recognition: it is the canonical citation target for every ADR. |
| C8 Cost | High link volume: the cross-repository links, the links in accepted ADR bodies that cannot be edited, README badges, and links in the white paper. Repository-name redirects mitigate most of it, but a migration plan must verify that ([Section 10](#10-compatibility-surfaces)). No package, import namespace, or registry artifact depends on this name. |

**Provisional outcome: RENAME RECOMMENDED.**

**Candidates.**

- **`basitra-architecture`**, the leading candidate. It states the scope and the responsibility, and it works under either OD-7 outcome. In a Basitra-named organization the repeated "basitra" is tolerable and common practice. In the current organization, `basis-foundation/basitra-architecture` reads as the Foundation stewarding Basitra's architecture, which is today's governance arrangement.
- **`architecture`**, which is viable only inside a Basitra-named organization (OD-7). Outside that context the name is generic: it is weak in search and in forks, and it collides with similarly named repositories elsewhere. It should not be adopted while the organization is `basis-foundation`.

**Confidence.** High that the current name misstates the repository's scope. Medium on the target string, because the choice between the two candidates depends on OD-7.

**Not RENAME REQUIRED BY ARCHITECTURE**, because no accepted text states that the name is inaccurate, and the repository does contain BASIS architecture. The stronger label would overreach.

**Dependencies.** OD-7 decides the final string. OD-1 is touched only through `GOVERNANCE.md` content, not through the repository name.

### 6.2 `basis-core`

**Evidence.** Distribution `basis-core` 0.2.1 and import package `basis_core`. `basis-gateway` and `basis-producer` both depend on `basis-core>=0.2.1,<0.3.0`. Its README calls it the "authorization kernel." It has at least 20 cross-repository file links. [`kernel-boundary-rules.md`](../kernel-boundary-rules.md) states import rules using `basis_core.*`. [`terminology-rules.md`](../standards/terminology-rules.md#component-naming-conventions) already accepts "the kernel" as shorthand for `basis-core`. ADR-0019 and ADR-0024 Decision 1 both state that BASIS is not `basis-core` alone.

**Responsibility under ADR-0024.** Authorization-kernel role and adapter interface contracts. Inside BASIS. It is the sole source of the authorization outcome.

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Indirect. "Core" names a position, not a responsibility. Read as "the core of BASIS" it is defensible: the kernel is the one role every governed decision depends on, and the only role with no substitute on the governed path. |
| C2 Membership | Accurate under every reading. |
| C3 Breadth | One role, so no breadth problem. |
| C4 Durability | Good, as long as the prefix stays `basis-`. Under a `basitra-` prefix, "core" would read as the core of the whole ecosystem, which would be misleading ([Section 12](#12-candidate-convention-analysis), Convention B). |
| C5 Comprehension | Mixed. Contributors may infer that `basis-core` *is* BASIS, the confusion ADR-0019 and ADR-0024 both rule out. The README and terminology rules already correct this. |
| C6 Ecosystem clarity | Neutral to slightly blurring. |
| C7 Continuity | The highest of any component: the oldest released package, the explicit successor to the PoC (README), and cited throughout the white paper. |
| C8 Cost | The highest of any component repository at the repository level: at least 20 cross-repository file links, white-paper citations, and the strongest name recognition. A repository rename would not by itself change the `basis-core` distribution, the `basis_core` import namespace, the two downstream dependency specifiers, or the kernel import rules. Those are OD-10 surfaces and change only through a separate OD-10 decision. If they stayed unchanged, the most widely consumed package would no longer match its repository's name. |

**Provisional outcome: LIKELY KEEP.**

**Candidates, if OD-9 decides otherwise.**

- `basis-kernel`, which is role-explicit and matches the accepted "the kernel" shorthand.
- `authorization-kernel`, which is the Convention C form.

Neither improves accuracy enough to justify the cost at this time.

**Confidence.** Medium. The ambiguity of "core" is real and documented (ADR-0019 had to state that BASIS is not `basis-core`). Retention rests on the prefix staying `basis-` and on the cost evidence, not on the name being ideal.

**Dependency.** If OD-9 or OD-10 adopts a `basitra-` prefix for implementation repositories, this outcome flips to RENAME RECOMMENDED, because `basitra-core` would misstate the role.

### 6.3 `basis-gateway`

**Evidence.** Distribution `basis-gateway` 0.2.0 and import package `basis_gateway`. Its README describes "the authenticated HTTP enforcement boundary for BASIS." [`basis-gateway.md`](basis-gateway.md) calls it the "reference enforcement API and trust-boundary runtime." It hosts subject establishment, producer admission, composition, its part of context admission, enforcement, and authorization audit evidence (ADR-0024 Placement Summary). It also owns the reserved `basis_gateway.*` evidence namespace ([`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md)). In the embedded model, ADR-0024 Decision 3 assigns the enforcement role to the adapter host, not to the gateway.

**Responsibility under ADR-0024.** Several BASIS roles, all realized at one networked trust boundary in front of the kernel library.

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Accurate as an implementation term. "Gateway" describes what this implementation is: the network entry boundary at which these roles are realized. |
| C2 Membership | Accurate under every reading. |
| C3 Breadth | Good. "Gateway" covers all six hosted roles. A role name such as "enforcement" would cover only one of them. |
| C4 Durability | Good. The gateway is the topology that defines this implementation. If enforcement is realized in another topology, such as the embedded model, it is realized by a different implementation (the adapter host), not by renaming this one. More BASIS responsibilities at the same boundary would not make the name misleading. |
| C5 Comprehension | Good within the ecosystem. One domain-specific risk: in OT, "gateway" commonly means a protocol-conversion gateway (for example, a BACnet/IP gateway). The prefix and the README carry the disambiguation. |
| C6 Ecosystem clarity | Neutral |
| C7 Continuity | High: a released package, demos, and an established deployment vocabulary (trusted NGINX ingress to `basis-gateway`, ADR-0009). |
| C8 Cost | High. It has at least 11 cross-repository file links and demos, and every accepted ADR from ADR-0008 to ADR-0023 refers to it by name. A repository rename would not by itself change the released package, the `basis_gateway` import namespace, or the `basis_gateway.*` evidence namespace (OD-10 and contract governance). If they stayed unchanged, they would no longer match the repository name. |

**"Enforcement" was evaluated and is not preferred.** It names a role, not an implementation. Naming this repository after one role would understate the five others it hosts. It would also suggest that enforcement exists only here, when the embedded model places it elsewhere.

**Provisional outcome: LIKELY KEEP.**

**Candidate, if OD-9 decides otherwise:** `basis-enforcement`. Not recommended, for the reasons above.

**Confidence.** Medium. The name is durable and broad enough. The residual uncertainty is the OT-domain ambiguity of "gateway", which documentation can manage.

### 6.4 `basis-identity`

**Evidence.** Distribution `basis-identity` 0.1.0 and import package `basis_identity`. Its README describes "the BASIS identity engine and federation boundary." [`basis-identity.md`](basis-identity.md) and [`identity-authority-modes.md`](identity-authority-modes.md) describe the role. No local checkout links to it by URL from another repository.

**Responsibility under ADR-0024.** Identity-engine role, inside BASIS. It is optional where the enforcement role verifies identity directly.

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Accurate. The repository implements the identity engine. |
| C2 Membership | Accurate under every reading. |
| C3 Breadth | One role. One caution: "identity" does not own *every* identity responsibility in BASIS. Subject establishment and workload admission belong to the enforcement role ([Section 4](#4-repositories-versus-roles)). The name does not claim them, and the README is precise. |
| C4 Durability | Good. It is not tied to an identity provider or protocol, and a federated or native mode change does not affect it. |
| C5 Comprehension | Good |
| C6 Ecosystem clarity | Good. It aligns with the broad reading of *Identity* in the BASIS expansion (ADR-0024 Decision 9) without depending on it. |
| C7 Continuity | Moderate |
| C8 Cost | Low. No cross-repository file links were observed. The released package (0.1.0) and import namespace are OD-10 surfaces and would not change with a repository rename. |

**Provisional outcome: KEEP.** **Candidates:** none needed. **Confidence:** High.

### 6.5 `basis-adapters`

**Evidence.** Distribution `basis-adapters` 0.2.0 and import package `basis_adapters`. It normalizes nine protocol families. `basis-producer` pins `basis-adapters==0.2.0`. Its README: "Protocol adapters for the BASIS ecosystem." Published contract examples in `basis-schemas` carry `adapter_source` values that embed the repository name, such as `basis-adapters:bacnet` and `basis-adapters:modbus`. Strategy document §11.3 P-1 lists adapters as the first community extension area.

**Responsibility under ADR-0024.** Protocol-adapter role, inside BASIS (Decision 5). Specific adapters are implementations of that role. Third-party adapters implement the same role without becoming Foundation-maintained (Decision 7).

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Accurate. The plural fits a first-party library holding several protocol-family implementations. |
| C2 Membership | Accurate under every reading. Extensibility does not affect this: it comes from the role-versus-implementation distinction, not from placing the role outside BASIS (Decision 5). |
| C3 Breadth | One role, many protocol families. The plural covers both. |
| C4 Durability | Good. Another protocol needs no rename. A third-party adapter repository can use any name without contradicting this one, because the first-party name claims no exclusivity. If first-party adapters were later split per protocol, the natural pattern would be derived names (one per family), not a rename of this one. |
| C5 Comprehension | Good. "protocol-adapters" would be marginally more explicit, but in this ecosystem "adapter" already means a protocol adapter. |
| C6 Ecosystem clarity | Good |
| C7 Continuity | Moderate to high: a released package, an extension-oriented contributor guide, and community-facing adapter vocabulary. |
| C8 Cost | Moderate. It has at least 8 cross-repository file links. A repository rename would not by itself change the package, `basis-producer`'s exact-pinned dependency on it, or the `adapter_source` evidence values that embed its name (OD-10 and contract governance, [Section 10](#10-compatibility-surfaces)). If those values stayed unchanged, they would name the repository by its former name. |

**Provisional outcome: KEEP.**

**Candidate, if OD-9 decides otherwise:** `basis-protocol-adapters`. Not recommended: it is redundant in context.

**Confidence.** High.

### 6.6 `basis-console`

**Evidence.** Distribution `basis-console` 0.2.0 and import package `basis_console`. Its package description: "Human-facing, gateway-first operational interface for the BASIS ecosystem." [`basis-console.md`](basis-console.md) describes it as "an operator-facing interface layer for BASIS itself," which "operates and administers BASIS" and "does not operate OT equipment." No local checkout links to it by URL from another repository.

**Responsibility under ADR-0024.** Administrative-interface role: a Basitra-level consumer of BASIS, outside BASIS (Decision 6).

**What does it administer?** Every runtime concern that an administrative interface can act on today is BASIS state: BASIS-governed configuration, the administrative context, and authorization and execution evidence. No non-BASIS runtime exists in Basitra for a console to administer. Deployment tooling is not established, and contract publication and governance are not runtime concerns. The console's object is therefore BASIS, and no evidence suggests it will extend to broader Basitra concerns. If it did, for example by administering deployment or distribution, its scope of concern would become Basitra and the answer below would change.

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Accurate. "Console" describes the administrative interface. Under R2, "basis" identifies what it administers. |
| C2 Membership | Accurate under R2. ADR-0024 Decision 12 explicitly reads "BASIS administrative interface" as naming "what is administered, not subsystem membership." Inaccurate under R3, which would claim BASIS membership that Decision 6 denies. |
| C3 Breadth | One role |
| C4 Durability | Good while the console administers BASIS only. Weak if its scope extends to Basitra-level concerns. |
| C5 Comprehension | A new contributor would correctly infer "the console for BASIS." They might also incorrectly infer that it is a BASIS component. That inference has no security consequence, because membership confers no authority (ADR-0024 Decision 2), but it is a misreading that documentation has to correct. |
| C6 Ecosystem clarity | Neutral under R2. Blurring under R3. |
| C7 Continuity | Moderate: a released package. |
| C8 Cost | Low. No cross-repository file links were observed. The released package (0.2.0) and import namespace are OD-10 surfaces and would not change with a repository rename. |

**Provisional outcome: LIKELY KEEP under the recommended convention (R2). RENAME RECOMMENDED if OD-9 adopts R3.**

**Candidates, only if R3 is adopted or the console's scope expands to Basitra-level concerns:**

- `basitra-console`;
- `console`, which is viable only within a Basitra-named organization (OD-7).

`basitra-admin` was considered and is less accurate. The repository also hosts inspection, diagnostics, and labeled simulation, which "admin" does not cover.

**Confidence.** Medium. The outcome is directly convention-dependent ([Section 12](#12-candidate-convention-analysis)). Under R2 the evidence for retention is strong, and ADR-0024's own reading rule supports it.

### 6.7 `basis-schemas`

**Evidence.** Distribution `basis-schemas` 0.2.2 and import package `basis_schemas`. The repository publishes contract files with `contract:` metadata, lifecycle states, examples, compatibility fixtures, and per-contract documentation. Examples: `decision-request`, `decision-response`, `audit-event`, `audit-evidence`, `adapter-evidence-reference`, `operation-aware-decision-request`, `evaluation-trace`, `gateway-audit-event`, and `action-string`. Each file states "Architecture governs; schemas publish; implementations consume." Its README calls it "the neutral home for the shared contracts of the BASIS ecosystem." Its charter is [`basis-schemas.md`](basis-schemas.md). Contract identifiers (for example `decision-request`) do not carry the repository name. The reserved `basis_gateway.*` evidence namespace and the example `adapter_source` values do carry component names.

**Responsibility under ADR-0024.** Contract publication: a Basitra-level role, outside BASIS. Every contract it publishes defines a crossing of a BASIS boundary, so its content is BASIS contract semantics (Decision 4). One exception: `contract-metadata` describes the publication format itself, not BASIS semantics.

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Mostly accurate. The repository publishes *contracts*. Machine-readable schemas are one component of a contract, beside lifecycle, compatibility fixtures, examples, and semantic references. "Contracts" would describe the durable responsibility more exactly. "Schemas" describes the most visible artifact. |
| C2 Membership | Accurate under R2: everything it publishes, except its own metadata, is BASIS contract semantics. Inaccurate under R3, because publication is outside BASIS. Publication's neutrality (Decision 4; `basis-schemas.md` §2) is neutrality toward *implementations*, so that no single role implementation is the de facto contract authority. A `basis-` prefix does not compromise it. A third-party implementation of a BASIS role expects to find BASIS contracts under a BASIS-scoped name. |
| C3 Breadth | One role |
| C4 Durability | Good while publication covers only BASIS contracts. If publication extends to non-BASIS contracts, such as deployment or distribution manifests or future conformance criteria, its scope of concern becomes Basitra and the prefix becomes inaccurate under R2 as well. |
| C5 Comprehension | Good: "BASIS's schemas" is what a contributor would infer, and it is correct. |
| C6 Ecosystem clarity | Neutral under R2. Blurring under R3. |
| C7 Continuity | Moderate to high: the most frequently released package (0.2.2), and the charter vocabulary "architecture proposes, schemas publish" is established. |
| C8 Cost | Moderate. The repository rename itself is modest (at least 3 cross-repository file links). The package, the contract identifiers, and the reserved namespaces are OD-10 and contract-governance surfaces that a rename must not silently move ([Section 10](#10-compatibility-surfaces)). |

**Schema versus contract.** "Contract" is the more accurate word for the durable responsibility. The difference alone does not justify a rename: the inaccuracy is small, and "schemas" is established in the charter, the threat model, and every component's dependency language. If OD-9 renames this repository for another reason, such as adopting R3 or a scope change, "contracts" should replace "schemas" in the same rename rather than in a second one.

**Provisional outcome: LIKELY KEEP under the recommended convention (R2). RENAME RECOMMENDED if OD-9 adopts R3, or if publication extends to non-BASIS contracts.**

**Candidates, if renamed:**

- `basitra-contracts` (scope-accurate under R3, responsibility-accurate);
- `basis-contracts` (if only the schema/contract accuracy were addressed under R2, which is not recommended on its own);
- `contracts`, which is viable only within a Basitra-named organization (OD-7).

**Confidence.** Medium. It is convention-dependent, like `basis-console`. The schema-versus-contract question is a real but secondary accuracy gain.

### 6.8 `basis-producer`

**Evidence.** Distribution `basis-producer` 0.1.0 (not released) and import package `basis_producer`, reserved by ADR-0010. Phases 2A–5 of the bounded producer slice are implemented. The local checkout also contains `authorization_execution_binding.py`, from Work Item 1 of the [bounded REST execution plan](bounded-rest-execution-implementation-plan.md), which is binding *creation*, a producer-role responsibility under ADR-0012. No protocol-executor or execution-evidence code is present. It depends on `basis-core` and pins `basis-adapters==0.2.0`. It has at least 1 cross-repository file link.

**Responsibility under ADR-0024.** Four BASIS roles (Placement Summary):

| Role | Basis for hosting it in this repository | Durable or first-slice? |
| - | - | - |
| Operation-producer role, including binding creation | ADR-0010; ADR-0012 | Durable. ADR-0010 makes this the permanent repository for the role. |
| Producer-intake boundary | ADR-0018: the producer role's ingress. Repository placement deferred. | Part of the producer role by definition. Its realization's placement is deferred. |
| Protocol-executor role | ADR-0011 Decision 3: colocated "for the first bounded execution reference slice." Decision 5 keeps a separated executor valid. Decision 4 reserves no executor repository. | First-slice topology, not decided as durable |
| Execution-evidence-producer role | ADR-0014: first-slice retention colocated | First-slice topology, not decided as durable |

**Has the role broadened beyond production?** The *repository's* hosted roles have broadened for the first execution slice. The *producer role* has not: ADR-0011's Relationship to ADR-0010 states that "the operation-producer role itself retains the responsibilities ADR-0010 gave it," and that execution is "not `basis-producer` Phase 6." Intake and binding creation are aspects of the producer role, not additional roles. Only the executor and execution-evidence roles are additions, and accepted architecture frames both as colocation for the first slice.

**Should the name follow the permanent role or every colocated role?** The permanent role. Naming after every colocated role would make the name depend on a topology choice that ADR-0011 deliberately leaves open. If the executor later separates, a name that described production and execution would become inaccurate. "Producer" would become exactly accurate. `basis-gateway` follows the same approach: an implementation hosting several roles is named for what it is, not for an enumeration of its roles.

**Does a future separated executor change the answer?** It strengthens it. If the executor separates, `basis-producer` hosts exactly the producer role and its ingress. If the executor stays colocated durably, `basis-producer` is still named for its primary, entry-point role, which leaves the name incomplete but not misleading.

| Criterion | Assessment |
| - | - |
| C1 Accuracy | Accurate for the permanent role |
| C2 Membership | Accurate under every reading |
| C3 Breadth | It names one of four hosted roles. The other two that are additions are first-slice colocation. |
| C4 Durability | Good under separation. Adequate under durable colocation. |
| C5 Comprehension | Good today: no executor code exists yet. Once executor code lands, the README must state the colocation. That is documentation work, not a naming change. |
| C6 Ecosystem clarity | Good |
| C7 Continuity | Lower than the older components (the package is unreleased), but fixed by an accepted ADR |
| C8 Cost | Low technical cost. Governance cost: a rename requires superseding ADR-0010 ([Section 9](#9-adr-0010-naming-constraint)). |

Alternatives considered:

- **`basis-runtime`** is rejected. It is too generic: `basis-gateway` is also a runtime.
- **`basis-operation-runtime`** and similar operation-lifecycle names are rejected. They are broader, but vaguer. They would bind the name to colocation that may not be durable.
- ADR-0010's own rejected alternatives (`basis-agent`, `basis-edge`, and `basis-operation-producer`) remain rejected for the reasons it gives.

**Provisional outcome: KEEP.**

**Revisit trigger.** OD-9 should record a condition under which the name is re-examined: an accepted ADR making executor colocation the durable default beyond the first bounded slice, or this repository beginning to host several protocol-specific executor implementations (ADR-0011 Decision 6). Neither condition holds today.

**Confidence.** High for the current answer. The revisit trigger exists because the executor's durable topology is open, not because the current evidence is ambiguous.

### 6.9 `basis-deploy` (planned, not established)

**Evidence.** No repository exists. The name appears in [`basis-ecosystem.md`](basis-ecosystem.md), [`terminology-rules.md`](../standards/terminology-rules.md#component-naming-conventions), ADR-0010's ownership list ("`basis-deploy` owns packaging and deployment tooling"), and the README's distribution list. Nothing is published under it.

**Responsibility under ADR-0024.** Deployment and distribution tooling: Basitra level, outside BASIS. It "packages BASIS and non-BASIS components" (Placement Summary).

| Criterion | Assessment |
| - | - |
| C1 Accuracy | "Deploy" describes the activity. A noun such as "deployment" describes a repository's responsibility slightly better. |
| C2 Membership | Inaccurate under R2 and R3. Its scope of concern is the whole distribution, including contract publication and the console, which are not BASIS. Accurate only under R1. |
| C3 Breadth | — |
| C4 Durability | Weak under a `basis-` prefix: every non-BASIS component it packages contradicts the name. |
| C5 Comprehension | It would suggest that the tool deploys BASIS only. |
| C6 Ecosystem clarity | Blurring |
| C7 Continuity | None. It exists only as a planned name in documents. |
| C8 Cost | None. There is nothing to migrate. Document references to the planned name are current-state text, reconciled in normal documentation PRs. |

**Provisional outcome: RENAME RECOMMENDED.** This is naming guidance for a future repository, not a rename: the planned name should not be adopted without OD-9 review.

**Candidates:**

- `basitra-deploy`, the closest to existing usage;
- `basitra-deployment`, a noun naming the responsibility;
- `deployment`, which is viable only within a Basitra-named organization.

If OD-9 renames the distribution term ([Section 7.1](#71-basis-core-services-distribution)), the deployment repository's name should be chosen consistently with it.

**Confidence.** High that `basis-deploy` should not be adopted as planned. Medium on the candidate.

**ADR-0010 note.** ADR-0010 mentions `basis-deploy` as current naming in its ownership restatement. It does not establish the component. A different name would not contradict any ADR-0010 decision ([Section 9](#9-adr-0010-naming-constraint)).

### 6.10 `basis-lab`

**Evidence.** The repository `basis-foundation/basis-lab` has a single commit dated 2025-08-12, titled "Initial commit." Its README contains only the title. The commit contains a lab scaffold from the predecessor era:

- Docker Compose stacks for an "as-is" and a "secured" topology;
- a DMZ NGINX proxy with oauth2-proxy and Keycloak (realm `bas-lab`);
- a SCADA placeholder;
- BACnet, Modbus, and OPC UA simulators;
- seed and smoke scripts;
- a hardening checklist.

Its test-scenarios document refers to a future step "when BASis is inserted," the predecessor name in `basis-foundation/basis`. ADR-0024 describes the repository as "uninitialized." More precisely, it is initialized with a dormant predecessor-era lab scaffold, but it has no current project description and no activity since its first commit.

**Responsibility under ADR-0024.** None. It hosts no role in the inventory. Its content, a lab environment with protocol simulators, would be outside BASIS under Decision 5 ("protocol test endpoints, simulators") if it were revived.

**Assessment.** The repository has no current role, so this assessment does not invent one. Three treatments are possible, and the evidence does not choose between them:

1. Treat it as a predecessor-era artifact and archive it with its name retained (the same treatment as Section 6.12).
2. Repurpose it as a Basitra-level lab or test environment. Its scope of concern would then be the whole stack, BASIS and non-BASIS, which under R2 points toward a Basitra-scoped name decided when it is repurposed.
3. Leave it as it is.

**Provisional outcome: DEFER.** **Candidates:** none proposed.

**Confidence.** Low, because the outcome depends on a decision about the repository's purpose that no architecture document makes. OD-9 may defer it explicitly or decide treatment 1 if the Lead Architect confirms that no revival is intended.

### 6.11 `basis-poc`

**Evidence.** A research proof of concept. Its README title is "BASIS — Building Automation Secure Identity Service," the historical expansion. The repository's last commit is 2026-05-17, a README restructure. This repository's [README](../../README.md#relationship-to-basis-poc-and-basis-core) describes it as a research implementation that validated the patterns and that `basis-core` succeeds architecturally. The white paper's Section 06 is about it.

**Responsibility under ADR-0024.** None. It is a historical artifact.

**Assessment.** The strategy document (§10.1 principle 7) states that historical artifacts keep their terminology. The PoC's name is cited by the white paper and by this repository's README as the name of a specific historical artifact. Renaming it would rewrite that history and gain nothing architecturally. At most it warrants a pointer to the current project identity, which the strategy document already anticipates (§12.2), without rewriting its title.

**Provisional outcome: RETIRE / ARCHIVE.** The name is retained. A historical pointer is the only appropriate change. Whether to set GitHub's archived (read-only) flag is a stewardship action that OD-9 may recommend but this assessment does not decide.

**Candidates:** none. **Confidence:** High.

### 6.12 `basis-foundation/basis`

**Evidence.** Five commits between 2025-08-12 and 2025-08-18. Its README title is "Building Automation Systems Identity Shield (BASis)." It contains a scaffold with API, CLI, plugin, and northside directories. Strategy document §3.2 records it as a historical predecessor artifact.

**Responsibility under ADR-0024.** None.

**Assessment.** Like `basis-poc`, it is a historical record. Its name, the bare repository name `basis` in the `basis-foundation` organization, is the oldest naming artifact in the ecosystem. Renaming it would erase its relationship to the BASis-era record. One forward-looking risk: a bare `basis` repository could be mistaken for "the BASIS subsystem" by a reader who does not know the history. A historical note in its README addresses that better than a rename does.

**Provisional outcome: RETIRE / ARCHIVE.** The name is retained, with an optional archival note (strategy document §12.2).

**Candidates:** none. **Confidence:** High.

---

## 7. Naming Concepts Beyond Repositories

### 7.1 BASIS Core Services Distribution

**Evidence.** Current canonical term ([`basis-ecosystem.md`](basis-ecosystem.md#basis-core-services-distribution); [glossary](../glossary.md#basis-core-services-distribution)): "the open-source, deployable set of components maintained under Foundation governance." The README lists its members. They are seven current `basis-*` component repositories plus the planned `basis-deploy`. ADR-0010 makes `basis-producer` "a component of the BASIS Core Services Distribution." No registry artifact carries the distribution's name.

**What ADR-0024 says.** [Decision 8](../adr/0024-long-term-scope-of-basis-within-basitra.md#8-basis-and-the-current-distribution-terminology) states that the distribution "does not correspond to BASIS." It contains Basitra-level components outside BASIS: contract publication, the administrative interface, and planned deployment tooling. Third-party implementations of BASIS roles are outside it. It also states: "Because it begins with 'BASIS,' the name now suggests that everything in the distribution is BASIS, which this decision makes inaccurate." It leaves to OD-9 and OD-10 whether the distribution "keeps, changes, or drops that name."

**Assessment.**

- **"BASIS" in the term.** The distribution's scope of concern is the set of Foundation-maintained Basitra components. That set is broader than BASIS, so the prefix is inaccurate under R2 and R3, and accepted text says so.
- **"Core."** It collides with `basis-core`, and the distribution is not the "core" of anything; it is the whole first-party set.
- **"Services."** The members include libraries (`basis-core`, `basis-adapters`), a contract publication repository, and an interface, not only services.

Each word in the term other than "Distribution" is now inaccurate or confusing.

Human-facing terminology and registry artifacts are separate questions:

| Surface | Today | Who decides |
| - | - | - |
| Human-facing term in architecture and documentation | *BASIS Core Services Distribution* | OD-9 |
| Shorthand | "the distribution" ([terminology rules](../standards/terminology-rules.md#component-naming-conventions)) | OD-9. It is unaffected by a rename and can stay. |
| Registry artifacts, such as a meta-package, bundle, container set, or release train name | None exist | OD-10 |
| Membership statements in accepted ADRs (ADR-0010) | "a component of the BASIS Core Services Distribution" | Historical text. It is read through ADR-0024 Decision 12 and not edited. |

**Provisional outcome: RENAME REQUIRED BY ARCHITECTURE** (rename or retire the term). This is the only item in the assessment where accepted text directly states that the current name is inaccurate. "Keep" remains formally available under Decision 8, but keeping the term would preserve a name that the accepted architecture describes as inaccurate.

**Directions:**

- **Rename to a Basitra-scoped term.** "Basitra distribution" is the leading candidate. "Basitra open-source distribution" is a more descriptive form.
- **Retire the proper name.** Describe the set plainly as "the Foundation-maintained distribution" or "first-party components." Under OD-1, the reference to the Foundation may change.

A term with "BASIS" in it and a narrowed meaning, for example a distribution of only the BASIS-role components, is rejected. It would define a new packaging concept to save a name, and it would separate the console, contract publication, and deployment tooling from the set the Foundation maintains.

**Confidence.** High that the current term should not remain the canonical term. Low on the replacement: whether the distribution needs a proper name at all depends on OD-1, because it is defined by Foundation stewardship, and on OD-10, because a registry-level artifact may need a name.

**Supersession.** None appears required. ADR-0010's membership sentence is a statement about the distribution with its meaning at the time. ADR-0024 Decision 12 already reads such uses as current naming. The OD-9 ADR should say this explicitly.

### 7.2 `basis-foundation` GitHub organization

This assessment records dependencies only. OD-7 governs migration.

- **Effect of OD-7 on OD-9.** Several candidates have two forms: a prefixed form (`basitra-architecture`, `basitra-deploy`) and an unprefixed form (`architecture`, `deployment`). The unprefixed forms are viable only if the organization becomes Basitra-named. The prefixed forms work under either OD-7 outcome. The recommendation in [Section 13](#13-recommendation-to-the-future-od-9-decision-non-normative) prefers forms that do not depend on OD-7.
- **Effect of OD-9 on OD-7.** If OD-9 adopts Basitra-scoped names for `basis-architecture` and deployment, the organization would host both `basitra-*` and `basis-*` repositories. Under R2, that mix accurately reflects scope of concern. It does not by itself require an organization rename, but it is an input to OD-7.
- **Sequencing.** A repository rename and an organization migration each move URLs. If both are expected, a migration plan should consider doing them together so that links move once. This is a migration-planning consideration, not an OD-9 decision.
- **Accepted ADR bodies cite organization-qualified URLs.** ADR-0010 cites `basis-foundation/basis-producer`, and other ADRs cite similar URLs. Those bodies are not edited (ADR supersession rule). Whether such a citation still resolves after migration is an OD-7 concern already recorded in the strategy document's OD-7 row.

### 7.3 Basis Foundation

OD-1 governs the Foundation's name and its relationship to Basitra. This assessment records only where repository and component naming depends on that outcome:

- the human-facing distribution term (Section 7.1), if it references Foundation stewardship;
- `GOVERNANCE.md` content in the architecture repository, but not that repository's name;
- the default `basis-foundation` organization name (Section 7.2), through OD-7.

No repository name in this assessment depends on the Foundation's name.

### 7.4 Surfaces not assessed

The public website source (`basis-website`) and the BASIS website are public-site migration concerns (strategy document §12.2 and Stage 5). They depend on OD-1 and OD-7 and are not assessed here. Ipotio is external and unaffected (OD-3).

---

## 8. Current Naming Versus Target Architecture

"Fit" is assessed against the target question in Section 1. Labels are provisional assessment labels only ([Section 2.2](#22-provisional-outcome-labels)).

| Current repository / concept | Architectural responsibility under ADR-0024 | BASIS membership | Current-name fit | Provisional recommendation | Confidence | Candidate target name(s) | Compatibility concern | Decision dependency |
| - | - | - | - | - | - | - | - | - |
| `basis-architecture` | Architecture and governance of Basitra, including BASIS | Outside (Basitra level) | Poor: scope is Basitra, name says BASIS | **RENAME RECOMMENDED** | High (direction); Medium (string) | `basitra-architecture`; `architecture` (only in a Basitra-named org) | Most-linked repository; URLs in accepted ADR bodies; no package | OD-7 (final string) |
| `basis-core` | Authorization-kernel role; adapter interface contracts | Inside | Adequate; "core" is positional and depends on the `basis-` prefix | **LIKELY KEEP** | Medium | `basis-kernel`; `authorization-kernel` (if renamed) | Most-linked component repository; if OD-10 keeps the package name, repository and package names diverge (package, two dependents, kernel import rules are OD-10 surfaces) | OD-10 (package/import); flips if a `basitra-` prefix is adopted |
| `basis-gateway` | Subject establishment, producer admission, composition, context admission, enforcement, audit evidence | Inside | Good as an implementation term | **LIKELY KEEP** | Medium | `basis-enforcement` (not recommended) | Released package; `basis_gateway.*` evidence namespace | OD-10; contract governance |
| `basis-identity` | Identity-engine role | Inside | Good | **KEEP** | High | — | Released package | — |
| `basis-adapters` | Protocol-adapter role | Inside | Good | **KEEP** | High | — | Released package; `adapter_source` values embed the name | OD-10; contract governance |
| `basis-console` | Administrative-interface role (consumer of BASIS) | Outside (Basitra level) | Good under R2; inaccurate under R3 | **LIKELY KEEP** (R2) / **RENAME RECOMMENDED** (R3) | Medium | `basitra-console`; `console` (only in a Basitra-named org) | Released package | OD-9 convention choice; OD-7 for unprefixed form |
| `basis-schemas` | Contract publication of BASIS contract semantics | Publication outside; semantics inside | Mostly good under R2; "contracts" more exact; inaccurate under R3 | **LIKELY KEEP** (R2) / **RENAME RECOMMENDED** (R3 or scope change) | Medium | `basitra-contracts`; `basis-contracts`; `contracts` (only in a Basitra-named org) | Released package; contract identifiers and reserved namespaces | OD-9 convention choice; OD-10; contract governance |
| `basis-producer` | Operation-producer role (with intake ingress and binding creation); first-slice colocated executor and execution evidence | Inside | Good for the permanent role | **KEEP** | High (current); revisit trigger recorded | — | Unreleased package; fixed by ADR-0010 | ADR-0010 (supersession needed for any rename) |
| `basis-deploy` (planned) | Deployment and distribution tooling | Outside (Basitra level) | Poor: scope is the whole distribution | **RENAME RECOMMENDED** (planned-name guidance) | High (direction); Medium (string) | `basitra-deploy`; `basitra-deployment`; `deployment` (only in a Basitra-named org) | None; not established | Distribution term (§7.1); OD-7 for unprefixed form |
| `basis-lab` | None (dormant predecessor-era lab scaffold) | — | Not assessable: no role | **DEFER** | Low | None proposed | Minimal | Repository purpose (not an architecture decision) |
| `basis-poc` | None (historical research artifact) | — | Historical | **RETIRE / ARCHIVE** (name retained) | High | — | Cited by white paper and README | — |
| `basis-foundation/basis` | None (historical predecessor artifact) | — | Historical | **RETIRE / ARCHIVE** (name retained) | High | — | Minimal | — |
| BASIS Core Services Distribution | Foundation-maintained distribution of BASIS and non-BASIS components | Not a membership concept (ADR-0024 Decision 8) | Inaccurate per ADR-0024 Decision 8 | **RENAME REQUIRED BY ARCHITECTURE** (rename or retire) | High (change); Low (replacement) | "Basitra distribution"; "Basitra open-source distribution"; or retire the proper name | No registry artifact; ADR-0010 membership sentence (historical) | OD-1; OD-10 (registry artifacts) |
| `basis-foundation` organization | Repository namespace | — | Not assessed | **DEFER** | — | — | All repository URLs | OD-7 (depends on OD-1) |
| Basis Foundation | Governance and stewardship body | — | Not assessed | **DEFER** | — | — | Governance text | OD-1 |

---

## 9. ADR-0010 Naming Constraint

**What ADR-0010 fixes.** [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md) (`Accepted`) establishes the following under the architecture accepted at that time:

- the permanent component name `basis-producer`;
- the permanent repository `basis-foundation/basis-producer`;
- the architectural name "BASIS Producer," or "the BASIS operation-producer runtime";
- the reserved import namespace `basis_producer`.

**ADR-0024 did not supersede it.** ADR-0024 Decision 12 states that ADR-0010 is "semantically unchanged," that "BASIS Producer" accurately describes "a Foundation-maintained implementation of a BASIS role," and that "ADR-0010 fixes the name `basis-producer`, so any rename requires a superseding ADR. That is an OD-9 concern."

**Consequence for OD-9.** If OD-9 selects a different repository or component name for `basis-producer`, the normative OD-9 decision must explicitly supersede ADR-0010, or the naming portion of it. This assessment does not recommend that ([Section 6.8](#68-basis-producer)), and it performs no supersession.

**A process gap OD-9 should know about.** The ADR process ([`docs/adr/README.md`](../adr/README.md#supersession)) defines whole-ADR supersession only. A superseded ADR's status changes to `Superseded`, and no lifecycle state exists for amending part of an ADR. ADR-0010 combines a naming decision with component-boundary, distribution, visibility, and implementation-authorization decisions. A rename would therefore need one of two things:

- a superseding ADR that restates every unchanged ADR-0010 decision, so that none is lost when ADR-0010 becomes `Superseded`;
- a process change introducing partial supersession or amendment, which is a governance question comparable to OD-5, not a naming question.

Under the recommended outcome (KEEP), neither is needed.

**Other accepted ADRs that fix names.** Each accepted ADR was checked for a comparably strong naming commitment:

| ADR | Naming content | Strength |
| - | - | - |
| ADR-0008 | Explicitly declines to name a permanent producer repository ("does not create or name a permanent `basis-producer` repository"). Defines no BASIS-specific URI namespace for workload identity. | None. No naming constraint. |
| ADR-0010 | As above | **Fixes a name** |
| ADR-0011 | Decision 4: "does not reserve a repository name" for an executor and does not create `basis-executor` | None. It explicitly reserves nothing. |
| ADR-0018 | Defers intake repository placement | None |
| ADR-0019 | Basitra names no component. BASIS is a standalone acronym. No acronym-driven names. Renames need migration plans. | Constrains *how* names are chosen. Fixes no repository name. |
| ADR-0022 | Logical workload identity is role-specific. ADR-0024 infers that a rename leaving role and scope unchanged does not by itself change a workload identity. | Compatibility relevant. A migration plan should confirm the inference. |
| ADR-0023 | "BASIS administrative interface," "BASIS-native tool": names of interfaces, read by ADR-0024 Decision 12 as naming what is administered | Supports R2 for `basis-console`. Not a name constraint ("about architectural roles, not names"). |
| ADR-0024 | Decision 1: BASIS does not name a repository prefix. Decision 8: the distribution name is transitional. Chooses no name. | Constrains conventions (Section 12). Fixes no name. |
| ADR-0007, ADR-0009, ADR-0012–ADR-0017, ADR-0020, ADR-0021 | Use current component names descriptively | None. They read as current naming under ADR-0024 Decision 12. |

ADR-0010 is the only accepted ADR that fixes a repository or component name. Canonical documents that are not ADRs also encode names. They need bounded documentation updates after any rename, not supersession:

- [`terminology-rules.md`](../standards/terminology-rules.md#component-naming-conventions) component list;
- [`basis-ecosystem.md`](basis-ecosystem.md) repository table;
- [`kernel-boundary-rules.md`](../kernel-boundary-rules.md) import rules;
- [`basis-schemas.md`](basis-schemas.md) and [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md) `basis_gateway.*` namespace.

---

## 10. Compatibility Surfaces

### 10.1 Which surfaces OD-9 decides

A repository rename does not imply that every surface carrying the name changes with it. Each surface has its own owner:

| Surface | Example | OD-9 decides? | Later owner |
| - | - | - | - |
| Repository name | `basis-architecture`, `basis-core` | Yes | OD-9 |
| Component / display name | "BASIS Producer"; `basis-gateway` in prose | Yes | OD-9 |
| Human-facing distribution term | *BASIS Core Services Distribution* | Yes | OD-9 (with OD-1) |
| Python distribution name | `basis-core` on a package index | No | OD-10 |
| Import namespace | `basis_core`, `basis_gateway` | No | OD-10. A breaking change under [`compatibility-philosophy.md`](compatibility-philosophy.md). |
| Inter-package dependency specifiers | `basis-core>=0.2.1,<0.3.0`; `basis-adapters==0.2.0` | No | OD-10 / migration plan |
| Container images | None published today | Usually later | OD-10 / migration plan |
| Configuration and environment keys | `BASIS_LOCAL_TOKEN_*` in `basis-gateway` | No | Compatibility plan |
| Schema and contract identifiers | `decision-request` and similar contract names (no component name) | No | OD-10 / contract governance |
| Reserved evidence namespaces | `basis_gateway.*` | No | OD-10 / contract governance |
| Evidence data values | `adapter_source: basis-adapters:bacnet` in published examples; values in retained evidence records | No | Contract governance. Retained evidence is historical and is never rewritten. |
| Workload identity strings | URI SANs (deployment-defined; ADR-0008) | No | Unaffected by architecture (ADR-0008; ADR-0022); a migration plan confirms |
| Documentation URLs | `github.com/basis-foundation/basis-architecture/...` | Consequence | Migration plan; redirects; OD-7 |
| GitHub Actions and automation references | Repository paths in workflows | Consequence | Migration plan |
| Badges, release artifacts, tags | Repository-qualified badge and release URLs | Consequence | Migration plan. Existing tags and release notes are historical. |
| Accepted ADR bodies and historical records | Repository names and URLs in ADR-0007 to ADR-0024, release notes, the white paper | No. Never rewritten. | Supersession rule; redirects |

### 10.2 Surfaces affected by each rename candidate

Only repositories and concepts assessed as RENAME RECOMMENDED or RENAME REQUIRED BY ARCHITECTURE, or conditionally so, are listed. Counts are of files in local checkouts and are indicative, not exhaustive. The "Package / import (OD-10)" and "Dependents" columns list surfaces that carry the name. They do not change with a repository rename unless OD-10 or the applicable compatibility process separately authorizes it.

| Candidate | Repository URL and links | Package / import (OD-10) | Dependents | Contract or evidence surfaces | Accepted-ADR text | Other |
| - | - | - | - | - | - | - |
| `basis-architecture` | At least 35 cross-repository file links. Relative links inside the repository are unaffected. | None | None | None | Repository named in ADR-0007, ADR-0008, ADR-0024, and the ADR README; not edited | README badges; white-paper links; external citations |
| `basis-deploy` (planned) | None | Not yet created | None | None | ADR-0010 ownership list (current naming) | Current-state documents listing the planned name |
| Distribution term | Not a URL | No registry artifact today | — | None | ADR-0010 "Distribution Membership" (historical) | Glossary, `basis-ecosystem.md`, README, writing guidelines, white-paper section titles (historical) |
| `basis-console` (only under R3) | No cross-repository links observed | `basis-console` / `basis_console` | None | None | ADR-0023 and ADR-0024 refer to it by name | Released 0.2.0 |
| `basis-schemas` (only under R3 or scope change) | At least 3 cross-repository file links | `basis-schemas` / `basis_schemas` | Contract consumers by documentation reference | Contract identifiers do not carry the name and should not change with it | Many ADRs cite it as "`basis-schemas`" | Released 0.2.2; charter vocabulary "schemas publish" |
| `basis-core` (only if a `basitra-` prefix is adopted) | At least 20 cross-repository file links | `basis-core` / `basis_core` | `basis-gateway`, `basis-producer` | Kernel import rules in `kernel-boundary-rules.md` | Cited throughout | Oldest released package; white paper |

GitHub generally redirects a renamed repository's web and Git URLs, as long as no new repository takes the old name. That reduces, but does not remove, link-migration cost. A migration plan must verify the behavior for every affected surface, including automation that pins repository paths. This assessment designs no migration.

---

## 11. Naming Coherence Across the Ecosystem

The individual outcomes in Sections 6–8 are checked here against each other.

**1. If some BASIS-role repositories keep `basis-*` while Basitra-level repositories use `basitra-*`, is that coherent?** It depends on what the prefix claims:

- **Under R2 (scope of concern)**, mixed prefixes are coherent. `basitra-` marks a responsibility that spans the ecosystem (architecture, deployment, the distribution). `basis-` marks one that concerns BASIS. Under this rule `basis-console` and `basis-schemas` stay `basis-`, because their whole object is BASIS.
- **Under R3 (membership)**, mixed prefixes are coherent only if every Basitra-level repository changes, including the console and schemas.

Either is internally consistent. Mixing them is not. For example, renaming `basis-schemas` for membership reasons while keeping `basis-console` would be incoherent.

**2. Would such a convention make architectural membership depend on naming?** Under R3, it would risk exactly that. A component that later hosts a BASIS role and a non-BASIS role, which ADR-0024 Decision 7 anticipates, would have no correct prefix. Any reassignment of a role would become a rename, and third-party implementations of BASIS roles would never carry the prefix, so the prefix could never be a complete membership signal. ADR-0024 Decision 1 already states that BASIS does not name a repository prefix. Under R2 the risk is smaller: the prefix describes what a repository is about, and membership stays where ADR-0024 puts it, in the Placement Summary and the ADR that introduces each role.

**3. Should prefixes encode subsystem membership at all?** The evidence says no. Membership is a role-level property (ADR-0024 Decision 7). Repositories host roles in combinations that change with topology (ADR-0011 Decisions 3 and 5). Encoding a role-level property in a repository-level string would be lossy and fragile.

**4. Would plain responsibility names be clearer?** Inside a Basitra-named organization, possibly. In the current organization, and anywhere a repository name appears out of context (forks, package indexes, citations, search), plain names such as `architecture`, `console`, or `contracts` lose all ecosystem association and collide with common names. Convention C is analyzed in Section 12.

**5. Does retaining `basis-*` for some repositories create unnecessary historical complexity?** Some, but it is bounded. Readers must learn that `basis-` means "concerns BASIS," not "is the project." That distinction is already required by ADR-0024 Decision 12's reading rule. Renaming every repository would not remove it: historical records would still use `basis-*` for everything.

**6. Would renaming all repositories produce needless compatibility churn?** Yes. Five of the seven current component repositories implement BASIS roles and are accurate under every reading of the prefix (Sections 6.2–6.5 and 6.8). The churn comes from what a repository rename actually changes, which is less than a combined repository-and-package migration:

- **What follows a repository rename.** Repository URLs, cross-repository links, badges, release and documentation URLs, and automation that pins repository paths (Section 10.1). Redirects absorb part of this, which a migration plan must verify.
- **What can stay unchanged.** Python distribution names, import namespaces, and dependency specifiers. A repository rename does not move them.
- **What needs separate authorization.** Changing any of those surfaces requires an OD-10 decision and the applicable compatibility process. If they stay unchanged, each renamed repository publishes a package whose name no longer matches it. That mismatch either persists or creates pressure for a later OD-10 migration.

For repositories whose names are already accurate, none of these consequences buys any gain in architectural accuracy.

**7. Can a contributor understand Basitra → BASIS from the repository names alone?** Not fully under any convention:

- Under R2, the names show which repositories concern the whole ecosystem and which concern BASIS. They do not show that `basis-console` and `basis-schemas` are outside BASIS.
- Under R3, the names would show membership for first-party repositories only, and only until a role moves.

The containment of BASIS within Basitra, and the placement of each role, should be carried by documentation: ADR-0024's Placement Summary, `basis-ecosystem.md`, and each repository's README. Names should be accurate, not exhaustive.

**Conclusion.** Architecture clarity outranks naming uniformity. The most accurate and least churning outcome is a responsibility-based convention whose prefix records scope of concern, not membership. Symmetry for its own sake is not a reason to rename.

---

## 12. Candidate Convention Analysis

All example names in this section are illustrative only.

### Convention A: Preserve most `basis-*`; rename Basitra-level repositories that misstate membership

```text
basitra-architecture
basis-core
basis-gateway
basis-identity
basis-adapters
basis-producer
basitra-console
basitra-contracts
basitra-deploy
```

This is R3 applied to first-party repositories.

| Benefits | Problems |
| - | - |
| Names signal membership for first-party repositories. | It encodes a role-level property in repository names, so any role reassignment, or a component hosting both BASIS and non-BASIS roles, forces a rename (Section 11, question 2). |
| Implementation repositories are untouched, which keeps compatibility churn moderate. | It renames the `basis-console` and `basis-schemas` repositories even though their current names are accurate as descriptions of what they concern. Their released packages need not change (OD-10), but if they do not, those repository and package names diverge. ADR-0024 Decision 12 supports reading "BASIS" in "BASIS administrative interface" as the object, not membership. |
| It matches ADR-0024's outside-BASIS placements at a glance. | Third-party BASIS-role implementations never carry the prefix, so the signal is incomplete by design. |
| | It treats publication neutrality as needing a non-BASIS name, which ADR-0024 Decision 4 does not require. |

### Convention B: `basitra-` prefix on every first-party repository

```text
basitra-architecture
basitra-core
basitra-gateway
basitra-identity
basitra-adapters
basitra-producer
basitra-console
basitra-schemas
basitra-deploy
```

| Benefits | Problems |
| - | - |
| Uniform, and brand-clear for Basitra. | Repository names stop communicating anything about BASIS. The subsystem ADR-0024 defines becomes invisible in the repository layer. |
| No membership claim is made by any name. | `basitra-core` would misstate the kernel as the core of the ecosystem, the confusion ADR-0019 already had to correct for `basis-core`. |
| | The widest repository-level churn: every repository URL, link, badge, and automation reference moves. Package and import names need not change (OD-10). If they do not, every repository's name diverges from its package's name. Closing that gap would require a separate OD-10 migration of every package and import namespace. |
| | It risks reading as "Basitra components," against ADR-0019's statement that Basitra names no component. |
| | Uniformity is the main justification, which the strategy document's principles do not accept as sufficient. |

### Convention C: Responsibility names independent of an ecosystem prefix

```text
architecture
authorization-kernel
enforcement
identity
protocol-adapters
console
contracts
producer
deployment
```

| Benefits | Problems |
| - | - |
| Conceptual purity: each name states only the responsibility. | Viable only inside a Basitra-named organization (OD-7). In `basis-foundation`, and anywhere names appear without their organization, these names lose all ecosystem association. |
| No prefix question at all. | Generic names collide with common repository and package names, and are weak in search and forks. |
| | "enforcement" names one role and would misdescribe the multi-role gateway (Section 6.3). |
| | Repository-level churn comparable to Convention B. Package and import names would not change with the repository renames. Because names this generic are unlikely to be available or distinctive as package names, OD-10 would likely keep prefixed package names, leaving every repository with a different name from its package. |

### Convention D: Hybrid, responsibility-based, prefix records scope of concern

```text
basitra-architecture      Basitra scope   architecture and governance of the ecosystem
basis-core                BASIS scope     implements a BASIS role
basis-gateway             BASIS scope     implements BASIS roles
basis-identity            BASIS scope     implements a BASIS role
basis-adapters            BASIS scope     implements a BASIS role
basis-producer            BASIS scope     implements BASIS roles
basis-console             BASIS scope     administers BASIS (Basitra-level consumer)
basis-schemas             BASIS scope     publishes BASIS contract semantics (Basitra-level role)
basitra-deploy            Basitra scope   packages BASIS and non-BASIS components
```

The rule: a repository's prefix reflects the scope its responsibility concerns. `basis-` is used where the repository implements a BASIS role or exists to serve, publish, or administer BASIS specifically. A Basitra-scoped name is used where the responsibility spans the ecosystem. The responsibility word is chosen per repository. Membership is never inferred from the prefix. It is carried by ADR-0024 and the documentation.

| Benefits | Problems |
| - | - |
| It follows ADR-0024 most closely: membership by role (Decision 7), no prefix as BASIS (Decision 1), "BASIS" as the object of an interface (Decision 12). | A contributor may still misread `basis-console` or `basis-schemas` as BASIS members. Documentation must correct this, and it has no security consequence (membership confers nothing, Decision 2). |
| The lowest churn consistent with accuracy: it changes only names that are inaccurate under any non-legacy reading (`basis-architecture`, the planned `basis-deploy`, the distribution term). | It needs a stated rule that reviewers can apply. "Scope of concern" is a judgment, though it is less volatile than membership, because it changes only when a repository's subject changes, not when a role is colocated or separated. |
| It stays correct when roles are colocated or separated (the `basis-producer` executor question). | The scope of `basis-console` and `basis-schemas` must be re-checked if either begins to concern non-BASIS matters (Sections 6.6 and 6.7). |
| It preserves the visibility of BASIS in the repository layer, which Convention B loses. | Two prefixes in one organization, which is less tidy than Convention B. |
| It works under either OD-7 outcome. | |

---

## 13. Recommendation to the Future OD-9 Decision (Non-Normative)

The following is a recommendation for the future OD-9 decision to consider. It is not a naming decision and adopts nothing.

**1. Which current names appear safe to keep?**

- `basis-identity`, `basis-adapters`, and `basis-producer` (KEEP; High).
- `basis-gateway` (LIKELY KEEP; Medium).
- `basis-core` (LIKELY KEEP; Medium, conditional on the `basis-` prefix being retained for implementation repositories).
- `basis-console` and `basis-schemas` (LIKELY KEEP; Medium), under the recommended scope-of-concern convention.
- `basis-poc` and `basis-foundation/basis`, retained as history (RETIRE / ARCHIVE; High).

**2. Which names appear misleading under ADR-0024?**

- The **BASIS Core Services Distribution** term. ADR-0024 Decision 8 itself states that the name is inaccurate (RENAME REQUIRED BY ARCHITECTURE).
- **`basis-architecture`**: it governs Basitra, not only BASIS (RENAME RECOMMENDED).
- The planned **`basis-deploy`**: its responsibility packages BASIS and non-BASIS components (RENAME RECOMMENDED as guidance for a future repository).

**3. Which names need explicit reconsideration?**

- `basis-console` and `basis-schemas`: their outcome depends on the convention OD-9 adopts (R2 versus R3).
- `basis-core`: its retention depends on not adopting a `basitra-` prefix for implementation repositories.
- `basis-producer`: kept, with a recorded revisit trigger tied to executor topology.
- `basis-lab`: deferred until its purpose is decided.

**4. High versus low confidence.**

- **High:**
  - keeping `basis-identity`, `basis-adapters`, and `basis-producer`;
  - the direction for `basis-architecture`, the planned `basis-deploy`, and the distribution term;
  - historical treatment of `basis-poc` and `basis-foundation/basis`.
- **Medium:**
  - `basis-core`, `basis-gateway`, `basis-console`, and `basis-schemas`;
  - the specific strings for `basis-architecture` and deployment.
- **Low:**
  - `basis-lab`, which has no current role to evaluate;
  - the replacement distribution term, which depends on OD-1 and OD-10.

**5. Dependencies on OD-1, OD-7, and OD-10.**

- **OD-7:** the final string for `basis-architecture` and deployment. Unprefixed forms are viable only in a Basitra-named organization. Sequencing repository and organization moves is also an OD-7 concern.
- **OD-1:** the replacement distribution term, if it references Foundation stewardship. `GOVERNANCE.md` content.
- **OD-10:** every package, import namespace, dependency specifier, registry artifact, reserved evidence namespace (`basis_gateway.*`), and evidence data value (`adapter_source`). An OD-9 repository rename must state explicitly that it does not change these, or defer them to OD-10.

**6. Does the evidence support one coherent convention?** Yes: **Convention D** with the scope-of-concern rule (R2). It accounts for every outcome in this assessment without exceptions, applies ADR-0024's own reading of "BASIS" as the object of an interface, and keeps compatibility churn proportionate to the inaccuracy corrected. Convention A (R3) is a coherent alternative if the Lead Architect prefers names to carry membership. It would add `basis-console` and `basis-schemas` (as `basitra-contracts`) to the rename set and make every future role reassignment a naming question. Conventions B and C are not recommended.

The recommended outcome set under Convention D, for OD-9 to consider:

```text
basis-architecture                →  Basitra-scoped name (basitra-architecture preferred; final form with OD-7)
basis-core                        →  keep
basis-gateway                     →  keep
basis-identity                    →  keep
basis-adapters                    →  keep
basis-console                     →  keep (re-check if scope extends beyond BASIS)
basis-schemas                     →  keep (re-check if publication extends beyond BASIS; use "contracts" if ever renamed)
basis-producer                    →  keep (ADR-0010; revisit trigger for durable executor colocation)
basis-deploy (planned)            →  do not adopt; Basitra-scoped name when created
basis-lab                         →  defer (purpose undecided)
basis-poc                         →  historical; name retained
basis-foundation/basis            →  historical; name retained
BASIS Core Services Distribution  →  rename to a Basitra-scoped term, or retire the proper name (with OD-1 / OD-10)
```

**7. Which accepted ADRs would need supersession if the recommendation were adopted?** None:

- ADR-0010's name is kept.
- ADR-0010's references to `basis-deploy` and to the BASIS Core Services Distribution are current-naming and distribution-membership statements, read through ADR-0024 Decision 12. A different deployment-repository name or distribution term would not contradict any ADR-0010 decision. The OD-9 ADR should state this explicitly.
- No other accepted ADR fixes a name (Section 9).

If OD-9 instead renames `basis-producer`, ADR-0010 must be superseded, and the whole-ADR supersession gap in Section 9 must be addressed.

**What the OD-9 decision also needs to state**, beyond names:

- the convention adopted (R2 or R3) and its rule, so future repositories, such as an intake realization or a separated executor, can be named by rule;
- display-name forms (for example "BASIS Producer"), consistent with [`terminology-rules.md`](../standards/terminology-rules.md#capitalization-rules);
- that no compatibility surface outside OD-9 (Section 10.1) changes with a repository rename unless OD-10 or a compatibility plan decides it;
- the follow-on documentation updates: `terminology-rules.md`, `basis-ecosystem.md`, the glossary, and the README.

---

## 14. Decision Quality Checks

| # | Check | Result |
| - | - | - |
| 1 | Is every recommendation derived from accepted architectural responsibility? | Yes. Each outcome cites the ADR-0024 placement and the governing ADRs. Where accepted architecture gives no role (`basis-lab`), the outcome is DEFER. |
| 2 | Did current naming influence the architectural classification? | No. Every placement is taken unchanged from ADR-0024. Names were evaluated only after placement. |
| 3 | Did the assessment distinguish role from implementation? | Yes (Section 4; Sections 6.3, 6.5, and 6.8). |
| 4 | Did it distinguish repository names from package and import names? | Yes (Section 10.1). Package, import, registry, contract-identifier, and evidence-value surfaces are assigned to OD-10 or contract governance. |
| 5 | Did it preserve ADR-0010's accepted naming constraint? | Yes. It recommends KEEP and records that any rename requires supersession (Section 9). |
| 6 | Did it avoid assuming all `basis-*` repositories must change? | Yes. Seven current repositories are KEEP or LIKELY KEEP. |
| 7 | Did it avoid assuming all Basitra-level repositories must use `basitra-*`? | Yes. Two Basitra-level repositories, `basis-console` and `basis-schemas`, are LIKELY KEEP under the recommended convention, and unprefixed forms are evaluated throughout. |
| 8 | Did it preserve historical repositories as history? | Yes (Sections 6.11 and 6.12). |
| 9 | Did it identify compatibility cost without designing migration? | Yes (Section 10). No migration steps, sequence, or deprecation periods are designed. |
| 10 | Could a future OD-9 ADR decide from this assessment without another repository inventory? | Yes, subject to re-verifying the implementation evidence against current remote branches (Section 2.1). The inventory, placements, criteria, candidates, ADR constraints, compatibility surfaces, and convention analysis are all here. The remaining open inputs are named decisions (the R2 or R3 choice, OD-1, OD-7, OD-10, and `basis-lab`'s purpose), not missing evidence. |

---

## 15. What This Assessment Does Not Decide

This assessment does not:

- resolve OD-9, or create the normative OD-9 ADR;
- rename, create, archive, or retire any repository, component, distribution, Python package, import namespace, schema or contract identifier, container image, configuration key, organization, foundation, or website;
- resolve OD-1, OD-7, or OD-10, or decide the Basis Foundation's name or the GitHub organization's migration;
- supersede or amend ADR-0010 or any other ADR, or edit ADR-0024;
- change any placement made by ADR-0024;
- design a compatibility or migration plan;
- modify any implementation repository, schema, contract, or Ipotio;
- rewrite any historical artifact.

---

## 16. References

- [ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md): long-term scope of BASIS within Basitra (Accepted): Decisions 1–2, 4–8, 12; Placement Summary; Follow-On Work
- [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md): Basitra identity and terminology hierarchy (Accepted)
- [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md): `basis-producer` as the operation-producer runtime (Accepted)
- [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [ADR-0011](../adr/0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](../adr/0012-authorization-to-execution-binding.md), [ADR-0014](../adr/0014-minimum-execution-evidence-semantics.md), [ADR-0015](../adr/0015-first-bounded-execution-target.md), [ADR-0018](../adr/0018-upstream-supervisory-producer-intake-boundary.md), [ADR-0022](../adr/0022-workload-credential-lifecycle-and-scope-boundary.md), [ADR-0023](../adr/0023-supervisory-platform-and-administrative-interface-boundary.md)
- [`docs/adr/README.md`](../adr/README.md): lifecycle states and the supersession rule
- [`basitra-ecosystem-and-boundary-aware-security.md`](basitra-ecosystem-and-boundary-aware-security.md): §3, §10, §12.2, §12.4, §12.5, §13
- [`basitra-target-architecture-discovery-assessment.md`](basitra-target-architecture-discovery-assessment.md): §4.3, §5, §12
- [`basis-ecosystem.md`](basis-ecosystem.md), [`basis-gateway.md`](basis-gateway.md), [`basis-identity.md`](basis-identity.md), [`basis-adapters.md`](basis-adapters.md), [`basis-console.md`](basis-console.md), [`basis-schemas.md`](basis-schemas.md), [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md)
- [`bounded-rest-execution-implementation-plan.md`](bounded-rest-execution-implementation-plan.md)
- [`compatibility-philosophy.md`](compatibility-philosophy.md), [`kernel-boundary-rules.md`](../kernel-boundary-rules.md)
- [`terminology-rules.md`](../standards/terminology-rules.md), [`glossary.md`](../glossary.md)
- [`GOVERNANCE.md`](../../GOVERNANCE.md), [`README.md`](../../README.md)
- Implementation repositories inspected locally on 2026-10-06 (cross-repository evidence, not hyperlinked): `basis-core`, `basis-gateway`, `basis-identity`, `basis-adapters`, `basis-console`, `basis-schemas`, `basis-producer`, `basis-lab`, `basis-poc`, `basis-foundation/basis`
