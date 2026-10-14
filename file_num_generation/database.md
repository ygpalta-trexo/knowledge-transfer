# file_num_generation — Database

## Technology

PostgreSQL through Sequelize 6. There is one database per deployment (`DB_NAME`), in the default schema. Schema history lives in `src/migrations/`, run with `sequelize-cli`: `.sequelizerc` points to `src/config/database.js`, which resolves credentials the same way the app does. One seeder creates the counters.

## Entities

### `FileNumbers` (model `FileNumber`), one row per issued number

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | `UUIDV4` default |
| `y1` | varchar(1) | Series letter |
| `y2` | varchar(1) | `A`–`Z` |
| `sequenceNumber` | integer | 1–999 |
| `matterType` | ENUM | `PATENT`, `TRADEMARK`, `DESIGN`, `GENERAL`, `COPYRIGHT`, `Matter with Opposing Parties` |
| `subOrg` | varchar(255) | `US`, `UK` or `US,UK`: the user's choice, not the counter sub-org |
| `generatedBy` | varchar(255) | Email from `X-User` |
| `reason` | varchar(255), null | The request basis |
| `used` | boolean | `true` once saved |
| `bulkGroupId` | UUID, null | Shared by the rows of one bulk request |
| `createdAt`, `updatedAt` | timestamptz | |

Indexes:
- `file_number_unique_idx`: **unique** `(y1, y2, sequenceNumber)`;
- `file_number_sequence_idx`: `(matterType, subOrg, y1, y2, sequenceNumber)`;
- `file_number_bulk_group_idx`: `(bulkGroupId)`.

### `SequenceCounters` (model `SequenceCounter`), one row per series

| Column | Notes |
|---|---|
| `id` | serial PK |
| `matterType`, `subOrg` | Unique together (`sequence_counters_matter_type_sub_org_unique`). `subOrg` is `US`, `UK` or `OTHER` |
| `currentY1`, `currentY2`, `currentSequence` | The **last issued** position. Seeded at `<first Y1>`, `A`, `0` |
| `lastUpdated` | Set on generation |

The seeder inserts **9 rows**: US PATENT / TRADEMARK / DESIGN, UK PATENT / TRADEMARK / DESIGN, and OTHER GENERAL / COPYRIGHT / Matter with Opposing Parties (`src/seeders/20240326000000-initial-sequence-counters.js`).

### Other tables

The initial migration also creates `MatterTypeMappings` and `FileNumberReservations`. Request handling does not use either.

## Relationships

None are declared. A `FileNumbers` row belongs to the `SequenceCounters` row with the same matter type and effective sub-org: `OTHER` for the three shared-series types, otherwise the row's own `US` / `UK`.

## Transactions

| Operation | Transactions |
|---|---|
| Generate (single) | T1 (row write) around T2 (counter update); T2 commits first |
| Generate (bulk) | One |
| Save | One (single or bulk) |
| Delete (single or bulk) | One: delete, read the counter, rewind if needed |

The isolation level is the Postgres default, READ COMMITTED.

## Query patterns

- Point lookups by `(y1, y2, sequenceNumber)`, served by the unique index.
- Bulk save and delete: `WHERE (y1=… AND y2=… AND sequenceNumber=…) OR …`, up to 100 terms.
- Rewind: the highest remaining row in the series, `ORDER BY y1 DESC, y2 DESC, sequenceNumber DESC LIMIT 1`.
- List: `WHERE generatedBy = ? ORDER BY createdAt DESC`.
- Export: all rows, `ORDER BY createdAt DESC`.

## Migrations

| File | What |
|---|---|
| `20240323000000-add-status-columns-to-file-numbers.js` | Adds `used`, `deleted`, and an index on `(used, deleted)` to `FileNumbers` |
| `20240326000000-create-initial-tables.js` | Creates `FileNumbers`, `FileNumberReservations`, `MatterTypeMappings` and `SequenceCounters`, with their indexes |
| `20250613104310-initial-schema.js` | No operations |
| `20250708091910-add-bulk-group-id-to-filenumber.js` | Adds `bulkGroupId` and its index |
| `20250708_remove-deleted-from-filenumbers.js` | Drops `deleted` |

Every migration defines a `down`.

### Setting up a new database

`sequelize-cli` runs migrations in filename order, so on an empty database `20240323…` (which alters `FileNumbers`) runs before `20240326…` (which creates it). Agree the set-up of a new environment's schema with the service owner instead of running `db:migrate` alone. After the schema exists, run `db:seed:all` once to create the 9 counter rows.

Before any schema change on an existing environment, check its state with `npx sequelize-cli db:migrate:status`, using that environment's variables.

## Connection management

- Pool: `max: 5`, `min: 0`, `acquire: 30000`, `idle: 10000`.
- In Secrets Manager mode, `beforeConnect` supplies fresh credentials for each new physical connection.
- `logging: false`.
- SSL is on for `qa`, `uat` and `production`. Set `DB_SSL_REJECT_UNAUTHORIZED=true` to verify the certificate chain.
