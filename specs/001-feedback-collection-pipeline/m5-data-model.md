# Phase 1 Data Model: M5 - Discontinuation

**Input**: `m5-plan.md`, `m5-research.md`

**Date**: 2026-09-24

## Milestone_Instances usage for M5

M5 reuses the existing `Milestone_Instances` module (from M0/M2/M3/M4),
patient-based, not lead-based:

| Field | M4 usage | M5 usage |
|---|---|---|
| `Milestone` | `"4 - Discharge"` | `"5B - Discontinuation, Email Fallback"` (existing picklist **value**, reused as-is — see `m5-research.md` Decision 6; not a schema change) |
| `Lead_Reference` | Not used | Not used |
| `Patient` (lookup to `Patients1`) | Populated | Populated — same rejoin pattern, Principle II |
| `Status`, `Token`, `Expiry_Date_Time`, `Submitted_Date_Time`, `Response_Data` | Same lifecycle | Same lifecycle, different `Response_Data` content (2 segments — the shortest of any milestone) |
| `Name` | `"Milestone " + milestone + " - " + token.subString(0,8)` (feature 003) | Unchanged — `issueFeedbackToken` is reused as-is |

**No new CRM fields.** `Milestone_Instances` remains at its CRM custom-field
cap (26 fields total, unchanged since `m2-data-model.md`/`m3-data-model.md`/
`m4-data-model.md` — re-confirmed live this session via `getFields`).

**One `Milestone_Instances` row per patient at `Milestone = "5B -
Discontinuation, Email Fallback"`.** M5 is one-time-per-patient
(`m5-research.md` Decision 3), so the "at most one row per patient per
Milestone value" assumption every non-M3 report/dashboard/function relies on
holds for M5 exactly as it does for M0/M2/M4.

## Field inventory, confirmed live (`getFields` on `Patients1` and `Milestone_Instances`, 2026-09-24)

**`Patients1`** (27 fields total; only the ones relevant to M5 shown):

| Field (API name) | Type | Values used by M5 |
|---|---|---|
| `Patient_Status` | picklist | `-None-`, `Active`, `Completed Treatment`, `Discontinued (Patient Choice)`, `Referred Out`, `No Show` — M5 watches for `Discontinued (Patient Choice)` **OR** `No Show` (see `m5-research.md` Decision 1) |
| `Session_Count` | integer | Not used by M5 |
| `Assigned_Therapist` | picklist | `-None-`, `Liana Preudhomme`, `Deborah Webster`, `Shana Lacastro` — not read by M5's trigger/functions, same as every prior milestone (Principle II: attribution happens only inside CRM, after the fact, and is not needed for issuance) |
| `Email` | email | `${trigger.Email}`, same as every prior milestone |
| `Created_Time` | datetime | `${trigger.Created_Time}`, mapped to `isCreatedAfterCutoff`'s `createdTime` parameter |
| `Name` ("Patient Name") | text | Full name only — no first-name-only field exists; see the email-personalization note below |

**`Milestone_Instances`** (26 fields total, unchanged): `Milestone` picklist
confirmed to already include `"5B - Discontinuation, Email Fallback"`;
`Status` picklist confirmed to already include `Send Failed`.

## Milestone picklist value

**Confirmed live 2026-09-24 via `getFields` on `Milestone_Instances`** (Zoho
CRM MCP — allowed under the standing access constraint, `Milestone_Instances`
is not a Leads/Patients module): `"5B - Discontinuation, Email Fallback"`
**already exists** as a picklist value on `Milestone`; reused as-is, no
schema change (`m5-research.md` Decision 6). Full picklist as of this check:
`-None-`, `0 - No Conversion`, `1 - Baseline Intake`, `2 - Early Alliance
Check`, `3 - Periodic Consolidated`, `4 - Discharge`, `5B - Discontinuation,
Email Fallback`, `5A - Discontinuation, Live Capture`. `5A` remains dead,
disregarded configuration, untouched by this build.

## Idempotency: `checkDiscontinuationExists`

New custom function, field-for-field identical in shape to
`checkDischargeExists` (`m4-implementation-notes.md` §2) except the function
name and the `Milestone` match string — per `m5-research.md` Decision 3, M5
is one-time-per-patient regardless of which of the two trigger legs fired it:

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

