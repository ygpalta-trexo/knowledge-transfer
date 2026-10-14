# docdb_ingestion — Operations and troubleshooting

How to start, watch and stop workers, and the retry commands, are in `docdb_ingestion: docs/server_run_guide.md`. This file adds how to check freshness, recognise common situations and re-process data.

## Recurring job

| Job | Command | Cadence |
|---|---|---|
| Front-file weekly update | `frontfile_runner.py` (`--mode catchup` applies every outstanding week; `--mode latest` only the newest) | Weekly (the guide suggests Monday 06:00) |

The run report email (`EMAIL_RECIPIENT`) is the notification for each run. If no email arrives, check the log to see whether the job ran and whether SMTP is configured.

## Is the data current?

Use the tracking tables; queries on `patent_documents` without an indexed key read the whole table.

```sql
-- Latest front-file deliveries applied
SELECT week_number, delivery_name, runner_mode, status, docs_upserted, docs_deleted, docs_skipped, completed_at
FROM frontfile_delivery_audit ORDER BY completed_at DESC LIMIT 10;

-- Anything unfinished or failed, per delivery
SELECT delivery_id, status, count(*) FROM delivery_files
WHERE status <> 'COMPLETED' GROUP BY 1, 2 ORDER BY 1;
```

Compare the newest `week_number` with the current EPO week. `--mode catchup --dry-run` prints the plan, with each delivery marked `new` / `pending` / `done`, without writing.

## Common situations

| Symptom | Cause | Check | Action |
|---|---|---|---|
| Report or log says `N file(s) in FAILED state; --retry-failed not set` | An earlier run left FAILED files in that delivery | `delivery_files.error_message` (full traceback) | Fix the cause, then `frontfile_runner.py --mode catchup --retry-failed` |
| Report says "halted on week …"; later weeks "not attempted" | A file failed during this run | Halt banner and traceback in `logs/<start-date>/pipeline_frontfile.log` | As above; later weeks run once the week completes |
| Process alive, log silent for a long time | A stalled download from EPO | `monitor_workers.sh` (temp dir size not growing); `ps` | `kill <pid>`, then re-run; a file in `DOWNLOADING` restarts its download |
| `No space left on device` | `EPO_TEMP_DIR` or the system temp dir is full | `df -h` on both | Free space, or point `TMPDIR` at a larger volume, then `--retry-failed` |
| `XMLSyntaxError … Entity '…' not defined` | A new named entity in EPO XML | Log line with the entity name | Add it to `_EPO_ISO_ENTITIES` in `stream_processor.py` with a test, then `--retry-failed` |
| `Bad ZIP file …` in the log | A corrupt inner volume; it is recorded as completed with no documents | `grep -r "Bad ZIP file" logs/` | Delete that `ingestion_checkpoints` row, set its `delivery_files` row to `PENDING`, and re-run; completed sibling volumes are skipped |
| `No internal ZIP volumes found` | Unexpected delivery layout; the file is recorded as completed | Log line | Inspect the delivery structure, then re-run as above |
| `password authentication failed` | The secret rotated, or the password is wrong | Log shows one automatic re-fetch and retry | If it persists, check the secret contents (without printing them) and the IAM role |
| `UndefinedTable` during planning or dry run | The schema has not been created in this database | — | `python setup_db.py` |
| No report email | Mail config | Log: `SMTP not configured …` or `No recipient configured` | Set the `EMAIL_*` variables |
| Backfile run finished but data is missing | Some files did not complete | `delivery_files` statuses; log `FAILED file(s)` | `pipeline run --retry-failed` over the same range |
| A lookup returns a bare record with no dates or parties | A void (CV/DV) stub row | `pub_doc_id` starts with `VOID_`, `source_status = 'SKIP'` | Expected for withdrawn publications |

## Re-processing

- **One outer file:**
  1. `UPDATE delivery_files SET status='PENDING', error_message=NULL WHERE file_id = …;`
  2. To re-parse volumes that already completed, also `DELETE FROM ingestion_checkpoints WHERE main_zip_id = …;`
  3. Run again: `pipeline run` for the backfile (with a range that includes it), or `frontfile_runner.py --mode catchup` for the front-file.
- **`--force-sync`** re-syncs finished deliveries, so files EPO added later get registered and processed; files already `COMPLETED` are still skipped.
- **To re-apply a whole week:** reset that week's `delivery_files` rows to `PENDING` and delete their `ingestion_checkpoints` rows, then run `catchup`. Re-apply weeks oldest first.
- **Parallel backfile:** ranges are 1-based positions in the `delivery_files` list for one delivery, ordered by `file_id`. Keep ranges disjoint and give each worker its own `EPO_TEMP_DIR` on the command line (see [deployment-and-environments.md](deployment-and-environments.md)).

## Logs

- The application log goes to `logs/<YYYY-MM-DD>/pipeline[_<worker>].log`, dated by the day the process started, plus stdout. `nohup` output goes to `logs/<name>.out`.
- `delivery_files.error_message` keeps the traceback of each failed file.
- `logs/` is git-ignored; archive or prune it periodically.

## Weekly checklist

1. The report email arrived and says completed, or completed with skips.
2. `frontfile_delivery_audit` has this week's rows; `delivery_files` has no `FAILED`, `DOWNLOADING` or `PARSING` rows older than a day.
3. `grep -r "Bad ZIP file\|No internal ZIP volumes" logs/<date>/` finds nothing.
4. Free disk on the temp volumes, and database storage headroom.
5. After a halt: fix the cause, run with `--retry-failed`, and confirm the later weeks ran.
