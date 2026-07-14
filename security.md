# Fluiq Security — Internal Architecture Reference

The **Security** worker (`fluiq-workers/security/`) is an async Kafka consumer
that scans traces for PII, secrets, and the full agentic-threat surface. It is
deployed **separately** from the evaluator so its heavy dependencies (torch +
spaCy/Presidio + sentence-transformers) don't bloat the evaluator's memory
footprint or block judge calls. Shared worker concerns (running, Kafka topics,
auth, producer settings, overall message flow) live in `docs/workers.md`.

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
| `injection.py` | Direct prompt-injection patterns |
| `jailbreak.py` | Jailbreak / role-play escapes (tiered strong/weak) |
| `skeleton_key.py` | Skeleton-key attack patterns (Microsoft KB) |
| `secrets.py` | Hardcoded credentials / high-entropy tokens |
| `semantic.py` | Cosine-similarity attack classifier (sentence-transformers) |
| `image_scan.py` | OCR of image media → text scanned for injection (pluggable, fail-open) |
| `openinference.py` | Normalizes raw OpenInference spans → scannable fields (fallback) |
| `scanners.py` | Orchestrator — `scan()` (full post-call) and `check()` (pre-call patterns only) |

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
(attack patterns are prompt-side and already caught pre-call). Blocks when an
attack type is present and `security_risk_score ≥ 0.5`. Returns
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
should_block, scan_latency, extra
```

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
