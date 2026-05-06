# Contributing to OACP

Thanks for considering a contribution to OACP. This document describes what kinds of contributions are most useful at the v0.1 working-draft stage and how to submit them.

## What we're looking for at v0.1

OACP v0.1 is a working draft. The most valuable contributions are:

| Contribution type | What it looks like |
|---|---|
| **Implementation feedback** | "I tried to implement §X and ran into Y." Concrete problems from real implementation attempts. |
| **Specification ambiguity reports** | "§X is ambiguous; here are two reasonable interpretations and the case where they diverge." |
| **Threat model gaps** | "§26 doesn't address threat Z; here's the scenario." |
| **Error code coverage** | "There's no error code for situation X; here's a reasonable name and definition." |
| **Conformance test vectors** | New test vectors covering edge cases not in the current set. |
| **Binding extensions** | Bindings to transports beyond CloudEvents/AsyncAPI/OpenAPI/OpenTelemetry. |
| **Reference-implementation experience** | "Implementing this in [language/framework] required X workaround." |

Less useful at this stage:

- Style suggestions for the spec (we'll do a copyediting pass before v1.0).
- Bikeshedding the working name "OACP" (public name is deferred by design).
- Proposals to remove frozen invariants (see [`GOVERNANCE.md`](./GOVERNANCE.md) — those go through invariant-modification process).
- New capability profiles (controlled-additive registry — propose them, but expect higher scrutiny).

## Before contributing

1. Read [`README.md`](./README.md) for the architectural fundamentals.
2. Read the relevant section(s) of [`OACP_SPEC.md`](./OACP_SPEC.md) before proposing a change to them.
3. Read [`GOVERNANCE.md`](./GOVERNANCE.md) for how decisions are made.
4. For security findings: read [`SECURITY.md`](./SECURITY.md). Don't open a public issue.

## How to submit a contribution

Pre-1.0, the contribution flow is:

1. **Open a discussion** describing the issue or proposal. For specification questions, "what does §X mean in case Y?" is a good opener.
2. **Wait for triage.** The maintainers will respond with one of:
   - "This is in scope; please open a PR."
   - "This is in scope but deferred to Tier 4."
   - "This is out of scope; here's why."
   - "We need more information."
3. **Submit a PR** if invited. Include rationale, the affected sections, and what tests or vectors validate the change.

For trivial corrections (typos, broken links, copy-paste errors), feel free to open a PR directly.

## What goes in a good PR

A good PR for a specification change includes:

- The motivation in one paragraph.
- The change itself, scoped narrowly.
- Updates to **all** affected places: spec body, schemas, conformance vectors, examples.
- A note in `CHANGES.md` (we'll help draft this if needed).
- For breaking changes pre-1.0: explicit statement that the change is breaking and what migrations existing implementations need.

PRs that leave the package in an inconsistent state (e.g., spec updated but schema not) will be requested for revision.

## Spec-changes that violate frozen invariants

Some changes touch the **frozen invariants** listed in [`README.md`](./README.md). These require a structured argument before they're considered:

1. Which invariant is at stake.
2. What concrete scenario reveals a conflict between the invariant and a real-world need.
3. What alternative invariant resolves the conflict.
4. What the migration story is for existing implementations.

Casual proposals to relax invariants ("can we just allow agents to approve their own proposals?") will be declined.

## Schema changes

Schema changes follow the same process as spec changes plus:

- Updated `$schema` (we use JSON Schema 2020-12).
- Backward-compat analysis: does this break existing valid envelopes?
- Updated conformance vectors that exercise the change.

For new optional fields: typically straightforward. For new required fields, removed fields, or changed semantics: schema-versioning rules from §27 apply.

## Conformance vector contributions

Conformance vectors are a high-leverage contribution. Each vector is:

- A JSON envelope (input).
- An expected validation result (pass/fail with specific error code).
- A short description of what's being tested.
- A reference to the spec section(s) the vector covers.

Vectors should be minimal — testing one thing — and should specify the receiver profile they assume.

The current vector set is in `conformance/vectors/`. Categories include happy-path, per-error-code, profile-boundary, cross-profile, state-machine sequence, hash-chain integrity, tenant-scope violation. New categories welcome.

## Binding extensions

Bindings document how OACP envelopes ride on a specific transport. The current set:

- CloudEvents
- AsyncAPI
- OpenAPI
- OpenTelemetry

Bindings to other transports (gRPC, WebSocket, MQTT, Kafka topics, etc.) are welcome. A good binding doc covers:

- How the envelope is serialized on the transport.
- How transport-level metadata maps to OACP envelope fields (or doesn't).
- How transport-level features interact with OACP semantics (e.g., transport-level retries vs. OACP idempotency).
- Profile compatibility — which OACP profiles this binding supports cleanly, which require additional compensation.

## Code of conduct

Be professional. The OACP community is small at v0.1; the bar is courtesy and focus on the technical work. We expect the same standards of conduct that any healthy technical community would expect.

We don't have a formal code of conduct document yet. Expect one before v1.0 as the community grows.

## License

By contributing, you agree your contributions are licensed under the [Apache License 2.0](./LICENSE) — the same license OACP itself is published under. Apache 2.0's contribution clause (Section 5) governs.

## Thanks

OACP becomes more valuable with each implementation that exists, each ambiguity that's surfaced, each threat that's enumerated, and each vector that's tested. The most useful contribution at v0.1 is *engagement that produces a clearer specification*. Everything else follows from that.
