# Operation-Producer Cross-Repository Discovery Assessment

> **Status: Non-normative discovery assessment.** This document records inspected evidence and candidate options. It does not assign final repository ownership or supersede an ADR.

**Scope:** A read-only, evidence-based inspection of the locally checked-out state of `basis-architecture`, `basis-schemas`, `basis-core`, `basis-gateway`, `basis-adapters`, `basis-console`, and `basis-identity`, plus an attempt to inspect `basis-deploy` and any demo/PoC/lab repository. It answers, with concrete evidence, what already exists for turning a protocol or platform operation into a complete, authenticated, evidence-backed, operation-aware authorization submission, and what remains architecture-only, contract-only, or entirely absent. It does not implement anything, does not create an ADR, and does not select a repository name or ownership model.

**Companion documents:** [`docs/architecture/operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) (the primary architecture document this assessment verifies against implementation), [ADR-0007](../adr/0007-adapter-evidence-construction.md) and [`docs/architecture/adapter-evidence-construction-semantics.md`](adapter-evidence-construction-semantics.md), [`docs/architecture/ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md), [`docs/roadmaps/identity-to-operation-contract-and-interoperability.md`](../roadmaps/identity-to-operation-contract-and-interoperability.md), [`docs/roadmaps/post-authentication-identity-activity-correlation-and-detection.md`](../roadmaps/post-authentication-identity-activity-correlation-and-detection.md), [`ROADMAP.md`](../../ROADMAP.md) (**Next Producer and Execution-Evidence Boundary**).

---

## 1. Executive Summary

This assessment inspected the locally checked-out state of all seven BASIS Core Services Distribution repositories that exist today. `basis-deploy` has no repository yet, consistent with `ROADMAP.md`, and was not inspected. No demo, PoC, lab, or integration repository beyond `basis-architecture`'s own `whitepapers/` PoC references was available in the working environment; the separate `basis-poc` research repository this repository's `README.md` refers to was not mounted and was not inspected.

The strongest confirmed finding is that `basis-architecture` has already performed most of the architecture-level discovery this task might otherwise be asked to redo. [`docs/architecture/operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) — architecture-planning status, present on the branch this assessment was authored on — already names the missing logical roles (operation-producer runtime, protocol executor, execution-evidence producer), states six distinct trust facts that are routinely conflated, defines a field-ownership table for the adapter-to-gateway handoff, records a correlation model, and proposes a ten-stage, decision-gated implementation sequence. [ADR-0007](../adr/0007-adapter-evidence-construction.md) (Accepted) resolves that document's Stage 2 decision gate: it assigns evidence-material construction, RFC 8785 canonicalization, and digest computation to `basis-adapters`, and reference assembly (`reference_id` minting, `adapter_source`, `redaction_classification`, request/correlation linkage) to the still-unimplemented operation-producer runtime. This assessment's implementation inspection confirms that ADR-0007's `basis-adapters`-owned half is genuinely implemented: `src/basis_adapters/evidence.py`'s `construct_adapter_evidence()` is a real, pure function producing the governed `basis-adapter-evidence-v1` material, RFC-8785-canonical bytes, and a SHA-256 digest, with 67 test functions found in `tests/test_evidence.py` (this assessment inspected but did not execute that suite; see §2).

The largest unresolved gap is exactly the one the architecture document already named and did not close: nothing in the ecosystem authenticates an operation-producer runtime to `basis-gateway`, assembles the final `AdapterEvidenceReference` from `basis-adapters`' constructed material, or invokes `basis-adapters` and `basis-gateway` in sequence for a real protocol operation. `basis-gateway`'s trusted-producer classification (`src/basis_gateway/auth/operation_producer.py`, `classify_operation_producer()`) is real and test-covered (23 test functions found in `tests/test_operation_producer_trust.py`; not executed by this assessment) and correctly implements a safe-default, configuration-driven, exact-match subject-ID allowlist — but it classifies an already-authenticated Bearer-token caller, not a purpose-built producer credential, and the architecture document itself states this allowlist "is not the adapter-to-gateway trusted-producer contract... which remains open." No component mints `reference_id`. No component calls both `basis-adapters` and `basis-gateway` in the same process or workflow. No execution component exists anywhere in the inspected repositories — `basis-adapters`' own architecture states explicitly that it has "no sockets, no packet parsing, no protocol stacks, no live protocol communication."

Test-function counts cited throughout this document (§2) reflect what this assessment found by inspection, not suites it executed.

No existing component owns the complete producer lifecycle, and the evidence does not support treating that as a defect: the boundary document explicitly defers the *repository-placement* question (§11) pending a bounded reference implementation, and nothing inspected contradicts that deferral. This assessment separates two questions the source document sometimes treats together: whether an existing repository can absorb the *complete* producer lifecycle without straining its charter (no, confirmed by inspection, §10), and whether a narrowly bounded producer *component* could initially be colocated inside an existing repository rather than requiring a standalone repository from day one (genuinely open — not the same question, §9–§10). Whether a *new repository* is eventually warranted remains indeterminate — the architecture document frames this as a decision gate, not a conclusion, resolved only after Stage 4 (authentication mechanism selection) and Stage 5 (a reference implementation) in its own §13 sequence.

Producer authentication is the first and most immediate architectural decision, but not the only unresolved behavior a bounded producer slice needs: a reference implementation must still resolve minimum reference-minting, redaction-policy application, persistence, correlation, and retry behavior, and its own provisional placement, without prematurely defining the execution plane (§13–§14). Selecting the mechanism unblocks that work; it does not complete it.

**Central conclusion.** The inspected ecosystem contains two independently implemented and test-covered segments — adapter normalization and deterministic evidence construction, and gateway-to-core operation-aware evaluation and audit. No inspected runtime joins them into an authenticated producer lifecycle, and no execution plane exists. The next normative decision is the producer-to-gateway workload-authentication and admission model. That decision should authorize a narrowly bounded reference slice and establish its provisional implementation location without deciding permanent repository placement, mandatory schema publication, evidence-storage topology, or execution architecture.

---

## 2. Scope and Methodology

### Repositories inspected

| Repository | Branch | HEAD SHA | Working tree at inspection start | Inspection date |
| - | - | - | - | - |
| `basis-architecture` | `docs/operation-producer-discovery-assessment` | `bd84395adb324947b20544ee51c586de1a6a1ff6` | clean | 2026-08-09 |
| `basis-schemas` | `main` | `da7832972dad36dea6ef2796161a1990fbbe6a05` | clean | 2026-08-09 |
| `basis-core` | `main` | `7a924970eb12daad46cfe49eb86258ea59e92c9a` | clean | 2026-08-09 |
| `basis-gateway` | `main` | `81f72a841e1d947655f41f3e43dd0d5d559765b5` | clean | 2026-08-09 |
| `basis-adapters` | `main` | `c4759a37d6e464bfc5ac97004c54f00afc7b45ca` | clean | 2026-08-09 |
| `basis-console` | `main` | `9c3fde8ed8e180467ec266a822a65768f187e8a5` | clean | 2026-08-09 |
| `basis-identity` | `main` | `2d9490fbb4018c7f5473e7add75ac4923ee66979` | clean | 2026-08-09 |

Branch and SHA were read with `git branch --show-current` and `git rev-parse HEAD`; working-tree cleanliness at the start of inspection was read with `git status --short`. No `git fetch`, `git pull`, `git checkout`, `git commit`, `git push`, or `git merge` was run against any repository, and no successful Git write operation occurred anywhere in the production of this assessment. During post-authoring validation, an attempted `git add -N docs/architecture/operation-producer-discovery-assessment.md` (run to check the new file for whitespace issues via `git diff --check`) failed because of a pre-existing `.git/index.lock` in `basis-architecture`; it staged nothing, changed no repository status, and was not retried. What is recorded in the table above is the **inspected local checkout at the start of inspection, before this document and the accompanying `README.md` correction were authored** — not a claim about the current state of any remote, and not a claim that `basis-architecture`'s working tree stayed unchanged for the duration of this work. `basis-architecture` necessarily changed once this document and the `README.md` correction were written to it; the other six inspected repositories were not modified at any point, before or after.

### Repositories not inspected

- **`basis-deploy`.** No repository exists yet in the working environment or, per `basis-architecture`'s own `ROADMAP.md` and ecosystem documentation, anywhere in the BASIS Core Services Distribution today. This is architecture-documented as an open gap, not a limitation of this assessment.
- **`basis-poc` / demo / lab / integration repositories.** `basis-architecture`'s `README.md` references a separate PoC research repository ("Relationship to BASIS PoC and basis-core") but that repository was not mounted in the working environment and could not be inspected. `basis-gateway` does contain an in-repository `demo/operation-aware/` bounded demonstration (inspected as part of `basis-gateway`, §5.4 below), which is not a separate repository.

No implementation was inferred for either of the above from architecture documents alone; both are recorded as **Not inspected** throughout §6–§8 and §11 rather than assessed by proxy.

### Evidence-classification method

Every material capability claim below is classified as exactly one of **Implemented**, **Partially implemented**, **Contract published**, **Documented only**, **Absent**, **Ambiguous**, or **Not inspected**, per the definitions in the assignment governing this assessment. Classification is based on direct inspection of source files, test files, and schema files in the repositories above — file paths, symbol names, and (where counted) test counts are cited inline. Architecture intent is never treated as implemented behavior; a published schema is never treated as runtime behavior; a request field's existence is never treated as permission to assert it; an evidence digest is never treated as proof of provenance, authorization, or execution.

### Limitations

- This assessment inspects source and test *files*; it does not execute the test suites of any implementation repository, so "tested" below means "a test file targeting this behavior exists and appears to exercise it," not "this assessment re-ran and confirmed passing tests."
- Large architecture documents (the two roadmap documents in particular) were read in full but exceed a single-pass token budget; page boundaries were tracked and both documents were read to completion.
- Uncommitted-state concern: all seven inspected repositories reported clean working trees at the start of inspection, so there was no pre-existing uncommitted-work risk to flag. This assessment made no commits and no successful Git write operation of any kind; its one write attempt (an intent-to-add during validation, described above) failed against a pre-existing lock file and changed nothing.

---

## 3. Governing Architectural Constraints

Restated from existing `basis-architecture` documents, not invented here:

Adapters normalize. `basis-adapters` translates protocol-specific operations into a `NormalizedAuthorizationRequest` and preserves protocol evidence verbatim; it performs no network I/O, no authentication, no authorization, and no execution ([`basis-adapters.md`](basis-adapters.md); confirmed in implementation, §5.5).

Gateway enforces. `basis-gateway` authenticates, composes canonical actions and resource identifiers, invokes `basis-core`, enforces the returned disposition at its own HTTP boundary, and emits `GatewayAuditEvent` — it does not invent authorization semantics or claim that native execution occurred merely because authorization allowed it ([`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §2; confirmed in implementation, §5.4).

