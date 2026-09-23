# 002 - Remove Subflow Dependency — Implementation Notes

> As-built record for `specs/002-remove-subflow-dependency`, per CLAUDE.md's
> same-session implementation-notes convention. Companion to `spec.md`, `plan.md`,
> `research.md` (which holds the retired subflow's verbatim internals),
> `data-model.md`, `contracts/issue-feedback-token.md`, `tasks.md`.
>
> **Changed by 003 (2026-09-23)**: `issueFeedbackToken`'s Name line now uses
> `token.subString(0,8)` instead of the recipient email, and each trigger flow has
> two On Error branches (alerts to costin@capeclarity.com; "Send Failed" marking).
> See `specs/003-issuance-privacy-and-failure-handling/implementation-notes.md`.
> The source in §3 below is otherwise current.
>
> Built: 2026-09-23. Status: **built and structurally verified; all flows OFF;
> live test pending (Costin)**. Nothing was switched on and no live/end-to-end test
> was run.

## 1. Plan confirmation (in-app)

- Zoho Store subscription: **Standard Plan, 5,000 tasks, yearly** (next payment
  2027-09-23).
- In the Flow builder on this plan, Built-ins → Logic lists only **Set Variable,
  Decision, Delay, If else**. There is no "Call a subflow" action, which confirms
  subflows aren't available on this plan (the public pricing page says otherwise).
  Custom Functions are available.

## 2. Component inventory (after this change)

| Component | Where | State |
|---|---|---|
| `issueFeedbackToken` | Zoho Flow custom function (workspace-shared) | New; used by M0, M2, M3 trigger flows |
| M0 - Lost Lead Feedback Token | Flow | Edited: `trigger → issueFeedbackToken → Send email (Zoho Mail)`; OFF |
| M2 - Session 3 Trigger | Flow | Edited: `trigger → checkAllianceCheckExists → If else (True) → issueFeedbackToken → Send email`; OFF |
| M3 - Periodic Check-In Trigger | Flow | Edited: `trigger → checkPeriodicCheckInDue → If else (True) → issueFeedbackToken → Send email`; OFF |
| [RETIRED] Subflow - Issue Feedback Token | Flow (ID `872426000000002154`, slug `subflow_issue_feedback_token`) | Renamed from "Subflow - Issue Feedback Token", description rewritten to say it's retired and point here; OFF; not deleted |
| `supersedeOpenInstances`, `generateFeedbackToken` | Custom functions | Untouched; still referenced only by the retired subflow |
| [RETIRED] M1 - Session 1 Trigger | Flow | Untouched; still contains a subflow call; OFF, must stay retired |
| All write-back flows, forms, CRM, Analytics | — | Untouched (M2 write-back re-checked: last saved Sep 11) |

## 3. `issueFeedbackToken` — verbatim source (re-read from the live editor)

Created via M3's builder: Built-ins → Developer Tools → Custom Functions →
"+ Custom Function". Wizard: name `issueFeedbackToken`, return type `map`, inputs
`milestone` string, `patientId` string, `leadId` string, `recipientEmail` string,
`clinician` string, `ttlDays` int. Body pasted with CodeMirror `setValue()`. Saved on
the first try ("We have successfully created the function"). **`throw` and the
4-argument `zoho.crm.createRecord(module, map, options, connection)` form were both
accepted**, so neither fallback in research.md §3/§4 was needed.

The editor re-indents on save (it strips the leading tab from top-level lines and
adds spaces after commas in the signature). Logic is identical to `data-model.md`:

```text
map issueFeedbackToken(string milestone, string patientId, string leadId, string recipientEmail, string clinician, int ttlDays)
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

Not executed (the panel's "Execute" button would create a real CRM record, so it's
off-limits without Costin). Whether `throw` actually halts the flow before the email
step is confirmed only by the live test.

## 4. Per-flow configuration (read back from each node after reload)

Output variable renamed to **`issueFeedbackToken_1` in all three flows** (defaults
were `_3`/`_3`/`_1`), so the email body's merge field is identical everywhere.

| | M0 | M2 | M3 |
|---|---|---|---|
| milestone | `0 - No Conversion` | `2 - Early Alliance Check` | `3 - Periodic Consolidated` |
| patientId | *(empty)* | `${trigger.id}` | `${trigger.id}` |
| leadId | `${trigger.id}` | *(empty)* | *(empty)* |
| recipientEmail | `${trigger.Email}` | `${trigger.Email}` | `${trigger.Email}` |
| clinician | `Liana Preudhomme` | `Liana Preudhomme` | `Liana Preudhomme` |
| ttlDays | `7` | `7` | `7` |

**Send email (Zoho Mail)** in each flow: connection "Connection to
info@capeclarity.com"; From `info@capeclarity.com`; To `${trigger.Email}`;
Reply-to/CC/BCC empty; variable name left at default (`sendEmail_2` M0, `sendEmail_4`
M2/M3; nothing references it). Subjects and HTML bodies exactly as in
`data-model.md` (body merge field `${issueFeedbackToken_1.token}`), verified by
reading the saved `text` field back after a full page reload.

Before deleting each "Call a subflow" node, its live parameters were re-read; all
three matched `data-model.md` (M2/M3 matched their 001 notes; M0 matched
research.md §0).

**Wiring**, confirmed after reload: every `b.in`/`b.out` endpoint in the chain carries
`jsplumb-connected`; hovering the If-else → issueFeedbackToken line in M2 and M3 shows
the **"True"** label. Canvas node lists (from the builder's hidden `input[name=delay]`
elements): M0 `issueFeedbackToken`; M2 `checkAllianceCheckExists, issueFeedbackToken`;
M3 `checkPeriodicCheckInDue, issueFeedbackToken`, plus the Zoho Mail node in each.

**Shared-function check (T018)**: opening `issueFeedbackToken` via "Edit function"
and clicking Save showed *"The function is used in the following flows"* listing 3
flows; cancelled, then exited without saving (Last saved time unchanged). The dialog
labels them "M0 - Lost Lead Feedback Token", **"M2 - Alliance Check-In Write-back"**,
"M3 - Periodic Check-In Trigger". The M2 label is wrong: the M2 write-back flow was
opened and confirmed untouched (trigger + `submitAllianceCheckInResponse` only, last
saved Sep 11). The dialog seems to show a stale name for "M2 - Session 3 Trigger",
probably from its clone lineage (001 m2 notes §3). Treat that dialog's flow names as
unreliable; the count is right.

## 5. Gotchas found this session (new)

1. **Dropping a node onto an If-else branch endpoint places it but does NOT wire
   it.** In M3 the function node landed next to the diamond looking attached (same
   position the old subflow node had), but the If-else `b.out` had no
   `jsplumb-connected` class and no connector line existed. The old
   `[class*="connector" i]` count didn't catch this (it counted 4 both ways). **Check
   the `jsplumb-connected` class on each `b.in`/`b.out`**, and hover the branch line
   for its "True"/"False" label. Fix: drag the node clear of the diamond (stepped
   synthetic drag on the node's icon), then drag from the If-else `div.jsplumb-endpoint`
   (right side = True) to the node's input endpoint. The synthetic drag may leave the
   connection half-drawn and time out the JS call; a real mouse hover + click on the
   target input endpoint completes it. Dropping onto a *plain action's* output
   endpoint (M0 trigger, and each function → Send email) wired correctly.
   **Implication for 001**: M3's notes (§1.5) reported its "Call a subflow" node as
   wired on the strength of a connector count; that may never have been true. Moot
   now, but the same check should be applied to any other If-else branch built that way.
2. **Custom-function parameter fields are CodeMirror chip editors.** Typing
   `${trigger.id}` turns `${` into an empty chip and corrupts the value. Reliable:
   `document.querySelector('#zf-chip-parent-<param> .CodeMirror').CodeMirror.setValue(v)`,
   focusing it first and then focusing another input, which syncs the hidden
   `input[name=<param>]`. The same works for Zoho Mail's From/To/Subject
   (`fromAddress`, `toAddress`, `subject`).
3. **Zoho Mail body**: click the editor's `</>` (source) toggle, set
   `textarea#html_text` with the native value setter + `input`/`keyup`/`change`
   events, toggle `</>` back; the saved value is in `textarea[name=text]`.
4. **CSS vs screenshot coordinates differ** (CSS viewport 1699 px wide vs a 1568 px
   screenshot frame, DPR 2). Coordinates from `getBoundingClientRect()` must be
   scaled (~0.923) before use with screenshot clicks. `scrollIntoView()` inside the
   builder scrolls the whole page and shifts everything; scroll the
   `.sidebarMenuList` container instead.
5. **Built-ins has its own "Send Email" (Notification)**, which is Zoho Flow's
   own mailer, not Zoho Mail. Use Apps → Zoho Mail → "Send email" (the connection
   field showing "Connection to info@capeclarity.com" confirms the right one).
6. **Badge after edits**: M0 and M2 showed **"Paused"** before editing and **"Draft"**
   after; M3 showed no badge before or after. The flow-list "Updated On" dates did not
   change. What "Draft" means for the live version when a flow is switched on wasn't
   testable without switching it on. **When Costin turns these on, confirm the
   running version is the new one** (the first live run's execution should show
   issueFeedbackToken + Send email, not "Call a subflow"; a run that tries the old
   subflow would fail, since the subflow is OFF).
7. Opening a node config panel and cancelling bumps "Last saved" (research.md §5.3);
   opening "Edit function" and cancelling out of the save-confirmation does not.

## 6. What's left

- Costin: live test per `quickstart.md` §B (folded into the coordinated M0/M2/M3
  test), including confirming gotcha 6.
- Costin: decide on research.md §5.1 (recipient email in the record Name and
  Analytics) and §5.2 (orphan record on mail failure).
- M4/M5: build their triggers with `issueFeedbackToken` + their own Send email step
  (CLAUDE.md convention).
