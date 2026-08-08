# Operator and Training Experience Doctrine and Roadmap

## Purpose and Scope

This document defines the long-term human interaction doctrine for the BASIS ecosystem: the intended relationship between Operator mode and Training mode in `basis-console`, the runtime and evidence invariants both must preserve, the dimensions along which they may diverge, and the principles that should guide future console, CLI, and investigation-workflow work. It combines durable interaction doctrine with a deferred, staged maturity roadmap, implementation prerequisites, and decision gates — it is a roadmap document, not a component architecture document.

Concretely, this document establishes:

- the intended human interaction model for BASIS — how a trained operator and a trainee experience the same underlying system;
- the long-term distinction between Operator mode and Training mode, and the doctrine that governs how each should evolve;
- the runtime and evidence invariants both modes must continue to share, regardless of how their interfaces diverge;
- where Operator and Training experiences may intentionally differ, and where they must not;
- principles for future UI, CLI, playbook, and investigation-workflow design;
- a staged, non-committal path from the current console to a mature operational platform.

This document does not implement a redesign of `basis-console`. It does not require immediate console work. It does not define new authorization semantics, and it does not introduce new contracts between `basis-console` and `basis-gateway`, `basis-core`, `basis-adapters`, or `basis-identity`. It does not imply that any capability described here as a future direction currently exists, and inclusion here is not a release commitment or an immediate implementation plan — consistent with how every other document under [`docs/roadmaps/`](.) is treated in this repository. Where this document uses illustrative field lists, example screens, or example commands, those examples are directional — they establish the shape of a future design space, not a committed interface.

[`docs/architecture/basis-console.md`](../architecture/basis-console.md) defines the console's current component boundaries and responsibilities — what the console is and is not responsible for as a component in the BASIS Core Services Distribution. This roadmap records the deferred evolution of the Operator and Training experiences within those boundaries: it defines the interaction doctrine for the two human-facing experiences the console (and, eventually, a companion CLI) presents on top of that responsibility set. `basis-console.md`'s component boundaries and design invariants govern; this document's interaction doctrine is constrained by them, not a peer to them.

---

## Current-State Foundation

`basis-console`, as it exists today, is a functional and semantically correct reference implementation. It has established, through its implementation phases, that a console can be built which:

- interacts with the authorization system exclusively through `basis-gateway` — it holds no dependency on `basis-core` and enforces that boundary mechanically in its own test suite;
- submits operation-aware requests and renders the typed result the gateway returns, alongside the legacy `POST /v1/evaluate` path, without collapsing the two;
- presents Operator mode and Training mode as two presentation layers over one identical application — the same routes, the same forms, the same gateway calls, the same rendered result — where Training mode is permitted only to add explanatory content, never to alter behavior;
- handles `ALLOW`, `DENY`, `NOT_APPLICABLE`, governed evaluation failure, and generic/transport failure as distinct, correctly labeled outcomes, rather than collapsing them into a binary allowed/blocked signal;
- separates what the console knows because the gateway said so from what the console is showing as sample or illustrative data, and states that distinction honestly on every page where it matters;
- redacts diagnostic and evidence data consistently with the gateway's own redaction behavior, rather than inventing a separate redaction policy;
- teaches the same runtime path it lets an operator use — Training mode's explanations are laid over live or gateway-shaped data, not a parallel simulated backend.

These are real architectural achievements, not incidental features. A console that got the gateway-first boundary wrong, or that let Training mode silently diverge from Operator mode, or that blurred `NOT_APPLICABLE` into `DENY` for display convenience, would have created technical debt that this document's doctrine would then have had to unwind. None of that happened. The current implementation — released as `basis-console` `v0.2.0`, per [`ROADMAP.md`](../../ROADMAP.md) — is the correct functional foundation for everything described below, and is complete for its approved scope.

What the current console is not, yet, is the final professional Operator experience BASIS eventually needs, or the fully developed guided Training experience described in this document. It is an honest, semantically correct, boundary-respecting reference implementation — a platform foundation upon which a denser, faster, more investigation-oriented Operator surface and a richer, playbook-driven Training surface can be built incrementally. Describing today's console as a limitation to be escaped would misstate the relationship: today's console is the substrate the future experience extends, not a mistake it corrects.

