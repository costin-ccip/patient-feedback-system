# Phase 0 Research: M5 - Discontinuation

**Input**: `spec.md` (Milestones table row 5; User Story 5; User Story 6 /
FR-018), `constitution.md` (v1.3.0), `CLAUDE.md`'s token-issuance and
legacy-patient cutoff-exclusion conventions, `m2-plan.md`/`m2-research.md`/
`m2-data-model.md`/`m2-implementation-notes.md`, `m3-plan.md`/`m3-research.md`/
`m3-data-model.md`/`m3-implementation-notes.md`, `m4-plan.md`/`m4-research.md`/
`m4-data-model.md`/`m4-implementation-notes.md`, `specs/002-remove-subflow-dependency/*`,
`specs/003-issuance-privacy-and-failure-handling/*`,
`specs/004-legacy-patient-exclusion/*`, Confluence "Customer Feedback
Collection" (page 563347491) and "Milestone 5 — Discontinuation
(Unplanned/Lapsed)" (page 564035589)

**Date**: 2026-09-24

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement,
written before any Zoho implementation begins for M5.

## Why M5 is not just "M4 again"

`m3-plan.md`'s prospective "Precedent for M4/M5" note and `m4-plan.md`'s own
closing note both already flagged this: M4's trigger condition ("discharge is
marked in the system of record") mapped cleanly onto a single `Patient_Status`
value transition, confirmed and built with no new logic. M5's trigger, per
spec.md's Milestones table, is a **pattern with two independent legs**:
"Cancellation with no rebooking within 14 days, OR 2 consecutive no-shows with
no reschedule in between." Unlike M4, no CRM field literally encodes "14 days
since cancellation" or "consecutive no-show count" — those are date-math/
counting rules over appointment-level activity, and this session found no
Appointments/Sessions-level CRM module to compute them from (see Decision 1).

Unlike M0–M4's own new survey copy (each flagged "drafted-not-sourced,
pending Costin/Liana review"), M5's Confluence source page (564035589,
"Milestone 5 — Discontinuation (Unplanned/Lapsed)") is directly available to
this session and contains fully worked content — trigger definitions, email
subject/body, 8 reason categories, both survey questions, and the
post-submission closing line — used verbatim below, not drafted. This is the
first milestone build where the survey copy itself needs no flagging.

## Decision 1: Mapping the two-leg trigger pattern to real CRM state

**Confirmed live via Zoho CRM MCP `getFields` on `Patients1`** (metadata only,
same allowed access as every prior milestone's schema check): the module's
`Patient_Status` picklist is `-None-`, `Active`, `Completed Treatment`,
`Discontinued (Patient Choice)`, `Referred Out`, `No Show`. No Appointments,
Sessions, Visits, or Bookings-type module with per-appointment
cancellation/no-show records was found in the org's module list
(`CalendarBookings__s` exists but this session did not open or query it —
querying it would risk surfacing identified appointment/attendee data, which
is out of scope for a planning-phase schema check the same way this session
avoids `Patients1`/`Leads` record-level reads).

Two designs were considered:

**Option A (chosen) — reuse `Patient_Status`, mirroring M0/M4's precedent.**
`Discontinued (Patient Choice)` and `No Show` are the two `Patient_Status`
values that are not `Active`, `Completed Treatment`, or `Referred Out` — i.e.
exactly the two values M4's own Decision 1 already flagged as "M5 territory"
without building on them. They line up one-to-one with spec.md's two trigger
legs:
- `Discontinued (Patient Choice)` ↔ "Cancellation with no rebooking within 14
  days" (a patient choosing to stop, which is how staff would record a
  cancellation that was never rebooked once the 14-day window judged it
  final).
