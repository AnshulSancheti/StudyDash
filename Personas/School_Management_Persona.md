# School Management (Admin) Persona: Features & Technical Implementation

## Persona Overview
**The School Management** (Principals, Admin Staff, Fee Collectors) requires a top-down view of the entire institution. Their primary concerns are compliance, communication, data accuracy, fee collection, and monitoring teacher/student performance. They interact with the system mostly through a **Desktop Web Portal** (React) rather than a mobile app, as they deal with bulk data, Excel sheets, and complex reporting.

---

## 1. Authentication & Access Control

### Features
- **Email + Password Login:** Admin accounts use email and password (not OTP — more secure for privileged access).
- **Role Variants:** A single Admin user can have sub-roles: Principal, Fee Collector, Class Coordinator. Each gets different dashboard views.
- **Role Overlap:** In smaller schools, an Admin might also teach. The `User` model supports a `TeacherProfile` attached to an Admin user simultaneously.
- **Session Management:** Shorter-lived sessions (4-hour access token) given elevated access. Refresh tokens last 7 days.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model User {
  id           String     @id @default(uuid())
  role         UserRole   // ADMIN (can also have TeacherProfile attached)
  name         String
  email        String?    @unique
  phoneNumber  String?    @unique
  passwordHash String?    // bcrypt, cost factor 12
  status       UserStatus @default(ACTIVE)
  schoolId     String
  createdAt    DateTime   @default(now())

  teacherProfile TeacherProfile?  // Some admins also teach
  refreshTokens  RefreshToken[]
  devices        DeviceToken[]
}

model RefreshToken {
  id        String   @id @default(uuid())
  userId    String
  token     String   @unique
  expiresAt DateTime
  revokedAt DateTime?

  user      User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}

enum UserRole   { STUDENT TEACHER PARENT ADMIN }
enum UserStatus { ACTIVE SUSPENDED DELETED }
```

#### NestJS Guards Stack
- `JwtAuthGuard` — validates Bearer token on every route
- `RolesGuard` — checks `@Roles(UserRole.ADMIN)` decorator
- `SchoolScopeGuard` — injects `schoolId` from JWT into all Prisma queries (multi-tenancy)

#### RBAC Permission Matrix (Admin-Specific)
| Action | Admin | Teacher | Student | Parent |
|--------|-------|---------|---------|--------|
| Broadcast to entire school | ✅ | — | — | — |
| Manage users & classes | ✅ | — | — | — |
| Create fee structures | ✅ | — | — | — |
| Generate fee invoices | ✅ | — | — | — |
| Record offline payment | ✅ | — | — | — |
| PixelTrace moderation | ✅ | — | — | — |
| View all attendance reports | ✅ | Own class | Own only | Own child |
| View audit logs | ✅ | — | — | — |
| Promote students (year-end) | ✅ | — | — | — |

---

## 2. User & Class Management (The Roster)

### Features
- **Bulk Import:** Upload an Excel/CSV with 500+ students, auto-assigning them to classes and generating credentials.
- **Manual User Creation:** Add individual students, teachers, or parent accounts.
- **Role Assignment:** Assign class teachers to sections (e.g., Mrs. Sharma → 10A class teacher).
- **Subject Mapping:** Define which teacher teaches which subject to which class.
- **Session Transitions (Year-End Promotion):** "Promote all students" to move 9A → 10A at year end.
- **Edit Window:** Correct wrong class assignments, admission numbers, or parent phone numbers.
- **Suspend / Reactivate:** Soft-delete users (sets `status = SUSPENDED`) without deleting records.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model School {
  id           String   @id @default(uuid())
  name         String
  board        String   // "CBSE" | "ICSE" | "State"
  city         String
  state        String
  logoUrl      String?
  timezone     String   @default("Asia/Kolkata")
  academicYear String   // "2024-2025"
}

model AcademicSession {
  id        String   @id @default(uuid())
  schoolId  String
  label     String   // "2024-2025"
  startDate DateTime
  endDate   DateTime
  isCurrent Boolean  @default(false)

  @@unique([schoolId, label])
}

model Class {
  id              String   @id @default(uuid())
  schoolId        String
  grade           String   // "10"
  section         String   // "A"
  sessionId       String
  classTeacherId  String?

  classTeacher    TeacherProfile? @relation(fields: [classTeacherId], references: [id])
  students        StudentProfile[]
  subjects        ClassSubject[]

  @@unique([schoolId, grade, section, sessionId])
  @@index([schoolId, sessionId])
}

model ClassSubject {
  id        String @id @default(uuid())
  classId   String
  subjectId String
  teacherId String

  @@unique([classId, subjectId])
}

model StudentProfile {
  id          String   @id @default(uuid())
  userId      String   @unique
  classId     String
  admissionNo String   @unique
  rollNo      Int
  dateOfBirth DateTime?
  gender      String?
  parentId    String?

  user        User          @relation(fields: [userId], references: [id])
  class       Class         @relation(fields: [classId], references: [id])
  consent     ConsentRecord?

  @@unique([classId, rollNo])
  @@index([classId])
}

model TeacherProfile {
  id         String @id @default(uuid())
  userId     String @unique
  employeeId String
  department String?

  classSubjects ClassSubject[]
}

// Track bulk import jobs for progress reporting
model BulkImportJob {
  id             String   @id @default(uuid())
  schoolId       String
  type           String   // 'STUDENTS' | 'TEACHERS'
  initiatedById  String
  status         String   @default("QUEUED") // QUEUED | PROCESSING | DONE | FAILED
  totalRows      Int      @default(0)
  processedRows  Int      @default(0)
  failedRows     Int      @default(0)
  errorReportUrl String?  // S3 URL to failed_rows.csv
  createdAt      DateTime @default(now())
  completedAt    DateTime?

  @@index([schoolId, createdAt])
}
```

