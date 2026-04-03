> **Note:** This persona was written before the MVP scope was locked. MVP features: Auth (Phone+OTP), Multi-child support, Attendance viewing, Homework/Notes (read-only), Fees payment (Razorpay), Bus tracking, Leave requests, Complaint tickets, Notifications. Features like PixelTrace, PTM booking, Library, Report cards, and WhatsApp alerts are post-MVP. See `MVP.md` for the canonical spec. The technical implementation details in this file remain useful reference.

# Parent Persona: Features & Technical Implementation

## Persona Overview
**The Parent** requires transparency and peace of mind regarding their child's academic progress, attendance, and administrative obligations (like fees). They are busy, often viewing the app quickly on their mobile device.
In the Indian context, a core requirement is **Multi-Child Support** (siblings in the same school but different grades) and seamless **UPI/Online Fee Payment** flows. They rely heavily on Push/SMS notifications to stay informed.

---

## 1. Authentication & Session Management

### Features
- **OTP-Only Login:** Phone number + 6-digit SMS OTP — no password. Passwords are a support nightmare for parents.
- **Single Number, Multiple Children:** One phone number maps to a single `ParentProfile` which can have multiple children linked. Both mother and father using the same number maps to the same household account.
- **Persistent Sessions:** 30-day refresh token — parents should rarely have to log in again.
- **Secure Sign-Out:** Clears device token so no further push notifications arrive on that device.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model User {
  id          String     @id @default(uuid())
  role        UserRole   // PARENT
  name        String
  phoneNumber String?    @unique  // Single source of truth for parent identity
  status      UserStatus @default(ACTIVE)
  schoolId    String

  parentProfile ParentProfile?
  refreshTokens RefreshToken[]
  devices       DeviceToken[]
}

model ParentProfile {
  id           String   @id @default(uuid())
  userId       String   @unique
  relationship String?  // "Father" | "Mother" | "Guardian"

  user         User             @relation(fields: [userId], references: [id])
  children     StudentProfile[] // One ParentProfile → Many StudentProfiles
  consent      ConsentRecord[]
}

