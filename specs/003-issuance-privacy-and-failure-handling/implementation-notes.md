# 003 - Issuance Privacy and Send-Failure Handling — Implementation Notes

> As-built, 2026-09-23. All flows still **OFF**; structural verification only.
> **One step is left for Costin: add the "Send Failed" Status value in CRM
> (§3).** Until then, the failure branches will try to write a value the Status
> picklist doesn't list.

## 1. Privacy (1B) — done

- **`issueFeedbackToken`** (shared by M0/M2/M3): Name line changed to
  `createMap.put("Name","Milestone " + milestone + " - " + token.subString(0,8));`.
  Saved with "We have successfully updated the function"; re-read from the editor.
  Everything else in the function is unchanged (source otherwise as in
  `specs/002-remove-subflow-dependency/implementation-notes.md` §3).
- **Existing records renamed** via CRM MCP `updateRecords` (all 3 returned
  SUCCESS): `6825601000004191001` → `Milestone 0 - No Conversion - 91a38157`;
  `6825601000004245001` → `Milestone 1 - Baseline Intake - d196ca74`;
  `6825601000004284001` → `Milestone 2 - Early Alliance Check - 0045fe45`.
  Re-query `select id from Milestone_Instances where Name like '%@%'` → **0 rows**.
- **Analytics sync** (viewed, cancelled, nothing saved): Milestone Instances →
  Email and Secondary Email already unselected; "Milestone Instance Name" is
  selected and greyed out (can't be deselected). No change needed. The Analytics
  copy of the renamed records updates at the next daily sync (17:00 ET).

## 2. Failure paths (2B) — built in M0, M2, M3

Each trigger flow now has, in addition to 002's chain:

| Branch | Nodes | Key values |
|---|---|---|
| On Error of `issueFeedbackToken` | Zoho Mail "Send email", variable `alertIssuanceFailed` | connection "Connection to info@capeclarity.com"; From info@capeclarity.com; To **costin@capeclarity.com**; Subject `Cape Clarity feedback pipeline: token issuance failed (<milestone>)`; body per data-model.md |
| On Error of the patient "Send email" | Zoho CRM "Update module entry", variable `markSendFailed` → Zoho Mail "Send email", variable `alertSendFailed` | CRM Connection; module Milestone Instances; layout Standard; Entry Id `${issueFeedbackToken_1.recordId}`; Status "Use a Custom Value" = `Send Failed`; nothing else set. Alert: same connection/From/To; Subject `Cape Clarity feedback pipeline: email send failed (<milestone>)`; body includes `${issueFeedbackToken_1.recordId}` |

Verified after a full reload in each flow: every new node's settings read back
(for `markSendFailed`, the panel needs up to ~30 s to load the CRM layout before
Entry Id/Status show; a first read that looked empty in M2 was just that);
endpoints `jsplumb-connected` as expected; the canvas shows the red "On Error"
labels on both branches. Alert bodies contain no `${trigger.*}` fields, so no
name/email/phone reaches the alert.

Flow "Last saved" times after the build: M3 12:02, M2 12:09, M0 12:16 (ET).

## 3. Left for Costin: "Send Failed" picklist value (T001)

The CRM MCP has no field-update tool, and CRM Setup pages wouldn't finish loading
in the automated browser this session (stuck on "Loading" across retries and a
fresh tab; Zoho Flow and Analytics loaded fine). **To do**: Zoho CRM → Setup →
Modules and Fields → Milestone Instances → Standard layout → Status field → add
the value **Send Failed** exactly (capital S, capital F). Takes a minute; then
`getFields` should list it.

## 4. Builder gotchas learned (add to 002 §5)

1. **Drop zones are explicit while dragging.** During a palette drag the canvas
   shows "DROP HERE" (normal next step) and red "ON ERROR / DROP HERE" boxes for
   each node's error branch. Hovering a real mouse over the exact box and
   clicking drops there. This is the reliable way to target an On Error branch;
   synthetic mouseup at a coordinate snapped to whatever the builder judged
   closest (usually the normal next step).
2. The JS stepped drag often leaves the drag "in flight" (ghost card following
   nothing); finishing it with real `hover` moves and a click on the drop box
   works, and is what made the error-branch drops land correctly.
3. Before that was found, a placeholder trick also worked: occupy the node's
   normal out with a throwaway node, then the next drop near it goes to the
   On Error branch; delete the placeholder after.
4. Context-menu "Delete" is `button.zf-btn-delete` inside the menu; when the menu
   opens below the viewport, click it via JS.
5. The "function is used in the following flows" dialog shows wrong names (this
   time it listed "[RETIRED] Subflow - Issue Feedback Token", which verifiably
   doesn't use the function; its summary lists only generateFeedbackToken and
   supersedeOpenInstances). Count is right; names aren't.
6. Zoom out (−) to 60% before adding branches; the canvas doesn't scroll with the
   mouse wheel.
