# reference_downloader_apis — Knowledge Transfer

Describes the code as of 2026-09-18 (`origin/main` at `b1b4507`, `origin/release` at `8dbc601`).

## What it is

The **Reference Downloader** assembles "reference packages" for patent work: for a list of patent references, one ZIP holding a combined PDF per reference (abstract/translation plus the original document) and a `Metadata.xlsx`. It:
1. fetches metadata and documents automatically via internal patent-data services;
2. flattens and normalises the PDFs through the Flatten-PDF Lambda;
3. caches every reference across all tenants in a shared `Patents` table, with the PDFs in Wasabi;
4. hands whatever automation cannot fetch to the internal **Collaboration Space** team, who upload the documents under an SLA.

A second product line in the same service is **PCT / 371 filing packages**: collecting WIPO documents (RO101, ISA237, ISA210, IASR, …) per PCT number for US national-phase or bypass-continuation filings (`controllers/pctRequestController.js`).

Users see it as "IDS Reference Downloader" (email subject); the frontend menu calls it "References Retrieval Queue".

## Who uses it

| Caller | How |
|---|---|
| **JUNE / IP Assistant chatbot** (end users) | The Lex chatbot Lambda POSTs `/requests` `{requestId, patentNumbers}` with `X-User` / `X-Tenant-Name` (`pa_lambda_functions: LexChatbot/lambda_function/reference_handler.py`) |
| **IP Assistant backend** | POSTs `/pctrequests` for AppGen / US filing (`patent-asistant-apis: controllers/usFiling.controller.js`, `COLLAB_API_URL`) |
| **Collaboration Space team** (internal experts) | Frontend `pages/CollaborationSpace/*` → gateway `/tenantAPI/reference-downloader/*` (requires the `COLLABORATE` app) |
| **Scheduler** | `GET /sendReminderEmails` |

Outbound, it calls back to the IP Assistant (`patent-asistant-apis`): `/postEmail` on request completion and `/usFilingPostCollab` on PCT submit.

## Stack

Node (Docker `node:24-alpine`, `.nvmrc` 22, `engines >=22`) and Express 5; Sequelize 6 on **one** PostgreSQL database (tenant is a column) with `sequelize-cli` migrations. Also:
- **AWS SQS** (three queues, polled in-process);
- **Wasabi** (S3-compatible) for reference PDFs, plus **AWS S3** for request ZIPs (presigned URLs);
- `pdf-lib`, `pdf-parse`, `pdf-to-png-converter` and `tesseract.js` (OCR language check);
- `archiver` and ExcelJS;
- Nodemailer (SMTP);
- Winston with a CloudWatch transport;
- optional Secrets Manager;
- Joi request validators;
- Swagger UI (development and qa);
- Jest 30.

## How it fits

See [../system-map.md](../system-map.md).

| Repo / system | Relationship |
|---|---|
| `pa_lambda_functions` (LexChatbot) | Creates reference requests |
| `pa_lambda_functions` (FlattenPDF, `FLATTEN_PDF_URL`) | Rasterises and normalises PDFs from the Wasabi temp bucket into the references bucket |
| `patent-asistant-apis` | Creates PCT requests; receives the completion callbacks; also connects to this service's database for its `AppGenInput` model (the schema is migrated here) |
| `trexo-frontend` / `trexo-apigateway` | Collaboration Space UI → `/tenantAPI/reference-downloader/*` |
| `epo-bdss-apis` (`EPO_BDSS_URL`) | Bulk bibliographic metadata via `POST /api/publication/bulk` (contract in [../epo-bdss-apis/api-and-integrations.md](../epo-bdss-apis/api-and-integrations.md)) |
| "patent-cli" service (`PATENT_CLI_URL`) | Original-document URLs (Google Patents, EPO OCR) |

## Run locally

1. `npm ci`. Configuration comes from dotenv files loaded by `loadEnv.js` (the same chain as the other Trexo services). Start from `.env.sample` and add the names in [deployment-and-environments.md](deployment-and-environments.md).
2. You need Postgres, the SQS queue URLs (a queue whose URL is unset is skipped), Wasabi and S3 credentials, and the internal service URLs.
3. `npm run start:dev` (port `PORT` or 8005). Swagger UI is at `/api-docs` in development and qa; get a token from `POST /api/v1/auth/token`.
4. Tests: `npm test`.

- Routes are mounted at `/` (no `/api` prefix).
- Run the app with Node directly rather than through the compose file.
- **Use `npm run setup` only against a new, empty local database.** It recreates the tables from the models (dropping any that exist) and sets `NODE_ENV=production`. Use migrations for every shared environment.
- Test with public patent references only.

## Existing repo documentation

This package reflects the code as of the date above; where the repo docs differ, follow this package.

| Repo doc | Use it for |
|---|---|
| `reference_downloader_apis: README.md` | The Request Validation section (validators and how to test them) |
| `reference_downloader_apis: PRODUCT_FLOW_DOCUMENTATION.md` | Product narrative |
| `reference_downloader_apis: REQUEST_FLOW_DIAGRAM.md` | Per-endpoint flowcharts (the queue layer is now AWS SQS) |
| `reference_downloader_apis: docs/API_REFERENCE.md` | The validation table |
| `reference_downloader_apis: docs/ENDPOINT_MAP.md` | Endpoint map |

## This package

| File | Read it for |
|---|---|
| [product-and-features.md](product-and-features.md) | Request lifecycle, completion rules, SLA, language and document rules, PCT packages |
| [architecture.md](architecture.md) | System context, the async pipeline, completion coordination, ZIP generation |
| [api-and-integrations.md](api-and-integrations.md) | Endpoint inventory and external systems |
| [database.md](database.md) | Entities, the shared cache, schema management |
| [auth-security.md](auth-security.md) | Trust model, validation, data handling |
| [jobs-and-queues.md](jobs-and-queues.md) | The three SQS queues and scheduled work |
| [deployment-and-environments.md](deployment-and-environments.md) | Branches, build, deploy steps, configuration |
| [testing.md](testing.md) | Test layout and commands |
| [operations-and-troubleshooting.md](operations-and-troubleshooting.md) | Logs and runbooks |
| [handover.md](handover.md) | Recent delivered work, dependencies, ownership |

## Suggested reading order

1. This file.
2. `PRODUCT_FLOW_DOCUMENTATION.md` (repo), then [product-and-features.md](product-and-features.md) for the rules as implemented.
3. [architecture.md](architecture.md) and [jobs-and-queues.md](jobs-and-queues.md).
4. [database.md](database.md).
5. [operations-and-troubleshooting.md](operations-and-troubleshooting.md) and [handover.md](handover.md).
