# M2 (Session 3 / Early Alliance Check) — Implementation Notes

> Scope: this file documents ONLY the M2 milestone (the automatic Session 3 /
> Early Alliance Check feedback loop). It is a companion to
> `specs/001-feedback-collection-pipeline/spec.md` (all six milestones) and to
> `m2-plan.md` / `m2-research.md` / `m2-data-model.md` / `m2-tasks.md`. This file
> tracks what has actually been built in Zoho, as it's built, per the
> `CLAUDE.md` convention of updating implementation notes in the same session
> as the change.
>
> Last updated: 2026-09-11
> Status: T001 (Cape Clarity Alliance Check-In form), T002/T003/T004 (the
> "M2 - Session 3 Trigger" Zoho Flow, its idempotency-check custom function,
> and the parameterized "Call a subflow" step), and T005/T006 (the
> "M2 - Alliance Check-In Write-back" Zoho Flow and its
> `submitAllianceCheckInResponse` custom function) are built and saved. Both
> flows are currently **OFF** (not yet enabled) pending the same live-test-data
> confirmation M1 required. Not yet started: Analytics formula columns/
> reports/dashboard (T017-T019), and all live-test tasks (T020-T023, blocked
> on a test Patient ID from Costin). See §5 for the full remaining-work list.

## 1. What's built so far (T001)

**Cape Clarity Alliance Check-In** — Zoho Form, Standard type, built from
scratch via "New Form" → "Blank Form" (not via Zoho Forms' "Duplicate" action,
which turned out to be non-functional via browser automation in this
account — see §4). Two URLs matter for this form, and they are **not the
same thing** — a distinction this file got wrong in an earlier draft and is
correcting here:

- **Owner/builder-session URL** (what you land on navigating the form from
  "My Forms" while signed in): `https://forms.zoho.com/lianapreudhommecapec1/form/CapeClarityAllianceCheckIn`.
  This renders the full public respondent view when you're signed in, which
  made it easy to mistake for the real public link — but it is **not** what
  an unauthenticated patient's browser can load.
- **Formal public permalink** (Share tab → Public → "Form Permalink (URL)",
  confirmed via direct DOM read of the input's `.value`, not just eyeballing
  the rendered panel — see §4 gotcha on why): `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityAllianceCheckIn/formperma/bRYJ3IA6ocer84TfXo45HWPYMWgMZXIFkcAQ3rSZ3ZE`.
  **This is the value used for the `survey_url` parameter** in the "M2 -
  Session 3 Trigger" flow's "Call a subflow" step (§3 below) — same
  `zohopublic.com/.../formperma/<token>` shape as M1's Wellbeing Check-In
  permalink (`m1-implementation-notes.md` §4A). The Share tab also confirmed
  the public link is already **Enabled**.

### 1.1 Fields, in canvas order

| Order | Field type | Internal name (DOM-verified) | Label | Notes |
|---|---|---|---|---|
| 0 | Description | n/a (not a data-bearing field) | "Intro" (admin-only label) | Free-tier workaround for the Welcome Page paywall — see §1.2. Renders as plain text above the first slider. |
| 1 | Slider | `Slider` | "Overall, how comfortable have you felt being open and honest with your therapist so far?" | Domain: Connection. Instructions: "Domain: Connection. 0 = Not comfortable, 10 = Very comfortable." Range 0-10, Mandatory. |
| 2 | Slider | `Slider1` | "Overall, how well do you feel your therapist has understood what matters to you so far?" | Domain: Understanding. Instructions: "Domain: Understanding. 0 = Not understood, 10 = Fully understood." Range 0-10, Mandatory. |
| 3 | Slider | `Slider2` | "Overall, how much do you feel you and your therapist agree on what you're working toward?" | Domain: Shared direction. Instructions: "Domain: Shared direction. 0 = Not aligned, 10 = Fully aligned." Range 0-10, Mandatory. |
| 4 | Slider | `Slider3` | "Overall, how well has your therapist's approach been working for you so far?" | Domain: Fit of approach. Instructions: "Domain: Fit of approach. 0 = Hasn't worked, 10 = Worked well." Range 0-10, Mandatory. |
| 5 | Single Line | `SingleLine` | "Token" | Visibility set to **Hide**, matching the Wellbeing Check-In form's Token field pattern (`m1-implementation-notes.md` §4A's field table). Confirmed hidden in the live respondent-view render (no Token input visible on the public form; only in the builder canvas). |

