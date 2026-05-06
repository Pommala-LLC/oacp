# Example 04 — Cross-tenant policy for regulated audit

## Scenario

A regulated deployment has multiple tenants — `tenant_001` and `tenant_002` are operational tenants serving end customers, and `regulator-tenant` is a compliance audit tenant. The regulator-tenant's audit aggregator needs to read audit-chain entries from the operational tenants for compliance review. **This is a cross-tenant flow** and requires explicit cross-tenant policy.

This example illustrates:

- The `CROSS_TENANT` scope type.
- The `crossTenantPolicy` artifact and its registration.
- Cross-tenant references with explicit `crossTenantPolicyRef`.
- Reference-scope compatibility rules (§2d.6a).
- Why receiver authorization for multiple tenants is **not sufficient** to authorize a cross-tenant reference.

**Profile:** `OACP-Regulated`
**Sections illustrated:** §2d.6, §2d.6a, §26.3f

---

## Pre-conditions

A cross-tenant policy is pre-registered in the global policy registry:

```json
{
  "policyId": "ctp_regulated_oversight_v1",
  "policyVersion": "1.0.0",
  "fromTenants": ["tenant_001", "tenant_002"],
  "toTenants": ["regulator-tenant"],
  "allowedReferenceTypes": ["AUDIT_RECORD", "PROPOSAL"],
  "auditRequired": true,
  "policySnapshotRef": "policy-snapshot-ctp-v1",
  "validFrom": "2026-01-01T00:00:00Z",
  "validUntil": "2027-01-01T00:00:00Z",
  "signedBy": "compliance-authority-001",
  "signature": "ed25519:..."
}
```

This policy:

- Authorizes flows from `tenant_001` and `tenant_002` to `regulator-tenant`.
- Restricts to two reference types: `AUDIT_RECORD` and `PROPOSAL`.
- Requires audit-recording on both source and destination chains.
- Is signed by a compliance authority (Regulated profile requires cryptographically signed cross-tenant policies — §2d.11).

The regulator-tenant's audit aggregator (`agent.compliance-aggregator`) is bound and authorized within its own tenant.

---

## Message — CROSS_TENANT-scoped audit request

