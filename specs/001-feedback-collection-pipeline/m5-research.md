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

**Revision (same day)**: this file originally scoped M5's trigger as a
two-legged OR across `Patient_Status = "Discontinued (Patient Choice)"` and
`Patient_Status = "No Show"`, covering both of spec.md's Milestones-table
trigger legs (cancellation-no-rebooking, and the no-show pattern). **Costin's
explicit decision**: build M5 against the `Discontinued (Patient Choice)` leg
only; drop the `No Show` leg from this build entirely. Decision 1 below and
everything downstream of it (idempotency, the trigger flow, testing) reflect
that decision. This is a scope narrowing for *this build*, not a spec
correction — spec.md's Milestones table still describes both legs as the
milestone's full intended trigger; see "Open item for spec.md" at the end of
this file.

## Why M5 turns out to be structurally like M4 after all

`m3-plan.md`'s prospective "Precedent for M4/M5" note and `m4-plan.md`'s
closing note both anticipated M5 needing genuinely new trigger logic, because
spec.md's Milestones table describes a two-legged pattern rather than a
single field transition. With Costin's decision to scope this build to the
`Discontinued (Patient Choice)` leg only, that concern no longer applies:
M5's trigger is now a single `Patient_Status` equality check, the same shape
as M0's `Lost Lead` and M4's `Completed Treatment` — no OR condition, no new
Flow-builder territory to confirm. What remains genuinely new about M5 is on
the content/reporting side, not the trigger: it is the first milestone with
zero numeric/scored survey content, and its survey/email copy is sourced
verbatim from Confluence (page 564035589) rather than drafted-and-flagged the
way M0-M4's own new copy had to be.

## Decision 1: Mapping "discontinuation" to a real CRM field — `Discontinued (Patient Choice)` only, by Costin's explicit scope decision

