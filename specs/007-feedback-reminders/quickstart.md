# Quickstart: Feedback Reminders

Structural checks first (A, B), then one live run folded into the coordinated test (C).
Live runs always need Costin's go-ahead. Use only test patient PT000 (`Patients1`
`6825601000004448028`) and the existing test lead. Seed `Milestone_Instances` rows with the
CRM tools or by hand (writes may need Costin's approval; auto-mode sessions cannot write to
CRM, see `CLAUDE.md`). Seed rows in age order, because `Created_Time` cannot be backdated:
control "4 days old" through `Expiry_Date_Time` instead.

## A. Function checks (Execute in the function editor, no email involved)

Run `claimNextDueReminder` against seeded test rows. Reset `Reminder_Sent_Date_Time` to empty
between runs. Expected outcomes:

| # | Seeded row (all for PT000 unless stated) | Expected |
|---|---|---|
| 1 | Issued, expiry in 2 days, no stamp, M2 | `found: true`, row stamped |
| 2 | Same row, run again | `found: false` (already stamped) |
| 3 | Issued, expiry in 5 days | `found: false` (not yet day 4) |
| 4 | Issued, expiry in the past | `found: false` |
| 5 | Status `Submitted`, expiry in 2 days | `found: false` |
| 6 | Status `Superseded` | `found: false` |
| 7 | Eligible row, PT000 opted out (`Email_Opt_Out` true) | `found: false`, no stamp |
| 8 | Eligible M5 row, PT000 status not "Discontinued (Patient Choice)" | `found: false` |
| 9 | Eligible M3 row, previous PT000 request `Expired` | `found: false` (brake) |
| 10 | Eligible M3 row, previous PT000 request `Submitted` | `found: true` |
| 11 | Eligible M3 row, previous request still `Issued` and unexpired | `found: true` (open, not unanswered) |
| 12 | Two eligible rows | the nearer expiry is returned first |
| 13 | Run outside 9 to 17 Eastern | `found: false` |
| 14 | Row with no `Patient` and no `Lead_Reference` | skipped; function throws if nothing else was sendable |

Also record: the function runs without error under `crm_connection`, and the COQL query returns
`Created_Time` and `Expiry_Date_Time` in a form `toDateTime()` accepts.

## B. Flow checks (flow Test, flow still OFF, no live email)

1. Test the function step with a seeded eligible row; confirm `claimNextDueReminder_1.found`.
2. Confirm each milestone value routes to its own Send email step (use flow Test with each
   milestone's seeded row).
3. Point one Send email step at a test mailbox only with Costin's go-ahead; confirm the
   survey link and the stop-emails link both carry the token and nothing else, and the body
   has no name or clinical content.
4. Force each On Error branch; confirm the alert has no email address and no token.

## C. Live run (Costin runs or approves)

1. Switch the flow ON for the test. With one eligible PT000 row, wait for the next in-window
   run (or trigger it); expect one reminder and a stamped row.
2. Open the reminder's survey link and submit; confirm the response rejoins to the right
   row in CRM exactly as from the original email, and the token is spent.
3. Click the stop-emails link from a reminder; confirm PT000 shows opted out and no further
   reminder is claimed. Re-subscribe PT000 afterwards (Costin's usual state for PT000 per
   006 T002 is opted out).
4. After the next 5:00 PM EDT Analytics sync, confirm `Reminded` and `Response Outcome` and
   the Reminder Effectiveness report show the test rows, and that the opt-out skip row is in
   `Excluded`.
5. Switch the flow OFF again unless Costin says otherwise.