#### CSV Import Flow (BullMQ)
```typescript
// 1. Admin uploads CSV → POST /api/v1/admin/users/bulk-import (multipart)
// 2. Backend saves CSV to S3 at: imports/{schoolId}/{jobId}.csv
// 3. Creates BulkImportJob { status: 'QUEUED' }
// 4. Enqueues BullMQ job on `bulk-import` queue: { jobId, s3Key, type: 'STUDENTS' }
// 5. Returns { jobId } immediately — admin polls for progress

// BulkImportWorker:
//   - Download CSV from S3
//   - Parse with csv-parser
//   - Validate each row with Zod:
//     z.object({ name: z.string(), admissionNo: z.string(), class: z.string(),
//                phoneNumber: z.string().regex(/^[6-9]\d{9}$/) })
//   - For valid rows: prisma.$transaction([
//       prisma.user.create({ data: { name, phoneNumber, role: 'STUDENT', ... } }),
//       prisma.studentProfile.create({ data: { userId, classId, admissionNo, rollNo } })
//     ])
//   - Collect invalid rows with reason into errors[]
//   - On completion:
//     * Generate failed_rows.csv, upload to S3
//     * UPDATE BulkImportJob { status: 'DONE', processedRows, failedRows, errorReportUrl }
//     * Push notification to admin: "Import done: 490 added, 10 failed. Download error report."

// Expected CSV columns:
// name, admissionNo, grade, section, rollNo, phoneNumber (parent), dob (optional)
```

#### Year-End Promotion (Background Job)
```typescript
// POST /api/v1/admin/classes/promote
// Body: { fromSessionId: "uuid-2023-24", toSessionId: "uuid-2024-25" }
// Returns: { jobId } — runs asynchronously

// Worker logic:
//   1. Fetch all Classes WHERE sessionId = fromSessionId
//   2. For each class, fetch all StudentProfiles
//   3. Create new Class in toSessionId (grade + 1, same section)
//   4. UPDATE StudentProfile.classId = new class
//   5. Final grade students (grade 12): UPDATE User.status = 'GRADUATED'
//   6. Preserve AttendanceRecord, FeeInvoice history (no deletion)
//   7. Push notification to admin when complete
```

