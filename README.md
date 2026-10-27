# ParentConnect AI

A Dialogflow ES assistant that answers parents' questions about their child's school with **detailed, concrete replies and tap-to-answer pills**: attendance, exam timetable, results, a full report card, fees, transport, cafeteria, health, meetings and more.

It runs **entirely inside Dialogflow. There is no webhook and no server to host.**

**LIVE CHATBOT LINK**: https://sritharan-blip.github.io/IDAI101-1000526--SRITHARAN/

| | |
|---|---|
| Platform | Dialogflow ES, shown in the Dialogflow Messenger widget |
| Intents | 49 |
| Training phrases | 1,645, with no duplicates across the agent |
| Entities | 33 |
| Pills (quick-reply buttons) | 223 across all replies, 4 to 6 on every reply |
| Language / timezone | English / Asia/Kolkata |

> **About the data.** All figures in the replies (marks, attendance, fees, dates, bus times, contacts) are **sample data** written into the agent, so every child sees the same numbers. Replace them with your school's real details by editing each intent's response text in the Dialogflow console (see section 10).

---

## 1. What changed in this version

| Problem | Fix |
|---|---|
| The bot answered the wrong topic | Found overlapping intents with a cross-validation test, then moved or re-pointed **33 phrases** and added about 800 new, distinct ones. Phrases that belonged to two topics (PTM information vs PTM booking, bus routes vs bus ETA, contact lists vs "talk to a human") were separated. |
| Too few pills | Every reply now has **4 to 6 pills** that lead to the most useful next step. The welcome and fallback replies have 5 to 6. Every pill is also a trained phrase, so tapping one never lands in the fallback. |
| Vague, generic answers | Replies are now full, formatted answers with concrete figures: a Mid-Term timetable, a full report card, a fee breakdown, leave and gate-pass procedures, bus routes with times, and contact extensions. |
| Pills went to the wrong place | Pills such as *Contact Accounts Office*, *Talk to School Nurse*, *Try Again* and *Reschedule* were trained to unrelated intents. They now go to the right one. |
| Questions that were not needed | Only the details a reply truly needs are asked for. For example *"Show attendance"* no longer insists on a grade. |

---

## 2. How it works

```text
Parent ──► Dialogflow Messenger
              │
              ▼
        Dialogflow ES agent
          1. Match the message to one of 49 intents
          2. Extract entities (child, subject, route, date, time...)
          3. Ask only for the details the reply needs
          4. Reply with a detailed answer and 4 to 6 pills
          5. Remember the child in the active_child_state context
              │
   no match ──► 3-stage fallback: rephrase ► category pills ► contact the office
```

A tapped pill sends its text as if the parent had typed it, and each pill text is a training phrase on its target intent.

---

## 3. Sample data used in the replies

### Report card (Term 1, 2026-27)

| Subject | Theory /80 | Internal /20 | Total /100 | Grade | Class average |
|---|---|---|---|---|---|
| English | 68 | 18 | 86 | A | 79 |
| Mathematics | 74 | 18 | 92 | A1 | 81 |
| Hindi | 62 | 16 | 78 | B | 74 |
| Science | 70 | 18 | 88 | A | 80 |
| Social Studies | 59 | 15 | 74 | B | 72 |
| Computer Science | 76 | 19 | 95 | A1 | 85 |

**Total 513/600 · Percentage 85.5% · Overall grade A · Class rank 7 of 42 · Attendance 93.6% (131 of 140 days) · Conduct A.** The report card reply also shows the class teacher's remarks and the grading scale (A1 91-100, A 81-90, B 71-80, C 61-70, D 51-60, E below 51).

Attendance: 140 working days, 131 present, 9 absent, 2 late arrivals. October: 17 of 19 days present.

### Mid-Term timetable (9:00 - 11:30 AM)

| Date | Subject |
|---|---|
| Mon 16 Nov | English |
| Tue 17 Nov | Mathematics |
| Wed 18 Nov | Hindi |
| Thu 19 Nov | Science |
| Fri 20 Nov | Social Studies |
| Mon 23 Nov | Computer Science |

