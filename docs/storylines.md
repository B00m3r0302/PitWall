# Storyline Catalog

This is the product spec for Pit Wall's core feature: the storyline feed. Each detector listed here lives in `backend/internal/storyline/` as its own struct with a `Detect(snapshot State) []Storyline` method. This doc describes the *product behavior* of each — what triggers it, what the user sees, and the edge cases that will bite you.

Write all ten detectors for v1. You can ship with five or six working well and the rest stubbed, but the feed will feel thin with fewer than five firing.

---

## Common shape

Every storyline carries these fields (see `ARCHITECTURE.md` §2.6):

- `ID` — stable and deduplicatable. Usually `<kind>:<session_key>:<driver_nums_joined>:<lap>` or similar. Crucial: a detector should produce the *same* ID for the *same situation* so dedup works.
- `Kind` — one of the strings below.
- `Priority` — 1 (trivial) to 10 (session-defining). Suggested ranges per kind are listed.
- `Headline` — one line, ≤80 chars, scannable at a glance.
- `Detail` — one paragraph, for the expanded card.
- `DriverNums` — involved drivers by car number.
- `CreatedAt` / `ExpiresAt` — drives auto-dismissal in the UI.
- `Payload` — structured JSON the frontend uses to draw the mini-chart when expanded.

A detector's job is pure: take a state snapshot, return zero or more storylines. No I/O. Easy to unit-test.

---

## 1. `pit_stop`

**Description.** A driver entered pit lane and completed a stop. Emit when the stop ends, not when they enter.

**Inputs.** Pit stop event from OpenF1 `/v1/pit`, their stint before and after from `/v1/stints`, position before and after.

**Detection logic.** Watch for new rows in the pit stops channel. Join with the stint that ends at this lap (tire coming off) and the stint that begins (tire going on). Position delta is positions at `lap-1` vs positions at `lap+1` (give it a lap to settle).

**Headline example.** "Verstappen pits — mediums to hards, 2.4s stop, back out in P3"

**Detail example.** "Verstappen took on hard tires after a 14-lap stint on mediums. Stop time 2.4s (team average this season: 2.3s). Emerged in P3, 8.2s behind the leader, 4.1s ahead of Russell."

**Priority.** 5–7. Bump to 8 for front-running drivers during a critical strategy phase.

**Payload.** `{ "driver": 1, "lap": 32, "duration": 2.4, "compound_off": "medium", "compound_on": "hard", "position_before": 2, "position_after": 3 }`

**Edge cases.**
- Double-stack stops (two teammates pit on consecutive laps) — emit both, don't merge.
- Drive-through penalties appear in `/v1/pit` but aren't real stops. Filter by duration (< ~10s is a normal stop; very short = drive-through; very long = problem stop → different storyline).
- Pit during VSC/SC — still emit, but add "under VSC" / "under SC" to the detail.

---

## 2. `closing_gap`

**Description.** Driver B is closing on driver A at a sustained rate. Project where they catch up.

**Inputs.** Last N intervals between adjacent drivers, both drivers' last 3 clean lap times, tire age and compound, known pit-loss time for this circuit.

**Detection logic.** Over the last 3 laps, gap between A and B has decreased at an average rate of ≥ 0.15s/lap. The trend must be monotonic-ish: not one big drop and two tiny opens. Fire once when conditions first hold; refresh the storyline (not duplicate) when the rate changes materially.

**Headline example.** "Norris closing on Verstappen at 0.31s/lap — projected pass lap 47"

**Detail example.** "Norris has taken 0.94s off Verstappen over the last three laps. At the current rate, he'd be in DRS range by lap 44 and ahead by lap 47. Both are on hard tires; Norris's are 6 laps fresher."

**Priority.** 4–7. Higher for position changes affecting the podium.

**Payload.** `{ "chaser": 4, "leader": 1, "gap_history": [2.8, 2.5, 2.2, 1.9], "rate_s_per_lap": 0.31, "projected_pass_lap": 47 }`

**Edge cases.**
- Don't fire if the closing is purely due to the leader pitting or being under yellow. Filter by `session_state == green`.
- Update the ID if the projected pass lap changes by more than 2 — treat that as a new storyline, not an update.
- Projection is notoriously fragile late in a stint as tires degrade. Include a "confidence" field and let the UI render low-confidence predictions more subtly.

---

## 3. `undercut_window_open`

**Description.** Conditions are right for driver B (just behind A) to undercut A by pitting first.

**Inputs.** Both drivers' current stints (compound, tire age, clean-lap pace), gap A→B, circuit pit-loss time.

**Detection logic.** Given current pace, tire age, and degradation curve for the compound: if B pits now and emerges N seconds behind where A will be 2 laps later, and B's fresh-tire pace advantage over those 2 laps exceeds what A gains by staying out, the undercut works. Fire when:

1. B is within ~22s of A (typical undercut threshold),
2. B's tires are at least 5 laps older than A's, OR B is on a harder compound that's degraded,
3. The projection says the undercut succeeds by at least 0.5s of cumulative advantage.

