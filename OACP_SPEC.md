# OACP — Working Draft Protocol Specification

**Version:** 0.1 (working draft)
**Status:** Draft for review
**Working name:** OACP. The name is a working identifier for v0.1; full name expansion and public name selection are deferred until v0.1 schemas and conformance vectors are stable. The name "OACP" is used throughout this document as the working identifier; substitution to a final public name will be a find-and-replace operation.
**License:** Apache 2.0 (see `LICENSE`)
**Reference implementation:** Ojas (planned; repository link pending)

---

## §0. Foreword

OACP is a transport-neutral, contract-first protocol for governed agent-runtime communication. It defines typed envelopes, profile-based validation, participant identity, tenant scope, authority, capability binding, authorization decisions, idempotency, broker state, response correlation, causation DAGs, conversation scope, typed references, review flows, proposal/decision separation, replay protection, message-age limits, run-scoped hash chains, and extension-safe validation.

OACP makes governed agent communication **deterministic, auditable, replay-safe, and policy-enforceable**.

This specification is the result of a structured design process across three tiers:

- **Tier 1 — Protocol Works.** Identity, capability binding, idempotency, broker state, conversation/causation. The protocol can safely process governed agent messages from start to finish.
- **Tier 2 — Protocol Is Precise.** Profile registry, precedence, references, tenant isolation, review, proposal/decision, message integrity. Two independent implementations validate and interpret the same OACP message the same way.
- **Tier 3 — Protocol Is Publishable.** Threat model, versioning, schemas, conformance vectors, bindings, governance documents.

Tier 4 (Governance-Rich: obligations, data handling, side-effect classes, quarantine release, concurrency, budgets) and Tier 5 (Ecosystem-Ready: formal registry authority, advanced regulated profile expansion) are scoped but not delivered in v0.1.

### §0.1 Scope and positioning

OACP is a governance envelope for governed agent runtimes. It is **not**:

- A specification of agent reasoning or model behavior.
- A specification of transport-layer security (TLS, mTLS, etc.).
- A specification of identity-provider or credential-issuer protocols.
- A replacement for A2A, MCP, or ACNBP — it layers on top of them, adding policy, evidence, audit, and proposal-boundary semantics those protocols leave to the host.

OACP's positive scope, by contrast, includes the workflow layer around AI safety, alignment, and truthfulness concerns. OACP implements operational governance support for alignment oversight, model-safety workflows, and truthfulness-verification processes. OACP does not implement or guarantee model alignment, model safety, or truthfulness. OACP provides governance primitives — identity, authority, capability binding, authorization decisions, tenant scope, tool mediation, review, proposal/decision separation, evidence references, validation results, audit chains, replay protection, and hash-chain integrity — that make these oversight workflows attributable, reviewable, auditable, and policy-enforceable. Future OACP v0.2 candidates strengthen this support through RunIntent gates, reasoning trace evidence, output safety classification, evidence quality scoring, tool side-effect classes, policy simulation, kill-switch semantics, and data handling / purpose limitation. Model-level guarantees remain the responsibility of the model provider, training and evaluation processes, and runtime sandboxing.

### §0.2 Reading this specification

Sections are organized along the protocol's processing order. Recommended reading order:

1. **§0–§2** — front matter, conventions, profiles, extensions, references, tenant isolation.
2. **§3** — envelope mechanics: identity, capabilities, idempotency, conversation, integrity.
3. **§10–§13** — message-kind protocols: broker, review, proposal/decision.
4. **§22** — conformance.
5. **§26–§27** — threat model and versioning policy.

Each section is self-contained; cross-references use section numbers. The processing-gate ordering in §1.2 gives the canonical sequence of validation; sections elaborate each gate.

### §0.3 Notational conventions

- **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** — RFC 2119 keywords.
- **Reserved values** — values defined by this specification that implementations MUST recognize.
- **Namespace prefixes** — `x-org-*`, `x-vendor-*`, `x-regulated-*`, `x-experimental-*` are reserved for non-spec extensions per §2b.7. Bare `x-*` is permitted only at lower profiles.
- **JSON examples** — illustrative; canonical hashing follows JSON Canonicalization Scheme (JCS, RFC 8785) per §3e.

### §0.4 Reference implementation

**Ojas** is the reference implementation of OACP. References to "Ojas" indicate the reference implementation; references to "OACP" indicate the protocol itself. Implementations of OACP are not required to be Ojas-derived; OACP is implementation-neutral.

---

## §1. Conventions

### §1.1 Message structure

