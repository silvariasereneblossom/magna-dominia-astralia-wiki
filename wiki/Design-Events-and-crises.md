# Events and crises

**Status:** in progress, as of 2026-11-23. The events engine, the MVP's 40 events, three situations, archaeology, first contact and the precursor chain are built (M3, M6, M11); faction agitation situations come from FC1; pirates are the one crisis in play; the marauder, awakened-empire and endgame crises are designed for M20.

Events are the narrative glue of the game: anomalies, digs, first contacts, leader deaths, era transitions and hyperspace incidents all run through one data-driven grammar of triggers, conditions, options and effects. Situations are progress bars a player steers with an approach and a price. Crises are galaxy-scale threats that arrive with the later eras and test every empire at once. The rule throughout is that only a real choice interrupts the player, and nothing tests which clock (real time or turns) is running.

## How it works

### The event grammar (built)

Events are data (`01 §12.1`; schema in `06-data-schemas.md`). Each has:

| Part | Content |
|---|---|
| Trigger | what can fire it; no clock-specific triggers exist |
| Scope | empire, planet (colony), fleet, leader, system, hex |
| Conditions | an expression tree |
| Timing | mean time to happen (MTTH) in days, or a fixed probability |
| Text and options | 1–4 options, each with conditions, costs and effects |

**Triggers built:** `on_survey`, `on_project_complete` (M3); `on_hex_survey` (M6); `on_colonize`, `on_tech`, `on_contact`, `on_leader_death`, `on_battle_end` (for each empire side) and `on_daily_mtth` (M11); `on_hyperspace` (F1); `on_fired` for chained events (`07 D85`).

