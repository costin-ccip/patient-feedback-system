# Phase 1 Data Model: M3 - Periodic Consolidated Check-In (Every 8th Session)

**Input**: `m3-plan.md`, `m3-research.md`

**Date**: 2026-09-17

## Milestone_Instances usage for M3

M3 reuses the existing `Milestone_Instances` module (from M0/M1/M2), patient-
based, not lead-based:

| Field | M2 usage | M3 usage |
|---|---|---|
| `Milestone` | `"2 - Early Alliance Check"` | `"3 - Periodic Consolidated"` (new picklist **value** on the existing field — see "Milestone picklist" below; not a schema change) |
| `Lead_Reference` | Not used | Not used |
| `Patient` (lookup to `Patients1`) | Populated | Populated — same rejoin pattern, Principle II |
| `Status`, `Token`, `Expiry_Date_Time`, `Submitted_Date_Time`, `Response_Data` | Same lifecycle | Same lifecycle, different `Response_Data` content (7 segments instead of 4) |

**No new CRM fields.** `Milestone_Instances` remains at its CRM custom-field
cap (confirmed by Costin, 2026-09-10, per `m2-data-model.md`) — this applies
to M3 exactly as it applied to M2, and rules out any dedicated field for
tracking which 8-session checkpoint a given instance corresponds to (see
"Recurring-checkpoint idempotency" below for how this is handled instead).

**Multiple `Milestone_Instances` rows per patient at the same `Milestone`
value, for the first time.** Every prior milestone's design assumed (and
several idempotency checks rely on) at most one `Milestone_Instances` row
per patient per `Milestone` value. M3 breaks that assumption deliberately:
a single patient in long-term treatment will accumulate one M3 row at
session 8, another at session 16, another at session 24, and so on. Any
future report, dashboard, or function that queries `Milestone_Instances`
by `Patient` + `Milestone` and assumes "at most one row" must be re-checked
against M3 specifically — flagged here so it isn't assumed silently. (The
unified FR-007 Admin Dashboard, still out of scope, will need to treat
"one row per patient per milestone" as an M0/M1/M2/M4/M5-only assumption,
not a pipeline-wide one, whenever it is eventually built.)

## Milestone picklist value

**Confirmed 2026-09-17 via `getFields` on `Milestone_Instances`** (Zoho CRM
MCP — allowed under the standing access constraint, since
`Milestone_Instances` is not a Leads/Patients module): `"3 - Periodic
Consolidated"` **already exists** as a picklist value on `Milestone`, no
action needed. Full picklist as of this check: `-None-`, `0 - No
Conversion`, `1 - Baseline Intake`, `2 - Early Alliance Check`, `3 -
Periodic Consolidated`, `4 - Discharge`, `5B - Discontinuation, Email
Fallback`, `5A - Discontinuation, Live Capture` — the `5A`/`5B` values match
spec.md's Assumptions note about dead, pre-existing live-capture
configuration for M5 (disregard, not part of this milestone). `0 - No
Conversion` is the existing M0 value (spec.md's Milestones table calls it
"Free consult, no conversion" — a wording difference between the spec's
prose and the CRM's actual picklist label, an M0 concern, not M3's).

## Recurring-checkpoint idempotency: `checkPeriodicCheckInDue`

New custom function, mirroring the *shape* of `checkAllianceCheckExists`
(`m2-implementation-notes.md` §3.1) — same "query `Milestone_Instances`
fresh, don't cache" discipline — but different logic, since a plain
existence check is wrong for a recurring milestone (`m3-research.md`
Decision 2).

**Revised 2026-09-17 (Costin)**: rather than open-ended `(existingCount +
1) * 8` arithmetic, the checkpoints are a short, explicit, hand-maintained
list — 8, 16, 24, 32, 40. Extending the check-in schedule later (e.g. a 6th
check-in at 48) is a one-line edit to `checkpoints` below, not a formula
change. A patient whose `existingCount` reaches the end of the list simply
stops getting M3 check-ins until the list is extended — see
`m3-research.md` Decision 2 for the flagged operational consequence.

```text
bool checkPeriodicCheckInDue(string patientId, int sessionCount)
{
	checkpoints = list();
	checkpoints.add(8);
	checkpoints.add(16);
	checkpoints.add(24);
	checkpoints.add(32);
	checkpoints.add(40);

	existingCount = 0;
	qmap = Map();
	allRecs = zoho.crm.getRecords("Milestone_Instances",1,200,qmap,"crm_connection");
	for each  r in allRecs
	{
		patientLookup = r.get("Patient");
		patientRecId = "";
		if(patientLookup != null)
		{
			patientRecId = patientLookup.get("id");
		}
		if(patientRecId == patientId && r.get("Milestone") == "3 - Periodic Consolidated")
		{
			existingCount = existingCount + 1;
		}
	}
	if(existingCount >= checkpoints.size())
	{
		return false;
	}
	nextThreshold = checkpoints.get(existingCount);
	return sessionCount >= nextThreshold;
}
```

