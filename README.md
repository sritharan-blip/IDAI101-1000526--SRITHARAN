# ParentConnect AI

A Dialogflow ES chatbot that answers parents' questions about their child's school: attendance, exams, fees, homework, transport, events, leave, report cards and more. It uses quick-reply "pills" (chips) so parents can tap instead of type.

This edition runs **entirely inside Dialogflow**. There is no webhook and no server to host. Every answer is a static reply written in the agent itself.

| | |
|---|---|
| Platform | Dialogflow ES (Dialogflow Messenger widget) |
| Intents | 37 (34 conversational + 3 fallback stages) |
| Training phrases | 636, with no duplicates across the agent |
| Entities | 24 custom entities |
| Pills (chips) | 81, and every one maps to a trained phrase |
| Language / timezone | English / Asia/Kolkata |

---

## 1. How it works

```text
Parent ──► Dialogflow Messenger (web widget)
              │
              ▼
        Dialogflow ES agent
          1. Match the message to an intent (NLU)
          2. Extract entities (child name, grade, subject, fee type...)
          3. Ask for any missing required detail (slot filling)
          4. Reply with text + tappable pills
          5. Remember the child/grade in a context
              │
        no match ──► 3-stage fallback (rephrase ► choose a category ► human)
```

A tapped pill sends its text back as if the parent had typed it, so every pill's text is a training phrase on the intent it should open.

---

## 2. Intents

### Student services

| Intent | Required params | Entities | Output contexts | Phrases | Pills shown |
|---|---|---|---|---|---|
| **AttendanceIntent** | `grade` | `@grade`, `@child_name`, `@section` | `active_child_state` | 34 | Check another grade, View Exam Schedule |
| **ChangeChildIntent** | `child_name` | `@child_name`, `@grade` | `active_child_state` | 24 | Check Attendance, Check Homework, Fee Payment Status |
| **ExamResultIntent** | `child_name`, `exam_type` | `@child_name`, `@subject`, `@exam_type`, `@grade` | `active_child_state` | 30 | Download Report Card, Talk to Class Teacher |
| **ExamScheduleIntent** | `grade`, `exam_type` | `@grade`, `@exam_type`, `@subject` | `active_child_state` | 27 | View Announcements, Check Report Card |
| **FeeDetailsIntent** | `fee_type`, `grade` | `@fee_type`, `@grade` | `active_child_state` | 18 | Check Payment Status, Contact Accounts Office |
| **FeePaymentStatusIntent** | `fee_type`, `child_name` | `@fee_type`, `@child_name` | `active_child_state` | 33 | Pay Now, Contact Accounts Office |
| **HomeworkIntent** | `grade`, `subject` | `@grade`, `@subject`, `@child_name` | `active_child_state` | 20 | Check Another Subject, Contact Class Teacher |
| **LeaveApplicationIntent** | `leave_reason`, `child_name` | `@leave_reason`, `@child_name`, `@grade` | `active_child_state` | 26 | Apply for Another Child, Check Homework |
| **ReportCardIntent** | `document_type`, `child_name` | `@document_type`, `@child_name`, `@grade` | `active_child_state` | 32 | Check Exam Results, Contact Accounts Office |
| **SyllabusIntent** | `grade` | `@grade`, `@subject` | `active_child_state` | 15 | Check Homework, Exam Schedule |
| **ScholarshipIntent** | `scholarship_type` | `@scholarship_type` | — | 15 | Contact Accounts Office, Check Fee Details |

### School information

| Intent | Required params | Entities | Output contexts | Phrases | Pills shown |
|---|---|---|---|---|---|
| **AnnouncementIntent** | — | — | — | 17 | Upcoming Events, Holiday List |
| **CafeteriaMenuIntent** | `meal_type` | `@meal_type` | — | 14 | Check School Timings, View Announcements |
| **EventInfoIntent** | `event_type`, `grade` | `@event_type`, `@grade` | `active_child_state` | 17 | View Announcements, Book PTM Slot |
| **ExtracurricularIntent** | `club_activity` | `@club_activity`, `@grade` | `active_child_state` | 15 | Enroll Now, View Event Calendar |
| **HolidayListIntent** | — | `@day_of_week` | — | 14 | School Timings, Upcoming Events |
| **LibraryInfoIntent** | — | — | — | 14 | Check Fee Details, School Timings |
| **SchoolPolicyIntent** | `policy_topic` | `@policy_topic` | — | 15 | Download Handbook, Contact Principal |
| **SchoolTimingsIntent** | — | `@grade` | `active_child_state` | 17 | Check Bus Timing, Holiday List |
| **TransportIntent** | `bus_route` | `@bus_route`, `@grade` | `active_child_state` | 19 | Contact Transport Coordinator, Check School Timings |
| **UniformInfoIntent** | — | `@grade` | `active_child_state` | 14 | School Timings, Fee Details |
| **WeatherClosureIntent** | — | — | — | 15 | View Announcements, Check Bus Status |
| **VolunteerIntent** | `volunteer_area` | `@volunteer_area` | — | 15 | View Event Calendar, Contact Event Coordinator |

