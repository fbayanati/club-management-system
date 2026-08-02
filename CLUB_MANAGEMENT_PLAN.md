# Club Management System — Complete Architecture & Implementation Plan

## Executive Summary

A comprehensive, cost-optimized club management system built on Angular 22 with Firebase backend, Angular Material UI, and AG Grid for data tables. The system supports membership management, event scheduling, financial tracking, communication tools, and advanced retention workflows — all within Firebase's generous free tier for small-to-medium clubs.

---

## 1. Technology Stack

| Layer                    | Technology                                    | Cost                 |
| ------------------------ | --------------------------------------------- | -------------------- |
| **Frontend**             | Angular 22 (standalone components, signals)   | Free                 |
| **UI Components**        | Angular Material 3                            | Free                 |
| **Data Tables**          | AG Grid Community                             | Free                 |
| **Calendar**             | FullCalendar (free version)                   | Free                 |
| **Backend/Data**         | Firebase Firestore (NoSQL, real-time)         | Free tier generous   |
| **Authentication**       | Firebase Authentication                       | Free tier generous   |
| **Serverless Functions** | Firebase Cloud Functions                      | Free tier generous   |
| **Hosting**              | Firebase Hosting (CDN, SSL)                   | Free tier sufficient |
| **State Management**     | NgRx SignalStore (entity management, caching) | Free                 |
| **Notifications**        | ngx-toastr                                    | Free                 |
| **Testing**              | Vitest (unit) + Playwright (e2e)              | Free                 |
| **Version Control**      | Git + GitHub                                  | Free                 |

**Total monthly cost for small club (1-500 members): $0** (within Firebase free tier)

---

## 2. Cost Optimization Strategy

### Firebase Free Tier (Spark Plan)

- Firestore: 1GB storage, 50K reads/day, 20K writes/day, 20K deletes/day, 10GB download
- Auth: 10K verifications/month, 10K emails/month
- Functions: 2M invocations/month, 400K GB-seconds
- Hosting: 10GB storage, 10GB/month transfer

### Cost Optimization Techniques

1. **Firestore Caching**: Enable offline persistence to reduce redundant reads
2. **Pagination**: Use AG Grid pagination + Firestore cursor-based queries
3. **Efficient Queries**: Use composite indexes, avoid unindexed field queries
4. **Batch Operations**: Use Firestore batch writes for multi-document operations
5. **Smart Listeners**: Use Firestore real-time listeners only on active views
6. **CDN for Assets**: Store images/documents in Firebase Storage with CDN
7. **Abstract Data Layer**: Interface-based design allows future migration to Supabase/PocketBase

### Alternative Backends (Migration Path)

- **Supabase**: More generous free tier (500MB DB, 50K MAU auth), PostgreSQL, open-source
- **PocketBase**: Zero-cost self-hosted (single binary, ~$5/month VPS)

---

## 3. Data Model (Firestore Collections)

### 3.1 Core Entities

#### clubs/{clubId}

```
name: string
description: string
logoURL: string
timezone: string
customFields: { [key: string]: FieldConfig }
integrations: { stripe?: {...}, sendgrid?: {...}, ... }
settings: { allowSelfRegistration, requireApproval, ... }
createdAt: timestamp
updatedAt: timestamp
```

#### members/{memberId}

```
clubId: string
firstName: string
lastName: string
email: string
phone: string
joinDate: timestamp
status: active | inactive | suspended | expired
membershipPlanId: string
role: admin | organizer | member
customFields: { [key: string]: any }
emergencyContact: { name, phone, relationship }
photoURL: string
tags: string[]
memberPreferences: { notificationPrefs, language, theme }
consents: { [key: string]: { givenAt, revokedAt } }
portalLastSeenAt: timestamp
createdAt: timestamp
updatedAt: timestamp
statusHistory: [{ status, at, by, reason }]
```

#### membershipPlans/{planId}

```
clubId: string
name: string
description: string
price: number
currency: string
billingCycle: monthly | quarterly | annual | one-time
benefits: string[]
gracePeriodDays: number
trialDays: number
maxDiscounts: number
isActive: boolean
sortOrder: number
createdAt: timestamp
updatedAt: timestamp
```

#### venues/{venueId}

```
clubId: string
name: string
address: string
capacity: number
latitude: number
longitude: number
amenities: string[]
isVirtual: boolean
virtualLink: string
createdAt: timestamp
updatedAt: timestamp
```

#### eventCategories/{categoryId}

```
clubId: string
name: string
color: string
icon: string
description: string
sortOrder: number
```

#### events/{eventId}

