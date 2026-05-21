# Threat Modeling and Security Considerations

This section identifies threats relevant to identity-aware authorization systems deployed in OT environments. The analysis follows a data flow and trust boundary approach: for each threat, the relevant system components and boundaries are identified, the mechanism of the threat is described, and the architecture's response is analyzed alongside the residual risks and operational constraints that persist regardless of how that response is implemented.

This is not a comprehensive threat model for OT security broadly. It is scoped to threats that are specifically relevant to, or materially affected by, the authorization architecture described in this paper. Threats addressed by independent controls — network segmentation, physical security, firmware integrity — are not treated in detail here, though their interaction with the authorization layer is noted where relevant.

---

## T-01: Unauthorized Command Issuance

**Description.** An actor without appropriate authorization issues a command to a field device — a setpoint change, a control override, a mode switch — either by impersonating an authorized principal or by exploiting a gap in authorization coverage.

**Mechanism.** In environments that rely on protocol-centric trust, any device or system on the same network segment as a BAS controller can issue valid commands without authentication. An attacker who gains access to the OT network segment — through a compromised jump host, a misconfigured VPN, or a rogue device — can issue commands that are indistinguishable from legitimate ones. In environments where authorization is partially deployed, commands routed through unenforced paths bypass the policy layer entirely.

**Architectural mitigations.** Identity-aware authorization requires that all commands carry verified identity context and that authorization decisions are applied at the enforcement point nearest to the field device. Protocol adapters normalize commands and submit them to the policy engine before forwarding. Commands that do not carry valid identity context are rejected at the enforcement point, not forwarded.

**Residual risks.** Authorization coverage depends entirely on enforcement point deployment. As Section 05 established, the field device zone is enforcement-downstream by design: devices receive commands that have been authorized at the edge boundary but cannot independently verify that authorization. Field devices that receive commands through paths bypassing the protocol adapter and enforcement point — direct serial connections, maintenance ports, out-of-band vendor tools — are outside the authorization layer's coverage entirely. Physical access controls and maintenance procedure discipline address these gaps, but the gap itself is structural to the model rather than a deployment deficiency. The authorization architecture improves coverage for the majority of access paths; it does not claim universal coverage.

---

## T-02: Rogue or Compromised Gateway

**Description.** An OT gateway or protocol adapter is replaced, misconfigured, or compromised, allowing it to forward unauthorized commands, suppress authorized commands, or manipulate identity context on traffic passing through it.

**Mechanism.** Gateways are high-value targets precisely because of their position in the trust boundary model described in Section 05: they mediate all traffic between zones, translate between protocol domains, and carry the identity context that downstream enforcement points depend on. A compromised gateway can strip identity context from inbound traffic (causing the enforcement point to reject legitimate requests), inject forged identity context (causing the policy engine to authorize requests from unauthorized principals), or selectively forward and suppress commands in ways that are difficult to detect from the supervisory layer.

**Architectural mitigations.** The architecture's primary protection against a compromised gateway is the design principle established in Section 04: policy evaluation is centralized, not delegated. A gateway that can manipulate traffic routing cannot alter the policy engine's decisions — it can only affect which traffic reaches the enforcement point, and which does not. Device certificates give the gateway a verifiable identity that appears in the authorization decision record, establishing accountability for gateway-level actions independent of the traffic it carries. Mutual TLS between the gateway and enforcement points provides channel integrity, constraining the gateway's ability to impersonate adjacent components. Monitoring gateway health and communication patterns through the telemetry pipeline provides behavioral visibility — though a gateway compromised at a low level may be able to influence its own health reporting.

**Residual risks.** A gateway compromised at firmware or hardware level before enrollment in the identity system may present valid credentials while behaving maliciously — a condition that device certificates and mutual TLS cannot detect, since both rely on the gateway's own cryptographic operations. Hardware attestation, supply chain controls, and physical security of the gateway hardware address this gap below the software layer. A compromised gateway can selectively suppress audit events, making its own behavior partially invisible in the central record; Section 07's analysis of audit delivery semantics noted that connectivity gaps produce the same symptom, making selective audit suppression difficult to distinguish from routine delivery delays without out-of-band integrity verification.

---

## T-03: Telemetry Spoofing