Every OACP message is a typed envelope with a typed payload. The envelope carries protocol-level metadata; the payload carries message-kind-specific content.

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_001",
  "messageKind": "TOOL_CALL",
  "messageType": "TOOL_CALL_REQUESTED",
  "createdAt": "2026-05-05T17:00:00Z",
  "scope": { "...": "..." },
  "from": { "...": "..." },
  "governance": { "...": "..." },
  "ttl": { "...": "..." },
  "run": { "...": "..." },
  "conversation": { "...": "..." },
  "causation": { "...": "..." },
  "responseTo": { "...": "..." },
  "delivery": { "...": "..." },
  "replay": { "...": "..." },
  "integrity": { "...": "..." },
  "refs": { "...": "..." },
  "extensions": { "...": "..." },
  "requiredExtensions": [],
  "payloadType": "tool-call-request-v1",
  "payload": { "...": "..." }
}
```

Top-level fields organize into blocks. Each block is defined in its respective section.

### §1.2 The processing gate

Every OACP message passes through governance gates in strict order. Failure at any gate produces a structured error and halts processing.

```
1.   Verify identity                               (§3a)
1.5. Verify tenant/workspace/system scope          (§2d)
2.   Verify participant authority                  (§3a.8)
3.   Verify capability manifest and binding        (§3b)
4.   Verify authorizationDecision                  (§3a.6)
5.   Verify idempotency / retry / replay state     (§3c)
6.   Verify broker state, if broker-managed        (§10)
7.   Verify responseTo, if response/result         (§3d.11)
8.   Verify causation DAG                          (§3d.7)
9.   Verify conversation scope                     (§3d.2)
10.  Verify reference scope and integrity          (§2c, §2d)
11.  Verify profile / extension / precedence       (§2a, §2b)
12.  Verify proposal / review / decision state     (§11a, §13)
13.  Verify message age / hash-chain / signature   (§3e)
14.  Validate payload                              (§1.3)
15.  Dispatch, reject, quarantine, or emit error
```

Steps 1–9 are envelope-authority gates; 10–13 integrity and profile; 14–15 payload-level. Implementations MAY skip gates where no relevant data is present (e.g., step 6 for non-broker messages), but MUST NOT skip a gate that has data to evaluate.

Tenant scope (gate 1.5) is **not skippable** at any profile when tenant scoping is configured (§2d.8).

### §1.3 Payload validation

Payloads are validated against their declared `payloadType`. The payload schema is identified by `payloadType` (e.g., `proposal-v1`, `tool-call-request-v1`), resolved through trusted resolvers (§2b.6 rule 7).

Receivers MUST reject payloads that fail schema validation with `PAYLOAD_VALIDATION_FAILED`. Receivers MAY emit additional structured errors describing specific validation failures.

---

## §2a. Governance profile registry and composition rules

### §2a.1 Two-axis profiles

OACP profiles compose along two axes:

- **Base profiles** form a strict hierarchy of assurance strictness. **Exactly one** base profile applies per message.
- **Capability profiles** are additive feature surfaces. **Zero or more** capability profiles apply.

### §2a.2 Base profile registry

| Base profile | Implies | Meaning |
|---|---|---|
| `OACP-Core` | None | Minimum assurance. Suitable for trusted in-process flows. |
| `OACP-Operational` | `OACP-Core` | Authentication and message-age enforcement required. |
| `OACP-Governed` | `OACP-Operational`, `OACP-Capability`, `OACP-Binding` | Full envelope governance, tenant isolation, capability binding, authorization decisions. |
| `OACP-Regulated` | `OACP-Governed`, `OACP-Capability`, `OACP-Binding`, `OACP-Reliable` | Cryptographic attestation, hash-chain enforcement, signed decisions, immutable audit. |

Base profiles imply lower base profiles. A receiver claiming `OACP-Governed` enforces all `OACP-Operational` rules and all `OACP-Core` rules.

The base profile registry is **major-version-closed**: new base profiles or restructured base hierarchy require a major protocol version bump (§27.2).

### §2a.3 Capability profile registry

| Capability profile | Minimum base | Requires | Purpose |
|---|---|---|---|
| `OACP-Capability` | `OACP-Operational` | None | Capability manifest registration |
| `OACP-Binding` | `OACP-Operational` | `OACP-Capability` | Capability binding lifecycle |
| `OACP-Reliable` | `OACP-Operational` | None | Idempotency, retry, replay semantics |
| `OACP-Tool` | `OACP-Operational` | `OACP-Binding`, `OACP-Reliable` | Tool broker mediation |
| `OACP-Review` | `OACP-Operational` | None | REVIEW protocol semantics |
| `OACP-Proposal` | `OACP-Operational` | None | Proposal/decision protocol |
| `OACP-Interop` | Any | None | Cross-deployment interoperability extensions |

Capability profiles are **additive**. A message claiming `OACP-Tool` carries Tool semantics in addition to whatever its base profile provides.

The capability profile registry is **controlled-additive**: new capability profiles MAY be added in minor protocol versions, but existing capability profiles' semantics are stable within a major version.

### §2a.4 Profile composition

Messages declare profile claims in `governance.profile` and `governance.profileVersion`:

```json
"governance": {
  "profile": "OACP-Governed",
  "profileVersion": "0.1.0",
  "capabilityProfiles": ["OACP-Tool", "OACP-Review"],
  "effectiveCapabilityProfiles": [
    "OACP-Capability", "OACP-Binding", "OACP-Reliable",
    "OACP-Tool", "OACP-Review"
  ]
}
```

`effectiveCapabilityProfiles` is the union of declared capability profiles and those implied by the base profile or by capability-profile prerequisites. Implementations MAY compute this; receivers SHOULD verify it matches their derivation.

### §2a.5 Profile claim integrity

Profile claims are **stable within a run** (§3d.2 — `runId`). A message in run R MUST claim the same `profile` and `profileVersion` as all other messages in run R. Profile drift mid-run is rejected with `PROFILE_DRIFT_WITHIN_RUN`.

Cross-run profile changes are permitted between runs in the same conversation.

### §2a.6 Receiver-side profile alignment

Receivers MAY enforce profile alignment:

1. A receiver claiming `OACP-Governed` MAY accept lower-profile messages but MUST NOT accept higher-profile claims it cannot satisfy.
2. A receiver claiming `OACP-Operational` MUST accept `OACP-Operational` messages; MAY accept lower-profile messages; MUST NOT accept `OACP-Governed`+ claims.
3. Mismatch rejections use `PROFILE_VERSION_UNSUPPORTED` (§27.10).

### §2a.7 Skipping gates at lower profiles

At lower profiles, more gates may legitimately be absent:

- `OACP-Core`: gates 3, 4, 6, 11–13 typically inapplicable.
- `OACP-Operational`: gates 6, 12 typically inapplicable.
- `OACP-Governed`: most gates apply; gate 13 is detective rather than blocking.
- `OACP-Regulated`: all applicable gates are blocking.

### §2a.8 Profile envelope authority

The profile claim is **envelope authority metadata** (Tier A per §2b.2). Payload content cannot override profile claims. A payload contradicting the envelope's profile is rejected with `ENVELOPE_PAYLOAD_CONFLICT`.

### §2a.9 Reserved error codes (§2a)

```
PROFILE_DRIFT_WITHIN_RUN             — profile claim changed mid-run
PROFILE_VERSION_UNSUPPORTED          — receiver doesn't support claimed profile version
PROFILE_INVALID                      — profile claim doesn't match registry
CAPABILITY_PROFILE_PREREQUISITE_NOT_MET — capability profile claimed without required prerequisite
EFFECTIVE_PROFILES_INCONSISTENT      — declared effectiveCapabilityProfiles doesn't match derivation
```

---

## §2b. Envelope/payload precedence and extension mechanism

### §2b.1 The four-tier precedence hierarchy

OACP fields organize into four tiers. Higher-tier fields prevail in conflicts.

| Tier | Name | Examples | Role |
|---|---|---|---|
| **A** | Envelope authority | `oacpVersion`, `messageKind`, `governance.profile`, `from.identity`, `scope.tenantId`, `integrity.envelopeHash` | Cannot be overridden |
| **B** | Governance metadata | `authorizationDecision`, `governance.riskClass`, `bindingId`, `policySnapshotId` | Informs decisions |
| **C** | Payload | All fields under `payload` | Message-kind content |
| **D** | Extensions | All fields under `extensions` | Optional metadata |

### §2b.2 Conflict resolution

**Conflicts are errors, not merges.** When a Tier-A field and a payload field carry conflicting information, the receiver rejects with `ENVELOPE_PAYLOAD_CONFLICT`. Examples:

1. Envelope `scope.tenantId: tenant_001`, payload `tenantId: tenant_002` → reject.
2. Envelope `from.identity.subject: agent_a`, payload `signedBy: agent_b` → reject.
3. Envelope `governance.profile: OACP-Operational`, payload `profileSemantics: regulated` → reject.

### §2b.3 Extension model

Extensions live under the top-level `extensions` block. Each has a namespaced key and a value.

```json
"extensions": {
  "x-org-finance-v1": {
    "approvalChainRef": "approval-chain-abc",
    "complianceTags": ["SOX", "PCI"]
  }
}
```

### §2b.4 Required extensions

The top-level `requiredExtensions` array lists extensions the message *requires* — receivers MUST honor or reject them.

```json
"requiredExtensions": ["x-org-finance-v1"]
```

`requiredExtensions` is the **single source of truth** for extension required-ness. Per-extension `required: true` flags inside extension values are NOT recognized.

### §2b.5 Extension namespace prefixes

Reserved prefixes:

| Prefix | Purpose |
|---|---|
| `x-org-*` | Organization-level extensions |
| `x-vendor-*` | Vendor-specific extensions |
| `x-regulated-*` | Regulated-domain extensions |
| `x-experimental-*` | Experimental, non-stable |

Bare `x-*` is permitted at `OACP-Core`/`OACP-Operational` only. At `OACP-Governed`+, bare `x-*` is rejected with `EXTENSION_NAMESPACE_INVALID`.

### §2b.6 Extension rules

1. **Extensions cannot override core envelope authority.** Tier D < Tier A.
2. **Required extensions MAY impose stricter validation** — additional payload fields, integrity checks, reference resolutions.
3. **`requiredExtensions` is the single source of truth.**
4. **Extensions cannot weaken core rules.** A required extension saying "ignore message age" is rejected with `EXTENSION_WEAKENS_CORE`.
5. **`OACP-Core` MUST honor or reject `requiredExtensions`.**
6. **Unknown optional extensions MAY be ignored** but SHOULD be preserved when forwarding.
7. **Schema resolution.** `schemaRef` MUST resolve through trusted local registry/cache. Live-fetch resolution is not mandatory; untrusted-resolver references are rejected with `EXTENSION_SCHEMA_REF_UNTRUSTED`.

### §2b.6a Extension preservation rule

When messages traverse intermediaries, extensions MUST be preserved or the message MUST be rejected. An intermediary stripping an unknown extension violates `EXTENSION_PRESERVATION_VIOLATION`.

For required extensions, a drop in transit triggers `REQUIRED_EXTENSION_DROPPED_BY_INTERMEDIARY` at the next receiver detecting the loss (compares `requiredExtensions` to the actual `extensions` block).

### §2b.7 Extension namespace ownership

| Namespace | Authority |
|---|---|
| `x-org-*` | The organization implied by the suffix |
| `x-vendor-*` | The vendor implied by the suffix |
| `x-regulated-*` | A formal regulatory authority (out of v0.1 scope) |
| `x-experimental-*` | The implementer; no stability guarantees |

### §2b.8 Reserved error codes (§2b)

```
ENVELOPE_PAYLOAD_CONFLICT            — Tier-A and lower-tier fields conflict
EXTENSION_NAMESPACE_INVALID          — bare x-* used at Governed/Regulated
EXTENSION_WEAKENS_CORE               — extension attempts to weaken core rule
EXTENSION_SCHEMA_REF_UNTRUSTED       — schemaRef not in trusted resolver set
EXTENSION_PRESERVATION_VIOLATION     — intermediary stripped extension
REQUIRED_EXTENSION_DROPPED_BY_INTERMEDIARY — required extension lost in transit
EXTENSION_VERSION_UNSUPPORTED        — required extension version not supported
EXTENSION_REQUIRED_NOT_HONORED       — receiver doesn't recognize required extension
ENVELOPE_EXTENSION_CONFLICT          — extension attempts to override envelope authority
```

---

## §2c. Reference model, ArtifactRef, and large-payload handling

### §2c.1 The reference family

OACP defines a `Reference` parent abstraction with five subtypes. Each subtype answers a specific question.

| Subtype | What it references | Carries content? |
|---|---|---|
| `ArtifactRef` | Externally-resolvable content artifact | Yes — fetchable bytes |
| `MessageRef` | An OACP message in scope | No — message identity |
| `RecordRef` | Control-plane or registry record | No — registry lookup |
| `DecisionRef` | Authorization/broker/external decision | No — decision lookup |
| `AuditRef` | Audit-chain entry | Sometimes (when exported) |

A field that names what's being referenced points to exactly one subtype.

### §2c.1a Non-artifact reference subtypes

**`MessageRef`** — references an OACP message:

```json
{
  "messageId": "msg_001",
  "messageType": "TOOL_CALL_REQUESTED",
  "runId": "run_001",
  "tenantId": "tenant_001"
}
```

Used for: `causationId`, `causationIds[]`, `responseTo.requestMessageId`, `replay.originalMessageId`. Bare-string forms remain permitted in within-run contexts at all profiles. Full `MessageRef` is required at `OACP-Governed`+ when the reference crosses run boundaries.

**`RecordRef`** — references a registered control-plane record:

```json
{
  "recordId": "manifest.agent.classifier.v1",
  "recordType": "CAPABILITY_MANIFEST",
  "registry": "manifest-registry",
  "scopeType": "TENANT",
  "tenantId": "tenant_001",
  "version": "1.0.0"
}
```

For globally-scoped registry records:

```json
{
  "recordId": "oacp-profile-registry-v0.1",
  "recordType": "PROFILE_REGISTRY",
  "registry": "oacp-registry",
  "scopeType": "GLOBAL"
}
```

Reserved `recordType` values: `CAPABILITY_MANIFEST`, `BINDING_RECORD`, `POLICY_SNAPSHOT`, `PROFILE_REGISTRY`, `CROSS_TENANT_POLICY`, `REVIEW_FEEDBACK`, `AUTO_APPROVAL_PRECONDITIONS`, `MESSAGE_SIGNATURE`.

**`DecisionRef`** — references a decision artifact:

```json
{
  "decisionId": "authz_decision_001",
  "decisionType": "AUTHORIZATION",
  "decidedBy": "policy-engine-001",
  "decidedAt": "2026-05-05T17:00:00Z",
  "tenantId": "tenant_001"
}
```

Reserved `decisionType` values: `AUTHORIZATION`, `BROKER`, `EXTERNAL_APPROVAL`, `EXTERNAL_REJECTION`, `RUNTIME_AUTO_APPROVAL`, `AUTO_APPROVAL`, `RUNTIME_REJECTION`, `RISK_ASSESSMENT`, `REPLAY_AUTHORIZATION`, `SUPERSESSION`, `EXPIRY`.

**`AuditRef`** — references an audit-chain entry:

```json
{
  "auditId": "audit_001",
  "auditChainRef": "chain_run_001",
  "tenantId": "tenant_001"
}
```

### §2c.2 ArtifactRef definition

**`ArtifactRef` is the canonical reference shape for externally-resolvable content artifacts whose integrity, size, MIME type, expiry, and tenant scope must be checked before use.**

An `ArtifactRef` points to bytes — a document, a generated PDF, an evidence report's serialized payload, a tool input/output, a sanitized payload. It does NOT replace `MessageRef`, `RecordRef`, `DecisionRef`, or `AuditRef`.

```json
{
  "artifactId": "artifact_001",
  "artifactType": "EVIDENCE_REPORT",
  "uri": "ojas://tenant_001/evidence/evidence_001",
  "mimeType": "application/json",
  "hash": "sha256:abc123...",
  "sizeBytes": 4096,
  "createdAt": "2026-05-05T17:00:00Z",
  "expiresAt": "2026-05-12T17:00:00Z",
  "tenantId": "tenant_001",
  "workspaceId": "workspace_001",
  "dataClasses": ["INTERNAL"],
  "retentionClass": "STANDARD_AUDIT"
}
```

### §2c.3 ArtifactRef field semantics

| Field | Required | Meaning |
|---|---|---|
| `artifactId` | Yes | Stable identifier within tenant scope |
| `artifactType` | Yes | Type discriminator |
| `uri` | Yes (when resolvable) | Resolution URI |
| `mimeType` | At Governed+ | MIME type |
| `hash` | At Governed+ | Algorithm-prefixed hash |
| `sizeBytes` | At Governed+ | Encoded byte size |
| `createdAt` | Yes | ISO-8601 emission timestamp |
| `expiresAt` | Yes | ISO-8601 expiration |
| `tenantId` | Yes | Tenant scope |
| `workspaceId` | When configured | Workspace scope |
| `dataClasses` | Optional | Data classification tags |
| `retentionClass` | Optional | Retention-policy class |

Reserved `artifactType` values: `DOCUMENT`, `TOOL_INPUT`, `TOOL_RESULT`, `SANITIZED_PAYLOAD`, `RECOMMENDATION`, `EVIDENCE_REPORT`, `VALIDATION_RESULT`, `AUDIT_EXPORT`, `TRACE_FRAGMENT`, `PROPOSAL`, `REVIEW_FEEDBACK`, `GENERATED_CONTENT`.

Custom `artifactType` MUST use namespace prefixes (§2b.7). Bare `x-*` artifact types are permitted only at lower profiles; at `OACP-Governed`+, they are rejected with `ARTIFACT_TYPE_NAMESPACE_INVALID`.

### §2c.4 References in higher-level fields

Fields are typed by Reference subtype:

| Field | Reference subtype |
|---|---|
| `inputRef` (tool) | `ArtifactRef` (`TOOL_INPUT`) |
| `resultRef` (tool) | `ArtifactRef` (`TOOL_RESULT`) |
| `sanitizedPayloadRef` (broker) | `ArtifactRef` (`SANITIZED_PAYLOAD`) |
| `recommendationRef` (proposal) | `ArtifactRef` (`RECOMMENDATION`) |
| `evidenceReportRef` (proposal) | `ArtifactRef` (`EVIDENCE_REPORT`) |
| `validationResultRef` | `ArtifactRef` (`VALIDATION_RESULT`) |
| `manifestId` | `RecordRef` (`CAPABILITY_MANIFEST`) |
| `bindingId` | `RecordRef` (`BINDING_RECORD`) |
| `policySnapshotId` | `RecordRef` (`POLICY_SNAPSHOT`) |
| `authorizationDecision.decisionRef` | `DecisionRef` (`AUTHORIZATION`) |
| `brokerDecisionId` | `DecisionRef` (`BROKER`) |
| `causationId`, `causationIds[]` | `MessageRef` |
| `responseTo.requestMessageId` | `MessageRef` |
| `replay.originalMessageId` | `MessageRef` |
| `auditRef`, `auditRefs[]` | `AuditRef` |

### §2c.5 Hash format

Hash fields use algorithm-prefixed format: `<algorithm>:<encoded-digest>`. Reserved algorithms:

| Algorithm | Minimum profile | Status |
|---|---|---|
| `sha256` | `OACP-Operational` | Default |
| `sha512` | `OACP-Governed` | Recommended at Governed+ |
| `blake3` | Optional | Reserved |

Format violations: `ARTIFACT_HASH_FORMAT_INVALID`, `ARTIFACT_HASH_ALGORITHM_UNSUPPORTED`.

### §2c.6 Tenant-scoped resolution

Every `ArtifactRef` MUST carry `tenantId`. Receivers verify the artifact's tenant matches the resolving message's `scope.tenantId` (§2d). Cross-tenant resolution is rejected with `ARTIFACT_TENANT_VIOLATION`.

### §2c.6a Trusted URI resolution

URI schemes referenced by `ArtifactRef.uri` are implementation-defined. To prevent the protocol from becoming an SSRF or data-exfiltration surface:

1. Receivers MUST resolve `ArtifactRef.uri` only through configured trusted resolvers.
2. Receivers MUST NOT dereference arbitrary URI schemes unless policy explicitly permits.
3. An untrusted-resolver URI is rejected with `ARTIFACT_URI_RESOLVER_UNTRUSTED`.
4. The trusted-resolver set is profile-influenced: lower profiles MAY accept any configured scheme; `OACP-Governed`+ SHOULD restrict to schemes whose resolvers can enforce tenant scope, hash verification, and expiry.

This rule mirrors §2b.6 rule 7: no resolution surface is "any URI." Both schema and artifact resolution flow through trusted local infrastructure.

### §2c.7 Expiry

Artifacts past `expiresAt` are non-resolvable for new operations. Receivers MUST reject with `ARTIFACT_EXPIRED`. Replay artifacts past expiry MAY be flagged with `ARTIFACT_EXPIRED_REPLAY_ONLY` (non-blocking) per §3c.

### §2c.8 Externalization rule

Senders SHOULD externalize any single payload field whose encoded size exceeds 30% of the active `maxEnvelopeBytes`. Receivers MAY enforce stricter thresholds. Receivers MUST reject envelopes whose total encoded size exceeds `maxEnvelopeBytes` with `PAYLOAD_TOO_LARGE` (or `ENVELOPE_TOO_LARGE`) before further validation.

### §2c.9 Per-profile `maxEnvelopeBytes`

Recommended (non-normative) defaults:

| Profile | `maxEnvelopeBytes` |
|---|---|
| `OACP-Core` | 1 MiB |
| `OACP-Operational` | 256 KiB |
| `OACP-Governed` | 64 KiB |
| `OACP-Regulated` | 32 KiB |

Deployments configure within profile guidance; out-of-range values are configuration-time errors.

### §2c.10 Reserved error codes (§2c)

```
ARTIFACT_NOT_RESOLVABLE              — uri cannot be resolved
ARTIFACT_HASH_MISMATCH               — content does not match declared hash
ARTIFACT_HASH_FORMAT_INVALID         — hash field malformed
ARTIFACT_HASH_ALGORITHM_UNSUPPORTED  — algorithm not supported
ARTIFACT_TENANT_VIOLATION            — artifact tenant outside authorized scope
ARTIFACT_EXPIRED                     — past expiresAt
ARTIFACT_EXPIRED_REPLAY_ONLY         — non-blocking; flagged in replay
ARTIFACT_TYPE_MISMATCH               — type doesn't match expected
ARTIFACT_TYPE_NAMESPACE_INVALID      — bare x-* type at Governed/Regulated
ARTIFACT_SIZE_EXCEEDED               — sizeBytes exceeds limit
ARTIFACT_URI_SCHEME_UNSUPPORTED      — scheme not supported
ARTIFACT_URI_RESOLVER_UNTRUSTED      — scheme not in trusted-resolver set
PAYLOAD_TOO_LARGE                    — envelope exceeds maxEnvelopeBytes
ENVELOPE_TOO_LARGE                   — synonym for PAYLOAD_TOO_LARGE
EXTERNALIZATION_REQUIRED             — payload field MUST be externalized
```

---

## §2d. Tenant-scoped references and cross-tenant isolation

### §2d.1 The principle

> **Every Reference resolved by an OACP receiver MUST be within the receiver's authorized tenant (and workspace, where configured) scope. Cross-tenant references require explicit deployment policy with auditable artifacts.**

This applies uniformly to all Reference subtypes: `ArtifactRef`, `MessageRef`, `RecordRef`, `DecisionRef`, `AuditRef`. The rule is a protocol-wide isolation invariant, not an artifact-specific safety property.

### §2d.2 The receiver's authorized scope

A receiver operates with a configured authorized scope:

```
authorizedScope = {
  tenantIds: ["tenant_001"],
  workspaceIds: ["workspace_001"],
  systemScopeAuthorized: false,
  crossTenantPolicy: null
}
```

Receivers MAY be authorized for multiple tenants. Each individual *message* belongs to exactly one tenant via `scope.tenantId`. The receiver's authorized scope determines which tenants' messages it may process.

### §2d.2a Scope types

The `scope` block carries a `scopeType` discriminator:

```json
"scope": {
  "scopeType": "TENANT",
  "tenantId": "tenant_001",
  "workspaceId": "workspace_001",
  "domainId": "sample-domain",
  "taskType": "SAMPLE_TASK",
  "environment": "development"
}
```

Allowed `scopeType` values:

| Value | Meaning | `tenantId` requirement |
|---|---|---|
| `TENANT` | Tenant-scoped (default) | Required |
| `WORKSPACE` | Workspace-scoped within a tenant | `tenantId` and `workspaceId` both required |
| `SYSTEM` | Platform/runtime control-plane message | `tenantId` MUST be absent |
| `CROSS_TENANT` | Authorized cross-tenant flow | `tenantId` required (sender's); `crossTenantPolicyRef` required |

Examples of legitimate `SYSTEM` messages: manifest registry updates, runtime health events, global policy publications, system-level control messages, cross-tenant audit aggregation initiated by the platform itself.

Rules:

1. `SYSTEM`-scoped messages MUST NOT reference tenant-scoped artifacts unless an explicit `crossTenantPolicyRef` or system-audit policy authorises.
2. Receivers MUST be explicitly authorized for system scope (`systemScopeAuthorized: true`).
3. `SYSTEM` messages flowing to a non-system-authorized receiver are rejected with `SYSTEM_SCOPE_NOT_AUTHORIZED`.
4. `CROSS_TENANT` messages without `crossTenantPolicyRef` are rejected with `CROSS_TENANT_POLICY_REF_MISSING`.

### §2d.3 The four enforcement points

Tenant scope is checked at four points during message processing:

1. **At message receipt** — envelope `scope.tenantId` MUST be in the receiver's authorized scope. Otherwise: `MESSAGE_TENANT_VIOLATION`.
2. **At identity verification** — verified identity's authorized tenant set MUST include `scope.tenantId`. Otherwise: `IDENTITY_TENANT_VIOLATION`.
3. **At reference resolution** — every Reference carries `tenantId` (implicit or explicit). Resolution MUST verify the reference's tenant matches the message's `scope.tenantId`. Otherwise: appropriate `*_TENANT_VIOLATION`.
4. **At binding scope check** — the binding's `bindingScope.tenantId` MUST match `scope.tenantId` (§3b.7).

### §2d.4 Tenant scope on each Reference subtype

| Subtype | Tenant carried | Required at |
|---|---|---|
| `ArtifactRef` | `tenantId` field | `OACP-Governed`+ |
| `MessageRef` | `tenantId` field; bare-string `messageId` implicitly scoped to current message tenant | When ref crosses runs, full `MessageRef` with `tenantId` required |
| `RecordRef` | `tenantId` field (or `scopeType: GLOBAL`) | When ref is to a tenant-scoped record |
| `DecisionRef` | `tenantId` field | All decisions are tenant-scoped |
| `AuditRef` | `tenantId` field | All audit entries are tenant-scoped |

**`RecordRef` scope discrimination:** every `RecordRef` MUST declare `scopeType`. `TENANT` and `WORKSPACE` records require tenant/workspace authorization. `GLOBAL` records require receiver authorization for global registry access (typically deployment-time configuration). A `GLOBAL` record without receiver authorization is rejected with `GLOBAL_RECORD_ACCESS_DENIED`.

Bare-string forms MAY be tenant-scoped implicitly when interpreted in the context of the current message's `scope.tenantId`. When ambiguous, full Reference subtype with explicit `tenantId` is required.

### §2d.5 Workspace as sub-tenant scope

Some deployments use `workspaceId` for finer isolation:

```
tenant_001
├── workspace_001 (sales)
├── workspace_002 (engineering)
└── workspace_003 (legal)
```

When workspace isolation is configured:

1. References MAY carry `workspaceId` in addition to `tenantId`.
2. Receivers' authorized scope MAY restrict to specific workspaces.
3. Cross-workspace references within the same tenant follow cross-tenant rules — explicit policy required.
4. Error codes parallel tenant violations: `MESSAGE_WORKSPACE_VIOLATION`, etc.

Workspace isolation is optional. **However, when a message carries `workspaceId` but the receiver does not have workspace isolation configured, the receiver MAY ignore `workspaceId` for authorization decisions but MUST preserve it in audit records.** Silently dropping `workspaceId` from forwarded messages or audit chains is a protocol violation: an intermediary stripping `workspaceId` violates `EXTENSION_PRESERVATION_VIOLATION` semantics applied to scope context.

### §2d.6 Cross-tenant references — explicit policy

Cross-tenant flows reference a registered policy via `crossTenantPolicyRef`:

```json
"crossTenantPolicyRef": {
  "recordId": "ctp_regulated_oversight_v1",
  "recordType": "CROSS_TENANT_POLICY",
  "registry": "policy-registry",
  "scopeType": "GLOBAL"
}
```

The full `crossTenantPolicy` artifact lives in the policy registry:

```json
{
  "policyId": "ctp_regulated_oversight_v1",
  "policyVersion": "1.0.0",
  "fromTenants": ["tenant_001", "tenant_002"],
  "toTenants": ["regulator-tenant"],
  "allowedReferenceTypes": ["AUDIT_RECORD", "PROPOSAL"],
  "auditRequired": true,
  "policySnapshotRef": "policy-snapshot-ctp-v1",
  "validFrom": "2026-01-01T00:00:00Z",
  "validUntil": "2027-01-01T00:00:00Z"
}
```

Rules:

1. Cross-tenant references without an active `crossTenantPolicy` are rejected with `CROSS_TENANT_REFERENCE_FORBIDDEN`.
2. Policies are registered ahead of time; mid-flow policy creation does not authorize prior cross-tenant flows.
3. Cross-tenant flows MUST be audit-recorded against both source and destination tenant audit chains.
4. The policy specifies which Reference subtypes are permitted to cross.
5. Cross-tenant references MUST carry full Reference objects with explicit `tenantId`.
6. Messages MUST reference cross-tenant policies, not embed them. An embedded `crossTenantPolicy` block is rejected with `CROSS_TENANT_POLICY_EMBEDDED`.

### §2d.6a Reference-scope compatibility (general rule)

All references inside a message MUST be scope-compatible with the message's `scope.scopeType` and tenant/workspace context. The compatibility table:

| Message `scopeType` | Permitted reference scopes |
|---|---|
| `TENANT` | Same-tenant references; `GLOBAL` records when receiver authorized |
| `WORKSPACE` | Same-workspace references; same-tenant cross-workspace only via explicit policy; `GLOBAL` records |
| `SYSTEM` | `GLOBAL` records; system-audit references; tenant-scoped only with `crossTenantPolicyRef` |
| `CROSS_TENANT` | References to tenants in resolved `crossTenantPolicy.fromTenants ∪ toTenants`; `GLOBAL` records |

Receivers verify reference-scope compatibility as part of gate 1.5. Violations: `REFERENCE_SCOPE_INCOMPATIBLE`.

This unified rule applies to all Reference subtypes; per-subtype tenant checks (§2d.4) implement it.

### §2d.7 Tenant authorization on the verification block

Identity claims (§3a.2) identify the participant. Tenant authorization — *which tenants this verified identity may operate under* — is **verified scope, not a sender-authored claim**. It belongs in the verification block (§3a.3), not in `identity`.

```json
"verification": {
  "verified": true,
  "verifiedBy": "runtime-adapter-001",
  "verifiedAt": "2026-05-05T17:00:00Z",
  "verificationMethod": "TOKEN_INTROSPECTION",
  "verificationRef": "verify_001",
  "tenantScope": {
    "authorizedTenants": ["tenant_001"],
    "authorizedWorkspaces": ["workspace_001"],
    "source": "credential-claims"
  }
}
```

| Field | Meaning |
|---|---|
| `tenantScope.authorizedTenants` | Tenants the verified identity is authorized for |
| `tenantScope.authorizedWorkspaces` | Workspaces (when isolation configured); empty = no restriction |
| `tenantScope.source` | How scope was derived: `credential-claims`, `policy-lookup`, `static-config` |

Receivers MUST verify that the message's `scope.tenantId` is in `verification.tenantScope.authorizedTenants` before proceeding past gate 1.5. The verification trail (§3a.3) preserves tenant-scope verification records across multi-hop runs.

**Required at:** `OACP-Governed`+. Lower profiles MAY use static configuration in lieu of an explicit `tenantScope` block.

### §2d.8 Placement in the processing gate

Step 1.5 (between identity verification and authority verification):

> 1. Verify identity.
> 1.5. **Verify tenant scope.** Message `scope.tenantId` is in receiver's authorized scope. Identity's `authorizedTenants` includes `scope.tenantId`. Workspace scope verified if configured.
> 2. Verify participant authority.
> ...

Tenant scope verification is **not skippable** at any profile when tenant scoping is configured.

### §2d.9 Audit and visibility

1. Tenant-violation rejections MUST be audit-recorded with `messageId`, claimed tenant, receiver's authorized scope, violation point.
2. Tenant violations from external participants SHOULD trigger deployment-level alerts.
3. Cross-tenant policy permits MUST be audit-recorded on both source and destination tenant audit chains.

### §2d.10 Reserved error codes (§2d)

```
MESSAGE_TENANT_VIOLATION             — message scope.tenantId outside receiver scope
MESSAGE_WORKSPACE_VIOLATION          — message workspaceId outside receiver scope
IDENTITY_TENANT_VIOLATION            — verification.tenantScope.authorizedTenants does not include scope.tenantId
ARTIFACT_TENANT_VIOLATION            — see §2c.10
ARTIFACT_WORKSPACE_VIOLATION         — artifact's workspaceId outside scope
RECORD_TENANT_VIOLATION              — record outside scope
DECISION_TENANT_VIOLATION            — decision artifact outside scope
AUDIT_TENANT_VIOLATION               — audit reference outside scope
MESSAGE_REF_TENANT_VIOLATION         — referenced message in different tenant
CROSS_TENANT_REFERENCE_FORBIDDEN     — cross-tenant ref without active policy
CROSS_TENANT_POLICY_NOT_APPLICABLE   — policy doesn't authorize this ref type or these tenants
CROSS_TENANT_POLICY_REF_MISSING      — CROSS_TENANT scope without crossTenantPolicyRef
CROSS_TENANT_POLICY_EMBEDDED         — message embeds policy instead of referencing
CROSS_WORKSPACE_REFERENCE_FORBIDDEN  — cross-workspace ref without explicit policy
SYSTEM_SCOPE_NOT_AUTHORIZED          — SYSTEM message to non-authorized receiver
GLOBAL_RECORD_ACCESS_DENIED          — receiver not authorized for GLOBAL record
REFERENCE_SCOPE_INCOMPATIBLE         — reference violates message-scope compatibility
```

### §2d.11 Conformance profile interactions

| Profile | Tenant isolation behavior |
|---|---|
| `OACP-Core` | Enforced if configured; `tenantId` may be absent in untenanted deployments |
| `OACP-Operational` | `scope.tenantId` required when receiver has authorized scope; identity-tenant binding required |
| `OACP-Governed` | All §2d rules; full Reference objects for cross-run refs; `tenantScope` required on verification |
| `OACP-Regulated` | All rules; cross-tenant policies cryptographically signed; tenant-violation audit records cryptographically immutable |

### §2d.12 Out of scope for v0.1

- Multi-tenancy primitives below workspace.
- Tenant federation across deployments.
- Tenant-scoped policy snapshots per tenant.
- Dynamic tenant authorization changes mid-run.
- Tenant migration semantics.

---

## §3a. Participant identity and authorization

### §3a.1 The from block

Every OACP message identifies its sender through a `from` block:

```json
"from": {
  "identity": {
    "assurance": "AUTHENTICATED",
    "subject": "principal.agent.classifier",
    "credentialRef": "cred_001",
    "credentialKind": "RUNTIME_TOKEN",
    "issuedAt": "2026-05-05T16:00:00Z",
    "expiresAt": "2026-05-05T22:00:00Z",
    "issuedBy": "runtime-adapter-001",
    "actingOnBehalfOf": null
  },
  "verification": {
    "verified": true,
    "verifiedBy": "runtime-adapter-001",
    "verifiedAt": "2026-05-05T17:00:00Z",
    "verificationMethod": "TOKEN_INTROSPECTION",
    "verificationRef": "verify_001",
    "tenantScope": {
      "authorizedTenants": ["tenant_001"],
      "authorizedWorkspaces": ["workspace_001"],
      "source": "credential-claims"
    }
  },
  "verificationTrail": [],
  "participantId": "agent.classifier",
  "participantType": "AGENT"
},
"authorizationDecision": {
  "decisionRef": "authz_decision_001",
  "decidedBy": "policy-engine-001",
  "decidedAt": "2026-05-05T17:00:00Z",
  "scope": {
    "messageType": "PROPOSAL_EMITTED",
    "bindingId": "bind_001",
    "runId": "run_001",
    "tenantId": "tenant_001"
  },
  "result": "ALLOW"
}
```

### §3a.2 Identity claim

The `identity` block is the participant's claim about who they are. **It is sender-authored and immutable.**

| Field | Required | Meaning |
|---|---|---|
| `assurance` | Yes | One of `DECLARED`, `AUTHENTICATED`, `ATTESTED` |
| `subject` | Yes | The participant's claimed identity |
| `credentialRef` | At Authenticated+ | Reference to the credential used |
| `credentialKind` | At Authenticated+ | Reserved enum |
| `issuedAt` | At Authenticated+ | Credential issuance time |
| `expiresAt` | At Authenticated+ | Credential expiration |
| `issuedBy` | At Authenticated+ | Credential issuer identifier |
| `actingOnBehalfOf` | Optional | Reserved placeholder for delegation (out of v0.1 scope) |

Reserved `credentialKind` values: `RUNTIME_TOKEN`, `BEARER_TOKEN`, `MTLS_CERTIFICATE`, `OAUTH_TOKEN`, `SIGNED_JWT`, `RUNTIME_KEY`, `WORKLOAD_IDENTITY`, `SERVICE_ACCOUNT_TOKEN`.

The identity claim is part of the envelope hash (§3e). Modification in transit invalidates `envelopeHash`.

### §3a.3 Verification block

The `verification` block records the **receiver-authored** result of verifying the identity claim. It is **appended, not overwritten** — multi-hop messages accumulate verification entries in `verificationTrail`.

| Field | Required | Meaning |
|---|---|---|
| `verified` | Yes | Boolean — verification succeeded |
| `verifiedBy` | Yes | Verifier identifier |
| `verifiedAt` | Yes | Verification timestamp |
| `verificationMethod` | At Authenticated+ | How verification was performed |
| `verificationRef` | Optional | Reference to verification record |
| `tenantScope` | At Governed+ | Per §2d.7 |

Reserved `verificationMethod` values: `TOKEN_INTROSPECTION`, `MTLS_VERIFY`, `JWT_VERIFY`, `RUNTIME_INTERNAL`, `OAUTH_VERIFY`, `STATIC_TRUST`.

Each hop's verification record is appended to `verificationTrail` rather than replacing the prior. The full trail enables auditors to reconstruct multi-hop verification history.

### §3a.4 Three assurance levels

| Level | Definition | Typical use |
|---|---|---|
| `DECLARED` | Participant claims an identity; no verification | `OACP-Core` trusted in-process |
| `AUTHENTICATED` | Identity backed by verifiable credential | `OACP-Operational`+ |
| `ATTESTED` | Identity backed by cryptographic attestation (signed credentials, runtime attestation) | `OACP-Regulated` |

### §3a.5 Verification freshness

Verification has freshness semantics independent of credential expiry:

- `verification.verifiedAt` MUST be present at `OACP-Operational`+.
- Freshness window is profile-driven; a verification older than the configured window triggers `IDENTITY_VERIFICATION_STALE`.
- Re-verification is cheap; producers SHOULD re-verify when crossing trust boundaries.

A failure to verify a message identity at any hop emits `IDENTITY_REVERIFICATION_FAILED`.

### §3a.6 AuthorizationDecision

The `authorizationDecision` block is **separate from identity**. It records the decision made by an authorization engine (typically Aegis in the Ojas reference deployment) about whether this specific message is permitted.

```json
"authorizationDecision": {
  "decisionRef": "authz_decision_001",
  "decidedBy": "policy-engine-001",
  "decidedAt": "2026-05-05T17:00:00Z",
  "scope": {
    "messageType": "PROPOSAL_EMITTED",
    "bindingId": "bind_001",
    "runId": "run_001",
    "tenantId": "tenant_001"
  },
  "result": "ALLOW",
  "policySnapshotId": "policy-snapshot-001"
}
```

Reserved `result` values: `ALLOW`, `DENY`, `SANITIZE`, `APPROVAL_NEEDED`, `QUARANTINE`, `CANCEL`.

| Field | Required | Meaning |
|---|---|---|
| `decisionRef` | Yes | Reference to the decision record |
| `decidedBy` | Yes | Authorization engine identifier |
| `decidedAt` | Yes | Decision timestamp |
| `scope` | Yes | Scope-bound: per-message-type, per-binding, per-run, per-tenant |
| `result` | Yes | One of the reserved values |
| `policySnapshotId` | At Governed+ | Policy snapshot under which the decision was made |

**Crucially, `authorizationDecision` is required at `OACP-Governed`+ for capability-scoped messages.** Lower profiles MAY omit it. The decision is consulted by the receiver at gate step 4; absence at the required profile triggers `AUTHORIZATION_DECISION_MISSING`.

The decision is **scope-bound** — it authorizes a specific message type, binding, run, tenant. Reusing a decision outside its scope is rejected with `AUTHORIZATION_SCOPE_VIOLATION`.

A revoked decision (per `decisionStatus` lifecycle, §13.6b) triggers `DECISION_AUTHORITY_REVOKED`.

### §3a.7 Identity, verification, and tenant scope are distinct

| Concept | What it is | Authored by |
|---|---|---|
| Identity | Who the participant claims to be | Sender |
| Verification | Whether the receiver believes the identity | Receiver |
| Tenant authorization | Which tenants the verified identity may operate under | Receiver (`verification.tenantScope`) |
| Authorization decision | Whether this specific message is permitted | Authorization engine (separate block) |
| Capability binding | Which capability this participant is bound to (§3b) | Runtime/Supervisor |

These are deliberately separate. Conflating them is a common implementation error.

### §3a.8 Authority matrix

The authority matrix governs which `participantType` may emit which `messageKind`. This is the protocol-level authority check, separate from tenant authorization (§2d) and binding (§3b).

| `participantType` | Permitted `messageKind` |
|---|---|
| `AGENT` | `EVENT`, `RESPONSE`, `PROPOSAL`, `TOOL_CALL` (request only), `GATE_RESULT` (when authorized) |
| `SUPERVISOR` | All `AGENT` permissions plus: `BINDING` lifecycle, `REVIEW_REQUESTED`, supersession decisions |
| `RUNTIME` | All permissions plus: `BINDING` creation, `RuntimeDecisionRef`, replay authorization, lifecycle events |
| `TOOL_BROKER` | `TOOL_CALL` (state transitions, results), broker decisions |
| `GATE_RUNNER` | `GATE_RESULT` |
| `EVIDENCE_COLLECTOR` | `EVIDENCE_REPORT` artifacts (referenced, not as message bodies) |
| `EXTERNAL_SYSTEM` | `EVENT` (notification only) |
| `EXTERNAL_REVIEWER` | `REVIEW_RESPONDED`, `REVIEW_MORE_INFORMATION_REQUIRED`, `ExternalDecisionRef` (when separately authorized) |

Mismatch: `AUTHORITY_VIOLATION`.

### §3a.9 actingOnBehalfOf

Reserved field for participant-on-behalf-of semantics (a runtime acting on behalf of an end user, a supervisor acting on behalf of an absent reviewer). **Out of v0.1 scope.** Implementations MAY include the field but MUST NOT rely on it for authorization decisions.

### §3a.10 Identity placement in the processing gate

Step 1: verify identity. Step 1.5: tenant scope. Step 2: participant authority. Step 4: authorization decision. The four are sequential; failure at any one halts processing.

### §3a.11 Identity immutability

The `identity` block is part of the envelope hash. It cannot be modified in transit without invalidating the envelope. Tampering attempts are detected at the next receiver via `ENVELOPE_HASH_INVALID`.

`verificationTrail` is the only `from`-block field that is appended in transit; appends MUST NOT modify prior trail entries.

### §3a.12 Clock skew

Receivers tolerate clock skew up to `maxClockSkewSeconds` (configuration-time per profile). Distinguish:

| Concern | Field | Rule |
|---|---|---|
| Credential freshness | `identity.expiresAt` | Receivers reject if past |
| Verification freshness | `verification.verifiedAt` | Receivers re-verify if older than freshness window |
| Message age | `createdAt` | Receivers reject if older than `maxMessageAgeSeconds` (§3e) |
| Clock skew tolerance | `maxClockSkewSeconds` | Tolerance for timestamp differences |

Excessive skew: `CLOCK_SKEW_EXCEEDED`.

### §3a.13 Escalation and review states

When an agent's emission requires expanded authority, an `ESCALATION_REQUESTED` message routes through the runtime/supervisor (per Tier 1 escalation flow). When external review is needed, a `REVIEW_REQUESTED` message routes via §11a. Both flows have TTL discipline; missing TTL triggers `ESCALATION_TTL_REQUIRED` or `REVIEW_TTL_REQUIRED`.

### §3a.14 Reserved error codes (§3a)

```
IDENTITY_MISSING                     — identity block absent at required profile
IDENTITY_ASSURANCE_INSUFFICIENT      — DECLARED used at Operational+
IDENTITY_CREDENTIAL_INVALID          — credential failed validation
IDENTITY_CREDENTIAL_EXPIRED          — credential past expiresAt
IDENTITY_CREDENTIAL_KIND_UNSUPPORTED — credentialKind not supported
IDENTITY_REVERIFICATION_FAILED       — re-verification at hop failed
IDENTITY_VERIFICATION_STALE          — verifiedAt older than freshness window
IDENTITY_TENANT_VIOLATION            — see §2d.10
CLOCK_SKEW_EXCEEDED                  — timestamp difference exceeds tolerance
AUTHORITY_VIOLATION                  — participantType cannot emit this messageKind
AUTHORIZATION_DENIED                 — authorizationDecision.result = DENY
AUTHORIZATION_DECISION_MISSING       — decision required but absent
AUTHORIZATION_SCOPE_VIOLATION        — decision used outside its scope
HANDOFF_ASSURANCE_VIOLATION          — handoff between participants violates assurance gradient
```

---

## §3b. Capability manifest and binding authority

### §3b.1 Two distinct concepts

| Concept | Question | Authored by |
|---|---|---|
| Capability manifest | What can this participant do **in principle**? | Manifest authority (typically platform-side governance) |
| Binding | Was this participant **selected** for this capability **in this run**? | `RUNTIME` or authorized `SUPERVISOR` |

A manifest is a declared capability profile. A binding is a runtime allocation of a manifest's capabilities to a specific run.

### §3b.2 CapabilityManifest

A `CapabilityManifest` is a registered artifact:

```json
{
  "manifestId": "manifest.agent.classifier.v1",
  "manifestVersion": "1.0.0",
  "subject": "principal.agent.classifier",
  "registeredAt": "2026-04-01T00:00:00Z",
  "registeredBy": "platform-governance",
  "lifecycleStatus": "ACTIVE",
  "capabilities": [
    {
      "capabilityId": "cap.classify_pdf",
      "inputPayloadTypes": ["pdf-document-v1"],
      "outputPayloadTypes": ["classification-result-v1"],
      "emitsMessageTypes": ["PROPOSAL_EMITTED", "EVIDENCE_REPORT_EMITTED"],
      "allowedHandoffs": ["agent.layout-extractor"],
      "allowedTools": ["read_source_file", "extract_layout"],
      "riskClass": "MEDIUM",
      "sideEffectClass": "READ_ONLY"
    }
  ],
  "tenantId": "tenant_001"
}
```

| Field | Meaning |
|---|---|
| `manifestId` | Stable identifier; immutable |
| `manifestVersion` | Semver per §27.4 |
| `subject` | The principal this manifest applies to |
| `lifecycleStatus` | Per §27.8 unified lifecycle |
| `capabilities[]` | Declared capabilities; each has its own scope |

### §3b.3 Manifest registry

Manifests live in a manifest registry. The registry is a control-plane component, not a runtime component.

**Manifest immutability:** once registered, a manifest is **immutable**. New versions produce new `manifestId` (or new version segment). Existing bindings reference specific manifest versions and are unaffected by new manifest versions of the same subject (§27.4).

Lifecycle states (§27.8 unified model):

| State | New references | Existing references |
|---|---|---|
| `ACTIVE` | Permitted | Operational |
| `DEPRECATED` | Permitted (audit warning) | Operational |
| `DEACTIVATED` | Rejected | Operational |
| `BLOCKED` | Rejected | **Force-revoked** |
| `RETIRED` | Rejected | Rejected |
| `REVOKED` | Rejected | Rejected (retroactively invalid) |

`MANIFEST_NOT_REGISTERED`, `MANIFEST_RETIRED`, `MANIFEST_REVOKED` reject at receive time.

### §3b.4 Binding artifact

A binding is created by `RUNTIME` or authorized `SUPERVISOR` to allocate a manifest's capabilities to a run:

```json
{
  "bindingId": "bind_001",
  "bindingVersion": "1.0.0",
  "manifestId": "manifest.agent.classifier.v1",
  "capabilityId": "cap.classify_pdf",
  "issuedBy": {
    "participantId": "runtime.ojas",
    "participantType": "RUNTIME"
  },
  "issuedAt": "2026-05-05T16:30:00Z",
  "bindingScope": {
    "runId": "run_001",
    "tenantId": "tenant_001",
    "workspaceId": "workspace_001",
    "messageTypes": ["PROPOSAL_EMITTED", "EVIDENCE_REPORT_EMITTED"]
  },
  "bindingValidityFrom": "2026-05-05T16:30:00Z",
  "bindingValidityUntil": "2026-05-05T18:30:00Z",
  "bindingState": "BOUND"
}
```

**Authority rule:** only `RUNTIME` or authorized `SUPERVISOR` may issue bindings. An `AGENT` cannot self-bind. `BINDING_AUTHORITY_INVALID` rejects unauthorized binding emissions.

### §3b.5 Binding lifecycle

```
REQUESTED                            participant requests a binding
    ↓