Input: `patientId` (string). Return: `bool`. Connection name
`crm_connection`, reused verbatim from every prior milestone's pattern. Built
genuinely fresh via Built-ins → Developer Tools → Custom Functions (not
cloned), per `m2-implementation-notes.md` §3.2's shared-custom-function-
across-clones gotcha.

## Legacy-patient cutoff-exclusion: reusing `isCreatedAfterCutoff`

Per `m5-research.md` Decision 4, M5 reuses the existing shared function
(`specs/004-legacy-patient-exclusion/implementation-notes.md`, unmodified):

```text
bool isCreatedAfterCutoff(string createdTime)
{
	cutoff = "2026-09-23T00:00:00-04:00".toDateTime();
	created = createdTime.toDateTime();
	return created >= cutoff;
}
```

`createdTime` mapped to `${trigger.Created_Time}`. Output variable renamed
`isCreatedAfterCutoff_1` in this flow (a separate per-flow instance).

## "M5 - Discontinuation Trigger" flow

1. **Trigger** — Zoho CRM "Updated module entry". Connection: "CRM
   Connection" (reused). Module: `Patients` (internal API name `Patients1`).
   Filter criteria (per `m5-research.md` Decision 2, **confirm the OR
   grouping against the live trigger-criteria builder before assuming it
   saves as written**):
   - **Preferred**: `Patient Status` `equals` `Discontinued (Patient Choice)`
     **OR** `Patient Status` `equals` `No Show`, as a single trigger-level
     filter.
   - **Fallback**, if the builder does not support an OR group on one field
     at the trigger level: drop this filter to `Patient Status` `is not
     empty` (or remove the trigger-level filter altogether) and move the
     OR-check into the flow's own `If else` (step 4).
2. **Custom Function** — `checkDiscontinuationExists(patientId)`. Parameter:
   `patientId` = `${trigger.id}`. Output variable
   `checkDiscontinuationExists_1`.
3. **Custom Function** — `isCreatedAfterCutoff(createdTime)` (shared,
   unmodified). Parameter: `createdTime` = `${trigger.Created_Time}`. Output
   variable `isCreatedAfterCutoff_1`.
4. **If else** — condition:
   - Under the **preferred** trigger-filter design: `checkDiscontinuationExists_1
     is false` **AND** `isCreatedAfterCutoff_1 is true` (the Patient_Status OR
     is already fully handled by the trigger filter, same 2-clause shape as
     M2/M3/M4's `If else` nodes).
   - Under the **fallback** design: `checkDiscontinuationExists_1 is false`
     **AND** `isCreatedAfterCutoff_1 is true` **AND**
     (`${trigger.Patient_Status} == "Discontinued (Patient Choice)"` **OR**
     `${trigger.Patient_Status} == "No Show"`) — a 3-clause condition, the
     first 3-clause `If else` this pipeline has needed; confirm the condition
     builder supports nesting an OR sub-group inside an outer AND the same
     way it's expected to (mirrors the trigger-level OR question in shape,
     just relocated).
   - **True branch → `issueFeedbackToken`** (shared, unmodified):
     - `milestone`: `5B - Discontinuation, Email Fallback`
     - `patientId`: `${trigger.id}`
     - `leadId`: *(empty)* — M5 is patient-based only, same as M2/M3/M4
     - `recipientEmail`: `${trigger.Email}`
     - `clinician`: `Liana Preudhomme` (hardcoded, same convention M2/M3/M4
       use — contractor attribution happens only inside CRM after the fact,
       per Principle II; this field is not read back for that purpose)
     - `ttlDays`: `7` (reused default; nothing in spec.md or the Confluence
       source suggests a different window for this milestone)

     Output variable renamed `issueFeedbackToken_1`.
   - **Then → Zoho Mail "Send email"** (Apps → Zoho Mail, not Built-ins →
     Notification): connection "Connection to info@capeclarity.com"; From
     `info@capeclarity.com`; To `${trigger.Email}`; Subject `Just checking
     in` (see the `[First Name]` note below); Body per `m5-research.md`
     Decision 5's sourced copy, link
     `<survey_url>?token=${issueFeedbackToken_1.token}` where `survey_url` is
     the "Cape Clarity Discontinuation Feedback" form's Share-tab permalink
     (confirmed at build time via DOM verification, same as every prior
     milestone).
   - **On Error of `issueFeedbackToken`** → Zoho Mail "Send email" alert:
     connection "Connection to info@capeclarity.com"; To
     `costin@capeclarity.com`; Subject `Cape Clarity feedback pipeline: token
     issuance failed (5B - Discontinuation, Email Fallback)`; body contains
     no `${trigger.*}` identity fields, per feature 003's convention.
   - **On Error of the patient "Send email"** → Zoho CRM "Update module
     entry" (module `Milestone_Instances`, Entry Id
     `${issueFeedbackToken_1.recordId}`, `Status` = "Use a Custom Value" =
     `Send Failed`) → Zoho Mail "Send email" alert: same connection/From/To
     as the issuance-failure alert; Subject `Cape Clarity feedback pipeline:
     email send failed (5B - Discontinuation, Email Fallback)`; body includes
     `${issueFeedbackToken_1.recordId}`, no `${trigger.*}` fields.
   - **False branch**: left empty.

