# Performance: goals, decisions and caveats

**Status:** current as of 2026-11-24 (main at `f765ea4`). Figures are host bench rows quoted from `00-status` and the `DESIGN.md` change log; none was re-measured for this page.

CPU cost is the project's first engineering constraint (`SPEC.md`, `08`). This page states what the goals are, where the measurements stand against them, which design decisions produced the measurements, which attempts did not pay, and how far the comparison with the reference games can be trusted. The full budget table and the engine's structure are on [Architecture and performance](Design-Architecture-and-performance); how the numbers are produced is on [Testing and benchmarking](Engineering-Testing-and-benchmarking).

## Goals

The governing statement is `08`'s opening: the game must stay cheap on the CPU at every point of a 300-year game on a Huge galaxy, "in a way that Stellaris and older RimWorld-style tick loops do not". Four goals follow from it.

| Goal | Line | Source |
|---|---|---|
| A paused game costs nothing | 0% CPU, no wake-ups; under 50 ms of CPU over 60 paused seconds | `08 §1`, `§5` |
| Normal play is a rounding error | speed 3 at Medium year 150 takes ≤ 3% of one core natively, ≤ 8% on the web | `08 §1` |
| The late game stays fast on the largest map | Huge year 300 at the design scale (≈ 450–1,000 colonies): one game year ≤ 3 s, worst tick ≤ 12 ms | `07 D234`, `D239` |
| Headroom for far more content than the MVP ships | the stress profile (Huge, ≈ 1,200 colonies, ≈ 6,000 cohorts, ≈ 40,000 ships, 20 commodities, crises active): one game year ≤ 15 s | `08 §6` (the owner's ceiling) |

Two rules govern how the goals are held:

- **A miss is fixed in the algorithm, never in the map.** "If a milestone misses a budget by more than 2× after algorithmic work, the fix is an algorithm, an incremental update or a stagger, never a smaller galaxy" (`08 §6`).
- **Content may spend the headroom; the architecture may not** (`08 §6`). A feature that adds per-tick work is judged on its cost per colony, so a larger world is told apart from a slower simulation (`DESIGN.md` change log, 2026-11-20 and 2026-11-24).

The reference machine is the test host: a virtual machine with 8 logical cores of a Ryzen 5 5500 and 20 GB, Windows Server 2025, release build. `08 §1` treats it as close to the 2020 laptop CPU the budgets were written for.

## Where the measurements stand

| Row | Goal or line | Latest recorded | Record |
|---|---|---|---|
| Medium year 150, daily tick | 300 µs as designed; re-baselined since (see below) | 1,766 µs p50 at calibration 1.20 on 288 colonies, which is 1,468 µs normalised, or 5.10 µs a colony; later merges read 5.22 and 5.38 µs a colony | `00-status`, M18b, T2 keys and T3 merges (2026-11-24) |
| Medium year 150, speed-5 throughput | ≥ 1,500 ticks/s as designed | 622 ticks/s (P-CMD, 2026-11-23); 560 ticks/s (M17b, 2026-11-22) | `00-status` |
| Web throughput on that world | ≥ 300 ticks/s | 349 to 412 ticks/s | `00-status`, smoke rows (2026-11-24) |
| AI plan, heaviest empire | ≤ 1 ms budget; 250 µs CI tripwire | 202.7 to 208.3 µs | `00-status` (2026-11-24) |
| Trade dispatcher | steady 195 µs line; worst case 441 µs line | 133.2 µs; 76.8 µs | `00-status`, M19c (2026-11-23) |
| Huge year 300, design scale | ≤ 3 s a year | 3.14 s a year at 746 colonies, accepted by the owner's "profile first, then accept" | `DESIGN.md` change log, 2026-11-24 |
| Huge year 300, worst day | ≤ 12 ms; 9.7 ms accepted baseline | 12.81 ms, open (track P, P-PROF4) | `00-status`, 2026-11-24 |
| Huge AI-only stress run (≈ 3,000 colonies) | tracked, not a gate | 6.09 s a year, 23.4 ms worst tick | `07 D239` (2026-11-18) |
| War storm on Huge year 300 | 15 s a year; 12 ms worst tick | 12.34 s a year at about 10% of systems contested; on a 2,400-colony world at 30% contested, about 15.3 s a year and a 75–80 ms worst tick | `07 D262`, `00-status`; `DESIGN.md` change log, 2026-11-21 |

Read plainly:

- **The Medium tick is about five times its original 300 µs budget.** The world it measures is also larger than the one the budget assumed: `08 §6` planned ≈ 180 colonies on Medium at year 300, and the bench world holds about 290 at year 150. Cost per colony has stayed near 5 µs across the last merges.
- **The original throughput goal of 1,500 ticks/s is not met.** At 622 ticks/s a Medium game year (360 ticks) takes about 0.58 s at full speed.
- **Huge at the design scale sits 5% over its 3 s line and is accepted there; the worst day is over its 12 ms budget and open.**
- **The stress ceiling is not yet asserted.** The 1.0 stress profile belongs to M23. The nearest runs sit at or just over 15 s a year, with worst ticks several times the 12 ms budget.

## Design decisions that produced the numbers

### Structural decisions (in the architecture from the start)

| Decision | What it removes | Source |
|---|---|---|
| The economy resolves every tick at 1/30 of its 30-day rates | the monthly spike: no tick carries a month's work | `08 §3`, `02 §3.3` |
| Timers in a binary heap; reactive work in flag sets | per-entity polling: daily cost follows events due, not entity count | `02 §3.5` |
| Incremental aggregates, checked against a full recompute in debug and `--verify` runs | monthly rebuilds of totals | `02 §5` |
| Modifiers refolded only for dirty keys | modifier re-application, named by Stellaris's developers as a top cost | `02 §5`, `08 §2` |
| Pops as cohorts per (colony, species, stratum) | per-pop production: about 900 rows where individuals would be about 90,000 | `08 §3`, `01 §6` |
| Struct-of-arrays columns with `u16`/`u32` ids; no per-entity heap objects | pointer chasing and allocation in the tick | `02 §3.2` |
| Routes computed once at order time; bounded solvers (trade capped at 6 sources per deficit, AI candidate lists capped in data) | pathfinding storms; worst cases that grow faster than entity count | `08 §3` |
| Surfaces and anomalies generated on first touch | cost for unexplored space | `08 §3` |
| The sim on its own thread, snapshots at most every 50 ms, repaint on demand | frame-bound simulation speed and idle CPU | `02 §7` |
| Rust, no managed runtime | garbage-collection pauses | `08 §2` |

### Measured decisions (made after profiling)

| Decision | Measured effect | Source |
|---|---|---|
| Pop growth and jobs staggered per colony; dispatch by per-hub source order (HP1) | Huge year 300 from 4.06 to 2.52 s a year; worst tick from 16 to 11.2 ms | `DESIGN.md` change log, 2026-11-17 |
| The ship index kept valid across fleet changes (P-CMD) | index rebuilds from about 81,000 a year to about 17; war storm 12.52 → 12.34 s a year | `00-status`, `07 D276` |
| Allied sensor cover read through a mask, not rescanned (M18a) | avoided doubling the sensor pass (845 against 420.6 ms a year on the storm row) | `DESIGN.md` change log, 2026-11-22 |
| AI plans run inline between ticks, budgeted in candidate evaluations | a plan costs 15–50 µs, below a thread pool's round trip | `07 D55` |
| The design-scale Huge row held at about 750 colonies by a cap in `tools` | the row measures the simulation, not how far a peaceful AI game happens to expand | `DESIGN.md` change log, 2026-11-24 |
| Per-dispatch Gateway flags and an Exchange-leg cache | exact; built alongside the cap fix, with no separate before-and-after figure recorded | `00-status`, 2026-11-24 |

### Attempts that did not pay

These are recorded so they are not tried again without new evidence.

| Attempt | Result | Source |
|---|---|---|
| `rayon` on the economy's loops | a parallel region cost 35–180 µs against 10–50 µs of work; removed in P1 | `07 D51` |
| A spin pool with a throttle | 4–5 ms spikes when a worker was descheduled on the loaded host; dropped | `07 D51` |
| Production and markets per colony through `sim::par` | exact, but a Huge year went from 4.16 to 4.47 s at six threads; reverted | `07 D261` |
| A parallel pass for the war storm | combat is not a candidate; none found | `07 D262` |
| Trade-dispatch cut C7 | saved nothing | `07 D276` |
| Snapping and lazy settling | dropped from P1's scope by the owner | `07 D51` |

`sim::par` stays in the tree with no caller, and the simulation runs on one thread.

### How the lines moved

The bench lines are not the original budgets. Since P1 a row's 1× line is the single-thread median of three host runs plus 10%, and CI fails at 2× (`07 D51`). The Medium year-150 line went from 300 µs through 882, 986, 1,145, 1,345, 1,546 and 1,505 µs as systems were added (`08 §1`). The Huge target went from under 2 s a year (`08 §2`, `§6`) to 3 s and 12 ms (`07 D234`), then to "judged at the design scale" (`07 D239`). Each change was an owner call or a recorded decision; the record of what was given up is in those decisions.

## Caveats compared to reference games

`08 §2` sets the project against two measured figures from Stellaris's developers: on a Huge galaxy, version 3.14 needed 30.52 s per game year at the start and 44.63 s later, and 4.3 started at 11.54 s per year (Dev Diary #410, 2026-02-05; `docs/research/2026-11-01-rust-ecosystem.md §13`). Against the accepted 3.14 s a year, that is roughly 3.7 times faster than Stellaris 4.3's opening and 10 to 14 times faster than 3.14. The comparison is indicative only, for the reasons below.

1. **Different hardware.** The Stellaris figures come from the developer's own machine, which the research note does not record. The project's figures come from one VM. No run compares both games on one machine.
2. **A headless row against a running game.** The Huge row is `tools huge`: the simulation alone, with no renderer or UI. The research note does not record how the Stellaris figures were taken. The project's own in-app rows (the smoke run at speed 5) exist only for the Medium world.
3. **Opposite ends of the game.** The Stellaris 4.3 figure is its opening year; the project's row is year 300. That favours the project, but the two late games do not hold the same things.
4. **Less military load in the measured world.** `08 §2` cites over 24,000 ships in late Stellaris saves. The design-scale row is an AI-only game, and AI-only runs "divert little to military" (`07 D239`). When fleets and battles are forced, cost rises steeply: about 12 to 15 s a year, with worst ticks of 75–80 ms.
5. **Less content.** The project is at its MVP. Each system added so far raised the tick (300 µs to about 1,500 µs at Medium), and Stellaris's developers note that better AI "tends to consume" performance gains (Dev Diary #385). The gap will narrow as content lands; the stress profile is the test of how much.
6. **The "heavily modded Stellaris" ceiling rests on an assumption.** `08 §6` takes a heavily modded game as 4–5 times vanilla's entity work. That ratio is not measured.
7. **Speed bought with granularity.** Pops are fractional cohorts, not individuals; pop growth and jobs run on a per-colony stagger; the trade solve considers at most 6 sources per deficit. These are design choices that a per-pop simulation does not make, so the two games do not compute the same thing.
8. **One thread.** The simulation uses a single core and, on current evidence, gains nothing from more. A game that threads its tick may close the gap on machines with many cores.
9. **Dated figures.** The Stellaris numbers are from February 2026 and the RimWorld notes from 2025 and October 2026; later patches may differ. They are quoted from the project's research note and were not re-checked for this page.
10. **RimWorld is not a numeric comparison.** It appears in `08` only as a source of anti-patterns (work not spread across ticks, batched pathfinding). No RimWorld figure is set against a project figure.
11. **The other reference games have no performance figures.** Starsector and Civilization, the other sources named in `SPEC.md §3`, are design references. Nothing recorded compares their speed.

## Discrepancies noted

- **The Huge target inside `08`.** `08 §2` says "under 2 s per year"; `08 §6` says "≥ 500 ticks/s, i.e. under 1 s per game year" and, in the stress paragraph, "Huge y300 ≤ 2 s per year"; `07 D234` and `D239` set 3 s and 12 ms. `08` has not been folded.
- **Design-scale colony counts.** `08 §6` gives ≈ 450 colonies for Huge at year 300; `07 D239` judges the row at ≈ 450–1,000; the row is held at about 750.
- **Ship counts at Huge.** `08 §2` gives "≈ 10,000 ships at year 300" as the Huge reference, while the `08 §6` table gives ≈ 800 fleets and the stress profile ≈ 40,000 ships.
- **The worst-day line.** The owner's accepted baseline is a 9.7 ms worst day; the budget is ≤ 12 ms; the latest reading is 12.81 ms. `00-status` measures it against 9.7 and does not say whether the 12 ms budget is the gate.
- **The current Medium line.** `08 §1` ends at 1,505 µs (`07 D151`); `00-status` quotes lines of 1,602, 1,765, 1,803 and 1,931 µs on later merges. The constant in `crates/tools/src/main.rs` is the authority.

## Sources

- SPEC.md
- 08 CPU budget: budgets, measurement, scaling targets
- 02 Architecture
- 07 Decisions: `D51`, `D55`, `D234`, `D239`, `D240`, `D261`, `D262`, `D263`, `D276`
- 00 Status (merge records of 2026-11-22 to 2026-11-24)
- DESIGN tracker (change log)
- Research: Rust ecosystem, §13

## Related pages

- [Testing and benchmarking](Engineering-Testing-and-benchmarking)
- [Architecture and performance](Design-Architecture-and-performance)
- [Roadmap and status](Design-Roadmap-and-status)
- [Pops and society](Design-Pops-and-society)
- [Trade and logistics](Design-Trade-and-logistics)
- [Home](Home)
