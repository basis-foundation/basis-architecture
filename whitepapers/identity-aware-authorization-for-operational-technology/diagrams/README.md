# Diagrams — Identity-Aware Authorization for Operational Technology

This directory contains diagrams that are specific to the white paper *Identity-Aware Authorization for Operational Technology*. These diagrams illustrate the architectural concepts described in the paper and are not intended as shared reference diagrams for other papers or documentation.

---

## Contents

| File | Type | Description | Status |
|---|---|---|---|
| [`identity-aware-authorization-flow.mmd`](identity-aware-authorization-flow.mmd) | Mermaid source | **Primary architecture diagram.** Full authorization flow: operator identity initiation, identity propagation, protocol normalization, authorization request/decision cycle, distributed enforcement, operational command execution, immutable audit generation, control plane / data plane separation | Complete |
| [`identity-aware-authorization-flow-spec.md`](identity-aware-authorization-flow-spec.md) | Specification | Full specification for the primary diagram: trust boundary rationale, component responsibilities, flow semantics, authorization semantics, control/data plane separation, offline operation, Draw.io authoring guidance, pre-commit checklist | Complete |
| [`ot-trust-boundary-overview.mmd`](ot-trust-boundary-overview.mmd) | Mermaid source | Trust zone overview: operational zones, protocol boundaries, enforcement points, control plane / data plane flows, audit event routing | Complete |
| [`ot-trust-boundary-overview-spec.md`](ot-trust-boundary-overview-spec.md) | Specification | Full diagram specification: title, layout description, component list, trust boundary definitions, style guide, color palette, Mermaid starter code, Draw.io authoring guidance, and pre-commit checklist | Complete |

---

## Usage

The Mermaid source file (`ot-trust-boundary-overview.mmd`) is suitable for embedding in Markdown documentation and renders in GitHub and most Markdown toolchains. It is the working reference version of the diagram.

The specification file (`ot-trust-boundary-overview-spec.md`) contains guidance for producing a higher-fidelity Draw.io version suitable for PDF export and white paper publication. The Draw.io source file (`ot-trust-boundary-overview.drawio`) should be committed to this directory when it is produced, alongside a PNG export in `../exported/`.

---

## Diagram Standards

All diagrams in this directory must conform to the standards defined in [`docs/diagrams/README.md`](../../../../docs/diagrams/README.md). This includes:

- Trust boundaries represented as dashed enclosing borders with labels
- Authorization enforcement points marked with `[AuthZ]` or equivalent
- Control plane and data plane flows visually distinguished
- All components labeled; all arrows labeled
- Diagrams legible in grayscale
- Source files committed alongside any exported images

Deviations from those standards should be documented in the relevant specification file with a rationale.

---

## Exported Images

Exported images (PNG, SVG) are stored in [`../exported/`](../exported/). The exported directory contains only derivative files — the source file in this directory is the authoritative artifact. When a source file is updated, the corresponding exported files must be regenerated.

Do not commit exported images without a corresponding source file.
