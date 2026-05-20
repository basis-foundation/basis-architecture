# Threat Modeling and Security Considerations

This section identifies threats relevant to identity-aware authorization systems deployed in OT environments. The analysis follows a data flow and trust boundary approach: for each threat, the relevant system components and boundaries are identified, the mechanism of the threat is described, and mitigations within the authorization architecture are noted alongside residual risks.

This is not a comprehensive threat model for OT security broadly. It is scoped to threats that are specifically relevant to, or materially affected by, the authorization architecture described in this paper. Threats addressed by independent controls — network segmentation, physical security, firmware integrity — are not treated in detail here, though their interaction with the authorization layer is noted where relevant.

---

## T-01: Unauthorized Command Issuance

**Description.** An actor without appropriate authorization issues a command to a field device — a setpoint change, a control override, a mode switch — either by impersonating an authorized principal or by exploiting a gap in authorization coverage.

**Mechanism.** In environments that rely on protocol-centric trust, any device or system on the same network segment as a BAS controller can issue valid commands without authentication. An attacker who gains access to the OT network segment — through a compromised jump host, a misconfigured VPN, or a rogue device — can issue commands that are indistinguishable from legitimate ones. In environments where authorization is partially deployed, commands routed through unenforced paths bypass the policy layer entirely.

**Architectural mitigations.** Identity-aware authorization requires that all commands carry verified identity context and that authorization decisions are applied at the enforcement point nearest to the field device. Protocol adapters normalize commands and submit them to the policy engine before forwarding. Commands that do not carry valid identity context are rejected at the enforcement point, not forwarded.

**Residual risks.** Authorization coverage depends on enforcement point deployment. Field devices that receive commands through paths that bypass the protocol adapter and enforcement point — for example, through direct serial connections, maintenance ports, or out-of-band vendor tools — are not protected by the authorization layer. Physical access controls and maintenance procedure discipline are required to close these gaps. Authorization alone cannot address direct physical access to field devices.

---

## T-02: Rogue or Compromised Gateway

**Description.** An OT gateway or protocol adapter is replaced, misconfigured, or compromised, allowing it to forward unauthorized commands, suppress authorized commands, or manipulate identity context on traffic passing through it.

**Mechanism.** Gateways are high-value targets because they sit at trust boundaries and mediate all traffic between zones. A compromised gateway can strip identity context from inbound traffic (causing the enforcement point to reject legitimate requests), inject forged identity context (causing the policy engine to authorize requests from unauthorized principals), or selectively forward and suppress commands in ways that are difficult to detect from the supervisory layer.

**Architectural mitigations.** Gateways should authenticate to the central identity provider using device certificates, and their identities should be included in the authorization decision record. Policy decisions should be made centrally, not delegated to the gateway, so a compromised gateway cannot alter policy outcomes — only traffic routing. Mutual TLS between the gateway and the enforcement point provides assurance that traffic has not been intercepted in transit. Gateway health and communication patterns should be monitored through the telemetry pipeline.

**Residual risks.** A gateway that is compromised at the firmware or hardware level before enrollment in the identity system may present valid credentials while behaving maliciously. Hardware attestation, supply chain controls, and physical security of gateway hardware are necessary complements to software-level controls. A compromised gateway can selectively drop audit events, making its own behavior partially invisible; out-of-band integrity checking of audit records is advisable.

---

## T-03: Telemetry Spoofing

**Description.** An actor injects false telemetry data into the telemetry pipeline, causing monitoring systems, anomaly detection, and operational dashboards to reflect a system state that does not correspond to the actual physical state of the building.

**Mechanism.** Telemetry data that originates from field devices and passes through protocol adapters carries no inherent integrity guarantee in most field protocols. If the telemetry pipeline accepts data from unauthenticated sources, or if the protocol adapter can be impersonated, false readings can be injected. False telemetry does not directly enable unauthorized commands, but it can suppress legitimate anomaly detection, provide cover for ongoing attacks, or cause operators to take inappropriate manual actions based on incorrect state information.

**Architectural mitigations.** Telemetry sources should be authenticated as part of the device identity model. Protocol adapters should attest the source device identity when injecting normalized telemetry into the pipeline. The telemetry pipeline should reject events from sources that cannot present valid credentials. Anomaly detection should include behavioral baselines that can flag telemetry that is statistically inconsistent with expected device behavior, as a supplement to source authentication.

**Residual risks.** Field devices that communicate over protocols without authentication cannot self-attest their identity; the protocol adapter attests on their behalf. If the adapter itself is compromised (see T-02), it can inject false telemetry with valid attestation. Telemetry integrity is therefore bounded by the integrity of the adapter layer.