Other exams: Unit Test 2 (Mon 11 Jan - Wed 13 Jan 2027) and the Final Exam (Mon 8 Mar - Fri 19 Mar 2027).

### Fees (same for all grades)

Tuition ₹16,000 per term, transport ₹4,000 per term, exam ₹1,500, library ₹1,000 and lab ₹2,000 per year: **₹64,500 a year** with transport. Payment plans: full payment ₹62,565 (3% off), 2 installments of ₹32,250, 4 installments of ₹16,125, or monthly ₹5,375. The fee status reply shows ₹22,000 outstanding (Term 2 tuition and transport pending, lab fee overdue).

### Bus routes

| Route | Stops | First pickup | Reaches school |
|---|---|---|---|
| Route 1 | Lakeview Colony → Sector 12 → City Mall | 6:50 AM | 7:40 AM |
| Route 2 | Green Park → Rose Garden → Metro Station | 6:45 AM | 7:35 AM |
| Route 3 | Hill View → Market Road → Bus Depot | 6:55 AM | 7:45 AM |
| Route 4 | Airport Road → Orchid Lane → Temple Street | 6:40 AM | 7:35 AM |
| Route 5 | Sunrise Apartments → Palm Grove → Library Chowk | 7:00 AM | 7:45 AM |
| Route 6 | North Gate → Cedar Park → Hospital Circle | 6:50 AM | 7:40 AM |
| Route 7 | Riverside → Old Town → Clock Tower | 6:45 AM | 7:40 AM |
| Route 8 | East End → Tech Park → Stadium Road | 6:55 AM | 7:50 AM |

### Events

| Date | Event | When |
|---|---|---|
| Sat 28 Nov | Parent-Teacher Meeting | 9:00 AM – 12:30 PM |
| Sat 12 Dec | Annual Day | 5:00 PM, school auditorium |
| Sat 9 Jan | Sports Day | 8:30 AM, school ground |
| Fri 29 Jan | Science Fair | 10:00 AM – 2:00 PM |
| Sat 20 Feb | Cultural Fest | 10:00 AM onwards |

---

## 4. Intents and pills

### Academics

| Intent | Asks for (if missing) | Sets context | Phrases | Pills |
|---|---|---|---|---|
| **AttendanceIntent** | `child_name` | `active_child_state` | 65 | Apply for Leave, Report Card, Exam Schedule, Academic Progress, Digital Gate Pass |
| **ExamScheduleIntent** | — | — | 44 | Mid-Term Syllabus, Homework Today, Check Exam Results, Academic Progress, School Timings |
| **ExamResultIntent** | `child_name` | `active_child_state` | 62 | Download Report Card, Academic Progress, Weak Subjects, Exam Schedule, Talk to Class Teacher |
| **ReportCardIntent** | `child_name` | `active_child_state` | 54 | Academic Progress, Weak Subjects, Check Exam Results, Request Documents, Talk to Class Teacher |
| **HomeworkIntent** | — | `active_child_state` | 37 | Assignment Status, Mid-Term Syllabus, Exam Schedule, Contact Class Teacher, Academic Progress |
| **SyllabusIntent** | — | — | 33 | Exam Schedule, Homework Today, Weak Subjects, Contact Class Teacher, Academic Progress |
| **AcademicProgressIntent** | `child_name` | `active_child_state` | 52 | Weak Subjects, Assignment Status, Download Report Card, Exam Schedule, Book Teacher Meeting |
| **SubjectWeaknessIntent** | `child_name` | `active_child_state` | 49 | Book Teacher Meeting, Contact Class Teacher, Academic Progress, Mid-Term Syllabus, Assignment Status |
| **AssignmentStatusIntent** | `child_name` | `active_child_state` | 41 | Homework Today, Weak Subjects, Contact Class Teacher, Academic Progress, Exam Schedule |
| **DocumentRequestIntent** | — | — | 26 | Download Report Card, Contact Accounts Office, Check Payment Status, Parent Portal Help |

### Fees

