# Terminology Rules

This document defines the rules governing terminology use across the BASIS architecture repository and, by extension, across all repositories in the BASIS ecosystem. Its purpose is to prevent vocabulary drift that can cause confusion, miscommunication between contributors, or inconsistency between architecture documents and implementation repositories.

This document extends — and does not replace — the terminology guidance in [`docs/standards/writing-guidelines.md`](writing-guidelines.md) (Section 3) and the definitions in [`docs/glossary.md`](../glossary.md). The writing guidelines establish the general approach to terminology consistency. The glossary defines individual terms. This document defines the rules for maintaining that consistency over time, especially as the ecosystem grows and new contributors encounter established vocabulary.

---

## The Core Principle

Terminology in the BASIS ecosystem is not decorative. Action names are policy contracts. Component names are architectural claims. The distinctions between related terms reflect real structural differences that affect how components are built, how they interoperate, and how deployments are operated.

When terminology drifts — when contributors invent synonyms for established terms, when related concepts are conflated, when capitalization conventions are ignored — the cost is not aesthetic. It is operational: engineers who read one document and then another cannot be certain whether the same word refers to the same concept.

The core rule is: use the established term. If the established term is inadequate, update it through the process described in the "Introducing New Terms" section — do not invent a local synonym.

---

## The Five Ecosystem Entities

The most consequential terminology requirement in this repository is the correct use of the five ecosystem entities. These are not interchangeable names for related things. They are distinct concepts with distinct roles, scopes, and governing authorities.

**Basis Foundation** — the nonprofit open-source governance body. Governs architectural standards, the schema and compatibility definitions that allow components to interoperate, and the open-source repositories under the BASIS namespace. Does not build commercial products, does not operate managed services.

- Correct: "The Basis Foundation governs the BASIS Core Services Distribution."
- Incorrect: "BASIS governs the open-source project."
- Incorrect: "The Foundation operates the authorization service."

**BASIS Core Services Distribution** — the set of open-source, deployable software components maintained under Foundation governance: basis-core, basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, and basis-schemas. Together these components constitute a functional, deployable identity-aware authorization system.

- Correct: "The BASIS Core Services Distribution provides a complete authorization system for self-hosted deployments."
- Incorrect: "BASIS provides managed hosting." (That is BASAuth.)
- Incorrect: "basis-core is the BASIS Core Services Distribution." (basis-core is one component of the distribution.)

**BASAuth** — the future for-profit commercial company that builds enterprise products and managed services on top of the open-source distribution. Does not govern the open-source components. Is not a prerequisite for the open-source distribution to function.

- Correct: "BASAuth offers managed hosting and fleet management services."
- Incorrect: "basis-core is maintained by BASAuth." (basis-core is maintained under Foundation governance.)
- Incorrect: "BASAuth controls the open-source project." (The Foundation controls the open-source project.)

**basis-core** — the isolated authorization kernel in the BASIS Core Services Distribution. Implements policy evaluation semantics, enforcement contracts, failure mode contracts, and audit event schema. Distinct from the PoC. Does not depend on higher-level services.

- Correct: "basis-core implements the policy evaluation logic that all other components depend on."
- Incorrect: "basis-core hosts the API." (basis-gateway hosts the API.)
- Incorrect: "basis-core includes the BACnet adapter." (basis-adapters includes protocol adapters.)

**basis-poc** — the research proof-of-concept, a monolithic, containerized research implementation used to validate the core architectural mechanisms. A research artifact, not a production system. Distinct from basis-core.

- Correct: "The basis-poc validated that identity propagation and protocol-agnostic enforcement are implementable."
- Incorrect: "basis-poc is the same as basis-core." (The PoC is a monolithic research artifact; basis-core is the isolated kernel direction that emerged from it.)
- Incorrect: "The PoC is production-ready." (It is explicitly a research artifact.)

When all five entities are referenced in a single document, the hierarchy is: Basis Foundation governs the BASIS Core Services Distribution (which includes basis-core among other components), and BASAuth builds commercial services on top. basis-poc is a research artifact separate from the distribution.

---

## Component Naming Conventions

