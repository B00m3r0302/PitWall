# Architecture

This document explains how Pit Wall is put together: the components, the flow of data, and why each piece exists. Pair it with the [roadmap](docs/roadmap.md), which describes *when* to build each part.

---

## 1. System diagram

```
                     +--------------------------+
                     |  OpenF1 API (REST/JSON)  |
                     |     (openf1.org)         |
                     +------------+-------------+
                                  | polled every 2-5s
                                  v
  +-------------------------------+-------------------------------+
  |                         Go backend                            |
  |                                                               |
  |  [ Pollers ] --chan--> [ Ingestor ] --> [ State ] + [Postgres]|
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

One long-running Go process. Many goroutines. One database. One websocket hub. No external message broker, no cache server, no microservices. The "event bus" is a set of Go channels.

## 2. Components

### 2.1 OpenF1 client (`internal/openf1`)

A thin wrapper over OpenF1's REST endpoints. Each method returns typed Go structs. Uses a single shared `http.Client` with a 10-second timeout. Tolerant of missing fields and null values — OpenF1 is a best-effort community project and its payloads shift.

Responsibility: speak HTTP, return typed data. No business logic.

### 2.2 Pollers (`internal/ingest`)

One goroutine per OpenF1 endpoint we care about, each on its own ticker:

- intervals: every 2s
- laps, pits, race_control: every 5s
- stints: every 10s
- weather: every 30s
- car_data: only on demand for specific features (it's huge)

Each poller diffs new data against what's already been ingested (by OpenF1's timestamp) and pushes only new records onto an in-process channel. Tickers are jittered at start so they don't fire in synchronized waves.

### 2.3 Ingestor

A single goroutine that reads from every poller's channel, writes new rows to Postgres in batches, and updates the in-memory state model. Batches are small (100 rows or 500ms, whichever comes first) to keep latency low.

### 2.4 State model (`internal/state`)

A struct representing the current session: drivers, their latest positions, current laps, stints, tires, race-control flag state. Held in memory for fast reads by analytics and the websocket hub. Persisted to Postgres so a restart doesn't lose the session.

Access is guarded by a `sync.RWMutex`. All mutations go through narrow setters that emit events onto the event bus.

### 2.5 Analytics (`internal/analytics`)

Pure functions over state snapshots. No I/O, trivially unit-testable. Produces derived metrics:

- Gap trend (closing or opening over N laps, in s/lap)
- Pace (rolling median of last 3 clean laps per driver)
- Tire degradation curve per compound, fit from clean laps this session
- Undercut window evaluation
- Projected catch lap with confidence interval
- DRS train detection
- Clean-lap filter (excludes in/out laps, laps under yellow or SC)

See [docs/storylines.md](docs/storylines.md) for how these feed the storyline engine.

### 2.6 Storyline engine (`internal/storyline`)

A set of small detector structs, each with a `Detect(snapshot) []Storyline` method. The engine runs all detectors after every state update, deduplicates by stable storyline ID, and emits new storylines onto the event bus.

Storylines carry a priority (1–10), a headline, a detail paragraph, involved drivers, and a structured payload the frontend can use to draw mini-charts. Each has an `ExpiresAt` so the UI can auto-dismiss stale cards.

### 2.7 Event bus

A single in-process channel typed as `chan Event`. Every interesting thing that happens — a position change, a pit stop, a new storyline, a flag state change — becomes an event. Consumers (currently: the websocket hub) subscribe via fan-out.

Why an event bus at all? It keeps the websocket hub decoupled from where events originate. Today there's one consumer; tomorrow you might add a notification service or a metrics recorder without threading them through every detector.

### 2.8 Websocket hub (`internal/ws`)

Owns a `map[clientID]*client`, each client has a bounded send channel. When an event lands on the event bus, the hub looks up which clients are subscribed to that session and enqueues the message. Slow clients with full channels are dropped — never block the hub.

Message shape:

```
{ "type": "storyline" | "timing_update" | "race_control" | "session_state" | "snapshot",
  "session_key": 9158,
  "data": { ... } }
