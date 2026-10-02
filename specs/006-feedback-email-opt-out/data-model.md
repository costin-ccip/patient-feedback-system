# Data Model: Feedback Email Opt-Out

**Feature**: `006-feedback-email-opt-out` | **Date**: 2026-10-01

Everything below is a draft until built. Deluge in particular must be saved and run with
Execute in the function editor before any flow uses it (T009 to T012), as with features 002
and 004.

## 1. CRM fields (Leads and Patients1, identical on both)

| Field label | API name (proposed) | Type | Notes |
|---|---|---|---|
| Feedback Opt-Out | `Feedback_Opt_Out` | Checkbox | The one flag every check reads. Default unchecked. |
| Feedback Opt-Out Date | `Feedback_Opt_Out_Date` | Date | Filled by the flow (email link) or the CRM workflow (staff). |
| Opted Out Via Email Link | `Feedback_Opt_Out_Via_Link` | Checkbox | Set only by the flow. Staff-recorded opt-outs leave it unchecked, and staff add a short CRM note saying how it arrived (phone, in person, reply). |
| Feedback Resubscribe Date | `Feedback_Resubscribe_Date` | Date | Filled by the CRM workflow when the flag is cleared. |

**Field budget (decided 2026-10-01, Option B):** four new fields per module, all in the
checkbox and date pools (2 checkboxes, 2 dates). No text or picklist field is used, because
that pool is nearly full on Leads and Patients1 (research.md §9). A "source" picklist was
considered and dropped.

Placement: a small "Feedback preferences" section on the record layout, visible to admin and
staff profiles only. Contractors have no CRM access (Costin, 2026-10-01), so no field-level
hiding is needed; if that ever changes, hide these four fields from contractor profiles first
(constitution Principle IV, spec FR-012). Turn on field-history tracking for the checkbox if
the plan allows.

Not added to: the Zoho Analytics sync field list, any Campaigns audience mapping, any export.

### CRM workflow rule (one per module)

| Rule | Condition | Action |
|---|---|---|
| Opt-out set | `Feedback_Opt_Out` changed to checked | If `Feedback_Opt_Out_Date` is empty, set it to today. |
| Opt-out cleared | `Feedback_Opt_Out` changed to unchecked | Set `Feedback_Resubscribe_Date` to today and uncheck `Feedback_Opt_Out_Via_Link`. Leave the opt-out date in place so the history stays readable. |

When the flow sets the flag it sets the date and the via-link checkbox itself in the same
update, so the first rule does not overwrite them. When someone opts out again after a resubscribe, the old date is
overwritten by the flow or by staff clearing and re-entering it; the CRM timeline holds the
earlier values.

## 2. `Milestone_Instances` change

| Field | Change |
|---|---|
| `Status` picklist | Add `Skipped - Opted Out`. Existing values stay: Issued, Submitted, Expired, Superseded, Send Failed, Captured Live. |

A skip row is written like this:

| Field | Value |
|---|---|
| `Name` | `Milestone <m> - skipped (opted out)` |
| `Milestone` | the milestone being skipped (picklist value, e.g. `3 - Periodic Consolidated`) |
| `Status` | `Skipped - Opted Out` |
| `Patient` | Patients1 ID for M2 to M5 |
| `Lead_Reference` | Lead ID for M0 |
| `Clinician` | the same literal the flow passes to `issueFeedbackToken` |
| `Token`, `Expiry_Date_Time`, `Email`, `Response_Data` | not set |

Created without CRM workflow, approval or blueprint triggers (no `trigger` option), so no
other automation reacts to it.

## 3. `skipIfOptedOut` (new shared custom function), draft

Signature: name `skipIfOptedOut`, return type `bool`, inputs `milestone` (string),
`patientId` (string), `leadId` (string), `clinician` (string).

Returns `true` when the person is opted out (the flow must not issue or send), `false`
otherwise. When it returns `true` it also writes the skip row (see §2) unless one already
exists and the milestone is not M3.