**Conditions built:** `all`, `any`, `not`, `flag`, `projectKind`, `stat`, `random`, `feature`, and the `hex.*` stat keys. The tech rework adds stats read from existing state for deeds: `empire.colonies`, `empire.systems`, `empire.contacts`, `empire.pacts`, `empire.era`, `empire.ktype` (built in T2's first slice), `empire.structures`, `empire.hubs`, and `has` for a tradition tree, perk or civic (`07 D278 §4`).

**Effects built:** `ledger`, `flag`, `event` (start another), `reveal` (M3); `stock`, `hex` (M6); `situation`, `leader` (xp, trait, loyalty, successor), `clue` (M11). `01 §12.1`'s list also names `add_modifier`, `spawn_fleet`, `change_ethics` and `kill_leader`; the M3–M11 build records do not say which of those are built. The `tech` effect (`{ "tech": "progress", "id", "points" }`) is specified in `06 §11` and will be built in T2 (`07 D278 §5`).

**MTTH.** Rolled daily per scope with `p_day = 1 − 2^(−1 / mtthDays)`. As built, the roll is a seed hash of the event, the scope and the day, so no random stream moves and only a hit evaluates the conditions; MTTH events take empire and colony scopes only (`07 D85`).

**Popups and timeouts.** Only events with a real choice interrupt; everything else goes to the notification feed. Hidden events (no popup) drive flavour outcomes. In TB, choices appear in the turn report. In both modes an option's effects apply on the tick after it is chosen, and an unanswered event takes its default option after `timeoutDays` (`01 §12.1`, `§15`). Event options whose conditions fail show greyed with their reason (M-QA2, `00-status`).

**Content.** The MVP holds 40 events: 10 anomaly events (M3), the feature events of hex surveys (M6: ruins and its dig, alien life, crash site, monolith and its hum, hazard, ore vein), two era events (M7), and M11's 20 events in `events/people.json` (`07 D85`). F1 adds 8 hyperspace incident events (below). `datacheck` refuses an MTTH event without days or with the wrong scope, an `on_fired` event nothing fires, an unknown situation, and missing strings (`05 M11`). 1.0 plans 300 events (`05` M22).

**`on_tech` events.** The trigger exists, and no content used it before the tech rework. `07 D278 §7` plans about a dozen with choices: an AI-rights chain after Sapient AI, a containment failure, a precursor signal answered.

### Anomalies (built)

A body survey rolls an anomaly at p 0.18 (× 1.5 in a system with a research-related modifier). Anomaly level 1–10 is weighted toward low early; the ship's scientist runs a research project of `20 × level / (1 + 0.1 × skill)` days, whose completion fires a weighted event from that anomaly's pool: a small resource, a deposit reveal, a tech boost, a chain start, a special project. There is no failure outcome; bad outcomes exist but are rare and event-driven, not a dice fail (`01 §4.2`).

### Archaeology (built, M11)

- Surveying a site's body gives the empire chapter 1 as a project.
- A science ship digs each chapter for `30 × difficulty / (1 + 0.15 × skill)` days, then rolls `d10 + skill ≥ difficulty` as a seed hash. A failure digs the chapter again with no reward; a success fires its chapter event and opens the next.
- The last chapter marks the site dug for that empire and pays a relic, tech or event chain (`01 §4.2`).
- `01 §4.2` specifies 3–6 chapters at difficulty 2–6. **As built** (`07 D86`): 3 chapters at difficulty 4, so a skill-3 scientist digs 3 × 83 = 249 days, the roll always passing.
- A dug site sets a flag that gates techs such as Xeno-archaeology (`07 D278 §4`).

### First contact (built, M11)

The sensor recovery pass starts a 60-day contact project for each empire whose ships or starbases an empire newly sees. The project's timer opens comms both ways (the flag diplomacy reads), ends the other side's project, and fires `on_contact` for both (`07 D85`). 1.0 will add contact events with choices: hostile, cautious, open (`01 §4.2`). The tech hook `universal_translators` will remove "the language delay on first contact" (`docs/20`). First contact and three species met gate Xeno-linguistics and Comparative Xenology (`07 D278 §4`).

### The precursor chain (built, M11)

A precursor site is a 180-day project like a chapter, and its event gives a clue. Every opened deep cache also gives a clue. The sixth clue reveals the precursor home system (the lowest system id among the six precursor sites), surveys it in full and fires `precursor_home` (`07 D85`, `D86`). The 1.0 lore design makes this chain the first act of the Concord's story, told through a codex of fragments ([Civilisations and species](Design-Civilisations-and-species)).

### Hyperspace incidents (built, F1)

F1 shipped 8 incident events on the `on_hyperspace` trigger. The owner ruled (2026-11-14) that the flight view is physical, with no incident dice: `signal` sites placed per (seed, cell) carry the incident events when the piloted fleet flies to them, and hostile groups are placed per (seed, cell, month) by depth. Simulated travel rolls (DESIGN pillar row; `07 D123`–`D125`). The rules are on [Flight view](Design-Flight-view).

### Situations (built)

A situation is a progress bar 0–100 with stages at 25, 50, 75 and 100, an **approach** chosen by the player that changes the per-30-day delta (applied daily) and the stage effects, and an outcome at 100 or at 0 (`01 §12.2`).

**As built** (`07 D85`, `D86`): a daily watch tests each situation's start over its scopes and keeps an arming row while the start holds; after its days the situation starts. A running situation moves `(base + approach) / 30` a day, pays the approach's cost daily, holds the approach's modifier rows, fires each stage once, and ends at 100 (its `up` event) or 0 (its `down` event). The scale starts at 20 with a base of +2 a month. A colony lost to another empire ends its situation.

| Situation | Starts when | Approaches (per month) | Outcome |
|---|---|---|---|
| Planetary Unrest | stability < 25 for 30 days | ignore +3; concede −6 with +0.05 happiness; suppress −4 for 2 influence a month | stage 50 protests; at 100 a revolt (−0.1 happiness and −5 stability on the colony for a year); at 0 calm (+5 stability for 6 months) |
| Economic Recession | credits < 0 for 180 days | ride out +2; austerity −5 with −5% job output; stimulus −3 for 3 influence a month | at 100 a crash: −10% job output for a year |
| Pirate Haven | piracy > 0.7 for 360 days | ignore +2; patrols −4 for 15 credits a month; bounties −6 for 2 influence a month | at 100 a lair: +20% piracy growth for two years |

The default approach is the one that worsens. The AI's situations advisor picks, on each planning day, the most negative delta it can carry: an influence cost while six months of it still leave a starbase claim's bill, a credit cost within its margin (`07 D86`). Resolving a situation at an ungoverned colony needs the persona's fleet in orbit ([Factions and characters](Design-Factions-and-characters)).

**Faction Agitation** (FC1, `07 D214`): a party under 30 approval for 24 months starts a `faction_agitation_<base>` situation with stage effects per base at 25 and 50 (for example Mining: miner output −25% at 50), progressing `(30 − approval) / 3 × rate` a month; FC4 adds a stage at 75 that starts the party leader's plot. Its endings at 100 are the faction system's ([Factions and characters](Design-Factions-and-characters)).

**Named for later** (`01 §12.2`): Expedition Mystery, Succession Crisis (autocracy on ruler death), Plague (1.0). 1.0 plans 10 situations in all (`05` M22).

### Era transitions

Reaching an era pays unity and fires a flavour event (`01 §9.6`; [Progression](Design-Progression)). The galactic era, the highest over all empires, drives crisis eligibility.

### Crises

| Crisis | Trigger | Shape | Status |
|---|---|---|---|
| Pirates | always on | emergent, local: regional piracy levels, raiders, bases, loot pools | built (M5, M8c; [Trade and logistics](Design-Trade-and-logistics)) |
| Marauder raids | galactic Era III | periodic raiding fleets from a marauder pocket; can be paid off | designed, M20 |
| Awakened empire | galactic Era IV, 10% per year | a fallen empire reactivates and demands tribute | designed, M20 |
| Endgame crisis | crisis year (default 200) if galactic Era ≥ III | one of three: Extradimensional Incursion (a portal; waves scale with galactic fleet power), Machine Uprising (robot pops revolt in the most synthetic empires), Devourer Swarm (enters from the rim, consumes planets); escalation stages every 5 years; defeat gives a galaxy-wide reward and a score bonus | designed, M20 |

M20 is estimated at 2.4k lines (`05` 1.0 table). The 1.0 balance target asks that the crisis spawns at the crisis year (`01 §18`).

**Ties to other designs:**
- The dormant power of the 1.0 backstory is a Concord remnant sealed in the abyss, which wakes as one of M20's crises; the owner's call is that the fallen precursors are an offshoot of the storyline precursor (`docs/17 §3.4`, `07 D230` call 9).
- Risky tech is approved (`07 D230` call 8); `docs/17 §3.3` proposed lost techs with an AI-uprising risk feeding M20.
- The score's crisis term, `crisisDefeated × 500`, waits for the crises (`07 D98`; [AI and victory](Design-Ai-and-victory)).
- The tech hook `existential_risk_management` "Reduces the damage of galaxy wide crises" (`docs/20`).
- Ethics drift pushes toward Spiritualist after a recent crisis (`01 §9.3`, not built).

### RTS and TB

Events, situations and their stages run on daily systems and heap timers in both modes. In RTS a choice can pause the game; in TB choices collect in the turn report and apply on the first tick of the next turn. No event trigger, modifier or rule may test which clock is active (`01 §15`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Narrative content as data | Leaders, situations, archaeology, contact days and the precursor chain in `narrative.json`; M11's 20 events in `events/people.json` (40 in all) | 2026-11-14 | `07 D85` |
| MTTH rolls | A seed hash per event, scope and day; empire and colony scopes only | 2026-11-14 | `07 D85` |
| Situations | Watched daily with arming rows; the default approach worsens; the AI picks the fastest-falling approach it can afford | 2026-11-14 | `07 D85`, `D86` |
| First contact | Starts in the sensor recovery pass, 60 days per pair; comms open both ways | 2026-11-14 | `07 D85` |
| Archaeology as built | 3 chapters at difficulty 4 (replaces `01 §4.2`'s 3–6 chapters at difficulty 2–6 as the built numbers) | 2026-11-14 | `07 D85`, `D86` |
| Precursor clues | From the six precursor sites and every opened deep cache | 2026-11-14 | `07 D85` |
| Unrest revolt | Costs its colony happiness and stability, not empire influence | 2026-11-14 | `07 D86` |
| Hyperspace incidents | The flight view is physical with no incident dice; simulated travel rolls | 2026-11-14 | DESIGN pillar row, `07 D123`–`D125` |
| Faction Agitation | A situation per agitating party, with stages by base | 2026-11-19 | `07 D214` |
| Plot stage | Agitation's stage 75 starts the party leader's plot | 2026-11-21 | `07 D219` |
| Dormant power | The fallen precursors are an offshoot of the storyline precursor | 2026-11-16 | `07 D230` call 9 |
| Tech events and the `tech` effect | About a dozen `on_tech` events; the `tech` effect pays insight | 2026-11-21 | `07 D278 §5`, `§7` |

## Not yet built or still open

- **M20, crises:** marauder raids, the awakened empire, the three endgame crises with escalation; the score's crisis term.
- **The `tech` event effect and the `on_tech` content** (T2 and later).
- **Contact events with choices** (1.0).
- **Situations named for later:** Expedition Mystery and Plague. **Succession Crisis** is named in `01 §12.2` as a situation; FC3 built AI succession wars as civil wars instead (`07 D217`), and no source says whether the situation will still exist.
- **Fleet, leader and system scopes for MTTH events** (built: empire and colony only).
- **1.0 content:** 300 events and 10 situations (`05` M22).
- **DX1's discovery sites** will re-pool the anomalies toward the codex and the site registry ([Civilisations and species](Design-Civilisations-and-species)).

## Discrepancies noted

None found.

## Sources

- 01 §12 Events, situations, crises, 01 §4.2 Interstellar exploration, 01 §15
- 05 M11, 05 1.0 milestones
- 07 decisions: `D85`, `D86`, `D98`, `D123`–`D125`, `D214`, `D217`, `D219`, `D230`, `D278`
- 17 Civilisations, discovery and terraforming (§3.3, §3.4)
- 20 Tech hooks
- DESIGN.md (M11 autonomous calls, the flight-view pillar row)
- 00-status.md (M3, M6 and M-QA2 records)

## Related pages

- [Overview](Overview)
- [Starts and exploration](Design-Starts-and-exploration)
- [Civilisations and species](Design-Civilisations-and-species)
- [Factions and characters](Design-Factions-and-characters)
- [Progression](Design-Progression)
- [Flight view](Design-Flight-view)
- [Trade and logistics](Design-Trade-and-logistics)
- [AI and victory](Design-Ai-and-victory)
- [Time and modes](Design-Time-and-modes)
- [Modding and data](Design-Modding-and-data)