#### Backend API (NestJS)
- **`POST /api/v1/admin/users/bulk-import`**
- **`GET /api/v1/admin/import-jobs/:id`** — polling endpoint for progress
- **`GET /api/v1/admin/import-jobs/:id/error-report`** — download failed rows CSV
- **`POST /api/v1/admin/users`** — create individual user
- **`PATCH /api/v1/admin/users/:id`** — edit user details
- **`PATCH /api/v1/admin/users/:id/suspend`**
- **`POST /api/v1/admin/classes`** — create class
- **`POST /api/v1/admin/classes/promote`**
- **`POST /api/v1/admin/classes/:id/subjects`** — assign teacher to subject in class

#### Frontend (React Web Admin Portal)
```typescript
// Stack: React + Vite + @mui/x-data-grid + react-query
<Dropzone
  accept={{ 'text/csv': ['.csv'], 'application/vnd.ms-excel': ['.xls', '.xlsx'] }}
  onDrop={files => uploadBulkImport(files[0])}
/>

// Progress polling
useQuery(['importJob', jobId], fetchImportJob, {
  refetchInterval: (data) => data?.status === 'DONE' ? false : 2000
});

// DataGrid for users — server-side pagination + inline edit
<DataGrid
  rows={users}
  columns={userColumns}
  pagination
  paginationMode="server"
  rowCount={total}
  onPageChange={setPage}
  processRowUpdate={updateUser}
/>
```

---

## 3. School-Wide Broadcasts

Reaching every student, parent, or teacher instantly.

### Features
- **Broadcast Targets:** All Students, All Parents, All Teachers, Specific Grade, Specific Class.
- **SMS/WhatsApp Fallback:** Critical notices trigger SMS via MSG91 for users who haven't opened the in-app notice within 1 hour.
- **Rich Text Body:** Format announcements with bold, bullet lists, or links.
- **Scheduled Announcements:** Schedule a notice for a future date/time (e.g., announce Monday holiday on Friday).
- **Read Receipt Analytics:** Track delivery rate, open rate, SMS fallback count per announcement.
- **Acknowledgement Requirement:** Mark critical circulars as "requires acknowledgement" — parents must tap "I've read this."

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model Notification {
  id               String        @id @default(uuid())
  schoolId         String
  title            String
  body             String        // Rich text (HTML/Markdown)
  bodyHi           String?       // Hindi translation
  priority         PriorityLevel // URGENT | NORMAL
  targetType       TargetType    // SCHOOL | ROLE | CLASS | INDIVIDUAL
  targetId         String?       // classId, or role string, or userId
  senderId         String        // adminId
  requiresAck      Boolean       @default(false)
  requiresFallback Boolean       @default(false)
  scheduledFor     DateTime?     // null = send immediately
  createdAt        DateTime      @default(now())

  states           UserNotificationState[]

  @@index([schoolId, createdAt])
  @@index([targetType, targetId])
}

model UserNotificationState {
  id             String    @id @default(uuid())
  userId         String
  notificationId String
  isRead         Boolean   @default(false)
  readAt         DateTime?
  deliveredAt    DateTime?
  acknowledgedAt DateTime?

  @@unique([userId, notificationId])
  @@index([userId, isRead])
}

enum PriorityLevel { URGENT NORMAL }
enum TargetType    { SCHOOL ROLE CLASS INDIVIDUAL }
```

#### Broadcast Fan-Out (BullMQ)
```typescript
// POST /api/v1/admin/notifications/broadcast
// Service immediately returns { notificationId }
// BullMQ job: notifications queue
// Worker:
//   - Determine audience based on targetType:
//     SCHOOL  → SELECT users WHERE schoolId = X AND status = ACTIVE
//     ROLE    → SELECT users WHERE schoolId = X AND role = TARGET_ROLE
//     CLASS   → SELECT studentProfiles WHERE classId = X → get userIds
//   - Batch insert UserNotificationState rows (createMany, 500 at a time)
//   - Fetch DeviceToken for each userId
//   - Send push via expo-server-sdk-node in batches of 100
//   - For each DeviceNotRegistered error: SET DeviceToken.isActive = false (token cleanup)

