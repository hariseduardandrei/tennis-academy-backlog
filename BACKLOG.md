# Tennis Academy Management App — Project Backlog (MVP → v1)
Date: 2026-04-29  
Scope: Performance tennis academy (50–60 students, 3 admins, 10–15 coaches, 1 fitness trainer, 4 courts)  
Platforms: Web (staff + student portal) → Mobile (React Native)  
Backend: Kotlin (Spring Boot 3) + PostgreSQL  
Payments: Manual monthly membership (no online payments)  
Languages: Romanian (RO) + English (EN)  
Notes: Two note fields required — internal (staff-only) and student-visible.

---

## 0) Repositories & structure
**Repos**
- `tennis-academy-meta` (git submodules, docs, high-level scripts)
- `tennis-academy-backend` (Spring Boot 3 API, DB, auth)
- `tennis-academy-web` (Next.js web app)
- `tennis-academy-mobile` (React Native / Expo)

**Meta repo tasks**
- [ ] Add submodules: backend, web, mobile
- [ ] Add root docs: `README.md`, `BACKLOG.md`, `ARCHITECTURE.md`
- [ ] Add `scripts/` (optional): helper scripts for local dev (start/stop, reset DB)

Acceptance criteria:
- `git clone --recurse-submodules ...` works and README documents update procedure for submodules.

---

## 1) Product goals & non-functional requirements
### Goals (MVP)
- Replace spreadsheets/WhatsApp with a simple system for:
  - Scheduling by court
  - Assigning students (often 2:1)
  - Attendance tracking
  - RPE + training load per student per session
  - Session notes (internal + student-visible)
  - Manual monthly membership status (paid/unpaid)
  - Student login portal (view schedule + history + membership)

### Non-functional requirements
- Low hosting cost (single small VPS to start)
- Simple UX for non-technical users
- Secure auth + role-based access
- Auditability basics (who created/updated sessions)
- English + Romanian UI, per-user language preference
- Timezone-safe scheduling (single academy timezone)

---

## 2) Roles & permissions (MVP)
Roles:
- **ADMIN**: full access, manage users, billing, all students, all sessions
- **COACH**: manage sessions they run (or all sessions if configured), complete sessions, write notes, see internal notes
- **TRAINER**: same as coach but primarily fitness sessions
- **STUDENT**: view only own schedule/history + student-visible notes + membership status

Permissions rules (must-have):
- Students must NOT see:
  - Internal notes
  - Other students’ data
- Staff must be able to:
  - Create/edit sessions
  - Assign students
  - Complete sessions (attendance + RPE/load + notes)

---

## 3) Data model (MVP)
Minimum entities:
- Users (auth, role, language)
- Students (profile + optional linked user account)
- Courts (4)
- Sessions (time, court, staff user, type, title)
- SessionStudents (many-to-many)
- SessionStudentMetrics (attendance + RPE + duration + load + internal_notes + student_notes)
- MembershipMonth (monthly membership per student)

Computed:
- `load = duration_minutes * rpe`

---

## 4) Backend backlog (Spring Boot 3 + Postgres)
### 4.1 Foundation
- [x] Initialize Spring Boot 3 project (Gradle Kotlin DSL)
- [x] Add configuration system (application.yml) for dev/prod profiles
- [x] Add Dockerfile for API (multi-stage, eclipse-temurin:21)
- [x] Add `docker-compose.yml` for local dev: API + Postgres
- [x] Add Flyway migrations + baseline migration
- [x] Add Spring Data JPA + HikariCP + Postgres driver
- [ ] Add structured logging (request id, user id via MDC)

Acceptance criteria:
- `docker compose up` starts API + DB, API health endpoint responds. ✅

### 4.2 Auth & users
- [ ] DB: `users` table (uuid, email unique, password_hash, role, language, created_at)
- [ ] Password hashing (BCrypt — `PasswordEncoder` bean already configured)
- [x] JWT auth scaffold (jjwt 0.12, stateless `SecurityFilterChain`)
- [ ] JWT filter implementation (`JwtAuthenticationFilter`)
- [ ] Endpoints:
  - [ ] `POST /auth/login`
  - [ ] `GET /me`
