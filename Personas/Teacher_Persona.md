# Teacher Persona: Features & Technical Implementation

## Persona Overview
**The Teacher** is the primary content creator and administrator of daily activities on the StudyDash app. They manage large classes (often 40-60 students in an Indian context), meaning bulk actions and speed are paramount. They require offline-capable workflows for classrooms with poor connectivity and simple, repetitive tasks to avoid "tool fatigue."

---

## 1. Authentication & Session Management

### Features
- **Password Login:** Teachers and staff log in with email + password (more secure for professional accounts).
- **OTP Login (Alternative):** Phone number + OTP as fallback or for non-email accounts.
- **Session Persistence:** Access token (15 min) + refresh token (30 days). Silent re-auth.
- **Multi-Class Support:** A teacher may be linked to multiple classes across subjects.

### Technical Implementation

#### JWT Token Structure
```typescript
interface TeacherJwtPayload {
  sub: string;        // userId
  role: 'TEACHER';
  schoolId: string;
  teacherId: string;  // TeacherProfile.id — avoids join on every request
  iat: number;
  exp: number;
}
```

#### Database Schema (Prisma)
```prisma
model TeacherProfile {
  id          String  @id @default(uuid())
  userId      String  @unique
  employeeId  String
  department  String?

  user        User    @relation(fields: [userId], references: [id])
  // Which subjects the teacher teaches and to which classes
  classSubjects ClassSubject[]
}

// Many-to-many: which teacher teaches which subject to which class
model ClassSubject {
  id        String @id @default(uuid())
  classId   String
  subjectId String
  teacherId String

  class     Class          @relation(fields: [classId], references: [id])
  subject   Subject        @relation(fields: [subjectId], references: [id])
  teacher   TeacherProfile @relation(fields: [teacherId], references: [id])

  @@unique([classId, subjectId])
}
```

#### Backend API (NestJS)
- **`POST /api/v1/auth/login`** — email + password. Returns `{ accessToken, refreshToken }`.
- **`POST /api/v1/auth/otp/send`** + **`/verify`** — phone OTP fallback.
- **`POST /api/v1/auth/refresh`** and **`/logout`** — same as student flow.

#### Frontend (React Native + Expo)
- **Screen:** `(auth)/login.tsx` — form with email/password fields using `react-hook-form` + `zod`.
- **Token storage:** `react-native-mmkv`.
- **Role-based routing:** After login, `expo-router` redirects to `(app)/teacher/dashboard` based on `role` in JWT.

---

## 2. Teacher Dashboard (Home Screen)

### Features
- **Today's Classes:** List of periods the teacher is scheduled for today.
- **Pending Actions:** Classes with attendance not yet marked (red badge).
- **Recent Activity:** Assignments nearing due date, recent note uploads.
- **Quick Actions:** One-tap "Mark Attendance for 10A", "Upload Note", "Create Assignment".

### Technical Implementation
- **Endpoint:** `GET /api/v1/teachers/dashboard`
- **Logic:** Aggregates today's `TimetablePeriod` for the teacher, checks which classes have no `AttendanceRecord` for today, fetches recent `Assignment` completions.
- **Redis cache:** `teacher:dashboard:{teacherId}` TTL 5 minutes.

---

## 3. Assignment Creation & Management

### Features
- **Create/Edit/Delete:** Add title, description, due date per subject + class.
- **Bulk Assignment:** Assign the same homework to multiple sections simultaneously (e.g., 9A, 9B, 9C).
- **Attachments:** Upload PDFs from the device or snap a photo of the board directly from camera.
- **Completion Tracking:** Visual summary showing "X/Y students completed" with a drilldown list.
- **WhatsApp Share (Bridge):** Share a deep link to the assignment via WhatsApp during transition period.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model Assignment {
  id          String   @id @default(uuid())
  schoolId    String
  classId     String
  subjectId   String
  teacherId   String
  title       String
  description String?
  dueDate     DateTime
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  class       Class                @relation(fields: [classId], references: [id])
  subject     Subject              @relation(fields: [subjectId], references: [id])
  attachments AssignmentAttachment[]
  completions AssignmentCompletion[]

  @@index([classId, dueDate])
  @@index([teacherId])
}

