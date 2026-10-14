# pa_lambda_functions — Auth and security (focus areas)

The FlattenPDF service doc's "Security Considerations" section covers input validation, temp-file permissions and presigned URLs.

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| Lex → LexChatbot | Invoked by Lex with the session attributes set by the chat backend (`user`, `tenantName`, `requestId`, `sessionId`) | `lambda_handler.py` |
| LexChatbot → services | Forwards `X-User` / `X-Tenant-Name` from the session | handlers |
| FlattenPDF input | `.pdf` / `.zip` only; path sanitisation; file-count and size limits; temp files `0o700` | service doc and code |
| FlattenPDF output | Presigned GET URL; default expiry 7 days (`URL_EXPIRATION_SECONDS`) | `utils.py` |
| Function URLs | FlattenPDF and GooglePat are reached through Lambda function URLs (`FLATTEN_PDF_LAMBDA_URL`) | configuration |

## Data handling

- Handler logs include intents, slots, headers and payloads, which carry user identity and client references; treat the Lambdas' CloudWatch logs as confidential.
- Keep function URLs, service base URLs and sample identities out of templates and READMEs; share them through the secrets store.
- GooglePat calls `patents.google.com` endpoints with a browser referer.
