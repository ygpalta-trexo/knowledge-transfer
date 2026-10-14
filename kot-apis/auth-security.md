# kot-apis — Auth and security

`kot-apis: docs/System_Architecture_Review.md` §10 describes the trust model (header trust, CORS, two-tier RBAC, privacy, DB-backed permissions seeded from the matrix). The working notes' `CONSTRAINTS.md` gives the rules every new endpoint follows.

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| Authentication | At the gateway: JWT HS512 plus a Redis token cache. The gateway forwards `X-User` (claims JSON) and `X-Tenant-Name` (Origin subdomain) | `trexo-apigateway: middleware/authentication.js` |
| Header checks | `X-User` must be a JSON object; `X-Tenant-Name` is required (KOT-264) | `src/app.js` |
| Tenant isolation | DB per tenant, selected by `X-Tenant-Name`; the gateway ties the JWT tenant to the Origin tenant | `src/app.js`, gateway |
| Authorisation | Per controller: `accessService.getUserAccess` → 403 unless `hasKotAccess \|\| isAdministrator`; section `can_view`/`can_edit` from `role_section_permissions`; the VIEW_ONLY `readOnlyRoles` block; the private-profile assignment rule | `services/access.service.js`, controllers |
| Validation | Joi on 21 of 29 routes; blank strings allowed by design (`validators/joi.js`) | README *Request Validation* |
| File upload | `.xlsx` only, 10 MB, 1 file, in memory; per-field validation | `middleware/xlsxUpload.js` |
| Download headers | `Content-Disposition` emits both RFC 6266 forms; user-supplied file names are sanitised | `DECISIONS.md` 2026-09-02 |
| Audit | Field-level `audit_logs` in the same transaction as the change | services |
| Secrets | DB credentials from env or Secrets Manager; S3 and CloudWatch clients use `AWS_ACCESS_KEY_ID` / `AWS_ACCESS_KEY_SECRET` | `config/` |
| Container | Non-root; `.env*` excluded by `.dockerignore` | `Dockerfile` |

## Permission levels used by the routes

- **Gate:** the caller has KoT access (or is an administrator).
- **Read-only block:** VIEW_ONLY users cannot mutate, export or download.
- **Section rights:** edits need `can_edit` on the section being changed.
- **Admin:** privacy changes and reactivation.
- **Privacy:** a private profile is visible to admins and assigned users only.

## Checklist for new endpoints

From `CONSTRAINTS.md` and `TESTS.md`. New endpoints must:
- call `getUserAccess` and return 403 without access;
- honour `readOnlyRoles` for mutate, export and download;
- read tenant data only through `req.models` / `req.sequelize`, never `global.tenantConnections` directly.

## Handling data

Client-profile content and the foreign-associate list (`foreign associates/fa table.csv`) contain confidential contact data. Keep client hand-off spreadsheets out of git.
