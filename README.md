# ParentConnect AI — O-Grade Production Delivery

This package is a full production refactor of the uploaded
`ParentConnectAI-EnhancedOG1.zip`, delivered as 5 modules.

```
dialogflow_agent/    The refactored, importable Dialogflow ES agent
                     (intents, entities, agent.json) -- ZIP this folder's
                     contents to re-import into Dialogflow ES console:
                     Settings (gear icon) -> Export and Import -> Restore.

webhook/             Complete, tested Node.js/Express fulfillment backend
                     for AttendanceIntent, FeePaymentStatusIntent,
                     ChangeChildIntent, LeaveApplicationIntent,
                     ReportCardIntent, and the Stage-3 fallback escalation.
                     See webhook/README.md to deploy and activate.

docs/
  01_NLU_Disambiguation_Matrix.md        Module 1
  02_Context_State_Engine.md             Module 2
  03_Fallback_UX_Design.md               Module 3
  (Module 4 is the /webhook folder itself)
  04_OGrade_Rubric_Audit_Report.md       Module 5

BUGFIX_REPORT.md     Round-2 IBCP CRS audit report (parameter standardisation,
                     dynamic attendance calculation, context lifecycle fix,
                     training-phrase annotation, mock DB integrity, payload
                     fallbacks). See dialogflow_agent/BUGFIX_REPORT.md for
                     the original bug-fix history, preserved for continuity.
```

## Recommended review order

1. Read `docs/04_OGrade_Rubric_Audit_Report.md` first — it maps every change
   in this delivery to the original bug report and to grading criteria, and
   is the fastest way to see what changed and why.
2. Re-import `dialogflow_agent/` into a Dialogflow ES console to test the
   NLU and fallback improvements interactively.
3. Follow `webhook/README.md` to deploy the backend and flip
   `webhook.available` to `true`, then re-test the 5 fulfillment intents
   with live (mock-DB-backed) data instead of static text.

## What changed at a glance

- **628** training phrases across **33** intents (up from ~460), with 0
  cross-intent duplicate phrases and 0 unresolved chip references, both
  verified programmatically against the final delivered files.
- A real session-state engine (`active_child_state`) that is advisory-only
  and never silently overrides what the user just said — directly
  addressing the root cause in `BUGFIX_REPORT.md`.
- A working 3-stage progressive fallback (clarify → categorize → escalate),
  replacing the flat single-message fallback.
- A complete, live-tested Node.js/Express webhook backend with mock-DB-backed
  fulfillment for the 5 highest-complexity intents, auth, validation, and
  defensive error handling throughout.
