---
description: "Task list for 003 - Issuance Privacy and Send-Failure Handling, generated prospectively before implementation"
---

# Tasks: Issuance Privacy and Send-Failure Handling

**Input**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `quickstart.md`

**Note**: PROSPECTIVE; check off only when done. No flow ON, no live test.

**Status 2026-09-23**: T002-T010 done; T001 left for Costin.

## Phase 1: Foundational

- [ ] T001 **(Costin — CRM Setup wouldn't load for automation; see implementation-notes §3)** Add "Send Failed" to `Milestone_Instances.Status` in CRM Setup; confirm
      via `getFields`.

## Phase 2: User Story 1 - No emails in record names (P1)

- [x] T002 [US1] Edit `issueFeedbackToken` Name line per data-model.md; confirm the
      "used in flows" dialog; re-read source.
- [x] T003 [US1] Rename the 3 existing records per data-model.md via CRM MCP
      `updateRecords`; re-run the `Name like '%@%'` query → 0.
- [x] T004 [US1] Confirm (view only) Email isn't selected in the Analytics CRM sync
      for Milestone Instances. (Done during research; re-confirm not needed unless
      the setup changes.)

## Phase 3: User Story 2 - Failure paths (P1)

- [x] T005 [US2] M3: On Error of Send email → CRM Update module entry (Status Send
      Failed) → alert Send email; On Error of issueFeedbackToken → alert Send email.
      Verify endpoints `jsplumb-connected`, read back values after reload.
- [x] T006 [US2] M2: same as T005.
- [x] T007 [US2] M0: same as T005.

## Phase 4: Documentation & push

- [x] T008 Write `implementation-notes.md` (as-built).
- [x] T009 Update 002 contract + implementation notes (function changed), 001
      m0/m2/m3 notes (dated pointer), CLAUDE.md convention (failure paths required
      for M4/M5).
- [x] T010 Commit and push via device bridge + gh.

## Phase 5: Validation (Costin)

- [ ] T011 Live test per quickstart §B, including a forced email failure.
- [ ] T012 Decide whether to stop syncing the Leads module's Last Name/Email into
      Analytics (flagged in spec Assumptions; out of scope here).
