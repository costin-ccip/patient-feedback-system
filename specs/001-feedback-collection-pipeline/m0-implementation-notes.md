# M0 (Free Consult / No-Conversion) — Implementation Notes

> Scope: this file documents ONLY the M0 milestone (the free-consult, no-conversion
> feedback loop). It is a companion to `specs/001-feedback-collection-pipeline/spec.md`,
> which covers all six milestones at the requirements level. This file exists to let a
> future session (human or AI) understand exactly what was built, why, and where the
> sharp edges are, without having to reverse-engineer it from Zoho.
>
> Last updated: 2026-09-30
> Status: M0 is built end-to-end (trigger → form → CRM write-back → Analytics
> reporting). As of this update, both "M0 - Lost Lead Feedback Token" and "M0 -
> Feedback Survey Write-back" are deliberately toggled OFF in Zoho Flow — per
> Costin, several flows are intentionally paused pending coordinated
> end-to-end testing (see `m1-implementation-notes.md`), not because of a
> defect. See "Process note" at the end — this milestone was built ad hoc,
> outside the spec-kit plan/tasks workflow the project constitution calls for.
> See §11 for a 2026-09-08 correction to the token-issuance trigger mechanism.
> **2026-09-23: token issuance no longer uses a subflow. See §12.**
> **2026-09-25: the public survey form was rebuilt as a new Zoho Forms object after
> the original's permalink started 404ing (Zoho-side bug, not a config error). The
> form Costin/patients see is still titled "M0 - Free Consult Non-Conversion
> Survey"; only its underlying object and permalink changed. See §13.**
> **2026-09-28: the patient-facing email's Subject and Body were redesigned to
> match the real Zoho Bookings confirmation template, with new Tone-of-Voice copy.
> Applied live. See §14 — also the reference section for M2/M3/M4/M5's matching
> changes.**
> **2026-09-28: the survey itself was simplified — the "Okay to reach back out" and
> "Anything else on your mind" fields were removed from the form, the write-back
> function, and Analytics, per Costin's direct instruction. Both M0 flows were
> turned OFF before this edit. See §15 — §4, §6 and §7 below are corrected in place
> to reflect the new as-built state; read §15 first if anything below looks
> inconsistent with it.**
> **2026-09-30: the trigger condition was narrowed. M0 now fires only when a Lead is
> marked Lost Lead AND its Consult Call Date is populated (a free consult actually
> happened); previously every Lost Lead fired it. See §16; §1 and §2 are corrected
> in place.**

## 1. What M0 does

M0 fires when a lead has a free consultation and does not convert to a paying client.
Token issuance is automatic: when someone on the team sets that Lead's `Lead_Status`
field to `Lost Lead` in CRM — a normal part of working the lead, not a separate
"start feedback" action — **and the Lead's `Consult_Call_Date` is populated (§16;
before 2026-09-30 every Lost Lead qualified, including leads who never had a free
consult)** — a realtime trigger flow fires and issues the token. (An
earlier version of this doc described this as a step the clinician triggered by
hand; §11 corrects that.) The lead receives a link to a public Zoho Form. When they submit it, a Zoho Flow captures
the response, matches it back to the correct `Milestone_Instances` CRM record by
token, validates the token hasn't expired and hasn't already been used, and writes the
parsed response data back onto that CRM record. Zoho Analytics syncs from CRM and
exposes the parsed fields as report-ready columns, which feed the M0 dashboard.

