# Example 02 — Tool call with broker sanitization

## Scenario

An agent (`agent.notifier`) needs to send a notification email. Per policy, outbound emails go through the Tool Broker, which evaluates the policy, finds that the email body contains potentially sensitive content (a customer ID), and **sanitizes** the payload before dispatch. The broker emits a sanitized payload as an `ArtifactRef` and dispatches the sanitized version. The original payload is preserved in the audit chain; only the sanitized version is sent to the email service.

This example illustrates:

- The Tool Broker as an OACP participant.
- The 15-state broker state machine.
- The `decision` vs `decisionState` distinction.
- Sanitization with derived idempotency key.
- The persist-before-emit rule.

**Profile:** `OACP-Governed`
**Sections illustrated:** §3a, §3b, §10

---

## Pre-conditions

- Agent `agent.notifier` is bound to capability `cap.send_email` via binding `bind_email_001`.
- The Tool Broker `broker.tool` is registered and authorized for tool calls in `tenant_001`.
- The active policy snapshot `policy-snapshot-001` requires sanitization of payloads containing customer-ID patterns before outbound email dispatch.

---

## Message 1 — TOOL_CALL_REQUESTED

The agent emits a tool call to send an email. The payload contains a customer ID.

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_tc_request_001",
  "messageKind": "TOOL_CALL",
  "messageType": "TOOL_CALL_REQUESTED",
  "createdAt": "2026-05-05T17:20:00Z",
  "scope": {
    "scopeType": "TENANT",
    "tenantId": "tenant_001",
    "environment": "production"
  },
  "from": {
    "identity": {
      "assurance": "AUTHENTICATED",
      "subject": "principal.agent.notifier",
      "credentialRef": "cred_notifier",
      "credentialKind": "RUNTIME_TOKEN",
      "issuedAt": "2026-05-05T16:00:00Z",
      "expiresAt": "2026-05-05T22:00:00Z",
      "issuedBy": "runtime-adapter-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "runtime-adapter-001",
      "verifiedAt": "2026-05-05T17:20:00Z",
      "verificationMethod": "RUNTIME_INTERNAL",
      "tenantScope": {
        "authorizedTenants": ["tenant_001"],
        "source": "credential-claims"
      }
    },
    "participantId": "agent.notifier",
    "participantType": "AGENT"
  },
  "governance": {
    "profile": "OACP-Governed",
    "profileVersion": "0.1.0",
    "bindingId": "bind_email_001",
    "policySnapshotId": "policy-snapshot-001"
  },
  "run": {"runId": "run_002", "correlationId": "corr_002"},
  "conversation": {"conversationId": "conv_002"},
  "causation": {
    "causationType": "LINEAR",
    "causationId": "msg_run2_init",
    "causationIds": ["msg_run2_init"]
  },
  "idempotencyKey": "send_email_customer_C12345_v1",
  "idempotencyScope": "RUN",
  "authorizationDecision": {
    "decisionRef": {
      "decisionId": "authz_email_001",
      "decisionType": "AUTHORIZATION",
      "decidedBy": "policy-engine-001",
      "decidedAt": "2026-05-05T17:20:00Z",
      "tenantId": "tenant_001"
    },
    "decidedBy": "policy-engine-001",
    "decidedAt": "2026-05-05T17:20:00Z",
    "scope": {
      "messageType": "TOOL_CALL_REQUESTED",
      "bindingId": "bind_email_001",
      "runId": "run_002",
      "tenantId": "tenant_001"
    },
    "result": "ALLOW",
    "policySnapshotId": "policy-snapshot-001"
  },
  "payloadType": "tool-call-request-v1",
  "payload": {
    "toolCallId": "tc_001",
    "toolId": "send_email",
    "inputRef": {
      "artifactId": "tc_input_001",
      "artifactType": "TOOL_INPUT",
      "uri": "ojas://tenant_001/tool-inputs/tc_input_001",
      "mimeType": "application/json",
      "hash": "sha256:original_input_hash...",
      "sizeBytes": 512,
      "createdAt": "2026-05-05T17:20:00Z",
      "expiresAt": "2026-05-05T18:20:00Z",
      "tenantId": "tenant_001"
    },
    "bindingId": "bind_email_001",
    "requestedBy": {
      "participantId": "agent.notifier",
      "participantType": "AGENT"
    }
  }
}
```

**Resolved tool input artifact `tc_input_001`:**

```json
{
  "to": "support@example.com",
  "subject": "Customer issue update",
  "body": "Customer C12345 reported an issue with their order on 2026-05-04. Internal account reference: ACC-9876543210."
}
```

The payload contains a customer ID (`C12345`) and an internal account reference (`ACC-9876543210`). The active policy mandates sanitization.

---

## Broker state machine: REQUESTED → RECEIVED → VALIDATING → POLICY_EVALUATING

The broker (`broker.tool`) processes the tool call:

1. **REQUESTED:** receives `TOOL_CALL_REQUESTED`.
2. **RECEIVED:** acknowledges receipt; emits `TOOL_CALL_RECEIVED` (per persist-before-emit, the broker persists the state transition before emitting).
3. **VALIDATING:** validates the request schema and binding scope.
4. **POLICY_EVALUATING:** consults the policy engine.

Policy engine returns `decision: SANITIZE` because the body matches the customer-ID pattern.

---

## Message 2 — TOOL_CALL_SANITIZED (broker decision)

The broker emits its decision artifact with sanitized payload reference:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_tc_sanitized",
  "messageKind": "TOOL_CALL",
  "messageType": "TOOL_CALL_SANITIZED",
  "createdAt": "2026-05-05T17:20:05Z",
  "scope": {
    "scopeType": "TENANT",
    "tenantId": "tenant_001",
    "environment": "production"
  },
  "from": {
    "identity": {
      "assurance": "AUTHENTICATED",
      "subject": "principal.broker.tool",
      "credentialRef": "cred_broker",
      "credentialKind": "RUNTIME_KEY",
      "issuedAt": "2026-05-05T16:00:00Z",
      "expiresAt": "2026-05-05T22:00:00Z",
      "issuedBy": "runtime-adapter-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "runtime-adapter-001",
      "verifiedAt": "2026-05-05T17:20:05Z",
      "verificationMethod": "RUNTIME_INTERNAL",
      "tenantScope": {
        "authorizedTenants": ["tenant_001"],
        "source": "credential-claims"
      }
    },
    "participantId": "broker.tool",
    "participantType": "TOOL_BROKER"
  },
  "governance": {
    "profile": "OACP-Governed",
    "profileVersion": "0.1.0",
    "policySnapshotId": "policy-snapshot-001"
  },
  "run": {"runId": "run_002", "correlationId": "corr_002"},
  "conversation": {"conversationId": "conv_002"},
  "causation": {
    "causationType": "LINEAR",
    "causationId": "msg_tc_request_001",
    "causationIds": ["msg_tc_request_001"]
  },
  "responseTo": {
    "requestMessageId": "msg_tc_request_001",
    "requestMessageType": "TOOL_CALL_REQUESTED",
    "responseStatus": "PARTIAL"
  },
  "payloadType": "broker-decision-v1",
  "payload": {
    "decision": {
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
      "decidedByType": "TOOL_BROKER",
      "decidedAt": "2026-05-05T17:20:05Z",
      "policySnapshotId": "policy-snapshot-001",
      "tenantId": "tenant_001"
    },
    "sanitizedPayloadRef": {
      "artifactId": "san_001",
      "artifactType": "SANITIZED_PAYLOAD",
      "uri": "ojas://tenant_001/sanitized/san_001",
      "mimeType": "application/json",
      "hash": "sha256:sanitized_payload_hash...",
      "sizeBytes": 480,
      "createdAt": "2026-05-05T17:20:05Z",
      "expiresAt": "2026-05-05T18:20:05Z",
      "tenantId": "tenant_001"
    },
    "sanitizedIdempotencyKey": "san:send_email_customer_C12345_v1:OACP-Governed:0.1.0:policy-snapshot-001"
  }
}
```

