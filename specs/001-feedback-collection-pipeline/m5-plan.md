# Implementation Plan: M5 - Discontinuation (Milestone 5)

**Branch**: `001-feedback-collection-pipeline`

**Date**: 2026-09-24

**Spec**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1, 2,
5, and 6 — Milestones table row 5)

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement
(same discipline M1/M2/M3/M4 followed — see `m1-plan.md`/`m2-plan.md`/
`m3-plan.md`/`m4-plan.md`). This plan is written before any Zoho
implementation begins for M5.

**Scope decision (Costin, 2026-09-24)**: spec.md's Milestones table describes
M5's trigger as two legs (cancellation with no rebooking, OR a no-show
pattern). This build covers the cancellation leg only —
`Patients1.Patient_Status = "Discontinued (Patient Choice)"`. The `No Show`
leg is explicitly out of scope; see `m5-research.md`'s "Open item for
spec.md" note. This also means M5's trigger is a single-condition check,
structurally identical in shape to M4's, not the two-legged OR this plan
originally scoped.

**Review checkpoints for this build (Costin, 2026-09-24)**: work pauses for
review at four points — (1) once these spec-kit planning docs are ready and
pushed [this commit], (2) after the Zoho Form is built, (3) after the Zoho
Flows (trigger + write-back) are built, (4) after the Analytics
reporting/dashboard is built. Each flow stays OFF and no live/end-to-end
test runs at any point without Costin, per the standing instruction — these
checkpoints are in addition to that, not a replacement for it.

## Summary

M5 fires automatically, exactly once per patient, when a patient cancels
without rebooking — scoped to `Patient_Status` transitioning to
`Discontinued (Patient Choice)`, the same one-time, single-field-transition
shape as M0's `Lost Lead` and M4's `Completed Treatment`. A new Zoho Flow
issues a single-use token and emails a link to a new "Cape Clarity
Discontinuation Feedback" survey — a genuinely short, two-question,
all-categorical exit-reason form, sourced verbatim from the Confluence design
page rather than drafted (the first milestone whose substantive survey/email
copy needed no drafting-and-flagging), via the same de-identified Zoho Form +
Flow write-back pattern every prior milestone validated. What's genuinely new
for M5, now that its trigger shape matches M4's: (1) it is the first
milestone with zero numeric/scored content, testing FR-018's "scored **or
categorical**" wording for the first time; and (2) no Clinical Safety Flag
work applies (the Alliance rule's own stated scope is Milestones 2–4, and M5
has no alliance data for it to evaluate).

## Technical Context

**Language/Runtime**: Zoho Deluge, Zoho Analytics formula language (same
stack as M0–M4).

**Primary Dependencies**: Zoho CRM (`Patients1.Patient_Status` — existing
field, new single-value trigger use; `Milestone_Instances` — existing, no
schema changes), Zoho Flow (new trigger flow + new write-back flow, reusing
the shared `issueFeedbackToken` and `isCreatedAfterCutoff` custom functions
rather than duplicating their logic — see `m5-research.md` Decisions 2–3),
Zoho Forms (new "Cape Clarity Discontinuation Feedback" form — 2 substantive
fields + hidden token, the shortest of any milestone), Zoho Mail, Zoho
Analytics (extends the shared "Milestone Instances" table: 2 new formula
columns, no modification to any existing shared column — see
`m5-data-model.md`).

**Storage**: `Milestone_Instances` remains the system of record, no new CRM
fields (still at the cap, re-confirmed live this session). `Response_Data`
reuses the delimited-blob pattern, 2 segments — the shortest of any milestone
— see `m5-research.md` Decision 4 and `m5-data-model.md`.

**Testing**: Same as M0–M4 — manual + Zoho CRM MCP tool-driven sample data,
no automated test suite. No live/end-to-end test without Costin (standing
instruction) — this session builds and verifies structurally only, with an
explicit review checkpoint after each of the Form, Flows, and Analytics
builds (see the Scope decision/checkpoints note above).

**Target Platform**: Same Zoho One stack.

**Project Type**: Low-code/no-code automation (same as M0–M4).

**Performance Goals**: N/A at current volume.

**Constraints**: Constitution Principles I (De-Identification by Design), II
(Single Rejoin Point), III (Automated Milestone Triggers), IV (Contractor
Blindness), and VII (Analytics Is Where Derived Values Get Computed). See
Constitution Check below.

