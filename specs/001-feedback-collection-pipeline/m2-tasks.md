---
description: "Task list for M2 - Early Alliance Check (Session 3), generated prospectively before implementation"
---

# Tasks: M2 - Early Alliance Check (Milestone 2)

**Input**: `m2-plan.md`, `spec.md` (User Stories 1, 2 & 4, plus the Clinical
Safety Flag Rules and User Story 6 / FR-018), `m2-research.md`,
`m2-data-model.md`

**Note**: PROSPECTIVE. Same discipline as M1 (`m1-tasks.md`): all tasks below
start unchecked; check them off as they're actually done, not in advance.
**Revised 2026-09-10** (Costin): `Milestone_Instances` is at its CRM
custom-field cap — the original T002 ("add 2 new CRM fields for the flag")
is removed. Task numbering below reflects this removal.
**Revised again 2026-09-10** (second correction, Liana): the flag is not
folded into the `Response_Data` blob either — it's computed entirely as two
new Zoho Analytics formula columns from data already parsed out of the blob's
4 domain segments, never stored in `Milestone_Instances` at all. See
`m2-data-model.md` "Revision 2" section. T006 reverts to M0/M1's plain
string-concatenation shape; the flag-detection work moves to T017 (Analytics)
instead.

## Phase 1: Setup

- [ ] T001 Build the public "Cape Clarity Alliance Check-In" Zoho Form: 4
      sliders (Connection, Understanding, Shared direction, Fit of approach —
      each 0-10, exact prompt text per `m2-research.md`), plus a hidden/
      single-line `Token` field with a `token` field alias (same
      prefill-URL pattern as the Wellbeing Check-In form, per
      `m1-tasks.md` T001). No name, email, or phone field, per Principle I.
      Build it generically (reusable unmodified at M3), not as an M2-only
      artifact.

## Phase 2: Foundational

- [ ] T002 Build "M2 - Session 3 Trigger" flow: watch
      `Patients1.Session_Count` for the transition to `3`, mirroring M1's
      "M1 - Session 1 Trigger" structure exactly (trigger → idempotency
      check → If-else → "Call a subflow"). Verify all links are genuinely
      connected before attempting to switch it on — `m1-implementation-notes.md`
      §5/§12 both document real cases where a visually-adjacent node/link
      was not actually wired; use the DOM connector-element check
      (`[class*="connector" i]`, expect 2 elements per connected pair) from
      that note rather than trusting the canvas layout alone.
- [ ] T003 Implement the idempotency check inside that flow
      (`checkAllianceCheckExists`, mirroring `checkBaselineIntakeExists`
      field-for-field per `m1-implementation-notes.md` §3.1 — query
      `Milestone_Instances` for an existing `Patient` + `Milestone = "2 -
      Early Alliance Check"` record before creating a new one) — satisfies
      spec.md FR-002.
- [ ] T004 On no existing record found: create a new `Milestone_Instances`
      record (`Patient` = triggering patient, `Milestone = "2 - Early
      Alliance Check"`, `Status = "Issued"`, fresh `Token`,
      `Expiry_Date_Time` — reuse M0/M1's 7-day TTL unless Costin specifies
      otherwise) and send the Alliance Check-In link via Zoho Mail, through
      the unchanged shared "Subflow - Issue Feedback Token" — no schema/field
      changes to that subflow are needed for M2.
- [ ] T005 Build "M2 - Alliance Check-In Write-back" flow: realtime
      Form-submission trigger on T001's form → write-back custom function.
      Mirror M1's connector-verification discipline (`m1-implementation-notes.md`
      §5) — confirm the trigger is genuinely wired to the function node, not
      just visually adjacent, before considering this done.
- [ ] T006 Implement the write-back function `submitAllianceCheckInResponse`:
      token lookup, `Status != "Issued"` rejection, expiry check +
      auto-expire, then concatenate the four domain answers into
      `Response_Data` — **same shape as M0/M1's `submitFeedbackResponse` /
      `submitWellbeingCheckInResponse`** (`m1-implementation-notes.md`
      §4A.1), pure string concatenation, no numeric parsing or conditional
      logic. **Revised 2026-09-10 (second correction)**: the flag evaluation
      originally planned for this function has moved entirely to Analytics
      (T017) — T006 carries no flag-related logic at all. **No new CRM field
      targets** in the update map — `Milestone_Instances` takes no schema
      changes for this milestone.

**Checkpoint**: Core pipeline (auto-trigger -> issue -> collect -> rejoin)
functional and idempotent, with zero CRM schema changes. Flag evaluation
happens downstream in Analytics (Phase 6), not in this pipeline.

## Phase 3: User Story 1 - Automated milestone trigger and de-identified delivery (Priority: P1)

**Scope for M2**: the Session-3 trigger case specifically (spec.md Milestones
table, row 2; Acceptance Scenario 1).

- [ ] T007 [US1] Verify `Session_Count == 3` reliably fires exactly once per
      patient in practice — same real-world test M1's T008 called for
      (`m1-tasks.md`), now against the count-3 transition instead of count-1.
