> **Note:** This persona was written before the MVP scope was locked. MVP features: Auth (Phone+OTP), Attendance, Homework/Notes, Timetable, Fees (view only), Bus Tracking, Notifications, Leave (view only). Features like PixelTrace, Library, Report Cards, and Search are post-MVP. See `MVP.md` for the canonical spec. The technical implementation details (schemas, API patterns, auth flows) in this file remain useful reference for implementation.

# Student Persona: Features & Technical Implementation

## Persona Overview
**The Student** is the primary end-user of the StudyDash mobile app. They need a single, reliable source of truth for their daily school life.
In the Indian context, they might have shared devices (using a parent's phone), face intermittent internet connectivity, and require a multi-lingual interface. They are highly motivated by the **PixelTrace** feature to find their event photos.

---

## 1. Authentication & Session Management

### Features
- **OTP Login:** Phone number + 6-digit OTP sent via SMS (no passwords for students).
- **Secure Sign-Out:** Clears device token and refresh token, preventing push notifications to a logged-out device — critical for shared devices.
- **Short-Lived Sessions:** Access token expires in 15 minutes; refresh token lasts 30 days. Silent re-auth in background.
- **Multi-Language Onboarding:** Language selection screen on first launch.

### Technical Implementation

#### JWT Token Structure
```typescript
interface JwtPayload {
  sub: string;        // userId
  role: 'STUDENT';
  schoolId: string;   // Multi-tenant scoping — every query filters by this
  classId: string;    // Injected for performance — avoids join on every request
  iat: number;
  exp: number;        // 15 minutes
}
```

#### Database Schema (Prisma)
```prisma
model User {
  id           String     @id @default(uuid())
  role         UserRole   // STUDENT
  name         String
  phoneNumber  String?    @unique
  status       UserStatus @default(ACTIVE)
  schoolId     String
  createdAt    DateTime   @default(now())

  studentProfile StudentProfile?
  refreshTokens  RefreshToken[]
  devices        DeviceToken[]
}

model RefreshToken {
  id        String   @id @default(uuid())
  userId    String
  token     String   @unique  // Stored hashed (bcrypt)
  expiresAt DateTime
  revokedAt DateTime?         // Set on sign-out

  user      User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}

model DeviceToken {
  id            String   @id @default(uuid())
  userId        String
  expoPushToken String   @unique
  platform      String   // 'android' | 'ios'
  lastSeen      DateTime @default(now())
  isActive      Boolean  @default(true)

  user          User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, isActive])
}

model StudentProfile {
  id          String   @id @default(uuid())
  userId      String   @unique
  classId     String
  admissionNo String   @unique
  rollNo      Int
  parentId    String?

  user        User          @relation(fields: [userId], references: [id])
  class       Class         @relation(fields: [classId], references: [id])
  parent      ParentProfile? @relation(fields: [parentId], references: [id])
  consent     ConsentRecord?

  @@unique([classId, rollNo])
  @@index([classId])
}

model UserPreferences {
  id               String   @id @default(uuid())
  userId           String   @unique
  language         String   @default("en")      // 'en' | 'hi' | 'mr' | 'ta'
  theme            String   @default("light")   // 'light' | 'dark'
  fontSize         String   @default("medium")  // 'small' | 'medium' | 'large'
  quietHoursStart  String?  // "22:00"
  quietHoursEnd    String?  // "07:00"
  wifiOnlyDownloads Boolean @default(false)

  user             User @relation(fields: [userId], references: [id])
}
```

#### Backend API (NestJS)
- **`POST /api/v1/auth/otp/send`** — validates phone number, generates 6-digit OTP, stores hashed in Redis as `otp:{phoneNumber}` with TTL 5 minutes, sends via MSG91 (DLT-registered template).
- **`POST /api/v1/auth/otp/verify`** — validates OTP, returns `{ accessToken, refreshToken }`.
- **`POST /api/v1/auth/refresh`** — validates refresh token against DB, issues new access token.
- **`POST /api/v1/auth/logout`** — revokes `RefreshToken`, sets `DeviceToken.isActive = false`.

#### Frontend (React Native + Expo)
- **Screen:** `(auth)/login.tsx` → `(auth)/otp-verify.tsx`
- **Token Storage:** `react-native-mmkv` (encrypted, faster than AsyncStorage) — key: `auth_tokens`.
- **Auto sign-out on token expiry:** `axios` interceptor attempts refresh on 401; if refresh fails, navigates to login.
- **Shared Device UX:** Prominent "Sign Out" button in Profile. On sign-out, immediately clears local SQLite data and MMKV tokens so next user starts fresh.

---

## 2. Dashboard (Home Screen)

The central hub for a student's day.

### Features
- **Today's Overview:** Current day's timetable periods, pending assignments due today, attendance status, unread notifications count.
- **Weekly Snapshot:** Upcoming deadlines, tests, and events (e.g., "Sports Day on Friday").
- **Quick Actions:** One-tap access to "Recent Notes", "Mark Assignment Done", "Open PixelTrace".
- **Offline Banner:** Shows "Last synced: X mins ago" badge when device is offline.

### Technical Implementation

#### Database Schema
The dashboard aggregates data from `Assignment`, `AttendanceRecord`, `Note`, `Notification`, and `TimetablePeriod`. No dedicated table.

#### Backend API (NestJS)
- **Endpoint:** `GET /api/v1/students/dashboard`
- **Controller/Service:** `StudentDashboardController`, `StudentDashboardService`
- **Response shape:**
```typescript
interface DashboardResponse {
  greeting: string;                     // "Good Morning, Arjun!"
  attendance: {
    todayStatus: AttendanceStatus | null;
    percentage: number;
    isLow: boolean;                      // Below school threshold (default 75%)
  };
  assignments: {
    dueToday: AssignmentSummary[];
    overdue: AssignmentSummary[];
    upcomingCount: number;
  };
  recentNotes: NoteSummary[];            // Last 3 uploaded for student's class
  unreadNotificationCount: number;
  todayPeriods: TimetablePeriod[];       // Today's classes
}
```
- **Caching:** Redis key `student:dashboard:{studentId}` — TTL 5 minutes. Invalidated when new `Assignment`, `Note`, or `Notification` row is created for the student's class.

#### Frontend (React Native + Expo)
- **Screen:** `(app)/student/dashboard.tsx`
- **State:** `useQuery(['studentDashboard', studentId], fetchDashboard)`
- **UI Components:**
```typescript
<GreetingHeader name={user.name} avatarUrl={user.avatar} />
<AttendancePill status={attendance.todayStatus} percentage={attendance.percentage} />
<TodayAgendaWidget periods={todayPeriods} />
<AssignmentDueTodayList items={assignments.dueToday} />
<RecentNotesRow notes={recentNotes} />
<QuickActionGrid />
```
- **Offline First:** Cache last fetched dashboard JSON in `expo-sqlite` table `student_dashboard_cache(data JSON, cachedAt DATETIME)`. Show `<OfflineBanner lastSynced={cachedAt} />` if offline.

---

## 3. Timetable

### Features
- **Daily View:** Ordered list of today's periods (subject, teacher, time slot).
- **Weekly View:** Horizontal scroll across Monday–Saturday.
- **Current Period Highlight:** Automatically highlights the ongoing class based on IST time.
- **Calendar Export:** Export schedule as `.ics` to Google/Apple Calendar.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model TimetablePeriod {
  id        String  @id @default(uuid())
  schoolId  String
  classId   String
  subjectId String
  teacherId String
  dayOfWeek Int     // 1=Mon ... 6=Sat
  periodNo  Int     // 1-8
  startTime String  // "09:00" (IST, 24h)
  endTime   String  // "09:45"
  sessionId String

  class     Class   @relation(fields: [classId], references: [id])
  subject   Subject @relation(fields: [subjectId], references: [id])

  @@unique([classId, dayOfWeek, periodNo, sessionId])
  @@index([classId, dayOfWeek])
}
```

#### Backend API (NestJS)
- **`GET /api/v1/students/timetable?day=1`** — returns periods for a given day of week.
- **`GET /api/v1/students/timetable/week`** — full week grid.

#### Frontend
```typescript
// Determine current period
const now = new Date(); // IST
const currentPeriod = periods.find(p =>
  parseTime(p.startTime) <= now && now < parseTime(p.endTime)
);

// Calendar export (ical-generator)
// Triggered via expo-sharing to share .ics file
```

---

## 4. Assignments

Replacing WhatsApp groups and scattered diaries with a structured assignment feed.

### Features
- **Daily/Weekly Feed:** Filter by "Due Today", "Overdue", "Completed", "All".
- **Details & Attachments:** View instructions and open/download attached PDFs and images.
- **Status Tracking:** Mark as "Complete" (client-side toggle with server sync).
- **Personal Notes:** Add private text notes on any assignment (visible only to the student).
- **Reminders:** Local push notification 12 hours before due date.
- **Late/Missed Highlighting:** Overdue assignments shown with a red indicator.

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

// Student's private notes on an assignment — never visible to teacher
model AssignmentNote {
  id           String   @id @default(uuid())
  assignmentId String
  studentId    String
  content      String
  updatedAt    DateTime @updatedAt

  @@unique([assignmentId, studentId])
}
```

#### Backend API (NestJS)
- **`GET /api/v1/students/assignments?filter=pending|overdue|completed&page=1&limit=20`**
- **`POST /api/v1/students/assignments/:id/complete`** — upsert `AssignmentCompletion`
- **`DELETE /api/v1/students/assignments/:id/complete`** — remove completion (mark undone)
- **`PUT /api/v1/students/assignments/:id/note`** — upsert `AssignmentNote`

#### Frontend (React Native)
- **Hook:** `useAssignments(filter)` using `useInfiniteQuery` for pagination
- **Component:** `<AssignmentCard />` with completion checkbox, due date chip, attachment count badge
- **Local Notifications:**
```typescript
// Schedule on first fetch of assignment list
for (const assignment of assignments) {
  const reminderTime = subHours(new Date(assignment.dueDate), 12);
  if (reminderTime > new Date()) {
    await Notifications.scheduleNotificationAsync({
      content: { title: 'Assignment Due Tomorrow', body: assignment.title },
      trigger: reminderTime,
    });
  }
}
```
- **Offline Queue:** Store "mark complete" actions in SQLite `assignment_action_queue(assignmentId, action, queuedAt)`. React Query's `onMutate` applies optimistic update; queue flushes on reconnect via `NetInfo` listener.

---

## 5. Class Notes

Access daily study materials uploaded by teachers.

### Features
- **Feed:** Chronological list of notes, filterable by subject and date range.
- **Search:** Full-text search by keyword across note titles.
- **Recently Uploaded Section:** Highlights notes added in the last 24 hours.
- **Viewer:** In-app PDF and image preview (no external app required).
- **Offline Access:** "Download" button saves notes locally for offline viewing.
- **Bookmarks:** Save important notes to a personal bookmarks collection.

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
  fileUrl       String   // CDN URL
  thumbnailUrl  String?  // WebP thumbnail, generated by worker
  fileType      String   // 'pdf' | 'image' | 'doc'
  fileSizeBytes Int      // Warn user before large downloads
  uploadedAt    DateTime @default(now())

  views         NoteView[]
  bookmarks     NoteBookmark[]

  @@index([classId, subjectId, uploadedAt])
}

