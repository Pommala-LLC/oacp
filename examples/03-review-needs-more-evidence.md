# Example 03 — Review with NEEDS_MORE_EVIDENCE re-run

## Scenario

An agent emits a proposal to classify a document. The proposal carries `MEDIUM` risk, and policy requires external review. A reviewer responds with `NEEDS_MORE_EVIDENCE` — they need additional verification before deciding. The runtime evaluates this signal, decides to re-run with additional inputs, and starts a **new run in the same conversation**. The new run produces a fresh proposal that references the original via `RESPONDS_TO_REVIEW`.

This example illustrates:

- The REVIEW protocol's distinction between `reviewType` (request) and `reviewOutcome` (response).
- That review feedback never finalizes a decision (`REVIEW_FEEDBACK_NOT_DECISION`).
- The cross-run conversation continuity model.
- Hash chains as run-scoped (not crossing the run boundary even within the same conversation).
- Proposal-to-proposal `RESPONDS_TO_REVIEW` relationship.

**Profile:** `OACP-Governed`
**Sections illustrated:** §3d, §11a, §13.8

---

## Run R1 — Initial proposal and review

### Pre-conditions

- Agent `agent.classifier` is bound via `bind_001`.
- Conversation `conv_001` is initiated.

### Message 1 — RUN_STARTED (R1, INITIATOR)

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_r1_001",
  "messageKind": "EVENT",
  "messageType": "RUN_STARTED",
  "createdAt": "2026-05-05T17:00:00Z",
  "scope": {"scopeType": "TENANT", "tenantId": "tenant_001", "environment": "production"},
  "from": {
    "identity": {"assurance": "AUTHENTICATED", "subject": "principal.runtime.ojas", "credentialRef": "cred_runtime", "credentialKind": "RUNTIME_TOKEN", "issuedAt": "2026-05-05T16:00:00Z", "expiresAt": "2026-05-05T22:00:00Z", "issuedBy": "runtime-adapter-001"},
    "verification": {"verified": true, "verifiedBy": "runtime-adapter-001", "verifiedAt": "2026-05-05T17:00:00Z", "verificationMethod": "RUNTIME_INTERNAL", "tenantScope": {"authorizedTenants": ["tenant_001"], "source": "credential-claims"}},
    "participantId": "runtime.ojas",
    "participantType": "RUNTIME"
  },
  "governance": {"profile": "OACP-Governed", "profileVersion": "0.1.0", "policySnapshotId": "policy-snapshot-001"},
  "run": {"runId": "run_001", "correlationId": "corr_r1"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {"causationType": "INITIATOR"},
  "payloadType": "run-started-v1",
  "payload": {"runId": "run_001", "startedAt": "2026-05-05T17:00:00Z"}
}
```

### Message 2 — PROPOSAL_EMITTED (prop_001)

The agent emits `prop_001` with `MEDIUM` risk:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_r1_002",
  "messageKind": "PROPOSAL",
  "messageType": "PROPOSAL_EMITTED",
  "createdAt": "2026-05-05T17:10:00Z",
  "scope": {"scopeType": "TENANT", "tenantId": "tenant_001", "environment": "production"},
  "from": {
    "identity": {"assurance": "AUTHENTICATED", "subject": "principal.agent.classifier", "credentialRef": "cred_agent_classifier", "credentialKind": "RUNTIME_TOKEN", "issuedAt": "2026-05-05T16:00:00Z", "expiresAt": "2026-05-05T22:00:00Z", "issuedBy": "runtime-adapter-001"},
    "verification": {"verified": true, "verifiedBy": "runtime-adapter-001", "verifiedAt": "2026-05-05T17:10:00Z", "verificationMethod": "RUNTIME_INTERNAL", "tenantScope": {"authorizedTenants": ["tenant_001"], "source": "credential-claims"}},
    "participantId": "agent.classifier",
    "participantType": "AGENT"
  },
  "governance": {"profile": "OACP-Governed", "profileVersion": "0.1.0", "bindingId": "bind_001", "policySnapshotId": "policy-snapshot-001"},
  "run": {"runId": "run_001", "correlationId": "corr_r1"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {"causationType": "LINEAR", "causationId": "msg_r1_001", "causationIds": ["msg_r1_001"]},
  "authorizationDecision": {"decisionRef": {"decisionId": "authz_r1_002", "decisionType": "AUTHORIZATION", "decidedBy": "policy-engine-001", "decidedAt": "2026-05-05T17:10:00Z", "tenantId": "tenant_001"}, "decidedBy": "policy-engine-001", "decidedAt": "2026-05-05T17:10:00Z", "scope": {"messageType": "PROPOSAL_EMITTED", "bindingId": "bind_001", "runId": "run_001", "tenantId": "tenant_001"}, "result": "ALLOW", "policySnapshotId": "policy-snapshot-001"},
  "payloadType": "proposal-emitted-v1",
  "payload": {
    "proposalId": "prop_001",
    "proposalRef": {"artifactId": "prop_001", "artifactType": "PROPOSAL", "uri": "ojas://tenant_001/proposals/prop_001", "mimeType": "application/json", "hash": "sha256:prop_001_hash", "sizeBytes": 4096, "createdAt": "2026-05-05T17:10:00Z", "expiresAt": "2026-05-05T19:10:00Z", "tenantId": "tenant_001"}
  }
}
```

