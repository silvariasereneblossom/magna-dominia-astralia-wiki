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
