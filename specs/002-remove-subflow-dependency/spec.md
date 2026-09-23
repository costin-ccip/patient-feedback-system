# Feature Specification: Remove Subflow Dependency from Token Issuance

**Feature Branch**: `002-remove-subflow-dependency`

**Created**: 2026-09-23

**Status**: Draft

**Input**: User description: "Update the Zoho Flow architecture this system was built on
to stop using subflows. The current architecture was built on a trial version of Zoho
Flow, which allowed subflows, so we built one subflow (Subflow - Issue Feedback Token).
On our long-term plan we don't have access to subflows anymore."

<!--
  Context for readers coming from specs/001-feedback-collection-pipeline:

  This is an infrastructure refactor of the pipeline specified in
  specs/001-feedback-collection-pipeline/spec.md, not a new milestone. It changes HOW
  token issuance is wired, not WHAT the pipeline does. Every requirement in 001's
  spec.md (FR-001 through FR-019) must still hold after this change; nothing here
  amends or supersedes them.

  Plan situation as confirmed on 2026-09-23: the Zoho Flow account is on the Standard
  plan (5,000 tasks/month, annual, confirmed via the Zoho Store subscription page).
  Costin states subflows are not available on this plan. (Zoho's public pricing page
  lists subflows under Standard; per Costin's instruction the design treats subflows as
  unavailable regardless, which is also the more durable position: nothing in the
  pipeline should depend on a feature whose plan availability is ambiguous.) Custom
  functions and unlimited live flows ARE available on Standard; that is load-bearing
  for this spec and is recorded as an assumption below.
-->

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Feedback requests keep going out after subflows are gone (Priority: P1)

As the practice, when a patient or prospect crosses an active milestone (0, 2, or 3
today; 4 and 5 once built), a feedback request still has to be issued and emailed
automatically, exactly as it is now, even though the shared "Issue Feedback Token"
subflow that every trigger currently hands off to can no longer be used.

**Why this priority**: Without this, no milestone can issue a single feedback request.
Every trigger flow (M0, M2, M3) currently ends in a call to the subflow, so losing
subflows silently breaks the whole pipeline's foundation (001 User Story 1, FR-001).

**Independent Test**: For each active milestone's trigger, confirm by inspection that
the trigger no longer references the subflow and that the steps which replace it
produce the same issuance outcome: one new Milestone Instance record in Issued
status with a fresh single-use token and expiry, any older still-open instance for the
same person and milestone marked Superseded, and one email to the patient or prospect
containing the milestone's survey link with the token attached. (Live end-to-end
confirmation is run by Costin, not by an automated session; see Assumptions.)

**Acceptance Scenarios**:

1. **Given** a patient/prospect crosses an active milestone condition, **When** the
   trigger fires, **Then** a Milestone Instance record is created in Issued status with
   a new single-use token, the correct milestone value, the correct patient or lead
   reference, the clinician snapshot, and an expiry the configured number of days out.
2. **Given** that same issuance, **When** it completes, **Then** exactly one email is
   sent to the patient/prospect from the practice's existing sending address, with the
   milestone's own subject, intro text, and survey link carrying the token.
3. **Given** an older Milestone Instance for the same person and milestone is still
   Issued, **When** a new one is issued, **Then** the older one is marked Superseded
   first, exactly as today.
4. **Given** the refactor is complete, **When** any active trigger flow is inspected,
   **Then** it contains no subflow call, and no active flow depends on the subflow
   existing or being switched on.

---

### User Story 2 - One definition of "how a token is issued", not three drifting copies (Priority: P2)

As whoever maintains this pipeline, I need the token-issuance logic (supersede old
instances, generate the token and expiry, create the record) to still live in one
place, so a fix or rule change is made once and every milestone picks it up, the same
way the subflow gave us today.

