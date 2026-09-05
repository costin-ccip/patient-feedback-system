<!--
Sync Impact Report
- Version change: 1.0.0 → 1.1.0
- Rationale for the original 1.0.0 (not a continuation of a previously-reported
  "v1.0.1"): this repository was found completely empty at the start of the session that
  first authored this file (zero commits, zero branches). An earlier working session had
  reportedly ratified a constitution and drafted a spec, but neither ever made it into
  this repository. Rather than fabricate an amendment history that cannot be verified,
  that document was treated as a fresh ratification of five principles as described by
  the project's operations lead, based on the source design doc (Confluence: "Feedback
  Collection — Proposed Tech Stack").
- 1.1.0 amendment: Principle IV (Contractor Blindness to Own Raw Feedback) narrowed for
  the first version — contractors now have NO dashboard/feedback-data access at all
  (previously they were described as having an aggregated, own-caseload dashboard). A
  restricted contractor view is left open as a possible later-version addition, still
  gated on the same blindness guarantee. This is a MINOR bump: it materially changes what
  the principle currently permits without removing the underlying protection it exists
  to guarantee.
- Modified principles: IV. Contractor Blindness to Own Raw Feedback (dashboard access
  narrowed to admins-only for v1)
- Added sections: none this amendment
- Removed sections: none this amendment
- Follow-up TODOs:
  - TODO(BAA_SCHEDULE): Confirm with the Zoho account manager which specific products
    (CRM, Flow, Mail, Forms, Analytics) are named in Cape Clarity's actual BAA schedule.
    Principle V is written to block on this per-product, not to assume it.
  - TODO(TRIGGER_SOURCE): Decide whether CRM session-count/status fields that drive
    milestone triggers are populated manually or derived from Zoho Bookings. Principle
    III requires automation of the *sending* of triggers once a milestone condition is
    true in CRM, but does not yet resolve how that condition gets set.
-->

# Cape Clarity Patient Feedback System Constitution

## Core Principles

### I. De-Identification by Design
The survey collection tool MUST NOT ever receive, store, or display patient identity.
The only patient-linking data any collection tool (currently Zoho Forms) ever sees is a
single-use, opaque token. No name, email address, phone number, or other direct or
indirect identifier may appear as a visible or hidden field in the collection tool
itself. Rationale: putting identity in the survey tool would require that tool to meet
the full weight of HIPAA technical safeguards and BAA coverage for identified PHI;
keeping identity out of it by construction is cheaper, safer, and does not depend on
that tool's own security features being correctly configured.

### II. Single Rejoin Point
Zoho CRM is the only system in the pipeline permitted to hold both a feedback token and
the patient identity it maps to, and the only place where token and identity are matched
back together. No other system, integration, export, or log may combine token and
identity. Contractor attribution to a feedback response MUST also be joined only inside
CRM, after the fact. Rationale: a single, access-controlled rejoin point is auditable and
limits the blast radius of any one system being compromised or misconfigured.

### III. Automated Milestone Triggers
Feedback requests MUST be initiated automatically by system-detected conditions (e.g.
session count reaching a threshold, a change in patient status, a cancellation/no-show
pattern) — never by a contractor or clinician manually remembering to request feedback
from their own patient. Rationale: manual triggering creates a conflict of interest (a
clinician deciding whether their own performance gets measured) and is not reliable
operationally.

### IV. Contractor Blindness to Own Raw Feedback
A contractor (the clinician who provided care) MUST NOT be able to see raw,
patient-identified feedback about their own patients. In the first version of this
system, contractors have NO access to feedback data or dashboards of any kind —
dashboard access is limited entirely to practice admins. A later version MAY introduce a
restricted, aggregated view for contractors of their own caseload, but only if it
continues to guarantee no contractor ever sees a direct, identified link between one of
their own patients and that patient's verbatim responses about them. Rationale:
unblinded raw feedback creates pressure on patients (real or perceived) and undermines
candid responses; starting with no contractor access at all is the simplest way to
guarantee that risk doesn't exist in the first version.

### V. BAA-Gated Adoption
No Zoho product may be used to process, store, or transmit patient data — including
de-identified tokens tied to a patient record — until that specific product's coverage
under Cape Clarity's Business Associate Agreement (BAA) with Zoho has been explicitly
confirmed. Coverage is NEVER assumed for a product just because other Zoho products are
already confirmed covered. Each product (CRM, Flow, Mail, Forms, Analytics, and any
future addition) MUST be checked individually. Rationale: the BAA only covers what is
actually named in its schedule; assuming broader coverage is a compliance risk, not a
convenience.

## Compliance & Data Handling Requirements

- Raw survey entries (Zoho Forms submissions) MUST be purged on a short, defined
  retention window once their scores have been copied into CRM; the exact window is
  still open (see Governance TODOs) but MUST be short, not indefinite.
- A used token MUST be invalidated immediately after its response is matched back to
  CRM; it MUST NOT be reusable or valid for a second submission.
- This constitution's principles reduce HIPAA exposure by design but do NOT by
  themselves constitute a compliance guarantee — the token-to-identity mapping still
  lives in CRM under Cape Clarity's control and remains in HIPAA scope. Any reliance on
  this design as a compliance position (rather than a security practice) SHOULD be
  reviewed with compliance counsel first.
- Zoho Mail and any other channel used to deliver patient-facing messages MUST meet the
  Security Rule's technical safeguards in active use (TLS enforcement, MFA, blocked
  auto-forwarding, audit log retention, access restrictions) before it is relied on for
  this pipeline, independent of its BAA status.

## Development Workflow

- Any change to the trigger logic, token generation/validation, the CRM rejoin step, or
  any dashboard's access rules MUST be checked against Principles I–IV before being
  merged; a change that would let an un-rejoined system see both token and identity, or
  that would let a contractor see feedback data or a dashboard at all, MUST be rejected
  regardless of any other benefit it offers.
- Before wiring up a new Zoho product (or a new use of an existing one) to touch patient
  data, Principle V's BAA confirmation MUST be completed and recorded first.

## Governance

This constitution supersedes any other undocumented practice or verbal agreement about
how this pipeline is built. Amendments require: (1) a documented proposal describing the
change and its rationale, (2) an explicit version bump following semantic versioning
(MAJOR for backward-incompatible principle removal/redefinition, MINOR for a new
principle or materially expanded guidance, PATCH for wording/clarification only), and
(3) an update to this file's Sync Impact Report recording what changed. Any plan, spec,
or task generated under Spec Kit for this project MUST be checked for compliance with
these principles before implementation begins.

**Version**: 1.1.0 | **Ratified**: 2026-09-04 | **Last Amended**: 2026-09-05
