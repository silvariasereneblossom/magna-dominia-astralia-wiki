# Galaxy and hyperspace

**Status:** built, as of 2026-11-23. Galaxy generation in M1, hyperspace and deep space in M8c (verified 2026-11-13); lock trade links in M19a.

The galaxy is generated as a pure function of the seed and the settings. There are no hyperlanes: star systems sit in hyperspace, an open and dangerous medium that fleets cross from any system to any other along routes planned at departure. Its weather (storms and currents) is fixed by the seed and the month, so travel is forecast exactly but never stable. Empires build beacon corridors that the shifting currents keep bending, and later techs add warp and buildable wormholes (Hyperspace Locks). Beyond charted space, deep space grows more dangerous and more rewarding with distance.

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
| 4 | Lock Theory (physics) | a Hyperspace Lock megastructure at an own starbase of size 6 or more |

**Hyperspace Locks** (`07 D46`, moved unchanged onto the megastructure model in S1):

- Cost 5,000 alloys and 800 parts from the starbase's market over 1,080 days; upkeep 30 credits and 4 parts.
- A fleet in a Hyperspace Lock system ordered to another own Hyperspace Lock system crosses in 1 day after the windup.
- A fleet may also cross between its own Hyperspace Lock and the Hyperspace Lock of an empire it holds open borders with, as between its own (`07 D114`).
- **Trade links** (`07 D283`, built in M19a on 2026-11-23): an empire's own hub pair with usable Hyperspace Locks at both hubs' systems (its own, or an open-borders partner's) takes the jump's days when quicker than the flown route, with no piracy and no loss on the jump, freight paid on the shorter days only, and closed by a blockade at either gate. A Hyperspace Lock serves the hub at its own system. Pact routes and the Galactic Exchange's legs still fly.

There are no natural wormholes: they were removed with the lanes, and Hyperspace Locks are the buildable wormholes (`07 D46`).

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
| Late travel | All three tiers plus buildable Hyperspace Locks as a fourth tier | 2026-11-12 | `07 D43` |
| Deep space | Danger rising with depth; three deep factions; caches, stashes and rich fields with deep outposts; built in M8c | 2026-11-12 | `07 D45` |
| Grid size | Cells of `d_min / 2` with a 12-cell margin (69 × 69 at Medium); replaces D43's "about 128 × 128 cells" | 2026-11-13 | `07 D46` |
| Piracy regions | 6 × 6 cells; replaces D43's 16 × 16 | 2026-11-13 | `07 D46` |
| Long routes | Cells after the next month take the next month's weather; replaces D43's exact forecast | 2026-11-13 | `07 D46` |
| Trade fields | Each hub recomputed on its own day of the month; replaces D43's one hub a day | 2026-11-13 | `07 D46` |
| Freight | `amount × (days + 2 × 2.5) × 0.05`, half on corridor share; replaces D43's `amount × routeDays × 0.05` | 2026-11-13 | `07 D46` |
| Storms slow | A storm adds its strength to the drag | 2026-11-13 | `07 D46` |
| Clouds after empires | Clouds rolled after placement and moved off homes | 2026-11-13 | `07 D46` |
| Natural wormholes | Removed; Hyperspace Locks are the wormholes | 2026-11-13 | `07 D46` |
| Allied Hyperspace Locks | Read as open borders, since alliances were 1.0 at the time | 2026-11-14 | `07 D114` |
| Hyperspace Anchor | Holds every corridor in its radius, not only its owner's | 2026-11-14 | `07 D78`, `DESIGN.md` |
| Hyperspace Lock trade | A route shortcut for an empire's own hub pairs, as the Mass Driver's | 2026-11-23 | `07 D283` |

## Not yet built or still open

- The `07 D46` call set (M8c's calls beyond the spec) is listed in `DESIGN.md` as still open for the owner's review.
- The `01` passages that M8c converted from hops by ratio (5 route days per old hop, 2 cells per jump; listed at the end of `07 D46`) await the owner's eye.
- `05 M8c`'s bench box (a route plan across Huge ≤ 200 µs × calibration, the monthly raster ≤ 2 ms, the full tick within 5% of M8b's) is unchecked: the full tick was 15–17% over M8b's, carried to P1.
- No AI builds Hyperspace Locks yet; the AI's megastructure advisor is M19c (`07 D283`).
- Storm suppression in the third age and storm-forecast techs are listed in the tech tree rework as future hooks (`07 D278`).

## Discrepancies noted

- **Allied Hyperspace Locks.** The `01 §2.1` tiers table still says "allied Hyperspace Locks join with alliances (`05 M10`)". `07 D114` (2026-11-14) reads this as open borders, because alliances were 1.0 then. Alliances were built in M18a (merged 2026-11-23, `07 D279`). The sources do not say whether alliance members' Hyperspace Locks now join as well as open-borders partners'.
- **Commodity names in hyperspace rules.** `01 §2` step 5 and `01 §2.1` still name `crystal` deposits and `parts` in the corridor, Hyperspace Lock and outpost bills, while `01 §7` records that crystal became rare ore and parts became components in E1 ("saves map them in place"). The numbers are unaffected; the names in `01 §2`–`§2.1` predate E1.

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
