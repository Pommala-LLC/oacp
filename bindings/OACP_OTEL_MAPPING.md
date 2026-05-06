# OACP Binding: OpenTelemetry

This document describes how OACP runs and messages produce OpenTelemetry traces, metrics, and logs. Unlike the other bindings (CloudEvents, AsyncAPI, OpenAPI), OpenTelemetry is not a transport — it's an observability framework. The "binding" describes how OACP semantics surface in OpenTelemetry artifacts.

## Status

Draft for v0.1.

## Mapping principle

OACP is deeply observable by design. Every gate produces an audit-worthy event; every run produces a hash-chain-validated trace. OpenTelemetry is the natural export target. The mapping is straightforward:

- **OACP runs** → OpenTelemetry **traces**.
- **OACP messages** and **gate steps** → **spans**.
- **OACP profile claims, decisions, and errors** → **span attributes** and **events**.
- **OACP audit-chain entries** → **log records** (correlated with spans).

OpenTelemetry does not replace OACP audit. OACP audit is the authoritative governance record; OpenTelemetry is a **secondary observability surface** for operational use.

## Trace mapping

### Trace = Run

An OACP `runId` corresponds to one OpenTelemetry **trace ID**. The trace begins at the run's `INITIATOR` message and ends at the run's terminal message (`RUN_COMPLETED`, `RUN_FAILED`, `RUN_CANCELLED`).

The OpenTelemetry trace ID and OACP `correlationId` are typically aligned. Runtimes SHOULD set `correlationId` to the OpenTelemetry trace ID when emitting trace context.

### Span = Message or gate step

Two granularities are useful:

- **Message-level span:** one span per OACP message. Span name follows `oacp.<messageKind>.<messageType>` (e.g., `oacp.proposal.proposal-emitted`). Span attributes carry OACP envelope metadata.
- **Gate-level span (optional):** child spans for each of the twelve processing gates. Useful for performance profiling but verbose; deployments may opt in.

Span hierarchy follows OACP causation:

