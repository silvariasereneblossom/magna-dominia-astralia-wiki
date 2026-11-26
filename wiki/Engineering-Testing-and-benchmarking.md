# Testing and benchmarking

**Status:** current as of 2026-11-24 (main at `f765ea4`), read from the scripts, the workflow files and `crates/tools/src/main.rs`. Whether the GitHub workflows currently pass on GitHub was not checked for this page.

Every claim that a milestone works, and every performance figure, comes from one automated run on one machine: the **remote gate** on the test host. The gate proves the code is correct and deterministic; the bench, smoke and balance runs that follow it prove it is fast enough, starts and plays, and still produces a healthy game. A track's work merges to `main` only on a passing run, and the run's result is written into `00-status`. What the figures mean is on [Performance: goals, decisions and caveats](Engineering-Performance).

## The rules

- **Tests and benches run on the host, not on the development PC** (the owner's rule, 2026-11-02; `07 D32`; `CLAUDE.md`). A local `cargo gate` is only a smoke check.
- **Only the remote log claims a criterion or supplies a bench row** (`CLAUDE.md`). The log is `target/gate-remote.log`.
- **No track merges an expansion regression** (`07 D86`): a `-Bench` run fails if the smoke run or a judged balance box fails.
- **Golden hashes change only in a `sim-change:` commit** (`02 §6.3`), and the merge record lists the new hashes.

## The layers

| Layer | Command | What it proves | Where it runs |
|---|---|---|---|
| Format and lints | `cargo fmt --all --check`; `cargo clippy --workspace --all-targets -- -D warnings` | style; the determinism bans (`std::time`, I/O, `rand`, `HashMap` iteration, unstable sorts, `std` float functions) hold in `sim` and `ai` | gate, CI |
| Lint fixture | clippy on `crates/lint-fixture`, which must **fail** | the bans are live, not silently disabled | CI |
| Content | `tools datacheck`; `tools mod check mods/example`; `tools shaders` | the content JSON, the example mod and the shaders are valid | gate (`datacheck` also in CI) |
| Tests | `cargo nextest run --workspace` | per-system behaviour on small worlds; 323 tests at the latest merge | gate, CI |
| Determinism | `tools det` | the same seed gives the same world, through a save and through a replay | gate, CI, and in a browser in CI |
| Builds | `cargo build --release`; `trunk build --release` | the native and web builds compile | gate, CI |
| Bench | `tools bench` | every budget row is inside 2× its line | `-Bench`, CI |
| Smoke | `tools smoke` | the release app starts, plays, saves and reloads, natively and in a browser | `-Bench` |
| Balance | `tools balance` | AI-only games still expand and stay fed, within the judged boxes | `-Bench`, nightly CI |
| Scale | `tools huge`; `tools bench --stress` | the Huge year-300 rows and the war storm | host, on request; not in the gate |

### Tests

Integration tests live in `crates/sim/tests/`, one file per area (`economy`, `colony`, `military`, `diplomacy`, `hyper`, `starbase`, `politics`, `rebels`, `tactical`, `det`, `loadstep` and others), with more in `crates/app/tests/` (`engine`, `frames`, `idle`). Market invariants are `proptest` properties, with their saved failing cases in `economy.proptest-regressions` (`02 §14`). The idle test asserts that a paused app uses under 50 ms of CPU over 60 seconds (`08 §5`). The parity test runs 360 real-time ticks against 12 turns and requires the same hash (`02 §14`).

### Determinism (`tools det`)

For each of three golden seeds, `det` runs the world twice, saves and reloads it, and requires every hash to equal the committed golden. For seed 1 it also replays the command log and requires the replay to match the live run. It then runs the battle fixture and compares its per-tick loss log with `tests/det/battle-log.txt`. The test `threads_change_nothing` hashes each golden alike on pools of 1, 2, 3 and 8 threads (`07 D261`). In CI the same determinism test also runs compiled to WebAssembly in headless Chromium, so native and web are held to one hash.

### The gate (`tools gate`)

`tools gate` runs nine steps in order and stops at the first failure: format, clippy, `datacheck`, the example mod's check, shaders, nextest, `det`, the release build and the web build. `cargo gate` is its alias.

## The remote gate

`scripts/gate-remote.ps1` is the only route to a claimed result.

1. **Mirror.** `robocopy /MIR` copies the working tree to the host, leaving out `target`, `dist`, `.git`, `node_modules` and logs.
2. **Stamp.** Every file it copied is given the host's clock time, so cargo cannot reuse an artifact built from the previous mirror (`07 D107`).
3. **Run.** Over WinRM, the host runs `tools gate`. With `-Bench` it then runs `tools bench`, `tools smoke` and `tools balance`, all release builds.
4. **Log.** Output streams to `target/gate-remote.log`, and the script exits with the remote status. A passing run ends `GATE EXIT 0`.

| Switch | Effect |
|---|---|
| `-Bench` | adds the bench, smoke and balance runs |
| `-Track <name>` | mirrors into a folder on the host, so concurrent tracks do not overwrite each other |

**Keeping timings clean.** At most two gates run at once (the owner, 2026-11-22). A bench, smoke or balance step takes a host-wide mutex (`Global\space-sim-bench`) and waits until no other track's gate is running; a gate does not start while a bench holds it. Waits are capped at three hours.

**Ad hoc runs.** `scripts/host-run.ps1` runs any one tool or build on the host under the same mutex, in the track's own scratch folder, with the run's memory and time capped in a job object. It writes a running marker so gates count it (2026-11-24).

## The bench

`tools bench` prints the `08 §1` rows as a table and exits non-zero if any row is over its limit.

- **Calibration.** Before measuring, the bench hashes 64 MiB with FNV-1a and times it. The reference is 60 ms. The scale is that time over 60 ms, never below 1.0; every line is multiplied by the scale, so a busy or slower machine is given proportionally more room. Recent host runs read 1.19 to 1.27.
- **Rows.** Empty tick (p50 and p99 over 200 samples of 1,000 ticks); 20 science ships exploring; galaxy generation at Medium and Huge (median of five); the economy systems and the full tick of a 320-colony world; the trade dispatcher, steady and worst case; Medium year 150 (tick, throughput, the heaviest AI plan); the ground battle day, tactical and large-action steps, route search, flight view data, and modifier refold.
- **Lines.** Each row has a 1× line and fails at 2×. Budget rows use the `08 §1` budget. The economy, full tick, dispatcher and year-150 rows use measured lines: the single-thread median of three host runs plus 10% (`07 D51`). The lines are constants in `crates/tools/src/main.rs` (for example `ECON_LINE_US` 392, `FULL_LINE_US` 780, `DISPATCH_LINE_US` 195).
- **Re-baselining.** A line is raised only by a recorded decision, when the world the row measures has grown and the cost per colony is flat (`DESIGN.md` change log, 2026-11-20). A merge whose total rose is judged per colony.

**Smoke.** `tools smoke` drives the release app through its own actions: new game, year 1 at speed 5, save, relaunch, load, switch to turn-based, end a turn, and check the turn report shows. It runs natively and in headless Chrome against the gate's `dist`, and prints two throughput rows, the year at speed 5 natively and in WebAssembly (`07 D170`).

**Balance.** `tools balance` runs AI-only games on the reference seeds, writes per-year CSVs and judges boxes drawn from `01 §18`, such as the First Jump by year 3 and colonies outside home by years 20 and 50; the other targets print as probes (`07 D147`). Some boxes are judged on an eight-seed mean (`DESIGN.md` change log, 2026-11-24).

**Scale runs.** `tools huge` runs Huge to year 300 (`--colonies N` holds the world at the design scale, `07 D263`). `tools bench --stress` runs the war storm on a Huge year-300 world cached on the host (`07 D262`). Both report against lines but are not part of the gate.

## GitHub Actions

| Workflow | Trigger | What it runs |
|---|---|---|
| `ci.yml`, native job | push to `main` and pull requests, except changes only to `docs/**` and `*.md`; manual | format, clippy, the lint fixture, `datacheck`, nextest, `det`, release build, bench (table in the job summary), binary size ≤ 15 MB, `cargo deny` on Linux. Linux only on push and pull request; Linux, Windows and macOS when started by hand |
| `ci.yml`, wasm job | same | `trunk build --release`, size ≤ 8 MB raw and ≤ 2.5 MB brotli, the determinism test in headless Chromium, then a Cloudflare Pages deploy on pushes to `main` |
| `nightly.yml` | 03:17 UTC daily; manual | `tools balance --years 200`, CSVs uploaded |
| `release.yml` | tags `v*` | release builds staged with licence files and attached to a GitHub Release |
| `sync-wiki.yml` | pushes to `main` touching `docs/wiki/**` | mirrors this wiki to the GitHub Wiki |

## How a result is recorded

A track gates its own branch with `-Bench -Track <name>`. The spec session verifies the log, merges, and adds a line to `00-status` naming the commit, the test count, `GATE EXIT 0`, any re-blessed golden hashes, the save version, the headline bench rows and the lines of code added. Design consequences go to the `DESIGN.md` change log.

## Not yet built or still open

- **Browser smoke in CI.** The wasm job's Playwright step is a placeholder that prints a TODO; the browser smoke runs only on the host through `tools smoke`.
- **Cross-platform CI by default.** Windows and macOS run only when the workflow is started by hand.
- **GPU rows.** Frame times and memory by backend come from `space-sim --gpu-bench` on the host's GTX 1060 and are not enforced anywhere (`08 §1`).
- **The stress assertion.** The 15 s stress profile is M23's; the war storm "returns false over its lines but is not in the gate" (`DESIGN.md` change log, 2026-11-19).
- **Profilers.** `08 §5` names Tracy and puffin behind features and criterion micro-benchmarks with a 10% threshold. `crates/sim/benches/step.rs` exists; whether the profilers are wired and the threshold enforced was not checked for this page.
- **Other tools.** `tools` also has `run`, `replay`, `soak`, `loadcheck`, `stuck`, `supply`, `fieldcost`, `pacts`, `ai` and `calibrate` subcommands. They are diagnostics outside the gate and are not described here.

## Discrepancies noted

- **Gate steps.** `CLAUDE.md` and `03 §6` list seven steps; `tools gate` runs nine (the example mod's check and shaders were added).
- **CI matrix.** `03 §8` says jobs run on "every push/PR, matrix over Linux x86-64, Windows x86-64, macOS AArch64"; `ci.yml` runs Linux alone unless started by hand.
- **Playwright.** `03 §8` lists a Playwright smoke on the built page; in `ci.yml` it is a TODO.
- **Web throughput in CI.** `08 §5` and `03 §8` say the wasm test measures throughput; in `ci.yml` it checks the determinism hash only. Web throughput comes from the host's smoke run.
- **`det` and `parallel`.** `03 §6` says `det` runs "with and without `parallel`"; that feature was removed in P1 (`07 D51`), and thread counts are covered by `threads_change_nothing`.
- **`tools bench` arguments.** `03 §6` documents `--size` and `--years`; the command as built takes no size argument, and Huge is `tools huge`.
- **Balance seeds.** `03 §6` says three reference seeds over 200 years; the judged lines now include eight-seed means.

## Sources

- CLAUDE.md
- 03 Tech stack: tools crate, CI
- 08 CPU budget: measurement
- 02 Architecture: determinism, replay and testing
- 07 Decisions: `D32`, `D51`, `D86`, `D107`, `D147`, `D170`, `D261`, `D262`, `D263`
- 00 Status
- `scripts/gate-remote.ps1`, `scripts/host-run.ps1`, `crates/tools/src/main.rs`, `crates/tools/src/smoke.rs`, `.github/workflows/ci.yml`, `nightly.yml`, `release.yml`

## Related pages

- [Performance: goals, decisions and caveats](Engineering-Performance)
- [Architecture and performance](Design-Architecture-and-performance)
- [Roadmap and status](Design-Roadmap-and-status)
- [Modding and data](Design-Modding-and-data)
- [Home](Home)
