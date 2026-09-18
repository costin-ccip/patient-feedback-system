# M3 (Periodic Consolidated Check-In, every 8th session) — Implementation Notes

> Scope: this file documents ONLY the M3 milestone (the recurring
> every-8th-session Periodic Consolidated Check-In). It is a companion to
> `specs/001-feedback-collection-pipeline/spec.md` (all six milestones) and to
> `m3-plan.md` / `m3-research.md` / `m3-data-model.md` / `m3-tasks.md`. This
> file tracks what has actually been built in Zoho, as it's built, per the
> `CLAUDE.md` convention of updating implementation notes in the same session
> as the change.
>
> Last updated: 2026-09-18
> Status: **Both M3 Zoho Flow flows now built. Pausing for Costin's
> check-in before starting any Analytics/reporting work, per his standing
> instruction.** Done so far: T014 (Clinical Safety Flag `Milestone` gate
> widened to cover M3), T017 (3 new M3-only Analytics formula columns), the
> title-only half of T016 (report renamed "M2 Flagged for Review" → "M2 &
> M3 Flagged for Review"; its Milestone filter is **not yet widened** —
> see §2.3), **T001** (the "Cape Clarity Periodic Check-In" Zoho Form —
> see §1.4; the Forms-plan blocker described in §3 was resolved 2026-09-18
> when Costin subscribed to a paid Zoho Forms plan), **T002-T004** (the
> "M3 - Periodic Check-In Trigger" flow — trigger, `checkPeriodicCheckInDue`
> custom function, if-else, and "Call a subflow" step — built fresh, wired
> end-to-end, and verified; left switched **off** — see §1.5), and
> **T005/T006** (the "M3 - Periodic Check-In Write-back" flow — trigger,
> `submitPeriodicCheckInResponse` custom function, all 8 parameters mapped
> to the form's field internal names — built fresh, wired end-to-end, and
> verified; left switched **off** — see §1.6). T018/T019 (the two
> remaining M3 reports + dashboard) have not been started. T020-T023 (live
> test data) remain explicitly blocked pending a test Patient ID from
> Costin, per standing instruction. **Both M3 flows (trigger and
> write-back) are now built, so per Costin's standing instruction this
> session stops here and checks in with him before touching any
> Analytics/reporting work (T016's remaining filter widening, T018, T019,
> the M3 dashboard).** See §4 for the full remaining-work list.

## 1. What's built so far

Nothing CRM- or Forms-side has been built for M3 itself yet (no new form, no
new flows, no new custom functions, no new CRM records). What exists so far
is entirely in the shared Zoho Analytics "Milestone Instances" table
(workspace `3251423000000083002`, view `3251423000000083317`) — the same
shared table M0/M1/M2 use, per `m3-data-model.md`'s "no new CRM fields"
design.

### 1.1 "Clinical Safety Flag" formula column widened (T014)

Per `m3-data-model.md`'s Decision 5 (share, don't fork, the M2 alliance
rule), the existing "Clinical Safety Flag" formula column's `Milestone` gate
was widened from M2-only to M2-or-M3. **Deviates from `m3-data-model.md`'s
drafted formula in one respect**: the drafted version used `IF(OR(a, b),
...)` — Zoho Analytics' formula parser rejected this with "Parsing of given
formula failed... Encountered: OR" (confirmed this session; `OR` is an
**infix** operator here, not a function callable as `OR(a,b)` — a new
Analytics-formula-language finding not documented in any prior milestone's
notes). Fixed by rewriting with infix `OR`, which saved successfully. Final,
live formula (verbatim, re-read via right-click → "Edit Formula Column"):

```
IF("Milestone Instances"."Milestone" = '2 - Early Alliance Check' OR "Milestone Instances"."Milestone" = '3 - Periodic Consolidated', IF("Milestone Instances"."Alliance Check-In Total" <= 20 OR to_integer("Milestone Instances"."Domain: Connection") <= 4 OR to_integer("Milestone Instances"."Domain: Understanding") <= 4 OR to_integer("Milestone Instances"."Domain: Shared Direction") <= 4 OR to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, 'true', 'false'), '')
```

