# ParentConnect AI — WACP Council Evaluator Guide

**Dialogflow ES school-parent assistant · Enhanced edition (4 new feature tracks, 12 new intents, 9 new entities)**

> **Build mode: static responses, no webhook.** In this edition every answer comes from Dialogflow itself (static replies, slot-filling prompts, contexts and quick-reply chips). `webhook.available` is `false` and no intent calls a webhook. The original Node.js/Express webhook in `webhook/` is kept in the repository as an optional extension and is **not** used. Replies are therefore illustrative and do not look up a live school database (see §8 *Known limitations*).

---

## 1. System architecture overview

```text
 Parent ──► Dialogflow Messenger (web widget, renders chips)
                │
                ▼
 ┌──────────────────────────────────────────────────────────────┐
 │                      Dialogflow ES agent                     │
 │                                                              │
 │  NLU ─► intent match ─► entity extraction ─► slot filling    │
 │  (853 training phrases, 49 intents, 33 entities)             │
 │                │                                             │
 │                ▼                                             │
 │  Session Context Engine                                      │
 │   active_child_state {active_child, active_grade}  (5-10 t)  │
 │   ptm_booking · feedback_ticket · fallback_stage_1/2         │
 │                │                                             │
 │                ▼                                             │
 │  Response: text variants + rich-content chips (pills)        │
 │                                                              │
 │  3-tier fallback: clarify ► category chips ► escalate        │
 └──────────────────────────────────────────────────────────────┘
                │   (optional, disabled)
                ▼
      webhook/ (Node.js + Express)  ← retained for future live data
```

### Alignment with WACP AI competencies

| WACP competency | How it is demonstrated |
|---|---|
| **NLU precision** | 853 training phrases, 0 duplicate phrases across the whole agent (scripted check), entity-annotated phrases in varied sentence shapes, synonyms for every new entity value, every chip text trained as an exact phrase on its target intent. |
| **Parameter extraction** | 33 entities (9 new + `@sys.date`/`@sys.time`). Multi-parameter utterances such as *"Pre-order a veg sandwich for Aarav tomorrow"* fill `meal_item`, `child_name` and `date` in one turn. |
| **Multi-turn state management** | `active_child_state` is written by every child-aware intent and used as a fallback default for `child_name` / `grade`; an explicit name in the message always overrides it. `PTMSlotBookingIntent` collects subject → date → time over several turns. |
| **Failure recovery** | Three-stage fallback chain built with ES contextual fallback intents; required slots re-prompt with the list of valid values (e.g. *"Routes 1 to 8 are available"*). |

---

## 2. What was added in this edition

| Track | New intents | New entities |
|---|---|---|
| 1. Academic performance & analytics | `AcademicProgressIntent`, `SubjectWeaknessIntent`, `AssignmentStatusIntent` | `@grade_term`, `@assignment_status`, `@subject_code` |
| 2. Financial & administrative operations | `FeeInstallmentCalculatorIntent`, `FeeDeferralRequestIntent`, `BusRouteETAIntent`, `DigitalGatePassIntent` | `@payment_method`, `@installment_plan`, `@route_number`, `@gatepass_reason` |
| 3. Health, wellness & cafeteria | `SickBayUpdateIntent`, `DietaryAllergyLogIntent`, `CafeteriaPreOrderIntent` | `@allergy_type`, `@meal_item`, `@urgency_level` (existing entity, extended with synonyms such as *serious*, *severe*, *immediately*) |
| 4. Parent–teacher ecosystem & escalation | `PTMSlotBookingIntent`, `AutomatedFeedbackRequestIntent` | reuses `@subject`, `@complaint_category`, `@urgency_level`, `@sys.date`, `@sys.time` |

