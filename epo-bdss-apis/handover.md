# epo-bdss-apis — Handover

As of 2026-09-18.

## Recent delivered work

| Date | Commit | Ticket / change |
|---|---|---|
| 2026-09-15 | `ce2b98f` | TP-317 Secrets Manager (`release`) |
| 2026-09-11 | `f8e5730` | TAI-543: US 10↔11-digit conversion, input sanitisation |
| 2026-09-01 | `51c1361` | TP-317 Secrets Manager (`main`) |
| 2026-08-31 | `e070eff` | Batched application-biblio export script |
| 2026-08-11 | `0651c43` | US numbers checked raw and parsed |
| 2026-07-31 | `40ae354` | KR 13-digit identifiers |
| 2026-06-16 | `fe7e561` | Family grouping by application |
| 2026-06-02 | `72307cf` | Family grant date |
| 2026-05-25 | `8bad2d9` | Input and output shaped for the reference downloader |
| 2026-05-21 | `fa4efd2` | Use the flattened DB table |

Design reasoning for these changes is in `main:.claude/docs/DECISIONS.md`.

## External dependencies

- **Callers:** `reference_downloader_apis` and `patent-asistant-apis` (SB-08 Generator and others), both via `EPO_BDSS_URL`, `POST /api/publication/bulk`. The contract is in [api-and-integrations.md](api-and-integrations.md#the-bulk-contract).
- **Data:** the `docdb` Postgres store, loaded by `docdb_ingestion` ([../docdb_ingestion/handover.md](../docdb_ingestion/handover.md)).
- CloudWatch; Secrets Manager.

## Ownership

| Area | Owner |
|---|---|
| epo-bdss-apis code | |
| DOCDB ingestion and data freshness | |
| The `docdb` database (350 GB, shared) | |
| Deploys | |
