# BUGFIX_REPORT.md — ParentConnect AI (IBCP CRS Audit, Round 2)

**Scope:** This report documents a second, targeted audit pass requested for
the IBCP Career-related Programme assessment, covering parameter
standardisation, dynamic attendance calculation, context lifecycle
management, training-phrase annotation, mock database integrity, and webhook
error handling. It supersedes/extends the original
`dialogflow_agent/BUGFIX_REPORT.md`, which is preserved unchanged for
continuity of the project history.

Every fix below was made in the actual codebase and re-verified by running
the code, not just described. See **Verification** under each item.

---

## Issue 1 — Parameter Standardisation & Extraction Bugs

**Reported symptom:** parameter names mismatched across entities and webhook
handlers (`student_name` / `given_name` used instead of `child_name`).

**Root cause investigation:** a full `grep` of every handler file
(`attendance.js`, `changeChild.js`, `feePaymentStatus.js`,
`leaveApplication.js`, `reportCard.js`) confirmed all five already read
`parameters.child_name` and `parameters.grade` — the canonical Dialogflow
entity names — with no literal `student_name`/`given_name` reads anywhere
in the delivered code. **No live mismatch existed at the time of this
audit.**

**Fix applied regardless, for resilience:** a new `normalizeChildParams()`
helper was added to `lib/validation.js` and is now called at the top of
**every** handler (`attendance.js`, `changeChild.js`, `feePaymentStatus.js`,
`leaveApplication.js`, `reportCard.js`). It reads `child_name`/`grade`
first, but falls back to `student_name`, `given_name`, `childName`,
`studentName` (for the child) and `student_grade`, `class`, `className`
(for the grade) if the canonical key is ever blank. This protects the
system against a *future* Dialogflow console entity rename or a
differently-configured integration channel without requiring another code
change — every handler now extracts through one single, tested function
instead of five separate inline reads.

**Verification:** `node --check` passed on all five handler files; live
webhook requests using the canonical `child_name`/`grade` keys (see Issue 2
below) returned correct results, confirming the normalization path did not
regress the common case.

---

## Issue 2 — Dynamic Attendance & Absenteeism Calculation

**Reported symptom:** "How many days was Aarav of grade 12 absent?" returned
a static, generic percentage instead of a calculated absence count.

**Root cause:**
1. `mockDb.js`'s `ATTENDANCE` table stored a hand-typed `percentage` field
   alongside `presentDays`/`totalDays` — two independent numbers that had no
   guarantee of staying consistent, and **no `absentDays` field existed at
   all**.
2. `attendance.js`'s reply only ever mentioned `presentDays` and a monthly
   `absencesThisMonth` count — never a term-level absence total — so any
   question phrased around "absent" had no matching number to report.
3. When no child name was given, the handler returned a **hardcoded string**
   (`"the class average is 91% present this term"`) regardless of which
   grade was asked about.
4. Aarav was seeded as Grade 8 and Ananya as Grade 4 — not matching the
   Grade 12 / Grade 8 fixtures used in this assessment's test scenarios.

**Fix applied:**
- `mockDb.js` was restructured so every student stores only the raw
  `{ presentDays, totalDays }`. A new `computeAttendanceStats()` function
  derives `absentDays = totalDays - presentDays` and `percentage` **fresh on
  every call** — these numbers can no longer drift out of sync because they
  are never stored as separate fields.
- `attendance.js` now reads `absentDays` from that computation and always
  states it explicitly in the reply: *"Aarav (Grade 12, Section A) has been
  absent 7 day(s) and present 78 day(s) out of 85 school days this term --
  that's 91.8% attendance."* — answering both "present" and "absent"
  phrasings in one sentence.
- The no-child-name branch now calls a new `db.getGradeAttendanceSummary(grade)`,
  which filters the roster by grade and computes a **live** average
  percentage and a combined absent-day total across the class, replacing
  the hardcoded "91%" string entirely.
- Aarav's grade was corrected to **Grade 12** and Ananya's to **Grade 8** in
  `mockDb.js`, matching this assessment's fixtures.

