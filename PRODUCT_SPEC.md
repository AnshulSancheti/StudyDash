# StudyDash — Product Specification

## Overview

StudyDash is a unified school management dashboard serving three user types: **School Administration**, **Teachers**, and **Parents/Students**. The core differentiator is an AI-powered face recognition platform (Pixel Trace) that enables event photo discovery and, once mature, automated classroom attendance.

---

## Release Roadmap

| Phase | Scope | Rationale |
|-------|-------|-----------|
| **v1 — Launch** | Bus GPS Tracking, Digital Attendance (manual), Fees Payment, Communication Channel, Timetable & Calendar | Core pain points, immediate adoption value |
| **v1.5** | Pixel Trace (event photo matching) | Differentiator, low-stakes AI deployment to train and validate the face model |
| **v2** | AI Attendance (class photo → auto roll call) | Only after face model is battle-tested through Pixel Trace |
| **v3 / On Demand** | Smart Notes (smartboard capture / student photo upload) | Build only if schools actively request it |

---

## 1. GPS Bus Tracking

### Problem
Parents call the school office dozens of times daily asking about bus location. Students wait at stops without knowing if the bus is delayed.

### Features
- Real-time GPS location of each school bus on a map
- Estimated arrival time at each stop
- Push notifications to parents: "Bus is 5 minutes away from your stop"
- Notification when child boards/alights (if paired with RFID or manual check-in)
- Route history and delay logs for school admin

### Implementation
- GPS hardware unit installed in each bus (OBD-II tracker or dedicated device)
- Driver app (minimal UI) as fallback if hardware isn't feasible
- Backend ingests location pings every 10-15 seconds
- ETA calculated using route geometry + real-time position
- Parent-facing map view in the dashboard/app

### Key Risks
- GPS signal loss in underground/covered areas — handle gracefully with "last known location" + timestamp
- Driver phone battery/data — hardware tracker is more reliable than a driver app
- Bus route changes (field trips, diversions) — admin must be able to update routes easily

---

## 2. Attendance (Student)

### Problem
Manual roll call consumes 5-10 minutes per period. Paper registers are error-prone and hard to digitize for reporting.

### Features
- Teacher marks attendance digitally per period
- Instant parent notification on student absence
- Daily/weekly/monthly attendance reports for admin
- Minimum attendance threshold alerts (e.g., below 75%)
- Late arrival tracking

### Implementation (v1 — Manual Digital)
- Teacher opens class roster in app, taps present/absent per student
- Default all students to "present" — teacher only marks absentees (faster flow)
- Data syncs to central dashboard in real-time
- Automated SMS/push notification to parent on absence

### Implementation (v2 — AI-Powered)
- Teacher takes a single photo of the classroom
- AI model identifies visible students using face recognition (trained via Pixel Trace data)
- System auto-marks identified students as present, generates absentee list
- Teacher verbally verifies absentee list before confirming (mandatory verification step)
- See Section 8 (Pixel Trace) for model details and privacy considerations

### Key Risks (AI Attendance)
- False negatives (marking present student as absent) trigger parent panic — verification step is non-negotiable
- Classroom conditions are hard for CV: occlusion, similar uniforms, poor lighting, side angles
- Must be framed as "smart attendance assist" not "facial recognition surveillance"
- Ship only after Pixel Trace model achieves high reliability in real school conditions

---

## 3. Fees Payment

### Problem
Fee collection is manual, tracking defaulters is tedious, and parents want digital payment options.

### Features
- Online fee payment (UPI, net banking, cards)
- Fee structure setup by admin (tuition, transport, lab, etc.)
- Installment plans and partial payment support
- Automated reminders for upcoming/overdue fees
- Receipt generation (PDF)
- Defaulter reports for admin
- Fee concession/scholarship tracking

### Implementation
- Integrate a payment gateway (Razorpay/Cashfree — both have education-sector pricing)
- Admin dashboard for fee structure configuration per class/section
- Parent view: outstanding balance, payment history, pay now button
- Webhook-based confirmation — update ledger on successful payment
- End-of-year fee reports exportable as CSV/Excel

### Key Risks
- Payment failures and reconciliation — need robust retry and manual override
- Refund handling must be clear and auditable
- PCI-DSS compliance handled by the payment gateway, but data storage must follow best practices

---

## 4. Communication Channel

