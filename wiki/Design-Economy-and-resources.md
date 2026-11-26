# Economy and resources

**Status:** built, as of 2026-11-23. The resource model shipped in E1 (merged 2026-11-14, save v20); the governor and AI economic governance in M9, the economy balance pass and the later balance rulings (`07 D100`–`D109`, `D162`, `D237`, `D264`, `D273`, `D274`).

The economy is the game's spine (`SPEC.md` pillar 1). Every colony and starbase is a market holding physical commodities with stock, production, demand and a price; goods are always somewhere, never "in the empire". On top of the markets sits an empire ledger of non-physical numbers (credits, influence, unity, research, administrative capacity). Seventeen commodities form a production chain from raw ore, gas, food and energy up to components, machine tools, charged cells and four fuel tiers, and energy is itself a physical commodity that backs the currency. The player sees alloys on one world at a given price, a shipment arriving in three days, and a shortage stalling a shipyard elsewhere; the governor and the AI read the same markets to decide what to build.

## How it works

### Two layers: the ledger and the markets

| Layer | Holds | Where | How it moves |
|---|---|---|---|
| Ledger | credits, influence, unity, research (physics, society, engineering), administrative capacity | a number on the empire | income and upkeep flow daily at 1/30 of the stated per-30-day rates; the UI shows 30-day rolling sums |
| Markets | physical commodities: stock, production, demand, price | per colony and per starbase colony (each starbase is a small colony with its own market, `07 D39`) | by production, consumption, construction draws and shipments; goods in transit are somewhere on a route |

Only commodities are physical; the ledger stays an empire number (`07 D39`, the spec session's call inside the owner's redesign). Content rates are written per 30 days and every market resolves every tick at 1/30 of them; only AI planning is periodic (`01 §7.1`, `CLAUDE.md`).

### The commodities

Seventeen commodities, within the registry cap of 32 (`docs/14 §2`, `07 D73`, built in E1). The first ten keep their save indices (crystal became rare ore, parts became components, machinery became machine tools; fuel stays as refined fuel) and the seven new ones are appended (`07 D57`).

| id | Name | Tier | Base price | Made by |
|---|---|---|---|---|
| food | Food | raw | 1.0 | Farmer |
| ore | Minerals | raw | 1.0 | Miner, mining station |
| gas | Volatiles | raw | 2.0 | Gas Extractor, gas station |
| rare_ore | Rare ore | raw | 3.0 | Rare Ore Miner, rare-ore station |
| alloy | Alloys | mfg | 4.0 | Metallurgist |
| goods | Consumer Goods | mfg | 2.0 | Artisan |
| components | Components | mfg | 8.0 | Fabricator |
| machine_tools | Machine tools | mfg | 12.0 | Toolmaker |
| luxury | Luxuries | mfg | 6.0 | Luxury Artisan |
| fuel | Refined fuel | mfg | 3.0 | Fuel Refiner |
| rare_metals | Rare metals | mfg | 6.0 | Metal Refiner (from rare ore); Smelter (from ore) |
| energy | Energy | raw | 1.0 | Technician, energy station |
| batteries | Batteries (empty) | mfg | 5.0 | Battery Maker |
| cells | Charged cells | mfg | 15.0 | any market with spare energy and empties (1 battery + 10 energy) |
| crude_fuel | Crude fuel | mfg | 2.0 | Depot Worker |
| exotic_matter | Exotic matter | raw | 10.0 | deep-space fields |
| exotic_fuel | Exotic fuel | mfg | 12.0 | Exotic Refiner (Vacuum Power) |

Uses (`01 §7.2`): alloys go into districts, buildings and ships; components into advanced buildings, ship systems and starbase sizes; machine tools into megastructures, terraforming and advanced buildings; the four fuels into fleets; luxuries to rulers and entertainers; goods to pops, research and bureaucracy.

### The chain and what rare metals mean

The owner's direction for E1 (`07 D73`, 2026-11-13) was higher-level commodities built on commodity supply: alloys from minerals, rare metals and energy; machine tools from alloys; energy moved in batteries made from alloys and rare metals. Crystals merged into rare metals (option A).

