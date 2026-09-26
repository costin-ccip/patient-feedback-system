# M3 (Periodic Consolidated Check-In, every 8th session) — Implementation Notes

> Scope: this file documents ONLY the M3 milestone (the recurring
> every-8th-session Periodic Consolidated Check-In). It is a companion to
> `specs/001-feedback-collection-pipeline/spec.md` (all six milestones) and to
> `m3-plan.md` / `m3-research.md` / `m3-data-model.md` / `m3-tasks.md`. This
> file tracks what has actually been built in Zoho, as it's built, per the
> `CLAUDE.md` convention of updating implementation notes in the same session
> as the change.
>
> **2026-09-23: token issuance no longer uses a subflow. See §6.**
>
> Last updated: 2026-09-24 (see §1.1/§1.3 changelog notes — the Clinical
> Safety Flag gate and the Flagged for Review report were both further
> widened by the M4 build, per `m4-tasks.md` T017/T019; nothing else in
> this file changed)
> Status: **All of Phase 6 (Reporting) is now built: T016 (filter widening
> completed), T018, and T019 are all done**, alongside everything from the
> prior update (T014, T017, the M3 Zoho Form, and both M3 Zoho Flow flows).
> Done so far: T014 (Clinical Safety Flag `Milestone` gate widened to cover
> M3), T017 (3 new M3-only Analytics formula columns), **T016** (report
> renamed "M2 Flagged for Review" → "M2 & M3 Flagged for Review" **and its
> Milestone filter now fully widened** to Wildcard `"2 - Early Alliance
> Check"` OR `"3 - Periodic Consolidated"` — see §1.3), **T001** (the "Cape
> Clarity Periodic Check-In" Zoho Form — see §1.4; the Forms-plan blocker
> described in §3 was resolved 2026-09-18 when Costin subscribed to a paid
> Zoho Forms plan), **T002-T004** (the "M3 - Periodic Check-In Trigger"
> flow — trigger, `checkPeriodicCheckInDue` custom function, if-else, and
> "Call a subflow" step — built fresh, wired end-to-end, and verified; left
> switched **off** — see §1.5), **T005/T006** (the "M3 - Periodic Check-In
> Write-back" flow — trigger, `submitPeriodicCheckInResponse` custom
> function, all 8 parameters mapped to the form's field internal names —
> built fresh, wired end-to-end, and verified; left switched **off** — see
> §1.6), **T018** (the "M3 Submitted Responses" report — see §1.7), and
> **T019** (the "M3 Status Breakdown" report plus three new-column
> distribution reports, bundled with T018's report and T016's shared
> Flagged for Review report into the new "M3 - Periodic Check-In Feedback"
> dashboard — see §1.8). T020-T023 (live test data) remain explicitly
> blocked pending a test Patient ID from Costin, per standing instruction —
> that is the only remaining substantive work before M3 is fully built and
> tested. See §4 for the full remaining-work list.

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

**Updated 2026-09-24 (M4 build, `m4-tasks.md` T017):** the `Milestone` gate
was widened a second time, to M2-or-M3-or-M4, per `m4-research.md` Decision
6 and `m4-data-model.md` (M4 reuses this same shared alliance-flag column
against its own Discharge-milestone domain readings, the same "one rule,
don't fork" reasoning this section's own M3 change used against M2). Edited
via the "Edit Formula Column" dialog's CodeMirror 6 editor (no global
`setValue` API — done via click positioning + arrow-key navigation,
verified with zoomed before/after screenshots, then confirmed via
`.cm-content.textContent` after reopening the dialog). Infix `OR` used
again throughout, consistent with the finding above. Live formula as of
2026-09-24:

```
IF("Milestone Instances"."Milestone" = '2 - Early Alliance Check' OR "Milestone Instances"."Milestone" = '3 - Periodic Consolidated' OR "Milestone Instances"."Milestone" = '4 - Discharge', IF("Milestone Instances"."Alliance Check-In Total" <= 20 OR to_integer("Milestone Instances"."Domain: Connection") <= 4 OR to_integer("Milestone Instances"."Domain: Understanding") <= 4 OR to_integer("Milestone Instances"."Domain: Shared Direction") <= 4 OR to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, 'true', 'false'), '')
```

