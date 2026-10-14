# file_num_generation — Handover

As of 2026-09-22 (`main` at `0091a95`).

## Recent work (last ~3 months)

| Date | Commit | Ticket | What |
|---|---|---|---|
| 2026-09-17 | `0091a95` (PR #13) | TP-272 (under TP-264) | Joi validation on every input route; header checks moved to `src/middleware/requestContext.js`; `errorHandler` answers header rejections with a JSON 400; the `.claude/docs/` trail; README "Request Validation" |
| 2026-07-24 | `eed94a2` | TP-284 | DB credentials from AWS Secrets Manager or env vars (`USE_SECRET_MANAGER`), TTL cache, `beforeConnect` rotation, SSL for qa/uat/production, `DB_SSL_REJECT_UNAUTHORIZED` |

Earlier history, for context:
- 2026-03-10: qa/uat database configuration and dependency updates.
- 2026-01-06: Dockerfile added.
- 2025-07-04: CFN-21 bulk generation.
- 2025-06-23: CFN-17 renamed the matter type to "Matter with Opposing Parties".
- First commit 2025-06-03.

## External dependencies

- `trexo-apigateway`: `FILE_NUMBER_GENERATOR_API_URL`; the `X-User`, `X-Tenant-Name` and `Request-Id` headers; `NUMBER_GENERATOR_TENANT` login rules.
- `trexo-frontend`: `src/app/pages/fileNumberGenerator/`, `REACT_APP_NUMBER_GENERATOR_TENANT`. The frontend is the consumer, so agree API shape changes with it first.
- Postgres, AWS Secrets Manager, CloudWatch Logs and Metrics.

## Ownership

| Area | Owner |
|---|---|
| file_num_generation code | |
| Numbering rules (series letters, rewind, who may delete) | |
| Number-generator tenant (business contact) | |
| Gateway route and tenant allow-list | |
| Deploys and infrastructure (image, env vars, database) | |

## Where to look

- Rules for changing the repo: `.claude/docs/CONSTRAINTS.md`.
- Design reasons: `.claude/docs/DECISIONS.md`.
- Tests and manual checks: `.claude/docs/TESTS.md`.
- Runbooks: [operations-and-troubleshooting.md](operations-and-troubleshooting.md).