model StudentProfile {
  id          String   @id @default(uuid())
  userId      String   @unique
  classId     String
  admissionNo String   @unique
  rollNo      Int
  parentId    String?  // FK to ParentProfile

  user        User          @relation(fields: [userId], references: [id])
  class       Class         @relation(fields: [classId], references: [id])
  parent      ParentProfile? @relation(fields: [parentId], references: [id])
  consent     ConsentRecord?
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
```

#### JWT Token Structure
```typescript
interface ParentJwtPayload {
  sub: string;       // userId
  role: 'PARENT';
  parentId: string;  // ParentProfile.id
  schoolId: string;
  iat: number;
  exp: number;       // 30 minutes
}
```

#### Backend API (NestJS)
- **`POST /api/v1/auth/otp/send`** — validates phone number, stores hashed OTP in Redis (`otp:{phoneNumber}`, TTL 5 min), sends via MSG91 (DLT-registered template required by TRAI).
- **`POST /api/v1/auth/otp/verify`** — validates OTP, returns `{ accessToken, refreshToken }`.
- **Guard:** `ParentOwnsChildGuard` — on every `/parents/children/:childId/` route, verifies `StudentProfile.parentId === req.user.parentId`. Prevents cross-family data access.

---

## 2. Multi-Child Dashboard (The "Switcher")

Parents often have 2–3 children in the same school across different grades.

### Features
- **Child Selector:** Horizontal scrollable row of children with name and class (e.g., "Aarav · 5A", "Diya · 8B"). Active child shown with a highlight ring.
- **Unified Alerts Feed:** Combined notification inbox, each item clearly labeled with the child's name.
- **Summary Cards:** Quick glance at today's attendance, upcoming fee dues, and pending assignment count for the selected child.
- **Deep-Link Switching:** Tapping an "Aarav was absent" push notification automatically switches to Aarav and opens the attendance screen.

### Technical Implementation

#### Backend API (NestJS)
- **`GET /api/v1/parents/me`** → `{ profile, children: [{ id, name, grade, section, admissionNo, rollNo }] }`
- All child-specific routes take `/:childId/` as a path parameter.
- Backend validates `ParentOwnsChildGuard` on every such route.

#### Frontend (React Native + Expo)
```typescript
// Zustand store
interface ParentStore {
  selectedChildId: string;
  children: ChildSummary[];
  setSelectedChild: (id: string) => void;
}

// Child switcher component (always visible in app header)
<ScrollView horizontal showsHorizontalScrollIndicator={false}>
  {children.map(child => (
    <Pressable
      key={child.id}
      onPress={() => setSelectedChild(child.id)}
      style={child.id === selectedChildId ? styles.active : styles.inactive}
    >
      <Avatar name={child.name} />
      <Text>{child.name} · {child.grade}{child.section}</Text>
    </Pressable>
  ))}
</ScrollView>

// All subsequent screens read selectedChildId from Zustand
```

---

## 3. Attendance Monitoring

### Features
- **Absence Push Alert:** Immediate push notification when the child is marked absent during first period (within ~60 seconds of the teacher submitting attendance).
- **Daily Status:** View today's attendance status (Present / Absent / Late / On Leave).
- **Monthly Calendar View:** Color-coded calendar (green = present, red = absent, orange = late).
- **Year Percentage:** Running academic year attendance percentage with threshold warning (< 75%).
- **Deep Link on Tap:** Tapping the "absent" push notification opens the Attendance screen for that child.

### Technical Implementation

#### Database Schema (Prisma)
No parent-specific tables — queries against `AttendanceRecord` using the child's `studentId`.

```prisma
// AttendanceRecord (owned by Teacher persona, queried by Parent)
model AttendanceRecord {
  id         String           @id @default(uuid())
  schoolId   String
  studentId  String
  classId    String
  date       DateTime         @db.Date
  status     AttendanceStatus
  remarks    String?
  markedById String

  @@unique([studentId, date])
  @@index([studentId, date])
}

enum AttendanceStatus {
  PRESENT ABSENT LATE HALF_DAY ON_LEAVE
}
```

#### Absence Alert Push Flow
```
1. Teacher submits bulk attendance via POST /teachers/attendance/bulk
2. AttendanceService post-transaction hook: find all ABSENT records from this submission
3. Enqueue BullMQ job on `attendance-alerts` queue:
   { studentId, status: 'ABSENT', date, classId }
4. Worker:
   a. SELECT StudentProfile WHERE id = studentId → get parentId
   b. SELECT ParentProfile WHERE id = parentId → get userId
   c. SELECT DeviceToken WHERE userId = parentUserId AND isActive = true
   d. Check UserPreferences.quietHoursStart/End — skip push if in quiet hours
   e. Send via expo-server-sdk-node:
      { title: "Attendance Alert", body: "Aarav (10A) was marked absent today." }
   f. Deep-link data: { type: "ATTENDANCE_ABSENT", childId: studentId }
5. Create UserNotificationState row for the parent's userId
```

#### Backend API (NestJS)
- **`GET /api/v1/parents/children/:childId/attendance/summary?month=2024-11`** → `{ days: [{date, status, remarks}], monthPercentage, yearPercentage, threshold, isLow }`
- **`GET /api/v1/parents/children/:childId/attendance/today`**

#### Frontend (React Native)
```typescript
// react-native-calendars
const STATUS_COLORS = {
  PRESENT: '#16a34a', ABSENT: '#dc2626',
  LATE: '#f97316', HALF_DAY: '#eab308', ON_LEAVE: '#6366f1'
};

// Push tap deep-link handler (App.tsx)
Notifications.addNotificationResponseReceivedListener(response => {
  const { type, childId } = response.notification.request.content.data;
  if (type === 'ATTENDANCE_ABSENT') {
    store.setSelectedChild(childId);
    router.push('/parent/attendance');
  }
});
```

---

## 4. Assignment & Academic Monitoring

### Features
- **Assignment Viewer:** Read-only view of daily homework for the selected child.
- **Completion Visibility:** See if the child has marked an assignment "done" — motivates parents to follow up.
- **Overdue Highlight:** Assignments past the due date without completion shown in red.
- **Note Visibility:** Parents can see teacher-published instructions but not the child's personal notes.

### Technical Implementation

No parent-specific tables — queries `Assignment` and `AssignmentCompletion` using the child's `classId` and `studentId`.

#### Backend API (NestJS)
- **`GET /api/v1/parents/children/:childId/assignments/pending`**
- **`GET /api/v1/parents/children/:childId/assignments/completed`**
- Parent endpoints are **read-only** — no `POST /complete` endpoint exposed to parent role. `RolesGuard` enforces this.

#### Frontend (React Native)
```typescript
// Read-only assignment card — no checkbox interaction
<AssignmentCard
  assignment={a}
  isCompleted={a.completedByChild}
  readOnly={true}
  completionBadge={a.completedByChild ? "✓ Child marked done" : null}
/>
```

---

## 5. Fee Payments & Invoices (Critical India Flow)

### Features
- **Fee Dashboard:** View total dues for the year broken down by term/installment (Q1, Q2, Q3, Q4).
- **Outstanding Balance:** Clear display of amount due, due date, and applicable late fine.
- **UPI Integration:** "Pay Now" button opening Razorpay — supports GPay, PhonePe, Paytm natively.
- **Payment History & Receipts:** All past payments with "Download Receipt" PDF button (useful for 80C tax declarations in India).
- **Late Fine Calculation:** Automatically shows how fine accumulates per day after the due date.
- **Payment Confirmation Push:** Immediate push notification on payment success with transaction ID.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model FeeStructure {
  id             String   @id @default(uuid())
  schoolId       String
  classId        String?  // null = school-wide fee
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
  razorpayOrderId String?       @unique  // Prevents double-order creation

  student         StudentProfile @relation(fields: [studentId], references: [id])
  feeStructure    FeeStructure   @relation(fields: [feeStructureId], references: [id])
  payments        FeePayment[]

  @@index([studentId, status])
}

model FeePayment {
  id              String        @id @default(uuid())
  invoiceId       String
  amountPaid      Decimal       @db.Decimal(10, 2)
  paymentDate     DateTime      @default(now())
  method          PaymentMethod // RAZORPAY | CASH | CHEQUE | NEFT
  transactionId   String?       // Razorpay payment_id
  razorpayOrderId String?
  receiptUrl      String?       // S3 URL to generated PDF receipt

  invoice         FeeInvoice @relation(fields: [invoiceId], references: [id])

  @@index([invoiceId])
}

enum FeeType       { TUITION TRANSPORT EXAM LIBRARY DEVELOPMENT }
enum InvoiceStatus { PENDING PARTIAL PAID WAIVED }
enum PaymentMethod { RAZORPAY CASH CHEQUE NEFT }
```

#### Razorpay Integration Flow
```typescript
// Step 1: Parent taps "Pay Now"
// POST /api/v1/parents/fees/checkout
// Service:
//   1. Fetch FeeInvoice, validate status is PENDING/PARTIAL
//   2. Calculate total = amountDue - alreadyPaid + lateFine (computed at request time)
//   3. Check if razorpayOrderId already exists — idempotent, reuse if so
//   4. Call Razorpay API: razorpay.orders.create({ amount: total * 100, currency: 'INR' })
//   5. Save razorpayOrderId to FeeInvoice
//   6. Return { orderId, key: RAZORPAY_KEY_ID, amount, studentName, invoiceId }

// Step 2: Client opens Razorpay checkout
// react-native-razorpay handles GPay / PhonePe / Paytm / Cards / NetBanking

// Step 3: Webhook — payment result
// POST /api/webhooks/razorpay (NOT behind auth — verified by signature)
// Service:
//   1. Verify signature: crypto.createHmac('sha256', WEBHOOK_SECRET).update(rawBody).digest('hex')
//   2. If event === 'payment.captured':
//      a. UPDATE FeeInvoice SET status = 'PAID', paidAt = NOW()
//      b. INSERT FeePayment row
//      c. Enqueue `pdf-generation` BullMQ job for receipt
//      d. Push notification to parent: "Fee paid ✅ ₹12,500 — TXN: pay_XXXXX"
//   3. If event === 'payment.failed':
//      a. Push notification: "Payment failed. Please retry or contact school."
//   4. Idempotency: check if FeePayment with transactionId already exists before inserting
```

#### Frontend (React Native)
```typescript
import RazorpayCheckout from 'react-native-razorpay';

const handlePay = async (invoice: FeeInvoice) => {
  const { orderId, key, amount } = await createCheckoutMutation.mutateAsync(invoice.id);
  try {
    const data = await RazorpayCheckout.open({
      key,
      amount,
      currency: 'INR',
      order_id: orderId,
      name: school.name,
      description: invoice.feeStructure.name,
      prefill: { contact: user.phoneNumber },
      theme: { color: '#4F46E5' },
    });
    // data.razorpay_payment_id available here
    // Webhook handles actual DB update — show a "Payment processing" state
    router.push('/parent/fees/success');
  } catch (e) {
    if (e.code === 'PAYMENT_CANCELLED') return; // User backed out
    showErrorToast('Payment failed. Please try again.');
  }
};
```

#### Receipt Download
```typescript
// GET /api/v1/parents/fees/payments/:paymentId/receipt
// Returns a short-lived (1-hour) pre-signed S3 URL
// Frontend: expo-sharing to open the PDF in native PDF viewer
await Sharing.shareAsync(receiptUrl, { mimeType: 'application/pdf' });
```

---

## 6. School Communication & Notices

### Features
- **Notice Board:** Read-only feed of school-wide circulars (holidays, PTM dates, event invites).
- **Push Notifications:** Urgent notices (e.g., "Rain Holiday") delivered as immediate push.
- **Multi-Lingual Notices:** "Translate to Hindi" button on any notice using Google Cloud Translation API.
- **Acknowledgement:** For critical circulars, a "I have read and understood" confirm button (required for fee hikes, policy changes).
- **SMS Fallback:** If a parent hasn't opened an urgent notice within 1 hour, an SMS is sent via MSG91 as fallback.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model Notification {
  id          String        @id @default(uuid())
  schoolId    String
  title       String
  body        String
  bodyHi      String?       // Hindi translation (pre-generated or on-demand)
  priority    PriorityLevel // URGENT | NORMAL
  targetType  TargetType    // SCHOOL | CLASS | INDIVIDUAL
  targetId    String?
  senderId    String
  requiresAck Boolean       @default(false)
  requiresFallback Boolean  @default(false)  // Trigger SMS if unread in 1hr
  createdAt   DateTime      @default(now())

  states      UserNotificationState[]
}

