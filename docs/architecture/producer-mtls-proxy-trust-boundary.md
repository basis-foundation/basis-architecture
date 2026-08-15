# Producer mTLS Proxy Trust Boundary

**Status:** Proposed architecture, supporting [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md). Not yet accepted. Not implemented in any repository.

**Companion documents:** [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md) (the durable decision this document supports and is normatively referenced by), [ADR-0008](../adr/0008-producer-workload-authentication-and-admission.md) (producer workload authentication and admission — preserved unchanged), [`bounded-operation-producer-reference-implementation-plan.md`](bounded-operation-producer-reference-implementation-plan.md) (Phase 1A/1B implementation planning this document unblocks), [`operation-producer-and-execution-boundary.md`](operation-producer-and-execution-boundary.md), [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md), [`docs/security/threat-model.md`](../security/threat-model.md), `basis-gateway`'s `docs/spikes/producer-mtls-certificate-exposure.md` (Phase 1A spike evidence this document is built on).

---

## 1. Status and Purpose

`basis-gateway`'s Phase 1A mTLS certificate-exposure spike, merged to `main` at `07586110b847d2c8fe5dfadd7a3eba027bc74024`, reached **Outcome B: direct application termination is not viable.** Stock Uvicorn, across all three of its shipped HTTP protocol implementations, does not expose a validated peer certificate to ASGI application code through any documented or stable boundary; the one standardized mechanism that would provide it, the ASGI TLS extension, remains unimplemented in the current Uvicorn release line after more than four years. This finding is not a private implementation gap `basis-gateway` can close by itself — it is an acknowledged, long-standing limitation of the Uvicorn/ASGI ecosystem, confirmed by this repository's own direct inspection of Uvicorn `0.52.3`'s protocol implementations and by the still-open, years-stalled upstream PR that would add the extension.

The bounded producer reference implementation plan named this exact outcome in advance and gated on it: "Before any proxy fallback is implemented, this planning document must first be updated in a small follow-up architecture/planning PR." This document, together with [ADR-0009](../adr/0009-trusted-producer-mtls-ingress-and-gateway-certificate-handoff.md), is that follow-up. It defines the trusted mTLS ingress topology precisely enough that Phase 1B of the bounded producer reference implementation can proceed once ADR-0009 is formally accepted, without inventing proxy trust semantics, certificate-forwarding semantics, header trust, backend channel protection, URI SAN ownership, or failure behavior during implementation.

This is a bounded **reference implementation topology** for the first producer slice ADR-0008 authorizes. It is not a permanent production ingress mandate for every BASIS deployment (§26).

---

## 2. Phase 1A Evidence

Summarized from `basis-gateway`'s spike document; not restated in full here.

