---
description: "Task list for M3 - Periodic Consolidated Check-In (every 8th session), generated prospectively before implementation"
---

# Tasks: M3 - Periodic Consolidated Check-In (Milestone 3)

**Input**: `m3-plan.md`, `spec.md` (User Stories 1, 2 & 4, Clinical Safety
Flag Rules, User Story 6 / FR-018), `m3-research.md`, `m3-data-model.md`

**Note**: PROSPECTIVE. Same discipline as M1/M2 (`m1-tasks.md`,
`m2-tasks.md`): all tasks below start unchecked; check them off as they're
actually done, not in advance.

## Phase 1: Setup

- [x] T001 Build the public "Cape Clarity Periodic Check-In" Zoho Form: 4
      sliders reusing M2's exact Alliance Check-In prompts/instructions/
      range verbatim (Connection, Understanding, Shared direction, Fit of
      approach), plus 2 new Practice Experience sliders
      (Scheduling/Communication; Billing, both 0-10) and 1 new Therapist
      Professionalism slider (0-10), plus a hidden/single-line `Token`
      field (same prefill-URL pattern as every prior form). On-screen
      section order per spec.md's fifth-pass note: Alliance Check-In (1),
      Practice Experience (2), Therapist Professionalism (3). No name,
      email, or phone field, per Principle I.

## Phase 2: Foundational

- [x] T002 Build "M3 - Periodic Check-In Trigger" flow: watch
      `Patients1.Session_Count` with a `>= 8` trigger-level filter (not an
      exact-value filter — see `m3-research.md` Decision 1), feeding a new
      idempotency/eligibility custom function rather than a plain existence
      check. Build fresh (do not clone M2's flow and edit its function node
      in place — `m2-implementation-notes.md` §3.2's shared-custom-function
      gotcha). Verify all links are genuinely connected via the DOM
      connector-element check (`[class*="connector" i]`) before attempting
      to switch it on.
- [x] T003 Implement `checkPeriodicCheckInDue(patientId, sessionCount)` per
      `m3-data-model.md`: fixed checkpoint list `[8, 16, 24, 32, 40]`
      (Costin, 2026-09-17 — extend this one list later rather than changing
      the logic), count existing `Milestone_Instances` rows for this
      patient at `Milestone = "3 - Periodic Consolidated"`, look up
      `nextThreshold = checkpoints[existingCount]` (return `false` once
      `existingCount` reaches the end of the list), return
      `sessionCount >= nextThreshold`. Confirm exact `zoho.crm.getRecords`
      pagination/list-indexing syntax against the live Deluge editor —
      satisfies spec.md FR-002 (no duplicate request per checkpoint) for a
      recurring milestone.
- [x] T004 On eligible (`checkPeriodicCheckInDue` true): create a new
      `Milestone_Instances` record (`Patient` = triggering patient,
      `Milestone = "3 - Periodic Consolidated"`, `Status = "Issued"`, fresh
      `Token`, `Expiry_Date_Time` — reuse M0/M1/M2's 7-day TTL unless Costin
      specifies otherwise) and send the Periodic Check-In link via Zoho
      Mail, through the unchanged shared "Subflow - Issue Feedback Token" —
      confirm the `milestone` parameter value is exactly
      `"3 - Periodic Consolidated"` and matches a real CRM picklist value
      (see T000-equivalent picklist check below).
- [x] T004a Confirm/add the `"3 - Periodic Consolidated"` picklist value on
      `Milestone_Instances.Milestone` via Zoho CRM MCP `getFields`/field-
      update tools (not browser — allowed under the standing access
      constraint since this module carries no PII). **Confirmed 2026-09-17:
      the value already exists** (full picklist recorded in
      `m3-data-model.md`) — no CRM change needed for this task.
- [ ] T005 Build "M3 - Periodic Check-In Write-back" flow: realtime
      Form-submission trigger on T001's form → write-back custom function.
      Build as a brand-new flow (not cloned), same reasoning M2's §3.4 gives
      for avoiding the shared-custom-function gotcha entirely on write-back
      flows. Confirm the trigger is genuinely wired to the function node
      before considering this done.
