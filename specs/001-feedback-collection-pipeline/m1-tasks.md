---
description: "Task list for M1 - Baseline Intake Survey (Session 1), generated prospectively before implementation"
---

# Tasks: M1 - Baseline Intake Survey (Milestone 1)

**Input**: `m1-plan.md`, `spec.md` (User Stories 1 & 2, plus the Clinical
Safety Flag Rules note under Constitution Check), `m1-research.md`,
`m1-data-model.md`

**Note**: PROSPECTIVE. Unlike M0, this task list was generated before any
implementation, per the constitution's Rollout Workflow requirement. All
tasks below start unchecked; check them off as they're actually done, not in
advance.

## Phase 1: Setup

- [x] T001 Build the public "Cape Clarity Wellbeing Check-In" Zoho Form: 5
      sliders (Personal wellbeing, Coping, Relationships and support, Hope
      and outlook, Sense of control — each 0-10, exact prompt text per
      `m1-research.md`), plus a hidden/single-line `Token` field. No name,
      email, or phone field, per Principle I. Build it generically
      (parameterized so it can be reused unmodified at M3/M4), not as an
      M1-only artifact.
      **Done (2026-09-06)**: built via Zoho Forms builder (Claude in Chrome,
      non-PII product, no Leads/Patients module touched). All 5 sliders
      range 0-10, mandatory, Min/Max labels shown, instructions carry the
      domain name for at-a-glance identification during testing. `Token`
      field is a hidden Single Line field; field alias `token` configured
      under Settings > Prefill > Field Alias - Prefill URL, so the survey
      link pattern is `<form permalink>?token=<value>`, matching M0's
      token-in-URL pattern. Form permalink:
      `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityWellbeingCheckIn/formperma/Qlvh_FeoU3Oof7fQz4WEeQBoFvQHYl2TLdNlQPON_Eg`.
      Note: this account is on the Zoho Forms Free plan, so the form-level
      HIPAA compliance add-on (Settings > Compliance & Audit > HIPAA) is
      unavailable, same constraint already logged in
      `data-retention-purge.md`. Not a blocker for this form specifically
      since it carries no PII (only a token and 0-10 scores), same design
      basis M0 already validated as compliant without that add-on.
- [x] T002 Confirm the `Milestone_Instances` module's existing `Milestone`
      picklist has a clean `"1 - Baseline Intake"` value (it does, per
      `m0-implementation-notes.md` §5) and that the `Patient` lookup field
      is in place and unused so far — no schema changes needed for M1 beyond
      what M0 already built.
      **Done (2026-09-06)**: confirmed via `getFields` (Zoho CRM MCP, schema
      read only, no record data touched). `Milestone` picklist includes
      `"1 - Baseline Intake"`; `Patient` (lookup), `Token` (text),
      `Response_Data` (textarea), `Expiry_Date_Time` (datetime), `Status`
      (picklist: Issued/Submitted/Expired/Superseded/Captured Live) all
      already exist from M0's build. No schema changes needed.

## Phase 2: Foundational

- [x] T003 Build "M1 - Session 1 Trigger" flow: watch
      `Patients1.Session_Count` for the transition to `1`.
- [x] T004 Implement the idempotency check inside that flow (query
      `Milestone_Instances` for an existing `Patient` + `Milestone = "1 -
      Baseline Intake"` record before creating a new one) — satisfies spec.md
      FR-002. This has no M0 equivalent; M0's manual trigger didn't need it.
