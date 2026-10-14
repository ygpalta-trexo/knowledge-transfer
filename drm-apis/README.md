# drm-apis — Knowledge Transfer

Describes the code as of 2026-09-18 (`origin/main` at `6b7254c`).

## What it is

`drm-apis` is the backend for **DRM (Data Risk Manager)**, a docket-audit product for IP law-firm tenants. Each tenant supplies spreadsheet or XML exports from its docketing system: patent and trademark dockets, bibliographic data, PAIR correspondence, India matters, working statements, and CPI / Inprotech / WBD variants. The service loads them into that tenant's own Postgres database, then runs audit stored procedures that record data-quality findings ("observations") in `audit_result` / `audit_result_tm`. Users review those in the frontend and mark them as managed, corrected or ignored.

"DRM" means Data Risk Manager, not Digital Rights Management.

## Who uses it

- **Tenant users**, through `trexo-frontend` → `trexo-apigateway` (`/tenantAPI/drm/*`). The gateway authenticates the JWT, checks that the tenant has the `DRM` app, and forwards `X-Tenant-Name` and `X-User`.
- **Scheduled operations**: SFTP ingestion, the daily audit run, the weekly PAIR rule switch and archive cleanup. These arrive as HTTP calls to the `/cron/*` routes from an external scheduler and from the `scripts` DRM job (see [../scripts/README.md](../scripts/README.md)).
- **Tech team**: receives notification emails and monitors queues in Bull Board at `/admin/queues`.

## Stack

Node 24 (Docker) and Express 5; Sequelize 6 on PostgreSQL (one super-admin DB plus one DB per tenant); Bull 4 on Redis; ExcelJS and fast-xml-parser for file parsing; `ssh2-sftp-client` for SFTP; Nodemailer (Outlook SMTP); Winston with a custom CloudWatch Logs transport; optional AWS Secrets Manager for DB credentials; Jest 30 with Supertest; ESLint with SonarJS; Husky pre-push.

## How it fits with other repos

| Repo | Relationship |
|---|---|
| `trexo-frontend` | DRM pages (`src/app/pages/DRM/*`) call `/tenantAPI/drm/<route>` |
| `trexo-apigateway` | Authenticates, then proxies `/tenantAPI/drm/*` to `DRM_API_URL` |
| `scripts` (`drm/`) | Loads tenant SFTP files and calls `POST /cron/run_audit` |
| `DB-objects` | Holds the `DRM/` SQL: production and per-tenant variants of `run_audit*`, including `run_audit_tm` and `run_audit_tm_cpi` |
| Super-admin DB (shared) | `Tenant` and `DrmMappings` tables; the tenant registry is shared with other services |

See [../system-map.md](../system-map.md).

## Run locally

The full steps are in the repo `README.md`. You need Node 24, a reachable Postgres containing the super-admin DB and at least one tenant DB with the DRM schema and procedures, and Redis. Then run `npm ci` and `npm run start:dev` (nodemon; `NODE_ENV=development`; port `PORT` or 8002).

- Run the app directly with Node rather than through `docker-compose.local.yml`.
- Set `NODE_ENV=development` **and** `APP_TARGET=local` to get console logs. Otherwise logs go to CloudWatch, which needs AWS credentials.
- Every request other than `/health`, `/cron/*` and `/admin/queues` needs the `X-Tenant-Name` and `X-User` (JSON) headers.
- Use synthetic workbooks for local testing, never client exports.

## Existing repo documentation

This package adds to the repo's own docs rather than repeating them. It reflects the code as of the date above; where the two differ, follow this package.

| Repo doc | Covers |
|---|---|
| `drm-apis: README.md` | Setup and scripts |
| `drm-apis: docs/architecture.md` | Request flow, module layout, middleware order |
| `drm-apis: docs/api.md` | Per-endpoint request and response shapes |
| `drm-apis: docs/configuration.md` | Environment variables |
| `drm-apis: docs/database.md`, `docs/data-model.md` | Topology, pools, model by model |
| `drm-apis: docs/queue.md` | Queue list and processing |
| `drm-apis: docs/storage.md`, `docs/input-output.md`, `docs/validation.md` | File handling and validation |
| `drm-apis: docs/authentication-authorization.md` | Service-level auth |
| `drm-apis: docs/logging.md`, `docs/observability.md`, `docs/error-handling.md` | Logging and errors |
| `drm-apis: docs/operations.md`, `docs/decisions.md` | Operations and design decisions |
| `drm-apis: POST_MERGE_STEPS.md` | The v2.0.0 release runbook |

## This package

| File | Read it for |
|---|---|
| [product-and-features.md](product-and-features.md) | Business flow: file families, audit lifecycle, weekly PAIR cycle, retention |
| [architecture.md](architecture.md) | Diagrams of the upload → queue → audit → review path and of startup |
| [api-and-integrations.md](api-and-integrations.md) | How the gateway exposes routes, endpoint inventory, external systems |
| [database.md](database.md) | Procedure lifecycle, where procedure source lives, provisioning |
| [auth-security.md](auth-security.md) | Trust model and security mechanisms |
| [jobs-and-queues.md](jobs-and-queues.md) | Bull processing and the scheduled routes |
| [deployment-and-environments.md](deployment-and-environments.md) | Branches and environments, build, CI, releases |
| [testing.md](testing.md) | Test layout and how to run it |
| [operations-and-troubleshooting.md](operations-and-troubleshooting.md) | Recurring jobs, logs, runbooks by symptom |
| [handover.md](handover.md) | Recent delivered work, dependencies, ownership |

## Suggested reading order

1. This file, then [product-and-features.md](product-and-features.md) → [architecture.md](architecture.md).
2. `drm-apis: docs/architecture.md`, `docs/api.md`, `docs/data-model.md`.
3. [database.md](database.md) and [jobs-and-queues.md](jobs-and-queues.md).
4. `drm-apis: POST_MERGE_STEPS.md` (the most recent production release).
5. [operations-and-troubleshooting.md](operations-and-troubleshooting.md) and [handover.md](handover.md).
