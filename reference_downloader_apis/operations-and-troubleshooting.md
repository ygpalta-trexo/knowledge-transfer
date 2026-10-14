# reference_downloader_apis — Operations and troubleshooting

## Recurring and operational jobs

| Job | Trigger | Check |
|---|---|---|
| SLA reminder | `GET /sendReminderEmails`, daily | Team mailbox "Reference Request Due Soon: <id>" |
| SQS pollers | Start after `app.listen`; log "Starting SQS Poller for … with concurrency N" | A "…_URL not set. Skipping …" warning means that queue is not being consumed |
| PDF normalisation backfill | Manual (`npm run backfill:<env> -- --dry-run` first) | Script output |
| Migrations | Manual (`npm run migrate:<env>`) at deploy | `SequelizeMeta` table |

## Logs and observability

- **CloudWatch:** `${NODE_ENV}/server` holds the shared logger, including all worker logs. Per-tenant groups (named by tenant code) hold request logs. The stream is `reference-downloader-YYYY-MM-DD`. A console transport is added with `NODE_ENV=development` + `APP_TARGET=local`.
- **Log prefixes to search:**
  - `[DownloadProcessor for request <id>]`
  - `[MetadataProcessor for request <id>]`
  - `[UploadProcessor for reference <ref>]`
  - `Auto completion check for job <ref>`
  - `Failed to Update IP Assistant for <id>`
  - `matched N of M Patents rows` (metadata UPDATE summary)
- **Correlation:** the `request-id` header becomes `req.id` and is forwarded downstream; worker logs are keyed by request id and reference.
- **Queue health:** SQS CloudWatch metrics (queue depth, oldest-message age).
- Credential problems at startup are also printed to the console.

## Runbooks

### A request is stuck in "To Do" / "In Progress"
1. Find it in the team queue (`GET /requests`) and read each reference's statuses and remarks.
2. Document status `Downloading` for a long time: search `[DownloadProcessor for request <id>]` to see whether the job is running or ended with an error. If it ended, the team uploads the document, or a new request for the reference queues it again.
3. Metadata `Pending` with no remark: the metadata fetch did not succeed. The team enters the metadata (`POST /metadata` or the upload dialog), or a new request for the reference re-queues it.
4. Everything `Available`, but not auto-completed:
   - Requests with non-US references are completed by the team.
   - Search `Failed to Update IP Assistant` for a callback problem; the status returns to its previous value.

   Fix: the team runs submit for review, then complete.
5. Status `Processing` for a long time: the ZIP build or callback was interrupted (for example by a restart). Reset the status to its previous value in the DB, then re-trigger (the team completes it, or a new upload triggers `handleJobCompletion`).

### "URL Generated. Could not Update JUNE IP Assistant"
The ZIP exists; the IP Assistant `/postEmail` call did not succeed. The error response contains the ZIP URL. Check `patent-asistant-apis` health and `IP_ASSISTANT_URL`, then complete again; a new ZIP is generated each time.

### Document "Pending: Document too small / Invalid PDF / File exceeds N-page limit"
By design ([product-and-features.md](product-and-features.md#document-rules)). The team uploads the document by hand.

### Flatten errors
The status returns to `Pending` and the log shows `Error calling Flatten PDF Lambda - Status: …`. Check the Lambda's logs and timeout, and that the object exists in the temp bucket at `<ref>/<file>`.

### Uploaded file rejected "Only pdf, excel, xml files are allowed"
MIME detection uses the `file` binary installed in the image. If every file is rejected, check the image build rather than the file.

### Service will not start
Look for `[FATAL] Failed to resolve DB credentials` on the console. In Secrets Manager mode, check `DB_SECRET_NAME`, IAM, and the secret keys (`username`, `password`, `host`, `engine`, optional `port` / `dbname`).

## Graceful shutdown

On SIGTERM, HTTP closes, the pollers stop, and in-flight messages get 30 s to finish. A document job cut off there returns to SQS after 300 s and runs again.

## Operational checklist

- [ ] `/health` returns 200.
- [ ] The startup log shows all three pollers started.
- [ ] SQS oldest-message age is low on all three queues.
- [ ] No requests in `Processing` for more than 15 min.
- [ ] Team queue: no requests past `slaTime` without action.
- [ ] CloudWatch: `Failed to Update IP Assistant` and `Error calling Flatten PDF Lambda` counts are normal.
- [ ] Before each release, run the pending migrations.
