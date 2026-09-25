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

### Step A — M0 (Lead-based)
1. Turn ON "M0 - Lost Lead Feedback Token" and "M0 - Feedback Survey Write-back".
2. Set the test Lead's `Lead_Status` to `Lost Lead`.
   **Expect**: 1 new `Milestone_Instances` record (`0 - No Conversion`, Issued,
   `Lead_Reference` = lead ID, token + expiry set, `Name` = `Milestone 0 - No Conversion - <8 hex>`
   with no email in it per feature 003), 1 email from `info@capeclarity.com`.
3. Repeat the same status set once more. **Expect**: first record → Superseded, second → Issued
   (idempotency/supersede check).
4. Submit the survey via the emailed link. **Expect**: write-back flow marks it Submitted,
   `Response_Data` filled per `m0-implementation-notes.md` §4's blob format.
5. (Failure path, optional) Issue to a Lead with an invalid email. **Expect**: `Status` → `Send Failed`,
   alert email to `costin@capeclarity.com` naming milestone + record ID only, no patient email sent
   (per `specs/003.../quickstart.md` §B.2).

### Step B — M2 (Session 3 / Early Alliance Check)
1. Turn ON "M2 - Session 3 Trigger" and "M2 - Alliance Check-In Write-back".
2. Set the test Patient's `Session_Count` to 3. **Expect**: 1 Issued `2 - Early Alliance Check`
   record, `Clinician` = the test Patient's actual `Assigned_Therapist` (confirms §0.2's fix),
   1 email.
3. Re-save at 3 again. **Expect**: no second record (idempotency).
4. **Cutoff-exclusion check** (per `specs/004.../quickstart.md` §C): confirm with Costin
   whether the test Patient's `Created_Time` is before or after 2026-09-23. If before, this
   step should produce **no** record at all regardless of Session_Count — if that's not what
   you want to test, use a Patient created on/after 2026-09-23.
5. Submit the survey. **Expect**: write-back marks Submitted, `Response_Data` filled per
   `m2-implementation-notes.md` §3.4.1.
6. **Clinical Safety Flag check**: submit (or edit in test data) at least one response where
   the total is ≤20 or any domain ≤4. **Expect**: "Clinical Safety Flag" formula column reads
   `true`, "Flag Rule Triggered" names the specific rule(s), the row appears in "M2, M3 & M4
   Flagged for Review", and — critically — no contractor-facing notification of any kind fires
   (there is none in the build to fire, but confirm no side effect appears anywhere).

### Step C — M3 (Periodic Consolidated, every 8th session)
1. Turn ON "M3 - Periodic Check-In Trigger" and "M3 - Periodic Check-In Write-back".
2. Set `Session_Count` to 8. **Expect**: 1 Issued `3 - Periodic Consolidated` record, 1 email.
3. Set `Session_Count` to 16. **Expect**: a second Issued record at the next checkpoint
   (`m3-implementation-notes.md` §1.5's `checkPeriodicCheckInDue` checkpoint-list logic).
4. Submit both surveys. **Expect**: write-back fills all 7 segments correctly (4 reused
   alliance domains + 3 new Practice Experience/Professionalism fields) — this is the first
   real-data confirmation that the reused M2 Analytics formula columns parse M3 rows
   correctly (flagged as unverified in `m3-implementation-notes.md` §1.2).

### Step D — M4 (Discharge)
1. Turn ON "M4 - Discharge Trigger" and "M4 - Discharge Write-back".
2. Set `Patient_Status` to `Completed Treatment`. **Expect**: 1 Issued `4 - Discharge`
   record, 1 email.
3. Submit the survey. **Expect**: write-back fills Looking-Ahead fields + reused alliance
   domains (`m4-implementation-notes.md` §3) — first real-data confirmation for M4's reused
   columns (flagged unverified, T020 in `m4-tasks.md`).

### Step E — M5 (Discontinuation)
1. Turn ON "M5 - Discontinuation Trigger" and "M5 - Discontinuation Write-back".
2. Set `Patient_Status` to `Discontinued (Patient Choice)`. **Expect**: 1 Issued
   `5B - Discontinuation, Email Fallback` record, 1 email (subject "Just checking in").
3. Submit the survey (2 fields only — reason + okay-to-reach-back-out). **Expect**: write-back
   fills `Response_Data`, and — separately — confirm a **non-response** case (don't submit a
   second test instance) still shows up as "issued but not submitted" in "M5 Submitted
   Responses"/"M5 Status Breakdown" rather than silently vanishing (FR-015/SC-... requirement).
4. **Cutoff-exclusion check** for M5 too, same as Step B.4.

### Step F — Dashboard/access checks (Story 3/User Story 4, cuts across all of the above)
1. As admin: open each milestone's dashboard, confirm the new test data renders in every
   panel (not just "No Data Available" anymore).
2. Confirm the "M2, M3 & M4 Flagged for Review" report shows the flagged row(s) from Step B.6,
   with no Patient/identity column.
3. **Negative test, needs a contractor login**: confirm a contractor account (Deborah's or
   Shana's, if one exists) cannot reach any of these dashboards/reports at all (FR-009/SC-004).
   This session cannot do this step — needs Costin or a contractor to attempt it.

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
