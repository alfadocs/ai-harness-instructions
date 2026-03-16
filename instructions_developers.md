## AI Harness Pipelines & Supabase Infrastructure

This document explains how the control-plane in this repo works, how it interacts with target apps (Lovable/Vibe-generated projects), and what developers need to know to support and extend it.

It covers:

- Guardrails pipeline (security/quality)
- Stable branch promotion
- Deploy pipeline (App Runner + Supabase)
- Per-app Supabase projects
- Required secrets and environment variables

---

## High-level architecture

- **Control-plane repo (this repo)**:
  - Owns all reusable workflows, scripts, and infra glue.
  - Receives `repository_dispatch` events from a GitHub App via an AWS Lambda.
  - Builds and deploys target apps to AWS App Runner.
  - Manages per-app Supabase projects (creation, migrations, edge functions).

- **Target repos (Lovable/Vibe apps)**:
  - Normal app code (Vite/React/etc.).
  - Pushes to `main` trigger guardrails.
  - Clean `main` gets promoted to `stable`, which is the deploy branch.

- **Supabase (per app)**:
  - Each target repo gets its own **Supabase project** for production.
  - Control-plane ensures the project exists and keeps migrations/functions up to date.

- **AWS App Runner (per app)**:
  - Runs each frontend as a static site (built via the shared `Dockerfile`).
  - Container image built in this repo’s CI; App Runner just pulls and runs it.

---

## 1. Guardrails pipeline

**Workflow:** `.github/workflows/guardrails.yml`  
**Dispatcher:** `functions/guardrails-dispatcher-lambda/src/handler.ts`  
**Guardrail scripts:** `scripts/guardrail/*`

### Trigger flow

1. The **Guardrails GitHub App is installed at the org level**.
   - The dispatcher only triggers pipelines for repos that were created by an allowed agent (currently Lovable), so not every org repo is affected.
2. A developer pushes to `main`:
   - GitHub sends a webhook to the **guardrails-dispatcher Lambda**.
3. The Lambda:
   - Verifies the webhook signature.
   - Filters events (only `push` to `main` for guardrails).
   - Resolves which platform created the repo (Lovable, etc.).
   - Sends a `repository_dispatch` to **this repo** with:
     - `event_type: run-guardrails`
     - `client_payload: { owner, repo, sha, ref, installation_id, ai_agent }`
4. This control-plane repo’s `guardrails.yml` workflow starts.

### What `guardrails.yml` does

- Checks out this control-plane repo.
- Uses the GitHub App token to check out the **target repo** at the specific SHA under `./remote`.
- Runs `workflow/install-dependencies.sh` to install tools (Semgrep, ESLint, Gitleaks, etc.).
- Runs the main guardrail entrypoint:
  - `scripts/guardrail/entrypoint.sh` → orchestrates Semgrep, ESLint, Gitleaks, etc.
- On **failure**:
  - Copies results into the target repo checkout for Codex.
  - Checks if there’s already an open “Codex fix” PR.
  - If not, runs `openai/codex-action` to auto-fix issues and `create-codex-fix-pr.js` to open a PR in the target repo.
  - Cleans up guardrail artifacts.
  - Creates a check run on the target repo so branch protection can require “Guardrails passed”.
- On **success**:
  - Marks the check run as successful.
  - Calls `scripts/guardrail/dist/promote-to-stable.js` to promote the passing code to `stable`.

### Promotion to `stable`

**Script:** `scripts/guardrail/src/promote-to-stable.ts`

Given `OWNER`, `REPO`, `SHA`, and `REF`:

- If the **`stable` branch does not exist** in the target repo:
  - Creates `refs/heads/stable` at the current SHA (first successful run seeds `stable`).
- If **`stable` exists**:
  - Finds an open PR from the current branch (`REF`) to `stable`.
  - If none exists:
    - Creates a PR `Promote <branch> to stable` with base = `stable`, head = `<branch>`.

This ensures:

- `main` remains the fast-moving dev branch.
- `stable` only receives code that passed guardrails and was explicitly merged (via PR) into `stable`.

---

## 2. Guardrails dispatcher Lambda

**Location:** `functions/guardrails-dispatcher-lambda`

### Purpose

- Centralizes GitHub webhook handling and dispatching to the control-plane.
- Avoids duplicating workflow logic in every target repo.

### Key behavior (`src/handler.ts`)

- Validates `X-Hub-Signature-256` using `GITHUB_WEBHOOK_SECRET`.
- Reads `X-GitHub-Event`:
  - Ignores `pull_request` events (we only use `push`).
  - For `push`:
    - Extracts `ref` (branch) and `after` (SHA).
    - For `refs/heads/main` or `refs/heads/master`:
      - Triggers `repository_dispatch: run-guardrails` in this repo.
    - For `refs/heads/stable`:
      - Triggers `repository_dispatch: deploy-stable` in this repo.
    - Other branches are logged and ignored.