---

## Shared Runtime Invariants

Operator mode and Training mode are presentation layers over one authorization runtime. This section is non-negotiable: it does not change as the console matures, and no future design proposal may weaken it without amending this document through the same review process that governs any other architectural change in this repository.

Both modes must continue to share the same:

- authentication boundary;
- gateway endpoint selection;
- request contract and request validation;
- gateway client method and invocation;
- response parsing;
- authorization semantics and kernel outcome;
- gateway disposition;
- returned evidence;
- correlation handling;
- redaction behavior;
- page-level safety behavior for consequential actions;
- audit and trace references, where available.

Mode must never determine:

- whether an operation is authorized;
- which gateway endpoint is called;
- which policy bundle is evaluated;
- whether evidence is trusted;
- whether redaction applies;
- how `NOT_APPLICABLE` is interpreted;
- whether a failure is treated as a policy denial;
- what data the gateway returns.

The architectural model is:

```text
User
  ↓
Shared BASIS request and evidence model
  ↓
basis-gateway
  ↓
basis-core
  ↓
Authoritative result
  ├── Operator experience
  └── Training experience
```

Everything down through the authoritative result is a `basis-gateway` contract, a `basis-core` outcome, or evidence the gateway or kernel produced. Only the branches beneath it are console-owned presentation. The two modes may look, navigate, and explain themselves completely differently over time. They must never disagree about what happened.

---

## Operator Mode Doctrine

Operator mode is a professional OT security operations environment, not a simplified default view that Training mode annotates. It should be characterized as serious, dense, precise, fast, keyboard-friendly, evidence-centered, context-preserving, predictable, resilient during degraded or abnormal conditions, and designed for repeat use by trained professionals.

Operator mode should optimize for: speed to relevant evidence; minimal navigation overhead; rapid correlation across identity, resource, and policy context; low-friction investigation; high information density; efficient repeated actions; deterministic workflows; visibility during partial failure; direct access to both raw and structured information; and minimal explanatory prose.

Operator mode should not optimize primarily for: first-use friendliness; decorative presentation; large introductory cards; repeated terminology explanations; tutorial callouts; marketing-oriented language; or simplified summaries that conceal operational detail. These are legitimate goals — for Training mode.

---

## Mastery Without Deliberate Obscurity

This distinction is the central doctrine of this document and must be applied explicitly to every future Operator-mode design decision:

> Operator mode should be hard to master because it exposes real depth and capability, not because it is confusing or inefficient. Training mode should make that mastery achievable.

Operator mode should take time to learn because BASIS itself is deep and capable — because a real investigation touches identity, producer trust, operation composition, policy applicability, evaluation outcome, disposition, evidence provenance, and correlation, and a professional interface reflecting that depth is not a small surface. It should not be difficult because of unclear labels, hidden state, inconsistent commands, arbitrary workflows, unnecessary modal dialogs, poor navigation, unexplained abbreviations, inaccessible controls, missing feedback, or avoidable visual clutter. None of those failure modes teach mastery; they only waste it.

Expertise should produce measurable, felt advantages: faster navigation; better correlation; more accurate diagnosis; fewer unnecessary steps; the ability to use advanced filters and commands; genuine understanding of raw evidence rather than reliance on a summary; confidence during degraded operation; and faster movement between the console and a future CLI. The system should reward knowledge. It must never mistake bad usability for depth, and it must never gate basic functionality behind obscurity merely to create a feeling of exclusivity.

A novice must not be blocked by arbitrary obscurity. An expert must not be slowed by beginner-focused hints, decorative content, repetitive explanations, or unnecessary confirmation steps. Training mode exists precisely so that the first condition is satisfied without compromising the second.

---

## Operator Information Density

The future Operator surface should be an operational workspace, not a conventional dashboard. Conventional dashboards optimize for a glance; an investigation workspace optimizes for sustained, information-dense use by someone who already knows what they are looking at.