- `No Show` ↔ "2 consecutive no-shows with no reschedule in between" (staff
  marking the attendance pattern once it's happened, the same way `Completed
  Treatment` is a staff judgment call marked after discharge, not a
  system-computed date).

Under this design, M5's Flow trigger watches `Patients1` for `Patient_Status`
becoming `Discontinued (Patient Choice)` **OR** `No Show` — the underlying
14-day/2-consecutive-no-show counting is a practice/staff judgment applied
before setting the status, exactly the same division of labor M4 established
for "discharge is marked" (Constitution Principle III requires the *feedback
request* to be system-triggered without contractor/staff action, not that the
practice's own status-marking be computed rather than judged — M0's `Lost
Lead` and M4's `Completed Treatment` are both staff judgment calls recorded as
a status value, and this pipeline's automation begins at that recorded value,
not before it).

**Option B (rejected) — compute the pattern from appointment-level data.**
Would require either a real Appointments/Sessions/Bookings CRM module (not
confirmed to exist in an accessible, non-PII-querying way — see above) or a
scheduled/polling Flow that scans `CalendarBookings__s` or an EHR export for
cancellation/no-show sequences and date math. Rejected for now: no such data
source was confirmed available to this pipeline (spec.md's own Assumptions
already establish all six milestones trigger off Zoho CRM, not the EHR, and
this session found no CRM module built for this purpose), and building one
would be new infrastructure well beyond a single milestone's scope — a much
larger undertaking than any milestone-specific decision this repo has made so
far. If a future session confirms a real appointment-level data source exists
and staff want the pattern computed automatically instead of staff-judged,
that is a follow-on feature, not a change to fold into M5's initial build.

**Flag for Costin/Liana** (same treatment as M4's Decision 1, which was
confirmed after the fact): confirm that `Discontinued (Patient Choice)` is
in fact what staff select for a cancellation-with-no-rebooking case, and
`No Show` for the 2-consecutive-no-show case — this is the single
highest-risk assumption in this milestone, more so than M4's, because it
maps *two* business conditions onto *two* CRM values that this session
inferred by elimination rather than by any explicit prior confirmation (M4's
`Completed Treatment` was at least the only plausible completion-shaped
value; M5's mapping is a considered inference across two values with no
single obviously-correct alternative). Building proceeds on this
best-supported inference and the flow is left OFF pending confirmation,
consistent with every prior milestone's practice of not blocking the whole
build on a question that doesn't prevent building anything.

## Decision 2: Trigger-level OR across two Patient_Status values — new builder territory

Every prior one-time-trigger milestone (M0, M2, M4) filtered its trigger on a
**single** `equals` condition. M5's Decision 1 needs an OR of two values on
the same field. This repo has not yet confirmed whether Zoho Flow's "Updated
module entry" trigger criteria builder supports a same-field OR group (Zoho
CRM's own list-view/workflow criteria builder generally supports "match ANY
of the following," but this specific Flow trigger UI has not been checked in
any prior milestone build, since none needed it).

**Planned approach, to be confirmed against the live builder during
implementation** (mirroring `m4-tasks.md` T004's "confirm exact syntax
against the live Deluge editor" style of flagging a builder detail to verify
rather than assume):
1. **Preferred**: if the trigger criteria builder supports an OR group on the
   same field, set the trigger-level filter directly to `Patient_Status
   equals "Discontinued (Patient Choice)" OR Patient_Status equals "No
   Show"` — narrowest possible trigger, consistent with every prior
   milestone's practice of filtering at the trigger level, not after it.
2. **Fallback**, if the builder only supports AND/a single condition: widen
   the trigger-level filter to `Patient_Status is not empty` (or drop the
   trigger-level filter entirely) and add the OR check as the first condition
   in the flow's own `If else` (`${trigger.Patient_Status} == "Discontinued
   (Patient Choice)" OR ${trigger.Patient_Status} == "No Show"`), ANDed with
   `checkDiscontinuationExists_1 is false` and `isCreatedAfterCutoff_1 is
   true` exactly as those two are already ANDed in M2/M3/M4's `If else`
   nodes. This changes where the check lives, not the underlying logic, so it
   does not affect Decision 1's mapping either way.

## Decision 3: Idempotency — plain existence check, same shape as M0/M2/M4

Per `m3-plan.md`'s "Precedent for M4/M5" note (M4 and M5 are both
one-time-per-patient conditions, not recurring like M3): `checkXExists`
shape, named `checkDiscontinuationExists`, field-for-field identical to
`checkDischargeExists` (`m4-implementation-notes.md` §2) except the match
string:

