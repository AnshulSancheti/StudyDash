# StudyDash — Feature Brief

StudyDash is a unified school management platform for Indian schools serving **Admins, Teachers, Parents, Students, and Drivers**. Mobile app for teachers/parents/students/drivers, web admin panel for school administration.

---

## MVP Features

### 1. School Onboarding
Self-service setup for schools. Guided wizard for school profile, classes, subjects, fee heads, and bus routes. CSV bulk import for students (with auto-generated parent accounts), teachers, and drivers. Row-level validation and error reporting.

### 2. Digital Attendance
Teachers mark attendance digitally per class (Present, Absent, Late). Parents get instant push notifications on absence. Students see a color-coded monthly calendar. Admins get compliance reports showing which teachers have/haven't marked attendance.

### 3. Homework & Class Notes
Teachers post assignments with due dates and file attachments, and upload class notes. Students view, download, bookmark, and mark assignments complete. Parents get read-only access and push notifications for new assignments.

### 4. Timetable
Admin configures class schedules (periods, subjects, teachers). Students, parents, and teachers each see their relevant view (today + weekly).

### 5. Fees Payment
Online fee payment via Razorpay (UPI, cards, net banking) with partial payment support. Admin creates fee structures, generates invoices, records offline payments, and runs defaulter reports. Automated due date reminders. PDF receipt generation.

### 6. Communication Channel
Structured in-app messaging replacing WhatsApp groups. Teacher class-wide announcements, parent complaint/query tickets (tracked to resolution), admin school-wide broadcasts.

### 7. Leave Applications
Parents submit digital leave requests for their child. Teachers approve/reject. Approved leaves auto-update attendance records to ON_LEAVE with conflict checking.

### 8. Push Notifications
Event-driven notifications via FCM/APNs: absence alerts, new homework, fee reminders, payment confirmations, leave status changes, bus alerts, announcements. In-app notification history.

### 9. GPS Bus Tracking
Live bus tracking using driver's phone GPS (no hardware needed). Parents see bus on a map with ETA to their stop. Notifications for route start and bus approaching stop (500m geofence). Admin configures routes, stops, and driver assignments.

### 10. Expense Tracker
Simple expense logging for school admin. Categories (Salaries, Maintenance, Utilities, Events, etc.), receipt uploads, monthly summaries with pie charts. No budgeting, no approval workflows — just a clean expense log.

---

## Post-MVP Roadmap

| Phase | Feature | Description |
|-------|---------|-------------|
| v1.5a | Pixel Trace Light | Manual photo tagging by teachers, consent-gated viewing by parents/students |
| v1.5b | Pixel Trace AI | Face recognition auto-matching with moderation queue |
| v1.5c | WhatsApp Integration | Automated alerts via WhatsApp Business API, fallback chain |
| v2a | School Email Provisioning | Google Workspace / Microsoft 365 student email management |
| v2b | PTM Scheduler | Parent-teacher meeting slot booking with reminders |
| v2c | Library Management | Book catalog, issue/return, fines, digital resources |
| v2d | AI Attendance | Classroom photo → auto roll call (requires Pixel Trace maturity) |
| v2e | Board Report Cards | CBSE/ICSE/State Board format PDF generation |
| v2f | Admission & Enrollment | Online inquiry → application → document upload → enrollment |
| v3 | Advanced Analytics | Multi-school dashboards, SIS integrations |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Node.js + TypeScript + Express |
| Database | Prisma + PostgreSQL |
| Mobile | React Native (Expo) |
| Web Admin | Next.js + React |
| Storage | AWS S3 |
| Queue | BullMQ + Redis |
| Payments | Razorpay |
| OTP | MSG91 or Twilio |
| Push | Firebase Cloud Messaging |
| PDF | Puppeteer |

---

## Authentication

| Role | Method | Platform |
|------|--------|----------|
| Parents & Students | Phone + SMS OTP | Mobile app |
| Teachers | Email + Password | Mobile app |
| Admin | Email + Password | Web panel |
| Drivers | Phone + SMS OTP | Mobile app |

---

## User Roles

| Role | Access |
|------|--------|
| **Admin** | Web panel — full school config, user management, fees, reports, expenses, bus routes |
| **Teacher** | Mobile — attendance, assignments, notes, leave approvals, announcements |
| **Parent** | Mobile — child's attendance/homework/fees/bus/leave, complaints, multi-child support |
| **Student** | Mobile — own attendance/homework/notes/timetable/bus |
| **Driver** | Mobile — start/end route, GPS streaming |
