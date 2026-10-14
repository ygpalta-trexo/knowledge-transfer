# pa_lambda_functions — Product and features (focus areas)

## 1. Embedded Font Removal (FlattenPDF)

**Business problem:** USPTO filings (for example IDS reference copies) must not carry embedded fonts, and should be uniform US-Letter pages. Patent PDFs from EPO, KIPO and Google Patents embed fonts, and older USPTO scans carry invisible OCR text layers. FlattenPDF turns every page into pixels (or strips the overlay) so the output has zero fonts, in grayscale, at 8.5×11 in.

**Who uses it:**
- **Chat users:** Lex `FlattenPdfIntent` ("Embedded Font Removal", task IDS). The user uploads PDFs or a ZIP; the result is a presigned link, plus an email sent by the Lambda.
- **`reference_downloader_apis`:** flattens every downloaded or uploaded reference PDF that has a text layer before it goes into the reference cache (`queues/flattenUploadProcessor.js`).

**How it decides** (details in `FlattenPDF/PDF_PROCESSING_SERVICE_DOCUMENTATION.md`):
1. If the PDF has no embedded fonts that need processing, it is returned as-is.
2. **Each page** is classified. It is `scanned` when its largest image covers ≥ 70% of the page **and** it draws no visible text (PyMuPDF `get_texttrace`; invisible render modes 3 and 7 do not count). Otherwise it is `text`.
3. `text` pages: Ghostscript rasterises to JPEG, then img2pdf repacks losslessly, so no fonts remain.
4. `scanned` pages: one Ghostscript pass with `-dFILTERTEXT -dFILTERVECTOR` drops the OCR overlay; pikepdf fixes the page boxes.
5. Mixed documents run both pipelines on their own page subsets (`-sPageList`) concurrently, and pikepdf interleaves the pages back into source order.
6. Any uncertainty resolves to `text`, the safe choice. `force_mode` overrides the choice.

**History:**

| Date | Commit | Change |
|---|---|---|
| 2026-02-27 | `1f7ddd2` (TAI-856) | Max page limit raised to 2,000 |
| 2026-06-25 | `f0151cf` | New embedded-font-removal processing: two Ghostscript pipelines, img2pdf repack, poppler-based detection, US Letter plus grayscale |
| 2026-07-08 | `0390765` | Per-page routing: each page is classified on its own, so mixed documents keep their visible text on text pages |
| 2026-07-08 / 07-23 | `8020b4d`, `e513bfa` | Lowercase output extensions; page-scanning update |
| 2026-07-24 | `cc197ea` | Docs updated |

**Limits** (from code):

| Rule | Value |
|---|---|
| Pages per PDF | Request `max_pages` (default 250), within 50–2,000. The chat sends `FLATTEN_PDF_MAX_PAGES` (default 1,000); the reference downloader sends 1,500 |
| DPI / JPEG quality | Ceilings of 150 DPI / quality 35; higher requests are capped |
| Files per request | 15 by default (1–20); total size 50 MB by default (10–100) |
| Inputs | `.pdf`, or `.zip` of PDFs |
| Output | A single processed PDF, or a ZIP for several; a presigned URL valid for `URL_EXPIRATION_SECONDS` (default 7 days) |
| Output extension | Lowercase |

**Notes:**
- Dynamic XFA PDFs (for example generated SB-08s) display an "Adobe Reader required" placeholder page in most renderers, so flattening yields that page.
- For very large documents, the service doc recommends a 15-minute Lambda timeout.

## 2. Lex intents (handed over)

All intents enter through `LexChatbot/lambda_function/lambda_handler.py` (see [architecture.md](architecture.md)).

| Intent (task → subtask) | Handler | Downstream | Recent work |
|---|---|---|---|
| **`Sb08Intent`** (IDS → "SB-08 Generator") | `docgen_handler.handle_docgen_intent` | `POST {ACTION_BASE_URL}/forms/SB-08` (patent-asistant-apis), form-encoded: `referenceNumbers` ← slot `patent_numbers`, `nplEntries` ← `npl_references`, `applicationNumbers` ← `application_nums` (optional), `docType` ← `DocType` | TAI-339 docType (2025-09); optional application number (2026-08-10); handling of null or unfilled slots (`8c9049a` 08-27, `20a7d86` 09-09) |
| **`Sb08And892Intent`** (IDS → "SB-08 and 892") | `ids_count_handler` | `GET {IDS_API_BASE_URL}/ids-api/sb08-892` with a JSON body `{applicationIds}`, timeout 900 s | TAI-1181 routing (`fa09485`, 2026-09-03); utterances in `LexChatbot/Intents/Sb08And892Intent.txt` |
| **`AIA81ADocgenIntent`** (Document Generation → "PoA/Revocation of PoA/Correspondence Add Change") | `docgen_handler` | `POST /forms/AIA-81A`; `form_details` parsed from a JSON-like string slot (Python `None`/`True` converted to JSON), practitioners, signatory | TAI-473 (2025-08) |
| **`AIA83DocgenIntent`** ("Attorney Withdrawal & Address Change Request") | `docgen_handler` | `POST /forms/AIA-83` | TAI-221 (2025-09) |
| **`FlattenPdfIntent`** (IDS → "Embedded Font Removal") | `flatten_pdf_handler` | `POST {FLATTEN_PDF_LAMBDA_URL}` `{input_bucket_name: S3_INPUT_BUCKET_NAME, output_bucket_name: S3_BUCKET_NAME, file_keys ← slot files, dpi 216, jpeg_quality 95, use_wasabi false, max_pages}` | The Lambda behind it |
| `EPOBiblioIntent` / `GlobalPatentIntent` (Bibliographic → EP / Global Patent) | `patent_epo_bib_handler` | `POST` to patent-asistant-apis (the logic moved there in TAI-1092, 2026-06-10) | TAI-725 Excel ordering; TP-249 Bulk API → BDSS and familyDates (2026-06-01/02) |

**Chat responses and email:**
- **SB-08 Generator:** success shows `TASK_COMPLETED_MESSAGE` with the link and patent-asistant-apis' `customMessage`. The intent is on the no-email list, so the chat reply carries the result.
- **SB-08 & 892:** ids-api sends its own email (via `/postEmail`), so the intent is on the Lambda's no-email list (TAI-1181 runbook).
- **Flatten:** the Lambda emails the result itself.

**Lex configuration:** intents, slots, slot types and bot builds are managed in the AWS console (bot and aliases per environment). The `Intents/*.txt` files are utterance lists for reference.
