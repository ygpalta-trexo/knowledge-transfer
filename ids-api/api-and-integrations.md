# ids-api — API and integrations

Routes are mounted at `/ids-api` (`app.js`). All read the body, including on `GET`, because the Lambda sends `GET` with a JSON body.

## Common behaviour

- Headers: `X-User` (valid JSON) and `X-Tenant-Name` are required, otherwise 400. `request-id` is optional.
- Validation: Joi via `validate(schema)`; failures give 400 through `middlewares/errorHandler.js`.
- Unknown route → 404; wrong method → 405.
- `GET /health` → 200.

## Routes

| Method | Route | Body | Success response |
|---|---|---|---|
| GET | `/ids-api/sb08-892` | `{applicationIds: string[1..100]}` | 200 `{fileUrl, errors[], customMessage? (single app), status:"Success"}`; 500 `{errors[]}` when nothing could be extracted or uploaded |
| GET | `/ids-api/data` | `{applicationIds: string[≥1]}` | Reference Count workbook link |
| GET | `/ids-api/download` | same | ZIP link |
| GET | `/ids-api/ref-check` | `{applicationIds[], referenceNumbers: string}` | Ref Check workbook link |
| POST | `/ids-api/citation-matrix` | multipart xlsx (`middlewares/excelUpload.js`) | ZIP link |

The Lambda reads `fileUrl`, `errors` and `customMessage` from the response.

## Outbound integrations

| System | Env names | Call | Notes |
|---|---|---|---|
| USPTO Open Data Portal | `API_URL`, `API_TOKEN` (`X-API-KEY`), `API_RETRIES` | `GET {API_URL}/{app}/documents`; `GET <downloadUrl>` (following the redirect to the final URL); then `GET` the PDF | The download-URL fetch retries `API_RETRIES` times, 2 s apart |
| **patent-asistant-apis: extract** | `PATENT_ASSISTANT_URL`, `EXTRACT_REFERENCES_TIMEOUT_MS` | `POST /api/extract-references`, multipart field `files` (multer `.array("files")`), headers `X-Tenant-Name`, `X-User`, `request-id` | 900,000 ms default. Returns `{data:{usPatentRows[], usPublicationRows[], foreignDocumentRows[], nplDocumentRows[]}, warnings[], azure_api_cost}`. NPL entries are strings; patent entries are objects with `number` / `patentNumber` / `foreignDocumentNumber`, `countryCode`, `kindCode`, `inventorName` |
| patent-asistant-apis: email | `PATENT_ASSISTANT_URL` | `POST /postEmail {status, output, errorMessages, requestId, hasObservations, customMessage?}` | |
| patent-asistant-apis: EPO family | `PATENT_ASSISTANT_URL` | `GET /getEpoFamilyData/:ref` | Ref Check |
| fast-api | `FAST_API_URL`, `FETCH_PDF_DATA_MODEL`, `FETCH_PDF_DATA_OCR`, `FETCH_PDF_DATA_USER_MESSAGE`, `FETCH_PDF_DATA_SYSTEM_PROMPT`, `FETCH_PDF_DATA_MAX_TOKEN`, `TENANTS_EXCEPTION` | `POST /api/vision/extract-references` (field `file`); `POST /api/utilities/azure-ocr/ids-json?form=` | Reference Count, Ref Check, Citation Matrix |
| AWS S3 | `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `JUNE_BUCKET_NAME`, `AWS_S3_URL_EXPIRATION` | PutObject + presigned GetObject | |
| CloudWatch | `AWS_ACCESS_KEY_SECRET`, `AWS_SESSION_TOKEN`, `LOG_LEVEL`, `LOG_RETENTION_DURATION_IN_DAYS` | Transport | Same transport family as the other Trexo services |

## The extract-references contract (shared with patent-asistant-apis)

Everything SB-08/892 renders depends on:
- the four section keys;
- the per-row fields above;
- NPL entries being plain strings;
- `azure_api_cost` being a number.

A change to this shape in `patent-asistant-apis` changes the SB-08/892 workbook, so review both repos together when either side changes.
