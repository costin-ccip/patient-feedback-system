# Phase 0 Research: Contractor Performance Analytics

**Input**: `spec.md` (User Stories 1-6), `constitution.md` (v1.3.0),
`m0-implementation-notes.md` §9.1 (Patient lookup never synced to Analytics),
`m2-implementation-notes.md` §12 / `m3-implementation-notes.md` §8 /
`m4-implementation-notes.md` §8 / `m5-implementation-notes.md` §7 (`Clinician`
now derived from `Assigned_Therapist` at issuance)

**Date**: 2026-09-29

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement, written
before any Zoho Analytics configuration begins for this feature.

## Decision 1: How to compute Story 2's "one vote per patient" weighting — RESOLVED 2026-09-29

**The original tension**: spec.md's Story 2 requires averaging each patient's own
Milestone 3 responses together first, then averaging those per-patient figures
across a contractor's caseload. That requires *some* per-patient grouping key
inside Analytics. Every milestone's Analytics work to date has deliberately
excluded the `Milestone_Instances.Patient` lookup from the sync entirely
(`m1-implementation-notes.md` §9.1), on the assumption that syncing it would carry
the linked patient's identity (name) into Analytics. `Token` doesn't solve this
either — it's per-response, not per-patient.

**Resolved by Costin, with live schema confirmation the same session**: `Patients1`
turns out to have two separate name-shaped fields, not one, and the earlier
assumption behind the exclusion above was based on an inaccurate read of them.
Confirmed live via `getFields` (Zoho CRM MCP, schema-only — no records read, per
the standing access constraint):

| Field | api_name | Label | Content |
|---|---|---|---|
| `Patients1`'s standard/primary field | `Name` | "Patient Name" | A pseudonymous code (`P` + patient number) — by explicit practice design, this field is never populated with a real name |
| `Patients1`'s custom field | `Full_Name_PHI` | "Full Name (PHI)" | The patient's actual identified name |

`Milestone_Instances.Patient` (the existing lookup, `api_name = "Patient"`) already
points at the same `Patients1` record and, by Zoho CRM's standard lookup display
behavior, already surfaces that record's primary `Name` field — i.e. the P-code —
wherever the lookup is shown, including on the Milestone Instance record layout
Costin checked directly. `Full_Name_PHI` is a separate field the lookup does not
surface and this feature never touches.

**Correction to prior documentation**: `m5-data-model.md` (its "Field inventory"
table) and `m5-research.md` (Decision 4) both characterized `Patients1.Name`
("Patient Name") as holding the patient's full real name, on par with
`Full_Name_PHI`, when reasoning about the M5 `[First Name]` email-personalization
question. That characterization was inaccurate — confirmed now, not at the time —
though it happens not to have mattered for M5's own outcome: a P-code wouldn't have
worked as a friendly email greeting either, so M5's actual fallback (drop
`[First Name]`, open with "Hi there,") remains the right call regardless. A short
correction note has been added to `m5-implementation-notes.md` for the record, per
CLAUDE.md's documentation-accuracy convention.

**Decision**: sync `Patients1.Name` (via the existing `Milestone_Instances.Patient`
lookup — no new CRM field, no hash, no new relationship) into the "Milestone
Instances" Analytics table as a new column, used as the per-patient grouping key
for Story 2's two-step aggregation (per-patient average, then per-contractor
average of those). Because the field is a deliberately de-identified code rather
than identity, it does not need the same never-displayed treatment a real
identifier would — it can appear in a report column if that's ever useful for
troubleshooting (the same way `Token`, an equally opaque per-response code, already
appears on every existing milestone report) — but this feature's own reports
(`data-model.md`) still don't surface it by default, since nothing in Stories 1-6
needs to display it, only to group by it.

**Options considered and rejected**, for the record:

1. *Sync a new hashed/internal ID instead of reusing `Patients1.Name`* — superseded
   by the above once it became clear a safe, already-existing field does the same
   job with no new field and no ambiguity about what it contains.
2. *Compute the per-patient pre-aggregation in Deluge instead* — still rejected,
   same reasoning as before: Principle VII's default is Analytics, and a
   Deluge-computed figure would freeze at submission time rather than recalculating
   if the weighting logic ever changes.
3. *Skip patient-weighting and average every response equally* — still rejected;
   Costin was asked this specific question in the clarifying round and gave a
   specific answer.
4. *Do the weighting outside Analytics entirely* (periodic export/re-import) — still
   rejected as needless infrastructure Analytics's own aggregate/query-table
   features already handle.

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