Logically identical to the drafted version — only the `OR` syntax changed,
not the condition structure. Verified M2's existing test case (row 1) still
evaluates to `false` as before the edit, so this did not regress M2's
already-shipped behavior. `m2-implementation-notes.md` §9.2 has been updated
to match (see that file's changelog note under §9.2).

### 1.2 Three new M3-only Analytics formula columns (T017)

Added to "Milestone Instances", all bounded `substring_between` (each
segment is followed by another `---`-delimited segment in M3's blob order,
per `m3-data-model.md`'s field-order design — none of these needs the
unbounded-last-field guard pattern "Domain: Fit Of Approach" uses):

- **Practice Experience: Scheduling/Communication** —
  `substring_between("Milestone Instances"."Response Data", 'Practice Experience: Scheduling/Communication (0-10): ', '---', 1)`
- **Practice Experience: Billing** —
  `substring_between("Milestone Instances"."Response Data", 'Practice Experience: Billing (0-10): ', '---', 1)`
- **Therapist Professionalism** —
  `substring_between("Milestone Instances"."Response Data", 'Therapist Professionalism (0-10): ', '---', 1)`

All three saved exactly as drafted in `m3-data-model.md` — no parse errors,
no deviations. Confirmed present via column-header enumeration on the table.
Not yet exercised against a real M3 submission (no M3 data exists yet —
T001/T005/T006 aren't built), so these are structurally verified only, per
T017's own "confirm against a real test submission before assuming it"
instruction — that confirmation is still pending on T020-T022.

The four reused alliance-domain columns (Domain: Connection/Understanding/
Shared Direction/Fit Of Approach) and Alliance Check-In Total were **not
modified** — per `m3-data-model.md`'s Decision 4, M3's blob deliberately
puts the 4 alliance segments last, in the same order and with the same
label text M2 uses, so these columns should parse M3 rows with zero edits.
This is unverified against real data for the same reason as above.

### 1.3 "M2 & M3 Flagged for Review" report — renamed, filter NOT yet widened (T016, partial)

The existing "M2 Flagged for Review" report (view ID
`3251423000000141219`) was renamed to "M2 & M3 Flagged for Review" this
session — confirmed via screenshot of the report showing the new title.
Renaming required a workaround: a first attempt (double-click title →
Ctrl+A → type) produced a corrupted concatenation ("M2 FlaggedM2 & M3
Flagged for Reviewfor Review") — the same class of text-field corruption
bug `m2-implementation-notes.md` §3.3 documents for Zoho Flow parameter
fields, now confirmed to also occur in Zoho Analytics' report-title field.
Fixed the same way §3.3 does: set the value via the native input setter
(`Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype,
'value').set.call(el, 'M2 & M3 Flagged for Review')`, dispatch an `input`
event, then press Return) — confirmed correct via a follow-up screenshot.

**The Milestone filter itself has NOT been widened yet.** Per
`m3-data-model.md`, the filter needs to become `Milestone` Wildcard Exactly
Matches `"2 - Early Alliance Check"` **OR** `"3 - Periodic Consolidated"`
(currently still M2-only, per `m2-implementation-notes.md` §9.6's original
build). At the point this file was written, the correct UI path to reach
this report's saved filter-criteria editor had not yet been found:

- The toolbar's **"Filter"** button opens only an ad-hoc per-column
  quick-filter row (Is/Contains dropdowns per visible column) — not the
  report-level saved filter panel `m2-implementation-notes.md` §9.6/§9.8
  describes (a "Filters (N)" count with Wildcard/Individual Values tabs).
- The toolbar's **"More"** dropdown only offers "Show/Hide Column", "Freeze
  Column", and "Wrap header" — also not it.
- Not yet tried at the point this was written: the "Edit Design" control
  (top-right of the report view). This is the most likely remaining
  candidate and is the natural next step once browser access to this
  Analytics workspace is available again (the browser session this work was
  using disconnected mid-task — see §4).

**Practical effect of the gap**: the report currently still shows only M2
rows (title says M2 & M3, filter still says M2-only). No M3 data would
appear in it even once M3 starts producing flagged rows, until this filter
is corrected. Flagging clearly rather than leaving it ambiguous from the
title alone. `m2-implementation-notes.md` §9.6 has been updated to note
both the rename and this open gap (see that file).

### 1.4 "Cape Clarity Periodic Check-In" Zoho Form built (T001)

Built 2026-09-18, after Costin subscribed to a paid Zoho Forms plan (see §3
— this removed the blocker that had stopped T001 the prior session). Public
form URL:

```
https://forms.zoho.com/lianapreudhommecapec1/form/CapeClarityPeriodicCheckIn
```

(builder URL: `.../form/CapeClarityPeriodicCheckIn/builder`.) 8 fields, in
on-screen order, exactly matching `m3-data-model.md`'s section order
(Alliance Check-In, then Practice Experience, then Therapist
Professionalism):

1. **Slider — Connection** (reused verbatim from M2): "Overall, how
   comfortable have you felt being open and honest with your therapist so
   far?" / "Domain: Connection. 0 = Not comfortable, 10 = Very comfortable."
   / range 0-10 / Mandatory.
2. **Slider — Understanding** (reused verbatim from M2): "Overall, how well
   do you feel your therapist has understood what matters to you so far?" /
   "Domain: Understanding. 0 = Not understood, 10 = Fully understood." /
   range 0-10 / Mandatory.
3. **Slider — Shared direction** (reused verbatim from M2): "Overall, how
   much do you feel you and your therapist agree on what you're working
   toward?" / "Domain: Shared direction. 0 = Not aligned, 10 = Fully
   aligned." / range 0-10 / Mandatory.
4. **Slider — Fit of approach** (reused verbatim from M2): "Overall, how
   well has your therapist's approach been working for you so far?" /
   "Domain: Fit of approach. 0 = Hasn't worked, 10 = Worked well." / range
   0-10 / Mandatory.
5. **Slider — Practice Experience: Scheduling/Communication** (new,
   drafted in `m3-research.md` Decision 3, **not sourced from Confluence —
   needs Costin/Liana review before this form goes live**, same as M1/M2's
   precedent for drafted-not-sourced prompts): "How satisfied have you been
   with scheduling and communication with our practice (not your therapist
   directly)?" / instructions "Domain: Scheduling/Communication. 0 = Not
   satisfied, 10 = Very satisfied." (this instructions line was not in
   `m3-research.md` — written this session to match M2's "Domain: X. 0 =
   ..., 10 = ..." caption style; flag for review alongside the prompt
   itself) / range 0-10 / Mandatory.
6. **Slider — Practice Experience: Billing** (new, per `m3-research.md`
   Decision 3, same not-sourced-from-Confluence caveat): "How satisfied have
   you been with the billing/insurance process?" / "Domain: Billing. 0 = Not
   satisfied, 10 = Very satisfied." / range 0-10 / Mandatory.
7. **Slider — Therapist Professionalism** (new, per `m3-research.md`
   Decision 3, same not-sourced-from-Confluence caveat): "How would you rate
   your therapist's professionalism (punctuality, preparedness, respectful
   conduct)?" / "Domain: Professionalism. 0 = Very poor, 10 = Excellent."
   (again, this exact instructions wording was written this session, not
   drafted in `m3-research.md`) / range 0-10 / Mandatory.
8. **Single Line — Token**: label "Token", Visibility = Hide. Same
   hidden-prefill-token pattern as M0/M1/M2's forms.

No name, email, or phone field, per Principle I. Confirmed via the live
public form (not just the builder) that all 7 sliders render in the correct
order and the Token field does not appear to a respondent.

**Field-order gotcha worth flagging for future milestone builds**: Zoho
Forms' drag-and-drop palette does not always drop a new field at the end of
the canvas — dropping near the bottom of the last existing field sometimes
inserted the new field mid-canvas instead (observed here: after configuring
slider 1, dropping a second blank slider from the palette landed it
*between* slider 1 and the field being configured next, not after it,
producing an out-of-order canvas: Connection, [blank], Understanding, Shared
direction, [blank] x3). Fixed by deleting the misplaced blank field (safe
before it's configured — no data loss) and re-adding it via drag-and-drop
into a position that put it, and 3 further re-additions, into a contiguous
run of 4 blanks immediately after "Shared direction" and before "Single
Line" (verified by scrolling the full canvas and reading field order
visually, not by DOM queries — the builder's canvas lives inside a
cross-origin iframe not readable via `javascript_tool`, and Zoho Forms
fields also don't respond to CSS selectors keyed on generic list-item
patterns like `[id$="-li"]`, per earlier sessions' notes on this builder).
Existing fields can also be reordered by a plain drag on the field's own
body (no separate drag handle) — used once here to move a misplaced blank
slider from after "Single Line" back to before it, after a new-field drop
landed in the wrong spot a second time. **Always do a full top-to-bottom
visual scroll of the canvas (or the live published form) to confirm field
order after any drag-and-drop addition**, rather than trusting the order
fields were dropped in.

### 1.5 "M3 - Periodic Check-In Trigger" flow built (T002-T004)

Built 2026-09-18, entirely from scratch — **not** cloned from M2's "M2 -
Alliance Check-In Trigger" flow, per `m2-implementation-notes.md` §3.2's
shared-custom-function gotcha (cloning a flow, or a single node within it,
creates a new node that still points at the same underlying Deluge
function object as the original; the only safe way to get an independent
function is the "+ Custom Function" wizard under Built-ins → Developer
Tools → Custom Functions). Flow URL:
`https://flow.zoho.com/#/workspace/872426000000002011/flows/m3_periodic_check_in_trigger/edit`.
Left switched **OFF** after building, matching M0/M1/M2's build-first-
verify-later convention — turning it on is deferred to T020-T023 (blocked
on a test Patient ID from Costin).

**Structure, verified via screenshot + DOM connector/endpoint counts (4
connectors, 25 jsPlumb endpoints; the "error" class hits found in a DOM
scan are all `zf-action-error` / "On Error" branch anchors that exist on
every action node, not actual configuration errors):**

1. **Trigger — "Updated module entry" (Zoho CRM)**. Connection: "CRM
   Connection" (reused, same as M0/M1/M2). Module: **Patients** (variable
   name `trigger`, unchanged default). Filter criteria: `Session Count`
   `greater than or equals` `8` — confirmed via the trigger's own
   Configure screen, matching `m3-data-model.md` exactly (a cheap
   pre-filter only; the real per-checkpoint eligibility test is
   `checkPeriodicCheckInDue`, not this filter — see `m3-research.md`
   Decision 1 for why a modulo/list condition can't live in the trigger
   filter itself).
2. **Custom Function — `checkPeriodicCheckInDue`**. Built genuinely fresh
   (not cloned). Parameters as wired: `patientId` = `${trigger.id}`,
   `sessionCount` = `${trigger.Session_Count}`; output variable
   `checkPeriodicCheckInDue_1`. **Verbatim Deluge source, re-read from the
   live "Edit function" editor to confirm no drift from the draft**:

   ```text
   bool checkPeriodicCheckInDue(string patientId, int sessionCount)
   {
   	checkpoints = list();
   	checkpoints.add(8);
   	checkpoints.add(16);
   	checkpoints.add(24);
   	checkpoints.add(32);
   	checkpoints.add(40);
   	existingCount = 0;
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
   		if(patientRecId == patientId && r.get("Milestone") == "3 - Periodic Consolidated")
   		{
   			existingCount = existingCount + 1;
   		}
   	}
   	if(existingCount >= checkpoints.size())
   	{
   		return false;
   	}
   	nextThreshold = checkpoints.get(existingCount);
   	return sessionCount >= nextThreshold;
   }
   ```

   **No deviation from `m3-data-model.md`'s draft** — pasted in and saved
   with zero syntax corrections needed (unlike T014's Analytics formula,
   which needed an `OR` syntax fix; this function's Deluge syntax was
   already correct as drafted). Connection name `crm_connection`, reused
   verbatim from M0/M1/M2's pattern, confirmed live.
3. **If else** — condition: `checkPeriodicCheckInDue_1` is `true`.
   - **True branch → "Call a subflow"**, calling the unchanged shared
     "Subflow - Issue Feedback Token" (execution behavior: "Wait and
     continue", the default). Input Fields as configured and verified via
     screenshot:
     - `clinician`: `Liana Preudhomme`
     - `survey_url`:
       `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityPeriodicCheckIn/formperma/1a3D0cvimvTXnchCSaZUI66bwR8Oj1BJe8Cs0JBY8W4`
       (confirmed byte-exact via `document.activeElement.value` read
       immediately after typing, same discipline as M2's §4)
     - `email_intro_text`: "Thanks for continuing your work with us. As
       you keep making progress, we'd love to check in on how things have
       been going overall. Please take a moment to share your feedback
       using the link below:"
     - `milestone`: `3 - Periodic Consolidated`
     - `patient_id`: `${trigger.id}`
     - `recipient_email`: `${trigger.Email}`
     - `ttl_days`: `7` (reused M0/M1/M2's default — not changed for the
       recurring case; flagged as an open question for Costin in
       `m3-data-model.md`, no answer needed to proceed since the default
       is a safe placeholder)
     - `email_subject`: `Quick check-in: how's therapy going overall? -
       Cape Clarity`
     - `lead_id`: left empty, intentionally (M3 is patient-based only,
       same as M2)
   - **False branch**: left empty, no action — matches
     `m3-data-model.md`'s spec exactly (idempotency short-circuit, same
     shape as every prior milestone).

**New UI gotchas discovered this session (Zoho Flow builder, this
workspace version) — not documented in any prior milestone's notes:**

- **Reliable node placement/wiring needs a synthetic stepped drag, not a
  single drag gesture.** A plain `left_click_drag` from the palette to
  the canvas visually places and wires a node inconsistently in this
  builder version. What worked reliably: dispatch a real `mousedown` on
  the palette item's `.ui-draggable` element (at its center), then 15-18
  incremental synthetic `mousemove` events stepping toward the drop
  target (each resolving its real event target via
  `document.elementFromPoint(x,y)`, not just firing at fixed
  coordinates), then a final `mousemove` + `mouseup` at the target
  (target also resolved via `document.elementFromPoint`). Verified by
  `document.querySelectorAll('[class*="connector" i]').length` increasing
  by the expected amount after each drop.
- **Whether dropping a node auto-opens its parameter panel is
  inconsistent by node type.** Dropping the `checkPeriodicCheckInDue`
  custom-function node onto the trigger's output connector auto-opened
  its "Enter parameter values" panel immediately. Dropping "Call a
  subflow" onto the if-else's True-branch connector did **not**
  auto-open any panel, even using the identical stepped-drag technique
  (confirmed by deleting and re-dropping several times, including trying
  a trailing synthetic `click` after `mouseup`).
- **The reliable way to open ANY already-placed node's configuration
  panel is the small pencil icon, not the node body or the kebab menu.**
  Every node has a small `<i class="zf-icon-edit">` pencil icon at its
  top-right corner, visually close to but distinct from the kebab "..."
  icon just below it. Clicking the node's body/icon/text opens an inline
  rename `<input>` (covers most of the node row) or, on a second click,
  selects it; clicking the kebab opens only Clone / View function (or
  Clone / Add Note / Delete for non-function nodes) / Add Note / Delete —
  none of these expose parameter mapping. **Clicking the pencil icon
  specifically is what opens the true configuration panel** (confirmed
  for both the custom-function node and the "Call a subflow" node this
  session, and re-confirmed just now when re-opening both nodes purely
  to verify this write-up). For a custom-function node's panel, there is
  also an "Edit function" button (top-right of the parameter panel) that
  opens the actual Deluge source editor — a further level in, not the
  same click target as the parameter panel itself.
- Together, these mean: don't assume a silently-unopened panel means the
  node failed to place — try the pencil icon before concluding a node
  needs to be deleted and re-dropped.

### 1.6 "M3 - Periodic Check-In Write-back" flow built (T005/T006)

Built 2026-09-18, entirely from scratch — same rationale as §1.5: **not**
cloned from M2's "M2 - Alliance Check-In Write-back" flow, to avoid the
shared-custom-function gotcha (a cloned function node still points at the
same underlying Deluge function object as the original). Flow URL:
`https://flow.zoho.com/#/workspace/872426000000002011/flows/m3_periodic_check_in_write_back/edit`.
Left switched **OFF** after building, matching every prior milestone's
build-first-verify-later convention — turning it on is deferred to
T020-T023 (blocked on a test Patient ID from Costin).

**Structure, verified via DOM connector count (2 connector elements,
matching `m2-implementation-notes.md` §3.4's confirmed wired-state count
for the equivalent M2 flow) and screenshot (blue arrow drawn from the
trigger's output to the function node's input):**

1. **Trigger — "Form entry submitted" (Zoho Forms), REALTIME.** Form:
   "Cape Clarity Periodic Check-In" (the form built in §1.4).
2. **Custom Function — `submitPeriodicCheckInResponse`**. Built genuinely
   fresh via Built-ins → Developer Tools → Custom Functions → "+ Custom
   Function" (not cloned). Signature: `map
   submitPeriodicCheckInResponse(string token, string connection, string
   understanding, string sharedDirection, string fitOfApproach, string
   scheduling, string billing, string professionalism)` — mirrors M2's
   `submitAllianceCheckInResponse` exactly for the 5 parameters M3 shares
   with M2 (`token`, `connection`, `understanding`, `sharedDirection`,
   `fitOfApproach`), plus 3 new parameters for M3's Practice
   Experience/Professionalism domains. Saved cleanly on the first attempt
   (Zoho's "We have successfully created the function" banner, no syntax
   errors) — written via the CodeMirror `setValue()` API, per the
   documented typing-corruption-avoidance discipline. Verbatim Deluge
   source (re-read from the live "Edit function" editor to confirm no
   drift):

   ```text
   map submitPeriodicCheckInResponse(string token,string connection,string understanding,string sharedDirection,string fitOfApproach,string scheduling,string billing,string professionalism)
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
   	responseText = "Practice Experience: Scheduling/Communication (0-10): " + ifnull(scheduling,"") + "---";
   	responseText = responseText + "Practice Experience: Billing (0-10): " + ifnull(billing,"") + "---";
   	responseText = responseText + "Therapist Professionalism (0-10): " + ifnull(professionalism,"") + "---";
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

   Blob field order deliberately follows `m3-data-model.md`'s spec
   (Scheduling/Communication, Billing, Professionalism, then the 4 reused
   alliance domains) rather than the form's on-screen order (which puts the
   4 alliance sliders first) — this matters because it's what keeps the
   existing M2 Analytics formula columns (Domain: Connection/Understanding/
   Shared Direction/Fit Of Approach, Alliance Check-In Total) parsing M3
   rows with zero edits, per §1.2. Label text for the 4 reused segments
   (`"Connection (0-10): "`, etc.) is copied verbatim from M2's
   `submitAllianceCheckInResponse`, confirmed character-for-character
   against `m2-implementation-notes.md`'s source before saving.

3. **Wiring**: connected via a plain `left_click_drag` from the trigger's
   output connector circle to the function node's input connector circle
   (no synthetic stepped-drag needed this time — unlike §1.5's node
   placement/wiring from the palette, this was a short drag between two
   already-placed connector handles, and the simple gesture worked on the
   first attempt). Confirmed wired via
   `document.querySelectorAll('[class*="connector" i]').length` going from
   `0` (placed but unwired) to `2` (wired) — same verification convention
   `m2-implementation-notes.md` §3.4 and this file's §1.5 use.

4. **Parameter mapping** — all 8 parameters mapped to the trigger's form
   fields and confirmed via screenshot after typing each (values persisted
   correctly on a follow-up re-open of the panel):

   | Parameter | Mapped to | Form question (for reference) |
   |---|---|---|
   | `token` | `${trigger.SingleLine}` | (hidden Token field) |
   | `connection` | `${trigger.Slider}` | "Overall, how comfortable have you felt being open and honest with your therapist so far?" |
   | `understanding` | `${trigger.Slider2}` | "Overall, how well do you feel your therapist has understood what matters to you so far?" |
   | `sharedDirection` | `${trigger.Slider3}` | "Overall, how much do you feel you and your therapist agree on what you're working toward?" |
   | `fitOfApproach` | `${trigger.Slider4}` | "Overall, how well has your therapist's approach been working for you so far?" |
   | `scheduling` | `${trigger.Slider1}` | "How satisfied have you been with scheduling and communication with our practice (not your therapist directly)?" |
   | `billing` | `${trigger.Slider5}` | "How satisfied have you been with the billing/insurance process?" |
   | `professionalism` | `${trigger.Slider6}` | "How would you rate your therapist's professionalism (punctuality, preparedness, respectful conduct)?" |

**New UI gotchas discovered this session, in addition to §1.5's three:**

- **Zoho Forms field internal names do not necessarily follow on-screen
  field order, and the Properties panel doesn't expose them.** Scrolled
  the full field-Properties panel (Field Label, Instructions, Field Size,
  Hover Text, Initial Value, Range, Unit, Step Value, Validation,
  Visibility) — no "Unique Name"/internal-name control anywhere in it.
  Confirmed this session as a live instance of a previously-suspected
  gotcha: this form's internal field names (`Slider`, `Slider1`...
  `Slider6`, `SingleLine`) were assigned at field-creation time, before the
  drag-and-drop reordering fixes documented in §1.4 moved fields to their
  final on-screen positions — so, e.g., the on-screen-5th field
  ("Scheduling/Communication") is internally `Slider1`, not `Slider5`.
  **Reliable technique found this session**: the builder's canvas lives in
  a cross-origin iframe (unreadable via `javascript_tool`), but the
  parameter-mapping panel's "Insert variable" tree (in Zoho Flow, same-
  origin) exposes each field's true internal token as the `data-value`
  attribute on its `a.jstree-anchor` element (e.g. `data-value=
  "trigger.Slider1"`), paired with the full (untruncated, even where the
  visible UI truncates it) question text as the anchor's `textContent`.
  Querying `document.querySelectorAll('a.jstree-anchor[data-value^=
  "trigger."]')` in the write-back flow's parameter panel gave a complete,
  authoritative label-to-internal-name mapping in one call — far more
  reliable than the previously-considered approach of reading the live
  public form's DOM (`div.fieldWrapper[id$="-li"]`), which this session
  didn't end up needing.
