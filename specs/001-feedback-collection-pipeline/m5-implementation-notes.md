# M5 (Discontinuation) — Implementation Notes

> Scope: this file documents ONLY the M5 milestone (Discontinuation —
> one-time-per-patient, single-condition trigger `Patient_Status =
> "Discontinued (Patient Choice)"`, per Costin's explicit decision to drop
> the "No Show" leg — see `m5-research.md`'s "Revision (same day)" note). It
> is a companion to `specs/001-feedback-collection-pipeline/spec.md` (all six
> milestones) and to `m5-plan.md` / `m5-research.md` / `m5-data-model.md` /
> `m5-tasks.md`. This file tracks what has actually been built in Zoho, as
> it's built, per the `CLAUDE.md` convention of updating implementation notes
> in the same session as the change.
>
> Last updated: 2026-09-24
> Status: **T001 (the "Cape Clarity Discontinuation Feedback" Zoho Form, all
> 3 fields + thank-you page) is built and verified by screenshot after each
> field save — see §1. One deviation from the sourced closing line, forced
> by a builder character limit, is recorded in §1.1 — flagged for Costin's
> awareness at Checkpoint 2, not silently absorbed.** Phase 2 is also built
> and verified structurally: the **"M5 - Discontinuation Trigger"** flow
> (single-condition trigger, both On Error branches) — see §2 — and the
> **"M5 - Discontinuation Write-back"** flow (`submitDiscontinuationFeedbackResponse`)
> — see §3. Both flows are left **OFF**; no live/end-to-end test was run.
> Checkpoint 3 (both flows built) was closed by Costin's "let's move to
> analytics" instruction. **Phase 7 (T019-T021: 2 Analytics formula columns,
> 4 reports, 1 dashboard) is also now built and verified structurally — see
> §4, including one naming deviation (the "Okay To Reach Back Out" formula
> column was renamed to avoid a case-insensitive collision with M0's existing
> column of nearly the same name).** This is Checkpoint 4 per Costin's
> standing checkpoint list — paused here for his review of the Analytics
> reporting/dashboard before Phase 8 (live test data).

## 1. "Cape Clarity Discontinuation Feedback" Zoho Form (T001)

Built fresh in Zoho Forms
(`https://forms.zoho.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedback/builder`),
Standard form type, matching `m5-data-model.md`'s field inventory. On-screen
order, top to bottom, confirmed via screenshot after each field was added and
saved (each field was dropped directly below the previous one and landed in
the intended position on the first attempt — no reordering needed):

| Order | Field type | Label (on-screen question text) | Choices | Mandatory | Visibility |
|---|---|---|---|---|---|
| 1 | Dropdown | What led to stepping away from sessions right now? | Felt better / reached their goals; Scheduling or timing conflict; Cost or insurance; Didn't feel like the right fit with their therapist; Life circumstances changed; Moved or relocated; Choosing to pause for now; Something else (8 total, source order preserved) | Yes | Show |
| 2 | Radio | Would it be okay to reach back out if things change? | Yes / No / Maybe | Yes | Show |
| 3 | Single Line | Token | — | No | **Hide** |

No name, email, or phone field (Principle I). No open-text field (per the
source Confluence doc's own explicit "no open text" design note, recorded in
`m5-research.md` Decision 4).

Builder mechanics notes (Zoho Forms, consistent with prior milestones):
choice-list editing happens in a separate "Choice Field Properties"
sub-panel opened via an "Edit" button under the Properties panel's Choices
section, not inline on the main panel; the Properties panel occasionally
rendered at less than full width immediately after opening (choice/field-
label text visually truncated) — this resolved itself after the next
click/interaction and did not affect what was actually saved, confirmed each
time via a follow-up screenshot before proceeding. One transient "Browser
extension is not connected" disconnection occurred mid-build; a retry of the
tab-context call reconnected it, and a screenshot was taken to confirm actual
page state before re-issuing the interrupted action (no double-submission or
lost input resulted).

