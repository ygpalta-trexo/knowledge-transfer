# pa_lambda_functions — Handover (focus areas)

As of 2026-09-18 (`main` at `f2580d7`).

## Handed-over areas

- **FlattenPDF (Embedded Font Removal):** `FlattenPDF/lambda_function/*`, its two docs and the layer build notes. History in [product-and-features.md](product-and-features.md#1-embedded-font-removal-flattenpdf).
- **Lex intents:** `Sb08Intent`, `Sb08And892Intent`, `AIA81ADocgenIntent`, `AIA83DocgenIntent` (handlers `docgen_handler.py`, `ids_count_handler.py`), and the `Sb08And892Intent.txt` utterances. `EPOBiblioIntent` / `GlobalPatentIntent` history (that logic now lives in patent-asistant-apis).

## External dependencies

- **patent-asistant-apis** (`/forms/*`, `/postEmail`, bibliographic data), **ids-api** (`/ids-api/*`), **reference_downloader_apis** (a caller of FlattenPDF, and called by the chat), **fastapi** (the GooglePat caller).
- AWS Lex V2 (console-managed), S3 and Wasabi buckets, the tenant DBs (task table), Lambda layers (Ghostscript, poppler).

## Ownership

| Area | Owner |
|---|---|
| FlattenPDF | |
| LexChatbot SB-08 / SB-08 & 892 / AIA-81A / AIA-83 intents | |
| Lex bot configuration and builds (AWS console) | |
| Lambda deployments | |
