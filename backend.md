# Fluiq Backend — Architecture Reference

The Fluiq backend (`fluiq-api/`) is a FastAPI application backed by Kafka, ClickHouse, PostgreSQL, Redis, and S3. It is the ingestion endpoint for SDK trace events, the authentication gateway (password + Google/GitHub OAuth), the data layer for the dashboard, the blog CMS, and the synchronous request/reply bridge to the security worker.

The heavy ML-based work (LLM-as-judge evaluation, torch/spaCy security scanning, trace persistence) runs in the separate `fluiq-workers/` processes. The API only *produces* jobs to Kafka and, for the few synchronous paths, *awaits a reply*. See `docs/workers.md` for the worker side.

---

## Tech Stack

| Component | Technology | Role |
|-----------|-----------|------|
| API server | FastAPI (Python 3.11+) | HTTP layer, JWT/OAuth auth, route handling |
| Message queue | Kafka (local: PLAINTEXT docker / prod: **AWS MSK**, SASL/SCRAM-SHA-512 over TLS) | Async trace / eval / security fan-out + sync request-reply |
| Trace store | ClickHouse (self-hosted on **AWS EC2**) | Append-only traces, costs, evaluations, security scans, audit log |
| Relational DB | PostgreSQL (**AWS RDS**) | Users, orgs, API keys, prompts, datasets, guardrail policies, alert settings, blog posts, model prices |
| Cache / reply bus | Redis | SDK cache proxy; in-process Kafka reply-future correlation |
| Object storage | S3 (private bucket) | Blog media — served via short-lived presigned URLs |
| Realtime | Server-Sent Events (SSE) | Live trace streaming to the dashboard |
| Auth | JWT (HS256) + OAuth2 | Access token + refresh token; Google & GitHub social login; OTP password reset |
| Email | Resend (HTTP) / SMTP | Password-reset OTP, model-request notifications, contact form |

---

## Entry Point

```
fluiq-api/main.py
```

`main.py` creates the FastAPI `app`, registers every router, configures CORS (incl. a localhost-origin regex for the blog prerender step), installs `AuditMiddleware` and a CORS-aware unhandled-exception handler, and wires the lifespan startup/shutdown hooks.

### Lifespan (startup / shutdown)

```python
await kafka_queue.start()                 # producer + reply consumers
await postgres_client.start()
await seed_judge_prompts()                # seed/refresh LLM-as-Judge prompt defaults
await clickhouse_client.start()
await trace_consumer.start()              # SSE fan-out (per-replica UUID group)
await alert_consumer.start()              # Slack alerts (stable shared group)
await security_reply_consumer.start()     # sync security_check / response_gate replies
await playground_reply_consumer.start()   # prompt playground eval replies
```

All resources are torn down in reverse order on shutdown.

---

## Router Map

All routers are mounted in `main.py`. Dashboard routers use JWT (`get_current_session`); SDK routers use an API key (`Authorization: Bearer flq_...` or request body).

| Router | Prefix | Auth | Description |
|--------|--------|------|-------------|
| `trace.router` | `/api/v1` | API key / JWT | `/ingest` (SDK), `/traces`, `/traces/stream`, `/traces/spending` |
| `agents_router` | `/api/v1` | JWT | `/agents/summary` |
| `quota_router` | `/api/v1` | JWT | `/quota` |
| `evaluate_router` | `/api/v1` | JWT / API key | `/evaluate` |
| `optimize_router` | `/api/v1/optimize` | JWT or API key (per endpoint) | Cache stats, profile, Redis proxy, CI eval gate |
| `secure_router` | `/api/v1` | API key | `/secure/check` (pre-call guard) |
| `guardrails_router` | `/api/v1` | JWT | `/guardrails` CRUD + `/guardrails/list` |
| `alerts_router` | `/api/v1` | JWT | `/alerts` get/save + `/alerts/test` |
| `prompts_router` | `/api/v1` | JWT or API key | `/prompts` CRUD, versions, environments, `/prompts/fetch/{slug}` |
| `datasets_router` | `/api/v1` | JWT | `/datasets` CRUD |
| `models_router` | `/api/v1` | mixed | `/models`, `/models/request`, `/models/report` |
| `blog_router` | `/api/v1` | public + admin | Public posts/media + admin CMS |
| `audit_router` | `/api/v1` | JWT (admin) | Audit log access |
| `contact_router` | `/api/v1` | — | Contact form |
| `auth.auth_router` | `/auth` | — | Register, login, refresh, logout, OAuth, password reset |
| `api_keys_router` | `/api-keys` | JWT | API key management |
| `admin_router` | `/admin` | Admin JWT | Internal ops — stats, users, orgs, plans, eval allowance, judge prompts, infra SQL/worker console, blog |

