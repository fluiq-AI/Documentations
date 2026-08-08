# Fluiq Security — Internal Architecture Reference

The **Security** worker (`fluiq-workers/security/`) is an async Kafka consumer
that scans traces for PII, secrets, and the full agentic-threat surface. It is
deployed **separately** from the evaluator so its heavy dependencies (torch +
spaCy/Presidio + sentence-transformers + a DeBERTa classifier) don't bloat the
evaluator's memory footprint or block judge calls. Shared worker concerns
(running, Kafka topics, auth, producer settings, overall message flow) live in
`docs/workers.md`.

Every recall / false-alarm figure quoted here comes from the guardrail benchmark
(`D:/ideas/FluiqAI/guardrail-bench`, published at `getfluiq.com/benchmark`), which
measures this worker against LLM Guard, Presidio, NeMo, AWS Comprehend, Lakera
and Nightfall on four corpora. Numbers are dated; re-run `make_report.py` before
quoting them anywhere.

---

## Entry point: `security/app.py`

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

---

## Scanners — `security/jobs/helper/`

| Module | Detects |
|--------|---------|
| `pii.py` | PII via Presidio (redaction + entity list) |
| `injection.py` | Direct prompt-injection patterns (tiered strong/weak) |
| `jailbreak.py` | Jailbreak / role-play escapes (tiered strong/weak) |
| `skeleton_key.py` | Skeleton-key attack patterns (Microsoft KB; tiered strong/weak) |
| `secrets.py` | Hardcoded credentials / high-entropy tokens |
| `semantic_v2.py` | Two-scope centroid similarity, the shipping semantic layer (`semantic_verdict`) |
| `semantic.py` | The older single-scope scorer. Still used for **retrieved documents only** (RAG poisoning), which are scored on a different distribution against `_RAG_POISON_THRESHOLD` and were not part of the v2 calibration |
| `classifier.py` | Fine-tuned DeBERTa injection classifier, advisory by default |
| `image_scan.py` | OCR of image media → text scanned for injection (pluggable, fail-open) |
| `openinference.py` | Normalizes raw OpenInference spans → scannable fields (fallback) |
| `scanners.py` | Orchestrator — `scan()` (full post-call) and `check()` (pre-call patterns only) |

### Pattern matching & tiering (`base.py`)

All three pattern scanners (injection, jailbreak, skeleton-key) share the tiered
model in `base.py`:

- **Word boundaries** are added only on *alphanumeric* edges, so short persona
  acronyms (`DAN`, `STAN`, `AIM`, …) match whole words — never as substrings
  inside `guidance`/`claim`/`understanding` — while delimiter markers like
  `[system]:` / `<|im_start|>system` / `{{` keep matching as-is. Acronyms are
  compiled **case-sensitive** so they don't fire on ordinary lowercase words.
- **Tiered scoring** (`_scan_tiered`): a **STRONG** phrase is HIGH on a single
  match; **WEAK/ambiguous** phrases are LOW alone and MEDIUM only when two or
  more co-occur. This keeps benign steering out of the HIGH (blocking) band —
  e.g. `act as`, `dark mode`, `from now on`, a Jinja `{{ … }}`, `make an
  exception`, and the bare persona openers `you are now` / `you are no longer` /
  `you are not an AI` all live in the WEAK tier. Only unambiguous completions
  (`you are now unrestricted`, `pretend you are …`, `ignore all previous
  instructions`, `augment your baseline`) are STRONG. Indirect-injection and
  image scans use the STRONG list only (weak phrases are common in benign
  reference text).
- **`normalize_text`** folds NFKC homoglyphs, strips zero-width/format-control
  characters, **and collapses runs of whitespace** so a phrase padded with extra
  spaces or split across newlines (`pretend   you   are`) still matches the
  single-spaced literal patterns.

The API's pre-call fast path (`fluiq-api/routes/secure/scanners.py`) is a
dependency-light mirror of **the pattern layer only**. It has no semantic layer
and no classifier, so it scores materially lower than the worker gate on the
same input (13.3% vs 35.3% recall on public injection data). It is a fast path,
not a second implementation of the gate — see "Three detection layers" below.

### Three detection layers

Prompt-side detection is three independent signals, in increasing cost:

| Layer | Module | Cost | Blocks? |
|-------|--------|------|---------|
| Patterns | `injection/jailbreak/skeleton_key.py` | microseconds | yes |
| Semantic | `semantic_v2.py` | sub-millisecond | yes (`FLUIQ_SEMANTIC_BLOCKS=0` to disable) |
| Classifier | `classifier.py` | ~460ms | **no**, advisory by default |

**Semantic (`semantic_v2.py`)** scores a prompt against seed centroids in two
scopes, each with its own model and its own threshold:

