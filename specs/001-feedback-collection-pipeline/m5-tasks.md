---
description: "Task list for M5 - Discontinuation, generated prospectively before implementation"
---

# Tasks: M5 - Discontinuation (Milestone 5)

**Input**: `m5-plan.md`, `spec.md` (User Stories 1, 2 & 5, User Story 6 /
FR-018), `m5-research.md`, `m5-data-model.md`

**Note**: PROSPECTIVE. Same discipline as M1/M2/M3/M4 (`m1-tasks.md`,
`m2-tasks.md`, `m3-tasks.md`, `m4-tasks.md`): all tasks below start
unchecked; check them off as they're actually done, not in advance.

## Phase 1: Setup

- [ ] T001 Build the public "Cape Clarity Discontinuation Feedback" Zoho
      Form: 1 single-select (dropdown or radio — decide against the live
      builder, `m5-data-model.md`) "reason for leaving" field with the 8
      Confluence-sourced options, 1 Yes/No/Maybe radio ("okay to reach back
      out"), plus a hidden/single-line `Token` field (same prefill-URL
      pattern as every prior form). No name, email, or phone field, per
      Principle I. No open-text field at all, per the Confluence source's
      explicit no-open-text design note — the shortest, most structured form
      of any milestone. Set the form's thank-you-page text to the sourced
      closing line ("Thank you for letting us know...").

## Phase 2: Foundational

- [ ] T002 Confirm the `"5B - Discontinuation, Email Fallback"` picklist
      value on `Milestone_Instances.Milestone` via Zoho CRM MCP `getFields`
      (not browser). **Confirmed 2026-09-24: the value already exists** (full
      picklist recorded in `m5-data-model.md`) — no CRM change needed for
      this task, per `m5-research.md` Decision 6.
- [ ] T002a **Flag for Costin/Liana, do not silently assume**: confirm
      `Patient_Status = "Discontinued (Patient Choice)"` is what staff select
      for a cancellation-with-no-rebooking case, and `Patient_Status = "No
      Show"` for the 2-consecutive-no-show case (`m5-research.md` Decision
      1) — the single highest-risk assumption in this milestone. Build
      proceeds on this best-supported inference regardless (per every prior
      milestone's practice of not blocking on a question that doesn't
      prevent building); this task tracks getting an explicit yes/no before
      the trigger flow is ever switched on, the same treatment M4's Decision
      1 got.
