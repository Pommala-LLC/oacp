# OACP — Working Draft Protocol Specification

**Version:** 0.1 (working draft)
**Status:** Draft for review
**Working name:** OACP. The name is a working identifier for v0.1; full name expansion and public name selection are deferred until v0.1 schemas and conformance vectors are stable.
**License:** Apache 2.0 — see [`LICENSE`](./LICENSE)

---

## What is OACP

OACP is a **transport-neutral, contract-first protocol for governed agent-runtime communication**. It defines:

- A typed envelope with deterministic processing semantics.
- A two-axis profile system (Core → Operational → Governed → Regulated, plus additive capability profiles).
- A reference model (ArtifactRef, MessageRef, RecordRef, DecisionRef, AuditRef) with tenant-scoped resolution.
- A proposal/decision boundary that prevents agents from approving their own outputs.
- A capability binding lifecycle, broker state machine, review flow, and idempotency model.
- Run-scoped hash chains, message-age limits, and signature semantics.
- An extension mechanism that can impose stricter validation but cannot weaken core enforcement.

Every OACP message passes through twelve governance gates — identity, tenant scope, authority, capability binding, authorization decision, idempotency, broker state, response correlation, causation DAG, conversation scope, payload validation, then dispatch — before any business effect occurs.

OACP is a **governance envelope that layers on top of A2A, MCP, and ACNBP**. It is not a replacement for them; it adds the policy, evidence, audit, and proposal-boundary semantics those protocols leave to the host.

**Ojas** is the planned reference implementation. Repository link: pending.

---

## Why OACP exists

Existing agent-communication protocols solve transport (A2A), tool integration (MCP), and capability negotiation (ACNBP). None of them prescribe what *governed* agent communication looks like — what makes a message replay-safe, what constitutes an approval, when an agent can be trusted to bind a capability, how cross-tenant references work, what audit chain a runtime must produce.

OACP is what fills that gap. It is opinionated about governance, deliberately minimal about transport, and designed to compose with existing protocols rather than replace them.

---

## Safety Boundary

OACP is an operational AI governance protocol. It helps make agent actions attributable, scoped, policy-bound, reviewable, auditable, replay-safe, and decision-separated.

OACP does not make AI models truthful, aligned, or inherently safe. It does not solve model hallucination, jailbreak resistance, AI alignment, or AGI safety.

OACP implements operational governance support for alignment oversight, model-safety workflows, and truthfulness-verification processes. OACP does not implement or guarantee model alignment, model safety, or truthfulness. OACP provides governance primitives — identity, authority, capability binding, authorization decisions, tenant scope, tool mediation, review, proposal/decision separation, evidence references, validation results, audit chains, replay protection, and hash-chain integrity — that make these oversight workflows attributable, reviewable, auditable, and policy-enforceable. Future OACP v0.2 candidates strengthen this support through RunIntent gates, reasoning trace evidence, output safety classification, evidence quality scoring, tool side-effect classes, policy simulation, kill-switch semantics, and data handling / purpose limitation.

OACP provides governance controls around agent execution. Model behavior, truthfulness, alignment, and content safety remain responsibilities of the model provider, runtime, deployment policy, validation systems, reviewers, and downstream controls.

---

## Reading order

| Document | Purpose |
|---|---|
| [`OACP_SPEC.md`](./OACP_SPEC.md) | The full consolidated specification (≈160 KB). Authoritative. |
| [`SECURITY.md`](./SECURITY.md) | Security posture, threat model summary, vulnerability reporting. |
| [`GOVERNANCE.md`](./GOVERNANCE.md) | How decisions about the protocol are made. |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to contribute. |
| [`IPR.md`](./IPR.md) | Intellectual property posture. |
| [`CHANGES.md`](./CHANGES.md) | Version history; known v0.2 review items. |
| [`V0.2_CANDIDATES.md`](./V0.2_CANDIDATES.md) | Formal v0.2 / Tier 4 candidate list with design sketches. |
| [`LICENSE`](./LICENSE) | Apache 2.0 license. |
| [`schemas/`](./schemas) | JSON Schema 2020-12 definitions for the envelope and core artifacts. |
| [`bindings/`](./bindings) | Transport bindings: CloudEvents, AsyncAPI, OpenAPI, OpenTelemetry. |
| [`conformance/`](./conformance) | Conformance test vectors. |
| [`examples/`](./examples) | End-to-end example flows. |

