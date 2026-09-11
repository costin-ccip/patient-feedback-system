# Implementation Plan: M1 - Baseline Intake Survey (Milestone 1)

> **RETIRED (2026-09-11)**: Milestone 1 has been eliminated per an operations-lead
> decision following a review of clinical/EHR overlap — see `spec.md`'s fifth-pass
> revision note and Assumptions. This file is kept as the historical record of what
> was planned and built, not deleted; it no longer reflects the live system. The
> corresponding Zoho objects (flows, form, custom function) are being decommissioned
> per the same decision — see `m1-implementation-notes.md`'s own RETIRED note for
> their disposition.

**Branch**: `001-feedback-collection-pipeline`

**Date**: 2026-09-06

**Spec**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1 & 2,
scoped to the session-1/intake trigger)

**Note**: PROSPECTIVE. Unlike M0, this plan was written before implementation,
per the constitution's Rollout Workflow requirement. See `m0-implementation-notes.md`
§10 and `m0-plan.md`/`m0-tasks.md` for why M0 didn't get this and how it was
backfilled.

## Summary

M1 fires automatically when a patient's `Session Count` (CRM Patients module,
already auto-incremented by an existing payment-triggered workflow) reaches 1.
A new Zoho Flow issues a single-use token and emails a link to the **Cape
Clarity Wellbeing Check-In** — a 5-domain, 0-10-slider custom survey — via the
same de-identified Zoho Form + Flow write-back pattern M0 validated. This is
the first of three milestones (M1, M3, M4) that reuse this exact instrument,
so M1's build should produce reusable infrastructure, not a one-off.

## Technical Context

**Language/Runtime**: Zoho Deluge, Zoho Analytics formula language (same stack
as M0).

**Primary Dependencies**: Zoho CRM (`Patients` module `Session Count` field —
existing; `Milestone_Instances` module — existing, from M0), Zoho Flow
(new automatic trigger flow + reused/extended write-back function pattern),
Zoho Forms (new Wellbeing Check-In form), Zoho Mail, Zoho Analytics.

**Storage**: `Milestone_Instances` remains the system of record. **Decided**
(2026-09-06, Costin): reuse M0's delimited-blob pattern (`Response_Data`,
parsed by Analytics formulas) rather than dedicated numeric fields, to stay
consistent with M0 and conserve CRM field budget for M2/M5. See Complexity
Tracking below for the tradeoff this accepts.

**Testing**: Same as M0 — manual + Zoho CRM MCP tool-driven sample data, no
automated test suite.

**Target Platform**: Same Zoho One stack.

**Project Type**: Low-code/no-code automation (same as M0).

**Performance Goals**: N/A at current volume.

**Constraints**: Constitution Principles I (De-Identification by Design), II
(Single Rejoin Point), and III (Automated Milestone Triggers) — this
milestone is the first real test of III as an unambiguous automatic CRM-field
trigger, unlike M0's documented manual fallback.

**Scale/Scope**: Single milestone build, but the survey instrument and its
storage/scoring pattern must be designed for reuse at M3 and M4 — scope is
narrower for triggering, wider for the survey/storage design.

## Constitution Check

*Principle names/numbers below match the repository's authoritative
constitution (v1.2.1). This plan was originally checked against a divergent,
non-authoritative local copy — see `CLAUDE.md` "Branch reconciliation" note.*

- **Principle I (De-Identification by Design)**: Design carries this forward
  unchanged from M0 — Wellbeing Check-In Form gets a hidden token field only,
  no name/email/phone.
- **Principle II (Single Rejoin Point)**: `Session Count` already lives in CRM
  (Patients module) and M1 reads it without duplicating it elsewhere; the
  token-to-Patient rejoin happens only in `Milestone_Instances`, same as M0.
- **Principle III (Automated Milestone Triggers)**: PASS — this is the
  milestone that actually delivers on this principle unambiguously, since
  `Session_Count == 1` is a real CRM field crossing a threshold (unlike M0's
  manual fallback case, which is a genuinely open question against this
  principle — see `m0-plan.md`).
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: PASS — M1
  builds no dashboard and grants contractors no access to feedback data,
  consistent with the constitution's zero-contractor-access framing for v1.
