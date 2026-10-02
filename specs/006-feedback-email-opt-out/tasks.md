---
description: "Task list for 006 - Feedback Email Opt-Out"
---

# Tasks: Feedback Email Opt-Out

**Input**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `quickstart.md`, `staff-procedure.md`

**Status 2026-10-01**: planned, nothing built. Spec, plan and these tasks are written; no Zoho
change has been made.

**Who does what**: tasks marked **[Costin]** need his hands or approval (CRM Setup pages do not
load for browser automation, and writes to CRM may need his approval). Tasks marked **[Claude]**
can be done by a session with browser or CRM access once he approves. Live tests are always
his go-ahead (`CLAUDE.md`). Nothing here switches a milestone flow ON that is currently OFF.

**Test records**: patient PT000 (`Patients1` `6825601000004448028`) and the existing test lead.
No other patient or lead record is read or changed.

## Phase 1: CRM setup (blocks everything)

- [ ] T001 [Costin or Claude via CRM connection, with approval] Create the four fields on
      **Leads**: `Feedback_Opt_Out` (checkbox), `Feedback_Opt_Out_Date` (date),
      `Feedback_Opt_Out_Via_Link` (checkbox), `Feedback_Resubscribe_Date` (date). Checkbox and
      date types only, no text or picklist (Option B, decided 2026-10-01). Before creating,
      confirm in Setup, Modules and Fields that room remains in the checkbox and date pools.
      See data-model.md §1.
- [ ] T002 [Costin] Same four fields on **Patients1** (module label "Patients").
- [ ] T003 [Costin] Add `Skipped - Opted Out` to the `Milestone_Instances.Status` picklist
      (same way `Send Failed` was added). [Claude] then confirms with a read-only `getFields`.
- [ ] T004 [Costin] Put the four fields in a "Feedback preferences" section on the Leads and
      Patients layouts.
- [ ] T005 [Costin] Create the two workflow rules on each of Leads and Patients1 (opt-out set
      fills the date; opt-out cleared fills the resubscribe date and unticks via-link),
      data-model.md §1. If Standard blocks the rules, build a small Zoho Flow flow instead.
- [ ] T006 [Costin] If the CRM plan allows, turn on field-history tracking for
      `Feedback_Opt_Out`. Record whether it was available; if not, the date fields and the CRM
      timeline are the history (research.md §6).
- [x] T007 ~~Hide the four fields from contractor profiles~~ **Dropped 2026-10-01**: contractors
      have no CRM access (Costin). If that changes, hide the fields from that profile first
      (constitution Principle IV, spec FR-012).
- [ ] T008 [Claude] Confirm tokens persist on closed rows: read-only look at one `Submitted`
      and one `Expired` test `Milestone_Instances` row to check `Token` is still populated
      (non-PII module). If a write-back clears it, stop and revise design decision D1 with
      Costin before going further.

**Checkpoint**: fields, status value, workflow and contractor visibility exist; token
persistence confirmed.

## Phase 2: Shared functions (blocks both user stories)

- [ ] T009 [Claude] Create the custom function `skipIfOptedOut` in Zoho Flow (Built-ins,
      Developer Tools, Custom Functions, "+ Custom Function"; return `bool`, inputs
      `milestone`, `patientId`, `leadId`, `clinician`), paste the body from data-model.md §3,
      save. Fix any Deluge syntax the editor rejects (in particular the 4-argument
      `getRecordById`) and update data-model.md to the version that saved.
- [ ] T010 [Claude] Create `recordFeedbackOptOut` (return `map`, input `token`),
      body from data-model.md §4, save; same rule about updating data-model.md.
- [ ] T011 [Claude] Execute checks for `skipIfOptedOut`: quickstart.md §A steps 1 to 5. Also
      confirm the `Status:equals:Skipped - Opted Out` search criterion matches (data-model.md
      §3 note). Needs T001 to T003 and Costin's OK to write to PT000.
- [ ] T012 [Claude] Execute checks for `recordFeedbackOptOut`: quickstart.md §A steps 6 to 10,
      including an `Expired` token and a `Submitted` token. Confirm `Token:equals:` search
      works on this text field, or switch to the looped read and note the 200 cap.

**Checkpoint**: both functions saved and verified in isolation; no flow uses them yet.

## Phase 3: User Story 1, stop feedback emails from the email itself (P1)

