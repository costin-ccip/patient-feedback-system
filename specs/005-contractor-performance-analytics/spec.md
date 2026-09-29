# Feature Specification: Contractor Performance Analytics

**Feature Branch**: `005-contractor-performance-analytics`

**Created**: 2026-09-29

**Status**: Draft

**Input**: User description: "As a practice owner I would like to understand, for each
contractor I hire, how they are performing across their pool of patients, so that I can
make informed business decisions with regards to that contractor." Scoped down from a
broader 22-item brainstorm (see the "Selected from brainstorm" note below) to six
specific stories Costin confirmed he wants built now: per-contractor Alliance score
averages (M2/M3/M4), per-contractor Practice Experience and Therapist Professionalism
averages (M3, including Scheduling/Communication and Billing), per-contractor
Likelihood-to-Recommend averages (M4), per-contractor Discontinuation reason breakdown
(M5), drill-down from a contractor's summary panel into existing milestone-level detail,
and a dashboard design that doesn't require rebuilding when a new contractor joins the
practice.

<!--
  Relationship to existing scope: spec.md's own User Story 3 (FR-007-009) already
  defines a unified, cross-milestone Admin Dashboard with a clinician filter -- but
  every milestone built so far (M0, M2-M5) has explicitly deferred it, and none of
  the shipped per-milestone dashboards group or filter by Clinician today. This
  feature is narrower than that eventual unified dashboard: it's a contractor-focused
  performance view, not a full patient-level browsing tool, and it can be built now
  using data every milestone already collects. It may end up feeding into, or being
  superseded by, that unified dashboard later -- not resolved here.

  "Selected from brainstorm": in-conversation planning (2026-09-28/29) produced 22
  candidate granular user stories across 9 themes (score rollups, response rate,
  trend-over-time, cross-contractor comparison, clinical safety flag visibility,
  caseload/volume context, statistical-integrity safeguards, navigation/scalability,
  business-decision support). Costin selected 6 of the 22 to build now (items 1-4,
  19-20 of that list); the rest -- response rate, trend-over-time, benchmarking/
  ranking, flag-rate-per-contractor, caseload size, small-sample suppression,
  export/alerting -- are explicitly out of scope for this feature (see Assumptions)
  and may become their own future feature(s).
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Per-contractor Alliance score averages, combined and per-milestone (Priority: P1)

As the practice owner, I want to see each contractor's average Alliance scores
(Connection, Understanding, Shared Direction, Fit of Approach, and the 0-40 Total),
both as one figure pooling all of that contractor's Alliance-track responses (Milestones
2, 3, and 4 together) and broken out separately by milestone (M2 only, M3 only, M4
only), so I can see both a contractor's overall Alliance standing and whether it looks
different early versus late in a patient's treatment with them.

**Why this priority**: The Alliance track is the pipeline's only relationship/
satisfaction signal collected at three separate points in a patient's care (early check,
periodic check-in, discharge), and it's the metric most directly about how a contractor
is doing with their patients. It's the foundation the rest of this feature's views sit
alongside.

**Independent Test**: Can be fully tested by seeding Milestone 2, 3, and 4 responses
across at least two different contractors' patients and confirming the dashboard shows,
for each contractor: one combined Connection/Understanding/Shared Direction/Fit of
Approach/Total average across all three milestones, and three further sets of the same
averages, one per milestone — all computed only from that contractor's patients' rows.

**Acceptance Scenarios**:

1. **Given** submitted Alliance-track responses (Milestone 2, 3, and/or 4) exist for a
   contractor's patients, **When** the practice owner opens that contractor's
   performance view, **Then** they see that contractor's combined average score for
   each of the four Alliance domains and the 0-40 Total, pooling all of that
   contractor's Alliance-track responses regardless of which of the three milestones
   each response came from.
2. **Given** the same data, **When** the practice owner looks at the per-milestone
   breakdown, **Then** they additionally see the same set of averages computed
   separately for that contractor's M2-only, M3-only, and M4-only responses.
3. **Given** two contractors each have their own patients' Alliance-track responses,
   **When** the practice owner views each contractor's averages (combined or
   per-milestone), **Then** one contractor's numbers are unaffected by the other
   contractor's patients' scores.
4. **Given** a contractor has no submitted Alliance-track responses yet for the
   combined view, or none at a specific milestone for the per-milestone view, **When**
   the practice owner opens that contractor's view, **Then** the absence of data is
   shown as such (e.g. no responses yet) for whichever of the two is missing, not as a
   zero or blank average that could be misread as a real score.

