# Meridian — Real Estate CRM

A small CRM for a real-estate sales team: manage leads through a sales
funnel, catalogue projects/buildings/units, and book units against leads
without ever double-booking the same unit.

Built for the Full-Stack Developer technical assignment.

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Backend | **FastAPI** + SQLAlchemy + SQLite | Fast to build correctly, automatic OpenAPI docs at `/docs`, typed request/response models via Pydantic. SQLite means the reviewer runs one command — no external database to install. Swapping to Postgres later is a one-line `DATABASE_URL` change. |
| Auth | JWT (python-jose) + bcrypt (passlib) | Stateless, simple to reason about, standard for an API consumed by a separate frontend. |
| Frontend | Vanilla JS (ES modules) + hand-written CSS | No build step — open `index.html` (served by the backend) and it works. Keeps the review focused on architecture and logic rather than a framework's opinions. |

No frontend framework was used deliberately — for an app this size a
build step (Vite/webpack/React) adds ceremony without adding clarity. The
frontend is still organized the way a framework app would be: a thin
`api.js` data layer, a single `app.js` with a small hash-router, and one
render function per view.

---

## Getting started

### 1. Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

# optional: populate demo data (recommended for first run)
python -m app.seed

uvicorn app.main:app --reload --port 8000
```

The API is now running at **http://localhost:8000** — and since `main.py`
also mounts the `frontend/` folder as static files, the **whole app**
(frontend + API) is available at **http://localhost:8000** with nothing
else to start.

Interactive API docs: **http://localhost:8000/docs**

### 2. Frontend (already served above)

No build step. If you'd rather run it separately (e.g. from VS Code's Live
Server) point it at `frontend/index.html` and it will call the API at
`http://localhost:8000/api` — CORS is open for local development.

### Demo accounts (created by `python -m app.seed`)

| Role | Email | Password |
|---|---|---|
| Admin | `admin@realestatecrm.io` | `admin123` |
| Sales | `priya@realestatecrm.io` | `sales123` |
| Sales | `arjun@realestatecrm.io` | `sales123` |