- [ ] T006 Implement the write-back function `submitPeriodicCheckInResponse`:
      token lookup, `Status != "Issued"` rejection, expiry check +
      auto-expire, then concatenate the 7 answers into `Response_Data`
      **in the field order specified in `m3-data-model.md`** (practice
      experience + professionalism first, alliance domains last — NOT the
      form's on-screen order) — pure string concatenation, no numeric
      parsing or conditional logic, per Constitution Principle VII. No new
      CRM field targets in the update map.

**Checkpoint**: Core pipeline (recurring auto-trigger -> issue -> collect ->
rejoin) functional and idempotent per-checkpoint, with zero CRM schema
changes.

## Phase 3: User Story 1 - Automated milestone trigger and de-identified delivery (Priority: P1)

**Scope for M3**: the recurring every-8th-session trigger case (spec.md
Milestones table, row 3; Acceptance Scenario 1).

- [ ] T007 [US1] Verify `Session_Count` crossing each listed checkpoint (8,
      16, 24, 32, 40) fires
      exactly once per checkpoint per patient in practice — extends M1/M2's
      T007/T008-equivalent real-world test to the recurring case
      specifically: confirm a SECOND M3 instance is correctly created once
      a test patient's session count later reaches 16, not blocked by the
      idempotency check the way a repeat at the SAME count is.
- [ ] T008 [US1] Confirm no duplicate Periodic Check-In invitation is ever
      sent for the same 8-session checkpoint (T003's recurring-checkpoint
      idempotency check) — satisfies spec.md FR-002 / Acceptance Scenario 2,
      re-verified for the recurring case.
- [ ] T009 [US1] Confirm feedback still goes out with zero contractor/
      clinician action required (spec.md Acceptance Scenario 3).

**Checkpoint**: US1 satisfied for M3 as a genuine automatic, recurring
trigger.

## Phase 4: User Story 2 - De-identified response collection and CRM rejoin (Priority: P1)

- [ ] T010 [US2] Confirm no patient identifier appears in the Periodic
      Check-In Form, the flow's parameters, or `Response_Data` (7 segments
      only, all scored/categorical) — Principle I.
- [ ] T011 [US2] Token match + rejoin implemented (T006) — response lands
      only on the matched `Patient`'s CRM record, never anywhere else.
- [ ] T012 [US2] Reuse rejection implemented via `Status != "Issued"` check
      (satisfies spec.md Acceptance Scenario 3 for User Story 2).
- [ ] T013 [US2] **Compliance gap, carried from M0/M1/M2, not resolved
      here**: purge of the raw Zoho Forms submission entry within "a short,
      defined retention window" (spec.md FR-006) is not implemented by this
      milestone either. M3 adds a fourth flow with the identical gap. Do
      not silently accept this gap a fourth time without at least
      confirming with Costin it's still being deferred deliberately — see
      `data-retention-purge.md`.

**Checkpoint**: Core de-identified rejoin works end-to-end for M3; retention
purge remains a known, tracked, cross-milestone gap.

## Phase 5: User Story 4 - Automated clinical safety flag to admin (Priority: P4, alliance rule, shared with M2)

**Scope for M3**: apply the same Clinical Safety Flag Rules alliance rule
M2 already built (total ≤20, or any domain ≤4) to M3's reused alliance
domains — by widening M2's existing formula columns, not forking a new pair
(`m3-research.md` Decision 5).