**Why this priority**: The subflow's main value was that M0, M2, and M3 shared one
implementation. Replacing it with hand-copied steps in every flow would recreate the
drift risk the constitution already warns about for derived values (Principle VII),
just for issuance logic instead. It is P2 because the pipeline works without it, but
it degrades with every milestone added (M4 and M5 still need building).

**Independent Test**: Confirm by inspection that every active trigger flow invokes the
same single shared issuance component, and that changing that component (in a
non-production dry check, not a live edit) is reported by the tool as affecting all of
those flows.

**Acceptance Scenarios**:

1. **Given** the refactor is complete, **When** the issuance logic is reviewed, **Then**
   supersede + token generation + record creation exist in exactly one shared
   component, called by every active trigger flow.
2. **Given** a future milestone (M4, M5) is being built, **When** its trigger is wired,
   **Then** the documented pattern is to call that same shared component plus a
   per-milestone email step, not to copy issuance logic into a new flow.

---

### User Story 3 - The as-built record matches reality (Priority: P3)

As the next person (or session) to work on this pipeline, I need every document that
currently describes "Call a subflow → Subflow - Issue Feedback Token" to describe what
is actually built after the change, including the subflow's internals, which have
never been written down in this repo until now.

**Why this priority**: Required by CLAUDE.md's same-session implementation-notes
convention. It is P3 only because it follows the build.

**Independent Test**: Search the repo for "Call a subflow" / "Subflow - Issue Feedback
Token" and confirm every remaining mention is either explicitly historical or points to
this feature's as-built notes.

**Acceptance Scenarios**:

1. **Given** the refactor is complete, **When** M0, M2, and M3's implementation-notes
   files are read, **Then** each describes the new issuance steps for its own trigger
   flow, with exact parameter values.
2. **Given** M4/M5 planning starts later, **When** CLAUDE.md is read, **Then** it
   states the no-subflow issuance pattern as a cross-milestone convention.

### Edge Cases

- **Record created but email step fails** (mail connection error, bad address): today
  the subflow runs the same two steps in sequence with the same failure mode (a record
  left in Issued with no email). The refactor must not make this worse; behavior should
  be identical, and the gap is recorded rather than silently fixed or widened.
- **Issuance component fails before creating the record**: no email may be sent. An
  email must never carry a token that has no matching Issued record (it would be
  rejected on submission and confuse the respondent).
- **Supersede finds more than one open instance**: all of them are superseded, same as
  today.
- **M0 has no idempotency check of its own** (unlike M2/M3). Its duplicate-protection
  today is entirely the supersede step inside the subflow. The refactor must keep
  supersede in M0's path, or M0 starts issuing duplicate open tokens per lead.
- **Retired M1 flows** still contain a subflow call. They are OFF and marked
  `[RETIRED]`; they stay untouched (not rebuilt), but must not be turned back on.
