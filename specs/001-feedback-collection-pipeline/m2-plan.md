# Implementation Plan: M2 - Early Alliance Check (Session 3) (Milestone 2)

**Branch**: `001-feedback-collection-pipeline`

**Date**: 2026-09-10

**Spec**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1, 2,
and 4 — the alliance half — scoped to the Session-3 trigger; User Story 6 /
FR-018 for reporting)

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement
(same discipline M1 followed — see `m1-plan.md`). This plan is written before
any Zoho implementation begins.

## Summary

M2 fires automatically when a patient's `Session_Count` (the same CRM field
M1 already watches) reaches 3. A new Zoho Flow issues a single-use token and
emails a link to the **Cape Clarity Alliance Check-In** — a 4-domain, 0-10
slider custom survey — via the same de-identified Zoho Form + Flow write-back
pattern M0/M1 validated. Unlike M1, M2's write-back function also evaluates
spec.md's Clinical Safety Flag Rules (Alliance half) in real time, since
those rules are absolute-threshold and can fire on this very first alliance
reading — no prior data point is needed, unlike the wellbeing rules M1
deferred to M3. M2 is therefore the first milestone to build real
clinical-safety-flag infrastructure (generic fields on `Milestone_Instances`,
reusable by M3/M4), not just the survey/trigger/reporting scaffolding M0/M1
already established.

## Technical Context

**Language/Runtime**: Zoho Deluge, Zoho Analytics formula language (same
stack as M0/M1).

