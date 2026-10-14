# reference_downloader_apis — Deployment and environments

## Branches, environments and releases

| Branch | Environment |
|---|---|
| `main` | Integration |
| `dev` | development |
| `qa` | qa |
| `uat` | uat |
| `release` | production |

The mapping follows the branch names and the `start:*` / `migrate:*` scripts. Changes are promoted from `main`. Release tags run from `v1.0.3` (2025-08) to `v3.1.1` (2026-07-13); `v3.0.0` (2026-02-11) is the first release on SQS. The last production deployment was PA-68 (`8dbc601`, 2026-07-31).

## Build and run

- `Dockerfile`: `node:24-alpine`, `apk add file` (MIME detection), `npm ci --omit=dev`, non-root user, `NODE_ENV=production`, `PORT=8010`, `npm start`.
- The gateway's `REFERENCE_DOWNLOADER_API_URL` must use the deployed port (8010 in the image; 8005 is the code default for local runs).
- The image includes `eng.traineddata` (the Tesseract English model, about 5 MB) for offline OCR.
- The worker runs inside the API process; there is no separate worker deployment.

## Deploy steps

1. Build the image from the environment branch.
2. **Run migrations** (`npm run migrate:<env>`) before starting the new code.
3. Start the container. The SQS queues, buckets and FlattenPDF Lambda must already exist.
4. Use migrations, never `npm run setup`, on shared environments (setup recreates tables).

## CI

`.github/workflows/pr-checks.yml`: on PRs and pushes to `main`, `dev`, `qa` and `uat`, Node 22 → `npm ci` → `npm run lint` → `npm run test:coverage`. Thresholds in `jest.config.js`: lines 95, statements 95, functions 90, branches 85. The Husky pre-push hook runs lint and coverage.

## Configuration variable names

Start from `.env.sample` and set the full list below.

| Group | Names |
|---|---|
| Runtime | `NODE_ENV`, `PORT`, `APP_TARGET`, `LOCAL_TENANT`, `BASE_URL`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` |
| Database | `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_DIALECT`, `DB_NAME`, `DB_PORT`, `USE_SECRET_MANAGER`, `DB_SECRET_NAME`, `AWS_REGION` (Secrets Manager default `us-east-1`) |
| SQS | `SQS_ORIGINAL_DOC_URL`, `SQS_FETCH_METADATA_URL`, `SQS_FLATTEN_UPLOAD_URL`, `CONCURRENT_QUEUE_TASKS` |
| Storage | `WASABI_ACCESS_KEY`, `WASABI_SECRET_KEY`, `WASABI_REGION`, `WASABI_ENDPOINT`, `WASABI_TEMP_BUCKET_NAME`, `WASABI_REFERENCES_BUCKET_NAME`, `WASABI_WIPO_DOCS_BUCKET`, `JUNE_REQUESTS_BUCKET_NAME`, `REQUEST_ZIP_EXPIRY`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| Logging | `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET` (CloudWatch uses this name for the secret key) |
| Internal services | `EPO_BDSS_URL`, `PATENT_CLI_URL`, `FLATTEN_PDF_URL`, `FLATTEN_MAX_PAGES`, `IP_ASSISTANT_URL` |
| PDF | `ABSTRACT_FILE_NAME`, `ORIGINAL_DOCUMENT_NAME`, `MAX_PDF_PART_SIZE` |
| Email | `EMAIL_HOST`, `EMAIL_USERNAME`, `EMAIL_PASSWORD`, `EMAIL_SENDER`, `COLLAB_SPACE_TEAM_EMAIL`, `COLLAB_SPACE_APPGEN_TEAM_EMAIL` |
| Swagger (development and qa) | `AUTH_USERNAME`, `AUTH_PASSWORD`, `SECRET` |

The page size is fixed to US Letter in code.

## Infrastructure dependencies

- Postgres, shared with `patent-asistant-apis`.
- 3 SQS queues.
- 3 Wasabi buckets and 1 S3 bucket.
- The FlattenPDF Lambda.
- The EPO BDSS and patent-cli services.
- The IP Assistant.
- SMTP relay.
- CloudWatch.
- The scheduler that calls `/sendReminderEmails`.

## Rollback

Code rollback is a redeploy of the previous image. Migrations have `down` steps; additive, nullable columns can stay in place when rolling back code.
