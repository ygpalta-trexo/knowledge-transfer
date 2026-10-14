# epo-bdss-apis — API and integrations

Routes are mounted under `/api` (`src/index.js`).

## Route inventory

| Method | Route | Returns | Notes |
|---|---|---|---|
| GET | `/health` | 200 | Liveness |
| GET | `/` | Service name and docs link | |
| GET | `/api/publication/:identifier` | Enriched biblio object (`buildBiblioResponse`) | 404 if not found; 400 if unparseable |
| POST | `/api/publication/bulk` | `{ "<original input>": biblio \| {error:"Failed to fetch meta data"} }` | **The endpoint both callers use.** Body: `references` (or `numbers`), a comma string or an array |
| GET | `/api/application/:identifier` | The `patent_documents` row (every column) | Filters on `app_country` / `app_number` |
| POST | `/api/application/bulk` | `{count, applications:[rows]}` | Body: `references` / `numbers` |
| POST | `/api/lookup/bulk` | `{publications[], publicationCount, applications[], applicationCount}` | Body: `publicationIdentifiers` / `applicationIdentifiers` (or `*Numbers`) |
| POST | `/api/v1/auth/token` | Swagger token | development and qa |
| GET | `/api-docs` | Swagger UI | development and qa |
| * | anything else | 404 `{error:'Endpoint not found'}` | |

Every route except `/health`, `/` and the Swagger paths expects `X-User` (JSON) and `X-Tenant-Name`.

Publication lookups filter on `country`, the partition key, so each query touches one partition. Application lookups filter on `app_country`, which spans partitions; prefer publication lookups for high-volume use.

## The bulk contract

Both callers depend on `POST /api/publication/bulk`:
- **`reference_downloader_apis`** (`services/epoService.js`, `queues/fetchMetadataQueueProcessor.js`);
- **`patent-asistant-apis`**: the SB-08 Generator, AppGen, the ODP service and the bib-data export. Its SB-08 code reads `country`, `kindCode`, `applicants[0]`, `inventors[0]` (and the count, for "et al."), and `patentIssueDate || publicationDate`, and treats "every key has an error" as a failed metadata fetch (`controllers/formGeneration.controller.js`, `case "SB-08"`).

**Request:** body `{ "references": ["US10123456B2", "EP1234567A1", …] }`, headers `X-Tenant-Name`, `X-USER`, `REQUEST-ID`. Callers upper-case and validate the references first.

**Response:** an object keyed by **the exact input string** (`original: raw.trim()`). Each value is either:
- the biblio object: `patentTitle`, `abstract`, `inventors[]`, `applicants[]`, `applicationNumber`, `applicationFilingDate`, `publicationNumber`, `publicationDate`, `patentNumber`, `patentIssueDate`, `familyPublicationDate`, `familyGrantDate`, `familyPatentNumber`, `priorityNumber`, `priorityCountry`, `priorityDate`, `kindCode`, `country`;
- or `{ "error": "Failed to fetch meta data" }`.

**Rules the callers rely on:**
- **Keys equal the strings sent.** Callers read `bulkResults[number]`, so keys are never normalised.
- A per-key `error` means "not available at source".
- A batch with no matches returns per-key `error` entries, never `{}`. The reference downloader treats an empty object as an outage and retries later.
- The reference downloader uses `patentIssueDate || publicationDate`, `applicants`, `inventors`, `applicationFilingDate` and `applicationNumber`.

Treat `buildBiblioResponse` and the key-by-original behaviour as a public API (`CONSTRAINTS.md` on `main`), and check both callers before changing either.

## Identifier matching

`parseIdentifierCandidates()` (`src/db.js`) turns one input into an **ordered list** of spellings (`lookupNumbers`), verbatim first for US, queries `IN (candidates)` and picks the row with `selectPublicationRow()`. It handles:
- US pre-grant publication numbers in both 10- and 11-digit widths (TAI-543, 2026-09-11; for example `US2026256035A1`);
- US numbers checked both raw and parsed (`0651c43`);
- WO numbers before 2004;
- KR 13-digit numbers with the type prefix (`40ae354`);
- input sanitisation (TAI-543).

The regression set for these rules is in `main:.claude/docs/TESTS.md` and runs with `npm run test:identifier`.

`npm run export:application-biblio` exports bibliographic data for a file of application numbers to xlsx, in batches.

## External systems

| System | Config | Notes |
|---|---|---|
| PostgreSQL (`docdb` data) | `DB_NAME`, `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_DIALECT`, `DB_PORT`, or `DATABASE_URL` / `POSTGRES_URL`; `USE_SECRET_MANAGER`, `DB_SECRET_NAME`, `AWS_REGION` | Single database; the service only reads; default pool |
| AWS Secrets Manager | as above | If the secret cannot be read, the service uses the env credentials and logs a warning |
| CloudWatch Logs | `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET`, `AWS_REGION`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` | Groups: per tenant, `${NODE_ENV}/server`, `${NODE_ENV}/crontab`; stream `epo-bdss-apis` |
