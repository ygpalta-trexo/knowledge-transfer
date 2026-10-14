# kot-apis — Deployment and environments

`docs/Development_Setup.md` has setup notes. This file records what the repo contains for build and deployment.

## Branches and environments

| Branch | Environment |
|---|---|
| `main` | Integration |
| `qa` | qa |
| `uat` | uat |
| `release` | production |

The mapping follows the branch naming used across the services. As of 2026-09-18, `main`, `uat` and `release` point at the same commit (`7daa826`). PRs are squash-merged (`… (#33)`).

## Build and runtime

- `Dockerfile`: `node:22-alpine`, `npm ci --omit=dev --ignore-scripts`, non-root, `NODE_ENV=production`, `PORT=8011`, `npm start`.
- Set `PORT` explicitly in every environment (the code default is 3000; the image uses 8011).
- One process; no workers or queues.
- Tenant connections are built at boot, so **restart after adding a tenant**.
- `.dockerignore` excludes `.env*`, `logs` and `coverage`.

## CI

- `.github/workflows/pr-checks.yml`: pushes and PRs to `main`, `dev`, `qa`, `uat` and `release` → Node 22 → `npm ci` → `npm run lint` → `npm run test:coverage`.
- A Husky `pre-push` hook runs the same pair.

## Configuration variable names

| Group | Names |
|---|---|
| Runtime | `NODE_ENV`, `PORT`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` |
| DB | `SUPERADMIN_DB`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_DIALECT`, `USE_SECRET_MANAGER`, `DB_SECRET_NAME`, `AWS_REGION` |
| AWS (S3, CloudWatch) | `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET`, `S3_BUCKET_NAME`, `KOT_TEMPLATE_NAME` |
| Integrations | `DOCUMENT_PROCESSOR_URL` |

## Environment loading

`src/loadEnv.js` chains `.env` → `.env.local` → `.env.<NODE_ENV>` → `.env.<NODE_ENV>.local` (plain `dotenv` with override).

## Tenant onboarding

Follow `docs/KOT_Tenant_Setup.md`, then restart the service. The related scripts (`setupKotTenant.js`, `initializeKotRoles.js`, `seedForeignAssociates.js`, `addUserToKot.js`) are described in [database.md](database.md#applying-schema-changes) and [product-and-features.md](product-and-features.md#access-administration).
