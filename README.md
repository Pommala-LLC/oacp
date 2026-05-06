# OACP — Open Agent Communication Protocol (working draft)

**Version:** 0.1 (working draft)  
**Status:** Draft for review  
**License:** Apache 2.0  

OACP is a **transport-neutral, contract-first protocol for governed agent-runtime communication**. It defines a typed envelope, deterministic processing gates, and a two‑axis profile system (Core → Operational → Governed → Regulated) that makes agent communication **auditable, replay‑safe, and policy‑enforceable**.

> **Safety boundary:** OACP is an **operational AI governance protocol**. It helps make agent actions attributable, scoped, policy‑bound, reviewable, and auditable. It does **not** guarantee model alignment, truthfulness, or safety. Those remain the responsibility of the model provider and deployment environment.

---

## 📦 Repository contents

- [`OACP_SPEC.md`](./OACP_SPEC.md) – The full v0.1 specification (approx. 160 KB).  
- [`schemas/`](./schemas) – JSON Schema 2020-12 definitions for the envelope and core artifacts.  
- [`bindings/`](./bindings) – Transport bindings (CloudEvents, AsyncAPI, OpenAPI, OpenTelemetry).  
- [`conformance/`](./conformance) – Conformance test vectors.  
- [`examples/`](./examples) – End‑to‑end example flows.  
- `GOVERNANCE.md`, `CONTRIBUTING.md`, `SECURITY.md`, `IPR.md` – Project governance and legal posture.

---

## 🧭 Why OACP exists

Current agent communication protocols (A2A, MCP, ACNBP) solve transport, tool integration, and capability negotiation. **None** prescribe what *governed* agent communication looks like:
- What makes a message replay‑safe?  
- What constitutes an approval vs. a proposal?  
- How do you enforce tenant isolation across references?  
- What audit chain must a runtime produce?

OACP fills that gap. It layers governance semantics on top of existing protocols without replacing them.

---

## 🔑 Key features

- **Deterministic processing gates** – 15 validation steps (identity, tenant scope, authority, binding, authorization decision, idempotency, broker state, causation DAG, hash chain, etc.) before any business effect.
- **Two‑axis profiles** – Base profiles (Core → Regulated) for assurance strictness, plus additive capability profiles (e.g., `OACP-Tool`, `OACP-Review`, `OACP-Proposal`).
- **Typed references** – `ArtifactRef`, `MessageRef`, `RecordRef`, `DecisionRef`, `AuditRef` with tenant‑scoped resolution.
- **Proposal/decision separation** – Agents emit proposals; approvals require separate decision artifacts. Self‑approval is forbidden.
- **Run‑scoped hash chains** – Immutable audit trails per execution instance.
- **Extensions that cannot weaken core** – Namespaced extensions can add stricter validation but never disable a core gate.

See [`OACP_SPEC.md`](./OACP_SPEC.md) for the complete specification.

---

## 🚦 Status

| Tier | Description | Status |
|------|-------------|--------|
| Tier 1 – Protocol Works | Identity, binding, idempotency, broker, conversation | ✅ Frozen |
| Tier 2 – Protocol Is Precise | Profiles, references, tenant isolation, review, proposal/decision | ✅ Frozen |
| Tier 3 – Protocol Is Publishable | Threat model, versioning, schemas, bindings, conformance | ✅ Frozen (v0.1) |
| Tier 4 – Governance‑Rich | Obligations, data handling, side‑effect classes, budgets, kill‑switch | 🔜 Scoped – v0.2 |
| Tier 5 – Ecosystem‑Ready | Federation, formal registry, advanced regulated profiles | 🔜 Scoped – later |

v0.1 is a **working draft**. Backward compatibility between v0.x releases is best‑effort; pin to a specific version for production experiments.

---

## 📖 Getting started

1. **Read the specification** – Start with [`OACP_SPEC.md`](./OACP_SPEC.md) §0 (Foreword) and §1 (Conventions).  
2. **Explore the examples** – [`examples/`](./examples) contains step‑by‑step walks and standalone JSON envelopes.  
3. **Run conformance tests** – Use the vectors in [`conformance/vectors/`](./conformance/vectors) to validate your implementation.  
4. **Check the threat model** – See §26 of the spec for eight threat surfaces and mitigation strategies.

---

## 🤝 Contributing

We welcome contributions that improve the specification, clarify ambiguities, fill threat model gaps, or add conformance test vectors.  
See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for detailed guidelines.

For security issues, follow the process in [`SECURITY.md`](./SECURITY.md). **Do not open a public issue for security findings.**

---

## 📄 License and IPR

OACP v0.1 is published under the **Apache License 2.0**. A formal IPR policy with patent non‑assertion commitments is deferred until v1.0. See [`IPR.md`](./IPR.md) for details.

---

## 🧭 Reference implementation

**Ojas** is the reference implementation of OACP. It is not required for OACP conformance; the protocol is implementation‑neutral.

---

## 📬 Contact

For questions about the specification or governance, please open a GitHub Discussion or reach out via the maintainer contact in [`GOVERNANCE.md`](./GOVERNANCE.md).
