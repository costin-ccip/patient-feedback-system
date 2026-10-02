# Quickstart: Feedback Email Opt-Out

**Feature**: `006-feedback-email-opt-out` | **Date**: 2026-10-01

How to check the feature, from safest to most live. Each part names what it proves.

**Test records**: patient PT000, `Patients1` ID `6825601000004448028` (designated by Costin,
see `CLAUDE.md`), and the existing test lead "Patient Feedback System - Test Lead"
(`Leads` ID `6825601000004448011`). Use only these. Parts A and B can be run by Claude
once the CRM fields exist and Costin approves the writes; Part C is run by Costin or
explicitly approved by him, as with every live test in this project.

## A. Function checks (Execute in the Flow function editor; no flow involved)

Prereq: the new Status value exists (task T003). The native field needs no setup.

1. `skipIfOptedOut("2 - Early Alliance Check", "<PT000 id>", "", "Liana Preudhomme")` with
   PT000's `Email Opt Out` unchecked. **Expect** `false`, no row written.
2. Tick `Email Opt Out` on PT000 and run it again. **Expect** `true`, and exactly one new
   `Milestone_Instances` row: Milestone `2 - Early Alliance Check`, Status `Skipped - Opted Out`,
   Patient PT000, no token, no email.
3. Run it a second time. **Expect** `true` and still exactly one such row (de-duplication).
4. Run it with milestone `3 - Periodic Consolidated`. **Expect** `true` and a new row each
   time it is run (M3 counts every skip).
5. Run it with both IDs empty. **Expect** the function throws; nothing is written.
6. `recordFeedbackOptOut` with a token taken from an existing test `Milestone_Instances` row
   for PT000, with PT000 currently unchecked. **Expect** `recorded`;
   PT000 now shows Email Opt Out checked with today's Unsubscribed Time, a CRM note about the email
   link, and a timeline entry with source `crm_api`; the
   Milestone_Instances row it matched is unchanged.
7. Run it again. **Expect** `already`; no field changed.
8. Run it with an empty string, and with `not-a-real-token`. **Expect** `no_match`; nothing
   written anywhere.
9. Repeat 6 to 8 with a token whose row is `Expired` and one that is `Submitted`.
   **Expect** the same results (FR-006).
10. Clear PT000's flag. **Expect** Unsubscribed Time and Mode are emptied; the timeline shows
    both the opt-out and the resubscribe with their times.

**Proves**: US1 scenarios 4 to 6 (idempotent, any token status, unknown token), US2 scenario 3
(skip rows persist), US4 scenario 2 (history discoverable).

## B. Opt-out path with the milestone flows still OFF

1. In the new Forms page's preview, confirm it shows only the explanation, one hidden
   `Token` field (not visible), and the "Stop feedback emails" button; nothing asks for a
   name, email or phone. **Proves** FR-003, FR-004.
2. Open the form link with a real test token in the browser and then close it without
   pressing the button. **Expect** no change on PT000 and no flow run. Also fetch the link
   with `curl` (a stand-in for a link scanner). **Expect** no change. **Proves** FR-002, SC-003.
3. Press the button. **Expect** the thank-you message, the flow runs once, PT000 shows the
   opt-out with a timeline entry from `crm_api`. **Proves** US1 scenario 3.
4. Submit the form with a made-up token. **Expect** the thank-you message, no CRM change,
   and an alert email to Costin containing no token or identity. **Proves** FR-007 (with D3).
5. Confirm the Forms entry, the flow's task history and the alert contain only a token, never
   a name or email address. **Proves** SC-005.

## C. Live scenario (Costin, as part of the coordinated live test)

Add these to `specs/001-feedback-collection-pipeline/coordinated-live-test-plan.md` when
the time comes. Each uses PT000 or the test lead.

| # | Do | Expect |
|---|---|---|
| 1 | Trigger M2 for PT000 normally (not opted out). | Email arrives with the survey link and the opt-out link; both carry the same token. (SC-001) |
| 2 | Use that email's opt-out link; confirm. | PT000 opted out (timeline source `crm_api`, link note present). |
| 3 | Re-trigger M2 for PT000. | No token issued, no email; one `Skipped - Opted Out` row for M2. |
| 4 | Drive M4 for PT000, then M5. | Each: no email, one skip row. |
| 5 | Drive M3 to the next checkpoint. | No email; one M3 skip row. Drive the next checkpoint: another skip row. |
| 6 | Move the test lead to Lost Lead with an opt-out set; update the lead twice more. | No email; exactly one M0 skip row. |
| 7 | Clear PT000's opt-out; trigger the next M3 checkpoint. | The next checkpoint after the skipped ones sends normally; skipped checkpoints do not send. (SC-007) |
| 8 | Staff-record an opt-out on the test lead by hand (no link). | The CRM stamps Unsubscribed Time; the timeline source is `crm_ui`; M0 for that lead is suppressed. (US3) |
| 9 | Opt PT000 out, then raise PT000's session count by a large jump past several checkpoints in one update, then resubscribe and update again. | Confirm what happens against research.md §3 (one skip per update; later checkpoints may still send one per update after a resubscribe) and record the actual behavior. |
| 10 | Open the Analytics Status Breakdown reports after the next sync. | `Skipped - Opted Out` appears as its own count in each milestone's report and is not counted as a non-response. (SC-007) |
| 11 | Only if a contractor ever gets CRM access: preview as that profile. | The Email Opt Out field is not visible on Leads or Patients. (SC-008; not needed while contractors have no CRM access) |

SC-002 wording for testing: after an opt-out has been recorded, no milestone that fires
afterward sends. An email already in flight in the few seconds around the opt-out may still
arrive (plan.md, Risks).

## D. Things to confirm are untouched

- `issueFeedbackToken` source unchanged; every idempotency check function unchanged.
- Native `Email_Opt_Out`, `Unsubscribed_Time` and `Unsubscribed_Mode` are not added to the Analytics CRM sync; the Leads table in Analytics still has no "@"
  anywhere (feature 003, T013 check).
- The Campaigns audience mapping does not copy `Email_Opt_Out` from a Patient into Email Audience (Campaigns is used with Leads only).
