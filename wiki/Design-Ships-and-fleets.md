# Ships and fleets

**Status:** built, as of 2026-11-23. The hull catalogue and fleet hierarchy in H1 (2026-11-13), on M8's ships and logistics; the persona's per-ship fitting in M12a and its officers in M12b.

Ships are known by named hulls (94 in the catalogue) while six general size classes carry the rules, in the manner of Starsector. Each hull belongs to one of six design lineages, is fitted from an ordnance-point budget with weapons, fighter wings and hull mods, and carries one ship system. Ships sail in fleets that are divided automatically into squadrons and task groups under a doctrine, supplied with fuel and readiness from starbases and support ships. The persona's own fleet is the only one fitted ship by ship; navies use designs.

## How it works

### Size classes

Six classes carry the rules (`07 D70`; `13 §2`). Classes are data in `hulls.json` (`tactics.classes`), so no code names a class (`07 D69`). A seventh class row, `starbase`, holds the starbase's tactical shape.

| Class | Role (`13 §2`) | Commanders | Pilotable | OP scale (mods) | Size | Radius | Turn |
|---|---|---|---|---|---|---|---|
| Fighter | strike craft in wings of 1–6, launched from hangars; not built alone | 0 | no | 0 | 0.25 | 8 | 200 |
| Frigate | screens, raiders | 0 | yes | 1 | 1 | 18 | 120 |
| Destroyer | escorts, gunships; the largest pilotable class | 0 | yes | 2 | 2 | 28 | 80 |
| Cruiser | line, carriers | 1 | no | 3 | 4 | 44 | 45 |
| Capital | battle line, carriers | 2 | no | 5 | 8 | 66 | 25 |
| Super-capital | flagships of a navy | 3 | no | 8 | 16 | 95 | 15 |

Size, radius and turn are the `tactics.classes` values in `data/content/hulls.json`; commanders, piloting and the OP scale are stated in `07 D65`, `D69` and `D70` and match that file. A hull short of its commanders fights at 0.75 rate of fire, thrust and turn rate (`07 D65`; see [combat.md](Design-Combat)).

**Yards.** A hull needs a starbase whose size's yard allows its class (`01 §10.1`, `§10.4`). Capitals build at a Bastion (size 5); super-capitals at a Fortress (size 6) and above (`07 D71`), or at a Shipyard Ring, whose `yard` hook builds super-capitals with 2 extra slots and repair × 2 (`07 D79`; see [megastructures.md](Design-Megastructures)). The yard column of `01 §10.4`'s size table predates H1 (it gives the Bastion "cruiser" and the Fortress "battleship"); `07 D71` is later and `data/content/starbases.json` carries it: Outpost civilian, Station frigate, Starport destroyer, Starhold cruiser, Bastion capital, Fortress, Citadel and Bulwark super-capital.

### Lineages

Lineages are design philosophies, not quality tiers (`13 §3`). Three are core and buildable from the start; three are exotic (`07 D70`).

| Lineage | Philosophy (`13 §3`) | Source (`07 D71`) | Classes it covers | Resolver dealt / taken |
|---|---|---|---|---|
| **Forge** (core) | heavy armour, kinetic and missile mounts, slow, cheap to repair | start | frigate to super-capital | 1 / 1 |
| **Line** (core) | balanced armour and shields, mixed mounts | start | frigate to super-capital | 1 / 0.9288 |
| **Lumen** (core) | light armour, strong capacitors and shields, energy mounts, fast | start | frigate to super-capital | 1 / 1 |
| **Veil** (exotic) | phase/void hulls that slip out of combat space | the tier-3 physics tech Veil Drives, after Corridor Stabilisation | frigate to capital | 1.3009 / 0.9081 |
| **Precursor** (exotic) | unique systems no tech reaches | not buildable: derelicts in a fifth of looted deep caches | frigate to super-capital | 0.6991 / 1.04 |
| **Automata** (exotic) | the deep faction's hulls | not buildable: derelicts from caches; capture by Boarding planned | frigate to capital | 1.1587 / 1.4872 |

