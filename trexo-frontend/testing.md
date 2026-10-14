# Testing: AIA-81A / AIA-83

## Test set-up

- Jest via `react-scripts test`, with React Testing Library 12 (`@testing-library/react`, `jest-dom`).
- The repo's automated tests cover shared utilities, services, Redux, routing, layouts and parts of the KOT module. The two forms are checked with the manual script below.
- Coverage: `package.json` → `jest.collectCoverageFrom` covers utilities, services, Redux, routes, hooks, layouts and selected components, against an 80 % global threshold. Page modules under `src/app/pages/**` (except `*menus*.js`) are outside that set.

## Commands

```bash
npm test                                         # watch mode; applies the patch scripts first
CI=true npx react-scripts test --watchAll=false  # single run
CI=true npm run build                            # lint + compile gate (ESLint warnings fail the build)
```

After a fresh `npm ci`, run `npm test` once (or `npm run patch:react-idle-timer`) before calling `react-scripts` directly, so the patches are applied.

## Manual test script

Run against UAT or dev, as a user whose tenant has both AIA tasks enabled. Use a real application number with applicants in USPTO records, and a second known-good number.

**Entry and cancel**

1. Choose the AIA-81A task and enter `16/123,456`-style input. Expected: the chat shows the digits and the modal opens.
2. Enter `abc`. Expected: the toast "The application number provided is invalid…" appears and no modal opens.
3. Open the modal, click X. Expected: "Request cancelled." appears, the chat resets and the task list is enabled again.
4. Press Esc or click outside the modal. Expected: the modal stays open.

**AIA-81A**

5. With the defaults (power of attorney submitted, `sameAsAbove`, applicant, signature checked), Submit stays disabled until Name is filled. Submit. Expected: an e-mail with one PDF. The "address associated with the above-identified Customer Number" box is ticked, the applicant is from USPTO and the signature reads `/Name/` with today's date.
6. Choose "Customer Number", leave it empty, Submit. Expected: the field turns red and nothing is sent.
7. Choose named practitioners and fill one row half. Expected: it is blocked. Fill one complete row. Expected: it generates, with practitioner 1 printed.
8. Choose Patent Owner without a date. Expected: blocked. Add a date. Expected: it is printed.
9. Uncheck signatures. Expected: Name is optional and no `/Name/` appears on the PDF.
10. Enter two application numbers. Expected: a ZIP with two PDFs, both with the same choices.

**AIA-83**

11. With the defaults (all practitioners, checklist all checked), Name and Registration Number are required for Submit. Submit. Expected: a PDF with the "all practitioners" box, the checklist ticks and `/Name/` with the date.
12. Choose the customer-number method. Expected: the 37 CFR reasons appear. Tick (b)(7) and type notes. Submit. Expected: the request completes, with the chosen reasons on the PDF.
13. Change the correspondence address → Applicant, fill in the address. Expected: the address is printed and the box is ticked.
14. Uncheck "Include signature information". Expected: Name and Registration Number are still required, and the PDF has no `/Name/` or date.

**Delivery**

15. Track Task shows the request with status and link. The e-mail link downloads the file.
