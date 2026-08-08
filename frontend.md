# Fluiq Frontend — Architecture & API Integration Reference

The Fluiq frontend (`fluiq-frontend/`) is a **Next.js (App Router)** application (migrated off the old Vite + React Router SPA). It still uses Redux Toolkit for auth/notification state. This document covers the app structure, the HTTP layer, auth flow, SSE streaming, page-level API calls, and TypeScript types.

---

## Configuration

```
NEXT_PUBLIC_API_BASE_URL=https://api.getfluiq.com   # production
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000        # local dev (default fallback)
```

`src/lib/api.ts` reads `process.env.NEXT_PUBLIC_API_BASE_URL` and strips any trailing slash. All API paths below are relative to this base.

> Some marketing pages need `next dev --webpack` in local dev.

---

## App Structure

Next.js App Router lives in `src/app/`. Route files are thin wrappers that render presentational **screen** components from `src/screens/` (the bulk of the UI). Shared building blocks live in `src/components/`, contexts in `src/contexts/` (incl. `ThemeContext` for class-based dark mode and the docs/examples `LanguageContext`), and the API/util layer in `src/lib/`.

```
src/
├── app/                      # Next.js App Router (thin route files)
│   ├── layout.tsx            # Root layout
│   ├── page.tsx              # Home (marketing)
│   ├── providers.tsx         # Redux + Theme providers
│   ├── sitemap.ts robots.ts  # SEO
│   ├── login/ signup/ auth/ forgot-password/ reset-password/
│   ├── pricing/ contact/ privacy/ terms/
│   ├── documentation/ examples/   # SDK docs (Python/TS language toggle)
│   ├── integrations/ blog/
│   ├── observability/ security/ evaluation/ optimization/ prompts/  # pillar landing pages
│   ├── *-alternative/        # comparison pages (langfuse, langsmith, helicone,
│   │                         #   braintrust, lakera, portkey)
│   ├── llm-cost-calculator/  # tool page (uses /models)
│   ├── response-gate-demo/   # free tool — recorded fluiq.secure(mode="block")
│   │                         #   transcripts, zero inference cost (see below)
│   ├── benchmark/            # guardrail benchmark results (see below)
│   ├── infrager/             # landing page for Infrager (separate OSS product,
│   │                         #   app lives at infrager.getfluiq.com)
│   ├── admin/
│   └── dashboard/            # authenticated app (see below)
├── screens/                  # presentational components rendered by routes
│   ├── Dashboard/{Agents,Alerts,ApiManagement,Audit,Datasets,GettingStarted,
│   │              Guardrails,Insights,JudgePrompts,Optimize,Overview,Profile,
│   │              Prompts,Security,Tests,Traces,UserManagement}
│   ├── Home/ Pricing/ Comparisons/ Platform/ Tools/ Documentation/
│   ├── Integrations/ Blog/ Authentication/ Legal/ Admin/
│   ├── Infrager.tsx          # /infrager landing page
├── components/ contexts/ lib/ store/ styles/ assets/
```

### Free tool pages

Two public pages exist to be linked to rather than to sell, both under
`src/screens/Tools/`.

**`/response-gate-demo`** (`ResponseGateDemo.tsx`) replays recorded
`fluiq.secure(mode="block")` transcripts from `fluiq-api/routes/demo/transcripts.json`
via `POST /demo/output-scan`. It runs no inference, so the page costs nothing per
visit. The transcripts are **pinned to a specific model** and will go stale: they
were recorded against Haiku 4.5, which refuses nearly every attack outright, so
the page's actual finding is that the leak is the PII inside the refusal rather
than a compliant answer.

**`/benchmark`** (`Benchmark.tsx`) renders entirely from
`src/lib/benchmarkData.json`, which is **generated**, not hand-written:
`guardrail-bench/make_report.py` writes `REPORT.md`, `report.html` and that JSON
in one pass, so the page, the report and
`public/Fluiq-Guardrail-Benchmark.pdf` cannot drift apart. Editing the JSON by
hand is always wrong; re-run the generator.

Conventions worth keeping when touching the page:

- **One row per product.** Competitors appear once, so Fluiq does too. Secondary
  configurations (the API fast path, the demo's inline scanner) score lower and
  are published in a footnote and a PDF appendix rather than dropped.
- **Bar widths are inline styles, never animations.** A scroll-triggered width
  that fails to fire leaves the bar at zero, which misreports a score rather
  than just missing an effect.
- **Recall and false alarm get a panel each.** Both run 0–100% but point in
  opposite directions; on a shared axis the taller bar reads as the better
  product. Fluiq is `#1860D3` (`#6FA8FF` dark), every competitor is the same
  neutral grey, and the product name is on every row so colour never carries
  identity alone.
- **The run date is displayed and comes from `data.generated`**, in the hero, on
  each corpus's source line, and in the Dataset JSON-LD (`datePublished` /
  `dateModified`). Formatted without `toLocaleDateString`, whose output depends
  on the runtime locale and would differ between the server render and the
  browser.
- **Vendor logos are wired but empty.** `LOGOS` in `Benchmark.tsx` maps a slug to
  `public/logos/<slug>.svg`; every unmapped row falls back to a neutral
  monogram. Competitors' trademarks are deliberately not vendored into the repo.
  `make_report.py` reads the same slug from `guardrail-bench/logos/`.

### Navigation

`SiteNavbar` composes three hover dropdowns: `NavPlatformDropdown`,
`NavIntegrationsDropdown` and `NavDeveloperDropdown` ("Resources"). Resources is
a three-column mega menu driven by `NAV_GROUPS`:

| Column | Items |
|--------|-------|
| Learn | Blogs, FAQ, Guardrail Benchmark |
| Build | Fluiq Docs, Code Samples |
| Tools | Response Gate Demo, LLM Cost Calculator, polygate, Infrager |

The panel is 880px wide because at 720px every description wrapped to two lines,
which doubled the row height and made three short columns read as one tall
block. There is no mobile variant — the whole nav is desktop-only.

### Dashboard routes — `src/app/dashboard/`

`layout.tsx` + `shell.tsx` provide the authenticated shell (sidebar/nav). Pages:

| Route | Screen | Purpose |
|-------|--------|---------|
| `/dashboard/overview` | Overview | Usage, quota, spending, cache-hit & agentic-eval summary tiles |
| `/dashboard/traces` | Traces | Trace list, detail drawer, span/architecture tree, live SSE, tool selection; the drawer's Evaluation tab runs eval right there (`TraceEvalConfig`): a **single-run** trace (one LLM turn, even with tool calls) shows a metric-chip + custom-scorer picker → `POST /evaluate/trace-metrics`, while a **multi-run** trace (2+ LLM/agent turns) shows the agentic depth/judge/jury config → **Run Agentic Evaluation** (`POST /evaluate/agentic`); both gate on a BYOK provider-key add-key flow; **AnnotateBar** (thumbs + note → `POST /traces/{id}/annotations`); human feedback/annotation rows render with end-user/team badges; per-score **Judge prompts** block shows the exact rendered prompt + version/source behind every eval |
| `/dashboard/agents` | Agents | Per-agent cost / token / latency rollup |
| `/dashboard/security` | Security | Security risk panel; agentic threat verdicts |
| `/dashboard/guardrails` | Guardrails | Guardrail policy editor |
| `/dashboard/alerts` | Alerts | Slack alert configuration |
| `/dashboard/optimize` | Optimize | Redis cache stats; MCP cache; Prompt Caching card |
| `/dashboard/tests` | Tests | Eval scores, CI gate status, dataset management, playground |
| `/dashboard/prompts` | Prompts | Template editor, version history, env deployments, playground; Save form has a **Completion / Judge** type toggle — Judge prompts (`kind='judge'`) are client-authored LLM-as-judges referenced by slug in `fluiq.eval(custom_judges=…)` |
| `/dashboard/datasets` | Datasets | Named trace collections for regression testing; batch runs (**agentic / security / metrics** — metric picker chips), per-run report, and a **Compare vs…** selector producing a run-vs-run regression view (per-metric deltas, regressed examples) |
| `/dashboard/judge-prompts` | JudgePrompts | Org-editable LLM-as-judge prompts (effective template vs platform, required-var chips, save/reset-to-platform, org version history + restore) — backed by `/api/v1/eval/judge-prompts` |
| `/dashboard/api-management` | ApiManagement | API key management |
| `/dashboard/audit` | Audit | Request audit log |
| `/dashboard/getting-started` | GettingStarted | Onboarding / install guide |
| `/dashboard/profile` | Profile | User profile & account settings |