model AssignmentAttachment {
  id           String @id @default(uuid())
  assignmentId String
  fileUrl      String   // Full CDN URL
  fileName     String
  fileType     String   // 'pdf' | 'image' | 'doc'
  fileSizeBytes Int

  assignment   Assignment @relation(fields: [assignmentId], references: [id], onDelete: Cascade)
}

model AssignmentCompletion {
  id           String   @id @default(uuid())
  assignmentId String
  studentId    String
  completedAt  DateTime @default(now())

  @@unique([assignmentId, studentId])
  @@index([studentId])
}
```

#### Backend API (NestJS)
- **`POST /api/v1/teachers/assignments`**
  ```json
  {
    "title": "Ch 5 Quadratic Equations",
    "description": "Solve exercises 5.1 and 5.2",
    "dueDate": "2024-11-20T23:59:00+05:30",
    "classIds": ["uuid-10A", "uuid-10B"],
    "subjectId": "uuid-math"
  }
  ```
  - If `classIds.length > 1`: Prisma `$transaction` creates one `Assignment` per class atomically.
- **`PUT /api/v1/teachers/assignments/:id`**
- **`DELETE /api/v1/teachers/assignments/:id`**
- **`GET /api/v1/teachers/assignments/:id/stats`** → `{ totalStudents: 45, completed: 12, completedStudents: [{id, name, completedAt}] }`

#### File Upload Flow (Presigned S3)
1. Teacher calls `POST /api/v1/upload/presign` with `{ fileName, contentType, context: 'assignment' }`.
2. Backend returns `{ uploadUrl, key }` — 15-min presigned S3 PUT URL.
3. App uploads directly to S3 (bypasses API server; saves server bandwidth).
4. Assignment creation request includes the S3 `key` in `attachments`.
5. Backend saves `fileUrl = CDN_BASE_URL + key`.

#### Frontend (React Native + Expo)
```typescript
// react-hook-form + zod
const schema = z.object({
  title: z.string().min(3).max(200),
  description: z.string().optional(),
  dueDate: z.date().min(new Date()),
  classIds: z.array(z.string().uuid()).min(1),
  subjectId: z.string().uuid(),
});

// Camera capture of board (common in Indian classrooms)
const photo = await ImagePicker.launchCameraAsync({ quality: 0.7 });

// Optimistic UI — assignment appears instantly, syncs in background
// Prevents teacher from waiting in front of the class
```

---

## 4. Class Notes Upload

Daily upload of study materials, previous year question papers (PYQs), or class summaries.

### Features
- **Daily Uploads:** Select PDFs or images per subject and class.
- **Note Reuse:** Pick a note uploaded last year or for another section and re-assign it.
- **Read Receipts:** See how many students have downloaded/viewed a specific note.
- **Background Upload:** If a large PDF upload is started while walking between classes, it continues uploading in the background and retries on reconnect.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model Note {
  id            String   @id @default(uuid())
  schoolId      String
  classId       String
  subjectId     String
  teacherId     String
  title         String
  fileUrl       String
  thumbnailUrl  String?  // Generated by BullMQ worker after upload
  fileType      String   // 'pdf' | 'image' | 'doc'
  fileSizeBytes Int
  uploadedAt    DateTime @default(now())

  views         NoteView[]

  @@index([classId, subjectId, uploadedAt])
  @@index([teacherId])
}

model NoteView {
  id        String   @id @default(uuid())
  noteId    String
  studentId String
  viewedAt  DateTime @default(now())

  @@unique([noteId, studentId])
}
```

#### Backend API (NestJS)
- **`POST /api/v1/teachers/notes`** — same S3 presign flow as assignments.
- **`GET /api/v1/teachers/notes`** — teacher's uploaded notes, most recent first.
- **`GET /api/v1/teachers/notes/:id/analytics`** → `{ totalStudents: 45, views: 28 }`
- **`DELETE /api/v1/teachers/notes/:id`**
- **`POST /api/v1/teachers/notes/:id/assign-to-class`** — reuse an existing note for a different class (creates new `Note` row pointing to same S3 key, no re-upload needed).

