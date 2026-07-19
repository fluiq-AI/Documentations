# Security Workflow Audit — 2026-07-18

Scope: the full `fluiq.secure()` path — SDK pre-call/response gate → API
`/secure/check` + guardrail storage → Kafka → security worker scanners →
ClickHouse `security_scans` + rollups → alert dispatcher → dashboard.

Method: static code audit (no exploitation against prod). The API-endpoint layer
got an independent second pass from a sub-agent; the SDK / worker / persistence
layers are from a single reviewer pass (the second-pass agents were cut off by a
session limit and should be re-run to confirm).

Severity legend: **Critical / High / Medium / Low**. Status: **FIXED** (this
change), **OPEN** (recommended, not yet done).

---

## Verified fine (no issue)

- **No dashboard XSS.** `SecurityPanel.tsx` / `Security/index.tsx` render every
  attacker-influenced field (injection/jailbreak/skeleton patterns, secret
  types, PII entities, redacted text) through plain JSX text interpolation.
  React escapes it; no `dangerouslySetInnerHTML`.
- **No SQL injection / no cross-tenant reads.** Guardrail CRUD is fully
  parameterized (only `_TABLE` is interpolated, from config). Every ClickHouse
  security read — `fetch_traces`, the `security=flagged` subquery, `get_root_rollups`,
  `fetch_dataset_security_results` — is scoped by `organization_id`.
- **Alerts leak no sensitive content.** The Slack path sends category names +
  risk level only, never raw prompt / PII / secret values. Stable shared
  consumer group → each alert fires once.
- **No ReDoS** in the API fallback scanner (all patterns `re.escape`d literals)
  or the secret regexes (linear).
- **PII redaction** stores the anonymized text in `security_scans` (not raw).

---

## HIGH

### H1 — SSRF in the worker's image OCR fetch — FIXED
`fluiq-workers/security/jobs/helper/image_scan.py`
`_image_bytes` fetched any user-supplied image URL server-side with no
private-IP blocklist, `http://` allowed, and redirects followed — a blind SSRF
to `169.254.169.254` (metadata) / internal hosts.
**Fix:** added `_is_public_http_url` (resolves host, rejects
private/loopback/link-local/multicast/reserved), `allow_redirects=False`, and a
`status_code != 200` reject. Residual: DNS-rebinding window (documented).

### H2 — Allow-list bypassed the deny-list and all scanning — FIXED
`fluiq-api/routes/secure/__init__.py`
The allow-list ran first and matched on a case-insensitive **substring**, so
`"support: ignore all previous instructions…"` returned `clean`.
**Fix:** reordered to deny-list → high-risk pattern block → **then** allow-list.
The allow-list can now only short-circuit the softer worker/medium tier, never a
deny or high-risk hit. Proven: `scanners.check` flags the embedded injection HIGH
so it blocks before the allow-list is consulted.

### H3 — `alert_webhook` / Slack webhook SSRF — FIXED
`routes/guardrails/__init__.py`, `routes/secure/__init__.py`, `shared/slack.py`,
new `shared/net.py`.
Webhooks were POSTed server-side with no URL validation.
**Fix:** new `shared/net.is_safe_public_url` (https-only + public-IP resolution).
Validated at save (422 on unsafe) and re-validated at fire time; `follow_redirects=False`
pinned on both httpx clients.

### H4 — Worker timeout/crash fails open, dropping PII/secrets/semantic — PARTIALLY FIXED
`routes/secure/__init__.py`, `config.py`
On Kafka timeout the endpoint silently downgraded to pattern-only (no PII/secrets/
semantic) and still returned `allow=True`.
**Fix:** response now carries `degraded: true` on the fallback path so callers can
fail closed; new `SECURE_FAIL_CLOSED` env (default off, preserving documented
fail-open) blocks on degraded when set. **Still OPEN:** per-category fail-closed
policy and a degraded-mode metric/alert.

---

## MEDIUM

### M1 — Unbounded input → OOM / CPU DoS — FIXED
`routes/secure/__init__.py` caps `CheckRequest.prompt` (`max_length`, and
truncates the scanned/forwarded text to 64 KB). `jobs/helper/scanners.py` caps
every scanned field (prompt/response/tool_outputs/context_docs/tool_inputs) to
100 KB before Presidio/spaCy/the transformer (the historical OOM path).

### M2 — Image decompression bomb — FIXED
`image_scan.py` sets `Image.MAX_IMAGE_PIXELS` and rejects images over ~40 MP in
both OCR backends before decode.

