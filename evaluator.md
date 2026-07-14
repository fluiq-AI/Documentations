# Fluiq Evaluator — Internal Architecture Reference

The **Evaluator** worker (`fluiq-workers/evaluator/`) is an async Kafka consumer
that scores traces for quality. It has no HTTP surface: the API publishes eval
jobs to the `evaluations` Kafka topic, the worker consumes them, runs one or
more evaluators, writes results to ClickHouse (`fluiq.evaluations`), and fans the
result back to the API as a `trace.enriched` event for live streaming.

There are **two families** of evaluation in one worker:

| Family | Trigger | Style | Question it answers |
|--------|---------|-------|---------------------|
| **Single-shot** | `fluiq.eval()`, playground, auto-retrieval | 1–3 judge calls per metric | *Is this answer good?* (hallucination, relevance, RAG quality, toxicity…) |
| **Agentic** | "Run Agentic Eval" button / `operation="agent_eval"` | Layered, multi-judge | *Did the agent use the right tools and accomplish the goal?* |

> **Rule of thumb:** `fluiq.eval()` stays **single-shot** (cheap, per-answer).
> Agentic evaluation is **opt-in** and runs on a **root trace** (heavier,
> per-run — it scores the whole run keyed by `root_trace_id`).
>
> **Evaluation is opt-in.** `instrument()` alone only traces — it does **not**
> evaluate. Scoring runs only when the SDK opts in via `fluiq.eval()` (which
> sends an `_eval` flag + `_eval_config`) or when an evaluation is triggered from
> the dashboard. The API gates every eval job on `eval_enabled = _eval or eval_config`;
> the old ambient auto-eval sample rate has been removed. So a trace with no
> `fluiq.eval()` and no dashboard action produces **no** `fluiq.evaluations` rows.

---

## 1. How it runs

```
Kafka topic "evaluations"  ->  app.consume()  ->  dispatch(message)  ->  handler
```

`app.py` owns the consumer loop. Every message is a JSON dict. `dispatch()`
picks a handler by the `operation` field (or infers one from the message shape):

| `operation` | Handler (`jobs/run.py`) | What it does |
|-------------|------------------------|--------------|
| `evaluate` | `run_evaluation` | Explicit single metric or the RAGAS bundle |
| `sdk_llm` | `auto_llm_eval` | **`fluiq.eval()`** — scores an LLM answer with the configured metrics |
| `auto` | `auto_evaluate_retrieval` | Auto-scores a vectorstore retrieval with Context Precision |
| `playground_eval` | `playground_eval` | Dashboard playground; replies to the API over Kafka |
| `agent_eval` | `agent_evaluate` | **Agentic evaluation** (the layered pipeline below) |

When `operation` is absent, `dispatch()` infers it: an `evaluator` field →
`evaluate`; an `eval_config` → `sdk_llm`; a trace carrying tool/MCP activity →
`agent_eval` (via `_looks_agentic()`); otherwise → `auto`.

All blocking work (judge HTTP calls) is pushed to a **single** worker thread
(`ThreadPoolExecutor(max_workers=1)`). This is deliberate — it keeps memory flat
on small instances. Judge calls are network-bound, so throughput is fine.

---

## 2. The LLM-as-Judge core

Every metric that needs a model uses `LLMJudge` (`jobs/helper/judge.py`):

- **Pluggable providers:** `openai`, `anthropic`, `gemini`, `fluiq`. Selected by
  `EVAL_JUDGE_PROVIDER` / `EVAL_JUDGE_MODEL`.
- **JSON-only output:** every judge call forces a single JSON object; a tolerant
  parser (`_parse_json_object`) recovers if the model wraps it in prose.
- **Response cache:** `PromptCache` + `InMemoryCache` — an LRU keyed by
  `sha256(model + prompt + params)`. Identical judge prompts (very common across
  similar traces) are served from cache. This is the main cost lever.
- **Admin-editable prompts:** all judge prompts live in a registry
  (`jobs/helper/judge_prompts.py`). Defaults are seeded into Postgres
  (`eval_judge_prompts`) and can be edited from **Admin → Judge Prompts** without
  a redeploy. Everything **fails open**: a missing/broken override falls back to
  the built-in default, so evaluations never break because of this table.

