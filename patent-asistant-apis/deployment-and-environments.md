# patent-asistant-apis — Deployment and environments (focus areas)

The general build and run details are in `patent-asistant-apis: README.md` → *Operations*: Dockerfile on Node 24 Alpine, `PORT=8085` in the image, 8006 locally.

## Where the focus-area code lives

- `/api/extract-references` (the LLM-OCR line) was merged to `main` in #514 (2026-09-03); the #526 changes followed on 2026-09-17. Builds for environments that serve ids-api's SB-08 & 892 need both.
- `update-llm-ocr` and `merge/main-into-update-llm-ocr` are earlier integration branches, superseded by `main`.
- Changes are promoted from `main` to the environment branches (`dev`, `qa`, `uat`, `staging`, `release`).

## Configuration to set per environment

| Setting | Why |
|---|---|
| `FETCH_PDF_DATA_SYSTEM_PROMPT_FILE=prompts/fetch_pdf_system_prompt.txt` (+ `FETCH_PDF_DATA_USER_MESSAGE_FILE`) | Makes the service use the version-controlled prompt files, which include the PTO-892 rules and the #526 changes (`docs/llm-ocr-kt.md` §6) |
| `QWEN_LLM_BASE_URL`, `QWEN_LLM_API_KEY`, `QWEN_LLM_MODEL` | The LLM client is created at startup, so these must be present when the process starts |
| `FETCH_PDF_LLM_TIMEOUT_MS`, `FETCH_PDF_DATA_MAX_TOKEN`, `FETCH_PDF_ENABLE_PAGEBREAK_FIXES` | Extraction behaviour and limits |
| `NPL_LLM_CONTEXT_TOKENS`, `NPL_LLM_TIMEOUT_MS`, `NPL_EXTRACTION_WINDOW_CHARS`, `NPL_EXTRACTION_ATTEMPTS` | NPL splitting (#526) |
| `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT`, `AZURE_DOCUMENT_INTELLIGENCE_KEY` | OCR |
| `EPO_BDSS_URL` | SB-08 metadata; point it at an epo-bdss-apis build that includes TAI-543 (US 10/11-digit publication handling) |
| `DOCUMENT_PROCESSOR_URL`, `DOCUMENT_PROCESSOR_V`, `IP_ASSISTANT_BUCKET_NAME` | SB-08 PDF/DOCX filling and templates |

`loadEnv.js` loads the `.env*` files with `override: true`, so values in those files take precedence over shell-exported variables (`docs/llm-ocr-kt.md` §7).

## Rollout notes

- SB-08 & 892 order: this service → ids-api → DB rows → Lambda → Lex build → Redis flush (the ids-api runbook).
- The TAI-1161 name formatting changes no data or schema, so rolling back is a code revert (per its trace).
