# Video Call Scheduler — Design Specification

**Status:** Draft v2 · **Target:** first open-source release (v0.1)

A self-hosted Python web app that lets a small, trusted group (e.g. a family)
schedule Google Meet video calls with a person who cannot operate a computer
themselves (e.g. an elderly parent). A companion **kiosk agent** runs on the
recipient's **Windows** laptop and automatically opens Chrome and joins the
meeting at the scheduled time.

Throughout this document the person being called is the **recipient**, the
people booking calls are **members**, and the person who runs the deployment is
the **admin**. The software is generic: nothing in the code should assume
"mom" or "family" beyond the docs and default copy.

### The system this replaces

The reference household today uses closed-source automation software on the
recipient's laptop that, at scheduled times, opens Chrome, navigates to a
Meet link, and presses **Enter** (which activates Meet's "Join now" button).
It works reliably, but only the admin can set schedules (via remote desktop),
and the tool can't be integrated with. This project replaces both halves:
members set their own schedules through the web app, and the agent replicates
the proven open-Chrome-and-press-Enter join technique.

Two facts about real usage carried into the design:

- **Calls are weekly.** Each member has a standing weekly time (e.g. "Sundays
  at 2pm"), so recurring weekly slots are a first-class v0.1 feature, not a
  later add-on.
- **Calls have a start time but no end time.** Members stay as long as they
  like and simply leave; Google Meet eventually disconnects the recipient
  once she's the only participant. The scheduler therefore stores no end
  times, and the agent never has to hang up.

---

## 1. Goals and non-goals

### Goals (v0.1)

- Admin manually manages members and assigns each member a Google Meet URL.
- Members sign in (with near-zero friction) and manage their calls: a
  standing weekly slot, one-off calls on specific dates, skipping a week,
  cancelling.
- One shared calendar for the recipient: bookings cannot start too close
  together (configurable minimum gap).
- A kiosk agent on the recipient's Windows machine, driven by Task Scheduler,
  joins the right Meet call at the right time with **zero interaction from
  the recipient** — fully replacing the closed-source automation tool.
- Small, well-tested codebase that a hobbyist can self-host (single server,
  SQLite, no external services required).

### Non-goals (v0.1)

- No Google account integration (OAuth, Calendar API, Meet API). Meet URLs
  are pasted in by the admin. (Roadmap item — see §12.)
- No email/SMS notifications. (Roadmap.)
- No multi-recipient support: one deployment = one recipient = one calendar.
- No real-time UI (websockets); plain server-rendered pages are fine.
- No Linux/macOS agent (the web app runs anywhere; the agent targets Windows
  first because that's where recipients' laptops overwhelmingly are).
- Not a general-purpose booking system (no payments, no public signup).

---

## 2. System overview

Two deployable components sharing one repository:

```
┌────────────────────────┐         ┌───────────────────────────────────┐
│  Scheduler web app     │  HTTPS  │  Kiosk agent (Windows laptop)     │
│  (any host / home      │◄────────┤                                   │
│   server / cheap VPS)  │  poll   │  Task Scheduler tick (per minute) │
│                        │         │   ├─ sync schedule → local cache  │
│  Flask + SQLite        │         │   └─ launch decision              │
│  - admin UI            │         │  joiner: open Chrome on Meet URL, │
│  - member booking UI   │         │  wait for page, press Enter       │
│  - JSON API for agent  │         │  (pluggable strategies)           │
└────────────────────────┘         └───────────────────────────────────┘
```

- **Web app** — source of truth. Stores members, Meet URLs, weekly slots and
  one-off bookings. Serves the booking UI to members and a small
  token-authenticated JSON API to the agent.
- **Kiosk agent** — a small CLI installed on the recipient's laptop. A
  Windows Task Scheduler task runs `kiosk-agent tick` every minute. Each tick
  syncs upcoming call occurrences into a local cache (so a brief server
  outage doesn't cancel a known call) and decides whether to launch the
  joiner. Schedule changes made in the web UI reach the laptop within a
  minute — no more remote desktop sessions to reprogram it.

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
| Agent joiner   | `subprocess` (Chrome) + pywinauto for focus + Enter keystroke | Replicates the proven join technique; Playwright DOM automation as an optional second strategy. |
| Scheduling     | Windows Task Scheduler (`schtasks`) | The Windows counterpart of cron; survives reboots; installable from the CLI. |
| Tests          | pytest + coverage               | Requirement: features land with unit tests.                           |
| Lint/format    | ruff (lint + format)            | One fast tool.                                                        |
| Packaging      | `pyproject.toml`, `pip install .[server]` / `.[agent]` extras | One repo, two install targets. |

Time handling: all timestamps stored/compared in UTC; a single **household
timezone** setting (IANA name, via `zoneinfo`) drives display and weekly-slot
expansion. Weekly slots are defined in household-local time ("Sundays 14:00")
so they stay at 2pm across DST changes; the server converts each concrete
occurrence to UTC before handing it to the agent.

---

## 4. Data model

Two sources of calls — a standing weekly slot and one-off bookings — plus
per-date skips. Concrete **occurrences** are computed on read by expanding
weekly slots over a date range, subtracting skips, and merging in one-offs.
Computing on read (rather than materializing rows ahead of time) means no
background jobs on the server and one obvious source of truth.

```
Member
  id            int PK
  name          str            # display name, e.g. "Aunt Carol"
  email         str, nullable  # optional in v0.1 (used by roadmap notifications)
  role          enum: admin | member
  meet_url      str, nullable  # required for role=member; validated https://meet.google.com/...
  login_token   str unique     # secret for magic-link sign-in (hashed at rest)
  is_active     bool           # soft delete; keeps history intact
  created_at    datetime (UTC)

WeeklySlot                     # "Aunt Carol calls Sundays at 14:00"
  id            int PK
  member_id     FK → Member
  weekday       int 0–6 (Monday=0)
  start_time    time            # household-local
  is_active     bool
  created_at    datetime (UTC)

SlotSkip                       # "…but not on 2026-09-06"
  id            int PK
  weekly_slot_id FK → WeeklySlot
  date          date            # household-local date of the skipped occurrence
  created_at    datetime (UTC)
  unique (weekly_slot_id, date)

Booking                        # one-off call on a specific date
  id            int PK
  member_id     FK → Member
  start_at      datetime (UTC)  # derived from chosen local date + time
  status        enum: scheduled | cancelled
  created_at    datetime (UTC)
  cancelled_at  datetime (UTC), nullable

Setting (one-row table)
  household_timezone   e.g. "America/Chicago"
  day_start / day_end  bookable window, e.g. 09:00–20:00 recipient-local
  min_gap_minutes      default 60   # no call may start within this gap after another starts
  time_step_minutes    default 15   # start times snap to :00/:15/:30/:45
  booking_horizon_days default 60   # how far ahead one-offs can be placed
  recipient_name       display only, e.g. "Grandma June"

AgentDevice
  id            int PK
  name          str            # "Mom's laptop"
  api_token     str unique     # hashed at rest; sent as Bearer token
  last_seen_at  datetime, nullable   # updated on every sync; surfaces dead agents
```

**No end times.** Calls are open-ended: members leave when they're done and
Meet disconnects the recipient once she's alone. Instead of end-time overlap
checks, conflicts are defined by start-time spacing:

> Two occurrences conflict if their start times are less than
> `min_gap_minutes` apart.

Enforced in the service layer inside a transaction, against the union of both
sources: a new weekly slot is checked against all active weekly slots (same
weekday, local-time distance) and against future one-offs; a new one-off is
checked against expanded weekly occurrences and other one-offs in its
vicinity. `min_gap_minutes` is the household's knob for "how long is a long
chat" — it protects the *next* caller's start, not the current caller's
freedom to talk for three hours on a day with nothing after.

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

| Route                          | Purpose                                                                  |
| ------------------------------ | ------------------------------------------------------------------------ |
| `GET /login/<token>`           | Magic-link sign-in → redirect to `/`                                     |
| `GET /`                        | Upcoming-calls view: everyone's occurrences for the next few weeks       |
| `GET /my-schedule`             | My weekly slot + my upcoming one-offs and skips                          |
| `POST /weekly-slot`            | Claim or move my weekly slot (weekday + time)                            |
| `POST /weekly-slot/skip`       | Skip a specific upcoming occurrence ("not this week")                    |
| `POST /bookings`               | Book a one-off call (date + time)                                        |
| `POST /bookings/<id>/cancel`   | Cancel my one-off                                                        |

Booking rules (service layer, all unit-tested):

- Start times snap to `time_step_minutes` and must fall inside the
  `day_start`–`day_end` window; one-offs must be in the future and within
  `booking_horizon_days`.
- The `min_gap_minutes` spacing rule (§4) is checked and committed in one
  transaction.
- v0.1 allows **one weekly slot per member** (the current family's pattern);
  multiple weekly slots per member is a trivial later relaxation.
- Members see everyone's schedule with names (it's a family; transparency is
  a feature) but can only modify their own.
- Skips and cancellations keep their rows for history.

### Admin-facing (`/admin`, role-gated)

- Member CRUD: create/deactivate members, set name and **Meet URL**
  (validated against `https://meet.google.com/…`), regenerate magic links.
- Settings: timezone, daily window, min gap, time step, horizon, recipient
  name.
- Devices: create/revoke agent tokens, see `last_seen_at` health.
- Full schedule view; edit or remove any slot/booking on a member's behalf;
  per-occurrence join telemetry (joined ✓ / failed ✗).

Admins can also be members (have a weekly slot) if they have a `meet_url`.

### UI notes

- Server-rendered, no JS required to book (progressive enhancement only).
- Large tap targets and readable fonts — some members may be elderly too.
- Every page shows times in the household timezone with an explicit label
  ("All times are Grandma June's time") to avoid cross-timezone confusion.

---

## 7. Agent API (JSON, bearer-token auth)

The server does all expansion; the agent only ever sees a flat list of
concrete occurrences.

```
GET /api/v1/occurrences?from=<iso>&to=<iso>
→ 200 {
    "generated_at": "2026-08-18T14:00:00Z",
    "timezone": "America/Chicago",
    "occurrences": [
      {
        "occurrence_id": "w12-2026-08-23",     # stable id: slot/booking + date
        "member_name": "Aunt Carol",
        "meet_url": "https://meet.google.com/abc-defg-hij",
        "start_at": "2026-08-23T19:00:00Z"
      }
    ]
  }

POST /api/v1/occurrences/<occurrence_id>/events    # best-effort telemetry
  body: {"event": "join_started" | "join_succeeded" | "join_failed",
         "at": "<iso>", "detail": "optional string"}
→ 204
```

Notes:

- `meet_url` is resolved server-side from the member — the agent never needs
  the member table.
- Every authenticated call updates `AgentDevice.last_seen_at`.
- The events endpoint feeds the admin's telemetry view; failures of this
  endpoint never block joining.
- API is versioned (`/api/v1/`) because the agent and server are deployed on
  different machines and may skew.

---

## 8. Kiosk agent (Windows)

A console script (`kiosk-agent`) installed on the recipient's Windows laptop
(Python 3.11+ in v0.1; a bundled PyInstaller `.exe` is a v0.2 item so
non-technical admins never touch Python). Configured by a single file,
`%APPDATA%\video-call-scheduler\agent.toml`:

```toml
server_url          = "https://calls.example.com"
api_token           = "…"
join_strategy       = "keystroke"        # or "playwright"
chrome_path         = 'C:\Program Files\Google\Chrome\Application\chrome.exe'
join_early_seconds  = 60                 # open the call this early
late_join_minutes   = 30                 # still join if we wake up this late
page_load_wait_seconds = 20              # keystroke strategy: wait before Enter
cache_path          = '%LOCALAPPDATA%\video-call-scheduler\schedule.json'
log_path            = '%LOCALAPPDATA%\video-call-scheduler\agent.log'
```

### Task Scheduler integration

Installed by `kiosk-agent install-task` (and removed by `uninstall-task`),
which shells out to `schtasks` to create a per-minute task running
`pythonw -m kiosk_agent tick` (the `pythonw` entry point keeps console
windows from flashing on the recipient's screen every minute). The task runs
in the interactive session, because the keystroke strategy needs a visible,
focusable Chrome window.

Setup docs (`docs/kiosk-setup.md`) cover the machine prerequisites — all of
which the current closed-source setup already implies:

- Windows auto-login enabled; screen lock and sleep disabled (screen *off*
  is fine; the session must stay unlocked).
- Chrome's default profile signed into the recipient's Google account, with
  camera + microphone permission granted for `meet.google.com` (visit any
  Meet once during setup and allow them — the grant persists).
- Chrome set as default or referenced by explicit `chrome_path`.

### `tick` algorithm

`tick` must complete in well under a minute; the browser session itself is
spawned as a detached **joiner** process guarded by a lockfile so overlapping
ticks can't double-launch.

1. **Sync:** `GET /api/v1/occurrences` for today ± 1 day; on success,
   atomically overwrite the local cache. On network failure, log and fall
   back to the cached copy (a server blip must not cancel a call the agent
   already knows about).
2. **Decide:** pure function `decide(now, occurrences, joiner_state) → action`:
   - `LAUNCH(occurrence)` — an occurrence satisfies
     `start_at − join_early ≤ now < start_at + late_join` and it hasn't been
     launched yet (the lockfile records the last-launched `occurrence_id`).
     Launching first kills any Chrome left over from a previous call.
   - `NONE` — otherwise.
3. **Act:** spawn the joiner; post telemetry (best-effort).

There is deliberately no `STOP` action: calls have no end time. The member
leaves, Meet times out the recipient, and the leftover browser window sits
harmlessly until the next call's launch cleans it up. (An optional
`cleanup_after_hours` config can close Chrome some hours after a launch, for
households that prefer a tidy screen — off by default.)

Keeping `decide` pure (inputs: clock, occurrence list, lockfile state) makes
the trickiest logic — laptop slept through the start, cancelled occurrence,
same-day second call, tick raced with a running joiner — trivially
unit-testable with a fake clock and no I/O. The late-join rule also gives
graceful wake-from-sleep behavior: waking at 19:10 for a 19:00 call still
joins; waking at 20:00 does not (the member is long gone).

### Joining the Meet: pluggable strategies

The joiner is a small strategy interface with two implementations:

**`keystroke` (default — the proven technique).**

1. Kill any prior Chrome the agent launched.
2. Launch Chrome via `subprocess` on the Meet URL, `--start-fullscreen`,
   using the default (signed-in) profile.
3. Wait `page_load_wait_seconds` for the pre-join screen to render.
4. Focus the Chrome window (pywinauto) and press **Enter** — Meet's pre-join
   screen focuses "Join now"/"Ask to join" by default, which is exactly what
   the household's current automation tool exploits.
5. Report `join_succeeded` (best-effort: the keystroke strategy can't verify
   in-call state, so this event means "sequence completed without error").

**`playwright` (optional, more observable).** Drives Chrome via CDP with a
persistent profile: waits for the actual pre-join DOM, verifies mic/camera
toggles, clicks the join button, and can confirm in-call state before
reporting `join_succeeded`. More robust feedback, but sensitive to Google's
periodic Meet DOM changes — selectors live in one small documented module
(`joiner/selectors.py`), the project's #1 maintenance hotspot. Households
can switch strategies with one config line if either breaks.

Reality checks encoded in the design:

- If a member's Meet link uses "Ask to join", the **member must be in the
  meeting first** to admit the recipient; `join_early_seconds` defaults to
  60 so the recipient is already knocking when the member arrives. Docs
  recommend members open their own link a couple of minutes early.
- If the join sequence fails partway, the agent leaves Chrome open on
  whatever screen it reached — the right Meet page at worst needs the
  member to admit her — and reports `join_failed` so the admin sees it.

---

## 9. Repository layout

```
video-call-scheduler/
├── pyproject.toml            # extras: [server], [agent], [dev]
├── DESIGN.md
├── README.md                 # quickstart for both components
├── docs/
│   ├── server-setup.md
│   └── kiosk-setup.md        # Windows prep: auto-login, no sleep, Chrome profile, install-task
├── src/
│   ├── scheduler/            # web app package
│   │   ├── app.py            # Flask app factory
│   │   ├── models.py
│   │   ├── services/         # occurrence expansion, spacing rule, auth (pure-ish, heavily tested)
│   │   ├── views/            # member, admin, api blueprints
│   │   ├── templates/ static/
│   │   ├── cli.py            # create-admin, init-db, …
│   │   └── migrations/
│   └── kiosk_agent/
│       ├── cli.py            # tick, install-task, join --occurrence (manual test)
│       ├── sync.py           # HTTP client + cache
│       ├── decide.py         # the pure decision function
│       ├── joiner/
│       │   ├── runner.py     # process management, lockfile, Chrome kill/launch
│       │   ├── keystroke.py  # default strategy (pywinauto + Enter)
│       │   ├── playwright_.py# optional strategy
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
integration tests over HTTP. The web app's tests run anywhere; agent tests
isolate the Windows-only bits (pywinauto, `schtasks`) behind interfaces so
the suite passes on Linux CI, with the thin Windows layer exercised by a
`windows-latest` CI job.

- **Occurrence expansion** (highest value): weekly slot × date range across
  DST transitions, skips subtracted, one-offs merged, stable occurrence ids.
- **Spacing rule:** weekly-vs-weekly on the same weekday, weekly-vs-one-off,
  one-off-vs-one-off, exactly-at-gap boundary, cross-midnight windows,
  concurrent-transaction race.
- **Booking rules:** window/horizon/step enforcement, one-weekly-slot-per-
  member, skip/cancel then rebook.
- **Auth:** magic-link happy path, regenerated token invalidates old link,
  deactivated member rejected, CSRF on mutating routes, admin gating.
- **Agent API:** bearer-token auth, expansion window, `last_seen_at`.
- **Agent `decide`:** table-driven with a fake clock — before window, in
  window, past late-join, already-launched (lockfile), occurrence cancelled
  between ticks, two calls same day, slept-through-start.
- **Agent `sync`:** server down → cache used; malformed response → cache
  kept; atomic cache write.
- **Joiner:** strategies behind an interface; unit tests mock pywinauto/
  subprocess and assert the sequence (kill → launch → wait → focus → Enter).
  `kiosk-agent join --occurrence <id>` exists for manual smoke tests on the
  real laptop; the actual keystroke landing in the real Meet UI is verified
  manually, not in CI.
- Tooling: `pytest`, `pytest-cov` (target ≥90% on `services/` and
  `kiosk_agent/decide.py`), injected clock (or freezegun), `ruff` in CI
  (GitHub Actions) on 3.11/3.12, Linux + Windows jobs.

---

## 11. Security & privacy

- Meet URLs are effectively capabilities — the app treats them as secrets:
  visible only to their member, the admin, and the agent API. Never logged.
- All tokens (login, agent) ≥ 32 random bytes, stored hashed.
- HTTPS required in deployment docs; cookies `Secure`, `HttpOnly`, `SameSite=Lax`.
- The kiosk laptop auto-logs-in and stays unlocked — that's inherent to the
  use case (and to the current setup). Setup docs spell out the resulting
  physical-access tradeoff and recommend disk encryption and a
  low-privilege Windows account for the recipient.
- SQLite file and agent config contain secrets → documented file
  permissions/ACLs.
- No analytics, no third-party calls from the web app. Data never leaves the
  self-hosted server except to the agent.

---

## 12. Roadmap

| Release | Theme                                                                                                     |
| ------- | --------------------------------------------------------------------------------------------------------- |
| **v0.1** | Everything in this document: member/admin UI, weekly slots + one-offs + skips, Windows agent with keystroke join. Definition of done: the reference family retires the closed-source tool — members manage their own weekly calls, and a real call joins end-to-end with no keyboard on the recipient's side. |
| v0.2    | Packaged agent installer (PyInstaller `.exe` + guided setup) so non-technical admins never touch Python; email notifications (booking confirmations, reminders, "join failed" admin alerts) with ICS attachments. |
| v0.3    | Quality of life: multiple weekly slots per member, per-member booking limits, agent health dashboard, `cleanup_after_hours` polish. |
| v0.4    | Google integration: OAuth + Calendar API to auto-create Meet links per occurrence (removes manual URL config; per-member static URLs remain as fallback). |
| v0.5    | Linux/macOS agent (cron/launchd, X11/AppleScript or Playwright-only join); optional Postgres.             |
| Later   | Multi-recipient households; i18n; pluggable meeting providers (Jitsi, Zoom) behind a provider interface.  |

Each release lands feature-by-feature behind the existing test suite; the
provider-agnostic seam around `meet_url` and the joiner strategies is why
that code stays isolated from the scheduling core from day one.

---

## 13. Open questions (decide before implementation)

1. **`min_gap_minutes` default** — 60 assumes at most one call per day per
   sibling-cluster; with weekly cadence gaps rarely bind. Right default, or
   larger (e.g. 120) to be safe out of the box?
2. **Member visibility** — members see who holds each slot. Any preference
   for anonymized "taken"? Could be a setting later.
3. **Keystroke `join_succeeded` semantics** — the default strategy can't
   verify in-call state, only that the sequence ran. Is "sequence completed"
   good enough for v0.1 telemetry, or should the Playwright strategy be the
   default where installed?
4. **Static Meet URL reuse** — a member's link is the same every call.
   Acceptable for v0.1 (family-trust model, matches current practice);
   v0.4's Calendar integration replaces it with per-call links.
