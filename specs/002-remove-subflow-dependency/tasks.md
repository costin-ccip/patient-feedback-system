---
description: "Task list for 002 - Remove Subflow Dependency from Token Issuance, generated prospectively before implementation"
---

# Tasks: Remove Subflow Dependency from Token Issuance

**Input**: `plan.md`, `spec.md`, `research.md`, `data-model.md`,
`contracts/issue-feedback-token.md`, `quickstart.md`

**Note**: PROSPECTIVE. Same discipline as 001's milestone task lists: start unchecked,
check off only when actually done. No task switches a flow ON or runs a live test.

**Status 2026-09-23**: T001-T023 done (see `implementation-notes.md`); T024 done;
T025-T026 are Costin's.

## Phase 1: Setup

- [x] T001 Capture the subflow's internals (both functions verbatim, CRM field
      mapping, email template) and M0's live "Call a subflow" values into
      `research.md` §0 (done during research, 2026-09-23).
- [x] T002 Confirm all three caller flows (M0, M2, M3) and the subflow are OFF
      before editing anything.

## Phase 2: Foundational (blocks all caller edits)

- [x] T003 Create `issueFeedbackToken` via Built-ins → Developer Tools → Custom
      Functions → "+ Custom Function" (not cloned): return type `map`, inputs per
      the contract. Paste the body from `data-model.md` via CodeMirror `setValue()`.
      Save.
- [x] T004 (Not needed: `throw` and the options form both saved.) If the save rejects `throw` or the `createRecord` options form, apply the
      fallback in `research.md` §3/§4, save, and record the deviation verbatim.
- [x] T005 Re-read the saved source from the live editor and confirm it matches what
      will be documented.

**Checkpoint**: shared issuance function exists and saves cleanly; no flow uses it yet.

## Phase 3: User Story 1 + 2 - Callers use the shared function, no subflow (P1/P2)

### M3 - Periodic Check-In Trigger

- [x] T006 [US1] Re-read the live "Call a subflow" parameter values and diff them
      against `data-model.md`'s M3 row; correct data-model.md if they differ.
- [x] T007 [US1] Delete the "Call a subflow" node; place `issueFeedbackToken` on the
      If-else True branch; map params per data-model.md.
- [x] T008 [US1] Place Zoho Mail "Send email" after it; configure connection, From,
      To, Subject, HTML body per data-model.md; token merge field
      `${issueFeedbackToken_1.token}` (use the actual output variable name).
- [x] T009 [US1] Verify: connector count, screenshot, param/body read-back, flow OFF.

### M2 - Session 3 Trigger

- [x] T010 [US1] Same as T006 for M2.
- [x] T011 [US1] Same as T007 for M2.
- [x] T012 [US1] Same as T008 for M2.
- [x] T013 [US1] Same as T009 for M2.

### M0 - Lost Lead Feedback Token

- [x] T014 [US1] Values already captured (research.md §0); re-confirm unchanged.
- [x] T015 [US1] Delete "Call a subflow"; place `issueFeedbackToken` on the trigger's
      output; map params (leadId = `${trigger.id}`, patientId empty).
- [x] T016 [US1] Place and configure "Send email" per data-model.md's M0 row.
- [x] T017 [US1] Verify as T009.

### Shared

- [x] T018 [US2] Open `issueFeedbackToken` in the editor and confirm (without saving
      a change) that it's the one function object used by all three flows (e.g. the
      Built-ins list shows one entry; any save prompt lists M0, M2, M3). Cancel out.
- [x] T019 [US1] Retire the subflow: rename to `[RETIRED] Subflow - Issue Feedback
      Token`, leave OFF, do not delete. Update its description to say it's retired
      and point to this feature.
- [x] T020 [US1] Confirm no remaining active (non-retired) flow contains "Call a
      subflow".

**Checkpoint**: SC-001/SC-002/SC-003 verifiable structurally.

## Phase 4: User Story 3 - Documentation (same session)

- [x] T021 [US3] Write `implementation-notes.md` for 002 (as-built: function source,
      per-flow node chains and values, deviations, gotchas).
- [x] T022 [US3] Update 001 `m0-implementation-notes.md` (component table, §11
      mechanism note), `m2-implementation-notes.md` §3/§3.3, and
      `m3-implementation-notes.md` §1.5 to describe the new wiring (dated changelog
      note, old subflow text kept as history).
- [x] T023 [US3] Update `CLAUDE.md` with the cross-milestone convention (no subflows;
      new milestones call `issueFeedbackToken` + own Send email step). Update 001
      `m3-tasks.md`/`m3-implementation-notes.md` remaining-work notes so M3's pending
      live test (T020-T023 there) covers the new path.
- [x] T024 Commit and push via the device-bridge + `gh` method in CLAUDE.md.

## Phase 5: Validation (Costin)

- [ ] T025 Costin runs `quickstart.md` §B (live test across M0/M2/M3), folded into
      the existing coordinated M0/M2/M3 live test.
- [ ] T026 Costin decides on the two out-of-scope findings in research.md §5
      (email in record Name/Analytics; orphan record on mail failure).

## Dependencies

T002 → T003-T005 → (T006-T009, T010-T013, T014-T017 in that order, each block
independent of the others) → T018-T020 → T021-T024 → T025.