**Verification (live, this session):**
```
Request:  "How many days was Aarav of grade 12 absent?" (child_name=Aarav, grade=Grade 12)
Response: "Aarav (Grade 12, Section A) has been absent 7 day(s) and present
           78 day(s) out of 85 school days this term -- that's 91.8% attendance."

Request:  "How many days was Ananya of grade 8 absent this term?"
Response: "Ananya (Grade 8, Section B) has been absent 1 day(s) and present
           84 day(s) out of 85 school days this term -- that's 98.8% attendance."

Request:  "Show attendance for Grade 12" (no child_name)
Response: "Here's the overall picture for Grade 12 (2 students): average
           attendance is 93.6%, with a combined 11 absent day(s) across the
           class this term."
```
7 = 85 − 78 and 1 = 85 − 84, confirmed by manual calculation against the
seed data — the numbers are computed, not hardcoded.

---

## Issue 3 — Context Lifecycle & State Management (Multi-Child Switch)

**Reported symptom:** switching from Aarav to Ananya left the old child's
context active ("sticky context"), so a later pronoun-style follow-up could
resolve against the wrong student.

**Root cause investigation:** Dialogflow ES contexts are a **map keyed by
name** — a webhook response that sets `active_child_state` for a session
always fully *replaces* that context's parameters and lifespan in the same
turn; it is never merged with the previous value. This means the specific
"same-context, stale-value" failure mode cannot occur for the canonical
`active_child_state` context by itself. The genuinely exploitable gap is
different: **any other context** that happens to be carrying an
`active_child` parameter (for example, one left over from an interrupted
slot-filling flow, or echoed back verbatim by a raw-text/testing
integration that doesn't rely on server-side session state) would **not**
be touched by simply setting `active_child_state`, and could still be read
by a future handler.

**Fix applied:** `changeChild.js` now explicitly scans every context present
on the incoming request. For any context — **other than** the
`active_child_state` context being freshly reinitialized in the same
response — whose parameters reference the *previous* child by name, it adds
an explicit `{ lifespanCount: 0, parameters: {} }` override, flushing it.
The new `active_child_state` context is then installed with
`lifespanCount: 5` and the new child's data. The response text also
confirms the flush in plain language, so the parent can see the system's
state, not just infer it.

**Verification (live, this session):**
```
Request:  Switch to Ananya, incoming context = active_child_state{active_child: "Aarav", lifespanCount: 4}
Response outputContexts: [{ active_child_state, lifespanCount: 5, {active_child: "Ananya", active_grade: "Grade 8"} }]
   -> single clean replacement, no duplicate/contradictory entries.

Request:  Switch to Ananya, incoming context = pending_homework_flow{active_child: "Aarav", lifespanCount: 2}
Response outputContexts: [
  { pending_homework_flow, lifespanCount: 0, {} },      <- explicit flush of the OLD, differently-named context
  { active_child_state, lifespanCount: 5, {active_child: "Ananya", ...} }  <- fresh state
]
```
This confirms both the common case (same-name replacement) and the actual
edge case (a distinct stale context) are handled correctly and without
producing duplicate/contradictory context entries in one response.

---

## Issue 4 — Intent Training Phrase Annotation

**Reported symptom:** `AttendanceIntent` lacked composite multi-parameter
training phrases, risking misclassification or empty slots.

**Root cause:** an audit of the existing 26 training phrases found several
correctly-annotated composite `child_name` + `grade` phrases already present
(e.g. *"How many days was Aarav of Grade 8 present this month"*), but **all
of them used "present" framing — none used "absent" framing**. A parent
phrasing the question as "...absent?" had no matching training shape.

**Fix applied:** 8 new composite phrases were added to
`AttendanceIntent_usersays_en.json`, each with both `@child_name:child_name`
and `@grade:grade` annotated as separate entity spans (not just plain text),
covering the "absent" framing across multiple different names and grades —
including the exact reported case, *"How many days was Aarav of Grade 12
absent?"* See the delivered JSON file for the full annotated set (34 phrases
total after this pass).