The class coverage and the dealt/taken multipliers are `lineages[]` in `data/content/hulls.json`; the multipliers are the resolver's per-lineage levers (`07 D72`). A derelict exotic hull joins the looting fleet at half hull (`07 D71`). Each lineage has a colour shown on the ship's core (`07 D71`).

**Pirate conversions.** Pirates field Mk II conversions of core hulls with extra mounts and built-in d-mods (`13 §3`); the catalogue's raider hull is the Brigand Mk II.

**Civic and species flavour.** Civics and species traits add modifier rows to lineages; no hull is species-locked (`13 §3`). Built rows (`07 D71`), applied to an empire's design stats and to its ship builds' daily draws:

| Civic | Effect |
|---|---|
| Warrior Culture | Forge builds 10% cheaper |
| Merchant Republic | Line builds 5% cheaper |
| Technocracy | Lumen +5% shields |

Species ship sets are visual: G2's Blender generators are one builder per lineage, parameterised by class and species ship set (silhouette, greeble, palette) (`05 G2`).

### The catalogue

H1 built 94 hulls (`05 H1`): 64 military, 12 fighters, 16 support and civilian, and 2 deep encounters (the Brigand Mk II raider and the Void Leviathan). All names are original (`07 D71`). The twelve pre-H1 hulls keep their ids and indices and take new names: corvette Bannerwick, destroyer Halyard, cruiser Lodestar, battleship Grandmarch, automaton Cogwarden (`07 D69`).

`13 §4` asked for 16 / 15 / 13 / 10 / 4 per class, which sums to 58, not 64; the build makes 64 as **18 frigates, 17 destroyers, 14 cruisers, 11 capitals and 4 super-capitals**, of which 6 Veil, 6 Precursor and 6 Automata (`07 D71`).

Military hulls by class and lineage (display names from `data/content/strings/en.json`):

| Class | Forge | Line | Lumen | Veil | Precursor | Automata |
|---|---|---|---|---|---|---|
| Frigate (18) | Rivetjaw, Cinderling, Slagbiter, Anvilkin | Bannerwick, Tidewarden, Quillon, Gallowglass, Keelrunner | Glasswing, Prismfin, Sunthread, Lanternmoth | Hushwing, Duskglass | Firstlight | Cogling, Rustwasp |
| Destroyer (17) | Slagwall, Pitforge, Breakwater, Cindermaul | Halyard, Warrantor, Keelmark, Morrowgard | Shimmerline, Aureole, Flarewing, Beamglass | Nightveil, Gloamskiff | Oldcrown | Gearmantis, Cogwarden |
| Cruiser (14) | Smeltmaster, Crucible, Ingotreach, Pyreholt | Lodestar, Constancy, Ironwake | Prismata, Halobright, Glintcrown | Umbravault | Firstwatch, Elderwatch | Cogcathedral |
| Capital (11) | Mountainforge, Anvilthrone, Cinderhall | Grandmarch, Sovereignty, Steadhold | Sunvault, Dawnspire | Eclipsehold | Eldermonarch | Cogmonolith |
| Super-capital (4) | Worldanvil | Concordance | Lightmother | — | Eldest | — |

Fighter wings (12, role `fighter`, carried in bays, not buildable alone): Dartfly, Needlejack, Pikebolt (Line); Whipsting, Emberbee, Burrfly (Forge); Sparkmoth, Glowmite, Thornseed (Lumen); Wraithlet (Veil); Eldermote (Precursor); Cogspark (Automata).

Support and civilian hulls (16): Wayfinder (science), Masonwright (construction), Seedbearer (colony), Cartling / Haulwright / Bulkmaw (freighters S / M / L), Fuelwick / Brimfull / Wellspring (tankers S / M / L), Marshaldeck (transport), Voyagehall (liner), Scrapcrane (salvage rig), Pathreader (survey ship), Towline (tug), Mendhold (repair tender), Provender (supply tender). The freighters S and L, the three tankers, the liner, salvage rig, survey ship, tug and both tenders are new in H1 (`07 D71`).

