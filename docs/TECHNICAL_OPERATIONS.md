# JobConnect Technical Operations Manual

**Setup, deployment, branching, testing, and maintenance procedures for JobConnect**

**Version:** v1.0  
**Date:** March 2026  
**Author:** Lonnex Njenga
**Classification:** Internal — Project Documentation

## Technical operations manual

This document is the runbook for setting up, running, deploying, and maintaining the JobConnect platform. It covers local development setup, environment configuration, the branching workflow, testing procedures, and deployment steps.

## 1. Prerequisites

| Tool                  | Version     | Install |
|-----------------------|-------------|---------|
| Node.js               | 20 LTS      | `nvm install 20` |
| pnpm                  | 9+          | `npm install -g pnpm` |
| Git                   | 2.40+       | OS package manager |
| PostgreSQL (local dev)| 16+         | `docker pull postgres:16` |
| Docker                | Latest      | docker.com/get-docker |

## 2. Repository structure

The project is a pnpm monorepo with the following structure:

| Path                     | Description |
|--------------------------|-------------|
| `apps/web/`              | Next.js 14 frontend application |
| `apps/api/`              | Express.js backend API |
| `packages/types/`        | Shared TypeScript type definitions and Zod schemas |
| `packages/config/`       | Shared ESLint, Prettier, and TypeScript configs |
| `.github/workflows/`     | GitHub Actions CI pipeline definitions |
| `docs/`                  | All project documentation (the 8 documents) |
| `.env.example`           | Template of required environment variables (committed, no secrets) |
| `BRANCH.md`              | Replaced on each feature branch to describe that branch's changes |
| `pnpm-workspace.yaml`    | Workspace configuration |

## 3. Local development setup

### 3.1 First-time setup
1. Clone the repository: `git clone https://github.com/[org]/JobConnect && cd JobConnect`
2. Install dependencies: `pnpm install`
3. Copy env files: `cp .env.example apps/api/.env && cp .env.example apps/web/.env.local`
4. Fill in all values in both `.env` files (see section 4 for variable reference)
5. Start PostgreSQL: `docker-compose up -d postgres`
6. Run migrations: `cd apps/api && pnpm prisma migrate dev`
7. Seed the database: `pnpm prisma db seed`
8. Start all services: `pnpm dev` (runs web on :3000, api on :4000)

### 3.2 Useful commands

| Command                        | What it does |
|--------------------------------|--------------|
| `pnpm dev`                     | Start all apps in dev mode with hot reload (turborepo) |
| `pnpm build`                   | Build all apps for production |
| `pnpm test`                    | Run all tests across all apps |
| `pnpm lint`                    | Run ESLint across all apps |
| `pnpm typecheck`               | Run `tsc --noEmit` across all apps |
| `pnpm prisma migrate dev`      | Apply pending migrations in development (from `apps/api/`) |
| `pnpm prisma migrate deploy`   | Apply migrations in production |
| `pnpm prisma db seed`          | Run the seed script (`apps/api/prisma/seed.ts`) |
| `pnpm prisma studio`           | Open Prisma Studio GUI for the local database |
| `pnpm prisma generate`         | Regenerate Prisma client after schema changes |

## 4. Environment variables

| Variable                | App  | Description | Example value |
|-------------------------|------|-------------|---------------|
| `DATABASE_URL`          | api  | PostgreSQL connection string (Neon or local) | `postgresql://user:pass@host/karibu?sslmode=require` |
| `ACCESS_TOKEN_SECRET`   | api  | HS256 signing secret for access tokens (≥32 random bytes) | generate with: `openssl rand -hex 32` |
| `REFRESH_TOKEN_SECRET`  | api  | HS256 signing secret for refresh tokens (≥32 random bytes) | generate with: `openssl rand -hex 32` |
| `CLOUDINARY_CLOUD_NAME` | api  | Cloudinary cloud name from account settings | `karibu-prod` |
| `CLOUDINARY_API_KEY`    | api  | Cloudinary API key | from Cloudinary dashboard |
| `CLOUDINARY_API_SECRET` | api  | Cloudinary API secret | from Cloudinary dashboard |
| `SMTP_HOST`             | api  | Gmail SMTP host | `smtp.gmail.com` |
| `SMTP_PORT`             | api  | Gmail SMTP port | `587` |
| `SMTP_USER`             | api  | Gmail address for sending | `karibu@gmail.com` |
| `SMTP_PASS`             | api  | Gmail App Password (not account password) | from Google Account Security |
| `CORS_ORIGIN`           | api  | Allowed frontend origin for CORS | `https://JobConnect` |
| `NODE_ENV`              | api  | Runtime environment | `production \| development \| test` |
| `PORT`                  | api  | Port for Express server | `4000` |
| `NEXT_PUBLIC_API_URL`   | web  | Backend API base URL used by the frontend | `https://api.JobConnect/api/v1` |

## 5. Branching workflow

### 5.1 Branch naming convention
- Feature branches: `feat/[feature-name]` (e.g. `feat/auth-register-login`)
- Bug fixes: `fix/[issue-description]` (e.g. `fix/ats-score-null-crash`)
- Infrastructure: `chore/[task]` (e.g. `chore/add-rate-limiting`)
- Hotfixes: `hotfix/[description]` (e.g. `hotfix/refresh-token-race-condition`)

