# drm-apis — Database

The baseline is in `drm-apis: docs/database.md` (topology and pools) and `drm-apis: docs/data-model.md` (model by model). This file adds the procedure lifecycle, where the SQL lives, and recent tables.

## Technology

PostgreSQL through Sequelize 6. The code uses Postgres features directly: `CALL` procedures, `JSONB`, `ON CONFLICT`, `RETURNING` and PL/pgSQL.

## Recent tables

| Table | Model | Added by | Notes |
|---|---|---|---|
| `fip_audit_dump` | `database/fipAuditDump.model.js` | WBD (`74ed3d8`) | WBD family |
| `open_docket_tm` | `database/openDocketTm.model.js` | WBD (`74ed3d8`) | WBD family |
| `user_column_preferences` | `models/tenant/userColumnPreferences.model.js` | DRM-189 | Unique index `uq_user_screen (personid, screen_name)`; `visible_columns JSONB`; created by `sync()` at boot |
| `closed_docket_pa.date_entered`, `.date_received` | `closedDocketPa.model.js` (`col.tenantStr()`) | DRM-379 | **Optional per tenant**: added with `ALTER TABLE` for tenants that adopt them (`POST_MERGE_STEPS.md` §4) |
| `*_previous` (for example `bibliographic_pa_previous`) | Written by `run_audit` | procedures | Snapshots of the prior batch, used for change-detection rules |

Model columns come from shared builders in `database/modelFactory.js` (`col.*`) and `database/sharedFields.js`. Most string columns are `NOT NULL DEFAULT ''`, so the loader writes `""` rather than `NULL`.

## Stored-procedure lifecycle

`run_audit` (as committed in `rules_procedure.js`) performs, in order:

1. `UPDATE file_upload_status SET isactive='0'` to archive the current batch.
2. `CREATE SEQUENCE serial`, a fixed-name sequence used to number observations.
3. For each rule block, `INSERT INTO audit_result … SELECT … FROM <source tables> WHERE … AND (SELECT status FROM mapped_rules_new WHERE id = N)`. Each rule's SQL is written in the procedure and switched on or off **by rule id**, so rule ids mean the same thing in every tenant DB.
4. Remove duplicates among unarchived rows (`ROW_NUMBER() OVER (PARTITION BY docket_number, application_number, audit_result, issue_description)`), producing `deleted_duplicates`.
5. Flag repeats of observations previously archived as `Corrected` (`is_duplicate='1'`), producing `total_duplicate`.
6. Auto-archive repeats of observations previously `Ignored` / `System Ignored`, producing `total_ignored`.
7. `DROP SEQUENCE serial`; clear the source tables; `COMMIT`.
8. Return four `INOUT` integers, which the cron uses to build the email report.

Operating notes:
- **Run one audit per tenant DB at a time.** The fixed sequence name means a second concurrent run fails; see the `serial` runbook in [operations-and-troubleshooting.md](operations-and-troubleshooting.md).
- The source tables are **transient**: they are empty after each run, so a new audit needs a new upload.
- `cancel_audit()` clears the FIP source tables and every `file_upload_status` row. It leaves the `_ap` (Inprotech), CPI, WBD and digest tables alone.

## Where the procedure source lives

| Source | Contents |
|---|---|
| `drm-apis: rules_procedure.js` | `run_audit`, `cancel_audit` (about 4,260 lines, UAT lineage); used by `setupDRM.js` |
| `drm-apis: database/drmSetup.js` | An older `run_audit` |
| `DB-objects: DRM/` | Production snapshots (`run_audit_trexo-prod*.sql`, `run_audit_tm_trexo-prod*.sql`), UAT variants, and per-tenant subfolders |
| Live tenant DBs | The applied version for each tenant |

Tenants can carry customised procedure versions, so the live DB is the reference for any one tenant. Take a copy (`\sf run_audit`) before changing a procedure.

## Provisioning a tenant (`setupDRM.js`)

`node setupDRM.js <tenantCode>` resolves credentials, looks the tenant up, `sync()`s every model in `database/drmModels.js`, then creates `run_audit` and `cancel_audit`. The CPI and WBD tables (`cpi_pa_rules`, `cpi_tm_rules`, `digest`, `fip_audit_dump`, `open_docket_tm`, `exception_sheet`), the `run_audit_tm*` procedures and the rule rows are provisioned separately.

**Use `setupDRM.js` only for new tenants.** It runs `CREATE OR REPLACE PROCEDURE run_audit`, which would replace an existing tenant's customised logic.

## Schema changes

There is no migration tool. The schema changes through:
- `sync()` at boot, which creates the `users`, `tenant_custom_mappings` and `user_column_preferences` tables where missing, on every tenant DB;
- `setupDRM` for new tenants;
- `ALTER TABLE` statements from release runbooks (for example `POST_MERGE_STEPS.md` §4).

Tenants intentionally differ (tenant-optional columns, custom mappings).

## Transactions

- The loader's `bulkCreate` runs as one INSERT per file. It adds rows to the source table, so upload each file type once per audit batch; cancel the batch to start again.
- The procedure `COMMIT`s at the end. Controllers do not open explicit transactions.

## Connection management

- API: a pool per tenant (max 20, `acquire` 120 s), created at boot for every tenant and kept for the process lifetime.
- Cron: a Sequelize instance (pool max 10) per run, connecting to a database named after the tenant code.
