# Handover: AIA-81A / AIA-83 in JUNE (trexo-frontend)

As of 2026-10-08 (`uat` at `44812268`).

## Where things stand

Both forms are live in JUNE and were announced in Release 4.0 (November 2025). Since then the AIA files have changed only through cross-cutting chat work: timestamps, input limits, Custom Forms, PropTypes.

## Recent work (last ~3 months) touching these files

| Date | Ticket | Change |
|---|---|---|
| 2026-10-06 | TP-174 | PropTypes on `AIA81Modal` / `AIA83Modal` (and AppGen) |
| 2026-09-30 | TAI-1118 | Custom Forms intents. `assistantUtils.js` gained EP-aware number parsing for those intents only; the AIA behaviour is unchanged |
| 2026-08-19 | TAI-820 | Timestamps on chat messages, including the modals' "Request cancelled." |
| 2026-08-17 | TAI-889 | Input limit changes in `assistantUtils.js` |

Earlier history, for context:
- 2025-08: TAI-473, AIA-81A in JUNE.
- 2025-08 → 09: TAI-221, AIA-83 in JUNE, including the 11.116(b)(7) notes.
- 2025-09: TAI-643, AIA-83 sends plain JSON. TAI-666, signature checkbox checked by default.
- 2025-10: AIA-81A correspondence value `sameAsAbove`.
- 2025-11: tutorial videos and Release 4.0 notes.

## External dependencies

| Dependency | Owner repo or system | Needed for |
|---|---|---|
| `/tenantAPI/trexo-assistant` route and `appAccess("JUNE")` | trexo-apigateway | Every chat call |
| `getChatResponse`, `addInitialRecord`, `checkIntentSubscription` | patent-asistant-apis | Submit, large-slot storage, permissions |
| Lex V2 bot: intents `AIA81ADocgenIntent` / `AIA83DocgenIntent`, slot `formDetails` | AWS console | Opening the modal |
| LexChatbot Lambda: `docgen_handler.py`, `restore_large_slot_data` | pa_lambda_functions | The generator call |
| `/forms/AIA-81A`, `/forms/AIA-83` generator | patent-asistant-apis | The PDFs |
| `form_templates` / `tenant_form_templates` rows and template objects in S3 | patent-asistant-apis DB + S3 | Template and field mapping |
| USPTO Open Data Portal | external | Applicant and matter data |

## Ownership

| Area | Owner |
|---|---|
| JUNE frontend: AIA-81A / AIA-83 modals and `TrexoChat.js` wiring | |
| LexChatbot AIA handlers | |
| `/forms` generator and `form_templates` data | |
| Lex bot configuration | |
| Product decisions on the forms | |

## Where configuration lives

- Lex intents, slots and prompts: the AWS Lex console, per bot alias.
- PDF field mappings: `form_templates` / `tenant_form_templates` rows in the database.
- Frontend limits: `REACT_APP_*` variables at build time (see [deployment-and-environments.md](deployment-and-environments.md)).

## Where to look

- The request contract at each hop: [api-and-integrations.md](api-and-integrations.md).
- Runbooks: [operations-and-troubleshooting.md](operations-and-troubleshooting.md).
- Manual checks: [testing.md](testing.md).