```text
bool checkDiscontinuationExists(string patientId)
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
		if(patientRecId == patientId && r.get("Milestone") == "5B - Discontinuation, Email Fallback")
		{
			found = true;
		}
	}
	return found;
}
```

Built genuinely fresh via Built-ins → Developer Tools → Custom Functions (not
cloned from any existing flow's node), per `m2-implementation-notes.md` §3.2's
shared-custom-function-across-clones gotcha, exactly as every prior
milestone's own idempotency function was built.

**Rejected alternative**: a single function covering both `Discontinued
(Patient Choice)` and `No Show` as separate, trackable milestone *instances*
(i.e. letting a patient who is later re-marked from one status to the other
generate a second feedback request). Rejected: spec.md's Edge Cases section
does not describe M5 as having two independently-triggerable sub-instances,
and the CRM's own `5A`/`5B` picklist history (a dead live-capture-vs-email
split, not a cancellation-vs-no-show split — see Decision 6) confirms M5 has
always been modeled as one milestone, one instance per patient, regardless of
which of the two conditions caused it. `checkDiscontinuationExists` therefore
checks only for *any* existing `"5B..."` instance for the patient, not one
scoped to which status value fired it.

## Decision 4: Legacy-patient cutoff-exclusion gate — yes, M5 needs it

