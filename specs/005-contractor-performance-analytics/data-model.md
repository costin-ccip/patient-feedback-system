# Phase 1 Data Model: Contractor Performance Analytics

**Input**: `plan.md`, `research.md`

**Date**: 2026-09-29

## No CRM schema changes

Unlike every milestone (M0-M5), this feature makes no change to
`Milestone_Instances` or any other CRM module's fields or picklists. `Clinician`
already exists and is already populated on every M0/M2/M3/M4/M5 row. The one new
data movement is research.md Decision 1's confirmed sync of the existing
`Patients1.Name` field (via the existing `Milestone_Instances.Patient` lookup)
into Analytics — an Analytics *sync* question, not a CRM schema change; no CRM
field, picklist, or relationship is added or modified.

## Analytics workspace changes

All new objects live on or alongside the existing "Milestone Instances" table in
the same Analytics workspace every milestone's reporting already uses (workspace
`3251423000000083002`, per `m0-implementation-notes.md` §7).

### New sync: per-patient grouping key

**Resolved 2026-09-29 (research.md Decision 1).** One new column on the
"Milestone Instances" Analytics table, sourced from `Milestone_Instances.Patient`
(the existing CRM lookup) — specifically its linked `Patients1.Name` value, the
`P`+number pseudonym code the practice already uses in place of the patient's
real name (`Full_Name_PHI`, a separate field, is never touched). No new CRM
field, no hash, no new relationship — the lookup already surfaces this value.
Used inside Story 2's aggregate formulas (below) as the per-patient grouping
key; not needed by any other story. Because the value is already de-identified
by design, it doesn't require the same never-displayed treatment a real
identifier would (see research.md for the full reasoning), but this feature's
own reports still don't surface it by default — nothing in Stories 1-6 needs to
display it, only to group by it. This is the one object in this data model that
is not simply grouping/aggregating data Analytics already has — everything else
below reads only columns that already exist.

### New formula/aggregate columns

All of the following read existing columns only (Domain: Connection/Understanding/
Shared Direction/Fit Of Approach, Alliance Check-In Total, Therapist
Professionalism, Practice Experience: Scheduling/Communication, Practice
Experience: Billing, Looking Ahead: Likelihood To Recommend, Reason For Leaving —
all already present per `m0`/`m2`/`m3`/`m4`/`m5-data-model.md`), grouped or
aggregated by the existing `Clinician` column, plus `Milestone` where a
per-milestone breakdown applies.

**User Story 1 — Alliance (combined + per-milestone)**:
- Aggregate views over `Milestone Instances` filtered to `Milestone IN ('2 - Early
  Alliance Check', '3 - Periodic Consolidated', '4 - Discharge')`, grouped by
  `Clinician`: average of each Domain column and of Alliance Check-In Total.
- The same, additionally grouped by `Milestone`, for the per-milestone breakout.
- No new base formula columns needed — this is aggregation (report/pivot-level
  grouping and averaging), not new row-level parsing.

**User Story 2 — Professionalism / Practice Experience (patient-weighted)**:
- Requires the two-step aggregation research.md Decision 1 describes: (a) a
  per-patient sub-aggregate — average Therapist Professionalism / Scheduling-
  Communication / Billing per (`Patients1.Name` grouping key, `Milestone = '3 -
  Periodic Consolidated'`) — then (b) a per-contractor aggregate of those
  per-patient figures, grouped by `Clinician`. Likely built as a query table
  (Analytics' saved-query mechanism) for step (a), feeding a standard aggregate
  report for step (b) — exact mechanism to confirm against the live workspace's
  available features at build time.
- Depends on the grouping key above; cannot be built structurally correct
  without it (a plain `AVG()` grouped only by `Clinician` would silently be
  response-weighted, not patient-weighted, which is the behavior Costin explicitly
  asked this feature NOT to have).

**User Story 3 — Likelihood to Recommend**:
- Aggregate view over `Milestone Instances` filtered to `Milestone = '4 -
  Discharge'`, grouped by `Clinician`: average of `Looking Ahead: Likelihood To
  Recommend`. No patient-weighting concern — M4 is one-time-per-patient
  (`m4-data-model.md`), so response-weighted and patient-weighted are the same
  thing here.

**User Story 4 — Reason for Leaving breakdown**:
- Aggregate/pivot view over `Milestone Instances` filtered to `Milestone = '5B -
  Discontinuation, Email Fallback'`, grouped by `Clinician` and `Reason For
  Leaving`: count (and/or share within contractor) per category. Same "no
  patient-weighting concern" reasoning as Story 3 — M5 is one-time-per-patient.

### New reports

- **"Contractor Alliance Summary"** — Tabular/pivot, one row per `Clinician`,
  columns for the four Domain averages + Total (combined across M2/M3/M4).
- **"Contractor Alliance by Milestone"** — same shape, grouped additionally by
  `Milestone`, so each contractor's M2/M3/M4 figures sit side by side.
- **"Contractor Professionalism & Practice Experience"** — one row per
  `Clinician`, columns for the three patient-weighted M3 averages.
- **"Contractor Likelihood to Recommend"** — one row per `Clinician`, one average
  column.
- **"Contractor Reason for Leaving Breakdown"** — grouped bar or pivot, `Clinician`
  × `Reason For Leaving`, count and/or share.
- No `Patient`/identity column on any of the above — same de-identification
  discipline as every existing milestone report. `Clinician` is the only
  person-adjacent value ever displayed, and it identifies the contractor, not a
  patient (consistent with every existing milestone report already showing
  `Clinician` in its sample-data walkthroughs, e.g. `m0-implementation-notes.md`
  §8).

### New dashboard

- **"Contractor Performance"** — new dashboard (built via "Create New Dashboards,"
  not "Save As," same convention every prior milestone dashboard followed),
  bundling the five reports above, with each panel's contractor rows linking
  (research.md Decision 4) to the corresponding existing M2/M3/M4/M5 milestone-
  level report, pre-filtered to that contractor (User Story 5).
- Because every report above groups by `Clinician` as a field rather than
  enumerating contractor names, a new `Clinician` picklist value with qualifying
  data appears in every panel automatically — no new panel, report, or dashboard
  edit required per contractor (User Story 6; research.md Decision 5).

## Key Entities (restated from spec.md, with data-model detail)

- **Contractor (Clinician)**: existing `Milestone_Instances.Clinician` picklist
  value. This feature's sole grouping dimension across all five new reports; no
  new entity introduced.
- **Contractor Performance View**: not a stored entity — a set of Analytics
  reports/dashboard panels computed at query time from existing
  `Milestone_Instances` rows, grouped by `Clinician` (and, for Story 2, by the
  synced `Patients1.Name` patient-grouping key). Nothing about a contractor's
  performance is persisted anywhere new; it's recomputed from the same rows
  every existing milestone report already reads.

## Out of scope

Same list as `research.md`'s "Out of scope for this feature" section — not
repeated here.