**DOM verification done (2026-09-11)**: confirmed via the live public form's
own rendered DOM (`forms.zoho.com` respondent view — each field's wrapper
`<div>` carries `id="<InternalName>-li"`, same pattern M1's notes describe for
the builder canvas) rather than assumption. All five inferred names above
(`Slider`, `Slider1`, `Slider2`, `Slider3`, `SingleLine`) matched exactly —
no surprises. Safe to use directly in `submitAllianceCheckInResponse`'s
`${trigger.<InternalName>}` parameter mappings (T006).

### 1.2 Welcome/closing copy — paid-plan blocker and workarounds

`m2-research.md`'s planned instruction/closing copy assumed Zoho Forms'
built-in "Welcome Page" and "Rich Text" Thank You Page features, the same way
M1's form presumably could have used them (M1's own notes never mention
this — M1 apparently never customized either page). Neither is available on
this Zoho Forms account's current plan:

- **Welcome Page customization is a paid-plan feature.** Attempting to save
  welcome-page text produced an "Elevate the first impression... Upgrade to a
  paid plan to use this feature" modal, and the edit was silently discarded
  (confirmed via a follow-up "Changes... have not been saved" alert on
  navigating away). **Workaround**: added a free-tier **Description** field
  (found under Form Fields → Instructions → Description) as the form's first
  field, containing the exact intro copy from `m2-research.md` verbatim:
  > "You've been meeting with your therapist for a few sessions now, and
  > we'd love to hear how that's felt for you so far. There's no right or
  > wrong answer, just your honest sense of things. Thinking about your
  > experience with your therapist so far, not just today, mark where you'd
  > place yourself on each line."

  No wording change was needed here — the Description field has no
  character cap that bit, unlike the Thank You page (below).

  **Gotcha**: the Description field's rich-text editor is a same-origin
  `<iframe class="ze_area">` with no `src` (content set directly into its
  `contentDocument`), not a plain textarea/contenteditable in the main
  document. Typing into it via the `computer` tool's `type` action after a
  click landed correctly for every other field in this session but appended
  literal `"Add content..."` placeholder text in front of the real content
  here — the placeholder was not a CSS `::before` overlay, it was already
  real text inside the iframe's editable `<div>` that a plain click didn't
  select/clear. Fixed by reaching into the iframe directly via
  `javascript_tool`: `document.querySelector('iframe.ze_area').contentDocument`,
  setting the inner `<div>`'s `innerHTML` directly, then dispatching `input`
  and `keyup` events on it so the form builder's own JS picked up the
  change before clicking "Done". Worth checking for this same iframe-editor
  shape on any other Zoho Forms rich-text field in future milestones.

- **Thank You Page rich text is also paid** (crown icon next to "Rich Text"
  in Settings → Thank You Page & Redirection), but **Plain Text is free**,
  capped at 100 characters. `m2-research.md`'s planned closing line is 117
  characters — over the cap. **Shortened** (not the verbatim research
  copy) to fit, preserving the same meaning:
  > "Thank you for reflecting on this. It helps make sure you're getting the
  > right support." (86 characters)

  This is a real wording deviation from `m2-plan.md`/`m2-research.md`'s
  documented copy, not just a mechanical workaround — **flagging for
  Costin/Liana review**, the same way M1's notes flagged its own
  not-yet-reviewed email copy (`m1-implementation-notes.md` §4). If the
  shortened wording isn't acceptable, either accept the character loss with
  different phrasing, or revisit whether a paid-plan upgrade is worth it for
  Rich Text Thank You pages (would also unblock the Welcome Page feature and
  remove the need for the Description-field workaround above).

## 2. Zoho Forms plan-tier findings (useful for M3-M5)

The account is on Zoho Forms' **Free** subscription plan (confirmed via
Settings → Subscription plan showing "Free" with an "Upgrade" link). Two
features hit paywalls this session (Welcome Page customization, Thank You
Page Rich Text — see §1.2); everything else needed for M0-M2's forms so far
(Slider/Single Line fields, field Visibility, the free-tier Description field,
Thank You Page Plain Text) has been available on Free. Worth keeping in mind
for M3-M5 form design: don't plan on Welcome Page or Rich Text Thank You
pages without first confirming a plan upgrade, or budget for the same
Description-field / shortened-plain-text workarounds used here.