`m2-implementation-notes.md` §9.2 has been updated to match (see that
file's changelog note under §9.2); `m4-implementation-notes.md` also
records this change.

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

### 1.3 "M2 & M3 Flagged for Review" report — renamed and filter widened (T016, done)

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

**The Milestone filter has now been widened.** The correct UI path,
previously unresolved, turned out to be the **"Edit Design"** button
(top-right of a Tabular View report in View Mode) — clicking it switches
the URL from `/view/<id>` to `/edit/<id>` and opens the full query editor
with "Tabular", "Filters (N)", and "User Filters (N)" tabs. This is
distinct from the toolbar's ad-hoc per-column quick-filter and its "More"
dropdown (neither of which exposes the report-level saved filter), both
already ruled out in the prior update to this section. Within the
"Filters" tab, clicking the existing "Milestone" filter opened its
criteria editor (Individual Values / Wildcard tabs); clicking "+" next to
the existing condition added a second, OR'd condition row. Final filter:
`Milestone` Wildcard Exactly Matches `"2 - Early Alliance Check"` **OR**
Exactly Matches `"3 - Periodic Consolidated"` (Criteria Expression `(1 OR
2)`, confirmed in the UI). Saved successfully ("Saved 'M2 & M3 Flagged for
Review' successfully" toast confirmed). View ID unchanged:
`3251423000000141219`.

**Practical effect**: the report now correctly scopes to both M2 and M3
flagged rows, matching its title. `m2-implementation-notes.md` §9.6 has
been updated to remove its "still M2-only" caveat (see that file).

**Updated 2026-09-24 (M4 build, `m4-tasks.md` T019):** renamed a second
time, to "M2, M3 & M4 Flagged for Review", and the Milestone Wildcard
filter widened with a third OR'd condition, Exactly Matches `"4 -
Discharge"` (added via the same "Edit Design" → Filters tab → "+" path;
Criteria Expression auto-updated to `(1 OR 2 OR 3)`). The native-input-
setter rename technique from this section was reused and again avoided the
text-corruption bug — confirmed via screenshot immediately after and again
after a full page reload. The report's second filter, `Clinical Safety
Flag` Exactly Matches `"true"`, was inspected and needs no change (no
Milestone-specific logic in it). Both the rename and the filter widening
were verified to persist across a full page reload before moving on. View
ID unchanged: `3251423000000141219`. `m2-implementation-notes.md` §9.6 has
been updated to match; `m4-implementation-notes.md` also records this
change. Separately confirmed for T018: no automated notification or CRM
action reaches a contractor when an M4 reading fires the Clinical Safety
Flag (M4's trigger flow's On-Error branches alert only
`costin@capeclarity.com`; its write-back flow only updates
`Response_Data`/`Status`/`Submitted_Date_Time`, with no notification step).

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

### 1.7 "M3 Submitted Responses" report built (T018)

Built as a new Tabular View on "Milestone Instances" (not a Save-As — M3's
column set differs from M2's/M1's by the 3 new Practice Experience/
Professionalism columns, same reasoning M2 gave for not reusing M1's
now-`[RETIRED]` version). 12 columns, in this order: Submitted Date Time,
Token, Practice Experience: Scheduling/Communication, Practice Experience:
Billing, Therapist Professionalism, Domain: Connection, Domain:
Understanding, Domain: Shared Direction, Domain: Fit Of Approach, Alliance
Check-In Total, Clinical Safety Flag, Flag Rule Triggered — no
`Patient`/identity column, same de-identification discipline as every
other M0/M1/M2 report. Wildcard filter (needed because `"3 - Periodic
Consolidated"` doesn't exist yet as a real picklist value in any row so far
in this table): `Milestone` Exactly Matches `"3 - Periodic Consolidated"`.
Saved to folder "Zoho CRM Modules (Data)". View ID `3251423000000186070`.
0 rows (expected — correctly filters out the 9 pre-existing non-M3 sample/
test rows already in the table; confirmed via "Click Here to Generate
Tabular" before saving, and again in View Mode after saving).

### 1.8 "M3 Status Breakdown" and distribution reports, plus the M3 dashboard (T019)

**"M3 Status Breakdown"** (status/volume view): saved via "Save As" off
"M2 Status Breakdown" (`m2-implementation-notes.md` §9.4, view ID
`3251423000000141053`), same bar chart config preserved (X-Axis `Status`
Actual, Y-Axis `Id` Count). Only change: the Milestone filter's Wildcard
value swapped from `"2 - Early Alliance Check"` to `"3 - Periodic
Consolidated"`. Saved to "Zoho CRM Modules (Data)". View ID
`3251423000000186076`. Regenerated graph shows "No Data Available"
(expected).

**Distribution views of M3's own new scored data** — `m3-data-model.md`
calls for "bar chart(s) over the 3 new formula columns" (plural allowed;
FR-018/User Story 6 requires only "at least one"). Because each of the 3
new columns (Practice Experience: Scheduling/Communication, Practice
Experience: Billing, Therapist Professionalism) is an independent 0-10
scored dimension, a single bar chart's X-axis can't meaningfully combine
all 3 without reshaping the underlying data (out of scope here) — so this
was built as **three separate distribution reports**, one per column,
collectively satisfying the "M3 Practice Experience & Professionalism
Distribution" task item:

- **"M3 Practice Experience: Scheduling/Communication Distribution"** —
  view ID `3251423000000186098`.
- **"M3 Practice Experience: Billing Distribution"** — view ID
  `3251423000000186132`.
- **"M3 Therapist Professionalism Distribution"** — view ID
  `3251423000000186154`.

All three were saved via "Save As" off "M2 Alliance Check-In Total
Distribution" (`m2-implementation-notes.md` §9.5, view ID
`3251423000000141097`), then had their X-axis column swapped to the
relevant new column and their Milestone filter's Wildcard value swapped to
`"3 - Periodic Consolidated"`. **Note on the `Dimension [Actual(D)] /
Treat as Text` gotcha**: `m2-implementation-notes.md` §9.5 warns that a
newly-dropped *numeric* column defaults to `Measure [Actual(M)]` (which
aggregates instead of showing one bar per distinct value) and must be
explicitly re-set to `Dimension [Actual(D)] / Treat as Text`. That gotcha
did not apply here — all 3 new Practice Experience/Professionalism formula
columns are **Text-typed** (same as the reused Domain: * columns), so they
default correctly to `Actual` dimension mode with no explicit re-set
needed; confirmed via the X-Axis dropdown showing plain `Actual` (not
`Actual(M)`) immediately after each column swap. All three regenerated
graphs show "No Data Available" (expected). Y-Axis unchanged on all three:
`Id` (Count).