**Primary Dependencies**: Zoho CRM (`Patients1.Session_Count` — existing,
already automated; `Milestone_Instances` — existing, extended with 2 new
fields), Zoho Flow (new trigger flow + new write-back flow, both mirroring
M1's pattern), Zoho Forms (new Alliance Check-In form), Zoho Mail, Zoho
Analytics.

**Storage**: `Milestone_Instances` remains the system of record. Reuses
M0/M1's delimited-blob pattern for the 4 survey answers (`Response_Data`).
**Revised 2026-09-10 (Costin)**: `Milestone_Instances` is at its CRM
custom-field cap, so the flag data is **not** new CRM fields — it's appended
as two more `---`-delimited segments on the existing `Response_Data` blob,
detected via two new Zoho Analytics formula columns instead (not subject to
the CRM field cap). See `m2-data-model.md` "Revision" section. No
`createFields` CRM call is part of this build.

**Testing**: Same as M0/M1 — manual + Zoho CRM MCP tool-driven sample data,
no automated test suite. No live/end-to-end test without Costin (standing
instruction) — this session builds and verifies structurally only.

**Target Platform**: Same Zoho One stack.

**Project Type**: Low-code/no-code automation (same as M0/M1).

**Performance Goals**: N/A at current volume.

**Constraints**: Constitution Principles I (De-Identification by Design), II
(Single Rejoin Point), III (Automated Milestone Triggers), and — newly load-bearing
for the first time — IV (Contractor Blindness to Own Raw Feedback) and the
FR-012 "no automated contractor notification" requirement, since M2 is the
first milestone to build a mechanism (the flag) that a less careful design
could easily leak to a contractor. See Constitution Check below.

**Scale/Scope**: Single milestone build; the Alliance Check-In survey and its
storage/scoring pattern must be designed for reuse at M3 (per
`m2-research.md`), same relationship the Wellbeing Check-In has to M1/M3/M4.

## Constitution Check

*Principle names/numbers match the repository's authoritative constitution
(v1.2.1).*

- **Principle I (De-Identification by Design)**: PASS — Alliance Check-In
  Form gets a hidden token field only, no name/email/phone, same as every
  prior form.
- **Principle II (Single Rejoin Point)**: PASS — `Session_Count` already
  lives in CRM and is read, not duplicated; token-to-Patient rejoin happens
  only in `Milestone_Instances`, same as M0/M1.
- **Principle III (Automated Milestone Triggers)**: PASS — `Session_Count ==
  3` is the same kind of real, system-detected CRM-field threshold M1 already
  validated (per `m1-plan.md`'s own PASS finding), not a manual fallback.
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: **This is the
  first milestone where this principle is directly load-bearing, not just
  trivially satisfied by "no dashboard exists yet."** M2 introduces the first
  real clinical-safety-flag data. Design response: the flag is stored inside
  `Milestone_Instances.Response_Data` (a module/field contractors have no
  access to, same as every other field in this system — confirmed per Cape
  Clarity's current tooling, only Liana holds a CRM/Analytics login),
  detected via Analytics formula columns, and surfaced only through a new
  Analytics report restricted the same way every other report in this
  workspace already is. No email, Flow action, or any other mechanism sends
  flag data toward a contractor. See `m2-research.md`'s reconciliation of the
  source Confluence page's stale "routes to the treating clinician" language
  against ratified FR-012. PASS, but flagged here as the principle actually
  being tested for the first time, not a formality.
- **Principle V (BAA-Gated Adoption)**: Same open gate as M0/M1 —
  `TODO(BAA_SCHEDULE)` still unresolved. M2 stays in test/sample-data mode
  until that's resolved.
- **Principle VI (Internal Use Only)**: PASS by construction, same as M0/M1
  — no connection to any public-facing or marketing tool anywhere in this
  design.
- **User Story 4 / FR-010–013 (Clinical Safety Flag)**: Newly **in scope**
  for M2's alliance half (see `m2-research.md` — this is a real scope
  addition versus M1, not carried-over infrastructure). The wellbeing half
  (FR-010) remains out of scope until M3, unchanged from M1's own finding.
- **User Story 6 / FR-018–019 (Baseline interpretive reporting)**: M2's
  reporting work is checked against this bar from the start (per the Rollout
  Workflow instruction), same as M1's T024 — see Phase 5 tasks. FR-019 is
  satisfied structurally: M2's baseline reporting (status/volume + domain
  distribution) is built as its own Analytics report/dashboard, not deferred
  pending FR-007's unified dashboard.
- **Compliance & Data Handling Requirements (purge of raw Zoho Forms
  entries)**: Same **unresolved gap** carried over from M0/M1 (`data-retention-purge.md`)
  — not solved by M2, and M2 adds a third flow with the identical gap. No new
  investigation needed; the existing decision (log as known limitation,
  revisit before go-live) stands.

## Project Structure

### Documentation (this feature)

```text
specs/001-feedback-collection-pipeline/
├── spec.md
├── m0-plan.md / m0-tasks.md / m0-implementation-notes.md
├── m1-plan.md / m1-research.md / m1-data-model.md / m1-tasks.md / m1-implementation-notes.md
├── m2-plan.md                 # this file
├── m2-research.md             # Phase 0 output
├── m2-data-model.md           # Phase 1 output
└── m2-tasks.md                # Phase 2 output (/speckit-tasks equivalent)
```

No `contracts/` or `quickstart.md` — same as M0/M1, this is a low-code Zoho
configuration project with no external/public API surface of its own to
contract against; the "interface" is the Zoho Form + email link, already
fully specified by the survey content and blob format above.

### "Source" (Zoho org)

```text
Customer Feedback System (Zoho Flow folder)
├── [existing] Subflow - Issue Feedback Token, M0/M1 flows
├── [new] M2 - Session 3 Trigger               # watches Patients1.Session_Count == 3, issues token
├── [new] M2 - Alliance Check-In Write-back    # realtime Form-submission trigger + write-back function (incl. flag evaluation)

Zoho CRM
├── Patients1 module: Session_Count (existing, already automated)
└── Milestone_Instances module (existing, NO new custom fields — at CRM field cap) —
    Patient lookup populated (not Lead_Reference);
    Response_Data reused for the 4-domain + 2-flag-segment blob format (see m2-data-model.md)

Zoho Forms
└── [new] Cape Clarity Alliance Check-In (4 sliders + hidden token) — designed for reuse at M3

Zoho Analytics
└── [extend] Milestone Instances table — new formula columns parsing the M2 blob format
    (4 domain columns, Total, plus 2 new flag-detection columns: Clinical Safety Flag,
    Flag Rule Triggered — these formula columns are how the flag becomes queryable,
    since Milestone_Instances itself gets no new CRM fields);
    [new] M2 Status Breakdown, M2 Alliance Check-In Total Distribution, M2 Flagged for Review reports;
    [new] "M2 - Early Alliance Check Feedback" dashboard bundling them (same shape as M0/M1's dashboards)
```

**Structure Decision**: Same configuration-only approach as M0/M1, with one
correction from this plan's first draft: `Milestone_Instances` gets **no**
new CRM fields (it's at its custom-field cap, per Costin 2026-09-10) — the
flag data lives in the `Response_Data` blob instead, same schema-light
approach the survey answers already use, with Analytics formula columns
(not CRM fields) doing the parsing/filtering work. See `m2-data-model.md`
"Revision" section.

## Complexity Tracking / Design Decisions

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| How to store the 4 domain scores | **Delimited blob** (M0/M1's `Response_Data` pattern) | Dedicated numeric fields per domain |
| Where to evaluate the Clinical Safety Flag Rules | **In the write-back Deluge function**, at submission time | In Analytics, as formula columns computed after CRM sync |
| How to store the flag result | **Two more segments appended to the `Response_Data` blob**, parsed by new Analytics formula columns | Two new dedicated CRM fields (`Clinical_Safety_Flag`, `Flag_Rule_Triggered`) — **rejected 2026-09-10**: `Milestone_Instances` is at its CRM custom-field cap (Costin), so this option isn't actually available, not merely dispreferred |
| How flag data reaches admins (FR-013) | **Milestone-scoped Analytics report** (same pattern as M1's User Story 6 reporting), not the unified FR-007 dashboard | Building FR-007's cross-milestone Admin Dashboard early, just to house this one flag view |

**Decision (flag evaluation location)**: computing the flag in Deluge rather
than Analytics keeps the "visible to admins" guarantee (FR-013) independent
of the CRM→Analytics sync ever running — the flag exists on the CRM record
itself the moment the patient submits, which is also the more defensible
reading of FR-010/011 ("the system MUST evaluate each ... reading against the
Clinical Safety Flag Rules ... whenever a rule is met") as an event-driven
requirement rather than a downstream reporting computation.

**Tradeoff accepted**: this makes M2's write-back function meaningfully more
complex than M0/M1's (real numeric parsing and conditional logic, not just
string concatenation) — flagged explicitly here since it's a genuine
precedent for this project's Deluge functions, not because the added
complexity is a problem in itself. Worth keeping in mind if M3/M4's wellbeing
trend rules turn out to need cross-record comparison that's awkward to do
inside a single write-back function call (per `m1-plan.md`'s own note that
trend evaluation "will need to parse and compare `Response_Data` blobs across
multiple `Milestone_Instances` records per patient") — that may end up as a
separate scheduled/triggered function rather than inline in the write-back
path, a decision for M3's own plan, not resolved here.
