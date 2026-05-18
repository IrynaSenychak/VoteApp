# CI/CD Implementation for Vote App

## Overview

The project consists of two separate repositories:

- **Backend** — `alexolashyn/vote-app-backend` (NestJS + PostgreSQL)
- **Frontend** — `alexolashyn/vote-app-client` (React + Vite)

Each repository has its own CI/CD pipeline powered by **GitHub Actions**. The backend is deployed to **Render**, the frontend to **Vercel**.

---

## Branch Structure

Both repositories follow the same branching model:

```
master   — production-ready code, protected branch
develop  — integration branch, staging environment
```

Workflow:
1. Development happens on `develop` or separate feature branches
2. Feature branches are merged into `develop` via Pull Request
3. After testing on staging, `develop` is merged into `master` via Pull Request
4. Merging into `master` automatically triggers a production deploy and creates a new version tag

---

## Backend CI/CD

### Tools
- **CI/CD:** GitHub Actions
- **Hosting:** Render (Web Service)
- **Database:** Render PostgreSQL
- **Linter:** ESLint with `@typescript-eslint`
- **Tests:** Jest (unit), SuperTest (e2e)

### Pipeline

File: `.github/workflows/backend-ci-cd.yml`

Triggered automatically on every `push` or `pull_request` to `master` and `develop`.

```
push to develop:
lint → test → e2e → build → deploy-staging

push to master:
lint → test → e2e → build → version → deploy-production

pull request:
lint → test → e2e → build
```

### Steps

**lint**
Runs ESLint across all `.ts` files in `src/` and `test/`. If any errors are found the pipeline stops immediately and no further steps are executed.

**test**
Runs unit tests via Jest with code coverage report generation. The report is uploaded to Codecov. Tests run against an in-memory SQLite database so they do not depend on any external services.

**e2e**
Runs end-to-end tests via SuperTest. Before execution a `.env.test` file is created automatically with SQLite configuration. Tests validate real HTTP requests against the full application stack.

**build**
Compiles TypeScript to JavaScript via `nest build`. The compiled `dist/` folder is saved as a build artifact on GitHub for 7 days and can be downloaded from the Actions tab.

**version** *(master only)*
Automatically determines the version bump type based on commit messages since the last tag and creates a new git tag.

**deploy-staging** *(develop only)*
Sends a POST request to the Render Deploy Hook URL. Render rebuilds and restarts the staging service. After the deploy the pipeline polls `GET /health` every 20 seconds for up to 10 attempts. If the health check fails the job exits with an error.

**deploy-production** *(master only)*
Same as staging but for the production service with more attempts (15 × 20 seconds).

### Render Environments

| Environment | Branch | Service |
|-------------|--------|---------|
| Staging | `develop` | `vote-app-backend-staging` |
| Production | `master` | `vote-app-backend` |

Both services use the same startup configuration:
- **Build Command:** `npm install --include=dev && npm run build`
- **Start Command:** `node ./node_modules/.bin/typeorm migration:run -d dist/data-source.js && node dist/main`

Database migrations run automatically on every service start, ensuring the database schema always matches the current code.

### Health Check Endpoint

A `GET /health` endpoint was added to `AppController` and returns `{ "status": "ok" }`. It is used by the pipeline to verify the service is up after each deployment.

```typescript
@Get('health')
health(): { status: string } {
  return { status: 'ok' };
}
```

---

## Frontend CI/CD

### Tools
- **CI/CD:** GitHub Actions
- **Hosting:** Vercel
- **Linter:** ESLint with `eslint-plugin-react-hooks`
- **Tests:** Vitest + Testing Library

### Pipeline

File: `.github/workflows/frontend-ci-cd.yml`

```
push to develop:
lint → test → build → deploy-staging

push to master:
lint → test → build → version → deploy-production

pull request:
lint → test → build → deploy-preview
```

### Steps

**lint**
Runs ESLint across all `.js` and `.jsx` files. The config includes React hooks rules, vitest globals (`describe`, `it`, `expect`) for test files, and Node.js globals for setup files.

**test**
Runs tests via Vitest with code coverage (`--coverage --run`). Vitest automatically detects the CI environment and runs tests once without watch mode.

**build**
Builds the project via `vite build`. The backend URL is injected at build time via the `VITE_API_URL` environment variable. The output (`dist/`) is saved as an artifact for 7 days.

