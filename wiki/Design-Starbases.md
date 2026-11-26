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