- **Clicking a field, then clicking (or JS-`.click()`-ing) the matching
  `a.jstree-anchor` in the "Insert variable" panel, inserts the merge
  token (`${trigger.SliderN}`) into that field** — confirmed as the
  correct mechanism for populating custom-function parameter fields
  from trigger data (as opposed to typing the token as literal text,
  which was not tried but is very likely fragile given prior sessions'
  documented text-field corruption bug). Dispatching `.click()` via
  `javascript_tool` on the anchor element worked identically to a real
  mouse click, letting each of the 8 parameters be mapped and
  independently verified via screenshot without needing precise pixel
  coordinates for the (long, scrollable) variable tree.
- **The Built-ins sidebar's "+ Custom Function" button can end up below
  the visible fold in a way mouse-wheel scroll doesn't reach.** A
  `scroll` action at the sidebar's coordinates left `scrollTop` at `0`
  across multiple attempts. Fixed by finding the actual scrollable
  container via `document.querySelectorAll('div')` filtered for
  `scrollHeight > clientHeight` (found `div.sidebarMenuList`, scrollHeight
  834 vs clientHeight 597) and setting `el.scrollTop = el.scrollHeight`
  directly via `javascript_tool` — revealed the button immediately.

## 2. CRM/picklist reference (unchanged from m3-data-model.md, confirmed live)