Dashboard routes are guarded by auth — an unauthenticated visit redirects to `/login`.

### Admin routes — `src/app/admin/`

`shell.tsx` wraps the admin layout in `RequireAdmin` (admin-only). Screens live in
`src/screens/Admin/`.

| Route | Screen | Purpose |
|-------|--------|---------|
| `/admin/overview` | Overview | Platform stats |
| `/admin/users` | Users | User list; change plan |
| `/admin/evaluations` | Evaluations | Search a user → view eval usage; add/subtract their org's eval allowance |
| `/admin/judge-prompts` | JudgePrompts | Edit LLM-as-Judge prompts (search, edit with required-var chips, version history + rollback, reset-to-default, add custom prompt) |
| `/admin/organizations` | Organizations | Org list |
| `/admin/plans` | Plans | Plan distribution |
| `/admin/blog` | Blog | Blog CMS (TipTap editor) |
| `/admin/architecture` | Architecture | Live AWS topology (React Flow) — self-hosted Kafka EC2, ECS autoscaling (1→N, CPU 65%), data stores; click a node for details |
| `/admin/infrastructure` | Infrastructure | Store cards (RDS · ClickHouse · Kafka); **Logs**, **Workers**, **Secrets** tabs |
| `/admin/sql-editor` | SqlEditor | Full-page read-only SQL editor with a results panel that drops up from the bottom (expand / CSV export) |

The Infrastructure screen (`src/screens/Admin/Infrastructure/`) has **Overview**
(store cards + a link to the SQL Editor), **Logs** (`LogsTab` → Kafka / ClickHouse
/ Postgres tail), **Workers** (`WorkersTab` → per-worker detail with **Logs** /
**Secrets** sub-tabs + a scaling stepper), and **Secrets** (`SecretsTab`) tabs.
The SQL console is now its own full-page route (`/admin/sql-editor`, `SqlEditor/`)
rather than a half-height panel. Add/delete actions use a shared typed-name
confirmation (`ConfirmDelete.tsx`).

Admin calls (via `authFetch`): `GET/POST /admin/users/{id}/evaluations` (eval
allowance), `GET/POST/PUT /admin/judge-prompts…` (judge prompts), and the
infrastructure endpoints — `POST /admin/infra/query`, `GET /admin/infra/workers`,
`GET /admin/infra/workers/{svc}/logs` (`q`/`start`/`end`/`limit`),
`GET /admin/infra/logs/{source}` (`source` = kafka · clickhouse · postgres),
`GET/PUT/DELETE /admin/infra/secrets`,
`GET/POST/DELETE /admin/infra/workers/{svc}/secrets`, and
`POST /admin/infra/workers/{svc}/scale` — see `docs/api.md` (Admin).

---

## HTTP Layer

### `apiRequest<T>(path, options)` — `src/lib/api.ts`

Base `fetch` wrapper used by all API calls.

```typescript
apiRequest<T>(path: string, options?: {
  method?: "GET" | "POST" | "PUT" | "DELETE" | "PATCH"
  body?: unknown
  token?: string | null
  signal?: AbortSignal
}): Promise<T>
```

Sets `Content-Type: application/json` when a body is present and `Authorization: Bearer <token>` when a token is provided. Parses JSON (falls back to raw text), extracts FastAPI's `{detail}` (incl. validation-error arrays), and throws `ApiError { status, detail }` on non-2xx. Network failures surface as `ApiError(0, ...)`.

### `authFetch<T>(path, options)` — `src/lib/authFetch.ts`

Wraps `apiRequest` with automatic token refresh. Uses the access token from Redux. On `401` it performs one refresh, retries, and logs out if the refresh also fails. Concurrent `401`s are coalesced to a single in-flight refresh.

### Domain helpers — `src/lib/`

`blog.ts` (public blog fetch + media URLs), `models.ts` (cost-calculator `/models`), `useModels.ts` (the eval model catalog from `/evaluate/models`, module-cached; `toSpecs`/`specLabel` turn rows into `provider:model` judge/jury specs), `prerender.ts` (build-time blog prerender), `seo.ts` / `seo-pages.ts` (metadata), `router-compat.tsx` (compat shim easing the React-Router → App-Router migration), `faqs.ts`, `utils.ts`.

