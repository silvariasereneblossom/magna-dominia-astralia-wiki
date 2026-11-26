# Civilisations and species

**Status:** designed, as of 2026-11-23. The MVP species, traits, ethics on cohorts and the precursor chain are built (M11, M14); the 1.0 scopes of `docs/17` (CV1 ethics and civics, SP1 species, LR1 lore and precursors, DX1 discovery sites) carry the owner's calls but are not placed on a track.

This system is what a civilisation is made of: the species its pops belong to, the ethics they hold, the civics that shape its government, the layered backstory of the precursors who came before, and the discoveries that reveal it. In the MVP an empire picks a species and traits in the creator and finds one precursor chain across the galaxy. The 1.0 design adds eight baseline species including humans, a persona with a species of its own, Stellaris's eight ethics, about 30 civics, and a Starsector-style backstory told only through what players dig up.

## How it works

### Species (built)

A species is data (`01 §3`): a name list, a portrait id, a climate preference, and 2–4 traits from a trait list with point costs. Traits are modifiers on the cohorts of that species, so they reach the economy through the modifier pathway ([Progression](Design-Progression)).

Built in M14 (`07 D99`): four species, each with combat traits as its own, and ten traits on a 2-point budget. The economic traits are the empire creator's choices.

| Species | Its own traits |
|---|---|
| Vaelan | Hardy |
| Orrin | Nimble, Keen-Eyed |
| Sesk | Hardy, Stalwart |
| Lumen | Keen-Eyed |

| Trait | Cost | Effect |
|---|---|---|
| Industrious | +2 | +4% job output |
| Intelligent | +2 | +5% to each research field |
| Rapid Breeders | +1 | +10% growth |
| Charismatic | +1 | +10% influence |
| Hardy | +1 | +5% hull |
| Stalwart | +1 | +5% armour |
| Keen-Eyed | +1 | +5% accuracy |
| Nimble | +1 | +5% evasion |
| Slow Learners | −1 | −15% skill gain |
| Quarrelsome | −1 | −3 stability |

The species' own traits are combat traits because economic defaults (Industrious, Rapid Breeders, Charismatic, Intelligent) grew seed 1's year-150 world from 127 to 144 colonies and pushed one AI plan over its CPU tripwire (`07 D99`). Leader and empire names come from each species' list; there are no free-text names yet (`07 D97`). FC2 added given and family names per species in `notables.json`, moddable (`07 D216`).

The Sol start offers a Human preset: continental preference, no traits (`01 §3.1`, [Starts and exploration](Design-Starts-and-exploration)).

### The empire (built)

An empire has a name, an adjective, a flag (shape and 2 colours), a species, **ethics** (3 points over the ethic axes; a fanatic pole costs 2 points), an **authority**, 2 **civics** and an origin (MVP: Standard only) (`01 §3`). Authority, civics and policies are described on [Progression](Design-Progression).

### Ethics

**As built** (`01 §9.3`, `§6.1`): four axes with two poles each: Collectivist–Individualist, Militarist–Pacifist, Materialist–Spiritualist, Xenophile–Xenophobe. Each pop cohort holds an 8-vector of pole weights, each axis summing to 1. Cohort ethics feed `ethicsMatch` in happiness, so pops that disagree with the state are unhappier. In the MVP the vectors are static; FC1's parties read them, and FC4's ideology spread pushes them ([Factions and characters](Design-Factions-and-characters)).

**As decided for 1.0** (`07 D230` call 1): **Stellaris's eight ethics**, not the four-axis proposal of `docs/17`. Collectivist–Individualist becomes **Authoritarian–Egalitarian** in CV1 (`docs/19 §1`). The economic drift conditions of `01 §9.3` (austerity toward Collectivist, high trade value toward Individualist) move out of ethics and into the material blocs of the faction system, so the new axis is about hierarchy and liberty while economic interest is read from jobs (`docs/19 §3.2`).

**Drift** (`01 §9.3`, 1.0, not built): each cohort's vector will move 1% per 30 days toward a target computed from lived conditions:

| Condition | Pushes toward |
|---|---|
| Empire at war in the last 12 months | Militarist |
| 10+ years of peace | Pacifist |
| ≥ 25% of planet pops are other species, happiness ≥ 50 | Xenophile |
| Foreign empire occupied or raided the planet | Xenophobe |
| Research output per pop above empire median | Materialist |
| Low stability or recent crisis | Spiritualist |
| Trade value per pop high, living standard Comfortable | Individualist |
| Austere living standard, rationing, or shortages | Collectivist |

