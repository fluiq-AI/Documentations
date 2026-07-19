# Fluiq SDK Reference (Python)

```
pip install fluiq
```

Requires Python ≥ 3.9.

The Python SDK (`fluiq-sdk/`) is a **thin, fail-open instrumentation client**. It auto-patches your LLM/agent/vectorstore libraries to emit traces, and exposes a few opt-in feature toggles (`optimize`, `eval`, `secure`). The heavy lifting — LLM-as-judge evaluation, security scanning, cache profiling — happens **server-side** in `fluiq-api` + `fluiq-workers`; the SDK only sends events and, for block-mode features, makes one synchronous call. Every instrumentation path is wrapped so a Fluiq failure never breaks your app (no exceptions, no console noise) — the only exceptions it ever raises are the intentional `FluiqSecurityError` / `FluiqEvalError` block-mode guards.

> A TypeScript SDK (`fluiq-sdk-typescript/`, published as `@fluiq/sdk`) mirrors this surface.

---

## Initialization

### `fluiq.instrument(api_key, *, endpoint=..., version="v1")`

Call once at application startup. Auto-patches every supported library that is installed; missing libraries are silently skipped.

```python
import fluiq

fluiq.instrument(api_key="flq_abc123...")
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `api_key` | `str` | `FLUIQ_API_KEY` env | Fluiq API key (sent as `Authorization: Bearer`) |
| `endpoint` | `str` (kw-only) | `FLUIQ_API_ENDPOINT` env or `https://api.getfluiq.com/api` | Ingest base URL |
| `version` | `str` (kw-only) | `"v1"` | Trace schema version — pin in production |

Patched on `instrument()`: **OpenAI** (chat/responses/parse/streaming, embeddings, images, audio — sync + async), **Anthropic** (messages + beta, sync + async), **Google Gemini / Vertex AI** (generate, streaming, count_tokens, embeddings), **LangChain**, **LangGraph**, **CrewAI**, **Google ADK**, vector stores (**Chromadb, Pinecone, Qdrant, Weaviate, FAISS**), **Voyage** embeddings, and **MCP** (`ClientSession.initialize` / `list_tools` / `call_tool`).

---

## `@trace` Decorator

### `fluiq.trace(func=None, *, name=None)`

Wraps a sync or async function to emit start + completion trace events. Nesting is automatic — inner traces resolve the correct `parent_id` via `contextvars`.

```python
from fluiq import trace

@trace
def run_pipeline(query: str) -> str: ...

@trace(name="custom_name")
async def async_step(x): ...
```

Emits a `status="running"` event at entry (for live UI) and a `status="success"`/`"error"` event on completion, capturing `latency`, `success`, and the exception string on failure. When `fluiq.optimize()` is active, decorated function results may be served from / written to the Fluiq cache (and the event is flagged `_cache_hit`).

---

## Feature Toggles

All three are opt-in, must be called **after** `instrument()`, and are no-ops until called.

### `fluiq.optimize(mode="cache")` — trace-driven Redis caching *(Team plan+)*

Fluiq analyses your historical traces to find frequently-repeated calls and provisions a dedicated Redis instance. On the first call after `optimize()`, the SDK fetches the cache profile from `/optimize/profile`, connects, and begins serving repeats from cache.

```python
fluiq.optimize()                 # "cache" — full caching
fluiq.optimize(mode="observe")   # measure would-be hits without serving cached responses
```

Covers **LLM responses** (matched on `model, messages, system, tools`), **MCP** `list_tools`/`call_tool` results, and **provider prefix caching** (see below). Invalid mode raises `ValueError`.

### `fluiq.eval(thresholds=None, metrics=None, mode="warn", judge_model="claude-haiku-4-5-20251001", custom_judges=None)`

Server-side LLM-as-judge evaluation after each LLM call.

| Parameter | Notes |
|-----------|-------|
| `thresholds` | per-metric pass bar, e.g. `{"hallucination": 0.8, "relevance": 0.7}` |
| `metrics` | defaults to `["hallucination", "relevance"]`. Supported: `hallucination, faithfulness, relevance, toxicity, coherence, completeness` |
| `mode` | `"warn"` (background, never interrupts) or `"block"` (synchronous; raises `FluiqEvalError` on a failing metric) |
| `judge_model` | judge model (default Claude Haiku 4.5) |
| `custom_judges` | client-defined judges as `{prompt_slug: threshold}` (TS: `customJudges`). Each slug must resolve to a `kind='judge'` prompt saved on the Prompts page |

