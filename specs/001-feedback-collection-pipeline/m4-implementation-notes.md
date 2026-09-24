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
> in the same session per `m4-data-model.md`'s Decision 6. T020-T022 (M4's
> own new Analytics: 2 formula columns, 3 reports, and the "M4 - Discharge
> Feedback" dashboard) are also built and verified structurally — see §5.
> The discharge-trigger filter value (`Patient_Status = "Completed
> Treatment"`) is confirmed correct by Costin (2026-09-24) — see §2.
> Remaining work (T023-T027 live test data) is listed in §6.

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
   Treatment` — **confirmed correct by Costin 2026-09-24** (`m4-research.md`
   Decision 1); no change needed.
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

## 5. M4's own Analytics: formula columns, reports, and dashboard (T020-T022)

Per `m4-tasks.md` Phase 6 (Reporting) and `m4-data-model.md`'s Analytics
formula columns/Reporting sections, built M4's own (not shared) Analytics
objects on the "Milestone Instances" table (workspace
`3251423000000083002`).

- **"Looking Ahead: Likelihood To Recommend" formula column**:
  ```
  substring_between("Milestone Instances"."Response Data", 'Looking Ahead: Likelihood To Recommend (0-10): ', '---', 1)
  ```
- **"Looking Ahead: Anything Else" formula column**:
  ```
  substring_between("Milestone Instances"."Response Data", 'Looking Ahead: Anything Else: ', '---', 1)
  ```
  Both added via Add → "Add Formula: Formula Column" (a different dialog
  from the "Edit Formula Column" one §4 used, but same CodeMirror 6
  `.cm-content` editor underneath). Both verified Text-typed (`T` icon in
  the raw table view header) and present via "Edit Formulas and Buckets" →
  search. **Gotcha**: the second "Add Formula Column" dialog (for "Anything
  Else") silently failed to register typed input on the first attempt —
  both the Name field and the formula editor read back empty via
  `document.querySelector('.cm-content').textContent` even though the
  `type` actions reported success. Fix: retried the identical
  click-then-type sequence; the retry worked and was verified via the same
  JS read before saving. Because both new columns are Text-typed, the
  distribution report below needed no `Dimension [Actual(D)] / Treat as
  Text` re-set (same precedent as `m3-implementation-notes.md` §1.8).

- **"M4 Submitted Responses" report**: new Tabular View (not a Save-As —
  its 11-column shape and column order don't match any existing report).
  Columns, in order: Submitted Date Time, Token, Looking Ahead: Likelihood
  To Recommend, Looking Ahead: Anything Else, Domain: Connection, Domain:
  Understanding, Domain: Shared Direction, Domain: Fit Of Approach,
  Alliance Check-In Total, Clinical Safety Flag, Flag Rule Triggered (no
  Patient/identity column, matching M2/M3's privacy pattern). Filtered to
  `Milestone` Wildcard Exactly Matches `"4 - Discharge"`. Saved to "Zoho
  CRM Modules (Data)". View ID `3251423000000232092`. Verified in View
  Mode: correct headers, 0 rows (expected).

  **New gotcha, not previously documented in any milestone's notes**: in
  the Tabular View builder, checking a column's checkbox in the left
  "Select/Drag and Drop the Columns" panel always adds it as a *tabular
  column*, even while the "Filters" tab is active — the UI silently
  auto-switches back to the "Tabular" tab to show the newly-added column.
  To add a column as a *filter* instead, you must switch to the "Filters"
  tab first and then **drag** the column (by its checkbox/label) onto the
  "Drop your columns here" filter drop zone; the checkbox click does not
  work there. Caught this by accidentally adding "Milestone" as a 12th
  tabular column, removing it via its "X", and discovering the drag
  requirement. A second, related slip: after clearing the column search
  box (which resets the left panel's scroll/row order), a stale
  drag-source coordinate landed on "Response Data" instead of "Milestone" —
  caught immediately (wrong filter appeared), removed, and fixed by
  re-searching "Milestone" and re-confirming its row position via
  screenshot immediately before each drag.

- **"M4 Status Breakdown" report**: Save As off "M3 Status Breakdown"
  (`m3-implementation-notes.md` §1.8, view ID `3251423000000186076`), same
  bar chart config preserved (X-Axis `Status` Actual, Y-Axis `Id` Count).
  Only change: Milestone Wildcard filter value swapped from `"3 - Periodic
  Consolidated"` to `"4 - Discharge"`. Saved to "Zoho CRM Modules (Data)".
  View ID `3251423000000232098`. Regenerated graph shows "No Data
  Available" (expected).

