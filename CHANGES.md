# Changes

Version history for the OACP specification.

## v0.1 — draft-00 (cleanup pass applied)

Final cleanup pass before draft-00 publication. Applied:

- **Name posture restored.** Removed "Open Agent Communication Protocol" expansion from README and `OACP_SPEC.md`. The name "OACP" is the working identifier; full name expansion and public name selection are deferred per the frozen naming rule.
- **Anthropic placeholder removed.** README no longer references `https://github.com/anthropics`; replaced with neutral "repository link: pending."
- **`(Pass 1)` footer removed** from `OACP_SPEC.md`.
- **Schema directory split.** Schemas are now organized into `schemas/core/` (17 envelope-block schemas) and `schemas/payloads/` (10 payload-type schemas). All `$ref` paths in payload schemas updated from bare names to `../core/<name>` to match the layout.
- **Bindings renamed and converted.** `cloudevents.md` → `OACP_CLOUDEVENTS_BINDING.md`, `opentelemetry.md` → `OACP_OTEL_MAPPING.md`. The Markdown explanations of AsyncAPI and OpenAPI bindings were replaced by **starter-quality machine-readable YAML**: `OACP_ASYNCAPI.yaml` (AsyncAPI 3.0) and `OACP_OPENAPI.yaml` (OpenAPI 3.1). Both YAML files are valid against their respective specifications and use `$ref` to OACP schemas.
- **Top-level conformance aliases added.** `conformance/` now includes 4 raw envelope JSON files alongside the runner-format vectors: `valid-envelope.json`, `invalid-missing-run-id.json`, `invalid-undeclared-handoff.json`, `invalid-proposal-claims-approval.json`. The runner-format vectors in `vectors/` are unchanged.
- **Conformance README renamed** to `OACP_CONFORMANCE.md` per audit recommendation.
- **Standalone JSON examples updated** to preview v0.2-direction kinds (`tool-request-example.json` uses `messageKind: TOOL_REQUEST`; `error-example.json` uses `messageKind: ERROR`, `messageType: ERROR_RESULT`; `proposal-example.json` uses `proposalStatus: EMITTED`). The envelope schema's `messageKind` enum was extended to include both v0.1 spec-body kinds and the v0.2-direction kinds.

## Known v0.2 Review Items

The following items surfaced during the draft-00 audit and post-audit design conversations. They are **deferred to v0.2 review** rather than fixed in v0.1, because they would require coordinated changes across the spec body, conformance vectors, examples, bindings, and schemas.

A formal candidate list with full design sketches lives in [`V0.2_CANDIDATES.md`](./V0.2_CANDIDATES.md). The queue is organized in three sections:

- **Accepted v0.2 review queue** (15 items) — ready for v0.2 review with full sketches.
- **Reframing required** (2 items) — real concerns whose original framing has protocol-shape problems; need redesign.
- **Deferred to v0.3 / Tier 5** (6 items) — valuable but not v0.2-first.

### Accepted v0.2 review queue summary

#### From draft-00 audit

1. **Message-kind taxonomy review.** The v0.1 draft uses `messageKind` values such as `TOOL_CALL`, `BINDING`, `CAPABILITY_NEGOTIATION`, `RUN`, and `SYSTEM`. The audit proposed an alternate taxonomy with `TOOL_CALL_REQUESTED` as a `messageType` of `messageKind: TOOL_REQUEST`, `ERROR` as a first-class kind, plus `COMMAND`, `QUERY`, `EVIDENCE`, `CONTROL`. The standalone JSON examples preview the v0.2 direction; the spec body, conformance vectors, and markdown walkthroughs continue to use v0.1 kinds. The `messageKind` enum in `core/envelope.schema.json` accepts both sets so neither is a validation failure.

2. **Proposal lifecycle naming review.** The v0.1 spec at §13.4 uses `proposalLifecycleState` with the value `PENDING`. The audit proposed `proposalStatus` with `EMITTED`. The standalone proposal example previews the v0.2 direction; the spec body and `core/proposal.schema.json` retain v0.1 naming.

#### Tier 4 — Protocol Is Governance-Rich

