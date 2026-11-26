# Architecture and performance

**Status:** built, as of 2026-11-23. The engine was built in M0–M2 and reworked in P1 (performance) and P2 (saves and registries); `sim::par` merged 2026-11-18 with no parallel caller yet.

The game is one deterministic Rust simulation that serves both clocks (real time with pause, and turn-based), runs natively and in a browser from the same code, and saves in well under a second. CPU cost is the first design constraint: the whole economy resolves every tick at 1/30 of its 30-day rates, so no tick carries a month's work, and a paused game uses no CPU. For the player this means a large late game that stays fast, saves that replay exactly, and turn-based and real-time play that give identical results.

## How it works

### Principles

From `02 §1`:

1. **The sim is a pure library crate.** `sim` has no I/O, no clocks, no randomness except its own streams, and is `#![forbid(unsafe_code)]`. Threads are allowed only through `sim::par` (see "Multi-core" below; `07 D240`). It exports `World::new(seed, settings)`, `World::step(&mut self, commands)`, `serialize`, `deserialize`, `hash` and read-only queries. Everything else is a client of that API.
2. **Commands in, snapshots out.** State changes only through serialisable commands applied at tick boundaries. The UI never mutates sim state.
3. **Struct-of-arrays columns.** Entities live in `Vec` columns indexed by small integer ids, with no per-entity heap objects.
4. **Daily, incremental, flat.** Everything resolves every tick at 1/30 of its per-30-day rate. Timers live in a heap, reactive work in flag sets, totals in incremental aggregates. AI planning is the one periodic job, staggered across the month.
5. **Fixed system order.** Systems run in a declared order.
6. **Data-driven content.** Content is JSON validated by `serde` and `schemars` and interned to indices at load (see [Modding and data](Design-Modding-and-data)).
7. **Minimal code.** Hand-rolled where a crate would add more surface than it removes; crates where they remove real work.

### Crates

The workspace `Cargo.toml` declares `members = ["crates/*"]` and excludes `crates/lint-fixture` (a crate that must fail clippy, linted separately in CI). Checked on 2026-11-23, the workspace has **nine member crates**:

| Crate | Depends on | Must not depend on | Contents |
|---|---|---|---|
| `sim` | `data` | `std::time`, I/O, `rand`, any UI or graphics crate; threads outside `sim::par` | world, tables, systems, commands, event engine, trade dispatcher, combat, galaxy generation, serialize, hash |
| `ai` | `sim`, `data` | UI, graphics | advisors, planners, personalities; emits commands |
| `tactics` | `serde`, `schemars`, `postcard`, `libm` | every game crate, `std::time`, threads, I/O | the tactical battle engine: a battle as a pure function of setup, seed and input log (added in C1) |
| `data` | `serde`, `schemars` | `sim` | content JSON, schema types, registry loader and interning, derived tables |
| `view` | `data` | `sim` internals | the `SimView` snapshot layout, the `Command` and `TickEvent` enums, the `table!` macro, the hash and the UI-visible tables |
| `render` | graphics backend, `view` | `sim` | galaxy and system renderers, instanced pipelines, viewport |
| `ui` | `egui`, `view`, `render` | `sim` | screens, store, tooltips, hex surface widget |
| `app` | all | none | the `eframe` shell (native and web), sim thread and web stepper, saves, settings, PWA assets |
| `tools` | `sim`, `ai`, `data` | graphics | headless runner, `bench`, `datacheck`, `det`, `replay`, balance, sprites, smoke |

Boundaries are enforced by `Cargo.toml` dependencies and by clippy `disallowed_types` and `disallowed_methods` in `sim` and `ai` (`02 §2`). `sim`, `ai`, `view` and `data` are `#![forbid(unsafe_code)]` (`02 §6.1`).

### The deterministic simulation

**World and tables.** `World` holds the seed, the tick (tick 0 is day 1 of year 1), immutable settings, the RNG streams and every table (`02 §3.1`). Tables are declared with the `table!` macro, which expands to struct-of-arrays columns with an `alive` bit vector, a generation per slot and a free list. Ids are `#[repr(transparent)]` newtypes over `u16` or `u32`; `Id::NONE` is `u16::MAX`. Writes set the table's dirty bit. Parent and child access (ships of a fleet, cohorts of a colony) uses a CSR index rebuilt lazily when the child table is dirty (`02 §3.2`). Since P2, registry-backed columns (commodities, jobs, districts, strata, policies) are `Row` columns as wide as the content's counts, not the caps (`07 D50`).

