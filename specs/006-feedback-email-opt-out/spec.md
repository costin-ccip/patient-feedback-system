# Feature Specification: Feedback Email Opt-Out

**Feature Branch**: `006-feedback-email-opt-out`

**Created**: 2026-10-01

**Status**: Draft (open questions answered 2026-10-01)

**Input**: User description: "Currently our feedback system works, but there's no way for
patients to unsubscribe if they wish to stop receiving these emails. I'd like to create
that. And since we're in the process of setting up Zoho Campaigns, which will have an
email preferences center, I believe the 'feedback emails' should be a category in there."

<!--
  Scoping decisions (Costin, 2026-10-01), made after reviewing the Zoho Campaigns spec
  repo (costin-ccip/cape-clarity-zoho-campaigns) and weighing three designs:

  1. WHERE THE OPT-OUT LIVES: CRM-owned, with its own link in each feedback email. NOT a
     native category in the Campaigns preference center. Reasons found in review: (a) the
     Campaigns patient audience holds only people who opted in to MARKETING and is gated
     on a BAA confirmation that is still pending, while feedback emails go to every
     patient who hits a milestone, so most recipients would never be in Campaigns; (b)
     feedback emails are sent by Zoho Flow through Zoho Mail, not by Campaigns, so a
     Campaigns category would not be consulted by the pipeline without a sync back; (c)
     both projects wall feedback off from marketing (this repo's constitution Principle
     VI, and Principle VI of the Campaigns repo), and loading every patient into the
     marketing tool to track a feedback preference would cut against that wall. A link
     from Campaigns' preference center to this opt-out can be added later as a pointer
     (see Out of Scope).
  2. GRANULARITY: one opt-out covers all feedback emails (M0 and M2-M5), not per milestone.
  3. CHANNELS: (a) the link in the email, (b) staff recording it in CRM when a patient
     asks by phone or in person, (c) a "please stop" reply to the sending mailbox,
     processed by staff into CRM. No automation of (b) or (c) beyond the CRM field.
  4. RESUBSCRIBE: staff-only (a patient who changes their mind tells the practice; staff
     clear the opt-out in CRM). No patient self-serve resubscribe in this version.

  Related, not duplicated: the Campaigns spec's unsubscribe/manage-subscription rules
  (its FR-009 to FR-013) govern marketing email and are unaffected. The two opt-outs are
  independent by design (see Assumptions).
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Stop feedback emails from the email itself (Priority: P1)

As a patient or prospect who has received a feedback request and does not want any more,
I can stop all further feedback emails from Cape Clarity directly from the email, without
logging in, remembering anything, or typing who I am.

**Why this priority**: This is the core of the request and the main way a recipient can
say no. Without it, the only way to stop is to complain to the practice, which is a poor
experience and a compliance and trust risk for a health practice.

**Independent Test**: Issue a test feedback request to a test patient. Open the
unsubscribe link in the email, confirm, and verify the confirmation page, the CRM opt-out
on that patient's record only, and that no feedback identity (name, email) appeared on any
page of the flow.

**Acceptance Scenarios**:

1. **Given** any feedback email from any milestone (0, 2, 3, 4, 5), **When** it is
   delivered, **Then** it contains a clearly labelled link to stop receiving feedback
   emails, in addition to the survey link.
2. **Given** a recipient opens that link, **When** the page loads, **Then** it shows an
   explanation in plain language and one explicit confirmation action, and does NOT yet
   record the opt-out (so automated link scanners that fetch the page cannot opt anyone
   out).
3. **Given** the recipient confirms, **When** the request is processed, **Then** the
   opt-out is recorded on the correct CRM record (and only there), the recipient sees a
   confirmation message, and no name, email address, or other identifier is shown or
   asked for at any step.
4. **Given** the recipient opens or confirms the link a second time, **When** it is
   processed, **Then** they see the same confirmation and nothing breaks or duplicates.
5. **Given** a feedback email sent more than 7 days ago (its survey token has expired) or
   whose survey was already submitted, **When** the recipient uses its unsubscribe link,
   **Then** it still works.
6. **Given** a link that cannot be matched to anyone (altered, truncated, or unknown),
   **When** opened, **Then** the recipient sees a neutral message with the practice's
   contact details, and nothing is recorded or disclosed about any person.

