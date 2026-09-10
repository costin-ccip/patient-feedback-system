# Phase 0 Research: M2 - Early Alliance Check (Session 3)

**Feature**: `specs/001-feedback-collection-pipeline/spec.md` (User Stories 1, 2, and 4 — M2 scope)

**Date**: 2026-09-10

This resolves the unknowns in `m2-plan.md`'s Technical Context before design.

## Trigger mechanism

**Decision**: Trigger off the existing `Session_Count` field on the CRM `Patients1`
module reaching `3`.

**Rationale**: Same field M1 already watches for `== 1` (`m1-research.md`
"Trigger mechanism"); M2 watches the identical field for `== 3`, per spec.md's
Milestones table (row 2) and confirmed against the source Confluence page
"Feedback Collection — CRM Triggers" (pageId 565248003): "2 — Early alliance
check | Session count reaches 3 | Same 'Session count' field; trigger fires at
count = 3." No new CRM field or counting mechanism needed — M2 reuses M1's
already-automated counter unchanged, consistent with Principle II (Single
Rejoin Point: one source of truth, not duplicated per milestone).

**Alternatives considered**: None — this is the same field M1 already
validated; introducing a parallel counter would duplicate a fact CRM already
tracks, which is exactly what Principle II counsels against.

## Survey content

**Decision**: The M2 survey is the **Cape Clarity Alliance Check-In** (custom
instrument), per Confluence page "Milestone 2 — Early Alliance Check (Session
3)" (pageId 564363267, space LP): 4 domains, each a 0-10 slider —

1. Connection — "Overall, how comfortable have you felt being open and honest
   with your therapist so far?" (0 Not comfortable – 10 Very comfortable)
2. Understanding — "Overall, how well do you feel your therapist has
   understood what matters to you so far?" (0 Not understood – 10 Fully
   understood)
3. Shared direction — "Overall, how much do you feel you and your therapist
   agree on what you're working toward?" (0 Not aligned – 10 Fully aligned)
4. Fit of approach — "Overall, how well has your therapist's approach been
   working for you so far?" (0 Hasn't worked – 10 Worked well)

Instruction copy shown to the patient: "You've been meeting with your
therapist for a few sessions now, and we'd love to hear how that's felt for
you so far. There's no right or wrong answer, just your honest sense of
things. Thinking about your experience with your therapist so far, not just
today, mark where you'd place yourself on each line." Closing line: "Thank
you for taking a moment to reflect on this. It helps make sure you're
getting the support that's right for you."

Summed for a 0-40 total; each domain must also be stored individually (not
collapsed into the total) — matches spec.md's Clinical Safety Flag Rules,
which need both the total ("≤20 of 40") and per-domain values ("any single
domain ≤4") independently evaluable.

**Rationale**: This is Costin's own already-designed instrument (same
Confluence-sourcing discipline `m1-research.md` established for the Wellbeing
Check-In), not something to invent during planning.

**Important cross-milestone implication**: per the same Confluence page, this
exact instrument is **reused as-is inside Milestone 3's consolidated survey**
("both pages need to move together if the questions change"). M2 therefore
builds reusable infrastructure (the Alliance Check-In Form and its
scoring/storage/flagging pattern) for M3 to depend on later, same relationship
M1's Wellbeing Check-In has to M3/M4.

## Clinical Safety Flag evaluation — in scope for M2, unlike M1's wellbeing flags

**Decision**: Unlike M1 (which explicitly deferred all flag evaluation to M3
because the wellbeing rules need 2-3 prior readings to compare against — see
`m1-research.md` "Trend-based flagging"), M2 **is** in scope for real flag
evaluation. Per spec.md's ratified Clinical Safety Flag Rules:

> **Alliance** — flag for admin review when either: the total alliance score
> is 20 or below (out of 40); or any single domain scores 4 or below.

