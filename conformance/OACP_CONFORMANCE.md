# OACP Conformance

This directory contains the conformance test vector suite, the conformance runner specification, and a small set of top-level alias examples for quick validator use.

## What conformance means in OACP

A conformant OACP implementation correctly implements the protocol at one or more **profiles** (`OACP-Core`, `OACP-Operational`, `OACP-Governed`, `OACP-Regulated`) and one or more **roles** (Producer, Receiver, Mediator, Decider, Runtime — see spec §22.3).

Conformance is evaluated by:

1. **Schema conformance** — envelopes valid against the JSON Schema 2020-12 definitions in `../schemas/`.
2. **Test vector conformance** — the implementation accepts vectors marked `expected: accept` and rejects vectors marked `expected: reject` (with the specified error code).
3. **Profile-specific conformance** — additional validation behaviors required by the claimed profile per spec §22.2.

This directory provides materials for (2). Schema conformance (1) is handled by any standard JSON Schema 2020-12 validator. Profile-specific conformance (3) is partially covered by vectors in `vectors/profile-boundary/`.

## File structure

```
conformance/
├── OACP_CONFORMANCE.md             ← this file
├── runner-spec.md                  ← how a conformance runner should work
├── valid-envelope.json             ← raw envelope alias: minimal valid OACP-Operational envelope
├── invalid-missing-run-id.json     ← raw envelope alias: missing run.runId at OACP-Operational
├── invalid-undeclared-handoff.json ← raw envelope alias: handoff to a target not in allowedHandoffs
├── invalid-proposal-claims-approval.json  ← raw envelope alias: agent self-approving its own proposal
└── vectors/
    ├── happy-path/                 ← envelopes that should be accepted
    ├── error-codes/                ← envelopes that should be rejected with specific error codes
    ├── profile-boundary/           ← envelopes at minimum/maximum profile thresholds
    ├── state-machine/              ← multi-message sequences testing state transitions
    ├── hash-chain/                 ← integrity and hash-chain integrity tests
    └── tenant-scope/               ← tenant-isolation violations
```

## Two tiers of conformance examples

This directory holds **two tiers** of conformance examples:

### Top-level raw-envelope aliases (4 files)

Plain JSON envelopes intended for quick consumption by:

- Human readers wanting to see canonical valid/invalid shapes.
- Simple validators that don't want to parse the runner-vector wrapper.
- Documentation generators that want to inline canonical examples.

These are NOT vector-format files — no `id`, no `expected`, no `receiverScope` wrapper. They are just the envelopes:

| File | Expected outcome | What it tests |
|---|---|---|
| `valid-envelope.json` | Accept at OACP-Operational | A minimal, structurally complete envelope |
| `invalid-missing-run-id.json` | Reject (`RUN_ID_MISSING`) | run.runId missing where required at Operational+ |
| `invalid-undeclared-handoff.json` | Reject | Handoff to a target not in source's manifest's allowedHandoffs |
| `invalid-proposal-claims-approval.json` | Reject (`SELF_DECISION_VIOLATION`) | Agent self-approving its own proposal in the proposal payload |

### Runner-format vectors (`vectors/`)

Wrapper format with `id`, `description`, `profile`, `receiverScope`, `envelope`, `expected`, `expectedErrorCode`, `expectedGateStep`. Designed for a conformance runner per `runner-spec.md`. See `vectors/` for the categorized vector set.


## Vector format

Each vector is a JSON file with this shape:

```json
{
  "id": "OACP-V0.1-CV-0001",
  "section": "§3a",
  "description": "Happy path: AUTHENTICATED identity with valid credential",
  "profile": "OACP-Operational",
  "envelope": { "...": "OACP envelope here" },
  "expected": "accept",
  "expectedErrorCode": null,
  "expectedGateStep": null,
  "notes": "Optional clarifying notes"
}
```

For rejection vectors:

```json
{
  "id": "OACP-V0.1-CV-0042",
  "section": "§2d",
  "description": "Tenant violation: receiver authorized for tenant_001, envelope claims tenant_002",
  "profile": "OACP-Governed",
  "envelope": { "...": "OACP envelope here" },
  "expected": "reject",
  "expectedErrorCode": "MESSAGE_TENANT_VIOLATION",
  "expectedGateStep": 1.5,
  "receiverScope": {
    "authorizedTenants": ["tenant_001"]
  },
  "notes": "Receiver MUST reject before further validation"
}
```

For sequence vectors (state machine, hash chain):

```json
{
  "id": "OACP-V0.1-CV-0100",
  "section": "§13.4",
  "description": "Proposal lifecycle: PENDING → EXTERNALLY_APPROVED",
  "profile": "OACP-Governed",
  "sequence": [
    { "envelope": "...", "expected": "accept" },
    { "envelope": "...", "expected": "accept" }
  ],
  "expectedFinalState": {
    "proposalId": "prop_001",
    "proposalLifecycleState": "EXTERNALLY_APPROVED"
  }
}
```

## Vector ID convention

Vector IDs follow `OACP-V<protocol-version>-CV-<sequence>`:

- `OACP-V0.1-CV-0001` — first conformance vector for v0.1.
- Numbers are not contiguous; gaps are intentional to allow insertion.

## Sample vectors

This v0.1 distribution includes a small set of sample vectors, one per category, demonstrating the format. The full conformance suite is expected to grow substantially through Tier 3 finalization and into v1.0.

| Sample | Category | Section |
|---|---|---|
| `vectors/happy-path/CV-0001-basic-event.json` | Happy path | §1, §3a |
| `vectors/error-codes/CV-0042-tenant-violation.json` | Error codes | §2d |
| `vectors/error-codes/CV-0050-self-decision.json` | Error codes | §13.6 |
| `vectors/profile-boundary/CV-0080-message-age.json` | Profile boundary | §3e.3 |
| `vectors/state-machine/CV-0100-proposal-lifecycle.json` | State machine | §13.4 |
| `vectors/hash-chain/CV-0120-broken-chain.json` | Hash chain | §3e.5 |
| `vectors/tenant-scope/CV-0140-cross-tenant-no-policy.json` | Tenant scope | §2d.6 |

## What's NOT in v0.1 conformance

- **Performance benchmarks.** Conformance is correctness, not throughput.
- **Inter-implementation interoperability tests** beyond the vector set.
- **Transport-binding-specific conformance.** Each binding documents its own conformance expectations.
- **Extension-specific conformance.** Extensions define their own.

## Versioning

Vectors are versioned with the protocol. v0.1 vectors test v0.1 envelopes. When the protocol bumps to v0.2, the vector set is reviewed and updated; vectors no longer applicable are retired.

## Running the vectors

See [`runner-spec.md`](./runner-spec.md) for the runner specification. A reference runner is part of the Ojas reference implementation; third-party runners are welcome.

## Contributing vectors

Conformance vectors are a high-leverage contribution. See [`../CONTRIBUTING.md`](../CONTRIBUTING.md). Good vectors are:

- **Minimal.** Test one thing.
- **Self-contained.** All required context in the vector file.
- **Profile-explicit.** Declare the receiver profile.
- **Spec-referenced.** Cite the section(s) being tested.
- **Result-explicit.** Specify expected outcome and error code (if any).
