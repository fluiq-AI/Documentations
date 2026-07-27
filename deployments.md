# Fluiq — Deployments

How each deployable ships. Every component has its own GitHub Actions workflow
(each sub-repo carries its own `.github/workflows/`). AWS region is **`us-east-2`**
throughout.

| Component | Target | Trigger | Workflow |
|-----------|--------|---------|----------|
| Frontend (`fluiq-frontend`) | AWS Amplify (Next.js SSR / WEB_COMPUTE) | push to `main` of the connected branch | `amplify.yml` (Amplify-managed) |
| API (`fluiq-api`) | Amazon ECS (cluster `fluiq`, service `fluiq-api`) | push to `main` · manual `workflow_dispatch` | `.github/workflows/deploy.yml` |
| Python SDK (`fluiq-sdk`) | PyPI (`fluiq`) | push a semver **git tag** | `.github/workflows/release.yaml` |
| TypeScript SDK (`fluiq-sdk-typescript`) | npm (`@fluiq/sdk`) | **GitHub Release published** · manual `workflow_dispatch` | `.github/workflows/publish.yml` |
| Workers (tracer/evaluator/security) | Amazon ECS | push to `main` · manual | each worker's `.github/workflows/deploy.yml` |
| Infrager web (`Infrager/apps/web`) | AWS Amplify (app `d1icguwdl9x4aw`) | push to `main` | `amplify.yml` (repo root, `appRoot: apps/web`) |
| Infrager API (`Infrager/apps/api`) | Amazon ECS (cluster `fluiq`, service `infrager-api`) | **manual** (`docker build` + `docker push`, no workflow yet) | see [Infrager](#infrager) |

> **DB migrations are automatic.** The API applies `db_queues/postgresql/schema.sql`
> and `db_queues/clickhouse/schema.sql` on startup (`_apply_schema()`), so deploying
> the API runs idempotent `CREATE TABLE IF NOT EXISTS` / `ALTER TABLE … ADD COLUMN
> IF NOT EXISTS` migrations. Deploy the **API before** any worker that reads a new
> column (see [Custom judges](#feature-note-custom-judges)).

---

## Prerequisites (one-time)

- **AWS** — repo secrets `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` with ECR
  push + ECS deploy + `iam:PassRole` for the task role. ECR repos and ECS
  services/task families already exist (`fluiq-api`, `fluiq-tracer`,
  `fluiq-evaluator`, `fluiq-security`).
- **Amplify** — the frontend app is connected to the GitHub repo with the branch
  framework set to **"Next.js - SSR"** and platform **WEB_COMPUTE**. Environment
  variables (e.g. `NEXT_PUBLIC_*`) are set in the Amplify console, not in code.
- **PyPI** — Trusted Publisher (OIDC) configured for the `fluiq` project against
  this repo's `release` environment. No `PYPI_TOKEN` secret is used.
- **npm** — Trusted Publisher (OIDC) configured at npmjs.com → `@fluiq/sdk` →
  Settings → Trusted Publisher → GitHub Actions, with workflow filename exactly
  `publish.yml`. No `NPM_TOKEN` secret is used.

---

## Frontend → AWS Amplify

Amplify builds and hosts the Next.js app. There is no ECR/ECS step.

**Automatic (normal path)**
1. Merge to `main`. Amplify detects the push and runs `amplify.yml`:
   `nvm use 20` → `npm ci` → `npm run build`, publishing the `.next` output via
   the managed SSR adapter.
2. Watch the build in the **Amplify console** → the connected branch.

**Manual**
- In the Amplify console, open the branch and click **Redeploy this version**
  (re-runs the last commit) or **Run build**.

**Verify**
- Open the deployed URL; confirm the changed page renders. For SSR/runtime
  issues check **Amplify → Monitoring → Hosting compute logs**.

**Notes**
- Next.js 16 needs Node ≥ 20.9 — `amplify.yml` pins Node 20 in `preBuild`.
- Type errors fail the build (`npm run build` runs `next build`). Run
  `npx tsc --noEmit` locally first.

---

## API → Amazon ECS

`fluiq-api/.github/workflows/deploy.yml` builds the Docker image, pushes to ECR,
and rolls the ECS service.

**Automatic (normal path)**
1. Merge to `main`. The workflow:
   - builds the image and pushes `:$GITHUB_SHA` and `:latest` to ECR `fluiq-api`;
   - **downloads the live task definition** from ECS (it is *not* committed —
     the repo's `ecs-task-def.json` is gitignored), renders it with the new
     image, and deploys to service `fluiq-api` on cluster `fluiq`;
   - waits for service stability (`wait-for-service-stability: true`).
2. On boot the API applies both `schema.sql` files (Postgres + ClickHouse) and
   seeds judge-prompt defaults (`seed_judge_prompts()`), so schema changes land
   automatically with the deploy.

**Manual**
- Actions tab → **Deploy to Amazon ECS** → **Run workflow** (`workflow_dispatch`).

**Verify**
- `aws ecs describe-services --cluster fluiq --services fluiq-api` → running
  count matches desired and the deployment `rolloutState` is `COMPLETED`.
- Hit a health endpoint / a changed route through the public API.

**Rollback**
- Re-run the deploy from the previous green commit, or in the ECS console pick a
  prior task-definition revision and **Update service**.

---

## Infrastructure — messaging, data, scaling

**Kafka (self-hosted EC2, replaces MSK).** MSK was ~87% of the AWS bill, so the
broker moved to a single-node Kafka (KRaft, Docker) on a `t4g.medium` EC2
(`172.31.13.149:9092`), private VPC, PLAINTEXT, SG-locked to ECS. Provisioning +
cutover + topic scripts are in `kafka-ec2/` (`user-data.sh`, `create-topics.sh`,
`cutover.sh`). Config is centralized in SSM (`/fluiq/prod/KAFKA_*`), so cutover is
a param overwrite + `force-new-deployment` (no task-def/code change). The MSK
cluster + configuration are deleted.

**Data stores (single-node EC2 / RDS).** ClickHouse on `t4g.medium` EC2
(`172.31.1.180:8123`), Postgres on RDS `db.t4g.micro`, Redis on ElastiCache
`t4g.micro` — all private, SG → ECS only. Chosen tier is elasticity-only (no
multi-node HA) to stay under the ~$150/mo budget; resilience is EC2 auto-recovery
+ DLM EBS snapshots.

**ECS autoscaling (Application Auto Scaling).** All four services scale on CPU
(target 65%) but never below 1 (always a warm task — Docker cold-start is too
slow to scale from 0): `fluiq-api` 1→4, `fluiq-tracer`/`fluiq-evaluator` 1→3,
`fluiq-security` 1→2, on-demand Fargate. Set up via `autoscale/setup-elasticity.sh`.

**Admin infra logs.** The Admin console tails Kafka (`docker logs` via SSM),
ClickHouse (`system.text_log`/`query_log`), and Postgres (RDS log API). These need
the extra IAM in `fluiq-api-infra-admin-policy.json` (ssm:SendCommand /
GetCommandInvocation, rds:DescribeDBLogFiles / DownloadDBLogFilePortion) on the
`fluiqECSTaskRole`; endpoints fail soft until it's attached.

**Shared with Infrager.** Infrager is a separate product (own repo, own domain)
but it borrows fluiq infrastructure rather than duplicating it: the `fluiq` ECS
cluster, the `fluiq-api-alb`, the `fluiq-postgres` RDS instance, and the ECS
security group. Anyone changing those four should know a second product depends
on them; the exact resources are listed under [Infrager](#infrager).

**Schema deploy ordering.** `_apply_schema()` runs on API boot (idempotent
`CREATE/ALTER … IF NOT EXISTS`), so **deploy the API before the workers**. New
columns/tables must exist before a worker's by-name insert references them:
`traces.retention_days` / `is_root` / `agent_key` / `agent_kind` (before the
tracer), the four `*_rollup` tables + MVs, `dataset_trajectory_spans`, and the
`users.trial_ends_at` / `trial_used` columns (before the API serves trials).
Dataset trajectory media is offloaded to the private S3 bucket (reuses the blog
task role). Roll-up tables are seeded from history once with
`db_queues/clickhouse/rollup_backfill.sql` (TRUNCATE + re-insert to avoid
double-counting if the MVs already captured recent rows).

---

## Python SDK → PyPI

`fluiq-sdk/.github/workflows/release.yaml` publishes `fluiq` to PyPI on a semver
git tag. **The tag name is the version** — it is `sed`-written into
`pyproject.toml` during the build, so you do not bump the version in code.

**Steps**
1. Land your SDK changes on `main`.
2. Choose the next version (current: `0.1.2`). Create and push the tag from the
   commit you want to ship:
   ```bash
   git tag 0.1.3            # or 0.1.3rc1 / 0.1.3b1 / 0.1.3a1 for pre-releases
   git push origin 0.1.3
   ```
   Accepted tag patterns: `X.Y.Z`, `X.Y.Za N`, `…b N`, `…rc N`.
3. The workflow: verifies the version is newer than PyPI, builds the sdist+wheel,
   publishes via **Trusted Publishing (OIDC)** to the `release` environment, and
   creates a GitHub Release with auto-generated notes.

**Verify**
- `pip index versions fluiq` (or the PyPI project page) shows the new version.
- `pip install --upgrade fluiq` then `python -c "import fluiq, fluiq.config"`.

**Notes**
- The job **fails fast** if the tag's version is not strictly greater than the
  latest on PyPI — bump the tag, don't reuse one.

---

## TypeScript SDK → npm

`fluiq-sdk-typescript/.github/workflows/publish.yml` publishes `@fluiq/sdk` to
npm via **Trusted Publishing (OIDC)** when a **GitHub Release is published**.
Unlike the Python SDK, the npm version is **not** derived from the tag — you must
bump `package.json` and commit it first.

**Steps**
1. Bump the version in `fluiq-sdk-typescript/package.json` (current: `0.1.2`) and
   commit it to `main`:
   ```bash
   cd fluiq-sdk-typescript
   npm version patch --no-git-tag-version   # or edit "version" by hand
   git commit -am "sdk-ts: v0.1.3"
   git push
   ```
2. Create a **GitHub Release** (which creates the tag) for that commit — e.g.
   ```bash
   gh release create ts-v0.1.3 --title "TS SDK 0.1.3" --generate-notes
   ```
   The `release: published` event fires `publish.yml`, which: installs npm@latest
   (Trusted Publishing needs npm ≥ 11.5.1), runs `npm install` (the lockfile is
   **not** committed, so `install`, not `ci`), `npm run build`, then `npm publish`.
3. Or run it manually: Actions tab → **Publish to npm** → **Run workflow**
   (`workflow_dispatch`) — still requires the committed version bump.

**Verify**
- `npm view @fluiq/sdk version` shows the new version (and a provenance badge).
- `npm install @fluiq/sdk@latest` in a scratch project; `import fluiq from "@fluiq/sdk"`.

**Notes**
- npm rejects a re-publish of an existing version — always bump `package.json`.
- `prepublishOnly` re-runs `clean` + `build`; the workflow also builds first so
  failures surface before the publish step.

---

## Workers → Amazon ECS

Each worker (`fluiq-workers/{tracer,evaluator,security}`) has its own
`deploy.yml` mirroring the API flow (build → ECR → render live task def → ECS
deploy → wait for stability) against its own ECR repo / ECS service. Push to
`main` deploys all changed workers; use each workflow's `workflow_dispatch` to
redeploy one. See `docs/workers.md` for shared topics/auth, and
`docs/tracer.md` / `docs/evaluator.md` / `docs/security.md` for per-worker env vars.

---

## <a id="infrager"></a>Infrager

Separate product, separate repo (`SaurabhKumbhar24/Infrager`), separate domain,
but it runs on fluiq's infrastructure to avoid paying twice. npm-workspaces
monorepo: `apps/web` (Next.js, the canvas/editor) and `apps/api` (Express +
Postgres). Codegen and security linting run entirely in the browser, so the API
only stores diagrams.

**Web → Amplify.** App `d1icguwdl9x4aw`, platform **WEB_COMPUTE** (the editor is
a dynamic `/editor/[id]` route, so static hosting fails). `amplify.yml` sits at
the repo root with `appRoot: apps/web`; artifact `baseDirectory` is `.next`,
resolved **relative to appRoot**, not the repo root. Build-time env var
`NEXT_PUBLIC_API_URL=https://infrager-api.getfluiq.com` is inlined into the
bundle, so changing the API host requires a rebuild, not just a restart.

**API → ECS.** Cluster `fluiq`, service `infrager-api`, task family
`infrager-api` (256 CPU / 512 MB, Fargate **Spot**, 1 task). Image
`383136686684.dkr.ecr.us-east-2.amazonaws.com/infrager-api`, built from
`apps/api/Dockerfile` with the **repo root as build context** (npm workspaces
need the root lockfile). Build for `linux/amd64` with `--provenance=false`;
a multi-platform manifest or an arm64 image fails to start on the x86 task.
There is no GitHub Actions workflow yet, so releases are manual:

```bash
aws ecr get-login-password --region us-east-2 | docker login --username AWS \
  --password-stdin 383136686684.dkr.ecr.us-east-2.amazonaws.com
docker build --platform linux/amd64 --provenance=false \
  -f apps/api/Dockerfile -t <ecr>/infrager-api:latest .
docker push <ecr>/infrager-api:latest
aws ecs update-service --cluster fluiq --service infrager-api --force-new-deployment
```

**Fluiq resources it borrows.** Change any of these with Infrager in mind:

| Resource | How Infrager uses it |
|----------|---------------------|
| RDS `fluiq-postgres` | Own database + role `infrager`, PUBLIC revoked. No grants on fluiq schemas. |
| ECS cluster `fluiq` | Runs `infrager-api` alongside the fluiq services. |
| `fluiq-api-alb` | Listener rule priority 20 on host `infrager-api.getfluiq.com` → target group `infrager-api-tg`. Needed its **own ACM cert** attached via SNI; the listener's original cert only covered `api.getfluiq.com`. |
| SG `sg-041902003fb6ef7f9` (`fluiq-ecs-sg`) | The API task runs in it, which is why it already reaches RDS. Added ALB → ECS ingress on **4000**. |
| `fluiqECSTaskExecutionRole` | Extra inline policy `infrager-ssm-secrets-access` for `/infrager/prod/*`. |

**Config** lives in SSM as SecureStrings: `/infrager/prod/DATABASE_URL` and
`/infrager/prod/AUTH_SECRET`, injected as task `secrets`. Plain env on the task:
`PORT=4000`, `PGSSL=require`, `CORS_ORIGINS=https://infrager.getfluiq.com`
(an exact origin match; a mismatch here shows up as browser CORS failures on
signup, not as an API error). Tables are created on boot, so there is no
migration step. Logs: `/ecs/infrager-api`, 14-day retention.

**Marketing surface.** `getfluiq.com/infrager` (see `docs/frontend.md`) plus the
Developer nav dropdown, homepage section, footer, and sitemap entries.

---

## <a id="feature-note-custom-judges"></a>Feature note — Custom client judges

The custom-judges feature spans the API, the evaluator worker, and a DB column,
so order the rollout:

1. **Deploy the API first.** Startup applies the `prompts.kind` migration
   (`ALTER TABLE prompts ADD COLUMN IF NOT EXISTS kind …`) and exposes the
   block-mode `/evaluate` `custom_judges` path.
2. **Ensure `POSTGRES_DSN` is set on the evaluator ECS task,** then deploy the
   evaluator worker. Warn-mode custom judges resolve `kind='judge'` rows from
   Postgres; without the DSN they are silently skipped (built-in metrics still
   run). See `docs/evaluator.md`.
3. **Ship the SDKs** (Python `custom_judges`, TS `customJudges`) so clients can
   reference judges by slug. Existing SDK builds keep working — the field is
   optional and fail-open.
4. **Frontend** (Amplify) gains the Prompts → Save-as-**Judge** toggle; deploy
   any time after the API.

---

## <a id="feature-note-agentic-eval"></a>Feature note — Agentic evaluator

Spans the SDK, API, evaluator worker, ClickHouse, Postgres, and frontend. Order:

1. **Deploy the API first.** Startup `_apply_schema()` runs the idempotent
   `ALTER TABLE fluiq.evaluations ADD COLUMN IF NOT EXISTS layer / step_id /
   run_score / run_passed`, and it seeds the 3 new judge prompts
   (`tool_selection_quality`, `trajectory_quality`, `agent_coordination`). The API
   also exposes `/evaluate/agentic` + `/evaluate/agentic-summary` and the
   per-LLM-call auto-eval fan-out (`EVAL_AUTO_SAMPLE_RATE`).
2. **Deploy the evaluator worker** — its by-name `insert_evaluation()` references
   the new columns, so it must go **after** the API. Optional agentic config:
   `EVAL_AGENT_DEPTH`, `EVAL_PANEL_MODE`, `EVAL_PANEL_GATE_MARGIN`,
   `EVAL_PANEL_MEMBERS` (see `docs/evaluator.md`).
3. **Ship the SDKs** — join-node `parent_ids` emission (LangGraph / CrewAI /
   Google ADK) + the generic `fluiq.join_parents(...)`. All optional / fail-open,
   so older builds keep working.
4. **Frontend** (Amplify) gains the drawer **Run Agentic Eval** button (root
   traces), the per-layer agentic rendering, and the Overview agentic tile; deploy
   any time after the API.

Calibration (`python -m jobs.calibration.runner`) and harvesting
(`python -m jobs.calibration.harvest`) are operator CLIs run in the evaluator
image — not part of the request path.
