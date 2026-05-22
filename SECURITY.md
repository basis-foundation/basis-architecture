# Security Policy

## Scope

This repository contains architecture documentation, white papers, diagrams, references, and research material for the BASIS (Building Automation Secure Identity Service) project. It does not contain production application code or deployable OT security software.

The BASIS proof-of-concept discussed in this repository is a research artifact. It is not production-ready and should not be deployed into real operational technology environments.

## Reporting Security Issues

Please report security concerns by opening a private security advisory on GitHub, or by contacting the repository owners directly via the profiles listed in `.github/CODEOWNERS`.

Appropriate reports include:

- exposed credentials or secrets committed to the repository
- repository configuration issues
- unsafe or misleading deployment guidance
- documentation that could materially misrepresent the production readiness of described architectures or components
- security claims that appear inaccurate or unsupported by the analysis

## Out of Scope

This repository does not provide production security support, vulnerability remediation for deployed systems, or operational guarantees for OT environments. It is an architecture and research repository, not a supported software project.

Do not treat any architecture description, diagram, or proof-of-concept reference in this repository as a production-ready security control. The white papers in this repository explicitly and repeatedly disclaim production readiness; those disclaimers are accurate and intentional.

## Security Philosophy

The documentation in this repository intentionally emphasizes operational constraints, residual risk, and deployment tradeoffs rather than presenting identity-aware authorization as a complete or sufficient security solution.

Identity-aware authorization can improve accountability and access control in OT environments. It does not eliminate the need for network segmentation, physical security, operational governance, vendor management, monitoring, patch management, or safety engineering. The repository's analysis is explicit about this throughout.
