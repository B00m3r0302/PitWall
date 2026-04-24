# Contributing

Thanks for poking at Pit Wall. This is a personal project, so contributions are not required, but if you want to send a PR or an issue, here's what will make it painless.

## Ground rules

- Be nice on the issue tracker.
- Open an issue before a large PR. A 5-line discussion saves a 500-line rewrite.
- Small focused PRs merge faster than sprawling ones.

## Dev setup

Prereqs: Go 1.22+, Node 20+, Docker, `make`.

```bash
git clone https://github.com/YOUR_USERNAME/pit-wall.git
cd pit-wall
docker compose up -d            # Postgres

cd backend
go run ./cmd/migrate up
go run ./cmd/server --backfill --session-key=9158   # grab a sample race
go run ./cmd/server --replay --session-key=9158 --speed=5x

# in another terminal
cd ../web
npm install
npm run dev
```

Open http://localhost:5173. You should see replay data flowing through the full stack.

## Code style

**Go.**
- `gofmt` and `goimports` on save. `golangci-lint run` before pushing.
- Small packages, small interfaces.
- Errors are wrapped with `fmt.Errorf("doing thing: %w", err)`.
- Context is threaded through every I/O path.
- Table-driven tests.
- No ORMs, no code-gen other than sqlc.

**TypeScript / React.**
- `eslint` and `prettier` on save. Config committed.
- Function components only. No class components.
- Types live next to the code that uses them until they need to be shared, then promoted to `src/types/`.
- Tailwind utility classes only. No inline styles unless dynamically computed (e.g., team colors).

## Commits and branches

- `main` is always deployable.
- Feature branches: `feat/closing-gap-detector`, `fix/ws-reconnect-race`.
- Commit messages: short imperative, scoped. `feat(storyline): add tire_cliff detector`.
- Squash merge on PRs. Keep `main` history tidy.

## PR checklist

- [ ] New code has tests where it makes sense (detectors and analytics — always).
- [ ] `go test ./...` and `npm run test` both green.
- [ ] Linters clean.
- [ ] If it touches the API contract, updated `ARCHITECTURE.md` §2.9 and `docs/api.md` (if present).
- [ ] If it adds a new storyline, updated `docs/storylines.md`.
- [ ] Screenshots/GIFs for UI changes.

## Where to find things

- [README.md](README.md) — overview and quick start
- [ARCHITECTURE.md](ARCHITECTURE.md) — how it all fits together
- [docs/roadmap.md](docs/roadmap.md) — the build plan
- [docs/storylines.md](docs/storylines.md) — detector catalog
- [docs/glossary.md](docs/glossary.md) — F1 terms
- `backend/internal/` — where most Go lives
- `web/src/` — where most React lives

## Things I'd love help with

In rough priority:

1. New storyline detectors (see `docs/storylines.md` for the v1 list; stretch ideas live in the roadmap).
2. Threshold tuning against replays.
3. Mobile-friendly layout.
4. Performance profiling of the websocket hub under load.
5. Additional deploy recipes (Railway, Render, self-host on a VPS).

## Reporting bugs

Please include:

- What session (`session_key`) you were watching / replaying.
- Timestamps (the frontend shows them in the storyline feed).
- Relevant logs from the backend (`slog` outputs JSON to stdout).
- Browser and version.

## Questions

Open a GitHub Discussion or an issue with the `question` label.
