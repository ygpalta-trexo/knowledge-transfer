# drm-apis — Auth and security

`drm-apis: docs/authentication-authorization.md` describes this service on its own. The full trust model includes the gateway, so this file covers the whole chain.

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| Authentication | Delegated to `trexo-apigateway`, which verifies an HS512 JWT, checks it against its Redis token cache, and checks the token's tenant matches the request | `trexo-apigateway: middleware/authentication.js` |
| App entitlement | The gateway requires `DRM` in the user's `apps` | `appAccess("DRM")` |
| Identity propagation | The gateway sends `X-User` (JSON of the JWT claims) and `X-Tenant-Name` (Origin subdomain); drm-apis reads them | `server.js` |
| Tenant isolation | A separate database per tenant, chosen by `X-Tenant-Name` | `server.js` tenant resolver |
| Network placement | drm-apis is designed to be reached through the gateway and by internal schedulers | — |
| CORS | `*.<company-domain>`, localhost and server-to-server (no-origin) requests; credentials allowed | `server.js` |
| Upload validation | Buffer MIME detection plus XLSX ZIP structure and path-traversal checks | `validateFileMimeType`, `POST_MERGE_STEPS.md` §1 |
| SQL | Audit procedure calls come from a fixed allowlist with bound parameters; the column-preferences upsert uses named replacements | `audit.controller.js`, `auditResults.controller.js` |
| DB credentials | Env vars, or AWS Secrets Manager (`USE_SECRET_MANAGER=true`), fetched once at startup | `config/dbCredentials/*` |
| Container | Non-root `nodejs` user; `.env*` excluded from the image by `.dockerignore` | `Dockerfile`, `.dockerignore` |
| Request logging | Every request is logged with `requestId`, `tenant` and `username` | `middleware/requestLogger.js` |

## File handling

- HTTP uploads stay in memory (Multer memory storage).
- SFTP downloads are written to `/tmp` inside the container.
- ZIP entry names containing `..` or absolute paths are rejected.

## Tenant isolation in the queue

Each Bull job carries `tenant` in its payload, set by the server-side producers (the upload controller and the cron). A tenant's custom mapping is copied per job, so it cannot affect another tenant (`queueProcessor.js` `getMappingAndTable`).

## Handling data

- Client docket data is confidential. Bull Board (`/admin/queues`) shows job payloads, which contain uploaded file rows, so treat access to it as access to client data.
- `cancel_audit`, `deleteFromArchives` and the 90-day purge delete data permanently.
