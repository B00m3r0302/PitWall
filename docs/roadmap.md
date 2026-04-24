# Pit Wall — Build Roadmap

A live F1 race companion. A Go backend ingests timing/telemetry from the OpenF1 API, analyzes it in real time, detects "storylines" (undercut windows, closing gaps, tire-age warnings, pit stops, DRS trains, fastest laps), and pushes them to a React/TypeScript dashboard over a websocket.

This doc is your map for the next 6–8 weeks. Read it end-to-end once before you write a line of code. Come back to each phase as you start it.

---

## 1. What you're actually building

**One sentence:** A browser dashboard that tells an F1 fan what to pay attention to right now, backed by a Go service that watches the race and writes a stream of insights.

**Three surfaces:**

1. **Ingestion + analytics service (Go).** Polls OpenF1, keeps an in-memory model of the current session (positions, gaps, stints, tires, laps), persists events to Postgres, and runs a "storyline detector" over the live state.
2. **Real-time API (Go).** A websocket stream of state updates and storylines, plus REST endpoints for race metadata, past sessions, and replays.
3. **Dashboard (React + TypeScript).** Timing board, storyline feed, small charts, tire-strategy visualization. A single-page app you open in a tab next to the stream.

**Definition of done for v1:** During a live session, a first-time user can open the page, immediately understand the race order, and within 60 seconds see a storyline card that teaches them something they wouldn't have noticed from the broadcast alone.

**Non-goals for v1:** user accounts, friend features, push notifications, mobile app, predictions, betting, fantasy integration. Pick these up as stretch goals (Section 16) after the core works.

---

## 2. Stack decisions (with rationale)

Every choice below is deliberate. You don't need the fanciest tool; you need tools that keep you moving.

| Layer | Choice | Why |
|---|---|---|
| Language (backend) | Go 1.22+ | Boot.dev taught it; perfect for concurrent polling + streaming. |
| HTTP router | `net/http` stdlib (Go 1.22 has pattern matching) + optionally `chi` | Stdlib is enough. `chi` adds middleware ergonomics if you want them. |
| Database | PostgreSQL | Matches boot.dev, real-world relevant, handles time-series OK at this scale. |
| DB access | `pgx/v5` + `sqlc` | Type-safe queries, zero-magic SQL. Boot.dev path. |
| Migrations | `goose` or `golang-migrate` | Either works; pick one and stick. |
| Websockets | `github.com/coder/websocket` | Actively maintained, modern API. Avoid the unmaintained `gorilla/websocket`. |
| Config | `envconfig` or plain `os.Getenv` + a typed struct | Keep it simple. |
| Logging | `log/slog` (stdlib) | Structured logging, no deps. |
| Testing | Stdlib `testing` + `testify` if you want asserts | Stdlib is enough. |
| Frontend | React + TypeScript + Vite | Smallest "new-to-React" cliff. Vite is fast. |
| Styling | Tailwind CSS | You skip writing CSS; utility classes only. |
| Data fetching | TanStack Query (React Query) | De-facto standard; handles caching, retries, loading states. |
| Charts | Recharts | Easiest React charting lib. Switch to `visx`/`d3` if you hit ceilings. |
| Client state | React built-in + Context for MVP; Zustand if it grows | Don't add Redux. |
| Deploy (backend) | Fly.io | Supports long-lived websockets cleanly, generous free tier, nice Go story. |
| Deploy (frontend) | Vercel | Zero-config for Vite, instant. |
| Monorepo layout | Two top-level dirs: `backend/` and `web/` | Deploy independently. |

**Things not to do:**

- Don't pick Next.js "just because." It adds SSR concepts you don't need for a live-data dashboard.
- Don't reach for Redux. React Query + a small Zustand store is plenty.
- Don't use an ORM (GORM/Ent). You'll learn more, write less code, and ship faster with sqlc.
- Don't introduce gRPC, Kafka, or Redis in v1. You have one process talking to one database.

---

## 3. External data sources — deep dive

Your entire product lives or dies on knowing this section.

### 3.1 OpenF1 (https://openf1.org)

Free, no auth, REST, JSON. Updates about every 3–4 seconds during live sessions. Historical data from 2023 onward. **This is your primary source.**

Endpoints you will actually use:

- `/v1/meetings` — race weekends (a "meeting" = Bahrain GP 2025).
- `/v1/sessions` — sessions within a meeting (FP1, FP2, FP3, Qualifying, Race, Sprint). Each has a `session_key`. Everything below filters by session.
- `/v1/drivers` — driver metadata for a session (numbers, team, colors, headshot URLs).
- `/v1/position` — running order by lap.
- `/v1/intervals` — live gap-to-leader and gap-to-ahead. **Critical.**
- `/v1/laps` — lap times, sector times, compound, is_pit_out_lap.
- `/v1/pit` — pit stops (duration, lap).
- `/v1/stints` — tire stints per driver (compound, start lap, end lap, tyre age at start).
- `/v1/car_data` — high-frequency telemetry (speed, throttle, brake, gear, DRS). Heavy. Use sparingly.
- `/v1/race_control` — flags, safety cars, investigations, penalties. **Critical for storylines.**
- `/v1/weather` — track/air temp, humidity, rainfall.

All endpoints take `?session_key=latest` to get the current live session, or a specific key for historical.

**Gotchas:**

- It's a best-effort community project. Expect occasional missing points, out-of-order timestamps, and backfills.
- Rate limits aren't formally published. Be polite — poll each endpoint every 2–5 seconds, not every 200ms.
- Timestamps are ISO-8601 UTC. Your DB columns should be `timestamptz`.
- `car_data` is huge. A full race for 20 drivers is millions of rows. Only pull it when you need telemetry for a specific feature.

### 3.2 Jolpica-F1 (https://api.jolpi.ca/ergast/f1)

Free, no auth, drop-in replacement for the old Ergast API. Use for:

- The season schedule (Jolpica is the clean source for "when is the next race").
- Circuit metadata (length, number of laps, historical winners).
- Historical results pre-2023 if you ever want to go back in time.

Update cadence: weekly. Not live.

### 3.3 Scraping / other

Don't scrape Formula1.com. It's a pain, their TOS is touchy, and you don't need it. If you want driver headshots that OpenF1 doesn't provide, use Wikipedia (via the MediaWiki API) or Wikidata.

---

## 4. Architecture at 10,000 feet

```
                     +--------------------------+
                     |  OpenF1 API (REST/JSON)  |
                     +------------+-------------+
                                  | polled every 2–5s
                                  v
  +-------------------------------+-------------------------------+
  |                         Go backend                            |
  |                                                               |
  |  [ Pollers ] --chan--> [ Ingestor ] --> [ State ] + [ Postgres]
  |                                           |                   |
  |                                           v                   |
  |                                    [ Analytics ]              |
  |                                           |                   |
  |                                           v                   |
  |                                    [ Storyline Engine ]       |
  |                                           |                   |
  |                                           v                   |
  |                   [ Websocket hub ] <-- [ Event bus ]         |
  |                          |                                    |
  |                          |        [ REST handlers ]           |
  +--------------------------+-----------------+------------------+
                             |                 |
                       websocket            HTTP GET
                             |                 |
                     +-------+-----------------+--------+
                     |     React + TypeScript SPA       |
                     |  (timing board, feed, charts)    |
                     +----------------------------------+
```

Key shape to internalize: **one long-running Go process, many goroutines, one database, one websocket hub.** The event bus inside the backend is just a set of Go channels; you're not introducing Kafka or Redis pub/sub.

---

## 5. Phase 0 — Foundation (Days 1–3)

Before any feature code.

1. **Explore OpenF1 by hand.** Use `curl` or Postman/Bruno/Hoppscotch against every endpoint in Section 3.1 with `session_key=latest` during free practice or qualifying, and with a specific historical `session_key` from the 2024 season. Save example JSON responses into a `docs/samples/` folder. You'll reference these constantly.
2. **Create the repo.** Monorepo layout:
   ```
   pit-wall/
     backend/
       cmd/server/
       internal/
         openf1/         # HTTP client for the API
         ingest/         # pollers
         state/          # in-memory session model
         analytics/      # gap/pace/tire math
         storyline/      # detectors
         ws/             # websocket hub
         api/            # HTTP handlers
         store/          # sqlc-generated
       db/
         migrations/
         queries/
     web/
       src/
       public/
     docs/
     docker-compose.yml  # just Postgres for local dev
     README.md
   ```
