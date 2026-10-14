# Auth and security: AIA-81A / AIA-83

The platform-wide auth model (tenant subdomain, stored tenant token, three axios clients, SSO through Cognito) is described in [../system-map.md](../system-map.md). This file covers what applies to the two forms.

## Mechanisms

| Layer | Mechanism | Where |
|---|---|---|
| Route | `/trexo-chat` is wrapped in `TenantAdminRoute`, which requires `TREXO_TENANT_AUTH_TOKEN` in `localStorage` | `TrexoChatRoute.js` |
| HTTP | `axios_tenant` sends `Authorization: Bearer <token> <sessionId>`. A 401 clears the tenant session and redirects to `/login` | `src/axios.js` |
| Gateway | `/tenantAPI/trexo-assistant/*` sits behind `checkAuthentication` and `appAccess("JUNE")` | `trexo-apigateway: routes/tenantAPI.router.js` |
| Intent permission | For each turn, `checkIntentSubscription` requires that the tenant has the intent's IP app active and not hidden, and that the user's allowed app ids include it, unless the intent is in `EXCLUDED_INTENT_LIST`. Otherwise it answers 400 with a subscription message, which JUNE shows | `patent-asistant-apis: utils/chatHelpers.js` |
| Frontend role | None for these forms. The only role check on the page hides "Create New Custom Form" from non-admins | `TrexoChat.js` |
| Input handling | Application numbers are reduced to digits before display. The chat bubble for the form shows a fixed label ("AIA 81 Form Data"), so no form text is rendered as HTML in the chat | `assistantUtils.js`, both modals |
| Audit trail | `ip_assistant_tasks` keeps `user_id` (e-mail), `large_slot_data` (the submitted form string), `request_json` (the full Lex event), `status`, output link and errors per request | pa_lambda_functions `lambda_handler.py`; patent-asistant-apis `chat.controller.js` |
| Output access | The output is a pre-signed S3 URL that expires after `AWS_S3_URL_EXPIRATION`. It is sent by e-mail and shown in Track Task | `patent-asistant-apis: utils/fileUpload.js` |

## The typed signature

With the signature box checked, the generator prints `/<name>/` and today's date. This is the USPTO S-signature format, so the PDF reads as signed by the named person. The name is whatever the user typed in the signatory fields. The person who files the form is responsible for it, and the audit trail records who generated it.

## Personal data

Both forms carry names, postal addresses, phone numbers and e-mail addresses of practitioners and applicants. That data is stored in `ip_assistant_tasks` (`large_slot_data`, `request_json`). The LexChatbot Lambda and patent-asistant-apis also write slot contents to their logs. Handle those tables and log groups as containing personal data.
