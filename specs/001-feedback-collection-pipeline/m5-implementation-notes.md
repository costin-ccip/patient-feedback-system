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
> **2026-09-28: the patient-facing email's Subject and Body were redesigned to
> match the real Zoho Bookings confirmation template, with new Tone-of-Voice copy.
> Applied live. See §12 (and `m0-implementation-notes.md` §14 for the shared
> design rationale across all five milestones).**
>
> **2026-09-28 (later same day): survey simplified to a single question,
> write-back function and Analytics updated to match. Form title changed to
> "Your Feedback on Stepping Away From Sessions", the dropdown's first choice
> reworded to first person, and the "Would it be okay to reach back out if
> things change?" Radio field deleted outright (not just hidden). Both flows
> turned OFF, `submitDiscontinuationFeedbackResponse` edited to drop the
> `okayToReachBackOut` parameter and its Response_Data segment (trailing `---`
> on the now-last "Reason For Leaving" segment kept unchanged). The
> "Discontinuation: Okay To Reach Back Out" formula column, the "M5 Okay To
> Reach Back Out Breakdown" report, and its dashboard panel were all deleted
> from Analytics. See §13 for the full change, including a new gotcha:
> `Show/Hide Column` on a Tabular View report is a display-only toggle and
> does **not** clear a column dependency — the column has to be removed from
> the report's actual column definition (Edit Design → the "×" on the column
> chip → regenerate → save) before the underlying formula column can be
> deleted.**
>
> Last updated: 2026-09-27
> Status update (2026-09-27): both flows are still live/ON and Costin is
> running live end-to-end tests himself. §9, §10 and §11 document three fixes
> found during that live testing (a missing Field Alias on the hidden Token
> field, a broken patient-facing survey link caused by a styled `<a>` tag
> instead of the established plain-URL convention, and — most recently — a
> genuine Zoho-side 404 bug on the public form permalink itself, fixed by
> rebuilding the form as a new object, same as M0 §13) — the rest of this
> header (below) predates live testing and is left as-is for history; treat
> §9/§10/§11 as the current state of the patient-facing survey form and Send
> Email step. **§11's "Resolved" subsection**: the "M5 - Discontinuation
> Write-back" flow's trigger was pointing at the now-retired form object and
> couldn't be repointed at first because the shared "Connection to Cape
> Clarity Zoho Forms" connection was showing "The connection does not have
> permission to process this trigger" on every flow that used it (confirmed
> on M4's write-back flow too). Costin reconnected the connection and gave
> explicit go-ahead for this session to complete a second, trigger-specific
> OAuth authorization step that the general reconnect didn't cover; the
> trigger's Form was then switched to the new form, its field mappings
> verified, and the change applied live. M4's trigger also cleared (no
> changes needed there — its Form was already correct). Only real
> end-to-end verification (a live submission actually landing in CRM) is
> still pending — see §11's closing note.
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
| 1 | Dropdown | What led to stepping away from sessions right now? | Felt better / reached their goals [Superseded 2026-09-28 — see §13, reworded to "Felt better / reached my goals"]; Scheduling or timing conflict; Cost or insurance; Didn't feel like the right fit with their therapist; Life circumstances changed; Moved or relocated; Choosing to pause for now; Something else (8 total, source order preserved) | Yes | Show |
| 2 | ~~Radio~~ **[RETIRED 2026-09-28 — see §13, field deleted]** | ~~Would it be okay to reach back out if things change?~~ | ~~Yes / No / Maybe~~ | ~~Yes~~ | ~~Show~~ |
| 3 | Single Line | Token | — | No | **Hide** |

Form title as originally built: "Cape Clarity Discontinuation Feedback".
**Superseded 2026-09-28 — see §13**: retitled to "Your Feedback on Stepping
Away From Sessions".

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

   **Superseded 2026-09-28 — see §13**: the `okayToReachBackOut` parameter
   and its Response_Data concatenation line were removed (now a two-
   parameter function, one Response_Data segment).
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

