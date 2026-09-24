# M4 (Discharge) — Implementation Notes

> Scope: this file documents ONLY the M4 milestone (Discharge — one-time-per-
> patient, triggered by "Discharge is marked in the system of record"). It is
> a companion to `specs/001-feedback-collection-pipeline/spec.md` (all six
> milestones) and to `m4-plan.md` / `m4-research.md` / `m4-data-model.md` /
> `m4-tasks.md`. This file tracks what has actually been built in Zoho, as
> it's built, per the `CLAUDE.md` convention of updating implementation notes
> in the same session as the change.
>
> Last updated: 2026-09-24
> Status: **T009-T013 (the "Cape Clarity Discharge Feedback" Zoho Form, all 7
> fields) are built and verified by top-to-bottom scroll. T003-T008 ("M4 -
> Discharge Trigger" flow, including both On Error branches) are also built
> and verified structurally — see §2. T008-T009/T014-T015 ("M4 - Discharge
> Write-back" flow: trigger, `submitDischargeFeedbackResponse` function,
> parameter mapping) are also built and verified structurally — see §3.**
> T017-T019 (widening the shared Clinical Safety Flag gate and "Flagged for
> Review" report to cover M4, plus confirming T018's no-contractor-
> notification requirement) are also built and verified — see §4, which
> also updated `m2-implementation-notes.md` and `m3-implementation-notes.md`
> in the same session per `m4-data-model.md`'s Decision 6. M4's own new
> Analytics (formula columns, reports, dashboard) is not yet started — see
> §5 for the full remaining-work list, which otherwise mirrors
> `m4-tasks.md`.

## 1. "Cape Clarity Discharge Feedback" Zoho Form (T009-T013)

Built fresh in Zoho Forms (`https://forms.zoho.com/lianapreudhommecapec1/form/CapeClarityDischargeFeedback/builder`),
matching `m4-data-model.md`'s field inventory exactly. On-screen order,
top to bottom, confirmed via full top-to-bottom scroll after the last field
was added (per `m3-implementation-notes.md` §1.4's drag-and-drop field-order
gotcha — no reordering was needed here, each field was dropped directly
below the previous one and landed in the intended position on the first
attempt):

| Order | Field type | Label (on-screen question text) | Instructions | Range | Mandatory |
|---|---|---|---|---|---|
| 1 | Slider | "Overall, how comfortable have you felt being open and honest with your therapist so far?" | "Domain: Connection. 0 = Not comfortable, 10 = Very comfortable." | 0-10 | Yes |
| 2 | Slider | "Overall, how well do you feel your therapist has understood what matters to you so far?" | "Domain: Understanding. 0 = Not understood, 10 = Fully understood." | 0-10 | Yes |
| 3 | Slider | "Overall, how much do you feel you and your therapist agree on what you're working toward?" | "Domain: Shared direction. 0 = Not aligned, 10 = Fully aligned." | 0-10 | Yes |
| 4 | Slider | "Overall, how well has your therapist's approach been working for you so far?" | "Domain: Fit of approach. 0 = Hasn't worked, 10 = Worked well." | 0-10 | Yes |
| 5 | Slider | "How likely are you to recommend Cape Clarity to someone in a similar situation?" | "0 = Not at all likely, 10 = Extremely likely." | 0-10 | Yes |
| 6 | Multi Line | "Is there anything else you'd like to share as you finish up your care with us?" | (none) | n/a | No |
| 7 | Single Line | "Token" | (none) | n/a | No |

**Deviation caught and corrected mid-build**: fields 1 and 2 (Connection,
Understanding) were initially saved with just the short domain name
("Connection" / "Understanding") as the Field Label, matching neither
`m4-data-model.md`'s drafted question text nor — critically — the actual
as-built convention M2/M3 already established (confirmed by re-reading
`m2-implementation-notes.md`'s own field table: the **full question text**
is the on-screen Field Label, and "Domain: X..." is a separate Instructions
line rendered beneath the slider). Caught before moving on to field 3;
fields 1 and 2 were reopened via Properties and corrected to the full
question text so all 7 fields are consistent with M2/M3's convention.
**Lesson for future milestones**: when a data-model doc's field inventory
gives both a short domain name and a full question in the same bullet
(`m4-data-model.md`'s format: "**Slider — Connection**: "Overall, how
comfortable...?" / "Domain: Connection...""), the quoted question text is
the Field Label, not the bolded domain name — the domain name is just the
doc's own shorthand for referring to the field, not on-screen content.

Field 7 (Token): Visibility set to **Hide** via the field's Properties panel
(Visibility section → Hide), matching the hidden-prefill-token pattern used
by every prior milestone's form. Not yet DOM-verified against the live
public-facing form render (M2's form got this verification per
`m2-implementation-notes.md` §1's "DOM verification done" note) — do this
before wiring the write-back flow's Insert Variable mappings in T014-T015,
the same way M2/M3 did, so the internal field names (`Slider`, `Slider1`,
etc.) used in `submitDischargeFeedbackResponse`'s parameter mapping are
confirmed rather than assumed.

No name, email, or phone field on this form, per Constitution Principle I —
matches every prior milestone.

Builder gotchas encountered this session (both already documented in prior
milestones' notes, reconfirmed here, no new findings):
- The Properties panel's Save button click doesn't always register on the
  first click attempt while a field value was just edited — clicking Save a
  second time closed the panel and persisted the change every time this
  happened. Not investigated further; treated the same as M2/M3's own
  "click twice if the toast doesn't appear" note.
- The browser window's viewport shrank mid-session (from the wider size set
  earlier to `1568x630`) for reasons not fully diagnosed — possibly an
  interaction with the account-menu dismissal earlier in the session. This
  did not block any field edits (the Properties panel's Save/Cancel buttons
  stayed reachable via scroll), so it was left as-is rather than chasing a
  `resize_window` call that did not visibly change the captured screenshot
  dimensions.

## 2. "M4 - Discharge Trigger" flow, step by step (T001-T008 — built)

Built fresh (not cloned) via Zoho Flow's builder
(`https://flow.zoho.com/#/workspace/872426000000002011/flows/m4_discharge_trigger/edit`),
following the token-issuance-without-subflows convention and the
legacy-patient cutoff-exclusion convention exactly as `CLAUDE.md` documents
them, and matching `m4-data-model.md`'s trigger-flow spec field-for-field:

1. **Trigger** — Zoho CRM "Updated module entry". Connection: CRM Connection.
   Module: `Patients`. Filter: `Patient Status` `equals` `Completed
   Treatment` (confirmation from Costin/Liana that this is the right
   discharge-trigger value is still outstanding — see §3).
2. **Custom Function** — `checkDischargeExists(patientId)`, created fresh via
   Built-ins → Developer Tools → Custom Functions → "+Custom Function" (not
   cloned from any other flow, avoiding the shared-function-object gotcha
   `m2-implementation-notes.md` §3.2 documents). Output variable
   `checkDischargeExists_1` (boolean). Input `patientId = ${trigger.id}`.
3. **Custom Function** — `isCreatedAfterCutoff(createdTime)`, the shared
   function from `specs/004-legacy-patient-exclusion` (reused, not
   recreated). Output variable `isCreatedAfterCutoff_1` (boolean). Input
   `createdTime = ${trigger.Created_Time}`.
4. **If else** — condition: `checkDischargeExists_1 is false` **AND**
   `isCreatedAfterCutoff_1 is true`.
   - **True branch → `issueFeedbackToken`** (shared, unmodified — same
     function M2/M3 call): `milestone` = `4 - Discharge`, `patientId` =
     `${trigger.id}`, `leadId` empty, `recipientEmail` = `${trigger.Email}`,
     `clinician` = `Liana Preudhomme`, `ttlDays` = `7`. Output variable
     renamed `issueFeedbackToken_1`.
   - **Then → Zoho Mail "Send email"** (patient-facing): connection
     "Connection to info@capeclarity.com"; From `info@capeclarity.com`; To
     `${trigger.Email}`; Subject "Cape Clarity — Your feedback as you finish
     your care with us"; Body's link ends
     `?token=${issueFeedbackToken_1.token}`, where the base URL is the
     "Cape Clarity Discharge Feedback" form's Share-tab permalink,
     DOM-verified (§1): `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDischargeFeedback/formperma/o0I33pZYZMvoDPrhFbRH2LxvW3Jw9MDWw4foW7rXtDA`.
   - **On Error of `issueFeedbackToken`** → `alertIssuanceFailed` (Zoho Mail
     "Send email"): connection/From same as above; To `costin@capeclarity.com`;
     Subject "Cape Clarity feedback pipeline: token issuance failed
     (4 - Discharge)"; body verbatim from
     `specs/003-issuance-privacy-and-failure-handling/data-model.md`'s
     `alertIssuanceFailed` template, `<flow>` = "M4 - Discharge Trigger",
     `<milestone>` = "4 - Discharge".
   - **On Error of the patient "Send email"** → `markSendFailed` (Zoho CRM
     "Update module entry"): module `Milestone_Instances`; Entry Id =
     `${issueFeedbackToken_1.recordId}`; `Status` = "Use a Custom Value" =
     `Send Failed` (typed into the Custom Value field even though "Send
     Failed" also exists as a real picklist option in this module — matches
     `m4-data-model.md`'s spec literally and M2/M3's own convention) → then
     `alertSendFailed` (Zoho Mail "Send email"): To `costin@capeclarity.com`;
     Subject "Cape Clarity feedback pipeline: email send failed
     (4 - Discharge)"; body verbatim from the same `data-model.md`'s
     `alertSendFailed` template (includes `${issueFeedbackToken_1.recordId}`,
     no `${trigger.*}` fields), `<flow>`/`<milestone>` as above.

Flow left **OFF** after this build, per every prior milestone's
build-first-verify-later convention. No live/end-to-end test was run — every
step above was verified structurally (screenshots of each node's saved
config, plus the connector-wiring checks in §2.1) rather than by executing
the flow, per the standing "no live/end-to-end tests without the user"
instruction.

### 2.1 Builder gotcha (new this session): the reliable way to wire a second On-Error branch when the source node's normal output is already occupied

`specs/003-issuance-privacy-and-failure-handling/implementation-notes.md` §4
documents two methods for landing a new node on an existing node's On Error
port: dragging a brand-new node from the palette onto the source node's body
(works when the source's normal output is still empty), and a "placeholder
trick" for when the normal output is already occupied. Building this flow's
*second* On-Error branch (patient "Send email" → `markSendFailed`, with
`Send email`'s normal output already wired to a placeholder) surfaced a
refinement worth recording:

- **Dragging a new node from the palette onto the occupied source node's
  body (or directly onto its red On Error circle) did not reliably create a
  connection** — repeated attempts (varying the exact drop coordinate, and
  the canvas zoom level from 76% up to 106%) all produced a node that only
  *visually* overlapped the source node, confirmed by dragging the new node
  away afterward and finding no connecting line at all. This happened
  consistently, unlike the first On-Error branch (`issueFeedbackToken` →
  `alertIssuanceFailed`, built in an earlier session), where the same
  palette-onto-body technique worked on the first try — the difference
  isn't yet fully explained (both source nodes had the same "normal output
  already occupied" precondition), but zoom level and exact drop pixel do
  not explain it either, since both were varied without success.
- **What actually worked**: place the new node anywhere empty on the canvas
  first (a plain palette drop with no target node under it, which always
  lands cleanly as a disconnected node), then draw the connection in
  reverse — `left_click_drag` starting **from the source node's own On Error
  red circle** and ending **on the disconnected target node's body**. This
  produced a real wired connection every time (confirmed both by the red
  "On Error" label rendering on the connecting line, and by reopening the
  target node's config afterward and checking that its Insert Variable panel
  now lists every node upstream of the source, including the source itself).
  This is the opposite direction from `specs/003`'s "placeholder trick",
  which drags a palette node onto the source; dragging from the source's own
  existing red port to an already-placed node is what worked here. Worth
  trying this direction first on a future milestone before reaching for the
  palette-onto-body technique, if the first attempt at that doesn't wire the
  way a screenshot suggests it should.
- Also reconfirmed `specs/003`'s existing warning that dragging between two
  circles on an *already-placed, disconnected* node can misfire: one attempt
  during this session landed the new node's connection on
  `isCreatedAfterCutoff`'s (unrelated, previously-empty) On Error port
  instead of the intended source, silently, with no error — caught only by
  reopening the new node's config and noticing `issueFeedbackToken` and
  `Send email` were missing from its Insert Variable panel where they should
  have appeared. Always verify a newly-wired On-Error connection this way
  (reopen the target's config, check which upstream nodes appear in Insert
  Variable) rather than trusting the canvas screenshot alone.
- **Dot-notation on a custom function's `Map` output inside a structured
  (non-rich-text) field**: the `Entry Id` field on `markSendFailed` (type
  `number`) uses a token-chip formula editor, not a plain text box. Typing
  the full merge expression as literal text (`${issueFeedbackToken_1.recordId}`)
  did not parse — it landed as raw, uninterpreted characters. What worked:
  click the variable in the Insert Variable panel to insert it as a proper
  chip (`issueFeedbackToken_1`), then click to the end of that chip and type
  `.recordId` as plain text immediately after it (no `${...}` wrapper,
  no re-opening the panel) — this matches the "Use '.' to access nested
  variables" example the field's own formula-help popover shows
  (`trigger.full_name`). This is a different mechanism from the plain-text/
  rich-text merge-field fields (email Body, CRM text fields), where the
  literal `${issueFeedbackToken_1.recordId}` string convention documented in
  `CLAUDE.md` is typed directly with no chip involved — the two syntaxes
  look similar but are not interchangeable between field types.

## 3. "M4 - Discharge Write-back" flow, step by step (T008-T009/T014-T015 — built)

Built as a **brand-new flow** (Create flow → App trigger → configure), not
cloned from any other milestone's write-back flow — same reasoning as every
prior milestone: avoids the shared-custom-function-across-clones gotcha
`m2-implementation-notes.md` §3.2 documents. New flow name
"M4 - Discharge Write-back", placed in the "Customer Feedback System" folder
alongside the other flows (this brings the folder to 9 flows total, M0-M4
each with a trigger + write-back pair except M0/M1 which only issue tokens
via subflow/one-off, matching this project's file listing).

1. **DOM verification (done first, before wiring anything)**: navigated to
   the "Cape Clarity Discharge Feedback" form's live public permalink
   (§1's DOM-verified link) in a scratch tab and queried the DOM directly
   (`document.querySelectorAll('input, textarea')` plus a slider-specific
   query keyed on `div.ui-slider`) rather than assuming field names from the
   builder's on-screen order. This confirmed the internal Zoho Forms field
   names — resolving the outstanding flag from §1 — by cross-referencing
   each `sld-<name>` slider's DOM id against its nearest `label`/instructions
   text:

   | Internal field name | On-screen question (abbreviated) | Write-back parameter |
   |---|---|---|
   | `Slider` | "...open and honest with your therapist..." (Connection) | `connection` |
   | `Slider1` | "...therapist has understood..." (Understanding) | `understanding` |
   | `Slider2` | "...agree on what you're working toward..." (Shared direction) | `sharedDirection` |
   | `Slider3` | "...therapist's approach been working..." (Fit of approach) | `fitOfApproach` |
   | `Slider4` | "How likely are you to recommend..." (Likelihood to recommend) | `likelihoodToRecommend` |
   | `MultiLine` | "Is there anything else..." (Anything else looking ahead) | `anythingElse` |
   | `SingleLine` | Token (hidden) | `token` |

   Confirms `m4-data-model.md`'s form field inventory maps to sliders
   `Slider`-`Slider4` in on-screen order (Alliance Check-In domains first,
   then the two new Looking Ahead fields last) — i.e. the internal names are
   assigned in on-screen order, not in `Response_Data` blob order, which is
   why this mapping table (by internal name) is a separate, necessary check
   from the blob's segment order.

2. **Trigger** — Zoho Forms "Form entry submitted" (Realtime). Connection:
   "Connection to Cape Clarity Zoho Forms" (pre-selected by default). Form:
   "Cape Clarity Discharge Feedback". Output variable left as the default
   `trigger`. No filter criteria (mirrors M2/M3's write-back triggers).
3. **Custom Function** — `submitDischargeFeedbackResponse`, created from
   scratch via Built-ins → Developer Tools → Custom Functions →
   "+Custom Function" (not cloned). Return type `map`; seven string input
   parameters, in this order: `token`, `likelihoodToRecommend`,
   `anythingElse`, `connection`, `understanding`, `sharedDirection`,
   `fitOfApproach`. Full source matches `m4-data-model.md`'s illustrative
   draft verbatim (confirmed against the live Deluge editor by setting the
   CodeMirror instance's value directly via the browser's JS console and
   visually verifying the rendered, syntax-highlighted result before
   saving — an alternative to typing character-by-character that avoided any
   risk of the editor's auto-indent/auto-bracket behavior corrupting a
   multi-line paste). The function's own multi-segment string concatenation
   (Looking Ahead segments first, then the 4 reused alliance domains) is
   exactly as specified in `m4-data-model.md`'s Response_Data blob format
   section — verified line-by-line via screenshot after saving.
4. **Placing and wiring the function node**: dragged the saved function from
   the Built-ins sidebar list onto empty canvas space below the trigger; this
   single drag-and-drop did **not** auto-wire the connection (confirmed via
   the DOM connector-element count being `0` immediately after the drop —
   unlike M2's build, where the equivalent drag-and-drop both placed and
   auto-wired the node in one action). Wired manually by dragging from the
   trigger's own output circle to the function node's body, which produced a
   real connection (connector count `2` = 1 wired connection, confirmed via
   `document.querySelectorAll('[class*="connector" i]').length`).
5. **Parameter mapping**: unlike M2's brand-new, never-run trigger (whose
   Insert Variable panel showed no field-level entries), this trigger's
   Insert Variable panel **did** surface per-field variables immediately,
   listed by friendly on-screen question text rather than internal field
   name. Mapped each parameter by clicking into its input field, then
   clicking the matching friendly-label entry in the Insert Variable panel
   (inserting a `Form entry submitted → <label>` chip) — cross-checked
   against the DOM-verified internal-name table in step 1 above rather than
   relying on label text alone, since two of the "Overall, how..." labels
   are easy to transpose at a glance:

   | Function parameter | Insert Variable label clicked |
   |---|---|
   | `token` | Token |
   | `likelihoodToRecommend` | How likely are you to recommend Cape Clarity to someone in a similar situation? |
   | `anythingElse` | Is there anything else you'd like to share as you finish up your care with us? |
   | `connection` | Overall, how comfortable have you felt being open and honest with your therapist so far? |
   | `understanding` | Overall, how well do you feel your therapist has understood what matters to you so far? |
   | `sharedDirection` | Overall, how much do you feel you and your therapist agree on what you're working toward? |
   | `fitOfApproach` | Overall, how well has your therapist's approach been working for you so far? |

   Each mapping was visually confirmed via screenshot immediately after
   clicking (the chip renders inline in the parameter field showing the full
   `Form entry submitted → <label>` text) before moving to the next field.

Flow ends at the function node (no further steps) — matches M2/M3's
write-back flow shape exactly (`m2-implementation-notes.md` §3.4: trigger →
function, no email or CRM update node beyond what the function itself does).
Left switched **OFF** after building, matching every prior milestone's
build-first-verify-later convention. No live/end-to-end test was run.

### 3.1 Builder gotcha (new this session): auto-wire is not guaranteed on function-node drop

M2's build (`m2-implementation-notes.md` §3.4) found that dragging a saved
custom function from the Built-ins sidebar onto the canvas near a trigger
both placed *and* auto-wired the connection in one action. This session's
equivalent drag did **not** auto-wire — the function landed as a fully
disconnected node (DOM connector count `0`). The fix was the same
manual-wiring technique documented in §2.1 for On-Error branches: drag from
the upstream node's own output circle to the new node's body. Lesson: always
verify connector count (or reopen the downstream node's Insert Variable
panel and check the upstream node appears) after a sidebar-to-canvas
function drop — don't assume auto-wire happened just because it did on a
previous milestone.

## 4. Shared Clinical Safety Flag gate and "Flagged for Review" report widened to M4 (T017-T019)

Per `m4-research.md` Decision 6 and `m4-data-model.md`, both shared,
M2/M3-owned Analytics objects on the "Milestone Instances" table
(workspace `3251423000000083002`) were widened to also cover M4, following
the same "one rule, one report, several milestones" pattern M3 established
against M2. Both changes made in Zoho Analytics' Edit Design UI, not by
forking parallel M4-only objects.

- **"Clinical Safety Flag" formula column (T017)**: the `Milestone` gate
  widened from M2-or-M3 to M2-or-M3-or-M4 (infix `OR`, per the
  Analytics-formula-language finding already documented in
  `m3-implementation-notes.md` §1.1). Edited via the "Edit Formula Column"
  dialog's CodeMirror 6 editor (no global `setValue` API — done via click
  positioning + arrow-key navigation, verified with zoomed before/after
  screenshots and by re-reading `.cm-content.textContent` after reopening
  the dialog). Live formula as of 2026-09-24:
  ```
  IF("Milestone Instances"."Milestone" = '2 - Early Alliance Check' OR "Milestone Instances"."Milestone" = '3 - Periodic Consolidated' OR "Milestone Instances"."Milestone" = '4 - Discharge', IF("Milestone Instances"."Alliance Check-In Total" <= 20 OR to_integer("Milestone Instances"."Domain: Connection") <= 4 OR to_integer("Milestone Instances"."Domain: Understanding") <= 4 OR to_integer("Milestone Instances"."Domain: Shared Direction") <= 4 OR to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, 'true', 'false'), '')
  ```
  **"Flag Rule Triggered" inspected, no change needed**: it keys off
  `Clinical Safety Flag = 'true'` only, with no Milestone-specific logic of
  its own, so it inherits the M4 gate automatically. Dialog opened to
  confirm this, then cancelled without saving.
- **"M2 & M3 Flagged for Review" report (T019)**: renamed to "M2, M3 & M4
  Flagged for Review" via the native-input-setter technique
  (`Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,
  'value').set.call(el, '<new title>')`, dispatch `input`, then Return —
  the workaround `m3-implementation-notes.md` §1.3 documents for Zoho
  Analytics' report-title text-corruption bug); no corruption observed.
  The `Milestone` Wildcard filter gained a third OR'd condition, Exactly
  Matches `"4 - Discharge"`, added via "Edit Design" → Filters tab → the
  "+" button next to the last condition (Criteria Expression auto-updated
  to `(1 OR 2 OR 3)`). The report's second filter, `Clinical Safety Flag`
  Exactly Matches `"true"`, was inspected and needs no change. Both the
  rename and the filter widening were verified to persist across a full
  page reload. View ID unchanged: `3251423000000141219`.
- **T018 (confirm-absence)**: no automated notification or CRM action
  reaches a contractor when an M4 reading fires the Clinical Safety Flag.
  Reviewed both of M4's own flows (§2, §3): the trigger flow's On-Error
  branches alert only `costin@capeclarity.com`, and the write-back flow
  only updates `Response_Data`/`Status`/`Submitted_Date_Time` on the
  Milestone Instance record — no notification step of any kind exists in
  either flow.

`m2-implementation-notes.md` §9.2/§9.6 and `m3-implementation-notes.md`
§1.1/§1.3 have been updated in the same session to record this change, per
CLAUDE.md's cross-milestone convention.

## 5. Not yet built

Per `m4-tasks.md`'s task list:
- **Analytics**: 2 new formula columns (Looking Ahead: Likelihood To
  Recommend, Looking Ahead: Anything Else) — M4's own, not shared.
- **Reporting**: M4 Submitted Responses, M4 Status Breakdown, M4 Looking
  Ahead: Likelihood To Recommend Distribution, M4 dashboard.
- **T023 / test data**: blocked pending a test Patient ID from Costin, per
  standing instruction — not pursued proactively.
- Confirmation from Costin/Liana on whether `Patient_Status = "Completed
  Treatment"` is the correct discharge-trigger filter value (flagged
  non-blocking in `m4-research.md` Decision 1).

## 6. Access constraint compliance

All form-building, flow-building, and Analytics work this session used the
Zoho Forms, Zoho Flow, and Zoho Analytics builders directly (not the CRM's
Leads/Patients modules), so the standing "don't open CRM Leads/Patients
modules without permission" constraint did not apply to this phase. No CRM
Leads or Patients records were viewed or edited in the browser during the
form, trigger-flow, write-back-flow, or Analytics build.
