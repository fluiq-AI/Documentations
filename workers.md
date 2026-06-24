# Fluiq Workers — Internal Architecture Reference

The workers project (`fluiq-workers/`) contains **three** independent async Kafka
consumers: **Tracer**, **Evaluator**, and **Security**. They have no HTTP surface —
all communication is through Kafka topics and writes to ClickHouse / PostgreSQL.

The Security worker is deployed separately from the Evaluator so the heavy
security dependencies (torch + spaCy/Presidio + sentence-transformers) don't bloat
the evaluator's memory footprint or block judge calls.

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
| `KAFKA_TRACE_TOPIC` (`traces`) | fluiq-api | Tracer | Raw trace events from the SDK |
| `KAFKA_EVAL_TOPIC` (`evaluations`) | fluiq-api | Evaluator | Auto-retrieval eval, explicit eval, SDK LLM eval, playground eval |
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

---

## Tracer Worker

### Entry point: `tracer/app.py`

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

### Ingest job: `tracer/jobs/ingest.py`

**Input message schema** (from `traces` topic)

```json
{
  "organization_id": "uuid",
  "api_key_prefix": "flq_abc123",
  "trace_id": "uuid",
  "event": {
    "trace_id": "uuid",
    "parent_id": "uuid | null",
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

**Processing steps**

1. Generate `trace_id` if missing.
2. Resolve `root_trace_id` via `RootTraceResolver` (LRU cache → ClickHouse → fallback to `parent_id`).
3. If `status == "running"` → publish a `"started"` message to `traces.persisted` and exit early (no DB write).
4. Insert row into ClickHouse `traces` table.
5. Publish a `"persisted"` message to `traces.persisted`.
6. Estimate cost via `estimate_trace_cost()` (queries PostgreSQL `model_prices`).
7. Insert row into ClickHouse `trace_costs` table.
8. Publish an `"enriched"` (cost) message to `traces.persisted`.

### Cost estimation: `tracer/jobs/helper/cost_estimator.py`

Queries `model_prices` in PostgreSQL. Normalizes token field names across SDKs,
deducts provider prefix-cached tokens from billable input tokens, detects
long-context pricing tiers, and returns a per-trace cost breakdown (`input_cost`,
`cached_input_cost`, `output_cost`, `total_cost`, `long_context`). Returns `None`
when no price record exists.

### Root resolver: `tracer/jobs/helper/root_resolver.py`

Determines `root_trace_id` (outermost `@trace` span) so dashboards can
`GROUP BY root_trace_id`. Resolution order: no parent → self; LRU cache hit
(≤10 000 entries); ClickHouse lookup; fallback to `parent_id`.

---

## Evaluator Worker

### Entry point: `evaluator/app.py`

Consumes `KAFKA_EVAL_TOPIC`. Pins all blocking work (judge calls) to a **single**
executor thread so torch/spaCy use one glibc malloc arena (the main RSS-growth
driver on small instances). On startup it also opens an optional Postgres pool
(`db/postgres.py`), seeds the judge-prompt defaults (fallback; see below), and
primes the prompt snapshot. Routes on `operation`:

| `operation` | Inferred when… | Handler |
|-------------|----------------|---------|
| `"evaluate"` | message has `evaluator` | `run_evaluation()` — explicit evaluator |
| `"sdk_llm"` | message has `eval_config` | `auto_llm_eval()` — score an SDK LLM trace (built-in metrics + any `eval_config.custom_judges`) |
| `"auto"` | (default, no operation) | `auto_evaluate_retrieval()` — vectorstore ContextPrecision |
| `"playground_eval"` | explicit | `playground_eval()` — dashboard playground, replies via Kafka |

**Environment variables**

| Variable | Default | Description |
|----------|---------|-------------|
| `KAFKA_EVAL_TOPIC` | `evaluations` | Input topic |
| `KAFKA_EVAL_GROUP_ID` | `evaluator-workers` | Consumer group |
| `KAFKA_SECURITY_REPLY_TOPIC` | — | (config only — reply topic shared with security) |
| `KAFKA_PLAYGROUND_REPLY_TOPIC` | — | Playground request-reply topic |
| `EVAL_JUDGE_PROVIDER` | `openai` | Judge LLM provider |
| `EVAL_JUDGE_MODEL` | provider default | Judge model name |
| `EVAL_JUDGE_THRESHOLD` | `0.7` | Pass/fail threshold |
| `EVAL_JUDGE_CACHE` / `_TTL` / `_MAX` | `1` / `3600` / `2048` | In-memory judge `PromptCache` settings |
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GEMINI_API_KEY` | — | Required for the matching judge provider |
| `CLICKHOUSE_*` incl. `CLICKHOUSE_EVALUATIONS_TABLE`, `CLICKHOUSE_SECURITY_TABLE` | — | ClickHouse connection + tables |
| `POSTGRES_DSN` | — | **Optional but required for warn-mode custom judges.** Postgres pool for admin-editable judge prompts *and* per-org client custom judges (resolves `custom_judges` slugs to `kind='judge'` rows in `prompts`). Unset → admin overrides are a no-op and custom judges are silently skipped; built-in metrics still run. |
| `EVAL_JUDGE_PROMPT_TTL` | `60` | Seconds a loaded judge-prompt snapshot is trusted before the next eval triggers a refresh |

