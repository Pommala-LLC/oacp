# Intellectual property posture

This document describes the intellectual property (IPR) posture of OACP at v0.1.

## Summary

OACP v0.1 is published under the [Apache License 2.0](./LICENSE). The Apache 2.0 license includes both a copyright grant and a patent grant; for v0.1, OACP relies on the Apache 2.0 patent grant as its IPR posture.

A formal IPR policy with explicit patent disclosure obligations, royalty-free covenants, and stakeholder commitments is **deferred until v1.0** or until a multi-stakeholder governance model is established (whichever comes first).

## What this means in practice

For implementers:

- You may implement OACP under the Apache 2.0 license, including the patent grant.
- If you contribute changes back to OACP, your contributions are governed by Apache 2.0's contribution clause (Section 5), which includes a patent grant from contributors covering their contributions.
- The Apache 2.0 patent grant is broad: contributors grant a perpetual, worldwide, non-exclusive, no-charge, royalty-free patent license for their contributions.

For organizations evaluating OACP for adoption:

- Apache 2.0 is a permissive, OSI-approved license used widely in enterprise contexts.
- The patent grant covers contributors' patent claims that read on their contributions, **not** all patents that might apply to implementations of OACP generally.
- A formal IPR policy with broader, protocol-level patent commitments is anticipated for v1.0.

For contributors:

- By contributing, you grant Apache 2.0's patent grant on your contributions.
- You are not required to disclose patents that might read on the protocol generally — that obligation will come with the formal IPR policy at v1.0.

## Why a formal policy is deferred

Pre-1.0, OACP is a working draft authored by POMMALA LLC. Investing in a formal multi-party IPR policy before the protocol's shape is stable would be premature. The deferral is deliberate.

A formal IPR policy will be necessary before v1.0 if OACP is to be adopted broadly across organizations. The expected shape includes:

- **Patent disclosure obligation.** Participants disclose patents they hold that may read on the protocol.
- **Royalty-free covenants.** Stakeholders covenant not to assert disclosed patents against conformant implementations of OACP.
- **Reciprocal terms.** Implementations relying on the protocol covenant similarly.
- **Defensive termination.** Participants who initiate patent litigation against the protocol or other participants lose their license grants.

This is a typical posture for industry-wide protocol standards. The exact shape will be decided as part of the v1.0 governance transition (see [`GOVERNANCE.md`](./GOVERNANCE.md)).

## Trademark posture

The name "OACP" is a **working name**, not a registered trademark. Public name selection is deferred until v0.1 schemas and conformance vectors are stable. When the public name is selected, trademark posture will be addressed as part of the v1.0 governance transition.

The name "Ojas" refers to the reference implementation, not to OACP itself. Use of "Ojas" by parties other than POMMALA LLC may require separate consideration of POMMALA LLC's trademarks.

## Scope of this document

This document covers IPR for the **specification** (the documents in this package: `OACP_SPEC.md`, schemas, bindings, conformance vectors, examples). IPR for implementations of OACP is governed by each implementation's own license and IPR posture.

For the reference implementation Ojas, consult its own license and IPR documents.

## Pre-1.0 disclaimer

OACP v0.1 is a pre-1.0 working draft. Specification changes between v0.x releases may be substantial. Implementers should treat IPR analysis as version-specific and re-examine when adopting newer versions.

## Contact

Questions about IPR posture: see [`GOVERNANCE.md`](./GOVERNANCE.md) for the governance contact.