3. **T4.1 Full Obligations Registry** — formalize obligations as first-class artifacts.
4. **T4.2 Data Handling Block (incl. Consent / Purpose Limitation)** — promote data classification to a structured envelope-level block; consent and purpose-of-use folded in.
5. **T4.3 Tool Side-Effect Classes** — protocol-enforced side-effect classification.
6. **T4.4 Quarantine Release Flow** — lifecycle for quarantined items beyond the v0.1 terminal state.
7. **T4.5 Concurrency Locks** — protocol-level locking primitives.
8. **T4.6 Budget Controls** — multi-tier budget mechanisms beyond per-message gating.
9. **T4.7 Reasoning Trace Evidence** — make agent reasoning observable and auditable when policy requires it, via a typed `REASONING_TRACE` artifact subtype and `reasoningTraceRef` on proposals. Token-level filtering remains explicitly out of scope for OACP — that belongs to runtime/model-serving controls.
10. **T4.8 Pre-Execution Run-Intent Gate** — decide whether a run should start before the agent spends tokens, resolves data, calls tools, or emits proposals. New `RunIntent` artifact subtype, `RunIntentDecision` decision subtype, `runIntentRef` and `runIntentDecisionRef` on `RUN_STARTED`, new gate step 0.5 ahead of identity verification.
11. **T4.9 Capability Risk Hints** — manifest-level risk metadata that the existing risk-derivation step consumes. Hints are advisory; agents still cannot self-assign risk class.
12. **T4.10 Policy Simulation / Dry-Run Mode** — protocol-level mechanism to ask "would this be allowed?" without executing. New `executionMode` field, simulation artifact subtypes, `WOULD_*` broker decision values.
13. **T4.11 Output Content Safety Classification** — classify generated output content (`SAFE`, `NEEDS_REVIEW`, `SENSITIVE`, `REGULATED`, `DISALLOWED`, `HIGH_IMPACT`) before delivery. Distinct from `riskClass` — content properties vs. action consequences.
14. **T4.12 Evidence Quality Scoring** — extend `EvidenceReport` with structured quality dimensions (completeness, source reliability, freshness, validation pass rate). New auto-approval precondition for evidence quality threshold.
15. **T4.13 Kill-Switch / Emergency Stop Semantics** — protocol-level emergency-stop with seven cumulative levels (`STOP_RUN` → `GLOBAL_EMERGENCY_STOP`), authority asymmetry (resume requires higher authority than stop), full audit trail.
16. **T4.14 Trajectory-Level Authorization & Session Risk Memory** — temporal governance complement to v0.1's spatial gates. Evaluates whether a sequence of individually valid actions becomes risky over time across a conversation/session. Default scope is `conversationId`; `runId` scope is open for v0.2 review. Candidate objects: `SessionRiskState`, `TrajectoryRiskScore`, `TrajectoryAuthorizationDecision`. Decision values: `ALLOW_CONTINUE`, `REQUIRE_REVIEW`, `SUSPEND_SESSION`, `SUSPEND_CAPABILITY`, `QUARANTINE_SESSION`, `TERMINATE_RUN`. Acknowledged overlap with T4.6 (budgets), T4.7 (reasoning), T4.8 (pre-execution), T4.13 (kill-switch); v0.2 review will decide whether trajectory authorization is standalone or composition.

### Reframing required

- **Runtime Health / Trust Posture** — original framing has self-attestation trust circularity. Reframed as externally-signed `RuntimeHealthAttestation`. Deferred (D5).
- **Agent Drift Detection** — observability concern not protocol primitive. Reframed as externally-emitted `AgentDriftAlert`. Deferred (D6).

### Deferred to v0.3 / Tier 5

D1 Human Review SLA & Escalation Rules · D2 Data Lineage / Provenance Chain · D3 Decision Appeal / Reconsideration Flow · D4 Delegation Chain Governance · D5 RuntimeHealthAttestation · D6 AgentDriftAlert.

See [`V0.2_CANDIDATES.md`](./V0.2_CANDIDATES.md) for the full candidate sketches with design questions, migration paths, and dispositions.

## v0.1 (working draft) — original release notes

Initial public working draft. This release is the result of a structured three-tier design process:

### Tier 1 — Protocol Works (5 deltas)

- T1.1: **Identity, verification, and authorization separation.** Sender-immutable identity, receiver-authored verification, separate `authorizationDecision` block. Three assurance levels (`DECLARED`, `AUTHENTICATED`, `ATTESTED`).
- T1.2: **Capability manifest and binding lifecycle.** Manifest registry with six-state lifecycle, separate binding artifact, binding evaluation at receive time.
- T1.3: **Idempotency, retry, and replay semantics.** `messageId` vs. `idempotencyKey` vs. `causationId`, four delivery modes, four replay modes with authority verification.
- T1.4: **Broker outcome state machine.** 15-state machine, decision-vs-state distinction, sanitization with derived idempotency keys, persist-before-emit rule.
- T1.5: **Conversation, causation, and correlation.** Five identity scopes, causation DAG with four types, fan-out/fan-in semantics, NEEDS_MORE_EVIDENCE re-run flow.

### Tier 2 — Protocol Is Precise (7 deltas)

- T2.1: **Profile registry and composition.** Two-axis profile model, base hierarchy, additive capability profiles, profile claim integrity within run.
- T2.2: **Envelope/payload precedence and extension mechanism.** Four-tier hierarchy, conflict-resolution rules, extension namespace prefixes, `requiredExtensions` as single source of truth.
- T2.3: **Reference model.** Five Reference subtypes (`ArtifactRef`, `MessageRef`, `RecordRef`, `DecisionRef`, `AuditRef`), tenant-scoped resolution, trusted URI resolver rule, externalization rule.
- T2.4: **Tenant isolation.** Four enforcement points, scope types, reference-scope compatibility, cross-tenant policy required, `verification.tenantScope` placement.
- T2.5: **Message age, replay protection, hash-chain semantics.** Three integrity dimensions, JCS-RFC8785 default canonicalization, run-scoped hash chains, signature material binding.
- T2.6: **REVIEW protocol.** State machine, `reviewType` vs `reviewOutcome` split, `REVIEW_FEEDBACK_NOT_DECISION` enforcement, indirect agent request via `REVIEW_HINT_EMITTED`.
- T2.7: **Proposal protocol and decision artifacts.** Proposal lifecycle, ten auto-approval preconditions, `RuntimeDecisionRef` for every closure, `decisionStatus` lifecycle, proposal-to-proposal relationships with authority requirements.