### Evaluators — `evaluator/jobs/helper/`

Judge logic lives entirely in the worker (`judge.py` → `LLMJudge`, `InMemoryCache`,
`PromptCache`; `hallucination.py`; `ragas.py`). Supported names (via `_build_evaluator`):

| Name(s) | Class |
|---------|-------|
| `hallucination`, `hallucinationevaluator` | `HallucinationEvaluator` |
| `faithfulness`, `ragas.faithfulness` | `Faithfulness` |
| `answer_relevancy`, `relevance`, `ragas.answer_relevancy` | `AnswerRelevancy` |
| `context_precision`, `ragas.context_precision` | `ContextPrecision` |
| `context_recall`, `ragas.context_recall` | `ContextRecall` |
| `toxicity`, `ragas.toxicity` | `Toxicity` |
| `coherence`, `ragas.coherence` | `Coherence` |
| `ragas` | Full RAGAS pipeline (one result per metric) |

**Client custom judges** — `custom_judge.py` → `CustomJudgeEvaluator`. When an SDK
trace's `eval_config.custom_judges` (`{slug: threshold}`) is present, `auto_llm_eval()`
resolves each slug to the org's `kind='judge'` row via
`db/postgres.py:fetch_custom_judge(org_id, slug)`, renders the template
(`string.Template` `$question`/`$answer`/`$context`, JSON `{"score","reason"}` contract
auto-appended if absent), scores it through the same `LLMJudge` + cache, and persists
with `metric=<slug>`, `evaluator="fluiq.eval"`. Best-effort: missing template / no
`POSTGRES_DSN` / judge error skips that judge without breaking the batch. (This is
**per-org and client-authored** — distinct from the platform-global admin judge prompts
below.)

### Judge prompts (admin-editable) — `evaluator/jobs/helper/judge_prompts.py`

The LLM-as-Judge prompt text is **externalized** so it can be reworked from the
Admin console (Admin → Judge Prompts) without a worker redeploy. `judge_prompts.py`
holds the canonical defaults (`_PROMPTS`, 11 entries: `system`,
`hallucination_claims/verify/no_context`, `faithfulness_statements/verify`,
`answer_relevancy`, `toxicity`, `coherence`, `context_precision`,
`context_recall`). `judge.py`, `hallucination.py`, and `ragas.py` render via
`judge_prompts.render(name, **vars)` / `system_prompt()` instead of inline strings.

- **Syntax** — `string.Template` (`$var` / `${var}`); literal JSON braces need no
  escaping.
- **Flow** — `app.dispatch()` calls `judge_prompts.refresh()` before each message
  (TTL-gated by `EVAL_JUDGE_PROMPT_TTL`); it pulls overrides from Postgres
  `eval_judge_prompts` into an in-memory snapshot, and the synchronous `render()`
  reads that snapshot in the worker thread (no DB on the hot path).
- **Fail-open everywhere** — missing `POSTGRES_DSN`, unreachable Postgres, an
  override missing a required `$var`, or an unknown name all fall back to the
  built-in default; `render()` uses `safe_substitute` so a stray token never raises.
- **Seeding** — the **API** is the authoritative seeder on startup
  (`db_queues/postgresql/judge_prompt_defaults.py`, a byte-identical mirror of
  `_PROMPTS`); the worker also seeds with `ON CONFLICT DO NOTHING` as a fallback.
- **Custom prompts** — admins can add new prompts (stored in the same table); the
  worker loads them into the snapshot but they only run once evaluator code calls
  `render("their_name", …)`. See `docs/api.md` (Admin → Judge Prompts) and
  `docs/backend.md`.

