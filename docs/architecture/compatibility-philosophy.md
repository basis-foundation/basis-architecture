# Compatibility Philosophy

This document defines the long-term compatibility philosophy for the BASIS ecosystem. It establishes why certain artifacts in the ecosystem — action names, resource identifiers, audit schemas, policy semantics, and adapter normalization contracts — must be treated as durable commitments rather than as implementation-convenience choices subject to revision.

This is an architectural document, not an operational specification. The policies described here are not yet fully operationalized. They reflect the compatibility commitments the ecosystem is designed to be governed by as components mature and stabilize.

Cross-references: [`docs/architecture/basis-ecosystem.md`](basis-ecosystem.md) describes component responsibilities and dependency direction. [`docs/kernel-boundary-rules.md`](../kernel-boundary-rules.md) defines the stability boundaries of the authorization kernel. [`docs/architecture-principles.md`](../architecture-principles.md) — Principles 3, 5, 9, and 14 — establishes the architectural commitments that compatibility protects.

---

## Why Compatibility Matters in Operational Infrastructure

Authorization infrastructure in OT environments is not software that is updated frequently, deployed uniformly, or tested comprehensively before reaching field conditions. The components that depend on it — enforcement points, protocol adapters, audit consumers, policy authoring systems — are deployed at diverse sites, across heterogeneous infrastructure, and under change management constraints that limit the speed and coordination with which updates can propagate.

A change to an audit schema may be trivial from an implementation perspective. From an operational perspective, it breaks every audit consumer that was built against the previous schema — consumers that may be running on edge systems with months-long maintenance windows, or on security infrastructure that a separate operations team manages and cannot update in coordination with an architecture change.

The asymmetry here is significant: the cost of a compatibility break is paid by operators in the field, while the benefit of the change accrues to the component that made it. That asymmetry argues for treating compatibility as a primary architectural concern rather than as a consideration to be addressed during implementation.

---

## Why Audit Continuity Matters

Audit records have a temporal continuity requirement that most software artifacts do not. A change that would be transparently backward-compatible for a live service — because all calling components are updated simultaneously — is not transparently backward-compatible for a historical audit record.

An audit query that covers a time range spanning a schema change must be able to interpret records from before and after the change in a consistent way. If the resource identifier format changed, or a required audit field was renamed, the records on either side of the change are structurally different. Forensic analysis, compliance audits, and incident investigations routinely span time ranges that include past schema versions.

This creates a requirement that goes beyond forward compatibility: audit schemas must support retrospective interpretation. A new schema version must be interpretable against audit records from prior schema versions, either through defined migration logic or through a stability commitment that makes migration unnecessary.

Architecture Principle 14 — Immutable Audit Logging — establishes that authorization decision records must be written in a form that cannot be modified after the fact. Compatibility is the precondition for that immutability to be analytically useful: an immutable record written in a format no longer interpretable by current tooling is preserved but not auditable.

---

## Why Semantic Stability Matters More Than Implementation Convenience

The artifacts governed by this document are not implementation details. They are the shared vocabulary that allows disparate components, built at different times by different teams, to interoperate and to produce a coherent audit record.

Action names are used in policies and audit records simultaneously. A policy that references `write:hvac:setpoint` and an audit record that captures `write:hvac:setpoint` must refer to the same operation — not by convention, but by contract. Renaming that action, even with documentation of the rename, produces a semantic gap: policies that use the old name will evaluate differently (or not at all) against implementations that emit the new name; audit records from before and after the rename must be correlated with explicit knowledge of the rename event.

The practical consequence is that changing an established action name, resource identifier format, audit field name, or policy evaluation semantic is not an implementation refactor — it is a breaking change to the ecosystem's shared vocabulary, with costs distributed across every component and deployment that depended on the prior form.

Semantic stability is not about freezing the vocabulary permanently. It is about ensuring that changes to shared vocabulary are treated with the weight they deserve: explicit, versioned, and communicated with enough lead time for dependent systems to adapt.

---

## Action Vocabulary Stability

Action names, once established and in use across policies and audit records, are durable contracts. The action vocabulary governance document ([`docs/architecture/action-vocabulary.md`](action-vocabulary.md)) defines the naming structure, conventions, and deprecation process.

From a compatibility perspective:

- An action name that appears in a deployed policy must continue to evaluate correctly for the lifetime of that policy.
- An action name that appears in an audit record must continue to be interpretable for the retention period of that record.
- Renaming an action is a breaking change. Adding an alias that evaluates identically to an existing action is not — it is an additive change.
- Narrowing the scope of an action (so that requests that previously matched no longer match) is a breaking change to policies that relied on the prior scope.
- Broadening the scope of an action (so that requests that previously did not match now match) is a breaking change to policies that relied on the prior scope to produce a deny for those requests.

The stability expectation for action names is long: years to decades in OT contexts, where audit retention requirements and device lifecycles operate on those timescales.

---

## Resource Identifier Stability

Resource identifiers appear in policies, audit records, and enforcement point configurations. The identifier format — the structural convention that allows a controller point on one protocol to be referenced consistently with a controller point on another — must be stable enough that a resource reference authored today remains valid without modification across the operational lifetime of the deployment.

