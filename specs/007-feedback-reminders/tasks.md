---
description: "Task list for 007 - Feedback Reminders"
---

# Tasks: Feedback Reminders

**Input**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `quickstart.md`

**Status 2026-10-09: ON HOLD.** Costin paused this feature for the foreseeable future and will
revisit it based on response rates. Do not start any task below until he lifts the hold. The one CRM
field exists (T001); nothing else is built, and no flow has been created or switched ON. Open
items when resumed: T003 (field rename), T005 (Flow task budget), Liana's wording approval (T016),
BAA confirmation (T020).

**Who does what**: tasks marked **[Costin]** need his hands or approval. Tasks marked
**[Claude]** can be done by a session with browser or CRM access once he approves. Live tests are
always his go-ahead (`CLAUDE.md`). Nothing here switches a flow ON without his say-so.

**Test records**: patient PT000 (`Patients1` `6825601000004448028`) and the existing test lead.
No other patient or lead record is read or changed.

## Phase 1: CRM setup

- [x] T001 [Claude] Create `Reminder_Sent_Date_Time` (datetime) on `Milestone_Instances`.
      Done 2026-10-09, field id `6825601000004791001`, with Costin's freed slot.
- [ ] T002 [Costin] Confirm the field appears in the `Milestone_Instances` layout and in the Flow
      field picker. Add it to the layout if not.
- [ ] T003 [Costin] Rename the field label to "Reminder Dispatched Date Time" in CRM Setup
      (decided 2026-10-09). The API name `Reminder_Sent_Date_Time` does not change. No
      field-update tool is available to Claude sessions, so this is a manual rename.
- [ ] T004 [Claude] Read-only `getFields` on `Milestone_Instances` to confirm the API name,
      type and that it is writable via API. (Non-PII module, MCP only.)
- [ ] T005 [Costin] Confirm the current monthly Flow task usage against the 5,000 limit and
      choose the run frequency (plan open question 2, the one still open). Default: hourly,
      9 AM to 5 PM Eastern.

## Phase 2: Spec 001 update (done with this drafting session)

- [x] T006 [Claude] Ninth-pass revision note, FR-002 clarification and Assumptions entry in
      `specs/001-feedback-collection-pipeline/spec.md`.

## Phase 3: The function

- [ ] T007 [Costin or Claude with browser] Create custom function `claimNextDueReminder` in
      the Flow function editor from data-model.md section 3, under the existing
      `crm_connection`.
- [ ] T008 [same] Execute-check the Deluge items on the "confirm" list: timezone `toString`,
      COQL via `invokeurl` and its datetime literal, `is null` on a datetime, `addDay`.
      Fix the source and update data-model.md with what actually worked.
- [ ] T009 [same] Confirm `crm_connection` can run COQL (read scope). If not, re-authorize the
      connection with the COQL read scope (Costin).
- [ ] T010 [Claude or Costin] Run quickstart section A rows 1 to 14 with seeded rows; record
      results in `implementation-notes.md`.

## Phase 4: The flow (build OFF)

- [ ] T011 [Costin or Claude with browser] Create flow "Feedback Reminder - Send" in the
      "Customer Feedback System" folder: Schedule trigger, function step, If else on
      `found`, milestone Decision, five Zoho Mail "Send email" steps (connection "Connection to
      info@capeclarity.com"), per data-model.md section 4.
- [ ] T012 [same] On Error branches: one on the function, one on each Send email; alerts to
      costin@capeclarity.com with milestone and `instanceId` only (no email, no token, no
      `${trigger.*}`), as feature 003.
- [ ] T013 [same] Add the feature 006 stop-emails line to each reminder body using
      `${claimNextDueReminder_1.token}`.
- [ ] T014 [same] Verify every node is wired (`jsplumb-connected` on endpoints) and the flow is
      OFF. Run quickstart section B steps 1, 2 and 4 (no live email).

## Phase 5: Wording

- [ ] T015 [Claude] Put the five drafts in data-model.md section 5 through the
      `liana-tov-rewrite` skill and revise.
- [ ] T016 [Costin, Liana] Approve each milestone's wording (FR-016). Paste approved text into
      the five Send email steps and record the final text in `implementation-notes.md`.
- [ ] T017 [Claude] Read-only look at one old test `Milestone_Instances` row to confirm an
      unanswered `Issued` row past expiry is still `Issued` (research section 1).

## Phase 6: Reporting

- [ ] T018 [Costin or Claude with browser] After the field syncs (daily at 5:00 PM EDT, or a
      manual sync), add the `Reminded` and `Response Outcome` formula columns (data-model
      section 6).
- [ ] T019 [same] Build the Reminder Effectiveness report and a dashboard panel; audit the five
      existing Status Breakdown reports for how they treat `Issued` past expiry and note any that
      mislead (change them in a separate task, not here).

## Phase 7: Test and go-live gate

- [ ] T020 [Costin] Confirm Zoho BAA coverage for Flow, Mail and CRM use in this flow (spec FR-013,
      constitution TODO(BAA_SCHEDULE)). Record it. No go-live before this.
- [ ] T021 [Costin] Run or approve quickstart section C (live run with PT000).
- [ ] T022 [Costin] Decide whether to switch the flow ON for real data after the legacy
      cutoff and BAA gates are clear.

## Phase 8: Documentation (same session as each change)

- [ ] T023 [Claude] Create `specs/007-feedback-reminders/implementation-notes.md` when the build
      starts; keep it current per `CLAUDE.md`.
- [ ] T024 [Claude] After build, add a short "Reminder convention" section to the repo-root
      `CLAUDE.md` (shared function, one reminder, same token, never call `issueFeedbackToken`),
      and a reminder step to `coordinated-live-test-plan.md`.
- [ ] T025 [Claude] Add a one-line pointer to the reminder to each milestone's implementation
      notes that describes the initial email (M0, M2, M3, M4, M5).

## Dependencies

T001 before everything. T007 to T010 before T011. T015 and T016 before the flow carries real
wording. T018 needs the field to have synced. T020 gates T022. Phases 3 and 5 can run in
parallel.
