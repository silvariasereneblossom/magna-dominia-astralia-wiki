# Pops and society

**Status:** built, as of 2026-11-23. Cohorts, jobs, happiness, migration, skill and taxation in M4b (`07 D24`, `D26`–`D28`); the per-colony stagger in HP1; halt growth in M-DEFICIT; civil upkeep with `07 D273`; party politics on cohorts in FC1. Ethics drift and Stellaris's eight ethics are designed for 1.0 (M15's remainder, CV1).

Population is held as fractional cohorts per colony, species and stratum rather than as individual pops. Cohorts are born and die as two separate flows, fill jobs by stratum, consume a basket that responds to prices, grow happier or unhappier toward a target, learn skill by working, migrate by gravity across hyperspace, and earn income that the state taxes and that builds the colony's private wealth. These are the levers through which the player's policies reach output, growth and stability.

## How it works

### Cohorts and strata

A cohort is `{ planet, species, stratum, count: float, happiness: 0–100, skill: 0–1, ethics: float[8] }`. Strata are **worker**, **specialist** and **ruler** (1.0 adds slave and robot). A planet typically has 3–9 cohorts. A cohort's ethics vector is set at creation as `0.7 × empireVec + 0.3 × speciesVec` and does not change in MVP. Cohorts that exist at game start have skill 0.5. When pops join a cohort (births, promotion, migration) its happiness, skill and ethics become the count-weighted means. No per-pop entity exists anywhere (`07 D14`: fractional cohorts).

If the cohorts table is full, new pops join the colony's cohort of the stratum, of any species, else any of its cohorts (`07 D175`).

### Update cadence

The economy runs every tick at 1/30 of the per-30-day rates. Since HP1 (merged 2026-11-17, `07 D175`), each colony's births, deaths, skill and migration run on its own day, `(tick + colony) mod 5`, at 5 days' rates (`pops.growthEvery` 5), and happiness and stability step on the same day with 5 days of the 0.1 smoothing at once (1 − 0.9⁵). Jobs, promotion and demotion stay daily.

### Births, deaths and crowding

```
crowding   = pops / housing
fertility  = 0.009 × (0.5 + hab/200) × foodSat × (0.75 + happiness/200) × clamp(1.5 − crowding, 0, 1) × speciesGrowthMod   // per pop per 30 days
mortality  = 0.0045 × max(0.5, 1 − 0.05 × (clinics + medics)) × (1 + max(0, crowding − 1)) × (starving ? 5 : 1)
starving   = foodSat < 0.5 for ≥ 90 consecutive days
```

- Births land in the worker cohort of the same species on the planet; deaths come from every cohort.
- Two flows give population momentum: a famine keeps killing until `foodSat` recovers, and a boom overshoots housing before crowding bites. Starvation is ×5 mortality, ≈ 2.25% of pops per 30 days.
- **Housing is a soft ceiling.** With full satisfaction, happiness 50 and no clinics, births equal deaths at pops = housing; a happy colony settles a little above it (happiness 76 → ≈ 1.04 × housing) and pays the crowding penalty; an unhappy or hungry one settles below.
- Low habitability halves fertility at worst, so domes grow slowly on their own and fill by immigration: colonies are settled, not bred.
- `pop.growth` modifiers (Rapid Breeders) multiply fertility; `pop.mortality` modifiers (Clinic, Medic: −5% each) multiply mortality.

**Halt growth** (`07 D264`, built 2026-11-19). `Command::HaltGrowth` stops births at an own colony and makes migrant packets skip it until lifted; deaths, leavers and resettlement go on. The player toggles it on the colony panel, with the reason in the tooltip. The AI halts a colony whose food runs a projected 30-day deficit, whose food stock covers fewer than `budget.haltCoverDays` (90) days, and where no food producer can be placed and staffed that plan. It lifts on a positive daily food balance, or when a food producer completes there. A colony too small to staff the cheapest food producer is not halted (DESIGN, 2026-11-20).

### Jobs and job filling

Jobs come from districts, buildings and the capital; each has a stratum. Filling runs for a colony whenever its jobs or pops change. Order: ruler jobs → specialist → worker; within a stratum by job priority (player-adjustable −1 / 0 / +1, default from designation), then in `jobs.json` order. A job is listed after every job of its stratum that makes one of its inputs, so producers fill ahead of their consumers (`07 D266`; `datacheck` enforces it).

