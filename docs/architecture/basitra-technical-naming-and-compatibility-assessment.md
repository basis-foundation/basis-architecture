# Basitra Technical Naming and Compatibility Assessment

> **Status: Non-normative assessment (OD-10).** This assessment informs the future OD-10 decision. OD-10 remains open. The document makes recommendations only. It changes no technical identifier and authorizes no migration.

It examines the compatibility-bearing technical names that [ADR-0025](../adr/0025-basitra-repository-and-component-naming.md) Decision 14 deliberately left undecided. It creates no ADR, resolves none of OD-1, OD-7, or OD-10, supersedes no accepted decision, and modifies no implementation repository, contract, schema, or published artifact.

---

## Contents

1. [Status and Purpose](#1-status-and-purpose)
2. [Evidence Rules and Statement Labels](#2-evidence-rules-and-statement-labels)
3. [Summary of Findings](#3-summary-of-findings)
4. [Governing Principles](#4-governing-principles)
5. [Surface Taxonomy](#5-surface-taxonomy)
6. [Inventory Matrix](#6-inventory-matrix)
7. [Release and Publication State](#7-release-and-publication-state)
8. [Python Distribution Naming](#8-python-distribution-naming)
9. [Python Import Namespace Naming](#9-python-import-namespace-naming)
10. [`basis-schemas` and Contract Publication](#10-basis-schemas-and-contract-publication)
11. [Evidence Namespaces and Durable Values](#11-evidence-namespaces-and-durable-values)
12. [Environment Variables and Configuration Keys](#12-environment-variables-and-configuration-keys)
13. [CLI and Executable Names](#13-cli-and-executable-names)
14. [Container and Registry Naming](#14-container-and-registry-naming)
15. [Distribution Terminology Versus Python Distributions](#15-distribution-terminology-versus-python-distributions)
16. [Relationship to `basitra-architecture`](#16-relationship-to-basitra-architecture)
17. [Relationship to Planned `basitra-deploy`](#17-relationship-to-planned-basitra-deploy)
18. [Relationship to OD-1 and OD-7](#18-relationship-to-od-1-and-od-7)
19. [External Registry Availability](#19-external-registry-availability)
20. [Third-Party Ecosystem Implications](#20-third-party-ecosystem-implications)
21. [Security and Supply-Chain Considerations](#21-security-and-supply-chain-considerations)
22. [Candidate OD-10 Policies](#22-candidate-od-10-policies)
23. [Recommendation (Non-Normative)](#23-recommendation-non-normative)
24. [Answers to the Required Questions](#24-answers-to-the-required-questions)
25. [Open Questions and ADR Boundary](#25-open-questions-and-adr-boundary)
26. [Proposed Follow-On Sequence](#26-proposed-follow-on-sequence)
27. [What This Assessment Does Not Do](#27-what-this-assessment-does-not-do)
28. [References](#28-references)

---

## 1. Status and Purpose

[ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) (`Accepted`) resolved OD-8: BASIS is the Boundary-Aware Security subsystem of Basitra that governs operations along the governed path, with membership attached to roles. [ADR-0025](../adr/0025-basitra-repository-and-component-naming.md) (`Accepted`) resolved OD-9: repository and component names record scope of concern, `basis-` for BASIS and `basitra-` for Basitra as a whole.

ADR-0025 Decision 14 separates those names from surfaces with their own compatibility characteristics, and states that a repository rename changes none of them. OD-10 now evaluates those surfaces. This document is the evidentiary input to that decision. It takes ADR-0019, ADR-0024, and ADR-0025 as fixed and does not reopen them.

The question, adapted from the strategy document's [Stage 4 test](basitra-ecosystem-and-boundary-aware-security.md#124-stage-4--basitra-target-architecture-and-repository-naming-reconciliation):

> If Basitra had been the project identity from the beginning but BASIS had still existed as the governed security subsystem, what should these technical identifiers be called, and which existing names are valuable enough to preserve?

Each surface is analyzed on its own. A cost counts only where that surface actually changes.

---

## 2. Evidence Rules and Statement Labels

### 2.1 Evidence

- **Inspection date: 2026-11-08.** Remote default branches must be reverified before any migration.
- **Local checkouts inspected:** `basis-architecture` (`3e3473a`), `basis-core` (`7a92497`), `basis-gateway` (`1e08348`), `basis-identity` (`2d9490f`), `basis-adapters` (`f5f6202`), `basis-console` (`969bd67`), `basis-schemas` (`ec43a2f`), `basis-producer` (`8c7410a`). Each of the seven component checkouts matched its GitHub `main` head on the inspection date. Historical repositories `basis`, `basis-poc`, and `basis-lab` and the `ipotio/architecture` documents were inspected for references only. No `basitra-*` or deployment repository exists locally.
- **Read-only registry and platform checks on the inspection date:** the PyPI JSON API, the GitHub API (repository, release, and asset metadata), and unauthenticated GHCR and Docker Hub lookups.
- A `pyproject.toml` version shows a **version declared locally**. Only a registry or release record shows a **published release verified**.
- Line references are to the inspected commits and may drift.

### 2.2 Statement labels

| Label | Meaning |
| - | - |
| **Observed** | Current state, traceable to a file, API response, or release record |
| **Accepted** | Fixed by an accepted ADR or canonical document |
| **Inference** | This assessment's reasoning from observed and accepted facts |
| **Recommended** | What this assessment proposes OD-10 decide. Not a decision |
| **Unresolved** | A question this assessment leaves open |
| **Migration** | A consideration for a later migration plan, if a name ever changes |

`Not verified` and `Requires migration-phase verification` mark incomplete evidence.

---

## 3. Summary of Findings

1. **Observed.** No first-party Python package is on a public package index. All seven distribution names appear in `pyproject.toml`, but "released" here means GitHub Releases at most ([Section 7](#7-release-and-publication-state)).
2. **Observed.** `basis-core` on PyPI is an **unrelated third-party project** (Functional Data Pipelines, one release, 2021). `basis-core`'s own README instructs `pip install basis-core`, which installs that project. `basis-gateway`'s README already documents the resulting failure.
3. **Inference.** The publication state cuts two ways. Renaming a distribution costs far less than a published-package assumption suggests. But keeping `basis-*` cannot be extended to a public index as it stands, because `basis-gateway` and `basis-producer` declare a dependency on a name someone else controls there.
4. **Observed.** Import namespaces are the most heavily used code surface: 494 first-party `from`/`import` source lines, a documented stable public API in `basis-core`, and import-boundary tests that hardcode the names.
5. **Observed.** The ecosystem has no general `BASIS_*` configuration convention. Most settings are unprefixed. Only eight prefixed variables were found, two of them test harness settings ([Section 12](#12-environment-variables-and-configuration-keys)).
6. **Observed.** No container image build definition and no CLI entry point exists in any of the seven component repositories.
7. **Observed.** `basis-adapter-evidence-v1` sits inside digested evidence material, and `adapter_source` sits inside a submission covered by `binding_digest`. Changing either alters what is digested, which goes beyond a rename.
8. **Observed.** `basis-architecture` has no package or import name, but 20 published contracts declare `governed_by: basis-architecture`, and tests assert it. The repository's accepted rename interacts with contract metadata ([Section 16](#16-relationship-to-basitra-architecture)).

---

## 4. Governing Principles

Principles 4.1, 4.5, and 4.6 follow from accepted documents. The others are this assessment's proposals for OD-10 to adopt.

**4.1 Architecture before technical naming.** ADR-0019 Decision 7 makes history an input to migration and not a constraint on the target. Technical names should reflect accepted architecture. Compatibility obligations then bear on *whether and when* a name moves, and do not define what the correct name is.

**4.2 Public identity and technical identity may differ.** A repository may be `basitra-architecture` while Python packages elsewhere remain `basis_core`. ADR-0025 Decision 14 accepts this divergence. Prefix uniformity is not a goal.

**4.3 Stable imports are APIs.** An import namespace change is a breaking API change. The strategy document's [Stage 5 table](basitra-ecosystem-and-boundary-aware-security.md#125-stage-5--public-artifact-naming) says so, and `basis-core/docs/public-api.md` specifies import paths as part of its stable tier.

**4.4 Published artifacts carry stronger obligations, by degree.** Evidence here requires grading this principle. In ascending cost: declared only, tagged, GitHub Release without artifacts, GitHub Release with wheels, public index. A name's obligations come from what actually happened to it.

**4.5 Evidence is historical truth.** Persisted evidence is not rewritten for branding. [ADR-0025](../adr/0025-basitra-repository-and-component-naming.md) Decision 14 and [`compatibility-philosophy.md`](compatibility-philosophy.md#why-audit-continuity-matters) both establish this.

**4.6 Machine-readable identifiers should not churn for branding alone.** Compatibility philosophy treats field and identifier renames as breaking, and requires a major version, an ADR, and a deprecation path ([Breaking Change Expectations](compatibility-philosophy.md#breaking-change-expectations)). An accurate identifier with compatibility value stays unless a concrete reason exists.

**4.7 Avoid dual naming without a purpose.** A name such as `basitra-basis-*` needs an architecture or registry reason. [Section 8](#8-python-distribution-naming) finds one registry reason, confined to one boundary.

**4.8 No migration only for visual symmetry.** Consistency has value, but not at the cost of needless breakage.

**4.9 A flat global registry is a different context from a repository (added by this assessment).** Under an organization, `basis-core` is qualified by its owner. On a flat index it is not, and "basis" is a common word. ADR-0025's "self-describing without the organization" reasoning (Decision 1) applies here with more force.

---

## 5. Surface Taxonomy

The suggested classes fit the evidence. One class is added: **G**, digest-bound identifiers, because the cost of changing them differs in kind from Class D.

| Class | Surface | Observed examples | Cost of change |
| - | - | - | - |
| **A** Human-facing | Descriptions, display titles, prose | `description` fields; `title: BASIS Decision Request`; FastAPI `title` | Low |
| **B** Discoverability | Distribution and image names | `basis-core` in `pyproject.toml` and wheel metadata | Depends on publication grade |
| **C** Code and API | Imports, ASGI entry points, logger names | `basis_core`; `basis_gateway.main:app`; `logging.getLogger("basis_core.audit")` | Breaking |
| **D** Contract and evidence | Evidence namespaces, contract metadata, wire headers, token claims | `basis_gateway.*`; `governed_by`; `X-BASIS-Producer-Client-Cert`; `basis.session_id` | Breaking for interoperability, and history is immutable |
| **E** Operational configuration | Environment variables, config values, service names | `BASIS_LOCAL_TOKEN_ISSUER`; `AUTH_MODE=basis_local_token`; `SERVICE_NAME` | Depends on deployment use |
| **F** Location | Repository URLs, registry paths, schema `$id` | `github.com/basis-foundation/...`; `https://basis-core/schemas/...` | Varies. Redirects cover some |
| **G** Digest-bound (added) | Values inside digested or bound material | `evidence_profile: basis-adapter-evidence-v1`; `adapter_source` via `binding_digest` | A change alters the digest. It is a new profile or format, never a rename |

---

## 6. Inventory Matrix

**Observed**, from each repository's `pyproject.toml`, `src/` layout, and tracked files. All seven expected distribution and import names were verified. All use `hatchling` with a single `src/<import>` wheel package. None declares `[project.scripts]` or entry points.

| Repository | Repository target (ADR-0025) | Distribution / version declared | Import | Other public identifiers | Published? | Sensitivity | Candidate treatment |
| - | - | - | - | - | - | - | - |
| `basis-core` | keep | `basis-core` 0.2.1 | `basis_core` | logger names; legacy `$id` `https://basis-core/schemas/*`; stable public API tiers | GitHub Release with wheel. PyPI name is foreign | Highest: kernel; all dependents | Keep name. Do not publish to a public index under it |
| `basis-gateway` | keep | `basis-gateway` 0.2.0 | `basis_gateway` | `basis_gateway.*` evidence; `basis_gateway.main:app`; `X-BASIS-Producer-Client-Cert`; `BASIS_LOCAL_TOKEN_*`; `/health` `service` | GitHub Release with wheel | High | Keep |
| `basis-identity` | keep | `basis-identity` 0.1.0 | `basis_identity` | `basis` claim object; token type `basis-local-identity` | GitHub Release, no artifact | Medium | Keep |
| `basis-adapters` | keep | `basis-adapters` 0.2.0 | `basis_adapters` | `basis-adapter-evidence-v1`; `adapter_source` label source; mapping `$id` values | GitHub Release with wheel | High: digest-bound | Keep |
| `basis-console` | keep | `basis-console` 0.2.0 | `basis_console` | `BASIS_CONSOLE_MODE`; `basis_console.main:app` | GitHub Release with wheel | Low to medium | Keep |
| `basis-schemas` | keep | `basis-schemas` 0.2.2 | `basis_schemas` | `published_by`; 20 contract names; metadata-only wheel | GitHub Release, no artifact | High for contracts, low for the package | Keep |
| `basis-producer` | keep | `basis-producer` 0.1.0 | `basis_producer` | composes `adapter_source`; `BASIS_GATEWAY_REPO_ROOT` (test harness) | None. Private repository, no release | Low today. Name fixed by ADR-0010 at repository level | Keep |
| `basis-architecture` | `basitra-architecture` | none | none | `governed_by` value in contracts ([Section 16](#16-relationship-to-basitra-architecture)) | v1.0.0 tag; documents only | Low, one interaction | Outside package scope |

**Dependency specifiers (Observed, all of them):**

| Dependent | Specifier |
| - | - |
| `basis-gateway` | `basis-core>=0.2.1,<0.3.0` |
| `basis-producer` | `basis-core>=0.2.1,<0.3.0` |
| `basis-producer` | `basis-adapters==0.2.0` |

No extras reference a first-party package. Nothing depends on the `basis-schemas`, `basis-identity`, or `basis-console` distributions. Dependency specifiers are a small surface today.

**Package metadata (Observed).** `authors = Basis Foundation` appears in `basis-identity` and `basis-schemas` only. `[project.urls]` pointing at `github.com/basis-foundation/...` appears in `basis-identity`, `basis-adapters`, and `basis-console`.

**CI (Observed).** GitHub Actions references in `basis-gateway` and `basis-producer` use `${{ github.repository_owner }}/basis-core` sibling checkouts. They are owner-relative and depend on repository leaf names. `basis-console` and `basis-architecture` have no workflows. No workflow publishes packages or images.

**Consumer-copyable material (Observed).** `basis-core/README.md:40` and `basis-schemas/README.md:480` mention `pip install <name>`. Every other repository documents `pip install -e` from a checkout.

---

## 7. Release and Publication State

**Observed**, 2026-11-08.

| Package | Version declared | Latest GitHub Release | Wheel attached | Asset downloads (wheel / sdist) | PyPI `basis-*` response |
| - | - | - | - | - | - |
| `basis-core` | 0.2.1 | v0.2.1, 2026-08-01 | Yes | 1 / 1 | **200: unrelated project** |
| `basis-gateway` | 0.2.0 | v0.2.0, 2026-08-04 | Yes | 0 / 0 | 404 |
| `basis-identity` | 0.1.0 | v0.1.0, 2026-07-08 | No | n/a | 404 |
| `basis-adapters` | 0.2.0 | v0.2.0, 2026-08-09 | Yes | 4 / 5 | 404 |
| `basis-console` | 0.2.0 | v0.2.0, 2026-08-08 | Yes | 0 / 0 | 404 |
| `basis-schemas` | 0.2.2 | v0.2.2, 2026-07-23 | No | n/a | 404 |
| `basis-producer` | 0.1.0 | None (private repository) | No | n/a | 404 |

Further observations for each package:

- **The PyPI `basis-core` project.** Version 0.1.0, 2021-07-03, summary "Functional Data Pipelines", a different author. Its wheel ships a top-level `basis` package and a `basis` console script, and requires `pandas`, `pyarrow`, and `sqlalchemy`. It does not ship `basis_core`, so `import basis_core` fails after installing it.
- **`basis` on PyPI** is a second unrelated project (NLP tooling, 2017).
- **First-party acknowledgment.** `basis-gateway/README.md:506` documents `pip` fetching "the unrelated PyPI package" with `numpy`, `pandas`, and `pyarrow` compile errors. `basis-producer/.github/workflows/ci.yml:30` states that `basis-core` is not published to PyPI.
- **Install-by-name guidance that is wrong today.** `basis-core/README.md:40` says `pip install basis-core`. `basis-core/docs/v0.1-readiness-review.md:170` lists a PyPI installability check as a release item. `basis-schemas/README.md:480` describes `pip install basis-schemas`, which has no public index target.
- **External usage.** Stars and forks: at most 1 star and 0 forks per repository. A GitHub code search for `basis_core` outside the organization returned only unrelated C and C++ hits, and it is not exhaustive. The GitHub dependents endpoint returned 404. **External adoption could not be established and cannot be excluded.** Download counts are consistent with first-party use, but do not prove it.
- **Documented import paths.** `basis-core`, `basis-adapters`, and `basis-identity` each have `docs/public-api.md`. Only `basis-core` declares a stable tier. `basis-adapters` states that its API is "pre-v1 and subject to change".
- **Would a rename need dual publication?** On a public index, no first-party name exists to transition from. For GitHub Release wheels, old releases simply remain under their old names.
- **Would abandoning a name create squatting risk?** On a public index nothing first-party would be abandoned. The risk already exists in the opposite direction: the names in use are not held ([Section 21](#21-security-and-supply-chain-considerations)).

A GitHub Release of `basis-console` v0.1.0 is titled "basis-control v0.1.0", which shows earlier naming drift. Historical release titles are not naming inputs.

---

## 8. Python Distribution Naming

**Inference.** A distribution name is not an import name. Python's packaging model allows them to differ, and the difference is common across the ecosystem. Every statement below concerns the `Name` field only.

| Policy | Accuracy | Effect | Verdict |
| - | - | - | - |
| **A. Keep all `basis-*`** | Accurate under ADR-0025's scope-of-concern reading | No change. Sound on every channel the project controls. **Fails at the public index:** `basis-core` is held there by another party, and PyPI does not accept direct-URL dependencies, so `basis-gateway` and `basis-producer` could not be published with an installable dependency | Keep for current channels. Insufficient alone for public indexes |
| **B. `basitra-core` etc.** | Inaccurate. Basitra names no runtime component ([ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) Decision 1), and `basitra-core` would claim the kernel is the core of the whole ecosystem. It erases the BASIS boundary that ADR-0024 fixed | Renames all with no accuracy gain | Reject |
| **C. `basitra-basis-core` everywhere** | Accurate but redundant. No benefit on channels where the owner already qualifies the name | Awkward as a general convention. **Useful at exactly one boundary:** a flat global index, where it adds provenance and avoids the collision. OD-8 has since settled the concern the [Stage 5 table](basitra-ecosystem-and-boundary-aware-security.md#125-stage-5--public-artifact-naming) raised about this pattern | Reject as a general convention. Retain as the public-index form |
| **D. Mixed by scope of concern** | Accurate, and mirrors ADR-0025 without copying it mechanically | Same as A for current packages, and gives the rule for future ones (`basitra-deploy`). Inherits A's public-index failure | Adopt as the base convention |

**Public-index options if publication is ever chosen:**

| Option | For | Against |
| - | - | - |
| **I1. Stay off public indexes** | The current state. Wheels with `SHA256SUMS` on GitHub Releases suit OT and air-gapped deployment | Harder discovery; `pip install` by name stays wrong unless documentation is fixed |
| **I2. PEP 541 transfer of `basis-core`, then publish `basis-*`** | Keeps one name everywhere | Outcome is not predictable. It depends on PyPI's process and the current owner. Six other names remain generic and unprotected. Prior third-party history under the name persists |
| **I3. Publish one exception (`basis-core` under another name, others unchanged)** | Minimal | Leaves the family inconsistent, and users would extrapolate `pip install basis-core` from `basis-gateway` |
| **I4. Family-wide `basitra-basis-<component>` at the index boundary, imports unchanged** | Avoids the collision, gives distinctive provenance, stays accurate under ADR-0024. Migration changes three specifiers and no imports | Longer names; name differs from import name; availability unverified |

**Recommended.** Adopt Policy D as the convention. Keep `basis-*` for every existing distribution on every channel the project controls. Make public-index publication a separately gated act: no first-party package is published to a public index under a bare `basis-*` name until OD-10's rule is satisfied. If publication is chosen, prefer **I4**, applied to the whole BASIS-scoped family at once. Basitra-wide packages stay `basitra-<name>` without double scoping. Confidence: **high** on keep and on the gate, **medium** on I4 over I2.

**Rule against dual identity.** A built wheel has one `Name`. Each component should have one distribution name at a time across channels. It should not ship `basis-core` on GitHub Releases and `basitra-basis-core` on an index for the same version.

**Migration.** Under I4, the rename would land at a single release. Only the three specifiers change. Compiled wheel filenames change. Import paths do not. Old GitHub Release wheels remain as history.

---

## 9. Python Import Namespace Naming

**Observed.** First-party source lines beginning `from basis_*` or `import basis_*` in `src/`:

| Repository | Own namespace | Cross-repository |
| - | - | - |
| `basis-core` | 102 | none |
| `basis-gateway` | 77 | 24 (`basis_core`) |
| `basis-identity` | 138 | none |
| `basis-adapters` | 91 | none |
| `basis-console` | 32 | none |
| `basis-producer` | 22 | 8 (`basis_core`, `basis_adapters`) |
| `basis-schemas` | 0 | none |

Tests, documentation, and configuration add more. Names are hardcoded in import-boundary enforcement (`basis-core/tests/test_import_boundaries.py:661`, `basis-console/tests/test_adr0023_console_boundary.py:69`), in `mypy` and `hatch` configuration, and in [`kernel-boundary-rules.md`](../kernel-boundary-rules.md). Logger names follow the import path, so log-filtering configurations inherit it.

**Inference on the honest counterargument.** Every distribution version is pre-1.0, and external consumers cannot be shown. Limited adoption makes this the cheapest time a rename will ever have, although a pre-1.0 package version does not itself relax any API compatibility obligation ([Section 24](#24-answers-to-the-required-questions)). That only matters if a better name exists.

| Option | Architectural accuracy | Cost | Verdict |
| - | - | - | - |
| **A. Keep `basis_*`** | Accurate. `basis_core` holds a BASIS role. `basis_console` and `basis_schemas` follow ADR-0025's scope-of-concern reading | None | **Recommended** |
| **B. `basitra_*`** | Less accurate, for the reason given under Policy B | Breaking in every repository, with no accuracy gain | Reject |
| **C. Common `basitra` namespace (`basitra.basis.core`)** | Accurate, but no problem today requires it | See below | Reject |
| **D. Dual namespace or shim** | Neutral | Long-lived complexity. See below | Reject unless a rename is ever separately adopted |

**Option C analysis.** A namespace package is a software architecture decision, and evidence does not support taking it:

- **No architectural need.** The packages are deliberately independent, and dependencies point toward the kernel and contracts ([`basis-ecosystem.md`](basis-ecosystem.md)). A shared top-level name expresses no real relationship.
- **Packaging fragility.** A PEP 420 namespace requires that no participant ship `basitra/__init__.py`. One mistaken file in any independently released package breaks the others' imports.
- **Migration.** All 494 source lines, the boundary tests, tool configuration, and documented public API paths change, in a breaking release across independent release cycles.
- **Third-party extensibility.** Conforming third parties integrate through the extension API `Protocol` classes and need no shared namespace. A shared namespace would invite third-party modules under `basitra.*`, implying first-party status that ADR-0024 Decision 7 separates from conformance.
- **Conceptual cost.** A `basitra` import root suggests a runtime component. ADR-0019 Decision 1 says Basitra names none.
- **Foreign collision.** A shorter `basis.*` root is not usable: both foreign PyPI projects use top-level `basis`.

**Option D analysis.** A re-exporting shim creates two module paths for one implementation. Depending on how the alias is built, classes can load twice, so `isinstance` checks and Pydantic model identity can fail silently. In an authorization kernel that is a correctness risk. Shims are viable only as a bounded transition within a separately justified rename, and none is justified.

**Recommended.** Keep all current `basis_*` imports indefinitely. New BASIS-scoped modules use flat `basis_<component>`. New Basitra-wide modules use flat `basitra_<component>`. Reject a common `basitra` namespace package. Confidence: **high**.

---

## 10. `basis-schemas` and Contract Publication

**Accepted.** Contract semantics are inside BASIS, and publication is a Basitra-level role ([ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md) Decision 4). ADR-0025 Decision 9 keeps `basis-schemas` as the publisher of BASIS contracts.

**Observed.**

| Item | State |
| - | - |
| Distribution / import | `basis-schemas` / `basis_schemas`; wheel contains metadata only (`src/basis_schemas`) and not the `schemas/` files |
| Contracts | 20 YAML files; `contract.name` values such as `decision-request` and `audit-event` carry no brand |
| Contract metadata | Every contract declares `governed_by: basis-architecture` and `published_by: basis-schemas`; tests assert both |
| Display titles | `title: BASIS Decision Request` (Class A) |
| Schema IDs and URLs | No `$id` or URL identifiers in `basis-schemas` |
| Schema bundles | None exist. `bundle_id` occurrences are policy-bundle data, unrelated to this question |
| Fixtures | `examples/operation-aware/compatibility/`; vendored into `basis-core/tests/fixtures/basis-schemas/v0.2.x/` with per-file SHA-256 manifests |
| External consumers | Not verified. No first-party distribution depends on the package; consumption is by pinned release, tag, submodule, or vendored copy |

Older `$id` forms exist elsewhere and are inconsistent:

- `basis-core` uses `https://basis-core/schemas/*.schema.json`, non-resolvable, with an open question recorded in `basis-core/docs/schema-versioning.md:124` about moving to `schemas.basis-foundation.org`.
- `basis-adapters` uses `https://basis.example/`, `https://basis-foundation.example/`, and in `rest-mapping.schema.json` `https://basis-foundation.dev/`. Unlike `.example`, `.dev` is a real public top-level domain. Ownership of that domain: **Not verified**.

**Inference.** `basis-schemas` and `basis_schemas` remain accurate: the package publishes BASIS contracts, and the publisher's own placement outside BASIS is already handled by ADR-0025's reading of the prefix. `basitra-schemas` would imply that the contracts are ecosystem-wide, which they are not.

**Recommended.** Keep `basis-schemas` and `basis_schemas`. Do not rename contract or schema identifiers for branding. Treat `governed_by` and `published_by` as contract-governance values ([Section 16](#16-relationship-to-basitra-architecture)). If a schema bundle artifact is created, name it `basis-schemas` or `basis-contracts-<version>` under contract governance. Any move to resolvable `$id` values is a contract-governance decision depending on OD-1 for a Foundation-named domain.

---

## 11. Evidence Namespaces and Durable Values

**Accepted.** Retained evidence is never rewritten (ADR-0025 Decision 14). Evidence data values and contract identifiers are under contract governance.

| Value | Location (Observed) | Kind | What it describes | Recommended treatment | Owner |
| - | - | - | - | - | - |
| `basis_gateway.*` (7 keys: `action_composed`, `resource_composed`, `original_action`, `original_resource_id`, `composed_action`, `composed_resource_id`, `resource_type`) | `RESERVED_CONTEXT_PREFIX`, `basis-gateway/src/basis_gateway/core/actions.py:29`; caller collisions rejected at `api/routes.py:575`; read by `basis-console` | Reserved namespace | The first-party enforcement implementation | Permanently stable. No brand migration | Contract governance. Inventory status "emerging" ([§3.11](ecosystem-contract-inventory.md)) |
| `adapter_source` first-party values (`basis-adapters:modbus`, `basis-adapters:rest:instance-1`) | Contract examples; supplied by producer configuration via `compose_adapter_source` | Provenance identifier; opaque | The first-party implementation and instance | Keep. A value changes only if the implementation identity actually changes, for newly emitted evidence only | Contract governance (ADR-0007) |
| `evidence_profile: basis-adapter-evidence-v1` | `basis-adapters/src/basis_adapters/evidence.py:105`; [ADR-0007](../adr/0007-adapter-evidence-construction.md); inside digested material | Contract identifier, Class G | The evidence format (BASIS contract) | Permanently stable. New semantics take a new profile | Contract governance |
| `binding_digest` inputs | `basis_producer.authorization_execution_binding`; covers the exact submission, which carries `adapter_evidence_reference` and thus `adapter_source` | Class G | The bound operation | Any value change alters the digest. Never migrate retained material | ADR-0012 |
| `X-BASIS-Producer-Client-Cert` | [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md); `basis-gateway/.../producer_mtls_trusted_proxy.py:110` | Wire contract, private | The BASIS trusted-ingress handoff | Keep. An alias is unsafe ([Section 21](#21-security-and-supply-chain-considerations)) | ADR-0009 |
| `basis.*` token claims; `basis_claim_namespace = "basis"`; token type `basis-local-identity` | `basis-identity/.../tokens/gateway_contract.py:459`, `tokens/local.py:265`; `basis-gateway/.../basis_local_token.py:128` | Contract between identity and gateway | The BASIS identity role | Keep. Accurate | Contract governance |
| `governed_by`, `published_by` | Every contract | Contract metadata | The repositories | [Section 16](#16-relationship-to-basitra-architecture) | Contract governance |
| `evaluated_by: basis-core`, `composed_by: basis-gateway` | `basis-console/src/basis_console/audit.py:166`, in a sample-event builder | Display and sample | Components | Class A; not persisted evidence on this code path | Console |
| Historical persisted evidence | Any retained store | Historical value | As written | **Immutable** | Contract governance |
| `"role": "basis_core"`, `"authority": "basitra"` | `ipotio/architecture/docs/adr/0001-evidence-schema-and-canonical-status-representation.md:1339` | External ecosystem value | First-party kernel | Outside OD-10; Ipotio's own record, relevant to OD-3 | Ipotio |

**Inference.** Names like `basis_gateway.*` and `basis-adapters:*` denote a first-party implementation, while ADR-0024 makes the role the durable unit. A conforming third-party enforcement implementation would raise a real question: does it write into `basis_gateway.*`? That is an architecture and contract question that branding should not decide. Contract inventory already records schema enforcement of this namespace as open.

**Recommended.** OD-10 should state that evidence namespaces and durable evidence values are not changed for branding, ever, and delegate any non-branding change to contract governance. No migration mechanism is needed. Confidence: **high**.

---

## 12. Environment Variables and Configuration Keys

**Observed.** `basis-gateway` and `basis-console` use `pydantic-settings` with `env_prefix=""`. Libraries (`basis-core`, `basis-identity`, `basis-adapters`, `basis-schemas`) and the `basis-producer` runtime read no environment variables.

| Variable | Scope | Category |
| - | - | - |
| `BASIS_LOCAL_TOKEN_ISSUER`, `_AUDIENCE`, `_PUBLIC_KEYS_JSON`, `_ALLOWED_ALGORITHMS`, `_LEEWAY_SECONDS` | `basis-gateway`; documented in `docs/configuration.md` | Package-local public configuration. Names the BASIS-local token type |
| `BASIS_CONSOLE_MODE` | `basis-console` | Package-local public configuration |
| `BASIS_GATEWAY_REPO_ROOT` | `basis-producer` CI, tests, demos | Test harness |
| `BASIS_TEST_REQUEST_LOG_PATH` | `basis-gateway/tests/integration` | Undocumented test internal |
| `SERVICE_NAME`, `HOST`, `PORT`, `LOG_LEVEL`, `ENVIRONMENT`, `AUTH_MODE`, `OIDC_*`, `JWKS_CACHE_TTL_SECONDS`, `POLICY_*`, `AUDIT_*`, `OPERATION_AWARE_*`, `OPERATION_PRODUCER_*`, `GATEWAY_*` | Gateway and console | Unprefixed package-local configuration |

Other `BASIS_*`-shaped names found (`BASIS_LOCAL_IDENTITY`, `KNOWN_BASIS_REPOS`, `BASIS_KEYS`, and similar) are Python constants or enum members, not environment variables. `AUTH_MODE=basis_local_token` is a Class E value. `SERVICE_NAME` defaults to the component name and appears in `/health`. `OIDC_AUDIENCE=basis-gateway` in `.env.example` is deployment-owned.

**Inference.** No ecosystem-wide `BASIS_*` namespace exists to migrate. The prefixed variables are accurate: they name a BASIS-local token and a BASIS-scoped console. Renaming yields no accuracy gain. Production usage is **Not verified**.

| Pattern | Assessment |
| - | - |
| Retain indefinitely | **Recommended** |
| Add aliases | Technically simple, but no value to justify it, and a security-relevant trust setting would gain two spellings |
| Gradual deprecation or future major | Available if a real reason ever arises |
| Leave outside OD-10 | Partly. ADR-0025 assigns these to a compatibility plan. OD-10 can state the convention and leave any individual change there |

**Recommended.** Retain existing names. Unprefixed package-local settings need no prefix added. A new setting that needs a prefix follows scope of concern: `BASIS_` for BASIS-scoped components, `BASITRA_` for Basitra-wide tools. Confidence: **high**.

---

## 13. CLI and Executable Names

**Observed.** No repository declares `[project.scripts]`, and none has a `__main__.py` or CLI module. Services start through ASGI paths, `uvicorn basis_gateway.main:app` and `uvicorn basis_console.main:app` (documented; the console also has `make run`). `basis-core/scripts/update_basis_schemas_snapshot.py` is an internal maintenance script.

**Inference.** The ASGI paths are the only de facto entry points. They are public, documented, and depend on the import namespace, so [Section 9](#9-python-import-namespace-naming) protects them. There is nothing to migrate.

**Recommended (forward convention).** A future CLI takes its component's name (`basis-gateway`, `basitra-deploy`). Avoid a bare `basis` command: the foreign `basis-core` package installs a `basis` console script, so the collision would be real. Do not create an umbrella `basitra` command by default: Basitra names no runtime component, and whether a Basitra-wide tool warrants that name can be decided when one exists. Check for collisions at adoption. Confidence: **medium**, because no CLI exists to test it against.

---

## 14. Container and Registry Naming

**Observed.** None of the seven component repositories tracks a `Dockerfile`, Compose file, Helm chart, or image-publishing workflow. No `ghcr.io` or `docker.io` reference appears. Mentions of Docker are prose, usually stating that deployment artifacts are out of scope. Historical `basis` and `basis-poc` contain Compose files for local use (ADR-0025 Decision 12 applies).

**Not verified.** Whether any image exists in a registry. The available GitHub token lacks `read:packages`, unauthenticated GHCR lookups returned 403 (inconclusive), and Docker Hub's `basisfoundation` organization lookup returned 404. **Requires migration-phase verification.**

**Inference.** If no images exist, OD-10 can define the forward convention at no migration cost.

| Candidate | Assessment |
| - | - |
| Leaf name equals component name, scope of concern (`basis-gateway`, `basitra-deploy`) | **Recommended.** Matches the components; the registry path supplies the owner, so no double scoping is needed |
| `basitra-basis-gateway` | Unneeded. Unlike PyPI, container registries are namespaced by owner |
| Unprefixed (`gateway`) | Readable under an owner, but loses context when images are mirrored into OT-site registries |
| Everything `basitra-*` | Same objection as Policy B |

An all-in-one image of the Basitra distribution, if ever created, contains non-BASIS components and would therefore take a `basitra-` name.

**Recommended.** Leaf name equals component name. Registry and organization path is decided with OD-7. Use a current organization path only as an interim, with OCI source labels recording provenance. Confidence: **medium to high**.

---

## 15. Distribution Terminology Versus Python Distributions

These are different concepts.

| Term | Meaning | Source |
| - | - | - |
| **The Basitra distribution** | The ecosystem's first-party software set, a human-facing stewardship term. Creates no technical artifact | ADR-0025 Decision 13 |
| **Python distribution package** | A packaged, installable unit with a `Name`, in Python packaging terminology | PyPA terminology |

**Accepted.** ADR-0025 Decision 13 creates no package or meta-package. Calling the software set "the Basitra distribution" does not imply that any Python distribution package is named `basitra-*`. A `basis-core` wheel can belong to the Basitra distribution.

**Recommended.** The ADR should use "distribution package" for the packaging sense. If a meta-package is ever wanted, it is a separate decision.

---

## 16. Relationship to `basitra-architecture`

**Observed.** `basis-architecture` has no `pyproject.toml`, requirements, Dockerfile, workflow, import namespace, or published artifact. Its compatibility-bearing surface is limited to:

- **URLs** cited by implementation repositories (for example `basis-gateway/README.md:599`), addressable by GitHub redirect after a rename. Redirect details are migration and OD-7 concerns.
- **`governed_by: basis-architecture`** in all 20 contracts, in `contract-metadata.yaml` examples, and asserted in `basis-schemas` tests (for example `tests/test_action_string_contract.py:58`). `basis-core` vendors per-file SHA-256 manifests of these contract files by version.
- **Repository lists in tests**, such as `KNOWN_BASIS_REPOS` at `basis-gateway/tests/doc_link_helpers.py:56`, which would need updating.

**Inference.** ADR-0025 Decision 4 correctly states that no package, import namespace, or registry artifact depends on this name. Contract metadata is outside that list, and this assessment does not reopen the decision. The interaction is one the migration plan must handle: rename the repository, and existing published contract versions still correctly say `basis-architecture`.

**Recommended.** Published contract versions are not edited. The rename migration plan, under contract governance, decides whether newer contract versions carry `basitra-architecture`, and bumps versions accordingly. The repository's low rename cost should not influence the released-package analysis.

---

## 17. Relationship to Planned `basitra-deploy`

**Accepted.** `basitra-deploy` is the target repository name (ADR-0025 Decision 11). The repository does not exist. No technical name is decided. These are candidates only.

| Surface | Candidate | Note |
| - | - | - |
| Repository | `basitra-deploy` | Accepted target |
| Distribution package, if Python | `basitra-deploy` | Not double-scoped: scope of concern is Basitra. Whether it is a Python package at all is undecided. Availability must be verified |
| Python module | `basitra_deploy` | Flat, not a namespace |
| CLI | `basitra-deploy` | Component name |
| Container image | `basitra-deploy` leaf | Registry path per OD-7 |
| Deployment labels | `app.kubernetes.io/name: basis-gateway`, `app.kubernetes.io/part-of: basitra` | Component keeps its name; the ecosystem label is accurate |

---

## 18. Relationship to OD-1 and OD-7

| Classification | Item | Reason |
| - | - | - |
| **Independent** | Import namespaces | No organization or Foundation in the name |
| **Independent** | Distribution names on current channels | Same |
| **Independent** | Future `basis-*` / `basitra-*` naming rule | Rests on ADR-0019, ADR-0024, ADR-0025 |
| **Independent** | Evidence, contract, and digest-bound identifier stance | No organization component |
| **Independent** | Environment variables and CLI conventions | Same |
| **Independent** | Container leaf name | Registry path is separate |
| **Requires coordination** | Public-index account or organization ownership | The registry publisher is a stewardship question (OD-1) |
| **Requires coordination** | Package `authors` metadata (`Basis Foundation`) | Reflects OD-1 outcome. Not a name convention |
| **Requires coordination** | `[project.urls]` | Follows OD-7; GitHub redirects are expected to cover the interim (Requires migration-phase verification) |
| **Actually blocked** | Final container registry organization path | Needs OD-7 |
| **Actually blocked** | Any Foundation-named or organization-qualified identifier (`schemas.basis-foundation.org`, a `basis-foundation`-scoped registry namespace) | Needs OD-1 or OD-7 |

**Inference.** The strategy document's table notes that organization-dependent choices may require coordination. The dependency is narrow, and OD-10's main decisions can proceed without waiting for OD-1 or OD-7.

---

## 19. External Registry Availability

**Availability must be verified before normative adoption.** The PyPI responses in this document are point-in-time observations. A 404 does not show a name is claimable: PyPI normalizes names and rejects some names for other reasons. This applies to `basitra-basis-*`, `basitra-deploy`, and every other candidate. No name has been reserved.

---

## 20. Third-Party Ecosystem Implications

**Accepted.** ADR-0024 Decision 7 distinguishes architectural conformance, ecosystem participation, and first-party implementation. It defines no third-party naming rules, and leaves them to future work.

| Question | Assessment |
| - | - |
| Does `basis-*` imply first-party? | In practice many readers will assume so. Technically it is only descriptive, and foreign `basis`-named packages already exist |
| Is it merely descriptive? | Yes, today. "basis" is a common word and cannot be exclusive |
| Should third parties be discouraged from reserved prefixes? | **Unresolved.** Governance question. This assessment invents no policy |
| Trademark or community governance instead of OD-10? | Yes. OD-10 should record the boundary. Strategy document §12.7 and OD-1 stewardship are the natural homes |
| Should conformance identifiers be separate from package naming? | Yes, per ADR-0024: conformance confers no name rights and a package name proves no conformance. Any future conformance identifier should be a distinct mechanism |

**Inference.** Under public-index option I4, the `basitra-basis-` prefix would function as a first-party signal only if the project governs its use, which is unaddressed here. A descriptive convention such as `<vendor>-basis-adapter-<protocol>` could be suggested later, under governance.

---

## 21. Security and Supply-Chain Considerations

| Risk | Status | Consideration for OD-10 |
| - | - | - |
| **Dependency confusion on `basis-core`** | **Present (Observed).** `basis-gateway` and `basis-producer` require `basis-core>=0.2.1,<0.3.0`. Default `pip` resolves against PyPI, where a different party controls that name. Today only 0.1.0 exists there, so the range fails. A future upload in the range would resolve, unless a sibling is already installed | Do not depend on the absence of a satisfying foreign release |
| **Unheld first-party names** | **Present (Observed).** The six other names returned 404. `basis-producer` pins `basis-adapters==0.2.0` | If those names can be registered, a third party could publish a matching version. Whether to register defensively is a decision for the ADR. Nothing has been reserved |
| **Wrong documentation** | **Present (Observed).** `pip install basis-core` installs foreign code | Correcting first-party install guidance need not wait for OD-10. It is outside this PR |
| **Package-name abandonment** | Future | Applies only once a name is actually on an index. Retain ownership after any later rename |
| **Dual-published drift** | Avoidable | The one-name-per-component rule prevents old and new packages drifting |
| **Import alias ambiguity** | Avoidable | Duplicate module objects weaken type-identity checks ([Section 9](#9-python-import-namespace-naming)) |
| **Header alias** | Avoidable | The ingress must overwrite `X-BASIS-Producer-Client-Cert` unconditionally (ADR-0009). A second accepted name would need the same treatment, and a missed overwrite creates a forged-identity path. Aliasing here is not viable |
| **Image registry confusion** | Future | OCI source labels and a single owner path. Signing is outside this assessment |
| **Unofficial packages claiming association** | Future | Governance boundary ([Section 20](#20-third-party-ecosystem-implications)). Package names do not establish trust (ADR-0023 Decision 7) |
| **Real domain in `$id`** | **Observed.** `basis-foundation.dev` in one `basis-adapters` schema | Ownership Not verified. Contract governance should confirm that this was intended |

---

## 22. Candidate OD-10 Policies

- **Candidate 1, Preserve published names:** keep `basis-*` / `basis_*`; Basitra naming only for future Basitra-wide artifacts.
- **Candidate 2, Basitra-prefix migration:** `basitra-core`, `basitra_core`, and so on.
- **Candidate 3, Hierarchical:** `basitra-basis-core` / `basitra.basis.core`.
- **Candidate 4, Surface-specific hybrid:** each surface treated on its evidence, with a defined public-index boundary.

| Criterion | 1 Preserve | 2 Basitra-prefix | 3 Hierarchical | 4 Hybrid |
| - | - | - | - | - |
| Architectural accuracy | Good | Poor: erases Basitra/BASIS distinction | Good, but encodes it everywhere | Good |
| Compatibility | Best | Worst | Very poor | Near best: imports, evidence, config unchanged |
| Discoverability | Good on GitHub; **fails on public index** | Good | Good on index | Good, with index case handled |
| Comprehension | Good | Misleading | Verbose | Good; one extra rule |
| Ecosystem coherence | Good | Superficially uniform | Rigid | Good, rule-based |
| Third-party extensibility | Neutral | Blurs first-party boundary | Invites `basitra.*` modules | Neutral; boundary recorded |
| Implementation complexity | None | High | Highest | Low: conventions now, work only if an index is chosen |
| Migration risk | None, until an index is attempted | High | High | Low |
| Future durability | Blocked if publication is desired | Fragile to the same OD-8 distinction | Heavy | Good |
| Consistency with ADR-0025 | Consistent | Conflicts in spirit | Not required | Consistent, without copying repository names mechanically |

---

## 23. Recommendation (Non-Normative)

**Recommended: Candidate 4, the surface-specific hybrid.** Candidate 1 is correct on nearly every surface. It fails on one evidence-based case, the public index, which Candidate 4 handles.

**The target question.** If Basitra had been the identity from the start and BASIS still existed, the kernel import would still be `basis_core`, because the code implements BASIS roles. Evidence keys would still name the BASIS component. The one thing likely to differ is the public-index distribution name, since a flat index would have forced a more distinctive name from the outset.

| Surface | Recommended OD-10 position | Confidence |
| - | - | - |
| Existing BASIS distribution packages | **KEEP** `basis-*` on channels the project controls | High |
| Public-index publication | **GATED.** Not under bare `basis-*`. If chosen, prefer family-wide `basitra-basis-<component>`. Availability must be verified | High on the gate; medium on the form |
| Existing `basis_*` imports | **KEEP** indefinitely | High |
| Future BASIS-scoped packages | `basis-<component>` / `basis_<component>` | High |
| Future Basitra-wide packages | `basitra-<component>` / `basitra_<component>` | High |
| Common `basitra` namespace package | **REJECT** | High |
| Evidence namespaces | **KEEP**; contract-governed | High |
| Historical evidence values | **IMMUTABLE** | High (accepted) |
| Digest-bound profiles | **PERMANENT**; new semantics, new profile | High |
| `adapter_source` values | **KEEP**; change only with actual implementation identity, for new evidence only | High |
| Contract and schema identifiers | **KEEP**; contract-governed | High |
| Environment and config names | **KEEP**; scope-of-concern prefix for new names where a prefix is needed | High |
| Wire headers and token claims | **KEEP** | High |
| Container images | Leaf equals component name; organization path with OD-7 | Medium to high |
| CLI | Component name; no bare `basis`; no default umbrella `basitra` | Medium |
| Low-compatibility display text | May change through ordinary documentation work | High |

**This table is a recommendation. OD-10 remains open.**

---

## 24. Answers to the Required Questions

Recommended answers. None is a decision.

| # | Question | Recommended answer |
| - | - | - |
| 1 | Existing released distributions keep `basis-*`? | Yes on controlled channels. They are not on any public index, and bare `basis-*` should not be extended to one |
| 2 | Current imports keep `basis_*`? | Yes |
| 3 | Future BASIS-scoped packages use `basis-*`? | Yes, subject to the public-index gate |
| 4 | Future Basitra-wide packages use `basitra-*`? | Yes, availability to be verified |
| 5 | Common `basitra` Python namespace? | No |
| 6 | Package names mirror repository names? | Not as a rule. They coincide today and diverge freely (ADR-0025 Decision 14) |
| 7 | Evidence namespaces migrate for branding? | Never |
| 8 | `adapter_source` values change? | No. Opaque, contract-governed |
| 9 | `BASIS_*` environment variables change? | No |
| 10 | Future container images? | Leaf equals component name, by scope of concern; organization path with OD-7 |
| 11 | Future CLIs? | Component name; no bare `basis` |
| 12 | Outside OD-10, contract governance? | Evidence value semantics, schema and contract IDs, `$id` policy, `governed_by` / `published_by` evolution, `basis_gateway.*` schema enforcement and role-neutrality |
| 13 | Depend on OD-1? | Publisher account ownership, Foundation-named identifiers, `authors` metadata |
| 14 | Depend on OD-7? | Registry organization path, organization-qualified identifiers, `[project.urls]` |
| 15 | Decidable now? | Everything in the "Independent" rows of [Section 18](#18-relationship-to-od-1-and-od-7) |
| 16 | Requires a major version? | Import renames, contract or evidence field and value changes, wire changes, and environment variable removal are breaking. The owning governance of each surface sets what it requires, and the containing package's version does not. A stable contract needs a contract major increment under `basis-schemas/docs/contract-governance.md`. [Compatibility philosophy](compatibility-philosophy.md#breaking-change-expectations) requires a major increment, an ADR, and a deprecation period for the surface being changed. See the note below |
| 17 | Aliases or deprecation viable? | Environment variables and image tags (viable, unnecessary); distribution packages (transition only); imports (risky). Not evidence, digest-bound values, or the ingress header |
| 18 | Permanently stable? | Retained evidence, digest-bound profiles, `basis_gateway.*` in retained records, ADR-accepted wire identifiers, published contract versions |

**Note on question 16: package versions and surface versions are separate lifecycles.**

**Accepted.** Compatibility philosophy states that [compatibility surfaces version independently](compatibility-philosophy.md#semantic-versioning-philosophy). `basis-schemas/docs/contract-governance.md` sets the rules for contracts by lifecycle state, and requires a contract major increment for a breaking change to a stable contract.

**Recommended.** Each compatibility-bearing surface follows the versioning and breaking-change rules of its owning contract or governance mechanism. Package-version changes do not substitute for contract, schema, evidence, API, or other surface-specific versioning obligations.

```text
Python distribution version
        ≠
contract version
        ≠
schema version
        ≠
evidence-profile version
        ≠
wire/API compatibility version
```

These may move together in a particular release, but one does not define another:

- **Package increments.** A Python package may move from `0.2.x` to `0.3.x`. That fact alone does not authorize a breaking contract change.
- **Contracts.** A stable contract that requires a contract major increment receives it, regardless of the containing package's pre-1.0 version.
- **Evidence and wire identifiers.** Evidence namespaces, evidence profiles, durable evidence vocabulary, digest-bound structures, and wire contracts follow their own owners and compatibility rules. No package increment authorizes a break to them. Retained evidence remains immutable.
- **Imports and APIs.** Import and API breakage follows the applicable package or API compatibility discipline. For `basis-core`, `docs/public-api.md` and `docs/breaking-change-discipline.md` apply: a stable-tier breaking change requires an ADR, and changing a public import path is breaking.
- **Dependency bounds.** A bound such as `<0.3.0` describes what a consumer accepts. It does not define the versioning policy for any surface inside the dependency.

**Unresolved.** This assessment does not settle how pre-1.0 package versions signal breaking changes. Compatibility philosophy says the ecosystem "has not yet operationalized a specific versioning scheme", and `basis-core/docs/schema-versioning.md` tracks this as `OPEN: schema-semver-operationalization`. Any pre-1.0 package-version exception must be explicitly established by the owner or governance of the affected surface, and this assessment assumes none.

**Observed.** Owner statements found so far are limited to their own surfaces. `basis-adapters/docs/public-api.md` calls that library's API pre-v1 with no semantic-versioning guarantees yet, while requiring breaking changes to be deliberate, documented, and reflected in schemas, examples, and contract tests. `basis-core/docs/schema-contracts.md:149` remarks in an open question that its pre-1.0 schemas carry no compatibility obligation. Neither states a package-version rule, and neither extends to contracts, evidence, or wire identifiers owned elsewhere.

Versioning does not bear on the public-index collision in [Section 8](#8-python-distribution-naming), which needs its own naming or ownership solution.

---

## 25. Open Questions and ADR Boundary

### Questions OD-10 should decide normatively

1. Adopt the surface-specific hybrid and principles of [Section 4](#4-governing-principles)?
2. Keep all existing distribution names and imports?
3. Adopt the public-index gate, and choose among I1 through I4 or defer the form until publication is proposed?
4. Reject a common `basitra` namespace package?
5. Adopt `basis-*` / `basitra-*` for future packages, images, and CLIs?
6. Adopt the one-name-per-component rule?
7. State the "no branding migration" position on evidence, contract, and digest-bound identifiers, and delegate the rest to contract governance?
8. Should first-party names be defensively registered on public indexes?

### Questions that belong to migration planning

- Repository-rename handling of `governed_by`, URLs, and test repository lists.
- Execution of any public-index adoption, including verification, account setup, and release sequencing.
- Container registry setup once images exist.
- Reverification of every observation in this document.

### Questions outside OD-10

- Repository rename execution and GitHub organization migration (OD-7).
- Foundation naming and succession (OD-1).
- Contract semantic changes and schema evolution (contract governance).
- Third-party naming rules, trademarks, and conformance marks (governance).
- Ipotio naming and its use of `basis_core` as a role label (OD-3, in Ipotio's repository).
- Website and domain migration.
- Correction of existing install documentation, a bounded fix in `basis-core` and `basis-schemas`.

---

## 26. Proposed Follow-On Sequence

1. Merge this non-normative assessment.
2. Draft a Proposed ADR for OD-10. ADR-0026 appears to be the next number, since none exists in `docs/adr/` and no open pull request claims it as of the inspection date. **Confirm before the ADR PR.**
3. Lead Architect review.
4. Codex cold review.
5. Merge the proposal.
6. Formal acceptance PR.
7. Technical migration plans only where names actually change. Under the recommendation, that is little or nothing until public-index publication is proposed.

Separately, and not gated by OD-10: correct `pip install basis-core` in `basis-core/README.md`.

---

## 27. What This Assessment Does Not Do

It does not create an ADR or resolve OD-10, and it does not:

- rename or change a Python package, import namespace, dependency specifier, repository, executable, environment variable, config key, schema or contract identifier, evidence namespace, or `adapter_source` value;
- publish, reserve, or alias a package, shim, or container image;
- rewrite retained evidence;
- migrate `basis-architecture`, create `basitra-deploy`, migrate the GitHub organization, rename the Basis Foundation, change Ipotio, or update the website or domain;
- modify implementation code, ADR-0010, ADR-0024, or ADR-0025.

---

## 28. References

- [ADR-0007](../adr/0007-adapter-evidence-construction.md), [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md), [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md), [ADR-0019](../adr/0019-basitra-ecosystem-identity-and-terminology-hierarchy.md), [ADR-0024](../adr/0024-long-term-scope-of-basis-within-basitra.md), [ADR-0025](../adr/0025-basitra-repository-and-component-naming.md)
- [`basitra-ecosystem-and-boundary-aware-security.md`](basitra-ecosystem-and-boundary-aware-security.md), including [Open Decisions](basitra-ecosystem-and-boundary-aware-security.md#13-open-decisions)
- [`basitra-repository-and-component-naming-assessment.md`](basitra-repository-and-component-naming-assessment.md)
- [`compatibility-philosophy.md`](compatibility-philosophy.md), [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`basis-schemas.md`](basis-schemas.md), [`basis-ecosystem.md`](basis-ecosystem.md), [`kernel-boundary-rules.md`](../kernel-boundary-rules.md)
- PEP 503 (name normalization), PEP 420 (namespace packages), PEP 541 (package index name retention). Packaging-ecosystem behavior cited here requires migration-phase verification.
