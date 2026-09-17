# Implementation Plan: M3 - Periodic Consolidated Check-In (Milestone 3)

**Branch**: `001-feedback-collection-pipeline`

**Date**: 2026-09-17

**Spec**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1, 2,
4, and 6 — Milestones table row 3; Clinical Safety Flag Rules, alliance half)

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement
(same discipline M1/M2 followed — see `m1-plan.md` / `m2-plan.md`). This plan
is written before any Zoho implementation begins for M3.

## Summary

M3 fires automatically and **repeatedly** for the same patient, at a short,
fixed list of session-count checkpoints — **8, 16, 24, 32, 40** — rather
than at one point in the patient's lifecycle. This is the first recurring
milestone in the pipeline; M0/M1/M2 are all one-time-per-record. **Revised
2026-09-17 (Costin)**: the original design derived checkpoints from
open-ended "any multiple of 8, forever" arithmetic; Costin asked for the
simpler version — an explicit, short list of the actual session counts that
get a survey, easy to read and easy to extend later by editing one list,
rather than trusting a modulo formula to keep behaving correctly
indefinitely (`m3-research.md` Decisions 1–2). A new Zoho Flow issues a
single-use token and emails a link to a new "Cape Clarity Periodic
Check-In" survey — the same 4-domain Alliance Check-In instrument M2 built
(reused verbatim, per M2's own "designed for reuse at M3" note), plus two
new sections (practice experience, therapist professionalism) — via the
same de-identified Zoho Form + Flow write-back pattern M0/M1/M2 validated.
The write-back function stays in the same pure-string-concatenation shape
M0/M1/M2 already established (Constitution Principle VII — no arithmetic
or flag logic in Deluge). What's genuinely new: (1) the idempotency check
can't be a simple existence test (that would block every occurrence after
the first) — it looks up the next checkpoint in the fixed list by position,
using a count of the patient's existing M3 instances as the index
(`m3-research.md` Decision 2), and stops firing once the patient has had
all 5 defined checkpoints, unless the list is later extended; (2) M3's blob has more
segments than M2's, so the field order is deliberately chosen to keep the
alliance domains last, preserving M2's already-shipped "Domain: Fit Of
Approach" Analytics formula column's unbounded-last-field assumption instead
of breaking it (`m3-research.md` Decision 4); (3) the Clinical Safety Flag
Rules apply to M3 the same way they apply to M2 (same alliance rule, no
trend logic — the wellbeing half is permanently retired, not merely deferred
past M3), so M3 widens M2's existing flag columns' `Milestone` gate rather
than forking a parallel copy (`m3-research.md` Decision 5) — the one place
this plan modifies already-shipped M2 Analytics infrastructure, not just new
M3-only work.

## Technical Context

**Language/Runtime**: Zoho Deluge, Zoho Analytics formula language (same
stack as M0/M1/M2).

**Primary Dependencies**: Zoho CRM (`Patients1.Session_Count` — existing,
already automated; `Milestone_Instances` — existing, no schema changes),
Zoho Flow (new trigger flow + new write-back flow, mirroring M0/M1/M2's
pattern but with a materially different idempotency function — see
`m3-research.md` Decisions 1–2), Zoho Forms (new "Cape Clarity Periodic
Check-In" form — 7 fields: the 4 reused alliance sliders + 2 practice-
experience sliders + 1 professionalism slider, plus the hidden token),
Zoho Mail, Zoho Analytics (extends the shared "Milestone Instances" table:
one modified formula column, two widened flag columns, three new formula
columns — see `m3-data-model.md`).

**Storage**: `Milestone_Instances` remains the system of record, no new CRM
fields (still at the cap `m2-data-model.md` confirmed). `Response_Data`
reuses the delimited-blob pattern, 7 segments instead of M2's 4, in a
deliberately non-form-order sequence (alliance domains last) — see
`m3-research.md` Decision 4 and `m3-data-model.md`.

