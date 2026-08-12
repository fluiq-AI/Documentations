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

**Identifier normalization.** `trace_id`, `root_trace_id`, and `parent_id` are coerced to valid UUIDs at ingest (`shared/ids.py::coerce_trace_uuid`): a value that is already a UUID passes through, any other non-empty string is mapped to a deterministic UUIDv5 (same string → same UUID, so a non-UUID trace tree stays internally linked). ClickHouse stores these as UUID columns, so without this a non-UUID id would crash the tracer insert and drop the trace; the response returns the effective (possibly remapped) `trace_id`. The tracer applies the same coercion defensively.

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
{ "tier": "Team", "traces": { "used": 12340, "limit": null }, "evaluations": { "used": 870, "limit": 10000 },
  "security_scans": { "used": 41200, "limit": 500000 }, "retention_days": null, "trial_ends_at": null }
```
`null` limit = unlimited.

| Tier | Traces | Retention | Evaluations / month | Security scans / month |
|------|--------|-----------|---------------------|------------------------|
| Free | Unlimited | 14 days (rolling) | 100 | 1,000 |
| Starter | Unlimited | Forever | 2,000 | 50,000 |
| Team | Unlimited | Forever | 10,000 | 500,000 |
| Growth | Unlimited | Forever | 50,000 | 2,000,000 |
| Enterprise | Unlimited | Forever | Unlimited | Unlimited |

Security scans are metered separately from evaluations because they cost something
different to run: scanning is regex plus spaCy NER with no LLM call, roughly
$0.0002 a scan against $0.03 or more for an agentic evaluation. See
`TIER_SECURITY_QUOTAS` in `shared/quotas.py`.

**Observability is free and unlimited on every tier.** There is no trace/span/agent
cap. The only paid axis is **retention**: Free keeps a rolling 14-day window, paid
keeps traces forever. Retention is enforced by a per-row ClickHouse TTL — the tracer
stamps `retention_days` on each row from the org's tier at ingest (see `docs/tracer.md`).

### POST `/api/v1/billing/trial`

Starts a no-card **5-day trial** of a paid tier (Starter, Team, or Growth) for the org (JWT
auth). Sets `users.trial_ends_at`; the tier is resolved lazily and auto-expires
back to Free when the window passes. Body: `{ "tier": "Team" | "Growth" }`. One
trial per org (`users.trial_used`).

---

### GET `/api/v1/quota/judge-usage`

Judge tokens spent by this org, in total and split by evaluator (JWT auth).
Query: `days` (1-365, default 30).

**Response** `200 OK`
```json
{ "window_days": 30, "input_tokens": 1840233, "output_tokens": 91200,
  "judge_calls": 4120, "eval_runs": 1370,
  "by_evaluator": [{ "evaluator": "fluiq.agent_eval", "input_tokens": 1700000,
                     "output_tokens": 84000, "judge_calls": 3900, "eval_runs": 1200 }] }
```

Aggregated in ClickHouse rather than by the caller. The `judge_*` columns on
`fluiq.evaluations` carry a whole eval message's totals on its **first** row, with
later rows of the same message at zero, because a jury's calls are not divisible
per metric. `SUM` over the window is therefore exact and any per-row or `AVG`
reading is meaningless. `eval_runs` counts distinct `trace_id`, which is the
billable unit; metric rows are not.

Returns tokens, never cost: the price of a token is a policy decision that
changes, while the count is ground truth.

---

## Provider credentials (BYOK)

Customer-supplied provider keys, used to run an org's judge calls on its own
provider account. Envelope-encrypted at rest (see `shared/crypto.py`): a
per-credential data key from KMS, AES-256-GCM ciphertext in Postgres, and the org
id bound in as additional authenticated data so a row read under the wrong org
fails its tag check instead of decrypting.

**There is no endpoint that returns a stored key**, for any role including admin.

### GET `/api/v1/credentials`

Lists this org's credentials (JWT). Never includes key material.

**Response** `200 OK`
```json
{ "configured": true, "providers": ["anthropic", "azure_openai", "bedrock", "gemini", "moonshot", "openai"],
  "credentials": [{ "credential_id": "…", "provider": "anthropic", "label": "Prod",
                    "key_preview": "…a1B2", "fingerprint": "9f2c…", "status": "active",
                    "last_verified_at": "…", "last_error": null, "created_at": "…" }] }
