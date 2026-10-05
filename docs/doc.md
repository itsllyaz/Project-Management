# TaskFlow — Project Management System

> A full-stack project & task management platform with a **role-aware** dashboard
> for 5 different job titles, granular permissions, JWT auth with silent token
> refresh, a QA verification workflow, and a notification engine.

---

## 📊 The Number You Asked For

| Metric | Count |
|---|---|
| **Total endpoints** | **48** |
| REST API endpoints (under `/api`) | 46 |
| Root/utility endpoints (`/`, `/_test-email`) | 2 |
| Static file mount (`/uploads`) | +1 (not a route) |
| Database models | 7 |
| Roles | 5 |
| Distinct permissions | 9 |
| Controllers | 8 (2,751 lines) |
| Frontend route areas | 5 (+ landing) |

### Where the 48 come from

| # | Group | Base path | Endpoints |
|---|---|---|---|
| 1 | Auth | `/api/auth` | **6** |
| 2 | Users | `/api/users` | **7** |
| 3 | Teams | `/api/teams` | **6** |
| 4 | Projects | `/api/projects` | **6** |
| 5 | Tasks | `/api/tasks` | **12** |
| 6 | Notifications | `/api/notifications` | **5** |
| 7 | Notification Prefs | `/api/notification-prefs` | **3** |
| 8 | Admin | `/api/admin` | **1** |
| 9 | Root | `/` | **2** |
| | | **TOTAL** | **48** |

Breakdown by HTTP verb:

```
GET     18
POST    13
PUT     14
DELETE   3
────────────
Total   48
```

---

## 🏗️ Architecture at a Glance

```
┌──────────────────────────────────────────────────────────────────────┐
│                          BROWSER (Port 5173)                         │
│                                                                      │
│   React 19 + Vite 7 + React Router 7 + Tailwind 4                   │
│   ├── Landing page (public)                                         │
│   ├── Login / Forgot / Reset password (public)                      │
│   └── 5 role-gated dashboard areas:                                 │
│         /admin        → ADMIN, SUPER_ADMIN                          │
│         /manager      → PROJECT_MANAGER                             │
│         /team-member  → TEAM_MEMBER                                 │
│         /qa-tester    → QA_TESTER                                   │
│         /dashboard    → any authenticated user                      │
│                                                                      │
│   Services layer ──► apiClient.js (fetch + auto token refresh)      │
│   Data layer    ──► @tanstack/react-query (caching)                 │
│   State         ──► AuthContext (user, role, permissions)           │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  HTTP  +  cookies (credentials: include)
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    EXPRESS 5 API (Port 5000)                         │
│                                                                      │
│   app.js — global middleware chain                                  │
│     helmet → CORS allowlist → cookieParser → json(10mb) → /uploads  │
│                                                                      │
│   routes/ (8 files, 46 endpoints)                                    │
│     ↓  authenticate  (verify JWT → load user → resolve permissions)  │
│     ↓  authorize()        ← ROLE gate                               │
│     ↓  requirePermission() ← PERMISSION gate                         │
│     ↓  validation schema → validateRequest                          │
│   controllers/ — business logic                                      │
│   utils/ — jwt, cookies, hashing, attachments, notifications, email │
│   Prisma ORM                                                         │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  SQL over TCP
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   POSTGRES 15 (Port 5432)                             │
│   User · Team · Project · Task · Comment · Notification              │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🚀 How To Run It

### Prerequisites

- Node.js **18+** (frontend Docker build uses Node 20)
- PostgreSQL **15** (or just let Docker handle it)
- Optional: Docker + Docker Compose

---

### Option A — The One-Command Way (Docker) ⭐

The project ships a `docker-compose.yml` that wires up all three services
and — nicely — **waits for Postgres to be healthy before starting the backend**.

```bash
# 1. Make sure backend/.env exists (it is gitignored)
#    It must point at host `db` (the compose service name), e.g.:
#    DATABASE_URL="postgresql://llyaz:1336@db:5432/internship_project?schema=public"
#    DIRECT_URL="postgresql://llyaz:1336@db:5432/internship_project?schema=public"
#    JWT_SECRET="something-long-and-random"
#    REFRESH_TOKEN_SECRET="another-long-random-string"

# 2. Fire it up
docker compose up --build

# 3. Open
#    Frontend → http://localhost:3000
#    Backend  → http://localhost:5000
```

What happens under the hood:

1. `db` starts Postgres and runs a `pg_isready` healthcheck every 5s
2. `backend` waits for that healthcheck to pass, then runs
   `npx prisma db push` (syncs the Prisma schema into the empty DB) followed by
   `npm run dev` (nodemon)
3. `frontend` runs `vite build` in a Node 20 stage, then serves the static
   `dist/` through nginx on port 80 → exposed as **3000**

Tear down (keeping data):

```bash
docker compose down
```

Nuke everything *including* the database volume:

```bash
docker compose down -v
```

---

### Option B — Split Terminals (Best for Development) ⭐⭐

This is what you'll use 95% of the time — you get hot reload on both sides.

**Terminal 1 — Postgres (only if you don't have one locally)**

```bash
docker run --name taskflow-db \
  -e POSTGRES_USER=llyaz \
  -e POSTGRES_PASSWORD=1336 \
  -e POSTGRES_DB=internship_project \
  -p 5432:5432 \
  -d postgres:15-alpine