**Step.** Each tick applies pending and external commands, runs the daily list, runs the AI planning day for the empire whose day it is, runs the yearly list every 360 ticks, advances the 30-day rings and drains events (`02 §3.3`). The daily list is 27 systems in a fixed order, from `commands`, `timers` and `fleet_movement` through the economy (`jobs`, `production`, `trade_dispatch`, `markets`, `shortages`, `ledger`, `pop_growth`, `happiness`) to `stats` and `cleanup` (`02 §3.4`). "Month" has no simulation meaning: the UI reads 30-day rolling sums, and a turn-based turn is 30 daily steps.

**Due-work structures** (`02 §3.5`):

| Structure | What it holds | Effect |
|---|---|---|
| Timer heap | `BinaryHeap<Reverse<(tick, TimerId)>>`, `TimerId` packing a kind and an id; constructions, arrivals, event timeouts, expeditions, governor re-evaluations, leader deaths, corridor drift | daily cost proportional to events due; cancellation is lazy through a `timer_gen` column |
| Flag sets | `in_transit`, `surveying`, `threatened`, `contested`, `sensor_dirty`, `invaded`, `jobs_dirty`, `shortage`, `below_target`, `above_target` and others | systems iterate only flagged items, in id order |
| Incremental aggregates | colony housing, jobs, amenities, defence; empire sprawl, fleet power, freight capacity | updated on every write path; a debug and `--verify` full recompute asserts equality |
| Rolling rings | 30-slot rings per series, slot-major with a running sum; market inbound rings of 64 slots | every 30-day figure is an O(1) read; the daily clear is one contiguous pass |

**Commands.** `Command` and `TickEvent` live in `view`, so `sim`, `ai`, `ui` and the replay file share one definition, serialised with `postcard` (`02 §3.7`). Every command applies at a tick boundary, so real time and turn-based behave alike. Invalid commands produce a rejection event.

**Snapshots.** `SimView` holds `Arc`s of the same tables the sim holds, grouped by change rate; panels compare `Arc` pointers to skip work. Publishes are rate-limited to 20 Hz. The web build reads the live columns with no copy (`02 §7.3`).

**Modifiers.** A modifier is `(scope, key, value, op, source, expires)`. The effective value is `base × Π(1 + mul) + Σ add`, held in a cache refolded only for dirty keys. Sources are kept so every tooltip shows the full stack (`02 §5`, `07 D36`).

**Determinism rules** (`02 §6.1`, `CLAUDE.md`). Banned in `sim` and `ai` by clippy: `std::time`, `std::fs`, `std::net`, `rand`, `HashMap` and `HashSet` iteration, `sort_unstable*`, `mul_add`, the `std` transcendental float methods (`powf`, `exp`, `ln`, `sin`, `cos`, `atan2` and the rest; `libm` is used through `fmath.rs`), and `f32::min`/`max` on possibly-NaN inputs. Sorts end with an id tiebreak. No fast-math or `+fma` flags are enabled. All randomness comes from PCG32 streams, one per subsystem, seeded as `splitmix64(seed ^ stream_id)`, saved with the world (`02 §6.2`). Combat draws per unit from `(seed, unit, tick)`.

**Hash and goldens.** `hash(&World)` is FNV-1a over every table's live columns, ledger scalars and rings, with NaN canonicalised (`02 §6.3`). Golden hashes for 3 seeds are committed and change only in a `sim-change:` commit.

### Threading model

| Platform | Model |
|---|---|
| Native | Two threads. The sim thread owns `World`, runs the time controller and publishes a snapshot at most every 50 ms and only when something changed. The UI thread renders and sends commands over a channel. The sim thread parks when paused, so a paused game uses 0% CPU (`02 §7.1`). |
| Web (wasm32) | One thread. Each frame the stepper runs ticks until the speed target is met or 8 ms is spent; a 1 Hz hidden-tab timer runs with 200 ms; a Web Lock avoids Chrome's Energy Saver freeze. Web Workers with shared memory are not used (`02 §7.2`, `§8`). |

Both clocks are one `Driver` (`crates/app/src/driver.rs`). Player commands in both modes go through `pending_commands` at `World.tick`, so they are saved while paused and revocable until the next tick (`02 §8`).