- **Principle V (BAA-Gated Adoption)**: Same open gate as M0 —
  `TODO(BAA_SCHEDULE)` still unresolved. M1 should stay in test/sample-data
  mode until that's resolved, same as M0.
- **Principle VI (Internal Use Only — Never a Testimonial or Marketing
  Pipeline)**: PASS by construction, same as M0.
- **Note — Clinical Safety Flag Rules (spec.md, User Story 4 / FR-010–013)**:
  the ratified spec.md includes a fully specified set of numeric flag rules
  for wellbeing readings (which apply to Milestones 1, 3, and 4) that this
  plan had not previously connected to. M1 produces the *first* wellbeing
  reading for a patient, so most of these rules can't evaluate yet (they need
  2-3 readings), consistent with `m1-research.md`'s existing scope boundary —
  but this is now grounded in ratified FRs rather than an informal,
  verbally-described item. See `m1-research.md` for the reconciled detail.
- **Compliance & Data Handling Requirements (purge of raw Zoho Forms
  entries)**: Same **unresolved gap** carried over from M0 (see `m0-plan.md`)
  — not re-solved by M1, and not M1's job to fix alone, but M1 adds a second
  flow that will have the identical gap if nothing changes. As with M0, the
  constitution itself only requires a "short, defined" window, not literally
  24 hours — that specific number is a working target, not yet a ratified
  value. Investigated a generic once-built fix (2026-09-06) and hit real
  technical blockers (no documented Zoho Forms delete-entry API; native
  Auto-Trash needs a plan upgrade and still leaves a 5-day recovery window).
  Logged as a known limitation to revisit — see `data-retention-purge.md`.

## Project Structure

### Documentation (this feature)

```text
specs/001-feedback-collection-pipeline/
├── spec.md
├── m0-plan.md / m0-tasks.md / m0-implementation-notes.md
├── m1-plan.md                 # this file
├── m1-research.md             # Phase 0 output
├── m1-data-model.md           # Phase 1 output
└── checklists/requirements.md
```

### "Source" (Zoho org)

```text
Customer Feedback System (Zoho Flow folder)
├── [existing] Subflow - Issue Feedback Token, M0 - Feedback Survey Write-back
├── [new] M1 - Session 1 Trigger              # watches Patients.Session Count == 1, issues token
├── [new] M1 - Wellbeing Check-In Write-back  # realtime Form-submission trigger + write-back function

Zoho CRM
├── Patients module: Session Count (existing, already automated)
└── Milestone_Instances module (existing) — Patient lookup populated (not Lead_Reference), Response_Data reused for the new blob format

Zoho Forms
└── [new] Cape Clarity Wellbeing Check-In (5 sliders + hidden token) — designed for reuse at M3/M4

Zoho Analytics
└── [extend] Milestone Instances table — new formula columns parsing the M1/M3/M4 blob format (see m1-data-model.md)
```

**Structure Decision**: Same configuration-only approach as M0, with the
storage decision now settled (see Complexity Tracking below and
`m1-data-model.md`).

## Complexity Tracking / Design Decision

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| How to store the 5 domain scores | **Delimited blob** (M0's `Response_Data` pattern), parsed by Analytics formulas — one blob format shared by M1/M3/M4 | Dedicated numeric fields (`Domain_Wellbeing`, `Domain_Coping`, `Domain_Relationships`, `Domain_Hope`, `Domain_Control`, `Domain_Total`) on `Milestone_Instances` |

**Decision (Costin, 2026-09-06)**: keep the blob pattern consistent with M0,
to conserve `Milestone_Instances`' CRM field budget for M2 and M5 (whose
question sets aren't designed yet).

**Tradeoff accepted**: M3's evaluation against the ratified Clinical Safety
Flag Rules (comparing a patient's current Wellbeing Check-In reading against
their prior one or two readings) will need to parse and compare
`Response_Data` blobs across multiple `Milestone_Instances` records per
patient, rather than doing direct CRM-field arithmetic. This is solvable
(Analytics can join/compare rows, or a Deluge function can do the comparison
at write-back time) but is real added complexity to flag now, before M3
planning starts, so it isn't a surprise later. Worth revisiting this call
specifically if M3's flagging logic turns out to be awkward against blobs —
migrating to dedicated fields later is possible but means a data migration
for any M1 instances already collected.
