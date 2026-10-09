# Feature Specification: Feedback Reminders

**Feature Branch**: `007-feedback-reminders`

**Created**: 2026-10-09

**Status**: Draft (design decisions answered by Costin 2026-10-09; nothing built except one CRM field)

**Input**: User description: "Think through a follow-up mechanism for when a person does not
respond to the feedback email. Propose follow-up behavior by milestone, considering all
milestones together."

<!--
  Scoping decisions (Costin, 2026-10-09), made after a milestone-by-milestone proposal:

  1. ONE reminder per feedback request, sent at day 4 of the 7-day token lifetime (3 days
     before the link expires). No second reminder. The token lifetime stays 7 days.
  2. All five active milestones (0, 2, 3, 4, 5) get a reminder, including M0 (prospects)
     and M5 (discontinuation).
  3. Milestone 3 gets a "fatigue brake": no reminder if the person's previous feedback
     request went unanswered.
  4. Reminder state is stored in one new custom field on Milestone_Instances, freed by
     Costin for this purpose and created 2026-10-09 (see data-model.md section 1).
  5. Liana approves the reminder wording per milestone, as she did the opt-out page text
     in feature 006.

  Related, not duplicated: feature 006 (opt-out) governs who may receive feedback emails;
  a reminder is a feedback email, so 006 applies to it in full (FR-005, FR-006 below).
  Feature 002 (shared token issuance) is deliberately NOT reused or changed: a reminder
  re-uses the original token and row and never issues a new one (FR-002).
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A person who has not answered gets one gentle reminder (Priority: P1)

As the practice, when a patient or prospect has been sent a feedback request and has not
answered by day 4 of the 7 days the link stays valid, we want one automatic, low-key
reminder sent to the same address with the same link, so that a missed email does not
silently become a missing data point, without any clinician or staff member having to
remember to chase it.

**Why this priority**: Without it, a single overlooked email permanently loses that
milestone's data point. Response rates drive how useful every milestone's reporting is,
and the practice has no other follow-up path (contractors must not chase their own
patients, constitution Principle III).

**Independent Test**: Seed a test `Milestone_Instances` row (status Issued, created 4 or
more days ago, still unexpired, no reminder sent) for the designated test patient. Run
the reminder flow and confirm exactly one reminder email goes to the test address, the
link inside it is the original token link, the row's reminder timestamp is set, and a
second run sends nothing more for that row.

**Acceptance Scenarios**:

1. **Given** a feedback request is still `Issued`, has not expired, has had no reminder,
   and 4 or more days have passed since it was issued, **When** the reminder flow next
   runs, **Then** one reminder email is sent to the same address and the row is marked as
   reminded.
2. **Given** a reminder has already been sent for a feedback request, **When** the flow
   runs again, **Then** no second reminder is sent for that request, now or on any later
   run.
3. **Given** a feedback request is less than 4 days old, **When** the flow runs, **Then**
   no reminder is sent for it yet.
4. **Given** a reminder is sent, **When** the person opens the link, **Then** it is the
   same single-use token link as the original email; submitting through it works exactly
   as it would have from the original email, and the response is rejoined in CRM as
   usual (spec 001 Story 2).
5. **Given** a contractor takes no action, **When** one of their patients has not
   responded by day 4, **Then** the reminder still goes out, and the contractor is not
   told whether a reminder was sent or whether the patient responded.

---

### User Story 2 - Reminders never reach people who should not get them (Priority: P2)

As the practice, we want a reminder suppressed whenever sending it would be wrong:
the person already answered, the link already expired or was replaced, the person
opted out of feedback emails, or (for two milestones) a milestone-specific condition
says to leave them alone.

**Why this priority**: A reminder to someone who already answered, or who told us to stop,
is worse than no reminder: it erodes trust and, for opt-outs, breaks a promise made on
the opt-out page. It is ordered after Story 1 because there must be a reminder before it
can be suppressed.