### Shared eval building blocks — `src/components/`

Three pieces are reused across the Prompts, Datasets, and Traces eval surfaces so
model lists and BYOK are consistent everywhere:

- **`SearchSelect.tsx`** — single-select searchable combobox; **`MultiSelectDropdown.tsx`** gained a `searchable` mode. Every model/judge/jury picker uses one of these fed by `useModels()`, so there are no hardcoded model catalogs in the UI.
- **`ProviderKeys.tsx`** — the shared BYOK layer: `useProviderKeys()` (reads `/api/v1/credentials`), `missingKeyProviders()`, `MissingKeysCallout`, and `ProviderKeyDialog` (verify + save an encrypted key). Any drawer that will call a provider gates its Run button on the in-use providers having a key and offers an inline add-key flow.

---

## Authentication

### Redux Slice — `src/store/auth/slice.ts`

```typescript
interface AuthState {
  user: UserPublic | null
  organization: OrganizationModel | null
  accessToken: string | null
  refreshToken: string | null
  status: "idle" | "loading" | "succeeded" | "failed"
  error: string | null
}
```

Persisted to `localStorage` under `fluiq.auth`. A separate `src/store/notifications/slice.ts` holds transient UI notifications. Store wiring: `src/store/index.ts`, typed hooks in `src/store/hooks.ts`.

| Thunk | Endpoint | Description |
|-------|----------|-------------|
| `registerThunk` | `POST /auth/register` | Create account, store session |
| `loginThunk` | `POST /auth/login` | Authenticate, store session |
| `refreshThunk` | `POST /auth/refresh` | Rotate token pair |
| `logoutThunk` | `POST /auth/logout` | Revoke refresh token, clear state |

OAuth (Google/GitHub) callbacks land on `/auth/*` routes which hydrate the same session shape.

---

## TypeScript Types — `src/lib/auth-types.ts`

```typescript
type UserType = "Free" | "Team" | "Growth" | "Enterprise" | "Admin"

interface UserPublic {
  user_id: string; email: string; name: string
  user_type: UserType; org_id: string
  created_at: string; updated_at: string | null
}

interface ApiKey { key_id: string; name: string; prefix: string; created_at: string }
interface ApiKeyCreated extends ApiKey { key: string }

interface OrganizationModel {
  org_id: string; name: string; user_id: string
  team_ids: string[]; api_keys: ApiKey[]
  api_key_limit: number; api_key_usage: number
  created_at: string; updated_at: string | null
}

interface AuthSession {
  user: UserPublic; organization: OrganizationModel
  access_token: string; token_type: string
  expires_in: number; refresh_token: string; refresh_expires_in: number
}
```

---

## Dashboard API Calls

All dashboard calls use `authFetch`. Screen-specific types live alongside each screen in `src/screens/Dashboard/*`.

