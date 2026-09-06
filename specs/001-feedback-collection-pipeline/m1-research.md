# Phase 0 Research: M1 - Baseline Intake Survey

**Feature**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1 & 2, M1 scope)

**Date**: 2026-09-06

This resolves the unknowns in `m1-plan.md`'s Technical Context before design.

## Trigger mechanism

**Decision**: Trigger off the existing `Session Count` field on the CRM Patients
module reaching `1`.

**Rationale**: Session Count already exists and is already automated (per Costin,
2026-09-06): a payment added to a patient record fires a CRM workflow rule that
calls a Zoho Flow to increment it. M1 doesn't need to build session counting —
it needs a new Flow that watches for `Session Count == 1` and issues a token,
mirroring M0's issuance pattern but automatic instead of manual. This is exactly
the automatic-trigger design Constitution Principle III (Automated Milestone
Triggers) calls for and that M0 could not use.

**Alternatives considered**: Deriving session count from Zoho Bookings directly.
Rejected — redundant, since CRM already has a working, payment-driven counter;
introducing a second source for the same fact runs against the constitution's
insistence (Principle II, Single Rejoin Point) that CRM stay the one place
trigger data and identity mappings live, not duplicated elsewhere.

**Confirmed** (2026-09-06, via Zoho CRM MCP field-metadata lookup on the
`Patients1` module — schema only, no records read, per Costin's explicit
go-ahead to use the connector for this): the field is `Session_Count`
(integer), on the `Patients1` module. The specific CRM workflow rule that
increments it on payment add was not independently confirmed (no workflow-rule
listing tool is exposed via the CRM connector) — not needed to build M1's
trigger, since M1 only reads `Session_Count`'s value and does not touch the
rule that maintains it.

## Survey content

**Decision**: The M1 survey is the **Cape Clarity Wellbeing Check-In** (custom
instrument), per Confluence page "Milestone 1 — Baseline Intake Survey"
(pageId 564428808, space LP): 5 domains, each a 0-10 slider —

1. Personal wellbeing — "How are you doing personally this week?"
2. Coping — "How are you managing the challenges you're facing (health,
   caregiving, or otherwise) this week?"
3. Relationships & support — "How are things going in your relationships and
   support system this week?"
4. Hope / outlook — "How hopeful do you feel about the future this week?"
5. Sense of control — "How much control do you feel you have over your
   situation this week?"

Summed for a 0-50 total; each domain must also be stored individually (not
collapsed into the total), because trend flagging needs per-domain movement,
not just an aggregate.

**Rationale**: This is Costin's own already-designed instrument, not something
to invent during planning. Confirmed directly via Confluence rather than
guessed from the generic "Outcome score (baseline)" line in the tech-stack doc.

**Important cross-milestone implication**: per the same Confluence page, this
exact instrument is **reused as-is at Milestones 3 and 4** ("all three
milestone pages need to move together" if it changes). M1 is therefore not
just building a one-off survey — it's building the Wellbeing Check-In Form and
its scoring/storage pattern that M3 and M4 will both depend on. Building it
once, generically (parameterized by milestone tag), rather than three times,
is a real efficiency worth deciding now rather than rediscovering at M3.

## Trend-based flagging

**Correction (2026-09-06)**: this section originally treated these thresholds
as informally described by Costin, ahead of any ratified requirement. They
are in fact already ratified in the repository's authoritative `spec.md`, as
the **Clinical Safety Flag Rules** (Requirements section) driving **User
Story 4 - Automated clinical safety flag to admin** (FR-010 through FR-013) —
this plan simply hadn't been checked against that spec before now (see
`CLAUDE.md` "Branch reconciliation" note). The numeric values Costin gave
directly this session match the ratified rules exactly, so no content
changed, only the citation and the removal of an invented severity tier
("case-consultation") the ratified spec does not use — every rule below
produces the same single outcome: a flag for admin review, never an automatic
notification to the treating contractor (FR-012).

**Decision (scope boundary)**: The wellbeing flag rules — total score drops 5+
points from the previous reading; any single domain drops 4+ points from the
previous reading; any single domain scores 3 or below at two consecutive
readings; or the total score hasn't moved more than ±3 points across 3
consecutive readings (stalled) — are **out of M1's build scope**. They apply
to wellbeing readings at Milestones 1, 3, and 4, and most require at least two
readings to evaluate, while M1 is the *first* reading for any given patient.
Flagging becomes real starting at M3 (the second Wellbeing Check-In instance
for a patient). M1's job is limited to capturing and correctly storing the
baseline so M3's flagging logic (FR-010) has something to compare against.

**Rationale**: Matches spec.md's own build ordering — User Story 4 (clinical
safety flagging) is P4, after Stories 1-3, and itself depends on scored
milestone data already existing. Building flagging logic now, before a second
data point can ever exist for any patient, would be premature.

## Delivery mechanism

**Decision**: Reuse M0's pattern — Zoho Form (public, token-only, no PII) +
Zoho Flow custom function write-back to `Milestone_Instances`, delivered via
Zoho Mail. Not a patient portal or intake tablet (both mentioned as options in
the original design doc, but M0 already validated the email+Form+Flow path end
to end, and the constitution requires BAA-covered tools (Principle V,
BAA-Gated Adoption) — reusing the same validated path is lower-risk than
introducing a new delivery surface).

## Storage: dedicated fields vs. delimited blob

**Decided (Costin, 2026-09-06): delimited blob**, consistent with M0's
`Response_Data` pattern, to conserve `Milestone_Instances`' CRM field budget
for M2 and M5. See `m1-plan.md` Complexity Tracking for the accepted tradeoff
(M3's trend-flagging logic will need to compare blobs across records rather
than doing direct field arithmetic).