| Mechanic | Rule (per 30 days, applied daily) |
|---|---|
| Promotion | `min(freeWorkers, 0.1 × openSpecialistJobs) × (0.5 + skill)` pops move from the worker cohort to the specialist cohort (same species); ruler promotion likewise from specialists |
| Demotion | specialists unemployed for 180 days move back |
| Unemployed | still consume |

Job outputs scale with designation (+15%), stability, skill, adjacency and service, vitality, species traits, techs, buildings and edicts. The job table itself is on [Economy and resources](Design-Economy-and-resources).

### Consumption basket

| Stratum | Food | Consumer goods | Luxuries |
|---|---|---|---|
| Worker | 1.0 | 0.25 | 0 |
| Specialist | 1.0 | 0.5 | 0 (0.25 under Comfortable) |
| Ruler | 1.0 | 1.0 | 0.5 |

Per pop per 30 days. The **living standard** policy scales goods and luxuries (Austere 0.6× and no luxuries, Standard 1×, Comfortable 1.4× with specialists gaining 0.25 luxuries) and trades consumption for happiness (−10 / 0 / +10). Food never scales.

**Price elasticity** (`07 D24`). Demand is `basket × livingStandard × priceFactor_c`, with `priceFactor_c = clamp(1 − elasticity_c × (price_c / basePrice_c − 1), 0.5, 1.25)`. Elasticity: food 0, goods 0.5, luxuries 1.0, everything else 0. Pops buy less of what is expensive before it runs short; the scaling costs happiness as `priceStrain`.

### Happiness and stability

```
ethicsMatch     = 5 × Σ_{p ∈ empire poles} points_p × (cohort[p] − cohort[opposite(p)])     // −15..+15; +10.5 at game start
happinessTarget = 50
          + livingStandardMod
          + ethicsMatch
          + amenitiesMod           // clamp((amenities − 0.5 × pops) × 2, −20, +10)
          − shortagePenalty        // food 25×(1−foodSat); goods 20×(1−goodsSat); luxuries 10×(1−luxSat) for strata with luxuries
          − habitabilityPenalty    // (80 − hab)/4 when hab < 80
          − unemploymentPenalty    // 10 if the cohort has ≥ 0.5 unemployed pops
          − crowdingPenalty        // 15 × max(0, crowding − 1)
          − priceStrain            // Σ over goods and luxuries of 5 × max(0, 1 − priceFactor_c); at most 5
          − taxStrain              // 20 × max(0, taxRate − fairTax)
          + prosperity             // min(5, wealth / (20 × pops))
          + eventMods + party approval
happiness += (happinessTarget − happiness) × 0.1 per day
stabilityTarget = weighted mean happiness (ruler ×2, specialist ×1.5, worker ×1) + fortressBonus + modifiers − 20 × piracyOfSystem
stability += (stabilityTarget − stability) × 0.1 per day
```

`foodSat`, `goodsSat` and `luxSat` are 30-day rolling means of daily satisfaction.

| Stability | Effect |
|---|---|
| < 10 | revolt risk |
| < 25 | −25% output; the unrest situation starts |
| > 60 | +5% output |
| > 80 | +10% output |

As built in M11, an unrest revolt costs its colony happiness and stability, not empire influence (`07 D86`). FC1's party approval moves members' happiness by ± (approval − 50) / 5 (`01 §9.4`).

### Migration (`07 D24`, `D26`)

```
border(s, d) = 1     same empire, or a migration treaty (1.0)
             = 0.5   d's immigration is Open
             = 0.25  d's immigration is Selective and the cohort's skill ≥ 0.6 or its species already lives on d; else 0
             = 0     d is Closed, s's emigration is Closed, or s and d are at war
standardPull = clamp(0.5 + 0.5 × livingStandard_d / livingStandard_s, 0.5, 1.5)
attract(d, stratum) = freeJobs_stratum(d) × (0.5 + hab_d/200) × (1 + happiness_d/100) × clamp(1.5 − crowding_d, 0, 1)
                      × border(s, d) × standardPull / (1 + cells(s, d) / 2)²
leavers per 30 days = (0.002 × count + 0.02 × (unemployed + (happiness < 30 ? count : 0))) × (displaced ? 5 : 1)
```