The metals-chain retune (`07 D162`, the owner's call of 2026-11-16) changed what rare metals mean: they are the **higher tier's input**. Components, batteries and luxuries bind on them, while alloys take only a pinch (0.2 rare metals per 3 alloys), so alloys are bound by ore and energy. A Metal Refinery makes rare metals from rare ore one for one; where the empire makes no rare ore in the colony's system, a Smelter makes them from ore at two for one. The governor and the AI build the Smelter there instead of a Refinery that would starve (`governor.fixes` entries can name `needs`). The metals producers (Foundry, Metal Refinery, Smelter) are billed partly in ore (120 ore plus 40, 40 and 30 alloys), so they do not wait behind the alloys they unblock.

### Jobs (output per pop per 30 days, applied daily at 1/30)

The current numbers, from `01 §7.3` with the later changes of `07 D101` and `D162` (the content file `data/content/jobs.json` carries the same values):

| Job | Stratum | Inputs | Outputs |
|---|---|---|---|
| Farmer | worker | — | 7 food |
| Miner | worker | — | 5 ore (`07 D101`; was 4) |
| Technician | worker | — | 18 energy |
| Gas Extractor | worker | — | 2 gas |
| Rare Ore Miner | worker | — | 1 rare ore |
| Clerk | worker | — | 2 trade value, +1 amenity |
| Depot Worker | worker | 3 gas | 3 crude fuel |
| Metal Refiner | specialist | 1.5 rare ore | 1.5 rare metals |
| Smelter | specialist | 3 ore | 1.5 rare metals |
| Metallurgist | specialist | 2.4 ore, 0.2 rare metals, 0.75 energy | 3 alloys |
| Artisan | specialist | 4 ore, 0.9 energy (`07 D101`; was 6 ore) | 6 goods |
| Fabricator | specialist | 2 alloys, 1 rare metals | 2 components |
| Luxury Artisan | specialist | 2 goods, 0.4 rare metals | 2 luxuries |
| Fuel Refiner | specialist | 2 gas, 1 energy | 2 refined fuel |
| Toolmaker | specialist | 2 alloys, 2 energy | 2 machine tools |
| Battery Maker | specialist | 1 alloy, 1 rare metals | 2 batteries |
| Exotic Refiner | specialist | 1 exotic matter, 2 energy | 2 exotic fuel |
| Researcher | specialist | 0.5 goods | 6 research |
| Bureaucrat | specialist | 0.5 goods | 3 unity, +2 admin cap |
| Entertainer | specialist | 0.5 luxuries | +4 amenities, +1 unity |
| Merchant | specialist | — | 4 trade value, +10 freight capacity |
| Administrator | ruler | — | 3 unity, +0.5 influence, +3 stability |

Soldiers, medics and teachers are listed in `01 §7.3` and belong to [Pops and society](Design-Pops-and-society). Outputs scale with designation (+15%), stability, skill, adjacency and service, vitality, species traits, tech modifiers, buildings and edicts. Inputs are drawn from the local market each day; if an input is short, that day's output scales by the satisfaction ratio.

Jobs fill in content order within a stratum, so producers are listed ahead of their consumers: the metal refiner and the smelter come before the jobs that use rare metals, and the battery maker before the service jobs (`07 D266`, 2026-11-20). `datacheck` refuses a job listed after a job of its own stratum that uses its output (`01 §6.3`).

### Energy as a physical commodity

- **Production.** Technicians (18 energy each), energy stations (2 energy per deposit unit to their market) and Dyson structures make energy; technicians and energy stations no longer pay credits (`07 D57`).
- **Within a system** energy is beamed between markets at a **10% loss**, with no batteries.
- **Between systems** it travels only as **charged cells**. Energy never enters the trade network's source lists or the push to a hub, so no trade-flow row carries it (`07 D57`).
- **Charging and discharging** run in the markets pass. A market below its energy target (90 days of demand) discharges cells: each yields 10 energy and an empty battery, less 2% wear per cycle. A market above its target charges its empties at 10 energy each. A market short of energy asks for cells, one with a surplus asks for empties, and the trade network carries both, like any commodity.
- **Cells are a commodity in their own right:** bought, sold, stockpiled, shipped and burned as battery fuel (the owner's answer, `07 D73`).
- **Storage.** Energy past a market's storage is lost (no overflow sale), except at the treasury, which banks it.

### The energy-backed currency

The owner's call (`07 D73`, 2026-11-13): energy is a market commodity that backs the currency; the more energy, the more credits, with diminishing returns, and spending credits on resources makes resources dearer. The numbers are the spec session's (`docs/14 §3`), as built in E1 (`01 §7.4`, `07 D57`):

- **The treasury** is the capital's market. Each day it banks, as backing:
  - the **Energy backing** policy's share of its energy production less its demand: Standard ½ (the default), High ¾, Low ¼;
  - every charged cell it holds, at 10 energy each (it keeps the empties);
  - whatever energy its storage cannot hold.
- The treasury asks for cells for its empire's energy surplus in other systems and charges none itself, so energy made in other systems still backs the currency.
- **Issuance.** On the last day of the empire's staggered month it issues `2 × B^0.8` credits for the `B` energy banked since the last issuance. Doubling the backing gives about 1.74× the credits (`2^0.8`), so Dyson-scale energy keeps paying at a worsening rate.
- **Price index.** It sets the empire's price index `π = max(1, (S / 1.5B)^0.5)`, where `S` is the credits spent on market purchases over 30 days. Its purchases pay `price × π`. Every purchase estimate by the governor and the AI includes `π`.
- **Cross-empire settlement.** A cross-empire import pays at the ratio of the two empires' indices (`π_importer / π_exporter`).
- **Credits keep their other sources:** taxes, trade value and market sales.

In the standard opening the capital backs 15.45 energy per 30 days (half its +30.9 surplus) and issues `2 × 15.45^0.8` = 17.9 credits on its first issuance (tick 29) (`01 §3`).

### Fuels

Four tiers in MVP (`07 D73` option A; numbers `docs/14 §4`, `01 §10.2`). Each fleet has a preferred fuel and falls back down the tiers.

| Tier | Source | Range (units per refined unit of range) | Speed | Readiness | Combat |
|---|---|---|---|---|---|
| Crude | crude fuel | 1.5 | −15% | recovery in supply −50% | engines −10% |
| Refined | refined fuel (baseline) | 1 | — | — | — |
| Battery | charged cells | 4 | +15% | — | capacitor and shield regeneration +20% |
| Exotic (tier 3) | exotic fuel | 0.5 | +10% | — | damage, engines and regeneration +10% |

The tank holds units by tier and burns the preferred tier first, then down, then up; the effects follow the tier burning. Docked, a fleet refuels with its preferred tier from the market, falling back down. Fuel is a bottleneck by design: a large mid- or late-game navy needs either a strong refinery chain (volatiles plus energy) or a large energy surplus charged into cells (the owner's design note, `07 D73`). Fleet movement and supply are on [Ships and fleets](Design-Ships-and-fleets).

### Daily market resolution

Every market resolves every tick in this order (`01 §7.5`, `02 §3.4`): **production → trade dispatch → arrivals and allocation → prices**. Construction draws its materials in the separate daily `construction` system. Daily quantities are the per-30-day rates divided by 30.

1. **Production.** Pass 1 adds raw-job output and station deliveries to stock (available today). Pass 2 records each manufacturing job's input requests and potential output; nothing is drawn yet.
2. **Demand** per commodity, in units per day, is the sum of:
   - pops: cohorts × the stratum's basket × living standard × price factor;
   - jobs: the manufacturing input requests;
   - builds: the remaining cost of in-progress and queued constructions ÷ 90 (counts toward targets, prices and caps; never allocated here);
   - fleets: the fuel fleets drew here over the last 30 days ÷ 30.
3. **Trade dispatch** (see [Trade and logistics](Design-Trade-and-logistics)): shipments leave source stock today and are placed in the destination's inbound ring.
4. **Arrivals and allocation.** Available = stock + today's inbound slot. Consumers are served in priority order, each getting min(request, remaining): (1) pops' food; (2) manufacturing job inputs, by job priority then job id, each job's satisfaction being the minimum over its inputs; (3) pops' goods, then luxuries. Realised manufacturing output enters stock, usable tomorrow.
5. **Spoilage and storage.** Food stock is multiplied by 0.99829 a day (5% per 30 days). Stock is capped at the market's capacity (1,000 plus depots; a Depot adds 500 per commodity); overflow is sold at 50% of the price (energy overflow is lost instead).
6. **Satisfaction** is allocated ÷ requested; the satisfaction used by shortage effects is its 30-day rolling mean.

**Conservation** is a property test per commodity per tick: Δstock + Δinbound = production + purchases − allocated − dispatched + arrived − construction draw − spoilage − overflow − sales − pirate losses. Fleet cargo and the spot market are external source and sink terms (`01 §7.5`, `§10.6`).

### Prices

After allocation (`01 §7.5`; content `economy.json` `market`):

```
inflow    = today's production (pass 1 + realised pass 2) + today's arrivals
flowRatio = (demand + ε) / (inflow + ε)                         // ε = 2/30 units a day
target    = 90 × demand                                          // 90 days of demand
stockAdj  = clamp((target − stock) / max(target, 1/30), −0.5, 0.5)
rawPrice  = basePrice × clamp(sqrt(flowRatio) × (1 + 0.5 × stockAdj), 0.25, 4.0)
price     = 0.95 × price + 0.05 × rawPrice                       // ≈ 20-day smoothing
```

Behaviour: a non-perishable at steady state (inflow = demand, stock at target) prices at base. Goods nobody demands price at 0.75 × base while stock is held, lower (to the 0.25 floor) if they keep flowing in, and are exported as surplus. A small unmet demand prices at about 0.9 × base while a stock lasts and about 1.5 × base once it is gone. Raw → manufactured works within a day on one planet; manufactured → manufactured lags a day; any cross-market input lags its transit time.

For energy, the price reads only the market's own use against production plus arrivals, so a producer exporting its surplus as cells still prices as if the surplus were unused; with a full store, the price falls under 0.5 × base once inflow exceeds 2.25× the market's own demand (`docs/research/2026-11-20-energy-price-and-rare-metals.md`). Counting the charging draw as demand (option E1 of that note) was dropped on 2026-11-20, since it would lift the median price on paper but move no energy (DESIGN change log).

The `01 §18` price sanity target: per commodity, the median of price ÷ base over market-days with demand lies in [0.5, 2.0], shortage days are under 15% of market-days with demand, and the empire freight utilisation median is under 90%.

### Shortage effects

Continuous while the 30-day rolling satisfaction is under 1 (`01 §7.6`):

| Commodity short | Effect |
|---|---|
| food | fertility × sat; happiness −25 × (1 − sat); starvation (×5 mortality) after 90 days at sat < 0.5 |
| goods | happiness −20 × (1 − sat) for all strata; research and unity jobs output × sat |
| luxury | happiness −10 × (1 − sat) for strata with luxuries in their basket; entertainer output × sat |
| raw inputs (ore, gas, rare ore) | downstream job output × sat (that day) |
| alloys, components | construction and shipbuilding progress × sat (daily draw) |
| fuel | fleets supplied from this market: readiness −10 × (1 − sat) per 30 days |

A starbase short of an upkeep commodity runs its yard, repair and upgrades at half rate (`01 §10.4`).

### Buying, selling and stock policies

Any market trades credits for goods with an off-map spot market at the local price: buy at `price × 1.1` (times the empire's price index), capped at `max(0.2 × stock, 50)` units per commodity per 30 days; sell at `price × 0.9`, uncapped (`01 §7.7`). Per market and commodity, standing orders set the stock target (`auto`, `stockpile N`, `export surplus above N`, `forbid import`, `forbid export`, and since M17d `ban`). Both are described in full, with the Galactic Exchange, on [Trade and logistics](Design-Trade-and-logistics).

### The ledger

**Credits** (`01 §7.4`):

| In | Out |
|---|---|
| energy issuance (above) | building, district, starbase, station, ship and leader upkeep |
| taxation: `taxRate × (G + Σ I)` (`01 §7.9`); at Standard exactly the trade value plus a pop tax of 0.3 / 0.6 / 2.0 per worker / specialist / ruler per 30 days | edict costs and subsidies |
| market sales | market purchases (at `price × π`) and cross-empire imports |
| | civil upkeep (`07 D273`, below) |

- **Negative credits** are allowed. While the balance is negative, all job outputs fall 25%, fleet readiness falls 5 per 30 days, and after 180 consecutive days the Recession situation starts. The owner kept the −25% rule (DESIGN change log, 2026-11-17).
- **Taxation, private wealth and vitality** (`07 D28`): tax rates Low 0.10, Standard 0.25, High 0.40, Punitive 0.60 with a one-year lock; vitality drifts toward `clamp(1 + (fairTax − taxRate), 0.5, 1.2)` with a half-life of about 3 years and multiplies every job output. Gross commerce `G = 4 × trade value` is collected at the hub (see [Trade and logistics](Design-Trade-and-logistics)). The full rules are on [Pops and society](Design-Pops-and-society).
- **Influence:** +3 base, +0.5 per administrator, +0.5 per rivalry (max 2); spent on claims, edicts, reforms, diplomacy and hub designation; hard cap 1,000.
- **Unity:** from bureaucrats, administrators, entertainers and monuments; spent on traditions and ascension perks.
- **Research:** per field, from researchers, the capital bonus, research stations and events.
- **Administrative capacity:** base 30 plus bureaucrats, offices and tech. `sprawl = colonies × 2 + systems × 0.5 + pops × 0.1 + districts × 0.2`. Each 1% of sprawl over the cap adds 1% to tech and tradition costs and 0.5% to all upkeep.
- **Civil upkeep** (`07 D273`, the owner's call of 2026-11-21, built): each day an empire pays `pops × u(n) ÷ 30` credits, where `u(n) = popUpkeep × max(0, n − colonyFree) × empire.popUpkeep`, with `popUpkeep` 0.01 credits a pop a month, `colonyFree` 10, and `n` the planetary colony count (starbases aside). It is booked on the sprawl row and not surcharged by sprawl. Wide costs more than tall. The founding gate that goes with it is on [Colonies and surfaces](Design-Colonies-and-surfaces); the statecraft techs, the Ringworld, the Matrioshka Brain, the `centralizer` ruler trait and the persona's administration skills reduce both.

| n (planetary colonies) | 12 | 15 | 20 | 22 | 30 | 36 |
|---|---|---|---|---|---|---|
| u(n): credits a pop a month | 0.02 | 0.05 | 0.10 | 0.12 | 0.20 | 0.26 |

### Construction: local sourcing, slots and the start rule

- **Local sourcing** (`07 D39`): every construction (a starbase's founding or upgrade, a ship in a yard, a station, a habitat) draws its daily bill only from the markets of its own system. The bill is posted as construction demand on the system's starbase market and the trade solver imports what the system lacks. An order needs an own starbase in the system. The rule is described with the rest of physical logistics on [Trade and logistics](Design-Trade-and-logistics).
- **Construction slots** per system: `max(3, 2 + local alloys per 30 days / 10)`, up to 6 (`07 D58`; content `governor.slots` = [2, 10, 6, 3]). Builds past the slots wait unstarted, draw nothing and post no demand; started builds keep their slots (`07 D55`).
- **The start rule** (`07 D100`): a build starts drawing only once its market holds 30 days of its daily draw of each bill commodity (`market.gatherDays` 30), or the whole bill if smaller. Until then it waits unstarted, posting demand so imports come; the UI shows "waiting for stock: Alloys 12/30 days". This applies to player builds too; there is no force-start.
- **Draw order.** Manufacturing holds back each market's next daily construction draw, so builds rank ahead of job inputs; builds draw in a daily rotation, and a build stalled on a missing commodity yields its slot (`07 D57`).
- **Bills carry credits** where content says so (`07 D102`): the mining district costs 10 alloys and 100 credits, since a mine billed in the ore it makes deadlocked ore-short markets.
- **Stuck builds are cancelled** (`07 D264`): a district, building, ship or upgrade that has started drawing and made no progress for over 360 days (`budget.cancelAfter`) is cancelled by the AI for its own builds, and by the governor for the builds it ordered. The slot and hex are freed and nothing drawn is refunded (no cancel refund exists). The player's own orders are never cancelled.

### Governance: the colony governor

The player's colony governor (`01 §17`) acts when a colony's state changes or once every 30 days per colony (staggered by colony id), never every tick.

- **Shortage projection.** It builds a fix when a shortage is projected within 360 days: `stock / max(dailyDemand − dailyProduction − imports30d, 0.0003) < 360`, never for a commodity that another market in the same system holds 90 days of (`07 D33`). The system's most populous own colony also counts its starbase's refined-fuel draw and stock (`07 D57`).
- **Fix order** (content `governor.fixes`): food, energy, goods, gas, refined fuel, alloys, rare ore, ore, components, luxuries, rare metals (Metal Refinery where rare ore is made in the system, else Smelter), machine tools, batteries, crude fuel. Energy's fix sits right after food, with a Power Plant (2 technicians) as the fallback (`07 D57`).
- **Back-off.** A fix that could not be ordered is not retried at that colony for 3 governor periods, or until a construction in its system finishes or the colony gains a slot or a surveyed hex (`07 D242`).
- **Slots and freight.** It keeps one building slot free (the fuel fix may take that spare slot, `07 D162`), and builds a Spaceport at the busiest market when empire freight utilisation exceeds 80%.
- **Markets automation.** All stock policies `auto`; sells overflow; buys to cover a shortage while credits stay above 3 months of upkeep.
- **The food buffer** (`07 D274`): on every colony's monthly governor timer, whether or not its governor is on, a market whose imports cannot land its food demand buys food on the spot market. Its rules are on [Trade and logistics](Design-Trade-and-logistics).

**The supply forecast and ordering rule** (`07 D55`, the owner's call of 2026-11-13: a supply-aware governor plus construction slots, with local sourcing intact). The forecast, computed only when a governor timer, an AI plan or a tooltip asks, adds per system and commodity over a 90-day horizon: stock (a starbase depot's at half), inbound shipments, production less non-construction demand, less every committed build's remaining bill, plus what an outside hub can deliver within the horizon. A build is ordered only if its system has a free slot and its bill fits the forecast. A producer of a missing input is judged on stock and inbound alone, and a starbase upgrade may count the empire's stock of commodities the empire produces. The player may order anything; the starbase panel shows the forecast with the slots and shortfalls. Further guards: the upkeep margin (a build only while the 30-day margin after its upkeep stays ≥ 0, unless its output value pays its upkeep back), fill-aware builds (only while idle pops plus 12 months' growth staff the new jobs), and the hub's projection of its spokes' unmet need.

### Governance: the AI economic model

The owner asked for the AI's economic root cause to be fixed rather than patched and chose option A (`07 D237`, 2026-11-17, built 2026-11-18): **one budget per AI empire, kept in the sim**, refreshed once per AI plan on the 30-day stagger and saved. The AI and the governor (for AI empires) read and debit it. The player's governor is unchanged.

- **What it holds:** a forecast of recurring income (spot sales excluded), committed upkeep (including builds and colonies in flight with their sprawl step), and a margin against a reserve of `reserveMonths` (3) × upkeep; the supply balance for food, energy, alloys, rare metals and goods, with growth and spoilage.
- **Pricing:** every commitment is priced by its marginal recurring cost (upkeep, sprawl step, the supply it draws, what it makes at expected staffing); a colony ship is priced as the colony it founds, including its civil-upkeep step (`07 D273`).
- **The priority ladder:**
  1. restore the reserve by selling surplus (never food, a short commodity, what its own builds wait for, or its spokes' 90 days);
  2. after `shedAfter` (3) plans in deficit, retire the building with the least output value per upkeep;
  3. fix supply at the best site in the empire (most idle pops, a slot, inputs by quantity);
  4. grow within the headroom (colonies, upgrades, military, pacts).
- **The Supply rung:** a commodity its markets make under `budget.critical` (0.5) of their demand is critical; the ladder may place a second producer for it in the same plan, at another colony, admitted on the empire's stock and inbound (`07 D237`, 2026-11-18).
- **Deficits** (`07 D264`, the owner's call of 2026-11-19): when a colony's or the empire's projected 30-day balance of a commodity is negative, the AI acts instead of waiting for the market. The ladder answers every forecast shortage, worst first, with up to `budget.deficitMax` (3) producers per commodity per plan, counting producers in flight; then each non-capital colony with a negative 30-day balance gets one local producer, skipped when another own market in the system makes a daily surplus that covers the gap (`07 D273`'s energy fix).
- **Halt growth** (`07 D264`): `Command::HaltGrowth` stops births and immigration at a colony. The player toggles it on the colony panel; the AI halts a colony whose food deficit has no producer placed this plan and whose stock covers under 90 days (`haltCoverDays`), lifting it on a positive food balance or when a food producer completes.
- **Expansion pace:** a colony ship is built only while the empire's planetary colonies plus ships in flight stay under `colonyBase` (8) + pops ÷ `popsPerColony` (40) (`07 D237`), and the founding gate of `07 D273` applies to every empire; the stricter binds.
- **Site supply** (`07 D274`, the owner's call of 2026-11-21: "Supply resilience should be a factor in colony formation"): each candidate site's food resilience R (the share of a new colony's food the empire can land there) multiplies its score by `1 − 0.5 × (1 − R)`, and a site with R < 1 and no farmland is vetoed (never leaving the AI with no legal site). The player sees the same figure on the colony-ship target and the Colonize button.
- **The old guards it replaced:** the colony-ship guard, `affordable`'s margin path for AI empires, the AI's producer and warship checks, the credits-freeze in `supply()` and the per-colony food fix for AI empires; the parked `07 D200`–`D202` patches were folded into the ladder.

AI commercial pacts, personalities and difficulty are on [AI and victory](Design-Ai-and-victory) and [Diplomacy and war](Design-Diplomacy-and-war); the pact trade rules are on [Trade and logistics](Design-Trade-and-logistics).

### The opening economy

The standard origin's opening ledger nets +8.2 credits on day 1 (+9.45 running, once the home system's self-sufficiency bonus takes 25% off the Station's upkeep), within E1's ±2 of +7 (`01 §3`, `07 D57`). The homeworld carries a Metal Refinery with 2 refiners from day 1; the opening stock adds rare metals 40, rare ore 20, energy 100 and refined fuel 50 (about 14 months). There is no day-1 refinery; the governor and AI build one before refined fuel falls under four months of demand (`07 D57`). The opening and the Sol start are on [Starts and exploration](Design-Starts-and-exploration).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Commodity count | 10 commodities (option A) at first; deferred to E1 by the owner and replaced by the 17 of `docs/14` | 2026-11-01; 2026-11-13 | `07 D4` |
| Ledger stays non-physical | Only commodities are physical; credits, influence, unity and research are empire numbers | 2026-11-08 | `07 D39` |
| Resource model | Energy backs the currency with diminishing returns; charged cells tradable as they are; crystals merge into rare metals; four fuel tiers with a preferred fuel and fallback. Replaced `01 §7.2`'s 10-commodity table, the energy credits of `§4.2` and `§7.9` and the single fuel of `§10.2` | 2026-11-13 | `07 D73`, `docs/14` |
| E1's numbers | Industry's energy draw halved from `docs/14`'s first proposal (alloys 0.25 per unit, goods 0.15); backing policy Standard ½ / High ¾ / Low ¼; the treasury banks cells and overflow and asks for cells from other systems; technicians 18 energy. Replaced `docs/14 §2`'s 0.5 and 0.3 | 2026-11-14 | `07 D57` |
| Supply-aware governor and slots | The owner's call: a supply forecast and construction slots with local sourcing intact; upkeep margin, fill-aware builds, the hub's projection | 2026-11-13 | `07 D55` |
| Construction slots | `max(3, 2 + local alloys / 10)` up to 6. Replaced `07 D55`'s `2 + floor(alloys / 20)` capped at 6 | 2026-11-14 | `07 D58` |
| The start rule | A build draws only once its market holds 30 days of its draw | 2026-11-14 | `07 D100` |
| Ore numbers | Miners 5 ore (was 4), metallurgists 2.4 ore (was 3.6), artisans 4 ore (was 6), in the owner's order of the supply try | 2026-11-14 | `07 D101` |
| Mines billed in alloys and credits | Mining district 10 alloys + 100 credits. Replaced 100 ore + 10 alloys | 2026-11-14 | `07 D102` |
| Building-materials priority | The owner's lever (construction draws before job draws at a source) was built, measured and then dropped by the owner; parked on `lever-park` | 2026-11-15 | `07 D108` |
| Metals-chain retune | Rare metals become the higher tier's input; metallurgists 0.2 rare metals per 3 alloys (was 0.6); the Smelter as fallback; metals producers billed partly in ore | 2026-11-16 | `07 D162` |
| AI economic model | One saved budget per AI empire with a priority ladder; replaced the separate guards and the parked debt patches | 2026-11-17 | `07 D237` |
| Rule-change bundle | No change to the sprawl cost; build-retry back-off; AI pacts only from year 25 (with `D236`) | 2026-11-18 | `07 D242` |
| Deficits, halt growth, stuck builds | The AI answers projected deficits; a halt-growth command; AI and governor cancel builds stuck over 360 days | 2026-11-19 | `07 D264` |
| Energy's price box | Counting the charging draw as demand dropped (moves no energy); the job-order fix (R1) cleared the box | 2026-11-20 | DESIGN change log, `07 D266` |
| Producers fill first | Rare-metal producers and the battery maker ahead of their consumers in the job order; `datacheck` enforces it | 2026-11-20 | `07 D266` |
| Empire size | A founding gate that steepens with colonies, and civil upkeep per pop rising with colony count; `colonyStep` shipped at 16. The owner declined a harsher sprawl cost (`D242`) | 2026-11-21 | `07 D273` |
| Supply resilience | Food resilience weighs AI site choice and vetoes farmless unreachable sites; a food buffer as the backstop for every empire | 2026-11-21 | `07 D274` |

## Not yet built or still open

- **Owner review still open:** the M5 implementer calls (`07 D34`) and the M8c calls (`07 D46`) stay on DESIGN's list of `07` call sets open for review.
- **Starsector-style extra commodities** (supplies, crew, marines, organics, contraband) were offered and are not yet scoped (DESIGN, open questions).
- **The stall box** ("constructions stuck a full year ≤ 15% of construction-years", carried from M9) is still unchecked in `05 E1`. After `07 D264`'s cancel rule and `D265`, the stuck share measured 0.2% mean (seeds 1–5 to year 150); the queue ("waiting over a year") box measured 20.2% mean, holding on two of five seeds.
- **Rare-metal ladder gates** (R2 of the 2026-11-20 research note: one free slot for a short commodity's producer) is held until rare metals are shown to stay binding (`07 D266`).
- **Perishable stock target** (a shorter `targetDays` for food) waits for a post-MVP economy pass (`07 D194`, `D195`).
- **Hub food forwarding** (hubs offering food above their reserve) was not built; the counter showed the import cap, not the source side, limits long-spoke food (`07 D194`).
- **A per-colony tax override** is listed for 1.0 (`01 §7.9`).
- **Difficulty extras:** Grand Admiral's free hub and full map are not built (`07 D55`).
- **The persona's AI cores** go to the tech tree and the synthetic-ascension scope, not to the economy (`07 D284` call 5).

## Discrepancies noted

- **Miner and artisan numbers in `01`.** `01 §7.3` gives "Miner | worker | — | 4 ore" and "Artisan | specialist | 6 ore, 0.9 energy | 6 goods"; `01 §7.2` gives goods as "1 ore + 0.15 energy" per unit; and `01 §3`'s opening line reads "homeworld 16 + moon 8 + rocky 8 + asteroid station 4 = 36; artisans 24 + metallurgists 7.2 = 31.2 → +4.8". `07 D101` (2026-11-14) says "Artisans draw 4 ore for their 6 goods (were 6)", "Miners make 5 ore (were 4)", and "The opening's ore line is now 44 made against 20.8 demanded". `data/content/jobs.json` has miner `"ore": 5` and artisan `"ore": 4`. D101 is later and matches content, so this page uses 5 and 4; the `01` text was not updated.
- **The resource model's status in the tracker.** `docs/DESIGN.md`'s pillars table lists "Resource model | ... | designed (E1)", while its own change log says "E1 merged (2026-11-14): 17 commodities, energy backing the currency, charged cells, four fuel tiers; save v20" and `05 E1`'s acceptance boxes are ticked. This page treats E1 as built.
- **Old commodity names in the shortage table.** `01 §7.6` lists "ore / gas / crystal" and "alloy / parts"; crystal became rare ore and parts became components in E1 (`07 D57`). This page uses the new names.
- **The governor's fix order.** `01 §17` gives "food → goods → gas (gas-vent district or station) → fuel (Refinery) → alloys → crystal and parts (crystal station, Workshop) → rare metals ... → luxuries (Luxury Works)", with no energy entry. `07 D57` says energy's fix comes "right after food", and `data/content/economy.json` `governor.fixes` lists food, energy, goods, gas, fuel, alloy, rare_ore, ore, components, luxury, rare_metals, machine_tools, batteries, crude_fuel (luxuries before rare metals). This page gives the content order.

## Sources

- SPEC.md
- 01 §3 Empires and species (the opening)
- 01 §7.1 Two layers
- 01 §7.2 Commodities
- 01 §7.3 Jobs
- 01 §7.4 Ledger flows
- 01 §7.5 Daily economic resolution
- 01 §7.6 Shortage effects
- 01 §7.7 Buying and selling
- 01 §7.9 Taxation, private wealth and vitality
- 01 §10.2 Movement and logistics (fuel)
- 01 §10.4 Starbases
- 01 §17 Automation defaults
- 01 §18 Default numbers and targets
- 05 Milestones (E1, M9, the 1.0 table)
- 07 Decisions: `D4`, `D28`, `D33`, `D39`, `D55`, `D57`, `D58`, `D73`, `D100`–`D108`, `D162`, `D194`, `D195`, `D237`, `D242`, `D264`, `D266`, `D273`, `D274`, `D284`
- 14 The resource model
- DESIGN.md (pillars, open questions, change log)
- Research: supply diagnosis (2026-11-16)
- Research: shortage diagnosis (2026-11-19)
- Research: shortage options (2026-11-19)
- Research: energy price and rare metals (2026-11-20)
- `data/content/jobs.json` and `data/content/economy.json` (content values)

## Related pages

- [Trade and logistics](Design-Trade-and-logistics)
- [Pops and society](Design-Pops-and-society)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Starbases](Design-Starbases)
- [Ships and fleets](Design-Ships-and-fleets)
- [Megastructures](Design-Megastructures)
- [Starts and exploration](Design-Starts-and-exploration)
- [AI and victory](Design-Ai-and-victory)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Progression](Design-Progression)
- [Architecture and performance](Design-Architecture-and-performance)
- [Overview](Overview)