#### Background Thumbnail Generation (BullMQ)
```typescript
// Queue: media-processing
// Job payload: { noteId, s3Key, fileType }
// Worker:
//   - PDF: render first page to PNG via pdf-lib, then sharp → 400px WebP
//   - Image: sharp resize to 400px wide, save as WebP
//   - Upload to S3: notes/thumbnails/{noteId}_thumb.webp
//   - UPDATE Note SET thumbnailUrl = ...
```

#### Frontend
```typescript
// expo-document-picker for PDF
const doc = await DocumentPicker.getDocumentAsync({
  type: ['application/pdf', 'image/*'],
  copyToCacheDirectory: true,
});

// Background upload via expo-task-manager
// Shows a persistent notification: "Uploading Chemistry Notes... 67%"
// Queues retry if upload fails mid-way
```

---

## 5. Attendance Marking

The most critical and repetitive daily task. Must be blazingly fast.

### Features
- **Default All Present:** Opens with all students marked Present — teacher only taps exceptions.
- **Bulk "Mark All Present" Button:** One tap to set entire class.
- **Status Options:** Present / Absent / Late / Half Day.
- **Leave Requests Integration:** Students with an approved `LeaveRequest` pre-filled as "On Leave".
- **Offline Mode:** If classroom has no signal, attendance is saved locally and auto-syncs when the teacher reaches the staff room.
- **Edit Window:** Modify today's attendance (e.g., a student arrives late after roll call).
- **Haptic Feedback:** Tactile confirmation when toggling Absent — important when managing a noisy classroom.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model AttendanceRecord {
  id         String           @id @default(uuid())
  schoolId   String
  studentId  String
  classId    String
  date       DateTime         @db.Date
  status     AttendanceStatus
  remarks    String?
  markedById String           // teacherId

  @@unique([studentId, date])
  @@index([classId, date])
  @@index([studentId, date])
}

enum AttendanceStatus {
  PRESENT
  ABSENT
  LATE
  HALF_DAY
  ON_LEAVE
}
```

#### Backend API (NestJS)
- **`GET /api/v1/teachers/classes/:classId/students`** — returns student list with today's status (null if not yet marked) and any pending approved `LeaveRequest` dates.
- **`POST /api/v1/teachers/attendance/bulk`**
  ```json
  {
    "classId": "uuid-10A",
    "date": "2024-11-15",
    "records": [
      { "studentId": "uuid-1", "status": "PRESENT" },
      { "studentId": "uuid-2", "status": "ABSENT", "remarks": "Sick" }
    ]
  }
  ```
  - Prisma `$transaction` with `upsert` for each record — idempotent if submitted twice.
  - Post-transaction: enqueues `attendance-alerts` BullMQ job for each ABSENT record → parent push notification.
- **`PATCH /api/v1/teachers/attendance/:recordId`** — edit single record.
- **`GET /api/v1/teachers/attendance/report?classId=&month=`** — class-level monthly summary.

#### Frontend (React Native)
```typescript
// FlashList (Shopify) — far better than FlatList for 60-item lists on budget Android
<FlashList
  data={students}
  estimatedItemSize={64}
  renderItem={({ item }) => (
    <StudentAttendanceRow
      student={item}
      status={statuses[item.id] ?? 'PRESENT'}
      onToggle={(status) => {
        updateStatus(item.id, status);
        Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium); // expo-haptics
      }}
    />
  )}
/>

