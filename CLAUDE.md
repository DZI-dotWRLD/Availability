# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Goal

Aability is a web app for tennis club members. Its purpose is to **increase match play and get members playing with
each other**. Members register, say when they're free, and the app finds doubles matches for them. The ladder rewards
playing often, not winning.

## Product rules

- **Registration:** self-registration with email verification (no admin approval or invite code). On sign-up the user
  picks a ladder: **Men, Women, Boys, or Girls**.
- **Juniors** (Boys/Girls) have their own login, but registration requires a parent email, and that parent is copied on
  notifications.
- **Format:** doubles with **rotating partners**. Individuals are ranked, not fixed teams; matchmaking forms the pairs.
- **Ladder rank = number of matches played** in the current season. There is no score submission and no win/loss
  record. Ladders **reset each season**. An admin sets each season's start and end dates.
- **A match counts automatically** once its time slot passes, unless it was cancelled first. Nobody confirms it.
  Members are trusted not to fake matches, so there is no anti-cheating mechanism. This was a deliberate choice; don't
  propose one again.
- **Availability:** a user submits multiple free-form time ranges when they can play (e.g. Tue 17:30–20:00). Each range
  is at least 1h30. A match lasts 1h30 and can start anywhere inside the overlap of all 4 players' ranges.
- **Matchmaking:** finds 4 players from the **same ladder** whose availability overlaps, preferring players **close in
  rank**. The proposed match is booked only when **all 4 accept**. If someone declines, the next candidate is tried.
- **Waiting:** if no match exists yet, the user's availability stays open. When a match becomes possible (e.g. someone
  else adds availability), they are notified by **email and in-app notification**.

- **Ties:** players with equal match counts share a rank (5, 5 → both 3rd).
- **Close in rank:** prefer players within ±5 positions; fall back to anyone in the ladder if that can't form a match.
- **Cancellation:** any participant can cancel a booked match before it starts. The other 3 are notified that the match
  is cancelled and their times are available again. Their availability returns to the pool and matchmaking reruns.
- **Out of scope:** court booking. The app only brings players together.

## Starting point

The code is a snapshot of BC_Ladder, an earlier Django app for fixed-team doubles ladders with score submission. The
code, `REQUIREMENTS.md`, `docs/architecture.md`, `docs/booking-concurrency.md`, and `docs/matchmaking-candidates.md`
still describe **that** domain. Treat them as a reference, not as the spec for new work.

- **Worth keeping:** registration with email verification and rate limiting, availability windows, the
  suggestion → all-sides-accept → match booking flow with its locking and overlap constraints, the email outbox, the
  settings hardening, and the deployment setup.
- **Obsolete under the new rules:** `Team`/`TeamMembership` (fixed teams); score submission, results, the point ledger,
  score corrections and conflict notifications; the points/wins-based standings. The ladder only covers Men/Women,
  with no Boys/Girls or parent email.

## Commands

Windows/PowerShell, Python 3.14, venv in `./venv`. `.env` (copied from `.env.example`) sets `DJANGO_DEBUG=true`; without
it, settings fail closed and refuse to start unless full production config is present.

```powershell
py -3.14 -m venv venv
.\venv\Scripts\python.exe -m pip install -r requirements-dev.txt
Copy-Item .env.example .env
.\venv\Scripts\python.exe manage.py migrate
.\venv\Scripts\python.exe manage.py seed_demo        # idempotent; demo-admin / demo-mens-1, password DemoPass123!
.\venv\Scripts\python.exe manage.py runserver 127.0.0.1:8000
```

Tests use Django's runner (no pytest). Test modules are `ladder/tests_*.py`:

```powershell
.\venv\Scripts\python.exe manage.py test --noinput
.\venv\Scripts\python.exe manage.py test ladder.tests_scoring
.\venv\Scripts\python.exe manage.py test ladder.tests_scoring.ClassName.test_name
```

Quality gate (check, migration drift, ruff format/lint, `git diff --check`, tests, pip-audit):

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\quality.ps1
powershell -ExecutionPolicy Bypass -File .\scripts\quality.ps1 -SkipFullTests -SkipAudit   # fast pass
```

Ruff: line length 140, rules `E,F,I,B,UP`, migrations excluded. Browser checks live in `tests/frontend/` (Playwright,
`npm test` there). They need a running server, and CI runs them.

PostgreSQL is the production target. Some tests (row locking, exclusion constraints, advisory locks) are skipped on
SQLite and run only when `DJANGO_DB_ENGINE=django.db.backends.postgresql` and the `DJANGO_DB_*` vars are set. CI runs
both backends.

## Architecture (current code)

One Django project (`config/`) and one app (`ladder/`). Keep the layering and locking rules below when building the
new domain. The model-specific details will change.

- **Layering:** views (`ladder/views.py`) handle auth, object visibility and HTTP only, and must not write domain
  state. All mutations go through `ladder/services.py`. Service functions take an `actor`, enforce authorization
  themselves, run in `transaction.atomic`, and raise `DomainError` subclasses (`InvalidInput`, `StaleState`,
  `AuthorizationFailure`, `BookingCollision`, `ProvisioningError`). Views translate these errors into messages.
- **Locking order:** mutations that affect booking lock `PlayerProfile` rows in ascending PK order, then teams, then
  the suggestion or membership, then availability, then reservations. Standings writers take a per-division PostgreSQL
  advisory lock first. Keep this order in new code, or deadlocks follow (see `docs/booking-concurrency.md`).
  PostgreSQL exclusion constraints (migration `0012`) are the final guard against overlapping intervals.
- **History is immutable:** participants, results, the point ledger, audits and workflow events are append-only and
  protected by `PROTECT` FKs. Corrections are new audited rows, not edits.
- **Events and email:** `workflow_events.py` appends `WorkflowEvent` with snapshotted recipients. The same transaction
  creates `EmailNotificationDelivery` outbox rows, and `email_notifications.py` delivers them according to
  `DJANGO_NOTIFICATION_DELIVERY_MODE`: `inline` (dev/tests), `thread`, or `scheduled` (prod, via
  `send_notification_emails` every minute). Account verification and password reset emails bypass the outbox.
- **Settings:** `config/settings.py` parses everything from env vars and validates the production config when
  `DJANGO_DEBUG=false`. It also wires in django-axes (login lockout), django-csp, whitenoise and Sentry
  (`config/sentry.py`, PII-filtered).
- **Frontend:** Django templates in `ladder/templates/`, with partials for the fragments that `ladder/static/ladder/app.js`
  loads. Pages must keep working without JavaScript.
- **Ops:** management commands in `ladder/management/commands/` (seed, reconcile/audit, expiry, anonymize, bootstrap
  admin, outbox worker). Deployment targets Render (`render.yaml`, `build.sh`, `DEPLOYMENT.md`).
