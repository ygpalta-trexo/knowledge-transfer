# reference_downloader_apis — API and integrations

The repo's `docs/API_REFERENCE.md` has the validation table. The inventory below is taken from `routes/index.js`.

## How callers reach it

| Caller | Path |
|---|---|
| Collaboration Space UI | Gateway `/tenantAPI/reference-downloader/<route>` → `REFERENCE_DOWNLOADER_API_URL/<route>`; the gateway checks the JWT and the `COLLABORATE` app |
| Lex chatbot Lambda | `REFERENCE_BASE_URL` (POST `/requests`), with `X-User` / `X-Tenant-Name` from the chat session |
| `patent-asistant-apis` | `COLLAB_API_URL` (POST `/pctrequests`) |

Every route except `/health` (and the Swagger paths) expects `X-User` (JSON) and `X-Tenant-Name`.

## Endpoint inventory

Routes are mounted at `/`.

| Method | Route | Purpose | Notes |
|---|---|---|---|
| GET | `/health` | Liveness | |
| POST | `/requests` | Create a reference request | 400 if the id exists or all references are invalid; 201 with an HTML `message` (SLA text or download link); may build the ZIP within the call |
| GET | `/requests` | Team queue | Requests created in the last 30 days or not yet completed, with their references |
| GET | `/requests/pending` | Requests with status `Pending` | |
| PUT | `/requests/sla/:requestId` | Change the SLA | Emails the user and the team |
| POST | `/files/upload/:requestPatentId` | Upload abstract and/or document PDFs and/or metadata | Up to 100 MB per file; MIME check; queues a flatten job for text PDFs |
| POST | `/metadata` | Bulk metadata rows | Processes rows in order |
| PUT | `/requests/submitForReview` | → `Under Review` | 400 "All components not available" until every reference is resolved |
| PUT | `/requests/complete` | → `Completed` (ZIP + callback) | Returns the ZIP URL in the error if the callback does not succeed |
| GET | `/zip/:requestId` | Stream a fresh ZIP | Built on demand |
| GET | `/patent/:patentNumber` | One reference as a ZIP | 429 for an invalid number format; 422 if not available |
| PUT | `/patent/invalid` | Flag the global `Patents` row invalid | Shows as an error line in later ZIPs for that reference, for every tenant |
| PUT | `/requestpatent/invalid` | Flag one request's reference invalid, with a reason code | Codes from `utils/invalidReasons.js` |
| GET | `/sendReminderEmails` | Send SLA reminders to the team | Called by the scheduler |
| POST | `/pctrequests` | Create a PCT request | 409 on a duplicate id; emails the AppGen team |
| GET | `/pctrequests` | PCT requests from the last 7 days | |
| PUT | `/pctrequests` | Upload WIPO documents or mark them unavailable | Applied to all open requests with the same PCT number |
| PUT | `/pctrequests/status` | Move to `Under Review` or similar | |
| PUT | `/pctrequests/submit` | Complete eligible requests; notify the IP Assistant | |
| GET | `/pctrequests/download/:requestId` | ZIP of the WIPO docs, or a single file via `?file=` | Streams from Wasabi |
| POST | `/api/v1/auth/token` | Swagger UI token (development and qa) | |
| GET | `/api-docs` | Swagger UI (development and qa) | |

## Notes on the repo docs

1. Routes are at `/` (`app.use('/', routes)`), not `/api`.
2. `GET /requests` returns an array and takes no paging parameters.
3. The 100 MB limit applies to `/files/upload`.
4. The queue layer is SQS, as `ENDPOINT_MAP.md` shows.

## External systems

| System | Env names | Call |
|---|---|---|
| EPO BDSS (bulk metadata) | `EPO_BDSS_URL` | `POST /api/publication/bulk {references}` |
| patent-cli | `PATENT_CLI_URL` | `GET /patent-cli/orig_doc/:ref` → download URL → GET the PDF; fallback `GET /patent-cli/orig_doc_epo/:ref` |
| FlattenPDF Lambda | `FLATTEN_PDF_URL`, `WASABI_TEMP_BUCKET_NAME`, `WASABI_REFERENCES_BUCKET_NAME` | `POST` (dpi 150, JPEG quality 35, `max_pages` 1500, page width/height) |
| IP Assistant | `IP_ASSISTANT_URL` | `POST /postEmail {output, errorMessages, requestId, status}`; `POST /usFilingPostCollab` |
| Wasabi | `WASABI_ACCESS_KEY`, `WASABI_SECRET_KEY`, `WASABI_REGION`, `WASABI_ENDPOINT`, bucket names | S3 API |
| AWS S3 (ZIPs) | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `JUNE_REQUESTS_BUCKET_NAME`, `REQUEST_ZIP_EXPIRY` | Multipart upload plus presigned GET |
| AWS SQS | `SQS_ORIGINAL_DOC_URL`, `SQS_FETCH_METADATA_URL`, `SQS_FLATTEN_UPLOAD_URL`, `AWS_REGION` (default `us-east-1`) | Send / long-poll receive / delete; see [jobs-and-queues.md](jobs-and-queues.md) |
| SMTP | `EMAIL_HOST`, `EMAIL_USERNAME`, `EMAIL_PASSWORD`, `EMAIL_SENDER`, `COLLAB_SPACE_TEAM_EMAIL`, `COLLAB_SPACE_APPGEN_TEAM_EMAIL` | Nodemailer port 587, STARTTLS |
| CloudWatch Logs | `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET`, `AWS_REGION`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` | Custom transport (shared design with drm-apis) |

S3 and CloudWatch read the AWS secret key from **different** variable names: `AWS_SECRET_ACCESS_KEY` for S3 and `AWS_ACCESS_KEY_SECRET` for CloudWatch. Set both.

## Headers forwarded downstream

The worker forwards the original `X-User` / `X-Tenant-Name` / request id to EPO BDSS, patent-cli and the IP Assistant.
