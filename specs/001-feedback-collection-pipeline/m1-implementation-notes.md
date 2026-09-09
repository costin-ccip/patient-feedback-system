# M1 (Session 1 / Baseline Intake) — Implementation Notes

> Scope: this file documents ONLY the M1 milestone (the automatic Session 1 /
> Baseline Intake feedback loop). It is a companion to
> `specs/001-feedback-collection-pipeline/spec.md` (all six milestones) and to
> `m1-plan.md` / `m1-research.md` / `m1-data-model.md` / `m1-tasks.md`. This file
> tracks what has actually been built in Zoho, as it's built, per the CLAUDE.md
> convention of updating implementation notes in the same session as the change.
>
> Last updated: 2026-09-08
> Status: Phase 1 (Setup: T001 form, T002 schema confirm), Phase 2 Foundational
> (T003–T007, including the write-back flow) are all built and saved. Both
> flows ("M1 - Session 1 Trigger" and "M1 - Wellbeing Check-In Write-back") are
> still OFF/unpublished — no live test has been run yet, per Costin's explicit
> instruction to test M1 and re-verify M0 end-to-end only once all pieces are
> in place.

## 1. What M1 does

M1 fires automatically the first time a Patient's `Session_Count` reaches 1 (no
manual clinician action, unlike M0's manual trigger). It checks for an existing
`Milestone_Instances` record for that patient at Milestone "1 - Baseline Intake"
(idempotency — this has no M0 equivalent, since M0's manual trigger couldn't
double-fire the way a field-driven trigger can). If none exists, it calls the
shared `Subflow - Issue Feedback Token` (the same subflow M0 uses) to create the
`Milestone_Instances` record and email the patient a link to the Wellbeing
Check-In form.

