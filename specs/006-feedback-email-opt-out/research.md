# Research: Feedback Email Opt-Out

**Feature**: `006-feedback-email-opt-out` | **Date**: 2026-10-01

## 0. What was checked

Read from this repo: the constitution, spec 001 and each milestone's implementation notes,
features 002 to 005, `CLAUDE.md`, and `data-retention-purge.md`. Read from CRM (read-only,
non-PII module): the `Milestone_Instances` field list. Read from the Campaigns repo
(`costin-ccip/cape-clarity-zoho-campaigns`) during scoping: its spec, constitution and audience
contract, to confirm the two projects stay walled off. No patient or lead records were read.

## 1. How feedback emails and tokens work today (facts this design depends on)

- Each milestone trigger flow calls the shared custom function `issueFeedbackToken`, then a
  Zoho Mail "Send email" step whose link ends `?token=${issueFeedbackToken_1.token}`
  (`CLAUDE.md`, feature 002). `issueFeedbackToken` creates one `Milestone_Instances` row per
  email with `Token`, `Expiry_Date_Time` (7 days), `Status = Issued`, `Email`, `Clinician`, and
  either `Patient` (lookup to `Patients1`) or `Lead_Reference` (a plain text field holding the
  Lead's record ID, not a lookup; confirmed in the live field list).
- Write-back flows move the row to `Submitted` or `Expired`. They do not delete the row or
  clear the token, so an old token still points at its row. (Confirm in T008.)
- Live `Status` picklist values: Issued, Submitted, Expired, Superseded, Send Failed,
  Captured Live. Only `Name` is system-mandatory, so a skip row can omit Token, Email and
  expiry.
- Survey forms take the token from a hidden form field filled by the URL: Zoho Forms
  Settings, Prefill, Field Alias, with the URL parameter matching the alias (`m2-implementation-notes.md`
  §13). The opt-out form needs the same setup.
- The survey forms are on Zoho Forms' Free plan (`data-retention-purge.md`). Raw form entries
  are purged by hand; the new form's entries (token only) join that routine.

## 2. How each milestone decides to fire (where the check has to sit)

| Flow | Today | Where `skipIfOptedOut` goes |
|---|---|---|
| M0 | Lead status change to Lost Lead with Consult Call Date filled, then `issueFeedbackToken`, then email. No idempotency check and no If-else; a later update to the same Lead fires it again (`m0-implementation-notes.md`). | New function and If-else between the trigger and `issueFeedbackToken`. |
| M2 | `checkAllianceCheckExists`, `isCreatedAfterCutoff`, If-else, then issue and send. | Inside the If-else True branch. |
| M3 | `checkPeriodicCheckInDue` (counts the person's existing M3 rows, next checkpoint is `[8,16,24,32,40][count]`, fires if `sessionCount >= next`), `isCreatedAfterCutoff`, If-else. | Inside the True branch. |
| M4 | `checkDischargeExists`, cutoff gate, If-else. | Inside the True branch. |
| M5 | `checkDiscontinuationExists`, cutoff gate, If-else. | Inside the True branch. |

Placing the check inside the True branch matters: a skip row is written only when the
milestone would genuinely have sent. Checking earlier would write a skip row on every
unrelated update to an opted-out person's record.

## 3. Why a skip row works with the existing idempotency checks (decision D4)

All four existence checks, and M3's count, ask "does a `Milestone_Instances` row exist for this
person and this milestone?" and ignore Status. So a row with Status `Skipped - Opted Out`:

- **M2, M4, M5**: the existence check becomes true, so the flow never fires again for that
  milestone, including after a resubscribe. That is exactly spec FR-010 and Story 2 scenario 3
  (never sent retroactively), with no change to those functions.
- **M3**: the row counts toward `existingCount`, so the next checkpoint index advances past
  the skipped one. Example: opted out at 8 sessions, so checkpoint 8 is skipped (one row);
  resubscribes at 20 sessions; checkpoint 16 would be next and `20 >= 16`, so it would fire.
  To keep checkpoints from backfilling, `skipIfOptedOut` is called on every firing while the
  person is opted out, so each update that finds a checkpoint due writes a skip row and
  advances the index (the same one-checkpoint-per-firing behavior `checkPeriodicCheckInDue`
  already has, documented in `coordinated-live-test-plan.md` Scenario 2). A person opted out
  across a long gap will therefore have a skip row for every checkpoint their session count
  passed while opted out, provided their record was updated at least once per checkpoint.
  A residual case remains: if the session count jumps over several checkpoints in a single
  update while opted out, only one row is written for that update; the rest are written on
  later updates. After a resubscribe those unwritten checkpoints could still fire, one per
  update. This is the same one-at-a-time behavior the existing function has for any jump; it is
  accepted and recorded (quickstart §C step 9 exercises it).
- **M0**: no existence check, so `skipIfOptedOut` itself de-duplicates: for any milestone other
  than M3 it does not write a second skip row if one already exists for that person and
  milestone.

## 4. Why one confirmation message (decision D3)

Zoho Forms can show one fixed thank-you message (or redirect to one fixed URL) after submit.
It cannot ask CRM whether the token matched and choose a message. Options considered:

| Option | Verdict |
|---|---|
| One neutral message, true whether or not the token matched, plus an alert to Costin on no match | **Chosen.** Meets FR-007 (nothing recorded or disclosed about anyone) and FR-008 (says what stopped, how to reach us) together. |
| Redirect to a Flow- or Webflow-hosted page that looks up the token and shows a matched or unmatched message | Rejected for now: needs a public endpoint that calls CRM, which means a new component and a larger security review for a rare case. Possible later. |
| Validate the token before showing the form | Not possible in Forms. |

Draft message (Liana to approve, spec Assumptions): "Thank you. We've received your request
and will stop sending feedback emails. If you didn't mean to do this, or you'd like to start
receiving them again, just contact us at info@capeclarity.com." The exact wording and the
contact line are tone-of-voice items for Liana.

## 5. Why not the alternatives (summary of scoping, for the record)

- **Native Zoho Campaigns preference-center category**: rejected at scoping (spec header
  comment). Audience scope, BAA gate, sender mismatch, and the feedback/marketing wall.
- **A separate permanent token per person (D1 alternative)**: would need a new field on
  Leads and Patients, token generation at some point before the first email, and a change to
  shared issuance. The existing per-email token already identifies the person; reusing it as
  a locator avoids all of that. Cost: the opt-out link accepts a token whose survey use has
  ended. That is intended (FR-006) and the function only ever reads the row and the person.
- **`issueFeedbackToken` returning "skipped" (D2 alternative)**: see plan.md.
- **Unsubscribe by replying "stop" automatically**: out of scope (spec). Replies go to
  `info@capeclarity.com` and staff handle them.

## 6. Where the opt-out data lives and who can see it

- **Fields on the person's record** (Leads and Patients1): easy for staff to set by hand
  (FR-013), read live by the flow at firing time, no sync lag.
- **CRM field history**: the date fields give "when", and the CRM timeline shows who changed
  the flag and when. Whether field-history tracking is available on the current CRM plan is
  unconfirmed; the resubscribe-date field exists so the history is discoverable either way
  (FR-015).
- **Not in Analytics**: the Leads sync already carries an empty Email column and a de-identified
  last name (feature 003, T013). The opt-out fields are left out of the sync field list so
  the status of a person never reaches a reporting tool. Admin counts come from skip rows.
- **Not in Campaigns**: the Campaigns audience contract maps a fixed field list; the new
  fields are not added. A note in this feature's quickstart reminds the Campaigns build not to.
- **Contractors**: contractors have no feedback access (constitution Principle IV). Whether any
  contractor profile can open Leads or Patients in CRM at all is not recorded in this repo, so
  the plan sets the four fields invisible on contractor profiles regardless (T007).

## 7. Reporting (FR-010)

Existing per-milestone "Status Breakdown" reports group `Milestone_Instances` by Status, so the
new value should appear as its own slice without changes. What has to be checked is any report,
formula or metric that computes a non-response or completion rate as "everything that is not
Submitted", which would otherwise count skips as non-responses. T030 audits every report and
formula that touches Status, including feature 005's contractor reports, and records the result.

## 8. Open items carried into the tasks

- Whether patient intake already asks about feedback emails, and where intake is captured
  (spec FR-018): T028.
- CRM plan support for field-history tracking: T006 note.
- The Deluge calls `getRecordById` and `updateRecord` on `Leads` and `Patients1` must be
  validated in the function editor before use; the signatures in data-model.md follow the
  patterns already used in this pipeline, but the 4-argument `getRecordById` form is the one
  call not yet used elsewhere here.
- Pre-existing 200-record scan limit in older functions: T040.
