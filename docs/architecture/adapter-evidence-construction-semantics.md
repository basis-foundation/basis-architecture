# Adapter Evidence Construction and Canonicalization Semantics

**Status:** Accepted architecture specification, adopted by [ADR-0007](../adr/0007-adapter-evidence-construction.md).
This document defines the authoritative architecture for constructing adapter evidence after
protocol normalization and before submission to `basis-gateway`. Acceptance authorizes downstream
implementation according to this decision; implementation remains separate and decision-gated. This
document does not itself implement Python runtime behavior, does not modify a JSON Schema, does not
create a producer runtime, does not choose a producer-authentication mechanism, and does not add
protocol execution or execution-result evidence. Throughout this document, "the operation-producer
runtime" names the logical role defined by
[`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §2.
[ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md) — Accepted —
establishes that this role's Foundation-maintained implementation is `basis-producer`. This
document's own construction/reference-assembly ownership split, below, is unchanged by that naming
decision: `basis-adapters` still constructs evidence material, canonicalizes it, and computes its
digest; the operation-producer runtime — `basis-producer` — still mints `reference_id` and assembles
the final `AdapterEvidenceReference`. The `basis-producer` repository now exists, and its first three
implementation phases are all implemented and merged: Phase 2A (evidence-retention foundation) —
it retains adapter-constructed evidence bytes, digest-addressed, and verifies them on retrieval —
Phase 2B (reference-lifecycle work: `reference_id` minting, reference binding, and final
`AdapterEvidenceReference` assembly), and Phase 3 (authenticated gateway client: independent mTLS
producer and bearer authorization-subject credentials, submission against `basis-gateway`'s
existing operation-aware endpoint). `basis-producer` now assembles `AdapterEvidenceReference`
instances and submits them to `basis-gateway`; it does not yet orchestrate `basis-adapters`
normalization to produce that evidence from a live protocol operation (Phase 4).

**Scope:** What constitutes adapter evidence material; how that material is serialized
deterministically; which component computes its digest; which component mints the evidence
reference identifier; which component assembles the final `AdapterEvidenceReference`; how
construction failures affect the operation handoff; and how future versions of the profile remain
verifiable against evidence created under an earlier one.

**Companion documents:** [`docs/architecture/operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md)
(§2, §4, §5, §11, §13 Stage 2 — the roles, handoff, and decision gate this document resolves),
[`docs/architecture/operation-aware-trace-audit-evidence.md`](operation-aware-trace-audit-evidence.md)
(§7 — the adapter evidence reference category this document narrows), [`docs/architecture/operation-aware-evidence-provenance-semantics.md`](operation-aware-evidence-provenance-semantics.md),
[`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md),
[`docs/security/threat-model.md`](../security/threat-model.md) (§3.3, §4.2, §6.3 — the trusted
adapter boundary and rogue-integrator/compromised-adapter adversaries this document extends),
[`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](../roadmaps/identity-to-operation-contract-and-interoperability.md)
(Phase 5's requirements-first, technology-selection-later discipline, applied here to
canonicalization), [`docs/glossary.md`](../glossary.md), `basis-adapters`'
`docs/operation-aware-handoff-alignment-plan.md` (the merged assessment this document answers),
`docs/contracts/normalization-contract.md`, `docs/contracts/adapter-contract.md`,
`docs/public-api.md`, `docs/compatibility.md`, `basis-schemas`'
`docs/adapter-evidence-reference.md`, `docs/redaction-classification.md`,
`docs/contract-metadata.md`, `basis-gateway`'s `docs/operation-aware-endpoint.md`.

---

## 1. Purpose and Scope

`basis-adapters` produces normalized protocol facts (`NormalizedAuthorizationRequest`) and
preserves protocol evidence (`protocol_evidence`, a verbatim `ProtocolOperation`) for all nine
currently published protocols and platforms. The ecosystem has published the safe-reference shape
those facts must eventually be summarized into — `basis-schemas`' `adapter-evidence-reference`
contract — and `basis-core` already models that reference verbatim as
`AdapterEvidenceReference`. What the ecosystem has not yet defined is the space between those two
facts: how normalization output becomes deterministic **evidence material**, how a **digest** is
calculated over that material, and how a future operation-producer runtime constructs the
`AdapterEvidenceReference` itself. `basis-adapters`' own merged handoff-alignment assessment
confirmed this precisely: three of the reference's four required fields
(`reference_id`, `evidence_digest`, `redaction_classification`) cannot be constructed responsibly
today because no canonicalization rule, minting-ownership decision, or classification-ownership
decision exists anywhere in the ecosystem, and building any of them without one would mean
inventing architecture inside an implementation repository. This document is that decision.

Stated precisely, the terms this document fixes mean:

- **Adapter evidence describes normalization facts.** It is what `basis-adapters` observed and
  produced while translating a protocol-native operation into BASIS semantics — nothing more.
- **Adapter evidence is not identity evidence.** It says nothing verified about who initiated the
  operation. A `subject_hint`-shaped value, wherever it appears, remains exactly what
  `basis-adapters`' own contract already calls it: an unverified hint, never promoted by this
  document into anything stronger.
- **Adapter evidence is not authorization evidence.** A digest, a reference, and a fully assembled
  `AdapterEvidenceReference` say nothing about whether the operation they describe was, is, or
  will be permitted. That determination belongs to `basis-core` alone, evaluating a request that
  may or may not carry this reference.
- **Adapter evidence is not execution evidence.** Nothing this document defines implies that a
  protocol operation was dispatched, attempted, or completed. That gap remains exactly where
  `operation-producer-and-execution-boundary.md` left it — unimplemented, and out of scope here.
- **A digest proves only correspondence with particular bytes under a defined canonicalization
  profile.** Given the same evidence material and the same profile, two conforming
  implementations produce the same digest; a verifier who recomputes it and gets a match has
  confirmed byte-for-byte correspondence with declared canonical input, nothing else.
- **A digest does not prove truth, authenticity, authorization, or execution.** A compromised
  adapter can compute a perfectly valid digest over fabricated evidence material. A digest is
  evidence of internal consistency between a claimed artifact and its claimed bytes; it is not,
  and this document does not let it become, a substitute for producer authentication, transport
  integrity, or authorization.

This document resolves the construction question. It does not relitigate the roles or trust rules
`operation-producer-and-execution-boundary.md` already established, and it does not expand
`basis-adapters` beyond a normalization library.

---

## 2. Governing Invariants

### Normalization invariant

The same valid protocol operation, adapter configuration, and governed mapping state must produce
the same normalized semantic evidence material. This restates, for evidence material specifically,
the determinism `basis-adapters`' own contract already requires of
`NormalizedAuthorizationRequest.to_dict()` ("same input always produces identical output") and of
policy evaluation more broadly (`docs/architecture/operation-aware-evaluation-semantics.md` §12–13).
Nondeterministic evidence material would make every downstream digest nondeterministic, which would
defeat the reference's purpose before canonicalization is even reached.

### Purity invariant

Evidence-material construction — assigned in §7 to `basis-adapters` — must remain deterministic,
synchronous, side-effect free, and independent of clocks, randomness, network calls, storage,
identity providers, gateway state, and kernel state. This is not a new rule invented for evidence
construction; it is the same purity discipline `docs/contracts/adapter-contract.md` and
`docs/contracts/normalization-contract.md` already impose on `normalize()` itself
("Normalization is a pure, in-process transformation. No network calls"), extended without
modification to whatever pure helper eventually computes evidence material and its digest.

### Ownership invariant

No component may claim ownership of facts it did not observe or establish. `basis-adapters`
observed the protocol operation and produced the normalized facts; it may own the evidence
material and its digest. It did not observe producer authentication, deployment configuration, or
redaction policy; it must not be assigned ownership of `reference_id` minting, `adapter_source`
selection under deployment policy, or `redaction_classification` assignment merely because those
values happen to travel alongside evidence it did produce.

### Identity invariant

`subject_hint` and other protocol-provided identity-like metadata — OPC UA's `session_id`, MQTT's
`client_id`, DNP3's `master_id`, IEC 61850's `origin`, KNX's `individual_address`, Niagara's
`niagara_user`/`niagara_role` — remain unverified evidence. None of them become authenticated
subject identity or producer workload identity merely because they are included in evidence
material, exactly as `docs/contracts/normalization-contract.md` and
`operation-producer-and-execution-boundary.md` §3 and §7 already state. This document does not
weaken that boundary; §4 decides, field by field, what evidence material may and may not carry
precisely to keep the boundary intact.

### Authorization invariant

Adapter evidence does not imply that the operation was allowed. A fully constructed
`AdapterEvidenceReference`, attached to a request the kernel later denies, remains exactly as valid
a piece of evidence as one attached to an allowed request — the reference describes normalization,
not the outcome of evaluating it.

### Execution invariant

Adapter evidence does not imply that protocol dispatch was attempted or that execution succeeded.
This restates `operation-producer-and-execution-boundary.md` §5's invariant that "an `ALLOW`
decision is never, on its own, evidence that an operation occurred," extended one step further
back: the evidence *supporting* the request that produced an `ALLOW` is, likewise, never evidence
that anything after the decision happened.

### Integrity invariant

A digest is not a signature, a producer credential, an authentication mechanism, or proof of
origin. `docs/roadmaps/identity-to-operation-contract-and-interoperability.md`'s "JSON and Contract
Semantics" section already states this as a general ecosystem principle — "no one mechanism
substitutes for all the others." This document does not create an exception for adapter evidence
digests: they carry no claim about who produced them or whether the channel that carried them was
protected in transit.

### Compatibility invariant

Previously created evidence must remain verifiable using the profile and version under which it
was created. §5 and §9 give this invariant a concrete mechanism: the canonicalization profile
carries its own version identity, and a verifier must know which profile version applied to a
given piece of evidence before attempting to recompute its digest.

---

## 3. Evidence Fact Model

Six distinct artifacts recur in this document. They are related, but none is a synonym for
another, and none is silently collapsed into `reference_id`.