Each evaluator subclasses `BaseEvaluator` and returns an `EvalResult`
(`{name, score in [0,1], passed, reason, details}`). `passed = score >= threshold`.

---

## 3. Single-shot metrics

These score a single `(question, answer, [context])` triple. Used by
`fluiq.eval()`, the playground, and auto-retrieval.

| Metric | File | Method |
|--------|------|--------|
| **Hallucination** | `hallucination.py` | Extract atomic claims -> verify each against context/reference. No-context fallback uses general knowledge. |
| **Faithfulness** (RAGAS) | `ragas.py` | Decompose answer into statements -> check entailment by context. |
| **Answer Relevancy** (RAGAS) | `ragas.py` | Judge how directly the answer addresses the question. |
| **Context Precision** (RAGAS) | `ragas.py` | Per-chunk usefulness -> rank-weighted (MAP-style) score. |
| **Context Recall** (RAGAS) | `ragas.py` | Which reference statements are covered by the context. |
| **Toxicity** | `ragas.py` | Detects toxic/unsafe content; reported as a safety score. |
| **Coherence** | `ragas.py` | Logical structure / readability. |
| **Custom judges** | `custom_judge.py` | Customer-authored judge prompt referenced by slug in `fluiq.eval(custom_judges=...)`. |

**`fluiq.eval()` path (`auto_llm_eval`):** pulls `metrics` + `thresholds` from
the SDK's `eval_config`, extracts the latest user message + the answer, and runs
each metric as a single-shot judge. One ClickHouse row per metric.

### Vision-grounded evaluation (`jobs/helper/vision.py`)

`vision_faithfulness` (`VisionFaithfulness`) grades whether the answer faithfully
describes the **image(s)** the call was about, using a **multimodal judge**
(`LLMJudge.judge_multimodal_json` → OpenAI / Anthropic / Gemini vision). It pulls
media from the trace via the SDK's payload-free `_media_ref`s, so there's an
important caveat: a ref is only *usable* when it carries a **URL**
(`source="url"`). **base64** media isn't stored in the trace (only its hash), so
it can only be judged when the caller passes the image inline (`media=[…]`); a
run with only base64 refs returns `applicable: false`. Vision runs on the
**worker** (warn-mode `sdk_llm`) — not the block-mode `/evaluate` path, which is
text-only. Add it via `fluiq.eval(metrics=["vision_faithfulness"])`.

---

## 4. Agentic evaluation — the layered pipeline

Single-shot metrics can't answer *"did the agent call the right tool with the
right arguments, and did the whole run accomplish the goal?"* That needs the
**agentic** pipeline (`jobs/agentic/`). It runs in four layers, cheapest first.

### The big picture

```
                         +-----------------------------------------------+
   any trace source      |            AGENTIC EVALUATOR                  |
   -----------------      |                                              |
   - Fluiq SDK envelope   |   L0  NORMALIZE   adapters.py                |
     (OpenAI / Anthropic  |   +--------------------------------------+   |
      / Gemini / MCP /    |   |  trace  ->  AgentRun                 |   |
      LangGraph / CrewAI  |   |            - goal                    |   |
      / Google ADK)       |   |            - available_tools(schema) |   |
   - OpenInference/OTel --+-->|            - steps[] -> tool_calls[] |   |
     (Phoenix, Arize,     |   |            - final_output            |   |
      LangSmith, Langfuse)|   +------------------+-------------------+   |
   - Raw JSON             |                      |                       |
                          |                      v                       |
                          |   L1  DETERMINISTIC   deterministic.py       |
                          |       (NO LLM, free, runs first)             |
                          |   +--------------------------------------+   |
                          |   | - tool in allowlist?                 |   |
                          |   | - args match JSON schema?            |   |
                          |   |   (type / enum / required / unknown) |   |
                          |   | - tool errored?                      |   |
                          |   | - duplicate / looping call?          |   |
                          |   +------------------+-------------------+   |
                          |                      | findings + det score  |
                          |                      v                       |
                          |   L2  TOOL SELECTION  tool_selection.py      |
                          |       (1 judge call)                         |
                          |   +--------------------------------------+   |
                          |   | "Right tool? Right arguments for the |   |
                          |   |  goal?" -> per-call verdict + score  |   |
                          |   |  (blended 70% judge / 30% det)       |   |
                          |   +------------------+-------------------+   |
                          |                      v                       |
                          |   L3  TRAJECTORY  trajectory.py              |
                          |       (1 judge call, depth >= standard)      |
                          |   +--------------------------------------+   |
                          |   | Decompose goal -> sub-goals; which   |   |
                          |   | did the run achieve? + efficiency    |   |
                          |   +------------------+-------------------+   |
                          |                      v                       |
                          |   L4  MULTI-AGENT PANEL  panel.py            |
                          |       (depth = deep)                         |
                          |   +--------------------------------------+   |
                          |   |  Jury of N judges vote on L2 & L3.   |   |
                          |   |  GATED: primary runs first; jurors   |   |
                          |   |  convene only when the score is near |   |
                          |   |  the threshold (low confidence).     |   |
                          |   +------------------+-------------------+   |
                          |                      v                       |
                          |   ORCHESTRATOR  orchestrator.py              |
                          |   run_score (min of layers) + run_passed     |
                          +----------------------+------------------------+
                                                 v
                          ClickHouse fluiq.evaluations + trace.enriched -> API (SSE)
```

