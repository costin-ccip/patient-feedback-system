# Implementation Plan: Issuance Privacy and Send-Failure Handling

**Branch**: `003-issuance-privacy-and-failure-handling` | **Date**: 2026-09-23 | **Spec**: [spec.md](./spec.md)

## Summary

Two changes on top of 002's issuance design:

1. **Privacy (1B)**: one-line change in the shared `issueFeedbackToken` function so
   the record name is `Milestone <m> - <token[0:8]>`; rename the 3 existing records
   whose names contain an email (via CRM MCP); confirm Email stays out of the
   Analytics sync (it already is).
2. **Failure handling (2B)**: add status value "Send Failed" to
   `Milestone_Instances.Status` (CRM Setup UI; the MCP has no field-update tool);
   in each of M0/M2/M3, hang two "On Error" branches:
   - on the Zoho Mail "Send email" step: CRM "Update module entry"
     (record `${issueFeedbackToken_1.recordId}`, Status = Send Failed) → Zoho Mail
     alert to `costin@capeclarity.com`;
   - on the `issueFeedbackToken` step: Zoho Mail alert only.

See research.md for decisions and alternatives.

## Technical Context

**Language/Version**: Deluge (one-line change); Zoho Flow native actions

**Primary Dependencies**: Zoho Flow Standard, Zoho CRM (`crm_connection` /
"CRM Connection"), Zoho Mail ("Connection to info@capeclarity.com"), Zoho Analytics
CRM sync (read-only check)

**Storage**: Milestone_Instances (new picklist value, no new fields)

**Testing**: structural verification (quickstart §A); Costin's live test incl. a
forced email failure (quickstart §B)

**Constraints**: no flow ON, no live tests, no Leads/Patients browsing; edits to the
shared function affect M0/M2/M3 at once (intended)

**Scale/Scope**: 1 function edit, 3 record renames, 1 picklist value, 3 flows × 2
error branches (6 alert steps, 3 CRM update steps)

**Task budget**: error branches run only on failure, so no extra Flow tasks on
normal runs.

## Constitution Check

| Principle | Result |
|---|---|
| I. De-identification | Pass. Forms untouched. |
| II. Single rejoin point | Pass, improved: Analytics copy no longer holds recipient emails in record names. Alerts carry record ID only; the rejoin still happens only by looking the record up in CRM. |
| III. Automated triggers | Pass. Failure handling is automatic; only the resend after a failure is manual (an ops recovery step, not a trigger decision). |
| IV. Contractor blindness | Pass. Alert goes to the practice owner/ops, never a contractor. |
| V. BAA-gated adoption | Pass. Same Zoho Mail connection and CRM connection; no new product or channel. |
| VI. Internal use only | Pass. |
| VII. Analytics computes derived values | Pass. "Send Failed" is a recorded event (what happened), not a derived score; it's written at the moment of failure because nothing else can observe it later. |

Result: PASS, no complexity tracking.

## Project Structure

```text
specs/003-issuance-privacy-and-failure-handling/
├── spec.md, plan.md, research.md, data-model.md, quickstart.md, tasks.md
├── checklists/requirements.md
└── implementation-notes.md   # written during build
```

Zoho objects:

```text
CRM  Milestone_Instances.Status          [EDIT] + "Send Failed"
CRM  3 records with email in Name         [EDIT] renamed
Flow issueFeedbackToken                  [EDIT] Name line only
Flow M0 / M2 / M3 trigger flows          [EDIT] + On Error branches on 2 steps each
Analytics CRM sync                       [CHECK] Email unselected (no change)
```

Docs updated in the same session: 002 contract + implementation notes (function
changed), 001 m0/m2/m3 notes (dated line pointing here), CLAUDE.md convention.

## Build order

1. Add "Send Failed" picklist value (CRM Setup → Modules and Fields → Milestone
   Instances → Status). Confirm via `getFields`.
2. Edit `issueFeedbackToken` (Name line); save (dialog will list the 3 flows;
   confirm, since changing all three is the point); re-read source.
3. Rename 3 existing records via `updateRecords`; re-query `Name like '%@%'` → 0.
4. M3, M2, M0: add the two error branches; wire explicitly and check
   `jsplumb-connected` on every endpoint (002 gotcha §5.1); read back values.
5. Docs, commit, push.