---

## T-04: Credential Reuse and Shared Operator Credentials

**Description.** Shared operator credentials in use across multiple principals allow a former employee, vendor, or compromised account to access OT systems indefinitely, or allow the actions of one principal to be attributed to another.

**Mechanism.** Shared credentials are common in OT environments for operational reasons described in the current-state section of this paper. When a credential is shared across multiple human operators, revocation requires changing the credential for all users simultaneously, which creates operational disruption and is therefore deferred. Compromised shared credentials may remain valid long after the compromise event.

**Architectural mitigations.** The identity-aware authorization model requires that individual operators authenticate with individual credentials and that those credentials are managed through the central identity provider. Credential lifecycle — including issuance, rotation, and revocation — should be automated where possible to reduce the administrative burden that drives shared-credential adoption. The identity provider should enforce MFA for operator accounts. Revocation should propagate to enforcement points rapidly and should not require manual cache invalidation.

**Residual risks.** Where operational workflow genuinely requires shared access (e.g., break-glass credentials for emergency response), shared credentials cannot be eliminated. Break-glass credentials should be tightly scoped, require dual-person authorization where feasible, be time-limited, and generate enhanced audit events. The existence of break-glass credentials should be treated as a known residual risk, documented, and reviewed periodically.

---

## T-05: Audit Log Tampering

**Description.** An actor modifies or deletes authorization decision records or security-relevant events in the audit log, either to conceal unauthorized actions or to create false evidence of actions that did not occur.

**Mechanism.** Audit logs that are stored in mutable form — on the same system that produces them, without out-of-band replication, or with administrative write access controlled by the same role that controls OT operations — can be modified by sufficiently privileged actors. In OT environments where a single operator has broad administrative access, the ability to modify audit records may be a practical reality rather than a theoretical concern.

**Architectural mitigations.** Audit events should be written to an append-only store that is administratively separate from the OT operational infrastructure. Write access to the audit store should require credentials that are distinct from OT operational credentials. Cryptographic chaining of audit records — where each record includes a hash of its predecessor — allows tampering to be detected even if individual records are modified. Replication of audit records to an out-of-band system (outside the administrative control of OT operations) provides a secondary copy that can be used for integrity verification.

**Residual risks.** If an attacker gains control of both the production audit store and the out-of-band replica, and can modify both consistently, tampering may go undetected. This threat is not practically addressable through audit architecture alone — it requires personnel security controls that limit the population of principals with simultaneous access to both systems.

---

## T-06: Policy Misconfiguration

**Description.** A policy is authored or deployed incorrectly, either granting access that should be denied (overly permissive policy) or denying access that should be permitted (overly restrictive policy). In either case, the authorization system enforces the wrong outcomes.

**Mechanism.** Policy authoring is a manual process in most current implementations. Policies expressed in declarative languages require that the author accurately model both the intended access rules and the data representation conventions used by the policy engine. Errors in either can produce outcomes that diverge from operational intent. Overly permissive policies may not surface immediately; overly restrictive policies produce access denials that operators report as problems.

**Architectural mitigations.** Policy changes should be subject to review and approval workflows distinct from those for OT operational changes. Policy testing — including automated simulation of known-good and known-bad access scenarios — should be part of the deployment pipeline for policy changes. Observable authorization decisions (as described in the architecture principles) allow policy behavior to be inspected against expected outcomes before and after changes. Policy changes should be recorded in the audit log with the identity of the author and reviewer.

**Residual risks.** Policy testing can only validate against scenarios that are explicitly defined. Novel access patterns that were not anticipated in test cases may produce unexpected outcomes in production. Ongoing monitoring of authorization decision patterns — with alerting on anomalous access grants or unusual denial volumes — provides a detection capability that supplements pre-deployment testing.

---

## T-07: Offline Authorization Failure Modes

**Description.** When the central policy engine is unreachable, enforcement points relying on a local policy cache may authorize requests that should be denied (if cached policy is stale), deny requests that should be authorized (if the cache has expired), or fail open (if offline behavior is not explicitly defined).

**Mechanism.** Local policy caches have a defined staleness limit. If connectivity is not restored within that limit, cached policy may no longer reflect current authorization state — for example, a revoked operator credential may remain valid in the cache beyond the revocation event. Conversely, a cache that expires and is not refreshed may deny access to operators with legitimate need.

