# SPEC-ENT-006: QA Preview Deployment on Dokploy

## TLDR

**Key Points:**
- Introduce a `preview` Docker build target and wire Dokploy's native GitHub webhook integration to deploy a per-PR ephemeral Open Mercato environment automatically when a PR targeting `develop` is labelled `preview-env`.
- Each PR preview runs at a unique subdomain under `*.openmercato.com` (Dokploy-generated via Traefik). The environment spins up its own ephemeral PostgreSQL container via Docker-in-Docker (docker.sock mount) and resets on every deployment.

**Scope:**
- New `preview` stage in `Dockerfile`
- Verify `docker/scripts/preview-entrypoint.sh` works in baked-image (no volume mount) Dokploy context
- Verify `yarn test:integration:ephemeral:start` path correctly builds and starts
- Dokploy application configuration
- Wildcard DNS record `*.openmercato.com` → Dokploy server IP

**Concerns:**
- docker.sock mount gives the container access to the host Docker daemon — a known privilege escalation vector. Acceptable for an internal QA environment; must not be used in production.
- `yarn initialize --reinstall` runs on every container startup (full build + DB migration), meaning first-ready latency is ~5–10 minutes per deployment.
- Concurrent PRs labelled `preview-env` each get their own Dokploy preview instance — resource usage scales linearly with open PRs. Set a concurrency cap in Dokploy settings.

---

## Overview

Open Mercato currently has no automated QA preview environment. Developers and reviewers must run the full stack locally or share a single staging environment. This spec introduces per-PR ephemeral preview deployments: when a pull request targeting `develop` is labelled `preview-env`, Dokploy's native GitHub webhook integration automatically builds and deploys the branch to a unique `*.qa.openmercato.com` subdomain. The environment is fully self-contained — it includes its own PostgreSQL container, initialises from scratch, and is torn down when the PR is closed or the label is removed.

> **Market Reference**: Vercel's Preview Deployments (per-branch ephemeral URLs) and Railway's PR environments were studied. The ephemeral-postgres-per-deploy pattern is adopted from both. Vercel's approach is rejected because it requires managed infrastructure and does not support custom Docker targets. Railway is rejected for cost unpredictability at scale. Dokploy's open-source self-hosted model is chosen for full infrastructure control.

---

## Problem Statement

- No automated QA environment exists. Testing a PR requires local setup or a shared staging server that gets overwritten.
- Reviewers cannot verify a feature without checking out the branch locally.
- Shared staging environments cause race conditions and unclear ownership.
- Setting up the full stack locally (Docker, DB, env files) takes 20–30 minutes for new contributors.

---

## Proposed Solution

A `preview` Docker build stage is introduced in the `Dockerfile`. Dokploy's GitHub App is installed on the repository and configured to listen for PR webhook events targeting `develop`. When a PR is labelled `preview-env`:

1. GitHub delivers a `pull_request` webhook event (type: `labeled`) to Dokploy.
2. Dokploy creates a preview application for the PR, builds the image using the `qa` Dockerfile target from the PR branch, and assigns a unique subdomain under `*.openmercato.com` via Traefik.
3. The container starts with docker.sock mounted. `preview-entrypoint.sh` invokes `yarn test:integration:ephemeral:start`, which: installs deps, generates module files, builds packages twice (pre/post generate), starts an ephemeral PostgreSQL container via the mounted docker.sock, runs `yarn initialize --reinstall`, builds the Next.js app, and serves it via `yarn start`.
4. Dokploy posts the generated preview URL as a commit status or PR comment via its GitHub integration.
5. When the PR is closed, GitHub delivers a `pull_request` webhook event (type: `closed`) to Dokploy, which stops and removes the preview application automatically.

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| Separate `preview` Dockerfile stage (not reuse `dev`) | `dev` is semantically tied to local hot-reload workflows with volume mounts. `preview` is for Dokploy baked-image deployments. Keeping them separate avoids accidental breakage. |
| Ephemeral PostgreSQL via docker.sock, not a Dokploy service | Matches existing `test:integration:ephemeral:start` infrastructure; zero additional Dokploy service config. Acceptable for QA. |
| Ephemeral DB (reset on every deploy) | Consistent with `test:integration:ephemeral:start` design; seeded demo data always present, no state drift between deployments. |
| Dokploy native GitHub webhook (no external CI intermediary) | All deployment logic lives in Dokploy. No external CI configuration required. PR label events flow directly from GitHub → Dokploy webhook → build and deploy. |
| `*.openmercato.com` wildcard subdomain | Lets Dokploy assign unique per-PR URLs without manual DNS changes per PR. One-time DNS setup. |
| Secrets via Dokploy env management UI | Secrets stay in Dokploy, not in GitHub. Reduces blast radius if GitHub is compromised. |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| Single shared `qa.openmercato.com` URL | Concurrent PRs overwrite each other. Unclear which PR is deployed. |
| External CI (GitHub Actions / CircleCI) triggering Dokploy API | Introduces an unnecessary intermediary. Dokploy's native GitHub integration handles this directly. |
| Persistent PostgreSQL between deploys | State drift makes tests non-deterministic. Adds cleanup complexity. |

