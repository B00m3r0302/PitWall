# Pit Wall

A live F1 race companion. Open it in a tab next to the stream and it tells you what to pay attention to right now — closing gaps, undercut windows, tires falling off a cliff, strategy calls you'd otherwise miss.

> Status: in development. Building toward a v1 release. See the [roadmap](docs/roadmap.md).

---

## What it does

The broadcast shows you what's happening. Pit Wall shows you what it *means*. A Go service watches the live session, analyzes timing and telemetry in real time, and pushes a stream of "storylines" to a React dashboard:

- "Norris closing on Verstappen at 0.31s/lap — projected pass lap 47"
- "Undercut window opening: Leclerc on 14-lap mediums vs Hamilton on 4-lap mediums"
- "DRS train forming in P6–P9, four cars within 1.0s for three laps"
- "Virtual safety car deployed — pit lane open"

Alongside the storyline feed you get a live timing board, a tire-strategy strip per driver, and on-demand lap-time charts.

## Screenshots

_Coming soon._

<!--
Drop in once the UI is real:
![Live view](docs/media/live.png)
![Replay mode](docs/media/replay.gif)
-->

## Features

- Live timing board (positions, gaps, last lap, tires, pit count)
- Real-time storyline feed with ten detector types
- Tire strategy strip showing every driver's stints
- Expandable lap-time charts per storyline
- Replay mode — re-watch any past session at up to 20x speed
- Flag-state banner (green / yellow / VSC / SC / red)

## Tech stack

**Backend**
- Go 1.22+ (stdlib `net/http`, `log/slog`, `context`)
- PostgreSQL via `pgx/v5` and `sqlc`
- Websockets via `coder/websocket`
- Migrations via `goose`

**Frontend**
- React 18 + TypeScript
- Vite
- Tailwind CSS
- TanStack Query
- Recharts

**Data**
- [OpenF1](https://openf1.org) — live timing, telemetry, race control
- [Jolpica-F1](https://api.jolpi.ca/ergast/f1) — schedule and historical metadata

**Infra**
- Fly.io (backend + Postgres)
- Vercel (frontend)

## Quick start (local dev)

Prereqs: Go 1.22+, Node 20+, Docker, `make`.

```bash
git clone https://github.com/YOUR_USERNAME/pit-wall.git
cd pit-wall

# 1. Start Postgres
docker compose up -d

# 2. Run migrations
cd backend
go run ./cmd/migrate up

# 3. Backfill a sample race (so you don't need a live session)
go run ./cmd/server --backfill --session-key=9158

# 4. Start the backend in replay mode
go run ./cmd/server --replay --session-key=9158 --speed=5x

# 5. In another terminal, start the frontend
cd ../web
npm install
npm run dev
```

Open http://localhost:5173. You should see a replay of the chosen race with storylines firing in real time.

## Project structure

```
pit-wall/
  backend/
    cmd/
      server/         # main binary
      migrate/        # migration CLI
    internal/
      openf1/         # HTTP client for OpenF1
      ingest/         # pollers, backfill, replay
      state/          # in-memory session model
      analytics/      # gaps, pace, tires, projections
      storyline/      # detectors
      ws/             # websocket hub
      api/            # REST handlers
      store/          # sqlc-generated DB code
    db/
      migrations/
      queries/
  web/
    src/
      components/
      hooks/
      lib/
  docs/
    roadmap.md
    architecture.md
    storylines.md
    glossary.md
  docker-compose.yml
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for a deeper walkthrough.

## Running against a live session

```bash
go run ./cmd/server --live
```

The backend polls `?session_key=latest` on OpenF1. When no session is active, it idles; when one starts, it begins ingesting.

## Documentation

- [Roadmap](docs/roadmap.md) — the full build plan, week by week
- [Architecture](ARCHITECTURE.md) — how the pieces fit together
- [Storyline catalog](docs/storylines.md) — every storyline type and how it's detected
- [F1 glossary](docs/glossary.md) — terms used throughout the code and UI
- [Contributing](CONTRIBUTING.md) — how to help out

## Credits

Pit Wall is built on the generous work of two community projects. Go support them:

- **[OpenF1](https://openf1.org)** — the live timing and telemetry API this entire project is built on.
- **[Jolpica-F1](https://github.com/jolpica/jolpica-f1)** — the community-maintained successor to Ergast, used for schedules and historical data.

This project is not affiliated with Formula 1, the FIA, or any team.

## License

MIT. See [LICENSE](LICENSE).
