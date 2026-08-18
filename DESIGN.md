# Video Call Scheduler — Design Specification

**Status:** Draft v1 · **Target:** first open-source release (v0.1)

A self-hosted Python web app that lets a small, trusted group (e.g. a family)
schedule Google Meet video calls with a person who cannot operate a computer
themselves (e.g. an elderly parent). A companion **kiosk agent** runs on the
recipient's laptop and automatically opens Chrome and joins the meeting at the
scheduled time.

Throughout this document the person being called is the **recipient**, the
people booking calls are **members**, and the person who runs the deployment is
the **admin**. The software is generic: nothing in the code should assume
"mom" or "family" beyond the docs and default copy.

---

## 1. Goals and non-goals

### Goals (v0.1)

- Admin manually manages members and assigns each member a Google Meet URL.
- Members sign in (with near-zero friction) and book, view, and cancel time
  slots on a calendar.
- One shared calendar for the recipient: no two members can book overlapping
  slots.
- A kiosk agent on the recipient's machine, driven by cron, joins the right
  Meet call at the right time with **zero interaction from the recipient**.
- Small, well-tested codebase that a hobbyist can self-host (single server,
  SQLite, no external services required).

### Non-goals (v0.1)

- No Google account integration (OAuth, Calendar API, Meet API). Meet URLs are
  pasted in by the admin. (Roadmap item — see §12.)
- No automatic Meet link creation, no email/SMS notifications, no recurring
  bookings. (Roadmap items.)
- No multi-recipient support: one deployment = one recipient = one calendar.
- No real-time UI (websockets); plain server-rendered pages are fine.
- Not a general-purpose booking system (no payments, no public signup).

---

## 2. System overview

Two deployable components sharing one repository:

```
┌────────────────────────┐         ┌───────────────────────────────┐
│  Scheduler web app     │  HTTPS  │  Kiosk agent                  │
│  (any host / home      │◄────────┤  (recipient's laptop)         │
│   server / cheap VPS)  │  poll   │                               │
│                        │         │  cron tick (every minute)     │
│  Flask + SQLite        │         │   ├─ sync schedule → local    │
│  - admin UI            │         │   │  cache (JSON)             │
│  - member booking UI   │         │   └─ launch/stop decision     │
│  - JSON API for agent  │         │  joiner (Playwright+Chrome)   │
└────────────────────────┘         └───────────────────────────────┘
```

- **Web app** — source of truth. Stores members, Meet URLs, and bookings.
  Serves the booking UI to members and a small token-authenticated JSON API to
  the agent.
- **Kiosk agent** — a small CLI installed on the recipient's machine. A cron
  entry runs `agent tick` every minute. Each tick syncs the day's schedule
  into a local cache (so a brief server outage doesn't cancel a known call)
  and decides whether to launch or stop the browser. Joining is done with
  Playwright driving the recipient's real Chrome profile.

The agent only ever **pulls**; the server never needs to reach the
recipient's machine, which keeps home-network setup trivial.

---

## 3. Tech stack

| Concern        | Choice                          | Rationale                                                             |
| -------------- | ------------------------------- | --------------------------------------------------------------------- |
| Language       | Python 3.11+                    | Requirement; broad contributor base.                                  |
| Web framework  | Flask                           | Small surface, server-rendered forms fit the UI, easy to unit test.   |
| ORM / DB       | SQLAlchemy + SQLite             | Zero-ops self-hosting; SQLAlchemy leaves room for Postgres later.     |
| Migrations     | Alembic                         | Schema will evolve release to release.                                |
| Templates      | Jinja2 + a classless CSS sheet  | No JS build step; accessibility-friendly.                             |
| Agent browser  | Playwright (channel: Chrome)    | Reliable "click Join now" automation against a persistent profile.    |
| Scheduling     | System cron on the kiosk machine| Requirement; simplest thing that survives reboots.                    |
| Tests          | pytest + coverage               | Requirement: features land with unit tests.                           |
| Lint/format    | ruff (lint + format)            | One fast tool.                                                        |
| Packaging      | `pyproject.toml`, `pip install .[server]` / `.[agent]` extras | One repo, two install targets. |

Time handling: all timestamps stored in UTC; a single **household timezone**
setting (IANA name) drives display and slot arithmetic. The agent compares
times in UTC, so a timezone mismatch between server and laptop cannot cause a
missed call.

---

## 4. Data model

```
Member
  id            int PK
  name          str            # display name, e.g. "Aunt Carol"
  email         str, nullable  # optional in v0.1 (used by roadmap notifications)
  role          enum: admin | member
  meet_url      str, nullable  # required for role=member; validated https://meet.google.com/...
  login_token   str unique     # secret for magic-link sign-in (hashed at rest)
  is_active     bool           # soft delete; keeps booking history intact
  created_at    datetime (UTC)

Booking
  id            int PK
  member_id     FK → Member
  start_at      datetime (UTC)   # derived from chosen date + slot in household tz
  end_at        datetime (UTC)
  status        enum: scheduled | cancelled | completed
  created_at    datetime (UTC)
  cancelled_at  datetime (UTC), nullable

  constraint: no two bookings with status=scheduled may overlap
              (enforced in the service layer inside a transaction,
               plus a unique index on start_at where status='scheduled'
               as a backstop — sufficient because all slots share one grid)

Setting (single-row key/value or a one-row table)
  household_timezone   e.g. "America/Chicago"
  slot_minutes         default 30
  day_start / day_end  bookable window, e.g. 09:00–20:00 recipient-local
  booking_horizon_days default 30      # how far ahead members can book
  recipient_name       display only, e.g. "Grandma June"

AgentDevice
  id            int PK
  name          str            # "Mom's laptop"
  api_token     str unique     # hashed at rest; sent as Bearer token
  last_seen_at  datetime, nullable   # updated on every sync; surfaces dead agents
```

**Slot model.** Bookable times form a fixed grid: from `day_start` to
`day_end` in steps of `slot_minutes`, computed in the household timezone (so
the grid stays aligned across DST changes). A booking occupies exactly one
slot in v0.1 (multi-slot bookings are a roadmap item). Because every booking
snaps to the same grid, "no overlap" reduces to "no duplicate `start_at`",
which keeps the conflict check simple and race-safe.

---

## 5. Authentication

Optimized for a small trusted group and minimal friction:

- **Members: magic links.** When the admin creates a member, the app
  generates a long random token and shows the admin a personal URL
  (`/login/<token>`) to share once (text message, etc.). Visiting it sets a
  long-lived session cookie. No passwords, no email infrastructure needed in
  v0.1. Admin can regenerate a member's link at any time (invalidates the old
  one). Tokens are stored hashed; the plaintext is only displayable at
  creation/regeneration time.