Other changes: `CapabilitiesIntent` now advertises the new features with extra chips; the six former webhook intents (`AttendanceIntent`, `ChangeChildIntent`, `FeePaymentStatusIntent`, `LeaveApplicationIntent`, `ReportCardIntent`, `Fallback - Stage3`) now answer statically and carry chips; chips were added to the generic fallback and goodbye replies; several chip labels that did not resolve to a trained phrase were fixed.

---

## 3. Feature & intent matrix

Generated directly from the agent JSON, so it matches what you import.

### 3.1 New intents (12)

**Track 1 Academic**

| Intent | Required params | Optional params | Entity types | Output contexts | Phrases | Backend |
|---|---|---|---|---|---|---|
| **AcademicProgressIntent** | `child_name` | `grade_term`, `subject`, `grade` | `@child_name`, `@grade`, `@grade_term`, `@subject` | `active_child_state` | 20 | Static reply + chips |
| **AssignmentStatusIntent** | `child_name` | `assignment_status`, `subject_code`, `subject`, `grade` | `@assignment_status`, `@child_name`, `@grade`, `@subject`, `@subject_code` | `active_child_state` | 18 | Static reply + chips |
| **SubjectWeaknessIntent** | `child_name` | `grade_term`, `grade` | `@child_name`, `@grade`, `@grade_term` | `active_child_state` | 18 | Static reply + chips |

**Track 2 Finance/Admin**

| Intent | Required params | Optional params | Entity types | Output contexts | Phrases | Backend |
|---|---|---|---|---|---|---|
| **BusRouteETAIntent** | `route_number` | `child_name` | `@child_name`, `@route_number` | `active_child_state` | 18 | Static reply + chips |
| **DigitalGatePassIntent** | `child_name`, `gatepass_reason` | `time`, `grade` | `@child_name`, `@gatepass_reason`, `@grade`, `@sys.time` | `active_child_state` | 18 | Static reply + chips |
| **FeeDeferralRequestIntent** | `child_name` | `fee_type`, `installment_plan`, `grade` | `@child_name`, `@fee_type`, `@grade`, `@installment_plan` | `active_child_state` | 16 | Static reply + chips |
| **FeeInstallmentCalculatorIntent** | `installment_plan` | `fee_type`, `payment_method`, `child_name` | `@child_name`, `@fee_type`, `@installment_plan`, `@payment_method` | `active_child_state` | 18 | Static reply + chips |

**Track 3 Health/Cafeteria**

| Intent | Required params | Optional params | Entity types | Output contexts | Phrases | Backend |
|---|---|---|---|---|---|---|
| **CafeteriaPreOrderIntent** | `child_name`, `meal_item` | `date`, `meal_type`, `grade` | `@child_name`, `@grade`, `@meal_item`, `@meal_type`, `@sys.date` | `active_child_state` | 16 | Static reply + chips |
| **DietaryAllergyLogIntent** | `child_name`, `allergy_type` | `urgency_level`, `grade` | `@allergy_type`, `@child_name`, `@grade`, `@urgency_level` | `active_child_state` | 16 | Static reply + chips |
| **SickBayUpdateIntent** | `child_name` | `health_concern`, `urgency_level`, `grade` | `@child_name`, `@grade`, `@health_concern`, `@urgency_level` | `active_child_state` | 16 | Static reply + chips |

**Track 4 PTM/Escalation**

| Intent | Required params | Optional params | Entity types | Output contexts | Phrases | Backend |
|---|---|---|---|---|---|---|
| **AutomatedFeedbackRequestIntent** | `complaint_category` | `urgency_level` | `@complaint_category`, `@urgency_level` | `feedback_ticket` | 16 | Static reply + chips |
| **PTMSlotBookingIntent** | `child_name`, `subject`, `date`, `time` | — | `@child_name`, `@subject`, `@sys.date`, `@sys.time` | `active_child_state`, `ptm_booking` | 18 | Static reply + chips |

### 3.2 Baseline intents (37)

