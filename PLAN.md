# StudyDash — Development Plan

## Phase 1: Project Setup & Infrastructure
- Initialize monorepo structure (backend, mobile, web-admin)
- Set up Node.js + TypeScript + Express backend
- Configure Prisma + PostgreSQL database
- Set up React Native (Expo) mobile app scaffolding
- Set up Next.js web admin panel scaffolding
- Configure AWS S3 bucket with presigned URL utilities
- Set up Redis + BullMQ job queue infrastructure
- Configure Firebase Cloud Messaging (FCM)
- Set up CI/CD pipeline (GitHub Actions)
- Configure environment management (dev, staging, prod)

## Phase 2: Authentication & Core Models
- Implement Phone + OTP login (parents, students, drivers) via MSG91/Twilio
- Implement Email + Password login (teachers, admin)
- JWT token management (access + refresh tokens, RS256 signing)
- Role-based access control (RBAC) middleware
- Multi-tenancy middleware (schoolId scoping on all queries)
- Core database models: School, User, StudentProfile, TeacherProfile, ParentProfile, DriverProfile, Class, Section, Subject

## Phase 3: School Onboarding
- Admin web panel: school setup wizard (profile, classes, subjects, fee heads, routes)
- CSV bulk import engine (students, teachers, drivers)
- Row-level validation and error reporting
- Auto-generation of parent accounts from student CSV
- Duplicate phone number detection and multi-child linking

## Phase 4: Attendance + Leave
- Teacher mobile: mark attendance per class (Present, Absent, Late)
- Student/parent mobile: attendance calendar view and summary
- Admin web: attendance compliance reports and override
- Parent mobile: submit leave requests
- Teacher mobile: approve/reject leave requests
- Leave ↔ attendance integration (auto-update ON_LEAVE with conflict check)
- Absence push notification to parents

## Phase 5: Homework, Notes & Timetable
- Teacher mobile: create assignments with file attachments (S3 presigned upload)
- Teacher mobile: upload class notes
- Student mobile: view assignments, mark complete, view/bookmark notes
- Parent mobile: read-only assignment and notes view
- Admin web: timetable configuration (periods, subjects, teachers)
- Mobile views: today + weekly timetable for all roles
- BullMQ job: thumbnail generation for uploaded files

## Phase 6: Fees Payment
- Admin web: fee structure configuration, invoice generation
- Admin web: record offline payments, defaulter reports, collection exports
- Parent mobile: view outstanding fees, pay via Razorpay
- Razorpay integration: checkout, webhook handling, signature verification
- Idempotency and partial payment tracking
- BullMQ job: PDF receipt generation
- Daily cron: overdue fee status update, reconciliation batch

## Phase 7: Communication Channel
- Teacher mobile: class-wide announcements
- Parent mobile: view announcements, submit complaint tickets
- Admin web: school-wide broadcasts, ticket management dashboard
- Ticket workflow: Open → In Progress → Resolved
- Push notifications for new messages

## Phase 8: GPS Bus Tracking
- Admin web: route and stop configuration, driver assignment, student-stop mapping
- Driver mobile: start/end route, background GPS streaming (10s intervals)
- Parent mobile: live map view (Google Maps SDK), ETA display
- Backend: location ingestion, geofence detection (500m), notification triggers
- Admin web: live bus map dashboard

## Phase 9: Expense Tracker
- Admin web: expense logging form (amount, category, date, description, receipt upload)
- Category management (predefined + custom)
- Expense list with filters and sorting
- Monthly summary dashboard (total, category pie chart, month comparison)

## Phase 10: Notifications & Polish
- Consolidate all notification triggers across features
- FCM + APNs delivery via BullMQ fan-out
- In-app notification history (DB-backed, paginated)
- Notification states: CREATED → DELIVERED → READ
- End-to-end testing of all notification paths
- UI polish, error handling, loading states across all screens

## Phase 11: Testing & Launch Prep
- Unit tests for backend services and API endpoints
- Integration tests for payment flow, attendance+leave, CSV import
- Mobile E2E tests for critical user journeys
- Performance testing (concurrent users, API response times)
- Security review (auth flows, input validation, webhook verification)
- Staging environment deployment and UAT with pilot school(s)

## Phase 12: Deployment & Go-Live
- Production infrastructure setup (hosting, database, Redis, S3)
- Database backup automation
- Monitoring and error tracking (Sentry + basic logging)
- Mobile app submission (Google Play Store + Apple App Store)
- Web admin panel deployment
- Pilot school onboarding and feedback loop

---

## Post-MVP Development Order
1. Pixel Trace Light (manual photo tagging)
2. Pixel Trace AI (face recognition matching)
3. WhatsApp Integration
4. School Email Provisioning
5. PTM Scheduler
6. Library Management
7. AI Attendance
8. Board-Specific Report Cards
9. Admission & Enrollment Management
10. Advanced Analytics & SIS Integrations