// Offline-first flow
// SQLite table: pending_attendance_submissions(id, payload JSON, createdAt, syncedAt)
// 1. Save to SQLite immediately
// 2. Attempt API call — on failure, keep in queue
// 3. NetInfo listener: on reconnect → flush queue to API
// 4. On success: set syncedAt, remove from queue
```

---

## 6. Class Notifications & Communication

### Features
- **Targeted Broadcasts:** Select specific classes, sections, or the whole grade.
- **Priority Toggle:** URGENT triggers immediate push; NORMAL appears silently in inbox.
- **Templates:** Pre-defined common templates to reduce typing.
- **WhatsApp Fallback Link:** Generate a formatted text summary to paste into a WhatsApp group during the transition period.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model Notification {
  id         String        @id @default(uuid())
  schoolId   String
  title      String
  body       String
  bodyHi     String?       // Optional Hindi translation
  priority   PriorityLevel // URGENT | NORMAL
  targetType TargetType    // CLASS | SCHOOL
  targetId   String?       // classId
  senderId   String        // teacherId
  createdAt  DateTime      @default(now())

  states     UserNotificationState[]

  @@index([senderId])
}

model UserNotificationState {
  id             String    @id @default(uuid())
  userId         String
  notificationId String
  isRead         Boolean   @default(false)
  readAt         DateTime?
  deliveredAt    DateTime?

  @@unique([userId, notificationId])
  @@index([userId, isRead])
}

// Reusable notification templates
model NotificationTemplate {
  id        String        @id @default(uuid())
  schoolId  String
  name      String        // "PTM Reminder"
  title     String
  body      String
  priority  PriorityLevel

  @@index([schoolId])
}
```

Pre-seeded templates: PTM Reminder, Fee Reminder, Holiday Homework, Bring Lab Coat, Exam Schedule.

#### Backend API (NestJS)
- **`POST /api/v1/teachers/notifications`**
  ```json
  {
    "title": "Bring lab coats tomorrow",
    "body": "Chemistry practical is in Period 3.",
    "priority": "NORMAL",
    "targetType": "CLASS",
    "targetIds": ["uuid-10A", "uuid-10B"]
  }
  ```
- API responds instantly; BullMQ `notifications` worker fans out push to all students in the target classes.
- **`GET /api/v1/teachers/notifications`** — sent history with open/delivery counts.
- **`GET /api/v1/teachers/notification-templates`** — list templates.

#### Frontend
```typescript
// Compose screen: (app)/teacher/notifications/compose.tsx
// Template picker modal → pre-fills title/body
// Class multi-select
// Priority switch: "Urgent (Push Now)" vs "Normal (Inbox Only)"
```

---

## 7. Attendance & Assignment Analytics

### Features
- **Completion Rate by Assignment:** Pie chart of completed vs pending per assignment.
- **Class Attendance Trend:** Line graph showing weekly attendance percentage for the class.
- **Low Attendance Students:** List of students in the class below the school threshold.

### Technical Implementation
- **`GET /api/v1/teachers/analytics/assignments?classId=&subjectId=`** → `[{ assignmentId, title, dueDate, completionRate }]`
- **`GET /api/v1/teachers/analytics/attendance?classId=&month=`** → weekly aggregation.
- **`GET /api/v1/teachers/analytics/low-attendance?classId=&threshold=75`** → student list.

---

## 8. Leave Request Review

### Features
- **Review Queue:** Pending leave requests from parents for students in the teacher's classes.
- **Approve/Reject:** With optional note.
- **Auto-Attendance Update:** On approval, `AttendanceRecord` rows for the leave dates are upserted with `status = ON_LEAVE`.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model LeaveRequest {
  id           String        @id @default(uuid())
  schoolId     String
  studentId    String
  parentId     String
  startDate    DateTime      @db.Date
  endDate      DateTime      @db.Date
  reason       String
  status       RequestStatus @default(PENDING)
  reviewedById String?       // teacherId
  reviewNote   String?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt

  @@index([schoolId, status])
}

