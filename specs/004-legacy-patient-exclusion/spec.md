# Feature Specification: Legacy Patient Exclusion (Cutoff Gate)

**Feature Branch**: `004-legacy-patient-exclusion`

**Created**: 2026-09-23

**Status**: Draft

**Input**: User description: "Pick up the cutoff-exclusion function now that custom
functions are unblocked" (verbal instruction, continuing a feature discussed but not
built in an earlier session), followed by "Yes, please document properly" when asked
whether to backfill a formal spec-kit feature folder for it, and "no need for cutoff
at M0" when asked whether M0 should get the same gate.

<!--
  Context for readers: this spec is written RETROACTIVELY. The cutoff gate described
  here was designed, built, wired, and verified in Zoho Flow before this spec-kit
  folder existed -- Costin asked for it directly, and the constitution's Rollout
  Workflow (specify → plan → tasks) wasn't followed at build time. This folder
  documents the decision and the as-built result properly, per Costin's explicit
  instruction to backfill it. Nothing in this spec describes new, not-yet-built work;
  every requirement below is already satisfied by the M2/M3 build documented in
  `specs/001-feedback-collection-pipeline/m2-implementation-notes.md` §11 and
  `m3-implementation-notes.md` §7. Costin separately decided M0 does NOT need this
  gate (see Assumptions).
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Existing patients don't get surprise feedback requests (Priority: P1)

As the practice, when the M2 (Session 3) or M3 (Periodic Check-In) session-count
trigger fires for a patient who was already in the system before this pipeline
existed, that patient should NOT receive a feedback request purely because their
already-high session count crosses the threshold the moment the trigger is switched
on. Feedback requests should only go to patients whose CRM record was created after
the pipeline had a defined cutoff point.

**Why this priority**: Without this, switching M2/M3 on for the first time would
immediately fire for every existing patient whose session count already happens to
sit at or above the trigger threshold, none of whom ever agreed to or were scoped
for this system. That is both a bad first impression and arguably outside what
Principle III's "system-detected condition" was meant to catch retroactively.

**Independent Test**: For a patient record created before the cutoff datetime, with
a session count that satisfies the milestone's own trigger condition, confirm the
trigger flow evaluates false and does not call `issueFeedbackToken`. For a patient
record created on or after the cutoff, confirm the same trigger evaluates true and
issues normally.

**Acceptance Scenarios**:

1. **Given** a patient created before 2026-09-23T00:00:00-04:00 whose session count
   reaches the M2 threshold, **When** the trigger fires, **Then** the If-else
   evaluates to False and no feedback request is issued.