Same reasoning as `m4-research.md` Decision 3, which itself generalizes from
M2/M3: the risk is inherent to any "Updated module entry" trigger evaluating
a CRM-level-state filter against a `Patients1` record's *current* field
values on *any* update to that record, not specific to how the milestone's
condition is worded or how often the watched field is expected to change. A
patient already marked `Discontinued (Patient Choice)` or `No Show` long
before this flow exists would otherwise get a surprise feedback request the
next time any unrelated field on their record is touched. M5's trigger is on
`Patients1`, exactly like M2/M3/M4 (not `Leads`, M0's genuine exception) — reuse
the same shared `isCreatedAfterCutoff(string createdTime) -> bool` function
(feature 004, unmodified), wired between `checkDiscontinuationExists` and the
`If else` (or, under Decision 2's fallback path, ANDed alongside the
Patient_Status OR-check inside the same `If else`), `createdTime` =
`${trigger.Created_Time}`, output variable `isCreatedAfterCutoff_1` (a
separate per-flow instance, same as every prior consumer).

**Rejected alternative**: skip the gate on the theory that a
patient-discontinuation status is even rarer/more deliberate than a discharge
status. Rejected for the same categorical reason `m4-research.md` Decision 3
already rejected it for M4: the trigger mechanism's behavior doesn't depend
on how rare the watched value is expected to be.

## Decision 5: Survey structure and Response_Data blob — verbatim from Confluence, no scored/numeric content

Per Confluence page 564035589, verbatim (not drafted):

**Form fields** (2 substantive + hidden token — the shortest form of any
milestone so far, and the first with zero sliders/numeric domains):

1. **Single-select (radio or dropdown — decide against the live Forms
   builder which control type Zoho Forms offers for a single-select list of
   8 options; a Dropdown is likely the cleaner fit for 8 options than 8
   radio buttons, but this is a builder-ergonomics choice, not a content
   one)** — "What led to stepping away from sessions right now?" Options,
   verbatim and in the source's own order:
   - Felt better / reached their goals
   - Scheduling or timing conflict
   - Cost or insurance
   - Didn't feel like the right fit with their therapist
   - Life circumstances changed
   - Moved or relocated
   - Choosing to pause for now
   - Something else
   Mandatory.
2. **Radio (3 options)** — "Would it be okay to reach back out if things
   change?" Options: Yes / No / Maybe. Mandatory.
3. **Hidden Token** single-line field, same prefill-URL pattern as every
   prior form.

No name/email/phone field, per Constitution Principle I. No open-text field
at all — the source doc's own "Design notes" section states this explicitly
("No open text in the email itself... structured categories only, consistent
with the no-open-text decision on Milestones 1–4"), a stricter stance than
M0's/M4's own optional freeform fields, not a gap to fill in.

**Response_Data blob** (2 segments — the shortest of any milestone; last
segment unbounded per the established pattern):

```text
Reason For Leaving: {value}---Okay To Reach Back Out: {value}
```

**Write-back function**: `submitDiscontinuationFeedbackResponse`, 3 string
input parameters (`token`, `reasonForLeaving`, `okayToReachBackOut`), same
token-lookup / `Status != "Issued"` rejection / expiry-check-and-auto-expire
structure as every prior write-back function, pure string concatenation, no
arithmetic or conditional logic, per Constitution Principle VII.

**Email** (per Confluence, verbatim — the first milestone whose delivery
copy is sourced rather than drafted):

| Field | Content |
|---|---|
| Subject | Just checking in, [First Name] |
| Body | Hi [First Name], We noticed it's been a little while since your last visit, and we wanted to reach out and see how you're doing. If you have a minute, we'd love to hear a bit about where things stand for you right now. This is completely optional, takes less than a minute, and won't affect anything if you'd ever like to come back. **[Share a quick note →]** (link to the form) Whatever's next for you, we're glad you gave us the chance to work together, and the door's always open. Warmly, The Cape Clarity Team |

`[First Name]` merge personalization was not used by any prior milestone's
email (M0–M4 all use a generic salutation-free body) — confirm during build
whether `Patients1` exposes a first-name field the Send Email step's Insert
Variable panel can map (the module's own `Name` field is "Patient Name," full
name, not first-name-only; `Full_Name_PHI` likewise). If no clean
first-name-only field/formula is available without new PII-handling work,
drop `[First Name]` from both the subject and body opening rather than
splitting or guessing at a name field — flag this substitution to
Costin/Liana rather than silently deviating from the sourced copy. The
post-submission closing line ("Thank you for letting us know. We wish you
well, and we're here whenever you're ready, if you ever are.") is Zoho
Forms' own thank-you-page text, set on the form itself, not the email.

**Rejected alternative**: drafting new copy the way M0–M4 had to. Not
applicable — rejected only in the sense that it wasn't needed; the source
page already has fully worked content, so no drafting judgment call was
required for the substantive question/answer copy itself (only the
[First Name] merge-field question above required a build-time judgment call,
and even that has a documented fallback).

## Decision 6: `Milestone` picklist value — reuse `"5B - Discontinuation, Email Fallback"` as-is, no CRM schema change

**Confirmed live via Zoho CRM MCP `getFields` on `Milestone_Instances`**:
current `Milestone` picklist is `-None-`, `0 - No Conversion`, `1 - Baseline
Intake`, `2 - Early Alliance Check`, `3 - Periodic Consolidated`, `4 -
Discharge`, `5B - Discontinuation, Email Fallback`, `5A - Discontinuation,
Live Capture`. Per spec.md's Assumptions, `5A` (`Staff Live Entry` capture
method, `Captured Live` status) is dead configuration from before the
operations lead's decision to drop the live-call path, and is to be
disregarded, not treated as part of the current design — this plan does not
use it and does not clean it up (out of scope, per spec.md's own "ideally
cleaned up in CRM" being a separate, not-yet-actioned suggestion, same as
`m0-implementation-notes.md` §5's treatment of the same dead value).

Two options considered for the value M5 actually issues:

**Option A (chosen)** — use `"5B - Discontinuation, Email Fallback"` exactly
as it already exists. Zero CRM schema/picklist change. Consistent with
`m4-plan.md`'s and `m2-data-model.md`'s repeated note that
`Milestone_Instances` is at (or near) its field cap and every prior milestone
has preferred reusing existing structure over adding to it. The "Email
Fallback" wording is now stale framing (there is no live-call primary path
left for it to be a fallback *from*, per spec.md's Assumptions), but it is a
display-label accuracy issue, not a functional one — the value still
correctly and uniquely identifies "Milestone 5, delivered by email," which is
the only path this version has.

**Option B (rejected, flagged for Costin as a future cleanup, not this
build)** — rename `5B`'s display label to a clean `"5 - Discontinuation"`
(matching the unprefixed naming of every other active milestone value:
`0 -`, `2 -`, `3 -`, `4 -`) via Zoho CRM Setup's field customization screen,
leaving `5A` untouched for audit. Rejected for this milestone's build: a
picklist rename is a CRM Setup/customization change, not a Milestone_Instances
*data* read — plausibly outside the standing "don't touch
Leads/Patients" constraint's letter (Setup is metadata, not patient records),
but it is still a live-CRM configuration change with no functional
requirement forcing it now, and this session prioritizes not touching CRM
configuration beyond what a milestone build actually needs. If Costin wants
the cleaner label, it's a one-screen Setup change independent of anything
else in this plan — noted here rather than actioned.

## Decision 7: Clinical Safety Flag Rules — not applicable to M5

spec.md's Clinical Safety Flag Rules section scopes the Alliance rule to
"alliance readings (Milestones 2, 3, 4)" explicitly. M5's survey (Decision 5)
has no alliance domains and no numeric scores of any kind — a reason-category
pick-list and a Yes/No/Maybe. User Story 4 / FR-011–013 are therefore out of
scope for M5, not merely inapplicable-by-omission: there is no widening
decision to make here the way M4's Decision 6 widened M2/M3's existing
columns, because M5 contributes no data those columns (or any new columns)
could evaluate. No Clinical Safety Flag work is planned for M5.

## Reporting (User Story 6 / FR-018 baseline bar)

Same "at minimum: a status/volume view and one distribution/summary view of
this milestone's own scored **or categorical** data" bar every prior
milestone has cleared (`m1-implementation-notes.md` §13,
`m2-implementation-notes.md` §9, `m3-implementation-notes.md` §1.7–1.8,
`m4-implementation-notes.md` §5). M5 is the first milestone whose own new
data is entirely categorical (no numeric domain) — FR-018's own wording
explicitly covers this ("scored **or categorical** data"; Acceptance Scenario
2 names "numeric or categorical (non-freeform) question" as triggering the
distribution-panel requirement), so a bar/breakdown of the "Reason For
Leaving" pick-list answer satisfies the distribution/summary requirement the
same way a numeric-domain bar chart did for M0–M4. The "Okay To Reach Back
Out" Yes/No/Maybe field is a second categorical field but does not need its
own separate distribution report to clear the FR-018 bar (one is sufficient,
per the requirement's "at least one" wording) — plan.md/data-model.md may
still include a second small breakdown for it if it's low-cost to add
alongside the first, consistent with M0's practice of not being stingier than
the bar requires when a report is cheap to add.

## Out of scope for M5

- The unified, cross-milestone Admin Dashboard (User Story 3 / FR-007–009) —
  explicitly deferred pipeline-wide, unaffected by this milestone.
- Raw Zoho Forms submission purge (FR-006) — the same open, tracked,
  cross-milestone gap `data-retention-purge.md` documents; M5 adds a sixth
  flow (fifth write-back flow; M0/M1 share a subflow-based issuance with no
  independent write-back flow of their own) with the identical gap, not
  resolved here.
- Computing the 14-day-no-rebooking / 2-consecutive-no-show pattern itself
  from appointment-level data (Decision 1, Option B) — no confirmed data
  source exists for this in the accessible CRM schema; the pattern remains a
  staff judgment reflected in `Patient_Status`, as it is for M0's `Lost Lead`
  and M4's `Completed Treatment`.
- Renaming the `5B` picklist value (Decision 6, Option B) — flagged as a
  possible future cleanup, not actioned by this build.
- `[First Name]` email personalization (Decision 5) if no clean first-name
  field/formula is confirmed available at build time — falls back to a
  generic salutation-free opening, matching M0–M4's own email bodies, rather
  than being solved with new PII-handling work in this milestone.
