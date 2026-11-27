# Megastructures

**Status:** built, as of 2026-11-26. The shared model with 8 Prima and 8 Maiora works and ruins in S1, the 5 Magna works and their ruins in S2 (both 2026-11-14), Hyperspace Lock trade links in M19a and the AI's orders for great works in T2 (both 2026-11-23); eleven works took the game's own names on 2026-11-26; `07 D277`'s first phase, O1 (2026-11-27), renamed the tiers to scales, raised the registry to 128 works, made five works produce goods at a place and removed the Nicoll-Dyson Lance; O2 (2026-11-27) added the 22 Prima and Maiora works below; O3–O6 are still to come.

Megastructures, the great works, are large staged constructions in three scales: Prima, Maiora and Magna Opera Astralia. The first set was inspired by the Stellaris mod Gigastructural Engineering & More; the names and identities here are the game's own. Forty-two of them are data in `megastructures.json`: they place by rules on stars, bodies or starbases, draw each stage's bill from their own system's markets like any construction, and act through modifier rows and a few code hooks (corridor anchors, yards, mass-driver links, habitats, the Hyperspace Lock). Ruined structures lie mostly in deep space and the abyss, guarded by automata, and can be restored at a quarter of their cost.

## How it works

### The shared model

One data-driven model serves every scale (`13 §6`, `07 D78`):

| Part | Rule |
|---|---|
| Scale | Prima, Maiora, Magna or Grandia Opera Astralia (`scale` in the content since O1; `tier` and `kilo`, `mega`, `giga` still read, `07 D277`) |
| Placement | on a star, a body or the system's starbase: body kinds, classes and size; the body's colony (any, none or own); star classes; the cell's depth; the starbase's least size; per-system and per-empire limits; one structure a body (`07 D78`) |
| Prerequisites | `requires` another of the builder's structures, active in the same system; `consumes` replaces it on completion (`07 D81`) |
| Stages | each stage and upgrade has a bill, days, upkeep (credits and commodities), modifier rows (`07 D78`), and an `output` of goods landed at the system's own starbase market, times the star's luminosity with `lum` and the body's size / 16 with `bySize` (`07 D277`, `D292`) |
| Failure and decay | optional yearly roll (ruin or decay) (`07 D78`); with `storm`, its odds scaled by the storm on the system's cell in the month of the roll (`07 D292`) |
| Hooks | code opted into by data (below) |
| Ruins | a ruin weight per structure (`07 D78`) |
| Settings switch | one per structure in the game settings; at most 128 structures, so the switch is one `u128` (`07 D78`, `D277`) |

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
| `gateway` | crossing days; the Hyperspace Lock keeps M8c's `gate` column, so hyperspace is unchanged |
| `anchor` (radius, depth) | caps depth at the Chebyshev distance less the radius; drift skips cells within the radius; holds every corridor in reach, not only its owner's |
| `yard` (class, slots, repair) | on the system's starbase |
| `driver` (fraction) | a new Mass Driver links to the empire's oldest unlinked driver in another system; routes between the two systems take the fraction of their days |
| `habitat` | founds a habitat colony around the body; on a star-anchored structure, around the system's star (the Ringworld) |
| `batteries` | adds batteries to the system's starbase |

### Prima Opera Astralia (8)

