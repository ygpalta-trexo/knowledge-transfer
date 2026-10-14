# pa_lambda_functions — API and integrations (focus areas)

## FlattenPDF request contract

The request is an HTTP POST to the function URL; the JSON body is read from `event["body"]` (`FlattenPDF/lambda_function/lambda_function.py`).

| Field | Default | Limits | Notes |
|---|---|---|---|
| `file_keys` | `[]` | ≤ `max_files` (15 by default, 1–20) | `.pdf` or `.zip` |
| `input_bucket_name` | `DEFAULT_BUCKET` env | — | |
| `output_bucket_name` | — | — | |
| `output_file_name` | — | — | Used by the reference downloader to write `<ref>/<file>` |
| `dpi` | 150 | ≤ 150 (`max_dpi`) | |
| `jpeg_quality` | 35 | ≤ 35 (`max_quality`) | |
| `max_pages` | 250 | 50–2,000 | |
| `page_size` | `"letter"` | `letter`, `a4`, `legal`, `source`, or `"8.5x11in"`-style | Selects the output page size |
| `grayscale` | `true` | — | |
| `force_mode` | `"auto"` | `auto` / `scanned` / `text` | |
| `use_wasabi` | `false` | — | |
| `request_id` | timestamp | — | Logging |

Response: `{statusCode, body: {message: string[], s3Url, processed_file_count}}`. Partial success returns 200 with per-file messages; errors return 400, 413, 422 or 500 with `errors[]` (see the service doc).

### Callers

| Caller | Sends |
|---|---|
| LexChatbot `FlattenPdfIntent` | `dpi: 216`, `jpeg_quality: 95` (capped at 150 / 35), `max_pages: FLATTEN_PDF_MAX_PAGES` (1,000), AWS S3 buckets |
| `reference_downloader_apis` `flattenUploadProcessor` | `dpi: 150`, `jpeg_quality: 35`, `max_pages: 1500`, `use_wasabi: true`, headers `X-User: admin`, `X-Tenant-Name: default` |

### Current values in code

Where `FlattenPDF/PDF_PROCESSING_SERVICE_DOCUMENTATION.md` gives other figures, use these:

| Setting | Value |
|---|---|
| `max_pages` | Default 250, range 50–2,000 (TAI-856) |
| DPI / quality | Ceilings 150 / 35; request defaults 150 / 35 |
| Bulk limits | Defaults 15 files / 50 MB; upper bounds 20 / 100 MB |
| Environment variables | `ACCESS_KEY_ID`, `SECRET_ACCESS_KEY`, `WASABI_ACCESS_KEY`, `WASABI_SECRET_KEY`, `S3_REGION`, `S3_BUCKET_NAME`, `DEFAULT_BUCKET`, `URL_EXPIRATION_SECONDS`, `FUNCTION_LOG_LEVEL` |
| `FUNCTION_LOG_LEVEL` | Code default `DEBUG`; set `INFO` in production |
| Runtime | Python 3.12 (bundler target) |

## LexChatbot outbound calls (focus intents)

| Intent | Method and URL (env) | Payload | Timeout |
|---|---|---|---|
| `Sb08Intent` | `POST {ACTION_BASE_URL}/forms/SB-08` | Form data: `referenceNumbers`, `nplEntries`, `applicationNumbers`, `docType` | 900 s |
| `AIA81ADocgenIntent` / `AIA83DocgenIntent` | `POST {ACTION_BASE_URL}/forms/AIA-81A` or `/forms/AIA-83` | JSON built from the parsed `form_details` slot | 900 s |
| `Sb08And892Intent` (plus Reference Count / Download / Ref Check) | `GET {IDS_API_BASE_URL}/ids-api/<route>` | JSON body `{applicationIds, referenceNumbers?}` | 900 s |
| `FlattenPdfIntent` | `POST {FLATTEN_PDF_LAMBDA_URL}` | See above | 900 s |
| `EPOBiblioIntent` / `GlobalPatentIntent` | `POST` to patent-asistant-apis (`ACTION_BASE_URL`) | Application or reference list | 900 s |
| Email (most intents) | patent-asistant-apis `/postEmail` via `utils.send_email` | Status, output URL, errors, requestId | — |

Headers: `X-User` (the session user JSON), `X-Tenant-Name` (the session `tenantName`), `sessionId` and `request-id`.

## Config names read by code

- **LexChatbot:** `ACTION_BASE_URL`, `IDS_API_BASE_URL`, `REFERENCE_BASE_URL`, `FASTAPI_BASE_URL`, `FLATTEN_PDF_LAMBDA_URL`, `FLATTEN_PDF_MAX_PAGES`, `S3_BUCKET_NAME`, `S3_INPUT_BUCKET_NAME`, `S3_REGION`, `AWS_ACCESS_KEY_S3_ID`, `AWS_SECRET_ACCESS_KEY_S3`, `DATABASE_URL` (plus per-tenant `DATABASE_URL_<tenant>`), `USPTO_ODS_API_KEY`, `USPTO_ODS_BASE_URL`, `USPTO_TSDR_API_KEYS`.
- **FlattenPDF:** see [Current values in code](#current-values-in-code).
