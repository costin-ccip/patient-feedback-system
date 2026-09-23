# Phase 1 Data Model: M4 - Discharge

**Input**: `m4-plan.md`, `m4-research.md`

**Date**: 2026-09-23

## Milestone_Instances usage for M4

M4 reuses the existing `Milestone_Instances` module (from M0/M2/M3), patient-
based, not lead-based:

| Field | M2/M3 usage | M4 usage |
|---|---|---|
| `Milestone` | `"2 - Early Alliance Check"` / `"3 - Periodic Consolidated"` | `"4 - Discharge"` (existing picklist **value** on the existing field — confirmed live, see "Milestone picklist value" below; not a schema change) |
| `Lead_Reference` | Not used | Not used |
| `Patient` (lookup to `Patients1`) | Populated | Populated — same rejoin pattern, Principle II |
| `Status`, `Token`, `Expiry_Date_Time`, `Submitted_Date_Time`, `Response_Data` | Same lifecycle | Same lifecycle, different `Response_Data` content (6 segments) |
| `Name` | `"Milestone " + milestone + " - " + token.subString(0,8)` (feature 003) | Unchanged — `issueFeedbackToken` is reused as-is, so M4 records get this same privacy-safe naming with no extra work |

**No new CRM fields.** `Milestone_Instances` remains at its CRM custom-field
cap (re-confirmed live this session via `getFields`: 26 fields total, same
count `m2-data-model.md`/`m3-data-model.md` recorded — see "Field inventory,
confirmed live" below) — this applies to M4 exactly as it applied to M2/M3.

**One `Milestone_Instances` row per patient at `Milestone = "4 - Discharge"`.**
Unlike M3, M4 is one-time-per-patient (`m4-research.md` Decision 2), so the
"at most one row per patient per Milestone value" assumption every pre-M3
report/dashboard/function relies on holds for M4 exactly as it does for
M0/M2 — M3 remains the only milestone that breaks it.

## Field inventory, confirmed live (`getFields` on `Patients1` and `Milestone_Instances`, 2026-09-23)

**`Patients1`** (27 fields total; only the ones relevant to M4 shown):

| Field (API name) | Type | Values used by M4 |
|---|---|---|
| `Patient_Status` | picklist | `-None-`, `Active`, `Completed Treatment`, `Discontinued (Patient Choice)`, `Referred Out`, `No Show` — M4 watches for `Completed Treatment` (see `m4-research.md` Decision 1) |
| `Session_Count` | integer | Not used by M4 (M2/M3's field) |
| `Assigned_Therapist` | picklist | `-None-`, `Liana Preudhomme`, `Deborah Webster`, `Shana Lacastro` — not read by M4's trigger/functions (contractor attribution happens only inside CRM after the fact, per Principle II; not needed for issuance) |
| `Email` | email | `${trigger.Email}`, same as every prior milestone |
| `Created_Time` | datetime | `${trigger.Created_Time}`, mapped to `isCreatedAfterCutoff`'s `createdTime` parameter |

**`Milestone_Instances`** (26 fields total, unchanged from `m3-data-model.md`'s
count): `Milestone` picklist confirmed to already include `4 - Discharge`;
`Status` picklist confirmed to already include `Send Failed` (added by Costin
since feature 003 — see `m4-research.md` Decision 7).

## Milestone picklist value

**Confirmed live 2026-09-23 via `getFields` on `Milestone_Instances`**
(Zoho CRM MCP — allowed under the standing access constraint, since
`Milestone_Instances` is not a Leads/Patients module): `"4 - Discharge"`
**already exists** as a picklist value on `Milestone`, no action needed.
Full picklist as of this check: `-None-`, `0 - No Conversion`, `1 - Baseline
Intake`, `2 - Early Alliance Check`, `3 - Periodic Consolidated`, `4 -
Discharge`, `5B - Discontinuation, Email Fallback`, `5A - Discontinuation,
Live Capture` — unchanged from `m3-data-model.md`.

## Idempotency: `checkDischargeExists`

New custom function, field-for-field identical in shape to
`checkAllianceCheckExists` (`m2-implementation-notes.md` §3.1) except the
function name and the `Milestone` match string — per `m4-research.md`
Decision 2, M4 is one-time-per-patient, so the plain existence-check pattern
applies with no modification:

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

Input: `patientId` (string). Return: `bool`. Connection name `crm_connection`,
reused verbatim from M0/M1/M2/M3's pattern. Built genuinely fresh via
Built-ins → Developer Tools → Custom Functions (not cloned), per
`m2-implementation-notes.md` §3.2's shared-custom-function-across-clones
gotcha.

## Legacy-patient cutoff-exclusion: reusing `isCreatedAfterCutoff`

Per `m4-research.md` Decision 3, M4 reuses the existing shared function
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
`isCreatedAfterCutoff_1` in this flow (a separate instance from M2's/M3's,
scoped per flow even though the function body is shared).

## "M4 - Discharge Trigger" flow

1. **Trigger** — Zoho CRM "Updated module entry". Connection: "CRM
   Connection" (reused). Module: `Patients` (internal API name `Patients1`,
   variable name `trigger`, unchanged default). Filter criteria:
   `Patient Status` `equals` `Completed Treatment` (per `m4-research.md`
   Decision 1 — **flagged for Costin/Liana confirmation**; a one-line filter
   value change if a different `Patient_Status` value turns out to be the
   correct discharge marker).
2. **Custom Function** — `checkDischargeExists(patientId)`. Parameter:
   `patientId` = `${trigger.id}`. Output variable `checkDischargeExists_1`.
3. **Custom Function** — `isCreatedAfterCutoff(createdTime)` (shared,
   unmodified). Parameter: `createdTime` = `${trigger.Created_Time}`. Output
   variable `isCreatedAfterCutoff_1`.
4. **If else** — condition: `checkDischargeExists_1 is false` **AND**
   `isCreatedAfterCutoff_1 is true`.
   - **True branch → `issueFeedbackToken`** (shared, unmodified):
     - `milestone`: `4 - Discharge`
     - `patientId`: `${trigger.id}`
     - `leadId`: *(empty)* — M4 is patient-based only, same as M2/M3
     - `recipientEmail`: `${trigger.Email}`
     - `clinician`: `Liana Preudhomme`
     - `ttlDays`: `7` (reused M0/M2/M3's default — no evidence a discharge
       survey needs a different window)

     Output variable renamed `issueFeedbackToken_1`, matching every other
     milestone's convention (so the email body's merge field is identical
     everywhere).
   - **Then → Zoho Mail "Send email"** (Apps → Zoho Mail, not Built-ins →
     Notification): connection "Connection to info@capeclarity.com"; From
     `info@capeclarity.com`; To `${trigger.Email}`; link
     `<survey_url>?token=${issueFeedbackToken_1.token}` where `survey_url`
     is the Discharge Feedback form's Share-tab permalink (confirmed at
     build time, not the signed-in builder URL); subject and intro text
     drafted in `m4-implementation-notes.md` once built (not yet reviewed by
     Costin/Liana, same caveat every prior milestone's own copy carries).
   - **On Error of `issueFeedbackToken`** → Zoho Mail "Send email" alert:
     connection "Connection to info@capeclarity.com"; To
     `costin@capeclarity.com`; Subject `Cape Clarity feedback pipeline: token
     issuance failed (4 - Discharge)`; body contains no `${trigger.*}`
     identity fields, per feature 003's convention.
   - **On Error of the patient "Send email"** → Zoho CRM "Update module
     entry" (module `Milestone_Instances`, Entry Id
     `${issueFeedbackToken_1.recordId}`, `Status` = "Use a Custom Value" =
     `Send Failed`) → Zoho Mail "Send email" alert: same connection/From/To
     as the issuance-failure alert; Subject `Cape Clarity feedback pipeline:
     email send failed (4 - Discharge)`; body includes
     `${issueFeedbackToken_1.recordId}`, no `${trigger.*}` fields.
   - **False branch**: left empty — flow ends, no email, no new record, same
     idempotency short-circuit shape as every prior milestone.