If you skip seeding, registering via `/api/auth/register` the *first*
time creates an Admin account (see decision #2 below); every account
after that must be created by an admin from **Team**.

### Resetting the database

Delete `backend/crm.db` and re-run `python -m app.seed`.

---

## Feature checklist against the brief

- **Leads** — create, edit, search (name/phone/email), filter by stage,
  7-stage funnel (New → Contacted → Site Visit → Interested →
  Negotiation → Booked → Lost), assignment to a sales employee, notes and
  a follow-up date.
- **Properties** — Projects → Buildings → Units hierarchy; units carry
  price, type, area, bedrooms, and availability status.
- **Bookings** — connect a lead to a unit; **a unit can never be
  double-booked** (see decision #1).
- **Dashboard** — total leads by stage, unit availability, bookings this
  month, booked revenue, and an upcoming-follow-ups list.
- **Auth** — JWT login, Admin and Sales Employee roles with different
  permissions throughout (see decision #3).
- **UI/UX** — responsive layout, search/filtering, loading skeletons,
  empty states, and error states with retry, on every list view.

---

## Key decisions

**1. Double-booking is prevented at two layers, not one.**
The obvious approach — check `unit.status == 'Available'` then create the
booking — has a race condition: two sales reps could both pass the check
in the same instant and both book the unit. So the booking-creation
endpoint checks *and* the database enforces a **partial unique index**
(`unit_id` unique where `status = 'Active'`). If two requests ever race
past the application check, the second `INSERT` fails at the database
level and the API turns that into a clean "this unit was just booked by
someone else, please refresh" response instead of a 500 error. This is
the one place in the assignment where "prevent two users from booking
the same unit" actually has to be a hard guarantee, not a best-effort UI
check.

**2. Bootstrapping admin access without an open registration endpoint.**
The brief asks for Admin and Sales roles, but doesn't say how the very
first account gets created. Leaving `/register` open to anyone would let
a random visitor grant themselves Admin; disabling it entirely means
nobody can ever log in on a fresh database. The resolution: registration
is only allowed while the `users` table is empty (and that first account
is always Admin); once any account exists, all further accounts must be
created by an existing Admin from the **Team** page. This mirrors how
most real admin panels bootstrap themselves.

**3. Sales employees see their own pipeline; Admins see everything.**
A sales rep's lead list and booking list are filtered to leads assigned
to them (plus unclaimed leads, so nothing new is invisible to everyone).
Admins see all leads, all bookings, and are the only ones who can delete
a lead outright, create/edit projects, buildings and units, or
permanently remove a lead — a sales rep marks a dead lead **Lost**
instead of deleting it, preserving history for reporting.

**4. A newly created lead is auto-assigned to whoever created it (if
they're Sales).**
Rather than leaving new leads unowned by default, the person who captured
the lead becomes its owner immediately, matching how a real sales team
actually works — the person who answers the phone owns the follow-up
unless a manager reassigns it.

**5. Cancelling a booking releases the unit, but keeps history.**
Bookings aren't deleted on cancellation — they're marked `Cancelled` with
a timestamp, and the unit flips back to `Available`. This means the
booking table is a complete, honest history of everything that happened
to a unit, not just its current state, which is exactly the kind of audit
trail a sales manager would expect to be able to pull later.

---

## Database / API overview

### Schema (SQLite via SQLAlchemy)

```
users        id, name, email, hashed_password, role[ADMIN|SALES], is_active
leads        id, name, phone, email, source, stage, budget_min/max,
             notes, follow_up_date, assigned_to_id → users.id
projects     id, name, city, address, description
buildings    id, name, project_id → projects.id
units        id, unit_number, building_id → buildings.id, unit_type,
             price, area_sqft, bedrooms, status[Available|Booked|On Hold]
             (unique per building+unit_number)
bookings     id, lead_id → leads.id, unit_id → units.id,
             created_by_id → users.id, status[Active|Cancelled],
             booking_amount, notes, created_at, cancelled_at
             (partial-unique on unit_id where status='Active')
```

### API surface (all under `/api`, all — except `/auth/login` and the
bootstrap `/auth/register` — require a `Bearer` JWT)

```
POST   /auth/register          bootstrap-only: first user ever, becomes Admin
POST   /auth/login              → { access_token, user }
GET    /auth/me
GET    /auth/users               list staff (for assignment dropdowns)
POST   /auth/users               admin-only: create a staff account

GET    /leads?search=&stage=&assigned_to_id=
POST   /leads
GET    /leads/{id}
PUT    /leads/{id}
DELETE /leads/{id}                admin-only

GET    /properties/projects       POST (admin)   DELETE (admin)
GET    /properties/buildings      POST (admin)   DELETE (admin)
GET    /properties/units?building_id=&project_id=&status=&unit_type=
                                   &min_price=&max_price=&search=
POST   /properties/units          admin-only
PUT    /properties/units/{id}     admin-only
DELETE /properties/units/{id}     admin-only

GET    /bookings
POST   /bookings                  the double-booking-safe endpoint
POST   /bookings/{id}/cancel

GET    /dashboard                 aggregate stats for the current user's scope
```

Full request/response schemas are viewable interactively at `/docs`
(FastAPI's auto-generated OpenAPI UI) once the server is running.

---

## What I'd add with more time

- Automated tests (pytest for the API — particularly a concurrency test
  hammering `/bookings` for the same unit; a couple of Playwright tests
  for the booking-conflict UI path).
- Pagination on the leads/units tables (fine at demo scale, would matter
  at real scale).
- Soft-delete + audit log for leads, so "who changed what, when" is
  fully reconstructible, not just booking history.
- Postgres in place of SQLite for concurrent-write headroom in
  production, plus Alembic migrations.
