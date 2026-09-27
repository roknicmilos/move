# Move MVP: Implementation Plan

Companion to [specs-mvp.md](specs-mvp.md). Part 1 reviews the spec and proposes design changes. Part 2 fixes the
architecture. Part 3 splits the work into steps, each ending in a feature you can try in the running app.

## Part 1: Spec review and proposed changes

### 1.1 Entities: before and after

**Workout**

| Spec                   | Proposed                   | Why                                                                                               |
|------------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| name, short_name       | keep                       | Both are useful: full name on detail pages, short name on cards and history.                      |
| instructions, purpose  | replace with `description` | `seeds/workouts.yaml` only has `description`. The instructions and purpose live on each exercise. |
| exercises (collection) | reverse FK `exercises`     | Comes from `Exercise.workout`, ordered by `Exercise.position`.                                    |

**Exercise**

| Spec                              | Proposed                                     | Why                                                                                                                                                                                                                                                                                                                                |
|-----------------------------------|----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| workout (FK)                      | keep                                         | No exercise is shared between the two workouts, so an M2M join table isn't needed yet. If exercises become reusable later, it can be introduced then.                                                                                                                                                                              |
| —                                 | **add `position`** (int, unique per workout) | The seed order is the flow order. Relying on `id` for ordering is fragile. The seed loader sets it from the exercise's position in the YAML list.                                                                                                                                                                                  |
| description                       | **replace with `instructions` + `purpose`**  | This matches `seeds/exercises.yaml`: instructions say how to do it, purpose says why.                                                                                                                                                                                                                                              |
| sets, reps, duration_sec, load_kg | keep (all optional)                          | `duration_sec` and `load_kg` are **per set**. Document this.                                                                                                                                                                                                                                                                       |
| —                                 | **add `per_side`** (bool, default false)     | Many exercises are done once per side (Fire Hydrants, Side Bend, 90/90, ...). The UI can then show "2 × 10 / side". Seed candidates to confirm: Reverse Step Up, Split Squats, External Rotation, Hip Flexor Kick Out, Side Bend, Couch Stretch, Fire Hydrants, Hip Abduction, 90/90 Rotation, Outer Hip Dropset, Pigeon Strength. |

**WorkoutSession**: the name is good, keep it.

| Spec        | Proposed                      | Why                                                                                            |
|-------------|-------------------------------|------------------------------------------------------------------------------------------------|
| —           | **add `user`** (FK, required) | There are multiple predefined users, so each session must belong to one.                       |
| workout     | keep (FK, `PROTECT`)          | A workout that has history can't be deleted.                                                   |
| started_at  | keep, set by the server       | Set to the current time on create.                                                             |
| ended_at    | keep, **nullable**            | `NULL` means "in progress". Setting it is how a session ends.                                  |
| constraints | **add**                       | At most one in-progress session per user (partial unique index), and `ended_at >= started_at`. |

**ExerciseSet: rename to `SessionExercise`**

"Set" is misleading. A row stands for one exercise done in one session, not one set (an exercise has many sets).

| Spec                  | Proposed                                         | Why                                                                                                                                                                                                         |
|-----------------------|--------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| —                     | **add `session`** (FK, related name `exercises`) | The spec had the collection on the session but no FK back to it.                                                                                                                                            |
| exercise              | keep (FK, `PROTECT`)                             |                                                                                                                                                                                                             |
| —                     | **add `completed_at`** (datetime)                | This is the checkmark. The row is created when the exercise is checked and deleted when it is unchecked.                                                                                                    |
| load_kg, duration_sec | keep, **add `sets`, `reps`** (all optional)      | These are the values actually done. They are copied from the exercise prescription when checked, so history stays correct if the prescription changes later. Editing them in the UI can come after the MVP. |
| —                     | unique `(session, exercise)`                     |                                                                                                                                                                                                             |

**User**: custom `accounts.User`, extending `AbstractUser` (the model Django's docs recommend starting from when the
built-in one needs any change, since swapping it later requires a fresh database). `username` stays the login field
(`USERNAME_FIELD`, unique, required); `email` becomes optional (`blank=True`). `AUTH_USER_MODEL = "accounts.User"` is
set in Step 1, before the first migration, so no app migrates against `auth.User` first. There is no registration.
Users are created with `manage.py createsuperuser` (the owner) and through Django admin (everyone else). Credentials
are never committed.

### 1.2 Other decisions