model UserNotificationState {
  id             String    @id @default(uuid())
  userId         String
  notificationId String
  isRead         Boolean   @default(false)
  readAt         DateTime?
  deliveredAt    DateTime?
  acknowledgedAt DateTime?  // Set when parent taps "I understand"

  @@unique([userId, notificationId])
  @@index([userId, isRead])
}

enum PriorityLevel { URGENT NORMAL }
enum TargetType    { SCHOOL CLASS INDIVIDUAL }
```

#### Backend API (NestJS)
- **`GET /api/v1/notifications?filter=school|class&page=1&limit=20`** — parent sees only school-level and class-level (for their children's classes)
- **`POST /api/v1/notifications/:id/read`**
- **`POST /api/v1/notifications/:id/acknowledge`** — sets `acknowledgedAt`
- **`GET /api/v1/notifications/:id/translate?lang=hi`** — calls Google Cloud Translation API, caches result in `Notification.bodyHi`

#### SMS Fallback Worker (BullMQ)
```typescript
// Queue: sms-fallback (cron every 30 minutes)
// Finds Notification WHERE requiresFallback = true AND createdAt > 30min ago
// For each: find UserNotificationState WHERE isRead = false
// Calls MSG91 for each unread user's phoneNumber
// DLT Template: pre-registered, subject "Important School Notice"
```

---

## 7. Digital Consent Management (DPDP Act)

### Features
- **PixelTrace Consent Gate:** Parents must explicitly opt in for their child's face to be used in photo matching. Shown as a prominent modal before the child can access PixelTrace.
- **Consent History:** Parent can view what they consented to and when.
- **Revoke Anytime:** Revoking consent removes all AI-generated photo tags for the child and deletes the face enrollment record.

### Technical Implementation

#### Database Schema (Prisma)
```prisma
model ConsentRecord {
  id              String   @id @default(uuid())
  studentId       String   @unique
  parentId        String
  pixelTraceOptIn Boolean  @default(false)
  consentedAt     DateTime?
  ipAddress       String?   // For legal audit trail (DPDP requirement)
  updatedAt       DateTime  @updatedAt

  student         StudentProfile @relation(fields: [studentId], references: [id])
}
```

#### Backend API (NestJS)
- **`GET /api/v1/parents/children/:childId/consent`**
- **`PUT /api/v1/parents/children/:childId/consent`** → `{ pixelTraceOptIn: true }`
  - Records `consentedAt = now()` and `ipAddress` from `req.ip`.
  - On revoke (`false`): triggers background job to delete `StudentFaceEnrollment` and all AI-generated `PhotoTag` rows for the student.
- Guard `PixelTraceConsentGuard` on all PixelTrace student routes: returns `403 { requiresConsent: true }`.

#### Frontend
```typescript
// Consent modal — shown on first app open if consent not yet decided
// Blocks rest of app (DPDP Act privacy-first design)
<ConsentModal
  childName={child.name}
  onAccept={() => updateConsent(child.id, true)}
  onDecline={() => updateConsent(child.id, false)}
  policyUrl="https://school.studydash.in/privacy"