Input: `patientId` (string), `sessionCount` (int, from
`${trigger.Session_Count}`). Return: `bool`. Connection name
`crm_connection`, reused verbatim from M0/M1/M2's pattern. **Illustrative
Deluge draft** — confirm exact `zoho.crm.getRecords` pagination behavior
(the `1,200` page-size literal copied from `checkAllianceCheckExists`
assumes fewer than 200 total `Milestone_Instances` rows exist; revisit if
that stops being true) and integer-comparison syntax against the live
Deluge editor when this is actually built, the same "verify, don't assume"
discipline `m2-implementation-notes.md` §3.2/§3.3 document for every prior
custom function.

**Trigger flow entity: "M3 - Periodic Check-In Trigger" (new)**

1. **Trigger** — Zoho CRM "Updated module entry", same connection/module as
   M2 (`Patients1`). Filter: `Session Count` `greater than or equal to` `8`
   (a cheap pre-filter only — the real eligibility test is
   `checkPeriodicCheckInDue`, not this filter; see `m3-research.md`
   Decision 1 for why a modulo condition can't live in the trigger filter
   itself).
2. **Custom Function** — `checkPeriodicCheckInDue(patientId, sessionCount)`,
   a genuinely new, independently-created function (per
   `m2-implementation-notes.md` §3.2's gotcha: **do not** clone M2's flow
   and edit its idempotency-function node in place — build this one fresh
   via Built-ins → Developer Tools → Custom Functions, the same
   corrected procedure §3.2 documents, to avoid silently rewriting M2's
   `checkAllianceCheckExists`).
3. **If else** — condition: `checkPeriodicCheckInDue_1` `is true`.
   - **True branch**: "Call a subflow" → `Subflow - Issue Feedback Token`
     (unchanged, shared subflow — no schema/parameter changes needed for
     M3), `milestone` parameter = `"3 - Periodic Consolidated"`,
     `survey_url` = the new Periodic Check-In form's permalink (confirmed
     at build time the same way M2's §1 documents — Share tab → Public →
     Form Permalink, not the signed-in builder URL), `ttl_days` = `7`
     (reuse M0/M1/M2's default unless Costin specifies otherwise for a
     recurring milestone specifically — **flag for Costin**: does a
     patient in month 9 of treatment need a longer window than a first-time
     M0/M2 respondent? No evidence either way; defaulting to the existing
     value rather than inventing a new one).
   - **False branch**: left empty — flow ends, no email, no new record,
     same idempotency short-circuit shape as every prior milestone.

## Response_Data blob format for the Periodic Check-In

7 `---`-delimited segments. **Field order is deliberately different from
the form's on-screen section order** — see `m3-research.md` Decision 4 for
the full rationale (preserves M2's already-shipped, unbounded-last-field
"Domain: Fit Of Approach" Analytics formula column instead of breaking it):

```text
Practice Experience: Scheduling/Communication (0-10): {value}---Practice Experience: Billing (0-10): {value}---Therapist Professionalism (0-10): {value}---Connection (0-10): {value}---Understanding (0-10): {value}---Shared direction (0-10): {value}---Fit of approach (0-10): {value}
```

The first 3 segments are new to M3. The last 4 segments are **byte-for-byte
identical label text** to M2's blob (`m2-data-model.md`'s "Response_Data
blob format" section) — same domain names, same `(0-10):` suffix, same
order among themselves — specifically so the existing "Domain: Connection",
"Domain: Understanding", "Domain: Shared Direction", and "Domain: Fit Of
Approach" Analytics formula columns parse M3 rows correctly with **zero
formula edits**, per `m3-research.md` Decision 4.

**Write-back function**: `submitPeriodicCheckInResponse` — same pure
string-concatenation shape as `submitAllianceCheckInResponse`
(`m2-implementation-notes.md` §3.4.1), extended to 7 input parameters
instead of 4 (`token`, `connection`, `scheduling`, `billing`,
`professionalism`, `understanding`, `sharedDirection`, `fitOfApproach` — 8
total including `token`). No arithmetic, no conditional flag logic, per
Constitution Principle VII — identical token-lookup /
`Status != "Issued"` rejection / expiry-check-and-auto-expire structure as
every prior write-back function.

## Analytics formula columns (Milestone Instances table)

