# file_num_generation — Product and features

## Business problem

A law firm needs a unique internal **family file number** for each new IP matter before work starts. Numbers come from separate series per matter type and office (sub-org), must never collide, and the firm keeps a record of who took which number and why. This service issues those numbers and keeps that record.

## Users and roles

| Who | What they do | Where the rule lives |
|---|---|---|
| Staff of the number-generator tenant | Generate, save and delete numbers; see My List | `trexo-frontend: pages/fileNumberGenerator/generate.js`, `myList.js` |
| Tenant admins (`TENANT_ROLE == "admin"`) | See the Admin page: user management (served by the gateway) and the Export | `trexo-frontend: pages/fileNumberGenerator/Admin.js`, `fileNumberGeneratorMenus.js` |

## The numbering scheme

Format: `<Y1><Y2><NNN>`, always a single-letter Y1 and three digits (`001`–`999`).

| Counter sub-org | Matter type | Y1 letters, in issue order | First number |
|---|---|---|---|
| US | PATENT | P, Q, R, S, A, B, E, F, H, I, J, K, N, O | PA001 |
| US | TRADEMARK | T, M | TA001 |
| US | DESIGN | D | DA001 |
| UK | PATENT | U, V, W, X | UA001 |
| UK | TRADEMARK | Z | ZA001 |
| UK | DESIGN | Y | YA001 |
| OTHER (shared) | GENERAL | G | GA001 |
| OTHER (shared) | COPYRIGHT | C | CA001 |
| OTHER (shared) | Matter with Opposing Parties | L | LA001 |

Source: `FileNumberService.MATTER_TYPE_MAPPINGS`, repeated in `src/seeders/20240326000000-initial-sequence-counters.js`.

How the counter advances:
1. NNN goes up by one.
2. After 999, NNN goes back to 001 and Y2 moves to the next letter.
3. After Y2 `Z`, Y1 moves to the next letter in the table and Y2 goes back to `A`.
4. After the last Y1 of a series, generation for that series stops with `Maximum file number limit reached`.

Capacity is 26 × 999 = **25,974 numbers per Y1 letter**. That makes 363,636 for US patents, 51,948 for US trademarks, and 25,974 for every single-letter series.

**Shared series.** GENERAL, COPYRIGHT and Matter with Opposing Parties count under sub-org `OTHER`, whichever sub-org the user picked. The row stores the user's choice (`US`, `UK` or `US,UK`). `US,UK` is accepted only for these three types.

**Uniqueness.** `(y1, y2, sequenceNumber)` is unique across the whole table, so the same number never exists in two series.

## Business rules

| Rule | Where |
|---|---|
| `generatedBy` is the email in the `X-User` header. Without it: `400 {"error": "User email is required"}` | `fileNumberController.generateFileNumber` |
| `reason` is optional in the API (at most 255 characters, may be empty). The UI requires it for a single number and sends none for bulk | validators; `trexo-frontend: generate.js` |
| Bulk count: 1–100 in the API; the UI offers 1–50 | same |
| A bulk request is all-or-nothing (one transaction), and its rows share one `bulkGroupId` | `generateBulkFileNumbers` |
| **Save** sets `used = true` | `markFileNumberAsUsed`, `bulkMarkFileNumbersAsUsed` |
| **Delete** removes the row | `deleteFileNumber`, `bulkDeleteFileNumbers` |
| **Rewind on delete:** if the deleted number is the counter's current position, the counter moves back to the highest remaining number in the series (ordered by Y1, Y2, sequence), so the next request is given the deleted number again. Deleting any other number leaves a gap that is not reused | same |
| If no number remains in the series, the counter is set to the **starting counter** for that series: `FILE_NUMBER_STARTING_COUNTER` (JSON) or the built-in values below. The counter is placed *on* that position, so the next number issued is the one after it | `getStartingCounter` |
| Numbers that were generated but never saved or deleted stay `used = false` | model default |

Built-in starting counters (used only after a series has been emptied by deletes):

| Sub-org | PATENT | TRADEMARK | DESIGN | GENERAL | COPYRIGHT | Matter with Opposing Parties |
|---|---|---|---|---|---|---|
| US | PA001 | TA001 | DA001 | | | |
| UK | UA001 | TZ001 | DY001 | | | |
| OTHER | | | | GA001 | CA001 | LA001 |