- [x] T005 On no existing record found: create a new `Milestone_Instances`
      record (`Patient` = triggering patient, `Milestone = "1 - Baseline
      Intake"`, `Status = "Issued"`, fresh `Token`, `Expiry_Date_Time` —
      reuse M0's expiry window unless Costin specifies otherwise for M1) and
      send the Wellbeing Check-In link via Zoho Mail.
- [x] T006 Build "M1 - Wellbeing Check-In Write-back" flow: realtime
      Form-submission trigger on T001's form -> write-back custom function.
      **Done (2026-09-08)**: built via Zoho Flow (Browser pane automation).
      Trigger: Zoho Forms "Form entry submitted" (Realtime) on the Wellbeing
      Check-In form. Connected to `submitWellbeingCheckInResponse` (T007).
      Confirmed correctly wired (not just visually adjacent — see
      `m1-implementation-notes.md` §5 for the connector-verification gotcha
      hit along the way). Flow saved, still OFF/unpublished. See
      `m1-implementation-notes.md` §4A for the full step-by-step.
- [x] T007 Implement the write-back function: token lookup, `Status !=
      "Issued"` rejection (reuse protection, same pattern as M0's
      `submitFeedbackResponse`), expiry check + auto-expire, delimited
      `Response_Data` write-back in the format from `m1-data-model.md`
      (`---`-joined, 5 domains, total NOT stored in the blob), `Status ->
      "Submitted"`.
      **Done (2026-09-08)**: `submitWellbeingCheckInResponse` custom function,
      directly mirroring M0's `submitFeedbackResponse` pattern field-for-field.
      Parameters mapped to the Wellbeing Check-In form's internal field names
      (`Slider`, `Slider1`...`Slider4`, `SingleLine`), discovered via direct
      Forms-builder DOM inspection (no PII exposed). Verbatim source in
      `m1-implementation-notes.md` §4A.1.

**Checkpoint**: Core pipeline (auto-trigger -> issue -> collect -> rejoin)
functional and idempotent.

## Phase 3: User Story 1 - Automated milestone trigger and de-identified delivery (Priority: P1)

**Scope for M1**: the baseline-intake/session-1 trigger case specifically
(spec.md Milestones table, row 1; Acceptance Scenario 1).

- [ ] T008 [US1] Verify `Session_Count == 1` reliably fires exactly once per
      patient in practice (not just in the idempotency check's logic) —
      i.e., confirm the underlying payment-triggered workflow that increments
      `Session_Count` doesn't itself re-fire in a way that would matter here.
      This is Principle III's real test: unlike M0, there's no manual
      fallback to lean on if the automatic path misbehaves.
- [ ] T009 [US1] Confirm no duplicate Wellbeing Check-In invitation is ever
      sent for the same patient's Milestone 1 occurrence (T004's idempotency
      check) — satisfies spec.md FR-002 / Acceptance Scenario 2.
- [ ] T010 [US1] Confirm feedback still goes out with zero contractor/
      clinician action required (spec.md Acceptance Scenario 3) — nothing
      about M1's design should require a clinician to remember or do
      anything, unlike M0's documented manual step.

**Checkpoint**: US1 satisfied for M1 as a genuine automatic trigger, not a
documented fallback.

## Phase 4: User Story 2 - De-identified response collection and CRM rejoin (Priority: P1)

- [ ] T011 [US2] Confirm no patient identifier appears in the Wellbeing
      Check-In Form, the flow's parameters, or `Response_Data` (Principle I)
      — same field-by-field review approach as `m0-implementation-notes.md`
      §5.
- [ ] T012 [US2] Token match + rejoin implemented (T007) — response lands
      only on the matched `Patient`'s CRM record, never anywhere else.
- [ ] T013 [US2] Reuse rejection implemented via `Status != "Issued"` check
      (satisfies spec.md Acceptance Scenario 3 for User Story 2).
- [ ] T014 [US2] **Compliance gap, carried from M0, not resolved here**:
      purge of the raw Zoho Forms submission entry within "a short, defined
      retention window" (spec.md FR-006) is not implemented by this
      milestone either. M1 adds a second flow with the identical gap. Not
      M1's job to fix alone — see `data-retention-purge.md` for the
      cross-milestone investigation and options. Do not silently accept this
      gap a second time without at least confirming with Costin that it's
      still being deferred deliberately.

**Checkpoint**: Core de-identified rejoin works end-to-end for M1; retention
purge remains a known, tracked, cross-milestone gap.

## Phase 5: Reporting (minimal — confirms the pipeline works)

*Full admin dashboard (User Story 3, FR-007–009) and clinical safety flagging
(User Story 4, FR-010–013) are out of scope for M1 — see `m1-data-model.md`
"Out of scope for M1." M1's reporting need is limited to proving the pipeline
actually captured and parsed data correctly.*

- [ ] T015 Build Analytics formula columns on "Milestone Instances" for the
      5 Wellbeing Check-In domains (`substring_between`/`SUBSTR` pattern from
      `m0-implementation-notes.md` §6, applied to the new blob format) plus
      a `SUM`-based Wellbeing Check-In Total column.
- [ ] T016 Build a minimal "M1 Submitted Responses" report (parsed domain
      columns + total, raw blob hidden) — same pattern as M0's, scoped to
      `Milestone = "1 - Baseline Intake"` — just enough to confirm the
      pipeline is working, not a full dashboard.

## Phase 6: Test data & validation

- [ ] T017 Get a test Patient record ID from Costin (per the standing
      Patients-module access restriction — do not look one up via CRM query
      or browser).
- [ ] T018 Using Zoho CRM MCP tools (not browser), exercise the full path:
      set/simulate `Session_Count` reaching 1 for the test patient (or, if
      that's not directly settable, create a `Milestone_Instances` record by
      hand to test the write-back half independently), submit a test
      Wellbeing Check-In response, and confirm the idempotency check
      correctly blocks a second attempt.
- [ ] T019 Confirm the Analytics formula columns (T015) parse the test
      submission correctly and the total matches a manual sum.
- [ ] T020 Delete test `Milestone_Instances` records via Zoho CRM MCP tools
      when done, same as M0's cleanup pattern.

## Phase 7: Polish & documentation

- [ ] T021 Write `m1-implementation-notes.md` (as-built reference, mirroring
      `m0-implementation-notes.md`'s structure: component inventory, verbatim
      Deluge source for the new write-back function, the confirmed blob
      format, Analytics formulas, CRM field reference, test-data approach,
      access constraints) once M1 is actually built.
- [ ] T022 Update `CLAUDE.md` if anything about the cross-milestone
      conventions (blob format, idempotency pattern, dead-config notes)
      needs amending based on what M1's build actually reveals.
- [ ] T023 Commit `m1-implementation-notes.md` and any plan/data-model
      corrections discovered during implementation, in the same session as
      the change, per the existing convention.

## Notes for whoever implements this

- This is the first milestone where the automatic trigger is real (Phase 3,
  T008) — treat T008 as the highest-risk task, not a formality, since there's
  no manual fallback to catch a misfire the way there is for M0.
- The Wellbeing Check-In Form and its blob format (T001, Response_Data
  format) are shared infrastructure for M3 and M4 — build them generically,
  not M1-specifically, and update this note (and `CLAUDE.md`) if that
  reuse plan changes.
- T014's retention gap is real and open across the whole pipeline, not just
  M1 — see `data-retention-purge.md` before assuming it's someone else's
  problem to solve later.