**AI planning** runs once every 30 days per AI empire, AI index `e` of `N` planning on day `floor(e × 30 / N)`. The plan reads the world at the start of its tick and its commands apply on the next tick. No thread is used: a plan costs 15–50 µs at Medium year 150, below a pool's round trip (`07 D55`). Budgets are counted in candidate evaluations, never in time, so a plan is lockstep-safe (`02 §9`).

### Multi-core (`sim::par`)

The history, in order:

1. Through M8c, an optional native `parallel` feature ran some loops on `rayon`.
2. P1 removed `rayon`, the `parallel` feature and `tools-par`: a `rayon` region cost 35–180 µs against the economy's 10–50 µs regions. A spin pool with a throttle was built, measured (4–5 ms spikes when a worker was descheduled on the loaded host) and dropped. The economy runs on one thread (`07 D51`).
3. On 2026-11-18 the owner asked for core offloading as a capability that changes speed only, never results (`07 D239`), and approved track P's scope (`07 D169`, `D240`).
4. `sim::par` merged on 2026-11-18 (P-par-0): a map over items whose results the caller applies in item order, `rayon` on native only. `det` hashes the goldens alike on pools of 1, 2, 3 and 8 threads, through `tools --threads N` and `--min-items N`, and the test `threads_change_nothing` (`00-status`). The module is 38 lines (`crates/sim/src/par.rs`).
5. Phase 1 (production and markets per colony) was exact but slower at Huge (a year 4.16 → 4.47 s at six threads), so it was reverted and the work stops there (`07 D261`, `00-status`). The war-storm bench found no worthwhile parallel pass either (`00-status`, P-WS).

The rules that keep results identical (`07 D169`): fixed partitions by id range; per-item writes only, with no atomics or shared accumulators; sums in the serial order; creation and removal applied serially in id order; no RNG in a parallel body.

### Saves and replay

| Aspect | Rule | Source |
|---|---|---|
| Format | `postcard` serialisation of header, tables and rings; columns written as raw bytes | `02 §13` |
| Compression | `zstd` level 3 natively; pure-Rust `miniz_oxide` deflate on the web | `02 §13` |
| Storage | native data directory with an autosave ring of 5; web IndexedDB with `navigator.storage.persist()`, plus export and import | `02 §13` |
| Tables by name | every table dump names its table and columns (generated by `table!` and `group!`); restore matches by name; a column the save predates takes its default or the value in `serialize::DEFAULTS` (the migration as data); a table the save predates starts empty | `07 D50` |
| Registry widths | a row saved narrower than the content is widened with defaults; a wider row refuses the load | `07 D50` |
| Mods | the header carries each enabled mod's id, version and content hash in load order; a different set warns | `07 D52` |
| Version | save version 13 at P2's foundation; 38, 39 and 40 across the 2026-11-23 merges (T1, M17d, T2's first slice) | `07 D50`, `00-status` |

**Replay.** Each save carries the player's `(apply_tick, Command)` log. AI commands are not logged, because the AI re-plans deterministically. `tools replay` recreates the world and compares yearly hashes (`02 §14`).

**Tests** (`02 §14`): unit tests per system on tiny worlds; market invariants as `proptest` properties; the RTS-against-TB parity test (360 ticks against 12 turns, identical hash); save, load and replay hash checks; smoke tests native and web (new game, year 1 at speed 5, save, reload, end a turn, turn report).

### The CPU budget

Reference machine: the test host, a virtual machine with 8 logical cores of a Ryzen 5 5500 and 20 GB, Windows Server 2025, release build (`07 D32`, `08 §1`). "Web" is the same machine in Chrome, single-threaded.

**Design budgets** (`08 §1`):

