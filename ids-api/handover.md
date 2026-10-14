# ids-api — Handover

As of 2026-09-18.

## Area being handed over

The **SB-08 and 892 Reference Analysis** module (TAI-1181, TAI-1201) and the upload and logging changes around it:
- `controllers/sb08_892.controller.js`;
- `services/referenceExtraction.js`;
- `services/patent-assistant/extract_references.js`;
- the `sb08And892BodySchema` in `validators/ids_api.validators.js`;
- the tests `tests/integration/controllers/sb08_892.controller.test.js`, `tests/unit/services/referenceExtraction.test.js` and the validator tests.

The feature was merged on 2026-09-03 (`2efb3b0`), with follow-ups on 2026-09-10/11 (`ce79ce1`, `2f5cc42`, `d06a68d`, `67af64c`). The rollout runbook is `docs/TAI-1181-deployment.md`.

## Tests

- Framework: Jest 30 with Supertest; layout in `ids-api: README.md`. Jenkins runs unit, integration, API, e2e and coverage.
- SB-08/892 coverage: controller integration test, `referenceExtraction` unit test, validator tests (batch cap, messages). The branch run recorded in the runbook: 26 suites / 161 tests passing.

## External dependencies

- `patent-asistant-apis`: `/api/extract-references`, `/postEmail`, `/getEpoFamilyData`.
- fast-api (`FAST_API_URL`), for the other modules.
- USPTO ODP (`API_URL`); AWS S3 (JUNE bucket); CloudWatch.
- `pa_lambda_functions` LexChatbot (`IDS_API_BASE_URL`).
- AWS Lex bot (intent `Sb08And892Intent`, slot `application_nums`); Redis cache in the chat layer.
- Superadmin DB (`IpApps`, `IpAppsTenantMapping`) and tenant DBs (`IpAppsUserMapping`).

## Ownership

| Area | Owner |
|---|---|
| SB-08/892 module | |
| Reference Count, Download, Ref Check, Citation Matrix | |
| LexChatbot Lambda and Lex bot | |
| Jenkins / deploys | |
| BA for TAI-1181 sign-offs | |