Confirmed 2026-09-17 via Zoho CRM MCP `getFields` (not browser, per the
standing access constraint — `Milestone_Instances` carries no PII): the
`"3 - Periodic Consolidated"` picklist value already exists on
`Milestone_Instances.Milestone`. No CRM schema change was needed or made.
Full picklist as of that check: `-None-`, `0 - No Conversion`,
`1 - Baseline Intake`, `2 - Early Alliance Check`, `3 - Periodic
Consolidated`, `4 - Discharge`, `5B - Discontinuation, Email Fallback`,
`5A - Discontinuation, Live Capture`. See `m3-data-model.md`'s "Milestone
picklist value" section — reproduced here only because it's load-bearing
for every task that references the exact string `"3 - Periodic
Consolidated"`.

## 3. Blocker (RESOLVED 2026-09-18): Zoho Forms plan no longer permitted creating a new form (T001)

**Resolved.** Costin subscribed to a paid Zoho Forms plan for this account
on 2026-09-18, removing the blocker described below. Confirmed resolved by
successfully opening the "New Form" wizard with no upgrade-plan modal
appearing (previously both paths below triggered one). T001 was then built
the same session — see §1.4. This section is kept as a historical record of
the blocker and how it was diagnosed, per the "not just a failed click"
verification discipline the rest of this file follows.

