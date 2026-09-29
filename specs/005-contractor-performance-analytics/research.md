# Phase 0 Research: Contractor Performance Analytics

**Input**: `spec.md` (User Stories 1-6), `constitution.md` (v1.3.0),
`m0-implementation-notes.md` §9.1 (Patient lookup never synced to Analytics),
`m2-implementation-notes.md` §12 / `m3-implementation-notes.md` §8 /
`m4-implementation-notes.md` §8 / `m5-implementation-notes.md` §7 (`Clinician`
now derived from `Assigned_Therapist` at issuance)

**Date**: 2026-09-29

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement, written
before any Zoho Analytics configuration begins for this feature.

## Decision 1: How to compute Story 2's "one vote per patient" weighting without syncing patient identity into Analytics

**The tension**: spec.md's Story 2 (resolved via clarifying question, 2026-09-29)
requires averaging each patient's own Milestone 3 responses together first, then
averaging those per-patient figures across a contractor's caseload. That requires
*some* per-patient grouping key inside Analytics. But every milestone's Analytics
work to date has deliberately excluded the `Milestone_Instances.Patient` lookup from
the sync entirely (`m1-implementation-notes.md` §9.1) — not merely hidden it from
reports, excluded it from the workspace — specifically so the reporting layer can
never trace a score back to an individual patient. `Token` doesn't solve this either:
it's per-response, not per-patient (a patient's session-8 and session-16 M3
responses have two different tokens), so it can't serve as the grouping key.

**Options considered**:

1. **Sync a new internal grouping key** (this plan's recommendation) — add one new
   column to the Analytics sync that carries something derived from
   `Milestone_Instances.Patient` (e.g. the CRM record ID itself, or a one-way hash
   of it) purely so formulas can group rows by "same patient," and never surface
   that column in any report's visible column list or any dashboard panel. It
   carries no name, email, phone, or other directly identifying value — a bare
   record ID or hash means nothing to a dashboard viewer without separate CRM
   access, and even then, dashboard access itself stays admin-only, not
   contractor-facing (Principle IV, FR-009). This is a narrower exposure than
   syncing `Patient` itself (which would also carry the patient's name via any
   lookup display value), but it is still new information entering Analytics that
   wasn't there before, so it's flagged for Costin's explicit confirmation rather
   than built on this plan's own authority — same treatment every prior
   milestone's own open schema/field questions got (e.g. M4/M5's trigger-field
   mapping, M3's TTL question).
2. **Compute the per-patient pre-aggregation in Deluge instead**, writing a
   precomputed per-patient average back to CRM or the blob at submission time —
   rejected per Principle VII's default (Analytics computes derived values, not
   Deluge) and its own stated reasoning: a Deluge-computed figure freezes at
   submission time and wouldn't recalculate if the weighting logic ever changed,
   unlike an Analytics formula/aggregate column. The constitution's own exception
   clause (a rule that compares across multiple records for one patient) is written
   to still keep the *comparison* in Analytics where practical — it isn't a license
   to move the whole computation to Deluge just because it's patient-scoped.
3. **Skip patient-weighting and average every response equally**, i.e. quietly
   under-deliver on the resolved clarifying answer — rejected outright; Costin was
   asked this specific question and gave a specific answer, so silently reverting to
   the simpler behavior isn't a real option.
4. **Do the weighting outside Analytics entirely** (e.g. a periodic export/script
   that recomputes contractor averages and re-imports them) — rejected as needless
   extra infrastructure for a computation Analytics's own aggregate-formula/query-
   table features are built to do, and it would reintroduce exactly the
   "frozen until re-run" staleness problem option 2 has, without even Deluge's
   excuse of running automatically on submission.