### Layer 0 — Normalize (`adapters.py`, `schema.py`)

Every trace source is mapped to one shape, `AgentRun`:

```
AgentRun
 |- goal              the RUN-level objective, structure-aware: the root event's
 |                    own input (LangGraph invoke state / @trace root args) →
 |                    else the composed task plan in execution order (CrewAI,
 |                    whose root has no input) → else the first user message.
 |                    (The old first-user-message-only heuristic anchored on the
 |                    first sub-agent's task prompt on multi-agent runs, making
 |                    the trajectory judge fail every such run.)
 |- final_output      the agent's final answer
 |- available_tools[] ToolSpec = { name, description, parameters(JSON schema), kind }
 |- steps[]           AgentStep = { order, type, tool_calls[] }
                        tool_calls[] -> ToolCall = { name, arguments, kind, server, result, error }
```

Adapters:
- `from_fluiq` — the SDK envelope. Understands OpenAI `tool_calls`, Anthropic
  `tool_uses`, Gemini `function_calls`, MCP `mcp_calls`, and reads the declared
  `tools` list as the **allowlist + argument schema**. Also reads `parent_ids`
  — the **multiple parents of a DAG join** node, emitted by the SDK for LangGraph
  (`langgraph_triggers`), CrewAI (task `context` dependencies), Google ADK
  (instruction `{state_key}` reads of upstream `output_key`s), and custom / A2A
  code (`fluiq.join_parents(...)`) — so fan-in structure survives into the graph
  (see Layer 3 / `graph.py`). Falls back to the single `parent_id` when absent.
- `from_openinference` — OTel/OpenInference GenAI spans (bring-your-own
  observability: Phoenix, Arize, LangSmith, Langfuse).
- `from_raw` — an already-normalized dict (offline/CI).

`normalize(message)` routes by `source`/shape. **Once normalized, nothing
downstream cares where the trace came from.**

**Bring-your-own-observability ingress.** External platforms send data via
`POST /api/v1/ingest/otel` (`routes/otel/` on the API), which converts
OpenInference/OTLP spans into Fluiq events and runs them through the normal
persist + eval + security pipeline: **Phoenix / Langfuse** push their OTel export
to the endpoint; **LangSmith / Braintrust** are pulled by the connectors in
`fluiq-api/connectors/`. The security worker also has its own OpenInference
fallback (`security/jobs/helper/openinference.py`) for raw spans on any path.

### Layer 1 — Deterministic checks (`deterministic.py`) — no LLM, free

Runs first and catches the large class of failures that don't need a judge:

| Check | Severity |
|-------|----------|
| `unknown_tool` — not in the declared toolset | error |
| `missing_required_arg` — required schema field absent | error |
| `arg_type_mismatch` / `arg_not_in_enum` | error |
| `unexpected_arg` (schema forbids) / `unknown_arg` | error / warning |
| `tool_execution_error` — the call returned an error | error |
| `duplicate_call` — identical call repeated (possible loop) | warning |