- **Admin: username + password**, set at install time via CLI
  (`scheduler create-admin`), stored as a passlib/argon2 hash. Admin uses the
  same session mechanism plus access to `/admin`.
- **Agent: static bearer token** per device, created in the admin UI, stored
  hashed. Sent as `Authorization: Bearer …` on API calls.
- Sessions: Flask's signed cookie session with a server-side `SECRET_KEY`;
  CSRF protection on all mutating forms (Flask-WTF).
- The app must be deployed behind HTTPS (documented requirement; magic links
  are bearer secrets).

---

## 6. Web app: pages and behavior

### Member-facing

| Route                        | Purpose                                                                 |
| ---------------------------- | ----------------------------------------------------------------------- |
| `GET /login/<token>`         | Magic-link sign-in → redirect to `/`                                    |
| `GET /`                      | Week view: 7-day grid of slots — free / mine / taken (with member name) |
| `POST /bookings`             | Book a slot (date + start time); re-renders with confirmation           |
| `POST /bookings/<id>/cancel` | Cancel **own** upcoming booking                                         |
| `GET /bookings/mine`         | List of my upcoming and past bookings                                   |

Booking rules (service layer, all unit-tested):

- Slot must lie on the grid, within the bookable window, in the future, and
  within `booking_horizon_days`.
- Slot must be free (checked and inserted in one transaction).
- Members see who holds a taken slot (it's a family; transparency is a
  feature), but can only cancel their own.
- Cancelling frees the slot immediately; the booking row is kept with
  `status=cancelled` for history.

### Admin-facing (`/admin`, role-gated)

- Member CRUD: create/deactivate members, set name and **Meet URL**
  (validated against `https://meet.google.com/…`), regenerate magic links.
- Settings: timezone, slot length, daily window, horizon, recipient name.
- Devices: create/revoke agent tokens, see `last_seen_at` health.
- Bookings: view all; cancel any booking on a member's behalf.

Admins can also be members (book slots) if they have a `meet_url`.

### UI notes

- Server-rendered, no JS required to book (progressive enhancement only).
- Large tap targets and readable fonts — some members may be elderly too.
- Every page shows times in the household timezone with an explicit label
  ("All times are Grandma June's time") to avoid cross-timezone confusion.

---

## 7. Agent API (JSON, bearer-token auth)

```
GET /api/v1/schedule?from=<iso>&to=<iso>
→ 200 {
    "generated_at": "2026-08-18T14:00:00Z",
    "timezone": "America/Chicago",
    "bookings": [
      {
        "id": 42,
        "member_name": "Aunt Carol",
        "meet_url": "https://meet.google.com/abc-defg-hij",
        "start_at": "2026-08-18T19:00:00Z",
        "end_at": "2026-08-18T19:30:00Z"
      }
    ]
  }

POST /api/v1/bookings/<id>/events        # best-effort telemetry
  body: {"event": "join_started" | "join_succeeded" | "join_failed" | "left",
         "at": "<iso>", "detail": "optional string"}
→ 204
```

