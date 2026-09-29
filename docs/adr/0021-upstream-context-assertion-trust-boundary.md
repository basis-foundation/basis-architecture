# ADR-0021: Upstream Context Assertion Trust Boundary

## Status

Proposed

Consistent with this repository's established practice (see ADR-0018's Context and ADR-0020's Status), this ADR is numbered when first proposed and submitted as `Proposed`. Merging it does not accept it. Acceptance requires a separate, dedicated formal-acceptance review and PR. Publication of this proposal authorizes no implementation, does not close cross-ecosystem gap ARCH-GAP-004, and makes no upstream platform's contract canonical.

## Context

[ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md) (Accepted) defines the producer-intake boundary through which an upstream supervisory system submits supervisory intent to the operation-producer role. Its Decision 4 decides only that request-scoped context *keeps its upstream origin* across that boundary. It defers the question of which context categories an upstream system, or an operation producer relaying upstream-originated context, may authoritatively assert (ADR-0018 **Deferred Decisions**, "Context-assertion trust (cross-ecosystem Workstream 3D)"). Until that question is decided, ADR-0018 Decision 4 imposes a fail-closed interim posture: the operation-producer role must not present upstream-originated context to `basis-gateway` as producer-only context. [ADR-0020](0020-operation-to-authorization-mapping-and-composition-boundary.md) (Accepted) keeps that posture and adds that upstream source references are mapping provenance, not authorization context (ADR-0020 Decision 10). This ADR answers the deferred question on the BASIS side.

The question spans things the repository currently holds in different states. This ADR records them as found on `main` before deciding anything.