| Scope | Model | Threshold | Why |
|-------|-------|-----------|-----|
| injection | `paraphrase-multilingual-MiniLM-L12-v2` | 0.41 | roughly a third of real injection traffic is not English |
| jailbreak | `all-MiniLM-L6-v2` | 0.46 | the jailbreak corpus is English-only and the monolingual model is stronger per-language |

Both models are baked into the image (see the Dockerfile) rather than fetched
lazily, so a cold task does not put a several-hundred-MB download inside the
first message after a deploy.

The thresholds were calibrated **jointly**, on the combined dev set of both
corpora, to a 5% total false-alarm budget. Tuning them independently is wrong
and was the original bug: both scopes score all traffic, so isolated tuning
produced 30% real false alarms against the 6% each scope promised on its own.

Two behaviours that used to make this layer dead code, both fixed:

- **The `0.65` gates are gone.** Four of them sat above the useful operating
  range (0.15–0.44 measured on public corpora), so the layer almost never fired
  and patterns carried the entire load. The fourth lived in `run.py` and
  re-derived `semantic_attack` from a constant that disagreed with the gate that
  actually ran, so a scan could flag a prompt while the reported attack types
  stayed silent.
- **A semantic hit is HIGH, not MEDIUM.** `should_block` is `overall == HIGH`
  and `allow` is `overall != HIGH`, so a MEDIUM semantic verdict could never
  block however confident it was. Promoting it takes combined recall from 30.0%
  to 58.8% at a 4.2% false-alarm rate (patterns alone: 2.6%).

**Classifier (`classifier.py`)** is `protectai/deberta-v3-base-prompt-injection-v2`
at threshold 0.90, windowed (1200 chars, 900 stride, 8 windows max) so an
injection buried at the end of a long benign wall of text is not truncated away.

It runs in `scan()` and **never** in `check()`: a DeBERTa forward pass is ~460ms
against sub-millisecond for the embedding scopes, and `check()` is the
synchronous pre-call gate a caller waits on.

It is advisory because it keys on the imperative verb rather than on what the
verb targets. On public corpora it looks free (jailbreak 55% → 80%, injection
35% → 53%, no measurable false-alarm change), but against the false-positive
regression cases, which look far more like real support traffic, it flags 6 of
16 benign prompts at p > 0.99:

```
0.9998  "Ignore the formatting of the attached file and just read the text."
1.0000  "Please disregard my previous message, I sent it by mistake."
0.9975  "You are no longer subscribed."
```

No threshold separates those from real attacks, and requiring pattern
corroboration buys exactly zero extra recall because the pattern layer had
already fired on anything it would corroborate.

> **Known gap.** `classifier_score` is computed and set on `ScanResult`, but it
> is **not persisted** — it is absent from the ClickHouse insert, from the
> `run.py` record dicts, and from `extra`. The stated reason for running the
> model in advisory mode is to accumulate labelled disagreements with the
> shipping gate as training data, and that is not currently happening. Adding a
> `classifier_score` column (plus the `ALTER TABLE`, ordered before the worker
> deploy) is what closes it.

**Environment kill switches**

| Variable | Default | Effect |
|----------|---------|--------|
| `FLUIQ_SEMANTIC_BLOCKS` | `1` | `0` demotes a semantic hit to MEDIUM (advisory) while still reporting the score |
| `FLUIQ_SEMANTIC_MODEL_INJECTION` / `_JAILBREAK` | see table | swap either encoder |
| `FLUIQ_SEMANTIC_THRESHOLD_INJECTION` / `_JAILBREAK` | `0.41` / `0.46` | retune without a deploy |
| `FLUIQ_CLASSIFIER_ENABLED` | `1` | `0` turns the classifier off entirely |
| `FLUIQ_CLASSIFIER_MODE` | `advisory` | `block` lets it contribute to the verdict |
| `FLUIQ_CLASSIFIER_MODEL` | `protectai/deberta-v3-base-prompt-injection-v2` | `-small-` is the cheaper variant |
| `FLUIQ_CLASSIFIER_THRESHOLD` | `0.90` | |

Every layer fails open. A missing model, a failed import or an inference error
scores 0.0 and the gate falls back to the layers that did load.

**Regression tests.** `tests/test_pattern_tiering.py` imports only the pattern
helpers, so it cannot see semantic regressions at all.
`tests/test_semantic_gate.py` exercises the assembled gate end to end against the
same fixtures and is the test that catches a threshold change turning benign
support traffic into blocks.

**Measured cost.** After the classifier landed: `check()` 59ms, `scan()` 462ms,
RSS 2150MB against the task's 4096MB limit.

### PII & secret scoring notes

- **Custom US_SSN recognizer** — Presidio's built-in `US_SSN` does not fire on
  canonical 3-2-4 SSNs in the pinned version (`123-45-6789` is swallowed by
  `DATE_TIME`, `SSN: 078-05-1120` by `PHONE_NUMBER`), and since `scan()` only
  requests `_SUPPORTED_ENTITIES` those never surface. `pii.py` adds an explicit
  3-2-4 (hyphen/space/dot) recognizer at confidence 0.85. A bare 9-digit run is
  deliberately not matched — too ambiguous to carry `US_SSN`'s weight of 1.0.