- [x] T014 [US4] Widen the existing "Clinical Safety Flag" formula column's
      `Milestone` gate to `'2 - Early Alliance Check' OR '3 - Periodic
      Consolidated'` per `m3-data-model.md`. Confirm it still fires
      correctly on every documented M2 case (re-verify M2's own test cases
      aren't broken by the widened gate) AND on the equivalent M3 cases
      (total ≤20 alone, a single domain ≤4 alone, both at once, and a
      reading that clears both thresholds does not fire) — satisfies
      FR-011 / Acceptance Scenario 2 for M3.
      **Update `m2-implementation-notes.md` §9.2 in the same session**,
      per CLAUDE.md's convention — this changes how M2's shipped Analytics
      build works, not just M3's.
- [ ] T015 [US4] Confirm no automated notification, email, or CRM action of
      any kind reaches a contractor when an M3 reading fires the flag —
      satisfies FR-012 / Acceptance Scenario 3. Confirm-absence task, not a
      build task — review T006's function, the flow, and the widened
      Analytics formula columns for any contractor-facing action.
- [ ] T016 [US4] Rename/widen the existing "M2 Flagged for Review" report to
      "M2 & M3 Flagged for Review" per `m3-data-model.md` (Milestone filter
      widened to Wildcard `"2 - Early Alliance Check"` OR `"3 - Periodic
      Consolidated"`), confirm a flagged M3 record is visible there with the
      patient identified only by Token (no `Patient`/identity column) —
      satisfies FR-013 for both milestones' data, milestone-scoped per
      `m2-research.md`'s FR-013 scope-boundary note (not the unified FR-007
      Admin Dashboard). **Update `m2-implementation-notes.md` §9.6 in the
      same session** — this renames/widens an M2-owned view.

**Checkpoint**: Alliance-rule clinical safety flagging works end-to-end for
M3, sharing (not duplicating) M2's infrastructure, admin-visible,
contractor-blind, with no CRM schema changes.

## Phase 6: Reporting (User Story 6 / FR-018 baseline bar)

*Full admin dashboard (User Story 3, FR-007–009) remains out of scope for
M3. M3's reporting must clear the FR-018 baseline bar from the start, per
the Rollout Workflow reminder — not just a raw submitted-responses table,
per `m1-implementation-notes.md` §13 / `m2-implementation-notes.md` §9's
established pattern.*

- [x] T017 Build the 3 new M3-only Analytics formula columns on "Milestone
      Instances" per `m3-data-model.md`: Practice Experience:
      Scheduling/Communication, Practice Experience: Billing, Therapist
      Professionalism (all bounded `substring_between`). Confirm the 4
      reused alliance-domain columns (Domain: Connection/Understanding/
      Shared Direction/Fit Of Approach) and Alliance Check-In Total parse
      M3 rows correctly with no edits, per the field-order design in
      `m3-data-model.md` — this is a confirmation step, not expected to
      require new work, but verify against a real test submission (Phase
      7) before assuming it, the same "verify, don't assume" discipline
      every prior milestone's Analytics build followed.
- [ ] T018 Build a minimal "M3 Submitted Responses" report (all 7 parsed
      columns + Alliance Check-In Total + the 2 shared flag columns, raw
      blob hidden, no `Patient`/identity column) — same pattern as
      M0/M1/M2's, scoped to `Milestone = "3 - Periodic Consolidated"`.
