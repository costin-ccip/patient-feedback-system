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
pattern M0/M1 validated, and the write-back function itself is unchanged in
kind from M0/M1's (pure string concatenation into `Response_Data`, no new
arithmetic or branching). Unlike M1, M2 also evaluates spec.md's Clinical
Safety Flag Rules (Alliance half), since those rules are absolute-threshold
and can fire on this very first alliance reading — no prior data point is
needed, unlike the wellbeing rules M1 deferred to M3. **Revised 2026-09-10
(second correction, Liana)**: that evaluation happens entirely in Zoho
Analytics, as formula columns computed from the domain values already parsed
out of the blob — not in Deluge, and not stored anywhere in CRM. M2 is
therefore the first milestone to build real clinical-safety-flag reporting
infrastructure (new Analytics formula columns and a flagged-records report,
reusable by M3/M4), not new CRM schema or write-back complexity.

## Technical Context

**Language/Runtime**: Zoho Deluge, Zoho Analytics formula language (same
stack as M0/M1).

**Primary Dependencies**: Zoho CRM (`Patients1.Session_Count` — existing,
already automated; `Milestone_Instances` — existing, no schema changes), Zoho
Flow (new trigger flow + new write-back flow, both mirroring M1's pattern),
Zoho Forms (new Alliance Check-In form), Zoho Mail, Zoho Analytics.

**Storage**: `Milestone_Instances` remains the system of record. Reuses
M0/M1's delimited-blob pattern for the 4 survey answers (`Response_Data`),
identical in shape to M1's blob — nothing appended for the flag. **Revised
2026-09-10 (Costin)**: `Milestone_Instances` is at its CRM custom-field cap,
so the flag data is **not** new CRM fields. **Revised again 2026-09-10
(second correction, Liana)**: the flag data also isn't appended to the
`Response_Data` blob — it's computed entirely as two new Zoho Analytics
formula columns, derived from the same 4 domain values Analytics already
parses out of the blob for the Total column, and never stored in CRM at all.
See `m2-data-model.md` "Revision 2" section. No `createFields` CRM call is
part of this build, and no Deluge-side flag logic either.

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
(v1.3.0).*

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
  real clinical-safety-flag data. Design response, **revised 2026-09-10
  (second correction)**: the flag is computed entirely by two new Zoho
  Analytics formula columns from data already inside
  `Milestone_Instances.Response_Data` (a module/field contractors have no
  access to, same as every other field in this system — confirmed per Cape
  Clarity's current tooling, only Liana holds a CRM/Analytics login) — the
  flag value itself is never stored anywhere, only computed at query time —
  and surfaced only through a new Analytics report restricted the same way
  every other report in this workspace already is. No email, Flow action, or
  any other mechanism sends flag data toward a contractor. See
  `m2-research.md`'s reconciliation of the source Confluence page's stale
  "routes to the treating clinician" language against ratified FR-012. PASS,
  but flagged here as the principle actually being tested for the first time,
  not a formality.
- **Principle V (BAA-Gated Adoption)**: Same open gate as M0/M1 —
  `TODO(BAA_SCHEDULE)` still unresolved. M2 stays in test/sample-data mode
  until that's resolved.
- **Principle VI (Internal Use Only)**: PASS by construction, same as M0/M1
  — no connection to any public-facing or marketing tool anywhere in this
  design.
- **Principle VII (Analytics Is Where Derived Values Get Computed)**: PASS —
  and the direct source of this principle. Liana's 2026-09-10 challenge to
  M2's original design (flag computed in the Deluge write-back function,
  stored in the blob) is what prompted this principle's addition to the
  constitution; M2's own redesign — flag computed entirely by Analytics
  formula columns, `submitAllianceCheckInResponse` doing pure string
  concatenation with no arithmetic or conditional logic — is the first
  build to conform to it. No documented exception is claimed for M2; the
  one plausible exception noted in the constitution (a trend-based rule
  needing cross-record comparison) does not apply here, since the Alliance
  Check-In flag rule is evaluable on a single reading.
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
├── [new] M2 - Alliance Check-In Write-back    # realtime Form-submission trigger + write-back function (plain string concatenation, same shape as M0/M1 — no flag evaluation here, see Zoho Analytics below)

Zoho CRM
├── Patients1 module: Session_Count (existing, already automated)
└── Milestone_Instances module (existing, NO new custom fields — at CRM field cap) —
    Patient lookup populated (not Lead_Reference);
    Response_Data reused for the 4-domain blob format, same shape as M1's —
    no flag segments (see m2-data-model.md "Revision 2")

Zoho Forms
└── [new] Cape Clarity Alliance Check-In (4 sliders + hidden token) — designed for reuse at M3

Zoho Analytics
└── [extend] Milestone Instances table — new formula columns parsing the M2 blob format
    (4 domain columns, Total) plus 2 new flag-detection columns: Clinical Safety Flag,
    Flag Rule Triggered — computed directly from the Total/Domain columns via IF/AND/OR/
    CONCATENATE, not parsed from the blob (there's nothing flag-related in the blob to
    parse — revised 2026-09-10, second correction);
    [new] M2 Status Breakdown, M2 Alliance Check-In Total Distribution, M2 Flagged for Review reports;
    [new] "M2 - Early Alliance Check Feedback" dashboard bundling them (same shape as M0/M1's dashboards)
```

**Structure Decision**: Same configuration-only approach as M0/M1, with two
corrections from this plan's first draft. First (Costin, 2026-09-10):
`Milestone_Instances` gets **no** new CRM fields — it's at its custom-field
cap. Second (Liana, same day): the flag data doesn't live in the
`Response_Data` blob either — it isn't stored anywhere in CRM at all. Both
the total and the flag are computed live by Analytics formula columns from
the 4 domain values the blob already carries, the same schema-light approach
the survey answers already used, now extended to the flag as well. See
`m2-data-model.md` "Revision" and "Revision 2" sections.

## Complexity Tracking / Design Decisions

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| How to store the 4 domain scores | **Delimited blob** (M0/M1's `Response_Data` pattern) | Dedicated numeric fields per domain |
| Where to evaluate the Clinical Safety Flag Rules | **In Analytics, as formula columns** computed from the already-parsed Domain/Total columns — **revised 2026-09-10 (second correction, Liana)** | In the write-back Deluge function, at submission time — this session's first-draft choice, reversed once challenged (see decision note below) |
| How to store the flag result | **Not stored at all** — computed live by the two Analytics formula columns above, revised 2026-09-10 (second correction) | (a) Two new dedicated CRM fields (`Clinical_Safety_Flag`, `Flag_Rule_Triggered`) — **rejected 2026-09-10**: `Milestone_Instances` is at its CRM custom-field cap (Costin), not actually available. (b) Two more segments appended to the `Response_Data` blob, computed in Deluge — this session's second design, itself **rejected 2026-09-10** once (a) had already been reconsidered: duplicated logic Analytics already needed for the Total column |
| How flag data reaches admins (FR-013) | **Milestone-scoped Analytics report** (same pattern as M1's User Story 6 reporting), not the unified FR-007 dashboard | Building FR-007's cross-milestone Admin Dashboard early, just to house this one flag view |

**Decision (flag evaluation location), revised 2026-09-10 (second
correction, Liana)**: the original reasoning for evaluating the flag in
Deluge — keeping the "visible to admins" guarantee independent of the
CRM→Analytics sync ever running — didn't hold up. FR-013 only requires the
flag be visible within an Analytics-based dashboard view either way (see
`m2-research.md`'s FR-013 decision), and FR-012 bans any automated,
time-critical action keyed off the flag, so nothing in this system actually
needs the flag to exist at the instant of submission rather than at report
query time. Meanwhile Deluge-side evaluation duplicated logic Analytics
already had to do for the 0-40 total (deliberately never stored, precisely
because it's fully derivable from the blob) — the flag conditions use that
same total plus the same four domain values, no new information. Computing
the flag in Analytics instead is not just simpler; it's also more correct
under a future rule change, since a formula-column edit recalculates every
existing response retroactively, where a value frozen into old records at
Deluge submission time would not.

**Precedent for M3/M4**: default to Analytics formula columns for derived
values (like flags), not Deluge, even where a field-budget workaround (like
the blob) would technically make Deluge-side storage and computation
possible. The one place this default may not hold: M3/M4's wellbeing trend
rules need cross-record comparison (per `m1-plan.md`'s own note that trend
evaluation "will need to parse and compare `Response_Data` blobs across
multiple `Milestone_Instances` records per patient") — that may turn out to
be awkward to express as a single Analytics formula column, in which case a
separate scheduled/triggered function might still be justified. That's a
decision for M3's own plan, to be argued from "why does this case need
Deluge" rather than assumed by default, not resolved here.
