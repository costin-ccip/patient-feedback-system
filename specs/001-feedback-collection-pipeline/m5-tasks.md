---
description: "Task list for M5 - Discontinuation, generated prospectively before implementation"
---

# Tasks: M5 - Discontinuation (Milestone 5)

> **Reconciled 2026-09-30** against the milestone's implementation-notes file and git history. T001-T009, T007a, T019-T021 are built; remaining open items are live verification, the retention-purge gap, and the no-show-leg decision. Notes §13 supersedes the original 2-question form (now 1 question).

**Input**: `m5-plan.md`, `spec.md` (User Stories 1, 2 & 5, User Story 6 /
FR-018), `m5-research.md`, `m5-data-model.md`

**Note**: PROSPECTIVE. Same discipline as M1/M2/M3/M4 (`m1-tasks.md`,
`m2-tasks.md`, `m3-tasks.md`, `m4-tasks.md`): all tasks below start
unchecked; check them off as they're actually done, not in advance.

**Scope decision (Costin, 2026-09-24)**: this milestone's trigger covers the
cancellation-with-no-rebooking leg only (`Patient_Status = "Discontinued
(Patient Choice)"`). The `No Show` leg from spec.md's Milestones-table row 5
is explicitly out of scope for this build — see `m5-research.md` Decision 1
and its "Open item for spec.md" note. Every task below reflects a
single-condition trigger, the same shape as M4's.

**Review checkpoints (Costin, 2026-09-24)**: pause for explicit review at
four points, in addition to (not instead of) the standing "no live/
end-to-end test without Costin" rule:

1. **Checkpoint 1 — specs ready and pushed.** Covers Phases 1-2 of this
   document being written and `m5-research.md`/`m5-data-model.md`/
   `m5-plan.md`/`m5-tasks.md` committed and pushed to GitHub.
2. **Checkpoint 2 — after the Zoho Form is built.** After Phase 1/T001.
3. **Checkpoint 3 — after the Zoho Flows are built.** After Phase 2's
   trigger flow and write-back flow (T003-T009).
4. **Checkpoint 4 — after the Analytics dashboard is built.** After Phase 7
   (T019-T021).

Do not proceed past a checkpoint into the next phase of Zoho building
without Costin's review at that checkpoint.

## Phase 1: Setup

- [x] T001 Build the public "Cape Clarity Discontinuation Feedback" Zoho
      Form: 1 single-select (dropdown or radio — decide against the live
      builder, `m5-data-model.md`) "reason for leaving" field with the 8
      Confluence-sourced options, 1 Yes/No/Maybe radio ("okay to reach back
      out"), plus a hidden/single-line `Token` field (same prefill-URL
      pattern as every prior form). No name, email, or phone field, per
      Principle I. No open-text field at all, per the Confluence source's
      explicit no-open-text design note — the shortest, most structured form
      of any milestone. Set the form's thank-you-page text to the sourced
      closing line ("Thank you for letting us know...").
      *Reconciled 2026-09-30: built (`m5-implementation-notes.md` §1). NOTE: the form was later simplified (§13): 'Okay to reach back out' removed, form is now a single question. §13 supersedes the field list above.*

**→ Checkpoint 2: pause here for Costin's review of the built form before
starting Phase 2.**

## Phase 2: Foundational

- [x] T002 Confirm the `"5B - Discontinuation, Email Fallback"` picklist
      value on `Milestone_Instances.Milestone` via Zoho CRM MCP `getFields`
      (not browser). **Confirmed 2026-09-24: the value already exists** (full
      picklist recorded in `m5-data-model.md`) — no CRM change needed for
      this task, per `m5-research.md` Decision 5.
      *Reconciled 2026-09-30: value exists (confirmed 2026-09-24).*
