# patent-asistant-apis — Product and features (focus areas)

## 1. SB-08 Generator (`POST /forms/SB-08`)

**Business purpose:** a practitioner lists the prior art they want to cite (patent reference numbers plus free-text non-patent literature). The service returns filled **PTO/SB/08** IDS forms, as PDF or DOCX, with an Excel reference list, ready to file. The bibliographic data (inventor surname, date, kind code) is looked up automatically.

**Entry:** Lex `Sb08Intent` ("SB-08 Generator" subtask) → `pa_lambda_functions: docgen_handler.py` → `POST {ACTION_BASE_URL}/forms/SB-08`. The same route serves AIA-82/96/123/122/83/81A/41, SB-30 and SB-30EFS (`allowedForms`).

**Flow** (`controllers/formGeneration.controller.js`):

```mermaid
sequenceDiagram
  participant L as Lambda
  participant F as formGeneration (SB-08)
  participant N as nplExtractor (Qwen)
  participant E as epo-bdss-apis
  participant D as Document processor
  participant S as S3
  L->>F: POST /forms/SB-08 {referenceNumbers, nplEntries, applicationNumbers?, docType}
  F->>F: dedupe refs ("Duplicate patent reference found"); ≤1 application number
  F->>N: extractNplCitations(nplEntries) — LLM splits free text into citations
  F->>F: refs must start with a 2-letter country code
  F->>E: POST /api/publication/bulk {references}
  E-->>F: {"<ref>": biblio | {error}}
  F->>F: classify US patent / US publication / foreign; format inventor surname (+ et al.)
  F->>F: sort; write <docket>_SB-08_References_List.xlsx
  alt docType = docx
    F->>D: fillSB08Doc(metaData, template)
  else pdf
    loop files = max(ceil(US/300), ceil(foreign/50), ceil(NPL/50))
      F->>D: fillXfaForm(SB-08 XFA template, chunk)
    end
  end
  F->>S: ZIP (xlsx + forms) → presigned URL
  F-->>L: {url, customMessage?, errorMessages[]}
```

**Business rules:**