- [ ] Admin-only: create staff users endpoint (coach/trainer/admin)
- [ ] RBAC via `@EnableMethodSecurity` + `@PreAuthorize` on handlers

Acceptance criteria:
- Admin can create coach user, coach can log in, `/me` returns role + language.

### 4.3 Students
- [ ] DB: `students` table (uuid, user_id nullable unique, names, dob, phone, status, notes, created_at)
- [ ] Endpoints:
  - [ ] `GET /students` (staff)
  - [ ] `POST /students` (staff)
  - [ ] `PATCH /students/{id}` (staff)
- [ ] Endpoint to create/link student login:
  - [ ] `POST /students/{id}/create-account` (admin)
    - creates user with role STUDENT + temp password OR sends reset flow (phase 2)

Acceptance criteria:
- Staff can create student profile; admin can create student login; student can log in.

### 4.4 Courts
- [ ] DB: `courts` table, seed Court 1–4 via migration
- [ ] Endpoint:
  - [ ] `GET /courts`

Acceptance criteria:
- Court list returns 4 courts.

### 4.5 Scheduling (manual)
- [ ] DB: `sessions`, `session_students`
- [ ] Session fields:
  - start_at, end_at (timestamptz)
  - court_id
  - staff_user_id (coach/trainer)
  - session_type (enum/text)
  - title (optional)
  - created_by, updated_at
- [ ] Endpoints:
  - [ ] `GET /schedule/week?start=YYYY-MM-DD` (staff)
  - [ ] `POST /sessions` (staff)
  - [ ] `PATCH /sessions/{id}` (staff)
  - [ ] `DELETE /sessions/{id}` (staff)
  - [ ] `PUT /sessions/{id}/students` (staff; replace list)

- [ ] Conflict validation (must-have):
  - court double-booking (overlapping time)
  - staff double-booking
  - student double-booking

Acceptance criteria:
- Creating a conflicting session returns a validation error with a friendly message.

### 4.6 Session completion (attendance + RPE/load + notes)
- [ ] DB: `session_student_metrics` table (PK session_id+student_id)
- [ ] Fields:
  - attendance_status (PRESENT/LATE/ABSENT)
  - duration_minutes (default from session length, editable)
  - rpe (1–10)
  - load (stored)
  - internal_notes (text)
  - student_notes (text)
- [ ] Endpoint:
  - [ ] `POST /sessions/{id}/complete` (staff)
    - upserts metrics per student
    - calculates load

Acceptance criteria:
- Completing a session stores per-student metrics and notes; load is computed and retrievable.

### 4.7 Student portal endpoints
- [ ] `GET /my/schedule/week?start=YYYY-MM-DD` (student)
- [ ] `GET /my/history?limit=&offset=` (student)
  - must include only student-visible notes
  - must hide internal notes and other students

Acceptance criteria:
- Student can only access own assigned sessions and sees only student_notes.

### 4.8 Billing (monthly membership, manual)
- [x] DB: `membership_month` unique(student_id, year, month)
- [x] Endpoints:
  - [x] `GET /billing/month?year=YYYY&month=M` (admin)
  - [x] `PATCH /billing/students/{id}/month?year=YYYY&month=M` (admin)
- [x] Report endpoint:
  - [x] `GET /billing/overdue?year=YYYY&month=M` (admin)

Acceptance criteria:
- ✓ Admin can mark paid/unpaid and see overdue list.

### 4.9 Quality & security
- [ ] Input validation (Bean Validation — `@Valid` on request DTOs)
- [x] Standard error format (problem+json style via `@ControllerAdvice`)
- [ ] Rate-limit login attempts (basic — e.g. Bucket4j or in-memory counter)
- [ ] CORS configuration for web + mobile origins
- [ ] Unit tests for conflict logic and permissions (JUnit 5 + MockK)
- [ ] Integration tests with Testcontainers (Postgres)