- [x] T002a **Flag for Costin/Liana, do not silently assume**: confirm
      `Patient_Status = "Discontinued (Patient Choice)"` is what staff select
      for the cancellation-with-no-rebooking case this milestone targets
      (`m5-research.md` Decision 1) — the single highest-risk assumption in
      this milestone. Build proceeds on this best-supported inference
      regardless (per every prior milestone's practice of not blocking on a
      question that doesn't prevent building); this task tracks getting an
      explicit yes/no before the trigger flow is ever switched on, the same
      treatment M4's Decision 1 got.
      *Reconciled 2026-10-01: confirmed by Costin: `Discontinued (Patient Choice)` is the status staff use for this case.*
- [x] T003 Build "M5 - Discontinuation Trigger" flow: watch
      `Patients1.Patient_Status` for `equals "Discontinued (Patient Choice)"`
      — a single trigger-level filter condition, same shape as M4's trigger.
      Build fresh (do not clone M4's or any other milestone's flow and edit
      its function nodes in place — `m2-implementation-notes.md` §3.2's
      shared-custom-function gotcha).
      *Reconciled 2026-09-30: built (§2).*
- [x] T004 Implement `checkDiscontinuationExists(patientId)` per
      `m5-data-model.md`: plain existence check over `Milestone_Instances` at
      `Milestone = "5B - Discontinuation, Email Fallback"` for this patient
      (same shape as `checkDischargeExists`). Confirm exact
      `zoho.crm.getRecords` pagination/list-indexing syntax against the live
      Deluge editor.
      *Reconciled 2026-09-30: built (§2).*
- [x] T005 Wire the shared `isCreatedAfterCutoff` function (feature 004,
      unmodified) into the trigger flow: `createdTime` =
      `${trigger.Created_Time}`, output variable `isCreatedAfterCutoff_1`,
      per `m5-research.md` Decision 3. Confirm all links are genuinely
      connected via the `jsplumb-connected` check (`m3-implementation-notes.md`
      §6, 002's implementation notes §5.1) before attempting to switch the
      flow on.
      *Reconciled 2026-09-30: built (§2).*
- [x] T006 If else — condition: `checkDiscontinuationExists_1 is false` AND
      `isCreatedAfterCutoff_1 is true`, same 2-clause shape as M2/M3/M4's `If
      else` nodes. True branch → the shared `issueFeedbackToken` function
      (unmodified) → a native Zoho Mail "Send email" step, per CLAUDE.md's
      no-subflow token-issuance convention — confirm the `milestone`
      parameter value is exactly `"5B - Discontinuation, Email Fallback"` and
      matches the real CRM picklist value (T002). False branch: empty.
      *Reconciled 2026-09-30: built (§2).*
- [x] T007 Add both On Error branches per CLAUDE.md's feature-003 convention,
      from the start (M5 is the second milestone, after M4, built after that
      convention already existed): On Error of `issueFeedbackToken` → Zoho
      Mail alert to `costin@capeclarity.com`; On Error of the patient Send
      email → Zoho CRM "Update module entry" (Status = `Send Failed`,
      already a valid picklist value) → Zoho Mail alert. Alerts must not
      include any `${trigger.*}` identity fields.
      *Reconciled 2026-09-30: built, both On Error branches (§2).*
- [x] T007a Confirm at build time whether a first-name-only merge field is
      available for the patient-facing email's `[First Name]` personalization
      (`m5-data-model.md`'s note); if not, use the documented fallback
      (generic salutation-free opening) and record the substitution in
      `m5-implementation-notes.md`.
      *Reconciled 2026-09-30: done: `[First Name]` fallback confirmed live (§2.1).*
- [x] T008 Build "M5 - Discontinuation Write-back" flow: realtime
      Form-submission trigger on T001's form → write-back custom function.
      Build as a brand-new flow (not cloned), same reasoning every prior
      milestone's write-back flow gives for avoiding the shared-custom-
      function gotcha entirely on write-back flows.
      *Reconciled 2026-09-30: built (§3).*
- [x] T009 Implement the write-back function
      `submitDiscontinuationFeedbackResponse`: token lookup, `Status !=
      "Issued"` rejection, expiry check + auto-expire, then concatenate the 2
      answers into `Response_Data` in the field order specified in
      `m5-data-model.md` (Reason For Leaving first, Okay To Reach Back Out
      last/unbounded) — pure string concatenation, no numeric parsing or
      conditional logic, per Constitution Principle VII. No new CRM field
      targets in the update map.
      *Reconciled 2026-09-30: built (§3). NOTE: edited in §13.2 to drop `okayToReachBackOut`; §13.2 is current.*

**Checkpoint**: Core pipeline (auto-trigger -> issue -> collect -> rejoin)
functional and idempotent, with zero CRM schema changes, and both On Error
branches present from the first build.

**→ Checkpoint 3: pause here for Costin's review of both built flows before
starting Phase 7 (Analytics).**

## Phase 3: User Story 1 - Automated milestone trigger and de-identified delivery (Priority: P1)

**Scope for M5**: the one-time discontinuation trigger case, cancellation
leg only (spec.md Milestones table, row 5; Acceptance Scenario 1).

- [ ] T010 [US1] Verify `Patient_Status` transitioning to `"Discontinued
      (Patient Choice)"` fires the trigger exactly once per patient —
      confirm a second, unrelated update to the same already-discontinued
      patient's record does NOT create a second M5 instance
      (`checkDiscontinuationExists` blocks it) and does NOT re-issue a
      feedback request for a pre-cutoff-date patient touched incidentally
      (`isCreatedAfterCutoff` blocks it).
- [ ] T011 [US1] Confirm no duplicate Discontinuation Feedback invitation is
      ever sent for the same patient — satisfies spec.md FR-002 / Acceptance
      Scenario 2.
- [ ] T012 [US1] Confirm feedback still goes out with zero contractor/
      clinician action required (spec.md Acceptance Scenario 3).

**Checkpoint**: US1 satisfied for M5 as a genuine automatic, one-time
trigger, legacy-patient-safe, for the cancellation leg.

## Phase 4: User Story 2 - De-identified response collection and CRM rejoin (Priority: P1)

- [ ] T013 [US2] Confirm no patient identifier appears in the
      Discontinuation Feedback Form, the flow's parameters, or
      `Response_Data` (2 segments only, both categorical, no freeform at
      all) — Principle I.
- [x] T014 [US2] Token match + rejoin implemented (T009) — response lands
      only on the matched `Patient`'s CRM record, never anywhere else.
      *Reconciled 2026-09-30: implemented in `submitDiscontinuationFeedbackResponse` (§3, §13.2).*
- [x] T015 [US2] Reuse rejection implemented via `Status != "Issued"` check
      (satisfies spec.md Acceptance Scenario 3 for User Story 2).
      *Reconciled 2026-09-30: implemented via the `Status != "Issued"` check (§3).*
- [ ] T016 [US2] **Compliance gap, carried from M0-M4, not resolved here**:
      purge of the raw Zoho Forms submission entry within "a short, defined
      retention window" (spec.md FR-006) is not implemented by this
      milestone either. M5 adds a sixth flow with the identical gap. Do not
      silently accept this gap a sixth time without at least confirming with
      Costin it's still being deferred deliberately — see
      `data-retention-purge.md`.
      *Reconciled 2026-09-30: decision made 2026-10-01: manual periodic deletion by Costin (`data-retention-purge.md`). Stays open until the cadence is set and the click-path is documented.*

**Checkpoint**: Core de-identified rejoin works end-to-end for M5; retention
purge remains a known, tracked, cross-milestone gap.

## Phase 5: User Story 5 - Discontinuation exit-reason capture (Priority: P5)

**Scope for M5**: this is M5's own dedicated user story (unlike M2-M4, which
each mapped primarily onto User Story 1/2/4). Verify the milestone-specific
acceptance scenarios directly, for the cancellation leg this build covers.

- [ ] T017 [US5] Confirm a patient's `Patient_Status` change to
      `"Discontinued (Patient Choice)"` results in an exit-reason feedback
      request sent by email automatically (spec.md Acceptance Scenario 1) —
      same token-based, de-identified mechanism as every other milestone, no
      separate identity-collecting mechanism introduced (Acceptance Scenario
      2).
      *Reconciled 2026-09-30: partly verified: Costin received the discontinuation email in a live test on 9/26 (§10). End-to-end write-back verification is still pending.*
- [ ] T018 [US5] Confirm non-response to the discontinuation request is
      visible as such in reporting (T021's status/volume view), not silently
      dropped (spec.md Acceptance Scenario 3 / FR-015) — the same
      non-response-visibility requirement M0-M4's own status/volume reports
      already satisfy for their milestones.

**Checkpoint**: User Story 5 satisfied for the cancellation leg. Note: per
`m5-research.md`'s "Open item for spec.md," this does not fully close out
spec.md's Milestones-table row 5 as written (the no-show leg remains
unbuilt, by Costin's explicit scope decision) — flagged, not silently
accepted.

## Phase 6: Not applicable — Clinical Safety Flag (User Story 4)

Per `m5-research.md` Decision 6: spec.md's Clinical Safety Flag Rules scope
the Alliance rule to Milestones 2, 3, 4 explicitly. M5 has no alliance
domains or any other scored data. No tasks in this phase — this section
exists only to record, the same way `m5-plan.md`'s Constitution Check does,
that this was checked and found not applicable, not silently skipped.

## Phase 7: Reporting (User Story 6 / FR-018 baseline bar)

*Full admin dashboard (User Story 3, FR-007-009) remains out of scope for
M5. M5's reporting must clear the FR-018 baseline bar from the start, per
the Rollout Workflow reminder — not just a raw submitted-responses table,
per `m1-implementation-notes.md` §13 / `m2-implementation-notes.md` §9 /
`m3-implementation-notes.md` §1.7-1.8 / `m4-implementation-notes.md` §5's
established pattern. M5 is the first milestone to clear this bar with
categorical, not numeric, distribution data — FR-018/Acceptance Scenario 2's
"numeric **or categorical**" wording applies literally here for the first
time.*

- [x] T019 Build the 2 new M5-only Analytics formula columns on "Milestone
      Instances" per `m5-data-model.md`: Reason For Leaving (bounded
      `substring_between`), Okay To Reach Back Out (unbounded last-field
      extraction). Confirm both parse a real test submission correctly
      before assuming it (Phase 8).
      *Reconciled 2026-09-30: built (§4). NOTE: the 'Okay To Reach Back Out' column was deleted in §13.3; only Reason For Leaving remains.*
- [x] T020 Build a minimal "M5 Submitted Responses" report (both parsed
      columns, raw blob hidden, no `Patient`/identity column, no
      scored/flag columns since none exist for M5) — same pattern as
      M0-M4's, scoped to `Milestone = "5B - Discontinuation, Email
      Fallback"`.
      *Reconciled 2026-09-30: built (§4).*
- [x] T021 [Spec.md User Story 6 / FR-018] Build, at minimum: a status/
      volume view ("M5 Status Breakdown" — Issued/Submitted/Expired counts,
      "Save As" pattern off M4's or any prior milestone's equivalent) and at
      least one distribution/summary view of M5's own categorical data ("M5
      Reason For Leaving Distribution", `Dimension [Actual(D)] / Treat as
      Text` mode per `m2-implementation-notes.md` §9.5's gotcha, if needed).
      Optionally also build "M5 Okay To Reach Back Out Breakdown" (not
      required to clear the FR-018 bar, which needs only one distribution
      view, but low-cost to add). Bundle T020, T021's report(s) into a new
      "M5 - Discontinuation Feedback" dashboard (built via "Create New
      Dashboards", not "Save As" off a mismatched-panel dashboard; no shared
      Flagged for Review panel to include, per Phase 6).
      *Reconciled 2026-09-30: built (§4): status/volume + Reason For Leaving distribution + dashboard, so M5 clears FR-018. The optional 'Okay To Reach Back Out Breakdown' was deleted in §13.3.*

**→ Checkpoint 4: pause here for Costin's review of the Analytics
reporting/dashboard before Phase 8 (live test data), which needs Costin's
involvement regardless per the standing no-live-test rule.**

## Phase 8: Test data & validation

- [x] T022 Get a test Patient record ID from Costin (per the standing
      Patients-module access restriction — do not look one up via CRM query
      or browser).
      *Reconciled 2026-09-30: open: blocked on a test Patient record ID from Costin.*
      *Resolved 2026-10-01: Costin designated test patient PT000 (Patients1 record ID 6825601000004448028) for all live tests; see `CLAUDE.md`.*
- [ ] T023 Using Zoho CRM MCP tools (not browser), exercise the
      discontinuation path: simulate `Patient_Status` transitioning to
      `"Discontinued (Patient Choice)"` for the test patient (confirm an M5
      instance is created and the execution shows `issueFeedbackToken` +
      Send email, not a subflow — per
      `specs/002-remove-subflow-dependency/implementation-notes.md` §6's
      confirmation discipline), confirm a second, unrelated update to the
      same record does not create a duplicate, submit a test Discontinuation
      Feedback response, and confirm a repeat status-set at the same value
      does not create a duplicate.
- [ ] T024 Confirm the Analytics formula columns (T019) parse the test
      submission correctly for the M5 row — both segments match a manual
      evaluation.
- [ ] T025 Test the legacy-patient cutoff-exclusion gate specifically
      (`m5-research.md` Decision 3): confirm a simulated pre-cutoff-date
      test patient whose `Patient_Status` is set to `"Discontinued (Patient
      Choice)"` does NOT trigger issuance, and a post-cutoff test patient
      does.