| Structure | Placement | Bill | Days | Upkeep /30 d | Effect |
|---|---|---|---|---|---|
| Belt Foundry | an asteroid belt, no colony | 2 stages × 400 alloys, 40 components | 240 | 3 cr a stage | 30 ore and 8 alloys a month a stage at the system's starbase market (`07 D277`) |
| Belt Redoubt | an asteroid belt, no colony | 500 alloys, 60 components | 240 | 2 cr, 0.5 components | +0.05 piracy security and 4 batteries on the system's starbase |
| Hanging Garden | a barren, molten, toxic or frozen body, no colony | 600 alloys, 60 components, 200 food | 300 | 3 cr | founds a habitat-like colony around the body |
| Atmospheric Processor | a planet with an own colony | 500 alloys, 50 components, 200 gas | 360 | 3 cr, 1 gas | +3 housing and +10% growth on the colony below |
| Cloudhold | a gas giant, no colony | 700 alloys, 80 components, 100 gas | 360 | 4 cr | a colonisable gas giant (`13 §6`): its habitat hook founds a habitat around the body |
| Coronal Collider | a star | 800 alloys, 120 components, 100 rare ore | 360 | 5 cr, 1 rare ore | +20% physics, each further one an empire finishes adding half of the one before (+10%, +5%: `stack` 0.5, `07 D278 §16`); 1 exotic matter a month at the system's starbase market (`07 D277`); a 2% yearly failure, which ruins it, scaled by the storm on its cell in the month of the roll, so never in a calm one (`07 D292`). An AI orders at most three (`worksGain`, `07 D293`) |
| Crown City | a planet of size 8–30, no colony | 900 alloys, 100 components, 200 goods | 420 | 5 cr | +6 housing, +2 amenities and +1 building slot on its habitat (`07 D292`) |
| Beacon Spire | a starbase of size ≥ 2 | 300 alloys, 30 components | 180 | 2 cr | a small hyperspace anchor: radius 3, holding corridors, no depth change |

### Maiora Opera Astralia (8)

| Structure | Placement | Stages × bill | Days a stage | Upkeep /30 d | Effect |
|---|---|---|---|---|---|
| Hyperspace Lock | a starbase of size ≥ 6, charted space (depth 0) | 1 × 5,000 alloys, 800 components | 1,080 | 30 cr, 4 components | fleet crossings between Hyperspace Locks (`01 §2.1`); trade shortcut (M19a, below) |
| Hyperspace Anchor | a starbase of size ≥ 4 | 1 × 2,500 alloys, 300 components, 200 rare ore | 720 | 15 cr, 2 components | radius 6: holds every corridor and lowers depth |
| Shipyard Ring | a starbase of size ≥ 4 | 1 × 3,000 alloys, 400 components, 100 machine tools | 720 | 20 cr, 2 components | builds super-capitals; +2 yard slots; repair × 2 |
| Mass Driver | a starbase of size ≥ 3 | 1 × 1,500 alloys, 200 components | 540 | 10 cr, 1 fuel | a linked pair's routes take 0.25 of their days; +5% freight |
| Dyson Swarm | an O/B, A, F, G or K star; known from the start (T5b, `07 D278 §16`) | six sections: 400 / 600 / 1,000 / 1,600 / 2,200 / 2,500 alloys and a tenth as many components | 240 / 240 / 300 / 360 / 360 / 360 | 2 / 3 / 5 / 8 / 10 / 12 cr | each section lands energy at the system's starbase market, times the star's luminosity: 40 / 60 / 120 / 230 / 350 / 400 a month (1,200 in all on a G star); sections two to six wait for Orbital Solar Arrays, Beamed Power, Stellar Collectors, Statite Engineering and Swarm Fabrication; finished, it is the Type I structure |
| World Quarry | a barren, molten, toxic or frozen body of size 8–30, no colony | 2 × 1,200 alloys, 150 components, 50 machine tools | 360 | 6 cr (and 0.5 machine tools in the first stage) | per stage 60 ore and 6 rare ore a month at the system's starbase market, times the body's size / 16 (`07 D277`, `D292`); 5% yearly decay |
| Core Auger | a molten body, no colony | 1 × 1,500 alloys, 200 components, 50 machine tools | 540 | 8 cr, 0.5 machine tools | 20 rare ore a month at the system's starbase market (`07 D277`) |
| Athenaeum | a starbase of size ≥ 4; 1 per empire | 2 × 2,000 alloys, 300 components, 200 rare ore | 540 | 15 cr a stage | +15% to each research field and +10% skill gain; a quantum-core upgrade (+10% physics) |

### Magna Opera Astralia (4)

Each Magna work is one per empire, with long staged builds and large upkeep (`13 §6`, `07 D82`).