- [ ] T019 [Spec.md User Story 6 / FR-018] Build, at minimum: a status/
      volume view ("M3 Status Breakdown" — Issued/Submitted/Expired counts,
      "Save As" pattern off M2's equivalent) and at least one distribution/
      summary view of M3's *own new* scored data ("M3 Practice Experience &
      Professionalism Distribution" — the 3 new columns, `Dimension
      [Actual(D)] / Treat as Text` mode per `m2-implementation-notes.md`
      §9.5's gotcha). Bundle T018, T019's two new reports, and T016's
      shared "M2 & M3 Flagged for Review" into a new "M3 - Periodic
      Check-In Feedback" dashboard (built via "Create New Dashboards", not
      "Save As" off a mismatched-panel dashboard).

## Phase 7: Test data & validation

- [ ] T020 Get a test Patient record ID from Costin (per the standing
      Patients-module access restriction — do not look one up via CRM
      query or browser).
- [ ] T021 Using Zoho CRM MCP tools (not browser), exercise the recurring
      path specifically: simulate `Session_Count` reaching 8 for the test
      patient (confirm an M3 instance is created), then simulate it
      reaching 16 (confirm a SECOND M3 instance is created — the case that
      differentiates M3 from every prior milestone's idempotency check),
      submit test Periodic Check-In responses covering a flag-triggering
      case and a clean case, and confirm a repeat trigger at the same
      session count does not create a duplicate.
- [ ] T022 Confirm the Analytics formula columns (T017, plus the widened
      T014 flag columns) parse test submissions correctly for M3 rows —
      each of the 7 segments, the total, and both flag columns match a
      manual evaluation — and confirm M2's existing test/sample data (if
      any exists by then) still evaluates correctly under the widened
      Milestone gate.
- [ ] T023 Delete test `Milestone_Instances` records via Zoho CRM MCP tools
      when done, same as M0/M1/M2's cleanup pattern.

## Phase 8: Polish & documentation

- [ ] T024 Write `m3-implementation-notes.md` (as-built reference, mirroring
      `m2-implementation-notes.md`'s structure: component inventory,
      verbatim Deluge source for `checkPeriodicCheckInDue` and
      `submitPeriodicCheckInResponse`, the confirmed 7-segment blob format
      and its field order, the 3 new + 2 modified Analytics formulas, CRM
      field/picklist reference, test-data approach, access constraints)
      once M3 is actually built — per CLAUDE.md's convention, in the same
      session as the change.
- [ ] T025 Update `m2-implementation-notes.md` for every change T014/T016
      made to M2-owned Analytics objects (the widened Clinical Safety Flag
      gate; the renamed/widened Flagged for Review report) — **in the same
      session as those changes**, not deferred to T024's M3-only notes
      file. This is the same convention CLAUDE.md already states, applied
      to a case where a later milestone's build changes an earlier
      milestone's documented behavior.
- [ ] T026 Update `CLAUDE.md` if this build reveals a new cross-milestone
      convention worth capturing (the recurring-checkpoint idempotency
      pattern in `m3-research.md` Decision 2 seems the most likely
      candidate for a future recurring milestone to need — consider
      promoting it, and the "shared vs. per-milestone Analytics columns"
      decision framework in Decision 5, once this build confirms both work
      as designed).
- [ ] T027 Commit `m3-implementation-notes.md`, the `m2-implementation-notes.md`
      updates, and any plan/data-model corrections discovered during
      implementation, in the same session as the change, per the existing
      convention.

## Notes for whoever implements this

- T003's recurring-checkpoint math is the single highest-risk piece of this
  milestone — it's new logic, not a copy of an established pattern, and it
  determines both correctness (no missed or duplicated checkpoints) and
  Principle II compliance (idempotency must be re-derivable from CRM state
  alone, never cached). Test it explicitly against the multi-checkpoint
  case in T021, not just a single trigger.
- T006's blob field order is easy to get backwards during implementation
  because it deliberately does NOT match the form's on-screen section
  order — re-read `m3-data-model.md`'s exact segment order before writing
  the parameter mapping, and verify each parsed value with
  `document.activeElement.value` immediately after typing, the same
  discipline `m2-implementation-notes.md` §4's last bullet documents.
- T014/T016 modify M2-owned Analytics objects. Treat this the same as any
  other cross-milestone change: verify M2's own documented test cases still
  hold after the change, and update `m2-implementation-notes.md` in the
  same session (T025) — do not leave M2's notes describing formulas that no
  longer match what's actually in the workspace.
- T013's retention gap is real and open across the whole pipeline, not just
  M3 — see `data-retention-purge.md` before assuming it's someone else's
  problem to solve later.
- The wellbeing half of the Clinical Safety Flag Rules, which
  `m2-data-model.md`'s "Out of scope for M2" section flagged as an open
  question for M3 to resolve, is now moot: spec.md's fifth-pass revision
  permanently removed it (no milestone collects a wellbeing reading
  anymore). Nothing in this task list resolves it because there is nothing
  left to resolve — do not reintroduce it.