Notes:

- `meet_url` is resolved server-side from the booking's member — the agent
  never needs the member table.
- Every authenticated call updates `AgentDevice.last_seen_at`.
- The events endpoint exists so the admin UI can show "call joined ✓ / join
  failed ✗" per booking; failures of this endpoint never block joining.
- API is versioned (`/api/v1/`) because the agent and server are deployed on
  different machines and may skew.

---

## 8. Kiosk agent

A console script (`kiosk-agent`) installed on the recipient's machine
(Linux or macOS laptop; Windows support is a roadmap item). Configured by a
single file, e.g. `~/.config/video-call-scheduler/agent.toml`:

```toml
server_url   = "https://calls.example.com"
api_token    = "…"
chrome_profile_dir = "/home/june/.config/kiosk-chrome-profile"
join_early_seconds = 60      # open the call this early
hangup_grace_seconds = 120   # linger after end_at before closing
cache_path   = "~/.local/state/video-call-scheduler/schedule.json"
log_path     = "~/.local/state/video-call-scheduler/agent.log"
```

### Cron integration

Installed by `kiosk-agent install-cron` (and removable by `uninstall-cron`):

```
* * * * * /usr/local/bin/kiosk-agent tick >> ~/.local/state/video-call-scheduler/cron.log 2>&1
```

`tick` must complete in well under a minute; it never blocks on the call
itself. Long-running work (the actual browser session) is spawned as a
detached **joiner** process. A pid/lockfile prevents overlapping joiners:

### `tick` algorithm

1. **Sync:** `GET /api/v1/schedule` for today ± 1 day; on success, atomically
   overwrite the local cache. On network failure, log and fall back to the
   cached copy (a server blip must not cancel a call the agent already knows
   about).
2. **Decide:** pure function `decide(now, schedule, joiner_state) → action`
   where action is one of:
   - `LAUNCH(booking)` — a booking satisfies
     `start_at - join_early ≤ now < end_at` and no joiner is running.
   - `STOP` — joiner is running but its booking ended more than
     `hangup_grace_seconds` ago (or was cancelled server-side — cancellation
     propagates via the sync in step 1, and `STOP` also fires for a running
     joiner whose booking has vanished from the schedule).
   - `NONE` — otherwise.
3. **Act:** spawn/kill the joiner accordingly; post a telemetry event
   (best-effort).

Keeping `decide` pure (inputs: clock, schedule list, running-joiner metadata)
makes the trickiest logic — restarts mid-call, missed ticks after laptop
sleep, cancelled bookings, back-to-back calls — trivially unit-testable with
a fake clock and no I/O.

Missed-tick behavior falls out of the rule above: if the laptop wakes at
19:10 for a 19:00–19:30 call, the next tick still matches `now < end_at` and
joins late rather than never.

### Joining the Meet

The joiner uses Playwright with a **persistent Chrome profile** that the
admin prepares once during setup (sign the recipient's Google account into
that profile; visit a test Meet and grant camera + microphone permission so
the grants are stored in the profile). Then, per call:

1. Launch Chrome (real Chrome channel, not bundled Chromium — Meet treats it
   better) with the persistent profile, full-screen/kiosk, on the Meet URL.
2. Wait for the pre-join screen; ensure mic and camera are **on** (click the
   toggles if Meet remembered them off).
3. Click "Join now" / "Ask to join". Selectors live in one small, documented
   module (`joiner/selectors.py`) because Google changes the Meet DOM
   periodically and this will be the #1 maintenance hotspot.
4. Report `join_succeeded` once in-call (participant UI detected), or
   `join_failed` with detail after a bounded retry (2 attempts).
5. **Degraded fallback:** if automation fails at step 2–3, leave the browser
   open on the pre-join/lobby screen rather than closing it — the family
   member on the other end can then admit the recipient, and the screen is at
   least showing the right place.
6. At `end_at + grace`, close the browser and report `left`.

Reality checks encoded in the design:

- If the member's Meet link uses "Ask to join", the **member must be in the
  meeting first** to admit the recipient. Docs will recommend members join a
  couple of minutes early; `join_early_seconds` defaults to 60 so the
  recipient is already waiting in the lobby.
- The laptop must be configured to never sleep (setup docs), and the agent
  logs loudly (and telemetry shows) when ticks have gaps.

---

## 9. Repository layout

