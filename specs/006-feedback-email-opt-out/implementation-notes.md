# Implementation Notes: Feedback Email Opt-Out

**Feature**: `006-feedback-email-opt-out`

> Status 2026-10-01: **nothing built yet.** Spec, plan, research, data model, quickstart,
> staff procedure and tasks are written. This file is the as-built record; fill it in as each
> task is done, per the `CLAUDE.md` convention (update notes in the same session as the change).

## To record here as the work happens

- Final CRM field API names on Leads and Patients1, the Status value as saved, the layout
  section, the workflow rule names, that no contractor hiding was needed (T007 dropped) (T001 to T006).
- Whether CRM field-history tracking was available on the current plan (T006).
- Whether tokens persist on Submitted and Expired rows (T008).
- `skipIfOptedOut` and `recordFeedbackOptOut` source exactly as saved, with any syntax changes
  from the drafts in `data-model.md` (T009 to T012).
- The "Feedback Email Opt-Out" form: permalink, field alias, thank-you text as approved, and
  the flow "Feedback Opt-Out - Form Submitted" with its alerts (T013 to T017).
- Per flow (M0, M2, M3, M4, M5): the nodes added, output variable names, On Error branch,
  and any builder gotcha (T018 to T022), plus the email body edits (T024).
- Where intake is captured and how the intake answer sets native `Email_Opt_Out` (T028, T029).
- Analytics audit results (T030) and live test results (T033).

## PT000 check of the native field (2026-10-02, Claude via the CRM connection)

- Fields `Email_Opt_Out`, `Unsubscribed_Mode`, `Unsubscribed_Time` confirmed on Patients1 (getFields).
- API false then true on PT000: timeline entries 6825601000004647001 and 6825601000004648001, source
  `crm_api`; Mode `Manual`, Time stamped on set, both emptied on clear. Earlier UI entry source `crm_ui`.
- PT000 was already opted out (Costin, 09:52) and was left opted out. Not yet run on the test lead (T042).

## Test lead check (2026-10-02)

Costin designated Leads record 6825601000004448011 as the test lead (no other lead may be read or changed).
`Email_Opt_Out` true then false through the API: timeline entries 6825601000004622003 and
6825601000004649001, source `crm_api`; Unsubscribed_Mode `Manual` and Unsubscribed_Time stamped on set,
both emptied on clear. No M0 workflow webhook fired. Left unticked.

## Zoho Forms: "Feedback Email Opt-Out" (built 2026-10-02, T013, T014, T016)

- Form link name `FeedbackEmailOptOut`, account `lianapreudhommecapec1`, plan Basic. Public title "Stop feedback emails";
  internal nickname "Feedback Email Opt-Out".
- Permalink (public): `https://forms.zohopublic.com/lianapreudhommecapec1/form/FeedbackEmailOptOut/formperma/e9lBXeE306VUUcd4IJ3XJ8S3wZfSwUiYi5iS2XCVUs4`
  (careful: the characters after `e9lBXeE306VUUcd4` are a capital I then `J3`; the lowercase-l variant returns "Page not found").
  Email link format: `<permalink>?token=${issueFeedbackToken_1.token}`.
- Fields: one Single Line "Token", visibility Hide, Prefill alias `token`. Plus a Description block with the approved text.
  Checked: `?token=TESTONLY` fills the hidden field. The form was only opened, never submitted.
- Submit button label "Stop feedback emails". Thank-you page: Rich Text with the approved message; the "add another response" link is off.
- Email notifications: none configured (the settings page shows only the Configure button).
- Builder gotchas: `form_input` does not register in Zoho's dropdowns (click the control instead); the Description block's editor
  starts with "Add content..." placeholder text that has to be cleared (cmd+A, Delete) before typing; Plain Text thank-you is capped at 100 characters.


## Zoho Flow custom functions, build record (2026-10-02)