```

On connect, the hub sends a `snapshot` containing full current state, then deltas. This makes reconnection trivial for the client.

### 2.9 REST handlers (`internal/api`)

Thin HTTP handlers for:

- `GET /api/sessions` — upcoming + recent
- `GET /api/sessions/:key` — detail + drivers
- `GET /api/sessions/:key/timing` — latest snapshot (fallback for clients that can't use websockets)
- `GET /api/sessions/:key/storylines?since=...` — history
- `GET /api/sessions/:key/laps?driver=...` — lap history
- `GET /api/health` — liveness

All JSON. CORS allows the frontend origin. No auth in v1.

### 2.10 Persistence (`internal/store`)

Generated by [sqlc](https://sqlc.dev) from SQL files in `db/queries/`. No ORM. Every query is explicit SQL you can read and `EXPLAIN ANALYZE`.

Schema highlights:

- `sessions`, `drivers`, `laps`, `positions`, `intervals`, `pit_stops`, `stints`, `race_control`, `weather`, `storylines`
- Every table indexed on `(session_key, timestamp)` and relevant foreign keys
- `raw_events` (JSONB) as a catch-all for anything not yet structured

### 2.11 Frontend (`web/`)

A Vite-built React + TypeScript SPA. No server-side rendering (nothing to SSR for a live-data app). Routes: a home page with session picker, a session view with timing board + storyline feed, a replay controls overlay.

Data flow:
- Static metadata via REST + TanStack Query
- Live data via websocket → a Zustand store → components re-render on slice changes

Styling: Tailwind utility classes only, no custom CSS except a handful of CSS variables for team colors.

## 3. Data flow — a worked example

A user opens the app during the 2025 Bahrain GP.

1. Frontend fetches `/api/sessions` → sees Bahrain is live.
2. Frontend opens websocket to `/ws?session_key=9158`.
3. Websocket hub adds the client, sends a `snapshot` with the full current state.
4. Meanwhile, the `intervals` poller fires. It fetches the latest intervals, sees three new rows since last poll, pushes them to the ingestor.
5. Ingestor writes them to Postgres and updates the state model.
6. State mutation emits an event onto the bus.
7. Analytics recomputes derived metrics for affected drivers.
8. Detectors run. The `closing_gap` detector notices Norris's gap to Verstappen has been shrinking for 4 laps at 0.31s/lap. It emits a new storyline.
9. Storyline lands on the event bus.
10. Websocket hub fans it out to every client subscribed to session 9158.
11. Frontend receives the message, upserts into its storyline store, a new card slides into the feed.

Total latency from OpenF1 data availability to user screen: ~200–800ms in practice, dominated by OpenF1's own update cadence.

## 4. Why these choices

**Single process, not microservices.** A race has 20 drivers. Data volume is modest. Splitting this into services would add deployment complexity for no scaling benefit.

**Postgres for time-series data.** Proper time-series DBs (Timescale, Influx) would be overkill. A season of data fits comfortably in Postgres with good indexing. If growth becomes a problem, Timescale is a drop-in extension.

**sqlc, not GORM.** You learn SQL. You can read and optimize every query. The generated code is plain Go.

**Go channels, not Redis pub/sub.** One process, one process's worth of events. Channels cost nothing.

**Websockets, not SSE.** SSE is server→client only. We don't need bidirectional today, but an admin dashboard or a future "pause replay" button does, and websockets get that for free.

**React + Vite, not Next.js.** No SEO concerns (a live-data app isn't indexable content). No SSR concerns (every page is dynamic). Less to learn, faster to build.

**TanStack Query for static data, websocket store for live data.** These solve different problems cleanly. Don't try to force live websocket data through Query's cache.

**Recharts, not D3 or visx.** Fastest path to charts that look fine. Switch only if you hit a ceiling on a specific viz.

**Fly.io for the backend.** Handles long-lived websockets cleanly. Provisions Postgres next door. Scale-to-zero keeps the free tier sustainable.

**Vercel for the frontend.** Zero config for Vite, instant CDN, preview deploys on every branch.

## 5. Non-goals

- **No auth in v1.** Nobody's data is at risk. Add when user features arrive.
- **No caching layer.** The in-memory state model is the cache.
- **No background job queue.** Pollers and detectors are already goroutines.
- **No container orchestration beyond what Fly.io handles.**
- **No mobile app.** The web UI is responsive enough for tablets. Mobile-first is a v2 redesign.
- **No multi-tenant features.** This is one app for one audience; no concept of organizations or teams.

## 6. Scaling notes (for future reference)

For reference in case anyone asks, "what if this gets popular?"

- The backend is stateless except for the in-memory session model and websocket connections. To scale horizontally you'd need to (a) run the ingestion pipeline on exactly one instance (leader election) and (b) move the event bus to Redis pub/sub or NATS so any instance's websockets can receive events.
- Postgres scales vertically plenty far. The bottleneck will be the cardinality of `car_data` if you ever ingest full telemetry; that's the moment to add Timescale.
- The frontend scales for free — it's just a static bundle on a CDN.

None of this is v1 work. It's here so future-you doesn't repaint the architecture the first time someone asks.

## 7. Security notes

- No user data to protect in v1.
- OpenF1 is public and unauthenticated; nothing to secure on the outbound side.
- Rate-limit incoming API requests modestly (`httprate` middleware, e.g. 60 req/min per IP) so an errant client can't DOS the backend.
- CORS locked to the frontend origin.
- Websocket origin checked on upgrade.
- All secrets (DB URL, anything else) in env vars, never committed. `.env.example` documents what's needed.

## 8. Observability

- `slog` JSON logs to stdout, captured by Fly.
- `/metrics` endpoint exposing Prometheus-format counters/gauges (poll durations, events/sec, ws connections, storylines by kind). No Prom server provisioned in v1; ad-hoc curling is fine.
- `/debug/pprof/*` mounted behind a build tag or admin auth for goroutine leak hunting.
- A frontend `/admin` page (dev-only) that reads metrics and renders a few numbers is a nice luxury.