| Situation | Native | Web |
|---|---|---|
| Paused, no input | 0% CPU (no wake-ups) | 0% |
| Speed 3 (4 ticks/s), Medium, year 150 | ≤ 3% of one core | ≤ 8% |
| Speed 5 throughput, Medium year 150 | ≥ 1,500 ticks/s | ≥ 300 ticks/s |
| Daily tick, Medium year 150, whole economy included | ≤ 0.3 ms median, p99 ≤ 3× | ≤ 1.2 ms |
| Economy per tick (≈ 320 markets, ≈ 1,500 cohorts) | ≤ 0.15 ms | ≤ 0.6 ms |
| AI planning, one empire | ≤ 1 ms | ≤ 3 ms |
| Worst single tick, Huge (1,000 systems, 24 empires) year 300 | ≤ 12 ms and a game year ≤ 3 s, judged at the design scale (≈ 450–1,000 colonies) | ≤ 15 ms |
| Owner's ceiling: one late-game year at speed 5 on the stress profile | ≤ 15 s | none |
| Map frame CPU, Huge, all layers | ≤ 1.5 ms | ≤ 4 ms |
| UI frame, busiest screen | ≤ 2 ms | ≤ 5 ms |
| Tactical step, 100 against 100 ships, 30 Hz | ≤ 0.5 ms | ≤ 1.5 ms |
| Large-action step, 2,000 ships, 30 Hz | ≤ 1.5 ms | ≤ 4 ms |
| Ground battle day (91 hexes, 20 brigades) | ≤ 25 µs | none |
| Flight view frame data | ≤ 0.3 ms | none |
| Empty shell resident memory | ≤ 100 MB | none |
| Resident memory, Huge year 300 | ≤ 150 MB | ≤ 250 MB |
| Save, Huge | ≤ 300 ms | ≤ 1 s |
| Cold start to main menu | ≤ 1 s | ≤ 3 s after download |

The GPU rows are on [Presentation and UI](Design-Presentation-and-ui).

**Re-baselined bench lines.** Since P1, the economy row, the full tick and M9's year-150 rows take as their 1× line the single-thread median of three host runs plus 10% (the throughput floor: the median over 1.1). CI fails at 2× (below half for throughput). Host background load is accepted and judged on medians (`07 D51`). The lines recorded in `08 §1`, at calibration 1.0:

| Row | Original line | Lines since (decision) |
|---|---|---|
| Economy systems, 320 colonies | 120 µs | 311 (`D51`), 392 (`D57`) |
| Full daily tick of that world | 300 µs | 574 (`D51`), 780 (`D57`) |
| Dispatcher, steady state | 80 µs (M5) | 115 (`D54`), 195 (`D57`) |
| Dispatcher, worst case | 600 µs fixed | 441 (`D57`) |
| M9 Medium year 150 tick | 300 µs | 882 (`D51`), 986 (`D82`), 1,145 (`D57`), 1,345 (`D105`), 1,546 (`D108`), 1,505 (`D151`) |
| M9 speed-5 throughput floor | 1,500 ticks/s | 1,063, 944, 794, 722, 630, 628 ticks/s (same decisions) |

The AI plan's CI tripwire is 250 µs × calibration, judged on the heaviest empire's median of nine timings at Medium year 150 (`07 D55`, `D110`).

**Huge.** `07 D239` judges the Huge target at the design scale (≈ 450–1,000 colonies at year 300); the AI-only run of about 3,000 colonies is a tracked stress benchmark, not a gate. On 2026-11-22 the owner accepted the design-scale baseline of about 3.0 s a year and a 9.7 ms worst day (731 colonies) after a bisect and a profile found no cut worth 0.2 s a year or 1.5 ms (`05 M14`). The web line is measured at 409 ticks/s against 300 (`05 M14`).

**Scaling targets** (`08 §6`): Medium 400 systems, 8 AIs, daily tick ≤ 0.3 ms, AI plan ≤ 1 ms; Large 700 systems, 16 AIs, ≤ 0.6 ms, ≤ 1.5 ms; Huge 1,000 systems, 24 AIs, ≤ 1.0 ms, ≤ 2 ms. "If a milestone misses a budget by more than 2× after algorithmic work, the fix is an algorithm, an incremental update or a stagger, never a smaller galaxy." The stress profile for the 1.0 bench is Huge, 24 empires, ≈ 1,200 colonies, ≈ 6,000 cohorts, ≈ 40,000 ships in ≈ 2,500 fleets, 20 commodities and 600 events, at ≤ 15 s a game year (`08 §6`).

### Techniques and banned patterns

Techniques (`08 §3`): daily resolution; the event heap for timers; flag sets instead of scans; struct-of-arrays columns; incremental aggregates; the dirty-key modifier cache; routes computed once at order time; bounded solvers (the trade solve capped at 6 sources per deficit, AI candidate lists capped in data); cohorts instead of individual pops; lazy generation of surfaces and anomalies; the sim decoupled from rendering; instanced map draws uploaded on change; interned strings and text formatted on change.

Banned (`08 §4`, `CLAUDE.md`):

