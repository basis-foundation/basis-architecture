# ADR-0025: Basitra Repository and Component Naming

## Status

Accepted

Consistent with this repository's established practice (see ADR-0018's Context, and the Status sections of ADR-0019, ADR-0023, and ADR-0024), this ADR was numbered when first proposed and submitted as `Proposed`, and its merging did not constitute acceptance. It has since undergone that separate, dedicated formal-acceptance review, and its status now records `Accepted`. Acceptance resolves open decision OD-9 in the [strategy document](../architecture/basitra-ecosystem-and-boundary-aware-security.md#13-open-decisions) by fixing target names. It changes no part of the decision below. It renames, creates, or migrates no repository, package, organization, or site (Decision 15), and performs none of the follow-on work listed under **Follow-On Work**, each item of which requires its own bounded work. No canonical reference document changes because of this acceptance. The mismatch between this numbering practice and [`README.md`](README.md#numbering) remains open decision OD-5.

## Context

[ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Accepted) made *Basitra* the long-term identity of the project, community, and ecosystem. Basitra names no runtime component. Its Decision 7 sets the governing principle for names: *historical continuity informs migration but does not constrain the target architecture.*

[ADR-0024](0024-long-term-scope-of-basis-within-basitra.md) (Accepted) resolved OD-8. BASIS is the Boundary-Aware Security subsystem of Basitra that governs operations along the governed path. Membership attaches to architectural roles, decided by the Path, Security, and Ownership conditions of its [Decision 2](0024-long-term-scope-of-basis-within-basitra.md#2-membership-rule), and not to repositories, components, or first-party implementations. Its [Placement Summary](0024-long-term-scope-of-basis-within-basitra.md#placement-summary) places every current role. Its Decision 1 states that BASIS does not name "a repository prefix or a set of repositories." Its [Decision 8](0024-long-term-scope-of-basis-within-basitra.md#8-basis-and-the-current-distribution-terminology) states that the *BASIS Core Services Distribution* does not correspond to BASIS and that its name is transitional. Its Follow-On Work item 2 hands the next question to OD-9:

> Given the architecture now accepted by ADR-0024, does the current name still accurately describe the architectural responsibility of this repository or component?

The evidentiary input is [`basitra-repository-and-component-naming-assessment.md`](../architecture/basitra-repository-and-component-naming-assessment.md) (the "naming assessment"). It is non-normative. It inventories every current repository and naming concept, the roles each hosts, and where each role sits ([its §4](../architecture/basitra-repository-and-component-naming-assessment.md#4-repositories-versus-roles)). It evaluates each against eight criteria ([§5](../architecture/basitra-repository-and-component-naming-assessment.md#5-evaluation-criteria-and-the-three-readings-of-basis-)), records compatibility surfaces and their owners ([§10](../architecture/basitra-repository-and-component-naming-assessment.md#10-compatibility-surfaces)), and analyzes four candidate conventions ([§12](../architecture/basitra-repository-and-component-naming-assessment.md#12-candidate-convention-analysis)). Its recommendation is Convention D. This ADR uses that evidence. It does not treat the recommendation as authority, and it decides the items the assessment left to OD-9: the reading of the `basis-` prefix, the target strings, the distribution term, and the treatment of `basis-lab`.

Four facts from the assessment carry most of the weight:

- **Three readings of a `basis-` prefix are in use** ([§5.2](../architecture/basitra-repository-and-component-naming-assessment.md#52-three-readings-of-a-basis--prefix)):
  - **R1, family label:** the repository is in the first-party `basis-*` family. This is the current, historical reading.
  - **R2, scope of concern:** the repository's responsibility concerns BASIS. It implements a BASIS role, or it exists to serve, publish, or administer BASIS specifically.
  - **R3, subsystem membership:** the repository implements a BASIS role, and nothing else.
- **Most current names are accurate under every reading.** `basis-core`, `basis-gateway`, `basis-identity`, `basis-adapters`, and `basis-producer` implement BASIS roles.
- **Four names depend on the reading.** `basis-console` and `basis-schemas` host Basitra-level roles whose whole object is BASIS. They are accurate under R1 and R2 and inaccurate under R3. `basis-architecture` and the planned `basis-deploy` host Basitra-level roles whose object is all of Basitra. They are accurate only under R1.
- **Only one accepted ADR fixes a name.** [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) fixes `basis-producer` ([naming assessment §9](../architecture/basitra-repository-and-component-naming-assessment.md#9-adr-0010-naming-constraint)). ADR-0024 Decision 12 leaves it semantically unchanged and states that any rename requires a superseding ADR.

## Problem

What naming convention should Basitra's repositories and components follow, and what is the target name of each current repository, planned repository, and the distribution? The answer must be specific enough that:

- each current and planned repository has a decided target treatment;
- a reviewer can name a future repository by rule, without another naming exercise;
- no name is read as deciding BASIS membership, which ADR-0024 fixes;
- repository naming stays separate from package, import-namespace, contract, and evidence names, which carry compatibility obligations of their own.

It must do this without renaming anything, reopening ADR-0019 or ADR-0024, superseding ADR-0010, or deciding OD-1, OD-7, or OD-10.

## Decision

**Basitra adopts a hybrid, responsibility-based naming convention (the naming assessment's Convention D). A repository or component name describes its architectural responsibility and its scope of concern. A `basis-` prefix records that the responsibility concerns BASIS. A `basitra-` prefix records that the responsibility concerns Basitra as a whole. A `basis-` prefix does not by itself assert architectural membership in BASIS. Membership is defined only by ADR-0024.**

```text
Basitra                                   project · community · ecosystem identity (ADR-0019)
│
├── architecture and governance           scope: Basitra
│   └── basitra-architecture              (target name; currently basis-architecture)
│
├── the Basitra distribution              Foundation-maintained first-party distribution
│                                         (human-facing term; Decision 13)
│
├── implementations of BASIS roles        scope: BASIS
│   ├── basis-core
│   ├── basis-gateway
│   ├── basis-identity
│   ├── basis-adapters
│   └── basis-producer
│
├── Basitra-level roles concerned specifically with BASIS        scope: BASIS
│   ├── basis-console                     administers BASIS (consumer; outside BASIS)
│   └── basis-schemas                     publishes BASIS contracts (publication; outside BASIS)
│
└── deployment and distribution tooling   scope: Basitra
    └── basitra-deploy                    (planned; not created)
```

The diagram groups repositories by **scope of concern**. It is not a membership diagram, a dependency graph, a deployment topology, or a distribution manifest. Repository names do not define BASIS membership. `basis-console` and `basis-schemas` sit under BASIS-scoped names and host roles outside BASIS. ADR-0024's diagram and Placement Summary remain the only statement of membership.

### 1. Naming convention: scope of concern

A repository's name has two parts, chosen separately:

1. **Scope prefix.** It records what the repository's durable responsibility concerns.
   - **`basis-`** is used where the repository implements one or more BASIS roles, or where its durable responsibility concerns, administers, serves, or publishes BASIS specifically.
   - **`basitra-`** is used where the repository's durable responsibility concerns Basitra as a whole, or BASIS together with capabilities outside it.
2. **Responsibility word.** It names what the repository owns or what the implementation is, in plain descriptive language. It is chosen per repository under the strategy document's [§10](../architecture/basitra-ecosystem-and-boundary-aware-security.md#10-naming-and-branding-principles) principles: responsibility first, no acronym-driven names, no technology- or topology-bound terms, and no new sub-brands.

Three further rules apply:

- **Durable scope, not history.** A responsibility whose durable scope is Basitra as a whole does not keep or receive a `basis-` name because of historical continuity alone (ADR-0019 Decision 7).
- **No forced uniformity.** A current name that answers the target question well is kept. Change is not desirable in itself, and prefix symmetry is not a reason to rename.
- **Self-describing without the organization.** Names are chosen so that they stay understandable outside their GitHub organization: in forks, package indexes, citations, and search. Unprefixed generic names are not adopted (Alternatives, Convention C).

### 2. The prefix does not decide membership

- **Repository names never determine BASIS membership.** Membership is decided only by ADR-0024 Decision 2, role by role, and is recorded by ADR-0024's Placement Summary and by the ADR that introduces each later role.
- **A `basis-` name may host a role outside BASIS.** It does so where that role's whole object is BASIS (Decisions 8 and 9).
- **A repository's name does not change because a role is placed, colocated, or separated.** It is re-examined only when the repository's durable scope of concern changes.
- **Third-party implementations of BASIS roles are unaffected.** This convention applies to Foundation-maintained repositories. It sets no rule for how third-party work names itself or describes its relationship to BASIS or Basitra (ADR-0024 Decision 7).

### 3. Readings of the prefix resolved

- **R2 (scope of concern) is adopted** as the target reading of `basis-` and `basitra-` for repository and component names.
- **R1 (family label) remains the accurate description of historical and current naming.** Where accepted ADRs, release notes, and existing documents use the `basis-*` family, that usage stays accurate as current naming, read under ADR-0024 Decision 12. R1 is not a target convention.
- **R3 (subsystem membership) is rejected** as the target convention (Alternatives, Convention A).

### 4. `basis-architecture`: target name `basitra-architecture`

**Target name: `basitra-architecture`.**

- Its durable responsibility is architecture and governance for all of Basitra. That includes ecosystem identity (ADR-0019), the Basitra/BASIS boundary (ADR-0024), Basitra-level roles outside BASIS, the boundary with external platforms, and this ADR.
- It defines BASIS as one subsystem within that ecosystem. A `basis-` name scopes the authority that draws the Basitra/BASIS boundary to one side of it, so `basis-architecture` understates its scope.
- `basitra-architecture` states both scope and responsibility. It stays understandable whether the GitHub organization remains `basis-foundation` or later becomes Basitra-named (Decision 16). `basis-foundation/basitra-architecture` reads as the Foundation stewarding Basitra's architecture, which is today's arrangement.
- Plain `architecture` is not selected. It depends on a Basitra-named organization (OD-7), and outside that context it is generic and collides with common names.

This ADR selects the target name only. It does not rename the repository. No package, import namespace, or registry artifact depends on this name.

### 5. `basis-core`: keep

**Keep `basis-core`.**

- It implements the authorization-kernel role, inside BASIS, together with the adapter interface contracts.
- "Core" is positional, not a responsibility word. Read as "the core of BASIS," it is defensible: the kernel is the sole source of the authorization outcome and the one role on the governed path with no substitute.
- `basitra-core` would wrongly suggest that the authorization kernel is the core of the whole ecosystem. That is the confusion ADR-0019 Decision 3 and ADR-0024 Decision 1 already had to correct ("BASIS is not `basis-core` alone").
- Role-explicit alternatives (`basis-kernel`, `authorization-kernel`) do not improve accuracy enough to justify a rename. A repository name is not changed only to make prefixes or vocabulary uniform.

This ADR decides nothing about the `basis-core` distribution name, the `basis_core` import namespace, the dependency specifiers of its dependents, or the import rules in [`kernel-boundary-rules.md`](../kernel-boundary-rules.md). Those are OD-10 concerns (Decision 14).

### 6. `basis-gateway`: keep

**Keep `basis-gateway`.**

- It implements several BASIS roles at one networked trust boundary in front of the kernel: subject establishment, producer admission, composition, its part of context admission, enforcement, and authorization audit evidence.
- "Gateway" names the implementation, which is what this repository is. A role name such as "enforcement" would cover one of six hosted roles.
- The embedded enforcement topology realizes the enforcement role in an adapter host (ADR-0024 Decision 3). This repository is therefore not the exclusive owner of the abstract enforcement role, and a role name would wrongly suggest that it is. The implementation name stays accurate in either topology.
- In OT, "gateway" often means a protocol-conversion gateway. The repository's documentation carries that disambiguation. It is not a reason to rename.

This ADR does not rename or modify the reserved `basis_gateway.*` evidence namespace, the `basis_gateway` import namespace, or the released distribution (Decision 14).

### 7. `basis-identity` and `basis-adapters`: keep

**Keep `basis-identity`.** It implements the identity-engine role, inside BASIS. The name is accurate, not tied to an identity provider or protocol, and durable. It claims the identity engine only. Subject establishment and workload admission remain enforcement-role responsibilities, as ADR-0024 places them.

**Keep `basis-adapters`.** It implements the protocol-adapter role, inside BASIS (ADR-0024 Decision 5), for several protocol families. Third-party extensibility does not change this placement and does not require the first-party repository to give up its BASIS-scoped name. The name claims no exclusivity, so a conforming third-party adapter repository may be named independently without contradicting it. A new protocol family needs no rename.

This ADR does not modify `adapter_source` values, the contracts or examples that carry them, or any retained evidence (Decision 14).

### 8. `basis-console`: keep, as the console for BASIS

**Keep `basis-console`.**

ADR-0024 Decision 6 places the administrative-interface role outside BASIS, as a Basitra-level consumer of BASIS. The name is kept under Decision 1, and its meaning is stated here so that no reader infers membership from the prefix:

```text
basis-console   means   the console for, and administrative interface to, BASIS
                not     a repository whose role is inside BASIS
```

- Every runtime concern an administrative interface can act on today is BASIS state: BASIS-governed configuration, the BASIS administrative context, and authorization and execution evidence. Its scope of concern is BASIS.
- This reading is the one ADR-0024 Decision 12 already applies to ADR-0023's "BASIS administrative interface" and "BASIS user interface": "BASIS" identifies what is administered, not subsystem membership.
- Retention changes nothing in ADR-0023 or ADR-0024 Decision 6. The console gains no standing, authority, or trust from its name. Membership confers none (ADR-0024 Decision 2), and the console's name does not confer membership.

**Reconsideration trigger.** If the console's durable responsibility expands to administer Basitra-wide concerns beyond BASIS, such as deployment, distribution, or other non-BASIS components, its scope of concern becomes Basitra. Its name should then be reconsidered by a later ADR under Decision 1. That ADR would decide the string.

### 9. `basis-schemas`: keep, as the publisher of BASIS contracts

**Keep `basis-schemas`.**

- Contract semantics belong to BASIS. Contract publication is a Basitra-level role, outside BASIS (ADR-0024 Decision 4).
- The repository publishes BASIS contracts. Apart from its own publication-format metadata, its whole content is BASIS contract semantics. Its scope of concern is BASIS, so a `basis-` name is accurate under Decision 1.
- Publication's neutrality is neutrality toward implementations: no single role implementation, first-party or third-party, is the de facto authority over a contract it consumes ([`basis-schemas.md`](../architecture/basis-schemas.md) §2). A BASIS-scoped name does not compromise that neutrality. A conforming third-party implementation of a BASIS role expects to find BASIS contracts under a BASIS-scoped name.
- **"Contracts" versus "schemas."** The repository publishes contracts, of which machine-readable schemas are one part beside lifecycle metadata, examples, and compatibility fixtures. "Contracts" would describe its durable responsibility somewhat more precisely. The improvement is not sufficient to justify a rename. "Schemas" is established in the repository's charter ("architecture proposes, schemas publish"), the threat model, and every component's dependency language.

**Reconsideration trigger.** If this repository begins publishing contracts whose semantics are not BASIS-specific, its scope of concern becomes Basitra-wide, and its name should be reconsidered by a later ADR under Decision 1. If it is renamed for that or any other reason, "contracts" should replace "schemas" in the same rename, not in a second one.

This ADR renames no package, import namespace, contract identifier, schema identifier, or reserved namespace (Decision 14).

### 10. `basis-producer`: keep; ADR-0010 preserved

**Keep `basis-producer`.**

- ADR-0010 establishes `basis-producer` as the permanent, Foundation-maintained repository and component for the operation-producer role, with the architectural name "BASIS Producer." ADR-0024 Decision 12 left that decision semantically unchanged.
- The repository hosts the operation-producer role, including binding creation (ADR-0012), and the producer-intake boundary as that role's ingress (ADR-0018). For the bounded reference topology it also hosts the protocol-executor and execution-evidence-producer roles, colocated (ADR-0011 Decision 3; ADR-0014; ADR-0015). ADR-0011 Decision 5 keeps a separated executor valid, and its Relationship to ADR-0010 states that the producer role keeps the responsibilities ADR-0010 gave it.
- The name follows the permanent role, not every colocated role. Temporary or topology-dependent colocation does not justify renaming the permanent producer implementation. If the executor separates, the name becomes exactly accurate. If colocation persists, the name is incomplete but not misleading. `basis-gateway` follows the same approach: an implementation hosting several roles is named for what it is.
- **"BASIS Producer"** remains its architectural display name. Under Decision 1 it reads as the Foundation-maintained producer implementation of a BASIS role, which is accurate.

**ADR-0010 is preserved. This ADR does not supersede, amend, or edit it.**

**Reconsideration trigger.** The name is re-examined only if accepted architecture makes executor or other runtime responsibilities durably broader than the repository's producer identity. Examples: an accepted ADR that makes executor colocation the durable default beyond the bounded reference topology, or this repository becoming the home of several protocol-specific executor implementations (ADR-0011 Decision 6). Neither condition holds today. A rename under such a trigger would require a superseding ADR for ADR-0010.

### 11. Planned `basis-deploy`: target name `basitra-deploy`

**`basis-deploy` must not become the established repository name. The target name for the planned deployment and distribution tooling repository is `basitra-deploy`.**

- Deployment and distribution tooling is a Basitra-level responsibility, outside BASIS (ADR-0024 Placement Summary).
- It packages and operates across both BASIS and non-BASIS components, including the console and contract publication. Its scope of concern is Basitra. Under a `basis-` prefix, every non-BASIS component it packages would contradict its name.
- The repository has not been established. Nothing is published under the planned name, so there is no compatibility reason to keep it. References to `basis-deploy` in current documents are current-state text, reconciled in normal documentation follow-on work.
- `basitra-deploy` is closest to existing usage and stays understandable regardless of the GitHub organization's name. `basitra-deployment` was a credible alternative and offered no material gain. Unprefixed `deployment` depends on OD-7 and is not selected.

ADR-0010 refers to `basis-deploy` in its ownership list, its rejected alternatives, and its consequences. Those references describe the planned deployment-tooling component by its then-current name. They do not establish a repository or fix a name. A different target name contradicts no ADR-0010 decision, and the references are read as current naming under ADR-0024 Decision 12.

This is naming guidance for a future repository. This ADR does not create the repository or any deployment tooling.

### 12. Repositories without a current role, and historical repositories

**`basis-lab`: no target name is selected until its purpose is established.** This is a resolved naming policy, not an open architecture question, and OD-9 does not stay open because of it.

- `basis-lab` holds a dormant, predecessor-era lab scaffold and has no established role in the target architecture. This ADR invents none.
- If it remains dormant or historical, its existing name may remain as history.
- If it is revived or repurposed, its name is chosen at that time under Decision 1, from the scope of concern of its new responsibility. For example, a lab or test environment spanning BASIS and non-BASIS components would be Basitra-scoped. That choice needs no new naming convention and may be recorded in the PR or decision that establishes the purpose.

**`basis-poc`: keep its historical name.** It is a historical research artifact. It is not renamed, and its historical BASIS terminology, including its README title, is not rewritten (strategy document §10.1, principle 7). Any archival treatment, such as setting GitHub's archived flag or adding a pointer to the current project identity, is a separate stewardship action and is not decided here.

**`basis-foundation/basis`: keep its historical name.** It is a predecessor artifact. It is not renamed. A historical note in its README may eventually be added as a stewardship action. This ADR does not edit that repository.

### 13. Distribution terminology: the Basitra distribution

**The human-facing term *BASIS Core Services Distribution* is replaced, for target terminology, by *the Basitra distribution*.**

Canonical interpretation:

> The Basitra distribution is the Foundation-maintained first-party distribution of Basitra software and supporting components. It may contain implementations of BASIS roles as well as Basitra-level components outside BASIS.

- **Prose form.** The normal form is "the Basitra distribution," with a lowercase *distribution*: it is a descriptive term, not a new proper name or brand. "The distribution" remains an acceptable shorthand where the referent is unambiguous. No acronym is introduced.
- **Why.** ADR-0024 Decision 8 already states that the current term is inaccurate: it begins with "BASIS" but contains components outside BASIS. "Core" collides with `basis-core`, and "Services" does not describe libraries, contract publication, or an interface. "Basitra" states the distribution's actual scope of concern.
- **Membership is unchanged.** Distribution membership and BASIS membership remain separate questions (ADR-0024 Decision 8). A conforming third-party implementation of a BASIS role remains outside the distribution.
- **Stewardship.** "Foundation-maintained" describes present stewardship by the Basis Foundation. The term does not depend on the Foundation's name. If OD-1 changes the Foundation's name or succession, the description of the steward changes and the term does not.
- **No technical artifact.** This decision creates no package, meta-package, release train, container bundle, registry artifact, or technical distribution identifier. Whether such an artifact is ever created, and its name, are OD-10 concerns.
- **History.** Existing uses of *BASIS Core Services Distribution* in accepted ADRs, release notes, the white paper, and other historical records remain accurate for their time and are not rewritten. ADR-0010's "a component of the BASIS Core Services Distribution" is a distribution-membership statement, read under ADR-0024 Decision 12, and needs no supersession. Current canonical documents keep the current term until the follow-on documentation reconciliation updates them.

### 14. Scope of this decision: repositories and display names, not technical identifiers

OD-9, and this ADR, decide:

- repository names;
- component and display names where directly associated with those repositories (for example, "BASIS Producer" for `basis-producer`);
- human-facing distribution terminology.

This ADR does **not** decide:

| Surface | Example | Owner |
| - | - | - |
| Python distribution names | `basis-core` on a package index | OD-10 |
| Python import namespaces | `basis_core`, `basis_gateway`, `basis_identity`, `basis_adapters`, `basis_console`, `basis_schemas`, `basis_producer` | OD-10, under [`compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) |
| Dependency specifiers | `basis-core>=0.2.1,<0.3.0`; `basis-adapters==0.2.0` | OD-10 / migration plan |
| Evidence namespaces | `basis_gateway.*` | OD-10 / contract governance |
| Evidence data values | `adapter_source: basis-adapters:bacnet` | Contract governance; retained evidence is never rewritten |
| Contract and schema identifiers | `decision-request`, `audit-event` | Contract governance |
| Container image and registry artifact names | none published today | OD-10 / migration plan |
| Environment and configuration keys | `BASIS_LOCAL_TOKEN_*` | Compatibility plan |
| URL redirect strategy | repository and organization redirects | Migration plan; OD-7 |

**A repository rename does not by itself change any of these surfaces.** Each changes only through its own owner's decision and the applicable compatibility process. A future repository rename may therefore leave package and import names unchanged at first, and this ADR does not require repository, package, and import names to migrate together. Where they diverge, the divergence is a known, accepted state that OD-10 may later address. It is not a defect that this ADR requires to be corrected.

OD-10 will determine target conventions for these compatibility-bearing technical names, including the import namespaces listed above and the corresponding distribution and package names. This ADR does not prejudge whether they are retained or migrated. In particular, keeping a repository's name here is not a decision to keep its package or import name, and selecting `basitra-` for a repository is not a decision to adopt a `basitra` package prefix.

### 15. No rename by acceptance

Acceptance of this ADR resolves OD-9 by fixing target names. It does not rename, create, archive, or redirect any repository, and it does not change any document's current-state component names. Each change of name, including `basis-architecture` → `basitra-architecture`, is executed only through a separately reviewed, bounded migration plan and PRs. Until then, current names stay current and are used as such.

### 16. Relationship to open decisions

- **OD-7 (GitHub organization) remains open.** The target names are chosen to work whether the organization remains `basis-foundation` or later becomes Basitra-named. `basitra-architecture` and `basitra-deploy` are preferred over generic `architecture` and `deployment` for that reason. This ADR decides neither the organization's name nor its migration. If a repository rename and an organization migration are both planned, whether to sequence them together so links move once is a migration-planning question.
- **OD-1 (Basis Foundation and Basitra) remains open.** This ADR does not decide whether the Basis Foundation is renamed, succeeded, or coexists with another Basitra organization. No repository name here depends on the Foundation's name, and the distribution term describes stewardship without depending on it (Decision 13).
- **OD-10 (packages, distributions, and import namespaces) remains open.** It becomes eligible after this ADR is accepted (Decision 14).
- **OD-3 (Ipotio terminology) remains deferred.** After acceptance, the project and component vocabulary is stable enough for Ipotio to consume, and its reconciliation becomes eligible in Ipotio's repository. This ADR changes nothing in Ipotio.
- **Public site and domain.** Public website naming, domain selection, and domain migration are downstream adoption work (strategy document [§12.5](../architecture/basitra-ecosystem-and-boundary-aware-security.md#125-stage-5--public-artifact-naming)). This ADR selects, reserves, and claims no domain, and depends on none.

### 17. Continuity with accepted ADRs

This ADR supersedes no ADR, modifies no accepted ADR's body, and changes no ADR's status.

| ADR | Effect |
| - | - |
| [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md) | Remains `Accepted` and unchanged. Its permanent name `basis-producer`, permanent repository, and architectural name "BASIS Producer" are kept (Decision 10). Its references to `basis-deploy` and to the BASIS Core Services Distribution are current naming and distribution-membership statements, read under ADR-0024 Decision 12. A different target name for the deployment repository or the distribution term contradicts no ADR-0010 decision. |
| [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) | Unchanged and applied. Basitra names no component; a `basitra-` scope prefix on a repository names its scope, not a component called Basitra. Decision 7 is applied by Decision 1's "durable scope, not history" rule. |
| [ADR-0023](0023-supervisory-platform-and-administrative-interface-boundary.md) | Unchanged. Its "BASIS administrative interface" phrasing is consistent with Decision 8. |
| [ADR-0024](0024-long-term-scope-of-basis-within-basitra.md) | Unchanged and applied. Every placement is taken from its Placement Summary. Decision 8's transitional distribution name is resolved for human-facing terminology by Decision 13 here. Decision 12's reading rule continues to apply to accepted ADR bodies. |
| Other accepted ADRs | Use current component names descriptively. Those uses remain accurate as current naming and are not edited. |

Because `basis-producer` is kept, this ADR does not need to resolve the whole-ADR supersession gap that the naming assessment identified for a rename of a name fixed inside a multi-decision ADR ([naming assessment §9](../architecture/basitra-repository-and-component-naming-assessment.md#9-adr-0010-naming-constraint)). That process question remains available to a later decision if it is ever needed.

## Target Naming Summary

| Current repository or concept | Scope of concern | BASIS membership of hosted roles (ADR-0024) | Target treatment | Decision |
| - | - | - | - | - |
| `basis-architecture` | Basitra | Outside (Basitra level) | **`basitra-architecture`** (target; not renamed by this ADR) | 4 |
| `basis-core` | BASIS | Inside | **Keep** | 5 |
| `basis-gateway` | BASIS | Inside | **Keep** | 6 |
| `basis-identity` | BASIS | Inside | **Keep** | 7 |
| `basis-adapters` | BASIS | Inside | **Keep** | 7 |
| `basis-console` | BASIS | Outside (Basitra-level consumer of BASIS) | **Keep**, read as the console for BASIS; reconsider if its scope becomes Basitra-wide | 8 |
| `basis-schemas` | BASIS | Publication outside; published semantics inside | **Keep**; reconsider if it publishes non-BASIS contracts | 9 |
| `basis-producer` | BASIS | Inside | **Keep** (ADR-0010 preserved); reconsider only on a durable broadening of runtime responsibilities | 10 |
| `basis-deploy` (planned) | Basitra | Outside (Basitra level) | **`basitra-deploy`** when created; `basis-deploy` is not adopted | 11 |
| `basis-lab` | None established | None | **No target name until a purpose is established**; name by Decision 1 then | 12 |
| `basis-poc` | Historical | None | **Keep historical name** | 12 |
| `basis-foundation/basis` | Historical | None | **Keep historical name** | 12 |
| *BASIS Core Services Distribution* | Basitra | Not a membership concept | **"the Basitra distribution"** (human-facing term; no technical artifact) | 13 |
| `basis-foundation` organization | — | — | Not decided (OD-7) | 16 |
| Basis Foundation | — | — | Not decided (OD-1) | 16 |

## Alternatives Considered

The four conventions analyzed in [naming assessment §12](../architecture/basitra-repository-and-component-naming-assessment.md#12-candidate-convention-analysis) were evaluated. Under each, package and import names would not change with a repository rename; any such change would be a separate OD-10 decision.

**Convention A: the prefix encodes BASIS membership (R3).** `basis-` only on repositories that implement BASIS roles; `basitra-architecture`, `basitra-console`, `basitra-contracts`, and `basitra-deploy` for the rest. Rejected as the target model:

- membership is a property of roles, not repositories (ADR-0024 Decision 7), and repositories may host several roles. A repository hosting both a BASIS role and a non-BASIS role would have no correct prefix;
- `basis-console` and `basis-schemas` accurately concern BASIS without implementing BASIS roles. ADR-0024 Decision 12 already reads "BASIS" in an interface's name as its object, not as membership;
- it turns every architectural placement change into a naming change;
- third-party implementations of BASIS roles never carry the prefix, so the signal would be incomplete by design.

**Convention B: `basitra-` on every first-party repository.** Rejected:

- `basitra-core` would imply that the authorization kernel is the core of the whole ecosystem;
- BASIS, the subsystem ADR-0024 defines, disappears from repository naming;
- it risks reading as "Basitra components," against ADR-0019's statement that Basitra names no component;
- it renames every repository, with wide link and automation churn, for uniformity rather than accuracy. It would not require any package or import migration, but every repository name would then differ from its package name until OD-10 decided otherwise.

**Convention C: generic responsibility names without an ecosystem prefix** (`architecture`, `identity`, `console`, `contracts`, and so on). Rejected as the current target:

- the names lose all ecosystem context outside a Basitra-named organization: in `basis-foundation`, forks, package indexes, citations, and search;
- common names have poor discoverability and collide with other repositories;
- it makes repository naming depend on OD-7;
- it renames every repository. As with B, package and import names would not change with it.

**Convention D: hybrid, responsibility and scope-of-concern names (R2).** Selected. It best satisfies the criteria the naming assessment applied:

- **Architectural accuracy.** It renames exactly the names that are inaccurate under any non-legacy reading (`basis-architecture`, the planned `basis-deploy`, and the distribution term) and keeps the rest.
- **Role and repository separation.** Membership stays with roles. A name changes only when a repository's subject changes, not when roles are colocated or separated.
- **Durability.** It survives the `basis-producer` executor question, a separated executor, new protocol families, and third-party implementations without renaming.
- **Ecosystem clarity.** `basitra-` marks ecosystem-wide responsibilities, and `basis-` keeps BASIS visible in the repository layer, which Convention B loses.
- **Compatibility restraint.** It has the lowest churn of any convention consistent with accuracy, and it works under either OD-7 outcome.
- **Contributor comprehension.** The rule is short and can be applied by a reviewer. Its residual risk, a reader inferring that `basis-console` or `basis-schemas` is inside BASIS, is addressed explicitly by Decisions 2, 8, and 9 and carries no security consequence (ADR-0024 Decision 2).

**Keep `basis-architecture` until OD-7.** Rejected. The name misstates the scope of the repository that defines the Basitra/BASIS boundary, and `basitra-architecture` is correct under either OD-7 outcome. Waiting would add nothing to the decision. Sequencing the actual rename with OD-7 remains open to the migration plan (Decision 16).

**Keep *BASIS Core Services Distribution*, or narrow it to the BASIS-role components.** Rejected. ADR-0024 Decision 8 states that the current term is inaccurate. Narrowing it would invent a new packaging concept to save a name and would separate the console, contract publication, and deployment tooling from the set the Foundation maintains.

**Retire the distribution's proper name entirely** ("the Foundation-maintained distribution"). Considered. Rejected as the primary term because it ties the distribution's name to the Foundation's name, which OD-1 may change. "The Basitra distribution" is a descriptive term, not a brand, so it gains the plainness of this option without that dependency.

**Rename `basis-schemas` to `basis-contracts` now.** Rejected. The accuracy gain is real but small, and it does not justify a rename on its own (Decision 9).

**Rename `basis-producer` to reflect colocated execution** (for example `basis-runtime` or `basis-operation-runtime`). Rejected. It would bind the name to a colocation that accepted architecture frames as topology-dependent, would be vaguer than "producer," and would require superseding ADR-0010 (Decision 10).

**Decide `basis-lab` now as archived.** Rejected. No architecture document establishes its purpose. Choosing archival or a target name now would invent a role. Decision 12 makes the treatment rule-based instead.

## Consequences

### Positive

- **OD-9 is resolved** by this ADR's acceptance. Every current and planned repository, and the distribution term, has a decided target treatment, and future repositories can be named by rule.
- **Basitra becomes visible where the concern is ecosystem-wide.** That is the architecture repository, deployment tooling, and the distribution. BASIS stays visible where the concern is BASIS.
- **BASIS remains a permanent architectural subsystem of Basitra.** Five of seven current component repositories keep names that are accurate under every reading of the prefix.
- **No accepted ADR is superseded.** ADR-0010 remains `Accepted` and unchanged, so the whole-ADR supersession gap does not have to be solved for OD-9.
- **Naming and membership stay separate.** A role reassignment, colocation, or separation is not a naming event.
- **Compatibility surfaces are protected.** No package, import, contract, schema, or evidence identifier moves because of a repository decision.

### Negative / Tradeoff

- **Two prefixes coexist in one organization.** This is less uniform than Convention B, and readers must learn that `basis-` means "concerns BASIS."
- **`basis-console` and `basis-schemas` can still be misread as BASIS members.** Documentation must state their placement. This ADR does so explicitly. There is no security consequence, because membership confers nothing.
- **Repository and package names may diverge.** No package depends on the `basis-architecture` name, so its rename creates no divergence. If a later decision renames a component repository whose package and import names OD-10 keeps, the two would differ. That state is accepted until OD-10 decides (Decision 14).
- **Current-state documents keep pre-OD-9 terminology** until bounded follow-on reconciliation updates them. Examples are the BASIS Core Services Distribution, `basis-deploy`, and `basis-architecture` as the architecture repository.
- **Reconsideration triggers require attention.** The scopes of `basis-console` and `basis-schemas`, and the runtime breadth of `basis-producer`, must be re-checked when their responsibilities change.

### Security Consequences

- The decision moves no authority, credential, admission, or trust, and changes no workload identity. ADR-0022 ties logical workload identities to roles and scope, not to repository names. A migration plan confirms this for any rename.
- It closes a reading that could otherwise grow: that a `basis-` name, or being in the Basitra distribution, implies standing on the governed path. Neither does.

## Follow-On Work

Each item needs its own bounded work. Some may need more than one PR. Item 1, the formal acceptance review that moved this ADR from `Proposed` to `Accepted`, is complete. Items 2 onward did not begin while this ADR was `Proposed`; with it accepted, each is now eligible as a separate bounded workstream, and none is performed by the acceptance:

1. **Formal acceptance review** of this ADR through independent architecture review. Complete.
2. **OD-10: package, distribution, import-namespace, and compatibility-bearing technical naming**, in coordination with OD-1 and OD-7 (Decision 14).
3. **Repository migration plan** for the names this ADR changes:
   - `basis-architecture` → `basitra-architecture`, including link, badge, automation, and redirect verification;
   - planned `basis-deploy` → `basitra-deploy`, applied before the repository is created;
   - the human-facing distribution term → "the Basitra distribution."
4. **Current-documentation reconciliation** of current-state text that still presents pre-ADR-0024 or pre-OD-9 terminology. This includes ADR-0024's listed follow-on documents ([`writing-guidelines.md`](../standards/writing-guidelines.md) §4.1, the glossary, [`terminology-rules.md`](../standards/terminology-rules.md), [`basis-ecosystem.md`](../architecture/basis-ecosystem.md), and the strategy document §3.1 and §8), the README, and the placement notes for `basis-console` and `basis-schemas` that Decisions 8 and 9 call for.
5. **OD-7: GitHub organization migration**, addressed independently when appropriate.
6. **OD-3: Ipotio terminology reconciliation**, in Ipotio's repository, after names are stable.
7. **Public website and domain migration**, in its own bounded workstream.

## Non-Goals

This ADR does not:

- rename `basis-architecture` or any other GitHub repository;
- create `basitra-architecture`, `basitra-deploy`, or any deployment tooling;
- migrate the `basis-foundation` organization or rename the Basis Foundation;
- rename Python packages, import namespaces, dependency specifiers, schema or contract identifiers, or evidence namespaces;
- modify `adapter_source` values, contracts, schemas, package metadata, GitHub Actions, or implementation code;
- create redirects;
- change Ipotio;
- select, purchase, reserve, or configure a domain, or migrate a website;
- define BASac;
- resolve OD-1, OD-7, or OD-10;
- supersede or amend ADR-0010 or any other ADR;
- change any placement made by ADR-0024.

## References

- [`basitra-repository-and-component-naming-assessment.md`](../architecture/basitra-repository-and-component-naming-assessment.md): §4 (repositories versus roles), §5 (criteria and the three readings), §6 (repository assessments), §7 (distribution and organization), §9 (ADR-0010 constraint), §10 (compatibility surfaces), §11 (coherence), §12 (conventions), §13 (recommendation)
- [ADR-0024](0024-long-term-scope-of-basis-within-basitra.md): Decisions 1–2, 4–8, 12; Placement Summary; Follow-On Work
- [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md): Decisions 1, 3, 6, 7
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md): permanent component name and repository; Distribution Membership; Naming Alternatives Considered
- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md), [ADR-0012](0012-authorization-to-execution-binding.md), [ADR-0014](0014-minimum-execution-evidence-semantics.md), [ADR-0015](0015-first-bounded-execution-target.md), [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md), [ADR-0022](0022-workload-credential-lifecycle-and-scope-boundary.md), [ADR-0023](0023-supervisory-platform-and-administrative-interface-boundary.md)
- [`basitra-ecosystem-and-boundary-aware-security.md`](../architecture/basitra-ecosystem-and-boundary-aware-security.md): §10, §12.4, §12.5, §13
- [`basis-ecosystem.md`](../architecture/basis-ecosystem.md), [`basis-schemas.md`](../architecture/basis-schemas.md), [`basis-console.md`](../architecture/basis-console.md), [`basis-gateway.md`](../architecture/basis-gateway.md), [`ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md)
- [`compatibility-philosophy.md`](../architecture/compatibility-philosophy.md), [`kernel-boundary-rules.md`](../kernel-boundary-rules.md)
- [`terminology-rules.md`](../standards/terminology-rules.md), [`writing-guidelines.md`](../standards/writing-guidelines.md), [`glossary.md`](../glossary.md)
- [`docs/adr/README.md`](README.md): lifecycle states and the supersession rule
- [`GOVERNANCE.md`](../../GOVERNANCE.md)