| Intent | Required params | Optional params | Entity types | Output contexts | Phrases | Backend |
|---|---|---|---|---|---|---|
| **AnnouncementIntent** | — | — | — | — | 17 | Static reply + chips |
| **AttendanceIntent** | `grade` | `child_name`, `section` | `@child_name`, `@grade`, `@section` | `active_child_state` | 37 | Static reply + chips (legacy `attendance.js` available, disabled) |
| **CafeteriaMenuIntent** | `meal_type` | — | `@meal_type` | — | 15 | Static reply + chips |
| **CapabilitiesIntent** | — | — | — | — | 12 | Static reply + chips |
| **ChangeChildIntent** | `child_name` | `grade` | `@child_name`, `@grade` | `active_child_state` | 25 | Static reply + chips (legacy `changeChild.js` available, disabled) |
| **ComplaintFeedbackIntent** | `complaint_category` | — | `@complaint_category` | `human_escalation` | 21 | Static reply + chips |
| **Default Fallback Intent** | — | — | — | `fallback_stage_1` | 0 | Static reply + chips |
| **Default Welcome Intent** | — | — | — | — | 1 | Static reply + chips |
| **EventInfoIntent** | `event_type`, `grade` | — | `@event_type`, `@grade` | `active_child_state` | 17 | Static reply + chips |
| **ExamResultIntent** | `child_name`, `exam_type` | `subject`, `grade` | `@child_name`, `@exam_type`, `@grade`, `@subject` | `active_child_state` | 30 | Static reply + chips |
| **ExamScheduleIntent** | `grade`, `exam_type` | `subject` | `@exam_type`, `@grade`, `@subject` | `active_child_state` | 27 | Static reply + chips |
| **ExtracurricularIntent** | `club_activity` | `grade` | `@club_activity`, `@grade` | `active_child_state` | 15 | Static reply + chips |
| **Fallback - Stage2 - CategorizedOptions** | — | — | — | `fallback_stage_2` | 0 | Static reply + chips |
| **Fallback - Stage3 - EscalateToHuman** | — | — | — | `human_escalation` | 0 | Static reply + chips (legacy `fallbackEscalate.js` available, disabled) |
| **FeeDetailsIntent** | `fee_type`, `grade` | — | `@fee_type`, `@grade` | `active_child_state` | 19 | Static reply + chips |
| **FeePaymentStatusIntent** | `fee_type`, `child_name` | — | `@child_name`, `@fee_type` | `active_child_state` | 33 | Static reply + chips (legacy `feePaymentStatus.js` available, disabled) |
| **Greeting Intent** | — | — | — | — | 13 | Static reply + chips |
| **HealthMedicalIntent** | `health_concern`, `child_name` | — | `@child_name`, `@health_concern` | `active_child_state`, `human_escalation` | 25 | Static reply + chips |
| **HolidayListIntent** | — | `day_of_week` | `@day_of_week` | — | 14 | Static reply + chips |
| **HomeworkIntent** | `grade`, `subject` | `child_name` | `@child_name`, `@grade`, `@subject` | `active_child_state` | 20 | Static reply + chips |
| **IDCardIntent** | `id_card_issue` | — | `@id_card_issue` | — | 15 | Static reply + chips |
| **LeaveApplicationIntent** | `leave_reason`, `child_name` | `grade` | `@child_name`, `@grade`, `@leave_reason` | `active_child_state` | 27 | Static reply + chips (legacy `leaveApplication.js` available, disabled) |
| **LibraryInfoIntent** | — | — | — | — | 14 | Static reply + chips |
| **ParentPortalLoginIntent** | `portal_issue_type` | — | `@portal_issue_type` | — | 16 | Static reply + chips |
| **ParentTeacherMeetingIntent** | `grade` | `staff_designation`, `event_type`, `section` | `@event_type`, `@grade`, `@section`, `@staff_designation` | `active_child_state` | 18 | Static reply + chips |
| **ReportCardIntent** | `document_type`, `child_name` | `grade` | `@child_name`, `@document_type`, `@grade` | `active_child_state` | 34 | Static reply + chips (legacy `reportCard.js` available, disabled) |
| **ScholarshipIntent** | `scholarship_type` | — | `@scholarship_type` | — | 15 | Static reply + chips |
| **SchoolPolicyIntent** | `policy_topic` | — | `@policy_topic` | — | 15 | Static reply + chips |
| **SchoolTimingsIntent** | — | `grade` | `@grade` | `active_child_state` | 17 | Static reply + chips |
| **SyllabusIntent** | `grade` | `subject` | `@grade`, `@subject` | `active_child_state` | 15 | Static reply + chips |
| **TalkToHumanIntent** | — | — | — | `human_escalation` | 20 | Static reply + chips |
| **TeacherContactIntent** | `staff_designation` | `grade`, `section` | `@grade`, `@section`, `@staff_designation` | `active_child_state` | 22 | Static reply + chips |
| **ThanksGoodbyeIntent** | — | — | — | — | 13 | Static reply + chips |
| **TransportIntent** | `bus_route` | `grade` | `@bus_route`, `@grade` | `active_child_state` | 19 | Static reply + chips |
| **UniformInfoIntent** | — | `grade` | `@grade` | `active_child_state` | 14 | Static reply + chips |
| **VolunteerIntent** | `volunteer_area` | — | `@volunteer_area` | — | 15 | Static reply + chips |
| **WeatherClosureIntent** | — | — | — | — | 15 | Static reply + chips |