```
clubId: string
title: string
description: string
purpose: string (free-text intent)
categoryId: string (normalized category reference)
venueId: string
startAt: timestamp
endAt: timestamp
isAllDay: boolean
isRecurring: boolean
recurrenceRule: string (RRULE format)
capacityUsed: number (computed)
waitlistEnabled: boolean
registrationOpenAt: timestamp
registrationCloseAt: timestamp
maxParticipants: number
status: draft | published | cancelled | completed
createdBy: string
createdAt: timestamp
updatedAt: timestamp
statusHistory: [{ status, at, by, reason }]
```

### 3.2 Financial Entities

#### invoices/{invoiceId}

```
memberId: string
clubId: string
membershipPlanId: string (optional)
amount: number
currency: string
status: draft | open | paid | void | failed
dueAt: timestamp
issuedAt: timestamp
paidAt: timestamp
voidedAt: timestamp
lineItems: [{ description, amount, quantity }]
autoRenew: boolean
renewalId: string (optional)
createdAt: timestamp
updatedAt: timestamp
```

#### payments/{paymentId}

```
invoiceId: string
memberId: string
clubId: string
amount: number
currency: string
method: card | bank | cash | check
paidAt: timestamp
providerRef: string (Stripe transaction ID)
status: pending | succeeded | failed | refunded
createdAt: timestamp
updatedAt: timestamp
```

#### renewals/{renewalId}

```
memberId: string
membershipPlanId: string
clubId: string
nextRenewalAt: timestamp
status: pending | processing | completed | failed | cancelled
reminderSentAt: timestamp
autoRenew: boolean
lastAttemptAt: timestamp
failureReason: string
createdAt: timestamp
updatedAt: timestamp
```

### 3.3 Participation & Attendance

#### eventParticipants/{participantId}

```
eventId: string
memberId: string
clubId: string
status: registered | attended | cancelled | no-show | waitlisted
rsvpAt: timestamp
checkInAt: timestamp
checkInMethod: manual | qr | nfc
checkedInBy: string (userId)
checkInCode: string (unique per event)
notes: string
createdAt: timestamp
updatedAt: timestamp
```

#### eventWaitlist/{waitlistId}

```
eventId: string
memberId: string
clubId: string
joinedAt: timestamp
priority: number
notifiedAt: timestamp
expiresAt: timestamp
status: waiting | notified | registered | expired
```

### 3.4 Communication & Engagement

#### announcements/{announcementId}

```
clubId: string
title: string
body: string
audienceRules: { tags?, membershipTypes?, roles?, attendanceBehavior? }
channel: in-app | email | both
scheduledAt: timestamp
sentAt: timestamp
sentCount: number
createdBy: string
createdAt: timestamp
updatedAt: timestamp
```

#### documents/{documentId}

```
clubId: string
type: waiver | policy | agreement
title: string
fileURL: string
version: string
content: string (for inline documents)
isActive: boolean
required: boolean
createdAt: timestamp
updatedAt: timestamp
```

#### memberDocuments/{memberDocId}

```
memberId: string
documentId: string
clubId: string
status: pending | signed | expired | revoked
signedAt: timestamp
revokedAt: timestamp
expiresAt: timestamp
signatureData: string (base64 or storage reference)
```

#### surveys/{surveyId}

```
clubId: string
title: string
description: string
eventId: string (optional, linked to specific event)
questions: [{ id, type, text, required, options }]
isActive: boolean
createdAt: timestamp
updatedAt: timestamp
```

#### surveyResponses/{responseId}

```
surveyId: string
memberId: string
eventId: string (optional)
clubId: string
answers: [{ questionId, answer }]
submittedAt: timestamp
createdAt: timestamp
```

#### tasks/{taskId}

```
clubId: string
title: string
description: string
assigneeId: string (memberId)
assignorId: string (userId)
type: follow-up | event-prep | outreach | billing
dueAt: timestamp
completedAt: timestamp
status: pending | in-progress | completed | cancelled
relatedEntityId: string
relatedEntityType: string
createdAt: timestamp
updatedAt: timestamp
```

### 3.5 Analytics & Audit

#### auditLogs/{logId}

```
clubId: string
action: string
entityType: string
entityId: string
userId: string
userRole: string
details: object (JSON)
ipAddress: string
userAgent: string
timestamp: timestamp
```

#### clubMetrics/{metricId}

```
clubId: string
period: daily | weekly | monthly
date: timestamp
metrics: {
  totalMembers: number
  activeMembers: number
  churnRate: number
  eventsHeld: number
  totalAttendance: number
  avgAttendance: number
  revenue: number
  outstandingDues: number
  atRiskMembers: number
  newMembers: number
}
```

---

## 4. Feature Breakdown

### Phase 1: Foundation (Week 1)

