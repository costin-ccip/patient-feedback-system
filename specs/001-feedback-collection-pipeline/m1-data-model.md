# Phase 1 Data Model: M1 - Baseline Intake Survey

> **RETIRED (2026-09-11)**: Milestone 1 has been eliminated per an operations-lead
> decision following a review of clinical/EHR overlap — see `spec.md`'s fifth-pass
> revision note and Assumptions. This file is kept as the historical record of the
> data model actually built, not deleted; it no longer reflects the live system. The
> corresponding Zoho objects are being decommissioned — see `m1-implementation-notes.md`'s
> own RETIRED note for their disposition.

**Input**: `m1-plan.md`, `m1-research.md`

**Date**: 2026-09-06

## Milestone_Instances usage for M1

M1 reuses the existing `Milestone_Instances` module (from M0) with these field
values and one structural difference from M0:

| Field | M0 usage | M1 usage |
|---|---|---|
| `Milestone` | `"0 - No Conversion"` | `"1 - Baseline Intake"` |
| `Lead_Reference` (text) | Populated — M0 subjects are pre-conversion Leads | **Not used** |
| `Patient` (lookup to `Patients1`) | Never populated (no converted patient yet) | **Populated** — M1 subjects are converted patients, and rejoining token to identity only inside CRM is the whole point of Principle II (Single Rejoin Point) |
| `Status`, `Token`, `Expiry_Date_Time`, `Submitted_Date_Time`, `Response_Data` | Same picklist/lifecycle as M0 | Same lifecycle, different content |

**Why this changes**: M0 fires before a Lead ever becomes a patient, so there's
no `Patient` record to link yet — `Lead_Reference` was the only identity handle
available. M1 fires on `Session Count == 1`, a field that only exists on
already-converted `Patients` records, so the natural, correct link is the
`Patient` lookup field, not `Lead_Reference`. This is a real, intentional
divergence from M0's field usage, not an inconsistency to fix.

**Process note**: because the `Patient` module is the standing-restricted one,
sample/test data for M1 needs a test Patient record ID supplied by Costin
(the same pattern used for the test Lead in M0), rather than looked up via
CRM query, when we get to the testing phase.

## Trigger flow entity: "M1 - Session 1 Trigger" (new)

Watches `Patients1.Session_Count` (confirmed field API name, integer) for the
transition to `1` (via the existing payment-triggered workflow that increments
it — M1 does not touch that mechanism, only reacts to it) and, on that event:

1. **Idempotency check** (new requirement, not present in M0's manual-trigger
   design): before creating a `Milestone_Instances` record, check whether one
   already exists for this `Patient` + `Milestone = "1 - Baseline Intake"`.
   If yes, do nothing. This satisfies spec.md FR-002 ("MUST NOT send more than
   one feedback request for the same milestone instance for the same patient
   or prospect") — M0 didn't need this check because a human clinician
   triggered issuance manually and wouldn't double-fire it; M1's automatic
   trigger could otherwise double-fire if the workflow rule runs more than
   once (e.g., a payment record edited/corrected after the fact).
2. If no existing record: create a new `Milestone_Instances` record —
   `Patient` = the triggering patient, `Milestone = "1 - Baseline Intake"`,
   `Status = "Issued"`, fresh `Token`, `Expiry_Date_Time` set (reuse M0's
   expiry window unless Costin wants a different one for M1).
3. Send the Wellbeing Check-In survey link via Zoho Mail (same token-link
   pattern as M0).

## Response_Data blob format for the Wellbeing Check-In

Reuses M0's `---`-delimited blob pattern, decided 2026-09-06 (see `m1-plan.md`
Complexity Tracking). Same 5 prompts across M1, M3, and M4 so one format and
one set of Analytics formulas serve all three milestones — differentiated only
by the `Milestone` field on each record.

```
Personal wellbeing (0-10): {value}---Coping (0-10): {value}---Relationships and support (0-10): {value}---Hope and outlook (0-10): {value}---Sense of control (0-10): {value}
```

Each `{value}` is the raw 0-10 slider answer (integer). The 0-50 total is
**not stored in the blob** — it's a derived value, computed by an Analytics
formula (`SUM` of the 5 parsed domain columns) rather than duplicated at
write-back time, consistent with keeping the write-back function as simple as
M0's.

**Planned Analytics formula columns** (to build during implementation, same
`substring_between` pattern M0 established — 4 of 5 domains have a following
`---` to bound them; "Sense of control" is last and needs the
`SUBSTR`/`INSTR`/`LENGTH` pattern M0 used for its unbounded final field):

- Domain: Wellbeing, Domain: Coping, Domain: Relationships, Domain: Hope,
  Domain: Control (5 columns, `substring_between`/`SUBSTR` per above)
- Wellbeing Check-In Total (`SUM` of the 5 domain columns above)

## Out of scope for M1 (carried from research.md)

- Clinical safety flag evaluation against the ratified Clinical Safety Flag
  Rules (spec.md, User Story 4 / FR-010–013: stalled, deteriorating, or
  low-domain wellbeing readings) — needs a second reading to exist; starts at
  M3.
- Dashboard panels — M1's reporting needs are limited to confirming the
  pipeline works (a submitted-responses view, similar to M0's), not the
  admin dashboard (User Story 3 / FR-007–009) or flag visualizations that
  depend on M3 existing and a dedicated dashboard milestone being planned.