MATCHED                              runtime found a manifest match
    ↓
BOUND                                binding active in registry
    ↓
ACTIVE                               binding currently in use by run
    ↓
    ├── SUSPENDED                    runtime paused the binding
    │       └── (resume → ACTIVE; or → REVOKED)
    │
    ├── REVOKED                      explicitly revoked
    ├── EXPIRED                      bindingValidityUntil elapsed
    └── FAILED                       binding-creation or activation failed
```

State transitions emit corresponding messages: `BINDING_REQUESTED`, `BINDING_MATCHED`, `BINDING_BOUND`, `BINDING_ACTIVE`, `BINDING_SUSPENDED`, `BINDING_REVOKED`, `BINDING_EXPIRED`, `BINDING_FAILED`.

### §3b.6 Binding evaluation

Bindings are evaluated **at the receiver's evaluation time**, not at the sender's emission time. Exception: in `replay` with `ORIGINAL_POLICY` mode (§3c), the binding is evaluated against the original message's emission time policy.

Bindings in terminal states (`REVOKED`, `EXPIRED`, `FAILED`) are non-usable. Messages claiming a binding in a terminal state are rejected with `BINDING_INVALID`.

### §3b.7 Binding consistency rules

1. The binding's `manifestId` MUST point to an `ACTIVE` or `DEPRECATED` manifest at binding creation.
2. The binding's `capabilityId` MUST match a declared capability in the manifest.
3. The message's `messageType` MUST be in the binding's `bindingScope.messageTypes`.
4. The message's `runId` MUST match `bindingScope.runId`.
5. The binding's tenant scope MUST match the message's `scope.tenantId` (cross-references §2d.4).
6. The message's `createdAt` MUST be within `bindingValidityFrom..bindingValidityUntil`.
7. Workspace scope: when configured, message workspace MUST match binding workspace.

Violations: `BINDING_SCOPE_MISMATCH`, `BINDING_CAPABILITY_INVALID`, `BINDING_VALIDITY_VIOLATION`.

### §3b.8 Capability negotiation (optional)

Optional capability negotiation handshake (deferred details — present in v0.1 only as a placeholder):

```
CAPABILITY_NEGOTIATION_REQUESTED     participant proposes capabilities
    ↓
CAPABILITY_NEGOTIATION_RESPONDED     runtime accepts or counter-proposes
    ↓