model NoteView {
  id        String   @id @default(uuid())
  noteId    String
  studentId String
  viewedAt  DateTime @default(now())

  @@unique([noteId, studentId])
}

model NoteBookmark {
  id        String   @id @default(uuid())
  noteId    String
  studentId String
  savedAt   DateTime @default(now())

  @@unique([noteId, studentId])
}
```

#### Backend API (NestJS)
- **`GET /api/v1/students/notes?subjectId=&page=1&limit=20`**
- **`GET /api/v1/students/notes/search?q={keyword}&subjectId=&from=&to=`** — PostgreSQL full-text search using `tsvector` GIN index on `title`.
- **`POST /api/v1/students/notes/:id/view`** — upsert `NoteView` (fires read receipt to teacher side).
- **`POST /api/v1/students/notes/:id/bookmark`**
- **`DELETE /api/v1/students/notes/:id/bookmark`**
- **Storage:** Pre-signed S3 URLs or direct CDN links for `fileUrl`. Download endpoint returns a pre-signed URL with 1-hour expiry.

#### Frontend (React Native)
- **PDF Viewer:** `react-native-pdf` with `expo-file-system` cache directory `FileSystem.documentDirectory + 'notes/'`
- **Image Viewer:** Standard `<Image />` with `react-native-zoom-reanimated`
- **Downloads:**
```typescript
// Check Wi-Fi preference before downloading
const netInfo = await NetInfo.fetch();
if (prefs.wifiOnlyDownloads && netInfo.type !== 'wifi') {
  Alert.alert('Wi-Fi Only', 'Enable downloads on mobile data in Settings?');
  return;
}
const downloadPath = FileSystem.documentDirectory + `notes/${note.id}.${ext}`;
await FileSystem.downloadAsync(note.fileUrl, downloadPath);
```
- **Bookmarks:** Local SQLite table `note_bookmarks(noteId, savedAt)` for instant UI, synced to server async.

---

## 6. Attendance

Tracking physical presence and alerting on trends.

### Features
- **Daily Status:** Present, Absent, Half-Day, Late, On Leave — clearly labeled.
- **Calendar View:** Monthly visual representation using colored dots per day.
- **Statistics:** Current academic year attendance percentage (e.g., 85%).
- **Alerts:** UI warning if attendance drops below the school's mandatory threshold (default 75% for CBSE).
- **History:** Full absent/late history with remarks (e.g., "Sick — informed by parent").

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
  ON_LEAVE  // Set when a LeaveRequest is APPROVED
}
```

