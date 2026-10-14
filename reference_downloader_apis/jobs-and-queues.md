# reference_downloader_apis — Jobs and queues

The queue layer is **AWS SQS** (it replaced Redis/Bull on 2026-01-14, `99e4a25`, "replace bull with AWS SQS poller and processor").

## Queue technology

Three standard SQS queues, consumed by a long-polling consumer (`worker/sqsPoller.js`) inside the API process (`worker/index.js`).

| Queue env | Producer | Consumer | Concurrency | Visibility timeout |
|---|---|---|---|---|
| `SQS_ORIGINAL_DOC_URL` | `POST /requests` (per new or retry reference) | `queues/originalDocDownloadProcessor.js` | 1 | 300 s |
| `SQS_FETCH_METADATA_URL` | `POST /requests` (one message with every pending reference) | `queues/fetchMetadataQueueProcessor.js` | 1 | 300 s |
| `SQS_FLATTEN_UPLOAD_URL` | `POST /files/upload` (manual text PDFs) | `queues/flattenUploadProcessor.js` | `CONCURRENT_QUEUE_TASKS` (default 4) | 900 s |

A queue whose URL is unset is skipped with a startup warning.

## Message payloads

- **original-doc:** `{patentNumber, requestId, patentId, sourceData, user, tenant, id}`
- **fetch-metadata:** `{patentNumbers[], requestId, sourceData, user, tenant, id}`
- **flatten-upload:** `{refNum, file, patentId, user, tenant, id, targetPageSize}`

Files go to the Wasabi temp bucket first, and messages carry references to them, to stay within SQS size limits.

## Processing model

| Aspect | Behaviour |
|---|---|
| Message lifecycle | Each message is processed once, then deleted by the poller |
| Unfinished references | When a fetch does not succeed, the reference is left in a state the next request for it re-queues (metadata `Pending` without a remark, or document and metadata "Not available at source"). The team can also complete it by hand |
| Idempotency | The document path writes the same Wasabi keys and statuses on a re-run; the metadata UPDATE is idempotent; completion is guarded by the `Processing` compare-and-set |
| Throughput | The document and metadata queues run one message at a time per process; more replicas add throughput |

## Completion as a job

Every processor ends with `handleJobCompletion(ref)`, which re-checks the requests in `Jobs[ref].requestId`. Auto-completion is described in [architecture.md](architecture.md#completion-coordination).

## Scheduled and manual work

| Job | Trigger | Notes |
|---|---|---|
| SLA reminder | `GET /sendReminderEmails`, once a day | Logic in [product-and-features.md](product-and-features.md#sla-reminder) |
| PDF page-size backfill | `npm run backfill[:env] [-- --dry-run --limit=N --batch=N]` | Re-runnable; normalises stored reference PDFs to the uniform size (`scripts/normalizeStoredPdfs.js`). Run with `--dry-run` first |
