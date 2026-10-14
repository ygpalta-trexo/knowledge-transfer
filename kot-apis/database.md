# kot-apis — Database

`kot-apis: docs/Database_Schema.md` is the base reference, together with `docs/System_Architecture_Review.md` §5 and §7. This file adds the rest.

## Topology

- **Super-admin DB** (`SUPERADMIN_DB`): the `Tenant` registry (`tenantCode`, `dbName`, …), read once at boot.
- **One DB per tenant.** KoT tables live in schema `keep_on_top` (32 models). KoT reads the platform's `public.users` (the `User` model), which the platform owns.
- One Sequelize pool per tenant (max 20, min 0, acquire 120 s, idle 10 s), created at boot. Credentials are applied through the `beforeConnect` hook (TP-310); database names come from the tenant registry.

## Tables beyond `Database_Schema.md`

| Table | Model | Purpose |
|---|---|---|
| `inventor_import_batches` | `InventorImportBatch` | One row per upload: file name, client profile, counts, status (KOT-101) |
| `inventor_import_rows` | `InventorImportRow` | Row-level outcome and remarks for the result download |
| `foreign_associate_master` | `ForeignAssociateMaster` | Dropdown reference list; integer PK; script-maintained |
| `agent_attorney_preferences` | `AgentAttorneyPreference` | Section "Agent & Attorney Preferences" |
| `profile_deactivation_events` | `ProfileDeactivationEvent` | Lifecycle history |
| `kot_roles` row `VIEW_ONLY` | — | Added by `scripts/20260729120000-add-view-only-role.sql` (KOT-117) |

The full model → table list is `src/models/tenant/*.js`.

## Applying schema changes

| Mechanism | What it does | When to use it |
|---|---|---|
| `scripts/*.sql` + `node scripts/runSqlMigration.js <file.sql> <tenantCode\|--all>` | Runs idempotent (`IF NOT EXISTS`) SQL against one tenant or every tenant; forward-only | **Schema changes on existing tenants** |
| `scripts/setupKotTenant.js <tenant>` | `model.sync({ alter: true })` for every KoT model | **New tenant DBs only.** `alter` reshapes tables to match the models, which can remove columns an existing table has |
| `scripts/seedForeignAssociates.js` | `ForeignAssociateMaster.sync({ alter: true })`, then upserts (and prunes with `--prune`); reads connection settings from `.env.development.local` | Refreshing the FA list |
| `scripts/initializeKotRoles.js` | `sync()` (create if missing) for the 3 RBAC tables, then `findOrCreate` from `rbacMatrix` | Role seeding |

The dated SQL scripts in `scripts/` (2026-05-11 → 2026-08-11) form the migration history. Record which scripts you run on which tenant.

**Adding a tenant model:** register it in `src/models/tenant/index.js` and create its table in every tenant DB. If it takes part in bulk update or RBAC, also add it to `config/schemaMap.js` and `utils/rbacHelper.js`.

## Keys, uniqueness, transactions

- Primary keys are `STRING` columns holding `uuidv4()` values. The exceptions are the integer `users.personid`, `profile_assignments.user_id` and `user_kot_access.user_id`.
- `clientNumber` uniqueness among active profiles is a service rule.
- Mutations and their audit rows share a transaction. Inventor import uses one transaction per row, plus one for the batch and row records.

## Query patterns

- The profile list and the audit-log list return all rows (pagination was removed in `e089a26`, 2026-06-23).
- Reports eager-load, then filter and flatten in the service.
- Access checks: `getUserAccess` runs two queries per gated request, and section checks read `role_section_permissions`.
