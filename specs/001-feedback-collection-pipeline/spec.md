# Feature Specification: Feedback Collection Pipeline

**Feature Branch**: `001-feedback-collection-pipeline`

**Created**: 2026-09-04

**Status**: Draft

**Input**: User description: "Cape Clarity needs an automated, de-identified patient
feedback collection pipeline: feedback requests fire on system-detected milestones (not
left to contractors to remember), the survey tool never sees patient identity, only CRM
ever rejoins a response to a patient, contractors have no access to feedback data or
dashboards in the first version, and there is one admin-only reporting dashboard with a
patient-level view (scores per patient) and an overall view across all patients that can
be filtered by clinician. Discontinuation (a patient leaving care) needs its own
exit-reason capture, delivered by email."

<!--
  Recreated from a description of prior, undocumented work: the repository holding this
  spec was found completely empty at the start of the session that authored this file.
  Content below reconstructs the four user stories, requirements, and scope described by
  the project's operations lead, cross-checked against the constitution
  (.specify/memory/constitution.md) and the source design doc.

  Revised 2026-09-05 per operations-lead feedback: (1) contractors have no dashboard
  access at all in v1 — dashboards are admin-only; (2) the dashboard is redefined as one
  admin-facing tool with a patient-level view and a clinician-filterable overall view,
  rather than two separate dashboards; (3) all references to a live-call discontinuation
  path removed — that was an old requirement no longer under consideration.
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Feedback requests fire automatically on milestones (Priority: P1)

As the practice, we need feedback requests to go out the moment a patient hits a defined
point in their care journey (e.g. a session-count threshold, a status change, a
cancellation/no-show pattern), without any clinician having to remember to ask or
manually kick it off.

**Why this priority**: Every other part of this feature depends on requests actually
going out reliably. If sending is contractor-dependent, coverage silently degrades to
whichever contractors remember to do it, and the whole feedback program becomes
unreliable and biased toward whoever opts in. This is the foundation everything else is
built on.

**Independent Test**: Can be fully tested by advancing a test patient record through
each defined milestone condition and confirming a feedback request is generated and sent
for each one, with zero manual action from a contractor at any point.

**Acceptance Scenarios**:

1. **Given** a patient's record crosses a defined milestone condition (e.g. session
   count reaches the configured threshold for that milestone), **When** the system next
   evaluates that record, **Then** a feedback request for that milestone is generated and
   sent automatically, with no clinician action required.
2. **Given** a patient has already received a feedback request for a given milestone
   instance, **When** the same milestone condition is still true on a later check,
   **Then** the system does not send a duplicate request for that same milestone
   instance.
3. **Given** a contractor takes no action at all with respect to feedback, **When** their
   patients cross milestone conditions, **Then** feedback requests still go out for those
   patients on schedule.

---

### User Story 2 - Feedback is collected de-identified and rejoined only in CRM (Priority: P2)

As the practice, we need patients to be able to answer honestly without the survey tool
itself ever knowing who they are, while still being able to tell, afterward, which
patient a given response belongs to — with that link made only inside the system of
record (CRM), not inside the survey tool.

**Why this priority**: This is the core privacy mechanism the entire program depends on
for trust and compliance. It's ordered after automated triggers because a trigger has to
exist before there's anything to collect, but it's more foundational than the reporting
built on top of it.

**Independent Test**: Can be fully tested by sending a feedback request, submitting a
response, and confirming (a) the survey tool's stored submission never contained a name,
email, or other direct identifier — only an opaque token — and (b) the practice can still
retrieve that response correctly matched to the right patient afterward, with the match
happening in CRM.

**Acceptance Scenarios**:

1. **Given** a feedback request is generated for a patient, **When** the request is sent,
   **Then** it includes a single-use token unique to that request, and no patient-identifying
   field is included in the collection form itself.
2. **Given** a patient submits a response using their token, **When** the submission is
   received, **Then** the system matches the token back to the correct patient record
   inside CRM, and the response's scores/answers are recorded against that patient there.
3. **Given** a token has already been used for a submission, **When** the same token is
   submitted again, **Then** the system rejects the second submission as invalid.
4. **Given** a response has been matched back to a patient in CRM, **When** enough time
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

### User Story 4 - Discontinuation exit-reason capture (Priority: P4)

As the practice, when a patient stops care (discontinuation), we need to capture why —
using the same de-identified, automated flow as every other milestone, delivered by
email.

**Why this priority**: This is a specific milestone instance built on top of Stories 1–3
rather than a separate mechanism; it's the last story because it depends on all three
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
  rejoin to the correct patient record as of issuance; the practice needs a documented
  answer for how CRM data changes mid-flight are handled (see Assumptions).
- What happens if a contractor is reassigned a patient's care after a response has
  already been collected and attributed to the prior contractor? The prior attribution
  should not silently transfer to the new contractor.
