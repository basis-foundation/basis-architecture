# Diagram Standards

This document establishes standards for architectural diagrams used throughout the BASIS documentation corpus — including the white paper, architecture decision records (ADRs), threat models, and supporting technical references. Its purpose is to ensure that diagrams are consistent, readable, and contribute to architectural understanding rather than visual decoration.

These standards apply to all diagrams committed to this repository, regardless of tooling used to produce them.

## Diagram Ownership and Location

**Shared standards** (this document and any reusable reference diagrams) are maintained in `docs/diagrams/`.

**White paper-specific diagrams** are maintained within each paper's own `diagrams/` subdirectory, adjacent to the sections and references they support. This keeps diagrams close to the content that uses them and avoids a single flat directory becoming a dumping ground for unrelated diagram artifacts.

Current whitepaper diagram directories:

- [`whitepapers/identity-aware-authorization-for-operational-technology/diagrams/`](../../whitepapers/identity-aware-authorization-for-operational-technology/diagrams/) — diagrams for the identity-aware authorization paper

Each whitepaper `diagrams/` directory has its own `README.md` that lists its contents, usage notes, and any paper-specific deviations from these standards.

---

## 1. Purpose of Architectural Diagrams

Diagrams in this repository exist to communicate architectural structure, system boundaries, data flows, and design decisions to readers with varying levels of familiarity with the system. A diagram succeeds when it reduces the cognitive load required to understand a concept — not when it is visually impressive.

Every diagram should be able to answer a specific question. Before creating a diagram, articulate the question it answers. If no clear question exists, the diagram probably should not exist.

Diagrams are not replacements for written explanation. They complement prose by making structural relationships visible. A diagram without accompanying explanation often creates ambiguity rather than resolving it. Diagrams that require extended legend or annotation to be understood should be simplified or split into multiple diagrams.

Diagrams in this repository are not intended for marketing, product positioning, or stakeholder communications outside the engineering context. Diagrams optimized for those purposes tend to obscure architectural detail in favor of visual appeal. That tradeoff is not appropriate here.

---

## 2. Diagram Categories

Diagrams in this repository fall into four functional categories. Each category has distinct conventions and appropriate levels of abstraction.

**Conceptual diagrams** show the logical structure of a system without reference to specific technologies, products, or deployment configurations. They are used to establish architectural intent and explain design decisions. Conceptual diagrams should be intelligible to someone unfamiliar with the specific implementation. Detail that is only meaningful in an implementation context does not belong in a conceptual diagram.

**Implementation diagrams** show how a system or component is realized in a specific deployment context — including specific protocols, software components, network configurations, and infrastructure choices. Implementation diagrams are used in ADRs, proof-of-concept documentation, and deployment references. They should be clearly identified as implementation-specific so readers do not misinterpret them as universal architecture.

**Flow diagrams** trace the path of a request, event, message, or data payload through a system over time. They are used to explain authorization flows, telemetry pipelines, identity propagation chains, and protocol translation sequences. Flow diagrams emphasize sequence and interaction rather than structural composition.

**Threat model diagrams** identify system components, data flows, and trust boundaries for the purpose of threat analysis. They follow Data Flow Diagram (DFD) conventions and are used alongside threat model documentation. They must accurately reflect system boundaries and external interactions and should not simplify away elements that carry security relevance.

When a diagram spans more than one category, it should be split. A diagram that attempts to show both conceptual structure and implementation-specific detail simultaneously tends to serve neither purpose well.

---

## 3. Visual Style Guidelines

The overriding style principle is clarity. Visual complexity that does not contribute to understanding should be removed.

**Use a minimal visual vocabulary.** Rectangles, cylinders, arrows, and dashed boundaries are sufficient for the majority of diagrams in this repository. Avoid three-dimensional shapes, shadows, gradients, stock icons, and decorative elements. They add visual noise without improving comprehension.

**Prefer flat shapes.** Flat rectangles and labeled boxes communicate component identity and hierarchy more reliably than iconographic representations, which require the reader to learn an icon vocabulary before interpreting the diagram.

**Use arrows to represent directed relationships.** Arrow direction should be consistent with data flow, request direction, or control direction as appropriate to the diagram type. Bidirectional arrows should only be used when both directions carry distinct semantic meaning and cannot be shown separately.

