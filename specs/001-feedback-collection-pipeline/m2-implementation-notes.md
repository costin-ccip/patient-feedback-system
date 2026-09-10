# M2 (Session 3 / Early Alliance Check) — Implementation Notes

> Scope: this file documents ONLY the M2 milestone (the automatic Session 3 /
> Early Alliance Check feedback loop). It is a companion to
> `specs/001-feedback-collection-pipeline/spec.md` (all six milestones) and to
> `m2-plan.md` / `m2-research.md` / `m2-data-model.md` / `m2-tasks.md`. This file
> tracks what has actually been built in Zoho, as it's built, per the
> `CLAUDE.md` convention of updating implementation notes in the same session
> as the change.
>
> Last updated: 2026-09-10
> Status: Phase 1 (T001, the Cape Clarity Alliance Check-In form) is built and
> saved. Nothing else in M2 has been built yet — no Zoho Flow, no custom
> functions, no CRM records, no Analytics formula columns/reports. This
> session paused before starting T002 (the "M2 - Session 3 Trigger" flow) to
> stay under a usage limit; resume there next session.

## 1. What's built so far (T001 only)

**Cape Clarity Alliance Check-In** — Zoho Form, Standard type, built from
scratch via "New Form" → "Blank Form" (not via Zoho Forms' "Duplicate" action,
which turned out to be non-functional via browser automation in this
account — see §3). Permalink (owner/builder-session URL, confirmed live and
rendering the full public respondent view):
`https://forms.zoho.com/lianapreudhommecapec1/form/CapeClarityAllianceCheckIn`
— this is the value to use for the `survey_url` parameter when T002/T004
build the "Call a subflow" step, mirroring `m1-implementation-notes.md` §4's
`survey_url` pattern. (Not yet cross-checked against the Share tab's formal
"permalink"/`formperma` URL the way M1's notes record one — worth a quick
Share-tab check before wiring the subflow call, in case Zoho Forms exposes a
distinct public-facing permalink the way it did for the Wellbeing Check-In
form.)

### 1.1 Fields, in canvas order

| Order | Field type | Internal name (discovered, not yet DOM-confirmed the way M1's were) | Label | Notes |
|---|---|---|---|---|
| 0 | Description | n/a (not a data-bearing field) | "Intro" (admin-only label) | Free-tier workaround for the Welcome Page paywall — see §2. Renders as plain text above the first slider. |
| 1 | Slider | `Slider` (by M1's type+order naming convention — not yet DOM-verified for this form the way M1's were) | "Overall, how comfortable have you felt being open and honest with your therapist so far?" | Domain: Connection. Instructions: "Domain: Connection. 0 = Not comfortable, 10 = Very comfortable." Range 0-10, Mandatory. |
| 2 | Slider | `Slider1` (expected, not yet DOM-verified) | "Overall, how well do you feel your therapist has understood what matters to you so far?" | Domain: Understanding. Instructions: "Domain: Understanding. 0 = Not understood, 10 = Fully understood." Range 0-10, Mandatory. |
| 3 | Slider | `Slider2` (expected, not yet DOM-verified) | "Overall, how much do you feel you and your therapist agree on what you're working toward?" | Domain: Shared direction. Instructions: "Domain: Shared direction. 0 = Not aligned, 10 = Fully aligned." Range 0-10, Mandatory. |
| 4 | Slider | `Slider3` (expected, not yet DOM-verified) | "Overall, how well has your therapist's approach been working for you so far?" | Domain: Fit of approach. Instructions: "Domain: Fit of approach. 0 = Hasn't worked, 10 = Worked well." Range 0-10, Mandatory. |
| 5 | Single Line | `SingleLine` (expected, not yet DOM-verified) | "Token" | Visibility set to **Hide**, matching the Wellbeing Check-In form's Token field pattern (`m1-implementation-notes.md` §4A's field table). Confirmed hidden in the live respondent-view render (no Token input visible on the public form; only in the builder canvas). |

**Important carry-over task before building T006's write-back function**:
M1's notes (§4A) explicitly warn that Zoho Forms names fields by type +
creation order, not by label, and that the only reliable way to get the exact
internal names is DOM inspection of the form builder
(`id="<InternalName>-li"` on each field's wrapper `<div>`), not assumption.
The `Slider`/`Slider1`/`Slider2`/`Slider3`/`SingleLine` names above are
inferred from M1's naming pattern and this form's build order, but **were not
independently DOM-verified for this form** the way M1's were — do that
verification before writing `submitAllianceCheckInResponse`'s
`${trigger.<InternalName>}` parameter mappings (T006), rather than trusting
this table blindly.

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

## 2. Zoho Forms plan-tier findings (new this session, useful for M3-M5)

The account is on Zoho Forms' **Free** subscription plan (confirmed via
Settings → Subscription plan showing "Free" with an "Upgrade" link). Two
features hit paywalls this session (Welcome Page customization, Thank You
Page Rich Text — see §1.2); everything else needed for M0-M2's forms so far
(Slider/Single Line fields, field Visibility, the free-tier Description field,
Thank You Page Plain Text) has been available on Free. Worth keeping in mind
for M3-M5 form design: don't plan on Welcome Page or Rich Text Thank You
pages without first confirming a plan upgrade, or budget for the same
Description-field / shortened-plain-text workarounds used here.

