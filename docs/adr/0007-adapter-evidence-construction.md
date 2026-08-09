# ADR-0007: Adapter Evidence Construction

## Status

Accepted

## Context

`basis-adapters` normalizes protocol-specific operations into
`NormalizedAuthorizationRequest` and preserves the originating protocol operation, verbatim, as
`protocol_evidence`. `basis-schemas` v0.2.0 published the safe-reference shape a future producer
would need to carry that evidence into an operation-aware request —
`adapter-evidence-reference` — and `basis-core` v0.2.0 already models that reference,
byte-identically, as its `AdapterEvidenceReference` domain type. `basis-gateway` v0.2.0 already
accepts, admits, and classifies-as-`trusted_producer_asserted` an `adapter_evidence_reference`
field on its operation-aware request, from a caller it has classified as a trusted operation
producer.

What none of these components define is the space between normalization and the reference: what
semantic material a digest should cover, how that material is serialized so two independent
implementations agree byte-for-byte, which component computes the digest, which component mints
`reference_id`, and which component assembles the final reference object. `basis-architecture`'s
`docs/architecture/operation-producer-and-execution-boundary.md` named this gap and, in its §13
Stage 2, required a `basis-adapters`-owned planning document to assess whether the library's
current public surface is sufficient before any architecture decision is made. That planning
document — `basis-adapters`' merged `docs/operation-aware-handoff-alignment-plan.md` — completed
the assessment: the bulk of the handoff (action, resource type, local resource identifier,
protocol, protocol evidence) is already sufficient, unchanged; the four required
`adapter-evidence-reference` fields (`reference_id`, `evidence_digest`, `adapter_source`,
`redaction_classification`) are not currently produced anywhere in the ecosystem; and three of the
four cannot be constructed responsibly without an architecture decision this repository, not
`basis-adapters`, is authorized to make.

This ADR is that decision.

## Decision

Adopt [`docs/architecture/adapter-evidence-construction-semantics.md`](../architecture/adapter-evidence-construction-semantics.md)
as the detailed specification for adapter evidence construction and canonicalization. In summary:

- **Adapter evidence material** is a specific, named, versioned profile —
  `evidence_profile: basis-adapter-evidence-v1` — carrying selected normalized facts (`protocol`,
  `action`, `resource_type`, `resource_id`) together with a **governed projection** of
  `protocol_evidence` (protocol, method, path, and only the documented, approved metadata keys for
  the applicable protocol — never the full, unrestricted `metadata` object), plus the profile's own
  `evidence_profile` and `canonicalization_profile` identity, and, when tracked,
  `normalization_version`/`mapping_version`. `NormalizedAuthorizationRequest.protocol_evidence`
  itself is unchanged by this ADR and remains complete, per `basis-adapters`' existing normalization
  contract; only the narrower, digested projection is governed here. `subject_hint` is excluded
  from the digested material; unknown metadata keys and any credential-, token-, or secret-shaped
  value are rejected at construction, never silently discarded and never excused by a
  `redaction_classification` assignment after the fact.
- **Canonicalization is `evidence_profile: basis-adapter-evidence-v1` / `canonicalization_profile: rfc8785`,
  adopted exactly, now.** RFC 8785 (the JSON Canonicalization Scheme, JCS) is not a candidate
  pending a future technology evaluation — it is the fixed canonicalization profile for the first
  evidence profile, decided by this ADR and implemented by `basis-adapters`. No operation-producer
  runtime, deployment, or future adapter selects or substitutes a different canonicalization
  mechanism for evidence carrying this profile. Both `evidence_profile` and
  `canonicalization_profile` are carried as fields inside the evidence material itself and are
  included in the canonical bytes and the digest.
- **`basis-adapters` constructs the governed evidence material, canonicalizes it with RFC 8785, and
  computes its digest**, through a pure, deterministic, side-effect-free helper consistent with its
  existing normalization contract. The **operation-producer runtime** mints `reference_id`,
  assembles the final `AdapterEvidenceReference`, supplies `adapter_source`, assigns
  `redaction_classification` under deployment policy, and attaches request/correlation linkage.
  `basis-gateway` authenticates and classifies the producer and validates and admits the reference;
  it does not regenerate the evidence or the digest.
