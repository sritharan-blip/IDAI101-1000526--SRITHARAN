# Module 3 — 3-Tier Progressive Fallback & UX Design

## 3.1 The problem with the shipped fallback

The original agent had exactly one `Default Fallback Intent`, four rotating
generic responses, and a static chip menu. Every failure — first misfire or
tenth in a row — got the identical treatment. That's a flat failure mode,
not a recovery *strategy*: it never gets more helpful, never narrows the
problem down, and never guarantees the parent reaches a human if the bot
truly cannot help.

## 3.2 The new chain

Dialogflow ES only ever auto-triggers on a genuine no-match, but a
**contextual fallback intent** (a fallback intent scoped to an input context)
only matches while that context is active — which is exactly the mechanism
used here to build a real 3-stage escalation without any external state
machine:

```
Turn 1 no-match
   -> "Default Fallback Intent"                (no input context; always eligible)
      sets output context: fallback_stage_1 (lifespan 1)
      => Stage 1: Targeted Clarification

Turn 2 no-match (fallback_stage_1 still active)
   -> "Fallback - Stage2 - CategorizedOptions"  (input context: fallback_stage_1)
      sets output context: fallback_stage_2 (lifespan 1)
      => Stage 2: Categorized Options / Chips Menu

Turn 3 no-match (fallback_stage_2 still active)
   -> "Fallback - Stage3 - EscalateToHuman"     (input context: fallback_stage_2)
      webhookUsed: true -> creates a support ticket
      => Stage 3: Ticket Creation / Human Escalation
```

If the parent's next utterance actually **matches a real intent** at any
point, that intent doesn't extend the `fallback_stage_*` context, so it
naturally expires after one turn (Dialogflow context lifespans decrement
every turn regardless of which intent fires) — the chain self-resets for
free with no extra bookkeeping required.

### Stage 1 — Targeted Clarification

> "I want to make sure I get this right for you — could you rephrase that,
> mentioning what you'd like to know (for example, attendance, fees, or
> homework)?"

Goal: recover from a one-off phrasing miss with a low-friction nudge, no
menu yet — most real users self-correct here.

### Stage 2 — Categorized Options / Chips Menu

> "I'm still having trouble understanding. Let's narrow it down — pick a
> category below and I'll take it from there."
> `[Academics (Homework, Syllabus, Exams)] [Fees & Payments] [Attendance & Leave] [Transport & Facilities] [Talk to a Human]`

Goal: two consecutive misses means the phrasing approach isn't working —
switch the interaction model entirely from free text to tap targets, which
cannot be misclassified.

### Stage 3 — Ticket Creation / Human Escalation

> "I haven't been able to help through chat, so I've raised support ticket
> TCKT-1789024525120 for you. The school office will follow up within one
> business day. You can also call the front desk directly at 8:30 AM–3:30 PM."

Goal: guarantee the conversation never dead-ends. This is webhook-backed
(`handlers/fallbackEscalate.js`) and always returns a real ticket ID in the
same turn — never a bare promise.

## 3.3 Response-template tone rewrite

Every template touched in this refactor follows a single enterprise voice
standard for K-12 school administration:

| Principle | Applied |
|---|---|
| **Authoritative** | States facts plainly ("has been present 78 of 85 school days") rather than hedging ("I think maybe...") |
| **Empathetic** | Urgency-aware phrasing on health/leave escalations ("Because this sounds urgent, I've also alerted the school office directly") |
| **Concise** | One or two sentences per turn; details (ticket IDs, due dates, links) inline rather than as a wall of text |
| **Professional** | No slang, no exclamation-mark enthusiasm, no emoji; consistent "I've..." / "Here's..." register throughout |
| **Honest** | `ChangeChildIntent` no longer promises persistent memory it can't reliably deliver — it confirms exactly what state was set and for how long the effect will last |

Example rewrite (`FeePaymentStatusIntent`, overdue case):

- **Before:** `"$child_name's $fee_type payment status is: Pending."` (flat,
  no due date, no next step, breaks if `fee_type` or `child_name` is blank)
- **After:** `"Transport Fee for Aarav: Pending — ₹12,000 due by
  2026-09-15."` with a `[Pay Now] [Fee Details] [Talk to a Human]` chip row
  when overdue, generated dynamically from live mock-DB state rather than a
  hardcoded percentage/status string.

## 3.4 Chip integrity

All 46 quick-reply chips across the agent were re-verified against the
(now-expanded) training data to confirm each chip's exact text still
resolves to its intended intent under the new `mlMinConfidence` (0.35) and
the collision fixes in Module 1. No chip references a phrase that was moved
or removed without a same-named replacement being added to the receiving
intent (verified programmatically, not by inspection alone — see
`docs/04_OGrade_Rubric_Audit_Report.md §4` for the verification method).