### Support & administration

| Intent | Required params | Entities | Output contexts | Phrases | Pills shown |
|---|---|---|---|---|---|
| **ComplaintFeedbackIntent** | `complaint_category` | `@complaint_category` | `human_escalation` | 21 | Track Complaint Status, Talk to Principal |
| **HealthMedicalIntent** | `health_concern`, `child_name` | `@health_concern`, `@child_name` | `active_child_state`, `human_escalation` | 25 | Talk to School Nurse, Apply for Leave |
| **IDCardIntent** | `id_card_issue` | `@id_card_issue` | — | 15 | Contact Accounts Office, Check Fee Details |
| **ParentPortalLoginIntent** | `portal_issue_type` | `@portal_issue_type` | — | 16 | Contact IT Support, Try Again |
| **ParentTeacherMeetingIntent** | `grade` | `@staff_designation`, `@event_type`, `@grade`, `@section` | `active_child_state` | 18 | Confirm Slot, Reschedule |
| **TalkToHumanIntent** | — | — | `human_escalation` | 20 | Contact Class Teacher, School Office Hours, Back to Main Menu |
| **TeacherContactIntent** | `staff_designation` | `@staff_designation`, `@grade`, `@section` | `active_child_state` | 22 | Book a PTM Slot, File a Complaint |

### Conversation

| Intent | Required params | Entities | Output contexts | Phrases | Pills shown |
|---|---|---|---|---|---|
| **Default Welcome Intent** | — | — | — | 1 | Check Attendance, Exam Schedule, Fee Details, Announcements |
| **Greeting Intent** | — | — | — | 13 | Check Attendance, Exam Schedule, Fee Details, Announcements |
| **CapabilitiesIntent** | — | — | — | 12 | Check Attendance, Fee Details, Exam Schedule, Talk to a Teacher |
| **ThanksGoodbyeIntent** | — | — | — | 13 | — |

### Fallback chain

| Intent | Required params | Entities | Output contexts | Phrases | Pills shown |
|---|---|---|---|---|---|
| **Default Fallback Intent** | — | — | `fallback_stage_1` | 0 | — |
| **Fallback - Stage2 - CategorizedOptions** | — | — | `fallback_stage_2` | 0 | Academics (Homework, Syllabus, Exams), Fees & Payments, Attendance & Leave, Transport & Facilities, Talk to a Human |
| **Fallback - Stage3 - EscalateToHuman** | — | — | `human_escalation` | 0 | Back to Main Menu, School Office Hours |

Required parameters are asked for automatically if the parent leaves them out, for example *"Apply medical leave"* triggers *"Which child is this leave application for?"*.

---

## 3. Entities

| Entity | Values |
|---|---|
| `@bus_route` | Route 1, Route 2, Route 3, Route 4 |
| `@child_name` | Aarav, Diya, Kabir, Meera, Ananya, Rohan, Ishaan, Sara, Vivaan, Ayesha, Arjun, Priya |
| `@club_activity` | Chess Club, Robotics Club, Debate Club, Football Team, Basketball Team, Music Club, Art Club, Dance Club |
| `@complaint_category` | Academic, Behavioral, Transport, Fee, Facility |
| `@day_of_week` | Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday |
| `@document_type` | Report Card, Transfer Certificate, Bonafide Certificate, Fee Receipt |
| `@event_type` | Sports Day, Annual Day, Science Fair, Parent-Teacher Meeting, Cultural Fest |
| `@exam_type` | Unit Test, Mid-Term, Final Exam, Surprise Test |
| `@fee_type` | Tuition Fee, Transport Fee, Exam Fee, Library Fee, Lab Fee |
| `@grade` | Grade 1, Grade 2, Grade 3, Grade 4, Grade 5, Grade 6, Grade 7, Grade 8, Grade 9, Grade 10, Grade 11, Grade 12 |
| `@health_concern` | Fever, Injury, Allergy, Medication Request, Emergency |
| `@id_card_issue` | Lost Card, Damaged Card, New Card Request |
| `@leave_reason` | Sick Leave, Personal Reason, Family Function, Travel |
| `@meal_type` | Breakfast, Lunch, Snacks |
| `@payment_status` | Paid, Pending, Overdue |
| `@policy_topic` | Dress Code, Discipline Policy, Anti-Bullying Policy, Mobile Phone Policy, Attendance Policy |
| `@portal_issue_type` | Login Issue, Password Reset, Account Locked, Profile Update |
| `@scholarship_type` | Merit Scholarship, Sports Scholarship, Need-based Scholarship, Sibling Discount |
| `@section` | Section A, Section B, Section C |
| `@staff_designation` | Class Teacher, Principal, Vice Principal, Counselor, Transport Coordinator, Accounts Officer, Event Coordinator |
| `@subject` | Maths, Science, English, Social Studies, Hindi, Computer Science |
| `@urgency_level` | Normal, Urgent |
| `@volunteer_area` | Library Help, Event Support, Canteen Duty, Sports Day Volunteer |
| `@weather_status` | Delayed, Closed, Normal, Early Dismissal |