The design should prioritize compact visibility of: subject identity or identity reference, when available; operation producer; action; resource; policy bundle; evaluation status; kernel outcome; failure reason; gateway disposition; HTTP/client state; execution state; correlation ID; request ID; trace reference; evidence timestamps; provenance; readiness and dependency state; related operations; and relevant audit evidence.

An illustrative, non-normative example of the density this doctrine implies:

```text
Evaluation       completed
Outcome          not_applicable
Disposition      deny
HTTP             403
Bundle           facility-west@17
Correlation      01J...
Producer         unavailable
Execution        not_attempted
```

This example is directional. It does not assert that every field shown is currently available from the gateway, and it is not a committed layout. It illustrates the doctrine that an operator's screen should read like a dense, labeled fact sheet — not a friendly status card that requires several clicks to expand into the same information.

---

## Persistent Investigation Workspace

A future Operator surface should preserve investigation context as the operator moves through the system, rather than discarding it on every navigation. Potential retained state includes: selected identity; selected producer; selected resource; selected operation; current policy bundle; request ID; correlation ID; trace reference; related audit events; filters; time range; open evidence panels; investigation notes; and current incident or case context.

An operator should not lose this context merely because they move between decision details, identity activity, resource history, policy evidence, gateway diagnostics, adapter status, execution evidence, and audit records — all of which are legitimate, distinct facets of one investigation and should feel like facets of one workspace rather than unrelated pages.

This document intentionally does not prescribe a database, session-storage mechanism, or state-management implementation for this capability. That is an implementation decision for `basis-console` (or a future CLI) to make when this stage of work begins. The requirement recorded here is user-facing only: investigation context should persist across navigation within an investigation.

---

## Responsiveness and Time to Evidence

BASIS should become a serious professional tool that lets an OT security engineer obtain the information needed to investigate and act without waiting for an entire dashboard, abandoning the console for raw tooling, or navigating through unnecessary screens. Performance and responsiveness are product behavior and operational architecture concerns here, not visual polish — they determine whether the future Operator surface is actually used under pressure or quietly bypassed.

The future Operator experience should be judged partly by how quickly it moves an operator from an operational question to a next useful action:

```text
operational question
        ↓
relevant facts
        ↓
trustworthy evidence
        ↓
next useful action
```

The durable doctrine is: reduce the time from an operator's question to trustworthy evidence. This document does not compare BASIS to named commercial products, and nothing below should be read as such a comparison.

**Useful state should appear early.** The console should present meaningful operational state as soon as it is available, rather than waiting for every dependency, panel, or evidence source to complete. A slow secondary source should not prevent the operator from seeing already-available facts such as request identifiers, correlation identifiers, evaluation status, kernel outcome, gateway disposition, dependency readiness, a known failure boundary, timestamps, or evidence already returned. This document does not claim that incremental rendering exists in the console today; it is a future design goal.

**Avoid whole-workspace blocking.** One slow or unavailable component should not unnecessarily block unrelated operator workflows or blank an entire investigation surface. The future UI should be capable of distinguishing loading, available, unavailable, delayed, stale, partial, unknown, and absent — these states must not be collapsed into one generic spinner or error.

**Preserve partial results.** When only part of an investigation is available, the console should retain and display the facts it has, clearly label what is missing, and continue to allow useful navigation. The console must not discard valid evidence merely because a later dependency failed. This aligns with, and does not replace, the existing doctrine in **Degraded and Incident Operation** above.

**Fast navigation matters.** Common transitions should not require the operator to reconstruct context or wait for data that was already retrieved. The future design should minimize unnecessary full-page reloads, repeated retrieval of unchanged information, loss of filters or investigation state, navigation through several pages to reach one related record, and blocking read-only work behind unrelated operations. This principle is implementation-neutral: it does not prescribe a JavaScript framework, cache library, browser architecture, or backend technology.

**Detail should not block the first useful view.** Large raw records, historical timelines, secondary correlations, and lower-priority evidence may arrive or expand after the first useful operational view, provided that the initial view is honest about incomplete data, provenance remains visible, the eventual detail does not silently replace earlier facts with a different interpretation, stale or delayed data is labeled, and the operator can intentionally request deeper evidence. Important safety or authorization facts must never be hidden merely to improve perceived speed.