De-identification: same model as M0, except M1 links to the patient via the
`Patient` lookup field on `Milestone_Instances` rather than a `Lead_Reference`
text field. (Earlier CLAUDE.md draft language forbade any automation from
populating/querying that lookup; Costin confirmed on 2026-09-07 that this lookup
points to a de-identified patient field, so the restriction was removed from
CLAUDE.md and does not apply to this build.

## 2. Component inventory

| Component | Type | Notes |
|---|---|---|
| Wellbeing Check-In Form | Zoho Form | T001. Public form, no patient identifier fields. Permalink: `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityWellbeingCheckIn/formperma/Qlvh_FeoU3Oof7fQz4WEeQBoFvQHYl2TLdNlQPON_Eg` |
| M1 - Session 1 Trigger | Zoho Flow | New flow, "Customer Feedback System" folder. Currently OFF. |
| `checkBaselineIntakeExists` | Custom Function (Deluge), scoped to the trigger flow | Idempotency check, linear-scan pattern matching M0's `submitFeedbackResponse` token lookup (see M0 notes §3.2) for consistency, not because it's technically superior. |
| Subflow - Issue Feedback Token | Zoho Flow (shared with M0) | Unchanged this session except the parameterization already completed for M0 (survey_url/email_subject/email_intro_text as explicit params). M1 reuses it as-is; no subflow changes were needed for M1 support — `patient_id`, `recipient_email`, `milestone`, `clinician`, `ttl_days`, `lead_id` were already parameters from when the subflow was originally generalized. |
| M1 - Wellbeing Check-In Write-back | Zoho Flow | T006. New flow, "Customer Feedback System" folder. Realtime Form-submission trigger on the Wellbeing Check-In form → `submitWellbeingCheckInResponse` custom function. Currently OFF. |
| `submitWellbeingCheckInResponse` | Custom Function (Deluge), scoped to the write-back flow | T007. Token lookup, `Status != "Issued"` rejection, expiry check + auto-expire, delimited `Response_Data` write-back, `Status -> "Submitted"`. Directly mirrors M0's `submitFeedbackResponse` (see `m0-implementation-notes.md` §3.1) field-for-field, adapted to the 5 Wellbeing Check-In domains instead of M0's survey questions. |

## 3. "M1 - Session 1 Trigger" flow, step by step

1. **Trigger** — Zoho CRM "Updated module entry". Connection: CRM Connection.
   Module: `Patients` (internal API name `Patients1`). Filter: `Session Count`
   `equals` `1`.
2. **Custom Function** — `checkBaselineIntakeExists(patientId)`, output variable
   `checkBaselineIntakeExists_1` (boolean). Input `patientId = ${trigger.id}`.
3. **If else** — condition: `checkBaselineIntakeExists_1` `is false`.
   - **True branch** (no existing Baseline Intake record — proceed): "Call a
     subflow" → `Subflow - Issue Feedback Token`, "Wait and continue" mode. See
     §4 for parameter values.
   - **False branch** (record already exists): left empty — flow simply ends,
     no email, no new record. This is the idempotency short-circuit (T004/FR-002).

### 3.1 `checkBaselineIntakeExists` — verbatim Deluge source

```
bool checkBaselineIntakeExists(string patientId)
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
		if(patientRecId == patientId && r.get("Milestone") == "1 - Baseline Intake")
		{
			found = true;
		}
	}
	return found;
}
```

Input: `patientId` (string). Return: `bool`. Connection name `crm_connection`
reused verbatim from M0's pattern.

## 4. "Call a subflow" parameters (M1 → Subflow - Issue Feedback Token)

| Parameter | Value | Notes |
|---|---|---|
| `clinician` | `Liana Preudhomme` | Reused from M0's value — only non-`-None-` option on the `Clinician` picklist. |
| `survey_url` | Wellbeing Check-In form permalink (§2) | |
| `email_intro_text` | "Thanks for starting your sessions with us. We'd love to hear how your first session went. Please take a moment to share your feedback using the link below:" | M1-specific; M0's equivalent was "We'd love to hear how things are going...". Drafted, not yet reviewed by Costin. |
| `milestone` | `1 - Baseline Intake` | Must match the CRM `Milestone` picklist value exactly (same pattern as M0's `0 - No Conversion`). |
| `patient_id` | `${trigger.id}` | Maps to the subflow's `Patient` lookup field via `${cf_trigger.patient_id}` in its "Create module entry" step. |
| `recipient_email` | `${trigger.Email}` | Patients module has an `Email` field (api_name `Email`), confirmed via `getFields` (schema only, no record data pulled) — same api_name as Leads, so the M0 pattern of `${trigger.Email}` carries over unchanged. |
| `ttl_days` | `7` | Reused M0's value verbatim, per the pending-tasks note "reuse M0's value unless Costin specifies otherwise." Not yet explicitly confirmed by Costin for M1. |
| `email_subject` | "How was your first session? Quick check-in from Cape Clarity" | M1-specific; drafted, not yet reviewed by Costin. |
| `lead_id` | *(empty)* | M1 is patient-based, not lead-based — subflow's `Lead Reference` field is left unpopulated for M1-originated records. |

## 4A. "M1 - Wellbeing Check-In Write-back" flow, step by step (T006/T007)

1. **Trigger** — Zoho Forms "Form entry submitted" (Realtime). Connection:
   "Connection to Cape Clarity Zoho Forms". Form: "Cape Clarity Wellbeing
   Check-In" (`CapeClarityWellbeingCheckIn`). Output variable left as the
   default `trigger`.
2. **Custom Function** — `submitWellbeingCheckInResponse`, dragged onto the
   canvas and connected to the trigger's output (see the connector gotcha in
   §5). Parameter mapping (typed directly as `${trigger.<FieldName>}`
   expressions — see §5 on why the Insert Variable panel couldn't be used
   here):

   | Function parameter | Value | Wellbeing Check-In field (friendly label) |
   |---|---|---|
   | `token` | `${trigger.SingleLine}` | Token (hidden) |
   | `personalWellbeing` | `${trigger.Slider}` | Personal wellbeing (0-10) |
   | `coping` | `${trigger.Slider1}` | Coping (0-10) |
   | `relationshipsSupport` | `${trigger.Slider2}` | Relationships and support (0-10) |
   | `hopeOutlook` | `${trigger.Slider3}` | Hope and outlook (0-10) |
   | `senseOfControl` | `${trigger.Slider4}` | Sense of control (0-10) |

   These internal field names were discovered by inspecting the Wellbeing
   Check-In form builder's DOM directly (`forms.zoho.com`, no PII, no edits
   made) rather than guessing: each field's wrapper `<div>` carries
   `id="<InternalName>-li"`. Zoho Forms names fields by type, incremented per
   field of that type in the order they were added to the form — not by
   custom field alias or visible label — hence `Slider`, `Slider1`, `Slider2`,
   `Slider3`, `Slider4` for the five 0-10 sliders in the order they appear on
   the form (Personal wellbeing, Coping, Relationships and support, Hope and
   outlook, Sense of control) and `SingleLine` for the hidden Token field.

