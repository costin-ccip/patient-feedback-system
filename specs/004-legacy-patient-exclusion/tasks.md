---
description: "Task list for 004 - Legacy Patient Exclusion (Cutoff Gate), written retroactively after implementation, per Costin's instruction to document it properly"
---

# Tasks: Legacy Patient Exclusion (Cutoff Gate)

**Input**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `quickstart.md`

**Note**: RETROACTIVE. Unlike 001/002's task lists (written before the work, then
checked off as completed), every task below was already done by the time this file
was written -- checked off immediately, with a note where the actual order departed
from the ideal specify → plan → tasks sequence.

**Status 2026-09-23**: T001-T012 done; T013 (this documentation) done; T014-T015
are Costin's/future sessions'.

## Phase 1: Setup

- [x] T001 Confirm the earlier blocker ("Your current plan does not support custom
      functions. Upgrade.") is resolved -- done by creating `isCreatedAfterCutoff`
      with no upgrade prompt appearing.
- [x] T002 Confirm M2 and M3 trigger flows are OFF before editing anything.

## Phase 2: Foundational (blocks both caller edits)

- [x] T003 Create `isCreatedAfterCutoff` via Built-ins → Developer Tools → Custom
      Functions → "+ Custom Function" (not cloned): return type `bool`, input
      `createdTime` (string). Paste the body from `data-model.md`. Save.
- [x] T004 Verify via Execute against a pre-cutoff and post-cutoff literal (safe:
      function was new/unshared at the time). Both results correct (quickstart.md
      §B).

**Checkpoint**: shared cutoff function exists, saves cleanly, and is verified
correct in isolation; no flow uses it yet.

## Phase 3: User Story 1 + 2 - Both flows gate on the cutoff, sharing one function (P1/P2)

### M2 - Session 3 Trigger

- [x] T005 [US1] Detach the existing wire from `checkAllianceCheckExists` to
      `If else`.
- [x] T006 [US1] Drop `isCreatedAfterCutoff` on the canvas; wire
      `checkAllianceCheckExists → isCreatedAfterCutoff → If else` explicitly in
      both directions; verify via `jsplumb-connected`, not visual position.
- [x] T007 [US1] Rename output variable to `isCreatedAfterCutoff_1`; map
      `createdTime` to `${trigger.Created_Time}` (found via the Insert Variable
      panel's "Updated module entry" → "Time created", after correcting an
      earlier mis-click where the merge tag landed in the panel's search box
      instead of the target field).
- [x] T008 [US1] Open `If else`; add a second ANDed clause,
      `isCreatedAfterCutoff_1 is true`, alongside the existing
      `checkAllianceCheckExists_1 is false`. Save.

### M3 - Periodic Check-In Trigger

- [x] T009 [US1] Same as T005 for M3 (existing wire:
      `checkPeriodicCheckInDue → If else`).
- [x] T010 [US1] Same as T006 for M3, dragging the SAME `isCreatedAfterCutoff`
      function object from M3's own Custom Functions sidebar list (confirms
      US2 -- it already appeared there as a shared object).
- [x] T011 [US1] Same as T007 for M3.
- [x] T012 [US1] Same as T008 for M3: second ANDed clause
      `isCreatedAfterCutoff_1 is true` alongside
      `checkPeriodicCheckInDue_1 is true`. Save.

**Checkpoint**: SC-001/SC-002 verifiable structurally on both flows; neither
switched ON.

## Phase 4: User Story 3 - Documentation

- [x] T013 [US3] Write this feature folder (spec.md, plan.md, research.md,
      data-model.md, quickstart.md, tasks.md, checklists/requirements.md,
      implementation-notes.md), cross-referencing the dated change sections
      already written in `m2-implementation-notes.md` §11 and
      `m3-implementation-notes.md` §7 earlier the same day. Update those two
      sections' "Open items" paragraph to point here instead of leaving the
      backfill question open. Add a CLAUDE.md convention note.

## Phase 5: Validation / open follow-through (not this session)

- [ ] T014 Costin runs quickstart.md §C (folded into the existing M2/M3 live
      test), when he runs that test.
- [ ] T015 If M4/M5 planning concludes they need the same gate, reuse
      `isCreatedAfterCutoff` following this feature's pattern rather than writing
      a new function (see research.md §3).

## Dependencies

T001-T002 → T003-T004 → (T005-T008, T009-T012 -- M3's block reuses the function
M2's block creates, otherwise independent) → T013 → T014-T015.