In `warn` mode the eval runs entirely server-side (Kafka → evaluator worker). In `block` mode the SDK additionally calls `/evaluate` synchronously and raises if a metric fails its threshold. Invalid mode raises `ValueError`.

**Custom judges.** A client can save its own LLM-as-judge prompt on the Prompts page (type **Judge**, `prompts.kind = 'judge'`). The template uses `string.Template` `$question` / `$answer` / `$context` placeholders and should return `{"score": float 0-1, "reason": str}` (a JSON output contract is appended automatically if absent). Reference it by slug with a threshold: `custom_judges={"refund-policy": 0.9}`. Custom judges are embedded in the warn-mode `_eval_config` and sent in the block-mode `/evaluate` body alongside the built-in metrics; scores land in the dashboard under the slug as the metric. A slug that doesn't resolve to a saved judge is silently skipped (fail-open).

### `fluiq.feedback(value, trace_id=None, name="user_feedback", comment=None)`

Record end-user feedback for a trace — call it when your user reacts to an AI
response.

| Parameter | Notes |
|-----------|-------|
| `value` | `True`/`False` for thumbs up/down, or a `0..1` rating |
| `trace_id` | defaults to the most recent LLM call's trace in the current context (`current_llm_trace_id()`, falling back to the current span) |
| `name` | feedback channel shown in the dashboard, e.g. `"thumbs"`, `"csat"` |
| `comment` | free-text from the user |

Fire-and-forget `POST /api/v1/feedback` (Bearer API key): never raises, network
failures only log locally. Stored in the ClickHouse `evaluations` table as
`evaluator='human.feedback'` and rendered in the trace drawer next to the judge
scores; excluded from the automated quality rollup.

### `python -m fluiq.ci` — CI quality gate (`src/fluiq/ci.py`)

Runs a dataset eval as a build gate: launches a batch run over every example,
polls the report, prints per-metric averages, and exits non-zero on failure
(GitHub Actions `::error` annotations included).

```bash
python -m fluiq.ci --dataset "checkout-agent" --kind metrics \
    --metrics hallucination,relevance,completeness \
    --fail-below 0.7 --min-example 0.5
# env: FLUIQ_API_KEY (or --api-key), FLUIQ_API_ENDPOINT (optional)
```

| Flag | Notes |
|------|-------|
| `--dataset` | dataset name (case-insensitive) or UUID |
| `--kind` | `metrics` (grades answers vs expected outputs) or `agentic` |
| `--metrics` | comma list for `--kind metrics` (default `hallucination,relevance`) |
| `--depth` | `fast/standard/deep` for `--kind agentic` |
| `--fail-below` | fail when the run's average score is below this (default `0.7`) |
| `--min-example` | also fail when ANY example's average is below this |
| `--timeout` / `--poll` | polling budget (default 600s / 10s) |

Exit codes: `0` pass · `1` threshold failed · `2` error/timeout. Backed by the
API-key routes `POST/GET /api/v1/ci/eval-runs[/{run_id}]`.

### `fluiq.secure(mode="warn", *, guardrail="default")` *(Growth plan+)*

Server-side security scanning against the named guardrail policy.

| `mode` | Behaviour |
|--------|-----------|
| `"warn"` | post-call scan only; security fields written into the stored trace; LLM calls never interrupted |
| `"block"` | pre-call guard — every prompt is checked via `/secure/check` **before** the LLM call; raises `FluiqSecurityError` when the server returns `allow=False`. Post-call scanning still runs on allowed calls. |

`guardrail` selects a dashboard-configured policy slug (unknown slugs fall back to `default` server-side). Free/Team keys receive a `402` and silently fall back to warn behaviour. Invalid mode raises `ValueError`.

---

## Prompt Management

### `fluiq.fetch_prompt(slug, env="production") -> Prompt`

Fetches a deployed prompt template from the dashboard (`GET /prompts/fetch/{slug}?env=`).

```python
prompt = fluiq.fetch_prompt("support-reply", env="production")
text = prompt.render(ticket_body="...", customer_tier="gold")
```

`Prompt` (`src/fluiq/prompts/__init__.py`) attributes: `slug, name, template, model, variables, version, environment`; `.render(**vars)` substitutes `{variable}` placeholders (raises `KeyError` on a missing variable). `env` ∈ `production | staging | development`. Raises `requests.HTTPError` (404 not deployed to env, 401 bad key).

---

## Tool-result Cache Lookup

### `fluiq.lookup_tool_result(tool_name, args)`

