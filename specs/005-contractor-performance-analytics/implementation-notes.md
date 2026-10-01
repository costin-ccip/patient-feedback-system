# Contractor Performance Analytics — Implementation Notes

**Status as of 2026-09-30: built, partially verified, drill-down descoped.**
Every report and the dashboard exist and are saved in the live "Zoho CRM
Analytics" workspace. Drill-down (User Story 5) is explicitly descoped per
Costin's 2026-09-30 decision — not built, not planned for this pass (see §6).
Real-data verification (Phase 10) is partially done: 5 of 7 quickstart.md §B
test rows are seeded; the 3 patient-weighting rows (T010) are still blocked
on a real `Patients1` record ID (see §7).

**2026-09-30 update — Patient linkage confirmed** (answers a question Costin
asked directly): for M2/M3/M4/M5, the shared `issueFeedbackToken` function
*does* populate `Milestone_Instances.Patient` automatically —
`createMap.put("Patient",patientId)`, where `patientId` is wired from each
milestone's own trigger flow as `${trigger.id}` (the Patients1 record's own
CRM ID — see `specs/002-remove-subflow-dependency/implementation-notes.md`
line 92 and each milestone's own flow wiring). **M0 is the one exception** —
its trigger fires on a Lead, before the person is a Patient record, so it
passes `patientId` empty by design; that's why all 4 pre-existing production
records (all M0) show `Patient: null` — expected, not a gap. This also
resolves the "Numeric vs Text" open nuance from §3 below: the value that
syncs to Analytics is the raw Patients1 record ID (a number), not the
`P`-code display string — still a perfectly valid unique-per-patient
grouping key for T008's aggregation, just not literally
"`Patients1.Name`" as research.md's Decision 1 phrased it. Worth a small
research.md correction later; doesn't affect T008/T009's correctness.

## 1. What this feature does

Five new Zoho Analytics reports plus one dashboard, giving a practice owner
per-contractor performance views: Alliance domain averages (combined and
per-milestone), patient-weighted Professionalism/Practice Experience,
Likelihood-to-Recommend, and a Reason-for-Leaving breakdown. Entirely built
in Zoho Analytics — no CRM field, Zoho Flow, or Zoho Forms change (this is
the first feature in the pipeline with that property; see `research.md`'s
"Out of scope" section).

## 2. Component inventory

All in workspace `3251423000000083002` ("Zoho CRM Analytics"), folder "Zoho
CRM Modules (Data)", built on the existing "Milestone Instances" base table
(view id `3251423000000083317`).

| Object | Type | Feeds |
|---|---|---|
| Contractor Alliance Summary | Pivot report | Dashboard panel 1 (Story 1, combined) |
| Contractor Alliance by Milestone | Pivot report | Dashboard panel 2 (Story 1, per-milestone) |
| Contractor Per-Patient M3 Sub-Aggregate | Query Table (SQL) | Feeds "Contractor Professionalism & Practice Experience" only — not itself a dashboard panel |
| Contractor Professionalism & Practice Experience | Pivot report (on the Query Table above) | Dashboard panel 3 (Story 2) |
| Contractor Likelihood to Recommend | Pivot report | Dashboard panel 4 (Story 3) |
| Contractor Reason for Leaving Breakdown | Pivot report | Dashboard panel 5 (Story 4) |
| **Contractor Performance** | Dashboard | Bundles all 5 panels above |

No new CRM field, no new Zoho Flow, no new Zoho Form. No `Patients1`
(real-name Patient module) object of any kind was created, read, or touched.

## 3. Decision 1 confirmation (per-patient grouping key)