---

## 5) Web app backlog (Next.js)
### 5.1 Foundation
- [x] Initialize Next.js (TypeScript)
- [x] Add MUI + theme (light, modern)
- [x] Add i18n (RO/EN) with per-user preference
- [x] API client layer (fetch wrapper) + typed DTOs
- [x] Auth:
  - [x] login page
  - [x] token storage strategy (implemented with `localStorage`; httpOnly cookie still preferred target)
  - [x] route protection by role

Acceptance criteria:
- User can log in, language switches, role-based navigation works.

### 5.2 Staff UX flows
#### Today dashboard (courts view)
- [x] “Today” page:
  - [x] Court 1–4 columns/cards
  - [x] sessions grouped by court with time
  - [x] quick open “Complete session”
- [ ] Conflict indicator badges

Acceptance criteria:
- Staff can see today’s schedule in one screen and open a session quickly.

#### Week schedule (by court)
- [x] Week view grid:
  - [x] columns = courts
  - [x] time rows
  - [x] session blocks
- [x] Create/edit session modal:
  - [x] time, court, staff user, type, title
  - [x] student picker (search)
  - [x] staff typeahead by first name/last name with selectable suggestions
- [x] Conflict errors displayed clearly
- [x] Quick actions on session block:
  - [x] complete
  - [x] edit
  - [x] delete (with confirmation)

Acceptance criteria:
- Staff can create/edit sessions and assign students; conflicts are shown.

#### Students
- [x] Students list with search/filter (active/inactive)
- [ ] Student profile:
  - [x] info
  - recent sessions + loads
  - membership status (month)
- [x] Admin action: create student login (set temp password and show once)

Acceptance criteria:
- Staff can manage student profiles; admin can create login.

#### Session completion
- [x] Session completion UI:
  - [x] list students in session
  - [x] attendance toggle
  - [x] RPE input 1–10
  - [x] duration (default from schedule)
  - [x] internal notes (staff-only)
  - [x] student-visible notes
  - [x] save button
- [ ] UX speed:
  - keyboard friendly
  - “copy previous values” optional (phase 2)

Acceptance criteria:
- Coach can complete a session in under ~60 seconds.

#### Billing (admin)
- [x] Month view table:
  - [x] students + status + due date + amount
- [x] Overdue filter
- [x] Mark paid/unpaid quickly

Acceptance criteria:
- Admin can see who is overdue and update status quickly.

### 5.3 Student portal (web)
- [ ] Student home:
  - [x] upcoming sessions
  - [ ] membership status
- [x] My Schedule week view (simpler than staff)
- [x] History:
  - [x] list sessions with load + student notes
  - [x] never show internal notes

Acceptance criteria:
- Student sees their schedule and notes in RO/EN.

### 5.4 Implemented frontend inventory (current web parity baseline for mobile)
Purpose: explicit list of what exists today in web, so mobile can target 1:1 parity.

#### Routing and role entry points
- [x] `/` auto-redirect:
  - unauthenticated -> `/login`
  - staff (`ADMIN`/`COACH`/`TRAINER`) -> `/today`
  - `STUDENT` -> `/student`
- [x] `/login` email/password login with API error mapping (`401` -> invalid credentials)
- [x] Role-guarded layouts:
  - [x] staff area (`/today`, `/schedule`, `/students`, `/students/[id]`, `/sessions/[id]/complete`)
  - [x] admin-only area (`/billing`, `/users`)
  - [x] student-only area (`/student`, `/student/schedule`, `/student/history`)

#### Global app shell and UX primitives
- [x] Shared responsive app shell (desktop permanent drawer + mobile temporary drawer)
- [x] Role-aware navigation menus (staff vs student)
- [x] User avatar menu with logout action
- [x] Language toggle (`RO`/`EN`) persisted in `localStorage` (`ta_locale`)
- [x] Global snackbar queue for success/error feedback
- [x] Unified API client with typed DTOs, bearer token injection, and problem+json error propagation