Left switched **OFF** after building, matching every prior milestone's
build-first-verify-later convention.

**`[First Name]` email personalization**: the Confluence source's subject
("Just checking in, [First Name]") and body opening ("Hi [First Name],")
are the first milestone copy in this pipeline to use a merge field beyond
the token link. `Patients1` has no first-name-only field (`Name` is full
"Patient Name"; `Full_Name_PHI` is likewise full name) — confirm at build
time whether the Send Email step's Insert Variable panel exposes a
first-name-only token derived from `Name` (some Zoho Mail send-email nodes
offer basic string functions on merge fields). If not, per `m5-research.md`
Decision 5's fallback, drop `[First Name]` from both the subject and body
opening rather than inventing new PII-handling — resulting text: Subject
`Just checking in`; Body opens `Hi, We noticed it's been a little while...`
adjusted to read naturally without the name (e.g. "Hi there," or dropping the
opening "Hi [First Name]," line entirely) — **flag this substitution
explicitly in `m5-implementation-notes.md` if it's needed**, since it's a
deviation from the sourced copy, unlike everything else in this milestone's
content.

## Response_Data blob format for the Discontinuation Feedback survey

2 `---`-delimited segments — the shortest of any milestone, and the first
with no numeric/scored content at all:

```text
Reason For Leaving: {value}---Okay To Reach Back Out: {value}
```

**Write-back function**: `submitDiscontinuationFeedbackResponse` — same pure
string-concatenation shape as every prior write-back function, 3 input
parameters (`token`, `reasonForLeaving`, `okayToReachBackOut`). Illustrative
draft (confirm exact syntax against the live Deluge editor when built):

```text
map submitDiscontinuationFeedbackResponse(string token,string reasonForLeaving,string okayToReachBackOut)
{
	resp = Map();
	if(token == null || token.trim() == "")
	{
		resp.put("status","error");
		resp.put("message","Missing token");
		return resp;
	}
	rec = Map();
	found = false;
	qmap = Map();
	allRecs = zoho.crm.getRecords("Milestone_Instances",1,200,qmap,"crm_connection");
	for each  r in allRecs
	{
		if(r.get("Token") == token)
		{
			rec = r;
			found = true;
		}
	}
	if(!found)
	{
		resp.put("status","error");
		resp.put("message","No Milestone Instance found for token");
		return resp;
	}
	recStatus = rec.get("Status");
	expiry = rec.get("Expiry_Date_Time");
	if(recStatus != "Issued")
	{
		resp.put("status","error");
		resp.put("message","Milestone Instance is not in Issued status (current: " + recStatus + ")");
		resp.put("recordId",rec.get("id"));
		return resp;
	}
	if(expiry != null && expiry != "" && expiry < zoho.currenttime)
	{
		expUpdateMap = Map();
		expUpdateMap.put("Status","Expired");
		zoho.crm.updateRecord("Milestone_Instances",rec.get("id"),expUpdateMap,Map(),"crm_connection");
		resp.put("status","error");
		resp.put("message","Token has expired");
		resp.put("recordId",rec.get("id"));
		return resp;
	}
	responseText = "Reason For Leaving: " + ifnull(reasonForLeaving,"") + "---";
	responseText = responseText + "Okay To Reach Back Out: " + ifnull(okayToReachBackOut,"");
	updateMap = Map();
	updateMap.put("Response_Data",responseText);
	updateMap.put("Status","Submitted");
	updateMap.put("Submitted_Date_Time",zoho.currenttime.toString("yyyy-MM-dd'T'HH:mm:ssXXX"));
	updateResp = zoho.crm.updateRecord("Milestone_Instances",rec.get("id"),updateMap,Map(),"crm_connection");
	resp.put("status","success");
	resp.put("message","Milestone Instance updated");
	resp.put("recordId",rec.get("id"));
	return resp;
}
```