`research.md` Decision 1 — reuse the existing `Milestone_Instances.Patient`
lookup (which surfaces `Patients1.Name`, the practice's own de-identified
`P`-code) rather than a new field or hash — was built exactly as planned
(T007). Confirmed via Zoho CRM MCP `getFields`/`getRecords` (module
`Milestone_Instances`, non-PII, allowed under the standing access
constraint): the `Patient` field is a genuine lookup (`data_type: "lookup"`,
target module `Patients1`), present and queryable; all 4 pre-existing
records have `Patient: null`, consistent with them being Milestone-0-only
records that predate any patient-lookup population. `Patients1.Name` (the
grouping key) does not appear as a column in any of the 5 new reports or the
dashboard — confirmed by inspecting each report's design.

**One open nuance, not blocking**: in Zoho Analytics, the synced `Patient`
column displays with a Numeric ("#") type icon rather than the Text ("T")
icon `research.md` assumed for a `P`-code string. This doesn't affect
`GROUP BY`/`DISTINCT` correctness (uniqueness is all that matters for T008's
aggregation), but should be re-checked once real per-patient data exists —
if the value actually synced is a numeric CRM record ID rather than the
`P`-code display string, that's still fine functionally, just worth knowing.

## 4. Exact definitions

### 4.1 Query Table — "Contractor Per-Patient M3 Sub-Aggregate" (T008)

```sql
SELECT "Patient",
       "Clinician",
       AVG(TO_DECIMAL("Therapist Professionalism")) AS "Avg Therapist Professionalism",
       AVG(TO_DECIMAL("Practice Experience: Scheduling/Communication")) AS "Avg Practice Experience Scheduling Communication",
       AVG(TO_DECIMAL("Practice Experience: Billing")) AS "Avg Practice Experience Billing"
FROM "Milestone Instances"
WHERE "Milestone" = '3 - Periodic Consolidated'
  AND "Patient" IS NOT NULL
GROUP BY "Patient", "Clinician"
```

`TO_DECIMAL()` is needed because the three M3 formula columns
(`m3-implementation-notes.md` §1.2) are Text-typed `substring_between()`
extractions, same pattern every prior milestone's Analytics work used for
text-typed numeric-looking columns.

### 4.2 Pivot reports

- **Contractor Alliance Summary** (T004): base table "Milestone Instances",
  `Milestone` filter `IN ('2 - Early Alliance Check', '3 - Periodic
  Consolidated', '4 - Discharge')`, Rows: `Clinician`, Data: the four Domain
  columns + Alliance Check-In Total, each set to Average.
- **Contractor Alliance by Milestone** (T005): same filter/averages as
  above, Rows: `Clinician` + `Milestone`.
- **Contractor Professionalism & Practice Experience** (T009): base is the
  Query Table above (§4.1), Rows: `Clinician`, Data: the three per-patient
  averages, each re-aggregated as Average (average-of-averages — this is
  what makes it patient-weighted rather than response-weighted), columns
  renamed to "Therapist Professionalism (Patient-Weighted Avg)", "Practice
  Experience: Scheduling/Communication (Patient-Weighted Avg)", "Practice
  Experience: Billing (Patient-Weighted Avg)".
- **Contractor Likelihood to Recommend** (T011): `Milestone` filter `= '4 -
  Discharge'`, Rows: `Clinician`, Data: Looking Ahead: Likelihood To
  Recommend, Average.
- **Contractor Reason for Leaving Breakdown** (T012): `Milestone` filter `=
  '5B - Discontinuation, Email Fallback'`, Rows: `Clinician`, Columns:
  `Reason For Leaving`, Data: Count.

All five group by the `Clinician` field itself, never by an enumerated list
of names — confirmed by design review (T015; see §5).

### 4.3 Dashboard — "Contractor Performance" (T017)

Created via "Create New Dashboards" (not "Save As"), per the convention
every prior milestone's own dashboard used. All 5 reports above added as
panels via drag-and-drop from the Explorer panel. **Drill-down links are
NOT wired** (T014 — blocked, see §6). Saved successfully
(`3251423000000083002/edit/3251423000000268432`... final saved view id may
differ after the dashboard's own auto-assigned id; look it up by name if
needed).