**"M3 - Periodic Check-In Feedback" dashboard**: new dashboard, view ID
`3251423000000186289`, built via "Create New Dashboards" (not "Save As" —
same reasoning M1/M2 gave: no exact panel-for-panel match to copy from).
Bundles exactly six reports, dragged in from the Reports panel:

- M3 Status Breakdown (this section)
- M3 Practice Experience: Scheduling/Communication Distribution (this
  section)
- M3 Practice Experience: Billing Distribution (this section)
- M3 Therapist Professionalism Distribution (this section)
- M3 Submitted Responses (§1.7)
- M2 & M3 Flagged for Review (§1.3, shared with M2)

Left at "Auto Add User Filters" on and "Make User Filters Global" off,
matching M0/M1/M2's dashboard defaults. All six panels verified rendering
correctly in View Mode: the four chart panels show "No Data Available"
and the two tabular panels (Submitted Responses, Flagged for Review) show
their correct column headers with 0 rows — all expected, since no real M3
submissions exist yet.

**One drag-and-drop mishap worth noting for future dashboard builds**: the
reports panel's list is alphabetically sorted and re-flows as items are
added, so a coordinate that pointed at "M3 Practice Experience: ..." one
moment pointed at "M2 Submitted Responses" the next (after an earlier drag
consumed a list slot). Caught immediately because the wrong panel rendered
real M2 sample data instead of "No Data Available"/the expected column
set — removed via the panel's "..." → "Remove" and re-added correctly.
Future sessions building multi-report dashboards should verify each
panel's rendered title/content immediately after each drag rather than
assuming the coordinate-to-report mapping stays stable across drags.

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
- **T016**: Done — see §1.3. Report renamed and Milestone filter fully
  widened (Wildcard, both M2 and M3 values).
