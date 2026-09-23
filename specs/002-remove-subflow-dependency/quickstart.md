# Quickstart / Validation: Remove Subflow Dependency

## A. Structural verification (done by the building session, no live data)

For each of M0, M2, M3 trigger flows:

1. Flow is still **OFF**.
2. No "Call a subflow" node on the canvas.
3. Node chain is: M0 `trigger → issueFeedbackToken → Send email`; M2/M3
   `trigger → check function → If-else (true) → issueFeedbackToken → Send email`.
   Wiring confirmed with the DOM connector check
   (`document.querySelectorAll('[class*="connector" i]').length`) plus screenshot.
4. `issueFeedbackToken` node parameters match data-model.md's per-flow table
   (read back from the panel's input values, not screenshots alone).
5. Send email node: connection, From, To, Subject, and HTML body match data-model.md
   byte for byte (read back via input values).
6. `issueFeedbackToken` source, re-read from the live editor, matches data-model.md
   (or the recorded deviation), and saving it lists exactly the intended flows.

Workspace-level:

7. `Subflow - Issue Feedback Token` is OFF, renamed `[RETIRED] Subflow - Issue
   Feedback Token`, and still present.
8. Write-back flows, forms, Analytics objects untouched (no edits made; spot-check
   "Last saved" isn't a reliable signal, see research.md §5.3).

## B. Live test (Costin runs this; not an automated session)

Prereqs: a test Lead and a test Patient Costin designates; flows switched ON by Costin.

1. **M0**: set the test Lead's Lead Status to Lost Lead. Expect: 1 new
   Milestone_Instances record (`0 - No Conversion`, Issued, Lead_Reference = lead ID,
   token + expiry set), 1 email from info@capeclarity.com with the M0 subject, link
   ends `?token=<same token>`. Repeat once: first record → Superseded, second Issued.
2. **M2**: set test Patient's Session Count to 3. Expect 1 Issued `2 - Early Alliance
   Check` record with Patient set, 1 email. Re-save at 3: no second record (M2's own
   idempotency check).
3. **M3**: Session Count 8, then 16. Expect one Issued `3 - Periodic Consolidated`
   record at each checkpoint, one email each.
4. Submit each survey via its emailed link. Expect the unchanged write-back flow to
   mark the record Submitted with Response_Data filled.
5. (Optional, failure path) Temporarily pass an invalid `milestone` value in a test
   copy: expect the function step to fail and no email.
6. Clean up test Milestone_Instances records via CRM MCP tools, as in 001.