## 3. "M2 - Session 3 Trigger" flow, step by step (T002/T003/T004 — built)

Built by **cloning** "M1 - Session 1 Trigger" at the flow level (right-click
the flow in the flow list → Clone), rather than rebuilding the node graph
from scratch. This preserved the entire node layout and all 3 connector
wirings intact (confirmed via the DOM connector-element check below both
before and after edits) — a real time/risk savings over M1's from-scratch
build, given how much of `m1-implementation-notes.md` §5/§12 is devoted to
jsPlumb connector-wiring pain. **Caveat that made this more involved than
expected**: the cloned custom-function node is not an independent copy — see
§3.2's gotcha before repeating this shortcut on a future milestone.

1. **Trigger** — Zoho CRM "Updated module entry". Connection: CRM Connection.
   Module: `Patients` (internal API name `Patients1`). Filter: `Session Count`
   `equals` `3` (edited from the cloned flow's inherited `equals 1`; saved and
   confirmed persisted via the trigger's own "Last saved" timestamp and a
   re-open of its Filter criteria panel).
2. **Custom Function** — `checkAllianceCheckExists(patientId)`, a genuinely
   new, independently-created function (§3.2 explains why this matters),
   output variable `checkAllianceCheckExists_1` (boolean). Input
   `patientId = ${trigger.id}` (typed directly into the parameter field —
   this flow's trigger already had cached execution history from M1's clone
   lineage, so the Insert Variable panel did surface trigger fields, but
   `id` itself wasn't one of the listed field rows, so the M1-established
   `${trigger.id}` literal-expression pattern was used instead of clicking a
   panel entry).
3. **If else** — condition: `checkAllianceCheckExists_1` `is false`. Built via
   the same Insert Variable → search → click pattern M1 documented: clicking
   a variable in the right-hand panel only inserts it into the condition
   field that currently has *focus* (clicking the panel entry first does
   nothing but highlight it — the target field's dropdown/search box must be
   open first).
   - **True branch** (no existing Alliance Check-In record — proceed): "Call
     a subflow" → `Subflow - Issue Feedback Token`, "Wait and continue" mode.
     See §3.3 for parameter values.
   - **False branch** (record already exists): left empty, unchanged from the
     M1 clone — flow simply ends, no email, no new record. Same idempotency
     short-circuit as M1 (FR-002-equivalent for M2).

Verified via the DOM connector-element check
(`document.querySelectorAll('[class*="connector" i]').length`) after every
structural edit: **6** elements throughout the final build (3 wired
connections: trigger→function, function→if-else, if-else(true)→call-a-
subflow), matching M1's reference shape.

### 3.1 `checkAllianceCheckExists` — verbatim Deluge source

```
bool checkAllianceCheckExists(string patientId)
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
		if(patientRecId == patientId && r.get("Milestone") == "2 - Early Alliance Check")
		{
			found = true;
		}
	}
	return found;
}
```

Input: `patientId` (string). Return: `bool`. Connection name `crm_connection`
reused verbatim from M0/M1's pattern. Field-for-field identical to
`checkBaselineIntakeExists` (`m1-implementation-notes.md` §3.1) except the
function/parameter name and the `Milestone` match string
(`"2 - Early Alliance Check"` vs `"1 - Baseline Intake"`).

### 3.2 Gotcha (critical, new this session): cloned custom-function nodes are shared references, not copies

Zoho Flow custom functions are **shared objects across flows**, not
independently duplicated by either flow-level "Clone" (cloning the whole
"M1 - Session 1 Trigger" flow to create "M2 - Session 3 Trigger") or
step-level "Clone" (right-click a function node on the canvas → Clone,
creating a second node). Both cloning paths produce a function-call node that
still points at the **same underlying Deluge function object** as the
original — editing that function's code and clicking Save triggers a
confirmation dialog: *"The function is used in the following flows. Are you
sure you want to modify it?"*, listing every flow that references it (in this
case, both "M1 - Session 1 Trigger" and "M2 - Session 3 Trigger"). Confirming
would have silently rewritten M1's live production function — renaming it and
changing its `Milestone` match string — breaking M1's idempotency check.

