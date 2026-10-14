# drm-apis — Jobs and queues

The baseline is in `drm-apis: docs/queue.md`. There are **20** queues, including `FIP Audit Dump` and `OpenDocketTM`; `WorkingStmtPaQueue` runs at concurrency 1.

## Queue technology

Bull 4 on Redis: one Bull queue per source-file type, each with its own Redis connections, registered in the API process after credentials resolve (`registerQueueProcessors`).

## Producers and consumers

| Producer | When | Payload |
|---|---|---|
| `uploadHandler` → `handleXlsxFile` / `handleXmlFile` | UI upload, or SFTP ingestion | `{tenant, cells, format, pendingCreate, fileObj, errorOccuredWhileUpload}`; `cells` holds the parsed rows |

The consumer is `queue/queueProcessor.js` `processQueue(data, fileName)`. It maps rows to columns (`queueMappings.json`, overridable per tenant), runs `bulkCreate(..., {returning:false})`, then sets the status row to `Pass` or `Fail` with the reason.

## Processing behaviour

| Aspect | Behaviour |
|---|---|
| Outcome | Every job ends with the status row set to `Pass` or `Fail`; `reason_of_failure` holds the error |
| Retention | Completed jobs are kept (`removeOnComplete: false`); failed jobs stay in Bull's failed set |
| Re-running | Re-uploading a file type adds its rows again; cancel the batch first (see [database.md](database.md#transactions)) |
| Concurrency | `CONCURRENT_QUEUE_TASKS` (default 4) per queue per process; each job is one `bulkCreate` |
| Ordering | Independent across file types; the audit waits on the `file_upload_status` rows |
| Status tracking | The `file_upload_status` row (`Pending` → `Pass`/`Fail`) plus Bull Board |
| Logging | Bull `failed` events are written to stdout (`console.error`) |

Every replica runs every processor, so workers scale with the API.

## Scheduled jobs ("cron")

`cron/index.js` defines four `CronJob`s (timezone Asia/Kolkata) as reference schedules. They are not started in-process; the same functions run when the scheduler calls the HTTP routes.

| Reference schedule | Function | HTTP route |
|---|---|---|
| Mon–Fri 11:00 | `processFilesForAllTenants` (SFTP pull, all tenants) | `POST /cron/upload?tenant=` (one tenant) |
| Tue 11:15 | `switchPairRulesOnly` (configured production tenant, rule id 16 only) | `POST /cron/switchPAIRRule?tenant=` or `/cron/run_audit?rule=pair` |
| Mon–Fri 11:30 | `runAuditForAllTenants` | `POST /cron/run_audit?tenant=` |
| Tue 11:55 | `setAllRulesActive` (patent rules) | `POST /cron/setAllRulesActive?tenant=` (patent and trademark) |
| — | — | `POST /cron/deleteFromArchives` (all tenants, 90 days) |

The `/cron/run_audit` route is also what the `scripts` DRM job calls after it loads SFTP files itself; see [../scripts/architecture.md](../scripts/architecture.md).

### In-process retry timer

When any active upload is `Pending`, `runAudit` schedules itself again with `setTimeout(15 min)`, up to 5 attempts. The timer lives in the process that received the call, so avoid deploying while a scheduled audit is waiting (up to 75 minutes after the `/cron/run_audit` call).

### Scope

Scheduled audits cover tenants **with SFTP config**. UI-only tenants run audits from the UI.
