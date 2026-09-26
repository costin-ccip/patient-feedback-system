# Coordinated Live Test Plan — M0, M2-M5 + Cross-Cutting Features

> Assembled 2026-09-25, ahead of Costin's first coordinated end-to-end test of the
> whole feedback pipeline. Every milestone (`m0`/`m2`/`m3`/`m4`/`m5`-implementation-notes.md)
> and every cross-cutting feature (`specs/002-remove-subflow-dependency`,
> `specs/003-issuance-privacy-and-failure-handling`, `specs/004-legacy-patient-exclusion`)
> already has its own "structural checks done / live test still needed" quickstart
> section. This file is the missing piece: one ordered runbook tying all of those
> together, plus the findings from a fresh structural re-verification pass done
> the same day. Nothing in this file was executed live — per standing instruction,
> no live/end-to-end test runs without Costin.

## 0. Pre-flight findings (2026-09-25 structural pass)

Verified via Zoho CRM MCP tools only (`getFields`, `getRecords` on `Milestone_Instances`;
`getFields` on `Patients1` with Costin's explicit one-time permission — schema only, no
patient records read). No browser access to Leads/Patients. No live test actions taken.

### 0.1 Confirmed consistent with the implementation notes
- `Milestone_Instances.Status` picklist includes `Send Failed` (feature 003 built correctly).
- `Milestone_Instances.Milestone` picklist has all 6 values, including the dead
  `5A - Discontinuation, Live Capture` leftover config, unchanged.
- Zero `Milestone_Instances` records exist for M3, M4, or M5 — consistent with "never
  tested" across all three milestones' notes.
- `Patients1.Patient_Status` picklist confirms `Completed Treatment` and
  `Discontinued (Patient Choice)` (M4/M5 trigger filter values) are live, exact-match values.
- `Patients1.Session_Count` is a plain integer field, as every M2/M3 note assumes.

### 0.2 New finding: Clinician is hardcoded, but the practice now has 3 contractors — FIXED 2026-09-25
Every trigger flow's `issueFeedbackToken` call (M0, M2, M3, M4, M5 — see each
milestone's implementation notes) passes `clinician` as a **typed literal**,
`"Liana Preudhomme"` — never derived from the record that fired the trigger.
`Milestone_Instances.Clinician` picklist has grown to 3 values: `Liana Preudhomme`,
`Deborah Webster`, `Shana Lacastro` (Deborah and Shana are onboarding per
`m2-implementation-notes.md`-era docs, not reflected in any flow). Confirmed via
`Patients1.getFields`: there's an exact-match field for this —

```
Assigned_Therapist (picklist): -None-, Liana Preudhomme, Deborah Webster, Shana Lacastro
```

— same 3 real values, same spelling, as `Milestone_Instances.Clinician`. (`Owner`
also exists on Patients1 but is the CRM record-owner field, not the assigned
therapist — not the right field.)

**Decision (Costin, 2026-09-25):** fix this before the coordinated test, for
M2/M3/M4/M5 only. **M0 stays hardcoded** — it triggers off Leads (no assigned-therapist
concept pre-conversion) and Liana is currently the only one running free consults.

**Fix spec** (not yet built — needs the Zoho Flow builder, browser automation; MCP has
no Zoho Flow access):

For each of "M2 - Session 3 Trigger", "M3 - Periodic Check-In Trigger",
"M4 - Discharge Trigger", "M5 - Discontinuation Trigger": open the `issueFeedbackToken`
step's parameter panel, and replace the `clinician` field's literal text
`Liana Preudhomme` with the chip for the trigger's own `Assigned_Therapist` field
(`${trigger.Assigned_Therapist}` — insert via the Insert Variable panel the same way
`patientId`/`recipientEmail` are already mapped from `${trigger.id}`/`${trigger.Email}`
on each of these flows; do not type it as literal text, per the documented text-field
corruption gotcha in `m2-implementation-notes.md` §3.3).

