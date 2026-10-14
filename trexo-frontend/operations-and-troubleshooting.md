# Operations and troubleshooting: AIA-81A / AIA-83

There are no scheduled jobs for this feature. Each request runs once, synchronously, as described in [architecture.md](architecture.md).

## Where to look

| Hop | What you can see |
|---|---|
| Browser | DevTools → Network → `getChatResponse`. The submit call's multipart body holds `largeSlotData` (the exact form string), `originatingRequestId`, `requestId` and `slotName`. The response holds `sessionState.intent.name` and `dialogAction.slotToElicit` |
| patent-asistant-apis logs | "Received chat request…", "Lex response: …" (full JSON, including slots), "Error in getChatResponse", "…is not subscribed to…". The generator logs "📄 AIA-81A form generation started for <appNo>" / "📄 AIA-83 …", "AIA-83 withdrawal method: …", "upload:start / upload:end", "Matter not found for …" |
| LexChatbot Lambda (CloudWatch) | Lines prefixed `AIA81ADocgenIntent -` / `AIA83DocgenIntent -`: "Original slots", "Parsed form_details successfully" or "JSON decode error", "Final form_data", "Response", "Critical error" with traceback |
| Tenant DB, `ip_assistant_tasks` | One row per request, keyed by `originating_request_id`. `request_id` is the id the user sees. Also `status`, `sub_task`, `large_slot_data.formDetails` (what the user submitted), `download_link`, `error_message`, `request_json` |
| E-mail / Track Task | What the user saw: link and error lines |

## Symptoms

| Symptom | Likely cause | Check / action |
|---|---|---|
| Toast "The application number provided is invalid…" | No entry has exactly 8 digits after stripping, or a line is empty after stripping | Ask for US application numbers, one per line or comma-separated |
| Toast "The count of numbers should not exceed N" | More numbers than `REACT_APP_JUNE_APPLICATION_NUMBER_LIMIT` | Split the request |
| Numbers are sent to JUNE as text and no modal opens | The Lex intent name or slot name differs from what the frontend expects (`aia81adocgenintent` / `aia83docgenintent`, slot `formDetails`) | Compare `sessionState.intent.name` and `slotToElicit` in the previous response with `TrexoChat.js` `handleChatSubmit` |
| JUNE replies "…is not included in your organization's subscription…" | The tenant lacks the app, or the user lacks the app permission | Enable the app for the tenant and grant it to the user |
| The chat resets without a message after Submit | Network failure between browser and gateway | Browser network tab, then gateway and patent-asistant-apis logs for that time |
| Toast "Internal server error" | Exception in `getChatResponse` (Lex call or DB write) | patent-asistant-apis "Error in getChatResponse" stack |
| Request ID shown but no e-mail | Generation still running (up to 900 s), stopped before e-mail, or e-mail delivery failed | Find the `ip_assistant_tasks` row by `request_id`, check `status` and `error_message`, then the Lambda and generator logs |
| E-mail: "Application Number is required for AIA-81A", although numbers were entered | The AIA-81A form string could not be parsed, usually because a text field has a word directly followed by a colon (e.g. "VP: Legal") | Lambda log "AIA81ADocgenIntent - JSON decode error". Remove the colon and resubmit |
| E-mail: "<appNo> - Application number not found" | USPTO ODP has no record, or ODP is unavailable | Try the number on USPTO. Check the generator "Matter not found" log and the ODP variables |
| E-mail: "<appNo> - No applicants or inventors found" | The ODP record has neither | Prepare that form by hand |
| E-mail: "… is required for AIA-81A / AIA-83" | A required value reached the generator empty | Compare `large_slot_data.formDetails` with the Lambda "Final form_data" |
| "Failed to load form template for AIA-8x" (504) | No `form_templates` row for the code, or the S3 object is missing | Check the row and the `templates_path` object in `IP_ASSISTANT_BUCKET_NAME` for that environment |
| One tenant gets a different layout or misplaced fields | A `tenant_form_templates` override for that form code | Inspect the override's `mappings` / `templates_path` |
| AIA-81A generated with no correspondence box ticked | `correspondenceMethod` holds a value the generator does not handle | Check `large_slot_data.formDetails`. Accepted values: `sameAsAbove`, `specificCustomerNumber`, `skip`, `firmOrIndividual` |
| Words in names or explanations print as `null` / `true` / `false` | The capitalised words None, True and False are converted to JSON literals on the way to the generator | Reword the text and resubmit |
| (b)(7) notes not on the AIA-83 PDF | The AIA-83 template mapping does not include `_11_116_b_7_notes` | Check `form_templates.mappings` for AIA-83 |

## Operational checklist (per environment)

- [ ] The Lex bot alias used by patent-asistant-apis (`LEX_BOT_ID` / `LEX_BOT_ALIAS_ID`) has both intents built, with the `formDetails` slot.
- [ ] The deployed LexChatbot Lambda includes the AIA-81A / AIA-83 handlers.
- [ ] `form_templates` has rows for `AIA-81A` and `AIA-83`, and their `templates_path` objects exist in `IP_ASSISTANT_BUCKET_NAME`.
- [ ] `ODP_API_KEY` is valid.
- [ ] The output bucket, `AWS_S3_URL_EXPIRATION` and e-mail credentials are set.
- [ ] Each tenant that should see the forms has the IP apps active and not hidden, and users have the app permission.
- [ ] The frontend build has `REACT_APP_JUNE_APPLICATION_NUMBER_LIMIT` set.
