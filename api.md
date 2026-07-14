# Fluiq API Reference

Base URL: `https://api.getfluiq.com` (production) · `http://localhost:8000` (local dev)

**Authentication**
- **Dashboard endpoints** — Bearer JWT in `Authorization: Bearer <access_token>`.
- **SDK endpoints** — Fluiq API key, sent as `Authorization: Bearer flq_...` (preferred) or in the request body as `api_key`.

Routers are mounted under `/api/v1` (most), `/auth`, `/api-keys`, and `/admin`.

---

## Authentication

### POST `/auth/register`

Create a new user account and organization. Returns a full session including an API key.

**Request**
```json
{ "name": "Alice", "email": "alice@example.com", "password": "secret123" }
```
Password: minimum 8 characters, at least one letter and one digit.

**Response** `201 Created`
```json
{
  "user": { "user_id": "uuid", "email": "alice@example.com", "name": "Alice", "user_type": "Free", "org_id": "uuid", "created_at": "..." },
  "organization": { "org_id": "uuid", "name": "Alice", "api_keys": [{ "key_id": "uuid", "name": "default", "prefix": "flq_..." }] },
  "access_token": "eyJ...", "token_type": "bearer", "expires_in": 3600,
  "refresh_token": "eyJ...", "refresh_expires_in": 604800
}
```

**Errors** · `400` email already registered or password too weak

---

### POST `/auth/login`

**Request** `{ "email": "alice@example.com", "password": "secret123" }`

**Response** `200 OK` — same shape as `/auth/register` · **Errors** · `401` invalid credentials

---

### POST `/auth/refresh`

Exchange a refresh token for a new token pair. The old refresh token is revoked.

**Request** `{ "refresh_token": "eyJ..." }`

**Response** `200 OK` — `{ access_token, token_type, expires_in, refresh_token, refresh_expires_in }`

**Errors** · `401` token expired, invalid, or already revoked

---

### POST `/auth/logout`

**Request** `{ "refresh_token": "eyJ..." }` · **Response** `200 OK` `{ "ok": true }`

---

### OAuth & Password Reset

| Endpoint | Description |
|----------|-------------|
| `GET /auth/google` · `GET /auth/google/callback` | Google social login (returns a session) |
| `GET /auth/github` · `GET /auth/github/callback` | GitHub social login |
| `POST /auth/password/forgot` | Send an OTP to the account email (`{ email }`) |
| `POST /auth/password/reset` | Reset with OTP (`{ email, otp, new_password }`) |

OAuth callbacks redirect back to `FRONTEND_BASE_URL` with the session. OTPs are length/expiry-bounded by `PASSWORD_RESET_OTP_LENGTH` / `PASSWORD_RESET_EXPIRE_MINUTES`.

---

## API Keys

### GET `/api-keys`
List all API keys for the authenticated organization. **Response** `OrganizationModel` (with `api_keys`).

### POST `/api-keys`
Create a key. The full plaintext key is returned **once**.
**Request** `{ "name": "production" }` → **Response** `201` `{ key_id, name, prefix, created_at, key }` · **Errors** · `400` key limit reached.

### DELETE `/api-keys/{key_id}`
**Response** `204` · **Errors** · `404` not found / not owned.

---

## Traces

### POST `/api/v1/ingest`

Primary SDK ingestion endpoint. API key auth.

**Request**
```json
{
  "api_key": "flq_abc123...",
  "event": {
    "trace_id": "uuid", "parent_id": null,
    "type": "llm", "integration": "OpenAI", "model": "gpt-4o", "status": "complete",
    "tokens": { "prompt": 100, "completion": 50, "total": 150 },
    "prompt_cached_tokens": 20, "cache_hit": false,
    "_eval": true,
    "_eval_config": { "metrics": ["hallucination"], "judge_model": "claude-haiku-4-5-20251001", "thresholds": {}, "custom_judges": { "refund-policy": 0.9 } },
    "_security_config": { "mode": "block", "guardrail": "default" }
  }
}
```

