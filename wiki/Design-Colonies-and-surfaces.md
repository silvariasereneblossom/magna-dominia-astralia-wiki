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
