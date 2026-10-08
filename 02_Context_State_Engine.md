# Module 2 — Multi-Turn Context & Session State Engine

## 2.1 Why the original context model had to be rebuilt, not just re-enabled

`BUGFIX_REPORT.md` correctly diagnosed and removed a dangerous pattern: 18
intents silently reused a remembered grade/child from a Dialogflow context
whenever the current utterance's parameter was ambiguous, with zero
indication to the user that a substitution had happened. The fix was to
delete that behavior entirely — which was the right emergency fix, but it
left the agent with **no session memory at all**, which is itself a
enterprise-UX defect: a parent who says "Switch to Aarav" and then "check
attendance" should not have to restate the child's name.

The redesign below restores state *safely*: session memory is now
**advisory, opt-in, and always visible to the user** — never a silent
override of what they just typed.

## 2.2 Context topology

| Context name | Set by | Lifespan | Read by | Purpose |
|---|---|---:|---|---|
| `active_child_state` | Any of 19 child/grade-bearing intents on successful resolution (Attendance, FeeDetails, FeePaymentStatus, ExamSchedule, ExamResult, Homework, Syllabus, ReportCard, LeaveApplication, HealthMedical, TeacherContact, EventInfo, Extracurricular, UniformInfo, Transport, ParentTeacherMeeting, SchoolTimings) and `ChangeChildIntent` | 5 turns (10 for explicit `ChangeChildIntent`) | Webhook handlers, to build a **suggestion chip**, never to auto-fill a required Dialogflow parameter | Lets a parent ask several follow-ups about the same child without repeating the name |
| `human_escalation` | `TalkToHumanIntent`, `ComplaintFeedbackIntent`, `HealthMedicalIntent`, urgent-reason `LeaveApplicationIntent`, and Fallback Stage 3 | 1 turn | Webhook / downstream ticketing | Marks that this turn produced a human-facing ticket, for analytics and to avoid double-ticketing on the same turn |
| `fallback_stage_1` / `fallback_stage_2` | Default Fallback Intent / Stage-2 fallback intent | 1 turn each | The next fallback intent in the chain (input context) | Drives the 3-tier escalating fallback (Module 3) — decrements automatically each turn, so a successful intent match in between resets the chain for free |

**Design rule enforced everywhere:** *no* required Dialogflow parameter is
ever populated from a context. `active_child_state` is deliberately never
listed in any intent's top-level `contexts` (input-context precondition)
field — doing so would make that intent only triggerable while the context is
active, which is not the goal and was not how the original bug manifested.
Instead, context data flows exclusively through `queryResult.outputContexts`
into the webhook, which decides, per turn, whether to *offer* it as a chip:

```
"Here's the overall attendance for Grade 8: the class average is 91%..."
[chip: "Did you mean Aarav? Tap to confirm, or type a different name."]
```

versus the old failure mode, which would have silently answered as if the
user had said "Aarav" without ever asking.

## 2.3 `ChangeChildIntent` — before/after

**Before:** zero parameters, zero contexts, a single static apology
("go ahead and ask your question again..."). It could not actually change
anything — the name was aspirational, not functional.

**After:**
- Captures `child_name` (required) and `grade` (optional) as real slots with
  follow-up prompts.
- Looks the child up via the webhook (`handlers/changeChild.js`), confirms
  a match against the roster, and only then sets `active_child_state` with a
  10-turn lifespan (longer than the default 5, since this is an *explicit*
  user action and should persist through a longer sub-conversation).
- Responds with an honest, verifiable confirmation: *"Got it — I've switched
  to Diya (Grade 6). What would you like to know?"* — matching what the
  system actually does, closing the "I'll remember this" honesty gap flagged
  in `BUGFIX_REPORT.md`.
- Offers immediate next-step chips (Check Attendance / Check Homework / Fee
  Payment Status) so the state change has an obvious, discoverable payoff.

## 2.4 Digression handling — "what if they ask about the cafeteria mid-flow?"

Dialogflow ES intent matching is inherently digression-safe *by construction*
here, because none of the informational intents (`CafeteriaMenuIntent`,
`WeatherClosureIntent`, `SchoolTimingsIntent`, `HolidayListIntent`,
`LibraryInfoIntent`, etc.) declare any required input context. Concretely:

1. Parent is mid slot-fill for `AttendanceIntent` (bot has just asked "Which
   grade or child?").
2. Parent instead types "What's on the cafeteria menu today?"
3. Because `CafeteriaMenuIntent` has no context precondition and its training
   phrases are a strong match, Dialogflow's slot-filling is interrupted and
   `CafeteriaMenuIntent` fires normally, answering the menu question.
4. Dialogflow ES does **not** natively resume the original slot-fill after an
   interruption — this is a platform limitation, not something patchable at
   the agent-JSON layer. The production-correct mitigation, implemented at
   the webhook/client layer, is:
   - The original intent's prompt is re-issued as a **quick-reply chip** on
     the digression's response: *"By the way, you were asking about
     attendance — want to continue? [Yes, continue] [No, that's all]"*.
   - This requires the client (web widget / WhatsApp integration) to track
     "last incomplete intent" client-side and re-inject it as the chip
     payload, since Dialogflow ES's own context stack is consumed by the
     digression. This hook point is documented in `webhook/README.md` under
     **"Resuming interrupted slot-fills"** with the exact payload shape the
     client should send back (`event: "resume_intent"` with the original
     intent name and partially-filled parameters carried in
     `active_child_state`/a dedicated `pending_intent` context we reserve for
     this — see below).
   - **Recommended next-phase extension** (not yet wired into the shipped
     agent, flagged here rather than silently omitted): add a
     `pending_intent_state` context, lifespan 2, populated by Dialogflow's
     own slot-filling event whenever `allRequiredParamsPresent: false`. A
     parent who says "actually go back to attendance" would then be met with
     the parameters already captured rather than starting over. This needs a
     small webhook hook (`webhookForSlotFilling: true`) on each of the 17
     state-refresh intents, which was deliberately left out of this release
     to keep webhook-triggered slot-filling latency off the critical path for
     high-frequency intents — recommended as a v1.1 enhancement once the
     core 5 fulfillment intents are validated in production.

## 2.5 Human-escalation triggers — explicit, not inferred silently

Three input paths converge on `human_escalation`:

1. **Direct request** — `TalkToHumanIntent` (any phrasing asking for a
   person, principal, or office).
2. **Formal complaint** — `ComplaintFeedbackIntent`, always escalated (a
   complaint should never be closed by a bot reply alone).
3. **Medical/urgency signal** — `HealthMedicalIntent` always escalates,
   and `LeaveApplicationIntent` escalates conditionally when the stated
   `leave_reason` matches an urgency keyword list (fainting, seizure,
   accident, surgery, emergency, chicken pox, high fever, hospitalization —
   see `webhook/handlers/leaveApplication.js::URGENT_REASONS`). This list is
   intentionally a documented, editable constant rather than a hidden
   heuristic, so school staff can tune it without code archaeology.
4. **Fallback exhaustion** — Stage 3 of the progressive fallback (Module 3)
   always escalates, guaranteeing no conversation dead-ends after two
   failed clarification attempts.

Every escalation path creates a real ticket via
`db.createSupportTicket()` and returns the ticket ID to the parent in the
same turn — never a bare "someone will contact you" with no reference.
