# ids-api — Deployment and environments

The SB-08/892 rollout procedure (6 systems, DB rows, Lex build, Redis flush, verification, rollback) is in `ids-api: docs/TAI-1181-deployment.md`.

## Branches

`main` is the integration branch; the environment branches are `dev`, `qa`, `uat`, `staging` and `release`. Fixes are sometimes applied to each environment branch as separate commits, so compare trees before merging between branches. Tags `v1.0.0`–`v1.0.3` exist.

## Build and CI

- `Dockerfile`: `node:24-alpine`, `npm ci --omit=dev --ignore-scripts`, non-root, `NODE_ENV=production`, port 8007.
- `Jenkinsfile`: Node tool `node-24` → `npm ci` → unit → integration → API → e2e → coverage (HTML plus JUnit).

## Configuration variable names

| Group | Names |
|---|---|
| Runtime | `NODE_ENV`, `PORT`, `APP_TARGET`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` |
| ODP | `API_URL`, `API_TOKEN`, `API_RETRIES` (set it; the download and extraction retries use it) |
| Extraction | `PATENT_ASSISTANT_URL`, `EXTRACT_REFERENCES_TIMEOUT_MS`, `FAST_API_URL`, `FETCH_PDF_DATA_MODEL`, `FETCH_PDF_DATA_OCR`, `FETCH_PDF_DATA_USER_MESSAGE`, `FETCH_PDF_DATA_SYSTEM_PROMPT`, `FETCH_PDF_DATA_MAX_TOKEN`, `TENANTS_EXCEPTION` |
| S3 | `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `JUNE_BUCKET_NAME`, `AWS_S3_URL_EXPIRATION` |
| Logging | `AWS_ACCESS_KEY_SECRET` (CloudWatch transport) |
| Citation Matrix and scripts | `CITATION_MATRIX_MAX_APPLICATIONS`, `BATCH_SIZE`, `BATCH_PAUSE_MS`, `MAX_RETRIES`, `REQUEST_DELAY_MS`, `RETRY_BASE_MS`, `FAMILY_REFERENCES_*` |

## Deploy order for SB-08/892

`patent-asistant-apis` (with `/api/extract-references`) → `ids-api` → superadmin and tenant DB rows → Lambda → Lex build → Redis flush. Details are in the TAI-1181 runbook.

## Root-level scripts

`script.js`, `ids_references_script.js`, `family_references_script.js` and `scripts/family_references_script.js` are batch reference-extraction utilities, separate from the service.
