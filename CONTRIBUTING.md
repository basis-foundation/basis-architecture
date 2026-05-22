# Contributing

This repository contains architecture documentation, white papers, diagrams, references, and research material for the BASIS (Building Automation Secure Identity Service) project. Contributions should preserve the repository's restrained engineering tone and operationally grounded analysis.

## Contribution Scope

Appropriate contributions include:

- corrections to terminology, references, or cross-references
- improvements to architectural clarity or analytical precision
- diagram updates and companion specification documents
- additional operational constraints or tradeoff analysis
- threat-model refinements
- documentation consistency fixes

This repository is not the right place for:

- product marketing or positioning
- speculative roadmap language
- unsupported or exaggerated security claims
- generic OT security content unrelated to the architecture described in the white papers
- changes that characterize the BASIS PoC as production-ready

## Writing Expectations

Before contributing, review:

- [`docs/glossary.md`](docs/glossary.md) — canonical terminology definitions
- [`docs/architecture-principles.md`](docs/architecture-principles.md) — guiding architectural principles
- [`docs/standards/writing-guidelines.md`](docs/standards/writing-guidelines.md) — tone, prohibited language, and style
- [`docs/standards/diagram-standards.md`](docs/standards/diagram-standards.md) — diagram conventions and visual style

Contributions should use consistent terminology, preserve architectural restraint, acknowledge limitations and residual risk, and treat OT operational constraints as first-order concerns rather than edge cases to be addressed later.

## Diagrams

Diagram contributions should include source files (`.mmd` or `.drawio`) and, where the diagram is used in a white paper section, a companion specification document following the pattern established in the existing `diagrams/` directories. Exported images belong in the `exported/` subdirectory and should not be committed without a corresponding source file.

## Pull Requests

Use pull requests for all changes. A useful pull request describes what changed, explains why the change improves the repository, and identifies any affected sections, diagrams, or references. Avoid mixing unrelated changes in a single pull request.

## Review Criteria

Reviews prioritize architectural accuracy, operational realism, terminology consistency, and restraint in claims. The goal is not to make the repository more persuasive — it is to make it more accurate, analytically useful, and technically grounded.

The repository's value comes from its willingness to document complexity, residual risk, and unresolved tradeoffs honestly. Contributions that increase precision or surface legitimate constraints are more valuable than contributions that simplify the analysis.
