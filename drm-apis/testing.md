# drm-apis — Testing

## Framework

Jest 30 with Supertest. `jest.config.js`:
- `testMatch: tests/**/*.test.js|spec.js`;
- setup file `tests/jest.setup.js`;
- `forceExit` and `detectOpenHandles`;
- 20 s timeout;
- coverage thresholds of 95% lines, functions and statements and 90% branches.

## How to run

```bash
npm test                 # all suites (NODE_ENV=test)
npm run test:coverage    # with thresholds; what CI and the pre-push hook run
npm run lint
```

Under `NODE_ENV=test`, `server.js` does not call `listen` or `process.exit`, so Supertest drives the app directly.

## What is covered (46 test files)

| Area | Files |
|---|---|
| Credential resolution | `tests/unit/config/dbCredentials/*` (6), `services/aws/secretsManagerClient.test.js` |
| Controllers | `tests/unit/controllers/*` (8) plus `tests/integration/*.integration.test.js` for activity codes, audit, audit results, file upload, rules, cron |
| Cron | `tests/unit/cron/cronHelpers`, `cronJobs`, `index` |
| Queue | `tests/unit/queue/queueProcessor`, `queues`, `tests/integration/queue.integration.test.js` |
| Middleware and bootstrap | `authentication`, `errorHandler`, `findTenantMiddleware`, `loggingMiddleware`, `requestContext`, `loadEnv`, `database`, `setupDRM`, `server.test.js` |
| Models | `closedDocketPa.model`, `tenantsModel`, `usersModel` |

The integration tests use mocked DB, Redis, SFTP and SMTP clients (`tests/helpers/mockHelpers.js`, `dbHelpers.js`). The stored procedures and real-file parsing are verified in UAT with one file per type, as the release runbook describes.

## Test data

`tests/fixtures/` holds JSON data; `tests/helpers/fileHelpers.js` is the place to generate synthetic workbooks. Do not commit real client exports.
