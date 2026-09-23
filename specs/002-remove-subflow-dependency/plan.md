# Implementation Plan: Remove Subflow Dependency from Token Issuance

**Branch**: `002-remove-subflow-dependency` | **Date**: 2026-09-23 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/002-remove-subflow-dependency/spec.md`

## Summary

Every milestone trigger flow currently ends with "Call a subflow → Subflow - Issue
Feedback Token", and subflows aren't available on the practice's long-term Zoho Flow
plan (Standard). Replace the subflow with one new shared custom function,
`issueFeedbackToken`, that performs the subflow's supersede + token + CRM-create steps
in a single call, followed by a native Zoho Mail "Send email" step in each trigger
flow. Same record, same email, same sender, one shared definition. Applies to M0, M2,
M3 (all currently OFF); M4/M5 adopt the pattern when built. See research.md §2 for the
alternatives rejected.

## Technical Context

**Language/Version**: Deluge (Zoho Flow custom functions)

**Primary Dependencies**: Zoho Flow (Standard plan), Zoho CRM (Standard edition,
`crm_connection`), Zoho Mail (connection to info@capeclarity.com)

**Storage**: Zoho CRM `Milestone_Instances` (unchanged schema)

**Testing**: Structural verification in the Flow builder (quickstart.md §A); live
test by Costin (quickstart.md §B)

**Target Platform**: Zoho Flow workspace `872426000000002011`, folder "Customer
Feedback System"

**Project Type**: No-code/low-code workflow configuration + one Deluge function

**Performance Goals**: N/A (a handful of issuances per week). Task usage: the new
path uses fewer Flow tasks per issuance than the subflow (2 actions vs subflow call +
4 actions), which matters on a 5,000-task/month plan.

**Constraints**: No subflows; no flow switched ON; no live tests; no Leads/Patients
browsing; M0/M2/M3 flows edited in place (not cloned, per 001 m2 notes §3.2)

**Scale/Scope**: 1 new function, 3 flows edited, 1 flow retired, 5 docs updated

## Constitution Check

*Gate: checked before research and re-checked after design. Result: PASS.*

| Principle | Assessment |
|---|---|
| I. De-identification by design | Pass. Forms still receive only the token in the URL. Nothing new reaches the collection tool. |
| II. Single rejoin point | Pass. Token + identity are still joined only in CRM (same record). Flow already carried recipient email and patient/lead IDs through the subflow; it now carries them through a function instead. No new system holds both. (Pre-existing observation: the recipient email lands in the record Name and Email fields, which sync to Analytics. Not introduced or widened here; flagged in research.md §5.1.) |
| III. Automated milestone triggers | Pass. Triggers unchanged. |
| IV. Contractor blindness | Pass. No contractor-facing step anywhere; email goes only to the patient/prospect. |
| V. BAA-gated adoption | Pass. No new product or channel: same Flow, CRM, and Mail connections. Deluge `sendmail` deliberately not used (research.md §2). |
| VI. Internal use only | Pass. No public/marketing connection. |
| VII. Analytics computes derived values | Pass. No derived score/flag involved; issuance plumbing only. |
| Development Workflow: changes to token generation must be checked against I-IV | Done above. Token algorithm copied verbatim, not changed. |
| Rollout Workflow (specify → plan → tasks) | Followed. |

Post-design re-check: PASS, no violations, Complexity Tracking not needed.

## Project Structure

### Documentation (this feature)

```text
specs/002-remove-subflow-dependency/
├── spec.md
├── plan.md                         # this file
├── research.md                     # subflow internals captured + decisions
├── data-model.md                   # record shape, function draft, per-flow values
├── contracts/issue-feedback-token.md
├── quickstart.md                   # structural checks + Costin's live test
├── checklists/requirements.md
├── tasks.md                        # /speckit-tasks output
└── implementation-notes.md         # as-built, written during implementation
```

### Zoho objects touched

```text
Zoho Flow workspace 872426000000002011 / Customer Feedback System
├── [NEW]      custom function issueFeedbackToken (workspace-shared)
├── [EDIT]     M0 - Lost Lead Feedback Token      (Call a subflow → function + Send email)
├── [EDIT]     M2 - Session 3 Trigger             (same, on If-else true branch)
├── [EDIT]     M3 - Periodic Check-In Trigger     (same, on If-else true branch)
├── [RETIRE]   Subflow - Issue Feedback Token     (rename [RETIRED], stays OFF, not deleted)
├── [UNTOUCHED] supersedeOpenInstances, generateFeedbackToken (still referenced by the retired subflow)
├── [UNTOUCHED] [RETIRED] M1 flows, all write-back flows, [Not used] New Payment
Zoho CRM / Forms / Analytics: no changes
```

Repo docs updated in the same session (CLAUDE.md convention): this feature's
`implementation-notes.md`; 001's `m0-`, `m2-`, `m3-implementation-notes.md`
(issuance wiring sections); `CLAUDE.md` (new cross-milestone convention: no
subflows, use `issueFeedbackToken` + per-flow Send email).

**Structure Decision**: A separate feature directory (002), because this is a
cross-milestone infrastructure change rather than a milestone; per-milestone as-built
changes still land in each milestone's own notes file in 001.

## Build order and risk notes

1. Create `issueFeedbackToken` first, standalone, via Built-ins → Custom Functions →
   "+ Custom Function" (never by cloning a node). Save; confirm `throw` and the
   createRecord options form are accepted; re-read source.
2. M3 first (most recently built, notes freshest), then M2, then M0. For each:
   re-read the live "Call a subflow" values into notes before deleting that node; drop
   the function node on the True-branch (or trigger) connector with the stepped
   synthetic drag (001 m3 notes §1.5); map params with the Insert-variable / native
   setter techniques; drop Zoho Mail "Send email"; fill it; verify connectors.
3. Retire the subflow only after all three callers are verified.
4. Nothing is switched ON at any point.

Main risks: node placement/wiring flakiness (mitigated by documented techniques and
the connector-count check); Deluge `throw`/options syntax (fallbacks in research.md
§3-4); accidentally editing a shared function in place (only the new function is
edited, and only before it's attached to more than one flow).
