---
description: "Task list for M4 - Discharge, generated prospectively before implementation"
---

# Tasks: M4 - Discharge (Milestone 4)

**Input**: `m4-plan.md`, `spec.md` (User Stories 1, 2 & 4, Clinical Safety
Flag Rules, User Story 6 / FR-018), `m4-research.md`, `m4-data-model.md`

**Note**: PROSPECTIVE. Same discipline as M1/M2/M3 (`m1-tasks.md`,
`m2-tasks.md`, `m3-tasks.md`): all tasks below start unchecked; check them
off as they're actually done, not in advance.

## Phase 1: Setup

- [ ] T001 Build the public "Cape Clarity Discharge Feedback" Zoho Form: 4
      sliders reusing M2/M3's exact Alliance Check-In prompts/instructions/
      range verbatim (Connection, Understanding, Shared direction, Fit of
      approach), plus 1 new "Likelihood to recommend" slider (0-10) and 1
      new optional multi-line "Anything else looking ahead" field, plus a
      hidden/single-line `Token` field (same prefill-URL pattern as every
      prior form). On-screen section order: Alliance Check-In (1), Looking
      Ahead (2). No name, email, or phone field, per Principle I.

## Phase 2: Foundational

- [ ] T002 Confirm/add the `"4 - Discharge"` picklist value on
      `Milestone_Instances.Milestone` via Zoho CRM MCP `getFields` (not
      browser — allowed under the standing access constraint since this
      module carries no PII). **Confirmed 2026-09-23: the value already
      exists** (full picklist recorded in `m4-data-model.md`) — no CRM
      change needed for this task.
- [ ] T002a Confirm the `Patients1.Patient_Status` picklist and flag
      `"Completed Treatment"` for Costin/Liana review as the field/value this
      milestone watches (`m4-research.md` Decision 1) — **confirmed live
      2026-09-23** via Zoho CRM MCP `getFields`; review of whether this is
      the correct discharge marker is still open and does not block the
      remaining build (a one-line filter-value change if the answer differs).
- [ ] T003 Build "M4 - Discharge Trigger" flow: watch
      `Patients1.Patient_Status` with an `equals "Completed Treatment"`
      trigger-level filter. Build fresh (do not clone M2's/M3's flow and
      edit its function nodes in place — `m2-implementation-notes.md` §3.2's
      shared-custom-function gotcha).
- [ ] T004 Implement `checkDischargeExists(patientId)` per
      `m4-data-model.md`: plain existence check over `Milestone_Instances`
      at `Milestone = "4 - Discharge"` for this patient (same shape as
      `checkAllianceCheckExists`, per `m4-research.md` Decision 2). Confirm
      exact `zoho.crm.getRecords` pagination/list-indexing syntax against the
      live Deluge editor.
- [ ] T005 Wire the shared `isCreatedAfterCutoff` function (feature 004,
      unmodified) into the trigger flow: `createdTime` = `${trigger.Created_Time}`,
      output variable `isCreatedAfterCutoff_1`, per `m4-research.md` Decision
      3. Confirm all links are genuinely connected via the
      `jsplumb-connected` check (`m3-implementation-notes.md` §6's
      correction to the plain connector-count check, and 002's
      implementation notes §5.1) before attempting to switch the flow on.
- [ ] T006 If else — condition `checkDischargeExists_1 is false` AND
      `isCreatedAfterCutoff_1 is true`. True branch → the shared
      `issueFeedbackToken` function (unmodified) → a native Zoho Mail "Send
      email" step, per CLAUDE.md's no-subflow token-issuance convention —
      confirm the `milestone` parameter value is exactly `"4 - Discharge"`
      and matches the real CRM picklist value (T002). False branch: empty.
