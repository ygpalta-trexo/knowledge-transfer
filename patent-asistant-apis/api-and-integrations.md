# patent-asistant-apis — API and integrations (focus areas)

The full route inventory is in `patent-asistant-apis: docs/api.md` and `docs/endpoints_documentation.md`. There is no global prefix; business routes need `X-Tenant-Name` and `X-User` (see `docs/architecture.md`). This file pins down the contracts that other repos depend on.

## Inbound contracts

### `POST /forms/SB-08` (from the LexChatbot `docgen_handler.py`)

| Body field | Meaning |
|---|---|
| `referenceNumbers` | Patent references (country code + number + optional kind), deduplicated |
| `nplEntries` | Free-text NPL string (empty when the user answered "No"), or an already-split array |
| `applicationNumbers` | Optional; at most one |
| `docType` | `pdf` (default) or `docx` |

Response: `{ url, customMessage?, errorMessages[] }`, where `url` is a presigned ZIP link. Per-reference problems are reported in `errorMessages` with HTTP 200.

### `POST /api/extract-references` (from ids-api)

- Multipart; the field name is **`files`** (multer `.array("files")`); the first file is processed.
- Optional form fields `system_prompt`, `user_message_template`, `model_name` and `max_tokens` override the server configuration for that request (used for prompt iteration).
- 200 → `{ azure_api_cost: number, data: { usPatentRows[], usPublicationRows[], foreignDocumentRows[], nplDocumentRows[] }, warnings: string[] }`. Every field is a string (never `null`); NPL entries are strings; patent rows carry `number`, `kindCode`, `issueDate`, `inventorName`, and `countryCode` for foreign rows.
- 400 no file · 422 schema check ("LLM returned unexpected data structure") · 500 "Processing failed: …".

ids-api renders exactly these keys, so coordinate any change with ids-api (see [../ids-api/api-and-integrations.md](../ids-api/api-and-integrations.md)).

### `POST /postEmail` (from ids-api, reference_downloader_apis, the Lambda)

Body `{status?, output, errorMessages, requestId, hasObservations?, customMessage?}`, with `X-Tenant-Name` / `X-User`. Sends the task-completion email (`chat.controller.js postEmail`).

### `POST /usFilingPostCollab`, `GET /getEpoFamilyData/:ref`

Used by reference_downloader_apis (PCT submit) and ids-api (Ref Check family data).

## Outbound integrations

| System | Config names | Used by | Limits |
|---|---|---|---|
| **epo-bdss-apis** `POST /api/publication/bulk` | `EPO_BDSS_URL` | SB-08 generation; also `utils/appgenUtility.js`, `services/odpApi/patent.js`, `controllers/bibData.controller.js` | |
| **Qwen LLM** (self-hosted, OpenAI-compatible; the `openai` SDK is the client) | `QWEN_LLM_BASE_URL`, `QWEN_LLM_API_KEY`, `QWEN_LLM_MODEL`, `FETCH_PDF_DATA_MODEL`, `FETCH_PDF_DATA_MAX_TOKEN`, `FETCH_PDF_LLM_TIMEOUT_MS`, `FETCH_PDF_ENABLE_PAGEBREAK_FIXES`, `NPL_LLM_CONTEXT_TOKENS`, `NPL_LLM_TIMEOUT_MS`, `NPL_EXTRACTION_WINDOW_CHARS`, `NPL_EXTRACTION_ATTEMPTS` | Extraction (per page); NPL splitting | Extraction: 120 s per page. NPL: 5 min per call, 3 attempts per pass |
| **Azure Document Intelligence** (`prebuilt-layout`) | `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT`, `AZURE_DOCUMENT_INTELLIGENCE_KEY` | Extraction OCR, one page at a time; cost computed locally from a price table | SDK poller |
| Prompts | `FETCH_PDF_DATA_SYSTEM_PROMPT_FILE` / `FETCH_PDF_DATA_SYSTEM_PROMPT`, `FETCH_PDF_DATA_USER_MESSAGE_FILE` / `FETCH_PDF_DATA_USER_MESSAGE`; the NPL prompt file `prompts/npl_extraction_system_prompt.txt` | Resolution order: request body → `*_FILE` → inline env | — |
| Document processor | `DOCUMENT_PROCESSOR_URL`, `DOCUMENT_PROCESSOR_V` | SB-08 XFA read/fill (`getXfaData`, `fillXfaForm`) and DOCX fill (`fillSB08Doc`) | |
| S3 | `IP_ASSISTANT_BUCKET_NAME` (templates) and the upload helpers | Templates in, ZIP out with a presigned URL | |
| EPO OPS (bulk) | `EPO_API_URL`, `EPO_AUTH_URL`, `EPO_BULK_URL`, `EPO_CLIENT_ID`, `EPO_CLIENT_SECRET`, `EPO_KEYS`, `EPO_HTTP_TIMEOUT_MS`, plus quota and rate settings (`SAFE_*`, `PER_KEY_CONCURRENCY`, `GLOBAL_MAX_PARALLEL_BATCHES`, …) | Bulk references | See `docs/configuration.md` |
| SQS / Redis | `SQS_QUEUE_URL`, `SQS_VISIBILITY_TIMEOUT`, `SQS_WAIT_TIME_SECONDS`, `SQS_MAX_NUMBER_OF_MESSAGES`, `REDIS_HOST`, `REDIS_PORT`, `REDIS_TLS`, `BULK_CACHE_TTL_SECONDS`, `INLINE_THRESHOLD` | Bulk jobs | See the README *Bulk processing* |
