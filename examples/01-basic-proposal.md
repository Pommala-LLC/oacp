# Example 01 — Basic proposal with runtime auto-approval

## Scenario

An agent (`agent.classifier`) classifies an incoming PDF document and emits a `PROPOSAL_EMITTED` message recommending a classification. The runtime evaluates the proposal against the ten auto-approval preconditions (§13.5), finds all of them satisfied, and auto-approves via a `RuntimeDecisionRef` of type `RUNTIME_AUTO_APPROVAL`. The proposal moves to `AUTO_APPROVED`.

This example illustrates:

- The proposal/decision boundary (the agent proposes; the runtime decides).
- The ten auto-approval preconditions in action.
- The decision lifecycle (`ISSUED` → `ACCEPTED`).
- The self-decision rule's runtime-auto-approval exception.

**Profile:** `OACP-Governed`
**Sections illustrated:** §3a, §3b, §3d, §13.2–§13.6

---

## Pre-conditions

- The agent `agent.classifier` is bound to capability `cap.classify_pdf` via binding `bind_001`.
- The active policy snapshot is `policy-snapshot-001`.
- The runtime is configured to permit auto-approval for `LOW_RISK` proposals from this binding.

---

## Message 1 — RUN_STARTED (INITIATOR)

The runtime emits an `INITIATOR` message starting the run.

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_001",
  "messageKind": "EVENT",
  "messageType": "RUN_STARTED",
  "createdAt": "2026-05-05T17:00:00Z",
  "scope": {
    "scopeType": "TENANT",
    "tenantId": "tenant_001",
    "environment": "production"
  },
  "from": {
    "identity": {
      "assurance": "AUTHENTICATED",
      "subject": "principal.runtime.ojas",
      "credentialRef": "cred_runtime",
      "credentialKind": "RUNTIME_TOKEN",
      "issuedAt": "2026-05-05T16:00:00Z",
      "expiresAt": "2026-05-05T22:00:00Z",
      "issuedBy": "runtime-adapter-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "runtime-adapter-001",
      "verifiedAt": "2026-05-05T17:00:00Z",
      "verificationMethod": "RUNTIME_INTERNAL",
      "tenantScope": {
        "authorizedTenants": ["tenant_001"],
        "source": "credential-claims"
      }
    },
    "participantId": "runtime.ojas",
    "participantType": "RUNTIME"
  },
  "governance": {
    "profile": "OACP-Governed",
    "profileVersion": "0.1.0",
    "policySnapshotId": "policy-snapshot-001"
  },
  "run": {"runId": "run_001", "correlationId": "corr_001"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {"causationType": "INITIATOR"},
  "payloadType": "run-started-v1",
  "payload": {"runId": "run_001", "startedAt": "2026-05-05T17:00:00Z"}
}
```

**Gate steps exercised:** 1 (identity), 1.5 (tenant scope), 2 (authority — RUNTIME may emit RUN_STARTED), 11 (profile).

---

## Message 2 — PROPOSAL_EMITTED

The agent emits a proposal recommending `INVOICE` classification with confidence 0.96.

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_002",
  "messageKind": "PROPOSAL",
  "messageType": "PROPOSAL_EMITTED",
  "createdAt": "2026-05-05T17:10:00Z",
  "scope": {
    "scopeType": "TENANT",
    "tenantId": "tenant_001",
    "environment": "production"
  },
  "from": {
    "identity": {
      "assurance": "AUTHENTICATED",
      "subject": "principal.agent.classifier",
      "credentialRef": "cred_agent_classifier",
      "credentialKind": "RUNTIME_TOKEN",
      "issuedAt": "2026-05-05T16:00:00Z",
      "expiresAt": "2026-05-05T22:00:00Z",
      "issuedBy": "runtime-adapter-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "runtime-adapter-001",
      "verifiedAt": "2026-05-05T17:10:00Z",
      "verificationMethod": "RUNTIME_INTERNAL",
      "tenantScope": {
        "authorizedTenants": ["tenant_001"],
        "source": "credential-claims"
      }
    },
    "participantId": "agent.classifier",
    "participantType": "AGENT"
  },
  "governance": {
    "profile": "OACP-Governed",
    "profileVersion": "0.1.0",
    "bindingId": "bind_001",
    "policySnapshotId": "policy-snapshot-001"
  },
  "run": {"runId": "run_001", "correlationId": "corr_001"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {
    "causationType": "LINEAR",
    "causationId": "msg_001",
    "causationIds": ["msg_001"]
  },
  "authorizationDecision": {
    "decisionRef": {
      "decisionId": "authz_001",
      "decisionType": "AUTHORIZATION",
      "decidedBy": "policy-engine-001",
      "decidedAt": "2026-05-05T17:10:00Z",
      "tenantId": "tenant_001"
    },
    "decidedBy": "policy-engine-001",
    "decidedAt": "2026-05-05T17:10:00Z",
    "scope": {
      "messageType": "PROPOSAL_EMITTED",
      "bindingId": "bind_001",
      "runId": "run_001",
      "tenantId": "tenant_001"
    },
    "result": "ALLOW",
    "policySnapshotId": "policy-snapshot-001"
  },
  "payloadType": "proposal-emitted-v1",
  "payload": {
    "proposalId": "prop_001",
    "proposalRef": {
      "artifactId": "prop_001",
      "artifactType": "PROPOSAL",
      "uri": "ojas://tenant_001/proposals/prop_001",
      "mimeType": "application/json",
      "hash": "sha256:b1ce3f...",
      "sizeBytes": 4096,
      "createdAt": "2026-05-05T17:10:00Z",
      "expiresAt": "2026-05-05T19:10:00Z",
      "tenantId": "tenant_001"
    }
  }
}
```

