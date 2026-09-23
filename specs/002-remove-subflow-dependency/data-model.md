# Data Model: Remove Subflow Dependency from Token Issuance

**Feature**: `002-remove-subflow-dependency` | **Date**: 2026-09-23

No CRM, Forms, or Analytics schema changes (spec FR-007). This file pins the record
shape the new path must reproduce and the draft code that reproduces it.

## Milestone_Instances record written at issuance (unchanged)

| Field (API) | Value | Source |
|---|---|---|
| `Name` | `"Milestone " + milestone + " - " + recipientEmail` | same string the subflow built |
| `Milestone` | `milestone` | caller |
| `Status` | `"Issued"` | fixed |
| `Token` | SHA-256 of time + 5 random 9-digit numbers | same algorithm as `generateFeedbackToken` |
| `Expiry_Date_Time` | now + `ttlDays`, `yyyy-MM-dd'T'HH:mm:ssXXX` | same |
| `Email` | `recipientEmail` | caller |
| `Clinician` | `clinician` | caller |
| `Patient` | `patientId` if non-empty, else not set | caller (M2, M3) |
| `Lead_Reference` | `leadId` if non-empty, else not set | caller (M0) |
| CRM triggers | workflow, approval, blueprint | matches "All the above" |

State transitions unchanged: any existing `Issued` record for the same person +
milestone → `Superseded` before the new record is created. Write-back functions
continue to move `Issued` → `Submitted` / `Expired`.

## `issueFeedbackToken` (new shared custom function) — draft

Signature (Create Function wizard): name `issueFeedbackToken`, return type `map`,
inputs `milestone` (string), `patientId` (string), `leadId` (string),
`recipientEmail` (string), `clinician` (string), `ttlDays` (int).

```text
map issueFeedbackToken(string milestone,string patientId,string leadId,string recipientEmail,string clinician,int ttlDays)
{
	hasPatient = patientId != null && patientId != "";
	hasLead = leadId != null && leadId != "";
	// 1. Supersede any still-Issued instance for this person + milestone
	//    (logic copied verbatim from supersedeOpenInstances)
	if(hasPatient)
	{
		criteria = "((Milestone:equals:" + milestone + ") and (Status:equals:Issued) and (Patient:equals:" + patientId + "))";
	}
	else
	{
		criteria = "((Milestone:equals:" + milestone + ") and (Status:equals:Issued) and (Lead_Reference:equals:" + leadId + "))";
	}
	openRecs = zoho.crm.searchRecords("Milestone_Instances",criteria,1,200,null,"crm_connection");
	supersededCount = 0;
	for each  rec in openRecs
	{
		supMap = Map();
		supMap.put("Status","Superseded");
		zoho.crm.updateRecord("Milestone_Instances",rec.get("id"),supMap,Map(),"crm_connection");
		supersededCount = supersededCount + 1;
	}
	// 2. Token + expiry (logic copied verbatim from generateFeedbackToken)
	seed = zoho.currenttime.toString() + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999);
	token = zoho.encryption.sha256(seed);
	expiry = zoho.currenttime.addDay(ttlDays).toString("yyyy-MM-dd'T'HH:mm:ssXXX");
	// 3. Create the Milestone Instance (same fields the subflow's native CRM step set)
	createMap = Map();
	createMap.put("Name","Milestone " + milestone + " - " + recipientEmail);
	createMap.put("Milestone",milestone);
	createMap.put("Status","Issued");
	createMap.put("Token",token);
	createMap.put("Expiry_Date_Time",expiry);
	createMap.put("Email",recipientEmail);
	createMap.put("Clinician",clinician);
	if(hasPatient)
	{
		createMap.put("Patient",patientId);
	}
	if(hasLead)
	{
		createMap.put("Lead_Reference",leadId);
	}
	triggerList = list();
	triggerList.add("workflow");
	triggerList.add("approval");
	triggerList.add("blueprint");
	opts = Map();
	opts.put("trigger",triggerList);
	createResp = zoho.crm.createRecord("Milestone_Instances",createMap,opts,"crm_connection");
	recordId = createResp.get("id");
	if(recordId == null || recordId == "")
	{
		// Halt the flow so the email step never sends a token with no record
		throw "issueFeedbackToken: Milestone Instance create failed: " + createResp.toString();
	}
	resp = Map();
	resp.put("status","success");
	resp.put("token",token);
	resp.put("expiry",expiry);
	resp.put("recordId",recordId);
	resp.put("superseded",supersededCount);
	return resp;
}
```

Per Principle VII: nothing here is a derived score/flag; it's issuance plumbing,
moved from one Flow construct (subflow) to another (custom function) with the same
logic. No Analytics duplication is introduced.

## Per-flow "Send email" step (Zoho Mail, native)

Identical in every caller except the three literals:

| Setting | Value |
|---|---|
| Connection | Connection to info@capeclarity.com |
| From | `info@capeclarity.com` |
| To | `${trigger.Email}` |
| Subject | milestone literal (below) |
| Body (HTML) | `<div>Hi,<br></div><div><br></div><div>{INTRO}<br></div><div><br></div><div>{SURVEY_URL}?token=${issueFeedbackToken_1.token}<br></div><div><br></div><div>Thank you,<br></div><div>Cape Clarity<br></div>` |

| Flow | Subject | Intro | Survey URL |
|---|---|---|---|
| M0 | A quick check-in from Cape Clarity | We'd love to hear how things are going. Please take a moment to share your feedback using the link below: | `https://forms.zohopublic.com/lianapreudhommecapec1/form/M0FreeConsultNonConversionSurvey/formperma/GFdd7kA1Yp8jIxhsMuTJTN1QDEiSl5K6FoQjceHsnAg` |
| M2 | How's therapy going so far? Quick check-in from Cape Clarity | Thanks for continuing your sessions with us. We'd love to hear how things have been going with your therapist so far. Please take a moment to share your feedback using the link below: | `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityAllianceCheckIn/formperma/bRYJ3IA6ocer84TfXo45HWPYMWgMZXIFkcAQ3rSZ3ZE` |
| M3 | Quick check-in: how's therapy going overall? - Cape Clarity | Thanks for continuing your work with us. As you keep making progress, we'd love to check in on how things have been going overall. Please take a moment to share your feedback using the link below: | `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityPeriodicCheckIn/formperma/1a3D0cvimvTXnchCSaZUI66bwR8Oj1BJe8Cs0JBY8W4` |

M2/M3 values are copied from 001 `m2-implementation-notes.md` §3.3 and
`m3-implementation-notes.md` §1.5 and must be re-read from the live "Call a subflow"
panel before it's deleted (T-level verification in tasks.md), in case the docs drifted.

## `issueFeedbackToken` call parameters per flow

| Param | M0 | M2 | M3 |
|---|---|---|---|
| milestone | `0 - No Conversion` | `2 - Early Alliance Check` | `3 - Periodic Consolidated` |
| patientId | *(empty)* | `${trigger.id}` | `${trigger.id}` |
| leadId | `${trigger.id}` | *(empty)* | *(empty)* |
| recipientEmail | `${trigger.Email}` | `${trigger.Email}` | `${trigger.Email}` |
| clinician | `Liana Preudhomme` | `Liana Preudhomme` | `Liana Preudhomme` |
| ttlDays | `7` | `7` | `7` |