---

### User Story 2 - Per-contractor Practice Experience and Therapist Professionalism averages (Priority: P2)

As the practice owner, I want to see each contractor's average Therapist Professionalism
score, alongside the average Practice Experience: Scheduling/Communication and Practice
Experience: Billing scores from the same patients, so I can see both the
contractor-specific professionalism signal and the operational-experience signal for
that contractor's caseload side by side. Because Milestone 3 recurs (every 8th session,
so one patient can submit several M3 responses over a long course of treatment), each
patient's own M3 responses are averaged together first, and a contractor's score is
then the average of those per-patient figures — so one long-tenured patient's repeated
check-ins can't outweigh a contractor's other patients.

**Why this priority**: This is the pipeline's most direct patient-facing measure of a
contractor's professionalism, and it's collected in the same Milestone 3 instrument as
two operational ratings the practice owner also wants visible per contractor, even
though those two are more about practice operations shared across contractors than about
any one contractor's individual conduct (see Assumptions).

**Independent Test**: Can be fully tested by seeding Milestone 3 responses across at
least two contractors' patients — including at least one patient with more than one M3
response — and confirming each contractor's view shows their own average Therapist
Professionalism, Scheduling/Communication, and Billing scores, computed by first
averaging each patient's own responses and then averaging those per-patient figures
across the contractor's caseload, using only that contractor's patients.

**Acceptance Scenarios**:

1. **Given** submitted Milestone 3 responses exist for a contractor's patients, **When**
   the practice owner opens that contractor's performance view, **Then** they see that
   contractor's average Therapist Professionalism score, average Practice Experience:
   Scheduling/Communication score, and average Practice Experience: Billing score, each
   computed across that contractor's own Milestone 3 responses only.
2. **Given** one of a contractor's patients has submitted more than one Milestone 3
   response (e.g. at session 8 and again at session 16), **When** the contractor's
   average is computed, **Then** that patient's multiple responses are first averaged
   together into one per-patient figure before being combined with the contractor's
   other patients' figures, so that patient's repeated check-ins count once, not once
   per response.
3. **Given** a contractor has no submitted Milestone 3 responses yet, **When** the
   practice owner opens that contractor's view, **Then** the absence of data is shown as
   such, not as a zero or blank average.

---

### User Story 3 - Per-contractor Likelihood-to-Recommend averages (Priority: P3)

As the practice owner, I want to see each contractor's average "Looking Ahead:
Likelihood to Recommend" score from their patients' Discharge (Milestone 4) responses,
so I can see whether patients discharged from a given contractor's care are, on
average, likely to recommend Cape Clarity.