**Recommendation**: Option 1, syncing a minimal internal grouping key (likely just
the CRM record ID Analytics can already reference via `Milestone_Instances`' own
row identity on the `Patient` lookup field — to be confirmed against what the Zoho
Analytics sync configuration actually exposes when this is built) used only inside
Story 2's aggregate formulas, never added to any report or dashboard's visible
columns. **Flagged for Costin's explicit confirmation before this is built** — this
is the one decision in this plan that changes what data leaves CRM for Analytics,
even in a de-identified form, and every prior milestone's own precedent is to get
that kind of change confirmed rather than assumed.

## Decision 2: Combined vs. per-milestone Alliance reporting (Story 1)

**Resolved via clarifying question, 2026-09-29**: both. A contractor's Alliance
domain/Total averages are shown combined (M2+M3+M4 pooled) and broken out
separately per milestone. Mechanically, this needs two sets of aggregate columns/
reports per contractor: one grouped by `Clinician` alone (combined), and one
grouped by `Clinician` + `Milestone` (per-milestone) — both straightforward
`AVG()`-style aggregate formulas over the existing Domain/Total columns, no new
parsing logic, since the four Domain columns and the Total already exist and
already carry consistent values across M2/M3/M4 (`m3-data-model.md`/
`m4-data-model.md` both confirm the domain columns are reused byte-for-byte across
milestones specifically so they parse identically).

## Decision 3: Reason-for-Leaving breakdown shape (Story 4)

M5's `Reason For Leaving` column is already categorical (8 fixed options, per
`m5-data-model.md`). A per-contractor breakdown is a count (or share) per category,
grouped by `Clinician` — the same `Dimension [Actual(D)] / Treat as Text` pattern
`m2-implementation-notes.md` §9.5 and `m5-data-model.md`'s own "M5 Reason For
Leaving Distribution" report already established, just with an added `Clinician`
grouping/filter dimension rather than a new visualization type.

## Decision 4: Drill-down mechanism (Story 5)

Zoho Analytics reports support linking from one view to another with an inherited
or passed filter (e.g. a table/chart configured so selecting or clicking a
contractor's row opens a pre-filtered version of the relevant milestone-level
report). The exact mechanism (URL-based report link with a `Clinician` filter
parameter, vs. a native "drill-down" configuration on the chart/table itself) is a
build-time detail to confirm against the live workspace, per every prior
milestone's own "confirm against the actual UI, don't assume" discipline — not
resolved here. Either way, the destination is always one of the *existing* M2/M3/
M4/M5 milestone-level reports (already de-identified), filtered to one contractor,
never a new patient-level view.

## Decision 5: New-contractor scalability mechanism (Story 6)

Every aggregate column/report this feature builds groups or filters by `Clinician`
as a field, not by a hardcoded list of contractor names. Zoho Analytics aggregate
formulas and pivot-style reports grouped by a picklist field automatically include
every value present in the underlying data — confirmed by existing precedent: no
milestone dashboard has ever needed to be told the specific list of `Milestone`
picklist values it covers; each report's `Milestone` filter names only the values
*that specific report* is scoped to, and a widened filter (as M2→M3→M4's shared
Clinical Safety Flag column shows, `m3-data-model.md`/`m4-data-model.md`) is a
one-line formula edit, not a structural rebuild. The same principle applies here:
grouping by `Clinician` needs no enumerated contractor list at all, so a fourth or
fifth contractor value appearing on new `Milestone_Instances` rows flows through
automatically. The one manual step that already exists independent of this feature
— adding the new contractor's name as a `Clinician`/`Assigned_Therapist` picklist
value in the first place — is unchanged by this feature and already how the
practice has onboarded its second and third contractor.

## Out of scope for this feature

Same boundary spec.md's own Assumptions section draws: per-contractor response
rate, trend-over-time, cross-contractor ranking/benchmarking, clinical-safety-flag
rate per contractor, caseload/volume context (would require syncing `Patients1`
aggregate data, not just `Milestone_Instances`), small-sample suppression/caveats,
and any export or threshold-alerting capability. None of these are touched by this
plan; each would need its own spec if wanted later.