---

## 4. Contexts

| Context | Set by | Holds | Lifespan |
|---|---|---|---|
| `active_child_state` | 18 child or grade related intents (for example attendance, exams, fees, leave, homework) | `active_child`, `active_grade` | 5 turns (10 for `ChangeChildIntent`) |
| `human_escalation` | `TalkToHumanIntent`, `ComplaintFeedbackIntent`, `HealthMedicalIntent`, fallback stage 3 | — | 1 turn |
| `fallback_stage_1` / `fallback_stage_2` | Fallback intents | — | 1 turn each |

`active_child_state` is **advisory**: it is stored so the child is remembered, but no intent silently reuses it. If a parent leaves out the grade or child, the bot asks. This prevents the earlier "Grade 11 became Grade 8" mix-up.

---

## 5. Three-stage fallback

| Stage | Intent | What the parent sees |
|---|---|---|
| 1 | `Default Fallback Intent` | A request to rephrase, naming the topic (attendance, fees, homework...) |
| 2 | `Fallback - Stage2 - CategorizedOptions` | Pills: Academics, Fees & Payments, Attendance & Leave, Transport & Facilities, Talk to a Human |
| 3 | `Fallback - Stage3 - EscalateToHuman` | A message that the school office will follow up within one business day, plus the front-desk hours |

Stages are chained with contexts, so two failed messages in a row move the parent to the next stage, and any successful match starts over.

---

## 6. Setup

1. Open the [Dialogflow ES console](https://dialogflow.cloud.google.com/) and create or open an agent.
2. Zip the **contents** of the `dialogflow_agent/` folder so `agent.json` is at the root of the zip. Zipping the folder itself fails the import.
3. Go to ⚙ Settings → Export and Import → **Restore from zip** and upload it. This replaces the whole agent.
4. Go to Integrations → **Dialogflow Messenger**, enable it and paste the embed snippet into a web page.
5. Leave Fulfillment → Webhook switched **off**.

The pills only display in Dialogflow Messenger. The console's test panel shows the text but not the pills.

---

## 7. Try it

| Say | Expect |
|---|---|
| `hi` | A welcome message and pills: Check Attendance, Exam Schedule, Fee Details, Announcements |
| `Is Aarav's tuition fee paid?` | The fee-status reply, with pills |
| `Apply medical leave` | Asks which child, then confirms the leave application |
| `Which bus route do I use?` | Asks for the route, then the transport reply |
| `What can you do?` | The list of capabilities |
| Tap any pill | The matching intent answers, never the fallback |
| `asdf qwer` three times in a row | Stage 1, then the category pills, then the escalation message |

---

## 8. Limitations

- **The answers are static.** The bot does not look up a real student. For example `AttendanceIntent` always replies with the same percentage and `FeePaymentStatusIntent` always says "Pending". Treat these as demo replies.
- **Nothing is stored or sent.** Leave applications, support requests and nurse alerts are confirmation messages only.
- **The remembered child is not applied automatically.** The bot asks for the child or grade each time it is needed.
- **Pills show only in Dialogflow Messenger.** Other channels need their own rendering.
- To use real data later, add a webhook back (set `webhookUsed` on the intents that need it and enable Fulfillment), or connect a school information system.

---

## 9. Repository layout

```text
README.md
dialogflow_agent/
  agent.json        agent settings (webhook disabled)
  intents/          37 intents and their training phrases
  entities/         24 entities with their values and synonyms
```
