# Implementation Plan: Feedback Email Opt-Out

**Branch**: `006-feedback-email-opt-out` | **Date**: 2026-10-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/006-feedback-email-opt-out/spec.md`

## Summary

Give every person who receives a feedback email a way to stop all of them, and make every
milestone honor that. The design reuses the pipeline's own pieces and adds no new Zoho
product:

1. **Record**: three small fields on both the Lead and Patient records in CRM (opt-out
   flag, date, source) plus a resubscribe date, and one new `Milestone_Instances.Status`
   value, `Skipped - Opted Out`.
2. **Opt out**: each feedback email gets a second link to a new Zoho Forms page
   ("Feedback Email Opt-Out"). The page carries only the hidden token already issued for
   that email. Opening it does nothing; pressing the one button submits it. A new Flow,
   "Feedback Opt-Out - Form Submitted", calls a new custom function `recordFeedbackOptOut`
   that matches the token to the person inside CRM and sets the fields.
3. **Honor**: a new shared custom function `skipIfOptedOut` is called by every milestone
   trigger flow right before `issueFeedbackToken`. If the person is opted out it writes a
   `Skipped - Opted Out` Milestone_Instances record and tells the flow to stop; otherwise
   the flow carries on exactly as today.
4. **Other channels**: staff tick the same CRM field (a CRM workflow fills in date and
   source), following a short written procedure. Resubscribe is staff clearing the field.

Nothing about what the emails ask, when milestones fire, or how scores are stored changes
(spec FR-016). `issueFeedbackToken` is not edited.

## Technical Context

**Language/Version**: Deluge (Zoho Flow custom functions); Zoho Forms configuration; Zoho
CRM field and workflow configuration

**Primary Dependencies**: Zoho Flow (Standard plan), Zoho Forms (Free plan, as the survey
forms), Zoho CRM (`crm_connection`), Zoho Mail (existing "Send email" steps). No new
product (spec Assumptions; constitution Principle V).

**Storage**: CRM only. Four new fields on `Leads` and on `Patients1`; one new picklist
value on `Milestone_Instances.Status`. Nothing new in Analytics (see "Reporting").

**Testing**: Structural checks via Execute and flow Test (quickstart.md §A, §B), then a
live run folded into the existing coordinated test (quickstart.md §C), against designated
test patient PT000 and the existing test lead. Live runs need Costin's go-ahead.

**Target Platform**: Zoho Flow workspace `872426000000002011`, folder "Customer Feedback
System"; Zoho Forms account `lianapreudhommecapec1`; CRM org `org889880832`

**Project Type**: No-code/low-code workflow configuration plus two Deluge functions

**Performance Goals**: One extra function call per milestone firing, and one new flow that
runs only on an opt-out submission. Negligible against the 5,000-task/month Flow plan.

**Constraints**: No flow switched ON by this work; no live tests without Costin; no
reading Leads or Patients records beyond designated test records; contractors must never
see opt-out status; nothing synced to Campaigns or Analytics; `issueFeedbackToken` and the
write-back functions untouched.

**Scale/Scope**: 2 new custom functions, 1 new flow, 1 new Forms form, 5 trigger flows
edited (M0, M2, M3, M4, M5), 5 email bodies edited, 8 CRM fields, 1 picklist value, 1 CRM
workflow rule, 1 staff procedure, doc updates.

## Design decisions

These were settled by reading how the pipeline is actually built (see research.md).
D1 to D3 are choices Costin may want to override; each has a default that this plan and
the tasks assume.

- **D1: the opt-out link reuses the milestone instance's token as a locator.** The token in
  the email already identifies one `Milestone_Instances` row, which points to the Patient
  (`Patient` lookup) or the Lead (`Lead_Reference` text field). `recordFeedbackOptOut`
  accepts it whatever the row's Status or expiry, and never changes the row. This meets
  FR-006 (works after expiry or submission) without touching `issueFeedbackToken` or adding
  a second token. Alternative: a separate permanent token per person (needs a change to
  shared issuance, so rejected for now).
- **D2: the opt-out check is its own shared function, called by each flow.** Same pattern as
  `isCreatedAfterCutoff` (feature 004). It sits inside each flow's "should this fire" True
  branch, so a skip is recorded only when the milestone really would have fired. Alternative:
  build the check into `issueFeedbackToken` (rejected: it would change shared issuance for
  all milestones at once, add a new return value every caller must branch on anyway, and
  have a failed check look like a normal send).
- **D3: one neutral confirmation message.** Zoho Forms cannot show a different thank-you page
  depending on whether the token matched. The page therefore shows one message that is true
  in both cases (spec FR-007 and FR-008 together), and an unmatched submission raises an
  identity-free alert to Costin so staff can follow up. See research.md §4.
- **D4: skip record = a normal `Milestone_Instances` row with a new Status.** Because every
  existing idempotency check (`checkAllianceCheckExists`, `checkDischargeExists`,
  `checkDiscontinuationExists`, and M3's `checkPeriodicCheckInDue` count) counts rows for a
  person and milestone regardless of Status, a skip row automatically stops a later
  retroactive send and advances M3's checkpoint index. No change to those functions. See
  research.md §3.
- **D5: opt-out status stays in CRM, out of Analytics.** FR-010 reporting comes from the
  skip rows already in `Milestone_Instances`, so the new person-level fields are not added
  to the Analytics sync and are hidden from contractor profiles.

## Constitution Check

*Gate: checked before research and re-checked after design. Result: PASS.*

| Principle | Assessment |
|---|---|
| I. De-identification by design | Pass. The opt-out form has one hidden field, the opaque token, and nothing else; the page never shows or asks for name, email, phone. The only thing the form stores is the same kind of token the survey forms already hold. |
| II. Single rejoin point | Pass. The token is matched to a person only inside the Flow custom function using the CRM connection, exactly as the survey write-back does. No other system sees token plus identity. |
| III. Automated milestone triggers | Pass. Skipping is automatic and rule-based; no clinician decides. Staff actions only record a person's own request. |
| IV. Contractor blindness | Pass, with a task. The new fields must be set invisible for contractor profiles (T007), and opt-out status appears in no Analytics table or contractor view. |
| V. BAA-gated adoption | Pass. Only Forms, Flow, CRM and Mail are used, all already in the pipeline. No new product. BAA confirmation (still open, constitution TODO) gates go-live exactly as for the existing pipeline. |
| VI. Internal use only | Pass. Nothing is sent to Campaigns, the website or any marketing tool; the opt-out is deliberately independent of marketing preferences (spec FR-012, FR-017). |
| VII. Analytics computes derived values | **Deviation, justified, same as feature 004.** `skipIfOptedOut` is a yes/no gate that must be answered at trigger-decision time, before any Analytics value exists. It derives no score or flag. The reporting side (counts of "opted out") is computed in Analytics from the Status column, per the principle. Recorded here per the Development Workflow. |
| Development Workflow: changes to trigger logic checked against I-IV | Done above. The change touches trigger logic in five flows, so each edited flow must be re-checked against I-IV at build time (tasks T018 to T022). |
| Rollout Workflow (specify, plan, tasks) | Followed: spec, then plan and tasks, before any Zoho change. |

Post-design re-check: PASS. The one deviation is the same shape as feature 004's and is
recorded rather than assumed.

## Project Structure

### Documentation (this feature)

```text
specs/006-feedback-email-opt-out/
├── spec.md
├── plan.md                         # this file
├── research.md                     # how the pipeline works today; why these choices
├── data-model.md                   # fields, status value, function source, form, flow
├── quickstart.md                   # structural checks and the live test script
├── staff-procedure.md              # FR-013 to FR-015 runbook for staff
├── tasks.md
├── checklists/requirements.md
└── implementation-notes.md         # as-built record; stub until work starts
```

### Zoho objects touched

```text
Zoho CRM
├── [NEW]  Leads and Patients1 fields: Feedback_Opt_Out, Feedback_Opt_Out_Date,
│          Feedback_Opt_Out_Source, Feedback_Resubscribe_Date
├── [EDIT] Milestone_Instances.Status picklist: + "Skipped - Opted Out"
├── [NEW]  Workflow rule on Leads and Patients1: fill date/source when the flag is set,
│          fill resubscribe date when it is cleared
└── [EDIT] Profiles: new fields invisible to contractor profiles