---

## User Stories / Use Cases

- **Reviewer** wants to open a PR preview URL directly from the GitHub PR page so that they can test the feature without cloning the branch locally.
- **Developer** wants the QA environment to deploy automatically when they add the `preview-env` label so that they don't have to trigger it manually.
- **DevOps** wants the QA environment to be torn down automatically when the PR is closed so that orphaned containers do not consume server resources.
- **Developer** wants to see the preview URL as a GitHub commit status so that they can share it with stakeholders.

---

## Architecture

```
GitHub PR (develop target) + label: preview-env
    │
    │  pull_request webhook (labeled / synchronize / closed)
    ▼
[Dokploy Server]  ← *.qa.openmercato.com  DNS A record
    │
    ├── On: labeled / synchronize (label=preview-env present)
    │       │
    │       ├── docker build --target preview . (PR branch)
    │       │       └── Packages built (yarn build:packages)
    │       │           preview-entrypoint.sh baked in
    │       │
    │       └── docker run
    │               ├── /var/run/docker.sock:/var/run/docker.sock (bind volume)
    │               ├── ENV from Dokploy UI
    │               └── preview-entrypoint.sh
    │                       └── yarn test:integration:ephemeral:start
    │                               ├── yarn install
    │                               ├── yarn build:packages
    │                               ├── yarn generate
    │                               ├── yarn build:packages (2nd pass)
    │                               ├── docker run postgres:16 (via docker.sock)
    │                               ├── yarn initialize --reinstall
    │                               ├── yarn build:app
    │                               └── yarn start  (PORT=3000)
    │
    └── On: closed
            └── Dokploy removes preview container + app automatically
```

### Domain Resolution

```
DNS: *.openmercato.com  →  A  →  {Dokploy server IP}

Dokploy Traefik generates per-PR subdomain:
  preview-{appName}-{uniqueId}.openmercato.com
      │
      └── Reverse-proxied to container port 5000
```

---

## Data Models

This spec introduces no database entities. Deployment state is managed entirely by Dokploy.

---

## API Contracts

No new application API endpoints. The integration relies entirely on Dokploy's GitHub App receiving standard GitHub webhook payloads (`pull_request` events).

---

## Configuration

### Required Dokploy Application Settings

| Setting | Value |
|---------|-------|
| Source | GitHub — `open-mercato` repo |
| Branch | PR branch (resolved dynamically by Dokploy per preview) |
| Dockerfile | `./Dockerfile` |
| Build target | `preview` |
| Exposed port | `3000` |
| Volume — Source | `/var/run/docker.sock` |
| Volume — Destination | `/var/run/docker.sock` |
| Volume — Type | Bind |
| Restart policy | `no` (ephemeral; container should not restart on exit) |
| Preview domain | `*.openmercato.com` |
| Preview trigger | PR label: `preview-env` (configure in Dokploy preview settings) |
| Max concurrent previews | `3` (configure in Dokploy to cap resource usage) |

### Required Dokploy Environment Variables (set in Dokploy UI)

| Variable | Value | Description |
|----------|-------|-------------|
| `NODE_ENV` | `production` | Runtime mode |
| Additional app-specific secrets | — | As required by `apps/mercato/.env.example` |

---

## Implementation Plan

### Phase 1: Dockerfile `qa` Stage

**Goal**: Add a `preview` build target to the Dockerfile, structurally identical to `dev` but semantically distinct for Dokploy preview deployments.

#### Steps

1. **Add `preview` stage to `Dockerfile`** — insert after the existing `dev` stage:


2. **Verify local build** — `docker build -t open-mercato:dev --target preview .` — must complete without errors.

#### File Manifest

| File | Action | Purpose |
|------|--------|---------|
| `Dockerfile` | Modify | Add `preview` stage after `dev` stage |

---

### Phase 2: Verify Ephemeral Production Startup

**Goal**: Confirm that `preview-entrypoint.sh` and `scripts/dev-ephemeral.ts --production` work correctly when the Docker image has code baked in (no volume mounts — Dokploy context).

