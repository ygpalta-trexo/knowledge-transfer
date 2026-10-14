# API and integrations: AIA-81A / AIA-83

The frontend calls only the gateway. The contracts further down are described because a form change usually spans **three repos**: the frontend, pa_lambda_functions and patent-asistant-apis.

## 1. Endpoints the frontend calls

All calls use `axios_tenant`: base `${REACT_APP_API_BASE_URL}/tenantApi`, header `Authorization: Bearer <TREXO_TENANT_AUTH_TOKEN> <sessionId>`.

| Method | Route (via gateway) | Purpose in this feature | Authn / authz | Notes |
|---|---|---|---|---|
| GET | `/tenantAPI/trexo-assistant/getIntentList` | Builds the task list, including the two AIA tasks | gateway `checkAuthentication` + `appAccess("JUNE")` | Loaded once when the page mounts |
| POST | `/tenantAPI/trexo-assistant/getChatResponse` | (a) Starts the intent from the task label. (b) Submits the form | same, plus the tenant and user intent permission check in patent-asistant-apis | multipart `FormData`; extra `sessionId` header |
| GET | `/tenantAPI/trexo-assistant/getTasks` | **Track Task** list, where the output link and errors appear | same | `IpAssistantTask.js` |

The gateway proxies `/tenantAPI/trexo-assistant` to `TREXO_ASSISTANT_API_URL`. The routes are in `patent-asistant-apis: routes/assistant.router.js`.

### `getChatResponse`: the form submission body

| Field | Value for AIA-81A / AIA-83 |
|---|---|
| `sessionId` | `chatSessionId` |
| `textInput` | `"Sample Text"` (a fixed placeholder; Lex receives this as the slot value) |
| `largeSlotData` | the serialised form (§3) |
| `originatingRequestId` | from the previous Lex response, `sessionState.originatingRequestId` |
| `requestId` | from the previous response, `sessionState.sessionAttributes.requestId` |
| `slotName` | `"formDetails"` |
| `intentName` | `"AIA81ADocgenIntent"` / `"AIA83DocgenIntent"` (the frontend uses it for pre-processing only) |

**Response:** `{ message: [...], sessionState }`. Messages are rendered in the chat. An empty message is replaced with "Your request ID … is being processed and will be delivered to your mailbox …".

**Errors:**

| Case | What the user sees |
|---|---|
| 400 from `getChatResponse` (for example, intent not subscribed) | The server's `message` appears as a JUNE reply, then the chat resets |
| 401 | Redirect to `/login` |
| Other HTTP errors, e.g. 500 "Internal server error" | Toast with the server message, then the chat resets |
| No response (network failure) | The chat resets without a message |

## 2. patent-asistant-apis → Lex → Lambda

- `getChatResponse` stores `{formDetails: largeSlotData}` with `addInitialRecord(originatingRequestId, requestId, sessionId, userEmail, …)`. If a row with that `originating_request_id` exists, the new value is merged into its `large_slot_data`. Otherwise a row is created with status "Ready for fulfillment".
- It then calls Lex `RecognizeText` with the text `"Sample Text"` and session attributes `tenantName`, `user` (JSON), `requestId` (new), `sessionId` and `forwardHeaders`.
- Lex fills `formDetails` and invokes the fulfilment Lambda. `IntentProcessor.process` upserts the task row and calls `restore_large_slot_data`, which overwrites the slot's `originalValue`, `interpretedValue` and `resolvedValues` with the stored string.

## 3. The `formDetails` string (frontend → Lambda)

### AIA-81A

`cleanData` turns every `""` into `null`, recursively. The result is `JSON.stringify`-ed, and the quotes around keys are removed with `.replace(/"([^"]+)":/g, "$1:")`.

Shape (pseudo):

```
{application_nums:"16123456,17234567", attorneyUpdateMethod:"powerOfAttorney"|"customerNumber"|"practitionerName",
 customerNumber:null|"…", practitioners:[{name,registrationNumber}×4], correspondenceMethod:"sameAsAbove"|"specificCustomerNumber",
 specificCustomerNumber, applicantType:"applicant"|"patentOwner", patentOwnerDate:"YYYY-MM-DD"|null,
 signatureRequired:true|false, signatoryInfo:{name,telephone,titleCompany}, formCount:"1", multipleFormsCheckbox:false}
```

The Lambda re-quotes keys with `re.sub(r"(\w+):", r'"\1":', s)`, converts `True`/`False`/`None` to `true`/`false`/`null`, and calls `json.loads`. The re-quoting applies to the whole string, so values must not contain a word directly followed by a colon. If parsing fails, the Lambda uses an empty form and the generator answers "Application Number is required for AIA-81A".