**Independent Test**: Seed test rows covering each suppression case below and confirm the
flow sends nothing for any of them, while still sending for a control row that qualifies.

**Acceptance Scenarios**:

1. **Given** the request's status is `Submitted`, `Expired`, `Superseded`, `Send Failed`
   or `Skipped - Opted Out`, **When** the flow runs, **Then** no reminder is sent.
2. **Given** a request is still `Issued` but its expiry time has passed, **When** the flow
   runs, **Then** no reminder is sent (a reminder never goes out for a dead link).
3. **Given** the person has opted out of feedback emails (feature 006) at any point
   before the reminder would be sent, **When** the flow runs, **Then** no reminder is
   sent, even though the original request went out before the opt-out.
4. **Given** the request is for Milestone 5 (discontinuation) and the patient's status is
   no longer "Discontinued (Patient Choice)" (they returned to care), **When** the flow
   runs, **Then** no reminder is sent.
5. **Given** the request is for Milestone 3 and the patient's immediately preceding
   feedback request (any milestone, ignoring superseded and skipped rows) was never
   answered, **When** the flow runs, **Then** no reminder is sent for this M3 request.
   The original M3 request is unaffected; only its reminder is held back.
6. **Given** a person's record cannot be read when the flow checks it, **When** the flow
   runs, **Then** it does not send (it fails closed) and Costin receives an identity-free
   alert.

---

### User Story 3 - Reminders keep every existing privacy guarantee (Priority: P2)

As the practice, we need reminders to obey the same rules as every other feedback email:
de-identified collection, a single rejoin point, no contractor visibility, and an
opt-out link in every email.

**Why this priority**: These are the constitution's non-negotiables. A reminder path that
quietly weakened them would undo the pipeline's central design.

**Independent Test**: Inspect a sent test reminder and the flow's alerts and logs; confirm
no name, phone or clinical content appears, the unsubscribe link works from the reminder,
and no contractor-visible surface changed.

**Acceptance Scenarios**:

1. **Given** a reminder email, **When** inspected, **Then** its only link payload is the
   same opaque token, it contains no name, clinical detail or reference to the
   clinician, and it carries the same "stop these emails" link as the original email
   (feature 006).
2. **Given** any failure in the reminder flow, **When** an alert is raised, **Then** the
   alert contains no patient or prospect identity and no token (same rule as feature 003).
3. **Given** a reminder is sent, **When** a contractor looks at any system they can
   reach, **Then** nothing reveals that the reminder exists or that the person
   responded (constitution Principle IV; contractors have no CRM or dashboard access).

---

### User Story 4 - Admins can see whether reminders help (Priority: P3)

As a practice admin, I want reporting that shows how many requests were reminded and
whether reminded requests get answered, per milestone, so I can tell whether the
reminder is worth sending and tune its timing, without opening records.

**Why this priority**: The reminder works without reporting, but spec 001 FR-015, FR-018
and FR-019 require non-response and per-milestone summaries to be visible. A new
mechanism that changes who responds must be measurable.

**Independent Test**: With seeded rows spanning reminded and unreminded requests and all
statuses, open the new report and confirm counts by milestone for issued, reminded,
submitted-after-reminder, submitted-without-reminder, and lapsed with no response; confirm
opted-out skips are not counted as non-response.

**Acceptance Scenarios**:

1. **Given** feedback requests exist, **When** an admin opens the reminder report,
   **Then** for each milestone it shows counts of requests issued, reminded, answered
   after a reminder, answered without one, and lapsed with no answer.
2. **Given** a request is `Issued` with an expiry time in the past, **When** reporting
   runs, **Then** it is shown as lapsed (no response), not as still open, even though
   its CRM status is still `Issued`.
3. **Given** a request was skipped because the person opted out, **When** reporting
   runs, **Then** it is not counted as a non-response (spec 006 FR-010).

