# F1 Glossary

Terms used throughout the code, docs, and UI. Ordered by how often you'll encounter them in this project, not alphabetically. Written for someone who knows Go but maybe not F1.

---

## Session types

- **Practice (FP1, FP2, FP3).** Teams tune their cars. Lap times are unreliable as signals — drivers run different fuel loads and tire compounds for different purposes. Pit Wall should de-emphasize predictions during practice.
- **Qualifying (Q1, Q2, Q3).** Three elimination rounds that set the grid. Short sessions, lots of activity, lots of fastest-lap events.
- **Sprint (at sprint weekends).** A short race on Saturday.
- **Race.** Sunday's Grand Prix. The main event.

In OpenF1, these are distinguished by `session.session_type`.

## Lap structure

- **Lap time.** Time for one complete lap, measured at the start/finish line.
- **Sector times.** A lap is split into three sectors (S1, S2, S3). Sector times are useful because a driver may nail S1 and bin it in S3.
- **In-lap.** The lap on which a driver enters the pit lane. The lap time includes pit entry, so it's slower than a normal lap.
- **Out-lap.** The lap after a pit stop, exiting the pit lane. Slower because the tires are cold and the driver is up to speed mid-lap.
- **Clean lap.** A lap with no in/out, no yellow flag, no traffic. The only laps you should use for pace and degradation analysis.
- **Flying lap.** A clean lap done at full attack. Synonymous with "hot lap."
- **Personal best (PB).** The driver's own fastest lap this session.
- **Session best / fastest lap.** The fastest lap anyone has set this session.

## Tire compounds

Pirelli supplies five dry compounds, three of which are nominated for each race weekend:

- **Soft (red sidewall).** Fastest, degrades quickest. Used for qualifying and short race stints.
- **Medium (yellow).** The in-between. Common race tire.
- **Hard (white).** Slowest raw pace, but durable. Long race stints.

And two wet-weather compounds:

- **Intermediate (green).** Light rain, damp track.
- **Wet (blue).** Heavy rain, standing water.

**Compound nomination:** the three dry compounds are called C1–C5 internally (C1 hardest, C5 softest). For each weekend, Pirelli picks three consecutive Cs and labels them soft/medium/hard for that weekend. So "soft" at Monaco is not the same rubber as "soft" at Silverstone. Your degradation model should be fit per session, not carried across races.

## Tire concepts

- **Stint.** A continuous run on one set of tires. A race has multiple stints separated by pit stops.
- **Tire age (in laps).** How many racing laps a set has done. Drives degradation.
- **Degradation ("deg").** The rate at which tires lose pace as they age. Roughly linear over most of a stint, then non-linear at the "cliff."
- **Cliff.** The point in a stint where a tire's pace drops off a cliff — suddenly much slower than the linear deg would predict. Usually from graining, overheating, or compound wear-through.
- **Graining.** Surface damage where the tire tears itself into small rolls; increases wear and drops pace.
- **Blistering.** Heat-induced damage; chunks of rubber come off.
- **Warm-up.** How long (in laps) a compound takes to reach operating temperature. Softs warm up quickest, hards slowest.

## Strategy concepts

- **1-stop / 2-stop / 3-stop.** How many pit stops a driver plans. The strategic choice.
- **Pit window.** The range of laps in which a pit stop is strategically sensible.
- **Pit-loss.** Time cost of a pit stop relative to staying out — typically 20–25 seconds, varies by circuit based on pit-lane length and speed limits.
- **Undercut.** Pitting earlier than the car ahead, trusting fresher tires to give enough pace advantage to emerge ahead when the rival pits.
- **Overcut.** The opposite: stay out longer, bank on the rival's tires degrading faster than yours while you run in clean air.
- **Box.** "Box, box" on team radio = "come into the pits." ("Box" is from Boxenstopp, the German loanword for pit stop.)
- **Mandatory compound change.** In dry races, each driver must use at least two different dry compounds. This forces at least one pit stop.
- **Free stop.** A pit stop that doesn't cost you positions (e.g., during a safety car).

## Gap / timing terms

- **Gap to leader.** Time behind P1, measured at the start/finish line.
- **Interval.** Time to the car immediately ahead.
- **Delta.** Change in something (lap time, gap) over time. "0.3s delta per lap" = the gap changes by 0.3s each lap.
- **DRS range.** Within 1.0s of the car ahead — triggers the DRS overtaking aid in DRS zones.