| Screen | Endpoint | Description |
|--------|----------|-------------|
| Overview | `GET /api/v1/quota` | Quota usage by tier |
| Overview | `GET /api/v1/traces/spending` | Spending rollup |
| Overview | `GET /api/v1/optimize/cache-stats?window_hours=24` | Cache hit rate |
| Traces | `GET /api/v1/traces?…` | Filtered/paginated trace list |
| Traces | `POST /api/v1/traces/rollups` | Per-run cost/quality/security/count roll-ups (root rows) |
| Traces | `GET /api/v1/traces/stream` | Realtime SSE stream |
| Agents | `GET /api/v1/agents/summary?limit=&offset=` | Agent roll-up (windowed pagination) |
| Security | `GET /api/v1/traces?security=…&limit=&offset=` | Trace security verdicts (paginated) |
| Guardrails | `GET /api/v1/guardrails` · `GET /api/v1/guardrails/list` | Read policy / slugs |
| Guardrails | `PUT /api/v1/guardrails?slug=` · `DELETE /api/v1/guardrails?slug=` | Save / delete |
| Alerts | `GET /api/v1/alerts` · `PUT /api/v1/alerts` · `POST /api/v1/alerts/test` | Slack alert config |
| Optimize | `GET /api/v1/optimize/cache-stats` · `GET /api/v1/optimize/prompt-cache-stats` | Cache stats |
| Tests | `GET /api/v1/traces` (filtered) · `POST /api/v1/evaluate/playground` | Eval scores / playground |
| Datasets | `GET/POST /api/v1/datasets`, `…/examples?limit=&offset=` | Dataset + example management (paginated) |
| Datasets | `GET …/examples/{id}/trajectory` | Pinned trajectory viewer (steps · agents · tools · MCP · media) |
| Datasets | `POST …/{id}/runs`, `GET …/runs/{id}`, `POST …/{id}/agents` | Batch agentic/security/metrics runs · Connect Agents |
| Datasets | `GET …/runs/{id}/compare?against=` | Run-vs-run regression comparison |
| JudgePrompts | `GET/PUT /api/v1/eval/judge-prompts[/{name}]`, `POST …/reset`, `GET …/versions`, `POST …/restore/{v}` | Org judge-prompt overrides |
| Traces | `POST /api/v1/traces/{id}/annotations` | AnnotateBar (team thumbs + note) |
| Traces | `POST /api/v1/evaluate/trace-metrics` · `POST /api/v1/evaluate/agentic` | Single-run metric/custom-scorer eval · multi-run agentic eval (drawer Evaluation tab) |
| Prompts | `GET/POST /api/v1/prompts`, `PATCH/DELETE …/{id}` | Template CRUD |
| Prompts | `POST …/{id}/environments/{env}` · `GET …/{id}/versions` · `POST …/versions/{v}/restore` | Env deploy / versions |
| Prompts | `POST /api/v1/evaluate/compare` | Compare models across providers in the eval drawer (BYOK, per-model metric scoring) |
| Prompts/Datasets/Traces | `GET /api/v1/evaluate/models` · `GET /api/v1/credentials` · `POST /api/v1/credentials` | Table-driven model picker · BYOK key gate + add-key dialog |
| Audit | `GET /api/v1/audit` | Request audit log |
| ApiManagement | `GET/POST /api-keys`, `DELETE /api-keys/{key_id}` | Key management |

Public/marketing calls: `GET /api/v1/models` (cost calculator), `GET /api/v1/blog/*` (blog), `POST /api/v1/contact`, `POST /api/v1/models/request|report`.

**Pagination.** Dashboard lists use **windowed Previous/Next** pagination, not
infinite "Load more" append — one page in memory at a time, the list is replaced
per page. A shared `components/Pagination.tsx` (with a `compact` variant for
sidebars) renders the control; each screen keeps `page` + `reloadKey` state and a
fetch effect on `offset = page * PAGE_SIZE`. Applies to Agents, Tests, Prompts,
Audit, Security, and the Datasets examples list. On the SSE-backed Security list,
live prepend/refresh is gated to page 0 so paging through history isn't disrupted.

**Per-run roll-ups.** The Traces/Agents tables render a root row's headline cost /
quality / security / span-count from `POST /traces/rollups` (fetched once per
page), so a root shows rolled-up numbers — e.g. a run whose evals live on child
spans still shows its quality — without expanding it. Falls back to client-side
child summation only when a rollup row is absent.

---

## Guardrails / Security Types

```typescript
type RiskLevel = "low" | "medium" | "high"

interface GuardrailPolicy {
  block_threshold: "medium" | "high"
  warn_threshold: RiskLevel
  block_categories: string[]      // subset of ALL_CATEGORIES
  custom_deny_list: string[]
  custom_allow_list: string[]
  pii_ignore: string[]            // subset of PII_ENTITIES
  allowed_tools: string[]
  alert_webhook: string | null
  alert_on: RiskLevel[]
  scan_responses: boolean
  org_id: string                  // present on responses
  slug: string
}
```

`ALL_CATEGORIES`: `prompt_injection, jailbreak, skeleton_key, semantic_attack, pii_detected, secrets_detected, indirect_injection, rag_poisoning, tool_exfiltration, tool_policy_violation, cross_agent_injection`.
`PII_ENTITIES`: `US_SSN, CREDIT_CARD, IBAN_CODE, CRYPTO, US_PASSPORT, EMAIL_ADDRESS, PHONE_NUMBER, PERSON, LOCATION, IP_ADDRESS`.

> **No UI yet for `alert_webhook` / `alert_on`.** The Guardrails page (`screens/Dashboard/Guardrails/index.tsx`) loads and round-trips both fields (so a `PUT` preserves them) but renders no input for them, so today the custom alert webhook is settable only via `PUT /api/v1/guardrails`. The user-facing behavior is documented in `screens/Documentation/AlertsPage.tsx` ("Custom webhook"). TODO: add the webhook field to the Guardrails page.

