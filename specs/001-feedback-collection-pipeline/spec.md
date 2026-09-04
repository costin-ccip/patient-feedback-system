# Feature Specification: Feedback Collection Pipeline

**Feature Branch**: `001-feedback-collection-pipeline`

**Created**: 2026-09-04

**Status**: Draft

**Input**: User description: "Cape Clarity needs an automated, de-identified patient
feedback collection pipeline: feedback requests fire on system-detected milestones (not
left to contractors to remember), the survey tool never sees patient identity, only CRM
ever rejoins a response to a patient, contractors never see their own patients' raw
feedback, and there are two separate reporting dashboards — one aggregate view for the
practice owner and one per-clinician view restricted to that clinician's own caseload.
Discontinuation (a patient leaving care) needs its own exit-reason capture, delivered by
email only for this version — no live-call collection path."

<!--
  Recreated from a description of prior, undocumented work: the repository holding this
  spec was found completely empty at the start of the session that authored this file.
  Content below reconstructs the four user stories, requirements, and scope described by
  the project's operations lead, cross-checked against the constitution
  (.specify/memory/constitution.md) and the source design doc.
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

### User Story 3 - Two separate reporting dashboards (Priority: P3)

As the practice owner, I need an aggregate view across all patients and all contractors
so I can see trends without seeing any single patient's identity. As a contractor, I need
a view of only my own caseload's outcome trends, without ever seeing another
contractor's caseload or my own patients' raw, identified responses.

**Why this priority**: Reporting is what turns collected feedback into something useful,
but it depends entirely on Stories 1 and 2 already working — there's nothing to report on
until requests go out and responses get collected and rejoined.

**Independent Test**: Can be fully tested by seeding feedback data across multiple
contractors and patients, then confirming the practice-owner view shows aggregated,
non-drill-down trends across everyone, while each contractor's own view shows only their
own caseload's trends and never another contractor's data or an identified link to their
own patients' verbatim responses.

**Acceptance Scenarios**:

1. **Given** feedback responses exist across multiple patients and contractors, **When**
   the practice owner opens their dashboard, **Then** they see aggregated trends
   (satisfaction, communication, billing, discontinuation reasons, likelihood-to-refer,
   per-contractor professionalism averages) with no per-patient drill-down available.
2. **Given** a contractor opens their own dashboard, **When** they view their caseload's
   trends, **Then** they see only outcome/alliance trends and flags for their own
   patients, and cannot see any other contractor's caseload.
3. **Given** a contractor's own patient has submitted raw, identified feedback, **When**
   that contractor views their dashboard, **Then** they cannot see that patient's raw
   verbatim response — only aggregated trend data, consistent with the constitution's
   contractor-blindness principle.

---

### User Story 4 - Discontinuation exit-reason capture, email only (Priority: P4)

As the practice, when a patient stops care (discontinuation), we need to capture why —
using the same de-identified, automated flow as every other milestone, delivered by
email only. A live-call collection path is explicitly out of scope for this version.

**Why this priority**: This is a specific milestone instance built on top of Stories 1–3
rather than a separate mechanism; it's the last story because it depends on all three
being in place, and it's the milestone most likely to need product refinement later
(e.g. adding a live-call path) once the core pipeline is proven.

**Independent Test**: Can be fully tested by moving a test patient into discontinuation
status and confirming an exit-reason feedback request is sent by email (using the same
token/de-identification/rejoin mechanism as other milestones), with no live-call step
anywhere in the flow.

**Acceptance Scenarios**:

1. **Given** a patient's status changes to discontinued, **When** the system detects
   this, **Then** an exit-reason feedback request is sent to the patient by email
   automatically, using the same token-based, de-identified mechanism as other
   milestones.
2. **Given** the discontinuation milestone has fired for a patient, **When** the
   feedback pipeline processes it, **Then** no live-call step is scheduled, attempted, or
   required anywhere in the flow — email is the only delivery channel for this version.
3. **Given** a discontinued patient does not respond to the emailed request, **When** the
   practice reviews discontinuation data, **Then** the absence of a response is visible
   as such (not silently dropped), without triggering any live-call fallback.

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
- **FR-007**: The system MUST provide a practice-wide dashboard, restricted to the
  practice owner, showing aggregated trends across all patients and contractors with no
  per-patient drill-down.
- **FR-008**: The system MUST provide a per-contractor dashboard, restricted by
  row-level access so each contractor sees only their own caseload's outcome/alliance
  trends and flags.
- **FR-009**: The system MUST NOT expose a contractor's own patient's raw, identified
  feedback response to that contractor, in either dashboard or any other view.
- **FR-010**: The system MUST detect a patient's discontinuation status change and
  automatically send an exit-reason feedback request by email, using the same
  token-based, de-identified mechanism as other milestones.
- **FR-011**: The system MUST NOT include or require a live-call step for
  discontinuation exit-reason capture in this version.
- **FR-012**: The system MUST make non-response to a feedback request (including
  discontinuation) visible in reporting rather than silently omitting it.
- **FR-013**: The system MUST NOT process, store, or transmit any patient data through a
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
  Has access to an own-caseload dashboard but never to raw, identified feedback about
  their own patients.
- **Practice Dashboard**: Aggregate reporting view, practice-owner only, no per-patient
  drill-down.
- **Clinician Dashboard**: Per-contractor reporting view, restricted to that
  contractor's own caseload via row-level access.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of patients who cross a defined milestone condition receive a
  feedback request without any manual/contractor-initiated step.
- **SC-002**: 0 feedback submissions ever contain a patient-identifying field visible to
  or stored by the collection tool.
- **SC-003**: 100% of valid, submitted responses are correctly matched to the right
  patient in CRM.
- **SC-004**: 0 instances of a contractor viewing their own patient's raw, identified
  feedback response.
- **SC-005**: 100% of discontinuation milestones result in an emailed exit-reason
  request, with 0 live-call steps required or attempted.
- **SC-006**: Raw feedback submissions are purged from the collection tool within the
  defined retention window in 100% of completed (rejoined) cases.

## Assumptions

- The practice owner is the sole audience for the aggregate practice dashboard; no
  additional aggregate-view roles (e.g. office manager) are in scope for this version.
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
  wired into the live pipeline, per FR-013.
- A live-call discontinuation path is a deliberately deferred, later enhancement, not an
  oversight — it is out of scope for this version by product decision.
