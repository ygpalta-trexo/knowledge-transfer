# drm-apis — Product and features

## Business problem

IP law firms run docketing systems (patent and trademark deadlines, filings, correspondence). Drift between those systems and official sources creates risk: missed deadlines, wrong status, missing entries. DRM audits a firm's docket exports against a rule set and returns a list of **observations** (errors) to review.

## Users and roles

| Actor | What they do |
|---|---|
| Tenant user (law firm) | Uploads files, toggles rules, runs or cancels an audit, reviews results, archives them, manages activity codes |
| Scheduler | Pulls files from the tenant's SFTP drop, runs the audit, switches rules, purges old results (`/cron/*` routes) |
| Tech team | Receives cron emails (failures, audit reports, rule switches); watches Bull Board |

Within drm-apis every user whose tenant has the `DRM` app has the same capabilities; app access is checked at the gateway.

## Source file families

A tenant's active upload set must belong to **one family at a time**; the scheduled audit checks this before running (`cron/runAuditForAllTenants.js` `validateFileTypes`).

| Family | Files (queue keys) | Audit procedures | Notes |
|---|---|---|---|
| FIP (standard) | `BibliographicPA`, `ClosedDocketPA`, `OpenDocketPA`, `PairPA`, `IndiaMattersPA`, `WorkingStmtPA`, `BibliographicTM`, `ClosedDocketTM` | `run_audit`, `run_audit_tm` | `PairPA` also arrives as XML `Outgoing_Correspondence_List` |
| CPI (trademark) | `CPI_Create_ActionTM`, `CPI_Last_ActionTM`, `CPI_BibliographyTM`, `User Name` | `run_audit_tm_cpi` (cron); the UI uses the `run_audit_cpi` / `run_audit_tm_cpi` pair | Any active `CPI_*` file switches the rule tables to `cpi_pa_rules` / `cpi_tm_rules` |
| Digest | `90dayOutput`, `PairPA` | `run_audit` | |
| Inprotech ("TREXO-*") | `TREXO-DRM-OPENDOCKETPA`, `TREXO-DRM-BIBLIOGRAPHICPA`, `TREXO-WORKINGSTMTPA`, `TREXO-DRMINDIAMATTERSPA`, `TREXO-DRM-BIBLIOGRAPHICTM` | `run_audit`, `run_audit_tm` | Loads the `*_ap` tables |
| WBD | `FIP Audit Dump`, `OpenDocketTM`, plus `BibliographicTM`, `ClosedDocketTM` | `run_audit`, `run_audit_tm` | Added July 2026 (PR #57) |

## Core workflow: upload → audit → review

1. **Upload.** The user uploads a file (UI `POST /uploadFile?fileName=<Type>.xlsx`), or the scheduler pulls it from SFTP (`/cron/upload`). The file name (text before the first `.`) selects the queue and the target table.
2. **Status tracking.** One `file_upload_status` row exists per file type: `Pending` → `Pass` / `Fail`, with `isactive='1'` while it belongs to the current batch. The UI polls `GET /getActiveFilesUploaded`.
3. **Duplicate guard (UI).** If a file type is already `Pass` and active, the UI upload returns "already uploaded" and does nothing. SFTP ingestion replaces the status row.
4. **Rule selection.** `GET /fetchMappedRules` lists rules whose `input_file_name` matches an active upload. The user toggles them with `POST /updateMapRule`.
5. **Run audit.** The UI calls `GET /runActiveRules?category=patent|trademark|all`, or the scheduler calls `/cron/run_audit`. Either way a stored procedure runs and:
   - archives the active `file_upload_status` rows (`isactive='0'`);
   - inserts observations into `audit_result` for each enabled rule;
   - de-duplicates, flags "corrected" repeats, and auto-archives observations the user already ignored;
   - clears the source tables, ready for the next batch.
6. **Review.** `GET /fetchAuditResult/v2` returns the patent and trademark results, the tenant's column mapping (`DrmMappings`) and the user's saved column preferences (DRM-189).
7. **Act.** `POST /updateAuditResult` sets `is_managed`, `action_taken` and `processed_by`, and archives the row (`is_archived='1'`). `POST /deleteFromArchives` deletes rows permanently.
8. **Cancel.** `GET /cancelActiveRules` calls `cancel_audit()`, which clears the FIP source tables and all `file_upload_status` rows for the tenant. Tell the user before cancelling on their behalf.

## Business rules in code

| Rule | Where |
|---|---|
| Only one file family per audit batch | `validateFileTypes` |
| The scheduled audit waits for `Pending` files: up to 5 attempts, 15 minutes apart (in-process timer) | `checkFileStatuses` |
| A `Fail` file stops the scheduled audit and emails the tech team | `checkFileStatuses` |
| Missing files in a family do not stop the audit; they are listed as "Missing Files" in the report | `getMissingFiles` |
| After a scheduled run, files in the SFTP `drop/` folder are moved to `processed/` with a timestamp suffix | `moveInSftp` |
| Observations with a `batch_date` older than 90 days are deleted by the purge job | `cron/deleteResults.js` |
| Weekly PAIR-only audit for one configured production tenant: Tuesday 11:15 IST switch to rule id 16 only, 11:30 audit, 11:55 re-enable all rules | `cron/index.js`, `cron/switchRules.js` |
| Rule id `16` is the PAIR rule | `switchPAIRRuleForTenant` |
| Excel dates normalised to `MM/DD/YYYY`; in date columns, `"true"`/`"false"` artefacts are stored as `""` | `fileUpload.controller.js`, `queueProcessor.js` |
| Values truncated to 255 characters on insert | `truncateString` |
| Trademark `class_code` normalised to a comma list; Inprotech TM descriptions split into class codes | `parseXlsxRow` |
| Tenant-specific column layouts override the defaults via `tenant_custom_mappings` | `getMappingAndTable` |
| Optional columns (for example `closed_docket_pa.date_entered` / `date_received`, DRM-379) exist only for tenants that adopt them | `POST_MERGE_STEPS.md` §4, `queueProcessor.js` |
| Report counts: new errors = added − deleted duplicates − ignored; unique = new − corrected duplicates | `generateAuditMessage` |

## Features

### File upload (UI)
- **Flow:** `POST /uploadFile` → Multer (memory) → MIME and XLSX structure check → HTTP response → parse and `queue.add` → worker → `bulkCreate` → status `Pass`/`Fail`.
- **Frontend:** `pages/DRM/DataRiskManager.js` and related pages.
- **Persistence:** `file_upload_status` plus one source table per file type.
- The HTTP response confirms receipt; the UI then polls the status row for the processing result.
- The file name must match a queue key exactly (see [jobs-and-queues.md](jobs-and-queues.md)).

### SFTP ingestion (scheduled)
`/cron/upload?tenant=` connects with the tenant's `sftpConfig` (stored in the super-admin `Tenant` row), lists `/data/drm/drop`, maps each file name to a canonical type (`cron/cronHelpers.js` `FILE_NAME_MAP`), downloads it to `/tmp`, and passes it to the same `uploadHandler` the UI uses. The `scripts` DRM job is a second ingestion path; see [../scripts/README.md](../scripts/README.md).

### Rules management
Four rule tables: `mapped_rules_new` (patent), `tm_rules`, `cpi_pa_rules`, `cpi_tm_rules`. Each rule is switched on or off by id through its `status` column; the rule SQL itself lives in the audit procedures.

### Activity codes
CRUD on `pair_code_mapping` (PAIR code → activity code and name), used by the PAIR rules.

### Column preferences (DRM-189)
`PUT /saveColumnPreferences` upserts `user_column_preferences(personid, screen_name)`. `GET /fetchAuditResult/v2` returns them as `columnPreferences`. `personid` comes from the gateway-built `X-User`.

## Critical paths

1. **Scheduled SFTP → audit → email**: the unattended path for SFTP tenants. The tech mailbox is where its results and failures appear.
2. **Stored procedures**: most business logic sits in per-tenant PL/pgSQL (see [database.md](database.md)).
3. **`cancel_audit` and the purge job**: both delete data permanently.