| Intent | Asks for (if missing) | Sets context | Phrases | Pills |
|---|---|---|---|---|
| **FeeDetailsIntent** | — | — | 46 | Check Payment Status, Fee Installment Options, Request Fee Deferral, Scholarships, Contact Accounts Office |
| **FeePaymentStatusIntent** | `child_name` | `active_child_state` | 69 | Fee Installment Options, Request Fee Deferral, Request Documents, Fee Details, Contact Accounts Office |
| **FeeInstallmentCalculatorIntent** | — | `active_child_state` | 32 | Request Fee Deferral, Check Payment Status, Fee Details, Contact Accounts Office |
| **FeeDeferralRequestIntent** | `child_name` | `active_child_state` | 30 | Fee Installment Options, Check Payment Status, Contact Accounts Office, Scholarships |
| **ScholarshipIntent** | — | — | 26 | Fee Details, Request Fee Deferral, Contact Accounts Office, Fee Installment Options |

### Transport & cafeteria

| Intent | Asks for (if missing) | Sets context | Phrases | Pills |
|---|---|---|---|---|
| **TransportIntent** | — | — | 29 | Bus Route ETA, Contact Transport Coordinator, School Timings, Fee Details, Digital Gate Pass |
| **BusRouteETAIntent** | `bus_route` | `active_child_state` | 55 | Bus Routes, Contact Transport Coordinator, Digital Gate Pass, School Timings |
| **CafeteriaMenuIntent** | — | — | 38 | Pre-order Lunch, Log Food Allergy, School Timings, Announcements |
| **CafeteriaPreOrderIntent** | `child_name`, `meal_item` | `active_child_state` | 36 | Cafeteria Menu, Log Food Allergy, School Timings, Back to Main Menu |

### School information

| Intent | Asks for (if missing) | Sets context | Phrases | Pills |
|---|---|---|---|---|
| **EventInfoIntent** | — | — | 31 | PTM Date, Book Teacher Meeting, Holiday List, Clubs and Activities, Announcements |
| **ExtracurricularIntent** | — | — | 34 | Upcoming Events, Volunteer, Fee Details, Contact Class Teacher |
| **HolidayListIntent** | — | — | 35 | Upcoming Events, Exam Schedule, School Timings, Announcements |
| **AnnouncementIntent** | — | — | 27 | Exam Schedule, PTM Date, Fee Details, Scholarships, Holiday List |
| **LibraryInfoIntent** | — | — | 22 | Fee Details, School Timings, Volunteer, Announcements |
| **SchoolPolicyIntent** | — | — | 27 | Uniform Details, Apply for Leave, Contact Principal, Contact Class Teacher |
| **SchoolTimingsIntent** | — | — | 27 | Bus Routes, Holiday List, Cafeteria Menu, Digital Gate Pass, Contact Class Teacher |
| **UniformInfoIntent** | — | — | 21 | School Policies, Fee Details, School Timings, Announcements |
| **WeatherClosureIntent** | — | — | 24 | Announcements, Bus Route ETA, Holiday List, School Timings |
| **VolunteerIntent** | — | — | 26 | Upcoming Events, Contact Event Coordinator, Clubs and Activities, Announcements |

### Health, leave & safety

| Intent | Asks for (if missing) | Sets context | Phrases | Pills |
|---|---|---|---|---|
| **HealthMedicalIntent** | — | `active_child_state`, `human_escalation` | 42 | Sick Bay Update, Log Food Allergy, Apply for Leave, Contact Class Teacher, Staff Directory |
| **SickBayUpdateIntent** | `child_name` | `active_child_state` | 43 | Digital Gate Pass, Log Food Allergy, Apply for Leave, Staff Directory |
| **DietaryAllergyLogIntent** | `child_name`, `allergy_type` | `active_child_state` | 35 | Pre-order Lunch, Cafeteria Menu, Talk to School Nurse, Back to Main Menu |
| **LeaveApplicationIntent** | `child_name`, `leave_reason` | `active_child_state` | 45 | Check Attendance, Digital Gate Pass, Homework Today, Contact Class Teacher |
| **DigitalGatePassIntent** | `child_name`, `gatepass_reason` | `active_child_state` | 36 | Apply for Leave, Talk to Class Teacher, School Timings, Back to Main Menu |

