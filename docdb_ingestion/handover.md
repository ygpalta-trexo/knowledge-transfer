# docdb_ingestion — Handover

As of 2026-09-19 (`main` = `origin/main` = `da98b16`).

## History

| When | What |
|---|---|
| Feb–Mar 2026 | Initial parser and a 10-table relational schema. EPO API download with the `delivery_files` / `ingestion_checkpoints` state. `extra_data` capture. `--start-index` / `--limit` / `--retry-failed` |
| Apr 2026 | Local-to-RDS consolidation (`merge_fast.py`) and schema adjustments |
| **2026-05-18** | **Switch to one flattened `patent_documents` table, LIST-partitioned by country.** Indexes reduced to three; UNLOGGED staging for the upsert; per-worker logging; `monitor_workers.sh` |
| 2026-05-19 → 22 | Configurable batch size; the **front-file runner**; entity handling for deliveries without DTDs |
| 2026-09-08 | EPO API base URL and response-field update (`c4bc63c`) |
| 2026-09-14 | Runner skips completed deliveries, halts on FAILED, emails a run report (`eb191cb`) |
| 2026-09-17 | Database credentials from AWS Secrets Manager, with a refetch on rotation (`da98b16`) |

## Branches

`main` is the only active branch.

## External dependencies

- **EPO BDDS public API:** front-file product `EPO_FRONTFILE_PRODUCT_ID`, and the backfile delivery (code defaults: product 14 / delivery 3071).
- **The shared PostgreSQL DOCDB store**, about 350 GB, also read by epo-bdss-apis.
- AWS Secrets Manager and an EC2 instance role (optional); SMTP for reports.
- **Downstream:** epo-bdss-apis, and through it reference_downloader_apis, the patent-asistant-apis SB-08 Generator and the chat Lambda's EP bibliographic intent.

## Ownership and contacts

| Area | Owner |
|---|---|
| This repo (code and GitHub repository) | |
| The EC2 host that runs ingestion, and its schedule | |
| The DOCDB PostgreSQL store (shared with epo-bdss-apis) | |
| EPO BDDS account, terms and product IDs | |
| Run-report recipients (`EMAIL_RECIPIENT`) | |