This account's Zoho Forms plan (`lianapreudhommecapec1`) — the same account
`m2-implementation-notes.md` §2 recorded as "Free" — no longer permitted
creating a new form by either path tried:

- **"New Form"** from the forms dashboard.
- **"Duplicate"** on an existing form (which M2's §4 already found
  non-functional via browser automation on this account for a different
  reason — worth noting this is now doubly blocked).

Both actions triggered a paid-plan upgrade modal
(`ZFUtil.upgradeAndshowReloadPopup`), linking to a Zoho Store purchase page.
This is a genuine external blocker, not an automation bug — confirmed by
the modal's own upgrade-prompt copy, not just a failed click.

**Per the standing operating constraint (never purchase or upgrade a paid
plan without Costin's explicit permission), no purchase was attempted.**
This blocks:

- **T001** directly — the "Cape Clarity Periodic Check-In" form cannot be
  built on the current plan.
- **T005/T006** transitively — the "M3 - Periodic Check-In Write-back" flow
  needs T001's form to exist as its realtime trigger.
- **T002-T004** are not directly blocked by this (the trigger flow and its
  custom function don't depend on the form existing yet), but T004's "Call a
  subflow" step needs a real `survey_url` value, which doesn't exist without
  T001. T002-T004 could proceed with a placeholder `survey_url` to be
  swapped in once the form exists, but that's a judgment call worth
  confirming with Costin rather than assuming.

**Resolution**: Costin subscribed to a paid Zoho Forms plan (2026-09-18),
removing this blocker. No purchase was made by this session — Costin
handled the subscription himself, per the standing constraint.

## 4. Remaining work (not yet built)

- **T001**: Done — see §1.4.
- **T002-T004**: Done — see §1.5. Trigger flow built fresh, wired
  end-to-end (verified via DOM connector/endpoint counts and screenshots),
  left switched off pending live test data.
- **T005/T006**: Done — see §1.6. Write-back flow built fresh, wired
  end-to-end (verified via DOM connector count going from 0 to 2), all 8
  `submitPeriodicCheckInResponse` parameters mapped to the form's true
  internal field names and verified via screenshot, left switched off
  pending live test data.
- **T007-T013**: Verification/confirmation tasks depending on T002-T006
  being built (now done) and (T020-T023) live-tested — not started.
- **T014**: Done — see §1.1.
- **T015**: Not yet reviewed — confirm-absence task (no contractor-facing
  notification anywhere in the M3 build); trivially true right now since
  nothing M3-specific has been built yet beyond the shared Analytics
  columns, but should be re-checked once T002-T006 exist.
- **T016**: Partially done — see §1.3. Filter widening still outstanding.
- **T017**: Done — see §1.2.
- **T018**: Not started — "M3 Submitted Responses" report.
- **T019**: Not started — "M3 Status Breakdown", "M3 Practice Experience &
  Professionalism Distribution", and the "M3 - Periodic Check-In Feedback"
  dashboard.
- **T020-T023**: Explicitly blocked pending a test Patient ID from Costin,
  per standing instruction (no live/end-to-end testing without him).
- **T024** (this file): In progress — being written/updated as work
  proceeds, per CLAUDE.md's same-session convention, rather than held until
  the whole milestone is done.
- **T025**: Done for the changes made so far (§1.1's flag-column widening,
  §1.3's report rename) — `m2-implementation-notes.md` §9.2 and §9.6 updated
  in this session. Will need a further update once T016's filter widening
  is actually completed.
- **T026**: Not yet assessed — worth revisiting once T002-T006 are built,
  per `m3-tasks.md`'s note that the recurring-checkpoint pattern and the
  shared-vs-per-milestone Analytics decision framework are the most likely
  candidates for promotion to `CLAUDE.md`.
- **T027**: This file and the `m2-implementation-notes.md` updates are being
  committed and pushed in this session (via the browser-upload workaround,
  same as the M3 planning docs) — see the top-level commit for this change.

## 5. Session note: browser access interrupted mid-task

The browser session this work was using (Zoho Analytics, via the Chrome
extension) disconnected while investigating T016's filter-editing UI (see
§1.3's "not yet tried" note). Per the "avoid rabbit holes" browser-tool
guidance, this was not repeatedly retried — instead, work pivoted to
non-browser tasks: writing this file, updating `m2-implementation-notes.md`,
and pushing both to GitHub.

**Update**: browser access was restored in a later session, which used it
to build T001 (the Periodic Check-In form), T002-T004 (the trigger flow),
and T005/T006 (the write-back flow — see §1.4/§1.5/§1.6) rather than
returning to T016's filter fix first, per Costin's go-ahead to proceed
into the Zoho Flow build. Both M3 flows are now built, wired, and
structurally verified. Per Costin's standing instruction to pause at
transition points between major task groups, this is where work stops:
T016's filter widening (§1.3) and T018/T019 (the two remaining M3 reports
+ dashboard) remain open and are the natural next step once Costin has
checked in on usage/progress and given the go-ahead to continue into
Analytics/reporting work.
