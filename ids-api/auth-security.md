# ids-api — Auth and security

## Mechanisms

| Concern | Mechanism | Where |
|---|---|---|
| Identity | The service reads `X-User` (valid JSON) and `X-Tenant-Name` from its callers and forwards them downstream | `app.js` |
| Entitlement | Module access (`IpApps` + tenant + user mappings) is checked upstream, in `patent-asistant-apis` / the chat layer (`checkIntentSubscription`) | `docs/TAI-1181-deployment.md` |
| Hardening | `helmet`, `hpp`, `compression`; Joi validation; SB-08/892 batch cap of 100 | `app.js`, validators |
| Outbound secrets | ODP API key and AWS keys from env | configuration |
| Deliverables | S3 objects in the JUNE bucket with a time-limited presigned URL (`AWS_S3_URL_EXPIRATION`) | `utils/filesCloud.js` |
| Container | `.dockerignore` excludes `.env*` | repo |

## Data handling

Application numbers, reference lists and the generated workbooks are client work product. Treat presigned links, request logs and deliverables as confidential.