#### Backend API (NestJS)
- **`GET /api/v1/students/attendance/summary?month=2024-11`** → `{ days: [{date, status, remarks}], monthPercentage, yearPercentage, threshold: 75, isLow: boolean }`
- **`GET /api/v1/students/attendance/history?sessionId=`** — full academic year records

#### Frontend (React Native)
```typescript
// react-native-calendars
const STATUS_COLORS = {
  PRESENT: '#16a34a', ABSENT: '#dc2626',
  LATE: '#f97316', HALF_DAY: '#eab308', ON_LEAVE: '#6366f1'
};

const markedDates = records.reduce((acc, r) => ({
  ...acc,
  [r.date]: { selected: true, selectedColor: STATUS_COLORS[r.status] }
}), {});

// Threshold warning
{attendance.isLow && (
  <WarningBanner
    message={`Attendance is ${percentage}% — below the required ${threshold}%`}
  />
)}
```
- **`<CircularProgress value={percentage} threshold={75} />`** using `react-native-svg`.

---

## 7. Notifications

Centralized alerts replacing SMS spam and WhatsApp groups.

### Features
- **Inbox:** Segregated by "School" (Principal announcements) and "Class" (Teacher notices).
- **Push Notifications:** Immediate alerts for urgent items (e.g., "Rain Holiday").
- **Read/Unread State:** Visual distinction; unread count shown on tab bar badge.
- **Priority Labels:** URGENT (red badge) and NORMAL.
- **Quiet Hours:** User-configurable silent period (e.g., 10 PM – 7 AM) — no push during this window.
- **Multi-lingual Body:** Critical notices optionally include a Hindi (`bodyHi`) translation.
- **Deep Links:** Tapping a push notification navigates directly to the relevant screen.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model Notification {
  id          String        @id @default(uuid())
  schoolId    String
  title       String
  body        String
  bodyHi      String?       // Optional Hindi translation
  priority    PriorityLevel // URGENT | NORMAL
  targetType  TargetType    // SCHOOL | CLASS | INDIVIDUAL
  targetId    String?       // classId or userId
  senderId    String
  createdAt   DateTime      @default(now())

  states      UserNotificationState[]

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

  @@unique([userId, notificationId])
  @@index([userId, isRead])
}