1. Initialize GitHub repository and make initial commit
2. Install dependencies (Angular Material, Firebase SDK, AG Grid, FullCalendar, ngx-toastr, NgRx SignalStore)
3. Set up environment files with Firebase config
4. Initialize Firebase app module + Firestore + Auth
5. Create shared models (all TypeScript interfaces)
6. Build abstract data service interface (for future backend swap)
7. Set up NgRx SignalStore base stores (auth store, club store, loading store)
8. Build authentication flow (login, register, password reset, Google SSO)
9. Create layout components (header, sidebar, footer, theme toggle)
10. Set up routing with auth guards + role guards
11. Configure Firestore security rules (club-scoped access)
12. Implement Firestore caching layer for cost optimization

### Phase 2: Core Features (Weeks 2-3)

1. **Dashboard** — upcoming events, recent members, quick stats, calendar mini-view
2. **Member Management** — list/detail/CRUD, search/filter, status history, custom fields, tags
3. **Membership Plans** — CRUD plans with pricing, billing cycles, benefits
4. **Event Management** — list/detail/CRUD, recurring events, registration windows, capacity
5. **Venue Management** — CRUD venues, virtual venue support
6. **Event Categories** — CRUD categories with color/icon
7. **Participant Tracking** — RSVP, attendance marking, status history
8. **Invoices & Payments** — create invoices, record payments, payment status
9. **Basic unit tests** for all services

### Phase 3: High-Value Additions (Weeks 4-5)

1. **Member Self-Service Portal** — profile updates, notification preferences, consents
2. **Automated Renewals** — renewal scheduling, reminder emails via Functions
3. **Event Waitlists** — auto-promotion, expiration handling
4. **QR/Check-in Attendance** — unique check-in codes, QR scanning
5. **Announcements & Messaging** — targeted by tags/membership type/attendance
6. **Digital Waivers & Documents** — e-signature flow, member document tracking
7. **Feedback & Surveys** — create surveys, link to events, collect responses
8. **Reporting Dashboards** — attendance trends, revenue, churn analysis
9. **Export Functionality** — CSV/PDF export

### Phase 4: Operational Excellence (Week 6)

1. **Granular Permissions** — canManageBilling, canCheckIn, canSendAnnouncements, canViewReports
2. **Task/Follow-up Tracking** — assign tasks, due dates, status tracking
3. **Status Histories** — full audit trail for all entities
4. **Behavioral Retention Workflows** — auto-tag at-risk members, trigger invites
5. **Dark/Light Theme** toggle
6. **Responsive Design** testing
7. **Performance Optimization**
8. **E2E Tests** with Playwright
9. **Deploy** to Firebase Hosting
10. **Documentation**

---

## 5. Key Design Decisions

1. **Standalone Components**: Angular 22 standalone components — no NgModules, less boilerplate
2. **NgRx SignalStore**: Modern NgRx SignalStore for entity management, caching, and shared state (SHARI principle applies: state is Shared, Hydrated, Available, Retrieved, and Impacted across many components)
3. **AG Grid Community**: Free, powerful data tables with filtering, sorting, pagination, export
4. **Abstract Data Layer**: Interface-based design allows future migration to Supabase/PocketBase
5. **Firestore Caching**: Built-in offline persistence + NgRx SignalStore caching for cost optimization
6. **Security Rules**: Club-scoped row-level security — users can only access data for their club
7. **Optimistic Updates**: UI updates immediately on user action, with rollback on error
8. **Lazy Loading**: Feature modules loaded on-demand for faster initial load
9. **Custom Fields**: JSON-based custom fields on clubs and members for domain-specific data
10. **Status History**: Every entity with a status field has a statusHistory[] array for auditability
11. **Behavioral Automation**: Firebase Cloud Functions for scheduled jobs (renewal reminders, at-risk detection)
12. **Event Schema**: Split purpose (free-text) and categoryId (normalized) for flexible categorization
13. **Registration Windows**: registrationOpenAt / registrationCloseAt on events for controlled registration
14. **No TanStack Query**: React-focused; NgRx SignalStore + Firestore observables cover the same use cases
15. **GitHub Version Control**: Repository initialized from day one for version tracking and collaboration

---

## 6. Behavioral Retention Workflow Example

**Scenario**: A member has not attended any events in 45 days.

**Automated Workflow**:

1. Firebase Cloud Function runs daily, queries members with no attendance in 45 days
2. Tags those members as "at-risk" in Firestore
3. Sends a targeted announcement/email with personalized event recommendations
4. Creates a follow-up task for an organizer to reach out
5. Tracks engagement after the intervention

This turns the data model into an active retention tool rather than just a record system.

---

## 7. Folder Structure

