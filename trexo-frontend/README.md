# trexo-frontend: AIA-81A and AIA-83 forms in JUNE (Knowledge Transfer)

This package reflects the code as of 2026-10-08 (`uat` at `44812268`). Where the repo's own docs differ, follow this package.

**Scope:** the two USPTO forms that users generate from the JUNE assistant page (`/trexo-chat`). The rest of trexo-frontend is out of scope. The AIA-01 / AIA-14 / AIA-96 forms in the **Docgen** module are a separate feature.

- **PTO/AIA/81A**: patent power of attorney, or revocation of power of attorney with a new power of attorney, and change of correspondence address.
- **PTO/AIA/83**: request for withdrawal as attorney or agent, and change of correspondence address.

## What it is

Two modal forms inside the JUNE chat. The user picks the task, types one or more US application numbers in the chat, fills in the modal and submits. The frontend collects, validates and serialises the form. The PDFs are generated server-side, one per application number, and are delivered by e-mail and in **Track Task**.

| Form | Lex intent | Task name |
|---|---|---|
| AIA-81A | `AIA81ADocgenIntent` | Document Generation → "PoA/Revocation of PoA/Correspondence Add Change" |
| AIA-83 | `AIA83DocgenIntent` | Document Generation → "Attorney Withdrawal & Address Change Request" |

Delivery history:

| When | Ticket | What |
|---|---|---|
| 2025-08 | TAI-473 | AIA-81A form generation in JUNE |
| 2025-08 → 09 | TAI-221 | AIA-83 form generation in JUNE, including the 11.116(b)(7) notes field |
| 2025-08 | n/a | AIA-81A signatory validation and application-number error handling |
| 2025-09 | TAI-643 | AIA-83 sends plain JSON |
| 2025-09 | TAI-666 | Signature checkbox optional and checked by default |
| 2025-10 | n/a | AIA-81A correspondence value `sameAsAbove` |
| 2025-11 | n/a | Tutorial videos; announced in JUNE **Release 4.0** (November 2025) |
| 2026-10 | TP-174 | PropTypes on both modals |

## Who uses it

Tenant users who can open JUNE: the gateway mounts `/tenantAPI/trexo-assistant` behind `appAccess("JUNE")`. Their tenant must also subscribe to the intent, and the user must hold the app permission. Otherwise JUNE answers "…is not included in your organization's subscription…". The frontend applies no extra role check to these two forms. The users are typically patent attorneys, agents and paralegals changing representation on US matters.

## Where the code is

| File | Role |
|---|---|
| `src/app/pages/TrexoChat/components/AIA81Modal.js` | AIA-81A modal: state, validation, serialisation, layout |
| `src/app/pages/TrexoChat/components/AIA83Modal.js` | AIA-83 modal: same structure, larger |
| `src/app/pages/TrexoChat/TrexoChat.js` | Host page. Intercepts the `formDetails` slot and opens the modal (`handleChatSubmit`). Sends the payload (`handleModuleSubmit`). Cancels (`handleStop`). Renders the modals |
| `src/app/pages/TrexoChat/utils/assistantUtils.js` | `simplifyApplicationNumbers` and `checkValidAppNum` (8-digit rule), `getCurrentTime` |
| `src/app/pages/TrexoChat/components/help-section/Tutorial.js` | Tutorial video entries for both forms |
| `src/app/pages/TrexoChat/components/help-section/release-notes/ReleaseNote4.0.js` | User-facing announcement |

The other hops are in patent-asistant-apis, pa_lambda_functions and trexo-apigateway. See [architecture.md](architecture.md).

## Stack (for this feature)

React 17 function components with MUI 5 inputs. Form state is plain `useState`, with no Formik/Yup and no Redux. Requests use the shared `axios_tenant` client (`src/axios.js`). The build is Create React App (`react-scripts` 5).

## Run locally

1. `npm ci`. `postinstall` applies the two patch scripts in `scripts/`.
2. Create a root `.env` with at least `REACT_APP_API_BASE_URL` (a gateway you can log in to) and the JUNE variables in [deployment-and-environments.md](deployment-and-environments.md).
3. `npm start` (dev server on port 3000). Tenants are resolved from the first label of the host name (`src/app/utils/tenantUtils.js`), so use a host name that carries a tenant label.
4. Running end to end needs the gateway, patent-asistant-apis, the Lex bot alias, the LexChatbot Lambda, the template bucket and the USPTO ODP API. Point the local frontend at the dev or UAT gateway. Running the frontend alone lets you open the modal and inspect the request in the browser's network tab.
5. Steps: log in as a tenant user with JUNE → `/trexo-chat` → Document Generation → the AIA-81A or AIA-83 task → enter an 8-digit application number → the modal opens.

The repo uses npm (`package-lock.json`).

## Related packages

| Package | Use it for |
|---|---|
| [../pa_lambda_functions/](../pa_lambda_functions/README.md) | The LexChatbot fulfilment Lambda, including the AIA-81A / AIA-83 intent handlers |
| [../patent-asistant-apis/](../patent-asistant-apis/README.md) | The IP Assistant backend that hosts `/trexo-assistant` and the form generator |
| [../system-map.md](../system-map.md) | How the services connect |

## This package

| File | Read it for |
|---|---|
| [product-and-features.md](product-and-features.md) | The user journey and every field of both forms, with frontend and generator rules |
| [architecture.md](architecture.md) | The end-to-end path, with diagrams |
| [api-and-integrations.md](api-and-integrations.md) | The request contract at each hop |
| [auth-security.md](auth-security.md) | Access checks, the typed signature, personal data |
| [deployment-and-environments.md](deployment-and-environments.md) | Build, configuration variable names, cross-repo release order |
| [testing.md](testing.md) | Test commands and the manual test script |
| [operations-and-troubleshooting.md](operations-and-troubleshooting.md) | Symptom → where to look |
| [handover.md](handover.md) | Recent work, dependencies, ownership |

**Suggested reading order:** this README → product-and-features → architecture → api-and-integrations → operations-and-troubleshooting → handover.