If you are reading OACP for the first time, the recommended path is:

1. This README.
2. `OACP_SPEC.md` §0 (Foreword) and §1 (Conventions).
3. `OACP_SPEC.md` §2a (Profiles), §3a (Identity), §13 (Proposal/Decision).
4. `examples/` — pick one end-to-end flow.
5. `schemas/core/envelope.schema.json` — the envelope itself.
6. `OACP_SPEC.md` §22 (Conformance) and §26 (Threat model).

---

## Architectural fundamentals

A short list of OACP's frozen invariants. Every section of the spec elaborates one or more of these.

1. **Identity is sender-authored and immutable. Verification is receiver-authored and appended.** A `verificationTrail` accumulates per-hop verification records.
2. **Tenant authorization lives in `verification.tenantScope`, not in `identity`.** Identity says who the participant claims to be; tenant scope says which tenants the verified identity may operate in.
3. **Proposals are recommendations, never approvals.** Approval requires a separate decision artifact (`ExternalDecisionRef` for human deciders, `RuntimeDecisionRef` for runtime auto-approval, expiry, supersession).
4. **Review feedback never finalizes a decision.** A reviewer responding "this looks good" is not approval. Approval is a distinct artifact under §13.
5. **Every proposal closure produces a decision record.** Even `EXPIRED` and `SUPERSEDED` proposals have decision artifacts. Closure histories are reconstructable from artifacts alone.
6. **Hash chains are run-scoped.** They do not cross run boundaries even when runs share a `conversationId`. Cross-run continuity uses causation, `proposalRefs[]`, and audit infrastructure.
7. **Tenant isolation is a protocol-level invariant.** Every Reference resolved by a receiver is checked against the receiver's authorized scope. Cross-tenant flows require explicit registered policy.
8. **Extensions cannot weaken core.** Required extensions can impose stricter validation; they cannot disable a core gate.
9. **Self-decision is forbidden.** A decider cannot equal the proposer. The only exception: runtime auto-approval, where the runtime is a separate participant from the agents whose proposals it governs.
10. **Manifests are immutable. Lifecycles are unified.** All registry artifacts (manifests, profile versions, extensions, payload schemas, policies) share the same six-state lifecycle (`ACTIVE`, `DEPRECATED`, `DEACTIVATED`, `BLOCKED`, `RETIRED`, `REVOKED`).

---

## Profiles

OACP messages compose two axes of profile claims.

**Base profiles** (exactly one per message; strict hierarchy):

| Base profile | Implies | Meaning |
|---|---|---|
| `OACP-Core` | None | Trusted in-process flows |
| `OACP-Operational` | `OACP-Core` | Authentication and message-age enforcement |
| `OACP-Governed` | `OACP-Operational`, `OACP-Capability`, `OACP-Binding` | Full envelope governance, tenant isolation, capability binding, authorization decisions |
| `OACP-Regulated` | `OACP-Governed`, `OACP-Reliable` | Cryptographic attestation, hash-chain enforcement, signed decisions, immutable audit |

**Capability profiles** (zero or more; additive):

| Capability profile | Purpose |
|---|---|
| `OACP-Capability` | Capability manifest registration |
| `OACP-Binding` | Capability binding lifecycle |
| `OACP-Reliable` | Idempotency, retry, replay |
| `OACP-Tool` | Tool broker mediation |
| `OACP-Review` | REVIEW protocol |
| `OACP-Proposal` | Proposal/decision protocol |
| `OACP-Interop` | Cross-deployment interoperability |

The base profile registry is **major-version-closed**; the capability profile registry is **controlled-additive**.

---

## What's in this package