**Responsiveness under degraded conditions.** The Operator experience should remain navigable and informative when a gateway, adapter, identity source, audit source, or other dependency is degraded. A dependency failure should affect the capabilities that depend on it, not automatically make the entire console unusable.

**Responsiveness does not override safety.** Performance serves trustworthy operation; it does not override correctness, evidence integrity, or safety. Responsiveness must never justify: bypassing `basis-gateway`; locally re-evaluating policy; presenting cached information as current without a timestamp or state label; hiding authorization or safety facts; treating incomplete evidence as complete; weakening redaction; skipping required confirmation for consequential operations; fabricating a kernel outcome; or suppressing a degraded-state warning.

### Future Measurement Concepts

Future implementation should define measurable operator-performance objectives, using concepts such as: time to first useful operational fact; time to trustworthy evidence; time to open a related identity, resource, request, or correlation; search and filter response time; time to recover useful context after a dependency failure; number of navigation steps for high-frequency workflows; and investigation completion time for representative scenarios.

These are future measurement categories, not current benchmarks. This document does not assign milliseconds, seconds, service-level objectives, percentile targets, or release gates — exact targets require representative workloads and real operator testing, which do not yet exist. Perceived speed alone is insufficient: BASIS must be fast without presenting incomplete information as complete or trading away evidence integrity.

---

## Structured and Raw Evidence Together

Operator mode should expose both a structured, normalized interpretation of evidence and the raw, redacted source evidence beneath it. Structured data provides speed; raw data provides fidelity and troubleshooting depth. Neither should silently contradict the other, and provenance — which component produced which fact — must remain visible throughout.

Raw values must remain redacted and safely rendered under the same redaction rules the gateway itself applies; the console must never reinterpret raw evidence into a stronger claim than the evidence supports, and an operator should never need to abandon the console merely to see the real underlying data.

Potential future presentation patterns — all illustrative, none committed — include side-by-side structured and raw views, expandable raw records, highlighted semantic differences between the two, copyable identifiers, related-record navigation, field-ownership labels (which component produced this fact), and source and timestamp visibility for each field.

---

## Keyboard-First and Command-Oriented Interaction

Keyboard efficiency is a future design goal for Operator mode, not a current commitment. Potential capabilities — again illustrative, not specified — include a command palette, global search, keyboard navigation, quick filters, jump-to-request/correlation/trace, opening a related identity or resource directly, copying an identifier, replaying or reconstructing a safe evaluation request, switching between structured and raw views, focusing evidence panels, saved operator layouts, and a list of recent investigations.

This document does not assign specific key bindings and does not claim any of these capabilities exist today. The governing doctrine is narrower and more durable than any specific binding: frequent operators should be able to perform common workflows with minimal mouse movement and minimal loss of investigation context. Specific bindings and command syntax are implementation decisions for a later stage.

---

## UI and CLI Equivalence

BASIS will eventually need a command-line surface for scriptable, operator-driven interaction, and the relationship between that future CLI and `basis-console` must be decided now, in doctrine, so that neither is designed as an afterthought to the other. The CLI must not become "the real tool" while the console remains a simplified facade for people who have not learned the CLI. Both are interfaces over the same authoritative runtime.

Both should consume the same contracts, terminology, authorization semantics, evidence, identifiers, applicable filters, request-ownership rules, and redaction rules. The architectural model is:

```text
Shared operational contracts and evidence
        ├── Operator console
        └── CLI
```

Future goals for this relationship — directional, not committed — may include displaying the CLI equivalent of a console action, opening a console investigation from a CLI-produced correlation ID, exporting a safe CLI command from a console request, copying normalized JSON between surfaces, viewing the same evidence in both interfaces, and preserving investigation references across surfaces.

This document does not define a CLI syntax, does not commit to a CLI compatibility surface, and does not name or imply a specific CLI repository. Any illustrative command shown in future planning documents should be clearly labeled non-normative until a dedicated architecture decision establishes otherwise.

---

## Degraded and Incident Operation