#### Staff pages and flows
- [x] `Today` dashboard (`/today`):
  - [x] sessions grouped by court
  - [x] time range + session type chip + title
  - [x] quick action -> session completion page
- [x] Weekly schedule (`/schedule`):
  - [x] week navigation (prev/next/current week)
  - [x] day sections with court-column time grid
  - [x] session visual blocks (colored by type)
  - [x] quick actions per block: complete, edit, delete
  - [x] create/edit modal (`SessionModal`)
    - [x] start/end datetime
    - [x] court select
    - [x] coach select via typeahead (search by first/last name; email fallback)
    - [x] session type + optional title
    - [x] multi-student picker
    - [x] create and patch session requests send selected `staffUserId`
    - [x] friendly conflict messages for `COURT_CONFLICT` / `STAFF_CONFLICT` / `STUDENT_CONFLICT`
  - [x] delete confirmation dialog
- [x] Students list (`/students`):
  - [x] server-backed search
  - [x] status filter (`ALL`/`ACTIVE`/`INACTIVE`)
  - [x] create student modal (first name, last name, phone, DOB, status)
  - [x] open profile action
- [x] Student profile (`/students/[id]`):
  - [x] view/edit personal and status fields
  - [x] edit notes
  - [x] admin-only create account action
  - [x] one-time temporary password confirmation dialog after account creation
- [x] Session completion (`/sessions/[id]/complete`):
  - [x] per-student attendance
  - [x] duration + RPE fields
  - [x] live load preview (`duration * rpe`)
  - [x] student notes + internal notes
  - [x] submit completion payload to backend

#### Admin pages and flows
- [x] Billing (`/billing`):
  - [x] month/year selectors
  - [x] tabs: monthly table and overdue report
  - [x] per-student status actions (`PAID`, `DUE`, `WAIVED`)
- [x] Staff users (`/users`):
  - [x] list staff users
  - [x] create staff user (first name, last name, email, password, role, language)
  - [x] edit staff user (first name, last name, role, language)
  - [x] role labels with user-facing names (`Fitness trainer`)

#### Student portal pages and flows
- [x] Student home (`/student`): upcoming sessions summary (next sessions)
- [x] Student schedule (`/student/schedule`): weekly navigation and per-day cards
- [x] Student history (`/student/history`):
  - [x] paginated history (`load more`)
  - [x] attendance chip
  - [x] duration, RPE, load and student notes visibility

#### Current web gaps (explicit)
- [ ] Student home membership status card not implemented yet
- [ ] Staff student profile: recent sessions + load panel not implemented yet
- [ ] Staff student profile: membership month panel not implemented yet

---

## 6) UX/UI Design backlog (deliverables)
### 6.1 Design principles
- Simple, high-contrast, large touch targets
- “One screen = one job”
- Avoid dense tables unless necessary
- Always show Court 1–4 clearly
- Make conflicts impossible to ignore

### 6.2 Design deliverables
- [ ] Global theme:
  - typography scale
  - spacing system
  - color palette
  - components: buttons, chips, badges, forms
- [ ] Key screens wireframes (RO first, then EN):
  - Login
  - Today dashboard
  - Week schedule (courts grid)
  - Create/edit session modal
  - Session completion
  - Students list + profile
  - Billing month + overdue
  - Student home + schedule + history
- [ ] UI copy dictionary (i18n keys) for RO/EN

Acceptance criteria:
- A coach can: open Today → open session → mark attendance + RPE + notes → save without guidance.

---

## 7) Mobile backlog (React Native / Expo)
### 7.1 Foundation
- [ ] Initialize Expo app (TypeScript)
- [ ] Navigation (tabs or stack)
- [ ] API client + auth token handling
- [ ] i18n (RO/EN) matching web keys

### 7.1.1 Prioritized delivery roadmap
#### P0 — Student parity core (recommended first release)
Goal: ship the student mobile app with parity for the web flows that already exist and are stable.