### 1.1 Deviation: thank-you page text shortened for a 100-character cap

`m5-data-model.md` and the source Confluence doc specify the closing line
verbatim as:

> "Thank you for letting us know. We wish you well, and we're here whenever
> you're ready, if you ever are." (103 characters)

Zoho Forms' Thank You Page field enforces a hard 100-character maximum in
both Plain Text and Rich Text mode (confirmed by attempting Rich Text — same
cap applies there too). The sourced line is 3 characters over. Rather than
guess at a rewrite, the single lowest-impact cut was made: dropping "ever"
from the final clause. As built:

> "Thank you for letting us know. We wish you well, and we're here whenever
> you're ready, if you are." (98 characters)

This is a builder-forced deviation from the verbatim-sourced copy, not a
content judgment call — flagged here for Costin's awareness at Checkpoint 2.
If a different shortening is preferred, this is a one-field edit in Zoho
Forms Settings → General → "Thank You Page & Redirection".

The default "Include a link to allow respondents to add another response"
checkbox was left checked (Zoho Forms' default); not evaluated against any
milestone-specific requirement — flagging in case Costin wants it unchecked
for a one-time discontinuation survey.

## 2. "M5 - Discontinuation Trigger" flow, step by step (T002-T007 — built)

Built fresh (not cloned) via Zoho Flow's builder
(`https://flow.zoho.com/#/workspace/872426000000002011/flows/m5_discontinuation_trigger/edit`),
following the token-issuance-without-subflows convention and the
legacy-patient cutoff-exclusion convention exactly as `CLAUDE.md` documents
them:

1. **Trigger** — Zoho CRM "Updated module entry". Connection: CRM Connection.
   Module: `Patients`. Filter: `Patient Status` `equals` (case-insensitive)
   `Discontinued (Patient Choice)` — single-condition only, per Costin's
   explicit decision to drop the "No Show" leg (`m5-research.md`'s
   "Revision (same day)" note; see this file's header).
2. **Custom Function** — `checkDiscontinuationExists(patientId)`, created
   fresh (not cloned), querying `Milestone_Instances` for an existing
   `Milestone == "5B - Discontinuation, Email Fallback"` record for that
   patient. Output variable `checkDiscontinuationExists_1` (boolean). Input
   `patientId = ${trigger.id}` (chip: `Updated module entry → Entry ID`).
3. **Custom Function** — `isCreatedAfterCutoff(createdTime)`, the shared
   function from `specs/004-legacy-patient-exclusion` (reused, not
   recreated). Output variable `isCreatedAfterCutoff_1` (boolean). Input
   `createdTime = ${trigger.Created_Time}` (chip: `Updated module entry →
   Time created`).
4. **If else** — condition: `checkDiscontinuationExists_1 is false` **AND**
   `isCreatedAfterCutoff_1 is true`.
   - **True branch → `issueFeedbackToken`** (shared, unmodified): `milestone`
     = `5B - Discontinuation, Email Fallback` (typed literal), `patientId` =
     chip `Updated module entry → Entry ID`, `leadId` empty, `recipientEmail`
     = chip `Updated module entry → Email`, `clinician` = `Liana Preudhomme`
     (typed literal), `ttlDays` = `7` (typed literal). Output variable
     renamed `issueFeedbackToken_1`.
   - **Then → Zoho Mail "Send email"** (patient-facing): connection
     "Connection to info@capeclarity.com"; From `info@capeclarity.com`; To =
     chip `Updated module entry → Email`; Subject `Just checking in`
     (dropping `[First Name]` per §2.1); Body sourced verbatim from
     `m5-research.md` Decision 4 (opening "Hi there," in place of a
     personalized greeting), with a "Share a quick note →" hyperlink to
     `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedback/formperma/6h7a2uGbEbrORj9hjp_eIKb5eV9ocY4ctW2efXxHXmw?token=${issueFeedbackToken_1.token}`.
   - **On Error of `issueFeedbackToken`** → Zoho Mail "Send email" (alert):
     connection/From same as above; To `costin@capeclarity.com`; Subject
     "Cape Clarity feedback pipeline: token issuance failed (5B -
     Discontinuation, Email Fallback)"; body names the milestone but includes
     no `${trigger.*}` identity fields.
   - **On Error of the patient "Send email"** → Zoho CRM "Update module
     entry": module `Milestone_Instances`; Entry Id = chip
     `issueFeedbackToken` + literal `.recordId` (same "insert chip, then type
     `.recordId` as plain text immediately after it" technique
     `m4-implementation-notes.md` §2.1 documents for this field type);
     `Status` = "Use a Custom Value" = `Send Failed` (an existing picklist
     value on this module, matching M2/M3/M4's convention) → then Zoho Mail
     "Send email" (alert): To `costin@capeclarity.com`; Subject "Cape Clarity
     feedback pipeline: email send failed (5B - Discontinuation, Email
     Fallback)"; body includes `${issueFeedbackToken_1.recordId}` (chip +
     literal `.recordId`, same technique), no `${trigger.*}` fields.

Flow left **OFF** after this build. No live/end-to-end test was run — every
step above was verified structurally (screenshots of each node's saved
config, plus zoomed connector-wiring checks) rather than by executing the
flow.

### 2.1 `[First Name]` fallback confirmed live (T007a)

`m5-data-model.md` / `m5-research.md` flagged the sourced email subject as
containing a `[First Name]` personalization placeholder with an explicit
fallback instruction if no first-name-only field exists on the trigger
module. Checked live via the Insert Variable panel on the patient-facing
"Send email" node: the `Patients1` / trigger module exposes only "Full Name
(PHI)" and "Patient Name" — no first-name-only field. Applied the documented
fallback: dropped `[First Name]` from the subject (`Just checking in`) and
opened the body with "Hi there," instead of a personalized greeting. No
further action needed.

### 2.2 Builder gotcha (new this session): a node dropped in free canvas space can land pre-wired to the wrong node's On Error port

The second On-Error branch's "Update module entry" node was dropped in a
prior session onto open canvas space (not by dragging from any node's own
port), on the assumption this would leave it disconnected pending manual
wiring — matching the usual "drop free, then reverse-drag from the source's
own port" technique `specs/003-issuance-privacy-and-failure-handling/implementation-notes.md`
§4 and `m4-implementation-notes.md` §2.1 document. On resuming this session,
the node was found **already wired** — but to the wrong source: the
`issueFeedbackToken` On-Error alert email's own On-Error port, not the
patient-facing "Send email" node's On-Error port. (Both nodes' On-Error
ports render close together on canvas when the alert email is positioned
directly above the patient email, which is likely how the mis-wire went
undetected before compaction.) Caught by zooming into the connector layout
and tracing each line's exact endpoints rather than trusting node proximity.
**Fix**: selected the errant connector (single click on the line itself,
not the node) — this highlights it and reveals a scissors/delete icon
mid-line — clicked it, confirmed "Are you sure you want to detach the wire?"
in the resulting dialog, then re-wired correctly via the standard reverse-
drag from the patient "Send email" node's own On-Error red circle. **Lesson
for future sessions**: after resuming a half-built On-Error chain, always
re-verify each connector's actual endpoints by zoomed screenshot before
configuring or trusting a node that appears pre-wired — proximity on canvas
is not evidence of correct wiring.

