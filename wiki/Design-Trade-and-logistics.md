# Trade and logistics

**Status:** built, as of 2026-11-23. The network, dispatcher and piracy shipped in M5 and were moved onto the hyperspace grid in M8c; physical logistics in M8b; trade fleets, subsidies and embargo, the Galactic Exchange and smuggling in M17a–d (merged 2026-11-22 to 2026-11-23); gateway trade in M19a (merged 2026-11-23).

Goods move between markets only by shipment. Every empire runs a hub-and-spoke trade network over the hyperspace grid, and a daily dispatcher ships surplus to deficit along it, paying freight, transit days and losses to piracy, under each market's accessibility and import cap. Nothing teleports: a construction draws only from its own system, so a remote starbase upgrade waits for its alloys to arrive. On top of the abstract network sit commercial pacts with tariffs, physical trade convoys, subsidies and embargoes, a neutral Galactic Exchange, smuggling, and gateway shortcuts. Trade is drawn on the map, so the player sees where goods flow, where pirates feed and where a blockade has cut a route.

## How it works

### Nothing teleports: physical logistics

The owner's redesign of 2026-11-08 (`07 D39`, built in M8b): resources never transmit magically through the empire but physically go place to place; resource flows to bases are managed, with self-sufficiency rewarded. It replaced every "nearest own market by hops" draw.