---

## Other Shared Types

```typescript
interface QuotaCounter { used: number; limit: number | null }
interface QuotaResponse { tier: UserType; traces: QuotaCounter; evaluations: QuotaCounter }

interface EvaluationScore {
  metric: string; score: number | null
  evaluator: string; judge_model: string   // evaluator "fluiq.agent_eval" = agentic
  details?: {
    reason?: string; per_chunk?: { id: string; score: number; useful: boolean }[]
    // Agentic (fluiq.agent_eval) detail blocks, rendered inline in the drawer:
    per_call?: { tool: string; appropriate: boolean; reason: string }[]      // L2 tool selection
    deterministic?: { score: number; passed: boolean; findings: { code: string; severity: string; message: string; tool?: string }[] } // L1
    subgoals?: { subgoal: string; achieved: boolean }[]; goal_completion?: number; efficiency?: number // L3 trajectory
    panel?: { convened: boolean; agreement?: number; votes_pass?: number; votes_total?: number; members?: { role: string; provider: string; model: string; score: number }[] } // L4
    run_score?: number; run_passed?: boolean
  }
}

interface TraceRecord {
  api_key_prefix: string
  event: Record<string, unknown>
  ingested_at: string
  cost?: number; currency?: string
  evaluations?: EvaluationScore[]
  security?: { risk_level: string; attack_types: string[] }
}
interface TraceListResponse { traces: TraceRecord[]; limit: number; offset: number }

interface AgentRow {
  agent_key: string; agent_kind: string; integration: string
  runs: number; total_cost: number; avg_cost_per_run: number
  total_tokens: number; avg_latency: number | null; last_run: string | null
}

interface CacheKindStats { kind: string; hits: number; misses: number; calls: number; hit_rate: number }
interface CacheStatsResponse {
  window_hours: number; hits: number; misses: number; calls: number
  hit_rate: number; per_kind: CacheKindStats[]
}
interface PromptCacheStatsResponse {
  window_hours: number
  anthropic_cache_read_tokens: number
  anthropic_cache_creation_tokens: number
  provider_cached_tokens: number
  total_cached_tokens: number
  calls: number; calls_with_hit: number
}

interface Prompt {
  id: string; slug: string; name: string; template: string
  model: string | null; variables: string[]; kind: "completion" | "judge"
  version: number; environments: Record<"development"|"staging"|"production", number | null>
  created_at: string; updated_at: string | null
}

interface Dataset { id: string; name: string; example_count: number; created_at: string }
interface DatasetExample { id: string; trace_id: string; event: Record<string, unknown>; added_at: string }
```

---

## Server-Sent Events

### `authStream(path, options)` — `src/lib/sse.ts`

Opens an authenticated SSE connection; handles `401 → refresh → retry` transparently.

```typescript
authStream(path: string, options: {
  onEvent: (eventName: string, data: unknown) => void
  onOpen?: () => void
  onError?: (err: Error) => void
  events?: string[]
}): AbortController
```

Throws `FatalSseError` (extends `Error`) for `402 / 403 / 404 / 5xx` — not retried.

### `useRealtimeStream(options)` — `src/lib/useRealtimeStream.ts`

React hook wrapping `authStream`; opens on mount, tears down on unmount.

```typescript
useRealtimeStream(options: {
  path: string | null
  events?: string[]
  onEvent: (eventName: string, data: unknown) => void
}): { error: FatalSseError | null; clearError: () => void }
```

SSE event names: `ready`, `ping`, `trace`, `trace.started`, `trace.enriched`.

---

## Error Handling Patterns

- **Form errors** — `<Alert>` banners inside the form.
- **`authFetch` errors** — `ApiError.detail` surfaced in component state.
- **SSE fatal errors** — `FatalSseError` captured by `useRealtimeStream`, rendered with context-specific messaging (quota exceeded, auth failure, server error).
- **Token expiry** — handled silently by `authFetch` (single refresh before logout).
- **Quota exhausted (402)** — surfaces as an upgrade prompt; `/secure/check` returns it once the org has spent its monthly scan allowance, not because of its plan (Growth+ required), and `/optimize/profile` gates Free (Team+ required).
