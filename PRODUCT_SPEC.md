# StudyDash — Product Specification

## Overview

StudyDash is a unified school management platform for Indian schools serving five user roles: **School Administration**, **Teachers**, **Parents**, **Students**, and **Drivers**. The platform consists of a mobile app (React Native) for Teachers, Parents, Students, and Drivers, and a web admin panel (Next.js) for School Administration. The core differentiator is Pixel Trace — an AI-powered face recognition platform for event photo discovery.

---

## Release Roadmap

| Phase | Scope | Rationale |
|-------|-------|-----------|
| **MVP** | Auth, School Onboarding, Digital Attendance, Homework & Class Notes, Timetable, Fees Payment, Communication Channel, Leave Applications, Push Notifications, GPS Bus Tracking, Expense Tracker | Core pain points, immediate adoption value |
| **v1.5a** | Pixel Trace Light (manual photo tagging) | Differentiator, builds photo library and consent base |
| **v1.5b** | Pixel Trace AI (face recognition matching) | Low-stakes AI deployment to train and validate face model |
| **v1.5c** | WhatsApp Integration | Notification fallback chain for reliable parent communication |
| **v2a** | School Email Provisioning | Google Workspace / Microsoft 365 student email management |
| **v2b** | PTM Scheduler | Parent-teacher meeting slot booking |
| **v2c** | Library Management | Book catalog, issue/return tracking, fine integration |
| **v2d** | AI Attendance (class photo → auto roll call) | Only after face model is battle-tested through Pixel Trace |
| **v2e** | Board-Specific Report Cards | CBSE/ICSE/State Board format report card generation |
| **v2f** | Admission & Enrollment Management | Online inquiry → application → enrollment flow |
| **v3** | Advanced analytics, SIS integrations, multi-school chain dashboards | Build on demand |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Node.js + TypeScript + Express |
| ORM / Database | Prisma + PostgreSQL |
| Mobile App | React Native (Expo) |
| Web Admin Panel | Next.js + React |
| File Storage | AWS S3 (presigned URLs) |
| Job Queue | BullMQ + Redis |
| Payments | Razorpay |
| SMS / OTP | MSG91 or Twilio |
| Push Notifications | Firebase Cloud Messaging (FCM) + APNs |
| PDF Generation | Puppeteer |

---

## Platforms

| Role | Platform |
|------|----------|
| School Admin | Web panel (Next.js) |
| Teacher | Mobile app (React Native) |
| Parent | Mobile app (React Native) |
| Student | Mobile app (React Native) |
| Driver | Mobile app (React Native) — driver mode |

---

## Authentication

### Parents, Students & Drivers — Phone + OTP
- Login with phone number → receive SMS OTP → verify → JWT issued
- Access token: 15 min expiry
- Refresh token: 30 days, stored hashed

### Teachers — Email + Password
- Login with email + password on mobile app
- Password reset via email link
- Access token: 15 min expiry
- Refresh token: 30 days

### Admin — Email + Password
- Login with email + password on web panel
- Access token: 15 min expiry
- Refresh token: 7 days (shorter for security)
- Password reset via email link

---

## Multi-Tenancy

- All data scoped by `schoolId`
- Every API query filters on `schoolId` extracted from JWT
- S3 paths prefixed: `{schoolId}/{feature}/{entityId}/{filename}`

---

## MVP Features

### 1. School Onboarding (Admin Web Panel)

**Problem:** Setting up each school manually doesn't scale. Schools need self-service onboarding.

**Setup Wizard:**
- Step 1: School profile (name, address, board affiliation, logo)
- Step 2: Academic year and terms
- Step 3: Classes and sections (e.g., Class 5 → Section A, B, C)
- Step 4: Subjects per class
- Step 5: Fee heads (Tuition, Transport, Lab, etc.) with amounts and installment due dates
- Step 6: Bus routes and stops

**CSV Bulk Import:**
- Students CSV: name, class, section, parent name, parent phone, address
- Teachers CSV: name, email, department, assigned classes/subjects
- Drivers CSV: name, phone, assigned route
- Validation with row-level error reporting (which rows failed, why)
- Auto-generates parent accounts linked by phone number
- Auto-generates student/teacher/driver accounts