- Uses the GitHub App to get an **installation token** for the target repo, then calls:
  - `triggerControlPlaneDispatch(installationToken, payload, eventType)`:
    - `eventType` = `"run-guardrails"` or `"deploy-stable"`.
    - `client_payload` includes:
      - `owner`, `repo`, `sha`, `ref`, `pr_number`, `installation_id`, `ai_agent`, `repository: owner/repo`.

### Configuration

Set via Lambda environment variables:

- `GITHUB_WEBHOOK_SECRET`
- `GITHUB_APP_ID`
- `GITHUB_APP_PRIVATE_KEY` or `GITHUB_APP_PRIVATE_KEY_SECRET_ARN`
- `CONTROL_PLANE_OWNER`, `CONTROL_PLANE_REPO`
- `DISPATCH_EVENT_TYPE` (defaults to `run-guardrails` but we now override per call)
- `ALLOWED_REPO_CREATORS` (optional allowlist: e.g. `lovable-dev[bot]`)

---

## 3. Deploy pipeline

**Workflow:** `.github/workflows/deploy-stable.yml`  
**Shared scripts:** `scripts/guardrail/src/*`  
**Shared Dockerfile:** `Dockerfile` at repo root

### Trigger flow

1. A target repo merges into `stable` (or pushes to `stable`).
2. GitHub sends a `push` webhook to the dispatcher Lambda.
3. The Lambda sees `ref == refs/heads/stable` and calls:
   - `repository_dispatch` with `event_type: deploy-stable` and `client_payload.repository = "owner/repo"`.
4. This repo’s `deploy-stable.yml` workflow runs.

### What `deploy-stable.yml` does

1. **Setup**
   - Checks out this control-plane repo.
   - Installs and builds `scripts/guardrail` TypeScript helpers.

2. **Ensure a Supabase project exists for this app**

   **Script:** `scripts/guardrail/src/ensure-supabase-project.ts`

   - Inputs:
     - `TARGET_REPOSITORY` (e.g. `org/my-app`) from `client_payload.repository`.
     - Env vars:
       - `SUPABASE_ACCESS_TOKEN` (GitHub secret)
       - `SUPABASE_ORG_SLUG` (GitHub secret/variable)
       - `SUPABASE_DB_PASSWORD` (GitHub secret)
       - `SUPABASE_DB_REGION` (GitHub variable, e.g. `eu-central-1`)
   - Behavior:
     - Derives project name: `owner-repo-prod`.
     - Calls Supabase Management API:
       - `GET /v1/projects` → looks for project with that name and org slug.
       - If found → returns its `ref`.
       - If not → `POST /v1/projects` to create it, then returns the `ref`.
   - The workflow captures this as `supabase_project.outputs.project_ref`.

3. **Verify guardrails passed for `stable`**

   - Uses `verify-guardrails-passed.js` to check the target repo’s latest guardrails status.
   - Ensures we don’t deploy code that failed guardrails.

4. **Checkout target repo at `stable`**

   - Uses `actions/checkout` with:
     - `repository = TARGET_REPOSITORY`
     - `ref = stable`
     - `path = app`

5. **Run Supabase migrations and deploy edge functions**

   - Installs Supabase CLI (`supabase/setup-cli@v1`).
   - In `./app`:
     ```bash
     supabase login --token "$SUPABASE_ACCESS_TOKEN"
     supabase link --project-ref "$SUPABASE_PROJECT_REF"
     supabase db push
     supabase functions deploy
     ```
   - This:
     - Links the local `supabase/` folder to the per-app project.
     - Applies DB migrations.
     - Deploys Supabase Edge Functions from the target repo to the correct project.

6. **Fetch publishable API key and URL for the frontend**

   - Still in the workflow (control-plane repo):
     ```bash
     supabase login --token "$SUPABASE_ACCESS_TOKEN"
     PUBLISHABLE_KEY=$(supabase projects api-keys --project-ref "$SUPABASE_PROJECT_REF" --output json \
       | jq -r '.[] | select(.type == "publishable") | .api_key' \
       | head -n 1)

     SUPABASE_URL="https://$SUPABASE_PROJECT_REF.supabase.co"

     echo "VITE_SUPABASE_KEY=$PUBLISHABLE_KEY" >> "$GITHUB_ENV"
     echo "VITE_SUPABASE_URL=$SUPABASE_URL" >> "$GITHUB_ENV"
     ```
   - This makes `VITE_SUPABASE_URL` and `VITE_SUPABASE_KEY` available to the Docker build step.