// SMS Fallback (cron worker every 30 min):
//   Finds Notifications WHERE requiresFallback=true AND createdAt > 30min ago
//   For each: UserNotificationState WHERE isRead=false → get phoneNumber
//   Call MSG91 bulk SMS API
//   DLT Template ID: pre-registered (mandatory for TRAI compliance)
```

#### Analytics Endpoint
```typescript
// GET /api/v1/admin/notifications/:id/analytics
{
  total: 850,
  delivered: 841,
  read: 523,
  readRate: "62%",
  acknowledged: 312,   // If requiresAck = true
  smsSent: 89,
  smsDelivered: 81,
  byChannel: { push: 841, sms: 89 }
}
```

#### Frontend (React Web)
```typescript
// Broadcast compose form
// react-quill (WYSIWYG) or TipTap for rich text body
// Target selector: multi-select dropdown (School / All Parents / Grade 10 / Class 10A)
// Scheduled send: DateTimePicker
// Analytics: Recharts PieChart + DataGrid with open-rate column
```

---

## 4. Analytics & Reporting

Tracking the operational health of the school.

### Features
- **Attendance Compliance Dashboard:** Real-time heatmap showing which classes have submitted attendance today. Flag defaulting teachers.
- **Low Attendance Defaulters:** Auto-generate list of students below 75% for PTA meetings. Exportable to CSV/Excel.
- **Teacher Activity Report:** Notes uploaded per teacher per week, assignment creation frequency.
- **Fee Collection Report:** Monthly collection totals, outstanding dues, defaulter count.
- **PixelTrace Metrics:** Storage used, mismatch flags, searches per event.
- **Export All Reports:** Every data grid has "Export to CSV/Excel" — non-negotiable for Indian school admins.

### Technical Implementation

No new tables — all aggregation over existing data.

#### Attendance Compliance
```typescript
// GET /api/v1/admin/reports/attendance-compliance?date=2024-11-15
// Logic:
//   1. SELECT all active Classes WHERE schoolId = X
//   2. For each class, check if ANY AttendanceRecord.date = :date exists
//   3. Return: [{ classId, grade, section, teacherName, hasMarked, lastMarkedAt }]
//   4. If !hasMarked AND current IST time > 10:00 AM → flag 'LATE'

// Frontend: DataGrid with inline "Send Reminder" button per row
// "Send Reminder" → POST /api/v1/admin/teachers/:teacherId/nudge
//   → BullMQ job → push to teacher: "Attendance for 10A is still pending."
```

#### Low Attendance Report (Raw SQL for performance)
```sql
-- GET /api/v1/admin/reports/low-attendance?threshold=75&sessionId=UUID
SELECT
  sp.id            AS student_id,
  u.name           AS student_name,
  c.grade,
  c.section,
  COUNT(*) FILTER (WHERE ar.status = 'PRESENT') * 100.0 / NULLIF(COUNT(*), 0) AS percentage
FROM student_profiles sp
JOIN users u ON u.id = sp.user_id
JOIN classes c ON c.id = sp.class_id AND c.session_id = :sessionId
LEFT JOIN attendance_records ar
  ON ar.student_id = sp.id
  AND ar.date BETWEEN :sessionStartDate AND CURRENT_DATE
