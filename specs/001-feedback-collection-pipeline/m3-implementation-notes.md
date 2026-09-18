# M3 (Periodic Consolidated Check-In, every 8th session) — Implementation Notes

> Scope: this file documents ONLY the M3 milestone (the recurring
> every-8th-session Periodic Consolidated Check-In). It is a companion to
> `specs/001-feedback-collection-pipeline/spec.md` (all six milestones) and to
> `m3-plan.md` / `m3-research.md` / `m3-data-model.md` / `m3-tasks.md`. This
> file tracks what has actually been built in Zoho, as it's built, per the
> `CLAUDE.md` convention of updating implementation notes in the same session
> as the change.
>
> Last updated: 2026-09-17
> Status: **Partially built, in progress.** Done so far: T014 (Clinical
> Safety Flag `Milestone` gate widened to cover M3), T017 (3 new M3-only
> Analytics formula columns), and the title-only half of T016 (report
> renamed "M2 Flagged for Review" → "M2 & M3 Flagged for Review"; its
> Milestone filter is **not yet widened** — see §2.3). **Blocked**: T001 (the
> Periodic Check-In Zoho Form) cannot be built — this account's Zoho Forms
> plan no longer permits creating a new form (see §3, needs a decision from
> Costin). T002-T013 (both flows and their custom functions) have not been
> started; T005/T006 (write-back flow) are also blocked transitively on T001
> since the write-back flow's trigger is that form. T018/T019 (the two
> remaining M3 reports + dashboard) have not been started. T020-T023 (live
> test data) remain explicitly blocked pending a test Patient ID from Costin,
> per standing instruction. See §4 for the full remaining-work list.

## 1. What's built so far

Nothing CRM- or Forms-side has been built for M3 itself yet (no new form, no
new flows, no new custom functions, no new CRM records). What exists so far
is entirely in the shared Zoho Analytics "Milestone Instances" table
(workspace `3251423000000083002`, view `3251423000000083317`) — the same
shared table M0/M1/M2 use, per `m3-data-model.md`'s "no new CRM fields"
design.

### 1.1 "Clinical Safety Flag" formula column widened (T014)

Per `m3-data-model.md`'s Decision 5 (share, don't fork, the M2 alliance
rule), the existing "Clinical Safety Flag" formula column's `Milestone` gate
was widened from M2-only to M2-or-M3. **Deviates from `m3-data-model.md`'s
drafted formula in one respect**: the drafted version used `IF(OR(a, b),
...)` — Zoho Analytics' formula parser rejected this with "Parsing of given
formula failed... Encountered: OR" (confirmed this session; `OR` is an
**infix** operator here, not a function callable as `OR(a,b)` — a new
Analytics-formula-language finding not documented in any prior milestone's
notes). Fixed by rewriting with infix `OR`, which saved successfully. Final,
live formula (verbatim, re-read via right-click → "Edit Formula Column"):

```
IF("Milestone Instances"."Milestone" = '2 - Early Alliance Check' OR "Milestone Instances"."Milestone" = '3 - Periodic Consolidated', IF("Milestone Instances"."Alliance Check-In Total" <= 20 OR to_integer("Milestone Instances"."Domain: Connection") <= 4 OR to_integer("Milestone Instances"."Domain: Understanding") <= 4 OR to_integer("Milestone Instances"."Domain: Shared Direction") <= 4 OR to_integer("Milestone Instances"."Domain: Fit Of Approach") <= 4, 'true', 'false'), '')
```