**Independent test**: quickstart.md §B.

- [ ] T013 [US1] [Claude] Create the Zoho Forms form "Feedback Email Opt-Out" (data-model.md §5):
      hidden Token field, explanation text, "Stop feedback emails" button, no notifications.
- [ ] T014 [US1] [Claude] Set up the Prefill field alias so `?token=` fills the hidden field
      (the alias must match the URL parameter, `m2-implementation-notes.md` §13). Record the
      form permalink in implementation-notes.md.
- [ ] T015 [US1] [Claude] Build the flow "Feedback Opt-Out - Form Submitted" (data-model.md §6),
      including the no-match alert and an On Error branch, both free of `${trigger.*}` values.
- [ ] T016 [US1] [Costin] Get Liana's approval of the page explanation, button label and
      thank-you message (research.md §4), then enter the final text in the form.
- [ ] T017 [US1] [Claude] Run quickstart.md §B (with Costin's go-ahead): page content, scanner
      stand-in does nothing, confirm records opt-out, bogus token alerts without identity,
      no identifier anywhere.

**Checkpoint**: an opt-out can be recorded from a link on a test record end to end, with all
milestone flows unchanged.

## Phase 4: User Story 2, an opted-out person never gets another email (P1)

**Independent test**: quickstart.md §C scenarios 1 to 7 and 10. Flows are edited one at a time;
check each in Flow before starting the next. Each edit: add `skipIfOptedOut` (output variable
`skipIfOptedOut_1`) inside the existing True branch, then an If-else `skipIfOptedOut_1 is false`
leading to the existing `issueFeedbackToken` and Send email steps, then an On Error branch on
`skipIfOptedOut` to the standard Zoho Mail alert (no `${trigger.*}` identity). Map `milestone`
to the same literal the flow passes to `issueFeedbackToken`, `patientId` to `${trigger.id}`
(or `leadId` for M0), `clinician` to the same literal.

- [ ] T018 [US2] [Claude] M2 - Session 3 Trigger (simplest; do first). Update
      `m2-implementation-notes.md` in the same session.
- [ ] T019 [US2] [Claude] M4 trigger flow. Update `m4-implementation-notes.md`.
- [ ] T020 [US2] [Claude] M5 trigger flow. Update `m5-implementation-notes.md`.
- [ ] T021 [US2] [Claude] M3 - Periodic Check-In Trigger (note the skip rows advance the
      checkpoint index, research.md §3). Update `m3-implementation-notes.md`.
- [ ] T022 [US2] [Claude] M0 - Lost Lead Feedback Token. It has no If-else today, so add the
      function and If-else between the trigger and `issueFeedbackToken` (milestone
      `0 - No Conversion`, `leadId` from the trigger). Update `m0-implementation-notes.md`.
- [ ] T023 [US2] [Costin] Get Liana's approval of the opt-out line added to the five emails
      (data-model.md §7).
- [ ] T024 [US2] [Claude] Add that line and link to each of the five "Send email" bodies
      (HTML code view; see `m0-implementation-notes.md` for the builder gotchas), using the
      form permalink and `?token=${issueFeedbackToken_1.token}`. Check each saved body.
- [ ] T025 [US2] [Claude] Structural verification of all five flows: wiring via
      `jsplumb-connected`, `issueFeedbackToken` and `isCreatedAfterCutoff` untouched, no flow's
      ON/OFF state changed, no `${trigger.*}` identity in any new alert.

**Checkpoint**: SC-001 and SC-002 verifiable structurally on all five flows.

## Phase 5: User Stories 3 and 4, staff channel and resubscribe (P2, P3)

- [ ] T026 [US3] [Costin] Review `staff-procedure.md` with Liana; adjust wording; confirm the
      one-business-day rule is what the practice will actually keep (spec FR-014).
- [ ] T027 [US3] [Costin] Name who checks the `info@capeclarity.com` replies each business day
      and where the procedure lives for staff.
- [ ] T028 [US1] [Costin] Intake is decided in the Campaigns project (its decisions of
      2026-10-01 and tasks T027 and T028): an optional "feedback requests" item in the EHR
      intake packet, recorded by staff in this feature's field, plus a notice (no checkbox)
      on the booking form; record only, no gate. Confirm that is still the plan, record the
      final field API names from T001 and T002 and tell the Campaigns build once (do not edit
      that repo from here), and note in implementation-notes.md. Also confirm that creating a patient from a lead copies no
      field from the Lead's feedback fields (there should be none; the new fields are not in
      any field mapping).