Core evaluates. `basis-core` remains deterministic, synchronous, side-effect free, protocol-neutral, and transport-neutral; it carries evidence *references*, never raw evidence, and never retrieves, authenticates, or verifies what a reference points to (`basis-core`: `src/basis_core/domain/evidence.py` module docstring — referenced by path, not hyperlinked, per this repository's cross-repository citation convention; confirmed in implementation, §5.3).

Console operates. `basis-console` observes and explains; it does not become the identity authority, the evidence authority, or the enforcement authority, and its Decision Simulator is explicitly a preview/simulation surface rather than a trusted-producer path (confirmed in implementation, §5.6).

Schemas define contracts, published only after implementation proves a stable shape (`ecosystem-contract-inventory.md` §1; ADR-0005's readiness discipline, restated throughout the roadmaps).

Deploy packages, once it exists; it will not own runtime semantics ([`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §10).

Subject and producer remain distinct. The authorization subject (whose authority is evaluated) and the operation producer (the caller trusted to assert operation-producer-only context) are two different concepts, and an authenticated subject is never automatically a producer (`basis-gateway`: `src/basis_gateway/auth/operation_producer.py` module docstring — referenced by path, not hyperlinked, per the same convention; confirmed in implementation, §5.4).

Authorization and execution remain distinct. An `ALLOW` disposition is never, on its own, evidence that an operation occurred ([`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) §5; the identity-to-operation roadmap's Execution invariant).

Evidence digest and provenance remain distinct. A structurally valid digest proves byte-correspondence with declared canonical input, never truthfulness, producer authenticity, authorization, or execution (ADR-0007 "Digest requirements"; restated verbatim in `basis-core`'s `domain/evidence.py` module docstring).

No new rule is introduced in this section; every statement above traces to an existing document or to source code inspected in §5.

---

## 4. Current End-to-End Lifecycle

### What works today (confirmed by inspection)

```text
── Segment A: basis-adapters (normalization + evidence construction) ──────
ProtocolOperation
    │  (basis-adapters: implemented — per-protocol adapters; test suites
    │   not executed by this assessment, see §2)
    ▼
NormalizedAuthorizationRequest (+ protocol_evidence, verbatim)
    │  (basis-adapters: construct_adapter_evidence(), implemented —
    │   67 test functions found in tests/test_evidence.py, not executed
    │   by this assessment — pure, deterministic: builds
    │   basis-adapter-evidence-v1 material, RFC-8785-canonicalizes it,
    │   computes a sha-256 digest)
    ▼
ConstructedAdapterEvidence (material + canonical_bytes + digest)
    │
    ╳  SEGMENT BOUNDARY — no component mints reference_id, assembles the
    │  final AdapterEvidenceReference, or authenticates to basis-gateway;
    │  this is where the two independently implemented segments below
    │  fail to connect into one producer lifecycle
    ▼
── Segment B: basis-gateway / basis-core (evaluation + audit) ─────────────
operation-aware gateway request  (POST /v1/evaluate/operation-aware,
    basis-gateway: implemented, feature-flagged OPERATION_AWARE_ENABLED,
    disabled by default)
    │  (basis-gateway: classify_operation_producer(), implemented —
    │   23 test functions found, not executed by this assessment —
    │   exact-match subject-ID allowlist, safe default UNTRUSTED)
    ▼
compose_operation_aware_request() (basis-gateway/core/operation_aware_composition.py:
    implemented — provenance-classifies every field, rejects caller-
    supplied is_trusted_operation_producer/producer_trust_classification,
    raises UntrustedOperationProducerContextError before any other
    composition step for an untrusted caller asserting producer-only fields)
    ▼
basis-core evaluate_operation_aware() (implemented — deterministic kernel
    decision, EvaluationTrace, AuditEvidence)
    ▼
GatewayAuditEvent (basis-gateway/audit/operation_aware_gateway_events.py:
    implemented — references AuditEvidence.evidence_id, never embeds it)
    ▼
basis-console Decision Simulator (implemented — presents the typed result
    in Operator/Training modes; explicitly a preview/simulation path, not
    a trusted-producer submission path)
    │
    ╳  STOP — nothing dispatches to a protocol endpoint
    ▼
[no protocol executor exists anywhere in the inspected repositories]
    ╳  STOP — nothing records execution evidence
```

### Where the lifecycle stops

Two independently implemented and test-covered segments exist: adapter normalization and evidence-material construction (`basis-adapters`), and gateway-to-core operation-aware evaluation and audit (`basis-gateway`/`basis-core`). No inspected runtime connects those segments into one end-to-end producer lifecycle. No source file in any of the seven inspected repositories calls both `basis_adapters.evidence.construct_adapter_evidence()` and a `basis-gateway` client in the same execution path. The gap is exactly the one `operation-producer-and-execution-boundary.md` §4 names: "the second and third stages are the gap this document names... binding workload identity to a request is a distinct act from constructing a safe evidence reference, and both are distinct from actually submitting the HTTP request under gateway-recognized authentication." This assessment's implementation inspection confirms that gap is still open in code, not merely in architecture prose.

A second, independent stop exists past authorization: no protocol executor, execution-result record, or execution-evidence producer exists in any inspected repository. `basis-adapters`' own architecture (`docs/architecture/basis-adapters.md`, restated in `operation-producer-and-execution-boundary.md` §2) states this as a design invariant, not an oversight: adapters are libraries, not daemons, with "no sockets, no packet parsing, no protocol stacks, no live protocol communication."

### Transitions documented but unimplemented

- Producer-to-gateway authentication beyond the subject-ID allowlist (architecture names four candidate mechanisms — mTLS, SPIFFE, OAuth client-credentials, "another" — and selects none; `operation-producer-and-execution-boundary.md` §13 Stage 4).
- `reference_id` minting and final `AdapterEvidenceReference` assembly by an operation-producer runtime (ADR-0007, "Ownership"; no such runtime exists in any inspected repository).
- Category-scoped producer trust — whether a producer trusted for `protocol_context` should automatically be trusted for `safety_context` (`operation-producer-and-execution-boundary.md` §3, §12) — remains all-or-nothing in `basis-gateway`'s implementation today (confirmed, §5.4).
- Execution dispatch, execution-result recording, and any execution-evidence contract (`ecosystem-contract-inventory.md` §4; `operation-producer-and-execution-boundary.md` §12).
- `basis-identity` production of the already-published `identity-evidence-reference` contract (confirmed absent in implementation, §5.7).

---

## 5. Per-Repository Findings

### 5.1 `basis-architecture`

**Producer responsibilities already described.** [`docs/architecture/operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md) (Status: Architecture-planning) is the single most direct treatment: it names the operation-producer runtime, protocol executor, and execution-evidence producer as logical roles (§2); states six distinct trust facts routinely conflated as "producer trust" (§3); defines a field-ownership table for the handoff (§4); states seven authorization-to-execution invariants (§5); defines a correlation model naming which component mints each identifier (§6); records a provenance/fact-ownership matrix reusing the existing closed `ProvenanceClassification` enum (§7); enumerates seventeen failure/degraded conditions with five preserved distinctions (§8); discusses three deployment topologies without prescribing one (§9); restates repository ownership, unchanged (§10); evaluates and rejects three of four alternatives for where the producer runtime should live, holding the fourth (a separate logical role, not necessarily a new repository) as a decision gate rather than a conclusion (§11); inventories what `basis-schemas`' twenty published contracts can and cannot already represent for this gap (§12); and proposes a ten-stage, decision-gated implementation sequence (§13).

**Intentionally unassigned responsibilities.** The document is explicit that it does not select a producer-authentication mechanism, does not decide whether the operation-producer runtime needs a new repository, does not finalize category-scoped producer trust, and does not define an execution-status vocabulary as a governed contract (§13, "Non-Goals," "Open Questions Deferred").

**ADRs already constraining producer ownership.** [ADR-0007](../adr/0007-adapter-evidence-construction.md) (Accepted) is the one ADR that has actually resolved part of this space: it fixes the `basis-adapter-evidence-v1` profile, adopts RFC 8785 exactly (not a future technology evaluation), and assigns material construction/canonicalization/digest to `basis-adapters` and reference assembly to the operation-producer runtime. [ADR-0006](../adr/0006-evaluation-orchestration-layer.md) (Status: **Proposed**, not Accepted, per its own header) is adjacent but orthogonal — it resolves a `basis-core`-internal kernel composition-boundary question (introducing the `evaluation` subpackage), not a producer-boundary question; this assessment's implementation inspection (§5.3) confirms `basis-core`'s `src/basis_core/evaluation/operation_aware/` package (`trace_assembly.py`, `engine.py`, `response_assembly.py`) exists and is populated, which is evidence the decision described by ADR-0006 has been acted on in practice even though the ADR document's own status field still reads "Proposed" — an internal documentation/implementation status disagreement worth flagging to whoever owns that ADR, not resolved by this assessment.

**Ecosystem contract inventory.** [`ecosystem-contract-inventory.md`](ecosystem-contract-inventory.md) does not list a producer, evidence-store, or execution contract at all — its twelve-row table (§2) covers action/resource vocabulary and composition, decision request/response, audit event, the reserved gateway evidence namespace, readiness, and policy loading. This is consistent with, not contradicted by, the later `operation-producer-and-execution-boundary.md`'s own §12 gap table, which independently names the same absence (execution-evidence record, producer-to-gateway trust contract, category-scoped capability, workload identity, and `basis-identity`'s non-production of `identity-evidence-reference`) using the newer document's own inventory method.

**Terminology.** `docs/glossary.md` was not found to contain entries for "operation-producer runtime," "protocol executor," or "execution-evidence producer" — `operation-producer-and-execution-boundary.md` itself states this explicitly ("naming a term here does not promote it to canonical status") in its closing section, so the absence is a documented, deliberate deferral, not an oversight this assessment is surfacing for the first time.

**Documents disagreeing or overlapping.** No direct contradiction was found between `operation-producer-and-execution-boundary.md`, the identity-to-operation roadmap, and the post-authentication roadmap; the boundary document explicitly narrows the roadmap's ten-phase program into "a first, concrete boundary" (its own "Relationship to Other Documents" section) and both roadmaps independently confirm they have not started (`identity-to-operation-contract-and-interoperability.md` Status line; `post-authentication-identity-activity-correlation-and-detection.md` "Required Status Restated"). Two internal inconsistencies were found and are worth flagging, neither resolved by this assessment: ADR-0006's document header still states "Status: Proposed" while its consequences (the `evaluation` subpackage) are confirmed implemented in `basis-core` (§5.3); and this repository's own `README.md` and `ROADMAP.md` both stated, prior to this correction pass, that `basis-adapters` "remains at `v0.1.0`" with no operation-aware integration, when `basis-adapters`' own `pyproject.toml`, README status banner, and `docs/release-readiness-v0.2.0.md` all confirm a released `v0.2.0` implementing ADR-0007 Stage 1 (§5.5). This assessment's `README.md` correction (in scope for this pass) fixes the latter; `ROADMAP.md` (out of scope for this pass) still carries the stale wording as of this writing.

**Questions requiring a new ADR.** Producer-to-gateway authentication mechanism selection; whether producer trust becomes category-scoped; whether the operation-producer runtime needs a new repository. All three are already named as open by `operation-producer-and-execution-boundary.md` itself (§13 Stage 4, §3/§12, §11) — this assessment's own §12 restates them with implementation evidence attached, but does not discover them for the first time.

### 5.2 `basis-schemas`

`basis-schemas` v0.2.2 (per `VERSION` and `CHANGELOG.md`) publishes twenty contracts under `schemas/`, all `lifecycle: experimental`.

**`adapter-evidence-reference` (v0.1.0).** Fully published (`schemas/adapter-evidence-reference/adapter-evidence-reference.yaml`). Required fields: `reference_id`, `evidence_digest` (nested `algorithm`/`value`), `adapter_source`, `redaction_classification` (closed five-value enum). Optional: `normalization_version`, `mapping_version`, `protocol`, `request_id`, `correlation_id`. `additional_properties: false`; the schema's own `invalid` examples explicitly demonstrate rejection of `raw_protocol_payload` and `unredacted_device_secret` fields. The field *shape* is structurally correct and unaffected by the finding below.

**Contract-metadata ownership conflict (confirmed).** The schema file's own informative `composition` block reads, verbatim:

```yaml
composition:
  produced_by: basis-adapters   # produces normalized adapter evidence and mints reference values
  consumed_by: basis-gateway, basis-core, basis-console  # reference evidence in requests, traces, audit, and explanations without retrieving raw material
```

That comment attributes *minting reference values* to `basis-adapters`. ADR-0007 (Accepted) assigns evidence-material construction, canonicalization, and digest generation to `basis-adapters`, but assigns final `AdapterEvidenceReference` *assembly* — including minting `reference_id`, supplying `adapter_source`, and assigning `redaction_classification` — to the operation-producer runtime specifically because, per ADR-0007's own rejected-alternatives list, having `basis-adapters` own the full reference "requires the library to hold deployment configuration and redaction policy it structurally must not acquire." This assessment's inspection of `src/basis_adapters/evidence.py` (§5.5) confirms the code itself honors ADR-0007, not the schema's comment: no function in that module mints `reference_id`, sets `adapter_source`, or assigns `redaction_classification`. The conflict is therefore confined to `basis-schemas`' embedded documentation metadata (an informative comment, not a validated field), not to the field-level contract shape and not to any implementation. This likely warrants a future `basis-schemas` metadata or documentation correction to bring the `composition` comment in line with ADR-0007's accepted ownership split; this assessment does not modify `basis-schemas` and does not conclude that a schema-*shape* release is required, since no inspected field definition itself conflicts with the accepted architecture.

**Producer identity and provenance.** Not represented in this or any published schema. No `basis-schemas` contract carries a producer workload identity, a producer-registration record, or a producer-authentication assertion shape.

**Operation submission envelope.** `operation-aware-decision-request` (v0.1.0) is the closest existing envelope; its own header states it explicitly: "does not assemble requests... does not retrieve evidence" (`schemas/operation-aware-decision-request/operation-aware-decision-request.yaml` lines 17–22). It carries `identity_evidence_reference` and `adapter_evidence_reference` as optional nested fields, reusing the two evidence-reference contracts rather than embedding new shape.

**Request/correlation identifiers.** `request_id` (required, non-empty) and `correlation_id` (optional, passed through verbatim, no format constraint) exist on `operation-aware-decision-request`, `adapter-evidence-reference`, and `identity-evidence-reference` alike. No idempotency-specific field (an idempotency key distinct from `request_id`/`correlation_id`) was found in any published schema.

**Persistence metadata.** Absent. No schema field represents where evidence material is stored, how long it is retained, or how a stored artifact is located for later retrieval.

**Execution-request / execution-result contracts.** Confirmed absent — no schema directory, file, or field anywhere under `schemas/` represents dispatch, acknowledgement, device-state confirmation, or execution failure. `operation-producer-and-execution-boundary.md` §12 independently reaches the same conclusion by architecture analysis; this assessment's direct schema-directory inspection corroborates it by implementation evidence.

**Lifecycle or operation-chain records.** Absent as a published contract; not confirmed to exist anywhere in the inspected repositories.

**Canonical examples covering a real producer submission.** `examples/operation-aware/compatibility/` contains five canonical scenarios (`allow-basic`, `deny-precedence`, `default-deny`, `not-applicable`, `invalid-policy-bundle`), each pairing a request and expected response fixture. These exercise the kernel's evaluation semantics; none of them exercises a trusted-producer-asserted, evidence-referencing submission end to end — the compatibility vectors validate `basis-core`'s evaluation behavior, not a producer's request-construction behavior.

This assessment does not interpret any of the above as proof that an implementation exists elsewhere merely because the schema shape is sufficient to represent it — the schema's own header language for both evidence-reference contracts is explicit that it "defines the reference shape only."

### 5.3 `basis-core`

**Operation-aware public models.** Confirmed present under `src/basis_core/`: `domain/operation_aware.py`, `domain/operation_aware_vocabulary.py`, `decisions/operation_aware.py`, `policy/operation_aware/`, `audit/operation_aware/evaluation_trace.py`, `enforcement/operation_aware.py`, and the `evaluation/operation_aware/` subpackage (`engine.py`, `trace_assembly.py`, `response_assembly.py`, `response.py`) that ADR-0006 describes.

**Evidence references entering evaluation.** `src/basis_core/domain/evidence.py` implements `IdentityEvidenceReference` and `AdapterEvidenceReference` as value objects. Its own module docstring states the boundary precisely: these models represent "a reference to evidence," not "proof that evidence is authentic," and explicitly enumerate what neither type does — retrieve the evidence, verify the source or producer, validate a signature, verify digest authenticity, or authenticate the producer. This is the same boundary `basis-schemas`' published contracts describe, implemented consistently rather than reinterpreted.

**Producer and subject information consumed.** `basis-core` consumes `subject_id`/`subject_roles`/`subject_attrs` and the optional evidence references as request context; it does not itself classify producer trust — that remains `basis-gateway`'s responsibility per the composition boundary confirmed in §5.4.

**Identifiers preserved or returned.** `trace_id` on `OperationAwareDecisionResponse`/`EvaluationTrace` and `AuditEvidence.evidence_id` are **not** generated inside `basis-core`. `basis-core`'s own `OperationAwareEnforcementPoint.evaluate()` (`src/basis_core/enforcement/operation_aware.py`) documents both as explicit keyword arguments in its docstring: "`trace_id`: caller-supplied; forwarded to the engine unchanged and never generated here" and "`evidence_id`: caller-supplied; forwarded to `assemble_audit_evidence` unchanged and never generated here." The kernel embeds whatever values it is given into its own artifact types; it does not decide their generation policy. Precisely which layer generates the values, and how, is resolved in §5.4 below — the answer is `basis-gateway`, via an injectable factory that defaults to `uuid.uuid4()`, not the kernel.

**Absence of persistence, transport, producer-authentication, adapter-invocation, or execution behavior.** No evidence of any of these was found in `src/basis_core/`. The kernel package structure (`domain`, `decisions`, `policy`, `audit`, `evaluation`, `enforcement`, `adapters`) matches `docs/kernel-boundary-rules.md`'s described import hierarchy; nothing inspected suggests the kernel boundary has been violated to accommodate producer or execution concerns.

**Whether any apparent gap belongs in core.** None of the confirmed gaps (§11) point toward `basis-core`. The kernel boundary is intact on this evidence, and this assessment does not recommend expanding it.

### 5.4 `basis-gateway`

**Trusted-producer configuration and classification.** Implemented, and test-covered per repository inspection (this assessment did not execute the suite; see §2). `src/basis_gateway/auth/operation_producer.py` defines `classify_operation_producer(subject, trusted_subject_ids)`, an exact-match, case-sensitive membership test against a configured allowlist (`GatewayConfig.operation_producer_subject_ids`, populated from `OPERATION_PRODUCER_SUBJECT_IDS`). The module's own docstring states the safe-default property explicitly: "with no `OPERATION_PRODUCER_SUBJECT_IDS` configured, no caller is a trusted operation producer," and enumerates what the module never does — infer trust from request-body fields, fall back to role membership, or perform wildcard/prefix/case-insensitive matching. `tests/test_operation_producer_trust.py` contains 23 test functions (found by inspection, not executed).

**Whether producer identity is authenticated independently from subject identity.** Not independently — `OperationProducerTrust.authorization_subject_id` and `operation_producer_subject_id` are, by the module's own docstring, "equal in this PR's only supported trust mechanism," an acknowledged implementation detail of the current transport rather than a claim that the two concepts have been given independent authentication paths. This is the exact limitation `operation-producer-and-execution-boundary.md` §3's provenance table already names.

**Ordinary callers prevented from asserting producer-only context.** Confirmed. `src/basis_gateway/core/operation_aware_composition.py` defines `UntrustedOperationProducerContextError`, raised (per its own docstring, line ~278–300 range) "before any other work" when an untrusted caller's request carries a producer-only field. The module also enumerates a closed `ProvenanceClassification` enum (`VERIFIED`, `GATEWAY_DERIVED`, `TRUSTED_PRODUCER_ASSERTED`, `UNTRUSTED_CALLER_ASSERTED`, `CONFIGURATION_DERIVED`, `UNAVAILABLE`) and classifies every request field against it — `identity_evidence_reference` and `adapter_evidence_reference` are both classified via `_producer_only_field_provenance()`, landing in `TRUSTED_PRODUCER_ASSERTED` when present from a trusted caller, `UNAVAILABLE` otherwise.

**Whether adapter evidence references are accepted end to end.** Accepted, provenance-classified, and passed through to kernel evaluation and audit — but never independently verified. This matches ADR-0007's ownership statement exactly ("`basis-gateway` authenticates and classifies the producer... does not regenerate the evidence or the digest").

**Whether the gateway persists evidence or only receives references.** No persistence code was found anywhere in `src/basis_gateway/`; a targeted search for storage/database/persistence terms in gateway source returned no substantive matches beyond the module and evaluator files that merely reference the concept in docstrings. The gateway receives and forwards references only, consistent with ADR-0007's "future evidence storage or evidence service" row being explicitly out of scope for every currently-implemented component.

**Whether the gateway mints any identifiers, precisely.** Yes, and the exact mechanism matters: `src/basis_gateway/core/operation_aware_evaluator.py` defines `_default_trace_id()` and `_default_evidence_id()`, two module-level functions each returning `str(uuid.uuid4())`. These are not called directly — they are the *default* values for `trace_id_factory: Callable[[], str]` and `evidence_id_factory: Callable[[], str]` parameters threaded through `build_operation_aware_evaluator()` into `OperationAwareGatewayEvaluator`, a frozen dataclass wrapper the gateway builds once at startup around the kernel's public `OperationAwareEnforcementPoint.for_bundle(bundle)`. At call time, `OperationAwareGatewayEvaluator.evaluate()` invokes `self._trace_id_factory()` and `self._evidence_id_factory()` (confirmed at the two call sites, lines 334–335) *before* calling the kernel, then passes the resulting strings into the kernel's `evaluate(trace_id=..., evidence_id=..., ...)` as the caller-supplied values its own docstring requires (§5.3). In short: identifier **generation** (the `uuid.uuid4()` call) is owned by `basis-gateway`'s integration layer, via an injectable factory pattern whose default implementation happens to be UUIDv4 — tests may inject deterministic factories instead. Identifier **embedding** (`trace_id` on `EvaluationTrace`, `evidence_id` on `AuditEvidence`) is owned by `basis-core`'s artifact types, which never generate the values themselves. Separately, `CorrelationMiddleware` (`src/basis_gateway/middleware/correlation.py`) generates a correlation ID unconditionally per HTTP request, ignoring any caller-supplied `X-Correlation-ID` header by documented policy (referenced in `api/routes.py` comments at two call sites) — a distinct identifier from `trace_id`/`evidence_id`, generated directly rather than through an injectable factory. The gateway does not mint `reference_id` for adapter or identity evidence — that remains, per ADR-0007, an operation-producer-runtime responsibility no component currently performs.

**Idempotency or replay protection.** No idempotency-key handling or replay-protection code was found in `src/basis_gateway/`. This is a confirmed absence, not merely an unexamined area — a targeted search for idempotency/replay terms in gateway source returned no matches.

**Execution or orchestration behavior.** None found. The gateway's own operation-aware endpoint (`POST /v1/evaluate/operation-aware`, feature-flagged via `OPERATION_AWARE_ENABLED`, disabled by default per `api/routes.py` comments) stops at enforcement and audit; `GatewayAuditEvent` (`src/basis_gateway/audit/operation_aware_gateway_events.py`) carries `enforcement_action` (the action selected) but no execution-result or execution-status field, consistent with the architecture document's observation that the audit schema "deliberately does not define an `enforcement_status`/`enforcement_result` field today."

**Remaining gateway-owned follow-up work.** Per the architecture document's own §13 Stage 4 (unchanged by this assessment): moving beyond the subject-ID allowlist toward a selected producer-authentication mechanism, and resolving or explicitly deferring category-scoped producer capability.

### 5.5 `basis-adapters`

**Public normalization surface.** Nine protocol adapters (BACnet, DNP3, IEC 61850, KNX, Modbus, MQTT, Niagara, OPC UA, REST) under `src/basis_adapters/`, each producing a `NormalizedAuthorizationRequest` per the published `docs/contracts/normalization-contract.md` and `docs/contracts/adapter-contract.md`.

**Evidence-construction surface.** Implemented, per ADR-0007's Stage 1, and test-covered per repository inspection (this assessment did not execute the suite; see §2). `src/basis_adapters/evidence.py` exports `construct_adapter_evidence(result, *, digest_algorithm=DIGEST_ALGORITHM_SHA256)`, a pure, deterministic, side-effect-free function (no clock, no randomness, no UUID generation, no network or filesystem access — confirmed by direct code inspection, not merely by docstring claim: the function body performs only structural validation, a per-protocol metadata projection against a closed allow-list table `_APPROVED_METADATA_KEYS`, RFC-8785 canonicalization via the `rfc8785` dependency, and `hashlib.sha256`). It fixes `EVIDENCE_PROFILE = "basis-adapter-evidence-v1"` and `CANONICALIZATION_PROFILE = "rfc8785"` as module-level constants, matching ADR-0007 exactly. `tests/test_evidence.py` is 1,355 lines containing 67 `def test_...` functions (found by inspection, not executed).

**Evidence fields adapters produce.** `AdapterEvidenceMaterial` (`evidence_profile`, `canonicalization_profile`, `protocol`, `action`, `resource_type`, `resource_id`, `protocol_evidence` projection) and `ConstructedAdapterEvidence` (`material`, `canonical_bytes`, `digest`). These are **not** the same object as the published `AdapterEvidenceReference` — no field named `reference_id`, `adapter_source`, or `redaction_classification` appears anywhere in `evidence.py`, confirming ADR-0007's ownership split is respected in the actual code, not just documented as intended.

**Evidence fields adapters intentionally do not produce.** `reference_id`, `adapter_source`, `redaction_classification`, request/correlation linkage — the module's own docstring states this exhaustively under "Ownership invariant," and the code contains no function capable of producing any of the four.

**Persistence, network communication, gateway submission, authentication, or execution.** None found anywhere in `src/basis_adapters/`. A targeted search for HTTP-client, socket, or database library usage in adapter source returned no results beyond the expected per-protocol client libraries adapters use to *represent* (not transmit) protocol operations during normalization.

**Accidental producer-like behavior.** None found. The evidence-construction module is a pure function operating on an already-completed `AdapterResult`; it does not call any network endpoint, does not know `basis-gateway` exists, and — per its own docstring — "does not call `basis-gateway`, does not authenticate a producer, and does not execute a protocol operation."

**A genuine adapter-owned gap.** None confirmed. `docs/operation-aware-handoff-alignment-plan.md` (referenced by both ADR-0007 and `operation-producer-and-execution-boundary.md` as the merged assessment that resolved this question) already concluded the bulk of the handoff surface is sufficient unchanged; this assessment's implementation inspection is consistent with that conclusion and did not surface a contradicting gap.

### 5.6 `basis-console`

**Operation-aware request support.** Implemented. `src/basis_console/gateway/operation_aware_models.py`, `src/basis_console/operation_aware_presentation.py`, and `src/basis_console/operation_aware_training.py` together implement a typed, mode-independent presentation layer over `GatewayClient.evaluate_operation_aware()` results, consumed by the Decision Simulator (`src/basis_console/simulator.py`, `build_operation_aware_simulation()`).

**Producer and subject displayed distinctly.** `operation_aware_presentation.py`'s own module docstring defines a closed `ContentSource` taxonomy (`SUBMITTED_INPUT`, `RETURNED_EVIDENCE`, `CONSOLE_EXPLANATION`, `FUTURE_CAPABILITY`) specifically so a submitted value is never presented as if the gateway had confirmed it. `operation_aware_training.py` carries a `producer_trust_classification` training key and `context_and_producer_trust_points`, confirming producer trust is explained, not silently folded into subject identity.

**Evidence references represented.** `RETURNED_EVIDENCE` is one of the four closed content-source categories; evidence-reference fields returned by the gateway are rendered under that category, distinct from console-authored explanation text.

**Whether the console can improperly assert trusted producer fields.** No evidence found that it can. A targeted search for `is_trusted_operation_producer` or `producer_trust_classification` as a value the console *sends* (as opposed to a training-mode label it *displays*) returned no matches in `simulator.py` or `gateway/client.py`; the console's simulator is explicitly documented (`views.py` module docstring) as building "a preview of the normalized request" rather than submitting as a trusted producer.

**Simulator traffic classification.** The simulator route's own module docstring states plainly: "The simulator POST always builds a preview." This is a confirmed, honest self-classification as simulation, not a claim this assessment is inferring from absence of a counter-example.

**Operation lifecycle or execution results represented.** Not found. No execution-result or lifecycle-state field appears in `operation_aware_models.py` or the presentation layer, consistent with no such contract or producer existing upstream to supply one.

**Future console work depending on unsettled backend contracts.** Execution-state display and any lifecycle/operation-chain exploration both depend on contracts (§11) that do not yet exist upstream — this is a downstream consequence of the gaps identified in §5.2 and §11, not a console-specific defect.

### 5.7 `basis-identity`

**Workload, service, machine, or agent identity.** `src/basis_identity/models/identity_context.py` defines `SubjectType.SERVICE = "service"` as one enum value, but this assessment found no accompanying establishment pipeline — no client-credentials flow, no mTLS/SPIFFE-shaped member on `AuthenticationProtocol` (which enumerates exactly `OIDC`, `SAML`, `OAUTH2`, `LDAP`, `LOCAL_DEV`, `UNKNOWN`), and no registration or revocation record type anywhere under `src/basis_identity/`. A targeted search for "workload," "machine identity," "service account," "client credential," "spiffe," and "mtls" across `src/` and `docs/` returned no substantive implementation matches beyond OIDC provider-configuration documentation unrelated to workload identity.

**Ability to authenticate a producer runtime.** Not found. `basis-identity`'s current scope (per its `docs/` and the module structure — `oidc/`, `sessions/`, `tokens/`, `normalization/`) is human/browser-flow OIDC identity federation and BASIS-local token issuance/verification for that flow; nothing inspected issues or validates a credential shaped for a machine caller authenticating to `basis-gateway` as an operation producer.

**Producer registration or revocation.** Confirmed absent. A targeted search for "revocation" and "registration" in `src/basis_identity/` matched only session-lifecycle files (`sessions/store.py`, `sessions/logout.py`, `sessions/lifecycle.py`) and token-signing files (`tokens/signing.py`) — session revocation for human logins, not producer credential revocation.

**Whether authority modes affect producer deployment.** `docs/architecture/identity-authority-modes.md` (in `basis-architecture`) defines federated/synchronized/standalone authority modes for human identity; no inspected `basis-identity` code path applies these modes to a producer or workload concept.

**Identity evidence supporting producer attribution.** `identity-evidence-reference` (v0.1.0) is published in `basis-schemas` and modeled in `basis-core`'s `domain/evidence.py` as `IdentityEvidenceReference`, but a targeted search for "identity_evidence_reference" and "IdentityEvidenceReference" anywhere under `basis-identity`'s `src/` and `docs/` returned **no matches**. This confirms, by direct implementation inspection, the alignment gap `operation-producer-and-execution-boundary.md` §12 already names architecturally: "the contract is published and `basis-gateway`'s schema already accepts it, but `basis-identity` neither produces nor references it anywhere in its current source."

**Whether producer identity belongs entirely, partly, or not at all within `basis-identity`.** No producer-identity implementation exists in `basis-identity` today — not workload authentication, not credential issuance, not registration, not revocation, not `identity-evidence-reference` production. Whether workload identity establishment belongs partly within `basis-identity` remains an architecture decision the evidence here does not settle; the architecture document's own framing (workload identity as a `basis-identity` gap, §12) is consistent with what this assessment found, but "no implementation exists" is a narrower and more precise claim than "producer identity belongs not at all in this repository," and this assessment makes only the narrower claim.

### 5.8 `basis-deploy`

Not inspected. No repository exists in the working environment. `ROADMAP.md` (`basis-architecture`) independently confirms this is expected: "`basis-deploy`... No repository exists yet." This assessment does not infer any `basis-deploy` implementation state from architecture documents.

### 5.9 Demo, PoC, lab, or integration repositories

Not inspected, with one partial exception found *inside* an already-inspected repository: `basis-gateway/demo/operation-aware/` (containing `README.md`, `run_demo.py`, `policy-bundles/`, `expected/`) is an in-repository bounded demonstration of the gateway's own operation-aware evaluation path — it is part of `basis-gateway`, not a separate repository, and this assessment did not find it invoking `basis-adapters` or performing any protocol dispatch; it appears scoped to demonstrating gateway-side evaluation and audit behavior only. The separate `basis-poc` repository referenced in `basis-architecture/README.md`'s "Relationship to BASIS PoC and basis-core" section was not available in the working environment and was not inspected; no implementation conclusion is drawn about it.

---

## 6. Cross-Repository Capability Matrix

Test counts in the Evidence column below reflect test functions found by inspection, not suites this assessment executed (see §2 Limitations).

| Capability | Architecture owner | Contract owner | Current implementation owner | Status | Evidence | Gap | Decision required |
| - | - | - | - | - | - | - | - |
| Workload/service authentication for a producer | Undecided (`operation-producer-and-execution-boundary.md` §13 Stage 4) | None published | None | **Absent** | Targeted search across `basis-identity` and `basis-gateway`; only `SubjectType.SERVICE` enum value exists (§5.7) | No establishment pipeline for machine/workload identity | Authentication/transport mechanism selection |
| Producer identity representation | `basis-architecture` (conceptual, §2) | None published | None | **Documented only** | `operation-producer-and-execution-boundary.md` §2 names "operation-producer runtime" as working vocabulary | No canonical model exists | Whether `basis-identity` introduces a workload-identity concept |
| Producer registration | Undecided | None | None | **Absent** | §5.7 targeted search | No registration record type anywhere | Registration mechanism |
| Trusted-producer configuration (gateway-side) | `basis-gateway` (`operation-producer-and-execution-boundary.md` §3, restated) | Gateway-internal config, not a schema | `basis-gateway` | **Implemented** (bounded) | `src/basis_gateway/auth/operation_producer.py`, 23 test functions found | All-or-nothing allowlist, not the ecosystem's final trust model per `ROADMAP.md`'s own words | Category-scoped trust; final authentication mechanism |
| Producer credential handling | Undecided | None | None | **Absent** | No credential-issuance code found in `basis-identity` or `basis-gateway` | — | Authentication mechanism selection |
| Producer-to-subject distinction | `basis-gateway` (implemented) | N/A | `basis-gateway` | **Implemented** | `operation_producer.py` module docstring + `OperationProducerTrust` dataclass separates `authorization_subject_id`/`operation_producer_subject_id` | Currently equal by construction (one trust mechanism only) | None immediate |
| Producer-to-adapter distinction | `basis-architecture` (§2) | N/A | Implicit (adapters never authenticate) | **Implemented** (by omission) | `basis-adapters` `evidence.py` docstring: "does not call basis-gateway, does not authenticate a producer" | — | None |
| Authorization to assert producer-only provenance | `basis-gateway` | N/A | `basis-gateway` | **Implemented** | `UntrustedOperationProducerContextError`, `operation_aware_composition.py` | — | None immediate |
| Producer revocation | Undecided | None | None | **Absent** | §5.7 | — | Revocation mechanism, once registration exists |
| Air-gapped producer identity support | `basis-architecture` (roadmap Phase 5, interoperability roadmap) | None | None | **Documented only** | `identity-to-operation-contract-and-interoperability.md` Phase 5 | — | Transport/store-and-forward requirements |
| Adapter selection / invocation | N/A (caller-owned per topology) | N/A | None (no producer runtime exists to invoke adapters) | **Absent** | No caller of `basis_adapters.evidence.construct_adapter_evidence()` found outside adapter test files | No runtime wiring `basis-adapters` output into `basis-gateway` input | Producer runtime existence/location |
| Normalized request construction | `basis-adapters` | `adapter-normalized-request-shape` (`ecosystem-contract-inventory.md` §3.6) | `basis-adapters` | **Implemented** | Nine adapters, `NormalizedAuthorizationRequest` | — | None |
| Canonical action/resource composition | `basis-gateway` | Contract-inventory candidate | `basis-gateway` | **Implemented** | `compose_action()`, `compose_resource_id()` (`ecosystem-contract-inventory.md` §3.3, §3.5, confirmed present) | Dual-purpose `resource_type` open question | Resource taxonomy (tracked elsewhere) |
| Deterministic evidence-material construction | ADR-0007 | N/A (pre-reference) | `basis-adapters` | **Implemented** | `construct_adapter_evidence()`, 67 test functions found | — | None |
| Evidence canonicalization (RFC 8785) | ADR-0007 | N/A | `basis-adapters` | **Implemented** | `rfc8785.dumps()` call in `evidence.py` | — | None |
| Digest generation | ADR-0007 | `evidence_digest_shape` (published) | `basis-adapters` | **Implemented** | `hashlib.sha256` in `evidence.py` | — | None |
| `AdapterEvidenceReference` assembly (`reference_id`, `adapter_source`, `redaction_classification`) | ADR-0007 (assigns to producer runtime) | Contract published, but its own embedded `composition` comment conflicts with ADR-0007 (§5.2) | None | **Absent** (implementation); **Ambiguous** (contract metadata) | No code anywhere mints `reference_id` or assigns `adapter_source`/`redaction_classification`; schema comment attributes minting to `basis-adapters` | No operation-producer runtime exists; `basis-schemas` metadata not yet reconciled with ADR-0007 | Producer runtime existence/location; separately, a `basis-schemas` documentation correction |
| Evidence storage/persistence | Undecided ("future evidence storage," ADR-0007 Consequences) | None | None | **Absent** | No persistence code in `basis-gateway` or `basis-adapters` | — | Evidence-storage architecture (not started) |
| Opaque reference minting, uniqueness, collision handling | ADR-0007 (deployment-local by default) | Partial (`reference_id` shape published) | None (no minter exists) | **Contract published**, **Absent** implementation | `adapter-evidence-reference.yaml` field spec; no minting code | — | Ecosystem-global uniqueness, ever? (ADR-0007 deferred) |
| Gateway acceptance of `AdapterEvidenceReference` | `basis-gateway` | N/A | `basis-gateway` | **Implemented** | `operation_aware_composition.py`, `TRUSTED_PRODUCER_ASSERTED` classification | Accepted but never independently verified (by design) | None |
| Gateway rejection of unauthorized producer assertions | `basis-gateway` | N/A | `basis-gateway` | **Implemented** | `UntrustedOperationProducerContextError` | — | None |
| Semantic preflight / schema validation | `basis-gateway` | `basis-schemas` (structural) | `basis-gateway`/`basis-schemas` | **Implemented** | Composition module + published schema constraints | — | None |
| Policy bundle selection | `basis-gateway`/`basis-core` | Provisional contract (`ecosystem-contract-inventory.md` §3.13) | `basis-gateway` | **Implemented** (provisional) | `policy/operation_aware_loader.py` | Policy provenance/lifecycle open | Policy lifecycle (tracked elsewhere) |
| Core invocation | `basis-gateway` → `basis-core` | N/A | `basis-gateway` | **Implemented** | `core/operation_aware_evaluator.py` | — | None |
| Decision response handling | `basis-gateway` | Published | `basis-gateway` | **Implemented** | Routes + evaluator | — | None |
| Correlation ID handling | `basis-gateway` | N/A (gateway-owned policy) | `basis-gateway` | **Implemented** | `CorrelationMiddleware` | Caller-supplied header explicitly ignored (by policy) | None |
| Audit emission | `basis-gateway`/`basis-core` | Emerging (`ecosystem-contract-inventory.md` §3.10) | `basis-gateway`/`basis-core` | **Implemented** | `GatewayAuditEvent`, `AuditEvidence` | Persistence, tamper resistance open | Audit persistence (tracked elsewhere) |
| Fail-closed behavior | `basis-gateway`/`basis-adapters` | N/A | Both | **Implemented** | `AdapterResult.success` gate in `evidence.py`; gateway fail-closed audit events referenced in architecture doc | — | None |
| Request/correlation/evaluation identifiers | Mixed (§6 of boundary doc) | Published (partial) | `trace_id`/`evidence_id` generation: `basis-gateway` (injectable factory, default `uuid.uuid4()`); embedding: `basis-core` artifact types; `correlation_id`: `basis-gateway` middleware | **Implemented** | `operation_aware_evaluator.py` factories + `OperationAwareEnforcementPoint.evaluate()` docstring ("caller-supplied... never generated here"); see §5.3, §5.4, §4 above | `reference_id` for evidence not minted by anyone | Producer runtime |
| Idempotency key / replay protection | Undecided | None published | None | **Absent** | Targeted search of `basis-gateway` source, no matches | — | Interoperability roadmap Phase 5 |
| Execution-request abstraction / protocol executor | Undecided | None | None | **Absent** | No component found anywhere; `basis-adapters` explicitly excludes this by design | — | Execution-plane architecture (not started) |
| Execution result / execution-result evidence | Undecided | None | None | **Absent** | `ecosystem-contract-inventory.md` §4 confirms; no code found | — | Execution-evidence contract (gated on reference implementation) |
| Console operation-aware simulation | `basis-console` | N/A | `basis-console` | **Implemented** | `simulator.py`, `operation_aware_presentation.py` | Explicitly simulation-only, not producer submission | None |
| Console producer/subject distinction | `basis-console` | N/A | `basis-console` | **Implemented** | `ContentSource` taxonomy | — | None |
| Console evidence-reference display | `basis-console` | N/A | `basis-console` | **Implemented** | `RETURNED_EVIDENCE` content source | — | None |
| Console execution-state display | Undecided (depends on execution contract) | None | None | **Absent** | No lifecycle/execution field in presentation models | Upstream contract does not exist | Gated on execution-evidence contract |
| Deployment packaging for a producer | `basis-deploy` (future) | None | None | **Not inspected** | No `basis-deploy` repository | — | `basis-deploy` establishment |
| `basis-identity` production of `identity-evidence-reference` | `basis-identity` (implied by contract's own `produced_by`) | Published | None | **Absent** | Targeted search, zero matches in `basis-identity` source | Alignment gap between published contract and intended producer | Whether/how `basis-identity` begins producing it |

---

## 7. Contract Inventory

| Contract | Version / lifecycle | Architecture source | Consuming repositories (declared) | Canonical examples? | Public models align? | Runtime support? | Notes |
| - | - | - | - | - | - | - | - |
| `adapter-evidence-reference` | 0.1.0 / experimental | `operation-aware-schema-readiness-plan.md` (ADR-0005) | `basis-gateway`, `basis-core`, `basis-console` (per schema's own `composition` block); `basis-adapters` is the declared *producer* of underlying material, not the reference itself | Yes (schema-file `valid`/`invalid` examples); no end-to-end producer-submission compatibility scenario | Yes — `basis-core`'s `AdapterEvidenceReference` domain model matches the schema fields | Partial — `basis-adapters` produces the material and digest (Stage 1 of ADR-0007), but no component assembles the final reference object | Missing contract: nothing represents the assembly step itself as a runtime API. **Also confirmed:** the schema's own embedded `composition` comment says `basis-adapters` "mints reference values," which conflicts with ADR-0007's accepted assignment of `reference_id` minting to the operation-producer runtime — a metadata/documentation misalignment, not a field-shape defect (§5.2) |
| `identity-evidence-reference` | 0.1.0 / experimental | Same | `basis-gateway`, `basis-core`, `basis-console`; `basis-identity` is the declared producer | Yes (schema-file examples) | Yes — `basis-core`'s `IdentityEvidenceReference` matches | **No** — `basis-identity` produces nothing matching this contract | Confirmed alignment gap (§5.7); published contract with zero producing implementation |
| `operation-aware-decision-request` | 0.1.0 / experimental | ADR-0001, ADR-0005 | `basis-gateway` (assembles), `basis-core` (evaluates) | Yes — five canonical compatibility scenarios | Yes | Yes (gateway assembly, kernel evaluation both implemented) | Sufficient shape for producer-only fields per its own header |
| `operation-aware-decision-response` | 0.1.0 / experimental | ADR-0002/0003 | `basis-core`, `basis-gateway`, `basis-console` | Yes | Yes | Yes | Structural separation of `outcome`/`evaluation_status` reusable pattern for a future execution contract |
| `evaluation-trace`, `trace-rule-evidence`, `audit-evidence`, `gateway-audit-event` | Various / experimental | ADR-0003, ADR-0006 companion doc | `basis-core`, `basis-gateway`, `basis-console` | Yes | Yes | Yes | Audit persistence and gateway/kernel correlation remain open per `ecosystem-contract-inventory.md` §4 |
| `policy-bundle`, `policy-rule`, `policy-condition` | Various / experimental | ADR-0004 | `basis-core`, `basis-gateway` | Yes | Yes | Yes | Policy lifecycle management explicitly out of scope |
| Producer-to-gateway authentication/trust assertion | Not published | Named as a gap, not a contract (`operation-producer-and-execution-boundary.md` §12) | None | No | N/A | No | **Missing contract** — no schema exists because no mechanism is selected |
| Execution-request / execution-result | Not published | Named as a gap (`ecosystem-contract-inventory.md` §4; boundary doc §12) | None | No | N/A | No | **Missing contract** — gated on a reference implementation existing first |
| Category-scoped producer capability | Not published | Named as a gap (boundary doc §3, §12) | None | No | N/A | No | **Ambiguous ownership** — schema-and-gateway-configuration question together, not schema alone |
| Workload/machine identity | Not published | Named as a `basis-identity` gap (boundary doc §12) | None | No | N/A | No | **Ambiguous ownership** — constrains, but is not itself, a future producer-authentication schema |

---

## 8. Trust-Concentration Analysis

The options below analyze *complete producer authority* concentrated in each existing repository — a different, more extreme scenario than a narrowly bounded producer component being colocated inside one of them (§9, §10 draw that distinction explicitly). Read each option as "what if this repository absorbed everything," not as a candidate for the bounded first slice.

**All producer responsibilities in `basis-gateway`.** Authentication authority: gateway already authenticates ordinary callers; extending this to producers is a natural, low-friction extension of existing code. Normalization authority: none today — the gateway has no protocol-parsing capability, and giving it one would reintroduce exactly the protocol-specific logic the adapter/gateway split exists to keep out of a shared, multi-tenant-facing component (`operation-producer-and-execution-boundary.md` §11, alternative 4, already rejected on this basis). Evidence authority: partial today (accepts references); full evidence-material construction would duplicate `basis-adapters`' now-implemented `construct_adapter_evidence()` (§5.5). Persistence authority: none today; adding it would give one component both enforcement and durable evidence custody. Submission/authorization/audit authority: already held. Execution authority: none, and none should be added per the architecture document's explicit rejection of embedding adapters in the gateway. **Concentration risk:** high if normalization or execution were added; low-to-moderate if only producer authentication were added, since that is closest to the gateway's existing authentication responsibility. **Compatibility with current boundaries:** authentication extension is compatible; normalization or execution extension is not.

**All producer responsibilities in `basis-adapters`.** Rejected as a default direction by the architecture document itself (§11, alternative 1): `basis-adapters`' released, non-negotiable non-responsibilities state "no live protocol communication... adapters are libraries, not daemons." This assessment's implementation inspection corroborates that the current `evidence.py` module is deliberately side-effect-free and stateless — adding credential-holding, network-facing responsibility would be a first-time architectural violation, not an extension of an existing capability. Concentration risk: the adapter library would become the single most trusted, most network-exposed component in the ecosystem, contradicting its current design as a portable, side-effect-free normalization library usable in both embedded and networked topologies.

**All producer responsibilities in `basis-identity`.** `basis-identity` today authenticates only human/browser OIDC flows; it has no workload-identity establishment pipeline (§5.7). Concentration risk if extended: identity issuance and producer-context assertion (location, device, protocol evidence) are conceptually distinct — `basis-identity`'s current architecture role is "identity engine and federation boundary," not operational context assertion. Folding producer responsibilities in would blur that boundary. Compatibility: workload/machine identity *authentication* is a plausible `basis-identity` extension (per the architecture document's own framing of the gap, §12); operational context assertion and evidence-reference assembly are not.

**All producer responsibilities in `basis-deploy`.** No repository exists. The architecture document is explicit (§10): "`basis-deploy`... packages and configures whatever runtime performs the operation-producer or protocol-executor role; it does not itself perform protocol dispatch or own execution semantics." Placing producer *logic* (as opposed to packaging) in `basis-deploy` would violate its stated responsibility outright.

**All producer responsibilities in `basis-console`.** Console invariant (restated across the ecosystem-contract-inventory, the identity-to-operation roadmap, and `basis-console`'s own architecture doc) prohibits the console from becoming an enforcement or execution authority. The console's current implementation (§5.6) already respects this — its simulator explicitly self-classifies as preview-only. Assigning producer responsibilities here would be a direct, confirmed-by-implementation violation of an invariant the console currently honors.

**A new producer runtime (standalone repository or logical component).** Authentication, normalization-consumption, evidence-assembly, and submission authority would concentrate in one new, purpose-built, narrowly-scoped component — the direction the architecture document itself leans toward (§11, alternative 2) without committing to a new repository specifically. Concentration risk is bounded by design: this component would hold a producer credential and evidence-assembly logic, nothing more; persistence, authorization, and execution would remain elsewhere. Compatibility: highest of all options, since it requires no existing repository to acquire responsibility outside its current charter.

**Multiple cooperating services (split producer/evidence/execution).** Lowest single-point concentration; highest operational and deployment complexity. The architecture document names this as alternative 3 (protocol-specific runtime hosts) without endorsing it as more than a plausible long-term end state (its own "premature to commit to now" language, quoted in full at §9 option 6). This assessment's implementation inspection found nothing to contradict that — no partial multi-service producer implementation exists to evaluate.

No option was selected merely because it requires fewer repositories; each is evaluated above against the authority categories the assignment specifies, and every finding traces to a specific architectural statement or a specific piece of inspected code.

---

## 9. Repository Ownership Options

Each option below is a candidate location for a narrowly bounded producer *component* (authentication plus evidence-reference assembly, per §10), not a proposal to relocate an existing repository's entire charter — that broader, rejected scenario is analyzed separately in §8.

1. **Gateway-owned producer behavior.** Benefit: reuses existing authentication infrastructure and request-composition code already covered by tests in the inspected repository (not re-run by this assessment; §2). Risk: pressure to also absorb normalization or evidence-assembly logic over time, eroding the adapter/gateway split. Violated/strained boundary: none if scoped strictly to authentication; the gateway's own architecture already anticipates *authenticating* a producer (§3 of the boundary doc) — it does not anticipate the gateway *becoming* the producer. Prerequisite contracts: a producer-authentication mechanism (§13 Stage 4). Deployment implications: none beyond existing gateway deployment. Execution remains separate: yes, unaffected.

2. **Adapter-host-owned producer behavior** (a runtime that hosts adapter libraries and adds producer responsibilities around them). Benefit: keeps adapter-consumption logic close to the library that produces evidence material. Risk: without careful scoping this becomes indistinguishable from "expand `basis-adapters` itself" (rejected, §11 alternative 1) unless it is a genuinely separate component that merely *depends on* `basis-adapters` as a library, matching the architecture document's own alternative 2 framing. Prerequisite contracts: producer authentication; evidence-reference assembly logic (which does not exist as code anywhere today — this assessment confirms `reference_id` minting has zero implementations). Deployment implications: a new deployable process, but not necessarily a new repository — could live in `basis-gateway`'s own repository as an optional client, per the architecture document's own suggestion (§11). Execution remains separate: yes, if scoped to submission only.

3. **Identity-integrated producer behavior.** Benefit: colocates producer credential issuance with the ecosystem's existing identity-federation component. Risk: `basis-identity` contains partial service-identity vocabulary (`SubjectType.SERVICE`) but no implemented workload-identity establishment, credential issuance, registration, or revocation pipeline (§5.7 — confirmed absence of implementation, not absence of vocabulary); this option requires `basis-identity` to build that pipeline before it could plausibly host producer submission logic too. Prerequisite contracts: workload/machine identity concept in `basis-identity` (an open question the architecture document names, §12, without resolving). Deployment implications: unclear until the workload-identity gap is closed. Execution remains separate: yes, if identity-integration is scoped to authentication only.

4. **Deployment-hosted but separately implemented producer runtime.** Benefit: `basis-deploy` (once it exists) packages the runtime without owning its semantics — consistent with the architecture document's own restated boundary (§10). Risk: `basis-deploy` does not exist yet, so this option has no near-term path independent of establishing that repository first. Prerequisite contracts: producer authentication, evidence assembly, and `basis-deploy`'s own establishment. Deployment implications: bundles cleanly once `basis-deploy` exists. Execution remains separate: yes.

5. **Standalone producer runtime (new repository).** Benefit: cleanest boundary — a new, narrowly-scoped, purpose-built component with no existing charter to strain. Risk: adds a repository, a release cadence, and a maintenance surface before a single reference implementation has validated that the architecture document's roles and rules (§2–§9) are sufficient. The architecture document itself treats this as a later decision gate (§13 Stage 5), not a default. Prerequisite contracts: producer authentication mechanism; evidence-reference assembly logic. Deployment implications: new deployable artifact, new packaging work for `basis-deploy` once it exists. Execution remains separate: yes, by construction — this option does not bundle a protocol executor.

6. **Split producer, evidence, and execution services.** Benefit: narrowest per-service trust and failure boundary; no single component holds producer credential, evidence custody, and execution capability together. Risk: highest operational complexity, and — per the architecture document's own words — "premature to commit to now, since no reference implementation of even one conforming runtime exists yet to validate that the shared contract is sufficient" (§11). Prerequisite contracts: all of the above, plus an execution-result contract (itself gated on a reference implementation, per §13 Stage 6). Deployment implications: most complex; multiple deployable artifacts, multiple trust boundaries to secure independently. Execution remains separate: yes, definitionally — this is the option that makes the separation most explicit.

---

## 10. New-Repository Necessity Assessment

**Can an existing repository own the complete producer lifecycle without violating its established purpose?** No. Every existing repository's current, implemented architecture excludes at least one required producer responsibility: `basis-gateway` has no protocol-normalization capability; `basis-adapters` is a non-negotiable library with no network or credential-holding capability; `basis-identity` has no implemented workload-identity establishment, credential-issuance, registration, or revocation pipeline (though it carries partial service-identity vocabulary, §5.7); `basis-console` is bound by an already-implemented, already-honored console invariant; `basis-deploy` does not exist and, per its own stated future scope, would not own semantics even once it does.

This is a distinct question from whether a new, separately bounded producer *component* could initially be colocated inside one of these repositories rather than owning that repository's entire existing charter — see the component-versus-repository distinction below.

**Which responsibilities can safely coexist?** Producer authentication and evidence-reference assembly can plausibly coexist in one component (both are, by ADR-0007's own ownership table, "operation-producer runtime" responsibilities already grouped together architecturally). Evidence-material construction should *not* be duplicated into that component — it is already implemented and correctly scoped inside `basis-adapters` (§5.5), and ADR-0007 explicitly rejected the alternative of having the producer runtime construct material itself ("risks normalization drift").

**Which responsibilities should remain separate?** Execution (protocol dispatch) should remain architecturally separate from producer submission, per the architecture document's own separated-topology discussion (§9) and its explicit statement that a compromised or confused executor accepting commands without checking disposition binding "reintroduces exactly the 'confused deputy' abuse case the interoperability roadmap already names." Evidence persistence should be treated as a distinct *responsibility* from reference assembly — ADR-0007 names "future evidence storage or evidence service" as a distinct future role, not a producer-runtime responsibility by default — but this is a responsibility distinction, not a mandatory deployment-topology decision: whether persistence and assembly may initially coexist within one narrowly scoped runtime process, versus requiring a separate service from day one, remains an ADR and reference-implementation decision this assessment does not resolve. What the evidence does support is narrower and non-negotiable regardless of topology: evidence custody, reference minting, authentication, authorization, and execution must not be casually collapsed into one unbounded authority — that is a concentration-of-trust concern (§8), not a process-count requirement.

**Does a distinct credential, scaling, persistence, or failure boundary justify a separate runtime?** On current evidence: plausibly, but not yet provably. A producer runtime would hold a credential no other component holds today (a workload credential distinct from the gateway's own service credentials and distinct from any human OIDC identity), which is a genuine, distinct security boundary. Scaling and failure-domain arguments are weaker on current evidence — no deployment-scale or load data exists in any inspected repository to evaluate them against, because no reference implementation exists yet to generate that data.

**Would a new repository reduce trust concentration or merely add operational complexity?** It would reduce concentration relative to placing producer behavior inside `basis-gateway` or `basis-adapters` (§8), but the architecture document's own §11 already reaches this qualitative conclusion without needing a new implementation inspection to confirm it; this assessment's contribution is confirming that no implementation currently exists to weigh against that qualitative judgment — the question remains genuinely open, not merely undecided by inertia.

**Is the evidence strong enough to recommend a new repository now?** No. No reference implementation exists anywhere in the inspected repositories to validate that the architecture document's roles and trust rules are sufficient in practice — the exact precondition the architecture document itself sets for this decision (§13 Stage 5: "a first, narrow implementation of the operation-producer runtime role... used to validate that the roles and rules this document defines are actually sufficient before generalizing them" must precede the repository decision, not follow it).

**Provisional conclusion:** **Indeterminate pending specific missing evidence** — specifically, pending (a) a selected producer-to-gateway authentication mechanism (§13 Stage 4) and (b) a bounded reference implementation of the operation-producer runtime role, scoped to one protocol and one deployment topology (§13 Stage 5), whose repository placement (existing-repository client library versus new repository) is itself the explicit decision gate that reference implementation is meant to resolve. This is not a disagreement with `operation-producer-and-execution-boundary.md`'s own framing — it is a confirmation, from independent implementation inspection, that the preconditions that document names for resolving the question are still unmet in code, not merely undocumented as met.

This conclusion is non-normative and subject to the next ADR.

---

## 11. Confirmed Gaps

**Architecture gaps.** Category-scoped producer trust (trusted for `protocol_context` but not `safety_context`) is named but unresolved (`operation-producer-and-execution-boundary.md` §3, §12). Producer-to-gateway authentication/transport mechanism is named but unselected (§13 Stage 4). ADR-0006's document status ("Proposed") appears stale relative to its consequences being implemented in `basis-core` (§5.1) — worth reconciling, not itself a producer-boundary gap.

**Schema gaps.** No producer-to-gateway authentication/trust-assertion contract. No execution-request or execution-result contract. No category-scoped producer-capability contract. No idempotency-key field distinct from `request_id`/`correlation_id`. No evidence-persistence/retrieval-locator contract. (All confirmed absent by direct `schemas/` directory inspection, §5.2, §7.)

**Contract-metadata gaps.** `adapter-evidence-reference.yaml`'s own embedded `composition` comment attributes reference-value minting to `basis-adapters`, conflicting with ADR-0007's accepted assignment of that responsibility to the operation-producer runtime (confirmed, §5.2, §7). The field-level contract shape is unaffected; this is a documentation/metadata alignment gap in `basis-schemas`, not a schema-shape gap.

**Public-model gaps.** `basis-core`'s `AdapterEvidenceReference`/`IdentityEvidenceReference` models are complete relative to the published schemas (no gap); the gap is entirely upstream (nothing assembles a real reference to hand to these models in a non-test context) and downstream (no execution-result model exists to receive).

**Gateway gaps.** No idempotency/replay protection (confirmed absent, §5.4). No evidence persistence (confirmed absent — expected, per ADR-0007's ownership table, but worth stating as a gap relative to a hypothetical end-to-end system rather than relative to the gateway's own current scope). Category-scoped producer trust unimplemented (all-or-nothing allowlist only).

**Producer-runtime gaps.** The runtime itself does not exist. No component mints `reference_id`. No component authenticates as a workload/machine producer. No component invokes both `basis-adapters` and `basis-gateway` in sequence.

**Persistence gaps.** No evidence-storage implementation anywhere. No stored-evidence locator semantics. No retention or deletion behavior.

**Identity gaps.** No workload/machine identity establishment pipeline in `basis-identity` (confirmed absent, §5.7). No production of the already-published `identity-evidence-reference` contract by `basis-identity` (confirmed absent — a genuine implementation/contract alignment gap, not merely an unstarted feature).

**Deployment gaps.** `basis-deploy` does not exist; no packaging, secrets-handling, startup-ordering, or health/readiness model exists for a future producer runtime.

**Console gaps.** No execution-state or lifecycle representation (expected, given no upstream contract exists to display).

**Execution-plane gaps.** No protocol executor. No execution-evidence producer. No execution-result vocabulary, governed or otherwise (the interoperability roadmap's five-state sketch is explicitly roadmap language, not a governed contract, per the architecture document's own §2 caveat).

**Test and conformance gaps.** No end-to-end test anywhere in the inspected repositories exercises a real (or faithfully simulated) producer submission calling both `basis-adapters`' evidence construction and `basis-gateway`'s trusted-producer classification in sequence. `basis-adapters`' 67 evidence tests and `basis-gateway`'s 23 producer-trust tests are both real and substantive, but they test their own repository's half of the handoff in isolation — no conformance scenario in `basis-schemas`' five canonical compatibility vectors exercises a trusted-producer-asserted, evidence-referencing request end to end (confirmed, §5.2).

---

## 12. Open Questions

The following is the smallest set of questions this assessment's evidence suggests the next ADR must answer, restated with the implementation evidence now attached to each (all were already named as open by `operation-producer-and-execution-boundary.md`; none are newly discovered by this assessment):

1. **Producer authentication.** Which mechanism (mTLS, SPIFFE, OAuth client-credentials, or another) establishes a producer workload identity toward `basis-gateway`, replacing or extending the current subject-ID allowlist? Confirmed unresolved in code as well as architecture (§5.4, §5.7).
2. **Producer registration and revocation.** What process registers a new producer's credential and what process revokes it? Confirmed no implementation exists anywhere (§5.7).
3. **Subject-versus-producer attribution.** Should the current implementation detail — that `authorization_subject_id` and `operation_producer_subject_id` are equal under the one supported trust mechanism — remain acceptable once a second mechanism exists, or does the architecture require them to diverge from day one of a new mechanism? (§5.4)
4. **Evidence persistence.** Is evidence-material persistence required before an `AdapterEvidenceReference` can be considered trustworthy at retrieval time, and if so, who owns it? Confirmed entirely unimplemented (§5.4, §11).
5. **Reference minting.** Does `reference_id` remain deployment-local by default indefinitely, or does some future state require ecosystem-global uniqueness? No implementation exists to inform this yet (§5.5, ADR-0007 "Deferred decisions").
6. **Redaction ownership in practice.** `redaction_classification` is assigned by the operation-producer runtime under deployment policy per ADR-0007 — what concretely constitutes "deployment policy" once a real runtime exists to apply it?
7. **Submission envelope.** Is `operation-aware-decision-request` sufficient as the producer-to-gateway envelope once real authentication exists, or does producer authentication itself require a distinct, separately-versioned envelope (a "producer-trust assertion contract," per `ecosystem-contract-inventory.md`'s and the boundary document's shared gap finding)?
8. **Correlation.** Once a real producer runtime exists, does it need its own correlation-ID generation policy distinct from the gateway's current "always generate, ignore caller-supplied header" policy, given that a producer runtime is a different trust class than an arbitrary external caller?
9. **Idempotency and retries.** No idempotency-key concept exists anywhere in the inspected implementation (§5.4, §5.2) — is one required before a producer runtime is considered safe to build against, or can it be deferred to a later phase per the interoperability roadmap's own Phase 5 sequencing?
10. **Stale decisions.** The architecture document names "execution must never occur before the authoritative disposition is received" as an invariant (§5) — what concrete mechanism (a decision expiration window, a nonce, something else) would a real producer runtime need to enforce this, given none exists to enforce it today?
11. **Execution handoff.** Should the reference implementation (§13 Stage 5) combine producer submission and execution in one process (§9's "combined producer/executor" topology) or deliberately separate them from the start, given the separated-topology confused-deputy risk the architecture document already names?
12. **Execution-result evidence.** Is the interoperability roadmap's five-state sketch (`authorized-but-not-attempted`, `attempted-and-completed`, `attempted-and-failed`, `partially-applied`, `execution-status-unavailable`) sufficient as a starting vocabulary, or will a bounded reference implementation surface different states? Cannot be answered without that implementation existing (§13 Stage 6).
13. **Deployment boundaries.** Once `basis-deploy` exists, does it package a combined or separated producer/executor topology by default, and does that default choice have security consequences the architecture document's §9 topology discussion already flags?
14. **Explicit repository exclusions.** Should the next ADR explicitly rule out any of the six options in §9 now (for example, ruling out `basis-console` or `basis-deploy` hosting producer logic directly, both of which this assessment's implementation inspection confirms would violate an already-implemented invariant), even while leaving the remaining options open pending a reference implementation?

---

## 13. Recommended ADR Scope

**Decisions the ADR must make.** Producer-to-gateway workload authentication and admission (§13 Stage 4 of the boundary document is the correct, narrower next decision — not the repository-placement question, which that same document explicitly gates on a reference implementation that does not yet exist). The specific trust fact the chosen mechanism establishes (a verified workload identity, a registered client, a certificate-bound identity — whichever the mechanism implies). How the authenticated producer remains distinct from the authorization subject once a second mechanism exists, rather than the two remaining equal by construction as they are today under the single subject-ID-allowlist mechanism (§5.4). How producer registration and revocation are represented at the architecture level (not necessarily implemented, but named as concepts the mechanism must support). How `basis-gateway`'s current subject-ID allowlist relates to the selected mechanism — replaced, layered alongside it during a transition, or retained for a narrower purpose. The minimum bounded reference slice authorized after acceptance (§13 Stage 5 language), and that slice's *provisional* implementation location — without deciding permanent repository ownership.

**Decisions it should deliberately defer.** Final, permanent repository placement for the operation-producer runtime (explicitly gated on Stage 5's reference implementation, per the architecture document's own sequencing). Permanent evidence-storage topology (§10's persistence-versus-assembly distinction). Execution-plane design and execution-result vocabulary (explicitly Stage 6+, gated on Stage 5). Console workflow and deployment packaging (both downstream of a working reference slice). A mandatory producer-trust payload schema (see below — contingent on the mechanism, not assumed). Category-scoped producer trust, *unless* the authentication decision cannot be expressed safely without it — for example, if the selected mechanism inherently carries per-category scope (such as OAuth client-credential scopes), the ADR may need to state how that maps onto today's all-or-nothing model even while deferring a full category-scoped redesign.

**Repositories whose maintainers or contracts are affected.** `basis-gateway` (admission logic would need to accept whatever new mechanism is selected, alongside the existing allowlist during any transition). `basis-identity` (if the selected mechanism involves `basis-identity`-issued workload credentials — not decided by this assessment, but a live possibility per §5.7 and §9 option 3). `basis-adapters` is *not* expected to be affected — its evidence-construction surface is already complete for this purpose per ADR-0007 and this assessment's confirmation that it is implemented correctly (§5.5).

**Schemas likely to follow — contingent, not assumed.** Whether a new cross-repository payload contract is needed at all depends entirely on which mechanism the ADR selects. An mTLS-based mechanism may need only a certificate/workload-identity profile and a gateway-side trust-anchor configuration contract, not a request-body schema. A SPIFFE-based mechanism implies a workload-identity document format largely external to `basis-schemas`. An OAuth-client-credentials mechanism implies a claims profile more than a new payload envelope. A gateway-registration-only mechanism may need no new `basis-schemas` contract at all, only gateway configuration. The correct sequencing is: select the mechanism, then determine whether it requires a shared schema, claims profile, certificate/workload-identity profile, producer-registration contract, gateway configuration contract, or no new cross-repository payload contract — and publish a machine-readable contract only after a stable boundary is demonstrated, per this ecosystem's existing readiness discipline (ADR-0005). No change to `adapter-evidence-reference` or `identity-evidence-reference` is anticipated regardless — both are structurally sufficient per this assessment's inspection (§5.2), independent of which authentication mechanism is chosen.

**Public-contract tests likely to follow.** A canonical compatibility scenario exercising a real (or faithfully simulated) trusted-producer submission end to end — the gap this assessment identified in §11's "Test and conformance gaps" row.

**Smallest safe first vertical slice after architecture acceptance.** A bounded reference implementation, scoped to exactly one protocol adapter and one deployment topology (the architecture document's own §13 Stage 5 language), that: invokes one existing adapter's normalization, calls the now-implemented `construct_adapter_evidence()`, mints a `reference_id` and assembles a real `AdapterEvidenceReference`, authenticates to `basis-gateway` using whatever mechanism the ADR selects, and submits one real operation-aware request end to end. This slice deliberately excludes execution — it stops at receiving the gateway's disposition, consistent with the architecture document's own invariant that execution must never occur before an authoritative disposition is received, and consistent with keeping this first slice narrow enough to actually validate the roles and rules rather than attempting the full lifecycle at once.

---

## 14. Recommended Sequencing

1. **Architecture ADR** — producer-to-gateway workload-authentication and admission model selection (§13 Stage 4 equivalent), scoped per §13 above. This is the correct next step per both the architecture document's own sequencing and this assessment's confirmation that every downstream question depends on it.
2. **Schema determination, then publication or correction only if needed** — determine whether the selected mechanism requires any new cross-repository payload contract at all (§13 above); publish only if a stable boundary is demonstrated. No correction to existing evidence-reference contracts is anticipated by this assessment's findings.
3. **Public-contract alignment** — a canonical compatibility scenario exercising a real trusted-producer submission, extending `basis-schemas`' existing five-scenario compatibility-vector discipline.
4. **Gateway trusted-producer follow-up** — `basis-gateway` admission-logic changes to accept the selected mechanism (§9's "Gateway producer-authentication and admission refinement," already named as Stage 4 implementation work following the Stage 4 decision).
5. **Producer-runtime implementation, if approved** — the bounded reference implementation described in §13 above, with *permanent* repository placement decided as part of or immediately after this stage, per the architecture document's own Stage 5 decision gate; the ADR in step 1 need only fix the slice's *provisional* location.
6. **Execution-plane architecture and implementation** — explicitly sequenced after the producer runtime exists to inform it (Stage 6+ in the architecture document), not before.
7. **Console integration** — execution-state display, once an execution-result contract exists to display.
8. **Deployment and demonstration** — `basis-deploy` packaging and an end-to-end demonstration extending `basis-gateway`'s existing `demo/operation-aware/` bounded demonstration to include a real or faithfully simulated protocol dispatch, per the architecture document's own §13 Stages 8–9.

This sequence matches the architecture document's own §13 ordering; this assessment did not find implementation evidence justifying a reordering. Producer authentication (step 1) is the first and most immediate architectural decision and the clearest current blocker, since every other piece of the handoff this assessment inspected — evidence construction, gateway admission, provenance classification, audit emission — is already implemented and waiting on it specifically. It is not, however, the *only* gap a working producer needs closed: step 5's reference implementation still has to resolve reference-minting mechanics, redaction-policy application, evidence persistence, correlation behavior, retry/idempotency behavior, and its own provisional placement, none of which the authentication ADR alone settles.

---

## 15. Appendix: Evidence Index

| Repository | File path | Symbol / endpoint / model / schema / test | Capability supported | Evidence classification |
| - | - | - | - | - |
| `basis-architecture` | `docs/architecture/operation-producer-and-execution-boundary.md` | Full document | Producer/executor role definitions, trust rules, correlation model, decision sequence | Documented only |
| `basis-architecture` | `docs/adr/0007-adapter-evidence-construction.md` | ADR-0007 (Accepted) | Evidence-construction ownership split | Documented only (governs; implementation confirmed separately below) |
| `basis-architecture` | `docs/adr/0006-evaluation-orchestration-layer.md` | ADR-0006 (Status: Proposed) | `basis-core` evaluation-layer composition boundary | Documented only (status inconsistency noted, §5.1) |
| `basis-architecture` | `docs/architecture/ecosystem-contract-inventory.md` | Full document, §3–§4 | Existing contract inventory; explicit non-coverage of producer/execution contracts | Documented only |
| `basis-architecture` | `ROADMAP.md` | "Next Producer and Execution-Evidence Boundary" section | Ecosystem-level status statement | Documented only |
| `basis-schemas` | `schemas/adapter-evidence-reference/adapter-evidence-reference.yaml` | `adapter_evidence_reference` contract; its `composition` comment ("mints reference values" for `basis-adapters`) | Evidence-reference shape; contract-metadata ownership statement | Contract published (shape); Ambiguous (metadata — conflicts with ADR-0007, §5.2) |
| `basis-schemas` | `schemas/identity-evidence-reference/identity-evidence-reference.yaml` | `identity_evidence_reference` contract | Identity-evidence-reference shape | Contract published |
| `basis-schemas` | `schemas/operation-aware-decision-request/operation-aware-decision-request.yaml` | `operation_aware_decision_request` contract | Submission envelope shape | Contract published |
| `basis-schemas` | `examples/operation-aware/compatibility/*` | Five canonical scenarios | Kernel evaluation conformance | Implemented (kernel-side); Absent (producer-submission scenario) |
| `basis-core` | `src/basis_core/domain/evidence.py` | `IdentityEvidenceReference`, `AdapterEvidenceReference` | Evidence-reference domain models | Implemented |
| `basis-core` | `src/basis_core/evaluation/operation_aware/{engine,trace_assembly,response_assembly}.py` | Evaluation orchestration modules | Kernel composition (ADR-0006's subject) | Implemented |
| `basis-gateway` | `src/basis_gateway/auth/operation_producer.py` | `classify_operation_producer()`, `OperationProducerTrust` | Trusted-producer classification | Implemented, 23 test functions found (not executed by this assessment) |
| `basis-gateway` | `tests/test_operation_producer_trust.py` | 23 `def test_...` functions | Producer-trust test coverage | Implemented (found by inspection, not executed) |
| `basis-gateway` | `src/basis_gateway/core/operation_aware_composition.py` | `ProvenanceClassification`, `UntrustedOperationProducerContextError`, `compose_operation_aware_request` (name approximate) | Request composition, provenance classification, producer-context rejection | Implemented |
| `basis-gateway` | `src/basis_gateway/core/operation_aware_evaluator.py` | `_default_trace_id()`/`_default_evidence_id()` (default `trace_id_factory`/`evidence_id_factory`), `OperationAwareGatewayEvaluator.evaluate()` | Identifier generation (via injectable factory, default `uuid.uuid4()`), kernel invocation | Implemented |
| `basis-core` | `src/basis_core/enforcement/operation_aware.py` | `OperationAwareEnforcementPoint.evaluate(trace_id=..., evidence_id=...)` docstring | Confirms `trace_id`/`evidence_id` are caller-supplied, never generated inside the kernel | Implemented (documents the boundary precisely) |
| `basis-gateway` | `src/basis_gateway/audit/operation_aware_gateway_events.py` | `GatewayAuditEvent` | Gateway-side audit | Implemented |
| `basis-gateway` | `src/basis_gateway/middleware/correlation.py` | `CorrelationMiddleware` | Correlation-ID generation | Implemented |
| `basis-gateway` | `src/basis_gateway/api/routes.py` | `POST /v1/evaluate/operation-aware`, `OPERATION_AWARE_ENABLED` | Operation-aware endpoint | Implemented, feature-flagged (disabled by default) |
| `basis-gateway` | `demo/operation-aware/` | `run_demo.py`, `policy-bundles/`, `expected/` | Bounded gateway-side demonstration | Implemented (gateway-scope only; no adapter/producer involvement) |
| `basis-adapters` | `src/basis_adapters/evidence.py` | `construct_adapter_evidence()`, `AdapterEvidenceMaterial`, `ConstructedAdapterEvidence`, `EvidenceDigest` | Evidence-material construction, canonicalization, digest | Implemented, 67 test functions found (not executed by this assessment) |
| `basis-adapters` | `tests/test_evidence.py` | 67 `def test_...` functions across 1,355 lines | Evidence-construction test coverage | Implemented (found by inspection, not executed) |
| `basis-adapters` | `docs/operation-aware-handoff-alignment-plan.md` | Planning document (merged) | Assessment underlying ADR-0007 | Documented only |
| `basis-console` | `src/basis_console/operation_aware_presentation.py` | `ContentSource`, `build_operation_aware_presentation()` | Producer/subject/evidence presentation | Implemented |
| `basis-console` | `src/basis_console/operation_aware_training.py` | `producer_trust_classification` training key | Training-mode producer-trust explanation | Implemented |
| `basis-console` | `src/basis_console/simulator.py`, `src/basis_console/ui/views.py` | `build_operation_aware_simulation()`, simulator route docstring | Simulation-only submission path | Implemented, explicitly self-classified as preview |
| `basis-identity` | `src/basis_identity/models/identity_context.py` | `SubjectType.SERVICE`, `AuthenticationProtocol` enum | Existing (insufficient) service-identity vocabulary | Partially implemented |
| `basis-identity` | (repository-wide targeted search) | "identity_evidence_reference", "IdentityEvidenceReference", "workload", "spiffe", "mtls" | Workload identity; `identity-evidence-reference` production | Absent (both) |
| `basis-deploy` | N/A | N/A | N/A | Not inspected (repository does not exist) |
| Demo/PoC/lab | N/A | N/A | N/A | Not inspected (repository not available in working environment) |

---

## Non-Goals (restated)

Consistent with the assignment governing this assessment: this document does not create an ADR, does not assign final repository ownership, does not invent or select a producer-repository name, does not create a new repository, does not modify any implementation repository, does not modify any schema, does not add Python models or endpoints, does not implement producer authentication or adapter invocation or gateway submission or execution, does not redesign the console, and does not add deployment manifests or change release documentation. No successful Git write operation occurred in the production of this document; the one attempted write (`git add -N`, during validation) failed against a pre-existing `.git/index.lock` and changed nothing — see §2.
