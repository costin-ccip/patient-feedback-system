# Quickstart / Validation: Issuance Privacy and Send-Failure Handling

## A. Structural checks (building session)

1. `getFields` on Milestone_Instances shows `Send Failed` in Status.
2. `issueFeedbackToken` source re-read: Name line uses `token.subString(0,8)`.
3. COQL `select id from Milestone_Instances where Name like '%@%'` returns 0 rows.
4. Analytics CRM sync setup: Milestone Instances → Email unchecked (view only).
5. Each of M0/M2/M3 (still OFF): Send email step's On Error endpoint is connected
   (`jsplumb-connected`) to "Update module entry" → alert "Send email";
   issueFeedbackToken's On Error endpoint is connected to an alert "Send email".
   Values read back after reload match data-model.md.

## B. Live test additions (Costin)

On top of 002's quickstart §B:

1. Happy path: new record's Name is `Milestone <m> - <8 hex chars>`.
2. Forced email failure: issue to a test record whose Email is invalid (e.g.
   `not-an-address`). Expect: record Status = Send Failed; alert at
   costin@capeclarity.com with milestone + record ID only; no patient email sent.
3. Check the flow execution shows the On Error branch ran.
4. Next Analytics sync (daily, 17:00): no email addresses in "Milestone Instance
   Name"; "Send Failed" appears in the Status Breakdown chart.

## C. Runbook: resending after "Send Failed"

1. From the alert, open the Milestone Instance record ID in CRM (Milestone Instances
   module).
2. Fix the recipient's email on the Patient/Lead record if it was wrong.
3. Copy the record's Token. Email the patient from info@capeclarity.com using the
   milestone's usual subject/intro text and the survey link with `?token=<Token>`
   (links are in `specs/002-remove-subflow-dependency/data-model.md`).
4. On the Milestone Instance: set Status to **Issued** and Expiry Date Time to 7
   days from now. (While it's Send Failed the survey link is rejected.)