#### Steps

1. **Review `docker/scripts/preview-entrypoint.sh`** — confirm it calls `yarhn test:integration:ephemeral:start`.

2. **Local smoke test** — run:
   ```bash
   docker build -t open-mercato:dev --target preview .
   docker run -p 3000:3000 \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -e NEXTAUTH_SECRET=test-secret \
     -e DEV_EPHEMERAL_PREFERRED_PORT=3000 \
     -e DEV_EPHEMERAL_POSTGRES_PUBLISHED_HOST=0.0.0.0 \
     -e DEV_EPHEMERAL_POSTGRES_CONNECT_HOST=host.docker.internal \
     open-mercato:qa
   ```
   Confirm the app becomes accessible at `http://localhost:3000/backend` within 10 minutes.

#### File Manifest

| File | Action | Purpose |
|------|--------|---------|
| `package.json` (root) | Verify / add | `preview:ephemeral` script must exist |

---

### Phase 4: Dokploy Application & GitHub Webhook Configuration

**Goal**: Create and configure the QA preview application in Dokploy, and connect it to the GitHub repository via Dokploy's native GitHub webhook integration. This phase is a manual ops runbook.

#### Steps

1. **Install Dokploy GitHub App** on the `open-mercato` repository:
   - In Dokploy: Settings → Git → GitHub → Install App
   - Grant access to the `open-mercato` repository
   - Dokploy registers a webhook on the repository that receives `pull_request` events

2. **Create a new Application** in Dokploy:
   - Name: `open-mercato-qa`
   - Source: GitHub → `open-mercato` repo → branch `develop` (base; overridden per PR preview)
   - Build type: `Dockerfile`
   - Dockerfile path: `./Dockerfile`
   - Build target: `preview`

3. **Enable Preview Deployments** in the application settings:
   - Navigate to application → Preview Deployments
   - Enable: On
   - Base branch: `develop`
   - Label filter: `preview-env` (Dokploy will only create previews for PRs with this label)
   - Preview domain: `*.preview.openmercato.com`
   - Max concurrent previews: `3`

   > Dokploy will generate unique subdomains following the pattern `preview-{appName}-{uniqueId}.openmercato.com` via Traefik. HTTPS is handled automatically using the wildcard Let's Encrypt certificate configured in Traefik (requires DNS-01 challenge for the wildcard).

4. **Add volume mount** (docker.sock):
   - Application → Advanced → Volumes
   - Source: `/var/run/docker.sock`
   - Destination: `/var/run/docker.sock`
   - Type: `Bind`

5. **Set environment variables** from the Configuration table above via Application → Environment.

7. **Verify webhook delivery** in GitHub → Repository Settings → Webhooks:
   - Confirm the Dokploy webhook is listed and receiving events
   - Trigger a test by opening a PR and adding the `preview-env` label
   - Check Dokploy's deployment logs to confirm the build was triggered

#### File Manifest

No files created. This phase is entirely Dokploy and DNS configuration.

---

### Phase 5: End-to-End Validation

**Goal**: Confirm the full pipeline works from PR label to accessible preview URL.

#### Steps

1. Open a test PR targeting `develop`.
2. Add label `preview-env`.
3. Confirm Dokploy receives the webhook and starts building the `preview` image (visible in Dokploy → Deployments).
4. Wait ~10 minutes for the full startup sequence (install → generate → build → DB init → Next.js start).
5. Confirm the generated preview URL (e.g. `https://preview-mercato-abc123.openmercato.com/backend`) is accessible and the backend login page loads.
6. Push a new commit to the PR branch. Confirm Dokploy rebuilds and redeploys the preview automatically.
7. Remove label `preview-env`. Confirm Dokploy stops and removes the preview application.
8. Close the PR (with label present on a second test PR). Confirm Dokploy removes the preview on PR close.

---

## Risks & Impact Review

### Data Integrity Failures

The QA environment is fully ephemeral and self-contained. No production data is involved. Risk is isolated to the QA environment itself.

### Cascading Failures & Side Effects

- **docker.sock mount failure**: If the host Docker daemon is unavailable, `yarn test:integration:ephemeral:start` fails at the PostgreSQL startup step. The container exits with a non-zero code. Dokploy marks the deployment as failed. No data loss. Re-trigger by pushing a new commit or re-applying the label.


### Migration & Deployment Risks

- Adding the `preview` stage to the Dockerfile is purely additive. Existing `builder`, `dev`, and `runner` stages are unchanged.
- No database migrations in Open Mercato's production database are involved.