Produces a `DeterministicReport` with per-call findings and a `score`
(1.0 = every call clean). A dependency-free JSON-Schema-lite validator does the
argument checking, so there's no extra library and no cost.

Loop detection is **DAG-aware** (`graph.py`): an identical repeated call is only
flagged when the two calls lie on the **same path** (one is an ancestor of the
other). Identical calls in sibling *parallel* branches are legitimate fan-out and
are not flagged.

### Layer 2 — Tool Selection Quality (`tool_selection.py`) — 1 judge call

Given the goal, the allowed tools, and the calls made, the judge rates each call
*appropriate?* and returns an overall score. Layer-1 findings are handed to the
judge so it doesn't re-derive schema problems. The final score is blended
**70% judge / 30% deterministic** so a schema-invalid call can't be rated
"perfect" by a lenient judge.

### Layer 3 — Trajectory (`trajectory.py`) — 1 judge call (depth >= standard)

Steps back to the whole run: decompose the goal into sub-goals, decide which the
trajectory achieved (goal completion), and rate efficiency (penalize redundant /
looping steps). Completion is weighted above efficiency. The trajectory is
rendered in **topological order** from the run's DAG (`graph.py`) with
`[FAN-OUT]` / `[JOIN <- parents]` markers, so the judge can reason about parallel
branches and whether a join actually combined them; a `structure` summary
(joins / fan-outs / `is_dag`) rides along in the result details.

### Layer 4 — Multi-agent panel (`panel.py`) — depth = deep

A **jury**: run a metric through several judges and aggregate (mean score,
majority pass, an **agreement** score from the spread). Two things make it
affordable:

- **Confidence gating** — the primary judge runs first; the rest of the jury is
  convened **only** when the primary's score is near the threshold
  (`|score - threshold| <= gate_margin`). On confident cases you pay for one
  judge, not the whole panel.
- **Eval-trace** — every member's score + reason is stored in `details.panel`,
  so a panel score is auditable, not a black box.

Modes (`EVAL_PANEL_MODE`): `off` (single judge) · `gated` (default, recommended)
· `always` (full panel every metric — max quality, max cost).

### Layer 5 — Multi-agent coordination (`coordination.py`) — multi-agent runs only

Only fires on a genuine multi-agent run (≥2 agent groups **or** any join node).
It groups steps by their agent (LangGraph node / role / function → a per-agent
breakdown of steps, tool calls, and errors) and, for **each join / fan-in node**,
asks the question unique to multi-agent DAGs: *did the aggregating agent actually
incorporate every incoming branch, or silently drop one?* This is the payoff of
the `parent_ids` fan-in capture — a synthesizer that ignores a branch is a
coordination failure L1–L3 can't see. Metric `agentic.coordination`; the judge
only runs on join nodes, so single-agent and linear runs cost nothing here.

### Depth tiers

| Depth (`EVAL_AGENT_DEPTH` or per-message `depth`) | Layers run | Use for |
|------|------------|---------|
| `fast` | L1 + L2 | high-volume streaming |
| `standard` (default) | L1 + L2 + L3 (+ **L5** if multi-agent) | normal online eval |
| `deep` | L1 + L2 + L3 (+ L5) + **L4 panel** | CI / audits / disputed runs |

### Orchestrator (`orchestrator.py`)

Composes the layers, maps each metric to its ClickHouse `layer`, and computes the
**run verdict**: `run_score = min(all layer scores)` and
`run_passed = deterministic passes AND every judged metric clears threshold`.

---

## 5. Persistence

Every result becomes a row in `fluiq.evaluations`:

| Column | Meaning |
|--------|---------|
| `organization_id`, `trace_id`, `root_trace_id` | scoping |
| `evaluator` | e.g. `fluiq.eval`, `fluiq.agent_eval` |
| `metric` | e.g. `hallucination`, `agentic.tool_selection_quality` |
| `score` | this metric's score in [0,1] |
| `judge_model` | model used |
| `details` | full JSON (per-call verdicts, deterministic findings, panel eval-trace) |
| `layer` | agentic only: `deterministic` / `tool_selection` / `trajectory` / `coordination` |
| `step_id` | agentic per-step granularity (reserved) |
| `run_score`, `run_passed` | agentic only: the run-level verdict |

