# drm-apis — Handover

As of 2026-09-18.

## Recent delivered work (last ~3 months)

| When | Ticket / commit | What |
|---|---|---|
| 2026-09-17 | DRM-189 `6b7254c` (PR #58) | Remember selected Audit Result columns: `PUT /saveColumnPreferences`, table `user_column_preferences`, `columnPreferences` in `fetchAuditResult/v2` |
| 2026-09-08 | `f422ef0`, tag `v2.0.0` | uat → release production release: Secrets Manager (TP-282), structured logging, MIME validation, 89 commits |
| 2026-09-08 | `07198f8` / `5632a76` / `51ac0fa` | Dockerfile update |
| 2026-07-22 | WBD `74ed3d8` (PR #57) | WBD file family (`FIP Audit Dump`, `OpenDocketTM`), duplicate handling between the WBD and FIP files |
| 2026-07-18 | DRM-379 `91c629e` (PR #56) | Tenant-optional `closed_docket_pa` columns; `returning:false`; date sanitisation |
| 2026-07-14 | TP-282 `16a8f04` (PR #51) | AWS Secrets Manager credential mode; queue processors registered after credentials |
| 2026-07-16 | TP-293 (PRs #53–55) | Dockerfile `apk --no-cache` (`v1.5.1`) |
| 2026-07-13 | PR #50 | Structured child logger and severity-aware error handling |
| 2026-06-22/25 | multiple | Sonar clean-up, `modelFactory` / `sharedFields`, `queueMappings.json`, cron helpers, ESLint, Husky, CI, coverage up to 95% |
| 2026-06-24 | `8953744` | `setAllRulesActiveForTenant` also activates trademark rules (`POST_MERGE_STEPS.md` §9.2) |

## Working practices

- PRs are squash-merged into `main`, then promoted `main` → `uat` → `release`.
- Before deleting a feature branch, compare trees (`git diff origin/main...<branch>`), because squash merges leave branches looking "ahead".
- Take `\sf` copies of a tenant's procedures before any procedure change.

## External dependencies

- `trexo-apigateway`: the `DRM_API_URL` proxy, the `X-User` shape and the `appAccess("DRM")` check.
- `trexo-frontend`: `src/app/pages/DRM/*`.
- `DB-objects/DRM/`: procedures, including per-tenant variants.
- The super-admin DB's `Tenant.sftpConfig` and `DrmMappings` rows.
- Each tenant's SFTP server and folder convention (`/data/drm/drop`, `/data/drm/processed`).
- The scheduler that calls `/cron/*`, and the `scripts` DRM job.
- Outlook SMTP account, CloudWatch, Secrets Manager.

## Ownership and contacts

| Area | Owner |
|---|---|
| drm-apis code | |
| DRM stored procedures (`DB-objects/DRM`) | |
| Production deploys and infrastructure (ECS/Redis/RDS) | |
| Scheduler that calls `/cron/*` | |
| Tech notification mailbox (`TECH_EMAIL`) | |
| Client file-format changes / onboarding new tenants | |
