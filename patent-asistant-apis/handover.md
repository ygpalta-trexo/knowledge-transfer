# patent-asistant-apis — Handover (focus areas)

As of 2026-09-18 (`origin/main` at `901618a`).

## Recent delivered work

| Date | Ticket / PR | What |
|---|---|---|
| 2026-09-17 | #526 (TAI-1212 / TAI-1200 / TAI-134) | SB-08 NPL citation limits (per-section form capacity, cite numbers restarting per form) and NPL splitting in passes; extraction fixes for quoted NPL titles, date rows, cite numbers, substitute SB-08s and misfiled NPLs |
| 2026-09-03 | #514 | LLM-OCR reference extraction (`/api/extract-references`) merged to `main` |
| 2026-07-23 | #493 | SB-08 application number optional |
| 2026-05-25 | #430 | SB-08 metadata from epo-bdss-apis |

**In review:** PR #528 (TAI-1161), inventor surname formatting with suffixes and particle surnames; its decisions and test cases are in `.claude/docs/DECISIONS.md` (N1–N6) and the TAI-1161 trace.

## External dependencies of the focus areas

- **epo-bdss-apis** (`EPO_BDSS_URL`): SB-08 metadata. See [../epo-bdss-apis/README.md](../epo-bdss-apis/README.md).
- **Self-hosted Qwen LLM server** and **Azure Document Intelligence**.
- **Document processor** (`DOCUMENT_PROCESSOR_URL`): SB-08 XFA/DOCX filling.
- **ids-api** (consumer of `/api/extract-references` and `/postEmail`); **pa_lambda_functions** LexChatbot (`Sb08Intent`, `Sb08And892Intent`).
- EPO bulk: EPO OPS keys; SQS queue; Redis (cluster-safe keys).

## Ownership

| Area | Owner |
|---|---|
| SB-08 Generator, NPL splitting | |
| LLM-OCR extraction pipeline (shared; see DECISIONS D1–D7 for rules by other developers) | |
| EPO bulk / key rotation | |
| Qwen LLM host | |
| PM sign-off for TAI-1161 | |
