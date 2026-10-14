# drm-apis — API and integrations

The per-endpoint request and response shapes are in `drm-apis: docs/api.md`. This file adds how callers reach each route, recent additions, and the external systems.

## How the routes are reached

- **Browser:** `https://<tenant>.<company-domain>/…` → gateway `/tenantAPI/drm/<route>` → `DRM_API_URL/<route>`.
- The gateway sets `X-Tenant-Name` from the Origin subdomain, `X-User` from the verified JWT (`personid`, `id`, `email`, `role`, `apps`, …) and `Request-Id`. Multipart bodies are streamed through unparsed.
- The `/cron/*` routes are called by the scheduler and the `scripts` DRM job; `/admin/queues` is the operations UI.

## Endpoint inventory

"Gateway" means the caller is an authenticated tenant user with the DRM app; drm-apis reads the identity from the forwarded headers. "Internal" routes are for schedulers and operations.

| Method | Route | Purpose | Caller | Dependencies |
|---|---|---|---|---|
| GET | `/health` | Liveness text `OK` | Load balancer | — |
| * | `/admin/queues` | Bull Board UI and its JSON API | Internal (ops) | Redis |
| GET | `/fetchAuditResult/v2` | Patent and/or TM results, tenant mapping, column preferences | Gateway | Tenant DB, super-admin DB |
| GET | `/fetchMappedRules` | Rules matching the active uploads | Gateway | Tenant DB |
| POST | `/updateMapRule` | Toggle one rule (Joi-validated) | Gateway | Tenant DB |
| POST | `/updateAuditResult` | Archive and annotate results (Joi-validated) | Gateway | Tenant DB |
| POST | `/deleteFromArchives` | Delete results by id (`ids` array) | Gateway | Tenant DB |
| GET | `/getActiveFilesUploaded` | Upload status list | Gateway | Tenant DB |
| GET | `/runActiveRules` | Run the audit procedure(s); returns when they finish | Gateway | Tenant DB procedures |
| GET | `/cancelActiveRules` | `CALL cancel_audit()` | Gateway | Tenant DB |
| POST | `/createActivityCode`, `/editActivityCode`, `/deleteActivityCode`; GET `/fetchActivityCode` | Activity code CRUD | Gateway | Tenant DB |
| POST | `/uploadFile` | Upload and queue (Multer, MIME and XLSX structure checks) | Gateway | Tenant DB, Redis |
| PUT | `/saveColumnPreferences` | Upsert the user's visible columns (DRM-189) | Gateway | Tenant DB |
| POST | `/cron/upload` | SFTP pull for `?tenant=` | Internal (scheduler) | SFTP, tenant DB, Redis |
| POST | `/cron/run_audit` | Optional rule switch (`&rule=pair|cpi|all`), then audit for `?tenant=` | Internal (scheduler, `scripts` job) | SFTP, tenant DB, SMTP |
| POST | `/cron/switchPAIRRule`, `/cron/setAllRulesActive` | Rule switching for `?tenant=` | Internal (scheduler) | Tenant DB, SMTP |
| POST | `/cron/deleteFromArchives` | Purge results older than 90 days, all tenants | Internal (scheduler) | Every tenant DB, SMTP |

## Additions to `docs/api.md`

1. `GET /fetchAuditResult/v2` returns all rows of `audit_result` and/or `audit_result_tm`, ordered by `id`; `limit`, `page` and `offset` are not used. The frontend filters archived rows.
2. It also returns `data.columnPreferences` and reads `?screen_name=` (default `AUDIT_RESULT`).
3. `POST /uploadFile`: an `.xlsx` whose name matches no queue, or an XML file without `Outgoing_Correspondence_List`, stays `Pending`; check the file name and structure (see [operations-and-troubleshooting.md](operations-and-troubleshooting.md)).
4. `/admin/queues` shows **20** queues, including `FIP Audit Dump` and `OpenDocketTM`.
5. `GET /runActiveRules` with an active `CPI_*` file and `category=patent|all` calls `run_audit_cpi`; trademark CPI runs call `run_audit_tm_cpi`.

## External systems

| System | Client / library | Config names | Call pattern |
|---|---|---|---|
| PostgreSQL (super-admin and tenants) | Sequelize 6 / `pg` | `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_DIALECT`, `SUPERADMIN_DB`, or `USE_SECRET_MANAGER` + `DB_SECRET_NAME` | Pools: API max 20 per tenant; cron max 10 per call |
| Redis | Bull 4 | `REDIS_HOST`, `REDIS_PORT` | 20 queues, each with its own connections |
| Tenant SFTP | `ssh2-sftp-client` 12 | Per-tenant `Tenant.sftpConfig` JSON (host, port, username, password) | `list`, `get`, `rename` under `/data/drm/{drop,processed}` |
| SMTP | Nodemailer 9, `service: "outlook"` | `EMAIL_CONFIG_AUTH_USER`, `EMAIL_CONFIG_AUTH_PASSWORD`, `TECH_EMAIL` | Notification and report emails |
| CloudWatch Logs and Metrics | AWS SDK v3 | `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` | Batched every 2 s; one `PutMetricData` per batch; a local dead-letter file at `utils/logs/cw_dead_letter.log` |
| AWS Secrets Manager | AWS SDK v3 | `DB_SECRET_NAME`, `AWS_REGION` (default `ap-south-1`) | Once per process |

## Long-running calls

`runActiveRules` and the `/cron/*` routes keep the HTTP request open until the procedure run (and, for cron, the SFTP transfer) finishes. `category=all` runs the patent and trademark procedures one after the other. `/cron/deleteFromArchives` loops over every tenant within one request.