- Every colony recomputes, on its 30-day governor timer, the three most attractive destinations per stratum among the colonies of any empire within 16 cells across open hyperspace, and caches them.
- A cohort whose best cached destination beats its own colony's `attract` by 1.2× sends its leavers in one packet to the best of them, and none while the packet would be under 0.01 pops (`minPacket`, `07 D242`).
- **Displaced** cohorts (colony starving, under bombardment, conquered within 5 years, or stability < 20) leave at ×5, move at a threshold of 1.0, and read the destination's refugee policy (Accept 1, Refuse 0) instead of its immigration policy.
- Policies: immigration Open / Selective / Closed (default Selective), emigration Free / Closed, refugees Accept / Refuse, each with a 1-year lock. Sedentary halves `leavers`.
- `Resettle { from, to, cohort, count }` moves pops between own colonies for 2 influence per pop, same day, the movers entering at −10 happiness. It is allowed into a halted colony.
- Foreign arrivals keep species, ethics and skill; a species new to the colony creates its cohort. Travel is abstract (same-day arrival). At game start no colony has a free job, so nothing moves until the governor builds one.

### Skill (`07 D27`)

```
skill += (1 − skill) × 0.01 × employedShare × pop.skill_gain     // per 30 days; half the gap closes in ≈ 6 years unmodified
skill −= skill × 0.02 × unemployedShare × pop.skill_decay          // per 30 days
job output × (1 + (0.25 + Σ pop.skill_output) × (skill − 0.5))     // 0.875 at 0, 1.0 at 0.5, 1.125 at 1, unmodified
```

| Source | Effect |
|---|---|
| Public Education (society tier 1) | `pop.skill_gain` +25% |
| Vocational Training (tier 2) | `pop.skill_output` +0.10 |
| Lifelong Learning (tier 3) | `pop.skill_decay` −50% |
| Neural Interfaces (tier 3) | `pop.skill_gain` +50% |
| Education policy | Minimal ×0.75 (free) / Standard (free) / Generous ×1.5 at 0.2 credits per pop per 30 days; 1-year lock |
| School's Teacher job | planet `pop.skill_gain` +25%, +1 amenity |
| Meritocracy civic | `pop.skill_gain` +15%, `pop.skill_output` +0.05 |
| Governor trait Educator | planet `pop.skill_gain` +25% |
| Prosperity perk Guilds | a skill modifier (`01 §6.8`) |
| 1.0: Athenaeum; Synthetic ascension | empire `pop.skill_gain` +25%; synthetic cohorts start at 0.75 |

Whatever the modifiers, the factor is 1.0 at skill 0.5, so the opening is unchanged.

### Taxation, private wealth and vitality (`07 D28`)

```
G       = 4 × tradeValue produced on the colony        // clerks 8, merchants 16 per pop per 30 days
I       = 1.2 worker, 2.4 specialist, 8.0 ruler         // gross income per pop per 30 days
taxRate ∈ { Low 0.10, Standard 0.25, High 0.40, Punitive 0.60 }      // empire policy, 1-year lock
ledger  += taxRate × (G + Σ I) × collection             // collection = 1 − 0.5 × maxPiracyOnSpoke
wealth  += 0.1 × (1 − taxRate) × (G + Σ I)              // per colony
wealth  −= 0.01 × wealth per 30 days                     // depreciation
fairTax  = 0.25 + Σ tax.tolerance
vitality target = clamp(1 + (fairTax − taxRate), 0.5, 1.2); vitality += (target − vitality) × 0.02 per 30 days   // half-life ≈ 3 years
every job output × vitality; I × vitality
```