| Structure | Placement | Stages × bill | Days a stage | Upkeep /30 d | Effect (`07 D82`) |
|---|---|---|---|---|---|
| Matrioshka Brain | a star | 3 × 4,000 alloys, 600 components, 400 rare ore | 720 | 30 cr (and 2 rare ore in the first stage) | +25% to each field per stage, then −10% tech cost and +5% build speed |
| Dyson Sphere | an O/B, A, F, G or K star; 1 per empire; requires and consumes the finished Dyson Swarm | 3 × 5,000 alloys, 500 components, 200 machine tools | 720 | 25 cr a stage | each stage lands 8,000 energy a month at the system's starbase market, times the star's luminosity; the third closes it and cools the system's planets (habitability −20) until the Dyson Aperture upgrade (3,000 alloys, 600 components, 200 machine tools, 720 days; tier 6) gives it back; closed, it is the Type II structure |
| Ringworld | an F, G or K star | 3 sections × 6,000 alloys, 600 components, 500 goods | 900 | 20 cr a section | a habitat around the star: +15 housing per section, +2 building slots in the first two, +5 amenities in the last |
| Sun Dredge | a star | 2 × 3,500 alloys, 500 components, 200 machine tools | 720 | 20 cr (and 1 machine tool in the first stage) | per stage 80 gas a month at the system's starbase market, and 4 exotic matter more in the second, times the star's luminosity; +5% physics in the second (`07 D277`, `D292`) |

Placement, bills, days and upkeep are from `data/content/megastructures.json`; the commodity names are E1's (parts became components, crystal rare ore, machinery machine tools; `07 D57`). `07 D79` gives the S1 ranges as Prima works 300–900 alloys, 30–120 components, 180–420 days, 2–5 credits upkeep, and Maiora works 1,200–3,000 alloys and 150–400 components a stage, 360–720 days a stage, 6–20 credits upkeep. Effects are first guesses for balance passes (`07 D79`, `D82`).

### O2's Prima and Maiora works (`07 D277 §4`–`§5`, `§22`)

| Work | Where | Stages | What it does |
|---|---|---|---|
| Heliostat Array | a star with light | 2 | energy 25 × lum a stage (under the Swarm section its tech opens) |
| Flux Tether | a gas giant, no colony | 1 | energy 90; 1% a year to decay |
| Mantle Tap | a molten body, no colony | 1 | energy 60 and rare ore 2; 2% a year to ruin; the Core Auger is built on it and replaces it |
| Accumulator Vault | a starbase of size 2 | 2 | the market's energy storage × 4, then × 8 |
| Cometary Icehouse | a frozen body or a belt, no colony | 1 | gas 12 |
| Skyhook | a planet with the builder's colony | 1 | accessibility +0.1 on that colony |
| Granary Ring | a body with the builder's colony | 1 | food at that market stops spoiling, and the food buffer holds 180 days |
| Soletta | a planet with the builder's colony | 1 | housing +2, growth +10% |
| Deepwatch | a starbase of size 2 | 1 | sensor range +4 cells; the deep sites within 4 found |
| Breaker's Yard | a starbase of size 3 | 1 | reverse engineering there, not only at the capital |
| Tempest Vane | a starbase of size 2 | 1 | storms within 2 cells at half strength (a Collider there is sheltered) |
| Careening Dock | a starbase of size 2 | 1 | a yard up to destroyers, repair × 1.5 |
| Praetorium | a starbase of size 3; one an empire | 2 | the persona's fleet repairs faster, then one officer more |
| Radiant Causeway | a starbase of size 3, paired | 1 | up to 2,000 energy a month to its pair over a standing corridor, 15% lost |
| Current Mill | a starbase of size 3 | 2 | energy 1,200 × the current's strength at its cell a stage |
| Trunkway | a starbase of size 3, paired | 1 | the corridor between the pair never drifts |
| Giant's Sieve | a gas giant, no colony | 2 | gas 40 and crude fuel 20 a stage |
| Exotic Condenser | a neutron star or pulsar | 2 | exotic matter 6 a stage; 3% a year to decay |
| Tabularium | a starbase of size 4; one an empire | 2 | admin capacity +20, then civil upkeep −10% |
| Aegis Shell | a starbase of size 4 | 2 | 12 batteries and piracy security +0.15 |
| Snare | a starbase of size 3 | 1 | drag +1 within 3 cells |
| Cenotaph | a body with the builder's colony | 1 | one for each dead persona: the heir gains a quarter of its renown; stability +5 |

