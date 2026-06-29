# Writing Guidelines

This document establishes writing standards for all content in the BASIS architecture repository — including white papers, architecture decision records (ADRs), architecture principles, threat models, and supporting documentation. It is intended as an internal reference for anyone contributing written content to this repository.

The goal is consistency in tone, terminology, and structure across a body of documentation that will be read by engineers, security practitioners, and researchers with varying familiarity with OT environments and IAM concepts.

---

## 1. Tone and Voice

### 1.1 Target register

Write for a technically literate reader — someone with engineering or systems security background who does not necessarily have deep OT expertise. The appropriate register is analytical and precise, not conversational. Avoid both academic formality and informal colloquialism.

Good reference points: IETF RFCs, NIST special publications, cloud architecture framework guidance, industrial security guidance documents from organizations like ICS-CERT or ENISA. The tone of this repository should feel like it belongs in that category.

### 1.2 Analytical rather than persuasive

Documentation in this repository should inform rather than persuade. Describe what systems do, what tradeoffs they involve, and what risks they carry — not why the reader should adopt or endorse any particular approach. Arguments for design decisions belong in ADRs, not in architectural descriptions.

Where the paper argues a position, state it plainly and provide supporting reasoning. Avoid rhetorical structures that substitute urgency or alarm for evidence.

### 1.3 Restrained confidence

Be precise about what is known, what is inferred, and what is uncertain. Use language that accurately reflects the epistemic status of claims:

- Known: "BACnet/IP does not include built-in authentication mechanisms."
- Inferred: "Organizations using shared credentials are likely to face audit attribution challenges."
- Uncertain: "The operational impact of policy evaluation latency in real BAS deployments has not been systematically measured."

Avoid stating inferences as facts. Avoid stating uncertainties as certainties. Avoid stating facts in ways that imply more certainty than the evidence supports.

### 1.4 Operational respect

OT environments have accumulated their current architecture through decades of engineering decisions made under real constraints. Documentation in this repository should reflect awareness of those constraints. Avoid framing legacy OT architecture as the product of negligence, ignorance, or poor judgment — frame it as the product of context that has since changed.

This is not a requirement to avoid criticism of specific design patterns. It is a requirement that criticism be grounded in the specific limitations of a pattern in a specific context, not in a general judgment about the quality of OT engineering.

---

## 2. Prohibited Language

### 2.1 Marketing and promotional language

Do not use the following categories of language anywhere in this repository:

**Superlatives and exaggerated claims**
- "the only solution," "uniquely positioned," "industry-leading," "best-in-class"
- "solves the problem of," "eliminates the risk of," "ensures security"
- "seamlessly," "effortlessly," "out of the box"

**Urgency and fear-based framing**
- "threat actors are increasingly targeting," "it's only a matter of time," "organizations cannot afford to ignore"
- Statistics presented without source citations as evidence of threat severity
- Language that characterizes OT environments as inherently broken or critically vulnerable

**Roadmap and forward-looking product language**
- "coming soon," "in a future release," "on our roadmap"
- "we are building," "our platform will," "stay tuned"

**Startup and pitch language**
- "reimagining," "reinventing," "disrupting," "transforming"
- "zero to one," "game-changing," "paradigm shift"
- Mission-statement phrasing: "we believe," "our mission is," "join us in"

### 2.2 Vague technical claims

Avoid technical claims that sound specific but carry no operational content:

- "enterprise-grade security" — not a technical specification
- "real-time visibility" — without defining what real-time means in the relevant context
- "comprehensive audit trail" — without specifying what events are covered and at what granularity
- "zero trust architecture" — without specifying which zero trust principles apply and how

If a claim is worth making, it should be specific enough to be falsifiable.

### 2.3 Passive threat escalation

Do not describe threats in ways that amplify urgency without adding analytical content. Threats should be described in terms of mechanism, likelihood, and impact — not in terms of threat actor sophistication or imminent danger.

Avoid: "sophisticated adversaries are actively exploiting these weaknesses."
Prefer: "An actor with network access to the BAS VLAN can issue valid BACnet commands without authentication."

---

## 3. Terminology Consistency

### 3.1 Use the glossary as the canonical reference

All technical terms defined in [`docs/glossary.md`](../glossary.md) should be used consistently with their definitions in that document. If a term is used in a way that diverges from its glossary definition, either update the glossary or use different terminology.

When introducing a term that is not in the glossary and that is used in a non-obvious way, add it to the glossary. Do not define terms inline in multiple places — the glossary is the single definition point.