**Key Rules:**
- Duplicate phone number detection (same parent with multiple children → link, don't duplicate)
- Re-upload CSV overwrites only new/changed rows (upsert by phone number or employee ID)

---

### 2. Digital Attendance

**Problem:** Manual roll call consumes 5-10 minutes per period. Paper registers are error-prone and hard to digitize.

**Teacher (Mobile):**
- Select class/section → see student list
- Mark each student: Present, Absent, Late
- Submit attendance (one submission per class per day)
- Edit window: can modify within same day until midnight

**Student (Mobile):**
- View today's attendance status
- Monthly calendar view (color-coded: green=present, red=absent, yellow=late, blue=on leave)
- Attendance percentage summary

**Parent (Mobile):**
- Same view as student, per child (multi-child switcher)
- Receives push notification on child marked absent

**Admin (Web):**
- Attendance compliance report: which teachers haven't marked attendance today
- Class-wise attendance summary
- Can override/correct attendance records

---

### 3. Homework & Class Notes

**Problem:** Homework communication is unreliable (verbal, WhatsApp). Notes get lost.

**Teacher (Mobile):**
- Create assignment: title, description, due date, class/section, subject
- Attach files (PDF, images, docs) via S3 presigned upload
- View completion stats (X of Y students marked complete)
- Upload class notes: title, file, class/section, subject

**Student (Mobile):**
- View assignments for their class, sorted by due date
- Mark assignment as complete (toggle)
- View and download class notes
- Bookmark notes for quick access

**Parent (Mobile):**
- Read-only view of child's assignments (title, description, due date, completion status)
- Read-only view of class notes
- Push notification when new assignment is posted

---

### 4. Timetable

**Problem:** Students/parents don't have easy access to class schedules.

**Admin (Web):**
- Configure timetable per class/section: periods, subjects, teachers, time slots
- Set period duration, break times, lunch

**Teacher (Mobile):**
- View their teaching schedule across all classes
- Today view + weekly view

**Student (Mobile):**
- View class timetable: today view + weekly view

**Parent (Mobile):**
- View child's timetable (read-only)

---

### 5. Fees Payment

**Problem:** Fee collection is manual, tracking defaulters is tedious, parents want digital payment options.

**Admin (Web):**
- Create fee structures per class (fee heads + amounts + due dates)
- Generate invoices per student (auto or manual)
- Record offline payments (cash, cheque, bank transfer) with reference number
- View payment status per student, per class
- Fee defaulter report (overdue > configurable days)
- Download collection reports (CSV/PDF)

**Parent (Mobile):**
- View outstanding fees per child (itemized by fee head)
- Pay online via Razorpay (full or partial payment)
- View payment history and download receipts (PDF)
- Push notification for upcoming due dates and overdue reminders

**Payment Flow:**
- Parent initiates payment → Razorpay checkout → webhook confirms success/failure
- Idempotency key per transaction (Razorpay order ID) — prevents duplicate processing
- Partial payments: track amount paid per fee head, update invoice status (PENDING → PARTIAL → PAID)
- Receipt auto-generated as PDF on successful payment (BullMQ job)
- Reconciliation: daily batch job compares Razorpay settlements with local records, flags mismatches

**Fee States:** PENDING → PARTIAL → PAID → OVERDUE (auto-set by daily cron if past due date and not fully paid)

---

### 6. Communication Channel

**Problem:** Teacher-parent communication happens over unstructured WhatsApp groups — no accountability, no records.

**Teacher (Mobile):**
- Send class-wide announcements (text + optional attachment)
- View and respond to parent messages about their class students

**Parent (Mobile):**
- View class announcements from teacher
- Submit complaint/query to school (ticket-based)
- Track ticket status (Open → In Progress → Resolved)
- View message history

**Admin (Web):**
- Send school-wide broadcasts (all parents, all teachers, or specific classes)
- View and manage complaint tickets (assign, respond, resolve)
- Ticket dashboard with status filters

**Message Types:**
- Announcement: one-to-many, no reply expected
- Ticket: parent → admin, tracked with status, supports back-and-forth thread

---

### 7. Leave Applications

**Problem:** Leave requests are informal (phone calls, notes) with no tracking.

**Parent (Mobile):**
- Submit leave request for child: date range, reason, type (Sick, Personal, Family, Other)
- View leave request history and status
- Push notification on approval/rejection

**Teacher (Mobile):**
- View pending leave requests for their classes
- Approve or reject with optional remarks
- View leave history per student

**Integration with Attendance:**
- Approved leave → auto-upserts AttendanceRecord as ON_LEAVE for each date in range
- Conflict check: if attendance already marked PRESENT for a leave date, flag for teacher review instead of auto-overwrite

**Leave States:** PENDING → APPROVED | REJECTED

---

### 8. Push Notifications

**Triggers:**

| Event | Recipients |
|-------|-----------|
| Student marked absent | Parent of that student |
| New assignment posted | Parents + students of that class |
| Fee due in 7 days | Parent |
| Fee overdue | Parent |
| Payment successful | Parent |
| Leave request status change | Parent |
| New announcement | Target audience (class/school) |
| Bus started route | Parents of students on that route |
| Bus near stop | Parent of students at that stop |

**Implementation:**
- FCM for Android, APNs for iOS (via Firebase)
- Device tokens stored per user (support multiple devices)
- BullMQ job per notification event → fans out to all recipient device tokens
- In-app notification history (stored in DB, paginated)
- Notification states: CREATED → DELIVERED → READ

---

### 9. GPS Bus Tracking

**Problem:** Parents call the school office dozens of times daily asking about bus location.

**Driver (Mobile):**
- Start/end route button
- App sends GPS coordinates every 10 seconds while route is active
- Background location tracking (React Native background geolocation)
- Shows current route and upcoming stops

**Parent (Mobile):**
- Map view showing bus location (Google Maps SDK)
- ETA to child's stop (distance ÷ average speed)
- Push notification: "Bus started route" and "Bus approaching your stop" (within 500m radius)
- Last known location with timestamp if GPS signal lost

**Admin (Web):**
- Define routes: ordered list of stops with names and coordinates
- Assign drivers and buses to routes
- Assign students to stops
- View all active buses on a map (live dashboard)
- Route history log

**Data Model:**
- Route → ordered Stops → assigned Students
- LiveLocation: { driverId, routeId, lat, lng, speed, timestamp } — overwritten per ping, history kept for today only
- Geofence per stop: 500m radius triggers "approaching" notification

**Key Risks:**
- Depends on driver actually running the app (human reliability)
- Phone battery drain during long routes
- GPS accuracy varies (±10-50m in cities, worse in rural areas)
- If driver's phone dies or loses signal, location goes stale — show "last known location" + timestamp

---

### 10. Expense Tracker (Admin Web Panel)

**Problem:** Schools have no visibility into where operational money goes.

**Features:**
- Log expense: amount, category, date, description, optional receipt upload (image/PDF to S3)
- Predefined categories: Salaries, Maintenance, Utilities, Events, Supplies, Transport, Other
- Admin can add custom categories
- Expense list view: table with filters (date range, category), sortable by date/amount
- Monthly summary: total spend, category-wise breakdown (pie chart)
- Year-over-year comparison: this month vs. last month total

**What it is NOT:**
- No budgeting or forecasting
- No approval workflows — admin logs it, done
- No integration with fee income (future analytics feature)
- No double-entry bookkeeping

---

## Post-MVP Features

### Pixel Trace (Event Photo Discovery) — USP

**Problem:** Schools photograph students at events but parents never receive these photos. Thousands sit unused.

**Phase 1 — Pixel Trace Light (v1.5a):**
- School uploads event photos to S3 (organized by event)
- Teachers manually tag students in photos
- Parents/students view photos their child is tagged in (consent-gated)
- Consent gate: parent opts in/out, data deleted on revocation

**Phase 2 — Pixel Trace AI (v1.5b):**
- Face embeddings generated from student ID photos
- Automatic matching of event photos to students
- Confidence threshold for auto-tagging; low-confidence matches go to moderation queue
- Teacher/admin reviews and confirms low-confidence matches
- Model improves over time with feedback

**Privacy & Compliance:**
- Opt-in only — parent signs digital consent form
- Face embeddings (vectors) stored, not raw biometric templates
- DPDP Act (India) compliance — data minimization, purpose limitation, right to erasure
- All face data deleted if student leaves school or parent revokes consent

**Strategic Importance:** Pixel Trace is the foundation of StudyDash's AI moat. The face model trained here directly powers the future AI Attendance system.

---

### WhatsApp Integration (v1.5c)

- Automated WhatsApp messages for: absence, fee reminders, homework, urgent announcements
- WhatsApp Business API via Gupshup or Interakt (India-focused, DLT-registered)
- Fallback chain: push notification → WhatsApp → SMS
- TRAI-compliant pre-approved message templates
- Admin configures which events trigger WhatsApp messages

---

### School Email Provisioning (v2a)

- School-issued emails (student@schoolname.edu.in)
- Google Workspace for Education / Microsoft 365 integration
- Managed through StudyDash admin panel
- Enables student benefits (Google Workspace, Microsoft 365, student discounts)

---

### PTM Scheduler (v2b)

- Admin creates PTM events with time slots per teacher
- Parents book slots online (first-come-first-served)
- Teacher sees full appointment schedule
- Automated reminders (1 day and 1 hour before)
- Post-PTM teacher notes per student, visible to parent

---

### Library Management (v2c)

- Book catalog with ISBN/barcode lookup
- Issue/return tracking per student
- Due date reminders and overdue fine calculation
- Digital resources section (NCERT PDFs, sample papers)
- Book availability visible to students/parents

---

### AI Attendance (v2d)

- Teacher takes one classroom photo → system auto-marks attendance
- Uses Pixel Trace face model, further trained on classroom conditions
- Teacher verification mandatory before submission
- Prerequisites: Pixel Trace live 2-3 months, >95% accuracy on real classroom photos

---

### Board-Specific Report Cards (v2e)

- Configurable grading: CBSE (9-point), ICSE (percentage), State Board (marks)
- Exam weightage configuration per board
- Co-scholastic area tracking (CBSE mandatory)
- Board-specific PDF report card templates via Puppeteer
- UDISE+ data export

---

### Admission & Enrollment (v2f)

- Online admission inquiry with document upload
- Lead tracking: inquiry → applied → interview → admitted → rejected
- Waitlist management with auto-notification
- TC generation for outgoing students

---

## User Roles & Access Matrix (MVP)

| Feature | Admin | Teacher | Parent | Student | Driver |
|---------|-------|---------|--------|---------|--------|
| School Onboarding | Full access | — | — | — | — |
| Attendance | View reports, override | Mark/Edit | View own child | View own | — |
| Homework & Notes | — | Create/Upload | View (read-only) | View/Complete | — |
| Timetable | Configure | View own | View child's | View own | — |
| Fees Payment | Configure, track, record offline | — | Pay & view | View | — |
| Communication | Manage tickets, broadcast | Announce, respond | View, submit tickets | View | — |
| Leave Applications | — | Approve/Reject | Apply for child | — | — |
| Notifications | Configure triggers | Receive | Receive | Receive | Receive |
| GPS Bus Tracking | Configure routes, live dashboard | — | Track bus | Track bus | Stream location |
| Expense Tracker | Full access | — | — | — | — |

---

## API Design

- RESTful API with `/api/v1/` prefix
- Role-based route prefixes: `/api/v1/admin/`, `/api/v1/teachers/`, `/api/v1/parents/`, `/api/v1/students/`, `/api/v1/drivers/`
- Standard error format: `{ error: string, code: string, details?: object }`
- Pagination: cursor-based for lists
- Rate limiting: 100 requests/min per user, 5 OTP requests/hour per phone number

---

## Security

- All API calls over HTTPS (TLS 1.2+)
- JWT signed with RS256
- Passwords hashed with bcrypt (12 rounds)
- OTP: 6-digit, expires in 5 minutes, max 5 attempts
- Razorpay webhook signature verification on every callback
- Input validation on all endpoints (express-validator)
- SQL injection prevented by Prisma parameterized queries
- File upload: MIME type validation (not just extension)
- Max file size: 25MB per upload

---

## Background Jobs (BullMQ)

| Queue | Purpose |
|-------|---------|
| notifications | Fan-out push notifications to device tokens |
| pdf-generation | Fee receipts, report cards |
| csv-processing | Bulk import validation and account creation |
| thumbnail-generation | Note/photo thumbnails |

- Retry policy: 3 retries with exponential backoff (1s, 4s, 16s)
- Failed jobs logged to dead letter queue
- Job timeout: 60 seconds (5 minutes for CSV processing)
