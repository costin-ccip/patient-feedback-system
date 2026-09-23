# Implementation Notes: Legacy Patient Exclusion (Cutoff Gate)

**Feature**: `004-legacy-patient-exclusion` | Built 2026-09-23 | Documented
retroactively 2026-09-23 (same day, later in the session)

This file is the as-built record for this feature specifically (mirroring how
`specs/002-remove-subflow-dependency/implementation-notes.md` holds the deep
narrative for that feature). The per-milestone summaries live in
`specs/001-feedback-collection-pipeline/m2-implementation-notes.md` §11 and
`m3-implementation-notes.md` §7; this file is where builder gotchas, chronology,
and things that didn't make it into those shorter sections are recorded.

## 1. Trigger for this work

The cutoff-exclusion idea itself predates this folder: it had been discussed and
blocked earlier ("Your current plan does not support custom functions. Upgrade.")
in an earlier session, before feature 002 confirmed custom functions were
available on the Standard plan after all. When 002/003 finished, Costin was told
this earlier blocker looked resolved, and explicitly said: "pick up the
cutoff-exclusion function now that custom functions are unblocked." The function
was designed and built directly against the live M2 and M3 flows in that same
session, without first going through specify → plan → tasks -- a deviation from
the constitution's Rollout Workflow, flagged to Costin immediately afterward
alongside the completed build. Costin's follow-up instructions: "Yes, please
document properly" (backfill this folder) and "no need for cutoff at M0" (scope
decision). This folder is that backfill.

## 2. Function build and verification

Created via Built-ins → Developer Tools → Custom Functions → "+ Custom
Function" on M2's builder (not cloned from anything). Name
`isCreatedAfterCutoff`, return type `bool`, input `createdTime` (string). Body
pasted via the CodeMirror `setValue()` method (typing `${` directly is known to
corrupt chip-based parameter fields elsewhere in this builder, though the
function-body editor itself is plain CodeMirror and wasn't actually at risk here
-- used out of caution/consistency with how `issueFeedbackToken` was edited).

```text
bool isCreatedAfterCutoff(string createdTime)
{
	cutoff = "2026-09-23T00:00:00-04:00".toDateTime();
	created = createdTime.toDateTime();
	return created >= cutoff;
}
```

Saved successfully (no plan-tier upgrade prompt -- confirms the earlier blocker is
gone). Execute button is disabled until after the first Save, per the established
gotcha; enabled and used immediately after. Tested via Execute, safe at this point
because the function was not yet attached to any flow (no "used in the following
flows" warning risk):

| Input | Result |
|---|---|
| `2026-09-12T14:10:21-04:00` | `false` |
| `2026-09-25T09:00:00-04:00` | `true` |

Both correct. Confirms `.toDateTime()` parses Zoho CRM's real
`YYYY-MM-DDTHH:mm:ss+/-HH:mm` datetime format for relational (`>=`) comparison --
this had been an open uncertainty in an earlier session and is now empirically
settled.

## 3. M2 - Session 3 Trigger build

Starting canvas (confirmed via screenshot): `trigger (Updated module entry) ->
checkAllianceCheckExists → If else (True) → issueFeedbackToken → Send email`,
with On Error branches on `issueFeedbackToken` and Send email (feature 003). Flow
OFF, Draft badge.

1. Detached the wire `checkAllianceCheckExists → If else` via the scissors icon
   that appears on hover, confirmed "Delete" on the detach-wire dialog.
2. Dragged `isCreatedAfterCutoff` from the sidebar Custom Functions list onto the
   canvas.
3. Wired `checkAllianceCheckExists` output → `isCreatedAfterCutoff` input, and
   `isCreatedAfterCutoff` output → `If else` input, via explicit drag between
   endpoint coordinates (not by dropping the node near an endpoint and assuming
   it connects -- see 001 `m3-implementation-notes.md` §6's wiring gotcha).
   Verified via
   `document.querySelectorAll('.jtk-endpoint.jsplumb-connected, [class*="endpoint"].jsplumb-connected').length`
   reading 16 after both drags (up from 14 before).
4. Opened `isCreatedAfterCutoff`'s parameter panel: renamed Output Variable Name
   from the default `isCreatedAfterCutoff_8` to `isCreatedAfterCutoff_1`
   (triple-click-select + type), for consistency with `issueFeedbackToken_1`'s
   naming convention.
5. Mapped `createdTime`: first attempt landed in the wrong place -- clicked the
   Insert Variable panel's search box and typed "created", then clicked "Time
   created" in the filtered results, which inserted the merge tag into the
   SEARCH BOX itself (it displayed the literal text
   `created${trigger.Created_Time}`) rather than into the `createdTime` field,
   because the search box still had focus. Fixed by clearing the search box,
   clicking directly into the `createdTime` field to give it focus, then
   expanding "Updated module entry" WITHOUT using search (browsing the
   alphabetical list down to "Time created" instead) before clicking the field
   name. Confirmed via screenshot: `createdTime` shows the chip
   `Updated module entry → Time created`. Saved.
6. Opened `If else`'s condition editor (existing single clause:
   `checkAllianceCheckExists_1 is false`). Clicked the `+` to add an ANDed
   clause, selected `isCreatedAfterCutoff_1` (found by typing into the field
   dropdown's own search, which is a different control from the Insert Variable
   panel's search and doesn't have the same mis-click risk), set operator
   `is true`. Saved (Done).

Final condition: `checkAllianceCheckExists_1 is false` **AND**
`isCreatedAfterCutoff_1 is true`. Flow left OFF.

## 4. M3 - Periodic Check-In Trigger build

Identical steps, same shared function object (confirmed: it already appeared in
M3's own Custom Functions sidebar list without needing to be recreated -- direct
evidence for spec User Story 2 / SC-002). One difference from M2: the node's
drop position landed visually overlapping the `If else` diamond's "True" label,
which was left as-is (cosmetic only, not a wiring problem -- confirmed by the
same `jsplumb-connected` check reading 16 afterward, matching M2's count).

`createdTime` mapping went cleanly this time (the search-box mis-click from M2
was avoided by clicking directly into the target field before browsing, not
searching, the variable list).

Final condition: `checkPeriodicCheckInDue_1 is true` **AND**
`isCreatedAfterCutoff_1 is true`. Flow left OFF.

## 5. Documentation and push

`m2-implementation-notes.md` §11 and `m3-implementation-notes.md` §7 were
written in the same session as the build (before this folder existed), then
committed and pushed via the device-bridge + `gh` method (commit `c6c45c0`).
After Costin's follow-up instructions, this folder was added and those two
sections' "Open items" paragraphs were updated to point here instead of leaving
the backfill question open; a CLAUDE.md convention note was added; M0 was
confirmed untouched.

## 6. Things deliberately not done

- No flow was switched ON.
- No live/end-to-end test was run.
- Leads/Patients modules were not opened in the browser or queried for real
  records at any point in this feature's build.
- `issueFeedbackToken`, both write-back flows, and all Analytics/Forms/CRM-schema
  objects were not touched.
- M0 was not given this gate (Costin's explicit decision).