- A system that iterates a whole table every tick to check a per-entity timer or state.
- Monthly work for all empires on one tick.
- Recomputing an aggregate that an incremental update could maintain.
- Pathfinding inside a per-tick loop.
- Nested loops over two entity tables without an index.
- Per-entity `Box`, `Rc`, `Vec` or `String` in sim tables.
- `HashMap` iteration in the sim.
- `std::time`, I/O and `rand::thread_rng` in the sim; threads outside `sim::par`.
- UI code that formats strings or sorts rows every frame.
- Repainting while paused with no input; polling timers in the UI loop.

### Benches and the remote gate

**The gate** (`CLAUDE.md`, `05`): `cargo fmt --all --check`, `clippy -D warnings`, `tools datacheck`, `nextest`, `tools det`, `cargo build --release` and `trunk build --release`, wrapped as `tools gate`.

**The remote gate.** Tests and benches run on the test host, not the dev box (`07 D32`). `scripts/gate-remote.ps1` mirrors the working tree with robocopy, runs `tools gate` there over WinRM and streams the log to `target/gate-remote.log`. Only that log claims a criterion or supplies a bench row.

| Switch | Effect |
|---|---|
| `-Bench` | also runs `tools bench`, then `tools smoke` (native and web) and `tools balance`, failing on a smoke failure or an expansion box (`07 D86`, `D170`) |
| `-Track <name>` | mirrors into a folder on the host, so concurrent tracks gate apart; at most two gates at once; a bench holds a host-wide mutex so its timings stay clean (script header) |

Copied files are stamped with the host's clock so cargo cannot reuse a stale artifact (`07 D107`). `scripts/host-run.ps1` runs ad-hoc host jobs under the same mutex.

