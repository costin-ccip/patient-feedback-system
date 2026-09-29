---
description: "Task list for Contractor Performance Analytics, generated prospectively before implementation"
---

# Tasks: Contractor Performance Analytics

**Input**: `plan.md`, `spec.md` (User Stories 1-6), `research.md`, `data-model.md`,
`quickstart.md`

**Note**: PROSPECTIVE. Same discipline as M0-M5's own task lists: all tasks below
start unchecked; check them off as they're actually done, not in advance. Per
standing instruction, nothing here includes a live/end-to-end test step Costin
hasn't run himself, and this feature touches no CRM Leads/Patients module in the
browser (Milestone_Instances is non-PII; MCP tool calls against it are fine per the
standing access constraint).

## Phase 1: Setup

- [ ] T001 Confirm the "Milestone Instances" Analytics table's current column list
      matches what `data-model.md` assumes is already there (the four Domain
      columns, Alliance Check-In Total, Therapist Professionalism, both Practice
      Experience columns, Looking Ahead: Likelihood To Recommend, Reason For
      Leaving, `Clinician`, `Milestone`) — a read-only confirmation, no build.

## Phase 2: Foundational

**⚠️ Gates User Story 2 only** (T007-T010 below) — User Stories 1, 3, 4, 5, and 6 do
not depend on this phase and can proceed as soon as Phase 1 is done.

- [x] T002 `research.md` Decision 1 is resolved (2026-09-29, Costin, confirmed live
      via CRM schema introspection): Story 2's per-patient grouping key reuses the
      existing `Patients1.Name` field (the practice's existing `P`+number
      de-identified code, surfaced by the existing `Milestone_Instances.Patient`
      lookup) — no new CRM field, no hash. Nothing further needed here before
      Story 2-specific work; see T007.
- [ ] T003 [P] Confirm the current live `Clinician` picklist values (via Zoho CRM
      MCP `getFields` on `Milestone_Instances` — non-PII module, allowed under the
      standing access constraint) to use as the real contractor set in later
      validation, and to sanity-check `data-model.md`'s assumption that `Clinician`
      is already populated on existing M0/M2/M3/M4/M5 rows.

**Checkpoint**: Decision 1 resolved (T002 done); real `Clinician` values confirmed.
User Story 1 and User Story 2 work can both start once T003 is done.

## Phase 3: User Story 1 - Per-contractor Alliance score averages, combined and per-milestone (Priority: P1)

**Goal**: A practice owner can see each contractor's Alliance domain/Total averages,
both pooled across M2/M3/M4 and broken out per milestone.

**Independent Test**: Seed M2/M3/M4 responses across two contractors and confirm
both the combined and per-milestone figures are correct and don't mix contractors
(spec.md Acceptance Scenarios 1-4).

- [ ] T004 [P] [US1] Build the "Contractor Alliance Summary" aggregate report:
      `Milestone Instances` filtered to `Milestone IN ('2 - Early Alliance Check',
      '3 - Periodic Consolidated', '4 - Discharge')`, grouped by `Clinician`,
      averaging the four Domain columns and Alliance Check-In Total. No new base
      formula columns needed (`data-model.md`).
- [ ] T005 [P] [US1] Build the "Contractor Alliance by Milestone" aggregate report:
      same filter/averages as T004, additionally grouped by `Milestone`.
- [ ] T006 [US1] Confirm a contractor with zero qualifying responses (combined, or
      at a specific milestone) shows an explicit "no data" state, not a zero or
      blank average — FR-006.

**Checkpoint**: User Story 1 fully functional and independently testable —
combined and per-milestone Alliance averages, correctly isolated per contractor.

## Phase 4: User Story 2 - Per-contractor Practice Experience and Therapist Professionalism averages (Priority: P2)

**Depends on T002 (done) and T003.** Decision 1 is resolved, so this story can
proceed directly to T007.

**Goal**: A practice owner can see each contractor's Professionalism, Scheduling/
Communication, and Billing averages, weighted one-vote-per-patient.

**Independent Test**: Seed M3 responses including one patient with 2+ responses for
the same contractor, and confirm that patient's figures are averaged together
before being combined with the contractor's other patients (spec.md Acceptance
Scenario 2; `quickstart.md` §B's concrete numbers).