- [ ] T008 [US1] Confirm no duplicate Alliance Check-In invitation is ever
      sent for the same patient's Milestone 2 occurrence (T003's idempotency
      check) — satisfies spec.md FR-002 / Acceptance Scenario 2.
- [ ] T009 [US1] Confirm feedback still goes out with zero contractor/
      clinician action required (spec.md Acceptance Scenario 3).

**Checkpoint**: US1 satisfied for M2 as a genuine automatic trigger.

## Phase 4: User Story 2 - De-identified response collection and CRM rejoin (Priority: P1)

- [ ] T010 [US2] Confirm no patient identifier appears in the Alliance
      Check-In Form, the flow's parameters, or `Response_Data` (4 domain
      segments only, same shape as M1's blob — no flag segments, per
      Revision 2) — Principle I — same field-by-field review approach as
      `m0-implementation-notes.md` §5 / `m1-tasks.md` T011.
- [ ] T011 [US2] Token match + rejoin implemented (T006) — response lands
      only on the matched `Patient`'s CRM record, never anywhere else.
- [ ] T012 [US2] Reuse rejection implemented via `Status != "Issued"` check
      (satisfies spec.md Acceptance Scenario 3 for User Story 2).
- [ ] T013 [US2] **Compliance gap, carried from M0/M1, not resolved here**:
      purge of the raw Zoho Forms submission entry within "a short, defined
      retention window" (spec.md FR-006) is not implemented by this
      milestone either. M2 adds a third flow with the identical gap. Do not
      silently accept this gap a third time without at least confirming with
      Costin that it's still being deferred deliberately — see
      `data-retention-purge.md`.

**Checkpoint**: Core de-identified rejoin works end-to-end for M2; retention
purge remains a known, tracked, cross-milestone gap.

## Phase 5: User Story 4 - Automated clinical safety flag to admin (Priority: P4, alliance half only)

**Scope for M2**: the alliance-rule half of the Clinical Safety Flag Rules
only (spec.md — "flag for admin review when either: the total alliance score
is 20 or below (out of 40); or any single domain scores 4 or below"). The
wellbeing half (FR-010) is out of scope until M3, per `m2-research.md`.

- [ ] T014 [US4] Confirm the flag evaluation logic (T017's Analytics formula
      columns, revised 2026-09-10 — no longer T006) correctly fires on every
      documented case: total ≤20 alone, a single domain ≤4 alone, both at
      once (confirm the `Flag Rule Triggered` formula records every condition
      that fired, not just the first match), and confirm it does **not** fire
      for a reading that clears both thresholds — satisfies FR-011 /
      Acceptance Scenario 2.
- [ ] T015 [US4] Confirm no automated notification, email, or CRM action of
      any kind reaches a contractor when a flag fires — satisfies FR-012 /
      Acceptance Scenario 3. This is a "confirm absence," not a build task:
      review T006's function, the flow, and T017's Analytics formula columns
      for any contractor-facing action and verify there is none, per
      `m2-research.md`'s reconciliation of the source Confluence page's stale
      "routes to the treating clinician" language.
- [ ] T016 [US4] Build the "M2 Flagged for Review" Analytics report (per
      `m2-data-model.md`, filtered on the new `Clinical Safety Flag` formula
      column `= "true"`) and confirm a flagged record is visible there with
      the patient identified only by Token (no `Patient`/identity column) —
      satisfies FR-013 for M2's own data, milestone-scoped per
      `m2-research.md`'s FR-013 scope-boundary note (not the unified FR-007
      Admin Dashboard).

**Checkpoint**: Alliance-half clinical safety flagging works end-to-end for
M2, admin-visible, contractor-blind, with no CRM schema changes.

## Phase 6: Reporting (User Story 6 / FR-018 baseline bar)

*Full admin dashboard (User Story 3, FR-007–009) remains out of scope for M2
— see `m2-data-model.md` "Out of scope for M2". M2's reporting must clear the
FR-018 baseline bar from the start, per the Rollout Workflow reminder — not
just a raw submitted-responses table, per `m1-implementation-notes.md` §13's
established pattern for what "clearing the bar" looks like in this project.*