- **"Discontinuation: Okay To Reach Back Out" formula column**
  **[RETIRED 2026-09-28 — see §13, column deleted]** — **named
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

  **Column set updated 2026-09-28 — see §13**: the "Discontinuation: Okay To
  Reach Back Out" column was removed from this report's own column
  definition (not merely hidden) — now a 3-column report (Submitted Date
  Time, Token, Reason For Leaving).

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

- **"M5 Okay To Reach Back Out Breakdown" report** **[RETIRED 2026-09-28 —
  see §13, report deleted along with its dashboard panel]** (optional second
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

  **Panel count updated 2026-09-28 — see §13**: the "M5 Okay To Reach Back
  Out Breakdown" panel was removed (dashboard now has 3 panels: M5 Status
  Breakdown, M5 Reason For Leaving Distribution, M5 Submitted Responses).

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

## 9. Fix (2026-09-25, pre-emptive): Field Alias missing for hidden Token field

Found while investigating a real M2 write-back failure Costin hit in his
live test — see `m2-implementation-notes.md` §13 for the full diagnosis.
Same root cause applies here: "Cape Clarity Discontinuation Feedback"'s
hidden `Token` field had no Field Alias configured (Settings → Prefill →
Field Alias - Prefill URL was on the empty "Configure Now" screen), so
the `?token=...` URL parameter the trigger flow sends would never have
reached the field.

**Fixed**: Field Label `Token`, Field Alias `token` → Save. Not yet
live-tested end-to-end (M5's flows are still OFF per §2/§3), so this is a
pre-emptive fix — worth confirming during M5's own live test that the
hidden field actually prefills, same as M2 §13's verification step.

## 10. Fix (2026-09-26): patient-facing survey link rendered as literal
Markdown-style text instead of a clickable link

**Symptom**: Costin ran a live M5 test (moved a test patient to
`Discontinued (Patient Choice)`) and received the discontinuation email, but
the survey link was broken/unclickable and the email "read weird" — it showed
the literal text `[Share a quick note →](https://forms.zohopublic.com/...)`
instead of a working hyperlink.

**Root cause**: the patient-facing "Send email" node's Body field (§2's step
"Then → Zoho Mail 'Send email' (patient-facing)") had the call-to-action built
as a real HTML anchor — `<a href="...token=${issueFeedbackToken_1.token}">
<b>Share a quick note →</b></a>` — unlike every other milestone's pattern
(M0-M4), which prints the raw survey URL directly in the body as plain text
and relies on the recipient's mail client to auto-linkify it (see e.g.
`m4-implementation-notes.md` line ~131: "Body's link ends
`?token=${issueFeedbackToken_1.token}`"). Confirmed via Task History → the
9/26 7:41:37 PM run's "Send email" step **input** (the fully-resolved payload
handed to Zoho Mail's API): the Body string Zoho Flow actually sent contained
a well-formed `<a href="https://forms.zohopublic.com/.../formperma/...?
token=<real-resolved-token>"><b>Share a quick note →</b></a>` — so the
`${issueFeedbackToken_1.token}` substitution itself was working correctly;
the bug was specifically that Zoho Mail's "Send email" action does not render
this style of anchor tag as a clickable link the way the rest of the flow's
Send Email nodes' plain-URL bodies do — it (or an intermediate plain-text
conversion) re-renders the anchor as `[link text](href)`.

Likely origin: `m5-research.md` Decision 4's source table represents the
call-to-action as `**[Share a quick note →]** (link to the form)` — its own
authoring shorthand meaning "make this text a link to the form" (the same way
`[First Name]` there is a placeholder marker, not literal text) — and an
earlier build pass took that literally and built a styled anchor instead of
following the established plain-URL convention.

**Fixed**: in the "M5 - Discontinuation Trigger" flow, Builder → the
patient-facing "Send email" node → pencil/edit icon (not the "⋮" menu, which
only opens a generic Clone/Delete context menu) opens the node's full
configuration panel. In the Body field's code view (`<>` toolbar icon),
replaced the `<a href="..."><b>Share a quick note →</b></a>` block with plain
text, matching M0-M4's convention exactly:

```
Share a quick note: https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedback/formperma/6h7a2uGbEbrORj9hjp_eIKb5eV9ocY4ctW2efXxHXmw?token=${issueFeedbackToken_1.token}
```

