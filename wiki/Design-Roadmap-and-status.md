# Roadmap and status

**Status:** in progress, as of 2026-11-23. The MVP milestones are built and M14 (MVP release 0.1) has two open boxes; several 1.0 milestones are built or under way; releases 0.1.0 and 0.2.0 are published privately.

This page is the build plan of Magna Dominia Astralia: what belongs to the MVP, 1.0 and later; every milestone with one line and its state; how the work is split across concurrent tracks; and what has been released and where. The game is built milestone by milestone by implementer sessions working from the spec, each milestone gated on the test host before the spec session merges it into `main`.

## How it works

### Scope tiers

From `SPEC.md §4`:

| Tier | Definition | Contents (summary, as written in `SPEC.md`) |
|---|---|---|
| **MVP** | A complete, winnable vertical slice an outside tester can play for 2–3 hours, natively or in a browser | galaxy generation with templated home systems, a pre-FTL opening with domes, habitats and in-system logistics, exploration, colonies, pops and jobs, 10 commodities with per-market prices and goods in transit, hub trade routes, piracy, planet surfaces with auto-expeditions (lite), tech (≈ 60), 3 tradition trees, ships, fleets and abstract combat, 8 AI empires, basic diplomacy and war, leaders, both time modes, save and load, score victory |
| **1.0** | Full feature set in this spec | full tech (≈ 250) and tradition set, full surface feature events and terraforming, factions and ethics drift, situations and crises, federations, trade fleets and tariffs, Galactic Exchange, megastructures, ascension, endgame crisis, modding via data packs |
| **Post-1.0** | Out of scope but the architecture must not preclude it | lockstep multiplayer, scripted mods (WASM or Lua), mobile layout, Steam release of the native build |

Later decisions moved items between tiers or past the `SPEC.md` numbers:

- **Commodities:** 17, from E1's resource model (`07 D73`; `D4` deferred the count to E1).
- **Techs:** an open tree of about 400 techs in eight tiers; 399 in T1's content (`07 D278`, `05 T`).
- **Megastructures:** kilo, mega and giga tiers in MVP (`07 D70`), built in S1 and S2.
- **Combat:** one deterministic tactical engine (play, watch, simulate) with a calibrated resolver, in place of abstract combat alone (`07 D48`, C1, C2).
- **Code mods:** sandboxed WebAssembly in 1.0, not post-1.0 (`07 D49`).
- **Built ahead of their 1.0 slot:** factions as FC1–FC4 (replacing M15's factions), employee modes (M16), trade fleets, subsidies, the Galactic Exchange and smuggling (M17), pacts (M18a) and gateway trade (M19a).

### Milestones

States follow `DESIGN.md`'s key: **built** (merged into `main` and verified), **in progress** (on a track, or partly merged), **designed** (specified, not started). Dates are merge or delivery dates from `00-status`.

**MVP and its additions** (the order in `05` and `00-status`):

| Milestone | One line | State |
|---|---|---|
| M0 | Workspace, pipeline, gate, backend measured (`glow`) | built 2026-11-01 |
| M1 | Galaxy generation and the map | built 2026-11-02 |
| M2 | Time, commands, saves: the simulation engine | built 2026-11-02 |
| M3 | Interstellar exploration and events v0 | built 2026-11-02 |
| M4a | Hex surfaces and colonies | built 2026-11-02 |
| M4b | Pops, the daily economy and the ledger | built 2026-11-03 |
| M5 | Markets, trade network, piracy | built 2026-11-03 (verified 2026-11-04) |
| M6 | Expeditions and features | built 2026-11-05 |
| M7 | Technology, traditions, modifiers, eras, policies | built 2026-11-06 |
| M8 | Ships, combat, starbases, armies | built 2026-11-08 |
| M8b | Starbases as colonies, eight sizes, physical logistics (`07 D39`) | built 2026-11-09 (verified 2026-11-11) |
| G1 | Look and pipeline: theme, icons, star shader, nebula, territory | built 2026-11-12 |
| M8c | Hyperspace: no lanes, the weather grid, corridors, deep space (`07 D43`) | built 2026-11-13 |
| C1 | Small fleet actions on the tactical engine | built 2026-11-13 |
| C2 | Large fleet actions in squadrons | built 2026-11-13 |
| M9 | AI empires and headless balance | built 2026-11-13 |
| P1 | Economy speed-up, exact single-thread rework (`07 D51`) | built 2026-11-14 |
| H1 | Hull catalogue (64 military hulls) and fleet hierarchy | built 2026-11-14 |
| P2 | Moddability and simplicity: registries, saves by name, mod system | built 2026-11-14 |
| S1 | Kilo- and megastructures, ruins | built 2026-11-14 |
| G2 | Ships and stations as lit Blender sprites | built 2026-11-14 |
| S2 | Gigastructures | built 2026-11-14 |
| G3 | Planets and surfaces | built 2026-11-14 |
| G4 | Motion and sound | built 2026-11-14 |
| M11 | Leaders, situations, archaeology, first contact, MVP events | built 2026-11-14 |
| M13 | Turn-based polish and parity | built 2026-11-14 |
| E1 | The resource model, 17 commodities | built 2026-11-14 |
| GC1 | Ground combat on the hex surface | built 2026-11-14 |
| M10 | Diplomacy and war | built 2026-11-14 |
| F1 | The flight view | built 2026-11-15 |
| M12a | Persona: framework, character, personal fleet | built 2026-11-15 |
| M12b | Persona: service, people and legacy | built 2026-11-16 |
| HP1 | Huge-scale economy performance (staggers, dispatch by per-hub source order) | built 2026-11-17 |
| M14 | MVP release 0.1: front end, UI pass, audio, packaging, balance and performance passes, smoke | in progress: balance, budget and smoke boxes ticked; external testers and the production web deploy open |
| T | The tech tree rework: T1 schema and content | built 2026-11-23 |
| T | T2 mechanics | built 2026-11-24 (rating, latch, age gates, diffusion, `research.speed`, idle bank cap, AI great works; insight, reverse engineering, the tech keys' consumers) |
| T | T3 AI | built 2026-11-24 (research by goals) |
| T | T4 UI | built 2026-11-24 (the tree view) |
| T | T5a hooks and bench rows | built 2026-11-25 (six trade and intrigue keys, two bench rows, the pace line; awaiting merge) |
| T | T5b the Dyson line and the pace | designed (`07 D278 §16`; five points open with the owner) |

**1.0 milestones** (`05`, 1.0 table):

| Milestone | One line | State |
|---|---|---|
| M15 | Factions and ethics drift; factions replaced by FC1–FC4 (`docs/19 §9`, `07 D214`), ethics drift stays here | ethics drift designed |
| FC1 | Parties, blocs and wealth classes | built 2026-11-19 |
| FC2 | Notables | built 2026-11-20 |
| FC3 | Patron notable, coalition consent, civil-war sides | built 2026-11-20 |
| FC4 | Intrigue and lords | built 2026-11-22 |
| M16 | Persona: employee modes (commander and leader kits, ranks by favour) | built 2026-11-21 |
| M17 | Trade fleets, subsidies and embargo, Galactic Exchange, smuggling (M17a–d, `07 D275`) | built: M17a and M17b 2026-11-22, M17c and M17d 2026-11-23 |
| M18 | Pacts, vassals, federations, laws, intel in five phases (`07 D279`) | in progress: M18a (pacts) built 2026-11-23, M18b (vassals) 2026-11-24, M18c (federations) built 2026-11-24 on track C, awaiting merge; M18d–e designed |
| M19 | Gateway trade links and the AI's megastructure advisor (`07 D283`) | in progress: M19a (gateway trade) built 2026-11-23; the rest designed |
| M20 | Crises: marauders, awakened empire, endgame crisis (3 variants) | designed |
| M21 | Modding data packs: pack manager, validation UI (see [Modding and data](Design-Modding-and-data)) | designed |
| M22 | Full content: techs, traditions, 300 events, situations, species packs, portraits, music | designed |
| M23 | 1.0 release: balance, performance (stress year ≤ 15 s), accessibility, itch.io and Steam-ready build | designed |

**Other designed work, not yet placed or started:**

| Work | One line | Source |
|---|---|---|
| Presentation pass (R1–R4, U1, U2, A1) | `wgpu`, a 3D camera with seamless zoom, 3D models and planets, bloom, portraits, the command-deck UI; after the MVP | `07 D231`, `18 §4` |
| Starbase stations | Starsector-style stations of sections replacing the batteries; capture makes the loser's fleets retreat | `07 D284`; milestone not yet placed (`DESIGN.md`) |
| Great works (O1) | A wider roster of structures under the four scales Prima, Maiora, Magna and Grandia Opera Astralia, scoped in a separate session | `07 D278`, `DESIGN.md` |
| Terraforming (TF1) | Handed from M19 to `docs/17`'s TF1 | `07 D283` |
| Core offloading phases | Parallel passes through `sim::par` beyond the harness, and the Cores setting | `07 D169`, `D240`, `D261` |
| Windows installer | Inno Setup, per-user, unsigned at first | `07 D270` |

### Tracks

From 2026-11-13 the work runs as concurrent tracks, each an implementer session in its own git worktree and its own host folder (`scripts/gate-remote.ps1 -Track <name>`), with benches locked (`07 D49`). The spec session verifies each milestone with its own host run and merges it into `main` itself, never pushing (`07 D49`).

| Track | Branch | Work recorded in the sources |
|---|---|---|
| **P** | `track-p` | P2's foundation first (merged early so the others rebase onto it), P1, the rest of P2; then S1, S2, M11, M13, the economy balance pass, M14's front end, rebindable keys, stuck-build work, performance passes (`sim::par`, the war-storm bench, release prep P-REL, P-DISP) and the 0.1.0 and 0.2.0 packaging (`00-status`, `DESIGN.md`) |
| **M** | `track-m` | M9 → E1 → M10 → M12a → M12b; economic victory, the §18 balance boxes and the pre-tester soak; the AI economic model; M17a–d; M19a (`00-status`, `DESIGN.md`) |
| **C** | `track-c` | C1 → C2 → H1 → G2 → G3 → G4 → GC1 → F1; M13's web zip box; M14's release checks and icon atlas; the pirate fix and HP1; FC1–FC4; M16; M18a (`00-status`, `DESIGN.md`) |
| **T** | `track-t` | the tech tree rework T1–T5, from 2026-11-21 (`07 D278`) |

Rules the tracks share: each track has its own range of decision ids (for example track P's D150–D169, then D260–D279; `07 D169`, `D260`); at most two track gates run at once, and a bench takes a host-wide mutex (the owner, 2026-11-22, `scripts/gate-remote.ps1`); `gate-remote -Bench` also runs smoke and balance, so no track merges an expansion regression (`07 D86`).

### Releases

| Release | What happened | Source |
|---|---|---|
| **0.1.0 prepared** | 2026-11-19: the version in Cargo, `scripts/package.ps1` building the Windows zip (8.0 MB, with CREDITS, CHANGELOG, licences and THIRD-PARTY for 528 crates) and the web zip (5.8 MB) on the host; not published | `07 D263`, `00-status` |
| **0.1.0 published privately** | 2026-11-19, on the owner's go: tag `v0.1.0` at `3c9094a`; a GitHub release in the private repository with the Windows zip (8.1 MB) and the web zip (5.9 MB), MIT `LICENSE` in both; the web build deployed to the tester room (sereneblossom.gay/tester, invite code only) | `00-status`, `07 D264` |
| **0.1.0 rebuilt in place** | 2026-11-20: no testers yet, so 0.1.0 was rebuilt from `main` with the same version rather than cut as 0.1.1; the tag moved, the zips were replaced and the tester room redeployed. Saves from the 19 November build load in it | `DESIGN.md`, `docs/release/0.1.0-notes.md` |
| **0.2.0 deployed** | 2026-11-23, on the owner's yes: annotated tag `v0.2.0` on `3230cac` (main `275247d` plus the version bump, so without M17b, M17c, M18a, T1 or M17d); the private GitHub release with Windows, web and Linux zips; the tester room updated. Notes approved 2026-11-22. Saves from 0.1.0 load in it | `00-status`, `docs/release/0.2.0-notes.md` |

**0.1.0's content** (`docs/release/0.1.0-notes.md`): galaxies Tiny to Huge with up to 24 AI empires and a Sol start; hex colonies; pops by species and stratum; physical markets, hubs, freight and piracy; ship design, fleets, a resolver with an optional tactical view, armies and invasions; wars, treaties and commercial pacts; politics and notables (from the 2026-11-20 rebuild); domination and end-year score victories; saves, autosaves, export and import; mods over the JSON content.

**0.2.0's additions** (`docs/release/0.2.0-notes.md`): rebellion and civil war; the Commander and Leader starts in an AI-run empire; intrigue and lords; trade fleets with insurance; colony pacing with civil upkeep; food supply and food buffers; AI improvements; trade routing speed-ups. Known issue: Huge at the design scale runs about 3.0 s a game year with 99 days in 100 under 9.4 ms. Until 1.0, a new minor version may break saves.

### Distribution

| Aspect | Current rule | Source |
|---|---|---|
| Target platforms | Native primary, though the web is the goal; since 2026-11-18 the native client is the real target and the web build an optimisation showcase that stays single-threaded | `07 D18`, `D240` |
| Storefront | The sereneblossom.gay storefront | `07 D9` |
| Repository | Private GitHub repository `magna-dominia-astralia` (renamed on 2026-11-24 from `magnum-dominium-astrale`) | `07 D10`, `DESIGN.md` |
| Testing room | The web build in the invite-only tester room at sereneblossom.gay/tester | `00-status` |
| Windows | x64 zip and (designed) installer from `scripts/package.ps1` on the host; unsigned, so SmartScreen asks once | `07 D270`, `03 §9` |
| Linux | x86-64 zip from the release workflow, with `libasound2-dev`; built, not tested | `07 D270` |
| macOS | Out of the release matrix for 0.1.x: it needs a Mac to test and sign | `07 D270` |
| Web | `trunk build --release`; PWA manifest and a network-first offline service worker; IndexedDB saves | `03 §9`, `07 D152` |
| Licence | MIT for the code; assets keep their own licences (CC0 and those in `CREDITS.md`) | `07 D264` |
| Later channels | itch.io as a 1.0 addition; Steam post-1.0 | `03 §9`, `SPEC.md §4` |
| Version shown | `CARGO_PKG_VERSION` in the main menu; saves record it | `03 §9` |

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Spec confirmed for M0 | The owner confirmed D18-A and D8-A and accepted every other recommended option | 2026-11-01 | `07` top, `00-status` |
| Name | Magnum Dominium Astrale (`D1`), then renamed Magna Dominia Astralia | 2026-11-13; renamed 2026-11-16 | `07 D1`, `D233`, `DESIGN.md` |
| Distribution | The sereneblossom.gay storefront. Replaced `D9`'s first recommendation (GitHub Releases plus Cloudflare Pages) | 2026-11-13 | `07 D9` |
| Concurrent tracks | P, M and C after M8c, each in its own worktree; the spec session merges, never pushes | 2026-11-13 | `07 D49` |
| Build then test | The owner tests at the end; implementer sessions do not block on per-milestone play-tests | 2026-11-03 onward | `00-status` passes ("the owner tests at the end") |
| Autonomous MVP drive | The spec session drives toward the MVP in a loop, logging every ambiguous call in `DESIGN.md` for review | 2026-11-14 | `DESIGN.md` |
| Presentation timing | Finish the MVP, then the presentation pass | 2026-11-16 | `07 D231` call 13 |
| Huge judged at the design scale | 3 s a year and 12 ms at ≈ 450–1,000 colonies | 2026-11-18 | `07 D239` |
| Licence | MIT | 2026-11-19 | `07 D264` |
| 0.1.0 in place | Rebuilt as 0.1.0 rather than 0.1.1 while no testers exist | 2026-11-19 | `DESIGN.md` |
| Release workflow | Windows only from the host's `package.ps1`; macOS out for 0.1.x; Linux fixed and shipped untested; a real Windows installer "at some point" | 2026-11-20 | `07 D270` |
| Tech tree rework | Open tree of about 400 techs, built by track T in T1–T5 | 2026-11-21 | `07 D278` |

## Not yet built or still open

- **M14's open boxes:** three external testers completing a game to year 100 without a blocking bug, at least one on the web; the web build deployed to production, native binaries on a tagged release, and save export and import checked on Chrome, Firefox and Safari (owner-manual) (`05 M14`).
- **Storefront mechanics** (builds, licence keys, web hosting) for `D9` are an open owner question (`DESIGN.md`).
- **Production web deploy and the tag:** track P prepared the internal half of release prep while the owner decides on the tag, the production web deploy and the order (`DESIGN.md`, 2026-11-19).
- **Windows installer:** designed (`07 D270`), not recorded as built.
- **Code signing:** a separate owner call, because it costs money (`07 D270`).
- **macOS build:** out for 0.1.x (`07 D270`).
- **T5b; M18's later phases; M19's later phases; M20–M23; ethics drift (M15):** designed.
- **Presentation pass, starbase stations, great works, terraforming (TF1):** designed, not placed.
- **Open owner questions in the tracker** (`DESIGN.md`): the `07` call sets D34–D36, D38, D40, D42, D46 for review; the Nicoll-Dyson Lance's rules; lineage names; whether `CLAUDE.md` should say nine crates; the resolver's structural term; the storefront's mechanics; `docs/17`'s civic count and the lore bible; F1's other calls; Starsector-style extra commodities; how the eras sit beside the three Kardashev ages, and the great works' roster.

## Discrepancies noted

- **Where the MVP ends.** `05`'s preamble says "Order: M0 → M13 is the MVP (release 0.1). M14 → M21 is 1.0", while `05`'s own headings name M14 "MVP release 0.1" and its 1.0 table lists M15–M23.
- **0.1.0's platforms.** `00-status` records the 2026-11-19 publication with the Windows and web zips only ("No Linux build yet (the host's WSL has no Rust toolchain)"), while `docs/release/0.1.0-notes.md` lists a Linux zip "built by GitHub Actions and not tested". The notes also describe a 2026-11-20 in-place rebuild, and `07 D270` (2026-11-20) fixed the Linux workflow, so the Linux zip may date from the rebuild; the sources do not say.
- **Distribution channel.** `07 D9` (owner, 2026-11-13) chose the sereneblossom.gay storefront, while `03 §9` still describes "the web build ... → Cloudflare Pages" and `03 §8` has `main` deploying to production on Cloudflare Pages. Both releases so far are a private GitHub release plus the invite-only tester room.
- **What "0.2" means.** `18 §3.6` describes the presentation pass as "0.2, the milestones below", and the owner chose "Finish the MVP, then the presentation pass" (`07 D231`). Release 0.2.0 shipped gameplay work and none of the presentation milestones.
- **The status doc's header.** `00-status.md`'s first line says "Updated 2026-11-09 (pass 22: M0–M8b delivered ...)", while its body records merges through 2026-11-23.

## Sources

- SPEC.md, scope tiers
- CHANGELOG.md, README.md
- 05 Milestones, 1.0 milestones
- 00 Status
- DESIGN tracker
- 03 Tech stack: CI, distribution
- 18 Presentation and UI: milestones
- 07 Decisions: `D1`, `D9`, `D10`, `D18`, `D49`, `D70`, `D73`, `D231`, `D233`, `D239`, `D240`, `D263`, `D264`, `D270`, `D275`, `D278`, `D279`, `D283`, `D284`
- Release notes 0.1.0, 0.2.0; `docs/release/README.txt`
- `scripts/gate-remote.ps1`

## Related pages

- [Overview](Overview)
- [Architecture and performance](Design-Architecture-and-performance)
- [Modding and data](Design-Modding-and-data)
- [Presentation and UI](Design-Presentation-and-ui)
- [Time and modes](Design-Time-and-modes)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Starbases](Design-Starbases)
- [Megastructures](Design-Megastructures)
- [Progression](Design-Progression)
- [Factions and characters](Design-Factions-and-characters)
- [Diplomacy and war](Design-Diplomacy-and-war)
- [Trade and logistics](Design-Trade-and-logistics)
- [Events and crises](Design-Events-and-crises)
- [AI and victory](Design-Ai-and-victory)