## 3. "M5 - Discontinuation Write-back" flow, step by step (T008-T009 — built)

Built as a **brand-new flow** (Create flow → App trigger → configure), not
cloned from any other milestone's write-back flow — same reasoning as every
prior milestone (avoids the shared-custom-function-across-clones gotcha
`m2-implementation-notes.md` §3.2 documents). New flow name
"M5 - Discontinuation Write-back", placed in the "Customer Feedback System"
folder alongside the other flows.

1. **Trigger** — Zoho Forms "Form entry submitted" (Realtime). Connection:
   "Connection to Cape Clarity Zoho Forms" (pre-selected by default). Form:
   "Cape Clarity Discontinuation Feedback". Output variable left as the
   default `trigger`. No filter criteria (mirrors M2/M3/M4's write-back
   triggers).
2. **Custom Function** — `submitDiscontinuationFeedbackResponse`, created
   from scratch via Built-ins → Developer Tools → Custom Functions →
   "+Custom Function" (not cloned). Return type `map`; three string input
   parameters, in this order: `token`, `reasonForLeaving`,
   `okayToReachBackOut` — the shortest write-back function of any milestone
   (2 `---`-delimited Response_Data segments, no numeric/scored content).
   Full source matches `m5-data-model.md`'s illustrative draft verbatim
   (same token-lookup / `Status != "Issued"` rejection / expiry-check-and-
   auto-expire structure as every prior write-back function), set via the
   CodeMirror instance's `setValue()` JS technique and visually verified
   (syntax-highlighted, correctly indented) before saving.