No other part of the Body changed. Saved via the node panel's "Done" (which
puts the flow into a **Draft** state — the live flow keeps running the old
version until explicitly published), then reviewed the single pending change
("Configuration changed in Send email") in the "Apply Changes?" dialog before
clicking **Apply** — confirmed via the success toast and by re-opening the
node afterward that the live version now matches. Node title is unchanged
("Send email"); no other node, connection, or field was touched.

**Builder gotcha worth flagging for future sessions**: the Body field's
WYSIWYG (rich-text) editor auto-linkifies a bare URL as soon as you type
adjacent text next to `https://` — if a character lands immediately before
the URL with no space (e.g. from an imprecise text replacement), the editor
can silently wrap the URL in its own generated `<a>` tag with a mangled href
(observed once mid-fix: `href="http://..."` — note the dropped `s` — wrapping
a duplicated copy of the URL text). Safer to make this kind of body edit
entirely in the code view (`<>` icon), never partly in WYSIWYG, and to verify
each edit with a small selection check (e.g. `Shift+→` over 1-2 characters)
before typing, rather than assuming a click landed at the intended character
offset.

**Confirmed (2026-09-27)**: Costin re-triggered M5 for a fresh test patient
and the email now renders the survey link as clean plain text (no more
literal `[Share a quick note →](url)`). That confirmed this fix — but revealed
a second, separate bug in the link itself: see §11.

## 11. Fix (2026-09-27): public survey form permalink 404'd — form rebuilt as
a new object, same bug class as M0 §13

**Symptom**: after §10's link-text fix was confirmed (email now shows the
plain-text link correctly), Costin re-tested M5 again and reported that
clicking the survey link now returns Zoho's own "Sorry! Page not found."
error page (screenshot attached to his report). The link he received:

```
https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedback/formperma/6h7a2uGbEbrORj9hjp_eIKb5eV9ocY4ctW2efXxHXmw?token=<real-resolved-token>
```

**Diagnosis**, following the exact playbook `m0-implementation-notes.md` §13
used for the same bug class:

- Reproduced the 404 independently by navigating directly to the formperma
  URL, with and without the `?token=` param — confirmed via
  `read_network_requests` that the document-level GET itself returns HTTP
  404 (a genuine server-side 404, not a client-rendering issue).
- Ruled out a copy/paste mismatch: compared the URL embedded in the flow's
  Send Email body against the form's own Share tab → Form Permalink (URL)
  field, byte-for-byte (zoomed screenshot comparison) — they matched
  exactly.
- Ruled out the share setting being off: Share Publicly showed **Enabled**.
- Tried the standard remedy anyway (toggle Share Publicly off, confirm the
  "this URL will be invalid henceforth" warning, then back on) — did **not**
  fix it; the 404 reproduced again immediately afterward, same as M0's
  finding.
- Ruled out an account-wide problem: a sibling form on the identical
  `forms.zohopublic.com/.../formperma/...` URL pattern and account (M4's
  "Cape Clarity Discharge Feedback") resolved fine (HTTP 200, form rendered
  correctly).
- Confirmed the form object itself is intact: its authenticated internal
  Builder preview rendered both fields (the "What led to stepping away..."
  dropdown and "Would it be okay to reach back out..." radio group)
  correctly.

Conclusion: the same genuine, reproducible Zoho-side bug M0 §13 already hit
and documented, specific to that one form object's public formperma link,
not a configuration error on our side.

**Fix**: rebuilt the form fresh, following M0 §13's remediation exactly.
Used Zoho Forms' own Duplicate action (My Forms → hover the form's row → "⋮"
→ Duplicate), named the duplicate "Cape Clarity Discontinuation Feedback
NEW" at creation time (this also becomes the permanent URL slug —
`CapeClarityDiscontinuationFeedbackNEW` — cosmetic-only and unaffected by
retitling afterward, same as M0's `...NEW` slug). Verified before touching
anything else:

- The hidden Token field's Field Alias (Settings → Prefill → Field Alias -
  Prefill URL) carried over from the original as `Token` → `token` —
  Zoho's Duplicate preserves this, so §9's fix did not need to be repeated.