This is an **absolute-threshold** rule, not a trend rule — it needs only the
single Session-3 reading M2 collects, not any prior data point. The source
Confluence page confirms this explicitly: "Since this is the first alliance
reading for most patients, there's no personal trend yet, so flagging here
relies on absolute thresholds rather than change-over-time" — and separately,
under Design notes: "First reading uses absolute thresholds since there's no
prior score to compare against; when alliance is re-measured at Milestone 3,
trend-based rules (like Milestone 1's) can supplement this." So M2, not M3, is
where User Story 4 / FR-010–013 first becomes buildable, at least for the
alliance half of the Clinical Safety Flag Rules (the wellbeing half still
starts at M3, unchanged from M1's scoping).

**Rationale**: Building this now — rather than deferring it the way M1
deferred wellbeing flagging — is not optional scope creep; it's what the
ratified spec and source design doc both actually call for at this milestone.
Deferring it here would leave FR-010/FR-011's alliance half unbuilt with no
principled reason, unlike M1 where deferral was the only coherent reading of
the requirement (a trend rule literally cannot evaluate on a first reading).

**Where the evaluation happens**: in the write-back custom function itself
(Deluge), not in Analytics. Unlike M0/M1's write-back functions — which only
ever concatenate raw answers into the `Response_Data` blob and leave all
arithmetic to Analytics formula columns — M2's write-back function already
receives the four domain scores as individual string parameters before
building the blob, so it can parse them to numbers and evaluate both flag
conditions (total ≤20, any domain ≤4) inline, without waiting for an
Analytics sync round-trip. See `m2-data-model.md` for exactly how the result
is stored — **revised 2026-09-10** (see "Storage" below): folded into the
existing `Response_Data` blob, not new CRM fields, because
`Milestone_Instances` is at its CRM custom-field cap.

## Reconciling a stale instruction in the source Confluence page

**Correction**: the M2 Confluence page's own "Flagging" table says a
qualifying reading "routes to the treating clinician and the supervision
queue." This directly conflicts with the ratified `spec.md` (User Story 4,
FR-012): "The system MUST NOT automatically notify a contractor of a clinical
safety flag raised about their own patient. Requesting a case consultation
from that contractor MUST remain a manual decision made by a practice admin."
It also conflicts with constitution Principle IV (Contractor Blindness to Own
Raw Feedback — no contractor access to feedback data of any kind in v1).

This is the same category of conflict spec.md's own revision history already
resolved once, generally, when User Story 4 was added ("per operations-lead
direction, this flags the practice admin, who then decides whether to request
a case consultation from the treating contractor; the contractor is never
notified automatically" — spec.md header comment, "Revised 2026-09-05 (second
pass)"). Per that same operations-lead direction and FR-012, M2's build routes
a triggered flag to admin-visible storage only — **no automated notification
or queue-routing to the treating clinician is built**. The Confluence page's
"supervision queue" language is treated as stale/superseded on this point,
the same way `m0-implementation-notes.md` (§5) already treats the CRM's dead
`5A - Discontinuation, Live Capture` configuration as stale, unused design
left over from before a later product decision.

## FR-013 / admin visibility — milestone-scoped, not the full Admin Dashboard

**Decision**: FR-013 requires a flag to be "visible to admins within that
patient's dashboard view (FR-007)." FR-007's unified Admin Dashboard is User
Story 3, explicitly out of scope for any single milestone (same deferral
already established by `m1-data-model.md` "Out of scope for M1" — it "reasonably
waits until enough milestone data exists for that aggregation to mean
anything," per spec.md's own User Story 6 rationale distinguishing it from the
per-milestone reporting floor). M2 satisfies FR-013 the same way M1 satisfied
FR-018 ahead of the full dashboard: by building a milestone-scoped view of its
own flagged records (see `m2-data-model.md`, `m2-tasks.md` Phase 5) — not by
building FR-007's cross-milestone dashboard early. This keeps M2's own flag
data genuinely visible to admins today (satisfying SC-007 for M2's own
records) without pulling the separately-scoped User Story 3 work forward.

**Rationale**: Matches the scope-boundary discipline spec.md's User Story 6 already
established for M1 ("This story is a floor each milestone clears on its own,
immediately, and is not gated on Story 3 or on any other milestone existing")
— applied here to FR-013 rather than FR-018, since both are "make this
milestone's own data visible now, without waiting for the unified dashboard."

## Delivery mechanism

**Decision**: Reuse M0/M1's pattern unchanged — Zoho Form (public, token-only,
no PII) + Zoho Flow custom function write-back to `Milestone_Instances`,
delivered via Zoho Mail, via the same shared "Subflow - Issue Feedback Token"
M0 and M1 already use unmodified.

**Rationale**: No new delivery surface is needed; M1 already validated this
exact path end-to-end, and the constitution's BAA-gated adoption principle
(V) favors reusing an already-validated tool over introducing a new one.

## Storage: dedicated fields vs. delimited blob

**Decision**: delimited blob, same as M0/M1's `Response_Data` pattern —
`Milestone_Instances` stays schema-light per the field-budget rationale
`m1-plan.md`'s Complexity Tracking already recorded.

**Revised 2026-09-10 (Costin)**: this session's first draft proposed two new
dedicated CRM fields (`Clinical_Safety_Flag`, `Flag_Rule_Triggered`) for the
flag data, reasoning that computed values are awkward to keep re-parsing out
of a blob. Costin corrected this: `Milestone_Instances` is **at its CRM
custom-field cap** — no new custom fields can be added to this module,
full stop. This isn't a new constraint invented for M2; it's exactly the
constraint `m1-plan.md`'s Complexity Tracking already named as the reason to
keep the blob pattern in the first place ("conserve `Milestone_Instances`'
CRM field budget for M2 and M5"), now hit in practice.

**Final decision**: the two flag values are appended as two more segments on
the existing `Response_Data` blob (see `m2-data-model.md`), not new fields.
Detection/filtering happens via two new **Zoho Analytics formula columns**
(same `substring_between`/guarded-`SUBSTR` pattern already used for every
other parsed value) — Analytics formula columns are not CRM custom fields and
are not subject to the module's field cap, so this fully sidesteps the
limitation while keeping the same "parse the blob in Analytics" architecture
every other milestone already uses. No `createFields` CRM call is needed for
this milestone at all.