- Standard reproduces the opening ledger exactly: the homeworld's `0.25 × (80 + 77.2) = 39.3` is trade value 20 plus pop tax 19.3.
- **Commerce policy** sets what the state's `G` share becomes: Credits (default), Mercantile (0.75 credits + 0.25 unity), Innovation (0.75 credits + 0.25 research). The `I` share is always credits.
- **Private wealth** is spent on private construction on zoned hexes ([Colonies and surfaces](Design-Colonies-and-surfaces#placement-adjacency-and-service-07-d29)), paying each day's draw and any credits in the bill. The **Levy** edict seizes 50% of every colony's wealth for 100 influence and −10 happiness empire-wide for a year. Conquest destroys half of a colony's wealth.
- **Tolerance** (`tax.tolerance`): Fiscal Administration (society tier 1, +0.05), Automated Audit (tier 3, +0.05), the Corporate Charter civic (+0.05, trade value +5%), the Prosperity perk Fair Levies (+0.05). With all four, Punitive settles at vitality 0.85 instead of 0.65.
- Worked cases: Punitive from day 1 gives 2.4× revenue at first (91 against 38 per 30 days), vitality 0.82 after 3 years and 0.68 after 10. Low from day 1 gives revenue 15, vitality 1.15 after 10 years and private construction at 2.3× the Standard pace.
- **Wealth classes** (FC1, DESIGN 2026-11-18): a colony's private wealth is held 5 / 25 / 70% by workers, specialists and rulers, at a 5% return a month against wages of 2 / 4 / 6 a pop; this sorts cohorts into owners or wage-earners for party politics.

### Civil upkeep per pop

`07 D273` (built 2026-11-21, every empire): each day the ledger charges pops × u(n) ÷ 30 credits, with u(n) = `popUpkeep` × max(0, n − `colonyFree`) × `empire.popUpkeep`; `popUpkeep` is 0.01 credits a pop a month, `colonyFree` 10, and n the planetary colony count. It rises with colonies, not pops, so tall is cheaper than wide. It is credits only, booked on the sprawl expense row and not surcharged by sprawl.

| n (planetary) | 12 | 15 | 20 | 22 | 30 | 36 |
|---|---|---|---|---|---|---|
| u(n), credits a pop a month | 0.02 | 0.05 | 0.10 | 0.12 | 0.20 | 0.26 |

The same decision gates founding on pops ([Colonies and surfaces](Design-Colonies-and-surfaces#founding-by-colony-ship)). Mitigations, multiplicative and clamped at 0.25:

| Source | Entry |
|---|---|
| `bureaucratic_protocols` (tech t1) | popUpkeep −0.10 |
| `planetary_bureaucracy` (tech t2) | colonyPops −0.10, popUpkeep −0.10 |
| `galactic_administration` (tech t3) | colonyPops −0.15, popUpkeep −0.15 |
| Ringworld section 3 | colonyPops −0.15 |
| Matrioshka Brain stage 3 | popUpkeep −0.15 |
| Ruler trait `centralizer` | colonyPops −0.10, popUpkeep −0.10 |
| Persona `planning` ranks 1/2/3 | colonyPops −0.05 / −0.10 / −0.15 |
| Persona `bureaucracy` ranks 1/2/3 | popUpkeep −0.05 / −0.10 / −0.15 |

The AI takes the same rules with no handicap; only the persona's skills are the player's (DESIGN, 2026-11-21).

### Ethics on cohorts

Cohorts carry an 8-vector, two poles per axis, each axis summing to 1, over four axes: Collectivist–Individualist, Militarist–Pacifist, Materialist–Spiritualist, Xenophile–Xenophobe. In MVP the vector is static and read only for `ethicsMatch`. FC1's monthly politics day drifts each cohort toward the material bloc of the jobs it works (Mining, Logistics, Researchers, Bureaucracy, Soldiers), its unemployed share back to unaffiliated; that is bloc membership, not ethics drift (`01 §9.4`; details on [Factions and characters](Design-Factions-and-characters)).

**Ethics drift (1.0, designed).** Each cohort's vector will move 1% per 30 days toward a target from lived conditions, plus +2% per 30 days toward the empire's ethics and faction propaganda:

| Condition | Pushes toward |
|---|---|
| Empire at war in the last 12 months | Militarist |
| 10+ years of peace | Pacifist |
| ≥ 25% of planet pops other species, happiness ≥ 50 | Xenophile |
| Foreign empire occupied or raided the planet | Xenophobe |
| Research output per pop above empire median | Materialist |
| Low stability or recent crisis | Spiritualist |
| Trade value per pop high, living standard Comfortable | Individualist |
| Austere living standard, rationing or shortages | Collectivist |

The owner chose Stellaris's eight ethics for 1.0 (`07 D230`), so Collectivist–Individualist will become Authoritarian–Egalitarian in CV1, and `docs/19 §3.2` proposes moving the economic rows of this table out of ethics and into the blocs.

### RTS and TB

Every rule above runs inside the daily tick (or its 5-day stagger) and every player lever (policies, Resettle, HaltGrowth, the Levy) is a Command applied at a tick boundary, so both modes behave alike; TB resolves the same days over a turn.

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Pop model | Fractional cohorts | 2026-11-01 (confirmed 2026-11-13) | `07 D14` |
| Beyond Stellaris | Births and deaths as two flows, housing as a soft ceiling, price-elastic baskets, skill, gravity migration; wages and age bins deferred to 1.0 | 2026-11-02 | `07 D24` |
| Cross-empire migration | Border factor from immigration, emigration and refugee policies; living-standard pull; displaced cohorts; Resettle | 2026-11-02 | `07 D26` |
| Skill modifiers | Three keys, four techs, Education policy, School and Teacher, Meritocracy, a perk, 1.0 hooks | 2026-11-02 | `07 D27` |
| Taxation | Tax on gross commerce and incomes, commerce policy, private wealth, vitality, `taxStrain`, Levy | 2026-11-02 | `07 D28` |
| M4b calls | Wealth accrues per colony; numbers corrected to the measured run (settling at ≈ 1.04 × housing) | 2026-11-03 | `07 D33` |
| Revolt cost | An unrest revolt costs local happiness and stability, not empire influence | 2026-11-14 | `07 D86` |
| Eight ethics | Stellaris's eight ethics, replacing the four-axis model, in 1.0 | 2026-11-16 | `07 D230` |
| HP1 stagger | Pop growth and happiness per colony every 5 days at 5 days' rates; jobs stay daily (replaces daily growth and happiness) | 2026-11-17 | `07 D175` |
| Migrant packets | One packet to the best destination, minimum 0.01 pops (replaces a daily three-way split) | 2026-11-18 | `07 D242` |
| Factions on cohorts | Parties from ethic poles plus material blocs and wealth classes (FC1) | 2026-11-18 | `07 D241`, `D214` |
| Halt growth | A command for player and AI; AI food-halt parameters | 2026-11-19 | `07 D264` |
| Job order | Producers fill ahead of their consumers | 2026-11-20 | `07 D266` |
| Empire size paced by pops | Per-pop civil upkeep rising past ten planetary colonies, with mitigations; same rules for the AI | 2026-11-21 | `07 D273` |

## Not yet built or still open

- **Ethics drift** (M15's remainder, 1.0) and **Stellaris's eight ethics** with the economic rows moved to blocs (CV1, 1.0).
- **Wages** (jobs paying credits per pop, floating with vacancies) and **age bins** per cohort, deferred to 1.0 (`07 D24`).
- Slave and robot strata, migration treaties, and migration routed through the inbound ring instead of same-day arrival (1.0).
- A per-colony tax override (1.0).
- 1.0 skill hooks: the Athenaeum and Synthetic ascension (the ascension paths left M19 for a later milestone, `07 D283`).

## Discrepancies noted

- **Update cadence.** `01 §6.2`, `§6.5`, `§6.6` and `§6.8` describe births, deaths, happiness, stability, migration and skill as daily ("daily, at 1/30 of these rates"; "happiness += ... each day"); `07 D175` and `00-status` (HP1, merged 2026-11-17) say "Pop growth and happiness run per colony every 5 days; jobs stay daily". The later decision is taken as current; `01 §6` has not been updated.
- **Migrant split.** Within `01 §6.6`, the `leavers` line's comment says "per cohort, split across the cached destinations by attract", while the prose below says a cohort "sends its leavers in one packet to the best of them" (`07 D242`). The prose and `07 D242` are taken as current.
- **Faction term.** `01 §6.5` lists "eventMods (+ factionMod, 1.0)" as a 1.0 term, while `01 §9.4` says "FC1 replaced this sketch" with members' happiness changing by ± (approval − 50) / 5, and `00-status` records FC1 merged into `main` on 2026-11-19.

## Sources

- 01 §6 Pops, strata, jobs: §6.2, §6.3, §6.4, §6.5, §6.6, §6.8
- 01 §7.9 Taxation, private wealth and vitality, §7.3 Jobs
- 01 §9.3 Ethics and drift, §9.4 Factions, §9.5 Government, civics, policies, edicts
- 07 decisions: `D14`, `D24`, `D26`, `D27`, `D28`, `D33`, `D86`, `D175`, `D214`, `D230`, `D241`, `D242`, `D264`, `D266`, `D273`
- 09 §1, §2, §3 (rationale for D26–D28)
- 17 §0 The owner's calls
- 19 §3 Macro layer
- 05 milestones (M4b, M15), 00 status, DESIGN tracker

## Related pages

- [Overview](Overview)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Economy and resources](Design-Economy-and-resources)
- [Trade and logistics](Design-Trade-and-logistics)
- [Factions and characters](Design-Factions-and-characters)
- [Civilisations and species](Design-Civilisations-and-species)
- [Progression](Design-Progression)
- [Events and crises](Design-Events-and-crises)
- [Starbases](Design-Starbases)