---

### User Story 2 - An opted-out person never gets another feedback email (Priority: P1)

As the practice, once someone has opted out, none of the automated milestones may send
them a feedback request, whichever milestone fires and however it is triggered.

**Why this priority**: An opt-out that is not honored is worse than having no opt-out. This
is as foundational as Story 1.

**Independent Test**: Mark a test patient as opted out, then drive each milestone's trigger
condition for that patient (and M0's for a test lead). Confirm no feedback token is issued,
no email is sent, and nothing reaches the person.

**Acceptance Scenarios**:

1. **Given** a patient is opted out, **When** any of Milestones 2, 3, 4, or 5 would
   otherwise fire for them, **Then** no feedback request is issued or emailed.
2. **Given** a lead is opted out, **When** Milestone 0 would otherwise fire, **Then** no
   request is issued or emailed.
3. **Given** a milestone was skipped because of an opt-out, **When** the person is
   resubscribed later, **Then** that skipped milestone is NOT sent retroactively; only
   milestones that fire after the resubscribe are sent.
4. **Given** a milestone was skipped because of an opt-out, **When** an admin reviews
   feedback reporting, **Then** the skip is visible as "opted out" and is not counted as
   an unexplained non-response (consistent with spec 001 FR-015).
5. **Given** the opt-out is recorded at nearly the same moment a milestone fires, **When**
   both happen, **Then** the opt-out wins and nothing is sent.
6. **Given** a request was issued before the person opted out, **When** time passes,
   **Then** the already-sent survey link simply runs out its normal lifetime; opting out
   does not cause another email.

---

### User Story 3 - Staff can record an opt-out received any other way (Priority: P2)

As a staff member, when a patient tells us by phone, in person, or by replying "please
stop" to a feedback email, I can record that in CRM and it takes effect exactly like the
email link.

**Why this priority**: Not everyone will use the link, and a stated request to stop must be
honored whatever the channel. It is P2 only because the CRM field delivered with Story 2
already makes it possible; this story adds the procedure and the reply handling.

**Independent Test**: Record an opt-out by hand on a test patient's CRM record. Drive a
milestone for that patient and confirm it is suppressed exactly as in Story 2.

**Acceptance Scenarios**:

1. **Given** a staff member sets the feedback opt-out on a patient's or lead's record,
   **When** the next milestone would fire, **Then** it is suppressed (Story 2).
2. **Given** a recipient replies asking to stop, **When** staff process the reply,
   **Then** the opt-out is recorded within one business day, with its source noted as a
   reply.
3. **Given** an opt-out of any source, **When** an admin looks at the record, **Then** the
   date and source (email link, staff-recorded, reply) are visible.

---

### User Story 4 - Staff can resubscribe someone who asks (Priority: P3)

As a staff member, when a person who opted out asks to receive feedback requests again, I
can clear the opt-out in CRM.

**Why this priority**: People change their minds, but this is rare and keeping it
staff-only keeps the first version small and prevents accidental re-enrollment.

**Independent Test**: Clear the opt-out on a test patient and confirm that the next
milestone that fires after that moment sends normally, and that earlier skipped milestones
do not.

**Acceptance Scenarios**:

1. **Given** an opted-out patient, **When** staff clear the opt-out, **Then** future
   milestones fire normally.
2. **Given** an opt-out was cleared, **When** an admin looks at the record's history,
   **Then** the original opt-out and the resubscribe are both discoverable.

---

### Edge Cases

- **A recipient is also in marketing email** (the Zoho Campaigns project): the two
  opt-outs are independent. Unsubscribing from marketing does not stop feedback emails,
  and vice versa. Neither system learns from the other who has opted out of what.
- **An email scanner or preview tool opens the link**: opening the link must never opt
  anyone out; only the explicit confirmation does (Story 1, scenario 2).
- **A prospect (Milestone 0) later becomes a patient**: the prospect's opt-out does not carry
  over automatically. The patient is asked again at intake, and that answer takes priority
  (FR-018). Until intake is completed, the patient record has no opt-out recorded.
- **Two records share one email address**: an opt-out recorded through the link applies to
  the record the link belongs to. When staff record a stop request for an address, they
  apply it to every record that uses that address.
- **A patient opts out, then has a clinical safety flag raised from an earlier response**:
  unaffected. Existing responses and flags are never altered by an opt-out, and the flag
  still goes only to admins (spec 001 User Story 4).
- **The same person is reassigned to a different contractor**: unaffected; the opt-out
  belongs to the person, not the contractor.
- **A person opts out after a recurring check-in (Milestone 3) has already been sent**:
  later checkpoints are skipped and recorded as opted out; they are not sent if the person
  resubscribes (Story 2, scenario 3).
- **A contractor looks for who opted out**: contractors have no access to this
  information (see FR-012).

## Requirements *(mandatory)*

### Functional Requirements

**Opting out (Story 1)**

- **FR-001**: Every automated feedback email, for every active milestone (0, 2, 3, 4, 5),
  MUST include a clearly labelled link that lets the recipient stop all further feedback
  emails.
- **FR-002**: Opening the link MUST NOT by itself record an opt-out. The opt-out MUST be
  recorded only after an explicit confirmation action by the recipient, so link scanners
  and previewers cannot opt people out.
- **FR-003**: The recipient MUST be able to opt out in no more than two steps (open the
  link, confirm), without logging in, creating an account, or typing any identifying
  information.
- **FR-004**: The link and every page it leads to MUST be de-identified in the same way as
  the feedback survey: they carry only an opaque token, and MUST NOT display, request, or
  store a name, email address, phone number, or any other direct or indirect identifier in
  the collection tool (constitution Principle I).
- **FR-005**: The opt-out MUST be matched to the correct person and recorded only inside
  CRM, the single rejoin point (constitution Principle II).
- **FR-006**: The unsubscribe link MUST keep working after its survey token has expired
  or been used, and repeating it MUST be harmless (idempotent). Using it MUST NOT alter
  the status of the survey token or any previously collected response.
- **FR-007**: A link that cannot be matched to a person MUST show a neutral message with
  the practice's contact details and MUST NOT record or disclose anything about anyone.
- **FR-008**: After a successful opt-out, the recipient MUST see a confirmation that says
  what stopped (feedback emails), in plain, warm language consistent with the practice's
  tone of voice, and that they can contact the practice if that was a mistake.

**Honoring the opt-out (Story 2)**

- **FR-009**: Every milestone trigger (0, 2, 3, 4, 5) MUST check the opt-out before
  issuing a feedback request, and MUST NOT issue a token or send an email to an opted-out
  person. This check MUST apply to any milestone added later.
- **FR-010**: A milestone skipped because of an opt-out MUST be recorded as skipped for
  that reason, MUST be visible to admins in feedback reporting as "opted out" (not as an
  unexplained non-response), and MUST NOT be sent retroactively if the person is
  resubscribed later. Reporting MUST show counts only, not who.
- **FR-011**: If an opt-out and a milestone trigger occur at nearly the same time, the
  opt-out MUST win.
- **FR-012**: Whether a person has opted out MUST NOT be visible to contractors (consistent
  with constitution Principle IV), and the opt-out status MUST NOT be sent to any
  marketing or public-facing system (constitution Principle VI; Campaigns spec FR-030).

**Other channels and resubscribe (Stories 3 and 4)**

- **FR-013**: CRM MUST let staff record a feedback opt-out directly on a patient or lead
  record, with the same effect as the email link, and MUST capture the date and source
  (email link, staff-recorded, reply) of every opt-out.
- **FR-014**: A stop request received by phone, in person, or as a reply to a feedback
  email MUST be recorded in CRM within one business day of the practice receiving it,
  following a written staff procedure.
- **FR-015**: Only staff MAY clear an opt-out, and only at the person's request. The date
  of the opt-out and of any resubscribe MUST remain discoverable.

**Boundaries**

- **FR-016**: This feature MUST NOT change what a feedback email asks, when milestones
  fire, how scores are stored, or how clinical safety flags work, other than suppressing
  sends to opted-out people.
- **FR-017**: This feature MUST NOT require Zoho Campaigns, and MUST NOT put any
  patient or prospect into Zoho Campaigns.

- **FR-018**: When a prospect becomes a patient, the feedback opt-out status on the patient
  record MUST be set from the patient's own answer at intake, and that answer takes priority
  over any opt-out recorded earlier on the Lead. The Lead's earlier opt-out MUST NOT be
  copied to the patient record automatically. (Dependency: the intake process needs a
  "feedback emails" question; confirm in planning whether one already exists.)

### Key Entities

- **Feedback Opt-Out**: A recorded decision, held on the person's CRM record (patient or
  lead), that no automated feedback email may be sent to them. Has a date, a source (email
  link, staff-recorded, reply), and can be cleared by staff.