- [ ] Auth foundation
  - [ ] login
  - [ ] persisted session
  - [ ] `/me` bootstrap and role-aware redirect
  - [ ] logout
- [ ] Student home
  - [ ] upcoming sessions list
- [ ] Student schedule
  - [ ] weekly navigation
  - [ ] grouped sessions by day
- [ ] Student history
  - [ ] paginated history
  - [ ] attendance, duration, RPE, load
  - [ ] student-visible notes only
- [ ] Shared mobile foundations needed by all screens
  - [ ] loading / empty / error states
  - [ ] snackbar / toast feedback
  - [ ] RO/EN parity with web keys

#### P0.5 — Student parity completion blockers / stretch items
Goal: close student-facing gaps that are not fully present in web yet or can follow immediately after P0.

- [ ] membership status on student home
- [ ] membership status details screen/card if needed by product

Note: this phase depends on finalizing the corresponding web/product behavior for membership in the student portal.

#### P1 — Coach/staff operational mobile flows
Goal: enable staff to operate daily from phone when away from desktop.

- [ ] Today dashboard parity
- [ ] Session completion parity
- [ ] Minimal weekly schedule parity
  - [ ] browse week schedule
  - [ ] open session details
  - [ ] create/edit session
  - [ ] coach typeahead selector
  - [ ] student picker
  - [ ] conflict error handling

#### P2 — Extended staff/admin parity
Goal: bring the remaining desktop administration flows to mobile if product value justifies it.

- [ ] Students management parity
  - [ ] students list
  - [ ] student profile edit
  - [ ] admin create student account
- [ ] Billing parity
- [ ] Staff users parity
- [ ] advanced UX polish
  - [ ] keyboard-optimized session completion
  - [ ] faster bulk actions
  - [ ] offline-friendly enhancements (optional)

### 7.2 Student MVP (first mobile release)
- [ ] Login
- [ ] My Schedule (week)
- [ ] Session history (load + student notes)
- [ ] Membership status

### 7.2.1 Mobile parity checklist from current web (P0)
- [ ] Auth parity:
  - [ ] login with same backend contract (`POST /auth/login`, `GET /me`)
  - [ ] role-aware root redirect logic parity
  - [ ] persistent auth + logout parity
- [ ] Student Home parity (`/student`):
  - [ ] upcoming sessions list (date/time, court, coach, session type, title)
- [ ] Student Schedule parity (`/student/schedule`):
  - [ ] week navigation prev/next/today
  - [ ] per-day grouped sessions
- [ ] Student History parity (`/student/history`):
  - [ ] pagination (`limit`/`offset`, load more)
  - [ ] attendance, duration, RPE, load, student notes
  - [ ] never expose internal notes
- [ ] i18n parity:
  - [ ] reuse same translation keys as web (`RO` + `EN`)

### 7.2.2 Staff/coach mobile parity candidates (derived from web, P1/P2)
- [ ] Today dashboard parity (`/today`)
- [ ] Week schedule parity (`/schedule`) including:
  - [ ] create/edit session
  - [ ] coach typeahead selector by first/last name
  - [ ] student multi-select
  - [ ] conflict error handling
- [ ] Session completion parity (`/sessions/{id}/complete`)
- [ ] Students parity (`/students`, `/students/{id}`)
- [ ] Admin-only parity (P2 candidate): billing + staff users

Acceptance criteria:
- Student can do everything available in the student web portal on mobile.

### 7.3 Coach MVP (optional phase)
- [ ] Today sessions
- [ ] Complete session flow (attendance + RPE + notes)

Acceptance criteria:
- Coach can complete sessions on phone if needed.

### 7.4 Push notifications (phase 2)
- [ ] Notify student when session changes/cancels (requires backend events + FCM/APNs)

---

## 8) Deployment backlog (low-cost VPS)
### 8.1 Environments
- **Local**: docker compose
- **Staging** (optional but recommended): same VPS with separate domain/subdomain
- **Production**: single VPS

