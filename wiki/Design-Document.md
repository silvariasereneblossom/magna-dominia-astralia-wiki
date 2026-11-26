# Magna Dominia Astralia: design document

Every design page in one document. Generated from `docs/wiki/design/`; edit the pages there, not this one.


---

# Time and modes

**Status:** built, as of 2026-11-23. The clocks in M2, the turn-based polish and parity in M13, the automation defaults across M3–M12b and M14.

The simulation advances in whole days (ticks), and every rule resolves on every tick. Two controllers drive that one deterministic loop: real-time-with-pause (RTS) and turn-based (TB), and the player can switch between them at any day in single-player without changing the game state. The automation defaults let a fresh game run with very little input, so the player can choose how much of the empire to manage by hand.

## How it works

### Units of time

| Unit | Definition |
|---|---|
| Tick | 1 day. Everything resolves: movement, combat, construction and its material draw, production, trade dispatch and arrivals, market allocation and pricing, pop growth and migration, research, ledger, happiness, piracy, events, diplomacy decay, AI reactions. |
| Month | 30 ticks. A display and rate period only: content rates are written per 30 days (`/mo`), the simulation applies `rate / 30` each tick, and the UI shows 30-day rolling sums ("income this month"). |
| Year | 12 months = 360 ticks. Autosave (RTS), score, crisis checks, leader ageing. |
| Turn (TB) | One month (30 ticks) by default; a 10-day option exists. |

Durations given in months mean 30-day spans. The month is not a processing boundary for the economy: stocks, prices, happiness and research move day by day.

### One step for both clocks

The simulation exposes `step(world, commandsForThisTick)`, which advances exactly one tick. Nothing in the simulation knows which clock drives it. No event trigger, modifier or rule may test which clock is active, and every feature must state its RTS and TB behaviour in its data or code comment; a feature that only makes sense with reactive input is cut or given a standing order (`01 §15`).

State changes from outside the simulation only through a `Command`, a serialisable order applied at a tick boundary. The UI never mutates simulation state.

### AI planning, the only periodic work

AI empires run their heavy planner once every 30 days, staggered across the month. With `day = tick % 30 ∈ 0..29`, AI empire `e` (0-based among AI empires) of `N` plans on `planDay = floor(e × 30 / N)` and queues its commands in the saved `pendingCommands` table to apply on the following tick. No tick carries more than one plan. Plans run inline at the start of the planning tick under either clock, with no worker thread; a tripwire fails CI if one plan exceeds 250 µs × the host's calibration (`07 D55`, the owner's call of 2026-11-13, replacing the worker-thread design first written into `05 M9`).

### The real-time clock (RTS)

| Speed | Ticks per second |
|---|---|
| Paused | 0 (pause is free and instant) |
| 1 | 1 |
| 2 | 2 |
| 3 (default) | 4 |
| 4 | 10 |
| 5 | as fast as the budget allows, capped at 120 |

At the default speed 3 one year takes about 90 s and a 300-year game about 7.5 h. Player commands apply at the start of the next tick. An event choice applies on the tick after the choice is made.

### The turn-based clock (TB)

A turn runs in four phases:

1. **Planning** (simulation halted). The player issues commands; they are queued, shown as pending, and can be revoked at no cost (the × on each pending row while no tick runs). Pending commands are part of the save.
2. **End turn.** The batch is locked and tagged with the tick it applies on, tick 1 of the turn. Movement orders are continuous once issued.
3. **Resolution.** 30 `step()` calls (10 on a 10-day turn). Reactions during resolution come only from standing orders (fleet stance, market policies, auto-build), never from new input. AI planners run on their planning days inside the resolution, as in RTS.
4. **Turn report.** A window that opens after each turn and is toggled by Report beside End turn. It holds the choices waiting, a playback scrubber and a digest (below). Event popups that require a choice are presented now, in order; their effects apply on the tick after the choice, which is tick 1 of the next turn.

As built in M13 (`07 D93`, `D94`):

- **Playback.** While a turn resolves, the app's driver records after each tick the fleets the player can see, at their map positions. The track lives in the turn report only and is dropped at the next turn; nothing is saved or hashed, so RTS and TB parity is untouched. Battles come from the battle events. While "Show on map" is ticked, the scrubber's frame replaces the live fleet layer on the galaxy map. One scrub step is budgeted at 2 ms (`07 D96`).
- **Digest.** Each empire's credits, influence, unity and banked research at the turn's start and end; the player's markets' price moves of 10% or more, the eight largest; the player's new techs, a count of other empires' techs, first contacts and conquests.
- **Turn length.** `Control::SetTurn` takes 30 or 10 while no tick runs; the TB bar shows 30 d and 10 d. A command issued during a resolving turn applies on the first tick of the next turn of that length. The length is a driver setting and is not saved: a loaded game starts at 30.
- **Auto end turn.** A checkbox beside End turn. In the planning phase it ends the turn when no choice is waiting and no battle waits for its result.
- **Stance nudge.** In TB each military fleet's orders show its stance as a picker, with the reason in the hover text: no reactions until the next turn.

### Battles and the clock

When a battle is the player's (its empire is in an engaging pair there), the battle waits for the tactical screen. The driver runs no tick while the battle waits, in RTS and inside a TB turn alike; the result applies on the next tick (`07 D66`). Ground battles are the exception: they are daily orders and do not pause the clock (`07 D89`). See [combat.md](Design-Combat) and [ground-combat.md](Design-Ground-combat).

### Events and timeouts

An unanswered event auto-resolves to its default option after `timeoutDays` in both modes.

### RTS and TB compared

| Concern | RTS | TB |
|---|---|---|
| Player command application | next tick | tick 1 of the next turn (movement continuous once issued) |
| Economy | every tick, all empires | identical (30 daily resolutions per turn) |
| AI planning | every 30 days per AI empire on its planning day `floor(e × 30 / N)` | identical (runs during resolution) |
| Reactions | standing orders + player can pause | standing orders only; the UI nudges stance selection on every fleet order |
| Events needing a choice | optional pause; effects the tick after the choice | collected into the turn report; effects on tick 1 of the next turn |
| Combat and price visibility | live | playback scrubber from battle reports and daily price logs |
| Autosave | yearly | every turn (ring of 5) |
| Research and tradition picks | popup when ready | shown in the turn report; auto-pick if unanswered (setting) |
| Undo | none | planning-phase commands revocable |
| Speed | 1–5 | "End turn" + optional auto end turn |
| Persona actions | commands next tick; contracts have tick deadlines | identical; the turn report lists contract state and cargo sales |

**Switching clocks** is allowed in single-player at any day; the state is identical. The choice is remembered per save (the save container's tag carries the clock mode).

**Parity test** (`rts_and_tb_runs_hash_alike`, `05 M13`, `07 D94`): Tiny, seed 3, 2 AI empires, six scripted commands at apply ticks 0, 17, 30, 95, 200 and 213, no event answered. 360 RTS ticks, 12 TB turns of 30 days and 36 of 10 days give one hash. A revoked planning command leaves the turn's hash equal to a run without it.

### Automation defaults

Automation runs as standing orders on the timer heap, never by per-tick polling. A fresh game with all automation on is meant to be playable by only picking techs, setting fleet orders in wars, and answering events.

| System | Default automation | Player override |
|---|---|---|
| Science ships | auto-explore and auto-research anomalies | manual target, project queue |
| Construction ships | auto-build stations on every surveyed deposit in owned systems (home system first), then habitats at the richest in-system bodies while alloys exceed 200, in owned systems only; materials by the local rule of `01 §10.4` | manual |
| Colonies | the governor builds districts and buildings toward the designation, keeps 1 slot free, fixes projected shortages in a priority order, builds a Spaceport when freight utilisation is over 80%, builds housing when crowding is over 1.1, zones before it builds; auto-expedition | per-planet "manual" toggle |
| Markets | all stock policies `auto`; sells overflow; buys to cover a shortage when credits exceed 3 months of upkeep | per commodity |
| Research | the goals' next step, else a prompt to pick (setting: auto-pick the cheapest) | — |
| Fleets | auto-upgrade when docked; auto-refuel; auto-return to supply when readiness < 50 or fuel < 10 hyperspace days (`2 × Σ fuelPerJump`) | stance, manual orders |
| Diplomacy | none (always manual); AI proposals arrive as notifications | — |
| Trade fleets (1.0) | auto-assign to the most profitable route | manual route |
| Hub designation | a suggestion notification when a spoke exceeds 30 route days | manual |

**The governor's shortage rule.** A shortage is projected when `stock / max(dailyDemand − dailyProduction − imports30d, 0.0003) < 360` days, where `imports30d` is the market's 30-day mean of arrivals. It is never raised for a commodity that another market in the same system holds 90 days or more of, so import-fed domes do not demand their own farms. The fix order is: food → goods → gas → fuel (Refinery) → alloys → crystal and parts (crystal station, Workshop) → rare metals (a Metal Refinery where the empire already makes rare ore in the system, else a Smelter, `07 D162`) → luxuries (Luxury Works). The system's most populous own colony also counts its starbase's refined-fuel draw and stock (`07 D57`). A fix that could not be ordered is not tried again at that colony for 3 governor periods, or until a construction in its system finishes or the colony gains a slot or a surveyed hex (`07 D242`).

**Governor timing.** Governor decisions are evaluated when a colony's state changes or once every 30 days per colony (staggered by colony id), never every tick.

**Stuck builds** (`07 D264`). A governor's own district or building that has made no progress for over `budget.cancelAfter` (360) days since it started drawing is cancelled on the colony's governor day. The slot and the hex are freed and nothing drawn comes back. The player's own orders are never cancelled. The AI cancels any district, building, ship or upgrade of its own stuck that long, on its planning day, in RTS and TB alike.

**Settings screen** (`07 D145`, built in M12b for M14). Three automation defaults, all on by default: research auto-pick, colony governors, and science ships' auto-explore. They are saved with the settings and passed to a new game as `GameSettings::manual`; the chosen ones start off for the player's colonies, science ships and research, including new colonies and new science ships. Each stays the per-colony, per-fleet or research toggle it was in play.

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Daily resolution | Every rule resolves every tick at 1/30 of the per-30-day content rate; the month is display only | 2026-11-01 | `01 §1` (v0.8) |
| Default TB turn length | One month (30 ticks) | 2026-11-13 (owner's answer to `D6`) | `07 D6` |
| AI planning | Inline at the start of the planning tick, no worker thread, a 250 µs × calibration tripwire; replaces the worker-thread plan of `05 M9` | 2026-11-13 | `07 D55` |
| Player battles pause the clock | No tick runs while the player's battle waits, in RTS and TB | 2026-11-13 | `07 D66` |
| Turn report | Playback kept by the driver, unsaved and unhashed; digest of ledgers and price moves; a TB window after each turn | 2026-11-14 | `07 D93` |
| 10-day turns and auto end turn | 10-day turns built as an unsaved driver setting; auto end turn waits only on choices and a waiting battle; TB stance picker on military fleets | 2026-11-14 | `07 D94` |
| Automation defaults on the settings screen | Research auto-pick, governors and auto-explore as saved settings passed to a new game | 2026-11-16 | `07 D145` |
| Governor retry back-off | A fix that could not be ordered waits 3 governor periods or a change | 2026-11-18 | `07 D242` |
| Stuck-build cancel | Governor and AI builds stuck 360 days are cancelled without refund; player orders never | 2026-11-19 | `07 D264` |
| Research auto-pick | The research draw and its category focus are gone; a field with Auto on takes its cheapest researchable tech when no goal step is available. This replaces the "auto-pick by focus" wording of `07 D145` | 2026-11-21 | `07 D278` |
| Threads | Threads only through `sim::par`, det hashing the same single- and multi-threaded; the harness is merged, the Cores setting is not built until a pass pays | 2026-11-18 | `07 D240`, `DESIGN.md` autonomous calls |

## Not yet built or still open

- The Cores setting for core offloading (`07 D240`) is not built; `sim::par` exists with no parallel caller.
- Trade fleets' auto-assign is a 1.0 feature.
- `05 M2`'s hidden-tab web check (ticks elapsed with the tab hidden) is owner-manual and not recorded as measured.
- Whether TB research and tradition picks auto-pick if unanswered, as the `01 §15` table states, is not confirmed by the M13 build record, which lists auto end turn waiting on choices and battles only.

## Discrepancies noted

- **Autosave toggle vs. fixed autosave cadence.** `01 §15` gives autosave as "yearly" (RTS) and "every turn (ring of 5)" (TB) with no off switch. `DESIGN.md` (autonomous calls, 2026-11-14, `07 D97`) records "the autosave toggle and a key list join G4's settings". Both may hold (a toggle over the cadence), but `01 §15` does not mention the toggle.
- **The research automation wording.** `01 §17` says "the goals' next step, else prompt to pick (setting: auto-pick the cheapest)". `07 D145` says the settings screen holds "research auto-pick by focus". `07 D278` (2026-11-21, built as T1 on 2026-11-23) removes the category focus, so "by focus" appears stale; `01 §17`'s wording matches `D278`.

## Sources

- 01 §1 Time model
- 01 §15 Turn-based vs real-time specifics
- 01 §17 Automation defaults
- SPEC.md (glossary: tick, month, turn, command, planning day)
- 07 Decisions: D6 (the owner's answers of 2026-11-13 at the top), D55, D66, D89, D93, D94, D96, D97, D145, D240, D242, D264, D278
- 05 Milestones: M2, M9, M13, M14
- DESIGN.md (pillar "Time and modes"; autonomous calls of 2026-11-14 and 2026-11-18)

## Related pages

- [Overview](Overview)
- [ai-and-victory.md](Design-Ai-and-victory)
- [architecture-and-performance.md](Design-Architecture-and-performance)
- [colonies-and-surfaces.md](Design-Colonies-and-surfaces)
- [combat.md](Design-Combat)
- [ground-combat.md](Design-Ground-combat)
- [events-and-crises.md](Design-Events-and-crises)
- [presentation-and-ui.md](Design-Presentation-and-ui)
- [progression.md](Design-Progression)
- [flight-view.md](Design-Flight-view)


---

# Galaxy and hyperspace

**Status:** built, as of 2026-11-23. Galaxy generation in M1, hyperspace and deep space in M8c (verified 2026-11-13); gateway trade links in M19a.

The galaxy is generated as a pure function of the seed and the settings. There are no hyperlanes: star systems sit in hyperspace, an open and dangerous medium that fleets cross from any system to any other along routes planned at departure. Its weather (storms and currents) is fixed by the seed and the month, so travel is forecast exactly but never stable. Empires build beacon corridors that the shifting currents keep bending, and later techs add warp and buildable wormholes (Gateways). Beyond charted space, deep space grows more dangerous and more rewarding with distance.

## How it works

### Conventions

Map coordinates are a 1000 × 1000 unit square. Ship `speed` is map units per 10 days. Generation is a pure function of `(seed, settings)`: same inputs, same galaxy, on every platform.

### Settings

| Setting | Options (default bold) |
|---|---|
| Size | Tiny 150, Small 250, **Medium 400**, Large 700, Huge 1000 systems |
| Shape | **Spiral-2**, Spiral-4, Elliptical, Ring, Clustered |
| AI empires | 0–24, **8** on Medium |
| Habitable density | Sparse 0.6×, **Normal 1×**, Dense 1.5× |
| Hyperspace weather | Calm 0.5×, **Normal 1×**, Turbulent 1.5× (scales storm count and current strength) |
| Pirates | Off, **Normal**, Aggressive |
| Endgame crisis year (1.0) | 100–400, **200**; Off |
| Game end year | 150–500, **300**; Endless |

### Generation algorithm

1. **Density mask** from the shape: spiral arms by a logarithmic spiral with noise; elliptical by a Gaussian; ring by an annulus; clustered by 6–12 Gaussian blobs.
2. **Placement:** Poisson-disc sampling within the mask, minimum separation `d_min = 0.9 × sqrt(area / N)` (45 units at Medium), until N systems. Neighbouring systems end up 45–90 units (2–4 cells) apart.
3. **Hyperspace clouds:** 6–12 cloud discs of radius 2–6 cells and 0.6–1.8 extra drag at the centre, rolled after empire placement (step 9) and moved off any home system, so no empire starts blind (`07 D46`).
4. **Stars** by weight:

   | Class | Weight |
   |---|---|
   | O/B | 5% |
   | A | 8% |
   | F | 15% |
   | G | 25% |
   | K | 25% |
   | M | 15% |
   | binary | 4% |
   | neutron | 1.5% |
   | black hole | 1% |
   | pulsar | 0.5% |

   Class drives planet count (0–7), temperature band and system modifiers: pulsar, shields −100% in combat; black hole, +25% physics research from a research station; neutron, +50% energy from a station.
5. **Bodies:** each planet slot rolls a type by the star's temperature band. Habitable classes are continental, ocean, tropical (wet); arid, desert, savanna (dry); alpine, tundra, arctic (cold). Non-habitable: barren, molten, toxic, frozen, gas giant, asteroid belt. Sizes: habitable 8–22 (normal 12–18), gas giant 25–30 (uncolonisable), others 5–15. Each body gets a surface seed and 0–3 orbital deposits `(kind ∈ ore | gas | crystal | energy | research, amount 2–6)`, exploitable by a station. Crystal became rare ore in E1 (`01 §7`).
6. **Home systems** follow one template (see [starts-and-exploration.md](Design-Starts-and-exploration)). Beyond the home system, each empire gets two habitable planets of the same climate group within 8 cells.
7. **Regions:** every system belongs to one of 8 sectors (nearest of 8 seed systems by map distance, ties by lower id); sectors name the map and own the pirate loot pools. A system whose cell's cloud drag reaches 2.2 is flagged **nebula** (6–19 systems at Medium), with a sensor penalty of −2 jumps (−4 cells) for a source inside, minimum 0 (own system only).
8. **Points of interest:** anomalies are not placed at generation; they are rolled at survey time from the body's seed. Archaeology sites 1 per 25 systems, precursor sites 6 per galaxy (one chain), derelicts 1 per 40 systems, pirate pockets 1 per 50 systems at least 10 cells from any homeworld (the pocket's region starts at piracy 0.5 with a pirate base).
9. **Empire placement:** greedy farthest-point by map distance, ties broken by the habitable count within 4 cells. The player picks "random" or a specific start quality.

**Output:** systems, bodies, sectors, points of interest and the hyperspace grid with its clouds. At Medium: 400 systems, about 1,300 bodies, of which about 150 are habitable.

### The hyperspace grid

Hyperspace is a coarse grid over the galaxy's bounding box (`07 D43`, corrected by `07 D46`):

| Size | Cells across the world | Grid with the 12-cell margin |
|---|---|---|
| Tiny | 28 | — |
| Medium | 45 | 69 × 69 |
| Huge | 71 | 95 × 95 |

Cells are `d_min / 2` (22.5 units at Medium). The 12-cell margin on every side holds the outer depth bands. Each cell stores byte columns:

| Column | Range | Changes |
|---|---|---|
| `drag` | 1.0 to 3.0, the factor on travel time | static, from the generation clouds |
| `storm` | 0 to 1 | monthly weather |
| `current` | a direction and a strength 0 to 1 | monthly weather |
| `corridor` | an owner and a strength 0 to 1 | when corridors are built, drift or break |

### Weather

Weather is a pure function of `(seed, month)`.

- **Storms:** K drifting discs, K = systems / 10 (times the weather setting). Each storm slot lives through periods of 20 months; each life lasts 3–18 months from a seeded origin anywhere on the grid, drifts up to 1.5 cells a month, has a radius of 2–6 cells, and its intensity rises and falls over the life (`0.6–1.0 × sin`) (`07 D46`).
- **Currents:** a low-frequency field of three sinusoidal modes per component, with periods of 120, 180 and 240 months, strength at most 0.5 at Normal. It rotates and drifts over years. The cost factor from currents runs 0.75–1.25.
- **Caching:** the current and the next month are rasterised and cached. Cells a route enters after the next month take the next month's weather; the route's computed legs are what the simulation applies, so a route arrives on its computed day with its computed fuel and damage by construction (`07 D46`, correcting D43's "exact forecast").
- The weather moves on the first tick of a month. The monthly raster is budgeted at 2 ms.

### Travel

- **Departure.** A fleet leaves a system after a 3-day jump windup at the system edge, enters hyperspace and follows a route planned once, at departure, on the forecast grid. Arrivals appear at the destination's edge on the route's computed day. Neighbouring systems 45–90 units apart are about 5–9 days at speed 100 in calm space (`01 §10.2`).
- **The planner:** A* over the eight neighbours of a cell, with days as the cost, at the slowest ship's speed.
- **Cell cost:** `cell / (speed / 10) × (drag + storm) × (1 − 0.5 × current·heading)` days. Inside a corridor of strength `s`, the cost is `× (1 − 0.33 s)`, and drag and storm are scaled by `(1 − s)`.
- **Stances:** `direct` minimises days; `cautious` adds the expected storm damage at 10 days per 1% hull and each expected encounter at 50 days. `FleetRoute` sets a fleet's stance for its next trips.
- **Settled, never stepped.** The route's cells, entry days, arrival day, fuel and storm damage are computed at departure, stored in a shared route arena (a CSR column, no per-fleet `Vec`) and applied by settling on the route's events. A fleet in transit does no work on the ticks between them.
- **Fuel:** `Σ fuelPerJump / 5` per hyperspace day, times `1 + storm` and the depth factor. A fleet's tank holds `10 × Σ fuelPerJump`.
- **Storm damage:** `0.5% × storm` of hull per day, armour first, halved by shields, never taking hull below one point. Weather wears fleets down; battles destroy them.
- **Stranding:** the route's cumulative fuel beyond the fleet's fuel at departure turns the rest of the route to 25% speed. A fleet that departs dry crawls the whole route.
- **Re-planning:** a route never re-plans in flight. A new order re-plans from the fleet's current cell, with no windup when the fleet stands in hyperspace. A fleet may be ordered to a cell; a fleet stopped in hyperspace stands at the cell's place, `SystemId(1024 + cell)`.
- **Emergency jumps** after a lost battle stay M8's straight 3-day jump to the nearest own system by distance, with no route, so nothing meets a retreating fleet (`07 D46`).

### Intercepts

Each route's `(cell, day)` footprint, one per day it spends in a cell, is indexed in an ordered map. When a route is planned, the planner finds the earliest shared `(cell, day)` with a hostile footprint the fleet can sense and schedules an intercept there as a heap event, if either side is willing (`01 §10.3`). At the event the other fleet must be in that cell that day, else the fleet's next event is scheduled. The battle happens at the cell's place, and survivors take up their orders when it ends. Nothing is polled.

### Corridors

Corridors are the built structure of hyperspace, trade routes that the player builds and that the currents keep trying to bend.

| Aspect | Rule |
|---|---|
| Order | `BuildCorridor` between two own starbases (tech: Beacon Engineering) |
| Path | the route a freighter plans that day |
| Cost | 20 alloys and 2 parts per cell, over 6 days per cell, built as two halves at the two starbases, each drawing half the bill by the local rule of `01 §10.4` (parts became components in E1) |
| Strength | built cells take 1.0 for the owner; a newer corridor takes a shared cell |
| Drift | each month a cell loses `0.2 × shear × (1 − stabilisation)`, where `shear` is the change in the current across it since last month over the Normal strength (so Turbulent weather wears faster) |
| Break | a cell below 0.3 breaks (0.2 with Corridor Stabilisation) |
| Repair | a broken corridor stops charting its cells; `BuildCorridor` on the same pair re-surveys it at the same cost |
| Effects | speed and storm and drag cover (above), freight at half cost, +0.2 piracy security in the regions it crosses |
| CPU | decay is settled per corridor on its own staggered day of the month by a heap timer; every corridor event recomputes its owner's trade fields |

Megastructures add corridor holding: a Hyperspace Anchor stabilises every corridor within its radius and lowers depth around it, and a Beacon Spire holds nearby corridor cells (`07 D78`; see [megastructures.md](Design-Megastructures)).

### Tiers of travel

| Tier | Tech | Effect |
|---|---|---|
| 1 | Hyperspace Drive (physics; leads to the First Jump) | hyperspace travel |
| 1 | Beacon Engineering (engineering) | `BuildCorridor` |
| 2 | Corridor Stabilisation (physics) | corridor decay −50% (`stabilisation` 0.5), break threshold 0.2 |
| 3 | Warp Drive (physics) | a warp core, a utility component with power 6. A fleet whose every ship carries one charges 10 days at the system edge in place of the windup when its target is within 8 cells, then arrives the next day, ignoring drag, storms and intercepts (no footprints), at 3× the fuel of the straight distance's days |
| 4 | Gateway Theory (physics) | a Gateway megastructure at an own starbase of size 6 or more |

**Gateways** (`07 D46`, moved unchanged onto the megastructure model in S1):

- Cost 5,000 alloys and 800 parts from the starbase's market over 1,080 days; upkeep 30 credits and 4 parts.
- A fleet in a Gateway system ordered to another own Gateway system crosses in 1 day after the windup.
- A fleet may also cross between its own Gateway and the Gateway of an empire it holds open borders with, as between its own (`07 D114`).
- **Trade links** (`07 D283`, built in M19a on 2026-11-23): an empire's own hub pair with usable Gateways at both hubs' systems (its own, or an open-borders partner's) takes the jump's days when quicker than the flown route, with no piracy and no loss on the jump, freight paid on the shorter days only, and closed by a blockade at either gate. A Gateway serves the hub at its own system. Pact routes and the Galactic Exchange's legs still fly.

There are no natural wormholes: they were removed with the lanes, and Gateways are the buildable wormholes (`07 D46`).

### Distance replaces hops

With no lanes, every range that once counted jumps is measured in cells or route days (`07 D46`):

- Supply range and sensor range are radii in cells from the source's cell. Content keeps its ranges in jumps, at `cellsPerJump` 2.
- The trade network's freight is `amount × (routeDays + 2 × 2.5) × 0.05`, at half on a route's corridor share. A shipment lands its route days plus one later; the longest route is 60 days before the split at a hub.
- Routes are planned per hub by a Dijkstra field of route days at freighter speed 100 over the month's grid, through the empire's corridors. Each hub is recomputed on its own day of the month (its market id mod 30), so every field is at most a month old; its tick is saved so a loaded world recomputes it on that month's grid.
- Piracy is kept per region of 6 × 6 cells (12 × 12 regions at Medium with the margin, 16 × 16 at Huge). A route's exposure is the sum over the regions it crosses of level × days. A region's security is its best starbase's, its patrolling ships', 0.1 × the claimed share of its systems, and +0.2 with an intact corridor through it.
- Migration looks within 16 cells, weighted by `1 + cells / 2`.
- Sectors, empire placement, retreats and the automations' "nearest" use map distance.

See [trade-and-logistics.md](Design-Trade-and-logistics) for the trade network built on these routes.

### Deep space

Danger and reward rise together with distance from charted space (`07 D45`).

**Depth.** Each cell has a depth: its distance in cells to the nearest system or to a corridor cell of strength 0.3 or more. Corridors push charted space outward, and a breaking corridor lets the deep back in. Depth is computed once per galaxy and refreshed by one breadth-first pass when a corridor is built or breaks, never per tick.

| Band | Depth (cells) |
|---|---|
| charted | 0–2 |
| fringe | 3–5 |
| deep | 6–9 |
| abyss | 10 or more |

**Attrition** from depth `d` above 2, per hyperspace day:

| Effect | Amount |
|---|---|
| Readiness | `−0.5 × (d − 2)` |
| Fuel | `× (1 + 0.1 × (d − 2))` |
| Hull wear | `0.1% × (d − 2)`, which repair restores |

**Unknown threats.** Three deep factions are spawned as encounters, never as standing fleets. All three fight under the pirate row (hostile to everyone, no ledger), with their own hulls, aggressive; what is left of them goes when their battle ends (`07 D46`).

| Faction | Where | Hull multiplier |
|---|---|---|
| Pirate raiders | fringe and deep | 1× (raider hulls) |
| Derelict automata (precursor guardians) | only as a cache's guards, met by the fleet that enters the cache | 1.5× (`automaton`) |
| Void fauna | abyss | 2× (`void_fauna`) |

Encounter odds per day are `0.002 × (d − 2)²`, capped at 0.15. Strength is `power = 200 × (d − 2)` times the faction's multiplier, in as many ships as reach it. An encounter is rolled for each cell-day of a route when the route is planned, from a stream derived from `(seed, cell, day)`, scheduled like an intercept and fought through the combat resolver. A planned encounter re-rolls at its event on the depth of that day, so a corridor built in between can spare it. A `cautious` route weighs expected encounters.

**Rewards.** Sites are a pure function of `(seed, cell)`, with a chance per cell of `0.01 × (d − 2)`, at most 0.12, never at depth 2 or less. Site depth is taken from the galaxy's own depth (systems only), so building or breaking a corridor never makes sites appear or vanish (`07 D46`). Kinds are weighted cache 0.3, stash 0.4, field 0.3; a cache only from the deep band inward, else a stash.

| Site | Contents |
|---|---|
| Precursor cache | guarded by automata; one of six named precursor items half the time (a unique component or hull mod for the persona's fittings), else 600 research spread over the fields. Opened caches also give precursor clues (`07 D85`) |
| Stash | a derelict destroyer at half hull (20%), a load of 50–200 into the hold (30%), or 200–800 credits |
| Rich resource field | a deposit of amount 6–10 of ore, gas, crystal or research, mined by a deep outpost |

- **Finding sites:** a site is found when a fleet's route passes within its sensor radius in cells, or by a `DeepSurvey` order: a science ship holds a cell for 20 days and reveals every site within 3 cells. A found site is looted or claimed by a fleet entering its cell.
- **Deep outposts:** `BuildOutpost` loads the bill (120 alloys, 10 parts) at the own starbase market where the construction ship stands when ordered. The ship flies to the found field and builds for 90 days. The hold fills at 6 per deposit unit per 30 days, up to 500, and a freighter standing at the outpost loads what fits and unloads where it docks. Nothing teleports (`07 D39`). A research field reports to the ledger like a research station.
- **Ruins:** megastructure ruins sit at 6% of systems, 80% of them in cells of depth 6 or more; deep ruins are claimed by beating automata (`07 D78`, `D79`; see [megastructures.md](Design-Megastructures)).

### The map layer

The hyperspace layer is one texel per cell (clouds violet, storms amber, corridors cyan, the deep darker), uploaded when the month or a corridor changes. Routes are drawn as lines from where each fleet is, trade flows along their legs' cells, the piracy haze per region; the system panel shows its cell's drag, storm and corridor (`07 D46`). G4 blends the storms toward next month in the shader. The fleet-centred flight view is described in [flight-view.md](Design-Flight-view).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| No hyperlanes | A dangerous hyperspace crossed from any system to any other, as in Starsector; replaces the hyperlanes | 2026-11-12 | `07 D43` |
| Simulation model | A coarse grid with monthly weather; routes planned at departure with arrival and hazards computed then (live storms with free flight rejected) | 2026-11-12 | `07 D43` |
| Strategic structure | Built beacon corridors bent by shifting currents; higher techs counter with stable corridors, warp or wormholes | 2026-11-12 | `07 D43` |
| Late travel | All three tiers plus buildable Gateways as a fourth tier | 2026-11-12 | `07 D43` |
| Deep space | Danger rising with depth; three deep factions; caches, stashes and rich fields with deep outposts; built in M8c | 2026-11-12 | `07 D45` |
| Grid size | Cells of `d_min / 2` with a 12-cell margin (69 × 69 at Medium); replaces D43's "about 128 × 128 cells" | 2026-11-13 | `07 D46` |
| Piracy regions | 6 × 6 cells; replaces D43's 16 × 16 | 2026-11-13 | `07 D46` |
| Long routes | Cells after the next month take the next month's weather; replaces D43's exact forecast | 2026-11-13 | `07 D46` |
| Trade fields | Each hub recomputed on its own day of the month; replaces D43's one hub a day | 2026-11-13 | `07 D46` |
| Freight | `amount × (days + 2 × 2.5) × 0.05`, half on corridor share; replaces D43's `amount × routeDays × 0.05` | 2026-11-13 | `07 D46` |
| Storms slow | A storm adds its strength to the drag | 2026-11-13 | `07 D46` |
| Clouds after empires | Clouds rolled after placement and moved off homes | 2026-11-13 | `07 D46` |
| Natural wormholes | Removed; Gateways are the wormholes | 2026-11-13 | `07 D46` |
| Allied Gateways | Read as open borders, since alliances were 1.0 at the time | 2026-11-14 | `07 D114` |
| Hyperspace Anchor | Holds every corridor in its radius, not only its owner's | 2026-11-14 | `07 D78`, `DESIGN.md` |
| Gateway trade | A route shortcut for an empire's own hub pairs, as the Mass Driver's | 2026-11-23 | `07 D283` |

## Not yet built or still open

- The `07 D46` call set (M8c's calls beyond the spec) is listed in `DESIGN.md` as still open for the owner's review.
- The `01` passages that M8c converted from hops by ratio (5 route days per old hop, 2 cells per jump; listed at the end of `07 D46`) await the owner's eye.
- `05 M8c`'s bench box (a route plan across Huge ≤ 200 µs × calibration, the monthly raster ≤ 2 ms, the full tick within 5% of M8b's) is unchecked: the full tick was 15–17% over M8b's, carried to P1.
- No AI builds Gateways yet; the AI's megastructure advisor is M19c (`07 D283`).
- Storm suppression in the third age and storm-forecast techs are listed in the tech tree rework as future hooks (`07 D278`).

## Discrepancies noted

- **Allied Gateways.** The `01 §2.1` tiers table still says "allied Gateways join with alliances (`05 M10`)". `07 D114` (2026-11-14) reads this as open borders, because alliances were 1.0 then. Alliances were built in M18a (merged 2026-11-23, `07 D279`). The sources do not say whether alliance members' Gateways now join as well as open-borders partners'.
- **Commodity names in hyperspace rules.** `01 §2` step 5 and `01 §2.1` still name `crystal` deposits and `parts` in the corridor, Gateway and outpost bills, while `01 §7` records that crystal became rare ore and parts became components in E1 ("saves map them in place"). The numbers are unaffected; the names in `01 §2`–`§2.1` predate E1.

## Sources

- 01 §2 Galaxy generation
- 01 §2.1 Hyperspace
- 01 §10.2 Movement and logistics
- 07 Decisions: D39, D43, D45, D46, D78, D79, D85, D114, D278, D279, D283
- 05 Milestones: M8c, S1, M19
- 00 Status (M8c verification; M19a merge, 2026-11-23)
- 13 Ships, fleets, megastructures (Hyperspace Anchor, Beacon Spire)
- DESIGN.md (pillars "Galaxy and hyperspace" and "Deep space"; open questions)

## Related pages

- [Overview](Overview)
- [starts-and-exploration.md](Design-Starts-and-exploration)
- [flight-view.md](Design-Flight-view)
- [trade-and-logistics.md](Design-Trade-and-logistics)
- [starbases.md](Design-Starbases)
- [ships-and-fleets.md](Design-Ships-and-fleets)
- [combat.md](Design-Combat)
- [megastructures.md](Design-Megastructures)
- [diplomacy-and-war.md](Design-Diplomacy-and-war)
- [progression.md](Design-Progression)
- [events-and-crises.md](Design-Events-and-crises)
- [architecture-and-performance.md](Design-Architecture-and-performance)


---

# Starts and exploration

**Status:** built, as of 2026-11-23. The standard origin and exploration in M3–M8c, the Sol start in M9, first contact, archaeology and the precursor chain in M11, shared sensor cover for allies in M18a.

Every empire begins as a fledgling near-space civilisation with no faster-than-light travel: a homeworld, three small off-world colonies and four resource stations in one home system. The home system is the first map, and interstellar flight is earned by exploiting it and researching Hyperspace Drive, which leads to the First Jump. The player can instead start in a realistic Sol system. After the First Jump, science ships explore outward: systems move from unknown to sensed to surveyed, surveys roll anomalies, and sensors set what each empire can see.

## How it works

### The home system template

Every generated homeworld system follows one template with randomised sizes, names and surface seeds, orbits 1–7 from the star (`01 §2` step 6):

| Orbit | Body | Size | Deposits and features |
|---|---|---|---|
| 1 | hot barren world | 6–8 | energy 3 |
| 2 | **homeworld**, with a barren moon | 16 (moon 5) | the moon's 7 hexes hold at least one `ore_deposit` and two hills |
| 3 | cold rocky world, base habitability 30 until terraformed | 10 | the centre ring holds a `crystal_field` and an `ore_deposit`, one hazard outside ring 0 |
| 4 | asteroid belt | — | ore 4, crystal 2 |
| 5 | gas giant with two ice moons (sizes 4 and 6) | — | gas 4; one moon gas 2, one moon research 2 |
| 6 | ice giant | — | gas 3, research 1 |
| 7 | outer ice body | 6 | research 2 |

Moons carry a `parent` body and share its orbit. Beyond the home system, each empire gets two habitable planets of its climate group within 8 cells. Crystal became rare ore in E1 (`01 §7`, `docs/14`), and the opening below uses the E1 names.

### The standard starting state

| Item | Value |
|---|---|
| Homeworld (orbit 2) | size 16, habitability 100 for the founding species; surface rings 0–2 (19 hexes) surveyed; ring 1 forced to contain 2 fertile basins, 2 hills, 1 geothermal, 1 gas vent. Districts: 2 city, 3 agriculture, 2 mining, 1 generator, 1 industrial (housing 24), placed by the template with zero adjacency and every district served. Buildings: Capital I, Research Lab, Foundry, Factory, Metal Refinery. Jobs 30: 4 clerks, 6 farmers, 4 miners, 2 technicians (workers); 4 artisans, 2 metallurgists, 2 metal refiners, 2 researchers, 2 bureaucrats (specialists); 2 administrators (rulers). Designation Capital |
| Highport (orbital habitat at the homeworld) | habitability 70; Habitat Core (housing 10, amenities 4); 1 city district (housing 5); Spaceport. Jobs 4: 2 clerks, 2 merchants |
| Moon base | domed colony, habitability 30; Dome Core (housing 6, amenities 4); 1 mining district (housing 2); all 7 hexes surveyed. Jobs 2: 2 miners |
| Cold rocky colony (orbit 3) | domed colony, habitability 30; Dome Core; 1 mining district, 1 rare-ore mining district (housing 3); centre ring surveyed. Jobs 3: 2 miners, 1 rare-ore miner |
| Pops | 39: 23 worker, 14 specialist, 2 ruler; one cohort per stratum per colony; skill 0.5 |
| Stations | asteroid belt ore (4/mo) and rare ore (2/mo), gas giant gas (4/mo), ice giant gas (3/mo); all deliver to the home Station's market, which offers them on to the homeworld by in-system shipment (1 day) |
| Starbase | a size-2 Station at the homeworld's star, a small colony with its own market and no crew at start |
| Ships | 1 science, 1 construction, 2 corvettes (auto-design); fuel tanks full; no hyperspace travel until the First Jump |
| Leaders | Ruler level 2; Scientist level 1 assigned to the science ship (both hired traitless from data, `07 D85`) |
| Techs | all 6 tier-0 techs known; Hyperspace Drive is a tier-1 physics tech |
| Ledger | 500 credits, 100 influence, 0 unity, 0 research; energy backing at Standard |
| Market stock | homeworld: 300 food, 400 ore, 100 gas, 150 alloys, 100 goods, 30 luxuries, 20 components, 50 refined fuel, 40 rare metals, 20 rare ore, 100 energy. Each off-world colony: 5 food, 10 goods, 20 ore (deliberately thin, so the in-system supply line starts on day 1) |
| Visibility | homeworld, moon, cold rocky world, asteroid belt, gas giant and ice giant surveyed; the hot barren world, the gas giant's moons and the outer ice body unsurveyed (the science ship's first job); systems within 4 cells (2 jumps) sensed by telescope, none surveyed |

All output multipliers start at 1.0: every cohort at happiness 50 and skill 0.5, every colony at stability 50, every market at base prices, policies at Taxation Standard, Commerce Credits, Education Standard and immigration Selective.

### The opening balance

Per 30 days, applied daily at 1/30 and asserted by the headless runner over the first 30 ticks.

| Check | Value |
|---|---|
| Housing | homeworld 44 for 30 pops; Highport 15 for 4; moon 8 for 2; rocky 9 for 3 |
| Jobs | 39 = pops, by stratum on every colony |
| Food | 6 farmers × 7 = 42; eaten 41 → **+1** on day 1, **+3** by day 30 |
| Goods | **+6.75** (early export surplus) |
| Ore | 36 produced, 31.2 used → **+4.8** |
| Gas | **+7**, no consumer at the start |
| Rare metals | **+2.6** (40 in stock) |
| Energy | **+30.9**, half backed at the treasury |
| Alloys, luxuries | +6; −1 |
| Refined fuel | about 14 months of stock |
| Credits | **+8.2** (+9.45 running from the first month) |
| Freight | capacity 295; in-system shipments spend about 4 |
| Stability targets | homeworld 74.5, Highport 66, moon 54, rocky 53 |
| Research | **7 per field**; Focused on physics gives 10.2 physics points per 30 days, so Hyperspace Drive (168 points) lands at about 17 months; Equal about 24 months |

**The sanity line** asserted over the first 30 ticks: housing ≥ pops + 4 on every colony; jobs = pops on every colony; empire food balance ≥ +1 on day 1 and ≥ +3 at day 30; goods and ore balance ≥ 0; energy balance ≥ +20; refined fuel ≥ 4 months of demand at day 30; credits ≥ +3, the first issuance included; stability ≥ 45 on every colony at day 30; births − deaths > 0 everywhere; no pop migrates except into the home Station's two crew jobs; vitality stays 1.0 and private wealth is 11.9 ± 0.3 at day 30 (the homeworld's 7.3); luxury stock ≥ 24 months of demand; every off-world market receives a food shipment by day 5 and no market's food satisfaction drops below 1; the homeworld receives ore from both domes by day 3.

See [economy-and-resources.md](Design-Economy-and-resources) and [colonies-and-surfaces.md](Design-Colonies-and-surfaces) for the rules behind these lines.

### The Sol start

A new-game option, `GameSettings.start = sol`, for the player's empire only (`07 D41`). The standard origin stays the default, and AI empires keep generated home systems. Sol takes the player's home position in hyperspace as any home system does. Body names are fixed (the system shows "Sol"), and the named major bodies are drawn with fixed NASA/USGS public-domain textures (`07 D41`, built in G3). The founding species is the player's choice, offered a Human preset (continental preference, no traits).

| Body | Orbit | Class, size | Deposits | At start |
|---|---|---|---|---|
| Mercury | 1 | molten, 6 | ore 5, energy 4 | — |
| Venus | 2 | toxic, 14 | gas 4 | — |
| Earth | 3 | continental, 16 | — | homeworld, as the standard origin's |
| Luna (Earth) | 3 | barren moon, 6 | ore 3 | moon base |
| Mars | 4 | arid cold rocky, 9 (habitability below 40, so a dome) | ore 4, rare ore 2 | the domed colony in place of the cold rocky colony |
| The Belt (Ceres) | 5 | asteroid belt | ore 6, rare ore 3 | ore and rare-ore stations |
| Jupiter | 6 | gas giant, 30 | gas 6, research 3 | gas station |
| Io, Europa, Ganymede, Callisto | 6 | molten 6; frozen 6; frozen 8; frozen 7 | energy 3; research 4; ore 3; ore 2 | — |
| Saturn | 7 | gas giant, 28 | gas 5 | gas station |
| Titan (Saturn) | 7 | toxic moon, 8 | gas 4 | — |
| Uranus | 8 | gas giant, 25 | gas 3 | — |
| Neptune, Triton | 9 | gas giant, 25; frozen moon, 5 | gas 3; ore 2 | — |

**The Sol opening** (M9, `07 D56`). The colonies, jobs, stock, ledger, ships, starbase, leaders and techs are the standard origin's, placed in Sol: Earth with the homeworld layout and Highport and the size-2 Station, the moon base on Luna, and the domed colony on Mars with the cold-rocky districts and forced hexes. The four stations sit on Sol's richer deposits: the Belt's ore 6 and rare ore 3, Jupiter's gas 6 and Saturn's gas 5. Earth, Luna, Mars, the Belt, the four giants and Jupiter's moons start surveyed, leaving four bodies for the science ship: Mercury, Venus, Titan and Triton. Neptune sits at orbit 9, one past the system edge (orbit 8).

| Check | Value (per 30 days) |
|---|---|
| Housing | as standard: Earth 44 for 30 pops, Highport 15 for 4, Luna 8 for 2, Mars 9 for 3 |
| Food, goods | as standard: **+1** (+3.1 at day 30), **+6.75** |
| Ore | Earth 16 + Luna 8 + Mars 8 + Belt station 6 = 38; used 31.2 → **+6.8** |
| Gas | Jupiter 6 + Saturn 5 → **+11** |
| Rare ore, rare metals | 4 rare ore; the Metal Refinery uses 3 → rare metals **+2.6**, rare ore +1 |
| Energy, refined fuel, alloys, luxuries, credits, stability, freight | as standard (freight: in-system shipments about 32 units, spending about 4) |
| Research | as standard, 7 per field; the player's headless First Jump median is +11% on the standard origin's (seeds 1–6: 798.5 against 717.5 ticks) |
| Unworked deposits | Mercury ore 5 and energy 4; Venus gas 4; Luna ore 3; Mars ore 4 and rare ore 2; Jupiter research 3; Io energy 3; Europa research 4; Ganymede ore 3; Callisto ore 2; Titan gas 4; Uranus gas 3; Neptune gas 3; Triton ore 2 |

The standard sanity line holds on the Sol start unchanged (`economy::sol_opening_passes_the_sanity_line`), and the First Jump pacing sits inside the required ±20% of the standard origin.

### Dawn: the pre-FTL era

- **No hyperspace travel at start.** Fleets move between the bodies of a system but cannot enter hyperspace until the empire knows **Hyperspace Drive** and has completed the **First Jump** special project. The project takes 60 days; it starts as soon as the tech is known and a science ship is at the system edge, by standing order or by an explicit `ScienceResearch` command. Hyperspace is refused empire-wide until it completes. The flight view is greyed out with a reason before FTL (M-QA2, 2026-11-20).
- **Hyperspace Drive's place in research.** Since the tech tree rework (`07 D278`, built as T1 on 2026-11-23), there is no research draw: Hyperspace Drive is every new empire's first research goal, and it carries an explicit cost of 150 so that the First Jump stays at about two years. This replaces `01 §4.1`'s rule that it is "always among the first three physics options offered".
- **In-system movement.** Within a system a fleet is at a body, at the starbase or at the edge; moving between two of these takes `1 + |orbit_a − orbit_b|` days (the edge is orbit 8; moons share their parent's orbit; the starbase sits at the homeworld's orbit) and uses no fuel.
- **Telescopes.** Systems within 4 cells are sensed from day 1 (star class, large bodies), so the player sees the neighbouring systems before they can reach them.
- **Three kinds of colony:**

  | Kind | Where | Rules |
  |---|---|---|
  | Surface colony | a habitable world | the homeworld; later the two guaranteed habitable planets nearby |
  | Domed colony | any body whose base habitability for the species is below 40 | habitability fixed at 30 (Domes II: 45); founded by a colony ship with a free Dome Core (housing 6, amenities 4, +1 building slot); no agriculture districts; consumption +40% |
  | Orbital habitat | around any body in an owned system, built by a construction ship (`BuildHabitat`) | 300 alloys + 50 parts from the system's markets, 180 days; its own body row (kind `habitat`), size 4 with 7 deck hexes; habitability 70 for every species; free Habitat Core (housing 10, amenities 4, +1 slot); city and industrial districts only (caps 2 and 1); opens with 1 worker pop moved from the nearest own colony in the system |

- **In-system logistics.** Every colony is its own market. Shipments between markets of the same system take 1 day, spend 2.5 days of freight (`amount × 0.125`), cross no piracy region and are never blockaded from outside.
- **What the era asks:** survey the remaining bodies, put a station on every deposit, found domes and habitats where the deposits are, build the home system's internal economy (food and goods outward; ore, gas and crystal inward), and research Hyperspace Drive. AI empires follow the same path: the Expansion advisor targets in-system bodies first, and the AI research split is Focused on physics until the First Jump, then Equal (`07 D55`).

### Visibility and fog

Each empire holds a visibility state per system:

| State | Meaning |
|---|---|
| unknown | nothing known |
| sensed | within sensor range of an owned starbase or fleet (or a telescope); shows the star and fleet blips |
| surveyed | all bodies, deposits and anomalies known |

Fog never reveals fleets outside sensor range. Intel on foreign empires (fleet strength, economy) is coarse without a comms or espionage treaty (espionage is 1.0). A dirty empire recomputes its whole cover from its sources on the ticks a source moved, arrived or was built, with no per-tick work otherwise (`07 D23`).

**Sensor ranges** are radii in cells from the source's cell, written in content in jumps of 2 cells each; the maximum over sources applies (`07 D46`):

| Source | Range (jumps) |
|---|---|
| Military fleet | 1 |
| Science ship | 2 |
| Starbase | by size, 1–5 |
| Tech | +1 |
| Source in a nebula | −2, minimum 0 (own system only) |

Routes also find deep sites within their sensor radius (see [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)). The nebula's old "fleets hidden unless adjacent" rule was dropped with the lanes (`07 D46`).

**Shared cover** (M18a, `07 D279`, merged 2026-11-23): allies share sensor cover, each seeing what the other's starbases and fleets sense. It is read through an unsaved alliance mask rather than rescanned from the allies' sources, and each empire's own recompute raises the systems it covers to sensed for its allies.

### Survey and anomalies

- **Survey time** per body: `days = 8 + 0.4 × size` (gas giants 12, asteroid belts 6), −5% per level of scientist skill.
- **Anomaly chance** per surveyed body: `p = 0.18` (× 1.5 if the system has a research-related modifier). Anomaly level 1–10, weighted toward low early.
- **Research project:** `days = 20 × level / (1 + 0.1 × skill)`. Completion fires a weighted event from that anomaly's pool (small resource, deposit reveal, tech boost, chain start, special project). There is no failure outcome; bad outcomes are rare and event-driven.
- A ship that finds an anomaly while auto-exploring researches it at once and then resumes exploring (`07 D23`).

**Auto-explore** (standing order, on by default when idle): the nearest unsurveyed system by map distance, excluding systems with hostile fleets above a threat threshold and systems inside closed borders. Ships flee hostiles automatically (evasive stance). A path may cross systems the empire has not sensed. Auto-explore is one of the three automation defaults on the settings screen (`07 D145`; see [time-and-modes.md](Design-Time-and-modes)).

**Deep survey:** a science ship holding a hyperspace cell for 20 days reveals every deep site within 3 cells.

### Stations and claims

- **Stations** (construction ship, on a surveyed deposit in an owned system): 60 alloys drawn from the system's markets by the local rule, 60 days, upkeep 1 credit/mo. A station yields `amount` units per 30 days of its deposit kind, delivered daily to its system's starbase market, from which in-system shipments carry it to the colonies. Nothing is delivered to another system except by trade (`07 D39`). Energy deposits yield 2 energy per unit; research deposits yield 1 research point per unit split equally across fields.
- **Claims:** a system is claimed by founding a starbase there, from a colony in the system or directly by a colony ship. Construction ships do not claim. See [starbases.md](Design-Starbases).

### Archaeology, first contact and the precursor chain

| Feature | Design rule (`01 §4.2`) | As built in M11 |
|---|---|---|
| Archaeology sites | 3–6 chapters; each takes `30 × difficulty / (1 + 0.15 × skill)` days, then rolls `d10 + skill ≥ difficulty` (difficulty 2–6); failure costs another attempt; the final chapter pays a relic, tech or event chain | 3 chapters at difficulty 4, so a skill-3 scientist digs 3 × 83 = 249 days, the roll always passing (`07 D86`) |
| First contact | sensing an unknown empire's ship or starbase starts a 60-day contact project per empire pair; completion opens comms (contact events with choices are 1.0) | starts in the sensor recovery pass, 60 days per pair; comms open both ways as a ledger bit (`07 D85`) |
| Precursor chain | 6 clues from survey and anomaly outcomes, each a special project; the sixth reveals the precursor home system (relic and unique system modifier) | clues from the six precursor sites and every opened deep cache; the home is the lowest-id precursor site's system (`07 D85`) |

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Home system as the first map | Empires start pre-FTL with a homeworld, three off-world colonies and four stations; exploiting the home system precedes interstellar flight | 2026-11-01 | `07 D20`, `01 §4.1` |
| Sensor recomputation | A dirty empire recomputes its cover on the ticks a source changes | 2026-11-02 | `07 D23` |
| Sol start offered | A new-game option for the player; standard origin stays; AI keeps generated homes | 2026-11-11 | `07 D41` |
| Sol's numbers | Realistic Sol, rebalanced with its own opening table and the same sanity line (the recommended mapping onto the standard template was not chosen) | 2026-11-11 | `07 D41` |
| Sol textures | NASA/USGS public-domain imagery, 512 × 256 WebP for the named major bodies | 2026-11-11 | `07 D41` |
| Sol rebalance | Same colonies, richer stations; Mars `arid` size 9; Neptune at orbit 9; four bodies left unsurveyed; First Jump +11% accepted | 2026-11-13 | `07 D56` |
| Ranges in cells | Sensor and supply radii in cells at 2 cells per jump; nebula adjacency rule dropped | 2026-11-13 | `07 D46` |
| AI opening | AI research Focused on physics until the First Jump, Hyperspace Drive weighted | 2026-11-13 | `07 D55` |
| M11 exploration content | Contact in the sensor recovery pass; archaeology at 3 chapters, difficulty 4; precursor clues from sites and deep caches | 2026-11-14 | `07 D85`, `D86` |
| Hyperspace Drive in research | No draw; Hyperspace Drive is every new empire's first goal, cost 150, First Jump at about two years; replaces the fixed slot among the first three options | 2026-11-21 | `07 D278` |
| Allied sensors | Allies share sensor cover through an unsaved mask (the owner's fix A) | 2026-11-22 | `07 D279` |

## Not yet built or still open

- Contact events with choices (hostile, cautious, open) are 1.0. Espionage is 1.0.
- Origins other than Standard are not in MVP (`01 §3`). The owner's post-MVP direction is persona origins rather than planet origins (`07 D230`).
- Post-MVP discovery (`docs/17`, `07 D230`): sites placed at generation with reveal, dig and deep dig, a fragment codex, relics as items or planet bonuses, and exploration by teams or the persona. This would change the current rule that anomalies are rolled at survey time.
- The shared-sensors federation law is part of M18d (`07 D279`), not yet built.
- Free-text empire and species names are a hook; names come from each species' list (`07 D97`).

## Discrepancies noted

- **Sol opening job count.** `01 §3.1`'s Sol opening table gives "Jobs: 37 = pops, by stratum on every colony", while the same section says the colonies and jobs are the standard origin's, its housing row lists 30 + 4 + 2 + 3 = 39 pops, and the standard table gives "Jobs: 39 = pops".
- **Sol deltas.** `07 D56` (2026-11-13) says "Every §3 line but ore (+6), gas (+3) and crystal (4) holds as written". The current `01 §3.1` table gives ore +6.8 (standard +4.8), gas +11 (standard +7), and no crystal line (crystal became rare ore in E1); it cites `07 D162` (2026-11-16), so the table appears to be the later restatement, but the two do not match.
- **Asteroid belt deposits.** `01 §2` step 6 gives the home asteroid belt "ore 4 and crystal 2 deposits", while `01 §3` gives its stations as "ore (4/mo) and rare ore (2/mo)". `01 §7` records that crystal became rare ore, so these likely agree, but `01 §2` still uses the old name.

## Sources

- 01 §2 Galaxy generation (step 6, home systems)
- 01 §3 Empires and species
- 01 §3.1 The Sol start
- 01 §4.1 Dawn
- 01 §4.2 Interstellar exploration
- 01 §10.2 Movement and logistics
- 07 Decisions: D20, D23, D39, D41, D46, D55, D56, D85, D86, D97, D145, D230, D278, D279
- 05 Milestones: M3, M9, M11, T
- 00 Status (M-QA2, 2026-11-20; M18a merge, 2026-11-23)
- 14 Resource model (crystal to rare ore)
- DESIGN.md (pillar "Starts")

## Related pages

- [Overview](Overview)
- [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)
- [time-and-modes.md](Design-Time-and-modes)
- [colonies-and-surfaces.md](Design-Colonies-and-surfaces)
- [economy-and-resources.md](Design-Economy-and-resources)
- [starbases.md](Design-Starbases)
- [civilisations-and-species.md](Design-Civilisations-and-species)
- [progression.md](Design-Progression)
- [events-and-crises.md](Design-Events-and-crises)
- [diplomacy-and-war.md](Design-Diplomacy-and-war)
- [ai-and-victory.md](Design-Ai-and-victory)
- [flight-view.md](Design-Flight-view)


---

# Flight view

**Status:** built, as of 2026-11-23. F1 (merged 2026-11-15, save v23), with the persona hooks wired by M12a.

The flight view is a Starsector-style, fleet-centred mode of the galaxy map for interstellar travel, beside the top-down strategic map. The player's fleet sits at the centre of the hyperspace field (weather, currents, corridors, depth) and is steered by commands. Travel has two modes with their own truth: simulated travel, used whenever the view is closed and by every AI fleet, rolls incidents from the conditions of each cell; piloted travel, for the fleet the player flies in the view, rolls no dice and meets what is there: storms it flies into, hostile groups that chase it once sensed, and points of interest it chooses to visit.

## How it works

### Foundation: the route model

The flight view sits on M8c's hyperspace (see [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)). A route is planned once at departure by A* on the forecast grid, stored in the route arena with its entry ticks, fuel, storm damage, wear and readiness, and applied by settling on its events, never stepped. The view therefore presents and commands the route; it never moves the fleet itself. Nothing in the view changes simulation state: all input becomes `Command`s at tick boundaries.

### The view

A close, fleet-centred mode of the galaxy map, a screen like the battle screen (`07 D120`):

| Aspect | Rule |
|---|---|
| Window | about 41 × 41 cells (about 900 map units at Medium), zoomable between 21 and 81 cells on the wheel; the camera eases after the fleet |
| The medium | the weather field (drag, storm and next month's storm blended across the month), currents as drifting arrows, corridors with strength and owner colour, depth bands darkening toward the abyss, nebulae |
| Places | systems and starbases in the window; found deep sites and outposts; signals |
| Subject fleet | its sprite moving smoothly on its route (interpolated between days), the route ahead with day ticks and the arrival day, its sensor ring |
| Other fleets | those within sensor range, with their routes when visible, and a marker on a scheduled intercept with a sensed hostile |
| Route marks | each day coloured by danger: whichever is higher of the storm and the encounter odds |
| Hazards | found sites and signals as gold diamonds; met hostile groups ringed in red, with a line to where they are heading |
| No spoilers | encounters are pre-rolled from the seed, so the view shows odds and never outcomes; a contact chasing the fleet is shown once it appears |

**The HUD** (as built, `05 F1`): fuel and days left, readiness, hull, the depth band, the storm, today's odds and the arrival. A piloted fleet rolls no incident dice; the simulated odds are shown for comparison. The *Pause on incident* toggle lives in the HUD.

### Controls

Click to move, plus nudge keys (`07 D121`):

| Input | Command |
|---|---|
| Click a cell | `FleetMove { to: Place::Cell(c) }` |
| Click a system or starbase | `FleetMove` to that system |
| Right-click | a move to the fleet's current cell: the fleet halts in hyperspace |
| WASD or arrows | one command per key press: a move to the cell 4 cells ahead in that direction, from where the fleet is now |
| Hover | plots the route the planner would take, its days, fuel and the strip of odds, through `Query::Plot { fleet, cell }` on the query channel, answered by the simulation's own planner on the live world and never committed; asked once per hovered cell change |
| `R` | toggles the route stance between `direct` and `cautious` |
| Tab | cycles the player's fleets in hyperspace |
| Esc | returns to the map, centred on the fleet |
| F (on the map) | flies the persona's fleet (`07 D137`) |

Held-key steering every tick was rejected because it is non-deterministic under frame timing and would re-plan a route every day.

### Time

The view runs on the strategic clock at the game speed the player has set (`07 D122`). There is one world and one clock; opening the view changes nothing. Incident popups pause the clock exactly as other event popups do.

### Simulated travel

Simulated travel covers every journey with the view closed, every AI fleet, and the player's fleets the player is not flying (`07 D123`).

- **Encounters:** M8c's intercepts and deep encounters, unchanged (`0.002 × (d − 2)²` a day, capped at 0.15, with deep-faction strength by depth).
- **Incidents are events.** Each hyperspace day of a route rolls an incident chance from the cell's conditions. On a hit, an event with the trigger `on_hyperspace` is drawn, by weight, among those whose conditions pass.
- **The odds:**

  `p = min(base × (1 + 2 × storm + 0.5 × (drag − 1) + 0.25 × max(d − 2, 0)) × (1 − 0.5 × corridor), cap)`

  with `base` 0.01 and `cap` 0.08, all of it data in `exploration.json` `hyperspace.incident`. A 20-day trip through charted calm runs about 18%; through a storm on the fringe, about 50%.
- **When it is rolled:** like encounters, a pure function of `(seed, cell, day)`, found when the route is planned. `hyper::next_event` schedules `EV_INCIDENT` on the heap, with no polling. Firing it does not change the route.
- **Who:** every empire's fleets under the same rules; the AI takes each event's default option. Pirates and deep factions roll none.

### Piloted travel

Piloted travel covers the fleet the player flies in the view. There are no incident dice; what happens comes from what is there, and flying well is how trouble is avoided (the owner's call, 2026-11-14).

- **Storms** hurt when flown into, through M8c's existing per-day storm damage (`0.5% × storm` of hull, armour first, halved by shields) and fuel `× (1 + storm)`.
- **Hostile groups are placed in space** (`07 D123` option A, the spec session's reading of the owner's call):
  - **Placement:** pirates and deep factions as a pure function of `(seed, cell, month)`, with density by depth (`patrolOdds × (d − 2)` per cell and month, up to `patrolMax`) and strength by depth as for encounters. None stand in charted space.
  - **Sighting:** a piloted route's sight event (`EV_SIGHT`) falls on the first day its sensor range reaches a group. The groups in sight then materialise at their home cells and are recorded for the month (the ledgers' `pmonth`, `pcells` and `pfleets`); a defeated group stays gone until the next month's placement; those far out of sight are dropped.
  - **Noticing:** a group notices the fleet only if its route passes within `patrolSense` cells (3, less than most sensor ranges) at a cell the group can reach while the fleet is there. It moves there and waits; otherwise it gives up where it is.
  - **The meeting:** a group waiting on the route is a rival there, so M8c's intercept decides the meeting. Every re-plan of the piloted fleet re-plans the chase, so a player who sees a group ahead can route around it before it notices.
  - **As built:** groups wait at their home cell for the month rather than patrolling, and a group that cannot catch the fleet gives up and is dropped once out of sight rather than on a 10-day timer.
  - **Cost:** groups are materialised only on the piloted fleet's route events, re-plans and month changes, and only for cells within its sensor range. No per-tick work.
- **Points of interest (`signal` sites):** a site kind placed as a pure function of `(seed, cell)` off the systems' cells (`signalOdds`), found within sensor range as other sites are. Flying into one fires an `on_hyperspace` event drawn among those whose conditions pass in its cell (`07 D124`).

### What incidents and signals do

Effects come through the events engine (`07 D124`):

- the existing effects: ledger, stock into the owner's capital, flags, chained events, clues, leaders;
- one new effect, `fleet`, acting on the scope's fleet, with operations `fuel`, `hull` and `readiness` (each a ± share), `survey` (reveals the sites within `amount` cells) and `ambush` (halts the fleet and spawns the band's deep faction at its cell, opening a battle as an encounter does).

**Scope:** the owner, the cell's place (`SystemId(1024 + cell)`), and the fleet in `other`.

**Condition stats:** `hyper.depth`, `hyper.storm`, `hyper.drag`, `hyper.corridor`, `fleet.ships`, `fleet.fuel`, `persona.aboard`.

**No incident costs fuel.** A route's fuel is fixed when it is planned, so an incident that burned fuel mid-route would empty a tank whose planned speed did not change.

**The eight MVP incidents** (`events/hyperspace.json`), shared by both modes: the simulated table for un-piloted fleets, and the content of the signals a piloted fleet flies into.

| Incident | Where | Effect |
|---|---|---|
| Ion squall | in a storm | hull or readiness |
| Slipstream eddy | — | fuel back |
| Derelict signal | from the fringe outward | chart the area, strip the wreck and meet its guards, or hold course |
| Distress beacon | charted | answer for credits and unity, at some readiness |
| Sensor ghosts | deep | readiness |
| Drag shoal | clouds | hull |
| The void song | abyss | physics research at a cost in readiness, or shut it out |
| Beacon chatter | in a corridor | readiness back |

### Piloting, entering and leaving

- **The piloted state** (`07 D125`): the command `FleetPilot { fleet, on }`. Opening the view on a fleet pilots it; closing the view or switching fleets lets it go. One piloted fleet per empire is kept as the ledger column `piloted`, so piloting another fleet lets the first go.
  - The command applies before the day's timers; a piloted fleet's roll already scheduled for today lapses at its event.
  - A fleet let go rolls again from tomorrow, so a toggle never fires a day's roll twice; its next route event is found again and taken if it comes sooner.
  - Because the switch is a command, both modes stay deterministic and a run repeats exactly.
- **Entering** (as built): the fleet orders' *Flight view* button (also on a fleet in a system, to plot a course out, centred on its system's cell), a double-click on the player's fleet in hyperspace on the map, or F on the map for the persona's fleet. The view is greyed out with a reason before the First Jump (M-QA2, 2026-11-20).
- **Subject:** the persona's fleet (its flagship's fleet) is the default subject since M12a; a double-click or the fleet list still flies any fleet of the player's (`07 D128`, `D137`).
- **Arrival:** the view stays open with the fleet at the system's edge and offers *Open system*.
- **Battle:** the battle screen opens over the view, and the view returns when it closes.
- **Destroyed:** the view closes with a notice.
- **Pauses:** a new incident popup (a simulated fleet's incident, or a signal flown into) pauses the RTS clock while *Pause on incident* is on, which is the default. Encounters and chases open a battle as other battles do (see [combat.md](Design-Combat)).

### Turn-based behaviour

In TB a turn resolves its days at once, and the view shows the fleet where the turn left it and its route ahead (`07 D127`). Clicks and nudges are queued commands that apply on the next turn's first tick. Incidents and signals show in the turn report and take their choice on the next turn. A fleet piloted in TB stays piloted until the view closes, across turns. Identical orders give identical results under both clocks, because nothing in the view touches the simulation. See [time-and-modes.md](Design-Time-and-modes).

### Persona hooks

Three modifier keys, registered by M12a with their sources and read only aboard the persona's fleet (`07 D128`, `D134`, `D137`):

| Key | Persona skill | Effect |
|---|---|---|
| `hyper.incident` | Navigation | the incident odds, −10 / 20 / 30% by rank |
| `hyper.sensor` | Survey | the cells of sensor radius for site finding, group sighting and the view's ring (`deep::sight_radius`) |
| `hyper.burn` | Logistics | fuel per day |

The `persona.aboard` stat lets incident conditions test for the persona, so companion and story incidents need no engine change. See [factions-and-characters.md](Design-Factions-and-characters).

### Budget

- Nothing runs while the view is closed: no snapshot is built and no frame is drawn.
- While it is open, its mesh of a 41 × 41-cell window, its routes and up to 64 sensed fleets is built in ≤ 0.3 ms native, rebuilt only when the window, the month, a route or the visible fleets change; a frame draws only the sprites' interpolation; no repaint while paused and idle. As measured at F1: 8.6 µs against 351 µs.
- The incident roll adds one hash per route cell-day at planning (the route's event scan with its incident rolls measured 13.1 µs); the route-plan row across Huge stayed inside its line (223.1 µs against 234 µs).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| A flight view | Top-down in strategic mode, plus a secondary Starsector-style flight view for interstellar travel; travel can also be simulated, with a percentage chance of an encounter or event depending on stellar conditions | 2026-11-13 | `07` D3 (the owner's answers of 2026-11-13) |
| What the view is | A close, fleet-centred mode of the galaxy map, about 41 × 41 cells (an overlay and a free-flight scene rejected) | 2026-11-14 | `07 D120` |
| Controls | Click to move, right-click halts, WASD nudges 4 cells, `R` stance, hover plots via `Query::Plot` | 2026-11-14 | `07 D121` |
| Time scale | The strategic clock, with interpolation | 2026-11-14 | `07 D122` |
| No dice when piloted | "The incident chance doesn't occur on the flight view - that one is the most clearly reminiscent of Starsector." Simulated travel keeps per-day incidents; piloted travel meets storms, placed groups and signals. This replaced the first design, in which every route day rolled incidents for every fleet | 2026-11-14 (the owner) | `07 D123`, `D125`, `DESIGN.md` change log |
| Hostile groups when piloted | Placed per `(seed, cell, month)` by depth (option A), rather than the encounter roll spawning a chaser at sensor range | 2026-11-14 | `07 D123` |
| Incident effects and content | The `fleet` effect, seven condition stats, eight MVP incidents shared by both modes; no incident costs fuel | 2026-11-14 | `07 D124` |
| Opening and pauses | A *Flight view* button; pause on incident on by default | 2026-11-14 | `07 D125` |
| Entering and leaving | Fleet panel, double-click, Tab, Esc, arrival offers *Open system* | 2026-11-14 | `07 D126` |
| TB | Orders queue for the next turn; incidents in the turn report | 2026-11-14 | `07 D127` |
| Persona hooks | Persona fleet as default subject; `hyper.incident`, `hyper.sensor`, `hyper.burn`; `persona.aboard` | 2026-11-14, wired 2026-11-15 | `07 D128`, `D137` |
| Budget and size | Nothing while closed; ≤ 0.3 ms mesh; about 1.6k LOC after the owner's call (built at +1,302) | 2026-11-14 | `07 D129`, `05 F1` |

## Not yet built or still open

- The owner has set the incident rule; the other flight-view calls (`07 D120`–`D129`) were built on the recommended options and are listed in `DESIGN.md` for the owner's review.
- The spec session's reading of the owner's call for hostile groups (placed groups, option A) is marked "for the owner to confirm" in `07 D123`.
- Cut in F1 (the cut-if-late list): the feed's *Flight view* button; the *Flight view on departure* setting's Auto and Never options (only the button exists); groups patrolling around their home cell within the month. The pause toggle is in the HUD, not in the saved settings.
- The presentation pass (`07 D231`, after the MVP) puts 3D models on the map; `docs/18` lists the flight view among the places ships would be drawn in 3D. Not started. See [presentation-and-ui.md](Design-Presentation-and-ui).
- Sensor and storm-forecast techs feeding `hyper.sensor` and `hyper.incident` are listed as hooks in the tech tree rework (`07 D278`).

## Discrepancies noted

- **HUD contents.** `docs/16` (D120) lists "the fleet's fuel (days left at today's burn), readiness and power"; `07 D120` lists "fuel, readiness, hull, supplies, the band and today's odds"; `05 F1` (as built) lists "fuel and days left, readiness, hull, the band, the storm, today's odds, the arrival". The game has no supplies commodity (`DESIGN.md` lists Starsector-style supplies as offered and not scoped). This page follows `05 F1`.
- **TB turn animation.** `docs/16` (D127) says "when a turn resolves, the view animates the turn's path over about a second (G4's turn replay)". `05 F1`'s cut-if-late list includes "the turn replay's animation in TB beyond the map's own", and its implementation notes say only "in TB the view shows where the turn left the fleet"; the logged deviations do not say whether the animation was cut.
- **What a piloted fleet stops rolling.** `05 F1` says "A piloted fleet's incident and encounter rolls lapse", while `docs/16` (D125) and `07 D125` speak only of "a piloted fleet's roll already scheduled for today", and `docs/16` §4 describes the chase as replacing "a piloted fleet's instant encounter". Whether M8c's intercepts with other empires' sensed hostile footprints still apply to a piloted fleet is not stated beyond "a group waiting on the route is a rival there, so M8c's intercept decides the meeting".

## Sources

- 16 The flight view (all sections; the calls in §3)
- 07 Decisions: the owner's answer to D3 (2026-11-13, top of the file), D120–D129, D134, D137, D231, D278
- 05 Milestones: F1, M12a
- 00 Status (F1 merge, 2026-11-15; M-QA2, 2026-11-20)
- 01 §2.1 Hyperspace
- 18 Presentation and UI
- DESIGN.md (pillar "Interstellar travel view"; autonomous calls of 2026-11-14; change log)

## Related pages

- [Overview](Overview)
- [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)
- [time-and-modes.md](Design-Time-and-modes)
- [ships-and-fleets.md](Design-Ships-and-fleets)
- [combat.md](Design-Combat)
- [events-and-crises.md](Design-Events-and-crises)
- [factions-and-characters.md](Design-Factions-and-characters)
- [presentation-and-ui.md](Design-Presentation-and-ui)
- [architecture-and-performance.md](Design-Architecture-and-performance)
- [starts-and-exploration.md](Design-Starts-and-exploration)


---

# Colonies and surfaces

**Status:** built, as of 2026-11-23. Surfaces, placed districts and colonies in M4a, the economy around them in M4b, expeditions and features in M6, starbase colonies in M8b; terraforming (TF1) and planetary exploration (PX1) are designed for 1.0.

A colony is a settled body with its own market, pops, jobs and a hex surface. The player develops it as a light citybuilder: districts are placed on surveyed hexes, neighbouring districts help or hurt each other, the capital's service radius decides which hexes work at full output, and zoned hexes are filled by private construction paid from the colony's wealth. Exploring the surface by expedition reveals features and raises what the colony can build.

## How it works

### Kinds of colony

| Kind | Where | Founded by | Habitability | Core building | What it may build |
|---|---|---|---|---|---|
| Surface colony | a habitable world | colony ship | by climate match (below) | Capital I | every district kind its hexes allow |
| Domed colony | any body whose base habitability for the species is below 40 (moons, barren and cold rocky worlds, ice bodies) | colony ship | fixed 30 (Domes II tech: 45) | Dome Core: housing 6, amenities 4, +1 building slot | mining, generator, gas, crystal and city districts from its hexes; no agriculture, so food comes from imports or a Hydroponics Bay |
| Orbital habitat | around any body in an owned system | construction ship (`BuildHabitat`) | 70 for every species | Habitat Core: housing 10, amenities 4, +1 slot | city (cap 2) and industrial (cap 1) districts only |
| Starbase colony | at a system's star | a colony in the system, or a colony ship | consumes like a habitat | `starbase_core` (gives nothing) | no districts, no buildings; see [Starbases](Design-Starbases) |

- An orbital habitat costs 300 alloys + 50 parts from the system's markets by the local rule of `01 §10.4`, takes 180 days, and is its own body row (kind `habitat`, parent the host body, same orbit), so a planet can carry a surface colony and a habitat at once. It is size 4, with 7 "deck" hexes that give no terrain caps. One habitat per host body (`07 D25`).
- A habitat opens with 1 worker pop moved from an own colony in the system. As built, the donor is the lowest-id own colony in the system with a pop to spare (`07 D25`); see Discrepancies noted.
- Stations exploit deposits without pops; they belong to [Starts and exploration](Design-Starts-and-exploration).

### Founding by colony ship

- A colony ship is built only in a system with an own starbase, from that system's stock by the local rule (`07 D39`): 120 alloys plus a cargo of 50 food, 20 goods, 100 ore and 50 alloys, 120 days. It is consumed on arrival at a body in an owned or unclaimed system.
- On a habitable body it founds a surface colony with Capital I; on any other body, a domed colony with a Dome Core. Either way the colony starts with 1 worker pop of the ship's species, the centre ring surveyed, the cargo as its market stock and all policies `auto`.
- In an unclaimed system the colony ship also founds the size-1 starbase at 75 influence (`07 D40`). New colonies fill by migration from the empire's own colonies: "colonies are settled, not bred" (`01 §6.2`).
- **The founding gate** (`07 D273`, built 2026-11-21, every empire). Ordering a colony ship needs the empire's pops ≥ R(n), where n counts its planetary colonies (starbases aside) plus colony ships flying or queued. R(n) = `colonyStep` × m(m + 1)/2 × `empire.colonyPops`, with m = max(0, n + 1 − `colonyFree`). Content ships `colonyFree` 10 and `colonyStep` 16 (the draft's 8 was raised after host runs). The gate is checked when the ship is ordered, so a ship in flight is never stranded, and it moves no pops. The order greys out with "needs N pops (has M)". Mitigations (techs, the ringworld, the matrioshka brain, a ruler trait, persona skills) are listed on [Pops and society](Design-Pops-and-society#civil-upkeep-per-pop).
- **Food-supply resilience** (`07 D274`, built 2026-11-21). For any site, R = 1 if an own market in its system makes a daily food surplus, else min(1, accessibility × `importCapMul`) while the best route lands at least `landFloor` (0.5), else 0. The colony-ship target and the Colonize button show "Food supply: imports land N% of demand" with its terms; it informs and refuses nothing for the player. The AI scores sites × (1 − 0.5 × (1 − R)) and vetoes a site with R < 1 and no farmland.
- **The food buffer** (`07 D274`, every empire, on by default for the player): on a colony's monthly governor timer, a market that makes less food than it uses, whose imports cannot land its demand (access × `importCapMul` under `governor.bufferMargin` 1.1) or which is starving (30-day food satisfaction under 0.5), and whose food stock plus inbound is under 60 days of demand, buys food on the spot market up to 90 days. It also tops up the system's starbase market. It is switched off per colony by the food row's Forbid import.

### Habitability

Base habitability by climate match: same group 80, adjacent (wet↔cold, dry↔cold) 60, opposite (wet↔dry) 20; Gaia 100, Tomb 30; homeworlds 100 for their founding species. Domes 30 (Domes II: 45), habitats 70. Modifiers come from traits, tech (+5 / +10 / +15) and terraforming. Effects:

| Effect | Rule |
|---|---|
| Fertility | × (0.5 + hab/200) |
| Happiness | −(80 − hab)/4 when hab < 80 |
| Food and goods consumption | +1% per point under 70 (a dome at 30 consumes +40%) |

### Surface hexes

Planet size `s` (5–25) gives rings `r = clamp(ceil(s/5), 1, 5)` and `H = 1 + 3r(r+1)` hexes:

| Size | 5 | 6–10 | 11–15 | 16–20 | 21–25 |
|---|---|---|---|---|---|
| Rings | 1 | 2 | 3 | 4 | 5 |
| Hexes | 7 | 19 | 37 | 61 | 91 |

Each hex carries a terrain (plains, hills, mountains, forest, wetland, ocean, desert, tundra, ice, volcanic, crater), a feature (none ≈ 60%, ore_deposit, gas_vent, crystal_field, fertile_basin, geothermal, ruins, alien_life, crash_site, monolith, hazard) and a state (hidden → surveyed). Terrain follows the planet class. Contents are generated lazily from the body seed on first survey and stored, never regenerated. A new colony starts with the centre ring (7 hexes) surveyed; homeworlds with 19.

### Districts and caps

A district's cap is the number of eligible free surveyed hexes plus the placed ones (`07 D25`: a fertile basin on plains is one Agriculture hex, not three).

| District | Eligible hexes | Jobs | Housing |
|---|---|---|---|
| City | any hex not ocean, ice or an uncleared hazard; 1 per 3 surveyed hexes | 2 Clerks | 5 |
| Agriculture | plains, wetland, forest, fertile_basin | 2 Farmers | 2 |
| Mining | hills, mountains, crater, ore_deposit | 2 Miners | 2 |
| Generator | desert, volcanic, tundra, geothermal | 2 Technicians | 2 |
| Industrial | as City; 1 per 4 surveyed hexes | 2 Metallurgists or 2 Artisans (by designation) | 2 |
| Gas extraction | gas_vent | 1 Gas Extractor | 1 |
| Crystal mining | crystal_field | 1 Crystal Miner | 1 |

Ocean and ice hexes give no cap by themselves. A hazard hex holds no district or zone until cleared (150 credits, 90 days).

**Costs.** Build 60 days; upkeep 1 credit a month. The bill is drawn from the local market daily over the build time, and each day's progress scales with that day's draw satisfaction, so a dry market stalls the build.

| District | Cost |
|---|---|
| City | 150 ore + 20 alloys |
| Agriculture, Generator | 100 ore + 10 alloys |
| Mining | 10 alloys + 100 credits (`07 D102`, replacing 100 ore + 10 alloys) |
| Industrial | 150 ore + 40 alloys |
| Gas extraction, Crystal mining | 120 ore + 30 alloys |

Bills may carry credits, paid with the order (`07 D102`). A construction starts drawing only once its market holds 30 days of its daily draw of every bill commodity; until then it queues and posts demand so imports come, with the waiting reason shown (`07 D100`).

### Placement, adjacency and service (`07 D29`)

Districts are placed, not counted; buildings stay in slots. A district occupies one surveyed, eligible hex. The centre hex holds the capital or the Core. Adjacency between developed hexes is folded into the district's yield modifier whenever a neighbour changes:

| District | Adjacency bonus |
|---|---|
| Agriculture | +10% per adjacent Agriculture (max +20%); −10% per adjacent Industrial or Mining (pollution) |
| Mining | +10% per adjacent Mining (max +20%) |
| Generator | +10% per adjacent Industrial or Generator (grid; max +20%) |
| Industrial | +10% per adjacent City (labour; max +20%) |
| City | +1 housing per adjacent City (max +3); −1 amenity per adjacent Industrial, Mining, Gas or Crystal; +1 amenity per adjacent Park (+2 when the park is forest, wetland or ocean) |
| Gas, Crystal | none |

Jobs of a district type scale by the mean yield modifier of that type's districts. The tech Urban Planning (engineering tier 1) multiplies every bonus by +50% (`adjacency.scale`).

**Service.** The capital serves every hex within radius 2 (Capital II 3, Capital III 4); the tech Mass Transit (engineering tier 2) adds +1. A served City extends service to its neighbours (a flood fill on change). An unserved district yields −25% and takes no housing adjacency. Domes and habitats are always served.

**Zoning and private construction.** Each hex has a zone (none, a district kind, or Park), free to set. A Park stays undeveloped and gives the City bonus. A zoned district hex is built by the private sector when the colony's wealth is ≥ 1.5 × the bill at local prices, one at a time per colony, checked on the governor timer, paying each day's draw (and any credits in the bill) from wealth. The state builds anywhere eligible at any time.

**Starting layout** (the home template, `07 D25`): capital at ring 0; ring 1 forced to hold 2 fertile basins, 2 hills, 1 geothermal and 1 gas vent, with Agriculture on both basins, the Generator on the geothermal and one Mining on a hill; the vent and the second hill stay free. Ring 2 holds the two Cities, a third Agriculture on a forced forest hex, the Industrial and the second Mining on a forced hill. Every starting adjacency is 0 and every starting district is served; `datacheck` asserts both.

Not in MVP: transit hexes, buildings on hexes, pollution spread, land value (`01 §5.8`).

### Buildings

Building slots = 1 + floor(pops / 8) + capital slots, max 12; the capital occupies none. Build 90 days (capital upgrades 180), drawn from the local market. Upkeep is credits only; jobs consume commodities, buildings do not.

| Building | Cost | Upkeep /mo | Jobs | Effect |
|---|---|---|---|---|
| Capital I / II / III | — / 200 alloys / 400 alloys + 50 components | 2 / 4 / 6 cr | 2 / 3 / 4 Administrators and 2 / 3 / 4 Bureaucrats | housing 20 / 30 / 40; amenities 15 / 25 / 35; slots +2 / +3 / +4; research +3 / +6 / +9 per field; II needs 20 pops, III 40 |
| Research Lab | 60 alloys | 2 cr | 2 Researchers | — |
| Foundry | 40 alloys, 120 ore | 2 cr | 2 Metallurgists | — |
| Metal Refinery | 40 alloys, 120 ore | 2 cr | 2 Metal Refiners | rare metals from rare ore |
| Smelter | 30 alloys, 120 ore | 2 cr | 2 Smelters | rare metals from ore where no rare ore is in reach (`07 D162`) |
| Factory | 80 alloys | 2 cr | 2 Artisans | — |
| Workshop | 100 alloys, 10 parts | 3 cr | 2 Fabricators | — |
| Luxury Works | 100 alloys, 10 parts | 3 cr | 2 Luxury Artisans | — |
| Refinery | 40 alloys, 120 ore | 2 cr | 2 Refiners | fuel |
| Administrative Office | 60 alloys | 2 cr | 2 Bureaucrats | + admin cap |
| Holo-Theatre | 60 alloys | 2 cr | 2 Entertainers | + amenities |
| Spaceport | 120 alloys, 20 parts | 4 cr | 2 Merchants | +75 freight capacity |
| Clinic | 60 alloys | 2 cr | 1 Medic | −5% mortality |
| School | 60 alloys | 2 cr | 1 Teacher | the Teacher carries the skill modifier |
| Fortress | 100 alloys | 3 cr | 2 Soldiers | +10 defence, +2 stability |
| Depot | 60 alloys | 1 cr | — | +500 stock capacity per commodity |
| Dome Core | free at founding | 1 cr | — | housing 6; amenities 4; slots +1 |
| Habitat Core | free at founding | 1 cr | — | housing 10; amenities 4; slots +1 |
| Hydroponics Bay | 60 alloys | 2 cr | 2 Farmers at 4 food each | the only food source on domes and habitats |

Buildings carry an `item` column for 1.0 loot (`07 D30`). 1.0 adds tiered upgrades, species-specific buildings and Apex-era megaplex buildings.

### Designation

Designation is set automatically by the dominant district type (Mining, Agri, Forge, Factory, Trade or Research World; ties in the district table's order) once that type counts `designationMinDistricts` (2) districts; before that a colony has the neutral designation with no bonus (`07 D33`). The homeworld is Capital (no job bonus; industrial districts make goods). A designation gives +15% output to the matching jobs and sets the industrial district's job (Forge → Metallurgists, otherwise Artisans). The player may override it.

### Expeditions and surface features

- An expedition costs 100 credits + 10 consumer goods and runs until the planet is fully surveyed, revealing `3 + 0.5 × leaderSkill` hexes per 30 days (1.5 without a leader), one hex at a time on a timer that carries the fraction (`07 D35`: 20 days exactly at 1.5).
- **Auto-expedition** is on at founding for every empire's colonies; it starts when the cost fits above three months of the empire's expenses, on the governor's 30-day timer (`07 D35`). The player can point an expedition at a hidden hex to chase a visible feature.
- **Feature events** (six in MVP, for every empire): an ore deposit gives +40 ore without a popup; ruins start a 2-chapter mini-archaeology; alien life offers tame (the hex becomes a fertile basin), cull or study; a crash site gives salvage research or unity; a monolith gives unity and a chain; a hazard reveals its clearing cost. A habitat is fully surveyed at founding.
- Surface exploration beyond this (sites, ruins and encounters on hexes) is out of MVP with hooks (`07 D92`): hidden hexes are fought over by [ground combat](Design-Ground-combat) as their terrain.

### Colony governor

The governor (default on, per-planet manual toggle) builds districts and buildings toward the designation and keeps one slot free. It answers a shortage projected within 360 days in the priority food → goods → gas → fuel → alloys → crystal and parts → rare metals → luxuries, builds housing when crowding exceeds 1.1, builds a Spaceport at the busiest market when empire freight utilisation exceeds 80%, zones before it builds (on the best eligible hex by adjacency) and leaves zoned hexes it has not reached to the private sector. A fix that could not be ordered is not retried at that colony for 3 governor periods, or until a construction in its system finishes or the colony gains a slot or a surveyed hex (`07 D242`). Its own district or building stuck with no progress for over 360 days is cancelled with no refund (`07 D264`); the player's own orders are never cancelled. Governor decisions run on a colony's state change or its staggered 30-day timer, never every tick.

### RTS and TB

Placement, zoning, expeditions, hazard clearing and colony-ship orders are Commands applied at tick boundaries, so both modes behave alike; adjacency and service are computed at placement, and the daily tick reads the folded per-colony multipliers.

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Opening as a fledgling civilisation | Empires start pre-FTL with a homeworld and three small off-world colonies; domes and habitats moved into the MVP | 2026-11-01 | `07 D20` |
| Surface exploration in MVP | Lite version (hex generation, survey, caps, auto-expedition, 6 feature events) in MVP; full feature set and terraforming in 1.0 | 2026-11-01 | `07 D7` |
| Citybuilder colonies | Placement, adjacency, service, zoning and parks (option A); transit hexes declined | 2026-11-02 | `07 D29` |
| M4a calls | Starting layout corrected (one Mining on ring 2); cap read as eligible free hexes; habitat donor is the lowest-id colony | 2026-11-02 | `07 D25` |
| Neutral designation | No designation bonus until the dominant kind counts two districts | 2026-11-03 | `07 D33` |
| Expedition details | Fractional timer, auto-expedition above three months' expenses, hazard clearing as a construction row | 2026-11-05 | `07 D35` |
| Starbases as colonies | A colony ship needs an own starbase in its yard system; colony ships claim unclaimed systems with a size-1 starbase | 2026-11-08 | `07 D39`, `D40` |
| Start rule and mine bills | Constructions wait for 30 days of stock before drawing; mines billed in 10 alloys + 100 credits | 2026-11-14 | `07 D100`, `D102` |
| Ground combat on surfaces | Surface exploration out of MVP with hooks | 2026-11-14 | `07 D92` |
| Post-MVP surfaces | Terraforming by four measures plus late gas giants, physical inputs, real but not guaranteed side effects; planetary exploration by teams or the persona; relics as items or planet bonuses; rated resource nodes | 2026-11-16 | `07 D230` |
| Governor back-off and stuck builds | Failed fixes back off three periods; stuck governor builds cancelled after 360 days | 2026-11-18, 2026-11-19 | `07 D242`, `D264` |
| Founding gate | Pops needed for the next colony rise past ten planetary colonies; ships at `colonyStep` 16 | 2026-11-21 | `07 D273` |
| Supply resilience | Food-supply resilience weighs site choice; AI veto; food buffer as the backstop | 2026-11-21 | `07 D274` |
| Terraforming placement | Left to TF1, not built as `01 §5.7`'s stopgap in M19 | 2026-11-23 | `07 D283` |

## Not yet built or still open

- **Terraforming (TF1, 1.0, ≈ 1.8k LOC).** The design that will replace `01 §5.7`'s single class swap (2,000 credits + 200 gas, 1,800 days) is `docs/17 §3.6`: four parameters (temperature, atmosphere, hydrosphere, biosphere) from which the climate class is derived; projects that move one parameter at a time over years with physical inputs (machine tools, gas, hauled ice, exotic matter late) and energy; side effects such as a runaway greenhouse or melt-flooding of low hexes, with a chance rather than a certainty; habitability per species as a function of the parameters; gas giants terraformable late. Hexes will re-roll their terrain for the derived class while surveyed state and features persist. Milestone placement is the spec session's.
- **Planetary exploration (PX1, 1.0, ≈ 2.0k LOC).** Expedition teams (a leader, a companion or the persona in person) will crawl hidden hexes on timers, finding rated resource nodes that boost districts, relics usable as items or planet bonuses, treasure caches, and dangers that hook into ground combat (`docs/17 §3.7`).
- **The private sector's scope.** Private wealth is spent only on zoned construction in MVP.
- **Per-colony tax override** and tiered or species-specific buildings are 1.0.
- The buildings' `item` column is a 1.0 hook for loot (Nanoforge: Foundry +50%).

## Discrepancies noted

- **Mining district cost.** `01 §5.3` lists "Agriculture, Mining, Generator | 100 ore + 10 alloys"; `07 D102` (2026-11-14) says "The mining district costs 10 alloys and 100 credits ... (was 100 ore and 10 alloys: 140)". The later decision is taken as current; the `01` table has not been updated.
- **Habitat donor.** `01 §4.1` and `§10.1` say the habitat starts with "1 worker pop moved from the nearest own colony in the system"; `07 D25` (built) says "the donor of its first pop is the lowest-id own colony in the system with a pop to spare (the homeworld)", having declined "the donor by orbit distance". The built rule is taken as current.
- **Habitat hexes.** `01 §5.8` says "Domes (7 hexes) and habitats (no hexes) are always served"; `01 §4.1` gives a habitat "7 'deck' hexes that give no terrain caps", and `07 D35` says "a habitat is 'fully surveyed' at founding (7 of 7)".
- **Terraforming text.** `01 §5.7` still describes the single class-swap project; `07 D230` and `D283` (2026-11-23) set `docs/17 §3.6`'s model as the design, left to TF1.

## Sources

- 01 §4.1 Dawn: exploiting the home system
- 01 §5 Planets and planetary exploration, especially §5.3, §5.5 and §5.8
- 01 §10.1 Hulls (colonisation)
- 01 §17 Automation defaults
- 07 decisions: `D7`, `D20`, `D25`, `D29`, `D33`, `D35`, `D39`, `D40`, `D92`, `D100`, `D102`, `D162`, `D230`, `D242`, `D264`, `D273`, `D274`, `D283`
- 09 §4 Colonies as mild citybuilders
- 17 §3.6 Terraforming, §3.7 Planetary exploration
- 15 D92
- 05 milestones (M4a, M4b, M6, M8b), 00 status, DESIGN tracker

## Related pages

- [Overview](Overview)
- [Pops and society](Design-Pops-and-society)
- [Starbases](Design-Starbases)
- [Starts and exploration](Design-Starts-and-exploration)
- [Economy and resources](Design-Economy-and-resources)
- [Trade and logistics](Design-Trade-and-logistics)
- [Ground combat](Design-Ground-combat)
- [Megastructures](Design-Megastructures)
- [Civilisations and species](Design-Civilisations-and-species)
- [Progression](Design-Progression)


---

# Pops and society

**Status:** built, as of 2026-11-23. Cohorts, jobs, happiness, migration, skill and taxation in M4b (`07 D24`, `D26`–`D28`); the per-colony stagger in HP1; halt growth in M-DEFICIT; civil upkeep with `07 D273`; party politics on cohorts in FC1. Ethics drift and Stellaris's eight ethics are designed for 1.0 (M15's remainder, CV1).

Population is held as fractional cohorts per colony, species and stratum rather than as individual pops. Cohorts are born and die as two separate flows, fill jobs by stratum, consume a basket that responds to prices, grow happier or unhappier toward a target, learn skill by working, migrate by gravity across hyperspace, and earn income that the state taxes and that builds the colony's private wealth. These are the levers through which the player's policies reach output, growth and stability.

## How it works

### Cohorts and strata

A cohort is `{ planet, species, stratum, count: float, happiness: 0–100, skill: 0–1, ethics: float[8] }`. Strata are **worker**, **specialist** and **ruler** (1.0 adds slave and robot). A planet typically has 3–9 cohorts. A cohort's ethics vector is set at creation as `0.7 × empireVec + 0.3 × speciesVec` and does not change in MVP. Cohorts that exist at game start have skill 0.5. When pops join a cohort (births, promotion, migration) its happiness, skill and ethics become the count-weighted means. No per-pop entity exists anywhere (`07 D14`: fractional cohorts).

If the cohorts table is full, new pops join the colony's cohort of the stratum, of any species, else any of its cohorts (`07 D175`).

### Update cadence

The economy runs every tick at 1/30 of the per-30-day rates. Since HP1 (merged 2026-11-17, `07 D175`), each colony's births, deaths, skill and migration run on its own day, `(tick + colony) mod 5`, at 5 days' rates (`pops.growthEvery` 5), and happiness and stability step on the same day with 5 days of the 0.1 smoothing at once (1 − 0.9⁵). Jobs, promotion and demotion stay daily.

### Births, deaths and crowding

```
crowding   = pops / housing
fertility  = 0.009 × (0.5 + hab/200) × foodSat × (0.75 + happiness/200) × clamp(1.5 − crowding, 0, 1) × speciesGrowthMod   // per pop per 30 days
mortality  = 0.0045 × max(0.5, 1 − 0.05 × (clinics + medics)) × (1 + max(0, crowding − 1)) × (starving ? 5 : 1)
starving   = foodSat < 0.5 for ≥ 90 consecutive days
```

- Births land in the worker cohort of the same species on the planet; deaths come from every cohort.
- Two flows give population momentum: a famine keeps killing until `foodSat` recovers, and a boom overshoots housing before crowding bites. Starvation is ×5 mortality, ≈ 2.25% of pops per 30 days.
- **Housing is a soft ceiling.** With full satisfaction, happiness 50 and no clinics, births equal deaths at pops = housing; a happy colony settles a little above it (happiness 76 → ≈ 1.04 × housing) and pays the crowding penalty; an unhappy or hungry one settles below.
- Low habitability halves fertility at worst, so domes grow slowly on their own and fill by immigration: colonies are settled, not bred.
- `pop.growth` modifiers (Rapid Breeders) multiply fertility; `pop.mortality` modifiers (Clinic, Medic: −5% each) multiply mortality.

**Halt growth** (`07 D264`, built 2026-11-19). `Command::HaltGrowth` stops births at an own colony and makes migrant packets skip it until lifted; deaths, leavers and resettlement go on. The player toggles it on the colony panel, with the reason in the tooltip. The AI halts a colony whose food runs a projected 30-day deficit, whose food stock covers fewer than `budget.haltCoverDays` (90) days, and where no food producer can be placed and staffed that plan. It lifts on a positive daily food balance, or when a food producer completes there. A colony too small to staff the cheapest food producer is not halted (DESIGN, 2026-11-20).

### Jobs and job filling

Jobs come from districts, buildings and the capital; each has a stratum. Filling runs for a colony whenever its jobs or pops change. Order: ruler jobs → specialist → worker; within a stratum by job priority (player-adjustable −1 / 0 / +1, default from designation), then in `jobs.json` order. A job is listed after every job of its stratum that makes one of its inputs, so producers fill ahead of their consumers (`07 D266`; `datacheck` enforces it).

| Mechanic | Rule (per 30 days, applied daily) |
|---|---|
| Promotion | `min(freeWorkers, 0.1 × openSpecialistJobs) × (0.5 + skill)` pops move from the worker cohort to the specialist cohort (same species); ruler promotion likewise from specialists |
| Demotion | specialists unemployed for 180 days move back |
| Unemployed | still consume |

Job outputs scale with designation (+15%), stability, skill, adjacency and service, vitality, species traits, techs, buildings and edicts. The job table itself is on [Economy and resources](Design-Economy-and-resources).

### Consumption basket

| Stratum | Food | Consumer goods | Luxuries |
|---|---|---|---|
| Worker | 1.0 | 0.25 | 0 |
| Specialist | 1.0 | 0.5 | 0 (0.25 under Comfortable) |
| Ruler | 1.0 | 1.0 | 0.5 |

Per pop per 30 days. The **living standard** policy scales goods and luxuries (Austere 0.6× and no luxuries, Standard 1×, Comfortable 1.4× with specialists gaining 0.25 luxuries) and trades consumption for happiness (−10 / 0 / +10). Food never scales.

**Price elasticity** (`07 D24`). Demand is `basket × livingStandard × priceFactor_c`, with `priceFactor_c = clamp(1 − elasticity_c × (price_c / basePrice_c − 1), 0.5, 1.25)`. Elasticity: food 0, goods 0.5, luxuries 1.0, everything else 0. Pops buy less of what is expensive before it runs short; the scaling costs happiness as `priceStrain`.

### Happiness and stability

```
ethicsMatch     = 5 × Σ_{p ∈ empire poles} points_p × (cohort[p] − cohort[opposite(p)])     // −15..+15; +10.5 at game start
happinessTarget = 50
          + livingStandardMod
          + ethicsMatch
          + amenitiesMod           // clamp((amenities − 0.5 × pops) × 2, −20, +10)
          − shortagePenalty        // food 25×(1−foodSat); goods 20×(1−goodsSat); luxuries 10×(1−luxSat) for strata with luxuries
          − habitabilityPenalty    // (80 − hab)/4 when hab < 80
          − unemploymentPenalty    // 10 if the cohort has ≥ 0.5 unemployed pops
          − crowdingPenalty        // 15 × max(0, crowding − 1)
          − priceStrain            // Σ over goods and luxuries of 5 × max(0, 1 − priceFactor_c); at most 5
          − taxStrain              // 20 × max(0, taxRate − fairTax)
          + prosperity             // min(5, wealth / (20 × pops))
          + eventMods + party approval
happiness += (happinessTarget − happiness) × 0.1 per day
stabilityTarget = weighted mean happiness (ruler ×2, specialist ×1.5, worker ×1) + fortressBonus + modifiers − 20 × piracyOfSystem
stability += (stabilityTarget − stability) × 0.1 per day
```

`foodSat`, `goodsSat` and `luxSat` are 30-day rolling means of daily satisfaction.

| Stability | Effect |
|---|---|
| < 10 | revolt risk |
| < 25 | −25% output; the unrest situation starts |
| > 60 | +5% output |
| > 80 | +10% output |

As built in M11, an unrest revolt costs its colony happiness and stability, not empire influence (`07 D86`). FC1's party approval moves members' happiness by ± (approval − 50) / 5 (`01 §9.4`).

### Migration (`07 D24`, `D26`)

```
border(s, d) = 1     same empire, or a migration treaty (1.0)
             = 0.5   d's immigration is Open
             = 0.25  d's immigration is Selective and the cohort's skill ≥ 0.6 or its species already lives on d; else 0
             = 0     d is Closed, s's emigration is Closed, or s and d are at war
standardPull = clamp(0.5 + 0.5 × livingStandard_d / livingStandard_s, 0.5, 1.5)
attract(d, stratum) = freeJobs_stratum(d) × (0.5 + hab_d/200) × (1 + happiness_d/100) × clamp(1.5 − crowding_d, 0, 1)
                      × border(s, d) × standardPull / (1 + cells(s, d) / 2)²
leavers per 30 days = (0.002 × count + 0.02 × (unemployed + (happiness < 30 ? count : 0))) × (displaced ? 5 : 1)
```

- Every colony recomputes, on its 30-day governor timer, the three most attractive destinations per stratum among the colonies of any empire within 16 cells across open hyperspace, and caches them.
- A cohort whose best cached destination beats its own colony's `attract` by 1.2× sends its leavers in one packet to the best of them, and none while the packet would be under 0.01 pops (`minPacket`, `07 D242`).
- **Displaced** cohorts (colony starving, under bombardment, conquered within 5 years, or stability < 20) leave at ×5, move at a threshold of 1.0, and read the destination's refugee policy (Accept 1, Refuse 0) instead of its immigration policy.
- Policies: immigration Open / Selective / Closed (default Selective), emigration Free / Closed, refugees Accept / Refuse, each with a 1-year lock. Sedentary halves `leavers`.
- `Resettle { from, to, cohort, count }` moves pops between own colonies for 2 influence per pop, same day, the movers entering at −10 happiness. It is allowed into a halted colony.
- Foreign arrivals keep species, ethics and skill; a species new to the colony creates its cohort. Travel is abstract (same-day arrival). At game start no colony has a free job, so nothing moves until the governor builds one.

### Skill (`07 D27`)

```
skill += (1 − skill) × 0.01 × employedShare × pop.skill_gain     // per 30 days; half the gap closes in ≈ 6 years unmodified
skill −= skill × 0.02 × unemployedShare × pop.skill_decay          // per 30 days
job output × (1 + (0.25 + Σ pop.skill_output) × (skill − 0.5))     // 0.875 at 0, 1.0 at 0.5, 1.125 at 1, unmodified
```

| Source | Effect |
|---|---|
| Public Education (society tier 1) | `pop.skill_gain` +25% |
| Vocational Training (tier 2) | `pop.skill_output` +0.10 |
| Lifelong Learning (tier 3) | `pop.skill_decay` −50% |
| Neural Interfaces (tier 3) | `pop.skill_gain` +50% |
| Education policy | Minimal ×0.75 (free) / Standard (free) / Generous ×1.5 at 0.2 credits per pop per 30 days; 1-year lock |
| School's Teacher job | planet `pop.skill_gain` +25%, +1 amenity |
| Meritocracy civic | `pop.skill_gain` +15%, `pop.skill_output` +0.05 |
| Governor trait Educator | planet `pop.skill_gain` +25% |
| Prosperity perk Guilds | a skill modifier (`01 §6.8`) |
| 1.0: Science Nexus; Synthetic ascension | empire `pop.skill_gain` +25%; synthetic cohorts start at 0.75 |

Whatever the modifiers, the factor is 1.0 at skill 0.5, so the opening is unchanged.

### Taxation, private wealth and vitality (`07 D28`)

```
G       = 4 × tradeValue produced on the colony        // clerks 8, merchants 16 per pop per 30 days
I       = 1.2 worker, 2.4 specialist, 8.0 ruler         // gross income per pop per 30 days
taxRate ∈ { Low 0.10, Standard 0.25, High 0.40, Punitive 0.60 }      // empire policy, 1-year lock
ledger  += taxRate × (G + Σ I) × collection             // collection = 1 − 0.5 × maxPiracyOnSpoke
wealth  += 0.1 × (1 − taxRate) × (G + Σ I)              // per colony
wealth  −= 0.01 × wealth per 30 days                     // depreciation
fairTax  = 0.25 + Σ tax.tolerance
vitality target = clamp(1 + (fairTax − taxRate), 0.5, 1.2); vitality += (target − vitality) × 0.02 per 30 days   // half-life ≈ 3 years
every job output × vitality; I × vitality
```

- Standard reproduces the opening ledger exactly: the homeworld's `0.25 × (80 + 77.2) = 39.3` is trade value 20 plus pop tax 19.3.
- **Commerce policy** sets what the state's `G` share becomes: Credits (default), Mercantile (0.75 credits + 0.25 unity), Innovation (0.75 credits + 0.25 research). The `I` share is always credits.
- **Private wealth** is spent on private construction on zoned hexes ([Colonies and surfaces](Design-Colonies-and-surfaces#placement-adjacency-and-service-07-d29)), paying each day's draw and any credits in the bill. The **Levy** edict seizes 50% of every colony's wealth for 100 influence and −10 happiness empire-wide for a year. Conquest destroys half of a colony's wealth.
- **Tolerance** (`tax.tolerance`): Fiscal Administration (society tier 1, +0.05), Automated Audit (tier 3, +0.05), the Corporate Charter civic (+0.05, trade value +5%), the Prosperity perk Fair Levies (+0.05). With all four, Punitive settles at vitality 0.85 instead of 0.65.
- Worked cases: Punitive from day 1 gives 2.4× revenue at first (91 against 38 per 30 days), vitality 0.82 after 3 years and 0.68 after 10. Low from day 1 gives revenue 15, vitality 1.15 after 10 years and private construction at 2.3× the Standard pace.
- **Wealth classes** (FC1, DESIGN 2026-11-18): a colony's private wealth is held 5 / 25 / 70% by workers, specialists and rulers, at a 5% return a month against wages of 2 / 4 / 6 a pop; this sorts cohorts into owners or wage-earners for party politics.

### Civil upkeep per pop

`07 D273` (built 2026-11-21, every empire): each day the ledger charges pops × u(n) ÷ 30 credits, with u(n) = `popUpkeep` × max(0, n − `colonyFree`) × `empire.popUpkeep`; `popUpkeep` is 0.01 credits a pop a month, `colonyFree` 10, and n the planetary colony count. It rises with colonies, not pops, so tall is cheaper than wide. It is credits only, booked on the sprawl expense row and not surcharged by sprawl.

| n (planetary) | 12 | 15 | 20 | 22 | 30 | 36 |
|---|---|---|---|---|---|---|
| u(n), credits a pop a month | 0.02 | 0.05 | 0.10 | 0.12 | 0.20 | 0.26 |

The same decision gates founding on pops ([Colonies and surfaces](Design-Colonies-and-surfaces#founding-by-colony-ship)). Mitigations, multiplicative and clamped at 0.25:

| Source | Entry |
|---|---|
| `bureaucratic_protocols` (tech t1) | popUpkeep −0.10 |
| `planetary_bureaucracy` (tech t2) | colonyPops −0.10, popUpkeep −0.10 |
| `galactic_administration` (tech t3) | colonyPops −0.15, popUpkeep −0.15 |
| Ringworld section 3 | colonyPops −0.15 |
| Matrioshka Brain stage 3 | popUpkeep −0.15 |
| Ruler trait `centralizer` | colonyPops −0.10, popUpkeep −0.10 |
| Persona `planning` ranks 1/2/3 | colonyPops −0.05 / −0.10 / −0.15 |
| Persona `bureaucracy` ranks 1/2/3 | popUpkeep −0.05 / −0.10 / −0.15 |

The AI takes the same rules with no handicap; only the persona's skills are the player's (DESIGN, 2026-11-21).

### Ethics on cohorts

Cohorts carry an 8-vector, two poles per axis, each axis summing to 1, over four axes: Collectivist–Individualist, Militarist–Pacifist, Materialist–Spiritualist, Xenophile–Xenophobe. In MVP the vector is static and read only for `ethicsMatch`. FC1's monthly politics day drifts each cohort toward the material bloc of the jobs it works (Mining, Logistics, Researchers, Bureaucracy, Soldiers), its unemployed share back to unaffiliated; that is bloc membership, not ethics drift (`01 §9.4`; details on [Factions and characters](Design-Factions-and-characters)).

**Ethics drift (1.0, designed).** Each cohort's vector will move 1% per 30 days toward a target from lived conditions, plus +2% per 30 days toward the empire's ethics and faction propaganda:

| Condition | Pushes toward |
|---|---|
| Empire at war in the last 12 months | Militarist |
| 10+ years of peace | Pacifist |
| ≥ 25% of planet pops other species, happiness ≥ 50 | Xenophile |
| Foreign empire occupied or raided the planet | Xenophobe |
| Research output per pop above empire median | Materialist |
| Low stability or recent crisis | Spiritualist |
| Trade value per pop high, living standard Comfortable | Individualist |
| Austere living standard, rationing or shortages | Collectivist |

The owner chose Stellaris's eight ethics for 1.0 (`07 D230`), so Collectivist–Individualist will become Authoritarian–Egalitarian in CV1, and `docs/19 §3.2` proposes moving the economic rows of this table out of ethics and into the blocs.

### RTS and TB

Every rule above runs inside the daily tick (or its 5-day stagger) and every player lever (policies, Resettle, HaltGrowth, the Levy) is a Command applied at a tick boundary, so both modes behave alike; TB resolves the same days over a turn.

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Pop model | Fractional cohorts | 2026-11-01 (confirmed 2026-11-13) | `07 D14` |
| Beyond Stellaris | Births and deaths as two flows, housing as a soft ceiling, price-elastic baskets, skill, gravity migration; wages and age bins deferred to 1.0 | 2026-11-02 | `07 D24` |
| Cross-empire migration | Border factor from immigration, emigration and refugee policies; living-standard pull; displaced cohorts; Resettle | 2026-11-02 | `07 D26` |
| Skill modifiers | Three keys, four techs, Education policy, School and Teacher, Meritocracy, a perk, 1.0 hooks | 2026-11-02 | `07 D27` |
| Taxation | Tax on gross commerce and incomes, commerce policy, private wealth, vitality, `taxStrain`, Levy | 2026-11-02 | `07 D28` |
| M4b calls | Wealth accrues per colony; numbers corrected to the measured run (settling at ≈ 1.04 × housing) | 2026-11-03 | `07 D33` |
| Revolt cost | An unrest revolt costs local happiness and stability, not empire influence | 2026-11-14 | `07 D86` |
| Eight ethics | Stellaris's eight ethics, replacing the four-axis model, in 1.0 | 2026-11-16 | `07 D230` |
| HP1 stagger | Pop growth and happiness per colony every 5 days at 5 days' rates; jobs stay daily (replaces daily growth and happiness) | 2026-11-17 | `07 D175` |
| Migrant packets | One packet to the best destination, minimum 0.01 pops (replaces a daily three-way split) | 2026-11-18 | `07 D242` |
| Factions on cohorts | Parties from ethic poles plus material blocs and wealth classes (FC1) | 2026-11-18 | `07 D241`, `D214` |
| Halt growth | A command for player and AI; AI food-halt parameters | 2026-11-19 | `07 D264` |
| Job order | Producers fill ahead of their consumers | 2026-11-20 | `07 D266` |
| Empire size paced by pops | Per-pop civil upkeep rising past ten planetary colonies, with mitigations; same rules for the AI | 2026-11-21 | `07 D273` |

## Not yet built or still open

- **Ethics drift** (M15's remainder, 1.0) and **Stellaris's eight ethics** with the economic rows moved to blocs (CV1, 1.0).
- **Wages** (jobs paying credits per pop, floating with vacancies) and **age bins** per cohort, deferred to 1.0 (`07 D24`).
- Slave and robot strata, migration treaties, and migration routed through the inbound ring instead of same-day arrival (1.0).
- A per-colony tax override (1.0).
- 1.0 skill hooks: the Science Nexus and Synthetic ascension (the ascension paths left M19 for a later milestone, `07 D283`).

## Discrepancies noted

- **Update cadence.** `01 §6.2`, `§6.5`, `§6.6` and `§6.8` describe births, deaths, happiness, stability, migration and skill as daily ("daily, at 1/30 of these rates"; "happiness += ... each day"); `07 D175` and `00-status` (HP1, merged 2026-11-17) say "Pop growth and happiness run per colony every 5 days; jobs stay daily". The later decision is taken as current; `01 §6` has not been updated.
- **Migrant split.** Within `01 §6.6`, the `leavers` line's comment says "per cohort, split across the cached destinations by attract", while the prose below says a cohort "sends its leavers in one packet to the best of them" (`07 D242`). The prose and `07 D242` are taken as current.
- **Faction term.** `01 §6.5` lists "eventMods (+ factionMod, 1.0)" as a 1.0 term, while `01 §9.4` says "FC1 replaced this sketch" with members' happiness changing by ± (approval − 50) / 5, and `00-status` records FC1 merged into `main` on 2026-11-19.

## Sources

- 01 §6 Pops, strata, jobs: §6.2, §6.3, §6.4, §6.5, §6.6, §6.8
- 01 §7.9 Taxation, private wealth and vitality, §7.3 Jobs
- 01 §9.3 Ethics and drift, §9.4 Factions, §9.5 Government, civics, policies, edicts
- 07 decisions: `D14`, `D24`, `D26`, `D27`, `D28`, `D33`, `D86`, `D175`, `D214`, `D230`, `D241`, `D242`, `D264`, `D266`, `D273`
- 09 §1, §2, §3 (rationale for D26–D28)
- 17 §0 The owner's calls
- 19 §3 Macro layer
- 05 milestones (M4b, M15), 00 status, DESIGN tracker

## Related pages

- [Overview](Overview)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Economy and resources](Design-Economy-and-resources)
- [Trade and logistics](Design-Trade-and-logistics)
- [Factions and characters](Design-Factions-and-characters)
- [Civilisations and species](Design-Civilisations-and-species)
- [Progression](Design-Progression)
- [Events and crises](Design-Events-and-crises)
- [Starbases](Design-Starbases)


---

# Starbases

**Status:** built, as of 2026-11-23, with stations of sections designed. Starbases as colonies, sizes and physical logistics in M8b (`07 D39`, `D40`); capture in M10 (`07 D113`, fixed by `D149`); the Starsector-style station (`07 D284`) has no milestone placed yet.

Every claimed system holds one starbase, and every starbase is a small colony with its own market, housing and crew. The starbase is the claim: a system belongs to whoever owns its starbase. Its size, from 1 to 8, scales everything it does (hull, yard, docking, storage, sensors, supply, repair, freight, security, upkeep), and the larger sizes are unlocked by general construction techs. Nothing reaches a starbase except by shipment, and a base fed from its own system earns a self-sufficiency bonus.

## How it works

### A starbase is a small colony

- It is its own body row (kind `starbase`, class `starbase`, parent the star) at the starbase orbit, surveyed by every empire (`07 D40`).
- It has a one-ring deck surface of seven hexes that takes no district and no building; the core `starbase_core` gives nothing; the size row's housing and crew jobs are added in the surface fold. No governor and no expedition run on it (`07 D40`).
- **Crew jobs** are worker jobs (`crew`) that produce nothing. The yard, repair and upgrades run at `50% + 50% × filled / jobs`. Crew pops consume like habitat pops and arrive by migration.
- Starbase colonies count as colonies for claims, supply and migration, and for their system's sprawl through their pops and districts only (not the colony term), but not toward the era gates' colony counts or victory (`07 D39`, `D40`). The founding gate of `07 D273` counts planetary colonies, starbases aside.
- There are no modules and no buildings on an empire starbase (`07 D39`).

### The claim and founding

| Route | Cost | Result |
|---|---|---|
| `FoundStarbase` from a colony in a system with no starbase | 100 alloys + 75 influence, 90 days; the influence at the order, the alloys from the founding colony's own market (`07 D40`) | a size-1 starbase; 1 worker pop moves from that colony to the crew on completion; refused while another founding runs in the system |
| A colony ship aimed at the star of an unclaimed system | 75 influence on arrival (checked at the order and again on arrival) | a size-1 starbase colony; the ship is consumed, its pop becomes the crew and its cargo the starbase market's stock |
| A colony ship founding a planetary colony in an unclaimed system | 75 influence | the colony and its size-1 starbase together |

A system without a starbase is unclaimed. Outposts and the construction ship's claim are gone (`07 D39`). A pirate base is a starbase row without a colony and adds no freight or docking (`07 D40`).

### Sizes

| Size | Name | Unlocked by | Upgrade cost | Days | Hull | Batteries | Yard (largest hull, slots) | Docking | Storage | Sensor | Supply range | Repair /day | Freight | Housing, crew jobs | Security | Upkeep /30 d |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Outpost | founding | (founding) | 90 | 1,000 | 0 | civilian, 1 | 2 | 250 | 1 | 1 | 1.5% | 25 | 2, 1 | 0.15 | 1 cr, 1 alloy, 0.5 fuel |
| 2 | Station | Industrial Automation (tier 0, known at start) | 200 alloys, 20 parts | 120 | 3,000 | 0 | corvette, 1 | 4 | 500 | 2 | 2 | 2% | 100 | 4, 2 | 0.30 | 5 cr, 2 alloys, 1 fuel |
| 3 | Starport | Assembly Robotics (tier 1) | 350 alloys, 40 parts | 180 | 5,000 | 1 | destroyer, 2 | 6 | 750 | 2 | 3 | 2.5% | 150 | 6, 3 | 0.35 | 7 cr, 3 alloys, 1.5 fuel |
| 4 | Starhold | Orbital Habitation (tier 2) and Era II | 500 alloys, 60 parts | 240 | 8,000 | 2 | cruiser, 2 | 8 | 1,000 | 3 | 4 | 3% | 200 | 8, 4 | 0.45 | 9 cr, 4 alloys, 2 fuel, 0.5 parts |
| 5 | Bastion | Fortified Installations (tier 2) | 800 alloys, 100 parts | 300 | 12,000 | 3 | cruiser, 3 | 10 | 1,500 | 3 | 4 | 3.5% | 250 | 10, 5 | 0.50 | 11 cr, 5 alloys, 2.5 fuel, 1 parts |
| 6 | Fortress | Molecular Fabrication (tier 3) and Era III | 1,200 alloys, 150 parts | 360 | 16,000 | 4 | battleship, 3 | 12 | 2,000 | 4 | 5 | 4% | 300 | 12, 6 | 0.55 | 14 cr, 6 alloys, 3 fuel, 1.5 parts |
| 7 | Citadel | Planetary Defense Grid (tier 3) | 1,800 alloys, 220 parts | 420 | 22,000 | 6 | battleship, 4 | 14 | 3,000 | 4 | 5 | 4.5% | 350 | 14, 7 | 0.60 | 17 cr, 7 alloys, 3.5 fuel, 2 parts |
| 8 | Bulwark | Arcology Design (tier 3) | 2,500 alloys, 300 parts | 480 | 30,000 | 8 | battleship, 5 | 16 | 4,000 | 5 | 6 | 5% | 400 | 16, 8 | 0.70 | 20 cr, 8 alloys, 4 fuel, 2.5 parts |

Each size is one row of `starbases.json`. Reading the columns:

| Column | Meaning |
|---|---|
| Batteries | mounts of the gun battery weapon (mass accelerator); the starbase fights as a stationary ship with the size's hull and those weapons, with no armour or shields (`07 D40`) |
| Yard | builds civilian hulls (colony ships included) at every size and military hulls up to the named class, with that many constructions in parallel; ships past the slots are accepted and wait; pirate hulls never; one upgrade at a time (`07 D40`) |
| Docking | adds to fleet capacity: 16 + Σ docking of own starbases + tech; each ship over raises ship upkeep +2% |
| Storage | the stock capacity of the starbase's own market |
| Sensor, supply range | in jumps of 2 cells each |
| Repair | share of hull and armour restored per day to docked ships, paid at 1 alloy per 100 points from the starbase market, as far as its alloys pay; a damaged starbase mends its own hull the same way (`07 D40`) |
| Freight | adds to the empire's freight capacity |
| Security | the starbase term of the piracy formula |
| Upkeep | credits to the ledger plus commodities drawn daily at 1/30 from the starbase's own market; a base short of a commodity runs its yard, repair and upgrades at half rate and shows the shortage |

A trade hub needs an own starbase of size 2 or more (`07 D40`). The opening's home Station is size 2 with no crew at start; its two crew jobs fill from the domes in the first days, so the home yard builds at half rate until then (`07 D40`).

### Physical logistics: nothing teleports (`07 D39`)

- Every construction (a starbase's founding or upgrade, a ship in a yard, a station, a habitat) draws its daily bill only from the markets of its own system. The order needs an own starbase in the system, so stations and habitats in unclaimed systems are gone.
- The bill is posted as construction demand on the system's starbase market, at least its daily draw (`07 D40`), so the trade solver imports what the system lacks like any other demand, with freight, transit days and piracy exposure. In-system markets ship to the starbase market in 1 day.
- Progress each day is limited by what is in stock; a stalled construction shows "waiting for alloys" (the commodity with the smallest share). The order is accepted without the materials present.
- Construction is never carried by the construction ship. The ledger (credits, influence, unity, research) stays an empire number; only commodities are physical.
- Stations deliver their output to the system's starbase market, which offers it on to the system's colonies by in-system shipment.
- **The starbase market is the system's depot** (`07 D40`): it offers its stock above its own reserve, takes no push and pushes nothing; what reaches it is pulled to the deficits by need.
- Fuel and alloys reach a starbase only by shipment. Docked fleets top up from its fuel, and its market demands the docked fleets' use so the solver stocks it (`07 D40`).

### Self-sufficiency

Each starbase keeps a rolling 90-day count, in three 30-day buckets, of the units of its upkeep and construction commodities (alloys, parts, fuel, food, goods) that reached its market from its own system against those imported. With the local share `r`, the starbase gains

```
bonus = 25% × clamp((r − 0.5) / 0.25, 0, 1)
```

to build and upgrade speed, battery damage and hull, and the same share off its credit and commodity upkeep: nothing at half or less, the full bonus from 75% local. Imports pay only what the trade network already charges and are never otherwise penalised. The bonus is a colony-scoped `starbase.local` modifier row with its breakdown in the tooltip. Units are counted at dispatch with the amount that will land; stations' deliveries and local output count as local; no units gives no bonus (`07 D40`). The home system earns the full bonus on the Station's first turn, which takes its upkeep from 5 to 3.75 credits (`07 D40`).

### Supply and docking

A fleet is in supply within its supply range of a friendly starbase: readiness +10 per 30 days (to 100). Out of supply: no refuel, no repair, readiness −5 per 30 days. Docked means at an own starbase in the system. The supply bits are rebuilt with the trade network rather than rechecked daily against fuel stock (`07 D40`).

### Combat and capture

- A starbase fights as a stationary ship: hull from its size, its batteries, no armour or shields. The self-sufficiency bonus raises its maximum hull and its battery damage (as readiness `100 × (1 + bonus)`) (`07 D40`).
- **Capture** (`07 D113`, as fixed by `D149`): a starbase at hull 0 in a battle with an empire at war with its owner passes to that empire, with its system and its starbase colony, recorded as an occupation, with hull reset to 10%. It passes only once no armed ship hostile to the captor still fights in the battle; until then it stays in the fight at one hull point. Other colonies in the system still need an invasion. Pirates destroy a base at 0 instead.
- A starbase colony cannot be invaded; it changes hands only by capture (`07 D40`, `D113`).

### Holdings (persona-owned)

A **holding** (`07 D30`) is a starbase row with `kind: Holding`, owned by the persona, not an empire starbase. As designed in `01 §10.4`: built at a free orbit by the persona's construction ship (150 alloys + 20 parts, 90 days) or leased at an empire's starbase (50 credits per 30 days, reputation ≥ 30); tiers 1–3 with 2 / 4 / 6 modules (upgrades 200 / 500 alloys); modules Hangar (stores ships, fleet cap +2), Warehouse (cargo 500 units), Workshop (repair 10% per day while docked, refit), Barracks (hire crew and armies), Trade Post (sell cargo with no tariff), Comms (+1 sensor range), Battery (defence). Pirates raid an undefended holding by the piracy rule. As built in M12a (`07 D133`): holdings in systems that are unowned, the persona's or where it is favoured, with the Warehouse and Trade Post modules only; a raider at the holding's system with no armed persona fleet there takes the piracy share a day.

### Stations of sections (designed, `07 D284`)

The owner's calls of 2026-11-23 move starbases toward Starsector's stations. The size ladder keeps docking, storage, yard, supply range, sensor, repair, freight, crew, security and upkeep; the batteries column will leave the table and become a separately built **station** that fights in the tactical engine as a body of sections. The implementer session's first numbers, to be corrected by the milestone's headless matchups:

| Tier | Available from | Cost | Days | Section mounts |
|---|---|---|---|---|
| 1 | size 2 | 150 alloys + 20 parts | 120 | 2 |
| 2 | size 4 | 400 alloys + 60 parts | 240 | 4 |
| 3 | size 6 | 900 alloys + 140 parts | 360 | 6 |

- **The core.** The size table's hull will stay as the starbase's own hull, so capture and the self-sufficiency bonus keep their meaning. A size-1 starbase, or one with no station, will fight as the bare core.
- **Sections.** Each mount will take a lineage, Forge, Line or Lumen (the core lineages of `13 §3`), chosen per section so one station can mix them, and a role: **weapon** (the lineage's family: kinetic and missile, mixed, energy) or **defence** (Forge heavy armour, Line armour and shield, Lumen a capacitor shield). Each section: 40 alloys + 10 parts, 60 days, upkeep 0.5 credits + 0.25 alloys per 30 days, its own hull. A destroyed section stops working and is rebuilt at the starbase's repair rate, paid in alloys. While any defence section stands, hits on the core will take 50% less.
- **Combat.** In the tactical engine the station will be the `starbase` class core with its sections as fixed, separately targetable bodies around it, weapon sections firing all round; the fast resolver will sum the sections as weapons and effective hull. Self-sufficiency's 25% will apply to section hull and weapon damage.
- **Capture and retreat.** Capture on defeat stays. At capture, every fleet of the losing empire in the system will take the emergency jump to its nearest own system on the same tick, and that side's part of a tactical battle ends; fleets of personas in that empire's service retreat with it, allies' fleets fight on. Captured sections keep their lineage and damage; the core resets to 10% hull. This also closes `07 D157`'s capture ping-pong.
- **Commander.** One character slot per station: an officer applies combat skills to every section; a governor gives half their administration bonus to section rebuild speed and no combat skills. One character cannot command a station and a ship at once.
- **AI.** The AI will build the tier its size allows on its planning day, within its budget, and fill sections in its fleet's leading lineage, half weapons and half defence.
- **RTS and TB.** Tiers, sections and the commander change by Command at tick boundaries; capture and retreat happen in the combat tick.
- **Save.** A version bump: existing starbases get the tier their size allows, with Line weapon sections in place of their batteries.
- **UI.** A station panel on the starbase view: tier, mounts with lineage and role, the commander slot, every number with its modifier breakdown.
- **AI cores** are not station items: they go to the tech tree (a researchable tech improving characters' efficiency) and to the synthetic ascension, not to the station milestone.
- Size about 1.4k lines plus content.

### RTS and TB

Founding, upgrades, ship orders and colony-ship claims are Commands applied at tick boundaries. Commodity upkeep is drawn inside the existing per-colony market pass, and the only per-starbase work outside constructions is the heap timer of its 30-day turn (`07 D40`), so both modes behave alike.

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Basebuilding | Persona holdings with modules, a raid stance, a building item slot | 2026-11-02 | `07 D30` |
| Starbase redesign | No modules and no starbase-only techs; 8 sizes unlocked by construction techs; one table scaling with size; freight plus a local bonus of up to 25%; the starbase as a small colony and the claim; colony ships built only in a starbase's system; nothing moves without a shipment (replaces M8's tiers, modules, buildings and outposts) | 2026-11-08 | `07 D39` |
| M8b calls | Starbase body and deck; crew jobs gating the rate; founding and colony-ship claims; batteries with no armour or shields; repairs paid in alloys; the starbase market as depot; self-sufficiency counted at dispatch | 2026-11-09 | `07 D40` |
| Capture | A starbase at hull 0 passes to the hostile empire at 10% hull | 2026-11-14 | `07 D113` |
| Capture waits for defenders | Capture only once no armed hostile ship still fights in the battle (fixes the ping-pong of `D157`) | 2026-11-16 | `07 D149` |
| Holdings as built | Warehouse and Trade Post only; leasing, the other modules and the tier-3 upgrade not built | 2026-11-15 | `07 D133` |
| Stations on Starsector's model | The size ladder keeps the services; batteries become a station of sections in Forge, Line or Lumen, mixed per section; capture stays and the loser's fleets in the system retreat; an officer or governor commands; AI cores to the tech tree and synthetic ascension | 2026-11-23 | `07 D284` |

## Not yet built or still open

- **The station milestone** (`07 D284`): designed, about 1.4k lines; the spec session is to name and place it. Until it lands, the size table with its batteries column stays as built. The implementer session's numbers (tiers, section costs, the 50% core protection, commander effects) are proposals the owner may overturn.
- **Holdings**: leasing, the Hangar, Workshop, Barracks, Comms and Battery modules, and the tier-3 upgrade into a starbase colony are not built (`07 D133`).
- **AI cores** as a researchable tech and as synthetic-ascension persona cores belong to the tech tree (`07 D278`) and the synthetic-ascension scope, not yet built.
- `07 D40` is among the M-calls DESIGN lists as open for the owner's review.

## Discrepancies noted

- **Repair at a starbase.** `01 §10.3` says "hull/armour repair at shipyards (5%/d) or 2%/d at a friendly starbase"; `01 §10.4`'s size table gives repair per day of 1.5% (size 1) rising to 5% (size 8), paid in alloys, and `07 D40` built that table's rate.
- **Victory cross-reference.** `01 §10.4` says starbase colonies do not count toward "the colony counts of the era gates (§9.6) or victory (§15)"; in `01` victory is `§14` and `§15` is "Turn-based vs real-time specifics".
- **Modules.** `01 §10.4` says "There are no modules and no buildings" for starbases, and the same section gives persona Holdings (also starbase rows) "2 / 4 / 6 modules". Read here as the empire starbase against the persona's holding, but the two statements sit in one section without that distinction.

## Sources

- 01 §10.4 Starbases, §10.3 Combat resolution, §10.1 Hulls, §4.2 Interstellar exploration
- 07 D39, 07 D40, 07 D284; also `D30`, `D113`, `D133`, `D149`, `D157`, `D273` in the same file
- 09 §5 Starsector-style basebuilding
- 13 §3 Design lineages, species and civics
- 05 milestones (M8b, M10), 00 status, DESIGN tracker

## Related pages

- [Overview](Overview)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Trade and logistics](Design-Trade-and-logistics)
- [Economy and resources](Design-Economy-and-resources)
- [Ships and fleets](Design-Ships-and-fleets)
- [Combat](Design-Combat)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Factions and characters](Design-Factions-and-characters)
- [Megastructures](Design-Megastructures)
- [Starts and exploration](Design-Starts-and-exploration)


---

# Economy and resources

**Status:** built, as of 2026-11-23. The resource model shipped in E1 (merged 2026-11-14, save v20); the governor and AI economic governance in M9, the economy balance pass and the later balance rulings (`07 D100`–`D109`, `D162`, `D237`, `D264`, `D273`, `D274`).

The economy is the game's spine (`SPEC.md` pillar 1). Every colony and starbase is a market holding physical commodities with stock, production, demand and a price; goods are always somewhere, never "in the empire". On top of the markets sits an empire ledger of non-physical numbers (credits, influence, unity, research, administrative capacity). Seventeen commodities form a production chain from raw ore, gas, food and energy up to components, machine tools, charged cells and four fuel tiers, and energy is itself a physical commodity that backs the currency. The player sees alloys on one world at a given price, a shipment arriving in three days, and a shortage stalling a shipyard elsewhere; the governor and the AI read the same markets to decide what to build.

## How it works

### Two layers: the ledger and the markets

| Layer | Holds | Where | How it moves |
|---|---|---|---|
| Ledger | credits, influence, unity, research (physics, society, engineering), administrative capacity | a number on the empire | income and upkeep flow daily at 1/30 of the stated per-30-day rates; the UI shows 30-day rolling sums |
| Markets | physical commodities: stock, production, demand, price | per colony and per starbase colony (each starbase is a small colony with its own market, `07 D39`) | by production, consumption, construction draws and shipments; goods in transit are somewhere on a route |

Only commodities are physical; the ledger stays an empire number (`07 D39`, the spec session's call inside the owner's redesign). Content rates are written per 30 days and every market resolves every tick at 1/30 of them; only AI planning is periodic (`01 §7.1`, `CLAUDE.md`).

### The commodities

Seventeen commodities, within the registry cap of 32 (`docs/14 §2`, `07 D73`, built in E1). The first ten keep their save indices (crystal became rare ore, parts became components, machinery became machine tools; fuel stays as refined fuel) and the seven new ones are appended (`07 D57`).

| id | Name | Tier | Base price | Made by |
|---|---|---|---|---|
| food | Food | raw | 1.0 | Farmer |
| ore | Minerals | raw | 1.0 | Miner, mining station |
| gas | Volatiles | raw | 2.0 | Gas Extractor, gas station |
| rare_ore | Rare ore | raw | 3.0 | Rare Ore Miner, rare-ore station |
| alloy | Alloys | mfg | 4.0 | Metallurgist |
| goods | Consumer Goods | mfg | 2.0 | Artisan |
| components | Components | mfg | 8.0 | Fabricator |
| machine_tools | Machine tools | mfg | 12.0 | Toolmaker |
| luxury | Luxuries | mfg | 6.0 | Luxury Artisan |
| fuel | Refined fuel | mfg | 3.0 | Fuel Refiner |
| rare_metals | Rare metals | mfg | 6.0 | Metal Refiner (from rare ore); Smelter (from ore) |
| energy | Energy | raw | 1.0 | Technician, energy station |
| batteries | Batteries (empty) | mfg | 5.0 | Battery Maker |
| cells | Charged cells | mfg | 15.0 | any market with spare energy and empties (1 battery + 10 energy) |
| crude_fuel | Crude fuel | mfg | 2.0 | Depot Worker |
| exotic_matter | Exotic matter | raw | 10.0 | deep-space fields |
| exotic_fuel | Exotic fuel | mfg | 12.0 | Exotic Refiner (Zero-Point Power) |

Uses (`01 §7.2`): alloys go into districts, buildings and ships; components into advanced buildings, ship systems and starbase sizes; machine tools into megastructures, terraforming and advanced buildings; the four fuels into fleets; luxuries to rulers and entertainers; goods to pops, research and bureaucracy.

### The chain and what rare metals mean

The owner's direction for E1 (`07 D73`, 2026-11-13) was higher-level commodities built on commodity supply: alloys from minerals, rare metals and energy; machine tools from alloys; energy moved in batteries made from alloys and rare metals. Crystals merged into rare metals (option A).

The metals-chain retune (`07 D162`, the owner's call of 2026-11-16) changed what rare metals mean: they are the **higher tier's input**. Components, batteries and luxuries bind on them, while alloys take only a pinch (0.2 rare metals per 3 alloys), so alloys are bound by ore and energy. A Metal Refinery makes rare metals from rare ore one for one; where the empire makes no rare ore in the colony's system, a Smelter makes them from ore at two for one. The governor and the AI build the Smelter there instead of a Refinery that would starve (`governor.fixes` entries can name `needs`). The metals producers (Foundry, Metal Refinery, Smelter) are billed partly in ore (120 ore plus 40, 40 and 30 alloys), so they do not wait behind the alloys they unblock.

### Jobs (output per pop per 30 days, applied daily at 1/30)

The current numbers, from `01 §7.3` with the later changes of `07 D101` and `D162` (the content file `data/content/jobs.json` carries the same values):

| Job | Stratum | Inputs | Outputs |
|---|---|---|---|
| Farmer | worker | — | 7 food |
| Miner | worker | — | 5 ore (`07 D101`; was 4) |
| Technician | worker | — | 18 energy |
| Gas Extractor | worker | — | 2 gas |
| Rare Ore Miner | worker | — | 1 rare ore |
| Clerk | worker | — | 2 trade value, +1 amenity |
| Depot Worker | worker | 3 gas | 3 crude fuel |
| Metal Refiner | specialist | 1.5 rare ore | 1.5 rare metals |
| Smelter | specialist | 3 ore | 1.5 rare metals |
| Metallurgist | specialist | 2.4 ore, 0.2 rare metals, 0.75 energy | 3 alloys |
| Artisan | specialist | 4 ore, 0.9 energy (`07 D101`; was 6 ore) | 6 goods |
| Fabricator | specialist | 2 alloys, 1 rare metals | 2 components |
| Luxury Artisan | specialist | 2 goods, 0.4 rare metals | 2 luxuries |
| Fuel Refiner | specialist | 2 gas, 1 energy | 2 refined fuel |
| Toolmaker | specialist | 2 alloys, 2 energy | 2 machine tools |
| Battery Maker | specialist | 1 alloy, 1 rare metals | 2 batteries |
| Exotic Refiner | specialist | 1 exotic matter, 2 energy | 2 exotic fuel |
| Researcher | specialist | 0.5 goods | 6 research |
| Bureaucrat | specialist | 0.5 goods | 3 unity, +2 admin cap |
| Entertainer | specialist | 0.5 luxuries | +4 amenities, +1 unity |
| Merchant | specialist | — | 4 trade value, +10 freight capacity |
| Administrator | ruler | — | 3 unity, +0.5 influence, +3 stability |

Soldiers, medics and teachers are listed in `01 §7.3` and belong to [Pops and society](Design-Pops-and-society). Outputs scale with designation (+15%), stability, skill, adjacency and service, vitality, species traits, tech modifiers, buildings and edicts. Inputs are drawn from the local market each day; if an input is short, that day's output scales by the satisfaction ratio.

Jobs fill in content order within a stratum, so producers are listed ahead of their consumers: the metal refiner and the smelter come before the jobs that use rare metals, and the battery maker before the service jobs (`07 D266`, 2026-11-20). `datacheck` refuses a job listed after a job of its own stratum that uses its output (`01 §6.3`).

### Energy as a physical commodity

- **Production.** Technicians (18 energy each), energy stations (2 energy per deposit unit to their market) and Dyson structures make energy; technicians and energy stations no longer pay credits (`07 D57`).
- **Within a system** energy is beamed between markets at a **10% loss**, with no batteries.
- **Between systems** it travels only as **charged cells**. Energy never enters the trade network's source lists or the push to a hub, so no trade-flow row carries it (`07 D57`).
- **Charging and discharging** run in the markets pass. A market below its energy target (90 days of demand) discharges cells: each yields 10 energy and an empty battery, less 2% wear per cycle. A market above its target charges its empties at 10 energy each. A market short of energy asks for cells, one with a surplus asks for empties, and the trade network carries both, like any commodity.
- **Cells are a commodity in their own right:** bought, sold, stockpiled, shipped and burned as battery fuel (the owner's answer, `07 D73`).
- **Storage.** Energy past a market's storage is lost (no overflow sale), except at the treasury, which banks it.

### The energy-backed currency

The owner's call (`07 D73`, 2026-11-13): energy is a market commodity that backs the currency; the more energy, the more credits, with diminishing returns, and spending credits on resources makes resources dearer. The numbers are the spec session's (`docs/14 §3`), as built in E1 (`01 §7.4`, `07 D57`):

- **The treasury** is the capital's market. Each day it banks, as backing:
  - the **Energy backing** policy's share of its energy production less its demand: Standard ½ (the default), High ¾, Low ¼;
  - every charged cell it holds, at 10 energy each (it keeps the empties);
  - whatever energy its storage cannot hold.
- The treasury asks for cells for its empire's energy surplus in other systems and charges none itself, so energy made in other systems still backs the currency.
- **Issuance.** On the last day of the empire's staggered month it issues `2 × B^0.8` credits for the `B` energy banked since the last issuance. Doubling the backing gives about 1.74× the credits (`2^0.8`), so Dyson-scale energy keeps paying at a worsening rate.
- **Price index.** It sets the empire's price index `π = max(1, (S / 1.5B)^0.5)`, where `S` is the credits spent on market purchases over 30 days. Its purchases pay `price × π`. Every purchase estimate by the governor and the AI includes `π`.
- **Cross-empire settlement.** A cross-empire import pays at the ratio of the two empires' indices (`π_importer / π_exporter`).
- **Credits keep their other sources:** taxes, trade value and market sales.

In the standard opening the capital backs 15.45 energy per 30 days (half its +30.9 surplus) and issues `2 × 15.45^0.8` = 17.9 credits on its first issuance (tick 29) (`01 §3`).

### Fuels

Four tiers in MVP (`07 D73` option A; numbers `docs/14 §4`, `01 §10.2`). Each fleet has a preferred fuel and falls back down the tiers.

| Tier | Source | Range (units per refined unit of range) | Speed | Readiness | Combat |
|---|---|---|---|---|---|
| Crude | crude fuel | 1.5 | −15% | recovery in supply −50% | engines −10% |
| Refined | refined fuel (baseline) | 1 | — | — | — |
| Battery | charged cells | 4 | +15% | — | capacitor and shield regeneration +20% |
| Exotic (tier 3) | exotic fuel | 0.5 | +10% | — | damage, engines and regeneration +10% |

The tank holds units by tier and burns the preferred tier first, then down, then up; the effects follow the tier burning. Docked, a fleet refuels with its preferred tier from the market, falling back down. Fuel is a bottleneck by design: a large mid- or late-game navy needs either a strong refinery chain (volatiles plus energy) or a large energy surplus charged into cells (the owner's design note, `07 D73`). Fleet movement and supply are on [Ships and fleets](Design-Ships-and-fleets).

### Daily market resolution

Every market resolves every tick in this order (`01 §7.5`, `02 §3.4`): **production → trade dispatch → arrivals and allocation → prices**. Construction draws its materials in the separate daily `construction` system. Daily quantities are the per-30-day rates divided by 30.

1. **Production.** Pass 1 adds raw-job output and station deliveries to stock (available today). Pass 2 records each manufacturing job's input requests and potential output; nothing is drawn yet.
2. **Demand** per commodity, in units per day, is the sum of:
   - pops: cohorts × the stratum's basket × living standard × price factor;
   - jobs: the manufacturing input requests;
   - builds: the remaining cost of in-progress and queued constructions ÷ 90 (counts toward targets, prices and caps; never allocated here);
   - fleets: the fuel fleets drew here over the last 30 days ÷ 30.
3. **Trade dispatch** (see [Trade and logistics](Design-Trade-and-logistics)): shipments leave source stock today and are placed in the destination's inbound ring.
4. **Arrivals and allocation.** Available = stock + today's inbound slot. Consumers are served in priority order, each getting min(request, remaining): (1) pops' food; (2) manufacturing job inputs, by job priority then job id, each job's satisfaction being the minimum over its inputs; (3) pops' goods, then luxuries. Realised manufacturing output enters stock, usable tomorrow.
5. **Spoilage and storage.** Food stock is multiplied by 0.99829 a day (5% per 30 days). Stock is capped at the market's capacity (1,000 plus depots; a Depot adds 500 per commodity); overflow is sold at 50% of the price (energy overflow is lost instead).
6. **Satisfaction** is allocated ÷ requested; the satisfaction used by shortage effects is its 30-day rolling mean.

**Conservation** is a property test per commodity per tick: Δstock + Δinbound = production + purchases − allocated − dispatched + arrived − construction draw − spoilage − overflow − sales − pirate losses. Fleet cargo and the spot market are external source and sink terms (`01 §7.5`, `§10.6`).

### Prices

After allocation (`01 §7.5`; content `economy.json` `market`):

```
inflow    = today's production (pass 1 + realised pass 2) + today's arrivals
flowRatio = (demand + ε) / (inflow + ε)                         // ε = 2/30 units a day
target    = 90 × demand                                          // 90 days of demand
stockAdj  = clamp((target − stock) / max(target, 1/30), −0.5, 0.5)
rawPrice  = basePrice × clamp(sqrt(flowRatio) × (1 + 0.5 × stockAdj), 0.25, 4.0)
price     = 0.95 × price + 0.05 × rawPrice                       // ≈ 20-day smoothing
```

Behaviour: a non-perishable at steady state (inflow = demand, stock at target) prices at base. Goods nobody demands price at 0.75 × base while stock is held, lower (to the 0.25 floor) if they keep flowing in, and are exported as surplus. A small unmet demand prices at about 0.9 × base while a stock lasts and about 1.5 × base once it is gone. Raw → manufactured works within a day on one planet; manufactured → manufactured lags a day; any cross-market input lags its transit time.

For energy, the price reads only the market's own use against production plus arrivals, so a producer exporting its surplus as cells still prices as if the surplus were unused; with a full store, the price falls under 0.5 × base once inflow exceeds 2.25× the market's own demand (`docs/research/2026-11-20-energy-price-and-rare-metals.md`). Counting the charging draw as demand (option E1 of that note) was dropped on 2026-11-20, since it would lift the median price on paper but move no energy (DESIGN change log).

The `01 §18` price sanity target: per commodity, the median of price ÷ base over market-days with demand lies in [0.5, 2.0], shortage days are under 15% of market-days with demand, and the empire freight utilisation median is under 90%.

### Shortage effects

Continuous while the 30-day rolling satisfaction is under 1 (`01 §7.6`):

| Commodity short | Effect |
|---|---|
| food | fertility × sat; happiness −25 × (1 − sat); starvation (×5 mortality) after 90 days at sat < 0.5 |
| goods | happiness −20 × (1 − sat) for all strata; research and unity jobs output × sat |
| luxury | happiness −10 × (1 − sat) for strata with luxuries in their basket; entertainer output × sat |
| raw inputs (ore, gas, rare ore) | downstream job output × sat (that day) |
| alloys, components | construction and shipbuilding progress × sat (daily draw) |
| fuel | fleets supplied from this market: readiness −10 × (1 − sat) per 30 days |

A starbase short of an upkeep commodity runs its yard, repair and upgrades at half rate (`01 §10.4`).

### Buying, selling and stock policies

Any market trades credits for goods with an off-map spot market at the local price: buy at `price × 1.1` (times the empire's price index), capped at `max(0.2 × stock, 50)` units per commodity per 30 days; sell at `price × 0.9`, uncapped (`01 §7.7`). Per market and commodity, standing orders set the stock target (`auto`, `stockpile N`, `export surplus above N`, `forbid import`, `forbid export`, and since M17d `ban`). Both are described in full, with the Galactic Exchange, on [Trade and logistics](Design-Trade-and-logistics).

### The ledger

**Credits** (`01 §7.4`):

| In | Out |
|---|---|
| energy issuance (above) | building, district, starbase, station, ship and leader upkeep |
| taxation: `taxRate × (G + Σ I)` (`01 §7.9`); at Standard exactly the trade value plus a pop tax of 0.3 / 0.6 / 2.0 per worker / specialist / ruler per 30 days | edict costs and subsidies |
| market sales | market purchases (at `price × π`) and cross-empire imports |
| | civil upkeep (`07 D273`, below) |

- **Negative credits** are allowed. While the balance is negative, all job outputs fall 25%, fleet readiness falls 5 per 30 days, and after 180 consecutive days the Recession situation starts. The owner kept the −25% rule (DESIGN change log, 2026-11-17).
- **Taxation, private wealth and vitality** (`07 D28`): tax rates Low 0.10, Standard 0.25, High 0.40, Punitive 0.60 with a one-year lock; vitality drifts toward `clamp(1 + (fairTax − taxRate), 0.5, 1.2)` with a half-life of about 3 years and multiplies every job output. Gross commerce `G = 4 × trade value` is collected at the hub (see [Trade and logistics](Design-Trade-and-logistics)). The full rules are on [Pops and society](Design-Pops-and-society).
- **Influence:** +3 base, +0.5 per administrator, +0.5 per rivalry (max 2); spent on claims, edicts, reforms, diplomacy and hub designation; hard cap 1,000.
- **Unity:** from bureaucrats, administrators, entertainers and monuments; spent on traditions and ascension perks.
- **Research:** per field, from researchers, the capital bonus, research stations and events.
- **Administrative capacity:** base 30 plus bureaucrats, offices and tech. `sprawl = colonies × 2 + systems × 0.5 + pops × 0.1 + districts × 0.2`. Each 1% of sprawl over the cap adds 1% to tech and tradition costs and 0.5% to all upkeep.
- **Civil upkeep** (`07 D273`, the owner's call of 2026-11-21, built): each day an empire pays `pops × u(n) ÷ 30` credits, where `u(n) = popUpkeep × max(0, n − colonyFree) × empire.popUpkeep`, with `popUpkeep` 0.01 credits a pop a month, `colonyFree` 10, and `n` the planetary colony count (starbases aside). It is booked on the sprawl row and not surcharged by sprawl. Wide costs more than tall. The founding gate that goes with it is on [Colonies and surfaces](Design-Colonies-and-surfaces); the statecraft techs, the Ringworld, the Matrioshka Brain, the `centralizer` ruler trait and the persona's administration skills reduce both.

| n (planetary colonies) | 12 | 15 | 20 | 22 | 30 | 36 |
|---|---|---|---|---|---|---|
| u(n): credits a pop a month | 0.02 | 0.05 | 0.10 | 0.12 | 0.20 | 0.26 |

### Construction: local sourcing, slots and the start rule

- **Local sourcing** (`07 D39`): every construction (a starbase's founding or upgrade, a ship in a yard, a station, a habitat) draws its daily bill only from the markets of its own system. The bill is posted as construction demand on the system's starbase market and the trade solver imports what the system lacks. An order needs an own starbase in the system. The rule is described with the rest of physical logistics on [Trade and logistics](Design-Trade-and-logistics).
- **Construction slots** per system: `max(3, 2 + local alloys per 30 days / 10)`, up to 6 (`07 D58`; content `governor.slots` = [2, 10, 6, 3]). Builds past the slots wait unstarted, draw nothing and post no demand; started builds keep their slots (`07 D55`).
- **The start rule** (`07 D100`): a build starts drawing only once its market holds 30 days of its daily draw of each bill commodity (`market.gatherDays` 30), or the whole bill if smaller. Until then it waits unstarted, posting demand so imports come; the UI shows "waiting for stock: Alloys 12/30 days". This applies to player builds too; there is no force-start.
- **Draw order.** Manufacturing holds back each market's next daily construction draw, so builds rank ahead of job inputs; builds draw in a daily rotation, and a build stalled on a missing commodity yields its slot (`07 D57`).
- **Bills carry credits** where content says so (`07 D102`): the mining district costs 10 alloys and 100 credits, since a mine billed in the ore it makes deadlocked ore-short markets.
- **Stuck builds are cancelled** (`07 D264`): a district, building, ship or upgrade that has started drawing and made no progress for over 360 days (`budget.cancelAfter`) is cancelled by the AI for its own builds, and by the governor for the builds it ordered. The slot and hex are freed and nothing drawn is refunded (no cancel refund exists). The player's own orders are never cancelled.

### Governance: the colony governor

The player's colony governor (`01 §17`) acts when a colony's state changes or once every 30 days per colony (staggered by colony id), never every tick.

- **Shortage projection.** It builds a fix when a shortage is projected within 360 days: `stock / max(dailyDemand − dailyProduction − imports30d, 0.0003) < 360`, never for a commodity that another market in the same system holds 90 days of (`07 D33`). The system's most populous own colony also counts its starbase's refined-fuel draw and stock (`07 D57`).
- **Fix order** (content `governor.fixes`): food, energy, goods, gas, refined fuel, alloys, rare ore, ore, components, luxuries, rare metals (Metal Refinery where rare ore is made in the system, else Smelter), machine tools, batteries, crude fuel. Energy's fix sits right after food, with a Power Plant (2 technicians) as the fallback (`07 D57`).
- **Back-off.** A fix that could not be ordered is not retried at that colony for 3 governor periods, or until a construction in its system finishes or the colony gains a slot or a surveyed hex (`07 D242`).
- **Slots and freight.** It keeps one building slot free (the fuel fix may take that spare slot, `07 D162`), and builds a Spaceport at the busiest market when empire freight utilisation exceeds 80%.
- **Markets automation.** All stock policies `auto`; sells overflow; buys to cover a shortage while credits stay above 3 months of upkeep.
- **The food buffer** (`07 D274`): on every colony's monthly governor timer, whether or not its governor is on, a market whose imports cannot land its food demand buys food on the spot market. Its rules are on [Trade and logistics](Design-Trade-and-logistics).

**The supply forecast and ordering rule** (`07 D55`, the owner's call of 2026-11-13: a supply-aware governor plus construction slots, with local sourcing intact). The forecast, computed only when a governor timer, an AI plan or a tooltip asks, adds per system and commodity over a 90-day horizon: stock (a starbase depot's at half), inbound shipments, production less non-construction demand, less every committed build's remaining bill, plus what an outside hub can deliver within the horizon. A build is ordered only if its system has a free slot and its bill fits the forecast. A producer of a missing input is judged on stock and inbound alone, and a starbase upgrade may count the empire's stock of commodities the empire produces. The player may order anything; the starbase panel shows the forecast with the slots and shortfalls. Further guards: the upkeep margin (a build only while the 30-day margin after its upkeep stays ≥ 0, unless its output value pays its upkeep back), fill-aware builds (only while idle pops plus 12 months' growth staff the new jobs), and the hub's projection of its spokes' unmet need.

### Governance: the AI economic model

The owner asked for the AI's economic root cause to be fixed rather than patched and chose option A (`07 D237`, 2026-11-17, built 2026-11-18): **one budget per AI empire, kept in the sim**, refreshed once per AI plan on the 30-day stagger and saved. The AI and the governor (for AI empires) read and debit it. The player's governor is unchanged.

- **What it holds:** a forecast of recurring income (spot sales excluded), committed upkeep (including builds and colonies in flight with their sprawl step), and a margin against a reserve of `reserveMonths` (3) × upkeep; the supply balance for food, energy, alloys, rare metals and goods, with growth and spoilage.
- **Pricing:** every commitment is priced by its marginal recurring cost (upkeep, sprawl step, the supply it draws, what it makes at expected staffing); a colony ship is priced as the colony it founds, including its civil-upkeep step (`07 D273`).
- **The priority ladder:**
  1. restore the reserve by selling surplus (never food, a short commodity, what its own builds wait for, or its spokes' 90 days);
  2. after `shedAfter` (3) plans in deficit, retire the building with the least output value per upkeep;
  3. fix supply at the best site in the empire (most idle pops, a slot, inputs by quantity);
  4. grow within the headroom (colonies, upgrades, military, pacts).
- **The Supply rung:** a commodity its markets make under `budget.critical` (0.5) of their demand is critical; the ladder may place a second producer for it in the same plan, at another colony, admitted on the empire's stock and inbound (`07 D237`, 2026-11-18).
- **Deficits** (`07 D264`, the owner's call of 2026-11-19): when a colony's or the empire's projected 30-day balance of a commodity is negative, the AI acts instead of waiting for the market. The ladder answers every forecast shortage, worst first, with up to `budget.deficitMax` (3) producers per commodity per plan, counting producers in flight; then each non-capital colony with a negative 30-day balance gets one local producer, skipped when another own market in the system makes a daily surplus that covers the gap (`07 D273`'s energy fix).
- **Halt growth** (`07 D264`): `Command::HaltGrowth` stops births and immigration at a colony. The player toggles it on the colony panel; the AI halts a colony whose food deficit has no producer placed this plan and whose stock covers under 90 days (`haltCoverDays`), lifting it on a positive food balance or when a food producer completes.
- **Expansion pace:** a colony ship is built only while the empire's planetary colonies plus ships in flight stay under `colonyBase` (8) + pops ÷ `popsPerColony` (40) (`07 D237`), and the founding gate of `07 D273` applies to every empire; the stricter binds.
- **Site supply** (`07 D274`, the owner's call of 2026-11-21: "Supply resilience should be a factor in colony formation"): each candidate site's food resilience R (the share of a new colony's food the empire can land there) multiplies its score by `1 − 0.5 × (1 − R)`, and a site with R < 1 and no farmland is vetoed (never leaving the AI with no legal site). The player sees the same figure on the colony-ship target and the Colonize button.
- **The old guards it replaced:** the colony-ship guard, `affordable`'s margin path for AI empires, the AI's producer and warship checks, the credits-freeze in `supply()` and the per-colony food fix for AI empires; the parked `07 D200`–`D202` patches were folded into the ladder.

AI commercial pacts, personalities and difficulty are on [AI and victory](Design-Ai-and-victory) and [Diplomacy and war](Design-Diplomacy-and-war); the pact trade rules are on [Trade and logistics](Design-Trade-and-logistics).

### The opening economy

The standard origin's opening ledger nets +8.2 credits on day 1 (+9.45 running, once the home system's self-sufficiency bonus takes 25% off the Station's upkeep), within E1's ±2 of +7 (`01 §3`, `07 D57`). The homeworld carries a Metal Refinery with 2 refiners from day 1; the opening stock adds rare metals 40, rare ore 20, energy 100 and refined fuel 50 (about 14 months). There is no day-1 refinery; the governor and AI build one before refined fuel falls under four months of demand (`07 D57`). The opening and the Sol start are on [Starts and exploration](Design-Starts-and-exploration).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Commodity count | 10 commodities (option A) at first; deferred to E1 by the owner and replaced by the 17 of `docs/14` | 2026-11-01; 2026-11-13 | `07 D4` |
| Ledger stays non-physical | Only commodities are physical; credits, influence, unity and research are empire numbers | 2026-11-08 | `07 D39` |
| Resource model | Energy backs the currency with diminishing returns; charged cells tradable as they are; crystals merge into rare metals; four fuel tiers with a preferred fuel and fallback. Replaced `01 §7.2`'s 10-commodity table, the energy credits of `§4.2` and `§7.9` and the single fuel of `§10.2` | 2026-11-13 | `07 D73`, `docs/14` |
| E1's numbers | Industry's energy draw halved from `docs/14`'s first proposal (alloys 0.25 per unit, goods 0.15); backing policy Standard ½ / High ¾ / Low ¼; the treasury banks cells and overflow and asks for cells from other systems; technicians 18 energy. Replaced `docs/14 §2`'s 0.5 and 0.3 | 2026-11-14 | `07 D57` |
| Supply-aware governor and slots | The owner's call: a supply forecast and construction slots with local sourcing intact; upkeep margin, fill-aware builds, the hub's projection | 2026-11-13 | `07 D55` |
| Construction slots | `max(3, 2 + local alloys / 10)` up to 6. Replaced `07 D55`'s `2 + floor(alloys / 20)` capped at 6 | 2026-11-14 | `07 D58` |
| The start rule | A build draws only once its market holds 30 days of its draw | 2026-11-14 | `07 D100` |
| Ore numbers | Miners 5 ore (was 4), metallurgists 2.4 ore (was 3.6), artisans 4 ore (was 6), in the owner's order of the supply try | 2026-11-14 | `07 D101` |
| Mines billed in alloys and credits | Mining district 10 alloys + 100 credits. Replaced 100 ore + 10 alloys | 2026-11-14 | `07 D102` |
| Building-materials priority | The owner's lever (construction draws before job draws at a source) was built, measured and then dropped by the owner; parked on `lever-park` | 2026-11-15 | `07 D108` |
| Metals-chain retune | Rare metals become the higher tier's input; metallurgists 0.2 rare metals per 3 alloys (was 0.6); the Smelter as fallback; metals producers billed partly in ore | 2026-11-16 | `07 D162` |
| AI economic model | One saved budget per AI empire with a priority ladder; replaced the separate guards and the parked debt patches | 2026-11-17 | `07 D237` |
| Rule-change bundle | No change to the sprawl cost; build-retry back-off; AI pacts only from year 25 (with `D236`) | 2026-11-18 | `07 D242` |
| Deficits, halt growth, stuck builds | The AI answers projected deficits; a halt-growth command; AI and governor cancel builds stuck over 360 days | 2026-11-19 | `07 D264` |
| Energy's price box | Counting the charging draw as demand dropped (moves no energy); the job-order fix (R1) cleared the box | 2026-11-20 | DESIGN change log, `07 D266` |
| Producers fill first | Rare-metal producers and the battery maker ahead of their consumers in the job order; `datacheck` enforces it | 2026-11-20 | `07 D266` |
| Empire size | A founding gate that steepens with colonies, and civil upkeep per pop rising with colony count; `colonyStep` shipped at 16. The owner declined a harsher sprawl cost (`D242`) | 2026-11-21 | `07 D273` |
| Supply resilience | Food resilience weighs AI site choice and vetoes farmless unreachable sites; a food buffer as the backstop for every empire | 2026-11-21 | `07 D274` |

## Not yet built or still open

- **Owner review still open:** the M5 implementer calls (`07 D34`) and the M8c calls (`07 D46`) stay on DESIGN's list of `07` call sets open for review.
- **Starsector-style extra commodities** (supplies, crew, marines, organics, contraband) were offered and are not yet scoped (DESIGN, open questions).
- **The stall box** ("constructions stuck a full year ≤ 15% of construction-years", carried from M9) is still unchecked in `05 E1`. After `07 D264`'s cancel rule and `D265`, the stuck share measured 0.2% mean (seeds 1–5 to year 150); the queue ("waiting over a year") box measured 20.2% mean, holding on two of five seeds.
- **Rare-metal ladder gates** (R2 of the 2026-11-20 research note: one free slot for a short commodity's producer) is held until rare metals are shown to stay binding (`07 D266`).
- **Perishable stock target** (a shorter `targetDays` for food) waits for a post-MVP economy pass (`07 D194`, `D195`).
- **Hub food forwarding** (hubs offering food above their reserve) was not built; the counter showed the import cap, not the source side, limits long-spoke food (`07 D194`).
- **A per-colony tax override** is listed for 1.0 (`01 §7.9`).
- **Difficulty extras:** Grand Admiral's free hub and full map are not built (`07 D55`).
- **The persona's AI cores** go to the tech tree and the synthetic-ascension scope, not to the economy (`07 D284` call 5).

## Discrepancies noted

- **Miner and artisan numbers in `01`.** `01 §7.3` gives "Miner | worker | — | 4 ore" and "Artisan | specialist | 6 ore, 0.9 energy | 6 goods"; `01 §7.2` gives goods as "1 ore + 0.15 energy" per unit; and `01 §3`'s opening line reads "homeworld 16 + moon 8 + rocky 8 + asteroid station 4 = 36; artisans 24 + metallurgists 7.2 = 31.2 → +4.8". `07 D101` (2026-11-14) says "Artisans draw 4 ore for their 6 goods (were 6)", "Miners make 5 ore (were 4)", and "The opening's ore line is now 44 made against 20.8 demanded". `data/content/jobs.json` has miner `"ore": 5` and artisan `"ore": 4`. D101 is later and matches content, so this page uses 5 and 4; the `01` text was not updated.
- **The resource model's status in the tracker.** `docs/DESIGN.md`'s pillars table lists "Resource model | ... | designed (E1)", while its own change log says "E1 merged (2026-11-14): 17 commodities, energy backing the currency, charged cells, four fuel tiers; save v20" and `05 E1`'s acceptance boxes are ticked. This page treats E1 as built.
- **Old commodity names in the shortage table.** `01 §7.6` lists "ore / gas / crystal" and "alloy / parts"; crystal became rare ore and parts became components in E1 (`07 D57`). This page uses the new names.
- **The governor's fix order.** `01 §17` gives "food → goods → gas (gas-vent district or station) → fuel (Refinery) → alloys → crystal and parts (crystal station, Workshop) → rare metals ... → luxuries (Luxury Works)", with no energy entry. `07 D57` says energy's fix comes "right after food", and `data/content/economy.json` `governor.fixes` lists food, energy, goods, gas, fuel, alloy, rare_ore, ore, components, luxury, rare_metals, machine_tools, batteries, crude_fuel (luxuries before rare metals). This page gives the content order.

## Sources

- SPEC.md
- 01 §3 Empires and species (the opening)
- 01 §7.1 Two layers
- 01 §7.2 Commodities
- 01 §7.3 Jobs
- 01 §7.4 Ledger flows
- 01 §7.5 Daily economic resolution
- 01 §7.6 Shortage effects
- 01 §7.7 Buying and selling
- 01 §7.9 Taxation, private wealth and vitality
- 01 §10.2 Movement and logistics (fuel)
- 01 §10.4 Starbases
- 01 §17 Automation defaults
- 01 §18 Default numbers and targets
- 05 Milestones (E1, M9, the 1.0 table)
- 07 Decisions: `D4`, `D28`, `D33`, `D39`, `D55`, `D57`, `D58`, `D73`, `D100`–`D108`, `D162`, `D194`, `D195`, `D237`, `D242`, `D264`, `D266`, `D273`, `D274`, `D284`
- 14 The resource model
- DESIGN.md (pillars, open questions, change log)
- Research: supply diagnosis (2026-11-16)
- Research: shortage diagnosis (2026-11-19)
- Research: shortage options (2026-11-19)
- Research: energy price and rare metals (2026-11-20)
- `data/content/jobs.json` and `data/content/economy.json` (content values)

## Related pages

- [Trade and logistics](Design-Trade-and-logistics)
- [Pops and society](Design-Pops-and-society)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Starbases](Design-Starbases)
- [Ships and fleets](Design-Ships-and-fleets)
- [Megastructures](Design-Megastructures)
- [Starts and exploration](Design-Starts-and-exploration)
- [AI and victory](Design-Ai-and-victory)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Progression](Design-Progression)
- [Architecture and performance](Design-Architecture-and-performance)
- [Overview](Overview)


---

# Trade and logistics

**Status:** built, as of 2026-11-23. The network, dispatcher and piracy shipped in M5 and were moved onto the hyperspace grid in M8c; physical logistics in M8b; trade fleets, subsidies and embargo, the Galactic Exchange and smuggling in M17a–d (merged 2026-11-22 to 2026-11-23); gateway trade in M19a (merged 2026-11-23).

Goods move between markets only by shipment. Every empire runs a hub-and-spoke trade network over the hyperspace grid, and a daily dispatcher ships surplus to deficit along it, paying freight, transit days and losses to piracy, under each market's accessibility and import cap. Nothing teleports: a construction draws only from its own system, so a remote starbase upgrade waits for its alloys to arrive. On top of the abstract network sit commercial pacts with tariffs, physical trade convoys, subsidies and embargoes, a neutral Galactic Exchange, smuggling, and gateway shortcuts. Trade is drawn on the map, so the player sees where goods flow, where pirates feed and where a blockade has cut a route.

## How it works

### Nothing teleports: physical logistics

The owner's redesign of 2026-11-08 (`07 D39`, built in M8b): resources never transmit magically through the empire but physically go place to place; resource flows to bases are managed, with self-sufficiency rewarded. It replaced every "nearest own market by hops" draw.

- **Local sourcing.** Every construction (a starbase's founding or upgrade, a ship in a yard, a station, a habitat) draws its daily bill only from the markets of its own system. The bill is posted as construction demand on the system's starbase market, and the trade solver imports what the system lacks like any other demand: with freight, transit days and piracy exposure. In-system markets ship to the starbase market in 1 day. A stalled construction shows "waiting for alloys". An order is refused only where the system has no own starbase (`01 §10.4`).
- **The construction ship carries nothing:** its bill is demand at the site's starbase market (`07 D39`, the spec session's call).
- **Colony ships** can only be built in the same system as a starbase with enough local resources (`07 D39`).
- **Fuel and alloys reach a starbase only by shipment;** docked fleets repair in alloys and refuel from that market (`01 §10.2`).
- **Energy crosses systems only as charged cells** (`docs/14 §2`); see [Economy and resources](Design-Economy-and-resources).
- **Cargo is physical.** A raiding fleet with no cargo space takes nothing (`07 D39`); a deep outpost's output fills a 500-unit hold that a freighter must collect (`01 §2.1`).
- **Self-sufficiency** (`07 D39` option A, the owner's call): imports pay only the network's freight, transit time and piracy exposure, and a starbase supplied from its own system gains a bonus. Each starbase keeps a rolling 90-day count of its upkeep and construction commodities (alloys, parts, fuel, food, goods) that came from its own system against those imported. With local share `r`, it gains `25% × clamp((r − 0.5) / 0.25, 0, 1)` to build and upgrade speed, battery damage and hull, and the same share off its upkeep: nothing at half or less, the full bonus from 75% local (`01 §10.4`).

### The network: hubs, fields, spokes and legs

From `01 §8.1`, on the hyperspace grid of `07 D43` (there are no hyperlanes):

- **Hubs.** Every empire has at least one hub (the capital by default). A hub is designated for 50 influence and needs a starbase of size 2 or more in the system. The AI designates a second hub at a market past 30 route days, or past 15 once eight colonies lie outside the home system and only the capital is a hub, upgrading that starbase first if only its size blocks it (`07 D55`). The player gets a suggestion when a spoke exceeds 30 route days (`01 §17`).
- **Fields.** Each hub keeps a Dijkstra search of route days from its cell over the month's hyperspace grid at freighter speed 100, through its empire's corridors, stopped once its targets (its empire's market systems, its hubs and its pact partners' hubs) are settled. Each hub is recomputed on its own day of the month, chosen when it becomes a hub as the day with the fewest hubs (earliest on a tie) and saved (`07 D268`), so every field is at most a month old. A corridor built, drifted or broken, or an ownership change, recomputes the fields concerned (`07 D46`).
- **Spokes and hub pairs.** Each market links to its nearest hub by route days, walked back along that hub's field (a spoke). Hubs link to each other (hub pairs) along the same fields. Commercial pacts link each party's hubs to the other's. Spokes, hub pairs and pact routes are the network's **legs**; each keeps the piracy regions it crosses with its days in each, the systems it passes (for blockades) and its corridor share.
- **A shipment route** is source spoke → source hub → hub pair → destination hub → destination spoke. A shipment arrives route days + 1 after dispatch; the inbound ring holds 64 days (`07 D46`).
- **In-system shipments** take 1 day, cost `amount × 0.125` freight, cross no region and are never blockaded from outside. Markets in a system that holds a hub are 0 days from it. A shipment that leaves a system adds 2.5 freight-days for each in-system end. The longest route is 60 days; longer routes are split at the hub (`07 D46`).
- **Corridors** give freight at half cost on a route's corridor share and +0.2 piracy security (`01 §2.1`).
- **The Mass Driver** is a structure the trade network reads: a linked pair's route days are scaled (`07 D78`, as summarised in `07 D283`). Megastructures are on [Megastructures](Design-Megastructures).

### Freight capacity

Empire freight capacity, in tonnage per 30 days (`01 §8.2`):

```
capacity  = 100 base + Σ starbase freight by size + Σ Spaceports × 75 + Σ merchants × 10
spend     = amount × (routeDays + 2 × 2.5) × 0.05      // the corridor share at half
```

- The daily allowance is `capacity / 30`; unused allowance does not carry over. 100 units over 15 route days spends 100.
- Cross-empire shipments spend the **importer's** freight. Trade fleets add nothing to capacity (their cargo moves physically, `07 D275` call 1).
- **Utilisation** is the 30-day rolling share of capacity spent. The governor builds a Spaceport at the busiest market when it exceeds 80% (`01 §17`).
- Starting capacity is 295 per 30 days (base 100, the homeworld's size-2 Station 100, Highport's Spaceport 75, two merchants 20).

Starbase freight and piracy security by size (`01 §10.4`):

| Size | 1 Outpost | 2 Station | 3 Starport | 4 Starhold | 5 Bastion | 6 Fortress | 7 Citadel | 8 Bulwark |
|---|---|---|---|---|---|---|---|---|
| Freight | 25 | 100 | 150 | 200 | 250 | 300 | 350 | 400 |
| Security | 0.15 | 0.30 | 0.35 | 0.45 | 0.50 | 0.55 | 0.60 | 0.70 |

### Accessibility and the import cap

Each market has an accessibility factor (Starsector's), from `01 §8.2` with the gentler curve of `07 D193`:

```
accessibility = clamp(0.3 + 0.7 × (1 − daysToHub / 80) − 0.5 × maxPiracyOnSpoke + modifiers, 0.1, 1.2)
```

`daysToHub` is the market's route days to its hub; `maxPiracyOnSpoke` is the worst region level on its spoke, its own region included. Imports into a market are capped at **`accessibility × 2 × demand` per day**, and the cap counts **landed** tonnage: the dispatcher ships up to the cap ÷ the route's landed share, so a lossy route still lands its cap and pirates take more of the larger shipment (`07 D272`, the owner's call of 2026-11-20). A market with accessibility under 0.5 can therefore land at most `accessibility × 2` of its demand however much freight there is; that structural case is what the food buffer below covers (`07 D274`).

### The daily dispatcher

Per empire, then cross-empire, each tick after production (`01 §8.3`, with `07 D164`, `D236`, `D265`, `D272`; content `economy.json` `trade`):

1. **Positions.** Only markets below target or above 1.5 × target are considered. For each market and commodity: target = the policy target (default 90 × daily demand); position = stock + Σ inbound + 30 × (potential production − demand); deficit = max(0, target − position); surplus = min(stock, max(0, position − 0.5 × target)), keeping a half-target reserve and never promising unproduced goods.
2. **Push (extraction colonies).** A market with zero demand for a commodity and more than 10 in stock ships the excess to the nearest same-empire market that demands it (its hub if the hub demands it), without waiting for a deficit there. This is how a mining dome's ore reaches the homeworld every day.
3. **Order.** Deficits are sorted by price ÷ base price, most stressed first (subsidised markets first, see below).
4. **Candidates.** Each deficit takes up to k = 6 candidate surplus markets reachable in the network, ordered by cost per unit landed:
   ```
   cost = (0.02 × routeDays + (1 − accessibility) + (blockaded ? ∞ : 0) + 0.06 × Σ (level × days)) / landedShare
   landedShare = Π (1 − min(1, lossMul × level × days)) over the regions crossed,  lossMul = 0.05
   ```
   A route landing under `landFloor` (0.5) is not used, the push included (`07 D164`). The shipment is min(deficit, surplus, source stock, the day's freight left ÷ the per-unit freight, the import cap). Stock is deducted from the source at dispatch.
5. **Losses.** In each region crossed at piracy level p for d days the shipment loses min(1, 0.05 × p × d) of what is left (`07 D265`); the goods vanish and their value at the source price goes to the loot pool of the sector of the region's nearest system. Blockaded routes carry nothing. The remainder lands in the destination's inbound ring at slot routeDays + 1.
6. **Cross-empire** (after the domestic dispatch, against the partner's stocks as they stand): see commercial pacts below.
7. **Flows** are recorded per leg (30-day rolling) for the map layer and the market screens.

Two further rules sit beside it:
- **The starvation draw** (`07 D55`): a spoke whose food in stock and inbound is below 30 days of its net need (`trade.starvingDays` 30) draws on its hub below the hub's reserve, down to the hub's own 30 days of demand. It keeps no landed floor, since starvation outranks the reserve (`07 D164`).
- **Energy** never enters the source lists or the push; it reaches other systems only as charged cells (`07 D57`).

### Stock policies

Per market per commodity (`01 §7.8`): `auto` (the solver targets 90 days of demand), `stockpile N` (target N; imports until reached), `export surplus above N`, `forbid import`, `forbid export`, and `ban` (M17d, below). Defaults are `auto` everywhere; never touching them is the low-overhead path.

### The spot market

Any market trades credits for goods with an off-map spot market at the **local** price (`01 §7.7`): `market.buy` creates stock at `price × 1.1` (times the empire's price index), capped at `max(0.2 × stock, 50)` units per commodity per 30 days as a rolling allowance; `market.sell` destroys stock at `price × 0.9`, uncapped. Both are external terms in the conservation test. Stock past capacity is sold at 50% of the price. The markets automation buys to cover a shortage while credits stay above 3 months of upkeep (`01 §17`).

**The food buffer** (`07 D274`, the owner's backstop, built 2026-11-21): on every colony's monthly governor timer, whether or not its governor is on, a market that makes less food than it uses is checked. If its imports cannot land its demand (access × `importCapMul` under `governor.bufferMargin` 1.1) or it is starving (30-day food satisfaction under 0.5), and its food stock plus inbound falls under 60 days (`bufferDays`) of demand, it buys food on the spot market up to 90 days of demand, within the allowance. It needs only the credits for the purchase, not the three-month reserve. It also tops up the system's own starbase market. The food row's Forbid import turns it off; the player's default is on, and the AI's is always on. The spend books on the purchases row.

### Commercial pacts and tariffs

- **Cross-empire trade needs a commercial pact.** Empire A's deficits draw on B's surpluses in step 6. A's ledger pays B's ledger at dispatch, at the source price times the ratio of the two empires' price indices (`π_A / π_B`, `07 D57`).
- **Tariffs** (MVP, `01 §8.3` step 5): A's trade policy sets the tariff t (free 0, protectionist 20%). It adds `3 × t` to the candidate cost and caps A's imports from B at (1 − t) × the accessibility cap. There is no tariff cash flow; tariffs act only through the solver.
- **Domestic sources first** (`07 D236`, the owner's call of 2026-11-17, T1 and T2 both): a pact source is a candidate only when A's own candidates cannot cover the deficit, and it ranks after them; A's pact imports over 30 days stay under `pactImportShare` (25%) of A's 30-day income. An empire with no income imports nothing over a pact.
- **AI pacts** (`07 D242`): an AI proposes Commerce only from year 25, and only when it and the partner each hold 2 trade hubs and meet the budget's reserve. This replaced `07 D116`'s "AIs propose no commercial pacts to each other". Diplomacy itself is on [Diplomacy and war](Design-Diplomacy-and-war).

### Trade value

Clerks (2) and merchants (4) produce trade value, collected daily at the hub along the spoke (`01 §8.4`). Gross commerce `G = 4 × tradeValue`; the state's share converts at the hub by the commerce policy: `credits += taxRate × G × (1 − 0.5 × maxPiracyOnSpoke)`. At Standard this is exactly `Σ tradeValue × (1 − 0.5 × maxPiracyOnSpoke)`. Lost trade value feeds the loot pool, which gives piracy a reason to exist on every spoke.

### Piracy

Per **region** of 6 × 6 hyperspace cells (12 × 12 regions at Medium with the margin), `piracy ∈ [0, 1]`; a system's piracy is its region's (`01 §8.5`, `07 D46`). Daily, at 1/30 of these rates:

```
exposure = (unitsThrough30d + tradeValueThrough30d) / 100     // only cargo that survives the region counts (07 D165)
security = starbaseTerm (the best own starbase in the region; 0.15 at size 1 to 0.70 at size 8)
         + min(0.6, 0.2 × patrollingShipsInRegion)
         + 0.1 × claimedSystems / systems
         + (an intact corridor through the region ? 0.2 : 0)
piracy   = clamp(piracy + (0.05 × exposure − 0.08 × security − 0.01) / 30, 0, 1)
```

- **Loot and raiders.** Losses go to the loot pool of the sector of the region's nearest system. When a sector's pool exceeds 2,000, a system in it has piracy over 0.5 and no friendly fleet is there, `clamp(lootPool / 1,000, 1, 8)` raiders appear with stance raid, joining a raider fleet already there up to 8 ships; the pool loses 1,000 per raider (`07 D174`). Raid fleets were bounded as a bug fix after Huge galaxies grew to 58,717 pirate ships (`07 D174`).
- **Pirate behaviour.** Raiders engage civilian ships and weaker fleets and flee stronger ones; in raid stance they take 10% of a market's stock per 30 days; after 180 days unchallenged they found a pirate base (+0.3 to the region's piracy, once, until destroyed); after 360 days unchallenged they retire, returning 1,000 per ship to the pool.
- **Route exposure** is Σ level × days over the regions crossed; it feeds the solver's cost and the landed share.
- **Worked examples** (`01 §8.5`, 50 units per 30 days through one region): an unclaimed region climbs from 0 to 0.5 in about 33 months; two patrolling corvettes (security 0.4) bring 0.5 down to 0.2 in about 18 months. Inside owned space a size-1 starbase alone holds piracy flat below about 44 units per 30 days (60 with the region fully claimed), a size-2 Station below about 68 (84). Heavy hub spokes need patrols, hangars, corridors or Starholds even in owned space.
- **Player counters:** patrol-stance fleets, starbase sizes and hangar modules, shorter spokes (more hubs), fewer trade-dependent planets. Rising piracy shows on the trade layer as red haze.

### Blockade

A route is **blockaded** when a system it passes holds a hostile fleet with blockade stance (`01 §8.1`). A blockaded route has infinite cost in the solver and carries nothing; blockaded legs are barred on the map. A blockade at either gate system closes a gateway jump (`07 D283`), and a blockade in its system closes the Galactic Exchange (`07 D275`). In-system shipments are never blockaded from outside. Separately, a fleet at a colony of an empire it is at war with steals 5% of the market's stock per 30 days into its cargo (`01 §10.6`).

### Trade fleets (M17a)

The owner's calls of 2026-11-21 (`07 D275`), built and merged 2026-11-22:

- **Routes.** `AssignTradeRoute` gives a freighter fleet (any fleet with cargo room; escorts merged into it ride along) a route between two own markets, or an own market and a pact partner's. The player picks up to three priority commodities; by default the destination's three most stressed commodities the source can spare.
- **Loading and unloading.** Docked at the source it loads each priority's stock above half its target (the dispatcher's reserve). Docked at the destination it unloads into stock **outside the import cap and the freight allowance**, which is what a convoy is for. A convoy adds nothing to freight capacity (call 1 A; `01 §8.2`'s trade-fleet capacity term was dropped).
- **The leg** is an ordinary fleet move with fuel, storms, intercepts and incidents. A convoy destroyed loses its cargo to the sector's loot pool.
- **Cross-empire** unloading pays as the dispatcher's pact flows do (source price × the indices' ratio, no tariff cash); no pact, no route.
- **Insurance** (call 2 A): every empire may insure a route; each loaded leg pays 1.2 × the expected loss × the hold's value at the source price. A convoy lost is paid its cargo and its lead hull's alloys. Premiums and payouts book on the ships expense row.
- **Automation.** An idle freighter with automation on takes a route from its capital's market to an own market in another system that cannot land its demand (`07 D274`'s test). The AI builds up to two freighters (`ai.caps.tradeFleets`) while it has capped markets.
- **Hull.** The MVP Freighter carries 150 cargo (`01 §10.1`).

### Subsidies and embargo (M17b)

Built 2026-11-22 (`07 D275` calls 5 A and 6 A, `01 §8.7`):

- **Subsidy.** `Subsidize { market, level }`, levels 1–3, multiplies every stock target of the market by 1.5 / 2 / 3 and sorts its deficits first in the dispatcher, for level × 0.05 × Σ base price × demand a day, booked on the edicts row. It raises targets only (the price step keeps its own target) and does not lapse in debt.
- **Embargo.** A diplomatic action that closes a commercial pact for 5 years and bars a new one, at −50 opinion with the target and −10 with its pact partners; convoys' routes between them are freed. It cannot be lifted early. The AI embargoes each rival it holds, one a plan.

### The Galactic Exchange (M17c)

Built and merged 2026-11-23 (`07 D275` call 3 A; the spec session's call to trade it as a pseudo pact partner, 2026-11-22):

- **Where.** A neutral market of the unowned row at the unclaimed system nearest the centroid of the first three pact-holding empires' capitals, founded on the third empire's first commercial pact (or monthly when absent, for older saves). No one owns it; there is no Exchange treasury.
- **Members** are empires with any commercial pact. In step 6, after a member's own pact sources, one Exchange candidate is tried per deficit: the member buys from the Exchange's stock at index × 1.05 × its price index, within the pact-import share. Each member's market offering most sells surplus into it at index × 0.95 while it is under target (one sale per member and commodity a day).
- **Index and target.** Monthly, per commodity, the index is the members' demand-weighted median price (base price while no member demands it); the target is 90 days (`trade.exchangeDays`) of the members' median market demand.
- **Freight is real.** Shipments run over a lossy leg from the member's hub field to the Exchange's cell, losing cargo to piracy like any. The pillar is a physical economy, so lossy freight is the point (the spec session's call, 2026-11-22). The 5% fee each way is money destroyed (buys) and created (sales). Exchange tonnage counts toward economic victory's cross-empire tonnage.

### Smuggling (M17d)

Built and merged 2026-11-23 (`07 D275` call 4 B):

- **Contraband.** A sixth stock policy, `ban`, per market: its target is 0 and no legal import reaches it (dispatcher, push and food buffer). Its black-market price is the market price, at most twice base, read at sale.
- **Detection.** A market's empire catches a smuggler where it patrols the system, at 0.3 less 0.25 per Smuggling rank. A catch seizes the cargo into that market, fines 50 and adds to the region's detected smuggling, which fades over 360 days.
- **The persona.** With its transponder off it may trade where outlaw and sell contraband at the black-market price, untaxed; Smuggling grows by such trades. A catch also costs reputation.
- **AI smugglers** (the owner chose B, AI empires too): on its planning day an AI runs up to two smuggling routes into the nearest rival market (rivalry or war) that bans a good one of its markets makes in surplus, when the margin beats the catch risk. It takes an idle freighter first, else one off a legal route. A caught AI smuggler costs −15 opinion for a year and a year's cooldown; the rival's patrol choice weighs detected smuggling beside piracy.

### Gateway trade (M19a)

The owner's call of 2026-11-23 (`07 D283` call 1 A: a route shortcut, as the Mass Driver's), merged the same day. An empire's own hub pair jumps when both hubs' systems hold a Gateway it may use (its own, or an open-borders partner's, as fleets use them) and the jump is quicker than the flown route. The jump's leg takes the gateway's days, crosses no piracy region, loses nothing, has no corridor share, and is closed by a blockade at either gate. Freight is paid on the shorter days only. It is derived at route time, with no field rebuild when a gate is built or falls. Pact routes and the Exchange's legs still fly (their hub pairs are two empires'). No AI builds a Gateway yet (that waits for M19c).

### Map visualisation

The trade layer is part of MVP and is the pillar "trade is visible" (`01 §8.8`): each leg drawn along its route's cells, width proportional to 30-day tonnage, coloured by dominant commodity; hubs ringed; piracy haze per region; blockaded legs barred; shortages as a pulsing market ring; goods in transit as moving ticks along the route; the player's trade routes from source to destination, as wide as the freighter's hold (`07 D275`). The hyperspace layer shows clouds, storms, corridors and the deep (`07 D46`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Physical logistics | Nothing teleports; every construction draws from its own system; imports through the trade network; self-sufficiency rewarded with up to 25%. Replaced every "nearest own market by hops" draw | 2026-11-08 | `07 D39` |
| Hyperspace routes | Hub fields over the hyperspace grid, regions of 6 × 6 cells, 60-day route limit, a 64-day inbound ring. Replaced hyperlane hops | 2026-11-13 | `07 D43`, `D46` |
| Second hub | Past 30 route days, or 15 once eight colonies lie outside home. Replaced `01 §17`'s 30 days alone for the AI | 2026-11-13 | `07 D55` |
| AI commercial pacts (first) | AIs propose no commercial pacts to each other; superseded by `D242` | 2026-11-14 | `07 D116` |
| Pirates bounded | Raids merge into a standing pirate fleet up to a cap and retire to the loot pool | 2026-11-16 | `07 D174` |
| Landed-share pricing | Sources priced per unit landed; routes landing under 0.5 refused | 2026-11-17 | `07 D164` |
| Surviving cargo counts | A region's traffic counts only what survives it | 2026-11-17 | `07 D165` |
| Accessibility curve | `accessibility.days` 40 → 80 (gentler) | 2026-11-17 | `07 D193` |
| Pact trade protects the importer | Domestic sources first (T1) and pact imports capped at 25% of income (T2) | 2026-11-17 | `07 D236` |
| AI pacts from a buffer | Diagnose first (B), then pacts only from a strong buffer; built as the year-25, two-hub, reserve floor | 2026-11-17; 2026-11-18 | `07 D235`, `D242` |
| Piracy loss weight | `lossMul` 0.1 → 0.05 in the landed share and the losses (option A); a 0.3 floor and AI patrols against refused routes not chosen | 2026-11-19 | `07 D265` |
| Balanced hub days | A new hub takes the least-loaded day of the month, saved. Replaced market id mod 30 | 2026-11-20 | `07 D268` |
| Import cap on landed tonnage | The cap counts landed tonnage; the dispatcher ships up to cap ÷ landed share. A food buffer or auto-buy for small import-fed colonies not chosen | 2026-11-20 | `07 D272` |
| Supply resilience | AI site choice weighs food resilience and vetoes farmless unreachable sites (D), with a food buffer as the backstop (B); a food floor on the import cap (C) not chosen | 2026-11-21 | `07 D274` |
| M17's six calls | Convoys physical only (capacity term dropped); insurance for every empire at 1.2 × expected loss; the Exchange a neutral unowned station with a 5% fee; smuggling with AI smugglers (B); three subsidy levels; embargo as `§8.7` with the AI using it | 2026-11-21 | `07 D275` |
| Exchange as a pseudo pact partner | Members trade with the Exchange through step 5 over lossy legs (A), not an off-map spot market (B) | 2026-11-22 | `07 D275`, DESIGN |
| Gateway trade | A route shortcut for an empire's own hub pairs (A), not gateway edges in the fields (B) | 2026-11-23 | `07 D283` |

## Not yet built or still open

- **The AI builds no Gateways** until M19c, the AI's megastructure advisor (`07 D283`), so gateway trade appears only for the player in AI-only runs.
- **AI patrols against refused routes** (R2 of `07 D265`) are kept as a later design option.
- **Narrowing each field's pact targets** (S2 of `07 D267`) would change pact routes and is not built.
- **A walk cut in the dispatcher** (compacting spent sources, "C2 at this scale") was proposed by track M and not built (`07 D274`); dispatch leads the design-scale Huge worst tick (see [Architecture and performance](Design-Architecture-and-performance)).
- **Hub food forwarding** and a **perishable stock target** are not built; long-spoke food stays a probe for the MVP (`07 D194`, `D195`).
- **The 1.0 Freighter** (cargo 200) is listed in `01 §10.1`.
- **Owner review still open:** the M5 calls (`07 D34`) and the M8c calls (`07 D46`) stay on DESIGN's open list.

## Discrepancies noted

- **Do embargoes close Exchange trade?** `07 D275` call 6 A reads "also closes the target's Exchange trade with the embargoing empire and any route between them". The M17c build record in `07 D275` and the DESIGN autonomous-calls row (2026-11-23) say "embargoes leave Exchange trade alone, since its stock is pooled" (alternative listed: "embargoes closing the target's Exchange trade"). `docs/00-status.md`'s M17c merge line says "embargoes close it between the parties". Which is current is not clear from the sources; this page does not state either as the rule.
- **Where the Exchange stands.** `01 §7.7` says "at the unclaimed system nearest the first three's capitals"; `07 D275` call 3 A and the M17c build say "the unclaimed system nearest the centroid of the first three pact-linked empires' capitals". This page follows the build record.
- **Piracy loss weight in `01 §8.3`.** `01 §8.3` steps 3–4 still write the landed share as `Π (1 − min(1, 0.1 × level × days))` and losses as "lose min(1, 0.1 × p × d)". `01 §8.5` says losses are "`0.05 × level` per day in each region (`07 D265`)", and `data/content/economy.json` has `"lossMul": 0.05`. `07 D265` is later, so this page uses 0.05.
- **Trade fleets, subsidies and the Exchange labelled 1.0.** `01 §8.6` and `§8.7` are headed "(1.0)", `01 §7.7` says "1.0 adds a Galactic Exchange", `01 §17` lists "Trade fleets (1.0)" and `01 §10.1` says empires build the Freighter "for the 1.0 trade fleets". `docs/00-status.md` records M17a–d merged on 2026-11-22 and 2026-11-23. This page treats them as built.
- **Hyperlanes in the spec.** `SPEC.md` pillar 2 says goods "travel along hyperlanes"; `07 D43` replaced hyperlanes with hyperspace (DESIGN: "No hyperlanes").
- **The import cap's wording in `01 §8.2`.** It reads "Imports into a market are capped at `accessibility × 2 × demand` per day", without `07 D272`'s rule that the cap counts landed tonnage. This page states D272's rule as current.

## Sources

- SPEC.md
- 01 §2 Galaxy generation (corridors, deep outposts)
- 01 §7.7 Buying and selling
- 01 §7.8 Stock targets and policies
- 01 §8 Trade routes and logistics
- 01 §8.1 Network
- 01 §8.2 Capacity
- 01 §8.3 Daily trade dispatch
- 01 §8.4 Trade value
- 01 §8.5 Piracy
- 01 §8.6 Trade fleets
- 01 §8.7 Subsidies and embargo
- 01 §8.8 Map visualisation
- 01 §10.2 Movement and logistics
- 01 §10.4 Starbases
- 01 §10.6 Raid stance and cargo
- 01 §17 Automation defaults
- 05 Milestones (M5, M8b, M8c, the 1.0 table: M17, M19)
- 07 Decisions: `D39`, `D43`, `D46`, `D55`, `D57`, `D78`, `D116`, `D164`, `D165`, `D174`, `D193`–`D195`, `D235`, `D236`, `D242`, `D265`, `D267`, `D268`, `D272`, `D274`, `D275`, `D283`
- 14 The resource model
- 00 Status (merge records, 2026-11-21 to 2026-11-23)
- DESIGN.md (pillars, autonomous calls, change log)
- Research: supply diagnosis (2026-11-16)
- Research: shortage diagnosis (2026-11-19)
- Research: shortage options (2026-11-19)
- `data/content/economy.json` (`trade`, `piracy`, `governor` values)

## Related pages

- [Economy and resources](Design-Economy-and-resources)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Starbases](Design-Starbases)
- [Ships and fleets](Design-Ships-and-fleets)
- [Combat](Design-Combat)
- [Megastructures](Design-Megastructures)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Factions and characters](Design-Factions-and-characters)
- [AI and victory](Design-Ai-and-victory)
- [Presentation and UI](Design-Presentation-and-ui)
- [Architecture and performance](Design-Architecture-and-performance)
- [Overview](Overview)


---

# Ships and fleets

**Status:** built, as of 2026-11-23. The hull catalogue and fleet hierarchy in H1 (2026-11-13), on M8's ships and logistics; the persona's per-ship fitting in M12a and its officers in M12b.

Ships are known by named hulls (94 in the catalogue) while six general size classes carry the rules, in the manner of Starsector. Each hull belongs to one of six design lineages, is fitted from an ordnance-point budget with weapons, fighter wings and hull mods, and carries one ship system. Ships sail in fleets that are divided automatically into squadrons and task groups under a doctrine, supplied with fuel and readiness from starbases and support ships. The persona's own fleet is the only one fitted ship by ship; navies use designs.

## How it works

### Size classes

Six classes carry the rules (`07 D70`; `13 §2`). Classes are data in `hulls.json` (`tactics.classes`), so no code names a class (`07 D69`). A seventh class row, `starbase`, holds the starbase's tactical shape.

| Class | Role (`13 §2`) | Commanders | Pilotable | OP scale (mods) | Size | Radius | Turn |
|---|---|---|---|---|---|---|---|
| Fighter | strike craft in wings of 1–6, launched from hangars; not built alone | 0 | no | 0 | 0.25 | 8 | 200 |
| Frigate | screens, raiders | 0 | yes | 1 | 1 | 18 | 120 |
| Destroyer | escorts, gunships; the largest pilotable class | 0 | yes | 2 | 2 | 28 | 80 |
| Cruiser | line, carriers | 1 | no | 3 | 4 | 44 | 45 |
| Capital | battle line, carriers | 2 | no | 5 | 8 | 66 | 25 |
| Super-capital | flagships of a navy | 3 | no | 8 | 16 | 95 | 15 |

Size, radius and turn are the `tactics.classes` values in `data/content/hulls.json`; commanders, piloting and the OP scale are stated in `07 D65`, `D69` and `D70` and match that file. A hull short of its commanders fights at 0.75 rate of fire, thrust and turn rate (`07 D65`; see [combat.md](Design-Combat)).

**Yards.** A hull needs a starbase whose size's yard allows its class (`01 §10.1`, `§10.4`). Capitals build at a Bastion (size 5); super-capitals at a Fortress (size 6) and above (`07 D71`), or at a Shipyard Ring, whose `yard` hook builds super-capitals with 2 extra slots and repair × 2 (`07 D79`; see [megastructures.md](Design-Megastructures)). The yard column of `01 §10.4`'s size table predates H1 (it gives the Bastion "cruiser" and the Fortress "battleship"); `07 D71` is later and `data/content/starbases.json` carries it: Outpost civilian, Station frigate, Starport destroyer, Starhold cruiser, Bastion capital, Fortress, Citadel and Bulwark super-capital.

### Lineages

Lineages are design philosophies, not quality tiers (`13 §3`). Three are core and buildable from the start; three are exotic (`07 D70`).

| Lineage | Philosophy (`13 §3`) | Source (`07 D71`) | Classes it covers | Resolver dealt / taken |
|---|---|---|---|---|
| **Forge** (core) | heavy armour, kinetic and missile mounts, slow, cheap to repair | start | frigate to super-capital | 1 / 1 |
| **Line** (core) | balanced armour and shields, mixed mounts | start | frigate to super-capital | 1 / 0.9288 |
| **Lumen** (core) | light armour, strong capacitors and shields, energy mounts, fast | start | frigate to super-capital | 1 / 1 |
| **Veil** (exotic) | phase/void hulls that slip out of combat space | the tier-3 physics tech Veil Drives, after Corridor Stabilisation | frigate to capital | 1.3009 / 0.9081 |
| **Precursor** (exotic) | unique systems no tech reaches | not buildable: derelicts in a fifth of looted deep caches | frigate to super-capital | 0.6991 / 1.04 |
| **Automata** (exotic) | the deep faction's hulls | not buildable: derelicts from caches; capture by Boarding planned | frigate to capital | 1.1587 / 1.4872 |

The class coverage and the dealt/taken multipliers are `lineages[]` in `data/content/hulls.json`; the multipliers are the resolver's per-lineage levers (`07 D72`). A derelict exotic hull joins the looting fleet at half hull (`07 D71`). Each lineage has a colour shown on the ship's core (`07 D71`).

**Pirate conversions.** Pirates field Mk II conversions of core hulls with extra mounts and built-in d-mods (`13 §3`); the catalogue's raider hull is the Brigand Mk II.

**Civic and species flavour.** Civics and species traits add modifier rows to lineages; no hull is species-locked (`13 §3`). Built rows (`07 D71`), applied to an empire's design stats and to its ship builds' daily draws:

| Civic | Effect |
|---|---|
| Warrior Culture | Forge builds 10% cheaper |
| Merchant Republic | Line builds 5% cheaper |
| Technocracy | Lumen +5% shields |

Species ship sets are visual: G2's Blender generators are one builder per lineage, parameterised by class and species ship set (silhouette, greeble, palette) (`05 G2`).

### The catalogue

H1 built 94 hulls (`05 H1`): 64 military, 12 fighters, 16 support and civilian, and 2 deep encounters (the Brigand Mk II raider and the Void Leviathan). All names are original (`07 D71`). The twelve pre-H1 hulls keep their ids and indices and take new names: corvette Bannerwick, destroyer Halyard, cruiser Lodestar, battleship Grandmarch, automaton Cogwarden (`07 D69`).

`13 §4` asked for 16 / 15 / 13 / 10 / 4 per class, which sums to 58, not 64; the build makes 64 as **18 frigates, 17 destroyers, 14 cruisers, 11 capitals and 4 super-capitals**, of which 6 Veil, 6 Precursor and 6 Automata (`07 D71`).

Military hulls by class and lineage (display names from `data/content/strings/en.json`):

| Class | Forge | Line | Lumen | Veil | Precursor | Automata |
|---|---|---|---|---|---|---|
| Frigate (18) | Rivetjaw, Cinderling, Slagbiter, Anvilkin | Bannerwick, Tidewarden, Quillon, Gallowglass, Keelrunner | Glasswing, Prismfin, Sunthread, Lanternmoth | Hushwing, Duskglass | Firstlight | Cogling, Rustwasp |
| Destroyer (17) | Slagwall, Pitforge, Breakwater, Cindermaul | Halyard, Warrantor, Keelmark, Morrowgard | Shimmerline, Aureole, Flarewing, Beamglass | Nightveil, Gloamskiff | Oldcrown | Gearmantis, Cogwarden |
| Cruiser (14) | Smeltmaster, Crucible, Ingotreach, Pyreholt | Lodestar, Constancy, Ironwake | Prismata, Halobright, Glintcrown | Umbravault | Firstwatch, Elderwatch | Cogcathedral |
| Capital (11) | Mountainforge, Anvilthrone, Cinderhall | Grandmarch, Sovereignty, Steadhold | Sunvault, Dawnspire | Eclipsehold | Eldermonarch | Cogmonolith |
| Super-capital (4) | Worldanvil | Concordance | Lightmother | — | Eldest | — |

Fighter wings (12, role `fighter`, carried in bays, not buildable alone): Dartfly, Needlejack, Pikebolt (Line); Whipsting, Emberbee, Burrfly (Forge); Sparkmoth, Glowmite, Thornseed (Lumen); Wraithlet (Veil); Eldermote (Precursor); Cogspark (Automata).

Support and civilian hulls (16): Wayfinder (science), Masonwright (construction), Seedbearer (colony), Cartling / Haulwright / Bulkmaw (freighters S / M / L), Fuelwick / Brimfull / Wellspring (tankers S / M / L), Marshaldeck (transport), Voyagehall (liner), Scrapcrane (salvage rig), Pathreader (survey ship), Towline (tug), Mendhold (repair tender), Provender (supply tender). The freighters S and L, the three tankers, the liner, salvage rig, survey ship, tug and both tenders are new in H1 (`07 D71`).

**Balance.** Every military hull fights its class's Line reference (4 Bannerwick, 3 Halyard, 2 Lodestar, 1 Grandmarch or 1 Concordance), auto-designed from every component with full commanders in open space; `tools calibrate --tune-hulls` bisected a factor on hull, armour and shields for each hull outside 25–75% win rate. Two passes moved 47 hulls; every military hull is now within 25–75%, the worst at 74% (`07 D71`, `05 H1`).

`01 §10.1` still prints M8's five-hull table (Corvette 300 hull HP, 40 alloys and 2 parts, 60 days, speed 120, through the 1.0 Carrier) and its civilian table. Those rows were the M8 base; per-hull numbers are now the `hulls.json` catalogue rows after the H1 factors.

### Hull schema

A hull row holds (`13 §4`, `05 H1`): size class, lineage, role, tech unlock and the yard class needed; ordnance points; mounts (size S/M/L × type); hangar bays; one ship system; built-in hull mods; hull, armour sectors, shields and capacitor; speed and turn rate; logistics (fuel per jump, cargo, crew minimum and maximum, supply use); readiness; and a composition cost used only by the AI and auto-design, never a deployment limit.

### Mounts, ordnance points and fitting

**One layout for designs and fitted ships** (`07 D69`). A design is `fit: [u8; 24]`: slots 0–15 are its mounts in the hull's order (a weapon, or a fighter wing on a bay), slots 16–23 are eight hull mods; `u8::MAX` is empty. The persona's per-ship fitting reads the same 24 bytes.

**Mounts** are a size (S, M, L) and a type: kinetic, energy, missile, hybrid (kinetic or energy), universal, or a bay (`B`, wings only). A weapon fits a mount of its size or larger and of its type (`07 D69`).

**Ordnance points (OP)** (`07 D69`):

| Item | OP |
|---|---|
| Weapon, S / M / L | 5 / 10 / 20 |
| Fighter wing | its own OP (8–14 in `hulls.json`) |
| Hull mod | its base OP × the class's `opScale` (frigate 1, destroyer 2, cruiser 3, capital 5, super-capital 8) |
| Built-in hull mod | free |
| Unspent OP | becomes capacitor at `capPerOp` = 6 shields per point |

`datacheck` checks every hull's auto-design against its budget, over every component and over the starting set; `DesignCreate` refuses a design over budget (reason `Power`). Civilian and support hulls carry no OP and are unfitted (`07 D69`).

**Components** (`data/content/hulls.json`, 59 rows):

| Kind | Count | Rows |
|---|---|---|
| Weapons | 12 | kinetic: mass driver (S), coilgun (S), railgun (M), mass accelerator (L); energy: laser (S), pulse laser (S), heavy laser (M), particle lance (L); missile: rocket pod (S), swarm missiles (M), torpedo (M), cruise missile (L) |
| Fighter wings | 12 | one per fighter hull |
| Hull mods | 32 | base OP 0–4; d-mods exist as mods flagged `dmod`, refused in a design (`07 D69`) |
| Combat computers | 3 | picket, line, artillery |

Weapon families follow `01 §10.1`: kinetic (high damage, low accuracy, good against armour), energy (high accuracy, good against shields), missile (long range, counterable by point defence). Each weapon has a range band, damage, cooldown in days, accuracy, tracking and penetration.

**The computer stays the hull's**, set by its role, with no computer slot (`07 D69`). In the resolver the computer type picks targets (smallest, largest, nearest in band) and the preferred band (`01 §10.3`).

### Ship systems

Ship systems are data: 32 systems over 8 kinds, one per hull, shared across hulls, each with power, seconds and cooldown. The tactical engine runs them; the AI triggers each kind on its own condition, and a pilot triggers it with F (`07 D69`).

| Kind | Systems in `hulls.json` |
|---|---|
| burn | Sprint Coil, Lunge Drive, Surge Vanes, Comet Kick |
| damper | Brace Field, Kinetic Sponge, Stone Skin, Bulkhead Lattice |
| overdrive | Breech Rush, Hot Loader, Volley Fervour, Gunlock Surge |
| phase | Veilstep, Ghostfold, Umbral Slip, Fade Gate |
| flare | Chaff Bloom, Glint Swarm, Decoy Fan, Mirror Spray |
| uplink | Spotter Net, Range Sextant, Longsight Mast, Plotting Choir |
| repair | Mender Swarm, Weld Choir, Scab Foam, Patchwork Drones |
| capacitor | Charge Well, Static Bank, Arc Reservoir, Flux Flush |

(System names are the `systems[]` ids in `data/content/hulls.json`, title-cased.)

### Designer and auto-design

A design is a hull plus its fit within the OP budget (`07 D69`, replacing `01 §10.1`'s "hull + chosen components within power budget"). **Auto-design** is the default: it generates a sensible design for each hull when new components are researched (`01 §10.1`). The player may make designs in the ship designer (`05 H1`). Auto-upgrade (refitting docked fleets) was cut in M8; obsolete designs are marked for refits (`05 M8`, `07 D38`). On loading a version-14 save every design is refitted by auto-design from what its empire knows (`07 D69`).

### The persona's fleet: per-ship fitting

Only the persona's personal fleet gets per-ship state (`07 D44`, `01 §19.1`):

- **Refit** (`Captain::Refit`) gives a ship its own design row, at a starbase the persona's empire owns or where it is trusted; its d-mods stay; the fit passes the design rules with the components the persona's empire or the yard's owner knows; its OP less the s-modded hull mods must fit the hull (`07 D135`).
- **s-mods.** A story point builds a hull mod into a ship, its OP off the budget (`07 D134`).
- **d-mods** are lasting damage carried by salvaged or badly repaired hulls (`01 §19.1`). The Damage Control skill removes one at a starbase for 50 / 35 / 20 alloys; Boarding's ranks plan one fewer d-mod on salvaged hulls (`01 §19.2`).
- **Templates** (`Captain::Template`) make a ship's fit an empire design with its d-mods cleared and no s-mods, which the empire's yards build like any design (`07 D135`).
- **Officers** (M12b): a leader class `officer`, assigned to a ship of the personal fleet, with two traits that add skill. There are 2 slots plus `persona.officers` (Coordination ranks, the neural link implant). Each officer's skill adds 2% to the personal fleet's damage, spread over its ships. A lost ship frees its officer; a companion officer whose loyalty reaches 0 leaves and takes their ship (`07 D140`).
- **Size cap.** The personal fleet holds at most `4 + renown / 50` ships; merges past it are refused (`07 D134`, `01 §19.2`).

### Fleet hierarchy

Ship → squadron → task group → fleet → navy (`07 D70`, `13 §5`). The hierarchy is **computed from a fleet's ships, not stored** (`military::groups`, `07 D72`):

| Level | Rule (`07 D72`, `13 §5`) |
|---|---|
| Squadron | the fleet's ships in id order, grouped by class and lineage, up to `squad` (20) each |
| Task group | led by a commander; holds up to `2 + coordination` squadrons; task group `g` takes squadrons from `g × (2 + coordination)` onward |
| Support group | task group 7, holding the fleet's support and civilian ships |
| Fleet | led by an admiral; holds up to `2 + admiral level / 3` combat task groups |
| Navy | the empire's whole fleet; its size set by fleet capacity |

Until M11's leaders, `rules.command` sets coordination 0 and admiral level 3, so a fleet holds 3 task groups of 2 squadrons (`07 D72`); `hulls.json` still holds those values. A merge or rejoin past the cap is refused (`Cap`). Fleets move and fight as one (`13 §5`).

**Detach and rejoin** (`07 D72`). A stopped fleet's task group leaves as its own fleet with its share of fuel and stock, keeps its parent, and takes the group's doctrine as its own. It rejoins its parent when both stand in the same system, within the cap. Merge, split and reinforce are commands from M8 (`05 M8`).

### Doctrine

Doctrine is a packed `u32` per fleet plus eight task-group overrides (`fleets.doctrine`); a task group without an override takes the fleet's (`07 D72`).

| Setting | Values (`07 D72`) |
|---|---|
| Engagement range | standard keeps the officer's; close closes to 45% of the longest weapon's range; long holds at 95% |
| Retreat | a hull share, which raises the side's withdrawal threshold |
| Formation | a squadron's formation in a large action (line, wedge, screen, englobe) |

The **stance** stays the fleet's (`01 §10.2`): aggressive, patrol, evasive, passive, blockade, and `raid` (pirate fleets only). Patrol is aggressive toward pirates and toward empires its owner is at war with, passive otherwise.

### Support ships

Support ships sail in the support group and draw stock loaded at a docked market, so nothing teleports (`13 §5`, `07 D72`):

| Ship | Effect (`07 D72`) |
|---|---|
| Tankers S / M / L | add 40 / 100 / 250 to the fleet's tank |
| Repair tender | holds 200 stock (alloys); mends 1% of hull a day outside a dock, drawing alloys at the starbase repair rate |
| Supply tender | holds 200 stock (goods); keeps the fleet in supply anywhere, drawing 0.1 goods per ship a day, so readiness recovers; with its stock spent the fleet is out of supply |

On a hyperspace route, repair stock halves hull wear and supply stock halves readiness attrition (`07 D72`). A fleet halted in hyperspace (an intercept, an ambush, a deep group, a cancelled move) has its supply re-checked where it stops; a tender with stock still keeps it supplied (`07 D191`).

### Movement

Fleets cross hyperspace on routes planned at departure at the slowest ship's speed, after a 3-day jump windup at the system edge; within a system, moving between a body, the starbase and the edge takes `1 + |orbit_a − orbit_b|` days and no fuel (`01 §10.2`). Routes, intercepts and depth are on [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace); flying a fleet by hand is on [flight-view.md](Design-Flight-view).

### Fuel

| Rule | Value (`01 §10.2`, `data/content/hulls.json`) |
|---|---|
| Tank | `10 × Σ fuelPerJump` (`tankJumps` 10), starts full, plus tankers |
| Burn | `Σ fuelPerJump / 5` per hyperspace day, × `1 + storm` and the depth factor |
| Fuel upkeep | drawn from the tank daily at 1/30 of the per-30-day figure |
| Docked top-up | every day in a system with a friendly market holding fuel stock, from that stock |
| Short route | the route beyond the fleet's fuel at departure runs at 25% speed; a fleet that departs dry crawls the whole route (`07 D46`) |
| Empty tank | −2 readiness per day |

**Fuel tiers** (E1, `07 D73`, `07 D57`). A tank holds units by tier, and a fleet burns its preferred tier first, then down the tiers, then up (`FleetFuel` sets the preference; new fleets prefer refined).

| Tier | Commodity | Units per refined unit of range | Effects of burning it |
|---|---|---|---|
| Crude | crude fuel | 1.5 | speed −15%; readiness recovery in supply −50%; engines −10% in battle |
| Refined | fuel | 1 | baseline |
| Battery | charged cells | 4 | speed +15%; shield regeneration +20% |
| Exotic | exotic fuel | 0.5 | speed +10%; damage, engines and regeneration +10% |

The Logistics skill cuts the personal fleet's fuel by 10 / 20 / 30% (`hyper.burn`, `07 D134`).

### Supply and readiness

A fleet is **in supply** within the supply range of a friendly starbase whose market holds fuel (range by starbase size, 1–6 jumps of 2 cells each), or anywhere with a stocked supply tender (`01 §10.2`, `§10.4`, `07 D72`).

| State | Effect (`01 §10.2`) |
|---|---|
| In supply | readiness +10 per 30 days, to 100 |
| Docked at a starbase | repair at the size's rate (1.5–5% a day) paid in alloys from its market (1 alloy per 100 points); refuel from its fuel stock |
| Out of supply | no refuel, no repair, readiness −5 per 30 days |
| Deep space, above depth 2 | per day, per cell of depth: readiness −0.5, fuel +10%, hull 0.1% (`07 D45`) |

Readiness sets battle performance: in the tactical engine, rate of fire × `0.25 + 0.75 × readiness` and accuracy × `0.5 + 0.5 × readiness` (`07 D65`); in the resolver, damage × `readiness / 100` (`01 §10.3`). Each minute of a tactical battle costs every fleet 2 readiness (`07 D66`). A fleet at readiness 0 starts no fight (`05 M8`).

### Fleet capacity and upkeep

Fleet capacity = 16 + Σ docking of own starbases + tech; each ship over it raises ship upkeep by 2% (`01 §10.4`; `fleetCap` 16, `overCap` 0.02). The Coordination skill adds +2 / 4 / 6 empire fleet capacity, and its elite rank exempts the personal fleet from the over-capacity upkeep (`01 §19.2`). Upkeep per hull is credits and fuel per 30 days (`01 §10.1`).

### Fleet power

Fleet power, shown in the UI and used by the AI and pirates, is `Σ (dps × ehp)^0.5` over ships, where dps = Σ damage / cooldown and ehp = hull + armour + shields, kept per fleet incrementally (`01 §10.3`, `05 M8`).

### The AI's navy

An AI empire that makes no components buys them at the yard within the governor's reserve and builds a warship from the bill on hand, but only at war or with its navy under half its target power, so losses are replaced (`07 D116`). Its war orders send the strongest idle fleet against the enemy system nearest its capital once its power is twice what stands there (`07 D116`; see [ai-and-victory.md](Design-Ai-and-victory)).

### RTS and TB

Designs, refits, merges, splits, doctrine, detach and rejoin are commands that apply on the next tick or turn in both clocks (`05 H1`, `05 M8`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Ship identity | Named hulls carry identity; six general classes carry the rules, after Starsector | 2026-11-13 | `07 D70` |
| Size classes | Six for now, to be tweaked | 2026-11-13 | `07 D70` |
| Lineages | Three core (Forge, Line, Lumen) and three exotic (Veil, Precursor, Automata), plus species and civics flavour; the names are spec-session proposals | 2026-11-13 | `07 D70` |
| Hull count | 60+; built as 64 military, 12 fighters, 16 support (18 / 17 / 14 / 11 / 4 per class, replacing `13 §4`'s 58-sum counts) | 2026-11-13 | `07 D70`, `D71` |
| Fleet structure | A layered hierarchy with doctrine and support ships, computed rather than stored | 2026-11-13 | `07 D70`, `D72` |
| Commanders | 1 for a cruiser, 2 for a capital, 3 for a super-capital | 2026-11-13 | `07 D70` |
| Fitting layout | One 24-byte fit (16 mounts, 8 mods) for designs and the persona's fitted ships; OP replaces `01 §10.1`'s power budget | 2026-11-13 | `07 D69` |
| Fitting scope | Per-ship fitting only on the personal fleet; templates carry fits to the navy | 2026-11-12 | `07 D44`, `D135` |
| Exotic sources | Veil by tech; Precursor and Automata from cache derelicts until Boarding | 2026-11-13 | `07 D71` |
| Fuel tiers | Four tiers with a preferred fuel and fallback; fuel as a real bottleneck for fleets | 2026-11-13 | `07 D73`, `D57` |
| Calibration | The owner accepted H1's reached resolver calibration ("accept and hold"), enforced at 26 / 26 | 2026-11-13 | `07 D72` |
| Officers per ship | Built in M12b with the companions (deferred from M12a) | 2026-11-16 | `07 D135`, `D140` |

## Not yet built or still open

- **Owner review.** The lineage names and the commander counts are spec-session proposals awaiting the owner (DESIGN.md, open questions).
- **Calibration to ±5 / ±10.** The resolver agrees with the tactical engine within 26 points of win rate and 26 of loss share; the full bounds of `11 §2` wait for a structural resolver term that no milestone carries (`07 D72`). See [combat.md](Design-Combat).
- **Boarding capture.** Automata hulls are to be captured by Boarding (`13 §3`, `01 §19.2`); M12a registered the boarding keys as rows with no reader (`07 D134`), so exotic hulls come only from derelicts.
- **Species flavour rows.** `13 §3`'s example (an aquatic species giving Lumen +5% capacitor) has no built counterpart; only the three civic rows of `07 D71` exist.
- **Command capacity.** `13 §5` says command capacity counts commanders and admirals while fleet capacity counts ships; no built rule for command capacity appears in `07 D72` or `05 H1`.
- **Leaders feeding the hierarchy.** `07 D72` set coordination 0 and admiral level 3 "until M11's leaders"; no source read records whether M11 or later milestones feed real commander and admiral values into the caps, and `hulls.json` still holds the fixed values.
- **Story-point ship recovery.** "Recover a ship lost in a battle" (`01 §19.2`) waits for its system (`07 D134`).
- **Flagship skill keys.** Gunnery's and Helm's flagship rows (`ship.*.damage`, `ship.evasion`, `ship.speed`) exist with no reader (`07 D134`).

## Discrepancies noted

- **Two squadron definitions.** `13 §5`: "Squadrons form automatically from ships of one class and lineage, 5–20 strong". `07 D72` (fleet hierarchy): "form squadrons by class and lineage, up to `squad` (20) each". `07 D65` (small tactical action): "By side and hull class in order of battle, 10 ships each" (`tactics.squad` 10). `07 D67` (large action): "Squadrons of up to 20 ships of one class" (`tactics.squadLarge` 20). The sources do not say whether the hierarchy's squadrons and the battle squadrons are meant to be the same groups.
- **The tracker's Character row.** `docs/DESIGN.md` (Pillars, "Character") gives the persona's personal fleet, per-ship fitting and templates as "designed (M12a, M12b)", while `00-status.md` records M12a merged on 2026-11-15 and M12b on 2026-11-16. The tracker's header reads "Last updated 2026-11-13", so its row may predate the merges.

## Sources

- 01 §10.1 Hulls
- 01 §10.2 Movement and logistics
- 01 §10.3 Combat resolution (fleet power)
- 01 §10.4 Starbases (yards, supply range, fleet capacity)
- 01 Game design: §19.1 (the personal fleet), §19.2 (skills)
- 13 §2 Size classes, §3 Lineages, §4 Hulls, systems and fitting, §5 Fleet hierarchy
- 07 Decisions: D38, D44, D45, D46, D57, D65, D69, D70, D71, D72, D73, D116, D134, D135, D140, D191
- 05 Milestones: M8, H1, G2
- 00 Status (M12a, M12b merges)
- DESIGN.md (pillars "Ships" and "Fleets"; open questions)
- `data/content/hulls.json` (classes, lineages, systems, components, rules) and `data/content/strings/en.json` (hull names)

## Related pages

- [Overview](Overview)
- [combat.md](Design-Combat)
- [starbases.md](Design-Starbases)
- [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)
- [flight-view.md](Design-Flight-view)
- [economy-and-resources.md](Design-Economy-and-resources)
- [factions-and-characters.md](Design-Factions-and-characters)
- [megastructures.md](Design-Megastructures)
- [ground-combat.md](Design-Ground-combat)
- [ai-and-victory.md](Design-Ai-and-victory)
- [modding-and-data.md](Design-Modding-and-data)


---

# Combat

**Status:** built, as of 2026-11-23. The resolver in M8, the tactical engine's small actions in C1 and large actions in C2 (2026-11-13), the catalogue calibration in H1; the resolver's calibration bounds are held below the original target.

Space battles run on two engines. Every battle the player's empire is in runs on one deterministic tactical engine, which the player can **play** (pilot a ship or command squadrons), **watch** in real time, or **simulate** headless, all three giving the same result. Every other battle runs on a fast daily resolver of range bands and stances, calibrated against the tactical engine. Small actions put up to 100 ships a side at per-ship fidelity; large actions put thousands of ships into squadrons, with a drop-in to a small action inside them.

## How it works

### Which engine fights a battle

| Battle | Engine | Source |
|---|---|---|
| The player's empire is in an engaging pair there | the tactical engine; the battle waits (`Battles::tac`) for the player to play, watch, simulate or concede | `07 D48`, `D66` |
| The player joins a resolver battle | becomes the player's battle | `07 D66` |
| Every other battle, and any battle the player concedes to "Auto-resolve" | the daily resolver of `01 §10.3` | `07 D48`, `D66` |

In a tactical battle, side 0 is the player's empire and side 1 is every party hostile to it; parties hostile to neither stay out (`07 D66`).

### When combat starts

Combat starts when two hostile fleets share a system, or meet in hyperspace at the cell where their routes share a day (an intercept), or a route meets a deep encounter, and at least one side is willing to engage (`01 §10.3`):

| Stance | Engages |
|---|---|
| Aggressive | always |
| Patrol | pirates and empires its owner is at war with |
| Raid (pirates only) | civilians or weaker fleets |
| Blockade | anything entering |
| Evasive, passive | — |

A battle in hyperspace is fought at the cell's place, `SystemId(1024 + cell)`, so the resolver, the fleets-in-a-system index and the battle reports work there unchanged (`02`, `07 D46`). A fleet at readiness 0 starts no fight (`05 M8`). Combat runs daily only over the `contested` set, so a tick with no battle spends nothing in combat (`02`, `05 M8`).

### The resolver (battles the player is not in)

One combat tick per day per contested system or cell (`01 §10.3`):

- **Bands.** Bands are ordered short < medium < long; a weapon fires when the engagement band ≤ its band. The day's band is the closest any engaging ship's computer prefers, approached one band a day from long.
- **Targeting.** Each ship with a weapon off cooldown and a target in band picks a weighted random enemy; the computer type weights smallest, largest or nearest in band.
- **Hit chance** = `clamp(accuracy − evasion × (1 − tracking), 0.05, 0.95)`.
- **Damage** = `damage × (readiness / 100) × (1 ± 0.1 rng)`, to shields first (× `1 − shieldPen`), then armour (× `1 − armourPen`), the remainder to hull.
- **Destruction.** A ship at 0 hull is destroyed; 30% of its alloys stay as debris recoverable by a construction ship (salvage of debris is not built, `05 M8`).
- **Disengagement.** A side whose fleet hull share falls under its stance's threshold (aggressive 20%, patrol 50%, evasive and raid 80%, blockade 30%) takes the **emergency jump**: a straight 3-day jump to the nearest own system by map distance, with no route, so nothing meets it in flight; each ship has a 10% chance to fail and be destroyed (`07 D46`). A fleet's doctrine retreat share raises the side's threshold (`07 D72`).
- **Shields** regenerate 10% a day out of combat.

Each ship and the starbase shoot from a stream derived from (seed, unit, tick) against the start-of-tick state, hits applied in (target, shooter, slot) order (`02`). The resolver fires every one of a unit's 16 slots (`07 D72`).

**Calibration coefficients** (`hulls.json` `rules.resolver`, `07 D68`): damage 1.454, evasion 1.148, disengage 1.35 (the scaled threshold capped at 0.95), size 0.002; damage dealt by class 0.75, 1.409, 1.0, 1.274; damage taken by class 1.0, 1.73, 0.902, 1.0; the jump's loss chance 0.387; focus 0.457; and a **closing term** 4.0: a ship whose weapons cannot yet reach the day's band takes damage × `(100 / speed)^4`, so small fast hulls spend fewer exchanges under fire while they close. H1 added per-class dealt and taken for the six classes and per-lineage dealt and taken (`07 D72`); the per-class values in `hulls.json` (`tactics.classes`) are listed on [ships-and-fleets.md](Design-Ships-and-fleets).

**Repair.** `01 §10.3`'s closing line gives hull and armour repair at shipyards 5% a day or 2% at a friendly starbase. `01 §10.4`'s size table (`07 D39`, later) sets repair per starbase size, 1.5% to 5% a day, paid in alloys; that table is the current rule. Repair tenders mend 1% a day outside docks (`07 D72`).

### The tactical engine

The engine is its own crate, `tactics`, `#![forbid(unsafe_code)]`, under the sim's determinism rules (fixed step, `libm` maths, derived RNG streams, id tiebreaks), and depends on nothing of the game (`11 §2`, `07 D65`). It is a pure function of `(battle setup, seed, input log)`. Its numbers are content in `hulls.json` (`tactics`), checked by `datacheck`.

**Core numbers** (`07 D65`, `D67`, `data/content/hulls.json` `tactics`):

| Rule | Value |
|---|---|
| Step | 30 steps a second of battle time |
| Weapon reload | its cooldown in days × 6 s |
| Speed | strategic speed × 1.2 arena units a second, reached in 3 s |
| Battle length | a few ships to 100 a side last 1–4 minutes |
| Beams (energy) | roll `accuracy − evasion × (1 − tracking)` per shot |
| Kinetic shots | fly with a lead and an aim error of `6° × (1 − accuracy)`; roll evasion on contact |
| Missiles | home at 120°/s; roll the beam formula on contact |
| Point defence | stops `pd × 2` missiles a second within 300 units |
| Bodies | the system's star and planets (moons left out) block shots and ships; a planet's battle radius is `30 + 3 × size` (`07 D67`) |
| Arena | a disc of radius `1,800 + 110 × √ships` |
| Targeting | a 250-unit grid rebuilt each step; each ship retargets every half second, staggered by index |

**Shields and capacitor** (simplified flux, `11 §3`, `07 D65`). A ship's capacitor holds 3 × its design's shields, drawn 4% a second while shields are up and recharged 8% a second while down; emptied, it overloads for 3 s. Ships raise shields when an enemy is within long range or they took damage in the last second. There is no per-weapon flux for the player to manage. Unspent OP adds 6 shields per point (`07 D69`).

**Armour.** Six 60° sectors per ship, each holding 0.4 × the design's armour; the resolver's armour formula applies per sector, and the result folds the sectors back to the design's scale (`07 D65`).

**Readiness and manning** (`07 D65`). Rate of fire × `0.25 + 0.75 × readiness`, accuracy × `0.5 + 0.5 × readiness`; there is no peak-performance timer (`11 §3`). A hull short of its commanders (cruiser 1, capital 2, super-capital 3) has its rate of fire, thrust and turn rate × 0.75.

**Officers and personalities** (`07 D65`). Every ship's officer has a personality (aggressive, steady, cautious) drawn from the battle seed and the ship. Each fleet carries 3 commanders, given to its largest hulls first. Aggressive ships close to 55% of their longest range, steady to 80%, cautious to 95% and withdraw below 30% hull. Doctrine overrides the range: close 45%, long 95% (`07 D72`). Unarmed hulls leave by the nearest edge.

**Ship systems.** Each hull's system (burn, damper, overdrive, phase, flare, uplink, repair, capacitor) runs in the engine; the AI triggers each kind on its own condition, a pilot with F (`07 D69`).

**Fuel tier.** The tier a fleet burns changes its battle rows: crude engines −10%; cells shield regeneration +20%; exotic damage, engines and regeneration +10% (`01 §10.2`, `07 D57`).

### Small fleet actions (C1)

- **Size.** Up to 100 ships a side on the field, deployed in up to three waves by the fleet's order of battle; the next wave arrives when a side is down to 50; the rest stay in reserve and keep their state. There are no deployment points (`11 §3`, `07 D65`).
- **Control is a toggle**, at any step, between piloting and command (`11 §3`, `07 D48`):
  - **Piloting:** fly the flagship (move, aim, fire groups, shields). Only frigates and destroyers can be piloted.
  - **Command:** select and order squadrons from a radial menu or one-key orders, with pause. Orders: attack (an enemy squadron), escort, hold (a point), flank (left or right of the enemy's centre) and retreat; `Auto` follows the officer's personality (`07 D65`). Squadrons form automatically by side and hull class, 10 ships each.
  - **Larger flagships are commanded, not piloted.** A cruiser, capital or super-capital flagship takes orders like any other ship, with the Command skills applying (`11 §3`, `07 D66`).
- **The flagship** is the largest hull of the player's lead fleet (its first in order of battle), "until M12's persona brings its own" (`07 D66`).
- **The end** (`07 D65`, `D67`). A side with nothing on the field and nothing to come is out. At 10 minutes, or after 60 s without a hit, the side with the smaller hull share on the field withdraws (the enemy on a tie). A side whose ships on the field average under 20% of their hull withdraws its free squadrons, in small and large actions alike.
- **Smoothness** (`11 §3`): one-key orders, no command-point cost, instant retreat by the nearest edge, camera follow and free camera, every ship's hull, armour, capacitor and order readable at a glance.

### Large fleet actions (C2)

- **When.** A side with more than 300 ships (`largeAt`) makes the battle a large action (`07 D67`); a target of 2,000 ships (100–200 squadrons) at 60 fps (`11 §4`).
- **Squadrons** of up to 20 ships of one class, by side and class in order of battle. A squadron moves and turns as one body at its slowest ship's speed and least agility; its ships stand on its formation's slots, spaced by three of their radii plus 30 units (`07 D67`).
- **Formations** (`07 D67`):

| Formation | Damage dealt | Damage taken | Speed |
|---|---|---|---|
| Line | 1.1 | 1.0 | 1.0 |
| Wedge | 1.0 | 1.1 | 1.15 |
| Screen | 0.9 | 0.85 | 1.0 |
| Englobe | 1.05 | 1.05 | 0.8 |

- **Aggregate fire.** Every ship's due weapon fires at a ship of its squadron's target (rotating through the target's ships) within range and within the squadron's heading arc. Every family uses the beam formula, with no projectiles in flight; point defence subtracts from a missile's chance, as in the resolver. Hull, armour sectors and capacitor stay per ship, so losses are whole ships (`07 D67`).
- **Command,** Total War-style: select squadrons (click, Shift-click), right-click an enemy to attack it or the ground to move there (a hold), Z / X / C / V for the formations, R retreat, Q free, Space pause (`07 D67`). The persona's flagship is present and its skills apply (`11 §4`).
- **The drop-in.** With the tier-3 computing tech **Battlenet Command** (after Predictive Analytics), D at the pointer takes the squadrons within 1,500 units of it, nearest first, while each side stays within 100 ships, and opens a small action on them on its own field, with terrain and every ship's state. The large battle steps it once per step; its squadrons keep their places, frozen and untargetable. Piloting, flight and orders go to it while it runs. When it ends, or on B, its ships' state folds back. All of it is in the one input log (`07 D67`). The elite Tactics skill is the second route to the drop-in (`11 §4`).

### Play, watch, simulate

- **Play.** The player's inputs (flight, orders, time dilation) join the input log with their step, so a played battle replays exactly (`11 §2`, `07 D65`).
- **Watch.** The same run drawn at the frame rate with interpolation between steps; time dilation from 0.25× to 4× (`11 §2`).
- **Simulate.** The same run headless at full speed. It runs at once on the UI thread (100 v 100 takes about 0.1 s), so no progress bar is shown (`07 D66`). The hash in `BattleResult` leaves out time dilation, so watching and simulating give the identical result (`07 D65`).
- **Conceding.** "Auto-resolve" hands the battle to the resolver from the state it opened with; a battle half played and then conceded loses its tactical progress (`07 D66`).

### Hand-over to the strategic sim

- **The clock.** No tick runs while the player's battle waits, in RTS and in a TB turn alike; the result applies on the next tick, also inside a TB turn (`07 D66`).
- **`BattleResult`** is the single command back into the sim: ships lost, hull, armour and shields left, readiness, the battle's length in steps, and the input-log hash. A saved game stores the result, not the battle (`11 §2`, `07 D66`).
- **Write-back** (`07 D66`). A destroyed ship leaves the resolver's debris. Survivors keep hull, armour and shields. Every fleet loses 2 readiness per minute fought. A fleet whose surviving ships all left the field takes the emergency jump without its loss roll. The system is then checked again.
- **Saving mid-battle** keeps the world as it stood when the battle opened and the input log so far; on load the battle opens where its log ends (`07 D66`).

### Calibration of the resolver

The harness (`tools calibrate`) runs matched setups through both engines (`07 D68`, `D72`):

- 20 matchups (C2's twelve: corvette and destroyer mirrors, swarms against capitals, mixed fleets, the player's fleet against the first AI empire's; plus eight across the catalogue: every lineage, carriers, super-capitals), 500 seeds each, in open space for the tactical runs.
- The win share is the side that holds the field; each side's loss share is ships destroyed or lost in the jump over ships at the start, counting strategic ships only (fighters are not resolver units).
- **Bounds.** `11 §2` set ±5 points of win rate and ±10% of loss share. C2 reached 7.6 / 24.9 on twelve matchups (`07 D68`). H1, with the refit catalogue, reached 25.8 / 25.2 on twenty, where the search plateaued; the worst win-rate gap is the 20-corvette swarm against 4 destroyers and 2 cruisers (27% wins on the engine, 0% in the resolver). The owner's call (2026-11-13) was **accept and hold**: `tools calibrate` and `tools bench` enforce 26 / 26 (`CAL_BOUNDS`) (`07 D72`).

### Starbases in battle

A starbase fights as a stationary ship with its size's hull and its batteries (`01 §10.4`; the `starbase` class in `tactics.classes`). A starbase at hull 0 in a battle with an empire at war with its owner passes to that empire with its system and starbase colony, at 10% hull (`07 D113`), but only once no armed ship hostile to that empire still fights in the battle; until then it stays in the fight at one hull point (`07 D149`). Pirates destroy a base at 0 instead (`07 D113`). The owner's station redesign (`07 D284`) is on [starbases.md](Design-Starbases).

### Battles and war

A battle is won by the side whose ships remain when it ends. War score gives +2 per battle won (up to 30) and 40 × enemy ships destroyed over its ships at the start (up to 40); exhaustion adds 20 × the side's ships lost over its ships at the start at each battle's end (`07 D112`; see [diplomacy-and-war.md](Design-Diplomacy-and-war)). When a battle destroys a fleet whose admiral is a notable and the persona's fleet fought on the winning side, one roll (50%) captures the admiral (`DESIGN.md`, autonomous calls 2026-11-21, `07 D219`).

### The persona in battle

| Effect | State | Source |
|---|---|---|
| Tactics: squad damage +5 / 10 / 15% | built, applied as readiness | `01 §19.2`, `07 D134` |
| Officers on the personal fleet: +2% fleet damage per officer skill | built | `07 D140` |
| The persona's Piloting and Command rows fold into each ship's `damage_mul` and `speed_mul` | hook from C1 (1 until M12) | `07 D65` |
| Gunnery, Helm flagship rows | rows with no reader | `07 D134` |

### Battle reports and playback

Each battle row keeps per-tick loss rings, and the battle report records per-tick losses for playback (`01 §10.3`, `05 M8`). Battle notifications, the battle reports and planet defence are in the UI from M8 (`00-status.md`, pass 21). In TB, the turn report's playback scrubber shows the turn's battles from the battle events: a battle runs from its open to its end, or to the turn's last tick while still open, and the scrubber rings each system fighting at the frame's tick (`07 D93`). The playback track is the driver's and is neither saved nor hashed (`07 D93`).

A committed battle fixture (Tiny, seed 7) writes one line per combat tick and is compared with `tests/det/battle-log.txt` on every gate (`05 M8`).

### Budgets

| Situation | Line (`11 §5`) | Measured |
|---|---|---|
| Small action step, 100 v 100, 30 Hz | ≤ 0.5 ms native, ≤ 1.5 ms web | 54.2 µs p50 / 261.8 µs p99 (H1, calibration 1.25) |
| Small action simulated, a 5-minute battle | ≤ 5 s native, ≤ 15 s web | 0.23 s (H1) |
| Large action step, 2,000 ships | ≤ 1.5 ms native, ≤ 4 ms web | 135.1 µs / 377.4 µs (H1) |
| Battle frame GPU, 2,000 ships, 1080p, GTX 1060 | ≤ 4 ms native | 0.071 ms p50 (C2) |
| Resolver, 60 ships | ≤ 60 µs | 39.2 µs (H1) |

Measured values from `05 C2` and `05 H1`. Combat is not a candidate for `sim::par` threading (`07 D262`).

### Presentation

Ships draw as G2's lit Blender sprites, with the class arrowhead below 18 px on screen (`05 G2`). The owner's presentation call keeps high-detail 2D in battles beside a 3D map (`07 D231`; see [presentation-and-ui.md](Design-Presentation-and-ui)).

### RTS and TB

The strategic clock pauses while the player's battle waits, in both clocks; the battle's result applies on the next tick (`07 D66`). Resolver battles are a daily system in both clocks; TB plays battles back from their rows (`05 M8`, `07 D93`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Combat model | Replaced D5-A's resolver-only model: small actions positional on the tactical engine; large actions use A's range bands and stances inside C2's squadron simulation | 2026-11-13 | `07` D5 (owner's answers at the top) |
| Fully simulated | A. Hybrid: the player's battles on one deterministic tactical run; others on the calibrated resolver | 2026-11-13 | `07 D48` |
| Control | A toggle between piloting and command; larger flagships need commanders and cannot be piloted | 2026-11-13 | `07 D48` |
| Large actions | A squadron-level simulation with a drop-in to a small action through late-game perks | 2026-11-13 | `07 D48` |
| Shields | A single capacitor with overload, not per-weapon flux | 2026-11-13 | `07 D48`, `D65` |
| Engine numbers | 30 Hz, capacitor 3× shields, six armour sectors, three waves of 100, 3 commanders per fleet | 2026-11-13 | `07 D65` |
| Conceding | Auto-resolve restarts from the opening state; simulate runs at once, no progress bar | 2026-11-13 | `07 D66` |
| Large threshold and drop-in | Large past 300 a side; drop-in by Battlenet Command, 1,500 units, ≤ 100 a side | 2026-11-13 | `07 D67` |
| Withdrawal | A side averaging under 20% hull withdraws its free squadrons | 2026-11-13 | `07 D67` |
| Calibration | Replaced ±5 / ±10 (`11 §2`) and C2's 8 / 25 (`07 D68`) with H1's reached 26 / 26, the owner's "accept and hold" | 2026-11-13 | `07 D72` |
| Emergency jump | A straight 3-day jump to the nearest own system, 10% loss per ship | 2026-11-12 | `07 D46` |
| Starbase capture | At hull 0 to an empire at war with the owner, only after the last armed defender falls | 2026-11-14, 2026-11-16 | `07 D113`, `D149` |
| Duels | None (the owner chose C over a flagship duel in the tactical engine) | 2026-11-21 | `DESIGN.md`, `07 D219` |

## Not yet built or still open

- **The resolver's structural term.** The full ±5 / ±10 calibration waits for a structural resolver term (battle length, or a side-level withdrawal mirroring the engine's), which no milestone carries (`07 D72`; DESIGN.md, open questions).
- **Cut terrain.** Storm terrain and sensor shadows in battle were cut in C1, as `05 C1` allows (`07 D65`).
- **The elite Tactics drop-in.** `07 D67` says "elite Tactics joins with M12"; the M12a record (`07 D134`) lists Tactics only as squad damage, and no source read records the drop-in through elite Tactics as built.
- **The persona's flagship.** `07 D66` makes the largest hull of the lead fleet the flagship "until M12's persona brings its own"; no source read records the tactical flagship changing to the persona's ship.
- **Captures and officer XP from battles.** `07 D66` left ship captures to M12's Boarding and officer XP to M11's leaders; Boarding's keys have no reader (`07 D134`), and no source read records officer XP from tactical battles.
- **Debris salvage.** Debris is recorded per system; its salvage by a construction ship is not built (`05 M8`).
- **Stations.** The owner's station of sections (`07 D284`) is designed, its milestone not yet placed; when built, the tactical engine and the resolver will fight it as a body of separately targetable sections and the calibration harness will add station matchups.

## Discrepancies noted

- **Range bands in large actions.** The owner's answer to D5 (`07`, top, 2026-11-13): "Large fleet actions use A's range bands and stances inside C2's squadron simulation, which is kept." `07 D67` (C2 as built, 2026-11-13): "Every ship's due weapon fires at a ship of its squadron's target ... within range and within the squadron's heading arc. Every family uses the beam formula". `07 D67` and `05 C2` do not mention range bands or stances in the large action, so whether the built large action follows the D5 answer is not stated.
- **The TB battle prompt.** `11 §2`: in TB a battle "resolves inside the turn with a prompt". `07 D66`: "The driver runs no tick while the battle waits, in RTS and in a TB turn alike; the result applies on the next tick, also inside a TB turn." `07 D94` (cited in the tracker) has auto end turn wait on a waiting battle. The two may describe the same behaviour, but `11 §2` was not updated to the as-built wording.

## Sources

- 01 §10.3 Combat resolution
- 01 §10.4 Starbases
- 11 §1 The owner's calls, §2 One core, three modes, §3 Small actions, §4 Large actions, §5 Budgets
- 02 Architecture (daily system 5, `combat`)
- 07 Decisions: D5 (the owner's answers of 2026-11-13 at the top), D46, D48, D57, D65, D66, D67, D68, D69, D72, D93, D112, D113, D134, D140, D149, D219, D231, D262, D284
- 05 Milestones: M8, C1, C2, H1, G2
- 00 Status (pass 21)
- DESIGN.md (pillar "Combat"; open questions; autonomous calls of 2026-11-21)
- `data/content/hulls.json` (`tactics`, `rules.resolver`)

## Related pages

- [Overview](Overview)
- [ships-and-fleets.md](Design-Ships-and-fleets)
- [starbases.md](Design-Starbases)
- [ground-combat.md](Design-Ground-combat)
- [diplomacy-and-war.md](Design-Diplomacy-and-war)
- [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)
- [time-and-modes.md](Design-Time-and-modes)
- [factions-and-characters.md](Design-Factions-and-characters)
- [presentation-and-ui.md](Design-Presentation-and-ui)
- [architecture-and-performance.md](Design-Architecture-and-performance)
- [ai-and-victory.md](Design-Ai-and-victory)


---

# Megastructures

**Status:** built, as of 2026-11-23. The shared model with 8 kilo- and 8 megastructures and ruins in S1, the 5 gigastructures and giga ruins in S2 (both 2026-11-14), Gateway trade links in M19a and the AI's orders for great works in T2 (both 2026-11-23); the AI megastructure advisor (M19c) and a new roster of great works are still to come.

Megastructures are large staged constructions in three tiers, kilo, mega and giga, drawn from the Stellaris mod Gigastructural Engineering. Twenty-one of them are data in `megastructures.json`: they place by rules on stars, bodies or starbases, draw each stage's bill from their own system's markets like any construction, and act through modifier rows and a few code hooks (corridor anchors, yards, mass-driver links, habitats, the Gateway, the Nicoll-Dyson Lance). Ruined structures lie mostly in deep space and the abyss, guarded by automata, and can be restored at a quarter of their cost.

## How it works

### The shared model

One data-driven model serves every tier (`13 §6`, `07 D78`):

| Part | Rule |
|---|---|
| Tier | kilo, mega or giga; tera is left to mods (`13 §6`) |
| Placement | on a star, a body or the system's starbase: body kinds, classes and size; the body's colony (any, none or own); star classes; the cell's depth; the starbase's least size; per-system and per-empire limits; one structure a body (`07 D78`) |
| Prerequisites | `requires` another of the builder's structures, active in the same system; `consumes` replaces it on completion (`07 D81`) |
| Stages | each stage and upgrade has a bill, days, upkeep (credits and commodities) and modifier rows (`07 D78`) |
| Failure and decay | optional yearly roll (ruin or decay) (`07 D78`) |
| Hooks | code opted into by data (below) |
| Ruins | a ruin weight per structure (`07 D78`) |
| Settings switch | one per structure in the game settings; at most 64 structures, so the switch is one `u64` (`07 D78`) |

The placement rule lives in one function, `view::explore::structure_ok`, so the sim's validation and the build menu's preview agree (`07 D78`). A refused order gives its reason (`Ineligible`, `Occupied`, `Cap`, `Requires`, `Unavailable`) (`05 S1`, `05 S2`).

**Stages are constructions** (`Work::Structure`), drawn at the system's own starbase market under the local-sourcing rule, so a stage stalls like any build; the next stage is ordered when one finishes; an upgrade is the same construction (`07 D78`). See [starbases.md](Design-Starbases) and [trade-and-logistics.md](Design-Trade-and-logistics).

**Effects are modifier rows**, added when each stage finishes under source `src::STRUCTURE`, and shown in every stack tooltip under the structure's name (`07 D78`, `05 S1`). Scopes (`07 D78`, `D81`):
- empire (the default);
- colony: the structure's habitat, the own colony on its body, or its system's own starbase colony;
- system: a row on every own colony of the system at the stage's completion, removed with the structure; colonies founded there later do not get it.

Nothing runs per tick: stages run on construction timers, effects are rows, failure is a yearly heap timer (`13 §6`). A bench row with 200 finished structures measures their bookkeeping (`05 S1`, `07 D81`).

**Presence.** In the Empire framework, ordering a megastructure in a system with an ungoverned colony needs the personal fleet stopped in that system; a governed colony's governor orders it remotely (`07 D136`, `01 §19.1`).

### Hooks

| Hook | What it does (`07 D78`, `D81`) |
|---|---|
| `gateway` | crossing days; the Gateway keeps M8c's `gate` column, so hyperspace is unchanged |
| `anchor` (radius, depth) | caps depth at the Chebyshev distance less the radius; drift skips cells within the radius; holds every corridor in reach, not only its owner's |
| `yard` (class, slots, repair) | on the system's starbase |
| `driver` (fraction) | a new Mass Driver links to the empire's oldest unlinked driver in another system; routes between the two systems take the fraction of their days |
| `habitat` | founds a habitat colony around the body; on a star-anchored structure, around the system's star (the Ringworld) |
| `batteries` | adds batteries to the system's starbase |
| `lance` | the Nicoll-Dyson Lance's strike (below) |

### Kilostructures (8)

| Structure | Placement | Bill | Days | Upkeep /30 d | Effect |
|---|---|---|---|---|---|
| Asteroid Manufactory | an asteroid belt, no colony | 400 alloys, 40 components | 240 | 3 cr | +5% artisan output |
| Asteroid Battery | an asteroid belt, no colony | 500 alloys, 60 components | 240 | 2 cr, 0.5 components | +0.05 piracy security and 4 batteries on the system's starbase |
| Orbital Ecosystem | a barren, molten, toxic or frozen body, no colony | 600 alloys, 60 components, 200 food | 300 | 3 cr | founds a habitat-like colony around the body |
| Atmospheric Processor | a planet with an own colony | 500 alloys, 50 components, 200 gas | 360 | 3 cr, 1 gas | +3 housing and +10% growth on the colony below |
| Gas Giant Stabiliser | a gas giant, no colony | 700 alloys, 80 components, 100 gas | 360 | 4 cr | a colonisable gas giant (`13 §6`): its habitat hook founds a habitat around the body |
| Stellar Particle Accelerator | a star | 800 alloys, 120 components, 100 rare ore | 360 | 5 cr, 1 rare ore | +10% physics; 2% yearly failure, which ruins it |
| Orbital Arcology | a planet of size 8–30, no colony | 900 alloys, 100 components, 200 goods | 420 | 5 cr | +6 housing and +2 amenities on its habitat |
| Beacon Spire | a starbase of size ≥ 2 | 300 alloys, 30 components | 180 | 2 cr | a small hyperspace anchor: radius 3, holding corridors, no depth change |

### Megastructures (8)

| Structure | Placement | Stages × bill | Days a stage | Upkeep /30 d | Effect |
|---|---|---|---|---|---|
| Gateway | a starbase of size ≥ 6, charted space (depth 0) | 1 × 5,000 alloys, 800 components | 1,080 | 30 cr, 4 components | fleet crossings between Gateways (`01 §2.1`); trade shortcut (M19a, below) |
| Hyperspace Anchor | a starbase of size ≥ 4 | 1 × 2,500 alloys, 300 components, 200 rare ore | 720 | 15 cr, 2 components | radius 6: holds every corridor and lowers depth |
| Shipyard Ring | a starbase of size ≥ 4 | 1 × 3,000 alloys, 400 components, 100 machine tools | 720 | 20 cr, 2 components | builds super-capitals; +2 yard slots; repair × 2 |
| Mass Driver | a starbase of size ≥ 3 | 1 × 1,500 alloys, 200 components | 540 | 10 cr, 1 fuel | a linked pair's routes take 0.25 of their days; +5% freight |
| Dyson Swarm | an O/B, A, F, G or K star; 2 per empire | 3 × 1,500 alloys, 150 components | 360 | 5 cr a stage | per stage +5% technician and +3% metallurgist output |
| Strip-mine Engine | a barren, molten, toxic or frozen body of size 8–30, no colony | 2 × 1,200 alloys, 150 components, 50 machine tools | 360 | 6 cr (and 0.5 machine tools in the first stage) | per stage +8% miner output; 5% yearly decay |
| Rare-ore Megabore | a molten body, no colony | 1 × 1,500 alloys, 200 components, 50 machine tools | 540 | 8 cr, 0.5 machine tools | +15% rare-ore (crystal) output |
| Science Nexus | a starbase of size ≥ 4; 1 per empire | 2 × 2,000 alloys, 300 components, 200 rare ore | 540 | 15 cr a stage | +5% to each research field and +10% skill gain; a quantum-core upgrade |

### Gigastructures (5)

Each gigastructure is one per empire, with long staged builds and large upkeep (`13 §6`, `07 D82`).

| Structure | Placement | Stages × bill | Days a stage | Upkeep /30 d | Effect (`07 D82`) |
|---|---|---|---|---|---|
| Matrioshka Brain | a star | 3 × 4,000 alloys, 600 components, 400 rare ore | 720 | 30 cr (and 2 rare ore in the first stage) | +10% to each field per stage, then −5% tech cost and +5% build speed |
| Dyson Sphere | an O/B, A, F, G or K star; requires and consumes the Dyson Swarm | 2 × 5,000 alloys, 500 components, 200 machine tools | 720 | 25 cr a stage | +20% technician and +15% metallurgist output per stage; the second stage dims the system: −15% growth on the system's own colonies |
| Ringworld | an F, G or K star | 3 sections × 6,000 alloys, 600 components, 500 goods | 900 | 20 cr a section | a habitat around the star: +15 housing per section, +2 building slots in the first two, +5 amenities in the last |
| Star Lifter | a star | 2 × 3,500 alloys, 500 components, 200 machine tools | 720 | 20 cr (and 1 machine tool in the first stage) | +20% gas per stage, +5% physics in the second |
| Nicoll-Dyson Lance | a star; requires a Dyson Sphere in the system (not consumed) | 2 × 6,000 alloys, 800 components, 400 rare ore | 900 | 40 cr, 2 components a stage | a strategic strike across hyperspace (below) |

Placement, bills, days and upkeep are from `data/content/megastructures.json`; the commodity names are E1's (parts became components, crystal rare ore, machinery machine tools; `07 D57`). `07 D79` gives the S1 ranges as kilostructures 300–900 alloys, 30–120 components, 180–420 days, 2–5 credits upkeep, and megastructures 1,200–3,000 alloys and 150–400 components a stage, 360–720 days a stage, 6–20 credits upkeep. Effects are first guesses for balance passes (`07 D79`, `D82`).

### The Nicoll-Dyson Lance

Rules set provisionally by the spec session (`07 D80`), built in S2 (`07 D82`), with the cost moved to energy by E1 (`07 D57`) and the war by M10 (`07 D115`):

| Rule | Value |
|---|---|
| Prerequisite | a complete Dyson Sphere in the same system; one Lance per empire |
| Charge | 180 days, starting at completion and restarting on each strike |
| Range | 30 cells: the grid distance between the two systems' cells |
| Strike cost | 500 energy from the Lance system's own starbase market |
| Effect on a fleet | every ship destroyed (the fleet is removed) |
| Effect on a starbase | 80% of its hull lost |
| Colonies | not targeted |
| Opinion | every empire that sees the target or the Lance's system, and the target's owner, take −100 opinion towards the firer for 120 months (a timed `diplomacy.opinion` row, source `LANCE`) |
| War | a strike on an empire's fleet or starbase while not at war with it declares war on it (to humiliate); a strike on pirates declares nothing |

The Lance has no ruins (`07 D81`).

### Failure and decay

A structure with a `failure` arms a yearly timer at a year plus its row mod 360; the roll is a pure function of `(seed, row, year)` (`07 D78`). Failure ruins it: its rows, upkeep and hooks go, and the row stays, unowned. Decay costs the top stage; a one-stage structure falls to ruin (`07 D78`). The built cases are the Stellar Particle Accelerator (2% a year, ruin) and the Strip-mine Engine (5% a year, decay). A fixture gave 202 failures in 10,000 structure-years at 2% (`05 S1`).

### Ruins

Galaxy generation scatters ruins, seed-hashed so no game stream moves (`07 D78`, `D81`; `megastructures.json` `ruins`):

| Rule | Value |
|---|---|
| Kilo and mega ruins | `perSystem` 0.06 × systems; `deepShare` 0.8 in cells of depth `deepMin` 6 or more (for kinds whose depth range reaches that), the rest on an eligible star or body of a system with no home |
| Giga ruins | `gigaPerSystem` 0.006 × systems; `gigaDeepShare` 0.8 in cells of depth `abyssMin` 10 or more |
| Deep ruin | a deep site of its own kind: the first fleet in meets automata like a cache, the next claims it |
| System ruin | known once its system is surveyed; needs no claim |
| Restoration | a construction ship (`RestoreRuin`) loads `restore` 0.25 of every stage's bill at its system's own starbase market and carries it; the restoration takes the same share of the days, and the structure comes back complete |

Medium seeds 1–3 gave 23, 24 and 23 ruins, of which 19, 21 and 21 lie at depth 6 or more, and 2, 2 and 1 giga ruins, all at depth 10 or more (`05 S1`, `05 S2`). Switching a structure off removes it from building and from generated ruins (`05 S1`). Deep space and its depth bands are on [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace).

### Gateway trade links (M19a)

An empire's hub pair takes the Gateway jump when both hubs' systems hold a Gateway it may use (its own, or an open-borders partner's) and the jump is quicker than the pair's route: the jump's days, no piracy, all cargo landed, closed by a blockade at either gate system. It is derived at route time, with no state, no save bump and no field rebuild; pact routes and the Galactic Exchange's legs still fly (`07 D283`, owner's call 1A; merged 2026-11-23 per `00-status.md`). The Mass Driver is the other structure trade reads (`07 D78`). See [trade-and-logistics.md](Design-Trade-and-logistics).

### Technology and the Kardashev gates

Every structure names its tech in `megastructures.json` (`tech`). Under the tech-tree rework, the Dyson Swarm is part of the gate to Kardashev Type I and the Dyson Sphere the gate to Type II; the Sphere's tech moves from tier 3 to tier 5, the Swarm's stays at tier 3; the works of tiers 6–7 are the path to Type III. The Lance's tech became `collapse_engineering` (Degenerative Collapse) (`07 D278`). A Kardashev type needs its structure for every empire (`07 D278`, call 8). See [progression.md](Design-Progression).

### The AI and megastructures

S1 left the AI building no structures (`07 D79`). T2 (merged 2026-11-23) has the AI order great works: once a year, with none of its own under construction and its month's margin not negative, an AI empire orders the first work it knows whose first stage its markets hold `worksStock` (2) times over, with the treasury's credits and influence for it; the next Kardashev type's structure before the others; at the first of its eight largest starbases whose system takes it (`ai::works`, `07 D278` call 8). Which works it favours beyond the gate's structure waits for T3 (`DESIGN.md` change log).

### Ownership

Structures stay with their builder; S1 built no transfer, on the premise that no MVP path moved a starbase to a new owner (`07 D78`).

### RTS and TB

Orders, stages and restorations are constructions and commands that resolve on ticks; failure and decay run on yearly timers; both clocks behave alike (`07 D78`, `13 §6`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Set and reference | The themed set, looking hard at Gigastructural Engineering | 2026-11-13 | `07 D70` |
| Scale | Up to gigastructures in MVP; replaces `01 §9.8`'s 1.0 list (Orbital Ring, Dyson Swarm, Science Nexus with +25% skill gain, Habitat) | 2026-11-13 | `07 D70` |
| Ruins | Deep-space rewards, sometimes in normal space as in Stellaris | 2026-11-13 | `07 D70` |
| The model | One data model; effects as modifier rows; hooks opted into by data; stages as constructions | 2026-11-14 | `07 D78` |
| Anchor reach | A Hyperspace Anchor holds every corridor in its radius, not only its owner's | 2026-11-14 | `07 D78` |
| Restoration | A construction ship carries 25% of the stages' bill from its system's starbase market; deep ruins claimed by beating automata | 2026-11-14 | `07 D78` |
| Driver link | A new Mass Driver auto-links to the empire's oldest unlinked driver; the pair's routes take a quarter of the days | 2026-11-14 | `07 D78`, `D79` |
| Cuts | The Strip-mine Engine's shrinking (it decays instead) and star ageing; the Ringworld kept its three sections | 2026-11-14 | `07 D79`, `D82` |
| Sphere and Swarm | The Sphere needs and replaces the Swarm; its dimming is a growth malus on the system's own colonies at completion | 2026-11-14 | `07 D81` |
| The Lance | Provisional rules for the owner to confirm or rebase; fires on energy since E1 (replacing S2's 500 fuel) | 2026-11-14 | `07 D80`, `D82`, `D57` |
| Lance war | A strike outside war declares war (to humiliate) | 2026-11-14 | `07 D115` |
| Megabore name | The Crystal Megabore became the Rare-ore Megabore (id `crystal_megabore` kept) | 2026-11-14 | `07 D57` |
| Kardashev gates | Swarm in the Type I gate, Sphere in the Type II gate; a new roster of great works scoped separately | 2026-11-21 | `07 D278` |
| AI and great works | The AI builds great works as part of development, triggered by having the resources | 2026-11-23 | `07 D278` call 8 |
| Gateway trade | A route shortcut, as the Mass Driver's | 2026-11-23 | `07 D283` call 1 |
| Capstones | New capstone structures, designed with the ascension paths later | 2026-11-23 | `07 D283` call 3 |
| AI advisor | M19c: one structure at a time | 2026-11-23 | `07 D283` call 5 |

## Not yet built or still open

- **The Lance's rules** are provisional; the owner is to confirm or rebase them (DESIGN.md, open questions).
- **The great works roster.** A separate session will design a wider roster of kilo-, mega- and gigastructures, not on Stellaris's template, under the owner's four scales: Prima, Maiora, Magna and Grandia Opera Astralia. How the eras of `01 §9.6` sit beside the Kardashev ages is open (`07 D278`; DESIGN.md open questions).
- **Capstone structures** for the three ascension paths will be designed with the paths, after the Expanse, precursor and genemodding scopes (`07 D283`).
- **M19c, the AI's megastructure advisor,** is planned (≈ 450 lines): on the planning day, the highest-value structure it can afford within its budget's reserve, by personality weights, one at a time, giga structures only past their tech and era (`07 D283`).
- **The Asteroid Battery under the station redesign.** `07 D284` removes the batteries column from starbases in favour of a station of sections; no source read says what becomes of the Asteroid Battery's `batteries` hook.
- **Star ageing** (the Star Lifter) and the **Strip-mine Engine's shrinking** were cut (`07 D79`, `D82`).

## Discrepancies noted

- **Two AI rules for building structures.** `07 D278` call 8 (2026-11-23, built in T2 and merged): an AI orders great works once a year, the first work it knows whose first stage its markets hold twice over, the next Kardashev type's structure first. `07 D283` call 5 (2026-11-23, owner's answer A): M19c will add "an advisor on the planning day: the highest-value structure it can afford within its budget's reserve (energy, research, trade and defence weights by personality), one at a time". `D283`'s survey also states "the AI builds none (`D78`'s gap)". The sources do not say how M19c's advisor relates to T2's `ai::works`.
- **Structures in a captured system.** `07 D78` (2026-11-14): "No MVP path moves a starbase to a new owner (conquest takes colonies), so no transfer was built." `07 D113` (2026-11-14, M10, later): "A starbase at hull 0 in a battle with an empire at war with its owner passes to that empire: the starbase, its system and its starbase colony". The sources do not say what happens to the old owner's structures in a system whose starbase changes hands.

## Sources

- 13 §6 Megastructures
- 13 §7 Milestones
- 01 §9.8 Megastructures and ascension
- 07 Decisions: D57, D70, D78, D79, D80, D81, D82, D113, D115, D136, D278, D283, D284
- 05 Milestones: S1, S2, M19
- 00 Status (S2, M19a and T2 merges)
- DESIGN.md (pillar "Megastructures"; open questions; autonomous calls of 2026-11-14; change log)
- `data/content/megastructures.json` (placement, stages, hooks, ruins) and `data/content/strings/en.json` (names)

## Related pages

- [Overview](Overview)
- [starbases.md](Design-Starbases)
- [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)
- [trade-and-logistics.md](Design-Trade-and-logistics)
- [economy-and-resources.md](Design-Economy-and-resources)
- [progression.md](Design-Progression)
- [colonies-and-surfaces.md](Design-Colonies-and-surfaces)
- [diplomacy-and-war.md](Design-Diplomacy-and-war)
- [ai-and-victory.md](Design-Ai-and-victory)
- [ships-and-fleets.md](Design-Ships-and-fleets)
- [modding-and-data.md](Design-Modding-and-data)


---

# Ground combat

**Status:** built, as of 2026-11-23. GC1 (2026-11-14) on M8's armies and invasion; the AI's invasions in M10; the persona's and companions' bonuses in M12a and M12b.

Colonies change hands by invasion. An empire recruits brigades of data-defined kinds at its colonies, carries them on transports, and lands them on an enemy colony whose system is not contested. The invasion is then fought as a front across the colony's own hex surface, one daily step at a time, shaped by terrain, districts, the fortress, morale and fire support from fleets in orbit. The colony falls when its core hex falls or its defence pool is spent, and the surviving brigades garrison it. Orbital bombardment can soften a colony first.

## How it works

### Recruiting and carrying armies

Brigades are recruited at a colony (`ArmyRecruit` takes a kind) and boarded onto a transport with `ArmyLoad`; `Invade` lands them (`15 §1`, `05 GC1`). A transport carries 4 armies (`01 §10.5`); the catalogue's transport hull is the Marshaldeck (see [ships-and-fleets.md](Design-Ships-and-fleets)). The player's UI carries ArmyLoad and Invade since M-QA (2026-11-19; before that they were dev-console only) (`DESIGN.md`, autonomous calls 2026-11-19; `00-status.md`). The planet screen has one recruit button per kind (`05 GC1`).

### Army kinds

Kinds are a registry in data, `hulls.json`'s `rules.army.kinds`; mods add kinds by adding rows (`07 D90`). Kind 0 is the pre-GC1 assault army, so `01 §10.5`'s numbers hold (`15 §4`).

| Kind | Credits | Other cost | Days | Strength | Attack | Morale | Landing bonus | Tech |
|---|---|---|---|---|---|---|---|---|
| Assault | 100 | 10 goods | 60 | 5 | — | — | — | — |
| Marines | 140 | 10 goods, 5 alloys | 60 | 4 | 1.3 | — | 0.5 | Vocational Training |
| Heavy armour | 250 | 10 goods, 20 alloys | 90 | 9 | 1.4 | 1.2 | — | Nanocomposites |

Values from `data/content/hulls.json` (`rules.army.kinds`). `15 §3` describes marines as "strong for the first days after landing" and heavy armour as "tech, slow and strong". A kind needs its tech and trains to its own strength and morale (`05 GC1`). The `Armies` table holds `{empire, colony, fleet, strength}` plus `kind` and `morale` (`15 §1`, `§4`).

### The defender: the defence pool

The defender's strength is the colony's **defence pool**, kept on the colony as `defence_max − def_lost` (`15 §1`); there are no defender brigades (`07 D90`):

`defence = Σ soldier jobs × 3 + fortresses × 10 + pops × 0.1` (`01 §10.5`)

Soldiers, buildings and pops keep feeding the pool through that formula (`15 §3`).

### The front across the hex surface

The invasion is fought on the colony's hex surface (`01 §5`, at most 91 hexes), daily, only for invaded colonies with landed brigades (`07 D89`, `05 GC1`). `combat::invasion` calls `sim::ground`'s daily step for each of them; the `Grounds` table holds one row per ground battle: the colony, the held hexes as two 64-bit sets, the order, the target hex, the days and the last exchange (`05 GC1`).

As built (`05 GC1`):
- brigades take a **landing hex at the rim**;
- the **held hexes follow the share of the defence pool taken** (the test: held = n × share, with the core last);
- the **target hex's terrain, district and fortress divide the day's blows**;
- **orbital fire support** raises the attack by up to +50%;
- **morale** falls with the fight; when it breaks, the brigades withdraw onto transports in orbit;
- the **core falls with the pool**.

The daily exchange underneath is `01 §10.5`'s: `attack = Σ strength × rng(0.8–1.2)`; the defence loses `attack / 10` a day and the attackers lose `defence / 10` (`15 §1`). Assault brigades on plains with no support fight that exchange unchanged (`05 GC1`). Armies land the tick the transport reaches orbit and trade blows that tick; defence `D` against strength `S` takes `ceil(D / (S × roll / 10))` exchanges (`05 M8`, `07 D38`). Invasion requires the system not to be contested by enemy fleets (`01 §10.5`).

**The colony falls** when its core hex falls or its defence pool is spent; **the invasion fails** when the brigades are destroyed or their morale breaks (`15 §3`).

### Terrain, districts and the fortress

Each terrain has a `defence` modifier in `surface.json` (`07 D90`):

| Terrain | Defence | Terrain | Defence |
|---|---|---|---|
| Plains | +0% | Tundra | +10% |
| Hills | +25% | Ice | +10% |
| Mountains | +50% | Volcanic | +20% |
| Forest | +20% | Crater | +10% |
| Wetland | +15% | Ocean | +30% |
| Desert | +0% | Deck | +30% |

Values from `data/content/surface.json`. A developed district adds +30% to its hex (urban fighting); the fortress building's hex adds +100% and its neighbours +25% (`15 §3`; `rules.army.ground`: `districtDefence` 0.3, `fortressDefence` 1.0, `fortressNear` 0.25). All mountains hold a front longer than all plains (`05 GC1`). Hidden (unsurveyed) hexes are fought over as their terrain, with no fog bonus (`07 D92`).

### Orbital support and bombardment

- **Fire support.** Friendly aggressive fleets in orbit raise the attack by up to +50%, scaled by their power against the defence (`07 D90`; `supportMax` 0.5, `supportPower` 4.0 in `rules.army.ground`).
- **Bombardment.** A fleet on aggressive stance in orbit of an enemy colony reduces defence by 2% a day and pops by 0.05% a day, with an opinion penalty with everyone (`01 §10.5`; `bombard` 0.02, `bombardPops` 0.0005). A siege can be bombarded before the landing (`15 §3`).

### Orders: play, watch, simulate

Orders are commands, so play, watch and simulate are one run by construction (`07 D89`):

| Mode | What it is |
|---|---|
| Play | daily orders as commands: `GroundOrder` (assault, hold, withdraw) and `GroundTarget` (the next target hex; a click on a frontier hex targets it) |
| Watch | the planet screen's **Ground tab**: the surface with held hexes tinted in the attacker's colour, the target ringed, the frontier marked; a side panel with the brigades by kind with strength and morale, the defence pool, the orbital support, the last day's exchange, and Assault / Hold / Withdraw. An enemy colony being invaded opens on it |
| Simulate | the default orders, assaulting the weakest frontier hex on the way to the core; every AI battle uses them |

Holding takes no hex; only the invader orders its battle (`05 GC1`). A ground battle lasts days, so it never pauses the clock as space battles do (`07 D89`).

### Occupation and conquest

At defence 0 the colony changes hands: pops keep their ethics, the colony takes stability −30 for 5 years (1,800 days), and the occupier gets a "recently conquered" modifier (`01 §10.5`). The surviving brigades garrison the colony and shrink that penalty by up to half, scaled by garrison strength against the pops, so it lies within −30 to −15 (`07 D91`, `05 GC1`; `occupation` 0.5, `garrisonPerPop` 1.0).

During a war, every colony taken is recorded as an occupation with its owner before; a colony retaken by its owner ends its occupation. Each occupied claim gives +10 war score, and a side's exhaustion grows by 30 per 30 days × its pops occupied over its pops (`07 D112`; see [diplomacy-and-war.md](Design-Diplomacy-and-war)). Capturing a system's starbase does not take its other colonies, which still need an invasion (`07 D113`).

### The persona and companions

| Effect | Rule | Source |
|---|---|---|
| Inspiration skill | empire army strength +5 / 10 / 15%, through the `army.attack` key | `01 §19.2`, `07 D91`, `D134` |
| Leading in person | a brigade's blows × 1.25 when the personal fleet stands in the colony's system | `07 D137` |
| Companion admirals | a companion admiral whose fleet stands in the colony's system adds 5% per skill point to the persona empire's brigades' blows there | `07 D140` |

### The AI

The AI fights its ground battles on the default orders (`07 D91`). Its war AI (M10) chooses invasions (`07 D116`): at war, the enemy colony needing the least force (`force_needed`) among those it can reach in space that are not contested (no battle, no armed enemy fleet, no hostile starbase, so a captured or absent starbase first). Transports at home load `force_needed × 1.5` of the home system's brigades and invade together; one brigade is recruited at the capital each plan, and transports are built up to the force, four at most. These numbers were left untuned until the economy balance pass (`07 D116`).

### Budget

A ground battle's day costs ≤ 25 µs native for 91 hexes and 20 brigades, and nothing while no colony is invaded; there is no per-tick polling (`07 D92`). GC1 measured 23.9 µs against its 30 µs line (25 µs × calibration 1.19) on the AI's largest surface (61 hexes) with 20 brigades (`05 GC1`).

### RTS and TB

One daily step in both clocks (`07 D92`). In RTS, orders take effect on the next tick, and the feed reports the player's landings and withdrawals; the clock never pauses for a ground battle. In TB, orders are given in the planning phase and the turn's 30 days resolve them; the turn report lists the days (`15 §3`).

### Other ground numbers in data

`rules.army.ground` in `data/content/hulls.json` also holds `landingDays` 5, `moraleLoss` 1.5, `moraleBreak` 0.25, `moraleRecover` 0.05 and `hold` 0.5. The design docs name these rules (the landing bonus, morale loss and break, holding) without spelling out the formulas these values enter.

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Ground combat | The owner: design a ground combat system; planet-surface exploration still undecided | 2026-11-13 | `07` D7 (the owner's answers at the top) |
| Representation | B. A front across the colony's hex surface, resolved daily (not the abstract exchange alone, nor a real-time ground engine of about 4k LOC) | 2026-11-14 | `07 D89` |
| Modes | Play = daily orders as commands, watch = the Ground tab, simulate = default orders; no clock pause | 2026-11-14 | `07 D89` |
| Unit kinds | A. A data registry: assault, marines, heavy armour | 2026-11-14 | `07 D90` |
| Defences | A. The defence pool stays the defender's strength; terrain, districts and the fortress hex shape where it holds | 2026-11-14 | `07 D90` |
| Orbital support | A. Fire support up to +50% plus bombardment | 2026-11-14 | `07 D90` |
| Persona and companions | A. Hooks in GC1, built with M12a and M12b | 2026-11-14 | `07 D91`, `D137`, `D140` |
| Occupation | A. Garrisons shrink the conquest penalty by up to half; revolts with M11's situations | 2026-11-14 | `07 D91` |
| AI | A. Default orders; M10's war AI decides invasions with `force_needed` | 2026-11-14 | `07 D91`, `D116` |
| RTS and TB | A. One daily step in both | 2026-11-14 | `07 D92` |
| Surface exploration | A. Out of MVP, with hooks | 2026-11-14 | `07 D92` |
| Owner review | The owner kept all of GC1's recommended calls ("Keep all") | 2026-11-14 | `07 D89`–`D92`, `DESIGN.md` |

## Not yet built or still open

- **The persona's own brigade.** Leading an assault in person as a brigade with the persona's bonus and a death roll, saved by a story point (`15 §3`, D91), was not built in M12a or M12b (`07 D137`, `D145`); FC2's plan moved it to M16 and FC3 (`07 D216`). No source read records it as built; only the ×1.25 leading-in-person bonus exists.
- **Revolts, resistance and partisan brigades** come with M11's situations, a hook in GC1 (`15 §3`, `07 D91`); no source read records them as built for occupied colonies.
- **Surface exploration** (sites, ruins and encounters on hexes) stays a later call for the owner (`07 D92`; `07` D7).
- **AI invasion tuning.** M10's invasion numbers were left to the economy balance pass (`07 D116`).

## Discrepancies noted

- **How the front advances.** `15 §3` (D89): "Each day they assault one frontier hex, the defender's hex next to theirs. ... A won day flips the hex." `05 GC1` (builds and acceptance, 2026-11-14): "held hexes following the share of the defence pool taken" and the test "held = n × share, the core last". The first describes one hex flipped per won day; the second ties the count of held hexes to the share of the pool taken. `05 GC1` is the build record, but `15 §3` was not amended, so which wording states the intended rule is not recorded.
- **Fortress or citadel.** `01 §10.5` and `15 §3` name the fortress building; `05 GC1` speaks of "the target hex's terrain, district and citadel" and "terrain and citadel defence", and its cut list names "the citadel's neighbours' bonus". The sources do not say whether "citadel" is the same building.

## Sources

- 01 §10.5 Armies and invasion
- 01 §5.2 Surface hexes
- 15 §1 What exists, §2 The reference games, §3 The calls, §4 How GC1 builds it
- 07 Decisions: D7 (the owner's answers of 2026-11-13 at the top), D38, D89, D90, D91, D92, D112, D113, D116, D134, D137, D140, D145, D216
- 05 Milestones: M8, GC1
- 00 Status (M-QA merge, 2026-11-19)
- DESIGN.md (pillar "Ground combat"; autonomous calls of 2026-11-14 and 2026-11-19)
- `data/content/hulls.json` (`rules.army`, defence and bombardment rules) and `data/content/surface.json` (terrain defence)

## Related pages

- [Overview](Overview)
- [combat.md](Design-Combat)
- [colonies-and-surfaces.md](Design-Colonies-and-surfaces)
- [diplomacy-and-war.md](Design-Diplomacy-and-war)
- [ships-and-fleets.md](Design-Ships-and-fleets)
- [factions-and-characters.md](Design-Factions-and-characters)
- [events-and-crises.md](Design-Events-and-crises)
- [starts-and-exploration.md](Design-Starts-and-exploration)
- [ai-and-victory.md](Design-Ai-and-victory)
- [time-and-modes.md](Design-Time-and-modes)


---

# Progression

**Status:** in progress, as of 2026-11-25. Traditions, government, policies, edicts, eras I–III and the modifier system are built (M7); the tech tree rework is on track T, with T1 to T4 and T5a (hooks and bench rows) built; T5b (the Dyson line and the pace) waits for the owner's answers on five design points.

Progression is how an empire changes what it is and what it can do over a 300-year game. Technology is an open tree of 399 techs climbing toward Kardashev Type III; traditions buy perks with unity and lead to ascension perks; authority, civics, policies and edicts set the rules of government; eras unlock systems as an empire grows. Every effect of all of these reaches the simulation through one path, the modifier system, so every number on screen can show the stack that made it.

## How it works

### The modifier system: one effect pathway

Every effect from techs, traditions, civics, authorities, policies, designations, ascension perks, edicts, events, situations, leaders, difficulty and the persona's skills is a modifier row (`02 §5`, built in M7, `07 D36`).

- A row is `(scope kind, scope id, key, value, op, source, expires tick)`. Keys are interned names such as `job.miner.output` or `pop.skill_gain`; an unknown key is refused at load.
- The effective value of a `(scope, key)` pair is `base × Π(1 + mul) + Σ add`.
- **Factors** (job outputs, research, unity, trade value, freight, costs, growth, mortality, skill gain and decay, adjacency scale, piracy growth) fold with base 1, and the consumer multiplies in its own base (for example 4.0 per miner). **Terms** (happiness, stability, amenities, housing, slots, caps, service radius, skill output, tax tolerance, accessibility, piracy security, influence's add) fold with base 0 (`07 D36`).
- The cache holds one entry per `(scope, target, key)` with `[Π(1 + mul), Σ add]`. Empire entries sit in a dense empire × key array; a colony's value is its owner's entry combined with its own, written into colony columns on refold.
- Any insert, removal or expiry marks its key dirty. Dirty entries refold in daily step #9 and again in #27 (`cleanup`), so nothing is reapplied unless its inputs changed. The cache is derived, never saved, and rebuilt on load.
- The tooltip query returns the full stack with sources, for example "Miner output: 4.0 base; +15% Mining World; +10% Mining Guilds; −25% low stability". `CLAUDE.md` requires a breakdown on every number shown.

The M7 acceptance test folds the Mining designation, Mining Guilds and Deep-Core Mining into the homeworld's miner factor `1 × 1.15 × 1.15 × 1.10`, and changing one modifier refolds exactly one cache key (05 M7).

Designed and not built: under the tech rework, tech modifiers will stop being rows. The fold of a dirty `(empire, key)` entry will add the known techs that carry the key from an index built at load, and the tooltip will list them as it lists rows (`07 D278 §13`; on T2's list of work still to do).

### Technology: an open tree to Kardashev Type III

The owner reworked the tree on 2026-11-21 (`07 D278`). The rules below are `01 §9.1` as rewritten for it.

**Fields and categories.** Three fields, physics, society and engineering, with four categories each (twelve in all). Physics gained astrogation, society gained security, engineering gained voidcraft (`07 D278 §1`). Each field has one active project, and its daily research points go to it.

**Researchable.** There is no draw. An empire may research any tech that is researchable:

| Condition | Rule |
|---|---|
| Prerequisites | all of its `prereqs` known, and at least one of its `any` when it names some; either may be in another field |
| Tier | tier t + 1 of a field opens once 6 of its tier-t techs are known (was 4 before the rework) |
| Deed | its `when` condition holds, when it has one: a dig, a first contact, a battle, a great work, a finished tradition tree; the tree shows the deed in words |
| Age | its age's Kardashev type is reached: Type I for tiers 4–5, Type II for tiers 6–7 |
| Insight (not built, T2) | or its insight has reached half, which waives all of the above (tech skipping) |

No prerequisite sits in a higher tier, and there are no forks: nothing an empire researches closes anything else (`07 D278 §3`).

**Tiers and ages.**

| Age | Tiers | Content | Gate |
|---|---|---|---|
| Toward Type I | 0–3 | near space, the first jump, an interstellar empire; ends with the Dyson Swarm | none |
| Toward Type II | 4–5 | stellar engineering toward a full Dyson Sphere | Type I |
| Toward Type III | 6–7 | speculative and gigastructural science | Type II |

All tier-0 techs are known at game start.

**Counts.** 399 techs, 133 a field, distributed 2 / 18 / 22 / 24 / 20 / 18 / 16 / 13 over tiers 0–7. 49 are gated by deeds, and 4 are found rather than researched (reached by insight from salvage once T2 is built). No empire is meant to finish the tree in a 300-year game: a strong one reaches Type III around years 280–320 knowing about 80% of it (`01 §9.1`). Techs grant buildings, districts' extra caps, ship hulls and components, modifiers, policies and the great works.

**Goals are the queue.** An empire keeps up to six goals in order. A field with no project takes the cheapest researchable tech of its own field on the way to the first goal that has one (a prerequisite or the goal itself; from an `any` group with none known, any of them). When the way holds a tech in a tier still shut, the field takes the cheapest researchable tech of the tier under the lowest shut one, so a goal opens its own tiers. With no such step, a field on Auto takes its cheapest researchable tech; with Auto off it waits and prompts. Every empire starts with the Hyperspace Drive as its first goal. `PickResearch` takes any researchable tech of the field at any time, and the field's banked points stay. Goals and picks are commands (`SetResearchGoals`, `PickResearch`) in both RTS and TB.

**Cost.**

```
cost(tech) = base(tier) × (1 + 0.012 × techsKnown) × sprawlPenalty × empire.techCost
             × (1 − diffusion) / research.speed(category)
base = [0, 110, 380, 2400, 4800, 10000, 22000, 50000][tier]
```

The Hyperspace Drive carries an explicit cost of 150. `research.speed.<category>` divides the cost; it replaced the old draw weights, at a third of their values (`07 D278 §2`).

**Diffusion.** A tech costs 4% less for each empire in contact that knows it, 8% for a commercial-pact partner, and 12% for a pact partner while the empire's trade policy has no tariff (Free), times `research.diffusion`, capped at 50%. Free trade with empires ahead is the fastest way to catch up (the owner's call 4, `07 D278 §15`). As built, diffusion is summed over the contact mask at the cost check (track T's call).

**The Kardashev rating and types** (built in T2's first slice).

| Rating | Energy made by the empire's markets a month |
|---|---|
| K 0.73 | 45 |
| K 1.0 | 2,000 |
| K 2.0 | 50,000 |
| K 3.0 | 1,000,000 |

The rating is read once a year, interpolated on the logarithm between the anchors (`research.kardashev.energy`). A type is a latch: an empire takes the next type when its rating reaches it and it holds that type's finished structure (the Dyson Swarm for Type I, the Dyson Sphere for Type II, none for Type III), checked yearly and when a work is finished. It keeps the type whatever happens to its energy afterwards. The same rule holds for AI empires, which order the great works themselves (below).

**An idle field's bank** holds at most the cost of one tech of the next age's first tier at that day's price: tier 4 before Type I, tier 6 before Type II, tier 7 after.

**Pacing.** The starting empire makes 7 points a field per 30 days (2 researchers × 6 / 3 plus Capital I's 3 per field). The Hyperspace Drive costs `150 × (1 + 0.012 × 6) = 161`, so it lands at about 2 years. Opening tier 2 takes six tier-1 techs at about 123 each, about 740 points a field. On the T2 host runs, by year 100 between 1 and 4 of the 8 AI empires had reached Type I, depending on the seed (`07 D278`, "Built: T2, the first slice").

**The three ways to learn a tech** (designed, T2; the owner's call 2, `07 D278 §5`):

| Way | Rule |
|---|---|
| Research | the empire's researcher pops, points a field a month against the project's cost |
| Reverse engineering | a ship, building or relic of unknown tech taken apart at the capital (`ReverseEngineer`, `Dismantle`): 60 days for a ship, scaled by hull size; each unknown tech aboard gains 30% insight; the item is consumed |
| Scavenging | digs and anomalies pay insight through the event effect `{ "tech": "progress", "id", "points" }`; caches pay hulls and relics to use or take apart |

**Insight** is the share of a tech's cost already paid. An empire holds up to sixteen insights; when full, the smallest gives way. Holding and using an item of unknown tech builds insight at 1% a month to a ceiling of 50%. A battle against a design with a component the empire cannot build gives 2% a battle, to 25%. FC4's steal-tech operation gives 50% on the stolen tech. At half, the tech can be researched without its prerequisites, tier, type or deed; at the whole, it is known.

**The tree view** (T4, built 2026-11-24) is the research screen: tiers as columns under the three ages' banners, a category a row, a tech a card, every prerequisite a line. The header carries the era, the techs known, the Kardashev rating and type, and what the next type still needs. A card shows its state (known, the project, researchable, waiting for its deed, locked), its insight, and in its hint what it does, what shuts it and its cost factor by factor. A click researches a researchable card and sets or clears a goal on any other; the goals (six, in order, drag to reorder), a search and the filters sit beside the tree. The fleet screen lists a fleet's unknown tech and carries the Reverse engineer order; reaching a type fires its event and a line in the turn report (`01 §9.1`).

**Tech hooks.** About 150 techs carry one modest modifier and point at an effect the game cannot express yet: M17's trade, FC4's espionage, the great works, new components, building grades, policies. `docs/20` lists every one (for example `swarm_fabrication`, `stellar_collectors` and `swarm_coordination` "Required for the Dyson Swarm great work"; `planetary_remodelling` "Unlocks terraforming of settled worlds"). Track T wires them in T2 and T5 as those systems land (docs/20).

**Bigger without power creep.** Over tiers 0–3, the stacked bonus on every modifier key the old tree had equals the old tree's; `07 D273`'s two size levers (`empire.colonyPops`, `empire.popUpkeep`) stay on their own techs (track T's call, `07 D278 §15`).

### Traditions and ascension

Built in M7 (`01 §9.2`).

- **Trees.** The MVP has three: Expansion, Prosperity and Discovery. 1.0 adds Harmony, Supremacy, Diplomacy, Mercantile and Adaptability (`05`'s M22 lists 8 trees).
- **Structure.** Each tree has an adoption effect, 5 perks (taken in an order within simple prerequisites) and a finisher.
- **Cost** in unity: `cost = 150 × (1 + 0.4 × perksTaken) × (1 + 0.01 × pops) × sprawlPenalty`.
- The Prosperity tree's perks include Guilds (`pop.skill_output` +0.05) and Fair Levies (`tax.tolerance` +0.05) (`07 D27`, `D28`).
- A second tree concurrently needs Era II.
- **Ascension perks.** Completing a tree grants an ascension perk slot. Built: three perks (Technological Ascendancy, Interstellar Dominion, One Vision). Master Builders and the ascension path openers are 1.0.

**Ascension paths** are designed and not built. `01 §9.8` describes Biological, Synthetic and Psionic paths, each a 3-perk chain with special projects and events. The owner revised this on 2026-11-23 (`07 D283`, call 2): the Psionic path is the path to the Singularity (the Expanse scope); a Synthetic path opens after contact with the Automata; Biological ties to genemodding, which a new scoping session will plan for both the persona and the empire's species. The capstones will be new structures designed with the paths (call 3, option B). M19b, the ascension phase, left M19 and will be built once those three scopes land. Full AI persona cores belong to the synthetic ascension; AI cores as a researchable tech that improves characters' efficiency are the owner's call of 2026-11-23 (`07 D284`).

### Authority, civics, policies and edicts

Built in M7 (`01 §9.5`).

**Authority.**

| Authority | Elections | Bonus |
|---|---|---|
| Democratic | every 5 years (`07 D241` call 5) | +5 stability; vote weight 1 per pop |
| Oligarchic | every 10 years (council) | +5% research |
| Autocratic | ruler for life | +1 influence |

Reform costs 200 influence and has a 10-year cooldown. 1.0 adds Hive, Machine and Corporate authorities (`01 §3`); machine and hive species are NPC only (`07 D230`). The authority also sets whose weight counts in politics, the agitation rate and suppression price ([Factions and characters](Design-Factions-and-characters)).

**Civics.** Two per empire, a third from a perk. Each is a set of data-driven modifiers. Built (8):

| Civic | Effect |
|---|---|
| Mining Guilds | +15% ore |
| Merchant Republic | +1 hub, +10% trade value |
| Technocracy | +10% research |
| Warrior Culture | +10% fleet damage |
| Agrarian Idyll | +food, +amenities from agricultural districts |
| Meritocracy | `pop.skill_gain` +15%, `pop.skill_output` +0.05 |
| Corporate Charter | `tax.tolerance` +0.05, +5% trade value |
| Feudal (FC4) | no modifier rows; NPC fiefs. Open to autocracy and oligarchy only; a seeded 25% share of AI empires hold it (`07 D219`) |

About 30 civics are proposed for 1.0 in CV1 ([Civilisations and species](Design-Civilisations-and-species)).

**Policies.** Free to set, with a 10-year lock unless stated:

| Policy | Options |
|---|---|
| Living standard | as `01 §6` |
| Research split | Equal (⅓ of researcher output to each field) or Focused on one field (60 / 20 / 20); the Capital's +3 per field is unaffected |
| Immigration | Open / Selective (default) / Closed; 1-year lock |
| Emigration | Free / Closed; 1-year lock |
| Refugees | Accept / Refuse; 1-year lock |
| Trade | Free (tariff 0) / Protectionist (domestic markets first, tariff 20%) |
| Taxation | Low 0.10 / Standard 0.25 / High 0.40 / Punitive 0.60; 1-year lock |
| Commerce | Credits / Mercantile / Innovation |
| Education | Minimal (`pop.skill_gain` × 0.75, free) / Standard (free) / Generous (× 1.5 at 0.2 credits per pop per 30 days); 1-year lock |
| War philosophy | defensive / unrestricted |
| AI and automation stance | flavour for 1.0 events |

Conceding a party's demand is a policy change.

**Edicts.** Temporary empire buffs paid in influence or credits per 30 days, available from Era II. Built: Mining Subsidies (+10% ore for 2 years at 30 credits per 30 days), Research Grants, the Trade Charter, and the Levy (100 influence once; seizes 50% of every colony's private wealth; −10 happiness for a year). M17b's market subsidies (levels 1–3) are paid on the edicts row.

### Eras

Built for eras I–III in M7 (`01 §9.6`); Era IV is 1.0.

| Era | Gate | Unlocks |
|---|---|---|
| I Dawn | start (pre-FTL) | in-system colonies, domes, habitats, stations |
| II Expansion | Hyperspace Drive known, First Jump completed, and ≥ 1 colony outside the home system | hub designation beyond the capital, starbase size 4, a second tradition tree concurrent, edicts |
| III Consolidation | 8 colonies across ≥ 4 systems or a federation, and any tier-3 tech | factions active (1.0), trade fleets (1.0), mid-game crisis timer (1.0), Galactic Exchange (1.0), starbase size 6 |
| IV Apex (1.0) | any tier-5 tech and 20 colonies or a megastructure | megastructures, ascension paths, endgame crisis eligible, Apex buildings, Citadel tier |

Era is per empire; the **galactic era** is the highest over all empires and drives crisis eligibility. Gates are checked daily. Reaching an era pays unity and fires a flavour event. Score weights scale by era (era bonus 0 / 50 / 150 / 400). FC4's AI operations take their rank from the era (rank 1, 2 or 3 in Dawn, Expansion and Consolidation, `07 D219`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Modifiers as the one pathway | Every effect is a modifier row folded into a dirty-keyed cache; factors at base 1, terms at base 0; refold in #9 and #27 | 2026-11-06 | `07 D36` |
| Skill modifiers | `pop.skill_gain`, `skill_decay`, `skill_output`; Education policy; Meritocracy; Guilds | 2026-11-02 | `07 D27` |
| Taxation | Four tax rates, commerce policy, the Levy edict, Fair Levies | 2026-11-02 | `07 D28` |
| Tree size and shape | About 400 techs as a climb to Kardashev Type III (replaced 76 techs over tiers 0–4) | 2026-11-21 | `07 D278 §15` call 1 |
| How research is chosen | An open tree with goals; research, reverse engineering and scavenging; use of unknown tech builds insight and allows skipping (replaced the Stellaris-style draw of three, `01 §9.1` before the rework and `07 D36`) | 2026-11-21 | `07 D278 §15` call 2 |
| Forks | No forks | 2026-11-21 | `07 D278 §15` call 3 |
| Deeds and diffusion | Deeds gate techs; diffusion, with free trade with stronger empires raising the catch-up rate | 2026-11-21 | `07 D278 §15` call 4 |
| Kardashev rating | Measured from energy captured | 2026-11-21 | `07 D278 §15` call 5 |
| Megastructures in the tree | Swarm part of the Type I gate, Sphere of Type II, speculative tech the path to Type III; a separate session scopes the great works (Prima, Maiora, Magna, Grandia Opera Astralia) | 2026-11-21 | `07 D278 §15` call 6 |
| Unknown tech | Use to 50%, dismantle for more | 2026-11-21 | `07 D278 §15` call 7 |
| The Dyson line | A basic Swarm known from the start; techs raise its enclosure; a full Sphere takes the star's whole output; a Dyson Aperture later; Type I is a full Swarm, Type II a closed Sphere; anchors and costs re-set to the pace | 2026-11-25 | `07 D278 §16` |
| The AI and the great works | The AI builds the great works as part of development, triggered by having the resources; a type needs its structure for every empire | 2026-11-23 | `07 D278 §15` call 8 |
| Track T's cost calls | Tier 3 at twice its first cost (base 2,400, tier 4 4,800, tier 5 10,000), replacing `07 D278 §6`'s starting base of 1,200 / 3,400 / 9,000; energy anchors 45 / 2,000 / 50,000 / 1,000,000; idle bank capped | 2026-11-23 | `07 D278 §15` |
| Ethics model | Stellaris's eight ethics (replaces `01 §9.3`'s Collectivist–Individualist axis with Authoritarian–Egalitarian in CV1) | 2026-11-16 | `07 D230` call 1 |
| Ascension paths | Psionic is the Singularity path; Synthetic after Automata contact; Biological with genemodding; new capstones; M19b leaves M19 (replaces `01 §9.8`'s three generic paths) | 2026-11-23 | `07 D283` calls 2–3 |
| Election intervals | 5 years for democracies, 10 for oligarchic councils (replaces `01 §9.5`'s 10 and 20) | 2026-11-18 | `07 D241` call 5 |
| Feudal civic | Autocracy and oligarchy only; no modifier rows; 25% of AI empires hold it | 2026-11-21 | `07 D219` |

## Not yet built or still open

- **Left from T2 to T4** (all three are built, 2026-11-24; `07 D278`'s "Built" blocks): taking a captured building apart and the colony screen's unknown-tech mark, which wait for a recorded capture (`07 D284`); `unlocks.deposits`, left out of T5 for T4's field readout or the exotic chain.
- **T5a** (built 2026-11-25, awaiting merge): six modifier keys on trade and intrigue (`exchange.fee`, `subsidy.cost`, `smuggle.detect`, `insurance.premium`, `intel.ops`, `intel.defence`) on thirteen techs, the two bench rows of `07 D278 §13`, and a pace line in `tools balance`. One tracked `07 D147` line still misses: no empire knows every MVP tier-3 tech by year 200 on seeds 1–2.
- **T5b, the Dyson line and the pace** (the owner, 2026-11-25; `07 D278 §16`): measured over 300 years, Type I came in years 56–86 against the 100–140 wanted and nobody reached Type II, because a Swarm and a Sphere were percentages, not energy. Every empire will know a basic Dyson Swarm from the start; a ladder of techs raises how much of the star it captures; a full Sphere takes the star's whole output; a Dyson Aperture later removes the Sphere's cost to its system. Open with the owner: how the ladder works, the section count, the Aperture's place, what the dimming is, and Type III's anchor before the great works land. Also found: no AI can order a Sphere, because none holds machine tools.
- **Ascension paths, their capstones and the ascension victory**, after the Expanse (`D281`), the precursors and Automata worldbuilding (`D280`) and a genemodding scope. `D280` and `D281` are named in `07 D283` but have no entry in `07` yet.
- **Era IV** (Apex) and its unlocks are 1.0.
- **Open with the owner:** how the eras of `01 §9.6` sit beside the tech tree's three Kardashev ages. Era IV's gate names a tier-5 tech and a megastructure, which under the rework reads as Type I (`07 D278 §15`; DESIGN's open questions).
- **The great works' roster** and their four scales are scoped in a separate session.
- **The civic count** for 1.0: about 30 stands as the proposal; the owner has not answered (`07 D230`).
- **Ethics drift** (`01 §9.3`) is M15's remainder and not built; cohort ethics vectors are static.
- **Stale text to fold:** `01 §4.1` still says the Hyperspace Drive is "always among the first three physics options offered"; `01 §3`'s research row prices it at 168 points (the old `1 + 0.02 × techsKnown` multiplier), where `01 §9.1` gives 161; `01 §9.5` still gives election intervals of 10 and 20 years; `05`'s M22 row still plans 250 techs.

## Discrepancies noted

- **Era III's unlocks against systems built without an era gate.** `01 §9.6` lists "Factions active (1.0), trade fleets (1.0), ... Galactic Exchange (1.0)" as Era III unlocks. FC1's parties form on every empire's politics day with no era condition in its plan or build record (`07 D214`); M17c founds the Galactic Exchange "on the third pact or monthly when absent" (`00-status`, 2026-11-23); M17a's trade fleets name no era. The sources do not say whether these systems are gated by Era III as built.
- **Tech counts gated by deeds.** `01 §9.1`: "49 are gated by deeds". `07 D278 §4`: "About 60 techs are gated." The T1 build record gives 399 techs but no deed count.

## Sources

- 01 §9 Civilisational progression (§9.1 to §9.8)
- 01 §3 Empires and species (opening research row)
- 02 §5 Modifiers
- 05 M7, 05 T and the 1.0 milestones
- 07 decisions: `D27`, `D28`, `D36`, `D219`, `D230`, `D241`, `D278`, `D283`, `D284`
- 20 Tech hooks
- 17 Civilisations, discovery and terraforming (§0, §1)
- DESIGN.md (open questions, change log)
- 00-status.md (merge records for T1 and T2's first slice)

## Related pages

- [Overview](Overview)
- [Civilisations and species](Design-Civilisations-and-species)
- [Factions and characters](Design-Factions-and-characters)
- [Megastructures](Design-Megastructures)
- [Economy and resources](Design-Economy-and-resources)
- [Pops and society](Design-Pops-and-society)
- [Events and crises](Design-Events-and-crises)
- [AI and victory](Design-Ai-and-victory)
- [Starts and exploration](Design-Starts-and-exploration)
- [Roadmap and status](Design-Roadmap-and-status)


---

# Civilisations and species

**Status:** designed, as of 2026-11-23. The MVP species, traits, ethics on cohorts and the precursor chain are built (M11, M14); the 1.0 scopes of `docs/17` (CV1 ethics and civics, SP1 species, LR1 lore and precursors, DX1 discovery sites) carry the owner's calls but are not placed on a track.

This system is what a civilisation is made of: the species its pops belong to, the ethics they hold, the civics that shape its government, the layered backstory of the precursors who came before, and the discoveries that reveal it. In the MVP an empire picks a species and traits in the creator and finds one precursor chain across the galaxy. The 1.0 design adds eight baseline species including humans, a persona with a species of its own, Stellaris's eight ethics, about 30 civics, and a Starsector-style backstory told only through what players dig up.

## How it works

### Species (built)

A species is data (`01 §3`): a name list, a portrait id, a climate preference, and 2–4 traits from a trait list with point costs. Traits are modifiers on the cohorts of that species, so they reach the economy through the modifier pathway ([Progression](Design-Progression)).

Built in M14 (`07 D99`): four species, each with combat traits as its own, and ten traits on a 2-point budget. The economic traits are the empire creator's choices.

| Species | Its own traits |
|---|---|
| Vaelan | Hardy |
| Orrin | Nimble, Keen-Eyed |
| Sesk | Hardy, Stalwart |
| Lumen | Keen-Eyed |

| Trait | Cost | Effect |
|---|---|---|
| Industrious | +2 | +4% job output |
| Intelligent | +2 | +5% to each research field |
| Rapid Breeders | +1 | +10% growth |
| Charismatic | +1 | +10% influence |
| Hardy | +1 | +5% hull |
| Stalwart | +1 | +5% armour |
| Keen-Eyed | +1 | +5% accuracy |
| Nimble | +1 | +5% evasion |
| Slow Learners | −1 | −15% skill gain |
| Quarrelsome | −1 | −3 stability |

The species' own traits are combat traits because economic defaults (Industrious, Rapid Breeders, Charismatic, Intelligent) grew seed 1's year-150 world from 127 to 144 colonies and pushed one AI plan over its CPU tripwire (`07 D99`). Leader and empire names come from each species' list; there are no free-text names yet (`07 D97`). FC2 added given and family names per species in `notables.json`, moddable (`07 D216`).

The Sol start offers a Human preset: continental preference, no traits (`01 §3.1`, [Starts and exploration](Design-Starts-and-exploration)).

### The empire (built)

An empire has a name, an adjective, a flag (shape and 2 colours), a species, **ethics** (3 points over the ethic axes; a fanatic pole costs 2 points), an **authority**, 2 **civics** and an origin (MVP: Standard only) (`01 §3`). Authority, civics and policies are described on [Progression](Design-Progression).

### Ethics

**As built** (`01 §9.3`, `§6.1`): four axes with two poles each: Collectivist–Individualist, Militarist–Pacifist, Materialist–Spiritualist, Xenophile–Xenophobe. Each pop cohort holds an 8-vector of pole weights, each axis summing to 1. Cohort ethics feed `ethicsMatch` in happiness, so pops that disagree with the state are unhappier. In the MVP the vectors are static; FC1's parties read them, and FC4's ideology spread pushes them ([Factions and characters](Design-Factions-and-characters)).

**As decided for 1.0** (`07 D230` call 1): **Stellaris's eight ethics**, not the four-axis proposal of `docs/17`. Collectivist–Individualist becomes **Authoritarian–Egalitarian** in CV1 (`docs/19 §1`). The economic drift conditions of `01 §9.3` (austerity toward Collectivist, high trade value toward Individualist) move out of ethics and into the material blocs of the faction system, so the new axis is about hierarchy and liberty while economic interest is read from jobs (`docs/19 §3.2`).

**Drift** (`01 §9.3`, 1.0, not built): each cohort's vector will move 1% per 30 days toward a target computed from lived conditions:

| Condition | Pushes toward |
|---|---|
| Empire at war in the last 12 months | Militarist |
| 10+ years of peace | Pacifist |
| ≥ 25% of planet pops are other species, happiness ≥ 50 | Xenophile |
| Foreign empire occupied or raided the planet | Xenophobe |
| Research output per pop above empire median | Materialist |
| Low stability or recent crisis | Spiritualist |
| Trade value per pop high, living standard Comfortable | Individualist |
| Austere living standard, rationing, or shortages | Collectivist |

plus +2% per 30 days toward the empire's ethics and faction propaganda. The last two rows are the ones `docs/19 §3.2` moves into the blocs under the eight-ethics remap.

### Civics and origins (CV1, designed)

- **Civics** will grow from the built 8 (`government.json`; seven from M7 plus FC4's Feudal) to about 30, two per empire with a third unlocked by tradition. Each will be a modifier set plus at most one unique rule, all in data, gated by ethics and authority (`docs/17 §3.1`). The owner has not answered the count; about 30 stands until she does (`07 D230`).
- There is no xenophobe civic in the content; closed immigration plays its part in refusing foreigners (`07 D139`).
- **Origins** are **persona origins, not planet origins** (`07 D230` call 2): an origin is the persona's background, not an empire's start world. `docs/17`'s proposal of about 12 start scenarios (Lost Colony, Ruined Gateway Home, Precursor Heirs, Void Dwellers, Shattered Ring and others) was offered as planet origins and is not the decided form.

### Baseline species and the persona's species (SP1, designed)

The owner's calls (`07 D230` calls 4–6):

- **Humans plus seven core species**, expandable by mods.
- **The persona has a species:** a lifespan and core traits, plus traits picked at creation.
- **Machine and hive species are NPC only:** they exist in the galaxy as non-playable species.

The `docs/17 §3.2` proposal within those calls: a species row will carry a climate preference, a lifespan in years, a trait set from the point budget, a ship set (G2's generator parameters), a portrait set and name lists, plus an AI personality bias and a home template. The seven alien species will span the climate groups and include one lithoid and one aquatic. The persona's lifespan will feed its mortality (M12b) and interact with Life Extension; species traits will apply to the persona as character modifiers where they make sense (intelligent → science skill XP); a short list of persona-only birthright traits will be picked at creation.

**Habitability per species** will become a function of the planet's four terraforming parameters against the species' preference, replacing the climate-group table, recomputed only when a parameter changes (`docs/17 §3.6`, TF1; [Colonies and surfaces](Design-Colonies-and-surfaces)).

**Genemodding** for both the persona and the empire's species is unscoped; the owner tied it to the Biological ascension path on 2026-11-23 (`07 D283`). Techs that point at it already exist as hooks: `gene_tailoring` (changes to species traits), `xeno_hybridisation` (traits taken from other known species), `clade_engineering` (a subspecies suited to a chosen world type) (`docs/20`).

### Precursors and the layered backstory

**Built** (M11, `07 D85`, `D86`):

- **Six precursor sites** placed at galaxy generation; each is a project of 180 days whose event gives a **clue**. Every opened deep cache also gives a clue (S1's caches, [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)).
- **The sixth clue** reveals the precursor home system and fires `precursor_home`; the home is the lowest system id among the six sites, and it is surveyed in full. `01 §4.2` describes the reward as "relic + unique system modifier".
- The deep-space **Automata** guard caches; the **Precursor** hull lineage comes from salvage; ruined mega- and gigastructures sit in deep space and sometimes in normal space ([Megastructures](Design-Megastructures)).
- Under the tech rework, the precursor chain is about twelve techs across the three ages with no ordinary way in: they are gated on clues and reached by insight (`07 D278 §5`, designed for T2).

**Designed** (LR1, `docs/17 §3.4`; names are placeholders for the owner):

| Layer | What it is |
|---|---|
| The Elders | an ancient, long-extinct civilisation; made the gigastructure ruins, the deep-space anomalies and the oldest relics |
| The Concord | a recent galactic polity that built the original Gateway network and the beacon corridors; fell in the Severance, whose cause stays ambiguous until late; its relics are the colony items and blueprints; its automata are the deep-space Automata faction |
| The Successors | the present empires and pirates, the Concord's scattered heirs |
| A dormant power | a Concord remnant state sealed in the abyss, which wakes as one of M20's crises |

The owner's calls (`07 D230` calls 9–11): the fallen precursors are an offshoot of the storyline precursor (the dormant power is a Concord offshoot); lore is delivered as a **codex of fragments**; **the owner writes the lore**, and the spec session keeps only placeholders and hooks. Each fragment will be a log, an inscription or an item description; chains of fragments will unlock a lost tech or reveal a site, and the built precursor chain becomes the Concord chain's first act.

FC4 built the hooks for the owner's 20–30 authored characters: a condition list (an empire of the character's species reaching an era, a dug precursor site, a crisis) read when the roster fills a vacancy, and codex fragments revealed through Converse. No characters ship until she writes them (`07 D219` call 6).

### Discovery

**Built** (M3, M6, M8c, M11, F1; summarised in `docs/17 §1`):

| Mechanic | Rule |
|---|---|
| Survey | per body; an anomaly roll of p 0.18 (× 1.5 in a system with a research-related modifier) |
| Anomalies | level 1–10, weighted toward low early; a research project of `20 × level / (1 + 0.1 × skill)` days ending in a weighted event from the anomaly's pool; no failure outcome (`01 §4.2`) |
| Archaeology | sites at about 1 per 25 systems; 3 chapters at difficulty 4 as built, each `30 × difficulty / (1 + 0.15 × skill)` days then `d10 + skill ≥ difficulty` ([Events and crises](Design-Events-and-crises)) |
| Deep space | caches, stashes and rich fields, found by sensors along routes or by a deep survey; odds rising with depth (`07 D45`) |
| Hex features | ruins, alien life, crash sites, monoliths, hazards, ore veins found by expeditions (M6) |
| Flight-view signals | `signal` sites placed per (seed, cell) that carry the hyperspace incident events when flown to ([Flight view](Design-Flight-view)) |

**Lost technology** (`07 D230` call 7: "a visible tree plus lost technology that has to be discovered manually") is realised by the tech rework: 4 techs are found rather than researched, and reverse engineering, scavenging and the use of unknown items build insight toward any tech (`07 D278 §5`; [Progression](Design-Progression)). **Risky tech** is approved (call 8): `docs/17 §3.3` proposed a handful of lost techs with risks such as an AI-uprising risk feeding M20's crises. The tech rework's events table names "an AI-rights chain after Sapient AI; a containment failure; a precursor signal answered" among about a dozen `on_tech` events (`07 D278 §7`).

**Discovery sites** (DX1, designed; `07 D230` calls 12–14):

- Sites will be a moddable data registry **placed at galaxy generation** in hyperspace, deep space and systems, revealed by sensors and survey.
- Proposed kinds (`docs/17 §3.5`): derelict ships (salvage hulls or blueprints), research stations (lost tech, with a guard chance), cryosleeper vaults (pops plus a crew boost), beacon wrecks (lore and corridor charts), comm relays (reveal regions), automata factories (guarded; Automata hulls).
- Each site will have a condition (intact, damaged, looted), danger set by depth, and a **layered payout: reveal, then dig, then deep dig**.
- **Planet conditions with ratings**, from survey: deposits sparse, moderate, abundant, rich, ultrarich; ruins scattered, widespread, extensive, vast. The ratings will feed district output and planetary exploration.
- Anomalies stay, with their pools rewritten to point into the codex and the site registry; F1's signals and simulated incidents will use the same content.

**Planetary exploration** (PX1, designed; calls 18–20): teams led by a leader, a companion or the persona in person will crawl hidden hexes on timers, finding rated resource nodes that boost districts, relics usable as items or planet bonuses, treasure caches and dangers that hook into ground combat ([Colonies and surfaces](Design-Colonies-and-surfaces), [Ground combat](Design-Ground-combat)).

### The 1.0 milestones of `docs/17`

| # | Milestone | Contents | Est. LOC |
|---|---|---|---|
| CV1 | Ethics, civics and origins | the ethics remap, about 30 civics, origins; merges into M15 | ≈ 1.8k (+ content) |
| SP1 | Species | about 8 baseline species, the persona's species and birthright, the habitability function | ≈ 1.0k (+ content, portraits) |
| TT1 | Tech tree to 1.0 | replaced by track T's rework (`07 D278`, T1–T5) | ≈ 1.2k |
| LR1 | Lore and precursors | the eras, the codex, fragment chains, the dormant remnant hook into M20 | ≈ 1.0k (+ lore) |
| DX1 | Discovery sites | the site registry, layered digs, planet condition ratings, anomalies re-pooled | ≈ 1.6k |
| TF1 | Terraforming | the parameter model, projects with physical inputs, side effects, new kilostructures | ≈ 1.8k |
| PX1 | Planetary exploration | expedition teams, hex crawl, resource nodes, relics as items, surface dangers | ≈ 2.0k |

Suggested order after the MVP: SP1 and CV1, then DX1 and LR1 together, then TT1, TF1 and PX1 (`docs/17 §4`). FC1–FC4's build order put CV1 first, but FC1 shipped on the four built axes with the rename left as "a content and string change on top" (`07 D214`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| MVP species and traits | Four species with combat traits; ten traits on a 2-point budget; economic traits are the creator's | 2026-11-14 | `07 D99` |
| Precursor chain as built | Clues from the six precursor sites and every opened deep cache; the home is the lowest-id site's system (the alternative, clues from anomaly outcomes, was not taken) | 2026-11-14 | `07 D85` |
| Ethics | Stellaris's eight ethics (replaces the four-axis set of `01 §9.3` and the `docs/17` proposal) | 2026-11-16 | `07 D230` call 1 |
| Origins | Persona origins, not planet origins | 2026-11-16 | `07 D230` call 2 |
| Civic count | Not answered; about 30 stands | 2026-11-16 | `07 D230` call 3 |
| Baseline species | Humans plus seven core species, expandable by mods | 2026-11-16 | `07 D230` call 4 |
| The persona's species | Yes: lifespan, core traits, traits picked at creation | 2026-11-16 | `07 D230` call 5 |
| Machine and hive species | NPC only | 2026-11-16 | `07 D230` call 6 |
| Research shape | A visible tree plus lost technology discovered manually (realised by `07 D278`) | 2026-11-16 | `07 D230` call 7 |
| Risky tech | Yes | 2026-11-16 | `07 D230` call 8 |
| Precursor eras | The fallen precursors are an offshoot of the storyline precursor | 2026-11-16 | `07 D230` call 9 |
| Lore | A codex of fragments; the owner writes it | 2026-11-16 | `07 D230` calls 10–11 |
| Discovery | Sites placed at generation; reveal, dig, deep dig; rated planet conditions | 2026-11-16 | `07 D230` calls 12–14 |
| Planetary exploration | By teams or by the persona; relics as items or planet bonuses; rated nodes that boost districts | 2026-11-16 | `07 D230` calls 18–20 |
| Authored characters | Hooks now, the owner's characters later | 2026-11-21 | `07 D219` call 6 |
| Genemodding | To be scoped for persona and species, tied to the Biological path | 2026-11-23 | `07 D283` call 2 |

## Not yet built or still open

- **CV1:** the eight-ethics remap, about 30 civics, persona origins. The civic count awaits the owner.
- **SP1:** the eight baseline species, the persona's species, birthright traits, the habitability function. Species art (portrait ids) is a hook left by M14.
- **LR1:** the codex, fragment chains, the dormant remnant; the lore itself, which the owner writes.
- **DX1:** the site registry, layered digs and condition ratings.
- **Ethics drift** (M15's remainder).
- **Genemodding** has no scope yet.
- **The precursors and Automata worldbuilding** (`D280`) is named in `07 D283` as a scope that the Synthetic ascension path waits on; it has no `07` entry.
- **Stale text to fold:** `docs/17 §1` describes the built precursor chain as "6 clues from survey and anomaly outcomes", where `07 D85` built clues from the six precursor sites and opened deep caches; `01 §3`'s trait examples (Industrious +10% ore, Thrifty, Sedentary, Nonadaptive) are not the built traits of `07 D99`.

## Discrepancies noted

- **Whether the three-era backstory is adopted.** `docs/17 §3.4` call 9 offered "A (recommended): three eras (Elders, Concord, Successors) plus one dormant remnant power", B one precursor empire, C two eras with no dormant power. The owner's recorded answer (`docs/17 §0` row 9 and `07 D230`) is "The fallen precursors are an offshoot of the storyline precursor (the dormant power is a Concord offshoot)". The answer confirms a dormant power descended from the Concord but does not say whether the Elders layer stands.

## Sources

- 01 §3 Empires and species, 01 §4.2 Interstellar exploration, 01 §9 Civilisational progression (§9.3)
- 17 Civilisations, discovery and terraforming: §0 the owner's calls, §3 proposals
- 19 Factions and characters (§1, §3.2)
- 20 Tech hooks
- 07 decisions: `D45`, `D85`, `D86`, `D97`, `D99`, `D139`, `D214`, `D216`, `D219`, `D230`, `D278`, `D283`
- DESIGN.md (open questions, change log)

## Related pages

- [Overview](Overview)
- [Progression](Design-Progression)
- [Factions and characters](Design-Factions-and-characters)
- [Starts and exploration](Design-Starts-and-exploration)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Pops and society](Design-Pops-and-society)
- [Megastructures](Design-Megastructures)
- [Events and crises](Design-Events-and-crises)
- [Ground combat](Design-Ground-combat)
- [Flight view](Design-Flight-view)
- [Modding and data](Design-Modding-and-data)


---

# Factions and characters

**Status:** built, as of 2026-11-23. The persona's character and personal fleet (M12a), service, people and legacy (M12b), the Employee modes (M16), and the four faction slices FC1–FC4 are merged; ethics drift and the eight-ethics rename remain.

The player is a **persona**: one character with a personal fleet who can trade, fight for hire, serve an AI empire, or rule an empire as deeply as she chooses, delegating as it grows. Around her, every empire has **political parties** built from what its pops believe and how they earn their living, and **named notables** (rulers, heirs, councillors, admirals, governors, magnates) who lead those parties, scheme, marry and rebel. The persona's personal acts (a quest done, a campaign funded, a marriage, an assassination) move macro politics, and party politics in turn decide who holds posts, who goes to war and who secedes.

## How it works

### The persona: frameworks and kits

One persona entity, three frameworks, seven starting kits (`01 §19`, `07 D31`, `D44`, `D218`). Modes are kits and UI emphasis, not separate rule sets.

| Framework | Kits | The persona is | Commands allowed |
|---|---|---|---|
| Empire | Empire | the ruler of empire 0 | everything |
| Independent | Trader, Mercenary, Settler; pirate as a path | a captain under a one-row free company with no colonies and a personal purse | own fleets and holdings, market trades, contracts |
| Employee | Commander, Leader, Scientist | a post in an AI-run empire 0 (`AI_RUN`) | the post's own fleet, colony or ship, and `Captain` acts; empire-wide orders are refused with `NotYourPost` |

**Start.** Persona games pre-run the galaxy AI-only for 10 years (about 3,650 ticks, deterministic, behind a progress bar; 0.61 s on the host bench against a 3 s line) so the home empire has FTL and neighbours; Empire mode starts at year 0.

| Kit | Starting state |
|---|---|
| Trader | 2,000 credits and one Freighter (civilian hull, 120 alloys, cargo 150) |
| Mercenary | 1,000 credits and 2 corvettes |
| Settler | 1,000 credits, one colony ship with its cargo, one corvette (`07 D44`) |
| Commander | Ensign in an AI-run empire 0, with a corvette (`07 D218`) |
| Leader | governor of one colony of an AI-run empire 0 |
| Scientist | captain of a new science ship at the capital (`07 D218`, the owner's call 1) |

**Transitions** happen only through play (`07 D218` call 4): Independent → Empire by founding a colony (the free company becomes an empire with a ledger and the Empire-mode screens open); Employee → Empire at High Command or Ruler (the AI ruler is dismissed and `AI_RUN` clears); Empire → Employee through the removal setting or by abdication (`Captain::Abdicate`).

### Trading, contracts and reputation (M12a)

- **Trading:** buy at `price × 1.05`, sell at `price × 0.95 × (1 − tariff)`, the external terms of `01 §7.7`; the market's empire keeps the tariff. Buys are capped at 0.2 × stock per 30 days. Prices answer through the market rules within 10 days. Delivery contracts pay `0.3 × price × amount` on top of the sale.
- **Contracts** are generated on the issuer's planning day, at most 6 open per empire:

| Contract | Pays |
|---|---|
| Bounty (a pirate fleet in sensor cover) | 300 + its power |
| Escort (a spoke with piracy > 0.3) | 200 + 2 per route day |
| Delivery (a market short within 180 days) | the bonus above |
| Survey (an unsurveyed system within 8 cells) | 150 |
| Blockade / Raid / Defend (at war) | 800 / 1,000 plus loot / 1,200 |

Taking one needs reputation ≥ 0 with the issuer, war contracts ≥ 30. Completion is detected from events the sim already emits; deadlines sit on the timer heap. An expired contract costs −5 reputation.
- **Reputation** per empire runs −100..100, drifting 1 per 30 days toward 0: +5 per contract, +20 per war contract, −20 for raiding a colony, −50 for attacking a fleet. At ≤ −30 the persona is an **outlaw** there (patrols engage, no docking); at ≥ 30 **trusted** (shipyard access, contracts, −5% prices); at ≥ 60 **favoured** (holdings in their space, a letter of marque).
- **Holdings** are persona-owned starbase rows; built: 90 days to build, a 500-unit Warehouse, a Trade Post without tariff, raids on undefended holdings. Leasing, the other modules and the tier-3 upgrade are not built (`07 D133`).

### The personal fleet (M12a, `01 §19.1`, `07 D44`)

- **The persona always has a personal fleet,** in every framework; in Empire mode it is the ruler's own fleet, formed by the two starting corvettes. Its size is capped at `4 + renown / 50` ships.
- **Per-ship fitting**, for its ships only: weapon groups, hull mods, one officer per ship, d-mods (lasting damage on salvaged or badly repaired hulls), and s-mods (hull mods built in by a story point at no slot cost). Navies use designs, so per-ship state stays bounded.
- **Templates:** a fitted ship can be saved as a template, which becomes an empire design without its officer, d-mods and s-mods.
- **Attention:** `5 + ruler skill + traditions` colonies. Each ungoverned colony beyond attention costs −2 stability on every ungoverned colony and +2 administrative capacity used. A colony under a governor does not count. The player is never forced to delegate.
- **Physical presence:** at a colony without a governor, changing its designation, issuing a planetary decree, starting a megastructure or terraforming, founding its starbase or upgrading it past size 4, and resolving its situations and crises need the personal fleet in orbit. A governed colony's governor does these remotely.
- **Colonisation needs a world:** a persona's empire with no colony cannot order colony ships or found starbases. The first world comes from a tier-3 holding upgraded into a size-1 starbase colony (300 alloys, 40 parts, 75 influence, 180 days), from conquest or purchase, or from the Settler kit.

### The character (M12a, `01 §19.2`, `07 D47`)

Skills reach three **scopes**: personal (the persona and its flagship), squad (the personal fleet) and empire (whatever the persona runs or holds a post in). Every effect is a modifier row with its tooltip.

**Progression is a hybrid.** Each skill grows by use, with ranks 1, 2 and 3 at 100, 400 and 1,200 XP of its own activity. The persona also gains levels 1–20 from all XP together, level n at `150 × n²` total. Each level gives one skill point (buys the next rank of any skill) and one story point. Rank 3 is **elite** and also costs a story point.

| Aptitude | Skill | Grows by | Rank effects (1 / 2 / 3; elite adds the last clause) |
|---|---|---|---|
| Command | Tactics | battles the personal fleet fights | squad damage +5 / 10 / 15%; empire admirals' XP +10 / 20 / 30%; elite: readiness counts +10 in combat |
| Command | Logistics | days in hyperspace | squad fuel −10 / 20 / 30%; empire supply range +1 / 2 / 3 cells; elite: no attrition at fringe depth |
| Command | Coordination | battles with ≥ 3 ships | officer slots +1 / 2 / 3; empire fleet capacity +2 / 4 / 6; elite: no over-capacity upkeep on the personal fleet |
| Command | Inspiration | victories | readiness recovery +25 / 50 / 75%; empire army strength +5 / 10 / 15%; elite: companions' loyalty never below 30 |
| Piloting | Gunnery | flagship damage dealt | flagship weapon damage +10 / 20 / 30%; elite: first volley ignores shields |
| Piloting | Helm | combat ticks on the flagship | evasion +5 / 10 / 15%, speed +5 / 10 / 15%; elite: always disengages |
| Piloting | Damage Control | hull repaired | repair +25 / 50 / 75%; a d-mod removed for 50 / 35 / 20 alloys; elite: survives one lethal hit per battle |
| Piloting | Boarding | derelicts and stashes claimed | capture after victory 5 / 10 / 15%; salvaged hulls one fewer d-mod; elite: automata caches boarded without a fight |
| Administration | Governance | days running ungoverned colonies | attention +1 / 2 / 3; stability +2 / 4 / 6 where the persona runs; elite: presence operations remote within 4 cells |
| Administration | Industry | constructions completed | build speed +5 / 10 / 15%; starbase upgrade cost −5 / 10 / 15%; elite: full self-sufficiency bonus at 65% local |
| Administration | Bureaucracy | decrees and edicts issued | administrative capacity +10 / 20 / 30; edict cost −10 / 20 / 30%; elite: one extra edict slot |
| Administration | Planning | colonies and starbases founded | colony ship cost −10 / 20 / 30%; new colonies grow +10 / 20 / 30% for 5 years; elite: founds with 2 pops |
| Trade & Intrigue | Haggling | units traded | spread −1 / 2 / 3 points; tariffs −10 / 20 / 30%; elite: buy cap 0.3 × stock |
| Trade & Intrigue | Rhetoric | deals, contracts, conversations | AI opinion of the persona +10 / 20 / 30; relation gains +25 / 50 / 75%; elite: one deal a year at score ≥ −20 |
| Trade & Intrigue | Smuggling | trades where outlaw or transponder off | sensor profile −1 / 2 / 3 cells; raiding reputation loss −25 / 50 / 75%; elite: outlaw markets still trade |
| Trade & Intrigue | Espionage | intel gathered | intel radius +2 / 4 / 6 cells; exact foreign fleet power from rank 2; elite: foreign planning-day intent visible |
| Exploration | Navigation | cells crossed at fringe depth or deeper | deep attrition −20 / 40 / 60%; storm damage −20 / 40 / 60%; elite: `cautious` routes cost no extra days |
| Exploration | Survey | surveys and deep surveys | detection radius +1 / 2 / 3 cells; `DeepSurvey` 15 / 10 / 5 days; elite: deep sites show their kind |
| Exploration | Salvage | stashes and derelicts looted | loot +25 / 50 / 75%; derelict recovery +10 / 20 / 30%; elite: caches also yield research |
| Exploration | Research | caches, anomalies, precursor sites | empire research +3 / 6 / 9% while ruler or minister; elite: precursor tech cost −25% |

Espionage also trains from FC4's operations, and Smuggling from M17d's smuggling (`07 D219`, `00-status`).

**Story points** come one per level and one per 5 caches resolved. Uses: personal (make a skill elite; recruit a companion or notable who would refuse; survive a death roll once), squad (an s-mod; recover a ship lost in battle; evade one rolled deep encounter), empire (an event option's best outcome; +20 stability on one colony for a year; one AI deal at score ≥ −20; a construction 50% sooner).

**Gear**, twelve items in three slots (`gear.json`): Implant (neural link +1 officer slot; combat cortex flagship damage +10%; longevity graft +15 years), Kit (survey rig +1 detection radius; trader's ledger spread −1 point; boarding kit capture +5%), Regalia (signet renown gain +25%; commission seal contract pay +20%; crown of office stability +3 where the persona rules).

**Renown** grows with victories (+power / 100), contracts (+5), discoveries (+10 a cache) and titles. It gates the personal fleet's size, which companions will join, and the offers below.

### Paths to power (M12b, FC3, M16)

| Path | Rule |
|---|---|
| Own colony | a holding grown into one, the Settler kit, or conquest |
| Commission | at reputation ≥ 30, paid `50 + 0.5 × renown` credits a month from the employer's ledger; the employer's enemies become the persona's; ends below 30, when the employer holds no colony, or by command (`07 D138`) |
| Vassal fief | at renown ≥ 300 and reputation ≥ 50, the empire grants its least-populated starbase system (never the capital); its colonies become the persona's under the liege's guarantee; 10% of the persona's 30-day income a month; independence by revolt (a war), buy-out (1,000 influence) or the liege holding no colony (`07 D138`) |
| Posts | in an empire that accepts foreigners (`empire.foreigners` ≥ 1: democratic +1, open immigration +1, closed −1): governor (+2 stability per Governance rank), admiral (+2 fleet capacity per Coordination rank), minister (+3% research per Research rank), councillor (oligarchies); each with a reputation and renown threshold and a salary; +1 reputation a month while held (`07 D139`). Minister and admiral posts need the coalition's consent (support-weighted coalition approval ≥ 40, `07 D217`) |
| Elections | every 5 years; score `renown + 5 × reputation + credits / 100`, won at 800. Inside a party, the score becomes a personal vote `min(score / 800, 1) × 0.5` added to the party's campaign term (`07 D217`) |
| The throne | the empire's colonies pass to the persona's empire, its fleets are rebuilt on copies of their designs, its leaders join (the ruler leaves), its treasury moves across, its wars end in white peace (`07 D139`) |

FC2 built the AI's levy on the persona's fief as `Command::Levy`, taking 10% of the persona's credits once per war of the liege, in place of `01 §19.2`'s "joining a campaign with the personal fleet" (DESIGN, FC2 calls).

### The Employee framework (M16, `07 D218`)

- **Ranks by favour:** 0 / 100 / 300 / 700 / 1,500 / 3,000 favour gives Ensign, Lieutenant, Captain, Commodore, Admiral, High Command. For the Commander and Scientist they cap the fleet at 2 / 4 / 8 / 16 / 32 / all ships. For the Leader they set the post: Governor, then governor of 2 / 4 / 8 colonies (standing in for a sector governor, since no sector system exists), Minister, Ruler. The Scientist holds the Minister post at Admiral.
- **Favour:** an assignment +20 to +60 on completion, −30 on failure, −10 per 30 days ignored; × 1.5 when it serves a governing coalition's mandate and × 0.5 against one; promotion speed × (1 + patron relation / 200) within ±50%. No other decay (call 3). `Query::Favour` gives the breakdown. The same ranks apply to a commission in a foreign empire.
- **Assignments** come from empire 0's plan, at most one at a time: Commander patrol, escort, blockade, defend (each names a system); Leader build X or raise stability; Scientist survey, research, excavate.
- Empire 0 plans on the first day of the month with no AI planner. The persona has its own purse under `AI_RUN`, paid a salary (the post's, or 20 a month for a Commander or Scientist).
- **The removal setting** (off by default; `docs/19` call 4): a coup or deposition against the persona succeeds as against an AI ruler. The persona is demoted with chance `(r + 100) / 200`, clamped to [0, 1], where `r` is its mean relation with empire 0's posted notables before the coup; demoted means Commodore (Commander) or Minister (Leader); otherwise it is exiled as a Mercenary inside empire 0's row. The setting never applies to an empire the persona founded; the Empire kit's starting empire does not count as founded (`07 D217` call 3).

### People (M12b, FC2)

- **Companions:** sixteen authored (five officers, four governors, four admirals, three scientists; six romanceable), each with a level, traits, a personality, a renown gate, a price, and likes and dislikes among event tags. Loyalty starts at 60 and moves by the tags of the persona's acts (contract, failed, victory, outlaw, trade, war, service, rule, cache). At 0 a companion leaves, an officer taking their ship. Recruited at the gate's renown or below it for a story point (`07 D140`).
- **Officers:** a leader class assigned to ships of the personal fleet; 2 slots plus `persona.officers`; each officer's skill adds 2% to the personal fleet's damage. A companion admiral in a colony's system adds 5% per skill point to the brigades' blows there (`07 D140`).
- **Relations** with foreign leaders met, at most 200 rows, drifting 1 a month toward 0; +5 per completed contract (the issuer's ruler), +1 per 20 credits of a gift (at most 25 a gift), multiplied by Rhetoric. The ruler's relation × 0.25 is an opinion term of its empire toward the persona's (`07 D142`). **Contacts** are notables at relation ≥ 30; their empire's contracts pay +20% and are listed first (`07 D216`).
- **Trainable hires:** a leader hired by the persona's empire picks one of its class's two traits at hire; the **academy** at a persona starbase of size ≥ 3 adds a chosen trait in 90 days for 100 credits × level (`07 D143`).
- **Marriage:** to a romanceable companion at loyalty ≥ 70, or (FC2) to an unmarried notable at relation ≥ 50 with a dowry paid into the notable's purse. A wedding gives +10 relation with every living member of the spouse's family, once. A child comes every 5–10 years while the spouse lives and comes of age at 16 as a leader with one inherited trait and a class chosen by the player (`07 D141`, `D216`).
- **Legacy:** the persona is mortal by default (an immortal new-game setting exists), starting at 30 with a lifespan rolled in 70–90 years. On death, Cloning (society tier 3) continues as a clone aged 25 with 75% of each skill's XP, once per 50 years; otherwise the designated heir (a child, companion or leader) continues with the fleet, holdings and fiefs, half the renown and a third of each skill's XP; elite ranks bought with story points are lost. Life Extension (society tier 2) adds 30 years (`07 D144`).

### Leaders (M11, `01 §9.7`, `07 D85`, `D86`)

Classes: Ruler, Governor, Scientist, Admiral, Expedition Leader, Officer (M12b) and Magnate (FC2); General is 1.0. A pool of 3 candidates per class per empire, levels 1–3, aged 25–45, lifespans 70–90, refreshed yearly. Hire 50 × level credits; upkeep 2 × the hiring level a month (levels gained by use cost nothing), +10% each beyond the cap of 6. XP 1 a day while assigned; level L → L+1 at 360 × L. Old age from 60 with a yearly 2% chance of an early death. Traits are modifier rows that follow the leader's post (for example Governor Educator `pop.skill_gain` +25%, Administrator +3 stability). A death fires an event whose `successor` option hires the best candidate into the post. AI empires never hire leaders; generated unpaid notables fill their heir and council seats up to 8 (cap 12) (DESIGN, FC2 calls).

### Macro layer: parties, blocs and classes (FC1, `01 §9.4`, `07 D214`)

FC1 replaced M15's faction sketch. Numbers live in `government.json` `politics`.

- **Blocs.** Each cohort drifts monthly (0.1 of the way) toward the material bloc of the jobs its stratum works on its colony: **Mining, Logistics, Researchers, Bureaucracy, Soldiers**. Its unemployed share drifts back to **unaffiliated**; jobs with no bloc keep pops unaffiliated.
- **Wealth class** (the owner's addition, `07 D241`): a colony's private wealth is held 5 / 25 / 70% by workers, specialists and rulers, at a 5% return a month against wages of 2 / 4 / 6 a pop. The owner share is `return / (return + wage)`; the rest is wage-earner.
- **Parties.** A party is an ethic pole plus a base, a bloc or a class (8 poles × 7 bases). Unaffiliated pops count through their class. A cohort's weight is `count × voteWeight`. Up to six cells with ≥ 10% of the empire's weight become parties; a party dissolves after 12 months under 5%.
- **Who counts** (by authority):

| Authority | voteWeight | Elections | Agitation rate | Suppression price × |
|---|---|---|---|---|
| Democratic | 1 per pop | every 5 years | 0.5 | 2.0 |
| Oligarchic | strata 0.25 / 1 / 4 × (1 + wealth per pop / 100) | every 10 years | 0.75 | 1.0 |
| Autocratic | strata 0 / 0.25 / 1, plus members' clout (FC2) | none | 1.0 | 0.75 |

- **Demands.** Each party has 4–5 demands, predicates over policies, edicts and empire stats; its base's first demand weighs 2. Class demands touch rents and returns: owners want taxation ≤ Standard, no Levy for 10 years, free trade; wage-earners want living standard ≥ Standard, unemployment < 5%, the Levy used or taxation ≥ High.
- **Approval** moves a quarter of the way each month toward `20 + 60 × demands met`, +10 in the coalition, −15 suppressed, +5 promoted, plus the leader terms below.
- **Effects.** Members' happiness ± (approval − 50) / 5. Influence `3 × support × max(0, approval − 50) / 50` per 30 days.
- **Elections:** vote share `support × (0.8 + approval / 250) × (1 + campaign) × leader charisma`; the largest parties form a coalition of half the vote. Each coalition party's core demand is a **mandate**: +5 approval and +25 influence if met at term end, −10 if missed. The persona stays head of state; the coalition's largest party's leader becomes **chancellor** (+2 clout).
- **Levers:** suppress or promote (20 or 10 influence a year per percent of support, × the authority multiplier); embrace (one ethic point toward the party's pole for 100 influence, with a reform-style cooldown); concede a demand by changing the policy. The AI concedes one demand a plan to its most-supported agitating party, and suppresses only when it cannot concede.
- **Agitation.** Under 30 approval for 24 months, a party agitates: a `faction_agitation_<base>` situation with stage effects at 25 and 50 (for example Mining: miner output −25% at 50), progressing `(30 − approval) / 3 × rate` a month. FC4 adds a stage at 75 that starts the party leader's plot.

### Micro layer: notables (FC2, `07 D216`)

- **The roster:** 8–12 unique people per empire, recomputed on its politics day: the ruler, the heir, up to 5 council seats (the highest-clout hired leaders), the two largest fleets' admirals, governors of the three largest colonies, 1–3 magnates, and each party's leader.
- **Magnates** are a leader class with no post and no wage: mining baron (Mining), merchant prince (Logistics), shipyard dynasty (Soldiers). Their wealth is read as a share of their home colony's private wealth.
- **Columns:** two of 16 personality traits (Ambitious, Loyal, Greedy, Honourable, Zealous, Cautious, Vengeful, Pragmatic, Charismatic, Corrupt, Reformer, Traditionalist, Brave, Paranoid, Generous, Schemer), a pole and a bloc, an ambition with a target and a deadline, wealth, clout, loyalty, up to 4 bonds (friend, rival, spouse, patron, client), and a family (the house, for everyone).
- **Clout** = posts (ruler 5, heir 2, council 2, admiral and governor 1) + wealth / 1,000 + the admiral's fleet-power share × 10.
- **Ambitions**, stepped every 90 ± 30 days on a seeded timer: win a post, get rich, pass a demand, ruin a rival (FC2), plus become ruler, avenge a death, gain a fief (FC4). One step per fire: a met ambition gives +10 relation to whoever helped; a blocked one adds a rival bond (or, for a Schemer, Vengeful or Ambitious notable under 40 loyalty, a plot); otherwise lobby, seek a post, offer the persona a quest, or save.
- **The persona's actions** (`Captain::` commands): converse (same system, or a post in their empire; reveals traits, and ambition at Rhetoric 1), gift, personal quest (relation ≥ 0; success gives relation, a favour owed and party approval), call in a favour (−3..+3 per relation), patronage (relation ≥ 30; a client bond), recruit (their loyalty < 40, relation ≥ 30, renown ≥ clout × 10), marriage, betray (−50 relation with them and their family, −20 reputation with their empire). A notable whose clout × 10 exceeds the persona's renown will not deal personally without a post, a contact or a story point.

### The bridge: leaders, campaigns and conflict (FC3, `07 D217`, `D271`)

- **Party leaders** change only at the yearly **party congress** or on the leader's death or departure. Leader terms on approval: trait modifiers (Charismatic +5, Corrupt −5), the leader's opinion of the ruler `(loyalty − 50) / 5` (±10), Charismatic × 1.1 vote share, Zealous core demand weight 3, Pragmatic 3 of 4 demands counts as all. A leaderless party takes −10 approval for a year.
- **Campaigns:** `campaign = 0.1 × ln(1 + spend / 500)`, where spend is the term's patronage paid to the party's leader and members plus 5% of each member magnate's wealth on election day.
- **The persona and parties:** join (with a post, fief or commission there, or in its own empire; other parties' leaders lose 10 relation); lead at congress when its clout (renown / 10 + posts + fiefs + credits / 1,000) exceeds the leader's, then naming the mandate and taking the party's unmet demands as quests; found (renown ≥ 300, in an empire that accepts foreigners or its own; starts at ≥ 5% support; dissolves after 24 months under 5%).
- **Endings at 100**, after FC1's step and `07 D271`'s ladder (a 20-year grace in which an ending takes only FC1's step; the FC3 step needs a second ending within 10 years of the first):

| Authority | First step (FC1) | Escalation (FC3) |
|---|---|---|
| Democratic | snap election | if still under 30 approval, colonies where the party holds the majority and stability is under 30 secede |
| Oligarchic | party forced into the coalition, −10 stability for a year | the council deposes the ruler when ≥ 3 councillors are members or rivals of the ruler; otherwise secession |
| Autocratic | Planetary Unrest on the party's majority colonies | a coup that succeeds when aligned admirals command ≥ 50% of fleet power; otherwise civil war |

- **Secession and civil war:** a rebel empire takes one of 8 reserved rebel rows (beyond the cap, a 10-year autonomy modifier: −50% of tax to the parent, +10 stability). Colonies transfer one a day; fleets of party admirals under 30 loyalty are rebuilt for the rebels. The war's goals are **reunify** and **independence**. A war of independence with no change for 3 years ends in the rebels' white peace. A rising never takes the parent's capital. Against the persona, a coup or deposition becomes a civil war unless the removal setting applies; the persona's personal fleet is always loyal.
- **Succession wars** (AI autocracies): the heir takes the throne, and a claimant (family, or a rival of the heir) with clout ≥ the heir's may press a claim as a civil war.
- **War-hawk weight:** `hawk = Σ coalition vote share × pole sign × 0.25`, × 0.5 / 1 / 1.5 by authority (Militarist +1, Pacifist −1). The AI declares war at power ratio `war / (1 + hawk)`. Brakes: every non-Militarist party loses 1 approval per 10 war exhaustion; a mostly Pacifist democratic coalition offers white peace at exhaustion 50.

### Intrigue and lords (FC4, `07 D219`)

- **Plots** ride the plotter's ambition step: coup and impeachment against the ruler, assassination of a notable, secession for the plotter's party. Progress per step `10 + 2 × clout + Σ allies' clout`, × 1.5 Schemer, × 0.5 Cautious. Discovery 5% per step, +10% under a Paranoid ruler, +5% per 25 progress. At 100 a plot fires FC3's ending. The player sees plots from progress `75 − 25 × Espionage rank` in empires where it has a foothold.
- **Plots against the persona** (the owner's rule): progress × 0.5; at 100 a kill roll of `35% − 5% × Espionage rank`; a kill runs the clone or heir path; a failed kill exposes the plot.
- **Operations**, one live at a time per actor, for the persona and for AI empires (owner's call 1B):

| Kind | Espionage rank | Effect |
|---|---|---|
| Expose a plot | 1 | it fails; +1 favour with the target |
| Blackmail | 1 | a favour owed by force, −30 relation |
| Sabotage | 1 | −25% job output at a colony for 90 days; one at a time per colony, then 90 days' immunity |
| Rig | 2 | the party's next vote share × 1.1 |
| Steal tech | 2 | 50% of the tech's cost |
| Hostage | 2 | a notable held for ransom, a favour or release |
| Assassinate | 3 | the target dies; its kin may seek vengeance |

Success is the kind's base + 10% per rank − 10% against a Paranoid target; exposure `30% − 5% × rank`, rolled whether or not it succeeded. An exposed persona loses 30 reputation with the target's empire. AI empires run sabotage, steal tech, rig and assassinate (never the persona), with a rank from their era; the player's defences lower success by 5% per Espionage rank and 10% under a Paranoid ruler. Hostages can also be captured in battle (50%) when the personal fleet destroys a notable admiral's fleet. There are no duels.
- **The Feudal civic and lords:** autocratic or Feudal empires grant governor notables and magnates one fief each, up to one per 8 colonies (at most 3), never the capital. A lord keeps 0.9 of the fief's tax (owes 10%), holds a **house guard** of 2 corvettes that counts on its side in a coup, pays a levy of its wealth at war, and may defect with the fief to a stronger rival at loyalty under 20. Under autocracy or Feudal, a house's head is its highest-clout member, who inherits a dead notable's party, half the wealth, rival bonds and fief.
- **Ideology spread:** a party at ≥ 60% approval for 12 months pushes its pole by 1% a month (cap 3% per pole) into the cohorts of empires with a commercial pact with its own, shown in `Query::Politics`.

### How the two layers connect

| Persona act | Macro result |
|---|---|
| A notable's quest completed | +approval for their party (shown in the tooltip) |
| Patronage paid | a client bond, campaign spend in elections |
| Joining, leading or founding a party | the persona's personal vote and mandates |
| Marriage into a family | +10 relation with every living member |
| Exposing a plot | the plot fails; −15 approval for the plotters' party, +10 loyalty for the target's |
| Serving the coalition's mandates (Employee) | favour × 1.5 |
| Operations (rig, sabotage, assassinate) | election swings, output loss, deaths that start vengeance |

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Player character and modes | One persona in three frameworks; Independent in MVP, Employee in 1.0 | 2026-11-02 | `07 D31` |
| One game | Fleet and empire in one game; delegation by soft penalty and presence; first world from a holding or Settler kit; per-ship fitting with templates | 2026-11-12 | `07 D44` |
| The character | Skills by use plus points, story points at three scopes, gear, renown; every path to power; companions, relations, trainable hires; heir, life extension, cloning, marriage | 2026-11-13 | `07 D47` |
| Factions | Ethic pole + material bloc + leader alignment; five blocs; who counts by government; persona stays head of state with an optional removal setting; elections 5 / 10 years; 8–12 notables per species list plus the owner's authored ones; family tracking for all, formal houses under autocracy or Feudal; dark actions; NPC-driven wars; at most 8 rebel empires; marriage with notables | 2026-11-18 | `07 D241` |
| Ownership axis | A wealth class (owner or wage-earner) beside the blocs | 2026-11-18 | `07 D241` |
| Escalation ladder | A 20-year grace and a 10-year ladder before coups and civil wars; rebels' white peace after 3 stalled years | 2026-11-20 | `07 D271` |
| Employee modes | A Scientist kit; removal by a reputation-scaled chance of demotion or exile; no favour decay; abdication | 2026-11-21 | `07 D218` |
| Intrigue | AI empires run operations; plots can kill the persona at lower odds; no duels (replaces `docs/19 §4.5`'s duels); hostages by operation and capture; ideology spread now; lore hooks now | 2026-11-21 | `07 D219` |
| Lord's tax | A lord owes 10% of the fief's tax | 2026-11-21 | `07 D219` (spec call A) |
| Fanatic parties | Left to 1.x (no effect specified) | 2026-11-20 | `07 D217` call 7 |
| Persona mortality | Mortal by default, with an immortal setting (replaces `01 §19`'s "exempt from the yearly death roll by default") | 2026-11-13 | `07 D47`, `D144` |

## Not yet built or still open

- **Ethics drift** (M15's remainder) and the **eight-ethics rename** (CV1); parties use the four built axes ([Civilisations and species](Design-Civilisations-and-species)).
- **Fanatic parties** are left to 1.x.
- **The owner's 20–30 authored characters:** the hooks exist; no characters ship until she writes them.
- **Not built inside the shipped slices:** holdings leasing, other modules and the tier-3 holding upgrade (`07 D133`); the persona's own brigade with a death roll (`07 D145`); lending a house guard by a favour; replacing a destroyed guard (`07 D219` calls 24–25); a defection test (`07 D219`).
- **The persona's species, birthright traits and origins** wait for SP1 and CV1 (`07 D230`).
- **Stale text to fold:** `01 §19` still says the persona "is exempt from the yearly death roll by default" and lists employee kits as commander and leader with the Scientist inside the Leader kit; `01 §9.5` still gives election intervals of 10 and 20 years; `docs/19` still describes duels and the draft blocs (Labour, Industry, Commerce, Rentiers, Intelligentsia), superseded by `07 D241` and `D219`.

## Discrepancies noted

- **The Commander kit's ships.** `01 §19` Start: "Commander (1.0) an Admiral at level 1 with 2 corvettes and 20 credits per 30 days salary". `07 D218` Builds: "The persona starts in it at Ensign with a corvette (Commander)". The salary matches (20 a month, `07 D218` call 8); the number of corvettes does not, and the build record does not restate it.

## Sources

- 01 §19 Player character and modes (with §19.1 and §19.2), 01 §9 Civilisational progression (§9.4, §9.7)
- 19 Factions and characters: §3 macro layer, §4 micro layer, §5 the bridge
- Research: Starsector and Mount & Blade
- 05 M12a, M12b and the 1.0 table
- 07 decisions: `D31`, `D44`, `D47`, `D85`, `D86`, `D130`–`D145`, `D214`, `D216`, `D217`, `D218`, `D219`, `D241`, `D271`
- DESIGN.md (FC2–FC4 autonomous calls, change log)
- 00-status.md (merge records for M12b, FC1–FC4, M16)

## Related pages

- [Overview](Overview)
- [Progression](Design-Progression)
- [Civilisations and species](Design-Civilisations-and-species)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Events and crises](Design-Events-and-crises)
- [AI and victory](Design-Ai-and-victory)
- [Ships and fleets](Design-Ships-and-fleets)
- [Starbases](Design-Starbases)
- [Trade and logistics](Design-Trade-and-logistics)
- [Flight view](Design-Flight-view)
- [Ground combat](Design-Ground-combat)
- [Pops and society](Design-Pops-and-society)


---

# Diplomacy and war

**Status:** in progress, as of 2026-11-24. M10 (opinion, treaties, claims, wars, peace, the war AI), M18a (defensive pacts, alliances, migration treaties, calls to war) and M18b (vassalage) are built, with M17b's embargo; M18c (federations) is built on track C and waits for its merge; M18d–e (federation laws, intel) are designed on the owner's answers and wait for the build.

Diplomacy is how empires, and the persona, deal with each other: an opinion that every party can read term by term, treaties that link trade networks and research, claims and wars over systems, and peace terms that move colonies. Wars are paid for by the economy and bounded by exhaustion, so no war runs forever. Late in the design come alliances that join each other's wars, vassals on negotiable terms, and federations that vote by fleet, diplomatic and economic weight.

## How it works

### Contact first

Diplomacy needs comms. Sensing an unknown empire's ship or starbase starts a 60-day contact project per empire pair; its completion opens comms both ways and fires `on_contact` for both sides (M11, `07 D85`; [Events and crises](Design-Events-and-crises)).

### Opinion (built, M10)

`opinion(A→B)` lies in [−1000, 1000] and is the sum of named terms. It is computed when read (an AI plan, `Query::Opinion`), never stored or decayed by a daily pass (`07 D111`).

| Term | Value |
|---|---|
| Ethics alignment | 60 × the dot product of the two empires' ethics points |
| Border friction | −20 per own system with one of the other's within 6 cells, down to −100 |
| Non-aggression pact | +20 while it holds |
| Commercial pact | +30 |
| Research agreement | +15 |
| Open borders | +10 |
| Defensive pact | +40 (M18a, `07 D279` rule 1) |
| Rivalry | −100 toward the declarer |
| Guarantee | +30 from the guaranteed toward the guarantor |
| Trust | +1 per 30 days since the oldest unbroken treaty, up to +100 |
| Insulted | −50, recovering by 1 per 30 days |
| At war | −50 |
| Cancelled treaty | −30 for 120 months |
| The Lance | −100 with every knowing empire (`07 D80`) |
| Embargo (M17b) | −50 with the target, −10 with its partners |
| Caught spying (FC4) | −30, decaying by months |
| Ruler's relation with the persona | × 0.25 (`07 D142`) |

Attitude buckets: hostile < −200, unfriendly < −50, neutral, friendly > 50, cordial > 200 (`01 §11.1`).

### Treaties and actions

**Built in M10** (`01 §11.2`, `07 D111`): non-aggression pact; commercial pact; research agreement (+10% research on all three fields for both); open borders; declare rivalry (+1 influence per 30 days for the declarer); insult; guarantee independence (the guarantor joins a war on the guaranteed); declare war. Treaties are ledger bits per other empire.

- **Commercial pact:** opens the trade network between the parties; each side's tariff comes from its own trade policy. Pact imports need credits: an empire with no credits imports nothing across a pact (`07 D111`). A pact supplies a deficit only when no domestic source in reach covers it, and pact imports over 30 days stay under 0.25 of the importer's 30-day income (`07 D236`).
- **Closed borders:** a fleet may not be ordered into a foreign system without open borders or a war (`Reason::Closed`). Hyperspace between systems stays open. Cancelling open borders sends the other's fleets inside home.
- **Allied Gateways:** a fleet may cross between its own Gateway and that of an empire it holds open borders with (`07 D114`).
- **War closes migration** between the belligerents (`border` 0).
- **Embargo** (M17b, `07 D275` call 6): closes a commercial pact for 5 years and bars a new one; −50 opinion with the target and −10 with its partners. The AI embargoes its rivals.

**Built in M18a** (`07 D279`, merged 2026-11-23):

| Treaty | Rule |
|---|---|
| Defensive pact | the defender calls its partners to war; AI acceptance needs a shared rival, or a threat of at least `defenceNeed` 1 (the strongest bordering empire's fleet power over the pair's) |
| Alliance | a defensive pact that also calls the partner to offensive wars, shares sensor cover and opens borders; needs a defensive pact (read on the pair's trust clock) for 5 years and opinion ≥ 100; breaking it costs −200 with the partner and "Treaty breaker" −50 with every empire that met both |
| Migration treaty | `border` reads 1 between the two empires, so pops move as within one empire; AI acceptance from pop pressure (unemployed share) |

**Calls to war** (the owner's answer 1): a defender calls its defensive-pact partners and allies; an attacker calls its allies. Refusing a defensive call costs −50 opinion with the caller; refusing an offensive call costs −10. An AI answers at the declaration; the player answers in the diplomacy screen within 30 days, and an unanswered player call joins a defensive war and refuses an offensive one. Only the war's two leads call, so one declaration cannot chain across the galaxy. A partner tied to both leads answers the defender.

### AI acceptance and proposals

The acceptance score is `opinion / 10 + Σ dealValue × personalityWeights − trustRisk`; the AI accepts above 0, and the UI shows the breakdown with no hidden rolls (`01 §11.2`). Deal values 2 / 3 / 2 / 1; risk 2 for open borders and 10 when the proposer broke a treaty with it; the defensive pact's content value −3 and the alliance's −2 (`07 D111`, `D279`).

The AI's diplomacy advisor runs on its planning day (`07 D116`): peace first (status quo, then white peace, when its own acceptance is above 0); one treaty a plan from those its personality values most; one rival, the contact it likes least once under −50; war by its personality's power ratio. **Commercial pacts:** the AI proposes one only from year 25, only when both sides hold at least 2 trade hubs and meet the budget's reserve (`07 D242`), and accepts one only from its reserve (`07 D237`).

### Claims and war (built, M10)

- **Claims** cost `50 + 25 × claimsHeld` influence, halved on systems within 4 cells of an own system. They lapse at peace (`07 D112`).
- **Declaring war** needs comms, no NAP, no truce, and for conquest a claim on the target. The target's guarantors (and from M18a its pact partners and allies, by their answers) join; treaties across the sides end.
- **War goals:** conquer claims and humiliate (MVP); reunify and independence for civil wars (FC3, [Factions and characters](Design-Factions-and-characters)). Liberate, vassalise and end threat are designed (M18b).
- **Occupation is conquest during a war:** invasion and capture transfer a colony, and the war records each colony taken with its owner before. A colony retaken by its owner ends its occupation.
- **Starbase capture:** a starbase at hull 0 in a battle with an empire at war with its owner passes to that empire (the starbase, its system and its starbase colony), at 10% hull, once no armed ship hostile to the captor still fights in that battle (`07 D113`, `D149`). Other colonies in the system still need an invasion ([Ground combat](Design-Ground-combat)).
- **The Lance:** a strike on an empire's fleet or starbase while not at war with it declares war on it, to humiliate (`07 D115`).

**War score** per side, computed when read, within 0–100: +10 per occupied claim, +2 per battle won (up to 30), and 40 × the enemy ships destroyed over its ships at the start (up to 40) (`07 D112`).

**War exhaustion** per side, accrued daily only while a war stands, capped at 100:

```
+0.5 per 30 days
+ 20 × ships lost / ships at the start       (at each battle's end)
+ 30 per 30 days × pops occupied / total pops
```

The player at 100 takes −10 happiness until peace. **Both sides at 100 end the war in the status quo**, which bounds a war at about 17 years when nothing else ends it; on the host, no war lasted past 16.7 years (`07 D112`, `05 M10`). From FC3, every non-Militarist party loses 1 approval per 10 exhaustion, and a mostly Pacifist democratic coalition offers white peace at 50.

**Peace terms:**

| Term | Effect |
|---|---|
| Status quo | each occupied colony in a system the holder claimed stays; the rest return |
| White peace | everything returns |
| Surrender | the offering side also hands over every system the other side claims of it |

A 10-year truce follows. The receiver's acceptance is `own exhaustion − other's + other's score − own score`, plus 0 / 0 / 100 for the three terms; an exhausted side accepts any (`07 D112`).

### The AI at war (built)

- **War target:** after the First Jump and at peace, the weakest contact it outguns by its personality's ratio (`ai.json` `war`: militarist 1.0, expansionist 1.5, industrialist 2.0, trader and scholar 3.0), under +50 opinion, with no NAP or truce. It claims the target's two systems nearest its capital while a claim's influence stays in reserve, and declares to conquer once it holds a claim (`07 D116`). FC3 divides the ratio by `1 + hawk` from the governing coalition's war-hawk weight; a `war` of 0 still never declares. From M18a, a target's defensive-pact partners' fleet power counts with its own.
- **War orders:** at war the target fleet power is half as much again; the strongest idle fleet attacks the enemy system nearest the capital, a claimed one first, once its power is twice what stands there.
- **Invasions** go through ground combat's front: transports load `force_needed × 1.5` and invade the enemy colony needing least force that is reachable and uncontested (`07 D116`).
- **The navy** buys its parts at the yard within the reserve and builds warships at war or when under half its target power.

On the host, the first war came in years 10.5, 2.7 and 35.5 on seeds 1–3 (`05 M10`).

### Designed, not built: M18b–e (`07 D279`)

**M18b, vassalage** (about 950 lines):
- A vassal contract has four terms: **tax** (0, 5, 10, 15 or 25% of the vassal's monthly income), **levies** (none; defensive; all), **diplomacy rights** (full, or none: treaties only with its liege and the liege's federation) and the **subject type**.

| Subject type | Tax | Levies | Diplomacy | Perk |
|---|---|---|---|---|
| Protectorate | 0% | none | none | techs its liege knows cost it 25% less |
| Tributary | 15% | none | full | none; buys peace with credits |
| Vassal | 10% | all | none | its fleet power counts toward its liege's in AI threat estimates (the persona's fief terms, `07 D138`) |

- A vassal cannot declare war except for independence; the liege may not declare on it; an attack on a vassal calls its liege, whose refusal frees it. Each term harsher than none costs the vassal's opinion of its liege ("Terms" −10 per tax step, −10 per levy step, −15 for no diplomacy).
- **Renegotiation:** either side proposes new terms once per 10 years per pair, accepted by score; a liege may instead impose them for 100 influence and "Terms imposed" −50 opinion decaying over 10 years (the owner's answer, A).
- **Offers:** an AI with ≥ 3× a bordering empire's fleet power offers the subject type worth most to itself whose acceptance score is above 0; the player offers and receives offers the same way.
- **War goals:** vassalise (surrender makes the loser a Vassal), liberate (frees a vassal of the target), end threat (disbands the loser's fleets above half its pre-war power); vassalise costs influence like a claim.
- **Independence:** a vassal whose fleet power exceeds its liege's ÷ 1.5 and whose opinion is under −50 declares a war of independence; or buys out for 1,000 influence; or goes free when its liege is destroyed, refuses its defence or becomes a vassal itself.
- Domination victory will count vassals' pops ([AI and victory](Design-Ai-and-victory)).

**M18c, federations** (about 750 lines): founded by two empires holding an alliance; others join by the president's invitation and a weighted vote; up to 6 members. The president rotates every 10 years to the member with the highest contribution over the term. A member's vote weight is `(fleet_i ÷ Σfleet + dipl_i ÷ Σdipl + econ_i ÷ Σecon) ÷ 3`, where diplomatic weight is `1 + treaties held + Σ max(0, opinion of it) ÷ 100` and economic weight the monthly credit income. More than half the total weight carries a vote. Wars, laws, invitations and expulsions vote this way; an attacked member calls every member. As built (`07 D279`, M18c's build calls): AI members vote at the motion and the player's vote waits 30 days (unanswered is no); only the president moves; a carried war binds every member; dues are 2% of income from the founding; a fallen member or a foreign liege's subject leaves at no cost, and under two members the federation ends. The federation is a section of the diplomacy screen.

**M18d, laws** (about 600 lines), each voted once per law per 10 years:

| Law | Effect |
|---|---|
| Centralisation | Loose / Federal / Union: dues ceiling 2 / 5 / 10%, leaving costs −50 / −100 / −200 opinion; at Union no treaties outside the federation and the president may declare a defensive war without a vote |
| Free trade | no tariffs between members; their hubs link as pact trees do |
| Shared sensors | every member sees what every member senses |
| Common migration | a migration treaty between every pair |
| Shared research pool | a tech any member knows costs each other member 25% less |
| Common currency (Federal) | the members' price indices merge; imports between members pay at ratio 1 |
| Shared fleet (Federal) | dues and ship donations fund a fleet owned by the president's empire, commanded in every federation war, passed on rotation, returned to donors on repeal |

**M18e, intel** (about 300 lines), for the player only (the AI reads the truth): level 0 unknown, 1 contact (name, capital, colony count, fleet strength as a band), 2 relations (any treaty other than rivalry, a vassal tie, or a foothold: exact fleet power, economy and tech counts, treaties), 3 deep (alliance, federation, liege–vassal, or Espionage rank 3 with a foothold: parties, plots, planning-day intent). Espionage adds one level from rank 2 with a foothold.

### RTS and TB

Every act is a command applied on the next tick or turn. An AI answers a proposal at once; its offers and peace terms wait for the player in the diplomacy screen. AI proposals, calls, votes and renegotiations run on planning days; tax, dues and the federation treasury on the economic day. In TB the player's 30-day answer windows span turns, and the turn report lists treaties, calls, votes and payments (`05 M10`, `07 D279` rule 16).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Opinion and treaties as built | Opinion computed when read with named terms; treaties as ledger bits; pact imports need credits; closed borders | 2026-11-14 | `07 D111` |
| Wars | Wars as rows; score and exhaustion computed per `01 §11.3`; both sides at 100 end in the status quo; 10-year truce | 2026-11-14 | `07 D112` |
| Starbase capture | A starbase at hull 0 passes to the hostile empire at 10% hull (`D113`), once no armed hostile ship still fights there (`D149`) | 2026-11-14, 2026-11-16 | `07 D113`, `D149` |
| Allied Gateways | Read as open borders until alliances | 2026-11-14 | `07 D114` |
| AI commercial pacts | From year 25, with 2 hubs and the reserve met on both sides (replaces `07 D116`'s "no commercial pact of its own") | 2026-11-18 | `07 D242` |
| Pact trade protects the importer | Domestic sources first; pact imports capped at 0.25 of 30-day income | 2026-11-17 | `07 D236` |
| Embargo | Closes a pact for 5 years, bars a new one; −50 / −10 opinion | 2026-11-21 | `07 D275` call 6 |
| Alliances and calls | Alliances join offensive wars too; refusing an offensive call −10, a defensive one −50 | 2026-11-22 | `07 D279` answer 1 |
| Vassal terms | Negotiable (tax, levies, diplomacy, subject type); offers by the strong; three subject types; propose or impose | 2026-11-22 | `07 D279` answers 2–3, vassal choices A and A |
| Federations | Shared fleet as a policy funded by dues and donations; votes by fleet, diplomatic and economic shares (replaces `01 §11.4`'s "weights by fleet power"); the larger law set | 2026-11-22 | `07 D279` answers 4–6 |
| Intel | For the player only | 2026-11-22 | `07 D279` answer 7 |
| War-hawk weight | A governing coalition's Militarist or Pacifist lean scales the AI's war ratio | 2026-11-20 | `07 D217` |
| Starbase defeat (designed) | Capture stays; every fleet of the losing empire in the system retreats at capture | 2026-11-23 | `07 D284` call 3 |

## Not yet built or still open

- **M18d–e:** federation laws and intel. The research pool waits for the tech tree's cost function (`07 D278`).
- **`tools det`'s scripted federation war** (`05 M18`'s acceptance).
- **Defensive pact, alliance and federation bench rows:** a declaration with a full federation on each side, and the AI plan max with alliances, vassals and federations formed, within 5% of main (`07 D279`).
- **The station rework's retreat at capture** (`07 D284`) is designed; its milestone is not yet placed ([Starbases](Design-Starbases)).
- **1.0 first contact events** with choices (hostile, cautious, open) are not built (`01 §4.2`).
- **Espionage beyond FC4's operations** is M18e's intel.
- **Stale text to fold:** `01 §11.2`'s 1.0 list still names defensive pact, alliance and migration treaty (built in M18a) and embargo (built in M17b); `01 §11.4` still weights federation votes by fleet power; `07 D146` still says "only a player with commercial pacts" can win economically, written before `07 D242` let AIs propose pacts; DESIGN's pillar row still reads "designed (M10)" after M10's merge on 2026-11-14.

## Discrepancies noted

None found.

## Sources

- 01 §11 Diplomacy and war, 01 §4.2 Interstellar exploration (first contact)
- 05 M10, 05 1.0 milestones (M18)
- 07 decisions: `D80`, `D85`, `D110`–`D116`, `D142`, `D149`, `D217`, `D219`, `D235`–`D237`, `D242`, `D275`, `D279`, `D284`
- 19 Factions and characters (§6 war load)
- DESIGN.md (M10 autonomous calls, change log)
- 00-status.md (merge records for M10, M17b, M18a)

## Related pages

- [Overview](Overview)
- [Factions and characters](Design-Factions-and-characters)
- [AI and victory](Design-Ai-and-victory)
- [Combat](Design-Combat)
- [Ground combat](Design-Ground-combat)
- [Starbases](Design-Starbases)
- [Trade and logistics](Design-Trade-and-logistics)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Pops and society](Design-Pops-and-society)
- [Events and crises](Design-Events-and-crises)


---

# Events and crises

**Status:** in progress, as of 2026-11-23. The events engine, the MVP's 40 events, three situations, archaeology, first contact and the precursor chain are built (M3, M6, M11); faction agitation situations come from FC1; pirates are the one crisis in play; the marauder, awakened-empire and endgame crises are designed for M20.

Events are the narrative glue of the game: anomalies, digs, first contacts, leader deaths, era transitions and hyperspace incidents all run through one data-driven grammar of triggers, conditions, options and effects. Situations are progress bars a player steers with an approach and a price. Crises are galaxy-scale threats that arrive with the later eras and test every empire at once. The rule throughout is that only a real choice interrupts the player, and nothing tests which clock (real time or turns) is running.

## How it works

### The event grammar (built)

Events are data (`01 §12.1`; schema in `06-data-schemas.md`). Each has:

| Part | Content |
|---|---|
| Trigger | what can fire it; no clock-specific triggers exist |
| Scope | empire, planet (colony), fleet, leader, system, hex |
| Conditions | an expression tree |
| Timing | mean time to happen (MTTH) in days, or a fixed probability |
| Text and options | 1–4 options, each with conditions, costs and effects |

**Triggers built:** `on_survey`, `on_project_complete` (M3); `on_hex_survey` (M6); `on_colonize`, `on_tech`, `on_contact`, `on_leader_death`, `on_battle_end` (for each empire side) and `on_daily_mtth` (M11); `on_hyperspace` (F1); `on_fired` for chained events (`07 D85`).

**Conditions built:** `all`, `any`, `not`, `flag`, `projectKind`, `stat`, `random`, `feature`, and the `hex.*` stat keys. The tech rework adds stats read from existing state for deeds: `empire.colonies`, `empire.systems`, `empire.contacts`, `empire.pacts`, `empire.era`, `empire.ktype` (built in T2's first slice), `empire.structures`, `empire.hubs`, and `has` for a tradition tree, perk or civic (`07 D278 §4`).

**Effects built:** `ledger`, `flag`, `event` (start another), `reveal` (M3); `stock`, `hex` (M6); `situation`, `leader` (xp, trait, loyalty, successor), `clue` (M11). `01 §12.1`'s list also names `add_modifier`, `spawn_fleet`, `change_ethics` and `kill_leader`; the M3–M11 build records do not say which of those are built. The `tech` effect (`{ "tech": "progress", "id", "points" }`) is specified in `06 §11` and will be built in T2 (`07 D278 §5`).

**MTTH.** Rolled daily per scope with `p_day = 1 − 2^(−1 / mtthDays)`. As built, the roll is a seed hash of the event, the scope and the day, so no random stream moves and only a hit evaluates the conditions; MTTH events take empire and colony scopes only (`07 D85`).

**Popups and timeouts.** Only events with a real choice interrupt; everything else goes to the notification feed. Hidden events (no popup) drive flavour outcomes. In TB, choices appear in the turn report. In both modes an option's effects apply on the tick after it is chosen, and an unanswered event takes its default option after `timeoutDays` (`01 §12.1`, `§15`). Event options whose conditions fail show greyed with their reason (M-QA2, `00-status`).

**Content.** The MVP holds 40 events: 10 anomaly events (M3), the feature events of hex surveys (M6: ruins and its dig, alien life, crash site, monolith and its hum, hazard, ore vein), two era events (M7), and M11's 20 events in `events/people.json` (`07 D85`). F1 adds 8 hyperspace incident events (below). `datacheck` refuses an MTTH event without days or with the wrong scope, an `on_fired` event nothing fires, an unknown situation, and missing strings (`05 M11`). 1.0 plans 300 events (`05` M22).

**`on_tech` events.** The trigger exists, and no content used it before the tech rework. `07 D278 §7` plans about a dozen with choices: an AI-rights chain after Sapient AI, a containment failure, a precursor signal answered.

### Anomalies (built)

A body survey rolls an anomaly at p 0.18 (× 1.5 in a system with a research-related modifier). Anomaly level 1–10 is weighted toward low early; the ship's scientist runs a research project of `20 × level / (1 + 0.1 × skill)` days, whose completion fires a weighted event from that anomaly's pool: a small resource, a deposit reveal, a tech boost, a chain start, a special project. There is no failure outcome; bad outcomes exist but are rare and event-driven, not a dice fail (`01 §4.2`).

### Archaeology (built, M11)

- Surveying a site's body gives the empire chapter 1 as a project.
- A science ship digs each chapter for `30 × difficulty / (1 + 0.15 × skill)` days, then rolls `d10 + skill ≥ difficulty` as a seed hash. A failure digs the chapter again with no reward; a success fires its chapter event and opens the next.
- The last chapter marks the site dug for that empire and pays a relic, tech or event chain (`01 §4.2`).
- `01 §4.2` specifies 3–6 chapters at difficulty 2–6. **As built** (`07 D86`): 3 chapters at difficulty 4, so a skill-3 scientist digs 3 × 83 = 249 days, the roll always passing.
- A dug site sets a flag that gates techs such as Xeno-archaeology (`07 D278 §4`).

### First contact (built, M11)

The sensor recovery pass starts a 60-day contact project for each empire whose ships or starbases an empire newly sees. The project's timer opens comms both ways (the flag diplomacy reads), ends the other side's project, and fires `on_contact` for both (`07 D85`). 1.0 will add contact events with choices: hostile, cautious, open (`01 §4.2`). The tech hook `universal_translators` will remove "the language delay on first contact" (`docs/20`). First contact and three species met gate Xeno-linguistics and Comparative Xenology (`07 D278 §4`).

### The precursor chain (built, M11)

A precursor site is a 180-day project like a chapter, and its event gives a clue. Every opened deep cache also gives a clue. The sixth clue reveals the precursor home system (the lowest system id among the six precursor sites), surveys it in full and fires `precursor_home` (`07 D85`, `D86`). The 1.0 lore design makes this chain the first act of the Concord's story, told through a codex of fragments ([Civilisations and species](Design-Civilisations-and-species)).

### Hyperspace incidents (built, F1)

F1 shipped 8 incident events on the `on_hyperspace` trigger. The owner ruled (2026-11-14) that the flight view is physical, with no incident dice: `signal` sites placed per (seed, cell) carry the incident events when the piloted fleet flies to them, and hostile groups are placed per (seed, cell, month) by depth. Simulated travel rolls (DESIGN pillar row; `07 D123`–`D125`). The rules are on [Flight view](Design-Flight-view).

### Situations (built)

A situation is a progress bar 0–100 with stages at 25, 50, 75 and 100, an **approach** chosen by the player that changes the per-30-day delta (applied daily) and the stage effects, and an outcome at 100 or at 0 (`01 §12.2`).

**As built** (`07 D85`, `D86`): a daily watch tests each situation's start over its scopes and keeps an arming row while the start holds; after its days the situation starts. A running situation moves `(base + approach) / 30` a day, pays the approach's cost daily, holds the approach's modifier rows, fires each stage once, and ends at 100 (its `up` event) or 0 (its `down` event). The scale starts at 20 with a base of +2 a month. A colony lost to another empire ends its situation.

| Situation | Starts when | Approaches (per month) | Outcome |
|---|---|---|---|
| Planetary Unrest | stability < 25 for 30 days | ignore +3; concede −6 with +0.05 happiness; suppress −4 for 2 influence a month | stage 50 protests; at 100 a revolt (−0.1 happiness and −5 stability on the colony for a year); at 0 calm (+5 stability for 6 months) |
| Economic Recession | credits < 0 for 180 days | ride out +2; austerity −5 with −5% job output; stimulus −3 for 3 influence a month | at 100 a crash: −10% job output for a year |
| Pirate Haven | piracy > 0.7 for 360 days | ignore +2; patrols −4 for 15 credits a month; bounties −6 for 2 influence a month | at 100 a lair: +20% piracy growth for two years |

The default approach is the one that worsens. The AI's situations advisor picks, on each planning day, the most negative delta it can carry: an influence cost while six months of it still leave a starbase claim's bill, a credit cost within its margin (`07 D86`). Resolving a situation at an ungoverned colony needs the persona's fleet in orbit ([Factions and characters](Design-Factions-and-characters)).

**Faction Agitation** (FC1, `07 D214`): a party under 30 approval for 24 months starts a `faction_agitation_<base>` situation with stage effects per base at 25 and 50 (for example Mining: miner output −25% at 50), progressing `(30 − approval) / 3 × rate` a month; FC4 adds a stage at 75 that starts the party leader's plot. Its endings at 100 are the faction system's ([Factions and characters](Design-Factions-and-characters)).

**Named for later** (`01 §12.2`): Expedition Mystery, Succession Crisis (autocracy on ruler death), Plague (1.0). 1.0 plans 10 situations in all (`05` M22).

### Era transitions

Reaching an era pays unity and fires a flavour event (`01 §9.6`; [Progression](Design-Progression)). The galactic era, the highest over all empires, drives crisis eligibility.

### Crises

| Crisis | Trigger | Shape | Status |
|---|---|---|---|
| Pirates | always on | emergent, local: regional piracy levels, raiders, bases, loot pools | built (M5, M8c; [Trade and logistics](Design-Trade-and-logistics)) |
| Marauder raids | galactic Era III | periodic raiding fleets from a marauder pocket; can be paid off | designed, M20 |
| Awakened empire | galactic Era IV, 10% per year | a fallen empire reactivates and demands tribute | designed, M20 |
| Endgame crisis | crisis year (default 200) if galactic Era ≥ III | one of three: Extradimensional Incursion (a portal; waves scale with galactic fleet power), Machine Uprising (robot pops revolt in the most synthetic empires), Devourer Swarm (enters from the rim, consumes planets); escalation stages every 5 years; defeat gives a galaxy-wide reward and a score bonus | designed, M20 |

M20 is estimated at 2.4k lines (`05` 1.0 table). The 1.0 balance target asks that the crisis spawns at the crisis year (`01 §18`).

**Ties to other designs:**
- The dormant power of the 1.0 backstory is a Concord remnant sealed in the abyss, which wakes as one of M20's crises; the owner's call is that the fallen precursors are an offshoot of the storyline precursor (`docs/17 §3.4`, `07 D230` call 9).
- Risky tech is approved (`07 D230` call 8); `docs/17 §3.3` proposed lost techs with an AI-uprising risk feeding M20.
- The score's crisis term, `crisisDefeated × 500`, waits for the crises (`07 D98`; [AI and victory](Design-Ai-and-victory)).
- The tech hook `existential_risk_management` "Reduces the damage of galaxy wide crises" (`docs/20`).
- Ethics drift pushes toward Spiritualist after a recent crisis (`01 §9.3`, not built).

### RTS and TB

Events, situations and their stages run on daily systems and heap timers in both modes. In RTS a choice can pause the game; in TB choices collect in the turn report and apply on the first tick of the next turn. No event trigger, modifier or rule may test which clock is active (`01 §15`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Narrative content as data | Leaders, situations, archaeology, contact days and the precursor chain in `narrative.json`; M11's 20 events in `events/people.json` (40 in all) | 2026-11-14 | `07 D85` |
| MTTH rolls | A seed hash per event, scope and day; empire and colony scopes only | 2026-11-14 | `07 D85` |
| Situations | Watched daily with arming rows; the default approach worsens; the AI picks the fastest-falling approach it can afford | 2026-11-14 | `07 D85`, `D86` |
| First contact | Starts in the sensor recovery pass, 60 days per pair; comms open both ways | 2026-11-14 | `07 D85` |
| Archaeology as built | 3 chapters at difficulty 4 (replaces `01 §4.2`'s 3–6 chapters at difficulty 2–6 as the built numbers) | 2026-11-14 | `07 D85`, `D86` |
| Precursor clues | From the six precursor sites and every opened deep cache | 2026-11-14 | `07 D85` |
| Unrest revolt | Costs its colony happiness and stability, not empire influence | 2026-11-14 | `07 D86` |
| Hyperspace incidents | The flight view is physical with no incident dice; simulated travel rolls | 2026-11-14 | DESIGN pillar row, `07 D123`–`D125` |
| Faction Agitation | A situation per agitating party, with stages by base | 2026-11-19 | `07 D214` |
| Plot stage | Agitation's stage 75 starts the party leader's plot | 2026-11-21 | `07 D219` |
| Dormant power | The fallen precursors are an offshoot of the storyline precursor | 2026-11-16 | `07 D230` call 9 |
| Tech events and the `tech` effect | About a dozen `on_tech` events; the `tech` effect pays insight | 2026-11-21 | `07 D278 §5`, `§7` |

## Not yet built or still open

- **M20, crises:** marauder raids, the awakened empire, the three endgame crises with escalation; the score's crisis term.
- **The `tech` event effect and the `on_tech` content** (T2 and later).
- **Contact events with choices** (1.0).
- **Situations named for later:** Expedition Mystery and Plague. **Succession Crisis** is named in `01 §12.2` as a situation; FC3 built AI succession wars as civil wars instead (`07 D217`), and no source says whether the situation will still exist.
- **Fleet, leader and system scopes for MTTH events** (built: empire and colony only).
- **1.0 content:** 300 events and 10 situations (`05` M22).
- **DX1's discovery sites** will re-pool the anomalies toward the codex and the site registry ([Civilisations and species](Design-Civilisations-and-species)).

## Discrepancies noted

None found.

## Sources

- 01 §12 Events, situations, crises, 01 §4.2 Interstellar exploration, 01 §15
- 05 M11, 05 1.0 milestones
- 07 decisions: `D85`, `D86`, `D98`, `D123`–`D125`, `D214`, `D217`, `D219`, `D230`, `D278`
- 17 Civilisations, discovery and terraforming (§3.3, §3.4)
- 20 Tech hooks
- DESIGN.md (M11 autonomous calls, the flight-view pillar row)
- 00-status.md (M3, M6 and M-QA2 records)

## Related pages

- [Overview](Overview)
- [Starts and exploration](Design-Starts-and-exploration)
- [Civilisations and species](Design-Civilisations-and-species)
- [Factions and characters](Design-Factions-and-characters)
- [Progression](Design-Progression)
- [Flight view](Design-Flight-view)
- [Trade and logistics](Design-Trade-and-logistics)
- [AI and victory](Design-Ai-and-victory)
- [Time and modes](Design-Time-and-modes)
- [Modding and data](Design-Modding-and-data)


---

# AI empires and victory

**Status:** built, as of 2026-11-23. AI empires (M9, with M10's war AI and the `07 D237` budget) and the score, domination and economic victories (M14) are built; the AI's megastructure advisor (M19c), the ascension victory and the score's crisis term are not.

AI empires play the same game as the player: the same rules, the same command API, the same markets, so their economies are real and can be traded with, blockaded or starved. Each has a personality that weights what it builds, researches, signs and attacks, and it plans once a month on its own staggered day. A game ends at the end year with the highest score, or earlier by domination, economic dominance or (when built) ascension; endless mode removes the end year.

## How it works

### What the AI is (built, `01 §13`)

- **Same rules, same commands.** The AI issues the same `Command`s as the player, including market trades and trade policies; its plan is validated like any player's (`01 §13`, `07 D218` call 9). In Era I every AI exploits its home system (stations, domes, habitats) and researches the Hyperspace Drive before anything interstellar.
- **Cadence.** Each AI empire runs its planner once every 30 days on its own planning day and a cheap reactive layer every day; its markets, growth and research resolve daily like everyone's. The reactive layer walks only the contested set: an AI's unarmed fleet idle in a contested system leaves for home, and armed fleets fight or disengage by the combat rules (`07 D55`). Each plan is timed against a CPU tripwire (250 µs × calibration, judged on each empire's median of nine timings, `07 D110`; [Architecture and performance](Design-Architecture-and-performance)).
- **No special treatment.** The AI takes the same colony founding gate and civil upkeep as the player, and the same mitigations; only the persona's skills are the player's (the spec session's reading of `07 D273`, DESIGN).

### Personalities

Personalities are data: they set utility weights and diplomatic weights. Built content (`data/content/ai.json`) holds five:

| Personality | War power ratio (`07 D116`) | Notes |
|---|---|---|
| Expansionist | 1.5 | |
| Industrialist | 2.0 | |
| Militarist | 1.0 | taxation one step up at war |
| Trader | 3.0 | immigration Open |
| Scholar | 3.0 | |

`01 §13` also lists Isolationist for the MVP and Xenophobe-Purifier for 1.0.

**Policies by personality** (`01 §13`, `07 D55`): Xenophile immigration Open and refugees Accept, Xenophobe Closed and Refuse, Trader Open, the others at the defaults; no personality changes the commerce policy. Taxation moves one step up on a planning day with a ledger deficit and one step down with a surplus above 12 months of upkeep and vitality < 0.95.

### The AI economy (built, `07 D237`)

One budget per AI empire, kept in the sim, refreshed before each plan and saved:
- a forecast of recurring income, committed upkeep (builds and colonies in flight and their sprawl step), and a margin against a reserve of 3 months of upkeep (`reserveMonths`);
- the supply balance for food, energy, alloys, rare metals and goods;
- every commitment priced by its marginal recurring cost;
- a **priority ladder:** restore the reserve by selling surplus; after 3 plans in deficit, shed the building with the least output value per upkeep; fix forecast shortages at the colony with the most idle pops; then grow within the headroom (colonies, upgrades, military, pacts).

**Expansion pace:** a colony ship is built only while the empire's colonies (starbases aside) plus colony ships in flight stay under 8 plus its pops over 40 (`07 D237`, the spec session's call). The colony founding gate of `07 D273` (pops ≥ `colonyStep × m(m + 1) / 2`, m the planetary colonies past 10) applies too. AI empires score colony sites by supply resilience (`07 D274`).

### What the player sees the AI do

| Area | Behaviour | Source |
|---|---|---|
| Diplomacy | one treaty a plan, peace first, one rival, commercial pacts from year 25 with 2 hubs and the reserve met | `07 D116`, `D242`; [Diplomacy and war](Design-Diplomacy-and-war) |
| War | declares on the weakest contact it outguns by its ratio ÷ (1 + the coalition's war-hawk weight); attacks at twice the power standing; invades on ground combat's front | `07 D116`, `D217` |
| Traditions | adopts or takes a perk in the personality's tree order when unity covers the cost | `07 D55` |
| Research | six goals from need and personality, which the engine's walk works toward (T3, built 2026-11-24) | `07 D278` |
| Great works | once a year, with none of its own under construction and a non-negative margin, orders the first work it knows whose first stage its markets hold twice over, the next Kardashev type's structure first | `07 D278 §15` call 8 |
| Situations | picks the fastest-falling approach it can afford | `07 D86` |
| Politics | concedes one demand a plan to its most-supported agitating party; suppresses only when it cannot concede | `07 D214` |
| Operations | sabotage, steal tech, rig, assassinate against rivals and the player's empire (never the persona), one live at a time | `07 D219` |
| Trade | embargoes its rivals; runs up to two smuggling routes into rivals that ban a good it makes in surplus | `00-status` (M17b, M17d) |
| Leaders | never hires leaders; generated unpaid notables fill its heir and council seats (8, cap 12) | DESIGN (FC2 calls) |
| Contracts | its planning day issues the persona's contracts | `01 §13`, `§19` |

### Difficulty (built, `07 D55`)

`GameSettings.difficulty` applies one row of `government.json` to the AI empires at the start, as modifier rows:

| Level | Effect on AI empires |
|---|---|
| Cadet | −25% job output |
| Captain (default) | none |
| Admiral | +10% output, +10% research |
| Grand Admiral | +25% output, +10% research |

`01 §13` also gives Grand Admiral +1 free hub and full map visibility; neither is built, and the AI reads only its own visibility (`07 D55`). An AI-run empire 0 under the Employee kits gets no difficulty rows (`07 D218` call 10).

### Victory (built, M14)

The game ends at the end year (default 300) or earlier by a victory. Victories are checked yearly; a won game marks the winner and opens the end screen with the scoreboard, Continue playing, Main menu and New game, and the sim runs on if the player continues (`07 D98`). **Endless mode** (end year 0) disables the end year.

**Score**, kept yearly and shown in the top bar and on the end screen (`01 §14`, `07 D98`):

```
score = 0.5 × pops
      + 2 × colonies                       (starbase colonies aside)
      + 0.02 × researched techs' base cost
      + 0.001 × cumulative trade value
      + 10 × completed tradition trees     (adopted with every perk)
      + fleetPower / 100
      + eraBonus                           (0 / 50 / 150 / 400)
      + 500 × crises defeated              (waits for the crises)
```

| Victory | Rule | Status |
|---|---|---|
| Score | the highest score at the end year | built |
| Domination | one empire holding 60% of the colonised planets' pops; vassals' pops will count once M18b builds vassals | built (own pops only) |
| Economic | 50% of galactic cross-empire trade tonnage through the empire's hubs (source or destination market a hub or one of its spokes), judged monthly in months with at least 30 tons moving, for 20 consecutive years (`07 D146`) | built |
| Ascension | complete an ascension path and build its capstone megastructure | designed; waits for the ascension paths ([Progression](Design-Progression)) |

**Balance targets** that bear on the AI (`01 §18`): by year 50 at least one commercial pact in the galaxy and at least one war; by year 100 galactic Era III and ≥ 2 hubs in the median empire; by year 200 no empire eliminated by economics alone and ≥ 1 empire knowing every MVP tier-3 tech. `tools balance` judges them on seeds 1–3 (`07 D147`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| AI behaviour in M9 | Traditions in personality order; the Trader's immigration Open; difficulty as modifier rows without Grand Admiral's free hub and full map; the reactive layer on the contested set | 2026-11-13 | `07 D55` |
| AI plan tripwire | Judged on each empire's median of nine timings; build loops capped at 8 tries a plan | 2026-11-14 | `07 D110` |
| AI war and navy | Diplomacy advisor, war by power ratio, war orders, invasions through ground combat | 2026-11-14 | `07 D116` |
| AI economic model | One budget per AI empire in the sim with a priority ladder (replaces the separate colony-ship, warship and credits guards) | 2026-11-17 | `07 D237` |
| AI commercial pacts | From year 25 with 2 hubs and the reserve met on both sides (replaces `07 D116`'s "no commercial pact of its own") | 2026-11-18 | `07 D242` |
| Empire size | Founding gate and civil upkeep paced by pops, for every empire alike | 2026-11-21 | `07 D273` |
| AI and the great works | The AI builds the great works as part of development, triggered by having the resources | 2026-11-23 | `07 D278 §15` call 8 |
| AI operations | AI empires run operations against rivals and the player | 2026-11-21 | `07 D219` call 1 |
| AI megastructure advisor | On the planning day, the highest-value structure it can afford within its reserve, one at a time (designed, M19c) | 2026-11-23 | `07 D283` call 5 |
| Scores and victories | Score kept yearly; domination and end-year score built; economic victory after M10 | 2026-11-14 | `07 D98` |
| Economic victory | Monthly windows, a 30-ton floor, 20-year streak | 2026-11-16 | `07 D146` |
| Ascension capstones | New capstone structures, designed with the paths later | 2026-11-23 | `07 D283` call 3 |

## Not yet built or still open

- **M19c, the AI's megastructure advisor** (about 450 lines). Which works the AI favours beyond the type gate's structure waits for T3.
- **The ascension victory** and its capstones, after the ascension paths.
- **The crisis score term**, after M20.
- **Domination counting vassals**, after M18b.
- **Grand Admiral's free hub and full map.**
- **The station rework's AI** builds the station tier its size allows within its budget (`07 D284`, designed).
- **Stale text to fold:** `07 D146` still says only a player with commercial pacts can win economically, written before `07 D242` let AIs propose pacts.

## Discrepancies noted

- **Personalities.** `01 §13`: "Personalities (data): Expansionist, Industrialist, Militarist, Isolationist, Trader, Scholar, Xenophobe-Purifier (1.0)." `data/content/ai.json` holds five: expansionist, industrialist, militarist, trader, scholar. No decision found removes Isolationist or defers it.
- **What the AI may know.** `01 §13`: "AI never gets information it should not have except at Grand Admiral (sees full map)", and `07 D55`: "the AI reads only its own visibility". `07 D279` answer 7 (2026-11-22): "Intel is the player's only (A): the AI keeps reading the truth", and rule 15 ends "The AI reads the truth." The sources do not say whether the AI's exact reading of foreign fleet power, economy and tech counts is meant to override `01 §13`'s rule, or whether `01 §13` covers map visibility only.

## Sources

- 01 §13 AI empires, 01 §14 Victory, 01 §18
- 05 M14, 05 M10
- 07 decisions: `D55`, `D86`, `D98`, `D110`, `D116`, `D146`, `D147`, `D214`, `D217`, `D218`, `D219`, `D237`, `D242`, `D273`, `D274`, `D278`, `D279`, `D283`, `D284`
- `data/content/ai.json` (the personality list)
- DESIGN.md (autonomous calls, change log)
- 00-status.md (M14, M17b, M17d records)

## Related pages

- [Overview](Overview)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Progression](Design-Progression)
- [Factions and characters](Design-Factions-and-characters)
- [Events and crises](Design-Events-and-crises)
- [Economy and resources](Design-Economy-and-resources)
- [Trade and logistics](Design-Trade-and-logistics)
- [Megastructures](Design-Megastructures)
- [Architecture and performance](Design-Architecture-and-performance)
- [Roadmap and status](Design-Roadmap-and-status)


---

# Architecture and performance

**Status:** built, as of 2026-11-23. The engine was built in M0–M2 and reworked in P1 (performance) and P2 (saves and registries); `sim::par` merged 2026-11-18 with no parallel caller yet.

The game is one deterministic Rust simulation that serves both clocks (real time with pause, and turn-based), runs natively and in a browser from the same code, and saves in well under a second. CPU cost is the first design constraint: the whole economy resolves every tick at 1/30 of its 30-day rates, so no tick carries a month's work, and a paused game uses no CPU. For the player this means a large late game that stays fast, saves that replay exactly, and turn-based and real-time play that give identical results.

## How it works

### Principles

From `02 §1`:

1. **The sim is a pure library crate.** `sim` has no I/O, no clocks, no randomness except its own streams, and is `#![forbid(unsafe_code)]`. Threads are allowed only through `sim::par` (see "Multi-core" below; `07 D240`). It exports `World::new(seed, settings)`, `World::step(&mut self, commands)`, `serialize`, `deserialize`, `hash` and read-only queries. Everything else is a client of that API.
2. **Commands in, snapshots out.** State changes only through serialisable commands applied at tick boundaries. The UI never mutates sim state.
3. **Struct-of-arrays columns.** Entities live in `Vec` columns indexed by small integer ids, with no per-entity heap objects.
4. **Daily, incremental, flat.** Everything resolves every tick at 1/30 of its per-30-day rate. Timers live in a heap, reactive work in flag sets, totals in incremental aggregates. AI planning is the one periodic job, staggered across the month.
5. **Fixed system order.** Systems run in a declared order.
6. **Data-driven content.** Content is JSON validated by `serde` and `schemars` and interned to indices at load (see [Modding and data](Design-Modding-and-data)).
7. **Minimal code.** Hand-rolled where a crate would add more surface than it removes; crates where they remove real work.

### Crates

The workspace `Cargo.toml` declares `members = ["crates/*"]` and excludes `crates/lint-fixture` (a crate that must fail clippy, linted separately in CI). Checked on 2026-11-23, the workspace has **nine member crates**:

| Crate | Depends on | Must not depend on | Contents |
|---|---|---|---|
| `sim` | `data` | `std::time`, I/O, `rand`, any UI or graphics crate; threads outside `sim::par` | world, tables, systems, commands, event engine, trade dispatcher, combat, galaxy generation, serialize, hash |
| `ai` | `sim`, `data` | UI, graphics | advisors, planners, personalities; emits commands |
| `tactics` | `serde`, `schemars`, `postcard`, `libm` | every game crate, `std::time`, threads, I/O | the tactical battle engine: a battle as a pure function of setup, seed and input log (added in C1) |
| `data` | `serde`, `schemars` | `sim` | content JSON, schema types, registry loader and interning, derived tables |
| `view` | `data` | `sim` internals | the `SimView` snapshot layout, the `Command` and `TickEvent` enums, the `table!` macro, the hash and the UI-visible tables |
| `render` | graphics backend, `view` | `sim` | galaxy and system renderers, instanced pipelines, viewport |
| `ui` | `egui`, `view`, `render` | `sim` | screens, store, tooltips, hex surface widget |
| `app` | all | none | the `eframe` shell (native and web), sim thread and web stepper, saves, settings, PWA assets |
| `tools` | `sim`, `ai`, `data` | graphics | headless runner, `bench`, `datacheck`, `det`, `replay`, balance, sprites, smoke |

Boundaries are enforced by `Cargo.toml` dependencies and by clippy `disallowed_types` and `disallowed_methods` in `sim` and `ai` (`02 §2`). `sim`, `ai`, `view` and `data` are `#![forbid(unsafe_code)]` (`02 §6.1`).

### The deterministic simulation

**World and tables.** `World` holds the seed, the tick (tick 0 is day 1 of year 1), immutable settings, the RNG streams and every table (`02 §3.1`). Tables are declared with the `table!` macro, which expands to struct-of-arrays columns with an `alive` bit vector, a generation per slot and a free list. Ids are `#[repr(transparent)]` newtypes over `u16` or `u32`; `Id::NONE` is `u16::MAX`. Writes set the table's dirty bit. Parent and child access (ships of a fleet, cohorts of a colony) uses a CSR index rebuilt lazily when the child table is dirty (`02 §3.2`). Since P2, registry-backed columns (commodities, jobs, districts, strata, policies) are `Row` columns as wide as the content's counts, not the caps (`07 D50`).

**Step.** Each tick applies pending and external commands, runs the daily list, runs the AI planning day for the empire whose day it is, runs the yearly list every 360 ticks, advances the 30-day rings and drains events (`02 §3.3`). The daily list is 27 systems in a fixed order, from `commands`, `timers` and `fleet_movement` through the economy (`jobs`, `production`, `trade_dispatch`, `markets`, `shortages`, `ledger`, `pop_growth`, `happiness`) to `stats` and `cleanup` (`02 §3.4`). "Month" has no simulation meaning: the UI reads 30-day rolling sums, and a turn-based turn is 30 daily steps.

**Due-work structures** (`02 §3.5`):

| Structure | What it holds | Effect |
|---|---|---|
| Timer heap | `BinaryHeap<Reverse<(tick, TimerId)>>`, `TimerId` packing a kind and an id; constructions, arrivals, event timeouts, expeditions, governor re-evaluations, leader deaths, corridor drift | daily cost proportional to events due; cancellation is lazy through a `timer_gen` column |
| Flag sets | `in_transit`, `surveying`, `threatened`, `contested`, `sensor_dirty`, `invaded`, `jobs_dirty`, `shortage`, `below_target`, `above_target` and others | systems iterate only flagged items, in id order |
| Incremental aggregates | colony housing, jobs, amenities, defence; empire sprawl, fleet power, freight capacity | updated on every write path; a debug and `--verify` full recompute asserts equality |
| Rolling rings | 30-slot rings per series, slot-major with a running sum; market inbound rings of 64 slots | every 30-day figure is an O(1) read; the daily clear is one contiguous pass |

**Commands.** `Command` and `TickEvent` live in `view`, so `sim`, `ai`, `ui` and the replay file share one definition, serialised with `postcard` (`02 §3.7`). Every command applies at a tick boundary, so real time and turn-based behave alike. Invalid commands produce a rejection event.

**Snapshots.** `SimView` holds `Arc`s of the same tables the sim holds, grouped by change rate; panels compare `Arc` pointers to skip work. Publishes are rate-limited to 20 Hz. The web build reads the live columns with no copy (`02 §7.3`).

**Modifiers.** A modifier is `(scope, key, value, op, source, expires)`. The effective value is `base × Π(1 + mul) + Σ add`, held in a cache refolded only for dirty keys. Sources are kept so every tooltip shows the full stack (`02 §5`, `07 D36`).

**Determinism rules** (`02 §6.1`, `CLAUDE.md`). Banned in `sim` and `ai` by clippy: `std::time`, `std::fs`, `std::net`, `rand`, `HashMap` and `HashSet` iteration, `sort_unstable*`, `mul_add`, the `std` transcendental float methods (`powf`, `exp`, `ln`, `sin`, `cos`, `atan2` and the rest; `libm` is used through `fmath.rs`), and `f32::min`/`max` on possibly-NaN inputs. Sorts end with an id tiebreak. No fast-math or `+fma` flags are enabled. All randomness comes from PCG32 streams, one per subsystem, seeded as `splitmix64(seed ^ stream_id)`, saved with the world (`02 §6.2`). Combat draws per unit from `(seed, unit, tick)`.

**Hash and goldens.** `hash(&World)` is FNV-1a over every table's live columns, ledger scalars and rings, with NaN canonicalised (`02 §6.3`). Golden hashes for 3 seeds are committed and change only in a `sim-change:` commit.

### Threading model

| Platform | Model |
|---|---|
| Native | Two threads. The sim thread owns `World`, runs the time controller and publishes a snapshot at most every 50 ms and only when something changed. The UI thread renders and sends commands over a channel. The sim thread parks when paused, so a paused game uses 0% CPU (`02 §7.1`). |
| Web (wasm32) | One thread. Each frame the stepper runs ticks until the speed target is met or 8 ms is spent; a 1 Hz hidden-tab timer runs with 200 ms; a Web Lock avoids Chrome's Energy Saver freeze. Web Workers with shared memory are not used (`02 §7.2`, `§8`). |

Both clocks are one `Driver` (`crates/app/src/driver.rs`). Player commands in both modes go through `pending_commands` at `World.tick`, so they are saved while paused and revocable until the next tick (`02 §8`).

**AI planning** runs once every 30 days per AI empire, AI index `e` of `N` planning on day `floor(e × 30 / N)`. The plan reads the world at the start of its tick and its commands apply on the next tick. No thread is used: a plan costs 15–50 µs at Medium year 150, below a pool's round trip (`07 D55`). Budgets are counted in candidate evaluations, never in time, so a plan is lockstep-safe (`02 §9`).

### Multi-core (`sim::par`)

The history, in order:

1. Through M8c, an optional native `parallel` feature ran some loops on `rayon`.
2. P1 removed `rayon`, the `parallel` feature and `tools-par`: a `rayon` region cost 35–180 µs against the economy's 10–50 µs regions. A spin pool with a throttle was built, measured (4–5 ms spikes when a worker was descheduled on the loaded host) and dropped. The economy runs on one thread (`07 D51`).
3. On 2026-11-18 the owner asked for core offloading as a capability that changes speed only, never results (`07 D239`), and approved track P's scope (`07 D169`, `D240`).
4. `sim::par` merged on 2026-11-18 (P-par-0): a map over items whose results the caller applies in item order, `rayon` on native only. `det` hashes the goldens alike on pools of 1, 2, 3 and 8 threads, through `tools --threads N` and `--min-items N`, and the test `threads_change_nothing` (`00-status`). The module is 38 lines (`crates/sim/src/par.rs`).
5. Phase 1 (production and markets per colony) was exact but slower at Huge (a year 4.16 → 4.47 s at six threads), so it was reverted and the work stops there (`07 D261`, `00-status`). The war-storm bench found no worthwhile parallel pass either (`00-status`, P-WS).

The rules that keep results identical (`07 D169`): fixed partitions by id range; per-item writes only, with no atomics or shared accumulators; sums in the serial order; creation and removal applied serially in id order; no RNG in a parallel body.

### Saves and replay

| Aspect | Rule | Source |
|---|---|---|
| Format | `postcard` serialisation of header, tables and rings; columns written as raw bytes | `02 §13` |
| Compression | `zstd` level 3 natively; pure-Rust `miniz_oxide` deflate on the web | `02 §13` |
| Storage | native data directory with an autosave ring of 5; web IndexedDB with `navigator.storage.persist()`, plus export and import | `02 §13` |
| Tables by name | every table dump names its table and columns (generated by `table!` and `group!`); restore matches by name; a column the save predates takes its default or the value in `serialize::DEFAULTS` (the migration as data); a table the save predates starts empty | `07 D50` |
| Registry widths | a row saved narrower than the content is widened with defaults; a wider row refuses the load | `07 D50` |
| Mods | the header carries each enabled mod's id, version and content hash in load order; a different set warns | `07 D52` |
| Version | save version 13 at P2's foundation; 38, 39 and 40 across the 2026-11-23 merges (T1, M17d, T2's first slice) | `07 D50`, `00-status` |

**Replay.** Each save carries the player's `(apply_tick, Command)` log. AI commands are not logged, because the AI re-plans deterministically. `tools replay` recreates the world and compares yearly hashes (`02 §14`).

**Tests** (`02 §14`): unit tests per system on tiny worlds; market invariants as `proptest` properties; the RTS-against-TB parity test (360 ticks against 12 turns, identical hash); save, load and replay hash checks; smoke tests native and web (new game, year 1 at speed 5, save, reload, end a turn, turn report).

### The CPU budget

Reference machine: the test host, a virtual machine with 8 logical cores of a Ryzen 5 5500 and 20 GB, Windows Server 2025, release build (`07 D32`, `08 §1`). "Web" is the same machine in Chrome, single-threaded.

**Design budgets** (`08 §1`):

| Situation | Native | Web |
|---|---|---|
| Paused, no input | 0% CPU (no wake-ups) | 0% |
| Speed 3 (4 ticks/s), Medium, year 150 | ≤ 3% of one core | ≤ 8% |
| Speed 5 throughput, Medium year 150 | ≥ 1,500 ticks/s | ≥ 300 ticks/s |
| Daily tick, Medium year 150, whole economy included | ≤ 0.3 ms median, p99 ≤ 3× | ≤ 1.2 ms |
| Economy per tick (≈ 320 markets, ≈ 1,500 cohorts) | ≤ 0.15 ms | ≤ 0.6 ms |
| AI planning, one empire | ≤ 1 ms | ≤ 3 ms |
| Worst single tick, Huge (1,000 systems, 24 empires) year 300 | ≤ 12 ms and a game year ≤ 3 s, judged at the design scale (≈ 450–1,000 colonies) | ≤ 15 ms |
| Owner's ceiling: one late-game year at speed 5 on the stress profile | ≤ 15 s | none |
| Map frame CPU, Huge, all layers | ≤ 1.5 ms | ≤ 4 ms |
| UI frame, busiest screen | ≤ 2 ms | ≤ 5 ms |
| Tactical step, 100 against 100 ships, 30 Hz | ≤ 0.5 ms | ≤ 1.5 ms |
| Large-action step, 2,000 ships, 30 Hz | ≤ 1.5 ms | ≤ 4 ms |
| Ground battle day (91 hexes, 20 brigades) | ≤ 25 µs | none |
| Flight view frame data | ≤ 0.3 ms | none |
| Empty shell resident memory | ≤ 100 MB | none |
| Resident memory, Huge year 300 | ≤ 150 MB | ≤ 250 MB |
| Save, Huge | ≤ 300 ms | ≤ 1 s |
| Cold start to main menu | ≤ 1 s | ≤ 3 s after download |

The GPU rows are on [Presentation and UI](Design-Presentation-and-ui).

**Re-baselined bench lines.** Since P1, the economy row, the full tick and M9's year-150 rows take as their 1× line the single-thread median of three host runs plus 10% (the throughput floor: the median over 1.1). CI fails at 2× (below half for throughput). Host background load is accepted and judged on medians (`07 D51`). The lines recorded in `08 §1`, at calibration 1.0:

| Row | Original line | Lines since (decision) |
|---|---|---|
| Economy systems, 320 colonies | 120 µs | 311 (`D51`), 392 (`D57`) |
| Full daily tick of that world | 300 µs | 574 (`D51`), 780 (`D57`) |
| Dispatcher, steady state | 80 µs (M5) | 115 (`D54`), 195 (`D57`) |
| Dispatcher, worst case | 600 µs fixed | 441 (`D57`) |
| M9 Medium year 150 tick | 300 µs | 882 (`D51`), 986 (`D82`), 1,145 (`D57`), 1,345 (`D105`), 1,546 (`D108`), 1,505 (`D151`) |
| M9 speed-5 throughput floor | 1,500 ticks/s | 1,063, 944, 794, 722, 630, 628 ticks/s (same decisions) |

The AI plan's CI tripwire is 250 µs × calibration, judged on the heaviest empire's median of nine timings at Medium year 150 (`07 D55`, `D110`).

**Huge.** `07 D239` judges the Huge target at the design scale (≈ 450–1,000 colonies at year 300); the AI-only run of about 3,000 colonies is a tracked stress benchmark, not a gate. On 2026-11-22 the owner accepted the design-scale baseline of about 3.0 s a year and a 9.7 ms worst day (731 colonies) after a bisect and a profile found no cut worth 0.2 s a year or 1.5 ms (`05 M14`). The web line is measured at 409 ticks/s against 300 (`05 M14`).

**Scaling targets** (`08 §6`): Medium 400 systems, 8 AIs, daily tick ≤ 0.3 ms, AI plan ≤ 1 ms; Large 700 systems, 16 AIs, ≤ 0.6 ms, ≤ 1.5 ms; Huge 1,000 systems, 24 AIs, ≤ 1.0 ms, ≤ 2 ms. "If a milestone misses a budget by more than 2× after algorithmic work, the fix is an algorithm, an incremental update or a stagger, never a smaller galaxy." The stress profile for the 1.0 bench is Huge, 24 empires, ≈ 1,200 colonies, ≈ 6,000 cohorts, ≈ 40,000 ships in ≈ 2,500 fleets, 20 commodities and 600 events, at ≤ 15 s a game year (`08 §6`).

### Techniques and banned patterns

Techniques (`08 §3`): daily resolution; the event heap for timers; flag sets instead of scans; struct-of-arrays columns; incremental aggregates; the dirty-key modifier cache; routes computed once at order time; bounded solvers (the trade solve capped at 6 sources per deficit, AI candidate lists capped in data); cohorts instead of individual pops; lazy generation of surfaces and anomalies; the sim decoupled from rendering; instanced map draws uploaded on change; interned strings and text formatted on change.

Banned (`08 §4`, `CLAUDE.md`):

- A system that iterates a whole table every tick to check a per-entity timer or state.
- Monthly work for all empires on one tick.
- Recomputing an aggregate that an incremental update could maintain.
- Pathfinding inside a per-tick loop.
- Nested loops over two entity tables without an index.
- Per-entity `Box`, `Rc`, `Vec` or `String` in sim tables.
- `HashMap` iteration in the sim.
- `std::time`, I/O and `rand::thread_rng` in the sim; threads outside `sim::par`.
- UI code that formats strings or sorts rows every frame.
- Repainting while paused with no input; polling timers in the UI loop.

### Benches and the remote gate

**The gate** (`CLAUDE.md`, `05`): `cargo fmt --all --check`, `clippy -D warnings`, `tools datacheck`, `nextest`, `tools det`, `cargo build --release` and `trunk build --release`, wrapped as `tools gate`.

**The remote gate.** Tests and benches run on the test host, not the dev box (`07 D32`). `scripts/gate-remote.ps1` mirrors the working tree with robocopy, runs `tools gate` there over WinRM and streams the log to `target/gate-remote.log`. Only that log claims a criterion or supplies a bench row.

| Switch | Effect |
|---|---|
| `-Bench` | also runs `tools bench`, then `tools smoke` (native and web) and `tools balance`, failing on a smoke failure or an expansion box (`07 D86`, `D170`) |
| `-Track <name>` | mirrors into a folder on the host, so concurrent tracks gate apart; at most two gates at once; a bench holds a host-wide mutex so its timings stay clean (script header) |

Copied files are stamped with the host's clock so cargo cannot reuse a stale artifact (`07 D107`). `scripts/host-run.ps1` runs ad-hoc host jobs under the same mutex.

**Measurement** (`08 §5`): `tools bench` runs the reference seeds after a calibration loop and prints every row against its 1× line, failing at 2×; `tools huge` measures Huge year 300 (`--colonies N` caps colonies for the design-scale row, `07 D263`); `tools bench --stress` runs the war storm on a cached Huge year-300 world (`07 D262`); criterion micro-benchmarks; Tracy and puffin profiling behind features; an idle test asserting under 50 ms of CPU over 60 paused seconds. GPU rows come from `space-sim --gpu-bench` and are not part of CI enforcement (`08 §1`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Rust, daily tick | The economy resolves every tick at 1/30 of 30-day rates; AI planning is the only staggered periodic job | 2026-11-01 | `00-status` pass 6 |
| Backend measured | `glow` chosen over `wgpu` by the M0 measurement (later reversed for the presentation pass, see [Presentation and UI](Design-Presentation-and-ui)) | 2026-11-01 | `07 D19`, `03 §3` |
| Testing host | Gates and benches run on the test host through `gate-remote.ps1` over WinRM | 2026-11-02 | `07 D32` |
| Architecture for performance and modding | Performance from architecture; per-colony markets kept; AI planning off the tick (M9); three concurrent tracks after M8c, each in its own worktree and host folder | 2026-11-13 | `07 D49` |
| Saves by name | Tables and columns restore by name, `serialize::DEFAULTS` as the migration list, save v13; only the previous version loads until 1.0 | 2026-11-13 | `07 D50` |
| P1 scope | Exact single-thread rework; snapping and lazy settling dropped; `rayon` and the spin pool dropped; bench lines re-baselined to host medians plus 10%, CI at 2×; 15 s per late-game year stays the hard guard. Replaced the earlier native `parallel` feature of `02 §6.4` | 2026-11-13 | `07 D51` |
| AI planning inline | Plans run between ticks with no thread; 250 µs tripwire | 2026-11-13 | `07 D55` |
| Huge judged at the design scale | 3 s a year and 12 ms worst tick at ≈ 450–1,000 colonies; the 3,000-colony run is a stress benchmark. Replaced the earlier ≤ 4 ms and ≤ 2 s Huge lines (`D234` before it) | 2026-11-18 | `07 D239`, `D263` |
| Core offloading | Threads only through `sim::par` (fixed partitions, ordered merges); det must hash the same single- and multi-threaded; the native client is the real target and the web stays single-threaded; a Cores setting (Auto, 1, or 2 up to cores − 1, capped at 8; Auto = min(cores − 1, 6) at 4+ cores) | 2026-11-18 | `07 D240` |
| Phase 1 reverted | Production and markets in parallel were slower at Huge; the harness stays with no caller; the Cores setting waits for a pass that pays | 2026-11-18 | `07 D261`, `DESIGN.md` change log |
| Huge baseline accepted | About 3.0 s a year and a 9.7 ms worst day at 731 colonies | 2026-11-22 | `05 M14` |

## Not yet built or still open

- **The Cores setting** (`07 D240`) is not built; it waits for a parallel pass that pays (`07 D261`). No parallel caller of `sim::par` exists.
- **Further parallel phases** (jobs, growth and happiness; the migration refreshes; per-empire dispatch) are scoped in `07 D169` and not built.
- **Threaded web build:** out of scope; it would need COOP/COEP, `SharedArrayBuffer` and a nightly `-Zbuild-std` build (`07 D169`).
- **The stress profile assertion** at ≤ 15 s per year is M23's (`08 §6`).
- **CI matrix:** the Linux, macOS and wasm32 determinism runs in CI were recorded as not verified in the early milestones, pending the owner's push (`00-status` passes 9–17); later status entries do not say whether they now run.
- **Multiplayer:** lockstep multiplayer is post-1.0 (`SPEC.md §4`).
- **Owner review:** the tracker lists "Whether `CLAUDE.md` should say nine crates (`tactics` added)" as an open question (`DESIGN.md`).

## Discrepancies noted

- **Crate count.** `CLAUDE.md` says "Rust workspace, eight crates (`docs/02 §2`)", and `02 §2` says "Eight crates keep incremental compile times short", while the table in `02 §2` lists nine crates including `tactics`. The workspace `Cargo.toml` (`members = ["crates/*"]`, excluding `crates/lint-fixture`) has nine members: `ai`, `app`, `data`, `render`, `sim`, `tactics`, `tools`, `ui`, `view`. `03 §12`'s workspace layout lists eight crate folders and omits `tactics`. The tracker keeps this as an open question for the owner.
- **Which saves load.** `07 D49` and `D50` say "Old saves: A. Pruned until 1.0 (the previous version only)". Later records describe a range: T1's merge has "save v38 (a v28–v37 save loads)" (`05 T`, `07 D278`), and `docs/release/0.2.0-notes.md` says "this build loads saves back to 0.1.0's". No decision found that widened the rule.
- **The current year-150 line.** `08 §1`'s last recorded re-baseline is 1,505 µs and 628 ticks/s (`07 D151`). Later merge records in `00-status` judge the row against other lines, for example "M9 1,767 µs (1,803 line), 560 ticks/s" (M17b, 2026-11-22) and "Medium y150 is 1,786 µs against its 1,931 line" (`07 D168`). The `DESIGN.md` change log mentions a re-baseline "M-BASE" for a 461-colony world. Which line is current is not recorded in `08 §1`.
- **Determinism tests and `parallel`.** `02 §6.3` and `§14` still say determinism tests run "with and without the `parallel` feature", and `03 §6` says `tools det` runs "with and without `parallel`", while `02 §6.4` (P1 note) and `07 D51` say the feature is removed and `07 D240` describes det across thread counts through `sim::par`.

## Sources

- SPEC.md, scope tiers
- CLAUDE.md
- 02 Architecture: crate layout, simulation core, determinism, threading, time controllers, save and load, replay and testing
- 03 Tech stack: tools crate, CI
- 04 Blockers and risks
- 05 Milestones (P1, P2, M14)
- 08 CPU budget: budgets, banned patterns, measurement, scaling targets
- 07 Decisions: `D32`, `D49`, `D50`, `D51`, `D55`, `D110`, `D169`, `D239`, `D240`, `D263`
- 00 Status
- DESIGN tracker
- `Cargo.toml`, `crates/sim/src/par.rs`, `scripts/gate-remote.ps1`

## Related pages

- [Performance: goals, decisions and caveats](Engineering-Performance)
- [Testing and benchmarking](Engineering-Testing-and-benchmarking)
- [Overview](Overview)
- [Time and modes](Design-Time-and-modes)
- [Modding and data](Design-Modding-and-data)
- [Presentation and UI](Design-Presentation-and-ui)
- [Roadmap and status](Design-Roadmap-and-status)
- [Economy and resources](Design-Economy-and-resources)
- [Trade and logistics](Design-Trade-and-logistics)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [AI and victory](Design-Ai-and-victory)
- [Combat](Design-Combat)


---

# Modding and data

**Status:** built, as of 2026-11-23. Data registries and saves by name in P2 step 1, the mod system and schema validation in P2 step 3, web zip import in M13; WebAssembly code mods are designed for 1.0.

All game content (commodities, jobs, buildings, techs, hulls, events, AI weights and the rest) is JSON validated against schemas and interned to dense indices at load, so the engine names almost no content in code. A mod is a folder of JSON that adds, replaces, patches or removes records by id; the merged set is validated as one, and saves record which mods were loaded. The goal, set by the owner, is a game as moddable as its reference games, with less code and no lost function (`07 D49`).

## How it works

### Content as JSON

- Content lives in `data/content/`, one file per kind (`02 §15`, `06`). The files on disk on 2026-11-23: `ai.json`, `bodies.json`, `buildings.json`, `colonies.json`, `commodities.json`, `constructions.json`, `districts.json`, `economy.json`, `exploration.json`, `galaxy.json`, `government.json`, `hulls.json`, `jobs.json`, `megastructures.json`, `narrative.json`, `persona.json`, `policies.json`, `starbases.json`, `stars.json`, `strata.json`, `surface.json`, `techs.json`, `traditions.json`, plus the `events/` and `strings/` folders.
- Ids are `snake_case` strings unique within their kind. Cross-references use ids and resolve to indices at load (`06`).
- **All rates in content are per 30 days** (outputs, inputs, upkeep, baskets, decay). The registry divides them by 30 when it builds the daily tables (`06`).
- Every numeric modifier goes through the modifier system with a key from the key registry (`06 §10`). Text lives in `strings/en.json` keyed `kind.id.field`, with placeholders `{planet}`, `{empire}`, `{leader}`, `{n}` (`06 §13`).
- Content is embedded in the binary with `include_bytes!` for release (`02 §15`).
- `commodities.json` holds 17 commodities on 2026-11-23 (the E1 resource model; see [Economy and resources](Design-Economy-and-resources)).

### Schemas and validation

- `serde` structs with `schemars` derives in the `data` crate are the single source of truth. `tools datacheck` emits `data/schema/*.schema.json`, which give editor autocompletion and serve as the modders' documentation (`03 §7`, `07 D53`).
- **`data::schema`** is a small validator in `data` (about 150 lines) covering only the keywords schemars emits. It runs first in `Content::parse`, so every load is checked the same way: base or modded, native or web, the app, the tests and datacheck. The `jsonschema` crate was rejected because it would have checked datacheck only and pulled a large dependency tree into `tools` (`07 D53`).
- Twenty single-field rules moved from hand-written checks onto `#[schemars(..)]` attributes: weights, build days, speeds and sizes, elasticity, the trade band and days, rings, the surface's ring counts, starbase sizes, survey levels, tech tiers, weapon-slot counts and the stance registry's size. `check` keeps references and cross-field and cross-file rules (`07 D53`).
- **Errors** name the file, the path with the record's id, and the mods that touched the file (`07 D53`).

**What `datacheck` enforces** (`06 §16`, abridged):

| Area | Rule |
|---|---|
| References | unknown modifier keys, unknown ids in any reference, duplicate ids fail |
| Registries | every registry between 1 and its cap; each commodity role on exactly one commodity; one crew job; one `constructions.json` row per construction kind |
| Starting layout | every starting district on an eligible hex, adjacency 0, every district served |
| Techs | every tech reachable from tier 0; unreferenced unlocks warn |
| Economy | a commodity with a consumer but no producer is an error; one with neither warns |
| Events | events without a trigger, options without effects and without `defaultOption`, missing strings fail |
| Ships | hulls whose slots exceed the auto-design's power budget warn |
| AI | personality weights outside [0.25, 4] fail |

### Registries with caps

The hard-coded kinds became data registries (`12 §3`, `07 D50`): commodities, strata, job kinds, construction kinds, building kinds, district kinds, hull classes, fleet stances, modifier keys, deep factions and gear slots. At load each registry assigns dense indices in load order; the sim uses the indices and content refers by string id. Rules with behaviour attach to kinds through flags in the data (for example a construction kind's `drawsFrom`), not through `match` on a constant.

**Caps** are validation limits checked by `datacheck` (`07 D50`):

| Registry | Cap |
|---|---|
| Commodities | 32 |
| Strata | 8 |
| Jobs | 32 (the fill's job mask is a `u32`) |
| Districts | 16 (a colony kind's district mask is a `u16`) |
| Policies | 16 |
| Buildings | 126 (the hexes' building bit) |
| Hulls and stances | 255 |

Table rows are as wide as the content's counts, not the caps: the first build sized rows to the caps and the economy row rose 13–15% on the host, so a registry-backed column became a `Row<T, W>` whose width is fixed per world (`07 D50`).

**Content that fills engine roles** (`06`, `07 D50`, `D54`):

- a commodity's `role` (`food`, `goods`, `luxury`, `fuel`, `alloy`), each role on exactly one commodity;
- a job's `crew: true` (the one starbase crew job);
- `strata.json`, lowest stratum first, each `{ id, income, happinessWeight, extraLuxury?, basket }`;
- `constructions.json`, one row per engine construction kind (`ship`, `station`, `found`, `upgrade`, `district`, `building`, `habitat`, `clear`, `army`, `corridor`, `gateway`, `outpost`) with `drawsFrom`, `yardRate`, `yardSlot`, `demandFloor`, `lapsesOnConquest`;
- each `colonies.json` kind's `role` (`surface`, `domed`, `habitat`, `starbase`), a designation's `home: true`, `economy.json`'s `governor.alloyBuilding`;
- `ai.json`'s producer table, warship order and input, admin buildings, idle stance and pact settings, checked by `ai::parse`.

What still names ids in code: the dev staging helpers for the bench and the screenshots, and tests (`07 D54`).

### The event and effect grammar

Events are compiled at load into an enum tree interpreted over `(world, scope)`, with no closures or boxed trait objects per event; trigger buckets index events by trigger kind (`02 §10`). The grammar (`06 §11`):

- **Triggers:** `on_survey`, `on_project_complete`, `on_colonize`, `on_hex_survey`, `on_daily_mtth` (with `mtthDays`, rolled with `p_day = 1 − 2^(−1/mtthDays)`), `on_contact`, `on_tech`, `on_era`, `on_leader_death`, `on_battle_end`, `on_fired`. No trigger may depend on which clock is active.
- **Conditions:** `all`, `any`, `not`, `flag`, `flagGlobal`, `stat` comparisons, `has` (tech, building, district, civic, tradition, perk, treaty, trait), `ethicsPole`, `era`, `year`, scope switches, `random`, and `pred` (a Rust predicate registered by id, the escape hatch).
- **Effects:** `ledger`, `stock`, `modifier`, `flag`, `event` (with `delayDays`), `situation`, `spawnFleet`, `reveal`, `tech`, `ethics`, `leader`, `hex`, `opinion`, scope switches and `weighted` branches.

Events with options interrupt unless hidden; turn-based mode defers them to the turn report. Unanswered events take their default option after `timeoutDays` in both clocks (`02 §10`). Situations, leaders, archaeology and species are data in `narrative.json` (`06 §12`, `07 D85`, `D97`).

### Folder mods with patches

**Layout** (`12 §2`):

```
mods/<id>/mod.json           { id, name, version, gameVersion, dependsOn[], loadAfter[], files[] }
mods/<id>/content/...        same tree as data/content/
mods/<id>/strings/<lang>.json
mods/<id>/assets/
mods/<id>/CREDITS.md
```

`files` lists the mod's files explicitly; it is not a glob. The same manifest reads from a folder and from a web zip, and the content hash is over exactly those files (`07 D52`).

**Operations per record, keyed by `id`** (`12 §2`, `06 §14`):

| Record in the mod | Effect |
|---|---|
| A new id | adds a record |
| `"$replace": true` | replaces the record |
| Any other object for an existing id | merge patch with RFC 7386 semantics: `null` removes a field, arrays replace |
| `"$remove": true` | deletes the record; refused while any file still names the id as a string value |

Records keyed by `id` merge in any array of records, not only at a file's top level, so a mod adds a hull inside `hulls.json`'s `hulls` without restating the file. An array without ids replaces. The `$remove` check is textual, so an unrelated field holding the same string also counts (`07 D52`).

**Load order** is resolved from the dependencies, then `loadAfter`, then the mod id. It is a pure function of the enabled set, so every machine merges identically (`12 §2`).

**Layering, not a second loader.** `data::mods::layer` merges a mod's JSON into the base files' JSON in load order before parsing, so the sim, the schemas and `check` see one content set (`07 D52`).

**One installed set per process.** `data::base()` returns the embedded files with the installed mods, installed before anything reads the content. The native app takes `--mod <dir>` (repeatable). There is no enable list or mod menu in MVP. `ai.json` is among the layered files, so a mod can retune the AI (`07 D52`).

**Web.** The web build imports a mod as a zip through a file picker. `Mod::from_zip` reads a zipped mod folder as the folder, with the same hash. Zips are stored in IndexedDB as `mod:<id>` and installed at the next start, because the content is fixed per process (`07 D95`, `05 M13`). A failed load names the mod difference (`05 M13`).

**Assets.** The sprite manifest maps hull, station and structure ids to atlas rectangles, so a mod adds a sprite by adding a row (`07 D76`). A body class names its planet look and atmosphere in `bodies.json`, so a mod's class picks a look (`07 D83`).

**Tooling.** `tools mod new <id>` scaffolds a mod. `tools mod check <dir>` layers the mod and the sibling folders it depends on, runs datacheck, plays a Tiny game for 400 days, saves, checks the header's mod list, reloads with the same hash and plays 100 more days. The gate runs it on `mods/example` (`07 D52`). Mods carry their own `CREDITS.md`, and the credits screen lists each installed mod's file (`12 §2`).

**The example mod.** `mods/example` ("Example: the spice trade", version 0.1.0) patches `commodities.json`, `strata.json`, `jobs.json`, `hulls.json`, `events/anomalies.json` and `techs.json` and adds strings. Its commodity appears in the markets matrix with no panel code (`07 D54`).

**Generic panels.** The screens build their rows from the registries (the markets matrix over the commodities, research over the techs, buildings, jobs and designs over their content), and `ui::rows` is the shared row list, so a new content kind appears without new panel code (`07 D54`).

### Saves keyed by name

- Every table dump names its table and each column, generated by `table!` and `group!`. Restore finds both by name (`07 D50`).
- A column the save predates takes its type's default, or the value `serialize::DEFAULTS` lists for it: the migration as data. A table the save predates starts empty.
- A registry row saved narrower than the content (the content gained entries) is widened with defaults. A row wider than the content refuses the load. This is why a save made with a mod's extra commodity is refused once the mod is removed (`05 M13`).
- The save header carries `(id, version, content hash)` per enabled mod in load order (save version 16). A load with a different set warns in the status line with both differences and loads anyway (`07 D52`).
- Old saves were pruned for MVP: the v2–v11 migration chain, its fixtures and the repairs only it called were deleted (`07 D50`). From 1.0, every release keeps its migrations for good (`12 §1`).

### WebAssembly code mods (designed, 1.0)

- **MVP:** everything moddable is data plus the event and effect grammar: events, conditions, effects, modifiers, AI weights (`12 §1`).
- **1.0:** sandboxed, deterministic WebAssembly code mods with fuel metering, hooked at fixed points (the planning day, events, UI panels), the same on native and web (`12 §1`, `07 D49`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Mod format | Folders with patches: a manifest with load order; add, replace, merge-patch or remove by id; strings and assets by path; the merged set validated; mod hashes in saves | 2026-11-13 | `07 D49`, `12 §1` |
| Code mods | Data and the grammar in MVP; sandboxed deterministic WebAssembly in 1.0. Replaced `SPEC.md §4`'s placement of scripted mods (WASM or Lua) in post-1.0 | 2026-11-13 | `07 D49` |
| Core kinds | Data registries with caps (commodities up to 32, strata up to 8) | 2026-11-13 | `07 D49` |
| Old saves | Pruned until 1.0 (the previous version only); from 1.0 every release keeps its migrations | 2026-11-13 | `07 D49`, `12 §1` |
| Rows as wide as the content | Caps are validation limits; table rows follow the content's counts, since cap-wide rows cost the economy row 13–15%. Replaced `12 §1`'s "arrays sized to the cap" | 2026-11-13 | `07 D50` |
| Saves by name | Tables and columns restore by name; `serialize::DEFAULTS` is the migration list; save version 13 | 2026-11-13 | `07 D50` |
| Layering | A mod's JSON merges into the base JSON before parsing; ids merge in any array; `files` listed, not globbed; one installed set per process; mod list in the save header (v16) | 2026-11-14 | `07 D52` |
| `$remove` | Built as `"$remove": true`. Replaced `06 §14`'s earlier `"$delete": true` | 2026-11-14 | `06 §14`, `07 D52` |
| Schema validation | A small in-tree validator run on every load; twenty single-field rules moved onto the types | 2026-11-14 | `07 D53` |
| Hot reload and coverage | Cut | 2026-11-14 | `07 D54` |
| Web zip import | Moved from P2 to M13; built with IndexedDB storage and install at next start | 2026-11-14 | `07 D54`, `D95` |

## Not yet built or still open

- **WebAssembly code mods** with fuel metering and fixed hooks (1.0, `12 §1`).
- **Multiplayer's refusal** to start with a different mod set (1.0, `12 §2`).
- **Hot reload** of content in dev builds: cut (`07 D54`).
- **An in-game mod menu or enable list:** none in MVP (`07 D52`). The 1.0 milestone M21 lists "pack manifest, load order, override rules, in-game pack manager, mod validation UI" (`05`, 1.0 table).
- **Mod removal screen** on the web: none until M14's settings (`DESIGN.md`, M13 call). No later record of it was found.
- **Gear slots** as a registry waited for C1's fitting (`12 §3` built note); no later record of the registry was found.
- **Content still named in code:** the dev staging helpers for the bench and screenshots, and tests (`07 D54`).
- **Species and empire skins** (panel frames as moddable data) are part of the presentation pass, designed only (see [Presentation and UI](Design-Presentation-and-ui)).

## Discrepancies noted

- **Commodity count in the schema doc.** `06 §1` says "Exactly the 10 ids in `01 §7.2` for MVP", while `commodities.json` holds 17 records and `DESIGN.md` describes 17 commodities from E1. `06 §1`'s P2 note raises the cap to 32 but does not restate the count.
- **M21's scope against P2.** The 1.0 table in `05` describes M21 "Modding data packs" as "pack manifest, load order, override rules, in-game pack manager, mod validation UI" at 1.2k lines, while `07 D52` records the manifest, load order and override rules as built in P2. It is not recorded whether M21 is reduced to the pack manager and validation UI.
- **Which saves load.** `07 D49` and `D50` keep only the previous save version loading until 1.0; T1's record says "a v28–v37 save loads" (`07 D278`), and the 0.2.0 notes say the build "loads saves back to 0.1.0's". No decision found that widened the rule.
- **Migrations dropped.** `12 §1` says "The v2–v10 migration chain and its fixtures are dropped, about 1.2k lines"; `07 D50` records the v2–v11 chain and "nine fixtures" deleted. The later record is the more specific.

## Sources

- SPEC.md, scope tiers
- 02 Architecture: content pipeline, save and load, event engine
- 03 Tech stack: content tooling
- 05 Milestones (P2, M13, the 1.0 table)
- 06 Data schemas: modifier keys, event, pack manifest
- 12 Modding: the owner's calls, the mod system, registries, simplicity
- 07 Decisions: `D49`, `D50`, `D52`, `D53`, `D54`, `D76`, `D83`, `D95`
- DESIGN tracker
- `data/content/`, `data/schema/`, `mods/example/mod.json`

## Related pages

- [Overview](Overview)
- [Architecture and performance](Design-Architecture-and-performance)
- [Presentation and UI](Design-Presentation-and-ui)
- [Roadmap and status](Design-Roadmap-and-status)
- [Economy and resources](Design-Economy-and-resources)
- [Events and crises](Design-Events-and-crises)
- [Ships and fleets](Design-Ships-and-fleets)
- [AI and victory](Design-Ai-and-victory)


---

# Presentation and UI

**Status:** built for the current look (G1–G4 and M14's UI pass); designed for the immersive presentation pass (`07 D231`, milestones R1–R4, U1–U2, A1, not placed), as of 2026-11-23.

The presentation is everything the player sees and hears: the galaxy and system maps, ships, planets and surfaces, the egui screens, sound and music. The built look pays for quality offline or once (Blender-rendered lit sprites, GPU-generated planets baked on first view) so each frame costs a few texture fetches per pixel, and a paused, idle game draws nothing. On 2026-11-16 the owner asked for a more immersive, release-like look in the spirit of the reference games, spending performance headroom; that pass is scoped and comes after the MVP.

## How it works

### Renderer and backend

- **Shell.** `eframe` provides the window, input, graphics context and frame loop on native and web (`02 §11.1`).
- **Backend.** `glow` (OpenGL 3.3 / WebGL2), chosen by the M0 measurement: 76 MB resident and a 0.30 s cold start, against 165–215 MB and 0.78–1.26 s for `wgpu`, which also failed the idle test (`03 §3`). The `wgpu` implementation was removed on 2026-11-02 at the owner's call (`02 §11.1`).
- **The `Backend` trait** in `render` began with six calls (`upload_instances`, `draw_layer`, `upload_texture`, `set_viewport`, `begin`, `end`). G1 added `bake_background` (`07 D42`) and G3 added `bake_planet` (`07 D83`), so a backend is still one file.
- **WebGL2 limits.** All pipelines use only vertex and instance buffers, uniforms of 16 KiB or less and textures, the intersection of OpenGL 3.3 and WebGL2 (`02 §11.2`, `10 §5`).
- **Repaint on demand.** The UI requests a frame when a new snapshot arrives, on input, and during animations; otherwise no frames are produced. A paused game with no pointer movement renders nothing (`02 §11.1`), proved by `app::frames` (`00-status`, G1).
- **Shaders.** The GLSL sources live in `crates/render/src/shaders`; `tools shaders` validates them for `#version 330 core` and `300 es` in the gate, printing a notice and passing where the validator is missing (the host) (`07 D42`).

### Graphics by element

| Element | As built | Source |
|---|---|---|
| Galaxy background | fBm nebula and star field baked once per galaxy on the GPU into a tileable texture; parallax 0.3 px per world unit of camera travel; every look parameter a field of `Bake` | `07 D42` |
| Stars | Star shader: limb-darkened disc, glow and corona by spectral colour, additive. Glow size `30 + 26 × (blue − ½ red)` pixels, clamped 24–56, ×1.3 for home systems; a colour under 0.5 luminance draws a black disc with a bright ring (the black hole); the corona turns only on High | `07 D42` |
| Territory | The centroid dual of a Delaunay triangulation over the systems plus unowned fill points; borders drawn from an edge distance in the fragment shader; re-meshed only when a claim the player sees changes | `07 D42` |
| Hyperspace layer | One texel per cell on a `TEX` quad, re-uploaded when the month or a corridor changes: clouds violet, storms amber, corridors cyan, the deep darker; fleet routes as lines | `02 §11.2` |
| Ships, stations, structures | Blender Cycles renders each design from straight above into albedo and normal-with-emissive atlases; the shader lights the quad from the system's star (upper left on the galaxy map). Two instances per ship: a `GLYPH` (the class arrowhead below 18 px on screen) and the lit `SPRITE`; LOD decided in the shader. 136 sprites in six lineages | `07 D76`, `00-status` (G2) |
| Planets | A procedural equirectangular texture per body, baked on the GPU on first view from its seed and class, drawn as a lit rotating sphere with a Fresnel atmosphere rim. Sixteen looks (continental, ocean, tropical, arid, desert, savanna, alpine, tundra, arctic, barren, molten, toxic, frozen, gas giant, ice giant, asteroid). One turn takes 40–120 s by seed; Low does not turn | `07 D83` |
| Planet cache | One RGBA8 texture array of 64 layers, each 256×128 with its mip chain: 128 KB a layer, about 10.7 MB in all; LRU by last view; the planet screen has its own 512×256 texture | `07 D83` |
| Sol | The 15 named major bodies use NASA, JPL and USGS public-domain maps at 512×256 WebP (0.17 MB) | `07 D41`, `D83` |
| Hex surface | One egui textured mesh: 12 terrains and 10 features rendered by Blender into one 1024×416 atlas (40 KB lossy WebP); hidden hexes as flat fog | `07 D84` |
| Gigastructures | Builders for the Matrioshka Brain, Dyson Sphere, Ringworld, Star Lifter and Nicoll-Dyson Lance; drawn around the star, 110 units across | `07 D84` |
| Motion | Hyperspace storms blend toward next month by the month's fraction; corridors shimmer; fleets trail six days of route in hyperspace and the last quarter of an in-system leg; a jump in or out gets a 1.6 s glint | `07 D87` |
| Battles | Instanced ships, shots, beams, impacts and particles on a second backend path (C1) | `00-status` (C1) |

**No post-processing** in the built look: glow is drawn into the sprites and the star shader with additive blending, so there is no bloom pass and no off-screen framebuffer (`07 D37`, `10 §2`).

**Presets.** Low, Standard and High (`10 §2`, `00-status` G1). The animation clock advances at 30 frames a second only while the map shows and the game runs. High animates throughout (corona, corridor shimmer); Standard animates only while something moves; Low never animates (`07 D87`).

**Asset budget.** ≤ 3 MB for the download (`03 §10`). To stay inside it, only the 2× sprite atlases ship and the UI box-filters lower sizes at load (`07 D84`); the normal atlas ships at 1×; the 2× albedo atlas ships as lossy WebP (quality 92). After G4 the download is **2.22 MB of 3 MB** (`07 D88`).

**GPU budgets** (`10 §3`, `08 §1`), measured on the host's GTX 1060 3 GB through `space-sim --gpu-bench`:

| Row | Budget native / web | Measured (host GTX 1060) |
|---|---|---|
| Map frame, Huge, all layers, 1080p | ≤ 2 ms / ≤ 4 ms | 0.329 ms p50 (G1); 0.431 ms with G4's weather in the shader |
| System view, 30 bodies and 40 ships, 1080p | ≤ 2 ms / ≤ 4 ms | 0.066 ms (G1); 0.147 ms with G3's lit spheres and sprites |
| Same at 4K (2 pixels per point) | ≤ 2 ms | map 0.952 ms p50, system 0.213 ms |
| Battle frame, 2,000 ships with effects, 1080p | ≤ 4 ms / ≤ 8 ms | 0.071 ms p50 (C2); 0.075 ms as sprites (G2) |
| Texture memory | ≤ 64 MB | 42.5 MB at 1080p, 48.4 MB at 4K (G3, mip chains and full planet cache counted) |
| First planet view | ≤ 5 ms per body / ≤ 15 ms | 0.34–1.40 ms over the 16 looks |
| Map frame CPU, Huge | ≤ 1.5 ms / ≤ 4 ms | not restated |
| UI frame, busiest screen | ≤ 2 ms / ≤ 5 ms | not restated |

Web GPU rows are not measured (`08 §1`).

### Audio

- **Engine:** `kira` 0.12 on both targets; on the web it drives Web Audio through `cpal`. It starts on the first click or key, meeting the browser's gesture gate. Vorbis in Ogg (`07 D88`).
- **Buses:** interface, effects, ambience and music under the master, each a slider on a 40 dB taper, plus a mute. Each sound plays at most every 70 ms. A paused, idle game queues nothing; the audio thread keeps the music going (`07 D88`).
- **Cues:** clicks; settings and planet view open and close; a rejected command errs; completions chime; research, eras, colonies and events notify; stalls, lost fleets, recessions, intercepts and conquests warn; the player's fleets jumping whoosh; weapon fire, hits and explosions in battle (`07 D88`).
- **Effects:** 19 CC0 effects (Kenney packs and OpenGameArt), mono, 32 kHz, 157 KB (`07 D88`).
- **Music as built:** three CC0 beds from OpenGameArt, one per era, each a 56 s loop at 48 kbps mono, 759 KB together. A procedural synthesis (`ui::audio::Synth`) plays where no bed fits: a battle, a crisis, an era without a bed, and the ambience bus always. Music crossfades over 3 s when the mood changes (`07 D88`).
- **Music direction (owner, 2026-11-17):** by mood, leaning orchestral. Exploration, trade and peace get atmospheric beds with orchestral colour, not piano; war, battles, crises and the late eras get orchestral percussion, brass and choir. The procedural synthesis stays as the fallback. The chosen beds will replace the current three under the same budget (about 1 MB of Opus on the web) (`07 D238`).

### The egui UI

- **Architecture.** One `UiStore` holds the latest `SimView`, selection, open screens, pending turn-based commands and per-panel caches keyed by snapshot version, so a frame with an unchanged snapshot touches no sim data. Large tables use `egui_extras::TableBuilder` with uniform-height rows, which skips rows off screen. Tooltips are one overlay fed by the query channel. Screens are tabs in a left rail with the map underneath (`02 §11.5`).
- **Theme.** egui theme through tokens, with Chakra Petch (OFL) as the display face (G1, `00-status`). The tokens gained a spacing scale (2, 4, 8, 16), icon sizes (14, 18, 24) and two commodity tints (`07 D182`).
- **Icons.** One atlas rasterised by `resvg` at start: 15 game-icons.net glyphs (CC BY 3.0, credited) and 26 generated CC0 glyphs (9 for panels and the HUD, 17 for the commodities), 41 in all (`07 D180`). Each commodity names its glyph in `commodities.json` (`icon`), so mods name one too (`07 D181`).
- **Interaction rules.** Every number has a tooltip with its modifier-stack breakdown; every list is sortable and filterable; keyboard: space pauses or ends the turn, 1–5 set speed, F-keys open screens, Esc closes (`01 §16`). Keys are rebindable on the settings screen and saved with the settings (`07 D154`).
- **Notices** (load, save, import) show as 6-second toasts wrapped to the window (`07 D106`). The top bar's buttons wrap to a row of their own at narrow widths (`07 D109`). Empty states were added in M14's UI pass (`07 D153`). An action that cannot be taken is greyed out with the reason (`docs/release/0.1.0-notes.md`).
- **Settings** are saved per user as `key=value` lines: `settings.txt` beside the saves natively, `localStorage` on the web (`07 D88`).

### Screens

From `01 §16`, with the milestone that built each where the sources record it:

| Screen | Contents | Built |
|---|---|---|
| Main menu | new game (galaxy settings, empire creator), load, settings, credits | M14 front end (`07 D97`) |
| Galaxy map | systems, the hyperspace layer, territories, fleets and routes; layers: political, trade (goods in transit), piracy, resources, diplomacy, surveyed | M1, G1, M8c |
| System view | bodies and moons in orbit rings, stations, habitats and domes, fleets, starbase; the primary screen of Era I | M1 onward |
| Planet view | surface hex map; tabs Overview, Build, Market, Expedition | M4a, M4b, M6 |
| Empire | ledger (30-day rolling income and expense), policies, government, civics, factions (1.0), edicts, eras | M4b, M7 |
| Markets | commodities × markets matrix: stock, inbound, price, 30-day trend; flows; shortages; freight utilisation | M5 |
| Research | the tree view: the Kardashev header, the three fields' projects, the goals, search and filters, a card per tech | T4 (built 2026-11-24) |
| Traditions | trees and ascension perks | M7 |
| Ship designer | hull, slots, stats preview, auto-design | M8 |
| Fleet manager | fleets, composition, stance, orders, upgrade, fuel, readiness, supply | M8 |
| Diplomacy | empires, opinion breakdown, treaties, claims, war status | M10 |
| Situation log | anomalies, projects, archaeology, situations, expeditions | M3, M11 |
| Leaders | the "Ldr" tab | M11 (`DESIGN.md`) |
| Captain (persona modes) | persona, reputation, contracts, cargo, purse, holding; rank and assignments | M12a; rank and assignments M16 |
| Battle screen | play, watch, simulate, auto-resolve | C1, C2 |
| Flight view | the persona's fleet in hyperspace | F1 (see [Flight view](Design-Flight-view)) |
| Outliner | planets with alerts, fleets, construction | M4a |
| Notifications | feed with filters; event popups with options | M3 |
| Turn report (TB) | playback scrubber, digest, pending choices | M13 |
| Settings | graphics, audio, autosave, automation defaults, keybinds | G4, M14 |
| Game end | scoreboard and victories | M14 |

### The immersive presentation pass (designed)

The owner's request (2026-11-16): something more immersive, similar to the reference games, spending performance gains on higher-quality, more release-like UX/UI with a strong sci-fi feel (`18`). Her calls (`07 D231`, `18 §0`):

| # | Call | The owner's answer |
|---|---|---|
| 1 | World view | A 3D camera with seamless zoom over the 2D sim |
| 2 | Ships | High-detail 2D sprites in battles, with 3D models on the map |
| 3 | Planets | Full 3D |
| 4 | Backend | `wgpu` (reverses M0's `glow`) |
| 5 | Post-processing | Glow (bloom) on by default (reverses `D37`'s "none") |
| 6 | GPU target | 60 fps at 1440p on the GTX 1060 3 GB |
| 7 | Characters | Animated portraits rendered from 3D, with a plugin hook for live 3D characters |
| 8 | Download | The web streams HD packs |
| 9 | Design language | Command deck |
| 10 | Skins | Frames as moddable data |
| 11 | UI technology | Keep `egui` with a skin layer |
| 12 | Layout | "Rails and outliner" for the empire (28 px status strip, 44 px icon rail with the signal rail, collapsible outliner on the right, selection dossier at bottom-left); the captain's console (ledger header, four mode tabs, dense sortable tables, an inspector, a course strip); type: Chakra Petch for display, IBM Plex Sans and Condensed for body, Plex Mono for numbers; motion: finite pulses that settle, nothing animating while paused and idle; an Okabe–Ito-based status palette with sign and icon redundancy; species skins pass a contrast check in `datacheck` |
| 13 | Timing | Finish the MVP, then the presentation pass |
| 14 | Where | Everything on the test VM (the test host, the GTX 1060 3 GB) |
| 15 | Install | Blender 5.2.3 LTS on the test VM: done 2026-11-16, library capped at 40 GB |

The "command deck" language (`18 §3.5`): dark, slightly translucent glass panels over a blurred backdrop, thin luminous edges and cut corners, two accent colours per empire, panels sliding in under 150 ms, numbers that tick when they change. Readability stays first: UI scale, text size and colour-blind palettes stay, and the 1.0 accessibility pass keeps its place. The proposal drops the 3 MB web budget for a first download of ≤ 25 MB plus streamed HD packs; native ships everything (`18 §3.4`).

**Proposed milestones** (`18 §4`; placement is the spec session's):

| # | Milestone | Contents | Est. LOC |
|---|---|---|---|
| R1 | Renderer foundation | `wgpu` backend behind the trait, WebGL2 fallback, 3D camera, glTF, PBR, instancing, HDR, bloom, tone mapping, new GPU rows | ≈ 2.5k |
| R2 | 2.5D galaxy and system | tilt and orbit camera, seamless zoom galaxy to system to planet, nebula and hyperspace layers, 3D planets with clouds, scattering and lights | ≈ 2.0k |
| R3 | Ship models | Blender → glTF pipeline per lineage and species set, LODs, engine and weapon effects, battles in 3D with the tactical camera | ≈ 1.5k + assets |
| R4 | Portraits and the creator | animated portrait loops from 3D busts, the persona composed from parts, gear shown | ≈ 1.0k + assets |
| U1 | UI design system | skin layer, tokens, type, motion, sounds, species skins as data | ≈ 1.5k |
| U2 | Screen-by-screen pass | every screen in the new language, captain's and state consoles, notification rail, dense tables | ≈ 2.0k |
| A1 | Asset pipeline and environment | the 3D modelling environment of `18 §5` | ≈ 0.5k (scripts) |

Order: A1 and R1 first, together; U1 beside R1; then R2 → R3 → R4 → U2 (`18 §4`). Nothing in the pass changes the simulation, its determinism or its CPU budgets (`18`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| UI framework | egui, to be polished into a high-quality, easy-to-navigate interface | 2026-11-01; restated 2026-11-13 | `07 D2` |
| Art direction | Pre-rendered 3D sprites from scripted Blender, flat vector UI, procedural nebula and planet shaders, aiming at Starsector's look diversity | 2026-11-01; restated 2026-11-13 | `07 D8` |
| Map presentation | Top-down in strategic mode, plus a Starsector-style flight view for travel | 2026-11-13 | `07 D3` |
| Backend | `glow`, by the M0 measurement; `wgpu` removed 2026-11-02 | 2026-11-01 | `07 D19`, `03 §3` |
| Ship lighting | Rendered once from above into atlases, lit by the shader from the system's star | 2026-11-05 | `07 D37` |
| Planets | Procedural GPU textures on first view, lit rotating spheres; Sol's bodies use fixed public-domain textures | 2026-11-11 | `07 D37`, `D41` |
| Post-processing | None for the built look | 2026-11-11 | `07 D37` |
| Art timing | A G-track of four milestones interleaved with the sim; G1 after M8 | 2026-11-08 | `07 D37` |
| GPU budgets | Four GPU rows in `08 §1`, measured on the host's GTX 1060 | 2026-11-11 | `07 D37` |
| Music (first call) | A CC0 bed where one fits, procedural synthesis where none does; a commission or licensed score is a 1.0 call | 2026-11-11 | `07 D37` |
| Planet cache size | 128 KB per layer (256×128 with mips) instead of `10 §2`'s 256 KB per body, to stay under 64 MB | 2026-11-14 | `07 D83` |
| Two atlases | Normal map carries emissive strength in blue; two atlases from three maps, two fetches per pixel | 2026-11-14 | `07 D76` |
| Asset budget room | Only the 2× atlases ship; the normal atlas at 1×; the albedo atlas as lossy WebP | 2026-11-14 | `07 D84`, `D88` |
| Immersive presentation | The fifteen calls of `18 §0`, reversing the `glow` choice and `D37`'s "none" for the coming presentation pass | 2026-11-16 | `07 D231` |
| Music by mood | Leaning orchestral; replaces the soft piano bed; procedural synthesis stays as fallback. Replaces `D37`'s per-era CC0 beds | 2026-11-17 | `07 D238` |

## Not yet built or still open

- **The presentation pass** (R1–R4, U1, U2, A1): designed, not placed in the milestone order (`18 §4`). Timing: after the MVP (`18 §0` call 13).
- **New music beds:** track C sources 6–8 CC0 candidates for the owner to hear before anything is downloaded or changed (`07 D238`).
- **Web GPU rows** are not measured (`08 §1`).
- **Species art:** the portrait ids are a hook left by M14 (`05 M14`).
- **Free-text names:** names come from each species' list; no free-text names yet (`DESIGN.md`, `07 D97`).
- **Accessibility pass:** 1.0, in M23 (`05`).
- **Owner review:** the `07` call sets still open for review include `D42` (G1's calls) (`DESIGN.md`).

## Discrepancies noted

- **Has the owner heard the audio?** The `DESIGN.md` pillar row says "G1–G4 built (audio not yet heard by the owner)", while `07 D238` (2026-11-17) says "The owner heard the G4 music for the first time" and chose new music direction.
- **The tracker's graphics pillar against `D231`.** `DESIGN.md`'s pillar row describes the design as "Lit Blender sprites ..., GPU-generated planets ..., no post-processing", while `07 D231` (2026-11-16) chose bloom on by default, `wgpu`, 3D on the map and full 3D planets. The row may describe the built state rather than the decided design; it is not marked either way.
- **What "0.2" means.** `18 §3.6`'s call 13 describes "finish the MVP as built (an internal 0.1); then a presentation pass (0.2, the milestones below)", and the owner chose "Finish the MVP, then the presentation pass". Release 0.2.0 was deployed on 2026-11-23 with gameplay features (rebellion, employee modes, intrigue, trade fleets) and none of R1–U2 (`docs/release/0.2.0-notes.md`, `00-status`).
- **Galaxy map layers.** `01 §16` lists the galaxy map's layers as "political, trade (with goods in transit), piracy, resources, diplomacy, surveyed", while `02 §11.2` lists draw layers (background, territory, hyper, trade, systems, fleets, transit, overlays, labels). Which of `01 §16`'s toggle layers are built is not recorded.

## Sources

- 01 Game design: UI screens
- 02 Architecture: rendering and UI
- 03 Tech stack: graphics and UI stack, assets
- 05 Milestones (G1–G4, M14)
- 08 CPU budget: budgets
- 10 Graphics scoping: approach, budgets, milestone track
- 18 Presentation and UI: the owner's calls, milestones
- 07 Decisions: `D2`, `D3`, `D8`, `D19`, `D37`, `D41`, `D42`, `D76`, `D83`, `D84`, `D87`, `D88`, `D97`, `D106`, `D153`, `D154`, `D180`–`D182`, `D231`, `D238`
- 00 Status
- DESIGN tracker
- Release notes 0.1.0, 0.2.0

## Related pages

- [Overview](Overview)
- [Architecture and performance](Design-Architecture-and-performance)
- [Modding and data](Design-Modding-and-data)
- [Roadmap and status](Design-Roadmap-and-status)
- [Flight view](Design-Flight-view)
- [Combat](Design-Combat)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Time and modes](Design-Time-and-modes)


---

# Roadmap and status

**Status:** in progress, as of 2026-11-23. The MVP milestones are built and M14 (MVP release 0.1) has two open boxes; several 1.0 milestones are built or under way; releases 0.1.0 and 0.2.0 are published privately.

This page is the build plan of Magna Dominia Astralia: what belongs to the MVP, 1.0 and later; every milestone with one line and its state; how the work is split across concurrent tracks; and what has been released and where. The game is built milestone by milestone by implementer sessions working from the spec, each milestone gated on the test host before the spec session merges it into `main`.

## How it works

### Scope tiers

From `SPEC.md §4`:

| Tier | Definition | Contents (summary, as written in `SPEC.md`) |
|---|---|---|
| **MVP** | A complete, winnable vertical slice an outside tester can play for 2–3 hours, natively or in a browser | galaxy generation with templated home systems, a pre-FTL opening with domes, habitats and in-system logistics, exploration, colonies, pops and jobs, 10 commodities with per-market prices and goods in transit, hub trade routes, piracy, planet surfaces with auto-expeditions (lite), tech (≈ 60), 3 tradition trees, ships, fleets and abstract combat, 8 AI empires, basic diplomacy and war, leaders, both time modes, save and load, score victory |
| **1.0** | Full feature set in this spec | full tech (≈ 250) and tradition set, full surface feature events and terraforming, factions and ethics drift, situations and crises, federations, trade fleets and tariffs, Galactic Exchange, megastructures, ascension, endgame crisis, modding via data packs |
| **Post-1.0** | Out of scope but the architecture must not preclude it | lockstep multiplayer, scripted mods (WASM or Lua), mobile layout, Steam release of the native build |

Later decisions moved items between tiers or past the `SPEC.md` numbers:

- **Commodities:** 17, from E1's resource model (`07 D73`; `D4` deferred the count to E1).
- **Techs:** an open tree of about 400 techs in eight tiers; 399 in T1's content (`07 D278`, `05 T`).
- **Megastructures:** kilo, mega and giga tiers in MVP (`07 D70`), built in S1 and S2.
- **Combat:** one deterministic tactical engine (play, watch, simulate) with a calibrated resolver, in place of abstract combat alone (`07 D48`, C1, C2).
- **Code mods:** sandboxed WebAssembly in 1.0, not post-1.0 (`07 D49`).
- **Built ahead of their 1.0 slot:** factions as FC1–FC4 (replacing M15's factions), employee modes (M16), trade fleets, subsidies, the Galactic Exchange and smuggling (M17), pacts (M18a) and gateway trade (M19a).

### Milestones

States follow `DESIGN.md`'s key: **built** (merged into `main` and verified), **in progress** (on a track, or partly merged), **designed** (specified, not started). Dates are merge or delivery dates from `00-status`.

**MVP and its additions** (the order in `05` and `00-status`):

| Milestone | One line | State |
|---|---|---|
| M0 | Workspace, pipeline, gate, backend measured (`glow`) | built 2026-11-01 |
| M1 | Galaxy generation and the map | built 2026-11-02 |
| M2 | Time, commands, saves: the simulation engine | built 2026-11-02 |
| M3 | Interstellar exploration and events v0 | built 2026-11-02 |
| M4a | Hex surfaces and colonies | built 2026-11-02 |
| M4b | Pops, the daily economy and the ledger | built 2026-11-03 |
| M5 | Markets, trade network, piracy | built 2026-11-03 (verified 2026-11-04) |
| M6 | Expeditions and features | built 2026-11-05 |
| M7 | Technology, traditions, modifiers, eras, policies | built 2026-11-06 |
| M8 | Ships, combat, starbases, armies | built 2026-11-08 |
| M8b | Starbases as colonies, eight sizes, physical logistics (`07 D39`) | built 2026-11-09 (verified 2026-11-11) |
| G1 | Look and pipeline: theme, icons, star shader, nebula, territory | built 2026-11-12 |
| M8c | Hyperspace: no lanes, the weather grid, corridors, deep space (`07 D43`) | built 2026-11-13 |
| C1 | Small fleet actions on the tactical engine | built 2026-11-13 |
| C2 | Large fleet actions in squadrons | built 2026-11-13 |
| M9 | AI empires and headless balance | built 2026-11-13 |
| P1 | Economy speed-up, exact single-thread rework (`07 D51`) | built 2026-11-14 |
| H1 | Hull catalogue (64 military hulls) and fleet hierarchy | built 2026-11-14 |
| P2 | Moddability and simplicity: registries, saves by name, mod system | built 2026-11-14 |
| S1 | Kilo- and megastructures, ruins | built 2026-11-14 |
| G2 | Ships and stations as lit Blender sprites | built 2026-11-14 |
| S2 | Gigastructures | built 2026-11-14 |
| G3 | Planets and surfaces | built 2026-11-14 |
| G4 | Motion and sound | built 2026-11-14 |
| M11 | Leaders, situations, archaeology, first contact, MVP events | built 2026-11-14 |
| M13 | Turn-based polish and parity | built 2026-11-14 |
| E1 | The resource model, 17 commodities | built 2026-11-14 |
| GC1 | Ground combat on the hex surface | built 2026-11-14 |
| M10 | Diplomacy and war | built 2026-11-14 |
| F1 | The flight view | built 2026-11-15 |
| M12a | Persona: framework, character, personal fleet | built 2026-11-15 |
| M12b | Persona: service, people and legacy | built 2026-11-16 |
| HP1 | Huge-scale economy performance (staggers, dispatch by per-hub source order) | built 2026-11-17 |
| M14 | MVP release 0.1: front end, UI pass, audio, packaging, balance and performance passes, smoke | in progress: balance, budget and smoke boxes ticked; external testers and the production web deploy open |
| T | The tech tree rework: T1 schema and content | built 2026-11-23 |
| T | T2 mechanics | built 2026-11-24 (rating, latch, age gates, diffusion, `research.speed`, idle bank cap, AI great works; insight, reverse engineering, the tech keys' consumers) |
| T | T3 AI | built 2026-11-24 (research by goals) |
| T | T4 UI | built 2026-11-24 (the tree view) |
| T | T5a hooks and bench rows | built 2026-11-25 (six trade and intrigue keys, two bench rows, the pace line; awaiting merge) |
| T | T5b the Dyson line and the pace | designed (`07 D278 §16`; five points open with the owner) |

**1.0 milestones** (`05`, 1.0 table):

| Milestone | One line | State |
|---|---|---|
| M15 | Factions and ethics drift; factions replaced by FC1–FC4 (`docs/19 §9`, `07 D214`), ethics drift stays here | ethics drift designed |
| FC1 | Parties, blocs and wealth classes | built 2026-11-19 |
| FC2 | Notables | built 2026-11-20 |
| FC3 | Patron notable, coalition consent, civil-war sides | built 2026-11-20 |
| FC4 | Intrigue and lords | built 2026-11-22 |
| M16 | Persona: employee modes (commander and leader kits, ranks by favour) | built 2026-11-21 |
| M17 | Trade fleets, subsidies and embargo, Galactic Exchange, smuggling (M17a–d, `07 D275`) | built: M17a and M17b 2026-11-22, M17c and M17d 2026-11-23 |
| M18 | Pacts, vassals, federations, laws, intel in five phases (`07 D279`) | in progress: M18a (pacts) built 2026-11-23, M18b (vassals) 2026-11-24, M18c (federations) built 2026-11-24 on track C, awaiting merge; M18d–e designed |
| M19 | Gateway trade links and the AI's megastructure advisor (`07 D283`) | in progress: M19a (gateway trade) built 2026-11-23; the rest designed |
| M20 | Crises: marauders, awakened empire, endgame crisis (3 variants) | designed |
| M21 | Modding data packs: pack manager, validation UI (see [Modding and data](Design-Modding-and-data)) | designed |
| M22 | Full content: techs, traditions, 300 events, situations, species packs, portraits, music | designed |
| M23 | 1.0 release: balance, performance (stress year ≤ 15 s), accessibility, itch.io and Steam-ready build | designed |

**Other designed work, not yet placed or started:**

| Work | One line | Source |
|---|---|---|
| Presentation pass (R1–R4, U1, U2, A1) | `wgpu`, a 3D camera with seamless zoom, 3D models and planets, bloom, portraits, the command-deck UI; after the MVP | `07 D231`, `18 §4` |
| Starbase stations | Starsector-style stations of sections replacing the batteries; capture makes the loser's fleets retreat | `07 D284`; milestone not yet placed (`DESIGN.md`) |
| Great works (O1) | A wider roster of structures under the four scales Prima, Maiora, Magna and Grandia Opera Astralia, scoped in a separate session | `07 D278`, `DESIGN.md` |
| Terraforming (TF1) | Handed from M19 to `docs/17`'s TF1 | `07 D283` |
| Core offloading phases | Parallel passes through `sim::par` beyond the harness, and the Cores setting | `07 D169`, `D240`, `D261` |
| Windows installer | Inno Setup, per-user, unsigned at first | `07 D270` |

### Tracks

From 2026-11-13 the work runs as concurrent tracks, each an implementer session in its own git worktree and its own host folder (`scripts/gate-remote.ps1 -Track <name>`), with benches locked (`07 D49`). The spec session verifies each milestone with its own host run and merges it into `main` itself, never pushing (`07 D49`).

| Track | Branch | Work recorded in the sources |
|---|---|---|
| **P** | `track-p` | P2's foundation first (merged early so the others rebase onto it), P1, the rest of P2; then S1, S2, M11, M13, the economy balance pass, M14's front end, rebindable keys, stuck-build work, performance passes (`sim::par`, the war-storm bench, release prep P-REL, P-DISP) and the 0.1.0 and 0.2.0 packaging (`00-status`, `DESIGN.md`) |
| **M** | `track-m` | M9 → E1 → M10 → M12a → M12b; economic victory, the §18 balance boxes and the pre-tester soak; the AI economic model; M17a–d; M19a (`00-status`, `DESIGN.md`) |
| **C** | `track-c` | C1 → C2 → H1 → G2 → G3 → G4 → GC1 → F1; M13's web zip box; M14's release checks and icon atlas; the pirate fix and HP1; FC1–FC4; M16; M18a (`00-status`, `DESIGN.md`) |
| **T** | `track-t` | the tech tree rework T1–T5, from 2026-11-21 (`07 D278`) |

Rules the tracks share: each track has its own range of decision ids (for example track P's D150–D169, then D260–D279; `07 D169`, `D260`); at most two track gates run at once, and a bench takes a host-wide mutex (the owner, 2026-11-22, `scripts/gate-remote.ps1`); `gate-remote -Bench` also runs smoke and balance, so no track merges an expansion regression (`07 D86`).

### Releases

| Release | What happened | Source |
|---|---|---|
| **0.1.0 prepared** | 2026-11-19: the version in Cargo, `scripts/package.ps1` building the Windows zip (8.0 MB, with CREDITS, CHANGELOG, licences and THIRD-PARTY for 528 crates) and the web zip (5.8 MB) on the host; not published | `07 D263`, `00-status` |
| **0.1.0 published privately** | 2026-11-19, on the owner's go: tag `v0.1.0` at `3c9094a`; a GitHub release in the private repository with the Windows zip (8.1 MB) and the web zip (5.9 MB), MIT `LICENSE` in both; the web build deployed to the tester room (sereneblossom.gay/tester, invite code only) | `00-status`, `07 D264` |
| **0.1.0 rebuilt in place** | 2026-11-20: no testers yet, so 0.1.0 was rebuilt from `main` with the same version rather than cut as 0.1.1; the tag moved, the zips were replaced and the tester room redeployed. Saves from the 19 November build load in it | `DESIGN.md`, `docs/release/0.1.0-notes.md` |
| **0.2.0 deployed** | 2026-11-23, on the owner's yes: annotated tag `v0.2.0` on `3230cac` (main `275247d` plus the version bump, so without M17b, M17c, M18a, T1 or M17d); the private GitHub release with Windows, web and Linux zips; the tester room updated. Notes approved 2026-11-22. Saves from 0.1.0 load in it | `00-status`, `docs/release/0.2.0-notes.md` |

**0.1.0's content** (`docs/release/0.1.0-notes.md`): galaxies Tiny to Huge with up to 24 AI empires and a Sol start; hex colonies; pops by species and stratum; physical markets, hubs, freight and piracy; ship design, fleets, a resolver with an optional tactical view, armies and invasions; wars, treaties and commercial pacts; politics and notables (from the 2026-11-20 rebuild); domination and end-year score victories; saves, autosaves, export and import; mods over the JSON content.

**0.2.0's additions** (`docs/release/0.2.0-notes.md`): rebellion and civil war; the Commander and Leader starts in an AI-run empire; intrigue and lords; trade fleets with insurance; colony pacing with civil upkeep; food supply and food buffers; AI improvements; trade routing speed-ups. Known issue: Huge at the design scale runs about 3.0 s a game year with 99 days in 100 under 9.4 ms. Until 1.0, a new minor version may break saves.

### Distribution

| Aspect | Current rule | Source |
|---|---|---|
| Target platforms | Native primary, though the web is the goal; since 2026-11-18 the native client is the real target and the web build an optimisation showcase that stays single-threaded | `07 D18`, `D240` |
| Storefront | The sereneblossom.gay storefront | `07 D9` |
| Repository | Private GitHub repository `magna-dominia-astralia` (renamed on 2026-11-24 from `magnum-dominium-astrale`) | `07 D10`, `DESIGN.md` |
| Testing room | The web build in the invite-only tester room at sereneblossom.gay/tester | `00-status` |
| Windows | x64 zip and (designed) installer from `scripts/package.ps1` on the host; unsigned, so SmartScreen asks once | `07 D270`, `03 §9` |
| Linux | x86-64 zip from the release workflow, with `libasound2-dev`; built, not tested | `07 D270` |
| macOS | Out of the release matrix for 0.1.x: it needs a Mac to test and sign | `07 D270` |
| Web | `trunk build --release`; PWA manifest and a network-first offline service worker; IndexedDB saves | `03 §9`, `07 D152` |
| Licence | MIT for the code; assets keep their own licences (CC0 and those in `CREDITS.md`) | `07 D264` |
| Later channels | itch.io as a 1.0 addition; Steam post-1.0 | `03 §9`, `SPEC.md §4` |
| Version shown | `CARGO_PKG_VERSION` in the main menu; saves record it | `03 §9` |

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Spec confirmed for M0 | The owner confirmed D18-A and D8-A and accepted every other recommended option | 2026-11-01 | `07` top, `00-status` |
| Name | Magnum Dominium Astrale (`D1`), then renamed Magna Dominia Astralia | 2026-11-13; renamed 2026-11-16 | `07 D1`, `D233`, `DESIGN.md` |
| Distribution | The sereneblossom.gay storefront. Replaced `D9`'s first recommendation (GitHub Releases plus Cloudflare Pages) | 2026-11-13 | `07 D9` |
| Concurrent tracks | P, M and C after M8c, each in its own worktree; the spec session merges, never pushes | 2026-11-13 | `07 D49` |
| Build then test | The owner tests at the end; implementer sessions do not block on per-milestone play-tests | 2026-11-03 onward | `00-status` passes ("the owner tests at the end") |
| Autonomous MVP drive | The spec session drives toward the MVP in a loop, logging every ambiguous call in `DESIGN.md` for review | 2026-11-14 | `DESIGN.md` |
| Presentation timing | Finish the MVP, then the presentation pass | 2026-11-16 | `07 D231` call 13 |
| Huge judged at the design scale | 3 s a year and 12 ms at ≈ 450–1,000 colonies | 2026-11-18 | `07 D239` |
| Licence | MIT | 2026-11-19 | `07 D264` |
| 0.1.0 in place | Rebuilt as 0.1.0 rather than 0.1.1 while no testers exist | 2026-11-19 | `DESIGN.md` |
| Release workflow | Windows only from the host's `package.ps1`; macOS out for 0.1.x; Linux fixed and shipped untested; a real Windows installer "at some point" | 2026-11-20 | `07 D270` |
| Tech tree rework | Open tree of about 400 techs, built by track T in T1–T5 | 2026-11-21 | `07 D278` |

## Not yet built or still open

- **M14's open boxes:** three external testers completing a game to year 100 without a blocking bug, at least one on the web; the web build deployed to production, native binaries on a tagged release, and save export and import checked on Chrome, Firefox and Safari (owner-manual) (`05 M14`).
- **Storefront mechanics** (builds, licence keys, web hosting) for `D9` are an open owner question (`DESIGN.md`).
- **Production web deploy and the tag:** track P prepared the internal half of release prep while the owner decides on the tag, the production web deploy and the order (`DESIGN.md`, 2026-11-19).
- **Windows installer:** designed (`07 D270`), not recorded as built.
- **Code signing:** a separate owner call, because it costs money (`07 D270`).
- **macOS build:** out for 0.1.x (`07 D270`).
- **T5b; M18's later phases; M19's later phases; M20–M23; ethics drift (M15):** designed.
- **Presentation pass, starbase stations, great works, terraforming (TF1):** designed, not placed.
- **Open owner questions in the tracker** (`DESIGN.md`): the `07` call sets D34–D36, D38, D40, D42, D46 for review; the Nicoll-Dyson Lance's rules; lineage names; whether `CLAUDE.md` should say nine crates; the resolver's structural term; the storefront's mechanics; `docs/17`'s civic count and the lore bible; F1's other calls; Starsector-style extra commodities; how the eras sit beside the three Kardashev ages, and the great works' roster.

## Discrepancies noted

- **Where the MVP ends.** `05`'s preamble says "Order: M0 → M13 is the MVP (release 0.1). M14 → M21 is 1.0", while `05`'s own headings name M14 "MVP release 0.1" and its 1.0 table lists M15–M23.
- **0.1.0's platforms.** `00-status` records the 2026-11-19 publication with the Windows and web zips only ("No Linux build yet (the host's WSL has no Rust toolchain)"), while `docs/release/0.1.0-notes.md` lists a Linux zip "built by GitHub Actions and not tested". The notes also describe a 2026-11-20 in-place rebuild, and `07 D270` (2026-11-20) fixed the Linux workflow, so the Linux zip may date from the rebuild; the sources do not say.
- **Distribution channel.** `07 D9` (owner, 2026-11-13) chose the sereneblossom.gay storefront, while `03 §9` still describes "the web build ... → Cloudflare Pages" and `03 §8` has `main` deploying to production on Cloudflare Pages. Both releases so far are a private GitHub release plus the invite-only tester room.
- **What "0.2" means.** `18 §3.6` describes the presentation pass as "0.2, the milestones below", and the owner chose "Finish the MVP, then the presentation pass" (`07 D231`). Release 0.2.0 shipped gameplay work and none of the presentation milestones.
- **The status doc's header.** `00-status.md`'s first line says "Updated 2026-11-09 (pass 22: M0–M8b delivered ...)", while its body records merges through 2026-11-23.

## Sources

- SPEC.md, scope tiers
- CHANGELOG.md, README.md
- 05 Milestones, 1.0 milestones
- 00 Status
- DESIGN tracker
- 03 Tech stack: CI, distribution
- 18 Presentation and UI: milestones
- 07 Decisions: `D1`, `D9`, `D10`, `D18`, `D49`, `D70`, `D73`, `D231`, `D233`, `D239`, `D240`, `D263`, `D264`, `D270`, `D275`, `D278`, `D279`, `D283`, `D284`
- Release notes 0.1.0, 0.2.0; `docs/release/README.txt`
- `scripts/gate-remote.ps1`

## Related pages

- [Overview](Overview)
- [Architecture and performance](Design-Architecture-and-performance)
- [Modding and data](Design-Modding-and-data)
- [Presentation and UI](Design-Presentation-and-ui)
- [Time and modes](Design-Time-and-modes)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Starbases](Design-Starbases)
- [Megastructures](Design-Megastructures)
- [Progression](Design-Progression)
- [Factions and characters](Design-Factions-and-characters)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Trade and logistics](Design-Trade-and-logistics)
- [Events and crises](Design-Events-and-crises)
- [AI and victory](Design-Ai-and-victory)