**Balance.** Every military hull fights its class's Line reference (4 Bannerwick, 3 Halyard, 2 Lodestar, 1 Grandmarch or 1 Concordance), auto-designed from every component with full commanders in open space; `tools calibrate --tune-hulls` bisected a factor on hull, armour and shields for each hull outside 25–75% win rate. Two passes moved 47 hulls; every military hull is now within 25–75%, the worst at 74% (`07 D71`, `05 H1`).

`01 §10.1` still prints M8's five-hull table (Corvette 300 hull HP, 40 alloys and 2 parts, 60 days, speed 120, through the 1.0 Carrier) and its civilian table. Those rows were the M8 base; per-hull numbers are now the `hulls.json` catalogue rows after the H1 factors.

### Hull schema

A hull row holds (`13 §4`, `05 H1`): size class, lineage, role, tech unlock and the yard class needed; ordnance points; mounts (size S/M/L × type); hangar bays; one ship system; built-in hull mods; hull, armour sectors, shields and capacitor; speed and turn rate; logistics (fuel per jump, cargo, crew minimum and maximum, supply use); readiness; and a composition cost used only by the AI and auto-design, never a deployment limit.

### Mounts, ordnance points and fitting

**One layout for designs and fitted ships** (`07 D69`). A design is `fit: [u8; 24]`: slots 0–15 are its mounts in the hull's order (a weapon, or a fighter wing on a bay), slots 16–23 are eight hull mods; `u8::MAX` is empty. The persona's per-ship fitting reads the same 24 bytes.

**Mounts** are a size (S, M, L) and a type: kinetic, energy, missile, hybrid (kinetic or energy), universal, or a bay (`B`, wings only). A weapon fits a mount of its size or larger and of its type (`07 D69`).

**Ordnance points (OP)** (`07 D69`):

| Item | OP |
|---|---|
| Weapon, S / M / L | 5 / 10 / 20 |
| Fighter wing | its own OP (8–14 in `hulls.json`) |
| Hull mod | its base OP × the class's `opScale` (frigate 1, destroyer 2, cruiser 3, capital 5, super-capital 8) |
| Built-in hull mod | free |
| Unspent OP | becomes capacitor at `capPerOp` = 6 shields per point |

`datacheck` checks every hull's auto-design against its budget, over every component and over the starting set; `DesignCreate` refuses a design over budget (reason `Power`). Civilian and support hulls carry no OP and are unfitted (`07 D69`).

**Components** (`data/content/hulls.json`, 59 rows):

| Kind | Count | Rows |
|---|---|---|
| Weapons | 12 | kinetic: mass driver (S), coilgun (S), railgun (M), mass accelerator (L); energy: laser (S), pulse laser (S), heavy laser (M), particle lance (L); missile: rocket pod (S), swarm missiles (M), torpedo (M), cruise missile (L) |
| Fighter wings | 12 | one per fighter hull |
| Hull mods | 32 | base OP 0–4; d-mods exist as mods flagged `dmod`, refused in a design (`07 D69`) |
| Combat computers | 3 | picket, line, artillery |

Weapon families follow `01 §10.1`: kinetic (high damage, low accuracy, good against armour), energy (high accuracy, good against shields), missile (long range, counterable by point defence). Each weapon has a range band, damage, cooldown in days, accuracy, tracking and penetration.

**The computer stays the hull's**, set by its role, with no computer slot (`07 D69`). In the resolver the computer type picks targets (smallest, largest, nearest in band) and the preferred band (`01 §10.3`).

### Ship systems

Ship systems are data: 32 systems over 8 kinds, one per hull, shared across hulls, each with power, seconds and cooldown. The tactical engine runs them; the AI triggers each kind on its own condition, and a pilot triggers it with F (`07 D69`).