**Scale/Scope**: Single milestone build, scoped to one trigger leg (see
above). Unlike M4, this plan touches no existing M2/M3/M4-owned Analytics
objects — M5's Clinical Safety Flag scope-check (Decision 6) concludes there
is nothing shared to widen.

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
  documents that the *underlying* 14-day judgment is made by staff and
  recorded as that status value (the same division of labor M0's `Lost
  Lead` and M4's `Completed Treatment` already establish) — what must be
  automated per this principle is the feedback request once the condition is
  recorded, which this design satisfies. Flagged for Costin/Liana
  confirmation is whether `Discontinued (Patient Choice)` correctly
  represents the cancellation-no-rebooking case, not whether the request
  itself is automated.
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: PASS — no
  email, Flow action, or CRM write in this design reaches a contractor
  anywhere; the On Error alert branches go only to `costin@capeclarity.com`.
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
  check, not a derived value computed from collected feedback data, so it
  isn't a Principle VII case at all. No Clinical Safety Flag logic exists
  for M5 to keep out of Deluge in the first place (Decision 6) — this
  milestone introduces zero derived/scored values of any kind, the simplest
  Principle VII posture of any milestone so far.
- **User Story 4 / FR-011–013 (Clinical Safety Flag)**: **Not applicable** —
  spec.md's Clinical Safety Flag Rules scope the Alliance rule to
  "Milestones 2, 3, 4" explicitly; M5 has no alliance domains or any other
  scored data for a flag rule to evaluate. See `m5-research.md` Decision 6.
- **User Story 6 / FR-018–019 (Baseline interpretive reporting)**: M5's
  reporting work is checked against this bar from the start, same as
  M1–M4 — see Phase 7 of `m5-tasks.md`. First milestone to test FR-018's
  "scored **or categorical**" wording literally, since M5 has no numeric
  score at all.
- **Compliance & Data Handling Requirements (purge of raw Zoho Forms
  entries)**: Same unresolved gap carried over from M0–M4
  (`data-retention-purge.md`) — not solved by M5, and M5 adds a sixth flow
  with the identical gap. No new investigation needed.
- **Legacy-patient cutoff-exclusion (feature 004)**: `m4-research.md`'s "Out
  of scope for M4" section explicitly left M5's own need for this gate as an
  open question for M5's own plan to resolve. **Resolved: yes** — see
  `m5-research.md` Decision 3, same any-update-re-evaluates-the-filter
  reasoning as M2/M3/M4 (M5's trigger is on `Patients1`, exactly like them,
  not `Leads` like M0's genuine exception).
- **Coverage of spec.md's Milestones-table row 5**: **Partial, by explicit
  Costin decision** — this build satisfies the cancellation-no-rebooking
  leg only, not the no-show leg. Flagged, not silently accepted; see
  `m5-research.md`'s "Open item for spec.md."

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

### "Source" (Zoho org)

```text
Customer Feedback System (Zoho Flow folder)
├── [existing, unmodified] issueFeedbackToken, isCreatedAfterCutoff (shared custom
│                                                 functions, features 002/004), M0/M2/M3/M4 flows
├── [new] M5 - Discontinuation Trigger           # watches Patients1.Patient_Status ==
│                                                 "Discontinued (Patient Choice)" only, gated by
│                                                 checkDiscontinuationExists (false) AND
│                                                 isCreatedAfterCutoff (true) — reuses both shared
│                                                 functions, built with both On Error branches from
│                                                 the start (m5-research.md Decisions 1-3)
├── [new] M5 - Discontinuation Write-back        # realtime Form-submission trigger + write-back
│                                                 function (plain string concatenation, 2 segments)

Zoho CRM
├── Patients1 module: Patient_Status (existing, new single-value trigger use)
└── Milestone_Instances module (existing, NO new custom fields — at CRM field cap,
    re-confirmed live) — Patient lookup populated (not Lead_Reference); reuses the
    existing dead-but-present picklist VALUE "5B - Discontinuation, Email Fallback"
    on the existing Milestone field (not a schema change — see m5-research.md
    Decision 5); Status picklist already includes "Send Failed"; Response_Data
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
      include, since Decision 6 finds no Clinical Safety Flag applicability
      for M5)
    [unchanged] no modification to any M2/M3/M4-owned shared Analytics object
```

## Complexity Tracking / Design Decisions

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| Trigger-condition-to-field mapping | `Patients1.Patient_Status = "Discontinued (Patient Choice)"` only, confirmed via live `getFields`, by Costin's explicit scope decision | Also wiring the `No Show` leg as an OR condition — this file's original draft; superseded by Costin's decision to keep one leg only for this build |
| Idempotency shape | Plain existence check (`checkDiscontinuationExists`) | M3's recurring-checkpoint-list machinery — needless complexity for a non-recurring condition |
| Legacy-patient cutoff gate | Yes — reuse `isCreatedAfterCutoff`, per `m5-research.md` Decision 3 | Skipping the gate — rejected for the same categorical reason M4 rejected it |
| `Milestone` picklist value | Reuse existing `"5B - Discontinuation, Email Fallback"` as-is, zero CRM schema change | Renaming it to a clean `"5 - Discontinuation"` — flagged as a possible future cleanup, not actioned now |
| Survey/email copy | Sourced verbatim from Confluence page 564035589 | Drafting new copy the way M0-M4 had to for their own new content — not needed here |
| Clinical Safety Flag scope | Not applicable — no widening, no new columns | Forking a parallel flag mechanism for M5's categorical data — rejected, spec.md's Alliance rule is explicitly scoped to Milestones 2-4 |

**Note for future sessions**: with the `No Show` leg out of scope, M5's
active-milestone coverage of spec.md's own Milestones table is partial on
row 5 specifically — see the Constitution Check's last line and
`m5-research.md`'s "Open item for spec.md." Once M5 ships as scoped here,
User Story 1's automated-triggers coverage is complete for the
cancellation-no-rebooking path; whether/when to add the no-show path is a
separate, future decision, not assumed here either way.
