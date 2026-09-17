# Phase 0 Research: M3 - Periodic Consolidated Check-In (Every 8th Session)

**Input**: `spec.md` (Milestones table row 3; Clinical Safety Flag Rules;
User Story 6 / FR-018), `constitution.md` (v1.3.0), `m2-plan.md` /
`m2-research.md` / `m2-data-model.md` / `m2-implementation-notes.md`

**Date**: 2026-09-17

**Note**: PROSPECTIVE, per the constitution's Rollout Workflow requirement,
written before any Zoho implementation begins for M3.

## Why M3 needs real research (unlike M2)

M0, M1 (retired), and M2 are all **one-time-per-record** milestones: a
condition becomes true exactly once in a patient's/prospect's lifecycle
(free consult with no conversion; session count reaches 1; session count
reaches 3), and the existing idempotency pattern (`checkXExists`: "does any
`Milestone_Instances` record already exist for this patient at this
Milestone value") is sufficient — once true, it stays satisfied forever, so
"already have one, don't make another" is the entire idempotency question.

M3 is different in kind: spec.md's Milestones table defines its trigger as
**"Every 8th session, recurring for the duration of treatment"** — the same
patient must receive a new M3 feedback request at session 8, again at
session 16, again at 24, and so on, indefinitely. A plain existence check
("does patient X already have a Milestone 3 instance?") would fire once at
session 8 and then permanently block every later occurrence — wrong
behavior, not just an edge case. This is the first milestone in the pipeline
that needs a genuinely different trigger/idempotency shape, not a
copy-and-adjust of the M0/M1/M2 pattern, and it is worth documenting clearly
since M4/M5 (both one-time-per-patient: discharge, discontinuation) don't
need this shape either — M3 may remain the only recurring milestone for a
while, but a future milestone reusing this pattern should start here rather
than rediscovering it.

## Decision 1: How to detect "session count is a multiple of 8"

**Revised 2026-09-17 (Costin)**: the open-ended "any multiple of 8, forever"
design below is more machinery than this needs. Costin's direction: if the
general recurrence logic is complex, just define the specific session
counts that get a survey — e.g. 8, 16, 24, 32, 40 — as a fixed, short list.
**This is now the actual decision** (see the revised Decision 2 below for
the full mechanism); the rest of this section is kept as a record of why
the open-ended version was the first draft and what specifically got
simplified, since a future recurring milestone may still need the
open-ended version if a fixed cap isn't appropriate there.

**Original decision (superseded)**: Keep the same event-driven shape as
M0/M1/M2 (a Zoho CRM "Updated module entry" trigger on `Patients1`,
watching `Session_Count`), but do **not** put a fixed-value filter (like
M2's `Session Count equals 3`) on the trigger step itself. Instead, filter
on `Session Count` `greater than or equal to` `8` (cheap pre-filter, just to
skip early-treatment noise) and push the actual "is this a genuine new
8-session checkpoint" test into a custom function, the same architectural
slot the idempotency-check function already occupies in every prior
milestone's flow — computing "is this a multiple of 8" with Deluge's `%`
operator, since Zoho CRM Flow's trigger-level filter criteria don't have a
modulo/"divisible by" comparator.

**What the revision actually changes**: the trigger step itself is
unaffected — `Session Count >= 8` remains a fine, simple pre-filter,
whether the checkpoint list has 5 entries or 50. What changes is the
*function's* internal logic: instead of open-ended modulo arithmetic that
has to be trusted to keep working correctly forever, the function checks
membership in a short, explicit, easily-inspected list (see Decision 2).
This is simpler to read, simpler to audit ("is 24 in this list?" vs. "is
this expression's modulo logic correct?"), and trivially extended later —
adding a 6th checkpoint (e.g. 48) is a one-line edit to the list, not a
change to the logic itself. The tradeoff, made explicitly and knowingly:
spec.md's Milestones table describes M3 as "recurring for the duration of
treatment," and a fixed list caps that at whatever the list's last entry
is (40, absent a later edit) — **flag for Costin/Liana**: patients whose
treatment continues past session 40 will stop receiving M3 check-ins
entirely, rather than continuing indefinitely, until someone extends the
list. This is the deliberate simplification Costin asked for, not an
oversight, but it's worth spec.md eventually reflecting this operational
reality rather than "for the duration of treatment" if the fixed-list
approach is the lasting design, not just this build's shortcut.

**Alternatives considered**:
- *A fixed-value trigger filter per checkpoint (`equals 8`, `equals 16`,
  ...), cloned indefinitely* — rejected for the trigger step itself (still
  one trigger with a loose `>= 8` filter, not five narrow ones); the
  *function* now does effectively this, but as one short list in one place,
  not five duplicated flow branches.
- *A scheduled (time-based, polling) Zoho Flow that runs daily and queries
  all `Patients1` records for `Session_Count % 8 == 0`.* Still rejected for
  the same reason as before: `Session_Count` already changes via a real,
  observable field-update event M0/M1/M2 already watch, so a second trigger
  *mechanism* would be an inconsistent addition for no benefit.

## Decision 2: Idempotency for a recurring milestone, without new CRM fields

**Constraint carried over from M2** (`m2-data-model.md`): `Milestone_Instances`
is at its CRM custom-field cap (confirmed by Costin, 2026-09-10). This rules
out a dedicated `Session_Count_At_Trigger` field on `Milestone_Instances`
that would otherwise be the obvious way to record exactly which checkpoint
each M3 instance corresponds to.

**Revised 2026-09-17 (Costin)**: rather than deriving the next checkpoint
from open-ended arithmetic (`(existingCount + 1) * 8`, unbounded), use a
short, explicit, hand-maintained list of the actual session counts that get
a survey: **8, 16, 24, 32, 40**. Extending the program later (a 6th
check-in at 48, say) is a one-line edit to this list, not a change to any
formula.

**Decision**: Derive the next expected checkpoint by looking up position
`existingCount` in the fixed list, where `existingCount` is still a
**count of the patient's existing M3 `Milestone_Instances` records** (no
stored checkpoint value needed — same reasoning as the original design):

```text
CHECKPOINTS = [8, 16, 24, 32, 40]

existingCount = COUNT(Milestone_Instances WHERE Patient = this patient
                      AND Milestone = "3 - Periodic Consolidated")
if existingCount >= length(CHECKPOINTS): fire = false   # patient has had every defined check-in
else: nextThreshold = CHECKPOINTS[existingCount]
      fire if: Session_Count >= nextThreshold
```

A new custom function, `checkPeriodicCheckInDue(patientId, sessionCount)`,
returns `true` only when this holds; the flow's existing If-else step
branches on that boolean exactly the way M0/M1/M2's If-else branches on
their `checkXExists` boolean. After a new instance is created,
`existingCount` for that patient increments by construction, which
automatically advances the lookup to the next list entry — no separate
"mark this checkpoint used" step, no new field.

**Why `>=` and not `==`** (unchanged from the original reasoning, still
applies to the fixed list): an exact-match test would silently and
permanently skip a checkpoint if `Session_Count` ever jumps past a listed
value without the flow observing it exactly — e.g. a backdated CRM
correction. With `>=`, the same jump still produces exactly one M3
instance for that checkpoint, and the next check resumes against the
following list entry. **Flag for Costin/Liana**: a large jump still labels
a somewhat-later session as "the 8th/16th/... session" reading rather than
skipping it — reasonable default, not a validated policy, same caveat as
before.

**What changed vs. the superseded open-ended design**: only the source of
`nextThreshold` (list lookup vs. multiplication) and the addition of an
explicit "past the end of the list" stop condition, which the open-ended
version never needed (it had no end). Everything else — counting existing
instances, `>=` matching, no new CRM field — is unchanged. **Operational
consequence, worth confirming with Costin/Liana explicitly**: a patient
in treatment past session 40 simply stops getting M3 check-ins once
`existingCount` reaches 5, until someone edits `CHECKPOINTS` in this
function. This is bounded and simple, per Costin's request, but it is a
real behavior change from spec.md's "recurring for the duration of
treatment" wording — flagged here, not silently absorbed.

**Idempotency under repeated trigger firings at the same count**: if the
same `Session_Count` value produces more than one "Updated module entry"
event (e.g. an unrelated field on the same `Patients1` record is edited
twice while `Session_Count` stays at 8), `checkPeriodicCheckInDue` is
re-evaluated fresh each time. The first firing creates a new instance,
which raises `existingCount` and `nextThreshold` (now 16) — the second
firing then computes `8 >= 16` = `false` and stops, exactly the same
"query CRM fresh, don't cache a decision" pattern `checkAllianceCheckExists`
already uses for M2 (`m2-implementation-notes.md` §3.1).

**Alternatives considered**:
- *Store the triggering `Session_Count` as a new field.* Rejected — no
  field budget (see above); also unnecessary, since the count-of-existing-
  records approach above derives the same information without persisting it.
- *Store it inside the `Response_Data` blob at issuance time, overwritten at
  write-back.* Rejected — would require the idempotency-check function to
  parse `Response_Data` on every *other* patient's records too (expensive
  and fragile), and blurs the blob's job (recording a submitted response)
  with tracking issuance state, which `Status`/`Token`/`Expiry_Date_Time`
  already do cleanly for every other milestone. Also more moving parts than
  a `COUNT(...)` for the same outcome.
- *Exact-match (`==`) per checkpoint.* Rejected — see "Why `>=` and not
  `==`" above.

## Decision 3: Survey structure — reuse M2's Alliance Check-In, add two new sections

spec.md's fifth-pass revision note is explicit about M3's shape: "remaining
sections renumber to Alliance Check-In (1), practice experience (2),
therapist professionalism (3)" — no wellbeing section (retired), no
"likelihood to continue/refer" question (already captured at Milestone 4).

**Decision**: M3 uses a **new, standalone Zoho Form** ("Cape Clarity
Periodic Check-In") rather than re-sending patients to M2's existing Alliance
Check-In form. It repeats the identical 4 alliance sliders M2 already built
(same prompts, same instructions, same 0-10 range, same field order) as its
first section, then adds two new sections:

- **Section 2 — Practice experience** (operational, not clinical; the
  practice's own scheduling/communication/billing experience, distinct from
  the therapeutic relationship the Alliance section already covers):
  - "How satisfied have you been with scheduling and communication with our
    practice (not your therapist directly)?" — slider, 0-10.
  - "How satisfied have you been with the billing/insurance process?" —
    slider, 0-10.
- **Section 3 — Therapist professionalism**:
  - "How would you rate your therapist's professionalism (punctuality,
    preparedness, respectful conduct)?" — slider, 0-10.

  This is the field spec.md's User Story 3 (Acceptance Scenario 2) refers to
  when it lists "per-clinician professionalism averages" among the overall
  dashboard's aggregate metrics — a single rating per M3 instance, averaged
  per clinician at report time (Analytics' job, not this survey's).

**This survey content is drafted, not sourced from Confluence** — per
spec.md's Assumptions ("exact wording/content of each milestone's survey
questions... is instrument-design content tracked in the source Confluence
pages, not a functional requirement of this specification") and consistent
with M1/M2's own precedent of drafting plausible copy and explicitly
flagging it unreviewed (`m1-implementation-notes.md` §4, `m2-implementation-notes.md`
§3.3). **Flag for Costin/Liana**: confirm this against the actual Confluence
milestone-3 detail page before the form goes live, same caveat carried for
every milestone's copy so far.

**Why a new form instead of reusing M2's form as-is**: the patient must see
all three sections as one consolidated request (spec.md: "a single request"
combining the alliance reading with the operational items), and M2's form is
already wired to M2's own write-back flow/function with a fixed 4-field
shape. Building a new form that starts with the same 4 alliance questions
keeps the *content* shared (verbatim reuse of the alliance instrument, as
M2's own notes anticipated — "designed for reuse at M3") without needing
Zoho Forms to conditionally show/hide sections per milestone, which it has
no clean mechanism for on the Free plan (per `m2-implementation-notes.md`
§2's plan-tier findings).

## Decision 4: Response_Data blob field order — alliance domains last, not first

This is the one genuinely new technical wrinkle, and it only shows up
because M3's blob has more than 4 segments. **Decision**: order the blob so
the four alliance domains come **last**, with the two new sections first:

```text
Practice Experience: Scheduling/Communication (0-10): {v}---Practice Experience: Billing (0-10): {v}---Therapist Professionalism (0-10): {v}---Connection (0-10): {v}---Understanding (0-10): {v}---Shared direction (0-10): {v}---Fit of approach (0-10): {v}
```

**Rationale**: M2 already shipped a "Domain: Fit Of Approach" Analytics
formula column (`m2-implementation-notes.md` §9.2) that finds the text
`'Fit of approach (0-10): '` and takes **everything after it to the end of
the string** — a deliberate design for when Fit of Approach is the blob's
true last field (guarding against a naive unbounded pattern producing
garbage on non-matching rows, per `m1-implementation-notes.md` §9.3's
zero-guard fix). That column, and the "Alliance Check-In Total"/"Clinical
Safety Flag"/"Flag Rule Triggered" columns built on top of it, are **shared
across the whole `Milestone Instances` table**, not duplicated per
milestone — M2 never built milestone-specific copies of "Domain: Connection"
etc.; it relies on the label text itself only appearing in rows that
actually contain it. Since M3 reuses the identical alliance question labels
verbatim (Decision 3), the existing "Domain: Connection" / "Domain:
Understanding" / "Domain: Shared Direction" columns (all **bounded**
`substring_between` searches) keep working unchanged for M3 rows with zero
edits. Only "Domain: Fit Of Approach" is at risk, because it assumes nothing
follows it — which stays true for M3 too, **as long as Fit of Approach
remains the last field in the blob**. Keeping alliance domains last (instead
of matching the survey's on-screen section order, which is Alliance first)
avoids having to touch that shared column's search logic at all, and avoids
re-verifying M2's now-shipped formula against a new boundary case.

**This is a deliberate divergence between the respondent-facing survey's
section order (Alliance, then Practice Experience, then Professionalism —
spec.md's stated order) and the internal storage blob's field order**
(Practice Experience and Professionalism first, Alliance domains last). The
respondent never sees the blob; only the write-back function's parameter
mapping needs to know this order, and it's documented here precisely so a
future milestone doesn't "fix" the blob order to match the form and quietly
break the shared Fit Of Approach column. **Confirm this decision holds**
once M3's write-back function is actually built (T-numbered task in
`m3-tasks.md`) — if it turns out cleaner to instead widen the shared column
to a bounded-or-end-of-string formula (checking for a following `'---'` and
falling back to end-of-string when none exists), that is the fallback
option and should be re-evaluated against real Zoho Analytics formula
syntax at build time, not assumed to work from this document alone.

**New M3-only formula columns needed** (not shared with any prior
milestone, since no other milestone's blob contains this text):
"Practice Experience: Scheduling/Communication" and "Practice Experience:
Billing" (bounded `substring_between`, since something always follows them
in this order) and "Therapist Professionalism" (bounded — Connection's
label immediately follows it in the blob). None of the three new columns
needs the unbounded-last-field pattern, since none of them is actually last
in the chosen order — that risk is fully absorbed by the pre-existing
Fit Of Approach column, addressed above.

## Decision 5: Clinical Safety Flag — extend the existing M2 columns, don't fork them

spec.md's Clinical Safety Flag Rules section states the Alliance rule
"applies to alliance readings (Milestones 2, 3, 4)" as **one rule**, not a
per-milestone variant. M2's existing "Clinical Safety Flag" and "Flag Rule
Triggered" formula columns (`m2-implementation-notes.md` §9.2) already
compute this exact rule, gated with `IF("Milestone Instances"."Milestone" =
'2 - Early Alliance Check', ..., '')` so they evaluate to blank for every
other milestone's rows.

**Decision**: widen that existing gate from a single-value equality to
`Milestone = '2 - Early Alliance Check' OR Milestone = '3 - Periodic
Consolidated'` (or the Analytics formula language's equivalent `IN` test),
rather than building a second, parallel "M3 Clinical Safety Flag" column.
This is a **modification to already-shipped M2 Analytics infrastructure**,
not new M3-only work — flagged here so it's not missed, and so
`m2-implementation-notes.md` gets updated in the same session per CLAUDE.md's
convention (a formula-column change is exactly the kind of "how a milestone
actually works" change that convention exists for), even though the change
is being made while building M3.

**Rationale**: one flag definition serving every alliance-bearing milestone
is more maintainable than three near-identical parallel formulas that must
be kept in sync by hand every time the threshold is revised (constitution
Principle VII's own stated rationale — a formula column "recalculates
retroactively across every existing record" the moment it's edited, which
only holds if there's one formula to edit, not three). It also directly
continues M2's own precedent (Domain columns already shared across
milestones by construction) rather than introducing a new, inconsistent
per-milestone-duplication pattern alongside it.

**Risk noted, low**: M2's own real production data is still zero rows
(`m2-implementation-notes.md` §9.3: "0 rows (expected), no real M2 data
exists yet" — M2's flows remain OFF pending live test data). Widening the
gate carries effectively no risk of corrupting real M2 results right now,
but should still be re-verified against M2's own sample/test data (once
that exists) as part of M3's own T014-equivalent verification task, not
assumed safe purely because M2 is quiet.

## Decision 6: `Milestone` picklist value

**Decision**: the new picklist value is `"3 - Periodic Consolidated"` —
same `"# - Name"` shape as `"1 - Baseline Intake"` (retired) and `"2 - Early
Alliance Check"`, taken directly from spec.md's Milestones table Name column
("Periodic consolidated"). To be confirmed via `getFields` on
`Milestone_Instances` (Zoho CRM MCP, not browser — `Milestone_Instances` is
not a Leads/Patients module, so this is within the standing access
constraint) before assuming it doesn't already exist; if missing, it needs
to be added as a picklist option (not a new field, so it is **not** blocked
by the field-count cap that blocks new fields).

## Out of scope for M3 (carried forward / newly deferred)

- The unified, cross-milestone Admin Dashboard (User Story 3 / FR-007–009)
  — same deferral M0/M1/M2 already established; M3 only builds its own
  milestone-scoped reporting (User Story 6 baseline bar + the FR-013
  flagged view, now shared with M2 per Decision 5).
- Raw Zoho Forms retention purge (`data-retention-purge.md`) — same
  cross-milestone open gap; M3 adds a fourth flow with the identical gap,
  not newly introduced or newly solved here.
- Any automated notification, queue, or routing to the treating contractor
  on a fired flag — same FR-012/Principle IV exclusion as M2.
- The wellbeing half of the Clinical Safety Flag Rules — permanently
  removed from spec.md as of the fifth-pass revision (Milestone 1 retired,
  no milestone collects a wellbeing reading anymore). `m2-data-model.md`'s
  "Out of scope for M2" note flagged this as an open question for "M3's own
  research/data-model" to resolve; it is now resolved by the spec itself,
  not by this document — there is no wellbeing trend-flag work for M3 to
  do, and none for M4 either.
- A validated policy for the "large session-count jump" edge case in
  Decision 2 (`>=` firing a checkpoint against a somewhat-later actual
  session) — reasonable default, not yet reviewed by Costin/Liana.
- Whether to extend `CHECKPOINTS` past 40, or whether spec.md's "recurring
  for the duration of treatment" wording should be revised to match the
  fixed-list reality — Costin's call, not resolved here (Decision 2).