| Kind | Systems in `hulls.json` |
|---|---|
| burn | Sprint Coil, Lunge Drive, Surge Vanes, Comet Kick |
| damper | Brace Field, Kinetic Sponge, Stone Skin, Bulkhead Lattice |
| overdrive | Breech Rush, Hot Loader, Volley Fervour, Gunlock Surge |
| phase | Veilstep, Ghostfold, Umbral Slip, Fade Gate |
| flare | Chaff Bloom, Glint Swarm, Decoy Fan, Mirror Spray |
| uplink | Spotter Net, Range Sextant, Longsight Mast, Plotting Choir |
| repair | Mender Swarm, Weld Choir, Scab Foam, Patchwork Drones |
| capacitor | Charge Well, Static Bank, Arc Reservoir, Flux Flush |

(System names are the `systems[]` ids in `data/content/hulls.json`, title-cased.)

### Designer and auto-design

A design is a hull plus its fit within the OP budget (`07 D69`, replacing `01 §10.1`'s "hull + chosen components within power budget"). **Auto-design** is the default: it generates a sensible design for each hull when new components are researched (`01 §10.1`). The player may make designs in the ship designer (`05 H1`). Auto-upgrade (refitting docked fleets) was cut in M8; obsolete designs are marked for refits (`05 M8`, `07 D38`). On loading a version-14 save every design is refitted by auto-design from what its empire knows (`07 D69`).

### The persona's fleet: per-ship fitting

Only the persona's personal fleet gets per-ship state (`07 D44`, `01 §19.1`):

- **Refit** (`Captain::Refit`) gives a ship its own design row, at a starbase the persona's empire owns or where it is trusted; its d-mods stay; the fit passes the design rules with the components the persona's empire or the yard's owner knows; its OP less the s-modded hull mods must fit the hull (`07 D135`).
- **s-mods.** A story point builds a hull mod into a ship, its OP off the budget (`07 D134`).
- **d-mods** are lasting damage carried by salvaged or badly repaired hulls (`01 §19.1`). The Damage Control skill removes one at a starbase for 50 / 35 / 20 alloys; Boarding's ranks plan one fewer d-mod on salvaged hulls (`01 §19.2`).
- **Templates** (`Captain::Template`) make a ship's fit an empire design with its d-mods cleared and no s-mods, which the empire's yards build like any design (`07 D135`).
- **Officers** (M12b): a leader class `officer`, assigned to a ship of the personal fleet, with two traits that add skill. There are 2 slots plus `persona.officers` (Coordination ranks, the neural link implant). Each officer's skill adds 2% to the personal fleet's damage, spread over its ships. A lost ship frees its officer; a companion officer whose loyalty reaches 0 leaves and takes their ship (`07 D140`).
- **Size cap.** The personal fleet holds at most `4 + renown / 50` ships; merges past it are refused (`07 D134`, `01 §19.2`).

### Fleet hierarchy

Ship → squadron → task group → fleet → navy (`07 D70`, `13 §5`). The hierarchy is **computed from a fleet's ships, not stored** (`military::groups`, `07 D72`):

| Level | Rule (`07 D72`, `13 §5`) |
|---|---|
| Squadron | the fleet's ships in id order, grouped by class and lineage, up to `squad` (20) each |
| Task group | led by a commander; holds up to `2 + coordination` squadrons; task group `g` takes squadrons from `g × (2 + coordination)` onward |
| Support group | task group 7, holding the fleet's support and civilian ships |
| Fleet | led by an admiral; holds up to `2 + admiral level / 3` combat task groups |
| Navy | the empire's whole fleet; its size set by fleet capacity |

Until M11's leaders, `rules.command` sets coordination 0 and admiral level 3, so a fleet holds 3 task groups of 2 squadrons (`07 D72`); `hulls.json` still holds those values. A merge or rejoin past the cap is refused (`Cap`). Fleets move and fight as one (`13 §5`).

**Detach and rejoin** (`07 D72`). A stopped fleet's task group leaves as its own fleet with its share of fuel and stock, keeps its parent, and takes the group's doctrine as its own. It rejoins its parent when both stand in the same system, within the cap. Merge, split and reinforce are commands from M8 (`05 M8`).

### Doctrine

Doctrine is a packed `u32` per fleet plus eight task-group overrides (`fleets.doctrine`); a task group without an override takes the fleet's (`07 D72`).

| Setting | Values (`07 D72`) |
|---|---|
| Engagement range | standard keeps the officer's; close closes to 45% of the longest weapon's range; long holds at 95% |
| Retreat | a hull share, which raises the side's withdrawal threshold |
| Formation | a squadron's formation in a large action (line, wedge, screen, englobe) |

The **stance** stays the fleet's (`01 §10.2`): aggressive, patrol, evasive, passive, blockade, and `raid` (pirate fleets only). Patrol is aggressive toward pirates and toward empires its owner is at war with, passive otherwise.

### Support ships

Support ships sail in the support group and draw stock loaded at a docked market, so nothing teleports (`13 §5`, `07 D72`):

| Ship | Effect (`07 D72`) |
|---|---|
| Tankers S / M / L | add 40 / 100 / 250 to the fleet's tank |
| Repair tender | holds 200 stock (alloys); mends 1% of hull a day outside a dock, drawing alloys at the starbase repair rate |
| Supply tender | holds 200 stock (goods); keeps the fleet in supply anywhere, drawing 0.1 goods per ship a day, so readiness recovers; with its stock spent the fleet is out of supply |

On a hyperspace route, repair stock halves hull wear and supply stock halves readiness attrition (`07 D72`). A fleet halted in hyperspace (an intercept, an ambush, a deep group, a cancelled move) has its supply re-checked where it stops; a tender with stock still keeps it supplied (`07 D191`).

### Movement

Fleets cross hyperspace on routes planned at departure at the slowest ship's speed, after a 3-day jump windup at the system edge; within a system, moving between a body, the starbase and the edge takes `1 + |orbit_a − orbit_b|` days and no fuel (`01 §10.2`). Routes, intercepts and depth are on [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace); flying a fleet by hand is on [flight-view.md](Design-Flight-view).

### Fuel

| Rule | Value (`01 §10.2`, `data/content/hulls.json`) |
|---|---|
| Tank | `10 × Σ fuelPerJump` (`tankJumps` 10), starts full, plus tankers |
| Burn | `Σ fuelPerJump / 5` per hyperspace day, × `1 + storm` and the depth factor |
| Fuel upkeep | drawn from the tank daily at 1/30 of the per-30-day figure |
| Docked top-up | every day in a system with a friendly market holding fuel stock, from that stock |
| Short route | the route beyond the fleet's fuel at departure runs at 25% speed; a fleet that departs dry crawls the whole route (`07 D46`) |
| Empty tank | −2 readiness per day |

**Fuel tiers** (E1, `07 D73`, `07 D57`). A tank holds units by tier, and a fleet burns its preferred tier first, then down the tiers, then up (`FleetFuel` sets the preference; new fleets prefer refined).

| Tier | Commodity | Units per refined unit of range | Effects of burning it |
|---|---|---|---|
| Crude | crude fuel | 1.5 | speed −15%; readiness recovery in supply −50%; engines −10% in battle |
| Refined | fuel | 1 | baseline |
| Battery | charged cells | 4 | speed +15%; shield regeneration +20% |
| Exotic | exotic fuel | 0.5 | speed +10%; damage, engines and regeneration +10% |

The Logistics skill cuts the personal fleet's fuel by 10 / 20 / 30% (`hyper.burn`, `07 D134`).

### Supply and readiness

A fleet is **in supply** within the supply range of a friendly starbase whose market holds fuel (range by starbase size, 1–6 jumps of 2 cells each), or anywhere with a stocked supply tender (`01 §10.2`, `§10.4`, `07 D72`).

| State | Effect (`01 §10.2`) |
|---|---|
| In supply | readiness +10 per 30 days, to 100 |
| Docked at a starbase | repair at the size's rate (1.5–5% a day) paid in alloys from its market (1 alloy per 100 points); refuel from its fuel stock |
| Out of supply | no refuel, no repair, readiness −5 per 30 days |
| Deep space, above depth 2 | per day, per cell of depth: readiness −0.5, fuel +10%, hull 0.1% (`07 D45`) |

Readiness sets battle performance: in the tactical engine, rate of fire × `0.25 + 0.75 × readiness` and accuracy × `0.5 + 0.5 × readiness` (`07 D65`); in the resolver, damage × `readiness / 100` (`01 §10.3`). Each minute of a tactical battle costs every fleet 2 readiness (`07 D66`). A fleet at readiness 0 starts no fight (`05 M8`).

### Fleet capacity and upkeep

Fleet capacity = 16 + Σ docking of own starbases + tech; each ship over it raises ship upkeep by 2% (`01 §10.4`; `fleetCap` 16, `overCap` 0.02). The Coordination skill adds +2 / 4 / 6 empire fleet capacity, and its elite rank exempts the personal fleet from the over-capacity upkeep (`01 §19.2`). Upkeep per hull is credits and fuel per 30 days (`01 §10.1`).

### Fleet power

Fleet power, shown in the UI and used by the AI and pirates, is `Σ (dps × ehp)^0.5` over ships, where dps = Σ damage / cooldown and ehp = hull + armour + shields, kept per fleet incrementally (`01 §10.3`, `05 M8`).

### The AI's navy

An AI empire that makes no components buys them at the yard within the governor's reserve and builds a warship from the bill on hand, but only at war or with its navy under half its target power, so losses are replaced (`07 D116`). Its war orders send the strongest idle fleet against the enemy system nearest its capital once its power is twice what stands there (`07 D116`; see [ai-and-victory.md](Design-Ai-and-victory)).

### RTS and TB

Designs, refits, merges, splits, doctrine, detach and rejoin are commands that apply on the next tick or turn in both clocks (`05 H1`, `05 M8`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Ship identity | Named hulls carry identity; six general classes carry the rules, after Starsector | 2026-11-13 | `07 D70` |
| Size classes | Six for now, to be tweaked | 2026-11-13 | `07 D70` |
| Lineages | Three core (Forge, Line, Lumen) and three exotic (Veil, Precursor, Automata), plus species and civics flavour; the names are spec-session proposals | 2026-11-13 | `07 D70` |
| Hull count | 60+; built as 64 military, 12 fighters, 16 support (18 / 17 / 14 / 11 / 4 per class, replacing `13 §4`'s 58-sum counts) | 2026-11-13 | `07 D70`, `D71` |
| Fleet structure | A layered hierarchy with doctrine and support ships, computed rather than stored | 2026-11-13 | `07 D70`, `D72` |
| Commanders | 1 for a cruiser, 2 for a capital, 3 for a super-capital | 2026-11-13 | `07 D70` |
| Fitting layout | One 24-byte fit (16 mounts, 8 mods) for designs and the persona's fitted ships; OP replaces `01 §10.1`'s power budget | 2026-11-13 | `07 D69` |
| Fitting scope | Per-ship fitting only on the personal fleet; templates carry fits to the navy | 2026-11-12 | `07 D44`, `D135` |
| Exotic sources | Veil by tech; Precursor and Automata from cache derelicts until Boarding | 2026-11-13 | `07 D71` |
| Fuel tiers | Four tiers with a preferred fuel and fallback; fuel as a real bottleneck for fleets | 2026-11-13 | `07 D73`, `D57` |
| Calibration | The owner accepted H1's reached resolver calibration ("accept and hold"), enforced at 26 / 26 | 2026-11-13 | `07 D72` |
| Officers per ship | Built in M12b with the companions (deferred from M12a) | 2026-11-16 | `07 D135`, `D140` |

## Not yet built or still open

- **Owner review.** The lineage names and the commander counts are spec-session proposals awaiting the owner (DESIGN.md, open questions).
- **Calibration to ±5 / ±10.** The resolver agrees with the tactical engine within 26 points of win rate and 26 of loss share; the full bounds of `11 §2` wait for a structural resolver term that no milestone carries (`07 D72`). See [combat.md](Design-Combat).
- **Boarding capture.** Automata hulls are to be captured by Boarding (`13 §3`, `01 §19.2`); M12a registered the boarding keys as rows with no reader (`07 D134`), so exotic hulls come only from derelicts.
- **Species flavour rows.** `13 §3`'s example (an aquatic species giving Lumen +5% capacitor) has no built counterpart; only the three civic rows of `07 D71` exist.
- **Command capacity.** `13 §5` says command capacity counts commanders and admirals while fleet capacity counts ships; no built rule for command capacity appears in `07 D72` or `05 H1`.
- **Leaders feeding the hierarchy.** `07 D72` set coordination 0 and admiral level 3 "until M11's leaders"; no source read records whether M11 or later milestones feed real commander and admiral values into the caps, and `hulls.json` still holds the fixed values.
- **Story-point ship recovery.** "Recover a ship lost in a battle" (`01 §19.2`) waits for its system (`07 D134`).
- **Flagship skill keys.** Gunnery's and Helm's flagship rows (`ship.*.damage`, `ship.evasion`, `ship.speed`) exist with no reader (`07 D134`).

## Discrepancies noted

- **Two squadron definitions.** `13 §5`: "Squadrons form automatically from ships of one class and lineage, 5–20 strong". `07 D72` (fleet hierarchy): "form squadrons by class and lineage, up to `squad` (20) each". `07 D65` (small tactical action): "By side and hull class in order of battle, 10 ships each" (`tactics.squad` 10). `07 D67` (large action): "Squadrons of up to 20 ships of one class" (`tactics.squadLarge` 20). The sources do not say whether the hierarchy's squadrons and the battle squadrons are meant to be the same groups.
- **The tracker's Character row.** `docs/DESIGN.md` (Pillars, "Character") gives the persona's personal fleet, per-ship fitting and templates as "designed (M12a, M12b)", while `00-status.md` records M12a merged on 2026-11-15 and M12b on 2026-11-16. The tracker's header reads "Last updated 2026-11-13", so its row may predate the merges.

## Sources

- 01 §10.1 Hulls
- 01 §10.2 Movement and logistics
- 01 §10.3 Combat resolution (fleet power)
- 01 §10.4 Starbases (yards, supply range, fleet capacity)
- 01 Game design: §19.1 (the personal fleet), §19.2 (skills)
- 13 §2 Size classes, §3 Lineages, §4 Hulls, systems and fitting, §5 Fleet hierarchy
- 07 Decisions: D38, D44, D45, D46, D57, D65, D69, D70, D71, D72, D73, D116, D134, D135, D140, D191
- 05 Milestones: M8, H1, G2
- 00 Status (M12a, M12b merges)
- DESIGN.md (pillars "Ships" and "Fleets"; open questions)
- `data/content/hulls.json` (classes, lineages, systems, components, rules) and `data/content/strings/en.json` (hull names)

## Related pages

- [Overview](Overview)
- [combat.md](Design-Combat)
- [starbases.md](Design-Starbases)
- [galaxy-and-hyperspace.md](Design-Galaxy-and-hyperspace)
- [flight-view.md](Design-Flight-view)
- [economy-and-resources.md](Design-Economy-and-resources)
- [factions-and-characters.md](Design-Factions-and-characters)
- [megastructures.md](Design-Megastructures)
- [ground-combat.md](Design-Ground-combat)
- [ai-and-victory.md](Design-Ai-and-victory)
- [modding-and-data.md](Design-Modding-and-data)