Non-agentic rows leave `layer`/`run_*` at defaults; agentic dashboards filter on
`evaluator = 'fluiq.agent_eval'`. The same result is also published as a
`trace.enriched` event so the API can stream it into the live trace view.

---

## 5b. Calibration — is the judge actually right? (`jobs/calibration/`)

Once evaluation has moving parts (decomposition, juries, panels), you have to
prove the judges agree with humans. The calibration harness measures that against
a held-out labelled corpus:

- **Golden set** — `jobs/calibration/golden/*.json`: human-labelled cases.
  Two kinds: **single-shot** (`{metric, inputs, expected_pass}`) for the judge
  metrics, and **agentic** (`{metric: "agentic.…", kind: "agentic", trace: {events: […]}, expected_pass}`)
  where a whole labelled trace is normalized into an `AgentRun` and scored by the
  matching layer (tool-selection / trajectory / coordination / deterministic).
  Extend either by adding files.
- **Runner** — scores each case with the real judge and reports agreement per
  metric: **accuracy**, **Cohen's κ** (chance-corrected — a judge that always
  says "pass" scores high accuracy but κ≈0), precision/recall/F1, and score
  MAE / Pearson. `passed` requires overall accuracy to clear `min_accuracy`
  **and** every metric to be individually calibrated — a release gate.

```bash
python -m jobs.calibration.runner          # configured judge
python -m jobs.calibration.runner --stub   # offline smoke
```

Run it in CI to gate releases and to catch judge drift when the model updates.

**Growing the corpus from production** (`jobs/calibration/harvest.py`): production
traces have no labels, so it's a *harvest → label → promote* loop. `harvest`
pulls real traces from ClickHouse, groups them into runs, and classifies each
into **draft** cases (`expected_pass = null`) — a plain LLM answer → single-shot
hallucination/relevance drafts; a run with tool calls / multiple steps / joins →
agentic tool-selection / trajectory / coordination drafts. Drafts land in
`golden/candidates/` (never loaded — `load_golden` skips unlabelled cases); a
human sets `expected_pass` (and redacts anything sensitive), then `promote_labeled`
merges the labelled ones into the active corpus.

```bash
python -m jobs.calibration.harvest --limit 200 --hours 168 [--suggest] [--no-scrub]
```
`--suggest` attaches the current judge's verdict to each draft as a review hint
(never the label). **PII scrubbing is on by default** (`jobs/calibration/scrub.py`):
a dependency-free regex pass redacts emails, phones, SSNs, credit cards, IPs and
the same API-key/token patterns as the security worker; if Presidio is installed
it augments with high-sensitivity entities (IBAN, crypto, passport…). Person and
location names are deliberately **kept** — they're usually the case's content.
Each draft's `note` records what was scrubbed. Scrubbing is a safety net, not a
guarantee — the human still reviews before promoting.

---

## 6. Worker runtime & configuration

**Entry point** — `evaluator/app.py` consumes `KAFKA_EVAL_TOPIC` and routes on
`operation` (see §1). All blocking judge work is pinned to a **single** executor
thread so torch/spaCy use one glibc malloc arena (the main RSS-growth driver on
small instances). On startup it opens an optional Postgres pool, seeds the
judge-prompt defaults, and primes the prompt snapshot. Shared worker concerns
(running, Kafka topics, auth, producer, message flow) live in `docs/workers.md`.

**Admin-editable judge prompts** (`jobs/helper/judge_prompts.py`) — the LLM-as-Judge
prompt text is externalized so it can be reworked from **Admin → Judge Prompts**
without a worker redeploy. Canonical defaults live in `_PROMPTS` (15 entries) and
render via `judge_prompts.render(name, **vars)`.
- **Syntax** — `string.Template` (`$var` / `${var}`); literal JSON braces need no escaping.
- **Flow** — `app.dispatch()` calls `refresh()` before each message (TTL-gated by
  `EVAL_JUDGE_PROMPT_TTL`), pulling overrides from Postgres `eval_judge_prompts`
  into an in-memory snapshot the synchronous `render()` reads (no DB on the hot path).
