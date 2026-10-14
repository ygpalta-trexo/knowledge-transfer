# reference_downloader_apis — Auth and security

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| UI traffic | Authenticated at `trexo-apigateway`: JWT plus the `COLLABORATE` app check | `trexo-apigateway: routes/tenantAPI.router.js` |
| Service callers | The Lex Lambda and `patent-asistant-apis` call the service internally and pass `X-User` / `X-Tenant-Name` | callers |
| Identity in the service | Read from `X-User` / `X-Tenant-Name`; stored on each request | `app.js` |
| Team views | The Collaboration Space queue (`GET /requests`, `GET /pctrequests`) lists requests from all tenants, for the internal team that works it | `requestController.getRequests` |
| Input validation | Joi on input-taking routes, with `stripUnknown`; header JSON parsing is guarded | `middleware/validate.js`, `validators/*` |
| File validation | Buffer MIME detection (PDF, Excel, XML, TXT allow-list); ZIP entry-count limit (500) and zip-slip checks for `.xlsx`; `validatePdf` loads each PDF with pdf-lib | `middleware/fileValidation.js`, `utils/pdfValidator.js` |
| Download links | S3 presigned URLs valid for `REQUEST_ZIP_EXPIRY` seconds (`.env.sample` suggests 7 days) | `services/wasabiService.js` |
| Secrets | DB credentials from env or Secrets Manager; Wasabi, S3, CloudWatch and SMTP credentials from env | `config/` |
| Container | Non-root user; `.env*` excluded by `.dockerignore` | `Dockerfile` |
| CORS | `*.<company-domain>`, localhost and server-to-server requests, with credentials | `app.js` |

## Data handling

- The reference PDFs are public patent documents.
- **Which references a client is citing, and for which matter, is confidential work product.** Treat request ids, ZIP links, presigned URLs and request lists as confidential.
- Requester details (name, email) are stored in `Requests.user` / `PctRequest.user` as the `X-User` JSON received.
- `PUT /patent/invalid` changes a cache row that every tenant shares; use it only for references that are invalid for everyone.