- **Seeds are the source of truth for workouts and exercises.** `manage.py load_seeds` upserts by `id` and is safe
  to run more than once. It runs on every backend start. Removing an entry from YAML does **not** delete it from the
  DB, because history references it.
- **Session expiry (2 weeks):** a Django session cookie with `SESSION_COOKIE_AGE = 1209600` (14 days), counted from
  login rather than extended on each request (`SESSION_SAVE_EVERY_REQUEST = False`). Fixed expiry is the simpler
  reading of "expire after 2 weeks". Switching to sliding expiry is a one-line change.
- **Session cookie auth, not JWT.** The frontend and API are served from the same origin, so there is no CORS setup
  and no tokens in JS storage. CSRF uses Django's cookie plus an `X-CSRFToken` header.
- **Time:** store UTC (`USE_TZ = True`) and show times in the browser's local time zone. History weeks start on
  **Monday**, and week and month ranges are computed in local time.
- **Terminology:** "session" means `WorkoutSession` in UI and API paths. The auth cookie is always called
  "login session" to avoid confusion.
- **Spec file:** Step 1 updates the Entities and Architecture sections of `specs-mvp.md` to match this plan.

### 1.3 Out of scope for the MVP (candidates for later)

Offline recording with background sync, editing values actually done per exercise, Activities and Garmin/Strava/
Google Fit sync (see `specs-draft.md`), a UI for editing workouts (use admin for now), and push notifications.

## Part 2: Architecture

### 2.1 Stack

| Layer    | Choice                                                                                                                  |
|----------|-------------------------------------------------------------------------------------------------------------------------|
| Backend  | Python, Django, Django REST Framework, drf-spectacular (OpenAPI and Swagger UI), gunicorn (prod), `uv` for dependencies |
| Database | PostgreSQL                                                                                                              |
| Frontend | React, TypeScript, Vite, React Router, TanStack Query, Tailwind CSS, date-fns, `vite-plugin-pwa`, npm                   |
| Proxy    | Caddy: one origin for SPA, `/api`, `/admin`, `/static`. Automatic HTTPS in prod                                         |
| Tests    | pytest + pytest-django (API), Vitest (date/grouping utils), ruff, ESLint, `tsc --noEmit`                                |

Pin the current stable version of each (Python, Django, PostgreSQL, Node LTS) when Step 1 is implemented.

### 2.2 Services (Docker Compose)

Everything runs in containers. Nothing needs to be installed on the host except Docker and Git.

```
                 ┌──────────── proxy (Caddy) :8080 dev / :80,:443 prod ────────────┐
 browser ──────▶ │  /api/*, /admin/*, /static/*  → backend:8000                     │
                 │  /*                           → frontend:5173 (dev, Vite HMR)    │
                 │                                 or built SPA files (prod)        │
                 └──────────────────────────────────────────────────────────────────┘
 backend (Django) ──▶ db (PostgreSQL, named volume `pgdata`)
```

- `compose.yaml` (dev): `db`, `backend` (runserver, source bind-mounted), `frontend` (Vite dev server, source
  bind-mounted, `node_modules` in a volume), `proxy`. Only the proxy port 8080 is published.
