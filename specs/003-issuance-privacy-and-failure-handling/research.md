# Research: Issuance Privacy and Send-Failure Handling

**Date**: 2026-09-23

## 1. Where the recipient email lives today

- `issueFeedbackToken` writes it to `Name` (`Milestone <m> - <email>`) and `Email`
  on each Milestone_Instances record (same as the retired subflow did).
- **Analytics sync setup** (Data Sources → Zoho CRM → Edit Setup, viewed and
  cancelled without saving): Milestone Instances fields selected = 13. **Email and
  Secondary Email are not selected.** "Milestone Instance Name" is selected and
  greyed out (required by the connector; can't be deselected).
- **So**: the only path for the email into Analytics is the record name. Fix = change
  the name format + rename existing records. No Analytics setup change is needed.
- Existing affected records (COQL `Name like '%@%'`, selecting only id/Milestone/
  Status/Token): 3 — `6825601000004191001` (0, Issued), `6825601000004245001`
  (1, Submitted), `6825601000004284001` (2, Issued).

**Decision**: name = `"Milestone " + milestone + " - " + token.subString(0,8)`.
Token prefix makes records distinguishable in CRM lists without identity.
**Alternatives**: record ID (not known until after create); timestamp (collides,
and adds nothing); leaving Name as milestone only (hard to tell records apart).

## 2. "Send Failed" status

- `Status` picklist today: `-None-, Issued, Submitted, Expired, Superseded,
  Captured Live`. The CRM MCP has `createFields` but no field-update tool, so the
  value is added in CRM Setup (Modules and Fields → Milestone Instances → layout →
  Status → edit values). This is a module-settings page, not the Leads/Patients
  modules, so it's within the access rule.
- Downstream effects, checked:
  - Write-back functions reject any status other than Issued → a Send Failed token
    can't be submitted. Good (the patient never received it anyway).
  - `supersedeOpenInstances` logic inside `issueFeedbackToken` only touches Issued
    records → Send Failed records are left alone. Good.
  - M2 `checkAllianceCheckExists` and M3 `checkPeriodicCheckInDue` count records at
    that milestone regardless of status → a Send Failed record blocks automatic
    re-issue. Accepted: resend is manual (below). Changing those checks would widen
    the scope into M2/M3 logic for a rare case.
  - Analytics "Status Breakdown" reports group by Status → "Send Failed" appears as
    its own bar automatically after sync. No report change needed.

**Manual resend runbook** (quickstart §C): fix the patient's email in CRM, open the
failed Milestone Instance, email the survey link with `?token=<Token>` from
info@capeclarity.com, set Status back to Issued and push Expiry out 7 days.

## 3. Failure paths in Zoho Flow

- Every action node has an **On Error** endpoint (`b.error.zf-action-error`, the red
  circle on the node's right). An action dropped there runs only if that step fails.
- **Email step failure** → CRM "Update module entry" (module Milestone Instances,
  record ID `${issueFeedbackToken_1.recordId}`, Status `Send Failed`) → Zoho Mail
  alert. Mark first, so the record is visible even if the alert also fails (same
  mail connection).
- **Issuance step failure** (the function's `throw`, or any CRM error) → Zoho Mail
  alert only. No record to mark, and the email step never runs.
- **Alert content** (de-identified): subject `Cape Clarity feedback pipeline: <what>
  failed (<milestone>)`; body names the flow, milestone, record ID (email failure
  only) and what to do. No `${trigger.*}` identity fields.
- **Alternatives rejected**: a Deluge `sendmail` alert (new sending channel,
  Principle V); a Flow-level "notify on failure" setting (not per step, can't mark
  the record); a separate "Pending until sent" status (option 2C, not chosen).
