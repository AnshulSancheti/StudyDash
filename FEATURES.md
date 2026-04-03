# StudyDash — Feature List

## MVP Features

### Student App (Mobile)

**Dashboard Home**
- Today overview (classes, assignments due, attendance status, new notifications)
- Weekly snapshot (upcoming deadlines)
- Quick actions (view notes, mark assignment done)

**Assignments**
- Daily and weekly assignment feed
- Subject/class filters
- Assignment details (instructions, attachments, due date)
- Mark as complete/incomplete
- Late/missed assignment highlighting
- Reminders before due time

**Class Notes**
- Notes list by date and subject
- File preview for PDF/images
- Download and offline access
- Bookmark important notes
- Search by keyword, class, date

**Attendance**
- Daily attendance status
- Monthly attendance calendar (color-coded)
- Attendance percentage and trends

**Timetable**
- Today view + weekly view
- Period details (subject, teacher, time)

**Bus Tracking**
- Map view of bus location
- ETA to their stop
- Push notification when bus is approaching

**Notifications**
- In-app notification inbox
- Push notifications for urgent announcements
- Read/unread states
- Notification filters (school, class, personal)

**Profile and Settings**
- Student profile view
- Class/section details
- Language preference
- Theme/accessibility settings

---

### Teacher App (Mobile)

**Assignments**
- Create, edit, delete assignments
- Attach files/resources
- View assignment completion summary per class

**Class Notes**
- Upload notes (PDF, images, docs)
- View download stats per note

**Attendance**
- Mark attendance per class/section (Present, Absent, Late)
- Edit within same day
- View class attendance summary

**Leave Management**
- View pending leave requests for their classes
- Approve or reject with remarks
- View leave history per student

**Communication**
- Send class-wide announcements
- View and respond to parent messages

**Timetable**
- View teaching schedule across all classes
- Today view + weekly view

**Notifications**
- Receive push notifications
- In-app notification history

---

### Parent App (Mobile)

**Multi-Child Support**
- Switch between children (if multiple enrolled)
- Per-child views for all features

**Attendance**
- View child's daily status and monthly calendar
- Attendance percentage
- Instant absence alert (push notification)

**Assignments & Notes**
- Read-only view of assignments and completion status
- Read-only view of class notes

**Fees**
- View outstanding fees (itemized)
- Pay online via Razorpay (full or partial)
- View payment history
- Download receipts (PDF)
- Fee due and overdue reminders

**Communication**
- View class announcements
- Submit complaints/queries (ticket-based)
- Track ticket status

**Leave Applications**
- Submit leave request for child
- View request history and status

**Bus Tracking**
- Live map view of child's bus
- ETA to stop
- "Bus approaching" notification

**Timetable**
- View child's timetable (read-only)

---

### Driver App (Mobile)

**Route Management**
- Start/end route
- View route stops in order
- GPS location streamed every 10 seconds

---

### Admin Web Panel

**School Onboarding**
- Setup wizard (school profile, classes, subjects, fee heads, routes)
- CSV bulk import (students, teachers, drivers)
- Validation with error reporting

**User Management**
- View/edit/deactivate students, teachers, parents, drivers
- Role and permission management
- Audit logs for sensitive actions

**Attendance**
- Compliance report (which teachers marked, which didn't)
- Class-wise summary
- Override/correct records

**Fee Management**
- Fee structure configuration per class
- Invoice generation (auto/manual)
- Record offline payments
- Payment status dashboard
- Defaulter reports
- Collection reports (CSV/PDF)

**Communication**
- School-wide broadcasts
- Complaint ticket management (assign, respond, resolve)
- Ticket dashboard with filters

**Timetable**
- Configure per class/section (periods, subjects, teachers, time slots)
- Break and lunch configuration

**Bus Tracking**
- Define routes and stops
- Assign drivers to routes
- Assign students to stops
- Live map of all active buses
- Route history

**Expense Tracker**
- Log expenses (amount, category, date, description, receipt)
- Category management (predefined + custom)
- Expense list with filters
- Monthly summary with pie chart
- Month-over-month comparison

**Analytics**
- Attendance trends
- Fee collection status
- Notification delivery metrics

---

## Post-MVP Features (in priority order)

1. **Pixel Trace Light** — Manual photo tagging, consent-gated viewing
2. **Pixel Trace AI** — Face recognition matching with moderation queue
3. **WhatsApp Integration** — Notification fallback chain (push → WhatsApp → SMS)
4. **School Email Provisioning** — Google Workspace / Microsoft 365 student emails
5. **PTM Scheduler** — Slot booking, reminders, post-meeting teacher notes
6. **Library Management** — Catalog, issue/return, fines, digital resources
7. **AI Attendance** — Classroom photo → auto roll call
8. **Board-Specific Report Cards** — CBSE/ICSE/State Board PDF generation
9. **Admission & Enrollment** — Online inquiry, waitlist, document upload, TC generation
10. **Advanced Analytics** — Multi-school dashboards, SIS integrations

---

## Platform-Wide Capabilities

| Capability | Description |
|---|---|
| **Push Notifications** | FCM + APNs for attendance, fees, messages, homework, bus alerts |
| **Offline Access** | Cached content accessible without internet; auto-syncs on reconnect |
| **Security** | JWT (RS256), bcrypt passwords, RBAC, input validation, MIME type checks |
| **Multi-Tenancy** | All data scoped by schoolId; S3 paths prefixed by schoolId |
| **Background Jobs** | BullMQ queues for notifications, PDFs, CSV processing, thumbnails |
| **File Storage** | AWS S3 with presigned URLs; 25MB max per upload |
