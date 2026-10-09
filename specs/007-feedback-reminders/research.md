# Research: Feedback Reminders

> ON HOLD since 2026-10-09 (see spec.md). Kept for when the hold is lifted.

Facts about the pipeline that this design rests on, with where each came from. Items marked
**unverified** are assumptions to confirm during the build (tasks T007 to T010, T017).

## 1. What the pipeline does today

- Every milestone issues a token with a 7-day life (`ttlDays` 7) and creates exactly one
  `Milestone_Instances` row (`Status` `Issued`, `Token`, `Email`, `Expiry_Date_Time`,
  `Clinician`, plus `Patient` or `Lead_Reference`). Source: `issueFeedbackToken` contract
  (`specs/002-remove-subflow-dependency/contracts/issue-feedback-token.md`); 006 research.md.
- `issueFeedbackToken` marks any older `Issued` row for the same person and milestone
  `Superseded` before creating the new one. A reminder that called it would therefore kill the
  original link and create a second instance. This is why reminders never call it (FR-002).
- `Expired` is written only by the write-back functions, and only when someone submits after
  the expiry time. Nothing polls. An unanswered row therefore stays `Issued` indefinitely.
  Source: the shared write-back structure in each milestone's implementation notes (for
  example `m0-implementation-notes.md` section 3.1). **Unverified in live CRM**; confirm
  with a read-only look at an old test row (T017). This is why reporting must compute
  "lapsed" itself (plan D7).
- `Milestone_Instances` had reached its custom-field cap (M2 data model, 2026-09-10). Costin
  freed one field on 2026-10-09; it is now `Reminder_Sent_Date_Time`.
- Feature 006: native `Email_Opt_Out` on `Patients1` and `Leads`; opt-out does not cancel a
  survey link already sent, which expires normally; every feedback email carries a "stop
  these emails" link built from the request's token; a skipped milestone is not sent
  retroactively. A reminder is a feedback email, so all of it applies.
- Zoho Flow is on the Standard plan: no subflows, and Built-ins offer only Set Variable,
  Decision, Delay and If else (`CLAUDE.md`). No loop, so one scheduled run handles one row.
- M3 research (`m3-research.md`) rejected a polling Flow for M3's own trigger, because
  `Session_Count` changes are a real event. The absence of a response is not an event, so a
  poll is the only way to detect it. That earlier rejection does not apply here.
- Created_Time is system-set and cannot be backdated by API, so the "day 4" rule is keyed
  to `Expiry_Date_Time` (writable), which keeps the function testable with seeded rows
  (data-model.md section 2).

## 2. Per-milestone reasoning (summary)

| Milestone | Why a reminder, and the special handling |
|---|---|
| M0, no conversion | No care relationship, the lowest expected response, and the only chance to learn why the person did not convert. Light copy. |
| M2, early alliance | Early alliance reading; no response means no reading, so the low-alliance flag (001 User Story 4) cannot fire for that patient. |
| M3, sessions 8, 16, 24, 32, 40 | Up to five asks per patient. The fatigue brake stops reminders to someone who let the previous request lapse. |
| M4, discharge | Final alliance reading plus "looking ahead". |
| M5, discontinuation | Person chose to leave; the most likely to find a reminder pushy. One reminder only, softest copy, and none if they returned to care. |

## 3. Alternatives considered and not chosen

- **CRM time-based workflow stamping the field, with a record-triggered Flow on the stamp.**
  Fits the one-record-per-run pattern. Not chosen because the Flow trigger would also fire on
  later updates to a row carrying the stamp (for example when it becomes `Submitted`), so
  every step would need defensive status checks, and CRM time-based actions may depend on
  the CRM edition (**unverified**). A viable fallback if scheduled-run task cost is a
  problem.
- **A status sweep that rewrites stale `Issued` rows to `Expired`.** Not chosen; it modifies
  data to fix a reporting gap that an Analytics formula can close without touching rows
  (Principle VII).
- **Reissuing a fresh token for the reminder.** Not chosen: it supersedes the original link
  and creates a second row, which would double-count requests and break 001 FR-002.
- **Extending the token life at reminder time.** Not chosen; the decision was to keep 7 days.
- **Clinician or staff nudging by phone or message.** Not chosen: forbidden by
  Principle III and the contractor-blindness model.
- **A second reminder.** Decided against by Costin (2026-10-09).

## 4. Risks

- **Flow task budget** (plan open question 2): 9 scheduled runs a day.
- **Claim-before-send** (plan D3) over-reports "reminded" for failed sends; failures are
  alerted, and the field is described as "dispatched".
- **Deluge syntax** (data-model section 3 "confirm" list): timezone `toString`, COQL via
  `invokeurl`, datetime literal format, `addDay`. Each is checked with Execute before use.
- **Alert fatigue**: a persistently unreadable candidate re-alerts each run for at most 3
  days.
- **Response-rate comparison bias**: reminded and unreminded requests are different ages when
  measured, so only closed rows are compared (data-model section 6).