The proposal artifact carries `riskAssessmentRef` returning `MEDIUM_RISK`. Policy mandates external review for MEDIUM-risk classifications.

### Message 3 — REVIEW_REQUESTED

The runtime emits a review request:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_r1_003",
  "messageKind": "REVIEW",
  "messageType": "REVIEW_REQUESTED",
  "createdAt": "2026-05-05T17:11:00Z",
  "ttl": {"expiresAt": "2026-05-05T18:11:00Z"},
  "scope": {"scopeType": "TENANT", "tenantId": "tenant_001", "environment": "production"},
  "from": {
    "identity": {"assurance": "AUTHENTICATED", "subject": "principal.runtime.ojas", "credentialRef": "cred_runtime", "credentialKind": "RUNTIME_TOKEN", "issuedAt": "2026-05-05T16:00:00Z", "expiresAt": "2026-05-05T22:00:00Z", "issuedBy": "runtime-adapter-001"},
    "verification": {"verified": true, "verifiedBy": "runtime-adapter-001", "verifiedAt": "2026-05-05T17:11:00Z", "verificationMethod": "RUNTIME_INTERNAL", "tenantScope": {"authorizedTenants": ["tenant_001"], "source": "credential-claims"}},
    "participantId": "runtime.ojas",
    "participantType": "RUNTIME"
  },
  "governance": {"profile": "OACP-Governed", "profileVersion": "0.1.0", "policySnapshotId": "policy-snapshot-001"},
  "run": {"runId": "run_001", "correlationId": "corr_r1"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {"causationType": "LINEAR", "causationId": "msg_r1_002", "causationIds": ["msg_r1_002"]},
  "payloadType": "review-request-v1",
  "payload": {
    "reviewId": "review_001",
    "reviewType": "EVIDENCE_REVIEW",
    "appliesTo": {
      "objectType": "PROPOSAL",
      "proposalRef": {"artifactId": "prop_001", "artifactType": "PROPOSAL", "uri": "ojas://tenant_001/proposals/prop_001", "hash": "sha256:prop_001_hash", "tenantId": "tenant_001"}
    },
    "requestedInputs": [
      "verify_layout_signature_block_completeness",
      "rerun_visual_similarity_against_reference_corpus"
    ],
    "reviewSystemRef": "review-system-001",
    "businessDeadline": "2026-05-05T18:00:00Z"
  }
}
```

Note `reviewType: EVIDENCE_REVIEW` — the kind of review being requested. `ttl.expiresAt` is required at OACP-Governed (§11a.10).

### Message 4 — REVIEW_RESPONDED with NEEDS_MORE_EVIDENCE

The reviewer responds:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_r1_004",
  "messageKind": "REVIEW",
  "messageType": "REVIEW_RESPONDED",
  "createdAt": "2026-05-05T17:35:00Z",
  "scope": {"scopeType": "TENANT", "tenantId": "tenant_001", "environment": "production"},
  "from": {
    "identity": {"assurance": "AUTHENTICATED", "subject": "principal.reviewer.alice", "credentialRef": "cred_reviewer_alice", "credentialKind": "OAUTH_TOKEN", "issuedAt": "2026-05-05T17:00:00Z", "expiresAt": "2026-05-05T23:00:00Z", "issuedBy": "review-system-001"},
    "verification": {"verified": true, "verifiedBy": "review-system-001", "verifiedAt": "2026-05-05T17:35:00Z", "verificationMethod": "OAUTH_VERIFY", "tenantScope": {"authorizedTenants": ["tenant_001"], "source": "credential-claims"}},
    "participantId": "reviewer.alice",
    "participantType": "EXTERNAL_REVIEWER"
  },
  "governance": {"profile": "OACP-Governed", "profileVersion": "0.1.0", "policySnapshotId": "policy-snapshot-001"},
  "run": {"runId": "run_001", "correlationId": "corr_r1"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {"causationType": "LINEAR", "causationId": "msg_r1_003", "causationIds": ["msg_r1_003"]},
  "responseTo": {"requestMessageId": "msg_r1_003", "requestMessageType": "REVIEW_REQUESTED", "responseStatus": "SUCCESS"},
  "payloadType": "review-response-v1",
  "payload": {
    "reviewId": "review_001",
    "reviewOutcome": "NEEDS_MORE_EVIDENCE",
    "reviewFeedbackRef": {
      "recordId": "rf_001",
      "recordType": "REVIEW_FEEDBACK",
      "registry": "review-registry",
      "scopeType": "TENANT",
      "tenantId": "tenant_001"
    },
    "reviewerNotes": "Layout signature block appears truncated. Need re-extraction with adjusted bounding box and a similarity check against the reference INVOICE corpus."
  }
}
```