```text
bool skipIfOptedOut(string milestone,string patientId,string leadId,string clinician)
{
	hasPatient = patientId != null && patientId != "";
	hasLead = leadId != null && leadId != "";
	if(!hasPatient && !hasLead)
	{
		// Nothing to check. Fail open would send to someone we cannot look up, so stop the flow.
		throw "skipIfOptedOut: no patientId or leadId supplied";
	}
	// 1. Read the person's record live (not the stale trigger payload)
	if(hasPatient)
	{
		personRec = zoho.crm.getRecordById("Patients1",patientId.toLong(),Map(),"crm_connection");
	}
	else
	{
		personRec = zoho.crm.getRecordById("Leads",leadId.toLong(),Map(),"crm_connection");
	}
	if(personRec == null)
	{
		throw "skipIfOptedOut: person record could not be read";
	}
	optedOut = personRec.get("Feedback_Opt_Out");
	if(optedOut == null || optedOut != true)
	{
		return false;
	}
	// 2. Opted out. Write one skip row (de-duplicate everywhere except M3).
	writeRow = true;
	if(milestone != "3 - Periodic Consolidated")
	{
		if(hasPatient)
		{
			criteria = "((Milestone:equals:" + milestone + ") and (Status:equals:Skipped - Opted Out) and (Patient:equals:" + patientId + "))";
		}
		else
		{
			criteria = "((Milestone:equals:" + milestone + ") and (Status:equals:Skipped - Opted Out) and (Lead_Reference:equals:" + leadId + "))";
		}
		existing = zoho.crm.searchRecords("Milestone_Instances",criteria,1,1,null,"crm_connection");
		if(existing != null && existing.size() > 0)
		{
			writeRow = false;
		}
	}
	if(writeRow)
	{
		createMap = Map();
		createMap.put("Name","Milestone " + milestone + " - skipped (opted out)");
		createMap.put("Milestone",milestone);
		createMap.put("Status","Skipped - Opted Out");
		createMap.put("Clinician",clinician);
		if(hasPatient)
		{
			createMap.put("Patient",patientId);
		}
		if(hasLead)
		{
			createMap.put("Lead_Reference",leadId);
		}
		createResp = zoho.crm.createRecord("Milestone_Instances",createMap,Map(),"crm_connection");
		if(createResp.get("id") == null || createResp.get("id") == "")
		{
			throw "skipIfOptedOut: skip row create failed: " + createResp.toString();
		}
	}
	return true;
}
```

Notes:

- It fails closed: if the person cannot be read, it throws, the flow stops, and the existing
  On Error alert (feature 003) tells Costin. An unreadable record must never become a send.
  (The On Error branch on `issueFeedbackToken` does not cover this step, so each edited flow
  needs an On Error branch on `skipIfOptedOut` too; tasks T018 to T022.)
- `searchRecords` criteria with a space in the picklist value may need the value quoted or
  escaped; confirm with Execute (T011). If it does not match, fall back to searching by
  `Milestone` and `Patient` only and filtering Status in the loop.
- Principle VII: this is a boolean gate needed at trigger time, same justification as
  `isCreatedAfterCutoff` (plan.md Constitution Check).
- No recipient email is read, logged or alerted on.

## 4. `recordFeedbackOptOut` (new custom function), draft

Signature: name `recordFeedbackOptOut`, return type `map`, input `token` (string).

Returns `status` of `"recorded"` (set now), `"already"` (was already opted out, nothing
changed), or `"no_match"` (token empty or not found).