An expert interface should become more valuable, not less, when conditions are abnormal. This is where the current console's honest, non-fabricating behavior matters most, and where the future Operator surface should invest deliberately rather than treating degraded-state handling as an edge case.

The future Operator experience should define clear, evidence-preserving behavior for: gateway unavailability; authentication failure; evaluator unavailability; a malformed or contract-invalid response; policy evaluation failure; adapter unavailability; a case where authorization succeeded but execution failed; partial evidence availability; a correlation mismatch; delayed audit evidence; and inconsistent dependency readiness.

In each of these conditions, Operator mode should: preserve every known fact rather than discarding partial state; distinguish "unknown" from "absent" instead of treating them as the same blank; avoid generic "something went wrong" messaging in favor of identifying which boundary failed; retain identifiers and timestamps even when the evaluation itself could not complete; identify the next useful investigation action available to the operator; avoid hiding raw redacted diagnostics behind a friendlier summary; and never fabricate a policy decision the system did not actually reach.

This section restates, at the interaction-design level, a principle the current console already honors mechanically: mode is never an excuse to mislead, and neither is a bad day for the gateway.

---

## Training Mode Doctrine

Training mode is a guided educational and simulation environment layered over the same BASIS Core Services Distribution Operator mode exposes — not a separate, lower-fidelity product. It should be friendly, explanatory, visual, progressive, scenario-based, safe, honest about the distinction between simulated and live data, and deeply educational rather than superficially helpful.

Training mode should help a trainee learn: ecosystem component boundaries; identity and producer ownership; action and resource composition; policy applicability; the distinction between outcome and disposition; evidence provenance; correlation; failure classification; how to interpret evidence; adapter normalization; execution evidence; and operator investigation technique itself.

Training mode should not merely be Operator mode with paragraphs added around it. As it matures, it may develop a substantially different layout and workflow sequence from Operator mode — optimized for progressive disclosure rather than density — while remaining connected to the same contracts and the same runtime truth the current console already enforces.

---

## Training Playbooks

A future playbook framework should structure guided Training exercises as a defined, repeatable format rather than ad hoc pages. A playbook should contain: a title; an intended skill level; a learning objective; scenario context; a starting state; guided actions; the evidence the trainee must locate; questions the trainee must answer; checkpoints; corrective feedback; an expected conclusion; the Operator-mode-equivalent workflow for the same scenario; a CLI equivalent, once one exists; and completion evidence or a score, if that capability is later supported.

Illustrative future playbook topics — none implemented, none scheduled — include: following an operation from request to disposition; explaining why `NOT_APPLICABLE` differs from `DENY`; investigating a governed evaluation failure; diagnosing a gateway-unavailable state; identifying caller-owned versus gateway-owned fields; identifying trusted-producer evidence; investigating a correlation-ID mismatch; determining why authorization succeeded but execution failed; tracing one machine identity across several operations; investigating suspicious workload or agent activity; comparing structured evidence with a raw event; determining which component owns a given failure; and reconstructing an investigation using request and trace identifiers.

This document does not implement playbooks. It establishes the format a future implementation should use.

---

## Training Progression

A possible maturity progression for a trainee, described directionally rather than as a committed curriculum schedule:

**Orientation** — ecosystem overview, terminology, component responsibilities, navigation.

**Guided fundamentals** — request construction, authentication, producer trust, authorization, evidence.

**Guided investigations** — known failure scenarios, correlation, policy interpretation, degraded states.

**Independent exercises** — minimal hints, real or realistic evidence, the trainee produces conclusions rather than following a script.

**Operator transition** — the same scenario repeated in Operator mode, no instructional overlays, keyboard and CLI equivalence introduced, and performance and accuracy becoming part of what mastery means.

```text
Orientation
  ↓
Guided fundamentals
  ↓
Guided investigations
  ↓
Independent exercises
  ↓
Operator transition
```

The progression's purpose is to make the transition from trainee to operator legible and achievable, not merely to add content to Training mode. A trainee who completes it should be able to perform the equivalent Operator-mode workflow with the instructional scaffolding removed.

---

## Mode Divergence Rules