### Meetings, contacts & support

| Intent | Asks for (if missing) | Sets context | Phrases | Pills |
|---|---|---|---|---|
| **ParentTeacherMeetingIntent** | — | — | 24 | Book Teacher Meeting, Download Report Card, Contact Class Teacher, Upcoming Events |
| **PTMSlotBookingIntent** | `child_name`, `meeting_with`, `date`, `time` | `active_child_state`, `ptm_booking` | 44 | Reschedule Meeting, PTM Date, Contact Class Teacher, Back to Main Menu |
| **TeacherContactIntent** | — | — | 41 | Book Teacher Meeting, Talk to a Human, Parent Portal Help, School Timings |
| **TalkToHumanIntent** | — | `human_escalation` | 32 | Staff Directory, Book Teacher Meeting, Share Feedback, Back to Main Menu |
| **ComplaintFeedbackIntent** | `complaint_category` | `human_escalation` | 52 | Track Complaint Status, Contact Principal, Talk to a Human, Back to Main Menu |
| **IDCardIntent** | — | — | 23 | Contact Accounts Office, Fee Details, Request Documents, Staff Directory |
| **ParentPortalLoginIntent** | — | — | 25 | Contact IT Support, Staff Directory, Talk to a Human, Back to Main Menu |

### Conversation & fallback

| Intent | Asks for (if missing) | Sets context | Phrases | Pills |
|---|---|---|---|---|
| **ChangeChildIntent** | `child_name` | `active_child_state` | 43 | Check Attendance, Report Card, Homework Today, Fee Payment Status, Academic Progress, Exam Schedule |
| **Default Welcome Intent** | — | — | 1 | Check Attendance, Exam Schedule, Report Card, Fee Details, Homework Today, Announcements |
| **Greeting Intent** | — | — | 19 | Check Attendance, Exam Schedule, Report Card, Fee Details, Homework Today, Bus Routes |
| **CapabilitiesIntent** | — | — | 22 | Check Attendance, Exam Schedule, Report Card, Fee Details, Bus Routes, Cafeteria Menu |
| **ThanksGoodbyeIntent** | — | — | 20 | Back to Main Menu, Check Attendance, Report Card, Announcements |
| **Default Fallback Intent** | — | `fallback_stage_1` | 0 | Check Attendance, Exam Schedule, Fee Details, Homework Today, Talk to a Human |
| **Fallback - Stage2 - CategorizedOptions** | — | `fallback_stage_2` | 0 | Academics (Homework, Syllabus, Exams), Fees & Payments, Attendance & Leave, Transport & Facilities, Health & Safety, Talk to a Human |
| **Fallback - Stage3 - EscalateToHuman** | — | `human_escalation` | 0 | Staff Directory, Back to Main Menu, School Timings, Share Feedback |

---

## 5. Entities

`@allergy_type`, `@assignment_status`, `@bus_route`, `@child_name`, `@club_activity`, `@complaint_category`, `@day_of_week`, `@document_type`, `@event_type`, `@exam_type`, `@fee_type`, `@gatepass_reason`, `@grade`, `@grade_term`, `@health_concern`, `@id_card_issue`, `@installment_plan`, `@leave_reason`, `@meal_item`, `@meal_type`, `@meeting_with`, `@payment_method`, `@payment_status`, `@policy_topic`, `@portal_issue_type`, `@scholarship_type`, `@section`, `@staff_designation`, `@subject`, `@subject_code`, `@urgency_level`, `@volunteer_area`, `@weather_status`

New or changed in this version: `@meeting_with` (class teacher, subject teachers, Principal, Counselor), `@bus_route` (now Routes 1 to 8), and `@staff_designation` (added School Nurse and IT Support).

---

## 6. Contexts and the active child