`reviewOutcome: NEEDS_MORE_EVIDENCE`. **This is NOT an approval or rejection.** It is a signal that the reviewer needs additional inputs before being able to decide.

Per §11a.7 (REVIEW_FEEDBACK_NOT_DECISION): the proposal `prop_001` does NOT transition based on this review response alone. The proposal is still in `PENDING`. A separate decision artifact would be required to finalize it.

### Run R1 terminates without decision

The runtime evaluates the review outcome. Per §11a.6:

> A `NEEDS_MORE_EVIDENCE` outcome does NOT itself start a new run. It signals that a re-run *may* be warranted. Only `RUNTIME` or authorized `SUPERVISOR` may start the new run, after evaluating against deployment policy.

The runtime decides to re-run with additional inputs. R1 terminates with `prop_001` still in `PENDING` and no decision artifact yet. (Eventually `prop_001` will move to `EXPIRED` when its `expiresAt` elapses, producing a `RuntimeDecisionRef` of type `EXPIRY` — that closure happens regardless of the new run.)

---

## Run R2 — Re-run with additional inputs

### Message 5 — RUN_STARTED (R2, INITIATOR with cross-run causation)

R2 begins. It has a fresh `runId` but the **same** `conversationId`. The causation references R1's terminal review response — this is permitted because both runs share `conversationId` (§3d.8 rule 4).

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_r2_005",
  "messageKind": "EVENT",
  "messageType": "RUN_STARTED",
  "createdAt": "2026-05-05T17:40:00Z",
  "scope": {"scopeType": "TENANT", "tenantId": "tenant_001", "environment": "production"},
  "from": {
    "identity": {"assurance": "AUTHENTICATED", "subject": "principal.runtime.ojas", "credentialRef": "cred_runtime", "credentialKind": "RUNTIME_TOKEN", "issuedAt": "2026-05-05T16:00:00Z", "expiresAt": "2026-05-05T22:00:00Z", "issuedBy": "runtime-adapter-001"},
    "verification": {"verified": true, "verifiedBy": "runtime-adapter-001", "verifiedAt": "2026-05-05T17:40:00Z", "verificationMethod": "RUNTIME_INTERNAL", "tenantScope": {"authorizedTenants": ["tenant_001"], "source": "credential-claims"}},
    "participantId": "runtime.ojas",
    "participantType": "RUNTIME"
  },
  "governance": {"profile": "OACP-Governed", "profileVersion": "0.1.0", "policySnapshotId": "policy-snapshot-001"},
  "run": {"runId": "run_002", "correlationId": "corr_r2"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {
    "causationType": "INITIATOR",
    "causationId": "msg_r1_004",
    "causationIds": ["msg_r1_004"]
  },
  "payloadType": "run-started-v1",
  "payload": {"runId": "run_002", "startedAt": "2026-05-05T17:40:00Z", "rerunOf": "run_001", "rerunReason": "NEEDS_MORE_EVIDENCE_RESPONSE"}
}
```

Notice: `causationType: INITIATOR` with `causationId: msg_r1_004`. This is permitted because:

1. R2's `runId` differs from R1's.
2. The conversation ID is the same.
3. The `causationId` references the terminal message of the prior run.
4. Per §3d.8 rule 4: cross-run causation is permitted within shared conversation.

**Hash chain:** R2 starts a new hash chain. R2's `previousMessageHash` is absent for this INITIATOR. R2's chain does NOT extend R1's chain — hash chains are run-scoped (§3e.7). Continuity comes from causation and conversation, not hash-chain extension.

### Message 6 — PROPOSAL_EMITTED (prop_002, RESPONDS_TO_REVIEW)

R2 produces a fresh proposal with the additional evidence:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_r2_006",
  "messageKind": "PROPOSAL",
  "messageType": "PROPOSAL_EMITTED",
  "createdAt": "2026-05-05T17:42:00Z",
  "scope": {"scopeType": "TENANT", "tenantId": "tenant_001", "environment": "production"},
  "from": {
    "identity": {"assurance": "AUTHENTICATED", "subject": "principal.agent.classifier", "credentialRef": "cred_agent_classifier", "credentialKind": "RUNTIME_TOKEN", "issuedAt": "2026-05-05T16:00:00Z", "expiresAt": "2026-05-05T22:00:00Z", "issuedBy": "runtime-adapter-001"},
    "verification": {"verified": true, "verifiedBy": "runtime-adapter-001", "verifiedAt": "2026-05-05T17:42:00Z", "verificationMethod": "RUNTIME_INTERNAL", "tenantScope": {"authorizedTenants": ["tenant_001"], "source": "credential-claims"}},
    "participantId": "agent.classifier",
    "participantType": "AGENT"
  },
  "governance": {"profile": "OACP-Governed", "profileVersion": "0.1.0", "bindingId": "bind_001", "policySnapshotId": "policy-snapshot-001"},
  "run": {"runId": "run_002", "correlationId": "corr_r2"},
  "conversation": {"conversationId": "conv_001"},
  "causation": {"causationType": "LINEAR", "causationId": "msg_r2_005", "causationIds": ["msg_r2_005"]},
  "authorizationDecision": {"decisionRef": {"decisionId": "authz_r2_006", "decisionType": "AUTHORIZATION", "decidedBy": "policy-engine-001", "decidedAt": "2026-05-05T17:42:00Z", "tenantId": "tenant_001"}, "decidedBy": "policy-engine-001", "decidedAt": "2026-05-05T17:42:00Z", "scope": {"messageType": "PROPOSAL_EMITTED", "bindingId": "bind_001", "runId": "run_002", "tenantId": "tenant_001"}, "result": "ALLOW", "policySnapshotId": "policy-snapshot-001"},
  "payloadType": "proposal-emitted-v1",
  "payload": {
    "proposalId": "prop_002",
    "proposalRef": {"artifactId": "prop_002", "artifactType": "PROPOSAL", "uri": "ojas://tenant_001/proposals/prop_002", "mimeType": "application/json", "hash": "sha256:prop_002_hash", "sizeBytes": 4096, "createdAt": "2026-05-05T17:42:00Z", "expiresAt": "2026-05-05T19:42:00Z", "tenantId": "tenant_001"}
  }
}
```