- Share Publicly was Enabled by default on the duplicate.
- The new public formperma URL resolves — confirmed via
  `read_network_requests` showing HTTP 200 on the document-level GET, both
  with and without a `?token=` test value appended:

  ```
  https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedbackNEW/formperma/Sw2QlbJj9kAM5h6Mh5sf_7-M7d7cYFSTdLGv96Dn4Vk
  ```

**Renaming**: old form's Form Properties → Form title changed to `[RETIRED]
Cape Clarity Discontinuation Feedback` (Zoho Forms has no direct dashboard
"Rename" action, same as M0). New form's Form Properties → Form title set to
the canonical `Cape Clarity Discontinuation Feedback` (its URL slug stays
`CapeClarityDiscontinuationFeedbackNEW`). The retired form's public sharing
was left as-is (already effectively dead since it 404s), not explicitly
disabled — matching M0's choice.

**Flow repointed**: the "M5 - Discontinuation Trigger" flow's patient-facing
"Send email" node Body was updated (same Draft → "Apply Changes?" review →
Apply pattern as §10, confirmed the dialog listed only "Configuration changed
in Send email") to the new permalink, keeping the same
`?token=${issueFeedbackToken_1.token}` suffix:

```
Share a quick note: https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedbackNEW/formperma/Sw2QlbJj9kAM5h6Mh5sf_7-M7d7cYFSTdLGv96Dn4Vk?token=${issueFeedbackToken_1.token}
```

Edited entirely in the Body field's code view, using the same click →
`Shift+→` → zoom-verify boundary technique as §10 (verified both the start
boundary right before `https` and the end boundary right before `?token=`
before typing the replacement), so no hand-retyping of the rest of the body
was needed and no WYSIWYG auto-linkification risk was introduced. Live flow
confirmed matching afterward.

**Resolved (2026-09-27, later same day)**: Costin reconnected "Connection to
Cape Clarity Zoho Forms" in Zoho Flow (Settings → Connections — the general
connection-level reconnect). That alone did **not** clear the error on
either write-back flow's trigger; reopening "M5 - Discontinuation
Write-back"'s trigger config still showed the same "The connection does not
have permission to process this trigger." with its own separate
"Reconnect" button. Confirmed with Costin before proceeding (per this
session's standing rule on OAuth/SSO grants), then clicked that trigger's
own "Reconnect" → "Authorize" (scoped to "Form entry submitted" only, per
the "Only specific triggers and actions" option already selected) — this
is a second, trigger-specific authorization step distinct from the
connection-level reconnect Costin had already done, and it's what actually
cleared the error (toast: "Connection reconnected").

With the trigger config now loading cleanly, changed its **Form** field from
`[RETIRED] Cape Clarity Discontinuation Feedback` to the new **Cape Clarity
Discontinuation Feedback** (the rebuilt object, §11 above). Opened the
downstream `submitDiscontinuationFeedbackResponse` function step and
confirmed all three parameter mappings resolved cleanly against the new
form's fields with no broken references: `token` → Form entry submitted →
Token, `reasonForLeaving` → Form entry submitted → "What led to stepping
away from sessions right now?", `okayToReachBackOut` → Form entry submitted
→ "Would it be okay to reach back out if things change?" (as expected,
since Duplicate preserves internal field API names — same pattern already
confirmed in M0 §13). Saved the trigger config ("Done"), which put the flow
into Draft, then **Apply changes** → confirmed the review dialog showed
exactly one change ("Configuration changed in Form entry submitted") →
**Apply**. Flow returned to **• Live**. Re-opened the trigger config
afterward to confirm the Form field change had actually persisted (it had).

**Cross-checked M4 while in there**: opened "M4 - Discharge Write-back"'s
trigger config (read-only check, no edits) — it now also loads cleanly with
no connection error and its Form field is already correctly set to "Cape
Clarity Discharge Feedback" (M4 was never affected by the 404/retirement
issue, only by the shared connection's permission error). So the
trigger-level Authorize step fixed the connection for both flows at once;
M4 needed no further changes and remains **Paused/OFF**, unchanged from
before this session.

**Verification still pending**: an actual end-to-end submission through the
new form (token prefill + write-back landing in CRM) has not been tested
yet. Costin should re-trigger M5 for a fresh test patient and confirm both
that the link loads (not 404s) and that, after submitting, the response
appears against the right `Milestone_Instances` record in CRM.

## 12. Change (2026-09-28): patient-facing email redesigned — real Zoho Bookings template, new ToV copy

Same redesign as M0/M2/M3/M4 — see `m0-implementation-notes.md` §14 for the full
shared rationale (real Zoho Bookings template as the visual basis, single
full-URL link, verification method used before saving). This section covers only
M5's own before/after copy.

**Old Subject** (live before this change): "Just checking in"
**New Subject**: "Checking in, and wishing you well"

**Old Body** (plain-text pattern, live before this change):

```
Hi there,
We noticed it's been a little while since your last visit, and we wanted to reach out and see how you're doing.
If you have a minute, we'd love to hear a bit about where things stand for you right now. This is completely optional, takes less than a minute, and won't affect anything if you'd ever like to come back.
Share a quick note:
https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedbackNEW/formperma/Sw2QlbJj9kAM5h6Mh5sf_7-M7d7cYFSTdLGv96Dn4Vk?token=${issueFeedbackToken_1.token}
Whatever's next for you, we're glad you gave us the chance to work together, and the door's always open.
Warmly,
The Cape Clarity Team
```

**New Body** (verbatim, now live in "M5 - Discontinuation Trigger"'s
patient-facing "Send email" step):

```html
<div class="cc-email-notification-container" style="background: #FFF; padding: 0; margin: 0; color: rgb(43, 43, 43); width: 100%; height: 100%; display: table; font-family: &quot;Open Sans&quot;, &quot;Trebuchet MS&quot;, sans-serif"><table align="center" width="600" style="text-align: center"><tbody><tr><td><table cellpadding="0" cellspacing="0" align="center" style="text-align: left; background: #FFF; margin-top: 50px; border-radius: 10px 10px 5px 5px; border: 1px solid rgb(224, 224, 224)"><tbody><tr><td><img alt="Cape Clarity Integrative Psychology" style="width: 100%; max-width: 600px; display: block; border: 0" width="600" src="https://cdn.prod.website-files.com/67dcadb2b4365cb9f4268b42/6a6a9e1731b99ce6392f9ecc_cape-clarity-email-header-band.png"></td></tr><tr><td><table class="inner-content" style="padding: 45px 55px"><tbody><tr><td><section><p style="margin:0px; line-height: 30px;"><span style="color:rgb(43, 43, 43)"><b><span style="font-size: 18px; margin: 0px; line-height: 30px;">Hi there,<br></span></b></span></p><div style="font-size: 15px; line-height: 28px; padding: 16px 0 0 0; color: rgb(43, 43, 43);">As you step away from care for now, we wanted to check in. People pause for all kinds of reasons, and we'd love to understand yours. Two quick questions, if you're willing:<br></div><div style="font-size: 15px; line-height: 24px; padding: 10px 0 4px 0; word-break: break-all;"><a rel="noopener noreferrer" target="_blank" href="https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedbackNEW/formperma/Sw2QlbJj9kAM5h6Mh5sf_7-M7d7cYFSTdLGv96Dn4Vk?token=${issueFeedbackToken_1.token}" style="color: #175328; text-decoration: underline; font-weight: 600;">https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedbackNEW/formperma/Sw2QlbJj9kAM5h6Mh5sf_7-M7d7cYFSTdLGv96Dn4Vk?token=${issueFeedbackToken_1.token}</a></div><div style="font-size: 15px; line-height: 28px; padding: 16px 0 0 0; color: rgb(43, 43, 43);">We wish you well, and we're here whenever you're ready, if you are.<br></div><hr style="opacity: 0.3; margin: 35px 0px 35px 0px"><p style="margin: 10px 0px"><span style="color:rgb(43, 43, 43)"><span style="font-size: 15px; margin: 10px 0px;">Warmly,</span></span><br></p><p style="margin: 10px 0px"><span style="font-size: 15px; margin: 10px 0px;"><b style="color: rgb(43, 43, 43); font-weight: 600;">Cape Clarity Integrative Psychology</b></span><br></p></section></td></tr></tbody></table></td></tr></tbody></table></td></tr><tr><td><table align="center" style="margin-top: 30px; text-align: center"><tbody><tr><td valign="middle" style="display: block; text-align: center; padding: 0px 0 10px; font-size: 13px; line-height: 22px"><span style="color: rgb(119, 119, 119); font-weight: normal;">This email may contain information intended only for the person named above. If you received it in error, please let us know and delete it.</span><br></td></tr><tr><td valign="middle" style="display: block; text-align: center; padding: 0px 0 40px; font-size: 13px; line-height: 22px"><span style="color: rgb(119, 119, 119); font-weight: normal;">Cape Clarity Integrative Psychology</span><br></td></tr></tbody></table></td></tr></tbody></table></div>
```

**Note on the copy change**: dropped "We noticed you've stepped away" (per
Costin's edit — this milestone's survey can also fire when a patient tells the
practice directly they'd like to end treatment, not only when they go quiet, so
the old "we noticed" framing didn't fit both cases). No em dashes in the new copy
(house style). The closing line ("We wish you well, and we're here whenever
you're ready, if you are") intentionally echoes the survey's own thank-you page
wording, which Costin already reviewed separately.

Applied to the live flow via Zoho Flow's "Apply changes" on a Draft (M5 was
already Live/ON going into this change, same as M0); the "Apply Changes?"
confirmation dialog showed exactly one item ("Configuration changed in Send
email") before Apply was clicked, confirming nothing else in the flow was
touched.

## 13. Change (2026-09-28): survey simplified to a single question, write-back function and Analytics updated to match

Three linked changes, all Costin's explicit direction, applied live in this
order: (1) the form — title reworded, first dropdown choice reworded to
first person, the "reach back out" Radio field deleted outright; (2) the
write-back function — parameter and Response_Data segment for the deleted
field removed, both flows left OFF throughout (they were already OFF at the
end of §11/§12's live testing); (3) Analytics — the now-orphaned formula
column, its dedicated breakdown report, and that report's dashboard panel
all deleted, plus (an extra step this change surfaced, not originally
anticipated) the column's reference in the separate "M5 Submitted Responses"
report.

### 13.1 Form: title, dropdown wording, field removal

All three changes made in the Zoho Forms builder on the live (rebuilt, §11)
form object — `CapeClarityDiscontinuationFeedbackNEW`
(`https://forms.zoho.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedbackNEW/builder`).