- **`reference_id`** is an opaque identifier for one adapter-evidence artifact, minted by the
  operation-producer runtime — never by `basis-adapters` — with deployment-local rather than
  ecosystem-global uniqueness assumed by default.
- **A deployment selects a required or optional evidence mode before the operation is submitted.**
  Under `required`, any evidence-construction failure blocks operation submission. Under `optional`,
  evidence may be omitted only when deployment policy explicitly permits omission and the omission
  reason is recorded auditably. A construction failure must never silently downgrade `required`
  evidence into an unrecorded `optional` omission. Neither condition fabricates a kernel `DENY`; a
  gateway rejection of a malformed reference remains a request-admission failure, and a later digest
  mismatch remains an evidence-integrity failure — none of these retroactively rewrites a prior
  authorization decision.
- **No `basis-schemas` change is made by this ADR.** The evidence-material profile identifier and
  canonicalization-profile identity are resolved by carrying them as fields inside the evidence
  material itself, not by adding a `basis-schemas` field. A distinct, narrower possibility — exposing
  either identifier directly on the published `AdapterEvidenceReference` for consumer convenience —
  remains deferred unless a concrete consumer requires it.

## Ownership

```text
basis-adapters
    constructs the governed evidence material
    canonicalizes it with RFC 8785
    computes the digest

operation-producer runtime
    mints reference_id
    assembles AdapterEvidenceReference
    supplies adapter_source
    assigns redaction classification
    attaches request linkage

basis-gateway
    authenticates and classifies the producer
    validates and admits the reference
    does not regenerate the evidence or digest

future evidence storage or evidence service
    persists evidence material, if required
    supports later retrieval and digest verification
```

A stable **adapter implementation identity** (which adapter code performed normalization, carried
inside evidence material) is distinct from **producer workload identity** (the authenticated
identity of the operation-producer runtime process, established entirely by future producer
authentication — `basis-adapters` never contributes to it) and from **deployment-instance identity**
(a deployment-chosen label for one configured instance, such as `AdapterContext.adapter_id`). The
operation-producer runtime may combine adapter implementation identity with a deployment-instance
label into the single opaque `adapter_source` value the published schema expects; that combination
is reference-assembly work, not evidence-material construction. See the architecture document §9.

Computing a digest in `basis-adapters` does not make the adapter a trust authority. Producer
authentication, transport integrity, and future evidence verification remain separate,
independently necessary controls.

## Canonicalization requirements

`basis-adapter-evidence-v1` evidence material is canonicalized under **RFC 8785 (the JSON
Canonicalization Scheme, JCS), exactly** — decided now, not deferred to a future technology
evaluation or left open to producer-runtime choice. A defined, exhaustive property list (see the
architecture document §5) restates RFC 8785's own requirements for evidence material specifically:
UTF-8 encoding with no byte-order mark; object-member-name ordering by RFC 8785's UTF-16 code-unit
comparison; unmodified array order; no insignificant whitespace; RFC 8785's own string escaping;
I-JSON (RFC 7493) input conformance (no duplicate names, no invalid Unicode, every number
representable as an IEEE 754 double); numeric serialization under the RFC 8785 / IEEE 754
double-precision model, with any value requiring greater precision carried as a governed string
field instead of a JSON number; and explicit construction failure — not silent coercion — for
`NaN`, `±Infinity`, duplicate object names, invalid Unicode, and unsupported value types.
Unicode strings are preserved as received; no Normalization Form C or other normalization is
applied, consistent with RFC 8785 itself. Canonicalization technology selection is closed for this
profile pairing: it is an architecture-owned decision implemented by `basis-adapters`, not a
producer-runtime or deployment choice.

## Digest requirements

The digest is computed over the canonical bytes of the evidence material, using the already-open,
already-published `evidence_digest_shape` (an open algorithm label plus a lowercase-hexadecimal
value). `sha-256` is recommended as an initial algorithm without closing the vocabulary. The
evidence material's own `evidence_profile` (`basis-adapter-evidence-v1`) and
`canonicalization_profile` (`rfc8785`) fields are included in the canonical bytes and therefore in
the digest, so a verifier who later retrieves the artifact always knows which rules to recompute
against without consulting anything outside the artifact itself. Digest equality proves
byte-correspondence with declared canonical input under the profile the artifact itself declares; it
does not prove truthfulness, producer authenticity, authorization, or execution.