**Measurement** (`08 §5`): `tools bench` runs the reference seeds after a calibration loop and prints every row against its 1× line, failing at 2×; `tools huge` measures Huge year 300 (`--colonies N` caps colonies for the design-scale row, `07 D263`); `tools bench --stress` runs the war storm on a cached Huge year-300 world (`07 D262`); criterion micro-benchmarks; Tracy and puffin profiling behind features; an idle test asserting under 50 ms of CPU over 60 paused seconds. GPU rows come from `space-sim --gpu-bench` and are not part of CI enforcement (`08 §1`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Rust, daily tick | The economy resolves every tick at 1/30 of 30-day rates; AI planning is the only staggered periodic job | 2026-11-01 | `00-status` pass 6 |
| Backend measured | `glow` chosen over `wgpu` by the M0 measurement (later reversed for the presentation pass, see [Presentation and UI](Design-Presentation-and-ui)) | 2026-11-01 | `07 D19`, `03 §3` |
| Testing host | Gates and benches run on the test host through `gate-remote.ps1` over WinRM | 2026-11-02 | `07 D32` |
| Architecture for performance and modding | Performance from architecture; per-colony markets kept; AI planning off the tick (M9); three concurrent tracks after M8c, each in its own worktree and host folder | 2026-11-13 | `07 D49` |
| Saves by name | Tables and columns restore by name, `serialize::DEFAULTS` as the migration list, save v13; only the previous version loads until 1.0 | 2026-11-13 | `07 D50` |
| P1 scope | Exact single-thread rework; snapping and lazy settling dropped; `rayon` and the spin pool dropped; bench lines re-baselined to host medians plus 10%, CI at 2×; 15 s per late-game year stays the hard guard. Replaced the earlier native `parallel` feature of `02 §6.4` | 2026-11-13 | `07 D51` |
| AI planning inline | Plans run between ticks with no thread; 250 µs tripwire | 2026-11-13 | `07 D55` |
| Huge judged at the design scale | 3 s a year and 12 ms worst tick at ≈ 450–1,000 colonies; the 3,000-colony run is a stress benchmark. Replaced the earlier ≤ 4 ms and ≤ 2 s Huge lines (`D234` before it) | 2026-11-18 | `07 D239`, `D263` |
| Core offloading | Threads only through `sim::par` (fixed partitions, ordered merges); det must hash the same single- and multi-threaded; the native client is the real target and the web stays single-threaded; a Cores setting (Auto, 1, or 2 up to cores − 1, capped at 8; Auto = min(cores − 1, 6) at 4+ cores) | 2026-11-18 | `07 D240` |
| Phase 1 reverted | Production and markets in parallel were slower at Huge; the harness stays with no caller; the Cores setting waits for a pass that pays | 2026-11-18 | `07 D261`, `DESIGN.md` change log |
| Huge baseline accepted | About 3.0 s a year and a 9.7 ms worst day at 731 colonies | 2026-11-22 | `05 M14` |

## Not yet built or still open

- **The Cores setting** (`07 D240`) is not built; it waits for a parallel pass that pays (`07 D261`). No parallel caller of `sim::par` exists.
- **Further parallel phases** (jobs, growth and happiness; the migration refreshes; per-empire dispatch) are scoped in `07 D169` and not built.
- **Threaded web build:** out of scope; it would need COOP/COEP, `SharedArrayBuffer` and a nightly `-Zbuild-std` build (`07 D169`).
- **The stress profile assertion** at ≤ 15 s per year is M23's (`08 §6`).
- **CI matrix:** the Linux, macOS and wasm32 determinism runs in CI were recorded as not verified in the early milestones, pending the owner's push (`00-status` passes 9–17); later status entries do not say whether they now run.
- **Multiplayer:** lockstep multiplayer is post-1.0 (`SPEC.md §4`).
- **Owner review:** the tracker lists "Whether `CLAUDE.md` should say nine crates (`tactics` added)" as an open question (`DESIGN.md`).

## Discrepancies noted

- **Crate count.** `CLAUDE.md` says "Rust workspace, eight crates (`docs/02 §2`)", and `02 §2` says "Eight crates keep incremental compile times short", while the table in `02 §2` lists nine crates including `tactics`. The workspace `Cargo.toml` (`members = ["crates/*"]`, excluding `crates/lint-fixture`) has nine members: `ai`, `app`, `data`, `render`, `sim`, `tactics`, `tools`, `ui`, `view`. `03 §12`'s workspace layout lists eight crate folders and omits `tactics`. The tracker keeps this as an open question for the owner.
- **Which saves load.** `07 D49` and `D50` say "Old saves: A. Pruned until 1.0 (the previous version only)". Later records describe a range: T1's merge has "save v38 (a v28–v37 save loads)" (`05 T`, `07 D278`), and `docs/release/0.2.0-notes.md` says "this build loads saves back to 0.1.0's". No decision found that widened the rule.
- **The current year-150 line.** `08 §1`'s last recorded re-baseline is 1,505 µs and 628 ticks/s (`07 D151`). Later merge records in `00-status` judge the row against other lines, for example "M9 1,767 µs (1,803 line), 560 ticks/s" (M17b, 2026-11-22) and "Medium y150 is 1,786 µs against its 1,931 line" (`07 D168`). The `DESIGN.md` change log mentions a re-baseline "M-BASE" for a 461-colony world. Which line is current is not recorded in `08 §1`.
- **Determinism tests and `parallel`.** `02 §6.3` and `§14` still say determinism tests run "with and without the `parallel` feature", and `03 §6` says `tools det` runs "with and without `parallel`", while `02 §6.4` (P1 note) and `07 D51` say the feature is removed and `07 D240` describes det across thread counts through `sim::par`.

## Sources

- SPEC.md, scope tiers
- CLAUDE.md
- 02 Architecture: crate layout, simulation core, determinism, threading, time controllers, save and load, replay and testing
- 03 Tech stack: tools crate, CI
- 04 Blockers and risks
- 05 Milestones (P1, P2, M14)
- 08 CPU budget: budgets, banned patterns, measurement, scaling targets
- 07 Decisions: `D32`, `D49`, `D50`, `D51`, `D55`, `D110`, `D169`, `D239`, `D240`, `D263`
- 00 Status
- DESIGN tracker
- `Cargo.toml`, `crates/sim/src/par.rs`, `scripts/gate-remote.ps1`

## Related pages

- [Performance: goals, decisions and caveats](Engineering-Performance)
- [Testing and benchmarking](Engineering-Testing-and-benchmarking)
- [Overview](Overview)
- [Time and modes](Design-Time-and-modes)
- [Modding and data](Design-Modding-and-data)
- [Presentation and UI](Design-Presentation-and-ui)
- [Roadmap and status](Design-Roadmap-and-status)
- [Economy and resources](Design-Economy-and-resources)
- [Trade and logistics](Design-Trade-and-logistics)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [AI and victory](Design-Ai-and-victory)
- [Combat](Design-Combat)
