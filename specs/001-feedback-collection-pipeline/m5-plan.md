# Implementation Plan: M5 - Discontinuation (Milestone 5)

**Branch**: `001-feedback-collection-pipeline`

**Date**: 2026-09-24

**Spec**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1, 2,
5, and 6 — Milestones table row 5)

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement
(same discipline M1/M2/M3/M4 followed — see `m1-plan.md`/`m2-plan.md`/
`m3-plan.md`/`m4-plan.md`). This plan is written before any Zoho
implementation begins for M5.

## Summary

M5 fires automatically, exactly once per patient, when a patient's care
lapses without a planned discharge — the last of the five active milestones,
and the only one whose spec.md trigger condition is a two-legged pattern
("cancellation with no rebooking within 14 days, OR 2 consecutive no-shows
with no reschedule") rather than a single field transition. This session
found no CRM-accessible appointment/session-level data to compute that
pattern automatically, so — mirroring M0's `Lost Lead` and M4's `Completed
Treatment` precedent, where a business judgment is recorded as a
`Patient_Status` value and the pipeline's automation begins there —
`m5-research.md` Decision 1 maps the two legs onto the two `Patient_Status`
values that are neither `Active`, `Completed Treatment`, nor `Referred Out`:
`Discontinued (Patient Choice)` (cancellation, no rebooking) and `No Show`
(the no-show pattern). A new Zoho Flow issues a single-use token and emails a
link to a new "Cape Clarity Discontinuation Feedback" survey — a genuinely
short, two-question, all-categorical exit-reason form, sourced verbatim from
the Confluence design page rather than drafted (the first milestone whose
substantive survey/email copy needed no drafting-and-flagging), via the same
de-identified Zoho Form + Flow write-back pattern every prior milestone
validated. What's genuinely new for M5: (1) the trigger's `Patient_Status`
condition is an OR across two values, not one — new territory for this
project's Flow-builder conventions, flagged for build-time confirmation
rather than assumed (`m5-research.md` Decision 2); (2) it is the first
milestone with zero numeric/scored content, testing FR-018's "scored **or
categorical**" wording for the first time; and (3) no Clinical Safety Flag
work applies (the Alliance rule's own stated scope is Milestones 2–4, and M5
has no alliance data for it to evaluate).

## Technical Context

**Language/Runtime**: Zoho Deluge, Zoho Analytics formula language (same
stack as M0–M4).

**Primary Dependencies**: Zoho CRM (`Patients1.Patient_Status` — existing
field, new dual-value trigger use; `Milestone_Instances` — existing, no
schema changes), Zoho Flow (new trigger flow + new write-back flow, reusing
the shared `issueFeedbackToken` and `isCreatedAfterCutoff` custom functions
rather than duplicating their logic — see `m5-research.md` Decisions 3–4),
Zoho Forms (new "Cape Clarity Discontinuation Feedback" form — 2 substantive
fields + hidden token, the shortest of any milestone), Zoho Mail, Zoho
Analytics (extends the shared "Milestone Instances" table: 2 new formula
columns, no modification to any existing shared column — see
`m5-data-model.md`).

**Storage**: `Milestone_Instances` remains the system of record, no new CRM
fields (still at the cap, re-confirmed live this session). `Response_Data`
reuses the delimited-blob pattern, 2 segments — the shortest of any milestone
— see `m5-research.md` Decision 5 and `m5-data-model.md`.

**Testing**: Same as M0–M4 — manual + Zoho CRM MCP tool-driven sample data,
no automated test suite. No live/end-to-end test without Costin (standing
instruction) — this session builds and verifies structurally only.

**Target Platform**: Same Zoho One stack.

**Project Type**: Low-code/no-code automation (same as M0–M4).

**Performance Goals**: N/A at current volume.

**Constraints**: Constitution Principles I (De-Identification by Design), II
(Single Rejoin Point), III (Automated Milestone Triggers), IV (Contractor
Blindness), and VII (Analytics Is Where Derived Values Get Computed). See
Constitution Check below.

**Scale/Scope**: Single milestone build. Unlike M4, this plan touches no
existing M2/M3/M4-owned Analytics objects — M5's Clinical Safety Flag
scope-check (Decision 7) concludes there is nothing shared to widen.

## Constitution Check

*Principle names/numbers match the repository's authoritative constitution
(v1.3.0).*

- **Principle I (De-Identification by Design)**: PASS — the new "Cape
  Clarity Discontinuation Feedback" form gets a hidden token field only, no
  name/email/phone. Both substantive questions are structured pick-lists
  (single-select reason category; Yes/No/Maybe) with no freeform field at
  all — the strictest de-identification posture of any milestone's form, and
  matches the Confluence source's own explicit "no open text" design note
  for this population's higher HIPAA-exposure profile (an already-active
  clinical record, unlike M0's pre-patient prospects).
- **Principle II (Single Rejoin Point)**: PASS — `Patient_Status` already
  lives in CRM and is only read, never duplicated. Token-to-Patient rejoin
  happens only in `Milestone_Instances`, same as every prior milestone.
- **Principle III (Automated Milestone Triggers)**: PASS — no contractor or
  staff action triggers the feedback request itself; it fires automatically
  off a system-detected `Patient_Status` value. `m5-research.md` Decision 1
  documents that the *underlying* 14-day/no-show-count judgment is made by
  staff and recorded as that status value (the same division of labor M0's
  `Lost Lead` and M4's `Completed Treatment` already establish, not a new
  exception) — what must be automated per this principle is the feedback
  request once the condition is recorded, which this design satisfies.
  Flagged for Costin/Liana confirmation is *which* two values correctly
  represent the two trigger legs, not whether the request itself is
  automated.
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: PASS — no
  email, Flow action, or CRM write in this design reaches a contractor
  anywhere; the On Error alert branches go only to `costin@capeclarity.com`.
  Not applicable in the additional sense M4's Principle IV check discussed
  (the shared Clinical Safety Flag columns), since M5 doesn't touch them at
  all (Decision 7).
- **Principle V (BAA-Gated Adoption)**: Same open gate as M0–M4 —
  `TODO(BAA_SCHEDULE)` still unresolved. M5 stays in test/sample-data mode
  until that's resolved.
- **Principle VI (Internal Use Only)**: PASS by construction, same as
  M0–M4 — no connection to any public-facing or marketing tool. Worth noting
  explicitly for M5 given its subject matter (patients who left care): the
  Confluence source's own design notes already anticipate and guard against
  this ("the reason captured here feeds the centralized business-level
  discontinuation objective... not through this structured capture" for
  anything clinically concerning) — nothing in this design routes M5 data
  anywhere public-facing or repurposes it as a testimonial.
- **Principle VII (Analytics Is Where Derived Values Get Computed)**: PASS.
  `submitDiscontinuationFeedbackResponse` does pure string concatenation, no
  arithmetic or conditional logic — same shape as every prior write-back
  function. `checkDiscontinuationExists` is a pre-collection existence
  check, not a derived value computed from collected feedback data, so (per
  `m4-plan.md`'s own explicit flag for its analogous function) it isn't a
  Principle VII case at all. No Clinical Safety Flag logic exists for M5 to
  keep out of Deluge in the first place (Decision 7) — this milestone
  introduces zero derived/scored values of any kind, the simplest Principle
  VII posture of any milestone so far.
- **User Story 4 / FR-011–013 (Clinical Safety Flag)**: **Not applicable** —
  spec.md's Clinical Safety Flag Rules scope the Alliance rule to
  "Milestones 2, 3, 4" explicitly; M5 has no alliance domains or any other
  scored data for a flag rule to evaluate. See `m5-research.md` Decision 7.
- **User Story 6 / FR-018–019 (Baseline interpretive reporting)**: M5's
  reporting work is checked against this bar from the start, same as
  M1–M4 — see Phase 6 of `m5-tasks.md`. First milestone to test FR-018's
  "scored **or categorical**" wording literally, since M5 has no numeric
  score at all.
- **Compliance & Data Handling Requirements (purge of raw Zoho Forms
  entries)**: Same unresolved gap carried over from M0–M4
  (`data-retention-purge.md`) — not solved by M5, and M5 adds a sixth flow
  with the identical gap. No new investigation needed.
- **Legacy-patient cutoff-exclusion (feature 004)**: `m4-research.md`'s "Out
  of scope for M4" section explicitly left M5's own need for this gate as an
  open question for M5's own plan to resolve. **Resolved: yes** — see
  `m5-research.md` Decision 4, same any-update-re-evaluates-the-filter
  reasoning as M2/M3/M4 (M5's trigger is on `Patients1`, exactly like them,
  not `Leads` like M0's genuine exception).

## Project Structure

### Documentation (this feature)

```text
specs/001-feedback-collection-pipeline/
├── spec.md
├── m0-plan.md / m0-tasks.md / m0-implementation-notes.md
├── m1-plan.md / m1-research.md / m1-data-model.md / m1-tasks.md / m1-implementation-notes.md   [RETIRED]
├── m2-plan.md / m2-research.md / m2-data-model.md / m2-tasks.md / m2-implementation-notes.md
├── m3-plan.md / m3-research.md / m3-data-model.md / m3-tasks.md / m3-implementation-notes.md
├── m4-plan.md / m4-research.md / m4-data-model.md / m4-tasks.md / m4-implementation-notes.md
├── m5-plan.md                 # this file
├── m5-research.md             # Phase 0 output
├── m5-data-model.md           # Phase 1 output
└── m5-tasks.md                # Phase 2 output (/speckit-tasks equivalent)
```

No `contracts/` or `quickstart.md` — same as M0–M4, this is a low-code Zoho
configuration project with no external/public API surface of its own.
(Feature 002's `contracts/issue-feedback-token.md` documents the one shared
contract M5 consumes but does not modify.)

### "Source" (Zoho org)

```text
Customer Feedback System (Zoho Flow folder)
├── [existing, unmodified] issueFeedbackToken, isCreatedAfterCutoff (shared custom
│                                                 functions, features 002/004), M0/M2/M3/M4 flows
├── [new] M5 - Discontinuation Trigger           # watches Patients1.Patient_Status ==
│                                                 "Discontinued (Patient Choice)" OR "No Show",
│                                                 gated by checkDiscontinuationExists (false) AND
│                                                 isCreatedAfterCutoff (true) — reuses both shared
│                                                 functions, built with both On Error branches from
│                                                 the start (m5-research.md Decisions 2-4)
├── [new] M5 - Discontinuation Write-back        # realtime Form-submission trigger + write-back
│                                                 function (plain string concatenation, 2 segments)

Zoho CRM
├── Patients1 module: Patient_Status (existing, new dual-value trigger use)
└── Milestone_Instances module (existing, NO new custom fields — at CRM field cap,
    re-confirmed live) — Patient lookup populated (not Lead_Reference); reuses the
    existing dead-but-present picklist VALUE "5B - Discontinuation, Email Fallback"
    on the existing Milestone field (not a schema change — see m5-research.md
    Decision 6); Status picklist already includes "Send Failed"; Response_Data
    reused for a 2-segment blob, the shortest of any milestone (see m5-data-model.md)

Zoho Forms
└── [new] Cape Clarity Discontinuation Feedback (1 dropdown/radio reason-category
    field + 1 Yes/No/Maybe radio + hidden token — no sliders, no freeform)

Zoho Analytics
└── [extend] Milestone Instances table —
    [new] "Reason For Leaving", "Okay To Reach Back Out" formula columns
      (substring_between / unbounded-last-field extraction);
    [new] M5 Submitted Responses, M5 Status Breakdown, M5 Reason For Leaving
      Distribution (User Story 6 / FR-018 baseline bar — first milestone
      clearing it with a categorical, not numeric, distribution view),
      optionally M5 Okay To Reach Back Out Breakdown;
    [new] "M5 - Discontinuation Feedback" dashboard bundling the above (same
      shape as M0-M4's dashboards; no shared Flagged for Review panel to
      include, since Decision 7 finds no Clinical Safety Flag applicability
      for M5)
    [unchanged] no modification to any M2/M3/M4-owned shared Analytics object
```

## Complexity Tracking / Design Decisions

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| Trigger-condition-to-field mapping | `Patients1.Patient_Status = "Discontinued (Patient Choice)"` OR `"No Show"`, confirmed via live `getFields`, inferred by elimination against M0's/M4's precedent, flagged for Costin/Liana confirmation | Computing the 14-day/no-show-count pattern from appointment-level data — rejected, no confirmed CRM data source exists for this, and building one is new infrastructure well beyond this milestone's scope |
| Trigger-level OR across two values on one field | Attempt trigger-level OR filter first; fall back to a broadened trigger + in-flow OR check ANDed with the existing 2-clause `If else` if the builder doesn't support it | Assuming one approach works without confirming against the live builder — rejected per this project's own "verify, don't assume" discipline, especially since no prior milestone needed a same-field OR at this layer |
| Idempotency shape | Plain existence check (`checkDiscontinuationExists`), one instance per patient regardless of which trigger leg fired it | Tracking the two trigger legs as independently-triggerable sub-instances — rejected, spec.md's Edge Cases don't describe M5 this way, and the CRM's own 5A/5B split was never a cancellation-vs-no-show split |
| Legacy-patient cutoff gate | Yes — reuse `isCreatedAfterCutoff`, per `m5-research.md` Decision 4 (same any-update-re-evaluates-the-filter risk M2/M3/M4 have, since M5's trigger is also an "Updated module entry" on `Patients1`) | Skipping the gate — rejected for the same categorical reason M4 rejected it |
| `Milestone` picklist value | Reuse existing `"5B - Discontinuation, Email Fallback"` as-is, zero CRM schema change | Renaming it to a clean `"5 - Discontinuation"` — flagged as a possible future cleanup, not actioned now; would be a live CRM Setup/customization change with no functional requirement forcing it |
| Survey/email copy | Sourced verbatim from Confluence page 564035589 | Drafting new copy the way M0-M4 had to for their own new content — not needed here, the source page already has fully worked content |
| Clinical Safety Flag scope | Not applicable — no widening, no new columns | Forking a parallel flag mechanism for M5's categorical data — rejected, spec.md's Alliance rule is explicitly scoped to Milestones 2-4 and nothing in M5's data resembles an alliance reading |

**Note for future sessions**: M5 is the last of the five *active* milestones
named in spec.md's Milestones table (0, 2, 3, 4, 5). Once M5 is built, this
pipeline's User Story 1 (automated triggers for all milestones) is complete
end-to-end; remaining pipeline-wide work is User Story 3 (the unified,
cross-milestone Admin Dashboard, FR-007–009) and the still-open raw-Forms
purge gap (FR-006, `data-retention-purge.md`) — both explicitly deferred by
every milestone's own "Out of scope" section, not newly discovered here.
