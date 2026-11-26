# Factions and characters

**Status:** built, as of 2026-11-23. The persona's character and personal fleet (M12a), service, people and legacy (M12b), the Employee modes (M16), and the four faction slices FC1–FC4 are merged; ethics drift and the eight-ethics rename remain.

The player is a **persona**: one character with a personal fleet who can trade, fight for hire, serve an AI empire, or rule an empire as deeply as she chooses, delegating as it grows. Around her, every empire has **political parties** built from what its pops believe and how they earn their living, and **named notables** (rulers, heirs, councillors, admirals, governors, magnates) who lead those parties, scheme, marry and rebel. The persona's personal acts (a quest done, a campaign funded, a marriage, an assassination) move macro politics, and party politics in turn decide who holds posts, who goes to war and who secedes.

## How it works

### The persona: frameworks and kits

One persona entity, three frameworks, seven starting kits (`01 §19`, `07 D31`, `D44`, `D218`). Modes are kits and UI emphasis, not separate rule sets.

| Framework | Kits | The persona is | Commands allowed |
|---|---|---|---|
| Empire | Empire | the ruler of empire 0 | everything |
| Independent | Trader, Mercenary, Settler; pirate as a path | a captain under a one-row free company with no colonies and a personal purse | own fleets and holdings, market trades, contracts |
| Employee | Commander, Leader, Scientist | a post in an AI-run empire 0 (`AI_RUN`) | the post's own fleet, colony or ship, and `Captain` acts; empire-wide orders are refused with `NotYourPost` |

**Start.** Persona games pre-run the galaxy AI-only for 10 years (about 3,650 ticks, deterministic, behind a progress bar; 0.61 s on the host bench against a 3 s line) so the home empire has FTL and neighbours; Empire mode starts at year 0.