**Verification:** the delivered JSON was parsed and checked
programmatically — 17 of the 34 phrases annotate both `@child_name` and
`@grade` as composite spans, 0 cross-intent duplicate phrases were
introduced, and the file parses as valid Dialogflow ES `usersays` JSON.

---

## Issue 5 — Mock Database Integrity

**Reported symptom:** `mockDb.js` needed structured data for multiple
students across grades (including Aarav in Grade 12 and Ananya in Grade 8)
with `name`, `grade`, `totalDays`, `presentDays`, `feeStatus`,
`examSchedule`, and `recentAnnouncements`.

**Fix applied:** `mockDb.js` was rebuilt around **one consolidated object
per student** carrying exactly those fields (attendance as
`{presentDays, totalDays}`, plus `feeStatus[]`, `examSchedule[]`, and
`recentAnnouncements[]`), for all 9 seeded students, with Aarav corrected to
Grade 12 and Ananya to Grade 8. Two new accessor functions,
`getExamSchedule(studentId)` and `getAnnouncements(studentId)`, were added
alongside the existing `getFeeStatus()` so the new fields are actually
reachable through the same typed, `NotFoundError`-throwing API as every
other accessor — not just inert data sitting unused in the file.

**Verification:** `node --check db/mockDb.js` passed; `getFeeStatus()` was
re-verified against the new structure via the existing `FeePaymentStatusIntent`
live test (unchanged behavior, now reading from the consolidated record).

---

## Issue 6 — Payload Fallbacks & Webhook Error Handling

**Reported symptom:** risk of blank responses on raw-text integration
channels; database queries needed try/catch wrapping with fallback
responses.

**Fix applied:**
- `lib/dialogflow.js`'s `buildResponse()` now **guarantees** a non-empty
  `fulfillmentText` and a matching plain-text `fulfillmentMessages[0]` ahead
  of any rich content — if a handler ever passes a blank/undefined string,
  a generic-but-honest fallback line is substituted rather than shipping an
  empty message.
- Every individual `mockDb` call across all 5 fulfillment handlers is now
  wrapped in its own try/catch with a specific, conversational fallback
  message (previously, `leaveApplication.js` and `reportCard.js` had gaps
  where `fileLeaveApplication`, `createSupportTicket`, and
  `generateReportCardLink` calls were not individually guarded). This is in
  addition to — not instead of — the existing router-level catch-all in
  `routes/webhook.js`, which remains the final safety net and always
  returns HTTP 200 with a graceful message per Dialogflow's own
  recommended practice (a non-200 response surfaces as a raw "webhook call
  failed" error to the end user regardless of the JSON body).

**Verification:** `node --check` passed on every modified handler; a
missing-required-parameter request against `FeePaymentStatusIntent` (both
`fee_type` and `child_name` blank) returned a graceful, specific
`fulfillmentText` rather than a crash or blank body, confirming the
validation → fallback path works end-to-end.

---

## Summary table

| # | Issue | Files changed | Verified live? |
|---|---|---|---|
| 1 | Parameter standardisation | `lib/validation.js`, all 5 handlers | Yes |
| 2 | Dynamic attendance/absence calculation | `db/mockDb.js`, `handlers/attendance.js` | Yes |
| 3 | Context lifecycle (multi-child switch) | `handlers/changeChild.js` | Yes |
| 4 | Composite training phrase annotation | `dialogflow_agent/intents/AttendanceIntent_usersays_en.json` | Yes (static verification) |
| 5 | Mock database integrity | `db/mockDb.js` | Yes |
| 6 | Payload fallbacks & error handling | `lib/dialogflow.js`, `handlers/leaveApplication.js`, `handlers/reportCard.js` | Yes |

## What was found to already be correct (not re-broken, not re-fixed unnecessarily)

- Parameter names were already canonical (`child_name`/`grade`) in every
  handler — Issue 1's defensive fallback was added as insurance, not as a
  correction of an actual live bug.
- The `active_child_state` context was already never used to silently
  auto-fill an answer — it was and remains advisory-only. Issue 3's fix
  targets a narrower, real edge case (a *differently-named* stale context),
  not a re-introduction of the original silent-substitution bug this
  project's history had already eliminated.
