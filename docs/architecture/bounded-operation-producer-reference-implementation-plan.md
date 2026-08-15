# Bounded Operation-Producer Reference Implementation Plan

**Status: Implementation planning. ADR-0008 is Accepted; the bounded producer reference slice described here is not yet implemented.**

This document is not an ADR and does not reopen any decision ADR-0008 or ADR-0007 already settled. It translates those accepted decisions into a concrete, repository-aware implementation sequence a lead engineer can execute without inventing security-critical behavior during coding. It is the implementation source of truth for the first producer slice until the slice is complete; implementation PRs should refer back to it rather than re-deciding architecture.

**Companion documents:** [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md), [ADR-0007](../adr/0007-adapter-evidence-construction.md), [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md), [`adapter-evidence-construction-semantics.md`](adapter-evidence-construction-semantics.md), [`operation-producer-discovery-assessment.md`](operation-producer-discovery-assessment.md), [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`ROADMAP.md`](../../ROADMAP.md) (**Next Producer and Execution-Evidence Boundary**).

---

## 1. Status and Purpose

ADR-0008 authorizes, but does not implement, a bounded operation-producer reference slice: one REST operation, normalized by `basis-adapters`, evidenced deterministically, retained, referenced, submitted to `basis-gateway` over mTLS, evaluated by `basis-core`, audited, and stopped before execution. This document is the implementation plan for that slice. It:

- names the repository each change belongs in;
- selects a provisional location for the not-yet-existing operation-producer runtime;
- specifies the minimal evidence-retention mechanism (digest-addressed blob retention plus a post-retention `reference_id` → digest binding), `reference_id` minting, `adapter_source`, and `redaction_classification` behavior for the reference slice;
- specifies the mTLS termination topology *as a hard Phase 1A implementation gate*, the certificate identity extraction point, and the admission-configuration model;
- fixes the first slice's dual-authentication model: mTLS authenticates the **producer workload**, the gateway's existing bearer path authenticates the **authorization subject**, and the two are never conflated;
- defines how the new mTLS path coexists with `OPERATION_PRODUCER_SUBJECT_IDS`;
- defines the identifier-ownership model across producer, gateway, and kernel, including the evidence digest, the digest-derived storage key, and the reference-binding record;
- defines fail-closed behavior, a layered failure taxonomy (transport authentication vs. producer admission vs. authorization), and retry/idempotency posture;
- defines the test and conformance strategy and a five-phase implementation PR sequence;
- states explicitly what remains deferred.

It does not implement any of the above. No code, schema, certificate, configuration, or dependency is added by this PR.

---

## 2. Governing Accepted Decisions

The following are Accepted and not reconsidered by this document:

- mTLS is the first normative producer-to-gateway workload-authentication profile (ADR-0008).
- The producer workload identity is derived from exactly one eligible URI SAN on the validated client certificate; Common Name is never a fallback (ADR-0008, "Certificate identity profile").
- Admission matching is exact, case-sensitive string matching against deployment-controlled configuration; no prefix, suffix, substring, wildcard, regex, or case-insensitive matching (ADR-0008).
- Authentication and admission are two independent steps; a valid certificate does not imply admission (ADR-0008, "Trust anchor is not admission").
- Producer identity is distinct from authorization-subject identity and is never automatically reused as one (ADR-0008, "Producer vs. authorization subject").
- Evidence must be successfully retained before `reference_id` is minted and the reference assembled (ADR-0008, "Bounded reference implementation authorized"; restates ADR-0007's `reference_id` ownership).
- REST is the reference adapter for this slice (ADR-0008).
- The slice stops at the authoritative gateway disposition; no protocol execution occurs (ADR-0008).
- `basis-adapters` owns evidence-material construction, RFC 8785 canonicalization, and digest computation; the operation-producer runtime owns `reference_id` minting, `adapter_source`, `redaction_classification`, request/correlation linkage, and final `AdapterEvidenceReference` assembly (ADR-0007).
- `OPERATION_PRODUCER_SUBJECT_IDS` is retained, not removed or reinterpreted as mTLS (ADR-0008, "Existing allowlist transition").
- No new `basis-schemas` contract is assumed necessary merely because this slice is being planned (ADR-0008, "Machine-readable contract decision"; ADR-0007, "No `basis-schemas` change is made by this ADR").
- Permanent operation-producer-runtime repository placement is not decided by ADR-0008 and is not decided by this document either (ADR-0008, "Provisional implementation location"; boundary document §11).

This document treats every one of the above as a fixed constraint, not a topic for renewed debate.

---

## 3. Inspected Repository Baseline

Inspected locally-checked-out state, read-only, at the time this plan was authored. Consistent with `operation-producer-discovery-assessment.md`'s own findings for the six repositories it inspected on `8f2fb...`-equivalent commits of `basis-core`, `basis-gateway`, `basis-adapters`, `basis-identity`, and `basis-console` (unchanged since that assessment); `basis-schemas` has advanced one merged PR since that assessment (the ADR-0007 ownership-metadata correction, confirmed still merged at the SHA below).

| Repository | Branch | HEAD SHA | Working tree | Released version |
| - | - | - | - | - |
| `basis-architecture` | `docs/plan-bounded-operation-producer-reference-slice` | `409eb8328244ed860dbf2cc5207ea9a22c620003` | clean | N/A |
| `basis-gateway` | `main` | `81f72a841e1d947655f41f3e43dd0d5d559765b5` | clean | `0.2.0` |
| `basis-adapters` | `main` | `c4759a37d6e464bfc5ac97004c54f00afc7b45ca` | clean | `0.2.0` |
| `basis-core` | `main` | `7a924970eb12daad46cfe49eb86258ea59e92c9a` | clean | `0.2.1` |
| `basis-schemas` | `main` | `ec43a2f362ebabfb9a64cf1953a3343c24478e47` | clean | `0.2.2` |
| `basis-identity` | `main` | `2d9490fbb4018c7f5473e7add75ac4923ee66979` | clean | `0.1.0` |
| `basis-console` | `main` | `9c3fde8ed8e180467ec266a822a65768f187e8a5` | clean | `0.2.0` |

No `basis-deploy` repository exists. No demo/PoC/lab repository beyond `basis-gateway`'s in-repository `demo/operation-aware/` (gateway-only, no adapter or producer involvement) was found in the working environment. This is consistent with `operation-producer-discovery-assessment.md`.

Direct inspection performed for this plan (paths cited inline in §7–§15) confirmed, without contradiction, every claim the discovery assessment already made, plus one refinement: `basis-schemas`' `adapter-evidence-reference.yaml` `composition` block has been corrected (PR #22, merged into `ec43a2f`) to attribute `produced_by: operation-producer runtime` and a separate `evidence_material_and_digest_constructed_by: basis-adapters` field — the metadata/ADR-0007 conflict the discovery assessment flagged is resolved. No implementation repository changed as part of producing this plan.

---

## 4. Bounded Slice Definition

Restated from ADR-0008, unchanged:

```text
REST operation (local mapping fixture, no network target)
    → basis-adapters normalization (RestAdapter.normalize())
    → construct_adapter_evidence() (basis-adapters, ADR-0007 Stage 1 — already implemented)
    → canonical evidence bytes + separately returned SHA-256 evidence digest
    → derive storage key from the evidence digest (reference producer; new)
    → durably retain canonical evidence bytes under the digest-derived key (reference producer; new)
    → confirm blob retention succeeded
    → mint opaque reference_id (reference producer; new)
    → durably persist reference_id → evidence-digest/storage-key binding (reference producer; new)
    → confirm reference binding succeeded
    → assign adapter_source, redaction_classification (reference producer; new)
    → assemble AdapterEvidenceReference (reference producer; new)
    → establish mTLS producer connection to basis-gateway (reference producer + basis-gateway; new)
    → establish a separate bearer-authenticated authorization subject on the same request (§14)
    → gateway validates the producer certificate at the selected TLS boundary (basis-gateway; new)
    → gateway derives producer identity from exactly one URI SAN (basis-gateway; new)
    → gateway performs exact admission match (basis-gateway; new)
    → gateway authenticates the bearer subject independently (existing authenticate() dispatch)
    → gateway composes and submits to basis-core's existing operation-aware path (unchanged)
    → basis-core evaluates the authorization subject (unchanged)
    → gateway returns the authoritative disposition and records GatewayAuditEvent
      (unchanged, additively enriched — §19)
    → STOP — no protocol execution
```

Everything left of "derive storage key from the evidence digest" already exists and is not reimplemented. Everything at and right of "establish mTLS producer connection," through gateway composition and kernel invocation, already exists as *plumbing* for a bearer-authenticated trusted producer and needs to accept a second, independently authenticated trust path (§13) alongside — never instead of — the existing bearer-subject authentication, without altering kernel invocation or audit-emission code.

The two authentication facts on that one request answer two different questions and are never substituted for one another (§14):

- **mTLS client certificate** → *which admitted workload is producing and submitting this operation?*
- **`Authorization: Bearer <token>`** → *whose authority is `basis-core` evaluating for this operation?*

---

## 5. Provisional Component Placement

This is the most consequential open decision in the slice. Five candidate locations were evaluated against: architectural clarity; dependency direction; risk of role conflation; ease of later extraction; packaging implications; testability; risk that production code begins depending on reference code; whether the location implies permanent ownership; whether it requires circular dependencies; release/versioning consequences.

### Option 1 — Separately bounded component inside `basis-gateway`

**For:** Reuses the repository that will authenticate and admit the producer; a precedent already exists (`demo/operation-aware/`, a top-level directory outside `src/basis_gateway`, excluded from the shipped package) for non-shipped, demo-scoped code living alongside production source under one repository's release discipline.

**Against:** `basis-gateway` is a versioned, released, production distribution component (`v0.2.0`) with its own compatibility and release discipline. The reference producer is materially different in kind from `demo/operation-aware/`: it would hold a new class of secret no other code in that repository holds (a producer mTLS client private key), model new runtime responsibilities (adapter invocation, evidence retention, an outbound HTTP client authenticating *to* the gateway), and add new dependencies unrelated to serving the gateway's own API. Placing a credential-holding client of the gateway inside the gateway's own repository creates a live risk that a future contributor imports `basis_gateway` internals directly into producer code (collapsing the in-process/out-of-process trust boundary the ADR requires be crossed over the network) and a real risk that deployment tooling or a new contributor reads "lives in `basis-gateway`" as "is gateway functionality," which it structurally is not — it is a caller of the gateway's public HTTP boundary, nothing more.

**Disposition:** Rejected as the primary location; retained only as the fallback described in §5's decision below if the standalone-repository cost proves materially higher than anticipated during Phase 2.

### Option 2 — Component or integration harness associated with `basis-adapters`

**For:** Keeps producer code physically close to the evidence-construction library it depends on.

**Against:** Directly contradicts `basis-adapters`' own, currently released, non-negotiable non-responsibility: "no live protocol communication... adapters are libraries, not daemons," confirmed by direct inspection of `src/basis_adapters/evidence.py`'s own docstring ("does not call `basis-gateway`, does not authenticate a producer, and does not execute a protocol operation") and by the fact that `construct_adapter_evidence()` performs no network or filesystem I/O of any kind (§8). Even a nominally separate directory inside that repository would give the library's own release artifact a sibling that holds a network credential — the exact role conflation `operation-producer-and-execution-boundary.md` §11 alternative 1 already rejects for the library itself, and the task's own strong constraint ("do not expand the `basis-adapters` library itself into a daemon or credential-holding network service") applies with equal force to a same-repository sibling package, not only to the published library.

**Disposition:** Rejected.

### Option 3 — New dedicated repository, immediately

**For:** Cleanest possible boundary — no existing repository's charter is strained, no existing release process is entangled, no risk that production code in another component accidentally imports it (a separate repository cannot be `import`ed without an explicit, visible dependency declaration), trivially extractable later because it is already extracted, and the component can hold its own network credential without redefining what any existing repository is permitted to hold. Matches `operation-producer-discovery-assessment.md` §9 option 5's own finding: "highest [compatibility] of all options, since it requires no existing repository to acquire responsibility outside its current charter."

**Against:** Adds a repository, a nominal maintenance surface, and a naming question before a single line of producer code exists.

**Disposition — selected, with discipline.** A new repository is justified here specifically because Option 1 and Option 2 each fail at least one of the task's own justifying conditions: Option 1 risks misleading production ownership (a released, versioned distribution component acquiring a credential-holding network client under its own release discipline); Option 2 violates an already-accepted repository boundary (`basis-adapters` must never hold network or credential responsibility). No provisional location among the existing six repositories can host the reference slice without one of those two failures. Per the task's new-repository discipline: **repository naming is not finalized by this document** (a working name is used below only for concreteness); it is not published to any package index and carries no independent release/versioning ceremony during the bounded slice. The working/reference name used in the rest of this document, subject to change without requiring an architecture decision, is **`basis-operation-producer-reference`**.

> **The repository is provisionally scoped as a reference implementation. Its long-term role, name, release policy, and inclusion in the BASIS Core Services Distribution remain subject to the post-reference implementation decision gate (§30).**

This is a statement about *scope*, not about disposability. The repository is a real, git-tracked, tested reference implementation whose accumulated code and dependency graph are the primary input to §30's review — it is not scratch work to be discarded on completion, and it is equally not promoted, by this plan, to permanent Core Services Distribution membership.

### Option 4 — An existing demo/integration repository

No such repository exists (§3). Not applicable.

### Option 5 — Another existing repository whose current charter genuinely fits

`basis-core`, `basis-console`, `basis-identity`, and `basis-deploy` were each evaluated and rejected per the task's own strong constraints: `basis-core` must remain protocol-, transport-, and persistence-independent and must never hold a network credential (`docs/kernel-boundary-rules.md`); `basis-console` is bound by an already-implemented, already-honored console invariant (its own simulator self-classifies as preview-only — `src/basis_console/ui/views.py` docstring, confirmed by inspection) that a producer's authoritative submission role would violate outright; `basis-identity` has no implemented workload-credential-holding pipeline today (confirmed absent, §12) and its architecture role is identity federation, not operational-context assertion; `basis-deploy` does not exist and, per its own stated future scope, would not own runtime semantics even once it does.

**Decision:** A new, explicitly provisional, reference-scoped repository (`basis-operation-producer-reference`, working name), reversible by construction — deleting or renaming this repository has zero blast radius on any released component, since nothing in the five released repositories ever depends on it.

---

## 6. Repository Responsibility Matrix

No row below has ambiguous dual ownership.

| Capability | Owner |
| - | - |
| REST normalization | `basis-adapters` (existing) |
| Evidence-material construction | `basis-adapters` (existing) |
| RFC 8785 canonicalization | `basis-adapters` (existing) |
| Digest generation | `basis-adapters` (existing) |
| Digest-derived storage-key derivation | reference producer (`basis-operation-producer-reference`, new) |
| Evidence blob retention (durable, digest-addressed) | reference producer (new) |
| `reference_id` minting (post-retention) | reference producer (new) |
| `reference_id` → digest/storage-key binding | reference producer (new) |
| `adapter_source` assignment | reference producer (new) |
| `redaction_classification` assignment | reference producer (new, under fixed reference-deployment policy — §10) |
| Final `AdapterEvidenceReference` assembly | reference producer (new) |
| Producer client certificate / private key custody | reference producer (new) |
| Bearer subject credential custody (separate from the certificate) | reference producer (new) |
| TLS termination and certificate validation | `basis-gateway` at the topology Phase 1A selects (new) |
| Producer identity derivation (URI SAN) | `basis-gateway` (new) |
| Producer admission (exact match) | `basis-gateway` (new) |
| Producer trust classification (mTLS path) | `basis-gateway` (new, additive to existing) |
| Producer trust classification (legacy allowlist path) | `basis-gateway` (existing, unchanged) |
| Authorization subject identity | `basis-gateway`'s existing `authenticate()` dispatch (unchanged) — **never** derived from producer identity |
| Operation-aware request composition | `basis-gateway` (existing, unchanged) |
| Policy selection | `basis-gateway` (existing, unchanged) |
| Evaluation | `basis-core` (existing, unchanged) |
| Enforcement (HTTP disposition) | `basis-gateway` (existing, unchanged) |
| Audit (`GatewayAuditEvent`, `AuditEvidence`) | `basis-gateway` / `basis-core` (existing, additively enriched — §19) |
| Execution | **no owner in this slice** — explicitly out of scope |

---

## 7. Reference Producer Responsibilities

The bounded operation-producer runtime owns, for this reference slice only:

1. receive or originate one bounded REST operation (a scripted `ProtocolOperation`, not a live HTTP listener — §8);
2. invoke the existing `RestAdapter.normalize()`;
3. obtain the `NormalizedAuthorizationRequest` via the returned `AdapterResult`;
4. invoke `construct_adapter_evidence()`;
5. receive `ConstructedAdapterEvidence` — three *separate* values: the material, the canonical bytes, and the SHA-256 digest computed over those bytes (§9);
6. derive a storage key from the evidence digest and durably retain the canonical evidence bytes under it (§9);
7. confirm blob retention succeeded (§9);
8. mint the opaque `reference_id` only after blob retention succeeds (§10);
9. durably persist, and confirm, the `reference_id` → evidence-digest/storage-key binding before any reference is assembled (§9);
10. assign `adapter_source` (§10);
11. assign `redaction_classification` (§10);
12. carry `normalization_version`/`mapping_version` when available (today: absent — §10, per `basis-adapters`' own confirmed gap);
13. establish request/correlation linkage (§16);
14. assemble the final `AdapterEvidenceReference`;
15. authenticate its own workload identity to `basis-gateway` using ADR-0008 mTLS (§11, §15);
16. present a **separate** bearer credential establishing the authorization subject whose authority `basis-core` will evaluate (§14, §15);
17. submit the operation-aware request to `POST /v1/evaluate/operation-aware`;
18. receive the authoritative gateway disposition;
19. surface enough result information for tests/demo diagnostics (§24);
20. stop.

It does not own: policy evaluation; authorization semantics; gateway enforcement; identity-provider federation; protocol execution; execution-result evidence; console behavior; permanent evidence-store architecture. It never becomes the authorization subject on the operation-aware request it submits — the subject is whatever `basis-gateway`'s existing `authenticate()` dispatch establishes from the request's separate bearer credential. The producer's own certificate-derived workload identity is never promoted, defaulted, or degraded into a `subject_id` (§14).

---

## 8. REST Reference Operation

The slice uses the REST adapter's existing, already-released public surface exactly as released: `RestAdapter(mapping=RestMappingConfig.from_dict(...), context=AdapterContext(adapter_id=...))` and `.normalize(ProtocolOperation(protocol="rest", method=..., path=...))`, confirmed by direct inspection of `src/basis_adapters/rest/adapter.py`. `RestAdapter.normalize()` performs no network I/O — it string-matches `method`/`path` against a configured `RestMappingConfig` and returns an `AdapterResult`; nothing about this call transmits an HTTP request to any target. This is exactly the behavior the task asks for: "If the REST adapter API currently models an operation without transmitting it, use that behavior rather than adding HTTP execution."

**Selected operation:** a single, deterministic, boring read operation — `GET` against a mapped path that resolves to `action=read` (or the adapter's equivalent read-shaped verb), `resource_type=point` (or an equivalent already-mapped resource type), and a rendered `resource_id` from a path capture — using one of the repository's own already-checked-in fixture mapping configs under `basis-adapters/examples/rest/` (`mapping-minimal.example.json` or `mapping.example.json`) rather than inventing a new mapping file. A read operation is deliberately preferred over a write for the first slice: it minimizes any temptation to conflate "the gateway returned `ALLOW`" with "a write was executed" (nothing is executed either way, but a read-shaped example makes that non-claim easier to state plainly in demo narration) and keeps the reference scenario aligned with `basis-schemas`' own canonical `allow-basic` compatibility vector shape.

No dependency on an external Internet service exists or is introduced. No claim is made that a "REST target" was contacted — there is no target; the adapter never dispatches.

---

## 9. Evidence Retention and Retrieval

### Why the storage model cannot be `reference_id`-addressed

ADR-0008's accepted lifecycle is `retain evidence successfully → confirm retention → mint reference_id → assemble AdapterEvidenceReference`. An evidence store addressed *by* `reference_id` would therefore require the identifier in order to write the bytes that must already be written before the identifier may exist. That is circular, and this plan does not adopt it. Content identity and logical reference identity are kept separate throughout:

- **evidence digest** — the SHA-256 over the canonical bytes; identifies the *retained content*; computed by `basis-adapters`, not by the producer;
- **storage key** — derived deterministically from the evidence digest; where the bytes physically live;
- **`reference_id`** — an opaque, deployment-local *logical* reference minted only after the content is already durably retained;
- **reference binding** — the durable `reference_id` → digest/storage-key record that makes `reference_id` resolvable at all.

`reference_id` is never required to locate the bytes until that binding has been successfully persisted.

### Mechanism selected

**Local filesystem: a digest-addressed evidence blob plus a post-retention reference binding, both written with a flush/fsync/atomic-rename discipline.** The reference producer writes `ConstructedAdapterEvidence.canonical_bytes` to a configured local evidence root (default: a repository-local or XDG-data-style path, injectable for tests — never hardcoded to a production-shaped location), keyed by the digest `basis-adapters` already returned alongside those bytes.

### Required lifecycle

```text
basis-adapters constructs canonical evidence bytes
    → basis-adapters computes the SHA-256 evidence digest (returned separately — §9 "Digest semantics")
    → reference producer derives a storage key from that digest
    → reference producer durably retains the canonical evidence bytes under the digest-derived key
    → reference producer confirms blob retention succeeded
    → reference producer mints the opaque reference_id
    → reference producer durably persists the reference_id → evidence-digest/storage-key binding
    → reference producer confirms the reference binding succeeded
    → reference producer assembles AdapterEvidenceReference
    → submission may proceed
```

No step may be reordered, and no request reaches `basis-gateway` unless **both** blob retention and reference binding have succeeded.

### Recommended on-disk structure

The exact directory names are an implementation detail the reference repository may adjust to its own conventions; the *shape* is fixed:

```text
evidence-root/
    blobs/
        sha256/
            <digest>
    refs/
        <reference_id>
```

- the blob file contains the exact canonical evidence bytes, byte-for-byte, and nothing else;
- the reference record contains the minimum metadata needed to resolve `reference_id` to the retained blob and verify it — at minimum the `reference_id`, the digest algorithm, the digest value, and the digest-derived storage key;
- the reference record **must not** duplicate the canonical evidence bytes.

The blob layer is naturally deduplicating: two operations producing byte-identical evidence resolve to the same blob and two distinct `reference_id` records may legitimately bind to it. That is a property of content addressing, not an idempotency mechanism (§18).

**This is not a new shared schema.** The reference record is an internal reference-implementation persistence detail, owned entirely by the reference repository, published nowhere, and versioned by nothing (§20).

### Required filesystem durability procedure

Atomic rename guarantees *visibility and atomicity* — a reader never observes a half-written file. It does not, on its own, guarantee that the bytes survive a crash or power loss. This plan therefore does not treat a successful rename as proof of durable retention. The required procedure for the evidence blob is:

```text
create a temporary file in the target filesystem/directory
    → write the complete canonical bytes
    → flush userspace buffers
    → fsync the temporary file
    → atomically rename the temporary file to the digest-addressed blob path
    → fsync the containing directory
    → retention succeeds
```

Equivalent care applies to the reference-binding record:

```text
write a temporary reference record
    → flush
    → fsync the file
    → atomically rename to refs/<reference_id>
    → fsync the refs directory
    → binding succeeds
```

The temporary file must be created on the same filesystem as its final path (otherwise the rename is not atomic). The implementation may wrap both sequences in one small filesystem-store abstraction. **No database is introduced merely to obtain durability semantics** — the sequence above is achievable with stdlib `os`/`pathlib` alone.

### Scope of the retention guarantee

Successful retention, for this bounded slice, means exactly:

> The reference implementation completed its configured local-filesystem persistence protocol before issuing the logical reference.

It does **not** guarantee replication, indefinite retention, protection against later disk failure, backup, tamper resistance, evidence authenticity, or future availability. This is the honest reading of `adapter-evidence-construction-semantics.md` §6's own framing — "retention-before-minting proves the reference was honest at the moment it was created, not that the material remains available or unaltered indefinitely afterward" — and no stronger production durability language is used anywhere in this plan.

### Alternatives evaluated

| Option | Deterministic tests | Durability | Resolution by `reference_id` | Complexity | Air-gap | Dependency footprint | Cleanup | Verdict |
| - | - | - | - | - | - | - | - | - |
| In-memory dict | Yes | None — lost on process exit | Trivial | Lowest | Yes | None | Automatic | Useful for construction-logic unit tests only; does not satisfy "retained evidence" for the integration proof, since nothing survives beyond the immediate call — exactly the risk ADR-0008 and this task both flag |
| **Local filesystem, digest-addressed blob + reference binding (selected)** | Yes | Survives process restart when the documented flush/fsync/atomic-rename discipline is followed; bounded by local disk | Two-step: read `refs/<reference_id>` → read `blobs/sha256/<digest>` | Low | Yes | None (stdlib `os`, `pathlib`, `hashlib`) | `rm -rf` the configured evidence root | **Selected** |
| Local filesystem, single file named by `reference_id` | Yes | Same | Direct | Lowest | Yes | None | Same | **Rejected — circular.** Requires `reference_id` to store bytes that must be retained *before* `reference_id` may be minted (ADR-0008) |
| Embedded local database (SQLite) | Yes | Same as filesystem, plus transactional guarantees across the blob/binding pair | Requires a query | Moderate (schema, connection lifecycle) | Yes | stdlib `sqlite3`, but adds schema-migration surface | Delete one file | Rejected for this slice — the two-write ordering is already fail-closed without a transaction (a bound-but-blobless state is unreachable, since the blob is written first), so nothing here needs multi-row atomicity |
| Cloud object storage / RDS / any hosted service | N/A | N/A | N/A | High | **No** — violates offline/air-gap requirement | External network dependency | External | Explicitly rejected — the task and ADR-0008 both require the first slice work locally and offline |

### What is retained; what is authoritative

The retained artifact is `ConstructedAdapterEvidence.canonical_bytes` — the exact RFC 8785 canonical bytes `basis-adapters` produced, which are authoritative for digest recomputation. The pre-canonicalization `AdapterEvidenceMaterial` structure is not separately retained; it is fully reconstructible by re-parsing the retained canonical bytes as JSON, since canonicalization is a deterministic, information-preserving transform of the same fields.

### Digest semantics (what is and is not inside the canonical bytes)

`basis-adapters`' construction model is:

```text
AdapterEvidenceMaterial
    → RFC 8785 canonical bytes
    → SHA-256(canonical bytes)
    → EvidenceDigest
```

`construct_adapter_evidence()` returns a `ConstructedAdapterEvidence` carrying `material`, `canonical_bytes`, and `digest` as three **separate** values (confirmed by inspection of `src/basis_adapters/evidence.py`).

**The canonical evidence bytes contain** the canonicalized representation of the governed adapter evidence material, including its profile fields — `evidence_profile`, `canonicalization_profile`, and the protocol-specific governed evidence fields. Those profile fields *are* inside the digested bytes, which is precisely what lets a verifier know which canonicalization rules to reapply (`adapter-evidence-construction-semantics.md` §6).

**The canonical evidence bytes do not contain** their own SHA-256 digest, the `reference_id`, or any final `AdapterEvidenceReference` metadata. A digest cannot be a member of the byte sequence whose digest it is. Any earlier reading of this plan suggesting the digest is recoverable "by parsing the retained bytes" is corrected here: the digest is computed *over* those bytes and returned *alongside* them.

### Digest verification on retrieval

```text
resolve reference_id (read refs/<reference_id>)
    → obtain the expected digest and digest-derived storage key from that binding
    → retrieve the canonical evidence bytes at that storage key
    → recompute SHA-256 over those exact retrieved bytes
    → compare the recomputed digest against the expected digest
    → success, or integrity mismatch
```

The **expected** digest never comes from the bytes under verification — that would compare a value against itself and prove nothing. It comes from either:

- the stored reference binding (`refs/<reference_id>`), for local store-integrity checks; or
- `AdapterEvidenceReference.evidence_digest`, when validating the full operation artifact end to end.

Where the reference binding stores the expected digest, that copy is **internal lookup and integrity metadata** for the reference runtime. It is not part of the canonical evidence material, is not published, and carries no contract status.

This verification is exercised only by tests and by the local demo's diagnostic output (§24) — it is not exposed as a network API (§17).

### Failure behavior

- **Blob retention failure** (disk full, permission error, path unwritable, fsync error): retention fails; **no `reference_id` is minted**; no reference is assembled; no submission occurs. The producer fails closed for that operation.
- **Blob retention succeeds but `reference_id` generation fails**: no reference is assembled and nothing is submitted. A retained blob with no reference is inert — it is content in a content-addressed store that nothing points at.
- **Blob retention succeeds and `reference_id` is minted, but reference-binding persistence fails**: **no `AdapterEvidenceReference` is assembled and nothing is submitted.** The minted identifier is discarded.
- **Orphan identifiers have no architectural meaning.** An identifier minted internally and never published in an `AdapterEvidenceReference` has never been asserted to any component, appears in no request, no audit record, and no contract. It is not a leak, not a dangling reference, and requires no compensating cleanup protocol in this slice.
- **No request reaches the gateway** unless both evidence retention and reference binding have succeeded. This restates, without weakening, ADR-0008's retain-before-mint invariant.
- **Retrieval miss** (a test or diagnostic asks for a `reference_id` the store has no binding for): returns an explicit "not found" result, distinguishable from "binding found but blob missing" and from "blob retrieved with mismatched digest" — three different failure classes the retrieval interface must not collapse into one exception type.
- **Cleanup:** tests and local demo runs use a per-run temporary directory (created and torn down by the test harness), never a shared, accumulating production-shaped path — consistent with the "no cloud, no shared service" bias of this entire slice.

### Evidence-store abstraction

An internal store abstraction (a small Python `Protocol`) is defined only so a future, non-filesystem implementation can be substituted without touching reference-assembly logic. The plan fixes its **semantic responsibilities**, not a mandatory public API; exact Python method names, signatures, and error types are implementation details for Phase 2:

| Responsibility | Meaning |
| - | - |
| `retain_blob(digest, canonical_bytes)` | Durably retain the exact canonical bytes under the digest-derived storage key, following the flush/fsync/atomic-rename discipline above; return only on confirmed success |
| `bind_reference(reference_id, digest)` | Durably persist and confirm the `reference_id` → digest/storage-key binding record, with the same discipline |
| `resolve_reference(reference_id)` | Return the binding record (digest algorithm, digest value, storage key), or an explicit not-found result |
| `get_blob(digest)` | Return the retained canonical bytes for a digest, or an explicit not-found result |

An interface of the form `put(reference_id, bytes)` is **specifically excluded**: it reintroduces the circular dependency this section exists to remove. The store is never exposed over a network — a public, network-reachable evidence-retrieval service is explicitly out of scope for this slice, per the task's own "Explicit Non-Goals."

---

## 10. Reference ID and Evidence Metadata

### `reference_id`

**Mechanism:** UUIDv4 (`str(uuid.uuid4())`), matching `adapter-evidence-reference`'s own documented recommended convention, generated through an **injectable factory** — mirroring `basis-gateway`'s own already-released pattern for `trace_id`/`evidence_id` generation (`_default_trace_id()`/`_default_evidence_id()` in `src/basis_gateway/core/operation_aware_evaluator.py`, confirmed by inspection: a module-level default backed by `uuid.uuid4()`, threaded through an injectable `Callable[[], str]` parameter) so tests can supply deterministic values without touching production-path logic.

**Uniqueness:** deployment-local, per ADR-0007 — no ecosystem-global namespace is introduced.

**Timing:** minted only after §9's blob-retention step returns confirmed success — never before, never speculatively. Minting is itself not sufficient to assemble a reference: the §9 reference-binding write must also succeed before the `reference_id` may appear in an `AdapterEvidenceReference` or in any submission.

**Collision:** the reference implementation does not need a collision-detection mechanism beyond UUIDv4's own collision probability; ADR-0007 does not require one, and inventing a stronger guarantee for the reference slice would exceed what is decided.

**Content prohibition:** the identifier is opaque and carries no embedded secret, subject identity, certificate content, protocol payload, or authorization outcome — enforced by construction (a bare UUID string has no capacity to carry any of those).

### `adapter_source`

**Mechanism:** a deterministic combination of adapter implementation identity and deployment-instance label, computed by the reference producer from fields already available on its own configuration and on the constructed evidence material — for example `f"basis-adapters:{material.protocol}:{adapter_context.adapter_id}"` — consistent with `adapter-evidence-construction-semantics.md` §9's own description of this combination as "reference-assembly work, not evidence-material construction," and with its confirmation that this value is "constructible today from existing public fields... without any library change." The value is a stable, configured runtime identifier, not a credential, authorization claim, or trust assertion — it identifies which normalization path produced the evidence for interpretation purposes only.

### `redaction_classification`

**Selected default for this slice:** `reference_only`, using the closed five-value `redaction-classification` vocabulary `basis-schemas` already publishes and `basis-core`'s `AdapterEvidenceReference` (Pydantic) already validates as a closed enum — an invalid value fails construction closed, at the model layer, with no additional enforcement needed in the reference producer itself.

**Why `reference_only`:** the REST fixture's evidence material carries only `protocol`, `method`, `path` (REST's projection is deliberately narrow per ADR-0007 §4 — "REST is the deliberate exception") and no protocol-native identity-adjacent metadata (unlike BACnet's `device_id`, MQTT's `client_id`, etc.), so there is no field-level reason to select a stricter tier such as `never_display`; at the same time nothing about the fixture data is safe to display unredacted by default, so `safe_to_expose` would overstate what the reference slice actually knows about its own fixture data. `reference_only` — matching ADR-0007's own worked BACnet example — is the conservative, honest middle default.

**Ownership:** assigned by the reference producer under a fixed, configuration-declared reference-deployment policy (a single hardcoded/config-file value for this slice, not an inferred one), consistent with ADR-0007's explicit requirement that `redaction_classification` selection belongs to "the producer runtime or evidence boundary under deployment policy, not... the protocol-normalization library." No redaction *engine* is built — this slice selects one static value for its one fixture scenario.

**Fail-closed:** an attempt to assign a value outside the closed enum is rejected at `AdapterEvidenceReference` construction (Pydantic validation, already implemented in `basis-core`) before any submission occurs.

---

## 11. mTLS Termination and Certificate Identity

### Current gateway topology (confirmed by inspection)

`basis-gateway` ships as a FastAPI/Starlette ASGI application (`src/basis_gateway/main.py`), served via `uvicorn[standard]` (a released dependency, confirmed in `pyproject.toml`). No TLS certificate configuration (`ssl_certfile`, `ssl_keyfile`, `ssl_ca_certs`, or equivalent) exists anywhere in `src/basis_gateway/` today — a targeted search found none. `GatewayConfig` (`src/basis_gateway/config.py`) sources all settings from environment variables and defines no TLS-related field. This confirms the current application does not itself terminate TLS in any deployment this repository's own source anticipates; the existing assumption, unstated but consistent with the absence of any TLS configuration surface, is that TLS termination (if any) happens upstream of the ASGI process today. `cryptography>=42.0.0` is **already** a `basis-gateway` runtime dependency (confirmed in `pyproject.toml`), which materially changes the dependency calculus for certificate parsing (§25).

### Option A — Application server directly terminates mTLS

The ASGI process itself would need to: (1) be launched with `ssl_cert_reqs=ssl.CERT_REQUIRED` (or the server's equivalent) and a configured CA bundle for producer client certificates, and (2) expose the validated peer certificate to application code so the gateway's own admission logic can read the URI SAN. Item (1) is a well-understood capability of the underlying TLS stack Python's `ssl` module and `uvicorn` build on. Item (2) is the open technical question this plan does not resolve by assertion: whether the currently-selected ASGI server (`uvicorn`) exposes validated client-certificate data to the application layer through a documented, stable mechanism, or whether an ASGI server with more complete TLS-extension support (for example, one implementing the ASGI `tls` scope extension) would be required. This is a genuine implementation-time unknown, not a design decision — resolving it is **Phase 1A** (§26), a blocking spike that must complete before any admission logic is written.

### Option B — Trusted reverse proxy terminates mTLS

A local proxy (for example `nginx` or `stunnel`, configured to require and validate client certificates) terminates TLS and forwards a verified-identity assertion to the gateway over a channel the gateway can trust independent of the assertion's own content — for example, a loopback-only Unix domain socket the proxy alone can reach, with the gateway configured to reject any connection not arriving over that socket. This avoids depending on uncertain ASGI-server TLS-extension capability, at the cost of requiring the reference implementation to stand up and configure a second local process and to design the proxy→gateway trust channel carefully enough that a spoofed `X-Client-Cert`-style header is structurally impossible, not merely discouraged — the ADR is explicit that this boundary must be defined and protected, not casually assumed.

### Planning decision — the topology spike is a hard implementation gate

This is **not** a soft implementation curiosity to be resolved opportunistically while admission code is written. It is Phase 1A (§26), and it gates everything downstream of it:

> **Technical spike: prove where mTLS terminates and how validated client-certificate identity reaches the gateway admission layer.**
>
> **No producer-admission implementation may proceed until this spike resolves the topology.**

**Preferred outcome: Option A**, because it follows directly from ADR-0008's own framing of "gateway-derived identity from authenticated connection state" — the same process that terminates TLS is the process that derives identity, with no intermediate boundary to design and protect. But the preference is contingent, not assumed: it holds only if the spike demonstrably satisfies the criteria below.

#### What the spike must answer

For the currently used server stack (`uvicorn[standard]>=0.29.0` serving a FastAPI/Starlette ASGI app, confirmed in `pyproject.toml`):

1. Can Uvicorn be configured to **require and validate** client certificates?
2. After validation, can gateway application code obtain the peer certificate through a **documented, stable interface**?
3. Can it obtain the certificate's **URI SAN without trusting any caller-controlled HTTP field**?
4. Can tests exercise all four of: a valid client certificate; an invalid/untrusted certificate; a certificate with no URI SAN; a certificate with multiple URI SANs?
5. Does the working solution require Uvicorn configuration only, an ASGI TLS extension, a different ASGI server, or a trusted reverse proxy?

#### Outcome A — direct application termination works

If validated certificate data reaches gateway code through a documented and acceptable boundary:

- keep direct application mTLS termination;
- document the exact mechanism (which interface, which server version, which configuration flags) in the gateway's own implementation notes;
- proceed to Phase 1B: URI SAN extraction and admission implementation.

#### Outcome B — direct termination does not provide a safe application identity boundary

**Stop before implementing producer admission.** Do not casually add `X-Client-Cert`, `X-SSL-Client-*`, or any other certificate-identity header as an implementation shortcut — a header the application trusts without a proven, structurally enforced trust channel is exactly the spoofable identity assertion ADR-0008 forbids.

Before any proxy fallback is implemented, **this planning document must first be updated in a small follow-up architecture/planning PR** defining:

- which proxy terminates mTLS for the reference environment;
- how the proxy-to-gateway channel is isolated;
- why ordinary callers cannot reach that channel directly;
- how certificate-derived identity is conveyed across it;
- how spoofed identity headers are stripped or overwritten;
- what fact, precisely, the gateway trusts about the proxy;
- how local tests prove that trust boundary holds;
- whether the resulting topology still conforms to ADR-0008.

Only after that update is reviewed may proxy-backed producer admission implementation proceed. Option B remains scoped tightly when it is eventually taken (loopback-only or Unix-domain-socket-only proxy-to-gateway channel; no bare HTTP header carrying certificate claims across any network-reachable boundary under any circumstance).

#### Phase 1 completion gate

Phase 1 is **not** complete merely because certificate-parsing code exists. It is complete only when the implementation has demonstrated:

> A validated producer certificate can be mapped to exactly one URI SAN through a trust boundary that does not depend on caller-controlled request data.

Either way, the reference slice does not attempt to support every deployment topology — one topology is selected, proven, and documented as this slice's own, with permanent production ingress architecture left for later.

### Certificate identity extraction location

**Inside `basis-gateway`, in a new module analogous to `src/basis_gateway/auth/operation_producer.py`** (for example `src/basis_gateway/auth/producer_mtls.py`), never inside `basis-core`. This follows directly from `docs/kernel-boundary-rules.md`'s prohibition on protocol/transport/certificate concerns entering the kernel, and from ADR-0008's own explicit instruction that this document must "specify where URI SAN extraction belongs in the gateway architecture. Do not put certificate parsing inside `basis-core`." The module reuses `basis-gateway`'s already-released `cryptography` dependency for X.509 parsing.

---

## 12. Producer Admission Configuration

Candidate minimal configuration surface, following `GatewayConfig`'s existing `pydantic-settings`-based, environment-variable-sourced pattern (confirmed by inspection of `src/basis_gateway/config.py`, which already uses this exact pattern for `OPERATION_PRODUCER_SUBJECT_IDS`):

| Concept | Type | Default | Fail-closed behavior | Secret? |
| - | - | - | - | - |
| mTLS producer path enabled | bool | `False` | Disabled by default — a deployment that takes no action observes no behavior change, mirroring `OPERATION_AWARE_ENABLED`'s own existing default-off discipline | No |
| Producer trust-anchor location (CA bundle path) | file path | none (required if enabled) | Startup fails closed if enabled without a configured, readable trust anchor | No (public CA material) — but the *file itself* must never be assumed world-readable by convention; that is a deployment concern, not a schema one |
| Admitted URI SAN identities | set of exact strings | empty set | Empty set → no producer is admitted (mirrors `OPERATION_PRODUCER_SUBJECT_IDS`'s existing safe-default-empty pattern exactly) | No (identities are not secrets) |
| Per-identity enabled/disabled flag | bool per entry | n/a | A disabled entry is treated identically to an absent one | No |
| Server-side TLS material (gateway's own cert/key, if Option A is selected) | file paths | none (required if enabled) | Startup fails closed if enabled without both configured | **Yes** — private key material; must use whatever secret-handling convention the deployment already applies to other private keys (for example, the `basis-identity` RS256 signing key), not a new pattern invented for this slice |

This configuration governs producer authentication and admission **only**. It neither replaces nor relaxes the gateway's existing bearer-subject authentication configuration (`AUTH_MODE` and its mode-specific settings — §14), which remains required and unchanged: enabling the mTLS producer path never makes subject authentication optional, and a deployment with the producer path enabled but no working bearer configuration is misconfigured, not "producer-only."

Exact environment-variable names are deferred to Phase 1B implementation, which may not begin until the Phase 1A topology spike (§11, §26) resolves; this plan fixes the *shape* of the configuration (bool toggle, trust-anchor path, exact-match admitted-identity set with per-entry enable/disable, distinct from `OPERATION_PRODUCER_SUBJECT_IDS`) without inventing a name that would need to be revisited once the topology question resolves. No wildcard, prefix, or pattern-based admission configuration is permitted, per ADR-0008.

---

## 13. Legacy Allowlist Coexistence

Two distinct trust paths, represented internally as **separate authentication profiles producing a common trust result type** (not a single boolean, and not silently merged into one code path that cannot say which mechanism established trust):

```text
Legacy/bounded compatibility path
    bearer-authenticated subject (existing authenticate() dispatch)
        → exact subject-ID allowlist match (classify_operation_producer(), unchanged)
        → OperationProducerTrust(source=CONFIGURED_SUBJECT_ID_ALLOWLIST, ...)

ADR-0008 path (new)
    validated producer client certificate
        (validated at the TLS boundary the Phase 1A spike selects — §11)
        → certificate identity handed to gateway trust logic
        → exactly one eligible URI SAN derived
        → exact explicit admission match against §12's configuration
        → OperationProducerTrust(source=<new value>, ...)
```

`OperationProducerTrust.source` (`OperationProducerTrustSource`, currently a closed three-value enum: `CONFIGURED_SUBJECT_ID_ALLOWLIST`, `NOT_CONFIGURED`, `SUBJECT_ID_NOT_ALLOWED`) gains additive members for the mTLS path so the classification remains fully auditable and it is never ambiguous, from the recorded source alone, which mechanism established trust for a given request. This is additive to the existing closed enum, not a redesign of it.

**Which failures may become `OperationProducerTrust` values, and which may not.** The additive members cover only outcomes the *application* can actually observe — that is, cases where a certificate identity was successfully validated and handed to gateway trust logic, and the remaining question is admission. Candidates: an admitted identity; zero eligible URI SANs; more than one eligible URI SAN; a URI SAN that could not be deterministically extracted; a valid-but-unadmitted identity; an admitted-then-disabled identity. Exact naming is Phase 1B implementation detail, not fixed here.

Certificate-chain and handshake failures are **not** in that list. A certificate rejected by the TLS stack during the handshake produces no HTTP request, therefore no FastAPI route invocation, therefore no `OperationProducerTrust` object at all — there is nothing to classify. A value such as `MTLS_CERTIFICATE_INVALID` would name a state the application, under Option A, generally never reaches, and modelling it invites implementers to believe every certificate failure surfaces as an application-level trust result. It does not. §17's layered failure taxonomy governs which layer owns which failure.

The legacy allowlist is not removed, not reinterpreted as mTLS, and not elevated to normative status by this slice — exactly as ADR-0008 requires. Both paths may be enabled simultaneously in a deployment; a given request is classified by whichever authentication mechanism actually applies to its connection (bearer token → allowlist check; mTLS handshake → certificate-derived admission check). This reference slice exercises only the mTLS path end to end; the legacy path's own existing 23 test functions (`tests/test_operation_producer_trust.py`, confirmed by inspection) continue to cover it unchanged and are not touched by this work beyond the additive enum members above.

Deprecation of the legacy allowlist is explicitly not decided here, per ADR-0008's own deferred-decisions list, and is not proposed by this plan.

---

## 14. Dual Authentication and the Producer Trust Internal Model

### 14.1 The two identities on one request — decided, not deferred

The published operation-aware request requires `subject_id`, and ADR-0008 explicitly forbids treating the producer certificate identity as the authorization subject. The first reference slice therefore needs an explicit source of authorization-subject identity, and this plan fixes it rather than deferring it to implementation:

> **The first reference slice uses mTLS to authenticate the operation-producer workload, and the gateway's existing bearer-token authentication path to authenticate the authorization subject.**

```text
mTLS client certificate
    → producer workload identity
    → URI SAN
    → exact producer admission

Authorization: Bearer <token>
    → existing gateway bearer authentication (auth/runtime.py authenticate())
    → authorization subject
    → subject_id / roles / attributes
    → basis-core evaluation
```

This is not redundant authentication. The two credentials answer different questions:

- **mTLS answers:** which admitted workload is producing and submitting this operation?
- **Bearer authentication answers:** whose authority is `basis-core` evaluating for this operation?

### 14.2 Identity and trust model

| Fact | Source | Meaning |
| - | - | - |
| Producer workload identity | mTLS client certificate URI SAN | Which admitted workload submitted the operation |
| Authorization subject identity | Existing bearer-token authentication (`AUTH_MODE`-selected verifier) | Whose authority `basis-core` evaluates |
| Producer admission | Gateway exact-match URI SAN configuration (§12) | Whether that workload may assert producer-owned context |
| Authorization decision | `basis-core` | Whether the subject may perform the requested operation |

**Producer admission never supplies `subject_id`.** No row above feeds the row below it. An admitted producer with no valid bearer subject gets no authorization evaluation at all; a valid bearer subject arriving from an unadmitted or uncertified producer may proceed as an ordinary caller but may not assert producer-owned context (§17).

### 14.3 Required invariant: the two identities are deliberately different

The first reference slice must intentionally demonstrate:

> `producer identity != authorization-subject identity`

Tests and the local demo use clearly distinct synthetic identities. These literal values are illustrative and are **not** standardized by this document; the URI namespace remains deployment/test defined, consistent with ADR-0008:

```text
producer workload:
    spiffe://example.test/basis/reference-producer-01

authorization subject:
    service-maintenance-operator-01
```

A positive integration test asserting `producer URI SAN != bearer subject_id` while still producing a valid authorized request is a first-class conformance requirement of this slice (§23).

### 14.4 Selected bearer mechanism for the reference slice

**Selected: `AUTH_MODE=basis_local_token`, with a locally generated RS256 key pair and a locally issued BASIS-local identity token.**

Rationale, from direct inspection of `basis-gateway`:

- `GatewayConfig.auth_mode` (`src/basis_gateway/config.py`) offers exactly two existing bearer modes: `oidc` (default) and `basis_local_token`. No third mechanism is invented by this plan.
- The `oidc` mode requires `OIDC_ISSUER`/`OIDC_JWKS_URI` and a reachable provider to verify a token. The reference slice must not depend on an external, Internet-accessible identity provider merely to obtain a subject credential, so `oidc` is rejected for the first slice's default configuration.
- `basis_local_token` verifies RS256-signed BASIS-local identity tokens against public keys supplied directly as configuration (`BASIS_LOCAL_TOKEN_PUBLIC_KEYS_JSON`), performing **no** network I/O and no JWKS fetch (`auth/runtime.build_basis_local_token_trust_config`, confirmed: "This function performs no I/O").
- This is already the gateway's own deterministic, offline test/demo bearer path: `demo/operation-aware/run_demo.py` generates an ephemeral in-memory RSA key, issues a signed BASIS-local identity token (`issue_demo_token()`), and submits it through the real `Authorization: Bearer ...` header against the real `authenticate()` dispatch — the same shape `tests/test_auth_mode_evaluate.py` uses. Reusing it adds no gateway capability, no new dependency, and no new trust mechanism.
- It does not require `basis-identity` to be deployed. `basis-identity` is the eventual issuer of BASIS-local tokens in production, but the gateway verifies them from configured public keys alone, so the reference slice can issue its own test-only token without depending on that repository (consistent with §27's "no `basis-identity` change anticipated").

The reference producer therefore holds two credentials of different kinds: an mTLS client certificate and private key (workload identity) and a bearer token (subject identity). Both are test-only fixtures generated offline (§22).

### 14.5 Rejected: any producer-as-subject substitution

The following are **not** permitted anywhere in this slice, and any earlier suggestion of them is withdrawn:

- using the producer URI SAN as a placeholder authorization subject;
- using it as a degenerate or "for this slice only" subject;
- deriving `subject_id` automatically from an admitted producer identity;
- treating an admitted producer as a substitute for the required bearer-authenticated subject.

Each would violate ADR-0008's "Producer vs. authorization subject" separation — "a producer does not become authoritative for subject identity merely because its mTLS connection is trusted." The question is closed for this slice, not deferred.

### 14.6 Future scope, not this slice

A future architecture phase may decide how machine subjects, workload subjects, subject-less producer operations, delegated identities, and `basis-identity` workload credentials should interact with producer authentication. None of those questions need to be solved here. For this slice, **mTLS producer + bearer subject is the fixed model.**

### 14.7 `OperationProducerTrust` refinement

**Current model (confirmed by inspection):** `OperationProducerTrust` is a frozen dataclass with `status` (`TRUSTED`/`UNTRUSTED`), `source` (the enum discussed in §13), `authorization_subject_id` (always present), and `operation_producer_subject_id` (present only when trusted). Its own docstring already states the two ID fields are "equal in this PR's only supported trust mechanism... but that equality is an implementation detail of the current transport, not a statement that the two concepts are the same fact" — the model already anticipates a second mechanism arriving.

**Assessment:** the current model is insufficient, without unsafe ambiguity, for a second, independently-authenticated trust mechanism, because `operation_producer_subject_id` conflates "the identity that established trust" with "a bearer subject ID" — an mTLS-derived URI SAN is not a subject ID and forcing it into that field would misname what it is.

**Smallest additive change:** introduce a producer-identity field distinct from `operation_producer_subject_id` — for example `producer_workload_identity: str | None` (the URI SAN when the mTLS path establishes trust; `None` on the legacy path) — plus retain `source` (§13) as the field that records *which* mechanism produced the classification. `authorization_subject_id` remains unchanged in meaning (the bearer-authenticated subject, which under §14.1 is always present on a well-formed reference-slice request) and, critically, is **not** automatically populated from `producer_workload_identity` — preserving ADR-0008's producer/subject distinction at the data-model level, not merely in prose.

**Scope of what this model represents.** `OperationProducerTrust` may be extended to represent: the producer workload identity; the authentication/admission profile that applied; the admitted/unadmitted classification; the admission source; and coexistence with the legacy allowlist. It is **not** required to represent TLS failures that occur before an HTTP/application request exists — under the topologies §11 considers, a handshake-rejected certificate produces no request, no route invocation, and therefore no trust object to populate (§13, §17). The exact internal enum vocabulary remains a `basis-gateway` implementation detail; what this plan fixes is that the model does not claim to be a universal ledger of certificate outcomes.

**No public schema change required.** `OperationProducerTrust` is an internal `basis-gateway` dataclass, not a published contract; this refinement stays entirely inside `src/basis_gateway/auth/`. Gateway ownership of trust classification, and the producer/subject distinction, are both preserved unchanged.

---

## 15. Gateway Client and Submission

The reference producer's gateway client requirements, informed by (but not dependent on) `basis-console`'s existing `GatewayClient` (`src/basis_console/gateway/client.py`, confirmed by inspection to use `httpx.Client` with injectable transport for tests — a directly reusable *pattern*, not a shared dependency, since the producer must never depend on `basis-console`):

- gateway base URL (configured, injectable for tests);
- client certificate + private key (for the Option A/mTLS path — `httpx.Client(cert=(certfile, keyfile))`, an already-supported `httpx` capability requiring no new dependency beyond `httpx` itself, which is not currently a `basis-adapters` dependency but is a reasonable, minimal addition to the new reference-producer repository only — never to `basis-adapters`);
- CA/trust bundle for validating the gateway's own server certificate (`httpx.Client(verify=ca_bundle_path)`);
- a **separate** bearer credential for the authorization subject (§14.4), supplied as an ordinary `Authorization: Bearer <token>` header on the submission — configured and injected independently of the client-certificate material, never derived from it, and never logged;
- timeouts (explicit, finite — no indefinite hang on an unreachable gateway);
- the existing `POST /v1/evaluate/operation-aware` endpoint, unchanged;
- serialization of the existing `operation-aware-decision-request` contract — no new envelope, and specifically **no** new request-body field carrying producer identity (that identity comes from the transport, §20);
- deterministic error handling distinguishing connection failure, TLS handshake failure (transport-layer — the request never reached the application, §17), HTTP-level rejection (4xx, including bearer-authentication failure and producer-admission rejection as distinguishable cases), and a parsed governed disposition (§17, §18).

The two credentials are configured through two independent settings and are never defaulted from one another: a client certificate does not imply a token, and a token does not imply a certificate. A misconfiguration that supplies only one is a producer-side startup error, not a request that gets attempted anyway.

No general-purpose SDK is built. This is a narrow, single-purpose client scoped to exactly the one endpoint this slice needs.

---

## 16. Identifier and Correlation Model

| Identifier / record | Owner | Created when | Purpose | Persisted? | Externally visible? | Idempotency key? |
| - | - | - | - | - | - | - |
| **SHA-256 evidence digest** | `basis-adapters` (computed inside `construct_adapter_evidence()`) | At evidence construction, over the canonical bytes — returned separately from them (§9) | Content identity and integrity check for the retained canonical bytes | Yes — inside the reference-binding record, and published on `AdapterEvidenceReference.evidence_digest` | Yes, on the submitted reference | **No** — identical evidence legitimately recurs across independent operations |
| **Digest-derived blob storage key** | Reference producer (derived deterministically from the digest) | Immediately before blob retention | Locates the retained canonical bytes without needing `reference_id` | Yes — it *is* the on-disk path (`blobs/sha256/<digest>`) | No — purely local to the producer's store | **No** |
| **Opaque `reference_id`** | Reference producer | Only after blob retention is confirmed (§9, §10) | Deployment-local logical reference to retained evidence | Yes — as the `refs/<reference_id>` record's name | Yes, inside `adapter_evidence_reference` | **No** (§18) |
| **Reference-binding record** (`refs/<reference_id>`) | Reference producer | After `reference_id` is minted, before any reference is assembled | Resolves `reference_id` → digest algorithm, digest value, storage key; supplies the *expected* digest for verification | Yes — this is its entire purpose | No — internal reference-runtime persistence detail, not a published contract (§20) | **No** |
| Producer operation/request ID (`request_id`) | Reference producer (or absent) | At submission composition time | Producer-side correlation of one logical operation | Only in producer diagnostics/logs | Yes, on the request/response bodies it already appears on today | **No** — this plan explicitly does not call it an idempotency key (§18) |
| Gateway transport correlation ID | `basis-gateway`'s `CorrelationMiddleware` | Unconditionally, per HTTP request; caller-supplied `X-Correlation-ID` is ignored by existing, documented policy | Per-request transport correlation | Yes, into `GatewayAuditEvent` | Yes (response header) | **No** |
| Kernel trace ID (`trace_id`) | `basis-gateway`'s injectable factory (default `uuid.uuid4()`), embedded by `basis-core` | Immediately before kernel invocation | Links a decision to its evaluation trace | Yes, into `EvaluationTrace` / `GatewayAuditEvent` | Yes | **No** |
| Kernel/gateway audit evidence ID (`AuditEvidence.evidence_id`) | `basis-gateway`'s injectable factory, embedded by `basis-core` | Immediately before kernel invocation | Identifies the audit-evidence artifact; referenced (never embedded) from `GatewayAuditEvent.audit_evidence_id` | Yes, by definition | Yes | **No** |
| Policy bundle identifier/version | `basis-gateway`'s existing policy loader, unchanged | At startup | Identifies which bundle produced a decision | Yes, into `EvaluationTrace`/audit where already recorded today | Yes | **No** |

**Neither the evidence digest nor `reference_id` is an idempotency key.** The digest is a content identity — two independent operations that happen to normalize identically produce the same digest and the same blob, and that recurrence is correct, not a duplicate submission. `reference_id` is a fresh opaque value per assembled reference. Neither is used for deduplication, replay rejection, or at-most-once submission, because no such behavior exists in this slice (§18).

Mutability: none of the above is mutated after creation. The blob and the binding record are write-once; the reference implementation never rewrites either in place.

No new identifier collapses two of these. The reference producer's own `request_id`/`reference_id` are never silently reused as the gateway's correlation ID, and vice versa — this plan is compatible with the gateway's existing "generate correlation ID unconditionally, ignore caller header" policy (confirmed unchanged, §11) precisely because it never asks that policy to change: the producer's own identifiers travel *inside* the request body/reference object, never as a competing `X-Correlation-ID` header. No new correlation model or schema field is required for producer-to-gateway correlation; the existing `request_id`/`correlation_id` fields already published on `operation-aware-decision-request` and `adapter-evidence-reference` are sufficient for this slice's scope. This is stated as a finding, not assumed: if Phase 3 implementation discovers a genuine gap (for example, a need to correlate a specific mTLS connection/handshake event with a specific evidence submission for diagnostics), that is a new planning gap to raise explicitly, not a silent reuse of an existing, semantically different field.

---

## 17. Failure and Fail-Closed Semantics

Failures in this slice belong to distinct layers, and the plan does not permit them to be collapsed. In particular, a TLS-layer rejection is not an authorization outcome, and an unadmitted producer is not a policy denial.

### 17.1 Failure layers

**Layer 0 — Producer-side, before any request exists.** Normalization, evidence construction, blob retention, and reference binding. Nothing has been transmitted; failure means no request is ever made.

**Layer 1 — TLS transport authentication.** Connection failure; TLS handshake failure; invalid certificate chain; untrusted certificate; expired or not-yet-valid certificate; TLS negotiation failure. Expected behavior: the request does **not** reach normal gateway HTTP processing; no application-level producer trust object is required or produced; the condition surfaces through TLS/server/client diagnostics; no producer-only context is parsed, because no request body is parsed at all.

**Layer 2 — Certificate identity derivation and producer admission.** Reached only after the gateway's selected topology (§11) has handed a successfully validated certificate identity to gateway trust logic. Zero eligible URI SANs; more than one eligible URI SAN; a URI SAN that cannot be deterministically extracted; a URI SAN that is not admitted; an admission entry that is disabled or revoked. These are the appropriate candidates for application-level producer admission/trust results (§13, §14.7), where the selected topology permits the application to observe them.

**Layer 3 — Authorization.** Reached only after producer authentication, producer admission, bearer-subject authentication, and structural request validation have all completed. `basis-core` returns `DENY`; evaluation fails; policy is not applicable. These must remain distinct from Layers 1 and 2 in code, in diagnostics, and in the demo's output (§24).

### 17.2 Failure taxonomy

| Condition | Layer | Required behavior |
| - | - | - |
| REST normalization fails (`AdapterResult.success is False`) | Producer, pre-evidence | No evidence construction, no retention, no reference, no submission — existing `basis-adapters` fail-closed rule, unchanged |
| Evidence-material construction, canonicalization, or digest failure | Producer, pre-retention | No retention, no reference, no submission (ADR-0007 "required evidence-construction failure") |
| Evidence blob retention failure | Producer persistence | **No `reference_id` is minted**; no reference; no submission (ADR-0008's retain-before-mint invariant) |
| `reference_id` generation fails after successful blob retention | Producer persistence | No reference assembled, no submission; the retained blob is inert |
| Reference-binding persistence failure | Producer persistence | **No `AdapterEvidenceReference` is assembled and nothing is submitted**; the minted identifier is discarded and has no architectural meaning |
| `AdapterEvidenceReference` assembly fails schema validation | Producer, pre-submission | No submission |
| Producer holds only one of the two required credentials (cert without token, or token without cert) | Producer configuration | Startup/configuration error; the submission is not attempted (§15) |
| TLS handshake or certificate-chain failure (untrusted, expired, wrong CA, negotiation failure) | **Transport authentication** | **The HTTP request does not enter the normal producer-admission path.** No `OperationProducerTrust` is produced; the producer observes a local TLS failure, not an authorization result |
| Gateway unreachable / connection refused / timeout | Transport | No disposition received; producer surfaces submission failure; nothing is treated as authorized |
| Zero eligible URI SANs, or more than one | Identity derivation | Fail closed. Identity derivation fails outright; no fallback, and Common Name is never consulted |
| URI SAN cannot be deterministically extracted | Identity derivation | Fail closed |
| Valid URI SAN but not admitted, or admission entry disabled | Producer admission | Reject the producer-owned context/request as designed — `UNTRUSTED` classification; the request may proceed only as an ordinary (non-producer) caller if it carries independent, valid bearer authentication, and any producer-only field it asserts is rejected |
| Ordinary (bearer-only, non-admitted) caller asserts producer-only fields | Producer admission | Existing `UntrustedOperationProducerContextError`, raised before any other composition step — unchanged |
| Bearer token missing, malformed, expired, or failing verification | **Subject authentication** | No authorization evaluation occurs — even when the producer's mTLS authentication and admission both succeeded. `basis-core` is never invoked |
| Bearer subject valid **and** producer valid/admitted | Authentication complete | Continue to operation-aware request composition |
| Malformed operation-aware request body | Structural validation | Existing structural 400 rejection at the gateway's schema boundary — unchanged |
| `basis-core` returns `DENY` | Authorization | Stop. No execution (none exists in this slice regardless) |
| `basis-core` evaluation fails (`evaluation_status: failed`) | Authorization | Stop. Never treated as an implicit allow |
| `basis-core` returns `not_applicable` | Authorization | Stop. A distinct kernel outcome, never collapsed into `ALLOW` |
| `basis-core` returns `ALLOW` | Authorization | Record the authorization result and stop. Still no execution — this is the slice's own hard boundary, not merely one branch among several |

The bounded slice never executes an operation under any row above. This restates, without weakening, `operation-producer-and-execution-boundary.md` §5's own invariants.

---

## 18. Retry and Idempotency Posture

ADR-0008 intentionally deferred a complete retry/idempotency model; this slice adopts the smallest safe posture consistent with that deferral:

- **No automatic retries** after an ambiguous gateway submission outcome (timeout, connection reset) unless a future implementation can demonstrate the retry is unquestionably safe for this specific no-execution slice — for the reference implementation, the default is: report the ambiguous outcome and stop, do not retry automatically.
- An explicit, caller-initiated retry (a human re-running the demo, or a test deliberately exercising the retry path) may produce a **repeated authorization evaluation** — this is expected and safe, because no execution exists to duplicate.
- `request_id`, `reference_id`, and the SHA-256 **evidence digest** are explicitly **not** called idempotency keys anywhere in this plan or its implementation, because no idempotency behavior (deduplication, replay rejection, at-most-once submission) actually exists — using the term for any of them would misrepresent what they do. The digest's content-addressing means two byte-identical evidence artifacts share one blob; that is storage deduplication of *content*, never deduplication of *submissions* (§9, §16).
- The reference producer logs (locally, for diagnostics only — §19) enough information on every submission — the `reference_id` used, the `request_id` used, the gateway's HTTP status, and (when available) the parsed disposition — that a human reviewing two submissions can tell whether they were the same logical operation retried or two independent operations, without the system itself claiming to enforce that distinction.
- No broader idempotency/replay protocol is created in this phase, consistent with the identity-to-operation interoperability roadmap's own Phase 5/Phase 8 deferral of this exact question at the ecosystem level.

---

## 19. Audit and Observability

`basis-gateway`'s existing `GatewayAuditEvent`/`basis-core`'s existing `AuditEvidence` already prove: the authenticated caller (today: bearer subject); the composed action/resource; the kernel's outcome; the enforcement disposition; the correlation ID; the evaluation trace reference. Confirmed unchanged and untouched by this plan.

What this slice needs, additively, for the mTLS path to be observably distinguishable from the legacy path in diagnostics: the producer authentication mechanism (`bearer+allowlist` vs. `mtls`); the derived producer URI SAN (when the mTLS path applies); the admission result; and the authorization subject recorded as a **separate, independently sourced field** so that a reader of any single audit record can see both identities and see that they differ (§14.2, §14.3). An audit record that shows only one identity, or that shows the producer identity in the subject position, fails this requirement. These additions belong in **gateway-internal diagnostic/audit context** (extending whatever internal fields `GatewayAuditEvent` construction already threads through, not necessarily every field on the *published* schema) — per the task's own bias, "determine whether that is actually necessary for the first slice or can remain internal diagnostic context." This plan's finding: it can remain internal for the reference slice. No certificate secret, private key, or raw certificate bytes are ever logged; only the derived URI SAN string (already, by ADR-0008's own design, a non-secret, deployment-registered identity string) and the admission boolean/reason are recorded. Gateway audit proves an authorization lifecycle and nothing about protocol execution — this plan does not claim otherwise anywhere, including in the local demonstration (§24).

---

## 20. Shared Contract Assessment

| Boundary item | Classification | Rationale |
| - | - | - |
| mTLS certificate identity profile (URI SAN as identity) | Existing contract sufficient (ADR-0008 itself, not a schema) | ADR-0008 already fixes the profile at the architecture level; nothing machine-readable is missing |
| Producer admission configuration | New repository-local configuration (`basis-gateway`) | Deployment-controlled, not cross-repository — §12 |
| Producer trust result (`OperationProducerTrust`) | Internal implementation detail | Not a published contract; additive dataclass fields only — §14.7 |
| Evidence store interface (blob retention + reference binding) | New repository-local implementation detail | Lives entirely inside the new reference-producer repository; no other component calls it |
| Digest-addressed blob layout (`blobs/sha256/<digest>`) | Internal reference-runtime persistence detail | A filesystem layout chosen by one runtime for its own local store; no second component reads it, so there is nothing for two components to agree on — §9 |
| Reference-binding record (`refs/<reference_id>`) | Internal reference-runtime persistence detail | Internal lookup/integrity metadata; explicitly **not** part of the canonical evidence material and not published — §9 |
| Evidence digest itself | Existing contract sufficient (`evidence-digest` on `adapter-evidence-reference`) | Already published, already carries algorithm + value; the corrected digest semantics (§9) change how the reference implementation *obtains* the expected digest, not what is transmitted |
| Reference-minting mechanism | Internal implementation detail | UUIDv4 generation, injectable factory — no cross-repository contract needed |
| Gateway request envelope | Existing contract sufficient (`operation-aware-decision-request`) | Already published, already sufficient per ADR-0008 and the discovery assessment's own confirmation — §16 |
| Producer workload identity on the wire | Existing transport identity sufficient | The URI SAN arrives on the validated TLS certificate, not in the request body. Adding a duplicate producer-identity request-body field would create a caller-controlled value that could contradict the transport fact — the exact spoofing surface ADR-0008 forbids |
| Authorization subject on the wire | Existing bearer authentication sufficient | `AUTH_MODE`-selected verifier, unchanged; `subject_id` already required by the published operation-aware request — §14.4 |
| Authentication-mechanism identifier (`source`/mechanism label) | Internal diagnostic field | §19 — kept internal for this slice |
| Diagnostic/audit fields (URI SAN, admission result) | Internal diagnostic field | §19 |

**Reassessed after the corrections in §9, §14, and §17: the answer is unchanged.** For every candidate above, "why must two or more independently versioned components share this machine-readable shape now?" still has no strong answer — every item is either already covered by an existing published contract, or entirely internal to one repository's own implementation. **No `basis-schemas` change is required to build this slice.**

Specifically, none of the six corrections creates a contract gap:

- the digest-addressed blob store and the reference-binding record are internal reference-runtime persistence details, read by exactly one runtime;
- the corrected digest-verification flow consumes an already-published field (`AdapterEvidenceReference.evidence_digest`) plus a purely local binding record;
- the dual-authentication model composes an **existing** transport identity, an **existing** bearer authentication mode, and the **existing** operation-aware request contract — it publishes nothing new;
- the layered failure taxonomy and the `OperationProducerTrust` scope correction are both internal to `basis-gateway`;
- the Phase 1A topology gate produces an implementation finding, not a shape.

Accordingly this plan publishes **no** filesystem persistence schema, **no** TLS identity schema, and **no** duplicate producer-identity request-body field. This restates, without weakening, both ADR-0008's and ADR-0007's own findings. If Phase 5 (§26) surfaces a genuine cross-repository shape two independently versioned components must agree on byte-for-byte, that is a new, later finding to raise explicitly — not something this plan authorizes preemptively.

---

## 21. Security Considerations

This plan does not introduce any security posture beyond what ADR-0008's own "Security consequences" section already analyzes threat-by-threat; it does not weaken any of those analyses. Specific points worth restating in implementation terms:

- The reference producer's mTLS private key is the single new *class* of credential this slice introduces anywhere in the ecosystem. It must be stored using whatever local-file-permission discipline the reference implementation's test/demo fixtures already assume for other non-production secrets (§22) — never committed to the repository in a form indistinguishable from a real credential, and never logged.
- The producer also holds a second, pre-existing class of credential: the bearer token establishing the authorization subject (§14.4). It is a token of a kind `basis-gateway` already accepts and already verifies, so it introduces no new verification surface — but it is a secret, is never logged in full (the gateway's own demo redaction convention applies), and is kept configurationally independent of the certificate material so neither can be derived from the other.
- Certificate parsing happens exactly once, inside `basis-gateway` (§11) — no second, independent parsing implementation is introduced in the reference producer (the producer only *presents* its own certificate via its HTTP client library; it does not need to parse certificates itself).
- The admission configuration (§12) is exact-match only, with no wildcard, prefix, substring, or case-insensitive matching anywhere in the implementation — this is the single most security-critical implementation requirement in the entire slice and should be the first thing Phase 1B's tests lock down (§23).
- If Option B (§11) is ultimately required, implementation **stops** and this planning document is updated first (§11, "Outcome B"). Only after that review may the proxy fallback be built, and the proxy-to-gateway trust channel must then be verified, by an explicit test, to be unreachable from anywhere except the proxy itself (loopback-only or Unix-domain-socket-only) before any header-based identity assertion is trusted. No `X-Client-Cert`/`X-SSL-Client-*`-style header exists in the implementation unless that separately reviewed trust boundary authorizes it.
- The producer's certificate-derived identity is never accepted from, cross-checked against, or overridable by any request-body field or caller-set header (§20).

No accepted-architecture conflict was discovered during this planning pass. See §29 for what remains open rather than contradictory.

---

## 22. Local PKI and Test Fixtures

Minimum fixture set, generated locally by the reference implementation's own test tooling (Python's `cryptography` library, already a `basis-gateway` dependency, is sufficient to generate every fixture below without a new dependency):

- a local producer CA (self-signed, test-only);
- a gateway server certificate (signed by a separate local test CA, or the same one, deployment's choice — kept distinct from the producer CA in the reference fixture set to exercise the ADR's own "gateway server authentication and producer client authentication are separate certificate roles" distinction);
- an admitted producer client certificate (exactly one URI SAN, matching an admission-configuration entry);
- a valid-but-unadmitted producer certificate (exactly one URI SAN, not present in admission configuration);
- an invalid/untrusted certificate (signed by a CA outside the configured trust anchor);
- a wrong-SAN certificate (a URI SAN present but not the expected value — distinct fixture from "unadmitted" to test exact-match logic specifically, not merely "no match");
- a certificate with zero eligible URI SANs;
- a certificate with multiple eligible URI SANs;
- an expired certificate, generated with a validity window in the past, using the same local CA and deterministic system-clock control the test harness already needs for other time-sensitive tests (no reliance on waiting for real-world expiry).

### Bearer-subject credential fixtures

Separate from, and independent of, every certificate above (§14.4). Generated the same way `basis-gateway`'s own `demo/operation-aware/run_demo.py` already does — an ephemeral in-memory RSA key plus a locally issued, RS256-signed BASIS-local identity token — with no network access and no `basis-identity` deployment:

- a signing key pair whose public half is supplied to the gateway via `BASIS_LOCAL_TOKEN_PUBLIC_KEYS_JSON`;
- a valid token for the synthetic authorization subject, whose `sub` is **deliberately different** from every producer URI SAN in the fixture set (§14.3);
- an invalid bearer token (wrong signature, wrong issuer, wrong audience, or expired) for the negative scenario where producer mTLS succeeds but subject authentication fails;
- optionally, a second valid token for a differently-authorized subject, so `ALLOW`/`DENY` can be driven by subject identity alone against one checked-in policy bundle.

No fixture is shared between the two credential kinds: no test derives a bearer token from a certificate, or a certificate identity from a token.

No real private credential is ever committed. All fixtures are clearly test-only (short validity where relevant, obviously fake subject/SAN naming such as `spiffe://basis-reference-test/...` or an equivalent private test namespace, never resembling a production identity). No test depends on public PKI or external network access — every fixture is generated and consumed entirely offline, consistent with the ecosystem-wide bias this entire slice inherits.

---

## 23. Test and Conformance Strategy

### `basis-adapters`

No new implementation behavior is anticipated. `construct_adapter_evidence()` and `RestAdapter.normalize()` are both already implemented and already covered (67 test functions found in `tests/test_evidence.py`, confirmed by inspection). This plan does not create adapter work without a demonstrated gap, and none was found.

### `basis-gateway`

- **TLS topology spike result (Phase 1A):** a test or documented reproducible harness recording *which* mechanism exposes the validated peer certificate to application code, and asserting that mechanism actually works on the pinned server version. Until this exists, no admission test below is meaningful.
- mTLS identity extraction: exactly-one-SAN success case; zero-SAN rejection; multiple-SAN rejection; non-deterministically-extractable SAN rejection.
- Exact admission: admitted identity succeeds; case-mismatch rejected; prefix-only match rejected; suffix-only match rejected; wildcard-shaped configuration value rejected or treated as invalid configuration at startup; unregistered-but-valid identity rejected; disabled identity rejected.
- Common-Name-only certificate rejected (no fallback).
- **TLS failures remain transport-layer failures where applicable:** a test asserting that a handshake-rejected certificate produces no application-level producer trust object and no route invocation — i.e. that the failure is observable as a transport failure, not as a producer-admission or authorization result (§17.1).
- Legacy allowlist coexistence: both paths independently testable; a request authenticated via one path never accidentally satisfies the other.
- **Producer/subject distinction:** the producer trust model never automatically populates the authorization subject — an admitted producer identity never becomes `authorization_subject_id` (§14.7).
- **Valid producer + invalid bearer subject fails before evaluation:** an admitted mTLS producer presenting a missing/malformed/expired/badly-signed token produces no `basis-core` invocation at all.
- **Valid bearer subject + invalid or unadmitted producer cannot assert producer-only context:** the request is either rejected at the transport layer (invalid certificate) or proceeds as an ordinary caller whose producer-only fields are rejected (unadmitted certificate).
- Producer-only field enforcement: unchanged existing `UntrustedOperationProducerContextError` tests continue to pass; new tests confirm the same rejection applies when the caller is mTLS-unadmitted rather than bearer-untrusted.
- Authentication-profile distinction: `OperationProducerTrust.source` correctly reflects which mechanism (§13) applied.
- Configuration validation: startup fails closed on invalid combinations (producer path enabled without trust anchor, enabled without server cert, enabled without a working bearer-subject configuration, etc.).

### Reference producer repository

- Adapter invocation and normalization pass-through (no duplicate normalization logic).
- **Digest-addressed blob retention:** bytes are written under the digest-derived storage key, byte-for-byte identical to `ConstructedAdapterEvidence.canonical_bytes`.
- **File and directory fsync behavior where testable:** the flush/fsync/atomic-rename discipline (§9) is exercised — at minimum by asserting the temporary file is created on the target filesystem and that the store's persistence routine is the only writer, and, where the platform and test harness permit, by instrumenting/faking the fsync calls to prove both the file and its containing directory are synced before success is reported.
- **Retention failure prevents ID minting:** a forced blob-write failure prevents `reference_id` minting and prevents submission — this remains the single most important test in the entire reference component, mirroring ADR-0008's required invariant.
- **Post-retention ID minting:** the ordering is asserted directly, not merely implied — the identifier factory is not invoked until retention has returned success.
- **Reference-binding persistence:** `refs/<reference_id>` is written and confirmed with the same durability discipline before any reference is assembled.
- **Binding failure prevents reference assembly:** a forced binding-write failure prevents `AdapterEvidenceReference` assembly and prevents submission, even though the blob was retained and the identifier was minted.
- **Retrieval by `reference_id`:** binding resolves to digest + storage key, which resolves to the retained bytes; "binding not found," "blob missing," and "digest mismatch" are three distinguishable results.
- **Digest recomputation and mismatch detection:** recomputing SHA-256 over the retrieved bytes matches the expected digest taken from the binding record (and, in the integration test, from `AdapterEvidenceReference.evidence_digest`); a deliberately corrupted blob is detected. A test must also assert the expected digest is **not** sourced from the bytes under verification.
- `reference_id` generation, via the injectable factory, deterministic under test.
- Metadata assignment (`adapter_source`, `redaction_classification`) — §10.
- Reference assembly against the published `AdapterEvidenceReference` schema (via `basis-core`'s existing Pydantic model — validation is free, reused, not reimplemented).
- **mTLS client credentials:** the client presents the configured certificate/key pair.
- **Separate bearer subject credential:** the client sends an independently configured `Authorization: Bearer` header; a test asserts neither credential is derived from the other and that omitting either is a configuration error rather than a silently attempted request.
- Gateway error handling: connection failure, TLS handshake failure, 4xx rejection (admission vs. bearer-authentication cases distinguishable), parsed `DENY`, parsed evaluation failure, parsed `not_applicable`, parsed `ALLOW`.
- **No execution:** an explicit test asserting that no code path in the reference producer performs any network call other than the one HTTP submission to `basis-gateway` and, where used, the local evidence-retention filesystem writes — nothing resembling protocol dispatch exists to accidentally trigger.

### Cross-repository integration

**Positive path:** valid admitted producer certificate **plus a valid bearer token for a different synthetic subject** → REST normalization (fixture mapping, §8) → evidence constructed → blob retained under the digest-derived key → retention confirmed → `reference_id` minted → binding persisted and confirmed → `AdapterEvidenceReference` assembled → mTLS connection established → gateway validates the certificate and derives exactly one URI SAN → gateway admits → gateway authenticates the bearer subject independently → gateway composes and invokes `basis-core` → deterministic `ALLOW` (or another configured, deterministic outcome) → `GatewayAuditEvent` recorded showing **both** identities → producer stops, no execution.

**Required conformance assertion.** The positive integration test must assert:

```text
producer URI SAN != bearer subject_id
```

while still producing a valid, authorized request. This distinction is a first-class conformance requirement of the slice, not an incidental property of the chosen fixtures — a future change that made the two identities equal must fail this test.

**Negative paths:** valid certificate but unadmitted SAN (producer-owned context rejected); invalid/untrusted certificate (TLS handshake itself fails — no application trust object); zero or multiple URI SANs (identity derivation fails closed); valid mTLS producer but invalid bearer subject token (no kernel invocation); valid bearer subject but invalid or unadmitted producer certificate (no producer-owned context); evidence blob-retention failure (no submission attempted at all); reference-binding persistence failure (no reference assembled, no submission); untrusted bearer caller asserting producer-only context (existing rejection path, exercised alongside the new mTLS tests to confirm no regression); deterministic `DENY` from a configured policy bundle (kernel path unchanged, producer correctly stops); gateway unavailable (producer surfaces failure locally, nothing treated as authorized).

This is the canonical scenario the discovery assessment's §11 explicitly names as a currently-confirmed gap ("no end-to-end test anywhere... exercises a real... producer submission calling both `basis-adapters`' evidence construction and `basis-gateway`'s trusted-producer classification in sequence"). Closing it is this slice's primary test-conformance contribution.

---

## 24. Local Demonstration

A deterministic, offline demo/test environment, structured analogously to `basis-gateway`'s existing `demo/operation-aware/` (README, a run script, checked-in policy/expected fixtures) but living inside the new reference-producer repository and driving both `basis-adapters` and a locally-configured `basis-gateway` instance:

The demonstration must prove **both** identities independently, and must make it impossible to mistake producer-authentication failure, subject-authentication failure, admission failure, and authorization denial for one another.

### Positive scenario

- valid producer client certificate;
- exactly one admitted URI SAN;
- a valid bearer credential for a **different** synthetic authorization subject (§14.3);
- evidence blob retained under the digest-derived key, retention confirmed;
- `reference_id` minted, binding persisted and confirmed, `AdapterEvidenceReference` assembled;
- gateway accepts the producer;
- `basis-core` evaluates the **bearer subject**, not the producer;
- deterministic `ALLOW` (or another configured, deterministic result);
- printed output includes the `reference_id`, the digest, the retained-evidence location, and the disposition;
- audit output clearly distinguishes producer identity from subject identity, side by side, with both values visible.

### Negative scenarios

Each prints a rejection reason naming its own layer (§17.1), so the four failure kinds are never conflated:

- valid producer certificate but **unadmitted** URI SAN — producer admission failure;
- invalid/untrusted producer certificate — transport authentication failure (the request never reaches application processing);
- zero URI SANs, and separately, multiple URI SANs — identity-derivation failure, fail closed;
- valid mTLS producer but **invalid bearer subject token** — subject authentication failure, no kernel invocation;
- valid bearer subject but invalid/unadmitted producer certificate — producer-owned context refused;
- evidence blob-retention failure — no `reference_id`, no submission at all;
- reference-binding persistence failure — reference never assembled, no submission at all;
- deterministic `DENY` against a checked-in policy bundle — authorization denial, kernel path unchanged.

### Inspection steps

- retained evidence/reference inspection — a small CLI/script step that resolves `reference_id` → binding → digest/storage key, retrieves the stored bytes, recomputes SHA-256 over them, and compares against the digest taken from the binding (and from the assembled reference), demonstrating that the expected digest is never read out of the bytes being verified (§9);
- gateway audit output for at least one scenario, to make the audit trail's contents visible without a separate audit-query tool.

No Docker or Kubernetes is required — the demo runs the gateway and the reference producer as two local processes (or, if Option A's spike allows, potentially in-process test fixtures for the fastest tests, with the demo itself always using two real processes to prove the network boundary is real) directly in a developer environment or Codespace, consistent with the existing `demo/operation-aware/` precedent. `basis-deploy` is not created or required for this demonstration.

---

## 25. Dependency Assessment

| Dependency | Repository | Why stdlib/current deps are insufficient | Runtime or test-only | Notes |
| - | - | - | - | - |
| `cryptography` (X.509 parsing, SAN extraction) | `basis-gateway` | **Already a dependency** (`>=42.0.0`, confirmed) | Runtime | No new dependency — this materially simplifies Phase 1 |
| `httpx` (mTLS-capable HTTP client) | new reference-producer repository | stdlib `http.client` supports client certificates but with a materially worse ergonomic and testing story; `httpx` is already the ecosystem's chosen pattern (`basis-console`'s `GatewayClient` already uses it) | Runtime | Reused pattern, not a shared code dependency — the producer does not import `basis-console` |
| `basis-adapters` (as a library dependency) | new reference-producer repository | This is the entire point of the slice — reuse, not reimplementation, of normalization and evidence construction | Runtime | Already published/installable; no version-pinning concern beyond the repository's existing compatibility discipline |
| `cryptography` (test-fixture certificate generation) | new reference-producer repository | Needed to generate the local PKI fixtures (§22) deterministically without external tooling | Test-only | Same library the gateway already depends on; no new maintenance burden ecosystem-wide |
| `PyJWT` (issue the local BASIS-local identity token for the bearer-subject credential) | new reference-producer repository | The subject credential must be signed offline; the gateway's own `demo/operation-aware/run_demo.py` already uses exactly this library for exactly this purpose | Test/demo-only (or runtime if the reference producer issues its own demo token at run time) | Already an established ecosystem library (`basis-identity` runtime dependency, `basis-gateway` demo dependency); the producer *verifies* nothing — the gateway does |
| ASGI server with confirmed TLS-extension support (only if the Phase 1A spike finds `uvicorn` insufficient) | `basis-gateway` | Contingent — not committed to in this plan | Runtime, contingent | Explicitly not decided here; §11. If the spike's answer is "a different ASGI server" or "a reverse proxy," Phase 1B does not begin until this document is updated (§11, Outcome B) |

No dependency is added by this planning PR. `basis-adapters` gains no new dependency under any branch of this plan. `basis-core` gains no new dependency. License/maintenance profile of every dependency named above is already established ecosystem-wide (`cryptography` and `httpx` are both already in production use in at least one released BASIS component) except the contingent ASGI-server question, which is explicitly deferred pending the Phase 1 spike's outcome.

---

## 26. Implementation PR Sequence

### Phase 1A — mTLS topology and certificate-exposure spike (`basis-gateway`)

**A hard gate. No producer-admission implementation may proceed until this phase resolves the topology (§11).**

1. Prove the TLS termination topology for the reference environment.
2. Prove client-certificate validation can be required and enforced.
3. Prove safe, documented, stable access to the validated peer certificate's URI SAN from gateway application code, without trusting any caller-controlled field.
4. Prove testability across the four certificate cases (valid; invalid/untrusted; zero URI SAN; multiple URI SANs).
5. Record which of the five mechanisms (§11) the working answer requires.

**Exit — Outcome A:** direct application termination works; document the exact mechanism and proceed to Phase 1B.
**Exit — Outcome B:** direct termination does not provide a safe application identity boundary. **Stop. Do not proceed to Phase 1B.** Update this planning document first, in a small follow-up architecture/planning PR covering the proxy trust boundary questions §11 enumerates, and have it reviewed. Only then may proxy-backed producer admission implementation begin.

### Phase 1B — Gateway producer authentication and admission foundation (`basis-gateway`)

Begins only after Phase 1A succeeds (or, under Outcome B, after the follow-up planning PR is reviewed).

1. Configuration additions (§12), fail-closed at startup.
2. Certificate identity extraction module (§11), using `cryptography` (already a dependency).
3. Exact URI SAN derivation and admission logic (§13's ADR-0008 path).
4. `OperationProducerTrust` additive refinement, scoped per §14.7 (no TLS-handshake failure states).
5. Legacy-allowlist coexistence (no removal, no reinterpretation) plus its coexistence tests.
6. Dual producer + bearer-subject authentication behavior (§14): both credentials required, independently verified, neither derived from the other.
7. Tests per §23, including the transport-vs-admission-vs-authorization layer separation (§17.1).
8. No producer runtime exists yet at the end of this phase — it is entirely gateway-internal.

**Phase 1 is not complete when certificate-parsing code exists.** It is complete only when a validated producer certificate can be mapped to exactly one URI SAN through a trust boundary that does not depend on caller-controlled request data (§11).

### Phase 2 — Reference evidence store and reference assembly (new `basis-operation-producer-reference` repository)

1. Repository scaffolding (minimal — no packaging/release ceremony, §5).
2. Digest-addressed blob store: `blobs/sha256/<digest>` (§9).
3. Flush/fsync/atomic-rename persistence discipline for both blobs and binding records (§9).
4. Post-retention `reference_id` minting via injectable factory (§9, §10) — never before confirmed retention.
5. Reference binding: durable, confirmed `refs/<reference_id>` → digest/storage-key record (§9).
6. `adapter_source`/`redaction_classification` assignment (§10).
7. Final `AdapterEvidenceReference` assembly, validated against `basis-core`'s existing Pydantic model, gated on confirmed binding.
8. Retrieval and digest-verification helper, with the expected digest sourced from the binding/reference — never from the bytes under verification (§9).
9. Unit tests for all of the above, including retention-failure/no-mint and binding-failure/no-assembly.

### Phase 3 — Reference producer gateway client

1. mTLS producer credential: `httpx` client certificate/key configuration (§15).
2. **Separate** bearer subject credential: independently configured `Authorization: Bearer` header (§14.4, §15).
3. Operation-aware request submission against the existing, unchanged endpoint.
4. Deterministic error handling across the layers of §17.
5. Tests against a locally running (or in-process test-harness) `basis-gateway` instance with Phase 1B's admission logic enabled.

### Phase 4 — REST vertical slice

1. Wire `RestAdapter.normalize()` (§8's fixture operation) into the reference producer's Phase 2/3 pipeline end to end: adapter → evidence → digest-addressed storage → reference binding → reference assembly → dual authentication → admission → gateway → core → audit.
2. No execution code added anywhere.
3. Cross-repository integration tests (§23's positive/negative paths), including the required `producer URI SAN != bearer subject_id` assertion.

### Phase 5 — Cross-repository conformance, demo, and review

1. Local demonstration, positive and negative scenarios (§24).
2. Evidence inspection and digest-verification demonstration (§24).
3. Trust-boundary verification — the selected topology's identity boundary is demonstrated, not asserted (§11, §21).
4. Documentation of what the slice proves and does not prove (§28).
5. Release-readiness assessment per repository (§27) — likely: no release required for `basis-adapters`, `basis-core`, or `basis-schemas`; a `basis-gateway` release is likely once Phase 1B lands; the reference-producer repository itself is not released to any package index.
6. The permanent repository decision gate (§30).

Phases are ordered to minimize rework and to prevent building on an unproven boundary: Phase 1A is a gate, not a task; gateway admission logic (Phase 1B) does not depend on the reference producer existing and can be fully tested against synthetic client certificates before any producer code is written; the evidence store (Phase 2) does not depend on gateway changes at all; only Phase 3 requires both to exist simultaneously.

---

## 27. Versioning and Release Impact

| Repository | Nature of change | Release likely? |
| - | - | - |
| `basis-adapters` | None anticipated | No — no adapter change is authorized or expected by this plan |
| `basis-core` | None anticipated | No — no kernel change is authorized or expected; this plan explicitly avoids changing `basis-core` absent a demonstrated contract gap, and none was found |
| `basis-schemas` | None anticipated | No — §20 found no contract gap requiring publication |
| `basis-gateway` | Additive: new configuration surface, additive internal dataclass fields, new certificate-identity-extraction module; existing `v0.1`/`v0.2` behavior unaffected | Likely, once Phase 1B lands, following the repository's own existing additive-release discipline (the same pattern `v0.2.0` itself already used for the operation-aware path) |
| `basis-identity` | None anticipated | No — this plan explicitly does not depend on new `basis-identity` workload-identity capability (§30), per ADR-0008's own finding that the mTLS profile is independently deployable without it |
| `basis-console` | None anticipated | No — §5.6 of the discovery assessment already confirms no console change is required for this slice, and this plan finds nothing to contradict that; the console is not touched merely because operation-aware results can be displayed |
| new reference-producer repository | New, entirely internal to itself | No formal release/package-index publication for this reference/demo-scoped component (§5) |

---

## 28. Completion Criteria

The future implementation is complete when, at minimum:

- **the selected mTLS termination topology has been proven by Phase 1A**, and no admission implementation preceded that proof;
- **no proxy-trusted identity header exists** anywhere in the implementation unless a separately reviewed fallback trust boundary (§11, Outcome B) explicitly authorized it;
- an mTLS producer connection to `basis-gateway` works locally, end to end, with no public Internet dependency anywhere in the path;
- certificate identity is derived from exactly one URI SAN, with Common Name never used as a fallback;
- exact, case-sensitive admission is enforced, with every ADR-0008-prohibited matching style (prefix/suffix/substring/wildcard/case-insensitive) demonstrated to fail;
- a valid-but-unadmitted producer certificate fails closed;
- an ambiguous-SAN certificate (zero or multiple) fails closed;
- the producer and the authorization subject remain distinct in every code path and every audit record produced;
- **mTLS producer identity is established independently of bearer subject identity**, through two separately configured credentials neither of which is derived from the other;
- **the positive conformance scenario uses distinct producer and subject identities**, asserted directly (`producer URI SAN != bearer subject_id`);
- **an invalid bearer subject fails even when producer mTLS succeeds** — no `basis-core` invocation occurs;
- **an invalid or unadmitted producer fails even when bearer subject authentication succeeds** — no producer-owned context is accepted;
- **TLS-handshake failures are not misrepresented as application-level authorization outcomes**, and are not modelled as `OperationProducerTrust` values;
- the REST adapter is used through its existing, unmodified public API — no duplicate normalization logic exists anywhere in the reference producer;
- evidence construction uses the released `basis-adapters` `construct_adapter_evidence()` unmodified;
- **the evidence blob is stored, under a digest-derived storage key, before `reference_id` is generated**;
- **the reference binding is persisted and confirmed before any reference is assembled or submitted**;
- **retrieved canonical bytes reproduce the expected SHA-256 digest**;
- **the expected digest is not obtained from the bytes being verified** — it comes from the stored binding or from `AdapterEvidenceReference.evidence_digest`;
- **local persistence uses the documented flush/fsync/atomic-rename discipline**, for both the blob and the binding record, rather than relying on atomic rename alone;
- the final `AdapterEvidenceReference` validates against the published contract (via `basis-core`'s existing model);
- `basis-gateway` receives the reference from an authenticated, admitted producer and invokes the existing, unmodified `basis-core` operation-aware evaluation path;
- deterministic `ALLOW` and `DENY` scenarios both work and are both observable in `GatewayAuditEvent`, with producer and subject identities separately visible;
- **no execution occurs** — no execution code exists or runs anywhere in the path;
- every test above is automated, not manually verified;
- implementation repository boundaries remain exactly as this plan and its governing ADRs describe them — no role conflation was introduced anywhere along the way.

---

## 29. Deferred Decisions

Restated from ADR-0008 and this plan's own findings, not resolved here:

- Permanent operation-producer-runtime repository placement (§30).
- Exact `basis-gateway` environment-variable names for the new configuration surface (§12), pending the Phase 1A spike.
- Whether Option A (in-process ASGI TLS termination) is technically achievable with the current server stack, or whether a server change or Option B is required (§11). **This is a gate, not an open design question:** Phase 1A must answer it before Phase 1B begins, and Outcome B requires a follow-up planning PR before implementation resumes.
- How machine subjects, workload subjects, subject-less producer operations, delegated identities, and `basis-identity` workload credentials should eventually interact with producer authentication (§14.6). For this slice the model is fixed: mTLS producer + bearer subject.
- Whether a production deployment would use `AUTH_MODE=oidc` rather than the reference slice's `basis_local_token` subject credential (§14.4) — a deployment choice the gateway already supports, unaffected by this plan.
- Category-scoped producer capability (unchanged from ADR-0008 — not addressed by this slice).
- Full credential lifecycle automation, certificate rotation tooling, and revocation-list distribution (`ROADMAP.md` Phase 4, unchanged).
- Whether `OPERATION_PRODUCER_SUBJECT_IDS` is ever deprecated (explicitly not decided by ADR-0008 or by this plan).
- Any execution-plane architecture, protocol executor design, or execution-result vocabulary.
- Production evidence-store topology (this plan's filesystem choice is reference-scoped only, §9).
- A global `reference_id` namespace.
- Ecosystem-wide retry/idempotency semantics beyond §18's minimal posture.

---

## 30. Permanent Repository Decision Gate

Permanent producer-runtime repository placement will be reconsidered only after this bounded slice is working end to end, per ADR-0008 and `operation-producer-and-execution-boundary.md` §11's own deferred gate. At that review, architecture should evaluate: the size and cohesion of the accumulated producer code; its dependency graph (does it still depend cleanly on `basis-adapters` as a library and on `basis-gateway` only over HTTP, with no reverse or circular dependency having crept in); the credential boundary (has the private-key-holding responsibility remained cleanly separated from every other component); the persistence boundary (has evidence retention remained a distinct, swappable concern, per the evidence-store responsibilities §9 defines, with content identity and logical reference identity still separate); whether an independent release cadence is actually needed; the deployment lifecycle once `basis-deploy` exists; how many producer protocols are eventually expected beyond REST; whether multiple future runtimes would share substantial code (in which case a shared library, not just a shared repository, becomes the real question); whether colocating the reference implementation inside its own small repository has caused any ownership confusion in practice; and whether extraction — which, for a repository that is already separate, mostly means *renaming and formalizing*, not physically moving code — is in fact straightforward. This document does not name a permanent repository and does not pre-judge that review's outcome.

---

## Appendix A — Repository Evidence Index

| Repository | File / Symbol | What it confirms |
| - | - | - |
| `basis-adapters` | `src/basis_adapters/rest/adapter.py` — `RestAdapter.normalize()` | REST normalization performs no network I/O; returns `AdapterResult` |
| `basis-adapters` | `src/basis_adapters/evidence.py` — `construct_adapter_evidence()`, `AdapterEvidenceMaterial`, `ConstructedAdapterEvidence`, `EvidenceDigest` | Pure, deterministic evidence construction; does not mint `reference_id`, `adapter_source`, or `redaction_classification`; already implements `EVIDENCE_PROFILE`/`CANONICALIZATION_PROFILE` per ADR-0007. `ConstructedAdapterEvidence` returns `material`, `canonical_bytes`, and `digest` as three **separate** attributes — the digest is computed over the canonical bytes, not carried inside them (§9) |
| `basis-adapters` | `src/basis_adapters/__init__.py` — `__all__` | Confirms the exact public surface this plan depends on |
| `basis-adapters` | `examples/rest/mapping-minimal.example.json`, `mapping.example.json` | Existing, checked-in fixture mapping configs usable for §8's reference operation without inventing new fixtures |
| `basis-gateway` | `src/basis_gateway/auth/operation_producer.py` — `classify_operation_producer()`, `OperationProducerTrust`, `OperationProducerTrustSource` | Current legacy allowlist model; confirms exact shape of the additive change in §13–§14 |
| `basis-gateway` | `src/basis_gateway/config.py` — `AuthMode`, `GatewayConfig`, `OPERATION_PRODUCER_SUBJECT_IDS`, `OPERATION_AWARE_ENABLED` | Confirms no TLS configuration surface exists today; confirms `pydantic-settings` environment-variable pattern to extend in §12; confirms exactly two existing bearer modes (`oidc`, `basis_local_token`) and that no third is invented by §14.4 |
| `basis-gateway` | `src/basis_gateway/auth/runtime.py` — `authenticate()`, `build_basis_local_token_trust_config()` | Confirms the existing bearer-subject dispatch §14 reuses unchanged; confirms `basis_local_token` verification performs **no** network I/O and no JWKS fetch, so the reference slice needs no external identity provider |
| `basis-gateway` | `demo/operation-aware/run_demo.py` — `issue_demo_token()`, `AUTH_MODE=basis_local_token` setup | Confirms the deterministic, offline bearer-credential path §14.4 selects already exists as gateway test/demo infrastructure (ephemeral in-memory RSA key, RS256 BASIS-local identity token, real `Authorization: Bearer` header, real verifier) |
| `basis-gateway` | `src/basis_gateway/core/operation_aware_composition.py` — `UntrustedOperationProducerContextError`, `ProvenanceClassification`, `present_operation_producer_only_fields()` | Confirms existing producer-only field rejection this plan does not alter |
| `basis-gateway` | `src/basis_gateway/core/operation_aware_evaluator.py` — `_default_trace_id()`, `_default_evidence_id()` | Confirms the injectable-factory identifier-generation pattern §10 reuses for `reference_id` |
| `basis-gateway` | `src/basis_gateway/middleware/correlation.py` — `CorrelationMiddleware` | Confirms unconditional correlation-ID generation and caller-header rejection §16 preserves |
| `basis-gateway` | `src/basis_gateway/main.py` | Confirms FastAPI/uvicorn ASGI entrypoint; no TLS termination configured in-app today |
| `basis-gateway` | `pyproject.toml` | Confirms `cryptography>=42.0.0` and `httpx>=0.27.0` are already dependencies |
| `basis-gateway` | `demo/operation-aware/README.md`, `run_demo.py` | Existing precedent for a bounded, offline, reproducible demo — pattern reused in §24 |
| `basis-core` | `src/basis_core/domain/evidence.py` — `AdapterEvidenceReference`, `EvidenceDigest`, `IdentityEvidenceReference` | Confirms the exact Pydantic model the reference producer's assembled reference must validate against; confirms closed `redaction_classification` enum enforcement |
| `basis-schemas` | `schemas/adapter-evidence-reference/adapter-evidence-reference.yaml` — `composition` block | Confirms the ADR-0007 ownership-metadata correction (PR #22) is merged; `produced_by: operation-producer runtime` is now accurate |
| `basis-identity` | `src/basis_identity/models/identity_context.py` — `SubjectType`, `AuthenticationProtocol` | Confirms no workload-credential establishment pipeline exists; confirms this plan's finding that no `basis-identity` dependency is required |
| `basis-console` | `src/basis_console/gateway/client.py` — `GatewayClient` | Confirms the `httpx`-based client pattern §15 references without creating a dependency on `basis-console` |

---

## Appendix B — Implementation Capability Matrix

| Capability | Exists today? | Owning repository once implemented | This plan's phase |
| - | - | - | - |
| REST normalization | Yes | `basis-adapters` | N/A (reused) |
| Evidence-material construction, canonicalization, digest | Yes | `basis-adapters` | N/A (reused) |
| Digest-addressed evidence blob retention (fsync/atomic-rename) | No | new reference-producer repository | Phase 2 |
| Post-retention `reference_id` minting | No | new reference-producer repository | Phase 2 |
| `reference_id` → digest/storage-key binding record | No | new reference-producer repository | Phase 2 |
| Retrieval + digest recomputation/verification helper | No | new reference-producer repository | Phase 2 |
| `adapter_source` / `redaction_classification` assignment | No | new reference-producer repository | Phase 2 |
| `AdapterEvidenceReference` assembly | No | new reference-producer repository | Phase 2 |
| mTLS client presentation | No | new reference-producer repository | Phase 3 |
| Separate bearer-subject credential presentation | No | new reference-producer repository | Phase 3 |
| Gateway submission | No | new reference-producer repository | Phase 3 |
| **mTLS termination topology proof (certificate exposure to application code)** | No | `basis-gateway` | **Phase 1A (gate)** |
| mTLS server-side termination / cert validation | No | `basis-gateway` | Phase 1B |
| URI SAN identity derivation | No | `basis-gateway` | Phase 1B |
| Exact admission matching | No | `basis-gateway` | Phase 1B |
| Producer trust internal model refinement | No | `basis-gateway` | Phase 1B |
| Dual producer-mTLS + bearer-subject authentication on one request | No (bearer subject auth itself: Yes) | `basis-gateway` | Phase 1B |
| Legacy allowlist coexistence | Yes (allowlist itself); No (coexistence tests) | `basis-gateway` | Phase 1B |
| Operation-aware request composition | Yes | `basis-gateway` | N/A (reused) |
| Kernel evaluation | Yes | `basis-core` | N/A (reused) |
| Gateway enforcement / HTTP disposition | Yes | `basis-gateway` | N/A (reused) |
| Audit emission | Yes (base); additive diagnostic fields, No | `basis-gateway` / `basis-core` | Phase 1B (additive fields only) |
| End-to-end REST-to-disposition integration | No | cross-repository | Phase 4 |
| Local demo / conformance evidence | No | new reference-producer repository | Phase 5 |
| Protocol execution | No, and not authorized by this plan | none | Out of scope |
