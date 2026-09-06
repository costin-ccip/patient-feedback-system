---
description: "Task list for M0 - Free Consult / No-Conversion Feedback (backfilled retroactively from the completed build)"
---

# Tasks: M0 - Free Consult / No-Conversion Feedback

**Input**: `m0-plan.md`, `spec.md` (User Stories 1 & 2), `m0-implementation-notes.md`

**Note**: RETROACTIVE. M0 was already built when this task list was written
(2026-09-06). Tasks are marked `[x]` where the implementation notes confirm the
work is done, and left `[ ]` where it genuinely is not — this file is an honest
record of what happened, not a checklist being completed to look compliant.
Future milestones (M1+) should have this file generated prospectively by
`/speckit-tasks`, before implementation, per the constitution.

## Phase 1: Setup

- [x] T001 Create Zoho Flow folder "Customer Feedback System" to hold all
      feedback-system flows.
- [x] T002 Create `Milestone_Instances` custom module in Zoho CRM with fields:
      `Name`, `Lead_Reference` (text), `Patient` (lookup, unused by M0),
      `Milestone` (picklist), `Status` (picklist), `Clinician` (picklist),
      `Token`, `Expiry_Date_Time`, `Submitted_Date_Time`, `Response_Data`.
- [x] T003 Create Zoho Analytics workspace synced from CRM
      (`3251423000000083002`), including the "Milestone Instances" table.

## Phase 2: Foundational

- [x] T004 Build "Subflow - Issue Feedback Token" (manual trigger, invoked by
      clinician) to create a `Milestone_Instances` record with a fresh `Token`,
      `Milestone = "0 - No Conversion"`, `Status = "Issued"`, and an
      `Expiry_Date_Time`.
- [x] T005 Build the public M0 Zoho Form (Feeling Heard rating, Biggest Factor
      radio, Additional Comments, Reach-Back-Out radio, Anything Else, plus a
      hidden/single-line `Token` field) with no name/email/phone fields
      (Constitution Principle I).
- [x] T006 Build "M0 - Feedback Survey Write-back" flow: realtime Form-submission
      trigger -> `submitFeedbackResponse` custom function.
- [x] T007 Implement `submitFeedbackResponse`: token lookup, `Status != "Issued"`
      rejection (reuse protection), expiry check + auto-expire, delimited
      `Response_Data` write-back, `Status -> "Submitted"`.

**Checkpoint**: Core pipeline (issue -> collect -> rejoin) functional.

## Phase 3: User Story 1 - Automated milestone trigger and de-identified delivery (Priority: P1)

**Scope for M0**: the free-consult/no-conversion trigger case specifically
(spec.md Acceptance Scenario 2).