No CRM schema change needed (the field and its values already exist and already match).
No Analytics change needed (Clinician is stored as submitted, not derived). Each flow
stays OFF after the edit, per the existing build-first-verify-later convention. Update
each milestone's implementation-notes file in the same session this is built, per
`CLAUDE.md`'s convention — this is exactly the kind of "change how a milestone actually
works" edit that rule covers.

**Structural verification after building**: reopen each of the 4 flows' `issueFeedbackToken`
parameter panel and confirm the `clinician` field shows a chip (not literal text) referencing
`Assigned_Therapist`; confirm via the same discipline the rest of this project uses
(read the live value back, don't trust a screenshot alone).

**Built and verified 2026-09-25.** All four flows ("M2 - Session 3 Trigger", "M3 -
Periodic Check-In Trigger", "M4 - Discharge Trigger", "M5 - Discontinuation Trigger")
now map `clinician` to the chip `Updated module entry → Assigned Therapist`
(`${trigger.Assigned_Therapist}`) instead of the literal `Liana Preudhomme`. Each
flow's parameter panel was reopened after saving and the chip confirmed present
(not just trusted from the save screenshot) — M2 and M3 confirmed inline during the
build; M4 and M5 confirmed in a follow-up pass. All four flows remain **OFF**. M0
was left hardcoded, per Costin's decision above. Full change record, per flow:
`m2-implementation-notes.md` §12, `m3-implementation-notes.md` §8,
`m4-implementation-notes.md` §8, `m5-implementation-notes.md` §7.

### 0.3 Stale test data — deletion attempted, blocked, needs Costin

`Milestone_Instances` currently holds 9 leftover records, none newer than 2026-09-12:

| id | Milestone | Status | Note |
|---|---|---|---|
| 6825601000004284001 | 2 - Early Alliance Check | Issued | Leftover from the M2 build session (`m2-tasks.md` T023 never run) |
| 6825601000004245001 | 1 - Baseline Intake | Submitted | Leftover under the now-retired M1 milestone |
| 6825601000004191001 | 0 - No Conversion | Issued | Real-looking hex token, created 2026-09-07 — **not** one of the documented dashboard samples |
| 6825601000004180001 | 0 - No Conversion | Issued | Same as above, created 2026-09-06 |
| 6825601000004179001 | 0 - No Conversion | Issued | Same as above, created 2026-09-06 |
| 6825601000004156001-004 | 0 - No Conversion | Submitted | The 4 documented M0 dashboard-sample records (`m0-implementation-notes.md` §8) |

Costin approved deleting all 9 via CRM MCP as pre-flight cleanup. The bulk `deleteRecords`
call was **blocked by this session's own auto-mode safety classifier** ("Unverifiable
Deletion Scope") before it reached Zoho — nothing was deleted. Per the classifier's own
instructions, this session will not retry the same deletion through another tool or in
smaller pieces. **Costin: this needs your direct action** — either approve the deletion
when it comes up for confirmation, or delete these 9 records yourself (CRM UI or asking
a session running with the right permission level). Three of the 9 (the `Issued`,
non-sample M0 records) are also worth a quick look before deleting — they don't match
the documented sample-data pattern and may reference real leads; this session can't
open Leads to check without your permission.

### 0.4 M0 live test in progress — form link bug found and fixed (2026-09-25)

Costin began running this plan manually himself (per standing instruction, he runs
live test actions, not Claude) and reached M0's Step A before this file's other
sections were fully worked through. He received the token-issuance email as
expected, but the survey link 404'd — a genuine Zoho-side bug on that one form
object, not a config error (full diagnosis, evidence, and fix in
`m0-implementation-notes.md` §13). Fix: the M0 form was rebuilt as a new Zoho Forms
object (Duplicate), the new permalink was verified to resolve (HTTP 200), "M0 -
Feedback Survey Write-back"'s trigger was repointed at the new form (field mappings
re-verified, not assumed), and "M0 - Lost Lead Feedback Token"'s email link was
updated to the new permalink. Both forms were renamed so the display title Costin
sees is unchanged (`M0 - Free Consult Non-Conversion Survey`); only the underlying
object changed. **Costin can resume Step A from where he left off** — set the test
Lead's `Lead_Status` to `Lost Lead` again (or re-click a freshly issued link) to
get a working survey URL. Everything else in Step A (idempotency/supersede check,
write-back behavior, failure path) is unaffected and still needs live verification.

## 1. Test data plan

- **One test Patient**, walked through the lifecycle sequentially (Costin's
  choice): `Session_Count` → 3 (M2), then 8 and 16 (M3, two checkpoints),
  then `Patient_Status` → `Completed Treatment` (M4), then `Patient_Status` →
  `Discontinued (Patient Choice)` (M5). Each milestone's own idempotency check
  is keyed to its own `Milestone` value, so this sequence exercises all four
  independently on one record without collisions. Needs: this test Patient's
  CRM record ID, its `Assigned_Therapist` value (pick one of the 3 — ideally
  **not** Liana, to also exercise the §0.2 fix), and a real-but-controlled
  inbox for the Email field so Costin can see/click the actual survey links.
- **One test Lead**, separately, for M0 (M0 triggers off Leads, unaffected by
  the above).
- Both provided by Costin per standing instruction — this session does not
  look one up.

## 2. Pre-test checklist (before flipping anything ON)

- [x] §0.2 Clinician-derivation fix built and structurally verified on M2/M3/M4/M5's trigger flows (2026-09-25).
- [ ] §0.3 Stale records cleared (Costin's action).
- [ ] Test Patient ID + Assigned_Therapist value + test email, and test Lead ID, provided by Costin.
- [ ] Confirm current state: all 9 flows (M0/M2/M3/M4/M5 trigger + write-back pairs, minus
      M0/M1 which only issue) still OFF (re-verify via browser, not assumed from notes).
- [ ] Costin decides go/no-go on the two still-open compliance gaps below (§4) before any
      of this touches real, non-test patient data.

## 3. Runbook (Costin runs the live actions; this session verifies structurally in between)

Flip ON only the flow(s) needed for the step being tested, not all 9 at once — makes
failures easier to attribute.

### 3.0 The full journey, and how to verify each stage

Every milestone's happy path is the same six-stage journey. This section describes it
once so the steps below don't have to repeat it — "tested end to end" means confirming
all six stages, not stopping once an email arrives or a CRM record looks right.

1. **Trigger fires** — a CRM field change (M0: Lead `Lead_Status`; M2/M3: Patient
   `Session_Count`; M4/M5: Patient `Patient_Status`) matches the flow's trigger filter
   and, for M2-M5, the milestone's own eligibility function (idempotency check, and for
   M2/M3/M4/M5 also `isCreatedAfterCutoff`) returns true.
2. **`Milestone_Instances` record created**, Status `Issued`, via the shared
   `issueFeedbackToken` function — token + expiry set, `Clinician` populated
   (hardcoded for M0 by design, derived from `Assigned_Therapist` for M2-M5 per
   §0.2's fix), `Name` follows the token-prefix pattern with no email in it
   (feature 003).
3. **Email sent** to the patient/lead's address, with the survey link and
   `?token=...` appended.
4. **Patient submits the form.** The milestone's write-back flow's "Form entry
   submitted" trigger fires, calls its `submitFeedbackResponse`-family function,
   which looks up the record by token, checks `Status == Issued` and expiry, and on
   success writes `Response_Data`, sets `Status` → `Submitted`, and stamps
   `Submitted_Date_Time`.
5. **Zoho Analytics syncs from CRM — and this is NOT real-time.** Confirmed live
   2026-09-25: the "Zoho CRM" data source on the shared Analytics workspace
   (`3251423000000083002`) syncs on a fixed **daily schedule, 5:00 PM EDT**, not on
   every CRM write (`Schedule: Daily at 17:00 hrs EST`, checked via the workspace's
   own Data Sources panel — not documented anywhere before this). Don't wait up to
   24 hours for a change to show up during this test — trigger it manually instead:
   in the Analytics workspace, left nav → **Data Sources** → **Zoho CRM** row →
   **Sync Now**. Wait for "Data Sync Successful" and a fresh "Last Data Sync Time"
   before checking any dashboard below.
6. **Dashboard/report reflects it.** Only after step 5 — open the milestone's
   dashboard and confirm the new record appears in every panel that should show it
   (status breakdown, submitted-responses table, any distribution charts, and the
   shared "M2, M3 & M4 Flagged for Review" report where applicable) — not just that
   the flow ran and the CRM record looks right. Panel names/view IDs are listed
   per milestone below and in each milestone's own implementation-notes "Analytics"
   section.

Each Step below marks its happy-path scenario `[Full journey]` to mean all 6 stages
get checked, sync included. The other scenarios per step are likely deviations from
the happy path — chosen because they're what "does the flow fire" testing tends to
miss (a response that never arrives, a link reused, a token that outlives its TTL,
a threshold jumped over), not because they're exhaustive.

### Step A — M0 (Lead-based)

Dashboard: **"M0 - Free Consult Non-Conversion Feedback"** (view `3251423000000083524`)
— panels: M0 Submitted Responses, M0 Biggest Factor, M0 Feeling Heard Distribution,
M0 % Reachable, plus the pre-existing Status Breakdown/Response Rate/Volume-by-Week.

**Scenario 1 — Happy path `[Full journey]`**
1. Turn ON "M0 - Lost Lead Feedback Token" and "M0 - Feedback Survey Write-back".
2. Set the test Lead's `Lead_Status` to `Lost Lead`.
   **Expect**: 1 new `Milestone_Instances` record (`0 - No Conversion`, Issued,
   `Lead_Reference` = lead ID, token + expiry set, `Name` = `Milestone 0 - No Conversion - <8 hex>`
   with no email in it per feature 003), 1 email from `info@capeclarity.com` with the
   (now-fixed, §0.4) survey link.
3. Submit the survey via the emailed link. **Expect**: write-back flow marks it
   Submitted, `Response_Data` filled per `m0-implementation-notes.md` §4's blob format.
4. Sync Analytics (§3.0 stage 5) and open the M0 dashboard above. **Expect**: the
   submission appears in M0 Submitted Responses, and — depending on the answers
   given — M0 Biggest Factor / M0 Feeling Heard Distribution / M0 % Reachable all
   move.

**Scenario 2 — Re-trigger before response (supersede)**
5. On a fresh test Lead, set `Lead_Status` to `Lost Lead`, then set it again before
   submitting. **Expect**: the first record → Superseded, the second → Issued — only
   the newest Issued record's link should still work; the superseded one's link
   should now be rejected if submitted (`Status != Issued`).

**Scenario 3 — Issued, never submitted (non-response)**
6. Issue a token to a third test Lead and don't submit it. **Expect**: it stays
   `Issued`. After an Analytics sync, confirm it shows up as `Issued` in the M0
   Status Breakdown panel rather than being absent from the dashboard entirely —
   this is the case most likely to be missed if reports are only ever eyeballed
   right after a submission.

**Scenario 4 — Duplicate/late submission**
7. After Scenario 1's link has been submitted once, load the same link again and
   resubmit. **Expect**: rejected (`Status != Issued` in `submitFeedbackResponse`)
   — `Response_Data`/`Submitted_Date_Time` must NOT change a second time, and the
   CRM record stays exactly as Scenario 1 left it.

**Scenario 5 — Expired token**
8. Pick one Issued test record and, via CRM MCP tools (not the browser), edit its
   `Expiry_Date_Time` to a past timestamp — waiting out the real 7-day TTL
   (`m0-implementation-notes.md` §12) isn't practical here. Submit that token.
   **Expect**: rejected, and the record auto-transitions to `Expired`
   (`m0-implementation-notes.md` §3.1).

**Scenario 6 — Issuance/send failure (optional; shared logic, worth testing once)**
9. Issue to a Lead with an invalid email. **Expect**: `Status` → `Send Failed`, alert
   email to `costin@capeclarity.com` naming milestone + record ID only, no patient
   email sent (per `specs/003.../quickstart.md` §B.2). This is shared infrastructure
   (`issueFeedbackToken`'s On Error branch, identical on every milestone per
   `CLAUDE.md`) — confirming it once here is representative; no need to repeat it
   identically on M2-M5 unless something about a specific milestone's wiring is in
   doubt.

**Standard scenario set, referenced from Steps B-E below**: Scenarios 2-6 above
(re-trigger/supersede, non-response, duplicate submission, expired token,
send-failure) apply the same way to every other milestone — same mechanics, just
substitute that milestone's own trigger field/value and `Milestone_Instances`
record. Steps B-E call out only what's genuinely different for that milestone
(milestone-specific eligibility logic, extra Analytics gates) rather than repeating
the same 5 scenarios five times.

### Step B — M2 (Session 3 / Early Alliance Check)

Dashboard: **"M2 - Early Alliance Check Feedback"** (view `3251423000000141282`) —
panels: M2 Status Breakdown, M2 Alliance Check-In Total Distribution, M2 Submitted
Responses, plus the shared "M2, M3 & M4 Flagged for Review" report.

**Scenario 1 — Happy path `[Full journey]`**
1. Turn ON "M2 - Session 3 Trigger" and "M2 - Alliance Check-In Write-back".
2. Set the test Patient's `Session_Count` to 3. **Expect**: 1 Issued
   `2 - Early Alliance Check` record, `Clinician` = the test Patient's actual
   `Assigned_Therapist` (confirms §0.2's fix), 1 email.
3. Submit the survey. **Expect**: write-back marks Submitted, `Response_Data` filled
   per `m2-implementation-notes.md` §3.4.1.
4. Sync Analytics (§3.0 stage 5) and open the M2 dashboard above. **Expect**: the row
   appears in M2 Submitted Responses and M2 Status Breakdown, and M2 Alliance
   Check-In Total Distribution moves.

**Scenario 2 — Idempotency (no duplicate issuance)**
5. Re-save `Session_Count` at 3 again (no change). **Expect**: no second record —
   the flow's own idempotency check, not the supersede logic, should suppress this
   (distinct from Step A Scenario 2, which is a genuine re-trigger before response).

**Scenario 3 — Cutoff-exclusion (legacy patient)**
6. Per `specs/004.../quickstart.md` §C: confirm with Costin whether the test
   Patient's `Created_Time` is before or after 2026-09-23. If before, this step
   should produce **no** record at all regardless of `Session_Count` — if that's not
   what you want to test here, use a Patient created on/after 2026-09-23 instead.

**Scenario 4 — Clinical Safety Flag**
7. Submit (or edit in test data) at least one response where the alliance total is
   ≤20 or any domain ≤4. **Expect**: the "Clinical Safety Flag" formula column reads
   `true`, "Flag Rule Triggered" names the specific rule(s), the row appears in
   "M2, M3 & M4 Flagged for Review" after syncing, and — critically — no
   contractor-facing notification of any kind fires (there is none in the build to
   fire; confirm no side effect appears anywhere).

**Scenarios 5-9 — standard set**: re-trigger/supersede, non-response, duplicate
submission, expired token, send-failure — same as Step A Scenarios 2/3/4/5/6, on
this milestone's own trigger/record.

### Step C — M3 (Periodic Consolidated, every 8th session)

Dashboard: **"M3 - Periodic Check-In Feedback"** (view `3251423000000186289`) —
panels: M3 Status Breakdown, M3 Practice Experience: Scheduling/Communication
Distribution, M3 Practice Experience: Billing Distribution, M3 Therapist
Professionalism Distribution, M3 Submitted Responses, plus the shared "M2, M3 & M4
Flagged for Review" report.

**Scenario 1 — Happy path, two checkpoints `[Full journey]`**
1. Turn ON "M3 - Periodic Check-In Trigger" and "M3 - Periodic Check-In Write-back".
2. Set `Session_Count` to 8. **Expect**: 1 Issued `3 - Periodic Consolidated`
   record, 1 email.
3. Set `Session_Count` to 16. **Expect**: a second Issued record at the next
   checkpoint (`checkPeriodicCheckInDue`'s checkpoint-list logic —
   `m3-implementation-notes.md` §1.5).
4. Submit both surveys. **Expect**: write-back fills all 7 segments correctly (4
   reused alliance domains + 3 new Practice Experience/Professionalism fields) —
   this is the first real-data confirmation that the reused M2 Analytics formula
   columns parse M3 rows correctly (flagged as unverified in
   `m3-implementation-notes.md` §1.2).
5. Sync Analytics (§3.0 stage 5) and open the M3 dashboard above. **Expect**: both
   submissions appear in M3 Submitted Responses/Status Breakdown, and all three
   distribution panels move.

**Scenario 2 — Skipped checkpoint**
6. On a fresh test Patient (or after resetting), set `Session_Count` directly to a
   value that jumps past an unclaimed checkpoint — e.g. straight to 20 without ever
   passing through 8 or 16. **Expect**: `checkPeriodicCheckInDue` only issues **one**
   record, for the next unclaimed threshold (8, since `existingCount` is 0) — it
   does NOT backfill a second record for 16 just because 20 also clears it. This
   is a direct reading of the function's logic (§3.0/`m3-implementation-notes.md`
   §1.5: `nextThreshold = checkpoints.get(existingCount)`, evaluated once per
   trigger firing) but has never been exercised live — worth confirming it behaves
   as the code implies rather than assuming.

**Scenario 3 — Cutoff-exclusion**
7. Same check as Step B Scenario 3, for this milestone's own trigger.

**Scenarios 4-8 — standard set**: Clinical Safety Flag (on either checkpoint's
submission), re-trigger/supersede, non-response, duplicate submission, expired
token, send-failure — same as Step A/B, on this milestone's own records.

### Step D — M4 (Discharge)

Dashboard: **"M4 - Discharge Feedback"** (view `3251423000000232202`) — panels: M4
Status Breakdown, M4 Looking Ahead: Likelihood To Recommend Distribution, M4
Submitted Responses, plus the shared "M2, M3 & M4 Flagged for Review" report.

**Scenario 1 — Happy path `[Full journey]`**
1. Turn ON "M4 - Discharge Trigger" and "M4 - Discharge Write-back".
2. Set `Patient_Status` to `Completed Treatment`. **Expect**: 1 Issued `4 - Discharge`
   record, 1 email.
3. Submit the survey. **Expect**: write-back fills Looking-Ahead fields + reused
   alliance domains (`m4-implementation-notes.md` §3) — first real-data confirmation
   for M4's reused columns (flagged unverified, T020 in `m4-tasks.md`).
4. Sync Analytics (§3.0 stage 5) and open the M4 dashboard above. **Expect**: the row
   appears in M4 Submitted Responses/Status Breakdown, and the Looking-Ahead
   distribution panel moves.

**Scenario 2 — Cutoff-exclusion**
5. Same check as Step B Scenario 3 — M4 also gates on `isCreatedAfterCutoff`
   (confirmed in `m4-implementation-notes.md`, not just M2/M3 as `CLAUDE.md`'s
   older summary implies).

**Scenario 3 — Clinical Safety Flag**
6. Same as Step B Scenario 4 — M4 reuses the shared alliance-domain columns, so a
   low-total M4 response should also land in "M2, M3 & M4 Flagged for Review".

**Scenarios 4-7 — standard set**: re-trigger/supersede, non-response, duplicate
submission, expired token, send-failure — same as Step A, on this milestone's own
records.

### Step E — M5 (Discontinuation)

Dashboard: **"M5 - Discontinuation Feedback"** (view `3251423000000232368`) —
panels: M5 Status Breakdown, M5 Reason For Leaving Distribution, M5 Submitted
Responses, M5 Okay To Reach Back Out Breakdown. **No shared Flagged for Review
panel** — not applicable to M5 (`m5-research.md` Decision 6).

**Scenario 1 — Happy path `[Full journey]`**
1. Turn ON "M5 - Discontinuation Trigger" and "M5 - Discontinuation Write-back".
2. Set `Patient_Status` to `Discontinued (Patient Choice)`. **Expect**: 1 Issued
   `5B - Discontinuation, Email Fallback` record, 1 email (subject "Just checking
   in").
3. Submit the survey (2 fields only — reason + okay-to-reach-back-out). **Expect**:
   write-back fills `Response_Data`.
4. Sync Analytics (§3.0 stage 5) and open the M5 dashboard above. **Expect**: the row
   appears in M5 Submitted Responses/Status Breakdown, and M5 Reason For Leaving
   Distribution / M5 Okay To Reach Back Out Breakdown both move.

**Scenario 2 — Issued, never submitted (non-response) — already flagged as a
required check, not optional**
5. On a second test Patient, issue but don't submit. **Expect**: still shows up as
   "issued but not submitted" in M5 Submitted Responses/M5 Status Breakdown rather
   than silently vanishing — this is an explicit product requirement (FR-015), not
   just a good-practice check like Step A Scenario 3.

**Scenario 3 — Cutoff-exclusion**
6. Same check as Step B Scenario 3, for M5's own trigger.

**Scenarios 4-6 — standard set**: re-trigger/supersede, duplicate submission,
expired token — same as Step A. (Send-failure already covered once in Step A
Scenario 6; M5 has no Clinical Safety Flag panel to re-check, per the dashboard
note above.)

### Step F — Dashboard/access checks (Story 3/User Story 4, cuts across all of the above)
1. As admin: open each milestone's dashboard (listed at the top of Steps A-E above),
   confirm the new test data renders in every panel (not just "No Data Available"
   anymore) — this doubles as the final confirmation of §3.0 stage 6 for every
   scenario run above, not only the happy paths.
2. Confirm the "M2, M3 & M4 Flagged for Review" report shows the flagged row(s) from
   Steps B/C/D's Clinical Safety Flag scenarios, with no Patient/identity column.
3. **Negative test, needs a contractor login**: confirm a contractor account
   (Deborah's or Shana's, if one exists) cannot reach any of these dashboards/reports
   at all (FR-009/SC-004). This session cannot do this step — needs Costin or a
   contractor to attempt it.

### Step G — Cleanup
- Delete all test `Milestone_Instances` records created above via CRM MCP tools (not browser).
- Decide, per flow, whether to leave it ON (go-live) or switch back OFF pending further
  review — record that decision in each milestone's implementation-notes file.

## 4. Open items this test does NOT resolve

These are known, already-documented gaps, unrelated to whether the pipeline mechanically
works — worth Costin's explicit go/no-go before any of the above touches real patient data,
not just test data:

- **Raw Zoho Forms retention purge** (`data-retention-purge.md`): no purge mechanism exists
  yet for either the "short window" the constitution requires or the 24-hour working target.
  Logged as an accepted open limitation since 2026-09-06, not resolved by this test.
- **`TODO(BAA_SCHEDULE)`** (constitution Principle V): which specific Zoho products are
  actually named in Cape Clarity's BAA schedule is still unconfirmed.
- **M5 no-show leg**: this build only covers the cancellation-with-no-rebooking leg of
  Milestone 5; the "2 consecutive no-shows" leg from spec.md's Milestones table was never
  built (`m5-research.md`'s "Open item for spec.md").