/>
// Modal cannot be dismissed without choosing Yes or No
```

---

## 8. Leave Requests

### Features
- **Submit Leave:** Enter reason and date range; submit on behalf of child.
- **Status Tracking:** View pending, approved, and rejected leave requests.
- **Approval Notification:** Push notification when the teacher approves or rejects.
- **Auto-Calendar:** Approved leaves appear as "On Leave" on the attendance calendar.

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
  reviewedById String?       // teacherId or adminId
  reviewNote   String?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt

  @@index([studentId, status])
}

enum RequestStatus { PENDING APPROVED REJECTED }
```

#### Backend API (NestJS)
- **`POST /api/v1/parents/children/:childId/leaves`** → `{ startDate, endDate, reason }`
- **`GET /api/v1/parents/children/:childId/leaves`** — history, most recent first
- On teacher approval: `AttendanceService` upserts `AttendanceRecord` with `ON_LEAVE` for each date in range. Push notification sent to parent: "Leave approved for Aarav (Nov 18–19)."

#### Frontend
```typescript
// (app)/parent/leaves/[childId]/new.tsx
// Date range picker (react-native-date-picker)
// Max leave duration: 10 consecutive days (configurable per school)
// Character limit on reason: 200 chars
```

---

## 9. PixelTrace — Parent View

