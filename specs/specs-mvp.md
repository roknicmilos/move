# Project Specifications

This project should be a PWA that tracks my workouts. I should be able to install it on my Android like a
native app. It should have a backend REST API with a database and frontend that talks to that API.

Information in this document is just an idea, a draft. This means that it should be reviewed and better alternatives
should be considered and suggested. Terminology and component (classes, variables, functions, etc.) names are also
things that should be considered for improvements.

## Workouts

The project should support 2 different workouts represented by a single Workout entity. These 2 workouts are:

- Flow: Main Low Back Mobility Session
- Mobility Flow: Supporting Low Back Mobility Session

### Entities

- Workout
    - name: str
    - short_name: str
    - description: str
    - exercises: Exercise collection (reverse FK, ordered by `position`)
- Exercise
    - workout: Workout (FK)
    - name: str
    - position: int (unique per workout, sets the flow order)
    - instructions: str
    - purpose: str (optional)
    - sets: int (optional)
    - reps: int (optional)
    - duration_sec: int (seconds, per set) optional
    - load_kg: float (kilograms, per set) optional
    - per_side: bool (default false)
- WorkoutSession
    - user: User (FK, required)
    - workout: Workout (FK, `PROTECT`)
    - started_at: datetime (set by the server, on create)
    - ended_at: datetime (nullable; `NULL` means in progress)
    - exercises: SessionExercise collection
    - constraints: at most one in-progress session per user, and `ended_at >= started_at`
- SessionExercise
    - session: WorkoutSession (FK, related name `exercises`)
    - exercise: Exercise (FK, `PROTECT`)
    - completed_at: datetime (created when checked, deleted when unchecked)
    - sets: int (optional)
    - reps: int (optional)
    - duration_sec: int (seconds) optional
    - load_kg: float (kilograms) optional
    - unique `(session, exercise)`
- User: custom `accounts.User`, extending Django's `AbstractUser` (the recommended base when the built-in user model
  needs any change). `username` is the login field, required and unique; `email` is optional. No registration;
  created with `manage.py createsuperuser` (the owner) and through Django admin (everyone else). Credentials are
  never committed.

### Fixtures (Seeds)

See [workouts.yaml](../seeds/workouts.yaml) and [exercises.yaml](../seeds/exercises.yaml) for the full list of
Workout and Exercise seed data (referenced by `workout` FK on each Exercise).

## Web App Features

- User should be able to have an overview of all workouts. Selecting a workout leads to that workout page with details
  about that workout and list of all exercises. Selecting an exercise leads to that exercise page with details about
  that exercise.
- User should be able to record a workout. When user selects a workout, there should be a button to start a workout. It
  opens a page that clearly shows that workout is in progress. Each exercise has a checkmark that user can select after
  completing that exercise. Starting a workout automatically sets started_at for that workout session. There should be a
  button to end the workout that sets ended_at for that workout.
- User should be able to see a history of their workouts. They should be able to select a view for this history: weekly
  and monthly. Default is weekly.

## Authentication

The app has predefined users. Each user has a username and password. There is no registration, only login. Authenticated
user session should expire after 2 weeks.

## Architecture

See [specs-mvp.plan.md](specs-mvp.plan.md) Part 2 for the full detail (stack versions, services, repository
layout, API, frontend routes). Summary:

### Backend

Python, Django, Django REST Framework, drf-spectacular (OpenAPI and Swagger UI), gunicorn (prod), `uv` for
dependencies, PostgreSQL. Apps: `accounts` (custom `User` model, auth API) and `workouts` (the other 4 models,
`load_seeds` command, API).
Session cookie auth (not JWT): the frontend and API are served from the same origin, so there is no CORS setup and
no tokens in JS storage. CSRF uses Django's cookie plus an `X-CSRFToken` header. The login session cookie expires
14 days from login (`SESSION_COOKIE_AGE = 1209600`, not extended on each request). Seeds (`seeds/workouts.yaml`,
`seeds/exercises.yaml`) are the source of truth for workouts and exercises: `load_seeds`
upserts by `id`, is safe to run more than once, and runs on every backend start. Removing an entry from YAML does
not delete it from the DB, because history references it. Datetimes are stored in UTC (`USE_TZ = True`).

### Frontend

React, TypeScript, Vite, React Router, TanStack Query, Tailwind CSS, date-fns, `vite-plugin-pwa`, npm. Mobile
first, with a bottom nav: Workouts · History · (account/logout). Routes:

| Route                                        | Page                                                                                |
|----------------------------------------------|-------------------------------------------------------------------------------------|
| `/`                                          | Workouts overview (landing), with a "Resume workout" banner when one is in progress |
| `/login`                                     | Login                                                                               |
| `/workouts/:workoutId`                       | Workout details, exercise list, **Start workout** button                            |
| `/workouts/:workoutId/exercises/:exerciseId` | Exercise details                                                                    |
| `/sessions/:sessionId`                       | Recording page while in progress, summary once ended                                |
| `/history?view=week\|month&date=YYYY-MM-DD`  | History (weekly by default)                                                         |

Times are shown in the browser's local time zone. History weeks start on Monday, and week/month ranges are
computed in local time.

### Infrastructure

Everything runs in Docker Compose: `db` (PostgreSQL), `backend` (Django), `frontend` (Vite dev server, dev only),
and `proxy` (Caddy), which is the single origin routing `/api`, `/admin`, `/static` to the backend and everything
else to the SPA. Nothing needs to be installed on the host except Docker and Git.