- **"M4 Looking Ahead: Likelihood To Recommend Distribution" report**: Save
  As off "M2 Alliance Check-In Total Distribution"
  (`m2-implementation-notes.md` §9.5, view ID `3251423000000141097`), then
  X-Axis column swapped from "Alliance Check-In Total" to "Looking Ahead:
  Likelihood To Recommend" and the Milestone Wildcard filter value swapped
  from `"2 - Early Alliance Check"` to `"4 - Discharge"`. Y-Axis unchanged:
  `Id` (Count). Confirmed the X-Axis dropdown shows plain `Actual` (not
  `Actual(M)`) immediately after the column swap, so — consistent with
  §M3's precedent — no explicit `Treat as Text` re-set was needed. Saved to
  "Zoho CRM Modules (Data)". View ID `3251423000000232114`. Regenerated
  graph shows "No Data Available" (expected). Note: the Save-As source,
  "M2 Alliance Check-In Total Distribution", currently renders one bar
  labeled "Unknown" with count 1 rather than the "No Data Available" state
  `m2-implementation-notes.md` §9.5 documents when it was built — not
  investigated (Save As produces an independent copy regardless of the
  source's current data state, so this didn't block anything), flagged here
  only in case it's relevant to a future session.

- **"M4 - Discharge Feedback" dashboard**: new dashboard, view ID
  `3251423000000232202`, built via "Create New Dashboards" (not Save As —
  same reasoning M1/M2/M3 gave: no exact panel-for-panel match to copy
  from). Bundles exactly four reports, dragged in from the Reports panel
  (searched by exact name, one at a time, to sidestep the alphabetical
  re-flow mishap `m3-implementation-notes.md` §1.8 documents for
  multi-report dashboard builds):
  - M4 Status Breakdown (this section)
  - M4 Looking Ahead: Likelihood To Recommend Distribution (this section)
  - M4 Submitted Responses (this section)
  - M2, M3 & M4 Flagged for Review (§4, shared across M2/M3/M4)

  Left at "Auto Add User Filters" on and "Make User Filters Global" off,
  matching M0/M1/M2/M3's dashboard defaults. All four panels verified
  rendering correctly in View Mode: the two chart panels (Status Breakdown,
  Likelihood To Recommend Distribution) show "No Data Available" and the
  two tabular panels (Submitted Responses, Flagged for Review) show their
  correct column headers with 0 rows — all expected, since no real M4
  submissions exist yet.

- **T020 (verify reused alliance-domain columns/Alliance Check-In Total
  parse M4 rows correctly)**: this is a live-data confirmation per the task
  text, not something to force structurally without real M4 submissions —
  deferred to the live test pass (§6, T023).

## 6. Not yet built

Per `m4-tasks.md`'s task list:
- **T023-T027 / live test data**: blocked pending a test Patient ID from
  Costin, per standing instruction — not pursued proactively. Includes
  confirming T020 (reused columns parse M4 rows correctly) against real
  data.

## 7. Access constraint compliance

All form-building, flow-building, and Analytics work this session used the
Zoho Forms, Zoho Flow, and Zoho Analytics builders directly (not the CRM's
Leads/Patients modules), so the standing "don't open CRM Leads/Patients
modules without permission" constraint did not apply to this phase. No CRM
Leads or Patients records were viewed or edited in the browser during the
form, trigger-flow, write-back-flow, or Analytics build.