7. **Build and push the container image**

   - Environment in build step:
     - `TARGET_REPOSITORY` (e.g. `org/my-app`)
     - `VITE_SUPABASE_URL`, `VITE_SUPABASE_KEY`
   - Commands:
     ```bash
     APP_BASE_NAME=$(echo "${TARGET_REPOSITORY}" | tr '/._' '-' | tr '[:upper:]' '[:lower:]')
     APP_IMAGE_TAG="${APP_BASE_NAME}-latest"

     docker build --build-arg APP_DIR=app \
       --build-arg VITE_SUPABASE_URL="$VITE_SUPABASE_URL" \
       --build-arg VITE_SUPABASE_KEY="$VITE_SUPABASE_KEY" \
       -t "$ECR_REPOSITORY:$IMAGE_TAG" \
       -t "$ECR_REPOSITORY:$APP_IMAGE_TAG" .

     docker push "$ECR_REPOSITORY:$IMAGE_TAG"
     docker push "$ECR_REPOSITORY:$APP_IMAGE_TAG"
     ```

8. **Deploy to AWS App Runner**

   **Script:** `scripts/guardrail/src/deploy-to-apprunner.ts`

   - Derives service name: `ai-harness-<owner-repo>` (truncated to 40 chars).
   - Checks if an App Runner service exists:
     - If not → `CreateService` with the image `<ECR_REPOSITORY>:<baseName>-latest`.
     - If yes → `UpdateService` to the latest image.
   - Logs the service URL after deployment.

---

## 4. Shared Dockerfile for target apps

**Location:** `Dockerfile` at repo root

This Dockerfile is generic and used by all target apps.

### Build stage

- Base: `node:20-alpine`
- Args:
  - `APP_DIR` (path to the checked-out target repo; `deploy-stable` passes `app`).
  - `VITE_SUPABASE_URL`
  - `VITE_SUPABASE_KEY`
- Steps:
  - Install Bun/NPM/Yarn/PNPM deps based on lockfiles.
  - `COPY ${APP_DIR}/. .`
  - Set:
    ```dockerfile
    ENV NODE_ENV=production
    ENV VITE_SUPABASE_URL=$VITE_SUPABASE_URL
    ENV VITE_SUPABASE_KEY=$VITE_SUPABASE_KEY
    ```
  - Run build:
    ```dockerfile
    RUN if [ -f bun.lockb ]; then \
          /root/.bun/bin/bun run build; \
        else \
          npm run build; \
        fi
    ```

This is what causes `import.meta.env.VITE_SUPABASE_URL` / `VITE_SUPABASE_KEY` in the target app to point at the **per-app Supabase project in production**.

### Runtime stage

- Base: `node:20-alpine`
- Installs `serve` and serves `dist/` at port 3000:
  ```dockerfile
  COPY --from=builder /app/dist ./dist
  EXPOSE 3000
  CMD ["serve", "-s", "dist", "-l", "3000"]
  ```

---

## 5. Supabase configuration and secrets (per app)

Each per-app Supabase project (created by `ensure-supabase-project.ts`) has:

- Its own database and schema (kept in sync by `supabase db push`).
- Its own edge functions (from the target repo).
- Its own API keys (publishable and secret).
- Its own **edge function secrets**:
  - Via dashboard: `Project → Functions → Secrets`.
  - Or CLI (from the target repo):
    ```bash
    supabase login
    supabase link --project-ref <ref>
    supabase secrets set MY_SECRET=... OTHER_SECRET=...
    supabase secrets list
    ```
  - Access from functions via `Deno.env.get("MY_SECRET")`.

Frontend apps:

- Use `VITE_SUPABASE_URL` and `VITE_SUPABASE_KEY` baked into the bundle to create a Supabase client.
- Talk to Supabase Edge Functions via the client (`supabase.functions.invoke`) or direct `fetch` to:
  - `https://<project-ref>.functions.supabase.co/<function-name>`

---

## 6. What developers need to do (control-plane maintainers)

- Keep these secrets/vars configured in this repo (GitHub Settings → Secrets/Variables):
  - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
  - `ECR_REPOSITORY`, `APP_RUNNER_SERVICE_PREFIX`, `APP_RUNNER_ECR_ACCESS_ROLE_ARN`
  - `SUPABASE_ACCESS_TOKEN` (Management API)
  - `SUPABASE_ORG_SLUG`, `SUPABASE_DB_PASSWORD`, `SUPABASE_DB_REGION`
  - `GUARDRAILS_APP_ID`, `GUARDRAILS_APP_PRIVATE_KEY`
- Maintain the guardrail rules and scripts under `scripts/guardrail/`.
- Maintain the dispatcher Lambda and its env vars under `functions/guardrails-dispatcher-lambda/`.

With this setup, Vibe/Lovable-generated apps get:

- Automated security/quality guardrails on `main`.
- A clean `stable` branch for deployment.
- Per-app Supabase projects created on demand.
- Automatic DB migrations, edge function deployment, and App Runner rollout on every `stable` push.