No arithmetic, no conditional flag logic, per Constitution Principle VII —
identical token-lookup / `Status != "Issued"` rejection / expiry-check-and-
auto-expire structure as every prior write-back function.

## "Cape Clarity Discontinuation Feedback" Zoho Form

2 substantive fields + hidden token — the shortest form of any milestone,
content sourced verbatim from Confluence (`m5-research.md` Decision 5), not
drafted:

1. **Dropdown (or Radio — decide against the live Forms builder's ergonomics
   for an 8-option single-select; a Dropdown likely reads cleaner than 8
   stacked radio buttons, but this is a builder/UX choice, not a content
   one)** — "What led to stepping away from sessions right now?" Options, in
   source order:
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
3. **Single Line — Token**: label "Token", Visibility = Hide. Same
   hidden-prefill-token pattern as every prior form.

No name, email, or phone field, per Principle I. No open-text field at all
(per the source's own explicit "no open text" design note).

**Thank-you page** (set on the form itself, not the email): "Thank you for
letting us know. We wish you well, and we're here whenever you're ready, if
you ever are."

## Analytics formula columns (Milestone Instances table)

**New, M5-only** (bounded `substring_between` for the first segment; the
second/last segment is unbounded, following the established "last segment
unbounded" pattern):

- **Reason For Leaving** —
  `substring_between("Milestone Instances"."Response Data", 'Reason For Leaving: ', '---', 1)`
- **Okay To Reach Back Out** — unbounded last-segment extraction (confirm
  the exact function against the live workspace; M4's own unbounded-last-
  field column, "Domain: Fit Of Approach", is the precedent to copy the
  approach from, not the formula text, since the label/prefix differs):
  a `substring_after`/equivalent function reading everything following
  `'Okay To Reach Back Out: '`.

**Not applicable**: no Clinical Safety Flag column changes — per
`m5-research.md` Decision 7, the Alliance rule doesn't apply to M5's data,
and M5 introduces no alliance domains for any rule to evaluate.

Exact Analytics formula syntax is to be verified against the live workspace
at build time, per every prior milestone's "confirm against the actual UI,
don't assume" discipline.

## Reporting (User Story 6 / FR-018 baseline bar)

- **"M5 Submitted Responses"** — new Tabular View: Submitted Date Time,
  Token, Reason For Leaving, Okay To Reach Back Out. No `Patient`/identity
  column, no scored/flag columns (none exist for M5). Filtered to
  `Milestone` Wildcard Exactly Matches `"5B - Discontinuation, Email
  Fallback"`.
- **"M5 Status Breakdown"** — status/volume view, "Save As" off M4's (or
  any prior milestone's) equivalent, Milestone filter swapped to `"5B -
  Discontinuation, Email Fallback"`. Satisfies FR-018's status/volume-view
  requirement.
- **"M5 Reason For Leaving Distribution"** — distribution/summary view over
  the categorical `Reason For Leaving` column (bar chart, one bar per reason
  category), `Dimension [Actual(D)] / Treat as Text` mode per
  `m2-implementation-notes.md` §9.5's gotcha if the column doesn't already
  render as plain `Actual`. Satisfies FR-018/User Story 6's "at least one
  distribution or summary view of that milestone's own scored **or
  categorical** data" requirement — the first milestone where this view is a
  categorical breakdown rather than a numeric-score bar chart.
- **"M5 Okay To Reach Back Out Breakdown"** (optional second breakdown, low
  cost to add alongside the one above per `m5-research.md`'s Reporting
  section — not required to clear the FR-018 bar, which needs only one) —
  same shape, over the 3-value `Okay To Reach Back Out` column.
- **"M5 - Discontinuation Feedback" dashboard** — new dashboard bundling the
  reports above, built via "Create New Dashboards" (not "Save As," since no
  prior dashboard has a matching panel set), same shape as
  M0/M1/M2/M3/M4's own dashboards. No shared "Flagged for Review" panel to
  include (Decision 7 — not applicable to M5).

## Out of scope for M5

Same list as `m5-research.md`'s "Out of scope for M5" section — not repeated
here; see that document.