---

## Middleware & Error Handling

### CORS (`CORSMiddleware`)

`FRONTEND_BASE_URL` controls the primary allowed origin; the `https://` and `https://www.` variants are both whitelisted. In addition, a localhost-origin regex (`https?://(localhost|127.0.0.1)(:\d+)?`) is always allowed so the frontend **prerender step** — a headless browser on `127.0.0.1:<random-port>` that fetches published blog posts at build time — passes CORS. Falls back to `["*"]` when no frontend URL is set.

### CORS-aware exception handler

A top-level `@app.exception_handler(Exception)` logs the real cause and re-applies the CORS allow-origin headers to the 500 response. Without this, a 500 raised outside `CORSMiddleware` reaches the browser as a misleading "CORS error" that masks the actual failure (common on cold-start DB timeouts).

### AuditMiddleware (`middleware/audit.py`)

Logs authenticated API requests to the ClickHouse `audit_log` table (HMAC-signed with `AUDIT_HMAC_SECRET` for tamper-evidence). Surfaced via `/api/v1/audit`.

---

## Database Clients

### PostgreSQL — `db_queues/postgresql/`

Async pool (`asyncpg`), now hosted on **AWS RDS** (migrated off the old managed Postgres). TLS verification is enabled when `POSTGRES_SSL_CA_FILE` points at the RDS CA bundle (verify-full); local dev runs without it. Tables:
- Users, organizations, API keys (hashed), revoked refresh tokens, password-reset OTPs
  - `organizations.eval_quota_bonus` — admin-granted adjustment added on top of the tier's monthly eval quota
- `guardrail_policies` — per-org guardrail policies (see below)
- `alert_settings` — per-org Slack alert configuration
- Prompt templates + version history; datasets + examples
- `eval_judge_prompts` + `eval_judge_prompt_versions` — platform-global LLM-as-Judge prompts (admin-editable, seeded on startup) read by the evaluator worker; see `docs/workers.md`
- `model_prices` — model cost table (read by the tracer worker for cost estimation; surfaced via `/models`)
- Blog posts (admin CMS) — body + media object keys

Helper modules include `auth.py` (`resolve_api_key`, `get_org_tier`, `get_org_eval_bonus`, admin user/eval helpers), `guardrails.py`, `eval_prompts.py` (judge-prompt CRUD + seeding) with `judge_prompt_defaults.py`, and the alert/prompt/dataset/blog helpers.

### ClickHouse — `db_queues/clickhouse/`

Async client (`clickhouse-connect`), **self-hosted on AWS EC2** (cut over from ClickHouse Cloud). Read queries live in `ClickHouseQueryMixin`. Tables:
- `traces` — all SDK events (JSON `event` column)
- `trace_costs` — per-trace cost breakdowns
- `evaluations` — LLM-judge scores per trace
- `security` — agentic security scan results (written by the security worker)
- `audit_log` — HMAC-signed request audit trail

Key query methods: `fetch_traces()` (paginated, joins evals + security + costs), `count_traces()` / `count_evaluations()` (quota), `fetch_cache_stats()`, `fetch_prompt_cache_stats()`, `fetch_optimization_profile()`, `fetch_agent_summary()`, `fetch_recent_evals()`, plus spending rollups.

### Redis — used by `routes/optimize/` and `db_queues/kafka/`

1. **SDK cache proxy** — `/optimize/cache` GET/POST namespace keys per-org (`fluiq:{org_id[:8]}:{key}`) with TTL.
2. **Reply correlation** — the synchronous Kafka request/reply paths (`security_check_sync`, `response_gate_check`, playground eval) correlate replies by `correlation_id`.

