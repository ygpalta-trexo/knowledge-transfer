# epo-bdss-apis — Knowledge Transfer

Describes the code as of 2026-09-18 (`main` at `f8e5730`, `release` at `ce2b98f`).

## What it is

A small, **read-only** HTTP service (`package.json` name `bdds-apis`) over a shared PostgreSQL store of **EPO DOCDB bibliographic data**. It turns a patent reference such as `EP1234567A1` or `US2026256035A1` into structured bibliographic data:
- title and abstract;
- applicants and inventors;
- application number and filing date;
- publication and grant dates;
- family publication and grant dates;
- priority.

"BDDS" refers to EPO's Bulk Data Distribution Service, the source of the DOCDB bulk files; the repo name spells it "bdss".

## Who uses it

Two services, both via `POST {EPO_BDSS_URL}/api/publication/bulk`:
- **`reference_downloader_apis`**: its metadata worker.
- **`patent-asistant-apis`**: the **SB-08 Generator** metadata (`controllers/formGeneration.controller.js`), plus `utils/appgenUtility.js`, `services/odpApi/patent.js` and `controllers/bibData.controller.js`.

It is an internal service; the frontend and gateway do not call it.

## Stack

- Node 22 (`node:22-alpine`), native ESM, Express 4.
- Sequelize 6 with `pg`; a single model, `PatentDocument` (table `patent_documents`).
- Winston with a CloudWatch transport (daily files in development).
- Swagger UI with a Swagger login (development and qa).
- Optional AWS Secrets Manager for DB credentials (TP-317).
- Jest 30.

## Data

`patent_documents` is **LIST-partitioned by `country`**, about **350 GB** in total (CN 149 GB / 56M rows, US 64 GB / 22M rows, JP 63 GB, DEFAULT 50 GB as of 2026-09-09). The indexes are the `(country, pub_doc_id)` PK, `(country, doc_number)` and `(app_country, app_number)`. Large JSONB columns hold texts, parties, priorities, classifications, citations and more (`src/models/PatentDocument.js`).

The data is loaded and refreshed by the separate repo `docdb_ingestion` (`~/repos/bdds/docdb_ingestion`); see [../docdb_ingestion/README.md](../docdb_ingestion/README.md). Ingestion stores void (CV/DV) records as stubs (`source_status='SKIP'`, `pub_doc_id` starting `VOID_`).

## Run locally

1. `npm install`, then `cp .env.example .env`.
2. Set `DB_NAME` plus either `DB_*` or `DATABASE_URL`, or `USE_SECRET_MANAGER=true` with `DB_SECRET_NAME`.
3. `npm run start:local` (nodemon) or `npm run start:dev`. Set `PORT` (the code default is 4000; the image uses 8020).
4. Every `/api` call needs `X-User` (JSON) and `X-Tenant-Name` headers.
5. Swagger UI is at `/api-docs` in development and qa.

Tests: `npm test`. The identifier regression script: `npm run test:identifier`.

## Existing documentation

| Doc | Where | Use it for |
|---|---|---|
| README | `epo-bdss-apis: README.md` | Setup and the Secrets Manager section. For routes, use [api-and-integrations.md](api-and-integrations.md) |
| Design notes | `.claude/docs/` on `main` (ARCHITECTURE, FLOW, DECISIONS, CONSTRAINTS, TESTS, ROLLBACK, HANDOVER); read with `git show origin/main:.claude/docs/<file>` | The identifier-candidate logic, partition rules, read-only rule and the identifier regression set |
| Swagger | `src/swagger.js` plus route JSDoc; served at `/api-docs` | Interactive route reference |

## This package

| File | For |
|---|---|
| [api-and-integrations.md](api-and-integrations.md) | Routes, the bulk contract both callers depend on, identifier matching |
| [auth-security.md](auth-security.md) | Header handling, Swagger login, credentials |
| [deployment-and-environments.md](deployment-and-environments.md) | Branches, image, configuration |
| [handover.md](handover.md) | Recent delivered work, dependencies, ownership |

Architecture, database and testing detail are in the design notes on `main`. There are no queues or jobs.

## Suggested reading order

1. This file.
2. `git show origin/main:.claude/docs/CONSTRAINTS.md`, then `ARCHITECTURE.md` and `FLOW.md`.
3. [api-and-integrations.md](api-and-integrations.md).
4. `git show origin/main:.claude/docs/TESTS.md` (the identifier regression set).
5. [handover.md](handover.md).