CAPABILITY_NEGOTIATION_COMPLETE      bindings created based on negotiation
```

The `OACP-Capability` capability profile mandates the negotiation contract. v0.1 does not require negotiation as a precondition for binding; deployments may pre-create bindings without negotiation.

### §3b.9 Reserved error codes (§3b)

```
MANIFEST_NOT_REGISTERED              — manifestId not in registry
MANIFEST_DEPRECATED                  — non-blocking warning
MANIFEST_DEACTIVATED                 — receive-time rejection
MANIFEST_BLOCKED                     — receive-time rejection
MANIFEST_RETIRED                     — receive-time rejection
MANIFEST_REVOKED                     — security-grade rejection
CAPABILITY_NOT_DECLARED              — capabilityId not in manifest
BINDING_INVALID                      — binding state terminal or malformed
BINDING_SCOPE_MISMATCH               — binding scope doesn't match message
BINDING_AUTHORITY_INVALID            — non-RUNTIME/SUPERVISOR issued binding
BINDING_CAPABILITY_INVALID           — capabilityId doesn't match manifest
BINDING_VALIDITY_VIOLATION           — message outside binding validity window
BINDING_NOT_FOUND                    — bindingId doesn't resolve
```

### §3b.10 Conformance profile interactions

| Profile | Capability/binding behavior |
|---|---|
| `OACP-Core` | Capability/binding optional |
| `OACP-Operational` | Manifest registry recommended; binding optional |
| `OACP-Capability` | Manifest registration required; capabilities declared |
| `OACP-Binding` | Full binding lifecycle required for capability-scoped messages |
| `OACP-Governed` | Binding required; binding evaluation enforced at receive time |
| `OACP-Regulated` | Bindings cryptographically signed; manifest registry signature-protected |

---

## §3c. Idempotency, retry, replay, and deduplication

### §3c.1 Three distinct concepts

| Concept | Field | Question |
|---|---|---|
| Message identity | `messageId` | Is this the same wire-level message? |
| Operation identity | `idempotencyKey` | Is this the same logical operation? |
| Causal predecessor | `causationId` | What caused this message? |

These are different. A retry uses a new `messageId` but the same `idempotencyKey`. A replay carries the original `messageId` plus `replay` metadata. A causally-related response has `causationId` pointing to its trigger.

### §3c.2 messageId

Stable identifier for a single emission. **One emission, N possible deliveries** (transports may deliver multiple times).

```json
"messageId": "msg_001"
```

Receivers MAY use `messageId` for transport-level deduplication within a `dedupeWindowSeconds`.

### §3c.3 idempotencyKey

Identifies a logical operation across retries:

```json
"idempotencyKey": "op_classify_doc_xyz_v1",
"idempotencyScope": "RUN"
```

Reserved `idempotencyScope` values:

| Scope | Meaning |
|---|---|
| `RUN` | Operation idempotent within a run |
| `TENANT` | Operation idempotent across the tenant |
| `WORKSPACE` | Operation idempotent within a workspace |
| `GLOBAL` | Operation idempotent across all tenants (rare; SYSTEM scope) |

A retry produces a new `messageId` but reuses the same `idempotencyKey` plus `idempotencyScope`. Receivers seeing the same key+scope+payload-hash combination MAY deduplicate.

### §3c.4 causationId

References the message that caused this one (§3d for full causation semantics):

```json
"causation": {
  "causationId": "msg_predecessor",
  "causationType": "LINEAR"
}
```

### §3c.5 Deduplication

When a receiver sees the same `idempotencyKey` + `idempotencyScope` within `dedupeWindowSeconds`:

| Payload hash | Behavior |
|---|---|
| Same | Deduplicate — return prior result if cached, or treat as redelivery |
| Different | Reject with `IDEMPOTENCY_CONFLICT` |

Same key, different payload, indicates an error. Either the sender reused a key incorrectly, or a payload tampering occurred. The receiver MUST NOT silently accept the new payload.

`IDEMPOTENCY_CONFLICT` may be overridden by `correctionMetadata` indicating a deliberate correction (rare; requires runtime authority).

### §3c.6 Retry and delivery semantics

```json
"delivery": {
  "deliveryMode": "AT_LEAST_ONCE",
  "retryAttempt": 2,
  "maxRetryAttempts": 5
},
"replayable": true
```

Reserved `deliveryMode` values:

| Mode | Meaning |
|---|---|
| `AT_MOST_ONCE` | Best-effort; loss possible |
| `AT_LEAST_ONCE` | Retried until ack; duplicates possible |
| `EXACTLY_ONCE_EFFECTIVE` | At-least-once + idempotency = effective-once semantics |
| `BEST_EFFORT` | No guarantees; informational |

OACP does not claim true exactly-once delivery. `EXACTLY_ONCE_EFFECTIVE` is the strongest mode and is the combination of `AT_LEAST_ONCE` transport plus `idempotencyKey`-based deduplication.

`maxRetryAttempts` is informational; receivers MAY enforce as deployment policy.

### §3c.7 Replay

Replay is governed and explicit. A replayed message carries `replay` metadata:

```json
"replay": {
  "replayId": "replay_001",
  "originalMessageId": "msg_001",
  "originalRunId": "run_001",
  "replayMode": "ORIGINAL_POLICY",
  "reason": "RECOVERY_AFTER_TIMEOUT",
  "replayedBy": "runtime.ojas",
  "replayedAt": "2026-05-05T19:00:00Z",
  "replayAuthorizationRef": {
    "decisionId": "replay_auth_001",
    "decisionType": "REPLAY_AUTHORIZATION",
    "decidedBy": "runtime.ojas",
    "tenantId": "tenant_001"
  }
}
```

Reserved `replayMode` values:

| Mode | Policy evaluation | Hash chain |
|---|---|---|
| `ORIGINAL_POLICY` | Use policy snapshot from original emission time | Original chain preserved |
| `CURRENT_POLICY` | Use current policy at replay time | New chain segment |
| `DIAGNOSTIC_ONLY` | No policy evaluation; for inspection | No chain |
| `FORKED_RUN` | New run with current policy | New run, new chain |

**Replay authority:** only `RUNTIME` or authorized `SUPERVISOR` may emit replay metadata. An `AGENT` attempting to author replay metadata triggers `REPLAY_AUTHORITY_INVALID`.

**Replay age-bypass:** age-bypass replay modes (`ORIGINAL_POLICY`, `DIAGNOSTIC_ONLY`) require `replayAuthorizationRef` at `OACP-Governed`+. Without it: `REPLAY_AUTHORIZATION_MISSING`.

### §3c.8 Replay and idempotency interaction

A replay reuses the original `idempotencyKey`. Whether this triggers deduplication depends on `replayMode`:

| Mode | Deduplication behavior |
|---|---|
| `ORIGINAL_POLICY` | Reuse if within `dedupeWindowSeconds`; otherwise process |
| `CURRENT_POLICY` | Process as new operation under current policy |
| `DIAGNOSTIC_ONLY` | Never trigger downstream effects |
| `FORKED_RUN` | Process in new run; new `idempotencyKey` recommended |

### §3c.9 Reserved error codes (§3c)

```
IDEMPOTENCY_KEY_MISSING              — required at profile but absent
IDEMPOTENCY_CONFLICT                 — same key+scope, different payload hash
IDEMPOTENCY_SCOPE_INVALID            — scope value not in registry
RETRY_LIMIT_EXCEEDED                 — retryAttempt > maxRetryAttempts
REPLAY_AUTHORITY_INVALID             — non-RUNTIME/SUPERVISOR authored replay
REPLAY_MODE_INVALID                  — replayMode not in registry
REPLAY_AUTHORIZATION_MISSING         — age-bypass replay without authorization ref
REPLAY_ORIGINAL_NOT_FOUND            — originalMessageId doesn't resolve
REPLAY_AGE_BYPASS_INVALID            — replay claims age bypass but mode doesn't authorize
DELIVERY_MODE_INVALID                — deliveryMode not in registry
DELIVERY_MODE_DOWNGRADE              — receiver requires stricter mode than declared
```

### §3c.10 Conformance profile interactions

| Profile | Idempotency/retry/replay behavior |
|---|---|
| `OACP-Core` | Optional |
| `OACP-Operational` | Idempotency keys recommended |
| `OACP-Reliable` | Full idempotency + retry + replay semantics |
| `OACP-Governed` | Idempotency required; replay authority enforced |
| `OACP-Regulated` | All rules; replay artifacts cryptographically signed |

---

## §3d. Conversation, correlation, causation, fan-out, fan-in

### §3d.1 Five identity scopes

OACP carries five distinct identity scopes:

| Scope | Field | Question |
|---|---|---|
| Message | `messageId` | This wire emission |
| Operation | `idempotencyKey` | This logical operation (§3c) |
| Run | `runId` | This governed execution |
| Conversation | `conversationId` | This task across one or more runs |
| End-to-end trace | `correlationId` | This work boundary across systems |

These do not collapse into each other. A single conversation may contain many runs; a single run may contain many messages; a single message has one operation key.

### §3d.2 The run block

```json
"run": {
  "runId": "run_001",
  "correlationId": "corr_001"
}
```

`runId` identifies one governed execution instance. A run begins with a `RUN_STARTED` event and terminates with `RUN_COMPLETED`, `RUN_FAILED`, or `RUN_CANCELLED`. Runs are units of:

- Hash-chain scope (§3e)
- Sandbox boundary (deployment-level)
- Audit chain segment (deployment-level)

A run is **immutable in shape** once started. Profile claims and tenant scope cannot change within a run (§2a.5, §2d).

### §3d.3 The conversation block

```json
"conversation": {
  "conversationId": "conv_001",
  "turnId": "turn_003",
  "parentTurnId": "turn_002",
  "interactionMode": "AUTONOMOUS"
}
```

| Field | Meaning |
|---|---|
| `conversationId` | Conversation that may span multiple runs |
| `turnId` | Sub-identifier within the conversation |
| `parentTurnId` | Optional — previous turn |
| `interactionMode` | One of `AUTONOMOUS`, `HUMAN_REVIEW`, `HUMAN_INITIATED`, `EXTERNAL_SYSTEM` |

Conversations are **cross-run**. A single user task may produce multiple runs (initial run, NEEDS_MORE_EVIDENCE re-run, etc.) sharing the same `conversationId`. Hash chains do NOT cross runs (§3e); conversation continuity uses causation, proposalRefs, and audit-chain stitching.

### §3d.4 The correlationId

`correlationId` is the **end-to-end trace boundary**. It identifies a unit of work across systems — typically aligned with OpenTelemetry trace IDs.

`correlationId` is **not** tenant-wide. It is bounded by the work unit. Two unrelated runs in the same tenant have different `correlationId` values.

### §3d.5 The causation block

Every non-INITIATOR message carries a `causation` block:

```json
"causation": {
  "causationId": "msg_predecessor",
  "causationIds": ["msg_predecessor"],
  "causationType": "LINEAR"
}
```

Reserved `causationType` values:

| Type | Meaning | `causationId` | `causationIds[]` |
|---|---|---|---|
| `INITIATOR` | First message in a run; no predecessor | Absent | Absent |
| `LINEAR` | Single causal predecessor | One | One entry (same as `causationId`) |
| `FAN_OUT` | One predecessor, multiple sibling responses | The shared predecessor | One entry |
| `FAN_IN` | Multiple predecessors merge | The primary predecessor | All predecessors |

`causationId` is the **primary** predecessor. `causationIds[]` is the full set; for LINEAR and FAN_OUT, it equals `[causationId]`; for FAN_IN, it includes all merged predecessors.

### §3d.6 The responseTo block

For `RESPONSE`, `TOOL_RESULT`, `GATE_RESULT`, and broker-decision messages, the `responseTo` block correlates the response to its request:

```json
"responseTo": {
  "requestMessageId": "msg_request_001",
  "requestMessageType": "TOOL_CALL_REQUESTED",
  "responseStatus": "SUCCESS"
}
```

| Field | Meaning |
|---|---|
| `requestMessageId` | The request this responds to |
| `requestMessageType` | The request's `messageType` |
| `responseStatus` | One of `SUCCESS`, `ERROR`, `PARTIAL`, `TIMEOUT` |

Receivers of responses MUST verify `requestMessageId` resolves to a known request (in the receiver's view). Failure: `RESPONSE_CORRELATION_FAILED`.

`responseTo` is required for response-kind messages. `causation.causationId` typically equals `responseTo.requestMessageId` for direct responses; they MAY diverge when the response is causally downstream of multiple events.

### §3d.7 INITIATOR messages

Messages of `causationType: INITIATOR` start a run. They have:

- No `causationId`, no `causationIds[]`.
- A new `runId` (typically aligned with `messageId` for the initiator).
- A new hash chain (no `previousMessageHash` per §3e).

Reserved INITIATOR message types include `RUN_STARTED`, `CONVERSATION_INITIATED`, `EXTERNAL_REQUEST_INITIATED`.

### §3d.8 Causation graph rules

Within a run, causation forms a directed acyclic graph (DAG):

1. **No cycles.** A message cannot causally precede itself, directly or transitively. Cycle detection: `CAUSATION_CYCLE_DETECTED`.
2. **No forward references.** A causationId MUST resolve to a message that *exists at the time of the referencing message*. Forward references: `CAUSATION_FORWARD_REFERENCE`.
3. **Cross-correlationId references forbidden.** Causation does not cross `correlationId` boundaries within a run. Cross-correlation: `CAUSATION_CROSS_CORRELATION`.
4. **Cross-run causation permitted within shared conversation.** A new run in conversation C MAY reference the prior run's terminal message via causation, when both runs share `conversationId`. Cross-conversation causation: `CAUSATION_CROSS_CONVERSATION`.
5. **Tenant scope coherence.** All `causationIds[]` MUST be in the same tenant scope as the referencing message. Violation: `CAUSATION_TENANT_VIOLATION`.
6. **Fan-in scope coherence.** All `causationIds[]` in a FAN_IN MUST be in the same `runId` (or in a parent run within the same conversation per rule 4).

### §3d.9 Fan-out

A FAN_OUT pattern: one predecessor produces multiple sibling responses. Example: a parallel-evidence-collection request produces three concurrent EVIDENCE_REPORT_EMITTED messages.

```
predecessor (msg_001)
    ├── child A (msg_002, causationId: msg_001)
    ├── child B (msg_003, causationId: msg_001)
    └── child C (msg_004, causationId: msg_001)
```

Each child has `causationType: FAN_OUT`, same `causationId`, same single-entry `causationIds[]`. They share the same `previousMessageHash` (§3e.5).

### §3d.10 Fan-in

A FAN_IN pattern: multiple predecessors merge into a single message. Example: aggregating three evidence reports into one consolidated proposal.

```
parents:
    ├── msg_evidence_1
    ├── msg_evidence_2
    └── msg_evidence_3
        ↓
merged child (msg_proposal, causationId: msg_evidence_1, causationIds: [msg_evidence_1, msg_evidence_2, msg_evidence_3])
```

`causationId` is the **primary** parent. `causationIds[]` lists all parents. The hash chain follows `causationId` (per §3e.5); optional `parentMessageHashes` carries all parent hashes for regulated audit.

### §3d.11 Response correlation rules

Per §3d.6, response-kind messages carry `responseTo`. Validation:

1. `requestMessageId` MUST resolve in the receiver's view. Otherwise: `RESPONSE_CORRELATION_FAILED`.
2. `requestMessageType` SHOULD match the actual request. Mismatch is suspicious; receivers MAY emit `RESPONSE_TYPE_MISMATCH` audit warning.
3. `responseStatus` MUST be in the reserved enum.

**Profile-specific orphan handling:** receivers at `OACP-Operational` and higher MUST reject responses whose `requestMessageId` doesn't resolve. `OACP-Core` MAY accept orphan responses with audit warning.

### §3d.12 Cross-run conversation continuity

A single conversation may span multiple runs:

```
conversation C:
    run R1: INITIATOR → ... → terminal
    run R2: INITIATOR (causationId: R1.terminal) → ... → terminal
    run R3: INITIATOR (causationId: R2.terminal) → ...
```

Each run has its own hash chain, identity, and run lifecycle. Cross-run causation references are permitted because `conversationId` is shared. A common pattern: NEEDS_MORE_EVIDENCE review (§11a.6) starts a new run in the same conversation.

### §3d.13 Reserved error codes (§3d)

```
RUN_ID_MISSING                       — required at profile but absent
CONVERSATION_ID_MISSING              — required at profile but absent
CORRELATION_ID_MISSING               — required at profile but absent
CAUSATION_TYPE_INVALID               — value not in registry
CAUSATION_CYCLE_DETECTED             — causation graph has cycle
CAUSATION_FORWARD_REFERENCE          — causationId references future message
CAUSATION_CROSS_CORRELATION          — causation crosses correlationId boundary
CAUSATION_CROSS_CONVERSATION         — causation crosses conversationId boundary
CAUSATION_TENANT_VIOLATION           — causationId in different tenant
CAUSATION_FAN_IN_SCOPE_VIOLATION     — fan-in parents in different runs (without shared conversation)
RESPONSE_CORRELATION_FAILED          — requestMessageId doesn't resolve
RESPONSE_TYPE_MISMATCH               — non-blocking warning
INITIATOR_HAS_CAUSATION              — INITIATOR message carries causation
CONVERSATION_TURN_OUT_OF_ORDER       — turnId references prior turn that doesn't exist
```

### §3d.14 Illustrative example: NEEDS_MORE_EVIDENCE re-run

A complete cross-run example (Tier 1 + Tier 2 mechanics):

```
Run R1, conversation C:
    msg_001 (INITIATOR): RUN_STARTED
    msg_002 (LINEAR): PROPOSAL_EMITTED, proposalId: prop_001
    msg_003 (LINEAR): REVIEW_REQUESTED, applies to prop_001
    msg_004 (LINEAR): REVIEW_RESPONDED, outcome: NEEDS_MORE_EVIDENCE
    Run R1 terminates (no decision, awaiting re-run authorization)

Runtime decides to re-run.

Run R2, conversation C (same conversationId):
    msg_005 (INITIATOR, causationId: msg_004): RUN_STARTED
    msg_006 (LINEAR): PROPOSAL_EMITTED, proposalId: prop_002
        proposal carries:
            proposalRefs: [
              { artifactId: prop_001, relationship: RESPONDS_TO_REVIEW }
            ]
            auditRefs: [audit_review_msg_004]
    msg_007 (LINEAR): EXTERNAL_DECISION_EMITTED, decision: APPROVAL, applies to prop_002
    Run R2 terminates with prop_002 → EXTERNALLY_APPROVED
