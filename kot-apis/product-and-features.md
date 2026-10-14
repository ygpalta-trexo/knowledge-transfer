# kot-apis — Product and features

Domain shape and invariants are covered in `kot-apis: docs/System_Architecture_Review.md` (§1, §5, §10 privacy, §15 failure scenarios). This file adds a product-level view: who does what, and the rules each feature enforces.

## The problem KoT solves

A law firm's knowledge about each client is usually spread across people: who the working team is, how to bill, which families are licensed or monitored, standing instructions for paralegals and PPS staff, preferred foreign associates, inventor details, annuity handling. KoT keeps it in a structured **client profile** with a change history, per-role visibility and exportable summaries.

## Roles

Roles are seeded from `src/config/rbacMatrix.js`; at runtime they are read from the DB tables `kot_roles` and `role_section_permissions`.

| Role id | Name | Notes |
|---|---|---|
| `ADMIN` | Admin | `isAdministrator`: bypasses section permissions and private-profile restrictions; the only role that may reactivate profiles or change privacy |
| `PPS` | PPS | Special handling in assignment rules (`updateAssignments` / `deleteAssignment`) |
| `FFS`, `US_PARALEGAL`, `RENEWAL_SPECIALIST`, `PRACTITIONER`, `PATENT_OPS` | — | Section rights come from the matrix |
| `VIEW_ONLY` | View Only | `readOnlyRoles`: never mutates, exports or downloads, whatever the section flags say (KOT-117) |

At runtime a user has one KoT role (`getUserAccess` returns a single `kotRoleId`).

## Sections of a client profile

These are the permission units (`rbacMatrix.sections`):
- Client Details
- Licensed Families (shown as "Related Parties" since KOT-116)
- Shadowing/Monitoring: status by family, instructions for the paralegal, instructions for PPS
- Recurring Actions
- Billing & Invoicing
- Client Preferences
- Foreign Associates
- Inventor Information
- Agent & Attorney Preferences
- Annuities
- Additional Information
- Working Team

Each role has `can_view` / `can_edit` per section.

## Features and rules

### Client profile lifecycle
- **Create:** `clientNumber` must be unique among *active* profiles (checked in the service), so a deactivated number can be reused. Private profiles need at least one working-team assignment.
- **Update:**
  - `PUT /client-profiles/:id` handles 1-to-1 sections and Client Details.
  - 1-to-N sections go through **bulk update** (`PUT /:id/bulk-update`), the "Track Changes" write path: a list of per-field `CREATE`/`UPDATE`/`DELETE` changes across section models, with no-op pruning and delete snapshots.

  The body limit is 5 MB, because large inventor lists send one change per field per record.
- **Deactivate / reactivate:** status changes only; profiles are not deleted through the API. Reactivation is admin-only. INACTIVE profiles are read-only.
- **Privacy and assignments:** non-admins see a private profile only if they are assigned to it. At least one working-team assignment must always remain.
- **Favourites:** per-user flag.
- **Audit:** every mutation writes `audit_logs` rows in the same transaction. The Track Changes UI reads them through `GET /audit-logs`.

### Exports
- **DOCX:** a template is fetched from S3 (`S3_BUCKET_NAME` / `KOT_TEMPLATE_NAME`), then POSTed with the profile JSON to the document processor, `DOCUMENT_PROCESSOR_URL/kot/GenerateFilledDoc`, which renders the document.
- **XLSX:** built locally with ExcelJS.
- View-only users cannot export.

### Reports

Three reports: Entity Status, Inventor, and Licensing/Related Parties (`reports.controller.js`, `reports.service.js`).
- JSON for the on-screen grid, or `format=xlsx` for download.
- Filters:
  - entity status: `clientName`, `assignedPps`, `entityType`;
  - inventor: `clientName`, `country`, `fromDate`, `toDate`;
  - licensing: `clientName`, `licensedTo`, `lastUpdated`.
- `columns` (KOT-30) narrows the xlsx to the grid's visible columns, in definition order; an empty value gives every column.
- Private-profile visibility is applied through `visibleProfileWhere`; filtering and flattening happen in the service after loading.

### Inventor bulk import (KOT-101, KOT-103, KOT-106, KOT-109)

`services/inventorImport.service.js`, `config/inventorLookups.js`:
1. Download the template (`GET /inventor-import/template`).
2. Upload `.xlsx` for **one** client profile (`POST /inventor-import/upload`): up to 2,000 rows and 10 MB, processed within the request.
3. Validation is per field. An unknown optional value is dropped and reported, and the row still imports ("Partial Success"). Rows with no name, duplicates or DB errors fail.
4. Each accepted row is written in its own transaction, together with its Track Changes entry, so one bad row does not affect the others.
5. Upload history (`GET /inventor-import/batches`), batch detail, and a downloadable row-level result named after the uploaded file (KOT-103).
6. File-level rejections are recorded in the history too (KOT-109).

### Foreign associate master list
- A read-only dropdown source, `GET /foreign-associates` (table `foreign_associate_master`).
- Maintained by script: `scripts/seedForeignAssociates.js <tenantCode> [--prune]`, from the tracked CSV `foreign associates/fa table.csv`. `--prune` removes entries no longer in the CSV. The latest refresh is KOT-129 (`7daa826`, 2026-09-11).
- Profile-level foreign associates store country, name and email as values (no foreign key to the master list), so refreshing the list never changes existing profiles.

### Access administration
- Tenant admins grant KoT access and a role from the tenant user-management screen (`trexo-frontend: pages/tenants/UserCard.js` → `POST /kot/admin/users/:personId/access`).
- `scripts/addUserToKot.js` does the same from the command line.

### Navigation
`GET /navigation` returns the menu items for users with KoT access.

## Data lifecycle

Profiles are deactivated rather than deleted, and `audit_logs` keeps the full change history.

## Critical paths

1. **Bulk update** (Track Changes): the most generic write path, with the largest payloads.
2. **Access administration**: controls who can see which profiles.
3. **DOCX export**: depends on S3 and the document processor.
4. **Tenant resolution**: tenants are loaded at startup, so restart after adding one.