WHERE sp.school_id = :schoolId
GROUP BY sp.id, u.name, c.grade, c.section
HAVING percentage < :threshold OR COUNT(*) = 0
ORDER BY percentage ASC NULLS FIRST;
```

#### Teacher Activity Report
```typescript
// GET /api/v1/admin/reports/teacher-activity?from=2024-11-01&to=2024-11-15
// Returns per-teacher:
//   - notesUploadedCount (SELECT COUNT FROM notes WHERE teacherId AND uploadedAt BETWEEN ...)
//   - assignmentsCreatedCount
//   - attendanceMarkingRate (days marked / school days in range * 100)
// Used to identify under-performing teachers (sent to principal)
```

---

## 5. Fee Management

### Features
- **Fee Structures:** Create named fee types (Tuition, Transport, Exam Fee) per class and session, with due dates.
- **Invoice Generation:** Bulk-generate invoices for all students in a class when a fee structure is created.
- **Late Fine Engine:** Automatically calculate accumulated late fines when a payment is made after the due date.
- **Defaulter Management:** Paginated list of students with pending dues; bulk SMS payment links.
- **Offline Payment Recording:** Admin records cash/cheque/NEFT payments received at the counter.
- **Waiver:** Mark invoices as WAIVED for scholarship/exemption cases with a logged reason.
- **Receipt Generation:** Auto-generate compliant PDF receipts uploaded to S3 for parent download.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model FeeStructure {
  id             String   @id @default(uuid())
  schoolId       String
  classId        String?  // null = school-wide
  sessionId      String
  name           String   // "Tuition Fee Q1"
  type           FeeType  // TUITION | TRANSPORT | EXAM | LIBRARY | DEVELOPMENT
  amount         Decimal  @db.Decimal(10, 2)
  dueDate        DateTime
  lateFinePerDay Decimal  @db.Decimal(8, 2) @default(0)

  invoices       FeeInvoice[]

  @@index([schoolId, sessionId])
}

model FeeInvoice {
  id              String        @id @default(uuid())
  schoolId        String
  studentId       String
  feeStructureId  String
  amountDue       Decimal       @db.Decimal(10, 2)
  lateFine        Decimal       @db.Decimal(10, 2) @default(0)
  status          InvoiceStatus // PENDING | PARTIAL | PAID | WAIVED
  dueDate         DateTime
  paidAt          DateTime?
  razorpayOrderId String?       @unique
  waivedById      String?
  waiverReason    String?

  student         StudentProfile @relation(fields: [studentId], references: [id])
  feeStructure    FeeStructure   @relation(fields: [feeStructureId], references: [id])
  payments        FeePayment[]

  @@index([studentId, status])
  @@index([schoolId, status])
}

model FeePayment {
  id              String        @id @default(uuid())
  invoiceId       String
  amountPaid      Decimal       @db.Decimal(10, 2)
  paymentDate     DateTime      @default(now())
  method          PaymentMethod // RAZORPAY | CASH | CHEQUE | NEFT
  transactionId   String?       // Razorpay payment_id or cheque number
  razorpayOrderId String?
  receiptUrl      String?       // S3 URL to PDF receipt
  collectedById   String?       // adminId for offline payments

  invoice         FeeInvoice @relation(fields: [invoiceId], references: [id])

  @@index([invoiceId])
}

enum FeeType       { TUITION TRANSPORT EXAM LIBRARY DEVELOPMENT }
enum InvoiceStatus { PENDING PARTIAL PAID WAIVED }
enum PaymentMethod { RAZORPAY CASH CHEQUE NEFT }
```

#### Backend API (NestJS)
- **`POST /api/v1/admin/fees/structures`** — create fee structure
- **`POST /api/v1/admin/fees/invoices/generate`** → `{ feeStructureId }` — triggers BullMQ `bulk-invoice-generation` job: iterates all students in the target class, creates one `FeeInvoice` per student.
- **`GET /api/v1/admin/fees/defaulters?sessionId=&status=PENDING&page=1&limit=50`**
- **`POST /api/v1/admin/fees/reminders/bulk`** → `{ invoiceIds[] }` — generates personalised SMS payment links via MSG91
- **`POST /api/v1/admin/fees/payments/offline`** → `{ invoiceId, amountPaid, method, transactionId }` — records cash/cheque payment, triggers receipt generation
- **`PATCH /api/v1/admin/fees/invoices/:id/waive`** → `{ reason }` — marks WAIVED, logs to `AuditLog`
- **`POST /api/webhooks/razorpay`** — Razorpay payment confirmation (signature-verified)

