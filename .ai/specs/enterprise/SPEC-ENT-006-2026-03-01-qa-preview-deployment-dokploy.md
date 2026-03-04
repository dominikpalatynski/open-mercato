# SPEC-ENT-006: QA Deployment on Dokploy via GitHub Actions

## TLDR

**Key Points:**
- Introduce a manual `workflow_dispatch` GitHub Actions workflow (`.github/workflows/qa.yml`) that builds the preview Docker image, pushes it to GHCR, and deploys it to a named QA slot on Dokploy via REST API.
- Each QA slot (e.g. `qa1`) maps to a pre-configured Dokploy application running as a Docker-provider app. Slots are long-lived environments, not ephemeral per-PR.
- The workflow can be triggered manually from the GitHub Actions UI for any branch, tag, or SHA.

**Scope:**
- `.github/workflows/qa.yml` — manual deployment workflow
- `docker/preview/Dockerfile` — dedicated preview image used for QA builds
- `docker/preview/preview-entrypoint.sh` — baked into the image; invokes `yarn test:integration:ephemeral:start`
- Fix `NODE_ENV` passthrough in `packages/cli/src/lib/testing/integration.ts`
- Dokploy application configuration (Docker provider, not GitHub source)
- GitHub repository secrets and variables for GHCR + Dokploy API

**Concerns:**
- docker.sock mount gives the Dokploy container access to the host Docker daemon — acceptable for internal QA; must not be used in production.
- First-ready latency is ~5–10 minutes per deployment (full install → generate → build → DB init → Next.js start).
- Concurrent slot deployments are serialised per slot via GitHub Actions `concurrency` group (`cancel-in-progress: false`). A second run for the same slot queues behind the first.

---

## Overview

Open Mercato has no automated QA preview environment. Developers must run the full stack locally or share a single staging server. This spec introduces named QA slot deployments: a developer manually triggers the `Deploy to Dokploy QA` workflow from the GitHub Actions UI, selects a slot (`qa1`), optionally specifies a branch/tag/SHA and an image tag override, and the workflow builds the preview image, pushes it to GHCR, updates the Dokploy application's Docker image, and triggers a redeploy. The environment uses an ephemeral PostgreSQL container (docker.sock) and resets on every deployment.

---

## Problem Statement

- No automated QA environment exists. Testing a PR requires local setup or a shared staging server that gets overwritten.
- Reviewers cannot verify a feature without checking out the branch locally.
- Setting up the full stack locally takes 20–30 minutes for new contributors.

---

## Proposed Solution

A manually-triggered GitHub Actions workflow builds and deploys the `docker/preview/Dockerfile` image to a pre-configured Dokploy slot. The workflow:

1. Checks out the requested ref (branch / tag / SHA; defaults to the triggering branch).
2. Builds the preview Docker image using `docker/preview/Dockerfile` via `docker/build-push-action` with GitHub Actions cache (`type=gha`).
3. Pushes the image to GHCR as `ghcr.io/<org>/<repo>:<slot>-<sha7>` (or a custom tag override).
4. Calls the Dokploy REST API (`application.saveDockerProvider`) to update the application's Docker image reference to the newly pushed tag.
5. Calls the Dokploy REST API (`application.deploy`) to trigger a redeploy of the slot.

The Dokploy application is configured as a **Docker provider** app (not a GitHub source app), so Dokploy pulls the already-built image from GHCR rather than building it itself.

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| Manual `workflow_dispatch` (not automatic webhook/label trigger) | Full developer control over when and what gets deployed to QA. Avoids race conditions from concurrent label events on multiple PRs. |
| Pre-build image in CI, push to GHCR, then tell Dokploy | Decouples build from deployment. CI has access to build cache (`type=gha`); Dokploy does not need GitHub App access. Cleaner separation of concerns. |
| Named slots (`qa1`, `qa2`) instead of per-PR ephemeral URLs | Predictable URLs, predictable resource usage, easier to share with stakeholders. Slots are long-lived Dokploy applications. |
| Dokploy REST API (`saveDockerProvider` + `deploy`) | No Dokploy GitHub App installation required. Works with any Git hosting or manual trigger. Simpler ops. |
| `slot-<sha7>` default image tag | Uniquely identifies the deployed build without manual tag management. Override available for rollbacks. |
| `cancel-in-progress: false` concurrency | Queues concurrent deploys to the same slot rather than cancelling in-flight builds; prevents partial deployments. |
| Standalone `docker/preview/Dockerfile` | Keeps preview tooling (docker-cli, jest config) isolated from the main production `Dockerfile`. |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| Dokploy native GitHub webhook (per-PR label trigger) | More complex to configure, requires Dokploy GitHub App installation, harder to control. Manual trigger is sufficient for QA. |
| Single shared `qa.openmercato.com` with Dokploy GitHub source | Concurrent deploys overwrite each other; no image caching across builds. |
| Persistent PostgreSQL between deploys | State drift makes tests non-deterministic. Ephemeral DB is consistent with existing `test:integration:ephemeral:start` design. |

---

## User Stories / Use Cases

