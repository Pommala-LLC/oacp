# OACP Schemas

JSON Schema 2020-12 definitions for OACP. The directory holds **two tiers of schemas** organized into subdirectories that serve different validation jobs.

## Layout

```
schemas/
├── README.md                  ← this file
├── core/                      ← envelope-block schemas
│   ├── envelope.schema.json
│   ├── identity.schema.json, verification.schema.json, scope.schema.json
│   ├── governance.schema.json, causation.schema.json, integrity.schema.json
│   ├── reference.schema.json + 5 ref subtypes (artifact-, message-, record-, decision-, audit-)
│   └── proposal.schema.json, capability-manifest.schema.json, binding.schema.json,
│       error-result.schema.json
└── payloads/                  ← payload-type schemas
    ├── oacp-envelope-v1.schema.json
    ├── oacp-error-v1.schema.json
    ├── oacp-handoff-v1.schema.json
    ├── oacp-tool-request-v1.schema.json, oacp-tool-result-v1.schema.json
    ├── oacp-gate-request-v1.schema.json, oacp-gate-result-v1.schema.json
    ├── oacp-evidence-ref-v1.schema.json
    ├── oacp-proposal-v1.schema.json
    └── oacp-escalation-request-v1.schema.json
```

## Conformance to JSON Schema 2020-12

All 27 schemas pass the Draft 2020-12 metaschema check. The dialect is declared in each schema's `$schema` field.

## Two-tier organization

| Tier | Path | Purpose | Naming |
|---|---|---|---|
| **Envelope blocks** | `core/` | Validate OACP envelope structure: identity, scope, governance, references, integrity, etc. | Bare names (`envelope.schema.json`, `identity.schema.json`) |
| **Payload types** | `payloads/` | Validate `payloadType`-specific content carried inside `payload`. | Versioned (`oacp-handoff-v1.schema.json`) |

A complete validation typically composes both: validate the envelope structurally with `payloads/oacp-envelope-v1.schema.json` (which references `core/envelope.schema.json`), then validate `payload` against the schema matching the declared `payloadType` (e.g., `payloads/oacp-handoff-v1.schema.json`).

## `$ref` rules

- **Sibling references inside `core/`** use bare names: `"$ref": "identity.schema.json"`.
- **Cross-tier references from `payloads/` to `core/`** use relative paths: `"$ref": "../core/record-ref.schema.json"`.
- **Self-references inside `payloads/`** (none currently) would use bare names.

`$id` URIs reflect the layout: `https://oacp.dev/schemas/v0.1/core/<name>.schema.json` and `https://oacp.dev/schemas/v0.1/payloads/<name>.schema.json`.

## Core schemas (17 files in `core/`)

### Top-level envelope

| File | Purpose |
|---|---|
| `envelope.schema.json` | The OACP envelope structure. References every other envelope-block schema. |

### Envelope blocks

| File | Purpose |
|---|---|
| `identity.schema.json` | Sender-authored identity claim. Immutable. |
| `verification.schema.json` | Receiver-authored verification record. Appended at hops. |
| `scope.schema.json` | Tenant/workspace/system/cross-tenant scope discriminator. |
| `governance.schema.json` | Profile claims, profile version, capability profiles, risk class. |
| `causation.schema.json` | Causation block: type, id, ids array. INITIATOR omits ids. |
| `integrity.schema.json` | Per-message hashes, hash chain, signature reference. |

### Reference subtypes

| File | Purpose |
|---|---|
| `reference.schema.json` | Parent abstraction over the five Reference subtypes (`oneOf`). |
| `artifact-ref.schema.json` | ArtifactRef — externally-resolvable content artifact. |
| `message-ref.schema.json` | MessageRef — references an OACP message. |
| `record-ref.schema.json` | RecordRef — references a registered control-plane record. |
| `decision-ref.schema.json` | DecisionRef — references a decision artifact. |
| `audit-ref.schema.json` | AuditRef — references an audit-chain entry. |

### Artifact schemas

| File | Purpose |
|---|---|
| `proposal.schema.json` | Resolved proposal artifact (the body fetched via `proposalRef`). |
| `capability-manifest.schema.json` | Registered capability manifest. |
| `binding.schema.json` | Capability binding artifact. |
| `error-result.schema.json` | Structured error return from gate failures. |

## Payload schemas (10 files in `payloads/`)

These validate `payload` content for messages declaring the matching `payloadType`. Each is named `oacp-<payload-type>-v<version>.schema.json` and is intended to be referenced by transport bindings, code generators, and conformance validators that need payload-version-aware schemas.

