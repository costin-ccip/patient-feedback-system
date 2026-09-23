# Quickstart / Validation: Legacy Patient Exclusion (Cutoff Gate)

## A. Structural verification (already performed, this session)

For both M2 and M3 trigger flows:

1. Flow is still **OFF**.
2. `isCreatedAfterCutoff` sits between the milestone's idempotency-check function
   and `If else`; both connections confirmed via
   `document.querySelectorAll('.jtk-endpoint.jsplumb-connected, [class*="endpoint"].jsplumb-connected').length`
   reading 16 (up from 14 before the two new wires -- one wire in, one wire out),
   not by visual position alone.
3. Output Variable Name renamed to `isCreatedAfterCutoff_1` on both flows (read
   back from the parameter panel, not assumed).
4. `createdTime` parameter shows the chip `Updated module entry → Time created`
   (i.e. `${trigger.Created_Time}`) on both flows (read back from the panel, not
   assumed).
5. `If else`'s condition list shows both clauses, ANDed, exact variable names
   confirmed via DOM read where the panel truncated the label visually
   (`checkPeriodicCheckInDue_1`, `isCreatedAfterCutoff_1`).
6. `isCreatedAfterCutoff` source, re-read from the live editor, matches
   data-model.md.
7. No write-back flow, form, CRM field, or Analytics object touched (no edits
   made to any of them this session).
8. M0 - Lost Lead Feedback Token: confirmed unmodified (no `isCreatedAfterCutoff`
   reference anywhere in that flow).

## B. Function-level test (already performed, this session)

Via Execute, before the function was attached to any flow (safe -- no "used in the
following flows" warning at that point):

| Input (`createdTime`) | Expected | Result |
|---|---|---|
| `2026-09-12T14:10:21-04:00` (pre-cutoff) | `false` | `false` |
| `2026-09-25T09:00:00-04:00` (post-cutoff) | `true` | `true` |

## C. Live test (Costin runs this later; not an automated session)

Folded into the existing M2/M3 coordinated live test (see
`specs/002-remove-subflow-dependency/quickstart.md` §B, steps 2-3), with one
addition: before switching M2/M3 ON, confirm with Costin whether any real
existing patient's `Session_Count` already sits at or above either milestone's
threshold. If so, expect NO feedback request for a patient created before
2026-09-23, and a normal one for a patient created on/after that date whose count
also crosses the threshold. Clean up any test Milestone_Instances records via CRM
MCP tools, as in 001/002.