### Edge Cases

- A person submits the survey between the flow selecting their row and the email being
  sent: the row is claimed first, then sent; a response arriving in that window is still
  accepted (the token is still valid). They may receive a reminder moments after
  responding. Accepted, because the flow cannot be atomic across Forms and CRM.
- A new M3 checkpoint request supersedes an earlier M3 request still `Issued`: the old row
  becomes `Superseded` and is no longer eligible; the new row earns its own reminder at
  its own day 4.
- A person has open requests for two different milestones at once (for example M2 and M3
  close together): each is reminded independently. Whether to hold one back is an open
  question (see Assumptions).
- A person opts out after a reminder was already sent: nothing is recalled; the already-sent
  link runs out its normal lifetime (same as feature 006).
- A person re-subscribes while a request is still open and unreminded: they become
  eligible again and may receive the reminder. Staff-only resubscribe makes this rare.
- A reminder email fails to send: the original request is untouched and its token stays
  valid. Costin is alerted. The reminder is not retried (FR-009).
- The flow does not run for a day (outage): rows are reminded on the next run if still
  inside the window; a row can lose its reminder only if the link expires first.
- Legacy-patient cutoff (feature 004): unaffected. Reminders only ever apply to requests
  that were issued, and the cutoff gates issuance.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST send at most one automatic reminder per feedback request
  (per `Milestone_Instances` row), for each of Milestones 0, 2, 3, 4 and 5, when the request
  is still `Issued`, unexpired, unreminded, and at least 4 days old.
- **FR-002**: A reminder MUST re-use the original request's token and `Milestone_Instances`
  row. The system MUST NOT issue a new token, create a new row, or call the shared
  issuance function for a reminder. (This is a clarification of spec 001 FR-002: "no more
  than one feedback request per milestone instance" now reads "one request, plus at
  most one reminder on the same token".)
- **FR-003**: The token lifetime MUST remain 7 days. A reminder MUST NOT extend, reset or
  otherwise change the request's expiry.
- **FR-004**: The system MUST record that a reminder was dispatched, with its date and
  time, on the request's row, and MUST use that record to guarantee no second reminder.
- **FR-005**: The system MUST NOT send a reminder to anyone who has opted out of feedback
  emails (feature 006), judged on the person's current opt-out state at the moment of
  sending, not at the moment the request was issued.
- **FR-006**: Every reminder MUST carry the same "stop these emails" link as the original
  request email (feature 006 FR-001), built from the same token.
- **FR-007**: The system MUST NOT send a reminder for a request whose status is anything
  other than `Issued`, or whose expiry time has passed.
- **FR-008**: For Milestone 5, the system MUST NOT send a reminder if the patient's status
  is no longer "Discontinued (Patient Choice)". For Milestone 3, the system MUST NOT
  send a reminder if the patient's immediately preceding feedback request (any
  milestone; excluding `Superseded`, `Skipped - Opted Out` and `Send Failed` rows) was not
  answered, meaning its status is `Expired` or it is `Issued` past expiry.
- **FR-009**: A failure while checking eligibility MUST stop the send (fail closed), and a
  failure while sending MUST NOT be retried. Either MUST raise an alert to Costin that
  contains no patient or prospect identity and no token. A failed reminder MUST NOT
  change the original request's status or token validity.
- **FR-010**: Reminders MUST be sent only within a defined daytime window (9 AM to 5 PM
  Eastern) so they do not arrive at night.
- **FR-011**: The reminder decision MUST be fully automatic. No clinician or staff member
  initiates, approves, sees or can suppress an individual reminder (constitution
  Principle III). Suppression happens only through the rules above.
- **FR-012**: The reminder email body MUST contain no name, clinical detail (diagnosis,
  treatment or session content) or clinician reference, and MUST NOT change the survey's collection form or its token handling
  (constitution Principles I and II).
