# Example 05 — Replay under ORIGINAL_POLICY mode

## Scenario

A policy update has occurred between time T1 and time T2. A regulated investigation requires re-evaluating an old proposal that closed at time T1, to see whether — under the **original** policy in effect at T1 — the runtime made the correct decision. The runtime replays the original `PROPOSAL_EMITTED` message under `ORIGINAL_POLICY` mode, with explicit replay authorization, and observes that the policy evaluation produces the same decision.

This example illustrates:

- Replay metadata and the replay authority rule.
- The four replay modes (focus on `ORIGINAL_POLICY`).
- Replay age-bypass with required `replayAuthorizationRef`.
- The relationship between replay and idempotency.
- That signing keys are validated against their status *as of* the original `signedAt`, not current.

**Profile:** `OACP-Governed`
**Sections illustrated:** §3c.7-§3c.8, §3e.4, §3e.9

---

## Context

- **T1 (2026-04-01T10:00:00Z):** Original message `msg_original` emitted; proposal `prop_original` closed via `RuntimeDecisionRef` of type `RUNTIME_AUTO_APPROVAL`. Policy in effect: `policy-snapshot-prior`.
- **T1.5 (2026-04-15):** Policy updated to a stricter version. Active snapshot becomes `policy-snapshot-current`.
- **T2 (2026-05-05T19:00:00Z):** A compliance investigation needs to verify: under the original policy at T1, was the auto-approval correct?

The runtime replays `msg_original` to re-execute the decision under the original snapshot. This is a **diagnostic verification**, not a re-issue of the decision.

---

## Replay authorization (pre-condition)

Before emitting the replay, the runtime obtains a replay authorization decision:

```json
{
  "decisionId": "replay_auth_001",
  "decisionType": "REPLAY_AUTHORIZATION",
  "decidedBy": "supervisor.compliance-investigator",
  "decidedByType": "SUPERVISOR",
  "decidedAt": "2026-05-05T18:55:00Z",
  "decisionStatus": "ISSUED",
  "appliesTo": {
    "objectType": "MESSAGE",
    "messageId": "msg_original"
  },
  "replayMode": "ORIGINAL_POLICY",
  "replayPurpose": "COMPLIANCE_VERIFICATION_2026Q2",
  "policySnapshotId": "policy-snapshot-prior",
  "tenantId": "tenant_001"
}
```

A `SUPERVISOR` (compliance investigator role) authorizes the replay. The decision references the replay mode (`ORIGINAL_POLICY`) and the original policy snapshot. Per §3c.7, only `RUNTIME` or authorized `SUPERVISOR` may emit replay metadata. An AGENT cannot author replay.

---