- [ ] T026 Delete test `Milestone_Instances` records via Zoho CRM MCP tools
      when done, same as M0-M4's cleanup pattern.

## Phase 9: Polish & documentation

- [x] T027 Write `m5-implementation-notes.md` (as-built reference, mirroring
      `m4-implementation-notes.md`'s structure: component inventory, verbatim
      Deluge source for `checkDiscontinuationExists` and
      `submitDiscontinuationFeedbackResponse`, the confirmed 2-segment blob
      format, the `[First Name]` personalization outcome (T007a), the 2 new
      Analytics formulas, CRM field/picklist reference, test-data approach,
      access constraints) once M5 is actually built — per CLAUDE.md's
      convention, in the same session as the change.
      *Reconciled 2026-09-30: `m5-implementation-notes.md` exists and is current through §14.*
- [x] T028 Update `CLAUDE.md` if this build reveals a new cross-milestone
      convention worth capturing. Likely candidate: whether "reuse an
      existing-but-differently-labeled picklist value as-is" (Decision 5) is
      worth naming as a convention alongside the existing "reuse, don't
      fork" pattern `issueFeedbackToken` and `isCreatedAfterCutoff` already
      established.
      *Reconciled 2026-09-30: done: the Analytics-deletion convention from this milestone's cleanup was added to `CLAUDE.md` (2026-09-28).*