The AI builds all but the paired works, the persona works and the Breaker's Yard, which wait for O6's advisor; it orders research works ahead of works of a kind it holds none of. Not yet built: the Cloudhold's gas districts (it needs a vent rule for habitats), the Breaker's Yard's scrapping and the Soletta's two upgrades (with TF1).

### The game's own names (2026-11-26)

Eleven works and five techs carry the game's own names (the owner's calls of 2026-11-26, `07 D292`). Only the display names changed: every id in `megastructures.json` and `techs.json` is kept, so saves, mods and sprites are untouched. Older decisions and milestone notes use the earlier names.

| Id | Earlier name | Name |
|---|---|---|
| `science_nexus` | Science Nexus | **Athenaeum** |
| `gateway` | Gateway | **Hyperspace Lock** |
| `asteroid_manufactory` | Asteroid Manufactory | **Belt Foundry** |
| `asteroid_battery` | Asteroid Battery | **Belt Redoubt** |
| `orbital_ecosystem` | Orbital Ecosystem | **Hanging Garden** |
| `gas_giant_stabiliser` | Gas Giant Stabiliser | **Cloudhold** |
| `stellar_particle_accelerator` | Stellar Particle Accelerator | **Coronal Collider** |
| `orbital_arcology` | Orbital Arcology | **Crown City** |
| `strip_mine_engine` | Strip-mine Engine | **World Quarry** |
| `crystal_megabore` | Rare-ore Megabore | **Core Auger** |
| `star_lifter` | Star Lifter | **Sun Dredge** |

Techs: Gateway Theory, Gateway Networks and Gateway Lattice became **Lock Theory**, **Lock Networks** and **Lock Lattice**; Zero-Point Power became **Vacuum Power**; Megaengineering became **Megascale Engineering**.

The scales are written Prima, Maiora, Magna and Grandia Opera Astralia, and since O1 the content's `scale` field reads `prima`, `maiora`, `magna` and `grandia` (`07 D277`).

### The Nicoll-Dyson Lance (removed)

Built in S2 on provisional rules (`07 D80`, `D82`; the war on a strike by M10, `D115`) and removed in O1 (`07 D277 §8`, the owner: "keep all except the lance"); three strategic weapons replace it in later phases. A saved Lance is dropped on load, its standing opinion rows run out, and a recorded strike order is refused.

### Failure and decay

A structure with a `failure` arms a yearly timer at a year plus its row mod 360; the roll is a pure function of `(seed, row, year)` (`07 D78`). Failure ruins it: its rows, upkeep and hooks go, and the row stays, unowned. Decay costs the top stage; a one-stage structure falls to ruin (`07 D78`). The built cases are the Coronal Collider (2% a year, ruin) and the World Quarry (5% a year, decay). A fixture gave 202 failures in 10,000 structure-years at 2% (`05 S1`).

### Ruins

Galaxy generation scatters ruins, seed-hashed so no game stream moves (`07 D78`, `D81`; `megastructures.json` `ruins`):

| Rule | Value |
|---|---|
| Prima and Maiora ruins | `perSystem` 0.06 × systems; `deepShare` 0.8 in cells of depth `deepMin` 6 or more (for kinds whose depth range reaches that), the rest on an eligible star or body of a system with no home |
| Magna ruins | `gigaPerSystem` 0.006 × systems; `gigaDeepShare` 0.8 in cells of depth `abyssMin` 10 or more |
| Deep ruin | a deep site of its own kind: the first fleet in meets automata like a cache, the next claims it |
| System ruin | known once its system is surveyed; needs no claim |
| Restoration | a construction ship (`RestoreRuin`) loads `restore` 0.25 of every stage's bill at its system's own starbase market and carries it; the restoration takes the same share of the days, and the structure comes back complete |

Medium seeds 1–3 gave 23, 24 and 23 ruins, of which 19, 21 and 21 lie at depth 6 or more, and 2, 2 and 1 Magna ruins, all at depth 10 or more (`05 S1`, `05 S2`). Switching a structure off removes it from building and from generated ruins (`05 S1`). Deep space and its depth bands are on [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace).