- [ ] T017 Build Analytics formula columns on "Milestone Instances" for the
      4 Alliance Check-In domains — 3 bounded (`substring_between`: Connection,
      Understanding, Shared Direction) and 1 unbounded (Fit Of Approach, the
      true last field in the blob — guarded `SUBSTR`/`INSTR`/`LENGTH` with the
      zero-guard from the start, per `m1-implementation-notes.md` §9.3) — an
      "Alliance Check-In Total" column (`SUM`/`to_integer()` pattern, per
      `m1-implementation-notes.md` §9.2), and, **revised 2026-09-10 (second
      correction)**, the 2 flag-detection columns computed directly from the
      Total/Domain columns rather than parsed from the blob: "Clinical Safety
      Flag" (`IF(OR(Total<=20, [Domain: Connection]<=4, ...), "true",
      "false")`) and "Flag Rule Triggered" (nested `IF`/`CONCATENATE`
      building a description of every condition that fired, per
      `m2-data-model.md`'s "Clinical Safety Flag evaluation" section). No
      substring parsing needed for either flag column — no flag data exists
      in the blob for them to parse.
- [ ] T018 Build a minimal "M2 Submitted Responses" report (parsed domain
      columns + total + the 2 flag columns, raw blob hidden) — same pattern
      as M0/M1's, scoped to `Milestone = "2 - Early Alliance Check"`.
- [ ] T019 [Spec.md User Story 6 / FR-018] Build, at minimum: a status/volume
      view ("M2 Status Breakdown" — Issued/Submitted/Expired counts, same
      "Save As" pattern off M0/M1's equivalent per `m1-implementation-notes.md`
      §13.1) and one distribution/summary view of the Alliance Check-In
      domains or Total ("M2 Alliance Check-In Total Distribution", same
      "Save As" + Milestone-filter-swap pattern per §13.2). Bundle T018,
      T019's two new reports, and T016's "M2 Flagged for Review" into a new
      "M2 - Early Alliance Check Feedback" dashboard (same shape as M1's
      dashboard, §13.3 — built via "Create New Dashboards", not "Save As" off
      an existing dashboard with mismatched panels).

## Phase 7: Test data & validation

- [ ] T020 Get a test Patient record ID from Costin (per the standing
      Patients-module access restriction — do not look one up via CRM query
      or browser).
- [ ] T021 Using Zoho CRM MCP tools (not browser), exercise the full path:
      set/simulate `Session_Count` reaching 3 for the test patient (or create
      a `Milestone_Instances` record by hand to test the write-back half
      independently), submit test Alliance Check-In responses covering both
      a flag-triggering case and a clean case, and confirm the idempotency
      check correctly blocks a second trigger attempt.
- [ ] T022 Confirm the Analytics formula columns (T017) parse test
      submissions correctly and the total matches a manual sum; confirm the
      flag columns match manual evaluation of the Clinical Safety Flag Rules
      for each test case.
- [ ] T023 Delete test `Milestone_Instances` records via Zoho CRM MCP tools
      when done, same as M0/M1's cleanup pattern.

## Phase 8: Polish & documentation

- [ ] T024 Write `m2-implementation-notes.md` (as-built reference, mirroring
      `m1-implementation-notes.md`'s structure: component inventory, verbatim
      Deluge source for the new write-back function (plain string
      concatenation, no flag logic — that's Analytics-only, per Revision 2),
      the confirmed blob format, Analytics formulas including the 2 flag
      columns, CRM field reference confirming no schema changes, test-data
      approach, access constraints) once M2 is actually built — per
      CLAUDE.md's convention, in the same session as the change, not a
      follow-up task.
- [ ] T025 Update `CLAUDE.md` if anything about the cross-milestone
      conventions (blob format, idempotency pattern, the now-established
      "flag data goes in the blob, not new fields" pattern for M3/M4) needs
      amending based on what M2's build actually reveals.
- [ ] T026 Commit `m2-implementation-notes.md` and any plan/data-model
      corrections discovered during implementation, in the same session as
      the change, per the existing convention.

## Notes for whoever implements this

- T006 is **not** an elevated-risk task, revised 2026-09-10 (second
  correction) — it's the same pure string-concatenation shape as M0/M1's
  write-back functions, no numeric parsing or conditional logic. The
  first-draft design had put real arithmetic and flag evaluation inline in
  this function; that moved entirely to Analytics (T017) after Liana pointed
  out it duplicated logic Analytics already had to do for the Total column,
  for no benefit. T017's `Flag Rule Triggered` formula (nested `IF`/
  `CONCATENATE` across 5 conditions) is now the more novel piece of this
  milestone — test it against each documented flag case (T014) rather than
  assuming a formula-column implementation is inherently lower-risk than a
  Deluge one just because it's declarative.
- T015 is a verification task, not a build task — the correct outcome is
  confirming nothing was built that shouldn't exist, not writing code.
- `Milestone_Instances` has **no CRM field budget left** — this was
  discovered during M2's planning (Costin, 2026-09-10) and applies to every
  future milestone too, not just M2. Separately, M2's own build then
  established a second, narrower precedent worth carrying forward too:
  default to computing derived values (like the flag) in Analytics formula
  columns rather than in Deluge, even when a field-budget workaround like the
  blob would technically make Deluge-side storage possible — Analytics
  formula columns recalculate retroactively on a rule change, and Deluge-time
  computation doesn't. M3/M4's wellbeing trend-flag data should start from
  that position, adjusting only if trend evaluation turns out to be
  genuinely awkward to express as a single Analytics formula column (see
  `m2-data-model.md` "Out of scope for M2"). Worth calling out explicitly in
  T025's CLAUDE.md update.
- The Alliance Check-In Form and its blob/flag pattern are shared
  infrastructure for M3 (per `m2-research.md`) — build them generically and
  update this note (and `CLAUDE.md`) if that reuse plan changes.
- T013's retention gap is real and open across the whole pipeline, not just
  M2 — see `data-retention-purge.md` before assuming it's someone else's
  problem to solve later.