#### PDF Receipt Generation (BullMQ)
```typescript
// Queue: pdf-generation
// Job payload: { paymentId }
// Worker:
//   1. Fetch FeePayment + FeeInvoice + StudentProfile + School
//   2. Render HTML via Handlebars template:
//      - School name, logo, address
//      - Student name, admission number, class
//      - Fee type, amount, date, method, transaction ID
//      - School stamp placeholder
//   3. Puppeteer: await page.pdf({ format: 'A4', printBackground: true })
//   4. Upload to S3: schools/{schoolId}/receipts/{paymentId}_receipt.pdf
//   5. UPDATE FeePayment SET receiptUrl = ...
//   6. Push notification to parent: "Receipt ready. Tap to download."
```

#### Late Fine Calculation
```typescript
// Computed at checkout time — not stored until payment
// Service:
const today = new Date();
const dueDatePassed = differenceInDays(today, invoice.dueDate);
const lateFine = dueDatePassed > 0
  ? dueDatePassed * Number(invoice.feeStructure.lateFinePerDay)
  : 0;
const totalDue = Number(invoice.amountDue) - alreadyPaid + lateFine;
// UPDATE FeeInvoice SET lateFine = lateFine before creating Razorpay order
```

---

## 6. PixelTrace Moderation & Privacy

### Features
- **Mismatch Review Queue:** Photos flagged as "This is not me" by students — side-by-side comparison of the photo with the tagged student's profile.
- **Content Takedown:** Unpublish a photo or entire event immediately.
- **Bulk Tag Removal:** Remove all tags from a photo in one action.
- **AI Confidence Review (v2):** Human verification queue for low-confidence AI matches before they're shown to students.
- **Audit Log:** Every moderation action logged with who did what and when.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model EventPhoto {
  id           String   @id @default(uuid())
  eventId      String
  schoolId     String
  photoUrl     String
  thumbnailUrl String
  uploadedById String
  uploadedAt   DateTime @default(now())
  isPublished  Boolean  @default(true)  // Admin can unpublish

  tags         PhotoTag[]
}

model PhotoTag {
  id           String   @id @default(uuid())
  photoId      String
  studentId    String
  taggedById   String
  confidence   Float?
  isVerified   Boolean  @default(false)
  flaggedWrong Boolean  @default(false)
  flaggedAt    DateTime?
  createdAt    DateTime @default(now())

  @@unique([photoId, studentId])
  @@index([flaggedWrong])   // Moderation queue index
}

model AuditLog {
  id          String      @id @default(uuid())
  schoolId    String
  action      AuditAction
  adminId     String
  targetTable String      // 'PhotoTag' | 'EventPhoto' | 'FeeInvoice' | 'User'
  targetId    String
  metadata    Json?       // { studentName, eventName, reason, previousValue }
  createdAt   DateTime    @default(now())

  @@index([schoolId, createdAt])
  @@index([adminId])
}