### AIA-83

Plain `JSON.stringify` of the state, with `""` kept as empty strings. The Lambda uses `safe_json_loads`: the same `True`/`False`/`None` conversion over the whole string, `json.loads`, and a second `json.loads` if the result is still a string.

```
{"application_nums":"…", "withdrawalMethod":"allPractitioners"|"specificPractitioners"|"customerNumberPractitioners",
 "customerNumber":"", "cfrReasons":{"_11_116_a_1":false, …, "_11_116_b_7":false, "_11_116_b_7_notes":""},
 "withdrawalChecklist":{"reasonableNotice":true,"deliveredPapers":true,"notifiedClient":true}, "explanation":"",
 "correspondenceChange":{"changeAddress":false,"method":"customerNumber","customerNumber":"","applicantAddress":{address,state,city,country,zipcode,telephone,email}},
 "signatureRequired":true, "signatoryInfo":{name,registrationNumber,addressLine1,state,city,country,zipcode,telephone,email}}
```

## 4. Lambda → generator

`POST {ACTION_BASE_URL}/forms/AIA-81A` or `/forms/AIA-83`, JSON, `timeout=900`. Headers: `X-User` (JSON), `X-Tenant-Name`, `sessionId`, `request-id`, `Content-Type: application/json`. Where `ACTION_BASE_URL` points is covered in [../pa_lambda_functions/api-and-integrations.md](../pa_lambda_functions/api-and-integrations.md).

| AIA-81A body key | From | AIA-83 body key | From |
|---|---|---|---|
| `applicationNumbers` | `application_nums` | `applicationNumbers` | `application_nums` |
| `attorneyUpdateMethod`, `customerNumber`, `correspondenceMethod`, `specificCustomerNumber`, `applicantType`, `patentOwnerDate`, `formCount` | same name, `null` → `""` | `withdrawalMethod` | same name, default `allPractitioners` |
| `practitioners` | array as sent | `customerNumber`, `explanation` | same name |
| `signatureRequired`, `multipleFormsCheckbox` | `"true"` / `"false"` strings | `signatureRequired` | `"true"` / `"false"` |
| `signatoryInfo` | JSON string | `cfrReasons`, `withdrawalChecklist`, `correspondenceChange`, `signatoryInfo` | JSON strings (the generator `JSON.parse`s them) |
| `signatoryName`, `signatoryTitle`, `signatoryTelephone` | from `signatoryInfo.name` / `titleCompany` / `telephone` | `attorneyName`, `attorneyRegistrationNumber`, `signatoryAddressLine1`, `attorneyCity`, `attorneyState`, `attorneyCountry`, `attorneyZip`, `attorneyTelephone`, `attorneyEmail` | from `signatoryInfo.*` |
| `correspondenceFirmName`, `…Address`, `…City`, `…State`, `…Zip`, `…Country`, `…Telephone`, `…Email` | `""` (used by the `firmOrIndividual` option) | | |

**Response:** `{ url, errorMessages }`. The Lambda turns it into a PlainText chat message with the request ID, the URL and any errors. An empty `url` becomes the "all application numbers failed" message, or the generator's `customMessage`.

## 5. Generator (`patent-asistant-apis: controllers/formGeneration.controller.js`)

- `formCode` must be in the allow-list, which contains `AIA-81A` and `AIA-83`.
- The template and field mappings come from `form_templates`, with the tenant override `tenant_form_templates`. The template file is read from `IP_ASSISTANT_BUCKET_NAME` and cached. The PDF is filled by `fillPdfFormStream(template, matter, mappings)`. The mapping from the `_…` matter keys to PDF field names is stored in those database rows.
- Matter data comes from `getApplication(appNo)` in `services/odpApi/patent` (USPTO Open Data Portal).
- The generator writes the result to `ip_assistant_tasks` through `updateDb` and e-mails it through `utils/sendEmail`.

## External systems (for this feature)

| System | Called by | How |
|---|---|---|
| Amazon Lex V2 bot | patent-asistant-apis | `RecognizeText`. Bot, alias and slot configuration are managed in the AWS console |
| USPTO Open Data Portal | generator | per application number |
| AWS S3 | generator | template read; output upload; pre-signed URL |
| E-mail | generator | `sendMail(updatedTask, user, …)` |
| Tutorial videos | frontend Help | `tutorialVideoUrl(JUNE, "AIA+81+A.mp4" / "AIA+83.mp4")`, base from `src/app/utils/constants/tutorialVideos.js` |
