# Data Model: Feedback Reminders

All code below is **draft**, not yet run. Execute each function in the Flow function editor
before any flow uses it (tasks T006 to T010), as with features 002, 004 and 006. Treat any
Deluge call marked "confirm" as unverified syntax.

## 1. CRM field (created 2026-10-09)

| Item | Value |
|---|---|
| Module | `Milestone_Instances` |
| Label | Reminder Sent Date Time (means "reminder dispatched"; see plan D3, open question 4) |
| API name | `Reminder_Sent_Date_Time` |
| Type | datetime |
| Field id | `6825601000004791001` |
| Meaning | Blank: no reminder dispatched. Set: one reminder was claimed for sending at that time. Never cleared by the pipeline. |
| Written by | `claimNextDueReminder` only |
| Contractor visibility | None. Contractors have no CRM access. If that changes, hide this field from their profile first (Principle IV). |

Related existing fields used, unchanged: `Status`, `Token`, `Email`, `Milestone`, `Patient`
(lookup), `Lead_Reference` (text), `Expiry_Date_Time`, `Created_Time`.

Not added: no new Status value, no tag, no skip row. Suppressed reminders leave no trace
(plan D4).

## 2. Eligibility, in one place

A row is **due** when all hold:

1. `Status` is `Issued`.
2. `Reminder_Sent_Date_Time` is empty.
3. `Expiry_Date_Time` is later than now (link still valid).
4. `Expiry_Date_Time` is at most **3 days** after now. This is "day 4 of 7" while the token
   lifetime is 7 days, and it is keyed off expiry because `Created_Time` is system-set and
   cannot be backdated for testing. If `ttlDays` ever changes, change `reminderDaysBeforeExpiry`
   to `ttlDays - 4` in the same edit.
5. The current time is between 9 AM and 5 PM Eastern.

A due row is **eligible** unless:

6. The person is opted out (`Email_Opt_Out` is true on `Patients1` or `Leads`, read live).
7. Milestone 5 and the patient's `Patient_Status` is no longer
   `Discontinued (Patient Choice)`.
8. Milestone 3 and the patient's immediately preceding feedback request was unanswered
   (section 3).
9. The person record cannot be read (counts as an error, row skipped, alert if nothing
   sendable).

All thresholds (4, 3, 9, 17) are named constants at the top of the function.

## 3. `claimNextDueReminder` (new shared custom function), draft

Signature: name `claimNextDueReminder`, return type `map`, no inputs.

Returns `{found: false}` when nothing is sendable. Otherwise
`{found: true, instanceId, token, email, milestone}`, after stamping
`Reminder_Sent_Date_Time` on that row.