| File | `payloadType` | Used in `messageType` (typical) |
|---|---|---|
| `oacp-envelope-v1.schema.json` | (envelope wrapper, not a payloadType) | Any v0.1 message |
| `oacp-error-v1.schema.json` | `error-v1` | `ERROR_EMITTED` |
| `oacp-handoff-v1.schema.json` | `handoff-v1` | `HANDOFF_REQUESTED`, `HANDOFF_ACCEPTED`, `HANDOFF_COMPLETED`, etc. |
| `oacp-tool-request-v1.schema.json` | `tool-request-v1` | `TOOL_CALL_REQUESTED` |
| `oacp-tool-result-v1.schema.json` | `tool-result-v1` | `TOOL_RESULT_EMITTED` |
| `oacp-gate-request-v1.schema.json` | `gate-request-v1` | `GATE_REQUESTED` |
| `oacp-gate-result-v1.schema.json` | `gate-result-v1` | `GATE_RESULT_EMITTED` |
| `oacp-evidence-ref-v1.schema.json` | (constrained ArtifactRef profile, not a payloadType) | Used as a typed-reference profile in payloads |
| `oacp-proposal-v1.schema.json` | `proposal-v1` | `PROPOSAL_EMITTED` |
| `oacp-escalation-request-v1.schema.json` | `escalation-request-v1` | `ESCALATION_REQUESTED` |

### Why both tiers exist

Envelope-block schemas validate that a message has the right **shape** as an OACP envelope — every required field, every reserved value, every cross-field constraint that JSON Schema can express. Payload-type schemas validate that the **content** inside `payload` matches what the declared `payloadType` requires.

A receiver typically does both:

1. Parse the envelope.
2. Validate against `payloads/oacp-envelope-v1.schema.json` (which composes `core/envelope.schema.json` via `$ref`).
3. Read `payloadType` from the validated envelope.
4. Look up the matching payload-type schema (e.g., `payloads/oacp-handoff-v1.schema.json`).
5. Validate `payload` against it.

Receivers MUST reject envelopes that fail at step 2 with a structural error code; receivers MUST reject payloads that fail at step 5 with `PAYLOAD_VALIDATION_FAILED`.

### Special cases

- **`payloads/oacp-envelope-v1.schema.json`** is technically not a `payloadType` value — it's a versioned profile of the envelope itself. It exists in this tier because it's named with the `oacp-…-v1` convention and consumed by code generators that target a specific OACP envelope version.
- **`payloads/oacp-evidence-ref-v1.schema.json`** is a constrained `ArtifactRef` profile (with `artifactType` pinned to `EVIDENCE_REPORT`), not a payload type. It's intended as a typed-reference target in payloads and validators that need to assert "this reference is specifically an evidence reference."

## What's NOT in these schemas

These schemas validate **structural conformance** at message receipt. They do **not** encode:

- **Cross-field validation** beyond what JSON Schema 2020-12 can express (e.g., "tenantId in scope must match tenantId in identity verification").
- **Tenant scope enforcement** — runtime check, not schema.
- **Hash verification** — schema requires the field; runtime verifies the hash matches content.
- **Authority matrix** — schema cannot express "AGENT cannot emit BINDING messages"; that's gate logic.
- **State machine transitions** — schema validates a single message, not a sequence.

The full set of validations is described in `OACP_SPEC.md` §1.2. Schemas cover gate steps 1, 4 (partial), 11 (partial), and 14 (payload validation). Other gates require runtime evaluation.

## Strictness

Schemas are **moderately strict**:

- Required fields are required.
- Enum values match spec-reserved sets.
- `additionalProperties: false` only where the spec is closed; most blocks accept additional fields for forward compatibility.

Implementations claiming higher profiles (`OACP-Governed`, `OACP-Regulated`) layer **stricter** validation on top of these schemas — for example, at `OACP-Governed`, `verification.tenantScope` is required even though the schema marks it optional. The schema reflects "valid OACP" structurally; profile rules add stricter constraints.

## Versioning

Per spec §27.5:

- **Envelope-block schemas** (`core/`) are versioned with the protocol (v0.1).
- **Payload-type schemas** (`payloads/`) are versioned independently per payload type (`-v1` suffix). Future payload versions (`-v2`) are added alongside, not replacing.

When the protocol bumps to v0.2, `core/` schemas may evolve; `payloads/` schemas evolve only when their content does.

## Validation tooling

Any JSON Schema 2020-12 validator works. Examples:

- **Python:** `jsonschema` library with `Draft202012Validator` and `referencing.Registry` for cross-schema `$ref`.
- **JavaScript:** `ajv` with `2020` build.
- **Go:** `xeipuuv/gojsonschema` (verify draft support) or `santhosh-tekuri/jsonschema/v5`.
- **Java:** `networknt/json-schema-validator`.

A reference validation script using the registry pattern is in `../conformance/` (see runner spec).

## Schema URIs

Each schema declares a stable `$id` under `https://oacp.dev/schemas/v0.1/core/` or `https://oacp.dev/schemas/v0.1/payloads/`. These URIs are illustrative for v0.1; before v1.0, they will resolve to actual schema documents at the published location. Implementations SHOULD NOT make live HTTP requests — schemas should be bundled or cached.

## Total counts

- **Core schemas:** 17 (in `core/`).
- **Payload schemas:** 10 (in `payloads/`).
- **Total:** 27 schemas, all validated against Draft 2020-12 metaschema.