3. **Placing and wiring the function node**: dragged the saved function from
   the Built-ins sidebar list onto empty canvas space below the trigger —
   did not auto-wire (landed as a disconnected node, matching the "auto-wire
   is not guaranteed" gotcha `m4-implementation-notes.md` §3.1 documents).
   Wired manually by dragging from the trigger's own output circle to the
   function node's body.
4. **Parameter mapping**: the trigger's Insert Variable panel surfaced
   per-field variables immediately, listed by on-screen question text
   (matching the form's 2 substantive fields + hidden Token, per §1's field
   table):

   | Function parameter | Insert Variable label clicked |
   |---|---|
   | `token` | Token |
   | `reasonForLeaving` | What led to stepping away from sessions right now? |
   | `okayToReachBackOut` | Would it be okay to reach back out if things change? |

   Each mapping confirmed via screenshot immediately after clicking (chip
   renders inline as `Form entry submitted → <label>`). Output Variable Name
   left at its default, `submitDiscontinuationFeedbackResponse_1` — already
   matches the per-flow shared-function-output convention with no rename
   needed.

Flow ends at the function node (no further steps) — matches M2/M3/M4's
write-back flow shape exactly (trigger → function, no email or CRM update
node beyond what the function itself does). Left switched **OFF** after
building. No live/end-to-end test was run.

## 4. M5's own Analytics: formula columns, reports, and dashboard (T019-T021)

Per `m5-tasks.md` Phase 7 (Reporting) and `m5-data-model.md`'s Analytics
formula columns/Reporting sections, built M5's own (not shared) Analytics
objects on the "Milestone Instances" table (workspace
`3251423000000083002`), reached directly at
`https://analytics.zoho.com/workspace/3251423000000083002` (already signed
in as `liana.preudhomme@capeclarity.com`). Per `m5-research.md` Decision 6,
no Clinical Safety Flag gate changes were needed for M5 (the Alliance rule
and its domains don't apply to M5's data) — confirmed by inspection, no
edit made.

- **"Reason For Leaving" formula column**:
  ```
  substring_between("Milestone Instances"."Response Data", 'Reason For Leaving: ', '---', 1)
  ```
  Added via Add → "Add Formula: Formula Column", matching `m5-data-model.md`'s
  spec verbatim. Saved without incident; Text-typed (`T` icon confirmed in
  the raw table view header) and present via "Edit Formulas and Buckets" →
  search.

- **"Discontinuation: Okay To Reach Back Out" formula column** — **named
  differently from `m5-data-model.md`'s literal spec ("Okay To Reach Back
  Out"), a deviation forced by a naming collision, not a content choice**:
  M0's own Analytics build (a prior milestone, not part of this file's
  scope) already has a formula column named "Okay to Reach Back Out"
  (lowercase "to"), which is *bounded* (`substring_between(...,
  'Okay to reach back out: ', '---', 1)` — M0 has further Response_Data
  segments after this field, unlike M5, where it's the last segment).
  Attempting to save a new column named "Okay To Reach Back Out" (title-case
  "To") failed with "Column Name already exists" — Zoho Analytics' formula
  column names are apparently case-insensitive for uniqueness purposes, even
  though the on-screen labels differ in case. Rather than overwrite or
  rename M0's existing column (out of scope, and still needed by M0's own
  reports), the new M5 column was named **"Discontinuation: Okay To Reach
  Back Out"** instead, using an *unbounded* extraction appropriate to M5's
  blob shape (this field is the last Response_Data segment, with no
  trailing `---`):
  ```
  substring_after("Milestone Instances"."Response Data", 'Okay To Reach Back Out: ')
  ```
  `substring_after(string_column, delimiter, delimiter_count)` was found via
  the formula editor's Functions panel (searched "substring"); its built-in
  tooltip documents `delimiter_count` as optional, defaulting to 1st
  occurrence, matching the exact behavior needed here (return everything
  after the first — and only — occurrence of the label text, to the end of
  the string). Confirmed Text-typed after saving, same as the M0 precedent's
  column type. **Lesson for future milestones**: before naming a new formula
  column, check for an existing column whose name differs only in case —
  the builder's uniqueness check is case-insensitive even though the UI
  will happily show you two differently-cased labels right up until the
  save attempt fails.

