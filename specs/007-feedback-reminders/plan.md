# Implementation Plan: Feedback Reminders

**Branch**: `007-feedback-reminders` | **Date**: 2026-10-09 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/007-feedback-reminders/spec.md`

## Summary

Add one automatic reminder per feedback request, sent at day 4 of the 7-day token life, for
all five active milestones. One new shared scheduled Zoho Flow runs several times a day in
a daytime window. Each run calls one new custom function that finds the oldest eligible
unreminded request, applies every suppression rule, claims the row by stamping
`Reminder_Sent_Date_Time`, and hands the token, address and milestone back to the flow,
which sends the milestone-appropriate reminder through Zoho Mail. The token, the row and
the expiry are untouched. Reporting is computed in Zoho Analytics.

## Technical Context

**Platform**: Zoho CRM (org `org889880832`), Zoho Flow workspace `872426000000002011`
(folder "Customer Feedback System", Standard plan, no loops or subflows), Zoho Mail
(connection "Connection to info@capeclarity.com"), Zoho Analytics workspace
`3251423000000083002` (syncs daily at 5:00 PM EDT).

**Storage**: One new custom field on `Milestone_Instances`:
`Reminder_Sent_Date_Time` (datetime), created 2026-10-09, field id `6825601000004791001`.
Nothing else is stored. No new module, no tags, no new Status value.

**Testing**: Structural checks with Execute and flow Test (quickstart.md A, B), then a live
run folded into the coordinated test (quickstart.md C) against designated test patient
PT000 (`Patients1` `6825601000004448028`) and the existing test lead. Live runs need
Costin's go-ahead (`CLAUDE.md`).

**Constraints**: No flow switched ON by this work; no live or end-to-end tests without
Costin; no reads of Patients or Leads records beyond PT000 and the test lead during
build and test (the runtime function reads them as the pipeline already does, exactly as
`skipIfOptedOut` does); contractors never see reminder state; `issueFeedbackToken`, the
write-back functions and the five trigger flows are not edited.

**Scale/Scope**: 1 new CRM field (done), 1 new custom function, 1 new flow, 5 reminder
email bodies, 2 new Analytics formula columns and 1 report, doc updates. 0 changes to
M0 to M5 flows.

## Design decisions

D1 to D3 are the choices most worth Costin overriding; each has a default that the data
model and tasks assume.

- **D1: a scheduled Flow plus one claim function, not a CRM time-based workflow.** The
  thing being detected is the absence of an event, so a poll is the only option; M3's
  research rejected polling for triggers because a real event existed there. Considered
  and not chosen: a CRM time-based workflow that stamps the field 4 days after creation
  and a record-triggered Flow listening for the stamp. It would fit the existing
  one-record-per-run pattern, but the Flow trigger fires on every later update of a row that
  already carries the stamp (for example `Submitted`), which makes double sends possible
  unless every step re-checks status, and it depends on the CRM edition supporting
  time-based actions (unverified, research.md section 3). Revisit it if scheduled runs prove
  too costly in Flow tasks.
- **D2: one reminder per run, several runs a day.** Standard plan Flow has no loop, so a run
  sends at most one reminder. The schedule runs hourly from 9 AM to 5 PM Eastern (9
  runs a day). A day's backlog clears within the window; anything left carries to the next
  day, and the 3-day remaining life absorbs that. If Flow's schedule cannot be limited to
  those hours, the function itself refuses to claim outside 9 to 17 Eastern (FR-010).
- **D3: claim before send.** The function stamps `Reminder_Sent_Date_Time` before the flow
  sends, so an overlapping or repeated run can never pick the same row twice. The cost is
  that a failed send still leaves a stamp. By decision a failed reminder is alerted, not
  retried (spec FR-009); a retry loop on a bad address would alert Costin hourly. The
  field therefore means "reminder dispatched", not "delivered".
- **D4: suppression is evaluated, not stored.** The function re-reads the candidate rows
  and the person each run and skips ineligible ones without marking them. The candidate set
  (Issued, unreminded, between day 4 and day 7) is small, so re-evaluation is cheap, and
  nothing new has to be stored for skipped rows. Cost: a person who re-subscribes inside the
  window becomes eligible again (spec edge case).
- **D5: a read-only opt-out check inside the claim function, not `skipIfOptedOut`.**
  `skipIfOptedOut` writes a skip row on opt-out. A reminder must not create rows. The claim
  function reads the same native `Email_Opt_Out` field on `Patients1` or `Leads` itself and
  only decides eligibility. Inlined rather than a second function because Flow custom
  functions are not known to be able to call each other. Same fail-closed behavior.
- **D6: one flow, milestone branches for the email.** Each milestone's survey link and
  wording differ, so the flow branches on the returned milestone into five "Send email"
  steps, which Costin can edit in the Zoho Mail step like every other milestone email. The
  alternative (the function returns subject and body) would put Liana's approved copy in
  code; rejected.
- **D7: non-response is computed in Analytics, not by a status sweep.** An `Issued` row
  past expiry stays `Issued` in CRM (expiry is only written when someone tries to submit
  late). The reporting side adds a formula column that treats `Issued` plus expired as
  lapsed (Principle VII), rather than a job that rewrites statuses.

## Constitution Check

*Gate: checked before research and re-checked after design. Result: PASS with one recorded
deviation.*

| Principle | Assessment |
|---|---|
| I. De-identification by design | Pass. The reminder reuses the opaque token; the collection forms are unchanged and receive nothing new. The email body carries no name or clinical content (FR-012). |
| II. Single rejoin point | Pass. The function reads the token and the address from CRM's own row and hands them to the Mail step, the same exposure the issuing flows already have. No new system holds token plus identity. |
| III. Automated milestone triggers | Pass. Reminders are fired by elapsed time and row state, never by a clinician or staff member (FR-011). |
| IV. Contractor blindness | Pass. Contractors have no CRM, flow or dashboard access; reminder state appears in no contractor view. If contractors ever gain CRM access, hide `Reminder_Sent_Date_Time` from their profile first. |
| V. BAA-gated adoption | Pass, still gated. No new product. This is a new use of Flow (a scheduled flow reading `Milestone_Instances`) and of Mail; BAA confirmation (open constitution TODO) must be recorded before it goes live, as for the rest of the pipeline. |
| VI. Internal use only | Pass. Wording is an operational follow-up, never a request for a testimonial or review (FR-014). Nothing flows to marketing tools. |
| VII. Analytics computes derived values | **Deviation, justified, same shape as features 004 and 006.** `claimNextDueReminder` decides eligibility at send time, before any Analytics value could exist, and its output is an action (send or not), not a score or flag. The reporting derivations (lapsed, answered after a reminder, response rate) are all Analytics formula columns. Recorded here per the Development Workflow. |
| Development Workflow: trigger logic and rules checked against I-IV | Done above. No existing trigger logic changes. |
| Rollout Workflow (specify, plan, tasks) | Followed: spec, plan and tasks before any Zoho change, except the one CRM field, created at Costin's direction on 2026-10-09. |

Post-design re-check: PASS.

## Project Structure

```text
specs/007-feedback-reminders/
├── spec.md
├── plan.md                  # this file
├── research.md              # how the pipeline works today; facts this design rests on
├── data-model.md            # field, functions (draft source), flow, email copy, Analytics
├── quickstart.md            # structural checks and live test script
├── tasks.md
└── checklists/requirements.md
```

Also updated in the same change: `specs/001-feedback-collection-pipeline/spec.md` (ninth-pass
revision note, FR-002 clarification, Assumptions entry). Implementation notes for this
feature (`implementation-notes.md`) are created when the build starts, per `CLAUDE.md`.

## Open Questions and Resolutions

1. **Two open requests at once.** *Resolved 2026-10-09 (Costin):* both are reminded
   independently; no hold-back rule.
2. **Flow task budget.** *Still open.* 9 scheduled runs a day is roughly 270 runs a month
   before any reminder is sent. Please confirm the workspace's current monthly task usage
   (limit noted as 5,000) so we can confirm there is room, or choose fewer runs (for example
   4 a day). Tracked as T005.
3. **Reminder skips in reporting.** *Resolved 2026-10-09 (Costin):* acceptable that skips
   caused by the M3 brake or a returned M5 patient leave no record and are not reported.
4. **Field label.** *Resolved 2026-10-09 (Costin):* rename to "Reminder Dispatched Date Time".
   The API name `Reminder_Sent_Date_Time` stays, since CRM keeps the API name when a label
   changes. The rename is a CRM Setup change (T003); until it is made, docs may still show
   the original label "Reminder Sent Date Time".