`trace_id` is auto-generated if omitted. `status:"running"` events stream to SSE but are not persisted. Keys prefixed with `_` (`_eval`, `_eval_config`, `_security_config`, `_cache_hit`) are stripped before storage and drive the eval/security fan-out.

**Evaluation is opt-in.** The API only publishes an eval job when the SDK opted in — `eval_enabled = _eval or (_eval_config is not None)`. A trace ingested by `instrument()` alone (no `fluiq.eval()`) is traced but **not** evaluated; the old ambient auto-eval sample rate is gone. Retrieval/LLM auto-scoring likewise only fires when eval is enabled.

**Response** `200 OK`
```json
{ "ok": true, "trace_id": "uuid", "eval_skipped": false,
  "response_blocked": false, "block_reason": null, "risk_level": null, "attack_types": [] }
```

When the org's active guardrail policy has `scan_responses: true`, `/ingest` runs the **response gate** synchronously (round-trips the security worker) and may return `response_blocked: true`.

**Errors** · `401` invalid API key · `402` trace quota exceeded · `413` event exceeds the 10 MB Kafka message ceiling

### POST `/api/v1/ingest/otel`

Ingest **OpenInference / OTLP** spans from an external observability platform, so
their traces flow through the same pipeline as native SDK traces (ClickHouse
persist + per-call eval + optional security scan; run the agentic evaluator on
the root afterwards). API-key auth; trace-quota gated. Each span → a Fluiq event
(`routes/otel/convert.py`).

Accepts one of: OTLP-JSON (`resourceSpans`), flat `spans` (Phoenix-style), or
pre-mapped `events` (used by the pull connectors).

**Request** `{ "api_key": "flq_...", "source": "phoenix|langfuse|langsmith|braintrust",
"resourceSpans": [...] | "spans": [...] | "events": [...],
"eval_config"?: {...}, "security_config"?: {...} }`
**Response** `200 OK` `{ "ok": true, "source": "...", "spans_received": N, "events_ingested": M, "root_trace_ids": [...] }`

**Integration paths** — **Phoenix / Langfuse**: point their OTel exporter at this
endpoint (OTLP push). **LangSmith / Braintrust**: run the pull connectors
(`python -m connectors.langsmith` / `connectors.braintrust`), which fetch from
the platform API, map to events, and POST here. **Errors** · `401` · `402` · `422` no convertible spans.

---

### GET `/api/v1/traces`

List traces for the org (JWT auth).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `key_id` | UUID | — | Filter by API key |
| `agent_key` | string | — | Filter by agent key |
| `agent_kind` | string | — | Filter by agent kind |
| `root_trace_id` | UUID | — | Return a single trace tree |
| `roots_only` | bool | false | Only root spans |
| `status` | string | — | `success` / `error` / `running` / `blocked` |
| `security` | string | — | Filter by security verdict |
| `integration` | string | — | e.g. `OpenAI`, `LangChain` |
| `quality` | string | — | Filter by eval quality bucket |
| `sort` | string | — | Sort order |
| `limit` | int (1–1000) | 100 | Page size |
| `offset` | int (≥0) | 0 | Pagination offset |

**Response** `200 OK`
```json
{
  "traces": [{
    "api_key_prefix": "flq_abc123",
    "event": { "trace_id": "uuid", "model": "gpt-4o", "type": "llm", "cache_hit": false },
    "ingested_at": "2025-01-01T00:00:00Z",
    "cost": 0.0023, "currency": "USD",
    "evaluations": [{ "metric": "hallucination", "score": 0.92, "evaluator": "fluiq.eval", "judge_model": "claude-haiku-4-5-20251001" }],
    "security": { "risk_level": "clean", "attack_types": [] }
  }],
  "limit": 100, "offset": 0
}
```

