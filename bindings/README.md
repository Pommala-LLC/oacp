# OACP Transport Bindings

This directory documents how OACP envelopes ride on specific transports. OACP itself is transport-neutral — the envelope shape and processing semantics are defined in `OACP_SPEC.md` independent of how messages move on the wire.

A **binding** documents:

- How the envelope is serialized on the transport.
- How transport-level metadata maps to OACP envelope fields (or is intentionally left disjoint).
- How transport-level features (retry, ordering, ack) interact with OACP semantics.
- Which OACP profiles this binding supports cleanly, which require additional compensation.

## Available bindings (v0.1)

| File | Format | Status |
|---|---|---|
| [`OACP_CLOUDEVENTS_BINDING.md`](./OACP_CLOUDEVENTS_BINDING.md) | Markdown narrative | Draft |
| [`OACP_ASYNCAPI.yaml`](./OACP_ASYNCAPI.yaml) | AsyncAPI 3.0 (machine-readable) | Starter / Draft |
| [`OACP_OPENAPI.yaml`](./OACP_OPENAPI.yaml) | OpenAPI 3.1 (machine-readable) | Starter / Draft |
| [`OACP_OTEL_MAPPING.md`](./OACP_OTEL_MAPPING.md) | Markdown narrative | Draft |

The CloudEvents and OpenTelemetry bindings remain Markdown narratives because their primary value is conceptual mapping; CloudEvents has its own context-attribute spec and OpenTelemetry is an observability framework, not a transport. The AsyncAPI and OpenAPI bindings are machine-readable YAML so they can drive code generation, schema validation, and tooling adoption.

These bindings are **drafts** at v0.1. The YAML files are **starter quality** — correct structure, valid against their respective metaschemas, illustrative endpoints — but not a fully fleshed-out production API surface. Refinement based on implementation experience is expected before v1.0.

## Bindings not yet covered

The following transports have plausible bindings but are not yet documented:

- gRPC (protobuf serialization, streaming)
- WebSocket (bidirectional, framed)
- MQTT (pub/sub, QoS levels)
- Kafka (partitioned topic, ordering guarantees) — though the AsyncAPI YAML uses Kafka as its illustrative server
- AMQP / RabbitMQ
- Server-Sent Events (one-way streaming)
- HTTP polling

Contributions of bindings for these transports are welcomed — see [`../CONTRIBUTING.md`](../CONTRIBUTING.md).

## What a binding does NOT do

A binding does not modify OACP semantics. It documents how the envelope rides on a transport. If a transport cannot support a particular OACP feature, the binding documents the limitation and any compensating mechanism.

Bindings do not change:

- Envelope structure.
- Reserved values (message kinds, types, decision types, etc.).
- Processing gate semantics.
- Profile requirements.

Bindings DO change:

- How the envelope is encoded (JSON, protobuf, CBOR).
- How envelope fields surface as transport metadata (HTTP headers, Kafka headers, MQTT topics).
- How idempotency keys interact with transport-level retry.
- How signatures are conveyed.

## Profile compatibility

| Profile | Typical transports |
|---|---|
| `OACP-Core` | In-process function calls, named pipes, shared memory |
| `OACP-Operational` | HTTP/REST (see `OACP_OPENAPI.yaml`), gRPC, AsyncAPI-described message brokers |
| `OACP-Governed` | All of the above, plus durable message brokers (Kafka, RabbitMQ) for audit chain durability |
| `OACP-Regulated` | Durable, signed transports; audit chain to immutable storage; mTLS or equivalent for transport-layer protection |

The binding documents in this directory expand on these defaults.

## Multiple bindings in one deployment

Real deployments often use multiple bindings simultaneously:

- HTTP/REST (per `OACP_OPENAPI.yaml`) for synchronous request-response between agent and runtime.
- Kafka (per `OACP_ASYNCAPI.yaml`) for asynchronous event streaming and audit chain persistence.
- OpenTelemetry (per `OACP_OTEL_MAPPING.md`) for trace export.
- AsyncAPI documents to describe broker topics.

OACP supports this. The envelope is the same; bindings just describe how it travels in each context.

## Validating the YAML bindings

Both YAML files use `$ref` pointers to the schemas in `../schemas/`. To resolve them in tooling:

- Use a JSON Schema 2020-12 validator (OpenAPI 3.1's dialect aligns with OACP).
- Bundle the schemas alongside the bindings (most generators support this).
- For AsyncAPI, the official AsyncAPI tools (https://www.asyncapi.com/tools) parse the document and validate it.
- For OpenAPI, Spectral or openapi-generator parse and validate the document.

The starter YAML files are intentionally minimal. Production deployments will extend them with deployment-specific channels, paths, and security schemes.