```

The original `prop_001` remains immutable. The new run produces fresh `prop_002` referencing prop_001 with `relationship: RESPONDS_TO_REVIEW`. Hash chains are independent; conversation continuity is via `conversationId` and audit infrastructure.

### §3d.15 Conformance profile interactions

| Profile | Conversation/causation behavior |
|---|---|
| `OACP-Core` | `runId` recommended; causation optional |
| `OACP-Operational` | `runId` and `causation` required; DAG enforced |
| `OACP-Governed` | All rules; cross-conversation references rejected |
| `OACP-Regulated` | All rules; `parentMessageHashes` for FAN_IN recommended |

---

## §3e. Message age, replay protection, and hash-chain semantics

This section operationalizes the integrity fields declared since §3a (`payloadHash`, `envelopeHash`, `previousMessageHash`), defines message-age limits as distinct from credential freshness, and specifies hash-chain validation rules.

### §3e.1 Three integrity dimensions

OACP messages have three distinct integrity dimensions:

| Dimension | Question | Field(s) |
|---|---|---|
| Per-message integrity | Has this message's content been modified? | `integrity.payloadHash`, `integrity.envelopeHash` |
| Message age | When was this message emitted, and is that recent enough? | `createdAt`, plus profile `maxMessageAgeSeconds` |
| Chain continuity | Does this message follow from the prior message in an unbroken chain? | `integrity.previousMessageHash` |

All three matter; none replaces the others.

### §3e.2 Per-message integrity

The envelope's `integrity` block:

```json
"integrity": {
  "canonicalization": "JCS-RFC8785",
  "hashAlgorithm": "sha256",
  "payloadHash": "sha256:abc123...",
  "envelopeHash": "sha256:def456...",
  "previousMessageHash": "sha256:ghi789...",
  "signatureRef": {
    "recordId": "sig_001",
    "recordType": "MESSAGE_SIGNATURE",
    "scopeType": "TENANT",
    "tenantId": "tenant_001"
  }
}
```

| Field | Covers |
|---|---|
| `payloadHash` | The canonical-serialized `payload` block only. Does NOT cover `payloadType` — `payloadType` is envelope metadata. |
| `envelopeHash` | The canonical-serialized envelope **excluding the entire `integrity` block**. Includes `payloadType`, all envelope authority fields, scope, governance, references. |
| `previousMessageHash` | The `envelopeHash` of the immediately-prior message (§3e.5). |
| `signatureRef` | Reference to a cryptographic signature artifact. |

**Canonicalization declaration:**

1. If `integrity` is present and any hash field is present, `canonicalization` MUST be declared or implied by the active transport binding.
2. `OACP-Regulated` MUST declare `canonicalization` explicitly on every message.
3. Reserved values: `JCS-RFC8785` (default), `OACP-CANON-V1` (reserved for future).
4. Receivers MUST reject envelopes whose `canonicalization` is unsupported with `HASH_CANONICALIZATION_UNSUPPORTED`.

**Hash algorithm:** `hashAlgorithm` declares the algorithm used in `payloadHash`, `envelopeHash`, and `previousMessageHash` when they share an algorithm. Per-field algorithm overrides MAY be carried in the hash prefix.

**Signature algorithms:** v0.1 reserves `Ed25519`, `ECDSA-P256`, and `RSA-PSS-SHA256`.

### §3e.3 Message age

A message's age is now − `createdAt`. Profiles configure `maxMessageAgeSeconds`:

**Recommended defaults (non-normative):**

| Profile | `maxMessageAgeSeconds` |
|---|---|
| `OACP-Core` | unlimited (no enforcement) |
| `OACP-Operational` | 86400 (24 hours) |
| `OACP-Governed` | 3600 (1 hour) |
| `OACP-Regulated` | 300 (5 minutes) |

The values are **recommended defaults, not normative requirements**. The profile requirement is that enforcement happens *when configured*. The configured value itself is deployment policy within profile guidance.

**Profile minimum/maximum guidance:**

| Profile | Minimum | Maximum |
|---|---|---|
| `OACP-Operational` | 60s | 7 days |
| `OACP-Governed` | 30s | 24 hours |
| `OACP-Regulated` | 5s | 1 hour |

Configurations outside profile range trigger `MESSAGE_AGE_LIMIT_OUT_OF_PROFILE_RANGE` at conformance-test time (not per-message runtime).

**Rule:** at `OACP-Operational`+, receivers MUST reject messages whose age exceeds `maxMessageAgeSeconds` with `MESSAGE_AGE_EXCEEDED`.

**Age vs. credential freshness:**

| Concern | Field | Rule |
|---|---|---|
| Credential freshness | `identity.expiresAt` | Reject if past |
| Verification freshness | `verification.verifiedAt` | Re-verify if older than window |
| **Message age** | `createdAt` | **Reject if older than maxMessageAgeSeconds** |
| Clock skew tolerance | `maxClockSkewSeconds` | Tolerance for timestamp differences |

A fresh credential does not protect against replay of an old message.

### §3e.4 Replay age-bypass

Legitimate replays per §3c.7 MAY bypass message-age limits:

| Replay mode | Bypass `maxMessageAgeSeconds`? |
|---|---|
| `ORIGINAL_POLICY` | Yes — replay intentionally evaluating old state |
| `CURRENT_POLICY` | No — replay envelope's own `createdAt` is current |
| `DIAGNOSTIC_ONLY` | Yes — diagnostic replay frequently targets old messages |
| `FORKED_RUN` | No — forked runs have current envelopes |

Age-bypass replay modes (`ORIGINAL_POLICY`, `DIAGNOSTIC_ONLY`) require `replayAuthorizationRef` at `OACP-Governed`+. Without it: `REPLAY_AUTHORIZATION_MISSING`.

### §3e.5 Hash-chain semantics

`integrity.previousMessageHash` is the prior message's `envelopeHash`. Since `envelopeHash` excludes the `integrity` block (§3e.2), there is no recursive self-reference.

**Rule:** `previousMessageHash[N] = envelopeHash[N-1]`. Receivers validate by recomputing the prior message's `envelopeHash` and comparing.

Chain rules:

1. **Chain root:** an `INITIATOR` message has no `previousMessageHash`.
2. **Chain node:** every non-INITIATOR LINEAR message has `previousMessageHash` = `envelopeHash(causation.causationId)`.
3. **Fan-out:** all siblings in a `FAN_OUT` share the same `previousMessageHash` (their common parent's `envelopeHash`).
4. **Fan-in:** `previousMessageHash` references the *primary* parent (`causation.causationId`); the full set of parents is in `causationIds[]`. The hash chain follows a single spine for tractable validation.
5. **Chain validation:** receivers MAY verify `previousMessageHash` by resolving prior and recomputing. Required at `OACP-Regulated`; optional at lower profiles.

**Optional `parentMessageHashes` for FAN_IN:**

```json
"integrity": {
  "previousMessageHash": "sha256:primary_parent_envelope_hash...",
  "parentMessageHashes": {
    "msg_evidence_1": "sha256:...",
    "msg_evidence_2": "sha256:...",
    "msg_evidence_3": "sha256:..."
  }
}
```

Rules:

1. `previousMessageHash` follows the primary parent.
2. `parentMessageHashes` MAY record all fan-in parent hashes, keyed by parent `messageId`.
3. Keys in `parentMessageHashes` MUST exactly match `causation.causationIds`. Mismatch: `PARENT_MESSAGE_HASHES_MISMATCH`.
4. `OACP-Regulated` SHOULD include `parentMessageHashes` for FAN_IN.
5. Optional even at Regulated; alternative is auditor-side reconstruction.

### §3e.6 Profile requirements

| Profile | `previousMessageHash` requirement |
|---|---|
| `OACP-Core` | Optional |
| `OACP-Operational` | SHOULD be present on non-INITIATOR; validation optional |
| `OACP-Governed` | MUST be present on non-INITIATOR; receivers SHOULD validate |
| `OACP-Regulated` | MUST be present and validated; broken chains hard-reject |

**Broken chain handling:**

- `OACP-Governed`: `HASH_CHAIN_BROKEN` non-blocking; surfaced for audit.
- `OACP-Regulated`: `HASH_CHAIN_BROKEN` blocking; message rejected.

### §3e.7 Hash-chain repair and run-scope

Hash chains cannot be silently repaired. Broken chains are immutable historical events.

**Chain scope rule:** hash-chain validation is **run-scoped** in v0.1. The chain spans messages within a single `runId`. It does not extend across runs even when those runs share `conversationId`.

Conversation continuity across runs uses:

1. `causation.causationId` referencing the prior run's terminal message.
2. `proposalRefs[]` linking new proposals to prior proposals (§13.8).
3. Audit infrastructure stitching together hash-chain segments.

Not permitted:

1. Computing `previousMessageHash` across run boundaries.
2. Treating multi-run conversations as a single hash chain.
3. "Bridging" two run-scoped chains through synthetic anchors.

Receivers MUST reject envelopes whose `previousMessageHash` references an `envelopeHash` from a different `runId` with `HASH_CHAIN_CROSS_RUN_VIOLATION`.

Why run-scoped: each run is a bounded governed execution with its own immutable audit chain. Cross-run hash chains would couple runs in ways that complicate independent audit, replay, and policy evaluation.

### §3e.8 Signature semantics

When `signatureRef` is present, it points to a signature artifact:

```json
{
  "signatureId": "sig_001",
  "signatureAlgorithm": "Ed25519",
  "canonicalization": "JCS-RFC8785",
  "hashAlgorithm": "sha256",
  "signedFields": ["envelopeHash", "payloadHash", "previousMessageHash"],
  "signedBy": {
    "participantId": "agent.classifier",
    "participantType": "AGENT",
    "credentialRef": "cred_001"
  },
  "signatureValue": "<base64-encoded>",
  "signedAt": "2026-05-05T17:00:00Z",
  "tenantId": "tenant_001"
}
```

Signature rules:

1. `signedFields` specifies which envelope fields are covered. At minimum, `envelopeHash` MUST be signed.
2. Receivers verifying the signature recompute the canonical concatenation of listed fields and verify against `signatureValue`.
3. The signing key's authority is checked at signature-emission time (analogous to §10.9 broker frozen-policy rule).
4. Signature failures: `SIGNATURE_INVALID` blocking at `OACP-Regulated`; lower profiles SHOULD record but MAY proceed.

**Signature material binding:** when a signature covers `envelopeHash` or `payloadHash`, the signature material MUST also bind `canonicalization` and `hashAlgorithm` identifiers — not just hash values.

The signature artifact's `canonicalization` and `hashAlgorithm` MUST match `integrity.canonicalization` and `integrity.hashAlgorithm` for the verified message. Mismatch: `SIGNATURE_BINDING_INVALID`.

Why this matters: if a signature covered only hash *values*, an attacker who can change `canonicalization` or `hashAlgorithm` can re-derive different bytes producing the same signed hash output — defeating signature integrity.

### §3e.9 Signing and replay interaction

A replayed message under `ORIGINAL_POLICY` carries the original signature. Verifying requires the original signing key (or its public key) still retrievable.

**Key revocation at emission time:**

| Key status | Replay verification |
|---|---|
| `ACTIVE` at `signedAt` | Verifies (under `ORIGINAL_POLICY`) |
| `REVOKED` after `signedAt` (forward revocation) | Still verifies — key was valid when used |
| `COMPROMISED` at or before `signedAt` (retroactive) | Fails with `SIGNATURE_KEY_REVOKED` |
| `REVOKED` retroactively to date ≤ `signedAt` | Fails with `SIGNATURE_KEY_REVOKED` |

Forward revocation does not invalidate prior signatures. Retroactive invalidation does. Receivers consult key status *as of `signedAt`*, not just current.

### §3e.10 Reserved error codes (§3e)

```
MESSAGE_AGE_EXCEEDED                     — message older than maxMessageAgeSeconds
MESSAGE_AGE_LIMIT_OUT_OF_PROFILE_RANGE   — config-time; deployment value outside profile range
PAYLOAD_HASH_INVALID                     — payloadHash doesn't match canonical-serialized payload
ENVELOPE_HASH_INVALID                    — envelopeHash doesn't match canonical-serialized envelope
PAYLOAD_HASH_MISSING                     — required by profile but absent
ENVELOPE_HASH_MISSING                    — required by profile but absent
PREVIOUS_MESSAGE_HASH_MISSING            — required at profile but absent
PREVIOUS_MESSAGE_HASH_FORMAT_INVALID     — hash field malformed
HASH_CHAIN_BROKEN                        — computed envelopeHash of prior ≠ claimed previousMessageHash
HASH_CHAIN_PRIOR_UNRESOLVABLE            — cannot resolve prior message to verify chain
HASH_CHAIN_CROSS_RUN_VIOLATION           — previousMessageHash references different run
HASH_CANONICALIZATION_MISMATCH           — sender and receiver disagree on canonicalization
HASH_CANONICALIZATION_UNSUPPORTED        — canonicalization not supported by receiver
PARENT_MESSAGE_HASHES_MISMATCH           — parentMessageHashes keys don't match causationIds
SIGNATURE_MISSING                        — required by profile but absent
SIGNATURE_INVALID                        — verification failed
SIGNATURE_ALGORITHM_UNSUPPORTED          — algorithm not supported
SIGNATURE_KEY_UNAVAILABLE                — signing key cannot be retrieved
SIGNATURE_KEY_REVOKED                    — key revoked at or before signedAt
SIGNED_FIELDS_INSUFFICIENT               — signature does not cover required fields
SIGNATURE_BINDING_INVALID                — canonicalization or hashAlgorithm mismatch
REPLAY_AGE_BYPASS_INVALID                — replay claims age bypass but mode does not authorize
REPLAY_AUTHORIZATION_MISSING             — see §3c.9
```

### §3e.11 Conformance profile interactions

| Profile | Age, replay, hash-chain behavior |
|---|---|
| `OACP-Core` | All rules optional |
| `OACP-Operational` | Message age enforced; per-message hashes optional |
| `OACP-Governed` | `previousMessageHash` required on non-INITIATOR; hashes required; chain broken-recorded |
| `OACP-Regulated` | All hashes required; chain validated; signatures required on capability-scoped messages |

### §3e.12 Out of scope for v0.1

- Post-quantum signature algorithms.
- Multi-signature schemes.
- Hash-chain Merkle proofs.
- Time-stamping authority integration.
- Cross-tenant chain federation.

---

## §10. Broker outcome state machine

The Tool Broker mediates capability-scoped tool calls. It is the protocol's choke point for tool governance — every tool call passes through the broker for policy evaluation, sanitization, optional approval, and auditing.

### §10.1 The broker is an OACP participant

The Tool Broker is a `participantType: TOOL_BROKER` participant per §3a.8. It receives `TOOL_CALL_REQUESTED` messages, evaluates them against policy, transitions through the state machine below, and emits `TOOL_CALL_*` lifecycle messages and `TOOL_RESULT` responses.

### §10.2 The 15-state machine

```
REQUESTED                            tool call request received
    ↓
RECEIVED                             broker acknowledged
    ↓
VALIDATING                           syntactic/contract validation
    ↓
POLICY_EVALUATING                    policy engine consulted
    ↓
    ├── ALLOWED                      policy allowed; ready to dispatch
    │       ↓
    │   DISPATCHED                   tool invocation initiated
    │       ↓
    │       ├── COMPLETED            tool returned successfully
    │       ├── FAILED               tool errored
    │       ├── EXPIRED              tool timed out
    │       └── CANCELLED            run cancelled mid-tool
    │
    ├── DENIED                       policy denied; terminal
    │
    ├── SANITIZED                    payload sanitized; ready to dispatch sanitized version
    │       ↓
    │   DISPATCHED (sanitized)
    │       ↓
    │       └── (same terminal states as ALLOWED branch)
    │
    ├── AWAITING_APPROVAL            policy requires external approval
    │       ↓
    │       ├── (approval received) → ALLOWED → DISPATCHED → ...
    │       └── (approval denied)   → REJECTED (terminal)
    │
    └── QUARANTINED                  policy flagged for inspection; terminal in v0.1
```

### §10.3 Decision vs. state

The broker has **two distinct concepts** that must not be conflated:

| Concept | Field | Value space |
|---|---|---|
| Policy decision (outcome) | `decision` | `ALLOW`, `DENY`, `SANITIZE`, `APPROVAL_NEEDED`, `QUARANTINE`, `CANCEL` |
| State entered (state machine) | `decisionState` | `ALLOWED`, `DENIED`, `SANITIZED`, `AWAITING_APPROVAL`, `QUARANTINED`, terminal states |

The policy decision is what the policy returned. The state is what the broker entered as a result. They are not the same enum.

### §10.4 Tool call request shape

```json
{
  "messageKind": "TOOL_CALL",
  "messageType": "TOOL_CALL_REQUESTED",
  "payloadType": "tool-call-request-v1",
  "payload": {
    "toolCallId": "tc_001",
    "toolId": "send_email",
    "inputRef": {
      "artifactId": "tc_input_001",
      "artifactType": "TOOL_INPUT",
      "uri": "ojas://tenant_001/tool-inputs/tc_input_001",
      "mimeType": "application/json",
      "hash": "sha256:...",
      "tenantId": "tenant_001"
    },
    "bindingId": "bind_001",
    "requestedBy": {
      "participantId": "agent.notifier",
      "participantType": "AGENT"
    }
  }
}
```

### §10.5 Broker decision artifact

The broker's decision is a `DecisionRef` of type `BROKER`:

```json
{
  "decisionId": "broker_dec_001",
  "decisionType": "BROKER",
  "decision": "SANITIZE",
  "decisionState": "SANITIZED",
  "appliesTo": {
    "objectType": "TOOL_CALL",
    "brokerDecisionId": "broker_dec_001",
    "toolCallId": "tc_001"
  },
  "decidedBy": "broker.tool",
  "decidedAt": "2026-05-05T17:05:00Z",
  "policySnapshotId": "policy-snapshot-001",
  "sanitizedPayloadRef": {
    "artifactId": "san_001",
    "artifactType": "SANITIZED_PAYLOAD",
    "uri": "ojas://tenant_001/sanitized/san_001",
    "hash": "sha256:...",
    "tenantId": "tenant_001"
  },
  "tenantId": "tenant_001"
}
```

### §10.6 ALLOW vs. tool success

**Important distinction:** `ALLOW` is the broker's policy outcome. It does NOT mean the tool succeeded. The tool must still be dispatched and may itself fail (`FAILED`, `EXPIRED`).

A `DISPATCHED → COMPLETED` transition emits `TOOL_RESULT` with `responseStatus: SUCCESS`. A `DISPATCHED → FAILED` transition emits `TOOL_RESULT` with `responseStatus: ERROR`. Both are downstream of `ALLOWED`.

### §10.7 Sanitization flow

When `decision = SANITIZE`:

1. The broker produces a sanitized version of the payload.
2. The sanitized payload is externalized as an `ArtifactRef` of type `SANITIZED_PAYLOAD`.
3. The broker dispatches the **sanitized version**, not the original.
4. `decisionState` enters `SANITIZED` then `DISPATCHED`.
5. The original payload is preserved in audit; the dispatched payload is the sanitized one.

**Sanitized idempotency key derivation:** when the broker creates a sanitized payload for an idempotency-keyed operation, the sanitized version uses a derived key:

```
sanitizedIdempotencyKey = "san:" + originalIdempotencyKey + ":" + profileId + ":" + profileVersion + ":" + policySnapshotId
```

This ensures sanitized retries are idempotent within their sanitization context but distinguish from the original operation's idempotency.

### §10.8 APPROVAL_NEEDED flow

When `decision = APPROVAL_NEEDED`:

1. `decisionState` enters `AWAITING_APPROVAL`.
2. The broker emits `TOOL_CALL_AWAITING_APPROVAL`.
3. The policy snapshot is **frozen at the decision time** — the snapshot is preserved with the broker decision.
4. An approval request is routed (typically to a review system; see §11a for review semantics or to an external decision authority).
5. The decider's identity is **validated at the time of decision emission** (not at the time the broker requested approval).
6. On approval: `decisionState` transitions back to `ALLOWED`, then `DISPATCHED`.
7. On rejection: `decisionState` transitions to `REJECTED` (terminal).
8. On TTL elapse without decision: `EXPIRED`.

**Frozen-policy / current-identity rule:** the policy under which approval is granted is the snapshot at decision time. The decider's identity authority is validated at emission time. This separation ensures that a user who lost authority between request and approval cannot retroactively approve.

### §10.9 ExternalDecisionRef on the broker decision

When approval involves an external decider, the broker decision references an `ExternalDecisionRef` (per §13.6) via `appliesTo.brokerDecisionId` linking the external decision to the broker's pending state:

```json
"appliesTo": {
  "objectType": "BROKER_DECISION",
  "brokerDecisionId": "broker_dec_001"
}
```

The broker correlates incoming `ExternalDecisionRef` artifacts to its pending `AWAITING_APPROVAL` decisions via this link.

### §10.10 Persist-before-emit rule

The broker MUST persist its state transition **before** emitting the corresponding lifecycle message. This ensures that if the broker crashes mid-emission, the state is recoverable and the broker can re-emit on recovery without producing duplicate state transitions.

A broker recovering from crash:

1. Reads its persisted state.
2. For each transition that was persisted but not confirmed-emitted, re-emits the lifecycle message (with the same `messageId` for idempotency).
3. For each pending transition that was not yet persisted, restarts from the prior persisted state.

Receivers tolerate duplicate broker lifecycle messages within `dedupeWindowSeconds` (per §3c).

### §10.11 Tool result correlation

`TOOL_RESULT` messages carry `responseTo`:

```json
"responseTo": {
  "requestMessageId": "msg_tool_call_request",
  "requestMessageType": "TOOL_CALL_REQUESTED",
  "responseStatus": "SUCCESS"
}
```

The `requestMessageId` references the original `TOOL_CALL_REQUESTED`, not any of the intermediate broker lifecycle messages.

### §10.12 Quarantine

`QUARANTINED` is a terminal state in v0.1. A quarantined tool call is held by the broker for inspection; downstream effects do not occur. Release-from-quarantine flow is **deferred to Tier 4 / v0.2**.

### §10.13 Reserved error codes (§10)

```
TOOL_CALL_REQUEST_INVALID            — payload validation failed
BROKER_STATE_INVALID                 — transition violates state machine
BROKER_DECISION_MISSING              — required at profile but absent
BROKER_POLICY_SNAPSHOT_MISSING       — frozen policy reference missing
BROKER_AUTHORITY_INVALID             — non-broker emitted broker lifecycle message
BROKER_RESPONSE_CORRELATION_FAILED   — TOOL_RESULT references non-existent request
SANITIZED_PAYLOAD_MISSING            — SANITIZE decision without sanitizedPayloadRef
SANITIZED_KEY_DERIVATION_INVALID     — sanitized idempotencyKey doesn't match derivation rule
APPROVAL_TIMEOUT                     — AWAITING_APPROVAL exceeded TTL
APPROVAL_DECIDER_AUTHORITY_INVALID   — decider not authorized for this tool/scope
QUARANTINE_RELEASE_NOT_SUPPORTED     — release attempted; deferred to Tier 4
```

### §10.14 Conformance profile interactions

| Profile | Broker behavior |
|---|---|
| `OACP-Core` | Broker optional |
| `OACP-Operational` | Broker recommended for tool calls |
| `OACP-Tool` | Full state machine required |
| `OACP-Governed` | Broker required for all capability-scoped tool calls; policy snapshots required |
| `OACP-Regulated` | Broker decisions cryptographically signed; persist-before-emit cryptographically verifiable |

---

## §11. Gate / validation protocol (sketch)

Validation gates produce typed `ValidationResult` artifacts. A `GATE_RESULT` message wraps the result:

```json
{
  "messageKind": "GATE_RESULT",
  "messageType": "GATE_RESULT_EMITTED",
  "payloadType": "gate-result-v1",
  "payload": {
    "gateId": "gate_jrxml_compile",
    "validationResultRef": {
      "artifactId": "vr_001",
      "artifactType": "VALIDATION_RESULT",
      "uri": "ojas://tenant_001/validation/vr_001",
      "hash": "sha256:...",
      "tenantId": "tenant_001"
    },
    "result": "PASSED"
  }
}
```

`result` reserved values: `PASSED`, `FAILED`, `INCONCLUSIVE`, `SKIPPED`.

Gate semantics in v0.1 are deliberately minimal — the gate runner is a participant; gate results are typed; downstream consumers (e.g., proposal-emission readiness checks per §13.5) reference `validationResultRef`. Detailed gate-execution semantics are deferred to Tier 4.

---

## §11a. REVIEW protocol

This section defines the REVIEW message kind, its state machine, the review types and outcomes, and the rules governing review-feedback flow back to the originating run.

### §11a.1 What review is, and what it is not

> **A review request asks for external input that informs a decision. A review response provides input. Neither emits a decision.**

| Concept | What it does | What it does NOT do |
|---|---|---|
| Review request | Asks an external participant for input | Cannot demand approval |
| Review response | Provides feedback, additional context, or correction signal | Cannot approve, reject, or finalize |
| Escalation | Requests expanded authority to perform an action | Cannot perform the action |
| External decision | Approves, rejects, or escalates a proposal | Final state for the proposal |

Review and escalation are siblings: both engage external participants. Review is for *decision support*; escalation is for *authority widening*.

### §11a.2 Review types and review outcomes

A review request carries `reviewType` (what kind of review is being requested). A review response carries `reviewOutcome` (what the reviewer determined). They are different fields with different value spaces.

**Reserved `reviewType` values** (on `REVIEW_REQUESTED`):

| `reviewType` | Triggered when |
|---|---|
| `CLARIFICATION_REQUEST` | A reviewer needs information to interpret the message |
| `DECISION_SUPPORT` | A reviewer is asked for feedback that will inform a separate decision |
| `EVIDENCE_REVIEW` | A reviewer is asked to assess evidence quality or completeness |
| `COMPLIANCE_REVIEW` | A reviewer is asked to evaluate against compliance criteria |

**Reserved `reviewOutcome` values** (on `REVIEW_RESPONDED`):

| `reviewOutcome` | Meaning |
|---|---|
| `FEEDBACK_PROVIDED` | Reviewer provided substantive input |
| `NEEDS_MORE_EVIDENCE` | Reviewer determined current evidence insufficient — typically triggers re-run |
| `NO_OBJECTION` | Reviewer has no concerns; **NOT approval** |
| `CONCERN_RAISED` | Reviewer flagged a concern; non-blocking |
| `UNABLE_TO_REVIEW` | Reviewer cannot complete review |

Custom values use namespace prefixes per §2b.7.

**Important:** `NEEDS_MORE_EVIDENCE` is a `reviewOutcome`, not a `reviewType`. A runtime requests `DECISION_SUPPORT`; the reviewer responds with `reviewOutcome: NEEDS_MORE_EVIDENCE` if more evidence is needed. The two fields are independent: any `reviewType` can produce any `reviewOutcome`.

**The boundary preserved:** none of these outcomes — including `NO_OBJECTION` — constitutes approval. Approval requires a separate `ExternalDecisionRef` (§13).

### §11a.3 Review request

```json
{
  "messageKind": "REVIEW",
  "messageType": "REVIEW_REQUESTED",
  "ttl": {
    "expiresAt": "2026-05-05T18:00:00Z"
  },
  "payloadType": "review-request-v1",
  "payload": {
    "reviewId": "review_001",
    "reviewType": "EVIDENCE_REVIEW",
    "appliesTo": {
      "objectType": "PROPOSAL",
      "proposalRef": {
        "artifactId": "prop_001",
        "artifactType": "PROPOSAL",
        "uri": "ojas://tenant_001/proposals/prop_001",
        "hash": "sha256:...",
        "tenantId": "tenant_001"
      }
    },
    "requestedInputs": [
      "rerun_visual_similarity_check",
      "include_page_3_overlay_diff",
      "validate_signature_block_split"
    ],
    "reviewerNotes": "Need proof that dynamic section did not clip footer.",
    "reviewSystemRef": "review-system-001",
    "businessDeadline": "2026-05-05T19:00:00Z"
  }
}
```

| Field | Meaning |
|---|---|
| `reviewId` | Stable identifier |
| `reviewType` | Per §11a.2 |
| `appliesTo` | What's being reviewed |
| `requestedInputs` | Structured list of needed information |
| `reviewerNotes` | Free-text note |
| `reviewSystemRef` | Review system handling this request |
| `businessDeadline` | Informational; **not protocol TTL** (use `ttl.expiresAt`) |

### §11a.4 Review state machine

```
REQUESTED                            runtime emits REVIEW_REQUESTED
    ↓
