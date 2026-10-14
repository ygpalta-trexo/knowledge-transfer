# scripts — Knowledge Transfer (focus: `drm/`)

Describes the code as of 2026-09-18 (`main` at `4bf29b1`). The repo lives at `~/repos/scripts`, outside the `trexo-global` workspace. Paths below are relative to the repo root.

## What this repo is

A collection of standalone operational scripts, mostly Python. It is not a service: each folder is independent.

| Folder | Purpose | Covered here |
|---|---|---|
| `drm/` | DRM ingestion and audit runner: tenant SFTP → tenant DB → drm-apis `/cron/run_audit` → summary emails | In depth |
| `docgen/` | Docgen helpers, including a tenant usage report (`docgen/common/usage-report/README.md`) | Named only |
| `Patents/` | Includes a US patent data ingestion pipeline with its own tests (`Patents/us-patent-data-ingestion-pipeline/README.md`) | Named only |
| `soc2/` | SOC 2 evidence tooling: Jira and GitHub extraction (`soc2/jira/README.md`), AWS IAM reports (`soc2/aws/aws-users/README.md`) | Named only |
| `misc/` | For example, a database roles and users manager (`misc/db-roles-users/README.md`) | Named only |
| `ip-assistant/`, `trademarks/`, `compare_tool/`, `mail-attachment-downloader/` | Other utilities | Not covered |

## `drm/` in one paragraph

`drm/sftp_uploader.py` is a command-line job that handles one tenant (and optionally one rule) per run. It reads the tenant's SFTP settings from the super-admin DB and picks the newest file of each required type from the tenant's SFTP `drop/` folder. It then calls `cancel_audit()` and loads each file into the tenant DB's DRM staging tables, the same tables the drm-apis upload path fills. Next it calls drm-apis `POST /cron/run_audit`, which runs the audit procedures, moves the SFTP files to `processed/` and returns the counts. Finally it emails the tenant contact a summary of the observations.

### How it relates to drm-apis

- It is an ingestion path that loads SFTP files directly and uses drm-apis only for the audit step (`/cron/run_audit`). It likely took over from the `automation/` Python scripts that used to live in drm-apis (drm-apis `POST_MERGE_STEPS.md` §2).
- It keeps its own copy of the DRM column mappings (`drm/file_processor.py` `queueMappings`). When a file type's mapping changes, change it here and in drm-apis `queue/queueMappings.json`. See [architecture.md](architecture.md#column-mappings).

## Stack

Python 3. Pinned in `drm/requirements.txt`: pandas and openpyxl (Excel), `lxml` (XML), paramiko (SFTP), psycopg2-binary and SQLAlchemy (Postgres), requests, python-dotenv, boto3 (Secrets Manager).

## Run locally

```
cd drm && python -m venv .venv && . .venv/bin/activate && pip install -r requirements.txt
cp .env.sample .env        # then set the names listed in deployment-and-environments.md
python sftp_uploader.py --tenant <tenantCode> [--rule <rule>] [--mark-previous-corrected]
```

Every run is live: it reads a real SFTP folder, reloads the tenant's DRM staging data and triggers drm-apis, which runs the audit, moves the files in `drop/` to `processed/` and sends emails. Point every setting at a non-production tenant, SFTP folder, drm-apis and mailbox when testing.

## Related documentation

- `drm/.env.sample` lists the configuration names; the full list is in [deployment-and-environments.md](deployment-and-environments.md).
- For what the audit rules check, the procedures and the `/cron` routes, see the drm-apis package: [../drm-apis/product-and-features.md](../drm-apis/product-and-features.md), [../drm-apis/api-and-integrations.md](../drm-apis/api-and-integrations.md), [../drm-apis/database.md](../drm-apis/database.md).

## This package

| File | Contents |
|---|---|
| [architecture.md](architecture.md) | End-to-end flow (sequence diagram), rules and required files, file recognition, loading, the division of work with drm-apis |
| [operations-and-troubleshooting.md](operations-and-troubleshooting.md) | Emails (who gets what), logs, common situations, manual-run checklist |
| [deployment-and-environments.md](deployment-and-environments.md) | How it runs, branches, configuration names, dependencies |
| [auth-security.md](auth-security.md) | Credentials and the data it handles |
| [handover.md](handover.md) | History, dependencies, ownership |

## Suggested reading order

1. This file.
2. [architecture.md](architecture.md).
3. [operations-and-troubleshooting.md](operations-and-troubleshooting.md).
4. [../drm-apis/product-and-features.md](../drm-apis/product-and-features.md) for the rule families (FIP, CPI, Digest, Inprotech, WBD).
5. [handover.md](handover.md).