**version** *(master only)*
Identical logic to the backend — Semantic Versioning based on Conventional Commits.

**deploy-preview** *(pull requests only)*
Every Pull Request gets a unique preview URL from Vercel, allowing changes to be reviewed before merging. Uses Vercel CLI: `vercel pull → vercel build → vercel deploy --prebuilt`.

**deploy-staging** *(develop only)*
Deploys to Vercel as a preview deployment (without `--prod` flag), creating a dedicated staging URL.

**deploy-production** *(master only)*
Deploys to Vercel as a production deployment (with `--prod` flag). After the deploy the pipeline checks that the site responds with HTTP 200 (5 attempts × 15 seconds).

### Vercel Environments

| Environment | Branch | Deploy type |
|-------------|--------|-------------|
| Preview | pull request | preview (unique URL per PR) |
| Staging | `develop` | preview |
| Production | `master` | production |

---

## Semantic Versioning

Both repositories use the same versioning scheme based on [Conventional Commits](https://www.conventionalcommits.org).

On every merge to `master` the pipeline:
1. Finds the latest git tag (or uses `v0.0.0` if no tags exist yet)
2. Reads all commits since that tag
3. Determines the bump type:

| Commit message | Bump type | Example |
|----------------|-----------|---------|
| contains `BREAKING CHANGE` | Major | `v1.2.3 → v2.0.0` |
| starts with `feat:` | Minor | `v1.2.3 → v1.3.0` |
| `fix:`, `ci:`, `docs:`, etc. | Patch | `v1.2.3 → v1.2.4` |

4. Creates and pushes a new git tag (e.g. `v0.1.0`)

Tags are visible in the **Tags** tab of each GitHub repository.

---

## Deployment Strategies

### Backend — Blue-Green

Two live environments are maintained simultaneously: staging and production. New code is first deployed to **staging** (`develop`), verified, and only then promoted to **production** (`master`). Render performs zero-downtime deploys — the new instance starts alongside the old one and traffic switches only after a successful health check.

If the health check fails after a deploy, the previous version remains active. Rollback is performed manually via the Render Dashboard (Events → Rollback).

### Frontend — Canary

Vercel atomically replaces the production deployment with each new build. If the post-deploy health check fails the pipeline exits with an error and the previous Vercel deployment stays active. Rollback is performed via `vercel rollback` or through the Vercel Dashboard (Deployments → Promote to Production).

---

## Rollback Procedure

### Backend
1. Open [Render Dashboard](https://dashboard.render.com)
2. Select the service → **Events**
3. Find the last successful deploy → **Rollback**

Or via git:
```bash
git revert HEAD
git push origin master
```

### Frontend
Via Vercel Dashboard:
1. Open the project → **Deployments**
2. Find the last successful deployment → **...** → **Promote to Production**

Or via CLI:
```bash
vercel rollback --token <VERCEL_TOKEN>
```

---

## Post-Deploy Monitoring

| | Backend | Frontend |
|--|---------|----------|
| **Method** | `GET /health` → `{ status: "ok" }` | HTTP status of the main page |
| **Staging** | 10 attempts × 20 s | — |
| **Production** | 15 attempts × 20 s | 5 attempts × 15 s |
| **On failure** | job exits with error | job exits with error |

---

## GitHub Secrets and Variables

### Backend Repository

| Type | Name | Description |
|------|------|-------------|
| Secret | `RENDER_DEPLOY_HOOK_URL` | Deploy Hook for the production Render service |
| Secret | `RENDER_STAGING_DEPLOY_HOOK_URL` | Deploy Hook for the staging Render service |
| Secret | `CODECOV_TOKEN` | Token for uploading coverage to Codecov |
| Variable | `PRODUCTION_URL` | Production service URL |
| Variable | `STAGING_URL` | Staging service URL |

### Frontend Repository

| Type | Name | Description |
|------|------|-------------|
| Secret | `VERCEL_TOKEN` | Vercel API token |
| Secret | `VERCEL_ORG_ID` | Vercel account ID |
| Secret | `VERCEL_PROJECT_ID` | Vercel project ID |
| Secret | `CODECOV_TOKEN` | Token for uploading coverage to Codecov |
| Variable | `VITE_API_URL` | Backend API URL injected at build time |
| Variable | `VERCEL_PRODUCTION_URL` | Production deployment URL |
| Variable | `VERCEL_STAGING_URL` | Staging deployment URL |
