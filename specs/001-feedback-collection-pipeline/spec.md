# Feature Specification: Feedback Collection Pipeline

**Feature Branch**: `001-feedback-collection-pipeline`

**Created**: 2026-09-04

**Status**: Draft

**Input**: User description: "Cape Clarity needs an automated, de-identified patient
feedback collection pipeline: feedback requests fire on six defined milestones (not left
to contractors to remember), the survey tool never sees patient identity, only CRM ever
rejoins a response to a patient, contractors have no access to feedback data or
dashboards in the first version, and there is one admin-only reporting dashboard with a
patient-level view (scores per patient) and an overall view across all patients that can
be filtered by clinician. Two of the milestones need special handling: a wellbeing/
alliance score crossing a clinical concern threshold must raise a flag for admin review
(never sent automatically to the treating contractor), and discontinuation (a patient
leaving care) needs its own exit-reason capture, delivered by email."

<!--
  Recreated from a description of prior, undocumented work: the repository holding this
  spec was found completely empty at the start of the session that authored this file.
  Content below reconstructs the user stories, requirements, and scope described by the
  project's operations lead, cross-checked against the constitution
  (.specify/memory/constitution.md) and the source design doc.

  Revised 2026-09-05 (first pass) per operations-lead feedback: (1) contractors have no
  dashboard access at all in v1 — dashboards are admin-only; (2) the dashboard is
  redefined as one admin-facing tool with a patient-level view and a clinician-filterable
  overall view, rather than two separate dashboards; (3) all references to a live-call
  discontinuation path removed — that was an old requirement no longer under
  consideration.

  Revised 2026-09-05 (second pass), after reviewing the full Confluence source
  ("Customer Feedback Collection" and its six milestone detail pages): (4) the six
  milestones (0–5) are now enumerated explicitly with their trigger conditions, since
  they are business requirements, not implementation detail; (5) added a clinical safety
  flag mechanism (User Story 4) for wellbeing/alliance scores crossing a concern
  threshold — per operations-lead direction, this flags the practice admin, who then
  decides whether to request a case consultation from the treating contractor; the
  contractor is never notified automatically; (6) added Principle VI to the constitution
  (data from this pipeline can never be used as a public testimonial or marketing
  material) and referenced it here; (7) resolved a conflict between the operations lead's
  explicit instruction (email-only, no live call) and the source doc's Milestone 5 design
  (live call as the primary path, email only as a fallback after a failed call attempt):
  per the operations lead's more recent and explicit instruction, this version uses email
  only, firing directly on the same trigger conditions without any call-attempt gate —
  see Assumptions.

  Revised 2026-09-05 (third pass): corrected the trigger source for all six milestones.
  The source Confluence doc describes Milestones 1–4 as triggering off the practice's
  EHR (SimplePractice); the operations lead has explicitly corrected this — all six
  milestones trigger off Zoho CRM, and the EHR is not part of the trigger loop at all.
  Updated the Milestones table, the Prospect entity, and Assumptions accordingly, and
  resolved (removed) the corresponding TODO(TRIGGER_SOURCE) in the constitution.
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Feedback requests fire automatically on milestones (Priority: P1)

As the practice, we need feedback requests to go out the moment a patient (or, for
Milestone 0, a prospect) hits one of six defined points in their journey, without any
clinician having to remember to ask or manually kick it off. The six milestones are: 0 —
free consult with no conversion, 1 — baseline intake, 2 — early alliance check, 3 —
periodic consolidated check-in, 4 — discharge, and 5 — discontinuation. Full trigger
definitions are in the Milestones subsection under Requirements.

**Why this priority**: Every other part of this feature depends on requests actually
going out reliably. If sending is contractor-dependent, coverage silently degrades to
whichever contractors remember to do it, and the whole feedback program becomes
unreliable and biased toward whoever opts in. This is the foundation everything else is
built on.

**Independent Test**: Can be fully tested by advancing a test patient (or prospect, for
Milestone 0) record through each of the six defined milestone conditions and confirming a
feedback request is generated and sent for each one, with zero manual action from a
contractor at any point.

**Acceptance Scenarios**:

1. **Given** a patient's (or prospect's) record crosses one of the six defined milestone
   conditions, **When** the system next evaluates that record, **Then** a feedback
   request for that milestone is generated and sent automatically, with no clinician
   action required.
2. **Given** a patient has already received a feedback request for a given milestone
   instance, **When** the same milestone condition is still true on a later check,
   **Then** the system does not send a duplicate request for that same milestone
   instance.