Left switched **OFF** after building, matching every prior milestone's
build-first-verify-later convention.

## Response_Data blob format for the Discharge Feedback survey

6 `---`-delimited segments. Field order deliberately different from the
form's on-screen section order — see `m4-research.md` Decision 5:

```text
Looking Ahead: Likelihood To Recommend (0-10): {value}---Looking Ahead: Anything Else: {value}---Connection (0-10): {value}---Understanding (0-10): {value}---Shared direction (0-10): {value}---Fit of approach (0-10): {value}
```

The last 4 segments are **byte-for-byte identical label text** to M2/M3's
blob — same domain names, same `(0-10):` suffix, same order among
themselves — specifically so the existing "Domain: Connection", "Domain:
Understanding", "Domain: Shared Direction", and "Domain: Fit Of Approach"
Analytics formula columns parse M4 rows correctly with **zero formula
edits**, per `m4-research.md` Decision 5.

**Write-back function**: `submitDischargeFeedbackResponse` — same pure
string-concatenation shape as `submitAllianceCheckInResponse`/
`submitPeriodicCheckInResponse`, 7 input parameters (`token`,
`likelihoodToRecommend`, `anythingElse`, `connection`, `understanding`,
`sharedDirection`, `fitOfApproach`). Illustrative draft (confirm exact syntax
against the live Deluge editor when built, per every prior milestone's
"verify, don't assume" discipline):

