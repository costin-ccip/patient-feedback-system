# M1 (Session 1 / Baseline Intake) — Implementation Notes

> Scope: this file documents ONLY the M1 milestone (the automatic Session 1 /
> Baseline Intake feedback loop). It is a companion to
> `specs/001-feedback-collection-pipeline/spec.md` (all six milestones) and to
> `m1-plan.md` / `m1-research.md` / `m1-data-model.md` / `m1-tasks.md`. This file
> tracks what has actually been built in Zoho, as it's built, per the CLAUDE.md
> convention of updating implementation notes in the same session as the change.
>
> Last updated: 2026-09-07
> Status: In progress. Phase 1 (Setup: T001 form, T002 schema confirm) and Phase 2
> (Foundational: T003–T005) are built and saved. Flow is still OFF/unpublished —
> no live test has been run yet, per Costin's explicit instruction to test M1 and
> re-verify M0 end-to-end only once all pieces are in place. T006/T007 (write-back
> flow) not yet built.

## 1. What M1 does

M1 fires automatically the first time a Patient's `Session_Count` reaches 1 (no
manual clinician action, unlike M0's manual trigger). It checks for an existing
`Milestone_Instances` record for that patient at Milestone "1 - Baseline Intake"
(idempotency — this has no M0 equivalent, since M0's manual trigger couldn't
double-fire the way a field-driven trigger can). If none exists, it calls the
shared `Subflow - Issue Feedback Token` (the same subflow M0 uses) to create the
`Milestone_Instances` record and email the patient a link to the Wellbeing
Check-In form.

