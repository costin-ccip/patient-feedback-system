# Implementation Plan: M4 - Discharge (Milestone 4)

**Branch**: `001-feedback-collection-pipeline`

**Date**: 2026-09-23

**Spec**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1, 2,
4, and 6 — Milestones table row 4; Clinical Safety Flag Rules)

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement
(same discipline M1/M2/M3 followed — see `m1-plan.md` / `m2-plan.md` /
`m3-plan.md`). This plan is written before any Zoho implementation begins for
M4.

## Summary

M4 fires automatically, exactly once per patient, when discharge is marked in
the system of record — a one-time-per-patient condition like M0/M2, not a
recurring one like M3 (`m3-plan.md`'s own "Precedent for M4/M5" note already
calls this). Because spec.md never names the exact CRM field/value behind
"discharge is marked," `m4-research.md` Decision 1 maps it to
`Patients1.Patient_Status = "Completed Treatment"` (confirmed via the live
`getFields` picklist; flagged for Costin/Liana confirmation, same as M0's
own inferred `Lost Lead` mapping). A new Zoho Flow issues a single-use token
and emails a link to a new "Cape Clarity Discharge Feedback" survey — the
same 4-domain Alliance Check-In instrument M2/M3 built (reused verbatim, per
M2's own "designed for reuse" note), plus a new 2-question "Looking ahead"
exit section (likelihood to recommend + optional freeform) — via the same
de-identified Zoho Form + Flow write-back pattern M0/M1/M2/M3 validated. The
write-back function stays in the same pure-string-concatenation shape
(Constitution Principle VII). What's genuinely new for M4: (1) it's the
first milestone built entirely after both the no-subflow token-issuance
convention (feature 002) and the On-Error failure-handling convention
(feature 003) already existed, so both are built in from the start rather
than retrofitted; (2) `m4-research.md` Decision 3 concludes M4 needs the
legacy-patient cutoff-exclusion gate (feature 004) for the same
any-update-re-evaluates-the-filter reason M2/M3 needed it — feature 004 had
explicitly left this undecided for M4; (3) the Clinical Safety Flag Rules
already name Milestone 4 in their own stated scope ("Milestones 2, 3, 4"),
so this plan widens M2/M3's existing flag columns' `Milestone` gate a second
time rather than forking a parallel copy, the same "one rule, one report"
reasoning M3 already established.

## Technical Context

**Language/Runtime**: Zoho Deluge, Zoho Analytics formula language (same
stack as M0/M1/M2/M3).

**Primary Dependencies**: Zoho CRM (`Patients1.Patient_Status` — existing
field, new trigger use; `Milestone_Instances` — existing, no schema changes),
Zoho Flow (new trigger flow + new write-back flow, reusing the shared
`issueFeedbackToken` and `isCreatedAfterCutoff` custom functions rather than
duplicating their logic — see `m4-research.md` Decisions 2–3), Zoho Forms
(new "Cape Clarity Discharge Feedback" form — 6 fields: the 4 reused alliance
sliders + 2 new Looking Ahead fields, plus the hidden token), Zoho Mail, Zoho
Analytics (extends the shared "Milestone Instances" table: two widened flag
columns, two new formula columns — see `m4-data-model.md`).

**Storage**: `Milestone_Instances` remains the system of record, no new CRM
fields (still at the cap `m2-data-model.md` confirmed, re-confirmed live this
session — see `m4-data-model.md`). `Response_Data` reuses the delimited-blob
pattern, 6 segments, alliance domains last (same field-order reasoning as
`m3-research.md` Decision 4) — see `m4-research.md` Decision 5 and
`m4-data-model.md`.

**Testing**: Same as M0/M1/M2/M3 — manual + Zoho CRM MCP tool-driven sample
data, no automated test suite. No live/end-to-end test without Costin
(standing instruction) — this session builds and verifies structurally only.

**Target Platform**: Same Zoho One stack.

**Project Type**: Low-code/no-code automation (same as M0/M1/M2/M3).

**Performance Goals**: N/A at current volume.

**Constraints**: Constitution Principles I (De-Identification by Design), II
(Single Rejoin Point), III (Automated Milestone Triggers), IV (Contractor
Blindness), and VII (Analytics Is Where Derived Values Get Computed, again
governing a modification to already-shipped M2/M3 formula columns, not just
new ones). See Constitution Check below.

**Scale/Scope**: Single milestone build. Like M3, this plan also touches
existing M2/M3 Analytics objects (widening the Clinical Safety Flag / Flag
Rule Triggered gate and the shared Flagged for Review report a second time)
— scoped narrowly to that one change, not a general M2/M3 rebuild.

## Constitution Check

*Principle names/numbers match the repository's authoritative constitution
(v1.3.0).*

- **Principle I (De-Identification by Design)**: PASS — the new "Cape
  Clarity Discharge Feedback" form gets a hidden token field only, no
  name/email/phone, same as every prior form. All 6 substantive questions are
  scored/categorical or free of identity content (the one freeform field,
  "Looking Ahead: Anything Else," is optional and never required to contain
  identifying content — same treatment M0's own freeform fields get).
- **Principle II (Single Rejoin Point)**: PASS — `Patient_Status` already
  lives in CRM and is only read, never duplicated. Token-to-Patient rejoin
  happens only in `Milestone_Instances`, same as M0/M1/M2/M3.
- **Principle III (Automated Milestone Triggers)**: PASS — discharge being
  marked (`Patient_Status` transitioning to `Completed Treatment`) is a
  system-detected CRM-field condition evaluated by a Zoho Flow trigger, with
  no contractor action required or possible. `m4-research.md` Decision 1
  documents the trigger-condition-to-field mapping and flags it for
  Costin/Liana confirmation; that confirmation affects *which* field value is
  watched, not whether the trigger is automated.
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: PASS, same
  design M2/M3 already established and validated for the alliance flag — M4
  reuses (widens) those exact columns rather than building a new mechanism.
  No email, Flow action, or CRM write reaches a contractor anywhere in this
  design; the On Error alert branches (feature 003's convention) go only to
  `costin@capeclarity.com`, an admin address, never to
  `Clinician`/`Assigned_Therapist`.
- **Principle V (BAA-Gated Adoption)**: Same open gate as M0/M1/M2/M3 —
  `TODO(BAA_SCHEDULE)` still unresolved. M4 stays in test/sample-data mode
  until that's resolved.
- **Principle VI (Internal Use Only)**: PASS by construction, same as
  M0/M1/M2/M3 — no connection to any public-facing or marketing tool.
- **Principle VII (Analytics Is Where Derived Values Get Computed)**: PASS.
  `submitDischargeFeedbackResponse` (the new write-back function) does pure
  string concatenation, no arithmetic or conditional logic — same shape as
  every prior write-back function. `checkDischargeExists` is a pre-collection
  existence check (same category as `checkAllianceCheckExists`), not a
  derived value computed from collected feedback data, so it isn't a
  Principle VII case at all — flagged explicitly, same as `m3-plan.md` did
  for its own idempotency function, so a future reviewer doesn't mistake it
  for one. The Clinical Safety Flag / Flag Rule Triggered widening keeps the
  derived-value computation entirely inside Analytics, per the principle's
  stated preference — no flag logic is duplicated into
  `submitDischargeFeedbackResponse` or the trigger flow.
- **User Story 4 / FR-011–013 (Clinical Safety Flag)**: In scope for M4 —
  spec.md's Clinical Safety Flag Rules already state "Milestones 2, 3, 4" as
  the rule's scope, so this is applying an already-scoped rule to its last
  named milestone, not extending the rule's reach. This plan **widens**
  M2/M3's existing "Clinical Safety Flag" / "Flag Rule Triggered" formula
  columns' `Milestone` gate a second time rather than forking a parallel
  M4-only pair — see `m4-research.md` Decision 6. Per CLAUDE.md's
  implementation-notes convention, this requires updating
  `m2-implementation-notes.md` and `m3-implementation-notes.md` in the same
  session this change is made, since it changes how both milestones'
  already-documented, shared Analytics objects actually work.
- **User Story 6 / FR-018–019 (Baseline interpretive reporting)**: M4's
  reporting work is checked against this bar from the start, same as
  M1/M2/M3 — see Phase 6 of `m4-tasks.md`.
- **Compliance & Data Handling Requirements (purge of raw Zoho Forms
  entries)**: Same unresolved gap carried over from M0/M1/M2/M3
  (`data-retention-purge.md`) — not solved by M4, and M4 adds a fifth flow
  with the identical gap. No new investigation needed.
- **Legacy-patient cutoff-exclusion (feature 004)**: `specs/004-legacy-patient-exclusion/spec.md`
  explicitly left M4's need for the `isCreatedAfterCutoff` gate undecided,
  to be resolved by this plan. **Resolved: yes** — see `m4-research.md`
  Decision 3. This reuses the existing shared function object; it does not
  modify feature 004's own files, so no update to
  `specs/004-legacy-patient-exclusion/implementation-notes.md` is needed
  beyond noting the new consumer, which is instead recorded here and in
  `m4-implementation-notes.md`.

## Project Structure

### Documentation (this feature)

```text
specs/001-feedback-collection-pipeline/
├── spec.md
├── m0-plan.md / m0-tasks.md / m0-implementation-notes.md
├── m1-plan.md / m1-research.md / m1-data-model.md / m1-tasks.md / m1-implementation-notes.md   [RETIRED]
├── m2-plan.md / m2-research.md / m2-data-model.md / m2-tasks.md / m2-implementation-notes.md
├── m3-plan.md / m3-research.md / m3-data-model.md / m3-tasks.md / m3-implementation-notes.md
├── m4-plan.md                 # this file
├── m4-research.md             # Phase 0 output
├── m4-data-model.md           # Phase 1 output
└── m4-tasks.md                # Phase 2 output (/speckit-tasks equivalent)
```

No `contracts/` or `quickstart.md` — same as M0/M1/M2/M3, this is a low-code
Zoho configuration project with no external/public API surface of its own.
(Feature 002's `contracts/issue-feedback-token.md` documents the one shared
contract M4 consumes but does not modify.)

### "Source" (Zoho org)

```text
Customer Feedback System (Zoho Flow folder)
├── [existing, unmodified] issueFeedbackToken, isCreatedAfterCutoff (shared custom
│                                                 functions, features 002/004), M0/M2/M3 flows
├── [new] M4 - Discharge Trigger                # watches Patients1.Patient_Status ==
│                                                 "Completed Treatment", gated by
│                                                 checkDischargeExists (false) AND
│                                                 isCreatedAfterCutoff (true) — reuses both
│                                                 shared functions, built with both On Error
│                                                 branches from the start (m4-research.md
│                                                 Decisions 2, 3, 8)
├── [new] M4 - Discharge Write-back              # realtime Form-submission trigger + write-back
│                                                 function (plain string concatenation, 6 segments)

Zoho CRM
├── Patients1 module: Patient_Status (existing, new trigger use)
└── Milestone_Instances module (existing, NO new custom fields — at CRM field cap,
    re-confirmed live) — Patient lookup populated (not Lead_Reference); new picklist
    VALUE "4 - Discharge" already exists on the existing Milestone field (not a schema
    change — see m4-research.md Decision 7); Status picklist already includes "Send
    Failed" (added by Costin since feature 003); Response_Data reused for a 6-segment
    blob, alliance domains ordered last (see m4-data-model.md)

Zoho Forms
└── [new] Cape Clarity Discharge Feedback (4 reused alliance sliders + 2 new Looking
    Ahead fields + hidden token)

Zoho Analytics
└── [extend] Milestone Instances table —
    [modify] "Clinical Safety Flag" and "Flag Rule Triggered" — widen the Milestone
      gate a second time to cover M4 alongside M2/M3 (see m4-research.md Decision 6) —
      requires m2-implementation-notes.md AND m3-implementation-notes.md updates in the
      same session;
    [modify] the shared "M2 & M3 Flagged for Review" report — widen and rename to
      "M2, M3 & M4 Flagged for Review";
    [new] "Looking Ahead: Likelihood To Recommend", "Looking Ahead: Anything Else"
      formula columns (bounded substring_between);
    [new] M4 Submitted Responses, M4 Status Breakdown, M4 Looking Ahead: Likelihood To
      Recommend Distribution (User Story 6 / FR-018 baseline bar), reusing the shared
      Flagged for Review view (widened, not duplicated);
    [new] "M4 - Discharge Feedback" dashboard bundling the above (same shape as
      M0/M1/M2/M3's dashboards)
```

## Complexity Tracking / Design Decisions

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| Trigger-condition-to-field mapping | `Patients1.Patient_Status = "Completed Treatment"`, confirmed via live `getFields`; confirmed correct by Costin 2026-09-24 | Waiting for Costin to name the exact field before writing plan.md — rejected as an unnecessary block; the mapping is a one-line change if wrong |
| Idempotency shape | Plain existence check (`checkDischargeExists`), per `m3-plan.md`'s own prospective precedent for M4/M5 | M3's recurring-checkpoint-list machinery generalized to a 1-item list — needless complexity for a non-recurring condition |
| Legacy-patient cutoff gate | Yes — reuse `isCreatedAfterCutoff`, per `m4-research.md` Decision 3 (same any-update-re-evaluates-the-filter risk M2/M3 have, since M4's trigger is also an "Updated module entry" on `Patients1`) | Skipping the gate on the theory that discharge is rarer than session-count increments — rejected: the risk is categorical (trigger mechanism), not a matter of how often the field changes |
| On Error branches | Built from the start (feature 003's convention already exists) | Building a bare happy-path flow and retrofitting error handling later, the way M0/M2/M3 had to — unnecessary now that the convention predates this milestone |
| Response_Data blob field order | Looking Ahead segments first, alliance domains last (Connection/Understanding/Shared direction/Fit of approach, Fit of approach last/unbounded) | Matching the form's on-screen order (alliance first) — would require re-deriving the unbounded-last-field guard for a domain column instead of reusing M2/M3's already-correct one |
| Where the Clinical Safety Flag lives for M4 | Widen M2/M3's existing "Clinical Safety Flag"/"Flag Rule Triggered" columns' Milestone gate a second time | A parallel M4-only pair of flag columns — rejected as unnecessary duplication of one rule spec.md already scopes across Milestones 2-4 |
| Survey delivery mechanism | New standalone "Cape Clarity Discharge Feedback" form (repeats M2/M3's 4 alliance questions verbatim, adds a new 2-question Looking Ahead section) | Extending M3's Periodic Check-In form with a discharge-only branch — rejected, spec.md describes M4 as its own milestone with its own instance of the alliance instrument (a "final reading"), and Zoho Forms has no per-submission conditional-section mechanism that would make a shared form simpler than a separate one |

**Note for future milestones**: M4 was the first milestone whose planning
started with two shared custom functions (`issueFeedbackToken`,
`isCreatedAfterCutoff`) and one shared On-Error pattern already established
as conventions in `CLAUDE.md`, rather than having to discover and document
them mid-build the way M0/M2/M3 did. M5, when it's built, should be able to
follow this plan's structure even more directly — the main open questions
left for M5's own research.md are (1) its own trigger-condition-to-field
mapping (spec.md's Milestone 5 trigger is a cancellation/no-show *pattern*,
not a single field transition, so it may need real new logic, unlike M4) and
(2) whether it needs the cutoff-exclusion gate (an open question this plan
does not resolve, per `m4-research.md`'s Out of scope section).
