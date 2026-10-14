# scripts/drm — Deployment and environments

## How it runs

- `python drm/sftp_uploader.py --tenant … [--rule …] [--mark-previous-corrected]`, from a Python environment with `drm/requirements.txt` installed. python-dotenv loads `.env` from the working directory.
- Each run covers one tenant, so scheduling is per tenant (and rule), likely one scheduler entry each. The daily log file (`drm_cron_YYYY-MM-DD.log`) collects all runs for the day.
- The host, its schedule entries and the checked-out branch are managed outside the repo; see the ownership table in [handover.md](handover.md).

## Branches

`main`, `uat`, `release` and `qa`, with changes promoted from `main`. After a promotion, update the checkout on the job host.

## Configuration variable names

Values live in `.env` (git-ignored) or, for the DB login, in Secrets Manager. Start from `drm/.env.sample` and add `TENANT_EMAILS` and `LOGS_DIRECTORY`.

| Name | Used for |
|---|---|
| `USE_SECRET_MANAGER` | Read DB credentials from Secrets Manager |
| `DB_SECRET_NAME`, `AWS_REGION` | The secret (keys `host` / `username` / `password` / `port`, or `DB_*` equivalents) and its region |
| `DB_URL`, `DB_USER`, `DB_PASSWORD`, `DB_PORT` | DB login when Secrets Manager is off. `DB_URL` is a host name |
| `SUPER_ADMIN_DB` | Name of the super-admin database (the `"Tenant"` registry) |
| `DRM_SFTP_DIRECTORY` | Root folder on each tenant's SFTP server (`drop/` below it). **Keep it equal to drm-apis's setting**, so both look at the same folder |
| `BASE_URL` | drm-apis base URL for `/cron/run_audit` |
| `EMAIL_CONFIG_HOST`, `EMAIL_CONFIG_AUTH_USER`, `EMAIL_CONFIG_AUTH_PASSWORD` | SMTP (port 587, STARTTLS) |
| `EMAIL_RECIPIENT` | Internal recipients for errors and notices (comma-separated) |
| `TENANT_EMAILS` | JSON `{ "<tenantCode>": { "email", "person_name" } }`: the tenant contact for missing-file and result emails |
| `LOGS_DIRECTORY` | Log folder |

## Dependencies on other systems

| System | Why |
|---|---|
| Super-admin Postgres DB | Tenant registry, SFTP settings |
| Tenant Postgres DB | Staging tables, `cancel_audit`, `file_upload_status`, audit result tables |
| Tenant SFTP server | Input files |
| drm-apis | Runs the audit procedures, moves SFTP files, returns counts |
| SMTP server | Notifications |
| AWS Secrets Manager (optional) | DB login |