`REDIS_SDK_URL` is the URL handed back to the SDK in the cache profile (may differ from the internal `REDIS_URL`).

---

## Kafka Integration — `db_queues/kafka/`

### Auth — `kafka_auth_kwargs()`

Shared by the producer and all consumers. `KAFKA_SECURITY_PROTOCOL=PLAINTEXT` (local docker) → no auth; `SASL_SSL` → SCRAM-SHA-512 username/password over TLS for **AWS MSK** (broker certs chain to Amazon Trust Services, so no CA file is needed). The identical helper exists in each worker's `config.py`.

### Producer

`KafkaQueue.add_job(message, topic, key)` — JSON-serializes (`Decimal → str`), publishes with `enable_idempotence=True`, `acks=all`. `key` is the `organization_id` for ordered per-org delivery. `KAFKA_MAX_REQUEST_SIZE` / `KAFKA_MAX_FETCH_BYTES` are raised to **10 MB** because large prompts/responses/tool-outputs exceeded the ~1 MB aiokafka/broker default (which surfaced as a 500 on `/ingest` with `MessageSizeTooLargeError`).

### Topics

| Env var | Purpose |
|---------|---------|
| `KAFKA_TRACE_TOPIC` | Raw SDK trace events → tracer worker |
| `KAFKA_TRACE_PERSISTED_TOPIC` | Enriched/persisted traces → SSE + alert consumers |
| `KAFKA_EVAL_TOPIC` | Evaluation jobs → evaluator worker |
| `KAFKA_SECURITY_TOPIC` | Security jobs → **dedicated security worker** (keeps torch/spaCy out of the evaluator) |
| `KAFKA_SECURITY_REPLY_TOPIC` | Sync security-check / response-gate replies → API |
| `KAFKA_PLAYGROUND_REPLY_TOPIC` | Prompt playground eval replies → API |

### Consumers (in the API)

| Consumer | Topic | Group | Purpose |
|----------|-------|-------|---------|
| `trace_consumer` | persisted | per-replica UUID | SSE fan-out (every replica sees every event) |
| `alert_consumer` | persisted | **stable shared** (`KAFKA_ALERTS_GROUP_ID`) | Slack alerts — fires once, not once-per-replica |
| `security_reply_consumer` | security reply | — | Resolves `wait_for_reply()` futures for `/secure/check` + response gate |
| `playground_reply_consumer` | playground reply | — | Resolves playground eval futures |

---

## Trace Ingestion — `routes/trace/__init__.py`

`POST /api/v1/ingest` is the primary SDK endpoint. Processing order:

