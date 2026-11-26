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