## 3. Zoho Forms automation gotchas (new this session)

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
  jsPlumb canvas (`m1-implementation-notes.md` §5/§12 documents that flow
  connections routinely fail to wire despite looking connected). Zoho Forms'
  field canvas uses jQuery UI draggable, and the `computer` tool's
  `left_click_drag` action placed every field correctly on the first attempt
  in this session, including dropping the Description field into a specific
  position (above the first Slider) rather than just at the canvas end. No
  special DOM-event-dispatch workaround was needed for form-field placement
  specifically — **this distinction does not carry over to the Flow build**;
  budget for the M1-documented connector-verification discipline once T002
  (Zoho Flow) starts.
- **Verification pattern used throughout this build**: rather than trusting
  screenshots or assuming a click landed where intended, every field-label/
  instructions/range/mandatory-checkbox edit in this session was confirmed by
  reading `document.activeElement.value` (or, for checkboxes, the specific
  `input[elname="field-ismandatory"]` element's `.checked`/`.offsetParent`
  state) via `javascript_tool` immediately after each `triple_click`/`type`
  action, before moving to the next field. This caught the Description-field
  iframe issue (§1.2) immediately rather than after a "Done"/"Save" click,
  and is worth continuing for the Flow build's parameter-mapping steps.

## 4. Remaining work (not yet built — full T002-T026 list)

Nothing beyond T001 has been started. In particular, no Zoho Flow, no custom
functions (`checkAllianceCheckExists`, `submitAllianceCheckInResponse`), no
new `Milestone_Instances` records, and no Analytics formula
columns/reports/dashboard exist yet for M2. Per `m2-tasks.md` and
`m2-data-model.md` (Revision 2 — flag computed entirely in Analytics, per
constitution Principle VII), the next steps in order are:

- **T002**: Build "M2 - Session 3 Trigger" Zoho Flow — mirror
  "M1 - Session 1 Trigger" exactly (`m1-implementation-notes.md` §3):
  trigger on `Patients1.Session_Count` transitioning to `3` → `checkAllianceCheckExists`
  custom function → If-else on its boolean output → true branch calls
  "Subflow - Issue Feedback Token". **Must** verify every link is genuinely
  wired via the DOM connector-element check (`m1-implementation-notes.md`
  §5/§12) before attempting to switch the flow on — M1 hit a real bug here
  twice (write-back flow, then the trigger flow itself) from
  visually-placed-but-unconnected nodes.
- **T003**: `checkAllianceCheckExists` — mirror `checkBaselineIntakeExists`
  (`m1-implementation-notes.md` §3.1) field-for-field, checking for an
  existing `Milestone_Instances` record at `Milestone = "2 - Early Alliance
  Check"` (exact picklist value confirmed in `m2-data-model.md`).
- **T004**: Wire the false/no-existing-record branch to
  "Subflow - Issue Feedback Token" with M2-specific parameters (mirror
  `m1-implementation-notes.md` §4's table): `milestone = "2 - Early Alliance
  Check"`, `survey_url` = this form's permalink (§1 above — confirm the
  formal Share-tab permalink first), plus `email_subject`/`email_intro_text`
  drafted but not yet reviewed by Costin/Liana (same caveat M1's notes
  carry for its own email copy).
- **T005**: Build "M2 - Alliance Check-In Write-back" Zoho Flow — realtime
  Form-submission trigger on this form → `submitAllianceCheckInResponse`.
  Same connector-verification discipline as T002.
- **T006**: `submitAllianceCheckInResponse` — mirror
  `submitWellbeingCheckInResponse` (`m1-implementation-notes.md` §4A.1)
  field-for-field, adapted to the 4 Alliance Check-In domains, writing the
  blob format `m2-data-model.md` specifies: `"Connection (0-10): {value}---
  Understanding (0-10): {value}---Shared direction (0-10): {value}---Fit of
  approach (0-10): {value}"`. **No flag-related logic of any kind** — pure
  string concatenation only, per constitution Principle VII and this
  milestone's Revision 2 correction.
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
- **T016 (data model doc's own numbering) / "M2 Flagged for Review"**:
  Tabular View filtered to `Milestone = "2 - Early Alliance Check"` AND
  `Clinical Safety Flag = "true"`.
- **T020**: **Blocked pending human input** — needs a test Patient record ID
  from Costin; the standing access restriction forbids looking one up via
  CRM query or browser (`CLAUDE.md` "Access constraints").
- **T021-T023**: Live test-data exercise and cleanup, depend on T020.
- **T025-T026**: Update `CLAUDE.md` if the build reveals new cross-milestone
  conventions worth capturing (e.g. the Zoho Forms iframe-editor gotcha in
  §1.2, or the plan-tier findings in §2, may be worth promoting there once
  M3-M5 confirm they recur); commit this file and any further M2 corrections
  in the same session as the change, same git workflow as §5 below.