```text
ProtocolOperation
    Protocol-native facts observed before normalization: protocol, method, path,
    and metadata, exactly as basis-adapters' AdapterContract already defines it.
    Produced by: basis-adapters, at the point of receiving wire-level input.

NormalizedAuthorizationRequest
    Normalized action/resource/protocol facts produced by the adapter: protocol,
    action, resource_type, resource_id, protocol_evidence, subject_hint. This is
    the existing, released, unchanged handoff artifact defined in
    docs/contracts/normalization-contract.md.
    Produced by: basis-adapters, via normalize().

Adapter evidence material
    A governed, deterministic, versioned semantic structure — defined in §4 —
    assembled from a subset of NormalizedAuthorizationRequest's own fields
    (never a new independent observation) and carrying its own evidence-profile
    and canonicalization-profile identity (§4, §9). It carries a governed
    projection of protocol_evidence, not the full object verbatim — §4 defines
    exactly which protocol_evidence keys the projection includes for each
    protocol. It is the exact input to canonicalization and, from there, to
    digesting.
    Produced by: basis-adapters, via a pure deterministic helper (§7).

Evidence digest
    An algorithm label plus a digest value, computed over the canonical bytes
    that canonicalizing the evidence material produces. Structurally identical
    to the evidence_digest_shape both adapter-evidence-reference and
    identity-evidence-reference already publish.
    Produced by: basis-adapters, via the same pure deterministic helper.

AdapterEvidenceReference
    The reference metadata object assembled for inclusion in an operation-aware
    request: reference_id, evidence_digest, adapter_source,
    redaction_classification, and the optional normalization_version,
    mapping_version, protocol, request_id, and correlation_id fields already
    published by basis-schemas and already modeled, byte-identically, by
    basis-core's AdapterEvidenceReference.
    Assembled by: the operation-producer runtime (§7).

Stored or retrievable evidence
    An optional future persistence concern — where, if anywhere, the full
    evidence material or protocol_evidence itself is retained so a reference
    can later be resolved back to something inspectable. Outside this
    decision except where §6 and §11 need to mention it to keep digest
    semantics coherent.
```

**What `reference_id` identifies.** Per §8, `reference_id` identifies one adapter-evidence
artifact — one instance of evidence material produced for one normalization/submission event. It
does not identify: a stored evidence object (none is required to exist by this document), a bare
normalization event independent of any evidence material (normalization can occur without ever
being submitted, and no reference is minted for a normalization that never proceeds), or the
`NormalizedAuthorizationRequest` itself (which has no reference of its own and is not the thing the
digest covers). `reference_id` does not silently mean all of these; it means exactly the first.

---

## 4. Adapter Evidence Material

### Alternatives considered

**Alternative A — Protocol evidence only.** Digest only `protocol_evidence` (the raw
`ProtocolOperation`). This proves that a specific claimed protocol operation was received, but it
does not prove anything about how `basis-adapters` normalized it — a compromised or buggy
normalization step could misnormalize the same `protocol_evidence` into an entirely different
`action`/`resource_type`/`resource_id`, and a digest scoped only to `protocol_evidence` would be
identical in both cases. This does not prove enough about the normalization result to be useful as
evidence that normalization was faithful, which is the property `operation-aware-trace-audit-
evidence.md` §7 and the threat model's rogue-integrator/compromised-adapter analysis (§4.2, §6.3)
both care about most. Rejected as the sole material.

