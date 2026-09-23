# Contract: `issueFeedbackToken` (Zoho Flow custom function, workspace-shared)

The one interface this feature introduces. Every milestone trigger flow (M0, M2, M3
now; M4, M5 when built) calls it in place of "Call a subflow → Subflow - Issue
Feedback Token".

## Inputs

| Name | Type | Required | Notes |
|---|---|---|---|
| `milestone` | string | yes | Must exactly match a `Milestone_Instances.Milestone` picklist value |
| `patientId` | string | one of patientId / leadId | Patients1 record ID; empty for M0 |
| `leadId` | string | one of patientId / leadId | Leads record ID; empty for patient milestones |
| `recipientEmail` | string | yes | `${trigger.Email}` |
| `clinician` | string | yes | Must match a `Clinician` picklist value |
| `ttlDays` | int | yes | Token lifetime; 7 for all current milestones |

## Output (map, variable `issueFeedbackToken_<n>`)

| Key | Meaning |
|---|---|
| `status` | `"success"` (only value ever returned; failures throw) |
| `token` | The issued token; the only value the email step needs |
| `expiry` | Expiry timestamp as written to CRM |
| `recordId` | ID of the created Milestone_Instances record |
| `superseded` | How many older Issued records were marked Superseded |

## Failure behavior

If the CRM create returns no record ID, the function throws, the step fails, and the
flow stops before the email step. (Fallback if `throw` can't be used: the caller adds
an If-else on `status == "success"` before the email step; see research.md §3.)

## Side effects

1. Updates matching `Issued` Milestone_Instances (same person + milestone) to
   `Superseded`.
2. Creates exactly one Milestone_Instances record (shape in data-model.md), firing CRM
   workflow/approval/blueprint triggers.

It never sends email and never reads or writes any module other than
Milestone_Instances.

## Change rule

This function is shared by every trigger flow. Any edit affects all of them at once
(Flow will list them on save). That is the intended behavior; treat an edit here like
editing the old subflow, and update this contract and the 002 implementation notes in
the same session.