- [x] T029 Commit `m5-implementation-notes.md` and any plan/data-model
      corrections discovered during implementation, in the same session as
      the change, per the existing convention.
      *Reconciled 2026-09-30: committed (see git history).*
- [x] T030 Once M5 ships, note that four of spec.md's five active milestones
      (0, 2, 3, 4) are fully built and M5 is built for its cancellation leg;
      the no-show leg remains open per `m5-research.md`'s "Open item for
      spec.md" — surface this to Costin rather than treating User Story 1 as
      fully closed pipeline-wide.
      *Reconciled 2026-10-01: decided: cancellation-only; spec.md Milestones row 5 amended (eighth pass). The no-show leg is not part of this version.*

- [x] T031 Delete the dead `5A - Discontinuation, Live Capture` picklist value (and the
      `Staff Live Entry` capture method / `Captured Live` status leftovers) from
      `Milestone_Instances` in CRM. Decided 2026-10-01 (Costin). Needs CRM Setup (UI);
      first check no existing record uses these values.
      *Done 2026-10-01 by Costin in CRM Setup (confirmed in chat). Pre-check: no `Milestone_Instances` record used the `5A` Milestone or `Captured Live` Status values.*

## Notes for whoever implements this

- T002a's trigger-field mapping (`Patient_Status = "Discontinued (Patient
  Choice)"`) is the assumption to get Costin/Liana's confirmation on before
  switching the flow on, even though nothing in the build itself is blocked
  on it — same treatment M4's own Decision 1 got.
- T007's On Error branches and T004/T005's idempotency/cutoff-gate wiring
  are not new *pattern* work — follow the same gotchas
  `specs/003-issuance-privacy-and-failure-handling/implementation-notes.md`
  §4 and `specs/004-legacy-patient-exclusion/implementation-notes.md`
  document, exactly as M4's build did.
- T016's retention gap is real and open across the whole pipeline, not just
  M5 — see `data-retention-purge.md` before assuming it's someone else's
  problem to solve later.
- Unlike M0-M4, this milestone's survey/email copy (T001, T006's email body)
  is sourced verbatim from Confluence, not drafted — there should be no
  "not yet reviewed by Costin/Liana" caveat needed for the *content* itself,
  only for the two build-time judgment calls this file already flags
  (T002a's field mapping, T007a's `[First Name]` fallback).
- Do not skip the four review checkpoints listed at the top of this file —
  they're Costin's explicit process request for this milestone, distinct
  from (and in addition to) the standing no-live-test rule.