- [x] T008 [US1] Manual clinician-initiated token issuance in place of an
      automatic CRM-field trigger. **Note**: re-checked against the
      authoritative constitution (v1.2.1) — Principle III ("Automated
      Milestone Triggers") does not explicitly carve out a manual-fallback
      allowance the way earlier internal notes assumed; see `m0-plan.md`
      Constitution Check for the open question this raises, flagged for
      Costin rather than resolved here.
- [x] T009 [US1] Token single-use enforcement via `Status` gate (T007) satisfies
      spec.md Acceptance Scenario 3 (no duplicate send/accept for the same
      milestone occurrence).
- [ ] T010 [US1] **Open**: no automatic detection path exists or is planned for
      M0 (by design — see T008); if this ever needs to become automatic,
      requires a proxy CRM signal (e.g., a specific Lead stage) that doesn't
      exist yet.

**Checkpoint**: US1 satisfied for M0's manual-fallback case; not applicable to
convert M0 to a fully automatic trigger without new CRM signal design.

## Phase 4: User Story 2 - De-identified response collection and CRM rejoin (Priority: P1)

- [x] T011 [US2] Confirm no patient identifier appears in the Form, the flow's
      parameters, or `Response_Data` (Constitution Principle I) — verified by
      field-by-field review in `m0-implementation-notes.md` §5.
- [x] T012 [US2] Token match + rejoin implemented (`submitFeedbackResponse`,
      T007) — response lands only on the matched CRM record.
- [x] T013 [US2] Reuse rejection implemented via `Status != "Issued"` check
      (satisfies spec.md Acceptance Scenario 2 for User Story 2).
- [ ] T014 [US2] **Open / compliance gap**: 24-hour purge of the raw Zoho Forms
      submission entry (constitution's Data Handling & Retention requirement,
      spec.md FR-007) is **not implemented**. `submitFeedbackResponse` copies
      data into CRM but never deletes or schedules deletion of the original
      Form entry. Needs either a scheduled Flow (e.g., a nightly purge flow
      keyed on submission timestamp) or a Zoho Forms retention setting review.
      Treat as a go-live blocker, not a nice-to-have.

**Checkpoint**: Core de-identified rejoin works end-to-end; retention purge is
a known, tracked gap.

## Phase 5: Reporting (supports User Story 3, M0 scope only)

*User Story 3 in spec.md is broader (Track 1 + Track 2, multi-contractor); M0
only needed to prove out its own reporting slice.*

- [x] T015 Build formula columns on "Milestone Instances": Feeling Heard Score,
      Biggest Factor, Additional Comments, Okay to Reach Back Out, Anything Else
      (see `m0-implementation-notes.md` §6 for verbatim formulas and the
      `substring_between` / `INSTR`-2-arg gotchas hit along the way).
- [x] T016 Build/update reports: M0 Submitted Responses (parsed columns, raw
      blob removed), M0 Biggest Factor (bar chart), M0 Feeling Heard
      Distribution (bar chart), M0 % Reachable (KPI tile).
- [x] T017 Assemble "M0 - Free Consult Non-Conversion Feedback" dashboard (7
      panels total, including 3 pre-existing panels).
- [ ] T018 Not started: any per-contractor (Clinician Dashboard) view or
      row-level security — out of scope for M0, tracked at the spec level
      under User Story 3 / FR-010 for whichever milestone first needs
      multi-contractor scoping.

## Phase 6: Test data & validation

- [x] T019 Delete stale/ad hoc test `Milestone_Instances` records via Zoho CRM
      MCP tools (not browser, per standing Leads/Patients access restriction).
- [x] T020 Create 4 sample records against one test Lead, varying Feeling Heard
      Score, Biggest Factor, Reach-Back-Out state, and open-text fields, to
      exercise every dashboard panel.
- [ ] T021 Not yet re-verified: confirm the dashboard panels render correctly
      against the new sample data once CRM-to-Analytics sync catches up (this
      was offered during the build but not followed up on).

## Phase 7: Polish & documentation

- [x] T022 Write `m0-implementation-notes.md` (as-built reference: components,
      verbatim Deluge source, data contract, formulas, CRM field reference,
      gotchas, access constraints).
- [x] T023 Add `CLAUDE.md` establishing the convention that a milestone's
      implementation-notes file is updated in the same session as any change to
      how that milestone works.
- [x] T024 Backfill this plan.md/tasks.md pair for M0 (this task), to bring M0
      into line with the constitution's Rollout Workflow requirement after the
      fact.

## Outstanding items (carried forward, not closed by this backfill)

- **T010**: M0's trigger stays manual by design; revisit only if an automatic
  signal becomes available.
- **T014**: 24-hour raw-Forms-entry purge is unimplemented — real compliance
  gap, go-live blocker.
- **T018**: No per-contractor dashboard scoping yet (not required for M0 alone).
- **T021**: Dashboard-against-new-sample-data re-verification not yet done.
- Constitution-level, cross-milestone blockers (not M0-specific but gate M0
  going live with real data too): `TODO(BAA_SCHEDULE)` unresolved; CRM
  session-count/status population mechanism still an open item for the other
  five milestones.
