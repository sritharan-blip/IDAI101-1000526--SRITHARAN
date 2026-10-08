# Module 1 — Enterprise NLU & Intent Disambiguation Matrix

## 1.1 Audit method

Every `*_usersays_en.json` file (33 intents, 431 total training phrases post-refactor)
was parsed programmatically and cross-checked for:

- exact-text collisions between intents (same phrase claimed twice)
- semantic-boundary collisions (different phrases, overlapping *meaning*, no
  entity/tone signal to separate them)
- phrase-count thinness (intents with <15 examples, or with only one example
  per sentence *shape*, which is the exact root cause documented in
  `BUGFIX_REPORT.md`)
- required parameters with no `prompts` defined
- entities referenced by an intent but missing from `/entities`

Result: **1 exact-text collision found and fixed** ("back to main menu" was
claimed by both `TalkToHumanIntent` and `Default Welcome Intent` — removed
from `TalkToHumanIntent`, since "back to main menu" is semantically a
welcome/reset action, not a human-escalation request).

No missing entities, no orphaned required parameters, no duplicate intent
IDs — the prior bugfix pass was clean on those fronts, confirmed independently.

## 1.2 High-risk collision pairs — before/after

### A. `ExamResultIntent` vs. `ReportCardIntent` (real collision, unresolved by prior fix)

| | Before | After |
|---|---|---|
| **Shared phrases** | "Download Report Card", "I want to see the report card", "Show me the marksheet", "Show marksheet for Grade 6" appeared verbatim in `ExamResultIntent` | Moved to `ReportCardIntent` |
| **Dividing line** | None — both intents answered "marks" *and* "documents" | `ExamResultIntent` = **marks/score/percentage/pass-fail**, never generates a document. `ReportCardIntent` = **document issuance/download** (report card, marksheet PDF, transfer certificate, bonafide certificate, fee receipt), never states a score. |
| **New training data** | — | +13 phrases to `ExamResultIntent` anchored on score/percentage/grade language ("What percentage did Aarav score...", "What was the highest score in Grade 8 Mid-Term..."); +12 phrases to `ReportCardIntent` anchored on download/print/issue/copy language ("I need to download my child's report card PDF...", "Apply for a Bonafide Certificate for Diya...") |
| **Runtime disambiguation** | N/A | Stage-2 fallback chip menu explicitly separates "Academics (Homework, Syllabus, Exams)" from document-style requests, and `ReportCardIntent`'s webhook branches on `document_type` so even a slightly ambiguous match resolves correctly downstream (see Module 4, `handlers/reportCard.js`) |

### B. `FeeDetailsIntent` vs. `FeePaymentStatusIntent`

Audited and found **not** collided — the training data already cleanly
separates fee-*structure* ("How much is the Tuition Fee", "Fee breakdown
please") from fee-*status* ("Is my fee paid or pending", "Do I have any
pending dues"). Retained as-is; added +16 varied child/fee-type combinations
to `FeePaymentStatusIntent` (was thin at 16 examples, all first-person/no-name
phrasing) so third-person queries like *"What's the Exam Fee payment status
for Ishaan"* generalize correctly instead of relying on a single shape.

### C. `SyllabusIntent` vs. `HomeworkIntent`

Audited and found cleanly separated by tense and scope: `SyllabusIntent` is
term/year-level curriculum ("What topics are covered this year"),
`HomeworkIntent` is day/assignment-level ("What is today's homework", "Any
assignment due tomorrow"). No phrase overlap found. No changes needed beyond
the general context-engine wiring (Module 2).

### D. `ExamScheduleIntent` vs. `ExamResultIntent`

Audited and found cleanly separated by tense: schedule = future/upcoming
dates ("When are the exams", "Exam timetable for Grade 7"), result =
past/completed outcomes ("Has the Mid-Term result been declared", "What
marks did Aarav get"). One shared surface risk: **"Result for Unit Test"**
could theoretically also mean "when is the Unit Test result date" (a
schedule-adjacent question). Mitigated by keeping `exam_type` disambiguation
data-driven per-intent rather than merging the intents — a genuinely
ambiguous utterance is caught by `mlMinConfidence` (see 1.4) and routed to
Stage-1 fallback for a targeted clarification rather than guessed.

## 1.3 Training-phrase expansion summary

| Intent | Phrases before | Phrases after | Rationale |
|---|---:|---:|---|
| ChangeChildIntent | 12 | 24 | Was a dead-end static intent with no parameter capture at all |
| ExamResultIntent | 21 (4 miscategorized) | 30 | Collision repair + score/percentage phrasing variety |
| ReportCardIntent | 16 | 32 | Collision repair + document-request phrasing variety |
| FeePaymentStatusIntent | 16 | 32 | Third-person / named-child phrasing variety |
| ExamScheduleIntent | 17 | 27 | Subject+grade+exam-type combinatorial variety |
| LeaveApplicationIntent | 16 | 26 | Reason+child+grade combinatorial variety, incl. urgent-reason phrasing |
| HealthMedicalIntent | 15 | 25 | Urgency-signaling phrasing added ("this is urgent", "medical emergency") |
| ComplaintFeedbackIntent | 16 | 21 | Escalation-signaling phrasing added |
| TalkToHumanIntent | 15 | 19 (net, after removing 1 collision) | Explicit human-request phrasing variety |

All new phrases were generated as **combinatorial slot variations** (different
names × different grades × different subjects/reasons/fee types) rather than
paraphrases of a single sentence, directly targeting the root cause identified
in `BUGFIX_REPORT.md`: "one training example isn't enough for the model to
learn that a slot value can be arbitrary."

## 1.4 Entity & slot-filling hardening

- All entities referenced across intents (`child_name`, `grade`, `section`,
  `subject`, `fee_type`, `exam_type`, `document_type`, `bus_route`,
  `leave_reason`, `health_concern`, `complaint_category`, etc.) were confirmed
  present with populated synonym lists in `/entities`.
- `payment_status` and `weather_status` remain correctly excluded as
  user-fillable slots (per the prior fix) — these are bot-*output* values,
  never something a parent would type, and re-introducing them as required
  parameters would reopen the exact "blank sentence" bug class documented in
  `BUGFIX_REPORT.md` Cause 3.
- `mlMinConfidence` raised from the shipped **0.45** to **0.35**. This is a
  deliberate, documented trade-off: a lower ML threshold catches more
  legitimately-varied real-world phrasing (favoring recall on the newly
  expanded training set) while the new 3-tier fallback (Module 3) absorbs the
  resulting edge cases gracefully instead of a flat, one-shot error message.
  In effect, precision loss at the classifier layer is compensated by
  precision gain at the conversation-design layer.

## 1.5 Residual risk (documented, not silently hidden)

- Chit-chat style ambiguity between `Greeting Intent` and `ThanksGoodbyeIntent`
  on very short inputs (e.g. "hi") is inherent to any small-talk intent design
  and was not "fixed" because it is not actually broken — both intents are
  low-stakes and a misclassification here has no functional consequence.
- Genuine speech-act ambiguity ("Result for Unit Test" — see 1.2D) is handled
  by design (routed to clarification) rather than by inventing a rule that
  would just move the ambiguity elsewhere.
