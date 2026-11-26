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
