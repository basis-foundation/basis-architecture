# ADR-0009: Trusted Producer mTLS Ingress and Gateway Certificate Handoff

## Status

Proposed

## Context

[ADR-0008](0008-producer-workload-authentication-and-admission.md) adopted mutual TLS as the first normative producer-to-gateway workload-authentication profile and authorized, without implementing, a bounded operation-producer reference slice. The slice's own implementation plan, [`docs/architecture/bounded-operation-producer-reference-implementation-plan.md`](../architecture/bounded-operation-producer-reference-implementation-plan.md), named the mTLS termination topology as a hard, blocking implementation gate — "Phase 1A" — and specified in advance what must happen if direct application-level TLS termination proved unsafe: implementation stops, and this repository's architecture is updated in a small follow-up planning PR before any proxy-backed fallback may be built.

`basis-gateway`'s Phase 1A spike (`docs/spikes/producer-mtls-certificate-exposure.md`, merged to `main` at `07586110b847d2c8fe5dfadd7a3eba027bc74024`) completed that gate and reached **Outcome B: direct application termination is not viable.** Uvicorn can require and validate producer client certificates at the TLS layer — that half of the spike's checklist passed cleanly, across 14 executed tests driving real TLS handshakes. But no documented, stable boundary carries the validated peer certificate from Uvicorn into `basis-gateway`'s ASGI application code: none of Uvicorn's three shipped HTTP protocol implementations populates the ASGI `scope`'s `"extensions"` key with certificate data, and the standardized mechanism that would (the ASGI TLS extension) remains unimplemented in the current Uvicorn release line after a four-year-old, still-open upstream effort. The spike also demonstrated, directly and empirically, that an ordinary HTTP header carrying certificate-shaped data (`X-Client-Cert`, `X-SSL-Client-Cert`, and similar) is forwarded to the ASGI application completely unmodified by the current stack — confirming that such a header is not a trust boundary unless something is specifically responsible for overwriting it.

This is the decision gate the implementation plan's own §11 named in advance: "which proxy terminates mTLS for the reference environment; how the proxy-to-gateway channel is isolated; why ordinary callers cannot reach that channel directly; how certificate-derived identity is conveyed across it; how spoofed identity headers are stripped or overwritten; what fact, precisely, the gateway trusts about the proxy; how local tests prove that trust boundary holds; whether the resulting topology still conforms to ADR-0008." This ADR, together with its supporting architecture document, answers that gate.

## Problem

How can the bounded BASIS reference deployment terminate producer mTLS outside Uvicorn, transport the authenticated peer-certificate fact into `basis-gateway`, and preserve ADR-0008's requirement that producer identity cannot be spoofed or substituted by ordinary caller-controlled HTTP input? The answer must establish both certificate-to-connection authenticity at the ingress and assertion authenticity across the ingress-to-gateway hop — neither alone is sufficient, and the Phase 1A spike's own header-spoofing observation demonstrates exactly what is missing if only the first is addressed.

## Decision

**BASIS adopts a trusted mTLS ingress proxy as part of the gateway authentication boundary for the bounded reference deployment.** The proxy terminates producer TLS, requires and validates the producer's client certificate against a deployment-configured trust anchor, and forwards the validated leaf certificate to `basis-gateway` over a channel the gateway can trust independent of the certificate's own content. `basis-gateway` application code — not the proxy — parses the forwarded certificate, derives the producer workload identity from its URI SAN, and performs producer admission, exactly as ADR-0008 already requires.

The full specification — proxy technology selection, protected backend channel, certificate handoff contract, header sanitization rule, trust-fact matrix, processing sequence, failure semantics, threat analysis, and Phase 1B implementation requirements — is defined in the companion architecture document, [`docs/architecture/producer-mtls-proxy-trust-boundary.md`](../architecture/producer-mtls-proxy-trust-boundary.md), which this ADR normatively references and does not duplicate. In summary:

