# patent-asistant-apis — Auth and security (focus areas)

The service-wide trust model (request context, `X-User` / `X-Tenant-Name`, the internal tenant-setup endpoint, admin checks) is in `patent-asistant-apis: docs/architecture.md` → *trust boundaries*.

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| Caller identity | `X-User` / `X-Tenant-Name` headers from the calling services, as in the sibling services | `routes/assistant.router.js` |
| Upload validation | `uploadMiddleware`: multer memory storage plus a PDF MIME check | `middleware/upload.js` |
| Prompt construction | OCR text is inserted into the prompt with a function replacer, so `$&`-style sequences in OCR text are kept literally | `docs/llm-ocr-kt.md` §5.3 |
| Model output | Treated as untrusted: parsed, repaired, schema-checked and reclassified by deterministic rules | pipeline design |

## Data handling

- Reference lists, NPL citations and uploaded prosecution documents are client work product. Keep sample documents and client spreadsheets out of git.
- The extraction logs OCR text and parsed data at debug-friendly levels; set `LOG_LEVEL` per environment accordingly.
- Share LLM, OCR and EPO keys only through the secrets store.