This was caught before any shared-state corruption occurred, on two separate
attempts (direct in-place edit of the cloned node, then a step-level "Clone"
of that same node tested the same way) — both times the warning dialog was
treated as a hard stop: cancelled, then exited via "Exit without saving?" →
"Exit" (discards the unsaved edit). M1's function integrity was independently
re-verified afterward via direct `document.querySelector('.CodeMirror').CodeMirror.getValue()`
reads on the M1 flow, not just trusted from the UI.

**Fix / correct pattern going forward**: when cloning a flow whose canvas
includes a custom-function node that needs *different* logic in the new
flow, do not edit or step-clone that node. Instead:

1. Delete the cloned function node from the new flow's canvas (kebab menu →
   Delete) — including any stray step-level clone left over from a prior
   attempt.
2. Create a genuinely independent function via the **Built-ins panel**:
   left sidebar → Built-ins tab → Developer Tools → "Custom Functions"
   (collapsed by default — click the category header to expand, confirming
   `m1-implementation-notes.md` §5's note that this category needs a real
   click to expand before its contents, including the quick-create button,
   are usable) → the **"+ Custom Function"** button at the bottom of the
   expanded list. Clicking it (not dragging it — it isn't itself draggable;
   only the individual saved-function list items above it are, each marked
   `ui-draggable` in the DOM) opens a **"Create Function"** wizard: Function
   Name, Return Type, Input Parameters → Create → a fresh CodeMirror editor
   pre-seeded with just the function signature and empty body → write the
   body (the CodeMirror `setValue()` direct-API approach, not simulated
   typing, per the corruption gotcha `m1-implementation-notes.md` §5
   documents) → Save. This produces a function with no shared lineage to
   any other flow's node.
3. Drag the newly-saved function from the Built-ins → Custom Functions list
   onto the canvas to create its call node, then wire it in.

**Drag-and-drop mechanics gotcha (new this session)**: the individual
function list items under Built-ins → Custom Functions are jQuery UI
`ui-draggable` elements (confirmed via `[draggable="true"]` returning zero
matches but `.ui-draggable`/`.ui-droppable` classes present in the DOM — this
is **not** HTML5 native drag-and-drop). A single-jump `left_click_drag` (or
one synthetic `mousedown`→`mouseup` pair with no intermediate moves) does not
register as a drag with jQuery UI's threshold-based drag detection. **Fix**:
dispatch a real `mousedown` on the draggable element, then a series (10-20)
of incremental synthetic `mousemove` events walking from the start point to
the drop target, then a final `mousemove` + `mouseup` on the drop target
(`div#zf-builder-canvas.ui-droppable` / its child `div#flowServices`) — this
reliably placed the function node onto the canvas and opened its parameter-
mapping panel automatically. The same technique (real `mousedown` → stepped
`mousemove`s → `mouseup`, at the actual SVG `<circle>` endpoint coordinates
found via `document.elementFromPoint`) was reused to wire the function
node's output to the If-else node's input, exactly matching
`m1-implementation-notes.md` §5's already-documented connector-wiring
pattern — confirming that pattern generalizes beyond the specific case M1
first found it in.

