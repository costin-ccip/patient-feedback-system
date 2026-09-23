# Research: Remove Subflow Dependency from Token Issuance

**Feature**: `002-remove-subflow-dependency` | **Date**: 2026-09-23

## 0. What the subflow actually does (captured live, 2026-09-23)

No document in the repo recorded the subflow's internals before this feature; 001's
notes only list the parameters each caller passes. Captured read-only from the Zoho
Flow builder (`flows/subflow_issue_feedback_token/edit`, flow ID `872426000000002154`),
no execution history opened (PII rule).

**Flow description (as saved in Flow)**: "Shared logic called by the M0/M4/M5 trigger
flows: generates a random URL-safe token, creates a Milestone Instance record
(Issued…" (truncated in the UI). The "M0/M4/M5" wording predates M2/M3 and is stale;
M0, M2, M3 (and retired M1) are the real callers.

**Subflow inputs** (`zf_trigger.*`): `milestone`, `patient_id`, `lead_id`,
`recipient_email`, `clinician`, `survey_url`, `email_subject`, `email_intro_text`,
`ttl_days`.

**Steps, in order:**

1. **Custom function `supersedeOpenInstances(milestone, patientId, leadId)`** →
   output `supersedeOpenInstances_4`. Verbatim:

   ```text
   string supersedeOpenInstances(string milestone, string patientId, string leadId)
   {
   if(patientId != null && patientId != "")
   {
   	criteria = "((Milestone:equals:" + milestone + ") and (Status:equals:Issued) and (Patient:equals:" + patientId + "))";
   }
   else
   {
   	criteria = "((Milestone:equals:" + milestone + ") and (Status:equals:Issued) and (Lead_Reference:equals:" + leadId + "))";
   }
   response = zoho.crm.searchRecords("Milestone_Instances",criteria,1,200,null,"crm_connection");
   count = 0;
   for each  rec in response
   {
   	updateMap = Map();
   	updateMap.put("Status","Superseded");
   	updateResp = zoho.crm.updateRecord("Milestone_Instances",rec.get("id"),updateMap,Map(),"crm_connection");
   	count = count + 1;
   }
   return "superseded:" + count;
   }
   ```

2. **Custom function `generateFeedbackToken(ttlDays)`** → output
   `generateFeedbackToken_1`. Verbatim:

   ```text
   map generateFeedbackToken(int ttlDays)
   {
   seed = zoho.currenttime.toString() + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999) + "|" + randomnumber(100000000,999999999);
   token = zoho.encryption.sha256(seed);
   resp = Map();
   resp.put("token",token);
   expiry = zoho.currenttime.addDay(ttlDays).toString("yyyy-MM-dd'T'HH:mm:ssXXX");
   resp.put("expiry",expiry);
   return resp;
   }
   ```

3. **Zoho CRM "Create module entry"** (connection "CRM Connection", module
   Milestone Instances, layout Standard, variable `createModuleEntry_2`):

   | CRM field | Value |
   |---|---|
   | Name | `Milestone ${zf_trigger.milestone} - ${zf_trigger.recipient_email}` |
   | Milestone | `${zf_trigger.milestone}` (custom value) |
   | Email | `${zf_trigger.recipient_email}` |
   | Expiry_Date_Time | `${generateFeedbackToken_1.expiry}` |
   | Patient | `${zf_trigger.patient_id}` |
   | Token | `${generateFeedbackToken_1.token}` |
   | Lead_Reference | `${zf_trigger.lead_id}` |
   | Status | `Issued` |
   | Clinician | `${zf_trigger.clinician}` (custom value) |
   | Trigger for | "All the above" (workflow, approval, blueprint) |
   | All others (Owner, Secondary Email, Submitted_Date_Time, Response_Data, Email Opt Out, Tag, Unsubscribed *) | empty |

4. **Zoho Mail "Send email"** (connection "Connection to info@capeclarity.com",
   variable `sendEmail_3`): From `info@capeclarity.com`; To
   `${zf_trigger.recipient_email}`; Reply-to/CC/BCC empty; Subject
   `${zf_trigger.email_subject}`; HTML body verbatim:

   ```html
   <div>Hi,<br></div><div><br></div><div>${zf_trigger.email_intro_text}<br></div><div><br></div><div>${zf_trigger.survey_url}?token=${generateFeedbackToken_1.token}<br></div><div><br></div><div>Thank you,<br></div><div>Cape Clarity<br></div>
   ```

**Current callers ("Call a subflow", "Wait and continue")**:

| Flow | State | Parameters (non-default) |
|---|---|---|
| M0 - Lost Lead Feedback Token | OFF | milestone `0 - No Conversion`; lead_id `${trigger.id}`; patient_id empty; recipient_email `${trigger.Email}`; clinician `Liana Preudhomme`; ttl_days `7`; survey_url `https://forms.zohopublic.com/lianapreudhommecapec1/form/M0FreeConsultNonConversionSurvey/formperma/GFdd7kA1Yp8jIxhsMuTJTN1QDEiSl5K6FoQjceHsnAg`; email_subject "A quick check-in from Cape Clarity"; email_intro_text "We'd love to hear how things are going. Please take a moment to share your feedback using the link below:" (captured live this session; not previously documented) |
| M2 - Session 3 Trigger | OFF | per 001 `m2-implementation-notes.md` §3.3 |
| M3 - Periodic Check-In Trigger | OFF | per 001 `m3-implementation-notes.md` §1.5 |
| [RETIRED] M1 - Session 1 Trigger | OFF | per 001 `m1-implementation-notes.md` §4; out of scope (retired) |

M0's trigger is just "Updated module entry (Leads) → Call a subflow"; it has no
idempotency check of its own, so its only duplicate protection is step 1 above.

## 1. Plan availability

- **Decision**: Design for Zoho Flow Standard with no subflows, custom functions
  available.
- **Rationale**: Confirmed 2026-09-23 on the Zoho Store subscription page (Standard
  Plan, 5,000 tasks, annual). Costin states subflows aren't available on it. On first
  load this session, Flow still showed a stale "You've been moved to the Free Plan"
  banner (Free has no custom functions and a 5-live-flow cap); it did not reappear
  after reload once the Standard subscription was in place.
- **Note**: Zoho's public pricing page lists subflows under Standard. This doesn't
  change the design (Costin's instruction, and not depending on an ambiguous feature
  is sturdier), but if subflows turn out to work after all, nothing here needs undoing.
- **Alternatives considered**: Free-plan redesign (move logic into CRM workflow
  rules). Not needed now; would be required if custom functions are ever lost.

## 2. What replaces the subflow

- **Decision**: One new workspace-level custom function, `issueFeedbackToken`, that
  does steps 1-3 (supersede, token/expiry, create record) and returns the token,
  expiry, and record ID. Each trigger flow calls it, then runs its own native Zoho
  Mail "Send email" step.
- **Rationale**:
  - Zoho Flow custom functions are shared across flows (001 `m2-implementation-notes.md`
    §3.2 found this as a hazard). Used on purpose, it gives exactly the reuse the
    subflow gave: one definition, edited once, Flow warns "used in the following
    flows" on save. This satisfies spec FR-003.
  - Keeping the email as a native Zoho Mail step (not Deluge `sendmail`) keeps the
    same connection and sender (`info@capeclarity.com`). Deluge `sendmail` in Flow
    sends through Zoho's own notification path, which would be a new channel needing
    its own Principle V / Mail-safeguard review (spec FR-005).
  - 2 nodes per caller (function + email) instead of 4. Node placement and wiring are
    the most failure-prone part of building in this Flow builder (001 m1/m2/m3 notes),
    so fewer nodes = less risk.
- **Alternatives considered**:
  - *Inline the subflow's 4 steps into every caller, reusing the two existing
    functions*: record-field mapping would then exist 3 times (5 with M4/M5), the
    drift problem FR-003 exists to prevent. 12 nodes to place and wire vs 6. Rejected.
  - *Fold the email into the function too (Deluge sendmail)*: new sending channel,
    rejected per above.
  - *Webhook flow as a pseudo-subflow* (callers POST to a webhook-triggered "issue
    token" flow): recreates a subflow, but pushes the recipient email and patient ID
    over an HTTP call and depends on Flow's inbound webhook trigger, which Free lists
    as a paid feature and adds a public endpoint. Rejected: more moving parts and a
    new PHI-in-transit surface for no gain.
  - *Modify `generateFeedbackToken` to also create the record*: it's still wired into
    the subflow and into retired M1; editing a shared function in place is exactly the
    §3.2 hazard. A new function avoids touching anything live. Rejected.

## 3. Failure handling (email must not go out without a record)

- **Decision**: `issueFeedbackToken` checks the create response and, if no record ID
  came back, stops the flow with a Deluge `throw`, so the email step never runs. If
  the Flow Deluge editor rejects `throw` on save, fall back to an If-else after the
  function on `issueFeedbackToken_1.status == "success"` (one extra node).
- **Rationale**: spec FR-004 and the edge case "never email a token with no Issued
  record". Today's subflow gets this for free (a failed native CRM step halts the
  flow). A Deluge function that just returns an error map would NOT halt it, so this
  needs an explicit mechanism.
- **Open to verify at build time**: that `throw` saves and that a thrown error marks
  the step failed and halts later steps. Structural check only (save succeeds); the
  halting behavior is confirmed during Costin's live test.

## 4. Preserving the CRM create exactly

- **Decision**: `zoho.crm.createRecord` with an options map
  `{"trigger": ["workflow","approval","blueprint"]}` to match the native step's
  "Trigger for: All the above"; put `Patient` / `Lead_Reference` only when non-empty.
- **Rationale**: The native step mapped an empty `patient_id` (M0) or empty `lead_id`
  (M2/M3) and Flow dropped the blank. In Deluge, putting an empty string on a lookup
  can fail the create, so blanks are skipped instead. Same resulting record.
- **Open to verify**: the exact option-map shape the Flow Deluge editor accepts. If
  the 4-argument form with options doesn't save, use the documented
  `zoho.crm.createRecord(module, map, options, connection)` variant the editor
  suggests and record what worked.

## 5. Things found that are out of scope, flagged for Costin

1. **Recipient email is written into the record Name** (`Milestone <m> - <email>`) and
   into `Email` on Milestone_Instances. CRM holding identity is allowed (Principle II),
   but this module is synced to Zoho Analytics ("Milestone Instances" table), so the
   Analytics copy holds the email too. No report shows it today, but it's a wider PHI
   footprint than 001's "only tokens in Analytics" framing implies. Preserved as-is
   here (FR-002, like-for-like); worth a separate decision.
2. **Record created, email failed** leaves an orphan Issued record. Same as today;
   it expires after TTL. Preserved.
3. **Opening a node's config panel and cancelling bumps the flow's "Last saved"
   time** (seen this session on the subflow and M0 trigger; no values changed). A
   builder quirk, not a change, but it means "Last saved" can't be trusted as a
   "was this edited" signal.