Note the **`decision` vs `decisionState` distinction**:

- `decision: SANITIZE` — the broker's policy outcome.
- `decisionState: SANITIZED` — the state the broker entered as a result.

These are different fields. Per §10.3, the protocol does not collapse them.

**Resolved sanitized payload artifact `san_001`:**

```json
{
  "to": "support@example.com",
  "subject": "Customer issue update",
  "body": "Customer [REDACTED-CUSTOMER-ID] reported an issue with their order on 2026-05-04. Internal account reference: [REDACTED-ACCOUNT-REF]."
}
```

The customer ID and account reference have been replaced with placeholders.

The **sanitized idempotency key** is derived per §10.7:

```
sanitizedIdempotencyKey = "san:" + originalIdempotencyKey + ":" + profileId + ":" + profileVersion + ":" + policySnapshotId
                        = "san:send_email_customer_C12345_v1:OACP-Governed:0.1.0:policy-snapshot-001"
```

This ensures sanitized retries are idempotent within their sanitization context but distinguishable from the original.

---

## Broker state: SANITIZED → DISPATCHED → COMPLETED

The broker:

1. **DISPATCHED:** sends the sanitized payload to the email service. Emits `TOOL_CALL_DISPATCHED` referencing `san_001` (not the original).
2. The email service responds with success.
3. **COMPLETED:** broker emits `TOOL_RESULT` with `responseStatus: SUCCESS`.