**Testing**: Same as M0/M1/M2 — manual + Zoho CRM MCP tool-driven sample
data, no automated test suite. No live/end-to-end test without Costin
(standing instruction) — this session builds and verifies structurally
only.

**Target Platform**: Same Zoho One stack.

**Project Type**: Low-code/no-code automation (same as M0/M1/M2).

**Performance Goals**: N/A at current volume.

**Constraints**: Constitution Principles I (De-Identification by Design), II
(Single Rejoin Point), III (Automated Milestone Triggers, now covering a
recurring — not one-time — condition for the first time), IV (Contractor
Blindness), and VII (Analytics Is Where Derived Values Get Computed, now
also governing a modification to an *already-shipped* formula column, not
just new ones). See Constitution Check below.

**Scale/Scope**: Single milestone build. Unlike M0/M1/M2, this plan also
touches existing M2 Analytics objects (widening the Clinical Safety Flag /
Flag Rule Triggered gate) — scoped narrowly to that one change, not a
general M2 rebuild.

## Constitution Check

*Principle names/numbers match the repository's authoritative constitution
(v1.3.0).*

- **Principle I (De-Identification by Design)**: PASS — the new "Cape
  Clarity Periodic Check-In" form gets a hidden token field only, no
  name/email/phone, same as every prior form. All 7 substantive questions
  are scored/categorical or free of identity content.
- **Principle II (Single Rejoin Point)**: PASS — `Session_Count` already
  lives in CRM and is only read, never duplicated; the recurring-checkpoint
  math (`m3-research.md` Decision 2) is computed from `Milestone_Instances`
  record counts already in CRM, not a new identity/token store. Token-to-
  Patient rejoin happens only in `Milestone_Instances`, same as M0/M1/M2.
