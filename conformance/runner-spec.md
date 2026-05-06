# OACP Conformance Runner Specification

This document specifies how a conformance runner SHOULD work. It does not constrain implementation language or tooling.

## What a runner does

A conformance runner takes:

1. A **system under test (SUT)** — an OACP receiver or runtime claiming a profile.
2. A **vector set** — JSON files in `vectors/`.

For each vector:

1. Configures the SUT with any vector-required state (`receiverScope`, etc.).
2. Submits the vector's envelope (or sequence of envelopes) to the SUT.
3. Compares the SUT's actual outcome to the vector's `expected` outcome.
4. Records pass/fail.

After all vectors:

5. Produces a conformance report.

## Runner-SUT interface

The runner needs a way to submit envelopes and observe outcomes. There is no normative protocol for this; in practice:

- **In-process:** the SUT exposes a function `process(envelope) -> result`. Result is `{accepted: true}` or `{accepted: false, errorCode: "..."}`.
- **HTTP:** the SUT exposes a `POST /process` endpoint per the OpenAPI binding (see `../bindings/openapi.md`). Status code + body are mapped to outcomes.
- **Other:** any binding the SUT supports.

The runner abstraction is parameterized over the SUT interface.

## Outcome comparison

For each vector, the runner determines pass/fail:

| Vector `expected` | SUT result | Pass condition |
|---|---|---|
| `accept` | accepted | Pass |
| `accept` | rejected | **Fail** (regression) |
| `reject` | rejected with matching `errorCode` | Pass |
| `reject` | rejected with different `errorCode` | **Fail** (wrong error) |
| `reject` | accepted | **Fail** (security / governance failure) |

For sequence vectors, each step is checked in order; the sequence passes only if every step passes AND the final state matches `expectedFinalState`.

## Profile filtering

Each vector declares its profile (`OACP-Core`, etc.). A runner targeting an `OACP-Operational` SUT runs:

- All `OACP-Core` vectors (the SUT inherits Core requirements).
- All `OACP-Operational` vectors.
- May also run `OACP-Governed` vectors with `expected: reject` due to profile mismatch (the SUT MUST reject envelopes claiming higher profiles per §2a.6 rule 2).

A runner targeting an `OACP-Governed` SUT runs Core, Operational, and Governed vectors. Regulated vectors are run only if the SUT claims Regulated.

## Receiver scope configuration

Some vectors require the SUT to be configured with specific authorized scope (e.g., `authorizedTenants: ["tenant_001"]`). The vector declares this in `receiverScope`. Runners apply this configuration before submitting the envelope.

If the SUT cannot be reconfigured per vector, the runner SHOULD use a fixed configuration that's consistent with the majority of vectors and skip those that require different configuration (recording skipped vectors in the report).

## Sequence vectors

Sequence vectors test multi-message flows (proposal lifecycle, broker state machine, etc.). The runner submits each step in order and checks that each step's outcome matches the step's `expected`. Some steps depend on prior steps' outcomes (e.g., a proposal lifecycle test references the proposalId from the first step's emission).

Runners SHOULD support **state carry-over** within a sequence: values like `proposalId`, `messageId`, `bindingId` emitted by step N are referenced by step N+1.

## Conformance report

The runner produces a structured report:

```json
{
  "runner_version": "0.1.0",
  "sut_identifier": "Ojas v0.5.2",
  "claimed_profile": "OACP-Governed",
  "vector_set_version": "v0.1",
  "ran_at": "2026-05-05T18:00:00Z",
  "summary": {
    "total": 250,
    "passed": 245,
    "failed": 3,
    "skipped": 2
  },
  "results": [
    {
      "vector_id": "OACP-V0.1-CV-0001",
      "profile": "OACP-Operational",
      "result": "pass",
      "duration_ms": 4
    },
    {
      "vector_id": "OACP-V0.1-CV-0042",
      "profile": "OACP-Governed",
      "result": "fail",
      "duration_ms": 6,
      "expected": "reject",
      "expectedErrorCode": "MESSAGE_TENANT_VIOLATION",
      "actualErrorCode": "AUTHORIZATION_DENIED",
      "details": "SUT rejected for the wrong reason."
    }
  ]
}
```

The report is consumable by humans (for debugging) and machines (for CI gating).

## Reference runner

A reference runner is implemented as part of the Ojas project (location TBD). Third-party runners are welcome and encouraged.

A runner is conformant when it:

1. Reads vector files in the format described in `README.md`.
2. Submits each vector's envelope per its sequence semantics.
3. Compares outcomes per the rules above.
4. Produces a report in the schema above (or a compatible superset).

## What a runner does NOT verify

- **Schema validation.** The runner submits envelopes; the SUT validates against schemas. The runner doesn't replicate schema validation logic.
- **Transport-binding correctness.** A runner exercises the SUT's API (in-process function, HTTP endpoint, etc.); transport conformance is separate.
- **Performance.** Latency is observed but not gated.
- **Implementation-internal behavior.** The runner is black-box.

## Self-conformance

A runner SHOULD also conform to its own claimed runner version. Specifically:

- Runner version 0.1.0 reads v0.1 vector format.
- Runner version 0.1.0 submits envelopes in v0.1 envelope format.
- A runner that updates with a new vector format bumps its own version.

## Open items

- **Mutation testing:** vector mutators that systematically alter fields to test edge cases. Plausible for v0.2.
- **Property-based testing harness:** generate envelopes within a profile's constraints and verify the SUT accepts them. Plausible for v0.2.
- **Conformance certification:** formal certification process (organization certifies conformance reports). Deferred to v1.0 governance transition.
