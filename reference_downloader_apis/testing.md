# reference_downloader_apis — Testing

The README's Request Validation section explains how to add validator tests.

## Framework and commands

Jest 30 with Supertest. Thresholds (`jest.config.js`): lines 95, statements 95, functions 90, branches 85.

```bash
npm test
npm run test:coverage   # CI and the pre-push hook
npm run lint
```

## Coverage map (62 test files)

| Area | Examples |
|---|---|
| Controllers | `tests/unit/controllers/{request,upload,patent,pctRequest,auth}*.test.js` |
| Queue processors and worker | `tests/unit/queues/*`, `tests/unit/worker/{index,sqsPoller}.test.js` |
| Services | epo, wasabi, flattenPDF, generateZip, jobCompletion, requestCompletion, slaEmail, emailTransporter, queueProducer, secretsManagerClient |
| Utils | SLA, `validate_Ref`, `isPDFEnglish`, `normalizePdfPages`, `pdfHasText`, `pdfValidator`, notifications, `updateIPAssistant`, invalidReasons, logger and CloudWatch transport |
| Validators | `tests/unit/validators/*` plus `tests/integration/routes/validation.test.js` (asserts 400 and that the controller is not called) |
| Root | `app`, `db`, `loadEnv`, `setup`, `swagger`, eslint/jest config |
| Credentials | `tests/unit/config/dbCredentials/*` |

External services, Postgres and AWS are mocked in the suite. Use public patent references for manual testing.