**Unchanged, reused as-is** (per Decision 4's field-order choice):
- Domain: Connection, Domain: Understanding, Domain: Shared Direction
  (bounded `substring_between`)
- Domain: Fit Of Approach (unbounded last-field pattern — stays correct
  for M3 rows only because Decision 4 keeps it last in the blob)
- Alliance Check-In Total (`SUM`/`to_integer()` of the 4 Domain columns)

**New, M3-only** (bounded `substring_between`, since each is followed by
another segment in the chosen order):
- **Practice Experience: Scheduling/Communication** —
  `substring_between("Milestone Instances"."Response Data", 'Practice Experience: Scheduling/Communication (0-10): ', '---', 1)`
- **Practice Experience: Billing** —
  `substring_between("Milestone Instances"."Response Data", 'Practice Experience: Billing (0-10): ', '---', 1)`
- **Therapist Professionalism** —
  `substring_between("Milestone Instances"."Response Data", 'Therapist Professionalism (0-10): ', '---', 1)`

**Modified, shared with M2** (`m3-research.md` Decision 5 — requires an
`m2-implementation-notes.md` update in the same build session):
- **Clinical Safety Flag** — widen the `Milestone` gate:
  ```
  IF(OR("Milestone Instances"."Milestone" = '2 - Early Alliance Check', "Milestone Instances"."Milestone" = '3 - Periodic Consolidated'), IF("Milestone Instances"."Alliance Check-In Total" <= 20 OR to_integer("Milestone Instances"."Domain: Connection") <= 4 OR to_integer("Milestone Instances"."Domain: Understanding") <= 4 OR to_integer("Milestone Instances"."Domain: Shared Direction") <= 4 OR to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, 'true', 'false'), '')
  ```
- **Flag Rule Triggered** — same nested `IF`/`concat` body as M2's version
  (`m2-implementation-notes.md` §9.2), unchanged except that it now
  evaluates for M3 rows too, since it already reads `Clinical Safety Flag`
  rather than re-testing `Milestone` itself:
  ```
  IF("Milestone Instances"."Clinical Safety Flag" = 'true', concat(
    IF("Milestone Instances"."Alliance Check-In Total" <= 20, concat('Total alliance score ', to_string("Milestone Instances"."Alliance Check-In Total"), '/40 (<=20); '), ''),
    IF(to_integer("Milestone Instances"."Domain: Connection") <= 4, concat('Connection domain ', "Milestone Instances"."Domain: Connection", '/10 (<=4); '), ''),
    IF(to_integer("Milestone Instances"."Domain: Understanding") <= 4, concat('Understanding domain ', "Milestone Instances"."Domain: Understanding", '/10 (<=4); '), ''),
    IF(to_integer("Milestone Instances"."Domain: Shared Direction") <= 4, concat('Shared direction domain ', "Milestone Instances"."Domain: Shared Direction", '/10 (<=4); '), ''),
    IF(to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, concat('Fit of approach domain ', "Milestone Instances"."Domain: Fit Of Approach", '/10 (<=4); '), '')
  ), '')
  ```

Exact Analytics formula syntax (function names, `IN`/`OR` availability,
whether `Edit Formula Column` accepts the widened condition as drafted) is
to be verified against the live workspace at build time, per every prior
milestone's own "confirm against the actual UI, don't assume" discipline —
this document records the *intended* logic, not a guarantee it will paste
in verbatim.

## Reporting (User Story 6 / FR-018 baseline bar; FR-013 flag visibility)

- **"M3 Submitted Responses"** — new Tabular View, same shape as M2's
  (`m2-implementation-notes.md` §9.3): Submitted Date Time, Token, all 7
  parsed columns (3 new + 4 shared), Alliance Check-In Total, Clinical
  Safety Flag, Flag Rule Triggered. No `Patient`/identity column. Filtered
  to `Milestone` Wildcard Exactly Matches `"3 - Periodic Consolidated"`.
- **"M3 Status Breakdown"** — status/volume view, "Save As" off M2's
  equivalent (or the shared `[RETIRED] M1 Status Breakdown` M2 itself
  copied from), Milestone filter swapped to `"3 - Periodic Consolidated"`.
- **"M3 Practice Experience & Professionalism Distribution"** —
  distribution/summary view satisfying FR-018/User Story 6's "at least one
  distribution or summary view of that milestone's own scored ... data"
  requirement for M3's *new* content specifically (not just the reused
  alliance domains, which M2's equivalent view already covers) — bar
  chart(s) over the 3 new formula columns, `Dimension [Actual(D)] / Treat
  as Text` mode per `m2-implementation-notes.md` §9.5's gotcha (a numeric
  column defaults to `Measure [Actual(M)]`, which aggregates instead of
  showing one bar per distinct value).
- **"M2 & M3 Flagged for Review"** — the existing M2 report
  (`m2-implementation-notes.md` §9.6), **renamed and its Milestone filter
  widened** to `Wildcard` `"2 - Early Alliance Check"` OR `"3 - Periodic
  Consolidated"`, rather than a second, parallel M3-only flagged view — same
  reasoning as the Clinical Safety Flag column widening above (one rule,
  one report). **This is a rename + filter change on an M2-owned view;
  update `m2-implementation-notes.md`'s §9.6 reference in the same session.**
- **"M3 - Periodic Check-In Feedback" dashboard** — new dashboard bundling
  the four views above, built via "Create New Dashboards" (not "Save As"),
  same shape as M0/M1/M2's own dashboards.

## Out of scope for M3

Same list as `m3-research.md`'s "Out of scope for M3" section — not
repeated here; see that document.
