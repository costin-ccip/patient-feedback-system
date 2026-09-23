# Feature Specification: Issuance Privacy and Send-Failure Handling

**Feature Branch**: `003-issuance-privacy-and-failure-handling`

**Created**: 2026-09-23

**Status**: Draft

**Input**: Costin's decisions "1B and 2B" on the two findings in
`specs/002-remove-subflow-dependency/research.md` §5:

- **1B**: stop putting the patient/prospect email in the feedback-request record's
  name, and keep the email out of Zoho Analytics.
- **2B**: when the feedback email fails to send, mark the record as a send failure
  and alert an admin, instead of leaving a silent "Issued" record. Alert recipient:
  `costin@capeclarity.com` (Costin, 2026-09-23).

<!--
  Builds on 002's shared issuance function (`issueFeedbackToken`) and per-flow Zoho
  Mail step. Doesn't change any 001 requirement; it tightens FR-015 (non-response
  visible, not silently dropped) and Principles I/II (identity stays in CRM).
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Feedback records don't carry patient emails outside the CRM (Priority: P1)

As the practice, I want each feedback-request record to be named without the
patient's or prospect's email address, so the reporting copy of these records
(Analytics) holds tokens and scores but no contact details.

**Why this priority**: It's a privacy fix to data that's already accumulating, and
the smallest change of the two.

**Independent Test**: Inspect the issuance logic and the existing records: new
records are named from the milestone and a short token prefix; no existing record's
name contains an email address; the Analytics sync for these records doesn't include
the Email field.

**Acceptance Scenarios**:

1. **Given** a feedback request is issued, **When** its record is created, **Then**
   the record name is the milestone plus the first 8 characters of its token, with no
   email address.
2. **Given** records created before this change, **When** the change is applied,
   **Then** each of them is renamed to the same format.
3. **Given** the Analytics sync for these records, **When** it's inspected, **Then**
   the Email field isn't synced.
4. **Given** the CRM record, **When** an admin opens it, **Then** it still shows who
   it was sent to (the patient/lead link and the Email field stay in the CRM).

---

### User Story 2 - A failed feedback email is visible and alerts an admin (Priority: P1)

As the practice admin, when a feedback email can't be sent, I want the record
marked as a send failure and an alert in my inbox, so I can fix the address and send
it by hand, and so the missed request isn't counted as a patient who chose not to
reply.

**Why this priority**: Without this, M2 and M3 silently skip that patient's check-in
forever (their duplicate checks see the record as already issued), and reporting
counts the failure as a non-response.

**Independent Test**: Inspect each milestone trigger flow: its email step has a
failure path that marks the record "Send Failed" and sends an alert to
`costin@capeclarity.com` containing only the milestone, the record ID and a short
reason. The issuance step also has a failure path that alerts (no record exists in
that case, so nothing is marked).

**Acceptance Scenarios**:

1. **Given** the feedback email step fails, **When** the flow handles the failure,
   **Then** the record's status becomes "Send Failed" and an alert is sent to
   `costin@capeclarity.com`.
2. **Given** the issuance step fails before a record exists, **When** the flow
   handles the failure, **Then** an alert is sent and no feedback email is sent.
3. **Given** an alert is sent, **When** it's read, **Then** it contains no patient or
   prospect name, email, or phone, only the milestone, the CRM record ID (if any), and
   what failed.
4. **Given** a record is "Send Failed", **When** the patient's survey link is used,
   **Then** it's rejected (only "Issued" tokens are accepted) until an admin resends
   and sets it back to "Issued".
5. **Given** reporting on a milestone, **When** statuses are counted, **Then** "Send
   Failed" shows as its own category, separate from unanswered "Issued"/"Expired".

### Edge Cases

- **Failure path itself fails** (e.g. the mail connection is down, so the alert can't
  send either): the record still gets marked "Send Failed" first, so it's visible in
  reporting even without the alert.
- **Admin resend**: M2/M3's duplicate checks count the failed record, so the flow
  won't re-issue on its own. Resending is a documented manual step (runbook in
  `quickstart.md`), not automated here.
- **Old-format names with emails in Analytics**: the Analytics copy updates on its
  next daily sync after the rename.
- **Retired flows** (M1, the retired subflow) are left alone.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: New feedback-request records MUST be named
  `Milestone <milestone> - <first 8 characters of token>`.
- **FR-002**: Every existing feedback-request record whose name contains an email
  address MUST be renamed to the FR-001 format.
- **FR-003**: The Email field of feedback-request records MUST NOT be synced to Zoho
  Analytics. (The record name can't be excluded from the sync; FR-001/002 cover it.)
- **FR-004**: A new record status "Send Failed" MUST exist.
- **FR-005**: In every active milestone trigger flow (M0, M2, M3), a failure of the
  feedback email step MUST set that record's status to "Send Failed", then send an
  alert to `costin@capeclarity.com`.
- **FR-006**: In every active milestone trigger flow, a failure of the issuance step
  MUST send an alert to `costin@capeclarity.com`; no feedback email is sent.
- **FR-007**: Alerts MUST NOT contain patient/prospect name, email, phone or survey
  answers; they may contain the milestone, CRM record ID and failure description.
- **FR-008**: Alerts MUST go through the practice's existing Zoho Mail connection (no
  new sending channel, per constitution Principle V).
- **FR-009**: No flow is switched ON and no live test is run by the building session.
- **FR-010**: The resend procedure for a "Send Failed" record MUST be documented.
- **FR-011**: The CLAUDE.md issuance convention MUST be updated so M4/M5 include the
  same failure paths.

### Key Entities

- **Feedback-request record** (Milestone_Instances): gains a new status value "Send
  Failed" and a new naming format. No new fields.
- **Admin alert**: an email to `costin@capeclarity.com`, de-identified.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 0 feedback-request records have an email address in their name after
  the change (checked by query).
- **SC-002**: 0 patient/prospect emails in the Analytics copy of these records after
  the next daily sync.
- **SC-003**: 3 of 3 active trigger flows have both failure paths (issuance and
  email).
- **SC-004**: In Costin's live test, a forced email failure produces 1 "Send Failed"
  record and 1 alert with no identifying details.

## Assumptions

- The Analytics sync already excludes the Email field (confirmed 2026-09-23 in the
  sync setup). The record name can't be deselected there (it's greyed out as
  required), which is why renaming is the fix for the name.
- Zoho Flow's per-step "On Error" branch runs when an action fails; confirmed
  structurally here, behaviorally only in the live test.
- Found during research, outside this feature: the Analytics sync also imports the
  **Leads** module including Last Name and Email (PHI) columns, and records carry
  a Lead Reference that joins to it. Flagged for Costin; not changed here.
