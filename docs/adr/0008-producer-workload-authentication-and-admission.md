# ADR-0008: Producer Workload Authentication and Gateway Admission

## Status

Proposed

## Context

[`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md)
named the gap between `basis-adapters` normalization and `basis-gateway` trusted-producer
ingestion, defined the logical roles across that gap (operation initiator, adapter normalization
library, operation-producer runtime, gateway, protocol executor, execution-evidence producer),
and — in its §3 — distinguished six facts that are routinely conflated when discussing "trusted
producers": authenticated workload identity, authorization to act as an operation producer,
authorization to assert specific context categories, protocol evidence, execution evidence, and
human/initiating-subject identity. That document's §13 Stage 4 named, but explicitly did not
resolve, a decision gate: "which authentication/transport mechanism (mTLS, SPIFFE, OAuth
client-credentials, or another) — explicitly not selected by this document." [ADR-0007](0007-adapter-evidence-construction.md)
subsequently resolved the adjacent evidence-construction question (what constitutes adapter
evidence material, how it is canonicalized, which component computes its digest, which component
mints `reference_id`) without resolving producer authentication, and stated as a deferred decision:
"how it authenticates to `basis-gateway`" — assigned explicitly to future work, not to that ADR.

`basis-adapters` `v0.2.0` completed the adapter-owned half of the evidence gap
(`construct_adapter_evidence()`, deterministic evidence-material construction, RFC 8785
canonicalization, digest generation). `basis-schemas`' `adapter-evidence-reference` contract's
ownership metadata was corrected to reflect ADR-0007's construction split. `basis-architecture`
reconciled ADR status governance and adapter release-state references. None of this prior work
authenticates an operation-producer runtime to `basis-gateway`. `ROADMAP.md`'s **Next Producer and
Execution-Evidence Boundary** section states plainly that `basis-gateway`'s existing
`OPERATION_PRODUCER_SUBJECT_IDS` "is not the adapter-to-gateway trusted-producer contract... which
remains open."

Direct inspection of `basis-gateway`'s current source confirms the shape of that gap precisely.
`src/basis_gateway/auth/operation_producer.py` classifies an operation producer from an
**already-authenticated** `NormalizedSubject` — the module's own docstring states its scope as
answering "exactly one question, for exactly one already-authenticated caller: *is this caller also
a trusted operation producer?*" `classify_operation_producer()` performs an exact,
case-sensitive membership test of `subject.subject_id` against `GatewayConfig.operation_producer_subject_ids`
(the `OPERATION_PRODUCER_SUBJECT_IDS` environment variable, empty by default — safe-default,
no-trust-by-default). The authentication that produces that `NormalizedSubject` in the first place
is `basis-gateway`'s existing `AuthMode` dispatch (`src/basis_gateway/config.py`): either OIDC/JWT
bearer-token verification against an external identity provider, or verification of a
`basis-identity`-issued, RS256-signed `basis_local_token`. Both authentication modes are
**bearer-token** mechanisms — a caller presents a token; the gateway verifies its signature, issuer,
audience, and expiry; the verified `sub` claim becomes `subject.subject_id`. There is no
transport-level workload authentication in either path today, and no code path in `basis-gateway`
establishes a workload identity independent of a bearer credential's claims.

Direct inspection of `basis-identity`'s current source (`src/basis_identity/models/identity_context.py`)
confirms the workload-identity gap the boundary document's §12 named: `SubjectType` is a closed
three-member enum (`USER`, `SERVICE`, `DEVICE`) with no workload-specific establishment pipeline
behind `SERVICE`, and `AuthenticationProtocol` is a closed enum (`OIDC`, `SAML`, `OAUTH2`, `LDAP`,
`LOCAL_DEV`, `UNKNOWN`) with no mTLS, SPIFFE, or client-credentials member. `basis-identity` does
not today issue, verify, or model a machine/workload credential distinct from a human-facing
identity protocol, and does not yet produce `identity-evidence-reference`. This ADR does not close
either gap; it records their present state because the producer-authentication decision below is
constructed to remain independently deployable without requiring either to close first.

This ADR is the decision gate the boundary document's §13 Stage 4 named. It is the first
architectural gate required before a bounded operation-producer reference implementation (§13
Stage 5 of that document) can be built.

## Problem

How does a BASIS operation-producer runtime prove its workload identity to `basis-gateway`, and
what exact trust fact does successful authentication allow the gateway to establish? The answer
must preserve, as separate concepts that this ADR does not collapse into a generic "trusted
caller": network/transport authentication; producer workload identity; producer admission;
authorization subject identity; authorization to assert producer-owned context; adapter evidence
integrity; evidence provenance; authorization disposition; and protocol execution.

## Decision

**BASIS adopts mutual TLS (mTLS) as the first normative producer-to-gateway workload-authentication
profile.** An operation-producer runtime connects to `basis-gateway` over HTTPS and presents a
client certificate during the TLS handshake. `basis-gateway` validates that certificate against a
deployment-configured set of producer trust anchors. A successful handshake establishes an
authenticated **producer workload identity**, derived from the certificate's Subject Alternative
Name (SAN) — specifically, a URI SAN, evaluated as the primary, canonical identity form.
**Common Name is not used as the identity source.** The gateway then performs a separate,
deployment-controlled **admission** decision: whether that authenticated workload identity is
configured as an admitted operation producer. Authentication and admission are two distinct steps
(see below); neither implies the other.

This ADR does not implement mTLS, does not generate certificates, does not add gateway settings,
does not change API routes, and does not modify `basis-gateway`, `basis-identity`,
`basis-adapters`, `basis-core`, or `basis-schemas`. It records the architectural decision and
authorizes a narrowly bounded reference implementation gate (see **Bounded reference implementation
authorized** below) once this ADR is accepted.

## Trust facts established

Successful mTLS establishes only that:

> The current connection possesses a private key corresponding to a certificate that chains to an
> accepted producer trust anchor, and whose authenticated identity has been admitted by the gateway
> as an operation producer.

It must not, and does not, imply that:

- the authorization subject is the producer;
- the producer is authorized for every operation;
- all producer-supplied context is truthful;
- referenced evidence exists;
- an evidence digest proves producer authenticity;
- authorization will return `ALLOW`;
- an allowed operation was executed;
- a target device accepted or completed an operation.

Each of these negative guarantees is preserved unchanged from existing architecture. The last five
restate, without altering, the trust-establishment rules already stated in
`operation-producer-and-execution-boundary.md` §3 ("a trusted producer's assertions remain
assertions unless another component independently verifies them") and its §5 lifecycle invariants
("execution success cannot retroactively legitimize an operation that was not authorized"). This
ADR narrows *how* the gateway establishes the first fact in that chain — authenticated producer
workload identity plus admission — without touching any fact downstream of it.

## Authentication vs. admission

These are two separate questions, and the gateway processing model (below) keeps them as two
separate steps.

**Authentication** answers: *what workload identity is on this producer-to-gateway connection?*
For this first profile, it is established entirely by mTLS certificate validation — chain-of-trust
validation against configured producer trust anchors, followed by deterministic derivation of a
workload identity string from the certificate's URI SAN. Authentication produces a workload
identity or fails; it does not, by itself, grant any capability.

**Admission** answers: *is this authenticated workload identity configured or registered as an
operation producer that may assert producer-owned request context?* Admission is a
deployment-controlled decision, checked after authentication succeeds, against gateway
configuration or a future producer registry (see **Registration and revocation** below). An
authenticated certificate must not automatically imply operation-producer admission — a workload
may hold a certificate that chains to an accepted trust anchor and still not be an admitted
producer, exactly as `classify_operation_producer()` today treats a verified bearer subject that is
absent from `OPERATION_PRODUCER_SUBJECT_IDS` as `UNTRUSTED` / `SUBJECT_ID_NOT_ALLOWED` rather than
trusted-by-default.

The gateway fails closed when: certificate validation fails; producer identity cannot be derived
from the certificate (zero or more than one eligible URI SAN — see **Certificate identity profile**
below for the exact-one-SAN rule and the full fail-closed list); the authenticated identity is not
admitted; or an ordinary (non-admitted) caller attempts to assert producer-only context. This
preserves, without weakening, the existing provenance-enforcement principle already implemented in
`basis-gateway`'s `operation_aware_composition.py` (`UntrustedOperationProducerContextError`,
raised before any other composition step) and the safe-default-empty-allowlist behavior already
implemented in `classify_operation_producer()`.

## Producer vs. authorization subject

This distinction is mandatory and is not weakened by adopting mTLS.

The **operation producer** is the workload submitting the normalized operation — a connection-level,
admission-level identity, established by the mechanism this ADR selects.

The **authorization subject** is the human, service, machine, agent, or other identity whose
authority is being evaluated by `basis-core` — a policy-evaluation-level identity, established by
whatever identity/authentication chain applies to that subject (today: `basis-gateway`'s existing
bearer-token `authenticate()` dispatch; potentially, in the future, a `basis-identity` canonical
identity context carried via `identity_evidence_reference`).

They may sometimes be the same underlying identity in a given deployment, but this architecture
never requires or assumes that they are identical, and this ADR does not reuse the producer
certificate identity as the authorization subject automatically. A producer does not become
authoritative for subject identity merely because its mTLS connection is trusted. This is the same
distinction `operation-producer-and-execution-boundary.md` §2 already draws between the *operation
initiator* and the *operation-producer runtime* ("an operator using a supervisory HMI is an
operation initiator whose request is carried, but not authenticated as, the operation producer that
actually submits to the gateway"), narrowed here specifically to the authentication mechanism.
Where subject identity is asserted or derived remains a separate contract and provenance question
governed by existing `basis-identity` and operation-aware architecture — this ADR does not decide
it.

## Certificate identity profile

The canonical producer workload identity form is a **URI SAN** on the client certificate. Common
Name (CN) is not the primary identity source and must not be used as one: CN has no defined
structure across issuers, is not designed as a machine-parseable identity field, and using it would
create exactly the kind of implicit, convention-dependent trust this architecture's zero-trust
framing already rejects for network position and role membership.

A SPIFFE-style URI (`spiffe://trust-domain/path`) is structurally compatible with this design and
is not required. Because no canonical BASIS-specific URI namespace exists in current architecture,
this ADR does not invent one. The first normative profile requires one deterministic mapping rule —
the workload identity is the URI SAN value, taken verbatim from the validated certificate — and
leaves the namespace itself deployment-defined, subject to deterministic exact-string matching at
admission time. A deployment may choose a SPIFFE URI, a private URI scheme, or any other URI form
its PKI issuance process produces; the gateway does not interpret the URI's internal structure
beyond treating it as an opaque, stable identity string. Support for other SAN forms may be added
later as an additive extension; it is not decided here.

**Identity derivation is gateway-derived, never caller-supplied.** The gateway derives producer
identity only from a successfully validated client certificate. Common Name is not a fallback
identity source — if no eligible URI SAN is present, identity derivation fails outright; the gateway
does not fall back to CN, to any other SAN type, or to any other certificate field. Request-body
fields cannot set or override producer identity, consistent with this ADR's existing rejection of
caller-supplied `is_trusted_operation_producer` / `producer_trust_classification`. Arbitrary HTTP
headers cannot set or override producer identity — no header is a substitute for the
connection-level, certificate-derived fact. Authorization-subject identity cannot set or override
producer identity either, restating this ADR's **Producer vs. authorization subject** distinction at
the derivation step specifically, not only at the conceptual level.

**Exact admission matching.** For the first normative profile, admission matching is exact matching
of the authenticated URI SAN identity against deployment-controlled producer admission
configuration:

> A producer is admitted only when the URI SAN identity derived from the validated client
> certificate exactly matches an enabled producer identity in gateway-controlled admission
> configuration.

The following are explicitly rejected as admission-matching mechanisms for this first profile:
substring matching; prefix matching; suffix matching; wildcard matching; regular-expression
matching; case-insensitive normalization (unless and until a future profile explicitly standardizes
one — the first profile performs exact, case-sensitive matching); Common Name fallback; role-based
inference; issuer-only admission (admitting every certificate issued by a given CA without a
per-identity check); and automatic admission of every certificate that merely chains to the producer
trust anchor. This list generalizes, to certificate-derived identity, the same closed-vocabulary
discipline `classify_operation_producer()` already applies to bearer subject IDs today ("no
wildcard, prefix, or case-insensitive matching").

**Multiple URI SANs.** A producer certificate used by the first BASIS mTLS profile must contain
exactly one URI SAN eligible for producer identity. Zero eligible URI SANs, or more than one
eligible URI SAN, causes producer authentication/admission to fail closed. This is a deliberately
simple, deterministic rule for the first profile: it avoids requiring an implementation to guess
which of several URI SANs represents the producer, which would reintroduce exactly the kind of
heuristic, convention-dependent identity inference this ADR otherwise rejects for Common Name and
network position. No existing BASIS architecture defines a different SAN-selection rule for this
boundary, so no heuristic selection is introduced in its place; a future profile may define
multi-SAN semantics explicitly if a concrete deployment need for one is demonstrated, but that is
not decided here.

**URI canonicalization.** This ADR does not invent a BASIS-specific URI normalization algorithm.
The authenticated URI SAN identity is treated as the certificate-presented identity value and
matched according to the deterministic representation the certificate/TLS parsing layer provides;
admission matching does not case-fold or rewrite the URI's host, path, query, or any other component
as part of admission. Deployments must register the exact producer identity string their issued
certificates present. If a future implementation phase finds that the parsing layer it uses already
defines canonical URI semantics, that layer's canonical form should be followed rather than this ADR
inventing a second, competing one — this ADR does not decide which parsing layer a future
implementation uses.

**Trust anchor is not admission.** Certificate-chain validation and producer admission are
independent checks, and both must succeed:

1. certificate authentication — the presented certificate is cryptographically valid, unexpired, and
   chains to an accepted producer trust anchor, and exactly one eligible URI SAN can be
   deterministically derived from it;
2. explicit producer admission — that derived URI SAN identity exactly matches an enabled entry in
   gateway-controlled admission configuration.

A certificate may therefore be cryptographically valid, issued by an accepted CA, and unexpired, and
still be rejected because its URI SAN identity is not admitted. This restates, at the certificate
layer specifically, the same authentication/admission separation already established generally in
**Authentication vs. admission** above, and it is the reason the **Mistakenly broad trust anchor**
scenario in **Security consequences** below is only partially, not fully, mitigated by admission
alone.

**Fail-closed cases.** Producer authentication/admission fails — and the request is rejected before
any producer-only context is composed — when: certificate validation fails; the certificate is
expired or otherwise invalid; no eligible URI SAN exists on the certificate; more than one eligible
URI SAN exists on the certificate under this first profile; the URI SAN cannot be deterministically
extracted; the derived URI SAN identity is not configured as admitted; the matching admission entry
is disabled or has been revoked; or an ordinary (non-admitted) bearer caller attempts to assert
producer-only context without an admitted producer identity on the connection. This list extends,
without narrowing, the fail-closed cases already stated in **Authentication vs. admission** above.

## Registration and revocation

**Registration.** A producer must not become trusted solely because its certificate chains to a
generally trusted CA. There must be a deployment-controlled admission decision binding an
authenticated workload identity (the URI SAN) to admitted-producer status — the same binding
principle `operation-producer-and-execution-boundary.md` §3 already states for the current
allowlist ("deployment configuration must bind an authenticated workload identity to producer
capabilities... an explicit, deployment-owned configuration artifact, not code-path inference"),
generalized here from a bearer subject ID to a certificate-derived workload identity. At minimum, a
registration record must be capable of identifying: the producer workload identity (URI SAN);
whether it is enabled/admitted; the applicable trust anchor or issuer boundary; and, if current
architecture requires it once a reference implementation exists, operational/environment scope.
This ADR does not select a permanent storage technology for registration — gateway configuration is
sufficient for the first bounded reference implementation; a future producer registry is a later,
undecided possibility. Category-scoped producer capability (trusted for `protocol_context` but not
`safety_context`) remains the open question `operation-producer-and-execution-boundary.md` §3 and
§12 already named; this ADR does not resolve it and does not invent it merely because a new
authentication mechanism is being adopted.

**Revocation.** The architecture must support removal of producer trust without requiring
continuous public connectivity. For the initial architecture: removal or disablement from gateway
admission configuration is always available and always sufficient by itself to deny a previously
admitted workload, independent of certificate state; certificate expiration and rotation provide a
time-bounded natural revocation for the authentication layer; trust-anchor removal (removing a CA
from the configured set of accepted producer trust anchors) revokes every workload whose certificate
chains only through that anchor; and CA- or deployment-provided revocation mechanisms (a local CRL,
for example) may be used when available. This ADR does not require continuous public OCSP access —
doing so would make cloud connectivity a prerequisite for producer authentication, which is
unacceptable for isolated and air-gapped OT deployments. Certificate lifecycle automation (issuance
tooling, automated rotation, renewal alerting) is operationally important and is not fully solved by
this ADR; `ROADMAP.md`'s Phase 4 already tracks "Certificate and credential lifecycle management
tooling" as a **Research direction**, unchanged by this decision.

## Gateway processing model

The expected gateway processing sequence, conceptual only:

```text
1. establish TLS
2. validate producer client certificate (chain, validity, revocation where available)
3. derive authenticated producer workload identity (exactly one eligible URI SAN;
   fail closed on zero or multiple)
4. perform producer admission lookup/configuration (exact-match only)
5. establish gateway-owned producer trust classification
6. parse and structurally validate the operation-aware request
7. reject unauthorized producer-only assertions before kernel evaluation
8. compose canonical operation-aware request
9. invoke basis-core
10. enforce and audit the returned disposition
```

Fail-closed behavior is preserved at every step: a failure at steps 1–5 never reaches step 6. No
HTTP handler and no request-body field may override the connection-established producer identity —
this generalizes the existing rule that `basis-gateway`'s operation-aware schema already enforces by
rejecting `is_trusted_operation_producer` / `producer_trust_classification` as caller-supplied
fields, and it generalizes the existing correlation-ID policy (`CorrelationMiddleware` generates
`X-Correlation-ID` unconditionally per request and explicitly ignores a caller-supplied header,
"because accepting one would allow external parties to influence the audit trail") to the identity
boundary: a request body must never be permitted to assert or override a connection-derived fact.

Steps 6–10 are unchanged by this ADR — they restate, without altering, `basis-gateway`'s existing
composition, kernel-invocation, and enforcement behavior in `operation_aware_composition.py`,
`core/operation_aware_evaluator.py` (per prior architecture), and `GatewayAuditEvent` recording.

## Existing allowlist transition

`OPERATION_PRODUCER_SUBJECT_IDS` and `classify_operation_producer()` classify an **already
bearer-authenticated** subject through an exact, case-sensitive, configured allowlist — a bounded,
honestly-documented, safe-default-empty mechanism. It is retained, not removed, by this ADR:

- it remains available as a compatibility, development, and migration mechanism;
- it must not be described as equivalent to independently authenticated producer workload identity
  — it classifies an authenticated *subject* (a bearer-token claim), not an independently
  authenticated *workload* (a connection-bound, private-key-proven identity); this is precisely the
  distinction §3 of `operation-producer-and-execution-boundary.md` already draws between
  "authenticated workload identity" and "authorization to act as an operation producer";
  the allowlist establishes the second without independently establishing the first;
- no implementation removal is authorized by this ADR alone;
- the eventual compatibility/deprecation policy for the allowlist — whether and when it is
  deprecated once an mTLS-authenticated path exists — belongs to a later `basis-gateway`
  implementation plan, not to this ADR.

mTLS becomes the first normative producer-authentication profile going forward; the allowlist is
not elevated to that status and is not declared obsolete by this decision.

## Bounded reference implementation authorized

This ADR authorizes, but does not itself implement, a first narrowly bounded producer reference
slice, to be built only after this ADR is accepted:

```text
Protocol/platform operation
    → existing basis-adapters normalization (REST adapter)
    → construct adapter evidence material (basis-adapters, ADR-0007 Stage 1, already implemented)
    → canonicalize evidence material (basis-adapters, ADR-0007 Stage 1, already implemented)
    → compute evidence digest (basis-adapters, ADR-0007 Stage 1, already implemented)
    → retain/persist the canonical evidence material (operation-producer runtime;
      mechanism not decided by this ADR)
    → confirm retention succeeded
    → mint opaque reference_id (operation-producer runtime, per ADR-0007)
    → assign adapter_source (operation-producer runtime, per ADR-0007)
    → assign redaction_classification (operation-producer runtime, per ADR-0007)
    → attach required request/correlation linkage
    → assemble AdapterEvidenceReference (operation-producer runtime, per ADR-0007)
    → authenticate producer to basis-gateway using mTLS
    → submit operation-aware request
    → gateway authenticates and admits producer
    → basis-core evaluates
    → gateway returns and audits disposition
    → STOP before protocol execution
```

The slice stops at the authoritative authorization disposition. It does not execute an OT protocol
operation.

**Evidence-retention-before-minting is a required invariant of this lifecycle, not an
implementation preference:**

> The operation-producer runtime must successfully retain the canonical evidence material before
> minting `reference_id` and assembling the final `AdapterEvidenceReference`.

An `AdapterEvidenceReference` asserts, by its own existence, that the evidence it references was
placed into durable retention. A reference minted ahead of, or independent of, successful retention
would make that assertion false at the moment of minting — a defect this ADR closes for the bounded
slice regardless of how a later, permanent evidence-storage architecture is designed.

**Required failure behavior.** If evidence retention fails, no valid `AdapterEvidenceReference` may
be assembled for that evidence: `reference_id` must not be minted as though durable evidence exists
when retention failed, and the operation-producer runtime must fail closed for that submission —
consistent with the fail-closed rule `operation-producer-and-execution-boundary.md` §2 and §5
already state for producer-runtime errors generally ("an operation-producer runtime that cannot
construct a valid request... must deny the operation locally rather than guess"). Gateway
authorization must never be used as a substitute for evidence retention — a producer must not
submit a request bearing a reference to unretained evidence on the theory that a subsequent `allow`
disposition would somehow validate it after the fact; the two facts (retention succeeded; the
operation is authorized) are independent and the second can never repair the first. Retry behavior
for a failed persistence attempt — whether and how the producer runtime retries before failing
closed — remains a later implementation/reliability decision, not resolved here.

This ADR does not define a database, object store, filesystem layout, evidence service, or
permanent storage topology for evidence retention. Evidence persistence remains a distinct
responsibility from reference assembly, restated from the **Persistence scope** discussion below; a
bounded reference implementation may colocate the two responsibilities temporarily, and permanent
topology remains deferred.

**Retrieval semantics — what retention proves, and what it does not.** Successfully retaining
evidence before minting a reference establishes only that the operation-producer runtime placed the
canonical evidence material into its configured retention mechanism before issuing the reference
that points to it. It does not prove: future availability of that material (a later storage failure
or retention-period expiry is a separate, later fact); the authenticity of the producer that
retained it; the correctness of the evidence's content; tamper resistance of the material after it
was persisted; that the operation was authorized; or that any operation was executed. Each of those
remains a separate trust property this ADR does not conflate with retention having occurred.

**Protocol scope.** REST is the recommended first reference adapter. `basis-adapters` already
ships a REST adapter (`src/basis_adapters/rest`); this phase validates the producer/authentication/
evidence-reference boundary rather than native OT execution semantics, and REST minimizes unrelated
protocol complexity while execution stays explicitly out of scope. No repository evidence surfaced
during this ADR's research indicates another existing adapter would provide a materially better
bounded test without broadening scope.

**Persistence scope.** The reference slice must retain the evidence bytes/material before assembling
a reference that claims they exist — see the retain-before-minting invariant and its required
failure behavior above, which govern this bullet's scope. Evidence persistence is a distinct
responsibility from reference assembly; this ADR does not decide the permanent evidence-store
topology. A simple reference implementation may colocate these responsibilities temporarily.
Permanent storage service/repository decisions remain deferred.

**Reference minting.** The bounded slice finally exercises the ADR-0007 ownership rule that has no
runtime implementation today: the producer runtime, only after confirming successful evidence
retention, mints `reference_id` and assembles the final `AdapterEvidenceReference` (per
`adapter-evidence-reference.yaml`'s own `composition.produced_by: operation-producer runtime`
metadata). This ADR does not impose ecosystem-global `reference_id` uniqueness — deployment-local
uniqueness remains the default, per ADR-0007.

**Correlation.** The reference slice must preserve, not repurpose, the distinct identifiers
`operation-producer-and-execution-boundary.md` §6 already names: the gateway's own
`CorrelationMiddleware`-generated correlation ID remains gateway-owned and unconditionally
generated; the kernel `trace_id` and `AuditEvidence.evidence_id` remain `basis-core`-owned; and
`AdapterEvidenceReference.reference_id` remains operation-producer-runtime-owned. The reference
slice's own producer/request identifiers must not silently become, or be confused with, the
gateway's transport correlation ID.

**Retry and idempotency.** The discovery assessment found no existing idempotency/replay contract.
This ADR does not invent a full reliability contract. For the bounded reference slice: retries must
not be hidden; duplicate authorization submissions must not be treated as proof of duplicate
execution, because execution does not exist in this slice; broader idempotency/replay semantics
remain an explicit follow-up architectural question, consistent with the identity-to-operation
interoperability roadmap's Phase 5 and Phase 8, which already name replay protection and idempotency
as open, later decision gates for producer transport generally.

**Provisional implementation location.** The preferred architectural outcome, per
`operation-producer-and-execution-boundary.md` §10–§11, is unchanged: the operation-producer
runtime remains a distinct logical component; it must not be folded into `basis-adapters`,
`basis-core`, `basis-console`, or a future `basis-deploy`; and it should not make `basis-gateway`
itself perform adapter normalization or become the producer. This ADR does not identify a specific
existing repository as sufficiently unmisleading for the bounded reference slice. The reference
slice requires an implementation-location decision immediately before implementation, but that
location is provisional and does not constitute permanent ecosystem repository ownership. This ADR
does not create or name a permanent `basis-producer` repository. A new repository remains a later
decision gate informed by the bounded implementation, per `operation-producer-and-execution-boundary.md`
§11's own deferred decision gate.

**What the slice must prove, stated once, unambiguously:**

> The bounded reference implementation must prove the producer lifecycle through authorization
> only: normalize one REST operation, construct deterministic adapter evidence, successfully retain
> the canonical evidence material, mint and assemble its `AdapterEvidenceReference`, authenticate
> the producer to `basis-gateway` using the ADR-0008 mTLS profile, receive gateway admission, obtain
> the authoritative `basis-core` disposition, and record gateway audit evidence. The slice ends
> there and performs no OT execution.

## Security consequences

For each scenario below: what mTLS mitigates, what gateway admission mitigates, what remains
residual risk, and what belongs to later architecture.

**Stolen producer private key.** mTLS mitigates: an attacker without the key cannot complete the
handshake. Admission mitigates: revoking the workload's admission entry immediately denies further
requests even if the key remains valid. Residual risk: requests made before revocation are
indistinguishable from genuine producer traffic; detection depends on anomaly monitoring outside
this ADR's scope. Belongs later: key-compromise detection and automated response.

**Compromised producer runtime.** mTLS mitigates nothing here — a compromised runtime with a valid
key authenticates normally. Admission mitigates: category-scoped capability, if it existed, would
bound the blast radius; today's all-or-nothing model does not, and this ADR does not change that.
Residual risk: a compromised, admitted producer can assert any producer-only context it is admitted
for, exactly as a compromised allowlisted bearer subject can today. Belongs later: category-scoped
producer trust; anomaly detection on producer-asserted context.

**Compromised producer CA.** mTLS mitigates nothing — a compromised CA can mint certificates
identical in shape to legitimate ones. Admission mitigates partially: admission still requires a
workload's specific URI SAN to be separately registered, so a compromised CA alone cannot admit an
unregistered identity, but it can mint a certificate matching an already-admitted identity's SAN if
the attacker knows it. Residual risk: CA compromise is a severe, ecosystem-external event. Belongs
later: CA operational security, out of BASIS's architectural scope.

**Mistakenly broad trust anchor.** mTLS mitigates nothing — trusting an over-broad CA (for example,
a public WebPKI root) would let any holder of a certificate from that CA complete the handshake.
Gateway admission mitigates the worst case: even with a broad trust anchor, the workload identity
still must be separately admitted — certificate-chain validation and producer admission remain
independent checks, per **Trust anchor is not admission** above, so a broad trust anchor alone does
not admit anything by itself. Residual risk: an operator who trusts too broad a CA has degraded
the strength of the authentication step even if admission still gates who is trusted; this is an
operator configuration error this ADR documents against rather than technically prevents. Belongs
later: none — this is the reason **Trust Anchors** below states public WebPKI trust stores must not
automatically validate producer identities.

**Ambiguous certificate identity.** Risk: a certificate carrying multiple SANs, an implementation
that falls back to Common Name, or an admission check that performs wildcard, prefix, substring, or
other heuristic matching could admit a workload the deployment did not intend to admit, or admit the
wrong identity when a certificate is reused or misissued. Mitigation: this ADR requires exactly one
deterministic identity source (a single eligible URI SAN), exact-match admission with no CN
fallback and no wildcard/prefix/substring/regex inference (**Certificate identity profile** above),
and fails closed on zero or multiple eligible URI SANs rather than guessing. Residual risk: a
non-conforming implementation that adds heuristic matching despite this ADR's requirement would
reintroduce the risk; this ADR documents the requirement but cannot itself prevent a
non-conforming implementation, the same limit already noted for **Producer/subject conflation**
above.

**Certificate expiration.** mTLS mitigates by design — an expired certificate fails handshake
validation, denying the connection. Residual risk: an unrenewed certificate causes a legitimate
producer outage rather than a security failure — an availability, not confidentiality/integrity,
concern. Belongs later: renewal automation and operator alerting.

**Stale producer admission.** Neither mTLS nor certificate validity addresses this — a valid
certificate for a workload whose admission should have been revoked, but wasn't, remains trusted.
Gateway admission mitigates only to the extent the deployment actively maintains its admission
configuration. Residual risk: stale admission entries are an operational hygiene problem this ADR
does not solve. Belongs later: admission lifecycle tooling, expiry policies for registration
entries.

**Failure to revoke a producer.** Same as stale admission above — the mitigation is operational
discipline in maintaining gateway admission configuration or a future registry, not a property this
ADR's cryptographic mechanism can enforce by itself.

**Body-field spoofing.** Gateway admission (specifically, the processing-model rule that no
HTTP handler or request-body field may override the connection-established producer identity)
fully mitigates this for producer identity itself, consistent with the existing
`is_trusted_operation_producer` / `producer_trust_classification` rejection rule. Residual risk:
none new — this restates an existing, already-enforced invariant.

**Producer/subject conflation.** Fully mitigated architecturally by this ADR's explicit **Producer
vs. authorization subject** section: the producer certificate identity is never reused as the
authorization subject automatically. Residual risk: an implementation that violates this rule
(mapping producer identity directly into a subject claim) would reintroduce the conflation; this ADR
does not prevent a non-conforming implementation, only documents the requirement.

**Replayed HTTP requests.** mTLS's session-bound nature mitigates simple cross-connection replay of
the TLS handshake itself. It does not mitigate replay of a captured, valid HTTP request within an
authenticated session or across a new session established with the same certificate. Residual risk:
full replay protection is an explicitly deferred, ecosystem-wide question (identity-to-operation
roadmap Phase 5 and Phase 8). Belongs later: replay-identifier and idempotency architecture.

**Confused-deputy behavior.** Gateway admission mitigates by keeping producer admission and
authorization-subject evaluation as separate facts — a producer cannot use its own admitted status
to obtain authority on behalf of a subject it does not represent, since `basis-core` evaluates the
authorization subject, not the producer, per existing architecture. Residual risk: a topology
combining producer and executor in one process (per `operation-producer-and-execution-boundary.md`
§9) concentrates risk the boundary document already names ("a compromised or confused executor that
accepts commands without checking their disposition binding reintroduces exactly the 'confused
deputy' abuse case"); this ADR does not add new mitigation for that topology beyond what the
boundary document already states.

**Evidence-reference spoofing.** Producer admission mitigates who may submit an
`adapter_evidence_reference` at all. It does not mitigate a genuinely admitted, but compromised or
dishonest, producer submitting a reference to fabricated or mismatched evidence — the gateway "does
not regenerate the evidence or digest" (ADR-0007) and cannot independently verify the reference
points to genuine material. Residual risk: this is the same limit ADR-0007 already states plainly:
"hashing does not make a compromised producer trustworthy." Belongs later: evidence-storage
architecture with independent verification.

**Dangling evidence references.** Risk: a producer issues an `AdapterEvidenceReference` for evidence
that was never actually retained — a `reference_id` pointing at nothing, discoverable only later,
if at all, when something attempts to dereference it. Mitigation: the retain-before-minting
invariant in **Bounded reference implementation authorized** above requires the operation-producer
runtime to confirm successful retention before it may mint `reference_id` or assemble the reference,
and requires the producer to fail closed on retention failure rather than mint a reference anyway.
Residual risk: later loss, corruption, or unauthorized alteration of evidence material that was
genuinely retained at minting time remains a storage-integrity concern this ADR does not solve —
retention-before-minting proves the reference was honest at the moment it was created, not that the
material remains available or unaltered indefinitely afterward. Belongs later: evidence-storage
architecture with retention guarantees and independent verification, the same future work named for
evidence-reference spoofing above.

**Compromised evidence storage.** Outside mTLS's and admission's scope entirely — evidence storage
does not exist as a defined component yet. Belongs later: future evidence-storage/evidence-service
architecture, explicitly deferred by both ADR-0007 and this ADR.

**Gateway compromise.** mTLS and admission both depend on the gateway's own integrity; a compromised
gateway can misclassify or fabricate admission decisions regardless of the authentication mechanism
used. Residual risk: this is a general infrastructure-security concern outside this ADR's scope, no
different in kind from the existing risk that a compromised gateway could fabricate bearer-token
verification results today.

**Air-gapped certificate lifecycle.** mTLS with a local/private CA and locally configured trust
anchors mitigates the connectivity dependency — no external service is required to validate a
certificate chain against a trust anchor the deployment already holds. Residual risk: certificate
issuance, rotation, and any revocation-list distribution must all be planned as offline or
store-and-forward processes; this ADR does not design that tooling. Belongs later: `ROADMAP.md`
Phase 4's certificate/credential lifecycle management tooling, still a research direction.

The governing security statement, restated from ADR-0007 and unchanged here: authenticating the
producer does not prove the truthfulness of adapter evidence, does not prove authorization, and does
not prove execution integrity by itself.

## Operational consequences

BASIS targets OT environments with long-lived, intermittently maintained deployments, constrained
maintenance windows, and frequently air-gapped or low-connectivity networks. mTLS redistributes
operational complexity rather than eliminating it, consistent with the principle this ADR inherits
from the rest of BASIS's operation-aware architecture:

- **Certificate rotation burden.** Every admitted producer now depends on certificate lifecycle
  events (issuance, renewal, rotation) that a bearer-token allowlist entry does not require to the
  same degree. This is a real, non-trivial operational cost, not minimized by this ADR.
- **Long-lived and intermittently maintained deployments.** A deployment that goes unmaintained for
  an extended period risks certificate expiration silently disabling a legitimate producer. Operator
  guidance for renewal timelines belongs to future `basis-deploy` and operational documentation, not
  this ADR.
- **Maintenance windows.** Certificate rotation should be planned within existing OT maintenance
  windows rather than assumed to be a zero-downtime, anytime operation.
- **Offline/air-gapped operation.** A local/private CA and deployment-configured trust anchors are
  fully compatible with air-gapped operation, since validation requires only locally held trust
  material, not external connectivity. This is a structural advantage of mTLS over mechanisms that
  assume reachability to an external issuer or introspection endpoint at request time.
- **Local/private PKI.** Deployments are expected to operate their own CA or use an organizationally
  provided one; this ADR does not design or provide a CA service.
- **Clock correctness and certificate validity.** Certificate validity checking depends on
  reasonably correct system clocks at the gateway; clock drift in constrained OT environments is a
  known operational hazard this ADR does not solve.
- **Gateway certificate renewal.** The gateway's own server-side TLS certificate is a separate
  concern from producer client certificates (see **Trust anchors** below); its renewal follows
  whatever process the deployment already uses for gateway TLS.
- **Producer credential renewal.** Distinct from gateway renewal; each admitted producer workload
  independently manages its own certificate lifecycle.
- **Degraded behavior when credentials expire.** An expired producer certificate fails closed — the
  producer cannot authenticate, and therefore cannot submit operation-aware requests, until its
  credential is renewed and, if the identity changed, re-admitted. This is a safe failure mode
  (unavailability, not a security failure) but is an availability cost operators must plan for.
- **Operator recovery.** Recovering a producer whose certificate has expired or been revoked in
  error requires an operator action (reissuing a certificate, updating admission configuration); this
  ADR does not define that recovery workflow.
- **High availability implications.** A gateway HA topology must consistently apply the same trust
  anchors and admission configuration across all instances; this ADR does not design gateway HA and
  assumes existing gateway configuration-distribution mechanisms apply unchanged.
- **Exact-match identity naming discipline.** Because admission matching is exact, not fuzzy (per
  **Certificate identity profile** above), deployments must manage producer identity naming
  carefully: choosing and recording the exact URI SAN string each producer's certificate will carry;
  keeping admission configuration synchronized with that exact string; ensuring certificate renewal
  reissues the same URI SAN rather than silently changing it (which would otherwise deny a producer
  that operators still consider the same workload); planning admission-configuration updates
  alongside any deliberate identity change during rotation; and revoking or disabling admission
  entries for retired identities rather than leaving them registered indefinitely. This is a naming
  and configuration-hygiene cost, not additional lifecycle automation this ADR designs.

Stronger identity infrastructure does not eliminate operational burden; it relocates it from
credential-string management to certificate lifecycle management. This ADR adopts that trade
deliberately, for the security properties described above, while explicitly not minimizing its cost.

## Alternatives considered

**Existing authenticated-subject allowlist (`OPERATION_PRODUCER_SUBJECT_IDS`).** Simple, already
implemented, and valuable for compatibility, local development, and migration. It conflates
subject and producer identity by construction (the module's own docstring calls the two "equal in
this PR's only supported trust mechanism... but that equality is an implementation detail of the
current transport, not a statement that the two concepts are the same fact") and depends entirely on
bearer-token limitations — theft or replay of the bearer credential grants producer trust with no
independent workload proof. **Disposition: retained as a compatibility/development mechanism;
rejected as the long-term normative producer workload-authentication model.**

**OAuth 2.0 client credentials.** A familiar, well-understood machine-to-machine pattern with
mature issuer integration, structured token claims, and established credential-rotation practices.
Its exposure surface is bearer-token theft/replay, same in kind as the existing allowlist's
weakness, though mitigated by shorter token lifetimes and issuer-side revocation where the issuer is
reachable. It requires either a reachable token issuer at request time or accepting offline
validation trade-offs, and today would require `basis-identity` to first become a client-credentials
issuer, which it does not do yet. Token-bound or proof-of-possession extensions (DPoP, mTLS-bound
tokens) would materially change this analysis by removing the pure-bearer weakness, but that is
itself closer to combining OAuth with the transport-bound identity this ADR already selects, not a
reason to prefer plain client credentials today. OAuth client credentials is not insecure in
general — it is not selected as the first BASIS profile because it either assumes issuer reachability
this ADR wants to avoid as a hard prerequisite for air-gapped deployments, or requires
`basis-identity` capability that does not exist yet, while mTLS needs neither.
**Disposition: viable future producer-authentication profile; not selected as the first normative
BASIS profile.**

**SPIFFE / SPIRE.** The SPIFFE identity model (URI SAN identity, standardized workload identity
documents) is deliberately preserved as forward-compatible by this ADR's certificate-identity
profile. Requiring a full SPIRE deployment — automated short-lived credential issuance, workload
attestation, a control-plane service — is a substantial additional infrastructure dependency,
disproportionate for small BASIS deployments and isolated OT networks, and unnecessary to validate
the first bounded reference implementation. **Disposition: preserve compatibility with SPIFFE-style
identities; do not require SPIRE for the first reference implementation; treat SPIFFE/SPIRE
integration as a future workload-identity profile or deployment option.**

**Static API keys or shared secrets.** Weak workload identity (a shared secret proves possession of
a string, not possession of a private key bound to one workload); distribution and rotation both
carry meaningful operational and security risk; a copied or leaked key is indistinguishable from a
legitimate one; attribution of a specific request to a specific workload is weak or absent.
**Disposition: rejected as the normative producer-authentication model.**

**Request-body producer assertions.** A request must never make itself trusted by declaring producer
identity, producer trust classification, or trusted-producer status — this must remain a
gateway-derived fact, exactly as `basis-gateway`'s existing rejection of caller-supplied
`is_trusted_operation_producer` / `producer_trust_classification` already enforces.
**Disposition: explicitly rejected.**

## Certificate identity profile / Trust anchors

Producer-client certificate trust must be explicitly configured by the deployment. Public WebPKI
trust stores must not automatically make arbitrary public certificates valid producer identities —
trusting a generally trusted public root for producer-client authentication would defeat the purpose
of an explicit producer trust boundary, as the **Mistakenly broad trust anchor** security scenario
above states. Deployments may use a private or local CA; air-gapped deployments may operate entirely
with local trust anchors requiring no external connectivity. Gateway server authentication (the
certificate `basis-gateway` presents to callers) and producer client authentication (the certificate
a producer presents to the gateway) are separate certificate roles, even where a deployment chooses
to issue both from a common PKI hierarchy — a deployment's choice to share a CA between the two
roles does not make them architecturally the same role. This ADR does not design a CA service.

## Machine-readable contract decision

This ADR does not assume it requires a new JSON Schema. For the mTLS profile it selects, no new
producer identity field should be added to any request body merely to duplicate transport-
authenticated identity — the producer identity is a fact about the connection, not a fact the
request body should restate or could safely be trusted to restate. After implementation planning
proceeds from this ADR, the stable machine-readable boundary — if any is needed at all — may turn
out to be: TLS/certificate profile documentation, gateway configuration, a future producer-
registration contract, a claims profile, a shared `basis-schemas` contract, or no new request-payload
contract whatsoever. A new JSON Schema is published only if a stable cross-repository data contract
is actually identified through the bounded reference implementation this ADR authorizes — consistent
with `ecosystem-contract-inventory.md`'s governing principle that contracts become schema-ready when
"implementation proves a stable shape," not before.

## Relationship to `basis-identity`

This ADR does not require `basis-identity` to become a producer credential issuer before the first
producer reference implementation can proceed. `basis-identity` today has partial service-identity
vocabulary (`SubjectType.SERVICE`) but no workload-identity establishment pipeline and no
`identity-evidence-reference` production, as confirmed by direct source inspection above. The mTLS
producer profile is therefore independently deployable using operator-managed or local PKI, without
waiting on `basis-identity` capability that does not exist today. Future `basis-identity` work may
participate in workload identity, credential lifecycle, registration, identity evidence, or SPIFFE/
OAuth profiles; this ADR does not assign those responsibilities to `basis-identity` permanently, and
does not schedule them.

## Relationship to adapter evidence

This ADR preserves ADR-0007 exactly. It does not change `basis-adapters` evidence-material
ownership, RFC 8785 canonicalization, SHA-256 (or any other) digest behavior, operation-producer
ownership of final reference assembly, `basis-gateway` ownership of producer admission and
provenance enforcement, or `basis-core`'s reference-only boundary. Authentication of the producer
does not prove the truthfulness of adapter evidence — a distinct fact ADR-0007 already establishes
and this ADR does not touch.

## Compatibility

This ADR is additive and forward-looking with respect to every currently released contract and
runtime behavior. It does not modify `basis-gateway`'s current `AuthMode` enum, `GatewayConfig`,
`classify_operation_producer()`, or `OPERATION_PRODUCER_SUBJECT_IDS` behavior — all of that code is
unchanged by this ADR and continues to function exactly as released until a future `basis-gateway`
implementation plan acts on this decision. It does not modify any `basis-schemas` contract, `basis-core`
model, or `basis-adapters` behavior. It does not change any published API route. A deployment that
takes no action after this ADR is accepted observes no behavior change whatsoever.

## Deferred decisions

The following are deliberately not decided by this ADR, so that later work does not interpret
silence as authorization:

- Permanent producer repository placement; creation of a new repository.
- Production evidence-store topology.
- Category-scoped producer capabilities, beyond what this ADR required to express the first mTLS
  profile (none were required; the all-or-nothing model is unchanged).
- Full credential lifecycle automation.
- SPIRE deployment.
- OAuth client-credentials implementation.
- Producer behavior analytics.
- Execution-plane architecture; protocol executor design; execution-result vocabulary;
  execution-result evidence.
- Console lifecycle workflows.
- `basis-deploy` packaging.
- Ecosystem-wide retry/idempotency semantics.
- A global `reference_id` namespace.
- Commercial/BASAuth product behavior.
- Whether or when `OPERATION_PRODUCER_SUBJECT_IDS` is deprecated.
- The specific gateway configuration surface (environment variables, file format, or future registry
  schema) that will express trust anchors and admission entries.
- Whether admission entries require an environment/operational scope field.

## Consequences

**Positive:**

- BASIS gains a precise, reviewable answer to the producer workload-authentication and admission
  problem, unblocking the bounded reference implementation gate named in
  `operation-producer-and-execution-boundary.md` §13 Stage 5.
- The distinction between authentication, admission, producer identity, and authorization subject —
  previously implicit in `basis-gateway`'s bearer-token-plus-allowlist implementation — becomes an
  explicit, reviewable architectural commitment that future implementations must preserve.
- A path exists for air-gapped and isolated OT deployments to authenticate producers without any
  dependency on external connectivity or a not-yet-built `basis-identity` capability.
- Forward compatibility with SPIFFE-style identity is preserved without committing to SPIRE
  infrastructure.

**Tradeoffs:**

- Certificate lifecycle management becomes a required operational capability for any deployment that
  adopts the mTLS profile, with real rotation, renewal, and revocation burden this ADR does not
  minimize.
- `OPERATION_PRODUCER_SUBJECT_IDS` now has two futures to reconcile (compatibility mechanism vs.
  normative model), which a future `basis-gateway` implementation plan must resolve explicitly rather
  than by default.
- The bounded reference implementation this ADR authorizes still requires a provisional
  implementation-location decision before any code is written, which this ADR does not make.

## Validation / implementation gate

Formal acceptance of this ADR, through this repository's ADR governance process
([`docs/adr/README.md`](README.md#lifecycle-states)), is the gate before:

- `basis-gateway` begins any implementation of mTLS validation, certificate-derived producer
  identity, or admission configuration;
- an operation-producer runtime reference implementation begins, per the bounded slice authorized
  above;
- any `basis-schemas` proposal for a producer-authentication-related contract is considered, per the
  machine-readable contract decision above.

This ADR, once merged, does not itself constitute acceptance — consistent with this repository's
established convention (see [ADR-0006](0006-evaluation-orchestration-layer.md)'s implementation
note and [`docs/adr/README.md`](README.md#lifecycle-states)) that merging an ADR does not by itself
change its status to `Accepted`.

## References

- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) — §2 (logical roles), §3 (trust establishment, the six distinct facts this ADR builds on), §10–§11 (repository ownership and the provisional-location decision gate), §13 Stage 4 (the decision gate this ADR resolves) and Stage 5 (the bounded reference implementation this ADR authorizes)
- [ADR-0007](0007-adapter-evidence-construction.md) — adapter evidence construction ownership, preserved unchanged by this ADR; its own deferred "how it authenticates to basis-gateway" question, resolved here
- [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](../roadmaps/identity-to-operation-contract-and-interoperability.md) — Phase 5 (Producer Authentication, Integrity, and Transport Profiles), whose requirements this ADR begins to satisfy for the specific producer-to-gateway boundary, without claiming to complete Phase 5's broader transport-integrity, replay, and idempotency scope
- [`docs/security/threat-model.md`](../security/threat-model.md) §3.3, §4.2, §6.3 — trusted-adapter boundary and rogue-integrator/compromised-adapter adversaries, extended by this ADR to the producer-authentication boundary
- [`docs/architecture/reference-vs-implementation.md`](../architecture/reference-vs-implementation.md) — the conceptual-architecture/reference-architecture/implementation distinction this ADR observes in declining to prescribe a specific gateway API shape
- [`docs/architecture/compatibility-philosophy.md`](../architecture/compatibility-philosophy.md) — the additive-change discipline this ADR's Compatibility section applies
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md) — "implementation proves a stable shape," the principle this ADR's machine-readable contract decision applies
- [`docs/glossary.md`](../glossary.md) — Zero Trust, applied to this ADR's rejection of network position, role membership, and Common Name as identity substitutes
- `basis-gateway`: `src/basis_gateway/auth/operation_producer.py` (`classify_operation_producer`, `OperationProducerTrust`), `src/basis_gateway/config.py` (`AuthMode`, `OPERATION_PRODUCER_SUBJECT_IDS`), `src/basis_gateway/core/operation_aware_composition.py` (`UntrustedOperationProducerContextError`), `src/basis_gateway/middleware/correlation.py` (`CorrelationMiddleware`) — current implementation this ADR's Context section verifies directly
- `basis-identity`: `src/basis_identity/models/identity_context.py` (`SubjectType`, `AuthenticationProtocol`) — current workload-identity gap this ADR's Context section verifies directly
- `basis-schemas`: `schemas/adapter-evidence-reference/adapter-evidence-reference.yaml` — the reference contract's `composition` ownership metadata this ADR's bounded reference implementation exercises without modifying
- `basis-adapters`: `src/basis_adapters/rest` — the recommended first reference adapter for the bounded implementation