- **Form Properties title**: `Cape Clarity Discontinuation Feedback` →
  **`Your Feedback on Stepping Away From Sessions`** — matches the naming
  pattern Costin set for M0/M4's retitle ("Your Feedback on ...", user-
  centered rather than internal-process-named). Saved via Form Properties →
  Save, then re-opened to confirm the canvas title persisted, per the
  save-toast-doesn't-guarantee-persisted-render gotcha `m4-implementation-notes.md`
  §12 documents.
- **Dropdown field, first choice**: opened the "What led to stepping away
  from sessions right now?" field's Properties panel → Choices → Edit
  sub-panel. First choice text `Felt better / reached their goals` → **`Felt
  better / reached my goals`** (third-person → first-person, matching the
  survey's own first-person voice elsewhere). Edited via click → `End` →
  `Shift+Home` → `Delete` → type replacement (the established technique for
  this sub-panel, since `Ctrl+A` does not select-all inside an individual
  choice text input — see the file's other milestones' notes on this same
  Choice Field Properties quirk), verified via zoomed screenshot before
  saving. All 7 other choices unchanged.
- **Field deletion**: the "Would it be okay to reach back out if things
  change?" Radio field was deleted outright (hover the field → trash icon →
  confirm the deletion dialog), not merely hidden — the survey is now a
  single substantive question (the dropdown) plus the hidden Token field.
  Verified via screenshot: the field no longer appears in the builder canvas
  or in the field list.