The modes may differ in: layout; information density; navigation style; explanatory content; use of diagrams; screenshots; progressive disclosure; guided sequencing; visual hierarchy; terminology assistance; playbooks; default panel arrangement; keyboard discoverability; and novice safeguards.

They must not differ in: authorization behavior; endpoint selection; trusted evidence; redaction; request-ownership rules; outcome meaning; disposition meaning; failure classification; security boundaries; permitted operation fields; or audit integrity.

This is the same invariant the current console already enforces mechanically through its mode-parity tests, restated here as durable doctrine so that future work — including work that changes the console's visual design substantially — does not need to rediscover it.

---

## Safety and Confirmation Design

A professional interface may be fast without being reckless. Future work on consequential operations should clearly distinguish evaluate, authorize, and execute as separate steps; distinguish simulation from live action at all times; surface the target resource and the operational consequence of an action before it is taken; require deliberate confirmation where appropriate; support stronger safeguards for higher-impact operations; and preserve evidence of who initiated an operation and the exact request and authorization result that followed.

Confirmation friction should not be added to read-only investigation actions — an operator reviewing evidence should never be interrupted by a dialog meant for someone about to change system state. Training mode must never be permitted to bypass a real safety boundary in the name of a smoother learning experience.

This document does not define specific confirmation dialogs, risk levels, or thresholds. Those are implementation decisions for whichever future phase introduces consequential, execution-capable operator actions.

---

## Accessibility

Expert-facing is not a license for inaccessible. Both modes must support full keyboard access, visible focus state, screen-reader-compatible labels, text labels in addition to color, readable contrast, scalable text, understandable error messages, layouts that remain usable at narrower widths, no information available only on hover, and no security-relevant meaning conveyed only through visual style.

Accessibility applies to both modes without exception. Training mode may add more explanation than Operator mode; it must not use that as license for Operator mode to be less accessible than Training mode.

---

## Anti-Patterns

**Operator-mode anti-patterns:** decorative dashboard cards with little operational value; large unused whitespace; repeated beginner explanations; hiding raw evidence; generic status labels; forcing the operator through multiple pages to complete one investigation; mouse-only workflows; collapsing outcomes into a green/red binary; concealing identifiers; modal-heavy navigation; terminal-only recovery paths; different authorization semantics between the UI and a future CLI; blanking the whole workspace while one panel loads; making every panel wait for the slowest dependency; generic indefinite loading indicators without useful state; discarding partial evidence after a secondary failure; repeatedly fetching information already present in the investigation; losing filters or selected context during navigation; hiding latency or staleness from the operator; optimizing visual animation while delaying useful facts; and requiring terminal use solely because the UI cannot expose evidence quickly enough.

**Training-mode anti-patterns:** static walls of text; fake evidence presented as real; screenshots disconnected from live workflows; tutorials that use different request semantics than the real system uses; excessive guidance that never transitions to independent work; scoring without meaningful skill evidence behind it; simplified terminology that contradicts the canonical vocabulary in [`docs/glossary.md`](../glossary.md); and an alternate training authorization path that runs outside the real gateway boundary.

**Shared anti-patterns:** UI mode becoming an authorization mode; local policy evaluation anywhere in the console; subject or producer inference performed by the console rather than sourced from the gateway; fabricated evidence; silently altered contracts; and presentation that hides a degraded system state instead of surfacing it.

---

## Relationship to Current and Future Repositories

`basis-architecture` owns the interaction doctrine defined in this document: human-facing architectural boundaries, future-state experience principles, terminology, and cross-surface invariants. It does not own or ship any interface itself.

`basis-console` will later implement, incrementally and against this doctrine: the operator workspace, the Training experience, playbooks, keyboard workflows, visual investigation tooling, and shared presentation over gateway-provided truth.

A future CLI repository or package — not yet named, not yet created — would implement command-oriented interaction, scriptable access, operator automation, and terminal-based investigation, governed by the same UI/CLI equivalence doctrine defined above. This document does not name or create that repository.