### Hyperspace Lock trade links (M19a)

An empire's hub pair takes the Hyperspace Lock jump when both hubs' systems hold a Hyperspace Lock it may use (its own, or an open-borders partner's) and the jump is quicker than the pair's route: the jump's days, no piracy, all cargo landed, closed by a blockade at either gate system. It is derived at route time, with no state, no save bump and no field rebuild; pact routes and the Galactic Exchange's legs still fly (`07 D283`, owner's call 1A; merged 2026-11-23 per `00-status.md`). The Mass Driver is the other structure trade reads (`07 D78`). See [trade-and-logistics.md](Design-Trade-and-logistics).

### Technology and the Kardashev gates

Every structure names its tech in `megastructures.json` (`tech`). Under the tech-tree rework, the Dyson Swarm is part of the gate to Kardashev Type I and the Dyson Sphere the gate to Type II; the Sphere's tech moves from tier 3 to tier 5 (and waits for 222 known techs, `07 D293`); the Swarm is known from the start and its sections climb a ladder of five techs to tier 3 (T5b, `07 D278 §16`); the works of tiers 6–7 are the path to Type III. The Lance's tech became `collapse_engineering` (Degenerative Collapse) (`07 D278`). A Kardashev type needs its structure for every empire (`07 D278`, call 8). See [progression.md](Design-Progression).

### The AI and megastructures

S1 left the AI building no structures (`07 D79`). T2 (merged 2026-11-23) has the AI order great works: once a year, with none of its own under construction and its month's margin not negative, an AI empire orders the first work it knows whose first stage its markets hold `worksStock` (2) times over, with the treasury's credits and influence for it; the next Kardashev type's structure before the others; at the first of its eight largest starbases whose system takes it (`ai::works`, `07 D278` call 8). After the gate's structure it orders a kind it holds none of before a kind it already holds (the owner, 2026-11-25, `07 D278 §16` call 20), and within each group the work its personality needs most (M19c, `07 D283`). T5b (`07 D278 §16` items 24 to 26): the reserve of twice the bill stays strict for what expansion spends (the colony ship's and the starbases' bills) and is met for the rest by what the empire holds and makes over the stage's build days; a Type 0 AI leaves its Swarm until it holds three colonies outside home (`worksColonies`); and an AI's half-built work does not start its next stage on its own: its plan orders each one under the same reserve, the type's own work first. T5c (`07 D293`): an AI orders a further work of a stacking kind only while it would still add `worksGain` (5%) of its largest effect, which stops it at three Coronal Colliders; before, AIs finished 138 of them in eight 300-year games while rare ore kept 21 empires from a first Athenaeum.

### Ownership

Structures stay with their builder; S1 built no transfer, on the premise that no MVP path moved a starbase to a new owner (`07 D78`).

### RTS and TB

Orders, stages and restorations are constructions and commands that resolve on ticks; failure and decay run on yearly timers; both clocks behave alike (`07 D78`, `13 §6`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Set | The themed set of great works in three scales (its first names were replaced with the game's own on 2026-11-26) | 2026-11-13 | `07 D70` |
| Scale | Up to the Magna scale in MVP; replaces `01 §9.8`'s 1.0 list (Orbital Ring, Dyson Swarm, Athenaeum with +25% skill gain, Habitat) | 2026-11-13 | `07 D70` |
| Ruins | Deep-space rewards, sometimes in normal space as in Stellaris | 2026-11-13 | `07 D70` |
| The model | One data model; effects as modifier rows; hooks opted into by data; stages as constructions | 2026-11-14 | `07 D78` |
| Anchor reach | A Hyperspace Anchor holds every corridor in its radius, not only its owner's | 2026-11-14 | `07 D78` |
| Restoration | A construction ship carries 25% of the stages' bill from its system's starbase market; deep ruins claimed by beating automata | 2026-11-14 | `07 D78` |
| Driver link | A new Mass Driver auto-links to the empire's oldest unlinked driver; the pair's routes take a quarter of the days | 2026-11-14 | `07 D78`, `D79` |
| Cuts | The World Quarry's shrinking (it decays instead) and star ageing; the Ringworld kept its three sections | 2026-11-14 | `07 D79`, `D82` |
| Sphere and Swarm | The Sphere needs and replaces the Swarm; its dimming is a growth malus on the system's own colonies at completion | 2026-11-14 | `07 D81` |
| The Lance | Provisional rules, fired on energy since E1; removed in O1 | 2026-11-14, 2026-11-27 | `07 D80`, `D82`, `D57`, `D277 §8` |
| Lance war | A strike outside war declares war (to humiliate) | 2026-11-14 | `07 D115` |
| Megabore name | The Crystal Megabore became the Rare-ore Megabore, now the Core Auger (id `crystal_megabore` kept) | 2026-11-14 | `07 D57` |
| Kardashev gates | Swarm in the Type I gate, Sphere in the Type II gate; a new roster of great works scoped separately | 2026-11-21 | `07 D278` |
| AI and great works | The AI builds great works as part of development, triggered by having the resources | 2026-11-23 | `07 D278` call 8 |
| Hyperspace Lock trade | A route shortcut, as the Mass Driver's | 2026-11-23 | `07 D283` call 1 |
| Capstones | New capstone structures, designed with the ascension paths later | 2026-11-23 | `07 D283` call 3 |
| AI advisor | M19c: one structure at a time | 2026-11-23 | `07 D283` call 5 |
| Own names | Eleven works and five techs take the game's own names, display names only with every id kept; each renamed work's own effect rides on the great-works phases; the docs stop using kilo, mega and giga | 2026-11-26 | `07 D292` |
| O1 | Scales for tiers, 128 works and 4,096 rows, five works making goods at a place, the Lance removed; save v50 | 2026-11-27 | `07 D277 §21` |

## Not yet built or still open

- **The great works roster.** `07 D277` sets it (63 works, 64 with the Automata core, aligned with main in its §20); O1 is built and O2–O6 are to come. How the eras of `01 §9.6` sit beside the Kardashev ages is open (`07 D278`; DESIGN.md open questions).
- **Capstone structures** for the three ascension paths will be designed with the paths, after the Expanse, precursor and genemodding scopes (`07 D283`).
- **The Belt Redoubt under the station redesign.** `07 D284` removes the batteries column from starbases in favour of a station of sections; no source read says what becomes of the Belt Redoubt's `batteries` hook.
- **The renamed works' own effects** (`07 D292`) are not built: each rides on a phase of the great-works roster (`07 D277`) or on the station and terraforming milestones, and is measured on the host when its phase is. Built in O1: the World Quarry's output by its body's size, the Sun Dredge's by its star's luminosity, the Coronal Collider's failure tied to hyperspace weather and the Crown City's slot. Still to come: the Hyperspace Lock drawing energy for a crossing (O3), the Core Auger as the second step of a Mantle Tap and the Cloudhold's gas districts (O2).
- **Star ageing** (the Sun Dredge) and the **World Quarry's shrinking** were cut (`07 D79`, `D82`).

## Discrepancies noted

- **Two AI rules for building structures.** `07 D278` call 8 (2026-11-23, built in T2 and merged): an AI orders great works once a year, the first work it knows whose first stage its markets hold twice over, the next Kardashev type's structure first. `07 D283` call 5 (2026-11-23, owner's answer A): M19c will add "an advisor on the planning day: the highest-value structure it can afford within its budget's reserve (energy, research, trade and defence weights by personality), one at a time". `D283`'s survey also states "the AI builds none (`D78`'s gap)". The sources do not say how M19c's advisor relates to T2's `ai::works`.
- **Structures in a captured system.** `07 D78` (2026-11-14): "No MVP path moves a starbase to a new owner (conquest takes colonies), so no transfer was built." `07 D113` (2026-11-14, M10, later): "A starbase at hull 0 in a battle with an empire at war with its owner passes to that empire: the starbase, its system and its starbase colony". The sources do not say what happens to the old owner's structures in a system whose starbase changes hands.

## Sources

- 13 §6 Megastructures
- 13 §7 Milestones
- 01 §9.8 Megastructures and ascension
- 07 Decisions: D57, D70, D78, D79, D80, D81, D82, D113, D115, D136, D278, D283, D284, D292
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