- [ ] T029 [US1] [Costin] When the optional intake item is added to the packet (Campaigns T027),
      add to the staff procedure that a "no" answer is recorded by checking
      `Patients1.Feedback_Opt_Out` (the date fills itself in; "via email link" stays
      unchecked, and the CRM note says "intake"). Patients with no answer stay not opted out.

**Checkpoint**: staff can record and clear opt-outs by procedure; intake sets the patient's
status.

## Phase 6: Reporting and boundaries (supports spec FR-010, FR-012, FR-017)

- [ ] T030 [Claude] Analytics audit after the first skip rows sync: confirm each milestone's
      Status Breakdown shows `Skipped - Opted Out`; read every report, formula column and
      metric that references Status (including feature 005's contractor reports and the
      "Flagged for Review" report) and confirm none counts a skip as a non-response, a
      completion, or a score. Fix any that do and record each in the notes. (Respect the
      deletion-order rules in `CLAUDE.md` if any report must change.)
- [ ] T031 [Costin] Confirm the four new fields are not in the Analytics CRM sync field list
      (they should be unchecked) and not in any Zoho Campaigns audience mapping; tell the
      Campaigns build once, as a note, that these fields must stay out (do not change the
      Campaigns repo from here).
- [ ] T032 [Costin] Add the "Feedback Email Opt-Out" form to the manual raw-entries purge
      routine in `data-retention-purge.md` (it holds tokens only, but is still a raw
      submission store).

## Phase 7: Live validation (Costin's go-ahead)

- [ ] T033 [Costin] Run quickstart.md §C, folded into the coordinated live test, once the
      milestone flows are switched ON for that test. Record results, including the scenario 9
      outcome.
- [x] T034 ~~Contractor-profile check~~ **Not needed 2026-10-01**: contractors have no CRM
      access. Revisit only if that changes.

## Phase 8: Documentation

- [ ] T035 [Claude] Write `implementation-notes.md` for this feature as the work happens:
      field API names, function source as saved, form permalink, flow names, per-flow wiring,
      builder gotchas, and the plan-tier answer from T006.
- [ ] T036 [Claude] Add a dated "Feedback opt-out" section to each of `m0`, `m2`, `m3`, `m4`,
      `m5` `-implementation-notes.md` as each flow is edited (T018 to T022), pointing here.
- [ ] T037 [Claude] Add a "Feedback opt-out convention" section to `CLAUDE.md`: every milestone
      flow, including future ones, calls `skipIfOptedOut` inside its fire-or-not True branch
      before issuing; do not copy its logic; do not edit it without updating data-model.md.
- [ ] T038 [Claude] Add the §C scenarios to `coordinated-live-test-plan.md`.
- [ ] T039 [Claude] In spec 001, add to FR-001 or Assumptions a pointer that every milestone
      email carries the opt-out link and is suppressed for opted-out people (the cross-reference
      already added at planning time covers the Assumptions side).
- [ ] T040 [Claude] Log the pre-existing 200-record scan limit (`checkAllianceCheckExists`,
      `checkDischargeExists`, `checkDiscontinuationExists`, `checkPeriodicCheckInDue`, and
      `submitFeedbackResponse`'s token lookup) as a follow-up for the pipeline as a whole,
      since skip rows add to the table. Not fixed by this feature.
- [ ] T041 [Costin] Ask compliance counsel whether any extra wording or handling is advisable
      for the opt-out, alongside the existing BAA and counsel items (spec Assumptions). Does
      not block the build; counsel's answer may change the email line or thank-you text.

## Dependencies

T001-T008 (CRM) → T009-T012 (functions, in order; T011 needs T001-T003) →
T013-T017 (opt-out path) and T018-T022 (flows; T018 first, then the rest in the listed order)
→ T023-T024 (email links, which need T013/T014 for the permalink and T023 for the wording)
→ T025 → T026-T029 (can run in parallel with Phase 4) → T030-T032 (T030 needs skip rows to
exist) → T033-T034 → T035-T039 (T035 and T036 run alongside the build, not after it) → T040-T041
(independent, any time).

Go-live is gated on: T017, T025, T033, the BAA confirmation (constitution TODO), and Liana's
approvals (T016, T023).