---

## 4. Context engine

| Context | Set by | Parameters | Lifespan | Purpose |
|---|---|---|---|---|
| `active_child_state` | All child-aware intents | `active_child`, `active_grade` | 5 turns (10 for `ChangeChildIntent`) | Remember which child the parent is talking about |
| `ptm_booking` | `PTMSlotBookingIntent` | `child`, `subject`, `date`, `time` | 5 | Carries the pending meeting details for follow-ups such as *Reschedule* |
| `feedback_ticket` | `AutomatedFeedbackRequestIntent` | `category`, `urgency` | 5 | Carries the ticket category for follow-ups such as *Track Complaint Status* |
| `fallback_stage_1` → `fallback_stage_2` → `human_escalation` | Fallback intents | — | 1 each | Drives the 3-tier fallback chain |

**How the active child is used.** In the new child-aware intents, `child_name` is a required slot whose default is `#active_child_state.active_child` (and `grade` defaults to `#active_child_state.active_grade`).

1. If the parent names a child in the message, that name is used, and it replaces the active child.
2. If no child is named and a child is active, the active child is used. **Every reply repeats the child's name**, so a wrong assumption is visible straight away.
3. If no child is named and none is active, the bot asks *"Which child is this for?"*.

---

## 5. Three-tier fallback

| Stage | Intent | Behaviour |
|---|---|---|
| 1 | `Default Fallback Intent` | Asks the parent to rephrase and name the topic; shows main-menu chips |
| 2 | `Fallback - Stage2 - CategorizedOptions` | Quick-choice chips: Academics, Fees & Payments, Attendance & Leave, Transport & Facilities, Talk to a Human |
| 3 | `Fallback - Stage3 - EscalateToHuman` | Tells the parent the matter has been passed to the school office (reply within one business day, front-desk hours) and offers *Back to Main Menu* |

Stages are chained with output/input contexts, so a successful match at any point resets the chain.

---

## 6. Council test bench

Type these into the Dialogflow console simulator or the Messenger widget. Start each test in a **new session** (reset the simulator or reload the page) unless stated otherwise. The expected results below follow from the agent configuration; the structure was machine-checked, but please confirm the live behaviour in the simulator.

### Test 1 — Happy path: multi-parameter slot filling