---

## Message 3 — TOOL_RESULT

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_tool_result_001",
  "messageKind": "TOOL_RESULT",
  "messageType": "TOOL_RESULT_EMITTED",
  "createdAt": "2026-05-05T17:20:08Z",
  "scope": {
    "scopeType": "TENANT",
    "tenantId": "tenant_001",
    "environment": "production"
  },
  "from": {
    "identity": {
      "assurance": "AUTHENTICATED",
      "subject": "principal.broker.tool",
      "credentialRef": "cred_broker",
      "credentialKind": "RUNTIME_KEY",
      "issuedAt": "2026-05-05T16:00:00Z",
      "expiresAt": "2026-05-05T22:00:00Z",
      "issuedBy": "runtime-adapter-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "runtime-adapter-001",
      "verifiedAt": "2026-05-05T17:20:08Z",
      "verificationMethod": "RUNTIME_INTERNAL",
      "tenantScope": {
        "authorizedTenants": ["tenant_001"],
        "source": "credential-claims"
      }
    },
    "participantId": "broker.tool",
    "participantType": "TOOL_BROKER"
  },
  "governance": {
    "profile": "OACP-Governed",
    "profileVersion": "0.1.0",
    "policySnapshotId": "policy-snapshot-001"
  },
  "run": {"runId": "run_002", "correlationId": "corr_002"},
  "conversation": {"conversationId": "conv_002"},
  "causation": {
    "causationType": "LINEAR",
    "causationId": "msg_tc_request_001",
    "causationIds": ["msg_tc_request_001"]
  },
  "responseTo": {
    "requestMessageId": "msg_tc_request_001",
    "requestMessageType": "TOOL_CALL_REQUESTED",
    "responseStatus": "SUCCESS"
  },
  "payloadType": "tool-result-v1",
  "payload": {
    "toolCallId": "tc_001",
    "brokerDecisionId": "broker_dec_001",
    "resultRef": {
      "artifactId": "tc_result_001",
      "artifactType": "TOOL_RESULT",
      "uri": "ojas://tenant_001/tool-results/tc_result_001",
      "mimeType": "application/json",
      "hash": "sha256:result_hash...",
      "sizeBytes": 256,
      "createdAt": "2026-05-05T17:20:08Z",
      "expiresAt": "2026-05-05T18:20:08Z",
      "tenantId": "tenant_001"
    }
  }
}
```

The `responseTo.requestMessageId` correlates to the **original** `TOOL_CALL_REQUESTED`, not to any of the broker's intermediate lifecycle messages (per §10.11).

---

## What this demonstrates

| Invariant | How |
|---|---|
| Tool Broker is an OACP participant | `from.participantType: TOOL_BROKER` with its own identity and authority. |
| `decision` and `decisionState` are different concepts | Broker's policy outcome (`SANITIZE`) is distinct from the state the broker entered (`SANITIZED`). |
| Sanitization preserves the original in audit | Original `tc_input_001` and sanitized `san_001` are both retained; only the sanitized version is dispatched. |
| Sanitized payloads have derived idempotency keys | `san:` prefix plus original key plus profile and policy snapshot context. |
| Tool result correlates to the original request | `responseTo.requestMessageId` points to `TOOL_CALL_REQUESTED`, not intermediate broker messages. |
| Broker authority is enforced | Only `TOOL_BROKER` participantType may emit broker lifecycle messages and decisions; an AGENT cannot. |
| Persist-before-emit | Broker persists each state transition before emitting; recovery from crash does not produce duplicate transitions silently. |

A second example pattern (not illustrated here): when policy returns `decision: APPROVAL_NEEDED`, the broker enters `AWAITING_APPROVAL`, the policy snapshot is frozen at decision time, and an external decider's authority is validated at the time of decision emission. See §10.8.