Zoho Forms
└── [NEW]  "Feedback Email Opt-Out" form (hidden Token field, one button, thank-you text)

Zoho Flow (workspace 872426000000002011 / Customer Feedback System)
├── [NEW]  custom function skipIfOptedOut
├── [NEW]  custom function recordFeedbackOptOut
├── [NEW]  flow "Feedback Opt-Out - Form Submitted"
├── [EDIT] M0 - Lost Lead Feedback Token          (+ skipIfOptedOut and If else)
├── [EDIT] M2 - Session 3 Trigger                 (same)
├── [EDIT] M3 - Periodic Check-In Trigger         (same)
├── [EDIT] M4 trigger flow                        (same)
├── [EDIT] M5 trigger flow                        (same)
├── [EDIT] the five "Send email" bodies           (+ opt-out link and reply line)
└── [UNTOUCHED] issueFeedbackToken, all idempotency-check functions, all write-back flows,
    isCreatedAfterCutoff

Zoho Analytics: Status Breakdown reports reviewed for the new value; no sync change
Zoho Campaigns: no change
```

**Structure Decision**: a separate feature directory (006), same reasoning as 002 and 004:
it is a cross-milestone behavior, not a milestone. Per-milestone as-built detail will also be
added to each milestone's own `m*-implementation-notes.md` when the flows are edited, per
`CLAUDE.md`.

## Build order

1. **CRM first** (nothing else works without it): fields, status value, workflow rule,
   contractor visibility (Phase 1 of tasks).
2. **Functions**, each verified with Execute before any flow uses it: `skipIfOptedOut`,
   then `recordFeedbackOptOut`.
3. **Opt-out form and handler flow**, built and tested while still switched OFF, so the
   whole opt-out path can be exercised on test records before any milestone flow changes.
4. **Milestone flows one at a time**, simplest first: M2, then M4, M5, M3 (the checkpoint
   counting), M0 last (no idempotency check, so it needs the skip de-duplication). Each
   edit: add the function and If-else, then add the link to that flow's email.
5. **Reporting review** in Analytics, then the staff procedure and intake question.
6. **Live test** (Costin's go-ahead), then docs and `CLAUDE.md` convention note.

Flows stay OFF unless they were already ON; if a flow is ON when edited, Costin decides
the window.

## Risks and how the plan handles them

- **Zoho Flow builder wiring** (known from 002, 003, 004): a node dropped near an endpoint is
  not wired until the connection is drawn. Verify with `jsplumb-connected`, as before. The
  new If-else inside each True branch is the fiddliest edit; do one flow, check it, then repeat.
- **Pre-existing 200-record scan limit.** `checkAllianceCheckExists`, `checkPeriodicCheckInDue`
  and the survey token lookup read `Milestone_Instances` with one page of 200 records. Skip
  rows add to that table. This feature's own functions use `searchRecords` instead (no 200
  cap on matches), but the older functions are unchanged and will start missing rows once the
  table passes 200. Not caused by this feature; logged as a follow-up in tasks (T040).
- **Race window of a few seconds.** The check reads the person's record live at the moment the
  flow runs, so an opt-out recorded before that point always wins. An opt-out landing in the
  seconds between the check and the send cannot be prevented without locking; the email
  already in flight carries the opt-out link. Spec SC-002 is worded accordingly in the
  quickstart.
- **Unknown intake mechanism (FR-018).** Where patient intake is captured, and whether it
  already asks about feedback emails, is not documented in this repo. Task T028 resolves it
  before go-live; it does not block the build.
- **Live changes need Costin.** CRM Setup pages do not load for browser automation, and
  `createFields` over the CRM tool may be refused in some modes, so the CRM tasks are marked
  as Costin's to run or approve.