plus +2% per 30 days toward the empire's ethics and faction propaganda. The last two rows are the ones `docs/19 §3.2` moves into the blocs under the eight-ethics remap.

### Civics and origins (CV1, designed)

- **Civics** will grow from the built 8 (`government.json`; seven from M7 plus FC4's Feudal) to about 30, two per empire with a third unlocked by tradition. Each will be a modifier set plus at most one unique rule, all in data, gated by ethics and authority (`docs/17 §3.1`). The owner has not answered the count; about 30 stands until she does (`07 D230`).
- There is no xenophobe civic in the content; closed immigration plays its part in refusing foreigners (`07 D139`).
- **Origins** are **persona origins, not planet origins** (`07 D230` call 2): an origin is the persona's background, not an empire's start world. `docs/17`'s proposal of about 12 start scenarios (Lost Colony, Ruined Gateway Home, Precursor Heirs, Void Dwellers, Shattered Ring and others) was offered as planet origins and is not the decided form.

### Baseline species and the persona's species (SP1, designed)

The owner's calls (`07 D230` calls 4–6):

- **Humans plus seven core species**, expandable by mods.
- **The persona has a species:** a lifespan and core traits, plus traits picked at creation.
- **Machine and hive species are NPC only:** they exist in the galaxy as non-playable species.

The `docs/17 §3.2` proposal within those calls: a species row will carry a climate preference, a lifespan in years, a trait set from the point budget, a ship set (G2's generator parameters), a portrait set and name lists, plus an AI personality bias and a home template. The seven alien species will span the climate groups and include one lithoid and one aquatic. The persona's lifespan will feed its mortality (M12b) and interact with Life Extension; species traits will apply to the persona as character modifiers where they make sense (intelligent → science skill XP); a short list of persona-only birthright traits will be picked at creation.

**Habitability per species** will become a function of the planet's four terraforming parameters against the species' preference, replacing the climate-group table, recomputed only when a parameter changes (`docs/17 §3.6`, TF1; [Colonies and surfaces](Design-Colonies-and-surfaces)).

**Genemodding** for both the persona and the empire's species is unscoped; the owner tied it to the Biological ascension path on 2026-11-23 (`07 D283`). Techs that point at it already exist as hooks: `gene_tailoring` (changes to species traits), `xeno_hybridisation` (traits taken from other known species), `clade_engineering` (a subspecies suited to a chosen world type) (`docs/20`).

### Precursors and the layered backstory

**Built** (M11, `07 D85`, `D86`):

- **Six precursor sites** placed at galaxy generation; each is a project of 180 days whose event gives a **clue**. Every opened deep cache also gives a clue (S1's caches, [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)).
- **The sixth clue** reveals the precursor home system and fires `precursor_home`; the home is the lowest system id among the six sites, and it is surveyed in full. `01 §4.2` describes the reward as "relic + unique system modifier".
- The deep-space **Automata** guard caches; the **Precursor** hull lineage comes from salvage; ruined mega- and gigastructures sit in deep space and sometimes in normal space ([Megastructures](Design-Megastructures)).
- Under the tech rework, the precursor chain is about twelve techs across the three ages with no ordinary way in: they are gated on clues and reached by insight (`07 D278 §5`, designed for T2).

**Designed** (LR1, `docs/17 §3.4`; names are placeholders for the owner):

| Layer | What it is |
|---|---|
| The Elders | an ancient, long-extinct civilisation; made the gigastructure ruins, the deep-space anomalies and the oldest relics |
| The Concord | a recent galactic polity that built the original Gateway network and the beacon corridors; fell in the Severance, whose cause stays ambiguous until late; its relics are the colony items and blueprints; its automata are the deep-space Automata faction |
| The Successors | the present empires and pirates, the Concord's scattered heirs |
| A dormant power | a Concord remnant state sealed in the abyss, which wakes as one of M20's crises |

The owner's calls (`07 D230` calls 9–11): the fallen precursors are an offshoot of the storyline precursor (the dormant power is a Concord offshoot); lore is delivered as a **codex of fragments**; **the owner writes the lore**, and the spec session keeps only placeholders and hooks. Each fragment will be a log, an inscription or an item description; chains of fragments will unlock a lost tech or reveal a site, and the built precursor chain becomes the Concord chain's first act.

FC4 built the hooks for the owner's 20–30 authored characters: a condition list (an empire of the character's species reaching an era, a dug precursor site, a crisis) read when the roster fills a vacancy, and codex fragments revealed through Converse. No characters ship until she writes them (`07 D219` call 6).

### Discovery

**Built** (M3, M6, M8c, M11, F1; summarised in `docs/17 §1`):

| Mechanic | Rule |
|---|---|
| Survey | per body; an anomaly roll of p 0.18 (× 1.5 in a system with a research-related modifier) |
| Anomalies | level 1–10, weighted toward low early; a research project of `20 × level / (1 + 0.1 × skill)` days ending in a weighted event from the anomaly's pool; no failure outcome (`01 §4.2`) |
| Archaeology | sites at about 1 per 25 systems; 3 chapters at difficulty 4 as built, each `30 × difficulty / (1 + 0.15 × skill)` days then `d10 + skill ≥ difficulty` ([Events and crises](Design-Events-and-crises)) |
| Deep space | caches, stashes and rich fields, found by sensors along routes or by a deep survey; odds rising with depth (`07 D45`) |
| Hex features | ruins, alien life, crash sites, monoliths, hazards, ore veins found by expeditions (M6) |
| Flight-view signals | `signal` sites placed per (seed, cell) that carry the hyperspace incident events when flown to ([Flight view](Design-Flight-view)) |

**Lost technology** (`07 D230` call 7: "a visible tree plus lost technology that has to be discovered manually") is realised by the tech rework: 4 techs are found rather than researched, and reverse engineering, scavenging and the use of unknown items build insight toward any tech (`07 D278 §5`; [Progression](Design-Progression)). **Risky tech** is approved (call 8): `docs/17 §3.3` proposed a handful of lost techs with risks such as an AI-uprising risk feeding M20's crises. The tech rework's events table names "an AI-rights chain after Sapient AI; a containment failure; a precursor signal answered" among about a dozen `on_tech` events (`07 D278 §7`).

**Discovery sites** (DX1, designed; `07 D230` calls 12–14):

- Sites will be a moddable data registry **placed at galaxy generation** in hyperspace, deep space and systems, revealed by sensors and survey.
- Proposed kinds (`docs/17 §3.5`): derelict ships (salvage hulls or blueprints), research stations (lost tech, with a guard chance), cryosleeper vaults (pops plus a crew boost), beacon wrecks (lore and corridor charts), comm relays (reveal regions), automata factories (guarded; Automata hulls).
- Each site will have a condition (intact, damaged, looted), danger set by depth, and a **layered payout: reveal, then dig, then deep dig**.
- **Planet conditions with ratings**, from survey: deposits sparse, moderate, abundant, rich, ultrarich; ruins scattered, widespread, extensive, vast. The ratings will feed district output and planetary exploration.
- Anomalies stay, with their pools rewritten to point into the codex and the site registry; F1's signals and simulated incidents will use the same content.

**Planetary exploration** (PX1, designed; calls 18–20): teams led by a leader, a companion or the persona in person will crawl hidden hexes on timers, finding rated resource nodes that boost districts, relics usable as items or planet bonuses, treasure caches and dangers that hook into ground combat ([Colonies and surfaces](Design-Colonies-and-surfaces), [Ground combat](Design-Ground-combat)).

### The 1.0 milestones of `docs/17`

| # | Milestone | Contents | Est. LOC |
|---|---|---|---|
| CV1 | Ethics, civics and origins | the ethics remap, about 30 civics, origins; merges into M15 | ≈ 1.8k (+ content) |
| SP1 | Species | about 8 baseline species, the persona's species and birthright, the habitability function | ≈ 1.0k (+ content, portraits) |
| TT1 | Tech tree to 1.0 | replaced by track T's rework (`07 D278`, T1–T5) | ≈ 1.2k |
| LR1 | Lore and precursors | the eras, the codex, fragment chains, the dormant remnant hook into M20 | ≈ 1.0k (+ lore) |
| DX1 | Discovery sites | the site registry, layered digs, planet condition ratings, anomalies re-pooled | ≈ 1.6k |
| TF1 | Terraforming | the parameter model, projects with physical inputs, side effects, new kilostructures | ≈ 1.8k |
| PX1 | Planetary exploration | expedition teams, hex crawl, resource nodes, relics as items, surface dangers | ≈ 2.0k |

Suggested order after the MVP: SP1 and CV1, then DX1 and LR1 together, then TT1, TF1 and PX1 (`docs/17 §4`). FC1–FC4's build order put CV1 first, but FC1 shipped on the four built axes with the rename left as "a content and string change on top" (`07 D214`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| MVP species and traits | Four species with combat traits; ten traits on a 2-point budget; economic traits are the creator's | 2026-11-14 | `07 D99` |
| Precursor chain as built | Clues from the six precursor sites and every opened deep cache; the home is the lowest-id site's system (the alternative, clues from anomaly outcomes, was not taken) | 2026-11-14 | `07 D85` |
| Ethics | Stellaris's eight ethics (replaces the four-axis set of `01 §9.3` and the `docs/17` proposal) | 2026-11-16 | `07 D230` call 1 |
| Origins | Persona origins, not planet origins | 2026-11-16 | `07 D230` call 2 |
| Civic count | Not answered; about 30 stands | 2026-11-16 | `07 D230` call 3 |
| Baseline species | Humans plus seven core species, expandable by mods | 2026-11-16 | `07 D230` call 4 |
| The persona's species | Yes: lifespan, core traits, traits picked at creation | 2026-11-16 | `07 D230` call 5 |
| Machine and hive species | NPC only | 2026-11-16 | `07 D230` call 6 |
| Research shape | A visible tree plus lost technology discovered manually (realised by `07 D278`) | 2026-11-16 | `07 D230` call 7 |
| Risky tech | Yes | 2026-11-16 | `07 D230` call 8 |
| Precursor eras | The fallen precursors are an offshoot of the storyline precursor | 2026-11-16 | `07 D230` call 9 |
| Lore | A codex of fragments; the owner writes it | 2026-11-16 | `07 D230` calls 10–11 |
| Discovery | Sites placed at generation; reveal, dig, deep dig; rated planet conditions | 2026-11-16 | `07 D230` calls 12–14 |
| Planetary exploration | By teams or by the persona; relics as items or planet bonuses; rated nodes that boost districts | 2026-11-16 | `07 D230` calls 18–20 |
| Authored characters | Hooks now, the owner's characters later | 2026-11-21 | `07 D219` call 6 |
| Genemodding | To be scoped for persona and species, tied to the Biological path | 2026-11-23 | `07 D283` call 2 |

## Not yet built or still open

- **CV1:** the eight-ethics remap, about 30 civics, persona origins. The civic count awaits the owner.
- **SP1:** the eight baseline species, the persona's species, birthright traits, the habitability function. Species art (portrait ids) is a hook left by M14.
- **LR1:** the codex, fragment chains, the dormant remnant; the lore itself, which the owner writes.
- **DX1:** the site registry, layered digs and condition ratings.
- **Ethics drift** (M15's remainder).
- **Genemodding** has no scope yet.
- **The precursors and Automata worldbuilding** (`D280`) is named in `07 D283` as a scope that the Synthetic ascension path waits on; it has no `07` entry.
- **Stale text to fold:** `docs/17 §1` describes the built precursor chain as "6 clues from survey and anomaly outcomes", where `07 D85` built clues from the six precursor sites and opened deep caches; `01 §3`'s trait examples (Industrious +10% ore, Thrifty, Sedentary, Nonadaptive) are not the built traits of `07 D99`.

## Discrepancies noted

- **Whether the three-era backstory is adopted.** `docs/17 §3.4` call 9 offered "A (recommended): three eras (Elders, Concord, Successors) plus one dormant remnant power", B one precursor empire, C two eras with no dormant power. The owner's recorded answer (`docs/17 §0` row 9 and `07 D230`) is "The fallen precursors are an offshoot of the storyline precursor (the dormant power is a Concord offshoot)". The answer confirms a dormant power descended from the Concord but does not say whether the Elders layer stands.

## Sources

- 01 §3 Empires and species, 01 §4.2 Interstellar exploration, 01 §9 Civilisational progression (§9.3)
- 17 Civilisations, discovery and terraforming: §0 the owner's calls, §3 proposals
- 19 Factions and characters (§1, §3.2)
- 20 Tech hooks
- 07 decisions: `D45`, `D85`, `D86`, `D97`, `D99`, `D139`, `D214`, `D216`, `D219`, `D230`, `D278`, `D283`
- DESIGN.md (open questions, change log)

## Related pages

- [Overview](Overview)
- [Progression](Design-Progression)
- [Factions and characters](Design-Factions-and-characters)
- [Starts and exploration](Design-Starts-and-exploration)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Pops and society](Design-Pops-and-society)
- [Megastructures](Design-Megastructures)
- [Events and crises](Design-Events-and-crises)
- [Ground combat](Design-Ground-combat)
- [Flight view](Design-Flight-view)
- [Modding and data](Design-Modding-and-data)