3. **Stand up Postgres locally** via `docker-compose`. Write a single migration that creates a `sessions` table. Connect from Go, insert a row, read it back. This kills 90% of "is my env broken?" questions.
4. **Decide on commit cadence.** Commit at least daily. Write short, imperative commit messages. Your future self (and anyone reviewing) will care.
5. **Write a one-page design doc in `docs/design.md`** describing the architecture in your own words. This is the single best predictor of whether you'll actually finish.

---

## 6. Phase 1 — Ingestion backbone (Week 1)

**Goal by end of week:** A single Go binary that connects to OpenF1, polls the key endpoints for the current or a chosen historical session, writes rows to Postgres, and logs what it's doing. No API, no frontend.

**Work breakdown:**

1. **`openf1` package.** A thin client that wraps OpenF1's REST endpoints. Methods return typed structs. One shared `http.Client` with a sane timeout. No retries yet — just log errors.
2. **Schema v1.** Migrations for:
   - `sessions` (session_key PK, meeting_key, name, type, start, end, circuit)
   - `drivers` (session_key, driver_number, name, team, color)
   - `laps` (session_key, driver_number, lap_number, lap_time, sector_1, sector_2, sector_3, compound, timestamp)
   - `positions` (session_key, driver_number, position, timestamp)
   - `intervals` (session_key, driver_number, gap_to_leader, interval_to_ahead, timestamp)
   - `pit_stops` (session_key, driver_number, lap, duration)
   - `stints` (session_key, driver_number, stint_number, compound, start_lap, end_lap, tyre_age_start)
   - `race_control` (session_key, timestamp, category, message, flag, driver_number, lap)
   - `raw_events` (session_key, source, payload jsonb, timestamp) — a catch-all for debugging.
   Index every table on `(session_key, timestamp)` and on the relevant foreign keys.
