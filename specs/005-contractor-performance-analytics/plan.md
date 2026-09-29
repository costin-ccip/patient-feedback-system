# Implementation Plan: Contractor Performance Analytics

**Branch**: `005-contractor-performance-analytics`

**Date**: 2026-09-29

**Spec**: `specs/005-contractor-performance-analytics/spec.md` (User Stories 1-6)

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement (same
discipline M2/M3/M4 followed) — written before any Zoho configuration for this
feature begins. Per standing instruction, this plan and its tasks stop at
"ready to build" — no live/end-to-end testing, and nothing gets switched on,
without Costin.

## Summary

This feature adds a contractor-grouped reporting layer on top of data the pipeline
already collects — it introduces no new trigger flow, survey form, or write-back
function. Every score/category this feature reports (Alliance domains and Total from
M2/M3/M4, Therapist Professionalism and Practice Experience from M3, Likelihood to
Recommend from M4, Reason for Leaving from M5) already exists as a formula column on
the "Milestone Instances" Analytics table, and every response row already carries a
`Clinician` value. What's new is entirely Zoho Analytics work: formula/aggregate
columns and reports that group existing data by `Clinician`, plus a dashboard to
present them, plus drill-down links into the existing per-milestone reports.

One design question this plan resolves before work starts (research.md Decision 1,
resolved 2026-09-29): Story 2's "one vote per patient" weighting (spec.md, resolved
2026-09-29) requires grouping a contractor's Milestone 3 responses by which patient
submitted them. Every milestone's Analytics work to date deliberately excluded the
`Milestone_Instances.Patient` lookup from the sync (`m1-implementation-notes.md`
§9.1), on the assumption that doing so would carry patient identity into Analytics —
but `Patients1` turns out to have two separate name-shaped fields, and the one the
lookup surfaces (`Name`, "Patient Name") is a deliberately de-identified `P`+number
code, not the real name (that's a separate field, `Full_Name_PHI`, which this
feature never touches). Costin confirmed this, and it was verified live via CRM
schema introspection the same session. This plan's approach — syncing the existing
`Patients1.Name` value (via the existing `Milestone_Instances.Patient` lookup, no
new CRM field, no hash) for use inside Story 2's aggregate formulas — is therefore
confirmed, not flagged.

## Technical Context

**Language/Runtime**: Zoho Analytics formula/aggregate-formula language only. No
Deluge, no Zoho Flow, no Zoho Forms changes — this is the first feature in the
pipeline that touches none of those three.

**Primary Dependencies**: Zoho Analytics (extends the existing "Milestone Instances"
table: new formula/aggregate columns, new reports, a new or extended dashboard) and
Zoho CRM as the source of the existing `Milestone_Instances.Patient` lookup /
`Patients1.Name` field synced per research.md Decision 1. No new Zoho product is
introduced.

**Storage**: No new CRM fields, no new `Milestone_Instances` schema. Per research.md
Decision 1, one additional column (the existing `Patients1.Name` value) syncs into
the existing Analytics workspace (not CRM) as the Story 2 patient-grouping key.

**Testing**: Same as every prior milestone — no automated test suite; structural
verification only (formulas checked against sample/seeded data via CRM MCP tool
calls, same discipline `m0-implementation-notes.md` §8 established). No live
end-to-end test without Costin, per standing instruction for this stream.

**Target Platform**: Same Zoho One stack (Zoho Analytics specifically).

**Project Type**: Low-code/no-code reporting configuration — narrower than prior
milestones, which also touched Flow/Forms/Mail.

**Performance Goals**: N/A at current volume (single-digit contractors, low
hundreds of responses at most).

**Constraints**: Constitution Principles I (De-Identification by Design), II
(Single Rejoin Point), IV (Contractor Blindness), and VII (Analytics Is Where
Derived Values Get Computed) — see Constitution Check below. Also spec.md's own
FR-009 (dashboard access remains admin-only) and this feature's own FR-009/FR-010.

**Scale/Scope**: One new (or extended) dashboard, grouped/filtered reporting only;
no change to any existing trigger, write-back, form, or CRM schema.

## Constitution Check

*Principle names/numbers match the repository's authoritative constitution (v1.3.0).*