```

**Terminal 2 — Backend**

```bash
cd backend

npm install

# ⚠️ IMPORTANT: your .env currently points at host `db` (the Docker service
# name). Running outside Docker means you must change it to `localhost`.
# backend/.env:
#   DATABASE_URL="postgresql://llyaz:1336@localhost:5432/internship_project?schema=public"
#   DIRECT_URL="postgresql://llyaz:1336@localhost:5432/internship_project?schema=public"

# Create the tables from prisma/schema.prisma
npx prisma db push

npm run dev        # → http://localhost:5000
```

> `npm run dev` has a `predev` hook that automatically runs
> `prisma generate` first, so the Prisma client is always fresh.

**Terminal 3 — Frontend**

```bash
cd frontend
npm install
npm run dev        # → http://localhost:5173
```

> No frontend `.env` is required. `apiClient.js` defaults
> `VITE_API_URL` to `http://localhost:5000/api`. Create
> `frontend/.env` only if you want to point somewhere else:
> ```
> VITE_API_URL=http://localhost:5000/api
> ```

> There is **no Vite dev proxy**, so cross-origin requests are real. The
> backend's CORS allowlist (`backend/src/config/env.js`) already includes
> `http://localhost:5173`, which is why it just works.

---

### Option C — Production Build

```bash
# Frontend
cd frontend && npm run build      # → frontend/dist  (static files)
npm run preview                   # → local preview server

# Backend
cd backend && npm run start       # node src/server.js  (no hot reload)
```

Deploy configs are included: `frontend/vercel.json`, `backend/vercel.json`, and
`backend/src/server.js` detects `process.env.VERCEL` and skips `app.listen()`
so Vercel's serverless runtime can import the app instead.

---

### Creating Your First User

⚠️ **There is no public signup.** `POST /api/auth/register` intentionally
returns `403 "Self-signup is disabled. Contact an admin to create your account."`

The repo ships two seed payloads at the root:

`admin.json`
```json
{ "name": "Admin User", "email": "admin@taskflow.com",
  "password": "admin123", "role": "ADMIN" }
```

`pm.json`
```json
{ "name": "Project Manager", "email": "pm@taskflow.com",
  "password": "pm123", "role": "PROJECT_MANAGER" }
```

Bootstrap an admin with either:

**A) SQL seed (simplest)**

```bash
docker exec -it <postgres-container> psql -U llyaz -d internship_project
```
```sql
INSERT INTO "User" (name, email, password, role)
VALUES ('Super Admin', 'root@taskflow.com',
        '$2b$10$YourBcryptHashHere', 'SUPER_ADMIN');
```
Generate the bcrypt hash with Node:
```bash
node -e "import('bcryptjs').then(async b=>console.log(await b.default.hash('admin123',10)))"
```

**B) Use the API (needs an existing admin)**

```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -b cookies.txt -c cookies.txt \
  -d '{"name":"QA Tester","email":"qa@taskflow.com","password":"qa12345","role":"QA_TESTER"}'
```
…after logging in as an admin:
```bash
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" -c cookies.txt \
  -d '{"email":"admin@taskflow.com","password":"admin123"}'
```

---

### Environment Variables (Backend)