- `compose.prod.yaml` (Step 7): `db`, `backend` (gunicorn), `proxy` (Caddy image with the built SPA and Django
  static files copied in, `{$DOMAIN}` with Let's Encrypt).
- Backend entrypoint: `migrate` → `load_seeds` → start the server.
- Config comes from `.env` (gitignored). `.env.example` is committed.

### 2.3 Repository layout (target)

```
backend/            Django project `config/`; apps `accounts/` (custom User model, auth API) and `workouts/`
                    (Workout, Exercise, WorkoutSession, SessionExercise, load_seeds command, API); Dockerfile,
                    pyproject.toml, uv.lock
frontend/           Vite React app: src/{api,components,pages,hooks,lib}; Dockerfile; package.json
proxy/              Caddyfile (dev), Caddyfile.prod, Dockerfile (prod)
seeds/              workouts.yaml, exercises.yaml (unchanged location; backend build context is repo root)
specs/              specs-draft.md, specs-mvp.md, specs-mvp.plan.md
compose.yaml, compose.prod.yaml, .env.example, README.md, .claude/CLAUDE.md
```

### 2.4 API (all under `/api/`, JSON in snake_case, login required except where noted)

| Step | Method & path                                     | Purpose                                                                                                |
|------|---------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| 1    | `GET /health/` (public)                           | Liveness check, including the DB                                                                       |
| 1    | `GET /workouts/` (public)                         | List: id, name, short_name, description, exercise_count                                                |
| 1    | `GET /schema/`, `GET /docs/` (DEBUG only)         | OpenAPI schema and Swagger UI                                                                          |
| 2    | `GET /auth/csrf/` (public)                        | Sets the CSRF cookie                                                                                   |
| 2    | `POST /auth/login/` (public, throttled)           | Username and password → login session cookie                                                           |
| 2    | `POST /auth/logout/`                              | Ends the login session                                                                                 |
| 2    | `GET /auth/me/`                                   | The current user, or 403                                                                               |
| 3    | `GET /workouts/{id}/` (public)                    | Workout with its ordered exercises                                                                     |
| 3    | `GET /exercises/{id}/` (public)                   | Exercise details                                                                                       |
| 4    | `POST /sessions/` `{workout_id}`                  | Start a session (`started_at = now`). **409** with the active session id if one is already in progress |
| 4    | `GET /sessions/active/`                           | The user's in-progress session, or 204                                                                 |
| 4    | `GET /sessions/{id}/`                             | Session with completed exercises                                                                       |
| 4    | `PUT /sessions/{id}/exercises/{exercise_id}/`     | Check an exercise. Idempotent, copies the prescription                                                 |
| 4    | `DELETE /sessions/{id}/exercises/{exercise_id}/`  | Uncheck an exercise                                                                                    |
| 4    | `POST /sessions/{id}/end/`                        | Set `ended_at = now`                                                                                   |
| 4    | `DELETE /sessions/{id}/`                          | Discard a session started by mistake                                                                   |
| 5    | `GET /sessions/?started_after=…&started_before=…` | History for the current user, newest first                                                             |

All session endpoints only return the current user's sessions. Another user's session returns 404. Checking or
unchecking an exercise is rejected on an ended session, and so is an exercise that belongs to another workout.

`/workouts/`, `/workouts/{id}/`, and `/exercises/{id}/` stay public permanently (not just until Step 2 lands), so
anyone can browse the workouts and exercises without logging in. Everything under `/sessions/` still requires login.

### 2.5 Frontend routes

| Route                                        | Page                                                                                |
|----------------------------------------------|-------------------------------------------------------------------------------------|
| `/`                                          | Workouts overview (landing), with a "Resume workout" banner when one is in progress |
| `/login`                                     | Login                                                                               |
| `/workouts/:workoutId`                       | Workout details, exercise list, **Start workout** button                            |
| `/workouts/:workoutId/exercises/:exerciseId` | Exercise details                                                                    |
| `/sessions/:sessionId`                       | Recording page while in progress, summary once ended                                |
| `/history?view=week\|month&date=YYYY-MM-DD`  | History (weekly by default)                                                         |

`/`, `/workouts/:workoutId`, and `/workouts/:workoutId/exercises/:exerciseId` are public (no route guard). Every
other route, and the **Start workout** action on the workout page, requires login; hitting them unauthenticated
redirects to `/login?next=…`.

The layout is mobile first, with a bottom nav: Workouts · History · (account/logout).

## Part 3: Implementation steps

Each step ends with a **"Try it"** check in the running app, passing tests, and updates to README/CLAUDE.md/spec
where relevant.

### Step 0: Prerequisites (Docker and Compose)

1. Check what is installed:
   `docker --version`, `docker compose version` (must be the Compose **v2+ plugin**, `docker compose`, not the legacy
   `docker-compose`), `docker buildx version`, `git --version`.
2. Check whether it is the latest version: `sudo apt-get update && apt-cache policy docker-ce docker-compose-plugin`.
   *Installed* must equal *Candidate*. If it doesn't, upgrade:
   `sudo apt-get install --only-upgrade docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`.
3. If Docker isn't installed at all, install it from Docker's official apt repository
   (https://docs.docker.com/engine/install/ubuntu/), not Ubuntu's `docker.io` package.
4. Make sure it runs without sudo: `sudo usermod -aG docker $USER` (then log in again), and
   `docker run --rm hello-world`.

*State on 2026-09-27 (dev laptop, Ubuntu 24.04): Docker 29.8.1 and Compose v5.5.1, which are the newest in the apt
repo.*

**Try it:** `docker run --rm hello-world` prints its greeting, and `docker compose version` works.

### Step 1: Stack skeleton and Workouts overview (landing page)

Backend

- Django project `config`, with settings read from env (`DJANGO_SECRET_KEY`, `DJANGO_DEBUG`, `DJANGO_ALLOWED_HOSTS`,
  `DJANGO_CSRF_TRUSTED_ORIGINS`, `POSTGRES_*`). Postgres via `DATABASE_*` settings. DRF and drf-spectacular set up.
- App `accounts` with the custom `User` model from 1.1 (`AUTH_USER_MODEL = "accounts.User"`), its migration, and
  admin registration — done first, before any other app's migration.
- App `workouts` with the `Workout` and `Exercise` models from 1.1, a migration, and admin registration.
- `load_seeds` management command: reads `/seeds/*.yaml`, `update_or_create` by id, sets `position` from the order
  within each workout, and validates that each FK exists. Seeds get the `per_side` flags.
- `GET /api/health/`, `GET /api/workouts/`. Both public — `workouts/` stays public permanently, per 2.4.
- `backend/Dockerfile` (uv, non-root user), entrypoint that runs migrate + load_seeds.

Frontend

- Vite + React + TS + Tailwind + React Router + TanStack Query, and a typed `api` client (fetch wrapper).
- Landing page `/`: a card for each workout (short name, name, description, exercise count).

Infra

- `compose.yaml` (db, backend, frontend, proxy), `proxy/Caddyfile`, `.env.example`, `.dockerignore`s, and
  `.gitignore` additions (`node_modules/`, `frontend/dist/`, `backend/staticfiles/`).

Docs

- Create **README.md** (see "Docs deliverables") and update **.claude/CLAUDE.md**.
- Update `specs-mvp.md`: Entities to match 1.1, Architecture to match Part 2.

Tests: `load_seeds` is idempotent (running it twice gives the same row counts and updates changed fields),
and `/api/workouts/` returns both workouts with the right exercise counts.

**Try it:** `cp .env.example .env && docker compose up --build`, then open http://localhost:8080 and see
"LBA Main" and "LBA Supporting" (11 and 9 exercises). http://localhost:8080/api/docs/ shows Swagger UI.

### Step 2: Authentication

Backend

- `accounts`: `csrf`, `login`, `logout`, `me` endpoints, on top of the `User` model from Step 1. `SessionAuthentication`
  and `IsAuthenticated` become the DRF defaults, with `AllowAny` explicitly overridden on `health`, `workouts` (list
  and detail), `exercises` (detail), `csrf`, and `login`. Login is throttled (for example 5/min per IP).
- Settings: `SESSION_COOKIE_AGE = 1209600`, `SESSION_SAVE_EVERY_REQUEST = False`, `SESSION_COOKIE_HTTPONLY = True`,
  `CSRF_COOKIE_HTTPONLY = False` (the SPA reads it), `SameSite=Lax`.
- Users: `docker compose exec backend python manage.py createsuperuser`, and other users through `/admin/`.

Frontend

- `/login` page. An auth context uses `GET /auth/me`. A route guard applies only to `/sessions/*` and `/history`,
  sending unauthenticated users to `/login?next=…`; `/`, `/workouts/:id`, and its exercise pages stay reachable
  logged out. The **Start workout** button sends a logged-out user to `/login?next=…` instead. A global handler for
  401/403 on any API call does the same redirect.
- The API client sends `X-CSRFToken` on unsafe methods.
- A logout action in the nav.

Tests: a wrong password gives 400, a correct one sets the cookie, `me` returns 200 when logged in and 403 when not,
`/api/workouts/` (list and detail) and `/api/exercises/{id}/` return 200 anonymously, `/api/sessions/*` returns 403
anonymously, and the login cookie max-age is 14 days.

**Try it:** create a user. Opening `/` anonymously shows the workouts (no redirect); opening `/history` redirects to
login. A wrong password shows an error, and the right one logs in. After logout, `/history` redirects to login
again but `/` still works. DevTools shows a `sessionid` cookie that expires in 14 days.

### Step 3: Workout and exercise details

Backend: `GET /api/workouts/{id}/` (with its exercises ordered by position) and `GET /api/exercises/{id}/`, both
public per 2.4.

Frontend

- `/workouts/:id`: name, description, and an ordered exercise list with the prescription formatted as
  "2 × 25", "2 × 60 s", "1 × 10 min", "/ side", "@ 5 kg". A disabled **Start workout** button (enabled in Step 4).
- `/workouts/:id/exercises/:exerciseId`: prescription, instructions, purpose, and prev/next exercise links.
- A shared `formatPrescription()` utility, with a Vitest test.

Tests: the API returns exercises in order, unknown ids return 404, and prescription formatting is covered.

**Try it:** landing → LBA Main → exercises listed Sled Pull … Side Bend → open Back Extensions, whose long
instructions are readable, and use next/prev to move through the flow.

### Step 4: Recording a workout session

Backend

- `WorkoutSession` and `SessionExercise` models from 1.1, with constraints (one in-progress session per user,
  `ended_at >= started_at`, unique `(session, exercise)`). A migration and admin registration.
- The session endpoints from 2.4, following those rules. Starting a second session returns 409 with the id of the
  active one.

Frontend

- Workout page: **Start workout** → `POST /sessions/` → navigate to `/sessions/:id`. On 409, offer to resume. If the
  visitor isn't logged in, it sends them to `/login?next=…` instead (the page itself stays visible logged out).
- Recording page (`/sessions/:id` while in progress): a clear "In progress" header with the workout name and a live
  elapsed timer, and the exercise list with a large checkbox per exercise. Checkboxes update optimistically and roll
  back on error. Tapping an exercise name opens its details, and Back returns to the session. There is a
  progress count ("4 / 11"), **End workout** (in-app confirm dialog, which warns if nothing is checked), and
  **Discard**.
- Landing page: a "Workout in progress → Resume" banner from `GET /sessions/active/`.
- Summary view (`/sessions/:id` once ended): date, start and end time, duration, and the completed exercises.

Tests: start, check, uncheck, end. Also covered: starting while one is active gives 409, checking after the session
ended is rejected, an exercise from the other workout is rejected, another user's session gives 404, and the
prescription snapshot is copied.

**Try it:** start LBA Supporting, check three exercises, reload the page (the state is kept), leave for the landing
page (the Resume banner shows), come back, and end the workout. The summary shows the duration and three exercises.
Try to start another while one is active, and the app offers to resume it.

### Step 5: Workout history (weekly / monthly)

Backend: `GET /api/sessions/?started_after=&started_before=`, the current user's sessions only. Each item has the
workout short name, `started_at`, `ended_at`, and the completed and total exercise counts. There is an index on
`(user, started_at)`.

Frontend

- `/history`: a **Week | Month** toggle (weekly by default) and prev/next/today navigation. The state is kept in the
  URL query, so reload and back work.
- Weekly: Mon–Sun rows, each listing that day's sessions (short name, time, duration, "9/11"). Monthly: a calendar
  grid with a marker for each session day. Tapping a day lists its sessions, and tapping a session opens its summary.
- Range and grouping helpers (local time zone, week starting Monday) in `src/lib/dates.ts`, with Vitest tests (week and
  month boundaries, a DST change, a session just before midnight).

Tests: the API filters by range and user and orders newest first.

**Try it:** record a couple of sessions (or add backdated ones in `/admin/`), open History, and see them in the week
view. Switch to Month and go to the previous month. The URL updates and reloading keeps the view.

### Step 6: Installable PWA

- `vite-plugin-pwa`: manifest (name "Move", short_name, `display: standalone`, theme and background colors,
  `start_url: /`), icons (192, 512, maskable), and an Apple touch icon.
- Service worker: precache the app shell. `/api/*` is network-only, so user data is never served stale and auth
  responses are never cached. `/admin` and `/api` are excluded from the navigation fallback. Prompt the user to
  reload when an update is available.
- Optional before Step 7: test on the phone through `ngrok http 8080` (add the ngrok host to
  `DJANGO_ALLOWED_HOSTS` and `DJANGO_CSRF_TRUSTED_ORIGINS`).

**Try it:** desktop Chrome at http://localhost:8080 shows the install icon, and Lighthouse's PWA/installability
check passes. The installed app opens in its own window.

### Step 7: Production deployment to a VPS

Prerequisites: a VPS (Ubuntu 24.04 LTS, 1 vCPU / 1 GB is enough), a domain with an A/AAAA record pointing to it,
Docker and Compose installed there following **Step 0**, and ports 80 and 443 open (ufw: 22, 80, 443 only).

- Settings for prod: `DEBUG=False`, secure cookies, `SECURE_PROXY_SSL_HEADER`, HSTS,
  `CSRF_TRUSTED_ORIGINS=https://$DOMAIN`,
  `collectstatic`, gunicorn, and API docs off.
- `compose.prod.yaml`: `db` (volume, no published port), `backend` (gunicorn), and `proxy`. The proxy is built from
  `proxy/Dockerfile` as a multi-stage build that compiles the SPA with Node and copies `dist/` and Django static into
  the Caddy image. `Caddyfile.prod` serves `{$DOMAIN}` with automatic Let's Encrypt HTTPS.
  All services use `restart: unless-stopped`.
- Deploy: clone the repo on the VPS, create `.env`, then
  `docker compose -f compose.prod.yaml up -d --build` and `createsuperuser`. Updates are `git pull` and the same
  command.
- Backups: a nightly host cron job runs `docker compose -f compose.prod.yaml exec -T db pg_dump …` and gzips the
  result to `~/backups`, keeping 14 days. The README documents how to restore.
- The README gets a "Deployment" section.

**Try it:** `https://<domain>` has a valid certificate. On Android Chrome, log in, choose ⋮ → *Install app*, and it
opens standalone from the home screen. Record a real workout and see it in History.

## Docs deliverables

### README.md (created in Step 1, extended in each step)

1. **Move**: a one-line description (a PWA for tracking the Low Back Ability workouts), with a screenshot added later.
2. **Stack**: a short list, and a link to `specs/`.
3. **Prerequisites**: Docker Engine and the Compose v2 plugin (latest), and Git. No local Python or Node needed.
4. **Quick start**: `cp .env.example .env` → `docker compose up --build` → `createsuperuser` → http://localhost:8080.
5. **Common commands**: logs, shell, `makemigrations`/`migrate`, `load_seeds`, backend tests, frontend tests/lint,
   rebuild, reset the DB (`docker compose down -v`).
6. **Project structure**: the tree from 2.3.
7. **Seed data**: YAML is the source of truth, and when and how it is loaded.
8. **Users and auth**: there is no sign-up, users are added through `createsuperuser` or `/admin/`, and login lasts 14
   days.
9. **Installing on Android** (Step 6+).
10. **Deployment** (Step 7): VPS setup, deploying, updating, backup and restore.

### .claude/CLAUDE.md additions (keep the existing *Commits* section as is)

```markdown
## Project

Move is a PWA workout tracker. Specs are in `specs/` (`specs-mvp.md` is the current scope, and
`specs-mvp.plan.md` is the implementation plan). Keep `specs-mvp.md` in sync when a design decision changes.

## Stack & layout

- `backend/`: Django + DRF. Apps: `accounts` (custom `User` model, auth API) and `workouts` (Workout, Exercise,
  WorkoutSession, SessionExercise). API under `/api/`, snake_case JSON.
- `frontend/`: React + TypeScript + Vite + TanStack Query + Tailwind, PWA via vite-plugin-pwa.
- `proxy/`: Caddy. It is the single origin that routes `/api`, `/admin` and `/static` to the backend and
  everything else to the SPA.
- `seeds/`: the source of truth for workouts and exercises. It is loaded idempotently by `manage.py load_seeds`
  on backend start. Never delete exercises that sessions reference.

## Running things

Everything runs in Docker. Never install Python or Node dependencies on the host.

- Start: `docker compose up --build` (app at http://localhost:8080)
- Backend commands: `docker compose exec backend python manage.py <cmd>` (and `uv add <pkg>` for dependencies)
- Frontend commands: `docker compose exec frontend npm <cmd>`
- Backend tests/lint: `docker compose exec backend pytest` · `docker compose exec backend ruff check .`
- Frontend tests/lint: `docker compose exec frontend npm test` · `npm run lint` · `npm run typecheck`
- After model changes: `makemigrations`, then commit the migration together with the model change.

## Conventions

- Auth uses a Django session cookie (same origin) and CSRF via the `X-CSRFToken` header. There is no JWT and no CORS.
- Store datetimes in UTC and show them in the browser's local time. Weeks start on Monday.
- Every session endpoint is scoped to `request.user`.
```

## Verification of this document

The plan is ready when `specs/specs-mvp.plan.md` exists with Part 1–3 and the docs section. Each step should have a
"Try it" check, and no other files should change (`git status` shows only the new file).