- **Developer** wants to deploy a specific branch to QA manually so that a reviewer can test the feature at a stable URL without cloning the repo.
- **Developer** wants to override the image tag so that they can redeploy a previously built image for a rollback.
- **Reviewer** wants to know the QA slot URL in advance so that they can open it directly from a shared link.
- **DevOps** wants deployments to a slot to be serialised so that a second trigger does not corrupt a running deployment.

---

## Architecture

```
Developer triggers workflow_dispatch
    │  inputs: slot=qa1, ref=<branch/sha>, image_tag=<optional>
    ▼
[GitHub Actions: .github/workflows/qa.yml]
    │
    ├── Checkout ref
    │
    ├── docker/build-push-action
    │       file: docker/preview/Dockerfile
    │       context: .
    │       push: true
    │       platforms: linux/amd64
    │       tags: ghcr.io/<org>/<repo>:<slot>-<sha7>
    │       cache-from/to: type=gha
    │
    ├── POST /api/application.saveDockerProvider
    │       { applicationId, dockerImage: "ghcr.io/...:<tag>" }
    │
    └── POST /api/application.deploy
            { applicationId }
                │
                ▼
        [Dokploy Server]
            └── docker pull ghcr.io/...:<tag>
            └── docker run
                    ├── /var/run/docker.sock:/var/run/docker.sock (bind)
                    ├── ENV from Dokploy UI
                    └── docker/preview/preview-entrypoint.sh
                            └── yarn test:integration:ephemeral:start
                                    ├── yarn install
                                    ├── yarn build:packages
                                    ├── yarn generate
                                    ├── yarn build:packages (2nd pass)
                                    ├── docker run postgres:16 (via docker.sock)
                                    ├── yarn initialize --reinstall
                                    ├── yarn build:app
                                    └── yarn start  (PORT=3000)
```

### Slot → Application ID Mapping

The workflow resolves the Dokploy `applicationId` from GitHub repository variables:

| Slot | Variable |
|------|----------|
| `qa1` | `vars.DOKPLOY_APP_ID_QA1` |
| `qa2` | `vars.DOKPLOY_APP_ID_QA2` |

---

## Data Models

No new database entities. Deployment state is managed entirely by Dokploy.

---

## API Contracts

### Dokploy REST API calls (outbound from GitHub Actions)

**`POST /api/application.saveDockerProvider`**
```json
{
  "applicationId": "<string>",
  "dockerImage": "ghcr.io/<org>/<repo>:<tag>"
}
```

**`POST /api/application.deploy`**
```json
{
  "applicationId": "<string>"
}
```

Both calls require `x-api-key: <DOKPLOY_API_KEY>` and `Content-Type: application/json`.

---

## Configuration

### GitHub Repository Secrets

| Secret | Description |
|--------|-------------|
| `DOKPLOY_URL` | Base URL of the Dokploy server (e.g. `https://dokploy.example.com`) |
| `DOKPLOY_API_KEY` | Dokploy API key with deploy permissions |

### GitHub Repository Variables

| Variable | Description |
|----------|-------------|
| `DOKPLOY_APP_ID_QA1` | Dokploy `applicationId` for the `qa1` slot |
| `DOKPLOY_APP_ID_QA2` | Dokploy `applicationId` for the `qa2` slot (optional) |

### GitHub Actions Permissions (workflow-level)

```yaml
permissions:
  contents: read
  packages: write   # required to push to GHCR
```

### Required Dokploy Application Settings (per slot)

| Setting | Value |
|---------|-------|
| Source type | Docker (not GitHub) |
| Docker image | `ghcr.io/<org>/<repo>:<latest-deployed-tag>` (updated by workflow) |
| Exposed port | `3000` |
| Volume — Source | `/var/run/docker.sock` |
| Volume — Destination | `/var/run/docker.sock` |
| Volume — Type | Bind |
| Restart policy | `no` |

---

## Implementation Plan

### Phase 1: Standalone Preview Dockerfile ✅

**Goal**: Create a dedicated `docker/preview/Dockerfile` for QA builds, fully isolated from the main `Dockerfile`.

#### Steps

1. **Create `docker/preview/Dockerfile`** — two-stage build (`builder` + `runner`):
   - `builder` stage: installs deps and runs `yarn build:packages`
   - `runner` stage: installs `docker-cli` (needed to spawn the ephemeral PostgreSQL container via docker.sock), copies the entrypoint script, runs `yarn install` + `yarn build:packages`

2. **Create `docker/preview/preview-entrypoint.sh`** — baked into the `runner` stage; calls `yarn test:integration:ephemeral:start`.

3. **Verify local build** — `docker build -f docker/preview/Dockerfile -t open-mercato:preview .` — must complete without errors.

#### File Manifest

| File | Action | Purpose |
|------|--------|---------|
| `docker/preview/Dockerfile` | Create | Standalone preview image (builder + runner stages) |
| `docker/preview/preview-entrypoint.sh` | Create | Entrypoint that invokes `yarn test:integration:ephemeral:start` |

---

### Phase 2: Fix NODE_ENV Passthrough ✅