- **The subflow itself**: once no active flow calls it, it stays in the workspace, OFF
  and renamed as retired, for audit, not deleted (deletion is Costin's call).
- **Flows are OFF during the change**: all M0/M2/M3 flows are currently OFF pending
  Costin's coordinated live test, so the refactor carries no risk of a half-migrated
  flow firing on a real patient. It must not switch any flow ON.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: No active trigger flow in the pipeline MAY call a subflow, and no part of
  token issuance MAY depend on the subflow feature being available on the practice's
  automation plan.
- **FR-002**: Every active trigger flow (M0, M2, M3) MUST produce the same issuance
  outcome it produces today: supersede any still-Issued instance for the same
  person + milestone; create one Milestone Instance with the same field values
  (Name, Milestone, Status = Issued, Token, Expiry, Patient or Lead Reference,
  Clinician, Email, and the same CRM workflow-trigger setting); then send one email.
- **FR-003**: Supersede, token/expiry generation, and record creation MUST be
  implemented once, in one shared component invoked by every trigger flow, not copied
  per flow.
- **FR-004**: The shared issuance component MUST return the issued token (and the
  record it created) to the calling flow, so the calling flow can place it in the
  email. The email MUST NOT be sent unless the component reports a successful record
  creation.
- **FR-005**: The patient/prospect email MUST continue to go out through the same
  sending channel and address as today (the practice's existing mail connection,
  `info@capeclarity.com`), with the same body structure, per-milestone subject, intro
  text, and survey link + token. No new sending channel may be introduced (constitution
  Principle V and the Zoho Mail safeguard requirement both gate channel changes).
- **FR-006**: Per-milestone values (milestone picklist value, survey URL, subject,
  intro text, TTL days, clinician, patient vs. lead reference) MUST be preserved
  exactly as currently configured for M0, M2, and M3.
- **FR-007**: The refactor MUST NOT change any write-back flow, form, CRM field,
  picklist, Analytics formula, report, or dashboard. Issuance output is byte-for-byte
  the same record shape the write-back functions and Analytics already read.
- **FR-008**: The refactor MUST NOT switch any flow ON, and MUST NOT perform any live
  or end-to-end test; verification is structural only, with Costin running the
  coordinated live test.
- **FR-009**: The subflow MUST be left in place, OFF, and clearly marked retired once
  no active flow calls it; it MUST NOT be deleted without Costin's explicit approval.
- **FR-010**: The subflow's internals (its two custom functions, its CRM field mapping,
  its email template) MUST be documented verbatim in this feature's as-built notes,
  since no document in the repo records them today.
- **FR-011**: M0, M2, and M3's implementation-notes files, and CLAUDE.md, MUST be
  updated in the same session as the change to describe the new issuance wiring.

### Key Entities

- **Issuance component**: the single shared piece of logic replacing the subflow's
  first three steps. Inputs: milestone, patient reference or lead reference,
  recipient email, clinician, TTL days. Output: the new token, its expiry, and the
  created record's ID, or an error.
- **Per-flow email step**: the one step each trigger flow still owns, carrying that
  milestone's subject, intro text, and survey link.
- **Milestone Instance**: unchanged (see 001 `m0-implementation-notes.md` §5); the
  record shape the issuance component must reproduce exactly.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 0 active flows reference a subflow after the change (M0, M2, M3 trigger
  flows all inspected).
- **SC-002**: 100% of the record fields the subflow sets today are set identically by
  the new path, for each of the 3 trigger flows (checked field by field against the
  documented mapping).
- **SC-003**: Issuance logic exists in exactly 1 place; the number of places to edit
  to change it is 1, not 3 (and stays 1 when M4/M5 are added).
- **SC-004**: When Costin runs the coordinated live test, each of M0, M2, M3 produces
  exactly 1 Issued record and 1 email per trigger, with the survey link accepted by
  the unchanged write-back flow.
- **SC-005**: 0 changes to write-back flows, forms, CRM schema, or Analytics objects.

## Assumptions

- The Zoho Flow Standard plan (confirmed active 2026-09-23) keeps custom functions and
  unlimited live flows. If the plan ever drops custom functions too, this design does
  not hold and a larger redesign is needed (every idempotency check and write-back in
  the pipeline already depends on custom functions, independent of this change).
- Zoho Flow custom functions are workspace-level objects shared across flows (confirmed
  in 001 `m2-implementation-notes.md` §3.2, where it was a hazard). This feature relies
  on that same property deliberately, as the replacement for subflow reuse.
- BAA status (constitution TODO(BAA_SCHEDULE)) is unchanged by this feature: it uses
  only products already in the pipeline (Flow, CRM, Mail), through the same
  connections, so no new Principle V check is triggered.
- The existing issuance behavior is treated as correct and preserved as-is, including
  two quirks worth Costin's attention but deliberately out of scope here (see
  `research.md`): the record's Name includes the recipient's email address, and a
  record-created/email-failed run leaves an Issued record with no email sent.
- Live end-to-end testing is run by Costin, per his standing instruction.