- [ ] T007 Add both On Error branches per CLAUDE.md's feature-003 convention,
      from the start (this is the first milestone built after that
      convention already existed — see `m4-research.md` Decision 8): On
      Error of `issueFeedbackToken` → Zoho Mail alert to
      `costin@capeclarity.com`; On Error of the patient Send email → Zoho
      CRM "Update module entry" (Status = `Send Failed`, already a valid
      picklist value — T002a's `getFields` check also confirmed this) →
      Zoho Mail alert. Alerts must not include any `${trigger.*}` identity
      fields.
- [ ] T008 Build "M4 - Discharge Write-back" flow: realtime Form-submission
      trigger on T001's form → write-back custom function. Build as a
      brand-new flow (not cloned), same reasoning M2's §3.4/M3's §1.6 give
      for avoiding the shared-custom-function gotcha entirely on write-back
      flows.
- [ ] T009 Implement the write-back function
      `submitDischargeFeedbackResponse`: token lookup, `Status != "Issued"`
      rejection, expiry check + auto-expire, then concatenate the 6 answers
      into `Response_Data` **in the field order specified in
      `m4-data-model.md`** (Looking Ahead segments first, alliance domains
      last — NOT the form's on-screen order) — pure string concatenation, no
      numeric parsing or conditional logic, per Constitution Principle VII.
      No new CRM field targets in the update map.

**Checkpoint**: Core pipeline (auto-trigger -> issue -> collect -> rejoin)
functional and idempotent, with zero CRM schema changes, and both On Error
branches present from the first build.

## Phase 3: User Story 1 - Automated milestone trigger and de-identified delivery (Priority: P1)

**Scope for M4**: the one-time discharge trigger case (spec.md Milestones
table, row 4; Acceptance Scenario 1).

- [ ] T010 [US1] Verify `Patient_Status` transitioning to `"Completed
      Treatment"` fires the trigger exactly once per patient — confirm a
      second, unrelated update to the same already-discharged patient's
      record does NOT create a second M4 instance (`checkDischargeExists`
      blocks it) and does NOT re-issue a feedback request for a
      pre-cutoff-date patient touched incidentally (`isCreatedAfterCutoff`
      blocks it) — the specific risk `m4-research.md` Decision 3 documents.
- [ ] T011 [US1] Confirm no duplicate Discharge Feedback invitation is ever
      sent for the same patient (T004's idempotency check) — satisfies
      spec.md FR-002 / Acceptance Scenario 2.
- [ ] T012 [US1] Confirm feedback still goes out with zero contractor/
      clinician action required (spec.md Acceptance Scenario 3).

**Checkpoint**: US1 satisfied for M4 as a genuine automatic, one-time
trigger, legacy-patient-safe.

## Phase 4: User Story 2 - De-identified response collection and CRM rejoin (Priority: P1)

- [ ] T013 [US2] Confirm no patient identifier appears in the Discharge
      Feedback Form, the flow's parameters, or `Response_Data` (6 segments
      only, all scored/categorical or optional freeform) — Principle I.
- [ ] T014 [US2] Token match + rejoin implemented (T009) — response lands
      only on the matched `Patient`'s CRM record, never anywhere else.
- [ ] T015 [US2] Reuse rejection implemented via `Status != "Issued"` check
      (satisfies spec.md Acceptance Scenario 3 for User Story 2).
- [ ] T016 [US2] **Compliance gap, carried from M0/M1/M2/M3, not resolved
      here**: purge of the raw Zoho Forms submission entry within "a short,
      defined retention window" (spec.md FR-006) is not implemented by this
      milestone either. M4 adds a fifth flow with the identical gap. Do not
      silently accept this gap a fifth time without at least confirming with
      Costin it's still being deferred deliberately — see
      `data-retention-purge.md`.

**Checkpoint**: Core de-identified rejoin works end-to-end for M4; retention
purge remains a known, tracked, cross-milestone gap.

## Phase 5: User Story 4 - Automated clinical safety flag to admin (Priority: P4, alliance rule, shared with M2/M3)

**Scope for M4**: apply the same Clinical Safety Flag Rules alliance rule
M2/M3 already built (total ≤20, or any domain ≤4) to M4's reused alliance
domains — by widening the existing formula columns a second time, not
forking a new pair (`m4-research.md` Decision 6).

- [ ] T017 [US4] Widen the existing "Clinical Safety Flag" formula column's
      `Milestone` gate to `'2 - Early Alliance Check' OR '3 - Periodic
      Consolidated' OR '4 - Discharge'` per `m4-data-model.md`. Confirm it
      still fires correctly on every documented M2/M3 test case (re-verify
      those aren't broken by the widened gate) AND on the equivalent M4
      cases (total ≤20 alone, a single domain ≤4 alone, both at once, and a
      reading that clears both thresholds does not fire) — satisfies FR-011
      / Acceptance Scenario 2 for M4. **Update `m2-implementation-notes.md`
      §9.2 AND `m3-implementation-notes.md` §1.1 in the same session**, per
      CLAUDE.md's convention — this changes how both milestones' shipped
      Analytics builds work, not just M4's.
- [ ] T018 [US4] Confirm no automated notification, email, or CRM action of
      any kind reaches a contractor when an M4 reading fires the flag —
      satisfies FR-012 / Acceptance Scenario 3. Confirm-absence task: review
      T009's function, the flow (including both On Error branches, which
      only ever address `costin@capeclarity.com`), and the widened Analytics
      formula columns for any contractor-facing action.
- [ ] T019 [US4] Rename/widen the existing "M2 & M3 Flagged for Review"
      report to "M2, M3 & M4 Flagged for Review" per `m4-data-model.md`
      (Milestone filter widened to Wildcard `"2 - Early Alliance Check"` OR
      `"3 - Periodic Consolidated"` OR `"4 - Discharge"`), confirm a flagged
      M4 record is visible there with the patient identified only by Token
      (no `Patient`/identity column) — satisfies FR-013 for all three
      milestones' data, milestone-scoped per `m2-research.md`'s FR-013
      scope-boundary note (not the unified FR-007 Admin Dashboard). **Update
      `m2-implementation-notes.md` §9.6 AND `m3-implementation-notes.md`
      §1.3 in the same session** — this renames/widens a report both earlier
      milestones' notes already describe.

**Checkpoint**: Alliance-rule clinical safety flagging works end-to-end for
M4, sharing (not duplicating) M2/M3's infrastructure, admin-visible,
contractor-blind, with no CRM schema changes.

## Phase 6: Reporting (User Story 6 / FR-018 baseline bar)

*Full admin dashboard (User Story 3, FR-007–009) remains out of scope for
M4. M4's reporting must clear the FR-018 baseline bar from the start, per
the Rollout Workflow reminder — not just a raw submitted-responses table,
per `m1-implementation-notes.md` §13 / `m2-implementation-notes.md` §9 /
`m3-implementation-notes.md` §1.7–1.8's established pattern.*

- [ ] T020 Build the 2 new M4-only Analytics formula columns on "Milestone
      Instances" per `m4-data-model.md`: Looking Ahead: Likelihood To
      Recommend, Looking Ahead: Anything Else (both bounded
      `substring_between`). Confirm the 4 reused alliance-domain columns and
      Alliance Check-In Total parse M4 rows correctly with no edits, per the
      field-order design in `m4-data-model.md` — a confirmation step, verify
      against a real test submission (Phase 7) before assuming it.
- [ ] T021 Build a minimal "M4 Submitted Responses" report (all 6 parsed
      columns + Alliance Check-In Total + the 2 shared flag columns, raw
      blob hidden, no `Patient`/identity column) — same pattern as
      M0/M1/M2/M3's, scoped to `Milestone = "4 - Discharge"`.
- [ ] T022 [Spec.md User Story 6 / FR-018] Build, at minimum: a status/
      volume view ("M4 Status Breakdown" — Issued/Submitted/Expired counts,
      "Save As" pattern off M2's or M3's equivalent) and at least one
      distribution/summary view of M4's *own new* scored data ("M4 Looking
      Ahead: Likelihood To Recommend Distribution", `Dimension [Actual(D)] /
      Treat as Text` mode per `m2-implementation-notes.md` §9.5's gotcha, if
      needed). Bundle T021, T022's two new reports, and T019's shared "M2,
      M3 & M4 Flagged for Review" into a new "M4 - Discharge Feedback"
      dashboard (built via "Create New Dashboards", not "Save As" off a
      mismatched-panel dashboard).

## Phase 7: Test data & validation

- [ ] T023 Get a test Patient record ID from Costin (per the standing
      Patients-module access restriction — do not look one up via CRM query
      or browser).
- [ ] T024 Using Zoho CRM MCP tools (not browser), exercise the discharge
      path: simulate `Patient_Status` transitioning to `Completed Treatment`
      for the test patient (confirm an M4 instance is created and the
      execution shows `issueFeedbackToken` + Send email, not a subflow — per
      `specs/002-remove-subflow-dependency/implementation-notes.md` §6's
      confirmation discipline), confirm a second, unrelated update to the
      same record does not create a duplicate, submit test Discharge
      Feedback responses covering a flag-triggering case and a clean case,
      and confirm a repeat status-set at the same value does not create a
      duplicate.
- [ ] T025 Confirm the Analytics formula columns (T020, plus the widened
      T017 flag columns) parse test submissions correctly for M4 rows — each
      of the 6 segments, the total, and both flag columns match a manual
      evaluation — and confirm M2's/M3's existing test/sample data (if any
      exists by then) still evaluates correctly under the widened Milestone
      gate.
- [ ] T026 Test the legacy-patient cutoff-exclusion gate specifically
      (`m4-research.md` Decision 3): confirm a simulated pre-cutoff-date
      test patient whose `Patient_Status` is set to `Completed Treatment`
      does NOT trigger issuance, and a post-cutoff test patient does.
- [ ] T027 Delete test `Milestone_Instances` records via Zoho CRM MCP tools
      when done, same as M0/M1/M2/M3's cleanup pattern.

## Phase 8: Polish & documentation

- [ ] T028 Write `m4-implementation-notes.md` (as-built reference, mirroring
      `m3-implementation-notes.md`'s structure: component inventory,
      verbatim Deluge source for `checkDischargeExists` and
      `submitDischargeFeedbackResponse`, the confirmed 6-segment blob format
      and its field order, the 2 new + 2 widened Analytics formulas, CRM
      field/picklist reference, test-data approach, access constraints) once
      M4 is actually built — per CLAUDE.md's convention, in the same session
      as the change.
- [ ] T029 Update `m2-implementation-notes.md` AND `m3-implementation-notes.md`
      for every change T017/T019 made to their shared Analytics objects (the
      widened Clinical Safety Flag gate; the renamed/widened Flagged for
      Review report) — **in the same session as those changes**, not
      deferred to T028's M4-only notes file.
- [ ] T030 Update `CLAUDE.md` if this build reveals a new cross-milestone
      convention worth capturing. Likely candidates: confirming the "one
      shared function, reused by every subsequent milestone" pattern
      (`issueFeedbackToken`, `isCreatedAfterCutoff`) continues to hold with
      zero drift now that a third and fourth milestone both consume the same
      objects; and whether "widen the shared flag/report a second time"
      needs its own named convention section now that it's happened twice
      (M3 then M4).
- [ ] T031 Commit `m4-implementation-notes.md`, the `m2-implementation-notes.md`
      and `m3-implementation-notes.md` updates, and any plan/data-model
      corrections discovered during implementation, in the same session as
      the change, per the existing convention.

## Notes for whoever implements this

- T002a's trigger-field mapping (`Patient_Status = "Completed Treatment"`)
  is the single highest-risk assumption in this milestone, unlike M3 where
  the highest-risk piece was new idempotency logic — M4's idempotency logic
  is a direct, low-risk copy of an established pattern. Get Costin/Liana's
  confirmation before switching the flow on, even though nothing in the
  build itself is blocked on it.
- T007's On Error branches are new work relative to M0/M2/M3's *original*
  builds (which got them retrofitted by feature 003) but are not new
  *pattern* work — follow `specs/003-issuance-privacy-and-failure-handling/implementation-notes.md`
  §4's gotchas (drop zones during a palette drag show explicit "ON ERROR /
  DROP HERE" boxes; hover and click, don't rely on a synthetic mouseup)
  exactly as that feature's own build did.
- T017/T019 modify M2- and M3-owned Analytics objects for a second time.
  Treat this the same as any other cross-milestone change: verify M2's and
  M3's own documented test cases still hold after the change, and update
  both files' notes in the same session (T029) — do not leave either file
  describing formulas or a report that no longer match what's actually in
  the workspace.
- T016's retention gap is real and open across the whole pipeline, not just
  M4 — see `data-retention-purge.md` before assuming it's someone else's
  problem to solve later.