`basis-gateway` continues to own authentication, request ownership, producer trust, orchestration, enforcement, HTTP classification, and gateway evidence. `basis-core` continues to own deterministic authorization semantics, policy evaluation, kernel outcomes, and governed failure semantics. `basis-adapters` will continue to supply protocol normalization, trusted operational evidence, execution boundaries, and execution results where applicable. Neither the console nor a future CLI replaces any of these responsibilities — they consume and present what those components produce, per [`docs/architecture/basis-console.md`](../architecture/basis-console.md).

---

## Staged Maturity Roadmap

The following stages are a deferred, directional roadmap, not an implementation schedule. No release numbers or dates are assigned, and inclusion here is not a commitment — consistent with how [`ROADMAP.md`](../../ROADMAP.md) treats every other forward-looking item in this repository.

**Stage 0 — Current functional foundation.** The operation-aware console; Operator and Training modes; typed evidence; semantic correctness. Already achieved, released as `basis-console` `v0.2.0`.

**Stage 1 — Operator information architecture.** Investigation-centered navigation; dense evidence layouts; consistent identifiers; removal of unnecessary operator-facing prose; persistent-context design; critical-path information; a first useful operational view; progressive detail; explicit partial and stale states.

**Stage 2 — Operator efficiency and responsiveness.** Keyboard navigation; command palette; search; quick filters; saved views; structured/raw switching; context-preserving navigation; representative responsiveness measurements; time-to-evidence evaluation.

**Stage 3 — Investigation workspace.** Identity, producer, operation, resource, policy, and evidence correlation; persistent investigation state; related-record traversal; degraded-state workflows.

**Stage 4 — Training playbook framework.** Guided scenarios; checkpoints; evidence-finding exercises; operator-transition exercises.

**Stage 5 — UI and CLI equivalence.** Shared request contracts; shared evidence; copy/export command support; cross-linking between terminal and console investigations.

**Stage 6 — Advanced operations.** Execution evidence; identity telemetry; behavioral detections; investigation timelines; bounded response workflows.

Each later stage depends on ecosystem capability this document does not assume exists yet — trusted-producer/adapter alignment, execution-result evidence, and durable identity-to-operation correlation remain future work elsewhere in the ecosystem, per [`ROADMAP.md`](../../ROADMAP.md) and the roadmaps referenced there.

---

## Implementation Entry Criteria

A broad Operator redesign should not begin until the following are substantially in place: real trusted-producer integration; a stable adapter-to-gateway operation contract; execution-result evidence; durable identity-to-operation correlation; enough real operator workflows to design from observed use rather than speculation; clear user stories for investigation and degraded-state response; a stable CLI direction; and agreed evidence and trace contracts.

Smaller, incremental Operator improvements — narrowing information density, removing beginner-oriented prose from Operator mode, adding correlation-ID visibility — may proceed earlier and independently of a broad redesign. A broad redesign undertaken before these prerequisites exist would be designed around guesses rather than real investigation workflows, which is precisely the failure mode this document's doctrine is meant to prevent.

---

## Decision Gates

Future implementation planning must answer, before committing to specific screens or commands: What are the highest-frequency operator tasks? What data must remain visible during navigation? Which actions require confirmation? Which workflows must remain keyboard-only capable? What is the canonical investigation object? How do the UI and CLI share context? Which raw evidence can safely be displayed? Which operator actions are read-only versus consequential? How are playbooks authored, versioned, and validated? How does a trainee graduate into Operator mode? Which capabilities belong in the console versus external SIEM/SOAR tooling? What must exist before execution controls are exposed? What information belongs on the critical path to the first useful view? Which evidence may load progressively? How are partial, stale, delayed, absent, and unknown data distinguished? Which workflows require measurable response-time budgets? Which information should remain locally available during dependency failure? How is investigation state retained without presenting stale data as current? What representative operator workflows will be used for performance testing? At what point does latency make the UI operationally inferior to the CLI?

These gates exist to prevent a premature visual redesign from substituting for the harder architectural questions a mature Operator and Training experience actually depends on.

---

## Success Criteria