| Context | Set by | Lifespan |
|---|---|---|
| `active_child_state` (`active_child`, `active_grade`) | Every intent that asks for a child | 5 turns (10 for *Change Child*) |
| `ptm_booking` | `PTMSlotBookingIntent` | 5 turns |
| `human_escalation`, `fallback_stage_1`, `fallback_stage_2` | Fallback and escalation intents | 1 turn |

When a reply needs a child, the bot uses the name in your message. If you did not give one, it uses the child from the last answer. If neither exists it asks *"Which child is this for?"*. **Every reply names the child**, so a wrong assumption is easy to spot.

---

## 7. Three-stage fallback

| Stage | Parent sees |
|---|---|
| 1 | A request to rephrase, plus pills: Check Attendance, Exam Schedule, Fee Details, Homework Today, Talk to a Human |
| 2 | Category pills: Academics, Fees & Payments, Attendance & Leave, Transport & Facilities, Health & Safety, Talk to a Human |
| 3 | The school office phone, email and hours, plus Staff Directory, Back to Main Menu, School Timings and Share Feedback |

Any successful match resets the chain.

---

## 8. Setup

1. Open the [Dialogflow ES console](https://dialogflow.cloud.google.com/) and create or open an agent.
2. Zip the **contents** of `dialogflow_agent/` so `agent.json` sits at the root of the zip. A ready-made `ParentConnectAI-agent-import.zip` is provided.
3. Go to ⚙ Settings → Export and Import → **Restore from zip** and upload it. This replaces the whole agent.
4. Go to Integrations → **Dialogflow Messenger**, enable it and paste the embed snippet into a web page.
5. Keep Fulfillment → Webhook **off**.
6. Pills only show in Dialogflow Messenger, not in the console's test panel.

---

## 9. Try it

| Say | Expect |
|---|---|
| `hi` | A greeting with 6 pills |
| `Show exam timetable` | The Mid-Term timetable with times and what to bring; pills include Mid-Term Syllabus and Check Exam Results |
| `Show Aarav's report card` | The full report card: subject marks, total 513/600, 85.5%, rank, attendance and remarks |
| `What are Ananya's marks` | Subject marks, total, percentage, rank, strongest and weakest subjects |
| `Attendance of Kabir` | Days present, absent and late, and the percentage |
| `How much fee is pending for Diya` | The fee status list with ₹22,000 outstanding |
| `Can I pay fees in installments` | The four payment plans with amounts |
| `Where is the bus` | Live-ETA guidance with route times |
| `Book a meeting with the Maths teacher for Aarav` | Asks for the date, then the time, then confirms |
| `I actually want info for my daughter Ananya`, then `Show report card` | Switches the child, then answers for Ananya without asking again |
| `asdf qwer` three times in a row | Fallback stage 1, then category pills, then the contact-the-office message |
| Tap any pill | The matching intent answers, never the fallback |

---

## 10. Editing the sample data

Open **Intents → *intent name* → Responses** in the Dialogflow console and change the text. Keep `$child_name`, `$date` and similar placeholders where the parent's own words should appear, then press **Save**. Truly per-student answers would need a webhook or a school database.

---

## 11. Accuracy checks and honest limits

- **Checked by script:** valid JSON, no duplicate phrases, every annotation matches a parameter, every reply placeholder is a required parameter, and all 223 pills resolve to a trained phrase.
- **Overlap test:** a text classifier trained on the phrases and scored by cross-validation placed **91%** of all phrases in the right intent, up from 78% before this version, and answered **50 of 50** unseen test questions correctly. This is a proxy for Dialogflow's own matching, which was not run here, so test in the console simulator.
- **Sample data only:** every child sees the same marks, fees and attendance. Nothing is stored or sent, so leave, gate-pass and pre-order replies tell the parent what to do next instead of claiming a request was filed.
- **Dates are fixed:** the sample dates will go stale and should be edited each term.
- **Pills show in Dialogflow Messenger only.**

---

## 12. Repository layout

```text
README.md
dialogflow_agent/
  agent.json       agent settings (webhook disabled)
  intents/         49 intents and 1,645 training phrases
  entities/        33 entities
```