| Rule | Detail |
|---|---|
| Application number | Optional (since #493, 2026-07-23); at most one per request |
| References or NPL required | "Reference list is required. Please provide patent references or NPL citations." |
| NPL-only mode | Allowed: the metadata call is skipped, and a form is produced with NPLs only |
| Reference format | Must start with a 2-letter country code, otherwise "Provide valid number with country code" |
| Metadata source | epo-bdss-apis `POST /api/publication/bulk` (since #430, 2026-05-25). A per-reference `{error}` → "Failed to fetch meta data". With no metadata and no NPLs, no form is produced |
| US patent vs publication | Patent if the number starts `D`/`RE`/`PP`/`H`/`AI`, or the kind is `A`/`B*`/`C*`, or the kind is S/P/T/X/RE before 2000; otherwise publication. A 10-character publication number is widened to 11 by inserting `0` after the year |
| Inventor name | First inventor's surname, formatted, plus " et al." when there are several distinct inventors (`getLastName`). PR #528 (TAI-1161) adds `utils/personNameFormatter.js` for suffixes (Jr., III…), particle surnames (de, van, St.…) and Mc/Mac/O' casing |
| Applicant fallback | The applicant name is formatted with the company `formatName` (DECISIONS N5) |
| Missing fields | "<ref> - Missing data: Inventor/Applicant Name, Country, Kind Code, Publication Date" |
| Sorting | US by date then number; foreign by country then date |
| Form capacity (PDF) | 300 US (patents + publications), 50 foreign, 50 NPL per form. Each section overflows independently, and cite numbers restart per form (TAI-134, 2026-09-14) |
| NPL per file | `us-nplcit` is reset for each file (TAI-134) |
| DOCX | A single DOCX (no capacity split), produced when patent metadata or NPLs exist |
| Output | ZIP: `<docket or app>_SB-08_References_List.xlsx`, plus `…_SB-08[_i].pdf` or `.docx` |

**Behaviour when a dependency is unavailable:**
- epo-bdss-apis: with NPLs, an NPL-only form is produced; without NPLs, no form.
- Document processor or S3: "SB-08 - Something went wrong. Please try again later".
- NPL splitting: a warning is logged (`NPL extraction failed, skipping`) and the form is produced without NPLs.

## 2. NPL citation splitting (`services/npl/nplExtractor.js`)

**Problem:** the user pastes NPL citations as one string. Commas appear inside citations (authors, journal, pages), so punctuation cannot split them; the model finds the boundaries.

**Behaviour (#526, TAI-1212):**
- An array input is trimmed and returned (no LLM). A string goes to the LLM.
- The model's output is parsed with `JSON.parse`; on a cut-off response the properly closed strings are kept.
- `max_tokens` and the input window are sized from the model context (`NPL_LLM_CONTEXT_TOKENS`, default 32768); non-Latin characters count as about 1 token.
- Input over the window (`NPL_EXTRACTION_WINDOW_CHARS`, around 24k characters) is processed in **passes**; each pass resumes after the last citation found, and a citation cut by the window is re-read in the next pass.
- Each pass is retried up to `NPL_EXTRACTION_ATTEMPTS` (3) times, each call bounded by `NPL_LLM_TIMEOUT_MS` (5 min). If every attempt fails, earlier citations are kept and the remaining text becomes one entry.
- Verified against the production model with 35, 110 and 250 citations with quoted titles (per the #526 commit message).

## 3. LLM + OCR reference extraction (`POST /api/extract-references`)

Shared ownership; the design is in `docs/llm-ocr-kt.md` and `docs/reference-extraction.md`. Consumers: **ids-api** (SB-08 & 892; Reference Count, Ref Check and Citation Matrix for tenants in `TENANTS_EXCEPTION`), and order-letter parsing through `parseModelOutput`.

### Changes in #526 (TAI-1200 / TAI-1212)

- **Quoted NPL titles:** `parseModelOutput` unescapes keys only when the object opens with an escaped key.
- **SB-08 date rows:** the harvester accepts `MM-DD-YYYY` and `M/D/YYYY`; 11-digit publications are stored as joined digits. NPL lines with more than 3 tokens before a date, or numbers under 4 digits, are rejected.
- **Section carry-over:** a row after an NPL page takes the previous page's section only if it is `usp`/`uspub`.
- **Alphanumeric cite numbers** (`A1`, `AA3`, `12a`) are recognised before a quote, a capitalised author or initials, which leaves terms like "PD-1", "CD4", "CO2" intact.
- **Substitute SB-08s:** "Examiner Signature" on sheet N of M (N < M) does not end the document (`isEndOfCitationsPage`).
- **Misfiled NPLs:** foreign rows with no digits, no country, kind or date, but with a year or quote, are rebuilt and moved to NPL.
- **Prompt rule:** a kind code after a document number is never a cite number (replay evidence: 6 of 12 good results with the rule vs 0–1 of 12 without).

Each rule change is recorded, with its alternative and reason, in `.claude/docs/DECISIONS.md` (D1–D7).

### Changes since `docs/llm-ocr-kt.md`

The guide was written at `update-llm-ocr` @ `14b74a1` (2026-08-10). Since then, on `main`:

| Topic | Now |
|---|---|
| `isJunkData` | Drops only 1–3 digit row labels, so bare patent numbers are kept |
| LLM call timeout | `FETCH_PDF_LLM_TIMEOUT_MS` (default 120 s) per page |
| Failed pages | Skipped; the row harvester (FIX-22) backfills rows from the OCR text |
| Controller size | `ocrLLM.controller.js` is 1,409 lines |
| `repairSections` tests | In `tests/unit/controllers/ocrLLM.controller.test.js` |

## 4. EPO bulk bibliographic data (overview)

`POST /bulk/references` (and `/bulk/referencesComaSeparated`) take reference lists:
- cached references return at once;
- small requests (≤ `INLINE_THRESHOLD`) run inline;
- larger ones become SQS batches, processed by the in-process worker against EPO **OPS** (Open Patent Services).

Supporting pieces (design in `services/bulkRefDownloader/task.md` and `optimization.md`):
- **multi-key rotation and rate/quota management** (`tokenManager.js`; `rateLimiter.js`; TP-207 "EPO Smart Key"), respecting hourly and weekly quotas, a rolling 60 s retrieval throttle and the self-throttling response headers;
- job state and results in Redis (`jobStore.js`, cluster-safe keys);
- status routes `/bulk/jobs[/:id[/results]]`, `/bulk/queue/health`, `/bulkRef/KeyStatus`.

Configuration is in `docs/configuration.md`. The worker keeps failed messages for retry and extends visibility while working (README *Bulk processing*).

SB-08 gets its metadata from epo-bdss-apis (#430), not from this path.