- What happens if the same patient discontinues and later returns to active care? The
  system must not treat their returning record as invalid or block new milestone
  triggers from firing for them going forward.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST detect, without contractor action, when a patient record
  crosses a defined milestone condition, and MUST generate a feedback request for that
  milestone instance automatically.
- **FR-002**: The system MUST NOT send more than one feedback request for the same
  milestone instance for the same patient.
- **FR-003**: The system MUST generate a single-use token unique to each feedback
  request and MUST NOT include any patient-identifying field (name, email, phone, or
  other direct/indirect identifier) in the collection form presented to the patient.
- **FR-004**: The system MUST match a submitted response's token back to the correct
  patient record, and this match MUST happen only inside CRM (the system of record), not
  in the collection tool or any other system.
- **FR-005**: The system MUST reject a second submission attempt using an
  already-used token.
- **FR-006**: The system MUST purge the raw submission held by the collection tool
  within a short, defined retention window after the response has been recorded against
  the patient in CRM.
- **FR-007**: The system MUST provide a dashboard, restricted to practice admins, with a
  patient-level view showing recorded scores/responses for an individual patient, so
  admins can confirm responses are being collected and interpret an individual result.
- **FR-008**: The system MUST provide, within that same admin dashboard, an overall view
  across all patients that can be filtered to a single clinician's caseload, so admins
  can see both practice-wide trends and any one clinician's trends on demand.
- **FR-009**: The system MUST NOT provide any contractor-facing dashboard or grant
  contractors any access to feedback data in this version; dashboard access is limited
  to practice admins.
- **FR-010**: The system MUST detect a patient's discontinuation status change and
  automatically send an exit-reason feedback request by email, using the same
  token-based, de-identified mechanism as other milestones.
- **FR-011**: The system MUST make non-response to a feedback request (including
  discontinuation) visible in reporting rather than silently omitting it.
- **FR-012**: The system MUST NOT process, store, or transmit any patient data through a
  given underlying product until that product's BAA coverage has been explicitly
  confirmed, per the constitution's BAA-gated adoption principle.

### Key Entities

- **Patient**: The individual receiving care. Holds identity, care status, session
  history, and milestone state. Lives only in CRM in identified form.
- **Milestone Trigger**: A system-detected condition on a patient's record (e.g. session
  count, status change, cancellation pattern, discontinuation) that causes a feedback
  request to be generated. Each instance is triggered and tracked independently.
- **Feedback Token**: A single-use, opaque identifier issued per milestone instance,
  used to link a survey submission back to a patient without exposing identity to the
  collection tool. Has a validity/expiry window and is invalidated after first use.
- **Feedback Response**: The patient's submitted answers/scores for a given milestone
  instance. Exists briefly in raw form in the collection tool, then is rejoined to the
  patient and persisted in CRM; the raw copy is purged after a short retention window.
- **Contractor (Clinician)**: The care provider associated with a patient's sessions.
  Has no dashboard or feedback-data access in this version. A later version may
  introduce a restricted, aggregated view of their own caseload, but never raw,
  identified feedback about their own patients.
- **Admin Dashboard**: The single reporting tool for this version, restricted to
  practice admins. Has a patient-level view (scores/responses for one patient) and an
  overall view across all patients that can be filtered to a single clinician's
  caseload.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of patients who cross a defined milestone condition receive a
  feedback request without any manual/contractor-initiated step.
- **SC-002**: 0 feedback submissions ever contain a patient-identifying field visible to
  or stored by the collection tool.
- **SC-003**: 100% of valid, submitted responses are correctly matched to the right
  patient in CRM.
- **SC-004**: 0 instances of a contractor account accessing any feedback dashboard or
  feedback data — dashboard access is limited to practice admins in this version.
- **SC-005**: 100% of discontinuation milestones result in an emailed exit-reason
  request.
- **SC-006**: Raw feedback submissions are purged from the collection tool within the
  defined retention window in 100% of completed (rejoined) cases.

## Assumptions

- Practice admins (which may be more than one person, and includes the practice owner)
  are the sole audience for the dashboard; no contractor-facing or other role has
  dashboard access in this version.
- "Contractor" and "clinician" are used interchangeably to mean the care-providing
  professional associated with a patient's sessions.
- Milestone condition thresholds (e.g. specific session counts) are configurable
  practice policy, not fixed values defined by this specification.
- How CRM's underlying session-count/status fields get populated (manual entry vs.
  derived from a scheduling system) is a separate, still-open decision and is not
  resolved by this specification — see constitution TODO(TRIGGER_SOURCE).
- Which specific products are confirmed under the practice's BAA is a separate, still-open
  decision and is not resolved by this specification — see constitution
  TODO(BAA_SCHEDULE). This spec assumes that confirmation happens before any product is
  wired into the live pipeline, per FR-012.
- The exact set of milestones (which conditions exist, and their trigger thresholds) is
  tracked separately and is not yet enumerated in this specification — see the open
  question raised alongside this revision.