## Overtaking aids and rules

- **DRS (Drag Reduction System).** A flap on the rear wing that opens in designated zones when within 1.0s of the car ahead, reducing drag and making overtaking easier. Detected by a "DRS detection point" then enabled through the zone.
- **DRS train.** Three or more cars within 1.0s of each other. Nobody can overtake because everyone behind also has DRS. Classic frustration pattern.
- **Slipstream / tow.** Drafting behind another car in a straight.
- **Dirty air.** Turbulent air behind another car that reduces downforce and makes following hard. Gets worse in corners.
- **Clean air.** No car ahead; you can extract full pace.

## Flags and session states

- **Green flag.** Normal racing.
- **Yellow flag (single or double).** Danger at a point on the track. Single = slow; double = slow significantly, be prepared to stop.
- **Red flag.** Session stopped. Cars return to the pit lane.
- **Checkered flag.** Session over.
- **Blue flag.** Faster car approaching behind; move over (lapped cars).
- **White flag.** Slow vehicle on track.
- **Safety Car (SC).** A physical pace car on track behind which the field bunches up. Used for major incidents requiring marshals on track.
- **Virtual Safety Car (VSC).** No physical car; drivers must drive to a delta time. Used for lesser hazards.
- **Formation lap.** The lap from the grid to the grid, with no racing — drivers warm up tires.
- **Standing start / rolling start.** How the race begins; standing is from a stop, rolling is behind the safety car.

## Penalties

- **Time penalty (5s, 10s).** Added to the driver's race time or served at their next pit stop.
- **Drive-through.** Drive through pit lane at the limit without stopping.
- **Stop-and-go.** Stop in the pit box for a set duration without servicing.
- **Grid drop.** Starting position penalty for the next race.
- **Reprimand.** Warning; accumulating three in a season = grid drop.
- **Black flag / DSQ.** Disqualification.
- **Track limits.** Going wide of the white line that defines the track. Three warnings = penalty at most circuits.
- **Unsafe release.** Released from a pit stop into traffic.

## Roles and comms

- **Race engineer.** The driver's primary radio contact on the pit wall. All the team-radio clips you hear are usually between driver and race engineer.
- **Strategist.** Plans pit stops and tire choices.
- **Pit wall.** The team's bank of seats and monitors along pit lane. Hence this project's name.
- **Marshals.** Trackside stewards who deploy flags, clear debris, rescue cars.
- **Stewards.** The panel (including a driver-steward) who adjudicate incidents and give penalties.

## People / team / car ecosystem

- **Driver number.** Each driver has a permanent number; use it as the canonical identifier. OpenF1 uses `driver_number`.
- **Team / constructor.** The entity that builds the car and enters two drivers.
- **Livery.** The paint scheme. Teams keep distinct colors year to year, which you can use for UI accents.
- **Car number vs position.** Car number is identity; position is live race order. Don't conflate.

## Technical / telemetry

- **Telemetry.** High-frequency car data: speed, throttle, brake, gear, RPM, DRS state, steering. OpenF1 exposes this as `/v1/car_data`.
- **ERS (Energy Recovery System).** Hybrid power deployment. Managed strategically; drivers might save ERS for attack laps.
- **Fuel load.** Cars start heavy and lighten through the race. Affects lap times by roughly 0.03s/lap/kg.
- **Ride height / setup.** Static configuration of the car; not visible in our data but worth knowing when someone says "setup difference."
- **Parc fermé.** Post-qualifying lockdown where teams can't change the car setup. Relevant to understanding why qualifying pace carries through to the race.

## Event / calendar

- **Grand Prix (GP).** A race weekend.
- **Meeting.** OpenF1's term for a GP weekend. One meeting contains multiple sessions.
- **Calendar.** The ~24 GPs in a season.
- **Sprint weekend.** A weekend with a Sprint race on Saturday as well as the GP on Sunday.
- **Round.** A race's position in the season calendar (Round 1 = first GP of the year).

---

Refer back here when you're writing detector comments, UI tooltips, or your README. A consistent vocabulary makes the whole project feel more professional and the code easier to read.