### M3 — Reply-registration race forced needless degraded fallback — FIXED
`routes/secure/__init__.py` now registers the reply future **before** publishing
the Kafka job, so a fast worker reply is never dropped.

### M4 — Caller `context` overwrote blocked-trace fields (audit forging) — FIXED
`_publish_blocked_trace` now merges caller `context` first and lets the reserved
security fields (`status`/`block_reason`/`risk_level`/…) win.

### M5 — No Unicode normalization → detector evasion — FIXED
`base.py` gains `normalize_text` (NFKC + strip zero-width/format/control chars),
applied in `_scan_patterns` / `_scan_tiered` and in `semantic_score`; the API
fallback `_preprocess` normalizes too. NOT applied to PII/secrets (their
recognizers are offset/format sensitive). Proven: zero-width and full-width
"ignore all previous instructions" now detected on both API and worker sides.

### M6 — Async scanner crash is fail-silent (no row written) — FIXED
`jobs/run.py` now inserts a `security_scans` row with `extra.scan_failed = true`
when `run_scan` throws, so the gap is queryable instead of looking clean.

### M7 — Tool outputs scanned by regex only (no semantic) — FIXED
`scanners.py` now also runs `semantic_score` over each `tool_output` (same
threshold as RAG poisoning), catching paraphrased/obfuscated tool-returned
injections.

### M8 — Raw PII/secrets persist in `traces.event` plaintext — DEFERRED
The `traces.event` column is ClickHouse `JSON`; an `ALTER … UPDATE` assigning a
String is not reliably supported and a destructive mutation can't be validated
offline. Left for a dedicated, reviewed change. Safe options: (a) a separate
`event_redacted` column written by the worker, or (b) redact at ingest (needs a
lightweight redactor in the tracer). Not shipping untested destructive SQL.

---

## LOW

- **L1 — FIXED** — `secrets.py` + `pii.py` recognizers now match modern
  `sk-proj-`/`sk-svcacct-` OpenAI keys, `gh[posur]_` + `github_pat_` GitHub
  tokens, `ASIA/AGPA/AIDA` AWS keys, and Slack `xox*` tokens. Verified.