### 8.2 Infrastructure
- [ ] VPS (small instance)
- [ ] Docker + Docker Compose
- [ ] Reverse proxy: Caddy (auto HTTPS) or Nginx
- [ ] Domains:
  - `api.<domain>`
  - `app.<domain>`
- [ ] Postgres persistent volume

Acceptance criteria:
- HTTPS enabled, API reachable, web served, DB persists across restarts.

### 8.3 CI/CD (GitHub Actions)
Backend repo:
- [ ] Build + tests on PR
- [ ] Build Docker image on main
- [ ] Deploy via SSH to VPS (pull image, restart compose)

Web repo:
- [ ] Lint + build on PR
- [ ] Build artifact / Docker image
- [ ] Deploy to VPS (static via Next output or container)

Mobile repo:
- [ ] Lint on PR
- [ ] EAS build pipeline (later)

### 8.4 Backups & monitoring
- [ ] Nightly Postgres backup (pg_dump) to:
  - low-cost object storage (Backblaze B2) or another server
- [ ] Basic monitoring:
  - uptime check endpoint
  - log rotation
  - disk space alerts

Acceptance criteria:
- Restore procedure documented and tested once.

---

## 9) Milestones
### Milestone M0 — Project ready (1–2 days)
- Repos + submodules
- Local dev up (API + DB + web skeleton)

### Milestone M1 — Scheduling + students (1–2 weeks)
- Auth, roles, students CRUD
- Courts seeded
- Week schedule create/edit + conflict checks

### Milestone M2 — Completion + student portal (1–2 weeks)
- Session completion (attendance + RPE/load + notes)
- Student login + my schedule + history

### Milestone M3 — Billing + polish (1 week)
- Monthly membership tracking + overdue list
- i18n RO/EN complete
- UX polish and permission hardening

### Milestone M4 — Mobile student release (2–4 weeks)
- Student mobile app parity with portal

---

## 10) Open questions (track but not blockers for MVP)
- Password reset / invite flows (email) vs manual temp passwords
- Data export requirements (CSV/Excel)
- Advanced analytics (weekly load charts, trends)
- Recurring sessions
- Multi-location support

---

## 11) Definition of Done (global)
A feature is “done” when:
- Backend endpoint exists + validated + permission-checked
- Web UI supports the full flow
- RO/EN strings added
- Basic tests added for core logic (conflicts, permissions)
- Deployed to staging (or production) and smoke-tested

---

## 12) Coach-first RPE & Performance Tracking Plan (new chapter)
Goal: make post-session completion extremely fast for coaches while producing high-quality performance data for professional tennis athletes.

### 12.1 Product outcomes (target)
- [ ] Coach can complete a full session in <= 60 seconds (target median <= 45s)
- [ ] RPE capture rate >= 90% for completed sessions
- [ ] Duration + RPE + load completeness >= 95%
- [ ] Athlete profile surfaces actionable trends (not only raw history)

Acceptance criteria:
- Session completion speed and data completeness are measurable from application telemetry.

### 12.2 RPE protocol standardization (data quality first)
- [ ] Define and document one academy-wide RPE scale (1-10) with plain-language descriptors
- [ ] Add in-app microcopy for each RPE band to reduce interpretation variance
- [ ] Enforce valid range (`rpe` 1-10) and required fields (`attendance_status`, `duration_minutes`, `rpe` when present)
- [ ] Record capture timing metadata (`captured_at`, optional `minutes_after_session_end`) for quality audits
- [ ] Keep computed load server-side only: `load = duration_minutes * rpe`

Acceptance criteria:
- Staff sees a consistent RPE definition in web + mobile; backend rejects invalid RPE payloads.