**Alternative B — Normalized request only.** Digest only `action`, `resource_type`, `resource_id`,
and `protocol`. This loses the protocol-native facts needed to explain *how* the normalized result
was derived — an auditor or a future verifier could confirm that a request claimed to normalize to
`write:hvac` / `hvac:zone-a`, but could not confirm that claim against anything the protocol layer
actually said, defeating the evidentiary purpose of preserving `protocol_evidence` in the first
place (`docs/contracts/adapter-contract.md`: "Protocol evidence enables... audit trails that record
what wire-level operation triggered an authorization request"). Rejected as the sole material.

**Alternative C — Versioned combined evidence material.** Digest a governed structure containing
both selected normalized facts and selected protocol evidence facts, carrying its own version
identity. This is the only alternative that lets a verifier confirm both what was normalized and
what protocol-level input produced it, which is exactly the correspondence
`operation-aware-trace-audit-evidence.md` §7 asks adapter evidence to support ("audit should be
able to link a decision to the adapter evidence that produced it"). **This is the preferred
direction, adopted below.**

**Alternative D — Producer-runtime-defined material.** Let each producer runtime decide its own
evidence material shape. This would let independently built producer runtimes verify against
mutually incompatible evidence definitions — a digest computed by one producer's chosen material
would be meaningless to a verifier assuming a different producer's chosen material, defeating
interoperability before it exists. This directly contradicts the identity-to-operation
interoperability roadmap's Schema invariant ("`basis-schemas` publishes stable contracts; it does
not invent them") applied one level down: if evidence material is producer-defined, there is no
governed contract for a conformance test (Phase 8 of that roadmap) to check against. Rejected.

### Decision

Adapter evidence material is a **versioned combined evidence-material profile** (Alternative C).
This document adopts a specific, named first profile — not a placeholder shape awaiting a later
architecture decision:

```text
evidence profile: basis-adapter-evidence-v1
```

`basis-adapter-evidence-v1` is fixed by this document. A future adapter that needs a materially
different projection, or a future revision that adds a governed key, is a new, explicitly versioned
profile (an additive revision of `basis-adapter-evidence-v1`, or a new profile identifier
altogether) decided in `basis-architecture` — never a silent expansion performed inside
`basis-adapters` or a producer runtime.

`NormalizedAuthorizationRequest.protocol_evidence` itself is **unchanged** by this document. It
remains exactly what `docs/contracts/normalization-contract.md` already defines: the complete,
verbatim `ProtocolOperation` `basis-adapters` returns from `normalize()`, mandatory, never stripped,
summarized, or partially withheld from that return value. What this document governs is a
**separate, narrower thing**: the subset of `protocol_evidence` that is projected into digested
evidence material. `basis-adapters`' existing output contract and this document's evidence-material
profile are not the same artifact, and the second does not require, imply, or authorize any change
to the first.

The `basis-adapter-evidence-v1` profile includes, at minimum:

```text
evidence_profile                 the fixed literal "basis-adapter-evidence-v1" (this
                                  profile's own identity) — see §9
canonicalization_profile         the fixed literal "rfc8785" (§5, §9)
protocol                         from NormalizedAuthorizationRequest.protocol
normalized action verb           from NormalizedAuthorizationRequest.action
resource_type                    from NormalizedAuthorizationRequest.resource_type
local resource_id                from NormalizedAuthorizationRequest.resource_id
protocol evidence projection     a governed subset of NormalizedAuthorizationRequest
                                  .protocol_evidence — protocol, method, path, and
                                  only the approved metadata keys documented below,
                                  never the full, unrestricted metadata object
normalization_version            when the producer tracks a normalization-contract
                                  version distinct from the package version (a genuine
                                  gap in basis-adapters today, per its own handoff
                                  assessment §5 — included in the profile's shape now
                                  so a future normalization_version value has
                                  somewhere governed to go, without this document
                                  requiring basis-adapters to add it yet)
mapping_version                  when a mapping artifact has a governed identity or
                                  version (also a genuine gap today; same treatment)
adapter implementation identity   a stable identifier of the adapter implementation
                                  that performed normalization (for example
                                  "basis-adapters:bacnet") — see §9 for why this is
                                  distinct from producer workload identity and from
                                  deployment-instance identity
```

### Protocol evidence projection: governed, not open

`protocol_evidence` carries an open `metadata` object today (`docs/contracts/normalization-contract.md`
documents each protocol's metadata fields, but does not close every protocol's key set — see below).
Digesting that object unrestricted would mean the evidence-material profile's shape depends on
whatever a given `ProtocolOperation` happened to carry, which is exactly the kind of ungoverned,
producer-defined material Alternative D was rejected for. `basis-adapter-evidence-v1` instead
projects a **governed, closed set of approved keys per protocol** into evidence material:

| Protocol | Approved `metadata` keys for `basis-adapter-evidence-v1` |
| - | - |
| `bacnet` | `service`, `object_type`, `object_instance`, `property_identifier`, `device_id`, `priority`, `value_present` |
| `modbus` | `function`, `unit_id`, `address`, `quantity`, `value_present`, `register_type`, `source_address`, `transaction_id` |
| `opcua` | `service`, `node_id`, `attribute_id`, `method_id`, `namespace_index`, `identifier`, `identifier_type`, `browse_name`, `parent_node_id`, `subscription_id`, `monitored_item_id`, `value_present`, `endpoint_url`, `session_id` |
| `mqtt` | `operation`, `topic`, `client_id`, `qos`, `retain`, `payload_type`, `protocol_version` |
| `dnp3` | `operation`, `source_address`, `destination_address`, `outstation_id`, `master_id`, `object_group`, `variation`, `point_index`, `point_type`, `function_code`, `qualifier`, `control_code`, `control_model`, `event_class`, `value` |
| `iec61850` | `operation`, `ied_name`, `logical_device`, `logical_node`, `data_object`, `data_attribute`, `functional_constraint`, `dataset`, `report_control_block`, `goose_control_block`, `sampled_values_control_block`, `control_model`, `origin`, `cause`, `quality`, `timestamp`, `value` |
| `knx` | `operation`, `group_address`, `individual_address`, `device_address`, `communication_object`, `datapoint_type`, `payload_type`, `value`, `priority`, `area`, `line`, `device` |
| `niagara` | `operation`, `station`, `host`, `ord`, `component`, `slot`, `point`, `point_type`, `value`, `facet`, `schedule`, `alarm`, `history`, `category`, `baja_type`, `nav_path`, `niagara_user`, `niagara_role` |
| `rest` | *(none — see below)* |

These lists reproduce exactly the per-protocol metadata fields `docs/contracts/normalization-contract.md`
already documents for the eight protocols whose metadata shape it enumerates. This document does
not invent a new key for any of them; it closes, for evidence-material purposes only, what was
already an enumerated (if not schema-closed) list. `docs/contracts/normalization-contract.md`
itself, and `protocol_evidence` as `basis-adapters` returns it, are unaffected — an adapter may
still populate `metadata` with any field its normalization contract allows; this table governs only
what a subsequent evidence-material construction step is permitted to project from it.

**REST is the deliberate exception.** `docs/contracts/normalization-contract.md` documents REST's
`metadata` as open-ended ("may include `subject_hint` and any other HTTP-level fields the adapter
was given"), unlike the other eight protocols' enumerated lists. A governed, closed v1 key
vocabulary cannot honestly be declared for an open-ended field. Rather than fail evidence
construction for every real REST operation, `basis-adapter-evidence-v1`'s REST projection carries
`protocol`, `method`, and `path` only; `metadata` is not projected into evidence material for REST
under this profile version. This is recorded here as a known, honest limitation of the first
profile, not a defect to route around silently: a future profile revision may extend REST's
projection once `basis-adapters`' own normalization contract closes REST's metadata key vocabulary
the way the other eight protocols' contracts already do.

**Construction-time rules for the projection:**

- **Unknown metadata keys are rejected, not silently discarded.** A `metadata` key outside the
  approved list for a given protocol (or, for REST, any `metadata` key at all under this profile
  version) causes evidence-material construction to fail per §11 — it is never dropped quietly and
  never passed through unexamined.
- **Prohibited values are rejected regardless of key name.** A value that is a credential, token,
  secret, authorization header, private key, or other raw credential material causes construction
  to fail per §11 even if it were to appear under an approved key name. No currently documented
  approved key carries such a value today; this rule is defense in depth against a future mapping
  error, not a response to a known present defect.
- **`redaction_classification` is not a substitute for safe material construction.** A producer
  runtime cannot make an otherwise-prohibited value safe by assigning a stricter
  `redaction_classification` (such as `never_display`) to the assembled reference. Sensitivity
  classification (§10) governs handling of an artifact that was safely constructed; it does not
  retroactively excuse constructing evidence material that should have failed construction in the
  first place.
- **Adding a new approved key requires a governed profile revision.** Whether that revision is an
  additive update to `basis-adapter-evidence-v1` or a new profile identifier is decided in
  `basis-architecture` when a real need is demonstrated (consistent with `ROADMAP.md`'s "changed
  based on real integration needs... not speculative taxonomy work" principle) — it is never a
  change `basis-adapters` or a producer runtime makes unilaterally.

The profile excludes:

```text
subject_hint                     Excluded from the digested material entirely — see
                                  the identity-boundary discussion below.
secrets, credentials,
authorization headers,
tokens, session secrets,
transport-layer credentials      None of these exist on NormalizedAuthorizationRequest
                                  or ProtocolOperation today (confirmed by
                                  basis-adapters' own handoff assessment §5 and by
                                  adapter-evidence-reference's own secret prohibitions);
                                  this document requires they never be added to
                                  evidence material even if some future protocol
                                  metadata happened to carry one — see §10.
timestamps not observed
by the adapter                   basis-adapters observes no wall-clock time during
                                  normalization; a future evidence_material_profile
                                  must not manufacture one merely to have a
                                  timestamp field.
random identifiers                No identifier basis-adapters generates today, and
                                  none this document authorizes it to generate,
                                  belongs in the digested material — reference_id
                                  is minted after the material exists (§7, §8) and
                                  is not itself an input to the digest it identifies.
gateway-generated
identifiers,
kernel-generated
identifiers                       correlation_id (gateway-owned) and trace_id
                                  (kernel-owned) do not exist at normalization
                                  time and must never be backfilled into evidence
                                  material after the fact — doing so would make the
                                  same normalization event produce different
                                  digests depending on when it happened to be
                                  submitted, violating the normalization invariant.
execution results                 No execution-evidence concept exists yet
                                  (operation-producer-and-execution-boundary.md §12);
                                  none is retrofitted here.
```

**On `subject_hint`.** `subject_hint` is excluded from the digested evidence material, deliberately
and for a narrower reason than "it might be sensitive": including an identity-adjacent value inside
material a digest vouches for creates the appearance that the digest also vouches for that value's
identity meaning, which the identity invariant (§2) forbids. For the one protocol that populates the
top-level `subject_hint` field today (REST, via `metadata["subject_hint"]`), this exclusion is
consistent with, and not additional to, the governed-projection decision above: REST's `metadata` is
not projected into `basis-adapter-evidence-v1` evidence material at all, so `subject_hint`'s
underlying value is absent from evidence material regardless of the top-level field's own exclusion.
Every other protocol's identity-adjacent fields (`session_id`, `client_id`, `master_id`, `origin`,
`individual_address`, `niagara_user`/`niagara_role`) are on this document's approved-key list above
and remain part of the governed projection, carried into evidence material under each protocol's
own, already-governed "evidence only, never identity" discipline
(`docs/contracts/normalization-contract.md`) — this document does not add a second, competing
redaction pass over their content; §10 addresses why not.

---

## 5. Canonicalization Semantics

### Fixed canonicalization profile

This document adopts, exactly and without qualification, a first canonicalization profile:

```text
canonicalization profile: rfc8785
```

Evidence material carrying `evidence_profile: basis-adapter-evidence-v1` is canonicalized under
**RFC 8785, the JSON Canonicalization Scheme (JCS)** — exactly, with no BASIS-specific deviation,
substitution, or configuration point. Canonicalization is an **architecture-owned decision**,
adopted here and implemented by `basis-adapters`; it is not a producer-runtime choice, a deployment
configuration option, or a per-adapter setting. No operation-producer runtime, deployment, or future
adapter selects, substitutes, or configures a different canonicalization mechanism for evidence
material carrying the `rfc8785` canonicalization-profile identifier. A future profile that required
a different canonicalization technology would carry its own, distinct canonicalization-profile
identifier and would itself require a new architecture decision in this repository (§9) — it is not
an implementation-time choice available today.

The following properties restate, for evidence material specifically, exactly what RFC 8785
already requires. They are not a narrower, looser, or BASIS-specific paraphrase of RFC 8785; where
RFC 8785 and a prior draft of this document disagreed (Unicode normalization, key ordering), RFC
8785 governs and the prior language is withdrawn.

| Property | Requirement |
| - | - |
| Character encoding | UTF-8, without a byte-order mark, per RFC 8785. |
| Object-key ordering | Object member names are sorted according to **RFC 8785's UTF-16 code-unit ordering** — the same ECMAScript-derived comparison JCS specifies — applied recursively to every nested object, including every object nested inside the governed protocol-evidence projection (§4). This is not an independently chosen "Unicode code-point order"; it is RFC 8785's own ordering rule, adopted exactly. |
| Array ordering | Arrays retain their existing, constructed order. RFC 8785 does not reorder array elements, and this profile does not add a rule beyond what RFC 8785 already specifies. |
| Unicode normalization | **Not applied.** Unicode strings are preserved exactly as received, with no Unicode Normalization Form C (or any other normalization form) applied before or during canonicalization. RFC 8785 does not require or perform Unicode normalization, and this profile does not add one. A prior draft of this document required NFC normalization before serialization; that requirement is withdrawn as inconsistent with exact RFC 8785 adoption. |
| Whitespace | No insignificant whitespace, per RFC 8785's ECMAScript-derived serialization: no spaces after `:` or `,`, no newlines, no indentation. |
| String escaping | RFC 8785's string-escaping rules exactly — the same minimal, ECMAScript `JSON.stringify`-equivalent escaping JCS specifies. No BASIS-specific escaping variant. |
| I-JSON conformance | Evidence material, before canonicalization, must satisfy the I-JSON (RFC 7493) constraints RFC 8785 requires of its input: no duplicate object member names, no invalid Unicode (no unpaired surrogates), and every number representable as an IEEE 754 double-precision value. Evidence material that does not satisfy I-JSON is not canonicalizable under this profile; construction fails per §11. |
| Numeric values | Every number in evidence material must be representable under the **RFC 8785 / IEEE 754 double-precision model** and is serialized using RFC 8785's own number-serialization algorithm (the ECMAScript `Number::toString` algorithm). No fixed-precision, locale-dependent, or BASIS-specific numeric formatting is applied. |
| Values requiring greater precision | A value whose natural representation would require more precision than an IEEE 754 double can hold exactly (for example, a large protocol-native identifier near or beyond the double-precision safe-integer range, or a fixed-point decimal requiring exact decimal precision) **must be represented through a governed string field** in evidence material, not a JSON number, decided at construction time by §4's evidence-material field definitions — never coerced or detected at canonicalization time. A field §4 defines as string-typed is a string in the material from the moment it is constructed; canonicalization does not attempt to infer that a number "should have been" a string. |
| Booleans | Serialized as the literals `true`/`false`, per RFC 8785. |
| Null values | An explicitly-present field with value `null` is serialized as the literal `null`, per RFC 8785. RFC 8785 canonicalizes whichever JSON structure it is given; whether a given BASIS field may be absent versus `null` is a construction-time decision (§4), not a canonicalization-time one. |
| Duplicate object keys | Prohibited by I-JSON. Evidence material must never contain one; §4's field-based construction (never an open bag) prevents this by construction, and RFC 8785 itself does not define duplicate-key resolution — a malformed input that somehow contained one fails construction per §11, it is not resolved by "last key wins" or any other implicit rule. |
| Unsupported value types | Only JSON-representable, I-JSON-conformant types are permitted: object, array, string, boolean, `null`, and number within the IEEE 754 double-precision model (or, per the rule above, a governed string substitute for higher-precision values). A value of any other type (for example, a raw `bytes` object, a language-specific date/time object not already rendered to a string, or a custom class instance) is unsupported input; construction fails per §11 rather than serializing it with an implementation-specific fallback. |
| Non-finite numeric values | `NaN` and `±Infinity` have no JSON representation and are excluded by I-JSON. If a value that would otherwise be non-finite reaches evidence-material construction, construction fails per §11 rather than being coerced into a sentinel string or a lossy substitute. |
| Invalid Unicode | A string containing an unpaired UTF-16 surrogate, or otherwise failing I-JSON's Unicode-validity requirement, causes evidence-material construction failure per §11 — it is not sanitized, replaced, or silently repaired. |
| Nested metadata | The governed protocol-evidence projection (§4) is canonicalized in full, at whatever nesting depth the approved keys define, subject to every rule on this page applying recursively at every depth. |
| Empty objects and arrays | Serialized as `{}` and `[]` respectively, per RFC 8785 — distinct from the field being absent and distinct from `null`. |
| Omitted optional fields vs. explicit null | Governed by §4's evidence-material field definitions, not by canonicalization. A field the profile defines as optional (`normalization_version`, `mapping_version`) is either **absent** from the canonical structure or **explicitly `null`**, and these are not the same canonical byte sequence — consistent with `operation-producer-and-execution-boundary.md` §4's rule that "absence must remain distinct from empty or unknown." Construction (§4), not canonicalization, is what must preserve whichever of the two states the underlying source data actually carries. |
| Protocol metadata keys | **Governed, not open.** §4 fixes a closed, approved key set per protocol for `basis-adapter-evidence-v1`; canonicalization never encounters an ungoverned metadata key, because construction (§4) never admits one into evidence material in the first place. |
| Failure behavior when canonicalization is impossible | See §11. Canonicalization failure is a construction failure, not a silently-degraded digest. |

### No open technology evaluation

This document does not name RFC 8785 as a candidate awaiting a future technology-evaluation
decision, and it does not defer canonicalization-technology selection to a producer runtime or a
deployment. Canonicalization technology selection is **closed** for the
`basis-adapter-evidence-v1` / `rfc8785` profile pairing: `basis-adapters` implements RFC 8785
exactly, as a pure, deterministic helper (§7), and no other component substitutes a different
mechanism for evidence carrying this profile identifier. A future evidence or canonicalization
profile that required different serialization rules would be a new, explicitly versioned profile
(§9), decided in `basis-architecture` — not a configuration choice available today, and not
something any producer runtime, deployment, or future adapter may select independently.

---

## 6. Digest Semantics

**What is digested.** The exact canonical bytes produced by applying §5's canonicalization rules to
the evidence material defined in §4 — never the pre-canonicalization structure, never a partial
subset chosen at digest time, and never `protocol_evidence` alone independent of the rest of the
material.

**Allowed algorithms.** The published `evidence-digest` shape (identical across
`adapter-evidence-reference` and `identity-evidence-reference`) already governs this: an open,
lowercase, kebab-case algorithm label (pattern `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`), not a closed enum.
This document does not narrow that openness or select a mandatory default; it records `sha-256` as
the recommended initial algorithm (consistent with every existing schema example) without closing
the vocabulary, because `basis-schemas` already deliberately left it open for algorithm agility
("algorithm agility" below) and this document has no authority to close a contract it does not own.

**Algorithm agility.** Because the algorithm label is open and carried alongside every digest
value, a verifier always knows which algorithm to recompute against — a future migration to a
stronger algorithm does not require a schema change, only a new value for an already-open field.

**Digest value representation.** Already governed by the published contract: lowercase hexadecimal,
no `0x` or `algorithm:` prefix, no whitespace, no padding (pattern `^[a-f0-9]+$`). This document
does not redefine that representation.

**Verification failure representation.** This document does not invent a new representation for a
verification-failure result — no such contract exists to carry one, and inventing one would be a
schema decision this document is not authorized to make (§16). What this document does establish is
the *meaning* of a verification failure when one is eventually represented: it is an
evidence-integrity finding about a specific artifact, never a retroactive authorization or
execution finding (§14 develops this further).

### Profile identity and the verification flow

Every piece of evidence material carries its own `evidence_profile` and `canonicalization_profile`
identity as fields inside the material itself (§4, §9) — both fields are included in the canonical
bytes and therefore in the digest, exactly like any other evidence-material field. This is what
lets a verifier proceed without guessing which rules produced a given digest's input.
This is exactly the verification flow `basis-producer`'s Phase 2A/2B already implement today,
bounded to evidence it itself retained; a more general, externally-invocable verification
component beyond that bounded scope remains undecided (§16, §17). The flow:

```text
1. retrieve the evidence artifact (the retained evidence material — from
   `basis-producer`'s own digest-addressed local retention for this bounded
   reference implementation today, Phase 2A/2B, implemented; a broader,
   generalized evidence-storage architecture remains undecided — §7, §16);
2. read the artifact's own declared evidence_profile and canonicalization_profile
   fields;
3. canonicalize the retrieved material according to those declared profiles
   (today, exclusively basis-adapter-evidence-v1 / rfc8785 — §4, §5);
4. recompute the digest over the resulting canonical bytes, using the algorithm
   named on the reference's evidence_digest;
5. compare the recomputed digest against the reference's evidence_digest value.
```

**Verifiability is conditional, not unconditional.** A digest recomputation of this kind is only
possible while both the evidence artifact and its declared profile identifiers remain available and
interpretable — this document does not claim that evidence is verifiable independent of whether the
artifact itself, or a specification for the profile it declares, still exists. Concretely: evidence
produced under `evidence_profile: basis-adapter-evidence-v1` / `canonicalization_profile: rfc8785`
remains re-verifiable for as long as (a) the evidence artifact and its declared profile fields are
retained somewhere a verifier can retrieve them, and (b) a conforming RFC 8785 implementation
remains available to the verifier — not merely because the algorithm label is open or because a
schema field exists. For the bounded reference implementation, `basis-producer`'s Phase 2A
digest-addressed retention and Phase 2B durable reference binding already guarantee (a) within that
bounded scope, implemented: retained evidence remains retrievable by digest or by `reference_id`,
with verified integrity checked on retrieval. Whether a broader, generalized evidence-storage
architecture guarantees (a) beyond that bounded scope is explicitly not decided by this document
(§16); this document requires only that, whenever an
evidence artifact does remain available, its declared profile identifiers are sufficient, on their
own, for a verifier to reproduce the original canonical bytes without consulting anything not
already inside the artifact.

```text
digest equality
    proves the verifier produced the same digest over the same canonical bytes,
    under the same evidence_profile and canonicalization_profile the artifact
    itself declares.

digest equality does not prove:
    the evidence is truthful
    the adapter was uncompromised
    the producer was authenticated
    the request was authorized
    the operation was executed
```

---

## 7. Construction Ownership

| Operation                             | Adapter library | Producer runtime | Gateway | Evidence store |
| ------------------------------------- | --------------: | ---------------: | ------: | -------------: |
| Observe protocol operation            |          **Owns** |                 — |       — |               — |
| Normalize protocol facts              |          **Owns** |                 — |       — |               — |
| Construct semantic evidence material  |          **Owns** |                 — |       — |               — |
| Canonicalize evidence material        |          **Owns** |                 — |       — |               — |
| Compute evidence digest               |          **Owns** |                 — |       — |               — |
| Mint `reference_id`                   |                — |           **Owns** |       — |               — |
| Select `adapter_source`               |                — |           **Owns** |       — |               — |
| Assign redaction classification       |                — |           **Owns** |       — |               — |
| Attach request linkage                |                — |           **Owns** |       — |               — |
| Assemble `AdapterEvidenceReference`   |                — |           **Owns** |       — |               — |
| Authenticate producer                 |                — |                 — | **Owns** |               — |
| Verify request admission              |                — |                 — | **Owns** |               — |
| Persist full evidence                 |                — |         **Owns** (bounded, digest-addressed; Phase 2A, implemented) |       — |         Undecided (future, generalized architecture) |
| Verify stored evidence against digest |                — |         **Owns** (bounded; Phase 2A/2B, implemented) |       — |         Undecided (future, generalized architecture) |

The preferred ownership split, restated in prose:

```text
basis-adapters
    defines and constructs semantic adapter-evidence material
    canonicalizes it through a pure deterministic helper
    computes its digest through a pure deterministic helper

operation-producer runtime
    mints reference_id
    selects adapter_source from governed runtime configuration
    carries normalization_version and mapping_version when available
    assigns redaction_classification under deployment policy
    adds request and correlation linkage
    assembles the final AdapterEvidenceReference
    submits the governed operation to basis-gateway
    persists evidence material durably, digest-addressed, with verified
        retrieval, bounded to this reference implementation (Phase 2A,
        Phase 2B; implemented)

basis-gateway
    authenticates the producer
    classifies producer trust
    validates and admits the reference
    records the reference without regenerating or reinterpreting adapter facts

future, generalized evidence storage or evidence service
    would, if and when decided, generalize or supersede the bounded
    producer-owned retention above for cross-deployment or long-term
    archival needs — undecided, not scheduled here
```

This adopts this split unless source review demonstrates a stronger alternative; none did. It is
the direct consequence of §2's ownership invariant applied to each operation in turn:
`basis-adapters` is the only component that observed the protocol operation and performed
normalization, so it is the only component positioned to construct material and compute a digest
over facts it actually produced; everything downstream of that — minting an identifier, selecting a
label under deployment configuration, applying a redaction policy the deployment (not the protocol)
determines — is contextual assembly that depends on facts `basis-adapters` cannot observe (its own
runtime identity within a specific deployment, the deployment's redaction policy, the specific
request this evidence is being submitted alongside), which is exactly the operation-producer
runtime's role as already defined in `operation-producer-and-execution-boundary.md` §2 and §4.

**Computing a digest in `basis-adapters` does not make the adapter a trust authority.** The digest
proves internal consistency between the material and its bytes; it says nothing about whether the
operation-producer runtime that received the material is itself trustworthy, whether the channel
between adapter and producer runtime was protected, or whether the producer runtime faithfully
carried the material forward without alteration before assembling the final reference. Producer
authentication remains a separate control this document does not substitute for or weaken: the
normative bounded producer path is mutual TLS — [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md)
selects the profile, [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md)
defines the trusted-ingress topology, `basis-gateway` Phase 1B implements certificate-derived
producer identity and admission, and `basis-producer` Phase 3 presents the client certificate
alongside a separately obtained bearer authorization-subject credential. `basis-gateway`'s legacy
`OPERATION_PRODUCER_SUBJECT_IDS` classification remains a retained compatibility path, not a future
state still waiting to be superseded. Transport integrity and evidence verification (§6, §7) remain
separate controls as well.

---

## 8. `reference_id` Semantics

`reference_id` is an **opaque identifier for one adapter-evidence artifact or normalization
record**. It is not the digest itself, not a gateway correlation ID, not a kernel trace ID, not a
protocol transaction ID, not proof of authenticity, and not proof of uniqueness across all
deployments unless a future governed mechanism explicitly makes that guarantee.

Minting is assigned to the **operation-producer runtime**, not to the pure adapter normalization
call, for the reason `basis-adapters`' own handoff assessment already surfaced: minting a random or
time-derived identifier inside `normalize()` would introduce nondeterminism into a call the
normalization contract defines as a pure, side-effect-free transformation. This is consistent with
`operation-producer-and-execution-boundary.md` §4's deliberately open phrasing ("constructs an
`AdapterEvidenceReference` — or accepts one the adapter library helps construct") resolved in the
producer runtime's favor for this one field specifically, while §7 assigns the adapter library the
material and digest it *can* compute purely.

| Question | Answer |
| - | - |
| Is uniqueness deployment-local or ecosystem-global? | Deployment-local. This document does not require or assume a global uniqueness guarantee across independently operated deployments; a future cross-deployment interoperability contract (per the identity-to-operation roadmap) may impose a stronger requirement later, but none exists today. |
| Must the identifier be stable across retries? | Not required by this document. A producer runtime may reuse a previously minted `reference_id` for a retried submission of the same evidence material if its own design makes that choice deliberately and documents it; this document does not mandate reuse. |
| Does a retry of the same normalized operation reuse or create a new reference? | Either is permitted; this document does not decide it. What is fixed regardless: digest equality across the retry is guaranteed by the normalization invariant (§2) independent of whether `reference_id` is reused, because the digest is a function of the evidence material, not of `reference_id`. |
| May identical evidence material have more than one `reference_id`? | Yes. Two independent normalization/submission events over the same protocol operation (for example, two retries that each mint a fresh identifier) legitimately produce the same digest under two different `reference_id` values. This is not an inconsistency — verification is keyed to the digest, never to `reference_id` exclusivity. |
| May one `reference_id` point to more than one evidence artifact? | No. Each `reference_id`, once minted, identifies exactly one evidence artifact. A producer runtime that reused a `reference_id` for materially different evidence would violate this document's ownership and normalization invariants, not merely a naming convention. |

This document does not choose a UUID version or generation library. UUID v4 remains the recommended
convention already stated by `adapter-evidence-reference`'s own documentation; nothing here narrows
or replaces that recommendation.

---

## 9. Source, Version, Mapping, and Profile Metadata

| Field | Semantics | Owner |
| - | - | - |
| `evidence_profile` | The fixed literal `basis-adapter-evidence-v1` (§4) — identifies which governed evidence-material shape (field set, protocol-metadata projection) produced this artifact. Included in the digested material itself, not merely alongside it. A future profile revision carries a new, distinct literal; this document does not version this field independently of the profile it names. | `basis-adapters`, as part of evidence-material construction (§7). |
| `canonicalization_profile` | The fixed literal `rfc8785` (§5) — identifies which canonicalization rules produced this artifact's canonical bytes. Included in the digested material itself. A future canonicalization profile carries a new, distinct literal. | `basis-adapters`, as part of evidence-material construction (§7). |
| `adapter implementation identity` | A stable identifier of the **adapter implementation** that performed normalization — for example, "this evidence was produced by the BACnet adapter implementation," independent of any specific deployment's configuration of it. This is distinct from two other identities this document does not let it be confused with: **producer workload identity** (the authenticated identity of the operation-producer runtime process that submits the operation, established entirely by the now-implemented mTLS producer-authentication mechanism per `operation-producer-and-execution-boundary.md` §3 — ADR-0008, ADR-0009; `basis-gateway` Phase 1B and `basis-producer` Phase 3, implemented — `basis-adapters` never establishes, asserts, or contributes to this) and **deployment-instance identity** (a deployment-chosen label for one configured instance of an adapter within one deployment, such as `AdapterContext.adapter_id`'s `"rest-primary"` example — meaningful only within that deployment's own configuration, and not itself a claim about which adapter implementation or code version ran). Evidence material carries adapter implementation identity only; `adapter_source` on the assembled reference (below) is where a producer runtime may combine implementation identity with deployment-instance identity into the single opaque label the published schema expects — that combination is reference-assembly work, not evidence-material construction. | `basis-adapters`, as part of evidence-material construction (§7), for the implementation-identity component; the operation-producer runtime, for combining it with deployment-instance identity into `adapter_source`. |
| `adapter_source` | Runtime/deployment identity of the adapter or normalization component that produced the evidence, as it appears on the assembled `AdapterEvidenceReference` — **not necessarily the Python class name**, and not the same field as evidence material's own adapter-implementation-identity value above, though a producer runtime may derive one from the other. `basis-adapters`' own handoff assessment already confirms this is constructible today from existing public fields (`protocol`, optionally `AdapterContext.adapter_id`) without any library change. | Operation-producer runtime, selecting from governed runtime configuration. |
| `normalization_version` | The version of the adapter normalization logic that produced the evidence, when tracked — **not automatically the same concept as the `basis-adapters` package version**. No such value exists in `basis-adapters` today; this document does not require `basis-adapters` to add one, and a future value, if added, must be an explicit normalization-contract version distinct from `pyproject.toml`'s package version, per the handoff assessment's own finding (§5 of that document). | `basis-adapters`, if and when it adds this field; carried forward by the producer runtime otherwise absent. |
| `mapping_version` | Meaningful only when a mapping artifact (a per-protocol mapping-configuration file) has a governed identity or version of its own. No mapping configuration in any of the nine adapters carries one today. **Absence of mapping version must not cause a fabricated placeholder** — a producer runtime that cannot determine a real mapping version must omit the field or pass `null`, never invent a synthetic value such as `"unknown"` or `"1.0.0"` as a default. | `basis-adapters`, if and when a mapping artifact gains a governed version; carried forward by the producer runtime otherwise absent. |
| `protocol` | The open, lowercase protocol label — already sufficient today, unchanged. | `basis-adapters`. |
| `request_id` | Remains producer-owned per `operation-producer-and-execution-boundary.md` §6: generated by the operation-producer runtime (or, absent one, the caller); `basis-gateway` never overwrites a caller-supplied value. | Operation-producer runtime (or caller). |
| `correlation_id` | Remains gateway-owned: generated unconditionally by `basis-gateway`'s `CorrelationMiddleware`; caller-supplied values are explicitly ignored by documented gateway policy. When an evidence reference carries its own `correlation_id`, it is a producer-supplied, pass-through value distinct from the gateway's own correlation identifier for the request it ends up attached to — consistent with the "broader cross-system trace id" semantics `adapter-evidence-reference.md` already documents. | Gateway, for the request's own correlation identifier; producer runtime, for whatever value it optionally attaches to the reference itself. |

The `evidence_profile` and `canonicalization_profile` rows are new, fixed decisions of this
document (§4, §5), not open questions. The remainder restates, without altering, the
field-ownership table already published in `operation-producer-and-execution-boundary.md` §4,
applied here specifically to the metadata fields `basis-adapters`' own handoff assessment found
genuinely open (`normalization_version`, `mapping_version`) or genuinely sufficient already
(`protocol`, and `adapter_source` once combined with a deployment-instance label).

**What can already be populated from released surfaces today, versus what requires a future
additive implementation change:**

```text
Already populatable today, without any basis-adapters change:
    protocol                          (existing, stable field)
    adapter_source                     (constructible by a producer runtime from
                                      adapter implementation identity + optional
                                      AdapterContext.adapter_id deployment-instance
                                      label)
    evidence_profile                   (fixed literal, this document)
    canonicalization_profile           (fixed literal, this document)

Requires a future additive basis-adapters implementation change, only if a real
consumer justifies it (per ROADMAP.md's own principle: "basis-adapters... should
be changed based on real integration needs... not speculative taxonomy work
performed ahead of it"):
    normalization_version    (genuine gap; no such value exists today)
    mapping_version           (genuine gap; no mapping artifact carries a version
                             today)
```

---

## 10. Redaction and Sensitive Material

Sensitivity classification responsibility, without implementing redaction:

- **Credentials or tokens accidentally present in protocol metadata.** None exist in any of the
  nine adapters' documented `metadata` shapes today (`docs/contracts/normalization-contract.md`),
  and `adapter-evidence-reference`'s own schema already rejects `credential`, `password`,
  `api_key`, `private_key`, and `unredacted_device_secret` as unknown fields at the reference level.
  This document adds no new field-level scanning behavior inside `protocol_evidence.metadata`
  itself — doing so would require `basis-adapters` to interpret protocol-specific metadata
  semantically, which contradicts its protocol-neutral evidence-material role (§4). The correct
  control is upstream, at the point a protocol mapping is authored and reviewed (a `basis-adapters`
  mapping-configuration concern), not inside evidence-material construction.
- **Authorization headers, session identifiers, device secrets, operator names or identifiers,
  network addresses, location and infrastructure metadata, values that may reveal operational
  state.** All remain inside `protocol_evidence.metadata` exactly as each protocol's already-
  documented "evidence only, never identity" fields describe them (session IDs, client IDs, device
  addresses, and the like). This document does not add a second, adapter-evidence-specific
  redaction pass over the same values `docs/contracts/normalization-contract.md` already governs
  per protocol — doing so would create two competing redaction authorities for the same data.

**Where sensitive values are handled.** Consistent with §7's ownership split: values are neither
excluded before canonicalization by `basis-adapters` (which does not interpret protocol-specific
metadata semantics) nor replaced with governed redacted representations at construction time.
Instead, the entire assembled reference — including whatever sensitivity the underlying evidence
material carries — is assigned a `redaction_classification` by the operation-producer runtime under
deployment policy, choosing among the five already-published values
(`safe_to_expose`, `safe_after_redaction`, `reference_only`, `never_store`, `never_display`). This
is a reference-level classification of the artifact the digest points to, not a field-level
redaction pass inside the material itself — `adapter-evidence-reference`'s own documentation
already frames `redaction_classification` this way ("handling requirement for the adapter evidence
artifact this reference points to").

**Digest coherence with retained representation.** The digest is calculated over the evidence
material as constructed (§4) — the same representation that `basis-producer`'s bounded Phase 2A
retention (implemented) retains today, and that a future, more generalized evidence store, if one is
ever decided, would retain for later inspection under whatever `redaction_classification` the
reference carries.
This document does not define a digest over material that is then discarded in a way that would
make later verification incoherent: if the evidence material contains something a
`never_store` classification would prohibit retaining, the resolution is that a future evidence
store simply does not retain that material at all under that classification — the digest remains
valid as a structural fact about bytes that once existed and were verified at construction time,
even if nothing is later persisted for a stranger to re-derive from. This document does not
mandate that evidence be persisted; §11 and §16 record persistence as future, deferred work.

**Final `redaction_classification` selection is assigned to the producer runtime or evidence
boundary under deployment policy, not to the protocol-normalization library** — restated directly
from §7's ownership table, because `basis-adapters` has, correctly, no sensitivity classification
of any kind today, and this document does not give it one.

---

## 11. Failure Semantics

**The deployment evidence mode.** Every deployment operates in exactly one of two evidence modes,
selected before the operation is submitted — not decided reactively, per-failure, at construction
time:

```text
required
    any evidence-construction failure blocks operation submission. This is
    the default and the conservative choice.

optional
    evidence may be omitted only when deployment policy explicitly permits
    omission, and only when the omission's reason is recorded auditably. A
    construction failure must never silently downgrade required evidence
    into an unrecorded optional omission — the mode is chosen ahead of time,
    not inferred after a failure occurs.
```

The table below classifies each failure condition under this mode distinction.

| Condition | Classification |
| - | - |
| Evidence-material construction failure (an unsupported value type reaches construction, a non-finite number, a duplicate key in malformed input) | **Required evidence-construction failure.** Under `required` mode, the operation-producer runtime must not submit the operation for authorization using this evidence. Under `optional` mode, the runtime may submit without evidence only if deployment policy explicitly permits the omission and records why. |
| Unsupported evidence value type | Same as above — a specific cause of required evidence-construction failure, not a distinct category. |
| Canonicalization failure (the material cannot be rendered into the canonical form §5 requires) | Same mode-dependent treatment as evidence-material construction failure, above. |
| Digest-generation failure (the selected algorithm is unavailable in the runtime, for example) | Same mode-dependent treatment as evidence-material construction failure, above. |
| Reference-ID generation failure | Same mode-dependent treatment as evidence-material construction failure, above, attributed to the operation-producer runtime rather than `basis-adapters`, since minting is its responsibility (§8). |
| Invalid `AdapterEvidenceReference` assembly (a required field is missing, or the assembled object fails the published schema's own validation) | Same mode-dependent treatment as evidence-material construction failure, above. |
| Evidence persistence failure (the retention mechanism cannot durably retain material it was asked to) | A distinct category from all of the above — an evidence-integrity/operational concern for whatever component owns persistence (§7) — `basis-producer`'s bounded Phase 2A retention, implemented, today; a future, generalized evidence-storage component, if one is ever decided — not a normalization or construction failure. This document does not decide its handling beyond noting it is governed by the same `required`/`optional` deployment evidence mode, not treated as an automatic required-evidence-construction failure irrespective of mode. |
| Later digest-verification failure (a verifier recomputes a digest against retained material and gets a mismatch) | **Evidence-integrity failure**, never retroactive rewriting of the original authorization decision — see §14. Not governed by the evidence mode, which applies only at construction/submission time. |

The governing distinctions, restated from `operation-producer-and-execution-boundary.md` §8 and
applied specifically to evidence construction:

```text
normalization failure
    no governed handoff may proceed — this is basis-adapters' own existing
    fail-closed rule (AdapterResult.success is False), unchanged by this
    document.

required evidence-construction failure (under a deployment's "required" evidence mode)
    the operation-producer runtime must not submit the operation for
    authorization. This is the category every row above falls into whenever
    the deployment is operating in "required" mode.

evidence explicitly and auditably omitted (under a deployment's "optional" evidence mode)
    the request may proceed without an adapter-evidence reference at all —
    adapter_evidence_reference is optional on operation-aware-decision-request
    (basis-schemas §9) — but only because the deployment selected "optional"
    mode before the operation was submitted, and only with the omission
    reason recorded auditably. A producer runtime that silently drops evidence
    it could have constructed, without recording why, has created exactly
    the kind of undocumented gap operation-aware-trace-audit-evidence.md §15
    already warns against for audit-emission failure generally. A
    construction failure occurring under "required" mode is never
    reclassified into this category after the fact.

gateway rejection of an invalid evidence reference
    a request-admission or composition failure at basis-gateway's existing
    boundary, not a kernel authorization result — the gateway's own
    OperationAwareEvaluateRequest validation (extra="forbid", structural
    schema checks) already rejects a malformed reference with 400 before the
    kernel is ever invoked, exactly as it does for any other malformed field.

later evidence-verification failure
    an evidence-integrity failure, not retroactive rewriting of the original
    authorization decision — developed further in §14.
```

This document does not create operation-class exceptions such as "writes always fail closed but
reads always continue." A `read` operation whose evidence-material construction fails is subject to
the same required-evidence-construction-failure rule as a `write` operation; whether the request may
proceed *without* evidence at all is governed by whether `adapter_evidence_reference` is required
under a given deployment's profile, not by the operation's verb.

**Distinguishing a construction failure from a kernel `DENY`.** A failed evidence-material
construction happens entirely upstream of `basis-gateway` and `basis-core` — the kernel is never
invoked, and no `DecisionRequest` or `OperationAwareDecisionRequest` is ever assembled with the
failed evidence. This is not a governed `DENY` outcome (`basis-core` never sees the request); it is
a producer-runtime decision not to submit, consistent with `operation-producer-and-execution-
boundary.md` §5's rule that "producer-runtime errors must fail closed" without inventing a kernel
outcome to describe them.

---

## 12. Correlation and Retry Semantics

Restating and narrowing `operation-producer-and-execution-boundary.md` §6 for evidence
specifically:

| Identifier | Relationship to adapter evidence |
| - | - |
| Protocol-native transaction ID | When one exists (Modbus's `transaction_id`, a DNP3/IEC 61850 control-block identifier), it is carried as part of `protocol_evidence.metadata` — inside the digested material, not a sibling of `reference_id`. |
| Producer request ID (`request_id`) | Distinct from `reference_id`: associates evidence with one specific decision request, optional and nullable because evidence is commonly produced before the specific request exists (§9). |
| Adapter evidence reference ID (`reference_id`) | Identifies the evidence artifact itself (§3, §8) — never conflated with any identifier in this table. |
| Gateway correlation ID | Gateway-owned, generated unconditionally per request; never influenced by, or influencing, evidence-material construction. |
| Kernel trace ID | Kernel-owned; not applicable to `basis-adapters` or evidence construction at all. |
| Future execution evidence ID | Does not exist yet; not addressed by this document. |

**Retries.**

- The same semantic evidence material produces the same digest on every recomputation — this is
  the normalization invariant (§2), not a special retry rule, but it is worth stating plainly for
  retries specifically: a producer runtime that retries a submission after a transient gateway
  failure, using the same normalized operation, recomputes (or reuses, if it cached the result) an
  identical digest every time.
- `reference_id` may be reused or freshly minted on retry, per §8's deliberate non-decision; either
  is compatible with digest determinism.
- `request_id` reuse across a retry is a producer-runtime and gateway concern this document does not
  decide — `operation-producer-and-execution-boundary.md` §4 already states the general rule that
  no component overwrites an identifier owned by another, which continues to apply.
- Duplicate submissions remain distinguishable from repeated execution because no execution concept
  exists in this document's scope at all — a duplicate *evidence submission* is a fact about the
  authorization chain; whether anything was ever dispatched twice is an execution-lifecycle question
  `operation-producer-and-execution-boundary.md` §12 already assigns elsewhere, unaffected by
  anything decided here.
- Evidence linkage never claims causation merely from matching identifiers. Two audit records
  sharing the same `reference_id` prove that both claim to describe the same evidence artifact; they
  do not prove that the artifact is genuine, that the component that wrote either record was
  authorized to, or that the operation the evidence supports actually occurred as described.

> Correlation establishes deterministic linkage. It does not independently prove authenticity,
> authorization, causation, or execution.

---

## 13. Threat Analysis

Extends `docs/security/threat-model.md` §3.3 (the trusted adapter boundary), §4.2 (rogue
integrator, compromised adapter), and §6.3 (adapter-specific threats), and
`operation-producer-and-execution-boundary.md` §7 (the provenance and fact-ownership matrix), to
the specific construction mechanics this document adds.

| Threat | Affected trust boundary | Prevention / mitigation | Residual risk | Owning component |
| - | - | - | - | - |
| Evidence-material tampering before canonicalization | Internal to the pure evidence-construction step, inside `basis-adapters`' own process | Purity invariant (§2): the step is deterministic and side-effect free, minimizing the code surface where tampering could occur silently; no network or storage access exists during construction to inject altered material | A fully compromised `basis-adapters` process could still construct a tampered but internally-consistent digest — this document reduces the surface, it does not eliminate a compromised-component threat, consistent with the threat model's existing candor about adapter compromise (§4.2) | `basis-adapters` |
| Normalized-output substitution | Adapter → producer runtime handoff | The digest covers normalized output as part of the evidence material (§4); a substitution after digesting would produce a digest mismatch upon verification | A bounded verification mechanism exists today within `basis-producer`'s own Phase 2A/2B implementation — retained evidence is digest-checked at retention and again at reference resolution (`resolve_reference_evidence()`) — but that verification is internal to the reference producer's own retained evidence; no additional, external, cross-deployment evidence-verification component exists beyond that bounded scope (§16) | Operation-producer runtime (must not alter material after receiving it, and verifies via its own bounded retention mechanism — Phase 2A/2B, implemented) |
| Protocol-evidence substitution | Same as above | Same as above — `protocol_evidence` is included in the digested material precisely so substitution is, in principle, detectable | Same as above | Same |
| Canonicalization ambiguity | Any two independent implementations of the pure helper | §5 fixes the canonicalization profile exactly (`rfc8785`, RFC 8785 JCS) rather than leaving it open: two conforming implementations cannot disagree on canonical bytes for the same material, because both are conforming to the same external specification, not to a `basis-architecture`-invented scheme | The profile decision itself is closed, and a reference implementation now exists and is released (`basis-adapters` `v0.2.0`'s `construct_adapter_evidence()`, tested against RFC 8785's own number-serialization and key-ordering test vectors, plus cross-protocol conformance tests exercising all nine adapters — `tests/test_evidence.py`); the remaining risk is narrower — continued conformance across future `basis-adapters` versions and any independent implementation, not the absence of a reference implementation | `basis-architecture` (for the requirements); `basis-adapters` (for released conformance and its continued maintenance) |
| Digest algorithm downgrade | Verifier ↔ evidence relationship | Algorithm identity travels with every digest value (§6); a verifier is never forced to trust an unlabeled or ambiguous algorithm | A verifier that accepts a weak algorithm label without deployment-level policy could still be induced to trust weak evidence — algorithm *acceptance* policy is a deployment/verifier concern this document does not govern | `basis-producer`'s bounded digest-recomputation mechanism (Phase 2A/2B, implemented); deployment policy (algorithm-acceptance policy remains undefined) |
| Digest substitution | Producer runtime → gateway → any future, generalized evidence store | The reference's `evidence_digest` is exactly what a verifier recomputes against; a substituted digest is detectable the moment verification against real evidence material occurs | The same bounded verification mechanism named for normalized-output substitution above applies: `basis-producer`'s Phase 2A/2B digest-recomputation catches substitution within its own retained-evidence scope; a substitution occurring entirely outside that bounded retention path remains undetectable in practice | Operation-producer runtime, via its own bounded verification mechanism (Phase 2A/2B, implemented); `basis-gateway` (admission only, not regeneration — §7) |
| `reference_id` collision or reuse | Producer runtime's minting responsibility | §8 requires uniqueness only at deployment scope, not ecosystem scope, and requires that one `reference_id` map to exactly one evidence artifact | A producer runtime that violates its own uniqueness discipline (reusing an identifier for materially different evidence) creates ambiguity this document has no mechanism to detect after the fact | Operation-producer runtime |
| Replay of an old evidence reference | Correlation, per §12 | `request_id`/`correlation_id` association narrows replay's blast radius by tying evidence to a specific request context, but this document does not add nonce-based or time-bounded replay protection | Consistent with the threat model's own candid position on replay generally (§7.6): "comprehensive replay protection... is partly a gateway-implementation and deployment concern rather than a fully settled architectural guarantee" | `basis-gateway`; deployment |
| Producer-runtime compromise | The entire handoff between `basis-adapters` output and `basis-gateway` admission | Ownership separation (§7): a compromised producer runtime can assemble a dishonest reference, but it cannot alter what `basis-adapters` itself already computed and returned, and it cannot forge gateway-owned or kernel-owned facts | This is the same residual risk `operation-producer-and-execution-boundary.md` §3 already names for the entire producer-runtime role; this document does not reduce it further | Deployment (producer-runtime credential protection); `basis-gateway` (authentication) |
| Evidence-reference attachment to the wrong request | Request/correlation linkage, §9, §12 | `request_id` on the reference associates it with one specific decision request; a producer runtime that attaches the wrong reference to a request has made a construction-time error this document does not detect automatically | No automatic cross-field consistency check exists — `operation-aware-decision-request.md` §26 already states this explicitly for the parent-field/nested-reference relationship generally | Operation-producer runtime; a future conformance test (§17, Stage 2) |
| Secret leakage through protocol metadata | `protocol_evidence.metadata`'s open key set | §10: no credential/token/secret field exists in any currently documented adapter metadata shape, and `adapter-evidence-reference`'s own schema rejects known secret-shaped field names at the reference level | A future protocol mapping could still, through misconfiguration, place a genuinely sensitive value inside an otherwise-innocuous `metadata` key this document has no field-name-based way to catch | `basis-adapters` mapping-configuration review (upstream of evidence construction, out of this document's scope) |
| Unbounded or recursively nested metadata | Canonicalization, §5 | §5 requires the canonicalization rules to apply "recursively at every depth," and §11 requires construction to fail on unsupported value types rather than silently truncate | This document does not impose a maximum depth or size bound on `metadata` — an extremely large or deeply nested value would canonicalize successfully but could still create a resource-exhaustion concern this document does not address | Future implementation (a depth/size bound, if warranted, is an implementation decision, not decided here) |
| Evidence deletion after a reference was emitted | Bounded producer-local retention (implemented); any future, generalized evidence storage | Partially mitigated for the bounded reference implementation: `basis-producer`'s Phase 2A defines a real digest-addressed local persistence mechanism with verified retrieval (§7), so evidence is not undefined-and-unretained by default. This document does not define a retention-period, replication, or deletion-audit-trail policy for that bounded store, and a broader, generalized ecosystem-wide evidence-storage architecture remains undecided (§16) | A `reference_id` pointing to evidence deleted from the bounded local store, or never durably retained due to an out-of-band failure, cannot be verified after the fact; this is a genuinely open question for whatever broader evidence-storage architecture, if any, eventually generalizes beyond the bounded reference implementation's local retention | `basis-producer` (bounded local retention, implemented — Phase 2A); future, generalized evidence-storage architecture (undecided) |
| False claims that digest verification proves execution | Conceptual/documentation risk | §1's explicit definitions and §6's closing statement ("digest equality does not prove... the operation was executed") foreclose this claim architecturally | None — this is a documentation discipline, not a technical control, and its only "residual risk" is a future document failing to repeat the caveat | `basis-architecture` (documentation discipline going forward) |

Hashing does not make a compromised producer trustworthy. Nothing in this document should be read,
or is intended to be read, as suggesting that a digest — however carefully canonicalized —
substitutes for producer authentication, transport integrity, or the kernel's own authorization
decision.

---

## 14. Representative Scenarios

### Scenario A — BACnet `WriteProperty`

```text
ProtocolOperation
    protocol=bacnet, method=WriteProperty,
    path="AV:4:presentValue", metadata={service, object_type, object_instance,
      property_identifier, device_id, priority, value_present}
→ normalized request
    action=write, resource_type=point, resource_id=<rendered>, protocol=bacnet
→ versioned adapter evidence material
    {evidence_material_profile_version, protocol, action, resource_type,
     resource_id, protocol_evidence: {protocol, method, path, metadata}}
    — device_id and priority are carried inside protocol_evidence.metadata,
    exactly as documented: evidence-only, never identity, never a control
    obligation.
→ canonical bytes
    UTF-8, sorted keys, no insignificant whitespace, per §5
→ evidence digest
    {algorithm: "sha-256", value: "<64 lowercase hex chars>"}
→ producer-created AdapterEvidenceReference
    {reference_id: <minted by producer runtime>, evidence_digest, adapter_source:
     "basis-adapters:bacnet", normalization_version: null (not tracked today),
     mapping_version: null (not tracked today), protocol: bacnet,
     redaction_classification: reference_only, request_id: <if known>}
→ gateway operation-aware request
    Attached as adapter_evidence_reference on OperationAwareEvaluateRequest,
    accepted only from a classified trusted producer, and classified
    trusted_producer_asserted in gateway-owned provenance — never verified.
```

Evidence-only identity-like fields in this scenario: `device_id`, `priority` (both inside
`protocol_evidence.metadata`). Sensitive values: none observed in this scenario — BACnet's
documented metadata carries no credential-shaped field.

### Scenario B — Modbus register write

```text
ProtocolOperation
    protocol=modbus, method=WriteSingleRegister,
    path="unit:12:addr:40012", metadata={function, unit_id, address, quantity,
      value_present, register_type, source_address, transaction_id}
→ normalized request
    action=write, resource_type=register, resource_id=<rendered>, protocol=modbus
→ versioned adapter evidence material
    Same structure as Scenario A — the profile does not vary by protocol. What
    varies is the content: Modbus's protocol_evidence carries only a function
    code, a unit/address pair, and (where present) a transaction_id. No object
    model, no device-identifier field beyond unit_id/source_address, no
    priority or origin concept.
```

Canonicalization does not invent device, safety, identity, or execution facts the protocol does not
provide. The evidence material for this operation is, correctly, sparser than Scenario A's — there
is no synthesized "device class" and no synthesized "safety mode" merely to make the two protocols'
evidence material look more alike than the underlying protocols actually are. A digest over this
sparser material is exactly as valid as one over BACnet's richer material; it simply attests to less,
because there is less to attest to.

### Scenario C — OPC UA `Browse` (read-only)

```text
ProtocolOperation
    protocol=opcua, method=Browse, path="ns=2;s=Building.AHU1",
    metadata={service, node_id, browse_name, parent_node_id, session_id,
      endpoint_url}
→ normalized request
    action=browse, resource_type=node, resource_id=<rendered>, protocol=opcua
→ versioned adapter evidence material
    Constructed identically to Scenarios A and B — this document's evidence
    material profile does not branch on operation_intent or on the underlying
    action verb. session_id and endpoint_url are carried inside
    protocol_evidence.metadata under the same "evidence only, never identity"
    discipline documented for every protocol.
```

This demonstrates that the architecture is not designed exclusively for writes or control actions:
a `browse` operation's evidence material is constructed by the identical procedure as a `write`
operation's, with no special-casing for read-only operations anywhere in §4 through §11.

### Scenario D — Construction failure

```text
A future adapter (or a future metadata field on an existing one) produces a
protocol_evidence.metadata value that is not JSON-representable — for example,
a raw bytes object carrying an undecoded payload fragment that a protocol
mapping author mistakenly left un-rendered to a string.

→ evidence-material construction reaches this value
→ per §5 ("unsupported value types") and §11 ("evidence-material construction
   failure"), construction fails
→ this is a required evidence-construction failure (§11): the operation-
   producer runtime must not submit the operation for authorization using
   this evidence
→ no kernel DENY is fabricated: basis-core is never invoked, because no
   request carrying this evidence was ever assembled
→ the producer runtime fails closed, per operation-producer-and-execution-
   boundary.md §5's existing rule for producer-runtime errors generally
```

### Scenario E — Retry

```text
A producer runtime submits an operation-aware request; the gateway is
temporarily unreachable (a network timeout, not a governed disposition).
The producer runtime retries the identical normalized operation.

→ NormalizedAuthorizationRequest is identical (same protocol operation, same
   adapter configuration, same governed mapping state — the normalization
   invariant, §2)
→ evidence material is identical
→ canonical bytes are identical
→ evidence digest is identical
→ reference_id may be reused or freshly minted (§8's deliberate non-decision)
→ request_id: per operation-producer-and-execution-boundary.md §6, the
   producer runtime owns this identifier and basis-gateway never overwrites a
   caller-supplied value — whether the retry reuses the original request_id
   or mints a new one is a producer-runtime design choice this document does
   not resolve, but either choice is compatible with everything decided here
```

---

## 15. Alternatives and Rejected Designs

1. **Producer runtime owns all evidence construction** (including material and digest). Rejected:
   this would require the producer runtime to re-derive or duplicate normalization knowledge
   `basis-adapters` already has, risking exactly the drift `ecosystem-contract-inventory.md`
   already warns about for contracts implemented in more than one place, and it would let a
   compromised or divergent producer runtime construct evidence material that does not actually
   correspond to what the adapter normalized.
2. **`basis-adapters` owns all evidence-reference construction** (including `reference_id`,
   `adapter_source`, `redaction_classification`). Rejected: this would require
   `basis-adapters` to hold deployment configuration (for `adapter_source` selection under runtime
   identity) and deployment redaction policy (for `redaction_classification`), neither of which a
   pure, side-effect-free normalization library can observe without acquiring exactly the
   stateful, deployment-aware responsibilities `docs/contracts/adapter-contract.md` and
   `operation-producer-and-execution-boundary.md` §10–§11 already forbid it from acquiring.
3. **Adapter library owns semantic material and digest; producer owns reference assembly**
   (§7's adopted split). Selected: matches the ownership invariant (§2) precisely — each component
   owns exactly the facts it can observe or establish, and no component is asked to reach beyond
   that.
4. **Gateway regenerates adapter evidence.** Rejected: `basis-gateway` has no protocol-parsing
   capability (its own documentation already states this: "the gateway cannot independently
   confirm the truth of a producer's claim; it has no device-state or protocol-parsing capability")
   and regenerating evidence it cannot independently verify against protocol reality would create a
   false appearance of gateway-side verification that does not exist.
5. **Digest only raw protocol evidence** (Alternative A, §4). Rejected there; does not prove
   enough about normalization.
6. **Digest only normalized output** (Alternative B, §4). Rejected there; loses protocol-native
   traceability.
7. **Digest a combined versioned evidence profile** (Alternative C, §4). Selected.
8. **Use `reference_id` as the digest.** Rejected: `reference_id` is minted by the producer
   runtime (§8) independent of the material's content, while the digest must be a deterministic
   function of the material itself (§2's normalization invariant); collapsing the two would make
   `reference_id` non-deterministic (violating §8's own uniqueness-independence design) or make the
   digest dependent on when and by whom it was minted (violating the normalization invariant).
   They serve different purposes and must remain structurally distinct fields, exactly as the
   published schema already keeps them.
9. **Publish a schema before a reference implementation exists.** Rejected: `basis-schemas`'
   own governing principle (repeated throughout `ecosystem-contract-inventory.md` and ADR-0005)
   is that "implementation proves a stable shape" before publication; this document identifies a
   potential future schema need (§16) without publishing one, consistent with that discipline.
10. **Omit adapter evidence entirely.** Rejected: this would remove the one mechanism the
    ecosystem has for linking a decision back to the protocol-level facts that produced it,
    directly contradicting `operation-aware-trace-audit-evidence.md` §7's requirement that "audit
    should be able to link a decision to the adapter evidence that produced it."

The preferred design — adapter library owns semantic material plus deterministic digest, producer
runtime owns contextual reference assembly, gateway authenticates and admits but does not
reinterpret — is selected because it is the only alternative among the ten above that assigns every
construction step to the one component actually positioned to perform it correctly, without
requiring any component to acquire knowledge (deployment configuration, protocol-parsing capability,
redaction policy) it structurally lacks. The boundary is also the one most resilient to
compatibility drift: because `basis-adapters` computes the digest from facts it alone produces, a
future change to producer-runtime implementation details (how it mints identifiers, how it selects
`adapter_source` under a new deployment topology) cannot silently change what a digest attests to.

---

## 16. Schema Impact Assessment

**Resolved, not a gap.** A prior draft of this document listed "evidence-material profile
identifier" and "canonicalization profile/version" as potential future schema gaps. Both are now
decided (§4, §5, §9): `evidence_profile` (`basis-adapter-evidence-v1`) and
`canonicalization_profile` (`rfc8785`) are fixed literals carried **inside the digested evidence
material itself**, not merely alongside it. This resolves the identification problem without
requiring any `basis-schemas` change: a verifier that retrieves the evidence material (§6) reads
these fields directly from the artifact it is verifying, the same way it reads every other
evidence-material field. No immediate `basis-schemas` field is required solely to carry the profile
identifiers.

A **future reference-level profile field** — exposing `evidence_profile` and/or
`canonicalization_profile` directly on the published `AdapterEvidenceReference`, rather than only
inside the evidence material a verifier must separately retrieve — remains a distinct, deferred
possibility, useful only if a concrete consumer needs to know an artifact's profile without
retrieving and parsing the full material first (for example, a future conformance test kit
filtering references by profile version at scale). This document does not conclude such a field is
needed; it remains deferred unless and until a concrete consumer demonstrates the need, consistent
with `ecosystem-contract-inventory.md`'s own "implementation proves a stable shape" principle.

| Possible gap | Real producer | Real consumer | Stable ownership | Concrete scenario | Compatibility impact | Existing field insufficient because |
| - | - | - | - | - | - | - |
| Reference-level profile field (as distinct from the evidence-material-internal fields resolved above) | `basis-adapters` (implemented and released, `v0.2.0`'s `construct_adapter_evidence()`) | A future evidence-verification component; a future conformance test kit | `basis-architecture` (this document; a future ADR would formalize) | A consumer needs to filter or route references by declared profile without retrieving and canonicalizing the full evidence material first | Additive if introduced; existing `adapter-evidence-reference` fields are unaffected | The evidence-material-internal fields (§9) already resolve the *verification* need; this row records only the narrower, distinct *reference-level convenience* question, which this document does not find a concrete consumer for today. |
| Stored-evidence locator semantics | `basis-producer`'s own bounded, digest-addressed local retention exists today (Phase 2A/2B, implemented); a generalized, cross-deployment evidence-storage or evidence-service component remains undecided | `basis-console` (for operator display), a future audit/investigation surface | Not yet decided beyond the bounded producer-local case — §7 distinguishes bounded producer-owned retention (implemented) from a still-undecided generalized evidence-storage architecture | Resolving `reference_id` back to full, retained evidence material for operator review or forensic investigation, especially across independently-operated producer deployments | Additive if introduced, and gated on whether a generalized, cross-deployment persistence architecture is ever decided | The bounded reference implementation already resolves retrieval within one producer's own local retention; no existing contract defines a generalized, cross-deployment evidence-storage or retrieval locator — `adapter-evidence-reference.md` still explicitly disclaims this at the schema level ("does not define... evidence storage, evidence retrieval") |

**Conclusion, per gap:**

- **Evidence-material profile identifier and canonicalization profile/version**: resolved by this
  document, not a schema gap. Both are fixed, named decisions (`basis-adapter-evidence-v1`,
  `rfc8785`) carried inside evidence material itself; no `basis-schemas` field is required to carry
  them.
- **Reference-level profile field**: a potential future schema addition is identified but remains
  deferred, pending a concrete consumer — see above. This document records the possibility; it does
  not add a field to `adapter-evidence-reference` or any other published contract.
- **Stored-evidence locator semantics**: no schema change is justified at this time. A bounded,
  producer-local persistence mechanism exists and is implemented (`basis-producer` Phase 2A/2B), but
  no generalized, cross-deployment evidence-storage architecture exists for an interoperable locator
  to describe.

No schema is modified by this document. No JSON Schema field is defined here. `basis-schemas`
continues to publish stabilized contracts only after implementation proves a stable shape; this
document does not invent one ahead of that evidence.

---

## 17. Implementation Handoff

| Stage | Owning repository | Prerequisite | Scope | Explicit non-goals | Completion evidence |
| - | - | - | - | - | - |
| 1 — `basis-adapters` evidence-material profile and pure helper — **COMPLETE** | `basis-adapters` | This document's adoption (via its companion ADR) | Implement the evidence-material construction, canonicalization, and digest-computation helper this document assigns to `basis-adapters` (§4, §5, §7), only because this ADR assigns material construction and digest calculation to `basis-adapters` | Does not add producer authentication, a gateway client, `reference_id` minting, `adapter_source` selection, or `redaction_classification` assignment — all remain producer-runtime responsibilities per §7 | **Done.** Released in `basis-adapters` `v0.2.0`'s `construct_adapter_evidence()`: deterministic (same input → same output across repeated calls and process restarts), producing `evidence_digest`-shaped output matching the published `adapter-evidence-reference.evidence_digest_shape` |
| 2 — Conformance tests across all nine adapters — **COMPLETE** | `basis-adapters` | Stage 1 | Verify that the same semantic input produces stable canonical bytes and digest for every currently published protocol (REST, BACnet, Modbus, OPC UA, MQTT, DNP3, IEC 61850, KNX, Niagara) | Does not test producer-runtime or gateway behavior — scoped to the pure helper only | **Done.** `basis-adapters`' `tests/test_evidence.py` exercises all nine adapters (`TestProtocolProjection`), asserts RFC 8785 canonicalization against RFC 8785's own number-serialization and key-ordering test vectors (`TestCanonicalization`), and asserts digest determinism (`TestDigest`) |
| 3 — Operation-producer runtime architecture — **RESOLVED** | `basis-architecture`, across [ADR-0010](../adr/0010-establish-basis-producer-as-operation-producer-runtime.md), [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), and [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) — three Accepted ADRs, together fulfilling what this stage originally envisioned as one future document | Stage 1 exists in principle (the ownership split this document defines does not require Stage 1 to be complete before the producer-runtime architecture is drafted, but a drafted architecture should assume Stage 1's shape) | Repository placement, lifecycle, and authentication decision for the operation-producer runtime role — all now resolved rather than open: ADR-0010 establishes `basis-producer` as the permanent repository, ADR-0008 selects mutual TLS as the authentication profile, ADR-0009 defines the trusted-ingress topology | This stage, as originally scoped, did not itself select a producer-authentication mechanism; that selection was made by a dedicated ADR (ADR-0008) rather than by this stage's own architecture document, consistent with the non-goal as written | **Done.** ADR-0010, ADR-0008, and ADR-0009 are all recorded `Accepted`, each naming where the runtime lives (`basis-producer`) and how it authenticates (mTLS, via a trusted NGINX ingress) |
| 4 — Producer-side `AdapterEvidenceReference` assembly — **COMPLETE** | `basis-producer` (per Stage 3's resolution) | Stages 1 and 3 | `reference_id` minting, `adapter_source`, `normalization_version`/`mapping_version` carry-forward, `redaction_classification`, and request/correlation linkage (§7's producer-runtime column) | Does not implement digest computation (already done in Stage 1) or gateway admission logic | **Done, split across two merged PRs.** Retention — Phase 2A (`FilesystemEvidenceBlobStore`, digest-addressed, verified retrieval). Reference lifecycle and final assembly — Phase 2B (`EvidenceReferenceLifecycle.create_reference()`, assembling `AdapterEvidenceReference` against the published schema) |
| 5 — `basis-gateway` admission and audit refinement — **COMPLETE for its bounded, approved scope** | `basis-gateway` | Stage 4 | Only where current behavior is insufficient — `basis-gateway` already accepts and admits `adapter_evidence_reference` from a classified trusted producer (§operation-aware-endpoint.md); this stage addresses any gap Stage 4's reference implementation actually surfaces, not a speculative one | Does not regenerate or reinterpret adapter facts (§7) | **Done.** `basis-gateway` Phase 1B (1B.1, 1B.2, 1B.3) implements certificate-derived producer identity, exact admission matching, and audit recording for the bounded, approved mTLS scope; producer authentication was selected by Stage 3's ADR-0008, not by this stage |
| 6 — Reference implementation validation — **NOT YET STARTED (Phase 4, in `basis-producer`'s own numbering)** | Whichever repositories Stages 1–5 touched | Stages 1–5 | End-to-end validation that a real evidence artifact, constructed per this document, flows from adapter normalization through producer assembly to gateway admission without reinterpretation. Stages 1–5 are each independently complete; what remains is composing them into one live path: `ProtocolOperation` → `RestAdapter.normalize()` → evidence construction → producer retention/reference lifecycle → authenticated gateway submission → gateway/core disposition → STOP — this is `basis-producer`'s Phase 4. Broader cross-repository conformance, beyond this bounded composition, is `basis-producer`'s Phase 5 | Does not add execution or execution-evidence (out of scope of this entire document, per `operation-producer-and-execution-boundary.md`) | Not yet produced. A bounded, reproducible demonstration analogous to `basis-gateway`'s existing `demo/operation-aware/` remains the target completion evidence once Phase 4 composes the independently-implemented stages above |
| 7 — `basis-schemas` proposal | `basis-schemas` | Stage 6 | Only if implementation proves a real contract gap. The evidence-material profile identifier and canonicalization-profile/version gap this stage originally named is resolved, not a live gap (§16) — do not resurrect it. Future `basis-schemas` work should occur only if Stage 6 (Phase 4/5) integration demonstrates one of the genuinely remaining gaps §16 still names: the deferred reference-level profile field, or a generalized, cross-deployment evidence-storage/locator contract, if either is actually needed once real composition exists | Does not publish a schema speculatively ahead of a demonstrated need | A `basis-schemas` PR, reviewed under the existing ADR-0005-style readiness discipline, citing a specific implementation gap Stage 6 surfaced |

No version numbers, dates, or PR counts are assigned to any stage, consistent with `ROADMAP.md`'s
own convention that such detail belongs to each repository's own implementation plan once that
stage of work actually begins.

---

## Non-Goals

This document does not: modify any implementation repository; add Python code; add JSON Schema;
add a hashing library; add canonicalization code; add evidence persistence; create an evidence
database; create a producer runtime; create a new repository; select producer authentication; add a
gateway client; change gateway runtime behavior; change kernel behavior; add protocol execution; add
execution-result evidence; redefine identity evidence; treat `subject_hint` as authenticated
identity; claim a digest proves authenticity; claim adapter evidence proves authorization; claim
authorization proves execution; or promise a release number or schedule.

---

## Relationship to Other Documents

This document answers the decision gate `operation-producer-and-execution-boundary.md` §13 Stage 2
named and `basis-adapters`' merged `docs/operation-aware-handoff-alignment-plan.md` assessed:
whether `basis-adapters` needs additive surface to construct adapter evidence, and — going further,
as this document was chartered to do — exactly what that construction means, who performs each
step, and how it remains verifiable. It does not revisit or alter the roles, trust rules,
correlation model, or repository-ownership decisions `operation-producer-and-execution-boundary.md`
already established; it narrows one part of that document's Stage 2/3 gap into a concrete
architecture. It restates, without altering, `basis-adapters`' normalization-only contract, the
published `adapter-evidence-reference`, `redaction-classification`, and `contract-metadata`
contracts, `basis-core`'s existing `AdapterEvidenceReference` domain model, and `basis-gateway`'s
existing producer-trust classification and request-admission behavior. Where this document
introduces working vocabulary not yet in `docs/glossary.md` — *adapter evidence material*,
*evidence canonicalization profile* — it follows the same disclaimer convention every other
operation-aware architecture document in this repository already uses: naming a term here does not
promote it to canonical status ahead of the glossary review this document's companion ADR triggers
(see §16 of that ADR's own text).