- **T017**: Done — see §1.2.
- **T018**: Done — see §1.7. "M3 Submitted Responses" report built,
  verified 0 rows in View Mode.
- **T019**: Done — see §1.8. "M3 Status Breakdown" plus three per-column
  distribution reports (covering the 3 new Practice Experience/
  Professionalism columns), bundled with T018's and T016's reports into
  the new "M3 - Periodic Check-In Feedback" dashboard. All panels verified
  rendering correctly in View Mode.
- **T020-T023**: Explicitly blocked pending a test Patient ID from Costin,
  per standing instruction (no live/end-to-end testing without him). This
  is now the only remaining substantive work before M3 is fully built and
  tested.
- **T024** (this file): In progress — being written/updated as work
  proceeds, per CLAUDE.md's same-session convention, rather than held until
  the whole milestone is done.
- **T025**: Done — `m2-implementation-notes.md` §9.2 and §9.6 updated in
  this session for T014's flag-column widening, T016's report rename, and
  (this update) T016's filter-widening completion.
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
transition points between major task groups, work stopped there and a
check-in message was sent.

**Further update**: Costin replied "continue", authorizing the remaining
Analytics/reporting work. That work is now done: T016's filter widening
was completed (the "Edit Design" UI path, previously unresolved, was
found — see §1.3), followed by T018 (§1.7) and T019 (§1.8, including the
"M3 - Periodic Check-In Feedback" dashboard). All of Phase 6 (Reporting)
is now complete. The only remaining work is T020-T023 (live test data),
explicitly blocked pending a test Patient ID from Costin per standing
instruction, and the lower-priority polish items T026/T027.

## 6. Change (2026-09-23): token issuance no longer uses a subflow

"M3 - Periodic Check-In Trigger" is now
`trigger → checkPeriodicCheckInDue → If else (True) → issueFeedbackToken → Send email`.
This supersedes §1.5 step 3's True branch. `issueFeedbackToken` parameters:
milestone `3 - Periodic Consolidated`, patientId `${trigger.id}`, leadId empty,
recipientEmail `${trigger.Email}`, clinician `Liana Preudhomme`, ttlDays `7`. Email
subject, intro text and survey link are §1.5's values unchanged (re-read from the live
node before it was deleted; they matched). `issueFeedbackToken` itself was created
from this flow's builder, so it first appeared in this flow.

**Correction to §1.5's wiring claim**: after the new function node was dropped onto
the If-else True endpoint (the same way the subflow node was placed on 2026-09-18),
it sat in the attached-looking position but was **not wired**: the If-else `b.out`
had no `jsplumb-connected` class and there was no connector line, while the
`[class*="connector" i]` count still read the same. It was fixed by moving the node
and dragging a connection from the True endpoint. The 2026-09-18 subflow node may
have had the same problem; §1.5's "verified via DOM connector counts" didn't detect
it either way. Use the `jsplumb-connected` check from 002's implementation notes §5.1
instead.