```text
map recordFeedbackOptOut(string token)
{
	resp = Map();
	if(token == null || token.trim() == "")
	{
		resp.put("status","no_match");
		return resp;
	}
	// Match by token only. Status and expiry are deliberately ignored (spec FR-006).
	criteria = "(Token:equals:" + token.trim() + ")";
	rows = zoho.crm.searchRecords("Milestone_Instances",criteria,1,1,null,"crm_connection");
	if(rows == null || rows.size() == 0)
	{
		resp.put("status","no_match");
		return resp;
	}
	row = rows.get(0);
	patientLookup = row.get("Patient");
	leadRef = row.get("Lead_Reference");
	if(patientLookup != null && patientLookup.get("id") != null)
	{
		moduleName = "Patients1";
		personId = patientLookup.get("id");
	}
	else if(leadRef != null && leadRef != "")
	{
		moduleName = "Leads";
		personId = leadRef;
	}
	else
	{
		resp.put("status","no_match");
		return resp;
	}
	personRec = zoho.crm.getRecordById(moduleName,personId.toLong(),Map(),"crm_connection");
	if(personRec != null && personRec.get("Feedback_Opt_Out") == true)
	{
		resp.put("status","already");
		return resp;
	}
	upd = Map();
	upd.put("Feedback_Opt_Out",true);
	upd.put("Feedback_Opt_Out_Date",zoho.currentdate.toString("yyyy-MM-dd"));
	upd.put("Feedback_Opt_Out_Via_Link",true);
	updResp = zoho.crm.updateRecord(moduleName,personId,upd,Map(),"crm_connection");
	resp.put("status","recorded");
	return resp;
}
```

Notes:

- It never modifies the Milestone_Instances row it found, so token status, expiry and any
  response are untouched (spec FR-006).
- Using `searchRecords` on `Token` avoids the 200-record linear scan used by the survey
  write-back. If `Token:equals:` does not search as expected on this text field, fall back to
  a looped `getRecords` like the write-back does, and note the 200 cap in the notes file.
- The return value contains no name, email or other identifier (spec FR-004).
- Idempotent: a second submission returns `already` and changes nothing (spec FR-006).

## 5. Zoho Forms: "Feedback Email Opt-Out"

| Setting | Value |
|---|---|
| Account | the same Zoho Forms account as the survey forms (`lianapreudhommecapec1`) |
| Fields | one hidden Single Line field `Token` (label and field alias `token`). Nothing else collects data. |
| Page content | a short plain-language explanation above the button (what will stop; that nothing else changes; how to reach the practice) |
| Submit button label | "Stop feedback emails" |
| Thank-you message | the single neutral message in research.md §4 (Liana to approve) |
| Prefill | Settings, Prefill, Field Alias, URL parameter `token` mapped to the Token field (see `m2-implementation-notes.md` §13: the alias, not the field name, must match) |
| Notifications | none to the submitter; no emails from Forms |
| Permalink | recorded in implementation-notes.md when created |

The link used in each email is `<form permalink>?token=${issueFeedbackToken_1.token}`.
Opening the link only displays the page. Nothing is recorded until the submit button is
pressed (spec FR-002).

## 6. Flow: "Feedback Opt-Out - Form Submitted"

```text
Trigger: Zoho Forms, "Form entry submitted" (realtime), form "Feedback Email Opt-Out"
   -> Custom function recordFeedbackOptOut(token = ${trigger.<Token field>})
   -> If else: recordFeedbackOptOut_1.status is "no_match"
        True  -> Zoho Mail "Send email" to costin@capeclarity.com:
                 subject "Feedback opt-out link did not match a record", body states only
                 that a submission arrived with no matching token. No token, no identity.
        False -> end
   On Error on the function step -> the same style of Zoho Mail alert (no ${trigger.*} values)
```

The alert body must not include the token or any `${trigger.*}` value, matching the rule
from feature 003.

## 7. Email additions (all five milestone emails)

One line added to the footer area of each "Send email" body, above the existing "This email
may contain information..." disclaimer. Draft wording, to be put through Liana's tone of
voice before use:

> If you'd rather not receive these feedback requests, you can <a href="{form permalink}?token=${issueFeedbackToken_1.token}">stop them here</a>, or simply reply to this email and let us know.

Rules: same link target in every milestone; nothing in the link but the token; the line
appears whether or not the milestone is the first email a person gets.