- **Fail-open** — missing `POSTGRES_DSN`, unreachable PG, an override missing a
  required `$var`, or an unknown name all fall back to the built-in default.
- **Seeding** — the **API** is the authoritative seeder on startup
  (`db_queues/postgresql/judge_prompt_defaults.py`, a byte-identical mirror of
  `_PROMPTS`); the worker also seeds `ON CONFLICT DO NOTHING` as a fallback.

| Env var | Default | Description |
|---------|---------|-------------|
| `KAFKA_EVAL_TOPIC` / `KAFKA_EVAL_GROUP_ID` | `evaluations` / `evaluator-workers` | Input topic + consumer group |
| `KAFKA_PLAYGROUND_REPLY_TOPIC` | — | Playground request-reply topic |
| `EVAL_JUDGE_PROVIDER` / `EVAL_JUDGE_MODEL` | `openai` / provider default | Judge provider + model |
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GEMINI_API_KEY` | — | Required for the matching judge provider (and vision judging) |
| `EVAL_JUDGE_THRESHOLD` | `0.7` | Global pass threshold |
| `EVAL_JUDGE_CACHE` / `_TTL` / `_MAX` | `1` / `3600` / `2048` | Judge response cache |
| `EVAL_JUDGE_PROMPT_TTL` | `60` | Seconds a judge-prompt snapshot is trusted before refresh |
| `EVAL_AGENT_DEPTH` | `standard` | Agentic depth tier |
| `EVAL_PANEL_MODE` | `gated` | `off` / `gated` / `always` |
| `EVAL_PANEL_GATE_MARGIN` | `0.12` | How near threshold triggers the jury |
| `EVAL_PANEL_MEMBERS` | — | Jurors as `provider:model,provider:model` |
| `EVAL_AUTO_SAMPLE_RATE` | `1.0` | (API-side) fraction of untagged LLM calls that get an ambient eval |
| `CLICKHOUSE_*` incl. `CLICKHOUSE_EVALUATIONS_TABLE` | — | ClickHouse connection + tables |
| `POSTGRES_DSN` | — | Admin-editable judge prompts + warn-mode custom judges (optional; fails open) |

---

## 7. When does each run? (the routing contract)

```
fluiq.eval()  ------------->  single-shot metrics        (per opted-in LLM answer)
"Run Agentic Eval" button ->  agent_eval -> L1..L4        (per ROOT trace, opt-in)
Dataset batch run --------->  agent_eval / security       (per example, over pinned trajectories)
```

- **`fluiq.eval()` is always single-shot** and **opt-in.** It is the cheap,
  per-answer path; `instrument()` alone never triggers it (see the opt-in note in
  §Overview). Once opted in, every LLM call in the process gets an evaluation
  number so the trace view can show a per-call score.
- **Dataset batch evaluation.** A Dataset can run agentic eval (or security) over
  every example. Trace-backed examples carry the run's **whole trajectory**,
  pinned into a no-TTL ClickHouse store (`dataset_trajectory_spans`) with media
  offloaded to S3 when they were added. The dataset run feeds those pinned
  span events straight into the same Layer-0 normalizer and layered pipeline, so
  agentic scoring works offline and is independent of the source trace's
  retention window. See `docs/api.md` (Datasets) and `docs/backend.md`.
- **Agentic evaluation is explicit and root-only.** Agentic eval scores a *whole
  run* — the root span plus its tool/MCP subtree, keyed by `root_trace_id` — so
  the **Run Agentic Eval** button appears in the drawer's **Evaluation** tab
  **only when the open trace is a root** (it is its own root, has no
  `root_trace_id`, or is the visible root of its group). It is intentionally
  hidden on child spans and on a selected tool, because a single span has no run
  to evaluate. On click, the API fetches every span sharing that
  `root_trace_id`, publishes one `agent_eval` job, and the layered pipeline runs;
  results stream back over the SSE `trace.enriched` channel.
