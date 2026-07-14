# FluiqAI — Business Plan

> *The only LLM observability platform with built-in security, evaluation, and cost optimization — shipped as a single SDK line.*

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Problem](#2-the-problem)
3. [The Solution](#3-the-solution)
4. [Product](#4-product)
5. [Market Analysis](#5-market-analysis)
6. [Competitor Analysis](#6-competitor-analysis)
7. [Our Differentiation & Moat](#7-our-differentiation--moat)
8. [Pricing](#8-pricing)
9. [Go-To-Market Strategy](#9-go-to-market-strategy)
10. [Business Model & Unit Economics](#10-business-model--unit-economics)
11. [Traction & Roadmap](#11-traction--roadmap)
12. [Team](#12-team)
13. [Contact](#13-contact)

---

## 1. Executive Summary

Every company building AI products today faces the same three invisible crises: they cannot see what their LLMs are actually doing, they cannot trust that outputs are accurate, and they have no idea whether their prompts are being attacked.

**FluiqAI** solves all three with a single SDK integration. Developers call `fluiq.instrument()`, `fluiq.eval()`, `fluiq.secure()`, and `fluiq.optimize()` and immediately get a production-grade observability, evaluation, security, and cost-optimization stack — without changing their application logic.

FluiqAI is a **SaaS LLMOps platform** targeting AI-native startups, enterprise AI teams, and developer agencies building production LLM applications. We charge on a usage + seat model, with a generous free tier for developer adoption and a security tier that unlocks paid conversion.

**Key metrics we are targeting by end of Year 1:**
- 500 active organizations
- $40K MRR
- 25M traces processed per month
- NPS > 60

---

## 2. The Problem

### 2.1 LLMs in production are a black box

When a company deploys a chatbot, RAG pipeline, or autonomous agent, they have virtually no visibility into:
- What prompts and responses are flowing through the system
- Whether the model is hallucinating
- Whether retrieved context is relevant to the question
- How much each LLM call is costing them
- Whether users are trying to jailbreak or extract sensitive data

### 2.2 The cost of ignorance is compounding

- **Hallucinations** erode user trust silently — the company often learns about them from customer complaints, not logs
- **Prompt injection attacks** are surging as LLMs are wired into databases, email, and financial systems
- **Redundant LLM calls** silently inflate monthly cloud bills — teams often pay for the same completion 10× a day
- **Provider prompt caching is unconfigured** — Anthropic prefix caching can cut input costs by 90% but requires explicit setup most teams skip
- **Evaluation is ad hoc** — most teams rely on manual spot-checks or fragile regex tests

### 2.3 The current tools are fragmented

Existing solutions force teams to stitch together separate tools for tracing, evaluation, and security. This creates integration overhead, data silos, and alert fatigue. Security for LLMs in particular is almost entirely unaddressed by the current generation of LLMOps tools.

---

## 3. The Solution

FluiqAI is a **unified LLMOps platform** delivered through a thin Python SDK. One `instrument()` call, full stack coverage.

```python
import fluiq

fluiq.instrument(api_key="flq_...")  # patches every supported provider automatically

# Optional: layer on security, optimization, and evaluation
fluiq.secure(mode="block")           # block prompt injection, PII leaks, agentic attacks (Growth+)
fluiq.optimize()                     # Redis caching + Anthropic prompt-prefix injection (Team+)
fluiq.eval(metrics=["hallucination", "relevance"], mode="warn")
```

That setup activates:
- **Real-time trace collection** — every prompt, response, latency, cost, and token count
- **Automatic evaluation** — LLM-judge scoring for hallucination, faithfulness, relevance, toxicity, coherence, plus **agentic evaluation** of whole agent runs (tool-selection correctness, trajectory/goal-completion, and multi-agent coordination), with judge **calibration** against a human golden set
- **Security scanning** — PII, prompt injection, jailbreak, skeleton key, secrets, indirect injection, plus agentic threats (RAG poisoning, tool-input exfiltration, tool-allowlist violations, cross-agent injection, trust-boundary escalation) and a response gate
- **Cost optimization** — semantic Redis cache + provider-level prompt prefix caching (Anthropic auto-injection, OpenAI/Gemini token capture) + MCP tool result caching

Everything flows through a Kafka-backed async pipeline to ClickHouse, with a real-time dashboard for instant visibility.

---

## 4. Product

### 4.1 Core Features

#### `fluiq.instrument()` — Observability
- Captures every LLM call: model, provider, prompt, response, latency, token usage, estimated cost
- **Supported providers**: OpenAI (chat, responses, streaming, embeddings, images, audio), Anthropic (Messages API, Beta, streaming), Google Gemini, Google Vertex AI, Voyage AI
- **Agent frameworks**: LangChain, LangGraph, LlamaIndex, CrewAI, Google ADK, MCP (Model Context Protocol)
- **Vector databases**: Chromadb, Pinecone, Qdrant, Weaviate, FAISS
- Multi-agent trace stitching: parent/child spans for agent pipelines and chains
- Real-time SSE streaming to dashboard — no polling, no page refresh
- `@fluiq.trace` decorator for custom Python functions

#### `fluiq.eval()` — Evaluation
- **Automatic evaluation** on every LLM trace (warn mode, server-side) or as a blocking guard (block mode)
- SDK metrics: Hallucination, Faithfulness, Relevance, Toxicity, Coherence, Completeness; the evaluator worker additionally supports RAGAS-family metrics (Answer Relevancy, Context Precision, Context Recall) for retrieval traces
- LLM-judge powered, configurable judge model (defaults to `claude-haiku-4-5`); all judging runs server-side in the evaluator worker
- GitHub Actions CI/CD gate: `GET /api/v1/optimize/evals` returns recent scores; fail the build if quality drops
- Results streamed to dashboard alongside traces via SSE — single pane of glass; prompt playground + model-compare endpoints for ad-hoc evaluation

#### `fluiq.secure()` — Security (Growth plan+)
- **Pre-call guard** (`/secure/check`): blocks prompt injection, jailbreak, skeleton key, and semantic attacks before the LLM call is made, with custom allow/deny lists
- **Response gate**: scans the LLM's output before returning it to the caller — blocks when the response itself carries an attack or PII echo
- **Post-call scan**: full async scan of prompt + response for PII (Presidio), secrets (API keys, tokens), and indirect injection in tool outputs
- **Agentic threat coverage** — RAG poisoning, tool-input exfiltration, tool-allowlist violations, cross-agent injection, and DAG-keyed trust-boundary escalation / session crescendo detection across multi-agent runs
- Semantic attack classifier (torch + spaCy) running in a **dedicated security worker** on its own Kafka topic — isolates heavy ML deps from the evaluator
- **Guardrail policies**: named, dashboard-configurable policies controlling block/warn thresholds, blocked categories, PII-ignore list, tool allowlist, custom allow/deny lists, response-gate toggle, and an alert webhook
- Results merged into the trace view; block or warn mode; fail-open guarantee (scan outage never blocks production traffic)

#### `fluiq.optimize()` — Cost Reduction (Team plan+)
- **LLM response cache**: trace-driven Redis caching; Fluiq's backend provisions dedicated Redis per org based on real traffic patterns; zero infrastructure to manage
- **MCP tool caching**: `ClientSession.list_tools()` and `call_tool()` results cached transparently — list_tools invalidated on server restart; call_tool keyed by `(server_url, tool_name, args_hash)`
- **Provider prompt caching**:
  - *Anthropic*: auto-injects `cache_control: {"type": "ephemeral"}` on system prompt and last tool — up to 90% savings on repeated system-prompt tokens
  - *OpenAI*: captures `cached_tokens` from `usage.prompt_tokens_details` automatically (no injection needed; OpenAI caches prompts ≥ 1024 tokens)
  - *Gemini*: captures `cached_content_token_count` from `usage_metadata` for `CachedContent`-backed calls
- **Optimize dashboard**: Redis hit rate by cache type (LLM, MCP, embeddings, vectorstore), Prompt Caching card with per-provider token savings
- `observe` mode: measures projected savings without serving cached responses

### 4.2 Architecture (Why It's Fast and Safe)

The SDK is intentionally thin — it embeds configuration into the trace envelope and sends it to `/ingest`. All heavy processing (LLM judge evaluation, security scanning, PII detection, semantic similarity) runs asynchronously in dedicated worker processes backed by Kafka. The user's application incurs zero latency from evaluations or security scans in warn mode.

For block mode, security uses a synchronous pre-call `/secure/check` and the response gate runs synchronously during `/ingest`; both round-trip the security worker over Kafka and read back the verdict. All heavy compute stays server-side.

Three dedicated workers consume from separate Kafka topics (security is split out so its torch/spaCy footprint never slows evaluation):

```
SDK → /ingest (API) → Kafka: traces     → Tracer Worker    → ClickHouse + SSE
                    → Kafka: evaluations → Evaluator Worker → ClickHouse + SSE   (LLM-judge metrics)
                    → Kafka: security    → Security Worker  → ClickHouse + SSE   (scan, response gate, agentic threats)

API also runs an alert consumer (stable shared group) that fires Slack alerts on eval/security thresholds — exactly once per event.
```

Production infra: Kafka on **AWS MSK** (SASL/SCRAM over TLS), ClickHouse self-hosted on **EC2**, PostgreSQL on **AWS RDS**, Redis for the SDK cache proxy + sync reply correlation, and S3 for blog media.

### 4.3 Dashboard Pages

| Page | What it shows |
|------|--------------|
| Overview | Traces, cost, token, and eval summary; spending charts; quota usage |
| Traces | Live trace stream; filter by model, agent, status, risk, integration, quality; span/architecture tree with tool-level selection |
| Agents | Per-agent aggregation: runs, total cost, avg latency, token count |
| Security | Risk level, PII types, attack patterns (incl. agentic threats), redacted fields per trace; response gate flag |
| Guardrails | Dashboard-configurable guardrail policies (thresholds, categories, allow/deny lists, tool allowlist, PII-ignore, webhook) |
| Alerts | Slack alert configuration for eval and security thresholds (+ test) |
| Optimize | Redis cache hit rate by type; MCP cache hit/miss; Prompt Caching card (cached tokens saved by provider) |
| Tests / Evals | Per-trace quality scores; pass/fail breakdown; eval playground; dataset management |
| Prompts | Template management, version history, environment deployments (dev/staging/prod), playground |
| Datasets | Named trace collections for regression testing |
| API Management | Create, reveal, and revoke workspace API keys |
| Audit | Tamper-evident (HMAC) request audit log |
| Getting Started / Profile | Onboarding guide; account settings |

Plus public marketing surfaces built into the same app: the LLM cost calculator (`/models`), an in-house blog CMS (TipTap editor, prerender-on-publish for SEO), per-pillar landing pages, and competitor-comparison pages.

---

## 5. Market Analysis

### 5.1 Total Addressable Market

The LLM application market is exploding. As of 2025:

- **GenAI market**: $67B in 2025, projected to reach **$1.3 trillion by 2032** (CAGR ~40%)
- **MLOps/LLMOps market**: $3.4B in 2025, projected **$23.1B by 2030** (MarketsandMarkets)
- **AI security market**: $24B in 2025, projected **$60B by 2028** as LLM attack surfaces expand
- Number of companies with production LLM applications: **~120,000+ globally** and growing 3× per year

### 5.2 Serviceable Addressable Market

Targeting companies with at least one production LLM application that requires observability:
- Estimated **~80,000 companies** globally in 2025
- Average spend on observability tooling: **$3,000–$50,000/year** depending on scale
- SAM: approximately **$800M/year**

### 5.3 Serviceable Obtainable Market (3-year target)

- Capture **0.5% of SAM** → **~400 paying customers** → **~$4M ARR**
- Conservative given developer-led PLG motion and strong free tier

### 5.4 Market Tailwinds

1. **Enterprise AI adoption is accelerating** — every Fortune 500 is deploying LLM applications in 2025–2026
2. **Regulatory pressure on AI outputs** — EU AI Act, GDPR for AI systems, and SEC guidance on AI disclosures are forcing auditability
3. **LLM attacks are mainstream** — prompt injection is now in the OWASP Top 10 for LLMs; enterprises need documented security posture
4. **LLM cost optimization pressure** — as AI budgets tighten, both Redis caching and provider-level prefix caching have direct CFO-level visibility
5. **RAG is now default architecture** — every RAG deployment needs retrieval quality evaluation
6. **MCP adoption exploding** — Model Context Protocol is becoming the standard for tool use; MCP tool caching is a new, unaddressed cost center

---

## 6. Competitor Analysis

### 6.1 Landscape Overview

| Platform | Tracing | Evaluation | Security | Optimization | Prompt Cache | MCP Cache | Pricing Start | Open Source |
|---|---|---|---|---|---|---|---|---|
| **FluiqAI** | ✅ | ✅ Full suite | ✅ **PII + injection + response gate** | ✅ Redis + provider | ✅ Auto-inject | ✅ | Free | ❌ |
| Langfuse | ✅ | ✅ Basic | ❌ | ❌ | ❌ | ❌ | Free / $59/mo | ✅ |
| LangSmith | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | Free / $39/mo | ❌ |
| Braintrust | ✅ | ✅ Strong | ❌ | ❌ | ❌ | ❌ | Free / custom | ❌ |
| Arize / Phoenix | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | Free / custom | ✅ Phoenix |
| Helicone | ✅ Proxy | ❌ | ❌ | ✅ Rule-based | ❌ | ❌ | Free / $80/mo | ✅ |
| Portkey | ✅ Proxy | ❌ | Partial | ✅ Semantic | ❌ | ❌ | Free / $49/mo | ❌ |
| Galileo | ✅ | ✅ Strong | ❌ | ❌ | ❌ | ❌ | $50K+/yr | ❌ |
| W&B Weave | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | Free / custom | ❌ |

### 6.2 Deep Competitor Profiles

---

#### Langfuse
**Positioning**: Open-source LLM observability, self-host or managed cloud.

**Strengths**
- Strong open-source community (20K+ GitHub stars)
- Good self-hosting story for privacy-conscious orgs
- Decent prompt management and dataset versioning
- Integrations with many LLM frameworks

**Weaknesses**
- Evaluation is manual-first — no automatic LLM-judge on every trace
- Zero security capability — no PII scanning, no attack detection, no response gate
- No cost optimization / caching layer of any kind
- Self-hosting burden is significant for small teams
- Open-source model limits monetization leverage

**Why customers leave Langfuse for FluiqAI**: They want evaluations to run automatically without building their own eval pipelines, and they need security for enterprise deals.

---

#### LangSmith (LangChain)
**Positioning**: The official observability platform for LangChain applications.

**Strengths**
- Deeply integrated with LangChain and LangGraph
- Good tracing visualization for chains and agents
- Dataset management and regression testing

**Weaknesses**
- **Vendor lock-in**: only valuable if you use LangChain/LangGraph
- No built-in security; no response gate
- Evaluation requires manual dataset curation — no real-time auto-eval
- Pricing scales aggressively at volume ($39/mo base but expensive at scale)

**Why customers leave LangSmith for FluiqAI**: They outgrow LangChain, use multiple frameworks, and need security + automatic eval.

---

#### Braintrust
**Positioning**: Evaluation-first platform with strong dataset and CI/CD integration.

**Strengths**
- Best-in-class dataset management and eval pipelines
- Strong CI/CD integration (eval on every PR)
- Clean, developer-friendly API

**Weaknesses**
- Primarily offline evaluation — not real-time production monitoring
- No security, no response gate, no cost optimization
- Pricing is opaque and enterprise-only at scale

**Why customers leave Braintrust for FluiqAI**: They need real-time production monitoring, not just pre-deployment eval, plus security compliance.

---

#### Portkey
**Positioning**: AI gateway with observability, routing, and semantic caching.

**Strengths**
- Semantic caching via proxy (similar to Fluiq's Redis cache)
- Model routing and fallback logic
- Good multi-provider support
- Reasonable pricing at entry level

**Weaknesses**
- **Proxy architecture** — routes all traffic through their servers (security and compliance risk)
- No LLM-as-judge evaluation
- No real security scanning (basic guardrails only, no PII or injection detection)
- No provider prompt-cache injection or MCP caching
- Proxy adds latency to every call

**Why customers leave Portkey for FluiqAI**: Proxy model is a non-starter for enterprise compliance; no real security or evaluation.

---

#### Helicone
**Positioning**: Lightweight proxy-based LLM logging with basic caching.

**Strengths**
- Zero-code integration via proxy (just change base URL)
- Very affordable entry pricing
- Has a caching layer (rule-based, not semantic)

**Weaknesses**
- **Proxy architecture is a security risk** — all API keys and data routed through their servers
- No evaluation capabilities
- No security scanning; no response gate
- Proxy adds latency to every call

**Why customers leave Helicone for FluiqAI**: They hit Helicone's ceiling within weeks — logs calls but offers no insight, and the proxy model is a non-starter for enterprise compliance.

---

#### Galileo
**Positioning**: Enterprise-grade LLM evaluation and hallucination detection.

**Strengths**
- Strong hallucination detection (flagship feature)
- Good enterprise packaging and compliance story

**Weaknesses**
- Enterprise-only pricing (typically $50K+/year) — no developer tier, no self-serve
- No security scanning; no response gate
- No cost optimization
- Long sales cycles, not PLG

**Why customers don't start with Galileo**: Price and sales process. A startup can't pay $50K/year on day one.

---

## 7. Our Differentiation & Moat

### 7.1 The Full Stack No Competitor Offers

| Capability | Us | Every Competitor |
|---|---|---|
| Real-time tracing (all providers + MCP + vectorstores) | ✅ | ✅ (most, limited) |
| Auto evaluation on every trace | ✅ | Partial (usually manual) |
| **Production security: PII + injection + response gate** | ✅ | ❌ |
| **Redis semantic cache** | ✅ | Partial (Helicone/Portkey, proxy only) |
| **Anthropic prompt-prefix injection** | ✅ | ❌ |
| **MCP tool result caching** | ✅ | ❌ |
| **Provider cached token capture (all 3 providers)** | ✅ | ❌ |

No competitor offers all of this. Most offer two or three capabilities at most.

### 7.2 Security Is Our Primary Moat

The LLM security space is essentially uncontested in the observability category. Every existing platform treats security as an afterthought. The **response gate** — scanning the LLM's output before it reaches the caller — is a capability no other observability platform offers.

**Why this is defensible:**
- Security is a **compliance forcing function** — once a CISO approves FluiqAI, switching becomes procurement, not a developer preference
- The response gate requires synchronous ingest — it is architecturally incompatible with proxy-based competitors (they cannot add it without fundamental redesign)
- Security data is deeply integrated with trace data — you cannot replicate this with a bolt-on tool

### 7.3 Cost Optimization Depth

Our optimization layer has three independent dimensions:
1. **Redis semantic caching** — same response, zero LLM cost
2. **Provider prompt prefix caching** — Anthropic auto-injection reduces input token cost by up to 90% on repeated system prompts
3. **MCP tool caching** — in agentic systems, `list_tools()` and `call_tool()` can dominate latency; caching both is unique to us

Together these deliver measurable savings across every dimension of an LLM application's cost structure. No competitor touches more than one.

### 7.4 Thin SDK + Server-Side Architecture

Our SDK does one thing: wraps calls and sends data. All compute is server-side. This means:
- Upgrading FluiqAI (new evaluators, better PII models, new attack signatures, new cache strategies) requires **zero SDK updates**
- Competitors with fat SDKs create update friction and compatibility headaches
- The thin SDK is also a security argument — we're not shipping ML models into customer environments

### 7.5 Unified Data Model = Network Effects

Because tracing, evaluation, security, and optimization all live in the same ClickHouse trace record, we can build cross-cutting insights no siloed tool can match:
- "Show me traces where hallucination score is high AND a security risk was detected"
- "Which MCP tools have the highest cache hit rate?"
- "What percentage of Anthropic cache_read savings came from system-prompt injection?"

This unified data model deepens as usage grows, making churn progressively harder.

---

## 8. Pricing

### 8.1 Pricing Philosophy

- **Observability is free and unlimited on every tier** — no trace/span/agent cap, ever. This is the top-of-funnel: teams instrument freely and only feel a paid boundary once they need history.
- **Retention is the primary paid axis** — Free keeps a rolling 14-day window; paid keeps traces forever. Simple to explain, and it converts on genuine need (incident forensics, regression datasets, compliance) rather than an artificial cap.
- **Optimization unlocks at Team; security unlocks at Growth** — the two additional paid value drivers ladder customers up the plans
- **A no-card 5-day trial** of a paid tier lets teams feel unlimited retention (and higher eval budgets) before deciding
- Simple, transparent pricing — no "contact sales" required below Enterprise

---

### 8.2 Plans

#### Free
**$0 — forever**

Full observability for your first pipeline. No credit card required.

| Feature | Detail |
|---|---|
| Traces | **Unlimited** (free, uncapped — on every tier) |
| Trace retention | 14 days (rolling window) |
| LLM-as-judge evaluations | 1,000 / month |
| Seats | 1 |
| Paid trial | 5-day, no-card trial of Team or Growth |
| Supported providers | OpenAI, Anthropic, Gemini, Vertex AI, Voyage; LangChain, LangGraph, LlamaIndex, CrewAI, Google ADK, MCP; all vectorstores |
| Trace explorer & live dashboard | ✅ |
| `fluiq.eval()` (warn mode) + GitHub Action CI/CD eval gates | ✅ |
| `fluiq.optimize()` | ❌ |
| `fluiq.secure()` | ❌ |
| Support | Community |

---

#### Team ⭐ Most Popular
**$499 / workspace / month**

Unlimited tracing, response caching, and provider prompt caching for teams shipping production AI pipelines.

| Feature | Detail |
|---|---|
| Tracing | Unlimited |
| Trace retention | **Forever** (never-expiring) |
| LLM-as-judge evaluations | 10,000 / month |
| Seats | 10 (+ $49 / extra seat) |
| Everything in Free | ✅ |
| `fluiq.optimize()` — Redis cache + Anthropic prompt-prefix injection + MCP caching | ✅ |
| Eval alerts to Slack | ✅ |
| `fluiq.secure()` | ❌ (Growth+) |
| Support | Priority |

---

#### Growth
**$1,499 / workspace / month**

Full security suite and higher eval throughput for production-scale, compliance-sensitive pipelines.

| Feature | Detail |
|---|---|
| Tracing | Unlimited |
| Trace retention | **Forever** (never-expiring) |
| LLM-as-judge evaluations | 100,000 / month |
| Seats | 20 (+ $39 / extra seat) |
| Everything in Team | ✅ |
| `fluiq.secure()` — PII, injection, jailbreak, secrets, agentic threats, response gate | ✅ |
| Guardrail policies (dashboard-configurable) | ✅ |
| Security alerts to Slack | ✅ |
| Support | Priority Slack |

---

#### Enterprise
**Custom — annual contract**

Compliance, on-prem deployment, and a dedicated success engineer.

| Feature | Detail |
|---|---|
| Tracing & evaluations | Unlimited |
| Seats & workspaces | Unlimited |
| Everything in Growth | ✅ |
| SSO / SAML & SCIM provisioning | ✅ |
| VPC or on-prem deployment | ✅ |
| Custom data retention & residency | ✅ |
| Audit logs & role-based access | ✅ |
| Dedicated Slack channel & SLA | ✅ |

---

## 9. Go-To-Market Strategy

### 9.1 Phase 1 — Developer Adoption (Months 1–6)

**Goal**: 200 free-tier signups, 20 paying customers

**Tactics:**

1. **Content marketing & SEO**
   - Publish weekly technical content: "How to detect prompt injection in production", "Why your RAG pipeline is hallucinating", "LLM cost optimization: how we cut our OpenAI bill by 60%", "Anthropic prompt caching: the 90% savings most teams are leaving on the table"
   - Target keywords: "llm observability", "prompt injection detection", "llm evaluation python", "openai tracing", "anthropic prompt caching"
   - Goal: 5,000 organic monthly visitors within 6 months

2. **Developer community presence**
   - Answer LLM observability questions on Reddit (r/MachineLearning, r/LangChain), HackerNews, and Discord communities
   - Ship useful open-source utilities to drive GitHub visibility
   - Submit to product directories: Product Hunt, There's An AI For That, Futurepedia

3. **Integration ecosystem**
   - Publish official integration guides for OpenAI, Anthropic, LangChain, LlamaIndex, CrewAI, Google ADK, MCP
   - Ensure we appear in "alternatives to LangSmith" and "Langfuse vs" searches

4. **Free tier is the product**
   - Make the free tier genuinely excellent — 5M lifetime traces covers most early-stage apps
   - Frictionless onboarding: working trace in < 5 minutes

---

### 9.2 Phase 2 — Conversion & Expansion (Months 6–18)

**Goal**: 200 paying customers, $40K MRR

**Tactics:**

1. **In-product upgrade triggers**
   - When a free user needs trace history beyond the rolling 14-day window: one-click upgrade to Team for unlimited (never-expiring) trace retention
   - When `fluiq.optimize()` is called below Team, or `fluiq.secure()` below Growth: graceful 402 with a one-click upgrade to the unlocking plan
   - When an LLM eval quota is exhausted mid-month: prompt to upgrade

2. **Security-led enterprise upsell**
   - Every enterprise security RFP for an AI system now asks: "How do you prevent prompt injection?" and "How do you detect PII in LLM outputs?"
   - Position `fluiq.secure()` + response gate as the complete answer
   - Partner with AI compliance consultants advising companies on EU AI Act readiness

3. **Cost savings story**
   - Anthropic prefix caching auto-injection is a uniquely compelling demo: a single `fluiq.optimize()` call can cut input token bills by 50–90% for prompts with large system prompts
   - Use the Optimize dashboard Prompt Caching card as the sales artifact

4. **Outbound to AI-first companies**
   - Target Series A–C AI-native startups (AngelList, Crunchbase)
   - Message: "Your LLM is in production. Do you know when it hallucinates?"

5. **Agency channel**
   - Developer agencies building LLM applications for clients
   - Offer agency partnerships: 20% revenue share, co-branded reports

---

### 9.3 Phase 3 — Enterprise & Platform (Months 18–36)

**Goal**: 500 customers, $150K MRR, first $2M ARR accounts

**Tactics:**

1. **Enterprise direct sales**
   - Hire first enterprise AE at Month 18
   - Target financial services, healthcare, and legal tech — highest regulatory pressure
   - Lead with compliance story: "FluiqAI gives your CISO a documented security posture for every LLM call"

2. **Marketplace presence**
   - AWS Marketplace, Azure Marketplace listings

3. **Ecosystem partnerships**
   - Co-sell with LLM providers (Anthropic, OpenAI) who want their customers to succeed
   - Integrate with existing enterprise observability stacks (Datadog, Grafana) via official exporters

4. **TypeScript SDK**
   - Python captures the ML/AI developer; TypeScript captures the application developer building LLM features in Node.js/Next.js backends
   - TypeScript SDK mirrors the Python SDK's security and optimization capabilities

---

### 9.4 Positioning Statement

**For**: AI engineering teams and AI-native startups building production LLM applications

**Who need**: visibility, reliability, and security in their LLM stack

**FluiqAI is**: a unified LLMOps platform

**That**: instruments any LLM call with tracing, automatic evaluation, security scanning (including response gate), and cost optimization (Redis cache + provider prompt-prefix caching + MCP tool caching) through a single SDK integration

**Unlike**: Langfuse, LangSmith, and Braintrust which require separate tools for evaluation and have no security capability; and Helicone/Portkey which route all traffic through a proxy

**FluiqAI**: delivers the complete LLMOps stack — observe, evaluate, secure, optimize — in one platform that takes 5 minutes to integrate and never touches your API keys

---

## 10. Business Model & Unit Economics

### 10.1 Revenue Model

- **Subscription** (monthly/annual): primary revenue driver, 80% of revenue
- **Overage**: metered usage above plan limits, 15% of revenue
- **Professional services**: onboarding, custom evaluators, enterprise integration support, 5% of revenue

### 10.2 Target Unit Economics (12-month horizon)

Blended ARPU assumes a paying-customer mix of ~70% Team, ~25% Growth, ~5% Enterprise.

| Metric | Target |
|---|---|
| Average Revenue Per User (ARPU) | ~$750/month (blended paying) |
| Customer Acquisition Cost (CAC) | ~$500 (PLG-assisted, low-touch) |
| LTV (24-month) | ~$18,000 |
| LTV:CAC ratio | ~36:1 |
| Gross margin | ~78% |
| Monthly churn (target) | < 2.5% |
| Payback period | < 1 month |

**MRR bridge to $40K target:**
- 40 Team customers × $499 = $19,960
- 12 Growth customers × $1,499 = $17,988
- 2 Enterprise customers × $1,500 avg = $3,000
- **Total: ~$40,950 MRR** (achievable at ~54 paying customers)

### 10.3 Cost Structure

- Infrastructure (ClickHouse, Kafka, Redis, compute): ~20% of revenue at scale
- LLM judge costs (evaluation compute): billed through to customer indirectly via evaluation quotas
- Engineering: 60% of headcount cost
- Sales & Marketing: 25% of headcount cost
- G&A: 15% of headcount cost

---

## 11. Traction & Roadmap

### 11.1 Current State (Platform Complete)

- **Python SDK**: thin, fail-open instrumentation client covering OpenAI, Anthropic, Gemini, Vertex AI, Voyage; LangChain, LangGraph, LlamaIndex, CrewAI, Google ADK, MCP; Chromadb, Pinecone, Qdrant, Weaviate, FAISS
- **TypeScript SDK** (`@fluiq/sdk`): mirrors the Python surface — all integrations + 5 vectorstores, server-side eval/security, Redis optimize; published via npm Trusted Publishing
- **Security**: PII (Presidio), prompt injection, jailbreak, skeleton key, secrets, indirect injection, semantic attacks; **agentic threats** — RAG poisoning, tool-input exfiltration, tool-allowlist violations, cross-agent injection, DAG-keyed trust-boundary escalation / session crescendo; response gate; dashboard-configurable guardrail policies. Runs in a dedicated security worker on its own Kafka topic
- **Optimization**: Redis semantic cache (trace-driven profile); MCP `list_tools()` and `call_tool()` caching; Anthropic `cache_control` auto-injection; OpenAI and Gemini cached-token capture; Optimize dashboard with Prompt Caching card
- **Evaluation**: hallucination, faithfulness, relevance, toxicity, coherence, completeness (+ RAGAS-family for retrieval); warn and block modes; playground + model-compare; GitHub Actions CI gate
- **Prompts**: template management, version history, environment deployments (dev/staging/prod), LLM-as-judge playground, `fetch_prompt()` runtime API
- **Datasets**: trace collection management for regression testing
- **Alerts**: Slack alerting on eval/security thresholds via a dedicated alert consumer (stable shared group, fires once)
- **Auth & admin**: password + Google/GitHub OAuth, OTP password reset, tamper-evident (HMAC) audit log, admin console
- **Marketing surfaces**: in-house blog CMS (prerender-on-publish SEO, S3 media), LLM cost calculator, pillar + competitor-comparison pages
- **Dashboard**: 13+ pages, real-time SSE streaming, dark mode, full observe/secure/eval/optimize coverage
- **Async pipeline & infra**: three Kafka workers (tracer, evaluator, security); self-hosted Kafka on EC2 (migrated off MSK to cut ~87% of that line item), self-hosted ClickHouse on EC2, PostgreSQL on RDS, Redis, S3; per-run roll-ups (AggregatingMergeTree); Fargate-Spot autoscaling to a ~$150/mo budget; cost estimation with provider rates

### 11.2 Roadmap

**2025 — Launch ✅ (Complete)**
- ~~Public beta; Free / Team / Growth plans live~~
- ~~Security: PII, injection, jailbreak, response gate, guardrail policies~~
- ~~Optimization: Redis cache, MCP caching, provider prompt caching~~
- ~~Evaluation: metrics suite, CI/CD gate, warn + block modes~~
- ~~Prompts: templates, versioning, environments, playground · Datasets~~

**H1 2026 — TypeScript SDK, Agentic Security & Alerts ✅ (Complete)**
- ~~TypeScript SDK (`@fluiq/sdk`) GA — mirrors Python~~
- ~~Agentic threat detection (RAG poisoning, tool exfiltration, tool allowlist, cross-agent injection, trust-boundary escalation)~~
- ~~Security split into a dedicated worker/topic~~
- ~~Slack alerts (eval + security)~~
- ~~Infra migration: MSK, ClickHouse-on-EC2, RDS~~
- ~~Blog CMS + LLM cost calculator + comparison/pillar pages~~

**H2 2026 — Agentic Depth & Platform (Shipping)**
- ~~Free, unlimited observability on every tier; retention as the paid axis (14-day Free / forever paid) + no-card 5-day trial~~
- ~~Multi-agent DAG tracing & visualization (LangGraph / CrewAI / Google ADK fan-out + join detection; agentic evaluator L0–L5)~~
- ~~Multimodal tracing via payload-free media references (image, audio, file)~~
- ~~Dataset trajectory capture + batch agentic-eval / security runs (regression suites)~~
- ~~External-observability ingestion (LangSmith / Langfuse / Phoenix / Braintrust via OpenInference)~~
- Enterprise plan GA; SSO / SAML & SCIM · SOC 2 Type I · Self-hosted / VPC deployment
- A/B testing for prompts

**2027 — Data & Analytics**
- Advanced cost analytics (per-model trend, anomaly detection)
- Experiment tracking (head-to-head prompt versions)
- Memory management (`fluiq.remember()` / `fluiq.recall()`) — not yet started
- Custom dashboard widgets

---

## 12. Team

**Saurabh** — Founder & CEO

This is not a first attempt. It's a third — and the clearest one yet.

**Serve My Table** was a restaurant technology platform that reached 200 paying customers. Then a venture-backed competitor entered and gave the same product away for free. It didn't fail because the product was wrong or the customers didn't care — it failed because unlimited runway is a force multiplier a bootstrapped founder cannot outlast. The lesson: *know which fights you can win.*

**Motleyscape** was an AR/VR platform for real estate that generated ₹200,000 INR from 15 paying clients before it became clear the market wasn't ready. The lesson: *conviction without market pull burns time that compounds elsewhere.*

After both, Saurabh moved to the United States for his Master's degree — carrying two closed companies and everything learned from them.

**FluiqAI is what's next.**

The LLM observability market is not early — it is arriving right now, driven by regulatory pressure, enterprise AI adoption, and a generation of developers shipping AI systems with no tools to operate them safely. The business model is SaaS with integration moats, compliance lock-in, and data network effects.

Every layer of FluiqAI was built by Saurabh directly: the SDK, the API, the Kafka pipeline, the ClickHouse schema, the security scanner, the evaluation workers, the dashboard, the MCP caching, the provider prompt injection. Not outsourced. Not prototyped. Built.

*The market is here. The product is built. This time, the conditions are right.*

---

## 13. Contact

**Email**: fluiqai@gmail.com

**Website**: [getfluiq.com](https://getfluiq.com)

**Dashboard**: [getfluiq.com/dashboard](https://getfluiq.com/dashboard)

**GitHub**: github.com/fluiqai

**LinkedIn**: linkedin.com/company/fluiqai

---

*FluiqAI — Observe. Evaluate. Secure. Optimize.*

*© 2026 FluiqAI. All rights reserved.*
