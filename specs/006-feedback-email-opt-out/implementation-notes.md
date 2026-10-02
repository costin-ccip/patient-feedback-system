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
