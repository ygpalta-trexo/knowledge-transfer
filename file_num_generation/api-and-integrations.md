# file_num_generation — API and integrations

The **validation rules** per endpoint are documented in `file_num_generation: README.md` → "Request Validation". For endpoint paths and response shapes, use this file.

## Paths

The router is mounted at `/`. Through the gateway every path sits under `/tenantAPI/file-number-generator/`, for example `POST /tenantAPI/file-number-generator/generate`.

## Common behaviour

- **Headers:** every request except `GET /health` needs `X-User` (a JSON object) and `X-Tenant-Name`, otherwise it gets a Boom `400` (`src/middleware/requestContext.js`). `Request-Id` is optional and becomes the log correlation id.
- **Order:** header check → CORS → `express.json()` (default 100 kb limit) → per-route Joi → handler → `errorHandler` (last).
- **CORS:** allowed when there is no `Origin`, when `NODE_ENV=development`, or when the origin is the platform's own domain or one of its subdomains. The browser talks to the gateway, which calls the service server-to-server.
- **Error shapes.** The frontend shows generic messages and does not read the error bodies (`.claude/docs/DECISIONS.md`).

  | Shape | Produced by |
  |---|---|
  | `{statusCode, error, message}` (Boom) | Header checks, Joi failures, bad JSON bodies, anything passed to `errorHandler` |
  | `{error: "<text>"}` | Controller and route handlers (404s, 500s, "User email is required") |
  | `{success: false, message, error?}` | `GET /list` failures |

## Endpoint inventory

| Method and path | Purpose | Input (validated) | Success | Failure |
|---|---|---|---|---|
| `GET /health` | Liveness | — (no headers) | `200 OK` | — |
| `POST /generate` | Issue one or N numbers | Body: `matterType`, `subOrg`, `reason?`, `bulk?`, `count` (required when bulk) | Single: `201 {sequence, y1, y2, sequenceNumber, subOrg}`. Bulk: `201 {bulk: true, count, numbers[]}` | `400` Boom; `400 {error: "User email is required"}`; `500 {error}` (for example `Sequence counter not found`, `Maximum file number limit reached`) |
| `POST /save` | Mark as used | Body: `fileNumber` (string or string[]), `bulk?` | `200 {message}` or `200 {bulk: true, results: [{fileNumber, status, error?}]}` | `400` Boom; `404 {error: "File number not found"}` (single); `500 {error}` |
| `POST /delete` | Delete, and rewind the counter when the highest number is deleted | as `/save` | `200 {message: "File number deleted successfully"}` or `200 {bulk: true, results}` | `400` Boom; `404`; `500 {error}` |
| `GET /file/:fileNumber` | Row lookup | Param `^[A-Z]+[A-Z]\d+$` | `200` the `FileNumber` row | `400` Boom; `404 {error}`; `500 {error}` |
| `GET /list?email=` | A user's numbers, bulk groups collapsed | Query `email` (one non-empty string) | `200 {success: true, data: [{sequence, count?, matterType, subOrg, status: {used}, reason, generatedBy, generatedAt, bulk}]}` | `400` Boom; `500 {success: false, message, error}` |
| `GET /export` | `.xlsx` of all numbers | — | `200` workbook, `Content-Disposition: attachment; filename="file_numbers_export.xlsx"` | `500 {error}` |
| `POST /update-counters` | Rebuild counters from data (not used by the frontend) | — | `200 {message: "Sequence counters updated successfully"}` | `500 {error}` |

## Frontend contract (`trexo-frontend: src/app/pages/fileNumberGenerator/`)

| UI action | Call | Payload |
|---|---|---|
| Generate (single) | `POST /file-number-generator/generate` | `{matterType, subOrg: selected.join(','), bulk: false, count: 1, reason}` |
| Generate (bulk) | same | `{matterType, subOrg, bulk: true, count}` |
| Save | `POST /file-number-generator/save` | `{fileNumber}` or `{fileNumber: [...], bulk: true}` |
| Delete | `POST /file-number-generator/delete` | as Save |
| My List | `GET /file-number-generator/list?email=${email}` | Email from `localStorage.TENANT_USER_MAIL` |
| Export | `GET /file-number-generator/export` | — (blob) |

`.claude/docs/CONSTRAINTS.md` treats public API shape changes as ask-first, because the frontend is the consumer.

## Gateway contract (`trexo-apigateway`)

- **Mount:** `tenantAPIRouter.use("/file-number-generator", createProxy("FILE_NUMBER_GENERATOR_API_URL"))`, after `findTenant` and `checkAuthentication`.
- **Headers set by the proxy:**
  - `Request-Id`;
  - `X-Tenant-Name`, taken from the first label of the browser `Origin`, or `LOCAL_TENANT` for a bare host;
  - `X-Origin`;
  - `X-User`: `JSON.stringify(req.user)`.
- **Login rules** for the tenant named in `NUMBER_GENERATOR_TENANT`: an IP allow-list, and a notification on login attempts by unknown emails.

## External systems

| System | How it is called | Config names |
|---|---|---|
| PostgreSQL | Sequelize 6 / `pg`, pool max 5. SSL (`require`) when `NODE_ENV` is `qa`, `uat` or `production` | `DB_NAME`, `DB_HOST`, `DB_USERNAME` (alias `DB_USER`), `DB_PASSWORD`, `DB_DIALECT`, `DB_SSL_REJECT_UNAUTHORIZED` |
| AWS Secrets Manager | `GetSecretValue(DB_SECRET_NAME)`, a JSON `{username, password, host, engine}`. Cached for `DB_CREDENTIALS_TTL_MS` (default 10 min) and refreshed in `beforeConnect`. If a refresh fails, the last good value is used | `USE_SECRET_MANAGER`, `DB_SECRET_NAME`, `AWS_REGION` (default `us-east-1`) |
| AWS CloudWatch Logs and Metrics | Winston transport: batched `PutLogEvents`, creates the log group and stream, sets retention, reports its own delivery metrics | `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_ACCESS_KEY_SECRET`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` |

The service makes no other outbound HTTP calls.