**Keep font sizes consistent and legible.** Do not use font sizes below 10pt in exported diagrams. Label text must remain legible at the zoom level at which the diagram will be rendered in its published context (typically a PDF or rendered Markdown viewport).

**Avoid crossing lines where possible.** Diagrams with many line crossings are difficult to read. If avoiding crossings requires layout changes that obscure structural relationships, it is preferable to split the diagram.

**Use whitespace intentionally.** Crowded diagrams are harder to read than diagrams with generous spacing between components. Whitespace is not wasted space — it communicates grouping and separation.

---

## 4. Trust Boundary Conventions

Trust boundaries are a primary architectural concept in BASIS documentation and must be represented consistently across all diagrams.

**Represent trust boundaries as dashed enclosing borders.** A trust boundary encloses a group of components that share a common trust level. The dashed border style visually distinguishes trust zones from functional groupings, which use solid borders where grouping is represented at all.

**Label every trust boundary.** Each dashed boundary must carry a short, descriptive label that names the trust zone (e.g., `Field Device Zone`, `Supervisory Zone`, `Enterprise Network`, `Central Services`). Labels should be placed at the top-left of the enclosing border in a consistent position across diagrams.

**Show the enforcement point at every boundary crossing.** Where a communication path crosses a trust boundary, the diagram must show the component responsible for enforcement at that crossing — typically a gateway, protocol adapter, or policy enforcement point. Do not draw an arrow directly from one trust zone to another without representing the intermediary.

**Use consistent zone terminology.** Trust zone names should match the terminology defined in the glossary and used in the white paper. Do not introduce new zone names in individual diagrams without updating the shared vocabulary.

**Do not use color alone to distinguish trust boundaries.** Color may supplement the dashed-border convention, but trust zones must be distinguishable without color for accessibility reasons (see Section 12).

---

## 5. Color Usage Guidance

Color in diagrams should serve a specific semantic purpose. Do not use color purely for visual variety.

**Define a limited, consistent color palette.** The following semantic color assignments apply across all diagrams in this repository:

| Semantic meaning                   | Suggested fill   | Notes                                            |
| ---------------------------------- | ---------------- | ------------------------------------------------ |
| Field device / OT component        | Light blue-gray  | Components at the device or controller layer     |
| Identity / authorization component | Light orange     | Policy engines, identity stores, token services  |
| Gateway / enforcement point        | Light yellow     | Components that enforce trust boundary crossings |
| Telemetry / audit infrastructure   | Light green      | Audit pipelines, telemetry buses, log stores     |
| External system / untrusted zone   | Light red        | Systems outside the defined trust model          |
| Operator / human actor             | White or no fill | Human principals in flow diagrams                |

These are baseline suggestions. If a diagram toolchain requires specific hex values for consistency, document the palette in a companion `palette.md` file in this directory.

**Use color sparingly on arrows.** Arrow color should only be used to distinguish multiple concurrent flows on the same diagram when those flows would otherwise be ambiguous. If arrows require color to be meaningful, consider whether the diagram contains too many flows to be shown simultaneously.

**Ensure sufficient contrast.** Fill colors must provide sufficient contrast with the text labels overlaid on them. Light fills with dark text is the reliable standard. Do not use fills that make labels difficult to read.

---

## 6. Labeling Standards

**Label every component.** No component should appear in a diagram without a label that identifies what it is. Unlabeled boxes require the reader to infer identity from position, which introduces ambiguity.

**Use consistent component names.** Component labels should match the names used in the glossary, white paper, and ADRs. Do not use abbreviations that are not defined in the glossary. Do not introduce informal shorthand in diagrams that differs from the terminology used in prose documentation.

**Label all arrows.** Every arrow should carry a short label describing what it represents — a protocol, a data type, a request type, or a relationship (e.g., `BACnet/IP`, `Authorization Request`, `Policy Update`, `Telemetry`). Unlabeled arrows require the reader to infer semantics from context, which is unreliable.

**Use verb phrases for flow diagram arrows.** In flow diagrams, arrow labels should describe the action being performed (e.g., `submits authorization request`, `returns allow/deny decision`, `forwards normalized event`). This makes sequence and causality explicit.

**Use noun phrases for structural diagram arrows.** In structural diagrams, arrow labels should describe the relationship or data type (e.g., `MQTT`, `Policy`, `Device Certificate`, `Audit Event`).