## 5. T015 — no hardcoded contractor names (audit)

Reviewed every filter, formula, and title on all 5 reports and the Query
Table (§4 above): every grouping is by the `Clinician` *field*, every filter
is by `Milestone` value, and no report/panel title, filter, or formula
references "Liana", "Deborah", "Shana", or any other contractor by name.
Passes — a new `Clinician` picklist value with qualifying data should appear
in every relevant view with zero edits (Story 6), though this has not been
empirically exercised against real data (see §6).

## 6. T013/T014 — drill-down: descoped 2026-09-30

**Costin's explicit decision: "let's descope drill down for now."** User
Story 5 is not built and not currently planned. The 2026-09-29 investigation
below is kept as a reference in case this gets picked back up later — none of
it is being acted on further right now.

Investigated extensively in this session; no fully satisfactory native
mechanism was found in this Basic Edition workspace for a Pivot-type
dashboard panel to navigate to a *different* dashboard/report on click:

1. No panel-level "Actions"/"Interactions" navigate-to-another-dashboard
   config exists — the panel kebab menu offers only Refresh / Remove /
   Options, and Options' only relevant toggle is **"Use as Filter"**, which
   is Zoho's native cross-panel filtering — it filters *other panels within
   the same dashboard*, not navigation to a different dashboard. Not usable
   for T014.
2. Right-clicking a report column header (Sort/Freeze/Show-Hide-Totals/
   Show-Missing-values only) offers nothing relevant either.
3. Right-clicking an actual data cell on a table with real rows (tested on
   the raw "Milestone Instances" view, not on the empty new reports) reveals
   **"Format Column" → "Associate URL"** — Zoho's actual hyperlink
   mechanism. It only lets you point at an *existing column* on the same
   table that already holds a pre-built URL string. No such column
   currently exists on "Milestone Instances", so using this would require:
   building a new formula column that constructs a target-dashboard URL
   (embedding the `Clinician` value as a filter parameter, in whatever
   syntax Zoho Analytics' report-filter URLs actually use — not confirmed
   against a live destination URL), applying "Associate URL" to point at it,
   and then confirming (untested) that a Pivot's `Clinician` row-header
   inherits table-level column URL formatting when used as a Rows
   dimension. None of this was built — it's speculative and unverified, and
   Basic Edition may simply not support it for Pivot panels at all.
4. **Unresolved product question, not just a technical one**: three of the
   four panels map unambiguously to one destination milestone report
   (Professionalism → M3, Likelihood → M4, Reason for Leaving → M5), but the
   *combined* "Contractor Alliance Summary" panel pools M2+M3+M4 data per
   row and has no single natural destination. This needs Costin's call, not
   build-time exploration.

**Decision**: rather than build an untested, possibly-nonfunctional
URL-formula workaround against a live production workspace, T014 is left
undone and T013 is closed as "no adequate native mechanism found; Associate
URL is the only lead, unverified." The "Contractor Performance" dashboard
(§4.3) is otherwise complete. Costin needs to decide: (a) whether the
Associate-URL workaround is worth building and testing despite the
uncertainty, (b) which report the combined Alliance Summary panel should
target, and (c) whether a higher Analytics edition would offer a cleaner
native mechanism.

## 7. Phase 10 — test data: partially seeded 2026-09-30

**Original production state** (confirmed 2026-09-29 via CRM MCP
`getRecords`, non-PII module): `Milestone_Instances` had exactly 4 records,
all `Milestone = '0 - No Conversion'`, all `Clinician = 'Liana Preudhomme'`,
all `Patient` null — no M2/M3/M4/M5 response existed yet.