PENDING                              review system has accepted; awaiting reviewer
    ↓
    ├── RESPONDED                    reviewer provided feedback
    │       │
    │       ├── (NEEDS_MORE_EVIDENCE) → may trigger re-run (§11a.6); review terminal
    │       ├── (CLARIFICATION provided) → informational; review terminal
    │       └── (DECISION_SUPPORT feedback) → ReviewFeedback referenced by subsequent ExternalDecisionRef
    │
    ├── MORE_INFORMATION_REQUIRED    reviewer asks for clarification
    │       │
    │       ├── (provided) → back to PENDING
    │       └── (cannot)   → CANCELLED
    │
    ├── CANCELLED                    requestor or runtime cancelled
    │
    └── EXPIRED                      ttl.expiresAt elapsed
```

State message types:

```
REVIEW_REQUESTED
REVIEW_PENDING
REVIEW_RESPONDED
REVIEW_MORE_INFORMATION_REQUIRED
REVIEW_INFORMATION_PROVIDED
REVIEW_CANCELLED
REVIEW_EXPIRED
```

### §11a.4a Review-response correlation

Every `REVIEW_RESPONDED` (and `REVIEW_MORE_INFORMATION_REQUIRED`, `REVIEW_INFORMATION_PROVIDED`) MUST carry both `responseTo` and `causation`:

```json
"responseTo": {
  "requestMessageId": "msg_review_request_001",
  "requestMessageType": "REVIEW_REQUESTED",
  "responseStatus": "SUCCESS"
},
"causation": {
  "causationId": "msg_review_request_001",
  "causationType": "LINEAR"
}
```

Both are required. `responseTo` correlates request/response; `causation` keeps the message in the audit DAG.

Receivers that cannot resolve `responseTo.requestMessageId` reject with `RESPONSE_CORRELATION_FAILED`.

### §11a.5 Review feedback artifact

A `RESPONDED` transition produces a `ReviewFeedback` artifact, referenced via `reviewFeedbackRef`. May be either a `RecordRef` or `ArtifactRef`:

```json
"reviewFeedbackRef": {
  "recordId": "rf_001",
  "recordType": "REVIEW_FEEDBACK",
  "registry": "review-registry",
  "scopeType": "TENANT",
  "tenantId": "tenant_001"
}
```

or:

```json
"reviewFeedbackRef": {
  "artifactId": "rf_artifact_001",
  "artifactType": "REVIEW_FEEDBACK",
  "uri": "ojas://tenant_001/reviews/rf_001",
  "mimeType": "application/json",
  "hash": "sha256:...",
  "tenantId": "tenant_001"
}
```

Selection: lightweight feedback uses `RecordRef`; feedback with attached evidence files or images uses `ArtifactRef`.

### §11a.6 The NEEDS_MORE_EVIDENCE flow

When a `REVIEW_RESPONDED` arrives with `reviewOutcome: NEEDS_MORE_EVIDENCE`:

1. The originating run remains in its state. The review terminates.
2. **A `NEEDS_MORE_EVIDENCE` outcome does NOT itself start a new run.** It signals that a re-run *may* be warranted. **Only `RUNTIME` or authorized `SUPERVISOR` may start the new run**, after evaluating against deployment policy.
3. If runtime decides to re-run: a new run begins with fresh `runId`, same `conversation.conversationId` as original.
4. The new run incorporates additional inputs from feedback.
5. The new run produces a fresh proposal. Proposal-reference relationships (`RESPONDS_TO_REVIEW`, `SUPERSEDES`) are governed by §13 Proposal protocol.
6. The new proposal does NOT mutate the prior proposal. Original remains immutable.
7. The new run flows through normal authorization, binding, evidence, and proposal-emission gates.

If runtime decides NOT to re-run, this is recorded as `RERUN_DECLINED` audit event.

An `AGENT` cannot initiate a re-run. Doing so triggers `RERUN_AUTHORITY_INVALID`.

### §11a.7 Review feedback is NOT approval

> **A `REVIEW_RESPONDED` message — including its `ReviewFeedback` artifact — does NOT constitute an approval, rejection, or any other terminal decision on the proposal under review. Only an `ExternalDecisionRef` with the appropriate `decisionType` (APPROVAL, REJECTION, ESCALATION) finalizes a proposal.**

Concrete consequences:

1. A reviewer responding "I think this should be approved" does NOT approve. Runtime must still wait for `ExternalDecisionRef`.
2. A reviewer responding to `DECISION_SUPPORT` provides feedback that *informs* a subsequent decision artifact.
3. Implementations conflating `REVIEW_RESPONDED` with approval are non-conformant. `REVIEW_FEEDBACK_NOT_DECISION` rejects such conflation.
4. The proposal-vs-approval boundary holds across review flows.

### §11a.8 Review and escalation interaction

A review may itself trigger an escalation:

1. Review enters `MORE_INFORMATION_REQUIRED` or terminates with note indicating "escalation requested."
2. A separate `ESCALATION_REQUESTED` message is emitted (per §3a.13).
3. Escalation is processed independently; may produce its own `ExternalDecisionRef`.
4. The original review is closed.

Reviews cannot escalate themselves. The two flows are separate state machines that may interleave but don't merge.

### §11a.9 Review authority

- `EXTERNAL_REVIEWER` MAY emit `REVIEW` and `RESPONSE` messages and `ExternalDecisionRef` (the last only when separately authorized).
- `RUNTIME` and `SUPERVISOR` MAY emit `REVIEW_REQUESTED` and may cancel/expire reviews.
- `AGENT` MAY NOT emit `REVIEW` directly.

**How agents request review indirectly:**

1. Emit a `PROPOSAL` with metadata indicating review is needed (e.g., HIGH-risk per policy).
2. Emit an `EVENT` of type `REVIEW_HINT_EMITTED`.
3. Emit an `ESCALATION` of type `REVIEW_REQUEST_HINT`.

The runtime evaluates the hint and decides whether to emit `REVIEW_REQUESTED`. Direct agent emission of `REVIEW_REQUESTED` triggers `AUTHORITY_VIOLATION`.

### §11a.10 TTL and expiry

1. Every `REVIEW_REQUESTED` MUST carry envelope `ttl.expiresAt`. Indefinite waiting is forbidden.
2. Without `ttl.expiresAt`: `REVIEW_TTL_REQUIRED`.
3. Payload-level fields like `businessDeadline` are informational; they cannot extend protocol TTL.
4. On expiry: `EXPIRED` state. Audit-recorded with `reviewId` and elapsed duration.

### §11a.11 Reserved error codes (§11a)

```
REVIEW_TTL_REQUIRED                  — REVIEW_REQUESTED lacks ttl.expiresAt
REVIEW_AUTHORITY_INVALID             — non-authorized participant emitted REVIEW
REVIEW_TYPE_UNKNOWN                  — reviewType not in registry, no namespace
REVIEW_OUTCOME_UNKNOWN               — reviewOutcome not in registry, no namespace
REVIEW_APPLIES_TO_INVALID            — appliesTo references non-existent or unauthorized object
REVIEW_FEEDBACK_NOT_DECISION         — implementation attempted to treat feedback as approval
REVIEW_RESPONSE_OUT_OF_ORDER         — RESPONDED arrives without prior REQUESTED/PENDING
REVIEW_STATE_INVALID                 — state transition violates state machine
RERUN_AUTHORITY_INVALID              — non-RUNTIME/SUPERVISOR attempted re-run
RERUN_INITIATION_FAILED              — runtime cannot start new run from review feedback
```

### §11a.12 Conformance profile interactions

| Profile | Review behavior |
|---|---|
| `OACP-Core` | REVIEW message kind supported; full state machine optional |
| `OACP-Operational` | Full state machine required; review TTL enforced |
| `OACP-Review` | Full review semantics; structured `requestedInputs`; review-and-escalation interaction |
| `OACP-Governed` | All rules; `ReviewFeedback` artifacts MUST be RecordRef-backed; NEEDS_MORE_EVIDENCE re-run flow MUST be supported |
| `OACP-Regulated` | All rules; `ReviewFeedback` cryptographically signed; review audit trail immutable |

---

## §13. Proposal protocol and decision artifacts

This section closes the proposal/decision boundary. It defines the proposal lifecycle, the decision artifact taxonomy (`ExternalDecisionRef`, `RuntimeDecisionRef`), the auto-approval flow, the relationships between successive proposals, and the rules governing when one proposal may reference, supersede, or correct another.

### §13.1 The proposal/decision boundary

> **Agents emit proposals. Proposals are recommendations — never approvals. Every proposal closure is either authoritative (a decision artifact) or non-authoritative (expiry, supersession, runtime auto-approval). Approval requires a separate decision artifact with explicit authority.**

Concrete consequences, repeated for emphasis:

1. A `PROPOSAL_EMITTED` message is never a decision.
2. A `REVIEW_RESPONDED` message is never a decision (per §11a.7).
3. Decisions are **separate artifacts** referenced via `DecisionRef`.
4. Decision authority is verified at the time of decision emission.

### §13.2 Proposal artifact

A proposal is a typed artifact (`ArtifactRef` with `artifactType: PROPOSAL`):

```json
{
  "artifactId": "prop_001",
  "artifactType": "PROPOSAL",
  "uri": "ojas://tenant_001/proposals/prop_001",
  "mimeType": "application/json",
  "hash": "sha256:...",
  "sizeBytes": 8192,
  "createdAt": "2026-05-05T17:10:00Z",
  "expiresAt": "2026-05-05T19:10:00Z",
  "tenantId": "tenant_001",
  "workspaceId": "workspace_001",
  "dataClasses": ["INTERNAL"]
}
```

The proposal's resolved content carries:

```json
{
  "proposalId": "prop_001",
  "proposalVersion": "1",
  "proposalLifecycleState": "PENDING",
  "proposedBy": {
    "participantId": "agent.classifier",
    "participantType": "AGENT",
    "bindingId": "bind_001"
  },
  "proposedAt": "2026-05-05T17:10:00Z",
  "recommendationRef": {
    "artifactId": "rec_001",
    "artifactType": "RECOMMENDATION",
    "uri": "ojas://tenant_001/recommendations/rec_001",
    "hash": "sha256:...",
    "tenantId": "tenant_001"
  },
  "evidenceReportRefs": [
    {
      "artifactId": "evidence_001",
      "artifactType": "EVIDENCE_REPORT",
      "uri": "ojas://tenant_001/evidence/evidence_001",
      "hash": "sha256:...",
      "tenantId": "tenant_001"
    }
  ],
  "validationResultRefs": [
    {
      "artifactId": "vr_001",
      "artifactType": "VALIDATION_RESULT",
      "uri": "ojas://tenant_001/validation/vr_001",
      "hash": "sha256:...",
      "tenantId": "tenant_001"
    }
  ],
  "riskAssessmentRef": {
    "decisionId": "risk_assess_001",
    "decisionType": "RISK_ASSESSMENT",
    "decidedBy": "risk-engine-001",
    "decidedAt": "2026-05-05T17:09:00Z",
    "tenantId": "tenant_001"
  },
  "calibration": {
    "confidence": 0.92,
    "calibrationMethod": "TEMPERATURE_SCALED",
    "modelVersion": "classifier-v3"
  },
  "proposalRefs": [],
  "policySnapshotId": "policy-snapshot-001",
  "tenantId": "tenant_001",
  "runId": "run_001",
  "conversationId": "conv_001"
}
```

| Field | Required | Meaning |
|---|---|---|
| `proposalId` | Yes | Stable identifier; immutable |
| `proposalVersion` | Yes | Increments only via SUPERSEDES (§13.8) |
| `proposalLifecycleState` | Yes | Per §13.4 |
| `proposedBy` | Yes | Emitting participant + binding |
| `recommendationRef` | Yes | The recommendation artifact |
| `evidenceReportRefs[]` | At Governed+ | Supporting evidence |
| `validationResultRefs[]` | When applicable | Validation gate results |
| `riskAssessmentRef` | At Governed+ | Risk classification (see §13.5) |
| `calibration` | When proposal involves model output | Confidence + calibration method |
| `proposalRefs[]` | When applicable | Relationships to other proposals (§13.8) |
| `policySnapshotId` | At Governed+ | Active policy at proposal time |

### §13.3 PROPOSAL_EMITTED message

The proposal artifact is emitted via a `PROPOSAL_EMITTED` message:

```json
{
  "messageKind": "PROPOSAL",
  "messageType": "PROPOSAL_EMITTED",
  "payloadType": "proposal-emitted-v1",
  "payload": {
    "proposalId": "prop_001",
    "proposalRef": {
      "artifactId": "prop_001",
      "artifactType": "PROPOSAL",
      "uri": "ojas://tenant_001/proposals/prop_001",
      "hash": "sha256:...",
      "tenantId": "tenant_001"
    }
  }
}
```

Receivers fetch the proposal artifact via the `proposalRef`. The proposal payload is intentionally minimal in the message; the artifact carries the body.

### §13.4 Proposal lifecycle states

```
DRAFT                               proposal authored, not yet emitted
    ↓
PENDING                             emitted; awaiting decision
    ↓
    ├── ESCALATED                   intermediate; runtime is routing for decision
    │       ↓
    │       (terminal states below)
    │
    ├── AUTO_APPROVED                runtime auto-approved per §13.5 (terminal)
    │
    ├── EXTERNALLY_APPROVED          external decider approved (terminal)
    │
    ├── REJECTED                     external decider rejected, or runtime rejected (terminal)
    │
    ├── EXPIRED                      ttl elapsed without decision (terminal)
    │
    └── SUPERSEDED                   later proposal supersedes this one (terminal)
```

Reserved `proposalLifecycleState` values: `DRAFT`, `PENDING`, `ESCALATED`, `AUTO_APPROVED`, `EXTERNALLY_APPROVED`, `REJECTED`, `EXPIRED`, `SUPERSEDED`.

`ESCALATED` is an intermediate state. From `ESCALATED`, the proposal transitions to one of the terminal states based on the escalation outcome. It does NOT loop back to `PENDING`.

**Every closure produces a decision record.** Even `EXPIRED` and `SUPERSEDED` produce decision records (§13.6, §13.7) — for audit and for the proposal's traceable end-state.

### §13.5 Auto-approval — the ten preconditions

Runtime auto-approval is rule-based and explicit. **All ten of the following preconditions MUST hold simultaneously** for runtime to auto-approve a proposal:

1. **Risk class is LOW.** `riskAssessmentRef.result` is `LOW_RISK` (or equivalent in deployment policy).
2. **Calibration meets threshold.** `calibration.confidence` ≥ deployment-configured threshold AND `calibration.calibrationMethod` is in the deployment-permitted set.
3. **All validation gates passed.** All `validationResultRefs[]` resolve to `PASSED` results.
4. **Evidence quality threshold met.** All `evidenceReportRefs[]` meet deployment-configured quality gates.
5. **Authority is bound and valid.** `proposedBy.bindingId` resolves to an `ACTIVE` or `BOUND` binding at decision time.
6. **Tenant scope verified.** Proposal's `tenantId` is in the auto-approval policy's authorized scope.
7. **Auto-approval is policy-permitted for this `messageType` and binding.** The active policy snapshot explicitly permits auto-approval for this proposal type.
8. **No active review.** No `REVIEW_REQUESTED` is in `PENDING` or `MORE_INFORMATION_REQUIRED` for this proposal.
9. **No active escalation.** No `ESCALATION_REQUESTED` is in flight for this proposal.
10. **Auto-approval preconditions are auditable.** The runtime emits an `AutoApprovalPreconditionsRecord` (RecordRef) capturing which preconditions held at decision time.

The full set is referenced via:

```json
"autoApprovalPreconditionsRef": {
  "recordId": "aap_record_001",
  "recordType": "AUTO_APPROVAL_PRECONDITIONS",
  "registry": "decision-registry",
  "scopeType": "TENANT",
  "tenantId": "tenant_001"
}
```

If any precondition fails, runtime MUST NOT auto-approve. The proposal proceeds to external review or escalation per deployment policy.

**Calibration scope:** `calibration` applies when the proposal's recommendation is the output of a probabilistic model. For deterministic recommendations (rule-based, code-generated), calibration is not required, and precondition 2 evaluates as N/A (does not block auto-approval). Whether calibration is required is determined by the proposal's `recommendationRef` artifact metadata.

### §13.6 Decision artifact taxonomy

OACP defines two decision artifact subtypes for proposal closure:

| Subtype | `decisionType` | Decider | When used |
|---|---|---|---|
| `ExternalDecisionRef` | `EXTERNAL_APPROVAL`, `EXTERNAL_REJECTION` | Human reviewer or external system | Proposal closure by human/external |
| `RuntimeDecisionRef` | `RUNTIME_AUTO_APPROVAL`, `RUNTIME_REJECTION`, `EXPIRY`, `SUPERSESSION` | Runtime | Proposal closure by runtime/system |

Both are `DecisionRef` (§2c.1a) with specific `decisionType` and additional fields.

**ExternalDecisionRef:**

```json
{
  "decisionId": "ext_dec_001",
  "decisionType": "EXTERNAL_APPROVAL",
  "decidedBy": "reviewer.alice",
  "decidedByType": "EXTERNAL_REVIEWER",
  "decidedAt": "2026-05-05T17:30:00Z",
  "appliesTo": {
    "objectType": "PROPOSAL",
    "proposalRef": {
      "artifactId": "prop_001",
      "artifactType": "PROPOSAL",
      "tenantId": "tenant_001"
    }
  },
  "decisionStatus": "ISSUED",
  "policySnapshotId": "policy-snapshot-001",
  "reviewFeedbackRef": {
    "recordId": "rf_001",
    "recordType": "REVIEW_FEEDBACK",
    "scopeType": "TENANT",
    "tenantId": "tenant_001"
  },
  "tenantId": "tenant_001"
}
```

**RuntimeDecisionRef:**

```json
{
  "decisionId": "rt_dec_001",
  "decisionType": "RUNTIME_AUTO_APPROVAL",
  "decidedBy": "runtime.ojas",
  "decidedByType": "RUNTIME",
  "decidedAt": "2026-05-05T17:11:00Z",
  "appliesTo": {
    "objectType": "PROPOSAL",
    "proposalRef": {
      "artifactId": "prop_001",
      "artifactType": "PROPOSAL",
      "tenantId": "tenant_001"
    }
  },
  "decisionStatus": "ISSUED",
  "policySnapshotId": "policy-snapshot-001",
  "autoApprovalPreconditionsRef": {
    "recordId": "aap_record_001",
    "recordType": "AUTO_APPROVAL_PRECONDITIONS",
    "scopeType": "TENANT",
    "tenantId": "tenant_001"
  },
  "tenantId": "tenant_001"
}
```

**Self-decision rule:** the decider's identity (`decidedBy`) MUST NOT equal the proposal's emitter (`proposedBy.participantId`). An agent cannot decide its own proposal. **Exception:** `RUNTIME` may auto-approve agent-emitted proposals — runtime is a separate participant from the agents whose proposals it governs.

`SELF_DECISION_VIOLATION` rejects same-participant decision emissions.

### §13.6a EXPIRY and SUPERSESSION as decision records

Both `EXPIRED` and `SUPERSEDED` proposal states produce `RuntimeDecisionRef` records:

**Expiry decision:**

```json
{
  "decisionId": "rt_dec_expiry_001",
  "decisionType": "EXPIRY",
  "decidedBy": "runtime.ojas",
  "decidedAt": "2026-05-05T19:10:00Z",
  "appliesTo": {
    "objectType": "PROPOSAL",
    "proposalRef": "prop_001"
  },
  "decisionStatus": "ISSUED",
  "expiryReason": "TTL_ELAPSED",
  "tenantId": "tenant_001"
}
```

**Supersession decision:**

```json
{
  "decisionId": "rt_dec_supers_001",
  "decisionType": "SUPERSESSION",
  "decidedBy": "runtime.ojas",
  "decidedAt": "2026-05-05T17:25:00Z",
  "appliesTo": {
    "objectType": "PROPOSAL",
    "proposalRef": "prop_001"
  },
  "supersededBy": {
    "artifactId": "prop_002",
    "artifactType": "PROPOSAL",
    "tenantId": "tenant_001"
  },
  "decisionStatus": "ISSUED",
  "tenantId": "tenant_001"
}
```

Why this matters: every proposal closure has an auditable decision record. Proposal histories can be reconstructed entirely from decision artifacts without inferring closure from timeline-side effects.

### §13.6b Decision lifecycle (decisionStatus)

A decision artifact, once emitted, has a lifecycle:

```
ISSUED                              decision emitted; effective
    ↓
