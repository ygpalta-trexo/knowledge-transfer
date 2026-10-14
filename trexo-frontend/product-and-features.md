# Product and features: AIA-81A and AIA-83 in JUNE

## The business problem

When a firm takes over or gives up a US patent matter, it files a change of representation with the USPTO. **PTO/AIA/81A** appoints practitioners or revokes and replaces a power of attorney. **PTO/AIA/83** asks for practitioners to be withdrawn. Both can also change the correspondence address. JUNE fills the matter data from USPTO records, applies one set of choices to **several applications in one request**, and returns ready-to-sign or e-signed PDFs.

## User journey (both forms)

1. Open JUNE (`/trexo-chat`) and choose **Document Generation** → the AIA-81A or AIA-83 task. The frontend sends the task label as chat text. Lex matches it to the intent and asks for the `formDetails` slot. The prompt wording is configured in the Lex bot.
2. The user types **US application numbers**, one per line or comma-separated. The frontend does not send this to Lex. It normalises the numbers and opens the modal:
   - non-digits are stripped, so `16/123,456` becomes `16123456`;
   - at least one entry must be exactly 8 digits;
   - the count may not exceed `REACT_APP_JUNE_APPLICATION_NUMBER_LIMIT`;
   - on failure a toast appears and the chat waits for new input.
3. The numbers appear as the user's chat message and the modal opens at 85 % width. The application numbers are not editable inside the modal. To change them, cancel and start again.
4. The user fills in the form. **Submit** stays disabled until the always-required fields are filled. Other rules are checked on click and highlight the field in red. **Reset** clears the form but keeps the application numbers. **X** cancels: the chat shows "Request cancelled." and a new chat session starts. Esc and clicking outside do not close the modal.
5. On submit the chat shows "AIA 81 Form Data" or "AIA 83 Form Data" as the user's message, and the form goes to the backend (see [api-and-integrations.md](api-and-integrations.md)).
6. JUNE replies with a request ID. The output arrives **by e-mail** as a link and is listed in **Track Task**.

### Output

- **One PDF per application number**, named `<applicationNumber>_<formName>_<YYYY-MM-DD>.pdf`. When the request yields more than one PDF, they are zipped as `<requestId>_<formName>_<date>.zip`. The file goes to S3 and the link is a pre-signed URL.
- The **same choices apply to every application number** in the request.
- Applicant names come from the USPTO Open Data Portal: the applicants, or the inventors if there are none. The form gets the first name, and the remaining names joined with `"; "`.
- If one number fails (not found, no applicants or inventors, a missing required value), that number gets an error line and the other numbers are still generated. The e-mail lists the errors.
- The signature date and the AIA-83 effective date are the server's current date.

## AIA-81A: field by field

Modal title: "PATENT- POWER OF ATTORNEY … AND CHANGE OF CORRESPONDENCE ADDRESS".

| Section (UI) | Control | Payload key | Values (default **bold**) | Frontend rule | Generator rule |
|---|---|---|---|---|---|
| Appointment of practitioners | radio | `attorneyUpdateMethod` | **`powerOfAttorney`** ("A Power of Attorney is submitted herewith"), `customerNumber`, `practitionerName` | | required, one of the three |
| Customer Number (shown for `customerNumber`) | text | `customerNumber` | | required for `customerNumber` | same |
| Practitioners (shown for `practitionerName`) | table, 4 rows | `practitioners[]` `{name, registrationNumber}` | 4 rows always sent; blanks become `null` | at least one complete row; no half-filled row | same, plus at most 4 |
| Correspondence address | radio | `correspondenceMethod` | **`sameAsAbove`** ("address associated with the above-identified Customer Number"), `specificCustomerNumber` | `specificCustomerNumber` needs a number | also supports `skip` and `firmOrIndividual` (firm name and address), which the UI does not offer |
| Customer Number (correspondence) | text | `specificCustomerNumber` | | required for `specificCustomerNumber` | same |
| "I am the" | radio | `applicantType` | **`applicant`**, `patentOwner` | | ticks the matching box |
| Statement under 37 CFR 3.73(c) (PTO/AIA/96) date | date | `patentOwnerDate` | `YYYY-MM-DD` | required for `patentOwner` | same |
| "Would you like to add signatures?" | checkbox | `signatureRequired` | **checked** | when checked, Name is required and Submit is disabled without it | when true: Name required, and the form gets `/Name/` plus today's date |
| Signatory | text ×3 | `signatoryInfo.{name, telephone, titleCompany}` | | | sent as `signatoryName`, `signatoryTelephone`, `signatoryTitle` |
| Multiple forms | checkbox + count | `multipleFormsCheckbox`, `formCount` | **unchecked**, **`"1"`** | free text | box On/Off, count printed as typed |