3. **Given** a contractor takes no action at all with respect to feedback, **When** their
   patients cross milestone conditions, **Then** feedback requests still go out for those
   patients on schedule.

---

### User Story 2 - Feedback is collected de-identified and rejoined only in CRM (Priority: P2)

As the practice, we need patients (and prospects, for Milestone 0) to be able to answer
honestly without the survey tool itself ever knowing who they are, while still being
able to tell, afterward, which person a given response belongs to — with that link made
only inside the system of record (CRM), not inside the survey tool.

**Why this priority**: This is the core privacy mechanism the entire program depends on
for trust and compliance. It's ordered after automated triggers because a trigger has to
exist before there's anything to collect, but it's more foundational than the reporting
built on top of it.

**Independent Test**: Can be fully tested by sending a feedback request, submitting a
response, and confirming (a) the survey tool's stored submission never contained a name,
email, or other direct identifier — only an opaque token — and (b) the practice can still
retrieve that response correctly matched to the right person afterward, with the match
happening in CRM.

**Acceptance Scenarios**:

1. **Given** a feedback request is generated, **When** the request is sent, **Then** it
   includes a single-use token unique to that request, and no identifying field is
   included in the collection form itself.
2. **Given** a patient or prospect submits a response using their token, **When** the
   submission is received, **Then** the system matches the token back to the correct
   record inside CRM, and the response's scores/answers are recorded against that record
   there.
3. **Given** a token has already been used for a submission, **When** the same token is
   submitted again, **Then** the system rejects the second submission as invalid.
4. **Given** a response has been matched back to a record in CRM, **When** enough time
   has passed to complete that rejoin, **Then** the original raw submission held by the
   collection tool is purged, within a short, defined retention window.

---

### User Story 3 - Admin dashboard for reviewing feedback data (Priority: P3)