| Kit | Starting state |
|---|---|
| Trader | 2,000 credits and one Freighter (civilian hull, 120 alloys, cargo 150) |
| Mercenary | 1,000 credits and 2 corvettes |
| Settler | 1,000 credits, one colony ship with its cargo, one corvette (`07 D44`) |
| Commander | Ensign in an AI-run empire 0, with a corvette (`07 D218`) |
| Leader | governor of one colony of an AI-run empire 0 |
| Scientist | captain of a new science ship at the capital (`07 D218`, the owner's call 1) |

**Transitions** happen only through play (`07 D218` call 4): Independent → Empire by founding a colony (the free company becomes an empire with a ledger and the Empire-mode screens open); Employee → Empire at High Command or Ruler (the AI ruler is dismissed and `AI_RUN` clears); Empire → Employee through the removal setting or by abdication (`Captain::Abdicate`).

### Trading, contracts and reputation (M12a)

- **Trading:** buy at `price × 1.05`, sell at `price × 0.95 × (1 − tariff)`, the external terms of `01 §7.7`; the market's empire keeps the tariff. Buys are capped at 0.2 × stock per 30 days. Prices answer through the market rules within 10 days. Delivery contracts pay `0.3 × price × amount` on top of the sale.
- **Contracts** are generated on the issuer's planning day, at most 6 open per empire:

| Contract | Pays |
|---|---|
| Bounty (a pirate fleet in sensor cover) | 300 + its power |
| Escort (a spoke with piracy > 0.3) | 200 + 2 per route day |
| Delivery (a market short within 180 days) | the bonus above |
| Survey (an unsurveyed system within 8 cells) | 150 |
| Blockade / Raid / Defend (at war) | 800 / 1,000 plus loot / 1,200 |

Taking one needs reputation ≥ 0 with the issuer, war contracts ≥ 30. Completion is detected from events the sim already emits; deadlines sit on the timer heap. An expired contract costs −5 reputation.
- **Reputation** per empire runs −100..100, drifting 1 per 30 days toward 0: +5 per contract, +20 per war contract, −20 for raiding a colony, −50 for attacking a fleet. At ≤ −30 the persona is an **outlaw** there (patrols engage, no docking); at ≥ 30 **trusted** (shipyard access, contracts, −5% prices); at ≥ 60 **favoured** (holdings in their space, a letter of marque).
- **Holdings** are persona-owned starbase rows; built: 90 days to build, a 500-unit Warehouse, a Trade Post without tariff, raids on undefended holdings. Leasing, the other modules and the tier-3 upgrade are not built (`07 D133`).

### The personal fleet (M12a, `01 §19.1`, `07 D44`)

- **The persona always has a personal fleet,** in every framework; in Empire mode it is the ruler's own fleet, formed by the two starting corvettes. Its size is capped at `4 + renown / 50` ships.
- **Per-ship fitting**, for its ships only: weapon groups, hull mods, one officer per ship, d-mods (lasting damage on salvaged or badly repaired hulls), and s-mods (hull mods built in by a story point at no slot cost). Navies use designs, so per-ship state stays bounded.
- **Templates:** a fitted ship can be saved as a template, which becomes an empire design without its officer, d-mods and s-mods.
- **Attention:** `5 + ruler skill + traditions` colonies. Each ungoverned colony beyond attention costs −2 stability on every ungoverned colony and +2 administrative capacity used. A colony under a governor does not count. The player is never forced to delegate.
- **Physical presence:** at a colony without a governor, changing its designation, issuing a planetary decree, starting a megastructure or terraforming, founding its starbase or upgrading it past size 4, and resolving its situations and crises need the personal fleet in orbit. A governed colony's governor does these remotely.
- **Colonisation needs a world:** a persona's empire with no colony cannot order colony ships or found starbases. The first world comes from a tier-3 holding upgraded into a size-1 starbase colony (300 alloys, 40 parts, 75 influence, 180 days), from conquest or purchase, or from the Settler kit.

### The character (M12a, `01 §19.2`, `07 D47`)

Skills reach three **scopes**: personal (the persona and its flagship), squad (the personal fleet) and empire (whatever the persona runs or holds a post in). Every effect is a modifier row with its tooltip.

**Progression is a hybrid.** Each skill grows by use, with ranks 1, 2 and 3 at 100, 400 and 1,200 XP of its own activity. The persona also gains levels 1–20 from all XP together, level n at `150 × n²` total. Each level gives one skill point (buys the next rank of any skill) and one story point. Rank 3 is **elite** and also costs a story point.

| Aptitude | Skill | Grows by | Rank effects (1 / 2 / 3; elite adds the last clause) |
|---|---|---|---|
| Command | Tactics | battles the personal fleet fights | squad damage +5 / 10 / 15%; empire admirals' XP +10 / 20 / 30%; elite: readiness counts +10 in combat |
| Command | Logistics | days in hyperspace | squad fuel −10 / 20 / 30%; empire supply range +1 / 2 / 3 cells; elite: no attrition at fringe depth |
| Command | Coordination | battles with ≥ 3 ships | officer slots +1 / 2 / 3; empire fleet capacity +2 / 4 / 6; elite: no over-capacity upkeep on the personal fleet |
| Command | Inspiration | victories | readiness recovery +25 / 50 / 75%; empire army strength +5 / 10 / 15%; elite: companions' loyalty never below 30 |
| Piloting | Gunnery | flagship damage dealt | flagship weapon damage +10 / 20 / 30%; elite: first volley ignores shields |
| Piloting | Helm | combat ticks on the flagship | evasion +5 / 10 / 15%, speed +5 / 10 / 15%; elite: always disengages |
| Piloting | Damage Control | hull repaired | repair +25 / 50 / 75%; a d-mod removed for 50 / 35 / 20 alloys; elite: survives one lethal hit per battle |
| Piloting | Boarding | derelicts and stashes claimed | capture after victory 5 / 10 / 15%; salvaged hulls one fewer d-mod; elite: automata caches boarded without a fight |
| Administration | Governance | days running ungoverned colonies | attention +1 / 2 / 3; stability +2 / 4 / 6 where the persona runs; elite: presence operations remote within 4 cells |
| Administration | Industry | constructions completed | build speed +5 / 10 / 15%; starbase upgrade cost −5 / 10 / 15%; elite: full self-sufficiency bonus at 65% local |
| Administration | Bureaucracy | decrees and edicts issued | administrative capacity +10 / 20 / 30; edict cost −10 / 20 / 30%; elite: one extra edict slot |
| Administration | Planning | colonies and starbases founded | colony ship cost −10 / 20 / 30%; new colonies grow +10 / 20 / 30% for 5 years; elite: founds with 2 pops |
| Trade & Intrigue | Haggling | units traded | spread −1 / 2 / 3 points; tariffs −10 / 20 / 30%; elite: buy cap 0.3 × stock |
| Trade & Intrigue | Rhetoric | deals, contracts, conversations | AI opinion of the persona +10 / 20 / 30; relation gains +25 / 50 / 75%; elite: one deal a year at score ≥ −20 |
| Trade & Intrigue | Smuggling | trades where outlaw or transponder off | sensor profile −1 / 2 / 3 cells; raiding reputation loss −25 / 50 / 75%; elite: outlaw markets still trade |
| Trade & Intrigue | Espionage | intel gathered | intel radius +2 / 4 / 6 cells; exact foreign fleet power from rank 2; elite: foreign planning-day intent visible |
| Exploration | Navigation | cells crossed at fringe depth or deeper | deep attrition −20 / 40 / 60%; storm damage −20 / 40 / 60%; elite: `cautious` routes cost no extra days |
| Exploration | Survey | surveys and deep surveys | detection radius +1 / 2 / 3 cells; `DeepSurvey` 15 / 10 / 5 days; elite: deep sites show their kind |
| Exploration | Salvage | stashes and derelicts looted | loot +25 / 50 / 75%; derelict recovery +10 / 20 / 30%; elite: caches also yield research |
| Exploration | Research | caches, anomalies, precursor sites | empire research +3 / 6 / 9% while ruler or minister; elite: precursor tech cost −25% |

Espionage also trains from FC4's operations, and Smuggling from M17d's smuggling (`07 D219`, `00-status`).

**Story points** come one per level and one per 5 caches resolved. Uses: personal (make a skill elite; recruit a companion or notable who would refuse; survive a death roll once), squad (an s-mod; recover a ship lost in battle; evade one rolled deep encounter), empire (an event option's best outcome; +20 stability on one colony for a year; one AI deal at score ≥ −20; a construction 50% sooner).

**Gear**, twelve items in three slots (`gear.json`): Implant (neural link +1 officer slot; combat cortex flagship damage +10%; longevity graft +15 years), Kit (survey rig +1 detection radius; trader's ledger spread −1 point; boarding kit capture +5%), Regalia (signet renown gain +25%; commission seal contract pay +20%; crown of office stability +3 where the persona rules).

**Renown** grows with victories (+power / 100), contracts (+5), discoveries (+10 a cache) and titles. It gates the personal fleet's size, which companions will join, and the offers below.

### Paths to power (M12b, FC3, M16)

| Path | Rule |
|---|---|
| Own colony | a holding grown into one, the Settler kit, or conquest |
| Commission | at reputation ≥ 30, paid `50 + 0.5 × renown` credits a month from the employer's ledger; the employer's enemies become the persona's; ends below 30, when the employer holds no colony, or by command (`07 D138`) |
| Vassal fief | at renown ≥ 300 and reputation ≥ 50, the empire grants its least-populated starbase system (never the capital); its colonies become the persona's under the liege's guarantee; 10% of the persona's 30-day income a month; independence by revolt (a war), buy-out (1,000 influence) or the liege holding no colony (`07 D138`) |
| Posts | in an empire that accepts foreigners (`empire.foreigners` ≥ 1: democratic +1, open immigration +1, closed −1): governor (+2 stability per Governance rank), admiral (+2 fleet capacity per Coordination rank), minister (+3% research per Research rank), councillor (oligarchies); each with a reputation and renown threshold and a salary; +1 reputation a month while held (`07 D139`). Minister and admiral posts need the coalition's consent (support-weighted coalition approval ≥ 40, `07 D217`) |
| Elections | every 5 years; score `renown + 5 × reputation + credits / 100`, won at 800. Inside a party, the score becomes a personal vote `min(score / 800, 1) × 0.5` added to the party's campaign term (`07 D217`) |
| The throne | the empire's colonies pass to the persona's empire, its fleets are rebuilt on copies of their designs, its leaders join (the ruler leaves), its treasury moves across, its wars end in white peace (`07 D139`) |

FC2 built the AI's levy on the persona's fief as `Command::Levy`, taking 10% of the persona's credits once per war of the liege, in place of `01 §19.2`'s "joining a campaign with the personal fleet" (DESIGN, FC2 calls).

### The Employee framework (M16, `07 D218`)

- **Ranks by favour:** 0 / 100 / 300 / 700 / 1,500 / 3,000 favour gives Ensign, Lieutenant, Captain, Commodore, Admiral, High Command. For the Commander and Scientist they cap the fleet at 2 / 4 / 8 / 16 / 32 / all ships. For the Leader they set the post: Governor, then governor of 2 / 4 / 8 colonies (standing in for a sector governor, since no sector system exists), Minister, Ruler. The Scientist holds the Minister post at Admiral.
- **Favour:** an assignment +20 to +60 on completion, −30 on failure, −10 per 30 days ignored; × 1.5 when it serves a governing coalition's mandate and × 0.5 against one; promotion speed × (1 + patron relation / 200) within ±50%. No other decay (call 3). `Query::Favour` gives the breakdown. The same ranks apply to a commission in a foreign empire.
- **Assignments** come from empire 0's plan, at most one at a time: Commander patrol, escort, blockade, defend (each names a system); Leader build X or raise stability; Scientist survey, research, excavate.
- Empire 0 plans on the first day of the month with no AI planner. The persona has its own purse under `AI_RUN`, paid a salary (the post's, or 20 a month for a Commander or Scientist).
- **The removal setting** (off by default; `docs/19` call 4): a coup or deposition against the persona succeeds as against an AI ruler. The persona is demoted with chance `(r + 100) / 200`, clamped to [0, 1], where `r` is its mean relation with empire 0's posted notables before the coup; demoted means Commodore (Commander) or Minister (Leader); otherwise it is exiled as a Mercenary inside empire 0's row. The setting never applies to an empire the persona founded; the Empire kit's starting empire does not count as founded (`07 D217` call 3).

### People (M12b, FC2)

- **Companions:** sixteen authored (five officers, four governors, four admirals, three scientists; six romanceable), each with a level, traits, a personality, a renown gate, a price, and likes and dislikes among event tags. Loyalty starts at 60 and moves by the tags of the persona's acts (contract, failed, victory, outlaw, trade, war, service, rule, cache). At 0 a companion leaves, an officer taking their ship. Recruited at the gate's renown or below it for a story point (`07 D140`).
- **Officers:** a leader class assigned to ships of the personal fleet; 2 slots plus `persona.officers`; each officer's skill adds 2% to the personal fleet's damage. A companion admiral in a colony's system adds 5% per skill point to the brigades' blows there (`07 D140`).
- **Relations** with foreign leaders met, at most 200 rows, drifting 1 a month toward 0; +5 per completed contract (the issuer's ruler), +1 per 20 credits of a gift (at most 25 a gift), multiplied by Rhetoric. The ruler's relation × 0.25 is an opinion term of its empire toward the persona's (`07 D142`). **Contacts** are notables at relation ≥ 30; their empire's contracts pay +20% and are listed first (`07 D216`).
- **Trainable hires:** a leader hired by the persona's empire picks one of its class's two traits at hire; the **academy** at a persona starbase of size ≥ 3 adds a chosen trait in 90 days for 100 credits × level (`07 D143`).
- **Marriage:** to a romanceable companion at loyalty ≥ 70, or (FC2) to an unmarried notable at relation ≥ 50 with a dowry paid into the notable's purse. A wedding gives +10 relation with every living member of the spouse's family, once. A child comes every 5–10 years while the spouse lives and comes of age at 16 as a leader with one inherited trait and a class chosen by the player (`07 D141`, `D216`).
- **Legacy:** the persona is mortal by default (an immortal new-game setting exists), starting at 30 with a lifespan rolled in 70–90 years. On death, Cloning (society tier 3) continues as a clone aged 25 with 75% of each skill's XP, once per 50 years; otherwise the designated heir (a child, companion or leader) continues with the fleet, holdings and fiefs, half the renown and a third of each skill's XP; elite ranks bought with story points are lost. Life Extension (society tier 2) adds 30 years (`07 D144`).

### Leaders (M11, `01 §9.7`, `07 D85`, `D86`)

Classes: Ruler, Governor, Scientist, Admiral, Expedition Leader, Officer (M12b) and Magnate (FC2); General is 1.0. A pool of 3 candidates per class per empire, levels 1–3, aged 25–45, lifespans 70–90, refreshed yearly. Hire 50 × level credits; upkeep 2 × the hiring level a month (levels gained by use cost nothing), +10% each beyond the cap of 6. XP 1 a day while assigned; level L → L+1 at 360 × L. Old age from 60 with a yearly 2% chance of an early death. Traits are modifier rows that follow the leader's post (for example Governor Educator `pop.skill_gain` +25%, Administrator +3 stability). A death fires an event whose `successor` option hires the best candidate into the post. AI empires never hire leaders; generated unpaid notables fill their heir and council seats up to 8 (cap 12) (DESIGN, FC2 calls).

### Macro layer: parties, blocs and classes (FC1, `01 §9.4`, `07 D214`)

FC1 replaced M15's faction sketch. Numbers live in `government.json` `politics`.

- **Blocs.** Each cohort drifts monthly (0.1 of the way) toward the material bloc of the jobs its stratum works on its colony: **Mining, Logistics, Researchers, Bureaucracy, Soldiers**. Its unemployed share drifts back to **unaffiliated**; jobs with no bloc keep pops unaffiliated.
- **Wealth class** (the owner's addition, `07 D241`): a colony's private wealth is held 5 / 25 / 70% by workers, specialists and rulers, at a 5% return a month against wages of 2 / 4 / 6 a pop. The owner share is `return / (return + wage)`; the rest is wage-earner.
- **Parties.** A party is an ethic pole plus a base, a bloc or a class (8 poles × 7 bases). Unaffiliated pops count through their class. A cohort's weight is `count × voteWeight`. Up to six cells with ≥ 10% of the empire's weight become parties; a party dissolves after 12 months under 5%.
- **Who counts** (by authority):

| Authority | voteWeight | Elections | Agitation rate | Suppression price × |
|---|---|---|---|---|
| Democratic | 1 per pop | every 5 years | 0.5 | 2.0 |
| Oligarchic | strata 0.25 / 1 / 4 × (1 + wealth per pop / 100) | every 10 years | 0.75 | 1.0 |
| Autocratic | strata 0 / 0.25 / 1, plus members' clout (FC2) | none | 1.0 | 0.75 |

- **Demands.** Each party has 4–5 demands, predicates over policies, edicts and empire stats; its base's first demand weighs 2. Class demands touch rents and returns: owners want taxation ≤ Standard, no Levy for 10 years, free trade; wage-earners want living standard ≥ Standard, unemployment < 5%, the Levy used or taxation ≥ High.
- **Approval** moves a quarter of the way each month toward `20 + 60 × demands met`, +10 in the coalition, −15 suppressed, +5 promoted, plus the leader terms below.
- **Effects.** Members' happiness ± (approval − 50) / 5. Influence `3 × support × max(0, approval − 50) / 50` per 30 days.
- **Elections:** vote share `support × (0.8 + approval / 250) × (1 + campaign) × leader charisma`; the largest parties form a coalition of half the vote. Each coalition party's core demand is a **mandate**: +5 approval and +25 influence if met at term end, −10 if missed. The persona stays head of state; the coalition's largest party's leader becomes **chancellor** (+2 clout).
- **Levers:** suppress or promote (20 or 10 influence a year per percent of support, × the authority multiplier); embrace (one ethic point toward the party's pole for 100 influence, with a reform-style cooldown); concede a demand by changing the policy. The AI concedes one demand a plan to its most-supported agitating party, and suppresses only when it cannot concede.
- **Agitation.** Under 30 approval for 24 months, a party agitates: a `faction_agitation_<base>` situation with stage effects at 25 and 50 (for example Mining: miner output −25% at 50), progressing `(30 − approval) / 3 × rate` a month. FC4 adds a stage at 75 that starts the party leader's plot.

### Micro layer: notables (FC2, `07 D216`)

- **The roster:** 8–12 unique people per empire, recomputed on its politics day: the ruler, the heir, up to 5 council seats (the highest-clout hired leaders), the two largest fleets' admirals, governors of the three largest colonies, 1–3 magnates, and each party's leader.
- **Magnates** are a leader class with no post and no wage: mining baron (Mining), merchant prince (Logistics), shipyard dynasty (Soldiers). Their wealth is read as a share of their home colony's private wealth.
- **Columns:** two of 16 personality traits (Ambitious, Loyal, Greedy, Honourable, Zealous, Cautious, Vengeful, Pragmatic, Charismatic, Corrupt, Reformer, Traditionalist, Brave, Paranoid, Generous, Schemer), a pole and a bloc, an ambition with a target and a deadline, wealth, clout, loyalty, up to 4 bonds (friend, rival, spouse, patron, client), and a family (the house, for everyone).
- **Clout** = posts (ruler 5, heir 2, council 2, admiral and governor 1) + wealth / 1,000 + the admiral's fleet-power share × 10.
- **Ambitions**, stepped every 90 ± 30 days on a seeded timer: win a post, get rich, pass a demand, ruin a rival (FC2), plus become ruler, avenge a death, gain a fief (FC4). One step per fire: a met ambition gives +10 relation to whoever helped; a blocked one adds a rival bond (or, for a Schemer, Vengeful or Ambitious notable under 40 loyalty, a plot); otherwise lobby, seek a post, offer the persona a quest, or save.
- **The persona's actions** (`Captain::` commands): converse (same system, or a post in their empire; reveals traits, and ambition at Rhetoric 1), gift, personal quest (relation ≥ 0; success gives relation, a favour owed and party approval), call in a favour (−3..+3 per relation), patronage (relation ≥ 30; a client bond), recruit (their loyalty < 40, relation ≥ 30, renown ≥ clout × 10), marriage, betray (−50 relation with them and their family, −20 reputation with their empire). A notable whose clout × 10 exceeds the persona's renown will not deal personally without a post, a contact or a story point.

### The bridge: leaders, campaigns and conflict (FC3, `07 D217`, `D271`)

- **Party leaders** change only at the yearly **party congress** or on the leader's death or departure. Leader terms on approval: trait modifiers (Charismatic +5, Corrupt −5), the leader's opinion of the ruler `(loyalty − 50) / 5` (±10), Charismatic × 1.1 vote share, Zealous core demand weight 3, Pragmatic 3 of 4 demands counts as all. A leaderless party takes −10 approval for a year.
- **Campaigns:** `campaign = 0.1 × ln(1 + spend / 500)`, where spend is the term's patronage paid to the party's leader and members plus 5% of each member magnate's wealth on election day.
- **The persona and parties:** join (with a post, fief or commission there, or in its own empire; other parties' leaders lose 10 relation); lead at congress when its clout (renown / 10 + posts + fiefs + credits / 1,000) exceeds the leader's, then naming the mandate and taking the party's unmet demands as quests; found (renown ≥ 300, in an empire that accepts foreigners or its own; starts at ≥ 5% support; dissolves after 24 months under 5%).
- **Endings at 100**, after FC1's step and `07 D271`'s ladder (a 20-year grace in which an ending takes only FC1's step; the FC3 step needs a second ending within 10 years of the first):

| Authority | First step (FC1) | Escalation (FC3) |
|---|---|---|
| Democratic | snap election | if still under 30 approval, colonies where the party holds the majority and stability is under 30 secede |
| Oligarchic | party forced into the coalition, −10 stability for a year | the council deposes the ruler when ≥ 3 councillors are members or rivals of the ruler; otherwise secession |
| Autocratic | Planetary Unrest on the party's majority colonies | a coup that succeeds when aligned admirals command ≥ 50% of fleet power; otherwise civil war |

- **Secession and civil war:** a rebel empire takes one of 8 reserved rebel rows (beyond the cap, a 10-year autonomy modifier: −50% of tax to the parent, +10 stability). Colonies transfer one a day; fleets of party admirals under 30 loyalty are rebuilt for the rebels. The war's goals are **reunify** and **independence**. A war of independence with no change for 3 years ends in the rebels' white peace. A rising never takes the parent's capital. Against the persona, a coup or deposition becomes a civil war unless the removal setting applies; the persona's personal fleet is always loyal.
- **Succession wars** (AI autocracies): the heir takes the throne, and a claimant (family, or a rival of the heir) with clout ≥ the heir's may press a claim as a civil war.
- **War-hawk weight:** `hawk = Σ coalition vote share × pole sign × 0.25`, × 0.5 / 1 / 1.5 by authority (Militarist +1, Pacifist −1). The AI declares war at power ratio `war / (1 + hawk)`. Brakes: every non-Militarist party loses 1 approval per 10 war exhaustion; a mostly Pacifist democratic coalition offers white peace at exhaustion 50.

### Intrigue and lords (FC4, `07 D219`)

- **Plots** ride the plotter's ambition step: coup and impeachment against the ruler, assassination of a notable, secession for the plotter's party. Progress per step `10 + 2 × clout + Σ allies' clout`, × 1.5 Schemer, × 0.5 Cautious. Discovery 5% per step, +10% under a Paranoid ruler, +5% per 25 progress. At 100 a plot fires FC3's ending. The player sees plots from progress `75 − 25 × Espionage rank` in empires where it has a foothold.
- **Plots against the persona** (the owner's rule): progress × 0.5; at 100 a kill roll of `35% − 5% × Espionage rank`; a kill runs the clone or heir path; a failed kill exposes the plot.
- **Operations**, one live at a time per actor, for the persona and for AI empires (owner's call 1B):

| Kind | Espionage rank | Effect |
|---|---|---|
| Expose a plot | 1 | it fails; +1 favour with the target |
| Blackmail | 1 | a favour owed by force, −30 relation |
| Sabotage | 1 | −25% job output at a colony for 90 days; one at a time per colony, then 90 days' immunity |
| Rig | 2 | the party's next vote share × 1.1 |
| Steal tech | 2 | 50% of the tech's cost |
| Hostage | 2 | a notable held for ransom, a favour or release |
| Assassinate | 3 | the target dies; its kin may seek vengeance |

Success is the kind's base + 10% per rank − 10% against a Paranoid target; exposure `30% − 5% × rank`, rolled whether or not it succeeded. An exposed persona loses 30 reputation with the target's empire. AI empires run sabotage, steal tech, rig and assassinate (never the persona), with a rank from their era; the player's defences lower success by 5% per Espionage rank and 10% under a Paranoid ruler. Hostages can also be captured in battle (50%) when the personal fleet destroys a notable admiral's fleet. There are no duels.
- **The Feudal civic and lords:** autocratic or Feudal empires grant governor notables and magnates one fief each, up to one per 8 colonies (at most 3), never the capital. A lord keeps 0.9 of the fief's tax (owes 10%), holds a **house guard** of 2 corvettes that counts on its side in a coup, pays a levy of its wealth at war, and may defect with the fief to a stronger rival at loyalty under 20. Under autocracy or Feudal, a house's head is its highest-clout member, who inherits a dead notable's party, half the wealth, rival bonds and fief.
- **Ideology spread:** a party at ≥ 60% approval for 12 months pushes its pole by 1% a month (cap 3% per pole) into the cohorts of empires with a commercial pact with its own, shown in `Query::Politics`.

### How the two layers connect

| Persona act | Macro result |
|---|---|
| A notable's quest completed | +approval for their party (shown in the tooltip) |
| Patronage paid | a client bond, campaign spend in elections |
| Joining, leading or founding a party | the persona's personal vote and mandates |
| Marriage into a family | +10 relation with every living member |
| Exposing a plot | the plot fails; −15 approval for the plotters' party, +10 loyalty for the target's |
| Serving the coalition's mandates (Employee) | favour × 1.5 |
| Operations (rig, sabotage, assassinate) | election swings, output loss, deaths that start vengeance |

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Player character and modes | One persona in three frameworks; Independent in MVP, Employee in 1.0 | 2026-11-02 | `07 D31` |
| One game | Fleet and empire in one game; delegation by soft penalty and presence; first world from a holding or Settler kit; per-ship fitting with templates | 2026-11-12 | `07 D44` |
| The character | Skills by use plus points, story points at three scopes, gear, renown; every path to power; companions, relations, trainable hires; heir, life extension, cloning, marriage | 2026-11-13 | `07 D47` |
| Factions | Ethic pole + material bloc + leader alignment; five blocs; who counts by government; persona stays head of state with an optional removal setting; elections 5 / 10 years; 8–12 notables per species list plus the owner's authored ones; family tracking for all, formal houses under autocracy or Feudal; dark actions; NPC-driven wars; at most 8 rebel empires; marriage with notables | 2026-11-18 | `07 D241` |
| Ownership axis | A wealth class (owner or wage-earner) beside the blocs | 2026-11-18 | `07 D241` |
| Escalation ladder | A 20-year grace and a 10-year ladder before coups and civil wars; rebels' white peace after 3 stalled years | 2026-11-20 | `07 D271` |
| Employee modes | A Scientist kit; removal by a reputation-scaled chance of demotion or exile; no favour decay; abdication | 2026-11-21 | `07 D218` |
| Intrigue | AI empires run operations; plots can kill the persona at lower odds; no duels (replaces `docs/19 §4.5`'s duels); hostages by operation and capture; ideology spread now; lore hooks now | 2026-11-21 | `07 D219` |
| Lord's tax | A lord owes 10% of the fief's tax | 2026-11-21 | `07 D219` (spec call A) |
| Fanatic parties | Left to 1.x (no effect specified) | 2026-11-20 | `07 D217` call 7 |
| Persona mortality | Mortal by default, with an immortal setting (replaces `01 §19`'s "exempt from the yearly death roll by default") | 2026-11-13 | `07 D47`, `D144` |

## Not yet built or still open

- **Ethics drift** (M15's remainder) and the **eight-ethics rename** (CV1); parties use the four built axes ([Civilisations and species](Design-Civilisations-and-species)).
- **Fanatic parties** are left to 1.x.
- **The owner's 20–30 authored characters:** the hooks exist; no characters ship until she writes them.
- **Not built inside the shipped slices:** holdings leasing, other modules and the tier-3 holding upgrade (`07 D133`); the persona's own brigade with a death roll (`07 D145`); lending a house guard by a favour; replacing a destroyed guard (`07 D219` calls 24–25); a defection test (`07 D219`).
- **The persona's species, birthright traits and origins** wait for SP1 and CV1 (`07 D230`).
- **Stale text to fold:** `01 §19` still says the persona "is exempt from the yearly death roll by default" and lists employee kits as commander and leader with the Scientist inside the Leader kit; `01 §9.5` still gives election intervals of 10 and 20 years; `docs/19` still describes duels and the draft blocs (Labour, Industry, Commerce, Rentiers, Intelligentsia), superseded by `07 D241` and `D219`.

## Discrepancies noted

- **The Commander kit's ships.** `01 §19` Start: "Commander (1.0) an Admiral at level 1 with 2 corvettes and 20 credits per 30 days salary". `07 D218` Builds: "The persona starts in it at Ensign with a corvette (Commander)". The salary matches (20 a month, `07 D218` call 8); the number of corvettes does not, and the build record does not restate it.

## Sources

- 01 §19 Player character and modes (with §19.1 and §19.2), 01 §9 Civilisational progression (§9.4, §9.7)
- 19 Factions and characters: §3 macro layer, §4 micro layer, §5 the bridge
- Research: Starsector and Mount & Blade
- 05 M12a, M12b and the 1.0 table
- 07 decisions: `D31`, `D44`, `D47`, `D85`, `D86`, `D130`–`D145`, `D214`, `D216`, `D217`, `D218`, `D219`, `D241`, `D271`
- DESIGN.md (FC2–FC4 autonomous calls, change log)
- 00-status.md (merge records for M12b, FC1–FC4, M16)

## Related pages

- [Overview](Overview)
- [Progression](Design-Progression)
- [Civilisations and species](Design-Civilisations-and-species)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Events and crises](Design-Events-and-crises)
- [AI and victory](Design-Ai-and-victory)
- [Ships and fleets](Design-Ships-and-fleets)
- [Starbases](Design-Starbases)
- [Trade and logistics](Design-Trade-and-logistics)
- [Flight view](Design-Flight-view)
- [Ground combat](Design-Ground-combat)
- [Pops and society](Design-Pops-and-society)