- [ ] T007 [US2] Add the `Patients1.Name` grouping key to the Analytics sync per
      `research.md` Decision 1 / `data-model.md` (via the existing
      `Milestone_Instances.Patient` lookup — no new CRM field, no hash). Confirm it
      does not appear in any report's column list or any dashboard panel anywhere
      in the workspace before proceeding (it's safe enough to appear, per
      research.md, but no story needs it displayed).
- [ ] T008 [US2] Build the per-patient sub-aggregate (likely a query table, per
      `data-model.md`): average Therapist Professionalism / Scheduling-
      Communication / Billing per (`Patients1.Name` grouping key, `Milestone = '3 -
      Periodic Consolidated'`).
- [ ] T009 [US2] Build the "Contractor Professionalism & Practice Experience"
      report: average of T008's per-patient figures, grouped by `Clinician`.
- [ ] T010 [US2] Verify against `quickstart.md` §B's seeded case: a contractor with
      one 2-response patient and one 1-response patient produces a figure where
      the 2-response patient's average counts once, not twice, relative to the
      1-response patient.

**Checkpoint**: User Story 2 fully functional — patient-weighted, not
response-weighted, and isolated per contractor.

## Phase 5: User Story 3 - Per-contractor Likelihood-to-Recommend averages (Priority: P3)

**Goal**: A practice owner can see each contractor's average Likelihood-to-Recommend
score from their M4 patients.

**Independent Test**: Seed M4 responses across two contractors and confirm each
contractor's average reflects only their own patients (spec.md Acceptance
Scenarios 1-2).

- [ ] T011 [US3] Build the "Contractor Likelihood to Recommend" report:
      `Milestone Instances` filtered to `Milestone = '4 - Discharge'`, grouped by
      `Clinician`, averaging Looking Ahead: Likelihood To Recommend. No
      patient-weighting concern (M4 is one-time-per-patient).

**Checkpoint**: User Story 3 fully functional, independently testable.

## Phase 6: User Story 4 - Per-contractor discontinuation reason breakdown (Priority: P4)

**Goal**: A practice owner can see, per contractor, how their discontinuing
patients' reasons for leaving break down by category.

**Independent Test**: Seed M5 responses with a mix of reason categories across two
contractors and confirm each contractor's breakdown reflects only their own
patients (spec.md Acceptance Scenarios 1-2).

- [ ] T012 [US4] Build the "Contractor Reason for Leaving Breakdown" report:
      `Milestone Instances` filtered to `Milestone = '5B - Discontinuation, Email
      Fallback'`, grouped by `Clinician` and `Reason For Leaving`, count and/or
      share per category. `Dimension [Actual(D)] / Treat as Text` mode if the
      column doesn't already render as plain `Actual` (`m2-implementation-notes.md`
      §9.5's gotcha).

**Checkpoint**: User Story 4 fully functional, independently testable.

## Phase 7: User Story 5 - Drill down into milestone-level detail (Priority: P5)

**Depends on** at least one of User Stories 1-4 existing to drill down from.

**Goal**: From any contractor summary panel, reach the corresponding existing
milestone-level detail report, pre-filtered to that contractor.

**Independent Test**: From a contractor's Alliance summary, use the drill-down
action and confirm it reaches that contractor's individual Alliance-track
responses with no patient-identifying column (spec.md Acceptance Scenarios 1-2).

- [ ] T013 [US5] Confirm the exact drill-down mechanism available in the live
      Analytics workspace (native drill-down configuration vs. a filtered report
      link — `research.md` Decision 4) before wiring anything.
- [ ] T014 [US5] Wire drill-down from each of the four Contractor Performance
      panels (T004/T005, T009, T011, T012) to its corresponding existing M2/M3/
      M4/M5 milestone-level report, pre-filtered to the selected contractor.
      Confirm no drill-down destination adds a `Patient`/identity column that
      wasn't already on that existing report.

**Checkpoint**: User Story 5 fully functional — every contractor panel reaches the
right existing detail, still de-identified.

## Phase 8: User Story 6 - New contractors appear without rebuilding the dashboard (Priority: P5)

**Best verified once at least one of User Stories 1-4 exists** — this is a
cross-cutting property of how T004-T012 were built, not a new data view of its own.

**Goal**: A new `Clinician` picklist value with qualifying data appears in every
relevant view automatically, with zero report/panel/dashboard edits.

