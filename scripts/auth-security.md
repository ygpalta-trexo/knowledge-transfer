# scripts/drm — Auth and security

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| DB access | One DB login for the super-admin DB and the tenant DBs (read the tenant registry and SFTP settings, write the staging tables, call procedures, update audit results), from Secrets Manager or environment variables | `sftp_uploader.py`, `aws_secrets.py` |
| SFTP access | Per-tenant credentials from `"Tenant"."sftpConfig"` in the super-admin DB | `sftp_uploader.py` |
| drm-apis call | Internal `POST /cron/run_audit` call to the drm-apis base URL | `run_audit.py` |
| Email | SMTP with STARTTLS on port 587 and a login | `email_server.py` |
| Secrets in the repo | `.env` and `logs/` are git-ignored (`*.env`, `*logs/`); `drm/.env.sample` holds placeholders only | `.gitignore` |

## Data handled

- **Client docket exports** (patent and trademark matters), downloaded from the tenant SFTP to `/tmp/<epoch>_<tenant>_<file>` on the host.
- **Client contact details** (name and email per tenant), held in the `TENANT_EMAILS` setting.
- **Logs** under `LOGS_DIRECTORY`, which contain tenant codes, file names, the tenant's custom mappings and the drm-apis response.

Treat the host's `/tmp`, the log folder and the `.env` file as client-confidential. Share the `.env` only through a secrets channel.
