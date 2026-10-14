# file_num_generation — Knowledge Transfer

This package reflects the code as of 2026-09-22 (`main` at `0091a95`). Where the repo's own docs differ, follow this package.

## What it is

The **File Number Generator** (shown in the frontend as "Family Number Generator"). It is a small Express service that issues IP family file numbers of the form `<Y1><Y2><NNN>`, for example `PA001`:

- **Y1** is a letter that identifies the matter type and sub-org, for example `P` for a US patent or `U` for a UK patent;
- **Y2** is a letter from `A` to `Z`;
- **NNN** is a sequence from `001` to `999`.

Each series (matter type × sub-org) has its own counter in Postgres. Users can:
- generate one number (with a reason) or a bulk range;
- **save** numbers (mark them used);
- **delete** numbers they do not want. Deleting the highest number in a series moves the counter back, so that number is issued again;
- see their own list ("My List");
- export every number to `.xlsx` (the Admin page).

## Who uses it

Staff of **one designated tenant**:
- The frontend shows the module when the tenant name matches `REACT_APP_NUMBER_GENERATOR_TENANT`, and sends that tenant's dashboard straight to `/file-number-generator`.
- At login the gateway applies an IP allow-list to the tenant named in `NUMBER_GENERATOR_TENANT`.

The service keeps a single numbering space per deployment: one database (`DB_NAME`), with no tenant column.

Callers: `trexo-frontend`, through `trexo-apigateway` at `/tenantAPI/file-number-generator/*` (`FILE_NUMBER_GENERATOR_API_URL`).

## Stack

- Node 24 (`node:24-alpine`) and **Express 5**.
- Sequelize 6 on PostgreSQL (`pg`), with `sequelize-cli` migrations and one seeder.
- Joi 18 request validation and `@hapi/boom` error payloads.
- ExcelJS for the export.
- Optional AWS Secrets Manager for DB credentials, with a TTL cache and rotation through a `beforeConnect` hook.
- Winston: console plus a daily file in development, and a CloudWatch transport elsewhere.
- Jest 30 with Supertest.

There are no queues, workers or scheduled jobs.

## Run locally

1. `npm ci`.
2. Postgres running locally, with an empty database.
3. Environment, in `.env` / `.env.development` (loaded by `loadEnv.js`): `DB_NAME`, `DB_HOST`, `DB_USERNAME`, `DB_PASSWORD`, optionally `DB_DIALECT`, `PORT`, `LOG_LEVEL`. Leave `USE_SECRET_MANAGER` unset. The full list is in [deployment-and-environments.md](deployment-and-environments.md).
4. Create the schema, then seed the counters with `npx sequelize-cli db:seed:all`. Single-number generation needs the counter rows to exist. For a new database, follow the steps in [database.md](database.md#setting-up-a-new-database).
5. `npm run dev` (nodemon, `NODE_ENV=development`). The port defaults to **8015**; the image sets its own `PORT`.
6. Every request except `GET /health` needs an `X-User` header (a JSON object with `email`) and `X-Tenant-Name`, as the gateway sends them.

Tests: `npm run test:coverage`. It runs serially against local Postgres, and the suites drop and recreate tables in `DB_NAME`, so point it at a local test database only. Use `test:coverage` or `npx jest --runInBand` rather than `npm test`. Details are in `file_num_generation: .claude/docs/TESTS.md`.

## Existing documentation

| Doc | Where | Use it for |
|---|---|---|
| Architecture | `file_num_generation: .claude/docs/ARCHITECTURE.md` | Modules, data movement, boundaries |
| Flow | `file_num_generation: .claude/docs/FLOW.md` | The header and validation path through the middleware |
| Decisions | `file_num_generation: .claude/docs/DECISIONS.md` | Why validation, error shapes and the doc trail are built as they are |
| Constraints | `file_num_generation: .claude/docs/CONSTRAINTS.md` | Rules for changing the repo (ask-first areas, test safety) |
| Test checklist | `file_num_generation: .claude/docs/TESTS.md` | Commands, expected results, manual checks |
| README: Database Credentials, Request Validation | `file_num_generation: README.md` | Credential modes; the per-endpoint validation rules |

For the endpoint list and the logging set-up, use [api-and-integrations.md](api-and-integrations.md) and [architecture.md](architecture.md).

## This package

| File | For |
|---|---|
| [product-and-features.md](product-and-features.md) | Numbering rules, series, workflows, business rules, per-feature flows |
| [architecture.md](architecture.md) | Diagrams, the transaction layout, start-up, logging |
| [api-and-integrations.md](api-and-integrations.md) | Endpoint inventory, response shapes, gateway and frontend contract |
| [database.md](database.md) | Tables, indexes, counters, migrations |
| [auth-security.md](auth-security.md) | How identity and access work |
| [deployment-and-environments.md](deployment-and-environments.md) | Image, environments, config names |
| [operations-and-troubleshooting.md](operations-and-troubleshooting.md) | Runbooks: counters, start-up, logs |
| [handover.md](handover.md) | Recent work, dependencies, ownership |

## Suggested reading order

1. This file, then [product-and-features.md](product-and-features.md): the numbering rules are the whole service.
2. `.claude/docs/CONSTRAINTS.md`, `ARCHITECTURE.md`, `FLOW.md`, `DECISIONS.md`.
3. [architecture.md](architecture.md) → [database.md](database.md) → [api-and-integrations.md](api-and-integrations.md).
4. [operations-and-troubleshooting.md](operations-and-troubleshooting.md) and [handover.md](handover.md).
