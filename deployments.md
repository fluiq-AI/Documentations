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
redeploy one. See `docs/workers.md` for env vars and topics.

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
   run). See `docs/workers.md`.
3. **Ship the SDKs** (Python `custom_judges`, TS `customJudges`) so clients can
   reference judges by slug. Existing SDK builds keep working — the field is
   optional and fail-open.
4. **Frontend** (Amplify) gains the Prompts → Save-as-**Judge** toggle; deploy
   any time after the API.
