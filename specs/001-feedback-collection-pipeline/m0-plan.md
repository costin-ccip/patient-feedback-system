# Implementation Plan: M0 - Free Consult / No-Conversion Feedback (Milestone 0)

**Branch**: `001-feedback-collection-pipeline` (backfilled retroactively for M0 scope)

**Date**: 2026-09-06 (backfilled; original build 2026-09-02 to 2026-09-06)

**Spec**: `specs/001-feedback-collection-pipeline/spec.md`

**Note**: RETROACTIVE. M0 was built ad hoc, directly in Zoho Flow/CRM/Analytics,
skipping `/speckit-plan` and `/speckit-tasks` — a deviation from the constitution's
Rollout Workflow requirement (see `m0-implementation-notes.md` §10). This plan and
its companion `m0-tasks.md` were written after the fact, from the completed
implementation, so M0 has the same documentation trail every milestone from M1
onward will get prospectively.

## Summary

M0 realizes User Story 1 (automated milestone trigger... the free-consult/
no-conversion case specifically) and User Story 2 (de-identified response
collection and CRM rejoin) from `spec.md`, scoped to a single milestone. The
clinician manually triggers token issuance when a free consult doesn't convert
(a documented manual fallback per Constitution Principle III — a non-conversion
isn't a session-count threshold like the other five milestones) via the Zoho Flow
subflow "Issue Feedback Token". The lead receives a link to a public Zoho Form
with no identifying fields. Submission triggers "M0 - Feedback Survey
Write-back", whose custom Deluge function `submitFeedbackResponse` matches the
token to the CRM `Milestone_Instances` record, validates status/expiry, and
writes the parsed response. Zoho Analytics syncs from CRM and exposes the parsed
fields via five formula columns feeding a dedicated M0 dashboard.

## Technical Context

**Language/Runtime**: Zoho Deluge (custom functions), Zoho Analytics formula
language — no general-purpose application code.

**Primary Dependencies**: Zoho Forms (public survey), Zoho Flow (trigger +
custom function + CRM connection), Zoho CRM (`Milestone_Instances` module),
Zoho Analytics (synced workspace + formula columns + dashboard).

**Storage**: Zoho CRM `Milestone_Instances` is the system of record (Constitution
Principle II); Zoho Analytics holds a synced, read-oriented copy for reporting
only.

**Testing**: Manual and MCP-tool-driven (Zoho CRM MCP: `getRecords`/
`createRecords`/`deleteRecords`) sample-data testing; no automated test suite
exists for this low-code stack.

**Target Platform**: Zoho One SaaS stack (Forms, Flow, CRM, Analytics) under
Cape Clarity's Zoho org.

**Project Type**: Low-code/no-code automation pipeline (not a compiled
application) — see Project Structure below for how the standard template maps
onto that.

**Performance Goals**: N/A at current volume (a ~200-record linear scan in the
token lookup is acceptable per `m0-implementation-notes.md` §3.2; revisit if
`Milestone_Instances` active-record count grows materially).

**Constraints**: Constitution Principle I (De-Identification by Design) and
Principle II (Single Rejoin Point) are hard constraints on every design
decision below.

**Scale/Scope**: Single milestone (of six), single test Lead used for
validation, low volume (one practice's free-consult volume).

## Constitution Check

*Re-checked retroactively against the built system, 2026-09-06 — and again
against the repository's actual authoritative constitution (v1.2.1), after
discovering this work had been built on a divergent, non-authoritative local
history. See `CLAUDE.md` "Branch reconciliation" note for how that happened.
Principle names/numbers below are corrected to match v1.2.1; nothing about
M0's built system changed, only which document it's being checked against.*

- **Principle I (De-Identification by Design)**: PASS. The Zoho Form and
  `Response_Data` never carry a name/email/phone; matching is entirely via
  `Token`. `Lead_Reference` stores a CRM record ID, not PII, and the `Patient`
  lookup field is never populated by this flow.
- **Principle II (Single Rejoin Point)**: PASS. `Milestone_Instances` in CRM is
  the only place the token-to-record mapping and Status/Expiry live; Analytics
  only reads a synced copy, and no other system combines token and identity.
- **Principle III (Automated Milestone Triggers)**: PARTIAL — documented
  fallback. Token issuance for M0 is manually triggered by the clinician,
  since a free-consult non-conversion isn't a session-count threshold crossing
  like the other five milestones. Note: v1.2.1's Principle III does not
  explicitly spell out a manual-fallback allowance the way earlier internal
  notes assumed — re-read literally, "never by a contractor or clinician
  manually remembering" is closer to an absolute rule. Flagging this as a real
  open question rather than assuming M0's manual trigger is compliant by
  default: worth a direct decision from Costin on whether M0's fallback is an
  accepted, documented exception or something that needs to become automatic
  (e.g. a CRM field marking "consult completed, no appointment within 14
  days," matching the ratified Milestone 0 trigger definition in `spec.md`).
- **Principle IV (Contractor Blindness to Own Raw Feedback)**: PASS. M0 has no
  contractor-facing dashboard or feedback-data access of any kind, consistent
  with v1.2.1's stricter framing (zero contractor access in v1, not a scoped
  "separation of duties" view as earlier internal notes assumed). No conflict
  found for M0 specifically, since no dashboard has been built yet — this
  becomes load-bearing once a dashboard milestone is planned, per FR-009.
- **Principle V (BAA-Gated Adoption)**: OPEN. `TODO(BAA_SCHEDULE)` in the
  constitution is still unresolved. M0 has only run against test/sample data
  so far, so this hasn't blocked development, but it MUST be resolved before
  M0 (or any milestone) handles real leads.
- **Principle VI (Internal Use Only — Never a Testimonial or Marketing
  Pipeline)**: PASS by construction. Nothing in M0 feeds a public-facing site,
  review platform, or marketing tool; the pipeline terminates in CRM/Analytics
  only.
- **Compliance & Data Handling Requirements (purge of raw Zoho Forms
  entries)**: **GAP — NOT IMPLEMENTED.** `submitFeedbackResponse` copies the
  response into CRM and marks the Milestone Instance `Submitted`, but nothing
  in the current flow deletes or schedules deletion of the raw Zoho Forms
  submission entry. The constitution (v1.2.1) requires this purge on "a
  short, defined retention window" once copied into CRM — it does not itself
  fix that window at 24 hours; the exact window is explicitly still open per
  its own Governance TODOs. Prior work on this milestone (and Costin
  separately, 2026-09-06) has been targeting 24 hours as the working number,
  which is a reasonable candidate but not yet a ratified constitutional value
  — worth Costin confirming explicitly if/when the constitution is next
  amended, rather than treating 24h as settled. Either way this is a real
  compliance gap, not just a documentation gap, and should be treated as a
  go-live blocker alongside the BAA confirmation. Investigated a generic fix
  and hit real technical blockers (no documented Zoho Forms delete-entry API;
  the native Auto-Trash feature needs a plan upgrade and still leaves a 5-day
  recovery window). Logged as a known limitation to revisit — see
  `data-retention-purge.md`.

## Project Structure

### Documentation (this feature)

```text
specs/001-feedback-collection-pipeline/
├── spec.md                        # all six milestones, requirements level
├── m0-plan.md                     # this file (backfilled)
├── m0-tasks.md                    # backfilled task list
├── m0-implementation-notes.md     # as-built reference (components, code, formulas, gotchas)
└── checklists/requirements.md
```

### "Source" (a Zoho org, not a code repository)

```text
Customer Feedback System (Zoho Flow folder)
├── Subflow - Issue Feedback Token         # manual trigger, invoked by clinician action
├── M0 - Feedback Survey Write-back        # realtime Form-submission trigger + submitFeedbackResponse

Zoho CRM
└── Milestone_Instances module             # Token, Status, Expiry_Date_Time, Response_Data, Lead_Reference, etc.

Zoho Analytics (workspace 3251423000000083002)
├── Milestone Instances (synced table) + 5 formula columns
├── M0 Status Breakdown / Response Rate / Volume by Week / Submitted Responses (reports)
├── M0 Biggest Factor / M0 Feeling Heard Distribution / M0 % Reachable (reports)
└── M0 - Free Consult Non-Conversion Feedback (dashboard, 7 panels)
```

**Structure Decision**: No application-code repository structure applies here —
this is a configuration-only build across four connected Zoho products.
Documentation lives in this repo (`spec.md` at the feature level, `m0-*.md` at
the milestone level, per the constitution's Rollout Workflow requirement); the
actual "source" lives inside the Zoho org itself, which is why
`m0-implementation-notes.md` exists as the verbatim record of that
configuration (custom function code, formulas, field schemas).

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| `Response_Data` stored as one delimited text blob instead of discrete CRM fields | `Milestone_Instances` is shared across all six milestones and Zoho CRM custom fields are capped; six milestones x several open-ended questions each would exhaust the field cap | Dedicated fields per milestone question was the natural first approach but doesn't scale across 6 milestones sharing one module |
| Manual (not CRM-field-triggered) token issuance for M0 | A free-consult non-conversion isn't a "session count crosses N" event like the other five milestones — there's no natural automatic trigger condition to detect | Automatic detection would need a proxy signal (e.g., a specific Lead status change) that doesn't exist in CRM yet; deferred rather than force-fit |

## Addendum (2026-09-08) — superseded: token issuance is now automatic

The Constitution Check's Principle III entry above, and this table's "Manual...
token issuance" row, describe the state as understood on 2026-09-06. A later
session found this is no longer accurate: a flow ("M0 - Lost Lead Feedback
Token") fires automatically off `Leads.Lead_Status` transitioning to `Lost Lead`
— the proxy CRM signal this table said didn't exist yet has since been built.
See `m0-implementation-notes.md` §11 for the full correction, including what's
still an open question (whether a team member changing a Lead's status counts as
sufficiently "automatic" under Principle III, versus M1's fully system-detected
trigger). This section is left in place as the retroactive record of the
2026-09-06 build; treat §11 as the current source of truth on this point.