**Operator mode succeeds when:** trained operators can reach relevant evidence quickly; difficult incidents do not force operators to abandon the console for raw tooling; raw and structured evidence are both accessible from within it; correlation remains intact across workflows; common tasks require minimal navigation; the UI and a future CLI reinforce rather than duplicate each other; operational state is shown precisely rather than simplified away; experienced operators trust the console during degraded conditions, not only during routine ones; useful operational state appears without waiting for every secondary source; slow dependencies do not unnecessarily block unrelated investigation work; operators can reach trusted evidence with few navigation steps; investigation context remains intact during navigation and partial failure; performance remains predictable enough for repeated professional use; and operators do not leave the console solely because the interface is slower than accessing the same evidence through lower-level tools.

**Training mode succeeds when:** trainees can explain component boundaries in their own words; trainees can distinguish outcome, disposition, and failure without prompting; trainees can locate evidence themselves rather than merely read an explanation of it; playbooks visibly transition trainees from guided to independent work; training uses the same real contracts and semantics Operator mode uses; and a trainee who has completed the progression can perform the equivalent workflow in Operator mode.

---

## Non-Goals

This document does not: redesign the current console; prescribe exact colors, fonts, or branding; define a finalized screen layout; introduce API changes; introduce schema changes; define CLI compatibility syntax; implement playbooks; create training content; introduce execution controls; define identity telemetry; create new repositories; schedule release dates; or claim current support for any future capability it describes.

---

## Terminology

This document uses the following terms consistent with [`docs/glossary.md`](../glossary.md) and [`docs/standards/terminology-rules.md`](../standards/terminology-rules.md): Operator mode, Training mode, operator, trainee, operation, authorization, kernel outcome, gateway disposition, returned evidence, console explanation, future capability, trusted producer, correlation ID, trace reference, execution evidence, and investigation workspace.

"Operator mode" and "Training mode" are added to the glossary alongside this document, since they were previously established only at the implementation level (`basis-console`'s `BASIS_CONSOLE_MODE` presentation modes) and not yet defined as ecosystem-level architectural terms. "Investigation workspace" is used descriptively in this document to name the future Operator surface described in the sections above; it is not introduced as a standalone glossary term at this time, since it does not yet name a concrete architectural component or contract.

The phrase "operations cockpit" may be used descriptively in future informal discussion, but must not become a canonical component name without a separate terminology decision through the process in `terminology-rules.md`. "Simple mode" and "advanced mode" should be avoided: Training mode is not a less-authoritative execution mode, and Operator mode is not a privileged authorization mode. Both are presentation layers over the same authorization runtime, differing only as described in **Mode Divergence Rules** above.

---

## Related Documents

- [`docs/architecture/basis-console.md`](../architecture/basis-console.md) — the canonical console component architecture: responsibilities, non-responsibilities, interaction model, and design invariants this document's doctrine is constrained by
- [`docs/architecture/basis-ecosystem.md`](../architecture/basis-ecosystem.md) — component responsibilities and dependency direction across the BASIS Core Services Distribution
- [`docs/architecture/operation-aware-trace-audit-evidence.md`](../architecture/operation-aware-trace-audit-evidence.md) — the evidence and trace model referenced throughout the Operator- and Training-mode evidence sections above
- [`docs/architecture/operation-aware-evidence-provenance-semantics.md`](../architecture/operation-aware-evidence-provenance-semantics.md) — provenance and explanation projection rules that govern what the console may honestly display
- [`docs/architecture/reference-vs-implementation.md`](../architecture/reference-vs-implementation.md) — the distinction between conceptual architecture, reference architecture, and implementation that this document's staged roadmap depends on
- [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) — the isolation rules that keep `basis-core` free of console or CLI dependencies
- [`docs/standards/terminology-rules.md`](../standards/terminology-rules.md) — the terminology governance process referenced in **Terminology** above
- [`docs/glossary.md`](../glossary.md) — canonical definitions, including the Operator Mode and Training Mode entries added alongside this document
- [`ROADMAP.md`](../../ROADMAP.md) — ecosystem-wide roadmap; see **Current State**, the Downstream Rollout Sequence, and the "Operator and Training Experience Direction" subsection this roadmap is cross-referenced from
