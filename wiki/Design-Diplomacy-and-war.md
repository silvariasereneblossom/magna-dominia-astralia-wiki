# Diplomacy and war

**Status:** in progress, as of 2026-11-24. M10 (opinion, treaties, claims, wars, peace, the war AI), M18a (defensive pacts, alliances, migration treaties, calls to war) and M18b (vassalage) are built, with M17b's embargo; M18c (federations) is built on track C and waits for its merge; M18d–e (federation laws, intel) are designed on the owner's answers and wait for the build.

Diplomacy is how empires, and the persona, deal with each other: an opinion that every party can read term by term, treaties that link trade networks and research, claims and wars over systems, and peace terms that move colonies. Wars are paid for by the economy and bounded by exhaustion, so no war runs forever. Late in the design come alliances that join each other's wars, vassals on negotiable terms, and federations that vote by fleet, diplomatic and economic weight.

## How it works

### Contact first

Diplomacy needs comms. Sensing an unknown empire's ship or starbase starts a 60-day contact project per empire pair; its completion opens comms both ways and fires `on_contact` for both sides (M11, `07 D85`; [Events and crises](Design-Events-and-crises)).

### Opinion (built, M10)

`opinion(A→B)` lies in [−1000, 1000] and is the sum of named terms. It is computed when read (an AI plan, `Query::Opinion`), never stored or decayed by a daily pass (`07 D111`).

| Term | Value |
|---|---|
| Ethics alignment | 60 × the dot product of the two empires' ethics points |
| Border friction | −20 per own system with one of the other's within 6 cells, down to −100 |
| Non-aggression pact | +20 while it holds |
| Commercial pact | +30 |
| Research agreement | +15 |
| Open borders | +10 |
| Defensive pact | +40 (M18a, `07 D279` rule 1) |
| Rivalry | −100 toward the declarer |
| Guarantee | +30 from the guaranteed toward the guarantor |
| Trust | +1 per 30 days since the oldest unbroken treaty, up to +100 |
| Insulted | −50, recovering by 1 per 30 days |
| At war | −50 |
| Cancelled treaty | −30 for 120 months |
| The Lance | −100 with every knowing empire (`07 D80`) |
| Embargo (M17b) | −50 with the target, −10 with its partners |
| Caught spying (FC4) | −30, decaying by months |
| Ruler's relation with the persona | × 0.25 (`07 D142`) |

Attitude buckets: hostile < −200, unfriendly < −50, neutral, friendly > 50, cordial > 200 (`01 §11.1`).

### Treaties and actions

**Built in M10** (`01 §11.2`, `07 D111`): non-aggression pact; commercial pact; research agreement (+10% research on all three fields for both); open borders; declare rivalry (+1 influence per 30 days for the declarer); insult; guarantee independence (the guarantor joins a war on the guaranteed); declare war. Treaties are ledger bits per other empire.

- **Commercial pact:** opens the trade network between the parties; each side's tariff comes from its own trade policy. Pact imports need credits: an empire with no credits imports nothing across a pact (`07 D111`). A pact supplies a deficit only when no domestic source in reach covers it, and pact imports over 30 days stay under 0.25 of the importer's 30-day income (`07 D236`).
- **Closed borders:** a fleet may not be ordered into a foreign system without open borders or a war (`Reason::Closed`). Hyperspace between systems stays open. Cancelling open borders sends the other's fleets inside home.
- **Allied Gateways:** a fleet may cross between its own Gateway and that of an empire it holds open borders with (`07 D114`).
- **War closes migration** between the belligerents (`border` 0).
- **Embargo** (M17b, `07 D275` call 6): closes a commercial pact for 5 years and bars a new one; −50 opinion with the target and −10 with its partners. The AI embargoes its rivals.

**Built in M18a** (`07 D279`, merged 2026-11-23):