The compliance aggregator emits a request to read an audit chain from `tenant_001`:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_audit_request_001",
  "messageKind": "EVENT",
  "messageType": "AUDIT_EXPORT_REQUESTED",
  "createdAt": "2026-05-05T18:00:00Z",
  "scope": {
    "scopeType": "CROSS_TENANT",
    "tenantId": "regulator-tenant",
    "environment": "production",
    "crossTenantPolicyRef": {
      "recordId": "ctp_regulated_oversight_v1",
      "recordType": "CROSS_TENANT_POLICY",
      "registry": "policy-registry",
      "scopeType": "GLOBAL"
    }
  },
  "from": {
    "identity": {
      "assurance": "ATTESTED",
      "subject": "principal.agent.compliance-aggregator",
      "credentialRef": "cred_compliance_001",
      "credentialKind": "MTLS_CERTIFICATE",
      "issuedAt": "2026-05-01T00:00:00Z",
      "expiresAt": "2026-08-01T00:00:00Z",
      "issuedBy": "compliance-pki-001"
    },
    "verification": {
      "verified": true,
      "verifiedBy": "regulator-runtime",
      "verifiedAt": "2026-05-05T18:00:00Z",
      "verificationMethod": "MTLS_VERIFY",
      "tenantScope": {
        "authorizedTenants": ["regulator-tenant"],
        "source": "credential-claims"
      }
    },
    "participantId": "agent.compliance-aggregator",
    "participantType": "AGENT"
  },
  "governance": {
    "profile": "OACP-Regulated",
    "profileVersion": "0.1.0",
    "bindingId": "bind_compliance_001",
    "policySnapshotId": "policy-snapshot-regulated-v1"
  },
  "run": {"runId": "run_compliance_001", "correlationId": "corr_compliance_001"},
  "causation": {"causationType": "INITIATOR"},
  "integrity": {
    "canonicalization": "JCS-RFC8785",
    "hashAlgorithm": "sha512",
    "envelopeHash": "sha512:envelope_hash_value...",
    "signatureRef": {
      "recordId": "sig_audit_request_001",
      "recordType": "MESSAGE_SIGNATURE",
      "scopeType": "TENANT",
      "tenantId": "regulator-tenant"
    }
  },
  "authorizationDecision": {
    "decisionRef": {
      "decisionId": "authz_audit_001",
      "decisionType": "AUTHORIZATION",
      "decidedBy": "regulator-policy-engine",
      "decidedAt": "2026-05-05T18:00:00Z",
      "tenantId": "regulator-tenant"
    },
    "decidedBy": "regulator-policy-engine",
    "decidedAt": "2026-05-05T18:00:00Z",
    "scope": {
      "messageType": "AUDIT_EXPORT_REQUESTED",
      "bindingId": "bind_compliance_001",
      "runId": "run_compliance_001",
      "tenantId": "regulator-tenant"
    },
    "result": "ALLOW",
    "policySnapshotId": "policy-snapshot-regulated-v1"
  },
  "payloadType": "audit-export-request-v1",
  "payload": {
    "requestedAuditChainRef": {
      "auditId": "audit_chain_run_op_001",
      "auditChainRef": "chain_run_op_001",
      "tenantId": "tenant_001"
    },
    "exportFormat": "JCS-RFC8785-JSON",
    "deliveryRef": {
      "artifactId": "audit_export_target_001",
      "artifactType": "AUDIT_EXPORT",
      "uri": "ojas://regulator-tenant/audit-exports/audit_export_target_001",
      "mimeType": "application/json",
      "hash": "sha512:placeholder_target_hash",
      "sizeBytes": 0,
      "createdAt": "2026-05-05T18:00:00Z",
      "expiresAt": "2026-05-05T20:00:00Z",
      "tenantId": "regulator-tenant"
    }
  }
}
```

Important features of this envelope:

1. **`scope.scopeType: CROSS_TENANT`** — the message is explicitly cross-tenant scoped.
2. **`scope.tenantId: regulator-tenant`** — the sender's tenant.
3. **`scope.crossTenantPolicyRef`** — references the registered policy. Required by §2d.6 rule 4 for `CROSS_TENANT` scope.
4. **The audit request payload references `tenant_001`** — `requestedAuditChainRef.tenantId: tenant_001`. This is in `crossTenantPolicy.fromTenants`.
5. **The delivery target is in `regulator-tenant`** — `deliveryRef.tenantId: regulator-tenant`. This matches `crossTenantPolicy.toTenants`.
6. **Identity is `ATTESTED`** — this is OACP-Regulated, requiring cryptographic attestation.
7. **Hash algorithm is `sha512`** — recommended at OACP-Governed+ per §2c.5.
8. **Signature is present** — required at OACP-Regulated for capability-scoped messages.

---

## Receiver-side validation

The receiver (the `tenant_001` audit-export service) validates per the gates:

### Gate 1 — Identity verified

The mTLS certificate is verified against `compliance-pki-001`. Identity is `ATTESTED`.

### Gate 1.5 — Tenant scope

The receiver checks `scope.scopeType: CROSS_TENANT`. This is a special path — the receiver:

1. Verifies it is **authorized to handle CROSS_TENANT messages** for `tenant_001`.
2. Resolves `scope.crossTenantPolicyRef` and fetches `ctp_regulated_oversight_v1`.
3. Verifies the policy is `ACTIVE` (not deprecated/blocked/retired).
4. Verifies `tenant_001 ∈ policy.fromTenants` and `regulator-tenant ∈ policy.toTenants`.
5. Verifies `validFrom ≤ createdAt ≤ validUntil`.
6. Verifies the policy's signature.

### Gate 10 — Reference scope compatibility (§2d.6a)

The receiver checks the references in the payload:

- `requestedAuditChainRef.tenantId: tenant_001` — in `policy.fromTenants ∪ policy.toTenants`. ✅
- `deliveryRef.tenantId: regulator-tenant` — same. ✅
- The reference type `AUDIT_RECORD` (and target's `artifactType: AUDIT_EXPORT`) — in `policy.allowedReferenceTypes`. ✅

If any reference were outside the policy's scope, the receiver would reject with `CROSS_TENANT_POLICY_NOT_APPLICABLE`.

### Gate 12 — Authorization decision

The receiver verifies the regulator-tenant's `authorizationDecision.result: ALLOW`. The decision was emitted by `regulator-policy-engine`, valid in regulator-tenant.

The receiver also verifies its own authorization to fulfill the request — typically via a separate authorization decision in `tenant_001`.

### Gate 13 — Integrity

Hash chain validation, signature verification (Regulated requires both).

### Audit recording on both chains

Per `policy.auditRequired: true`, the export operation must be recorded:

1. On `tenant_001`'s audit chain (the source): "audit_chain_run_op_001 was exported to regulator-tenant under policy ctp_regulated_oversight_v1".
2. On `regulator-tenant`'s audit chain (the destination): "audit export from tenant_001 received under policy ctp_regulated_oversight_v1".

---

## What if the policy were missing?

Suppose the same envelope arrives but `scope.crossTenantPolicyRef` is absent:

- Receiver rejects at gate 1.5 with `CROSS_TENANT_POLICY_REF_MISSING`.

Or the envelope has `crossTenantPolicyRef` but the policy doesn't authorize this specific tenant pair:

- Receiver rejects at gate 1.5 with `CROSS_TENANT_POLICY_NOT_APPLICABLE`.

Or the envelope embeds a `crossTenantPolicy` block instead of referencing one:

- Receiver rejects with `CROSS_TENANT_POLICY_EMBEDDED` (§2d.6 rule 6). Cross-tenant policies must be **registered**, not embedded.

---

## Why receiver multi-tenancy doesn't bypass this

Conformance vector `CV-0140-cross-tenant-no-policy` (in `../conformance/vectors/tenant-scope/`) tests this:

> The receiver is technically authorized for both tenant_001 AND tenant_002, but that is NOT what authorizes a cross-tenant reference.

A receiver's **authorized scope** describes which tenants' messages it may *process*. It does NOT authorize **cross-tenant references** within those messages. Cross-tenant references require explicit, registered policy regardless of receiver scope.

This protects against tenant boundary erosion via shared receiver infrastructure. Even if `regulator-tenant` and `tenant_001` are served by the same physical runtime, the boundary is enforced by policy, not by infrastructure topology.

---

## What this demonstrates

| Invariant | How |
|---|---|
| Cross-tenant flows require explicit registered policy | Policy `ctp_regulated_oversight_v1` is referenced by `crossTenantPolicyRef`. |
| `CROSS_TENANT` scope requires `crossTenantPolicyRef` | The envelope declares `scopeType: CROSS_TENANT` with the reference. |
| Reference scope compatibility | All references in the payload conform to `crossTenantPolicy.fromTenants ∪ toTenants` and `allowedReferenceTypes`. |
| Audit on both chains | Per `policy.auditRequired: true`, the export is recorded on both source and destination tenants. |
| Receiver multi-tenancy ≠ cross-tenant authorization | Even authorizing both tenants, the receiver would reject without the explicit policy. |
| Regulated profile requirements | `ATTESTED` identity, `sha512` hash, signature reference, signed cross-tenant policy. |
| Policies are referenced, not embedded | Embedding the policy in the envelope is rejected with `CROSS_TENANT_POLICY_EMBEDDED`. |