```text
map claimNextDueReminder()
{
	result = Map();
	result.put("found",false);
	// ---- constants (policy, not validated values; spec Assumptions) ----
	reminderDaysBeforeExpiry = 3;
	windowStartHour = 9;
	windowEndHour = 17;
	// ---- 1. daytime window, US Eastern (confirm toString with a timezone) ----
	hourNow = zoho.currenttime.toString("HH","America/New_York").toLong();
	if(hourNow < windowStartHour || hourNow >= windowEndHour)
	{
		return result;
	}
	nowStr = zoho.currenttime.toString("yyyy-MM-dd'T'HH:mm:ssXXX");
	latestExpiryStr = zoho.currenttime.addDay(reminderDaysBeforeExpiry).toString("yyyy-MM-dd'T'HH:mm:ssXXX");
	// ---- 2. candidates, oldest first (COQL through the CRM connection; confirm scope) ----
	coql = "select id, Token, Email, Milestone, Patient, Lead_Reference, Created_Time, Expiry_Date_Time from Milestone_Instances where (Status = 'Issued' and Reminder_Sent_Date_Time is null and Expiry_Date_Time > '" + nowStr + "' and Expiry_Date_Time <= '" + latestExpiryStr + "') order by Expiry_Date_Time asc limit 25";
	body = Map();
	body.put("select_query",coql);
	resp = invokeurl
	[
		url :"https://www.zohoapis.com/crm/v6/coql"
		type :POST
		parameters:body.toString()
		connection:"crm_connection"
	];
	rows = resp.get("data");
	if(rows == null || rows.size() == 0)
	{
		return result;
	}
	errors = 0;
	for each  row in rows
	{
		milestone = row.get("Milestone");
		patientMap = row.get("Patient");
		hasPatient = patientMap != null && patientMap.get("id") != null;
		patientId = "";
		if(hasPatient)
		{
			patientId = patientMap.get("id").toString();
		}
		leadId = ifnull(row.get("Lead_Reference"),"");
		if(!hasPatient && leadId == "")
		{
			errors = errors + 1;
			continue;
		}
		// ---- 3. re-read the row: another run may have claimed it ----
		fresh = zoho.crm.getRecordById("Milestone_Instances",row.get("id").toLong(),Map(),"crm_connection");
		if(fresh == null || fresh.get("Status") != "Issued" || (fresh.get("Reminder_Sent_Date_Time") != null && fresh.get("Reminder_Sent_Date_Time") != ""))
		{
			continue;
		}
		// ---- 4. read the person live; unreadable = fail closed for this row ----
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
			errors = errors + 1;
			continue;
		}
		// 4a. opted out (feature 006): never remind
		if(personRec.get("Email_Opt_Out") == true)
		{
			continue;
		}
		// 4b. M5: only while still discontinued
		if(milestone == "5B - Discontinuation, Email Fallback" && personRec.get("Patient_Status") != "Discontinued (Patient Choice)")
		{
			continue;
		}
		// 4c. M3 fatigue brake: hold back if the immediately preceding request went unanswered
		if(milestone == "3 - Periodic Consolidated" && hasPatient)
		{
			all = zoho.crm.searchRecords("Milestone_Instances","(Patient:equals:" + patientId + ")",1,200,null,"crm_connection");
			thisCreated = row.get("Created_Time").toDateTime();
			prevCreated = null;
			prevStatus = "";
			prevExpiry = null;
			for each  r in all
			{
				st = r.get("Status");
				if(st == "Superseded" || st == "Skipped - Opted Out" || st == "Send Failed" || r.get("id").toString() == row.get("id").toString())
				{
					continue;
				}
				rc = r.get("Created_Time").toDateTime();
				if(rc < thisCreated && (prevCreated == null || rc > prevCreated))
				{
					prevCreated = rc;
					prevStatus = st;
					prevExpiry = r.get("Expiry_Date_Time");
				}
			}
			if(prevCreated != null)
			{
				unanswered = (prevStatus == "Expired") || (prevStatus == "Issued" && prevExpiry != null && prevExpiry.toDateTime() < zoho.currenttime);
				if(unanswered)
				{
					continue;
				}
			}
		}
		// ---- 5. claim: stamp first, send after (plan D3). Failure here stops the send. ----
		upd = Map();
		upd.put("Reminder_Sent_Date_Time",nowStr);
		updResp = zoho.crm.updateRecord("Milestone_Instances",row.get("id").toLong(),upd,Map(),"crm_connection");
		if(updResp.get("id") == null || updResp.get("id") == "")
		{
			throw "claimNextDueReminder: claim stamp failed";
		}
		result.put("found",true);
		result.put("instanceId",row.get("id").toString());
		result.put("token",row.get("Token"));
		result.put("email",row.get("Email"));
		result.put("milestone",milestone);
		return result;
	}
	if(errors > 0)
	{
		throw "claimNextDueReminder: " + errors + " candidate(s) could not be checked";
	}
	return result;
}
```

Notes:

- Fails closed per row: an unreadable person or a row with no person reference is skipped,
  never sent. If nothing else was sendable and any row errored, the function throws so the
  existing On Error alert fires. A persistently unreadable row can re-alert each run until its
  link expires (at most 3 days); accepted, and a reason to fix such rows promptly.
- Oldest-expiry-first, so the request closest to lapsing is reminded first.
- Principle VII: an action gate needed at send time, not a derived score; recorded in plan.md.
- Confirm in Execute (T007 to T010): the `toString` timezone form; the COQL `is null` on a
  datetime and the datetime literal format; that `crm_connection` has the COQL read scope; that
  `searchRecords` returns `Created_Time`; and that `zoho.currenttime.addDay` exists (else
  compute with `addMinutes(reminderDaysBeforeExpiry * 1440)`).
- No recipient email is logged or placed in any alert.

## 4. Flow: "Feedback Reminder - Send" (new, built OFF)