**Headline example.** "Undercut open: Leclerc on 14-lap mediums vs Hamilton on 4-lap mediums"

**Detail example.** "Leclerc's medium tires are 14 laps older than Hamilton's. Pit-loss at this circuit is about 22s. If Leclerc pits this lap and Hamilton follows two laps later, fresh-tire advantage gives Leclerc roughly 1.1s over the stop cycle — enough to exit ahead."

**Priority.** 5–8. Bumped near the front or in tight midfield battles.

**Payload.** `{ "attacker": 16, "defender": 44, "gap_s": 1.8, "pit_loss_s": 22, "projected_advantage_s": 1.1 }`

**Edge cases.**
- Pit lane closed (safety car period) — suppress.
- Driver has already pitted for their mandatory compound change — the undercut is less meaningful if it means a costly third stop.
- Weather turning: if rain is incoming, the calculus flips and the undercut might be moot. Degrade to lower priority if `weather_change` storyline recently fired.

---

## 4. `tire_cliff`

**Description.** A driver's lap times are falling off the expected degradation curve for their compound — the "cliff."

**Inputs.** Last 5 clean laps for a driver, the session-fitted degradation curve for their current compound.

**Detection logic.** Fit a per-compound linear degradation model from all drivers' clean laps this session (lap time ≈ base_time + deg_rate × tire_age). For a given driver, compare observed last-lap to predicted. If observed is more than 2 standard deviations above prediction for 2 consecutive laps, they're past the cliff.

**Headline example.** "Hamilton's hards are falling off — last two laps 0.8s slower than model"

**Detail example.** "Hamilton's last two laps came in 0.8s and 0.9s above the hard-compound deg curve, suggesting end-of-stint graining. Expect a pit stop imminently or a visible drop in pace."

**Priority.** 4–6.

**Payload.** `{ "driver": 44, "compound": "hard", "tire_age_laps": 28, "recent_laps": [94.2, 94.9, 95.1], "predicted_lap": 94.3 }`

**Edge cases.**
- Single-driver-on-compound situations break the model. Require ≥ 3 drivers' clean laps on a compound before firing for that compound.
- Track evolution (grip increases over a session) can masquerade as degradation if your model doesn't include a session-time term. Either add one or use a rolling fit over the last 15 minutes.
- Suppress if the driver has just been held up (check gap-to-ahead shrunk meaningfully that lap).

---

## 5. `fastest_lap`

**Description.** A new session-fastest lap.

**Inputs.** Incoming lap rows from `/v1/laps`.

**Detection logic.** On every new lap, if `lap_time < current_session_best`, emit. Dedup by `session_key` — each new session-best produces one card.

**Headline example.** "Fastest lap: Leclerc, 1:32.456 on fresh softs"

**Detail example.** "Leclerc sets a new session fastest lap, 1:32.456, on 2-lap-old soft tires. Previous best was Verstappen at 1:32.612 (lap 21)."

**Priority.** 3–5. Higher late in a race when the fastest-lap point is in play.

**Payload.** `{ "driver": 16, "lap_number": 28, "lap_time": 92.456, "compound": "soft", "tire_age": 2, "previous_best_driver": 1, "previous_best": 92.612 }`