### Problem
Teacher-parent communication happens over unstructured WhatsApp groups — no accountability, no records, important messages get buried.

### Features

**Teacher → Parent (Complaints/Updates)**
- Teacher can send a direct message to a specific parent about their child
- Categorized messages: academic concern, behavioral, appreciation, general update
- Read receipts so teacher knows parent has seen the message
- Broadcast messages to entire class (e.g., "bring art supplies tomorrow")

**Parent → School (Issue Raising)**
- Structured complaint/request form (category, description, attachments)
- Ticket-style tracking: submitted → acknowledged → resolved
- Escalation path: teacher → coordinator → principal
- Resolution history visible to parent

### Implementation
- In-app messaging system (not email/SMS dependent, but SMS fallback for critical alerts)
- Push notifications for new messages
- Admin panel to view all open issues, response times, resolution rates
- Message templates for common communications (fee reminder, PTM invite, etc.)

### Key Risks
- Teachers feel surveilled if admin can read all messages — define clear visibility rules
- Parents may overuse the complaint system — add rate limiting or categorization to prevent noise
- Must NOT become another WhatsApp — keep it structured and purposeful

---

## 5. Daily Timetable

### Features
- Class-wise timetable visible to students, parents, and teachers
- Teacher-wise timetable (their schedule across classes)
- Period substitution reflected in real-time (linked to substitute teacher module)
- Push notification if timetable changes mid-day

### Implementation
- Admin configures master timetable at start of term
- Substitution engine: when a teacher marks leave, system suggests available substitutes and updates timetable on confirmation
- Calendar view and list view options

---

## 6. Homework & Class Notes

### Features
- Teacher posts homework per subject per class with due date
- Homework visible to both student and parent
- Optional: student marks homework as completed (self-reported)
- Class notes attached to each period (photo or file upload)

### Implementation (Homework)
- Simple form: subject, description, due date, optional file attachment
- Push notification to parents when new homework is posted

### Implementation (Notes — v3, build only on demand)
- If school has smartboards: integrate with smartboard software's export/save function to auto-upload session files
- If no smartboard: designated student photographs notes after class, uploads via app
- Notes tagged to subject + date + class for easy retrieval

### Key Risks (Notes)
- Notes feature depends on human discipline (student uploading) — unreliable without incentive structure
- Smartboard integration varies wildly by vendor — scope this per school
- Deprioritize unless schools explicitly request it

---

## 7. Academic Records (Marks & Reports)

### Features
- Exam-wise marks entry by teacher
- Auto-calculated totals, percentages, grades (configurable grading system)
- Report card generation (PDF, printable)
- Historical performance trends visible to parents
- Class-wide analytics for teachers (average, distribution, toppers)

### Implementation
- Admin defines exam schedule and weightage (unit test 20%, mid-term 30%, final 50%, etc.)
- Teacher enters marks via simple spreadsheet-style grid
- System computes derived metrics
- Parent view: subject-wise breakdown, rank (if school policy allows), trend graph

---

## 8. Pixel Trace (Event Photo Discovery) — USP

### Problem
Schools photograph students at events (sports day, annual day, field trips) but students/parents never receive these photos. Thousands of photos sit unused in school archives.

### Features
- Parent/student searches by name → gets all event photos containing that student
- Browse by event (Sports Day 2026, Annual Function, etc.)
- Download individual photos or full event albums
- Strictly **opt-in**: requires explicit parent consent before student's face is enrolled
- Consent can be revoked at any time — student's face data is deleted on revocation

### Implementation
- **Photo Storage**: School uploads event photos to a designated Google Drive folder (organized by event → date)
- **Face Enrollment**: Student's ID card photo is used as the base reference image. System extracts face embedding and stores it (not the raw biometric — only the mathematical vector)
- **Matching Pipeline**:
  1. On photo upload, system runs face detection on each image
  2. Detected faces are converted to embeddings
  3. Embeddings are compared against enrolled student database
  4. Matches above confidence threshold are tagged
  5. Low-confidence matches are flagged for manual review (not auto-tagged)
- **Access**: Parent/student logs in → searches by name → sees matched photos grouped by event
- **Re-enrollment**: ID photos should be refreshed annually (or when student appearance changes significantly) to maintain accuracy