The form's public formperma URL, Field Alias (`Token` → `token`), and Thank
You Page text (§1.1) are all unaffected by these edits — no other field or
setting was touched.

### 13.2 Flows: confirmed OFF, then `submitDiscontinuationFeedbackResponse` edited

Both "M5 - Discontinuation Trigger" and "M5 - Discontinuation Write-back"
were already switched OFF at the end of Costin's 2026-09-27 live testing (per
this file's header); confirmed still OFF before editing, and left OFF
throughout this change — no live/end-to-end test was run.

Edited `submitDiscontinuationFeedbackResponse` (open the Write-back flow →
Builder → the function node's "⋮" menu → **View function** — not the pencil
icon, which opens parameter mapping, and not the node title, which does
nothing useful here):

- **Signature**: `map submitDiscontinuationFeedbackResponse(string token,
  string reasonForLeaving, string okayToReachBackOut)` → **`map
  submitDiscontinuationFeedbackResponse(string token, string
  reasonForLeaving)`** — the `okayToReachBackOut` parameter removed.
- **Body**: the line `responseText = responseText + "Okay To Reach Back Out:
  " + ifnull(okayToReachBackOut,"");` was deleted. The preceding line's
  trailing `"---"` delimiter — `responseText = "Reason For Leaving: " +
  ifnull(reasonForLeaving,"") + "---";` — was **deliberately left unchanged**
  even though "Reason For Leaving" is now the last (only) Response_Data
  segment. This follows the same "keep trailing delimiter on a new-last-
  field" pattern `m0-implementation-notes.md` §15 documents: the existing
  "Reason For Leaving" formula column (§4) is a *bounded*
  `substring_between(..., 'Reason For Leaving: ', '---', 1)` extraction, and
  changing the blob's terminator would have required editing that
  already-working formula too. No Analytics formula column edits were
  needed as a result.

