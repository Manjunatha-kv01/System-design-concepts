# Parking Lot Management System

A production-shaped backend implementation of the classic "design a parking
lot" system design exercise: a FastAPI service backed by PostgreSQL
(SQLite for local dev/tests), with real database persistence, race-safe
spot allocation, Alembic migrations, Docker packaging, and a pytest suite
that includes an actual concurrent-request test.

This takes the in-memory Python model from the design walkthrough and
implements the "Part 13: turn the mini-design into a real backend
service" section for real, rather than describing it.

## Architecture

```
app/
  core/
    enums.py        VehicleType, SpotType, SpotStatus, TicketStatus,
                     and the vehicle -> spot-type preference table
    exceptions.py    HTTP-agnostic domain exceptions
  models/
    db_models.py     SQLAlchemy ORM models (lots, floors, spots,
                      tickets, active_sessions)
  schemas/
    schemas.py       Pydantic request/response models
  services/
    fee_calculator.py   FeeStrategy interface + HourlyFeeStrategy
    parking_service.py  park() / exit_vehicle() / availability() —
                         the concurrency-safe allocation logic lives here
    admin_service.py    facility setup: create lot/floor/spot
  api/
    routes.py        HTTP routes
    deps.py          DB session + admin-auth dependencies
  main.py             FastAPI app, exception -> HTTP status mapping
  config.py           env-driven settings
  database.py         SQLAlchemy engine/session setup
alembic/               schema migrations
tests/                 pytest suite (unit + API + concurrency)
scripts/seed.py         demo data for local runs
```

**Why this layering:** `ParkingService` has no idea it's running behind
HTTP — it takes a SQLAlchemy `Session` and domain values, and raises
plain Python exceptions. `app/main.py` is the only place that knows
`NoSpotAvailableError` means HTTP 409. That separation is what makes the
service layer testable without spinning up a web server (see
`tests/test_parking_service.py`) and portable to a different transport
later (a queue worker, a CLI, gRPC) without rewriting business logic.

## The concurrency problem, and how it's actually solved

The design doc's Part 12/13 identify the core production risk: two
requests can both read a spot as "available" before either finishes
writing, and both believe they got it. This project solves that two
ways, both enforced by the database rather than by application-level
locking:

1. **Spot allocation** (`ParkingService._claim_spot`): instead of
   `SELECT` a free spot, then `UPDATE` it, we run a single conditional
   `UPDATE parking_spots SET status='occupied' WHERE id=:id AND
   status='available'`. If a competing request already flipped that row,
   `rowcount` is 0 and we simply retry against the next candidate — the
   loser never believes it won. This works identically on SQLite and
   Postgres without needing `SELECT ... FOR UPDATE`.

2. **Duplicate active-ticket prevention** (`ActiveSessionModel`): a
   separate table holds exactly one row per currently-parked plate, with
   a `UNIQUE` constraint on `license_plate`. Opening a ticket means
   inserting into this table; if a racing request already inserted a row
   for the same plate, the database raises an `IntegrityError`, which we
   translate into `DuplicateEntryError`. The uniqueness constraint *is*
   the lock.

`tests/test_parking_service.py::test_two_cars_compete_for_the_last_compact_spot`
exercises this directly: two real threads call `park()` for the same
spot type against a shared file-backed SQLite database at (as close to)
the same instant, synchronized with a `threading.Barrier`. Exactly one
succeeds; the other gets `NoSpotAvailableError`; the spot is never
double-allocated.

## Design patterns used (Part 14 of the write-up)

- **Strategy** — `FeeStrategy` / `HourlyFeeStrategy` in
  `fee_calculator.py`. Adding a flat weekend rate or a membership
  discount later means writing a new `FeeStrategy` subclass, not
  touching `ParkingService`.
- **Repository-ish separation** — `admin_service.py` (facility setup)
  is kept separate from `parking_service.py` (the park/exit workflow)
  because they're different responsibilities with different trust
  boundaries (the latter is behind an admin API key).

## Running it

### Option A: local, SQLite, zero setup

```bash
pip install -r requirements-dev.txt
python scripts/seed.py          # creates parking.db with sample lot/floors/spots
uvicorn app.main:app --reload
```

Open http://localhost:8000/docs for interactive Swagger UI.

### Option B: Docker Compose (Postgres)

```bash
docker compose up --build
```

This starts Postgres, runs Alembic migrations, then starts the API on
`localhost:8000`. Seed it the same way, pointed at the compose stack:

```bash
DATABASE_URL=postgresql+psycopg2://parking:parking@localhost:5432/parking python scripts/seed.py
```

### Running tests

```bash
pytest -v
```

### Migrations

```bash
alembic upgrade head                                  # apply
alembic revision --autogenerate -m "describe change"  # after editing db_models.py
```

## API reference

All request/response bodies are JSON.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/parking/entries` | — | Park a vehicle, get back a ticket |
| GET | `/parking/tickets/{ticket_code}` | — | Look up a ticket |
| POST | `/parking/tickets/{ticket_code}/exit` | — | Close a ticket, compute the fee |
| GET | `/parking/availability` | — | Spot counts by type |
| GET | `/parking/floors/{floor_id}/spots` | — | List a floor's spots |
| POST | `/admin/lots` | `X-Admin-Api-Key` | Create a lot |
| POST | `/admin/floors` | `X-Admin-Api-Key` | Create a floor |
| POST | `/admin/spots` | `X-Admin-Api-Key` | Create a spot |

**Entry request:**
```json
{ "license_plate": "KA-01-AB-1234", "vehicle_type": "car" }
```

**Ticket response:**
```json
{
  "ticket_code": "T000001",
  "license_plate": "KA-01-AB-1234",
  "vehicle_type": "car",
  "entry_time": "2026-10-28T09:00:00",
  "exit_time": null,
  "fee": null,
  "status": "active",
  "spot_label": "C-01",
  "floor_name": "Ground Floor"
}
```

Error responses are `{"detail": "..."}` with these status codes:

| Status | Meaning |
|---|---|
| 400 | `exit_time` not after `entry_time` |
| 401 | missing/invalid admin API key |
| 404 | ticket/floor/lot not found, or vehicle not currently parked |
| 409 | no compatible spot available, or duplicate active ticket |

## What's intentionally out of scope

Matching the design doc's own stated assumptions, this project does not
implement real payments, reservations, discounts/memberships, or
hardware gate integration. Each has an obvious extension point:

- **Payments** → a `PaymentService` called from `exit_vehicle` before
  the spot is released (the design doc explicitly flags that the spot
  must not be freed if payment is required but fails).
- **Reservations** → a `Reservation` model + a check in `_claim_spot`.
- **Pricing changes** → a new `FeeStrategy` implementation.
- **Multi-tenant auth** → replace the shared-secret `X-Admin-Api-Key`
  dependency in `app/api/deps.py` with real auth (OAuth/JWT/mTLS); the
  rest of the code doesn't need to change.

## Relation to the original design-interview write-up

This project is the practical counterpart to the companion design
document's mastery checklist (Part 17) — the two are meant to be used
together: read the design rationale there, see the production
implementation of each piece here (allocation algorithm in
`_claim_spot`, enums in `app/core/enums.py`, the fee edge cases in
`tests/test_fee_calculator.py`, concurrency handling as above).
