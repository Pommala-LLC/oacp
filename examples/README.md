# OACP Examples

This directory contains end-to-end example flows that illustrate how OACP messages compose into governed agent interactions. Each example walks through one realistic scenario.

These examples are **illustrative**, not normative. They use canonical-looking values (e.g., `tenant_001`, `agent.classifier`) for clarity. Real deployments use deployment-specific identifiers.

## How to read these examples

Each example file:

1. Describes the scenario in one paragraph.
2. Walks through the message sequence, showing envelopes inline.
3. Annotates each message with the gate steps it exercises and the spec sections it illustrates.
4. Notes the architectural invariants the example demonstrates.

Examples are not minimal. They include realistic envelope content (full `from` blocks, governance metadata, integrity blocks where applicable) so you can see what a complete envelope looks like in context. For minimal envelopes used in conformance testing, see [`../conformance/vectors/`](../conformance/vectors).

## Available examples

The directory holds two kinds of examples that serve different audiences.

### Markdown walkthroughs (long-form, narrative)

| File | Scenario | Profile |
|---|---|---|
| [`01-basic-proposal.md`](./01-basic-proposal.md) | An agent emits a proposal; runtime auto-approves under the ten preconditions. | `OACP-Governed` |
| [`02-tool-call-with-broker.md`](./02-tool-call-with-broker.md) | An agent calls a governed tool; the broker sanitizes the payload before dispatch. | `OACP-Governed` |
| [`03-review-needs-more-evidence.md`](./03-review-needs-more-evidence.md) | A reviewer responds with NEEDS_MORE_EVIDENCE; runtime starts a new run in the same conversation. | `OACP-Governed` |
| [`04-cross-tenant-policy.md`](./04-cross-tenant-policy.md) | A regulator-tenant audit reads from operational tenants under explicit cross-tenant policy. | `OACP-Regulated` |
| [`05-replay-original-policy.md`](./05-replay-original-policy.md) | Runtime replays an old message under ORIGINAL_POLICY mode for diagnostic comparison. | `OACP-Governed` |

The walkthroughs are for readers learning OACP. They show envelope sequences with annotated commentary on which gate steps and spec sections each message exercises.

### Standalone JSON examples (short, machine-readable)

| File | Payload type | Profile |
|---|---|---|
| [`handoff-example.json`](./handoff-example.json) | `handoff-v1` | `OACP-Governed` |
| [`tool-request-example.json`](./tool-request-example.json) | `tool-request-v1` | `OACP-Governed` |
| [`proposal-example.json`](./proposal-example.json) | `proposal-v1` | `OACP-Governed` |
| [`error-example.json`](./error-example.json) | `error-v1` | `OACP-Governed` |

The JSON examples are for tooling: code generators, schema validators, integration tests, transport-binding implementations. Each is a complete, valid OACP envelope that passes Draft 2020-12 validation against `oacp-envelope-v1.schema.json` AND its declared payload schema.

These examples are **not minimal**. They include the full envelope structure (identity, verification, governance, run, causation, authorization decision) so receivers see a realistic shape. For minimal envelopes used in conformance test fixtures, see [`../conformance/vectors/`](../conformance/vectors).

## Prerequisites for understanding examples

Read these spec sections first:

- §1 (Conventions) — the envelope shape and processing gate.
- §2a (Profiles) — what `OACP-Governed` means.
- §2c (Reference model) — what `ArtifactRef`, `MessageRef`, etc. are.
- §3a (Identity) — `from`, `verification`, `authorizationDecision`.
- §3d (Conversation/causation) — `runId`, `conversationId`, `causation`.
- §13 (Proposal) — proposal lifecycle and decision artifacts.

The examples assume familiarity with these.

## What examples are NOT

- They are not test vectors. Use [`../conformance/vectors/`](../conformance/vectors) for testing implementations.
- They are not complete reference implementations. They show envelope sequences, not runtime code.
- They are not exhaustive. Many OACP features (cross-tenant policy, regulated profile signature handling, complex fan-in) are illustrated minimally.
- They are not benchmarks. Performance characteristics depend on deployment infrastructure.

## Domain naming conventions in examples

For consistency across examples, the following conventions are used:

| Identifier pattern | Example |
|---|---|
| Tenants | `tenant_001`, `tenant_002`, `regulator-tenant` |
| Workspaces | `workspace_001` |
| Runs | `run_001`, `run_002` |
| Conversations | `conv_001` |
| Agents | `agent.classifier`, `agent.notifier`, `agent.layout-extractor` |
| Runtime | `runtime.ojas` |
| Brokers | `broker.tool` |
| Reviewers | `reviewer.alice`, `reviewer.bob` |
| Manifests | `manifest.agent.classifier.v1` |
| Bindings | `bind_001`, `bind_002` |
| Proposals | `prop_001`, `prop_002` |
| Decisions | `ext_dec_001`, `rt_dec_001`, `authz_001` |

These are illustrative, not normative. Real deployments use whatever naming scheme they prefer.

## Contributing examples

New examples that illustrate features not covered by the existing five are welcome. Good examples:

- Cover one scenario clearly.
- Include enough envelope content to be realistic.
- Annotate each step with spec section references.
- End with a "what this demonstrates" summary.

See [`../CONTRIBUTING.md`](../CONTRIBUTING.md).