Logically identical to the drafted version — only the `OR` syntax changed,
not the condition structure. Verified M2's existing test case (row 1) still
evaluates to `false` as before the edit, so this did not regress M2's
already-shipped behavior. `m2-implementation-notes.md` §9.2 has been updated
to match (see that file's changelog note under §9.2).

### 1.2 Three new M3-only Analytics formula columns (T017)

Added to "Milestone Instances", all bounded `substring_between` (each
segment is followed by another `---`-delimited segment in M3's blob order,
per `m3-data-model.md`'s field-order design — none of these needs the
unbounded-last-field guard pattern "Domain: Fit Of Approach" uses):

- **Practice Experience: Scheduling/Communication** —
  `substring_between("Milestone Instances"."Response Data", 'Practice Experience: Scheduling/Communication (0-10): ', '---', 1)`
- **Practice Experience: Billing** —
  `substring_between("Milestone Instances"."Response Data", 'Practice Experience: Billing (0-10): ', '---', 1)`
- **Therapist Professionalism** —
  `substring_between("Milestone Instances"."Response Data", 'Therapist Professionalism (0-10): ', '---', 1)`

All three saved exactly as drafted in `m3-data-model.md` — no parse errors,
no deviations. Confirmed present via column-header enumeration on the table.
Not yet exercised against a real M3 submission (no M3 data exists yet —
T001/T005/T006 aren't built), so these are structurally verified only, per
T017's own "confirm against a real test submission before assuming it"
instruction — that confirmation is still pending on T020-T022.

The four reused alliance-domain columns (Domain: Connection/Understanding/
Shared Direction/Fit Of Approach) and Alliance Check-In Total were **not
modified** — per `m3-data-model.md`'s Decision 4, M3's blob deliberately
puts the 4 alliance segments last, in the same order and with the same
label text M2 uses, so these columns should parse M3 rows with zero edits.
This is unverified against real data for the same reason as above.

### 1.3 "M2 & M3 Flagged for Review" report — renamed, filter NOT yet widened (T016, partial)

The existing "M2 Flagged for Review" report (view ID
`3251423000000141219`) was renamed to "M2 & M3 Flagged for Review" this
session — confirmed via screenshot of the report showing the new title.
Renaming required a workaround: a first attempt (double-click title →
Ctrl+A → type) produced a corrupted concatenation ("M2 FlaggedM2 & M3
Flagged for Reviewfor Review") — the same class of text-field corruption
bug `m2-implementation-notes.md` §3.3 documents for Zoho Flow parameter
fields, now confirmed to also occur in Zoho Analytics' report-title field.
Fixed the same way §3.3 does: set the value via the native input setter
(`Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype,
'value').set.call(el, 'M2 & M3 Flagged for Review')`, dispatch an `input`
event, then press Return) — confirmed correct via a follow-up screenshot.

**The Milestone filter itself has NOT been widened yet.** Per
`m3-data-model.md`, the filter needs to become `Milestone` Wildcard Exactly
Matches `"2 - Early Alliance Check"` **OR** `"3 - Periodic Consolidated"`
(currently still M2-only, per `m2-implementation-notes.md` §9.6's original
build). At the point this file was written, the correct UI path to reach
this report's saved filter-criteria editor had not yet been found:

- The toolbar's **"Filter"** button opens only an ad-hoc per-column
  quick-filter row (Is/Contains dropdowns per visible column) — not the
  report-level saved filter panel `m2-implementation-notes.md` §9.6/§9.8
  describes (a "Filters (N)" count with Wildcard/Individual Values tabs).
- The toolbar's **"More"** dropdown only offers "Show/Hide Column", "Freeze
  Column", and "Wrap header" — also not it.
- Not yet tried at the point this was written: the "Edit Design" control
  (top-right of the report view). This is the most likely remaining
  candidate and is the natural next step once browser access to this
  Analytics workspace is available again (the browser session this work was
  using disconnected mid-task — see §4).

**Practical effect of the gap**: the report currently still shows only M2
rows (title says M2 & M3, filter still says M2-only). No M3 data would
appear in it even once M3 starts producing flagged rows, until this filter
is corrected. Flagging clearly rather than leaving it ambiguous from the
title alone. `m2-implementation-notes.md` §9.6 has been updated to note
both the rename and this open gap (see that file).

## 2. CRM/picklist reference (unchanged from m3-data-model.md, confirmed live)

Confirmed 2026-09-17 via Zoho CRM MCP `getFields` (not browser, per the
standing access constraint — `Milestone_Instances` carries no PII): the
`"3 - Periodic Consolidated"` picklist value already exists on
`Milestone_Instances.Milestone`. No CRM schema change was needed or made.
Full picklist as of that check: `-None-`, `0 - No Conversion`,
`1 - Baseline Intake`, `2 - Early Alliance Check`, `3 - Periodic
Consolidated`, `4 - Discharge`, `5B - Discontinuation, Email Fallback`,
`5A - Discontinuation, Live Capture`. See `m3-data-model.md`'s "Milestone
picklist value" section — reproduced here only because it's load-bearing
for every task that references the exact string `"3 - Periodic
Consolidated"`.

## 3. Blocker: Zoho Forms plan no longer permits creating a new form (T001)

This account's Zoho Forms plan (`lianapreudhommecapec1`) — the same account
`m2-implementation-notes.md` §2 recorded as "Free" — no longer permits
creating a new form by either path tried:

- **"New Form"** from the forms dashboard.
- **"Duplicate"** on an existing form (which M2's §4 already found
  non-functional via browser automation on this account for a different
  reason — worth noting this is now doubly blocked).

Both actions triggered a paid-plan upgrade modal
(`ZFUtil.upgradeAndshowReloadPopup`), linking to a Zoho Store purchase page.
This is a genuine external blocker, not an automation bug — confirmed by
the modal's own upgrade-prompt copy, not just a failed click.

**Per the standing operating constraint (never purchase or upgrade a paid
plan without Costin's explicit permission), no purchase was attempted.**
This blocks:

- **T001** directly — the "Cape Clarity Periodic Check-In" form cannot be
  built on the current plan.
- **T005/T006** transitively — the "M3 - Periodic Check-In Write-back" flow
  needs T001's form to exist as its realtime trigger.
- **T002-T004** are not directly blocked by this (the trigger flow and its
  custom function don't depend on the form existing yet), but T004's "Call a
  subflow" step needs a real `survey_url` value, which doesn't exist without
  T001. T002-T004 could proceed with a placeholder `survey_url` to be
  swapped in once the form exists, but that's a judgment call worth
  confirming with Costin rather than assuming.

**This needs a decision from Costin**, not something to route around
unilaterally: upgrade the Zoho Forms plan, find another way to create the
form on the current plan (e.g. a support ticket to Zoho, or building it
under a different, non-paywalled account/workspace if one exists), or some
other approach. Not yet raised with Costin as of this file's last update —
raising it is the immediate next step.

## 4. Remaining work (not yet built)

- **T001**: Blocked — see §3. Needs Costin's decision before proceeding.
- **T002-T004**: Not started. Could partially proceed (trigger flow +
  `checkPeriodicCheckInDue` function) independent of T001, but the
  "Call a subflow" step's `survey_url` parameter depends on T001 for a real
  value — worth confirming with Costin whether to build with a placeholder
  now or wait.
- **T005/T006**: Blocked transitively on T001 (write-back flow's trigger is
  the form).
- **T007-T013**: Verification/confirmation tasks depending on T002-T006
  being built and (T020-T023) live-tested — not started.
- **T014**: Done — see §1.1.
- **T015**: Not yet reviewed — confirm-absence task (no contractor-facing
  notification anywhere in the M3 build); trivially true right now since
  nothing M3-specific has been built yet beyond the shared Analytics
  columns, but should be re-checked once T002-T006 exist.
- **T016**: Partially done — see §1.3. Filter widening still outstanding.
- **T017**: Done — see §1.2.
- **T018**: Not started — "M3 Submitted Responses" report.
- **T019**: Not started — "M3 Status Breakdown", "M3 Practice Experience &
  Professionalism Distribution", and the "M3 - Periodic Check-In Feedback"
  dashboard.
- **T020-T023**: Explicitly blocked pending a test Patient ID from Costin,
  per standing instruction (no live/end-to-end testing without him).
- **T024** (this file): In progress — being written/updated as work
  proceeds, per CLAUDE.md's same-session convention, rather than held until
  the whole milestone is done.
- **T025**: Done for the changes made so far (§1.1's flag-column widening,
  §1.3's report rename) — `m2-implementation-notes.md` §9.2 and §9.6 updated
  in this session. Will need a further update once T016's filter widening
  is actually completed.
- **T026**: Not yet assessed — worth revisiting once T002-T006 are built,
  per `m3-tasks.md`'s note that the recurring-checkpoint pattern and the
  shared-vs-per-milestone Analytics decision framework are the most likely
  candidates for promotion to `CLAUDE.md`.
- **T027**: This file and the `m2-implementation-notes.md` updates are being
  committed and pushed in this session (via the browser-upload workaround,
  same as the M3 planning docs) — see the top-level commit for this change.

## 5. Session note: browser access interrupted mid-task

The browser session this work was using (Zoho Analytics, via the Chrome
extension) disconnected while investigating T016's filter-editing UI (see
§1.3's "not yet tried" note). Per the "avoid rabbit holes" browser-tool
guidance, this was not repeatedly retried — instead, work pivoted to
non-browser tasks: writing this file, updating `m2-implementation-notes.md`,
and pushing both to GitHub. Continuing T016's filter fix, and starting
T018/T019, is the natural next step once browser access to the Zoho
Analytics workspace is available again.