The Traces/Agents tables read per-run headline numbers (cost, quality, security,
span count) from precomputed roll-up tables keyed by `root_trace_id`, so a root
row shows the whole run's rolled-up quality **without** expanding it or summing
children at read time.

---

### POST `/api/v1/traces/rollups`

Batch-fetch the precomputed per-run roll-ups for a set of root trace ids (JWT
auth). Body: `{ "root_trace_ids": ["uuid", …] }`. Returns, per root, the merged
`run_cost` / `run_tokens`, `quality_min` / `quality_avg` / `quality_count`,
`security` (max risk, should-block, detections), and `span_count`. Backed by
ClickHouse `AggregatingMergeTree` roll-up tables fed incrementally by
materialized views as children finish; the dashboard calls this once per page.

---

### GET `/api/v1/traces/spending`

Cost rollups for the org over a time window (JWT auth). Returns per-period / per-model spend used by the Overview spending charts.

---

### GET `/api/v1/traces/stream`

Server-Sent Events stream of trace activity (JWT auth). Query: `key_id` (optional).

| Event | Sent when |
|-------|-----------|
| `ready` | Connection established |
| `ping` | Idle keepalive |
| `trace` | New trace persisted |
| `trace.started` | In-progress trace reported |
| `trace.enriched` | Cost / evaluation / security data available |

---

## Quota

### GET `/api/v1/quota`

Plan tier and quota usage (JWT auth).

**Response** `200 OK`
```json
{ "tier": "Team", "traces": { "used": 12340, "limit": null }, "evaluations": { "used": 870, "limit": 10000 } }
```
`null` limit = unlimited.

| Tier | Traces | Retention | Evaluations / month |
|------|--------|-----------|---------------------|
| Free | Unlimited | 14 days (rolling) | 1,000 |
| Team | Unlimited | Forever | 10,000 |
| Growth | Unlimited | Forever | 100,000 |
| Enterprise | Unlimited | Forever | Unlimited |

**Observability is free and unlimited on every tier.** There is no trace/span/agent
cap. The only paid axis is **retention**: Free keeps a rolling 14-day window, paid
keeps traces forever. Retention is enforced by a per-row ClickHouse TTL — the tracer
stamps `retention_days` on each row from the org's tier at ingest (see `docs/tracer.md`).

### POST `/api/v1/billing/trial`

Starts a no-card **5-day trial** of a paid tier (Team or Growth) for the org (JWT
auth). Sets `users.trial_ends_at`; the tier is resolved lazily and auto-expires
back to Free when the window passes. Body: `{ "tier": "Team" | "Growth" }`. One
trial per org (`users.trial_used`).

---

## Agents

### GET `/api/v1/agents/summary`

Cost / token / latency rollup grouped by agent (JWT auth), read from the
precomputed per-run roll-up tables. Query: `limit` + `offset` (windowed
pagination — the dashboard pages 50 at a time). The roll-up groups by the
denormalized `agent_key` / `agent_kind` / `integration`; a LangGraph run appears
as `LangGraph(node_a, node_b, …)`.

```json
{ "agents": [{
  "agent_key": "my_rag_chain", "agent_kind": "chain", "integration": "LangChain",
  "runs": 412, "total_cost": 1.84, "avg_cost_per_run": 0.00447,
  "total_tokens": 920400, "avg_latency": 1.23, "last_run": "2025-01-01T12:00:00Z"
}], "limit": 100 }
```

An "agent" is a trace **root** keyed by its `@trace` function name (or chain /
`langgraph_node` / `provider:model`). The root definition matches `/traces`
`roots_only`: a span counts when `trace_id == root_trace_id` **or** its
`root_trace_id` points at a never-persisted (orphan) parent — so a named
`@trace(name=…)` span that shows in Traces also rolls up here.

---

## Optimize