The resolved proposal artifact carries:

```json
{
  "proposalId": "prop_002",
  "proposalVersion": "1",
  "proposalLifecycleState": "PENDING",
  "proposalRefs": [
    {
      "proposalRef": {"artifactId": "prop_001", "artifactType": "PROPOSAL", "uri": "ojas://tenant_001/proposals/prop_001", "tenantId": "tenant_001"},
      "relationship": "RESPONDS_TO_REVIEW"
    }
  ],
  "auditRefs": [
    {"auditId": "audit_review_msg_r1_004", "auditChainRef": "chain_run_001", "tenantId": "tenant_001"}
  ],
  "tenantId": "tenant_001",
  "runId": "run_002",
  "conversationId": "conv_001"
}
```

The `proposalRefs[]` declares: this proposal `RESPONDS_TO_REVIEW` of `prop_001`. Per §13.8:

- `RESPONDS_TO_REVIEW` does NOT mutate prop_001's state. prop_001 remains in its terminal state.
- prop_002 exists alongside prop_001, not as a replacement.
- Authority is fine: agents may emit `RESPONDS_TO_REVIEW`-type relationships (it doesn't mutate the prior proposal).

The `auditRefs` references R1's review feedback so auditors can reconstruct the lineage.

### R2 continues with normal authorization flow

prop_002 enters the normal flow. Eventually an `ExternalDecisionRef` (or `RuntimeDecisionRef`) closes it. That part of the example is omitted for brevity.

---

## What this demonstrates

| Invariant | How |
|---|---|
| `reviewType` and `reviewOutcome` are different fields | Request carries `reviewType: EVIDENCE_REVIEW`; response carries `reviewOutcome: NEEDS_MORE_EVIDENCE`. |
| Review feedback never finalizes a decision | `NEEDS_MORE_EVIDENCE` does not move prop_001 to APPROVED or REJECTED. The proposal is still PENDING. |
| Runtime authority required for re-run | Only RUNTIME starts R2. An AGENT proposing a re-run would trigger `RERUN_AUTHORITY_INVALID`. |
| Conversations span runs; hash chains do not | Both runs share `conversationId: conv_001`. R1 and R2 have independent hash chains. |
| Cross-run causation is permitted within a conversation | R2's INITIATOR references R1's terminal message because they share conversation. |
| Proposals are immutable | prop_001 is never modified. prop_002 is a fresh proposal that references prop_001. |
| `RESPONDS_TO_REVIEW` does not mutate | prop_001 remains in its state; the relationship is informational lineage. |
| Agent indirect-review pattern preserved | Agent emits a proposal with MEDIUM risk; runtime decides to seek review. Agent does not directly emit REVIEW_REQUESTED. |