**Goal**: Confirm that `yarn test:integration:ephemeral:start` inherits `NODE_ENV=production` from the container environment.

#### Steps

1. **Fix `NODE_ENV` passthrough in `packages/cli/src/lib/testing/integration.ts`** — change the two hardcoded `NODE_ENV: 'test'` assignments to `NODE_ENV: process.env.NODE_ENV ?? 'test'`. This allows the ephemeral environment to inherit `NODE_ENV=production` set in the Dokploy container, while still defaulting to `'test'` in CI/local runs.

#### File Manifest

| File | Action | Purpose |
|------|--------|---------|
| `packages/cli/src/lib/testing/integration.ts` | Modify | `NODE_ENV: process.env.NODE_ENV ?? 'test'` (two occurrences) |

---

### Phase 3: GitHub Actions Workflow ✅

**Goal**: Create `.github/workflows/qa.yml` implementing the manual build-push-deploy pipeline.

#### Steps

1. **Create `.github/workflows/qa.yml`** with:
   - Trigger: `workflow_dispatch` with inputs `slot`, `ref`, `image_tag`
   - Concurrency: `dokploy-<slot>`, `cancel-in-progress: false`
   - Steps: checkout → QEMU → Buildx → GHCR login → compute tags → build+push → resolve app ID → `saveDockerProvider` → `deploy` → summary

2. **Add required secrets** to the GitHub repository: `DOKPLOY_URL`, `DOKPLOY_API_KEY`.

3. **Add required variables** to the GitHub repository: `DOKPLOY_APP_ID_QA1`.

#### File Manifest

| File | Action | Purpose |
|------|--------|---------|
| `.github/workflows/qa.yml` | Create | Manual QA deployment workflow |

---

### Phase 4: Dokploy Application Configuration

**Goal**: Create and configure the QA slot application(s) in Dokploy. This phase is a manual ops runbook.

#### Steps

1. **Create a new Application** in Dokploy for each slot:
   - Name: `open-mercato-qa1` (repeat for `qa2` if needed)
   - Source type: Docker
   - Docker image: set to any valid placeholder initially; the workflow will overwrite it on first run
   - Exposed port: `3000`

2. **Add volume mount** (docker.sock):
   - Application → Advanced → Volumes
   - Source: `/var/run/docker.sock` / Destination: `/var/run/docker.sock` / Type: `Bind`

3. **Set environment variables** from the Configuration table above via Application → Environment.

4. **Copy the `applicationId`** from Dokploy (Application → Settings → General) and set it as `DOKPLOY_APP_ID_QA1` in GitHub repository variables.

5. **Configure domain** for the slot (e.g. `qa1.openmercato.com`) in Dokploy → Domains. Requires a DNS A record pointing to the Dokploy server IP.

#### File Manifest

No files created. This phase is entirely Dokploy and DNS configuration.

---

### Phase 5: End-to-End Validation

**Goal**: Confirm the full pipeline works from workflow trigger to accessible QA URL.

#### Steps

1. Trigger the `Deploy to Dokploy QA` workflow from GitHub Actions UI with `slot=qa1` and a target branch.
2. Confirm the build completes and the image is pushed to GHCR.
3. Confirm Dokploy receives the deploy trigger and starts pulling the new image (visible in Dokploy → Deployments).
4. Wait ~10 minutes for the full startup sequence (install → generate → build → DB init → Next.js start).
5. Confirm the QA slot URL (e.g. `https://qa1.openmercato.com/backend`) is accessible and the backend login page loads.
6. Trigger a second deployment to the same slot. Confirm the first completes before the second starts (concurrency serialisation).
7. Trigger a deployment with a custom `image_tag` override. Confirm the correct image is deployed.

---

## Risks & Impact Review

### Data Integrity Failures

The QA environment is fully ephemeral and self-contained. No production data is involved. Risk is isolated to the QA slot.

### Cascading Failures & Side Effects

- **GHCR push failure**: The `build-push-action` step fails; downstream Dokploy steps are skipped. Re-trigger the workflow.
- **Dokploy API unavailable**: `saveDockerProvider` or `deploy` curl call fails (`-fsS` flags cause non-zero exit). Workflow fails with clear error. Re-trigger after resolving Dokploy connectivity.
- **docker.sock mount failure**: If the host Docker daemon is unavailable, `yarn test:integration:ephemeral:start` fails at the PostgreSQL startup step. The container exits with a non-zero code. Dokploy marks the deployment as failed. Re-trigger the workflow.

### Migration & Deployment Risks

- `docker/preview/Dockerfile` is standalone and entirely separate from the main `Dockerfile`. Existing `builder`, `dev`, and `runner` stages in the main Dockerfile are unchanged.
- The `NODE_ENV: process.env.NODE_ENV ?? 'test'` change in `integration.ts` is backward-compatible: CI and local runs that do not set `NODE_ENV` continue to default to `'test'`.
- No database migrations in Open Mercato's production database are involved.
- The workflow only has `contents: read` and `packages: write` permissions — it cannot modify repository settings or other GitHub resources.