**DOM-text-search caveat (new this session)**: `document.body.innerText` /
`.textContent` searches for on-canvas node label text (e.g. `"Updated module
entry"`) reliably returned empty/not-found in this Flow builder, even though
the same text is clearly visible in screenshots and the DOM does contain the
connector SVG elements the existing verification pattern already relies on.
Root cause not fully diagnosed (canvas node labels may be rendered in a way
`innerText`'s layout-visibility algorithm treats as hidden). **Practical
takeaway**: don't rely on text-content DOM queries to find or verify canvas
node content in this app — use `document.elementFromPoint(x, y)` plus
`getBoundingClientRect()` (which worked reliably throughout) or screenshots
instead. The connector-element-count check
(`[class*="connector" i]`) is unaffected by this and remains the reliable
wiring-verification method.

### 3.3 "Call a subflow" parameters (M2 → Subflow - Issue Feedback Token)

| Parameter | Value | Notes |
|---|---|---|
| `clinician` | `Liana Preudhomme` | Unchanged from the M1 clone — only non-`-None-` option on the `Clinician` picklist. |
| `survey_url` | `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityAllianceCheckIn/formperma/bRYJ3IA6ocer84TfXo45HWPYMWgMZXIFkcAQ3rSZ3ZE` | Confirmed formal Share-tab permalink (§1 above) — replaces the cloned-in M1 Wellbeing Check-In URL. |
| `email_intro_text` | "Thanks for continuing your sessions with us. We'd love to hear how things have been going with your therapist so far. Please take a moment to share your feedback using the link below:" | M2-specific; drafted, **not yet reviewed by Costin/Liana** (same caveat M1's notes carry for its own email copy). |
| `milestone` | `2 - Early Alliance Check` | Must match the CRM `Milestone` picklist value exactly (confirmed a clean existing value per `m2-data-model.md`). Replaces the cloned-in `1 - Baseline Intake`. |
| `patient_id` | `${trigger.id}` | Unchanged from the M1 clone — same pattern. |
| `recipient_email` | `${trigger.Email}` | Unchanged from the M1 clone — same pattern. |
| `ttl_days` | `7` | Unchanged from the M1 clone — reused per `m2-data-model.md`'s "reuse M0/M1's 7-day TTL unless Costin specifies otherwise" default. |
| `email_subject` | "How's therapy going so far? Quick check-in from Cape Clarity" | M2-specific; drafted, **not yet reviewed by Costin/Liana**. |
| `lead_id` | *(empty)* | Unchanged from the M1 clone — M2 is patient-based, not lead-based. |

**Field-editing gotcha (new this session)**: editing a parameter field that
already contains cloned-in text from M1 by clicking it and pressing
`ctrl+a` then typing did **not** reliably select-and-replace the existing
value in this panel — in practice the click landed mid-string, `ctrl+a`/
`Delete`/`Backspace` had no visible effect on the field's rendered content,
and the subsequently-typed text was silently inserted at the stale cursor
position, producing corrupted concatenations of old+new text (caught by
reading `document.activeElement.value` after the fact, not by trusting the
screenshot). **Fix**: set the field via the native input value setter
directly — `Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype,
'value').set.call(el, newValue)` — then dispatch `input` and `change`
events, targeting the element via `document.querySelector('input[name="..."]')`.
This reliably replaced the full value in one shot for every parameter field
in §3.3's table. Worth using as the default approach for editing any
already-populated Zoho Flow parameter field, not just ones cloned from
another milestone.

The flow was left **OFF** after this build (same as M1's own build-first,
verify-later approach) — not yet switched on, pending the same live-test-data
step T020-T023 already require.

## 3.4 "M2 - Alliance Check-In Write-back" flow, step by step (T005/T006 — built)

Built as a **brand-new flow** (Create flow → App trigger → configure), not by
cloning M1's write-back flow — deliberately, to avoid a repeat of §3.2's
shared-custom-function-across-clones gotcha. New flow name
"M2 - Alliance Check-In Write-back", placed in the "Customer Feedback System"
folder alongside the other 7 flows.

1. **Trigger** — Zoho Forms "Form entry submitted" (Realtime). Connection:
   "Connection to Cape Clarity Zoho Forms" (pre-selected by default). Form:
   "Cape Clarity Alliance Check-In". Output variable left as the default
   `trigger`. No filter criteria (mirrors M1's write-back trigger).
2. **Custom Function** — `submitAllianceCheckInResponse`, created from
   scratch via Built-ins → Developer Tools → Custom Functions →
   "+Custom Function" (the click-triggered quick-create wizard, per §3.2's
   documented mechanic — not cloned from any existing function). Return type
   `map`; five string input parameters: `token`, `connection`,
   `understanding`, `sharedDirection`, `fitOfApproach`. Full source in §3.4.1.
3. **Placing and wiring the function node**: dragged the saved function from
   the Built-ins sidebar list (a genuine jQuery UI `ui-draggable` item, same
   as §3.2) onto the canvas near the trigger. Unlike §3.2's build, this
   single drag-and-drop **both placed and auto-wired** the node to the
   trigger's output in one action — confirmed via the DOM connector-element
   count (`[class*="connector" i]` → 2 elements = 1 wired connection) both
   immediately after the drop and again after running "Auto Arrange" (the
   hierarchy icon in the canvas toolbar), which cleanly restacked the two
   nodes vertically (trigger on top, function below) without breaking the
   connection (count stayed at 2). A first attempt at the drag using a
   synthetic `mousedown`/stepped-`mousemove`/`mouseup` sequence dispatched
   via `javascript_tool` produced no visible effect (item never left the
   sidebar) even with added `mouseover`, `pageX`/`pageY`/`screenX`/`screenY`
   properties and small `await sleep(...)` delays between steps; what
   actually worked was the `computer` tool's built-in `left_click_drag`
   action with plain start/end coordinates — worth trying first before
   reaching for the manual synthetic-event approach in future milestones.
4. **Parameter mapping**: dropping the function node opened its parameter
   panel directly (no separate "configure" click needed). The Insert
   Variable panel's right-hand pane showed only "System Variables" plus a
   collapsed "Form entry submitted" trigger category with no field-level
   entries — consistent with §5's M1-era finding that a brand-new,
   never-run trigger's Insert Variable panel doesn't surface per-field
   variables yet. Typed the five `${trigger.<InternalName>}` expressions
   directly into the plain-text parameter fields instead, using the
   DOM-verified internal names from §1.1 (`SingleLine`, `Slider`, `Slider1`,
   `Slider2`, `Slider3`):

   | Function parameter | Value | Alliance Check-In field (friendly label) |
   |---|---|---|
   | `token` | `${trigger.SingleLine}` | Token (hidden) |
   | `connection` | `${trigger.Slider}` | Domain: Connection (0-10) |
   | `understanding` | `${trigger.Slider1}` | Domain: Understanding (0-10) |
   | `sharedDirection` | `${trigger.Slider2}` | Domain: Shared direction (0-10) |
   | `fitOfApproach` | `${trigger.Slider3}` | Domain: Fit of approach (0-10) |

   Each field's value was confirmed via `document.activeElement.value`
   immediately after typing (not trusted from the screenshot), per the
   verification discipline §4's last bullet documents.

### 3.4.1 `submitAllianceCheckInResponse` — verbatim Deluge source

```
map submitAllianceCheckInResponse(string token,string connection,string understanding,string sharedDirection,string fitOfApproach)
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
	responseText = "Connection (0-10): " + ifnull(connection,"") + "---";
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

Return type `map`. Five string inputs: `token`, `connection`, `understanding`,
`sharedDirection`, `fitOfApproach`. Connection name `crm_connection`, reused
verbatim from M0/M1's pattern. Directly mirrors
`submitWellbeingCheckInResponse` (`m1-implementation-notes.md` §4A.1)
field-for-field, adapted to the 4 Alliance Check-In domains. `Response_Data`
format matches `m2-data-model.md`'s Revision 2 exactly: four `---`-joined
domains, pure string concatenation, no numeric parsing or flag logic of any
kind (the Clinical Safety Flag and Flag Rule Triggered values are computed
entirely in Analytics formula columns per constitution Principle VII — not
this function's job, and not stored anywhere in `Milestone_Instances`).

The flow was left **OFF** after this build, same as "M2 - Session 3 Trigger"
— not yet switched on, pending the same live-test-data step T020-T023
already require.

## 4. Zoho Forms automation gotchas

- **"Duplicate" form action is non-functional via browser automation in this
  account.** Tried both a real coordinate-targeted `MouseEvent` click on the
  "Duplicate" dropdown option (after opening the form card's kebab menu,
  which did work) and calling `ZFForm.manager.showDuplicateForm(dupLi)`
  directly via JS (no error thrown, but no visible effect). No new form was
  ever created either way. **Not worth further automation-debugging time** —
  abandoned in favor of building the Alliance Check-In form from scratch via
  "New Form" → "Blank Form", which worked cleanly for a small 5-field form.
  Revisit only if a future milestone needs to clone a much larger form where
  from-scratch rebuilding would be materially more expensive.
- **The forms list/dashboard view can render blank** after enough JS
  poking around a stuck feature (seen after the failed Duplicate attempts,
  and again once this session after a plain `navigate` reload) — no form
  list, no modal, just an empty content pane. **Fix**: a hard `navigate`
  reload to the dashboard URL followed by clicking "My Forms" in the left
  sidebar (not just reloading) reliably restores the normal list view.
- **The field-panel search box does not filter results** when typing a
  field-type name (e.g. "Single Line") — the displayed category/field list
  stays unfiltered regardless of query text. Workaround: scroll the left
  panel manually and locate fields by visual inspection under their category
  headers (e.g. "Single Line" and "Multi Line" under "Textbox"; "Description"
  under "Instructions", well below "Rating Scales" — takes several scroll
  actions to reach from the top of the panel).
- **Drag-and-drop field placement is reliable here**, unlike Zoho Flow's
  jsPlumb canvas (`m1-implementation-notes.md` §5/§12, and §3.2 above,
  document that flow connections/node placement routinely fail to wire
  despite looking correct). Zoho Forms' field canvas uses jQuery UI
  draggable, and the `computer` tool's `left_click_drag` action placed every
  field correctly on the first attempt in this session, including dropping
  the Description field into a specific position (above the first Slider)
  rather than just at the canvas end. No special DOM-event-dispatch
  workaround was needed for form-field placement specifically — **this
  distinction does not carry over to the Flow build** (§3.2's stepped-
  mousemove technique was needed there instead).
- **Share tab's rendered panel can be effectively unreadable at the Browser
  pane's default width** (long labels/headings wrap to one character per
  line in a narrow right-hand column, making the actual Form Permalink field
  invisible on screen even though it's present in the DOM). **Fix**: use
  `Claude_Browser__resize_window` to emulate a wider viewport (1400x900
  worked) before reading Share-tab content, then reset to the `desktop`
  preset afterward. Reading the permalink input's `.value` directly via
  `javascript_tool` also works without resizing and is more reliable than
  screenshot-reading regardless.
- **Verification pattern used throughout this build**: rather than trusting
  screenshots or assuming a click landed where intended, every field-label/
  instructions/range/mandatory-checkbox edit in this session was confirmed by
  reading `document.activeElement.value` (or, for checkboxes, the specific
  `input[elname="field-ismandatory"]` element's `.checked`/`.offsetParent`
  state) via `javascript_tool` immediately after each `triple_click`/`type`
  action, before moving to the next field. This caught the Description-field
  iframe issue (§1.2) immediately rather than after a "Done"/"Save" click.
  The same discipline caught the parameter-field corruption gotcha in §3.3.

## 5. Remaining work (not yet built)

T001-T006 are done (§1, §3, §3.4). Not yet built: all Analytics work and all
live-test tasks. Per `m2-tasks.md` and `m2-data-model.md` (Revision 2 — flag
computed entirely in Analytics, per constitution Principle VII), the next
steps in order are:

- **T007-T016**: Verification/confirmation tasks (US1/US2/US4), including
  confirming with Costin whether the raw-Forms-entry retention-purge gap is
  still being deliberately deferred, and confirming no contractor-facing
  notification exists anywhere in the build.
- **T017**: Analytics formula columns on "Milestone Instances" — 3 bounded
  `substring_between` domain columns (Connection, Understanding, Shared
  Direction) + 1 guarded `SUBSTR`/`INSTR`/`LENGTH` column for Fit Of Approach
  (the unbounded last field, needs the zero-guard from
  `m1-implementation-notes.md` §9.3) + Alliance Check-In Total + Clinical
  Safety Flag + Flag Rule Triggered, exact formulas in `m2-data-model.md`
  ("Planned Analytics formula columns" / "Clinical Safety Flag evaluation").
- **T018-T019**: "M2 Submitted Responses" report, "M2 Status Breakdown", "M2
  Alliance Check-In Total Distribution", and the "M2 - Early Alliance Check
  Feedback" dashboard bundling them (mirror M1's §13 pattern).
- **"M2 Flagged for Review"** report: Tabular View filtered to
  `Milestone = "2 - Early Alliance Check"` AND `Clinical Safety Flag =
  "true"`.
- **T020**: **Blocked pending human input** — needs a test Patient record ID
  from Costin; the standing access restriction forbids looking one up via
  CRM query or browser (`CLAUDE.md` "Access constraints").
- **T021-T023**: Live test-data exercise and cleanup, depend on T020. Also
  covers switching "M2 - Session 3 Trigger" and the write-back flow **on**
  (both currently OFF) once live-tested.
- **T025-T026**: Update `CLAUDE.md` if the build reveals new cross-milestone
  conventions worth capturing (e.g. the Zoho Forms iframe-editor gotcha in
  §1.2, the plan-tier findings in §2, or the cloned-custom-function-sharing
  gotcha in §3.2 — the last of these seems especially likely to recur on
  M3-M5 if flow-cloning keeps being used as a shortcut, so it may be worth
  promoting to `CLAUDE.md` once a second milestone confirms it); commit this
  file and any further M2 corrections in the same session as the change,
  same git workflow as `CLAUDE.md`'s "Pushing to GitHub" section.