After a series has been emptied, check its counter row before users generate again (see [operations-and-troubleshooting.md](operations-and-troubleshooting.md#correcting-a-counter-by-hand)).

## Paths that need care

1. **Generation** moves the shared counter (`SequenceCounters`).
2. **Delete** can move the counter backwards.
3. **Counter corrections** go through the manual procedure in [operations-and-troubleshooting.md](operations-and-troubleshooting.md#correcting-a-counter-by-hand). `POST /update-counters` is not part of the normal workflow.

`.claude/docs/CONSTRAINTS.md` asks for agreement before any change to numbering or counter logic.

## Features

### Generate one number

- **Frontend:** `generate.js` posts `{matterType, subOrg: "US" | "UK" | "US,UK", bulk: false, count: 1, reason}` and then shows the number with Save and Delete buttons.
- **Backend flow:**
  1. `POST /generate` → `validate(generateFileNumberBodySchema)` → `FileNumberController.generateFileNumber`, which reads the email from `X-User`.
  2. `FileNumberService.generateFileNumber` opens transaction T1 and checks that the series' `SequenceCounter` exists (`Sequence counter not found` otherwise).
  3. `getNextY1Y2` runs in its own transaction T2: it reads the counter, computes the next position, updates the counter and commits.
  4. Back in T1: `FileNumber.upsert({...})` with the number, matter type, the user's sub-org, `generatedBy`, `reason` and `used: false`, then commit.
  5. Response `201 {sequence, y1, y2, sequenceNumber, subOrg}`.
- **Persistence:** one counter update and one `FileNumbers` row.

### Generate in bulk

- **Frontend:** the same form in bulk mode, `{bulk: true, count: N}`.
- **Backend flow:**
  1. `FileNumberService.generateBulkFileNumbers` works in one transaction: find or create the counter, then loop N times.
  2. Each slot is created as a new row. If a row already exists at that slot, it is updated with the new request's details.
  3. The counter is updated once, at the end.
- **Response:** `201 {bulk: true, count, numbers: ["PA010", …]}`.

### Save (confirm)

- **Frontend:** `{fileNumber: "PA010"}`, or for bulk `{fileNumber: [..], bulk: true}`.
- **Single save:** parse with `^([A-Z]+)([A-Z])(\d+)$`, look the row up, then set `used = true`. `404` if the row is not found.
- **Bulk save:** one lookup for all entries, then one update per row, in one transaction.
  - Response: `200 {bulk: true, results: [{fileNumber, status: "success" | "error", error?}]}`.
  - Send numbers zero-padded, as the service issued them (`PA001`, not `PA1`).

### Delete (discard)

- **Single delete**, in one transaction:
  1. Remove the row.
  2. Load the series counter.
  3. If the counter equals the deleted number, move it to the next-highest remaining number, or to the starting counter if none remain.
- **Bulk delete:** removes every found row, then applies the rewind rule to the range. It expects one ascending bulk group from a single series, which is what the UI sends.

### My List

- **Frontend:** `GET /list?email=<the signed-in user's email>`.
- **Backend:** `getFileNumbersByUser(email)` returns the user's numbers, newest first. Each bulk group is collapsed into one row: `sequence: "PA010 - PA019"`, `count`, and `status.used` = every row used.
- **Response:** `200 {success: true, data: [...]}`.

### Export (admin)

- **Frontend:** `Export.js` → `GET /export` (downloads a file).
- **Backend:** builds an `.xlsx` of all numbers, grouped as in My List, with 6 columns:
  - Family Number Request Basis (the reason);
  - Sub-org;
  - Matter Type;
  - Family Number Generated (a number or a bulk range);
  - Number Generated by userid;
  - Date/Time File Number Generated (formatted `en-GB` in the server's time zone).
- **Response:** `Content-Disposition: attachment; filename="file_numbers_export.xlsx"`.

### Look up one number

`GET /file/:fileNumber` → the `FileNumber` row, or `404 {error: "File number not found"}`. The frontend does not call it.

### Recalculate counters

`POST /update-counters` rebuilds counter rows from the existing file numbers. The frontend does not call it. Correct counters with the manual procedure in [operations-and-troubleshooting.md](operations-and-troubleshooting.md#correcting-a-counter-by-hand).