enum PriorityLevel { URGENT NORMAL }
enum TargetType    { SCHOOL CLASS INDIVIDUAL }
```

#### Backend API (NestJS)
- **`GET /api/v1/notifications?filter=school|class&page=1&limit=20`**
- **`POST /api/v1/notifications/:id/read`**
- **`POST /api/v1/notifications/read-all`**
- **Push Delivery Flow:**
  1. Sender creates notification via API.
  2. `NotificationService` inserts row, enqueues `BullMQ` job on `notifications` queue.
  3. Worker fetches all `DeviceToken` rows for target audience.
  4. Check `UserPreferences.quietHoursStart/End` — if user is in quiet hours, delay delivery.
  5. Sends via `expo-server-sdk-node` in batches of 100.
  6. Updates `UserNotificationState.deliveredAt`.

#### Frontend (React Native)
```typescript
// App.tsx — foreground and background handlers
Notifications.addNotificationReceivedListener(notification => {
  queryClient.invalidateQueries(['notifications']);
  updateTabBadge();
});

Notifications.addNotificationResponseReceivedListener(response => {
  const { type, id } = response.notification.request.content.data;
  if (type === 'ATTENDANCE_ABSENT') router.push('/student/attendance');
  if (type === 'NEW_ASSIGNMENT')    router.push(`/student/assignments/${id}`);
  if (type === 'NEW_NOTE')          router.push(`/student/notes/${id}`);
});
```

---

## 8. PixelTrace (Event Photo Discovery)

The "killer app" feature — finding yourself in school event photos.

### Features
- **Event Albums:** Photos organized by event (e.g., "Annual Day 2024").
- **Privacy Gate:** Only shows photos where the student is tagged (verified). No untagged photo browsing.
- **Consent Required:** PixelTrace tab is hidden until parent has set `pixelTraceOptIn = true`. A consent prompt is shown to guide the parent.
- **Mismatch Reporting:** A "This is not me" button on every photo for incorrect tags.
- **Favorites:** Save favourite event photos to a local collection.
- **Share:** Generate a short-lived (1-hour) pre-signed CDN link to share a photo.

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
  createdById   String
  createdAt     DateTime @default(now())

  photos        EventPhoto[]

  @@index([schoolId, date])
}

model EventPhoto {
  id           String   @id @default(uuid())
  eventId      String
  schoolId     String
  photoUrl     String   // Full-res CDN URL
  thumbnailUrl String   // 300px WebP CDN URL (generated by worker)
  uploadedById String
  uploadedAt   DateTime @default(now())
  isPublished  Boolean  @default(true)  // Admin can unpublish

  event        Event      @relation(fields: [eventId], references: [id])
  tags         PhotoTag[]

  @@index([eventId])
}

model PhotoTag {
  id           String   @id @default(uuid())
  photoId      String
  studentId    String
  taggedById   String
  confidence   Float?   // null for manual; 0-1 for AI (v2)
  isVerified   Boolean  @default(false)
  flaggedWrong Boolean  @default(false)
  flaggedAt    DateTime?
  createdAt    DateTime @default(now())

  photo        EventPhoto     @relation(fields: [photoId], references: [id])
  student      StudentProfile @relation(fields: [studentId], references: [id])

  @@unique([photoId, studentId])
  @@index([studentId, isVerified])
  @@index([flaggedWrong])
}

// DPDP Act compliance — parent must opt in before student can use PixelTrace
model ConsentRecord {
  id               String   @id @default(uuid())
  studentId        String   @unique
  parentId         String
  pixelTraceOptIn  Boolean  @default(false)
  consentedAt      DateTime?
  ipAddress        String?  // Audit proof
  updatedAt        DateTime @updatedAt

  student          StudentProfile @relation(fields: [studentId], references: [id])
}
```