```text
Trigger: Schedule, hourly from 9 AM to 5 PM Eastern (if Flow cannot limit the hours,
         hourly all day; the function enforces the window)
  -> Custom function claimNextDueReminder()                          [output claimNextDueReminder_1]
       On Error -> Zoho Mail alert to costin@capeclarity.com (no identity, no token)
  -> If else: claimNextDueReminder_1.found is true
       Then -> Decision on claimNextDueReminder_1.milestone
                 "0 - No Conversion"                      -> Zoho Mail "Send email" (M0 reminder)
                 "2 - Early Alliance Check"               -> ... (M2)
                 "3 - Periodic Consolidated"              -> ... (M3)
                 "4 - Discharge"                          -> ... (M4)
                 "5B - Discontinuation, Email Fallback"   -> ... (M5)
               each Send email:
                 connection "Connection to info@capeclarity.com"
                 To   ${claimNextDueReminder_1.email}
                 link <that milestone's survey permalink>?token=${claimNextDueReminder_1.token}
                 plus the feature 006 stop-emails link built from the same token
                 On Error -> Zoho Mail alert to costin@capeclarity.com
                             (milestone and instanceId only; no email, no token; no retry)
       Else -> end
```

The milestone strings above are the live `Milestone_Instances.Milestone` picklist values as read
2026-10-09. The picklist still lists the dead `5A - Discontinuation, Live Capture` and `1 - Baseline
Intake` values and the `Captured Live` Status, although spec 001 says the live-capture values were
deleted; the reminder logic ignores them, and the Analytics `Excluded` bucket covers `Captured Live`. Build rules
from `CLAUDE.md` apply: connect every dropped node by hand and check `jsplumb-connected`;
hover the red "ON ERROR / DROP HERE" box during the drag for On Error branches. Do not
switch the flow ON before tasks T020 to T022.

A failed send does not touch the row's `Status` or `Token`: the original link stays valid.

## 5. Reminder email wording (DRAFT, for Liana's approval, FR-016)

Same structure for all five, placed through Liana's tone of voice before use
(`liana-tov-rewrite`). No name, no clinical content, no clinician reference. Each ends with
the feature 006 line. `{survey link}` is the milestone's own permalink with `?token=`.

| Milestone | Subject | Body (draft) |
|---|---|---|
| 0 | Still happy to hear from you | A few days ago we sent you a short note about your recent conversation with us. If you have two minutes, your answers help us improve things for the next person. The link below works for a few more days. [Share your thoughts]({survey link}) |
| 2 | A gentle reminder from Cape Clarity | We sent you a short check-in a few days ago and wanted to make sure it didn't get lost. It takes about two minutes and your honest answers really do shape how we work. [Open the check-in]({survey link}) |
| 3 | A gentle reminder: your quick check-in | Just a reminder about the short check-in we sent a few days ago. It's quick, it's private, and it helps us keep getting better. The link works for a few more days. [Open the check-in]({survey link}) |
| 4 | A gentle reminder: your feedback as you finish | We sent you a short note about your time with us a few days ago. If you'd like to share how it went, we'd be glad to hear it. [Share your feedback]({survey link}) |
| 5 | No pressure: a gentle reminder from Cape Clarity | This is the only reminder you'll get. If you're open to sharing what led you to step away, it would help us a great deal; if not, no problem at all. [Share your feedback]({survey link}) |

Footer on all, matching feature 006 section 7:

> If you'd rather not receive these feedback requests, you can <a href="{opt-out form permalink}?token=${claimNextDueReminder_1.token}">stop them here</a>, or simply reply to this email and let us know.

Rules: one reminder only (say so in M5 only; the others stay silent about it); the survey and
opt-out links carry nothing but the token; no deadline date is printed.

## 6. Analytics (computed in Zoho Analytics, Principle VII)

On the "Milestone Instances" table, after the new field syncs (daily at 5:00 PM EDT):

| Column | Definition (pseudo-formula; adapt to Analytics syntax at build) |
|---|---|
| `Reminded` | `Yes` when `Reminder Sent Date Time` is not empty, else `No` |
| `Response Outcome` | `Answered` if Status is `Submitted`; `Lapsed` if Status is `Expired`, or `Issued` with `Expiry Date Time` before now; `Open` if `Issued` and not yet expired; `Excluded` for `Superseded`, `Send Failed`, `Skipped - Opted Out` (and the dead `Captured Live`) |

New report **Reminder Effectiveness**: rows `Milestone`; columns `Reminded` by
`Response Outcome`; counts, plus an `Answered / (Answered + Lapsed)` rate. It satisfies spec
FR-015, and the baseline bar of 001 FR-018 (status/volume view plus a distribution) for this
feature. `Open` rows are excluded from the rate. Opt-out skips fall under `Excluded`, so they
never count as non-response (006 FR-010).

Caveat: the rate compares reminded with unreminded requests that differ in age at
measurement (reminded rows are by definition at least 4 days old). Compare only closed
(Answered plus Lapsed) rows, as above.