Text was selected and deleted using the established mouse-based technique
(double-click to select a word, Shift+click/Shift+Arrow to extend the
selection across the rest of the line, Delete) since CodeMirror doesn't
support the SDK's direct text-manipulation tooling in this session. Saved
through the now-familiar "spurious dependency" false-positive confirmation
dialog (Confirm). Reopened the function afterward (View function again) to
confirm both the new signature and body persisted correctly.

**Parameter mapping auto-cleaned, no orphan reference**: reopened the
function node's parameter-mapping panel (pencil icon) after saving — it now
shows only `token` and `reasonForLeaving`, each still correctly mapped to
their form fields (Token, "What led to stepping away from sessions right
now?"); the removed `okayToReachBackOut` mapping was cleared automatically,
with no leftover/broken chip to clean up manually.

### 13.3 Analytics: formula column, report, and dashboard panel deleted

Reached via `https://analytics.zoho.com/workspace/3251423000000083002`
(already signed in). Order followed the established safe-deletion sequence
(clear dependents first, delete the column last), since attempting the
column delete first produces a blocking error naming every report that still
references it.

1. **Dashboard panel removed first**: opened the "M5 - Discontinuation
   Feedback" dashboard (Dashboards → search) → Edit Design → selected the
   "M5 Okay To Reach Back Out Breakdown" panel → its toolbar's trash icon →
   confirmed "Are you sure you want to remove the selected views from this
   dashboard?" → **Save**. This only unbundles the panel from the dashboard;
   it does not delete the underlying report (that's step 3). Dashboard now
   shows 3 panels (§4's panel list updated accordingly).
2. **"M5 Okay To Reach Back Out Breakdown" report deleted**: Explorer →
   searched "Okay To Reach Back Out" → the report's "⋮" menu → Delete →
   confirmed "Are you sure you want to delete 'M5 Okay To Reach Back Out
   Breakdown'?" — checked its **Dependency Details** panel first (Parent
   Tables: Milestone Instances; Dashboards: M5 - Discontinuation Feedback,
   1 reference) to confirm nothing else depended on it before deleting.
3. **Attempted the formula column delete — blocked, revealing a new
   gotcha**: Data → Milestone Instances table → right-click the
   "Discontinuation: Okay To Reach Back Out" column header → **Delete
   Column** → blocked with "Column 'Discontinuation: Okay To Reach Back Out'
   cannot be deleted due to the following reason: Used in the Report 'M5
   Submitted Responses'". This report (§4) directly embeds the column as one
   of its 4 tabular columns — not the report just deleted in step 2.
4. **First attempt at clearing it — did not work, worth flagging**: tried
   the "M5 Submitted Responses" report's **More → Show/Hide Column** dialog
   → unchecked "Discontinuation: Okay To Reach Back Out" → OK → the column
   visually disappeared from the report and an explicit **Save** click
   returned "Already saved. No modification found." (implying success).
   Re-opening the report later in the same session showed the column back
   again, still checked in Show/Hide Column, and the column delete attempt
   still failed with the identical dependency error. **Lesson for future
   sessions**: `Show/Hide Column` on a Tabular View report is a
   session/display-level toggle only — it does not modify the report's
   actual column definition and does not clear a formula-column dependency,
   even though its own Save action reports success.
5. **What actually worked**: opened the report → **Edit Design** (top
   right) → its "Tabular" design panel shows the real column list as
   removable chips (`Submitted ...`, `Token`, `Reason For ...`,
   `Discontinua...`, each with an "×") → clicked the "×" on the
   "Discontinua..." chip → this surfaces a **"Click Here to Generate
   Tabular"** prompt (the design change doesn't take effect until the view
   is regenerated) → clicked it → tabular preview updated to 3 columns → top
   toolbar **Save** → confirmed via success toast ("Saved 'M5 Submitted
   Responses' successfully"). This is the actual removal; §4's "M5 Submitted
   Responses" entry is updated to reflect the new 3-column shape.
6. **Formula column delete retried — succeeded**: back on the Milestone
   Instances table, right-click the "Discontinuation: Okay To Reach Back
   Out" column header → **Delete Column** → confirmation dialog (no
   remaining-dependency error this time) → **Yes**. Verified by scrolling to
   the end of the table's columns: "Reason For Leaving" is now the last
   column; "Discontinuation: Okay To Reach Back Out" no longer appears
   anywhere in the table.

M0's Analytics build (its own similarly-named "Okay to Reach Back Out"
column, lowercase "to" — §4's naming-collision note) was not touched by any
of this — out of scope, unaffected.

### 13.4 Verification

All changes verified structurally (screenshots/zoomed inspection at each
step, plus the Analytics dependency-checker's own errors/successes as a
built-in verification signal), consistent with this session's no-live-test-
without-Costin standing instruction — no form was actually submitted, and
neither flow was turned back ON.

## 14. Correction (2026-09-29, during feature 005 scoping): `Patients1.Name`
("Patient Name") is a de-identified code, not the patient's real name

§2.1 and `m5-data-model.md`/`m5-research.md` reasoned about `[First Name]`
email personalization on the assumption that `Patients1.Name` ("Patient
Name") held the patient's full real name, the same as the separate custom
field `Full_Name_PHI`. While scoping feature 005 (Contractor Performance
Analytics), Costin clarified this was wrong: `Name` is, by explicit practice
design, never populated with a real name — it holds a deliberately
de-identified pseudonym code (`P` + patient number). `Full_Name_PHI` is a
distinct field that does hold the real name, and M5 never reads it.
Confirmed live the same session via `Zoho_CRM getFields` (schema-only, no
records) on `Patients1`: `Name` (not custom, `json_type: string`, length
120, label "Patient Name") vs. `Full_Name_PHI` (custom, length 255, label
"Full Name (PHI)") — two distinct fields.

This does not change M5's actual built behavior: a `P`+number code wouldn't
have worked as a friendly `[First Name]` greeting any more than a full name
would have, so the fallback documented in §2.1 (drop `[First Name]`, open
with "Hi there,") remains correct either way. The only thing that changes is
the *reasoning* recorded in the docs — corrected in `m5-data-model.md`
(field inventory table and the `[First Name]` personalization note) and
`m5-research.md` (Decision 4) in the same session as this note, per
CLAUDE.md's documentation-accuracy convention. Full context: feature 005's
`research.md` Decision 1, which needed this clarification to resolve a
different question (a safe per-patient grouping key for Analytics) and
surfaced this correction as a side effect.