#### Backend API (NestJS)
- **`GET /api/v1/pixeltrace/events`** — list events where the student has verified photos
- **`GET /api/v1/pixeltrace/events/:eventId/my-photos`** — photos where `studentId = me AND isVerified = true AND flaggedWrong = false AND isPublished = true`
- **`POST /api/v1/pixeltrace/tags/:tagId/flag`** — sets `flaggedWrong = true`, `flaggedAt = now()`, enqueues moderation alert to Admin.
- **Guard:** `PixelTraceConsentGuard` on all PixelTrace routes — returns `403` with `{ requiresConsent: true }` if opt-in is false. Frontend shows a prompt directing parent to their consent screen.

#### Frontend (React Native)
```typescript
// Screen: (app)/student/pixeltrace/[eventId].tsx
// Masonry grid: @shopify/flash-list with numColumns={2}
// Full-screen viewer: react-native-image-viewing

// Favorites stored in local SQLite: favorite_photos(photoId, savedAt)
// Share: call GET /pixeltrace/photos/:photoId/share-link → pre-signed URL (1hr TTL)

// "Not me" button in image viewer header
<TouchableOpacity onPress={() => flagMutation.mutate(tag.id)}>
  <Text style={{ color: 'red' }}>This is not me</Text>
</TouchableOpacity>
```

---

## 9. Profile & Settings

### Features
- **Student Profile View:** Name, class/section, roll number, admission number, profile photo.
- **Language Preference:** Switch between English and Hindi (Marathi, Tamil in future).
- **Theme:** Light / Dark mode.
- **Accessibility:** Font size selector (Small / Medium / Large).
- **Quiet Hours:** Configure notification silent window.
- **Wi-Fi Only Downloads:** Toggle to restrict note downloads to Wi-Fi.
- **Device Management:** View logged-in devices; sign out individual sessions.
- **Password/Security:** Change phone number (requires OTP re-verification).