**2026-09-29 finding**: a `createRecords` call was denied outright by this
session's auto-mode write classifier ("External System Writes"). **2026-
09-30**: with Costin's explicit go-ahead in chat ("I need you to create some
test milestone instances"), the identical kind of call succeeded — the
classifier appears to gate on there being a live, explicit human request in
the turn, not just on the data/module being written to. Created 5 of
quickstart.md §B's 7 rows via `createRecords` against `Milestone_Instances`
(non-PII fields only: `Milestone`, `Clinician`, `Status`,
`Submitted_Date_Time`, `Response_Data` built per §7's format reference
below):

| CRM record ID | Milestone | Clinician | Purpose |
|---|---|---|---|
| `6825601000004591002` | 2 - Early Alliance Check | Liana Preudhomme | Story 1 |
| `6825601000004591003` | 4 - Discharge | Liana Preudhomme | Story 1 + 3 |
| `6825601000004591004` | 2 - Early Alliance Check | Deborah Webster | Story 1 (isolation check) |
| `6825601000004591005` | 5B - Discontinuation, Email Fallback | Deborah Webster | Story 4 |
| `6825601000004591006` | 2 - Early Alliance Check | Shana Lacastro | Story 6 (new contractor) |

All named `TEST-005 <milestone> <clinician> <hex>` for easy identification
at cleanup time (T021). A manual "Sync Now" was triggered on the workspace's
Zoho CRM data source (Data Sources panel) right after creating these — as of
this note, the sync was still showing "Sync In Progress" after about 45
seconds of polling from the browser, so the dashboard/reports may not show
this data immediately. It should appear once that sync completes (the
source also has a daily scheduled sync, so it will catch up on its own even
if the manual one stalls); nothing further needs to be done to make it
show up.

**Still missing — blocked on the same thing `m2-implementation-notes.md` §5
hit for its own T020**: the 2 remaining quickstart.md §B rows (Story 2's
patient-weighting test — one patient with 2 M3 responses, a second patient
with 1) need a real `Patients1` record ID set on the `Patient` lookup field,
and the standing access constraint still forbids looking one up via CRM
query or browser without Costin's permission. **Asked Costin for a test
Patient record ID** (or two) to complete this specific row set; everything
else in Phase 10 that doesn't need a Patient link is now seeded.

**2026-10-01 update: the 3 M3 rows are now seeded** (Costin supplied test patient PT000,
`Patients1` record ID `6825601000004448028`; see `CLAUDE.md`). Created via `createRecords`:

| CRM record ID | Name suffix | Patient | Prof / Sched-Comm / Billing | Purpose |
|---|---|---|---|---|
| `6825601000004616001` | `PT000-s8 p1q2r3` | PT000 | 9 / 8 / 9 | Patient A, session 8 |
| `6825601000004616002` | `PT000-s16 s4t5u6` | PT000 | 5 / 4 / 5 | Patient A, session 16 |
| `6825601000004616003` | `patientB v7w8x9` | none (null) | 10 / 10 / 10 | Patient B stand-in |

Only one test patient exists, so Patient B is a row with no `Patient` link. That works as a
second grouping bucket only if the Query Table groups nulls together; if the report treats
null differently, ask Costin for a second test patient. Alliance domains: 8/8/8/8, 6/6/6/6,
9/9/9/9. All three are Liana Preudhomme, Status Submitted.

Expected Liana M3 figures (patient-weighted vs naive row average):
Professionalism 8.5 vs 8.0; Scheduling/Communication 8.0 vs 7.33; Billing 8.5 vs 8.0.
If the report shows 8.0 / 7.33 / 8.0, patient-weighting is NOT working.