```
oacp/
├── README.md                  ← this file
├── LICENSE                    ← Apache 2.0
├── OACP_SPEC.md               ← full consolidated spec
├── SECURITY.md                ← security posture
├── GOVERNANCE.md              ← protocol governance
├── CONTRIBUTING.md            ← contribution guide
├── IPR.md                     ← intellectual property posture
├── CHANGES.md                 ← version history
├── V0.2_CANDIDATES.md         ← v0.2 / Tier 4 candidate list
│
├── schemas/
│   ├── README.md
│   │
│   ├── core/                  ← envelope-block schemas (17 files)
│   │   ├── envelope.schema.json
│   │   ├── identity.schema.json
│   │   ├── verification.schema.json
│   │   ├── scope.schema.json
│   │   ├── governance.schema.json
│   │   ├── causation.schema.json
│   │   ├── integrity.schema.json
│   │   ├── reference.schema.json
│   │   ├── artifact-ref.schema.json
│   │   ├── message-ref.schema.json
│   │   ├── record-ref.schema.json
│   │   ├── decision-ref.schema.json
│   │   ├── audit-ref.schema.json
│   │   ├── proposal.schema.json
│   │   ├── capability-manifest.schema.json
│   │   ├── binding.schema.json
│   │   └── error-result.schema.json
│   │
│   └── payloads/              ← payload-type schemas (10 files)
│       ├── oacp-envelope-v1.schema.json
│       ├── oacp-error-v1.schema.json
│       ├── oacp-handoff-v1.schema.json
│       ├── oacp-tool-request-v1.schema.json
│       ├── oacp-tool-result-v1.schema.json
│       ├── oacp-gate-request-v1.schema.json
│       ├── oacp-gate-result-v1.schema.json
│       ├── oacp-evidence-ref-v1.schema.json
│       ├── oacp-proposal-v1.schema.json
│       └── oacp-escalation-request-v1.schema.json
│
├── bindings/
│   ├── README.md
│   ├── OACP_CLOUDEVENTS_BINDING.md
│   ├── OACP_ASYNCAPI.yaml          ← AsyncAPI 3.0 (machine-readable)
│   ├── OACP_OPENAPI.yaml           ← OpenAPI 3.1 (machine-readable)
│   └── OACP_OTEL_MAPPING.md
│
├── conformance/
│   ├── OACP_CONFORMANCE.md
│   ├── runner-spec.md
│   ├── valid-envelope.json                        ← raw envelope alias
│   ├── invalid-missing-run-id.json                ← raw envelope alias
│   ├── invalid-undeclared-handoff.json            ← raw envelope alias
│   ├── invalid-proposal-claims-approval.json      ← raw envelope alias
│   │
│   └── vectors/                ← runner-format test vectors
│       ├── happy-path/
│       ├── error-codes/
│       ├── profile-boundary/
│       ├── state-machine/
│       ├── hash-chain/
│       └── tenant-scope/
│
└── examples/
    ├── README.md
    ├── handoff-example.json                ← standalone JSON
    ├── tool-request-example.json           ← standalone JSON
    ├── proposal-example.json               ← standalone JSON
    ├── error-example.json                  ← standalone JSON
    ├── 01-basic-proposal.md                ← markdown walkthrough
    ├── 02-tool-call-with-broker.md
    ├── 03-review-needs-more-evidence.md
    ├── 04-cross-tenant-policy.md
    └── 05-replay-original-policy.md
```

---

## Versions

This is OACP **v0.1**. It is a pre-1.0 working draft. Backward compatibility between v0.x releases is best-effort, not normative. Implementations targeting v0.x SHOULD pin to a specific version.

Post-1.0 (v1.x and later), all changes within a major version MUST be backward-compatible. See [`OACP_SPEC.md`](./OACP_SPEC.md) §27.

---

## Reference implementation

**Ojas** is the reference implementation of OACP. Implementations of OACP are not required to be Ojas-derived; OACP is implementation-neutral. The protocol exists independently of any single runtime.

---

## Status

| Tier | Status |
|---|---|
| Tier 1 — Protocol Works | ✅ Frozen |
| Tier 2 — Protocol Is Precise | ✅ Frozen |
| Tier 3 — Protocol Is Publishable | ✅ Frozen (this release) |
| Tier 4 — Governance-Rich (obligations, data handling, side-effect classes, quarantine release, concurrency, budgets) | 🔜 Scoped, deferred |
| Tier 5 — Ecosystem-Ready (formal registry authority, federation, advanced regulated profile expansion) | 🔜 Scoped, deferred |

Tier 4 work begins after community review of v0.1.

---

## Contact and feedback

Feedback on the v0.1 draft is welcomed. See [`CONTRIBUTING.md`](./CONTRIBUTING.md).

For security issues, see [`SECURITY.md`](./SECURITY.md).
