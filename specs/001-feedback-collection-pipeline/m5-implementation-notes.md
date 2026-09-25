# M5 (Discontinuation) — Implementation Notes

> Scope: this file documents ONLY the M5 milestone (Discontinuation —
> one-time-per-patient, single-condition trigger `Patient_Status =
> "Discontinued (Patient Choice)"`, per Costin's explicit decision to drop
> the "No Show" leg — see `m5-research.md`'s "Revision (same day)" note). It
> is a companion to `specs/001-feedback-collection-pipeline/spec.md` (all six
> milestones) and to `m5-plan.md` / `m5-research.md` / `m5-data-model.md` /
> `m5-tasks.md`. This file tracks what has actually been built in Zoho, as
> it's built, per the `CLAUDE.md` convention of updating implementation notes
> in the same session as the change.
>
> Last updated: 2026-09-24
> Status: **T001 (the "Cape Clarity Discontinuation Feedback" Zoho Form, all
> 3 fields + thank-you page) is built and verified by screenshot after each
> field save — see §1. One deviation from the sourced closing line, forced
> by a builder character limit, is recorded in §1.1 — flagged for Costin's
> awareness at Checkpoint 2, not silently absorbed.** Nothing past T001 has
> been built yet (no flows, no Analytics). This is Checkpoint 2 per Costin's
> standing checkpoint list — paused here for his review before Phase 2 (the
> Zoho Flows).

## 1. "Cape Clarity Discontinuation Feedback" Zoho Form (T001)

Built fresh in Zoho Forms
(`https://forms.zoho.com/lianapreudhommecapec1/form/CapeClarityDiscontinuationFeedback/builder`),
Standard form type, matching `m5-data-model.md`'s field inventory. On-screen
order, top to bottom, confirmed via screenshot after each field was added and
saved (each field was dropped directly below the previous one and landed in
the intended position on the first attempt — no reordering needed):

| Order | Field type | Label (on-screen question text) | Choices | Mandatory | Visibility |
|---|---|---|---|---|---|
| 1 | Dropdown | What led to stepping away from sessions right now? | Felt better / reached their goals; Scheduling or timing conflict; Cost or insurance; Didn't feel like the right fit with their therapist; Life circumstances changed; Moved or relocated; Choosing to pause for now; Something else (8 total, source order preserved) | Yes | Show |
| 2 | Radio | Would it be okay to reach back out if things change? | Yes / No / Maybe | Yes | Show |
| 3 | Single Line | Token | — | No | **Hide** |

No name, email, or phone field (Principle I). No open-text field (per the
source Confluence doc's own explicit "no open text" design note, recorded in
`m5-research.md` Decision 4).

Builder mechanics notes (Zoho Forms, consistent with prior milestones):
choice-list editing happens in a separate "Choice Field Properties"
sub-panel opened via an "Edit" button under the Properties panel's Choices
section, not inline on the main panel; the Properties panel occasionally
rendered at less than full width immediately after opening (choice/field-
label text visually truncated) — this resolved itself after the next
click/interaction and did not affect what was actually saved, confirmed each
time via a follow-up screenshot before proceeding. One transient "Browser
extension is not connected" disconnection occurred mid-build; a retry of the
tab-context call reconnected it, and a screenshot was taken to confirm actual
page state before re-issuing the interrupted action (no double-submission or
lost input resulted).

### 1.1 Deviation: thank-you page text shortened for a 100-character cap

`m5-data-model.md` and the source Confluence doc specify the closing line
verbatim as:

> "Thank you for letting us know. We wish you well, and we're here whenever
> you're ready, if you ever are." (103 characters)

Zoho Forms' Thank You Page field enforces a hard 100-character maximum in
both Plain Text and Rich Text mode (confirmed by attempting Rich Text — same
cap applies there too). The sourced line is 3 characters over. Rather than
guess at a rewrite, the single lowest-impact cut was made: dropping "ever"
from the final clause. As built:

> "Thank you for letting us know. We wish you well, and we're here whenever
> you're ready, if you are." (98 characters)

This is a builder-forced deviation from the verbatim-sourced copy, not a
content judgment call — flagged here for Costin's awareness at Checkpoint 2.
If a different shortening is preferred, this is a one-field edit in Zoho
Forms Settings → General → "Thank You Page & Redirection".

The default "Include a link to allow respondents to add another response"
checkbox was left checked (Zoho Forms' default); not evaluated against any
milestone-specific requirement — flagging in case Costin wants it unchecked
for a one-time discontinuation survey.

## 2. Remaining work (not yet built)

- Phase 2 (T002-T009): "M5 - Discontinuation Trigger" flow and "M5 -
  Discontinuation Write-back" flow — **paused, pending Checkpoint 2
  approval.**
- Phase 7 (T019-T021): Analytics formula columns, reports, dashboard —
  pending Checkpoint 3 approval (flows built first).
- Phase 8 (T022-T026): live test data — pending Checkpoint 4 approval and
  direct involvement from Costin (a test Patient ID, per his no-live-test-
  without-me instruction).
- The open item recorded in `m5-research.md` ("Open item for spec.md") is
  still open: this build covers only the cancellation leg of spec.md's
  Milestones-table row 5, not the no-show leg. Unaffected by today's work.