- **Unsubscribe Link**: A de-identified link in every feedback email that carries only an
  opaque token and leads to a confirmation step. Its token identifies the person inside
  CRM only.
- **Skipped Milestone**: A record that a milestone would have fired but did not because of
  an opt-out, kept so it appears in reporting and is never sent retroactively.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of feedback emails across milestones 0, 2, 3, 4 and 5 contain the
  unsubscribe link.
- **SC-002**: 100% of confirmed opt-outs result in zero further feedback emails to that
  person from any milestone.
- **SC-003**: 0 opt-outs are ever recorded from a link being merely opened or previewed
  (confirmation required).
- **SC-004**: A recipient can complete an opt-out in under 1 minute and at most two steps,
  with no identifying information typed.
- **SC-005**: 0 feedback-tool pages or stored submissions contain a name, email, or other
  direct identifier as a result of this feature.
- **SC-006**: 100% of phone, in-person, and reply stop requests are recorded in CRM within
  one business day.
- **SC-007**: 100% of skipped milestones appear in admin reporting as opted out; 0 are
  sent retroactively after a resubscribe.
- **SC-008**: 0 contractor accounts can see who has opted out.

## Assumptions

- **Feedback emails are operational messages, not marketing.** They are sent by the
  practice's feedback pipeline, separate from Zoho Campaigns. The practice offers an
  opt-out because it is the right thing to do and builds trust, not because this feature
  decides what the law requires for these messages. Whether any additional wording or
  handling is advisable should be confirmed with compliance counsel alongside the
  Campaigns project's counsel review; it does not block this feature's planning.