**Description.** An actor injects false telemetry data into the telemetry pipeline, causing monitoring systems, anomaly detection, and operational dashboards to reflect a system state that does not correspond to the actual physical state of the building.

**Mechanism.** Telemetry data that originates from field devices and passes through protocol adapters carries no inherent integrity guarantee in most field protocols. If the telemetry pipeline accepts data from unauthenticated sources, or if the protocol adapter can be impersonated, false readings can be injected. False telemetry does not directly enable unauthorized commands, but it can suppress legitimate anomaly detection, provide cover for ongoing attacks, or cause operators to take inappropriate manual actions based on incorrect state information.

**Architectural mitigations.** The architecture addresses telemetry integrity through path-level authentication rather than device-level attestation: protocol adapters attest the source device identity when normalizing telemetry for the pipeline, and the pipeline can reject events from unauthenticated sources. This provides integrity that is bounded by the adapter layer's own integrity rather than by the field devices themselves — a consequence of the asymmetric trust model that Section 05 identified as structural to how OT telemetry flows. Anomaly detection against behavioral baselines provides a supplementary detection layer where source authentication is insufficient, but its effectiveness depends on having established accurate baselines under normal operating conditions, a prerequisite that heterogeneous field deployments with irregular operational patterns may not easily satisfy.

**Residual risks.** Field devices that communicate over protocols without authentication cannot self-attest their identity; the protocol adapter attests on their behalf. This is not a gap in the authorization architecture — Section 05 explicitly identified the asymmetric telemetry trust model as structural to the way field devices operate, distinct from the verifiable-identity model that governs inbound commands. If the adapter is compromised (see T-02), it can inject false telemetry with valid attestation. Telemetry integrity is therefore bounded by the integrity of the adapter layer, and no architectural refinement within the authorization model raises that bound.

---

## T-04: Credential Reuse and Shared Operator Credentials

**Description.** Shared operator credentials in use across multiple principals allow a former employee, vendor, or compromised account to access OT systems indefinitely, or allow the actions of one principal to be attributed to another.

**Mechanism.** Shared credentials are common in OT environments for operational reasons described in the current-state section of this paper. When a credential is shared across multiple human operators, revocation requires changing the credential for all users simultaneously, which creates operational disruption and is therefore deferred. Even where individual credentials are in use, the revocation propagation window analyzed in Section 07 means that edge enforcement points operating on cached policy may continue honoring a revoked credential until the next successful synchronization — potentially hours after the revocation event. Compromised or decommissioned credentials may therefore remain operationally valid across portions of the deployment longer than the formal revocation timestamp implies.

**Architectural mitigations.** The identity-aware authorization model requires that individual operators authenticate with individual credentials and that those credentials are managed through the central identity provider. Credential lifecycle — including issuance, rotation, and revocation — should be automated where possible to reduce the administrative burden that drives shared-credential adoption. The identity provider should enforce MFA for operator accounts. Revocation should propagate to enforcement points rapidly and should not require manual cache invalidation.

**Residual risks.** Where operational workflow genuinely requires shared access — break-glass credentials for emergency response being the clearest example — shared credentials cannot be eliminated. The governance requirements for break-glass credentials (tight scoping, time-limited validity, enhanced audit coverage) are the same requirements that Section 07 identified as operationally difficult to maintain under exactly the conditions that trigger emergency override procedures. A break-glass process that is theoretically complete but practically unusable under real emergency conditions provides weaker protection than its design implies. The existence of break-glass credentials should be treated as a known residual risk, its governance tested under realistic conditions, and its use reviewed periodically rather than assumed to be operating as designed.

---

## T-05: Audit Log Tampering

**Description.** An actor modifies or deletes authorization decision records or security-relevant events in the audit log, either to conceal unauthorized actions or to create false evidence of actions that did not occur.

**Mechanism.** Audit logs that are stored in mutable form — on the same system that produces them, without out-of-band replication, or with administrative write access controlled by the same role that controls OT operations — can be modified by sufficiently privileged actors. In OT environments where a single operator has broad administrative access, the ability to modify audit records may be a practical reality rather than a theoretical concern.