- **"M5 Submitted Responses" report**: new Tabular View (not a Save-As — no
  existing report has this exact 4-column shape). Built via the "+" (Create)
  menu → "Tabular View" → base table "Milestone Instances". Columns, in
  order: Submitted Date Time, Token, Reason For Leaving, Discontinuation:
  Okay To Reach Back Out (no Patient/identity column, no scored/flag
  columns — matches M5's minimal-fields spec and every prior milestone's
  privacy pattern). Filtered to `Milestone` Wildcard Exactly Matches `"5B -
  Discontinuation, Email Fallback"`, added via the Filters tab's drag-the-
  column-onto-the-drop-zone technique `m4-implementation-notes.md` §5
  documents (checkbox-click only adds a tabular column, even on the Filters
  tab). Saved to "Zoho CRM Modules (Data)". View ID `3251423000000232237`.
  Verified in View Mode: correct headers, 0 rows (expected, no M5
  submissions exist yet).

- **"M5 Status Breakdown" report**: Save As off "M4 Status Breakdown"
  (`m4-implementation-notes.md` §5, view ID `3251423000000232098`), same bar
  chart config preserved (X-Axis `Status` Actual, Y-Axis `Id` Count). Only
  change: Milestone Wildcard filter value swapped from `"4 - Discharge"` to
  `"5B - Discontinuation, Email Fallback"`. Saved to "Zoho CRM Modules
  (Data)". View ID `3251423000000232243`. Regenerated graph shows "No Data
  Available" (expected).

- **"M5 Reason For Leaving Distribution" report**: Save As off "M2 Alliance
  Check-In Total Distribution" (`m2-implementation-notes.md` §9.5, view ID
  `3251423000000141097`), then X-Axis column swapped from "Alliance
  Check-In Total" to "Reason For Leaving" and the Milestone Wildcard filter
  value swapped to `"5B - Discontinuation, Email Fallback"`. Y-Axis
  unchanged: `Id` (Count). Confirmed the X-Axis dropdown shows plain
  `Actual` (not `Actual(D)`) immediately after the column swap — since
  "Reason For Leaving" is Text-typed, no explicit `Treat as Text` re-set was
  needed, consistent with M4's precedent (`m4-implementation-notes.md` §5).
  Saved to "Zoho CRM Modules (Data)". View ID `3251423000000232265`.
  Regenerated graph shows "No Data Available" (expected).

