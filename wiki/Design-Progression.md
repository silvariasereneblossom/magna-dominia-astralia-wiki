# Progression

**Status:** in progress, as of 2026-11-25. Traditions, government, policies, edicts, eras I–III and the modifier system are built (M7); the tech tree rework is on track T, with T1 to T4 and T5a (hooks and bench rows) built; T5b (the Dyson line and the pace) waits for the owner's answers on five design points.

Progression is how an empire changes what it is and what it can do over a 300-year game. Technology is an open tree of 402 techs climbing toward Kardashev Type III; traditions buy perks with unity and lead to ascension perks; authority, civics, policies and edicts set the rules of government; eras unlock systems as an empire grows. Every effect of all of these reaches the simulation through one path, the modifier system, so every number on screen can show the stack that made it.

## How it works

### The modifier system: one effect pathway

Every effect from techs, traditions, civics, authorities, policies, designations, ascension perks, edicts, events, situations, leaders, difficulty and the persona's skills is a modifier row (`02 §5`, built in M7, `07 D36`).

- A row is `(scope kind, scope id, key, value, op, source, expires tick)`. Keys are interned names such as `job.miner.output` or `pop.skill_gain`; an unknown key is refused at load.
- The effective value of a `(scope, key)` pair is `base × Π(1 + mul) + Σ add`.
- **Factors** (job outputs, research, unity, trade value, freight, costs, growth, mortality, skill gain and decay, adjacency scale, piracy growth) fold with base 1, and the consumer multiplies in its own base (for example 4.0 per miner). **Terms** (happiness, stability, amenities, housing, slots, caps, service radius, skill output, tax tolerance, accessibility, piracy security, influence's add) fold with base 0 (`07 D36`).
- The cache holds one entry per `(scope, target, key)` with `[Π(1 + mul), Σ add]`. Empire entries sit in a dense empire × key array; a colony's value is its owner's entry combined with its own, written into colony columns on refold.
- Any insert, removal or expiry marks its key dirty. Dirty entries refold in daily step #9 and again in #27 (`cleanup`), so nothing is reapplied unless its inputs changed. The cache is derived, never saved, and rebuilt on load.
- The tooltip query returns the full stack with sources, for example "Miner output: 4.0 base; +15% Mining World; +10% Mining Guilds; −25% low stability". `CLAUDE.md` requires a breakdown on every number shown.

The M7 acceptance test folds the Mining designation, Mining Guilds and Deep-Core Mining into the homeworld's miner factor `1 × 1.15 × 1.15 × 1.10`, and changing one modifier refolds exactly one cache key (05 M7).

Designed and not built: under the tech rework, tech modifiers will stop being rows. The fold of a dirty `(empire, key)` entry will add the known techs that carry the key from an index built at load, and the tooltip will list them as it lists rows (`07 D278 §13`; on T2's list of work still to do).

### Technology: an open tree to Kardashev Type III

The owner reworked the tree on 2026-11-21 (`07 D278`). The rules below are `01 §9.1` as rewritten for it.

**Fields and categories.** Three fields, physics, society and engineering, with four categories each (twelve in all). Physics gained astrogation, society gained security, engineering gained voidcraft (`07 D278 §1`). Each field has one active project, and its daily research points go to it.

**Researchable.** There is no draw. An empire may research any tech that is researchable:

| Condition | Rule |
|---|---|
| Prerequisites | all of its `prereqs` known, and at least one of its `any` when it names some; either may be in another field |
| Tier | tier t + 1 of a field opens once 6 of its tier-t techs are known (was 4 before the rework) |
| Deed | its `when` condition holds, when it has one: a dig, a first contact, a battle, a great work, a finished tradition tree; the tree shows the deed in words |
| Age | its age's Kardashev type is reached: Type I for tiers 4–5, Type II for tiers 6–7 |
| Insight (not built, T2) | or its insight has reached half, which waives all of the above (tech skipping) |

No prerequisite sits in a higher tier, and there are no forks: nothing an empire researches closes anything else (`07 D278 §3`).

**Tiers and ages.**

| Age | Tiers | Content | Gate |
|---|---|---|---|
| Toward Type I | 0–3 | near space, the first jump, an interstellar empire; ends with the Dyson Swarm | none |
| Toward Type II | 4–5 | stellar engineering toward a full Dyson Sphere | Type I |
| Toward Type III | 6–7 | speculative and gigastructural science | Type II |

All tier-0 techs are known at game start.

**Counts.** 402 techs (133 in physics and society, 136 in engineering since T5b's Dyson Swarms at tier 0 and Dyson Aperture at tier 6 and SY1a's Robotic Workforce at tier 2), distributed 7 / 54 / 67 / 72 / 60 / 54 / 49 / 39 over tiers 0–7. 51 are gated (49 by deeds; two by the number of techs known, `07 D278 §16`), and 4 are found rather than researched (reached by insight from salvage once T2 is built). No empire is meant to finish the tree in a 300-year game: a strong one reaches Type III around years 280–320 knowing about 80% of it (`01 §9.1`). Techs grant buildings, districts' extra caps, ship hulls and components, modifiers, policies and the great works.

**Goals are the queue.** An empire keeps up to six goals in order. A field with no project takes the cheapest researchable tech of its own field on the way to the first goal that has one (a prerequisite or the goal itself; from an `any` group with none known, any of them). When the way holds a tech in a tier still shut, the field takes the cheapest researchable tech of the tier under the lowest shut one, so a goal opens its own tiers. With no such step, a field on Auto takes its cheapest researchable tech; with Auto off it waits and prompts. Every empire starts with the Hyperspace Drive as its first goal. `PickResearch` takes any researchable tech of the field at any time, and the field's banked points stay. Goals and picks are commands (`SetResearchGoals`, `PickResearch`) in both RTS and TB.

**Cost.**

```
cost(tech) = base(tier) × (1 + 0.012 × techsKnown) × sprawlPenalty × empire.techCost
             × (1 − diffusion) / research.speed(category)
base = [0, 110, 380, 4600, 4800, 10000, 22000, 50000][tier]
```

The Hyperspace Drive carries an explicit cost of 150. `research.speed.<category>` divides the cost; it replaced the old draw weights, at a third of their values (`07 D278 §2`).

**Diffusion.** A tech costs 4% less for each empire in contact that knows it, 8% for a commercial-pact partner, and 12% for a pact partner while the empire's trade policy has no tariff (Free), times `research.diffusion`, capped at 50%. Free trade with empires ahead is the fastest way to catch up (the owner's call 4, `07 D278 §15`). As built, diffusion is summed over the contact mask at the cost check (track T's call).

**The Kardashev rating and types** (built in T2's first slice).

| Rating | Energy made by the empire's markets a month |
|---|---|
| K 0.73 | 45 |
| K 1.0 | 2,000 |
| K 2.0 | 50,000 |
| K 3.0 | 1,000,000 |

The rating is read once a year, interpolated on the logarithm between the anchors (`research.kardashev.energy`). A type is a latch: an empire takes the next type when its rating reaches it and it holds that type's finished structure (the Dyson Swarm for Type I, the Dyson Sphere for Type II, none for Type III), checked yearly and when a work is finished. It keeps the type whatever happens to its energy afterwards. The same rule holds for AI empires, which order the great works themselves (below).

**An idle field's bank** holds at most the cost of one tech of the next age's first tier at that day's price: tier 4 before Type I, tier 6 before Type II, tier 7 after.

**Pacing.** The starting empire makes 7 points a field per 30 days (2 researchers × 6 / 3 plus Capital I's 3 per field). The Hyperspace Drive costs `150 × (1 + 0.012 × 6) = 161`, so it lands at about 2 years. Opening tier 2 takes six tier-1 techs at about 123 each, about 740 points a field. On the T2 host runs, by year 100 between 1 and 4 of the 8 AI empires had reached Type I, depending on the seed (`07 D278`, "Built: T2, the first slice").

**The three ways to learn a tech** (designed, T2; the owner's call 2, `07 D278 §5`):

| Way | Rule |
|---|---|
| Research | the empire's researcher pops, points a field a month against the project's cost |
| Reverse engineering | a ship, building or relic of unknown tech taken apart at the capital (`ReverseEngineer`, `Dismantle`): 60 days for a ship, scaled by hull size; each unknown tech aboard gains 30% insight; the item is consumed |
| Scavenging | digs and anomalies pay insight through the event effect `{ "tech": "progress", "id", "points" }`; caches pay hulls and relics to use or take apart |

**Insight** is the share of a tech's cost already paid. An empire holds up to sixteen insights; when full, the smallest gives way. Holding and using an item of unknown tech builds insight at 1% a month to a ceiling of 50%. A battle against a design with a component the empire cannot build gives 2% a battle, to 25%. FC4's steal-tech operation gives 50% on the stolen tech. At half, the tech can be researched without its prerequisites, tier, type or deed; at the whole, it is known.

**The tree view** (T4, built 2026-11-24) is the research screen: tiers as columns under the three ages' banners, a category a row, a tech a card, every prerequisite a line. The header carries the era, the techs known, the Kardashev rating and type, and what the next type still needs. A card shows its state (known, the project, researchable, waiting for its deed, locked), its insight, and in its hint what it does, what shuts it and its cost factor by factor. A click researches a researchable card and sets or clears a goal on any other; the goals (six, in order, drag to reorder), a search and the filters sit beside the tree. The fleet screen lists a fleet's unknown tech and carries the Reverse engineer order; reaching a type fires its event and a line in the turn report (`01 §9.1`).

**Tech hooks.** About 150 techs carry one modest modifier and point at an effect the game cannot express yet: M17's trade, FC4's espionage, the great works, new components, building grades, policies. `docs/20` lists every one (for example `swarm_fabrication`, `stellar_collectors` and `swarm_coordination` "Required for the Dyson Swarm great work"; `planetary_remodelling` "Unlocks terraforming of settled worlds"). Track T wires them in T2 and T5 as those systems land (docs/20).

**Bigger without power creep.** Over tiers 0–3, the stacked bonus on every modifier key the old tree had equals the old tree's; `07 D273`'s two size levers (`empire.colonyPops`, `empire.popUpkeep`) stay on their own techs (track T's call, `07 D278 §15`).

### Traditions and ascension

Built in M7 (`01 §9.2`).

- **Trees.** The MVP has three: Expansion, Prosperity and Discovery. 1.0 adds Harmony, Supremacy, Diplomacy, Mercantile and Adaptability (`05`'s M22 lists 8 trees).
- **Structure.** Each tree has an adoption effect, 5 perks (taken in an order within simple prerequisites) and a finisher.
- **Cost** in unity: `cost = 150 × (1 + 0.4 × perksTaken) × (1 + 0.01 × pops) × sprawlPenalty`.
- The Prosperity tree's perks include Guilds (`pop.skill_output` +0.05) and Fair Levies (`tax.tolerance` +0.05) (`07 D27`, `D28`).
- A second tree concurrently needs Era II.
- **Ascension perks.** Completing a tree grants an ascension perk slot. Built: three perks (Technological Ascendancy, Interstellar Dominion, One Vision). Master Builders and the ascension path openers are 1.0.

**Ascension paths** are designed and not built. `01 §9.8` describes Biological, Synthetic and Psionic paths, each a 3-perk chain with special projects and events. The owner revised this on 2026-11-23 (`07 D283`, call 2): the Psionic path is the path to the Singularity (the Expanse scope); a Synthetic path opens after contact with the Automata; Biological ties to genemodding, which a new scoping session will plan for both the persona and the empire's species. The capstones will be new structures designed with the paths (call 3, option B). M19b, the ascension phase, left M19 and will be built once those three scopes land. Full AI persona cores belong to the synthetic ascension; AI cores as a researchable tech that improves characters' efficiency are the owner's call of 2026-11-23 (`07 D284`).

### Authority, civics, policies and edicts

Built in M7 (`01 §9.5`).

**Authority.**

| Authority | Elections | Bonus |
|---|---|---|
| Democratic | every 5 years (`07 D241` call 5) | +5 stability; vote weight 1 per pop |
| Oligarchic | every 10 years (council) | +5% research |
| Autocratic | ruler for life | +1 influence |

Reform costs 200 influence and has a 10-year cooldown. 1.0 adds Hive, Machine and Corporate authorities (`01 §3`); machine and hive species are NPC only (`07 D230`). The authority also sets whose weight counts in politics, the agitation rate and suppression price ([Factions and characters](Design-Factions-and-characters)).

**Civics.** Two per empire, a third from a perk. Each is a set of data-driven modifiers. Built (8):

| Civic | Effect |
|---|---|
| Mining Guilds | +15% ore |
| Merchant Republic | +1 hub, +10% trade value |
| Technocracy | +10% research |
| Warrior Culture | +10% fleet damage |
| Agrarian Idyll | +food, +amenities from agricultural districts |
| Meritocracy | `pop.skill_gain` +15%, `pop.skill_output` +0.05 |
| Corporate Charter | `tax.tolerance` +0.05, +5% trade value |
| Feudal (FC4) | no modifier rows; NPC fiefs. Open to autocracy and oligarchy only; a seeded 25% share of AI empires hold it (`07 D219`) |

About 30 civics are proposed for 1.0 in CV1 ([Civilisations and species](Design-Civilisations-and-species)).

**Policies.** Free to set, with a 10-year lock unless stated:

| Policy | Options |
|---|---|
| Living standard | as `01 §6` |
| Research split | Equal (⅓ of researcher output to each field) or Focused on one field (60 / 20 / 20); the Capital's +3 per field is unaffected |
| Immigration | Open / Selective (default) / Closed; 1-year lock |
| Emigration | Free / Closed; 1-year lock |
| Refugees | Accept / Refuse; 1-year lock |
| Trade | Free (tariff 0) / Protectionist (domestic markets first, tariff 20%) |
| Taxation | Low 0.10 / Standard 0.25 / High 0.40 / Punitive 0.60; 1-year lock |
| Commerce | Credits / Mercantile / Innovation |
| Education | Minimal (`pop.skill_gain` × 0.75, free) / Standard (free) / Generous (× 1.5 at 0.2 credits per pop per 30 days); 1-year lock |
| War philosophy | defensive / unrestricted |
| AI and automation stance | flavour for 1.0 events |

Conceding a party's demand is a policy change.

**Edicts.** Temporary empire buffs paid in influence or credits per 30 days, available from Era II. Built: Mining Subsidies (+10% ore for 2 years at 30 credits per 30 days), Research Grants, the Trade Charter, and the Levy (100 influence once; seizes 50% of every colony's private wealth; −10 happiness for a year). M17b's market subsidies (levels 1–3) are paid on the edicts row.

### Eras

Built for eras I–III in M7 (`01 §9.6`); Era IV is 1.0.

| Era | Gate | Unlocks |
|---|---|---|
| I Dawn | start (pre-FTL) | in-system colonies, domes, habitats, stations |
| II Expansion | Hyperspace Drive known, First Jump completed, and ≥ 1 colony outside the home system | hub designation beyond the capital, starbase size 4, a second tradition tree concurrent, edicts |
| III Consolidation | 8 colonies across ≥ 4 systems or a federation, and any tier-3 tech | factions active (1.0), trade fleets (1.0), mid-game crisis timer (1.0), Galactic Exchange (1.0), starbase size 6 |
| IV Apex (1.0) | any tier-5 tech and 20 colonies or a megastructure | megastructures, ascension paths, endgame crisis eligible, Apex buildings, Citadel tier |

Era is per empire; the **galactic era** is the highest over all empires and drives crisis eligibility. Gates are checked daily. Reaching an era pays unity and fires a flavour event. Score weights scale by era (era bonus 0 / 50 / 150 / 400). FC4's AI operations take their rank from the era (rank 1, 2 or 3 in Dawn, Expansion and Consolidation, `07 D219`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Modifiers as the one pathway | Every effect is a modifier row folded into a dirty-keyed cache; factors at base 1, terms at base 0; refold in #9 and #27 | 2026-11-06 | `07 D36` |
| Skill modifiers | `pop.skill_gain`, `skill_decay`, `skill_output`; Education policy; Meritocracy; Guilds | 2026-11-02 | `07 D27` |
| Taxation | Four tax rates, commerce policy, the Levy edict, Fair Levies | 2026-11-02 | `07 D28` |
| Tree size and shape | About 400 techs as a climb to Kardashev Type III (replaced 76 techs over tiers 0–4) | 2026-11-21 | `07 D278 §15` call 1 |
| How research is chosen | An open tree with goals; research, reverse engineering and scavenging; use of unknown tech builds insight and allows skipping (replaced the Stellaris-style draw of three, `01 §9.1` before the rework and `07 D36`) | 2026-11-21 | `07 D278 §15` call 2 |
| Forks | No forks | 2026-11-21 | `07 D278 §15` call 3 |
| Deeds and diffusion | Deeds gate techs; diffusion, with free trade with stronger empires raising the catch-up rate | 2026-11-21 | `07 D278 §15` call 4 |
| Kardashev rating | Measured from energy captured | 2026-11-21 | `07 D278 §15` call 5 |
| Megastructures in the tree | Swarm part of the Type I gate, Sphere of Type II, speculative tech the path to Type III; a separate session scopes the great works (Prima, Maiora, Magna, Grandia Opera Astralia) | 2026-11-21 | `07 D278 §15` call 6 |
| Unknown tech | Use to 50%, dismantle for more | 2026-11-21 | `07 D278 §15` call 7 |
| The Dyson line | A basic Swarm known from the start; techs raise its enclosure; a full Sphere takes the star's whole output; a Dyson Aperture later; Type I is a full Swarm, Type II a closed Sphere; anchors and costs re-set to the pace | 2026-11-25 | `07 D278 §16` |
| The AI and the great works | The AI builds the great works as part of development, triggered by having the resources; a type needs its structure for every empire | 2026-11-23 | `07 D278 §15` call 8 |
| Track T's cost calls | Tier 3 at twice its first cost (base 2,400, tier 4 4,800, tier 5 10,000), replacing `07 D278 §6`'s starting base of 1,200 / 3,400 / 9,000; energy anchors 45 / 2,000 / 50,000 / 1,000,000 (**T5b, `07 D278 §16`:** tier 3 at 4,600, carrying Type I's timing, and the anchors 45 / 1,200 / 15,000 / 1,000,000, set from the Dyson line's yields); idle bank capped | 2026-11-23 | `07 D278 §15` |
| Ethics model | Stellaris's eight ethics (replaces `01 §9.3`'s Collectivist–Individualist axis with Authoritarian–Egalitarian in CV1) | 2026-11-16 | `07 D230` call 1 |
| Ascension paths | Psionic is the Singularity path; Synthetic after Automata contact; Biological with genemodding; new capstones; M19b leaves M19 (replaces `01 §9.8`'s three generic paths) | 2026-11-23 | `07 D283` calls 2–3 |
| Election intervals | 5 years for democracies, 10 for oligarchic councils (replaces `01 §9.5`'s 10 and 20) | 2026-11-18 | `07 D241` call 5 |
| Feudal civic | Autocracy and oligarchy only; no modifier rows; 25% of AI empires hold it | 2026-11-21 | `07 D219` |

## Not yet built or still open

- **Left from T2 to T4** (all three are built, 2026-11-24; `07 D278`'s "Built" blocks): taking a captured building apart and the colony screen's unknown-tech mark, which wait for a recorded capture (`07 D284`); `unlocks.deposits`, left out of T5 for T4's field readout or the exotic chain.
- **T5a** (built 2026-11-25, awaiting merge): six modifier keys on trade and intrigue (`exchange.fee`, `subsidy.cost`, `smuggle.detect`, `insurance.premium`, `intel.ops`, `intel.defence`) on thirteen techs, the two bench rows of `07 D278 §13`, and a pace line in `tools balance`. One tracked `07 D147` line still misses: no empire knows every MVP tier-3 tech by year 200 on seeds 1–2.
- **T5b, the Dyson line and the pace** (the owner, 2026-11-25; `07 D278 §16`): measured over 300 years, Type I came in years 56–86 against the 100–140 wanted and nobody reached Type II, because a Swarm and a Sphere were percentages, not energy. Every empire will know a basic Dyson Swarm from the start; a ladder of techs raises how much of the star it captures; a full Sphere takes the star's whole output; a Dyson Aperture later removes the Sphere's cost to its system. Open with the owner: how the ladder works, the section count, the Aperture's place, what the dimming is, and Type III's anchor before the great works land. Also found: no AI can order a Sphere, because none holds machine tools.
- **Ascension paths, their capstones and the ascension victory**, after the Expanse (`D281`), the precursors and Automata worldbuilding (`D280`) and a genemodding scope. `D280` and `D281` are named in `07 D283` but have no entry in `07` yet.
- **Era IV** (Apex) and its unlocks are 1.0.
- **Open with the owner:** how the eras of `01 §9.6` sit beside the tech tree's three Kardashev ages. Era IV's gate names a tier-5 tech and a megastructure, which under the rework reads as Type I (`07 D278 §15`; DESIGN's open questions).
- **The great works' roster** and their four scales are scoped in a separate session.
- **The civic count** for 1.0: about 30 stands as the proposal; the owner has not answered (`07 D230`).
- **Ethics drift** (`01 §9.3`) is M15's remainder and not built; cohort ethics vectors are static.
- **Stale text to fold:** `01 §4.1` still says the Hyperspace Drive is "always among the first three physics options offered"; `01 §3`'s research row prices it at 168 points (the old `1 + 0.02 × techsKnown` multiplier), where `01 §9.1` gives 161; `01 §9.5` still gives election intervals of 10 and 20 years; `05`'s M22 row still plans 250 techs.

## Discrepancies noted

- **Era III's unlocks against systems built without an era gate.** `01 §9.6` lists "Factions active (1.0), trade fleets (1.0), ... Galactic Exchange (1.0)" as Era III unlocks. FC1's parties form on every empire's politics day with no era condition in its plan or build record (`07 D214`); M17c founds the Galactic Exchange "on the third pact or monthly when absent" (`00-status`, 2026-11-23); M17a's trade fleets name no era. The sources do not say whether these systems are gated by Era III as built.
- **Tech counts gated by deeds.** `01 §9.1`: "49 are gated by deeds". `07 D278 §4`: "About 60 techs are gated." The T1 build record gives 399 techs but no deed count.

## Sources

- 01 §9 Civilisational progression (§9.1 to §9.8)
- 01 §3 Empires and species (opening research row)
- 02 §5 Modifiers
- 05 M7, 05 T and the 1.0 milestones
- 07 decisions: `D27`, `D28`, `D36`, `D219`, `D230`, `D241`, `D278`, `D283`, `D284`
- 20 Tech hooks
- 17 Civilisations, discovery and terraforming (§0, §1)
- DESIGN.md (open questions, change log)
- 00-status.md (merge records for T1 and T2's first slice)

## Related pages

- [Overview](Overview)
- [Civilisations and species](Design-Civilisations-and-species)
- [Factions and characters](Design-Factions-and-characters)
- [Megastructures](Design-Megastructures)
- [Economy and resources](Design-Economy-and-resources)
- [Pops and society](Design-Pops-and-society)
- [Events and crises](Design-Events-and-crises)
- [AI and victory](Design-Ai-and-victory)
- [Starts and exploration](Design-Starts-and-exploration)
- [Roadmap and status](Design-Roadmap-and-status)