Returns a cached tool result or `None`. `args` may be a dict or JSON string (keys are sorted before hashing, so order doesn't matter). Useful for short-circuiting expensive tool calls inside your own tool functions.

```python
result = fluiq.lookup_tool_result("get_weather", {"location": "London"})
if result is not None:
    return result
return call_weather_api("London")
```

Always returns `None` on any error (fail-open).

---

## Exceptions

```python
from fluiq import FluiqSecurityError, FluiqEvalError
```

| Exception | Raised when |
|-----------|-------------|
| `FluiqSecurityError` | `secure(mode="block")` and the server blocks the prompt. Carries `block_reason`, `risk_level`, `attack_types`. |
| `FluiqEvalError` | `eval(mode="block")` and a metric falls below its threshold. |

These are the **only** exceptions the SDK raises. All infrastructure/network failures are logged and swallowed (fail-open).

---

## Telemetry Model

### `LogTrace` — `src/fluiq/integrations/shared/models.py`

The Pydantic model used for every emitted event. `model_config = ConfigDict(extra="allow")`, so integrations attach provider-specific fields freely (e.g. `prompt_cached_tokens`, `mcp_calls`, security/eval enrichments).

```python
class Tokens(BaseModel):          # extra="allow"
    prompt: int | None
    completion: int | None
    total: int | None
    # Provider prompt-cache fields are attached as extras when present:
    #   prompt_cache_read_tokens     — Anthropic: served from prefix cache
    #   prompt_cache_creation_tokens — Anthropic: written to create the cache entry
    #   prompt_cached_tokens         — OpenAI / Gemini: served from provider cache

class TraceType(str, Enum):
    OpenAI = "OPENAI"
    Anthropic = "ANTHROPIC"
    Gemini = "GEMINI"
    LangChain = "LANGCHAIN"
    LangGraph = "LANGGRAPH"
    LlamaIndex = "LLAMAINDEX"
    CrewAI = "CREWAI"
    GoogleADK = "GOOGLEADK"
    AutoGen = "AUTOGEN"
    AgentToAgent = "AGENTTOAGENT"
    ChromaDB = "CHROMADB"
    Pinecone = "PINECONE"
    Qdrant = "QDRANT"
    Weaviate = "WEAVIATE"
    FAISS = "FAISS"
    General_Function = "OTHERFUNCTION"
```

Core `LogTrace` fields: `trace_id, parent_id, parent_ids, function, integration, type, timestamp, started_at, latency, model, input, output, messages/contents, response, system, tools, tool_calls/tool_uses, mcp_servers/mcp_calls/mcp_results, tokens, thinking, success, status, finish_reasons, stop_reason, error_traceback` — plus any extras.

`parent_ids` (optional list) carries the **multiple parents of a DAG join / fan-in node**, emitted **only on real joins** (≥2 parents) so single-parent nodes keep just `parent_id`. Four sources:

- **LangGraph** — modern LangGraph names a node's `langgraph_triggers` after the *destination* channel (e.g. `branch:to:synthesize`), not the source nodes, so trigger parsing alone can't see a fan-in. The integration instead captures the graph's declared edges at `StateGraph.compile()` and resolves a join node's parents from its static predecessors intersected with the nodes that actually ran this graph invocation — reliable regardless of the trigger encoding.
- **CrewAI** — a Task with ≥2 dependency tasks (`task.context`) is a join; the integration maps each dependency task to its execution run_id.
- **Google ADK** — an agent whose `instruction` reads ≥2 upstream `{state_key}` outputs (each written by an agent's `output_key`, e.g. the children of a `ParallelAgent`) is a join; the plugin maps each referenced key to the writer agent's run_id.
- **Generic / A2A / custom** — wrap the aggregating call in `with fluiq.join_parents(id_a, id_b): ...` (accepts varargs or a list). The next `@fluiq.trace` call emits those as its `parent_ids`. This is the seam for agent-to-agent frameworks that have no built-in integration.

**Full-DAG rendering — `predecessors`.** `parent_ids` only carries fan-*in* joins, so single-predecessor fan-*out* edges would be invisible. The integrations therefore also stamp a node's complete predecessor set: LangGraph writes node **names** to `event.langgraph.predecessors` (from the same static edge graph), while CrewAI and Google ADK write predecessor **run_ids** to a top-level `event.predecessors`. The dashboard draws the real DAG (fan-out, joins, loop-backs) from these; the evaluator uses `parent_ids` for its coordination metric. See `docs/evaluator.md`.

**Agent identity.** A run surfaces in the Agents view by a stable key: `function`/`name` for `@trace` and plain chains, the `langgraph_node` for a node, or — for a whole graph run whose container span is otherwise anonymous — `LangGraph(node_a, node_b, …)` (named from the compiled graph's nodes, mirroring CrewAI's `Crew(agent_1, …)`). The tracer denormalizes this onto each root at ingest so the Agents roll-up needs no read-time scan.

**Multimodal — media references.** The SDK never stores raw image/audio/video/document payloads (large, sensitive, over the Kafka ceiling), but it no longer *drops* them either. `_strip_media` (OpenAI / Anthropic / Gemini, via `integrations/shared/media.py:media_reference`) replaces each media content part with a payload-free `_media_ref` — `{kind, mime, source (url|base64|file), bytes, sha256}`. Text parts are kept verbatim; Anthropic `thinking`/`redacted_thinking` blocks are still dropped (captured separately). So a vision/audio call is traced as its text plus a compact reference, making the *presence* of media visible to the evaluator and security scanners (which remain text-only for now).

---

## Provider Integrations

After `instrument()`, the following are auto-patched — no application code changes required.

| Provider | Patched surface |
|----------|-----------------|
| OpenAI | `chat.completions.create`, responses API, `parse`, streaming, embeddings, images, audio (sync + async) |
| Anthropic | `messages.create` + beta, streaming (sync + async); `cache_control` injection when `optimize()` is on |
| Google Gemini / Vertex AI | `generate_content`, streaming, `count_tokens`, embeddings (sync + async) |
| LangChain | callback handler injected via configure hook — chains, tools, retrievers |
| LangGraph | auto-detected from LangChain metadata (`langgraph_node`, `langgraph_step`) |
| CrewAI | agent/task/crew execution |
| Google ADK | plugin injected into the `PluginManager` |
| Vector stores | `query` / `search` / `near_text` / `near_vector` / `hybrid` / `bm25` / `fetch_objects` on Chromadb, Pinecone, Qdrant, Weaviate, FAISS |
| Voyage | embeddings (sync + async) |
| MCP | `ClientSession.initialize` / `list_tools` / `call_tool` |

---

## Optimize Internals — provider prefix caching

When `fluiq.optimize()` is active, in addition to the Redis response/MCP cache the SDK enables provider-level prefix caching and surfaces cached token counts (all three feed the Optimize dashboard's **Prompt Caching** card via `/optimize/prompt-cache-stats`):

- **Anthropic** — injects `cache_control: {"type": "ephemeral"}` onto the system prompt and the last tool definition. Captures `usage.cache_read_input_tokens` (`prompt_cache_read_tokens`, billed ~10%) and `usage.cache_creation_input_tokens` (`prompt_cache_creation_tokens`, billed ~125%).
- **OpenAI** — automatic for prompts ≥ 1024 tokens. Captures `usage.prompt_tokens_details.cached_tokens` (`prompt_cached_tokens`).
- **Gemini** — user-managed `CachedContent`; the SDK injects nothing but captures `usage_metadata.cached_content_token_count` (`prompt_cached_tokens`).

The Redis cache primitives live in `src/fluiq/optimization/caching/` (`BaseCache`, `make_key`, `RedisCache`) and the cache client in `src/fluiq/optimization/client.py`. These are internal to `optimize()` — there is no public in-SDK caching/reranking/RAG toolkit; that functionality is now server-orchestrated.

---

## Configuration — `src/fluiq/config.py`

```python
ENDPOINT = os.getenv("FLUIQ_API_ENDPOINT", "https://api.getfluiq.com/api")
VERSION  = "v1"
# auth_headers() → {"Authorization": "Bearer <api_key>"}
```

`_config` holds the live state: `api_key, endpoint, version, enabled`, and the per-feature flags (`optimize`/`optimize_mode`, `eval`/`eval_mode`/`eval_metrics`/`eval_thresholds`/`eval_judge_model`/`eval_custom_judges`, `secure`/`secure_mode`/`secure_guardrail`). All SDK → API requests carry the API key as an `Authorization: Bearer` header.

---

## Context Variables

Trace context uses Python `contextvars` so nested calls resolve parent IDs automatically.

```python
from fluiq.integrations.shared.context import (
    push_trace_id,      # returns a token
    pop_trace_id,       # takes the token
    current_parent_id,  # active trace ID or None
)
```

Used internally by `@trace` and the provider patches; direct use is only needed when building custom integrations.