### Technical Implementation

#### Backend API (NestJS)
- **`GET /api/v1/users/me`** → `{ user, studentProfile, preferences }`
- **`PATCH /api/v1/users/me/preferences`** — updates `UserPreferences`
- **`GET /api/v1/users/me/devices`** — list active `DeviceToken` rows
- **`DELETE /api/v1/users/me/devices/:deviceId`** — revoke a session (sign out other device)

#### Frontend
- **Language:** `react-i18next` with lazy-loaded locale JSON files: `locales/en.json`, `locales/hi.json`. Locale stored in `UserPreferences` and synced across devices.
- **Theme:** `ThemeContext` provider wrapping root navigator. Persisted in MMKV.

---

## 10. Global Search

### Features
- **Unified Search:** Single search bar covering assignments, notes, and notifications.
- **Filters:** Narrow by type (Assignment / Note / Notification), subject, date range.
- **Recent Searches:** Last 10 queries stored locally.

### Technical Implementation
- **`GET /api/v1/search?q=quadratic&types=assignment,note`** — PostgreSQL full-text search using GIN-indexed `tsvector` columns on `Assignment.title`, `Note.title`.
- **Response:** Results grouped by type, each with `{ type, id, title, subtitle, createdAt }`.
- **Frontend:** Debounced 300ms input; results grouped into sections. Recent searches in MMKV key `recent_searches`.

---

## 12. Library

### Features
- **Browse Catalog:** Search books by title, author, or subject. See availability (copies available / total copies).
- **My Books:** View currently borrowed books with due dates and overdue status.
- **Digital Resources:** Access NCERT PDFs, sample papers, and question banks filtered by subject.
- **Overdue Reminder:** Push notification 1 day before due date and on the day of.

### Technical Implementation
- **`GET /api/v1/students/library/catalog?q=&subjectId=&category=&page=`** — paginated book search
- **`GET /api/v1/students/library/my-books`** → `[{ bookTitle, issuedAt, dueDate, isOverdue, fine }]`
- **`GET /api/v1/students/library/resources?subjectId=&classId=`** — digital resources, same S3/CDN as notes
- Student cannot self-issue books — issue/return done by librarian. Student view is read-only for catalog and borrowing status.
- Overdue reminder: BullMQ scheduled job created at issue time; `expo-notifications` local notification as backup.

---

## 13. Report Card

### Features
- **Board-Specific Report Card:** View report card in school's board format (CBSE/ICSE/State Board) per academic session.
- **Subject-Wise Marks:** Each subject's marks per exam type with computed final grade.
- **Co-Scholastic Summary (CBSE):** Activity, discipline, and value grades.
- **Download PDF:** Save for offline reference or sharing.

### Technical Implementation
- **`GET /api/v1/students/report-cards?sessionId=`** → `[{ session, subjectResults, coScholastic, reportCardUrl }]`
- `reportCardUrl`: 1-hour pre-signed S3 URL — `expo-sharing` opens native PDF viewer.
- Students see only their own data — `JwtAuthGuard` + student `sub` scoped to their `studentId` only.

---

## 11. Technical Edge Cases & Considerations (Indian Context)

1. **Shared Devices:** `SecureStore` / MMKV encrypted — tokens not accessible to other apps. Prominent sign-out on profile. Short-lived access tokens (15 min) reduce risk window.
2. **App Size:** Target APK < 30MB. Use Expo's lazy import for heavy screens (PixelTrace gallery). Use WebP for all images. Avoid bundling large font families.
3. **Language:** `react-i18next` with lazy-loaded locale bundles. Initial languages: English, Hindi. UI must right-to-left ready even if not immediately needed.
4. **Data Usage:** WebP images throughout. "Download only on Wi-Fi" setting. Pre-signed CDN URLs with cache headers for notes (immutable, 1-year cache-control).
5. **Low-End Devices:** `FlashList` (Shopify) instead of FlatList for all long lists — significantly better performance on budget Androids. Avoid heavy animations on low-end device detection.
6. **Connectivity:** `@react-native-community/netinfo` listeners throughout. All mutations queue offline, flush on reconnect. No spinner blocking critical flows.