enum AuditAction {
  REMOVED_TAG
  CONFIRMED_TAG
  DELETED_PHOTO
  UNPUBLISHED_EVENT
  WAIVED_FEE
  BULK_IMPORT_USERS
  PROMOTED_CLASS
  SUSPENDED_USER
  REVOKED_CONSENT
}
```

#### Backend API (NestJS)
- **`GET /api/v1/admin/pixeltrace/moderation-queue?page=1&limit=20`** — `PhotoTag WHERE flaggedWrong = true` with joined photo URL + student profile photo for side-by-side display
- **`DELETE /api/v1/admin/pixeltrace/tags/:tagId`** — removes tag, inserts `AuditLog { action: 'REMOVED_TAG' }`
- **`POST /api/v1/admin/pixeltrace/tags/:tagId/confirm`** — sets `isVerified = true, flaggedWrong = false`, logs `CONFIRMED_TAG`
- **`PATCH /api/v1/admin/events/:eventId/photos/:photoId`** → `{ isPublished: false }` — unpublishes, logs `DELETED_PHOTO`
- **`PATCH /api/v1/admin/events/:eventId`** → `{ isPublished: false }` — unpublishes entire event, logs `UNPUBLISHED_EVENT`
- **`GET /api/v1/admin/audit-logs?from=&to=&action=&adminId=`** — full audit trail, paginated

#### Frontend (React Web)
```typescript
// Tinder-like moderation interface for speed
// Left: EventPhoto showing the student in question (thumbnailUrl)
// Right: StudentProfile card (name, class, profile photo)
// Keyboard shortcuts for power users:
//   [A] = Remove Tag (flagged wrong)
//   [D] = Confirm Match (valid)
//   [S] = Skip (review later)

// Recharts BarChart for "flags per event" overview
// DataGrid for full audit log with date/action/admin filters + CSV export
```

---

## 7. Timetable Management

### Features
- **Period Creation:** Define period slots (time, day, subject, teacher) per class per academic session.
- **Teacher Conflict Detection:** Prevent assigning a teacher to two classes at the same time.
- **Bulk Import:** Upload a timetable CSV for a class.
- **Publish/Unpublish:** Changes are staged until published to avoid confusing students mid-week.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model TimetablePeriod {
  id        String   @id @default(uuid())
  schoolId  String
  classId   String
  subjectId String
  teacherId String
  dayOfWeek Int      // 1=Mon, 2=Tue, ... 6=Sat
  periodNo  Int      // 1-8
  startTime String   // "09:00" (IST, 24h)
  endTime   String   // "09:45"
  sessionId String
  isDraft   Boolean  @default(true)  // Not visible to students/teachers until published

  class     Class   @relation(fields: [classId], references: [id])
  subject   Subject @relation(fields: [subjectId], references: [id])

  @@unique([classId, dayOfWeek, periodNo, sessionId])
  @@index([teacherId, dayOfWeek])  // Conflict detection index
}
```

#### Backend API (NestJS)
- **`POST /api/v1/admin/timetable`** — create/upsert period
  - On creation: check `SELECT COUNT FROM TimetablePeriod WHERE teacherId = X AND dayOfWeek = Y AND periodNo = Z AND isDraft = false AND sessionId = CURRENT` — if > 0, return `409 Conflict`.
- **`POST /api/v1/admin/timetable/publish?classId=`** — sets `isDraft = false` for all periods of a class.
- **`POST /api/v1/admin/timetable/import`** — CSV import, same bulk pattern as user import.

---

## 8. Academic Calendar & Events

### Features
- **School Calendar:** Create holidays, exam dates, PTM dates, event days. Visible to all users.
- **PixelTrace Event Management:** Create events for photo upload (Annual Day, Sports Day). Admin can link a Calendar Event to a PixelTrace Event.
- **Holiday Alerts:** On creating a holiday, admin can trigger an URGENT broadcast notification automatically.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model CalendarEvent {
  id          String   @id @default(uuid())
  schoolId    String
  title       String
  date        DateTime @db.Date
  type        CalendarEventType // HOLIDAY | EXAM | PTM | EVENT | OTHER
  description String?
  isHoliday   Boolean  @default(false)

  @@index([schoolId, date])
}

enum CalendarEventType { HOLIDAY EXAM PTM EVENT OTHER }
```

#### Backend API (NestJS)
- **`POST /api/v1/admin/calendar`** — create calendar event
  - If `isHoliday = true`, auto-enqueue a `notifications` BullMQ job broadcasting to all school users.
- **`GET /api/v1/calendar?from=&to=`** — public endpoint (accessible by all roles, no PII).

---

## 9. Data Correction Tools

### Features
- **Attendance Fix:** Override any historical attendance record with a correction note.
- **Student Re-class:** Move a student from one class to another mid-year with an audit trail.
- **Duplicate Merge:** If a student was accidentally created twice, merge records under one `admissionNo`.
- **Parent Phone Change:** Update the phone number used for OTP login (requires admin verification).

### Technical Implementation

```typescript
// PATCH /api/v1/admin/attendance/:recordId
// Body: { status, remarks, correctionReason }
// Inserts AuditLog: { action: 'ATTENDANCE_CORRECTED', metadata: { from, to, reason } }

