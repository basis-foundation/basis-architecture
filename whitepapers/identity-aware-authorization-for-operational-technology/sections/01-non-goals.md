# Non-Goals

Architectural research is most useful when it is honest about what it does not address. This section states explicitly what this paper is not attempting to argue, demonstrate, or solve. These boundaries are not limitations to be overcome in a future version — they are deliberate scope decisions that reflect the nature of the problem and the maturity of this work.

---

## This Paper Does Not Propose Replacing OT Protocols

BACnet, Modbus, LonWorks, and related field protocols are deeply embedded in existing OT infrastructure. They are not going away, and this paper does not argue that they should. These protocols solve real problems reliably within the environments they were designed for, and the installed base depending on them is substantial.

The architectural direction explored here assumes that field-level protocols remain in place and that identity-aware authorization operates above and alongside them — through protocol adapters, gateways, and enforcement points — rather than inside them. Protocol replacement is a separate, much larger problem that is outside the scope of this work.

---

## This Paper Does Not Argue for Eliminating Network Segmentation

Network segmentation remains a necessary and appropriate component of OT security architecture. VLAN separation, firewalls between operational zones, and physical isolation where feasible are not alternatives to identity-aware authorization — they are complementary controls that should coexist with it.

The critique of network segmentation as a *sole* authorization mechanism is not a critique of segmentation itself. The position is simply that segmentation alone cannot express or enforce the fine-grained access decisions that complex OT environments increasingly require. Adding identity-aware authorization does not mean removing the network boundary — it means that crossing the network boundary is no longer the only check in place.

---

## This Paper Does Not Claim the Proof-of-Concept Is Production-Ready

The BASIS proof-of-concept (basis-poc) referenced in this paper is a research implementation. It demonstrates that the architectural patterns described here are implementable and coherent, not that they are ready for deployment in production OT environments.

Production readiness requires sustained validation against real operational conditions, formal security review, interoperability testing against a meaningful range of OT hardware and protocols, and operational tooling that does not yet exist at the maturity level appropriate for critical infrastructure. This paper does not claim that threshold has been reached, and readers should not interpret the PoC work as evidence that it has.

The BASIS Core Services Distribution — the open-source ecosystem of components governed by the Basis Foundation — represents the direction that builds toward production capability. basis-core, basis-gateway, basis-adapters, and the other distribution components address the engineering separation and operational discipline that the PoC intentionally deferred. The distribution's readiness for any specific production deployment must be evaluated independently against the constraints of that deployment.

---

## This Paper Does Not Assume Cloud-Only or Cloud-Dependent Architectures

The identity and policy infrastructure described in this paper can be hosted on-premises, in a private data center, at the edge, or in a hybrid configuration. Cloud deployment is one option, not a requirement.

Many OT environments have explicit requirements — regulatory, operational, or contractual — that preclude cloud-hosted infrastructure for certain functions. The architectural patterns in this paper are designed to be deployment-agnostic. Where cloud-specific tooling is referenced as an example, it is illustrative rather than prescriptive. The core concepts — centralized policy evaluation, identity propagation, immutable audit logging — do not depend on cloud infrastructure to function.

---

## This Paper Does Not Claim That Identity-Aware Authorization Alone Solves OT Security

Identity-aware authorization addresses a specific gap in OT security posture: the absence of verified identity context in access decisions and audit records. It does not address firmware vulnerabilities, protocol weaknesses, physical security, supply chain integrity, engineering workstation hygiene, vendor security practices, or any of the many other dimensions of OT security that matter in practice.

A building automation system with identity-aware authorization and a poorly patched controller firmware is not a secure system. It is a system with better access accountability than it had before. The improvement is real and worthwhile, but it should not be characterized as more than it is. Security posture is the product of many overlapping controls; this paper addresses one of them.

---

## This Paper Does Not Prescribe a Specific Technology Stack

The protocols, software components, and infrastructure choices referenced in this paper are illustrative. Where specific tools are named — Keycloak, MQTT, OPA — they appear because they were used in the proof-of-concept implementation, not because they are the only valid choices or because they are endorsed as production solutions.

Organizations evaluating identity-aware authorization for their own OT environments should assess available tooling against their specific operational requirements, existing infrastructure, regulatory context, and support considerations. The architectural patterns described here are intended to be technology-agnostic enough to guide that assessment without constraining it to a particular vendor ecosystem.

---

## This Paper Does Not Address All OT Environment Types

The examples and scenarios in this paper draw primarily from building automation systems — HVAC, lighting, access control, and similar BAS domains. While the architectural principles are broadly applicable to OT environments, the specific operational realities of industrial process control, power generation, water treatment, or manufacturing automation may introduce constraints and requirements not fully represented here.

Readers working in domains beyond commercial and institutional building systems should treat the architectural patterns as directional rather than directly applicable without further analysis of their specific operational environment.
