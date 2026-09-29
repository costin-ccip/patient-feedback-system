# Quickstart / Validation: Contractor Performance Analytics

**Note**: PROSPECTIVE — written before this feature is built. Per standing
instruction ("no live/end-to-end tests without me"), this guide covers
structural verification and sample-data checks only. Section C is Costin's own
live pass, not something this session runs.

## A. Structural verification (to perform once built)

For each of the five new reports and the "Contractor Performance" dashboard:

1. Confirm the report's grouping/filter is expressed by field (`Clinician`,
   `Milestone`), never as a hardcoded list of contractor names or a per-contractor
   copy of the report.
2. Confirm no report or dashboard panel has a `Patient` or other patient-identifying
   column in its visible column list — same check `m0-implementation-notes.md` §7's
   "raw `Response Data` blob was removed... no reason to show clinicians/ops the raw
   delimited string" already models for prior milestones.
3. For the Story 1 combined view: confirm the `Milestone` filter is `IN ('2 - Early
   Alliance Check', '3 - Periodic Consolidated', '4 - Discharge')`, not accidentally
   scoped to only one or two of the three.
4. For the Story 1 per-milestone view: confirm figures for the same contractor
   differ (or are correctly equal, if their underlying data happens to match) across
   the M2/M3/M4 breakdown versus the combined view, and that the combined figure is
   mathematically consistent with the per-milestone ones given the underlying
   response counts.
5. For the Story 2 patient-weighted view: confirm a contractor with one patient who
   has 2+ Milestone 3 responses produces a different (patient-weighted) figure than
   a naive row-level average of all that contractor's M3 responses would — see the
   sample-data test in section B for the concrete numbers to check this against.
6. Confirm the `Patients1.Name` grouping key from research.md Decision 1 appears
   in no report's column list and no dashboard panel anywhere in the workspace.
7. For User Story 5: click through each panel's drill-down and confirm it lands on
   the correct existing M2/M3/M4/M5 report, filtered to the contractor that panel
   represents.
8. For User Story 6: confirm by inspecting each report/column definition that
   nothing references a specific contractor's name in a formula, filter, or report
   title — the only place a contractor's name should ever appear is as data (a
   `Clinician` field value), never as configuration.
9. Confirm no CRM field, picklist, Zoho Flow, or Zoho Form was touched — this
   feature's entire build surface is Zoho Analytics (per plan.md's Technical
   Context).

## B. Sample-data verification (via Zoho CRM MCP tools, no browser)

Same discipline `m0-implementation-notes.md` §8 established: manage
`Milestone_Instances` test records via `getRecords`/`createRecords`/
`deleteRecords` MCP tool calls, not by browsing the Leads/Patients modules
(off-limits per the standing access constraint) or the Milestone_Instances
module's own records list unnecessarily.

Suggested seed set to exercise every view in this feature, across at least two
`Clinician` values (e.g. `Liana Preudhomme` and `Deborah Webster`):

| Milestone | Clinician | Notes |
|---|---|---|
| `2 - Early Alliance Check` | Liana | 1 response, mid-range domain scores |
| `3 - Periodic Consolidated` | Liana | 1 response for Patient A (session 8) |
| `3 - Periodic Consolidated` | Liana | 2nd response for the SAME Patient A (session 16) — this is the row that tests Story 2's patient-weighting; without it, patient-weighted and response-weighted averages can't be told apart |
| `3 - Periodic Consolidated` | Liana | 1 response for a different Patient B (session 8) |
| `4 - Discharge` | Liana | 1 response, with a Likelihood-to-Recommend value |
| `2 - Early Alliance Check` | Deborah Webster | 1 response, deliberately different domain scores from Liana's, to confirm the two contractors' averages don't mix |
| `5B - Discontinuation, Email Fallback` | Deborah Webster | 1 response, any Reason For Leaving category |

**Expected results**:
- Story 1 combined view: Liana's figure reflects her M2 + both M3 + M4 responses
  (4 responses); Deborah's reflects only her single M2 response. Unaffected by each
  other.
- Story 1 per-milestone view: Liana's M3 figure reflects both her M3 responses
  (Patient A ×2, Patient B ×1 — response-weighted, 3 data points).
- Story 2 view: Liana's Professionalism/Practice Experience figure is computed as
  (Patient A's own 2-response average) and (Patient B's single response), then
  averaged across those two patients — i.e. Patient A's two check-ins should carry
  the same combined weight as Patient B's one, not twice the weight. This is the
  concrete check that distinguishes patient-weighted from response-weighted.
- Story 3: only Liana has a Discharge response in this seed set; her figure shows,
  Deborah's view shows no data (not zero).
- Story 4: only Deborah has a Discontinuation response; her breakdown shows one
  category at 100%; Liana's view shows no data (not zero).
- Story 5: drilling down from Liana's Story 1 panel reaches the existing "M2
  Submitted Responses" / "M3 Submitted Responses" / "M4 Submitted Responses"
  reports (whichever the panel corresponds to), filtered to Liana only.
- Story 6: add a third `Clinician` value with one qualifying response (reusing the
  practice's actual 3rd contractor, `Shana Lacastro`, is fine) and confirm it
  appears in the relevant view(s) with zero report/dashboard edits.

Clean up all seeded test records afterward via `deleteRecords`, same as every
prior milestone's own sample-data pass.

## C. Live test (Costin runs this later; not part of this session's work)

Not applicable in the usual "flip a flow on and watch it fire" sense this
pipeline's other live tests use — this feature has no flow to switch on. Costin's
own pass is to open the built "Contractor Performance" dashboard against real
production data and sanity-check that the numbers for each real contractor look
plausible against what he already knows about their caseloads, and to confirm the
drill-down links behave as expected in the live workspace UI (not just structurally,
per section A above).
