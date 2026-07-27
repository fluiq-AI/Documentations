# FluiqAI — Open Source Products

Two standalone products ship under the FluiqAI name: **polygate** and
**Infrager**. Both are MIT licensed, free, and usable without a Fluiq account.
Neither is monetized. They exist to reach developers earlier in their workflow
than an observability platform can, and to make the engineering behind Fluiq
inspectable.

This document covers what they are, how they are built, and how to operate them.
Deployment specifics for Infrager's shared AWS resources live in
[`deployments.md`](./deployments.md#infrager); the strategic rationale is in
[`business_plan.md`](./business_plan.md) §4.4.

| | polygate | Infrager |
|---|---|---|
| What | Unified LLM client across four providers | Cloud architecture diagrams compiled to Terraform, with security linting |
| Repo | `SaurabhKumbhar24/polygate` | `SaurabhKumbhar24/Infrager` |
| Local | `D:\ideas\FluiqAI\polygate` | `D:\ideas\FluiqAI\Infrager\infrager` |
| Site | `polygate.getfluiq.com` | `infrager.getfluiq.com` |
| Ships as | PyPI `polygate` + npm `polygate`, both `0.2.0` | Hosted web app |
| Backend | None (client library) | Express + Postgres on ECS |
| License | MIT | MIT |

---

## polygate

### What it is

One function, any LLM provider. `chat()` takes a provider name, a model, and a
list of messages, and returns the same response shape no matter who served the
request. Swapping providers is a string change instead of a rewrite.

```python
from polygate import chat

response = chat(
    provider="anthropic",              # or "openai", "gemini", "moonshot"
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Say hi in one word."}],
    api_key="sk-...",                  # or ANTHROPIC_API_KEY in the environment
)

print(response.content)                # "Hi!"
print(response.usage)                  # Usage(prompt_tokens=…, completion_tokens=…, total_tokens=…)
```

The TypeScript API mirrors it (`camelCase` params, `await chat({...})`).

### Scope, deliberately narrow

polygate normalizes the request and response shape across providers and stops
there. No retries, no caching, no routing, no cost tracking. Each provider
adapter is readable in under 100 lines. The comparison it invites is LiteLLM or
Portkey, and it loses that comparison on features by design: the pitch is
something small enough to read in one sitting and depend on without worrying
about what else it does.

That narrowness is also the honest limitation. Teams that need production
gateway behavior will outgrow it, and the intended answer when they do is Fluiq,
not a bigger polygate.

### Providers

| Provider | Aliases | Env var fallback |
|---|---|---|
| Anthropic (Claude) | `anthropic`, `claude` | `ANTHROPIC_API_KEY` |
| OpenAI | `openai`, `gpt` | `OPENAI_API_KEY` |
| Google Gemini | `gemini`, `google` | `GEMINI_API_KEY` |
| Moonshot (Kimi K2) | `moonshot`, `kimi` | `MOONSHOT_API_KEY` |

Every call returns `content`, `role`, `model`, `provider`, `usage`, and `raw`
(the untouched provider response, for anything not normalized). Any parameter
beyond the known set is forwarded straight through to the provider payload, so
tools, `top_p`, and stop sequences work without polygate knowing about them.

**Fluiq dogfoods it.** As of 2026-07-26 the evaluator worker's LLM-as-Judge
routes every text judge call through `polygate.chat` (`_chat_via_polygate` in
`fluiq-workers/evaluator/jobs/helper/judge.py`) instead of per-provider SDKs —
one transport for `openai`/`anthropic`/`gemini`/`moonshot`. See
[`evaluator.md` §2](evaluator.md). (The vision/multimodal judge still uses the
native SDKs, since image blocks aren't expressible through polygate's
string-content messages yet.)

### Layout

```
polygate/
├── sdk/
│   ├── python/       polygate/{client,types,exceptions}.py + providers/{anthropic,openai,gemini,moonshot}.py
│   ├── typescript/   src/{client,types,errors}.ts + providers/ + __tests__/
│   ├── README.md CONTRIBUTING.md LICENSE
└── website/          Next.js static export → polygate.getfluiq.com
```

Adding a provider is one adapter file plus one line in the registry, in each
language.

### Releasing

Both packages are published at `0.2.0` (verified on PyPI and npm). `sdk/README.md`
now carries the `pip install polygate` / `npm install polygate` instructions. The
`website/` is a Next.js **static export** (`baseDirectory: out`) deployed by
Amplify from `amplify.yml` at the repo root with `appRoot: website`.

---

## Infrager

### What it is

A Figma-style canvas for cloud architecture that emits production-ready
Terraform. Drag resources on, connect them to express real relationships, and
the HCL appears next to the diagram, dependency-ordered. A security linter reads
the same diagram and flags problems while you draw, rather than after a
`terraform plan` or a failed compliance review.

The premise: infrastructure mistakes are visible in the diagram long before they
are visible in code. A security group open to the world, a public bucket, an
unencrypted database, an IAM role with `Action: "*"` — all of it is in the
drawing already.

### Architecture

The load-bearing decision is the **intermediate representation**. The canvas
serializes into a typed IR that knows nothing about React Flow or about
Terraform syntax; codegen and linting both consume that IR. Adding a resource
type means extending the IR union, adding one emitter, and optionally adding
lint rules.

```
apps/web        Next.js (App Router) — canvas, dashboard, and ALL codegen + linting (client-side)
  lib/ir/       IR schema: discriminated-union nodes, typed relationship edges
  lib/catalog/  Per-provider service catalogs (aws.ts, gcp.ts) — one line per service
  lib/codegen/  IR → HCL: one emitter per rich type, a generic emitter, topological ordering
  lib/lint/     Rule engine: each rule is a predicate over the IR returning findings
  lib/editor/   React Flow ↔ IR serialization (the only place canvas types appear)
  lib/api.ts    Bearer-token client for the API
apps/api        Express + TypeScript: JWT auth, PostgreSQL persistence
```

Codegen and linting never touch the server. The API stores diagrams and does not
interpret them, so a diagram becomes Terraform without leaving the browser.

### Two tiers of resources

Modeling every cloud service with typed properties would take months, so the
palette has two tiers:

- **Core (rich)** — eight AWS types with typed properties, bespoke emitters, and
  security rules: VPC, Subnet, Security Group, EC2, ALB, RDS, S3, and IAM
  Role/Policy (one node that expands into role + inline policy + instance
  profile when an EC2 assumes it). Marked `LINTED` in the palette.
- **Catalog (generic)** — 150+ AWS and GCP services, each one line in a catalog
  file: label, category, canonical Terraform resource type, default attributes.
  These emit real `resource` blocks with user-editable freeform attributes.

Growing toward full coverage is data entry rather than engineering. Adding a
cloud is a new catalog file plus a provider header block; Azure is next.

### Relationships

Edge direction always means *source references target*, matching the direction
of a Terraform attribute reference. Dependency ordering then falls out of a
topological sort for free.

| Relationship | From → To |
|---|---|
| `in_vpc` | subnet, security group → VPC |
| `in_subnet` | EC2, ALB, RDS → subnet |
| `uses_security_group` | EC2, ALB, RDS → security group |
| `assumes_role` | EC2 → IAM role |
| `targets` | ALB → EC2 |
| `depends_on` | anything → anything (fallback) |

Connections are validated as they are drawn, and a connection drawn "backwards"
is silently flipped to the correct direction.

### Security rules

| Rule | Severity |
|---|---|
| Security group exposing SSH, RDP, or a database port to `0.0.0.0/0` | error (warning for other ports) |
| S3 bucket without a public access block | error |
| RDS or S3 without encryption at rest | error |
| IAM `Allow` with `Action: "*"` **and** `Resource: "*"` | error (warning for either alone) |
| RDS marked publicly accessible | error |
| Missing VPC/subnet attachment, or an ALB in fewer than two AZs | warning |
| Catalog resource with no attributes configured | warning |

Findings appear three places: a badge on the canvas node, the Issues panel
(click to pan to the node), and the inspector for the selected resource.

New resources default to encrypted, private, and closed, so warnings appear when
someone opts **into** risk rather than greeting them on a blank canvas.

### Honest output

Codegen never invents infrastructure. A subnet with no VPC edge emits
`vpc_id = "" # INFRAGER TODO: connect this subnet to a VPC` — the file stays
parseable and the gap is impossible to miss. Secrets never enter the diagram:
RDS passwords and ACM certificate ARNs are emitted as Terraform variables.

### Stack and operations

Next.js 16 + TypeScript + Tailwind v4, React Flow (`@xyflow/react`), hugeicons,
Express + `pg`, PostgreSQL. Auth is a JWT Bearer token in `localStorage` (no
cookies), which is the accepted tradeoff for a cookie-less SPA against a
separate API: an XSS on the page can read the token, where an httpOnly cookie
could not.

Local development needs Postgres and both apps:

```bash
docker run -d --name infrager-pg \
  -e POSTGRES_USER=infrager -e POSTGRES_PASSWORD=infrager -e POSTGRES_DB=infrager \
  -p 5433:5432 postgres:16

npm install
npm run dev        # api → :4000, web → http://localhost:3000
```

Tables are created on boot, so there is no migration step. Smoke tests for the
two engines print their output directly:

```bash
cd apps/web
npx tsx scripts/codegen-smoke.ts   # generated HCL for a sample diagram
npx tsx scripts/lint-smoke.ts      # findings for secure and insecure variants
```

Production deployment, including the fluiq resources Infrager borrows and the
manual image-release steps, is documented in
[`deployments.md`](./deployments.md#infrager).

### Known gaps

- **No CI for `apps/api`.** Web deploys automatically from `main`; the API image
  is built and pushed by hand. This is the most worthwhile thing to fix next.
- **Generic-tier codegen is best-effort.** It emits the right resource type with
  the attributes given, but it does not know service-specific naming rules
  (BigQuery dataset IDs rejecting hyphens, for example). That fidelity is what
  promoting a service to the rich tier buys.
- **No `terraform validate` in CI.** Output is verified by typecheck and smoke
  tests, not by the real binary.
- **No finding suppressions.** There is no way to record "this bucket is
  intentionally public", which real use will demand. It belongs with
  persistence, since suppressions need somewhere to live.
- **One-way only.** Diagram → Terraform. Importing existing state back into a
  diagram is a roadmap item, not a v1 feature.

---

## Shared conventions

Both products follow the FluiqAI design system (`DESIGN.md`): warm-paper and ink
with a single cobalt signal, Figtree and JetBrains Mono, hugeicons rather than
phosphor, and class-based dark mode matching the `ThemeContext` pattern in
`fluiq-frontend`.

Both are surfaced on `getfluiq.com` through the Developer nav dropdown, the
footer's Open Source column, and the sitemap. Infrager additionally has a
landing page at `/infrager` and a section on the homepage.

Neither product has an attribution path back to Fluiq signups today. Referral
traffic is the only signal, so any claim about their contribution to the funnel
is currently inferred rather than measured. Worth instrumenting before either
gets more investment.
