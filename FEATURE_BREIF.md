# StudyDash — Feature Overview

StudyDash is a unified school management platform serving **Admins, Teachers, Parents, and Students**. Below is a brief description of all planned features.

---

## Core Features (v1 Launch)

### 1. GPS Bus Tracking
Real-time map tracking of school buses with estimated arrival times at each stop. Parents receive push notifications when the bus is nearby and when their child boards or alights.

### 2. Digital Attendance
Teachers mark attendance digitally per period via a class roster. Parents are instantly notified of absences. Admins get daily/weekly/monthly reports and automatic alerts when a student's attendance falls below the threshold.

### 3. Fees Payment
Parents can pay school fees online (UPI, cards, net banking). Supports installment plans, automated due reminders, PDF receipt generation, and concession/scholarship tracking. Admins get defaulter reports and full payment history.

### 4. Communication Channel
Structured in-app messaging replacing unorganized WhatsApp groups. Teachers can send direct updates or class-wide broadcasts to parents. Parents can raise complaints through a ticket-style flow (submitted → acknowledged → resolved) with full escalation history.

### 5. Timetable & Calendar
Class-wise and teacher-wise timetables with real-time substitution updates. School-wide academic calendar with holidays, exams, PTM dates, and events. Supports device calendar sync and configurable event reminders.

### 6. Homework & Class Notes
Teachers post homework assignments per subject with due dates and optional file attachments. Parents and students are notified. Class notes (photos or files) can be attached to each period for student reference.

### 7. Pixel Trace — Event Photo Discovery *(USP)*
AI-powered photo discovery for school events. Parents search by their child's name and receive all event photos containing that student, organized by event (Sports Day, Annual Function, etc.). Strictly opt-in with explicit parent consent; face data is deleted on consent revocation. Compliant with India's DPDP Act.

### 8. AI Attendance *(v2 — after Pixel Trace is validated)*
Teacher takes a single classroom photo; the AI auto-marks attendance and generates an absentee list for teacher verification. Built on the same face model trained through Pixel Trace. Teacher confirmation is mandatory before submission.

### 9. Academic Records & Report Cards
Exam-wise marks entry by teachers. Auto-calculated totals, percentages, and grades. PDF report card generation. Historical performance trends for parents and class-wide analytics for teachers.

### 10. Leave Applications
Parents submit student leave requests digitally; class teachers approve or reject with comments. Approved leaves auto-update attendance records. Teacher leave triggers a substitute assignment workflow linked to the timetable.

### 11. WhatsApp Integration
Automated school alerts (absence, fee reminders, homework) delivered via WhatsApp Business API — the channel parents already check. Serves as a reliable fallback when push notifications go undelivered. Parents can reply to acknowledge messages. Uses TRAI-compliant templates via Gupshup or Interakt.

### 12. Board-Specific Grading & Report Cards
Configurable grading system supporting CBSE (9-point scale with co-scholastic areas), ICSE (percentage-based), and State Board formats. Admins set exam weightage per board; teachers enter marks in the appropriate format; the system auto-calculates grades and generates board-compliant PDF report cards in bulk. Includes UDISE+ data export for government compliance.

### 13. Admission & Enrollment Management
End-to-end digital admission flow: online inquiry forms, document upload (Aadhaar, birth certificate, previous TC), waitlist management with auto-notification on seat availability, and fee collection at admission. Transfer Certificate (TC) generation for outgoing students. Reduces dependency on walk-in paperwork and manual registers.

### 14. PTM Scheduler
Admins create Parent-Teacher Meeting events with configurable time slots per teacher. Parents book slots online; teachers see their full appointment schedule for the PTM day. Automated reminders sent 1 day and 1 hour before each slot. After the meeting, teachers add per-student notes visible to the parent, building a structured communication trail.

### 15. Library Management
Digital library system covering book issue/return with barcode lookup, due date reminders, and overdue fine calculation. Students can browse the catalog, check availability, and access a digital resources section (NCERT PDFs, sample papers, question banks). Borrowing history and overdue status visible to students and parents.

---

## Admin & Management Features

### 16. Teacher Management
Teacher attendance tracking, salary management with payslip generation, substitute teacher pool, and assignment flow integrated with the timetable.

### 17. User & Role Management
Full user management for students, teachers, and staff. Role-based access control (RBAC) ensuring each user sees only what they're permitted to.

### 18. Analytics & Reporting
Dashboards for assignment completion rates, attendance trends, fee collection status, notification delivery metrics, and Pixel Trace usage and accuracy.

---

## Platform-Wide Capabilities

| Capability | Description |
|---|---|
| **Push Notifications** | Real-time alerts for attendance, fees, messages, homework, and bus arrival |
| **Offline Access** | Cached content accessible without internet; auto-syncs on reconnect |
| **Security** | JWT auth, role-based permissions, data encryption in transit and at rest |
| **Privacy & Compliance** | DPDP Act (India) compliance; consent management for minors' biometric data |
| **Integrations** | School ERP/SIS sync, Google Drive (Pixel Trace), calendar export, SMS fallback, WhatsApp Business API |

---

## Release Roadmap Summary

| Phase              | Scope                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------- |
| **v1 — Launch**    | Bus GPS, Digital Attendance, Fees, Communication, Timetable, Homework                       |
| **v1.5**           | Pixel Trace (event photo matching), WhatsApp Integration, PTM Scheduler, Library Management |
| **v2**             | AI Attendance (classroom photo → auto roll call), Board Report Cards, Admission Management  |
| **v3 / On Demand** | Advanced analytics, SIS integrations                                                        |