```
src/
├── app/
│   ├── core/
│   │   ├── data/                    # Abstract data layer (swappable backend)
│   │   │   ├── interfaces/
│   │   │   ├── services/
│   │   │   ├── repositories/
│   │   │   └── cache/
│   │   ├── services/                # Auth, error handler, notifications
│   │   ├── guards/                  # Auth guard, role guard
│   │   ├── interceptors/            # Error interceptor
│   │   └── models/                  # All TypeScript interfaces
│   │
│   │   ├── shared/
│   │   ├── components/
│   │   │   ├── layout/              # Header, sidebar, footer
│   │   │   ├── ui/                  # Status badges, member cards, etc.
│   │   │   ├── forms/               # Member form, event form, venue form
│   │   │   └── tables/              # AG Grid wrapper components
│   │   │       ├── base-table/
│   │   │       ├── member-table/
│   │   │       ├── event-table/
│   │   │       └── participant-table/
│   │   ├── pipes/                   # Member name, date range, etc.
│   │   └── directives/              # Role visibility, etc.
│   │
│   │   ├── features/
│   │   │   ├── auth/                # Login, register, reset password
│   │   │   ├── dashboard/           # Stats, upcoming events, calendar
│   │   │   ├── members/             # CRUD, search, tags, custom fields
│   │   │   ├── membership-plans/    # CRUD plans, pricing, benefits
│   │   │   ├── events/              # CRUD, recurring, registration windows
│   │   │   ├── venues/              # CRUD venues, virtual support
│   │   │   ├── categories/          # CRUD categories
│   │   │   ├── participants/        # RSVP, attendance, waitlists, QR
│   │   │   ├── invoices/            # CRUD invoices, payment tracking
│   │   │   ├── payments/            # Record payments, payment methods
│   │   │   ├── renewals/            # Auto-renewal, reminders
│   │   │   ├── announcements/       # Targeted announcements
│   │   │   ├── documents/           # Waivers, policies, e-signatures
│   │   │   ├── surveys/             # Surveys, responses
│   │   │   ├── reports/             # Dashboards, analytics, exports
│   │   │   ├── tasks/               # Follow-up tracking
│   │   │   └── settings/            # Club settings, permissions
│   │
│   ├── app.routes.ts
│   ├── app.config.ts
│   ├── app.html
│   └── app.scss
│
├── environments/
│   ├── environment.ts
│   └── environment.prod.ts
│
├── styles/
│   ├── _variables.scss
│   ├── _mixins.scss
│   └── themes/
│
└── firebase/
    ├── firebase.config.ts
    └── firestore.rules
```

---

## 8. Firestore Security Rules (Summary)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users must be authenticated
    function isSignedIn() {
      return request.auth != null;
    }

    // User must belong to the club
    function isClubMember(clubId) {
      return isSignedIn() &&
        exists(/databases/$(database)/documents/clubs/$(clubId)) &&
        get(/databases/$(database)/documents/clubs/$(clubId)).data.memberIds.hasAny([request.auth.uid]);
    }

    // User must be admin or organizer
    function isClubAdmin(clubId) {
      return isSignedIn() &&
        get(/databases/$(database)/documents/members/$(request.auth.uid)).data.clubId == clubId &&
        get(/databases/$(database)/documents/members/$(request.auth.uid)).data.role in ['admin', 'organizer'];
    }

    // Club-scoped access for all collections
    match /clubs/{clubId} {
      allow read, update, delete: if isClubAdmin(clubId);
      allow create: if isSignedIn();
    }

    match /members/{memberId} {
      allow read: if isClubMember(resource.data.clubId);
      allow create, update, delete: if isClubAdmin(resource.data.clubId);
    }

    // ... similar rules for all other collections
  }
}
```

---

## 9. Final Decisions

### Payment Processor

**Start with manual payment tracking** (record cash/check/card payments manually). Stripe integration will be added in Phase 3 as an optional module. This keeps Phase 1-2 focused on core functionality without payment processor complexity.

### Club Domain

**Truly generic** with multi-club support. The system is designed to work for any club type:

- **Phase 1 target**: Volunteer club (default custom fields: volunteer interests, availability, skills)
- **Phase 2 target**: Training club (default custom fields: certification expiry, skill level, training history)
- **Future**: Sports clubs, professional associations, hobby groups, etc.

The `customFields` system on clubs and members ensures domain-specific data can be added without schema changes. Default field templates will be provided per club type.

---

## 10. Next Steps

1. Switch to Act Mode
2. Begin Phase 1 implementation immediately
3. Set up the project structure, dependencies, and Firebase configuration
4. Build the authentication flow and core layout
5. Begin implementing core features in order of priority

---

_Generated: August 2, 2026_
_Project: Club Management System_
_Tech Stack: Angular 22 + Firebase + Angular Material + AG Grid_