### 5.2 Branch lifecycle
1. Create from main: `git checkout main && git pull && git checkout -b feat/my-feature`
2. Write the `BRANCH.md` first — document what you are building before writing code
3. Develop the feature with small, focused commits
4. Keep branch up to date: `git fetch origin && git rebase origin/main`
5. Run full checks locally: `pnpm lint && pnpm typecheck && pnpm test`
6. Open a PR to main — PR description should mirror the `BRANCH.md`
7. Wait for GitHub Actions to pass (lint + typecheck + test)
8. Self-review the PR diff — check for console.logs, hard-coded secrets, missing migrations
9. Merge to main (squash merge) and delete the feature branch

### 5.3 BRANCH.md template
Every feature branch must include a `BRANCH.md` at the repository root with the following sections:
- Feature name and branch name
- What this branch adds / changes (2–5 sentences)
- Database changes: list of tables created, altered, or dropped; migration file names
- New API endpoints: method, path, auth requirement, brief description
- Modified API endpoints: what changed and why
- Environment variables required (new ones added by this branch)
- How to test manually: step-by-step instructions a reviewer can follow
- Known limitations / deferred work

## 6. Testing strategy

| Layer            | Tool                  | Scope | Coverage target |
|------------------|-----------------------|-------|-----------------|
| Unit tests       | Jest                  | Pure functions: ATS scoring, match score, token helpers | All utility functions |
| Integration tests| Jest + Supertest      | All API routes — request/response, DB side effects | ≥ 80% of route handlers |
| E2E tests        | Playwright            | Critical user paths: register, build CV, book session, apply to job | 4 core flows |
| Type checks      | `tsc --noEmit`        | Entire codebase | 0 type errors |
| Lint             | ESLint + Prettier     | Entire codebase | 0 lint errors on merge to main |

### 6.1 Running tests
- All tests: `pnpm test` (from repo root)
- API tests only: `cd apps/api && pnpm test`
- E2E tests: `cd apps/web && pnpm test:e2e` (requires running dev server)
- Coverage report: `cd apps/api && pnpm test:coverage`

### 6.2 Test database
- Use a separate `DATABASE_TEST_URL` in `.env.test` pointing to a local test database
- Run `prisma migrate deploy` (not dev) in the test environment before test runs
- Each integration test suite uses `beforeEach`/`afterEach` to seed and tear down test data
- Never run integration tests against the production database

## 7. Deployment

### 7.1 Frontend — Vercel
1. Push to main triggers automatic deployment on Vercel
2. Set all `NEXT_PUBLIC_*` environment variables in the Vercel dashboard
3. Preview deployments are created automatically for every PR
4. Custom domain: configure `JobConnect` in Vercel domains settings

### 7.2 Backend — Railway / Render
1. Connect the GitHub repository in the Railway or Render dashboard
2. Set the start command: `node apps/api/dist/index.js`
3. Set the build command: `pnpm build` (runs tsc in `apps/api`)
4. Add all API environment variables in the Railway/Render dashboard
5. Enable auto-deploy on main branch push
6. Set the health check path to `GET /api/v1/health`

### 7.3 Database — Neon
1. Create a new project on neon.tech (free tier)
2. Copy the connection string to `DATABASE_URL` in all environments
3. Run `pnpm prisma migrate deploy` from the CI pipeline on every main merge
4. Enable automatic backups in Neon dashboard

## 8. Monitoring and maintenance

| Activity | Frequency | Tool / method |
|----------|-----------|---------------|
| Check Neon database metrics (storage, connections) | Weekly | Neon dashboard |
| Review Cloudinary usage (storage, transformations) | Weekly | Cloudinary dashboard |
| Run `npm audit` for vulnerability report | Every PR + weekly | `npm audit` / Dependabot |
| Check Railway/Render logs for errors | Daily at launch, weekly after stable | Platform dashboard |
| Test token refresh flow end-to-end | After any auth change | Manual + Playwright |
| Verify email notifications are sending | Weekly at launch | Send a test session request |
| Clear expired refresh tokens from DB | Weekly (cron or manual) | `DELETE FROM refresh_tokens WHERE expires_at < NOW()` |
| Review and respond to job listings that have expired | Weekly | Admin dashboard or direct DB query |

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| 401 on all requests after deploy | `ACCESS_TOKEN_SECRET` changed in prod env | Restore old secret or force all users to log in again |
| PDF export times out | Puppeteer needs more memory on free tier | Increase Railway memory limit, or switch to react-pdf |
| tsvector search returns no results | Search index not rebuilt after new jobs | Run: `UPDATE jobs SET search_vector = to_tsvector(...)` |
| Emails not sending | Gmail App Password rotated or SMTP blocked | Re-generate Gmail App Password, check `SMTP_PASS` env var |
| Neon DB connection pool exhausted | Too many open Prisma connections | Add `connection_limit=5` to `DATABASE_URL` query string |
| Cloudinary upload fails | API key/secret mismatch | Verify `CLOUDINARY_API_KEY` and `CLOUDINARY_API_SECRET` in prod |
| Match score always 0 | Job has no `job_skills` rows | Re-run seed or check employer job posting flow |
| CI fails on type errors | Shared types package out of sync | `cd packages/types && pnpm build` then re-push |