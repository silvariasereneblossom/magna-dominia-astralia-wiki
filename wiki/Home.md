# Magna Dominia Astralia

A grand-strategy space simulation in Rust, native-first and browser-capable: a deterministic daily-tick sim, playable in real time with pause or turn-based, with a physical economy, a dangerous hyperspace, one tactical combat engine, and CPU cost as the first constraint.

## Start here

- [Overview](Overview) — the project in one page
- [Design document](Design-Index) — the current design of every system, one page each
- [Performance: goals, decisions and caveats](Engineering-Performance) — what the CPU goals are, where the benchmarks stand, and how far the comparison with other games holds
- [Testing and benchmarking](Engineering-Testing-and-benchmarking) — the gate, the remote host, the bench, smoke and balance runs, and CI
- Session logs — what recent sessions decided
- [Doc map](Map) — every section of the project docs, linked

## Project docs

- CLAUDE.md · ≈662 tokens · Read `SPEC.md` first, then `docs/08-cpu-budget.md`, `docs/02-architecture.md` and the milestone you are implementing in `docs/05-milestones.
- CREDITS.md · ≈572 tokens · Magna Dominia Astralia — credits Code: the Magna Dominia Astralia authors (MIT OR Apache-2.0).
- README.md · ≈4,714 tokens · A grand-strategy space simulation in Rust: Stellaris-style pops, jobs and progression, Starsector-style physical commodity markets and trade
- SPEC.md · ≈2,678 tokens · Name: **Magna Dominia Astralia** (the owner's D1 of 2026-11-13, renamed by the owner on 2026-11-16, `07 D233`; `space-sim` remains the folde
- docs/00-status.md · ≈37,391 tokens · Updated 2026-11-09 (pass 22: M0–M8b delivered; pass 21: M0–M8 delivered; pass 20: M0–M7; earlier, pass 17: **spec v0.10, M0–M4b delivered, D
- docs/01-game-design.md · ≈37,627 tokens · Every system, with the rules and numbers an implementer needs.
- docs/02-architecture.md · ≈18,957 tokens · How the game is built so that one deterministic Rust simulation serves both clocks, resolves its whole economy every tick within the CPU bud
- docs/03-tech-stack-tooling.md · ≈5,456 tokens · Every tool needed to build, test, profile, ship and maintain the game, with the alternatives considered.
- docs/04-blockers-risks.md · ≈2,892 tokens · Facts about browsers were verified on 2026-11-01 (sources in `docs/research/`); Rust ecosystem versions are in `03` and `docs/research/2026-
- docs/05-milestones.md · ≈59,193 tokens · Each milestone is sized for an implementing agent working from this spec.
- docs/06-data-schemas.md · ≈10,251 tokens · Content is JSON under `data/content/`, deserialised into `serde` structs with `schemars` derives in the `data` crate and interned to indices
- docs/07-open-decisions.md · ≈200,137 tokens · **Status (2026-11-01): the owner confirmed D18-A and D8-A explicitly and accepted every other recommended option (A) as the decision.** D19 
- docs/08-cpu-budget.md · ≈7,944 tokens · The governing engineering constraint: the game must stay cheap on the CPU at every point of a 300-year game on a Huge galaxy, in a way that 
- docs/09-scoping-2026-11-02.md · ≈6,969 tokens · Status: proposals written on 2026-11-02 at the owner's request; **decided the same day, all at option A** (`07 D26`–`D32`) and folded into `
- docs/10-graphics-scoping-2026-11-05.md · ≈2,252 tokens · The owner asked (2026-11-05) for the graphics and the tools that make them, with the best quality per unit of performance as the goal, and g
- docs/11-combat-engine.md · ≈2,377 tokens · The owner asked (2026-11-13) for a real combat engine instead of simulated battles only: Starsector's appeal without its clunkiness, smooth 
- docs/12-modding.md · ≈1,928 tokens · The owner wants a highly moddable game, like its reference games, and less code with more simplicity and extensibility, without losing any f
- docs/13-ships-fleets-megastructures.md · ≈3,113 tokens · The owner asked (2026-11-13) to start scoping megastructures and fleet structure, with different names for the ships, looking at **Starsecto
- docs/14-resource-model.md · ≈1,689 tokens · The owner's redesign (2026-11-13) of the commodity chain: - higher abstractions built on commodity supply; - energy as a physical commodity 
- docs/15-ground-combat.md · ≈2,598 tokens · The owner's call `07 D7` (2026-11-13) was to create ground combat, with planet-surface exploration still to be decided.
- docs/16-flight-view.md · ≈4,736 tokens · The owner's `07 D3`: "top-down in strategic mode, plus a secondary **Starsector-style flight view** for interstellar travel.
- docs/17-civilisations-discovery-terraforming.md · ≈4,431 tokens · The owner asked (2026-11-16) to scope, for after the MVP: - civics; - baseline species, including the persona; - the tech tree; - precursor 
- docs/18-presentation-and-ui.md · ≈3,460 tokens · The owner, 2026-11-16: "I want something more immersive, something similar to the reference games.
- docs/19-factions-and-characters.md · ≈11,830 tokens · The owner asked (2026-11-18) to scope **factions** on two layers, and the link between them: - **Macro:** political factions inside empires,
- docs/20-tech-hooks.md · ≈3,513 tokens · Each tech below carries one modest modifier today and points at an effect the game can't express yet: a system still being built (M17's trad
- docs/DESIGN.md · ≈35,657 tokens · The current state of the game's design in one place, kept up to date by the spec session after every owner call.
- docs/release/0.1.0-notes.md · ≈1,158 tokens · *Approved by the owner on 2026-11-19 (`07 D263`, `D264`).
- docs/release/0.2.0-notes.md · ≈1,665 tokens · *Approved by the owner on 2026-11-22.
- docs/research/2026-11-01-platform-constraints.md · ≈2,691 tokens · Research note produced by a subagent on 2026-11-01.
- docs/research/2026-11-01-review-pass-1.md · ≈4,155 tokens · An independent review agent read the v0.1 spec cold.
- docs/research/2026-11-01-review-pass-2.md · ≈2,657 tokens · A second independent agent verified the v0.2 fixes by recomputing the spec's own arithmetic.
- docs/research/2026-11-01-review-pass-3.md · ≈2,021 tokens · A third independent agent recomputed every worked number in `01` from the rules alone.
- docs/research/2026-11-01-review-pass-4.md · ≈2,589 tokens · A fourth independent agent recomputed the four-colony opening (`01 §3`, `§4.1`) from the rules.
- docs/research/2026-11-01-rust-ecosystem.md · ≈3,629 tokens · Research note produced by a subagent on 2026-11-01.
- docs/research/2026-11-01-tech-landscape.md · ≈1,970 tokens · Research note produced by a subagent on 2026-11-01.
- docs/research/2026-11-02-sim-engine-scope.md · ≈3,594 tokens · Requested after M1 landed ("scope a new simulation engine to drive this").
- docs/research/2026-11-13-character-starsector-mountblade.md · ≈869 tokens · The owner asked (2026-11-13, through the G1 session) for the persona design (`01 §19`, `07 D44`) to look at Starsector and Mount & Blade in 
- docs/research/2026-11-16-scifi-ui-references.md · ≈12,551 tokens · **Summary.** - Surveyed 13 references: 8 strategy/4X games, 3 cockpit or console UIs, and 2 film/TV screen-graphics bodies of work (The Expa
- docs/research/2026-11-16-supply-diagnosis.md · ≈4,163 tokens · Written for the spec session to take to the owner as a design question (`07 D161`).
- docs/research/2026-11-17-stuck-builds-diagnosis.md · ≈1,866 tokens · Track P, for the spec session (`07 D163`).
- docs/research/2026-11-18-core-offloading-scope.md · ≈3,219 tokens · The owner wants the sim able to use several cores, as a capability and a player setting: "single-core versus multi-core", which changes spee
- docs/research/2026-11-18-huge-dispatch-profile.md · ≈1,783 tokens · The spec session asked for a profiling pass on `07 D234`'s Huge year-300 miss, with no code changed.
- docs/research/2026-11-18-huge-economy-profile.md · ≈1,089 tokens · The spec session asked for the next three Huge costs after `timers` and dispatch, in the same shape: jobs (843 ms a year), pop growth (568) 
- docs/research/2026-11-18-huge-timers-profile.md · ≈1,428 tokens · After the dispatcher prunes (`D166`), `timers` is the largest daily system at Huge year 300.
- docs/research/2026-11-18-rule-change-bundle.md · ≈857 tokens · Each of these changes results, so the goldens move.
- docs/research/2026-11-19-shortage-diagnosis.md · ≈2,873 tokens · The `docs/05 M14` box: *demand at markets under 0.9 30-day satisfaction is under 15% of market demand* (docs/01 §18).
- docs/research/2026-11-19-shortage-options.md · ≈1,968 tokens · This note follows `2026-11-19-shortage-diagnosis.md` (P-BAL1).
- docs/research/2026-11-20-energy-price-and-rare-metals.md · ≈2,104 tokens · The spec session asked two questions: 1.

## Keeping the wiki alive

- [Conventions](README) — layout, link style, commit rules
- [Style guide](Meta-Style-guide) — voice, tense, page types
- [How the AI-maintained wiki works](Meta-Ai-managed-wiki)
- Protocols: [1 Harmonize](Protocols-1-harmonize) · [2 Spot-check](Protocols-2-spot-check) · [3 Verify](Protocols-3-verify) · [4 Recursive harmonize](Protocols-4-recursive-harmonize) · [5 Session log](Protocols-5-session-log) · [6 Coverage](Protocols-6-homepage-coverage) · [7 Commit & ship](Protocols-7-commit-and-ship)