- [ ] T003 Build "M5 - Discontinuation Trigger" flow: watch
      `Patients1.Patient_Status` for `"Discontinued (Patient Choice)"` OR
      `"No Show"`. **First confirm whether the trigger-criteria builder
      supports an OR group on one field** (`m5-research.md` Decision 2); use
      the trigger-level OR filter if supported, otherwise widen the trigger
      filter and add the OR check inside the flow's own `If else` per the
      documented fallback. Build fresh (do not clone M4's or any other
      milestone's flow and edit its function nodes in place —
      `m2-implementation-notes.md` §3.2's shared-custom-function gotcha).
- [ ] T004 Implement `checkDiscontinuationExists(patientId)` per
      `m5-data-model.md`: plain existence check over `Milestone_Instances` at
      `Milestone = "5B - Discontinuation, Email Fallback"` for this patient
      (same shape as `checkDischargeExists`). Confirm exact
      `zoho.crm.getRecords` pagination/list-indexing syntax against the live
      Deluge editor.
- [ ] T005 Wire the shared `isCreatedAfterCutoff` function (feature 004,
      unmodified) into the trigger flow: `createdTime` =
      `${trigger.Created_Time}`, output variable `isCreatedAfterCutoff_1`,
      per `m5-research.md` Decision 4. Confirm all links are genuinely
      connected via the `jsplumb-connected` check (`m3-implementation-notes.md`
      §6, 002's implementation notes §5.1) before attempting to switch the
      flow on.
- [ ] T006 If else — condition per `m5-data-model.md` (2-clause if the OR is
      handled at the trigger level, 3-clause with a nested OR sub-group if
      not). True branch → the shared `issueFeedbackToken` function
      (unmodified) → a native Zoho Mail "Send email" step, per CLAUDE.md's
      no-subflow token-issuance convention — confirm the `milestone`
      parameter value is exactly `"5B - Discontinuation, Email Fallback"` and
      matches the real CRM picklist value (T002). False branch: empty.
- [ ] T007 Add both On Error branches per CLAUDE.md's feature-003 convention,
      from the start (M5 is the second milestone, after M4, built after that
      convention already existed): On Error of `issueFeedbackToken` → Zoho
      Mail alert to `costin@capeclarity.com`; On Error of the patient Send
      email → Zoho CRM "Update module entry" (Status = `Send Failed`,
      already a valid picklist value) → Zoho Mail alert. Alerts must not
      include any `${trigger.*}` identity fields.
- [ ] T007a Confirm at build time whether a first-name-only merge field is
      available for the patient-facing email's `[First Name]` personalization
      (`m5-data-model.md`'s note); if not, use the documented fallback
      (generic salutation-free opening) and record the substitution in
      `m5-implementation-notes.md`.
- [ ] T008 Build "M5 - Discontinuation Write-back" flow: realtime
      Form-submission trigger on T001's form → write-back custom function.
      Build as a brand-new flow (not cloned), same reasoning every prior
      milestone's write-back flow gives for avoiding the shared-custom-
      function gotcha entirely on write-back flows.
- [ ] T009 Implement the write-back function
      `submitDiscontinuationFeedbackResponse`: token lookup, `Status !=
      "Issued"` rejection, expiry check + auto-expire, then concatenate the 2
      answers into `Response_Data` in the field order specified in
      `m5-data-model.md` (Reason For Leaving first, Okay To Reach Back Out
      last/unbounded) — pure string concatenation, no numeric parsing or
      conditional logic, per Constitution Principle VII. No new CRM field
      targets in the update map.

**Checkpoint**: Core pipeline (auto-trigger -> issue -> collect -> rejoin)
functional and idempotent, with zero CRM schema changes, and both On Error
branches present from the first build.

## Phase 3: User Story 1 - Automated milestone trigger and de-identified delivery (Priority: P1)

**Scope for M5**: the one-time discontinuation trigger case (spec.md
Milestones table, row 5; Acceptance Scenario 1), including its two-legged
OR condition.

- [ ] T010 [US1] Verify `Patient_Status` transitioning to either
      `"Discontinued (Patient Choice)"` or `"No Show"` fires the trigger
      exactly once per patient — confirm a second, unrelated update to the
      same already-discontinued patient's record does NOT create a second M5
      instance (`checkDiscontinuationExists` blocks it) and does NOT
      re-issue a feedback request for a pre-cutoff-date patient touched
      incidentally (`isCreatedAfterCutoff` blocks it).
- [ ] T010a [US1] Specifically verify the OR condition itself: a patient
      transitioning to `"Discontinued (Patient Choice)"` fires the trigger,
      AND a separate patient transitioning to `"No Show"` also fires it —
      both legs genuinely work, not just one (this is the first milestone
      whose trigger has more than one firing condition, so it needs its own
      explicit check, distinct from T010's idempotency/cutoff checks).
- [ ] T011 [US1] Confirm no duplicate Discontinuation Feedback invitation is
      ever sent for the same patient regardless of which leg (or both, in
      sequence) fired — satisfies spec.md FR-002 / Acceptance Scenario 2.
- [ ] T012 [US1] Confirm feedback still goes out with zero contractor/
      clinician action required (spec.md Acceptance Scenario 3).

**Checkpoint**: US1 satisfied for M5 as a genuine automatic, one-time,
dual-condition trigger, legacy-patient-safe.

## Phase 4: User Story 2 - De-identified response collection and CRM rejoin (Priority: P1)

- [ ] T013 [US2] Confirm no patient identifier appears in the
      Discontinuation Feedback Form, the flow's parameters, or
      `Response_Data` (2 segments only, both categorical, no freeform at
      all) — Principle I.
- [ ] T014 [US2] Token match + rejoin implemented (T009) — response lands
      only on the matched `Patient`'s CRM record, never anywhere else.
- [ ] T015 [US2] Reuse rejection implemented via `Status != "Issued"` check
      (satisfies spec.md Acceptance Scenario 3 for User Story 2).
- [ ] T016 [US2] **Compliance gap, carried from M0-M4, not resolved here**:
      purge of the raw Zoho Forms submission entry within "a short, defined
      retention window" (spec.md FR-006) is not implemented by this
      milestone either. M5 adds a sixth flow with the identical gap. Do not
      silently accept this gap a sixth time without at least confirming with
      Costin it's still being deferred deliberately — see
      `data-retention-purge.md`.

**Checkpoint**: Core de-identified rejoin works end-to-end for M5; retention
purge remains a known, tracked, cross-milestone gap.

## Phase 5: User Story 5 - Discontinuation exit-reason capture (Priority: P5)

**Scope for M5**: this is M5's own dedicated user story (unlike M2-M4, which
each mapped primarily onto User Story 1/2/4). Verify the milestone-specific
acceptance scenarios directly.

- [ ] T017 [US5] Confirm a patient's `Patient_Status` change to either
      trigger value results in an exit-reason feedback request sent by
      email automatically (spec.md Acceptance Scenario 1) — same
      token-based, de-identified mechanism as every other milestone, no
      separate identity-collecting mechanism introduced (Acceptance Scenario
      2).
- [ ] T018 [US5] Confirm non-response to the discontinuation request is
      visible as such in reporting (T022's status/volume view), not silently
      dropped (spec.md Acceptance Scenario 3 / FR-015) — the same
      non-response-visibility requirement M0-M4's own status/volume reports
      already satisfy for their milestones.

**Checkpoint**: User Story 5 satisfied — M5 is the last of the five active
milestones, so User Story 1 (automated triggers for all milestones) is now
complete pipeline-wide.

## Phase 6: Not applicable — Clinical Safety Flag (User Story 4)

Per `m5-research.md` Decision 7: spec.md's Clinical Safety Flag Rules scope
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

- [ ] T019 Build the 2 new M5-only Analytics formula columns on "Milestone
      Instances" per `m5-data-model.md`: Reason For Leaving (bounded
      `substring_between`), Okay To Reach Back Out (unbounded last-field
      extraction). Confirm both parse a real test submission correctly
      before assuming it (Phase 8).
- [ ] T020 Build a minimal "M5 Submitted Responses" report (both parsed
      columns, raw blob hidden, no `Patient`/identity column, no
      scored/flag columns since none exist for M5) — same pattern as
      M0-M4's, scoped to `Milestone = "5B - Discontinuation, Email
      Fallback"`.
- [ ] T021 [Spec.md User Story 6 / FR-018] Build, at minimum: a status/
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

## Phase 8: Test data & validation

- [ ] T022 Get a test Patient record ID from Costin (per the standing
      Patients-module access restriction — do not look one up via CRM query
      or browser).
- [ ] T023 Using Zoho CRM MCP tools (not browser), exercise the
      discontinuation path for **both** trigger legs: simulate
      `Patient_Status` transitioning to `"Discontinued (Patient Choice)"`
      for one test patient and to `"No Show"` for a second (or the same
      patient reused across two separate test passes, cleaned up between —
      Costin/Liana's call), confirm an M5 instance is created for each and
      the execution shows `issueFeedbackToken` + Send email, not a subflow
      (per `specs/002-remove-subflow-dependency/implementation-notes.md`
      §6's confirmation discipline), confirm a second, unrelated update to
      the same record does not create a duplicate, submit test
      Discontinuation Feedback responses, and confirm a repeat status-set at
      the same value does not create a duplicate.
- [ ] T024 Confirm the Analytics formula columns (T019) parse test
      submissions correctly for M5 rows — both segments match a manual
      evaluation.
- [ ] T025 Test the legacy-patient cutoff-exclusion gate specifically
      (`m5-research.md` Decision 4): confirm a simulated pre-cutoff-date
      test patient whose `Patient_Status` is set to either trigger value
      does NOT trigger issuance, and a post-cutoff test patient does, for
      both legs.
- [ ] T026 Delete test `Milestone_Instances` records via Zoho CRM MCP tools
      when done, same as M0-M4's cleanup pattern.

## Phase 9: Polish & documentation

- [ ] T027 Write `m5-implementation-notes.md` (as-built reference, mirroring
      `m4-implementation-notes.md`'s structure: component inventory, verbatim
      Deluge source for `checkDiscontinuationExists` and
      `submitDiscontinuationFeedbackResponse`, the confirmed 2-segment blob
      format, whether the trigger-level OR or the in-flow-`If else` fallback
      was actually used (T003), the `[First Name]` personalization outcome
      (T007a), the 2 new Analytics formulas, CRM field/picklist reference,
      test-data approach, access constraints) once M5 is actually built —
      per CLAUDE.md's convention, in the same session as the change.
- [ ] T028 Update `CLAUDE.md` if this build reveals a new cross-milestone
      convention worth capturing. Likely candidates: how the trigger-level
      vs. in-flow OR question (T003) actually resolved, for the benefit of
      any future milestone that needs a multi-value trigger condition; and
      whether "reuse an existing-but-differently-labeled picklist value
      as-is" (Decision 6) is worth naming as a convention alongside the
      existing "reuse, don't fork" pattern `issueFeedbackToken` and
      `isCreatedAfterCutoff` already established.
- [ ] T029 Commit `m5-implementation-notes.md` and any plan/data-model
      corrections discovered during implementation, in the same session as
      the change, per the existing convention.
- [ ] T030 Once M5 ships, note in `spec.md` or a follow-up that all five
      active milestones (User Story 1) are now built end-to-end — the
      unified Admin Dashboard (User Story 3) is the natural next piece of
      pipeline-wide work, not a new milestone.

## Notes for whoever implements this

- T002a's trigger-field mapping (`Patient_Status` = `"Discontinued (Patient
  Choice)"` OR `"No Show"`) is the single highest-risk assumption in this
  milestone — higher-risk than M4's own flagged assumption, since it maps
  *two* business conditions onto *two* CRM values inferred by elimination,
  not one obviously-correct value. Get Costin/Liana's confirmation before
  switching the flow on, even though nothing in the build itself is blocked
  on it.
- T003's trigger-level OR question is genuinely new builder territory for
  this project — no prior milestone's trigger needed more than one `equals`
  condition. Confirm against the live builder rather than assuming either
  the OR-group approach or the in-flow-fallback approach works before
  building the rest of the flow around it.
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
