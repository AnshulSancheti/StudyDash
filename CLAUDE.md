# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

StudyDash is a unified school management platform for Indian schools serving four user roles: School Administration, Teachers, Parents, and Students. The key differentiator is "Pixel Trace" — an AI-powered face recognition system for event photo discovery.

**Status**: Specification and planning phase. No implementation code exists yet. See `PRODUCT_SPEC.md` for the full technical spec and `Personas/` for role-specific requirements.

## Planned Tech Stack

- **Backend**: Node.js + TypeScript, Prisma ORM
- **Mobile**: React Native (iOS/Android) with Expo push notifications
- **Database**: PostgreSQL (multi-tenant, all queries scoped by `schoolId`)
- **Auth**: JWT — access tokens (15 min), refresh tokens (30 days, stored hashed)
- **Job Queues**: BullMQ for background tasks (reminders, report generation, media processing)
- **Storage**: AWS S3 for documents/media, Google Drive API for event photos
- **Payments**: Razorpay or Cashfree
- **Messaging**: WhatsApp Business API (Gupshup/Interakt), SMS fallback
- **PDF Generation**: Puppeteer (report cards, transfer certificates)
- **Face Recognition**: Custom embedding pipeline for Pixel Trace

## Architecture Decisions

- **Multi-tenancy**: Every database query and API endpoint must be scoped by `schoolId`. This is the primary data isolation boundary.
- **RBAC**: Four roles (ADMIN, TEACHER, PARENT, STUDENT) with granular feature-level permissions.
- **Offline-first mobile**: Local caching with auto-sync on reconnect.
- **Notification chain**: Push notification → WhatsApp → SMS fallback.
- **DPDP Act compliance**: Consent management required for biometric data (Pixel Trace). Face embeddings stored as vectors, not raw biometric templates. Strict opt-in with revocation support.
- **India-specific**: UDISE+ export, TRAI-compliant WhatsApp templates, board-specific grading (CBSE/ICSE/State Board).

## Feature Phases

- **v1 (Core)**: GPS bus tracking, digital attendance, fees payment, communication channel, timetable/calendar, homework/class notes
- **v1.5 (Differentiation)**: Pixel Trace, WhatsApp integration, PTM scheduler, library management
- **v2 (AI)**: AI attendance (classroom photo → auto roll call), board-specific report cards, admission/enrollment management

## Key Specification Files

| File | Contains |
|------|----------|
| `PRODUCT_SPEC.md` | Complete technical specification |
| `FEATURE_BREIF.md` | High-level feature overview |
| `FEATURES.md` | Detailed feature list by role |
| `PLAN.md` | Development phases roadmap |
| `Personas/Student_Persona.md` | Auth flows, DB schema examples, API patterns |
| `Personas/Teacher_Persona.md` | Teacher workflows and feature access |
| `Personas/Parent_Persona.md` | Parent app features |
| `Personas/School_Management_Persona.md` | Admin features and management |
