---
description: "Basic manual test scenario for M1 (Baseline Intake / Wellbeing Check-In) — verifies information actually flows end-to-end before Costin runs the full coordinated live test"
---

# M1 Test Scenario: Does the information flow?

> **RETIRED (2026-09-11)**: Milestone 1 has been eliminated per an operations-lead
> decision following a review of clinical/EHR overlap — see `spec.md`'s fifth-pass
> revision note and Assumptions. This file is kept as the historical record of the
> test scenario used, not deleted; it no longer reflects the live system. The
> corresponding Zoho objects are being decommissioned — see `m1-implementation-notes.md`'s
> own RETIRED note for their disposition.

**Purpose**: a minimal, low-risk check that a Wellbeing Check-In response
actually travels Form → write-back function → `Milestone_Instances` fields →
Analytics formulas correctly — i.e., the "information flow" question — without
touching the automatic `Session_Count` trigger or the Patients module. This
maps to `m1-tasks.md` Phase 6 (T017–T020) but narrows it to the piece that can
be tested safely without Costin's direct involvement in the moment.

**Not covered here**: the automatic trigger (US1, T008–T010) — that requires
changing a real Patient's `Session_Count`, which needs Costin or someone with
Patients-module access, per the standing access restriction. See "Optional
full-trigger test" at the end for that piece.

**Before running this**: both `m1-implementation-notes.md` §7 and the standing
project rule say live M1 testing happens once Costin is ready to coordinate it
— this file is the written plan to have ready for that session, not a signal
to run it unilaterally. Both M1 flows ("M1 - Session 1 Trigger" and "M1 -
Wellbeing Check-In Write-back") are currently OFF; step 1 turns on only the
one this scenario needs.

## Prerequisites

- [ ] Confirm with Costin this is an OK time to test (write-back flow needs to
      be ON, and it will touch the live public form + CRM).
- [ ] Turn ON the **"M1 - Wellbeing Check-In Write-back"** flow in Zoho Flow.
      (Leave "M1 - Session 1 Trigger" OFF — not needed for this scenario.)

## Step-by-step

### 1. Create a test `Milestone_Instances` record

Via Zoho CRM MCP tools (`createRecords`) — no browser, no PII, consistent with
the M0 test-data pattern (`m0-implementation-notes.md` §8):

| Field | Value |
|---|---|
| `Milestone` | `1 - Baseline Intake` |
| `Status` | `Issued` |
| `Token` | a unique test value, e.g. `TEST-M1-20260909-01` |
| `Expiry_Date_Time` | a few days in the future |
| `Patient` | leave blank — not required for this test; Analytics never reads it anyway (§9.1) |
| `Response_Data`, `Submitted_Date_Time` | leave blank — these should get filled by the write-back function, not set by hand |

### 2. Submit the public form with that token

Open (or have Costin open) the Wellbeing Check-In form permalink with the test
token appended, e.g.:

```
https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityWellbeingCheckIn/formperma/Qlvh_FeoU3Oof7fQz4WEeQBoFvQHYl2TLdNlQPON_Eg?token=TEST-M1-20260909-01
```

Use known, easy-to-verify slider values, e.g.:

| Domain | Value |
|---|---|
| Personal wellbeing | 7 |
| Coping | 6 |
| Relationships and support | 8 |
| Hope and outlook | 5 |
| Sense of control | 4 |

(Expected total: 7+6+8+5+4 = **30** — pick your own numbers if you like, just
know the sum going in so step 4 is a real check, not a rubber stamp.)

### 3. Verify the write-back landed correctly

Via CRM MCP (`getRecords` on `Milestone_Instances`, filtered to the test
token):

- [ ] `Status` is now `Submitted`
- [ ] `Submitted_Date_Time` is populated
- [ ] `Response_Data` reads exactly:
      `Personal wellbeing (0-10): 7---Coping (0-10): 6---Relationships and support (0-10): 8---Hope and outlook (0-10): 5---Sense of control (0-10): 4`

### 4. Verify Analytics parses it correctly

Open the "Milestone Instances" table or the "M1 Submitted Responses" report in
Zoho Analytics and find the test row:

- [ ] Domain: Wellbeing = 7, Domain: Coping = 6, Domain: Relationships = 8,
      Domain: Hope = 5, Domain: Control = 4
- [ ] Wellbeing Check-In Total = 30 (matches your manual sum from step 2)
- [ ] The row appears in "M1 Submitted Responses" (confirms the report's
      `Milestone = "1 - Baseline Intake"` filter is working)
- [ ] No `Patient`/identity column appears anywhere in the report (confirms
      de-identification held)

### 5. Verify reuse rejection (Status != "Issued" check)

Submit the *same* form URL/token a second time, then check the write-back
flow's run history in Zoho Flow (not the CRM record, which shouldn't change):

- [ ] The second run returns an error result — message along the lines of
      "Milestone Instance is not in Issued status (current: Submitted)"
- [ ] The CRM record's `Response_Data` and `Submitted_Date_Time` are
      unchanged from step 3 (the second submission didn't overwrite anything)

### 6. Clean up

- [ ] Delete the test `Milestone_Instances` record via CRM MCP
      (`deleteRecords`), same as M0's cleanup pattern.
- [ ] Decide with Costin whether to leave "M1 - Wellbeing Check-In Write-back"
      ON (i.e., this doubles as going live) or turn it back OFF.

## Optional: full trigger test (needs Costin, not part of the basic scenario)

To also exercise the automatic side (US1, T008–T010) — the part this basic
scenario skips:

1. Get a test Patient record ID from Costin (T017 — do not look one up via
   CRM query or browser, per the standing restriction).
2. Turn ON "M1 - Session 1 Trigger".
3. Have Costin (or someone with Patients-module access) set/confirm that
   Patient's `Session_Count` reaches `1`.
4. Confirm exactly one `Milestone_Instances` record is created and exactly
   one email is sent — no duplicates.
5. Trigger a second `Session_Count`-reaches-1 event for the same patient (if
   possible to simulate) and confirm no second record/email is created
   (T004/T009 idempotency).
6. Clean up the same way as step 6 above.
