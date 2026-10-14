# kot-apis — Knowledge Transfer

Describes the code as of 2026-09-18 (`main` = `uat` = `release` at `7daa826`).

## What it is

The backend for **Keep On Top (KoT)**, a module of the Trexo IP platform that keeps a law firm's knowledge about each client in one place. It covers:
- client details;
- licensed families / related parties;
- shadowing and monitoring instructions;
- recurring actions, billing, correspondence, personnel and preferences;
- foreign associates, inventors and annuities;
- working-team assignments.

Around that sit:
- a field-level change audit ("Track Changes");
- role and section-based permissions;
- exports (DOCX via the document processor, XLSX locally);
- three reports;
- bulk inventor import from `.xlsx`.

Multi-tenant, **database per tenant**. KoT tables live in a `keep_on_top` schema inside each tenant DB, and KoT reads the platform's `public.users` table.

## Who uses it

Law-firm staff in KoT roles: Admin, PPS, FFS, US Patent Paralegal, Patent Renewal Specialist, Practitioners, Patent Ops, View Only. They use `trexo-frontend` (`pages/Kot/*`, plus the tenant user-management card for granting access) through `trexo-apigateway` at `/tenantAPI/kot/*`.

## Stack

- Node 22 (`node:22-alpine`) and Express 4.
- Sequelize 6 on PostgreSQL.
- Joi request validation (KOT-264).
- `docx`, ExcelJS, multer (in memory, `.xlsx` up to 10 MB).
- AWS S3 (DOCX template) and axios to the document-processor service.
- Winston with a CloudWatch transport, or daily-rotate files locally.
- Optional AWS Secrets Manager for DB credentials (TP-310).
- Jest 29 with Supertest; ESLint with SonarJS; a Husky pre-push hook.

## Run locally

1. `npm ci`.
2. Create `.env.development.local` with the variable names in [deployment-and-environments.md](deployment-and-environments.md).
3. `npm run start:dev`. Set `PORT` explicitly (the image uses 8011).
4. You need a super-admin DB with a tenant row, and a tenant DB prepared with the steps in `kot-apis: docs/KOT_Tenant_Setup.md`. Restart the service after adding a tenant.

Tests: `npm run test:coverage` (thresholds: lines 95, statements 95, functions 90, branches 85).

## Existing documentation

This package adds to these rather than repeating them. It reflects the code as of the date above; where they differ, follow this package.

| Doc | Where | Use it for |
|---|---|---|
| System Architecture Review | `kot-apis: docs/System_Architecture_Review.md` | The main architecture reference: domain, layering, RBAC, privacy, failure scenarios, operations |
| Architecture Overview | `kot-apis: docs/Architecture_Overview.md` | Target-state overview |
| API Reference | `kot-apis: docs/API_Reference.md` | Request and response examples and the validation section; see [api-and-integrations.md](api-and-integrations.md) for current paths |
| Database Schema | `kot-apis: docs/Database_Schema.md` | Core tables; see [database.md](database.md) for the rest |
| Development Setup | `kot-apis: docs/Development_Setup.md` | Setup notes |
| Tenant setup runbook | `kot-apis: docs/KOT_Tenant_Setup.md` | Onboarding a tenant |
| Postman collection | `kot-apis: docs/Keep-On-Top-APIs.postman_collection.json` | Trying the API |
| README | `kot-apis: README.md` | Request Validation and Secrets Manager sections |
| KoT working notes | `CONSTRAINTS.md`, `DECISIONS.md`, `FLOW.md`, `TESTS.md`, `ROLLBACK.md`, `traces/` | Design decisions, rules for new endpoints, request flow, test approach, rollback |

## This package

| File | For |
|---|---|
| [product-and-features.md](product-and-features.md) | Roles, sections, workflows and business rules |
| [api-and-integrations.md](api-and-integrations.md) | Route inventory, how the gateway exposes it, external calls |
| [database.md](database.md) | Topology, tables beyond the schema doc, how schema changes are applied |
| [auth-security.md](auth-security.md) | Trust model and permission checks |
| [deployment-and-environments.md](deployment-and-environments.md) | Branches, CI, image, configuration, tenant onboarding |
| [handover.md](handover.md) | Recent delivered work, dependencies, ownership |

Architecture, testing, operations and rollback are covered by the review and the working notes listed above.

## Suggested reading order

1. This file.
2. `docs/System_Architecture_Review.md`.
3. The working notes: `CONSTRAINTS.md` and `DECISIONS.md`.
4. [product-and-features.md](product-and-features.md) → [api-and-integrations.md](api-and-integrations.md) → [database.md](database.md) → [auth-security.md](auth-security.md).
5. [handover.md](handover.md).