```

`configured: false` means no encryption backend is set on the deployment, so the
feature is off rather than degraded. There is deliberately no unencrypted path.

### POST `/api/v1/credentials`

Saves a key (JWT). Verified against the provider first with a free, token-less
list-models call, so a typo is rejected at paste time rather than surfacing as a
failed eval hours later. A rejected key returns **422**, not 401: the caller's
session is fine, the key is not.

### POST `/api/v1/credentials/{credential_id}/verify`

Re-checks a stored key and flips `status` to `active` or `invalid`.

### DELETE `/api/v1/credentials/{credential_id}`

Hard-deletes the row and flushes the decrypted-data-key cache, so revocation is
immediate rather than eventually consistent. `204`.

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

## Security

### POST `/api/v1/secure/check`

Pre-call security guard. Called by the SDK in block mode before forwarding the prompt. API key auth.

**Available on every plan**, including Free. Gated by scan volume rather than tier:
a 402 is returned only once the org has used its monthly allowance
(`TIER_SECURITY_QUOTAS`), not because of the plan it is on.

**Request**
```json
{ "api_key": "flq_abc123...", "prompt": "Ignore previous instructions and ...",
  "trace_id": "uuid", "guardrail": "default", "context": { } }
```

Flow (order matters for security): **deny-list** (hard block) → **fast pattern check** → **allow-list** short-circuit → **full scan** via the security worker (`security_check_sync`, with `{block_threshold, block_categories, pii_ignore}`) → **pattern-only fallback** on timeout (**fail-open**; set `SECURE_FAIL_CLOSED` to block instead). The deny-list and the pattern block are evaluated *before* the allow-list, so an allow-listed phrase cannot neutralize a real attack embedded next to it. The fast pattern check (`routes/secure/scanners.py`) is tiered and word-boundary/case-sensitive-acronym aware (mirrors the worker), so a single ambiguous phrase (`act as`, `dark mode`, a Jinja `{{`) is LOW, not a HIGH block, and acronyms like `DAN` never match inside `guidance`/`claim`.

**Response** `200 OK`
```json
{ "allow": false, "block_reason": "Blocked by fluiq.secure: prompt_injection",
  "risk_level": "high", "attack_types": ["prompt_injection"] }
```

When `allow: false` the SDK raises `FluiqSecurityError` and the LLM call is never made; a `status:"blocked"` trace is published (dashboard row → blocked) and the policy's `alert_webhook` is fired when `risk_level ∈ alert_on`. When the scan service is unreachable the endpoint returns the pattern-only verdict (fail-open).

> The fast pattern check is the **pattern layer only**. It has no semantic layer
> and no classifier, so it detects materially less than the worker's full gate
> (13.3% vs 35.3% recall on public injection data). That is the latency tradeoff
> of a synchronous pre-call path, not a bug, but it is why the `/secure/check`
> flow escalates to the worker rather than answering from the fast path alone.
> See `docs/security.md` § "Three detection layers".

---

## Demo (public, unauthenticated)

`routes/demo/` backs the free `/response-gate-demo` page. **No auth, no API key,
no LLM call, and therefore no per-request cost.** Model responses are real
transcripts captured once from `claude-haiku-4-5` and committed to
`transcripts.json`; what runs on every request is the genuine gate
(`routes.secure.scanners.check` on the input, `routes.demo.output_scan` on the
response). Nothing is mocked.

Because there is no per-request cost the guards are light and fail **open**: a
per-IP hourly ceiling (`DEMO_PER_IP_HOURLY`, default 120) via Redis, and an input
cap (`DEMO_MAX_INPUT_CHARS`, default 20000). An earlier draft called the model
live and had to fail closed; removing the cost removed that constraint.

### GET `/api/v1/demo/scenarios`

Recorded scenarios with the gate verdict already computed for each.

### POST `/api/v1/demo/scan`

Runs the real gate over text the visitor supplies.

```json
{ "prompt": "...", "response": "..." }
```

`output_scan.py` is deliberately dependency-free (no Presidio, no spaCy) so it
runs inline in the API. It detects planted canaries, credentials, and PII, with
card numbers confirmed by Luhn so ordinary 16-digit order numbers do not trip it.

> **Transcripts are model-pinned and will go stale.** They were recorded against
> Haiku 4.5, which refuses nearly every attack outright. The demo's actual
> finding is that the leak is the PII inside the refusal, not a compliant answer,
> and that framing depends on the recorded model's behaviour.

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
Get / save the org's Slack alert settings (JWT). Both eval and security alerts require **any paid plan** (Starter and above).

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
| `POST /api/v1/datasets/{id}/runs` | Launch a batch run `{ kind: "agentic" \| "security" \| "metrics", depth?, metrics?, custom_judges? }` over every example. `kind:"metrics"` grades each example's recorded answer against its `expected_output` with the chosen metrics (fresh trace ids per run for clean attribution; `metadata.output` is used as the answer when present). Also accepts `judge` and `jury` as `"provider:model"`, applied to every example in the run |
| `GET /api/v1/datasets/{id}/runs` · `GET /api/v1/datasets/runs/{run_id}` | List runs · fetch a run's report (per-metric averages + per-item scores for `metrics` runs) |
| `GET /api/v1/datasets/runs/{run_id}/compare?against={run_id}` | **Run-vs-run regression report** (agentic + metrics kinds): per-metric deltas over examples present in both runs, and per-example `regressed / improved / unchanged` (ε = 0.05), joined by `example_id` |
| `POST /api/v1/datasets/{id}/agents` · `GET`/`DELETE .../agents` | **Connect Agents**: link a traced agent → imports all its runs to date (deduped, full trajectory pinned) and auto-appends future runs |
| `POST /api/v1/ci/eval-runs` · `GET /api/v1/ci/eval-runs/{run_id}` | **CI variants** of launch + report — **API-key** auth (Bearer), `dataset_id` or case-insensitive `dataset_name`. Backing for `python -m fluiq.ci` |

All JWT-authed except the `/ci/eval-runs` pair (API key).

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

### GET `/api/v1/evaluate/models`
JWT-authed. The chat-model catalog that drives every model picker in the app
(Prompts compare drawer, judge/jury selectors, Datasets + Traces eval). Read
straight from the `model_prices` table (`modality='Text'`, providers Anthropic /
OpenAI / Google / Moonshot) so there is **no hardcoded model list** in the
frontend or the API — adding a row to `model_prices` makes the model selectable
everywhere. Non-chat models (audio/image/embedding/realtime/dated snapshots) are
filtered out and slugs are prettified for display.
```json
{ "models": [ { "id": "claude-sonnet-5", "label": "Claude Sonnet 5", "provider": "anthropic" } ] }
```

### POST `/api/v1/evaluate/compare`
JWT-authed **BYOK**. Runs the same prompt against several models **in parallel,
across providers** (Anthropic / OpenAI / Gemini / Moonshot), each call made with
the org's own stored provider key (see **Provider credentials**) over plain
`httpx` — no provider SDKs. Models are validated against `/evaluate/models`.
Returns per-model output, latency, tokens, and estimated cost (priced from
`model_prices`). When `metrics` (and a `judge_model`) are supplied, each model's
output is also scored on those metrics by a BYOK LLM-as-judge and returned as a
per-model `metrics` array. A model whose provider key is missing returns a
per-model error rather than failing the whole request.

**Request** `{ "prompt": "...", "models": ["claude-sonnet-5", "gpt-5.6-sol"],
"metrics"?: ["relevance"], "judge_model"?: "claude-haiku-4-5", "context"?: "..." }`

### POST `/api/v1/evaluate/trace-metrics`
JWT-authed **BYOK**. Scores a **single trace** (one LLM turn — including one that
makes tool calls) on built-in metrics and/or custom client judges, using the
org's own provider key for the judge. Backs the **single-run** mode of the trace
drawer's Evaluation tab. Results are persisted to ClickHouse `evaluations`
(`evaluator = "fluiq.eval"`) and streamed back over SSE (one enriched message per
metric), so the score renders inline on the trace.

**Request** `{ "trace_id": "uuid", "root_trace_id": "uuid?", "prompt": "...",
"response": "...", "context"?: "...", "metrics": ["relevance", "coherence"],
"judge_model": "claude-haiku-4-5", "custom_judges"?: { "refund-policy": 0.9 },
"thresholds"?: {} }` → same `EvaluateResponse` shape as `/evaluate`.

### POST `/api/v1/evaluate/agentic`
JWT-authed. Triggers **agentic (multi-layer) evaluation** for a whole run — fired by the **Run Agentic Evaluation** button in the trace drawer's **multi-run** mode (a root trace with more than one LLM/agent turn; a single LLM turn — even one with tool calls — uses `/evaluate/trace-metrics` instead). Fetches every span sharing the `root_trace_id`, publishes one `agent_eval` job to the eval worker (async), and returns immediately; layered results (deterministic + tool-selection + trajectory [+ panel]) stream back over the SSE `trace.enriched` channel. Quota-gated.

**Request**
```json
{ "trace_id": "uuid", "root_trace_id": "uuid?", "depth": "fast|standard|deep?",
  "judge": "anthropic:claude-sonnet-5",
  "jury": ["anthropic:claude-haiku-4-5", "openai:gpt-4o-mini"] }
```

`judge` and `jury` are `"provider:model"` strings, the same vocabulary the API,
the Kafka message, and the evaluator all use, so there is no translation layer to
keep in sync. Both fall back to the server default when absent or unparseable, so
an older SDK keeps working and a bad value from a dropdown degrades rather than
failing a run. `jury` applies only at `depth: "deep"`, where a panel is convened.

With a saved provider credential the selected provider's key is the org's own, so
judge tokens bill to their account (see **Provider credentials** above).
**Response** `200 OK` `{ "ok": true, "trace_id": "uuid", "status": "queued", "events": 7 }` · `status:"skipped"` when eval quota is exceeded. **Errors** · `404` no trace events · `422` invalid id.

### GET `/api/v1/evaluate/recent-evals`
CI eval gate (API key auth — no login, safe for CI). Query: `window_minutes` (def 30), `threshold` (def 0.7), `limit` (def 200). No tier gate. *Moved here from `/api/v1/optimize/evals` when the optimization pillar was removed.*
```json
{ "window_minutes": 30, "total": 45, "passed": 40, "failed": 5, "avg_score": 0.883,
  "entries": [{ "trace_id": "uuid", "metric": "hallucination", "score": 0.91, "evaluator": "fluiq.eval", "judge_model": "claude-haiku-4-5-20251001" }] }
```

### GET `/api/v1/evaluate/agentic-summary?window_hours=24`
JWT-authed. Aggregates agentic-eval health for the Overview tile over a window (1..720h): run count, run pass-rate, avg run score, and per-layer average scores.

**Response** `200 OK`
```json
{ "window_hours": 24, "runs": 42, "pass_rate": 0.83, "avg_run_score": 0.79,
  "layers": [ { "layer": "deterministic", "score": 0.88, "count": 42 },
              { "layer": "tool_selection", "score": 0.81, "count": 42 },
              { "layer": "trajectory", "score": 0.74, "count": 40 } ] }
```

### Judge Prompts (org-editable) — `/api/v1/eval/judge-prompts`

JWT-authed. A customer's view of the LLM-as-judge prompt templates their
evaluations use, with per-org overrides. Resolution in the evaluator worker is
**org override → platform template → code default**; every eval score records
which one produced it (`details.judge_prompts`).

| Endpoint | Description |
|----------|-------------|
| `GET /api/v1/eval/judge-prompts` | All prompts with this org's override state (`template` = effective, `platform_template` = what reset reverts to, `is_overridden`, `version`) |
| `PUT /api/v1/eval/judge-prompts/{name}` | Save an org override `{ template }` — `400` if a required `$var` placeholder is dropped |
| `POST /api/v1/eval/judge-prompts/{name}/reset` | Delete the override → revert to the platform prompt (history kept) |
| `GET .../versions` · `POST .../restore/{version}` | Org version history · restore a version |

### Feedback & Annotations

Human signals, stored in the ClickHouse `evaluations` table
(`evaluator = 'human.feedback' | 'human.annotation'`) so they render next to
judge scores in the trace drawer — but excluded from the automated quality
rollup.

| Endpoint | Description |
|----------|-------------|
| `POST /api/v1/feedback` | **API-key** auth. End-user feedback `{ trace_id, value: bool \| 0..1, name?: "thumbs", comment?, root_trace_id? }`. Backing for `fluiq.feedback()`. `202` |
| `POST /api/v1/traces/{trace_id}/annotations` | JWT-authed. Team verdict `{ value, metric?, comment?, root_trace_id? }` — e.g. agree/disagree with a judge score (`metric` targets it). `201` |

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
