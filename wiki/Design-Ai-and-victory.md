# AI empires and victory

**Status:** built, as of 2026-11-23. AI empires (M9, with M10's war AI and the `07 D237` budget) and the score, domination and economic victories (M14) are built; the AI's megastructure advisor (M19c), the ascension victory and the score's crisis term are not.

AI empires play the same game as the player: the same rules, the same command API, the same markets, so their economies are real and can be traded with, blockaded or starved. Each has a personality that weights what it builds, researches, signs and attacks, and it plans once a month on its own staggered day. A game ends at the end year with the highest score, or earlier by domination, economic dominance or (when built) ascension; endless mode removes the end year.

## How it works

### What the AI is (built, `01 §13`)

- **Same rules, same commands.** The AI issues the same `Command`s as the player, including market trades and trade policies; its plan is validated like any player's (`01 §13`, `07 D218` call 9). In Era I every AI exploits its home system (stations, domes, habitats) and researches the Hyperspace Drive before anything interstellar.
- **Cadence.** Each AI empire runs its planner once every 30 days on its own planning day and a cheap reactive layer every day; its markets, growth and research resolve daily like everyone's. The reactive layer walks only the contested set: an AI's unarmed fleet idle in a contested system leaves for home, and armed fleets fight or disengage by the combat rules (`07 D55`). Each plan is timed against a CPU tripwire (250 µs × calibration, judged on each empire's median of nine timings, `07 D110`; [Architecture and performance](Design-Architecture-and-performance)).
- **No special treatment.** The AI takes the same colony founding gate and civil upkeep as the player, and the same mitigations; only the persona's skills are the player's (the spec session's reading of `07 D273`, DESIGN).

### Personalities

Personalities are data: they set utility weights and diplomatic weights. Built content (`data/content/ai.json`) holds five:

| Personality | War power ratio (`07 D116`) | Notes |
|---|---|---|
| Expansionist | 1.5 | |
| Industrialist | 2.0 | |
| Militarist | 1.0 | taxation one step up at war |
| Trader | 3.0 | immigration Open |
| Scholar | 3.0 | |

`01 §13` also lists Isolationist for the MVP and Xenophobe-Purifier for 1.0.

**Policies by personality** (`01 §13`, `07 D55`): Xenophile immigration Open and refugees Accept, Xenophobe Closed and Refuse, Trader Open, the others at the defaults; no personality changes the commerce policy. Taxation moves one step up on a planning day with a ledger deficit and one step down with a surplus above 12 months of upkeep and vitality < 0.95.

### The AI economy (built, `07 D237`)

One budget per AI empire, kept in the sim, refreshed before each plan and saved:
- a forecast of recurring income, committed upkeep (builds and colonies in flight and their sprawl step), and a margin against a reserve of 3 months of upkeep (`reserveMonths`);
- the supply balance for food, energy, alloys, rare metals and goods;
- every commitment priced by its marginal recurring cost;
- a **priority ladder:** restore the reserve by selling surplus; after 3 plans in deficit, shed the building with the least output value per upkeep; fix forecast shortages at the colony with the most idle pops; then grow within the headroom (colonies, upgrades, military, pacts).

**Expansion pace:** a colony ship is built only while the empire's colonies (starbases aside) plus colony ships in flight stay under 8 plus its pops over 40 (`07 D237`, the spec session's call). The colony founding gate of `07 D273` (pops ≥ `colonyStep × m(m + 1) / 2`, m the planetary colonies past 10) applies too. AI empires score colony sites by supply resilience (`07 D274`).

### What the player sees the AI do

| Area | Behaviour | Source |
|---|---|---|
| Diplomacy | one treaty a plan, peace first, one rival, commercial pacts from year 25 with 2 hubs and the reserve met | `07 D116`, `D242`; [Diplomacy and war](Design-Diplomacy-and-war) |
| War | declares on the weakest contact it outguns by its ratio ÷ (1 + the coalition's war-hawk weight); attacks at twice the power standing; invades on ground combat's front | `07 D116`, `D217` |
| Traditions | adopts or takes a perk in the personality's tree order when unity covers the cost | `07 D55` |
| Research | six goals from need and personality, which the engine's walk works toward (T3, built 2026-11-24) | `07 D278` |
| Great works | once a year, with none of its own under construction and a non-negative margin, orders the first work it knows whose first stage its markets hold twice over, the next Kardashev type's structure first | `07 D278 §15` call 8 |
| Situations | picks the fastest-falling approach it can afford | `07 D86` |
| Politics | concedes one demand a plan to its most-supported agitating party; suppresses only when it cannot concede | `07 D214` |
| Operations | sabotage, steal tech, rig, assassinate against rivals and the player's empire (never the persona), one live at a time | `07 D219` |
| Trade | embargoes its rivals; runs up to two smuggling routes into rivals that ban a good it makes in surplus | `00-status` (M17b, M17d) |
| Leaders | never hires leaders; generated unpaid notables fill its heir and council seats (8, cap 12) | DESIGN (FC2 calls) |
| Contracts | its planning day issues the persona's contracts | `01 §13`, `§19` |

### Difficulty (built, `07 D55`)

`GameSettings.difficulty` applies one row of `government.json` to the AI empires at the start, as modifier rows:

| Level | Effect on AI empires |
|---|---|
| Cadet | −25% job output |
| Captain (default) | none |
| Admiral | +10% output, +10% research |
| Grand Admiral | +25% output, +10% research |

`01 §13` also gives Grand Admiral +1 free hub and full map visibility; neither is built, and the AI reads only its own visibility (`07 D55`). An AI-run empire 0 under the Employee kits gets no difficulty rows (`07 D218` call 10).

### Victory (built, M14)

The game ends at the end year (default 300) or earlier by a victory. Victories are checked yearly; a won game marks the winner and opens the end screen with the scoreboard, Continue playing, Main menu and New game, and the sim runs on if the player continues (`07 D98`). **Endless mode** (end year 0) disables the end year.

**Score**, kept yearly and shown in the top bar and on the end screen (`01 §14`, `07 D98`):

```
score = 0.5 × pops
      + 2 × colonies                       (starbase colonies aside)
      + 0.02 × researched techs' base cost
      + 0.001 × cumulative trade value
      + 10 × completed tradition trees     (adopted with every perk)
      + fleetPower / 100
      + eraBonus                           (0 / 50 / 150 / 400)
      + 500 × crises defeated              (waits for the crises)
```

| Victory | Rule | Status |
|---|---|---|
| Score | the highest score at the end year | built |
| Domination | one empire holding 60% of the colonised planets' pops; vassals' pops will count once M18b builds vassals | built (own pops only) |
| Economic | 50% of galactic cross-empire trade tonnage through the empire's hubs (source or destination market a hub or one of its spokes), judged monthly in months with at least 30 tons moving, for 20 consecutive years (`07 D146`) | built |
| Ascension | complete an ascension path and build its capstone megastructure | designed; waits for the ascension paths ([Progression](Design-Progression)) |

**Balance targets** that bear on the AI (`01 §18`): by year 50 at least one commercial pact in the galaxy and at least one war; by year 100 galactic Era III and ≥ 2 hubs in the median empire; by year 200 no empire eliminated by economics alone and ≥ 1 empire knowing every MVP tier-3 tech. `tools balance` judges them on seeds 1–3 (`07 D147`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| AI behaviour in M9 | Traditions in personality order; the Trader's immigration Open; difficulty as modifier rows without Grand Admiral's free hub and full map; the reactive layer on the contested set | 2026-11-13 | `07 D55` |
| AI plan tripwire | Judged on each empire's median of nine timings; build loops capped at 8 tries a plan | 2026-11-14 | `07 D110` |
| AI war and navy | Diplomacy advisor, war by power ratio, war orders, invasions through ground combat | 2026-11-14 | `07 D116` |
| AI economic model | One budget per AI empire in the sim with a priority ladder (replaces the separate colony-ship, warship and credits guards) | 2026-11-17 | `07 D237` |
| AI commercial pacts | From year 25 with 2 hubs and the reserve met on both sides (replaces `07 D116`'s "no commercial pact of its own") | 2026-11-18 | `07 D242` |
| Empire size | Founding gate and civil upkeep paced by pops, for every empire alike | 2026-11-21 | `07 D273` |
| AI and the great works | The AI builds the great works as part of development, triggered by having the resources | 2026-11-23 | `07 D278 §15` call 8 |
| AI operations | AI empires run operations against rivals and the player | 2026-11-21 | `07 D219` call 1 |
| AI megastructure advisor | On the planning day, the highest-value structure it can afford within its reserve, one at a time (designed, M19c) | 2026-11-23 | `07 D283` call 5 |
| Scores and victories | Score kept yearly; domination and end-year score built; economic victory after M10 | 2026-11-14 | `07 D98` |
| Economic victory | Monthly windows, a 30-ton floor, 20-year streak | 2026-11-16 | `07 D146` |
| Ascension capstones | New capstone structures, designed with the paths later | 2026-11-23 | `07 D283` call 3 |

## Not yet built or still open

- **M19c, the AI's megastructure advisor** (about 450 lines). Which works the AI favours beyond the type gate's structure waits for T3.
- **The ascension victory** and its capstones, after the ascension paths.
- **The crisis score term**, after M20.
- **Domination counting vassals**, after M18b.
- **Grand Admiral's free hub and full map.**
- **The station rework's AI** builds the station tier its size allows within its budget (`07 D284`, designed).
- **Stale text to fold:** `07 D146` still says only a player with commercial pacts can win economically, written before `07 D242` let AIs propose pacts.

## Discrepancies noted

- **Personalities.** `01 §13`: "Personalities (data): Expansionist, Industrialist, Militarist, Isolationist, Trader, Scholar, Xenophobe-Purifier (1.0)." `data/content/ai.json` holds five: expansionist, industrialist, militarist, trader, scholar. No decision found removes Isolationist or defers it.
- **What the AI may know.** `01 §13`: "AI never gets information it should not have except at Grand Admiral (sees full map)", and `07 D55`: "the AI reads only its own visibility". `07 D279` answer 7 (2026-11-22): "Intel is the player's only (A): the AI keeps reading the truth", and rule 15 ends "The AI reads the truth." The sources do not say whether the AI's exact reading of foreign fleet power, economy and tech counts is meant to override `01 §13`'s rule, or whether `01 §13` covers map visibility only.

## Sources

- 01 §13 AI empires, 01 §14 Victory, 01 §18
- 05 M14, 05 M10
- 07 decisions: `D55`, `D86`, `D98`, `D110`, `D116`, `D146`, `D147`, `D214`, `D217`, `D218`, `D219`, `D237`, `D242`, `D273`, `D274`, `D278`, `D279`, `D283`, `D284`
- `data/content/ai.json` (the personality list)
- DESIGN.md (autonomous calls, change log)
- 00-status.md (M14, M17b, M17d records)

## Related pages

- [Overview](Overview)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Progression](Design-Progression)
- [Factions and characters](Design-Factions-and-characters)
- [Events and crises](Design-Events-and-crises)
- [Economy and resources](Design-Economy-and-resources)
- [Trade and logistics](Design-Trade-and-logistics)
- [Megastructures](Design-Megastructures)
- [Architecture and performance](Design-Architecture-and-performance)
- [Roadmap and status](Design-Roadmap-and-status)
