# StudyDash MVP Specification

## Overview

StudyDash is a unified school management platform for Indian schools. This document defines the locked MVP scope, tech stack, and feature specifications.

**Target Users:** Indian K-12 schools (CBSE, ICSE, State Boards)
**User Roles:** Admin, Teacher, Parent, Student, Driver

### Platforms

| Role | Platform |
|------|----------|
| School Admin | Web panel (Next.js) |
| Teacher | Mobile app (React Native) |
| Parent | Mobile app (React Native) |
| Student | Mobile app (React Native) |
| Driver | Mobile app (React Native) — driver mode |

### Tech Stack

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

### Multi-Tenancy

- All data scoped by `schoolId`
- Every API query filters on `schoolId` extracted from JWT
- S3 paths prefixed: `{schoolId}/notes/`, `{schoolId}/receipts/`, `{schoolId}/expenses/`, etc.

---

## Authentication

### Parents & Students — Phone + OTP
- Login with phone number
- Receive SMS OTP via MSG91/Twilio
- OTP verified → JWT access token (15 min) + refresh token (30 days)
- Device token stored for push notifications

### Teachers — Email + Password
- Login with email + password on mobile app
- Password reset via email link
- JWT access token (15 min) + refresh token (30 days)

### Admin — Email + Password
- Login with email + password on web panel
- JWT access token (15 min) + refresh token (7 days — shorter for security)
- Password reset via email link

### Driver — Phone + OTP
- Same flow as parents/students
- Driver role assigned by admin during onboarding

---

## MVP Features

### 1. School Onboarding (Admin Web Panel)

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

**Limitations (documented for users):**
- Accuracy depends on driver's phone GPS (±10-50m)
- Battery drain warning for drivers on long routes
- Stale location shown with timestamp if signal lost > 60 seconds

---

### 10. Expense Tracker (Admin Web Panel)

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

**Expense States:** Simply logged. No workflow. Edit and delete allowed.

---

## Post-MVP Roadmap (in order)

| Phase | Feature | Description |
|-------|---------|-------------|
| v1.5a | Pixel Trace Light | Manual photo tagging by teachers, consent-gated parent/student viewing |
| v1.5b | Pixel Trace AI | Face recognition matching with moderation queue |
| v1.5c | WhatsApp Integration | Notification fallback chain (push → WhatsApp → SMS) |
| v2a | School Email Provisioning | Google Workspace / Microsoft 365 student email management |
| v2b | PTM Scheduler | Parent-teacher meeting slot booking |
| v2c | Library Management | Book catalog, issue/return tracking, fine integration |
| v2d | AI Attendance | Classroom photo → automatic roll call (requires Pixel Trace AI maturity) |
| v2e | Board-Specific Report Cards | CBSE/ICSE/State Board format report card generation |
| v2f | Admission & Enrollment | Online inquiry → application → document upload → enrollment flow |

---

## Key Architecture Decisions

### API Design
- RESTful API with `/api/v1/` prefix
- Role-based route prefixes: `/api/v1/admin/`, `/api/v1/teachers/`, `/api/v1/parents/`, `/api/v1/students/`, `/api/v1/drivers/`
- Standard error format: `{ error: string, code: string, details?: object }`
- Pagination: cursor-based for lists (consistent ordering, no offset drift)
- Rate limiting: 100 requests/min per user, 5 OTP requests/hour per phone number

### Database
- PostgreSQL with Prisma ORM
- All models include `schoolId` foreign key (multi-tenancy)
- Soft deletes for user-facing data (isDeleted flag + deletedAt timestamp)
- Audit log table for sensitive operations (attendance edit, payment record, user deletion)
- Indexes on: `[schoolId, classId]`, `[studentId, date]` (attendance), `[schoolId, category]` (expenses)

### File Storage
- AWS S3 with presigned URLs (upload and download)
- Path convention: `{schoolId}/{feature}/{entityId}/{filename}`
- Max file size: 25MB per upload
- Allowed types: PDF, PNG, JPG, JPEG, DOC, DOCX, XLS, XLSX

### Background Jobs (BullMQ)
- Queues: notifications, pdf-generation, csv-processing, thumbnail-generation
- Retry policy: 3 retries with exponential backoff (1s, 4s, 16s)
- Failed jobs logged to dead letter queue for manual inspection
- Job timeout: 60 seconds (5 minutes for CSV processing)

### Security
- All API calls over HTTPS (TLS 1.2+)
- JWT signed with RS256
- Passwords hashed with bcrypt (12 rounds)
- OTP: 6-digit, expires in 5 minutes, max 5 attempts
- Razorpay webhook signature verification on every callback
- Input validation on all endpoints (express-validator)
- SQL injection prevented by Prisma parameterized queries
- File upload: MIME type validation (not just extension)

---

## Open Items (to resolve during implementation planning)

1. **SMS provider final choice:** MSG91 vs Twilio — cost comparison needed for Indian SMS volume
2. **Hosting platform:** AWS vs GCP vs Railway — decide during infra setup
3. **CI/CD pipeline:** GitHub Actions setup — define during first sprint
4. **Monitoring stack:** Sentry for errors, basic CloudWatch/Railway logs to start
5. **Database backups:** Automated daily backups — configure during deployment setup