**Resolved proposal artifact (fetched from `proposalRef`):**

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
    "hash": "sha256:7a4c1e...",
    "tenantId": "tenant_001"
  },
  "evidenceReportRefs": [
    {
      "artifactId": "evidence_001",
      "artifactType": "EVIDENCE_REPORT",
      "uri": "ojas://tenant_001/evidence/evidence_001",
      "hash": "sha256:9c2f8a...",
      "tenantId": "tenant_001"
    }
  ],
  "validationResultRefs": [
    {
      "artifactId": "vr_001",
      "artifactType": "VALIDATION_RESULT",
      "uri": "ojas://tenant_001/validation/vr_001",
      "hash": "sha256:5e1d2b...",
      "tenantId": "tenant_001"
    }
  ],
  "riskAssessmentRef": {
    "decisionId": "risk_001",
    "decisionType": "RISK_ASSESSMENT",
    "decidedBy": "risk-engine-001",
    "decidedAt": "2026-05-05T17:09:00Z",
    "tenantId": "tenant_001"
  },
  "calibration": {
    "confidence": 0.96,
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

**Gate steps exercised:** 1, 1.5, 2 (AGENT may emit PROPOSAL), 3 (binding `bind_001` valid), 4 (authorizationDecision present and ALLOW), 8 (causation valid), 11 (profile drift check), 12 (proposal valid).

---

## Runtime evaluates ten auto-approval preconditions

The runtime checks each precondition (§13.5):

1. **Risk class is LOW.** `risk_001` returns `LOW_RISK`. ✅
2. **Calibration meets threshold.** `confidence: 0.96` ≥ deployment threshold (0.90). `calibrationMethod: TEMPERATURE_SCALED` is in the permitted set. ✅
3. **Validation gates passed.** `vr_001` resolves to `PASSED`. ✅
4. **Evidence quality threshold met.** `evidence_001` meets quality gates. ✅
5. **Authority bound and valid.** `bind_001` is `ACTIVE`. ✅
6. **Tenant scope verified.** `tenant_001` is in auto-approval policy's authorized scope. ✅
7. **Auto-approval policy-permitted.** Active policy permits auto-approval for `cap.classify_pdf` proposals. ✅
8. **No active review.** No `REVIEW_REQUESTED` is `PENDING` for `prop_001`. ✅
9. **No active escalation.** No `ESCALATION_REQUESTED` is in flight. ✅
10. **Preconditions auditable.** Runtime emits `AutoApprovalPreconditionsRecord` capturing all of the above. ✅

All ten preconditions hold. Runtime auto-approves.

---

## Message 3 — RUNTIME_AUTO_APPROVAL decision emitted

The runtime emits a `RuntimeDecisionRef`:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_003",
  "messageKind": "EVENT",
  "messageType": "RUNTIME_DECISION_EMITTED",
  "createdAt": "2026-05-05T17:11:00Z",
  "scope": {
    "scopeType": "TENANT",
    "tenantId": "tenant_001",
    "environment": "production"
  },
  "from": {
    "identity": {
      "assurance": "AUTHENTICATED",
      "subject": "principal.runtime.ojas",
      "credentialRef": "cred_runtime",
      "credentialKind": "RUNTIME_TOKEN",
      "issuedAt": "2026-05-05T16:00:00Z",
      "expiresAt": "2026-05-05T22:00:00Z",
      "issuedBy": "runtime-adapter-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "runtime-adapter-001",
      "verifiedAt": "2026-05-05T17:11:00Z",
      "verificationMethod": "RUNTIME_INTERNAL",
      "tenantScope": {
        "authorizedTenants": ["tenant_001"],
        "source": "credential-claims"
      }
    },
    "participantId": "runtime.ojas",
    "participantType": "RUNTIME"
  },
  "governance": {
    "profile": "OACP-Governed",
    "profileVersion": "0.1.0",
    "policySnapshotId": "policy-snapshot-001"
  },
  "run": {"runId": "run_001", "correlationId": "corr_001"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {
    "causationType": "LINEAR",
    "causationId": "msg_002",
    "causationIds": ["msg_002"]
  },
  "payloadType": "runtime-decision-emitted-v1",
  "payload": {
    "decision": {
      "decisionId": "rt_dec_001",
      "decisionType": "RUNTIME_AUTO_APPROVAL",
      "decidedBy": "runtime.ojas",
      "decidedByType": "RUNTIME",
      "decidedAt": "2026-05-05T17:11:00Z",
      "decisionStatus": "ISSUED",
      "appliesTo": {
        "objectType": "PROPOSAL",
        "proposalId": "prop_001"
      },
      "policySnapshotId": "policy-snapshot-001",
      "tenantId": "tenant_001"
    },
    "autoApprovalPreconditionsRef": {
      "recordId": "aap_record_001",
      "recordType": "AUTO_APPROVAL_PRECONDITIONS",
      "registry": "decision-registry",
      "scopeType": "TENANT",
      "tenantId": "tenant_001"
    }
  }
}
```

**Self-decision rule check:** the decision's `decidedBy` is `runtime.ojas`; the proposal's `proposedBy.participantId` is `agent.classifier`. They differ. The runtime is a separate participant from the agent. The self-decision rule is satisfied via the runtime-auto-approval exception (§13.6).

**Decision lifecycle:** the decision starts in `ISSUED`. The runtime then transitions it to `ACCEPTED` and updates the proposal's `proposalLifecycleState` to `AUTO_APPROVED`.

---

## What this demonstrates

| Invariant | How |
|---|---|
| Proposals are recommendations, never approvals | The agent's PROPOSAL_EMITTED does not finalize anything; the runtime's separate decision artifact does. |
| Self-decision is forbidden (with runtime exception) | Runtime auto-approves an agent's proposal; this is the explicit exception in §13.6. |
| Every proposal closure produces a decision record | `rt_dec_001` is the closure record for `prop_001`. |
| Auto-approval requires all ten preconditions | The runtime evaluates each one before emitting the decision. |
| Decision lifecycle ISSUED → ACCEPTED | The proposal lifecycle advances only when the decision is `ACCEPTED`, not on `ISSUED`. |
| Causation forms a DAG | `msg_001` (INITIATOR) → `msg_002` (proposal) → `msg_003` (decision). |