**Why this priority**: This is a forward-looking, business-relevant signal (would this
contractor's discharged patients recommend the practice), but it depends on patients
actually completing treatment and reaching discharge, so it has a narrower and slower-
growing data set than the Alliance or Professionalism views.

**Independent Test**: Can be fully tested by seeding Milestone 4 responses across at
least two contractors' patients and confirming each contractor's view shows their own
average Likelihood-to-Recommend score, computed only from that contractor's patients'
Milestone 4 responses.

**Acceptance Scenarios**:

1. **Given** submitted Milestone 4 responses exist for a contractor's patients, **When**
   the practice owner opens that contractor's performance view, **Then** they see that
   contractor's average Likelihood-to-Recommend score, computed across that contractor's
   own Milestone 4 responses only.
2. **Given** a contractor has no submitted Milestone 4 responses yet, **When** the
   practice owner opens that contractor's view, **Then** the absence of data is shown as
   such, not as a zero or blank average.

---

### User Story 4 - Per-contractor discontinuation reason breakdown (Priority: P4)

As the practice owner, I want to see, for each contractor, how their discontinuing
patients' stated reasons for leaving (from Milestone 5) break down by category, so I
can tell whether a contractor's departing patients cluster around a particular reason
(for example, fit with the therapist, cost, or scheduling) rather than reasons spread
evenly across the practice.

**Why this priority**: This is the pipeline's only direct signal on why a contractor's
patients leave, but it's the last-priority milestone in a patient's journey and the
smallest data set of the four score/category views (only patients who both discontinue
and respond).

**Independent Test**: Can be fully tested by seeding Milestone 5 responses with a mix of
"Reason for Leaving" categories across at least two contractors' patients and confirming
each contractor's view shows a breakdown (count or share per category) computed only
from that contractor's own patients' Milestone 5 responses.

**Acceptance Scenarios**:

1. **Given** submitted Milestone 5 responses exist for a contractor's patients, **When**
   the practice owner opens that contractor's performance view, **Then** they see a
   breakdown of "Reason for Leaving" by category, computed across that contractor's own
   Milestone 5 responses only.
2. **Given** a contractor has no submitted Milestone 5 responses yet, **When** the
   practice owner opens that contractor's view, **Then** the absence of data is shown as
   such, not as an empty or misleadingly flat breakdown.

---

### User Story 5 - Drill down from a contractor's summary into milestone-level detail (Priority: P5)

As the practice owner, when a number on a contractor's summary panel looks notable
(high, low, or unexpected), I want to drill down from that summary into the existing
milestone-level detail reports (the same de-identified, response-level views the M0/M2/
M3/M4/M5 dashboards already provide), so I can see the underlying distribution or
individual responses without leaving the contractor view or re-deriving the same
filter by hand.

**Why this priority**: This depends on Stories 1-4 already existing (there's nothing to
drill down from otherwise), and it's a navigation/usability improvement on top of them
rather than new data.

**Independent Test**: Can be fully tested by opening a contractor's summary panel for
one of the four score/breakdown views above, using its drill-down action, and
confirming it lands on the corresponding existing milestone-level report or a
new equivalent, filtered to that same contractor, with the same de-identification
discipline as every existing milestone report (no patient-identifying column).

**Acceptance Scenarios**:

1. **Given** the practice owner is viewing a contractor's Alliance score summary,
   **When** they use the drill-down action, **Then** they reach a milestone-level view
   of that contractor's individual Alliance-track responses, with no patient-identifying
   column, consistent with every existing milestone report's de-identification
   discipline.
2. **Given** the same drill-down action for the Professionalism/Practice Experience,
   Likelihood-to-Recommend, or Discontinuation Reason summaries, **Then** each reaches
   its own corresponding milestone-level detail, filtered to that contractor.

---

### User Story 6 - New contractors appear without rebuilding the dashboard (Priority: P5)

As the practice owner, when I bring on a new contractor, I want that contractor to show
up in every view built by this feature without anyone needing to add a new panel,
report, or filter by hand, so the dashboard keeps working as the practice grows.

**Why this priority**: Tied with Story 5 as a cross-cutting property rather than its
own data view; it doesn't deliver a new number on its own, but every one of Stories 1-4
becomes maintenance-heavy (and error-prone) at practice scale if it isn't true. It's
placed here because it's best verified once at least one of Stories 1-4 already exists
to test it against.

**Independent Test**: Can be fully tested by adding a new value to the
`Milestone_Instances.Clinician` picklist (the mechanism the practice has already used
twice to add its second and third contractor), seeding at least one submitted response
attributed to that new value across any of the milestones covered by Stories 1-4, and
confirming that contractor's numbers appear in the relevant view(s) without any report,
panel, or dashboard-definition change.

**Acceptance Scenarios**:

1. **Given** a new contractor is added to the `Clinician` picklist and has at least one
   submitted response, **When** the practice owner opens the relevant performance
   view(s), **Then** that contractor's own averages/breakdown appear alongside existing
   contractors', with no engineering change required to surface them.
2. **Given** a contractor is added to the picklist but has no submitted responses yet,
   **When** the practice owner opens the dashboard, **Then** that contractor either
   appears with an explicit "no data yet" state or does not appear until they have data
   — either is acceptable, but silently showing a misleading zero/blank average is not
   (same requirement as Stories 1-4's own "no data" acceptance criteria).

### Edge Cases

- **Milestone 0 is excluded from every view in this feature.** M0 (free consult,
  no conversion) is hardcoded to a single clinician (Liana Preudhomme only ever runs
  free consults) and its instrument has no Alliance, Professionalism, or Likelihood-to-
  Recommend content, so it contributes nothing to Stories 1-3 and isn't a contractor-
  comparison case at all.
- **Milestone 3 recurs per patient** (every 8th session, so one patient can have
  multiple Milestone 3 rows). Per Costin's decision, a contractor's Professionalism/
  Practice Experience averages (Story 2) average each patient's own M3 responses
  together first, then average those per-patient figures across the contractor's
  caseload — a patient in month 9 of treatment counts once, not once per check-in.
  Story 1's Alliance averages (M2/M3/M4 combined and per-milestone) are NOT
  patient-weighted this way; see Assumptions for why the two stories treat this
  differently.
- **A contractor with very few submitted responses.** With only two or three active
  contractors and a still-growing response volume, a contractor's average could rest on
  a handful of responses, or even one. Per Costin's explicit decision, this feature
  defines no minimum sample size, suppression rule, or "low confidence" indicator for a
  small-n average in this first version (see Assumptions) — revisit if it becomes a
  real problem once more data exists. Whatever the number of underlying responses, it
  MUST remain de-identified to the individual patient exactly as today's
  milestone-level reports already are.
- **A patient is reassigned to a different contractor mid-treatment.** Consistent with
  spec.md's existing edge case for this scenario (001-feedback-collection-pipeline), a
  response already attributed to the prior contractor at issuance keeps that
  attribution; it does not retroactively move to the new contractor. This feature
  reports on `Clinician` as recorded on each `Milestone_Instances` row at issuance time,
  not on a patient's current assignment.
- **A response predates the M2-M5 clinician-derivation fix (2026-09-25).** Any
  `Milestone_Instances` rows created before that fix could carry an incorrect
  hardcoded `Clinician` value rather than the patient's actual assigned therapist at
  the time. This feature reports whatever `Clinician` value is on the row; it does not
  attempt to detect or correct pre-fix misattribution -- flagged here so it isn't
  mistaken for a new bug if noticed later.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST compute, for each contractor, average scores for each of
  the four Alliance domains (Connection, Understanding, Shared Direction, Fit of
  Approach) and the Alliance Total, combined across that contractor's patients'
  Milestone 2, 3, and 4 responses into one figure per domain/Total.
- **FR-001a**: The system MUST additionally compute the same set of Alliance domain and
  Total averages separately for each of Milestone 2, Milestone 3, and Milestone 4, so a
  contractor's Alliance standing can be viewed both combined and per-milestone.
- **FR-002**: The system MUST compute, for each contractor, average Therapist
  Professionalism, Practice Experience: Scheduling/Communication, and Practice
  Experience: Billing scores, across that contractor's patients' Milestone 3 responses,
  weighted so that each patient counts once: a patient's own multiple Milestone 3
  responses (from recurring 8th-session check-ins) are averaged together first, and the
  contractor's score is the average of those per-patient figures, not a raw average of
  every individual response.
- **FR-003**: The system MUST compute, for each contractor, an average
  Likelihood-to-Recommend score, across that contractor's patients' Milestone 4
  responses.
- **FR-004**: The system MUST compute, for each contractor, a breakdown (count and/or
  share) of "Reason for Leaving" categories, across that contractor's patients'
  Milestone 5 responses.
- **FR-005**: Every average or breakdown in FR-001 through FR-004 MUST be computed only
  from the responses attributed to that specific contractor (via the `Clinician` value
  recorded on each `Milestone_Instances` row at issuance), never mixed with another
  contractor's responses.
- **FR-006**: When a contractor has no submitted responses for a given view, the system
  MUST show that absence explicitly rather than a zero, blank, or otherwise misleading
  average.
- **FR-007**: The system MUST provide a way to navigate from each of the four
  contractor-level views (FR-001 through FR-004) to the corresponding existing
  milestone-level detail report(s), filtered to that same contractor, preserving the
  same de-identification discipline (no patient-identifying column) as every existing
  milestone report.
- **FR-008**: The system MUST surface a newly added contractor (a new value on the
  `Milestone_Instances.Clinician` picklist) in every view built by this feature once
  that contractor has qualifying data, without requiring a new report, panel, or
  dashboard definition to be built for that specific contractor.
- **FR-009**: This feature MUST NOT grant any contractor account access to their own or
  any other contractor's performance view or underlying data, consistent with
  spec.md's Principle IV / FR-009 (dashboard access remains practice-admin-only).
- **FR-010**: This feature MUST NOT introduce any new patient-identifying field into
  Zoho Analytics or any report/dashboard it creates, consistent with Principle I/II —
  contractor attribution (`Clinician`) is not patient identity, but no
  patient-identifying data becomes newly visible because of this feature.
- **FR-011**: Milestone 0 MUST be excluded from every contractor-level view this
  feature builds (see Edge Cases).

### Key Entities

- **Contractor (Clinician)**: The care-providing professional a patient is assigned to.
  Already represented on every `Milestone_Instances` row via the `Clinician` picklist
  field (derived from the patient's `Assigned_Therapist` at issuance for Milestones
  2-5; hardcoded for Milestone 0). This feature's central grouping dimension; it does
  not introduce a new entity, only new views grouped by an existing one.
- **Contractor Performance View**: A per-contractor summary combining the four
  score/breakdown views from Stories 1-4, plus drill-down links (Story 5) into
  milestone-level detail. Automatically includes any contractor value present in the
  underlying data (Story 6) rather than being hand-built per contractor.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For any contractor with at least one submitted Alliance-track response,
  the practice owner can see that contractor's own combined Alliance domain and Total
  averages within the dashboard, with no manual filtering or export required.
- **SC-001a**: For any contractor with at least one submitted response at a specific
  Alliance-track milestone (M2, M3, or M4), the practice owner can see that
  contractor's Alliance domain and Total averages for that milestone specifically,
  distinct from the combined figure.
- **SC-002**: For any contractor with at least one submitted Milestone 3 response, the
  practice owner can see that contractor's own Professionalism, Scheduling/
  Communication, and Billing averages within the dashboard, computed one-vote-per-patient
  rather than one-vote-per-response.
- **SC-003**: For any contractor with at least one submitted Milestone 4 response, the
  practice owner can see that contractor's own Likelihood-to-Recommend average within
  the dashboard.
- **SC-004**: For any contractor with at least one submitted Milestone 5 response, the
  practice owner can see that contractor's own Reason-for-Leaving breakdown within the
  dashboard.
- **SC-005**: From any of the four contractor-level views, the practice owner can reach
  the corresponding milestone-level detail in one navigation action.
- **SC-006**: Adding a new contractor to the `Clinician` picklist and seeding one
  qualifying response for them requires 0 changes to any report, panel, or dashboard
  definition before that contractor's data appears in the relevant view(s).
- **SC-007**: 0 instances of a contractor account accessing any view built by this
  feature.

## Assumptions

- This feature is scoped to exactly the six stories above, selected by Costin from a
  larger 22-item brainstorm covering per-contractor response rate, trend-over-time,
  cross-contractor comparison/benchmarking, clinical-safety-flag rate per contractor,
  caseload/volume context, small-sample statistical safeguards, and business-decision-
  support items (export, alerting). Those are explicitly out of scope for this feature
  and may be scoped as later, separate features if wanted.
- Scheduling/Communication and Billing ratings (part of Story 2) are, by their content,
  more reflective of practice-wide operations than of an individual contractor's own
  conduct. They're included per Costin's explicit choice, on the premise that seeing
  them broken out by contractor's caseload may still surface a real pattern (e.g. one
  contractor's patients consistently reporting scheduling friction) even though the
  underlying cause may not be that contractor personally. This spec does not resolve
  how that ambiguity should be presented (e.g. a caveat/label distinguishing it from
  Therapist Professionalism) -- left for planning.
- **Resolved 2026-09-29 (Costin, via clarifying questions on this spec)**: Story 1's
  Alliance averages are shown both combined (M2+M3+M4 pooled into one figure) and
  broken out per milestone, so a contractor's early-treatment alliance can be compared
  against their later-treatment alliance. Story 1 is deliberately NOT patient-weighted
  the way Story 2 is (below) — unlike Story 2, nothing in the Alliance track's design
  makes one milestone recur per patient the way Milestone 3 does (a given patient
  contributes at most one M2 response and one M4 response; M3 is the only recurring
  leg, and it's folded into the same combined/per-milestone treatment as M2/M4 here
  rather than getting Story 2's separate per-patient averaging — if that inconsistency
  turns out to matter in practice, it's worth revisiting at planning time).
- **Resolved 2026-09-29**: Story 2's averages are computed one-vote-per-patient, not
  one-vote-per-response — each patient's own Milestone 3 responses are averaged
  together first, then those per-patient figures are averaged across the contractor's
  caseload, so a long-tenured patient's repeated 8th-session check-ins can't outweigh
  a contractor's other patients.
- **Resolved 2026-09-29**: No minimum sample size or small-n suppression/caveat is
  defined by this spec, by Costin's explicit choice — ship the six stories as scoped
  without it for now, even though with only two or three active contractors today,
  some averages may rest on very few responses. Revisit as a future feature if this
  becomes a real problem once more data exists.
- This feature is expected to be built entirely within Zoho Analytics (new formula
  columns, reports, and a dashboard grouped/filtered by the existing `Clinician`
  field), consistent with spec.md's Principle VII (derived values computed in
  Analytics, not Deluge) and requiring no new Zoho product, and therefore no new
  BAA-gating decision (Principle V) — to be confirmed at planning time.
- Whether this feature's views live inside one of the existing per-milestone
  dashboards, as new panels on a new standalone "Contractor Performance" dashboard, or
  some combination, is a planning-level design decision, not resolved by this spec.