enum RequestStatus { PENDING APPROVED REJECTED }
```

#### Backend API (NestJS)
- **`GET /api/v1/teachers/leaves?classId=&status=PENDING`**
- **`PATCH /api/v1/teachers/leaves/:id`** → `{ status: 'APPROVED', reviewNote: '...' }`
  - On APPROVED: Prisma `$transaction` upserts `AttendanceRecord` with `ON_LEAVE` for each date in `startDate..endDate`.
  - Push notification to parent: "Leave approved for Aarav (Nov 18–19)."

---

## 9. PixelTrace — Event Photo Upload & Tagging

Teachers act as unofficial photographers during school events (Annual Day, Sports Day).

### Features
- **Create Event:** Name, date, and optional description (e.g., "Independence Day 2024").
- **Bulk Upload:** Select up to 50 photos from the gallery per batch; continue in multiple batches.
- **Manual Tagging (MVP):** Full-screen photo view; tap to place a marker on a face and search/select a student to tag.
- **Client-Side Compression:** Photos compressed on-device before upload to save teacher's mobile data.
- **Tagging Progress:** Visual indicator showing "12/50 photos tagged".

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model Event {
  id            String   @id @default(uuid())
  schoolId      String
  name          String
  date          DateTime
  description   String?
  coverPhotoUrl String?
  createdById   String   // teacherId
  createdAt     DateTime @default(now())

  photos        EventPhoto[]

  @@index([schoolId, date])
}

model EventPhoto {
  id           String   @id @default(uuid())
  eventId      String
  schoolId     String
  photoUrl     String   // Full-res CDN URL
  thumbnailUrl String   // 300px WebP, generated by worker
  uploadedById String   // teacherId
  uploadedAt   DateTime @default(now())
  isPublished  Boolean  @default(true)

  tags         PhotoTag[]

  @@index([eventId])
}

model PhotoTag {
  id           String   @id @default(uuid())
  photoId      String
  studentId    String
  taggedById   String   // teacherId
  confidence   Float?   // null for manual; 0-1 for AI (v2)
  isVerified   Boolean  @default(true)  // Auto-verified when tagged by teacher
  flaggedWrong Boolean  @default(false)
  flaggedAt    DateTime?
  createdAt    DateTime @default(now())

  @@unique([photoId, studentId])
  @@index([studentId, isVerified])
  @@index([flaggedWrong])
}
```

#### Backend API (NestJS)
- **`POST /api/v1/teachers/events`** → create event
- **`POST /api/v1/teachers/events/:eventId/photos`** — multipart array upload (up to 50 photos). Uses `multer-s3` to stream directly to S3. Triggers `media-processing` BullMQ job per photo.
- **`GET /api/v1/teachers/events/:eventId/photos/untagged`** — photos with no tags yet.
- **`POST /api/v1/teachers/photos/:photoId/tags`** → `{ studentId }` — creates `PhotoTag` with `isVerified = true`.
- **`DELETE /api/v1/teachers/photos/:photoId/tags/:tagId`** — remove an incorrect manual tag.

#### Frontend (React Native)
```typescript
// Client-side compression before upload (saves ~90% bandwidth)
const compressed = await ImageManipulator.manipulateAsync(
  localUri,
  [{ resize: { width: 1920 } }],
  { compress: 0.8, format: ImageManipulator.SaveFormat.JPEG }
);
// Typical result: 5MB RAW → ~400KB

// Gallery multi-select: expo-image-picker
const result = await ImagePicker.launchImageLibraryAsync({
  allowsMultipleSelection: true,
  selectionLimit: 50,
  mediaTypes: ImagePicker.MediaTypeOptions.Images,
});

// Tagging UI
// Screen: (app)/teacher/pixeltrace/tag/[photoId].tsx
// Tap anywhere on image → shows SearchableStudentList modal filtered by teacher's class
// On student selected → POST /teachers/photos/:photoId/tags
```

---

## 10. Technical Edge Cases & Considerations (Indian Context)

1. **Low Bandwidth:** Image compression before upload is non-negotiable. Teachers on 1.5GB daily data caps will abandon the app if a 10-photo upload consumes 50MB. Always compress to < 500KB per photo on-device before uploading.
2. **Speed & Offline Attendance:** The attendance flow must work entirely offline and sync silently. A teacher cannot wait for a loading spinner in front of 50 noisy students. Pre-load the student list when the teacher opens the app (not only when they open the attendance screen).
3. **Defaults Matter:** Date defaults to today. All students default to Present. Subject defaults to the teacher's most recently used subject. Reducing required taps from 5 to 2 is the difference between consistent use and abandonment.
4. **WhatsApp Fallback:** Give teachers an option to "Share Link to WhatsApp" for urgent notes/notices during the transition period. This is a bridge, not a crutch — track usage and phase it out once adoption is proven.
5. **Data Entry Friction:** Teachers hate entering the same information twice. The note reuse feature (assigning a prior year note to a new class) saves meaningful time for teachers who teach multiple sections.
6. **Notification Fatigue:** Teachers sending too many notifications will cause students to turn off push. Dashboard analytics showing open rates helps teachers self-moderate.