De-identification: same model as M0, except M1 links to the patient via the
`Patient` lookup field on `Milestone_Instances` rather than a `Lead_Reference`
text field. (Earlier CLAUDE.md draft language forbade any automation from
populating/querying that lookup; Costin confirmed on 2026-09-07 that this lookup
points to a de-identified patient field, so the restriction was removed from
CLAUDE.md and does not apply to this build.

## 2. Component inventory

| Component | Type | Notes |
|---|---|---|
| Wellbeing Check-In Form | Zoho Form | T001. Public form, no patient identifier fields. Permalink: `https://forms.zohopublic.com/lianapreudhommecapec1/form/CapeClarityWellbeingCheckIn/formperma/Qlvh_FeoU3Oof7fQz4WEeQBoFvQHYl2TLdNlQPON_Eg` |
| M1 - Session 1 Trigger | Zoho Flow | New flow, "Customer Feedback System" folder. Currently OFF. |
| `checkBaselineIntakeExists` | Custom Function (Deluge), scoped to the trigger flow | Idempotency check, linear-scan pattern matching M0's `submitFeedbackResponse` token lookup (see M0 notes §3.2) for consistency, not because it's technically superior. |
| Subflow - Issue Feedback Token | Zoho Flow (shared with M0) | Unchanged this session except the parameterization already completed for M0 (survey_url/email_subject/email_intro_text as explicit params). M1 reuses it as-is; no subflow changes were needed for M1 support — `patient_id`, `recipient_email`, `milestone`, `clinician`, `ttl_days`, `lead_id` were already parameters from when the subflow was originally generalized. |

## 3. "M1 - Session 1 Trigger" flow, step by step

1. **Trigger** — Zoho CRM "Updated module entry". Connection: CRM Connection.
   Module: `Patients` (internal API name `Patients1`). Filter: `Session Count`
   `equals` `1`.
2. **Custom Function** — `checkBaselineIntakeExists(patientId)`, output variable
   `checkBaselineIntakeExists_1` (boolean). Input `patientId = ${trigger.id}`.
3. **If else** — condition: `checkBaselineIntakeExists_1` `is false`.
   - **True branch** (no existing Baseline Intake record — proceed): "Call a
     subflow" → `Subflow - Issue Feedback Token`, "Wait and continue" mode. See
     §4 for parameter values.
   - **False branch** (record already exists): left empty — flow simply ends,
     no email, no new record. This is the idempotency short-circuit (T004/FR-002).

### 3.1 `checkBaselineIntakeExists` — verbatim Deluge source

```
bool checkBaselineIntakeExists(string patientId)
{
	found = false;
	qmap = Map();
	allRecs = zoho.crm.getRecords("Milestone_Instances",1,200,qmap,"crm_connection");
	for each  r in allRecs
	{
		patientLookup = r.get("Patient");
		patientRecId = "";
		if(patientLookup != null)
		{
			patientRecId = patientLookup.get("id");
		}
		if(patientRecId == patientId && r.get("Milestone") == "1 - Baseline Intake")
		{
			found = true;
		}
	}
	return found;
}
```

Input: `patientId` (string). Return: `bool`. Connection name `crm_connection`
reused verbatim from M0's pattern.

## 4. "Call a subflow" parameters (M1 → Subflow - Issue Feedback Token)

| Parameter | Value | Notes |
|---|---|---|
| `clinician` | `Liana Preudhomme` | Reused from M0's value — only non-`-None-` option on the `Clinician` picklist. |
| `survey_url` | Wellbeing Check-In form permalink (§2) | |
| `email_intro_text` | "Thanks for starting your sessions with us. We'd love to hear how your first session went. Please take a moment to share your feedback using the link below:" | M1-specific; M0's equivalent was "We'd love to hear how things are going...". Drafted, not yet reviewed by Costin. |
| `milestone` | `1 - Baseline Intake` | Must match the CRM `Milestone` picklist value exactly (same pattern as M0's `0 - No Conversion`). |
| `patient_id` | `${trigger.id}` | Maps to the subflow's `Patient` lookup field via `${cf_trigger.patient_id}` in its "Create module entry" step. |
| `recipient_email` | `${trigger.Email}` | Patients module has an `Email` field (api_name `Email`), confirmed via `getFields` (schema only, no record data pulled) — same api_name as Leads, so the M0 pattern of `${trigger.Email}` carries over unchanged. |
| `ttl_days` | `7` | Reused M0's value verbatim, per the pending-tasks note "reuse M0's value unless Costin specifies otherwise." Not yet explicitly confirmed by Costin for M1. |
| `email_subject` | "How was your first session? Quick check-in from Cape Clarity" | M1-specific; drafted, not yet reviewed by Costin. |
| `lead_id` | *(empty)* | M1 is patient-based, not lead-based — subflow's `Lead Reference` field is left unpopulated for M1-originated records. |

## 5. Gotchas discovered building M1 (new, beyond what M0 notes already cover)

- **Zoho Flow "Decision" branching block cannot reference custom function
  outputs in this account.** Its condition editor's field/value selectors and
  "Insert Variable" panel only ever showed System Variables (current
  date/datetime) — confirmed by typing the exact variable name, searching,
  reloading, and checking all three condition boxes. Typing `${...}` directly
  into the field is treated as a search query, not accepted as literal input.
  **Fix: use an "If else" logic block instead** — its Insert Variable panel
  correctly surfaces custom function outputs under a "Custom Functions"
  section (e.g. `checkBaselineIntakeExists_1`, typed `boolean`, clickable to
  insert). For a boolean output specifically, "If else"'s operator dropdown
  offers clean "Is true" / "Is false" options once the variable is inserted —
  no need to type a literal comparison value.
- **Canvas drag-and-drop onto an existing connector can replace a node
  instead of inserting alongside it.** Dropping a new "Set Variable" step
  directly onto the connector between `checkBaselineIntakeExists` and the
  (at-the-time) Decision node deleted `checkBaselineIntakeExists` from the
  chain. Recovered via the canvas undo button. Dropping onto an empty output
  circle (rather than a connector between two already-connected nodes) did
  not have this problem.
- **Custom function editor bracket/autocomplete corruption**, same family as
  the M0-documented pipe-character bug: typing the full `checkBaselineIntakeExists`
  body in one `type` action left 2 stray closing `}` and inserted a literal
  `<opr> <expression>` autocomplete placeholder after `return found;`. Fixed by
  deleting the excess and retyping the final line in two pieces (`return found`
  without the semicolon, then `;` separately) to avoid re-triggering the
  snippet. Verify brace balance visually via a zoomed screenshot before moving on.

## 6. Access constraint change (2026-09-07)

CLAUDE.md previously stated: "Never populate or query the `Patient` lookup
field on `Milestone_Instances` ... from any automation, report, or session."
This was written before M1 planning matured. When it blocked M1's core design
(the idempotency check queries `Patient`; T005 populates it), I stopped and
flagged the conflict to Costin rather than building around it. Costin
confirmed the lookup points to a de-identified patient field and had the
line removed from CLAUDE.md. No other access constraints changed — the
Leads/Patients browser-access restriction and the "no live tests without
Costin's involvement" rule both still stand.

## 7. Test/sample data approach

Not yet applicable — no live test has been run for M1. Per Costin's explicit
instruction, M1 testing (and M0 re-verification) will happen end-to-end once
T006/T007 (write-back flow) are also built, not incrementally per task.

## 8. Remaining work (not yet built)

- T006: "M1 - Wellbeing Check-In Write-back" flow — realtime Form-submission
  trigger on the Wellbeing Check-In form → write-back custom function.
- T007: write-back function — token lookup, `Status != "Issued"` rejection
  (reuse M0's `submitFeedbackResponse` pattern, see M0 notes §3.1), expiry
  check + auto-expire, delimited `Response_Data` write-back per
  `m1-data-model.md` (`---`-joined, 5 domains, total NOT stored in the blob),
  `Status -> "Submitted"`.
- Phase 3+ (US1/US2 verification tasks, T008–T014) depend on T006/T007 and on
  the deferred end-to-end live test.
