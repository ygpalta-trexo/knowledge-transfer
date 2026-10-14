# file_num_generation — Operations and troubleshooting

## Recurring jobs

None. There is no cron, queue or scheduled task.

## Operational dependencies

| Dependency | Needed at start-up | Needed per request |
|---|---|---|
| Postgres | Yes (`authenticate()`; the process exits if it fails) | Yes |
| Secrets Manager (if `USE_SECRET_MANAGER=true`) | Yes | Only when a new pooled connection opens after the 10-minute TTL; the cached value is used if the refresh fails |
| CloudWatch | No | No (batched in the background; undelivered batches go to `logs/cw_dead_letter.log`) |
| `trexo-apigateway` | No | Yes (identity headers) |

## Logs

- **Development:** console, plus `logs/YYYY-MM-DD.log`.
- **Other environments:** CloudWatch log group `<NODE_ENV>/server`, stream `number-generator`, JSON lines with `requestId` and `username`.
- **Correlation:** the gateway's `Request-Id` is reused as `requestId`. Every request logs `<METHOD> <url>` at `http` level on entry.
- **Useful messages:**

  | Message | Meaning |
  |---|---|
  | `File number generated successfully: <n>` | Single generation succeeded |
  | `Bulk file numbers generated successfully: <count>` | Bulk generation succeeded |
  | `Validation failed: …` | Joi 400 (field list only) |
  | `Error generating file number: <stack>` | 500 on generate |
  | `[configResolver] USE_SECRET_MANAGER…` | Which credential mode started |
  | `Database connection has been established successfully.` | Start-up reached the database |
  | `Failed to start server:` | Start-up failure, followed by exit code 1 |

- Header rejections (missing `X-User` / `X-Tenant-Name`) are logged by the root logger, without a request id.
- With `NODE_ENV=test` all logging is silent.

## Common situations

| Symptom | Cause | What to do |
|---|---|---|
| Container exits right after start | Database unreachable, wrong credentials, missing `DB_NAME`, or Secrets Manager not readable | Read the first log lines: `Failed to start server:` with a `CredentialResolutionError`, `EnvCredentialError` or Sequelize error. In Secrets Manager mode, check `DB_SECRET_NAME`, IAM permissions, `AWS_REGION` and the secret's JSON keys |
| Every request returns 400 `"X-User" header is required` | Called directly instead of through the gateway | Call through the gateway. For local tests, add `X-User` and `X-Tenant-Name` |
| `500 {"error": "Sequence counter not found"}` on single generate | The series has no `SequenceCounters` row | `SELECT * FROM "SequenceCounters"`. On a new database, run `db:seed:all`; otherwise insert the missing row at the correct position (below) |
| `500 … Maximum file number limit reached` | The series has used its last Y1 letter | Agree the next letters with the business owner, then extend `MATTER_TYPE_MAPPINGS` (ask first, per CONSTRAINTS.md) |
| The next number skips one | A number below the counter was deleted, or a generation stopped after the counter moved | Gaps are not reused; no action needed |
| The next number is one that was issued before | The highest number in the series was deleted, and the counter rewound | Designed behaviour; confirm in the logs |
| A series was emptied by deletes | The counter was set to the starting counter for that series | Check the counter row, and correct it with the procedure below if needed |
| Generate requests take ~30 s and then fail | Database connection pool exhausted | Look for `SequelizeConnectionAcquireTimeoutError` in the logs, then restart the service |

## Correcting a counter by hand

The counter stores the **last issued** position. Take a backup of `SequenceCounters` before changing anything.

1. Work out the series' effective sub-org: `OTHER` for GENERAL, COPYRIGHT and Matter with Opposing Parties; otherwise `US` or `UK`.
2. Find the highest issued number in **issue order**. The Y1 order is in [product-and-features.md](product-and-features.md#the-numbering-scheme). It matches alphabetical order only while a series is on its first letters.
   ```sql
   SELECT y1, y2, "sequenceNumber" FROM "FileNumbers"
   WHERE "matterType" = 'PATENT' AND "subOrg" = 'US'   -- for shared series, filter by matterType only
   ORDER BY y1 DESC, y2 DESC, "sequenceNumber" DESC LIMIT 5;
   ```
3. `UPDATE "SequenceCounters" SET "currentY1"=…, "currentY2"=…, "currentSequence"=… WHERE "matterType"=… AND "subOrg"=…;`
4. Generate one number in a non-production environment first, and confirm it is the expected next number.

Use this procedure for counter corrections rather than `POST /update-counters`.

## Operational checklist

- [ ] `GET /health` returns 200, and the start-up log shows `Database connection has been established successfully.` (the health check does not query the database).
- [ ] `SequenceCounters` has 9 rows, and each `currentY1` is one of its series' letters.
- [ ] The CloudWatch log group receives lines, and `logs/cw_dead_letter.log` in the container is empty.
- [ ] Migration state checked with `db:migrate:status` before any schema change.
- [ ] After a deploy, run the manual checks in `.claude/docs/TESTS.md` → "Manual checks that no test covers", through the gateway.