2. **Given** a patient created on or after 2026-09-23T00:00:00-04:00 whose session
   count reaches the M2 threshold, **When** the trigger fires, **Then** the If-else
   evaluates to True (provided the milestone's own condition is also true) and a
   feedback request is issued exactly as before this feature existed.
3. **Given** the same two scenarios for M3's periodic checkpoints (8/16/24/32/40),
   **Then** the same cutoff behavior applies.

---

### User Story 2 - One cutoff, reused, not redefined per milestone (Priority: P2)

As whoever maintains this pipeline, the cutoff check should exist in one place, so
the cutoff date (or the comparison logic) is changed once and every milestone that
uses it picks up the change, rather than three flows each carrying their own copy
that can drift.

**Why this priority**: The pipeline already has this problem solved once for
issuance (`issueFeedbackToken`, feature 002); repeating a per-flow copy here would
recreate the exact drift risk that feature closed, for a second piece of logic. P2
because the pipeline still works without sharing it, but degrades as more milestones
adopt it.

**Independent Test**: Confirm by inspection that M2 and M3 both call the same
function object (Flow's "used in the following flows" listing shows both), not two
separately-created functions with the same body.

**Acceptance Scenarios**:

1. **Given** the cutoff gate exists on M2 and M3, **When** the shared function is
   opened in the builder, **Then** it is the same function object on both canvases.

---

### User Story 3 - The as-built record matches reality (Priority: P3)

As the next person (or session) to work on this pipeline, I need this feature's
design decision, its as-built wiring, and the fact that M0 deliberately does NOT use
it, written down in one place, rather than left as an undocumented ad hoc change.

**Why this priority**: Required by CLAUDE.md's same-session implementation-notes
convention and by Costin's explicit request to document this properly. P3 because it
follows the build rather than gating it.

**Independent Test**: Read `m2-implementation-notes.md` §11 and
`m3-implementation-notes.md` §7 and confirm each points here for the shared
function's source and rationale; read `m0-implementation-notes.md` and confirm it
records the explicit decision not to add this gate.

**Acceptance Scenarios**:

1. **Given** this feature is documented, **When** a future session reads
   `CLAUDE.md`, **Then** it states the cutoff-gate convention and which milestones
   it applies to.

### Edge Cases

- **Patient record has no Created Time value** (shouldn't happen in CRM, but if a
  record were somehow missing it): `.toDateTime()` on a null/empty string throws in
  Deluge rather than silently returning false. Not specifically handled: this failure
  mode is the same class already accepted for `issueFeedbackToken`'s own `throw`
  (research.md §1), and a record with no Created_Time would be a CRM data problem,
  not a scenario this feature needs to paper over.
- **Cutoff date itself, exactly at the boundary**: the comparison is `>=` (created at
  or after cutoff is eligible), so a patient created at exactly
  2026-09-23T00:00:00-04:00 IS eligible. Chosen deliberately (see research.md §2).
- **M0**: explicitly out of scope. Costin's instruction: "no need for cutoff at M0."
  M0 fires on a Lead status change (Lost Lead), not a session-count/backlog
  condition, so there is no equivalent "already has a high count" legacy case for it.
- **Timezone**: the cutoff literal is hardcoded to `-04:00` (America/New_York /
  Eastern, the practice's own timezone), matching the format Zoho CRM's own
  `Created_Time` values already carry (confirmed empirically, see research.md §2).
  If the practice ever operates across a DST boundary near a future cutoff change,
  the literal offset would need updating by hand; this is a known limitation, not
  handled dynamically.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: M2 and M3's trigger flows MUST NOT issue a feedback request for a
  patient whose CRM record `Created_Time` is earlier than a defined cutoff datetime,
  even when the milestone's own trigger condition is otherwise satisfied.
- **FR-002**: The cutoff comparison MUST be implemented once, in one shared
  component invoked by every milestone trigger flow that uses it, not copied per
  flow.
- **FR-003**: The cutoff comparison MUST be evaluated in addition to (ANDed with)
  each milestone's existing trigger condition, not in place of it, as a second
  clause in the same If-else rather than a new decision node.
- **FR-004**: The cutoff datetime MUST be represented in the same format Zoho CRM's
  own `Created_Time` field already produces, so no format conversion is needed at
  comparison time.
- **FR-005**: The change MUST NOT alter `issueFeedbackToken` (feature 002/003) or
  any write-back flow, form, CRM field, picklist, Analytics formula, report, or
  dashboard.
- **FR-006**: The change MUST NOT switch any flow ON, and MUST NOT perform any live
  or end-to-end test; verification is structural only (per standing instruction).
- **FR-007**: M0 MUST NOT receive this gate, per Costin's explicit decision.
- **FR-008**: `m2-implementation-notes.md`, `m3-implementation-notes.md`, and
  `CLAUDE.md` MUST describe the cutoff gate as a cross-milestone convention.

### Key Entities

- **Cutoff gate**: the single shared piece of logic (`isCreatedAfterCutoff`)
  replacing what would otherwise be a per-flow hardcoded date check. Input: a
  datetime string (a record's `Created_Time`). Output: boolean.
- **Milestone trigger's If-else condition**: unchanged in kind, extended with one
  additional ANDed clause referencing the cutoff gate's output.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of patients created before the cutoff are excluded from M2 and M3
  issuance, verified structurally (condition logic inspected, not live-tested).
- **SC-002**: The cutoff comparison exists in exactly 1 place (1 shared function),
  reused by 2 flows (M2, M3), not 2 separate copies.
- **SC-003**: 0 changes to `issueFeedbackToken`, write-back flows, forms, CRM schema,
  or Analytics objects.
- **SC-004**: M0 unmodified.

## Assumptions

- The cutoff datetime is 2026-09-23T00:00:00-04:00 -- the date this feature was
  built and this decision was made -- not a date tied to any particular patient
  record or historical event. Costin did not specify an alternate date; "patients
  created before this feature existed" was read literally as "before today."
- M0 does not need this gate. Costin's explicit decision (see Edge Cases): M0 fires
  on a Lead status change, which has no equivalent backlog-of-already-qualifying
  records problem the way a session-count threshold does.
- Whether M4/M5 (not yet built) will need the same gate is not decided here; the
  shared function is available to them if their own planning concludes they do,
  following the same reuse pattern feature 002 established for `issueFeedbackToken`.
- Live end-to-end testing is run by Costin, per his standing instruction; this
  feature's own verification is structural only (already completed, see
  `implementation-notes.md`).