As a practice admin, I need to be able to see that feedback is actually being collected
and to interpret it — both at the level of an individual patient (to sanity-check the
pipeline is working and understand one patient's results) and across the whole practice,
with the ability to narrow that overall view down to one clinician's caseload at a time.
In this first version, this dashboard is admin-only; contractors have no access to
feedback data or dashboards of any kind.

**Why this priority**: Reporting is what turns collected feedback into something useful,
but it depends entirely on Stories 1 and 2 already working — there's nothing to report on
until requests go out and responses get collected and rejoined.

**Independent Test**: Can be fully tested by seeding feedback data across multiple
patients and clinicians, then confirming an admin can (a) open a patient-level view and
see that patient's recorded scores/responses, and (b) open an overall view showing
practice-wide trends that can be filtered down to a single clinician's caseload — while
confirming no contractor account can reach either view.

**Acceptance Scenarios**:

1. **Given** a patient has submitted feedback, **When** an admin opens that patient's
   record in the dashboard, **Then** they see that patient's recorded scores/responses,
   confirming the response was captured and rejoined correctly.
2. **Given** feedback responses exist across multiple patients and clinicians, **When**
   an admin opens the overall view with no filter applied, **Then** they see aggregated,
   practice-wide trends (satisfaction, communication, billing, discontinuation reasons,
   likelihood-to-refer, per-clinician professionalism averages).
3. **Given** the same overall view, **When** an admin filters it to a single clinician,
   **Then** they see trends limited to that clinician's own caseload.
4. **Given** any contractor account, **When** that contractor attempts to access the
   dashboard, **Then** access is denied — no contractor-facing dashboard or feedback data
   view exists in this version.

---

### User Story 4 - Automated clinical safety flag to admin (Priority: P4)

As a practice admin, I need to be notified automatically when a patient's wellbeing or
alliance score data crosses a defined clinical concern threshold (a stall, a
deterioration, or a low alliance reading), so I can decide whether to request a case
consultation with that patient's treating contractor. The contractor is never
automatically notified of the flag — reaching out to them is my decision to make.

**Why this priority**: This is a patient-safety mechanism, distinct from general
reporting, so it's prioritized ahead of discontinuation handling. It depends on scored
milestone data already existing (Stories 1–2) and surfaces through the same admin-facing
dashboard as Story 3, so it follows both.

**Independent Test**: Can be fully tested by seeding a patient's wellbeing or alliance
score data to cross one of the defined flag rules (see Requirements) and confirming an
admin sees a flag identifying that patient and the rule that fired, while confirming the
treating contractor receives no automated notification or visibility into it.

**Acceptance Scenarios**:

1. **Given** a patient's wellbeing score data meets a defined stall or deterioration
   rule, **When** the system evaluates the new reading, **Then** a flag is raised for
   practice-admin review identifying the patient and which rule fired.
2. **Given** a patient's alliance score data meets the defined low-alliance rule, **When**
   the system evaluates the new reading, **Then** a flag is raised for practice-admin
   review identifying the patient and which rule fired.
3. **Given** a flag has been raised for a patient, **When** the system processes it,
   **Then** the patient's treating contractor receives no automated notification or view
   of the flag — only a practice admin can see it, and only an admin can decide to
   request a case consultation from that contractor.
4. **Given** a flag exists for a patient, **When** an admin opens that patient's record in
   the dashboard (Story 3), **Then** the flag and the rule that triggered it are visible
   there.

---

### User Story 5 - Discontinuation exit-reason capture (Priority: P5)

As the practice, when a patient stops care (discontinuation), we need to capture why —
using the same de-identified, automated flow as every other milestone, delivered by
email.

**Why this priority**: This is a specific milestone instance built on top of Stories 1–3
rather than a separate mechanism; it's the last story because it depends on all of them
being in place, and it's the milestone most likely to need product refinement later once
the core pipeline is proven.

**Independent Test**: Can be fully tested by moving a test patient into discontinuation
status and confirming an exit-reason feedback request is sent by email, using the same
token/de-identification/rejoin mechanism as other milestones.

**Acceptance Scenarios**:

1. **Given** a patient's status changes to discontinued, **When** the system detects
   this, **Then** an exit-reason feedback request is sent to the patient by email
   automatically, using the same token-based, de-identified mechanism as other
   milestones.
2. **Given** a discontinuation feedback request is generated, **When** its token is
   issued, **Then** it follows the same single-use token rules as every other milestone
   (Story 2) — no separate identity-collecting mechanism is introduced for
   discontinuation.
3. **Given** a discontinued patient does not respond to the emailed request, **When** the
   practice reviews discontinuation data, **Then** the absence of a response is visible
   as such, not silently dropped.

### Edge Cases

- What happens if a patient crosses two different milestone conditions at nearly the
  same time (e.g. a session-count milestone and a cancellation-pattern milestone)? Each
  milestone instance should still be triggered and tokenized independently, so both
  requests go out rather than one silently overwriting the other.
- How does the system handle a token that is never used (patient never responds)? It
  should still expire/invalidate after a defined window rather than remaining valid
  indefinitely, consistent with the short-retention principle.
- What happens if a patient's identity or milestone data changes in CRM after a token has
  already been issued but before the response comes back? The response should still
  rejoin to the correct record as of issuance; the practice needs a documented answer for
  how CRM data changes mid-flight are handled (see Assumptions).
- What happens if a contractor is reassigned a patient's care after a response has
  already been collected and attributed to the prior contractor? The prior attribution
  should not silently transfer to the new contractor.
- What happens if the same patient discontinues and later returns to active care? The
  system must not treat their returning record as invalid or block new milestone
  triggers from firing for them going forward.
- What happens if a prospect (Milestone 0) later becomes a patient? Their prospect-stage
  response and their later patient-stage milestones should both exist without one
  overwriting or being confused with the other.
- What happens if a patient's data continues to meet a clinical safety flag rule on
  multiple consecutive readings? The original flag must remain visible to admins, not get
  silently superseded or lost as later readings come in.

## Requirements *(mandatory)*

### Milestones

Six milestones drive feedback requests. All six trigger off events/data recorded in
Zoho CRM — the practice's EHR is not part of this pipeline's trigger loop. Trigger
thresholds below (day windows, session counts, no-show counts) are the practice's
current starting policy and are configurable, not fixed values defined by this
specification (see Assumptions).

| # | Name | Trigger | Notes |
|---|------|---------|-------|
| 0 | Free consult, no conversion | A free consult is completed and no first appointment is booked within 14 days | Applies to a prospect, not yet a patient |
| 1 | Baseline intake | Automatic at the patient's first session (intake) | Establishes the patient's baseline wellbeing reading, used for later trend comparisons |
| 2 | Early alliance check | Patient's session count reaches 3 | First alliance-only reading; no prior alliance data yet, so its flag rule (below) uses a fixed threshold rather than a trend |
| 3 | Periodic consolidated | Every 4th session, recurring for the duration of treatment | Combines a wellbeing reading, an alliance reading, and operational/contractor items into one request |
| 4 | Discharge | Discharge is marked in the system of record | One-time final reading; closes the pre/post comparison against Milestones 1 and 2 |
| 5 | Discontinuation | Cancellation with no rebooking within 14 days, OR 2 consecutive no-shows with no reschedule in between | Captures exit reason; delivered by email (User Story 5) — see Assumptions for why no live-call path exists in this version |

### Clinical Safety Flag Rules

These rules apply to wellbeing readings (Milestones 1, 3, 4) and alliance readings
(Milestones 2, 3, 4), and drive User Story 4. Thresholds are the practice's current
starting policy, not validated cutoffs, and are configurable (see Assumptions).

- **Wellbeing** — flag for admin review when any of: the total score drops 5 or more
  points from the previous reading; any single domain drops 4 or more points from the
  previous reading; any single domain scores 3 or below at two consecutive readings; or
  the total score has not moved more than ±3 points across 3 consecutive readings
  (stalled).
- **Alliance** — flag for admin review when either: the total alliance score is 20 or
  below (out of 40); or any single domain scores 4 or below.

### Functional Requirements

- **FR-001**: The system MUST detect, without contractor action, when a patient or
  prospect record crosses one of the six milestone conditions defined above, and MUST
  generate a feedback request for that milestone instance automatically.
- **FR-002**: The system MUST NOT send more than one feedback request for the same
  milestone instance for the same patient or prospect.
- **FR-003**: The system MUST generate a single-use token unique to each feedback
  request and MUST NOT include any identifying field (name, email, phone, or other
  direct/indirect identifier) in the collection form presented to the patient or
  prospect.
- **FR-004**: The system MUST match a submitted response's token back to the correct
  record, and this match MUST happen only inside CRM (the system of record), not in the
  collection tool or any other system.
- **FR-005**: The system MUST reject a second submission attempt using an
  already-used token.
- **FR-006**: The system MUST purge the raw submission held by the collection tool
  within a short, defined retention window after the response has been recorded against
  the record in CRM.
- **FR-007**: The system MUST provide a dashboard, restricted to practice admins, with a
  patient-level view showing recorded scores/responses for an individual patient, so
  admins can confirm responses are being collected and interpret an individual result.
- **FR-008**: The system MUST provide, within that same admin dashboard, an overall view
  across all patients that can be filtered to a single clinician's caseload, so admins
  can see both practice-wide trends and any one clinician's trends on demand.
- **FR-009**: The system MUST NOT provide any contractor-facing dashboard or grant
  contractors any access to feedback data in this version; dashboard access is limited
  to practice admins.
- **FR-010**: The system MUST evaluate each wellbeing reading against the Clinical Safety
  Flag Rules above and raise a flag for practice-admin review, identifying the patient
  and the rule that fired, whenever a rule is met.
- **FR-011**: The system MUST evaluate each alliance reading against the Clinical Safety
  Flag Rules above and raise a flag for practice-admin review, identifying the patient
  and the rule that fired, whenever a rule is met.
- **FR-012**: The system MUST NOT automatically notify a contractor of a clinical safety
  flag raised about their own patient. Requesting a case consultation from that
  contractor MUST remain a manual decision made by a practice admin.
- **FR-013**: The system MUST make an active clinical safety flag, and the rule that
  triggered it, visible to admins within that patient's dashboard view (FR-007).
- **FR-014**: The system MUST detect a patient's discontinuation status change and
  automatically send an exit-reason feedback request by email, using the same
  token-based, de-identified mechanism as other milestones.
- **FR-015**: The system MUST make non-response to a feedback request (including
  discontinuation) visible in reporting rather than silently omitting it.
- **FR-016**: The system MUST NOT process, store, or transmit any patient or prospect
  data through a given underlying product until that product's BAA coverage has been
  explicitly confirmed, per the constitution's BAA-gated adoption principle.
- **FR-017**: The system MUST NOT make any data collected through this pipeline
  available to a public-facing website, review platform, social media tool, or marketing
  system, per the constitution's internal-use-only principle.

### Key Entities

- **Patient**: The individual receiving care. Holds identity, care status, session
  history, and milestone state. Lives only in CRM in identified form.
- **Prospect**: An individual who completed a free consult but has not become a patient
  (no first appointment booked). Only Milestone 0 applies to a prospect. A prospect who
  later becomes a patient is a separate record moving into the Patient lifecycle, not a
  conversion of the same record.
- **Milestone Trigger**: A system-detected condition on a patient's or prospect's record,
  matching one of the six defined milestones (0–5, see Milestones above), that causes a
  feedback request to be generated. Each instance is triggered and tracked
  independently.
- **Feedback Token**: A single-use, opaque identifier issued per milestone instance,
  used to link a survey submission back to a patient or prospect without exposing
  identity to the collection tool. Has a validity/expiry window and is invalidated after
  first use.
- **Feedback Response**: The submitted answers/scores for a given milestone instance.
  Exists briefly in raw form in the collection tool, then is rejoined to the record and
  persisted in CRM; the raw copy is purged after a short retention window.
- **Clinical Safety Flag**: A flag raised for admin review when a patient's wellbeing or
  alliance score data meets one of the Clinical Safety Flag Rules above. Never
  automatically shown to the treating contractor; an admin decides whether to request a
  case consultation from that contractor.
- **Contractor (Clinician)**: The care provider associated with a patient's sessions.
  Has no dashboard or feedback-data access in this version, and is never automatically
  notified of a clinical safety flag about their own patient. A later version may
  introduce a restricted, aggregated view of their own caseload, but never raw,
  identified feedback about their own patients.
- **Admin Dashboard**: The single reporting tool for this version, restricted to
  practice admins. Has a patient-level view (scores/responses and any active clinical
  safety flag for one patient) and an overall view across all patients that can be
  filtered to a single clinician's caseload.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of patients/prospects who cross one of the six defined milestone
  conditions receive a feedback request without any manual/contractor-initiated step.
- **SC-002**: 0 feedback submissions ever contain an identifying field visible to or
  stored by the collection tool.
- **SC-003**: 100% of valid, submitted responses are correctly matched to the right
  record in CRM.
- **SC-004**: 0 instances of a contractor account accessing any feedback dashboard or
  feedback data — dashboard access is limited to practice admins in this version.
- **SC-005**: 100% of discontinuation milestones result in an emailed exit-reason
  request.
- **SC-006**: Raw feedback submissions are purged from the collection tool within the
  defined retention window in 100% of completed (rejoined) cases.
- **SC-007**: 100% of readings that meet a Clinical Safety Flag Rule result in a flag
  visible to admins, with 0 automatic notifications sent to the treating contractor.

## Assumptions

- Practice admins (which may be more than one person, and includes the practice owner)
  are the sole audience for the dashboard; no contractor-facing or other role has
  dashboard access in this version.
- "Contractor" and "clinician" are used interchangeably to mean the care-providing
  professional associated with a patient's sessions.
- The six milestones and their trigger conditions (Milestones subsection above) come from
  the practice's Confluence design pages; trigger thresholds (14-day windows, specific
  session counts, no-show counts) and the Clinical Safety Flag Rules' thresholds are the
  practice's current starting policy, explicitly called out in that source as not yet
  validated, and may be tuned once real data exists. Neither is a fixed value defined by
  this specification.