**Architectural mitigations.** The architecture's response depends on two properties that must be designed together: administrative independence of the audit store from the operational systems it covers, and structural tamper evidence in the record itself. Administrative independence means the credentials required to write to or delete from the audit store are distinct from OT operational credentials — so the population of principals whose actions are recorded cannot also modify what is recorded. Cryptographic chaining provides tamper evidence by making modifications detectable after the fact, even when they cannot be prevented in real time. Replication to an out-of-band system provides a secondary reference that survives compromise of the primary store. As Section 07's analysis noted, the BASIS PoC explicitly deferred administrative separation as a development gap; production deployment must address this before the audit record can serve as trustworthy evidence of what the authorization layer enforced.

**Residual risks.** If an actor gains control of both the production audit store and the out-of-band replica and can modify both consistently, tampering may go undetected. This residual risk is not addressable through audit architecture alone. It depends on personnel security controls and administrative separation that are organizational properties of the deployment — properties the authorization architecture can require in its design but cannot enforce on its own.

---

## T-06: Policy Misconfiguration

**Description.** A policy is authored or deployed incorrectly, either granting access that should be denied (overly permissive policy) or denying access that should be permitted (overly restrictive policy). In either case, the authorization system enforces the wrong outcomes.

**Mechanism.** Policy authoring is a manual process in most current implementations. Policies expressed in declarative languages require that the author accurately model both the intended access rules and the data representation conventions used by the policy engine. Errors in either can produce outcomes that diverge from operational intent. Overly permissive policies may not surface immediately; overly restrictive policies produce access denials that operators report as problems.

**Architectural mitigations.** The architecture addresses policy misconfiguration primarily through observability rather than prevention: authorization decisions are recorded with sufficient context to detect incorrect outcomes after deployment, providing a feedback signal that pre-deployment testing alone cannot supply. Review workflows and automated test cases reduce misconfiguration likelihood before a policy change reaches production, but as Section 07 noted, the distributed nature of policy propagation means a misconfiguration takes effect at different enforcement points at different times — testing that passes against the pre-change state may not catch interactions with the fleet's partially-updated policy during the distribution window.

**Residual risks.** Policy testing can only validate against scenarios that are explicitly defined; novel access patterns not anticipated in test cases may produce unexpected outcomes in production. This is a structural limitation — no finite test suite can cover the full intersection of subject attributes, resource descriptors, action types, and contextual conditions that a context-aware policy evaluates. Ongoing monitoring of authorization decision patterns provides a detection layer that supplements pre-deployment testing, but that detection is retrospective: it identifies incorrect policy behavior after it has produced incorrect decisions, not before.

---

## T-07: Offline Authorization Failure Modes

**Description.** When the central policy engine is unreachable, enforcement points operating against a local policy cache exist in a state of bounded policy inconsistency whose security consequences depend on how far the cached state has drifted from the current authoritative state. Section 07's analysis of offline authorization and policy distribution examined this condition in operational detail; the threat analysis here focuses on the security consequences of that distributed systems tradeoff: an enforcement point may authorize requests that current policy would deny (if cached policy is stale following a change or revocation event), deny requests that current policy would permit (if the cache has expired before reconnection), or fail open entirely if offline behavior is not explicitly defined.

**Mechanism.** Local policy caches have a defined staleness limit that sets the maximum acceptable divergence between the distributed policy state at the enforcement point and the authoritative state at the policy engine. If connectivity is not restored within that limit, cached policy may no longer reflect current authorization state — a revoked operator credential may remain valid in the cache beyond the revocation event, with the window potentially measured in hours depending on synchronization frequency and site connectivity profile. Conversely, a cache that expires and is not refreshed before the staleness limit is reached may deny access to operators with legitimate need, particularly at sites where connectivity to central services is genuinely unreliable rather than intermittently interrupted.

**Architectural mitigations.** The staleness limit is the primary design parameter governing this threat, and its value encodes a deployment-specific judgment: which risk is larger, stale authorization decisions or excessive access denial under connectivity interruptions. There is no universally correct value; as Section 07's analysis emphasized, this is a tradeoff that the architecture makes explicit but cannot resolve on behalf of the deployment. Cache expiration behavior — what the enforcement point does when cached policy has aged beyond its limit and the policy engine remains unreachable — is a failure-mode decision that must be specified per deployment. Credential revocation events can be propagated through an out-of-band channel where the connectivity topology permits, reducing the stale-window specifically for high-urgency revocations without requiring a reduction in the general staleness limit.