- **The opt-out is independent of marketing preferences.** Because the feedback pipeline
  and Campaigns are walled off from each other both ways, unsubscribing from one does
  not unsubscribe from the other. A single combined "preferences" page, or a link from
  Campaigns' preference center to this opt-out, is possible later but is out of scope
  (see below).
- **Stop requests by reply are handled by hand.** Staff read replies to the sending
  mailbox and record the opt-out in CRM; nothing in this version parses replies.
- **Already-sent survey links are left alone.** Opting out stops new feedback emails; it
  does not cancel a survey link already sent, which expires on its normal schedule (7
  days).
- **Patients and prospects already excluded by the legacy-patient cutoff** (feature 004)
  receive no feedback emails regardless; this feature changes nothing for them.
- **Tone and wording** of the link label, page copy, and confirmation message follow
  Liana's tone of voice and are approved by her before go-live, as with the other
  patient-facing emails. This is a delivery step, not a requirement in this spec.
- The product pieces already in use for the pipeline (Zoho Forms for the confirmation
  step, Zoho Flow for the trigger checks, Zoho CRM as system of record) remain covered by
  the BAA position the pipeline already depends on (constitution Principle V). If the
  plan needs any other Zoho product, its BAA coverage must be confirmed first.

### Out of Scope (this version)

- Patient self-serve resubscribe (staff-only; see Story 4).
- Per-milestone opt-out choices.
- Automatic parsing of "stop" replies.
- A native "Feedback emails" category inside the Zoho Campaigns preference center, or
  syncing this opt-out to Campaigns. A link from Campaigns' preference center to this
  opt-out page may be added later as a pointer, without carrying any person's status.
- A combined preferences page covering both feedback and marketing.

### Resolved questions

- **Q1 (lead to patient)**: decided 2026-10-01. A Lead's opt-out does not carry over. The
  person is asked again during intake and the intake answer is authoritative, whether it
  opts them in or out.
- **Q2 (Campaigns "Do Not Contact")**: decided 2026-10-01. No. The marketing flag does not
  suppress feedback emails; the two stay independent.
