# Implementation Plan: Legacy Patient Exclusion (Cutoff Gate)

**Branch**: `004-legacy-patient-exclusion` | **Date**: 2026-09-23 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/004-legacy-patient-exclusion/spec.md`

**Note**: written retroactively; the build described here was already completed
before this plan existed (see spec.md's context note). This documents the design
as it was actually built, not a forward-looking plan.

## Summary

M2 and M3's trigger flows would otherwise fire for any existing patient whose
session count already sits at or above the milestone's threshold the moment the
flow is switched on, none of whom ever agreed to or were scoped for this pipeline.
Add one new shared custom function, `isCreatedAfterCutoff(createdTime) → bool`,
that each flow's If-else ANDs into its existing condition, so only patients created
on or after a fixed cutoff datetime (2026-09-23T00:00:00-04:00) are eligible. M0 is
explicitly excluded per Costin's decision. See research.md for why a custom
function was the only viable mechanism.

## Technical Context

**Language/Version**: Deluge (Zoho Flow custom functions)

**Primary Dependencies**: Zoho Flow (Standard plan), Zoho CRM (`crm_connection`,
via the trigger's own `Updated module entry` merge fields -- no new connection)

**Storage**: N/A -- no new record, no schema change

**Testing**: Structural verification via Execute (quickstart.md §A); live test by
Costin later, folded into the existing M2/M3 coordinated test

**Target Platform**: Zoho Flow workspace `872426000000002011`, folder "Customer
Feedback System"; flows "M2 - Session 3 Trigger" and "M3 - Periodic Check-In
Trigger"

**Project Type**: No-code/low-code workflow configuration + one Deluge function

**Performance Goals**: N/A. Adds one function call per trigger firing; negligible
task-count impact on the 5,000-task/month plan.

**Constraints**: No flow switched ON; no live tests; no Leads/Patients browsing;
M0 untouched per Costin's decision; must not alter `issueFeedbackToken` (002/003)
or any write-back flow

**Scale/Scope**: 1 new function, 2 flows edited (M2, M3), 2 docs updated
(m2-/m3-implementation-notes.md, already done in-session), 1 CLAUDE.md convention
added

## Constitution Check

*Gate: checked after the fact, since the build preceded this plan. Result: PASS.*

| Principle | Assessment |
|---|---|
| I. De-identification by design | Pass. No survey/collection-tool change; the gate only reads `Created_Time`, already available to the trigger, and never reaches Forms. |
| II. Single rejoin point | Pass. No new system holds token + identity; the gate runs entirely inside the existing CRM-triggered Flow, before issuance. |
| III. Automated milestone triggers | Pass. Still fully automatic; this narrows WHEN the automatic trigger fires, it doesn't introduce a manual step. |
| IV. Contractor blindness | Pass. No contractor-facing step involved. |
| V. BAA-gated adoption | Pass. No new product or connection; reuses the trigger's own already-connected CRM data. |
| VI. Internal use only | Pass. No public/marketing connection. |
| VII. Analytics computes derived values | **Deviation, justified**: this is a boolean gate evaluated inside Flow (a Deluge custom function), not an Analytics formula column. Principle VII's stated exception is a rule that must compare across multiple stored records for one patient -- this isn't quite that shape (it's a single-record comparison against a constant), so the stricter reading is that Analytics could compute this too. It was built as a Flow gate instead because the value is needed at *trigger-decision* time (whether to issue at all), before any Analytics-side flag would exist for Analytics to have computed it -- the same reasoning Principle VII's own rationale gives for why some values need to exist pre-query-time. This mirrors the shape of the pipeline's own idempotency checks (`checkAllianceCheckExists`, `checkPeriodicCheckInDue`), which are also single-record Flow-side boolean gates, not Analytics formulas, for the same reason. Recorded here per the Development Workflow's requirement that a documented reason accompany any derived value NOT computed in Analytics. |
| Development Workflow: changes checked against I-IV | Done above. |
| Rollout Workflow (specify → plan → tasks) | **Not followed at build time** -- built directly against the live flows per Costin's explicit instruction, then backfilled into this folder per his explicit follow-up instruction to document it properly. Recorded as a deviation, not silently corrected. |

Post-design re-check: PASS, with the two deviations above both explicitly recorded
rather than assumed.

## Project Structure

### Documentation (this feature)

```text
specs/004-legacy-patient-exclusion/
├── spec.md
├── plan.md                         # this file
├── research.md                     # why a custom function; cutoff value; timezone
├── data-model.md                   # function source, per-flow If-else conditions
├── quickstart.md                   # structural checks already performed
├── checklists/requirements.md
├── tasks.md                        # written after the fact, retroactively
└── implementation-notes.md         # as-built, cross-references m2-/m3-implementation-notes.md
```

### Zoho objects touched

```text
Zoho Flow workspace 872426000000002011 / Customer Feedback System
├── [NEW]   custom function isCreatedAfterCutoff (workspace-shared)
├── [EDIT]  M2 - Session 3 Trigger            (If-else gains a second ANDed clause)
├── [EDIT]  M3 - Periodic Check-In Trigger    (same)
├── [UNTOUCHED] M0 - Lost Lead Feedback Token (per Costin's decision)
└── [UNTOUCHED] issueFeedbackToken, checkAllianceCheckExists, checkPeriodicCheckInDue, all write-back flows
Zoho CRM / Forms / Analytics: no changes
```

**Structure Decision**: A separate feature directory (004), same reasoning as 002:
this is a cross-milestone gate, not a milestone itself, but per-milestone as-built
detail still also lives in 001's own `m2-`/`m3-implementation-notes.md` (already
written, dated 2026-09-23, before this folder existed).

## Build order (as actually executed)

1. Create `isCreatedAfterCutoff` standalone (Built-ins → Developer Tools → Custom
   Functions → "+ Custom Function"), save, verify via Execute against a pre-cutoff
   and a post-cutoff literal (safe: function was new and unshared at the time).
2. M2 first: detach the existing wire between `checkAllianceCheckExists` and
   `If else`; drop `isCreatedAfterCutoff` between them; wire both directions
   (verified via `jsplumb-connected`, not visual position); rename its output
   variable to `isCreatedAfterCutoff_1`; map `createdTime` to
   `${trigger.Created_Time}`; open `If else` and add the second ANDed clause.
3. M3: identical steps, reusing the same function object (confirmed shared:
   `isCreatedAfterCutoff` already appeared in M3's Custom Functions sidebar once
   created from M2's builder).
4. Update `m2-implementation-notes.md` §11 and `m3-implementation-notes.md` §7 in
   the same session; commit and push via the device-bridge + `gh` method.
5. (This folder, added after Costin's follow-up instruction to document it
   properly.)

Main risk actually encountered: none beyond the already-known If-else wiring
gotcha (a node dropped near an endpoint isn't wired until explicitly dragged),
mitigated the same way as 002/003.