## AIA-83: field by field

Modal title: "REQUEST FOR WITHDRAWAL AS ATTORNEY OR AGENT AND CHANGE OF CORRESPONDENCE ADDRESS".

| Section (UI) | Control | Payload key | Values (default **bold**) | Frontend rule | Generator rule |
|---|---|---|---|---|---|
| Withdrawal method | radio | `withdrawalMethod` | **`allPractitioners`**, `specificPractitioners` ("listed on the attached paper(s)"), `customerNumberPractitioners` | | required |
| Customer Number (inline) | text | `customerNumber` | | required for `customerNumberPractitioners` | same |
| Attached list for `specificPractitioners` | none | | | the user attaches the list when filing | |
| Reasons, 37 CFR 11.116 (shown for `customerNumberPractitioners`) | 10 checkboxes | `cfrReasons._11_116_a_1` … `_a_3`, `_b_1` … `_b_7` | all unchecked | | applied for `customerNumberPractitioners` |
| 11.116(b)(7) notes (shown when (b)(7) is ticked) | multiline | `cfrReasons._11_116_b_7_notes` | | | sent inside `cfrReasons` |
| Withdrawal checklist ("If a box is left unchecked, the request will likely not be approved") | 3 checkboxes | `withdrawalChecklist.{reasonableNotice, deliveredPapers, notifiedClient}` | **all checked** | | passed to the template |
| Explanation | multiline | `explanation` | | | printed |
| Change correspondence address | checkbox | `correspondenceChange.changeAddress` | **unchecked** | | |
| … method | radio | `correspondenceChange.method` | **`customerNumber`**, `applicant` | `customerNumber` needs a number | ticks the matching box |
| … Customer Number | text | `correspondenceChange.customerNumber` | | required for `customerNumber` | |
| … applicant address | 7 text fields | `correspondenceChange.applicantAddress.{address, state, city, country, zipcode, telephone, email}` | | none required | printed |
| "Include signature information" | checkbox | `signatureRequired` | **checked** | | when true: `/Name/` signature plus today's date |
| Signatory (withdrawing attorney) | 9 text fields | `signatoryInfo.{name, registrationNumber, addressLine1, state, city, country, zipcode, telephone, email}` | | **name and registration number always required** | sent as `attorneyName`, `attorneyRegistrationNumber`, `signatoryAddressLine1`, `attorneyCity` … ; name and registration number required |

The note under the signature section reads: "Withdrawal is effective when approved rather than when received."

## Business rules worth knowing

- **Typed signature.** When the signature box is checked, the generator prints the name between slashes (`/Jane Doe/`) and today's date. This is the USPTO S-signature format.
- **Applicants and inventors.** The forms list applicants. If USPTO data has none, they list inventors. If there are neither, that application number fails.
- **Duplicates.** Repeated application numbers in one request are reported as "Duplicate application number" and generated once.
- **Template per tenant.** The PDF template and its field mappings come from `form_templates`, with an optional per-tenant override in `tenant_form_templates`. A tenant can receive a different layout without a code change.

## Points to check when working on these forms

- **Values must match across the three repos.** A radio value the generator does not handle produces a PDF with that box unticked, and the request still succeeds. When adding or renaming a value, update the generator first (see [deployment-and-environments.md](deployment-and-environments.md)).
- **One request covers many matters.** Every PDF in the ZIP repeats the same choices, so check one generated PDF fully before relying on a batch.
- **Free-text formatting for AIA-81A.** Do not put a colon directly after a word in the text fields (write "VP Legal", not "VP: Legal"). The AIA-81A payload uses a key/value format in which `word:` marks a field name.
- **Capitalised True, False and None.** On the way to the generator, these words are converted to JSON literals wherever they appear in the text.

## Neighbouring features (not covered)

- **AIA-41** (`AIA41DocgenIntent`) and **SB-30 EFS** (`RCESB30Modal.js`) use the same `formDetails` mechanism and generator.
- **AppGen** (`UsFilingModal.js`) generates AIA-01 / AIA-14 / AIA-15 / AIA-82.
- The **Docgen** module (`src/app/pages/Docgen/`) has its own AIA form editor and USPTO filing.