### Privacy & Compliance
- Opt-in only — parent signs digital consent form before enrollment
- Face embeddings (vectors) stored, not raw biometric templates
- Comply with DPDP Act (India) — data minimization, purpose limitation, right to erasure
- If targeting international markets: COPPA (US), GDPR (EU) considerations for minors' data
- Clear privacy policy explaining what data is collected, how it's used, and how to opt out
- All face data deleted if student leaves school or parent revokes consent

### Key Risks
- ID card photos vs. real-world conditions: lighting, angles, aging, accessories — accuracy will vary
- "School uses facial recognition on children" headlines — proactive PR and transparent communication with parents is critical
- Google Drive API rate limits on large photo batches — implement queued processing
- Storage costs scale with number of events and photos — monitor and set upload guidelines

### Strategic Importance
Pixel Trace is not just a feature — it is the **foundation of StudyDash's AI moat**. The face model trained and validated here directly powers the v2 AI Attendance system. Every event processed improves the model's accuracy in real school conditions.

---

## 9. Yearly Calendar

### Features
- School-wide calendar showing holidays, exams, PTM dates, events
- Filterable by category (holiday, exam, event, deadline)
- Sync with device calendar (iCal export)
- Push notifications for upcoming events (configurable: 1 day before, 1 week before)

### Implementation
- Admin manages calendar via dashboard
- Recurring events support (e.g., weekly assembly every Monday)
- Calendar integrates with exam module (Section 7) and timetable (Section 5)

---

## 10. Leave Applications

### Features

**Student Leave**
- Parent submits leave request (date range, reason, category: sick/personal/family)
- Class teacher approves/rejects with optional comment
- Approved leave auto-updates attendance records (marked as "on leave" not "absent")

**Teacher Leave**
- Teacher submits leave request to admin/coordinator
- Leave balance tracking (casual, sick, earned leave)
- On approval, triggers substitute teacher assignment flow (linked to timetable)

### Implementation
- Approval workflow: submit → pending → approved/rejected
- Push notifications at each stage
- Leave history and balance visible to applicant

---

## 11. Teacher Management (Admin-Only)

### Features
- Teacher attendance tracking (separate from student attendance)
- Salary management: structure, payslips, payment history
- Substitute teacher pool and assignment
- Teacher performance data (attendance rate, communication responsiveness — optional)

### Implementation
- Teacher checks in via app (or biometric integration if school has existing hardware)
- Salary module: admin sets structure, system generates monthly payslips
- Substitute flow: teacher marks leave → system shows available teachers for that subject/period → admin confirms → timetable updates automatically

---

## 12. AI Attendance (v2 — Future)

### Problem
Even digital manual attendance takes time. With 6-8 periods and 30+ sections, cumulative time lost is significant.

### Features
- Teacher takes one photo of the classroom
- System identifies students and auto-marks attendance
- Absentee list generated for teacher verification
- Teacher confirms or corrects before submission

### Implementation
- Uses the same face recognition model as Pixel Trace, further trained on classroom conditions
- Handles partial visibility (students behind others) by marking as "unverified" rather than "absent"
- Requires minimum photo quality checks before processing (blur detection, lighting check)
- Runs on server-side — not on teacher's phone (avoids device capability issues)
- Model improves over time with correction feedback (teacher fixes → retraining data)

### Prerequisites Before Launch
- Pixel Trace must be live for at least 2-3 months with measurable accuracy metrics
- Face model must achieve >95% accuracy on real classroom photos in pilot schools
- Privacy impact assessment completed
- Parent communication and consent updated to cover attendance use case

---

## User Roles & Access Matrix

| Feature | Admin | Teacher | Parent | Student |
|---------|-------|---------|--------|---------|
| Bus GPS Tracking | Configure routes | — | Track bus | Track bus |
| Student Attendance | View reports | Mark/Edit | View own child | View own |
| Fees Payment | Configure & track | — | Pay & view | View |
| Communication | Monitor all | Send/Receive | Send/Receive | View |
| Timetable | Configure | View own | View child's | View own |
| Homework & Notes | — | Post | View | View |
| Marks & Reports | Configure exams | Enter marks | View child's | View own |
| Pixel Trace | Upload photos | — | View/Download (opt-in) | View/Download |
| Calendar | Manage | View | View | View |
| Leave Applications | Approve (teacher) | Apply/Approve (student) | Apply (child) | — |
| Teacher Management | Full access | View own salary/attendance | — | — |
| AI Attendance (v2) | Configure | Use | View own child | View own |

---

