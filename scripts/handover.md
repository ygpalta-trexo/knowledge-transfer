# scripts — Handover (focus: `drm/`)

As of 2026-09-18 (`main` at `4bf29b1`).

## History of `drm/`

| When | Change |
|---|---|
| 2025-02 | Initial script (`3fec771`); `file_upload_status` active check; empty-file handling; column-mapping fixes (including BibliographicTM `local_date_filed`) |
| 2025-07 | Missing-PAIR-file email to the tenant; unique error count in the audit email; trademark result section; multiple recipients |
| 2025-09 | Email only when a result is returned; all-string dtype (DRM-257) |
| 2025-10 | Excel header and footer detection, openpyxl engine (DRM-261) |
| 2025-12 | Tenant custom mappings passed into the loader; ClosedDocket completion-date comment not loaded (DRM-278) |
| 2026-04-09 | Tenant name in the audit email; `--mark-previous-corrected` (#50) |
| 2026-06-05 | Newest duplicate SFTP file wins (`303e0df`, DRM-374) |
| 2026-07-18 | DRM-379: more ClosedDocket date columns parsed as dates |
| 2026-07-21 | WBD mappings and rule (`87b2dce`, #76) |
| 2026-09-11 | Secrets Manager for the DB login (`58243ad`, #84) |

Earlier work also added client-specific handling (DRM-263 / 297 / 298 / 305 / 331), `cancel_audit` before upload (`f18b9c9`) and CPI scheduling.

## External dependencies

drm-apis (`/cron/run_audit`, the DRM stored procedures, the SFTP move to `processed/`, the tech report email); the super-admin DB and the tenant DBs; each tenant's SFTP server; the SMTP relay; AWS Secrets Manager.

## Ownership

| Area | Owner |
|---|---|
| `drm/` script | |
| Host and scheduler that run it | |
| `TENANT_EMAILS` (client contacts) | |
| DRM mappings (both copies) | |