## Message — Replay envelope

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_replay_001",
  "messageKind": "EVENT",
  "messageType": "PROPOSAL_EMITTED",
  "createdAt": "2026-05-05T19:00:00Z",
  "scope": {
    "scopeType": "TENANT",
    "tenantId": "tenant_001",
    "environment": "production"
  },
  "from": {
    "identity": {
      "assurance": "AUTHENTICATED",
      "subject": "principal.runtime.ojas",
      "credentialRef": "cred_runtime_current",
      "credentialKind": "RUNTIME_TOKEN",
      "issuedAt": "2026-05-05T18:00:00Z",
      "expiresAt": "2026-05-06T00:00:00Z",
      "issuedBy": "runtime-adapter-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "runtime-adapter-001",
      "verifiedAt": "2026-05-05T19:00:00Z",
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
    "policySnapshotId": "policy-snapshot-prior"
  },
  "run": {"runId": "run_replay_001", "correlationId": "corr_replay_001"},
  "causation": {"causationType": "INITIATOR"},
  "idempotencyKey": "auto_classify_doc_xyz_v1",
  "idempotencyScope": "TENANT",
  "replay": {
    "replayId": "replay_001",
    "originalMessageId": "msg_original",
    "originalRunId": "run_original",
    "replayMode": "ORIGINAL_POLICY",
    "reason": "COMPLIANCE_VERIFICATION_2026Q2",
    "replayedBy": "runtime.ojas",
    "replayedAt": "2026-05-05T19:00:00Z",
    "replayAuthorizationRef": {
      "decisionId": "replay_auth_001",
      "decisionType": "REPLAY_AUTHORIZATION",
      "decidedBy": "supervisor.compliance-investigator",
      "decidedAt": "2026-05-05T18:55:00Z",
      "tenantId": "tenant_001"
    }
  },
  "replayable": false,
  "payloadType": "proposal-emitted-v1",
  "payload": {
    "proposalId": "prop_original_replay",
    "proposalRef": {
      "artifactId": "prop_original",
      "artifactType": "PROPOSAL",
      "uri": "ojas://tenant_001/proposals/prop_original",
      "mimeType": "application/json",
      "hash": "sha256:original_proposal_hash",
      "sizeBytes": 4096,
      "createdAt": "2026-04-01T10:00:00Z",
      "expiresAt": "2026-04-01T12:00:00Z",
      "tenantId": "tenant_001"
    }
  }
}
```

Key features:

1. **`messageId: msg_replay_001`** — a fresh message ID. The replay is a new message with its own ID; the original `msg_original` is not modified.

2. **`createdAt: 2026-05-05T19:00:00Z`** — the replay's emission time, NOT the original. Despite this being a recent timestamp, the message effectively re-evaluates state from 2026-04-01.

3. **`governance.policySnapshotId: policy-snapshot-prior`** — explicitly references the original snapshot, not `policy-snapshot-current`. Per §3c.7, `ORIGINAL_POLICY` mode evaluates against the original snapshot.

4. **`replay` block populated** — required for replay messages.

   - `replayMode: ORIGINAL_POLICY` — evaluates under original snapshot.
   - `originalMessageId: msg_original` — the original being replayed.
   - `originalRunId: run_original` — the original run.
   - `replayAuthorizationRef` — required at OACP-Governed+ for age-bypass replay modes (§3e.4). `ORIGINAL_POLICY` is age-bypass because the original message would otherwise fail the `MESSAGE_AGE_EXCEEDED` check.

5. **`replayable: false`** — the replay itself is not eligible to be replayed. Replays of replays are not supported in v0.1.

6. **`run.runId: run_replay_001`** — fresh runId. The replay creates a new run for its own execution and audit.

7. **`idempotencyKey: auto_classify_doc_xyz_v1`** — the same key as the original. Per §3c.8 for `ORIGINAL_POLICY` mode, deduplication may apply if within `dedupeWindowSeconds`. In this case, the original was emitted >30 days ago — well outside any reasonable dedupe window — so the replay is processed.

---

## Receiver-side handling

The receiver processes the replay envelope:

### Gate 1.5 — Tenant scope

`tenant_001` is in receiver's authorized scope. ✅

### Gate 5 — Idempotency / replay handling

The receiver detects `replay` block. It verifies:

- `replayedBy` is a permitted authority (RUNTIME). ✅
- `replayMode` is in the registry. ✅
- `replayAuthorizationRef` is present (required for `ORIGINAL_POLICY` at OACP-Governed). ✅
- `replayAuthorizationRef.replayMode` matches the envelope's `replayMode`. ✅
- The replay authorization is `ISSUED` and not revoked. ✅

### Gate 11 — Profile

The replay envelope claims `OACP-Governed`. The original was emitted under `OACP-Governed`. Profile claim consistency holds.

### Gate 12 — Proposal validity under original policy

Per `ORIGINAL_POLICY` mode, the receiver evaluates proposal validity against `policy-snapshot-prior`, not the current snapshot. This may produce different gate-pass results than evaluating under current policy:

- Auto-approval preconditions are checked against `policy-snapshot-prior`'s thresholds.
- Calibration thresholds, risk class evaluation, side-effect classifications all use the original snapshot's rules.

### Gate 13 — Message age (bypassed)

Normally the message would fail `MESSAGE_AGE_EXCEEDED` because its content references events from 2026-04-01. **Age check is bypassed because `replayMode: ORIGINAL_POLICY`** is in the bypass set (§3e.4) **and** `replayAuthorizationRef` is present.

---

## Hash chain

The replay starts a new run (`run_replay_001`) with its own hash chain. The replay's chain does NOT extend the original's chain — chains are run-scoped (§3e.7).

Chain continuity for the *replay run* would resemble:

- `msg_replay_001` (INITIATOR for `run_replay_001`) — no `previousMessageHash`.
- Subsequent messages in `run_replay_001` chain to each other, not to `run_original`'s chain.

For audit purposes, the replay's audit chain references the original's audit via `replay.originalMessageId` and `replay.originalRunId`, but the cryptographic hash chains are independent.

---

## Signing key validation

The original message's `signatureRef` (if present at the Regulated profile, or computed for diagnostic verification) was signed by a key valid at 2026-04-01.

Per §3e.9, signing keys are validated against their status **as of `signedAt`**, not current:

| Key status at signedAt (2026-04-01) | Replay verification |
|---|---|
| ACTIVE | Signature verifies (as expected for legitimate original) |
| REVOKED forward (revoked after 2026-04-01) | Still verifies — key was valid when used |
| COMPROMISED retroactively (revocation backdated to 2026-03-01) | Fails with `SIGNATURE_KEY_REVOKED` — key was retroactively invalid |

This rule allows forward key rotation without invalidating prior signatures, while retaining the ability to retroactively invalidate compromised keys.

---

## Outcome

The replay completes. The runtime evaluates the proposal under `policy-snapshot-prior` and emits an internal compliance-verification record showing the evaluation result.

The replay is observational. It does NOT:

- Re-issue any decision against `prop_original`.
- Modify `prop_original`'s state.
- Cause any business effect.

The original `prop_original` remains in its terminal `AUTO_APPROVED` state. The replay just verifies, for audit purposes, that the original auto-approval was consistent with the original policy.

---

## What this demonstrates

| Invariant | How |
|---|---|
| Replay authority is restricted | Only RUNTIME or authorized SUPERVISOR may author replay metadata. |
| Replay does not mutate the original | `prop_original` is unchanged. The replay is a diagnostic re-execution, not a re-issue. |
| `ORIGINAL_POLICY` mode evaluates under the original snapshot | The envelope's `policySnapshotId` references the prior snapshot, not the current one. |
| Age-bypass replay requires explicit authorization | `replayAuthorizationRef` is mandatory for `ORIGINAL_POLICY` and `DIAGNOSTIC_ONLY` modes at OACP-Governed+. |
| Hash chains are run-scoped | The replay starts a new chain in `run_replay_001`. The original's chain is preserved separately. |
| Idempotency interacts with replay mode | `ORIGINAL_POLICY` with a long-elapsed original triggers fresh processing rather than dedupe. |
| Signing key validation respects retroactivity | Forward revocation does not invalidate prior signatures; retroactive compromise does. |

A second pattern (not illustrated): `replayMode: DIAGNOSTIC_ONLY` evaluates without policy at all — pure inspection. No downstream effects can occur. Used for "what would this message look like" without affecting any state.