De-identification: the survey and its response data are keyed to a `Milestone_Instances`
record and a `Lead_Reference` (a plain text field holding the Lead's CRM record ID),
not to any patient identity. No `Patient` module fields are populated or referenced by
this flow (see §5, `Patient` field note).

## 2. Component inventory

| Component | Name | Where |
|---|---|---|
| Public form | M0 feedback survey (Zoho Forms) | Linked from the token issuance email/flow |
| Trigger flow | "M0 - Lost Lead Feedback Token" | Zoho Flow, folder "Customer Feedback System" — realtime "Updated module entry" on `Leads`, fires when `Lead_Status` (picklist field, confirmed via CRM `getFields`; label "Lead Status") equals `Lost Lead` AND `Consult_Call_Date` is populated (trigger filter, §16). Calls the shared subflow below. See §11. |
| Token issuance (shared) | "Subflow - Issue Feedback Token" | Zoho Flow, folder "Customer Feedback System" — called by the trigger flow above; also reused by M1 |
| Write-back flow | "M0 - Feedback Survey Write-back" | Zoho Flow, folder "Customer Feedback System" |
| Write-back logic | Custom Deluge function `submitFeedbackResponse` | Inside the write-back flow |
| CRM module | `Milestone_Instances` | Zoho CRM |
| Analytics workspace | workspace ID `3251423000000083002` | Zoho Analytics |
| Analytics table | "Milestone Instances" | Synced from CRM module, same workspace |
| Dashboard | "M0 - Free Consult Non-Conversion Feedback" | view ID `3251423000000083524` |

## 3. The write-back flow, step by step

Flow: **M0 - Feedback Survey Write-back** (folder: Customer Feedback System)

1. **Trigger**: "Form entry submitted" (Zoho Forms, realtime) — fires when the M0
   survey form is submitted.
2. **Action**: custom function `submitFeedbackResponse`, called with the form fields
   mapped as follows:

   | Function parameter | Form field |
   |---|---|
   | `token` | `${trigger.SingleLine}` |
   | `ratingHeard` | `${trigger.Rating}` (Feeling Heard Score, 1-5) |
   | `reasonNotMovingForward` | `${trigger.Radio}` (Biggest Factor) |
   | `addAnything` | `${trigger.MultiLine}` (Additional Comments) |

   **(2026-09-28: `reachBackOk`/`${trigger.Radio1}` and `anythingElse`/
   `${trigger.MultiLine1}` were removed — see §15. The table above shows the
   current, 4-parameter mapping.)**

### 3.1 `submitFeedbackResponse` — verbatim Deluge source

**Captured live on 2026-09-28, after the §15 signature change — this is the
current, authoritative source.** The original 2026-09-06 capture had six
parameters (`reachBackOk`, `anythingElse` included); see §15 for the exact
before/after diff if you need the retired version.

```
map submitFeedbackResponse(string token, string ratingHeard, string reasonNotMovingForward, string addAnything)
{
	resp = Map();
	if(token == null || token.trim() == "")
	{
		resp.put("status","error");
		resp.put("message","Missing token");
		return resp;
	}
	rec = Map();
	found = false;
	qmap = Map();
	allRecs = zoho.crm.getRecords("Milestone_Instances",1,200,qmap,"crm_connection");
	for each  r in allRecs
	{
		if(r.get("Token") == token)
		{
			rec = r;
			found = true;
		}
	}
	if(!found)
	{
		resp.put("status","error");
		resp.put("message","No Milestone Instance found for token");
		return resp;
	}
	recStatus = rec.get("Status");
	expiry = rec.get("Expiry_Date_Time");
	if(recStatus != "Issued")
	{
		resp.put("status","error");
		resp.put("message","Milestone Instance is not in Issued status (current: " + recStatus + ")");
		resp.put("recordId",rec.get("id"));
		return resp;
	}
	if(expiry != null && expiry != "" && expiry < zoho.currenttime)
	{
		expUpdateMap = Map();
		expUpdateMap.put("Status","Expired");
		zoho.crm.updateRecord("Milestone_Instances",rec.get("id"),expUpdateMap,Map(),"crm_connection");
		resp.put("status","error");
		resp.put("message","Token has expired");
		resp.put("recordId",rec.get("id"));
		return resp;
	}
	responseText = "How much did you feel heard and understood (1-5): " + ifnull(ratingHeard,"") + "---";
	responseText = responseText + "Biggest factor in decision: " + ifnull(reasonNotMovingForward,"") + "---";
	responseText = responseText + "Anything you would like to add: " + ifnull(addAnything,"") + "---";
	updateMap = Map();
	updateMap.put("Response_Data",responseText);
	updateMap.put("Status","Submitted");
	updateMap.put("Submitted_Date_Time",zoho.currenttime.toString("yyyy-MM-dd'T'HH:mm:ssXXX"));
	updateResp = zoho.crm.updateRecord("Milestone_Instances",rec.get("id"),updateMap,Map(),"crm_connection");
	resp.put("status","success");
	resp.put("message","Milestone Instance updated");
	resp.put("recordId",rec.get("id"));
	return resp;
}
```

### 3.2 Logic notes

- Token lookup is a linear scan (`zoho.crm.getRecords` up to 200 records, looped) —
  not an indexed/COQL search. Fine at current volume; if `Milestone_Instances` grows
  past ~200 active records this may need to switch to `zoho.crm.searchRecords` or
  `executeCOQLQuery`.
- Idempotency / re-submission protection comes from the `Status != "Issued"` check —
  once a token is used (`Submitted`) or expires (`Expired`), a second submission with
  the same token is rejected, not overwritten.
- Expiry is checked and enforced (auto-transitions the record to `Expired`) even
  though the token is otherwise valid, if `Expiry_Date_Time` has passed.

## 4. The Response_Data contract

**Corrected 2026-09-28 — see §15.** `Milestone_Instances.Response_Data` is a
single text field holding the three open answers concatenated with a `---`
(triple-hyphen) delimiter, in this fixed order, with a trailing `---` after the
last field (kept deliberately — see §15 for why):

```
How much did you feel heard and understood (1-5): {value}---Biggest factor in decision: {value}---Anything you would like to add: {value}---
```

(Historical format, live 2026-09-06 through 2026-09-28, before the "Okay to
reach back out" and "Anything else on your mind" fields were removed:
`...Anything you would like to add: {value}---Okay to reach back out:
{value}---Anything else on your mind: {value}`, no trailing delimiter.)

**Why a blob instead of 5 separate CRM fields:** `Milestone_Instances` is a shared
module across all six milestones, and Zoho CRM custom fields are capped; storing every
milestone's open-ended answers as dedicated fields would exhaust that cap quickly.
The blob-plus-parsing-in-Analytics approach keeps CRM schema-light and pushes
structure into Analytics formula columns instead.

**Why `---` and not `\n\n` or `|`:** an early version used newline-based delimiting
and a later attempt used `|`. Pipe-delimiting broke because the Zoho Flow Deluge
script editor auto-closes brackets/quotes, which corrupted literal `|` characters
inside string literals when edited in-browser. `---` was chosen as a delimiter that's
extremely unlikely to appear in a genuine free-text answer and is immune to that
editor bug.

**Backward compatibility:** the very first test records were written with the older
`\n\n`-delimited format before this contract was finalized. All formula columns except
"Anything Else" depend on `substring_between` bounded by `---`, so they will not parse
old-format data correctly. The "Anything Else" column (see §6) is deliberately built
without relying on the delimiter for its start boundary, which happens to make it the
one column that still parses correctly on old-format data. All real M0 test data was
deleted and replaced with format-consistent samples (see §8), so this is a historical
note, not a live problem.

## 5. Milestone_Instances CRM field reference

Retrieved via `getFields` on `Milestone_Instances`.

| Field (API name) | Type | Notes |
|---|---|---|
| `Name` | text | Mandatory on create. Free text label, e.g. "Sample M0 - Found Provider". |
| `Lead_Reference` | **text** | NOT a CRM lookup — stores the Lead's record ID as a plain string. Easy to mistake for a lookup; it isn't one. |
| `Patient` | lookup | Points to a module with `api_name: "Patients1"` — this is effectively the (renamed/hidden) Patients module. **Never populate or query this field from any automation or report** — it's PII and out of scope/access for this system per standing project rules. |
| `Milestone` | picklist | Values: `-None-, 0 - No Conversion, 1 - Baseline Intake, 2 - Early Alliance Check, 3 - Periodic Consolidated, 4 - Discharge, 5B - Discontinuation Email Fallback, 5A - Discontinuation Live Capture`. M0 uses `"0 - No Conversion"`. **`5A - Discontinuation Live Capture` is dead, unused configuration** — per the ratified `spec.md` Assumptions (confirmed by a separate session's live-CRM inspection), Milestone 5 was built before Costin's decision to drop the live-call path and use email only; `5A` and its associated `Staff Live Entry` capture method / `Captured Live` status should be disregarded, not treated as part of the current design, and are candidates for CRM cleanup later. `5B` (email) is the live path. |
| `Status` | picklist | Values: `-None-, Issued, Submitted, Expired, Superseded, Captured Live`. |
| `Clinician` | picklist | Values: `-None-, Liana Preudhomme`. |
| `Token` | text | Unique token embedded in the survey link. |
| `Expiry_Date_Time` | datetime | Token expiry; enforced by `submitFeedbackResponse` (see §3.1). |
| `Submitted_Date_Time` | datetime | Set by the write-back function on successful submission. |
| `Response_Data` | text (long) | The delimited blob — see §4. |

## 6. Zoho Analytics — formula columns (all on the "Milestone Instances" table)

**2026-09-28: the "Okay to Reach Back Out" and "Anything Else" columns below
were deleted from Analytics — see §15.** They're kept here, clearly marked, as
a historical record of what they computed; only "Feeling Heard Score,"
"Biggest Factor," and "Additional Comments" are current/live.

All five verbatim, as saved in Analytics on 2026-09-06:

**Feeling Heard Score**
```
SUBSTR("Milestone Instances"."Response Data", INSTR("Milestone Instances"."Response Data",'How much did you feel heard and understood (1-5): ')+LENGTH('How much did you feel heard and understood (1-5): '), INSTR("Milestone Instances"."Response Data",'---')-INSTR("Milestone Instances"."Response Data",'How much did you feel heard and understood (1-5): ')-LENGTH('How much did you feel heard and understood (1-5): '))
```

**Biggest Factor**
```
substring_between("Milestone Instances"."Response Data", 'Biggest factor in decision: ', '---', 1)
```

**Additional Comments**
```
substring_between("Milestone Instances"."Response Data", 'Anything you would like to add: ', '---', 1)
```

**[RETIRED 2026-09-28] Okay to Reach Back Out**
```
substring_between("Milestone Instances"."Response Data", 'Okay to reach back out: ', '---', 1)
```

**[RETIRED 2026-09-28] Anything Else** (last field — no trailing `---` to bound it, so this one can't use `substring_between`)
```
SUBSTR("Milestone Instances"."Response Data", INSTR("Milestone Instances"."Response Data",'Anything else on your mind: ')+LENGTH('Anything else on your mind: '), LENGTH("Milestone Instances"."Response Data"))
```

### Gotchas discovered building these

- `substring_between(string, sub1, sub2, [start_pos])` is a real Zoho Analytics
  function — not obvious from the standard docs, found via the in-editor Functions
  panel tooltip. It returns the text between the first and second substring, searching
  from `start_pos` (1-indexed, default 1). Much cleaner than manual `INSTR`/`SUBSTR`
  arithmetic once you know it exists.
- `INSTR` in Zoho Analytics formulas takes 2 arguments (string, substring) — there is
  no 3-argument `INSTR(string, substring, start_pos)` overload. An earlier attempt to
  use a 3-arg `INSTR` to work around ambiguous repeated substrings failed for this
  reason; `substring_between`'s own `start_pos` argument was the actual fix.
- Only "Anything Else" needs the manual `SUBSTR`/`INSTR`/`LENGTH` combination, because
  it's the last field in the blob with no closing delimiter to bound it — there's
  nothing for `substring_between`'s second argument to match.

## 7. Zoho Analytics — reports and dashboard

**Corrected 2026-09-28 — see §15.** Workspace: `3251423000000083002`.

| Report | Type | Purpose |
|---|---|---|
| M0 Status Breakdown | (pre-existing) | Issued/Submitted/Expired counts |
| M0 Response Rate | (pre-existing) | Submitted ÷ Issued |
| M0 Volume by Week | (pre-existing) | Issuance trend |
| M0 Submitted Responses | Tabular | Submitted Date, Lead Reference, Biggest Factor, Additional Comments. Raw `Response Data` blob was removed from this view once the parsed columns existed — no reason to show clinicians/ops the raw delimited string. "Anything Else" was removed from this report's column list on 2026-09-28 (§15) when the underlying field was dropped from the survey. |
| M0 Biggest Factor | Chart (bar) | X = Biggest Factor, Y = Count of Id |
| M0 Feeling Heard Distribution | Chart (bar), view `3251423000000083576` | X = Feeling Heard Score, Y = Count of Id |
| ~~M0 % Reachable~~ | ~~Summary/KPI, view `3251423000000083587`~~ | **[RETIRED 2026-09-28, §15]** Deleted entirely (report/view, dashboard panel, and its underlying workspace Aggregate Formula "% Reachable") when "Okay to Reach Back Out" was dropped from the survey. Formula was: `count_if("Milestone Instances"."Okay to Reach Back Out"='Yes' OR "Milestone Instances"."Okay to Reach Back Out"='Maybe')*100/count_if("Milestone Instances"."Status"='Submitted')` |

Dashboard **"M0 - Free Consult Non-Conversion Feedback"** (view `3251423000000083524`)
now has **6 panels** (down from 7 as of 2026-09-28, §15): the three pre-existing
status/rate/volume panels, the Submitted Responses table (now 4 columns), and the
Biggest Factor / Feeling Heard Distribution charts. The "M0 % Reachable" KPI panel
that used to sit alongside these was removed — see §15.

**UX rationale, as originally designed** (historical — kept for context; the
second and third bullets describe panels/columns retired 2026-09-28, §15):
- Feeling Heard Score (1-5 rating) → distribution bar chart, because a single average
  hides whether the practice is polarizing (lots of 1s and 5s) vs. consistently
  mediocre (lots of 3s) — different variance profiles need different action. **Still
  current.**
- ~~Okay to Reach Back Out (Yes/Maybe/No) → a single "% Reachable" KPI tile
  (Yes + Maybe, over Submitted), because the actionable question is "how many of
  these people could we still follow up with," not the raw 3-way split.~~ **Retired
  — the field itself was dropped from the survey.**
- Additional Comments (open text) → parsed into a real column on the existing
  tabular report rather than a new visualization, since open text doesn't
  aggregate; the win was just making it readable instead of buried in a blob.
  (Originally applied to "Anything Else" too; that field is retired.)

## 8. Test/sample data approach

CRM records were managed directly via the Zoho CRM MCP tools (`getRecords`,
`deleteRecords`, `createRecords`), not through the browser — the Leads and Patients
modules are off-limits for direct browsing per standing project rule, and
`Milestone_Instances` itself contains no PII, so tool-based CRUD was fine.

- All old/ad hoc test `Milestone_Instances` records were deleted.
- 4 new sample records were created, all referencing the same test Lead
  (`Lead_Reference: "6825601000003999001"`), all `Milestone: "0 - No Conversion"`,
  `Status: "Submitted"`, `Clinician: "Liana Preudhomme"` — varying Feeling Heard Score
  (1, 3, 4, 5), Biggest Factor, all three Okay-to-Reach-Back-Out states, and a mix of
  blank/filled open-text fields, specifically to exercise every dashboard panel above.
- Gotchas: `deleteRecords`' `ids` parameter must be passed as a single comma-separated
  **string**, not a JSON array, despite the tool schema showing it as an array type —
  an array throws `UNABLE_TO_PARSE_DATA_TYPE`. `createRecords` requires `Name` even
  though it isn't documented as obviously mandatory in casual field review — omitting
  it throws `MANDATORY_NOT_FOUND`.

## 9. Access constraints that shaped this build

Per standing project rule, Claude sessions working on this system do not open the
CRM's Leads or Patients modules in the browser without explicit permission. This is
why: (a) CRM record management for `Milestone_Instances` was done via MCP tool calls
rather than browser automation, (b) the real public survey form was tested by asking
Costin to trigger status changes / view Lead-side state manually rather than Claude
navigating there, and (c) the `Patient` lookup field (§5) is treated as untouchable by
any automation in this system.

## 10. Process note — spec-kit deviation

The project's `.specify/memory/constitution.md` (Rollout Workflow section) states:

> "Each new milestone, delivery channel, or dashboard added to this system MUST go
> through this same spec-driven process (specify → plan → tasks) rather than being
> configured ad hoc directly in Zoho Flow."

M0 was **not** built this way — it was implemented directly in Zoho Flow/CRM/Analytics
through iterative conversation, with no `plan.md` or `tasks.md` ever generated for it.
`specs/001-feedback-collection-pipeline/spec.md` covers M0 at the requirements level
(it's one of the six milestones in that single spec), but the actual build skipped the
`/speckit-plan` and `/speckit-tasks` steps entirely.

This file is offered as retroactive input material if Costin later wants to backfill a
formal `plan.md`/`tasks.md` for M0 to bring it into compliance with the constitution,
and as a cautionary note for future milestones: the constitution's own rule is to spec
first, not after the fact.

## 11. Correction (2026-09-08) — token issuance is automatic, not manual

A session picking up M1 work found a flow in Zoho Flow, "M0 - Lost Lead Feedback
Token," that isn't named anywhere in this file or in `m0-plan.md`/`m0-tasks.md`.
Those documents describe M0's token issuance as a manual step the clinician
performs by hand (framed as a documented Principle III fallback, since a
free-consult non-conversion has no natural automatic trigger condition). Checking
the live flow's execution history (filter criteria: `Lead_Status equals Lost
Lead`) and confirming the field via CRM `getFields` (schema only, no Lead records
opened — `Lead_Status`, label "Lead Status", picklist, includes the value `Lost
Lead`) showed this is not what's actually built.

**Confirmed with Liana (2026-09-08)**: the real mechanism is that someone on the
team sets a Lead's `Lead_Status` to `Lost Lead` in CRM as a normal part of working
that lead — not a dedicated "start feedback" action. That field change fires "M0 -
Lost Lead Feedback Token" (a realtime "Updated module entry" trigger) automatically,
which calls the shared "Subflow - Issue Feedback Token" to create the
`Milestone_Instances` record, generate the token, and send the email. There is no
separate manual token-issuance step beyond the ordinary act of marking a lead lost.

This is a real, load-bearing correction, not just a naming fix: `m0-plan.md`'s
Constitution Check flags an open question against Principle III (Automated
Milestone Triggers) specifically because it believed M0's issuance was manual.
With an automatic CRM-field trigger in place — the same pattern M1 uses for
`Session_Count` — that open question is substantially addressed, though whether
"a team member changes a Lead's status" counts as sufficiently automatic (versus
M1's fully system-detected `Session_Count` threshold) is still worth Costin's
explicit sign-off rather than assuming resolved by this note alone. `m0-plan.md`
and `m0-tasks.md` (T008/T010) have not been rewritten to preserve their original
retroactive record, but T010 should be treated as resolved — see the dated
addendum at the end of each of those files rather than treating their original
Constitution Check / task text as current.

Separately, both "M0 - Lost Lead Feedback Token" and "M0 - Feedback Survey
Write-back" were found toggled OFF in Zoho Flow on 2026-09-08. Per Costin, this is
intentional — several flows are deliberately paused pending a coordinated
end-to-end test of M0 and M1 together, not a sign anything is broken. Execution
history shows real runs going back to 2026-09-03 (16 total: 4 completed, 8
filtered, 4 failed); this session did not open individual executions' input/output
data (which would show a real Lead's name/email), per the standing restriction on
viewing Lead/Patient PII in the browser, so whether those runs processed real or
test leads was not verified here.

## 12. Change (2026-09-23): token issuance no longer uses a subflow

"M0 - Lost Lead Feedback Token" is now
`Updated module entry (Leads, Lead_Status → Lost Lead) → issueFeedbackToken → Send email`.
`issueFeedbackToken` parameters: milestone `0 - No Conversion`, patientId empty,
leadId `${trigger.id}`, recipientEmail `${trigger.Email}`, clinician
`Liana Preudhomme`, ttlDays `7`. Email subject "A quick check-in from Cape Clarity";
intro "We'd love to hear how things are going. Please take a moment to share your
feedback using the link below:"; survey link
`https://forms.zohopublic.com/lianapreudhommecapec1/form/M0FreeConsultNonConversionSurvey/formperma/GFdd7kA1Yp8jIxhsMuTJTN1QDEiSl5K6FoQjceHsnAg`.
(These M0 subflow parameter values were never written down in this file before;
they were captured live on 2026-09-23 before the old node was deleted.) M0 still has
no idempotency check of its own; the supersede step inside `issueFeedbackToken` is
its duplicate protection, exactly as it was inside the subflow.

This is the as-built change for `specs/002-remove-subflow-dependency` (subflows
aren't available on the Zoho Flow Standard plan). The flow's "Call a subflow →
Subflow - Issue Feedback Token" step was deleted and replaced with two steps: the
shared custom function `issueFeedbackToken` (output variable `issueFeedbackToken_1`;
does supersede + token/expiry + Milestone_Instances create, the same logic the
subflow ran) and a native **Zoho Mail "Send email"** step (connection "Connection to
info@capeclarity.com", From `info@capeclarity.com`, To `${trigger.Email}`, same
subject/intro/survey link as before, link ends `?token=${issueFeedbackToken_1.token}`).
Record shape, email content and sender are unchanged. The flow stays **OFF**. Full
function source, per-flow values and new builder gotchas:
`specs/002-remove-subflow-dependency/implementation-notes.md`. The retired subflow's
internals (never documented here before) are in that feature's `research.md` §0.
Earlier text in this file that describes "Call a subflow" is kept as history and no
longer describes the live build.

The component table in §2 is superseded on two rows: "Token issuance (shared)" is now
the custom function `issueFeedbackToken` (used by M0, M2, M3), and
"Subflow - Issue Feedback Token" is renamed `[RETIRED] Subflow - Issue Feedback
Token`, OFF, kept for audit.

**Follow-up (2026-09-23, feature 003)**: record names no longer contain the
recipient email (`Milestone <m> - <token prefix>`), and this flow now has On Error
branches: issuance failure → alert to costin@capeclarity.com; patient-email failure →
record Status `Send Failed` → alert. Resend runbook:
`specs/003-issuance-privacy-and-failure-handling/quickstart.md` §C.

## 13. Fix (2026-09-25): M0 public form permalink 404'd — form rebuilt as a new object

Costin began manually running the coordinated live test (per his own standing
instruction that he runs live tests, not Claude) and reached M0. He received the
token-issuance email as expected, but the survey link 404'd:
`https://forms.zohopublic.com/lianapreudhommecapec1/form/M0FreeConsultNonConversionSurvey/formperma/GFdd7kA1Yp8jIxhsMuTJTN1QDEiSl5K6FoQjceHsnAg`
(the exact permalink recorded in §12) returned "Sorry! Page not found." on direct
navigation.

**Diagnosis.** Confirmed via `read_network_requests` that the document-level GET to
that formperma URL itself returned HTTP 404 (not a client-side rendering issue).
Ruled out, in order: a copy/paste typo (the URL matched byte-for-byte across three
independent UI renderings — Share tab, Custom Form Link Name settings, and a
zoomed screenshot compare); a token/prefill problem (reproduced the 404 with and
without the `?token=` query param); account-wide misconfiguration (a sibling form,
M2's "Cape Clarity Alliance Check-In," resolves fine on the identical
`forms.zohopublic.com/.../formperma/...` URL pattern on the same account); a
billing/plan restriction (subscription is Basic Plan, active and paid through 18
Sep 2027); and a custom domain or Custom Form Link Name setting (neither
configured for this form). Toggling the form's "Share Publicly" setting off and
back on — the standard remedy for this class of Zoho Forms glitch — did **not**
fix it; the 404 was reproduced again immediately afterward. This was a genuine,
reproducible Zoho-side bug specific to that one form object, not a config error on
our side.

**Fix.** Costin approved rebuilding the form fresh rather than continuing to chase
the underlying Zoho bug. Used Zoho Forms' own "Duplicate" action (My Forms → "⋮" on
the form → Duplicate) to clone "M0 - Free Consult Non-Conversion Survey" into a new
form object — same questions, same field structure, new internal form ID and a
new permalink token. The duplicate's public formperma URL was verified to resolve
(HTTP 200 via `read_network_requests`, confirmed with the browser's own "Access
Form" link so the URL was never hand-transcribed):

```
https://forms.zohopublic.com/lianapreudhommecapec1/form/M0FreeConsultNonConversionSurveyNEW/formperma/yoRiV9UKbs18x9p5nfIBADh_KlyZ2sriJF6ytCwk1XA
```

(URL slug `M0FreeConsultNonConversionSurveyNEW` is a permanent artifact of
duplication-time naming and does not change when the form's display title is
edited afterward — see renaming below.)

Two downstream components had to be repointed at the new form object (duplicating
a Zoho Form does **not** automatically move flows that reference the old one):

1. **"M0 - Feedback Survey Write-back"**'s trigger ("Form entry submitted", Zoho
   Forms) was reconfigured from the old form to the new one. Re-verified — did not
   assume — that the `submitFeedbackResponse` parameter mappings (§3) still
   resolved correctly against the new form's field schema: all six parameters
   (`token`, `ratingHeard`, `reasonNotMovingForward`, `addAnything`, `reachBackOk`,
   `anythingElse`) still showed bound, resolved field labels in the Insert
   Variable panel after the trigger form was switched, with no "field not found"
   errors. This confirms Zoho's Duplicate preserved the same internal field
   API names (`SingleLine`, `Rating`, `Radio`, `MultiLine`, `Radio1`,
   `MultiLine1`) from §3's table — expected but not assumed, and now confirmed.
2. **"M0 - Lost Lead Feedback Token"**'s Send Email step body was updated to the
   new permalink above, keeping the same `?token=${issueFeedbackToken_1.token}`
   suffix. (First attempt at this edit accidentally swallowed the `?` character
   when replacing a partial text selection in the rich-text body editor — caught
   by re-zooming on the edited text before saving, and fixed by triple-clicking to
   select the whole line and retyping it complete, including the token suffix.
   Worth flagging for any future session editing this same rich-text field: verify
   the full string after any partial-selection edit, don't assume the boundary
   landed where intended.)

Both flow changes were applied to the live flows (Zoho Flow's "Apply changes" on a
Draft), not left as unpublished drafts.

**Renaming.** Old form renamed `[RETIRED] M0 - Free Consult Non-Conversion Survey`
(Form Properties → Form title, in the Builder — Zoho Forms has no direct
"Rename" action in the dashboard's "⋮" menu). New form's Form Properties title was
set to the canonical `M0 - Free Consult Non-Conversion Survey` (its URL slug stays
`M0FreeConsultNonConversionSurveyNEW`, which is cosmetic-only and does not need to
match the display title). The retired form's public sharing was left as-is
(already effectively dead since it 404s); it was not explicitly disabled.

**Component table (§2) correction**: the "Public form" row's underlying Zoho Forms
object changed from the original (now `[RETIRED]`) form to the new one at the
`M0FreeConsultNonConversionSurveyNEW` slug; the display title Costin and patients
see is unchanged (`M0 - Free Consult Non-Conversion Survey`).

No CRM data, Analytics formulas, or the `submitFeedbackResponse` function body
changed — this was purely a Zoho Forms object swap plus repointing the two flows
that referenced it.

## 14. Change (2026-09-28): patient-facing email redesigned — real Zoho Bookings template, new ToV copy

Costin asked whether the milestone emails (M0, M2, M3, M4, M5) could look as
polished as the free-consult booking-confirmation email, then supplied the actual
production HTML behind that confirmation (a Zoho Bookings notification template)
to build from directly, rather than an invented design. New copy was drafted in
Cape Clarity's Tone of Voice (Confluence: "Cape Clarity ToV"), reviewed with Costin
via a review artifact, revised per his pointed edits (below), and applied directly
to each milestone's live/paused "Send email" step — Subject and Body only. No
trigger, function, On Error branch, or CRM write changed in any of the five flows.
**This section documents the shared design/rationale; M2/M3/M4/M5's own
implementation-notes files each have a matching, shorter section for their own
before/after copy and cross-reference here.**

**What changed and why (applies to all five milestones):**
- Layout/markup rebuilt from the real Zoho Bookings confirmation HTML: same
  hosted header-band image
  (`https://cdn.prod.website-files.com/.../cape-clarity-email-header-band.png`),
  card border/radius, `"Open Sans", "Trebuchet MS"` font stack, and link-green
  `#175328` (read directly from the real template, not approximated from a
  screenshot). Zoho Bookings' own `%customername%`/`%staffname%`-style merge tags
  don't apply here — these emails send through Zoho Flow's "Send email" step,
  which merges with Flow's own `${...}` chip syntax instead, so the survey link
  uses `${issueFeedbackToken_1.token}`. The greeting is a fixed "Hi there," rather
  than a first-name merge — none of these triggers' modules expose a first-name-
  only field (established during M5's build, §10).
- Link style: each email now shows a single full-URL link (the URL is both the
  `href` and the visible link text), replacing the old pattern of a short intro
  sentence followed by a bare-text URL on its own line. Costin's call: a full URL
  as the link itself is clearer to the recipient about what they're clicking, and
  it stays close to the plain-bare-URL pattern already proven safe on every
  milestone — avoiding a repeat of the M5 §10 bug, where a styled `<a>` with a
  *nested* `<b>` tag got mangled into literal `[text](url)` text by Zoho Flow's
  renderer. This markup has no nested tag inside the anchor.
- Copy: shorter and more direct throughout per Costin's edits — no em dashes, no
  exact-time-expectation phrasing ("it only takes a minute" language), and
  milestone-specific framing changes documented in each file's own section (M2's
  confidentiality line, M3's benefit framing, M5's opening line).

**Verification method used for every milestone** (worth reusing for any future
edit to a Zoho Flow "Send email" Body field): the Body field's HTML code-view is a
plain `<textarea class="ze_editor_textarea">`, not a CodeMirror instance (confirmed
via DOM inspection — a `.CodeMirror.setValue()` shortcut does **not** apply here,
unlike the Deluge function editors elsewhere in this project). Content was
select-all + retyped in full (never a partial-range edit — see §13's swallowed-`?`
lesson), then verified via `javascript_tool` by reading `textarea.value` directly
and checking: exact character length against the source fragment, exactly one
`href="` (single-link pattern), the token chip appearing exactly twice (once in
`href`, once as the visible link text), the string starting with
`<div class="cc-email-notification-container"` and ending with `</div>`, and the
old copy's key phrase being absent while the new phrase is present — all before
clicking Done/Apply.

**M0 — old Subject** (live before this change): "A quick check-in from Cape
Clarity"
**M0 — new Subject**: "A note about your consultation with Cape Clarity"

**M0 — old Body** (see §12 for the plain-text version this replaces).

**M0 — new Body** (verbatim, now live in "M0 - Lost Lead Feedback Token"'s "Send
email" step):

```html
<div class="cc-email-notification-container" style="background: #FFF; padding: 0; margin: 0; color: rgb(43, 43, 43); width: 100%; height: 100%; display: table; font-family: &quot;Open Sans&quot;, &quot;Trebuchet MS&quot;, sans-serif"><table align="center" width="600" style="text-align: center"><tbody><tr><td><table cellpadding="0" cellspacing="0" align="center" style="text-align: left; background: #FFF; margin-top: 50px; border-radius: 10px 10px 5px 5px; border: 1px solid rgb(224, 224, 224)"><tbody><tr><td><img alt="Cape Clarity Integrative Psychology" style="width: 100%; max-width: 600px; display: block; border: 0" width="600" src="https://cdn.prod.website-files.com/67dcadb2b4365cb9f4268b42/6a6a9e1731b99ce6392f9ecc_cape-clarity-email-header-band.png"></td></tr><tr><td><table class="inner-content" style="padding: 45px 55px"><tbody><tr><td><section><p style="margin:0px; line-height: 30px;"><span style="color:rgb(43, 43, 43)"><b><span style="font-size: 18px; margin: 0px; line-height: 30px;">Hi there,<br></span></b></span></p><div style="font-size: 15px; line-height: 28px; padding: 16px 0 0 0; color: rgb(43, 43, 43);">We're glad you reached out. Thank you for taking the time to talk with us about starting therapy. We'd love to hear what shaped your decision, whatever it was. It's quick and easy:<br></div><div style="font-size: 15px; line-height: 24px; padding: 10px 0 4px 0; word-break: break-all;"><a rel="noopener noreferrer" target="_blank" href="https://forms.zohopublic.com/lianapreudhommecapec1/form/M0FreeConsultNonConversionSurveyNEW/formperma/yoRiV9UKbs18x9p5nfIBADh_KlyZ2sriJF6ytCwk1XA?token=${issueFeedbackToken_1.token}" style="color: #175328; text-decoration: underline; font-weight: 600;">https://forms.zohopublic.com/lianapreudhommecapec1/form/M0FreeConsultNonConversionSurveyNEW/formperma/yoRiV9UKbs18x9p5nfIBADh_KlyZ2sriJF6ytCwk1XA?token=${issueFeedbackToken_1.token}</a></div><div style="font-size: 15px; line-height: 28px; padding: 16px 0 0 0; color: rgb(43, 43, 43);">Whatever comes next for you, we wish you well.<br></div><hr style="opacity: 0.3; margin: 35px 0px 35px 0px"><p style="margin: 10px 0px"><span style="color:rgb(43, 43, 43)"><span style="font-size: 15px; margin: 10px 0px;">Warmly,</span></span><br></p><p style="margin: 10px 0px"><span style="font-size: 15px; margin: 10px 0px;"><b style="color: rgb(43, 43, 43); font-weight: 600;">Cape Clarity Integrative Psychology</b></span><br></p></section></td></tr></tbody></table></td></tr></tbody></table></td></tr><tr><td><table align="center" style="margin-top: 30px; text-align: center"><tbody><tr><td valign="middle" style="display: block; text-align: center; padding: 0px 0 10px; font-size: 13px; line-height: 22px"><span style="color: rgb(119, 119, 119); font-weight: normal;">This email may contain information intended only for the person named above. If you received it in error, please let us know and delete it.</span><br></td></tr><tr><td valign="middle" style="display: block; text-align: center; padding: 0px 0 40px; font-size: 13px; line-height: 22px"><span style="color: rgb(119, 119, 119); font-weight: normal;">Cape Clarity Integrative Psychology</span><br></td></tr></tbody></table></td></tr></tbody></table></div>
```

Applied to the live flow via Zoho Flow's "Apply changes" on a Draft (M0 was
already Live/ON going into this change); the "Apply Changes?" confirmation dialog
showed exactly one item ("Configuration changed in Send email") before Apply was
clicked, confirming nothing else in the flow was touched.

## 15. Change (2026-09-28): survey simplified — "Okay to reach back out" and "Anything else on your mind" fields removed

Per Costin's direct instruction, the M0 survey was trimmed from five open
questions to three. No live form submissions had occurred yet (Costin
confirmed any test data so far was discardable), so this was done as a direct
edit to the live form/flow/Analytics objects rather than a staged migration.
Both M0 flows ("M0 - Lost Lead Feedback Token" and "M0 - Feedback Survey
Write-back") were confirmed OFF before this work and remain OFF afterward —
consistent with the project's "still in testing" phase and the standing "no
live/end-to-end tests without me" rule.

### 15.1 Form content changes

Made directly on the live "M0 - Free Consult Non-Conversion Survey" form
(the post-§13 object, slug `M0FreeConsultNonConversionSurveyNEW`):

- Form title changed to **"Your Feedback on Your Free Consultation."**
- The form's intro paragraph (Description field above the questions) was
  removed.
- The "Biggest factor in decision" question was reworded to reference "your
  free consultation" explicitly, rather than the more generic original
  phrasing.
- Field **`Radio1`** ("Would it be okay to reach back out to you in the
  future? Yes/Maybe/No") was **deleted** from the form.
- Field **`MultiLine1`** ("Is there anything else on your mind you'd like to
  share?") was **deleted** from the form.
- The thank-you page text had "and your story" removed from its closing
  sentence.

These six edits were made in the prior working segment (before the Analytics
cleanup below) and are recorded here together for a single point-in-time
reference. Costin noted the drag-and-drop gotcha where a newly added field can
land mid-canvas applies to intro/description-field placement, not to field
deletion, so it didn't come up in this change — no new field was added, only
removed. (It's relevant for M3 and M5, where Costin is placing new
Description/intro fields himself — see the M3 handoff note and M5's own
notes file.)

### 15.2 Write-back function: `submitFeedbackResponse` signature change

Edited live inside "M0 - Feedback Survey Write-back" (function opened via the
node's "⋮" menu → "View function" — not the pencil icon, which opens
parameter mapping, and not clicking the node title, which enters inline
rename mode).

**Old signature** (6 params):
```
map submitFeedbackResponse(string token, string ratingHeard, string reasonNotMovingForward, string addAnything, string reachBackOk, string anythingElse)
```

**New signature** (4 params):
```
map submitFeedbackResponse(string token, string ratingHeard, string reasonNotMovingForward, string addAnything)
```

The two `responseText` concatenation lines building the `reachBackOk` and
`anythingElse` segments were deleted outright, matching the precedent
established in `m4-implementation-notes.md` §10.2 for
`submitDischargeFeedbackResponse`. See §3.1 above for the full current
function source (now the authoritative capture, superseding the 2026-09-06
one) and §4 for the corrected `Response_Data` format string.

**Deliberate decision — trailing delimiter kept, not stripped.** The old blob
ended with `...Anything you would like to add: {value}---Okay to reach back
out: {value}---Anything else on your mind: {value}` (no trailing `---`, since
"Anything else..." was the true last field). After removing the last two
fields, `addAnything` becomes the new last field. Rather than stripping its
trailing `---` to match the old "no trailing delimiter on the last field"
pattern, the `---` was **left in place** — the new body ends
`...+ ifnull(addAnything,"") + "---"`. Reason: the "Additional Comments"
Analytics formula column (§6) is a `substring_between(..., 'Anything you
would like to add: ', '---', 1)` call that depends on finding a closing
`---` after the value. Stripping the trailing delimiter would have broken
that already-working formula and required editing it too, for no functional
benefit — a blob with a harmless trailing delimiter is simpler to reason
about than an exception to the "fields end in `---`" pattern. Worth
remembering for M5's equivalent field removal (Task list item 11): check
whether the same trade-off applies there before deciding whether to touch the
trailing delimiter.

**Flow builder gotcha reconfirmed**: saving this function triggered a
"Configuration changed" confirmation dialog that cited an unrelated flow
("TEST - Bookings to CRM PHI Field Mapping") — the same false-positive
"spurious dependency" pattern already documented in
`m4-implementation-notes.md` §10.2/§10.3. Confirmed through it; reopened the
function afterward and verified the 4-parameter signature persisted.

### 15.3 Analytics cleanup

All changes made in workspace `3251423000000083002`, in this order (order
matters — see the Unified Metrics discovery below):

1. Removed the **"M0 % Reachable" KPI panel** from the "M0 - Free Consult
   Non-Conversion Feedback" dashboard (Edit Design → hover panel → "⋮" →
   Remove → Save).
2. Removed the **"Anything Else" column** from the "M0 Submitted Responses"
   report (Edit Design → "X" on the column chip → regenerate tabular view →
   Save). "Okay to Reach Back Out" was never a column on this particular
   report, so no action was needed there.
3. Deleted the **"M0 % Reachable" report/view** itself (Explorer tile "⋮" →
   Delete). This succeeded cleanly since its dashboard panel was already gone.
4. Attempted to delete the **"Okay to Reach Back Out"** formula column from
   the base "Milestone Instances" table — **blocked**: "Column 'Okay to
   Reach Back Out' cannot be deleted due to the following reason: Used in
   the formula column '% Reachable' in the view 'Milestone Instances'."
   Deleting the report in step 3 did **not** resolve this — the identical
   error recurred on retry.
5. **Root-cause finding (new gotcha, worth flagging for M2/M3/M4/M5's own
   Analytics work if a similar block ever recurs there):** "% Reachable" is
   not the report deleted in step 3 — it's a separate, workspace-level
   **Aggregate Formula** ("Metric") object. This object type is invisible to
   the normal column-management surfaces: the base table's "Show/Hide/
   Reorder Column(s)" dialog returned "No matches found" for "Reachable,"
   and the column's own Dependency Details panel ("Child Views") also
   returned "No view found" for "Reachable." The only place it's visible or
   manageable is the **"Unified Metrics"** view, found under the Data
   sidebar's tile labeled "An overview of all the metrics defined in your
   workspace." Deleted the Aggregate Formula named "% Reachable" from there
   (hover row → trash icon → confirm). Its formula, for the record:
   ```
   count_if("Milestone Instances"."Okay to Reach Back Out" = 'Yes' OR "Milestone Instances"."Okay to Reach Back Out" = 'Maybe')*100/count_if("Milestone Instances"."Status" = 'Submitted')
   ```
6. Retried deleting the **"Okay to Reach Back Out"** formula column from the
   base table — succeeded, now that the Aggregate Formula was gone.
7. Deleted the **"Anything Else"** formula column from the base table the
   same way — succeeded on the first attempt (it had no Aggregate Formula
   dependency).
8. Reloaded the full "M0 - Free Consult Non-Conversion Feedback" dashboard
   end-to-end and confirmed all 6 remaining panels render with no errors:
   "M0 Status Breakdown," "M0 Response Rate," "M0 Submitted Responses" (now
   4 columns), "M0 Volume by Week," "M0 Biggest Factor," "M0 Feeling Heard
   Distribution."

**General lesson for future Analytics cleanup on this project**: if a
column-delete attempt is blocked citing "Used in the formula column 'X' in
the view 'Y'" and neither a normal report/view search nor the column's
Dependency Details panel can find that reference, check "Unified Metrics"
next — it's a real, separate object type (Aggregate Formula / Metric), not a
bug in the error message.

### 15.4 Net result

- M0's survey now asks 3 open questions (Feeling Heard, Biggest Factor,
  Additional Comments) instead of 5.
- `submitFeedbackResponse` takes 4 params instead of 6.
- `Response_Data` is 3 fields instead of 5 (see corrected §4).
- Analytics has 3 live formula columns on Milestone Instances for M0 instead
  of 5 (see corrected §6); the dashboard has 6 panels instead of 7 (see
  corrected §7).
- Both M0 flows remain OFF, as they were going into this change.
- No CRM schema change, no changes to M1-M5, and no changes to the
  2026-09-28 email redesign documented in §14 (that work and this work were
  independent changes made in the same session).

## 16. Change (2026-09-30): trigger narrowed to Lost Leads who had a free consult

Per Costin's decision: M0's survey asks about the free consultation, so it should go
only to leads who actually had one. Before this change, "M0 - Lost Lead Feedback Token"
fired for every Lead whose `Lead_Status` became `Lost Lead`, including leads who were
lost before any consult happened. The CRM signal for "a free consult happened" is the
Lead's **Consult Call Date** field: populated means a consult took place, empty means
it did not.

### 16.1 Field

Confirmed via CRM `getFields` on `Leads` (schema only; no Lead records were read or
opened): API name `Consult_Call_Date`, label "Consult Call Date", `data_type` `date`,
custom field, not system-mandatory (so empty is a normal, expected state).

### 16.2 As-built trigger filter

Edited in Zoho Flow on "M0 - Lost Lead Feedback Token", on the trigger node ("Updated
module entry", Zoho CRM, module `Leads`) under **Filter criteria**. It was one row; it
is now three rows joined with AND:

| # | Field | Operator | Value |
|---|---|---|---|
| 1 | Lead Status | equals | Lost Lead (unchanged) |
| 2 | Consult Call Date | is not null | (none) |
| 3 | Consult Call Date | is not empty | (none) |

Why two rows for one idea: Zoho Flow's operator list for this field offers `is blank`,
`is empty`, `is not empty`, `is null`, `is not null`. It was not verified (doing so
would have meant opening a Lead or running a live test, both off-limits) whether an
unset CRM date reaches Flow as null or as an empty string. A populated date satisfies
both rows; an unset date fails at least one under either representation. Row 3 or row
2 can be dropped later if a coordinated test shows which representation Flow actually
uses; keeping both is harmless.

This was done as a trigger filter, not an If-else node, so non-qualifying leads show
as **Filtered** in execution history (same as non-Lost-Lead updates already do) rather
than as completed runs with a dead branch. Nothing else in the flow changed:
`issueFeedbackToken` parameters, the Send email step, the On Error branches, the email
copy (§14) and the form link (§13) are all untouched.

State at the end of this session: the change is saved in the flow's builder (verified
by reloading the builder and reopening the trigger: all rows persisted), the flow
shows a **Draft** badge, and the flow is still **OFF**. Not run, not live-tested (per
the standing "no live/end-to-end tests without Costin" rule). When M0 is next switched
ON for the coordinated test, check that the builder shows no pending Draft/"Apply
changes" prompt, and apply it if it does, so the live version carries the new filter.

### 16.3 Behavior and edge cases

- **Lost, no consult**: filtered out, no `Milestone_Instances` record, no email.
- **Lost, consult held**: unchanged from before (token issued, email sent).
- **Consult date added later**: "Updated module entry" re-evaluates on every update to
  a Lead. If a Lead is set to Lost Lead while Consult Call Date is empty (filtered),
  and someone later fills in Consult Call Date while the status is still Lost Lead,
  that update now passes the filter and issues a token. This is the correct outcome
  (a consult did happen), but it means the survey can go out at the moment the date is
  back-filled rather than the moment the lead was marked lost.
- **Consult date present but later edited, status still Lost Lead**: any further
  update to that Lead re-fires the flow. Same as before this change, `issueFeedbackToken`
  supersedes any earlier Issued record, so the recipient only ever has one live link.
- **Consult Call Date means "had a consult", not "attended"**: the filter keys purely
  off the field being populated. If the practice ever fills it in for a scheduled but
  not-yet-held or no-show consult, those leads would qualify. Worth keeping in mind for
  how the team uses the field.
  **Accepted risk (Costin, 2026-10-01):** the team fills Consult Call Date *before* the
  consult is held, so some leads (e.g. a consult that is scheduled but no-shows) can receive
  the survey without having had one. Costin is fine with this.

### 16.4 Also updated in this session

- `spec.md`: Milestones table row 0 and a sixth-pass revision note.
- `coordinated-live-test-plan.md` Step A: a prerequisite (test Leads need a Consult
  Call Date) and a new Scenario 7 (Lost Lead with no consult date: expect Filtered,
  nothing created).
- `CLAUDE.md`: no change needed (its M0 references are about the cutoff gate, which M0
  still does not use).

## 17. Change (2026-10-02): opt-out gate added to "M0 - Lost Lead Feedback Token" (feature 006, T022)

Edited in the Zoho Flow builder; the flow stayed OFF throughout (status "Draft",
as before). Unlike M2-M5, this flow had no If-else, so the gate was inserted
between the trigger and `issueFeedbackToken`.

Structure after the edit: Updated module entry (Leads) -> **skipIfOptedOut** ->
**If else (`skipIfOptedOut_1` is false)** -> issueFeedbackToken -> Send email.
The existing On Error branches from feature 003 are untouched, and the trigger
filter from section 16 is unchanged. skipIfOptedOut inputs: milestone literal
`0 - No Conversion` (same literal issueFeedbackToken gets), patientId blank,
leadId = Updated module entry -> Entry ID, clinician = literal `Liana
Preudhomme` (the same literal issueFeedbackToken uses). Its On Error branch goes
to a new Zoho Mail alert (cloned from the existing alert, variable
`alertOptOutCheckFailed`, subject "Cape Clarity feedback pipeline: opt-out check
failed (0 - No Conversion)"). The clone's body was fully rewritten: it names only
the flow and milestone, with no prospect identity.

The check reads the native `Email_Opt_Out` field on the Lead. When opted out,
skipIfOptedOut writes a skip row to Milestone_Instances and the If-else's True
branch is not taken, so no token is issued and no email is sent.

Builder note: the Builder page can end up with its outer containers scrolled
(canvas coordinates then no longer match screenshots); resetting `scrollTop` on
`body`, `.singleFlow` and `.liquid-child` restores them.