**Independent Test**: Add a new value to the `Clinician` picklist, seed one
qualifying response for it, and confirm it appears in the relevant view(s)
unedited (spec.md Acceptance Scenarios 1-2).

- [ ] T015 [US6] Audit every report/formula built in T004-T014: confirm none
      references a specific contractor's name in a filter, formula, or report
      title — a contractor's name should only ever appear as data.
- [ ] T016 [US6] Live-verify: add a new `Clinician` picklist value (reusing the
      practice's real third contractor, `Shana Lacastro`, per `quickstart.md` §B,
      is fine), seed one qualifying response, and confirm it surfaces in the
      relevant view(s) with zero report/dashboard edits.

**Checkpoint**: User Story 6 verified — the dashboard scales with the practice's
own contractor roster, no per-contractor rebuild required.

## Phase 9: Dashboard assembly

- [ ] T017 Build the "Contractor Performance" dashboard (via "Create New
      Dashboards," not "Save As," per every prior milestone's own convention),
      bundling T004, T005, T009, T011, and T012's reports, with T014's drill-down
      links wired on each panel.

## Phase 10: Test data & validation

- [ ] T018 Seed the sample `Milestone_Instances` test records from `quickstart.md`
      §B via Zoho CRM MCP tools (`createRecords`/`getRecords`), not the browser —
      the module carries no PII, so tool-based CRUD is allowed under the standing
      access constraint.
- [ ] T019 Work through `quickstart.md` §A's structural verification checklist
      against everything built in Phases 3-9.
- [ ] T020 Confirm actual report output against `quickstart.md` §B's expected
      results table, especially the Story 2 patient-weighting check (T010) and the
      Story 6 new-contractor check (T016).
- [ ] T021 Delete all seeded test `Milestone_Instances` records via Zoho CRM MCP
      tools when done, same cleanup discipline as every prior milestone.

## Phase 11: Polish & documentation

- [ ] T022 Write `specs/005-contractor-performance-analytics/implementation-notes.md`
      (as-built reference, mirroring the shape of `m0`-`m5-implementation-notes.md`:
      component inventory, exact formula/aggregate definitions, confirmation that
      Decision 1's `Patients1.Name` grouping key was built as planned, dashboard/
      report names, test-data approach) once this feature is actually built — per
      CLAUDE.md's convention, in the same session as the change.
- [ ] T023 Update `CLAUDE.md` if this build establishes a new cross-milestone
      convention worth capturing — likely candidates: this being the first
      Analytics-only feature (no Deluge/Flow/Forms touched) in the pipeline, and,
      if built, the internal-grouping-key pattern as a reusable precedent for any
      future feature that needs per-patient cross-record aggregation without
      syncing patient identity.
- [ ] T024 Commit `implementation-notes.md`, the `CLAUDE.md` update (if any), and
      any plan/research/data-model corrections discovered during implementation,
      in the same session as the change, following CLAUDE.md's "Pushing to GitHub"
      process appropriate to whichever environment the build session is running in.

## Notes for whoever implements this

- T002/Decision 1 is resolved (see Phase 2) — Story 2 reuses the existing
  `Patients1.Name` de-identified code via the existing lookup, no new field. Every
  other story (1, 3, 4, 5, 6) proceeds directly from data already in Analytics
  with no sync or schema question at all.
- This is the first feature in the pipeline that touches zero Deluge, zero Zoho
  Flow, and zero Zoho Forms — the entire build surface is the Zoho Analytics
  workspace. Don't reach for a write-back function or a trigger flow here; if a
  task in this list starts to look like it needs one, that's a signal the task has
  drifted outside this feature's actual scope (see `research.md`'s "Out of scope"
  section) and should be raised with Costin rather than built.
- T014's drill-down destinations must always be one of the *existing* M2/M3/M4/M5
  milestone-level reports — never a new patient-level view. If a drill-down
  seems to need patient-level detail beyond what those reports already show,
  that's out of scope for this feature (see spec.md's Assumptions on the
  relationship to the still-unbuilt unified Admin Dashboard, FR-007-009).
- No flow gets switched on or off by this feature (there is no flow), but the
  same "don't run a live/end-to-end test without Costin" standing instruction
  still applies to T016's picklist-value addition and any real-data validation —
  keep it structural and sample-data-based per Phase 10, and let Costin do his
  own pass per `quickstart.md` §C.