From a compatibility perspective:

- The structural format of resource identifiers is a compatibility surface. Changing the format requires migrating every policy and audit record that contains the prior format.
- Identifier components (type classification, addressing components, qualifier fields) must be additive-extensible. New fields may be added; existing fields must not be removed or semantically redefined.
- Resource identifiers that appear in policies must be resolvable by the policy engine across policy engine version increments. A policy that references a resource in one identifier format must not silently fail to match after a version increment that changed the format.
- Changes to controller object models in the field — new firmware, reconfigured points — may invalidate specific resource identifiers without constituting a format-level compatibility break. These changes are operational events, not architecture changes, and must be managed through adapter configuration updates and policy review rather than through compatibility provisions.

---

## Audit Schema Compatibility

The audit event schema defines the fields, types, and semantic meaning of authorization decision records. It is a compatibility surface shared by: enforcement points that emit events, the audit pipeline that transports them, the audit store that persists them, and every consumer — query interface, compliance reporter, forensic tool — that reads them.

Compatibility expectations for audit schemas:

- Required fields must not be removed. A field that audit consumers depend on cannot be eliminated without breaking those consumers.
- Field names must not be changed. Renaming a field is a breaking change to every consumer that reads it by name.
- Field semantics must not be changed. A field that previously meant "subject identifier as it appeared in the access token" must not be redefined to mean "subject identifier as it appeared in the policy record" without an explicit versioned break.
- New optional fields may be added without a version increment, provided they carry a defined absence semantics (consumers that receive a record without the new field must not fail).
- New required fields constitute a breaking change if they cannot be backfilled for records produced under prior schema versions.
- Schema version identifiers must be present in every audit record so that consumers can select the appropriate interpretation logic for records from different schema versions.

The schema evolution rules in the following section apply directly to audit schema evolution.

---

## Policy Behavior Compatibility

Policy evaluation semantics define what happens when a specific subject, resource, action, and context is presented to the kernel. The compatibility surface is the behavior: a policy that evaluated to `allow` under version N must evaluate to `allow` under version N+1 for the same inputs, unless the policy behavior change was explicitly versioned and communicated.

From a compatibility perspective:

- The evaluation semantics of existing policy constructs must be stable across kernel version increments. Adding a new policy construct is additive. Changing the evaluation behavior of an existing construct is a breaking change.
- Default evaluation outcomes — the behavior when no policy matches a given request — must be stable. Changing the default from deny to allow, or adding conditions under which the default applies differently, is a breaking change to every deployment that relies on default behavior.
- The policy format specification is a compatibility surface. Changes to the policy format that make previously valid policy documents invalid, or that change their evaluation outcome, are breaking changes.
- Policy evaluation must remain deterministic. A version increment that introduces non-deterministic evaluation behavior — where the same inputs can produce different outcomes across evaluations — is a regression, not a version.

---

## Adapter Normalization Compatibility

Protocol adapters translate field-protocol messages into the subject-resource-action representation the kernel evaluates. The normalization mapping — how specific protocol operations are represented as action names and resource identifiers — is itself a compatibility surface.

From a compatibility perspective:

- The normalization mapping for an established protocol operation must be stable. If a BACnet WriteProperty operation on a specific object type has been normalized to `write:hvac:setpoint`, that normalization must remain consistent across adapter versions, unless an explicit versioned break is documented.
- Changes to normalization mapping produce policy and audit inconsistency without any visible error. A policy written against one normalization will silently evaluate differently — or not at all — if the underlying normalization changes. There is no runtime signal that the normalization on which the policy was written no longer matches.
- Adapter versions must be recorded in audit events so that normalization-level changes are traceable in the audit record. An audit event produced by adapter version 1.2 is not directly comparable to an audit event produced by adapter version 2.0 if the normalization mapping changed between those versions.
- Device configuration changes that affect protocol semantics — new firmware, reconfigured object models, expanded register maps — must trigger adapter normalization review. A device that changes its semantics without an adapter update creates a normalization drift that compromises the correctness of authorization decisions downstream.

---

## Schema Evolution Philosophy

Schemas that are compatibility surfaces — audit event schemas, authorization request and response schemas, policy format schemas — should evolve according to the following principles:

**Additive changes are preferred.** Adding new optional fields, new enumeration values with defined semantics, and new optional sections does not break existing consumers. Additive changes may be deployed without a major version increment, provided the addition is described in a schema changelog.

**Semantic redefinition is a breaking change.** Changing the meaning of an existing field, narrowing or broadening the valid value range of an existing field, or redefining what absence of a field means are all breaking changes, regardless of whether the field's name or type changes.

**Removal is a breaking change.** Removing a field, a required section, or an enumeration value is always a breaking change.

**Breaking changes require major version increments.** When a breaking change is necessary, it must be introduced under a new major schema version. The prior schema version must be supported in parallel for a defined transition period, during which existing deployments can migrate.