- **Principle III (Automated Milestone Triggers)**: PASS, and the first
  milestone where this principle governs a **recurring**, not one-time,
  system-detected condition. `Session_Count` crossing each of the 5 fixed
  checkpoints (8/16/24/32/40, per Costin's 2026-09-17 simplification) is
  still a real, system-detected CRM-field condition — no contractor
  involvement at any point — see `m3-research.md` Decisions 1–2 for why the
  detection mechanism differs from M0/M1/M2 without weakening this
  principle.
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: PASS, same
  design M2 already established and validated for the alliance flag — M3
  reuses (widens) those exact columns rather than building a new mechanism,
  so the same reasoning applies: flag computed only in Analytics, from data
  contractors have no access to, surfaced only through an admin-only
  Analytics report. No email, Flow action, or CRM write reaches a
  contractor anywhere in this design.
- **Principle V (BAA-Gated Adoption)**: Same open gate as M0/M1/M2 —
  `TODO(BAA_SCHEDULE)` still unresolved. M3 stays in test/sample-data mode
  until that's resolved.
- **Principle VI (Internal Use Only)**: PASS by construction, same as
  M0/M1/M2 — no connection to any public-facing or marketing tool.
- **Principle VII (Analytics Is Where Derived Values Get Computed)**: PASS,
  and the first milestone to also **modify** an already-shipped Analytics
  formula column rather than only add new ones. `submitPeriodicCheckInResponse`
  (the new write-back function) does pure string concatenation, no
  arithmetic or conditional logic — same shape as M0/M1/M2. The recurring-
  checkpoint arithmetic in `checkPeriodicCheckInDue` (Deluge, inside the
  trigger flow) is **not** a Principle VII violation: it isn't computing a
  derived *value/score/flag from collected feedback data* the way the
  Clinical Safety Flag is — it's an idempotency/eligibility gate on whether
  to create a new `Milestone_Instances` record at all, the same category of
  logic `checkAllianceCheckExists` already performs in Deluge for M2 (a
  pre-collection existence check, not a post-collection derived value). No
  documented Principle VII exception is being claimed for it because it
  doesn't fall under the principle's scope in the first place — flagged
  explicitly here so a future reviewer doesn't mistake it for one.
- **User Story 4 / FR-011–013 (Clinical Safety Flag)**: In scope for M3,
  same alliance rule as M2, no new rule needed — see `m3-research.md`
  Decision 5. This plan **widens** M2's existing "Clinical Safety Flag" /
  "Flag Rule Triggered" formula columns' `Milestone` gate rather than
  forking a parallel M3-only pair. Per CLAUDE.md's implementation-notes
  convention, this requires updating `m2-implementation-notes.md` in the
  same session this change is made (not just `m3-implementation-notes.md`),
  since it changes how M2's already-documented Analytics build actually
  works.
- **User Story 6 / FR-018–019 (Baseline interpretive reporting)**: M3's
  reporting work is checked against this bar from the start, same as
  M1/M2 — see Phase 6 of `m3-tasks.md`.
- **Compliance & Data Handling Requirements (purge of raw Zoho Forms
  entries)**: Same unresolved gap carried over from M0/M1/M2
  (`data-retention-purge.md`) — not solved by M3, and M3 adds a fourth flow
  with the identical gap. No new investigation needed.

## Project Structure

### Documentation (this feature)

```text
specs/001-feedback-collection-pipeline/
├── spec.md
├── m0-plan.md / m0-tasks.md / m0-implementation-notes.md
├── m1-plan.md / m1-research.md / m1-data-model.md / m1-tasks.md / m1-implementation-notes.md   [RETIRED]
├── m2-plan.md / m2-research.md / m2-data-model.md / m2-tasks.md / m2-implementation-notes.md
├── m3-plan.md                 # this file
├── m3-research.md             # Phase 0 output
├── m3-data-model.md           # Phase 1 output
└── m3-tasks.md                # Phase 2 output (/speckit-tasks equivalent)
```

No `contracts/` or `quickstart.md` — same as M0/M1/M2, this is a low-code
Zoho configuration project with no external/public API surface of its own.

### "Source" (Zoho org)

```text
Customer Feedback System (Zoho Flow folder)
├── [existing] Subflow - Issue Feedback Token, M0/M1/M2 flows
├── [new] M3 - Periodic Check-In Trigger        # watches Patients1.Session_Count >= 8,
│                                                 gated by checkPeriodicCheckInDue's recurring-
│                                                 checkpoint math (m3-research.md Decision 2)
├── [new] M3 - Periodic Check-In Write-back     # realtime Form-submission trigger + write-back
│                                                 function (plain string concatenation, 7 segments)

Zoho CRM
├── Patients1 module: Session_Count (existing, already automated)
└── Milestone_Instances module (existing, NO new custom fields — at CRM field cap) —
    Patient lookup populated (not Lead_Reference); new picklist VALUE
    "3 - Periodic Consolidated" added to the existing Milestone field (not a
    schema change — see m3-research.md Decision 6);
    Response_Data reused for a 7-segment blob, alliance domains ordered last
    (see m3-data-model.md)

Zoho Forms
└── [new] Cape Clarity Periodic Check-In (4 reused alliance sliders +
    2 practice-experience sliders + 1 professionalism slider + hidden token)

Zoho Analytics
└── [extend] Milestone Instances table —
    [modify] "Domain: Fit Of Approach" formula column, if a bounded-or-end-of-string
      rewrite turns out cleaner than the blob-ordering fix at build time (see
      m3-research.md Decision 4 — the primary plan keeps this column untouched);
    [modify] "Clinical Safety Flag" and "Flag Rule Triggered" — widen the Milestone
      gate to cover M3 alongside M2 (see m3-research.md Decision 5) — requires an
      m2-implementation-notes.md update in the same session;
    [new] "Practice Experience: Scheduling/Communication", "Practice Experience:
      Billing", "Therapist Professionalism" formula columns (bounded substring_between);
    [new] M3 Submitted Responses, M3 Status Breakdown, M3 distribution/summary
      view(s) of the new practice-experience/professionalism data (User Story 6 /
      FR-018 baseline bar), reusing the shared M2/M3 Flagged for Review view
      (widened, not duplicated);
    [new] "M3 - Periodic Check-In Feedback" dashboard bundling the above (same
      shape as M0/M1/M2's dashboards)
```

**Structure Decision**: Same configuration-only approach as M0/M1/M2. The one
structural departure from M2: this plan deliberately reaches back into M2's
already-shipped Analytics objects for one narrow, documented change (the
Clinical Safety Flag gate), rather than treating M2 as frozen — because
spec.md defines the Clinical Safety Flag Rules as one rule set spanning
Milestones 2–4, not per-milestone variants, and duplicating the formula
three times would violate Principle VII's own stated rationale for
preferring a single editable formula column. See `m3-research.md` Decision 5
for the full argument and the low-risk assessment (M2 has zero real
production rows as of this writing).

## Complexity Tracking / Design Decisions

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| How to detect a recurring "every 8th session" condition | Broad trigger filter (`Session_Count >= 8`) + a Deluge eligibility function checking membership in a fixed checkpoint list `[8,16,24,32,40]` (**revised 2026-09-17, Costin** — see below) | Open-ended modulo arithmetic ("any multiple of 8, forever") — this session's first draft, superseded once Costin asked for the simpler, bounded version; a modulo/divisibility filter directly on the CRM trigger step is also not supported by Zoho Flow's trigger filter criteria either way |
| How to make a recurring milestone idempotent without new CRM fields | Derive `nextThreshold` by indexing into `CHECKPOINTS[COUNT(existing M3 instances for this patient)]`, fire on `Session_Count >= nextThreshold`, stop once the count exceeds the list | A new `Session_Count_At_Trigger` field (blocked — no field budget); storing the checkpoint inside `Response_Data` (rejected — conflates issuance-tracking with response-recording, more fragile); open-ended `(existingCount + 1) * 8` arithmetic (superseded per above — same idempotency mechanism, just unbounded) |
| Checkpoint-match strictness | `>=` (catches up after any skip/jump, fires exactly once per checkpoint) | Exact `==` per listed checkpoint (can permanently and silently drop a checkpoint on a backdated/bulk session-count correction) |
| Bounded vs. open-ended recurrence | **Fixed list, 8/16/24/32/40**, extendable later by editing one list (Costin, 2026-09-17) | Open-ended "every 8th session for the duration of treatment" (spec.md's literal wording) — simpler to build and audit, but means a patient past session 40 gets no further M3 check-ins until the list is extended; flagged for Costin/Liana in `m3-research.md` Decision 2 |
| Response_Data blob field order | Practice-experience + professionalism segments first, alliance domains last | Matching the form's on-screen section order (alliance first) — would break M2's already-shipped unbounded "Domain: Fit Of Approach" column, which assumes nothing follows it |
| Where the Clinical Safety Flag lives for M3 | Widen M2's existing "Clinical Safety Flag"/"Flag Rule Triggered" columns' Milestone gate to include M3 | A parallel M3-only pair of flag columns — rejected as unnecessary duplication of one rule spec.md defines as shared across Milestones 2–4 |
| Survey delivery mechanism | New standalone "Cape Clarity Periodic Check-In" form (repeats M2's 4 alliance questions verbatim, adds 2 new sections) | Sending patients back to M2's existing Alliance Check-In form and combining data out-of-band — rejected, spec.md describes M3 as one consolidated request, and M2's form/flow is already fixed to a 4-field shape |

**Precedent for M4/M5**: M4 (Discharge) and M5 (Discontinuation) are both
one-time-per-patient conditions like M0/M2, so neither needs M3's recurring-
checkpoint machinery — the `checkXExists`-style plain existence check
remains the right default for them. If a future milestone *does* need a
recurring trigger, start from `m3-research.md` Decision 2's
count-of-existing-instances approach rather than re-deriving it from
scratch.
