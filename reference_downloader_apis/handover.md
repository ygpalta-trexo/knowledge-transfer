# reference_downloader_apis — Handover

As of 2026-09-18.

## Recent delivered work (last ~3 months)

| When | Ticket / commit | What |
|---|---|---|
| 2026-09-08 | `b1b4507` (#139) | Upgrade eslint, nodemailer, pdf-parse (v2), pdf-to-png-converter |
| 2026-09-07 | `a49b33b` | Text extraction adjusted for pdf-parse v2 (page joiner excluded) |
| 2026-09-03 | TP-163 `24968a0` (#147) | SonarQube maintainability refactor |
| 2026-09-01 | TP-271 `569b330` (#145) | Joi HTTP request validators on every route |
| 2026-09-01 | `e1cf347` | Reference normalisation before queueing; metadata match summary in logs |
| 2026-08-26 | `c0f91e7` / TAI-1129 | Record why a reference is invalid (`invalidReason`, migration `20260825…`) |
| 2026-08-20 | `881cd59`, `6c93dfa` | CloudWatch transport resilience; flush timer `unref` |
| 2026-07-31 | PA-68 `8dbc601` | Production deployment |
| 2026-07-30 | #144 | Re-processing of manually completed references |
| 2026-07-14 | #143 | AWS Secrets Manager for DB credentials |
| 2026-07-08/09 | #141 | Structured request-scoped logging |
| 2026-07-07 | TAI-1109 (#140) | PDFs normalised to US Letter; page-size columns (migration `20260702…`); backfill script |
| 2026-07-03/10 | TAI-1119, TAI-959 | Metadata source attribution; blank metadata cells clear values |
| 2026-06-30 | TAI-519 (#136) | Messaging for PDFs over 1000 pages |
| 2026-06-25 | multiple | Test coverage to ~99%, Sonar fixes, API docs (`docs/*.md`) |

## Working practices

- Run migrations before deploying new code; the frontend's invalid-reasons list ships together with backend changes to `utils/invalidReasons.js`.
- PRs are squash-merged; compare trees before deleting a feature branch.

## External dependencies

- `pa_lambda_functions` LexChatbot (`REFERENCE_BASE_URL`): creates the requests.
- `pa_lambda_functions` FlattenPDF (`FLATTEN_PDF_URL`).
- `patent-asistant-apis`: `/postEmail`, `/usFilingPostCollab`, the PCT request creator, and the shared `AppGenInput` table.
- `trexo-apigateway` (`REFERENCE_DOWNLOADER_API_URL`, `COLLABORATE` app) and `trexo-frontend` Collaboration Space pages.
- EPO BDSS service (`EPO_BDSS_URL`), patent-cli service (`PATENT_CLI_URL`).
- Wasabi (3 buckets), AWS S3 (ZIP bucket), AWS SQS (3 queues), SMTP, CloudWatch, Secrets Manager.

## Ownership and contacts

| Area | Owner |
|---|---|
| reference_downloader_apis code | |
| Collaboration Space team (queue operators; recipients of `COLLAB_SPACE_TEAM_EMAIL` and `COLLAB_SPACE_APPGEN_TEAM_EMAIL`) | |
| IP Assistant (`patent-asistant-apis`) callback contract | |
| FlattenPDF Lambda | |
| patent-cli and EPO BDSS services | |
| AWS infrastructure (SQS, S3, ECS), Wasabi account | |
| Scheduler that calls `/sendReminderEmails` | |
