# Phase 1 Data Model: M2 - Early Alliance Check (Session 3)

**Input**: `m2-plan.md`, `m2-research.md`

**Date**: 2026-09-10 (revised 2026-09-10 — see "Revision" notes below; two
revisions landed the same day)

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

**Revised approach (superseded by Revision 2 below)**: the write-back function
appends two more `---`-delimited segments to the same `Response_Data` blob it
already writes, and two new Zoho Analytics formula columns parse them back out
for filtering/reporting — same mechanism every other parsed value (domain
scores, M0's Biggest Factor, etc.) already uses. Nothing about the CRM schema
changes; only the blob's content and the Analytics workspace's formula
columns do.

## Revision 2 (2026-09-10): flag computed entirely in Analytics — not stored anywhere

**Liana's challenge**: why does M2's write-back function need to compute the
flag in Deluge at all, when the four domain values it's already writing to
the blob are all Analytics needs to derive the flag itself — the same way
Analytics already derives the 0-40 total without that total ever being
stored? Appending Deluge-computed flag segments to the blob made M2's
write-back function meaningfully more complex than M0/M1's (real arithmetic
and conditional branching, not just string concatenation) to compute a value
that duplicates information already sitting in the blob.

**Why the challenge is correct**: the flag conditions (`total ≤20` OR `any
domain ≤4`) use nothing but the four domain scores and their sum — both
already fully derivable in Analytics from the blob's existing 4 segments.
There is no new information in the flag that isn't already present. Computing
it in Deluge and storing the result duplicated logic Analytics was already
going to do for the total, for no benefit: FR-013 only requires the flag be
visible in an Analytics-based dashboard view, and FR-012 bans any automated,
time-critical action keyed off the flag, so nothing in this system needs the
flag to exist at the instant of submission rather than at query time. Storing
a Deluge-computed result also freezes that result against future rule
changes — a threshold revision would need reprocessing old records, whereas
an Analytics formula column recalculates every existing response
automatically the moment the formula is edited.

**Final approach**: `submitAllianceCheckInResponse` reverts to pure string
concatenation of the 4 domain answers, the same shape as M0/M1's write-back
functions — no numeric parsing, no conditional logic, no blob segments beyond
the 4 domains. `Clinical Safety Flag` and `Flag Rule Triggered` become two new
Zoho Analytics formula columns computed directly from the Domain and Total
formula columns (see "Response_Data blob format" and "Planned Analytics
formula columns" below) — never stored in `Milestone_Instances` at all, not
as CRM fields (ruled out by the field cap) and not as blob segments either
(this revision's correction). `Response_Data`'s blob format for M2 is now
identical in shape to M1's.

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

## Response_Data blob format for the Alliance Check-In (Revision 2: same shape as M1)

Same `---`-delimited blob pattern as M0/M1, just the 4 survey domains
(Confluence table order) — **no appended flag segments**, per Revision 2
above:

```text
Connection (0-10): {value}---Understanding (0-10): {value}---Shared direction (0-10): {value}---Fit of approach (0-10): {value}
```

Each domain `{value}` is the raw 0-10 slider answer (integer). The 0-40 total
is **not stored in the blob** (same reasoning as M1) — it's a derived
Analytics formula column, and so are both flag values now (see below).

**Ordering note**: with no segments appended after the domains, "Fit of
approach" is the true last field, same as it would be in an unmodified M1-style
blob — it needs the guarded `SUBSTR`/`INSTR`/`LENGTH` pattern with the
zero-guard `m1-implementation-notes.md` §9.3 already found and fixed for this
exact bug class (an unbounded last-field formula returns garbage, not blank,
on non-matching/other-milestone rows sharing the same cross-milestone table).
The other 3 domains are bounded (`substring_between` works cleanly).

**Planned Analytics formula columns**:

- Domain: Connection, Domain: Understanding, Domain: Shared Direction (3
  columns, `substring_between`, bounded)
- Domain: Fit Of Approach (guarded `SUBSTR`/`INSTR`/`LENGTH` pattern — it's
  the unbounded last field)
- Alliance Check-In Total (`SUM`/`+` of the 4 domain columns, `to_integer()`
  cast as M1's did)
- Clinical Safety Flag (**new, Revision 2**: `IF(OR(Total<=20,
  [Domain: Connection]<=4, [Domain: Understanding]<=4,
  [Domain: Shared Direction]<=4, [Domain: Fit Of Approach]<=4), "true",
  "false")` — references the Total and 4 Domain formula columns directly, not
  the raw blob; no substring parsing of its own)
- Flag Rule Triggered (**new, Revision 2**: nested `IF`/`CONCATENATE` building
  a description from the same Total/Domain columns — see "Clinical Safety
  Flag evaluation" below for the exact logic)

## Clinical Safety Flag evaluation (Revision 2: Analytics formula columns, not Deluge)

`submitAllianceCheckInResponse` does **no** flag-related arithmetic — it's
pure string concatenation of the 4 domain answers, the same shape as
M0/M1's write-back functions. All flag logic lives in Analytics, evaluated
against the already-parsed Domain and Total formula columns (spec.md
Clinical Safety Flag Rules, Alliance):

- **Clinical Safety Flag** column: `"true"` if `Total <= 20` OR any one of
  the four Domain columns `<= 4`, else `"false"` — a single `IF(OR(...), ...)`
  formula, no parsing of its own since it reads the already-numeric Total/
  Domain columns.
- **Flag Rule Triggered** column: a description of every condition that
  fired (a patient can trip more than one on the same reading — e.g. a low
  total AND a low single domain — and spec.md doesn't say only the first
  match counts, so all firing conditions are recorded, not just one), built
  with nested `IF`/`CONCATENATE` referencing the same Total/Domain columns —
  e.g. `"Total alliance score 18/40 (<=20); Connection domain 3/10 (<=4)"` —
  blank when the flag is `"false"`.

**No notification action of any kind is built** for a fired flag — per
`m2-research.md`'s reconciliation of the Confluence "supervision queue"
language against ratified FR-012. The flag is a pair of Analytics formula
columns, computed at query/report time from data already visible to
Analytics; making it visible to admins is a reporting/Analytics concern (see
below and `m2-tasks.md` Phase 6), not a Flow/notification/Deluge concern.

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
  has no field budget left, wellbeing flag data should follow the same
  pattern this document lands on for M2 — computed as Analytics formula
  columns from data already parseable out of stored blobs, not dedicated
  fields and not appended blob segments either. The one place this may
  legitimately differ: M3/M4's wellbeing rules are trend-based (comparing
  across multiple `Milestone_Instances` records per patient, per
  `m1-plan.md`), which may be awkward for a single Analytics formula column
  to express — that's a decision for M3's own research/data-model, not
  resolved here, but "Analytics over Deluge" should be the default position
  to argue away from, not toward.
- The unified, cross-milestone Admin Dashboard (User Story 3 / FR-007–009) —
  same deferral M1 already established; M2 only builds its own milestone-scoped
  reporting (User Story 6 baseline bar + the FR-013 flagged view above).
- Raw Zoho Forms retention purge (`data-retention-purge.md`) — same
  cross-milestone open gap M0/M1 already carry; M2 adds a third flow with the
  identical gap, not newly introduced or newly solved here.
- Any automated notification, queue, or routing to the treating contractor on
  a fired flag — explicitly excluded per FR-012/Principle IV, not merely
  unbuilt for lack of time.