Repository and component names in the BASIS ecosystem follow consistent conventions that must be preserved across all documentation:

- Component names are always lowercase and hyphenated: `basis-core`, `basis-gateway`, `basis-console`, `basis-adapters`, `basis-identity`, `basis-deploy`, `basis-schemas`
- Component names are never capitalized or spaced: not `Basis Core`, not `Basis-Core`, not `BasisCore`
- `basis-architecture` refers to this documentation repository, not to the architecture itself
- "the kernel" is an acceptable shorthand for basis-core in contexts where the referent is unambiguous; "the authorization kernel" is the full form
- "the distribution" is an acceptable shorthand for the BASIS Core Services Distribution in contexts where the referent is unambiguous

When referring to a component in prose, use its full name on first reference in any document section, then the shorthand thereafter: "basis-core (the authorization kernel)" on first use, then "basis-core" or "the kernel" subsequently.

---

## Capitalization Rules

The capitalization rules defined in writing-guidelines.md Section 3.3 apply to all content in this repository and in all BASIS ecosystem documentation. Do not reproduce the full capitalization table here; consult writing-guidelines.md directly.

The most commonly violated capitalization rules are:

- **BASIS** is always all caps (acronym)
- **BASAuth** is mixed case (proper name of the commercial entity) — not `BASAUTH`, not `Basauth`, not `basauth`
- **Basis Foundation** uses initial caps (proper name of the organization) — not `BASIS Foundation`, not `basis foundation`
- **basis-core** is always lowercase (component name, treated as a code identifier) — not `Basis-Core`, not `BasisCore`
- **BACnet** is mixed case (protocol proper name) — not `BACNET`, not `Bacnet`
- **Modbus** is initial cap (protocol proper name) — not `MODBUS`, not `modbus`

When in doubt, check writing-guidelines.md Section 3.3 before choosing a capitalization form.

---

## Authorization vs. Authentication

These terms are frequently conflated in general security writing. In this repository they have distinct, precise meanings that must be maintained.

**Authentication** is the process of verifying that a subject is who or what it claims to be. Authentication produces a verified identity: a credential or assertion that confirms the subject's identity attributes. Authentication is the responsibility of the identity provider. basis-core does not authenticate subjects.

**Authorization** is the process of determining whether a verified subject is permitted to perform a requested action on a specific resource, according to applicable policy. Authorization assumes authentication has already occurred; it takes verified identity as an input, not as a responsibility. Authorization is the responsibility of the policy engine and the enforcement infrastructure.

Do not use "authentication" when the precise meaning is "authorization." Do not describe authorization as "access authentication." Do not describe authentication as "identity authorization."

---

## Policy Engine vs. Enforcement Point

**Policy engine** — the component responsible for evaluating authorization requests against defined policy and producing authorization decisions. Evaluates; does not enforce directly. In the BASIS ecosystem, the policy evaluation logic belongs to basis-core; the runtime that hosts policy evaluation is basis-gateway.

**Enforcement point** — a component positioned at a trust boundary that applies authorization decisions to operational traffic. Receives a decision from the policy engine and acts on it: permitting or blocking operational traffic accordingly. In the BASIS ecosystem, enforcement points are implemented by basis-gateway instances (at the operations and supervisory zone boundaries) and by basis-adapters components (at the edge zone boundary, mediating protocol-layer command delivery).

Do not describe an enforcement point as a policy engine. They have distinct responsibilities: the policy engine evaluates; the enforcement point applies. The kernel (basis-core) is neither — it is the evaluation logic that policy engines call into.

---

## Audit vs. Telemetry

These two concepts are distinct in the authorization model and must not be conflated.

**Audit** refers to the structured record of authorization decisions: who requested what, on what resource, under what policy, with what outcome. Audit records are security-relevant and must be written with immutability guarantees. They are generated by enforcement points and the policy engine. They carry verified subject identity.

**Telemetry** refers to operational data collected from devices and system components: sensor readings, device health indicators, communication statistics, control state. Telemetry describes the operational behavior of the field infrastructure. Telemetry records carry path metadata but not verifiable device identity.