| Step | Parent says | Expected |
|---|---|---|
| 1a | `I need a gate pass for Aarav for early pickup at 2 pm` | No questions asked. Reply names **Aarav** and **Early Pickup**. Chips: Talk to Class Teacher · Check School Timings · Back to Main Menu. `active_child_state` = Aarav |
| 1b | `Pre-order a veg sandwich for Ananya tomorrow` | Reply confirms **Veg Sandwich** for **Ananya**; allergy note; chips include Log Food Allergy |
| 1c | `Book a meeting with the Maths teacher for Aarav` | Bot asks *"Which date would you prefer…?"* → reply `Friday` → bot asks *"What time works best…?"* → reply `4 pm` → confirmation with Maths, Aarav, date and time |
| 1d | `Pre-order lunch` | Bot asks *"Which child is this for?"* → `Diya` → asks which item → `pasta` → confirmation |

### Test 2 — Multi-turn context switching (`ChangeChildIntent` → `AcademicProgressIntent`)

| Step | Parent says | Expected |
|---|---|---|
| 2a | `I actually want info for my daughter Ananya` | `ChangeChildIntent` fires; active child becomes **Ananya** |
| 2b | `Show me academic progress` | **No question about the child.** Reply is about **Ananya**. Chips: Weak Subjects · Assignment Status · Download Report Card |
| 2c | `Weak Subjects` (tap the chip) | `SubjectWeaknessIntent`; reply names **Ananya** |
| 2d | `How is Kabir doing academically` | Explicit name wins. Reply names **Kabir** and the active child switches to Kabir |
| 2e | `Assignment status` | Reply names **Kabir**, not Ananya |

### Test 3 — Three-tier fallback

| Step | Parent says | Expected |
|---|---|---|
| 3a | `blorp zxqv wibble` | Stage 1: asks the parent to rephrase, with main-menu chips |
| 3b | `qwerty asdf lorem` (immediately after) | Stage 2: category chips (Academics, Fees & Payments, Attendance & Leave, Transport & Facilities, Talk to a Human) |
| 3c | `zzz yyy xxx` (immediately after) | Stage 3: escalation message, chips *Back to Main Menu* and *School Office Hours* |
| 3d | Tap *Fees & Payments* during stage 2 instead of 3c | `FeeDetailsIntent` answers; the fallback chain is reset |

### Test 4 — Edge cases

| Scenario | Parent says | Expected |
|---|---|---|
| Missing student (fresh session) | `Generate a gate pass for early pickup` | Bot asks *"Which child is this for?"* instead of guessing |
| Student name not on the roster (fresh session) | `Show academic progress for Zoltan` | `Zoltan` is not a known `@child_name`, so the bot asks for the child again; no progress reply is produced for an unknown name |
| Invalid route | `ETA of route 99` | Bot does not give an ETA; it re-prompts *"Which route number is it? Routes 1 to 8 are available."* or falls back |
| Missing required slot | `Request a fee deferral` | Bot asks for the child before confirming |
| Unsupported allergy | `Log an allergy to pollen for Diya` | Bot asks *"What is the allergy or dietary restriction? For example peanut, dairy, gluten or egg."* |
| Out-of-range date | `Book a meeting with the Maths teacher for Aarav on 31 February at 4 pm` | Dialogflow's `@sys.date` does not validate calendar logic in this static build. See §8 |

### Test 5 — Chips

Tap every chip on the welcome message, a new-track reply and each fallback stage. Every chip should route to an intent and never to the fallback. A script confirmed that each of the 126 chips matches a training phrase exactly.

---

## 7. Setup & deployment

### 7.1 Import the agent into Dialogflow ES

