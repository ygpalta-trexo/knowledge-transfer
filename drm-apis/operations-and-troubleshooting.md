# drm-apis — Operations and troubleshooting

`drm-apis: docs/operations.md` covers health, startup and shutdown. `POST_MERGE_STEPS.md` §8 has the post-deploy checks. This file adds runbooks by symptom.

## Recurring jobs

| Job | Trigger (reference time, IST) | Expected effect | Where to see it |
|---|---|---|---|
| SFTP pull | `POST /cron/upload?tenant=` (Mon–Fri 11:00) | Files from `/data/drm/drop` loaded, status rows `Pass` | `file_upload_status`; log "Total files found for …" |
| Audit run | `POST /cron/run_audit?tenant=[&rule=]` (Mon–Fri 11:30) | Procedures run; files moved to `processed/`; report email | Tech mailbox "Run Audit Job for <tenant>" |
| PAIR-only / re-enable all | Tuesday 11:15 / 11:55 | Rule `status` flips | Emails "PAIR Rules Activated", "All Rules Re-Activated" |
| 90-day purge | `POST /cron/deleteFromArchives` | Old results deleted for all tenants | Email "Delete Audit Results Cron" |

## Logs and observability

- **CloudWatch log groups:** `${NODE_ENV}/server` (shared logger, cron and queue logs), and one group per tenant code (request logs). The stream name is `drm-apis-YYYY-MM-DD`.
- **Log format:** JSON with `requestId`, `tenant` and `username` (since July 2026). Metric filters written for the older `[id] [tenant] msg` format need updating (`POST_MERGE_STEPS.md` §8).
- **Correlation:** the gateway sends `Request-Id`, which drm-apis reuses as `req.id`. Search both services' logs by that id.
- **Container stdout:** Bull `failed` events, email errors and some controller and cron messages are written to stdout; check the container logs as well as CloudWatch.
- **Metrics:** `Custom/DailyRotateCloudWatchTransport` `LogEventsSent` / `LogEventsFailed`.
- **Dead-letter file:** `utils/logs/cw_dead_letter.log` inside the container.

## Runbooks

### Upload stuck in `Pending`
Check in this order:
1. Does the file name (text before the first `.`) match one of the 20 queue keys exactly (case-sensitive for UI uploads)? Log line: `No queue function found for file name`.
2. For XML: does it contain `Outgoing_Correspondence_List.OutgoingCorrespondence`?
3. Bull Board `/admin/queues`: is the job waiting (processors not registered, or Redis down), active (a large file) or failed?
4. Is Redis reachable from the container?
5. For SFTP uploads, look for `Error processing files for Tenant` in `${env}/server`.

Fix: correct the file and re-upload, or cancel the batch. `cancel_audit` clears all status rows and FIP source data for the tenant, so tell the user first.

### Upload `Fail` with "Please verify that the file fields are correct"
The worker's `bulkCreate` failed. Typical causes:
- a column missing in this tenant's schema, for example a tenant-optional column such as DRM-379;
- a value too long for a non-string column;
- the tenant's `tenant_custom_mappings` points at the wrong column indexes.

`reason_of_failure` holds the DB error message. Compare the tenant's schema with the model (`database/*.model.js`).

### "Only Excel, CSV and XML files are allowed"
MIME validation. Check that the file is a real OOXML `.xlsx` saved from Excel, not a renamed `.xls`.

### Report email says "File Upload pending … Max attempt limit reached"
A file stayed `Pending` through 5 × 15 min; follow the first runbook. The retry timers live in memory, so a deploy during the window also stops them.

### Audit fails with "relation "serial" already exists"
An earlier run stopped mid-procedure, or two runs overlapped. Confirm nothing is running (`pg_stat_activity`), then run `DROP SEQUENCE serial;` in that tenant DB.

### Audit fails with "procedure run_audit_cpi does not exist"
The tenant does not have that procedure applied. Trademark CPI runs use `run_audit_tm_cpi`; check `DB-objects: DRM/` for the tenant's procedure set.

### Tenant requests return 403 "<tenant> isn't registered"
`Tenant.tenantCode` does not match the Origin subdomain the gateway derived. A tenant added after boot is picked up on its first request, with no restart.

### Service will not start
- `[FATAL] Failed to resolve DB credentials`: with `USE_SECRET_MANAGER=true`, check `DB_SECRET_NAME`, the IAM grant, and that the secret has all four keys (`username`, `password`, `host`, `engine`).
- `Model "…" accessed before database initialization`: some module touches a super-admin model at require time.
- If the super-admin DB is unreachable the service still starts, and tenant requests fail at lookup; check DB connectivity.

### Emails not arriving
Check container stdout for SMTP auth errors (`EMAIL_CONFIG_AUTH_*`).

## Operational checklist

Daily or after deploys:
- [ ] `GET /health` returns 200.
- [ ] Startup logs show `Admin Database Connected` and one `Connected to … Database` per tenant.
- [ ] Bull Board: waiting and failed counts are not growing.
- [ ] The tech mailbox got an audit report for each SFTP tenant on weekdays.
- [ ] No `file_upload_status` rows `Pending` for more than 1 h.
- [ ] Redis memory and Postgres connection counts are within normal range.

## Shutdown

Stop traffic before stopping the service. The CloudWatch transport flushes on SIGINT/SIGTERM. Bull retries jobs that were active at shutdown once it detects them as stalled; check the status rows of those files afterwards.