- **FR-013**: The reminder mechanism MUST use only products already part of the pipeline
  (Zoho CRM, Flow, Mail, Analytics, Forms) and MUST NOT go live before their BAA coverage
  is confirmed (spec 001 FR-016, constitution Principle V).
- **FR-014**: The reminder mechanism MUST NOT make any pipeline data available to a
  website, review platform or marketing system (spec 001 FR-017, Principle VI). A
  reminder is an operational follow-up to a request already made; it MUST NOT be
  worded as a request for a testimonial or review.
- **FR-015**: Reporting MUST show, per milestone, requests issued, reminded, answered
  after a reminder, answered without one, and lapsed with no answer; it MUST treat an
  `Issued` row past expiry as lapsed; and it MUST NOT count opt-out skips as
  non-response. Derived values MUST be computed in Zoho Analytics (Principle VII).
- **FR-016**: Each milestone's reminder wording MUST be approved by Liana before use. The
  five wordings MAY differ in tone; M0 and M5 use the softest register.

### Key Entities

- **Reminder**: The single follow-up email for one `Milestone_Instances` row. It is not its
  own record; it exists as a timestamp on the row (`Reminder_Sent_Date_Time`) plus the
  email that was sent.
- **Feedback Request (Milestone_Instances row)**: unchanged from specs 001 and 002, plus
  the one new field.
- **Reminder Window**: from 4 days after issuance until expiry (day 7), sent between 9 AM
  and 5 PM Eastern.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of requests that are `Issued`, unexpired, unreminded, 4 or more days old
  and eligible under FR-005 and FR-008 receive exactly one reminder inside the window,
  barring an outage longer than the remaining lifetime.
- **SC-002**: 0 requests ever receive a second reminder.
- **SC-003**: 0 reminders go to a person who was opted out at send time, 0 go for a
  non-`Issued` or expired request, and 0 go for an M5 patient no longer discontinued.
- **SC-004**: 0 reminder emails, alerts or logs contain a name, clinical content or token
  in an alert.
- **SC-005**: 0 contractor-visible surfaces show reminder state.
- **SC-006**: Reporting shows reminded versus unreminded response rates for every active
  milestone, within one Analytics sync of a reminder being sent.
- **SC-007**: A person who clicks "stop these emails" in a reminder is opted out exactly as
  from the original email (feature 006 SC criteria hold for reminders).

## Assumptions

- The 4-of-7 timing, single reminder, all-milestone coverage, M3 brake, and the use of one
  `Milestone_Instances` field were decided by Costin on 2026-10-09 and are policy, not
  validated values. The reminder day and the brake rule are configuration constants in
  the one function that decides eligibility, so they can be tuned once response data
  exists.
- "Day 4 of 7" is implemented as "3 or fewer days of link life remain", read from the
  request's expiry time, because that value can be set for testing while the creation time
  cannot. The two are the same while the token lifetime is 7 days; if the lifetime ever
  changes, the reminder constant must be updated with it (data-model.md section 2).
- Zoho Flow is on the Standard plan with no loops or subflows (`CLAUDE.md`), so one
  scheduled run can safely send one reminder; the schedule runs several times a day to
  clear a day's backlog (plan.md D2).
- "Unanswered" for the M3 brake means the previous request was not `Submitted`. A previous
  request still inside its own 7 days (still open) is treated as not yet unanswered, so
  the brake only holds a reminder back once the earlier request has actually lapsed.
- Open question, not decided: when one person has two open requests for different
  milestones (for example M2 and M3 issued close together), both are reminded
  independently in this version. Costin may want a rule that holds one back; see
  plan.md Open Questions.
- Reminder state is deliberately not shown to contractors; contractors have no CRM or
  dashboard access in this version (spec 001 FR-009), and this feature adds none.
- A reminder never changes a milestone's trigger, cutoff, idempotency or clinical safety
  flag logic. Non-response is not itself a clinical safety flag.