**The producer-only context surface.** The operation-aware gateway request carries nine fields that `basis-gateway` accepts only from a caller it has classified as an admitted operation producer: `operation_intent`, `location`, `device`, `protocol_context`, `safety_context`, `environment_context`, `risk_context`, `identity_evidence_reference`, and `adapter_evidence_reference` ([`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §1).

**Producer trust is all-or-nothing across that surface.** [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) §3 separates *producer workload identity* from *authorization to act as an operation producer*, and both from *authorization to assert specific context categories*. It records the last as "not established today": admission "grants an all-or-nothing capability across all nine producer-only fields." [ADR-0008](0008-producer-workload-authentication-and-admission.md) (Accepted) keeps that model unchanged and names its consequence under **Compromised producer runtime**: "a compromised, admitted producer can assert any producer-only context it is admitted for," with category-scoped producer trust deferred.

**Provenance is one flat dimension.** `basis-gateway` classifies every present producer-only field `trusted_producer_asserted`, never `verified`, from the closed `ProvenanceClassification` vocabulary (`verified`, `gateway_derived`, `trusted_producer_asserted`, `untrusted_caller_asserted`, `configuration_derived`, `unavailable`; `operation-producer-and-execution-boundary.md` §7). That label correctly says an admitted producer asserted the value. It cannot say that the producer merely relayed a value some other system originated, or which system that was. Applied to relayed context, it erases the origin. That is why ADR-0018 Decision 4 withholds trust in the interim.

**Accepted architecture already fixes some origins and not others.** `operation-producer-and-execution-boundary.md` §4's field-ownership table assigns `protocol_context` to the producer, "derived from adapter evidence"; `adapter_evidence_reference` to the producer, constructed from adapter output (ADR-0007); `identity_evidence_reference` to the producer, "if `basis-identity` produced one for the initiating subject"; `location` and `device` to the producer, "if known from deployment configuration or protocol evidence"; and `safety_context`, `environment_context`, and `risk_context` to the producer, "if a governed upstream source exists." None of those rows says who may originate a value upstream of the producer, or how an upstream origin is represented.

**Absence has defined, sometimes permissive, policy semantics.** [`operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md) §10 defines missing-context behavior, and [`condition-operator-semantics.md`](../architecture/condition-operator-semantics.md) §7–§8 defines absence per operator. An absent value makes most conditions `no_match` and makes `not_exists` `match`. A `DENY` rule conditioned on a present value therefore does not fire when that value is absent, and an `ALLOW` rule may be conditioned on absence. Absence is a meaningful policy input. It is not a neutral "no information" state that a rejected value can safely fall back to.

**Current implementation (read-only evidence; architecture governs).** At the revisions inspected — `basis-gateway` `1e08348`, `basis-producer` `8c7410a`, `basis-core` `7a92497`:

- `basis-gateway` gates all nine fields on one trusted/untrusted producer classification (`OPERATION_PRODUCER_ONLY_FIELDS`; `UntrustedOperationProducerContextError`) and classifies each present field `trusted_producer_asserted`. It has no per-category capability, no representation of an upstream source identity, and no freshness check. The field-provenance map is recorded in gateway audit events; it is not a kernel input.
- `basis-producer` has no intake realization and no representation for upstream-originated context. Its bounded REST composition leaves every context field except `adapter_evidence_reference` absent because it has "no governed source for any of them."
- `basis-core`'s context models (`OperationAwareSafetyContext`, `OperationAwareRiskContext`, and the rest) carry values only: no source identity, no observation time, and no provenance.

**Motivating consumer.** Ipotio, a supervisory platform, tracks the cross-ecosystem form of this question in its own architecture repository as **ARCH-GAP-004** (authorization-context freshness, staleness, and category-scoped producer-assertion trust). It is referenced here by name, not by hyperlink, per this repository's cross-repository citation convention, and only as the motivating integration. Its current architecture proposes that some of its operational facts, such as asset-derived location or device and supervisory safety or risk state, reach BASIS as authorization context. That proposal is evidence of one consumer's needs. It is not assumed correct here. The model below is stated for any upstream supervisory system.

**Terminology.** This ADR uses *BASIS*, not the proposed *Basitra* ([ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md), Proposed). *Upstream supervisory system*, *producer-intake boundary*, and *upstream workload identity* are ADR-0018's working vocabulary. *Context category*, *context assertion*, *origin class*, *originating source*, *category trust policy*, *category assertion authority*, and *trust laundering* are introduced below as working vocabulary under the same convention and are not promoted to the glossary.

## Problem

How does BASIS decide whether an operational context value that originated upstream of the operation-producer boundary may be used as authorization context, so that:

- admission of an upstream workload, or of an operation producer, never grants authority over every context category;
- a producer cannot turn an upstream claim into a producer fact by forwarding it;
- the original source of a fact survives every relay and remains attributable;
- a value too old, or with too little provenance, for its category is never evaluated as current;
- a rejected value never silently turns into an absent one;
- subject identity, operation identity, and credential lifecycle remain governed by their own decisions;
- the model works for any upstream system?

## Decision

**Authority to assert operational authorization context is category-scoped, origin-aware, and conjunctive. For every context category, a BASIS-governed category trust policy states which origin classes the category may have, which originating sources may supply it, which operation producers may assert or relay it, and what provenance and freshness it requires. A context value reaches policy evaluation only when every one of those conditions holds for that value. Relaying a value adds an accountable relay hop. It never changes who originated the fact, never upgrades its provenance, and never refreshes its age. A context assertion that is present but cannot be admitted fails closed. It is never dropped and treated as absent. Absence stays absence, and policy alone decides what absence means.**

The decision rests on one principle:

```text
The originating source decides whether a candidate fact is
truthful and current enough to offer.

BASIS decides whether that source, and each producer that relays
the fact, is trusted to supply or assert that category.

Neither decision substitutes for the other.
```

Stated as an admissibility condition, for a context value `V` in category `C`:

```text
common(V, C) =                                         -- every origin class
      origin class O of V permitted for C              (Decision 4)
  AND asserting/relaying producer authorized for (C, O) (Decisions 1, 3)
  AND required provenance present and preserved        (Decisions 2, 5, 6)
  AND category freshness/validity satisfied, where required (Decision 7)
  AND no unresolved conflict for C                     (Decision 9)

upstream_only(V, C) =                                  -- O = upstream-originated only
      originating upstream source authorized for C     (Decisions 1, 3)
  AND source-declared eligibility is current           (Decision 7)

admissible(V, C) =
      common(V, C)
  AND (O != upstream-originated  OR  upstream_only(V, C))
```

For an upstream-originated value, the producer term in `common` is the producer's *relay* grant, so the two-hop rule of Decision 3 (upstream source grant AND producer relay grant) holds in full. Producer-derived, adapter/protocol-derived, and identity/evidence-derived values need no upstream source grant and no source eligibility status. They are governed by category and origin-class authority, provenance, freshness where their category requires it, and conflict handling. This ADR defines no eligibility metadata for them.

Every applicable conjunct is required. No trusted component can supply a missing conjunct on another component's behalf. Admission to the producer-intake boundary (ADR-0018) and producer admission at `basis-gateway` (ADR-0008, ADR-0009) remain preconditions; they are necessary and never sufficient.

### 1. Assertion authority is category-scoped, not producer-global

The question BASIS answers is *may principal X assert or supply category C, with origin class O?*, not *is X trusted?*

- **Producer admission is not category authority.** An operation producer admitted at `basis-gateway` has no authority over any context category until the category trust policy explicitly grants it for that category. Admission remains necessary.
- **Upstream admission is not category authority.** An upstream workload admitted to the producer-intake boundary (ADR-0018 Decision 2) may be allowed to request operations while being authorized to supply no context category, or only some. For example, it may supply `location` but not `safety_context` or `risk_context`. This is expressed against the one upstream workload identity. It never requires creating additional or synthetic producer identities.
- **Default deny.** No explicit category grant means no assertion authority. Authority is never inferred from admission, role membership, attribute values, network position, identity-string prefixes or wildcards, the request body, producer self-declaration, caller-supplied roles, or any upstream assertion.
- **Governed configuration.** Category grants are explicit, deployment-owned, BASIS-governed configuration. They are attributable to an authenticated administrative authority, in the same sense as producer admission (ADR-0008 "Registration and revocation") and mapping configuration (ADR-0020 Decision 8). No intake request, producer submission, or runtime interaction can create, widen, or activate a grant.
- **Scope granularity.** Where a category permits more than one origin class, a producer's authority is scoped by origin class as well as category. Authority to originate category C is not authority to relay upstream-originated C, and the reverse also holds.

This ADR does not select a configuration format, storage, registry, policy language, administrative API, or environment-variable syntax.

### 2. Origin is a durable fact, and it survives every relay

Every context value has exactly one **origin class** and, where the class requires one, one **originating source**. The origin classes the architecture recognizes for producer-supplied context are:

| Origin class | Meaning | Originating source |
| - | - | - |
| **Producer-derived** | The operation-producer role derived the value itself from BASIS-governed inputs it holds: its own deployment configuration, or its own governed observation | The producer workload |
| **Adapter/protocol-derived** | Derived by the operation-producer role from `basis-adapters` normalization output or evidence for the preserved operation | The producer workload, from the preserved operation (ADR-0007, ADR-0020 Decision 4) |
| **Identity/evidence-derived** | Produced by an identity or evidence authority (today, prospectively, `basis-identity`) and referenced by the producer | That authority |
| **Upstream-originated** | Originated by a system outside BASIS and relayed into BASIS | The authenticated upstream workload that supplied it |

`basis-gateway`'s own classes, such as gateway-derived values and verified authorization-subject identity, are not origins a producer may assert.

**Anything a producer did not derive from BASIS-governed inputs is not producer-derived.** A value the producer obtained from a system outside BASIS is upstream-originated, whatever channel it arrived through. Today the only governed channel for upstream-originated context is the ADR-0018 producer-intake boundary, and the originating source is the authenticated upstream workload identity. A producer that obtains an external value through any other channel may not present it as producer-derived. Such a value is not admissible until a future decision governs that channel under the same conditions.

**Relay never changes origin.** A producer that relays an upstream-originated value is an accountable relay hop, not the value's origin. The upstream workload remains the originating source through every relay, handoff, and composition step, up to the point where the gateway admits or rejects the value.

### 3. Upstream supply and producer relay are independent authorities

For an upstream-originated value, two separate grants are required, and each is checked on its own:

```text
allow_upstream_context(U, P, C) =
      upstream_category_authorized(U, C)     -- U may supply C
  AND producer_category_authorized(P, C, upstream-originated)
                                              -- P may relay upstream-originated C
  AND C permits upstream-originated origin
```

- **An authorized producer cannot amplify an unauthorized source.** If `U` is not authorized for `C`, the value is rejected however broadly `P` is trusted.
- **An authorized source cannot borrow a producer's breadth.** If `U` is authorized for `location` only, a `risk_context` value from `U` is rejected even when `P` may relay `risk_context` from other sources.
- **A producer's own authority does not cover relayed values.** If `P` may assert producer-derived `C` but not relay upstream-originated `C`, an upstream value for `C` is rejected. The producer cannot rescue it by re-labeling it as its own (Decision 10).

### 4. The category trust policy constrains permitted origin

For each category, the category trust policy states which origin classes are permitted. It can express any subset of the classes in Decision 2. A value whose origin class is not permitted for its category is inadmissible, whoever asserts it.

Where accepted architecture already establishes a category's origin, the category trust policy may not contradict it. See **Current Producer-Only Context Inventory** below. Where accepted architecture does not decide a category's origin, this ADR does not decide it either. The model can express any assignment, and each deployment's category trust policy states its own. For those categories this ADR adds only default deny: an origin class the policy does not permit is not permitted.

### 5. Origin and assertion trust are separate dimensions; trusted is not verified

The evidence for an admitted context value must keep the following facts separately reconstructible. They must never be collapsed into a single label that loses any of them:

- **origin**: the origin class and the originating source (Decision 2);
- **relay path**: each producer that asserted or relayed the value;
- **authority basis**: the category trust policy decision that admitted it, and its identity or version;
- **assertion status**: whether the value is a trusted *assertion* or an independently *verified* fact;
- **freshness state**: the validity condition that applied and its result (Decision 7).

**A trusted assertion is not an independently verified fact.** A category trust policy grant means the policy permits that source, through that producer, to supply that category. It does not mean BASIS measured, observed, or confirmed the fact. An admitted upstream `safety_context` is a trusted upstream assertion. BASIS did not observe the safety condition. `verified` provenance is never used to mean "trusted enough for policy evaluation." The existing rule that producer-only fields are never promoted to `verified` (`operation-producer-and-execution-boundary.md` §3) extends unchanged to upstream-originated values.

This ADR does not choose enum values, field names, or a schema for these dimensions (see **Relationship to the Current Provenance Vocabulary**).

### 6. Required provenance for upstream-originated context

For every upstream-originated context value admitted for authorization use, BASIS preserves enough information to attribute at least:

- the context category;
- the originating upstream workload identity;
- the identity of each producer that relayed or asserted it;
- the origin class;
- a source provenance reference that the originating source supplied and that stays attributable to it;
- the source's observation or as-of time, where the category's freshness requirement depends on one;
- the source-declared eligibility status, where the source declares one (Decision 7);
- the freshness or validity result the category trust policy required.

None of these may be discarded, overwritten, or collapsed at any hop. A value for which any required element is missing or unattributable is inadmissible (Decision 8). This ADR defines no wire representation, envelope, schema, storage, signing format, or correlation API for these facts.

### 7. Freshness and validity are part of category admissibility

**A context value must not be used for authorization beyond the freshness or validity conditions its category requires.**

Two separate responsibilities apply to an upstream-originated value:

- **The originating source is responsible for truthful source facts.** It supplies the value, its observation or as-of time, its provenance reference, and its own eligibility determination. It offers a value only if its own rules consider the value current. A value that the source itself marks as not current — stale, unknown, conflicted, never observed, or equivalent — is not admissible as authorization context. The source's eligibility rules are its own; BASIS does not adopt them.
- **BASIS is responsible for its acceptance condition.** The category trust policy states what validity metadata the category requires and the condition that metadata must satisfy, for example a maximum age relative to evaluation, or a source-declared validity bound. BASIS enforces that condition at authorization evaluation, against the gateway's evaluation time.

```text
source says:   value V, category C, observed/as-of T, provenance R, status S
BASIS asks:    is the source authorized for C?  is the producer authorized for C?
               is the provenance sufficient?     is S current?
               does T satisfy C's freshness condition at evaluation time?
```

The following rules also apply:

- **Unestablishable freshness fails closed.** If a category requires freshness and it cannot be established, for example because the source time is missing, unparseable, or in the future beyond tolerance, the value is inadmissible.
- **Relay time is not observation time.** No producer, intake realization, or gateway may substitute its receipt, relay, submission, or composition time for the source's observation or as-of time.
- **No component manufactures a timestamp.** No component may set, advance, or "refresh" a source time without a new observation by the originating source.
- **Intake validity is not context freshness.** An intake request that satisfies ADR-0018 Decision 7's bounded validity may still carry a stale context value, and a fresh value does not make an ineligible request eligible. Each condition is checked on its own terms.
- **Producer-derived values have validity requirements too.** The category trust policy may require them, and a producer-derived value that cannot satisfy its category's requirement is inadmissible on the same terms.

This ADR selects no durations, clock source, clock-synchronization mechanism, skew tolerance, or time format. Whether a context value's aging between authorization and dispatch makes a pending bound operation ineligible belongs to the open broader replay/freshness family ADR-0012, ADR-0013, and ADR-0016 already hold. It is not decided here.

### 8. Present-but-inadmissible context fails closed; missing stays missing

**Present but rejected is not the same as never supplied.** When a context value is explicitly supplied and cannot be admitted, the attempt fails closed before that value, or any substitute for it, reaches policy evaluation. The value is never dropped and treated as absent. The reason is structural. Absence has defined, sometimes permissive, policy semantics (see Context). Silently converting a rejected `safety_context` into an absent one could let a policy whose `DENY` rule depends on it, or whose `ALLOW` rule depends on its absence, authorize an operation that should have been refused.

Fail closed means:

- **at intake:** the intake request is rejected (ADR-0018 Decision 6);
- **at the operation-producer role:** production fails closed (ADR-0020 **Failure Behavior**);
- **at `basis-gateway`:** the request is rejected before kernel evaluation, as it is today for an unadmitted producer asserting producer-only context (ADR-0008).

Each outcome belongs to the "not executed" family (`operation-producer-and-execution-boundary.md` §8) and is recorded with the specific cause.

**Missing stays missing.** When no source supplied a category and no BASIS-owned component legitimately derives it, the category is absent. The producer does not fill it. The intake boundary does not fill it. The gateway does not fill it. The kernel and the policy then decide what absence means for that operation, under existing missing-context semantics. This ADR does not define "missing `safety_context` means deny," "missing `risk_context` means deny," or any other universal absence semantics. The trust boundary guarantees honest presence and absence and fail-closed admission. Policy decides whether an operation requires a category.

These states are distinct, and none may be converted into another:

```text
absent                                 -- never supplied; a policy input by absence
present, admitted                      -- trusted assertion, usable by policy
present but unauthorized               -- source or producer lacks category authority  → fail closed
present but invalid origin             -- origin class not permitted for category      → fail closed
present but provenance-deficient       -- required attribution missing                 → fail closed
present but stale                      -- freshness condition not satisfied            → fail closed
present but unknown / never observed   -- source-declared non-current                  → fail closed
present but conflicted                 -- unresolved competing candidates              → fail closed
present but malformed                  -- structurally invalid                         → fail closed
```

A future governed representation of an explicitly unknown or non-current value as a policy-visible state is a schema decision (`operation-producer-and-execution-boundary.md` §4). Until such a representation exists, such values are not admissible.

### 9. Conflicting context fails closed

When more than one candidate value exists for the same category in one governed operation — from several upstream sources, from an upstream source and the producer, or as an internally contradictory value — and no accepted, category-specific reconciliation rule governs the case, the attempt fails closed. No component may pick the newest value, prefer the producer's value, prefer one source, merge values heuristically, or delegate resolution to an AI or any other ungoverned mechanism. A future category-specific aggregation or reconciliation contract may define otherwise. This ADR defines none.

### 10. A relay never rewrites upstream semantics

For an upstream-originated value, the intake boundary and the operation-producer role may validate structure and carry out their governed relay duties. They must not:

- change the value;
- change the observation or as-of time;
- remove, replace, or re-attribute the originating source;
- change the origin class, for example re-labeling the value producer-derived;
- upgrade provenance or assertion status;
- change a stale, unknown, conflicted, or never-observed status to current;
- convert a candidate, inferred, or derived fact into an authoritative one;
- synthesize missing supporting evidence or provenance.

If a category ever requires a transformation, such as unit conversion or aggregation, the transformation must have an explicitly governed owner and its own provenance contract, recorded by a later decision. This ADR assigns no transformation authority to the producer or to any other component.

### 11. Enforcement placement

| Component | Responsibility under this decision | Does not |
| - | - | - |
| **Originating upstream system** | Offers only values its own eligibility rules consider current; supplies truthful source time, provenance reference, and eligibility status | Grant itself category authority; become a producer (ADR-0018) |
| **Producer-intake boundary** (ingress of the operation-producer role) | Authenticates and admits the upstream workload (ADR-0018); enforces its category-supply scope and the category's permitted origins; requires and preserves origin and provenance; rejects inadmissible values early | Decide authorization; re-label origin; fill absent categories |
| **Operation-producer role** | Preserves origin, source, and provenance unchanged (Decision 10); submits only what its own category authority covers, and fails closed locally otherwise; asserts producer-derived and adapter/protocol-derived values only from BASIS-governed inputs | Launder, upgrade, refresh, or synthesize; choose among conflicting values |
| **`basis-gateway`** | Final enforcement before evaluation: producer admission (unchanged), then category trust evaluation of every present value: permitted origin, originating-source authority, producer authority, provenance sufficiency, freshness at evaluation time, conflict. Rejects inadmissible values pre-kernel and records the admission basis | Authenticate upstream workloads; fill absent categories; accept values on the strength of an earlier boundary's check |
| **`basis-core`** | Evaluates the governed context it receives under policy, including existing missing-context semantics | Authenticate any workload; validate provenance; evaluate category trust; know which upstream system a value came from |

**The gateway's check covers both scopes.** The intake boundary's check is an earlier rejection point. It is not a substitute for the gateway's check. The gateway evaluates both the originating source's grant and the producer's grant against the same governed category trust policy. A misconfigured or compromised intake realization therefore cannot widen what reaches the kernel beyond what the policy grants to the attributed source.

**Residual trust in attribution.** Until a future decision selects an end-to-end origin-integrity mechanism, the gateway learns the originating source's identity through the producer's attestation. ADR-0018 defers the intake integrity mechanism, and this ADR does not select one. A compromised producer that misattributes origin therefore defeats this decision in the same way it defeats the ADR-0012 binding (ADR-0020 **Consequences**, residual risks). Category scoping still limits what a compromised producer can assert, to the categories and origins the policy grants it.

**`basis-core` stays neutral.** `basis-core` stays an authorization evaluator. It does not become a workload-authentication, provenance-validation, or source-trust service, and it does not need to know which upstream platform a value came from. Whether provenance facets should ever be exposed to policy as evaluable inputs is not decided here.

### 12. Trust decisions are attributable

For every admitted or rejected context value, evidence must eventually be able to answer:

- which category was asserted;
- who originated it;
- which producer relayed or asserted it;
- which category trust policy decision permitted or rejected it, and why;
- what provenance supported it;
- what freshness state applied.

This ADR establishes only the requirement. The record shape, the evidence storage, and the operator-facing representation belong to Workstream 4's evidence-correlation work and to later schema decisions. None is defined here.

### 13. What context trust does not touch

- **Subject identity.** Context never establishes, substitutes for, or modifies the authorization subject. `location`, `device`, or any other category never decides who the subject is. `identity_evidence_reference` stays under the subject-identity boundary (ADR-0008 "Producer vs. authorization subject"; ADR-0018 Decision 3). A category grant for it is necessary but never sufficient, and it never creates a second, ungoverned authentication path. Unverified subject hints stay hints.
- **Operation identity and mapping (ADR-0020).** Context never influences or rewrites target resolution, the preserved protocol-shaped operation, normalization, `resource_type`, the local resource identifier, the canonical action, or the canonical resource. Context affects policy evaluation only where policy consumes it. It never changes what operation is being authorized.
- **Mapping provenance is not context.** Upstream source references, including upstream durable asset identity, remain provenance under ADR-0020 Decision 10. They never become `device`, `location`, or any other category automatically. A value becomes context only if the originating source explicitly supplies it as a context assertion in that category and every condition of this decision holds.
- **Credentials.** This decision relies on abstract authenticated identities: an upstream workload identity `U` and a producer workload identity `P`. It selects no certificate profile, SPIFFE, OIDC client credentials, token format, secret manager, issuance, rotation, revocation transport, bootstrap, HA credential sharing, or disconnected-operation behavior. Those remain Workstream 3E (ARCH-GAP-011).

### 14. Platform neutrality

This ADR defines no type, field, category, vocabulary, or rule named after, or shaped for, any upstream platform. The same decision applies to a supervisory platform, an industrial application, a Niagara integration, a SCADA supervisory workload, a custom OT orchestrator, or any future admitted upstream system. BASIS remains fully usable with no upstream system at all. In that case every context value is producer-derived, adapter/protocol-derived, or identity/evidence-derived, under the same category trust policy.

## Admissibility Decision Matrix

Each row is a distinct decision. None implies another.

| Question | Meaning | Decided by | Established by |
| - | - | - | - |
| Upstream source admitted? | May this workload submit intake requests at all? | Producer-intake boundary | ADR-0018 Decision 2 |
| Upstream category-authorized? | Upstream-originated values only: may this source supply category C? | Category trust policy | This ADR, Decisions 1, 3 |
| Producer admitted? | May this workload act as an operation producer? | `basis-gateway` | ADR-0008, ADR-0009 |
| Producer category-authorized? | May this producer assert, or relay, category C with this origin class? | Category trust policy | This ADR, Decisions 1, 3 |
| Origin permitted? | May category C have this origin class? | Category trust policy, within accepted architecture | This ADR, Decision 4 |
| Provenance sufficient? | Does the value carry every required origin and evidence attribution? | Category trust policy | This ADR, Decisions 5, 6 |
| Source eligibility current? | Upstream-originated values only: did the source offer the value as current? | Originating source | This ADR, Decision 7 |
| Freshness admissible? | Does the value satisfy C's validity condition at evaluation? | Category trust policy, enforced by `basis-gateway` | This ADR, Decision 7 |
| Conflict-free? | Is there exactly one candidate, or a governed reconciliation rule? | Category trust policy / future contract | This ADR, Decision 9 |
| Policy relevance? | Does authorization policy consume C, and what does its absence mean? | Policy, evaluated by `basis-core` | Existing evaluation semantics |

## Current Producer-Only Context Inventory

This table records what accepted architecture already establishes for each of the nine current fields. It does not redefine any field's semantics. "Open" means accepted architecture does not decide the question. The category trust policy can express any answer, and this ADR does not choose one.

| Field | Established semantic owner / origin | Upstream relay legitimate? | Producer derivation legitimate? | Nature | Other constraint |
| - | - | - | - | - | - |
| `operation_intent` | Producer, "if it knows" (§4) | **Open.** ADR-0020 makes the upstream's own operation intent upstream-owned provenance, not interpreted; it can never become this field through mapping provenance | Yes, where the producer has a governed basis (§4) | Operational context (closed enum) | Must not influence normalization or composition (ADR-0020) |
| `location` | Producer, from deployment configuration or protocol evidence (§4) | **Open.** This is the principal Workstream 3D question; upstream asset identity is provenance only (ADR-0020) | Yes, from deployment configuration or protocol evidence (§4) | Operational context | Not derived by the gateway (§4) |
| `device` | As `location` (§4) | **Open.** As `location` | Yes, as `location` | Operational context | Distinct from the canonical resource (ADR-0020 Decision 1) |
| `protocol_context` | Producer, derived from adapter evidence (§4) | **Not permitted by this ADR.** Accepted architecture establishes adapter/protocol-derived origin from the preserved operation (ADR-0020 Decision 4); permitting upstream relay would need a later decision | Yes, adapter/protocol-derived only | Operational context derived from evidence | Distinct from the mandatory `protocol_evidence` reference |
| `safety_context` | Producer, "if a governed upstream source exists" (§4) | **Open.** §4 contemplates an upstream source without governing one; this ADR supplies the governing conditions, not the assignment | Only if the producer itself has a BASIS-governed basis; otherwise open | Operational context | §4 calls the category-trust question most acute here |
| `environment_context` | As `safety_context` (§4) | **Open**, as `safety_context` | As `safety_context` | Operational context | — |
| `risk_context` | As `safety_context` (§4) | **Open**, as `safety_context` | As `safety_context` | Operational context; deployment-defined, never a calculated or verified value (`basis-core` model) | — |
| `identity_evidence_reference` | `basis-identity`, referenced by the producer (§4; ADR-0008) | **Not governed by this ADR.** Conveyance of subject-related material from upstream is subject-credential conveyance (ADR-0018 Decision 3; deferred) | No. The producer references it; it does not originate it | Evidence/reference material, not operational context | Subject-identity boundary governs (Decision 13); `basis-identity` does not yet produce it |
| `adapter_evidence_reference` | Producer assembles from `basis-adapters` material and digest (ADR-0007) | **No.** Accepted architecture fixes its construction at the producer from the preserved operation | Yes, adapter/protocol-derived, per ADR-0007 | Evidence/reference material | ADR-0007 construction ownership unchanged |

Under default deny (Decision 1), even a field whose origin accepted architecture fixes is assertable only by a producer the category trust policy explicitly authorizes for it.

## Threat Cases

**Risk-context laundering.** Upstream workload `U` is admitted to intake but has no `risk_context` grant. Admitted producer `P` relays a `risk_context` value from `U`. The value is rejected: `upstream_category_authorized(U, risk_context)` is false, and nothing about `P` can make it true (Decision 3).

**Producer overreach.** `P` is authorized for `protocol_context` and not for `safety_context`. A `safety_context` value from `P` is rejected even though `P` is admitted (Decision 1).

**Origin stripping.** `P` receives an upstream `device` value and submits it as producer-derived. That is prohibited (Decisions 2, 10). The value did not come from BASIS-governed inputs `P` holds, so it is not producer-derived. The gateway's final check still requires the attributed origin to satisfy both grants. The residual risk is a compromised `P` that misattributes origin (Decision 11).

**Freshness laundering.** A source observed a `safety_context` value at `T1`. `P` relays it at `T2`, after `T1` has aged past the category's freshness condition. `T2` never replaces `T1`, and no component may advance `T1`. The value is inadmissible (Decision 7).

**Status upgrade.** A source offers a value it marks as inferred or unknown. `P` re-labels it current or authoritative. That is prohibited (Decision 10). A source-declared non-current value is inadmissible in any case (Decision 7).

**Missing provenance.** A `risk_context` value is present, but its originating source cannot be attributed. It is rejected (Decisions 6, 8).

**Rejected-then-absent bypass.** A policy's `DENY` rule fires on `safety_context.mode = "interlock-engaged"`. An unauthorized source supplies that value. If the value were dropped rather than failing closed, the `DENY` rule would see absence, return `no_match`, and an `ALLOW` could win. Under Decision 8 the attempt fails closed instead.

**Absent context.** No source supplies `safety_context`, and no BASIS component derives it. It stays absent, and the applicable policy decides whether absence permits or denies (Decision 8).

**Conflicting upstream values.** Two sources each supply a `safety_context` value, and the values disagree. No governed reconciliation rule exists, so the attempt fails closed (Decision 9).

**Mapping-to-context promotion.** An upstream request carries an asset reference as ADR-0020 source provenance. No component may turn that reference into `device` or `location` (Decision 13).

## Failure Behavior

Each outcome is semantic. This ADR defines no error codes, status vocabulary, or wire representation.

| Condition | Detected at (earliest → final) | Outcome |
| - | - | - |
| Producer not admitted asserts any producer-only context | Gateway | Rejected pre-kernel (ADR-0008, unchanged) |
| Admitted producer lacks authority for the category, or for that origin class | Producer (local) → gateway | Production fails closed / rejected pre-kernel |
| Upstream workload admitted to intake but not authorized for the category | Intake → gateway | Intake rejected / rejected pre-kernel |
| Origin class not permitted for the category | Intake or producer → gateway | Rejected |
| Originating source missing or unattributable for an upstream-originated value | Intake or producer → gateway | Rejected |
| Required provenance missing | Intake or producer → gateway | Rejected |
| Required freshness metadata missing, invalid, or unsatisfied at evaluation | Intake (early) → gateway (at evaluation time) | Rejected |
| Source-declared status stale, unknown, conflicted, or never observed | Intake → gateway | Rejected |
| Conflicting candidates with no governed reconciliation rule | Intake or producer → gateway | Rejected |
| Relay altered value, time, origin, source, or status (detectable) | Any later boundary | Rejected |
| Category trust policy unavailable or indeterminate | Wherever consulted | Rejected; never treated as permissive |
| Category never supplied and not legitimately derived | — | Absent; policy decides (no rejection by this ADR) |

In every rejected case the value is never forwarded as absent, and the operation is *not executed*.

## Security Invariants

When accepted and implemented, this decision makes the following true on the governed admitted-producer path:

```text
admitted upstream                     != trusted for all context
trusted (admitted) producer           != trusted for all categories
producer relay                        != producer origin
trusted assertion                     != independently verified fact
relay time                            != observation time
absent != stale != unknown != conflicted != unauthorized != malformed != provenance-deficient
present but inadmissible              -- never silently becomes absent
upstream context provenance           -- survives every relay
context trust                         -- does not alter operation identity (ADR-0020)
context trust                         -- does not substitute for subject identity
no explicit category authority        -> no assertion authority
```

## Relationship to the Current Provenance Vocabulary

`basis-gateway`'s closed `ProvenanceClassification` vocabulary answers one question per field: what trust basis the gateway accepted the value on. `trusted_producer_asserted` is accurate for a value an admitted producer derived itself. It cannot represent a value that was *upstream-originated, producer-relayed, and category-authorized* without losing the originating source. That loss is exactly what Decision 5 prohibits. `operation-producer-and-execution-boundary.md` §7 records that a fact the six values cannot represent "is itself a finding for the architecture process." This ADR is that finding for upstream-originated context.

This ADR therefore requires that the provenance recorded for context evolve so that origin, relay path, authority basis, assertion status, and freshness state stay separately reconstructible. It does not decide whether that means new values, a second dimension beside the existing classification, or a structured provenance record. A multi-dimensional representation is expected to fit better than another flattened label, but that choice belongs to a later schema and gateway decision. Nothing here renames, removes, or reinterprets an existing value, and no admitted upstream-originated value may ever be classified `verified` (Decision 5).

## Relationship to ADR-0018

This ADR answers the "Context-assertion trust (cross-ecosystem Workstream 3D)" item ADR-0018 deferred. It does not modify ADR-0018's body.

- **Decision 4's origin preservation** is the premise here, restated as Decisions 2 and 10.
- **Decision 4's interim posture** withholds trust. This ADR defines when trust may be granted. The interim posture remains in force for any realization that cannot yet establish every condition of the admissibility conjunction: preserved origin, both category grants, provenance, and freshness. Acceptance of this ADR therefore relaxes nothing in practice until an implementation can establish those conditions. Even then, only the categories and sources the governed policy grants are relaxed.
- **Intake admission (Decision 2) and bounded validity (Decision 7)** are unchanged and remain separate from category authority and context freshness (Decisions 1, 7).
- **Subject context crossing intake (Decision 3)** remains governed by ADR-0018 and the subject-identity chain, not by this ADR.

## Relationship to Other Decisions

- **ADR-0008 / ADR-0009.** Producer authentication and admission are unchanged and remain preconditions. This ADR supplies the category-scoped capability ADR-0008 deferred, and it bounds the blast radius ADR-0008 records for a compromised admitted producer to that producer's granted categories and origins.
- **ADR-0007.** Adapter evidence construction ownership is unchanged. `adapter_evidence_reference` stays adapter/protocol-derived, assembled by the producer.
- **ADR-0010 / ADR-0011.** The role placements are unchanged. The operation-producer role gains no transformation authority (Decision 10).
- **ADR-0012.** The binding content is unchanged. Whether context aging invalidates a pending bound operation stays in the open replay/freshness family.
- **ADR-0020.** This ADR does not reopen ADR-0020. Mapping, normalization, composition, the governed mapping unit, and mapping provenance are unchanged. Mapping provenance never becomes context (Decision 13).
- **Existing evaluation semantics.** Missing-context behavior and the absent-value rules for condition operators are unchanged and are relied on (Decision 8).

## Relationship to ARCH-GAP-004

ARCH-GAP-004 is a joint cross-ecosystem gap tracked in Ipotio's architecture repository. It covers authorization-context freshness, staleness, and category-scoped producer-assertion trust. Its BASIS half is cross-ecosystem Workstream 3D-A; its Ipotio half is Workstream 3D-B.

- **BASIS-side prerequisite:** established by this decision once it is Accepted. While this ADR is Proposed, the prerequisite is proposed, not established.
- **ARCH-GAP-004:** remains open. This ADR does not close it, whether Proposed or Accepted. Closure requires (1) acceptance of this ADR and (2) Ipotio's Workstream 3D-B reconciliation in its own repository.

That reconciliation will need to confirm, at least:

- which of its context categories the upstream platform may offer, and that each maps to a BASIS category and an origin class this decision permits;
- how the upstream platform's own eligibility and freshness rules map onto Decision 7, with the source deciding what is current enough to offer and BASIS deciding what it accepts;
- that the upstream platform preserves value, observation or as-of time, eligibility status, and provenance reference unchanged (Decisions 6, 10);
- that its requesting-workload identity is the originating source identity that category grants name (Decisions 1, 3);
- that stale, unknown, conflicted, or never-observed values are never offered as current and are never laundered into usable context (Decisions 7, 8);
- that its durable asset identity remains mapping provenance, and reaches `device` or `location` only as an explicit, category-authorized context assertion (Decision 13);
- that it claims no producer authority (ADR-0018).

Nothing here requires a BASIS type, field, or vocabulary specific to Ipotio. This ADR does not write Ipotio architecture.

## Alternatives Considered

**Keep all-or-nothing producer trust, and let admitted producers relay upstream context as `trusted_producer_asserted`.** Rejected. It is the trust-laundering path this ADR exists to close. Any admitted upstream source could reach every category through any admitted producer, and the originating source would vanish from provenance.

**Producer-scoped trust only (category grants for producers, none for upstream sources).** Rejected. A producer authorized to relay `risk_context` would accept `risk_context` from every admitted upstream source equally. The producer would act as a privilege amplifier, and admission to intake would silently become category authority.

**Upstream-scoped trust only (the producer as a transparent pipe).** Rejected. It removes the producer's accountability for what it submits, and it contradicts ADR-0008's model, where the gateway admits producers rather than upstream systems. A compromised or misconfigured producer could then carry any category from any source it chose to attribute.

**Treat admitted upstream workloads as producers for context purposes.** Rejected for the reasons in ADR-0018's **Alternatives Considered**: it collapses the upstream and producer roles and their identities.

**Mark admitted upstream context `verified`.** Rejected. BASIS observed nothing. Overloading `verified` would erase the difference between a policy-permitted assertion and an independently confirmed fact, and that difference is exactly what the "never promoted to `verified`" rule protects.

**Drop inadmissible context and continue as though it were absent.** Rejected. Absence is a policy input with permissive cases (Decision 8). Dropping a rejected value is an authorization bypass.

**Let the producer, gateway, or kernel reconcile conflicting values (newest wins, source priority, merging).** Rejected as an ungoverned default. It would make an unreviewed heuristic decisive for authorization. A category-specific reconciliation contract may be proposed later.

**Enforce category trust in `basis-core`.** Rejected. It would make the kernel a workload-trust and provenance-validation service, contrary to kernel isolation ([`kernel-boundary-rules.md`](../kernel-boundary-rules.md)) and the gateway's role as the trust boundary.

**Enforce category trust only at the intake boundary.** Rejected as insufficient. The gateway is the last point before evaluation, and it is the only point that sees both the producer's authenticated identity and the evaluation time. A check made only at intake would make a single intake realization's correctness decisive for what the kernel sees.

**Define a concrete context envelope, per-category schemas, or an Ipotio-shaped contract now.** Rejected as premature, for the same reasons ADR-0018 and ADR-0020 declined to fix a wire shape before the semantics were settled.

## Consequences

### Positive

- Every context category has an explicit, attributable authority basis. Admission alone no longer grants blanket context trust, on either the upstream side or the producer side.
- Trust laundering has a name, a prohibition, and a structural defense: independent grants, preserved origin, and fail-closed admission.
- Freshness becomes part of what "trusted context" means, and the source's truthfulness is kept separate from BASIS's acceptance.
- The compromised-producer blast radius recorded in ADR-0008 is bounded to granted categories.
- The model works identically for any upstream system, and for deployments with none.

### Negative / Tradeoff

- Deployments must author and maintain a category trust policy covering upstream sources, producers, origins, provenance, and freshness. Under default deny, context a deployment previously passed through will stop flowing until it is explicitly granted.
- When implemented, admitted producers lose today's implicit authority over all nine fields. That is a deliberate narrowing of current gateway behavior.
- Fail-closed rejection of a present-but-inadmissible value refuses the whole operation, including when the policy might have permitted it without that value. Integrations must omit what they cannot supply admissibly rather than supply it hopefully.
- Freshness enforcement depends on trustworthy clocks and source times, whose mechanisms are deferred.
- Until an origin-integrity mechanism exists, upstream attribution at the gateway rests on the producer's attestation.

### Security Consequences

**Prevented, when implemented:** the threat cases above, including upstream privilege amplification through a trusted producer, producer category overreach, origin stripping and re-labeling, timestamp refresh, status upgrade, provenance omission, rejected-to-absent bypass, heuristic conflict resolution, and promotion of mapping provenance into context.

**Still requiring follow-on architecture:** end-to-end origin integrity for upstream-originated values; the provenance representation (Decision 5); category-trust-policy administration; clock and skew handling; a category-specific reconciliation contract, if one is ever needed; governed transformations, if any are ever needed; the intake envelope; credential lifecycle (3E); evidence correlation (Workstream 4).

**Residual risks:** a compromised or mistaken upstream source can supply a false value within its granted categories. That value is a trusted assertion, not a verified fact, which is why grants should be narrow. A compromised producer can misattribute origin within its own grants (Decision 11). A compromised category-trust-policy authority can grant too much, just as a compromised policy author can write harmful policy.

## Deferred Decisions

- Category assignments this ADR leaves **open** (see **Current Producer-Only Context Inventory**), as well as any new context category.
- The representation of multi-dimensional context provenance, and any change to the `ProvenanceClassification` vocabulary (`basis-gateway`, `basis-schemas`).
- Category-trust-policy configuration format, storage, administrative surface, approval workflow, and identity or versioning.
- Per-category freshness conditions, durations, clock source, and skew tolerance.
- End-to-end origin-integrity mechanism (signing, message authentication) for upstream-originated values, together with ADR-0018's intake integrity mechanism.
- A governed representation of "explicitly unknown" or other non-current states as policy-visible values.
- Category-specific reconciliation or aggregation contracts.
- Governed value transformations for any category.
- Governed channels for external-origin context other than ADR-0018 intake.
- Whether provenance facets become policy-evaluable inputs.
- Whether context aging invalidates a pending bound operation (ADR-0012 / ADR-0016 family).
- Subject-credential conveyance from upstream (ADR-0018, ADR-0010).
- Upstream and producer credential technology and lifecycle (Workstream 3E, ARCH-GAP-011).
- Intake transport, envelope, schema, serialization, and time format (ADR-0018).
- Complete evidence correlation and operator-facing representation (Workstream 4).
- The one-to-many/batch contract (ADR-0020 Decision 11), including how context applies across several targets.

## Non-Goals

This ADR does not: define Ipotio semantics or modify Ipotio's repository; define a concrete context envelope, schema, or per-category schema; choose a transport, serialization, or signing format; choose credential technology or lifecycle; implement category trust or modify any implementation repository or schema; define a policy authoring language or universal absence semantics; define evidence storage or close Workstream 4; resolve one-to-many/batch semantics; alter ADR-0020 mapping or composition; alter the ADR-0012 binding; define device-credential custody; resolve Workstream 3E or ARCH-GAP-011; close ARCH-GAP-004; modify any Accepted ADR's body; accept ADR-0019 or perform any branding migration; or promote its working vocabulary to the glossary.

## Validation / Implementation Gate

This ADR is `Proposed`. Merging it does not accept it, and neither this proposal nor its later acceptance authorizes implementation. After acceptance, the following gaps between current implementation and this decision become inputs to a bounded implementation plan:

- **`basis-gateway`:** replace the single all-or-nothing producer-only gate with category trust evaluation (origin permission, source grant, producer grant, provenance, freshness at evaluation time, conflict), rejecting pre-kernel; record the admission basis; evolve the provenance representation (Decision 5); keep the all-or-nothing gate's fail-closed behavior for unadmitted producers.
- **`basis-producer`:** represent origin class, originating source, relay attribution, source time, eligibility status, and provenance reference for relayed values; enforce its own category scope locally; never re-label, refresh, or synthesize.
- **Intake realization:** enforce upstream category-supply scope and permitted origins at admission, which remains gated on every ADR-0018 prerequisite.
- **Shared contracts (`basis-schemas`):** only once implementation proves a stable shape, per [`ecosystem-contract-inventory.md`](../architecture/ecosystem-contract-inventory.md).

`basis-core` is expected to need no change for this decision. ADR-0018 Decision 4's interim posture remains in force until an implementation can establish every admissibility condition.

## References

- [ADR-0007](0007-adapter-evidence-construction.md): adapter evidence construction
- [ADR-0008](0008-producer-workload-authentication-and-admission.md): producer authentication and admission; producer vs. authorization subject; category scope deferred
- [ADR-0009](0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md): trusted producer mTLS ingress
- [ADR-0010](0010-establish-basis-producer-as-operation-producer-runtime.md): operation-producer runtime
- [ADR-0011](0011-protocol-execution-role-and-bounded-reference-topology.md): protocol-executor role; credential classes
- [ADR-0012](0012-authorization-to-execution-binding.md): authorization-to-execution binding
- [ADR-0016](0016-bounded-target-replay-freshness-posture.md): bounded-target replay/freshness posture
- [ADR-0018](0018-upstream-supervisory-producer-intake-boundary.md): producer-intake boundary; Decision 4 interim context posture
- [ADR-0019](0019-basitra-ecosystem-identity-and-terminology-hierarchy.md) (Proposed): Basitra terminology
- [ADR-0020](0020-operation-to-authorization-mapping-and-composition-boundary.md): operation-to-authorization mapping; mapping provenance is not context
- [`operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md): §1, §3 (category trust open), §4 (field ownership; absence), §7 (provenance vocabulary), §8 (not-executed family)
- [`operation-aware-evaluation-semantics.md`](../architecture/operation-aware-evaluation-semantics.md): §10 missing-context behavior
- [`condition-operator-semantics.md`](../architecture/condition-operator-semantics.md): §7–§8 absent-value semantics
- [`operation-aware-authorization-model.md`](../architecture/operation-aware-authorization-model.md): §3 context categories
- [`kernel-boundary-rules.md`](../kernel-boundary-rules.md)
- [`threat-model.md`](../security/threat-model.md): §7.2, §7.5, §7.7, §11
- Ipotio architecture repository: ARCH-GAP-004 and Workstream 3D-B (cross-repository reference, not hyperlinked)
