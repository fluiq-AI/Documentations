# Fluiq Tracer — Internal Architecture Reference

The **Tracer** worker (`fluiq-workers/tracer/`) is an async Kafka consumer that
persists SDK trace events and enriches them with cost. It has no HTTP surface —
it consumes `KAFKA_TRACE_TOPIC`, writes to ClickHouse, and fans out enrichment
messages on `traces.persisted`. Shared worker concerns (running, Kafka topics,
auth, producer settings, overall message flow) live in `docs/workers.md`.

---

## Entry point: `tracer/app.py`

Consumes `KAFKA_TRACE_TOPIC` and dispatches on the `operation` field (default
`"ingest"` → `ingest_trace`). Manual commit after each message.

**Environment variables**

| Variable | Default | Description |
|----------|---------|-------------|
| `KAFKA_BOOTSTRAP_SERVERS` | `localhost:9092` | Kafka broker |
| `KAFKA_TRACE_TOPIC` | `traces` | Input topic |
| `KAFKA_TRACE_GROUP_ID` | `tracer-workers` | Consumer group |
| `KAFKA_TRACE_PERSISTED_TOPIC` | `traces.persisted` | Output topic |
| `POSTGRES_DSN`, `POSTGRES_POOL_MIN/MAX`, `POSTGRES_SSL_CA_FILE` | — | Postgres pool for model-price lookups (TLS verify-full when CA set) |
| `CLICKHOUSE_*` | — | Host / port / user / password / database |
| `CLICKHOUSE_TRACE_TABLE` | `traces` | Traces table |
| `CLICKHOUSE_TRACE_COSTS_TABLE` | `trace_costs` | Costs table |

---

## Ingest job: `tracer/jobs/ingest.py`

**Input message schema** (from `traces` topic)

```json
{
  "organization_id": "uuid",
  "api_key_prefix": "flq_abc123",
  "trace_id": "uuid",
  "event": {
    "trace_id": "uuid",
    "parent_id": "uuid | null",
    "parent_ids": ["uuid", "..."],
    "type": "llm | function | tool | vectorstore | mcp",
    "status": "running | complete | error | blocked",
    "integration": "OpenAI | Anthropic | Gemini | ...",
    "model": "gpt-4o",
    "tokens": { "prompt": 100, "completion": 50, "total": 150 },
    "prompt_cache_read_tokens": 0,
    "prompt_cache_creation_tokens": 0,
    "prompt_cached_tokens": 0
  }
}
```

`prompt_cache_read_tokens` / `prompt_cache_creation_tokens` are set on Anthropic
LLM traces. `prompt_cached_tokens` is set on OpenAI and Gemini LLM traces.
`parent_ids` (optional) carries the multiple parents of a DAG join / fan-in node
(LangGraph / CrewAI / Google ADK / `fluiq.join_parents`); it's stored verbatim and
read by the evaluator's agentic graph (see `docs/evaluator.md`). Multimodal
content parts arrive as payload-free `_media_ref`s (see `docs/sdk.md`).

**Processing steps**

1. Generate `trace_id` if missing.
2. Resolve `root_trace_id` **and an `is_root` flag** via the root resolver (LRU cache → ClickHouse → fallback). `is_root` encodes "own root OR orphan (phantom parent never captured)"; a self-referential `parent_id == trace_id` (e.g. CrewAI's crew span) is forced to `is_root = 1`.
3. Derive the denormalized **agent identity** (`agent_key`, `agent_kind`) from the event: `function` → `chain`/`function`, else `langgraph_node`, else empty. This is what the Agents roll-up groups by, so it's computed once at ingest instead of scanned at read time.
4. Stamp **`retention_days`** from the org's tier (Free = 14, paid = 36500 "forever"), looked up per org. ClickHouse applies a per-row TTL of `ingested_at + toIntervalDay(retention_days)` — this is the mechanism behind *free, unlimited observability with retention as the only paid axis*.
5. If `status == "running"` → publish a `"started"` message to `traces.persisted` and exit early (no DB write).
6. Insert row into ClickHouse `traces` table (with `is_root`, `agent_key`, `agent_kind`, `retention_days`). The insert also feeds the incremental **count** and (via `trace_costs`) **cost** roll-up materialized views.
7. Publish a `"persisted"` message to `traces.persisted`.
8. Estimate cost via `estimate_trace_cost()` (queries PostgreSQL `model_prices`).
9. Insert row into ClickHouse `trace_costs` table.
10. Publish an `"enriched"` (cost) message to `traces.persisted`.

---

## Cost estimation: `tracer/jobs/helper/cost_estimator.py`

Queries `model_prices` in PostgreSQL. Normalizes token field names across SDKs,
deducts provider prefix-cached tokens from billable input tokens, detects
long-context pricing tiers, and returns a per-trace cost breakdown (`input_cost`,
`cached_input_cost`, `output_cost`, `total_cost`, `long_context`). Returns `None`
when no price record exists.

## Root resolver: `tracer/jobs/helper/root_resolver.py`

Determines `root_trace_id` (outermost `@trace` span) so dashboards can
`GROUP BY root_trace_id`. Resolution order: no parent → self; LRU cache hit
(≤10 000 entries); ClickHouse lookup; fallback to `parent_id`.

---

## ClickHouse tables written

- `traces` — one row per event (the full `event` JSON is stored) plus the
  denormalized columns `is_root`, `agent_key`, `agent_kind`, and `retention_days`
  (per-row TTL).
- `trace_costs` — the per-trace cost breakdown from the cost estimator.

Both inserts drive **per-run roll-up** materialized views
(`trace_cost_rollup`, `trace_count_rollup`, and — fed by the evaluator/security
workers — `trace_quality_rollup`, `trace_security_rollup`), which precompute
cost / quality / security / span-count per `root_trace_id` so the Traces and
Agents tables read one aggregated row instead of summing children at read time.
The MVs only capture rows inserted after they exist; `db_queues/clickhouse/rollup_backfill.sql`
seeds history once.

See `docs/backend.md` for the full ClickHouse schema and the dashboard read
queries that join traces + costs + evaluations + security.
