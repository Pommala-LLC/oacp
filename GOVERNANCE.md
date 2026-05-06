# Governance

This document describes how decisions about OACP are made — who authors changes, how disputes are resolved, and how the protocol evolves.

For questions about how OACP itself governs *agent communication*, see [`OACP_SPEC.md`](./OACP_SPEC.md). This document is about governance *of* the protocol, not governance *by* it.

## Status

OACP v0.1 is a working draft authored by **POMMALA LLC** as part of the Ojas project. Ojas is the reference implementation.

**Until v1.0**, governance is held by POMMALA LLC. Decisions about the specification — what's in scope, what's deferred, how reviewer feedback is incorporated, what constitutes a frozen invariant — are made by POMMALA LLC and disclosed in `CHANGES.md`.

**At or before v1.0**, governance will transition to a multi-stakeholder model. The transition timing depends on:

- Adoption of OACP by implementations beyond Ojas.
- Resolution of pre-1.0 open items (Tier 4 capabilities, formal IPR posture, conformance test suite stability).
- Identification of stakeholder organizations willing to participate in governance.

## How decisions are made (pre-1.0)

Pre-1.0 decisions follow a structured process:

1. **Proposal.** A change is proposed — a new capability profile, a new error code, a clarification, a breaking change — with rationale.
2. **Review.** The proposal is reviewed against existing invariants (see [`README.md`](./README.md) — *Architectural fundamentals*). A change that violates a frozen invariant requires explicit invariant modification.
3. **Decision.** POMMALA LLC accepts, defers, or rejects. Rejected proposals are recorded with reason. Deferred proposals are tracked for a future tier.
4. **Disclosure.** Accepted changes are recorded in `CHANGES.md` with the version they apply to.

Pre-1.0, all decisions are reversible. A change accepted in v0.2 may be reverted in v0.3 if it proves problematic. Post-1.0, reversibility is constrained by backward-compatibility rules (see §27 of the spec).

## What's frozen and what isn't

Pre-1.0 OACP has multiple stability levels:

| Stability level | Examples | Pre-1.0 change rules |
|---|---|---|
| **Frozen invariants** | Sender-immutable identity, proposal/decision separation, run-scoped hash chains, tenant isolation as protocol-level invariant | Will not be relaxed. Strengthening permitted. |
| **Reserved values** | `messageKind`, `messageType`, `causationType`, `decisionType`, lifecycle states | Additions permitted; semantics stable per major version. |
| **Profile registry** | Base profiles, capability profiles | Base registry major-version-closed; capability registry controlled-additive. |
| **Error codes** | The ~150 reserved codes | Additions permitted; existing codes' semantics stable. |
| **Schemas** | JSON Schema 2020-12 definitions | Pre-1.0: best-effort backward compat; post-1.0: strict per major. |
| **Wire format** | Envelope structure, JCS canonicalization | Stable within major; structural changes require major version. |

The **frozen invariants** are the strongest commitment. Everything else is "stable but evolvable."

## Roles

Pre-1.0:

| Role | Held by | Responsibility |
|---|---|---|
| **Specification author** | POMMALA LLC | Writes and maintains the spec. |
| **Reference implementer** | POMMALA LLC (Ojas) | Implements the spec; bug reports flow back into spec corrections. |
| **Reviewer** | Open | Submits feedback via the contribution process. |

Post-1.0 (anticipated):

| Role | Held by | Responsibility |
|---|---|---|
| **Steering committee** | TBD | Strategic direction, version planning, conflict resolution. |
| **Maintainer team** | TBD | Day-to-day spec maintenance, schema changes, error code additions. |
| **Implementer working group** | TBD | Coordination across implementations; conformance test suite. |
| **Security working group** | TBD | Threat model evolution, vulnerability disclosure. |

The composition of these groups will be determined as the v1.0 transition approaches.

## Conflict resolution

Pre-1.0:

1. Technical disagreements are resolved by the specification author with explicit rationale recorded in `CHANGES.md`.
2. Disputes over invariants require a structured argument: which invariant is at stake, what concrete scenario reveals the conflict, what alternative invariants would resolve it.
3. Reviewer feedback is welcomed but not binding. Pre-1.0, the spec author has final say.

Post-1.0:

- Routine technical disagreements: maintainer team consensus, escalated to steering committee on impasse.
- Invariant changes: steering committee with documented stakeholder input.
- Security findings: security working group, with steering committee approval for spec-level changes.

## Version cadence

Pre-1.0:

- **v0.x releases:** when material progress justifies a release. No fixed cadence.
- **CHANGES.md:** updated with every release.
- **Breaking changes:** permitted between v0.x releases; documented explicitly.

Post-1.0 (anticipated):

- **Major versions (v1, v2):** rare; only for breaking changes. No more than one per ~24 months expected.
- **Minor versions (v1.1, v1.2):** roughly quarterly to semi-annually for additive changes.
- **Patch versions (v1.1.1):** as needed for clarifications and security fixes.

## Relationship to Ojas

**OACP and Ojas are distinct.** OACP is the specification; Ojas is the reference implementation. The two evolve in coordination but do not move in lockstep.

- A change in Ojas that requires a spec change goes through this governance process before being published as part of OACP.
- A change in OACP becomes binding on Ojas in a subsequent Ojas release.
- Ojas-specific behavior beyond OACP (deployment-specific extensions, internal mechanics) is not part of OACP and is not governed here.

Implementations of OACP other than Ojas are welcomed and encouraged. OACP's value depends on multiple conformant implementations existing.

## Contact

Pre-1.0, governance contact is via POMMALA LLC. Specific addresses for governance questions, security disclosures, and contribution coordination will be added as the project formalizes its public infrastructure.

For now: feedback on v0.1 should follow the process in [`CONTRIBUTING.md`](./CONTRIBUTING.md).

## Open governance items

The following governance items are open and tracked for resolution before v1.0:

- Specific organizations to invite into the steering committee.
- Mechanism for stakeholder representation (working groups, voting weights, etc.).
- Hosting model for the specification (foundation membership, neutral hosting, etc.).
- Trademark posture for the OACP name (currently working name; public name selection deferred).
- Formal IPR policy (currently lightweight, tied to Apache 2.0; see [`IPR.md`](./IPR.md)).
- Conformance test suite governance — who certifies, how disputes are resolved.

These items are intentionally not pre-decided. They will be resolved through the same structured process that governs spec changes: proposal, review, decision, disclosure.