- **Principle I (De-Identification by Design)**: PASS. This feature adds no new
  patient-facing collection surface at all (no new form, no new field patients
  see), so the principle's core concern — identity never reaching the collection
  tool — isn't touched. Research.md Decision 1's grouping key (the existing
  `Patients1.Name` P-code, synced via the existing `Milestone_Instances.Patient`
  lookup) carries no name, email, phone, or other directly identifying value —
  it's the same deliberately de-identified pseudonym the practice already relies
  on elsewhere — and this feature's own reports don't surface it as a visible
  column regardless, since nothing in Stories 1-6 needs to display it, only to
  group by it.
- **Principle II (Single Rejoin Point)**: PASS. Nothing in this feature creates a
  new place where a token and patient identity are matched — token-to-identity
  rejoin still happens only in `Milestone_Instances`/CRM, unchanged. Research.md
  Decision 1's grouping key is a de-identified code with no name/email/phone
  attached to it in Analytics at all; `Full_Name_PHI` (the real name) is a
  separate field this feature never touches, and this feature gives no dashboard
  viewer, contractor, or admin a way to resolve the grouping key back to a named
  patient from within Analytics.
- **Principle III (Automated Milestone Triggers)**: N/A — this feature adds no
  trigger of any kind; it reports on triggers/submissions that already fired under
  M2-M5's existing, unchanged trigger logic.
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: PASS — spec.md's
  FR-009 and this feature's own FR-009 both require the dashboard stay
  practice-admin-only. Nothing in this feature's design creates or implies
  contractor access; it's an extension of the existing admin-only Analytics
  workspace, not a new access surface.
- **Principle V (BAA-Gated Adoption)**: PASS, no new product. This feature is built
  entirely inside Zoho Analytics (plus the confirmed synced field, still within
  CRM/Analytics), both already in active use by the shipped pipeline — no new Zoho
  product is being wired in, so no new BAA confirmation is triggered.
- **Principle VI (Internal Use Only)**: PASS by construction — this is an
  admin-only internal reporting view, same as every existing dashboard; nothing
  connects it to any public-facing or marketing surface.
- **Principle VII (Analytics Is Where Derived Values Get Computed)**: PASS, and
  this feature is the clearest case yet for the principle's stated default. Every
  average and breakdown in Stories 1-4 is computed from data Analytics already has
  (or, for Decision 1's grouping key, from a plain synced field with no logic of
  its own) — no Deluge write-back function changes, no new conditional logic
  anywhere outside Analytics formula/aggregate columns. Story 2's per-patient
  weighting is exactly the kind of "compare across multiple stored records for one
  patient" case the constitution's own exception text anticipates (it names this
  scenario directly) — but per that same exception's design intent, the
  cross-record comparison itself still happens in Analytics (a per-patient
  sub-aggregate, then an aggregate of those), not in Deluge; only the *grouping
  key* enabling that comparison is a possible CRM-to-Analytics sync question
  (Decision 1), not a computation moved out of Analytics.
- **spec.md FR-007–009 (unified Admin Dashboard) relationship**: Not in scope,
  unchanged. This feature does not build or modify the still-unbuilt unified
  cross-milestone Admin Dashboard; it's a separate, narrower, contractor-grouped
  set of views, as spec.md 005's own intro note describes.

## Project Structure

### Documentation (this feature)

```text
specs/005-contractor-performance-analytics/
├── spec.md                    # Feature specification (User Stories 1-6)
├── checklists/
│   └── requirements.md        # Spec quality checklist (all items pass)
├── plan.md                    # This file (/speckit-plan output)
├── research.md                # Phase 0 output
├── data-model.md              # Phase 1 output
├── quickstart.md              # Phase 1 output — structural validation guide
└── tasks.md                   # Phase 2 output (/speckit-tasks)
```

No `contracts/` — same reasoning as every prior milestone's own plan (M0-M4): this
is a low-code Zoho configuration project with no external/public API surface. This
feature doesn't even add a new Zoho Form (no patient-facing surface at all), so it
has less of an interface footprint than any milestone built so far.

### "Source" (Zoho org)