**Residual risks.** No cache-based approach eliminates stale authorization risk; it bounds it. The staleness window is a persistently accepted inconsistency rather than a temporary operational condition — for sites with genuinely unreliable connectivity, the enforcement point may operate against cached policy as its steady state rather than as a fallback. The acceptable bound is deployment-specific and must be established by the teams who understand both the operational sensitivity of the resources governed and the connectivity profile of the site. What the architecture provides is a framework for making this tradeoff explicit and auditable; it cannot make the tradeoff on behalf of the deployment.

---

## T-08: Fail-Open Conditions

**Description.** Under specific failure conditions — policy engine unavailability, enforcement point crash, network partition — the authorization system defaults to allowing all traffic rather than denying it, creating a window of unrestricted access.

**Mechanism.** Fail-open conditions typically arise from one of two sources: explicit configuration that treats authorization errors as permit decisions (to avoid operational disruption), or implicit behavior where a failed enforcement point ceases to intercept traffic rather than blocking it. Fail-open is a predictable design choice in some OT contexts where operational continuity requirements dominate, but it should be treated as a known risk rather than an undocumented default.

**Architectural mitigations.** Fail-open conditions are a governance problem as much as a technical one. The technical specification — what the enforcement point does when it cannot evaluate a request — is the visible part; the harder question, examined in Section 07's analysis of fail-open governance, is which team holds organizational authority to accept that specification for a given enforcement point and under what conditions. The choice between fail-open and fail-closed should reflect a documented judgment about which failure mode is operationally less harmful for the specific resources governed, made by the stakeholders accountable for both the security posture and the operational continuity of those resources. Where fail-open is accepted for specific paths, the failure event itself must generate an alert through monitoring that is independent of the authorization infrastructure — a silent bypass is operationally indistinguishable from an authorized permit decision until a subsequent audit review detects the gap.

**Residual risks.** Any explicit fail-open configuration represents an accepted risk with two dimensions: operational and adversarial. Operationally, the failure condition may persist longer than anticipated — a brief enforcement point outage expected to last minutes may extend for hours if the root cause is not immediately diagnosable, making the unprotected window wider than the fail-open justification assumed. Adversarially, the fail-open condition may be deliberately induced — a network partition or enforcement point crash that produces an authorized-looking bypass. Both dimensions require monitoring that is independent of the authorization infrastructure itself. The governance challenge noted in Section 07 persists here: the team responsible for detecting and responding to an induced failure may not be the same team that approved the fail-open configuration, and the response ownership gap may not surface until an incident reveals it.

---

## T-09: Operator Impersonation

**Description.** An actor authenticates to the system using credentials that belong to a legitimate operator — through credential theft, phishing, session hijacking, or coercion — and performs actions that are authorized for the legitimate operator but not for the actual actor.

**Mechanism.** Identity-aware authorization can only authorize based on the identity that is presented and verified. If the credential verification succeeds — because the actor has obtained valid credentials — the policy engine treats the request as coming from the legitimate principal. The authorization system is not designed to detect that the credential has been obtained by an unauthorized party; it is designed to enforce policy for the presented identity. The identity propagation mechanism described in Section 04, which carries verified identity context through every component in the authorization chain, compounds this: a successfully impersonated credential produces audit records that attribute every downstream action to the legitimate operator, making post-event attribution dependent on out-of-band evidence rather than the authorization record itself.

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

No single control in the authorization architecture eliminates any of these threats entirely, and the residual risks across the nine entries share a common structure: they are bounded rather than eliminated, operationally persistent rather than addressable with sufficient engineering, and dependent on organizational governance properties that the architecture requires but cannot enforce. The architecture reduces the surface area across which these threats can operate, improves the detection capability available after the fact, and makes policy state and enforcement history observable — but it does not substitute for the network, physical, and personnel controls that address the gaps the authorization layer structurally cannot reach. The production deployment constraints analyzed in Section 07 — synchronization windows, degraded operation modes, ownership fragmentation, audit delivery limitations — interact directly with several of these threats, meaning the residual risk profile of a specific deployment depends not only on how the architecture is designed but on how it is operated.