// PATCH /api/v1/admin/students/:studentId/class
// Body: { newClassId, reason }
// Prisma transaction:
//   UPDATE StudentProfile SET classId = newClassId
//   INSERT AuditLog { action: 'STUDENT_RECLASSED', metadata: { fromClass, toClass, reason } }
```

---

## 10. Security, Privacy & Compliance (DPDP Act)

### Features
- **Consent Dashboard:** View all students, their `pixelTraceOptIn` status, and consent dates. Filter by "Not Yet Decided".
- **Data Deletion Request:** Process a parent's "right to erasure" request — deletes PII, face embeddings, photo tags. Generates a confirmation report.
- **Data Retention Policy:** Auto-archive events older than 3 years; auto-delete face embeddings after student graduates.
- **Audit Log Access:** Full searchable audit trail for all sensitive actions.

### Technical Implementation

```typescript
// POST /api/v1/admin/privacy/delete-request
// Body: { studentId, reason }
// Transaction:
//   1. DELETE StudentFaceEnrollment WHERE studentId = X
//   2. DELETE PhotoTag WHERE studentId = X AND confidence IS NOT NULL (AI-generated only)
//   3. Anonymize User: SET name = 'DELETED', phoneNumber = NULL, email = NULL
//   4. SET ConsentRecord.pixelTraceOptIn = false
//   5. INSERT AuditLog { action: 'DATA_DELETED', metadata: { reason, requestedBy } }

// Retention Policy (Cron, runs on midnight of last day of month):
//   1. Find Events WHERE date < NOW() - 3 years
//   2. DELETE EventPhoto + PhotoTag records
//   3. Delete S3 files via batch S3 delete
//   4. INSERT AuditLog { action: 'AUTO_ARCHIVED' }
```

---

## 11. Technical Edge Cases & Considerations (Indian Context)

1. **The "Excel Culture":** Every data grid MUST have "Export to CSV/Excel" (`xlsx` npm package). Every setup flow must allow CSV import. Admins use Excel for everything from attendance reports to fee reconciliation. If this is missing, the system will be rejected.
2. **Role Overlap:** In small schools (< 500 students), the principal also teaches, and the accountant also does admin. A `User` can have both `ADMIN` role and a `TeacherProfile` — the JWT role is `ADMIN` but the admin portal shows Teacher-specific views too.
3. **Connectivity:** Web portal is on school broadband, which can be patchy. React Query on the web must handle retries gracefully. Show offline indicators in the DataGrid header.
4. **Data Privacy (DPDP Act):** Indian Digital Personal Data Protection Act requires: written consent for minors, opt-in before processing photos, right to erasure, and audit trails. The Consent Dashboard and Data Deletion workflow are not optional — they are legal requirements before enabling PixelTrace.
5. **SMS Provider DLT Registration:** All SMS messages (OTP, fee reminders, urgent alerts) require TRAI DLT registration. Every SMS template must be pre-approved. Budget at least 2-3 weeks for DLT approval before launch.
6. **Razorpay Settlement:** School receives payments in their bank account via Razorpay settlement (T+2 days). The admin portal should show a "Settlement Report" reconciling Razorpay payouts with `FeePayment` records. Discrepancies must be flaggable.
7. **Storage Cost Growth:** Photos + PDF receipts + notes grow fast. Enforce per-school storage quotas. Implement S3 lifecycle policies: move objects > 1 year to S3-IA (Infrequent Access), > 3 years to S3 Glacier. Show current storage usage in the admin dashboard.
