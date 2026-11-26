# Megastructures

**Status:** built, as of 2026-11-23. The shared model with 8 kilo- and 8 megastructures and ruins in S1, the 5 gigastructures and giga ruins in S2 (both 2026-11-14), Gateway trade links in M19a and the AI's orders for great works in T2 (both 2026-11-23); the AI megastructure advisor (M19c) and a new roster of great works are still to come.

Megastructures are large staged constructions in three tiers, kilo, mega and giga (names and tiers under revision as the great works get their own identities). Twenty-one of them are data in `megastructures.json`: they place by rules on stars, bodies or starbases, draw each stage's bill from their own system's markets like any construction, and act through modifier rows and a few code hooks (corridor anchors, yards, mass-driver links, habitats, the Gateway, the Nicoll-Dyson Lance). Ruined structures lie mostly in deep space and the abyss, guarded by automata, and can be restored at a quarter of their cost.

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
| Set | The themed set of kilo-, mega- and gigastructures (its first names are being replaced with the game's own) | 2026-11-13 | `07 D70` |
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