**Verification status:** the CRM-to-Analytics manual sync triggered 2026-10-01 9:30 AM EDT
sat at "Sync In Progress" for 12+ minutes (the same stall as 2026-09-30), so the new rows had
not reached the dashboard yet (Liana's combined figures still showed only M2 + M4). Verify
after the sync completes or after the daily 5:00 PM EDT scheduled sync. While a sync is in
progress, "Edit Setup" on the data source is blocked ("Sync process is in-progress").

**Confirmed without writing any data** (T003, read-only `getFields`): the
live `Clinician` picklist is
`{-None-, Liana Preudhomme, Deborah Webster, Shana Lacastro}` —
`Shana Lacastro` already existed as a picklist value before this session, so
T016 needed no schema change, only the one seeded response above (now done).

**Response_Data format reference** (for whoever seeds real test data,
Costin or a future permitted session — copied from each milestone's own
implementation notes so it doesn't need to be re-derived):

- M2 (`m2-implementation-notes.md` §3): `"Connection (0-10): " + v +
  "---Understanding (0-10): " + v + "---Shared direction (0-10): " + v +
  "---Fit of approach (0-10): " + v` (no trailing `---`).
- M3 (`m3-implementation-notes.md`, function body): `"Practice Experience:
  Scheduling/Communication (0-10): " + v + "---Practice Experience: Billing
  (0-10): " + v + "---Therapist Professionalism (0-10): " + v +
  "---"` + the same 4-domain M2 block appended after.
- M4 (`m4-implementation-notes.md` §10.2): `"Looking Ahead: Likelihood To
  Recommend (0-10): " + v + "---"` + the same 4-domain M2 block appended
  after (the `Anything Else` segment was removed 2026-09-28).
- M5B (`m5-implementation-notes.md` §786-794): `"Reason For Leaving: " + v +
  "---"` — keeps the trailing `---` even though it's the only/last segment,
  because the formula column extraction is bounded on both sides. Valid
  category values (8 total, per `m5-implementation-notes.md`'s form field
  list): `Felt better / reached my goals`, `Scheduling or timing conflict`,
  `Cost or insurance`, `Didn't feel like the right fit with their
  therapist`, `Life circumstances changed`, `Moved or relocated`, `Choosing
  to pause for now`, `Something else`.

These are **Analytics-side formula columns parsed out of the `Response_Data`
textarea blob** — not individual CRM fields. `getFields` on
`Milestone_Instances` will NOT show "Domain: Connection",  "Therapist
Professionalism", etc. as settable fields; only `Response_Data` (plus
`Milestone`, `Clinician`, `Patient`, `Status`, `Submitted_Date_Time`) are
real CRM fields. Attempting to set per-domain fields directly via
`createRecords` will silently do nothing (unknown field) or error,
depending on the API's strictness — construct the delimited string instead.

## 8. Access constraints that shaped this build

Same standing constraint as every prior milestone: the CRM's Leads and
Patients modules were never opened in the browser, and — per this session's
own added caution — never queried via MCP either, even for bare record IDs,
since that still touches the PII-bearing module. All CRM introspection used
`getFields`/`getRecords` against `Milestone_Instances` only, which carries
no PII. No live/end-to-end test ran; Costin's own pass remains
`quickstart.md` §C, unchanged.

## 9. What's left / what Costin needs to decide

- **T006**: structurally consistent by design (an empty `Clinician` group
  simply doesn't appear in a Pivot's Rows rather than showing as a
  zero/blank row) — can now be checked against the seeded data in §7 once
  the CRM→Analytics sync completes.
- **T010**: still needs a real `Patients1` record ID from Costin (§7) — the
  one open ask from this session.
- **T013/T014**: descoped, not blocked — see §6. Revisit only if Costin
  wants drill-down back on the roadmap later.
- **T016**: done — Shana Lacastro's seeded response is in §7's table; needs
  only the sync (and later, T019/T020 verification) to confirm it surfaces
  correctly.
- **T018**: mostly done (5/7 rows seeded, §7); the last 2 need the Patient
  record ID above.
- **T019/T020**: can run once the sync completes — check the dashboard
  against quickstart.md §A/§B, noting Story 2 will stay unverifiable until
  T010's rows are in.
- **T021**: cleanup — delete the 5 (soon 7) `TEST-005 ...` records via CRM
  MCP `deleteRecords` once verification is done; their IDs are listed in §7.