1. Create or open an agent in the [Dialogflow ES console](https://dialogflow.cloud.google.com/).
2. Zip the **contents** of `dialogflow_agent/` so that `agent.json` sits at the root of the zip. Zipping the folder itself fails the import.
3. In the console go to ⚙ Settings → Export and Import → **Restore from zip** (this replaces the agent) and upload the zip.
4. Go to Integrations → **Dialogflow Messenger**, enable it and paste the embed snippet into a web page. Chips only render in Messenger, not in the console's test panel.
5. Leave Fulfillment → Webhook **disabled**.

### 7.2 Optional: legacy webhook (not needed)

The `webhook/` folder is retained for future live-data work. To run it:

```bash
cd webhook
npm install
cp .env.example .env     # set WEBHOOK_AUTH_TOKEN
npm test                 # dependency-free smoke test
npm start                # http://localhost:8080
```

Using it also requires a public HTTPS URL, enabling Fulfillment in Dialogflow and setting `webhookUsed` back to `true` on the six webhook intents. See `webhook/README.md`. **This edition does not include new webhook handlers for the 12 new intents.**

### 7.3 Repository layout

```text
dialogflow_agent/     agent.json · intents/ (49 intents + training phrases) · entities/ (33)
webhook/              optional Node.js/Express service (unused in this edition)
docs/                 NLU disambiguation matrix, context engine, fallback UX, O-Grade audit,
                      and the previous README (README_baseline_webhook_edition.md)
BUGFIX_REPORT.md      history of the defects fixed in earlier editions
```

---

## 8. Known limitations (stated openly)

- **Static, illustrative answers.** Replies describe where to find information or confirm a request; they do not read a real student record. The previous baseline's hard-coded figures (such as attendance percentages) remain in the six former webhook intents.
- **No real side effects.** *Gate pass issued*, *ticket created*, *deferral recorded* and *pre-order placed* are confirmation messages only. Nothing is stored, and no reference numbers are generated.
- **No validation beyond entities.** Unknown names, unsupported allergies and route numbers outside 1–8 are caught by entity matching and re-prompts. Calendar validity of dates, past dates and clashing PTM slots are not checked.
- **Active-child default can mask a typo.** If a child is already active and the parent names an unrecognised child, Dialogflow may fall back to the active child. The reply repeats the child's name so the parent can notice and correct it.
- **Untested against a live Dialogflow project.** JSON structure, entity references, duplicate phrases and chip resolution were verified by script. Intent ranking and slot-filling were not run in a live agent. Run §6 in the simulator before relying on the results.
- **Making it live** would need the webhook handlers, `mockDb` data and routing for the new intents, or a real school-information integration.

---

## 9. Grading rubric scorecard

Criteria follow the O-Grade rubric used in `docs/04_OGrade_Rubric_Audit_Report.md`. The ratings are a self-assessment of this static edition.

| Criterion | Evidence in this edition | Gaps |
|---|---|---|
| **NLU accuracy** | 853 phrases across 49 intents; 12 new intents with 16-20 annotated phrases each; synonym-rich entities; 0 duplicate phrases; every chip resolves to a trained phrase; `mlMinConfidence` 0.35 with the three-stage fallback behind it | Live confusion between similar intents (for example `PTMSlotBookingIntent` vs `ParentTeacherMeetingIntent`) should be checked in the simulator |
| **State retention** | `active_child_state` written by all child-aware intents; used as default with explicit-input override; `ptm_booking` and `feedback_ticket` carry multi-turn details; every reply names the child | State lives only in Dialogflow contexts (a few turns), not in persistent storage |
| **Code architecture** | Entities, intents and responses cleanly separated; the 12 new intents were generated from one declarative spec and validated by script; the optional webhook keeps its routes / handlers / lib / db layering | No new webhook handlers in this edition (static build by design) |
| **Safety** | No personal data stored; no silent child substitution (the name is always echoed); allergy replies tell the parent to send medical details to the nurse; urgent health matters point to the front desk; unsupported requests escalate to a human | Replies that confirm actions could be read as real; see §8 |
| **User experience** | Quick-reply chips on every intent reply (126 chips); short, parent-friendly text with text variants; prompts list the valid choices; the capabilities menu advertises the new features | Chips render only in Dialogflow Messenger |
