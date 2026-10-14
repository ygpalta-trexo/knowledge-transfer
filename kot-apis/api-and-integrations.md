# kot-apis — API and integrations

Base docs:
- `kot-apis: docs/API_Reference.md`: request and response examples, and the validation section;
- `kot-apis: README.md` → *Request Validation*;
- the Postman collection.

This file gives the route inventory as the code mounts it.

## How it is reached

Browser → gateway `/tenantAPI/kot/<route>` → `KOT_API_URL/<route>`. The gateway:
- authenticates the JWT;
- derives `X-Tenant-Name` from the Origin subdomain;
- forwards `X-User` (JSON of the claims, including `personid`).

Inside the service, routes are mounted without a prefix (`/client-profiles`, etc.).

## Route inventory

| Method | Route | Purpose | Joi |
|---|---|---|---|
| GET | `/health` | Liveness | — |
| GET | `/me/access` (also `/admin/access`) | Caller's KoT access and per-model permissions | — |
| GET | `/admin/users/:userId/access` (also `/me/users/...`) | A user's KoT access | ✓ |
| POST | `/admin/users/:userId/access` (also `/me/users/...`) | Grant or revoke KoT access and set a role (tenant user-management card) | ✓ |
| GET | `/admin/roles` (also `/me/roles`) | List KoT roles | — |
| GET | `/admin/users/roles` (also `/me/users/roles`) | Map of users → roles | — |
| GET | `/navigation` | Menu | — |
| GET | `/client-profiles` | List (`status` ACTIVE / INACTIVE / FAVORITES, `search`) | ✓ |
| GET | `/client-profiles/:id` | Profile plus per-section permissions | ✓ |
| POST | `/client-profiles` | Create | ✓ |
| PUT | `/client-profiles/:id` | Update 1-to-1 sections / details | ✓ |
| PUT | `/client-profiles/:id/bulk-update` | Track Changes batch | ✓ |
| PUT | `/client-profiles/:id/privacy` | Toggle private (admin) | ✓ |
| PUT | `/client-profiles/:id/assignments` | Replace assignments | ✓ |
| DELETE | `/client-profiles/:id/assignments/:userId` | Remove an assignment | ✓ |
| POST | `/client-profiles/:id/deactivate`, `/activate` | Lifecycle (activate is admin-only) | ✓ |
| POST / DELETE | `/client-profiles/:id/favorite` | Favourite | ✓ |
| GET | `/client-profiles/:id/export?format=docx\|xlsx` | Export | ✓ |
| GET | `/audit-logs` | Track Changes list | ✓ |
| GET | `/reports/entity-status`, `/inventor`, `/licensing` | JSON or `format=xlsx`, with `columns` | ✓ |
| GET | `/foreign-associates` | FA master dropdown | — |
| GET | `/inventor-import/template` | Template xlsx | — |
| POST | `/inventor-import/upload` | Import inventors (multipart `file`; validated per field inside the service) | — |
| GET | `/inventor-import/batches[/:batchId[/result]]` | History, detail, result file | Partly |

The access routes are mounted at both `/admin` and `/me` (`app.js`), so each exists under both prefixes. Permission checks are described in [auth-security.md](auth-security.md).

## Notes on current behaviour

| Topic | Behaviour |
|---|---|
| Access route paths | `/me/access`, `/admin/users/{id}/access` |
| Lists | `/client-profiles` and `/audit-logs` return all rows; `page` / `size` are not used |
| Report downloads | `format=xlsx` (otherwise JSON) |
| Newer routes | `/inventor-import/*` (5 routes), `/foreign-associates`, `/admin/roles`, `/admin/users/roles` |
| Error shapes | Controllers return `{code, message}`; validation and middleware return Boom `{statusCode, error, message}`. Both carry `message` |
| Body size | A body over 5 MB returns 413 (`1e017c4`) |

Controllers map service errors to HTTP codes by `error.message`, so keep service error messages stable when refactoring (review §6).

## External systems

| System | Config names | Used for |
|---|---|---|
| Super-admin Postgres | `SUPERADMIN_DB`, `DB_*` / Secrets Manager | Tenant registry at boot |
| Tenant Postgres (×N) | `Tenant.dbName`; same credentials | All KoT data; pool max 20 per tenant |
| AWS S3 | `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET`, `S3_BUCKET_NAME`, `KOT_TEMPLATE_NAME` | DOCX template (GetObject, transfer-accelerated endpoint) |
| Document processor | `DOCUMENT_PROCESSOR_URL` | `POST /kot/GenerateFilledDoc` (multipart: template + JSON) for DOCX export |
| AWS Secrets Manager | `USE_SECRET_MANAGER`, `DB_SECRET_NAME`, `AWS_REGION` | DB credentials, once per process |
| CloudWatch Logs and Metrics | `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET`, `AWS_REGION`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` | Custom transport (same family as drm-apis and reference_downloader_apis) |

The document processor is likely the `DocumentProcessor` repo in the workspace.