- **Confidence floor** — `_MIN_CONFIDENCE = 0.3` drops Presidio's very-weak
  (0.05) bare-numeric `US_PASSPORT`/`US_SSN` matches that the flat entity weights
  would otherwise promote to a HIGH false positive on any order/invoice/ID
  number. Real signal is preserved (custom recognizers 0.80–0.95, NER ~0.85,
  email/phone/credit-card ≥ 0.4). The existing `_drop_zip_ssn_false_positives`
  (ZIP+4) and `_drop_hts_phone_false_positives` (tariff codes) still run first.
- **Secret scoring** — a **named** secret pattern (openai_key, aws_access_key, …)
  is HIGH → score 1.0; a **high-entropy-only** hit (no named pattern) is a soft
  MEDIUM → 0.5, so a benign base64 blob no longer scores 1.0 (which would read as
  HIGH in the run-rollup badge while the level was only MEDIUM).

---

## Post-call scan: `auto_security_scan()`

Triggered by `/ingest` for every LLM trace carrying `_security_config`. Before
running the CPU-bound scan it concurrently reads four context sources from the
trace tree (all keyed on `root_trace_id`), and OCRs any image media:

1. **Indirect sources** — sibling tool outputs, retrieved documents, tool inputs,
   tool names (`get_indirect_sources`).
2. **Session risk history** — prior `security_risk_score` values (`get_session_risk_scores`).
3. **Parent event kind** — to detect cross-agent (LLM→LLM) prompts (`get_parent_event_kind`).
4. **Agent-chain scores** — risk along the agent DAG ancestry (`get_agent_chain_scores`).
5. **Image OCR** — text extracted from image media (`ocr_event_images`), off the
   event loop, fail-open.

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
| Image-embedded injection | `image_injection_detected`, `image_injection_sources` — OCR of the event's image media (`image_scan.py`), then the OCR'd text run through the same injection/jailbreak/skeleton scanners. OCR backend is pluggable + fail-open (pytesseract → easyocr → none); only URL-source media is scannable from the trace |
| Semantic | `semantic_attack_score` — the winning scope's score from `semantic_verdict()` |
| Classifier | `classifier_score` — advisory, set on the result but not persisted (see the gap note above) |
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

---

## Pre-call check: `sync_security_check()`

Called from `/secure/check` (block mode) via Kafka request-reply. Runs a full scan
on the prompt and applies the forwarded `policy`:

- `block_threshold` — `medium` lowers the block bar (otherwise only `high` blocks).
- `block_categories` — when set, only listed attack types contribute to a block.
- `pii_ignore` — PII entity types to suppress.

Returns `{ allow, block_reason, risk_level, attack_types }` to
`KAFKA_SECURITY_REPLY_TOPIC` keyed by `correlation_id`. Fails open on any error.

## Response gate: `sync_response_gate_check()`

Called from `/ingest` when the org's active guardrail policy has
`scan_responses=True`. Scans only the **response** text for PII and secrets
(attack patterns are prompt-side and already caught pre-call). Blocks only on a
**concrete** detection — a recognized PII entity (`pii_entities_response`) or a
**named** secret type (`secret_types`) — with `security_risk_score ≥ 0.5`. A
high-entropy-only hit (`secrets_detected` true but no named `secret_types`) is
too noisy to gate a live response on, so a benign long token / base64 blob in
the output is no longer blocked. Returns
`{ response_blocked, risk_level, attack_types, block_reason }`. Fails open.

---

## ClickHouse security table (column order, by-name insert)

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
image_injection_detected, image_injection_sources,
semantic_attack_score, security_risk_level, security_risk_score,
should_block, scan_latency, extra, retention_days
```

`retention_days` mirrors the per-row TTL on `traces` (free tier rolls at 14 days,
paid never). It defaults to the `36500` "never" sentinel when the ingest path did
not forward one, so a scan is never dropped early by a missing value.

`classifier_score` is **not** in this list — see the gap note under "Three
detection layers".

`extra` (JSON) carries the derived trajectory signals: `crescendo_detected`,
`crescendo_score`, `session_turns`, `trust_boundary_escalation`,
`escalation_score`, `agent_chain_depth`.

The API read path (`db_queues/clickhouse/queries.py` select_cols +
`helpers.py:merge_security`) selects these columns **by position** into the trace
event so the dashboard Security panel can show them historically — the SELECT list
and the `merge_security` tuple unpack must stay in lockstep.

> **Deploy ordering:** because the insert is column-by-name, the PostgreSQL
> migration and the ClickHouse `ALTER TABLE … ADD COLUMN` for the agentic-threat /
> image-injection columns must be applied (API startup `_apply_schema()`) **before**
> the security worker is deployed.