- Root span: the run's INITIATOR message.
- Each subsequent message's span has a parent set from `causation.causationId`.
- Fan-out: multiple sibling spans share a parent.
- Fan-in: span links (OpenTelemetry's mechanism for non-parent causal relationships) reference all parents in `causationIds[]`.

## Span attributes

Reserved span attributes for OACP messages:

| Attribute | Source | Type |
|---|---|---|
| `oacp.message_id` | `messageId` | string |
| `oacp.message_kind` | `messageKind` | string |
| `oacp.message_type` | `messageType` | string |
| `oacp.tenant_id` | `scope.tenantId` | string |
| `oacp.run_id` | `run.runId` | string |
| `oacp.conversation_id` | `conversation.conversationId` | string |
| `oacp.profile` | `governance.profile` | string |
| `oacp.profile_version` | `governance.profileVersion` | string |
| `oacp.binding_id` | `governance.bindingId` | string |
| `oacp.idempotency_key` | `idempotencyKey` | string |
| `oacp.from.participant_id` | `from.participantId` | string |
| `oacp.from.participant_type` | `from.participantType` | string |
| `oacp.from.assurance` | `from.identity.assurance` | string |
| `oacp.payload_type` | `payloadType` | string |

Attribute names use the `oacp.` prefix per OpenTelemetry semantic conventions guidance for non-standard domains.

## Span events

Each gate failure or significant lifecycle transition becomes a span **event** (point-in-time markers within the span):

| Event name | When emitted |
|---|---|
| `oacp.gate.identity.failed` | Gate step 1 rejected |
| `oacp.gate.tenant_scope.failed` | Gate step 1.5 rejected |
| `oacp.gate.authority.failed` | Gate step 2 rejected |
| `oacp.gate.binding.failed` | Gate step 3 rejected |
| `oacp.gate.authorization.failed` | Gate step 4 rejected |
| `oacp.gate.idempotency.conflict` | Gate step 5 conflict |
| `oacp.gate.broker.state_invalid` | Gate step 6 violation |
| `oacp.gate.causation.cycle` | Gate step 8 detected cycle |
| `oacp.gate.scope.violation` | Gate step 10 violation |
| `oacp.gate.profile.drift` | Gate step 11 drift |
| `oacp.gate.proposal.invalid` | Gate step 12 invalid |
| `oacp.gate.integrity.failed` | Gate step 13 broken hash chain |
| `oacp.gate.payload.invalid` | Gate step 14 schema fail |
| `oacp.proposal.lifecycle_transition` | Proposal entered new state |
| `oacp.broker.state_transition` | Broker entered new state |
| `oacp.review.state_transition` | Review entered new state |
| `oacp.binding.state_transition` | Binding entered new state |
| `oacp.replay.detected` | Message detected as replay |

Each event carries event attributes including the relevant OACP error code where applicable:

```
event: oacp.gate.tenant_scope.failed
attributes:
  oacp.error_code: "MESSAGE_TENANT_VIOLATION"
  oacp.violating_field: "scope.tenantId"
  oacp.expected_tenants: "[\"tenant_001\"]"
  oacp.actual_tenant: "tenant_002"
```

## Span status

OpenTelemetry span status: `Unset`, `Ok`, `Error`. OACP mapping:

- **`Ok`:** message processed successfully through all gates and dispatched.
- **`Error`:** any gate rejected the message; an error code is recorded as an event.
- **`Unset`:** in-progress spans (run not terminal).

The `status_message` SHOULD be the OACP error code (e.g., `MESSAGE_TENANT_VIOLATION`).

## Metrics

Recommended OpenTelemetry metrics for OACP runtimes:

| Metric | Type | Attributes |
|---|---|---|
| `oacp.messages_processed` | counter | `tenant_id`, `message_kind`, `result` (success/fail) |
| `oacp.gate_failures` | counter | `tenant_id`, `gate_step`, `error_code` |
| `oacp.proposal_lifecycle_transitions` | counter | `tenant_id`, `from_state`, `to_state` |
| `oacp.broker_decisions` | counter | `tenant_id`, `decision`, `decision_state` |
| `oacp.review_outcomes` | counter | `tenant_id`, `review_outcome` |
| `oacp.auto_approval_rate` | gauge | `tenant_id`, `agent_id` |
| `oacp.auto_approval_precondition_failures` | counter | `tenant_id`, `precondition` |
| `oacp.run_duration` | histogram | `tenant_id` |
| `oacp.message_age` | histogram | `tenant_id`, `profile` |
| `oacp.signature_failures` | counter | `tenant_id`, `failure_type` |

These metric names are recommendations; deployments are free to extend or rename.

## Logs

OACP runtimes typically emit log records correlated with spans (via OpenTelemetry's log API or trace context propagation). Log records correspond to:

- Audit-chain entries (one log per gate transition, one per state change).
- Authorization decisions.
- Tenant violations (security-relevant; SHOULD be elevated severity).
- Replay events.
- Hash-chain validation results.

Log records carry the same `oacp.*` attributes as spans for cross-correlation.

**Important:** OpenTelemetry logs are **not** the audit chain. The audit chain is OACP's authoritative governance record (run-scoped, tenant-isolated, hash-chained, ideally immutable per profile). Logs are operational. A deployment producing only OpenTelemetry logs and not maintaining a separate audit chain is non-conformant at OACP-Governed+.

## Trace propagation

OACP `correlationId` aligns with W3C Trace Context. Runtimes SHOULD:

1. On run start: generate a new OpenTelemetry trace ID and use it as `correlationId`.
2. On message emission: set the OpenTelemetry trace context to the run's trace ID; child span IDs propagate via `traceparent` header (in HTTP/CloudEvents bindings).
3. On message receive: extract trace context from transport; correlate with envelope `correlationId`.

## Correlation across runs

A single conversation may span multiple runs. Each run is a separate trace (separate trace ID). OpenTelemetry does not natively model "linked traces"; the cross-run correlation lives in `conversation.conversationId` and in span attributes (`oacp.conversation_id`).

Tools building dashboards over multi-run conversations query by `oacp.conversation_id`, not by trace ID.

## Tenant scope in observability

Tenant isolation extends to observability infrastructure. Recommendations:

- Tenant-scoped trace exporters (separate exporter pipelines per tenant).
- Tenant-scoped metric labels (so cross-tenant aggregation is opt-in).
- Tenant-scoped retention policies (some tenants may require longer audit retention).

OACP profile rules don't dictate observability infrastructure, but `OACP-Regulated` deployments typically require tenant-scoped trace storage as part of the larger isolation posture.

## Profile compatibility

| Profile | OpenTelemetry expectations |
|---|---|
| `OACP-Core` | Optional. Runs may be observed via runtime-specific tooling. |
| `OACP-Operational` | Recommended. Trace IDs aligned with `correlationId`. |
| `OACP-Governed` | Required. Span attributes for all reserved fields; gate failures as span events; OACP audit chain MUST exist independently. |
| `OACP-Regulated` | Required. Tenant-scoped exporter; signed log records; OACP audit chain in immutable storage. |

## OpenInference compatibility

[OpenInference](https://github.com/Arize-ai/openinference) is an observability spec for LLM/agent systems built on OpenTelemetry. OACP's span attributes are intentionally compatible with OpenInference conventions where they overlap (model identity, span hierarchy, trace propagation).

For OACP-Regulated deployments using LLM-based agents, runtimes can emit OpenInference-conformant spans alongside OACP-specific attributes — the two coexist without conflict.

## Open items

- **Distributed audit chain export:** how OACP audit chains export to immutable storage in a way that's queryable from OpenTelemetry-compatible tooling. Likely a future binding (Tier 4).
- **Cardinality control:** `oacp.tenant_id` and `oacp.run_id` are high-cardinality. Deployments must apply standard OpenTelemetry cardinality controls.
- **Standardization:** OACP's `oacp.*` attribute names are conventions, not formally registered semantic conventions. Promotion to a formal CNCF / OpenTelemetry semantic convention is plausible for v1.0.