### GET `/api/v1/optimize/cache-stats`
Redis cache hit/miss aggregated from SDK traces (JWT auth). Query: `window_hours` (1–720, default 24).
```json
{ "window_hours": 24, "hits": 842, "misses": 301, "calls": 1143, "hit_rate": 0.737,
  "per_kind": [ { "kind": "mcp_call", "hits": 210, "misses": 40, "calls": 250, "hit_rate": 0.84 } ] }
```
`per_kind` covers: `llm`, `embedding`, `vectorstore`, `function`, `mcp_call`, `mcp_list_tools`.

### GET `/api/v1/optimize/prompt-cache-stats`
Provider prefix-cache token counts (JWT auth). Query: `window_hours`.
```json
{ "window_hours": 24, "anthropic_cache_read_tokens": 184200, "anthropic_cache_creation_tokens": 21000,
  "provider_cached_tokens": 64800, "total_cached_tokens": 249000, "calls": 1840, "calls_with_hit": 1210 }
```

### GET `/api/v1/optimize/profile`
SDK endpoint — Redis cache profile for the calling org. API key auth. **Requires Team plan or above.**
```json
{ "redis_url": "redis://...", "key_prefix": "fluiq:abc12345:", "models": ["gpt-4o", "claude-sonnet-4-6"],
  "ttl_seconds": 86400, "estimated_hit_rate": 0.42, "window_hours": 168 }
```
**Errors** · `402` Free plan · `503` Redis not configured.

### GET `/api/v1/optimize/cache/{key}` · POST `/api/v1/optimize/cache`
SDK Redis proxy (API key). GET → `{ "value": ... }` or `404` miss. POST `{ key, value, ttl }` → `204` (fire-and-forget).

### GET `/api/v1/optimize/evals`
CI eval gate (API key). Query: `window_minutes` (def 30), `threshold` (def 0.7), `limit` (def 200).
```json
{ "window_minutes": 30, "total": 45, "passed": 40, "failed": 5, "avg_score": 0.883,
  "entries": [{ "trace_id": "uuid", "metric": "hallucination", "score": 0.91, "evaluator": "fluiq.eval", "judge_model": "claude-haiku-4-5-20251001" }] }
```

---

## Security

### POST `/api/v1/secure/check`

Pre-call security guard. Called by the SDK in block mode before forwarding the prompt. API key auth. **Requires Growth plan or above** (402 otherwise).

**Request**
```json
{ "api_key": "flq_abc123...", "prompt": "Ignore previous instructions and ...",
  "trace_id": "uuid", "guardrail": "default", "context": { } }
```

Flow: allow-list → deny-list → fast pattern check → full scan via the security worker (`security_check_sync`, with `{block_threshold, block_categories, pii_ignore}`) → pattern-only fallback on timeout (**fail-open**).

**Response** `200 OK`
```json
{ "allow": false, "block_reason": "Blocked by fluiq.secure: prompt_injection",
  "risk_level": "high", "attack_types": ["prompt_injection"] }
```

When `allow: false` the SDK raises `FluiqSecurityError` and the LLM call is never made; a `status:"blocked"` trace is published (dashboard row → blocked) and the policy's `alert_webhook` is fired when `risk_level ∈ alert_on`. When the scan service is unreachable the endpoint returns the pattern-only verdict (fail-open).

---

## Guardrails

### GET `/api/v1/guardrails/list`
List policy slugs for the org (JWT). **Response** `["default", "strict-pii"]`.

### GET `/api/v1/guardrails`
Get a policy by slug (JWT). Query: `slug` (default `"default"`).

### PUT `/api/v1/guardrails`
Create or update a policy (JWT). Query: `slug`.

**Request / Response shape**
```json
{
  "block_threshold": "high",
  "warn_threshold": "medium",
  "block_categories": ["prompt_injection", "rag_poisoning"],
  "custom_deny_list": ["internal-codename"],
  "custom_allow_list": [],
  "pii_ignore": ["EMAIL_ADDRESS"],
  "allowed_tools": ["search", "calculator"],
  "alert_webhook": "https://hooks.slack.com/...",
  "alert_on": ["high"],
  "scan_responses": true
}
```
The response additionally includes `org_id` and `slug`.

