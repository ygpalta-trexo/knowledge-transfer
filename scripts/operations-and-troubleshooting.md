# scripts/drm — Operations and troubleshooting

## Where to look

- **Script log:** `LOGS_DIRECTORY/drm_cron_YYYY-MM-DD.log` on the job host. Level INFO, lines prefixed with `[tenant]`. It lists the files found, missing, ignored and selected per logical name, each load, and the drm-apis response.
- **stdout:** credential and tenant-lookup messages are printed before the tenant logger starts, so check the scheduler's captured output too.
- **drm-apis logs** (CloudWatch) for the audit itself, and the drm-apis report email in its tech mailbox.
- **Tenant DB:** `file_upload_status` (which types are loaded and active), the staging tables, and `audit_result` / `audit_result_tm`.
- **Tenant SFTP:** `drop/` (not yet processed) and `processed/` (moved by drm-apis after an audit, with a timestamp suffix).

## Emails the job sends

| Situation | To | Subject | Run continues? |
|---|---|---|---|
| DB credentials cannot be loaded | `EMAIL_RECIPIENT` | Database credential error | No |
| Tenant lookup or super-admin DB error | `EMAIL_RECIPIENT` | Database error | No |
| SFTP connection fails | `EMAIL_RECIPIENT` | Failed to connect to SFTP server | No |
| Digest rule: more than one XML or Excel file in `drop/` | `EMAIL_RECIPIENT` | Invalid files in SFTP for Digest rule | No |
| `drop/` is empty | `EMAIL_RECIPIENT` | DRM Audit Not Executed - No Files Found (\<tenant\>) | No |
| Pair or digest rule: USPTO correspondence or PL Digest file missing | Tenant contact | DRM Audit Execution Failed - Action Required | No |
| Pair or digest rule: any required file missing | `EMAIL_RECIPIENT` | Missing required files | No |
| Recognised files that the rule does not use | `EMAIL_RECIPIENT` | Files Ignored in SFTP | Yes |
| drm-apis returns an error or cannot be reached | `EMAIL_RECIPIENT` | Failed to run audit for tenant … / Error during audit for tenant … | No |
| Any other exception (parse error, load error, "already uploaded") | `EMAIL_RECIPIENT` | Error processing files for Tenant: … | No |
| Audit returned patent or trademark results | Tenant contact | DRM Audit Completed for \<tenant\> \<date\> – Summary of Observations | — |

Some cases are recorded in the log only: an incomplete SFTP configuration for the tenant, a missing SFTP root folder, an audit with no result, and a failed email send. If an expected email did not arrive, start with the log.

## Common situations

1. **A client renamed an export.** Recognition is a substring match on known names (see [architecture.md](architecture.md#step-by-step)), so an unknown name is skipped. Under `pair` or `digest`, the run stops with a "missing file" email; under other rules, the audit runs and the tenant email lists the file as missing. **Fix:** rename the file on SFTP, or add the new name to `get_file_name` in `sftp_uploader.py`.
2. **A client changed a column layout.** Columns are mapped by position, so compare the export's header row with the mapping whenever a client announces a format change, or when a tenant's observation count jumps. **Fix:** add a tenant custom mapping (`tenant_custom_mappings`), or update both mapping copies for a change that applies to all tenants.
3. **"File \<type\> has already been uploaded."** An active `file_upload_status` row exists when loading starts. Check the log for the `cancel_audit()` step, and whether another run or a UI upload was active for the same tenant; then re-run.
4. **drm-apis failed or was unreachable.** A failure email is sent. The data stays loaded but not audited, and the files stay in `drop/`. **Fix:** once drm-apis is healthy, re-run the script. It calls `cancel_audit()` and reloads, so a re-run is safe.
5. **The run seems stuck.** It waits for drm-apis to answer. Check drm-apis logs for the audit in progress.
6. **A load fails midway** (bad file, DB error). The files loaded before it stay in the staging tables and no audit runs. Fix the file and re-run; the next run's `cancel_audit()` clears the FIP tables and the status rows.
7. **Duplicate files.** For most rules the newest file of each type is used, and drm-apis moves all the files, including older copies, to `processed/`. The digest rule refuses duplicates with a failure email.
8. **A user's UI upload disappeared.** A scheduled run's `cancel_audit()` clears the tenant's pending batch; see [architecture.md](architecture.md#data-written).

## Housekeeping

- Clear old downloads from the host's `/tmp` regularly; each run writes the client files there.
- Watch the log folder's size.

## Manual run checklist

1. Confirm the tenant code, the rule, and whether `--mark-previous-corrected` is wanted (it closes every open observation from earlier batches).
2. Look at the tenant's SFTP `drop/`: are the expected files there, with recognisable names, and only one of each for digest?
3. Make sure nobody is uploading or auditing for that tenant in the DRM UI at the same time.
4. Run the job from the configured environment. Watch the log for "Selected file for logical name", each "Uploaded", and the drm-apis response.
5. Check that the files moved to `processed/`, that `audit_result` / `audit_result_tm` have a new batch, and that the tenant email was sent.