- Uvicorn can be configured to require and validate producer client certificates (`ssl_cert_reqs=CERT_REQUIRED`, `ssl_ca_certs`); untrusted-CA, missing-certificate, and expired-certificate connections all fail the TLS handshake before the ASGI application is invoked, across 14 executed integration tests against a real Uvicorn subprocess and real TLS handshakes.
- Peer-certificate data, including the URI SAN, is fully obtainable at the raw `asyncio`/`ssl` transport layer (`transport.get_extra_info("ssl_object").getpeercert()`) — the underlying cryptographic material and validation are not the problem.
- None of Uvicorn's three shipped `asyncio.Protocol` implementations (`H11Protocol`, `HttpToolsProtocol`, `ZttpProtocol`) reads that transport-layer certificate data when constructing the ASGI `scope`. A successfully validated mTLS connection produces an ASGI `scope` with no `"extensions"` key at all.
- The one standardized mechanism that would carry this data into application code, the [ASGI TLS extension](https://asgi.readthedocs.io/en/latest/specs/tls.html) (`scope["extensions"]["tls"]`), is not implemented by any released Uvicorn version; the four-year-old upstream PR remains open, blocked on unresolved questions in the ASGI specification itself.
- The spike explicitly evaluated and rejected subclassing Uvicorn's internal protocol classes to inject certificate data into `scope` manually: no supported extension point exists for scope construction, and reaching into unstable internals to carry a security-critical identity fact is precisely the kind of private/unsupported dependency Outcome B's own criteria disqualify.
- An ordinary, already-mTLS-authenticated caller that sets `X-Client-Cert`, `X-SSL-Client-Cert`, `X-Client-DN`, or `X-Forwarded-Client-Cert` with an attacker-chosen value has that value forwarded to the ASGI application **byte-for-byte unmodified**. Nothing in the Uvicorn/FastAPI/Starlette stack strips, validates, or protects any of these header names. This is the direct evidence for §11's header-sanitization requirement.

The spike's own stated implication: "Phase 1B is blocked pending a reviewed proxy-to-gateway trust-boundary plan." This document is that plan.

---

## 3. Governing ADR Constraints

This document does not reopen, and treats as fixed, everything ADR-0008 already decided: mTLS as the first normative producer authentication profile; URI SAN as the sole producer workload identity source, taken verbatim, with no Common Name fallback; exactly one eligible URI SAN required, fail-closed otherwise; exact, case-sensitive producer admission with no prefix/suffix/substring/wildcard/case-insensitive matching; authentication and admission as two independent steps; producer workload identity distinct from authorization-subject identity, never automatically derived from one another; `OPERATION_PRODUCER_SUBJECT_IDS` retained as a compatibility path, not superseded; REST as the bounded reference adapter; retain-before-mint evidence semantics; and the reference slice stopping before protocol execution. Where this document's topology touches any of these, it is implementing the *transport* that carries the already-decided facts to the gateway, not deciding the facts themselves.

---

## 4. Problem Statement

The bounded reference deployment needs to terminate producer mTLS outside the Uvicorn process that runs `basis-gateway`, transport the authenticated peer-certificate fact from that termination point into gateway application code, and preserve ADR-0008's requirement that producer identity cannot be spoofed or substituted by ordinary caller-controlled HTTP input. Two properties are required together, and neither alone is sufficient:

1. **Certificate-to-connection authenticity at the ingress** — the component terminating TLS must actually have validated the producer's certificate against a configured trust anchor, and the fact it forwards must be the real, TLS-authenticated peer certificate, not a value taken from anywhere else.
2. **Assertion authenticity across the ingress-to-gateway hop** — the channel carrying that fact from the ingress to `basis-gateway` must be one no other party can write to, and any externally supplied copy of the same-shaped data must be discarded before the ingress's own copy is attached.

A topology that satisfies only the first (for example, a caller-writable header carried over an ordinary network-reachable backend) reintroduces exactly the header-spoofing risk the Phase 1A spike demonstrated directly (§2). A topology that satisfies only the second (an isolated channel carrying an unauthenticated or caller-supplied value) protects a channel that carries nothing trustworthy.

---

## 5. Trust-Boundary Requirements

Restated as requirements this document's topology must satisfy, each traceable to a source:

- No network-reachable path may allow a caller to reach `basis-gateway` while bypassing the trusted ingress and still have proxy-authenticated producer-certificate metadata accepted (§9, §18).
- The ingress must unconditionally remove or overwrite any externally supplied copy of the reserved internal certificate header before attaching its own (§11, evidenced directly by §2's header-spoofing observation).
- The certificate fact that crosses the ingress-to-gateway hop must be the authenticated leaf certificate itself, not a proxy-derived identity string, preserving ADR-0008's assignment of URI SAN derivation and admission to `basis-gateway` (§10, §12).
- `basis-gateway` must not gain a network TCP listener that ordinary callers can reach directly in trusted-proxy producer-mTLS mode (§9).
- The bearer-authenticated authorization subject path must be forwarded unchanged and must remain entirely independent of the certificate-handoff mechanism (§14).
- The topology must operate fully offline: local trust anchors, no public OCSP, no cloud load balancer, no Kubernetes, no service mesh, no dynamic control plane (§21).
- The selected mechanism must be provable by a deterministic, CI-feasible local test, not merely asserted (§22, Appendix B).

---

## 6. Proxy Technology Evaluation

Four candidates were evaluated against current, primary-source documentation — no blog posts, forum threads, or Stack Overflow answers were used as the basis for capability claims.

### NGINX

`ngx_http_ssl_module` (nginx.org, current mainline documentation) confirms: `ssl_verify_client on | optional | optional_no_ca` requests and validates a client certificate against `ssl_client_certificate` (a configured CA bundle); `$ssl_client_escaped_cert` (added 1.13.5) returns the validated client certificate in URL-encoded PEM format, purpose-built for use in `proxy_set_header`; `$ssl_client_verify` distinguishes `SUCCESS`/`FAILED:reason`/`NONE`. `ngx_http_upstream_module` confirms upstream servers may be specified as a `unix:` socket path (`server unix:/tmp/backend3;`), and `proxy_pass` to such an upstream is standard, documented usage — mixing TCP and Unix-domain-socket upstreams in one group is explicitly supported. `proxy_set_header NAME value` unconditionally sets the named outbound header to the given value, discarding any inbound value of the same name; this is nginx's standard mechanism for overwriting, not appending to, a header. `listen unix:/path;` is documented, standard `ngx_http_core_module` syntax for the frontend socket as well, though this topology uses a TCP frontend listener for the public producer-facing side and a Unix-socket backend connection to the gateway (§9).

### HAProxy

Primary documentation and current `bind`/`server` directive references confirm: `bind` accepts `ca-file` (client trust anchor) and `verify {none|optional|required}`; `verify required` aborts the handshake for a missing or untrusted client certificate; `ssl_c_verify`, `ssl_c_s_dn`, `ssl_c_i_dn`, `ssl_c_sha1`, `ssl_c_der` fetches expose verified-connection certificate facts to `http-request set-header`. HAProxy has no built-in fetch that produces the full certificate in PEM format directly — `ssl_c_der` returns DER, and `,base64` converts it to base64-encoded DER; producing URL-encoded PEM (this document's selected encoding, §13) requires either a Lua conversion script or accepting base64 DER as the encoding instead. `server` lines accept a Unix-domain-socket address (`unix@/path/to/socket`), and `bind`/`unix-bind` support Unix-socket listeners with a configurable path prefix. HAProxy is fully capable of this topology; its native output format for the certificate does not match this document's preferred encoding without an additional conversion step.

### Envoy

Envoy's HTTP connection manager `x-forwarded-client-cert` (XFCC) mechanism, documented at `envoyproxy.io`, includes a `Cert` element defined as "the entire client certificate in URL encoded PEM format" and a `Chain` element for the full presented chain, selected via the `forward_client_cert_details`/`set_current_client_cert_details` HTTP connection manager options (`SANITIZE_SET` mode fully replaces any inbound XFCC value, satisfying the sanitization requirement natively). Envoy's `Pipe` address type (`config.core.v3.Pipe`, `address.proto`) is a first-class Unix-domain-socket address usable for both listeners and cluster upstream endpoints, including Linux abstract-namespace sockets. Envoy is technically fully capable of every requirement in §5, including native production of exactly this document's preferred encoding. Its configuration model (bootstrap YAML, listener/filter-chain/cluster resource graph, optional xDS control plane) is materially more complex than this bounded reference deployment's scope requires, and its operational and dependency footprint is disproportionate to a small, air-gapped reference slice whose entire routing surface is one endpoint.

### stunnel

The stunnel manual (`stunnel.org`, current manual page) confirms `verifyPeer`/`verifyChain` with `CAfile`/`CApath` for client-certificate validation, and confirms that an address parameter to `accept`/`connect` may be "a Unix socket path (Unix only)." stunnel is a generic TLS-encryption wrapper for arbitrary TCP connections — it has no HTTP-layer awareness. Its `protocol` option negotiates a small, fixed set of application-level startup handshakes (`cifs`, `capwin`, `connect`, `exec`, `imap`, `nntp`, `pgsql`, `pop3`, `proxy`, `smtp`, `socks`, and similar) for pre-TLS protocol negotiation; none of these give stunnel the ability to parse an HTTP request and inject or overwrite an HTTP header. stunnel cannot produce this document's required certificate-handoff header at all, regardless of encoding choice. This is a capability gap, not a configuration-complexity tradeoff.

### Summary of the primary-source findings

| Requirement | NGINX | HAProxy | Envoy | stunnel |
| - | - | - | - | - |
| mTLS client-cert validation against configured CA | Yes (`ssl_verify_client`, `ssl_client_certificate`) | Yes (`verify required`, `ca-file`) | Yes (transport socket `require_client_certificate`) | Yes (`verifyPeer`/`verifyChain`, `CAfile`) |
| Reject missing/invalid/untrusted/expired certificate before backend reached | Yes | Yes | Yes | Yes |
| Access authenticated leaf certificate | Yes (`$ssl_client_escaped_cert`) | Yes (`ssl_c_der`, DER only) | Yes (XFCC `Cert=`, native URL-encoded PEM) | Yes, but not exposable as an HTTP header (no HTTP awareness) |
| Forward leaf certificate as an HTTP header | Yes | Yes (needs DER→PEM conversion for this document's encoding) | Yes, natively in the target encoding | **No** |
| Unconditional inbound-header overwrite | Yes (`proxy_set_header`) | Yes (`http-request set-header`) | Yes (`SANITIZE_SET`) | N/A — no header concept |
| Unix-domain-socket backend | Yes (`server unix:...`) | Yes (`server ... unix@...`) | Yes (`Pipe` address) | Yes (address-parameter Unix path), but moot given no HTTP capability |
| Minimal, boring, auditable configuration for one route | Yes | Yes | No — xDS/resource-graph model is disproportionate for one static route | N/A |
| No control plane / dynamic config required | Yes | Yes | Optional (static bootstrap is usable, but the ecosystem defaults toward xDS) | Yes |

---

## 7. Selected Reference Ingress Technology

**NGINX is selected for the bounded reference deployment.**

NGINX is the only candidate that satisfies every requirement in §5 with a native, single-purpose configuration directive and no conversion step: `$ssl_client_escaped_cert` produces exactly this document's selected encoding (§13) without a scripting layer, `proxy_set_header` unconditionally overwrites the reserved internal header, and `server unix:...;` provides the protected backend channel — all with a static `nginx.conf` a reviewer can read start to finish. HAProxy is a close, fully capable second choice, deferred only because its native certificate fetch requires an additional conversion step (Lua or a base64-DER encoding choice) to reach the same encoding nginx produces directly; either the Lua step or accepting DER/base64 as the encoding are legitimate alternatives if a future implementation phase finds a concrete reason to prefer HAProxy. Envoy is rejected for this bounded slice specifically because of its disproportionate configuration and operational complexity relative to one static route serving one bounded reference deployment — not because of any capability gap; a future, larger BASIS ingress topology (multiple routes, dynamic backend discovery, mesh integration) is exactly the kind of scenario where Envoy's evaluation could reverse (§26). stunnel is rejected outright: it structurally cannot perform the one operation this topology's certificate handoff requires, injecting an HTTP header derived from the validated peer certificate, because it has no HTTP-layer awareness at all.

NGINX is mature (first released 2004, in continuous production use across the majority of the public internet), has a small, well-audited configuration surface for the single route this reference profile requires, is packaged in every mainstream Linux distribution's official repositories with no additional build step (the `ssl` module needed here ships in nginx's standard prebuilt packages), requires no dynamic control plane, and is trivially reproducible in CI (§22).

---

## 8. Selected Reference Topology

```text
operation producer
    │  HTTPS + required client certificate
    │  Authorization: Bearer <basis_local_token> (same request)
    ▼
NGINX — dedicated producer mTLS listener, one route only
    │  requires + validates client certificate (ssl_verify_client on;
    │  ssl_client_certificate <producer CA bundle>)
    │  discards any inbound X-BASIS-Producer-Client-Cert
    │  (proxy_set_header always overwrites; nothing "hides" a header —
    │  overwriting is sufficient, see §11)
    │  sets X-BASIS-Producer-Client-Cert: $ssl_client_escaped_cert
    │  forwards Authorization unchanged
    ▼
Unix domain socket (§9), restrictive filesystem permissions
    ▼
basis-gateway (Uvicorn, --uds, no TCP listener in this mode)
    │  parses X-BASIS-Producer-Client-Cert (§10)
    │  derives exactly one URI SAN (ADR-0008)
    │  performs exact producer admission (ADR-0008)
    │  independently authenticates the bearer subject (existing dispatch)
    ▼
basis-core (unchanged)
```

Only the route this reference slice needs (`POST /v1/evaluate/operation-aware`) is exposed through this listener (§20). This is a dedicated reference producer-mTLS listener, not a general-purpose ingress redesign — it does not attempt to solve permanent coexistence with browser traffic, ordinary bearer-only callers, every gateway route, or future console traffic (§20).

---

## 9. Protected Backend Channel

**Selected: a Unix domain socket with restrictive filesystem permissions**, per the following evaluation.

**Unix domain socket (selected).** Uvicorn's documented `--uds PATH` flag binds the ASGI application to a Unix-domain-socket listener instead of a TCP port — explicitly documented as intended for exactly this pattern ("useful if you want to run Uvicorn behind a reverse proxy"). NGINX's `server unix:PATH;` upstream syntax connects to it (§6). In this mode `basis-gateway` opens no network-reachable TCP listener at all for the trusted-proxy producer-mTLS profile; the only way to reach the ASGI application is through the socket file. The socket is created by whichever process starts first per the deployment's process-supervision configuration (a reference deployment starts `basis-gateway` first, waits for the socket file to exist, then starts NGINX; this ordering is deployment/process-supervisor configuration, not application code); its owning user/group and mode (recommended `0770` or stricter, owned by a dedicated service account shared only by the NGINX and gateway processes) determine which local processes may connect to it. Cleanup is the deployment's responsibility (removing a stale socket file before startup, or using a process supervisor's `RemoveOnStop`-equivalent behavior) — this document does not invent a new mechanism for it. This works identically in containers and in Codespace/CI environments as an ordinary filesystem path; no additional container capability is required. Residual risk: the socket has no notion of authenticating the identity of the connecting process beyond filesystem permission — a second process running as the same user/group as NGINX could also connect to the socket. This is a same-host attack surface the filesystem permission boundary bounds, not eliminates (§18).

**Loopback TCP (evaluated, not selected).** A loopback-only TCP port (`127.0.0.1:PORT`) is simpler to reason about in some deployment tooling but introduces failure modes a Unix socket does not: an accidental `0.0.0.0` bind (a one-line configuration mistake) exposes the port to the network; other local processes and, in a shared-namespace container, other containers on the same network namespace can reach a loopback port depending on the container runtime's networking mode; Codespace/devcontainer port-forwarding conventions sometimes forward loopback ports outward by convention or misconfiguration, which a Unix socket path cannot be forwarded by the same mechanism. A loopback port carrying a trusted header is not automatically equivalent to a protected Unix socket — the header's trust still depends on nothing else reaching that port, and a Unix socket makes "nothing else reaches it" a filesystem-permission fact rather than a network-configuration fact.

**Independently authenticated backend mTLS (evaluated, not selected).** The proxy could authenticate to `basis-gateway` with its own separate client certificate over a second TLS session, independent of the producer's certificate. This is a stronger boundary in principle — it removes even same-host process-identity reliance — but adds a second certificate lifecycle (proxy-to-gateway, distinct from producer-to-proxy) and a second TLS termination inside the same process for which §2 already found no safe application-layer boundary. Deferred as a future option if a concrete deployment need for stronger same-host isolation is demonstrated; not required for the bounded reference slice given the Unix-socket boundary already satisfies §5's requirements.

**Decision:** Unix domain socket, restrictive permissions, `basis-gateway` listening only on that socket in trusted-proxy producer-mTLS mode.

---

## 10. Certificate Handoff Contract

**Selected: the trusted ingress forwards the authenticated leaf client certificate itself; `basis-gateway` derives producer identity from it.** This is Option A from the candidate set the implementation plan already named. It is preferred, and Option B (forwarding only a proxy-derived identity string) is rejected, because it preserves ADR-0008's assignment of URI SAN interpretation and producer admission to `basis-gateway` unchanged: the ingress proves nothing about identity semantics, only that the presented certificate passed TLS-layer validation. Option C (forwarding a Common Name or generic subject DN) is rejected outright — Common Name is not an identity source under ADR-0008, and forwarding one would invite exactly the fallback this architecture already forbids.

Moving identity derivation into the proxy (Option B) would also mean two independently versioned components (an ingress configuration file and `basis-gateway`'s Python code) would need to agree on SAN-selection and exact-match semantics — the multi-SAN and zero-SAN fail-closed rules ADR-0008 requires are exactly the kind of logic this architecture keeps inside `basis-gateway`, tested with the language and test harness the rest of the gateway's admission logic already uses (§22), not reimplemented a second time in proxy configuration language.

---

## 11. Header Sanitization and Spoof Resistance

The reserved internal header name is **`X-BASIS-Producer-Client-Cert`**, following the `X-BASIS-...` naming convention already established by this ecosystem's other BASIS-specific transport concerns. It is a **private internal ingress-to-gateway transport contract**, not a public caller-facing contract, and is not published to `basis-schemas` (§24).

**Sanitization rule, mandatory:** the NGINX configuration for the producer mTLS listener sets `proxy_set_header X-BASIS-Producer-Client-Cert $ssl_client_escaped_cert;` unconditionally, on every request that reaches the proxy stage (i.e., every request whose TLS handshake already succeeded). `proxy_set_header` in nginx **replaces** the named outbound header's value; it does not append to, or preserve alongside, any client-supplied value with the same name. There is no code path in this configuration where a client-supplied `X-BASIS-Producer-Client-Cert` value reaches `basis-gateway` unmodified — the directive always executes, for every request the proxy forwards, and always writes nginx's own value. This is the direct configuration-level answer to the exact spoofing behavior the Phase 1A spike observed for `X-Client-Cert`/`X-SSL-Client-Cert`/`X-Client-DN`/`X-Forwarded-Client-Cert` (§2): those headers were forwarded unmodified specifically because nothing overwrote them; this topology's one header is always overwritten.

**Required properties:**

- **Exact header name:** `X-BASIS-Producer-Client-Cert`. Case-insensitive per HTTP header semantics; `basis-gateway` parsing treats it as such.
- **Encoding:** URL-escaped PEM, produced by nginx's `$ssl_client_escaped_cert` (§13).
- **Scope of value:** the leaf client certificate only, not the full presented chain. The producer's mTLS profile (ADR-0008) requires exactly one eligible URI SAN on the leaf certificate; chain material is not needed to derive it, and forwarding chain material would enlarge the header for no admission-relevant benefit.
- **Maximum accepted size:** bounded by nginx's `large_client_header_buffers`/`proxy_buffer_size`-class limits on the ingress side and by `basis-gateway`'s own request-header size limits on the gateway side; a reference deployment should set an explicit, small maximum (a single RSA-2048 or typical ECDSA leaf certificate PEM is well under 4 KB before URL-escaping expansion; a generous bound such as 16 KB comfortably covers realistic certificate sizes including reasonable extension data without inviting a header-size denial-of-service surface). Both bounds are ordinary deployment configuration values, not new architecture.
- **Duplicate-header behavior:** `basis-gateway` must treat more than one occurrence of the header as a malformed assertion and fail closed (Layer 2, §18) — this is a defense-in-depth check; in the selected topology the ingress writes exactly one occurrence, but the gateway does not assume the proxy is the only thing that could ever write to a socket it trusts (§9's residual risk) and validates shape rather than assuming it.
- **Malformed-value behavior:** a value that does not URL-decode to a well-formed PEM certificate, or that decodes but fails X.509 parsing, is rejected at Layer 2 (fail closed; no producer admission).
- **Missing-value behavior:** the absence of the header on a request that reached `basis-gateway` through the trusted-proxy producer-mTLS listener is treated as "no producer certificate presented" — the request may proceed only as an ordinary bearer-authenticated caller (subject to the existing `UntrustedOperationProducerContextError` rejection of producer-only fields), never as an admitted producer.
- **Logging/redaction:** `basis-gateway` does not log the raw header value. Diagnostics and audit record only the derived URI SAN string (itself a non-secret, deployment-registered identity value per ADR-0008) and the admission boolean/reason, consistent with the bounded implementation plan's existing §19 finding (§17 of this document).

---

## 12. URI SAN Derivation Ownership

Unchanged from ADR-0008 and the implementation plan: `basis-gateway` parses the forwarded leaf certificate (using its already-released `cryptography` dependency), requires exactly one eligible URI SAN, fails closed on zero or multiple, never falls back to Common Name, and never accepts a caller-supplied or request-body identity value. This document changes only *how* the validated certificate bytes reach the parsing code (§10) — the parsing, SAN-selection, and fail-closed rules themselves are ADR-0008's, unmodified. The certificate identity extraction module remains inside `basis-gateway` (for example `src/basis_gateway/auth/producer_mtls.py`, per the implementation plan's own naming), never inside `basis-core`, per `docs/kernel-boundary-rules.md`.

---

## 13. Producer Admission Ownership

Unchanged from ADR-0008: exact, case-sensitive matching of the derived URI SAN against gateway-owned admission configuration, with no prefix/suffix/substring/wildcard/case-insensitive matching, performed entirely inside `basis-gateway` after §12's derivation step. The trusted ingress performs no admission decision of any kind — it has no concept of "admitted producer," only "certificate passed TLS-layer validation."

**Encoding decision, stated once:** URL-escaped PEM (nginx's `$ssl_client_escaped_cert` format, RFC 3986 percent-encoding of a standard PEM block) is selected because it preserves the exact leaf certificate bytes unambiguously, is produced natively by the selected proxy with no conversion step, fits safely in one bounded HTTP header without embedding literal newlines (PEM's line breaks are percent-encoded, avoiding the multiline-header handling that `nginx` itself has historically had defects around for the unescaped `$ssl_client_cert` variable — the reason `$ssl_client_escaped_cert` is the currently documented, non-deprecated variable), and is reversed on the gateway side by a single URL-decode followed by standard PEM parsing before handing the resulting DER bytes to `cryptography`'s X.509 parser. No custom certificate serialization format is invented.

---

## 14. Authorization-Subject Authentication

The proxy does not parse, interpret, or derive anything from the `Authorization` header. NGINX forwards it unchanged (nginx's default `proxy_pass` behavior already forwards client request headers to the upstream unless a directive explicitly overrides one; no directive touches `Authorization` in this configuration). `basis-gateway`'s existing bearer-authentication dispatch (`AUTH_MODE`-selected verifier, unchanged) continues to establish the authorization subject exactly as it does today, entirely independent of the mTLS producer path. The bounded reference slice's dual-identity requirement — producer URI SAN and bearer `subject_id` are deliberately different test identities, and neither derives from the other — is unaffected by this document; this document only changes how the producer half of that pair reaches `basis-gateway`.

---

## 15. Configuration Ownership

| Concern | Owner |
| - | - |
| Producer client CA / trust anchor | Trusted ingress (NGINX) configuration — `ssl_client_certificate` |
| TLS handshake requirements (`ssl_verify_client on`, protocol/cipher policy) | Trusted ingress configuration |
| Reserved internal header name/encoding, sanitization directive | Trusted ingress configuration (`proxy_set_header`) — the *contract* is defined here (§11), enforced by the ingress |
| Unix-socket path, ownership, and mode | Deployment/process-supervision configuration, coordinated between ingress and gateway startup (§9) |
| Producer admission (admitted URI SAN set) | `basis-gateway` configuration, unchanged from the implementation plan's §12 shape |
| Trusted-proxy producer-mTLS mode enabled/disabled | `basis-gateway` configuration — a new boolean, analogous in spirit to `OPERATION_AWARE_ENABLED`'s default-off discipline; exact environment-variable naming remains a Phase 1B implementation detail, not fixed here |
| Gateway's own server-side TLS material | Not applicable in this topology for the producer-mTLS listener — the gateway does not terminate producer TLS at all; NGINX's own server certificate (for the NGINX↔producer TLS session) is ingress/deployment configuration |
| Authorization-subject bearer configuration (`AUTH_MODE`, OIDC/`basis_local_token` settings) | `basis-gateway` configuration, unchanged |

The bounded implementation plan's existing §11/§12 language, written before Phase 1A's outcome was known, describes gateway-owned TLS material "if Option A is selected." That conditional resolves to: the producer client-CA/trust-anchor material moves to the trusted ingress under this topology; `basis-gateway` owns no producer TLS material at all in trusted-proxy mode (§28 records the exact plan-document correction).

---

## 16. Trust-Fact Matrix

| Fact | Owner | Trust basis |
| - | - | - |
| Producer possesses the client private key | Trusted mTLS ingress (NGINX) | Successful TLS handshake |
| Producer certificate chains to the configured trust anchor | Trusted mTLS ingress | TLS certificate-chain validation (`ssl_client_certificate`) |
| Certificate is valid for the current time | Trusted mTLS ingress | TLS validity-period validation |
| Forwarded certificate corresponds to the TLS peer | Trusted mTLS ingress | `$ssl_client_escaped_cert` is read directly from the validated connection, not from any request-supplied value |
| Forwarded certificate was not supplied by an external HTTP caller | Trusted mTLS ingress | `proxy_set_header` unconditional overwrite (§11) |
| Certificate assertion reached the gateway through a trusted backend channel | Ingress/deployment boundary | Unix domain socket, restrictive filesystem permissions (§9) |
| Producer URI SAN | `basis-gateway` | Parsed from the forwarded, authenticated leaf certificate (§12) |
| Exactly-one-SAN rule | `basis-gateway` | ADR-0008 |
| Producer admission | `basis-gateway` | Exact configured URI SAN match (ADR-0008, §13) |
| Authorization subject | `basis-gateway`'s existing bearer authentication | `basis_local_token` / OIDC, unchanged (§14) |
| Authorization decision | `basis-core` | Deterministic policy evaluation, unchanged |

---

## 17. Processing Sequence

```text
 1. producer opens a TLS connection to the trusted ingress (NGINX)
 2. ingress requires a client certificate
 3. ingress validates chain, validity, and the configured producer CA
 4. TLS failure stops the connection before the gateway is reached
 5. ingress reads the authenticated leaf peer certificate from the
    validated connection
 6. ingress overwrites any inbound X-BASIS-Producer-Client-Cert
    (proxy_set_header executes unconditionally — nothing needs to be
    separately "removed" first, §11)
 7. ingress sets X-BASIS-Producer-Client-Cert to the URL-escaped PEM
    encoding of that leaf certificate
 8. ingress forwards the request over the protected Unix-socket
    backend channel
 9. the Authorization bearer header is forwarded unchanged
10. gateway receives the request from the socket; no other path
    reaches it in this mode
11. gateway validates the internal assertion's shape/encoding (no
    duplicate header, decodable, parseable X.509)
12. gateway parses the X.509 leaf certificate
13. gateway requires exactly one eligible URI SAN
14. gateway treats the URI SAN verbatim, per ADR-0008
15. gateway performs exact producer admission
16. gateway independently authenticates the bearer authorization
    subject via its existing, unchanged dispatch
17. gateway validates/composes the operation-aware request
18. basis-core evaluates
19. gateway enforces the disposition and records GatewayAuditEvent
20. STOP — no protocol execution occurs in this slice
```

The proxy performs no step at or after step 10. It never determines admission, never authenticates the bearer subject, never composes a request, and never calls `basis-core`.

---

## 18. Failure Semantics

**Layer 1 — Producer TLS authentication at the ingress.** Missing client certificate; untrusted CA; expired/not-yet-valid certificate; TLS negotiation failure. Required: the connection is rejected by NGINX before any HTTP request is forwarded; `basis-gateway` is never reached; no `OperationProducerTrust` object exists for the attempt, because no application-level request occurred.

**Layer 2 — Proxy/backend trust boundary.** NGINX cannot reach the Unix socket; the socket is unavailable; the internal certificate header is missing, duplicated, or malformed on a request that otherwise reached the gateway process. Required: fail closed; no kernel evaluation; no producer trust established. A socket-connection failure surfaces to the producer as an ordinary upstream/gateway-unavailable error at the ingress, distinguishable in ingress logs from a TLS-layer rejection.

**Layer 3 — Gateway certificate identity derivation.** Zero eligible URI SANs; multiple eligible URI SANs; a certificate that fails to parse. Required: fail closed; no producer admission; unchanged from ADR-0008's existing fail-closed list.

**Layer 4 — Producer admission.** The derived URI SAN is not present in, or is disabled in, gateway admission configuration. Required: the connection may proceed only as an ordinary (unadmitted) caller; any producer-only field it asserts is rejected by the existing `UntrustedOperationProducerContextError` path.

**Layer 5 — Subject authentication.** Missing, malformed, expired, or invalid bearer token. Required: no authorization evaluation occurs, independent of whether producer authentication and admission succeeded.

**Layer 6 — Authorization.** `ALLOW`/`DENY`/`NOT_APPLICABLE`/evaluation failure, exactly as today. Execution remains absent regardless of outcome.

This restates, without narrowing, the implementation plan's own §17 layered taxonomy; the only addition this document makes is Layer 2, the proxy/backend trust boundary, which did not exist as a distinct layer before a proxy was introduced into the topology.

---

## 19. Threat Analysis

Twenty items reviewed; each states mitigation, residual risk, and later work where applicable.

1. **Caller header spoofing.** Mitigated: `proxy_set_header` unconditionally overwrites the reserved header on every forwarded request (§11); demonstrated necessary by the Phase 1A spike's own direct observation of unmodified header pass-through when nothing overwrites it. Residual: none new, given the ingress functions as configured.
2. **Direct gateway bypass.** Mitigated: `basis-gateway` has no TCP listener in trusted-proxy producer-mTLS mode; only the Unix socket exists, and only processes with filesystem access to it can connect (§9). Residual: same-host process access to the socket (§18 below).
3. **Same-host Unix-socket access.** Partially mitigated by restrictive socket ownership/mode (§9). Residual: a malicious process running as the same user/group as the ingress could connect to the socket and present arbitrary headers, including a forged internal certificate header — Unix sockets provide no process-identity attestation beyond filesystem permission. This is a structural limit of the selected mechanism, not a defect in this document's configuration of it; it is treated as part of this profile's trusted computing base (§9, §26 records it as a deferred stronger option).
4. **Proxy compromise.** Not mitigated by mTLS or by gateway admission alone: a compromised NGINX process can forge the internal header for any URI SAN it chooses, including one that is already admitted. Mitigation available: minimal NGINX configuration (one route, no unrelated modules), restricted process privileges, immutable/reference configuration, filesystem permissions bounding what the compromised process can otherwise reach. Residual: ingress integrity is part of this profile's trusted computing base; no purely gateway-side check can fully mitigate a compromised component that sits ahead of every check the gateway performs.
5. **Proxy misconfiguration.** Mitigated in part by this document fixing the exact directive (`proxy_set_header ... $ssl_client_escaped_cert;`, `ssl_verify_client on;`) rather than leaving it to ad hoc implementation choice; Phase 1B's test suite (Appendix B) must exercise the configuration directly, not merely assert it is correct. Residual: a future deployment that hand-edits the configuration incorrectly (for example, omitting the `proxy_set_header` overwrite) reintroduces header spoofing; this is an operational/configuration-review risk, not one this document can eliminate by specification alone.
6. **Broad producer CA.** Mitigated the same way ADR-0008 already documents: a broad trust anchor does not by itself admit anything, because admission is a separate, exact-match gateway check (§13); mitigated only partially, since authentication strength is still degraded if an operator configures an over-broad CA at the ingress. Belongs to operator configuration discipline, unchanged from ADR-0008's own treatment of this scenario.
7. **Stale producer admission.** Unaffected by this document; remains gateway admission-configuration hygiene, per ADR-0008.
8. **Malformed forwarded certificate.** Mitigated: Layer 2/3 fail-closed handling (§18) rejects an assertion that does not decode to a well-formed, parseable X.509 certificate.
9. **Duplicate forwarded headers.** Mitigated: `basis-gateway` treats more than one occurrence of the reserved header as malformed and fails closed (§11, §18) rather than silently choosing the first or last occurrence.
10. **Oversized certificate/header values.** Mitigated by explicit size bounds at both the ingress and the gateway (§11); prevents a header-size resource-exhaustion vector.
11. **Certificate with zero eligible URI SANs.** Mitigated: unchanged ADR-0008 fail-closed rule, applied to the certificate as forwarded by this topology (§12, §18).
12. **Certificate with multiple eligible URI SANs.** Same as above.
13. **Common-Name fallback attempts.** Mitigated by construction: the gateway's derivation logic never consults Common Name, regardless of what the ingress forwards; this document does not add a code path that could accidentally do so.
14. **Valid producer + invalid subject.** Mitigated: Layer 5 fails independently of Layers 1–4 succeeding (§18); unchanged from the implementation plan.
15. **Invalid producer + valid subject.** Mitigated: the request either never reaches the gateway (Layer 1 TLS rejection) or proceeds only as an ordinary, unadmitted caller whose producer-only context is rejected (Layer 4); unchanged from the implementation plan.
16. **Replayed HTTP requests.** Not addressed by this document; unchanged from ADR-0008's own explicit deferral of ecosystem-wide replay/idempotency semantics. mTLS's session-bound handshake mitigates simple cross-connection replay of the handshake itself but not replay of a captured, valid request within a session.
17. **Proxy-to-gateway availability failure.** Mitigated by fail-closed Layer 2 behavior (§18): a socket-connection failure produces no evaluation, never a silent allow.
18. **Accidental raw-certificate logging.** Mitigated: `basis-gateway` logs only the derived URI SAN and admission result, never the raw header value (§11, §20).
19. **Bearer-token leakage through proxy logging.** Mitigated by not enabling default NGINX access-log formats that capture the full `Authorization` header value for this listener; this is a standard, narrow logging-configuration requirement for the reference deployment, not new architecture.
20. **Air-gapped certificate lifecycle.** Unaffected by this document beyond moving trust-anchor configuration to the ingress (§15); the ingress validates against a locally configured CA bundle with no external connectivity requirement, consistent with §21.

---

## 20. Audit and Logging

`basis-gateway`'s existing audit and diagnostic behavior is unchanged in kind by this document (unchanged `GatewayAuditEvent`/`AuditEvidence` ownership, per the implementation plan's own §19 finding). The only addition this document motivates is that the gateway's internal diagnostic context can now record which trust-boundary topology produced a given producer-mTLS classification (trusted-proxy mode vs. a hypothetical future direct-termination mode), if a future implementation finds that distinction useful for operational diagnosis — this document does not require it and does not add a new field to any published contract. The raw forwarded certificate header, private keys, bearer tokens, and complete `Authorization` header values are never logged, per §11 and §19 item 18/19 above.

---

## 21. Air-Gapped Operation

The selected topology requires no Internet access at runtime: NGINX validates producer certificates against a locally configured CA bundle (`ssl_client_certificate`), performs no OCSP lookup by default (OCSP stapling/validation is not enabled by this document's configuration), and the Unix-socket backend channel is inherently local. No cloud load balancer, no Kubernetes, no service mesh, and no dynamic proxy control plane (xDS or otherwise) is required — the entire configuration is one static `nginx.conf` file. Certificate issuance remains operator/local-PKI scope, unchanged from ADR-0008.

---

## 22. Local/CI Test Strategy

NGINX is available as a standard, pinned package in GitHub Actions' Ubuntu runner images and via the official `nginx` package in every mainstream Linux package manager, requiring no build step and no container image beyond what CI environments already provide; it is equally installable in a Codespace/devcontainer through the same package manager. The recommended mechanism is installing the distribution-provided `nginx` package (already includes `ngx_http_ssl_module` in the standard prebuilt binaries used by Debian/Ubuntu and most other distributions) as a CI/test-fixture dependency of `basis-gateway`'s integration test suite, configured with a test-local `nginx.conf` generated or checked in alongside the test harness, pointed at Phase 1A's existing certificate-fixture generator (`tests/integration/mtls_certs.py`) and a test-local Unix-socket path. This avoids a strategy that would silently require public network access during routine test runs; nothing about installing a pinned distribution package or running a locally configured NGINX process against local certificate fixtures reaches the network. A pinned container image is an acceptable alternative if a future implementation phase finds the distribution-package approach insufficiently deterministic across CI runner image updates; this document does not require one for the bounded slice.

---

## 23. Phase 1B Implementation Requirements

After ADR-0009 is formally accepted, Phase 1B may implement, in the sequence the bounded implementation plan's own §26 already establishes (with this document resolving what that section left open):

1. trusted-ingress mode configuration (§15) — a new gateway boolean, default off;
2. protected internal certificate-header parsing (§11) — strict shape/encoding validation;
3. strict missing/duplicate/malformed header rejection (§11, §18 Layer 2/3);
4. X.509 parsing of the forwarded leaf certificate (§12), reusing the already-released `cryptography` dependency;
5. exactly-one URI SAN extraction (§12), unchanged from ADR-0008;
6. exact URI SAN admission (§13), unchanged from ADR-0008;
7. additive `OperationProducerTrust` fields/source recording which mechanism established trust, per the implementation plan's existing §14.7;
8. independent `basis_local_token`/OIDC subject authentication, unchanged (§14);
9. `OPERATION_PRODUCER_SUBJECT_IDS` coexistence, unchanged (ADR-0008, implementation plan §13);
10. audit/diagnostic enrichment (§20), internal only;
11. tests using the Phase 1A PKI harness plus a locally installed NGINX instance (§22, Appendix B);
12. no producer runtime is implemented by Phase 1B — this document does not change the implementation plan's own phase boundaries (Phase 2 onward remains the reference-producer repository's scope);
13. no evidence persistence in Phase 1B;
14. no execution, at any phase of the bounded slice.

Phase 1B is not complete merely because NGINX configuration exists and certificate-parsing code runs — the same completion standard the implementation plan's §11 already states applies unchanged: a validated producer certificate must be mapped to exactly one URI SAN through a trust boundary demonstrated, by test, not to depend on caller-controlled request data, and this document additionally requires that same test to demonstrate the header-overwrite behavior (§11, Appendix B) directly, not merely assume the proxy configuration is correct.

---

## 24. Shared Contract Assessment

No `basis-schemas` change is required. TLS/client-certificate material remains transport security, outside any published contract's scope. The internal `X-BASIS-Producer-Client-Cert` header is an implementation-local transport contract between the trusted ingress and `basis-gateway`, not a cross-repository or cross-version contract — no second independently versioned component needs to agree on its shape, because nothing outside `basis-gateway`'s own configuration and code ever reads or writes it except the ingress that this document itself specifies. Producer URI SAN derivation remains entirely gateway-owned, unchanged from ADR-0008. The existing operation-aware request contract is unchanged; this document adds no field to any request body, and specifically does not add a caller-supplied producer-identity field to the public request contract — doing so would reintroduce exactly the caller-controlled identity assertion ADR-0008 already forbids.

---

## 25. Compatibility

This document is additive with respect to every currently released contract and runtime behavior, in the same sense ADR-0008 states for itself: it modifies no released `basis-gateway`, `basis-core`, `basis-adapters`, `basis-identity`, or `basis-schemas` behavior. A deployment that takes no action after this document and ADR-0009 are accepted observes no behavior change. It narrows one previously open implementation question (the mTLS termination topology) that the bounded implementation plan explicitly left as a blocking, undecided gate; it does not reopen or weaken any decision ADR-0008 made.

---

## 26. Reference-vs-Production Scope

| Property | Bounded reference decision | Permanent production status |
| - | - | - |
| Selected proxy (NGINX) | Chosen for the first reference slice | Not mandated for every BASIS deployment |
| Unix-socket backend | Chosen for the first reference slice | Future deployment decision; independently authenticated backend mTLS (§9) remains an evaluated, deferred alternative |
| Internal certificate header (`X-BASIS-Producer-Client-Cert`) | Reference implementation contract | May evolve with future ingress architecture; not published |
| Single dedicated mTLS listener, one route | Reference scope | Not a universal ingress-topology requirement; broader coexistence with other gateway routes and callers is explicitly out of scope here |
| Local/private CA | Supported, required for the reference slice | PKI architecture remains deployment-owned, unchanged from ADR-0008 |
| High-availability ingress topology | Not addressed | Deferred; `ROADMAP.md` Phase 4 territory |
| Ingress clustering/redundancy | Not addressed | Deferred |
| Certificate lifecycle automation | Not addressed | Deferred; `ROADMAP.md` Phase 4 research direction, unchanged |

This document's decisions are scoped to the first bounded producer reference slice. They do not make NGINX, Unix-socket backends, or this document's exact header contract a permanent requirement for every future BASIS gateway deployment.

---

## 27. Deferred Decisions

- Permanent ingress-technology selection for production BASIS deployments beyond the bounded reference slice.
- Whether a future, larger ingress topology (multiple routes, additional callers, mesh integration) should revisit the Envoy evaluation in §6.
- Independently authenticated backend mTLS between the ingress and `basis-gateway` (§9), evaluated and deferred, not selected.
- High-availability and clustering topology for the trusted ingress.
- Certificate lifecycle automation for either producer or ingress-facing certificates (`ROADMAP.md` Phase 4, unchanged).
- Permanent repository/packaging ownership of the reference ingress configuration (reference/demo/deployment configuration vs. a future `basis-deploy`; no `basis-deploy` repository exists and none is created by this document).
- Exact `basis-gateway` environment-variable names for trusted-proxy mode configuration (§15), left to Phase 1B implementation.
- Whether a future implementation phase should prefer HAProxy's DER/base64 encoding over URL-escaped PEM if a concrete deployment reason emerges (§6, §13).

---

## 28. Completion Criteria

This document, together with ADR-0009, is complete when:

- the trusted mTLS ingress boundary is specified precisely enough that Phase 1B can proceed without inventing proxy trust semantics, certificate-forwarding semantics, header trust, backend channel protection, URI SAN ownership, failure behavior, or deployment assumptions (§7–§18);
- the bounded implementation plan's own stale Option A/B uncertainty and any stale gateway-owned-TLS-material assumption are corrected (§15, and the implementation-plan update this PR makes);
- ADR-0008 is preserved unchanged and this document's relationship to it is stated explicitly (§3);
- no `basis-schemas` change is proposed (§24);
- the ADR remains `Proposed`, not `Accepted` (formal acceptance is a separate governance action).

---

## Appendix A — Primary-Source Technology Evidence

| Claim | Source |
| - | - |
| `ssl_verify_client`, `ssl_client_certificate`, `$ssl_client_escaped_cert`, `$ssl_client_verify` | `ngx_http_ssl_module` — https://nginx.org/en/docs/http/ngx_http_ssl_module.html (current mainline documentation, fetched directly) |
| `server unix:PATH;` upstream syntax, mixed TCP/Unix-socket upstream groups | `ngx_http_upstream_module` — https://nginx.org/en/docs/http/ngx_http_upstream_module.html (current mainline documentation, fetched directly) |
| `bind ... ca-file ... verify required`, `ssl_c_verify`, `ssl_c_s_dn`, `ssl_c_der`, `unix@` server address | HAProxy configuration documentation (cbonte.github.io HAProxy configuration manual; haproxy.com documentation) |
| XFCC `forward_client_cert_details`, `Cert=` (URL-encoded PEM), `Chain=`, `SANITIZE_SET` | Envoy HTTP header manipulation documentation — https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers.html (fetched directly) |
| `Pipe` address type for listeners and clusters | Envoy `config.core.v3.Pipe` — https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/address.proto |
| `verifyPeer`/`verifyChain`/`CAfile`, Unix-socket address parameter, `protocol` option's fixed application-protocol set (no generic HTTP header manipulation) | stunnel manual page — https://www.stunnel.org/manual.html (fetched directly) |
| Uvicorn `--uds` Unix-domain-socket binding, documented for use behind a reverse proxy | Uvicorn deployment/settings documentation (uvicorn.org) |
| Phase 1A spike findings (Outcome B, ASGI scope contents, header-spoofing observation) | `basis-gateway` `docs/spikes/producer-mtls-certificate-exposure.md`, merged to `main` at `07586110b847d2c8fe5dfadd7a3eba027bc74024` |

---

## Appendix B — Phase 1B Test Matrix

**TLS rejection:** no client certificate → gateway never reached; untrusted certificate → gateway never reached; expired certificate → gateway never reached — each provable by asserting the Unix socket records no connection attempt, not merely that the HTTP client received an error.

**Header spoof resistance:** an already-mTLS-authenticated test client sets an attacker-chosen `X-BASIS-Producer-Client-Cert` value → the value the gateway actually parses must be the ingress-derived one, never the attacker-supplied one. This is the direct regression test for the Phase 1A spike's own spoofing observation (§2) and must be run against the real NGINX configuration, not a mock.

**Certificate identity:** one URI SAN → gateway derives it; zero URI SANs → gateway rejects; multiple URI SANs → gateway rejects; Common-Name-only certificate → gateway rejects, no fallback.

**Admission:** valid certificate + admitted URI SAN → producer admitted; valid certificate + unadmitted URI SAN → producer rejected; case-mismatched URI SAN → rejected; prefix/suffix/wildcard-shaped configuration values → rejected or treated as invalid configuration at startup.

**Dual identity:** valid producer + valid distinct bearer subject → evaluation proceeds, both identities separately visible; valid producer + invalid bearer subject → no evaluation; invalid producer + valid bearer subject → producer-owned context rejected, no trusted-producer evaluation.

**Backend bypass:** no externally reachable gateway TCP listener exists in trusted-proxy mode; a direct connection attempt to the gateway process bypassing the Unix socket must be structurally impossible (no listening TCP socket to connect to), not merely rejected by application logic.

**Proxy/backend failure:** NGINX configured with an unreachable or missing Unix-socket path → no evaluation occurs, and the failure is distinguishable in ingress logs from a Layer 1 TLS rejection.

**Malformed/duplicate assertion:** a request crafted to reach the gateway with a malformed or duplicated internal header (constructed via a direct socket-level test harness that bypasses the ingress, simulating what a same-host process with socket access could send) → fails closed, per §18 Layer 2/3.

**No execution:** an `ALLOW` disposition still ends at the authorization disposition; no execution code exists or runs anywhere in the test path.