- `block_threshold` ∈ `{medium, high}`; `warn_threshold`/`alert_on` values ∈ `{low, medium, high}`.
- `block_categories` (empty = block on any) must be a subset of: `prompt_injection, jailbreak, skeleton_key, semantic_attack, pii_detected, secrets_detected, indirect_injection, rag_poisoning, tool_exfiltration, tool_policy_violation, cross_agent_injection`.
- `pii_ignore` must be a subset of: `US_SSN, CREDIT_CARD, IBAN_CODE, CRYPTO, US_PASSPORT, EMAIL_ADDRESS, PHONE_NUMBER, PERSON, LOCATION, IP_ADDRESS`.

**Errors** · `422` unknown category / pii entity / invalid `alert_on`.

### DELETE `/api/v1/guardrails`
Delete a non-default policy (JWT). Query: `slug` (required). **Errors** · `400` cannot delete `default`.

---

## Alerts

### GET `/api/v1/alerts` · PUT `/api/v1/alerts`
Get / save the org's Slack alert settings (JWT). Eval alerts require **Team+**; security alerts require **Growth+**.

### POST `/api/v1/alerts/test`
Fire a test Slack message to the configured webhook (JWT).

---

## Prompts

### GET `/api/v1/prompts`
List templates (JWT). Each prompt serializes its `environments` dict (`development`/`staging`/`production`).

### POST `/api/v1/prompts`
Create a template (JWT). `{ slug, name, template, variables, kind? }` → `201`. `kind` is `"completion"` (default) or `"judge"`; a `judge` template must reference the `$answer` placeholder (`422` otherwise) and is referenced by slug from `fluiq.eval(custom_judges=...)`.

### PATCH `/api/v1/prompts/{prompt_id}`
Update (JWT) — creates a new version.

### DELETE `/api/v1/prompts/{prompt_id}`
`204`.

### POST `/api/v1/prompts/{prompt_id}/environments/{env}`
Promote / unpromote the current version to an environment (`development`/`staging`/`production`).

### POST `/api/v1/prompts/{prompt_id}/deploy`
Legacy deploy toggle — `{ "deploy": true|false }`.

### GET `/api/v1/prompts/{prompt_id}/versions` · POST `.../versions/{version}/restore`
List version history / restore a previous version (JWT).

### GET `/api/v1/prompts/fetch/{slug}`
SDK read endpoint (API key). Query: `env` (default `production`). Used by `fluiq.fetch_prompt()`.
```json
{ "slug": "support-reply", "name": "Support Reply", "template": "You are ...",
  "model": null, "variables": ["ticket_body"], "version": 3, "environment": "production" }
```
**Errors** · `404` not deployed to env · `401` invalid key.

---

## Datasets

Curated golden sets built from real traces. A trace-backed example pins the run's
**whole trajectory** (all spans: LLM calls, tool/MCP calls, the multi-agent DAG,
media) into a no-TTL ClickHouse store (`dataset_trajectory_spans`, media offloaded
to S3), so agentic eval / security can run over it offline, retention-independent.