```text
map submitDischargeFeedbackResponse(string token,string likelihoodToRecommend,string anythingElse,string connection,string understanding,string sharedDirection,string fitOfApproach)
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
	responseText = "Looking Ahead: Likelihood To Recommend (0-10): " + ifnull(likelihoodToRecommend,"") + "---";
	responseText = responseText + "Looking Ahead: Anything Else: " + ifnull(anythingElse,"") + "---";
	responseText = responseText + "Connection (0-10): " + ifnull(connection,"") + "---";
	responseText = responseText + "Understanding (0-10): " + ifnull(understanding,"") + "---";
	responseText = responseText + "Shared direction (0-10): " + ifnull(sharedDirection,"") + "---";
	responseText = responseText + "Fit of approach (0-10): " + ifnull(fitOfApproach,"");
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

## "Cape Clarity Discharge Feedback" Zoho Form

6 substantive fields + hidden token, on-screen order: Alliance Check-In (1,
reused sliders), Looking Ahead (2, new fields):

1. **Slider — Connection** (reused verbatim from M2/M3): "Overall, how
   comfortable have you felt being open and honest with your therapist so
   far?" / "Domain: Connection. 0 = Not comfortable, 10 = Very comfortable."
   / range 0-10 / Mandatory.
2. **Slider — Understanding** (reused verbatim): "Overall, how well do you
   feel your therapist has understood what matters to you so far?" /
   "Domain: Understanding. 0 = Not understood, 10 = Fully understood." /
   range 0-10 / Mandatory.
3. **Slider — Shared direction** (reused verbatim): "Overall, how much do
   you feel you and your therapist agree on what you're working toward?" /
   "Domain: Shared direction. 0 = Not aligned, 10 = Fully aligned." / range
   0-10 / Mandatory.
4. **Slider — Fit of approach** (reused verbatim): "Overall, how well has
   your therapist's approach been working for you so far?" / "Domain: Fit
   of approach. 0 = Hasn't worked, 10 = Worked well." / range 0-10 /
   Mandatory.
5. **Slider — Likelihood to recommend** (new, drafted this session, **not
   yet reviewed by Costin/Liana** — per `m4-research.md` Decision 4): "How
   likely are you to recommend Cape Clarity to someone in a similar
   situation?" / "0 = Not at all likely, 10 = Extremely likely." / range
   0-10 / Mandatory.
6. **Multi Line — Anything else looking ahead** (new, same review caveat):
   "Is there anything else you'd like to share as you finish up your care
   with us?" / Optional.
7. **Single Line — Token**: label "Token", Visibility = Hide. Same
   hidden-prefill-token pattern as every prior form.

No name, email, or phone field, per Principle I.

## Analytics formula columns (Milestone Instances table)

**Unchanged, reused as-is** (per Decision 5's field-order choice):
- Domain: Connection, Domain: Understanding, Domain: Shared Direction
  (bounded `substring_between`)
- Domain: Fit Of Approach (unbounded last-field pattern — stays correct for
  M4 rows only because Decision 5 keeps it last in the blob)
- Alliance Check-In Total (`SUM`/`to_integer()` of the 4 Domain columns)

**New, M4-only** (bounded `substring_between`, since each is followed by
another segment in the chosen order):
- **Looking Ahead: Likelihood To Recommend** —
  `substring_between("Milestone Instances"."Response Data", 'Looking Ahead: Likelihood To Recommend (0-10): ', '---', 1)`
- **Looking Ahead: Anything Else** —
  `substring_between("Milestone Instances"."Response Data", 'Looking Ahead: Anything Else: ', '---', 1)`

**Modified, shared with M2/M3** (`m4-research.md` Decision 6 — requires
`m2-implementation-notes.md` AND `m3-implementation-notes.md` updates in the
same build session):
- **Clinical Safety Flag** — widen the `Milestone` gate a second time:
  ```
  IF("Milestone Instances"."Milestone" = '2 - Early Alliance Check' OR "Milestone Instances"."Milestone" = '3 - Periodic Consolidated' OR "Milestone Instances"."Milestone" = '4 - Discharge', IF("Milestone Instances"."Alliance Check-In Total" <= 20 OR to_integer("Milestone Instances"."Domain: Connection") <= 4 OR to_integer("Milestone Instances"."Domain: Understanding") <= 4 OR to_integer("Milestone Instances"."Domain: Shared Direction") <= 4 OR to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, 'true', 'false'), '')
  ```
  (infix `OR`, per `m3-implementation-notes.md` §1.1's confirmed finding that
  `OR(a,b)` as a function call fails to parse in this workspace.)
- **Flag Rule Triggered** — same nested `IF`/`concat` body as M2/M3's version,
  unchanged except that it now evaluates for M4 rows too, since it already
  reads `Clinical Safety Flag` rather than re-testing `Milestone` itself — no
  formula text change needed for this column, only for `Clinical Safety
  Flag` above.

Exact Analytics formula syntax is to be verified against the live workspace
at build time, per every prior milestone's "confirm against the actual UI,
don't assume" discipline.

## Reporting (User Story 6 / FR-018 baseline bar; FR-013 flag visibility)

- **"M4 Submitted Responses"** — new Tabular View, same shape as M2/M3's:
  Submitted Date Time, Token, Looking Ahead: Likelihood To Recommend, Looking
  Ahead: Anything Else, Domain: Connection, Domain: Understanding, Domain:
  Shared Direction, Domain: Fit Of Approach, Alliance Check-In Total,
  Clinical Safety Flag, Flag Rule Triggered. No `Patient`/identity column.
  Filtered to `Milestone` Wildcard Exactly Matches `"4 - Discharge"`.
- **"M4 Status Breakdown"** — status/volume view, "Save As" off M2's or M3's
  equivalent, Milestone filter swapped to `"4 - Discharge"`.
- **"M4 Looking Ahead: Likelihood To Recommend Distribution"** —
  distribution/summary view satisfying FR-018/User Story 6's "at least one
  distribution or summary view of that milestone's own scored...data"
  requirement for M4's *new* content specifically (the reused alliance
  domains are already covered by M2's/M3's own equivalent views) — bar chart
  over the new Likelihood To Recommend column, `Dimension [Actual(D)] / Treat
  as Text` mode per `m2-implementation-notes.md` §9.5's gotcha (confirm this
  column is Text-typed like M3's equivalents, which needed no explicit
  re-set; a numeric-typed column would need one). The freeform "Looking
  Ahead: Anything Else" column gets no distribution panel — freeform text
  isn't scored/categorical, same treatment M0's own freeform fields got.
- **"M2, M3 & M4 Flagged for Review"** — the existing shared report
  (`m3-implementation-notes.md` §1.3, view ID `3251423000000141219`),
  **renamed and its Milestone filter widened a second time** to `Wildcard`
  `"2 - Early Alliance Check"` OR `"3 - Periodic Consolidated"` OR `"4 -
  Discharge"`, rather than a third, parallel M4-only flagged view — same
  reasoning as the Clinical Safety Flag column widening above. **This is a
  rename + filter change on a report M2/M3 already own; update
  `m2-implementation-notes.md`'s §9.6 AND `m3-implementation-notes.md`'s §1.3
  references in the same session.**
- **"M4 - Discharge Feedback" dashboard** — new dashboard bundling the four
  views above, built via "Create New Dashboards" (not "Save As"), same shape
  as M0/M1/M2/M3's own dashboards.

## Out of scope for M4

Same list as `m4-research.md`'s "Out of scope for M4" section — not repeated
here; see that document.