### 3.2 Prefer specificity over generality

Use the most specific term available. Prefer "BAS controller" over "OT device" when the context is specifically about building automation controllers. Prefer "authorization decision" over "access control result." Prefer "policy engine" over "authorization service" when referring to the policy evaluation component.

### 3.3 Consistent capitalization of technical terms

The following capitalization conventions apply throughout:

| Term | Convention | Notes |
|---|---|---|
| BASIS | All caps | Acronym: Building Automation Secure Identity Service |
| BAS | All caps | Acronym: Building Automation System |
| OT | All caps | Acronym: Operational Technology |
| IT | All caps | Acronym: Information Technology |
| IAM | All caps | Acronym: Identity and Access Management |
| RBAC | All caps | Acronym: Role-Based Access Control |
| ABAC | All caps | Acronym: Attribute-Based Access Control |
| MFA | All caps | Acronym: Multi-Factor Authentication |
| VPN | All caps | Acronym |
| VLAN | All caps | Acronym |
| BACnet | Mixed case | Proper name of the protocol |
| Modbus | Initial cap | Proper name of the protocol |
| LonWorks | Mixed case | Proper name of the protocol |
| MQTT | All caps | Acronym |
| policy engine | Lowercase | Common noun, not a proper name |
| identity provider | Lowercase | Common noun |
| trust boundary | Lowercase | Common noun |
| zero trust | Lowercase | Architectural concept, not a proper name |
| Control Plane | Lowercase | Common noun: "control plane" |
| Data Plane | Lowercase | Common noun: "data plane" |

For terms not listed here, follow standard technical writing conventions: capitalize proper nouns and acronyms; use lowercase for common technical terms.

### 3.4 Avoid overloading "security"

The word "security" is often used to mean different things in different contexts: confidentiality, integrity, availability, authorization, authentication, audit, and so on. Where precision matters, use the specific term rather than the umbrella term. "Access security" is usually better expressed as "authorization" or "access control." "Security event" is usually better expressed as "authorization decision event" or "authentication failure event."

---

## 4. BASIS Positioning

### 4.1 What BASIS is, in this repository

BASIS refers to the open-source core services distribution governed by the Basis Foundation. The distribution is intended to provide a complete, genuinely useful set of components for identity-aware authorization in OT environments. It is not a single product, a commercial offering, or a finished system — it is an open-source project with defined component boundaries, governance, and architectural standards.

**The three layers of the BASIS ecosystem** are described in [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md). Content in this repository should be consistent with that document.

When referring to the components:

- "the BASIS proof-of-concept" or "the BASIS PoC" or "basis-poc" refers specifically to the research implementation that validated the core architectural mechanisms. It is a research artifact, not a production system.
- "basis-core" refers to the isolated authorization kernel — the stable evaluation component that the other distribution services depend on. It is distinct from the PoC.
- "the BASIS Core Services Distribution" refers to the full set of open-source components (basis-core, basis-gateway, basis-console, basis-adapters, basis-identity, basis-deploy, basis-schemas).
- "BASAuth" refers to the future for-profit commercial entity that builds enterprise services on top of the open-source distribution. Do not describe BASAuth capabilities as part of the open-source distribution, and do not describe open-source distribution capabilities as if they require BASAuth.

Do not conflate the PoC with basis-core. The PoC demonstrated architectural feasibility in a monolithic, research-scope form. basis-core is the isolated kernel direction that emerged from the lessons the PoC surfaced. They are related but distinct things.

Do not describe the BASIS Core Services Distribution as artificially limited or as a "freemium" crippled version of a commercial product. The distribution is designed to be complete enough for real deployments. Commercial services from BASAuth address managed operations, fleet management, enterprise integrations, and similar concerns — they do not address missing authorization capability that the distribution intentionally withholds.

### 4.2 Distinguishing architecture from implementation

Maintain a clear distinction between the architectural patterns described in the white paper (which are general and technology-agnostic) and the specific implementation choices made in the BASIS PoC (which are specific and illustrative). Language like "the architecture requires Keycloak" conflates the two. Language like "the BASIS PoC uses Keycloak as its identity provider" is accurate.

Similarly, maintain a clear distinction between what basis-core owns (policy evaluation semantics, enforcement contracts, audit event schemas) and what higher-level components own (API hosting, protocol normalization, user interfaces, deployment tooling). Do not attribute to the kernel capabilities that belong in the gateway, adapter, or console layers.

