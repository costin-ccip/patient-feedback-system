# M0 (Free Consult / No-Conversion) — Implementation Notes

> Scope: this file documents ONLY the M0 milestone (the free-consult, no-conversion
> feedback loop). It is a companion to `specs/001-feedback-collection-pipeline/spec.md`,
> which covers all six milestones at the requirements level. This file exists to let a
> future session (human or AI) understand exactly what was built, why, and where the
> sharp edges are, without having to reverse-engineer it from Zoho.
>
> Last updated: 2026-09-06
> Status: M0 is live and functional end-to-end (form → CRM write-back → Analytics
> reporting). See "Process note" at the end — this milestone was built ad hoc,
> outside the spec-kit plan/tasks workflow the project constitution calls for.

## 1. What M0 does

M0 fires when a lead has a free consultation and does not convert to a paying client.
The clinician (Liana) triggers issuance of a feedback survey token for that lead. The
lead receives a link to a public Zoho Form. When they submit it, a Zoho Flow captures
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
| Token issuance | "Subflow - Issue Feedback Token" | Zoho Flow, folder "Customer Feedback System" |
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
   | `reachBackOk` | `${trigger.Radio1}` (Okay to Reach Back Out: Yes/Maybe/No) |
   | `anythingElse` | `${trigger.MultiLine1}` (Anything Else) |

### 3.1 `submitFeedbackResponse` — verbatim Deluge source

Captured directly from the Flow builder on 2026-09-06. This is the authoritative
source; if you change the function, update this block too.

```
map submitFeedbackResponse(string token, string ratingHeard, string reasonNotMovingForward, string addAnything, string reachBackOk, string anythingElse)
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
	responseText = responseText + "Okay to reach back out: " + ifnull(reachBackOk,"") + "---";
	responseText = responseText + "Anything else on your mind: " + ifnull(anythingElse,"");
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

`Milestone_Instances.Response_Data` is a single text field holding all five open
answers concatenated with a `---` (triple-hyphen) delimiter, in this fixed order:

```
How much did you feel heard and understood (1-5): {value}---Biggest factor in decision: {value}---Anything you would like to add: {value}---Okay to reach back out: {value}---Anything else on your mind: {value}
```

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

**Okay to Reach Back Out**
```
substring_between("Milestone Instances"."Response Data", 'Okay to reach back out: ', '---', 1)
```

**Anything Else** (last field — no trailing `---` to bound it, so this one can't use `substring_between`)
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

Workspace: `3251423000000083002`.

| Report | Type | Purpose |
|---|---|---|
| M0 Status Breakdown | (pre-existing) | Issued/Submitted/Expired counts |
| M0 Response Rate | (pre-existing) | Submitted ÷ Issued |
| M0 Volume by Week | (pre-existing) | Issuance trend |
| M0 Submitted Responses | Tabular | Submitted Date, Lead Reference, Biggest Factor, Additional Comments, Anything Else. Raw `Response Data` blob was removed from this view once the parsed columns existed — no reason to show clinicians/ops the raw delimited string. |
| M0 Biggest Factor | Chart (bar) | X = Biggest Factor, Y = Count of Id |
| M0 Feeling Heard Distribution | Chart (bar), view `3251423000000083576` | X = Feeling Heard Score, Y = Count of Id |
| M0 % Reachable | Summary/KPI, view `3251423000000083587` | Aggregate formula: `count_if("Milestone Instances"."Okay to Reach Back Out"='Yes' OR "Milestone Instances"."Okay to Reach Back Out"='Maybe')*100/count_if("Milestone Instances"."Status"='Submitted')` |

Dashboard **"M0 - Free Consult Non-Conversion Feedback"** (view `3251423000000083524`)
currently has 7 panels: the three pre-existing status/rate/volume panels, the updated
Submitted Responses table, and the three new panels above.

**UX rationale for the three newer panels** (for context if revisiting the design):
- Feeling Heard Score (1-5 rating) → distribution bar chart, because a single average
  hides whether the practice is polarizing (lots of 1s and 5s) vs. consistently
  mediocre (lots of 3s) — different variance profiles need different action.
- Okay to Reach Back Out (Yes/Maybe/No) → a single "% Reachable" KPI tile
  (Yes + Maybe, over Submitted), because the actionable question is "how many of
  these people could we still follow up with," not the raw 3-way split.
- Additional Comments / Anything Else (open text) → parsed into real columns on the
  existing tabular report rather than a new visualization, since open text doesn't
  aggregate; the win was just making it readable instead of buried in a blob.

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