### Job handlers — `evaluator/jobs/run.py`

- **`auto_evaluate_retrieval()`** — for vectorstore retrieval events
  (`type="vectorstore"` + retrieval `api`). Runs `ContextPrecision`, emits per-chunk
  `useful` flags. `evaluator="vectorstore.context_relevance"`.
- **`run_evaluation()`** — explicit evaluator run (`operation="evaluate"`); supports
  the full `ragas` pipeline.
- **`auto_llm_eval()`** — scores an SDK LLM trace against `eval_config.metrics`
  (default `["hallucination", "relevance"]`). `evaluator="fluiq.eval"`.
- **`playground_eval()`** — dashboard playground; runs the requested metrics and
  publishes a reply (`scores`, `results`, `passed`, `failures`) to
  `KAFKA_PLAYGROUND_REPLY_TOPIC` keyed by `correlation_id`. `evaluator="fluiq.playground"`.

Every result is persisted to ClickHouse via `insert_evaluation()` and fanned out as
an `"enriched"/"evaluation"` message to `traces.persisted`.

### ClickHouse `evaluations` table

```
organization_id   UUID
api_key_prefix    String
trace_id          String
root_trace_id     String
evaluator         String
metric            String
score             Float32
judge_model       String
details           String  -- JSON
```

---

## Security Worker

### Entry point: `security/app.py`

Consumes `KAFKA_SECURITY_TOPIC`. Like the evaluator, pins blocking scan work to a
single executor thread (torch/spaCy memory control). Routes on `operation`:

| `operation` | Source | Handler |
|-------------|--------|---------|
| `"sdk_security"` (default, no operation) | `/ingest` async fan-out | `auto_security_scan()` |
| `"security_check_sync"` | `/secure/check` (block mode) | `sync_security_check()` |
| `"response_gate_check"` | `/ingest` response gate | `sync_response_gate_check()` |

**Environment variables**

| Variable | Default | Description |
|----------|---------|-------------|
| `KAFKA_SECURITY_TOPIC` | — | Input topic (dedicated, not the eval topic) |
| `KAFKA_SECURITY_GROUP_ID` | — | Consumer group |
| `KAFKA_SECURITY_REPLY_TOPIC` | — | Synchronous request-reply topic for pre-call / response-gate |
| `KAFKA_TRACE_PERSISTED_TOPIC` | — | Async `enriched/security` fan-out (shared with tracer; routed by `kind`) |
| `CLICKHOUSE_SECURITY_TABLE` | — | Security results table |
| `CLICKHOUSE_TRACE_TABLE` | `traces` | Read-only access for indirect-injection / agent-chain context |

### Scanners — `security/jobs/helper/`

| Module | Detects |
|--------|---------|
| `pii.py` | PII via Presidio (redaction + entity list) |
| `injection.py` | Direct prompt-injection patterns |
| `jailbreak.py` | Jailbreak / role-play escapes (tiered strong/weak) |
| `skeleton_key.py` | Skeleton-key attack patterns (Microsoft KB) |
| `secrets.py` | Hardcoded credentials / high-entropy tokens |
| `semantic.py` | Cosine-similarity attack classifier (sentence-transformers) |
| `scanners.py` | Orchestrator — `scan()` (full post-call) and `check()` (pre-call patterns only) |

### Post-call scan: `auto_security_scan()`

Triggered by `/ingest` for every LLM trace carrying `_security_config`. Before
running the CPU-bound scan it concurrently reads four context sources from the
trace tree (all keyed on `root_trace_id`):

1. **Indirect sources** — sibling tool outputs, retrieved documents, tool inputs,
   tool names (`get_indirect_sources`).
2. **Session risk history** — prior `security_risk_score` values (`get_session_risk_scores`).
3. **Parent event kind** — to detect cross-agent (LLM→LLM) prompts (`get_parent_event_kind`).
4. **Agent-chain scores** — risk along the agent DAG ancestry (`get_agent_chain_scores`).

`scan()` then produces a `ScanResult` covering the full agentic-threat surface:

| Field group | Detection |
|-------------|-----------|
| PII | `prompt_redacted`, `response_redacted`, `pii_entities_prompt/response` (honors per-org `pii_ignore`) |
| Direct attacks | `injection_*`, `jailbreak_*`, `skeleton_key_*` |
| Secrets | `secrets_detected`, `secret_types` |
| Indirect injection | `indirect_injection_detected`, `indirect_injection_sources` (tool outputs + docs) |
| RAG poisoning (A.2) | `rag_poisoning_detected/sources/score` — semantic match on retrieved docs (threshold 0.55) |
| Tool-input exfiltration (B.2) | `tool_exfiltration_detected/types/sources` — PII/secrets sent into tool args |
| Tool allowlist (B.3) | `tool_policy_violation_detected/tool_policy_violations` — tool called outside the org `allowed_tools` |
| Cross-agent injection (C.1) | `cross_agent_injection_detected` — attack content arriving from another agent's output |
| Semantic | `semantic_attack_score` |
| Aggregate | `security_risk_level`, `security_risk_score`, `should_block` |

Two trajectory signals are derived in `run.py` and stored in the `extra` column:

- **Crescendo** — linear-regression slope over the session's per-turn risk scores
  (`crescendo_detected` when slope ≥ 0.3, ≥ 3 turns, current score ≥ 0.25).
- **Trust-boundary escalation (C.2)** — the same slope keyed on the agent DAG
  ancestry rather than flat session time; only meaningful for agent-sourced prompts.

When `response_gated: true` (the response gate already scanned the output
synchronously in `/ingest`), the response text is skipped to avoid double-billing
PII/secret detection. Results are written to ClickHouse `CLICKHOUSE_SECURITY_TABLE`
and an `"enriched"/"security"` message is published to `traces.persisted`.

### Pre-call check: `sync_security_check()`

Called from `/secure/check` (block mode) via Kafka request-reply. Runs a full scan
on the prompt and applies the forwarded `policy`:

- `block_threshold` — `medium` lowers the block bar (otherwise only `high` blocks).
- `block_categories` — when set, only listed attack types contribute to a block.
- `pii_ignore` — PII entity types to suppress.

Returns `{ allow, block_reason, risk_level, attack_types }` to
`KAFKA_SECURITY_REPLY_TOPIC` keyed by `correlation_id`. Fails open on any error.

### Response gate: `sync_response_gate_check()`

Called from `/ingest` when the org's active guardrail policy has
`scan_responses=True`. Scans only the **response** text for PII and secrets
(attack patterns are prompt-side and already caught pre-call). Blocks when an
attack type is present and `security_risk_score ≥ 0.5`. Returns
`{ response_blocked, risk_level, attack_types, block_reason }`. Fails open.

### ClickHouse security table (column order, by-name insert)

```
organization_id, api_key_prefix, trace_id, root_trace_id, mode,
prompt_redacted, response_redacted,
pii_entities_prompt, pii_entities_response,
injection_detected, injection_patterns,
jailbreak_detected, jailbreak_patterns,
skeleton_key_detected, skeleton_key_patterns,
secrets_detected, secret_types,
indirect_injection_detected, indirect_injection_sources,
rag_poisoning_detected, rag_poisoning_sources, rag_poisoning_score,
tool_exfiltration_detected, tool_exfiltration_types, tool_exfiltration_sources,
tool_policy_violation_detected, tool_policy_violations,
cross_agent_injection_detected,
semantic_attack_score, security_risk_level, security_risk_score,
should_block, scan_latency, extra
```

`extra` (JSON) carries the derived trajectory signals: `crescendo_detected`,
`crescendo_score`, `session_turns`, `trust_boundary_escalation`,
`escalation_score`, `agent_chain_depth`.

> **Deploy ordering:** because the insert is column-by-name, the PostgreSQL
> migration and the ClickHouse `ALTER TABLE … ADD COLUMN` for the agentic-threat
> columns must be applied **before** the security worker is deployed.

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
  ├─ Kafka: traces ──────────────► Tracer Worker
  │                                   ├─ ClickHouse: INSERT traces
  │                                   ├─ traces.persisted "persisted"
  │                                   ├─ PostgreSQL: SELECT model_prices
  │                                   ├─ ClickHouse: INSERT trace_costs
  │                                   └─ traces.persisted "enriched/cost"
  │
  ├─ Kafka: evaluations ─────────► Evaluator Worker
  │                                   ├─ judge eval (auto / sdk_llm / explicit / playground)
  │                                   ├─ ClickHouse: INSERT evaluations
  │                                   └─ traces.persisted "enriched/evaluation"
  │                                   └─ (playground) KAFKA_PLAYGROUND_REPLY_TOPIC
  │
  └─ Kafka: KAFKA_SECURITY_TOPIC ─► Security Worker (async, "sdk_security")
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
