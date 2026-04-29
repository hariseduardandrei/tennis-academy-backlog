# Tennis Academy Management App — Project Backlog (MVP → v1)
Date: 2026-04-29  
Scope: Performance tennis academy (50–60 students, 3 admins, 10–15 coaches, 1 fitness trainer, 4 courts)  
Platforms: Web (staff + student portal) → Mobile (React Native)  
Backend: Kotlin (Ktor) + PostgreSQL  
Payments: Manual monthly membership (no online payments)  
Languages: Romanian (RO) + English (EN)  
Notes: Two note fields required — internal (staff-only) and student-visible.

---

## 0) Repositories & structure
**Repos**
- `tennis-academy-meta` (git submodules, docs, high-level scripts)
- `tennis-academy-backend` (Ktor API, DB, auth)
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

## 4) Backend backlog (Ktor + Postgres)
### 4.1 Foundation
- [ ] Initialize Ktor project (Gradle Kotlin DSL)
- [ ] Add configuration system (HOCON or YAML) for dev/prod
- [ ] Add Dockerfile for API
- [ ] Add `docker-compose.yml` for local dev: API + Postgres
- [ ] Add Flyway migrations + baseline migration
- [ ] Add Exposed + HikariCP + Postgres driver
- [ ] Add structured logging (request id, user id where possible)

Acceptance criteria:
- `docker compose up` starts API + DB, API health endpoint responds.

### 4.2 Auth & users
- [ ] DB: `users` table (uuid, email unique, password_hash, role, language, created_at)
- [ ] Password hashing (BCrypt/Argon2)
- [ ] JWT auth (access token)
- [ ] Endpoints:
  - [ ] `POST /auth/login`
  - [ ] `GET /me`
- [ ] Admin-only: create staff users endpoint (coach/trainer/admin)
- [ ] RBAC middleware/route guards

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
- [ ] DB: `membership_month` unique(student_id, year, month)
- [ ] Endpoints:
  - [ ] `GET /billing/month?year=YYYY&month=M` (admin)
  - [ ] `PATCH /billing/students/{id}/month?year=YYYY&month=M` (admin)
- [ ] Report endpoint:
  - [ ] `GET /billing/overdue?year=YYYY&month=M` (admin)

Acceptance criteria:
- Admin can mark paid/unpaid and see overdue list.

### 4.9 Quality & security
- [ ] Input validation (Kotlinx serialization + validation layer)
- [ ] Standard error format (problem+json style)
- [ ] Rate-limit login attempts (basic)
- [ ] CORS configuration for web + mobile
- [ ] Unit tests for conflict logic and permissions

---

## 5) Web app backlog (Next.js)
### 5.1 Foundation
- [ ] Initialize Next.js (TypeScript)
- [ ] Add MUI + theme (light, modern)
- [ ] Add i18n (RO/EN) with per-user preference
- [ ] API client layer (fetch wrapper) + typed DTOs
- [ ] Auth:
  - login page
  - token storage strategy (httpOnly cookie preferred; otherwise secure storage)
  - route protection by role

Acceptance criteria:
- User can log in, language switches, role-based navigation works.

### 5.2 Staff UX flows
#### Today dashboard (courts view)
- [ ] “Today” page:
  - Court 1–4 columns/cards
  - sessions grouped by court with time
  - quick open “Complete session”
- [ ] Conflict indicator badges

Acceptance criteria:
- Staff can see today’s schedule in one screen and open a session quickly.

#### Week schedule (by court)
- [ ] Week view grid:
  - columns = courts
  - time rows
  - session blocks
- [ ] Create/edit session modal:
  - time, court, staff user, type, title
  - student picker (search)
- [ ] Conflict errors displayed clearly

Acceptance criteria:
- Staff can create/edit sessions and assign students; conflicts are shown.

#### Students
- [ ] Students list with search/filter (active/inactive)
- [ ] Student profile:
  - info
  - recent sessions + loads
  - membership status (month)
- [ ] Admin action: create student login (set temp password and show once)

Acceptance criteria:
- Staff can manage student profiles; admin can create login.

#### Session completion
- [ ] Session completion UI:
  - list students in session
  - attendance toggle
  - RPE input 1–10
  - duration (default from schedule)
  - internal notes (staff-only)
  - student-visible notes
  - save button
- [ ] UX speed:
  - keyboard friendly
  - “copy previous values” optional (phase 2)

Acceptance criteria:
- Coach can complete a session in under ~60 seconds.

#### Billing (admin)
- [ ] Month view table:
  - students + status + due date + amount
- [ ] Overdue filter
- [ ] Mark paid/unpaid quickly

Acceptance criteria:
- Admin can see who is overdue and update status quickly.

### 5.3 Student portal (web)
- [ ] Student home:
  - upcoming sessions
  - membership status
- [ ] My Schedule week view (simpler than staff)
- [ ] History:
  - list sessions with load + student notes
  - never show internal notes

Acceptance criteria:
- Student sees their schedule and notes in RO/EN.

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

### 7.2 Student MVP (first mobile release)
- [ ] Login
- [ ] My Schedule (week)
- [ ] Session history (load + student notes)
- [ ] Membership status

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