# Data Model: Feedback Email Opt-Out

**Feature**: `006-feedback-email-opt-out` | **Date**: 2026-10-01

Everything below is a draft until built. Deluge in particular must be saved and run with
Execute in the function editor before any flow uses it (T009 to T012), as with features 002
and 004.

## 1. CRM fields: none added (native `Email Opt Out`)

Patients has no room for any new field of any type (Costin, 2026-10-02), so this feature adds
no CRM field. It uses the native **Email Opt Out** field (`Email_Opt_Out`, boolean) that already
exists on Leads and Patients1, together with its two read-only system fields. Verified on
PT000 on 2026-10-02 (implementation-notes.md):

| Native field | API name | What it gives us |
|---|---|---|
| Email Opt Out | `Email_Opt_Out` | The one flag every check reads. Writable through the API and Deluge. |
| Unsubscribed Time | `Unsubscribed_Time` | Opt-out date and time. Set by the CRM when the flag is ticked, whoever ticks it. Read-only. |
| Unsubscribed Mode | `Unsubscribed_Mode` | Shows `Manual` for both a UI tick and an API tick, so it does NOT tell link from staff. Not used. |

**Where the other facts live**

- **Opted out via the email link vs recorded by staff**: the record timeline entry for the
  change carries a `source`: `crm_api` when the function set it (link), `crm_ui` when staff
  ticked it. `recordFeedbackOptOut` also adds a short CRM note ("Feedback emails stopped via
  the email link") so staff can see it without opening the timeline. Staff add a note for phone,
  in-person and reply requests (staff-procedure.md).
- **Resubscribe**: unticking clears `Unsubscribed_Mode` and `Unsubscribed_Time` (verified), so no
  resubscribe date remains on the record. The timeline keeps both events with their times
  (`Email_Opt_Out` true to false, who, when). Staff also add a note when they resubscribe someone.
  Spec FR-015 ("discoverable") is met by the timeline plus the note.
- **Earlier opt-out after a resubscribe**: the timeline keeps every opt-out and resubscribe in order.

**Shared field, shared consequence.** `Email_Opt_Out` is a general CRM field: ticking it also
stops CRM mass email to that person. Feedback emails go out through the Zoho Mail step in
Flow, so they are unaffected by CRM's own suppression and this feature reads the flag itself.
Costin accepted this on 2026-10-02. Rules that follow:

- Campaigns is used with Leads only (Costin, 2026-10-02). Patients1 is not synced to Campaigns.
- On **Leads** the native flag is also what Campaigns syncs as "unsubscribed". A lead who ticks
  feedback opt-out is therefore also shown as unsubscribed in Campaigns, and a lead who
  unsubscribes from a Campaigns email will be skipped for M0 feedback. This coupling is real on
  Leads; Costin accepted this on 2026-10-02 (tasks T042).
- The Campaigns build must not copy `Email_Opt_Out` from a Patient into Email Audience.
- Staff must not use the flag for marketing preferences on Patients1 (staff-procedure.md).

Placement: no layout change (the field is already on the record). Not added to the Analytics
sync field list, as before.

No CRM workflow rule is needed: the date comes from the CRM itself.

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
	optedOut = personRec.get("Email_Opt_Out");
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
	if(personRec != null && personRec.get("Email_Opt_Out") == true)
	{
		resp.put("status","already");
		return resp;
	}
	upd = Map();
	upd.put("Email_Opt_Out",true);
	updResp = zoho.crm.updateRecord(moduleName,personId,upd,Map(),"crm_connection");
	// Best-effort note so staff can see how it arrived (does not affect the result if it fails)
	try
	{
		parentMap = Map();
		parentMap.put("module",{"api_name":moduleName});
		parentMap.put("id",personId);
		noteMap = Map();
		noteMap.put("Parent_Id",parentMap);
		noteMap.put("Note_Title","Feedback emails stopped via email link");
		noteMap.put("Note_Content","Opt-out recorded from the link in a feedback email on " + zoho.currentdate.toString("yyyy-MM-dd") + ".");
		zoho.crm.createRecord("Notes",noteMap,Map(),"crm_connection");
	}
	catch (e)
	{
		info "note not written";
	}
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