Parents receive a read-only view of their child's tagged event photos, provided consent has been given.

### Features
- **Access Via Child Profile:** PixelTrace accessible under each child's section.
- **Consent Gate:** Cannot access until parent has opted in (same `ConsentRecord` gate as student).
- **Shared View:** Parent sees the same photos as the child — photos tagged with `isVerified = true`.
- **Download:** Save photos to device camera roll for family albums.

### Technical Implementation

#### Backend API (NestJS)
- **`GET /api/v1/parents/children/:childId/pixeltrace/events`**
- **`GET /api/v1/parents/children/:childId/pixeltrace/events/:eventId/photos`**
- Same underlying query as student PixelTrace — `PhotoTag WHERE studentId = childId AND isVerified = true`.
- `ParentOwnsChildGuard` + `PixelTraceConsentGuard` both applied.

---

## 11. WhatsApp Notifications

Parents receive automated WhatsApp messages for the events that matter most — the channel they already check.

### Features
- **Absence Alert via WhatsApp:** If push notification undelivered within 5 minutes, WhatsApp message sent: "Aarav (10A) was marked absent today."
- **Fee Reminder:** "Fee of ₹12,500 for Q2 is due on Nov 30. Pay here: [link]"
- **Homework Alert:** "New homework posted in Mathematics for 10A. Due: Nov 20."
- **Acknowledge:** Parent can reply "OK" to acknowledge a critical notice — logged as read receipt.

