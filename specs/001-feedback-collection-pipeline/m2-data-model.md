# Phase 1 Data Model: M2 - Early Alliance Check (Session 3)

**Input**: `m2-plan.md`, `m2-research.md`

**Date**: 2026-09-10 (revised 2026-09-10 — see "Revision" note below)

## Milestone_Instances usage for M2

M2 reuses the existing `Milestone_Instances` module (from M0/M1) the same way
M1 does — patient-based, not lead-based:

| Field | M1 usage | M2 usage |
|---|---|---|
| `Milestone` | `"1 - Baseline Intake"` | `"2 - Early Alliance Check"` (confirmed a clean existing picklist value, per `m0-implementation-notes.md` §5) |
| `Lead_Reference` | Not used | Not used |
| `Patient` (lookup to `Patients1`) | Populated | Populated — same rejoin pattern, Principle II |
| `Status`, `Token`, `Expiry_Date_Time`, `Submitted_Date_Time`, `Response_Data` | Same lifecycle | Same lifecycle, different content (4 domains + 2 flag segments instead of 5 domains) |

**No new CRM fields.** `Milestone_Instances` is at its CRM custom-field cap
(confirmed by Costin, 2026-09-10) — this rules out the two dedicated flag
fields (`Clinical_Safety_Flag`, `Flag_Rule_Triggered`) this document
originally planned. See "Revision" below: the flag data is instead folded
into the existing `Response_Data` blob, and detected via new Analytics
formula columns (which aren't CRM fields and aren't subject to the cap).

## Revision (2026-09-10): flag storage moved from new CRM fields into the blob

**Original plan** (superseded): add `Clinical_Safety_Flag` (boolean) and
`Flag_Rule_Triggered` (text) as new fields on `Milestone_Instances`.

**Why it changed**: Costin confirmed `Milestone_Instances` cannot take any
more custom fields — a real, hard CRM limitation, not a design preference.
This is exactly the scenario `m1-plan.md`'s Complexity Tracking already
anticipated when it chose the blob pattern over dedicated fields "to conserve
`Milestone_Instances`' CRM field budget for M2 and M5" — the budget has now
run out for real.