```
video-call-scheduler/
├── pyproject.toml            # extras: [server], [agent], [dev]
├── DESIGN.md
├── README.md                 # quickstart for both components
├── docs/
│   ├── server-setup.md
│   └── kiosk-setup.md        # Chrome profile prep, cron install, no-sleep
├── src/
│   ├── scheduler/            # web app package
│   │   ├── app.py            # Flask app factory
│   │   ├── models.py
│   │   ├── services/         # booking rules, slot grid, auth (pure-ish, heavily tested)
│   │   ├── views/            # member, admin, api blueprints
│   │   ├── templates/ static/
│   │   ├── cli.py            # create-admin, init-db, …
│   │   └── migrations/
│   └── kiosk_agent/
│       ├── cli.py            # tick, install-cron, join --booking-id (manual test)
│       ├── sync.py           # HTTP client + cache
│       ├── decide.py         # the pure decision function
│       ├── joiner/
│       │   ├── runner.py     # process management, lockfile
│       │   ├── meet.py       # Playwright join flow
│       │   └── selectors.py
│       └── config.py
└── tests/
    ├── scheduler/            # unit + Flask test-client integration tests
    └── kiosk_agent/          # decide/sync/runner tests with fakes
```

---

## 10. Testing strategy

Principle: push logic out of views and I/O code into pure functions and
service classes, then unit-test those exhaustively; keep a thinner layer of
integration tests over HTTP.

- **Slot grid & booking rules** (highest value): grid generation across DST
  transitions, horizon/window enforcement, double-booking race (two inserts
  in parallel transactions), cancel-and-rebook.
- **Auth:** magic-link happy path, regenerated token invalidates old link,
  deactivated member rejected, CSRF on mutating routes, admin gating.
- **Agent API:** bearer-token auth, schedule serialization, `last_seen_at`.
- **Agent `decide`:** table-driven tests with a fake clock — before window,
  in window, mid-call restart, after grace, cancelled booking with joiner
  running, back-to-back bookings, laptop-slept-through-start.
- **Agent `sync`:** server down → cache used; malformed response → cache
  kept; atomic cache write.
- **Joiner:** `meet.py` behind a small interface; unit tests mock Playwright.
  One optional, non-CI smoke test (`pytest -m live_meet`) that a maintainer
  can run against a real Meet URL — the Meet DOM can't be meaningfully
  CI-tested.
- Tooling: `pytest`, `pytest-cov` (target ≥90% on `services/` and
  `kiosk_agent/decide.py`), `freezegun` or an injected clock, `ruff` in CI
  (GitHub Actions) on 3.11/3.12.

---

## 11. Security & privacy

- Meet URLs are effectively capabilities — the app treats them as secrets:
  visible only to their member, the admin, and the agent API. Never logged.
- All tokens (login, agent) ≥ 32 random bytes, stored hashed.
- HTTPS required in deployment docs; cookies `Secure`, `HttpOnly`, `SameSite=Lax`.
- The kiosk profile is signed into the recipient's Google account: setup docs
  cover OS user separation and disk encryption; the agent never handles the
  Google password itself.
- SQLite file and agent config contain secrets → documented file permissions
  (0600).
- No analytics, no third-party calls from the web app. Data never leaves the
  self-hosted server except to the agent.

---

## 12. Roadmap

| Release | Theme                                                                                                     |
| ------- | --------------------------------------------------------------------------------------------------------- |
| **v0.1** | Everything in this document: member/admin UI, bookings, agent + cron join. Definition of done: a family can run a real call end-to-end with no keyboard on the recipient's side. |
| v0.2    | Email notifications: booking confirmations, reminders, "call failed to join" admin alerts; ICS attachments. |
| v0.3    | Quality of life: multi-slot bookings, recurring bookings, per-member booking limits, agent health dashboard. |
| v0.4    | Google integration: OAuth + Calendar API to auto-create Meet links per booking (removes manual URL config; per-member static URLs remain as fallback). |
| v0.5    | Windows agent support; packaged installers; optional Postgres.                                            |
| Later   | Multi-recipient households; i18n; pluggable meeting providers (Jitsi, Zoom) behind a provider interface.  |

Each release lands feature-by-feature behind the existing test suite; the
provider interface in "Later" is why `meet_url`/joiner code stays isolated
from the scheduling core from day one.

---

## 13. Open questions (decide before implementation)

1. **Slot length vs. per-booking duration** — v0.1 fixes duration to one
   grid slot. Is 30 minutes the right default for the target household?
   (Configurable either way.)
2. **Member visibility** — spec says members see who booked each slot.
   Any household preference for anonymized "taken"? Could be a setting later.
3. **Agent platform for v0.1** — spec assumes the recipient's laptop runs
   Linux or macOS (cron). If the real target machine is Windows, v0.5's
   Task-Scheduler work moves up.
4. **Same Meet URL reuse** — with static per-member URLs, a member's link is
   the same every call. Acceptable for v0.1 (family-trust model); v0.4's
   Calendar integration replaces it with per-call links.