## Reference-ID semantics

`reference_id` identifies exactly one adapter-evidence artifact. It is not the digest, not a
gateway correlation ID, not a kernel trace ID, and not a protocol transaction ID. It is minted by
the **operation-producer runtime**, never by `basis-adapters`. Uniqueness is deployment-local by
default (a **future producer-runtime decision** could later impose an ecosystem-global guarantee,
but none exists today). Identical evidence material may legitimately carry more than one
`reference_id` across independent submissions or retries; one `reference_id` may point to exactly
one evidence artifact.

## Failure behavior

A deployment operates in one of two evidence modes, selected before the operation is submitted:

```text
required
    any evidence-construction failure blocks operation submission

optional
    evidence may be omitted only when deployment policy explicitly permits
    omission and the omission reason is recorded auditably
```

Evidence-material construction failure, canonicalization failure, digest-generation failure,
reference-ID generation failure, and invalid reference assembly are all **required
evidence-construction failures** under the `required` mode: the operation-producer runtime must not
submit the operation for authorization. A construction failure must never silently downgrade
`required` evidence into an unrecorded `optional` omission. This is distinct from evidence being
explicitly and auditably omitted under a deployment operating in `optional` mode, distinct from a
gateway's ordinary request-admission rejection of a malformed reference, and distinct from a later
evidence-integrity (digest-verification) failure — none of these fabricate or retroactively rewrite
a kernel `DENY`.

## Consequences

- `basis-adapters` gains a decision-gated, additive implementation path (Stage 1 in the
  architecture document's §17) for a pure evidence-material/canonicalization/digest helper. No
  existing public field, model, or behavior changes.
- The operation-producer runtime's future architecture (not yet drafted, and not a new repository
  decision made by this ADR) inherits a concrete assembly responsibility rather than an open one.
- `basis-gateway`'s existing admission behavior for `adapter_evidence_reference` is confirmed
  correct and unchanged: it authenticates, classifies, and admits; it does not regenerate.
- No `basis-schemas` contract changes. Two potential future schema additions are recorded as
  deferred, not decided.
- Future evidence-storage and evidence-verification architecture remains an open, later decision;
  this ADR establishes that digest semantics remain coherent whenever that architecture is
  eventually defined, without designing it now.

## Compatibility impact

Purely additive and forward-looking. No existing `basis-schemas` contract (`adapter-evidence-
reference`, `redaction-classification`, `contract-metadata`, `operation-aware-decision-request`),
no `basis-core` model, and no `basis-gateway` runtime behavior changes as a result of this ADR.
Evidence constructed under a future implementation of this ADR remains re-verifiable against the
`evidence_profile`/`canonicalization_profile` and digest algorithm it declares — but only for as
long as the evidence artifact and its declared profile fields remain retrievable and a conforming
RFC 8785 implementation remains available to the verifier. This ADR does not claim indefinite
verifiability independent of those conditions; whether a future evidence-storage architecture
guarantees artifact availability is not decided here. See the architecture document's compatibility
invariant (§2) and its profile-identity/verification-flow discussion (§6).

## Security considerations

This ADR extends, without altering, `docs/security/threat-model.md` §3.3 (the trusted adapter
boundary), §4.2 (rogue integrator, compromised adapter), and §6.3 (adapter-specific threats). The
architecture document's §13 records a threat-by-threat analysis (evidence-material tampering,
normalized-output substitution, canonicalization ambiguity, digest algorithm downgrade, digest
substitution, `reference_id` collision, replay, producer-runtime compromise, misattached evidence,
secret leakage, unbounded metadata, evidence deletion, and false execution-proof claims), each with
its trust boundary, mitigation, residual risk, and owning component. The governing security
statement is unchanged from the rest of this ecosystem's operation-aware architecture: hashing does
not make a compromised producer trustworthy, and a digest is not a substitute for producer
authentication, transport integrity, or the kernel's own authorization decision.

## Alternatives considered

1. Producer runtime owns all evidence construction (material and digest). Rejected — risks
   normalization drift between what `basis-adapters` observed and what the producer runtime
   digests.