This is the as-built change for `specs/002-remove-subflow-dependency` (subflows
aren't available on the Zoho Flow Standard plan). The flow's "Call a subflow →
Subflow - Issue Feedback Token" step was deleted and replaced with two steps: the
shared custom function `issueFeedbackToken` (output variable `issueFeedbackToken_1`;
does supersede + token/expiry + Milestone_Instances create, the same logic the
subflow ran) and a native **Zoho Mail "Send email"** step (connection "Connection to
info@capeclarity.com", From `info@capeclarity.com`, To `${trigger.Email}`, same
subject/intro/survey link as before, link ends `?token=${issueFeedbackToken_1.token}`).
Record shape, email content and sender are unchanged. The flow stays **OFF**. Full
function source, per-flow values and new builder gotchas:
`specs/002-remove-subflow-dependency/implementation-notes.md`. The retired subflow's
internals (never documented here before) are in that feature's `research.md` §0.
Earlier text in this file that describes "Call a subflow" is kept as history and no
longer describes the live build.

**Effect on remaining work (§4)**: T020-T023's live test now exercises the new path.
In the test, confirm the execution shows `issueFeedbackToken` + Send email steps,
not "Call a subflow" (see 002 implementation notes §5.6 on the "Draft" badge).

**Follow-up (2026-09-23, feature 003)**: record names no longer contain the
recipient email (`Milestone <m> - <token prefix>`), and this flow now has On Error
branches: issuance failure → alert to costin@capeclarity.com; patient-email failure →
record Status `Send Failed` → alert. Resend runbook:
`specs/003-issuance-privacy-and-failure-handling/quickstart.md` §C.

## 7. Change (2026-09-23): cutoff-exclusion for legacy patients

Same change as `m2-implementation-notes.md` §11, reusing the same shared custom
function rather than building a separate copy (Zoho Flow custom functions are
workspace-level and already showed up in this flow's Custom Functions sidebar
list once created from M2's builder):

```
bool isCreatedAfterCutoff(string createdTime)
{
	cutoff = "2026-09-23T00:00:00-04:00".toDateTime();
	created = createdTime.toDateTime();
	return created >= cutoff;
}
```

"M3 - Periodic Check-In Trigger" is now
`trigger → checkPeriodicCheckInDue → isCreatedAfterCutoff → If else (both true) → issueFeedbackToken → Send email`.
Same build steps as M2: existing wire from `checkPeriodicCheckInDue` to `If
else` detached, the shared function dropped onto the canvas between them,
wired explicitly in both directions (16 `jsplumb-connected` endpoints total
afterward, up from 14 - same verification method as 002's implementation
notes §5.1), output variable renamed to `isCreatedAfterCutoff_1` (a separate
instance from M2's, since Output Variable Name is scoped per flow even though
the function body is shared), and its `createdTime` parameter mapped to
`${trigger.Created_Time}`. The If-else condition now reads
`checkPeriodicCheckInDue_1 is true` **AND** `isCreatedAfterCutoff_1 is true`
(previously just the first clause). Flow stays **OFF**.

**Why**: see m2's §11 - no date-range/`>=` operator exists anywhere in Zoho
Flow's no-code condition UI, confirmed again in this flow's own If-else editor.

**Follow-up (2026-09-23, same session)**: same as m2 §11 - not run through a
formal spec-kit feature folder at build time. Raised with Costin, who said to
document it properly and confirmed M0 does NOT need the same gate. Backfilled as
`specs/004-legacy-patient-exclusion`; see that folder's `implementation-notes.md`
for full chronology and builder gotchas.

## 8. Change (2026-09-25): clinician now derived from Assigned Therapist, not hardcoded

Same bug and same fix as `m2-implementation-notes.md` §12 (read that section
for the full finding, the `Assigned_Therapist` picklist confirmation, and
the clearing/insertion mechanics — not repeated here). On "M3 - Periodic
Check-In Trigger", the `issueFeedbackToken` node's `clinician` parameter
was changed from the literal text `Liana Preudhomme` (§6 above shows the
pre-fix parameter list) to the chip `Updated module entry → Assigned
Therapist` (`${trigger.Assigned_Therapist}`). All other `issueFeedbackToken`
parameters unchanged. Cleared via click → `End` → `Backspace` ×25, inserted
via the Insert Variable panel's "Updated module entry" category (expand
via the category header, not the search box, to avoid stealing focus from
the target field), saved, then reopened to confirm the chip persisted.

M0 deliberately left unchanged, same reasoning as M2's §12. Identical
treatment applied the same session to M2, M4, and M5 — see
`m2-implementation-notes.md` §12, `m4-implementation-notes.md`, and
`m5-implementation-notes.md`. Flow stays **OFF**.

## 9. Fix (2026-09-25, pre-emptive): Field Alias missing for hidden Token field

Found while investigating a real M2 write-back failure Costin hit in his
live test — see `m2-implementation-notes.md` §13 for the full diagnosis.
Same root cause applies here: "Cape Clarity Periodic Check-In"'s hidden
`Token` field had no Field Alias configured (Settings → Prefill → Field
Alias - Prefill URL was on the empty "Configure Now" screen), so the
`?token=...` URL parameter the trigger flow sends would never have
reached the field — this would have failed the same way M2 did the
moment Costin live-tested M3, not something specific to M3's own build.

**Fixed**: Field Label `Token`, Field Alias `token` → Save. Not yet
live-tested end-to-end (M3's flows are still OFF per §1), so this is a
pre-emptive fix, not a confirmed-working one the way M2's is — worth
confirming during M3's own live test that the hidden field actually
prefills, same as M2 §13's verification step.