**Keep labels short.** Labels should be as short as they can be while remaining unambiguous. If a label requires more than a few words to be meaningful, the component or relationship likely needs to be explained in the accompanying prose rather than in the diagram label.

---

## 7. Diagram Simplification Rules

Diagrams should represent the level of detail that is necessary for the reader to understand the architectural point being made — no more.

**One diagram, one question.** Each diagram should answer a single, specific architectural question. If a diagram is trying to show system structure, component interactions, and deployment topology simultaneously, it should be split into separate diagrams for each concern.

**Aggregate where repetition obscures structure.** If a system contains many instances of the same component type (e.g., dozens of BACnet field devices), show a representative instance with a label indicating multiplicity (e.g., `BACnet Field Device (n)`). Repeating identical components adds visual clutter without adding information.

**Suppress internal detail at the wrong level of abstraction.** A conceptual diagram of the authorization system does not need to show internal database schemas or internal service APIs. A flow diagram for a specific authorization sequence does not need to show all possible paths — only the path relevant to the sequence being documented.

**Do not show what is not architecturally significant.** Not every component in a system needs to appear in every diagram. Show only the components that are relevant to the diagram's specific question. Components that appear but play no role in the depicted flow or structure add noise and may mislead the reader about their significance.

**Review diagrams for removable elements before committing.** Before committing a diagram, identify any element that could be removed without changing what the diagram communicates. If an element can be removed without loss, remove it.

---

## 8. Recommended Diagram Types

The following diagram types are appropriate for specific documentation contexts within this repository.

**Context diagrams** show the system in relation to its external actors and adjacent systems, without depicting internal structure. Use context diagrams to establish scope at the beginning of a document, or to show where a component sits relative to the broader environment. Context diagrams are always conceptual.

**Component diagrams** show the internal structure of a system — the components it is composed of, and the relationships between them. Use component diagrams when the internal organization of a system is architecturally significant and needs to be explained. Component diagrams may be either conceptual or implementation-level, and should be clearly identified as one or the other.

**Sequence diagrams** show the ordered interactions between components over time for a specific scenario. Use sequence diagrams to document authorization flows, identity propagation sequences, audit event pipelines, and protocol translation steps. Sequence diagrams are well-supported by Mermaid and are preferable to flow diagrams for temporal interactions.

**Data flow diagrams (DFDs)** show how data moves through a system, crossing trust boundaries and passing through processing components. Use DFDs for threat model documentation. Follow standard DFD conventions: processes as circles or rounded rectangles, data stores as parallel lines or cylinders, external entities as rectangles, and data flows as directed arrows.

**Deployment diagrams** show how software components are distributed across physical or virtual infrastructure. Use deployment diagrams in implementation-specific documentation where deployment topology is relevant to the architectural decision being described.

Avoid diagram types that do not correspond to a recognized convention unless there is a specific justification. Novel diagram vocabularies impose a learning cost on readers.

---

## 9. Mermaid vs Draw.io Guidance

Both Mermaid and Draw.io are acceptable diagram tools for this repository. Choose based on the needs of the specific diagram.

**Use Mermaid for:**

- Sequence diagrams where temporal ordering of interactions is the primary content
- Simple flow diagrams with a limited number of components and flows
- Diagrams that are embedded directly in Markdown documents and must render in GitHub or documentation toolchains
- Diagrams that should be version-controlled as text and reviewed in pull requests with readable diffs

Mermaid files use the `.mmd` extension and should be committed alongside the Markdown files that reference them.

**Use Draw.io for:**

- Architecture diagrams with complex layout requirements, trust boundary overlays, or multi-zone compositions
- Diagrams that require precise visual control over component placement and grouping
- Diagrams that will be exported to PDF or high-resolution image formats for white paper inclusion
- Threat model diagrams with DFD conventions

Draw.io files use the `.drawio` extension and should be committed as XML source files. Exported image files (PNG, SVG) may be committed alongside the source for rendering convenience but the `.drawio` source is the authoritative file.

**Do not commit only an exported image without a source file.** Exported images cannot be edited or reviewed as text. Every diagram must have a committed source file that can be modified and re-exported.

---

## 10. File Naming Conventions

Diagram file names must be descriptive, lowercase, hyphen-separated, and indicate the diagram's subject and type. Do not use generic names like `diagram1.drawio` or `architecture.mmd`.

**Format:** `{subject}-{type}.{extension}`

Where:

- `{subject}` describes the specific system, flow, or concept depicted
- `{type}` optionally describes the diagram category (e.g., `overview`, `flow`, `threat-model`, `sequence`)
- `{extension}` is the tooling extension (`.drawio` or `.mmd`)

**Examples:**

| File name                             | Description                                                           |
| ------------------------------------- | --------------------------------------------------------------------- |
| `ot-trust-boundary-overview.drawio`   | Conceptual overview of OT trust zones and boundary enforcement points |
| `identity-aware-control-flow.mmd`     | Sequence diagram of an identity-aware device control request          |
| `basis-poc-architecture-v1.drawio`    | Implementation-level PoC deployment architecture, version 1           |
| `authorization-decision-sequence.mmd` | Mermaid sequence diagram of the policy evaluation flow                |
| `audit-pipeline-data-flow.drawio`     | DFD of the audit event pipeline from enforcement point to storage     |
| `telemetry-ingestion-overview.drawio` | Conceptual diagram of telemetry ingestion architecture                |
| `protocol-adapter-component.drawio`   | Component diagram of a protocol adapter's internal structure          |
| `bas-zone-context.drawio`             | Context diagram showing BAS operational zones and external actors     |

**Versioning in file names** should only be used when multiple versions of the same diagram are intentionally retained — for example, when documenting architectural evolution across PoC phases. When a diagram is being updated in place, use version control (git) rather than file name versioning. The `-v1`, `-v2` suffix should be reserved for cases where both versions need to remain accessible simultaneously.

---

## 11. Versioning and Source Control

All diagram source files must be committed to the repository alongside the documentation that references them. Diagrams are part of the documentation and are subject to the same review process as written content.

**Commit diagram source files, not only exports.** As noted in Section 9, the `.drawio` or `.mmd` source file is the authoritative artifact. Exported images are derivative. When both are committed, they must be kept in sync — an outdated exported image that no longer matches its source creates confusion.

**Reference diagrams by file path, not by embedded image.** Where a document references a diagram that is maintained as a separate file, use a relative file path reference rather than embedding the exported image inline, unless the diagram toolchain renders the source directly. This ensures that the reference is to the current version of the diagram.

**Review diagram changes in pull requests.** Changes to `.mmd` files are reviewable as text diffs. Changes to `.drawio` files are XML diffs that are not always human-readable, but the file should still be committed and reviewed for completeness. When a structural change to a diagram is made, the PR description should describe what changed and why, as it would for a code or documentation change.

**Do not commit binary exports as the sole record.** PNG and SVG exports may be committed for convenience, but a repository that contains only exported images with no source files is not maintainable. If source files are lost or never committed, diagrams cannot be updated without being fully redrawn.

---

## 12. Accessibility and Readability Considerations

Diagrams must be readable by people with color vision deficiencies and must remain intelligible when printed in grayscale.

**Do not use color as the sole distinguishing attribute.** Trust zones, component types, and flow categories must be distinguishable by shape, pattern, border style, or label — not only by color. Color may supplement these distinctions but cannot replace them.

**Use patterns or border styles to distinguish zones when color is unavailable.** Dashed borders for trust boundaries (as defined in Section 4) provide a non-color distinction. Internal groupings may use solid borders, rounded corners, or background shading in patterns that reproduce in grayscale.

**Test diagrams in grayscale before committing.** Export the diagram to grayscale and verify that all components remain distinguishable and all labels remain legible. Most diagram tools support grayscale export or preview.

**Ensure font sizes are legible at intended render sizes.** Diagrams that will be embedded in Markdown documentation and rendered at screen resolution have different minimum font size requirements than diagrams exported at high resolution for PDF inclusion. Verify legibility in both contexts if a diagram is used in multiple output formats.

**Keep text horizontal.** Rotated text on arrows or component labels is harder to read and is inaccessible to screen readers in some export formats. Where a label would require rotation to fit, shorten the label or adjust the diagram layout.

**Write alt-text for diagrams embedded in Markdown.** When a diagram is embedded in a Markdown document as an image, the image reference must include descriptive alt-text that conveys the structural content of the diagram for readers who cannot see the image.

```markdown
![Diagram showing three OT trust zones — field device, supervisory, and enterprise — with gateway enforcement points at each zone boundary](../diagrams/ot-trust-boundary-overview.png)
```

Alt-text for diagrams should describe structure and content, not just label the diagram type. "Architecture diagram" is not useful alt-text.