### 4.3 No forward-looking claims about unbuilt components

Documentation in this repository reflects what has been designed, specified, or implemented. When describing components of the BASIS Core Services Distribution that are defined in architecture but not yet implemented, use language that clearly indicates their status — "defined in architecture," "specified but not yet implemented," or similar. Do not describe planned future components as if they are currently available.

---

## 5. Citation and Reference Guidance

### 5.1 When to cite

Cite sources when:
- Quoting or closely paraphrasing a specific document, standard, or publication
- Stating a statistical claim or empirical finding
- Referencing a specific protocol specification, standard number, or RFC
- Attributing a design pattern or methodology to a specific external source

Do not cite sources for general technical knowledge that is common background in the relevant field.

### 5.2 Citation format

Use inline citations in the format `[Author/Organization, Year]` for prose citations. Maintain a references section in each white paper section that lists full citations. For standards and specifications, include the standard number and issuing organization.

Examples:
- `[NIST SP 800-82, Rev. 3]` for NIST industrial control systems security guidance
- `[BACnet International, ANSI/ASHRAE 135]` for the BACnet protocol standard
- `[ENISA, 2023]` for a specific ENISA publication

Maintain the canonical reference list in [`whitepapers/identity-aware-authorization-for-operational-technology/references/sources.md`](../../whitepapers/identity-aware-authorization-for-operational-technology/references/sources.md).

### 5.3 Unsupported claims

Do not state empirical claims without a basis. If a claim cannot be cited or grounded in the documented PoC work, it should be framed as an inference, hypothesis, or open question — not stated as fact.

---

## 6. Diagram Referencing

### 6.1 How to reference a diagram in prose

When referencing a diagram in a document, refer to it by its figure label and provide the relative path to the source file. Do not embed diagrams inline without alt-text.

Example reference in prose:
> The trust zone structure is shown in Figure 1 (`diagrams/ot-trust-boundary-overview.mmd`). The diagram identifies five operational zones and marks enforcement points at each trust boundary crossing.

### 6.2 Diagram placement

Diagrams should appear close to the prose that explains them. A diagram that requires reading ahead several paragraphs to understand is poorly placed. A diagram that is not referenced in adjacent prose should be moved or removed.

### 6.3 Diagram standards reference

All diagrams committed to this repository must comply with the standards defined in [`docs/diagrams/README.md`](../diagrams/README.md). When deviating from those standards for specific reasons, document the deviation in the diagram specification file.

---

## 7. Speculative Language

### 7.1 When speculation is appropriate

Speculation — projections, hypotheses, open research questions — is appropriate in clearly delineated sections such as "Future Direction" or "Open Questions." It is not appropriate in sections that describe current architecture, current threats, or current PoC implementation.

When speculating, use language that makes the speculative nature explicit:
- "One possible direction is..."
- "Further research is needed to determine whether..."
- "It is reasonable to expect that..."
- "Whether this approach would scale to..."

### 7.2 What speculation is not appropriate in this repository

Do not speculate about:
- Future regulatory requirements for OT security
- Future threat actor capabilities
- Market adoption of identity-aware OT architecture
- Future versions of BASIS or planned features

These topics are outside the scope of this repository and their inclusion would shift the tone toward advocacy or product positioning.

---

## 8. Structural Conventions

### 8.1 Section numbering

White paper sections use a numeric prefix in the filename (e.g., `02-current-state-of-ot-authorization.md`) that corresponds to their position in the paper narrative. Sub-sections within a major section use `## ` and `### ` heading levels. Do not use heading levels below `### ` in white paper sections — if more subdivision is needed, restructure the content.

### 8.2 Introduction paragraph

Each major section should open with a brief paragraph that states what the section covers and why it matters in the context of the paper. This paragraph should not repeat the section title and should not begin with "This section will cover..." — state the substance directly.

### 8.3 Length and depth

White paper sections should be substantive but not exhaustive. The standard for appropriate depth is: enough detail to support the argument or description, not enough to substitute for the primary sources or implementation documentation. A section that requires a reader to read external documents to understand the core argument is too thin. A section that attempts to replace external documents is too detailed.

### 8.4 Lists and prose

Use prose for connected analytical content. Use lists for genuinely enumerable items — components, threat categories, protocol names — where the items are parallel and the sequence or relationship between them does not need to be expressed in prose. Do not use bullet lists as a substitute for paragraphs that should be written as paragraphs.
