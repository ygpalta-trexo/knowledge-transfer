# docdb_ingestion — Database

PostgreSQL. `DatabaseManager.init_schema()` creates the schema with `CREATE … IF NOT EXISTS` on each `connect()`, unless the caller passes `init_schema=False` (`docdb_ingestion/database.py`).

Row shape and JSONB layouts are in `docdb_ingestion: docs/single_table_partitioned_design.md`. This file adds the table and index set and the merge semantics.

## Which database

`get_dsn_from_env()` builds the DSN from `POSTGRES_*`, or from the Secrets Manager secret plus `POSTGRES_DB`, and uses `DATABASE_URL` when `POSTGRES_DB` is unset. The template names the database `bulk-data`; the code's fallback is `docdb`. epo-bdss-apis reads `patent_documents` from its own configured `DB_NAME`.

## Tables

| Table | Key | Purpose |
|---|---|---|
| `patent_documents` | PK `(country, pub_doc_id)` | One row per publication. `PARTITION BY LIST (country)` |
| `patent_documents_{us,ep,wo,cn,jp,kr,de,gb,fr}` | as parent | 9 dedicated partitions |
| `patent_documents_default` | as parent | Every other country |
| `patent_documents_stage` | index on `stage_key` | **UNLOGGED** staging for the batch merge; rows are removed within the same transaction |
| `delivery_files` | `file_id` (the EPO item id) | Outer files per `(product_id, delivery_id)`, with `status` and `error_message` (the traceback) |
| `ingestion_checkpoints` | `filename` (the inner ZIP basename) | `STARTED` / `COMPLETED`, parent `main_zip_id`, `first_doc_number` / `last_doc_number` |
| `frontfile_delivery_audit` | `id` | One row per front-file delivery run: week, mode, status, counts, error. The backfile CLI writes it too, with empty week and mode |

**Size** (2026-09-09): about 350 GB in total — CN 149 GB / 56M rows, US 64 GB / 22M rows, JP 63 GB, DEFAULT 50 GB.

## `patent_documents` columns

- **Identity:**
  - `country`, `pub_doc_id` (the EPO `doc-id`, a stable surrogate; if absent, `country+doc_number+kind`);
  - `doc_number` (as EPO delivers it, so US pre-grant numbers appear at two widths; epo-bdss-apis handles both);
  - `kind_code`, `extended_kind`.
- **Publication:** `date_publ`, `family_id`, `is_representative`, `is_grant`, `originating_office`, `date_added_docdb`, `date_last_exchange`.
- **Application:** `app_doc_id`, `app_country`, `app_number`, `app_kind_code`, `app_date`. Documents without a `docdb` application reference get `app_doc_id = 'UNKNOWN_<pub_doc_id>'`, `app_country = 'XX'` and `app_number = 'UNKNOWN'`.
- **`source_status`:** the last document status applied (`C`, `A`, or `SKIP` for void stubs).
- **JSONB** (each with a CHECK on its type):
  - `parties` `{applicants, inventors, others}`;
  - `priorities`, `classifications`, `citations` (passages nested), `texts` (titles and abstracts);
  - `designations`, `availability_dates`;
  - `app_extra_data`, `pub_extra_data`.
- `created_at`, `updated_at`.

## Indexes

Created by the code:
- the PK `(country, pub_doc_id)`;
- `idx_patent_documents_country_doc_number (country, doc_number)`;
- `idx_patent_documents_country_app_number (app_country, app_number)`.

The earlier B-tree indexes (`app_doc_id`, `family_id`, `date_publ`, `is_grant`) and GIN indexes on the JSONB columns were removed on 2026-05-18 (`2da2d5b`). Check a database's actual set with `\di patent_documents*`.

Query by `(country, doc_number)` (one partition, indexed) or `(app_country, app_number)` (indexed, across partitions). Queries on other columns read the full table.

## Write semantics

See [architecture.md](architecture.md#writing-databasebulk_upsert_safe) for the steps. What they mean for the data:

- **Amend = full row replacement**, matching EPO's "wipe and load" rule for `status="A"`.
- **Within a batch, the last record for a key wins** (`DISTINCT ON … stage_seq DESC`): `D` then `C` for the same key upserts; `C` then `D` deletes.
- **Deletes** remove `(country, pub_doc_id)`.
- **Void (CV/DV) records** are stored as stub rows with `pub_doc_id = 'VOID_<cc><number><kind>'` and `source_status = 'SKIP'`, without dates or parties.
- The audit's `docs_deleted` counts delete records processed.

## Connections and transactions

- One `psycopg` connection per `DatabaseManager`. The front-file runner opens one per delivery, and the constructor runs the schema DDL each time.
- The merge commits once per batch; status and checkpoint updates commit individually.
- The Secrets Manager path retries a connection once after a password-authentication failure (secret rotation).

## Schema changes

`init_schema` adds missing objects only. Apply column changes and index changes on existing partitions by hand; for indexes, use `CREATE INDEX CONCURRENTLY` per partition, then attach to the partitioned index.
