# epo-bdss-apis — Deployment and environments

`main:.claude/docs/ROLLBACK.md`: deployment is via the Dockerfile; roll back by redeploying the previous image tag.

## Branches

| Branch | Environment |
|---|---|
| `main` | Integration |
| `qa` | qa |
| `uat` | uat |
| `release` | production |

The mapping follows the branch naming used across the services, with changes promoted from `main`. Compare trees before promoting, because some changes have been applied to branches as separate commits (for example TP-317 as `51c1361` on `main` and `ce2b98f` on `release`).

## Build and runtime

- `Dockerfile`: `node:22-alpine`, `npm install --omit=dev`, non-root, `NODE_ENV=production`, `PORT=8020`.
- Boot: resolve credentials → `sequelize.authenticate()` → listen. The service needs the database at startup and exits if it cannot connect.
- Stateless; any number of replicas can run.
- Run tests locally before promoting: `npm test` and `npm run test:identifier`.

## Configuration variable names

From `.env.example` and the code.

| Group | Names |
|---|---|
| Server | `PORT`, `BASE_URL` (Swagger server URL), `NODE_ENV`, `LOCAL_TENANT` (Swagger-path tenant) |
| DB | `DB_NAME` (always from env), `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_DIALECT`, `DB_PORT`, legacy `DATABASE_URL` / `POSTGRES_URL` |
| Secrets Manager | `USE_SECRET_MANAGER`, `DB_SECRET_NAME`, `AWS_REGION` (default `us-east-1`) |
| AWS | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (SDK chain), `AWS_ACCESS_KEY_SECRET` (CloudWatch transport) |
| Logging | `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` |
| Swagger (development and qa) | `AUTH_USERNAME`, `AUTH_PASSWORD`, `SECRET` |

## Infrastructure dependencies

- The `docdb` PostgreSQL database (partitioned, about 350 GB), kept current by `docdb_ingestion`.
- CloudWatch Logs; Secrets Manager (optional).
- The callers' `EPO_BDSS_URL` (in `reference_downloader_apis` and `patent-asistant-apis`) must point at this service's port.