**Architectural mitigations.** Cache staleness limits should be set based on the operational risk of stale policy in each deployment context. Shorter limits reduce the window for stale authorization decisions but increase the impact of connectivity interruptions. Cache expiration behavior — what happens when cached policy expires and the policy engine is still unreachable — must be explicitly defined: the choice between failing open, failing closed, or preserving the most recent cached decision should be documented per deployment. Credential revocation events should be propagated out-of-band where possible, rather than relying solely on cache invalidation.

**Residual risks.** No cache-based approach can fully eliminate the risk of stale authorization decisions. The acceptable staleness window is an operational judgment call that involves tradeoffs between security and availability that are specific to each environment. This tradeoff should be explicitly documented and periodically reviewed.

---

## T-08: Fail-Open Conditions

**Description.** Under specific failure conditions — policy engine unavailability, enforcement point crash, network partition — the authorization system defaults to allowing all traffic rather than denying it, creating a window of unrestricted access.

**Mechanism.** Fail-open conditions typically arise from one of two sources: explicit configuration that treats authorization errors as permit decisions (to avoid operational disruption), or implicit behavior where a failed enforcement point ceases to intercept traffic rather than blocking it. Fail-open is a predictable design choice in some OT contexts where operational continuity requirements dominate, but it should be treated as a known risk rather than an undocumented default.

**Architectural mitigations.** Default authorization behavior under failure conditions must be explicitly specified per deployment. The choice between fail-open and fail-closed should be documented with a clear rationale, reviewed by both security and operations stakeholders, and revisited when operational context changes. Where fail-open is chosen for specific paths to preserve operational continuity, those paths should be narrowly defined and accompanied by enhanced monitoring and alerting. Enforcement point failures should generate immediate alerts — a silent failure that results in bypass is far more dangerous than a noisy failure that is immediately detected and addressed.

**Residual risks.** Any explicit fail-open configuration represents an accepted risk. The residual risk is that the failure condition is longer-lived than anticipated, or that an attacker deliberately induces the failure condition to bypass authorization. Detection of induced failures requires monitoring that is independent of the authorization infrastructure itself.

---

## T-09: Operator Impersonation

**Description.** An actor authenticates to the system using credentials that belong to a legitimate operator — through credential theft, phishing, session hijacking, or coercion — and performs actions that are authorized for the legitimate operator but not for the actual actor.

**Mechanism.** Identity-aware authorization can only authorize based on the identity that is presented and verified. If the credential verification succeeds — because the attacker has obtained valid credentials — the policy engine treats the request as coming from the legitimate principal. The authorization system is not designed to detect that the credential has been stolen; it is designed to enforce policy for the presented identity.

**Architectural mitigations.** Multi-factor authentication for all operator accounts reduces the risk that credential theft alone enables impersonation. Authentication events should be logged with sufficient context — source IP, device, time, geolocation where available — to enable detection of anomalous authentication patterns. Session activity monitoring can flag sessions where behavior diverges significantly from baseline patterns for the authenticated operator. Time-limited sessions with re-authentication requirements reduce the window during which a stolen session token can be used.

**Residual risks.** MFA can be bypassed through social engineering, SIM swapping, or other credential theft techniques that target both factors simultaneously. The authentication security model is a ceiling on what identity-aware authorization can achieve — authorization that rests on compromised authentication is compromised. Personnel security controls, physical security of authentication factors, and security awareness programs are necessary complements to technical authentication controls.

---

## Summary of Threats and Mitigations

| Threat | Primary mitigation layer | Residual risk |
|---|---|---|
| T-01: Unauthorized command issuance | Enforcement points at trust boundaries | Physical access paths; incomplete coverage |
| T-02: Rogue or compromised gateway | Device identity; mutual TLS; telemetry monitoring | Hardware-level compromise before enrollment |
| T-03: Telemetry spoofing | Source authentication; behavioral anomaly detection | Compromised adapter layer |
| T-04: Credential reuse | Per-operator identity; MFA; automated lifecycle | Break-glass credentials; operational workflow constraints |
| T-05: Audit log tampering | Append-only store; cryptographic chaining; out-of-band replication | Simultaneous control of all audit stores |
| T-06: Policy misconfiguration | Review workflows; automated testing; observability | Novel access patterns not covered by test cases |
| T-07: Offline authorization failure modes | Explicit cache staleness limits; defined offline behavior | Stale policy windows; revocation propagation delays |
| T-08: Fail-open conditions | Explicit failure behavior specification; independent monitoring | Induced failures; extended failure durations |
| T-09: Operator impersonation | MFA; authentication audit; session monitoring | Credential theft targeting multiple factors |

No single control in the authorization architecture eliminates any of these threats entirely. The architecture reduces attack surface, improves detection capability, and limits blast radius — it does not eliminate the need for complementary controls at the network, physical, and personnel levels.