**Migration paths must be defined before breaking changes are deployed.** A breaking schema change without a defined migration path from the prior version to the new version is not deployable in an ecosystem where components update at different speeds.

---

## Deprecation Philosophy

Deprecation is the process by which a compatibility surface that will be changed or removed is signaled to its consumers in advance of the change.

**Deprecation is not removal.** A deprecated action name, schema field, or policy construct continues to function during the deprecation period. Removal occurs after the deprecation period has ended and dependent components have had the opportunity to migrate.

**Deprecation notices must be explicit.** A compatibility surface is deprecated when a deprecation is formally recorded in the schema changelog, the action vocabulary log, or the relevant ADR — not when a comment is added to implementation code or when a verbal notice is given in a meeting.

**Deprecation periods must be long enough to be operationally meaningful.** In OT deployments with constrained maintenance windows and slow update cycles, a deprecation period measured in weeks is not meaningful. The appropriate deprecation period depends on the pace at which dependent deployments can realistically migrate, which in most OT contexts is measured in months or quarters.

**Deprecation does not guarantee continuation.** A deprecated surface will eventually be removed. Deployments that depend on deprecated surfaces should treat the deprecation notice as an operational obligation to migrate, not as confirmation that the surface remains supported indefinitely.

---

## Breaking Change Expectations

A breaking change is any change to a compatibility surface that causes a component built against the prior surface to fail, behave differently, or produce incorrect output when operating against the new surface. The following changes are always breaking, regardless of how they are scoped or described:

- Removing a field from a schema
- Renaming a field in a schema
- Changing the semantic meaning of a field
- Removing an action name from the vocabulary
- Changing the evaluation outcome of an existing policy construct
- Changing the normalization mapping for an established protocol operation
- Changing the format of resource identifiers in a way that invalidates existing identifiers
- Removing a required section from a document format specification

Breaking changes require: a major version increment, an ADR documenting the rationale and the migration path, and a deprecation period for the surface being changed. Breaking changes that cannot be accompanied by a defined migration path should not be made until the migration path is understood.

---

## Semantic Versioning Philosophy

The BASIS ecosystem has not yet operationalized a specific versioning scheme. This section describes the philosophy that versioning, when operationalized, should embody.

**Version numbers express compatibility expectations, not feature counts.** A major version increment signals a breaking change. A minor version increment signals additive changes. A patch version signals corrections that do not change semantics.

**Compatibility surfaces version independently.** The audit schema version, the action vocabulary version, the policy format version, and the kernel version are not necessarily in lockstep. A kernel version increment that does not change the audit schema does not require an audit schema version increment.

**Version constraints must be expressed by consumers, not inferred.** A component that depends on a specific version of the audit schema must declare that constraint explicitly, so that breaking changes can be identified before they are deployed to dependent components.

**Kernel versioning carries the highest compatibility weight.** basis-core version increments affect every component in the distribution. The bar for a breaking change to basis-core semantics — evaluation behavior, decision contracts, failure mode contracts — is higher than the bar for a breaking change to a higher-level service.

---

## Forward and Backward Compatibility Expectations

**Backward compatibility** means that a component built against schema version N can correctly operate against data or calls produced by a component at schema version N-1. Backward compatibility is the property that allows newer components to receive data from older components without failure.

**Forward compatibility** means that a component built against schema version N can correctly handle data or calls produced by a component at schema version N+1. Forward compatibility is harder to guarantee, but it is particularly important in ecosystems where components update at different speeds.

The ecosystem's compatibility target:

- Audit records must be forward-compatible for their full retention period. An audit consumer that receives a record with fields it does not recognize must not fail; it must preserve the unrecognized fields without modification.
- Authorization request schemas must be backward-compatible across minor version increments. A basis-gateway at version N must be able to submit requests to a basis-core at version N-1 without losing evaluation correctness for the request forms both versions support.
- Policy format must be backward-compatible across minor version increments. Policies authored against version N must evaluate correctly against the version N+1 policy engine, unless a breaking change was explicitly versioned.

These are design targets. The degree to which they are operationally achievable depends on implementation discipline and testing coverage, not on architectural assertion alone.

---

## Compatibility Testing Expectations

Compatibility commitments that are not tested are not commitments — they are aspirations. The following testing practices are expected for compatibility surfaces as the ecosystem matures:

- **Schema evolution tests** verify that a component operating against a new schema version can correctly interpret data produced under the prior schema version, and vice versa.
- **Policy evaluation regression tests** verify that evaluation outcomes for a defined set of policy-input pairs do not change across kernel version increments, except for changes covered by explicit breaking-change documentation.
- **Audit record interpretation tests** verify that audit records produced by prior component versions remain correctly interpretable by current audit consumers.
- **Action vocabulary tests** verify that action names used in deployed policies continue to resolve to the same operations across action vocabulary version increments.

Compatibility testing is a governance concern, not solely an engineering one. The Basis Foundation is responsible for defining the compatibility testing expectations for each compatibility surface and for requiring that they be satisfied before a major version increment is accepted. This requirement exists at the architectural level; operationalizing it is work for the implementation phase.