| Endpoint | Description |
|----------|-------------|
| `GET /api/v1/datasets` | List datasets |
| `POST /api/v1/datasets` | Create `{ name, description? }` |
| `DELETE /api/v1/datasets/{id}` | Delete (also drops runs, agent links) |
| `GET /api/v1/datasets/{id}/examples` | List examples — windowed pagination (`limit` ≤200 default 50, `offset`); each row auto-enriched with eval/security/cost by `source_trace_id` |
| `POST /api/v1/datasets/{id}/examples` | Add `{ input, expected_output?, metadata }`. Pass `metadata.source_trace_id` (the run's **root** trace id) and the server snapshots that run's full trajectory + derives an IO summary. Empty-input trace-backed examples are accepted (e.g. a CrewAI crew root). |
| `DELETE /api/v1/datasets/{id}/examples/{example_id}` | Remove example |
| `GET /api/v1/datasets/{id}/examples/{example_id}/trajectory` | The pinned trajectory as a compact, display-ready summary (DFS-ordered steps with type/agent/model/tool-calls/MCP/media + rollup stats) |
| `POST /api/v1/datasets/{id}/runs` | Launch a batch run `{ kind: "agentic" \| "security", depth? }` over every example |
| `GET /api/v1/datasets/{id}/runs` · `GET /api/v1/datasets/runs/{run_id}` | List runs · fetch a run's report |
| `POST /api/v1/datasets/{id}/agents` · `GET`/`DELETE .../agents` | **Connect Agents**: link a traced agent → imports all its runs to date (deduped, full trajectory pinned) and auto-appends future runs |

All JWT-authed.

---

## Evaluate

### POST `/api/v1/evaluate`

Run LLM-as-judge evaluation for a prompt/response pair (API key auth). Called by the SDK after each LLM call when `fluiq.eval()` is active (block mode). **No tier gate.** Supported metrics: `hallucination, faithfulness, relevance, toxicity, coherence, completeness`.

**Request**
```json
{ "api_key": "flq_...", "trace_id": "uuid", "model": "gpt-4o",
  "prompt": "What is RAG?", "response": "RAG combines retrieval with generation...",
  "context": "RAG stands for Retrieval-Augmented Generation...",
  "metrics": ["hallucination", "relevance"],
  "judge_model": "claude-haiku-4-5-20251001",
  "thresholds": { "hallucination": 0.8 },
  "custom_judges": { "refund-policy": 0.9 } }
```
`custom_judges` (`{slug: threshold}`) resolves each slug to the org's `kind='judge'` prompt and runs it alongside the built-in metrics; results are keyed by slug. Unknown slugs are skipped.

**Response** `200 OK`
```json
{ "trace_id": "uuid",
  "scores": { "hallucination": 0.91, "relevance": 0.86, "refund-policy": 0.94 },
  "results": [ { "metric": "hallucination", "score": 0.91, "reason": "...", "passed": true } ],
  "passed": true, "failures": [] }
```
Results are stored in ClickHouse and fanned out to SSE (one `enriched` message per metric). **Errors** · `422` no valid metrics *and* no custom judges.

### POST `/api/v1/evaluate/playground`
JWT-authed playground. Publishes a `playground_eval` job to the eval worker and awaits its reply (judge logic lives only in the worker). Same response shape. **Errors** · `504` worker timeout.

### POST `/api/v1/evaluate/compare`
JWT-authed. Runs the same prompt against multiple Claude models in parallel (`claude-haiku-4-5-20251001`, `claude-sonnet-4-6`, `claude-opus-4-7`), returning per-model output, latency, tokens, and estimated cost.

### POST `/api/v1/evaluate/agentic`
JWT-authed. Triggers **agentic (multi-layer) evaluation** for a whole run — fired by the **Run Agentic Eval** button in the trace drawer (root traces only). Fetches every span sharing the `root_trace_id`, publishes one `agent_eval` job to the eval worker (async), and returns immediately; layered results (deterministic + tool-selection + trajectory [+ panel]) stream back over the SSE `trace.enriched` channel. Quota-gated.

**Request** `{ "trace_id": "uuid", "root_trace_id": "uuid?", "depth": "fast|standard|deep?" }`
**Response** `200 OK` `{ "ok": true, "trace_id": "uuid", "status": "queued", "events": 7 }` · `status:"skipped"` when eval quota is exceeded. **Errors** · `404` no trace events · `422` invalid id.

### GET `/api/v1/evaluate/agentic-summary?window_hours=24`
JWT-authed. Aggregates agentic-eval health for the Overview tile over a window (1..720h): run count, run pass-rate, avg run score, and per-layer average scores.

**Response** `200 OK`
```json
{ "window_hours": 24, "runs": 42, "pass_rate": 0.83, "avg_run_score": 0.79,
  "layers": [ { "layer": "deterministic", "score": 0.88, "count": 42 },
              { "layer": "tool_selection", "score": 0.81, "count": 42 },
              { "layer": "trajectory", "score": 0.74, "count": 40 } ] }
```

---

## Models (LLM Cost Calculator)

### GET `/api/v1/models`
Public. Text-modality models with input + output prices from `model_prices`.
```json
{ "models": [ { "id": 1, "provider": "Anthropic", "model": "claude-sonnet-4-6",
  "input_per_million": 3.0, "output_per_million": 15.0, "cached_input_per_million": 0.3 } ], "count": 1 }
```

### POST `/api/v1/models/request`
Public. Visitor asks to add a model `{ provider?, model, email?, note? }` → stored + team-emailed. `{ "ok": true }`.

### POST `/api/v1/models/report`
Public. Report a price change `{ model_price_id?, provider?, model, reported_input?, reported_output?, source_url?, email?, note? }` → stored + team-emailed. Both fail soft (a mail outage never fails the request).

---

## Blog

### Public
| Endpoint | Description |
|----------|-------------|
| `GET /api/v1/blog/posts` | Published posts (list) |
| `GET /api/v1/blog/posts/{slug}` | Single published post |
| `GET /api/v1/blog/slugs` | All published slugs (for prerender/sitemap) |
| `GET /api/v1/blog/media/{media_id}` | `307` redirect to a short-lived presigned S3 URL |

### Admin (`require_admin`)
List / create / get / patch / delete posts, `publish`, and media upload (≤5 MB, image types). Publishing POSTs the Render deploy hook to trigger a static rebuild + prerender.

---

## Contact

### POST `/api/v1/contact`
Public contact form → emails the team. `{ name, email, message }` → `{ "ok": true }`.

---

## Audit

### GET `/api/v1/audit`
Admin JWT. Returns HMAC-signed request audit-log entries from ClickHouse `audit_log`.

---

## Admin (`/admin`, `require_admin`)

Internal operator endpoints, gated to `user_type == "Admin"`. Besides platform
stats, user/org listing, plan changes, blog CMS, and the infrastructure console
(read-only SQL, worker status/logs, secrets, and scaling), the admin surface
includes evaluation-allowance and judge-prompt management.

### Evaluations — per-org allowance

Admins can grant or deduct evaluations on top of an org's tier quota
(`organizations.eval_quota_bonus`). The effective monthly eval cap becomes
`max(0, tier_eval_quota + bonus)`; unlimited tiers stay unlimited.

| Endpoint | Description |
|----------|-------------|
| `GET /admin/users/{user_id}/evaluations` | Resolve a user → org tier, eval usage this month, base quota, admin bonus, effective limit |
| `POST /admin/users/{user_id}/evaluations` | Adjust the org's bonus by `{ "delta": <non-zero int> }` (may be negative); invalidates the quota cache so it applies immediately |

```json
// GET / POST response
{ "user_id": "uuid", "email": "...", "name": "...", "user_type": "Team",
  "org_id": "uuid", "org_name": "Acme", "tier": "Team",
  "eval_used": 870, "eval_bonus": 5000, "base_eval_quota": 10000, "eval_limit": 15000 }
```

### Judge Prompts — LLM-as-Judge templates

Platform-global prompts the evaluator worker renders (see `docs/evaluator.md`).
Stored in `eval_judge_prompts`; seeded on API startup. Templates use
`string.Template` `$var` placeholders; an edit/create is rejected if it drops a
required placeholder.

| Endpoint | Description |
|----------|-------------|
| `GET /admin/judge-prompts` | List all prompts (`template`, `default_template`, `description`, `required_vars`, `is_overridden`, `version`) |
| `POST /admin/judge-prompts` | Create a custom prompt `{ name, template, description?, required_vars[] }`; name must match `^[a-z][a-z0-9_]{1,63}$` (`201`; `409` if it exists) |
| `GET /admin/judge-prompts/{name}` | Get one |
| `PUT /admin/judge-prompts/{name}` | Update `{ template }` (validates required placeholders; bumps version) |
| `POST /admin/judge-prompts/{name}/reset` | Reset to the built-in `default_template` |
| `GET /admin/judge-prompts/{name}/versions` | Version history |
| `POST /admin/judge-prompts/{name}/restore/{version}` | Restore a prior version (saved as a new version) |

A custom (admin-created) prompt is stored for the future — it runs only once
evaluator code references it by name via `render("name", …)`.

### Infrastructure — workers, logs, secrets, scaling

Live AWS operator surface for the API + workers. The four manageable services are
`fluiq-api`, `fluiq-tracer`, `fluiq-evaluator`, `fluiq-security`. Reads need scoped
AWS permissions on the API task role; the **mutating** endpoints (✎) additionally
need `ssm:Put/DeleteParameter`, `ecs:RegisterTaskDefinition` / `UpdateService`, and
`iam:PassRole` — every mutation emits a structured audit log line
(`infra-audit admin=… action=… …`). Secret **values are never returned**.

| Endpoint | Description |
|----------|-------------|
| `POST /admin/infra/query` | Read-only SQL (`SELECT/WITH/SHOW/DESCRIBE/EXPLAIN`, single statement) against Postgres or ClickHouse; capped at 1000 rows |
| `GET /admin/infra/workers` | ECS service status (`running/desired/pending`, task def, rollout) |
| `GET /admin/infra/workers/{service}/logs` | CloudWatch logs. **No params** → tail the latest stream. Any of `q` (case-sensitive substring), `start`, `end` (epoch ms) → `filter_log_events` across **all** streams, ascending by timestamp. `limit` 1–2000 (default 200; 500 on tail) |
| `GET /admin/infra/secrets` | List SSM parameter names/metadata under `/fluiq/prod/` (never values) |
| ✎ `PUT /admin/infra/secrets` | Create/update an SSM parameter `{ name, value, type?, description? }` (`type` `SecureString`\|`String`; name must match `/fluiq/prod/<NAME>`) |
| ✎ `DELETE /admin/infra/secrets?name=` | Delete an SSM parameter (`404` if not found) |
| `GET /admin/infra/workers/{service}/secrets` | List the worker's task-def secret bindings (`name` + `value_from` ARN) |
| ✎ `POST /admin/infra/workers/{service}/secrets` | Wire an SSM param into the worker as an env var `{ env_name, ssm_name, value?, type? }`. With `value` the SSM param is created/updated first. Registers a new task-def revision and updates the service — **redeploys the worker** |
| ✎ `DELETE /admin/infra/workers/{service}/secrets/{env_name}` | Remove a binding (new revision + redeploy). The SSM parameter itself is left intact |
| ✎ `POST /admin/infra/workers/{service}/scale` | Set desired task count `{ desired }` (0–10; 0 stops the service). Returns live `{ desired, running, pending }` |

Mutations fail with `502` (`detail` carries the AWS error) when the IAM policy
is not yet attached; unknown `service` → `404`, out-of-range/invalid input → `400`.

---

## Error Format

```json
{ "detail": "Human-readable error message" }
```

---

## Token Details

**Access token** — HS256 JWT (`JWT_EXPIRE_MINUTES`). Pass as `Authorization: Bearer <token>`.
**Refresh token** — HS256 JWT (`JWT_REFRESH_EXPIRE_DAYS`). Rotate via `/auth/refresh`; revoked on logout.
