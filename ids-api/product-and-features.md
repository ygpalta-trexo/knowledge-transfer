# ids-api — Product and features

## Business problem

Before filing, and while prosecuting, a US application, a paralegal needs to know what prior art has been cited:
- **by the applicant**, in the **IDS**, submitted on form **PTO/SB/08** (or the older 1449);
- **by the examiner**, on the **PTO-892** "Notice of References Cited".

Reading scanned forms by hand across many applications is slow. The JUNE chatbot lets the user type application numbers and receive a spreadsheet by email.

## SB-08 and 892 Reference Analysis (TAI-1181, TAI-1201)

**Purpose:** for up to 100 application numbers, count the references in every SB-08, and list every reference from both SB-08 and PTO-892 in one workbook. It ships alongside Reference Count (`GET /ids-api/data`), which is unchanged.

**Flow:**
1. Chat → Lex `Sb08And892Intent` (slot `application_nums`) → the Lambda → `GET /ids-api/sb08-892 {applicationIds:[…]}`.
2. Per application, in turn, the ODP document list is fetched:
   - SB-08 = `documentCode === "IDS"` with a description containing `SB08`;
   - PTO-892 = `documentCode === "892"`.
3. For each matching document, in turn: resolve the PDF download URL, download it, then POST it to `patent-asistant-apis /api/extract-references` (OCR plus LLM), which detects the form type itself.
4. The workbook `SB08_892_Data_<epoch>.xlsx` is built, uploaded to the JUNE S3 bucket, and a presigned link is created.
5. `patent-asistant-apis /postEmail` sends the notification email, including the observations. The Lambda does not send a second email for this intent (runbook step 4).
6. The JSON response `{fileUrl, errors[], customMessage?, status}` goes back to the Lambda.

Results are delivered by email. For large batches, the email can arrive after the chat response.

**Business rules** (`controllers/sb08_892.controller.js`, `services/referenceExtraction.js`, `validators/ids_api.validators.js`):

| Rule | Detail |
|---|---|
| Batch cap | 1–100 application numbers per request; more gives 400 "Only 100 application numbers allowed per request" |
| Normalisation | `,` and `/` are stripped; exactly 8 characters are required, otherwise the observation "The application number provided is invalid…" |
| Duplicates | Repeats are dropped, each recorded as "Duplicate application number." |
| Independence | Missing SB-08 and missing PTO-892 are reported separately; an application can have one without the other |
| Summary sheet (`SB-08 Reference Count Summary`) | SB-08 only. One row per SB-08 filing (mail-room date), grouped and merged per application, with the total in column B. Columns: US Patents / US Publications / Foreign References / NPL / Status. The layout follows Reference Count (TAI-170) |
| Detail sheet (`SB-08 and 892 Reference Details`) | One row per reference from both sources. Columns: Application, Mail Room Date, Source (`SB-08` / `PTO-892`), Reference Number, Reference Type |
| Sheet name | Excel's 31-character limit applies, so the detail sheet name omits "PTO-" |
| Reference number format | US: `US` + number + kind (the prefix is added if missing). Foreign: country + number + kind, with separators stripped. NPL: the citation text |
| Unidentified rows | A patent-section row with no digit in its number becomes type "Unidentified Reference", showing what OCR captured |
| Red highlighting | Failure rows are red, except "No IDS filed", which is an expected outcome |
| "No IDS" total | Shows `0`; other failures show `-` |
| Single-application message | "The total references count is N", with a "(partial)" variant when a mail-room date could not be read |
| Retries | The download-plus-extraction cycle is retried up to `API_RETRIES` times, 1 s apart |
| Extraction warnings | Logged |
| Cost | The OCR spend (`azure_api_cost`) is summed and logged per application and overall |

**User-facing messages** (verbatim; they appear in the email's *Track Task & Observation* section, so change them only with a ticket):
- `NO_SB08`: "No IDS filed for the provided application number."
- `NO_892`: "No PTO-892 document found for the provided application number."
- `INVALID_APPLICATION`: "The application number provided is invalid. Please verify and try again."
- `RETRIEVAL_FAILURE`: "An unexpected error occurred while retrieving the SB-08 or PTO-892 document. Please try again."
- `DUPLICATE_APPLICATION`: "Duplicate application number."

**Rollout gating:** the module is visible to a user when an `IpApps` row plus a tenant mapping plus a per-user mapping exist, and the Lex locale has been built. It ships as `release_stage = 'beta'`. The steps are in `docs/TAI-1181-deployment.md`.

**Timing and cost** (measured for the runbook): a 1-page PTO-892 takes about 12 s and $0.01; a 4-page SB-08 about 18 s and $0.02.

**Testing tip:** dynamic XFA SB-08 PDFs (generator output) contain no scanned text, so use real ODP scans as test fixtures.

## Related changes (September 2026)

| Commit | What |
|---|---|
| `2efb3b0` | SB-08 and 892 Reference Analysis (TAI-1181) |
| `ce79ce1` | TAI-1181/TAI-1201 application-number error message |
| `2f5cc42` | User-facing errors sent; transport errors logged |
| `d06a68d` | Multi-application xlsx upload |
| `67af64c` | S3 upload helper without a request logger |
| `fab608b` / `de52f2b` | Zip upload checksum |
| `22f8836`, `baeb800` | File-upload logging |

## Other modules (summary)

| Module | What it does | Extraction backend |
|---|---|---|
| Reference Count (`/data`) | Counts SB-08 references per mail-room date | `extractReferencesWithOcrAndLlm`: fast-api `/api/vision/extract-references` by default; `patent-asistant-apis` for tenants listed in `TENANTS_EXCEPTION` (since `0942819`, 2026-09-10) |
| Download (`/download`) | Zips the 1449, 892 and SB-08 PDFs per application | none |
| Reference Check (`/ref-check`) | Checks user-given reference numbers against SB-08 and 892 citations, including EPO family members (`patent-asistant-apis /getEpoFamilyData`) | Same routing as Reference Count |
| Citation Matrix (`/citation-matrix`) | xlsx of application families in, ZIP of per-family citation matrices out (max `CITATION_MATRIX_MAX_APPLICATIONS`, default 25) | Reuses Ref Check's `fetchRefCheckDocuments` / `extractOcrReferences`. It imports `REFERENCE_TYPE_BY_SECTION` and `UNIDENTIFIED_REFERENCE` from `services/referenceExtraction.js`, so renaming those labels changes Citation Matrix output too |