| Treaty | Rule |
|---|---|
| Defensive pact | the defender calls its partners to war; AI acceptance needs a shared rival, or a threat of at least `defenceNeed` 1 (the strongest bordering empire's fleet power over the pair's) |
| Alliance | a defensive pact that also calls the partner to offensive wars, shares sensor cover and opens borders; needs a defensive pact (read on the pair's trust clock) for 5 years and opinion ≥ 100; breaking it costs −200 with the partner and "Treaty breaker" −50 with every empire that met both |
| Migration treaty | `border` reads 1 between the two empires, so pops move as within one empire; AI acceptance from pop pressure (unemployed share) |

**Calls to war** (the owner's answer 1): a defender calls its defensive-pact partners and allies; an attacker calls its allies. Refusing a defensive call costs −50 opinion with the caller; refusing an offensive call costs −10. An AI answers at the declaration; the player answers in the diplomacy screen within 30 days, and an unanswered player call joins a defensive war and refuses an offensive one. Only the war's two leads call, so one declaration cannot chain across the galaxy. A partner tied to both leads answers the defender.

### AI acceptance and proposals

The acceptance score is `opinion / 10 + Σ dealValue × personalityWeights − trustRisk`; the AI accepts above 0, and the UI shows the breakdown with no hidden rolls (`01 §11.2`). Deal values 2 / 3 / 2 / 1; risk 2 for open borders and 10 when the proposer broke a treaty with it; the defensive pact's content value −3 and the alliance's −2 (`07 D111`, `D279`).

The AI's diplomacy advisor runs on its planning day (`07 D116`): peace first (status quo, then white peace, when its own acceptance is above 0); one treaty a plan from those its personality values most; one rival, the contact it likes least once under −50; war by its personality's power ratio. **Commercial pacts:** the AI proposes one only from year 25, only when both sides hold at least 2 trade hubs and meet the budget's reserve (`07 D242`), and accepts one only from its reserve (`07 D237`).

### Claims and war (built, M10)

- **Claims** cost `50 + 25 × claimsHeld` influence, halved on systems within 4 cells of an own system. They lapse at peace (`07 D112`).
- **Declaring war** needs comms, no NAP, no truce, and for conquest a claim on the target. The target's guarantors (and from M18a its pact partners and allies, by their answers) join; treaties across the sides end.
- **War goals:** conquer claims and humiliate (MVP); reunify and independence for civil wars (FC3, [Factions and characters](Design-Factions-and-characters)). Liberate, vassalise and end threat are designed (M18b).
- **Occupation is conquest during a war:** invasion and capture transfer a colony, and the war records each colony taken with its owner before. A colony retaken by its owner ends its occupation.
- **Starbase capture:** a starbase at hull 0 in a battle with an empire at war with its owner passes to that empire (the starbase, its system and its starbase colony), at 10% hull, once no armed ship hostile to the captor still fights in that battle (`07 D113`, `D149`). Other colonies in the system still need an invasion ([Ground combat](Design-Ground-combat)).
- **The Lance:** a strike on an empire's fleet or starbase while not at war with it declares war on it, to humiliate (`07 D115`).

**War score** per side, computed when read, within 0–100: +10 per occupied claim, +2 per battle won (up to 30), and 40 × the enemy ships destroyed over its ships at the start (up to 40) (`07 D112`).

**War exhaustion** per side, accrued daily only while a war stands, capped at 100:

```
+0.5 per 30 days
+ 20 × ships lost / ships at the start       (at each battle's end)
+ 30 per 30 days × pops occupied / total pops
```

The player at 100 takes −10 happiness until peace. **Both sides at 100 end the war in the status quo**, which bounds a war at about 17 years when nothing else ends it; on the host, no war lasted past 16.7 years (`07 D112`, `05 M10`). From FC3, every non-Militarist party loses 1 approval per 10 exhaustion, and a mostly Pacifist democratic coalition offers white peace at 50.

**Peace terms:**

| Term | Effect |
|---|---|
| Status quo | each occupied colony in a system the holder claimed stays; the rest return |
| White peace | everything returns |
| Surrender | the offering side also hands over every system the other side claims of it |

A 10-year truce follows. The receiver's acceptance is `own exhaustion − other's + other's score − own score`, plus 0 / 0 / 100 for the three terms; an exhausted side accepts any (`07 D112`).

### The AI at war (built)

- **War target:** after the First Jump and at peace, the weakest contact it outguns by its personality's ratio (`ai.json` `war`: militarist 1.0, expansionist 1.5, industrialist 2.0, trader and scholar 3.0), under +50 opinion, with no NAP or truce. It claims the target's two systems nearest its capital while a claim's influence stays in reserve, and declares to conquer once it holds a claim (`07 D116`). FC3 divides the ratio by `1 + hawk` from the governing coalition's war-hawk weight; a `war` of 0 still never declares. From M18a, a target's defensive-pact partners' fleet power counts with its own.
- **War orders:** at war the target fleet power is half as much again; the strongest idle fleet attacks the enemy system nearest the capital, a claimed one first, once its power is twice what stands there.
- **Invasions** go through ground combat's front: transports load `force_needed × 1.5` and invade the enemy colony needing least force that is reachable and uncontested (`07 D116`).
- **The navy** buys its parts at the yard within the reserve and builds warships at war or when under half its target power.

On the host, the first war came in years 10.5, 2.7 and 35.5 on seeds 1–3 (`05 M10`).

### Designed, not built: M18b–e (`07 D279`)

**M18b, vassalage** (about 950 lines):
- A vassal contract has four terms: **tax** (0, 5, 10, 15 or 25% of the vassal's monthly income), **levies** (none; defensive; all), **diplomacy rights** (full, or none: treaties only with its liege and the liege's federation) and the **subject type**.

| Subject type | Tax | Levies | Diplomacy | Perk |
|---|---|---|---|---|
| Protectorate | 0% | none | none | techs its liege knows cost it 25% less |
| Tributary | 15% | none | full | none; buys peace with credits |
| Vassal | 10% | all | none | its fleet power counts toward its liege's in AI threat estimates (the persona's fief terms, `07 D138`) |

- A vassal cannot declare war except for independence; the liege may not declare on it; an attack on a vassal calls its liege, whose refusal frees it. Each term harsher than none costs the vassal's opinion of its liege ("Terms" −10 per tax step, −10 per levy step, −15 for no diplomacy).
- **Renegotiation:** either side proposes new terms once per 10 years per pair, accepted by score; a liege may instead impose them for 100 influence and "Terms imposed" −50 opinion decaying over 10 years (the owner's answer, A).
- **Offers:** an AI with ≥ 3× a bordering empire's fleet power offers the subject type worth most to itself whose acceptance score is above 0; the player offers and receives offers the same way.
- **War goals:** vassalise (surrender makes the loser a Vassal), liberate (frees a vassal of the target), end threat (disbands the loser's fleets above half its pre-war power); vassalise costs influence like a claim.
- **Independence:** a vassal whose fleet power exceeds its liege's ÷ 1.5 and whose opinion is under −50 declares a war of independence; or buys out for 1,000 influence; or goes free when its liege is destroyed, refuses its defence or becomes a vassal itself.
- Domination victory will count vassals' pops ([AI and victory](Design-Ai-and-victory)).

**M18c, federations** (about 750 lines): founded by two empires holding an alliance; others join by the president's invitation and a weighted vote; up to 6 members. The president rotates every 10 years to the member with the highest contribution over the term. A member's vote weight is `(fleet_i ÷ Σfleet + dipl_i ÷ Σdipl + econ_i ÷ Σecon) ÷ 3`, where diplomatic weight is `1 + treaties held + Σ max(0, opinion of it) ÷ 100` and economic weight the monthly credit income. More than half the total weight carries a vote. Wars, laws, invitations and expulsions vote this way; an attacked member calls every member. As built (`07 D279`, M18c's build calls): AI members vote at the motion and the player's vote waits 30 days (unanswered is no); only the president moves; a carried war binds every member; dues are 2% of income from the founding; a fallen member or a foreign liege's subject leaves at no cost, and under two members the federation ends. The federation is a section of the diplomacy screen.

**M18d, laws** (about 600 lines), each voted once per law per 10 years:

| Law | Effect |
|---|---|
| Centralisation | Loose / Federal / Union: dues ceiling 2 / 5 / 10%, leaving costs −50 / −100 / −200 opinion; at Union no treaties outside the federation and the president may declare a defensive war without a vote |
| Free trade | no tariffs between members; their hubs link as pact trees do |
| Shared sensors | every member sees what every member senses |
| Common migration | a migration treaty between every pair |
| Shared research pool | a tech any member knows costs each other member 25% less |
| Common currency (Federal) | the members' price indices merge; imports between members pay at ratio 1 |
| Shared fleet (Federal) | dues and ship donations fund a fleet owned by the president's empire, commanded in every federation war, passed on rotation, returned to donors on repeal |

**M18e, intel** (about 300 lines), for the player only (the AI reads the truth): level 0 unknown, 1 contact (name, capital, colony count, fleet strength as a band), 2 relations (any treaty other than rivalry, a vassal tie, or a foothold: exact fleet power, economy and tech counts, treaties), 3 deep (alliance, federation, liege–vassal, or Espionage rank 3 with a foothold: parties, plots, planning-day intent). Espionage adds one level from rank 2 with a foothold.

### RTS and TB

Every act is a command applied on the next tick or turn. An AI answers a proposal at once; its offers and peace terms wait for the player in the diplomacy screen. AI proposals, calls, votes and renegotiations run on planning days; tax, dues and the federation treasury on the economic day. In TB the player's 30-day answer windows span turns, and the turn report lists treaties, calls, votes and payments (`05 M10`, `07 D279` rule 16).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Opinion and treaties as built | Opinion computed when read with named terms; treaties as ledger bits; pact imports need credits; closed borders | 2026-11-14 | `07 D111` |
| Wars | Wars as rows; score and exhaustion computed per `01 §11.3`; both sides at 100 end in the status quo; 10-year truce | 2026-11-14 | `07 D112` |
| Starbase capture | A starbase at hull 0 passes to the hostile empire at 10% hull (`D113`), once no armed hostile ship still fights there (`D149`) | 2026-11-14, 2026-11-16 | `07 D113`, `D149` |
| Allied Gateways | Read as open borders until alliances | 2026-11-14 | `07 D114` |
| AI commercial pacts | From year 25, with 2 hubs and the reserve met on both sides (replaces `07 D116`'s "no commercial pact of its own") | 2026-11-18 | `07 D242` |
| Pact trade protects the importer | Domestic sources first; pact imports capped at 0.25 of 30-day income | 2026-11-17 | `07 D236` |
| Embargo | Closes a pact for 5 years, bars a new one; −50 / −10 opinion | 2026-11-21 | `07 D275` call 6 |
| Alliances and calls | Alliances join offensive wars too; refusing an offensive call −10, a defensive one −50 | 2026-11-22 | `07 D279` answer 1 |
| Vassal terms | Negotiable (tax, levies, diplomacy, subject type); offers by the strong; three subject types; propose or impose | 2026-11-22 | `07 D279` answers 2–3, vassal choices A and A |
| Federations | Shared fleet as a policy funded by dues and donations; votes by fleet, diplomatic and economic shares (replaces `01 §11.4`'s "weights by fleet power"); the larger law set | 2026-11-22 | `07 D279` answers 4–6 |
| Intel | For the player only | 2026-11-22 | `07 D279` answer 7 |
| War-hawk weight | A governing coalition's Militarist or Pacifist lean scales the AI's war ratio | 2026-11-20 | `07 D217` |
| Starbase defeat (designed) | Capture stays; every fleet of the losing empire in the system retreats at capture | 2026-11-23 | `07 D284` call 3 |

## Not yet built or still open

- **M18d–e:** federation laws and intel. The research pool waits for the tech tree's cost function (`07 D278`).
- **`tools det`'s scripted federation war** (`05 M18`'s acceptance).
- **Defensive pact, alliance and federation bench rows:** a declaration with a full federation on each side, and the AI plan max with alliances, vassals and federations formed, within 5% of main (`07 D279`).
- **The station rework's retreat at capture** (`07 D284`) is designed; its milestone is not yet placed ([Starbases](Design-Starbases)).
- **1.0 first contact events** with choices (hostile, cautious, open) are not built (`01 §4.2`).
- **Espionage beyond FC4's operations** is M18e's intel.
- **Stale text to fold:** `01 §11.2`'s 1.0 list still names defensive pact, alliance and migration treaty (built in M18a) and embargo (built in M17b); `01 §11.4` still weights federation votes by fleet power; `07 D146` still says "only a player with commercial pacts" can win economically, written before `07 D242` let AIs propose pacts; DESIGN's pillar row still reads "designed (M10)" after M10's merge on 2026-11-14.

## Discrepancies noted

None found.

## Sources

- 01 §11 Diplomacy and war, 01 §4.2 Interstellar exploration (first contact)
- 05 M10, 05 1.0 milestones (M18)
- 07 decisions: `D80`, `D85`, `D110`–`D116`, `D142`, `D149`, `D217`, `D219`, `D235`–`D237`, `D242`, `D275`, `D279`, `D284`
- 19 Factions and characters (§6 war load)
- DESIGN.md (M10 autonomous calls, change log)
- 00-status.md (merge records for M10, M17b, M18a)

## Related pages

- [Overview](Overview)
- [Factions and characters](Design-Factions-and-characters)
- [AI and victory](Design-Ai-and-victory)
- [Combat](Design-Combat)
- [Ground combat](Design-Ground-combat)
- [Starbases](Design-Starbases)
- [Trade and logistics](Design-Trade-and-logistics)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Pops and society](Design-Pops-and-society)
- [Events and crises](Design-Events-and-crises)
