# docdb_ingestion — Deployment and environments

The operator procedures (venv setup, workers, `nohup`, monitoring, the front-file runner, scheduling, retry, Secrets Manager) are in `docdb_ingestion: docs/server_run_guide.md`.

## Where it runs

- A run is a git clone, a venv and `pip install -r requirements.txt`, then Python run in place.
- The server guide targets an EC2 host with an instance role and describes scheduling the weekly front-file run there.
- The code has no environment concept: whatever `.env` (or the secret) points to is the target. The DOCDB store is shared with epo-bdss-apis, so **treat any run with production credentials as a production write.**

## Configuration variables (names only)

All are read from the environment after `load_dotenv()`. `get_dsn_from_env()` calls `load_dotenv(override=True)`, so **values in `.env` replace shell-exported ones**. Keep per-worker settings out of `.env` and pass them as CLI flags (`--batch-size`, `--worker-name`, `--log-file`) or on the command line.

| Group | Variables | Notes |
|---|---|---|
| Database (direct) | `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `DATABASE_URL` | `DATABASE_URL` is used when `POSTGRES_DB` is unset |
| Database (secret) | `USE_SECRET_MANAGER` (`true` enables it), `DB_SECRET_NAME`, `AWS_REGION`; `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` only off EC2 | Secret JSON keys: `host`/`username`/`password`/`port`, or `DB_HOST`/`DB_USER`/`DB_PASSWORD`/`DB_PORT`. `POSTGRES_DB` names the database. IAM needs `secretsmanager:GetSecretValue`, plus `kms:Decrypt` for a customer-managed key |
| EPO | `EPO_API_BASE_URL`, `EPO_PRODUCT_ID` (default 14), `EPO_DELIVERY_ID` (default 3071), `EPO_FRONTFILE_PRODUCT_ID` (template: 3), `BACKFILE_TIME` (ISO-8601 cutoff) | 14 / 3071 are the backfile delivery defaults |
| Runtime | `EPO_TEMP_DIR` (default `./tmp_downloads`), `DOCDB_BATCH_SIZE` (default 1000), `PIPELINE_LOG_FILE`, `PIPELINE_WORKER_NAME` | CLI flags override the environment |
| Run report | `EMAIL_RECIPIENT`, `EMAIL_CONFIG_HOST`, `EMAIL_CONFIG_PORT`, `EMAIL_CONFIG_AUTH_USER`, `EMAIL_CONFIG_AUTH_PASSWORD` | The same names as the `scripts` DRM job |
| Earlier-schema utility | `RDS_HOST`, `RDS_PORT`, `RDS_DB`, `RDS_USER`, `RDS_PASSWORD` | `merge_fast.py` only; run it with `USE_SECRET_MANAGER=false` |

Share the working `.env` only through a secrets channel.

## Infrastructure dependencies

- **PostgreSQL** reachable from the runner, with room for the data set (about 350 GB) plus WAL and autovacuum headroom.
- **Local disk** for `EPO_TEMP_DIR` (one outer ZIP, then its extracted inner ZIPs) and the **system temp directory** (each inner XML plus a patched copy during parsing).
- **Outbound HTTPS** to the EPO BDDS public API.
- **SMTP** for reports.
- **AWS Secrets Manager** and an instance role, when enabled.

## Deploying a change

1. Merge to `main` and push.
2. On the server: `git pull`, `pip install -r requirements.txt`, then `python -c "import docdb_ingestion"` and `python -m docdb_ingestion.pipeline --help`.
3. Before the next scheduled run: `python frontfile_runner.py --mode catchup --dry-run`.
4. Schema changes: see [database.md](database.md#schema-changes).
