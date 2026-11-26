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
