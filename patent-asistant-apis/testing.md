# patent-asistant-apis — Testing (focus areas)

The general setup is in `patent-asistant-apis: docs/testing.md` and the README: Jest `unit` and `integration` projects, Testcontainers Postgres 15 and Redis 6, with an 85% global coverage threshold in `jest.config.js`.

## Suites for the focus areas (on `origin/main`)

| Suite | Covers |
|---|---|
| `tests/unit/controllers/ocrLLM.controller.test.js` | Extraction orchestration (mocked services), `repairSections` rules, the #526 changes |
| `tests/unit/services/openai-llm.test.js` | The `parseModelOutput` repair pipeline. Read it before editing the parser: each case is a real model output |
| `tests/unit/services/ocrHelper.test.js` | OCR filter (`filterOcrContent`, `isJunkData`) |
| `tests/unit/controllers/nplPatentRowGuard.test.js` | Keeps patent rows out of the NPL rescue |
| `tests/unit/services/nplExtractor.test.js` | NPL splitting: passes, truncation, retries |
| `tests/unit/controllers/formGeneration.controller.test.js` | Form generation, including SB-08 |
| `tests/unit/utils/personNameFormatter.test.js` | On the TAI-1161 branch (PR #528): 80 table-driven cases from the ticket's lists |

When evaluating a change, compare the failing set before and after the change rather than relying on the overall pass count.

## Accuracy testing (not unit tests)

- **Harness:** `scripts/extract-refs.js` makes real OCR and LLM calls (about $0.01 per page) and scores F1 against `.expected.json`. Usage is in `docs/llm-ocr-kt.md` §13.
- **NPL checks:** `scripts/testNplExtraction.js` with `scripts/npl_samples/`.
- **Judging prompt changes:** temperature 0 is not deterministic on the batched inference server, so judge prompt changes on corpus-level F1 and replays (for example "6 of 12 runs"), never on a single run (`docs/llm-ocr-kt.md` §13; DECISIONS 2026-09-17).
