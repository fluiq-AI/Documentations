# FluiqAI Evaluation — Competitive Audit & Gap List

_2026-07-25. Compares Fluiq's evaluation product against LangSmith, Langfuse,
Braintrust, Arize Phoenix, Ragas, and DeepEval. Every "have" below was verified
against the code (`fluiq-workers/evaluator`, `fluiq-api/routes/evaluate`,
`fluiq-frontend`), not assumed._

---

## What Fluiq already has (verified)

| Area | Status |
|------|--------|
| Single-shot LLM-as-judge | hallucination, faithfulness, relevance, toxicity, coherence, completeness (in UI) |
| RAGAS suite (backend) | faithfulness, answer_relevancy, **context_precision**, **context_recall**, toxicity, coherence, completeness (`jobs/helper/ragas.py`) |
| Multimodal eval | `vision_faithfulness`, `media_faithfulness` |
| Agentic evaluator | L0 normalize → L1 deterministic → L2 tool-selection → L3 trajectory → L4 multi-agent panel/jury; depth tiers fast/standard/deep; DAG coordination |
| Custom client judges | `kind='judge'` prompts referenced by slug |
| Judge-prompt transparency | exact rendered prompt + version/source shown per score; org + admin editable with version history |
| BYOK judges | per-org, multi-provider (Anthropic/OpenAI/Gemini/Moonshot) |
| Datasets | batch runs (agentic/security/metrics), run-vs-run regression compare, Connect Agents (auto-append), pinned trajectory snapshots |
| Human signal | `fluiq.feedback()` + team annotations (stored, rendered, excluded from rollup) |
| CI gate | `python -m fluiq.ci`, `/ci/eval-runs` |
| Playground | multi-provider model compare + per-model metric scoring (new) |
| Traces eval | single-run metrics + multi-run agentic, in the drawer (new) |
| Operational | judge-token accounting, opt-in eval, SSE streaming, Slack alerts |
| Judge calibration | **internal only** — golden corpus + agreement (accuracy, Cohen's κ, MAE) in `jobs/calibration/`; a CLI/release-gating tool, not wired into the live worker or the dashboard |

This is a strong, broad base — the agentic layers, judge-prompt transparency, and
BYOK are genuinely ahead of most competitors.

---

## Gaps — prioritized

### Tier 1 — competitors have it, we don't, and it matters

1. **Online / production auto-evaluation (sampling rules).**
   No "sample X% of live traffic matching filter Y → auto-run judge Z" config.
   Eval is per-call opt-in (SDK `fluiq.eval()`) or a manual click in the
   dashboard; the old ambient auto-sample was deliberately removed. LangSmith,
   Langfuse, Phoenix, and Braintrust all offer continuous online eval on
   production. This is the single most visible gap for "monitor quality in prod."

2. **Pairwise / preference (A/B) evaluation.** _(shipped 2026-07-25)_
   ~~We run models in parallel and score each independently, but there is no
   head-to-head "which answer is better" judge.~~ **Done:** `POST /evaluate/pairwise`
   (BYOK) — a judge ranks the compared outputs and picks a winner; for exactly two
   candidates it runs both orderings and only declares a winner if they agree,
   cancelling position bias (else "too close to call"). Wired into the Prompts
   compare drawer as **"🏆 Pick the winner"** with a winner badge + verdict banner.
   Verified end-to-end (clear winner on good-vs-bad; correct tie on two strong
   answers). Still missing: `answer_correctness` / semantic-similarity vs a
   ground-truth reference (Ragas has it).

3. **Close the human → judge loop.**
   Calibration exists (`jobs/calibration/`, κ/MAE vs a golden set) but: (a) it's
   not surfaced to customers as a "how accurate is my judge" view; (b) the
   dashboard's own annotations/feedback are **not** fed into the harvest→label→
   promote loop (labelling is hand-edited JSON files); (c) no few-shot
   auto-correction of judges from human labels. LangSmith "corrections" and
   Braintrust human-review-improves-scorer both close this loop.

4. **Surface the RAG retrieval metrics we already built.** _(partially shipped
   2026-07-25)_
   `context_precision` and `context_recall` are implemented in the worker
   (`ragas.py`) but were **not selectable in any UI metric picker**.
   - **Done:** `context_precision` is now offered in the **Traces** and **Prompts**
     eval drawers, added to the API single-shot judge (`routes/evaluate/judge.py`),
     and the context box shows when it's selected. Verified end-to-end on a real
     BYOK judge (score 0.95).
   - **Not done (needs real plumbing, not a wiring fix):** `context_recall` and
     surfacing these in **Datasets** runs. Testing showed why — these are
     *retrieval* metrics that need **real retrieved contexts**. Text dataset
     examples have none (the synthetic event's `response` is the expected output),
     so the dataset path would feed `contexts=[reference]` and score a
     trivially-meaningless ~1.0, and `context_recall` needs a ground-truth
     reference *and* retrieved contexts together, which no path supplies today.
     **Real fix:** capture retrieved contexts into trace-backed dataset examples,
     then have the worker score context metrics against those. (A defensive guard
     was added so the RAGAS evaluators degrade gracefully instead of raising when
     contexts are absent — applies on the next evaluator image rebuild.)

### Tier 2 — valuable, would differentiate

5. **Synthetic test-set generation.** Generate eval datasets from docs / schemas
   / a prompt (Ragas testset generator, DeepEval synthesizer). We only build
   datasets from captured real traces.

6. **Trials / repeated sampling for variance.** Run the same input N times to
   measure consistency; report variance + confidence intervals. Our run-vs-run
   compare gives deltas but no statistical significance.

7. **Annotation queues.** A structured human-review workflow (assignment, queue,
   inter-annotator agreement) rather than inline thumbs/notes.

8. **Conversational / multi-turn metrics.** Role adherence, knowledge retention,
   conversation-level completeness (DeepEval conversational). Our agentic layer
   covers trajectory but not whole-conversation quality.

9. **Eval-driven prompt optimization.** Auto-suggest better prompts from eval
   scores (Braintrust Loop, DSPy-style). Our `optimize` is caching/cost only, not
   quality.

### Tier 3 — nice-to-have

10. G-Eval-style auto-generated rubric steps for custom criteria.
11. Benchmark datasets (MMLU, etc.) for model selection.
12. Span-level evidence highlighting in score explanations.
13. Cost-vs-quality trade-off view in experiments.
14. Adversarial / red-team eval as a first-class eval kind (we scan for security separately).

---

## Recommended next moves (highest value / lowest effort first)

1. ~~Surface `context_precision` / `context_recall` in the metric pickers (gap 4)~~ — **done for `context_precision` (Traces + Prompts)**; `context_recall` + Datasets need retrieved-context capture first (see gap 4).
2. ~~Pairwise judge endpoint + "pick a winner" in the compare drawer (gap 2)~~ — **done**.
3. **Online eval rules** (gap 1) — the biggest strategic gap still open; needs a rules table (org_id, filters, sample_rate, metrics, judge), a sampling hook at `/ingest`, and a config UI. A multi-part feature, not a wiring change — scoped as the next build.
4. **Wire dashboard annotations into the calibration harvest + a customer-facing judge-accuracy card** (gap 3).