Do not use "telemetry" to refer to audit records, and do not use "audit" to refer to device telemetry. They travel through different pipelines, have different trust properties, and serve different operational purposes. Section 05 of the white paper (the discussion of asymmetric trust: commands inward, telemetry outward) provides detailed analysis of why this distinction matters.

---

## Adapter vs. Gateway

**Gateway** (basis-gateway) — the API and runtime wrapper around the authorization kernel. Hosts the network-facing API surface, manages the request lifecycle, and acts as the enforcement point for operations and supervisory zone boundary crossings. Handles session-level and command-level authorization for human and system principals.

**Adapter** (component of basis-adapters) — a protocol normalization component that translates field-protocol messages into the subject-resource-action representation the kernel evaluates, and delivers authorized commands back to the field protocol layer. Adapters operate at the edge zone boundary, mediating between OT field protocols and the shared authorization vocabulary.

Do not use "gateway" and "adapter" interchangeably. They occupy different positions in the architecture, perform different functions, and are maintained as separate components for reasons described in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md).

---

## Synonyms to Avoid

The following synonyms for established terms are prohibited. When these alternatives appear in contributions, they should be replaced with the canonical term.

| Avoid | Use instead | Reason |
| - | - | - |
| "the BASIS platform" | "the BASIS Core Services Distribution" | "Platform" implies a commercial product or managed service |
| "the BASIS system" | "the BASIS Core Services Distribution" or the specific component | Too vague; ambiguous about which entity is meant |
| "the kernel service" | "basis-core" | Adds "service" which implies a runtime API; basis-core is a library/kernel, not a service |
| "the auth engine" | "the policy engine" or "basis-core" | Informal shorthand that conflates authentication and authorization |
| "the access control system" | "the authorization system" | "Access control" is less precise than "authorization" in this context |
| "the BASIS server" | "basis-gateway" | Ambiguous; implies a monolithic deployment model |
| "BASIS components" without qualification | Specify which components or "the distribution components" | Ambiguous about which ecosystem layer is meant |
| "the PoC system" | "basis-poc" or "the BASIS proof-of-concept" | Maintain the full name or the code identifier |

---

## Introducing New Terms

When a new technical concept must be introduced in documentation:

1. **Check the glossary first.** The concept may already be defined under a different name. Use the existing term.
2. **Check the white paper sections.** Concepts are often introduced and defined in the white paper before appearing in standalone documents. Use the white paper's terminology.
3. **If the concept is genuinely new and not present in any existing document**, add a definition to [`docs/glossary.md`](../glossary.md) before using the term in any other document. Do not define terms inline in individual documents — the glossary is the single definition point.
4. **Do not invent synonyms.** If the existing term is awkward in a specific context, the appropriate response is either to rephrase the sentence or to propose updating the glossary definition. Inventing a synonym creates two terms for one concept.
5. **Consult the writing guidelines** before adding new ecosystem-level terms. Writing-guidelines.md Section 3 establishes the general approach; this document establishes the specific rules for ecosystem entities and components.

Additions to the glossary that introduce new canonical terms or change existing definitions should be reviewed and accepted through the contribution process defined in [`CONTRIBUTING.md`](../../CONTRIBUTING.md), and may require an ADR if the new term reflects a significant architectural decision. See [`docs/adr/README.md`](../adr/README.md) for when an ADR is appropriate.

---

## Cross-Repository Consistency

The terminology rules in this document apply to all BASIS ecosystem repositories, not only to basis-architecture. Implementation repositories (basis-core, basis-gateway, basis-adapters, and others) should use the same terminology in their documentation, API surface naming, and code comments.

When an implementation repository introduces a term that diverges from the canonical vocabulary, the correct response is to reconcile the divergence — either by updating the implementation repository's terminology or by proposing an update to the canonical vocabulary through the contribution process. Implementation repositories should not silently establish alternative terminology for canonical concepts.

The architecture documentation in basis-architecture is the authoritative source for ecosystem-level terminology. If an implementation repository's documentation and basis-architecture documentation use different terms for the same concept, the basis-architecture definition takes precedence for architectural communication. The implementation repository should be updated to align.