**Edge cases.**
- Practice sessions see dozens of new session-bests. Rate-limit or lower priority during practice.
- Ignore laps under VSC/SC (they can't be fastest laps anyway but guard against weird data).

---

## 6. `overtake`

**Description.** A position change during green-flag running.

**Inputs.** Position rows, session state.

**Detection logic.** When a driver's position changes while `session_state == green` and neither driver just pitted (check last 2 laps). Emit from the overtaker's perspective.

**Headline example.** "Piastri passes Russell for P4 on lap 38"

**Detail example.** "Piastri overtook Russell on the run to turn 4, gaining P4. Piastri is on 12-lap-old mediums; Russell on 19-lap-old hards."

**Priority.** 4–8. Higher for position changes in the top 5.

**Payload.** `{ "overtaker": 81, "overtaken": 63, "new_position": 4, "lap": 38 }`

**Edge cases.**
- Laps-lapped cars can cause phantom position changes. Guard on driver not being more than 1 lap behind leader.
- Multi-car incidents (pile-up, crash) can swap many positions at once; don't emit a storm. Either rate-limit by driver or emit a meta-storyline ("Multi-car incident reshuffles the order").
- Start-of-race order shuffle: suppress overtake storylines for the first 2 laps (too noisy, and fans are watching anyway).

---

## 7. `drs_train`

**Description.** Three or more cars within 1.0s of each other for multiple laps — classic "nobody can overtake" pattern.

**Inputs.** Intervals across consecutive drivers.

**Detection logic.** For each position P, check intervals(P→P+1), intervals(P+1→P+2), etc. If three consecutive drivers all have interval ≤ 1.0s for 2+ laps, emit.

**Headline example.** "DRS train forming: P6–P8 within 0.8s for three laps"

**Detail example.** "Gasly, Alonso, and Stroll have been nose-to-tail for three laps, all within DRS range of the car ahead. Without fresh tires or a strategy offset, positions likely hold to the end of the stint."

**Priority.** 3–5.

**Payload.** `{ "drivers": [10, 14, 18], "positions": [6, 7, 8], "intervals": [0.6, 0.8], "laps_held": 3 }`

**Edge cases.**
- Train composition changes (someone drops off the back, someone new joins at the front). Treat meaningful changes as new storylines with new IDs.
- Avoid firing about long snaking queues behind a slow leader during the opening stint — wait until lap 5+.

---

## 8. `flag_change`

**Description.** Session state change: green → yellow, yellow → VSC, SC deployed/ended, red flag, checkered.

**Inputs.** Race control messages.

**Detection logic.** Parse `/v1/race_control` messages for category = Flag or SafetyCar. Track current session state; fire on transitions.

**Headline example.** "Virtual Safety Car deployed"

**Detail example.** "VSC triggered after debris reported in sector 2. All cars must slow to the VSC delta. Pit lane remains open. Typical VSC duration: 2–4 laps."

**Priority.** 7–10. SC and red flags are 9–10; VSC is 7–8; sector yellows are 5.

**Payload.** `{ "state": "vsc", "reason": "debris_sector_2", "lap": 31 }`

**Edge cases.**
- Sector yellows come and go constantly in practice/qualifying; only fire for full-course yellows or worse in those sessions.
- Race control sometimes publishes clarifications minutes after the event. De-dup carefully.
- Red flag restarts are a separate storyline (kind: `red_flag_restart`) because they change the whole race — consider creating it as a follow-up.

---

## 9. `penalty_applied`

**Description.** A driver has been given a time penalty, grid drop, or reprimand.

**Inputs.** Race control messages.

**Detection logic.** Parse `/v1/race_control` for penalty-related categories ("Penalty", "Investigation", etc.). Emit on investigation-under-way, and again on penalty-given (treat as related but distinct — separate IDs).

**Headline example.** "5-second penalty for Sainz — forcing another driver off track"

**Detail example.** "Sainz has been given a 5-second time penalty for forcing Alonso off the track at turn 9, lap 22. The penalty will be added at his next pit stop or at the end of the race."

**Priority.** 5–8. Higher for race-defining penalties (track limits accumulating, DSQ-relevant).

**Payload.** `{ "driver": 55, "penalty_type": "time_penalty_5s", "reason": "forcing_off_track", "lap": 22 }`

**Edge cases.**
- Investigations often don't lead to penalties. Ensure the "under investigation" card auto-expires if no penalty is given within a few laps.
- Some messages are informational and not a real penalty ("Track limits warning"). Classify carefully and don't spam.

---

## 10. `weather_change`

**Description.** Material change in track or air conditions.

**Inputs.** `/v1/weather` — track temp, air temp, humidity, rainfall, wind.

**Detection logic.** Rolling window over last 10 minutes of weather samples. Fire on:
- rainfall starts (was 0, now > 0)
- rainfall stops
- track temp swings > 5°C within 10 minutes
- wind direction shifts > 45°

**Headline example.** "Rain starting in sector 3"

**Detail example.** "Rainfall sensor now reading 0.4mm. Expect intermediates imminent if rate sustains or grows. Track temp dropping quickly; tire prep assumptions shift."

**Priority.** 6–9. Rainfall start/stop is 8–9; temp swings are 6.

**Payload.** `{ "event": "rain_start", "rainfall_mm": 0.4, "track_temp_c": 34.5, "air_temp_c": 22 }`

**Edge cases.**
- Sensor noise at single sectors. Require ≥ 2 consecutive readings before firing.
- Crossing the "dry" threshold then back within 2 minutes (damp/drizzle): throttle or downgrade priority.

---

## Detector quality rules

Apply these to every detector:

1. **Stable IDs.** Same situation → same ID. A detector firing twice for the same state produces two storylines with the same ID; the dedup layer drops the second.
2. **Silent on stale data.** If the state snapshot's latest-update timestamp is older than 15 seconds, skip. Don't fire off stale data.
3. **Guard on session state.** Many detectors (closing_gap, overtake, undercut) are meaningless or misleading under yellow. Always filter.
4. **Confidence, not certainty.** Projections should carry uncertainty. Let the UI convey confidence visually.
5. **Tunable thresholds.** Every magic number (0.15s/lap, 1.0s for DRS, 5 laps for tire-age difference) lives in a single config struct and is tunable without recompiling detectors.

## Tuning playbook

Once all detectors are implemented:

1. Run replay on 3 varied races (wet, 1-stop, chaotic SC race) at 10x speed.
2. For each, list every storyline that fired. Rate each card: "useful", "wrong", "noise".
3. Adjust thresholds per detector. Re-run. Target: ≥ 80% "useful" rate and roughly one storyline every 30–90 seconds during green-flag running.
4. Record the tuned thresholds in the config and commit with a note on what race you tuned against.

The storyline feed is the product. The detectors are the product. Budget real time for tuning — it's the difference between a demo and a thing people use.