1. **Resolve API key** → 401 on failure.
2. **Quota check** — trace quota enforced before any Kafka work → 402 on over-cap.
3. **Strip internal SDK flags** — `_eval_config`, `_security_config` (and `_`-prefixed cache flags) extracted and removed from the stored event.
4. **Response gate** (synchronous, only when the org's policy has `scan_responses=True` and the event is an LLM response): publishes a `response_gate_check` job to `KAFKA_SECURITY_TOPIC`, awaits the reply, and stamps the gate decision into the event before persistence.
5. **Kafka publish** — event published to `KAFKA_TRACE_TOPIC` (returns 413 on `MessageSizeTooLargeError`).
6. **Eval fan-out** — if `_eval_config` present, LLM trace, and eval quota not over: retrieval traces → `KAFKA_EVAL_TOPIC` (`auto`); LLM eval configs → `sdk_llm` job.
7. **Security fan-out** — if `_security_config` present: publishes `sdk_security` (`auto_security_scan`) to `KAFKA_SECURITY_TOPIC`, forwarding the org's `pii_ignore` and `allowed_tools` so the worker can apply policy.

The expanded `GET /api/v1/traces` accepts `key_id`, `agent_key`, `agent_kind`, `root_trace_id`, `roots_only`, `limit`, `offset`, `sort`, `status`, `security`, `integration`, and `quality` filters. `GET /api/v1/traces/spending` returns cost rollups.

---

## Security Pipeline — `routes/secure/__init__.py`

### Pre-call guard: `POST /api/v1/secure/check` (Growth+)

Gated to `{"Growth", "Enterprise"}` tiers (402 otherwise). Flow:

1. **Allow-list** short-circuit → `allow=True`.
2. **Deny-list** short-circuit → block.
3. **Fast pattern check** (`scanners.check()`, deterministic, near-zero latency) — blocks obvious injection/jailbreak/skeleton-key before any Kafka round-trip.
4. **Full scan** — publishes `security_check_sync` to `KAFKA_SECURITY_TOPIC` with `{block_threshold, block_categories, pii_ignore}` and awaits the reply (`KAFKA_SECURITY_CHECK_TIMEOUT`).
5. **Fallback** — on timeout/error, re-uses the step-3 pattern result with policy applied (**fail-open**).

`CheckResponse = {allow, block_reason, risk_level, attack_types}`. On a block with a `trace_id`, a `status:"blocked"` trace is published so the dashboard row transitions from running→blocked, and `alert_webhook` is POSTed (with retries) when `risk_level ∈ alert_on`.

### Post-call scan & response gate

Run in the **security worker** (`auto_security_scan`, `response_gate_check`). Results land in the ClickHouse `security` table and fan out to SSE. See `docs/workers.md`.

---

## Guardrail Policies — `routes/guardrails/__init__.py`

One policy per `(org, slug)`; default slug `"default"` (undeletable). Cached 60 s. Policy shape:

| Field | Type | Notes |
|-------|------|-------|
| `block_threshold` | `"medium" \| "high"` | `medium` lowers the block bar |
| `warn_threshold` | `"low" \| "medium" \| "high"` | |
| `block_categories` | `string[]` | empty = block on any category |
| `custom_deny_list` | `string[]` | phrases that hard-block |
| `custom_allow_list` | `string[]` | phrases that bypass scanning |
| `pii_ignore` | `string[]` | PII entity types to suppress |
| `allowed_tools` | `string[]` | tool allowlist (forwarded to the worker) |
| `alert_webhook` | `string?` | POSTed on block |
| `alert_on` | `string[]` | risk levels that trigger the webhook (default `["high"]`) |
| `scan_responses` | `bool` | enables the synchronous response gate on `/ingest` |

`ALL_CATEGORIES` = `prompt_injection, jailbreak, skeleton_key, semantic_attack, pii_detected, secrets_detected, indirect_injection, rag_poisoning, tool_exfiltration, tool_policy_violation, cross_agent_injection`.

`PII_ENTITIES` = `US_SSN, CREDIT_CARD, IBAN_CODE, CRYPTO, US_PASSPORT, EMAIL_ADDRESS, PHONE_NUMBER, PERSON, LOCATION, IP_ADDRESS`.

---

## Alerts — `routes/alerts/__init__.py` + `realtime/alert_consumer.py`

Per-org Slack alerting stored in `alert_settings`. `GET/PUT /alerts` configure it; `POST /alerts/test` fires a test message. Eval alerts require **Team+**, security alerts require **Growth+**.

`alert_consumer` runs on a **stable shared consumer group** so each enriched/persisted event is handled by exactly one replica (alerts fire once). It joins the per-org `AlertSettings`, then fires Slack on eval-threshold breaches, rolling failure-rate thresholds, and matching security categories — in realtime or buffered into a digest.

---

## Optimization Layer — `routes/optimize/__init__.py`

- `GET /optimize/profile` (SDK, API key, **Team+**) — returns `REDIS_SDK_URL`, per-org key prefix, hot models, TTL, and an estimated hit rate derived from ClickHouse traffic.
- `GET /optimize/cache/{key}` & `POST /optimize/cache` (SDK proxy) — Redis GET/SET, org-namespaced.
- `GET /optimize/cache-stats` (JWT) — Redis hit/miss by cache kind.
- `GET /optimize/prompt-cache-stats` (JWT) — provider prefix-cache token aggregates (Anthropic read/creation + OpenAI/Gemini cached).
- `GET /optimize/evals` (SDK, API key) — CI eval gate (recent scores, pass/fail vs threshold).

---

## Prompt Management — `routes/prompts/__init__.py`

CRUD + versioned templates. Every update creates a new version; previous versions are preserved and restorable. Prompts are promoted into named **environments** (`development`/`staging`/`production`) independently via `/environments/{env}`; `/deploy` is the legacy `{deploy: bool}` toggle. `GET /prompts/fetch/{slug}?env=` is the SDK read path used by `fluiq.fetch_prompt()`.

Each prompt has a `kind` — `'completion'` (default) or `'judge'`. A **judge** prompt is a client-authored LLM-as-judge template (placeholders `$question`/`$answer`/`$context`, returns `{"score","reason"}`) that the customer references by slug from `fluiq.eval(custom_judges={slug: threshold})`. The block-mode `/evaluate` path resolves it via `get_custom_judge_template(org_id, slug)`; the evaluator worker resolves it for warn mode. See `docs/workers.md` and `docs/sdk.md`.

---

## Blog CMS — `routes/blog/__init__.py`

In-house admin CMS (TipTap editor on the frontend). Public endpoints serve posts and media; media lives in a **private S3 bucket** and `/blog/media/{id}` issues a 307 redirect to a short-lived presigned GET URL (`S3_PRESIGN_TTL`). Admin endpoints (`require_admin`) handle list/create/get/patch/delete/publish and media upload (≤5 MB, image types). On publish/update/unpublish the API POSTs the **Render deploy hook** (`RENDER_DEPLOY_HOOK_URL`) to trigger a static rebuild + prerender so posts are baked into static HTML for SEO (fails open if unset).

---

## Real-time SSE — `realtime/`

`trace_consumer` subscribes to the persisted topic (per-replica group) and pushes to connected SSE clients keyed by org:

| Kafka message `kind` | SSE event |
|---------------------|-----------|
| `started` | `trace.started` |
| `persisted` | `trace` |
| `enriched` (cost/eval/security) | `trace.enriched` |

`GET /api/v1/traces/stream` maintains an in-process broker keyed by org ID; connections close cleanly on disconnect or `402` quota events.

---

## Tiers & Quotas — `shared/quotas.py`

Four tiers. `(trace_quota, eval_quota)` per calendar month; `-1` = unlimited. Counts come from ClickHouse filtered on org + current month, with a 60 s in-process TTL cache and optimistic bumping on the hot `/ingest` path.

| Tier | Traces / month | Evaluations / month |
|------|----------------|---------------------|
| Free | 50,000 | 1,000 |
| Team | Unlimited | 10,000 |
| Growth | Unlimited | 100,000 |
| Enterprise | Unlimited | Unlimited |

Admins can adjust a single org's eval allowance via `organizations.eval_quota_bonus`
(Admin → Evaluations): `get_quota_status()` adds it to the tier eval quota
(`max(0, tier_quota + bonus)`; unlimited tiers stay unlimited), and the adjustment
endpoint invalidates the org's cache so it applies on the next `/ingest`.

Feature gates: `fluiq.eval()` warn mode is open to all; `fluiq.secure()` / `/secure/check` require **Growth+**; `/optimize/profile` and `fluiq.optimize()` require **Team+**; security alerts require **Growth+**, eval alerts **Team+**.

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `JWT_SECRET`, `JWT_ALGORITHM`, `JWT_EXPIRE_MINUTES`, `JWT_REFRESH_EXPIRE_DAYS` | JWT config |
| `POSTGRES_DSN`, `POSTGRES_POOL_MIN/MAX` | RDS Postgres pool |
| `POSTGRES_SSL_CA_FILE` | RDS CA bundle → verify-full TLS (unset locally) |
| `POSTGRES_*_TABLE` | user / org / revoked-token / password-reset / guardrails / alerts table names |
| `CLICKHOUSE_HOST/PORT/USER/PASSWORD/DATABASE` | EC2 ClickHouse connection |
| `CLICKHOUSE_TRACE_TABLE`, `CLICKHOUSE_TRACE_COSTS_TABLE`, `CLICKHOUSE_EVALUATIONS_TABLE`, `CLICKHOUSE_SECURITY_TABLE`, `CLICKHOUSE_AUDIT_TABLE` | table names |
| `AUDIT_HMAC_SECRET` | audit-log HMAC signing key |
| `KAFKA_BOOTSTRAP_SERVERS` | broker(s) |
| `KAFKA_SECURITY_PROTOCOL` | `PLAINTEXT` (local) / `SASL_SSL` (MSK) |
| `KAFKA_SASL_MECHANISM`, `KAFKA_SASL_USERNAME`, `KAFKA_SASL_PASSWORD` | MSK SCRAM creds |
| `KAFKA_TRACE_TOPIC`, `KAFKA_TRACE_PERSISTED_TOPIC`, `KAFKA_EVAL_TOPIC`, `KAFKA_SECURITY_TOPIC` | topics |
| `KAFKA_SECURITY_REPLY_TOPIC`, `KAFKA_SECURITY_CHECK_TIMEOUT` | sync security reply path |
| `KAFKA_PLAYGROUND_REPLY_TOPIC`, `KAFKA_PLAYGROUND_CHECK_TIMEOUT` | playground reply path |
| `KAFKA_ALERTS_GROUP_ID` | stable alert consumer group (default `api-alerts`) |
| `KAFKA_MAX_REQUEST_SIZE`, `KAFKA_MAX_FETCH_BYTES` | 10 MB message ceilings |
| `REDIS_URL`, `REDIS_SDK_URL`, `REDIS_DEFAULT_TTL_SECONDS` | cache proxy + reply bus |
| `RESEND_API_KEY`, `SMTP_FROM_EMAIL`, `SMTP_FROM_NAME` | email |
| `PASSWORD_RESET_OTP_LENGTH`, `PASSWORD_RESET_EXPIRE_MINUTES` | OTP password reset |
| `GOOGLE_CLIENT_ID/SECRET`, `GITHUB_CLIENT_ID/SECRET`, `API_BASE_URL` | OAuth |
| `AWS_REGION`, `S3_BLOG_MEDIA_BUCKET`, `S3_PRESIGN_TTL` | blog media on S3 |
| `RENDER_DEPLOY_HOOK_URL` | static rebuild on blog publish (optional) |
| `FRONTEND_BASE_URL` | CORS allowed origin |
| `ANTHROPIC_API_KEY` | judge / server-side LLM |

---

## Project Structure

```
fluiq-api/
├── main.py                   # FastAPI app, router registration, CORS, lifespan
├── config.py                 # Env vars + kafka_auth_kwargs()
├── middleware/
│   └── audit.py              # AuditMiddleware → ClickHouse audit_log (HMAC)
├── routes/
│   ├── trace/                # /ingest, /traces, /traces/stream, /traces/spending
│   ├── auth/                 # password + Google/GitHub OAuth + OTP reset
│   ├── api_keys/             # API key CRUD
│   ├── agents/               # /agents/summary
│   ├── quota/                # /quota
│   ├── optimize/             # cache stats, profile, Redis proxy, CI eval gate
│   ├── secure/               # /secure/check pre-call guard (+ scanners.py)
│   ├── guardrails/           # guardrail policy CRUD
│   ├── alerts/               # Slack alert config + test
│   ├── evaluate/             # synchronous /evaluate
│   ├── prompts/              # templates, versions, environments, SDK fetch
│   ├── datasets/             # dataset + example CRUD
│   ├── models/               # /models, /models/request, /models/report
│   ├── blog/                 # public posts/media + admin CMS
│   ├── audit/                # audit log read
│   ├── admin/                # internal admin ops
│   └── contact/              # contact form
├── db_queues/
│   ├── clickhouse/           # client + ClickHouseQueryMixin
│   ├── kafka/                # producer + reply consumers + wait_for_reply
│   └── postgresql/           # auth, guardrails, alerts, prompts, datasets, blog,
│                             #   eval_prompts (+ judge_prompt_defaults), schema.sql
├── realtime/                 # SSE broker + trace_consumer + alert_consumer
└── shared/                   # quotas, email, s3, slack helpers
```