**Confirmed live via Zoho CRM MCP `getFields` on `Patients1`** (metadata
only, same allowed access as every prior milestone's schema check): the
module's `Patient_Status` picklist is `-None-`, `Active`, `Completed
Treatment`, `Discontinued (Patient Choice)`, `Referred Out`, `No Show`.

This session initially considered both `Discontinued (Patient Choice)` and
`No Show` as candidate values (one per spec.md trigger leg — see the
Revision note above for the original two-value reasoning). **Costin's
decision**: use `Discontinued (Patient Choice)` only. `No Show` is
explicitly out of scope for this build — not wired into the trigger, not
referenced by the idempotency check, not covered by this milestone's test
plan. If a `No Show`-triggered version of M5 (or its own separate milestone)
is wanted later, that is new, separate planning work, not an extension to
retrofit onto this build.

M5's trigger is therefore, structurally, identical in shape to M4's Decision
1 (`m4-research.md`): a single `Patient_Status` value, inferred from the live
picklist (not named verbatim in spec.md's prose), flagged for Costin/Liana
confirmation the same way M0's `Lost Lead` and M4's `Completed Treatment`
mappings were. **Flag for Costin/Liana**: confirm `Discontinued (Patient
Choice)` is in fact what staff select for the cancellation-with-no-rebooking
case this milestone targets — the underlying 14-day-window judgment remains a
staff call recorded as this status value, the same division of labor M0/M4
already establish (Constitution Principle III requires the *feedback
request* to be system-triggered without contractor/staff action, not that
the practice's own status-marking be computed rather than judged).

**Rejected alternative (this build)**: including the `No Show` leg as a
second OR condition, per this file's original draft. Superseded by Costin's
explicit instruction to keep one leg only.

## Decision 2: Idempotency — plain existence check, same shape as M0/M2/M4

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

## Decision 3: Legacy-patient cutoff-exclusion gate — yes, M5 needs it

Same reasoning as `m4-research.md` Decision 3, which itself generalizes from
M2/M3: the risk is inherent to any "Updated module entry" trigger evaluating
a CRM-level-state filter against a `Patients1` record's *current* field
values on *any* update to that record, not specific to how the milestone's
condition is worded or how often the watched field is expected to change. A
patient already marked `Discontinued (Patient Choice)` long before this flow
exists would otherwise get a surprise feedback request the next time any
unrelated field on their record is touched. M5's trigger is on `Patients1`,
exactly like M2/M3/M4 (not `Leads`, M0's genuine exception) — reuse the same
shared `isCreatedAfterCutoff(string createdTime) -> bool` function (feature
004, unmodified), wired between `checkDiscontinuationExists` and the `If
else`, `createdTime` = `${trigger.Created_Time}`, output variable
`isCreatedAfterCutoff_1` (a separate per-flow instance, same as every prior
consumer).

**Rejected alternative**: skip the gate on the theory that a
patient-discontinuation status is even rarer/more deliberate than a discharge
status. Rejected for the same categorical reason `m4-research.md` Decision 3
already rejected it for M4: the trigger mechanism's behavior doesn't depend
on how rare the watched value is expected to be.

## Decision 4: Survey structure and Response_Data blob — verbatim from Confluence, no scored/numeric content

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

## Decision 5: `Milestone` picklist value — reuse `"5B - Discontinuation, Email Fallback"` as-is, no CRM schema change

**Confirmed live via Zoho CRM MCP `getFields` on `Milestone_Instances`**:
current `Milestone` picklist is `-None-`, `0 - No Conversion`, `1 - Baseline
Intake`, `2 - Early Alliance Check`, `3 - Periodic Consolidated`, `4 -
Discharge`, `5B - Discontinuation, Email Fallback`, `5A - Discontinuation,
Live Capture`. Per spec.md's Assumptions, `5A` (`Staff Live Entry` capture
method, `Captured Live` status) is dead configuration from before the
operations lead's decision to drop the live-call path, and is to be
disregarded, not treated as part of the current design — this plan does not
use it and does not clean it up.

**Chosen**: use `"5B - Discontinuation, Email Fallback"` exactly as it
already exists. Zero CRM schema/picklist change, consistent with
`Milestone_Instances` being at (or near) its field cap and every prior
milestone's preference for reusing existing structure over adding to it. The
"Email Fallback" wording is stale framing (there is no live-call primary
path left for it to be a fallback *from*), but that's a display-label
accuracy issue, not a functional one.

**Rejected alternative, flagged for Costin as a possible future cleanup, not
this build**: rename `5B`'s display label to a clean `"5 - Discontinuation"`
(matching the unprefixed naming of every other active milestone value) via
Zoho CRM Setup's field customization screen. Not actioned now — a live CRM
Setup/configuration change with no functional requirement forcing it.

## Decision 6: Clinical Safety Flag Rules — not applicable to M5

spec.md's Clinical Safety Flag Rules section scopes the Alliance rule to
"alliance readings (Milestones 2, 3, 4)" explicitly. M5's survey (Decision 4)
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
alongside the first.

## Out of scope for M5

- The `No Show` trigger leg (Decision 1) — Costin's explicit scope decision
  for this build. Not wired into the trigger, idempotency check, or test
  plan. Revisit as separate, future planning if wanted later.
- The unified, cross-milestone Admin Dashboard (User Story 3 / FR-007–009) —
  explicitly deferred pipeline-wide, unaffected by this milestone.
- Raw Zoho Forms submission purge (FR-006) — the same open, tracked,
  cross-milestone gap `data-retention-purge.md` documents; M5 adds a sixth
  flow with the identical gap, not resolved here.
- Renaming the `5B` picklist value (Decision 5) — flagged as a possible
  future cleanup, not actioned by this build.
- `[First Name]` email personalization (Decision 4) if no clean first-name
  field/formula is confirmed available at build time — falls back to a
  generic salutation-free opening, matching M0–M4's own email bodies, rather
  than being solved with new PII-handling work in this milestone.

## Open item for spec.md (not actioned here — flagging only)

spec.md's Milestones table (row 5) and User Story 5 still describe M5's
trigger as both legs ("Cancellation with no rebooking within 14 days, OR 2
consecutive no-shows with no reschedule in between"). Building against the
`Discontinued (Patient Choice)` leg only (Decision 1) means this build does
not fully satisfy that row as written — it satisfies the cancellation leg,
not the no-show leg. Every prior scope change of this kind in this repo (the
Milestone 1 retirement, the wellbeing-rule removal) was recorded as a dated
revision-note in spec.md itself, with rationale, rather than left as a silent
gap between the spec and what got built. This file does not make that edit —
it's flagged here for Costin to decide: treat the no-show leg as deferred
(and, if so, worth a short spec.md note saying so, the same way this
project has documented every other narrowing of scope), or treat it as
dropped for good (which would be a more permanent spec.md revision, in the
same spirit as the fifth-pass revision that retired Milestone 1).