- **Local sourcing.** Every construction (a starbase's founding or upgrade, a ship in a yard, a station, a habitat) draws its daily bill only from the markets of its own system. The bill is posted as construction demand on the system's starbase market, and the trade solver imports what the system lacks like any other demand: with freight, transit days and piracy exposure. In-system markets ship to the starbase market in 1 day. A stalled construction shows "waiting for alloys". An order is refused only where the system has no own starbase (`01 §10.4`).
- **The construction ship carries nothing:** its bill is demand at the site's starbase market (`07 D39`, the spec session's call).
- **Colony ships** can only be built in the same system as a starbase with enough local resources (`07 D39`).
- **Fuel and alloys reach a starbase only by shipment;** docked fleets repair in alloys and refuel from that market (`01 §10.2`).
- **Energy crosses systems only as charged cells** (`docs/14 §2`); see [Economy and resources](Design-Economy-and-resources).
- **Cargo is physical.** A raiding fleet with no cargo space takes nothing (`07 D39`); a deep outpost's output fills a 500-unit hold that a freighter must collect (`01 §2.1`).
- **Self-sufficiency** (`07 D39` option A, the owner's call): imports pay only the network's freight, transit time and piracy exposure, and a starbase supplied from its own system gains a bonus. Each starbase keeps a rolling 90-day count of its upkeep and construction commodities (alloys, parts, fuel, food, goods) that came from its own system against those imported. With local share `r`, it gains `25% × clamp((r − 0.5) / 0.25, 0, 1)` to build and upgrade speed, battery damage and hull, and the same share off its upkeep: nothing at half or less, the full bonus from 75% local (`01 §10.4`).

### The network: hubs, fields, spokes and legs

From `01 §8.1`, on the hyperspace grid of `07 D43` (there are no hyperlanes):

- **Hubs.** Every empire has at least one hub (the capital by default). A hub is designated for 50 influence and needs a starbase of size 2 or more in the system. The AI designates a second hub at a market past 30 route days, or past 15 once eight colonies lie outside the home system and only the capital is a hub, upgrading that starbase first if only its size blocks it (`07 D55`). The player gets a suggestion when a spoke exceeds 30 route days (`01 §17`).
- **Fields.** Each hub keeps a Dijkstra search of route days from its cell over the month's hyperspace grid at freighter speed 100, through its empire's corridors, stopped once its targets (its empire's market systems, its hubs and its pact partners' hubs) are settled. Each hub is recomputed on its own day of the month, chosen when it becomes a hub as the day with the fewest hubs (earliest on a tie) and saved (`07 D268`), so every field is at most a month old. A corridor built, drifted or broken, or an ownership change, recomputes the fields concerned (`07 D46`).
- **Spokes and hub pairs.** Each market links to its nearest hub by route days, walked back along that hub's field (a spoke). Hubs link to each other (hub pairs) along the same fields. Commercial pacts link each party's hubs to the other's. Spokes, hub pairs and pact routes are the network's **legs**; each keeps the piracy regions it crosses with its days in each, the systems it passes (for blockades) and its corridor share.
- **A shipment route** is source spoke → source hub → hub pair → destination hub → destination spoke. A shipment arrives route days + 1 after dispatch; the inbound ring holds 64 days (`07 D46`).
- **In-system shipments** take 1 day, cost `amount × 0.125` freight, cross no region and are never blockaded from outside. Markets in a system that holds a hub are 0 days from it. A shipment that leaves a system adds 2.5 freight-days for each in-system end. The longest route is 60 days; longer routes are split at the hub (`07 D46`).
- **Corridors** give freight at half cost on a route's corridor share and +0.2 piracy security (`01 §2.1`).
- **The Mass Driver** is a structure the trade network reads: a linked pair's route days are scaled (`07 D78`, as summarised in `07 D283`). Megastructures are on [Megastructures](Design-Megastructures).

### Freight capacity

Empire freight capacity, in tonnage per 30 days (`01 §8.2`):

```
capacity  = 100 base + Σ starbase freight by size + Σ Spaceports × 75 + Σ merchants × 10
spend     = amount × (routeDays + 2 × 2.5) × 0.05      // the corridor share at half
```

- The daily allowance is `capacity / 30`; unused allowance does not carry over. 100 units over 15 route days spends 100.
- Cross-empire shipments spend the **importer's** freight. Trade fleets add nothing to capacity (their cargo moves physically, `07 D275` call 1).
- **Utilisation** is the 30-day rolling share of capacity spent. The governor builds a Spaceport at the busiest market when it exceeds 80% (`01 §17`).
- Starting capacity is 295 per 30 days (base 100, the homeworld's size-2 Station 100, Highport's Spaceport 75, two merchants 20).

Starbase freight and piracy security by size (`01 §10.4`):

| Size | 1 Outpost | 2 Station | 3 Starport | 4 Starhold | 5 Bastion | 6 Fortress | 7 Citadel | 8 Bulwark |
|---|---|---|---|---|---|---|---|---|
| Freight | 25 | 100 | 150 | 200 | 250 | 300 | 350 | 400 |
| Security | 0.15 | 0.30 | 0.35 | 0.45 | 0.50 | 0.55 | 0.60 | 0.70 |

### Accessibility and the import cap

Each market has an accessibility factor (Starsector's), from `01 §8.2` with the gentler curve of `07 D193`:

```
accessibility = clamp(0.3 + 0.7 × (1 − daysToHub / 80) − 0.5 × maxPiracyOnSpoke + modifiers, 0.1, 1.2)
```

`daysToHub` is the market's route days to its hub; `maxPiracyOnSpoke` is the worst region level on its spoke, its own region included. Imports into a market are capped at **`accessibility × 2 × demand` per day**, and the cap counts **landed** tonnage: the dispatcher ships up to the cap ÷ the route's landed share, so a lossy route still lands its cap and pirates take more of the larger shipment (`07 D272`, the owner's call of 2026-11-20). A market with accessibility under 0.5 can therefore land at most `accessibility × 2` of its demand however much freight there is; that structural case is what the food buffer below covers (`07 D274`).

### The daily dispatcher

Per empire, then cross-empire, each tick after production (`01 §8.3`, with `07 D164`, `D236`, `D265`, `D272`; content `economy.json` `trade`):

1. **Positions.** Only markets below target or above 1.5 × target are considered. For each market and commodity: target = the policy target (default 90 × daily demand); position = stock + Σ inbound + 30 × (potential production − demand); deficit = max(0, target − position); surplus = min(stock, max(0, position − 0.5 × target)), keeping a half-target reserve and never promising unproduced goods.
2. **Push (extraction colonies).** A market with zero demand for a commodity and more than 10 in stock ships the excess to the nearest same-empire market that demands it (its hub if the hub demands it), without waiting for a deficit there. This is how a mining dome's ore reaches the homeworld every day.
3. **Order.** Deficits are sorted by price ÷ base price, most stressed first (subsidised markets first, see below).
4. **Candidates.** Each deficit takes up to k = 6 candidate surplus markets reachable in the network, ordered by cost per unit landed:
   ```
   cost = (0.02 × routeDays + (1 − accessibility) + (blockaded ? ∞ : 0) + 0.06 × Σ (level × days)) / landedShare
   landedShare = Π (1 − min(1, lossMul × level × days)) over the regions crossed,  lossMul = 0.05
   ```
   A route landing under `landFloor` (0.5) is not used, the push included (`07 D164`). The shipment is min(deficit, surplus, source stock, the day's freight left ÷ the per-unit freight, the import cap). Stock is deducted from the source at dispatch.
5. **Losses.** In each region crossed at piracy level p for d days the shipment loses min(1, 0.05 × p × d) of what is left (`07 D265`); the goods vanish and their value at the source price goes to the loot pool of the sector of the region's nearest system. Blockaded routes carry nothing. The remainder lands in the destination's inbound ring at slot routeDays + 1.
6. **Cross-empire** (after the domestic dispatch, against the partner's stocks as they stand): see commercial pacts below.
7. **Flows** are recorded per leg (30-day rolling) for the map layer and the market screens.

Two further rules sit beside it:
- **The starvation draw** (`07 D55`): a spoke whose food in stock and inbound is below 30 days of its net need (`trade.starvingDays` 30) draws on its hub below the hub's reserve, down to the hub's own 30 days of demand. It keeps no landed floor, since starvation outranks the reserve (`07 D164`).
- **Energy** never enters the source lists or the push; it reaches other systems only as charged cells (`07 D57`).

### Stock policies

Per market per commodity (`01 §7.8`): `auto` (the solver targets 90 days of demand), `stockpile N` (target N; imports until reached), `export surplus above N`, `forbid import`, `forbid export`, and `ban` (M17d, below). Defaults are `auto` everywhere; never touching them is the low-overhead path.

### The spot market

Any market trades credits for goods with an off-map spot market at the **local** price (`01 §7.7`): `market.buy` creates stock at `price × 1.1` (times the empire's price index), capped at `max(0.2 × stock, 50)` units per commodity per 30 days as a rolling allowance; `market.sell` destroys stock at `price × 0.9`, uncapped. Both are external terms in the conservation test. Stock past capacity is sold at 50% of the price. The markets automation buys to cover a shortage while credits stay above 3 months of upkeep (`01 §17`).

**The food buffer** (`07 D274`, the owner's backstop, built 2026-11-21): on every colony's monthly governor timer, whether or not its governor is on, a market that makes less food than it uses is checked. If its imports cannot land its demand (access × `importCapMul` under `governor.bufferMargin` 1.1) or it is starving (30-day food satisfaction under 0.5), and its food stock plus inbound falls under 60 days (`bufferDays`) of demand, it buys food on the spot market up to 90 days of demand, within the allowance. It needs only the credits for the purchase, not the three-month reserve. It also tops up the system's own starbase market. The food row's Forbid import turns it off; the player's default is on, and the AI's is always on. The spend books on the purchases row.

### Commercial pacts and tariffs

- **Cross-empire trade needs a commercial pact.** Empire A's deficits draw on B's surpluses in step 6. A's ledger pays B's ledger at dispatch, at the source price times the ratio of the two empires' price indices (`π_A / π_B`, `07 D57`).
- **Tariffs** (MVP, `01 §8.3` step 5): A's trade policy sets the tariff t (free 0, protectionist 20%). It adds `3 × t` to the candidate cost and caps A's imports from B at (1 − t) × the accessibility cap. There is no tariff cash flow; tariffs act only through the solver.
- **Domestic sources first** (`07 D236`, the owner's call of 2026-11-17, T1 and T2 both): a pact source is a candidate only when A's own candidates cannot cover the deficit, and it ranks after them; A's pact imports over 30 days stay under `pactImportShare` (25%) of A's 30-day income. An empire with no income imports nothing over a pact.
- **AI pacts** (`07 D242`): an AI proposes Commerce only from year 25, and only when it and the partner each hold 2 trade hubs and meet the budget's reserve. This replaced `07 D116`'s "AIs propose no commercial pacts to each other". Diplomacy itself is on [Diplomacy and war](Design-Diplomacy-and-war).

### Trade value

Clerks (2) and merchants (4) produce trade value, collected daily at the hub along the spoke (`01 §8.4`). Gross commerce `G = 4 × tradeValue`; the state's share converts at the hub by the commerce policy: `credits += taxRate × G × (1 − 0.5 × maxPiracyOnSpoke)`. At Standard this is exactly `Σ tradeValue × (1 − 0.5 × maxPiracyOnSpoke)`. Lost trade value feeds the loot pool, which gives piracy a reason to exist on every spoke.

### Piracy

Per **region** of 6 × 6 hyperspace cells (12 × 12 regions at Medium with the margin), `piracy ∈ [0, 1]`; a system's piracy is its region's (`01 §8.5`, `07 D46`). Daily, at 1/30 of these rates:

```
exposure = (unitsThrough30d + tradeValueThrough30d) / 100     // only cargo that survives the region counts (07 D165)
security = starbaseTerm (the best own starbase in the region; 0.15 at size 1 to 0.70 at size 8)
         + min(0.6, 0.2 × patrollingShipsInRegion)
         + 0.1 × claimedSystems / systems
         + (an intact corridor through the region ? 0.2 : 0)
piracy   = clamp(piracy + (0.05 × exposure − 0.08 × security − 0.01) / 30, 0, 1)
```

- **Loot and raiders.** Losses go to the loot pool of the sector of the region's nearest system. When a sector's pool exceeds 2,000, a system in it has piracy over 0.5 and no friendly fleet is there, `clamp(lootPool / 1,000, 1, 8)` raiders appear with stance raid, joining a raider fleet already there up to 8 ships; the pool loses 1,000 per raider (`07 D174`). Raid fleets were bounded as a bug fix after Huge galaxies grew to 58,717 pirate ships (`07 D174`).
- **Pirate behaviour.** Raiders engage civilian ships and weaker fleets and flee stronger ones; in raid stance they take 10% of a market's stock per 30 days; after 180 days unchallenged they found a pirate base (+0.3 to the region's piracy, once, until destroyed); after 360 days unchallenged they retire, returning 1,000 per ship to the pool.
- **Route exposure** is Σ level × days over the regions crossed; it feeds the solver's cost and the landed share.
- **Worked examples** (`01 §8.5`, 50 units per 30 days through one region): an unclaimed region climbs from 0 to 0.5 in about 33 months; two patrolling corvettes (security 0.4) bring 0.5 down to 0.2 in about 18 months. Inside owned space a size-1 starbase alone holds piracy flat below about 44 units per 30 days (60 with the region fully claimed), a size-2 Station below about 68 (84). Heavy hub spokes need patrols, hangars, corridors or Starholds even in owned space.
- **Player counters:** patrol-stance fleets, starbase sizes and hangar modules, shorter spokes (more hubs), fewer trade-dependent planets. Rising piracy shows on the trade layer as red haze.

### Blockade

A route is **blockaded** when a system it passes holds a hostile fleet with blockade stance (`01 §8.1`). A blockaded route has infinite cost in the solver and carries nothing; blockaded legs are barred on the map. A blockade at either gate system closes a gateway jump (`07 D283`), and a blockade in its system closes the Galactic Exchange (`07 D275`). In-system shipments are never blockaded from outside. Separately, a fleet at a colony of an empire it is at war with steals 5% of the market's stock per 30 days into its cargo (`01 §10.6`).

### Trade fleets (M17a)

The owner's calls of 2026-11-21 (`07 D275`), built and merged 2026-11-22:

- **Routes.** `AssignTradeRoute` gives a freighter fleet (any fleet with cargo room; escorts merged into it ride along) a route between two own markets, or an own market and a pact partner's. The player picks up to three priority commodities; by default the destination's three most stressed commodities the source can spare.
- **Loading and unloading.** Docked at the source it loads each priority's stock above half its target (the dispatcher's reserve). Docked at the destination it unloads into stock **outside the import cap and the freight allowance**, which is what a convoy is for. A convoy adds nothing to freight capacity (call 1 A; `01 §8.2`'s trade-fleet capacity term was dropped).
- **The leg** is an ordinary fleet move with fuel, storms, intercepts and incidents. A convoy destroyed loses its cargo to the sector's loot pool.
- **Cross-empire** unloading pays as the dispatcher's pact flows do (source price × the indices' ratio, no tariff cash); no pact, no route.
- **Insurance** (call 2 A): every empire may insure a route; each loaded leg pays 1.2 × the expected loss × the hold's value at the source price. A convoy lost is paid its cargo and its lead hull's alloys. Premiums and payouts book on the ships expense row.
- **Automation.** An idle freighter with automation on takes a route from its capital's market to an own market in another system that cannot land its demand (`07 D274`'s test). The AI builds up to two freighters (`ai.caps.tradeFleets`) while it has capped markets.
- **Hull.** The MVP Freighter carries 150 cargo (`01 §10.1`).

### Subsidies and embargo (M17b)

Built 2026-11-22 (`07 D275` calls 5 A and 6 A, `01 §8.7`):

- **Subsidy.** `Subsidize { market, level }`, levels 1–3, multiplies every stock target of the market by 1.5 / 2 / 3 and sorts its deficits first in the dispatcher, for level × 0.05 × Σ base price × demand a day, booked on the edicts row. It raises targets only (the price step keeps its own target) and does not lapse in debt.
- **Embargo.** A diplomatic action that closes a commercial pact for 5 years and bars a new one, at −50 opinion with the target and −10 with its pact partners; convoys' routes between them are freed. It cannot be lifted early. The AI embargoes each rival it holds, one a plan.

### The Galactic Exchange (M17c)

Built and merged 2026-11-23 (`07 D275` call 3 A; the spec session's call to trade it as a pseudo pact partner, 2026-11-22):

- **Where.** A neutral market of the unowned row at the unclaimed system nearest the centroid of the first three pact-holding empires' capitals, founded on the third empire's first commercial pact (or monthly when absent, for older saves). No one owns it; there is no Exchange treasury.
- **Members** are empires with any commercial pact. In step 6, after a member's own pact sources, one Exchange candidate is tried per deficit: the member buys from the Exchange's stock at index × 1.05 × its price index, within the pact-import share. Each member's market offering most sells surplus into it at index × 0.95 while it is under target (one sale per member and commodity a day).
- **Index and target.** Monthly, per commodity, the index is the members' demand-weighted median price (base price while no member demands it); the target is 90 days (`trade.exchangeDays`) of the members' median market demand.
- **Freight is real.** Shipments run over a lossy leg from the member's hub field to the Exchange's cell, losing cargo to piracy like any. The pillar is a physical economy, so lossy freight is the point (the spec session's call, 2026-11-22). The 5% fee each way is money destroyed (buys) and created (sales). Exchange tonnage counts toward economic victory's cross-empire tonnage.

### Smuggling (M17d)

Built and merged 2026-11-23 (`07 D275` call 4 B):

- **Contraband.** A sixth stock policy, `ban`, per market: its target is 0 and no legal import reaches it (dispatcher, push and food buffer). Its black-market price is the market price, at most twice base, read at sale.
- **Detection.** A market's empire catches a smuggler where it patrols the system, at 0.3 less 0.25 per Smuggling rank. A catch seizes the cargo into that market, fines 50 and adds to the region's detected smuggling, which fades over 360 days.
- **The persona.** With its transponder off it may trade where outlaw and sell contraband at the black-market price, untaxed; Smuggling grows by such trades. A catch also costs reputation.
- **AI smugglers** (the owner chose B, AI empires too): on its planning day an AI runs up to two smuggling routes into the nearest rival market (rivalry or war) that bans a good one of its markets makes in surplus, when the margin beats the catch risk. It takes an idle freighter first, else one off a legal route. A caught AI smuggler costs −15 opinion for a year and a year's cooldown; the rival's patrol choice weighs detected smuggling beside piracy.

### Gateway trade (M19a)

The owner's call of 2026-11-23 (`07 D283` call 1 A: a route shortcut, as the Mass Driver's), merged the same day. An empire's own hub pair jumps when both hubs' systems hold a Gateway it may use (its own, or an open-borders partner's, as fleets use them) and the jump is quicker than the flown route. The jump's leg takes the gateway's days, crosses no piracy region, loses nothing, has no corridor share, and is closed by a blockade at either gate. Freight is paid on the shorter days only. It is derived at route time, with no field rebuild when a gate is built or falls. Pact routes and the Exchange's legs still fly (their hub pairs are two empires'). No AI builds a Gateway yet (that waits for M19c).

### Map visualisation

The trade layer is part of MVP and is the pillar "trade is visible" (`01 §8.8`): each leg drawn along its route's cells, width proportional to 30-day tonnage, coloured by dominant commodity; hubs ringed; piracy haze per region; blockaded legs barred; shortages as a pulsing market ring; goods in transit as moving ticks along the route; the player's trade routes from source to destination, as wide as the freighter's hold (`07 D275`). The hyperspace layer shows clouds, storms, corridors and the deep (`07 D46`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Physical logistics | Nothing teleports; every construction draws from its own system; imports through the trade network; self-sufficiency rewarded with up to 25%. Replaced every "nearest own market by hops" draw | 2026-11-08 | `07 D39` |
| Hyperspace routes | Hub fields over the hyperspace grid, regions of 6 × 6 cells, 60-day route limit, a 64-day inbound ring. Replaced hyperlane hops | 2026-11-13 | `07 D43`, `D46` |
| Second hub | Past 30 route days, or 15 once eight colonies lie outside home. Replaced `01 §17`'s 30 days alone for the AI | 2026-11-13 | `07 D55` |
| AI commercial pacts (first) | AIs propose no commercial pacts to each other; superseded by `D242` | 2026-11-14 | `07 D116` |
| Pirates bounded | Raids merge into a standing pirate fleet up to a cap and retire to the loot pool | 2026-11-16 | `07 D174` |
| Landed-share pricing | Sources priced per unit landed; routes landing under 0.5 refused | 2026-11-17 | `07 D164` |
| Surviving cargo counts | A region's traffic counts only what survives it | 2026-11-17 | `07 D165` |
| Accessibility curve | `accessibility.days` 40 → 80 (gentler) | 2026-11-17 | `07 D193` |
| Pact trade protects the importer | Domestic sources first (T1) and pact imports capped at 25% of income (T2) | 2026-11-17 | `07 D236` |
| AI pacts from a buffer | Diagnose first (B), then pacts only from a strong buffer; built as the year-25, two-hub, reserve floor | 2026-11-17; 2026-11-18 | `07 D235`, `D242` |
| Piracy loss weight | `lossMul` 0.1 → 0.05 in the landed share and the losses (option A); a 0.3 floor and AI patrols against refused routes not chosen | 2026-11-19 | `07 D265` |
| Balanced hub days | A new hub takes the least-loaded day of the month, saved. Replaced market id mod 30 | 2026-11-20 | `07 D268` |
| Import cap on landed tonnage | The cap counts landed tonnage; the dispatcher ships up to cap ÷ landed share. A food buffer or auto-buy for small import-fed colonies not chosen | 2026-11-20 | `07 D272` |
| Supply resilience | AI site choice weighs food resilience and vetoes farmless unreachable sites (D), with a food buffer as the backstop (B); a food floor on the import cap (C) not chosen | 2026-11-21 | `07 D274` |
| M17's six calls | Convoys physical only (capacity term dropped); insurance for every empire at 1.2 × expected loss; the Exchange a neutral unowned station with a 5% fee; smuggling with AI smugglers (B); three subsidy levels; embargo as `§8.7` with the AI using it | 2026-11-21 | `07 D275` |
| Exchange as a pseudo pact partner | Members trade with the Exchange through step 5 over lossy legs (A), not an off-map spot market (B) | 2026-11-22 | `07 D275`, DESIGN |
| Gateway trade | A route shortcut for an empire's own hub pairs (A), not gateway edges in the fields (B) | 2026-11-23 | `07 D283` |

## Not yet built or still open

- **The AI builds no Gateways** until M19c, the AI's megastructure advisor (`07 D283`), so gateway trade appears only for the player in AI-only runs.
- **AI patrols against refused routes** (R2 of `07 D265`) are kept as a later design option.
- **Narrowing each field's pact targets** (S2 of `07 D267`) would change pact routes and is not built.
- **A walk cut in the dispatcher** (compacting spent sources, "C2 at this scale") was proposed by track M and not built (`07 D274`); dispatch leads the design-scale Huge worst tick (see [Architecture and performance](Design-Architecture-and-performance)).
- **Hub food forwarding** and a **perishable stock target** are not built; long-spoke food stays a probe for the MVP (`07 D194`, `D195`).
- **The 1.0 Freighter** (cargo 200) is listed in `01 §10.1`.
- **Owner review still open:** the M5 calls (`07 D34`) and the M8c calls (`07 D46`) stay on DESIGN's open list.

## Discrepancies noted

- **Do embargoes close Exchange trade?** `07 D275` call 6 A reads "also closes the target's Exchange trade with the embargoing empire and any route between them". The M17c build record in `07 D275` and the DESIGN autonomous-calls row (2026-11-23) say "embargoes leave Exchange trade alone, since its stock is pooled" (alternative listed: "embargoes closing the target's Exchange trade"). `docs/00-status.md`'s M17c merge line says "embargoes close it between the parties". Which is current is not clear from the sources; this page does not state either as the rule.
- **Where the Exchange stands.** `01 §7.7` says "at the unclaimed system nearest the first three's capitals"; `07 D275` call 3 A and the M17c build say "the unclaimed system nearest the centroid of the first three pact-linked empires' capitals". This page follows the build record.
- **Piracy loss weight in `01 §8.3`.** `01 §8.3` steps 3–4 still write the landed share as `Π (1 − min(1, 0.1 × level × days))` and losses as "lose min(1, 0.1 × p × d)". `01 §8.5` says losses are "`0.05 × level` per day in each region (`07 D265`)", and `data/content/economy.json` has `"lossMul": 0.05`. `07 D265` is later, so this page uses 0.05.
- **Trade fleets, subsidies and the Exchange labelled 1.0.** `01 §8.6` and `§8.7` are headed "(1.0)", `01 §7.7` says "1.0 adds a Galactic Exchange", `01 §17` lists "Trade fleets (1.0)" and `01 §10.1` says empires build the Freighter "for the 1.0 trade fleets". `docs/00-status.md` records M17a–d merged on 2026-11-22 and 2026-11-23. This page treats them as built.
- **Hyperlanes in the spec.** `SPEC.md` pillar 2 says goods "travel along hyperlanes"; `07 D43` replaced hyperlanes with hyperspace (DESIGN: "No hyperlanes").
- **The import cap's wording in `01 §8.2`.** It reads "Imports into a market are capped at `accessibility × 2 × demand` per day", without `07 D272`'s rule that the cap counts landed tonnage. This page states D272's rule as current.

## Sources

- SPEC.md
- 01 §2 Galaxy generation (corridors, deep outposts)
- 01 §7.7 Buying and selling
- 01 §7.8 Stock targets and policies
- 01 §8 Trade routes and logistics
- 01 §8.1 Network
- 01 §8.2 Capacity
- 01 §8.3 Daily trade dispatch
- 01 §8.4 Trade value
- 01 §8.5 Piracy
- 01 §8.6 Trade fleets
- 01 §8.7 Subsidies and embargo
- 01 §8.8 Map visualisation
- 01 §10.2 Movement and logistics
- 01 §10.4 Starbases
- 01 §10.6 Raid stance and cargo
- 01 §17 Automation defaults
- 05 Milestones (M5, M8b, M8c, the 1.0 table: M17, M19)
- 07 Decisions: `D39`, `D43`, `D46`, `D55`, `D57`, `D78`, `D116`, `D164`, `D165`, `D174`, `D193`–`D195`, `D235`, `D236`, `D242`, `D265`, `D267`, `D268`, `D272`, `D274`, `D275`, `D283`
- 14 The resource model
- 00 Status (merge records, 2026-11-21 to 2026-11-23)
- DESIGN.md (pillars, autonomous calls, change log)
- Research: supply diagnosis (2026-11-16)
- Research: shortage diagnosis (2026-11-19)
- Research: shortage options (2026-11-19)
- `data/content/economy.json` (`trade`, `piracy`, `governor` values)

## Related pages

- [Economy and resources](Design-Economy-and-resources)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Starbases](Design-Starbases)
- [Ships and fleets](Design-Ships-and-fleets)
- [Combat](Design-Combat)
- [Megastructures](Design-Megastructures)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Factions and characters](Design-Factions-and-characters)
- [AI and victory](Design-Ai-and-victory)
- [Presentation and UI](Design-Presentation-and-ui)
- [Architecture and performance](Design-Architecture-and-performance)
- [Overview](Overview)
