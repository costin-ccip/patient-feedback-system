# Data Model: Legacy Patient Exclusion (Cutoff Gate)

**Feature**: `004-legacy-patient-exclusion` | **Date**: 2026-09-23

No CRM, Forms, or Analytics schema changes. No new record type. This file pins the
function body and the per-flow condition/wiring the build reproduces.

## `isCreatedAfterCutoff` (new shared custom function)

Signature (Create Function wizard): name `isCreatedAfterCutoff`, return type
`bool`, input `createdTime` (string).

```text
bool isCreatedAfterCutoff(string createdTime)
{
	cutoff = "2026-09-23T00:00:00-04:00".toDateTime();
	created = createdTime.toDateTime();
	return created >= cutoff;
}
```

Verified via Execute (function was new/unshared at test time): input
`2026-09-12T14:10:21-04:00` → `false`; input `2026-09-25T09:00:00-04:00` ->
`true`. See research.md §2 for why this literal, format, and comparison
direction.

## Per-flow wiring and condition

| | M2 - Session 3 Trigger | M3 - Periodic Check-In Trigger |
|---|---|---|
| Node chain | `trigger → checkAllianceCheckExists → isCreatedAfterCutoff → If else → issueFeedbackToken → Send email` | `trigger → checkPeriodicCheckInDue → isCreatedAfterCutoff → If else → issueFeedbackToken → Send email` |
| `createdTime` parameter | `${trigger.Created_Time}` | `${trigger.Created_Time}` |
| Output variable name | `isCreatedAfterCutoff_1` | `isCreatedAfterCutoff_1` (separate instance, same name -- scoped per flow) |
| If-else condition (was) | `checkAllianceCheckExists_1 is false` | `checkPeriodicCheckInDue_1 is true` |
| If-else condition (now) | `checkAllianceCheckExists_1 is false` **AND** `isCreatedAfterCutoff_1 is true` | `checkPeriodicCheckInDue_1 is true` **AND** `isCreatedAfterCutoff_1 is true` |
| Flow state | OFF (unchanged) | OFF (unchanged) |

`${trigger.Created_Time}` is the "Time created" field already exposed by both
flows' `Updated module entry` trigger (Zoho CRM connection) -- no new merge field,
no new connection.

## M0 - Lost Lead Feedback Token

Untouched. No `isCreatedAfterCutoff` call, no condition change. See research.md §5.