### 4A.1 `submitWellbeingCheckInResponse` — verbatim Deluge source

```
map submitWellbeingCheckInResponse(string token,string personalWellbeing,string coping,string relationshipsSupport,string hopeOutlook,string senseOfControl)
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
	responseText = "Personal wellbeing (0-10): " + ifnull(personalWellbeing,"") + "---";
	responseText = responseText + "Coping (0-10): " + ifnull(coping,"") + "---";
	responseText = responseText + "Relationships and support (0-10): " + ifnull(relationshipsSupport,"") + "---";
	responseText = responseText + "Hope and outlook (0-10): " + ifnull(hopeOutlook,"") + "---";
	responseText = responseText + "Sense of control (0-10): " + ifnull(senseOfControl,"");
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

Return type `map`. Six string inputs: `token`, `personalWellbeing`, `coping`,
`relationshipsSupport`, `hopeOutlook`, `senseOfControl`. Connection name
`crm_connection`, reused verbatim from M0's pattern. `Response_Data` format
matches `m1-data-model.md` exactly: five `---`-joined domains, total NOT
stored in the blob (that's an Analytics formula column concern per
`m1-tasks.md` T015, not this function's job).

## 5. Gotchas discovered building M1 (new, beyond what M0 notes already cover)

- **A drag-and-drop that visually places a node next to the trigger does not
  necessarily connect it.** After dragging `submitWellbeingCheckInResponse`
  onto the canvas and configuring its parameter mapping (which saved fine —
  parameter mapping is configurable even on an unconnected node), the two
  nodes rendered overlapping rather than cleanly stacked like every other
  flow in this project, and `Auto Arrange` didn't fix the layout. Confirmed
  via DOM inspection that the two were genuinely **not connected**: a
  connected pair always has exactly 2 elements matching `[class*="connector"
  i]` in the DOM (an `svg.jsplumb-connector` + `path.jsplumb-connector-outline`
  — confirmed present on the M0 reference flow, and confirmed absent here
  before the fix). `window.jsPlumb.getConnections()` is not a reliable check
  in this app — it returned an empty array even after the connection was
  correctly made and persisted, presumably because the global `jsPlumb`
  object isn't the same instance the flow canvas's Ember component uses; the
  DOM connector-element check above is the reliable one.
  **Fix**: a real mouse drag from the trigger's `out` endpoint circle to the
  function's `in` endpoint circle is needed to wire them — dragging the
  *node* onto empty canvas near the trigger, as opposed to dragging directly
  onto/into the trigger's connector output point, does not auto-wire it.
  This was done by dispatching synthetic `mousedown` → several `mousemove`
  steps → `mouseup` MouseEvents at the endpoints' real CSS-pixel coordinates
  (via JS, not the coordinate-based click tool) — jsPlumb's own drag manager
  listens for standard mouse events on the endpoint elements and doesn't
  require a "trusted" OS-level event, so this worked on the first attempt.
  Verified by reloading the flow from scratch (hard reload, not just a
  client-side re-render) and re-checking the connector-element count.
- **Zoho Flow "Decision" branching block cannot reference custom function
  outputs in this account.** Its condition editor's field/value selectors and
  "Insert Variable" panel only ever showed System Variables (current
  date/datetime) — confirmed by typing the exact variable name, searching,
  reloading, and checking all three condition boxes. Typing `${...}` directly
  into the field is treated as a search query, not accepted as literal input.
  **Fix: use an "If else" logic block instead** — its Insert Variable panel
  correctly surfaces custom function outputs under a "Custom Functions"
  section (e.g. `checkBaselineIntakeExists_1`, typed `boolean`, clickable to
  insert). For a boolean output specifically, "If else"'s operator dropdown
  offers clean "Is true" / "Is false" options once the variable is inserted —
  no need to type a literal comparison value.
- **The "Insert Variable" panel only surfaces trigger field variables once
  the trigger has execution history (a cached schema).** A brand-new,
  never-run flow's Insert Variable panel shows only a "System Variables"
  section — no per-trigger category at all — unlike an already-exercised
  flow (e.g. M0's write-back), whose panel shows a full category of labeled
  field variables. **Fix/workaround**: this is not a blocker — it's the same
  pattern M0's own saved parameter values already use — type the expression
  directly into the plain-text parameter field as `${trigger.<InternalName>}`,
  using the internal field name discovered from the target form's builder DOM
  (see §4A). No UI limitation here, just nothing to click on for a new flow.
- **A custom function's code editor can be opened directly via its
  kebab-menu "View function" option**, without going through a node's
  configuration panel — useful when a node's own panel UI is otherwise
  awkward to drive via browser automation.
- **Sidebar categories under "Built-ins" (e.g. "Custom Functions" under
  "Developer Tools") are collapsed by default** (`display:none` on the
  category's `<ul>`) and need a genuine click on the category header to
  expand before any of its items (including the "Custom Function" quick-create
  button, the last `<li>` in the expanded list) can be dragged onto the
  canvas.
- **Coordinate-scaling bug in the Browser pane's `computer` tool**: for a
  chunk of this session, the `computer` tool's `coordinate` parameter (for
  click/hover/drag, when not using `ref`) was landing at roughly 1.4x the
  intended position relative to real CSS pixels from `getBoundingClientRect()`
  — diagnosed via `document.elementFromPoint()` mismatches. This affected
  category-expand clicks, dropdown selection, and drag-and-drop, and was
  worked around by scaling real-pixel targets by roughly 0.713 (x) / 0.712
  (y) before passing them to `coordinate`. This is an automation-environment
  quirk, not a Zoho Flow behavior — noted here only because it cost
  significant time diagnosing UI actions that looked like they should have
  worked. Direct JS-dispatched DOM events (`javascript_tool`) sidestep this
  entirely and were more reliable for anything precision-dependent, including
  the connector fix above.
- **Canvas drag-and-drop onto an existing connector can replace a node
  instead of inserting alongside it.** Dropping a new "Set Variable" step
  directly onto the connector between `checkBaselineIntakeExists` and the
  (at-the-time) Decision node deleted `checkBaselineIntakeExists` from the
  chain. Recovered via the canvas undo button. Dropping onto an empty output
  circle (rather than a connector between two already-connected nodes) did
  not have this problem.
- **Custom function editor bracket/autocomplete corruption**, same family as
  the M0-documented pipe-character bug: typing the full `checkBaselineIntakeExists`
  body in one `type` action left 2 stray closing `}` and inserted a literal
  `<opr> <expression>` autocomplete placeholder after `return found;`. Fixed by
  deleting the excess and retyping the final line in two pieces (`return found`
  without the semicolon, then `;` separately) to avoid re-triggering the
  snippet. Verify brace balance visually via a zoomed screenshot before moving on.
  For `submitWellbeingCheckInResponse` this whole class of corruption was
  avoided by writing the function body via direct CodeMirror API access
  (`document.querySelector('.CodeMirror').CodeMirror.setValue(code)`) instead
  of simulated typing — worth using as the default approach for any future
  custom function code, not just as a fallback.

## 6. Access constraint change (2026-09-07)

CLAUDE.md previously stated: "Never populate or query the `Patient` lookup
field on `Milestone_Instances` ... from any automation, report, or session."
This was written before M1 planning matured. When it blocked M1's core design
(the idempotency check queries `Patient`; T005 populates it), I stopped and
flagged the conflict to Costin rather than building around it. Costin
confirmed the lookup points to a de-identified patient field and had the
line removed from CLAUDE.md. No other access constraints changed — the
Leads/Patients browser-access restriction and the "no live tests without
Costin's involvement" rule both still stand.

## 7. Test/sample data approach

Not yet applicable — no live test has been run for M1. Both flows
("M1 - Session 1 Trigger" and "M1 - Wellbeing Check-In Write-back") are built
and saved but remain OFF/unpublished. Per Costin's explicit instruction, M1
testing (and M0 re-verification) will happen end-to-end once Costin runs the
coordinated test — not incrementally per task, and not initiated by this
session.

## 8. Remaining work (not yet built)

- T006/T007 are done (see §2, §4A) — both the write-back flow and its
  `submitWellbeingCheckInResponse` function are built, saved, and confirmed
  correctly wired (trigger connected to function; see the connector gotcha
  in §5).
- Phase 3+ (US1/US2 verification tasks, T008–T014, Phase 5 reporting T015–T016,
  Phase 6 test/validation T017–T020) all depend on Costin's deferred
  end-to-end live test and have not been started.