- **Selected ingress technology: NGINX**, chosen over HAProxy, Envoy, and stunnel after evaluation against current primary-source documentation for all four (architecture document §6–§7). NGINX natively produces this decision's selected certificate encoding with no conversion step, has the smallest configuration surface for the one route this slice requires, and needs no dynamic control plane. stunnel is rejected outright: it has no HTTP-layer awareness and cannot inject the required header at all.
- **Selected backend channel: a Unix domain socket** between NGINX and `basis-gateway` (Uvicorn's `--uds` mode), with restrictive filesystem permissions. `basis-gateway` opens no network-reachable TCP listener in this mode (architecture document §9).
- **Selected certificate handoff: the trusted ingress forwards the authenticated leaf client certificate itself**, not a proxy-derived identity string, in a private internal header (`X-BASIS-Producer-Client-Cert`, URL-escaped PEM), unconditionally overwritten by the ingress on every forwarded request regardless of any client-supplied value with the same name (architecture document §10–§11).
- **URI SAN derivation and producer admission remain entirely inside `basis-gateway`**, unmodified from ADR-0008. The ingress performs no admission decision, infers no BASIS role, determines no authorization subject, evaluates no policy, and never calls `basis-core`.

**This ADR's relationship to ADR-0008:** the trusted mTLS ingress is part of the gateway authentication boundary for the bounded reference deployment. It performs TLS-level certificate validation on behalf of that boundary. `basis-gateway` application code still derives the producer workload identity from the authenticated leaf certificate and remains solely responsible for admission. mTLS remains the authentication mechanism; certificate possession remains proven at the TLS boundary; the gateway application does not trust a caller-created identity string; URI SAN interpretation and producer admission remain gateway-owned. This interpretation was evaluated against ADR-0008's text and found fully consistent — no contradictory language was identified, and ADR-0008 is not modified by this ADR.

This ADR does not implement a proxy, does not implement gateway header parsing, does not implement URI SAN extraction or admission, does not create `OperationProducerTrust` fields, does not create a producer certificate or producer runtime, and does not modify `basis-gateway`, `basis-identity`, `basis-adapters`, `basis-core`, or `basis-schemas`. It records the architectural decision and, once accepted, unblocks Phase 1B of the bounded reference implementation plan.

## Alternatives considered

**Custom Uvicorn protocol subclass injecting certificate data into ASGI `scope`.** Technically demonstrated as *possible* at the raw `asyncio`/`ssl` layer by the Phase 1A spike, but requires subclassing Uvicorn's internal, non-public `asyncio.Protocol` implementations — there is no supported extension point for `scope` construction, only a full protocol-class replacement. Putting a security-critical identity boundary on unstable, private implementation details was exactly what the Phase 1A spike's own Outcome B criteria disqualify. **Disposition: rejected.**

**A different ASGI server with standardized TLS-extension support.** No currently released ASGI server was found, during this ADR's research, to implement the ASGI TLS extension either — the specification-level blockers the Phase 1A spike documented are upstream of any single server's implementation choice. Migrating servers would also carry ecosystem-maturity, packaging, and operational costs disproportionate to a bounded reference slice, without a demonstrated server that actually solves the underlying gap. **Disposition: not selected; not evaluated further without evidence a specific alternative server actually closes the gap.**

**Trusted reverse proxy terminating mTLS.** The selected direction. See architecture document §6–§9 for the technology evaluation and topology.

**Bare certificate-forwarding header over an ordinary, network-reachable backend.** Rejected outright — the Phase 1A spike directly demonstrated that a caller-writable header over a network-reachable backend is not a trust boundary; nothing prevents an attacker from bypassing the proxy and writing the header directly. **Disposition: rejected.**

**Proxy-derived URI SAN only, rather than the full leaf certificate.** Would move certificate identity derivation out of `basis-gateway` and into proxy configuration, requiring two independently versioned components (a proxy configuration file and gateway Python code) to agree on SAN-selection and exact-match semantics, and diluting ADR-0008's assignment of that logic to the gateway. **Disposition: rejected; see architecture document §10.**

**Generic subject DN / Common Name forwarding.** Common Name is not an identity source under ADR-0008. **Disposition: rejected.**

**Independently authenticated backend mTLS between the proxy and the gateway**, rather than a Unix domain socket. A stronger boundary in principle, removing even same-host process-identity reliance, but adds a second certificate lifecycle for no demonstrated necessity given the Unix-socket boundary already satisfies this ADR's requirements. **Disposition: evaluated, deferred as a future option; see architecture document §9.**

## Consequences

**Positive:**

- Phase 1B of the bounded operation-producer reference implementation plan is unblocked, contingent only on this ADR's formal acceptance.
- The trust boundary the Phase 1A spike found missing is now fully specified — proxy technology, backend channel, certificate encoding, header contract, and failure semantics — precisely enough that Phase 1B does not need to invent any of it during implementation.
- ADR-0008's ownership model (gateway-owned URI SAN derivation and admission) is preserved exactly, closing the topology gap without reopening the identity-model decision.
- The reference topology is fully air-gap-compatible and requires no new infrastructure class (no service mesh, no dynamic control plane, no cloud dependency).

**Tradeoffs:**

- The bounded reference deployment now depends on a second local process (NGINX) in addition to `basis-gateway`, with its own configuration file that is part of this profile's trusted computing base — a compromised or misconfigured proxy can forge producer-certificate assertions (architecture document §19, items 4–5).
- Same-host access to the Unix domain socket is a residual attack surface that filesystem permissions bound but do not eliminate (architecture document §9, §19 item 3) — Unix sockets provide no process-identity attestation beyond filesystem-permission enforcement.
- A future, larger BASIS ingress topology (additional routes, additional callers, mesh integration) may find NGINX's evaluation reversed in favor of Envoy or another technology; this ADR does not extend its selection beyond the bounded reference slice (architecture document §26).

## Validation / implementation gate

Formal acceptance of this ADR, through this repository's ADR governance process ([`docs/adr/README.md`](README.md#lifecycle-states)), is the gate before: `basis-gateway` begins any implementation of the trusted-ingress configuration, the internal certificate-header contract, or Phase 1B's admission logic built on top of it; and before any `basis-schemas` proposal related to this boundary is considered (none is anticipated — see architecture document §24). This ADR, once merged, does not itself constitute acceptance, consistent with this repository's established convention (see [ADR-0006](0006-evaluation-orchestration-layer.md)'s implementation note and [`docs/adr/README.md`](README.md#lifecycle-states)).

## References

- [`docs/architecture/producer-mtls-proxy-trust-boundary.md`](../architecture/producer-mtls-proxy-trust-boundary.md) — the detailed architecture this ADR normatively references: topology, trust facts, proxy evaluation, certificate handoff, failure semantics, threat analysis, test expectations, and implementation consequences
- [ADR-0008](0008-producer-workload-authentication-and-admission.md) — producer workload authentication and admission, preserved unchanged by this ADR
- [`docs/architecture/bounded-operation-producer-reference-implementation-plan.md`](../architecture/bounded-operation-producer-reference-implementation-plan.md) — the implementation plan whose §11 named this decision gate in advance and whose Phase 1A/1B sequencing this ADR unblocks
- [`docs/architecture/operation-producer-and-execution-boundary.md`](../architecture/operation-producer-and-execution-boundary.md) — the logical-role and trust-establishment architecture this ADR's topology operates within
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the boundary rule this ADR's placement of certificate parsing inside `basis-gateway`, never `basis-core`, observes
- [`docs/security/threat-model.md`](../security/threat-model.md) — trusted-adapter and compromised-component threat framing this ADR's threat analysis extends to the ingress boundary
- `basis-gateway`: `docs/spikes/producer-mtls-certificate-exposure.md` (merged to `main` at `07586110b847d2c8fe5dfadd7a3eba027bc74024`) — the Phase 1A spike evidence this ADR is built on; `tests/integration/mtls_certs.py`, `tests/integration/spike_app.py`, `tests/integration/test_producer_mtls_transport.py` — the retained, reusable test harness
