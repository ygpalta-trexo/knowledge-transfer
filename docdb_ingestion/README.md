# docdb_ingestion — Knowledge Transfer

**Start here.** Describes the code as of 2026-09-19 (`main` at `da98b16`).

## What it is

A Python batch pipeline that downloads the EPO's **DOCDB bulk bibliographic data** (weekly XML deliveries from the EPO Bulk Data Distribution Service, "BDDS") and loads it into PostgreSQL. It writes one large table, `patent_documents`, LIST-partitioned by publication `country` (`docdb_ingestion/database.py`).

That table is what **epo-bdss-apis** serves; its bulk bibliographic lookups feed reference_downloader_apis, the patent-asistant-apis SB-08 Generator and the chat Lambda's EP bibliographic intent (see [../epo-bdss-apis/README.md](../epo-bdss-apis/README.md)).

- **Location:** `~/repos/bdds/docdb_ingestion`, outside `~/repos/trexo-global/`.
- **Repository:** GitHub `bulkdata-ingestion-pipeline`.
- **History:** 25 commits between 2026-02-25 and 2026-09-17.

**Why it exists:** it replaces per-reference calls to the rate-limited EPO APIs with a local copy of the whole data set.

**Users:** no human users and no API. It is an operator-run batch job; the consumers are downstream services that read the table.

## Two ways data gets in

| Path | Entry point | What it loads | Run style |
|---|---|---|---|
| **Backfile** (the one-time historical snapshot) | `python -m docdb_ingestion.pipeline sync` then `run` | One EPO delivery (`EPO_PRODUCT_ID` / `EPO_DELIVERY_ID`), split across workers with `--start-index` / `--limit` | Long `nohup` runs, several workers |
| **Front-file** (weekly updates after the backfile cutoff) | `python frontfile_runner.py --mode catchup \| latest` | Every delivery of product `EPO_FRONTFILE_PRODUCT_ID` published after `BACKFILE_TIME`, week by week, in the order DeleteRekey → CreateDelete → Amend | Weekly; emails a run report |

Both paths go through the same `PipelineOrchestrator`: download → extract → stream-parse → staged upsert → checkpoint. See [architecture.md](architecture.md).

## Stack

- Python 3.10 with `lxml` iterparse, `psycopg` 3 (COPY and upsert), `pydantic` v2 models and `requests` (`requirements.txt`).
- PostgreSQL; the table is about 350 GB (2026-09-09).
- Optional AWS Secrets Manager for database credentials (`boto3`); SMTP for run reports.

## Run locally

See `docdb_ingestion: README.md` ("Setup & Installation") and `docdb_ingestion: docs/server_run_guide.md`. In short:

1. `python -m venv .venv && pip install -r requirements.txt`
2. `cp .env.example .env`, then set the `POSTGRES_*` values for a local database.
3. `python setup_db.py` to create the tables and partitions.
4. Parse without writing: `python process_folder.py <folder of zips> --dry-run --limit 1`.
5. Plan front-file work without writing: `python frontfile_runner.py --mode catchup --dry-run` (needs the schema from step 3).

**Use `python reset_db.py` only against a local database.** It drops `patent_documents`, `ingestion_checkpoints` and `delivery_files` in whatever database `.env` resolves to, without a prompt.

## Existing repo docs

This package reflects the code as of the date above; where the repo docs differ, follow this package.

| Doc | Covers |
|---|---|
| `README.md` | Features, setup, pipeline commands, batch tuning, `process_folder.py`, Excel export |
| `docs/server_run_guide.md` | **The main operator guide:** running on EC2, workers, temp dirs, logs, `monitor_workers.sh`, the front-file runner, scheduling, retry, Secrets Manager |
| `docs/single_table_partitioned_design.md` | Rationale for one partitioned table, JSONB shapes, partition routing, query patterns (for the current index set, see [database.md](database.md)) |
| `docs/technical_documentation.md` | The original design and utility scripts (earlier schema) |
| `EPO Data Extraction and Database Integration.md` | Background on the DOCDB XML format: `doc-id`, statuses C/D/A/CV/DV, re-key ordering |

## This package

| File | Read it for |
|---|---|
| [architecture.md](architecture.md) | Components, the state machine, parsing, the staged merge, ordering rules, external systems |
| [database.md](database.md) | Tables, indexes, columns and merge semantics |
| [deployment-and-environments.md](deployment-and-environments.md) | Where it runs, configuration names, deploying a change |
| [operations-and-troubleshooting.md](operations-and-troubleshooting.md) | Freshness checks, common situations, re-processing, a weekly checklist |
| [testing.md](testing.md) | The test suite and how to run it |
| [handover.md](handover.md) | History, dependencies, ownership |

## Suggested reading order

1. This README, then `docdb_ingestion: docs/server_run_guide.md`.
2. [architecture.md](architecture.md) and [database.md](database.md).
3. [operations-and-troubleshooting.md](operations-and-troubleshooting.md).
4. [handover.md](handover.md).