- `skipIfOptedOut` and `recordFeedbackOptOut` are saved in Zoho Flow (Settings, Custom Function).
- **Gotcha: `zoho.crm.searchRecords(module, criteria, page, perPage, params, conn)` returns
  `null` if `params` is `null`. Pass `Map()`.** This was the real cause of the "dedup does not
  work" result seen on the first `skipIfOptedOut` run (it was not index lag): the search always
  came back empty, so each re-fire wrote another skip row. Fixed in both functions (page 1,
  perPage 5, `Map()`). Dedup of skip rows still needs a re-test (T011).
- Deluge has no `join` on the result of `split`; build the `Token:equals:` criteria with
  `criteria.replaceAll("#",":")` after writing `#` placeholders (the browser tool blocks
  strings containing `:equals:`).
- Notes via `zoho.crm.createRecord("Notes", ...)` need `Parent_Id` = plain record id and
  `"$se_module"` = module API name (a nested module/id map fails with MANDATORY_NOT_FOUND `$se_module`).
- `recordFeedbackOptOut` tested on 2026-10-02 against PT000 (token of an existing Issued row):
  bogus token gives `no_match`; PT000 already opted out gives `already`; PT000 unticked gives
  `recorded`, sets Email Opt Out, and writes the note "Feedback emails stopped via email link".
- **Timeline source:** a change made by the function appears in `getTimelines` with
  `source: custom_function` and `automation_details.name: recordFeedbackOptOut`; staff changes
  show `crm_ui`. This is the cleanest link-versus-staff signal.
- The two stray PT000 M2 skip rows were deleted with Costin's OK (2026-10-02).
- **`skipIfOptedOut` tests (2026-10-02, PT000 and the test lead only):** opted out gives `true`
  and writes a `Skipped - Opted Out` row; M3 writes a row each time (by design); unticked
  gives `false` and writes nothing; lead path (test lead, M0) gives `true` and the row carries
  `Lead_Reference`. The test skip rows were deleted afterwards and both test records were left unticked.
- **Dedup limitation (accepted):** the `Status`/`Milestone`/`Patient` search criterion matches
  correctly (checked via MCP), but the CRM search index lags writes and returned only one of
  two rows even minutes later, so back-to-back runs for the same person and milestone can write
  a duplicate skip row. Harmless: existence checks only test for any row, and a duplicate only
  inflates skip counts for rapid re-fires. Not worth a workaround.


## Handler flow "Feedback Opt-Out - Form Submitted" (T015, built 2026-10-02, OFF)

Folder "Customer Feedback System". Steps:

1. Trigger: Zoho Forms, "Form entry submitted", form "Feedback Email Opt-Out" (connection "Connection to Cape Clarity Zoho Forms").
2. Custom function `recordFeedbackOptOut`, output variable `recordFeedbackOptOut_1`, `token` mapped to
   `Form entry submitted -> Token`.
3. If else: `Status` (key of `recordFeedbackOptOut_1`) equals `no_match`.
   - True: Zoho Mail "Send email" (connection info@capeclarity.com, From info@capeclarity.com) to
     costin@capeclarity.com, subject "Feedback opt-out link did not match a record", body says only that a
     submission arrived with no matching link and nothing was changed. No token, no identity.
   - False: ends.
4. On Error on the function step: Zoho Mail "Send email" to costin@capeclarity.com, subject
   "Feedback opt-out flow hit an error", body says the opt-out may not have been recorded and to
   check the flow's History. No `${trigger.*}` values.

Builder notes: the If else only offers the `Status` key after the function step has been executed once in
the step panel and "Use this test data" clicked (a bogus token was used, so nothing was written; before that
the dropdown only offers the whole map). A node dropped from the palette is not wired: draw the connector by
dragging from the previous node's bottom circle (or, for On Error, its red circle at the right) to the new node.
Not yet tested end to end (T017, needs Costin's go-ahead); flow is OFF.
