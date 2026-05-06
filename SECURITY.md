# Security policy

This document describes OACP's security posture, the threat model summary, and how to report security issues.

For the full threat model with all eight threat surfaces and out-of-scope threats, see [`OACP_SPEC.md`](./OACP_SPEC.md) §26.

## Reporting a vulnerability

Security issues in OACP — the protocol specification or its reference materials — should be reported privately. **Do not open a public issue for security-sensitive findings.**

To report a vulnerability:

1. Email the maintainers at the address listed in [`GOVERNANCE.md`](./GOVERNANCE.md).
2. Include: a description of the issue, the affected section(s) of the spec or schemas, a reproduction or attack scenario if applicable, and the impact you anticipate.
3. Allow a reasonable time for response before disclosure. The exact disclosure window is negotiable but will not be unreasonably extended.

For vulnerabilities in implementations of OACP (including Ojas), report to that implementation's security contact. OACP itself is a specification; implementations have their own disclosure processes.

## Security posture

OACP v0.1 is a **specification**, not a runtime. It defines what conformant governed agent communication looks like; it does not run code. Security findings against the specification fall into three categories:

| Category | Examples |
|---|---|
| **Specification ambiguity** | A field whose semantics permit two interpretations, one of which weakens governance. |
| **Specification gap** | A threat the specification does not address but reasonably should. |
| **Specification inconsistency** | Two sections that contradict each other in ways that produce a security-relevant divergence. |

Implementation bugs (e.g., a runtime that fails to validate `previousMessageHash`) are out of scope here; report those to the implementation's security contact.

## Threat model summary

OACP's threat model is built on three principles:

1. **Trust the protocol; verify the participant.** OACP defines what a conformant participant looks like. It does not assume any participant is trustworthy. Receivers verify identity, authority, and integrity at every hop.

2. **The runtime is the trust root, but it is not infallible.** OACP detects many forms of compromise — forged messages, replays, authority bypass — but cannot detect a fully compromised runtime emitting valid messages with full authority. A compromised runtime is a catastrophic failure outside OACP's prevention surface; OACP minimizes the blast radius via tenant isolation, audit chains, and cryptographic attestation at higher profiles.

3. **Auditability ≠ prevention.** Many threats are addressed by audit chains (detection) rather than by validation (prevention). The threat model is explicit about which threats are prevented and which are merely detectable.

### Eight threat surfaces

The full enumeration is in §26.3 of the spec. Briefly:

| Surface | What it covers | Primary mitigation |
|---|---|---|
| Identity and impersonation | Forged identity claims, stolen credentials, replayed tokens, verification trail tampering | Identity verification gate (§3a), per-hop verification trail, profile-driven assurance |
| Capability misuse | Agent uses capability not in manifest or outside binding scope | Manifest registry immutability, binding evaluation at receive time |
| Authorization bypass | Forged or stale authorization decisions, omitted decision blocks | `authorizationDecision` required at Governed+, scope binding, signature at Regulated |
| Replay and freshness | Replay of old messages, cross-run replay, decision replay | `maxMessageAgeSeconds`, idempotency keys, replay metadata with authority verification |
| Audit chain integrity | Hash chain breakage, stripped extensions, segmented chains | `previousMessageHash` validation, extension preservation rule, off-runtime audit at Regulated |
| Tenant isolation breach | Cross-tenant messages, references, identity scope violations | Four enforcement points (§2d.3), `verification.tenantScope`, cross-tenant policy required |
| Decision integrity | Forged decisions, self-decision, retroactive backdating | Self-decision rule (§13.6), authority verification, policy snapshot consistency |
| Extension abuse | Required extensions claiming envelope authority, extensions that weaken core, schema SSRF | Tier hierarchy enforcement, `EXTENSION_WEAKENS_CORE`, trusted resolver rule |

### Out of scope for v0.1

OACP v0.1 explicitly does **not** defend against:

- Compromised runtime (catastrophic failure mode).
- Compromised authorization engine within the compromised tenant.
- Side-channel attacks on runtime infrastructure.
- Network-layer attacks below the OACP envelope (TLS termination, DNS hijacking).
- Denial-of-service at the transport layer.
- Model-level attacks on agent reasoning (prompt injection, jailbreak).
- Long-term cryptographic compromise of signature keys via cryptanalysis.
- Insider threats with full administrative access to the runtime.

These are addressed by deployment infrastructure, not by OACP.

## Security Boundary

OACP implements operational governance support for alignment oversight, model-safety workflows, and truthfulness-verification processes. OACP does not implement or guarantee model alignment, model safety, or truthfulness.

OACP provides governance primitives — identity, authority, capability binding, authorization decisions, tenant scope, tool mediation, review, proposal/decision separation, evidence references, validation results, audit chains, replay protection, and hash-chain integrity — that make these oversight workflows attributable, reviewable, auditable, and policy-enforceable.

Future OACP v0.2 candidates strengthen this support through RunIntent gates, reasoning trace evidence, output safety classification, evidence quality scoring, tool side-effect classes, policy simulation, kill-switch semantics, and data handling / purpose limitation.

Implementations claiming OACP conformance for AI-safety contexts must not represent the protocol as a model-safety, alignment, or truthfulness guarantee. Those properties are characteristics of the model and its deployment that OACP makes evaluable but does not establish.

## Security-relevant deferrals

The following security-relevant items are deferred to **Tier 4 / v0.2**:

- Message-level encryption (v0.1 relies on transport-layer confidentiality).
- Quarantine release flow (v0.1 makes `QUARANTINED` a terminal state).
- Side-effect class enforcement beyond `READ_ONLY` declarations.
- Multi-runtime cross-validation for compromised-runtime detection.
- Cryptographic attestation of runtime binaries.
- Post-quantum signature algorithms.

Deployments with elevated security needs should be aware of these gaps and apply compensating controls (immutable audit storage, multi-runtime consensus, hardware attestation) at the deployment layer.

## Cryptographic algorithms

Reserved hash algorithms:

| Algorithm | Minimum profile | Status |
|---|---|---|
| `sha256` | Operational | Default |
| `sha512` | Governed | Recommended at Governed+ |
| `blake3` | Optional | Reserved |

Reserved signature algorithms (v0.1):

- `Ed25519`
- `ECDSA-P256`
- `RSA-PSS-SHA256`

Algorithm agility is supported via the algorithm prefix in hash fields and explicit `signatureAlgorithm` declaration. Migration to post-quantum algorithms is anticipated for Tier 5 / v0.3+.

## Profile-specific security guidance

| Profile | Security expectations |
|---|---|
| `OACP-Core` | Trusted in-process flows. No cryptographic protection assumed. |
| `OACP-Operational` | Authentication and message-age enforcement. Transport-layer encryption assumed (TLS or equivalent). |
| `OACP-Governed` | Full envelope governance. Capability binding, authorization decisions, tenant isolation enforced. Hash chains validated detectively. |
| `OACP-Regulated` | All Governed rules + cryptographic attestation, signed decisions, hard-blocking hash chain validation, immutable audit. Separation of authorization engine from runtime recommended. |

Implementors at Governed and above should consult §26.7 for implementation guidance.

## Disclosure history

No disclosed vulnerabilities at v0.1.

---

*OACP is licensed under Apache 2.0 (see [`LICENSE`](./LICENSE)). The license includes a patent grant; security findings that would invoke patent claims should be reported under this disclosure process.*
