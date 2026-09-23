# Research: Legacy Patient Exclusion (Cutoff Gate)

**Feature**: `004-legacy-patient-exclusion` | **Date**: 2026-09-23

## 0. Why this exists

M2 and M3 have both existed, switched OFF, since before this session. Every
patient already in CRM has a `Created_Time` in the past and, for at least some of
them, a `Session_Count` already at or above M2/M3's own thresholds. The first time
either flow is switched ON, its existing trigger condition alone would fire for
every one of those patients simultaneously -- none of whom were ever told about,
consented to, or scoped for this pipeline when their care began. This isn't a
correctness bug in the pipeline built so far (both flows do exactly what
`specs/001-feedback-collection-pipeline/spec.md` asks), it's a backlog problem
specific to turning a new automated system on inside CRM that already has real
history in it.

Raised by Costin, previously blocked on custom-function availability (see §1);
picked up once that blocker was confirmed resolved.

## 1. Why a custom function, not a no-code condition

- **Decision**: Add a Deluge custom function that does the datetime comparison,
  called from the trigger flow and ANDed into the existing If-else.
- **Rationale**: Zoho Flow's no-code condition builders -- both the trigger's own
  filter step and the If-else condition editor -- offer only text-style operators
  for datetime fields (`starts with`, `ends with`, `contains`, `does not contain`,
  `equals` (case-insensitive), `is blank`, `is empty`, `is not empty`, `equals`,
  `not equals`, `is null`, `is not null`). No `>=`/date-range/relational operator
  exists anywhere in either editor for a datetime field. Confirmed by inspection of
  both editors on M2's flow. A custom function is the only mechanism in this Flow
  plan capable of a relational datetime comparison.
- **Alternatives considered**:
  - *Trigger filter with a hardcoded date-prefix `starts with`/`does not contain`
    hack*: would need one clause per excluded day and can't express "before X",
    only string matches on specific values. Rejected as unworkable for an
    open-ended past.
  - *Zoho Analytics formula column, per Principle VII's default*: would compute
    the flag, but the trigger flow needs the answer before it decides whether to
    issue at all -- Analytics query time is after the point where this decision has
    to be made (same reasoning the pipeline already accepts for
    `checkAllianceCheckExists`/`checkPeriodicCheckInDue`, both Flow-side booleans
    for the identical reason). Rejected, recorded as a Constitution Check
    deviation (plan.md) rather than silently working around Principle VII.
  - *A second, separate function per milestone*: would duplicate the same 4-line
    comparison twice, recreating the drift risk feature 002 was built to close for
    issuance. Rejected in favor of one shared function (spec FR-002).

## 2. Cutoff literal, format, and comparison direction

- **Decision**: `cutoff = "2026-09-23T00:00:00-04:00".toDateTime()`; compare with
  `created >= cutoff`, where `created = createdTime.toDateTime()` and `createdTime`
  is `${trigger.Created_Time}`.
- **Rationale**: Zoho CRM's `Created_Time` merge field was confirmed empirically
  (Execute test) to already come through in the format
  `YYYY-MM-DDTHH:mm:ss+/-HH:mm` (example seen live:
  `2026-09-12T14:10:21-04:00`), and Deluge's `.toDateTime()` parses that format
  directly with no reformatting needed. `-04:00` matches the practice's own
  timezone (America/New_York, EDT) and the offset already present on every
  Created_Time value seen. `>=` (not `>`) means a patient created at exactly the
  cutoff instant is eligible, which is the more natural reading of "created after
  this feature existed" for a feature whose own creation IS the cutoff moment.
- **Verified via Execute** (safe: the function was new and unshared at test time):
  input `2026-09-12T14:10:21-04:00` (pre-cutoff, real patient-record timestamp
  format) → `false`; input `2026-09-25T09:00:00-04:00` (post-cutoff) → `true`.
- **Alternatives considered**: Using `zoho.currenttime` instead of a hardcoded
  literal would make the cutoff silently drift forward every time the function
  runs (effectively "always exclude anyone older than now minus zero", i.e. always
  false) -- clearly wrong, since the point is a fixed historical line, not a
  rolling window. Rejected.

## 3. Function signature and reuse mechanism

- **Decision**: `bool isCreatedAfterCutoff(string createdTime)`, workspace-level,
  created once from M2's builder (Built-ins → Developer Tools → Custom Functions
  → "+ Custom Function", never cloned), then dragged onto M3's canvas from its
  own Custom Functions sidebar list, where it already appeared as a shared object.
- **Rationale**: Zoho Flow custom functions are workspace-level and shared across
  flows (established in 001 `m2-implementation-notes.md` §3.2, reused
  deliberately by feature 002 for `issueFeedbackToken`, and reused again here).
  Output Variable Name is scoped per flow even though the function body is
  shared, so each caller renames its own instance (`isCreatedAfterCutoff_1` on
  both M2 and M3 -- same name, different flow, not a collision).
- **Alternatives considered**: none beyond §1's rejected options; this part
  followed 002's already-established pattern directly.

## 4. Where it sits in the trigger chain

- **Decision**: Insert between the milestone's existing idempotency-check function
  (`checkAllianceCheckExists` for M2, `checkPeriodicCheckInDue` for M3) and the
  `If else` node, then add a second ANDed clause to that same `If else` rather
  than adding a second decision node.
- **Rationale**: Keeps exactly one decision point per flow (spec FR-003); the
  existing True branch (-> `issueFeedbackToken` → Send email) needed no change
  at all, only the condition feeding it.
- **Verified**: wiring confirmed via the `jsplumb-connected` DOM check (per 002's
  implementation notes §5.1), not by visual node position -- both directions
  (idempotency-check → cutoff-gate, cutoff-gate → If-else) were explicitly
  dragged and re-verified after the drop landed the node in a merely-adjacent,
  not-yet-wired position on both flows (same gotcha documented in 001
  `m3-implementation-notes.md` §6's "Correction to §1.5's wiring claim").

## 5. M0 explicitly excluded

- **Decision**: No cutoff gate added to M0 ("M0 - Lost Lead Feedback Token").
- **Rationale**: Costin's explicit instruction ("no need for cutoff at M0"). M0
  fires on a Lead status change (Lead Status → Lost Lead), a one-time event with
  no equivalent "already past the threshold at switch-on time" backlog condition --
  unlike M2/M3's session-count thresholds, a lead's status doesn't retroactively
  become "Lost" just because the flow was switched on; it changes when a real
  action happens after that point. No further reasoning was requested or
  explored beyond Costin's decision.