3. **Pollers.** One goroutine per endpoint, each on its own interval (2s for intervals, 5s for laps/pits/race_control, 10s for weather). Each polls, diffs against "what did we already store?" (use OpenF1's timestamps), and inserts new rows only. Communicate via channels to a central ingestor.
4. **Graceful shutdown.** Use `context.Context` threaded through every goroutine. `os/signal.Notify` on `SIGINT`/`SIGTERM`. All pollers must stop within a few seconds.
5. **Rate limiting yourself.** A single `time.Ticker` per poller. Jitter the start times by a few hundred ms so you don't hammer OpenF1 in synchronized waves.
6. **Structured logging.** `slog` with JSON output. Include `session_key` and endpoint in every log line.

**Acceptance test for Phase 1:** Run the service during a practice session (or point it at a 2024 race session_key). After 10 minutes, Postgres has thousands of intervals, dozens of laps per driver, and a complete driver list. No panics, no leaked goroutines, clean shutdown on Ctrl-C.

---

## 7. Phase 2 — Historical replay engine (Week 2, first half)

**This is the most important insight in this whole doc.** You cannot develop against live races only. A race is 90 minutes a week, maybe. You need a way to replay stored sessions at normal or accelerated speed so you can iterate on analytics any time.

**Work:**

1. **Backfill mode.** A CLI flag `--backfill --session-key=9158` pulls the *entire* session's data from OpenF1 in one pass and writes everything to Postgres. Rate-limit yourself (sleep between pages) to be polite. Pick 2–3 recent races with varied storylines (a wet race, a 1-stop strategy race, a chaotic safety-car race) and backfill them.
2. **Replay mode.** A CLI flag `--replay --session-key=9158 --speed=5x` reads rows from Postgres in timestamp order and emits them onto the same internal channels that live pollers use. From the analytics engine's perspective, replay is indistinguishable from live. This is gold.
3. **Virtual clock.** A `Clock` interface with `Real` and `Virtual` implementations. Everything downstream uses `clock.Now()`, never `time.Now()` directly. The virtual clock advances as replay events flow. Time-based analytics (gap deltas per minute, tire age) will break subtly otherwise.

After Phase 2 you can develop and test every remaining feature offline, during the off-season, on a plane.

---

## 8. Phase 3 — Analytics engine (Week 2, second half)

The math layer. Pure functions over snapshots of state. No I/O, easy to unit-test.

**Core computations:**

1. **Live gaps.** You already get gap-to-leader and interval-to-ahead from OpenF1, but you'll want derived metrics: gap trend over last N laps (positive = closing, negative = opening), gap velocity in seconds per lap.
2. **Pace.** Rolling mean/median of last 3 clean laps per driver. Exclude in-laps, out-laps, and laps under safety car. "Clean" means no yellow flag in any sector that lap.
3. **Tire age and degradation.** Age = current lap − stint start lap. Per-compound degradation curve fit from clean laps in this session (simple linear regression, not ML). Compare observed pace to fitted curve to flag "tires are falling off a cliff."
4. **Undercut/overcut windows.** Given driver A in front of B: if B pits now and emerges N seconds behind where A is, will A's tire-age-adjusted pace over the next 2 laps let A stay ahead when A pits? This is a fun one to model. Pit-loss time is roughly constant per track (~22s at most circuits; look it up per venue).
5. **Projected catch lap.** If gap is closing at 0.3s/lap and current gap is 2.4s, projected pass around 8 laps from now (± confidence interval based on pace variance).
6. **DRS trains.** Detect 3+ cars within 1.0s of each other for 2+ consecutive laps — classic DRS train where nobody can overtake.
7. **Session state.** Green / yellow / VSC / SC / red flag / checkered. Derived from race_control messages.

**Testing.** Write table-driven unit tests with hand-crafted state snapshots. For each analytic, test 3–5 edge cases (empty data, single driver, driver just pitted, safety car just deployed).

---

## 9. Phase 4 — Storyline engine (Week 3)

The brain. Turns raw analytics into human-readable cards.

**Shape of a storyline:**

```
type Storyline struct {
  ID          string    // stable, deduplicatable
  Kind        string    // "undercut_window", "closing_gap", "tire_cliff", ...
  Priority    int       // 1–10, for UI ranking
  Headline    string    // "Norris closing on Verstappen at 0.3s/lap"
  Detail      string    // one-paragraph explanation
  DriverNums  []int     // which drivers this involves
  CreatedAt   time.Time
  ExpiresAt   time.Time // auto-dismiss after this
  Payload     json.RawMessage // structured data for charts
}
```

**Starter storyline types (implement all for v1):**

1. `pit_stop` — someone pitted. Duration, compound on/off, position before/after.
2. `closing_gap` — gap between two adjacent drivers shrinking consistently for 3+ laps, projected pass.
3. `undercut_window_open` — conditions right for a driver to undercut the car ahead.
4. `tire_cliff` — a driver's laptimes falling off vs. their compound's deg curve.
5. `fastest_lap` — new session-fastest lap.
6. `overtake` — position change during green flag running.
7. `drs_train` — described above.
8. `flag_change` — yellow/VSC/SC/red activated or cleared.
9. `penalty_applied` — time penalty, grid drop, etc., from race_control.
10. `weather_change` — rainfall started/stopped, track temp swing, wind shift.

**Detection pattern.** Each detector is a small struct with a `Detect(snapshot State) []Storyline` method. A scheduler runs them after every state update. Emit to the event bus. A deduplication layer filters by `ID` so the same storyline doesn't fire every poll.

**Prioritization.** Simple scoring: flag changes > overtakes at front > pit stops > closing gaps in midfield. Let the UI sort by priority + recency.

**Quality bar.** Tune thresholds against replay data until storylines feel right. Too many and the feed is noise; too few and the product feels dead. Target: roughly one storyline every 30–90 seconds during a typical race.

---

## 10. Phase 5 — HTTP API and websockets (Week 4, first half)

Everything so far ran in a single process with no outside interface. Now expose it.

**REST endpoints (JSON):**

- `GET /api/sessions` — upcoming + recent sessions.
- `GET /api/sessions/:key` — session detail + drivers.
- `GET /api/sessions/:key/timing` — latest timing snapshot (fallback if websocket fails).
- `GET /api/sessions/:key/storylines?since=...` — storyline history.
- `GET /api/sessions/:key/laps?driver=...` — lap history for a driver.
- `GET /api/health` — trivial 200 OK.

**Websocket:**

- `GET /ws?session_key=...` upgrades to websocket.
- Server sends JSON messages with a `type` field: `timing_update`, `storyline`, `race_control`, `session_state`.
- Client sends pings every 30s; server drops connections with no pong in 60s.
- On connect, server sends a `snapshot` message with full current state, then deltas thereafter. Simplifies the client.

**Websocket hub.** A single goroutine owning a `map[clientID]chan Message`. Subscriptions are scoped by `session_key`. When a new event lands on the event bus, the hub fans it out to interested clients. Slow clients get their channel filled and are dropped — don't block the hub.

**Concerns to handle:**

- **CORS.** Allow your frontend origin. `rs/cors` or write it yourself — three headers.
- **Auth.** Skip for v1; add a simple API key later if you expose anything sensitive.
- **Backpressure.** Bounded channels everywhere. Measure.
- **Reconnection.** The client will drop and reconnect. Give it an idempotent way to resume (the snapshot-on-connect pattern handles this).

---

## 11. Phase 6 — React learning on-ramp (Week 4, second half)

If you're new to React/TS, don't try to learn it while also implementing features. Spend 2–3 focused days on fundamentals using a toy project, then come back.

**Minimum viable React/TS knowledge for this project:**

1. **TypeScript basics.** Types vs interfaces, generics, `unknown` vs `any`, discriminated unions (you'll use these for websocket messages). Skim the TS handbook's "everyday types" section.
2. **React fundamentals.** Function components, `useState`, `useEffect`, `useMemo`, `useCallback`, props, children, conditional rendering, lists and keys. The React docs' "Learn" section is the single best resource; do the interactive tutorial.
3. **React Query.** Read the first few pages of the TanStack Query docs. Understand `useQuery` (GET) and cache invalidation. You won't need `useMutation` until you add auth/user features.
4. **Tailwind.** Read their "Utility-First Fundamentals" page. Install the VS Code extension — autocomplete makes it trivial.
5. **Vite + Router.** `npm create vite@latest` with React + TS template. React Router is overkill for v1 (you have basically one page); skip it and add later if needed.
6. **Websocket clients.** Plain `WebSocket` API is fine. Wrap it in a small hook.

Pace yourself. Build a toy counter, a todo list, a fetch-a-joke-API page. Then come back.

---

## 12. Phase 7 — Frontend implementation (Weeks 5–6)

**Project setup (day 1):** `npm create vite@latest web -- --template react-ts`. Install `tailwindcss`, `@tanstack/react-query`, `recharts`, `clsx`. Wire Tailwind per their Vite guide. Configure dev server proxy so `/api` and `/ws` hit your local Go backend.

**Build order (don't skip — each unlocks the next):**

1. **Layout skeleton.** Header with session name and state, two-pane layout: timing board left (60%), storyline feed right (40%). Stub with hardcoded data. Get it looking OK on a 1440p monitor; worry about mobile in polish phase.
2. **Fetch and render the driver list.** Use React Query to hit `/api/sessions/:key`. Render rows for each driver with team color stripe, number, name, team. No interactivity yet.
3. **Websocket hook.** A custom `useSessionStream(sessionKey)` hook that opens a websocket, handles the initial snapshot, updates a local state store as deltas arrive, and exposes `{ timing, storylines, sessionState }`. Reconnects with exponential backoff.
4. **Live timing board.** Rows sorted by position, showing position, driver, gap to leader, interval, last lap time (color-coded: purple=personal best, green=improving, yellow=worse), tire compound icon + age, pit count. Update in place on websocket deltas; don't remount rows or you'll get re-render jank.
5. **Storyline feed.** A vertically scrolling list. New cards slide in at top. Each card has an icon (by kind), headline, optional expanded detail, timestamp. Auto-expire after `ExpiresAt`. Keep the last 100 in view; virtualize if it gets slow.
6. **Session state banner.** A top bar that changes color on flag state: green during green flag, yellow during yellow, striped yellow-and-red during VSC, full red during SC, red during red flag. Big visual anchor for users.
7. **Tire strategy strip.** For each driver, a horizontal bar showing their stints (colored by compound) from lap 1 to current lap. Dense, glanceable.
8. **Lap-time chart.** When a storyline card is expanded, show a Recharts line chart of the relevant drivers' last-20-lap times. This is where your `Payload` JSON from storylines earns its keep.
9. **Session picker.** A dropdown of recent sessions. Selecting one opens a replay view. This lets you demo the app without needing a live race.

**UI quality checklist** (run through before calling it done):
- Nothing jumps or flashes on update. Every change should feel smooth.
- Loading states for initial fetches. Error states for websocket disconnects.
- Reasonable empty states ("No session active — try a replay").
- Keyboard focus visible. Color contrast checks. Don't encode info by color alone.
- Responsive down to ~768px (tablet). Mobile is a stretch goal.

---

## 13. Phase 8 — Replay mode UI (Week 6, end)

You already built replay on the backend. Now expose it.

- Session picker offers any session in the DB (live or historical).
- When replaying, show a "REPLAY" badge and playback controls: play/pause, speed (1x / 2x / 5x / 20x), jump to lap N.
- Replay state is held server-side (one replay goroutine per replay session) with a unique replay session ID, so multiple users can replay different sessions independently. Broadcast only to subscribers of that replay ID.
- This doubles as your demo mode. For your portfolio, record a 90-second video of a well-chosen race replayed at 10x with storylines firing.

---

## 14. Phase 9 — Polish, deploy, observe (Weeks 7–8)

**Deploy the backend to Fly.io:**

1. `fly launch` in `backend/`. Accept the generated Dockerfile, tweak if needed.
2. Provision Fly Postgres: `fly postgres create`. Attach: `fly postgres attach`.
3. Run migrations on deploy (a `release_command` in `fly.toml` that runs your migration binary).
4. Set secrets: `fly secrets set OPENF1_BASE_URL=...` etc.
5. Fly handles websockets cleanly. Turn on autostop/autostart to save free-tier hours.

**Deploy the frontend to Vercel:**

1. Connect the repo. Set root directory to `web/`. Vercel detects Vite.
2. Environment variables: `VITE_API_BASE` pointing at your Fly app, `VITE_WS_BASE` same with `wss://`.
3. Add a custom domain if you have one. HTTPS is automatic.

**Observability:**

- `slog` JSON logs to stdout; Fly captures them. For anything more, add `grafana/loki` later.
- Metrics: expose `/metrics` with `prometheus/client_golang` — poll durations, events per second, websocket connections, storyline counts by kind. You won't set up a Prometheus server in v1 but the endpoint is useful for ad-hoc curling.
- One health dashboard page on the frontend (`/admin`) that reads `/metrics` and renders a few numbers. Skip if tight on time.

**Performance sanity:**

- Backend should idle around 50–100 MB RAM. If it's growing monotonically you have a goroutine leak — check with `/debug/pprof`.
- Frontend should hit 60fps during updates. React DevTools profiler, look for unnecessary re-renders. Memoize row components.
- DB: after a full season of data, queries on `(session_key, timestamp)` indexes should stay fast. If you see slowness, `EXPLAIN ANALYZE` first.

**README and demo:**

- Screenshot at the top. GIF of replay mode second. Paragraph on what it is. Paragraph on why. Local-dev setup (docker-compose up + 3 commands). Architecture diagram (you already have one, Section 4). Credits to OpenF1 and Jolpica.
- Demo video, 60–90 seconds, screen-recorded. Put it on YouTube unlisted and embed it.

---

## 15. Pitfalls to avoid

1. **Building against live races only.** Phase 2 exists for a reason. Don't skip it.
2. **Perfecting the storyline engine before anything else.** Ship 3 detectors and iterate. You'll discover which storylines matter only by watching the feed during a replay.
3. **Over-engineering the event bus.** Go channels are the entire event bus. No Redis, no NATS.
4. **Trying to get every telemetry feature in v1.** Leave telemetry overlays for v2. Gaps, pace, and tires are 90% of the value.
5. **Letting the DB become the bottleneck.** Batch inserts (copy-from for high-volume endpoints). Don't insert one row at a time inside a hot loop.
6. **Not shutting down cleanly.** If Ctrl-C leaves pollers running, you have a real bug. Use contexts rigorously.
7. **Forgetting time zones.** Store UTC everywhere. Convert on the frontend using the browser's locale.
8. **Treating OpenF1 as a contract.** It's a community project. Your client should tolerate missing fields, null values, and brief outages. Wrap parses in defensive code.
9. **Silent failures.** Every poll failure should log with context. Silent drops are the hardest bug to diagnose later.
10. **Scope creep.** Auth, friends, comments, fantasy — all tempting. All v2.

---

## 16. Stretch goals (only after v1 ships)

In rough difficulty order:

- **Session notifications.** Email or web-push when the next session starts.
- **User accounts + saved preferences.** Followed drivers get higher-priority storylines.
- **Per-driver focus mode.** Click a driver → feed filters to storylines involving them.
- **Telemetry overlay.** For a specific lap, show speed/throttle/brake traces over a track map. Pull `car_data`. Recharts or visx will do.
- **Predictions.** Let users guess podium, pole, fastest lap before the session; score after. This is your bridge to the "Paddock Picks" idea if you ever want to merge them.
- **Shareable storyline clips.** Export a storyline card as an image for Twitter/Bluesky.
- **Mobile-first layout.** Whole separate design pass.
- **Race summary auto-generated post-session.** A page with "the 8 things you missed" from the storyline feed, ranked.

---

## 17. Week-by-week calendar

A concrete pace. Each "week" is ~10–15 focused hours.

| Week | Focus | Deliverable |
|---|---|---|
| 1 | Phase 0 + Phase 1 | Go binary polling OpenF1 into Postgres |
| 2 | Phase 2 + Phase 3 | Replay mode works; analytics engine computes gaps/pace/tires |
| 3 | Phase 4 | Storyline engine producing 5+ kinds of storylines against replay |
| 4 | Phase 5 + Phase 6 | REST + WS API done; React fundamentals learned on toy app |
| 5 | Phase 7 (first half) | Timing board + storyline feed live against your backend |
| 6 | Phase 7 (second half) + Phase 8 | Charts, tire strategy, replay UI, polish |
| 7 | Phase 9 | Deployed to Fly + Vercel, observability, README |
| 8 | Buffer / stretch | Demo video, one stretch goal, final polish |

If you're running hot: compress to 6 weeks by trimming stretch goals.
If you're running cold: extend to 10 weeks. Don't skip Phase 2 or Phase 9 — they're what separate "works on my machine" from "portfolio piece."

---

## 18. Resources

**Go:**
- `pgx` docs — the Postgres driver you'll use.
- `sqlc` docs — generates Go from SQL.
- `coder/websocket` README — short and sufficient.
- `slog` package docs — structured logging.
- Ardan Labs "Ultimate Go" video series if you want deeper Go mental models.

**F1 data:**
- OpenF1 docs at openf1.org — read every endpoint page.
- Jolpica-F1 at jolpi.ca — skim their endpoint list.
- FastF1 (Python lib) docs — great inspiration for analytics ideas even though you won't use it.
- r/F1Technical on Reddit — where strategy nerds hang out; good for idea-gathering.

**React/TypeScript:**
- react.dev "Learn" — do the interactive tutorial.
- typescriptlang.org handbook — "Everyday Types" + "Narrowing".
- TanStack Query docs — "Overview" + "Important Defaults".
- Tailwind docs — utility-first fundamentals + the interactive playground.

**Charting and viz:**
- Recharts docs and examples.
- visx examples if you outgrow Recharts.

**Deployment:**
- Fly.io docs on Go apps and Postgres.
- Vercel docs on Vite.

---

## 19. First three commits you should make

A concrete kickoff so you're not staring at a blank page.

1. `chore: initial monorepo layout with README` — folders from Section 5, an empty Go module, an empty Vite app.
2. `feat(backend): minimal openf1 client with Sessions and Drivers` — stdlib `net/http`, typed structs, a small test hitting the real API.
3. `feat(backend): docker-compose postgres + goose migrations + first schema` — sessions and drivers tables, a `cmd/migrate` binary, a `cmd/server` that connects and logs "ready".

After that you're in Phase 1 proper.

---

**Final thought.** The hard part of this project isn't any single technical piece — it's carrying a 6–8 week build to done without losing steam. The structural things that help most: the replay engine (you can work any time), the early end-to-end deploy (real URL early keeps morale up — do it at end of Week 4 if you can), and a visible changelog in your README that you update every commit. Make the project feel real to yourself from day one.