2. `basis-adapters` owns the entire `AdapterEvidenceReference` (including `reference_id`,
   `adapter_source`, `redaction_classification`). Rejected — requires the library to hold
   deployment configuration and redaction policy it structurally must not acquire.
3. Adapter library owns semantic material and digest; producer owns reference assembly. **Adopted.**
4. Gateway regenerates adapter evidence. Rejected — the gateway has no protocol-parsing capability
   and cannot independently verify normalization.
5. Digest only raw protocol evidence. Rejected — does not prove enough about the normalization
   result.
6. Digest only normalized output. Rejected — loses protocol-native traceability.
7. Digest a combined versioned evidence profile. **Adopted.**
8. Use `reference_id` as the digest. Rejected — conflates a producer-minted identifier with a
   deterministic function of evidence content.
9. Publish a `basis-schemas` contract before a reference implementation exists. Rejected — violates
   the ecosystem's established "implementation proves a stable shape" publication discipline.
10. Omit adapter evidence entirely. Rejected — removes the only mechanism linking a decision back
    to the protocol-level facts that produced it.

## Deferred decisions

- **Decided now:** the `basis-adapter-evidence-v1` evidence-material profile's field composition,
  its governed per-protocol metadata projection, and its exclusions; the `rfc8785` canonicalization
  profile, adopted exactly; digest semantics; `reference_id` semantics and its operation-producer-
  runtime ownership; the construction-ownership matrix; the required/optional evidence-mode
  distinction and failure-classification rules; carrying `evidence_profile`/`canonicalization_profile`
  inside the digested evidence material.
- **Required property, implementation mechanism deferred:** the specific digest algorithm beyond
  the recommended `sha-256` default; the pure helper's concrete implementation in `basis-adapters`.
- **Future schema possibility:** a reference-level field exposing `evidence_profile` and/or
  `canonicalization_profile` directly on the published `AdapterEvidenceReference` (as distinct from
  the evidence-material-internal fields this ADR already resolves), deferred unless a concrete
  consumer requires it.
- **Future producer-runtime decision:** where the operation-producer runtime lives (repository
  placement, per `operation-producer-and-execution-boundary.md` §11); how it authenticates to
  `basis-gateway`; whether `reference_id` uniqueness should ever become ecosystem-global; whether
  `reference_id` is reused or freshly minted on retry; which of the `required`/`optional` evidence
  modes a given deployment selects.
- **Future evidence-storage decision:** whether, where, and how evidence material is persisted for
  later retrieval and digest verification; stored-evidence locator semantics; whether artifact
  availability is guaranteed for any stated retention period.
- **Future evidence-profile revision:** a governed process for adding an approved protocol-metadata
  key (including closing REST's currently open-ended metadata for evidence-material purposes) as an
  additive revision of `basis-adapter-evidence-v1` or a new profile identifier — not decided by this
  ADR, and never a change made unilaterally inside `basis-adapters` or a producer runtime.

## References

- [`docs/architecture/adapter-evidence-construction-semantics.md`](../architecture/adapter-evidence-construction-semantics.md) — the detailed specification this ADR adopts
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) — the roles, trust rules, and decision gate (§13 Stage 2) this ADR resolves
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) §7 — the adapter evidence reference category this ADR narrows
- [`docs/architecture/operation-aware-evidence-provenance-semantics.md`](../architecture/operation-aware-evidence-provenance-semantics.md)
- [`docs/architecture/ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md)
- [`docs/security/threat-model.md`](../security/threat-model.md) §3.3, §4.2, §6.3
- [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](../roadmaps/identity-to-operation-contract-and-interoperability.md) — Phase 5's requirements-first, technology-selection-later discipline, applied here to canonicalization
- `basis-adapters`: `docs/operation-aware-handoff-alignment-plan.md` (the merged assessment this ADR answers), `docs/contracts/normalization-contract.md`, `docs/contracts/adapter-contract.md`, `docs/public-api.md`, `docs/compatibility.md`
- `basis-schemas`: `docs/adapter-evidence-reference.md`, `docs/redaction-classification.md`, `docs/contract-metadata.md`, `docs/operation-aware-decision-request.md`
- `basis-gateway`: `docs/operation-aware-endpoint.md`
- `basis-core`: `src/basis_core/domain/evidence.py` (the existing `AdapterEvidenceReference` domain model this ADR does not modify)
