# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

StudyDash is a unified school management platform for Indian schools serving five user roles: Admin, Teacher, Parent, Student, and Driver. The key differentiator is "Pixel Trace" — an AI-powered face recognition system for event photo discovery (post-MVP).

**Status**: Specification complete. MVP scope locked. Implementation not started.

## MVP Scope (10 features)

1. School Onboarding (wizard + CSV bulk import)
2. Digital Attendance (manual, teacher-marked)
3. Homework & Class Notes
4. Timetable
5. Fees Payment (Razorpay + offline recording)
6. Communication Channel (announcements + complaint tickets)
7. Leave Applications
8. Push Notifications (FCM/APNs)
9. GPS Bus Tracking (driver phone GPS)
10. Expense Tracker (admin-only)

## Platforms

- **Mobile app (React Native/Expo):** Teachers, Parents, Students, Drivers
- **Web admin panel (Next.js):** School Administration

## Tech Stack

- **Backend**: Node.js + TypeScript + Express, Prisma ORM + PostgreSQL
- **Mobile**: React Native (Expo)
- **Web Admin**: Next.js + React
- **Storage**: AWS S3 (presigned URLs), path: `{schoolId}/{feature}/{entityId}/{filename}`
- **Queue**: BullMQ + Redis (notifications, PDFs, CSV processing, thumbnails)
- **Payments**: Razorpay (webhook + signature verification)
- **OTP**: MSG91 or Twilio
- **Push**: Firebase Cloud Messaging
- **PDF**: Puppeteer

## Authentication

- Parents, Students, Drivers: Phone + SMS OTP
- Teachers: Email + Password
- Admin: Email + Password
- JWT access tokens (15 min) + refresh tokens (30 days mobile, 7 days admin)
- RS256 signing, bcrypt password hashing (12 rounds)

## Architecture Decisions

- **Multi-tenancy**: All queries scoped by `schoolId` from JWT. S3 paths prefixed by schoolId.
- **RBAC**: Five roles (ADMIN, TEACHER, PARENT, STUDENT, DRIVER) with role-based route prefixes (`/api/v1/admin/`, `/api/v1/teachers/`, etc.)
- **API**: RESTful, cursor-based pagination, standard error format `{ error, code, details? }`
- **Rate limiting**: 100 req/min per user, 5 OTP/hour per phone
- **Background jobs**: BullMQ with 3 retries, exponential backoff, dead letter queue
- **File uploads**: 25MB max, MIME type validation, presigned URLs

## Key Specification Files

| File | Contains |
|------|----------|
| `MVP.md` | Locked MVP design spec with all feature details |
| `PITCH.md` | Investor-facing pitch document |
| `PRODUCT_SPEC.md` | Full technical specification (MVP + post-MVP) |
| `FEATURES.md` | Feature list organized by role and platform |
| `FEATURE_BREIF.md` | High-level feature summaries and roadmap |
| `PLAN.md` | Development phases and implementation order |
| `Personas/*.md` | Role-specific technical implementation details (schemas, APIs, flows) |

## Post-MVP Roadmap (in order)

1. Pixel Trace Light (manual tagging)
2. Pixel Trace AI (face recognition)
3. WhatsApp Integration
4. School Email Provisioning
5. PTM Scheduler
6. Library Management
7. AI Attendance
8. Board-Specific Report Cards
9. Admission & Enrollment
