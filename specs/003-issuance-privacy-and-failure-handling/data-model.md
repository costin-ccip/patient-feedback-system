# Data Model: Issuance Privacy and Send-Failure Handling

## Milestone_Instances changes

| Item | Before | After |
|---|---|---|
| `Name` on create | `Milestone <m> - <recipient email>` | `Milestone <m> - <first 8 chars of Token>` |
| `Status` picklist | -None-, Issued, Submitted, Expired, Superseded, Captured Live | + **Send Failed** |
| `Email` | set on create; not synced to Analytics | unchanged |

Status transitions added: `Issued → Send Failed` (email step fails, automatic);
`Send Failed → Issued` (admin manual resend).

## `issueFeedbackToken` change (one line)

```text
createMap.put("Name","Milestone " + milestone + " - " + recipientEmail);
```
becomes
```text
createMap.put("Name","Milestone " + milestone + " - " + token.subString(0,8));
```

## Existing-record rename

| id | New Name |
|---|---|
| 6825601000004191001 | `Milestone 0 - No Conversion - 91a38157` |
| 6825601000004245001 | `Milestone 1 - Baseline Intake - d196ca74` |
| 6825601000004284001 | `Milestone 2 - Early Alliance Check - 0045fe45` |

## Per-flow error branches (M0, M2, M3 identical except `<milestone>`/`<flow>`)

**A. On Error of "Send email" (patient email)**

1. Zoho CRM "Update module entry": connection CRM Connection; module Milestone
   Instances; record ID `${issueFeedbackToken_1.recordId}`; Status `Send Failed`;
   nothing else.
2. Zoho Mail "Send email": connection "Connection to info@capeclarity.com"; From
   `info@capeclarity.com`; To `costin@capeclarity.com`; Subject
   `Cape Clarity feedback pipeline: email send failed (<milestone>)`; Body:

```html
<div>A feedback request email could not be sent.<br></div><div><br></div><div>Flow: <flow><br></div><div>Milestone: <milestone><br></div><div>Milestone Instance record ID: ${issueFeedbackToken_1.recordId}<br></div><div><br></div><div>The record has been marked Send Failed. To resend, follow the runbook in specs/003-issuance-privacy-and-failure-handling/quickstart.md (section C).<br></div>
```

**B. On Error of `issueFeedbackToken`**

1. Zoho Mail "Send email": same connection/From/To; Subject
   `Cape Clarity feedback pipeline: token issuance failed (<milestone>)`; Body:

```html
<div>A feedback request could not be issued: the Milestone Instance record was not created, and no email was sent to the patient or prospect.<br></div><div><br></div><div>Flow: <flow><br></div><div>Milestone: <milestone><br></div><div><br></div><div>Check the flow's execution history in Zoho Flow for the error.<br></div>
```

| Flow | `<flow>` | `<milestone>` |
|---|---|---|
| M0 | M0 - Lost Lead Feedback Token | 0 - No Conversion |
| M2 | M2 - Session 3 Trigger | 2 - Early Alliance Check |
| M3 | M3 - Periodic Check-In Trigger | 3 - Periodic Consolidated |
