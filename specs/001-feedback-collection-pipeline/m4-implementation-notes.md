# M4 (Discharge) — Implementation Notes

> Scope: this file documents ONLY the M4 milestone (Discharge — one-time-per-
> patient, triggered by "Discharge is marked in the system of record"). It is
> a companion to `specs/001-feedback-collection-pipeline/spec.md` (all six
> milestones) and to `m4-plan.md` / `m4-research.md` / `m4-data-model.md` /
> `m4-tasks.md`. This file tracks what has actually been built in Zoho, as
> it's built, per the `CLAUDE.md` convention of updating implementation notes
> in the same session as the change.
>
> Last updated: 2026-09-23
> Status: **T009-T013 (the "Cape Clarity Discharge Feedback" Zoho Form, all 7
> fields) are built and verified by top-to-bottom scroll.** Trigger flow,
> write-back flow, and Analytics/reporting build are not yet started — see §4
> for the full remaining-work list, which otherwise mirrors `m4-tasks.md`.

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

## 2. Not yet built

Per `m4-tasks.md`'s task list:
- **T001-T002**: `checkDischargeExists` custom function (Foundational phase).
- **T003-T008 / "M4 - Discharge Trigger" flow**: trigger, `checkDischargeExists`,
  `isCreatedAfterCutoff` (reused), If-else, `issueFeedbackToken` (reused),
  Send email, both On Error branches. Left OFF once built, per every prior
  milestone's build-first-verify-later convention.
- **T014-T017 / "M4 - Discharge Write-back" flow**: `submitDischargeFeedbackResponse`
  custom function, trigger, parameter mapping to the 7 form fields above.
  Left OFF once built.
- **Analytics**: 2 new formula columns (Looking Ahead: Likelihood To
  Recommend, Looking Ahead: Anything Else), Clinical Safety Flag /
  Flag Rule Triggered `Milestone` gate widened a second time (M2, M3, M4),
  "M2 & M3 Flagged for Review" renamed and widened to "M2, M3 & M4 Flagged
  for Review" — requires `m2-implementation-notes.md` and
  `m3-implementation-notes.md` updates in the same session per
  `m4-data-model.md`'s Decision 6 note.
- **Reporting**: M4 Submitted Responses, M4 Status Breakdown, M4 Looking
  Ahead: Likelihood To Recommend Distribution, M4 dashboard.
- **T023 / test data**: blocked pending a test Patient ID from Costin, per
  standing instruction — not pursued proactively.
- Confirmation from Costin/Liana on whether `Patient_Status = "Completed
  Treatment"` is the correct discharge-trigger filter value (flagged
  non-blocking in `m4-research.md` Decision 1).

## 3. Access constraint compliance

All form-building work this session used the Zoho Forms builder directly
(not a CRM module), so the standing "don't open CRM Leads/Patients modules
without permission" constraint did not apply to this phase. No CRM records
were viewed or edited in the browser during the form build.