### Tier 3 — Protocol Is Publishable

- T3.1: **Threat model.** Eight threat surfaces, auditability vs prevention, compromised-runtime as catastrophic boundary, valid-but-malicious participant, schema/manifest/policy rollback, confidentiality boundary, required-extension downgrade.
- T3.2: **Versioning and compatibility.** Five versioned axes, pre-1.0 vs post-1.0 expectations, unified six-state lifecycle, deprecation as recommendation, runtime version negotiation deferred.
- T3.3–T3.7: **Publication package.** Schemas, bindings, conformance vectors, examples, governance documents, license, IPR posture.

### Cleanup edits applied

51 cleanup edits from review feedback were incorporated across all deltas:

- 12 from Tier 1 deltas
- 24 from Tier 2 deltas
- 7 from T3.1 (threat model)
- 8 from T3.2 (versioning)

All edits preserve the architectural invariants frozen during the design process.

### Payload-type schemas (additive)

After the Pass 2 sealed archive, ten payload-type schemas were added alongside the seventeen envelope-block schemas. The package now contains a two-tier schema organization:

- **Envelope-block schemas** (17) — validate OACP envelope structure (identity, scope, governance, references, integrity, etc.).
- **Payload-type schemas** (10) — validate `payloadType`-specific content carried inside `payload`.

The payload-type schemas added:

- `oacp-envelope-v1.schema.json` — versioned envelope wrapper (delegates to `envelope.schema.json` with `oacpVersion: "0.1"` constraint).
- `oacp-error-v1.schema.json` — `ERROR_EMITTED` payload.
- `oacp-handoff-v1.schema.json` — `HANDOFF_REQUESTED` and related lifecycle payloads. Models a governed transition between participants/roles, distinct from the binding artifact (which is authority assignment, not transition). Reserved statuses: `REQUESTED`, `VALIDATING`, `ACCEPTED`, `DENIED`, `COMPLETED`, `FAILED`, `EXPIRED`, `CANCELLED`.
- `oacp-tool-request-v1.schema.json` / `oacp-tool-result-v1.schema.json` — Tool Broker request/result payloads.
- `oacp-gate-request-v1.schema.json` / `oacp-gate-result-v1.schema.json` — gate runner request/result payloads.
- `oacp-evidence-ref-v1.schema.json` — constrained `ArtifactRef` profile with `artifactType` pinned to `EVIDENCE_REPORT`.
- `oacp-proposal-v1.schema.json` — `PROPOSAL_EMITTED` payload (carries `proposalRef` to the resolved artifact body).
- `oacp-escalation-request-v1.schema.json` — `ESCALATION_REQUESTED` payload per spec §3a.13.

Four standalone JSON envelope examples were added alongside the markdown walkthroughs: `handoff-example.json`, `tool-request-example.json`, `proposal-example.json`, `error-example.json`. Each validates against `oacp-envelope-v1.schema.json` and its declared payload schema.

The envelope-block schemas, conformance vectors, and markdown walkthroughs from the Pass 2 sealed archive are unchanged. The additive layout was confirmed via explicit decision rather than overwrite, preserving the prior frozen archive's structural integrity.

### Frozen architectural invariants

The following invariants are frozen for v0.1 and will not be relaxed in subsequent releases:

1. Identity is sender-authored and immutable; verification is receiver-authored and appended.
2. Tenant authorization lives in `verification.tenantScope`, not in `identity`.
3. Proposals are recommendations; approvals require separate decision artifacts.
4. Review feedback never finalizes a decision.
5. Every proposal closure produces a decision record.
6. Hash chains are run-scoped.
7. Tenant isolation is a protocol-level invariant.
8. Extensions cannot weaken core.
9. Self-decision is forbidden (with runtime auto-approval as the only exception).
10. Manifests are immutable; lifecycles are unified across registry artifacts.

### Known deferrals

The following are scoped but explicitly deferred to **Tier 4 / v0.2**:

- Obligations / data handling beyond `dataClasses` declarations.
- Side-effect class enforcement beyond `READ_ONLY` declaration.
- Quarantine release flow.
- Concurrency locks.
- Budget controls (rate limits, cost caps).
- Mid-flow protocol version negotiation.
- Message-level encryption.

Deferred to **Tier 5 / v0.3+**:

- Federation across deployments.
- Formal global registry authority.
- Advanced regulated profile expansion.
- Post-quantum cryptographic algorithms.

### Working-name and public-name

OACP v0.1 ships with **OACP** as the working name throughout. Public name selection is deferred until v0.1 schemas and conformance vectors are stable. When selected, the public name will be substituted via find-and-replace; no other changes are anticipated.

---

*Future releases will be added here in reverse chronological order.*