- **"M5 Okay To Reach Back Out Breakdown" report** (optional second
  breakdown, per `m5-tasks.md`'s Phase 7 task text): Save As off "M5 Reason
  For Leaving Distribution" (this section), X-Axis column swapped to
  "Discontinuation: Okay To Reach Back Out", filter unchanged (already `"5B
  - Discontinuation, Email Fallback"` via the copy). Saved to "Zoho CRM
  Modules (Data)". View ID `3251423000000232293`. Regenerated graph shows
  "No Data Available" (expected).

- **"M5 - Discontinuation Feedback" dashboard**: new dashboard, view ID
  `3251423000000232368`, built via "Create New Dashboards" (not Save As —
  no prior dashboard has a matching panel set). Bundles all four reports
  above, dragged in from the Reports panel (searched by exact name, one at a
  time, to sidestep the alphabetical re-flow mishap
  `m3-implementation-notes.md` §1.8 documents for multi-report dashboard
  builds): M5 Status Breakdown, M5 Reason For Leaving Distribution, M5
  Submitted Responses, M5 Okay To Reach Back Out Breakdown. **No shared
  "Flagged for Review" panel** — not applicable to M5, per `m5-research.md`
  Decision 6 (unlike M4, which widened the shared M2/M3 gate to also cover
  itself). Left at "Auto Add User Filters" on and "Make User Filters Global"
  off, matching every prior milestone's dashboard defaults. All four panels
  verified rendering correctly in View Mode: the two chart panels (Status
  Breakdown, Reason For Leaving Distribution) show "No Data Available" and
  the two tabular/breakdown panels (Submitted Responses, Okay To Reach Back
  Out Breakdown) show their correct column headers / "No Data Available"
  with 0 rows — all expected, since no real M5 submissions exist yet.

## 5. Remaining work (not yet built)

- Phase 8 (T022-T026): live test data — paused at Checkpoint 4 (just
  reached, per §4) pending Costin's review of the Analytics
  reporting/dashboard, and pending his direct involvement (a test Patient
  ID, per his no-live-test-without-me instruction). Includes confirming the
  new formula columns and reports actually parse a real M5 submission
  correctly.
- The open item recorded in `m5-research.md` ("Open item for spec.md") is
  still open: this build covers only the cancellation leg of spec.md's
  Milestones-table row 5, not the no-show leg. Unaffected by today's work.

## 6. Access constraint compliance

All Analytics work this session used the Zoho Analytics builder directly
(not the CRM's Leads/Patients modules), so the standing "don't open CRM
Leads/Patients modules without permission" constraint did not apply to this
phase. No CRM Leads or Patients records were viewed or edited in the
browser during the Analytics build. No live/end-to-end test or real test
data was submitted, per the standing no-live-test-without-Costin
instruction, which this file's header confirms extends explicitly to
Analytics.

## 7. Change (2026-09-25): clinician now derived from Assigned Therapist, not hardcoded

Same bug and same fix as `m2-implementation-notes.md` §12 (read that
section for the full finding, the `Assigned_Therapist` picklist
confirmation, and the clearing/insertion mechanics — not repeated here).
On "M5 - Discontinuation Trigger", the `issueFeedbackToken` node's
`clinician` parameter was changed from the literal text `Liana Preudhomme`
(§2 above shows the pre-fix parameter list) to the chip `Updated module
entry → Assigned Therapist` (`${trigger.Assigned_Therapist}`). All other
`issueFeedbackToken` parameters unchanged (in particular, `milestone`
stays the typed literal `5B - Discontinuation, Email Fallback`, confirmed
unaffected while reopening this panel for the clinician fix). Cleared via
click → `End` → `Backspace` ×25, inserted via the Insert Variable panel's
"Updated module entry" category (expand via the category header, not the
search box, to avoid stealing focus from the target field), saved, then
reopened to confirm the chip persisted.

M0 deliberately left unchanged, same reasoning as M2's §12. Identical
treatment applied the same session to M2, M3, and M4 — see
`m2-implementation-notes.md` §12, `m3-implementation-notes.md` §8, and
`m4-implementation-notes.md` §8. This was the last of the four flows to
be fixed; flow stays **OFF**.
