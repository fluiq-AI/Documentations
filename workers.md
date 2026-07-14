# Fluiq Workers — Shared Architecture Reference

The workers project (`fluiq-workers/`) contains **three** independent async Kafka
consumers: **Tracer**, **Evaluator**, and **Security**. They have no HTTP surface —
all communication is through Kafka topics and writes to ClickHouse / PostgreSQL.

This file covers only what the three workers **share** (running, Kafka topics,
auth, message flow, producer). Each worker's internals have their own reference:

| Worker | Consumes | Does | Reference |
|--------|----------|------|-----------|
| **Tracer** | `KAFKA_TRACE_TOPIC` | Persists SDK events to ClickHouse, resolves `root_trace_id`, estimates cost | [`docs/tracer.md`](tracer.md) |
| **Evaluator** | `KAFKA_EVAL_TOPIC` | LLM-as-judge single-shot + agentic (L1–L5) + vision eval; calibration | [`docs/evaluator.md`](evaluator.md) |
| **Security** | `KAFKA_SECURITY_TOPIC` | PII/secrets + full agentic-threat scan (incl. image-injection OCR); pre-call & response gates | [`docs/security.md`](security.md) |

The Security worker is deployed **separately** from the Evaluator so its heavy
dependencies (torch + spaCy/Presidio + sentence-transformers) don't bloat the
evaluator's memory footprint or block judge calls.

---

## Running

```bash
# From the fluiq-workers directory, each worker is its own process
python -m tracer.app
python -m evaluator.app
python -m security.app

# Or via Docker Compose (starts all workers + Kafka + ClickHouse + PostgreSQL)
docker-compose up
```

Each worker has its own `config.py` and `db/` package (clickhouse / kafka /
postgresql). All three share an identical `kafka_auth_kwargs()` helper so they can
run against local PLAINTEXT Kafka or AWS MSK with SASL/SCRAM over TLS (see below).

---

## Kafka Topics

| Topic (env var) | Producer | Consumer | Description |
|-----------------|----------|----------|-------------|
| `KAFKA_TRACE_TOPIC` (`traces`) | fluiq-api | Tracer | Raw trace events from the SDK (and OTLP-ingested external traces) |
| `KAFKA_EVAL_TOPIC` (`evaluations`) | fluiq-api | Evaluator | Auto-retrieval eval, explicit eval, SDK LLM eval, agentic eval, playground eval |
| `KAFKA_SECURITY_TOPIC` | fluiq-api | Security | Async post-call scans + synchronous pre-call / response-gate checks |
| `KAFKA_TRACE_PERSISTED_TOPIC` (`traces.persisted`) | all three workers | fluiq-api (SSE + alert consumer) | Enrichment fan-out (`started` / `persisted` / `enriched`) |
| `KAFKA_SECURITY_REPLY_TOPIC` | Security | fluiq-api | Synchronous request-reply for pre-call check & response gate |
| `KAFKA_PLAYGROUND_REPLY_TOPIC` | Evaluator | fluiq-api | Synchronous request-reply for the prompt playground |

The security topic is **dedicated** — it is *not* the evaluations topic. The
security worker routes a message with no explicit `operation` as a full post-call
scan.

---

## Kafka Authentication — `kafka_auth_kwargs()`

Every worker `config.py` exposes an identical helper used for both consumer and
producer:

```python
def kafka_auth_kwargs() -> dict:
    # PLAINTEXT (default, local docker-compose)        → no auth
    # SASL_SSL → SCRAM-SHA-512 username/password over TLS (AWS MSK)
```

| Variable | Default | Description |
|----------|---------|-------------|
| `KAFKA_SECURITY_PROTOCOL` | `PLAINTEXT` | `PLAINTEXT` (local) or `SASL_SSL` (MSK) |
| `KAFKA_SASL_MECHANISM` | `SCRAM-SHA-512` | SASL mechanism for MSK |
| `KAFKA_SASL_USERNAME` | — | MSK SCRAM user |
| `KAFKA_SASL_PASSWORD` | — | MSK SCRAM password |
| `KAFKA_MAX_FETCH_BYTES` | `10 MiB` | Consumer per-partition fetch ceiling (trace events can be multi-MB) |
| `KAFKA_MAX_REQUEST_SIZE` | `10 MiB` | Producer max request size (tracer re-publish) |

MSK broker certs chain to Amazon Trust Services (in the default CA bundle), so no
CA file is needed — `ssl.create_default_context()` is used.

> **Production topology (2026):** MSK was the single largest line item on the AWS
> bill, so production Kafka was migrated to a **single-node self-hosted broker on
> EC2** speaking **PLAINTEXT** inside the VPC; the MSK cluster is decommissioned.
> The `kafka_auth_kwargs()` SASL_SSL path is retained for portability but is not
> used in production today. This is a deliberate elasticity-over-HA trade-off (see
> `docs/deployments.md`).

---

## Message Flow

```
SDK / fluiq-api
  │
  ├─ [pre-call check, block mode] ─► Kafka: KAFKA_SECURITY_TOPIC "security_check_sync"
  │                                        └─ Security Worker (sync reply)
  │                                              └─ KAFKA_SECURITY_REPLY_TOPIC → /secure/check
  │
  ├─ [response gate] ─────────────► Kafka: KAFKA_SECURITY_TOPIC "response_gate_check"
  │                                        └─ Security Worker (sync reply)
  │                                              └─ KAFKA_SECURITY_REPLY_TOPIC → /ingest embeds result
  │
  ├─ Kafka: traces ──────────────► Tracer Worker            (docs/tracer.md)
  │                                   ├─ ClickHouse: INSERT traces
  │                                   ├─ traces.persisted "persisted"
  │                                   ├─ PostgreSQL: SELECT model_prices
  │                                   ├─ ClickHouse: INSERT trace_costs
  │                                   └─ traces.persisted "enriched/cost"
  │
  ├─ Kafka: evaluations ─────────► Evaluator Worker         (docs/evaluator.md)
  │                                   ├─ judge eval (auto / sdk_llm / agent_eval / explicit / playground)
  │                                   ├─ ClickHouse: INSERT evaluations
  │                                   └─ traces.persisted "enriched/evaluation"
  │                                   └─ (playground) KAFKA_PLAYGROUND_REPLY_TOPIC
  │
  └─ Kafka: KAFKA_SECURITY_TOPIC ─► Security Worker (async)  (docs/security.md)
                                       ├─ ClickHouse: INSERT <security table>
                                       └─ traces.persisted "enriched/security"

traces.persisted ─────────────────► fluiq-api SSE route        (live dashboard)
                 └────────────────► fluiq-api alert_consumer    (Slack alerts)
```

---

## Kafka Producer Settings

All workers use the same producer config (`db/kafka.py`):

| Setting | Value | Reason |
|---------|-------|--------|
| `enable_idempotence` | `True` | Exactly-once delivery per partition |
| `acks` | `all` | Wait for full ISR replication |
| `key` | `organization_id` | Consistent per-org ordering / partitioning |
| Value serializer | JSON with `Decimal → str` | Safe decimal serialization |
| `**kafka_auth_kwargs()` | PLAINTEXT or SASL_SSL | Local docker vs AWS MSK |

The producer's `default_topic` is `KAFKA_TRACE_PERSISTED_TOPIC`; sync replies pass
an explicit `topic=` (security / playground reply topics).