- **L2 — FIXED** — module docstring reconciled to "Growth tier or above".
  (Plan name in the 402 is the caller's own tier — left as useful UX, not a leak.)
- **L3 — FIXED** — the pre-call job now carries `org_id`, the worker echoes it,
  and the reply consumer rejects a reply whose `org_id` mismatches the waiting
  request (verification is optional, so the response-gate path is unchanged).
  Verified against the real consumer path.
- **L4 — OPEN (intentional tradeoff)** — `system`/`developer` roles are excluded
  by design to cut false positives; only a risk in multi-agent setups where a
  sub-agent's system prompt is built from untrusted upstream text. Left as-is.

---

---

## Second pass — independent re-audit (2026-07-18 evening)

Three independent agents re-audited the hardened SDK, worker, and persistence
layers. They CONFIRMED all first-round + first-open-round fixes are correct, and
found new issues. Status of the new ones:

### Fixed this pass
- **SDK-C1 (Critical) — TS LangChain block mode never fired.** `handleLLMStart` /
  `handleChatModelStart` (`langchain.ts:387,421`) called the async `preCallGuard`
  **unawaited** inside a sync `void` callback, so a `FluiqSecurityError` became an
  unhandled rejection and the LLM call proceeded. Made both callbacks `async` and
  `await` the guard, re-throwing a real block (fail open only on infra errors).
  tsc clean.
- **PERSIST-H1 (High) — `security='flagged'` not subtree-aware** (a regression in
  the earlier security-tab fix). Keyed on `t.trace_id`; agentic detections live on
  child spans (`trace_id != root`) while the page lists roots, so flagged runs
  were missing. Rekeyed on `t.root_trace_id` mirroring the blocked/failed branches
  (`queries.py`). Fast path retained; verified.
- **SDK-M2/M3 — TS `/secure/check` parity.** Added `authHeaders()` (key was
  body-only → proxy/APM leakage; the server does accept a body key so this was
  leakage, not a hard bypass) and now sends `trace_id` + `guardrail`. tsc clean.
- **WORKER-M2 — multimodal content-splitting evasion.** `_extract_scan_prompt`
  `str()`-ed list content, splitting a phrase across parts; now joins the text of
  each part (`run.py _content_to_text`).
- **WORKER-M3 — image OCR stalled the sync gates into fail-open.** All security
  work runs on one sequential consumer; inline image fetch (≤6×8s) blocked the
  pre-call/response gates. Moved OCR to a dedicated executor with a 12s wall-time
  budget (`run.py`). Residual: full isolation needs a dedicated sync consumer/topic.
- **WORKER-L6 — zero-width secrets in tool inputs.** Exfil detection now also
  scans a normalized copy of each `tool_input` (`scanners.py`).
- **WORKER-L7 — crescendo read the oldest 19 turns.** Now takes the most recent 19
  (`ORDER BY … DESC` + re-sort) so late-ramping attacks aren't diluted.
- **NET-L (partial) — IPv4-mapped IPv6 SSRF bypass.** `::ffff:169.254.169.254`
  now unwrapped and rejected in both validators (`shared/net.py`, `image_scan.py`).
  Verified.

### Fixed after re-audit (third batch)
- **PERSIST-M2 (Medium) — Security badge/score/high-risk count now run-level.**
  The Security page fetches per-run rollups (`POST /traces/rollups`) for the
  visible roots and derives the badge, the row score, and the "High risk" metric
  from `security_risk_max` / `security_should_block` (subtree-aware) instead of
  the root span's own scan. Also removed the client-side `hasSecurityData &&
  isSecurityRisk` re-filter — it inspected only the root event and would have
  dropped the child-flagged runs the server now returns; the page now trusts the
  authoritative server filter and only drops in-flight rows. Added a
  `levelOverride` prop + exported `levelFromScore` on `SecurityBadge`. tsc clean.
  Residual: the flags column and the detail drawer still read the root span's own
  scan, so per-category flags can be empty for a child-flagged run — the badge no
  longer lies, but drilling into the specific child detection is a follow-up.
- **PERSIST-M3 (Medium) — `security_scans` now has per-row retention + TTL.**
  Added `retention_days UInt16 DEFAULT 36500` + `TTL ingested_at +
  toIntervalDay(retention_days)` (mirrors `traces`, kept on DateTime64 to avoid
  the sentinel-overflow trap). The API already forwards `retention_days` on the
  security job; the worker now stamps it on both the scan row and the
  `scan_failed` marker (defaulting to the never-sentinel when absent, so nothing
  is dropped early). **Deploy order:** run the `ADD COLUMN` migration BEFORE the
  worker deploy (it inserts the column by name); `MODIFY TTL` after. Existing rows
  backfill to the never-sentinel, so no history is deleted.

### Still open (documented)
- **WORKER/NET residual (Low)** — DNS-rebinding TOCTOU: validate-then-connect
  re-resolves. Needs validated-IP pinning for full closure.
- **WORKER-M-cap (Low/Med)** — `_cap` truncation lets a trailing injection after
  100KB of filler evade; scan a tail window too.
- **WORKER-L5 (Low)** — cross-script confusables/leetspeak still evade NFKC; add a
  confusables fold (attack path only).
- **WORKER-L8 (Low)** — `scan_failed` rows store `security_risk_level='clean'`;
  add a first-class column/sentinel so failed scans aren't counted safe.
- **SDK-M4 (Medium)** — Python forwards full `messages`/`system`/`tools` as
  `context`; a big multimodal payload can exceed the 2s timeout → fail open. Bound
  the context.
- **SDK-M5 (Medium)** — OpenAI Responses API `input_text` parts not extracted
  cleanly by the SDK gate.
- **SDK-L6/L8 (Low)** — TS scans all roles vs Python `user`/`tool`; a 200 lacking a
  boolean `allow` is treated as allow with no signal.
- **M8 / L4** (from first round) — redaction-at-rest (schema-aware); system-role
  scanning for multi-agent.

## Follow-ups

1. Remaining open items are the Low residuals above plus SDK-M4/M5, M8, L4.
2. Deploy notes:
   - **PERSIST-M3 needs a ClickHouse migration**: run
     `ALTER TABLE fluiq.security_scans ADD COLUMN IF NOT EXISTS retention_days
     UInt16 DEFAULT 36500;` BEFORE deploying the security worker, then
     `ALTER TABLE fluiq.security_scans MODIFY TTL ingested_at +
     toIntervalDay(retention_days);` (both are in `schema.sql`).
   - Everything else is code-only. New this audit: `shared/net.py`,
     `SECURE_FAIL_CLOSED` env. The L3 org-echo and the TS client change span
     API + worker + TS SDK — all back-compat, any order.
3. Nothing is committed or deployed.
