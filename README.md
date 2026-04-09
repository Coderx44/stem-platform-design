# STEM Platform — Architecture Design Document

**Version:** 1.0 | **Date:** April 9, 2026 | **Status:** Draft

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Actors & Modules](#2-actors--modules)
3. [Architectural Decisions](#3-architectural-decisions)
4. [System Architecture](#4-system-architecture)
5. [Module Structure & Licensing](#5-module-structure--licensing)
6. [Database Design & Multi-Tenancy](#6-database-design--multi-tenancy)
7. [Workflow Engine](#7-workflow-engine)
8. [Offline Sync](#8-offline-sync)
9. [OMR Processing Pipeline](#9-omr-processing-pipeline)
10. [Notifications](#10-notifications)
11. [Role-Based Access Control](#11-role-based-access-control)
12. [Reporting & Analytics](#12-reporting--analytics)
13. [Phasing](#13-phasing)
14. [Risk Register](#14-risk-register)
15. [Open Decisions](#15-open-decisions)

---

## 1. Executive Summary

Multi-tenant, module-based STEM education management platform for NGOs. Manages donors, schools, students, trainers, teachers, sessions, attendance, assessments, and reporting.

**Key characteristics:** Multi-tenant SaaS with plug-and-play modules, mobile-first with offline capability, configurable approval workflows, OMR-based assessment with auto-scoring, built-in analytics with RAG scoring.

**Stack:**

| Layer | Technology |
|---|---|
| Web | Next.js |
| Mobile | React Native |
| Backend | TypeScript (Modular Monolith) |
| Auth | Supabase Auth (JWT, OTP, Google SSO) |
| Database | Supabase PostgreSQL + RLS |
| File Storage | AWS S3 |
| Cache & Queues | Redis / in-memory |
| Notifications | In-App + Push (FCM) + WhatsApp (Twilio) |
| OMR Processing | OMRChecker (Python microservice) |

---

## 2. Actors & Modules

### Roles

**Operational hierarchy (permissions inherit downward — PM has all Trainer permissions):**

| Designation | System Role | Responsibilities |
|---|---|---|
| Trainer | Trainer | Field data entry — attendance, assessments, sessions, school management |
| Cluster Lead / Manager | Project Manager | All Trainer permissions + approvals, monitoring trainers, reviewing rubrics |
| Regional Manager / Director | Admin | Full system control — targets, config, trainer management |

**Separate branch (view-only, not in operational hierarchy):**

| Designation | System Role | Responsibilities |
|---|---|---|
| Donor | Donor | View-only access to funded schools — dashboards, impact reports. No write access. |

### Modules

| # | Module | Key Features | Phase |
|---|---|---|---|
| M1 | School Management | CRUD, calendar, kits/assets, growth rubrics, STEM club/committee, volunteers | 1 |
| M2 | Student Management | Unique ID, bulk import, longitudinal tracking, grade promotion | 1 |
| M3 | Trainer Management | Profile, auto-login, leave, competency matrix, L&D | 1 |
| M4 | Teacher Management | Profile, competency rubric, teacher-led sessions | 1 |
| M5 | Session Planning | Create/edit/clone, assign trainers & students, cancel/reschedule | 1 |
| M6 | Attendance | Session + non-session, geo-tagging, overlap validation, STEM fest/workshop tracking | 1 |
| M7 | Assessment | OMR scanning, baseline/endline, subject/skill mapping, auto-scoring | 2 |
| M8 | Donor Management | Donor details, view-only dashboards, school-level impact | 2 |
| M9 | Reporting | Dashboards, RAG scoring, utilization, Excel/PDF export | 1 |

### Module Dependencies

```
Core Platform (Auth, Tenancy, Licensing)
    ├── M1: School (Core)
    │     ├── M2: Student
    │     ├── M4: Teacher
    │     └── M8: Donor
    ├── M3: Trainer (Core)
    ├── M5: Session (depends on M1, M2, M3)
    │     └── M6: Attendance (depends on M3, M5)
    ├── M7: Assessment (depends on M2)
    └── M9: Reporting (reads from all)

Shared: Workflow Engine, Notification Hub, OMR Pipeline
```

---

## 3. Architectural Decisions

### Decision 1: Multi-Tenancy — Shared DB + RLS

**Why:** Single Supabase PostgreSQL instance with Row-Level Security. All tenants share one database, isolated by `tenant_id` column + RLS policies.

**Why not alternatives:**
- Schema-per-tenant: excessive complexity for tens-to-hundreds of tenants
- Separate DB per tenant: highest ops cost, unnecessary for education NGOs

**Trade-off:** Risk of data leakage if RLS policy is missed on a new table — mitigated by integration tests that verify isolation.

### Decision 2: Architecture — Hybrid Modular Monolith

**Why:** Custom TypeScript backend (modular monolith) for all business logic. Supabase handles auth + database. S3 handles files.

**Why not alternatives:**
- Pure modular monolith (no Supabase): more infrastructure to self-manage
- Supabase-native + Edge Functions: Edge Function execution limits can't handle OMR processing or complex workflow logic

**Trade-off:** More infrastructure than pure Supabase, but full control over business logic. Can migrate off Supabase independently if needed.

### Decision 3: Offline — Hybrid (Critical Writes Only)

**Why:** Trainers work in schools with poor connectivity for a few hours. They need to write attendance and assessments offline. Everything else (approvals, reports, admin ops) requires connectivity.

**Why not alternatives:**
- Full local-first: extremely complex, overkill for hours-long offline windows
- Read-only caching: trainers need to write, not just read

**Trade-off:** Not all features available offline, but conflict resolution surface stays manageable (trainers only edit their own data).

### Decision 4: Workflow Engine — Generic + Configurable

**Why:** All 7+ approval flows follow the same pattern (submit → pending → approve/reject). A generic engine with configurable roles, steps, and escalation avoids duplicating logic across modules. New workflows = configuration, not code.

**Why not hardcoded per-module:** Duplicates the same pattern 7+ times, inconsistent audit trails, harder to add new flows.

### Decision 5: OMR — OMRChecker (Open Source)

**Why:** Built-in OMR processing using OMRChecker wrapped as a Python microservice behind a Redis/in-memory job queue. Purpose-built for education OMR sheets, faster to ship than custom OpenCV.

**Why not alternatives:**
- Third-party API: external dependency, ongoing cost
- Custom OpenCV: full control but significantly more dev effort (fallback for Phase 3 if needed)

### Decision 6: Notifications — Multi-Channel + User Preferences

**Why:** In-App + Push (FCM) + WhatsApp (Twilio). Users configure which notifications they receive on which channels. Preference hierarchy: user > tenant default > system default.

### Decision 7: Geo-Tagging — Capture + Soft Validation

**Why:** GPS captured with attendance, compared against school location. If outside expected radius, the entry is flagged (warning) but not blocked. Anomalies visible to managers.

**Why not strict geofencing:** Trainers with GPS issues or submitting later would be blocked. Soft validation gives visibility without blocking operations.

---

## 4. System Architecture

```
CLIENT: Next.js Web  |  React Native Mobile (offline-capable)
                     |
                  HTTPS / REST API
                     |
API GATEWAY: [Rate Limiting] [JWT Validation] [Tenant Resolution] [Module Gating]
                     |
BUSINESS LOGIC (Modular Monolith):
   Modules: School, Student, Trainer, Teacher, Session, Attendance, Assessment, Donor, Reporting
   Shared:  Workflow Engine, Notification Hub, OMR Pipeline, Module Registry, Event Bus
                     |
DATA LAYER:
   Supabase Auth (JWT)  |  Supabase PostgreSQL + RLS  |  AWS S3  |  Redis/in-memory (cache + queues)
                     |
EXTERNAL: Twilio (WhatsApp)  |  FCM (Push)  |  CloudFront CDN
```

### Design Principles

- **Tenant-First:** Every request resolves tenant from JWT. RLS enforces at DB level. `tenant_id` on every table.
- **Module-Gated:** Middleware checks tenant's licensed modules (cached in Redis/in-memory, 5min TTL). Disabled modules return 403.
- **Offline-Resilient:** Mobile app queues critical writes locally. Background sync when connectivity returns. Server is source of truth.
- **Event-Driven Internals:** Modules communicate via event bus. Attendance marked → notification triggered. Assessment scored → report update.

### Request Flow

1. Client sends request with Supabase JWT
2. Gateway validates JWT, extracts `tenant_id` from `app_metadata`
3. Module gating checks `tenant_modules` (Redis/in-memory cached)
4. Role permission check
5. Route → Controller → Service → Repository → Database
6. Database: `set_config('app.tenant_id', id)` → RLS enforces isolation

---

## 5. Module Structure & Licensing

### Module Package Layout

```
modules/<domain>/
  ├── routes.ts        # API routes
  ├── controller.ts    # Request/response (thin)
  ├── service.ts       # Business logic + transactions
  ├── repository.ts    # DB queries (Supabase client)
  ├── types.ts         # DTOs, interfaces, enums
  ├── events.ts        # Event producers/consumers
  ├── validators.ts    # Zod validation schemas
  ├── module.ts        # Registration (key, deps, routes)
  └── __tests__/       # Tests
```

### Licensing Tables

**tenants:** id, name, slug, plan_tier, status, settings (JSONB)

**tenant_modules:** tenant_id, module_id, enabled, licensed_until, max_users, config_overrides (JSONB)

**plan_templates:** id, name (Basic/Pro/Enterprise), included_modules[], max_users, price

**How it works:** Plan template pre-populates `tenant_modules` on signup. Admins can override per module. `config_overrides` JSONB allows per-tenant customization (custom activity types, RAG thresholds, etc).

---

## 6. Database Design & Multi-Tenancy

### Tenant Isolation (RLS)

1. Supabase Auth issues JWT with `tenant_id` in `app_metadata`
2. Backend sets PostgreSQL session variable: `set_config('app.tenant_id', id)`
3. Every table has RLS policy: `tenant_id = current_setting('app.tenant_id')::uuid`

Even buggy application code cannot leak cross-tenant data.

### Core Tables

**users:** id (= Supabase auth.uid), tenant_id, role, designation, email, phone, is_active, notification_prefs (JSONB)

**schools:** id, tenant_id, name, address, district, geo_lat, geo_lng, geo_radius_m, donor_id, academic_year, status

**students:** id, tenant_id, unique_student_id, school_id, name, grade, section, academic_year, is_active

**teachers:** id, tenant_id, school_id, name, subject, rubric_level, is_champion

**donors:** id, tenant_id, name, contact_info (JSONB), assigned_school_ids[]

**session_plans:** id, tenant_id, school_id, trainer_id, date, start_time, end_time, topic, status, approval_id

**session_attendance:** id, tenant_id, session_id, student_id, is_present, trainer_present, geo_lat, geo_lng, geo_distance_m, geo_flagged, photo_url, volunteer_name, volunteer_hours, marked_at, synced_at

**non_session_attendance:** id, tenant_id, trainer_id, date, activity_type, duration_type, start_time, end_time, remarks, supporting_doc_url, approval_id, synced_at

**assessments:** id, tenant_id, student_id, template_id, type (baseline/endline), omr_image_url, omr_status, answers (JSONB), scores (JSONB), total_score, max_score, confidence

**assessment_templates:** id, tenant_id, name, grade, subject, num_questions, answer_key (JSONB), question_skill_map (JSONB), scoring_rubric (JSONB), omr_layout_config (JSONB)

**approval_workflows:** id, tenant_id, workflow_type, entity_type, entity_id, current_step, status, submitted_by, decided_by, comments, audit_trail (JSONB)

**workflow_definitions:** id, tenant_id, entity_type, submit_roles[], steps (JSONB), on_approve, on_reject, notification_config (JSONB)

**notifications:** id, tenant_id, user_id, channel, type, title, body, entity_type, entity_id, is_read, sent_at

**audit_logs:** id, tenant_id, user_id, action, entity_type, entity_id, changes (JSONB), ip_address, timestamp

### Database Design Decisions

1. **`tenant_id` on every table** — Even when derivable via joins. Enables uniform RLS. Slight denormalization, massive security benefit.

2. **JSONB for flexible fields** — Scores, audit trails, config overrides, notification prefs. Per-tenant customization without schema migrations. Validated at application layer.

3. **Composite indexes** — `(tenant_id, trainer_id, date)` on attendance, `(tenant_id, school_id, academic_year)` on students, `(tenant_id, status)` on workflows. Keeps tenant-scoped queries fast.

4. **Soft deletes** — All entities use `deleted_at` instead of hard deletes. Audit compliance and data recovery.

---

## 7. Workflow Engine

### State Machine

```
DRAFT ──submit()──> PENDING ──approve()──> APPROVED (locked)
                       ├──reject()──> REJECTED ──resubmit()──> PENDING
                       └──clarify()──> CLARIFICATION ──respond()──> PENDING
```

### Workflow Definition (per tenant)

```json
{
  "entity_type": "trainer_competency",
  "submit_roles": ["trainer"],
  "steps": [
    { "order": 1, "approver_role": "project_manager", "can_clarify": true, "auto_escalate_hours": 48, "escalate_to": "admin" },
    { "order": 2, "approver_role": "admin", "can_clarify": false }
  ],
  "on_approve": "lock_entity",
  "on_reject": "notify_submitter"
}
```

### All Workflow Types

| Workflow | Submitter | Approval Chain | Escalation | On Approve |
|---|---|---|---|---|
| Non-session attendance | Trainer | PM | 48hrs → Admin | Lock entry |
| Session plan | Trainer | PM | 48hrs → Admin | Activate session |
| Teacher rubric | Trainer | PM (bulk) | 48hrs → Admin | Lock rubric |
| School growth rubric | Trainer | PM | 48hrs → Admin | Lock rubric |
| Leave request | Trainer | PM | 24hrs → Admin | Update calendar |
| Trainer competency | Trainer | PM → Admin (2-step) | 48hrs each | Lock matrix |
| Edit locked entry | Trainer | PM | — | Unlock + re-lock |

### Engine Behavior

- **Events:** Each state transition emits events → Notification Hub routes alerts
- **Bulk ops:** `bulkApprove(ids[])` and `bulkReject(ids[], comment)` for teacher rubrics
- **Escalation cron:** Hourly job re-assigns overdue approvals to next-level role
- **Audit trail:** Every transition recorded as append-only JSONB array

---

## 8. Offline Sync

### What Works Offline

| Offline | Online Only |
|---|---|
| Session attendance marking | Approval workflows |
| Non-session attendance logging | Reports & dashboards |
| Assessment score entry | Session plan creation |
| OMR photo capture (queued upload) | Student bulk import |
| Session status + volunteer details | All admin/PM operations |

### Sync Flow

```
Trainer action → Local SQLite/WatermelonDB → Detect connectivity → Batch sync API → Server validates + reconciles
```

### Conflict Resolution

Conflicts are rare because trainers only edit their own data over short offline windows (hours, not days). When they occur: **server wins, trainer gets notified**, conflict logged for PM review.

---

## 9. OMR Processing Pipeline

```
1. CAPTURE  → Trainer photographs OMR sheet [OFFLINE OK]
2. UPLOAD   → Queued locally, uploaded to S3 when online
3. QUEUE    → Backend creates job in Redis/in-memory queue
4. PROCESS  → OMR worker: perspective correction → detect bubbles → extract answers
5. SCORE    → Compare vs template answer key, compute subject/skill scores
6. RESULT   → Scores saved, trainer notified
```

**Assessment templates** (created by Admin): name, grade, subject, answer key, question-to-skill mapping, scoring rubric, OMR layout config.

**Error handling:** Low confidence → flag for manual review. Unreadable image → notify trainer to re-scan. Processing failure → retry 3x. Manual entry always available as fallback. All results reviewable before finalization.

---

## 10. Notifications

### Architecture

Event sources (workflow transitions, cron reminders, OMR completion, admin announcements) → **Notification Hub** (resolve recipients, check user preferences, route to channels, queue via Redis/in-memory) → Delivery channels (In-App via WebSocket, Push via FCM, WhatsApp via Twilio).

### Notification Types

| Notification | Recipient | In-App | Push | WhatsApp | Trigger |
|---|---|---|---|---|---|
| New approval pending | PM | Yes | Yes | Yes | Workflow submitted |
| Entry approved/rejected | Trainer | Yes | Yes | Yes | Workflow decided |
| Attendance not logged | Trainer | Yes | Yes | — | Cron (6 PM) |
| Approval pending > 48hrs | PM | Yes | — | Yes | Escalation cron |
| Trainer no log X days | PM | Yes | — | Yes | Cron daily |
| OMR processing complete | Trainer | Yes | Yes | — | OMR worker |
| Admin announcement | All | Yes | Yes | — | Admin action |
| Cancellation rate > 30% | Trainer | Yes | — | — | Cron weekly |

**User preferences:** Each user has JSONB notification_preferences. They toggle notification types per channel. Hierarchy: user pref > tenant default > system default.

---

## 11. Role-Based Access Control

### Role Structure

**Operational hierarchy:** Trainer → Project Manager → Admin. PM inherits all Trainer permissions and adds approvals + management.

**Donor:** Separate branch — view-only access to funded schools. Not part of the operational hierarchy.

### Permission Matrix (system defaults, configurable per tenant)

| Action | Trainer | PM | Donor | Admin |
|---|---|---|---|---|
| Mark attendance | Yes | Yes | View | Yes |
| Log non-session activity | Yes | Yes | — | Yes |
| Approve/reject entries | — | Yes | — | Yes |
| Create session plans | Yes | Yes | — | Yes |
| Upload student data | Yes | Yes | — | Yes |
| View reports | Own data | Cluster | Funded schools | All |
| Manage schools | Yes | Yes | — | Yes |
| Manage trainers | — | Yes | — | Yes |
| Set targets & config | — | — | — | Yes |
| Module & license config | — | — | — | Yes |

### Per-Tenant Permission Overrides

Role types are system-defined (Trainer, PM, Donor, Admin) — tenants don't create custom roles. But each tenant can toggle specific permissions on/off per role.

**Data model:**

- **permissions:** id, key (e.g. "mark_attendance", "create_session"), module_id, description
- **role_default_permissions:** role, permission_id, granted — system-wide defaults, works out of the box
- **tenant_role_overrides:** tenant_id, role, permission_id, granted — sparse, only stores exceptions

**Resolution:** System defaults → merge tenant overrides → cache in Redis/in-memory per tenant+role.

**Example:** Tenant A uses defaults (Trainers can create sessions). Tenant B overrides: Trainers cannot create sessions (only PMs can). Only the exception is stored — one row in `tenant_role_overrides`.

### Four Layers of Access Control

1. **RLS (Database):** Tenant isolation — no cross-tenant access possible
2. **Module Gating (Middleware):** Unlicensed modules return 403
3. **Role Permissions (Controller):** Action-level checks per user role, resolved from defaults + tenant overrides
4. **Data Scoping (Query):** Role determines visible data scope (see below)

### Data Scoping

| Role | Sees |
|---|---|
| Trainer | Own assigned schools + own entries |
| PM | All trainers + schools in their cluster |
| Donor | Only funded schools and related metrics |
| Admin | Everything within the tenant |

---

## 12. Reporting & Analytics

Two approaches under consideration. May coexist as a hybrid.

### Approach A: Traditional Dashboards + Materialized Views

Pre-built dashboards backed by PostgreSQL materialized views refreshed nightly.

**Dashboards:** Trainer Utilization, Attendance, Leave Summary, Student Improvement, School RAG, Teacher Readiness, Monthly Productivity, Donor Impact, Volunteer Report, Trainer Competency.

**Pros:** Instant loads, no LLM cost, 100% accurate, works offline (cacheable), familiar UX.
**Cons:** Every new report = dev work, can't answer ad-hoc questions, data up to 24hrs stale.

### Approach B: AI Natural Language Query Engine

Users ask questions in English → system generates SQL → validates → executes → renders charts. Based on KindKart's proven production system.

**Flow:** User query → keyword-match domain schema docs → LLM generates SQL + chart metadata → SQL validator (table whitelist, no DML, no injection) → execute → render (table/bar/line/pie/kpi).

**Multi-tenant adaptations:** Auto-inject `tenant_id` in SQL, RLS as second layer, module-aware schema loading (unlicensed module tables hidden from LLM), role-based query scoping.

**Pros:** Self-service (zero dev work per report), infinite variety, proven in production, less frontend work.
**Cons:** LLM cost per query, 2-5s latency, ~90-95% accuracy, requires internet, non-deterministic.

### Possible Hybrid

Build 3-5 critical dashboards traditionally (trainer daily view, PM approval queue, school RAG, attendance summary) — these must be fast and work offline. Use AI engine for everything else (ad-hoc analysis, donor reports, year-over-year comparisons). Users can pin useful queries as reusable dashboards.

### RAG Scoring (shared across both approaches)

Auto-computed Red/Amber/Green per school based on configurable KPI thresholds:

| | GREEN | AMBER | RED |
|---|---|---|---|
| Attendance | > 80% | 60-80% | < 60% |
| Sessions completed | > 90% | 70-90% | < 70% |
| Assessment improvement | > 15% | 5-15% | < 5% |

Thresholds and KPI weights configurable per tenant via `tenant_settings` JSONB.

### Export Engine (shared)

- **Excel:** Job queue (Redis/in-memory) for large exports → generate on backend → S3 → presigned download URL
- **PDF:** HTML → PDF via Puppeteer/Playwright, tenant-branded, schedulable (monthly/quarterly auto-generation)

---

## 13. Phasing

### Phase 1 — Core Platform
Modules: M1-M6, M9 + Workflow Engine + Notification Hub (in-app + push) + Offline sync.
**Exit criteria:** Trainers can log daily attendance, PMs can approve entries, basic reports available.

### Phase 2 — Assessment & Donor
Modules: M7, M8 + OMR worker + WhatsApp notifications + Geo-tagging + Advanced reporting.
**Exit criteria:** OMR sheets scanned and auto-scored, donors can view funded school metrics.

### Phase 3 — Advanced
AI NL query engine (if not in Phase 1), scheduled auto-reports, activity calendar, advanced analytics, performance tuning.

---

## 14. Risk Register

| Risk | Mitigation |
|---|---|
| Missed RLS policy → data leakage | Integration tests verify tenant isolation on every table |
| OMR accuracy too low | Manual review fallback, manual entry always available |
| Offline sync conflicts | Server-wins + notification, short offline window reduces probability |
| LLM generates incorrect SQL | SQL validator blocks dangerous queries, results always reviewable |
| Module boundaries erode | Lint rules enforcing import boundaries, code review |
| Supabase vendor lock-in | Auth + DB abstracted behind interfaces, can migrate independently |

---

## 15. Open Decisions

| Decision | Status |
|---|---|
| Reporting approach (Traditional vs AI vs Hybrid) | Pending — both documented |
| OMR template standard (custom vs standard layout) | Pending — needs sample sheets |
| School calendar (per-school vs shared tenant) | Pending |
| Student section transfers mid-year | Pending |
| Voice-to-text for trainer remarks | Pending — nice to have |

---

*Generated April 9, 2026. Living document — update as decisions are finalized.*