### 12.3 Coach speed UX plan (post-session flow)
#### Phase A — MVP speedups
- [ ] Default attendance to `PRESENT` for roster on open
- [ ] Prefill `duration_minutes` from scheduled duration
- [ ] Replace free numeric RPE typing with single-tap 1-10 selector
- [ ] Add sticky primary CTA: Save session
- [ ] Add progress indicator (`completed athletes / total athletes`)

#### Phase B — Performance workflow accelerators
- [ ] Quick actions: `apply to all`, `copy previous athlete`, `reset row`
- [ ] Structured quick tags for notes (technical / physical / mental / recovery)
- [ ] Optional voice-to-note input (phase 2)
- [ ] Draft autosave and retry-safe submission

Acceptance criteria:
- Coaches can complete common 6-10 athlete sessions with mostly taps and minimal typing.

### 12.4 Athlete performance model (beyond single-session load)
Derived metrics (per athlete):
- [ ] Weekly load (`sum(load)` last 7 days)
- [ ] Rolling 28-day load
- [ ] Acute:Chronic ratio (`7-day load / 28-day average weekly equivalent`)
- [ ] Attendance compliance (% present vs scheduled)
- [ ] Session monotony and strain (phase 2)

Data model additions:
- [ ] Add athlete daily/weekly aggregates table or materialized view strategy
- [ ] Add metric provenance fields (time window start/end, recompute timestamp)
- [ ] Backfill job for historical sessions

Acceptance criteria:
- Athlete profile shows trend metrics that update reliably after session completion.

### 12.5 Coach dashboard + athlete profile deliverables
- [ ] Coach dashboard card: today load, absent athletes, completion backlog
- [ ] Athlete profile snapshot: weekly load, avg RPE, attendance %, current focus tags
- [ ] Trend charts: 7-day and 28-day load curves, attendance trend
- [ ] Recent session insights list with student-visible notes separated from internal insights
- [ ] Risk/attention badges (phase 2): sudden load spikes, low attendance streaks

Acceptance criteria:
- Coaches can identify "who needs intervention today" in <= 10 seconds per athlete profile.

### 12.6 Backend/API plan
- [ ] Extend `POST /sessions/{id}/complete` contract for quick tags + capture metadata
- [ ] Add read endpoints for athlete performance summary and trends
- [ ] Add authorization checks so students never receive staff-only/internal analytics
- [ ] Add tests for load calculations, rolling windows, and permission boundaries

Acceptance criteria:
- API serves coach analytics safely; student endpoints expose only student-safe fields.

### 12.7 Web/mobile implementation plan (RO-first)
- [ ] Web: optimize existing completion page for keyboard + single-click entry
- [ ] Mobile: implement coach completion flow with thumb-zone controls and large tap targets
- [ ] Reuse identical i18n keys (RO + EN) for RPE labels, quick tags, and trend labels
- [ ] Add empty/error/loading states for all analytics cards

Acceptance criteria:
- Coach flow and terminology are consistent across web and mobile; Romanian copy is primary.

### 12.8 Delivery phases and sequencing
#### Sprint P1 (highest value)
- [ ] RPE protocol + validation + server-side load hardening
- [ ] Completion UX speedups (defaults, tap RPE, progress)
- [ ] Basic athlete snapshot metrics (weekly load, avg RPE, attendance)

#### Sprint P2
- [ ] Rolling 28-day trends + profile charts
- [ ] Quick tags + structured insight blocks
- [ ] Data quality telemetry and completeness dashboards

#### Sprint P3 (optional/premium)
- [ ] Acute:chronic ratio monitoring
- [ ] Risk flags and intervention queue
- [ ] Voice notes and advanced workflow shortcuts

### 12.9 Success measurement (must track)
- [ ] Median and p90 session completion time
- [ ] Missing-data rate (`duration`, `rpe`, `load`)
- [ ] Coach correction rate (edits after first save)
- [ ] Athlete profile weekly active usage by staff
- [ ] % athletes with at least one updated focus theme per week

Definition of done for this chapter:
- Coach flow is measurably faster, RPE data is consistent, and athlete profiles provide actionable performance insight without leaking internal notes to students.