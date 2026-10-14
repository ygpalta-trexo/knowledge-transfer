# kot-apis — Handover

As of 2026-09-18 (`release` = `main` = `uat` at `7daa826`).

## Recent delivered work (last ~3 months)

| Date | Commit | Ticket | What |
|---|---|---|---|
| 2026-09-11 | `7daa826` | KOT-129 | Foreign-associate master refresh: new CSV and `seedForeignAssociates.js --prune` |
| 2026-09-02 | `9c2059f` | KOT-103 | Validation result file named after the uploaded file (RFC 6266 headers) |
| 2026-08-31 | `63f25ec` | TP-310 (#33) | AWS Secrets Manager for DB credentials (`beforeConnect` hook) |
| 2026-08-20 | `8e8d660` | KOT-264 (#32) | Joi request validation on 21 of 29 routes |
| 2026-08-13 | `112f983` | KOT-30 (#31) | Report xlsx honours the grid's column picker |
| 2026-08-11 | `6fbc371` | KOT-123 (#30) | Client Details: corresponding section update |
| 2026-08-10 | `a1f1f7c` | KOT-101 (#29) | Inventor bulk upload |
| 2026-07-31 | `f64e14d` | KOT-117 (#27) | VIEW_ONLY role; read-only export block; inventor import script |
| 2026-07-31 | `b413c26` | KOT-115 (#28) | See PR |
| 2026-07-30 | `a8ca429` | KOT-116 (#25) | Related Parties (formerly Licensed Families) |
| 2026-07-17 | `1e017c4` | (#24) | Bulk-update payload limit (413) and CloudWatch shipping |
| 2026-07-07 | `248e858` | KOT-84 (#23) | Sort correspondence info |
| 2026-06-23 | `e089a26` | — | Lists return all rows (profiles, audit logs) |

## Working practices

- PRs are squash-merged; compare trees (`git diff origin/main...<branch>`) before deleting a feature branch.
- For design context on any of the above, see `DECISIONS.md` and `traces/` in the KoT working notes.

## External dependencies

- `trexo-apigateway` (`KOT_API_URL`): identity headers.
- `trexo-frontend` `pages/Kot/*` and `pages/tenants/UserCard.js` (access grants).
- Document-processor service (`DOCUMENT_PROCESSOR_URL`).
- AWS S3 (DOCX template), CloudWatch, Secrets Manager.
- Super-admin DB and per-tenant DBs; the platform-owned `public.users`.

## Ownership

| Area | Owner |
|---|---|
| kot-apis code | |
| KoT product decisions (RBAC matrix, retention, who grants access) | |
| Foreign-associate list hand-offs | |
| Document processor | |
| Deploys / infrastructure | |
