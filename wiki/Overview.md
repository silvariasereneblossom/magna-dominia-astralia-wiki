# Magna Dominia Astralia — overview

Magna Dominia Astralia is a grand-strategy space simulation written in Rust, native-first and also built for the browser from the same code. One deterministic daily-tick simulation plays either in real time with pause or turn-based. The governing constraint is "maximum complexity, minimum overhead", read three ways: CPU cost, player clicks and developer effort (SPEC.md §1).

The project was named Magnum Dominium Astrale until the owner's rename of 2026-11-16 (`07 D233`). The folder is still `space-sim`. The GitHub repository was renamed from `magnum-dominium-astrale` to `magna-dominia-astralia` on 2026-11-24; the Cloudflare Pages project in `ci.yml` keeps the old name.

## The game in six pillars

| Pillar | Meaning | Design pages |
|---|---|---|
| Economy is the spine | Every action moves goods, prices, pops and credits; military, diplomacy and research are paid for by the economy | [Economy and resources](Design-Economy-and-resources), [Pops and society](Design-Pops-and-society) |
| Trade is physical and visible | Every unit of every good is somewhere and moves by shipment; flows are drawn on the map and can be pirated, blockaded and taxed | [Trade and logistics](Design-Trade-and-logistics) |
| Civilisations progress | Technology, traditions, government, ethics and eras change what an empire is, and pops push back | [Progression](Design-Progression), [Factions and characters](Design-Factions-and-characters) |
| Planets are places | Each planet has a hex surface to survey and build on; the home system is the first map | [Colonies and surfaces](Design-Colonies-and-surfaces), [Starts and exploration](Design-Starts-and-exploration) |
| One simulation, two clocks | Real time with pause and turn-based are two controllers over one tick loop | [Time and modes](Design-Time-and-modes) |
| Native-first, browser-capable, always cheap | One codebase, a desktop binary and a WebAssembly build; idle costs nothing | [Architecture and performance](Design-Architecture-and-performance) |

The pillars are quoted from SPEC.md §2. The [design index](Design-Index) lists every system page.

## How the parts connect

```
data/content/*.json ──► data ──► sim ◄── ai
   (content, mods)      (schemas)  │      (planners, one empire per day)
                                   │ Command in, Snapshot out
                                   ▼
                       view ──► ui / render ──► app (native and web)
                                   ▲
                               tactics (the player's battles)
        tools: bench · det · datacheck · run · replay · balance · gate
```

- **`sim`** owns all game state in struct-of-arrays tables. State changes only through `Command`; the UI reads published snapshots and never mutates state.
- **`ai`** plans for each AI empire once per 30 days on a staggered day.
- **`data`** deserialises the JSON content under `data/content/` and validates it (`tools datacheck`). Gameplay numbers live there, not in Rust.
- **`tactics`** is the tactical combat engine for battles the player plays or watches; other battles go through the calibrated resolver in `sim`.
- **`view`**, **`ui`**, **`render`** and **`app`** present the snapshot with egui on the `glow` backend.
- **`tools`** is the headless runner: determinism hashes, benches against the CPU budget lines, replay, balance runs and the gate.

The workspace holds nine crates under `crates/` plus the excluded `lint-fixture`. `CLAUDE.md` says eight; the difference is `tactics`, and the question is open with the owner (DESIGN.md, open questions).

## Where the truth for each thing lives

| Question | Source of truth |
|---|---|
| What is the game, its scope tiers and glossary | SPEC.md |
| What is the current design of a system, in one place | The [design pages](Design-Index) of this wiki, which summarise and cite the rows below |
| The exact rules and numbers | docs/01-game-design.md |
| What the owner decided, and when | docs/07-open-decisions.md (D-numbered; a later decision overrides `01` until folded in) |
| Area designs added after the base spec | `docs/09` to `docs/20`, listed on [Home](Home) |
| The state of each pillar and the change log of owner calls | docs/DESIGN.md, the tracker the spec session updates after every call |
| What is built and verified | docs/00-status.md and docs/05-milestones.md |
| How the code is built | docs/02-architecture.md, docs/06-data-schemas.md |
| What the code may cost | docs/08-cpu-budget.md, which outranks feature convenience |
| Whether a change is verified | The remote gate, `scripts/gate-remote.ps1`; only its log on the test host claims a criterion |

The code and the gate are the final authority. The wiki describes them and is checked against them (Protocols 2 and 3).

## How work is run

The owner directs and makes design calls. A spec session keeps the documents and verifies and merges the tracks. Implementation runs on concurrent git worktree branches (tracks), each built by Opus agents and gated on the test host before a merge. Cargo, tests and benches do not run on the development PC (`07 D32`).

## Related pages

- [Home](Home)
- [Design index](Design-Index)
- [Roadmap and status](Design-Roadmap-and-status)
- [Doc map](Map)