ACCEPTED                            proposal lifecycle state has reflected this decision
    ↓
    ├── REVOKED                     decision retracted by authorized authority (terminal)
    ├── SUPERSEDED                  later decision overrides this one (terminal)
    └── INVALIDATED                  decision invalidated (e.g., signature key revoked retroactively) (terminal)
```

Reserved `decisionStatus` values: `ISSUED`, `ACCEPTED`, `REVOKED`, `SUPERSEDED`, `INVALIDATED`.

Important rules:

1. **Proposal state transitions only on `ACCEPTED` decisions.** A proposal moves to `EXTERNALLY_APPROVED` only when its associated `ExternalDecisionRef` reaches `ACCEPTED`. An `ISSUED` decision is effective for *audit*, but the proposal's lifecycle does not advance until the decision is `ACCEPTED` by runtime.
2. **Revoking a decision does NOT mutate the prior proposal state.** If a proposal moved to `EXTERNALLY_APPROVED` based on a now-`REVOKED` decision, the proposal's prior state is *not* retroactively reverted. A new decision is required to move the proposal to a new state.
3. **Decision revocation requires the same or higher authority as the original decision.** A reviewer's `EXTERNAL_APPROVAL` can be revoked only by the same reviewer (or runtime/supervisor authority). `DECISION_REVOCATION_AUTHORITY_INVALID` rejects unauthorized revocations.
4. **`INVALIDATED` is for cryptographic invalidation.** Used when a signature key is retroactively compromised (per §3e.9). Distinct from `REVOKED` (deliberate retraction).

### §13.7 Proposal emission readiness

A proposal MAY be emitted when:

1. `proposedBy` has a valid binding for the proposal's `messageType` (gate 3).
2. All `evidenceReportRefs[]`, `validationResultRefs[]`, and `recommendationRef` resolve and pass tenant/profile checks.
3. `riskAssessmentRef` is present at `OACP-Governed`+.
4. `policySnapshotId` is the active policy snapshot (or proposal references the snapshot effective at proposal time).

Receivers reject malformed proposals with `PROPOSAL_INVALID`.

### §13.8 Proposal-to-proposal relationships

The `proposalRefs[]` array on a proposal artifact records its relationships to other proposals:

```json
"proposalRefs": [
  {
    "proposalRef": {
      "artifactId": "prop_001",
      "artifactType": "PROPOSAL",
      "tenantId": "tenant_001"
    },
    "relationship": "RESPONDS_TO_REVIEW"
  }
]
```

Reserved `relationship` values:

| Relationship | Meaning | Mutates prior state? |
|---|---|---|
| `RESPONDS_TO_REVIEW` | New proposal addresses review feedback on the referenced proposal | No — both proposals exist independently |
| `SUPERSEDES` | New proposal supersedes the referenced proposal | Yes — referenced proposal moves to `SUPERSEDED` |
| `REFERENCES` | Informational — references prior proposal for context | No |
| `CORRECTS` | New proposal corrects an error in the referenced proposal | Yes — only via runtime/supervisor authority |

Rules:

1. **Authority requirement.** `SUPERSEDES` and `CORRECTS` mutate the referenced proposal's state. Both require the new proposal to be emitted by `RUNTIME` or authorized `SUPERVISOR`. An agent cannot supersede or correct another agent's proposal. `PROPOSAL_RELATIONSHIP_AUTHORITY_INVALID` rejects unauthorized state-mutating relationships.
2. **State mutation produces a decision.** When `SUPERSEDES` is emitted, runtime emits a `SUPERSESSION` `RuntimeDecisionRef` against the referenced proposal (per §13.6a).
3. **`RESPONDS_TO_REVIEW` does not mutate the prior proposal.** The prior proposal remains in its terminal state (often `EXPIRED` or `REJECTED`). The new proposal exists alongside it.
4. **Tenant scope coherence.** All `proposalRefs[]` MUST be in the same tenant scope as the new proposal.
5. **Conversation scope.** Cross-conversation proposal references are forbidden. `PROPOSAL_REF_CROSS_CONVERSATION` rejects them.

### §13.9 Risk assessment

A `RISK_ASSESSMENT` decision artifact:

```json
{
  "decisionId": "risk_assess_001",
  "decisionType": "RISK_ASSESSMENT",
  "decidedBy": "risk-engine-001",
  "decidedAt": "2026-05-05T17:09:00Z",
  "appliesTo": {
    "objectType": "PROPOSAL_DRAFT",
    "proposalId": "prop_001"
  },
  "result": "LOW_RISK",
  "riskFactors": ["READ_ONLY_OPERATION", "INTERNAL_DATA_CLASS"],
  "policySnapshotId": "policy-snapshot-001",
  "tenantId": "tenant_001"
}
```

Reserved `result` values: `LOW_RISK`, `MEDIUM_RISK`, `HIGH_RISK`, `INELIGIBLE_FOR_AUTO_APPROVAL`.

The risk assessment is consulted by the auto-approval evaluator (per §13.5 precondition 1).

### §13.10 Decision authority verification

When an `ExternalDecisionRef` is received, the runtime verifies:

1. The decider's identity (per §3a) at decision emission time.
2. The decider's authority for this proposal type (`appliesTo.proposalRef`) and tenant.
3. The policy snapshot referenced is the snapshot under which approval was sought.
4. The decision is within TTL of the original proposal.

Failures:

- `DECISION_AUTHORITY_INVALID` — decider not authorized.
- `DECISION_AUTHORITY_REVOKED` — decider's authority was revoked at or before `decidedAt`.
- `DECISION_POLICY_SNAPSHOT_MISMATCH` — decision references a policy snapshot inconsistent with the proposal's policy snapshot.
- `DECISION_AFTER_PROPOSAL_TTL` — decision emitted after proposal `expiresAt`.

### §13.11 Decision-to-proposal correlation

Each decision references its target via `appliesTo`:

```json
"appliesTo": {
  "objectType": "PROPOSAL",
  "proposalRef": {
    "artifactId": "prop_001",
    "artifactType": "PROPOSAL",
    "tenantId": "tenant_001"
  }
}
```

Multiple decisions may target the same proposal across its lifecycle (e.g., a `RISK_ASSESSMENT` early, then an `EXTERNAL_APPROVAL` later). Each is a distinct artifact with its own `decisionId`.

### §13.12 Proposal cancellation

Runtime MAY cancel a `PENDING` or `ESCALATED` proposal at any time before terminal state:

```json
{
  "decisionId": "rt_dec_cancel_001",
  "decisionType": "RUNTIME_REJECTION",
  "decidedBy": "runtime.ojas",
  "decidedAt": "2026-05-05T17:20:00Z",
  "appliesTo": {
    "objectType": "PROPOSAL",
    "proposalRef": "prop_001"
  },
  "rejectionReason": "RUN_CANCELLED",
  "tenantId": "tenant_001"
}
```

The proposal moves to `REJECTED`. Receivers correlate the cancellation via `appliesTo`.

### §13.13 Reserved error codes (§13)

```
PROPOSAL_INVALID                     — proposal artifact malformed
PROPOSAL_LIFECYCLE_INVALID           — state transition violates lifecycle
PROPOSAL_AUTHORITY_INVALID           — proposer lacks binding authority
PROPOSAL_REF_CROSS_CONVERSATION      — proposalRefs[] crosses conversationId
PROPOSAL_REF_TENANT_VIOLATION        — proposalRefs[] in different tenant
PROPOSAL_RELATIONSHIP_AUTHORITY_INVALID — non-RUNTIME/SUPERVISOR attempted state-mutating relationship
PROPOSAL_REF_INVALID                 — proposalRef doesn't resolve
PROPOSAL_TTL_EXCEEDED                — decision emitted after proposal expiresAt
SELF_DECISION_VIOLATION              — decider equals proposer (excluding runtime auto-approval)
DECISION_AUTHORITY_INVALID           — decider not authorized
DECISION_AUTHORITY_REVOKED           — decider's authority revoked at or before decidedAt
DECISION_POLICY_SNAPSHOT_MISMATCH    — decision policy snapshot inconsistent with proposal
DECISION_AFTER_PROPOSAL_TTL          — decision emitted after proposal expired
DECISION_STATUS_INVALID              — decisionStatus value not in registry
DECISION_REVOCATION_AUTHORITY_INVALID — non-original-or-elevated revoked decision
DECISION_DUPLICATE                   — multiple ISSUED decisions targeting same proposal at same lifecycle stage
DECISION_SCOPE_VIOLATION             — appliesTo references object outside decider scope
RISK_ASSESSMENT_MISSING              — required at Governed+ but absent
RISK_ASSESSMENT_INVALID              — risk assessment artifact malformed
AUTO_APPROVAL_PRECONDITION_FAILED    — runtime auto-approved without preconditions met
AUTO_APPROVAL_RECORD_MISSING         — runtime auto-approved without preconditions record
CALIBRATION_MISSING                  — required for probabilistic recommendation
CALIBRATION_METHOD_UNSUPPORTED       — calibrationMethod not in deployment-permitted set
RERUN_AUTHORITY_INVALID              — see §11a.11
```

### §13.14 Conformance profile interactions

| Profile | Proposal/decision behavior |
|---|---|
| `OACP-Core` | Proposal optional; no auto-approval |
| `OACP-Operational` | Proposal lifecycle required; decisions optional |
| `OACP-Proposal` | Full proposal/decision protocol |
| `OACP-Governed` | All rules; auto-approval preconditions enforced; risk assessment required |
| `OACP-Regulated` | All rules; decision artifacts cryptographically signed; auto-approval audit-immutable |

---

## §22. Conformance

This section consolidates per-section conformance requirements into per-profile behavior matrices. Implementations claiming a profile MUST implement all behaviors marked for that profile.

### §22.1 Conformance dimensions

A conformant OACP implementation is evaluated along five dimensions:

| Dimension | Question |
|---|---|
| **Validation** | Does the implementation correctly accept/reject envelopes per §1.2 processing gate? |
| **Emission** | Does the implementation emit envelopes that meet the claimed profile? |
| **Reference resolution** | Does the implementation resolve typed references with tenant scoping (§2c, §2d)? |
| **State machines** | Does the implementation implement broker, review, proposal lifecycle correctly? |
| **Audit** | Does the implementation produce auditable records per profile? |

### §22.2 Per-profile conformance matrix

| Behavior | Core | Operational | Governed | Regulated |
|---|---|---|---|---|
| Identity claim required | Optional (`DECLARED` permitted) | Required, `AUTHENTICATED`+ | Required | Required, `ATTESTED` |
| Verification trail | Optional | Required at hops | Required at all hops | Required, immutable |
| Tenant isolation | If configured | Required when configured | Required, full reference scoping | Required, cryptographically immutable |
| Capability manifest | Optional | Recommended | Required for capability-scoped messages | Required, signed |
| Binding lifecycle | Optional | Optional | Required | Required, signed |
| Authorization decision | Optional | Optional | Required for capability-scoped messages | Required, signed |
| Idempotency keys | Optional | Recommended | Required | Required |
| Replay metadata | Optional | Optional | Required when replaying | Required, signed |
| Hash chain | Optional | SHOULD | Required, validated | Required, hard-blocking |
| Message age limit | Unlimited | 24h default | 1h default | 5min default |
| Signatures | Optional | Optional | Optional | Required on capability-scoped messages |
| Profile claim drift | Permitted | Forbidden | Forbidden | Forbidden |
| Required extensions | Honored | Honored | Honored | Honored, signed |
| Broker state machine | Optional | Recommended for tools | Required for capability-scoped tool calls | Required, signed |
| Review TTL | Recommended | Required | Required | Required |
| Review feedback artifact | Optional | Required for `RESPONDED` | Required, RecordRef-backed | Required, signed |
| Proposal lifecycle | Optional | Required | Required, full lifecycle | Required, signed |
| Auto-approval preconditions | N/A | Optional | Required when auto-approving | Required, signed, audit-immutable |
| Risk assessment | Optional | Optional | Required | Required, signed |
| Cross-tenant policy | Optional | Optional | Required for cross-tenant flows | Required, signed |
| Audit chain | Optional | Recommended | Required | Required, signed, immutable |

### §22.3 Conformance levels

An implementation MAY claim conformance at one or more of the following levels:

- **L1 Producer.** Emits envelopes that pass receiver-side validation at the claimed profile.
- **L2 Receiver.** Validates incoming envelopes correctly per the claimed profile.
- **L3 Mediator.** Forwards envelopes preserving extensions, references, and integrity (per §2b.6a, §2c).
- **L4 Decider.** Emits decision artifacts (`ExternalDecisionRef`, `RuntimeDecisionRef`) per §13.
- **L5 Runtime.** Implements full broker (§10), proposal lifecycle (§13), review (§11a), idempotency (§3c), and audit infrastructure.

A conformant runtime claiming `OACP-Governed` is L5 + L4 + L3 + L2 + L1 at the Governed profile.

### §22.4 Conformance test vectors

Conformance test vectors live in `conformance/` (Pass 2 deliverable). Each vector is a JSON envelope plus expected validation result. Categories:

- Per-section happy-path examples.
- Per-error-code negative tests.
- Profile-boundary tests (envelope at minimum/maximum thresholds).
- Cross-profile tests (Governed envelope to Operational receiver).
- State-machine sequence tests (broker, review, proposal lifecycle).
- Hash-chain integrity tests.
- Tenant-scope violation tests.

Implementations MUST pass all applicable test vectors at their claimed profile to claim conformance.

### §22.5 Out-of-scope from v0.1 conformance

- Inter-implementation interoperability beyond the test vector set.
- Performance benchmarks.
- Transport-binding-specific conformance (left to bindings — Pass 2 deliverable).
- Extension-specific conformance (extensions define their own).

---

## §26. Threat model

This section enumerates threats OACP defends against, threats it does not defend against, and the security properties of each profile. Threats organize by surface, not by attacker type.

### §26.1 Cross-cutting principles

OACP's threat model rests on three principles:

1. **Trust the protocol; verify the participant.** OACP defines what a conformant participant looks like. It does not assume any participant is trustworthy. Receivers verify identity, authority, and integrity at every hop.
2. **The runtime is the trust root, but it is not infallible.** OACP detects many forms of compromise — including attempts to forge messages, replay messages, or bypass authority. It cannot detect a compromised runtime emitting valid messages with full authority. A compromised runtime is a catastrophic failure outside OACP's prevention surface; OACP minimizes the blast radius via tenant isolation, audit chains, and cryptographic attestation at higher profiles.
3. **Auditability ≠ prevention.** Many threats are addressed by audit chains (detection) rather than by validation (prevention). The threat model is explicit about which threats are prevented and which are merely detectable.

### §26.2 Auditability vs. prevention

| Threat type | Prevented by | Detected by |
|---|---|---|
| Forged identity at low profile | None | Runtime audit |
| Forged identity at Governed+ | Identity verification | Verification trail |
| Replay of old message | `maxMessageAgeSeconds` (Operational+) | Audit |
| Replay of valid message | Idempotency keys | Audit |
| Privilege escalation via authority field | Authority matrix gate (§3a.8) | Audit |
| Cross-tenant data exfiltration | Tenant scope gates (§2d) | Audit |
| Decision forgery at Governed | Signature on decisions (Regulated only) | Audit at all profiles |
| Hash chain tampering | Chain validation (Regulated) | Chain validation (Governed, detective) |
| Manifest substitution | Manifest registry signature (Regulated) | Audit |
| Compromised runtime | None | None within the compromised tenant; cross-tenant audit may detect |

### §26.3 Threat surface enumeration

#### §26.3a Identity and impersonation

**Threats:**

- Forged identity claims (claiming to be another agent).
- Stolen credentials (legitimate credential held by attacker).
- Replayed authentication tokens.
- Identity confusion across hops (verification trail tampering).
- Insufficient assurance for sensitive operation.

**Mitigations:**

- Identity verification at receive (gate 1, §3a).
- Per-hop verification recorded in `verificationTrail`.
- Credential expiry (`identity.expiresAt`).
- Verification freshness (`verification.verifiedAt` + `IDENTITY_VERIFICATION_STALE`).
- Profile-driven assurance requirements (`OACP-Governed` rejects `DECLARED`).
- At Regulated: cryptographic attestation (`ATTESTED`).

**Residual risk:** stolen credentials valid within their lifetime cannot be detected by OACP. Out-of-band detection (e.g., behavioral anomaly) is required.

#### §26.3b Capability misuse

**Threats:**

- Agent attempts to use a capability not in its manifest.
- Agent attempts to use a capability outside its binding scope.
- Agent claims a binding that doesn't exist.
- Agent claims a binding with terminal state (revoked, expired).
- Manifest substitution (referenced manifest swapped).

**Mitigations:**

- Capability declared in manifest required (§3b.7).
- Binding evaluation at receive time (§3b.6).
- Terminal-state binding rejection.
- Manifest registry immutability (§3b.3).
- At Regulated: manifest signature.

**Residual risk:** runtime that controls the binding registry can issue a binding outside policy. This is contained by the cross-tenant audit but not prevented within the tenant.

#### §26.3c Authorization bypass

**Threats:**

- Agent emits `authorizationDecision: ALLOW` when policy would have denied.
- Stale authorization decision reused outside scope.
- Authorization decision signed by unauthorized authority.
- `authorizationDecision` block omitted to bypass evaluation.

**Mitigations:**

- `authorizationDecision` required at `OACP-Governed`+ for capability-scoped messages.
- Decision scope binding (§3a.6).
- Decision authority verification at receive.
- Policy snapshot reference for verification.
- At Regulated: decision signature with policy-key binding.

**Residual risk:** if the authorization engine is compromised, decisions it emits are valid by construction. Cross-tenant audit detects abnormal decision rate.

#### §26.3d Replay and freshness

**Threats:**

- Replay of old (but valid) messages to trigger duplicate effects.
- Replay across runs to confuse audit chain.
- Replay of completed proposals to trigger duplicate decisions.
- Replay of decisions to override later ones.

**Mitigations:**

- `maxMessageAgeSeconds` rejection (Operational+).
- Idempotency keys + dedupe windows (§3c).
- Replay metadata required and authority-verified (§3c.7).
- Hash-chain segmentation per run (§3e.7).
- At Regulated: signature timestamp validation against current time.

**Residual risk:** within `dedupeWindowSeconds`, idempotent replay is by design indistinguishable from retry. Operational windows trade safety against availability.

#### §26.3e Audit chain integrity

**Threats:**

- Hash chain breakage to hide messages.
- Forwarded messages with stripped extensions.
- Audit chain segmentation to obscure cross-run causation.
- Runtime that lies about what was emitted.

**Mitigations:**

- `previousMessageHash` validation (§3e.5).
- Extension preservation rule (§2b.6a).
- Cross-run causation via `causation.causationId` and `proposalRefs[]`.
- At Regulated: hash chain hard-blocking, signed.

**Residual risk:** runtime-controlled audit storage can be tampered with. Off-runtime audit aggregation (e.g., immutable storage) is recommended at Regulated; OACP defines the message contract, not the storage layer.

#### §26.3f Tenant isolation breach

**Threats:**

- Cross-tenant message accepted at receiver.
- Cross-tenant reference resolution.
- Identity authorized for one tenant operating in another.
- Cross-tenant flow without policy.

**Mitigations:**

- Four enforcement points (§2d.3).
- `verification.tenantScope` carrying authorized tenants (§2d.7).
- Reference-scope compatibility (§2d.6a).
- Cross-tenant policy required (§2d.6).
- Audit on tenant violations.

**Residual risk:** receiver misconfiguration that authorizes a receiver for tenants beyond intent. Configuration audits required out of band.

#### §26.3g Decision integrity

**Threats:**

- Forged decision artifacts (claiming approval that didn't occur).
- Decision artifacts with stale policy snapshots.
- Self-approval (decider equals proposer).
- Authority elevation in decision.
- Decision retroactive backdating.

**Mitigations:**

- Self-decision rule (§13.6).
- Decision authority verification (§13.10).
- Policy snapshot consistency check (§13.10).
- `decidedAt` clock-skew tolerance.
- Decision lifecycle (§13.6b) — revocation requires equal/elevated authority.
- At Regulated: decision signature, key validity at `decidedAt`.

**Residual risk:** an authorized decider deliberately approving against policy is not prevented — it is detected via audit and out-of-band review.

#### §26.3h Extension abuse

**Threats:**

- Extension with `required: true` claiming envelope-authority semantics.
- Extension that weakens core (e.g., declaring "ignore message age").
- Extension namespace squatting.
- Schema reference SSRF.

**Mitigations:**

- Tier hierarchy enforcement (§2b.1).
- `EXTENSION_WEAKENS_CORE` (§2b.6 rule 4).
- Namespace enforcement at Governed+.
- Trusted schema resolver rule (§2b.6 rule 7).
- Trusted URI resolver rule (§2c.6a).

**Residual risk:** required extensions add validation that increases complexity; bugs in extension handlers are an attack surface. Extension reviews recommended.

### §26.4 Out-of-scope threats (v0.1)

OACP v0.1 explicitly does not defend against:

1. **Compromised runtime** (per §26.1 principle 2).
2. **Compromised authorization engine** within the compromised tenant.
3. **Side-channel attacks** on the runtime infrastructure.
4. **Network-layer attacks** below the OACP envelope (TLS termination, DNS hijacking).
5. **Denial-of-service** at the transport layer.
6. **Model-level attacks** on the agent's reasoning (prompt injection, jailbreak).
7. **Long-term cryptographic compromise** of signature keys via cryptanalysis.
8. **Insider threats with full administrative access** to the runtime.

These are addressed by deployment infrastructure, not by OACP.

The threats above are matters of *prevention*; OACP does not prevent them. Adjacent to that, a separate distinction worth being explicit about: OACP implements operational governance support for alignment oversight, model-safety workflows, and truthfulness-verification processes. OACP does not implement or guarantee model alignment, model safety, or truthfulness. OACP provides governance primitives — identity, authority, capability binding, authorization decisions, tenant scope, tool mediation, review, proposal/decision separation, evidence references, validation results, audit chains, replay protection, and hash-chain integrity — that make these oversight workflows attributable, reviewable, auditable, and policy-enforceable. Future OACP v0.2 candidates strengthen this support through RunIntent gates, reasoning trace evidence, output safety classification, evidence quality scoring, tool side-effect classes, policy simulation, kill-switch semantics, and data handling / purpose limitation. The distinction between "supports the workflow" and "guarantees the property" is load-bearing for any deployment claiming OACP conformance against AI-safety regulation.

### §26.5 Specific threat scenarios

#### §26.5a Compromised runtime

A compromised runtime can:

- Emit valid envelopes claiming any agent identity.
- Issue arbitrary bindings.
- Auto-approve any proposal.
- Forge audit chain entries.
- Suppress messages.

OACP prevents none of this within the compromised tenant. **Mitigations are deployment-level:**

- Cross-tenant audit aggregation (immutable storage outside runtime control).
- Multi-runtime cross-validation (parallel runtimes producing independent audit chains).
- Cryptographic attestation of runtime binary (out of v0.1 scope).
- External signed-decision authority (deciders not under runtime control).

This is a catastrophic failure mode. It defines the limit of OACP's prevention surface.

#### §26.5b Valid-but-malicious participant

A participant with valid credentials and authority, acting against policy intent:

- An agent emits a HIGH-risk proposal it should not have.
- A reviewer approves something they should not have.
- A binding holder uses a tool in ways within scope but against intent.

OACP does not detect intent; it detects *protocol violation*. A compliant-but-malicious participant is detected via policy evaluation outside OACP — risk assessment, anomaly detection, post-hoc audit review. OACP provides the audit substrate for these analyses; it does not perform them.

#### §26.5c Schema/manifest/policy rollback attack

An attacker (or compromised authority) substitutes:

- A schema with weaker validation.
- A manifest with broader capabilities.
- A policy with weaker rules.

**Mitigations:**

- Manifest immutability (§3b.3) — registered manifests cannot be modified.
- Manifest registry signature (Regulated).
- Policy snapshots referenced in decisions; downstream verification compares against the proposal's referenced snapshot.
- Schema resolution through trusted resolvers only (§2b.6 rule 7).

**Residual risk:** if the registry itself is compromised, immutability claims are weakened. Out-of-runtime registry attestation (Regulated) addresses this.

#### §26.5d Confidentiality boundary

OACP does not encrypt envelopes by default. Confidentiality is a transport-layer concern. The protocol assumes:

1. Transport-layer encryption (TLS, mTLS) protects envelope contents in transit.
2. At-rest encryption (storage layer) protects audit chain.
3. PII handling is governed by `dataClasses` on `ArtifactRef` and deployment-level policy.

OACP envelopes contain `dataClasses` to communicate confidentiality requirements but do not enforce them via cryptographic message-layer protection. **`OACP-Regulated` SHOULD layer message-level encryption** but the encryption mechanism is deferred to Tier 4 / v0.2.

#### §26.5e Required-extension downgrade

An attacker (or compromised intermediary) strips a required extension or its `requiredExtensions[]` entry, hoping receivers process the envelope under weaker rules.

**Mitigations:**

- `requiredExtensions[]` is part of the envelope, covered by `envelopeHash` and signature.
- Receivers compare `requiredExtensions[]` to actual `extensions{}` at gate 11.
- `REQUIRED_EXTENSION_DROPPED_BY_INTERMEDIARY` rejects on inconsistency.
- `EXTENSION_PRESERVATION_VIOLATION` flags intermediary drops.
- At Regulated: signature covers both `requiredExtensions[]` and `extensions{}` keys.

**Residual risk:** an intermediary that re-signs envelopes (legitimate use case for sanitization brokers) can drop extensions and re-sign. Authority for re-signing is itself a configuration concern.

#### §26.5f Side-effect classification (partial coverage)

OACP v0.1 partially addresses side-effect threats:

- **Read-only operations** are explicitly classified via `sideEffectClass` on capability declarations (§3b.2).
- **Idempotent operations** are protected via idempotency keys.
- **Non-idempotent side-effecting operations** are governed by binding scope and policy but lack a uniform protocol-level "transaction" or "rollback" mechanism. **Deferred to Tier 4 / v0.2.**

### §26.6 Threat model evolution

The threat model evolves with the protocol:

| Tier | Threat model maturity |
|---|---|
| Tier 1–2 (v0.1) | Single-tenant, single-runtime; basic threats enumerated |
| Tier 3 (this) | Multi-tenant, cross-tenant policy; eight threat surfaces |
| Tier 4 (v0.2) | Quarantine release, side-effect classes, message-level encryption |
| Tier 5 (v0.3+) | Federation, cross-deployment threats, multi-runtime cross-validation |

Each tier may introduce new threats (e.g., federation enables cross-deployment attacks not present at v0.1). The threat model is updated with each major version.

### §26.7 Implementation guidance

Implementors at each profile SHOULD:

- **Operational+:** Implement message-age limits with strict tolerance; treat clock-skew exceptions as alerts.
- **Governed:** Separate authorization engine from runtime where deployment supports it; ensure decision artifacts cannot be self-emitted by runtime under non-RUNTIME identity.
- **Regulated:** Cross-tenant audit aggregation in immutable storage; rotate signing keys with documented retroactive-validity policy.

---

## §27. Versioning and compatibility

This section defines the versioning model for OACP. It distinguishes five versioned axes, sets pre-1.0 vs post-1.0 expectations, and describes lifecycle semantics for each axis.

### §27.1 Five versioned axes

OACP defines five distinct versioning axes. Each evolves on its own cadence under its own rules:

| Axis | Identifier | Meaning |
|---|---|---|
| **Protocol version** | `oacpVersion` | The protocol specification itself |
| **Profile version** | `governance.profileVersion` | A specific profile's semantics |
| **Manifest version** | `manifest.manifestVersion` | A specific capability manifest |
| **Payload schema version** | `payloadType` | The schema of a payload type |
| **Extension version** | Extension key suffix | A specific extension's semantics |

These do not move in lockstep. A new protocol version may be released without changing existing profile versions; a new manifest version is per-subject; payload schemas evolve per payload type.

### §27.2 Protocol version

`oacpVersion` is the version of the OACP specification. Format: `<major>.<minor>` (semver-like, but two-segment).

| Pre-1.0 | Post-1.0 |
|---|---|
| Best-effort backward compatibility | Strict backward compatibility within major |
| Breaking changes per minor allowed | Breaking changes only per major |

**Pre-1.0 (v0.x):** This specification (v0.1) is pre-1.0. Backward compatibility between v0.1 and v0.2 is best-effort, not normative. Breaking changes will be documented in the `CHANGES.md` of each release. Implementations targeting v0.x SHOULD pin to a specific version.

**Post-1.0 (v1.x and later):** Within a major version, all changes MUST be backward-compatible. Breaking changes require a major version bump.

**What constitutes a breaking change:**

- Removing a reserved value from any registry.
- Renaming a reserved field.
- Changing the semantics of a reserved value.
- Adding a required field to an existing message kind.
- Changing the behavior of an existing gate.
- Removing or restructuring a base profile from the registry (§2a.2).

**What does not constitute a breaking change:**

- Adding new reserved values to existing registries.
- Adding new optional fields to existing message kinds.
- Adding new message kinds.
- Adding new capability profiles (§2a.3 — controlled-additive).
- Tightening optional behavior (e.g., adding a recommended profile, where deployments may opt in).

### §27.3 Profile version

Each profile (`OACP-Core`, `OACP-Operational`, `OACP-Governed`, `OACP-Regulated`, plus capability profiles) has its own `profileVersion`.

`governance.profileVersion` is asserted by the sender. Receivers verify compatibility with their supported profile versions. Mismatch: `PROFILE_VERSION_UNSUPPORTED`.

Profile versions evolve more conservatively than the protocol. A profile's normative requirements are stable within a major protocol version. Profile minor versions may add recommended-but-optional behavior; profile patch versions clarify wording without changing requirements.

The base profile registry (§2a.2) is **major-version-closed**. The capability profile registry (§2a.3) is **controlled-additive** — new capability profiles MAY be added in protocol minor versions.

### §27.4 Manifest version

A capability manifest has `manifestVersion` (semver: `<major>.<minor>.<patch>`). Manifest versions are managed by the manifest authority.

**Manifest immutability:** once a manifest version is registered, it MUST NOT be modified. New behavior requires a new manifest version (typically `manifestId` with new version segment, or a new `manifestId` entirely).

A manifest may add capabilities, remove capabilities, narrow capability scopes, or widen capability scopes. Existing bindings reference a specific manifest version; a binding bound to `manifest.v1.0.0` is unaffected by `manifest.v1.1.0`.

**Manifest version compatibility:**

| Change type | Version increment | Effect on existing bindings |
|---|---|---|
| Add new capability | Minor | None (existing bindings unaffected) |
| Remove deprecated capability | Major | Existing bindings using removed capability become invalid |
| Narrow capability scope | Major | Existing bindings using narrowed-out behavior become invalid |
| Widen capability scope | Minor | None (existing bindings continue to work; new capability available to new bindings) |
| Clarify documentation | Patch | None |

### §27.5 Payload schema version

`payloadType` carries an embedded version: `<schema-name>-v<version>`. Examples: `proposal-v1`, `tool-call-request-v2`, `evidence-report-v1`.

Payload schemas evolve under registry control. A payload schema author defines compatibility per schema. Within a major schema version, additions are backward-compatible; required-field additions or semantic changes require major schema version bump.

Receivers MUST validate payloads against the declared `payloadType`. Receivers unable to handle the declared version reject with `PAYLOAD_TYPE_VERSION_UNSUPPORTED`.

### §27.6 Extension version

Extensions carry version in their key suffix: `x-org-finance-v1`, `x-org-finance-v2`. A receiver supporting `x-org-finance-v1` does not automatically support `x-org-finance-v2`.

For required extensions (`requiredExtensions[]`), receivers verifying extension support compare extension keys (including version suffix) exactly. Receivers MUST NOT silently accept a different version of a required extension.

Extensions evolve under their namespace authority's discipline (§2b.7).

### §27.7 Deprecation

OACP recommends — but does not normatively require — a deprecation policy for breaking changes:

| Action | Recommended notice |
|---|---|
| Deprecate a reserved value | 90 days before removal |
| Deprecate a profile | 90 days before removal |
| Deprecate an extension | 90 days before removal |
| Deprecate a manifest | Per manifest authority |
| Deprecate a payload schema | Per schema authority |

**The 90-day window is a recommendation, not a normative requirement.** Major-version transitions may shorten the window for security-critical deprecations. Implementations SHOULD signal deprecation via:

- Audit logs marking deprecated-value usage.
- `lifecycleStatus: DEPRECATED` on registry records.
- Documentation of removal timeline.

### §27.8 Unified six-state lifecycle

OACP uses a unified six-state lifecycle for all versioned registry artifacts (manifests, profile versions, extensions, payload schemas, policies):

| State | New uses | Existing uses |
|---|---|---|
| `ACTIVE` | Permitted | Operational |
| `DEPRECATED` | Permitted with audit warning | Operational |
| `DEACTIVATED` | Rejected | Operational |
| `BLOCKED` | Rejected | **Force-revoked** |
| `RETIRED` | Rejected | Rejected |
| `REVOKED` | Rejected | Rejected (retroactively invalid) |

Distinctions:

- **`DEPRECATED` vs `DEACTIVATED`:** deprecated allows continued use during transition; deactivated stops new use but preserves existing.
- **`DEACTIVATED` vs `BLOCKED`:** deactivated preserves existing operations; blocked force-revokes them (typically for security).
- **`RETIRED` vs `REVOKED`:** retired is end-of-life with no expectation of validity; revoked is retroactive invalidation (e.g., security compromise).

This unified lifecycle applies to:

- Capability manifests (§3b.3).
- Profile versions.
- Extension versions.
- Payload schema versions.
- Policy snapshots.
- Cross-tenant policies.

Each registry independently manages its artifacts' lifecycle states.

### §27.9 Compatibility evaluation

When a receiver evaluates an incoming envelope, compatibility is checked **axis by axis**:

| Axis | Failure code |
|---|---|
| `oacpVersion` not supported | `PROTOCOL_VERSION_UNSUPPORTED` |
| `profileVersion` not supported | `PROFILE_VERSION_UNSUPPORTED` |
| Manifest version not in registry or terminal state | `MANIFEST_NOT_REGISTERED` / `MANIFEST_RETIRED` / `MANIFEST_REVOKED` |
| `payloadType` version not supported | `PAYLOAD_TYPE_VERSION_UNSUPPORTED` |
| Required extension version not supported | `EXTENSION_VERSION_UNSUPPORTED` |

Mismatch on one axis does not cascade to others. A receiver can support `oacpVersion: 0.1` but only specific profile versions, specific manifest versions, etc.

### §27.10 Version negotiation

Mid-flow protocol version negotiation is **deferred to Tier 4 / v0.2**. v0.1 supports only static version assertion: senders declare a version; receivers accept or reject.

Future version negotiation (v0.2+) may include:

- Capability advertising at handshake.
- Negotiated downgrade to common version.
- Mid-run version migration (post-1.0 only, with strict guards).

### §27.11 Reserved error codes (§27)

```
PROTOCOL_VERSION_UNSUPPORTED         — oacpVersion not supported
PROFILE_VERSION_UNSUPPORTED          — profileVersion not supported
MANIFEST_VERSION_UNSUPPORTED         — manifestVersion not supported
PAYLOAD_TYPE_VERSION_UNSUPPORTED     — payloadType version not supported
EXTENSION_VERSION_UNSUPPORTED        — required extension version not supported
LIFECYCLE_STATE_INVALID              — lifecycleStatus value not in registry
LIFECYCLE_STATE_TERMINAL             — operation attempted against terminal-state artifact
DEPRECATION_WINDOW_VIOLATED          — non-blocking warning for early-deprecation use
```

### §27.12 Per-version conformance

A conformance claim names the specific protocol version. An implementation conformant at `OACP v0.1` does not automatically conform at `OACP v0.2`. Conformance test vectors are versioned alongside the protocol.

### §27.13 Out of scope for v0.1

- Mid-flow version negotiation.
- Backward-compatible auto-downgrade.
- Cross-version envelope translation.
- Federation across protocol versions.

These are scoped for Tier 4 / v0.2 or later.

---

## §99. Canonical freeze paragraph

> **OACP is a transport-neutral, contract-first specification of communication mechanics for governed agent runtimes. It defines a typed envelope with deterministic processing semantics: every message passes through twelve governance gates — identity, tenant scope, authority, capability binding, authorization decision, idempotency, broker state, response correlation, causation DAG, conversation scope, payload validation, then dispatch — before any business effect occurs. Profiles compose along two axes: a four-step base hierarchy (Core→Operational→Governed→Regulated) for assurance strictness, and additive capability profiles (Capability, Binding, Reliable, Tool, Review, Proposal, Interop) for feature surface. References are typed (ArtifactRef, MessageRef, RecordRef, DecisionRef, AuditRef) and tenant-scoped by default, with cross-tenant flows requiring explicit registered policy. Proposals are recommendations and never approvals; approvals require separate decision artifacts (ExternalDecisionRef for human reviewers, RuntimeDecisionRef for auto-approval, expiry, supersession) with full audit lineage. Review feedback informs decisions but never finalizes them. Idempotency is operation-keyed across retries; replay is governed and auditable. Integrity is layered into per-message hashes, message-age limits, and run-scoped hash chains, with cryptographic signatures binding canonicalization and algorithm identifiers. Extensions are namespaced and registered, can impose stricter validation, but cannot override or weaken core enforcement. Ojas is the reference implementation.**

---

## End of OACP_SPEC.md

This is the consolidated v0.1 working draft of OACP. It supersedes all prior internal documents and applies all 51 cleanup edits agreed during the Tier 1, Tier 2, and Tier 3.1–3.2 review process.

The supporting publication package — `schemas/`, `bindings/`, `conformance/`, `examples/`, `README.md`, `LICENSE` (Apache 2.0), `SECURITY.md`, `GOVERNANCE.md`, `CONTRIBUTING.md`, and `IPR.md` — accompanies this specification.

The working name "OACP" is used throughout. Public name selection is deferred until v0.1 schemas and conformance vectors are stable.