```text
Zoho CRM
└── Milestone_Instances module (existing, unchanged schema) — Clinician picklist
    (existing since M0) becomes this feature's primary grouping dimension.
    Story 2's grouping-key source (research.md Decision 1) is likewise existing
    and unchanged: Milestone_Instances.Patient lookup → Patients1.Name — no new
    CRM field, only a new Analytics sync question.

Zoho Analytics
└── [extend] Milestone Instances table —
    [new] Per-contractor Alliance aggregate columns/reports — combined (M2+M3+M4)
      and per-milestone (M2-only, M3-only, M4-only), grouped by Clinician, over the
      existing Domain: Connection/Understanding/Shared Direction/Fit Of Approach and
      Alliance Check-In Total columns (User Story 1 / FR-001, FR-001a);
    [new] Per-contractor Professionalism/Practice Experience aggregate columns/
      reports, grouped by Clinician, patient-weighted per research.md Decision 1,
      over the existing Therapist Professionalism / Practice Experience: Scheduling-
      Communication / Practice Experience: Billing columns (User Story 2 / FR-002);
    [new] Per-contractor Likelihood-to-Recommend aggregate columns/reports, grouped
      by Clinician, over the existing Looking Ahead: Likelihood To Recommend column
      (User Story 3 / FR-003);
    [new] Per-contractor Reason-for-Leaving breakdown report, grouped by Clinician,
      over the existing Reason For Leaving column (User Story 4 / FR-004);
    [new] "Contractor Performance" dashboard bundling the four views above, with
      drill-down links to the existing M2/M3/M4/M5 milestone-level reports (User
      Story 5 / FR-007), built so a new Clinician picklist value appears in every
      view automatically (User Story 6 / FR-008) — see data-model.md for exactly
      how each report avoids being hand-built per contractor;
    [new, confirmed] a per-patient grouping-key column, synced from
      Milestone_Instances.Patient → Patients1.Name (the existing de-identified
      P+number code — no new CRM field, no hash), used inside the Story 2
      aggregate formulas (research.md Decision 1) — not added to any report's
      visible column list, since no story needs to display it.
```

## Complexity Tracking / Design Decisions

| Decision | Chosen | Rejected Alternative |
|---|---|---|
| Where this feature's logic lives | Entirely in Zoho Analytics (formula/aggregate columns, reports, dashboard) | Any Deluge/Flow computation — rejected per Principle VII; nothing here needs a value to exist before Analytics query time |
| Grouping dimension | The existing `Clinician` field on `Milestone_Instances` (already populated at issuance since the 2026-09-25 fix) | Deriving contractor from `Patients1.Assigned_Therapist` at report time — rejected: would require joining a second, currently-unsynced table into Analytics for no benefit, since `Clinician` already carries the same value at the response level |
| Story 2 per-patient weighting mechanism | Sync the existing `Patients1.Name` de-identified code as a grouping key (confirmed, research.md Decision 1) so Analytics can average per-patient first, then per-contractor | Sync a new hashed/internal ID instead — rejected once the existing field was confirmed safe and sufficient; leaving Story 2 as a raw per-response average (no patient de-duplication) — also rejected, this is exactly what Costin's resolution to the clarifying question ruled out |
| Story 1 vs. Story 2 weighting consistency | Story 1 (Alliance) stays response-weighted, not patient-weighted, per Costin's resolution — spec.md's Assumptions flag this asymmetry explicitly for revisit if it matters in practice | Applying the same patient-weighting to Story 1 for consistency — rejected for now: not what was asked, and M2/M4 don't recur per patient the way M3 does, so the asymmetry has less practical effect on the Alliance side |
| Dashboard placement | One new "Contractor Performance" dashboard, separate from the five existing per-milestone dashboards | Adding contractor-grouped panels onto each existing per-milestone dashboard instead — rejected: this feature's own comparative value (seeing one contractor across several milestones at once) is lost if the views stay scattered across five separate milestone-scoped dashboards |
| Drill-down mechanism (User Story 5) | Link from each Contractor Performance panel to the corresponding existing M2/M3/M4/M5 report, pre-filtered to that contractor | Building new, separate contractor-filtered detail reports duplicating the existing milestone-level ones — rejected as unnecessary duplication of reports that already exist and already carry the right de-identification discipline |
| New-contractor scalability (User Story 6) | Every report/column in this feature groups by `Clinician` generically (no contractor named in any formula or filter) | Building one panel/report per named contractor (e.g. a "Liana" panel, a "Deborah" panel) — rejected outright: this is exactly the rebuild-per-contractor problem Story 6 exists to prevent |

**Note for whoever builds this**: unlike M0-M5, this feature touches no Deluge, no
Flow, and no Forms — the entire build surface is the Zoho Analytics workspace.
Research.md Decision 1 (the grouping key for Story 2's patient-weighting) is
resolved — it reuses the existing `Patients1.Name` field via the existing lookup,
no new field or Costin sign-off needed before building it — so every item in this
plan can proceed directly from `Clinician` and columns that already exist.