**Revised approach**: the write-back function appends two more `---`-delimited
segments to the same `Response_Data` blob it already writes (see "Response_Data
blob format" below), and two new Zoho Analytics formula columns parse them
back out for filtering/reporting — same mechanism every other parsed value
(domain scores, M0's Biggest Factor, etc.) already uses. Nothing about the
CRM schema changes; only the blob's content and the Analytics workspace's
formula columns do.

## Trigger flow entity: "M2 - Session 3 Trigger" (new)

Directly mirrors M1's "M1 - Session 1 Trigger" (`m1-implementation-notes.md`
§3), watching the same `Patients1.Session_Count` field for the transition to
`3` instead of `1`:

1. **Idempotency check** (new custom function, e.g. `checkAllianceCheckExists`,
   mirroring M1's `checkBaselineIntakeExists` field-for-field): before creating
   a `Milestone_Instances` record, check whether one already exists for this
   `Patient` + `Milestone = "2 - Early Alliance Check"`. If yes, do nothing —
   satisfies spec.md FR-002.
2. If no existing record: create a new `Milestone_Instances` record —
   `Patient` = the triggering patient, `Milestone = "2 - Early Alliance
   Check"`, `Status = "Issued"`, fresh `Token`, `Expiry_Date_Time` (reuse M0/M1's
   7-day TTL unless Costin specifies otherwise for M2 — same "reuse unless told
   otherwise" default `m1-implementation-notes.md` §4 used).
3. Send the Alliance Check-In survey link via Zoho Mail, through the shared
   "Subflow - Issue Feedback Token" (unchanged — already generically
   parameterized by M1, per `m1-implementation-notes.md` §2). No field/schema
   changes needed on this subflow for M2.

## Response_Data blob format for the Alliance Check-In (revised)

Same `---`-delimited blob pattern as M0/M1, 4 survey domains (Confluence
table order) **plus 2 appended flag segments**:

```text
Connection (0-10): {value}---Understanding (0-10): {value}---Shared direction (0-10): {value}---Fit of approach (0-10): {value}---Clinical Safety Flag: {true|false}---Flag Rule Triggered: {text or blank}
```

Each domain `{value}` is the raw 0-10 slider answer (integer). The 0-40 total
is **not stored in the blob** (same reasoning as M1) — it's a derived
Analytics formula column. `Clinical Safety Flag` is a literal `"true"` or
`"false"` string, and `Flag Rule Triggered` is a semicolon-joined description
of every condition that fired (blank when the flag is `false`) — both
computed by the write-back function (see below), not left for Analytics to
derive.

**Ordering note**: appending the two flag segments after the 4 domains means
"Fit of approach" — previously the last, unbounded field in M2's original
(pre-revision) blob design — is now a **bounded** field (`substring_between`
works cleanly, no guard needed). Only "Flag Rule Triggered", now the true
last field, needs the guarded `SUBSTR`/`INSTR`/`LENGTH` pattern with the
zero-guard `m1-implementation-notes.md` §9.3 already found and fixed for this
exact bug class (an unbounded last-field formula returns garbage, not blank,
on non-matching/other-milestone rows sharing the same cross-milestone table).

**Planned Analytics formula columns**:

- Domain: Connection, Domain: Understanding, Domain: Shared Direction, Domain:
  Fit Of Approach (4 columns, all `substring_between` — all bounded now)
- Alliance Check-In Total (`SUM`/`+` of the 4 domain columns, `to_integer()`
  cast as M1's did)
- Clinical Safety Flag (`substring_between`, text column holding `"true"`/`"false"`)
- Flag Rule Triggered (guarded `SUBSTR`/`INSTR`/`LENGTH` pattern, since it's
  the unbounded true-last field)

## Clinical Safety Flag evaluation (write-back function, Deluge)

Unlike M0/M1's write-back functions, M2's `submitAllianceCheckInResponse`
does real arithmetic in Deluge (not just string concatenation), because the
flag rule needs to be evaluated at write-back time per `m2-research.md`:

- Parse the four domain parameters to numbers (`.toLong()`).
- `total = connection + understanding + sharedDirection + fitOfApproach`.
- Flag conditions (spec.md Clinical Safety Flag Rules, Alliance):
  `total <= 20` OR any one of the four domains `<= 4`.
- Build a list of every condition that fired (a patient can trip more than
  one on the same reading — e.g. a low total AND a low single domain — and
  spec.md doesn't say only the first match counts, so all firing conditions
  are recorded, not just one) and join them into `Flag Rule Triggered`'s text
  (e.g. `"Total alliance score 18/40 (<=20); Connection domain 3/10 (<=4)"`).
- Set `Clinical Safety Flag` to `"true"` if any condition fired, else
  `"false"`, and `Flag Rule Triggered` to the joined text or blank.
- **Revised**: append both as the 5th/6th `---`-delimited segments on the
  `responseText` string the function already builds, then write it to
  `Response_Data` in the **same** single `zoho.crm.updateRecord` call that
  already sets `Status`/`Submitted_Date_Time` — one record update, not two,
  and no new field targets in that update map.

**No notification action of any kind is built** for a fired flag — per
`m2-research.md`'s reconciliation of the Confluence "supervision queue"
language against ratified FR-012. The flag is data inside the blob; making it
visible to admins is a reporting/Analytics concern (see below and
`m2-tasks.md` Phase 6), not a Flow/notification concern.

## Milestone-scoped flag visibility (FR-013, satisfies User Story 4 for M2's own data)

New Analytics report, e.g. **"M2 Flagged for Review"**: Tabular View, base
table "Milestone Instances", filtered to `Milestone = "2 - Early Alliance
Check"` AND the new `Clinical Safety Flag` formula column `= "true"`. Columns:
Submitted Date Time, Token, the four Domain columns, Alliance Check-In Total,
Flag Rule Triggered — no `Patient`/identity column, same de-identification
discipline every other M0/M1/M2 report already follows (per
`m1-implementation-notes.md` §9.1, the `Patient` lookup is deliberately never
synced into Analytics). Bundled into M2's own dashboard alongside the User
Story 6 baseline panels (see `m2-tasks.md` Phase 6) — not the unified FR-007
Admin Dashboard, which stays out of scope for M2 (see `m2-research.md`).

## Out of scope for M2 (carried forward / newly deferred)

- The wellbeing half of the Clinical Safety Flag Rules (Milestones 1, 3, 4) —
  unchanged from M1's scoping; still starts at M3 once a second wellbeing
  reading exists. Note for whoever builds M3/M4: since `Milestone_Instances`
  has no field budget left, wellbeing flag data will need the same
  blob-segment approach this document uses for M2, not dedicated fields.
- The unified, cross-milestone Admin Dashboard (User Story 3 / FR-007–009) —
  same deferral M1 already established; M2 only builds its own milestone-scoped
  reporting (User Story 6 baseline bar + the FR-013 flagged view above).
- Raw Zoho Forms retention purge (`data-retention-purge.md`) — same
  cross-milestone open gap M0/M1 already carry; M2 adds a third flow with the
  identical gap, not newly introduced or newly solved here.
- Any automated notification, queue, or routing to the treating contractor on
  a fired flag — explicitly excluded per FR-012/Principle IV, not merely
  unbuilt for lack of time.