- Milestone 5's original design (per the source doc) used a live phone call as the
  primary collection path, with email only as a fallback triggered after a documented
  failed contact attempt. Per the operations lead's explicit, more recent product
  decision (2026-09-05), this version uses email only: the emailed request fires
  directly on Milestone 5's trigger conditions (cancellation with no rebooking, or a
  no-show pattern), with no live-call step and no call-attempt gate of any kind.
  Confirmed against the live build: the practice's Zoho CRM `Milestone_Instances` module
  does contain leftover picklist values for a live-capture path (`5A - Discontinuation,
  Live Capture`, a `Staff Live Entry` capture method, and a `Captured Live` status). Per
  the operations lead, this was built before the decision to drop live calls and is dead,
  unused configuration — it should be disregarded (and ideally cleaned up in CRM) rather
  than treated as part of the current design.
- The source Confluence doc describes Milestones 1–4 as triggering off the practice's
  EHR (SimplePractice). Per explicit operations-lead correction (2026-09-05), this is
  not accurate: all six milestones (0–5) trigger off events/data recorded in Zoho CRM,
  and the EHR is not part of this pipeline's trigger loop at all. Treat the source doc's
  EHR references as stale on this point; this specification's Milestones table above is
  the current source of truth.
- Which specific products are confirmed under the practice's BAA is a separate,
  still-open decision and is not resolved by this specification — see constitution
  TODO(BAA_SCHEDULE). This spec assumes that confirmation happens before any product is
  wired into the live pipeline, per FR-016.
- The exact wording/content of each milestone's survey questions (e.g. specific slider
  labels, intro/closing copy) is instrument-design content tracked in the source
  Confluence pages, not a functional requirement of this specification.