| Variable | Required | Purpose | Notes |
|---|---|---|---|
| `JWT_SECRET` | ✅ **yes** | Signs access + reset tokens | Server refuses to boot without it |
| `REFRESH_TOKEN_SECRET` | ✅ **yes** | Signs refresh tokens | Falls back to `JWT_SECRET` if omitted |
| `DATABASE_URL` | ✅ | Pooled Prisma connection | |
| `DIRECT_URL` | ✅ | Direct connection for migrations | |
| `PORT` | ➖ | Defaults to `5000` | |
| `NODE_ENV` | ➖ | `development` \| `production` | Controls cookie `secure` flag |
| `ALLOWED_ORIGINS` | ➖ | Comma-separated CORS allowlist | Defaults to localhost:5173 + the Vercel URL |
| `COOKIE_SECURE` | ➖ | Force `Secure` on cookies | Auto-true when `NODE_ENV=production` |
| `ACCESS_TOKEN_TTL` | ➖ | Defaults to `15m` | |
| `REFRESH_TOKEN_TTL` | ➖ | Defaults to `7d` | |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` | ➖ | Password-reset emails | Gmail: port 587, `SMTP_SECURE=false` |
| `FRONTEND_URL` | ➖ | Builds the reset-password link | |

---

## 🔁 How A Request Actually Flows

This is the single most important thing to understand about the backend.
Every protected request walks through the same 5-station conveyor belt,
defined in `backend/src/app.js` + `backend/src/routes/*.routes.js`:

```
  Browser
    │  fetch(..., { credentials: "include" })   ← cookies ride along
    ▼
┌─────────────────────────────────────────┐
│ 1. GLOBAL MIDDLEWARE  (app.js)           │
│    helmet        → security headers      │
│    cors          → origin allowlist       │
│    cookieParser  → req.cookies            │
│    express.json  → 10 MB body limit       │
│    /uploads      → static file serving    │
└─────────────────┬───────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│ 2. authenticate  (auth.middleware.js)    │
│    • token = cookie OR "Bearer <jwt>"    │
│    • verifyAccessToken()                 │
│    • hit DB, confirm user still exists   │
│    • getEffectivePermissions(role, perms) │
│    → req.user = { id, role, ... }       │
│    ✗ 401 "No token provided"             │
└─────────────────┬───────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│ 3. authorize(...)  ← THE ROLE GATE       │
│    normalizes "project-manager" and      │
│    "PROJECT_MANAGER" to the same thing   │
│    ✗ 403 "Forbidden: insufficient role"  │
└─────────────────┬───────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│ 4. requirePermission(...)                │
│    ← THE PERMISSION GATE                 │
│    "*" (super admin) always passes       │
│    requirePermission  → ALL of them      │
│    requireAnyPermission → AT LEAST one   │
│    ✗ 403 "Forbidden: insufficient        │
│         permission"                      │
└─────────────────┬───────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│ 5. validateRequest                       │
│    express-validator schema; collects    │
│    all errors, returns 400 with the list │
└─────────────────┬───────────────────────┘
                  ▼
          Controller → Prisma → Postgres
                  ▼
              res.json(...)
```

> **Two separate gates.** Most write endpoints need *both* a role check and a
> permission check. That's deliberate — it lets an Admin grant granular
> capabilities (e.g. give a PM `manage_projects`) without touching their role.

---

## 🔐 The Auth System (deep dive)

### Dual-token, cookie-based, auto-rotating

```
  POST /api/auth/login
        │
        ├── bcrypt.compare(password, user.password)
        ├── status === "inactive"?  → 403 "account is inactive"
        │
        ├── accessToken  = jwt.sign({ id, role }, JWT_SECRET,        { 15m })
        ├── refreshToken = jwt.sign({ id }, REFRESH_TOKEN_SECRET,     { 7d  })
        │
        ├── bcrypt.hash(refreshToken, 10) ──► user.refreshTokenHash
        ├── user.refreshTokenExpiry = now + 7 days
        │
        └── Set-Cookie: accessToken  (httpOnly, sameSite=lax, 15 min)
                       Set-Cookie: refreshToken (httpOnly, sameSite=lax, 7 days)
```

The **raw refresh token is never stored** — only its bcrypt hash. So a database
leak can't be replayed against the refresh endpoint.

### The Silent Refresh Loop ✨

This is the neatest trick in the codebase, and it lives in
`frontend/src/services/apiClient.js`:

```js
// Every 401 on a non-auth endpoint triggers ONE automatic retry:
if (response.status === 401 && !isAuthEndpoint && !options._retriedAfterRefresh) {
  await refreshSession();                              // POST /auth/refresh
  return apiClient(endpoint, { ...options, _retriedAfterRefresh: true });
}
```

And `refreshSession` is **promise-deduplicated** via a module-level
`refreshPromise` variable — so if 5 requests all 401 at the same instant, they
share a *single* refresh call, not five. That's a real production-grade detail.

### Password Reset (with a nice security touch)

```
POST /forgot-password { email }
  ├─ always returns 200 with the same message
  │   (so attackers can't enumerate which emails exist)
  ├─ token = jwt.sign({ id, email }, JWT_SECRET + user.password, { 1h })
  │        ↑ NOTE: the secret is JWT_SECRET *concatenated with the current
  │          password hash*. Change the password → every outstanding reset
  │          link is instantly invalidated. Nice.
  ├─ emails the link via nodemailer (SMTP)
  └─ in development only, ALSO returns resetToken in the response body
       → so you can test the flow without a mailbox

POST /reset-password { token, newPassword }
  ├─ jwt.decode → user lookup
  ├─ token must equal user.resetToken exactly
  ├─ expiry check
  ├─ jwt.verify with the same JWT_SECRET + password-derived secret
  ├─ new password hashed and saved
  ├─ resetToken, refreshTokenHash, refreshTokenExpiry all nulled  ← kills sessions
  └─ cookies cleared
```

### Logout

Clears the DB refresh-token record (so the token is dead even if it leaked)
then clears both cookies. Returns `204 No Content`.

---

## 👥 Roles & The Permission System

### 5 Roles

| Role | Dashboard | What they own |
|---|---|---|
| `SUPER_ADMIN` | `/admin` | Everything. Wildcard `"*"` permission. |
| `ADMIN` | `/admin` | Users, projects, reports, settings |
| `PROJECT_MANAGER` | `/manager` | Teams, tasks, projects, reports |
| `TEAM_MEMBER` | `/team-member` | Their own tasks; update progress |
| `QA_TESTER` | `/qa-tester` | Tests and passes/fails tasks |

> The frontend uses kebab-case (`"project-manager"`) while the DB uses
> `PROJECT_MANAGER`. `role.middleware.js` normalizes with
> `.toUpperCase().replace(/-/g, "_")`, so both work everywhere.

### 9 Permissions

```
view_dashboard   manage_users   manage_teams    manage_projects
assign_tasks     test_tasks     view_reports    manage_settings
manage_roles
```

Baseline grants (`backend/src/config/permissions.js`):

| Role | Baseline permissions |
|---|---|
| `SUPER_ADMIN` | `["*"]` |
| `ADMIN` | view_dashboard, manage_users, manage_projects, view_reports, manage_settings |
| `PROJECT_MANAGER` | view_dashboard, manage_teams, assign_tasks, view_reports **+** manage_teams, manage_projects (via `ROLE_DEFAULT_PERMISSIONS`) |
| `QA_TESTER` | view_dashboard, test_tasks |
| `TEAM_MEMBER` | view_dashboard |

### The Override Trick 🔥

A user's `permissions` JSON column doesn't *replace* their role baseline — it
**overrides** it. `backend/src/utils/permissionResolver.js` supports two
special forms:

| Stored value | Meaning |
|---|---|
| `"manage_projects"` | **grant** an extra permission the role doesn't have |
| `"!manage_projects"` | **revoke** a permission the role *does* have |

```
effective = (roleBaseline ∪ grants) − revocations
```

So the resolution chain is:

```
getEffectivePermissions(role, storedPermissions)
   │
   ├─ roleBaseline  = ROLE_PERMISSIONS[role] ∪ ROLE_DEFAULT_PERMISSIONS[role]
   ├─ grants        = stored items NOT starting with "!"
   ├─ revocations   = stored items starting with "!" (strip the "!")
   ├─ if role baseline or result contains "*" → return ["*"]
   └─ return (roleBaseline ∪ grants) − revocations
```

And `encodePermissionsForRole()` goes the other way: when an admin ticks the
permission checkboxes in the UI, it diffs the selection against the role
baseline and stores only the *deltas* as grants/revocations. That's a tidy
little piece of design.

---

## 🗄️ The Data Model

```
┌──────────────┐
│    Team      │  name, description, lead
│──────────────│
│ has_many ────┼──► users
│              └──► projects
└──────────────┘

┌──────────────┐        ┌──────────────────────────────────┐
│    User      │        │             Project               │
│──────────────│        │──────────────────────────────────│
│ id, name     │◄───────┤ managerId (rel: "ProjectManager") │
│ email (uniq) │  owns  │ name, description                 │
│ password     │        │ status  = PLANNED (default)       │
│ role         │        │ startDate, endDate                 │
│ permissions  │        │ teamId ──► Team                   │
│   (Json)     │        │ attachment* (Name/Url/MimeType)    │
│ status       │        └──────────────┬───────────────────┘
│ phone        │                       │ has_many
│ location     │                       ▼
│ skill        │        ┌──────────────────────────────────┐
│ notification │        │              Task                 │
│   Prefs(Json)│        │──────────────────────────────────│
│ teamId ──►Team│        │ title, description                │
│──────────────│        │ status = PENDING (default)        │
│ resetToken   │        │ progress, previousProgress        │
│ resetExpiry  │        │ dueDate, priority = MEDIUM        │
│ refreshHash  │        │ estimatedHours, actualHours       │
│ refreshExpiry│        │ attachment* (Name/Url/MimeType)    │
└──────┬───────┘        └───┬───────────────────────┬──────┘
       │                    │ assigneeId            │ projectId
       │  "TaskAssignee" ◄──┘                       │ (required)
       │                    │ qaTesterId            │
       │  "TaskQATester" ◄─┘                       │
       │                                            │
       │                                            ▼
       │                              ┌────────────────────────┐
       │  userId                      │        Comment         │
       └─────────────────────────────►│ content (@db.Text)     │
                                      │ taskId, userId         │
                                      └────────────────────────┘

┌──────────────────────────────────────┐
│         Notification                 │  id = cuid (string!)
│──────────────────────────────────────│  type, title, message
│ userId ──► User  (onDelete: CASCADE) │  data (Json), link
│ read = false (default)               │  3 indexes: userId,
└──────────────────────────────────────┘                     (userId,read), createdAt
```

**Relation naming trick:** a `User` can be a task's *assignee* **and** a task's
*QA tester* at the same time. Prisma handles this with named relations —
`Task.assignee User @relation("TaskAssignee")` and
`Task.qaTester User @relation("TaskQATester")`.

---

## 🔄 The Task Lifecycle (the most interesting logic)

Task status is a small state machine, and **the QA tester sits in the middle of
it**. Read `backend/src/controllers/task.controller.js` around the
`updateTaskStatus` handler.

### The states

```
PENDING ──▶ IN_PROGRESS ──▶ COMPLETED ──▶ IN_TEST ──▶ PASSED
   ▲            │                                           │
   │            │                                           │
   └────────────┘                                           │
   ▲                                                         │
   └── PENDING_RETEST ◀── FAILED ────────────────────────────┘
                              │
   BLOCKED · REVIEW           │
```

### The rules

- When a **TEAM_MEMBER** pushes progress to `100`, the backend **auto-flips**
  status to `COMPLETED` — or to `IN_TEST` if a QA tester is attached.
- A **QA_TESTER** submitting `PASSED` forces `progress = 100`.
- A **QA_TESTER** submitting `FAILED` resets progress back to
  `getRetestProgress(task)` and sets `PENDING_RETEST`.
- **Admins/supervisors are exempt from task notifications** —
  `shouldSkipTaskNotificationForRole()` silences the noisy
  `task_assigned` / `task_completed` / `comment_added` alerts for
  `ADMIN` and `SUPER_ADMIN` so their bell stays useful.

### The `previousProgress` trick

A `previousProgress Int` column exists so that a **failed** task can be shown
as rolled back, rather than snapping from 100 to 0:

```js
const getRetestProgress = (task) => {
  const previousProgress = clampProgress(task?.previousProgress ?? 0);
  if (previousProgress > 0 && previousProgress < 100) {
    return previousProgress;          // resume from where the dev actually got to
  }
  const currentProgress = clampProgress(task?.progress ?? 0);
  ...
};
```

And on any progress update:

```js
updateData.previousProgress =
  nextProgress < 100 ? nextProgress
  : currentProgress < 100 ? currentProgress
  : task.previousProgress ?? 0;
```

Neat. It means the UI can honestly answer *"how far did they actually get before
it bounced back?"*

---

## 📬 The Notification Engine

Notifications aren't a background job — they're written **inline** during the
same request that triggers them, via `createNotification()` in
`backend/src/utils/notificationHelper.js`.

### How one is born

```
POST /api/tasks  (create task)
   │
   ├─ prisma.task.create(...)
   ├─ createNotification({ userId: assignee.id, type: "task_assigned", ... })
   │      │
   │      ├─ 1. fetch recipient's notificationPrefs (Json column)
   │      ├─ 2. skip if role is ADMIN/SUPER_ADMIN and type is task-related
   │      ├─ 3. map type → pref key ("task_assigned" → "taskAssigned")
   │      ├─ 4. skip if prefs[prefKey] === false
   │      ├─ 5. skip if prefs.inAppNotifications === false
   │      └─ 6. prisma.notification.create(...)
   │
   └─ res.status(201).json({ message: "Task created", task })
```

`createBulkNotifications()` does the same filtering across many users, then a
single `createMany()` — one round-trip for the whole fan-out.

### 22 notification types → 19 preference toggles

| Category | Types |
|---|---|
| **Admin** | `user_registered`, `user_deleted`, `team_created`, `team_deleted`, `project_created`, `project_completed`, `system_alert`, `role_changed` |
| **Manager** | `task_assigned`, `task_completed`, `task_overdue`, `project_deadline`, `member_joined`, `member_removed`, `added_to_team` |
| **Team member** | `task_assigned_to_me`, `task_completed_by_me`, `task_overdue_for_me`, `deadline_approaching`, `comment_added` |

Users toggle these at `/api/notification-prefs` (stored as the
`notificationPrefs` **Json** column on `User`). `POST /api/notification-prefs/reset`
restores the defaults.

### Delivery endpoints

```
GET    /api/notifications                  list mine (newest first, "time ago" formatted)
GET    /api/notifications/unread-count     badge number, polled by the bell
PUT    /api/notifications/:id/read         mark one read
PUT    /api/notifications/read-all         mark all read
DELETE /api/notifications/:id              delete one
```

> Emails for notifications are stubbed out (there's a `// TODO: Send email if
> enabled` comment) — today it's in-app only.

---

## 📎 Attachments

Projects and tasks both accept an attachment. It's **base64 in the JSON body**,
not multipart — the frontend reads the file with `FileReader` and sends:

```json
{
  "attachment": {
    "name": "spec.pdf",
    "type": "application/pdf",
    "content": "data:application/pdf;base64,JVBERi0xLj..."
  }
}
```

`backend/src/utils/attachment.js` then:

| Protection | Detail |
|---|---|
| MIME allowlist | PDF, JPEG, PNG, TXT, DOCX, XLSX — anything else is rejected |
| 5 MB cap | Enforced on the decoded buffer, not the base64 string |
| Filename sanitizing | Non-`[a-zA-Z0-9._-]` → `_`, truncated to 80 chars |
| Collision-proof naming | `${Date.now()}-${uuid}-${safeName}${ext}` |
| Write flag | `fs.writeFile(..., { flag: "wx" })` — fails rather than overwriting |
| Path traversal guard | `resolveAttachmentPath()` rejects any resolved path outside `uploads/` |
| Extension consistency | Filename extension is cross-checked against the MIME type |

Stored in `backend/uploads/<scope>/` and served statically at
`http://localhost:5000/uploads/...` via `express.static`.

---

## 🌐 Frontend Deep Dive

### Stack

React 19 · Vite 7 · React Router 7 (data router) · Tailwind CSS 4 ·
TanStack Query 5 · Axios (declared) · Recharts · Framer Motion (`motion`) ·
Lucide icons · jsPDF + jspdf-autotable (PDF reports) · SheetJS/xlsx (Excel export)

> React Compiler is enabled via `babel-plugin-react-compiler` in
> `vite.config.js`.

### Folder conventions

```
src/
├── services/          ← ALL HTTP calls live here. One file per domain.
│     apiClient.js         fetch wrapper: cookie creds, auto-refresh, friendly errors
│     authService.js       login, logout, fetchCurrentUser, persistCurrentUser
│     usersService.js      7 user calls
│     teamsService.js      6 team calls
│     projectsService.js   project CRUD + members
│     tasksService.js      task CRUD, status, assign, comments + normalization
│     notificationService.js
│
├── context/AuthContext.jsx      ← global user + all the hasRole/hasPermission helpers
├── auth/                       ← Login, ForgetPassword, ResetPassword, ProtectedRoute
├── config/permissions.js       ← mirrors the backend's permission constants
├── AppGuards.jsx               ← AdminAreaGuard, ManagerAreaGuard, QAAreaGuard,
│                                  ProjectManagementGuard, ProjectWorkspaceGuard
├── loader/                     ← React Router data loaders (prefetch before render)
│     admin/  manager/
│     features/                 ← feature slices (projects, tasks, teams, users,
│                                  settings, notifications, dashboard)
│                                 each with its own `*Access.js` permission map
├── pages/                      ← grouped by role: admin/ projectManager/
│                                 qaTester/ teamMember/ dashboard/
├── Component/                  ← layouts, sidebars, notification bells, landing/
└── services/ + App.jsx         ← the route tree
```

### `apiClient.js` — the heart of the frontend

Handles four things so no other file has to:

1. **Cookie credentials** — `credentials: "include"` on every request.
2. **Cache busting** — GETs get `&_=${Date.now()}` appended, plus
   `cache: "no-store"`. (Aggressive, but keeps dashboards honest.)
3. **The silent-refresh retry** described earlier.
4. **Human-readable errors** — `getFriendlyErrorMessage()` turns raw backend
   text into UX copy:
   ```
   401 on /auth/login         → "Email or password is incorrect."
   401 elsewhere              → "Your session has expired. Please log in again."
   403                        → "You do not have permission to perform this action."
   5xx                        → "Something went wrong on our side. Please try again..."
   ```
   `error.rawMessage` is still attached for debugging.

### Two layers of route protection

```
<ProtectedRoute allowedRoles={[...]} requiredPermissions={[...]}>
```
1. **Server-side** — `authenticate` → `authorize` → `requirePermission`
2. **Client-side** — `ProtectedRoute` + the `*Access.js` files hide nav items and
   buttons the user can't use

Both are needed: the client layer is UX, the server layer is security.

### Route areas

| Path | Guard | Notes |
|---|---|---|
| `/` | public | Lazy-loaded landing page |
| `/login`, `/forgot-password`, `/reset-password` | public | `/signup` redirects to `/login` |
| `/admin/*` | `AdminAreaGuard` | dashboard, users, teams, projects, reports, settings, notifications |
| `/manager/*` | `ManagerAreaGuard` | dashboard, teams, users, projects, tasks, progress, reports, settings, notifications |
| `/team-member/*` | `ProtectedRoute` | dashboard, tasks, progress, reports, profile, notifications |
| `/qa-tester/*` | `QAAreaGuard` | dashboard, tasks, projects, profile, notifications |
| `/dashboard` | `ProtectedRoute` | generic dashboard for any authed user |
| `*` | — | redirects to `/login` |

### Task normalization (frontend)

The backend speaks `UPPER_SNAKE` (`IN_PROGRESS`); the UI speaks `kebab-case`
(`in-progress`). `tasksService.js` bridges them with bidirectional maps, and
`normalizeTask()` also clamps progress to `0–100` and forces `100` for any
terminal status. So the rest of the UI never has to think about it.

---

## 📡 The Complete Endpoint Reference

Base URL: `http://localhost:5000`
Auth: `accessToken` + `refreshToken` httpOnly cookies (or `Authorization: Bearer <accessToken>`)

Legend — 🔒 auth · 🛡 role gate · 🔑 permission gate

---

### 1. Auth — `/api/auth` · 6 endpoints

| # | Method | Path | What it does |
|---|---|---|---|
| 1 | `POST` | `/api/auth/register` | 🔒 Always `403`. Self-signup is disabled by design. |
| 2 | `POST` | `/api/auth/login` | Validates credentials → sets both cookies → returns `{ user }`. |
| 3 | `POST` | `/api/auth/refresh` | Rotates tokens from the refresh cookie. |
| 4 | `POST` | `/api/auth/logout` | Kills the stored session + clears cookies. `204`. |
| 5 | `POST` | `/api/auth/forgot-password` | Emails a reset link. Always `200` (no enumeration). Returns `resetToken` in dev. |
| 6 | `POST` | `/api/auth/reset-password` | Consumes a reset token, sets a new password, kills all sessions. |

---

### 2. Users — `/api/users` · 7 endpoints

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 7 | `GET` | `/api/users` | 🔒 🛡 admin·super 🛡(users ∨ teams) | List users (passwords stripped) |
| 8 | `POST` | `/api/users` | 🔒 🛡 admin·super 🔑 `manage_users` | Create a user (admin-created accounts) |
| 9 | `PUT` | `/api/users/:id/role` | 🔒 🛡 admin·super 🔑(users ∨ roles) | Change role **and** rewrite permissions as delta overrides |
| 10 | `GET` | `/api/users/me` | 🔒 | The caller's own profile |
| 11 | `PUT` | `/api/users/me/profile` | 🔒 | Update own name / phone / location / skill |
| 12 | `PUT` | `/api/users/me/password` | 🔒 | Change own password (verifies current) |
| 13 | `DELETE` | `/api/users/:id` | 🔒 🛡 admin·super 🔑 `manage_users` | Delete a user |

---

### 3. Teams — `/api/teams` · 6 endpoints

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 14 | `GET` | `/api/teams` | 🔒 🛡(teams ∨ users) | List teams with member counts |
| 15 | `GET` | `/api/teams/:teamId` | 🔒 🛡(teams ∨ users) | One team + its members |
| 16 | `POST` | `/api/teams` | 🔒 🔑 `manage_teams` | Create team (optionally notify admins) |
| 17 | `DELETE` | `/api/teams/:teamId` | 🔒 🔑 `manage_teams` | Delete team + notify affected members |
| 18 | `PUT` | `/api/teams/:teamId/add-member` | 🔒 🔑 `manage_teams` | Add a user; fires `added_to_team` notifications |
| 19 | `PUT` | `/api/teams/:teamId/remove-member` | 🔒 🔑 `manage_teams` | Remove a user; fires `member_removed` notifications |

---

### 4. Projects — `/api/projects` · 6 endpoints

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 20 | `POST` | `/api/projects` | 🔒 🛡 admin·super·PM 🔑 `manage_projects` | Create project (+ optional attachment) |
| 21 | `GET` | `/api/projects` | 🔒 🛡 admin·super·PM | List projects with team + manager + task stats |
| 22 | `GET` | `/api/projects/:id` | 🔒 🛡 admin·super·PM·member | Single project detail |
| 23 | `PUT` | `/api/projects/:id` | 🔒 🛡 admin·super·PM 🔑 `manage_projects` | Update project / replace attachment |
| 24 | `DELETE` | `/api/projects/:id` | 🔒 🛡 admin·super·PM 🔑 `manage_projects` | Delete project + cascade its tasks |
| 25 | `GET` | `/api/projects/:projectId/members` | 🔒 🛡 admin·super·PM·member | Project roster |

---

### 5. Tasks — `/api/tasks` · 12 endpoints ⭐ (the biggest group)

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 26 | `POST` | `/api/tasks` | 🔒 🛡 admin·super·PM 🔑 `assign_tasks` | Create task (+ assignee, QA tester, attachment) |
| 27 | `GET` | `/api/tasks` | 🔒 🛡 admin·super·PM | All tasks |
| 28 | `GET` | `/api/tasks/my-tasks` | 🔒 🛡 member·QA | Tasks assigned to the caller |
| 29 | `GET` | `/api/tasks/project/:projectId` | 🔒 🛡 admin·super·PM·member·QA | Tasks in a project |
| 30 | `GET` | `/api/tasks/:id` | 🔒 🛡 all roles | One task, with normalized progress |
| 31 | `PUT` | `/api/tasks/:id` | 🔒 🛡 admin·super·PM 🔑 `assign_tasks` | Edit task details |
| 32 | `PUT` | `/api/tasks/:id/status` | 🔒 🛡 admin·super·PM·member·QA | **Status + progress.** The QA state machine lives here. |
| 33 | `PUT` | `/api/tasks/:id/assign` | 🔒 🛡 admin·super·PM 🔑 `assign_tasks` | Reassign the assignee / QA tester |
| 34 | `DELETE` | `/api/tasks/:id` | 🔒 🛡 admin·super·PM 🔑 `assign_tasks` | Delete task + comments + attachment |
| 35 | `GET` | `/api/tasks/:id/comments` | 🔒 🛡 all roles | Comments (newest first, with author info) |
| 36 | `POST` | `/api/tasks/:id/comments` | 🔒 🛡 all roles | Add a comment (fires `comment_added`) |
| 37 | `DELETE` | `/api/tasks/:id/comments/:commentId` | 🔒 🛡 all roles | Delete a comment |

---

### 6. Notifications — `/api/notifications` · 5 endpoints

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 38 | `GET` | `/api/notifications` | 🔒 | My notifications, with `"2h ago"` style timestamps |
| 39 | `GET` | `/api/notifications/unread-count` | 🔒 | Unread badge count |
| 40 | `PUT` | `/api/notifications/:id/read` | 🔒 | Mark one as read |
| 41 | `PUT` | `/api/notifications/read-all` | 🔒 | Mark all as read |
| 42 | `DELETE` | `/api/notifications/:id` | 🔒 | Delete one |

---

### 7. Notification Prefs — `/api/notification-prefs` · 3 endpoints

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 43 | `GET` | `/api/notification-prefs` | 🔒 | Read my prefs (defaults filled in) |
| 44 | `PUT` | `/api/notification-prefs` | 🔒 | Toggle any of the ~19 in-app switches |
| 45 | `POST` | `/api/notification-prefs/reset` | 🔒 | Back to defaults |

---

### 8. Admin — `/api/admin` · 1 endpoint

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 46 | `GET` | `/api/admin/dashboard` | 🔒 🛡 admin·super | `{ totalUsers, totalTeams, totalProjects, totalTasks, completedTasks, inProgressTasks }` |

---

### 9. Root — 2 endpoints

| # | Method | Path | Guard | What it does |
|---|---|---|---|---|
| 47 | `GET` | `/` | public | `{ message: "Task Management API running" }` |
| 48 | `GET` | `/_test-email?to=` | dev only (`404` in production) | Sends a test password-reset email via SMTP |

Plus the static mount `GET /uploads/<scope>/<file>` — serves attachments.

---

### Quick smoke test

```bash
# 1. Is it alive?
curl http://localhost:5000/
# → {"message":"Task Management API running"}

# 2. Log in and stash the cookies
curl -s -c jar.txt -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@taskflow.com","password":"admin123"}' | jq

# 3. Use them
curl -s -b jar.txt http://localhost:5000/api/users/me | jq
curl -s -b jar.txt http://localhost:5000/api/admin/dashboard | jq

# 4. Watch a refresh happen automatically
curl -s -i -b jar.txt http://localhost:5000/api/users/me | grep -i set-cookie
```

---

## 📁 Project Layout

```
Project-Management/
├── docker-compose.yml          3 services: db, backend, frontend
├── admin.json                  seed payload for an admin
├── pm.json                     seed payload for a project manager
├── docs/
│   └── doc.md                  ← you are here
│
├── backend/                    Express 5 + Prisma + Postgres
│   ├── .env                    (gitignored) DB + JWT + SMTP config
│   ├── Dockerfile              node:18-alpine + openssl for Prisma
│   ├── vercel.json             serverless deploy
│   ├── prisma/
│   │   ├── schema.prisma       7 models
│   │   └── migrations/         16 migrations (init → refresh tokens)
│   └── src/
│       ├── server.js           boots app, asserts env vars
│       ├── app.js              global middleware + route mounting
│       ├── config/             db.js, env.js, permissions.js
│       ├── middleware/         auth, role, permission, validate
│       ├── routes/             8 files → 46 endpoints
│       ├── controllers/        8 files, 2,751 lines
│       ├── validation/         express-validator schemas per domain
│       └── utils/              jwt, hash, authCookies, attachment,
│                               email, notificationHelper, permissionResolver
│
└── frontend/                   React 19 + Vite 7 + Tailwind 4
    ├── Dockerfile              build → nginx:alpine
    ├── vercel.json
    └── src/
        ├── main.jsx  App.jsx  (the 531-line route tree)
        ├── AppGuards.jsx  ProtectedRoute.jsx
        ├── context/AuthContext.jsx
        ├── services/           apiClient + 6 domain services
        ├── loader/             15 data loaders
        ├── features/           7 feature slices w/ permission maps
        ├── pages/              role-grouped screens
        └── Component/          layouts, sidebars, bells, landing
```

---

## ⚠️ Gotchas & Known Rough Edges

Things I noticed while reading the code — useful to know before you debug:

1. **`.env` host only works in Docker.** `backend/.env` points at `@db:5432`,
   which is the compose service name. Running `npm run dev` outside Docker
   requires changing it to `localhost`.

2. **`/_test-email` is dev-only** but the check is `NODE_ENV === "production"`.
   Leave `NODE_ENV` unset locally and it stays reachable.

3. **The admin dashboard task counts are always zero.** Tasks are created with
   `status: "PENDING"` / `"COMPLETED"` (uppercase), but
   `admin.controller.js` counts `where: { status: "completed" }` and
   `"in-progress"` (lowercase). Postgres is case-sensitive, so `completedTasks`
   and `inProgressTasks` never match anything. Should be `"COMPLETED"` /
   `"IN_PROGRESS"`.

4. **Two frontend calls hit nonexistent backend routes** and will `404`:
   - `PUT /tasks/:id/progress` (`tasksService.js:233` → `updateTaskProgress`)
     — the backend only has `PUT /tasks/:id/status`
   - `POST /projects/:projectId/members` and
     `DELETE /projects/:projectId/members/:userId`
     (`projectsService.js` → `addMemberToProject` / `removeMemberFromProject`)
     — the backend only has `GET /projects/:id/members`

5. **No multer.** Attachments travel as base64 in the JSON body, so the 10 MB
   `express.json` limit is really ~7.5 MB of raw file after base64 inflation
   (and the app-level cap is 5 MB).

6. **`GET /api/tasks` and `GET /api/projects` don't scope by user.** A PM
   requesting them sees everything the PM can see. Row-level filtering for
   team members happens via `/tasks/my-tasks` instead.

7. **`register` is a 403 stub.** That's intentional (commit
   *"I removed public signup, admin can add new user"*), but the route still
   exists, which is confusing on first read.

8. **`swagger-ui-express` and `yamljs` are installed but unused.** No OpenAPI
   spec is wired up in `app.js` — this doc is the substitute.

9. **15-minute access tokens** are aggressive for a long session, but the
   silent-refresh loop makes it a non-issue in practice.

10. **`prisma generate` runs twice in Docker** — once via the explicit
    `npx prisma generate` in the Dockerfile (needed before `COPY . .` so the
    binaries exist) and again via `npm run dev`'s `predev` hook.

---

## 🧭 Cheat Sheet

```
BACKEND  http://localhost:5000
FRONTEND http://localhost:5173      (Docker: http://localhost:3000)
DB       localhost:5432/internship_project

npm run dev            backend (nodemon)
npm run start          backend (no reload)
npx prisma db push     sync schema → DB
npx prisma studio      browse data in a GUI  ← very handy
npx prisma migrate dev name   create a migration

npm run dev            frontend (vite)
npm run build          frontend → dist/
npm run lint           frontend eslint

docker compose up --build      everything
docker compose down -v        everything + wipe DB

20%  SETUP     30%  AUTH & PERMISSIONS   30%  ROUTING & GUARDS
10%  PRISMA    10%  NOTIFICATIONS & ATTACHMENTS
```

**Files to read first, in order:**

1. `backend/src/app.js` — the whole shape of the API in 72 lines
2. `backend/src/middleware/auth.middleware.js` — how identity is resolved
3. `backend/src/utils/permissionResolver.js` — the grant/revoke override logic
4. `backend/src/routes/task.routes.js` — the most permission-dense router
5. `frontend/src/services/apiClient.js` — the silent-refresh loop
6. `frontend/src/App.jsx` — the route tree and guard composition
7. `backend/prisma/schema.prisma` — the data model

---

*Docs generated from source at commit `742f4c4` (branch `main`).*
