# drm-apis — Deployment and environments

`drm-apis: docs/deployment.md` describes the Dockerfile.

## Environments and branches

| Branch | Environment |
|---|---|
| `main` | Integration; the PR target |
| `dev` | development |
| `qa` | qa |
| `uat` | uat |
| `release` | production |

The mapping follows the branch names, the `package.json` start scripts (`start:qa`, `start:uat`, `start`) and the `.env.<env>` files. Changes flow `main` → `uat` → `release`.

Release tags run from `v1.3.0` (2025-12-17) through `v1.5.1` (2026-07-16) to `v2.0.0` (2026-09-08, the uat → release merge `f422ef0`). `pre-merge-release-backup` (`ddf2629`) is the rollback target named in `POST_MERGE_STEPS.md` §10.

## Build

- `Dockerfile`: `node:24-alpine`, `apk add file` (for MIME detection), `npm ci --omit=dev --ignore-scripts`, non-root user, `PORT=8004`, `npm start` (`NODE_ENV=production`).
- `.dockerignore` excludes `.env*`, so each environment supplies its configuration through the runtime environment.

## CI

`.github/workflows/pr-checks.yml`, triggered on PRs and on pushes to `main`, `dev`, `qa` and `uat`:
- Node 22;
- `npm ci`, then `npm run lint`, then `npm run test:coverage`. The thresholds in `jest.config.js` are 95% lines, functions and statements, and 90% branches.

A Husky `pre-push` hook runs `npm run lint && npm run test:coverage` locally.

## Runtime infrastructure dependencies

- PostgreSQL: the super-admin DB plus one DB per tenant, with the DRM procedures applied (see [database.md](database.md)).
- Redis (Bull).
- Network access to each tenant's SFTP server.
- Outlook SMTP.
- CloudWatch Logs and Metrics; Secrets Manager if enabled.
- The scheduler that calls `/cron/*`, and the `scripts` DRM job.
- The gateway's `DRM_API_URL` pointing at this service.

## Configuration variable names

The full list is in `drm-apis: docs/configuration.md`. Also used:

| Name | Used by | Effect |
|---|---|---|
| `APP_TARGET` | `utils/logger/index.js` | Adds a console transport when `local` and `NODE_ENV=development` |
| `DB_SECRET_NAME`, `USE_SECRET_MANAGER` | `config/dbCredentials/*` | Secrets Manager credential mode |

## Release procedure

`POST_MERGE_STEPS.md` is the runbook for the v2.0.0 production release, and a template for later ones. It covers:
- verification: lint and the test suite;
- UAT upload checks for each file type;
- rollback artefacts: `pg_dump` / `\sf` of the procedures;
- deploying with Secrets Manager off, then enabling it;
- post-deploy checks;
- rollback to `ddf2629`.

## Rollback

Code rollback is a redeploy of the previous image or tag; releases that change no DB objects need nothing more. Queue names are stable, so Redis jobs survive a rollback. Procedure changes (in `DB-objects` or via `setupDRM`) are rolled back separately, from the `\sf` copies taken before the release.
