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
