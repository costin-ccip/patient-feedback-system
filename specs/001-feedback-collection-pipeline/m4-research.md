# Phase 0 Research: M4 - Discharge

**Input**: `spec.md` (Milestones table row 4; Clinical Safety Flag Rules;
User Story 6 / FR-018), `constitution.md` (v1.3.0), `CLAUDE.md`'s token-issuance
and legacy-patient cutoff-exclusion conventions, `m2-plan.md` / `m2-research.md`
/ `m2-data-model.md` / `m2-implementation-notes.md`, `m3-plan.md` /
`m3-research.md` / `m3-data-model.md` / `m3-implementation-notes.md`,
`specs/002-remove-subflow-dependency/*`, `specs/003-issuance-privacy-and-failure-handling/*`,
`specs/004-legacy-patient-exclusion/*`

**Date**: 2026-09-23

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement,
written before any Zoho implementation begins for M4.

## Why M4 is closer to M2 than to M3

M3 introduced a genuinely new trigger/idempotency shape (recurring,
checkpoint-based) because "every 8th session, for the duration of treatment"
can fire many times per patient. M4 (Discharge) does not have that problem:
spec.md's Milestones table defines its trigger as **"Discharge is marked in
the system of record"** — a condition that becomes true exactly once in a
patient's lifecycle, the same shape as M0's "free consult, no conversion" and
M2's "session count reaches 3". `m3-plan.md`'s own "Precedent for M4/M5" note
(written prospectively during M3's build) already calls this: M4 and M5 are
both one-time-per-patient conditions like M0/M2, so neither needs M3's
recurring-checkpoint machinery, and the `checkXExists`-style plain existence
check remains the right default. This research file starts from that
precedent rather than re-deriving it.

What M4 *does* need that M0/M2/M3 didn't have to resolve from scratch at
build time: (1) mapping spec.md's abstract trigger condition to a real CRM
field/value, since — unlike M0's "Lost Lead" or M2/M3's `Session_Count`
threshold — spec.md never names the exact field; and (2) deciding, as its own
plan.md must per the legacy-patient cutoff-exclusion convention
(`specs/004-legacy-patient-exclusion/spec.md`'s Assumptions: "Whether M4/M5...
will need the same gate is not decided here"), whether M4 needs that same
gate.

## Decision 1: Mapping "Discharge is marked in the system of record" to a real CRM field

**Confirmed live via Zoho CRM MCP `getFields` on `Patients1`** (allowed under
the standing access constraint — this is field/picklist metadata, not a PII
record read): the module has a `Patient_Status` picklist field (label
"Patient Status") with these values: `-None-`, `Active`, `Completed
Treatment`, `Discontinued (Patient Choice)`, `Referred Out`, `No Show`.

There is no field or value literally spelled "Discharge" or "Discharged".
**`Completed Treatment` is the value this plan uses to mean "discharge"** —
it's the only `Patient_Status` value describing a patient finishing care in
good standing, which is what spec.md's Milestone 4 (an alliance reading "final
reading" plus a "looking ahead" exit section, distinct from M5's
discontinuation/attrition path) describes. The other non-`Active` values map
to M5 territory instead:

- `Discontinued (Patient Choice)` — a patient-initiated stop, which is
  attrition, not a completed course of care. This is conceptually M5
  territory (spec.md's Milestone 5, cancellation/no-show pattern), though M5
  itself is out of scope for this milestone and not being built here.
- `Referred Out` — the practice referring the patient elsewhere, not a
  completion of the patient's own treatment goals here.
- `No Show` — an attendance-pattern value, not a lifecycle-completion value;
  also M5-adjacent (spec.md's M5 trigger explicitly is a no-show pattern).

This is the same class of decision `m0-implementation-notes.md` documents for
mapping M0's "free consult with no conversion" to the CRM's actual
`Lead_Status = Lost Lead` value — an inference from the real picklist, not a
value named verbatim in spec.md's prose, confirmed against the live schema
rather than assumed. **Flag for Costin/Liana**: confirm `Completed Treatment`
is in fact what practice staff select when a patient is discharged, the same
way M0/M1/M2/M3's drafted-not-sourced survey copy has been flagged for review
before going live — this is a load-bearing trigger-condition mapping, not
wording, so it's worth an explicit yes/no rather than silent adoption.

**Confirmed 2026-09-24 by Costin**: `Patient_Status = "Completed Treatment"`
is the correct discharge-trigger filter value. No change needed to the
already-built "M4 - Discharge Trigger" flow (`m4-implementation-notes.md`
§2) — it was built against this same value as its best inference.

**Rejected alternative**: waiting for Costin to specify the field before
writing plan.md/data-model.md. Every other milestone's planning has proceeded
on a documented, live-schema-confirmed best inference and flagged it for
review rather than blocking the whole milestone on a question that doesn't
prevent building anything (the trigger's filter value is a one-line change if
the answer turns out to be different) — consistent with this session running
unattended-tolerant per its operating mode.

## Decision 2: Idempotency — plain existence check, not a recurring one

Per `m3-plan.md`'s precedent (quoted above) and per Decision 1 establishing
M4 as a one-time-per-patient condition: reuse the `checkXExists` shape
(`m2-implementation-notes.md` §3.1's `checkAllianceCheckExists`, field-for-
field identical except the match string), named `checkDischargeExists`:

```text
bool checkDischargeExists(string patientId)
{
	found = false;
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
		if(patientRecId == patientId && r.get("Milestone") == "4 - Discharge")
		{
			found = true;
		}
	}
	return found;
}
```

Built genuinely fresh via Built-ins → Developer Tools → Custom Functions (not
cloned from any existing flow's node), per `m2-implementation-notes.md` §3.2's
shared-custom-function-across-clones gotcha, which applies here exactly as it
did to every prior milestone's trigger function.

**Rejected alternative**: reusing M3's checkpoint-list machinery
(`checkPeriodicCheckInDue`) generalized to a 1-item list. Rejected as
needless complexity for a condition that is not recurring — a plain boolean
existence check is simpler, more auditable, and matches M0/M2's already-
proven pattern exactly.

## Decision 3: Legacy-patient cutoff-exclusion gate — yes, M4 needs it

`specs/004-legacy-patient-exclusion/spec.md` built the shared
`isCreatedAfterCutoff(string createdTime) -> bool` function for M2/M3 and
explicitly left M4/M5's need for it undecided, to be resolved by each
milestone's own plan.md rather than assumed either way.

**M4 needs it.** The reasoning that justified the gate for M2/M3 is not about
recurrence — it's about how a Zoho Flow "Updated module entry" trigger
behaves: the trigger fires on *any* update to a `Patients1` record while the
flow is ON, and then evaluates its filter criteria against the record's
*current* field values, not against what changed in that specific update.
M4's own trigger (Decision 5 below) is exactly this same trigger type,
filtered on `Patient_Status equals "Completed Treatment"`. A patient who
completed treatment and was marked `Completed Treatment` months before this
flow is ever switched on would still cause the trigger to fire and the filter
to evaluate true the next time *any* field on their record is touched (a
phone number correction, an owner reassignment, anything) — exactly the
"surprise feedback request to a long-closed case" problem
`specs/004-legacy-patient-exclusion/spec.md`'s User Story 1 describes for
M2/M3's `Session_Count` filter. This is not a recurrence-specific risk; it's
inherent to any CRM-level-state filter on an "Updated module entry" trigger,
which M4's trigger is. M0 is the one milestone that's genuinely exempt, per
CLAUDE.md's existing note, because it triggers on `Leads` (a different
module, and Costin's explicit call) — that reasoning doesn't extend to M4,
which is on `Patients1` exactly like M2/M3.

Reuse the same function object (per feature 002/004's established reuse
pattern, and per CLAUDE.md's explicit instruction to reuse rather than
recreate): wire `isCreatedAfterCutoff` between `checkDischargeExists` and the
`If else`, `createdTime` mapped to `${trigger.Created_Time}`, output variable
renamed `isCreatedAfterCutoff_1` (a separate instance from M2's/M3's, since
Output Variable Name is scoped per flow even though the function body is
shared — same as `m3-implementation-notes.md` §7 documents). The `If else`
condition becomes `checkDischargeExists_1 is false` **AND**
`isCreatedAfterCutoff_1 is true` (`checkDischargeExists` returning `false`
means "not yet issued", so the gate is on its negation — mirroring how M0/M2's
own `checkXExists` functions are wired: their `If else` fires on `... is
false`).

**Rejected alternative**: skip the gate for M4 on the theory that "discharge"
is a deliberate, rare staff action less likely to be touched incidentally
post-completion than an ever-incrementing `Session_Count`. Rejected because
the underlying trigger mechanism (any update to the record re-evaluates the
filter against current state) is identical regardless of how often the field
is expected to change afterward — the risk is categorical, not a matter of
degree, and the fix is free (the function already exists and is designed for
reuse).

## Decision 4: Survey structure — reuse M2/M3's alliance domains, add a new "Looking ahead" section

Per spec.md's Milestones table (row 4, Notes column): "An alliance reading
(final reading) plus a 'looking ahead' exit section; no wellbeing reading, so
this milestone no longer closes any pre/post wellbeing comparison." Per the
fifth-pass spec revision note: Milestone 3's "likelihood to continue/refer"
question was removed because it's "already captured at Milestone 4" — meaning
M4 is expected to carry a likelihood-to-refer/recommend question in its
Looking Ahead section. No exact wording for this section exists anywhere in
this repo (the source Confluence design pages aren't available to this
session) — same situation M1/M2/M3 were in for their own new content, each
flagged as drafted-not-sourced pending Costin/Liana review.

**Structure** (2 new fields, drafted this session, **not yet reviewed by
Costin/Liana** — same caveat as every prior milestone's own new copy):

1. **Slider — Likelihood to recommend** (0-10): "How likely are you to
   recommend Cape Clarity to someone in a similar situation?" / "0 = Not at
   all likely, 10 = Extremely likely." Mandatory. This is the question
   `m3-research.md` (Decision 3) and spec.md's fifth-pass note both point to
   as already-scoped-for-M4 content, so it's the one piece of M4's Looking
   Ahead section with a documented reason to exist, even though its exact
   wording is still new.
2. **Multi Line — Anything else looking ahead**: "Is there anything else
   you'd like to share as you finish up your care with us?" Optional
   (freeform text is never a Clinical Safety Flag input and doesn't need to
   be mandatory to satisfy any requirement here — mirrors M0's optional
   "Anything Else" field).

Plus the 4 alliance domain sliders, reused **verbatim** from M2/M3 (same
prompt/instructions/range text, byte-for-byte) — Connection, Understanding,
Shared direction, Fit of approach — since spec.md describes M4's alliance
reading as "final" in content, not different in instrument. On-screen section
order: Alliance Check-In (1, reused sliders), Looking Ahead (2, the two new
fields) — the reverse of M3's order (Alliance last) because M4 has only two
sections and spec.md's own prose order for M4 is "an alliance reading...plus
a looking ahead...section", alliance first.

Plus the hidden `Token` single-line field, same prefill-URL pattern as every
prior form.

**Rejected alternative**: a wellbeing pre/post close-out question, since M4
is the natural point in a patient's journey to "close the loop" on any
earlier wellbeing reading. Rejected per spec.md's own explicit text: no
milestone collects a wellbeing reading anymore (Milestone 1 retired,
Milestones 3/4 both had their wellbeing sections removed in the fifth-pass
revision), so there is nothing left to close out — reintroducing one here
would resurrect content the spec deliberately eliminated.

## Decision 5: Response_Data blob field order

6 segments. Same reasoning as `m3-research.md` Decision 4 (M3): put the new
content first, keep the 4 reused alliance domains last in the exact same
relative order M2/M3 use, so the existing "Domain: Connection", "Domain:
Understanding", "Domain: Shared Direction" (bounded `substring_between`) and
"Domain: Fit Of Approach" (the unbounded-last-field pattern) Analytics
formula columns parse M4 rows with **zero formula edits**, exactly as they
already do for M3 rows:

```text
Looking Ahead: Likelihood To Recommend (0-10): {value}---Looking Ahead: Anything Else: {value}---Connection (0-10): {value}---Understanding (0-10): {value}---Shared direction (0-10): {value}---Fit of approach (0-10): {value}
```

The freeform "Looking Ahead: Anything Else" segment sits mid-blob (not last),
bounded by `substring_between` like every other non-terminal segment — the
same pattern `m0-implementation-notes.md` §4/§6 already establishes for its
own two freeform fields ("Additional Comments", "Anything Else"), one of
which is mid-blob and bounded exactly this way. A freeform answer containing
the literal delimiter text (`---`) would corrupt parsing from that point
forward; this is an accepted, undocumented-until-now edge case already
implicit in M0's design and not something this milestone introduces or needs
to newly solve.

**Write-back function**: `submitDischargeFeedbackResponse` — same pure
string-concatenation shape as `submitAllianceCheckInResponse`/
`submitPeriodicCheckInResponse`, 7 input parameters (`token`,
`likelihoodToRecommend`, `anythingElse`, `connection`, `understanding`,
`sharedDirection`, `fitOfApproach`), identical token-lookup /
`Status != "Issued"` rejection / expiry-check-and-auto-expire structure as
every prior write-back function. No arithmetic, no conditional flag logic,
per Constitution Principle VII.

## Decision 6: Clinical Safety Flag — extend the existing M2/M3 columns again, don't fork

spec.md's Clinical Safety Flag Rules section is explicit: "These rules apply
to alliance readings (Milestones 2, 3, 4)". M4 is already named in the rule's
own scope, so this isn't a judgment call the way M3's Decision 5 was framed
(there, spec.md defined the rule as spanning 2-4 and M3's plan had to decide
*how* to implement that span; here, M4 is simply the last milestone that
span already accounted for). Widen the existing "Clinical Safety Flag" and
"Flag Rule Triggered" formula columns' `Milestone` gate a second time, from
`'2 - Early Alliance Check' OR '3 - Periodic Consolidated'` to also include
`'4 - Discharge'`, rather than forking a third parallel pair of columns —
same rationale `m3-research.md` Decision 5 and Constitution Principle VII
give: one rule, one editable formula column, no drift risk between three
copies.

Live formula (current, per `m3-implementation-notes.md` §1.1), to be widened:

```
IF("Milestone Instances"."Milestone" = '2 - Early Alliance Check' OR "Milestone Instances"."Milestone" = '3 - Periodic Consolidated', IF("Milestone Instances"."Alliance Check-In Total" <= 20 OR to_integer("Milestone Instances"."Domain: Connection") <= 4 OR to_integer("Milestone Instances"."Domain: Understanding") <= 4 OR to_integer("Milestone Instances"."Domain: Shared Direction") <= 4 OR to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, 'true', 'false'), '')
```

New gate clause adds a third `OR`'d `Milestone = '4 - Discharge'` condition
(infix `OR`, per `m3-implementation-notes.md` §1.1's confirmed Analytics-
formula-language finding — `OR(a,b)` as a function call fails to parse in
this workspace). Full drafted formula is in `m4-data-model.md`.

The existing "M2 & M3 Flagged for Review" report gets a third Milestone
Wildcard condition and is renamed "M2, M3 & M4 Flagged for Review" — same
"one rule, one report" reasoning as its own prior rename. **Both changes
require updating `m2-implementation-notes.md` §9.2/§9.6 and
`m3-implementation-notes.md` §1.1/§1.3 in the same session**, per CLAUDE.md's
convention, since this milestone's build changes two earlier milestones'
already-documented, shared Analytics objects for a second time.

## Decision 7: `Milestone` picklist value

**Confirmed live 2026-09-23 via Zoho CRM MCP `getFields` on
`Milestone_Instances`** (allowed under the standing access constraint — not a
Leads/Patients module): `"4 - Discharge"` **already exists** as a picklist
value on `Milestone`. No CRM schema change needed. Full picklist as of this
check: `-None-`, `0 - No Conversion`, `1 - Baseline Intake`, `2 - Early
Alliance Check`, `3 - Periodic Consolidated`, `4 - Discharge`, `5B -
Discontinuation, Email Fallback`, `5A - Discontinuation, Live Capture` —
unchanged from what `m3-data-model.md` recorded, and the `5A`/`5B` dead
live-capture values remain disregarded per spec.md's Assumptions.

Also confirmed live: `Milestone_Instances.Status` picklist now includes
`Send Failed` (added by Costin between features 003 and this session, per
`specs/003-issuance-privacy-and-failure-handling/implementation-notes.md`
§3's outstanding TODO) — so M4's On Error → "Update module entry" branch
(CLAUDE.md's token-issuance convention, point 3) can set `Status = "Send
Failed"` against a real, already-existing picklist value with no further CRM
change needed.

## Decision 8: On Error branches — build them from the start, not retrofitted

M0/M2/M3 each got their On Error branches (alert-on-issuance-failure,
mark-and-alert-on-send-failure) retrofitted after the fact by feature 003,
because 003 postdates their original trigger-flow builds. M4 is the first
milestone built *after* the token-issuance convention (feature 002) and the
On Error convention (feature 003) both already exist — build both branches
as part of the initial trigger flow, per CLAUDE.md's "M4/M5 must follow this"
instruction, rather than building a bare happy path and circling back.

## Reporting (User Story 6 / FR-018 baseline bar)

Same "at minimum: a status/volume view and one distribution/summary view of
this milestone's own scored data" bar every prior milestone has cleared
(`m1-implementation-notes.md` §13, `m2-implementation-notes.md` §9,
`m3-implementation-notes.md` §1.7/§1.8). M4's own new scored data is exactly
one numeric field (Looking Ahead: Likelihood To Recommend — the freeform
"Anything Else" field is not scored/categorical and doesn't need a
distribution panel, same treatment M0's own freeform fields got). Full report
list is in `m4-data-model.md`.

## Out of scope for M4

- Milestone 5 (Discontinuation) — separate milestone, not built here.
- The unified, cross-milestone Admin Dashboard (User Story 3 / FR-007–009) —
  explicitly deferred pipeline-wide, unaffected by this milestone.
- Raw Zoho Forms submission purge (FR-006) — the same open, tracked,
  cross-milestone gap `data-retention-purge.md` documents; M4 adds a fifth
  flow with the identical gap, not resolved here.
- ~~Resolving whether `Completed Treatment` is in fact the correct
  discharge-marking value (Decision 1)~~ — **confirmed 2026-09-24 by Costin**,
  no longer open.
- Whether M5 also needs the legacy cutoff-exclusion gate — M5's own future
  planning should make that call the same way this file makes it for M4,
  rather than assuming either answer. (As of 2026-09-24, M2/M3/M4 all use the
  gate; M0 is the only active flow that doesn't, by Costin's own earlier,
  explicit decision — see `CLAUDE.md`.)
