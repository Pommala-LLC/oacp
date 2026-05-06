# OACP Binding: CloudEvents 1.0

This document specifies how OACP envelopes are carried as CloudEvents 1.0 messages. CloudEvents is a CNCF specification for describing event data in a common way; it provides a useful structure for OACP to ride on top of.

## Status

Draft for v0.1. Refinement expected based on implementation experience.

## Mapping principle

CloudEvents has its own envelope. OACP has its own envelope. The binding does NOT collapse them into one — it carries the OACP envelope **inside** the CloudEvents `data` field, with a small, well-defined set of CloudEvents context attributes derived from OACP envelope fields for transport-layer routing.

```
+-------------------------------+
| CloudEvents envelope          |
|  - id, source, type, time     |
|  - dataschema (optional)      |
|  - extensions (oacp-*)        |
|  - data: <OACP envelope>      |
+-------------------------------+
```

This preserves OACP's typed envelope contract intact. CloudEvents context attributes are derived metadata for transport routing; the authoritative governance state is inside the OACP envelope.

## Required CloudEvents context attributes

| CloudEvents attribute | OACP source | Notes |
|---|---|---|
| `specversion` | (literal `1.0`) | CloudEvents version. |
| `id` | OACP `messageId` | One-to-one mapping. |
| `source` | OACP `from.participantId` (URI form) | A URI that identifies the OACP participant. |
| `type` | `oacp.<messageKind>.<messageType>` | E.g., `oacp.proposal.proposal-emitted`. |
| `time` | OACP `createdAt` | Same value, ISO-8601. |
| `datacontenttype` | `application/json` (or `application/cbor`, etc.) | Content type of `data`. |
| `dataschema` | URI of the relevant schema | E.g., `https://oacp.dev/schemas/v0.1/envelope.schema.json`. |

## OACP-specific extension attributes

CloudEvents allows extension context attributes. OACP uses these to surface key envelope fields for transport-layer routing without duplicating the entire envelope:

| Extension attribute | OACP source | Purpose |
|---|---|---|
| `oacptenantid` | `scope.tenantId` | For tenant-scoped routing/filtering. |
| `oacprunid` | `run.runId` | For run-scoped routing. |
| `oacpcorrelationid` | `run.correlationId` | For trace correlation. |
| `oacpprofile` | `governance.profile` | For profile-aware routing (e.g., regulated vs operational queues). |
| `oacpidempotencykey` | `idempotencyKey` | For broker-level deduplication. |

CloudEvents extension attribute names are lowercase per CloudEvents spec; OACP follows this rule.

**Note on derived metadata:** these extension attributes are convenience copies. Authoritative values live inside the OACP envelope in `data`. If a CloudEvents attribute and the embedded envelope value disagree, the OACP envelope is authoritative — `ENVELOPE_PAYLOAD_CONFLICT` rules apply (per spec §2b.2).

## Data field

The CloudEvents `data` field carries the full OACP envelope. The encoding follows `datacontenttype`:

- `application/json`: OACP envelope serialized as JSON. JSON Canonicalization Scheme (JCS) is recommended for hash-stable serialization at OACP-Governed+.
- `application/cbor`: CBOR-serialized OACP envelope; integrity hash recomputation must use the same canonicalization.

## Profile compatibility

| Profile | CloudEvents binding behavior |
|---|---|
| `OACP-Core` | CloudEvents `data` carries OACP envelope; extension attributes optional. |
| `OACP-Operational` | All required CloudEvents attributes; `oacptenantid` and `oacprunid` extensions. |
| `OACP-Governed` | All extensions; CloudEvents `dataschema` set to OACP envelope schema URI; transport-layer encryption (TLS) assumed. |
| `OACP-Regulated` | All Governed rules; CloudEvents wire format MUST use a canonicalization that matches OACP `integrity.canonicalization`; CloudEvents context attributes are signed alongside data (typically via JWS over the structured-mode encoding). |

## Structured mode vs. binary mode

CloudEvents supports two transport modes. Both work for OACP:

- **Structured mode:** the entire CloudEvents (context + data) is serialized as a single JSON object. Easier for systems that don't natively manipulate transport headers. Recommended for OACP at all profiles when feasible.
- **Binary mode:** context attributes ride as transport headers (e.g., HTTP headers prefixed `ce-`); `data` is the body. More efficient for binary-data payloads.

OACP-Regulated SHOULD use structured mode, because cryptographic signatures over a single canonical structure are cleaner.

## Transport-specific notes

CloudEvents itself is transport-neutral. Common transports for CloudEvents-bound OACP:

- **HTTP:** structured-mode in body, `Content-Type: application/cloudevents+json`. Binary-mode uses `ce-id`, `ce-source`, `ce-type`, `ce-oacptenantid`, etc. as headers.
- **Kafka:** structured-mode in record value; binary-mode uses Kafka record headers.
- **NATS:** structured-mode in message body; subject naming is deployment-specific.

## Idempotency interaction

CloudEvents `id` and OACP `messageId` are the same value. Transport-level deduplication (e.g., Kafka idempotent producer) operates on this ID. OACP idempotency operates on `idempotencyKey`, which is independent.

**Rule:** transport-level dedup using CloudEvents `id` is for wire-level retries. OACP `idempotencyKey`-based dedup is for logical-operation retries. Both can be active simultaneously without conflict.

## Retry interaction

If a transport retries delivery of a CloudEvents message, the same `id` (= OACP `messageId`) is used. OACP receivers de-dup on `messageId` within `dedupeWindowSeconds` (per §3c).

## Open items

- **Signature encoding:** for OACP-Regulated, the recommended encoding of message-level signatures over CloudEvents structured-mode bodies is JWS Compact Serialization. Detail TBD before v1.0.
- **Trace propagation:** OACP `correlationId` aligns with W3C Trace Context. CloudEvents has its own `traceparent` extension; the recommendation is to populate both when both are present.
- **Schema discovery:** the relationship between CloudEvents `dataschema` and OACP `payloadType` is not fully specified. v0.1 recommends `dataschema` for the envelope schema URI; payload-specific schema discovery uses `payloadType` semantically.