### Technical Implementation
- Delivery chain: Expo push → 5-min BullMQ delay check → if `UserNotificationState.deliveredAt` is null → Gupshup/Interakt WhatsApp API → if failed → MSG91 SMS
- Template IDs stored in `WhatsAppConfig.enabledTriggers` — only pre-approved TRAI templates sent
- Parent opt-out: `UserPreferences.whatsappOptOut boolean` — skips WhatsApp step if true

---

## 12. PTM Slot Booking

### Features
- **Browse Available Slots:** For each PTM event, parent sees teacher-wise time slots and books one.
- **Multi-Child Handling:** If parent has two children with the same teacher, system warns of conflict and suggests adjacent slots.
- **Booking Confirmation:** Push + WhatsApp confirmation with slot details after booking.
- **Cancellation:** Cancel up to 2 hours before PTM; slot released to next parent on waitlist.
- **Post-PTM Notes:** After the meeting, teacher's notes appear in the child's profile — parent can read but not edit.

### Technical Implementation
- **`GET /api/v1/parents/ptm/:ptmEventId/slots?teacherId=`** → available slots list
- **`POST /api/v1/parents/ptm/bookings`** → `{ slotId, childId }` — idempotent, prevents double-booking same slot
- **`DELETE /api/v1/parents/ptm/bookings/:id`** — cancellation with 2hr cutoff check
- **`GET /api/v1/parents/children/:childId/ptm/notes`** — post-PTM notes for the child

---

## 13. Report Card View

### Features
- **Board-Specific Report Card:** View child's report card in the format matching their school's board (CBSE/ICSE/State Board).
- **Subject-Wise Breakdown:** Marks per exam type and final computed grade.
- **Historical Report Cards:** Past sessions available for download.
- **Download PDF:** Save report card to device for offline reference or printing.

### Technical Implementation
- **`GET /api/v1/parents/children/:childId/report-cards?sessionId=`** → `[{ session, subjectResults, coScholastic, reportCardUrl }]`
- `reportCardUrl`: pre-signed S3 URL (1-hour TTL) to generated PDF
- `ParentOwnsChildGuard` enforced on all report card routes

---

## 14. Library — Child's Borrowing Status

### Features
- **Current Borrowed Books:** View books currently issued to the child with due dates.
- **Overdue Warning:** Red indicator if any book is overdue; shows accumulated fine.
- **Borrowing History:** Past issues and return dates.

### Technical Implementation
- **`GET /api/v1/parents/children/:childId/library/issues?status=active|all`** → issued books list
- Read-only — parent cannot issue or return books.

---

## 10. Technical Edge Cases & Considerations (Indian Context)

1. **Shared Phone Numbers:** The most common scenario — both parents use one number for school records. Authenticate the phone number, attaching it to a single `ParentProfile`. All children linked to that number are accessible from one account. Do not create two separate accounts for the same phone number.
2. **OTP Reliability:** Use MSG91 with DLT-approved template IDs to avoid TRAI blocking. Secondary fallback: Firebase Auth SMS. OTP should be 6 digits, 5-minute TTL. Rate limit: 5 OTP attempts per phone per hour.
3. **Regional Languages:** The parent app interface must support Hindi and one major regional language per state (Marathi for Maharashtra, Tamil for Tamil Nadu). Use `react-i18next` with lazy-loaded locale bundles. The "Translate Notice" feature fills the gap for school-generated content.
4. **App Size:** Parents use budget smartphones. Keep APK under 30MB — defer loading heavy assets, use system fonts, lazy-load PixelTrace gallery components.
5. **Payment Edge Cases:** Network drops during UPI payment are common. The Razorpay webhook is the source of truth — never update `FeeInvoice` based only on the client callback. If the webhook arrives but the client never got a success response, the payment history will show success on next app open via server state.
6. **SMS for Non-App Users:** Not all parents will install the app. Critical notices (emergency closures, fee deadlines) must fall back to SMS for all parents, regardless of app installation.
