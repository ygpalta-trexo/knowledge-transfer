# pa_lambda_functions — Deployment and environments

## How the Lambdas are built and deployed

- `LexChatbot/lambda_bundler.py` (and the FlattenPDF and GooglePat equivalents) builds `lambda_function_bundle_<timestamp>.zip` and a separate dependency **layer** zip for Python 3.12, using a mamba/conda environment. Uploading and alias moves are done in the AWS console. Record the branch and bundle each time you deploy an environment.
- **FlattenPDF layers:** Ghostscript (`FlattenPDF/ghostscript-layer.zip`) and poppler-utils (build steps in `FlattenPDF/Readme.md`). Attach both; without poppler, detection uses PyMuPDF.
- **LexChatbot DB migrations:** Alembic (`LexChatbot/alembic.ini`, `alembic/versions/`) against each tenant DB. `tenant_setup.py` and `db_verification.py` are helper scripts.
- **Lex bot:** intents, slots and builds are managed in the AWS console. A locale build ships the whole DRAFT of the shared bot, so coordinate builds with other developers (ids-api TAI-1181 runbook, step 5). **Deploy the Lambda before building the bot**, so new intents reach their handlers.

## Branches

`main` is the integration branch; the environment branches are `dev`, `qa`, `uat` and `release`. Build each environment's bundle from its branch after promoting changes from `main`.

## Environment-dependent settings for the focus areas

| Setting | Where | Note |
|---|---|---|
| `IDS_API_BASE_URL` | LexChatbot | An ids-api build that serves `/ids-api/sb08-892` |
| `ACTION_BASE_URL` | LexChatbot | patent-asistant-apis (`/forms/SB-08`, `/forms/AIA-81A`, `/forms/AIA-83`) |
| `FLATTEN_PDF_LAMBDA_URL`, `FLATTEN_PDF_MAX_PAGES` | LexChatbot | |
| `S3_INPUT_BUCKET_NAME`, `S3_BUCKET_NAME` | LexChatbot → FlattenPDF | Input and output buckets for chat uploads |
| `ACCESS_KEY_ID`, `SECRET_ACCESS_KEY`, `WASABI_ACCESS_KEY`, `WASABI_SECRET_KEY`, `S3_REGION`, `S3_BUCKET_NAME`, `DEFAULT_BUCKET`, `URL_EXPIRATION_SECONDS`, `FUNCTION_LOG_LEVEL` | FlattenPDF | Set `FUNCTION_LOG_LEVEL=INFO` in production |
| Lambda memory / timeout | FlattenPDF | The service doc recommends 1,024 MB and a 15-minute timeout |
