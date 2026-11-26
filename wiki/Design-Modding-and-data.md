# Modding and data

**Status:** built, as of 2026-11-23. Data registries and saves by name in P2 step 1, the mod system and schema validation in P2 step 3, web zip import in M13; WebAssembly code mods are designed for 1.0.

All game content (commodities, jobs, buildings, techs, hulls, events, AI weights and the rest) is JSON validated against schemas and interned to dense indices at load, so the engine names almost no content in code. A mod is a folder of JSON that adds, replaces, patches or removes records by id; the merged set is validated as one, and saves record which mods were loaded. The goal, set by the owner, is a game as moddable as its reference games, with less code and no lost function (`07 D49`).

## How it works

### Content as JSON

- Content lives in `data/content/`, one file per kind (`02 §15`, `06`). The files on disk on 2026-11-23: `ai.json`, `bodies.json`, `buildings.json`, `colonies.json`, `commodities.json`, `constructions.json`, `districts.json`, `economy.json`, `exploration.json`, `galaxy.json`, `government.json`, `hulls.json`, `jobs.json`, `megastructures.json`, `narrative.json`, `persona.json`, `policies.json`, `starbases.json`, `stars.json`, `strata.json`, `surface.json`, `techs.json`, `traditions.json`, plus the `events/` and `strings/` folders.
- Ids are `snake_case` strings unique within their kind. Cross-references use ids and resolve to indices at load (`06`).
- **All rates in content are per 30 days** (outputs, inputs, upkeep, baskets, decay). The registry divides them by 30 when it builds the daily tables (`06`).
- Every numeric modifier goes through the modifier system with a key from the key registry (`06 §10`). Text lives in `strings/en.json` keyed `kind.id.field`, with placeholders `{planet}`, `{empire}`, `{leader}`, `{n}` (`06 §13`).
- Content is embedded in the binary with `include_bytes!` for release (`02 §15`).
- `commodities.json` holds 17 commodities on 2026-11-23 (the E1 resource model; see [Economy and resources](Design-Economy-and-resources)).

### Schemas and validation

- `serde` structs with `schemars` derives in the `data` crate are the single source of truth. `tools datacheck` emits `data/schema/*.schema.json`, which give editor autocompletion and serve as the modders' documentation (`03 §7`, `07 D53`).
- **`data::schema`** is a small validator in `data` (about 150 lines) covering only the keywords schemars emits. It runs first in `Content::parse`, so every load is checked the same way: base or modded, native or web, the app, the tests and datacheck. The `jsonschema` crate was rejected because it would have checked datacheck only and pulled a large dependency tree into `tools` (`07 D53`).
- Twenty single-field rules moved from hand-written checks onto `#[schemars(..)]` attributes: weights, build days, speeds and sizes, elasticity, the trade band and days, rings, the surface's ring counts, starbase sizes, survey levels, tech tiers, weapon-slot counts and the stance registry's size. `check` keeps references and cross-field and cross-file rules (`07 D53`).
- **Errors** name the file, the path with the record's id, and the mods that touched the file (`07 D53`).

**What `datacheck` enforces** (`06 §16`, abridged):

| Area | Rule |
|---|---|
| References | unknown modifier keys, unknown ids in any reference, duplicate ids fail |
| Registries | every registry between 1 and its cap; each commodity role on exactly one commodity; one crew job; one `constructions.json` row per construction kind |
| Starting layout | every starting district on an eligible hex, adjacency 0, every district served |
| Techs | every tech reachable from tier 0; unreferenced unlocks warn |
| Economy | a commodity with a consumer but no producer is an error; one with neither warns |
| Events | events without a trigger, options without effects and without `defaultOption`, missing strings fail |
| Ships | hulls whose slots exceed the auto-design's power budget warn |
| AI | personality weights outside [0.25, 4] fail |

### Registries with caps

The hard-coded kinds became data registries (`12 §3`, `07 D50`): commodities, strata, job kinds, construction kinds, building kinds, district kinds, hull classes, fleet stances, modifier keys, deep factions and gear slots. At load each registry assigns dense indices in load order; the sim uses the indices and content refers by string id. Rules with behaviour attach to kinds through flags in the data (for example a construction kind's `drawsFrom`), not through `match` on a constant.

**Caps** are validation limits checked by `datacheck` (`07 D50`):

| Registry | Cap |
|---|---|
| Commodities | 32 |
| Strata | 8 |
| Jobs | 32 (the fill's job mask is a `u32`) |
| Districts | 16 (a colony kind's district mask is a `u16`) |
| Policies | 16 |
| Buildings | 126 (the hexes' building bit) |
| Hulls and stances | 255 |

Table rows are as wide as the content's counts, not the caps: the first build sized rows to the caps and the economy row rose 13–15% on the host, so a registry-backed column became a `Row<T, W>` whose width is fixed per world (`07 D50`).

**Content that fills engine roles** (`06`, `07 D50`, `D54`):

- a commodity's `role` (`food`, `goods`, `luxury`, `fuel`, `alloy`), each role on exactly one commodity;
- a job's `crew: true` (the one starbase crew job);
- `strata.json`, lowest stratum first, each `{ id, income, happinessWeight, extraLuxury?, basket }`;
- `constructions.json`, one row per engine construction kind (`ship`, `station`, `found`, `upgrade`, `district`, `building`, `habitat`, `clear`, `army`, `corridor`, `gateway`, `outpost`) with `drawsFrom`, `yardRate`, `yardSlot`, `demandFloor`, `lapsesOnConquest`;
- each `colonies.json` kind's `role` (`surface`, `domed`, `habitat`, `starbase`), a designation's `home: true`, `economy.json`'s `governor.alloyBuilding`;
- `ai.json`'s producer table, warship order and input, admin buildings, idle stance and pact settings, checked by `ai::parse`.

What still names ids in code: the dev staging helpers for the bench and the screenshots, and tests (`07 D54`).

### The event and effect grammar

Events are compiled at load into an enum tree interpreted over `(world, scope)`, with no closures or boxed trait objects per event; trigger buckets index events by trigger kind (`02 §10`). The grammar (`06 §11`):

- **Triggers:** `on_survey`, `on_project_complete`, `on_colonize`, `on_hex_survey`, `on_daily_mtth` (with `mtthDays`, rolled with `p_day = 1 − 2^(−1/mtthDays)`), `on_contact`, `on_tech`, `on_era`, `on_leader_death`, `on_battle_end`, `on_fired`. No trigger may depend on which clock is active.
- **Conditions:** `all`, `any`, `not`, `flag`, `flagGlobal`, `stat` comparisons, `has` (tech, building, district, civic, tradition, perk, treaty, trait), `ethicsPole`, `era`, `year`, scope switches, `random`, and `pred` (a Rust predicate registered by id, the escape hatch).
- **Effects:** `ledger`, `stock`, `modifier`, `flag`, `event` (with `delayDays`), `situation`, `spawnFleet`, `reveal`, `tech`, `ethics`, `leader`, `hex`, `opinion`, scope switches and `weighted` branches.

Events with options interrupt unless hidden; turn-based mode defers them to the turn report. Unanswered events take their default option after `timeoutDays` in both clocks (`02 §10`). Situations, leaders, archaeology and species are data in `narrative.json` (`06 §12`, `07 D85`, `D97`).

### Folder mods with patches

**Layout** (`12 §2`):

```
mods/<id>/mod.json           { id, name, version, gameVersion, dependsOn[], loadAfter[], files[] }
mods/<id>/content/...        same tree as data/content/
mods/<id>/strings/<lang>.json
mods/<id>/assets/
mods/<id>/CREDITS.md
```

`files` lists the mod's files explicitly; it is not a glob. The same manifest reads from a folder and from a web zip, and the content hash is over exactly those files (`07 D52`).

**Operations per record, keyed by `id`** (`12 §2`, `06 §14`):

| Record in the mod | Effect |
|---|---|
| A new id | adds a record |
| `"$replace": true` | replaces the record |
| Any other object for an existing id | merge patch with RFC 7386 semantics: `null` removes a field, arrays replace |
| `"$remove": true` | deletes the record; refused while any file still names the id as a string value |

Records keyed by `id` merge in any array of records, not only at a file's top level, so a mod adds a hull inside `hulls.json`'s `hulls` without restating the file. An array without ids replaces. The `$remove` check is textual, so an unrelated field holding the same string also counts (`07 D52`).

**Load order** is resolved from the dependencies, then `loadAfter`, then the mod id. It is a pure function of the enabled set, so every machine merges identically (`12 §2`).

**Layering, not a second loader.** `data::mods::layer` merges a mod's JSON into the base files' JSON in load order before parsing, so the sim, the schemas and `check` see one content set (`07 D52`).

**One installed set per process.** `data::base()` returns the embedded files with the installed mods, installed before anything reads the content. The native app takes `--mod <dir>` (repeatable). There is no enable list or mod menu in MVP. `ai.json` is among the layered files, so a mod can retune the AI (`07 D52`).

**Web.** The web build imports a mod as a zip through a file picker. `Mod::from_zip` reads a zipped mod folder as the folder, with the same hash. Zips are stored in IndexedDB as `mod:<id>` and installed at the next start, because the content is fixed per process (`07 D95`, `05 M13`). A failed load names the mod difference (`05 M13`).

**Assets.** The sprite manifest maps hull, station and structure ids to atlas rectangles, so a mod adds a sprite by adding a row (`07 D76`). A body class names its planet look and atmosphere in `bodies.json`, so a mod's class picks a look (`07 D83`).

**Tooling.** `tools mod new <id>` scaffolds a mod. `tools mod check <dir>` layers the mod and the sibling folders it depends on, runs datacheck, plays a Tiny game for 400 days, saves, checks the header's mod list, reloads with the same hash and plays 100 more days. The gate runs it on `mods/example` (`07 D52`). Mods carry their own `CREDITS.md`, and the credits screen lists each installed mod's file (`12 §2`).

**The example mod.** `mods/example` ("Example: the spice trade", version 0.1.0) patches `commodities.json`, `strata.json`, `jobs.json`, `hulls.json`, `events/anomalies.json` and `techs.json` and adds strings. Its commodity appears in the markets matrix with no panel code (`07 D54`).

**Generic panels.** The screens build their rows from the registries (the markets matrix over the commodities, research over the techs, buildings, jobs and designs over their content), and `ui::rows` is the shared row list, so a new content kind appears without new panel code (`07 D54`).

### Saves keyed by name

- Every table dump names its table and each column, generated by `table!` and `group!`. Restore finds both by name (`07 D50`).
- A column the save predates takes its type's default, or the value `serialize::DEFAULTS` lists for it: the migration as data. A table the save predates starts empty.
- A registry row saved narrower than the content (the content gained entries) is widened with defaults. A row wider than the content refuses the load. This is why a save made with a mod's extra commodity is refused once the mod is removed (`05 M13`).
- The save header carries `(id, version, content hash)` per enabled mod in load order (save version 16). A load with a different set warns in the status line with both differences and loads anyway (`07 D52`).
- Old saves were pruned for MVP: the v2–v11 migration chain, its fixtures and the repairs only it called were deleted (`07 D50`). From 1.0, every release keeps its migrations for good (`12 §1`).

### WebAssembly code mods (designed, 1.0)

- **MVP:** everything moddable is data plus the event and effect grammar: events, conditions, effects, modifiers, AI weights (`12 §1`).
- **1.0:** sandboxed, deterministic WebAssembly code mods with fuel metering, hooked at fixed points (the planning day, events, UI panels), the same on native and web (`12 §1`, `07 D49`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| Mod format | Folders with patches: a manifest with load order; add, replace, merge-patch or remove by id; strings and assets by path; the merged set validated; mod hashes in saves | 2026-11-13 | `07 D49`, `12 §1` |
| Code mods | Data and the grammar in MVP; sandboxed deterministic WebAssembly in 1.0. Replaced `SPEC.md §4`'s placement of scripted mods (WASM or Lua) in post-1.0 | 2026-11-13 | `07 D49` |
| Core kinds | Data registries with caps (commodities up to 32, strata up to 8) | 2026-11-13 | `07 D49` |
| Old saves | Pruned until 1.0 (the previous version only); from 1.0 every release keeps its migrations | 2026-11-13 | `07 D49`, `12 §1` |
| Rows as wide as the content | Caps are validation limits; table rows follow the content's counts, since cap-wide rows cost the economy row 13–15%. Replaced `12 §1`'s "arrays sized to the cap" | 2026-11-13 | `07 D50` |
| Saves by name | Tables and columns restore by name; `serialize::DEFAULTS` is the migration list; save version 13 | 2026-11-13 | `07 D50` |
| Layering | A mod's JSON merges into the base JSON before parsing; ids merge in any array; `files` listed, not globbed; one installed set per process; mod list in the save header (v16) | 2026-11-14 | `07 D52` |
| `$remove` | Built as `"$remove": true`. Replaced `06 §14`'s earlier `"$delete": true` | 2026-11-14 | `06 §14`, `07 D52` |
| Schema validation | A small in-tree validator run on every load; twenty single-field rules moved onto the types | 2026-11-14 | `07 D53` |
| Hot reload and coverage | Cut | 2026-11-14 | `07 D54` |
| Web zip import | Moved from P2 to M13; built with IndexedDB storage and install at next start | 2026-11-14 | `07 D54`, `D95` |

## Not yet built or still open

- **WebAssembly code mods** with fuel metering and fixed hooks (1.0, `12 §1`).
- **Multiplayer's refusal** to start with a different mod set (1.0, `12 §2`).
- **Hot reload** of content in dev builds: cut (`07 D54`).
- **An in-game mod menu or enable list:** none in MVP (`07 D52`). The 1.0 milestone M21 lists "pack manifest, load order, override rules, in-game pack manager, mod validation UI" (`05`, 1.0 table).
- **Mod removal screen** on the web: none until M14's settings (`DESIGN.md`, M13 call). No later record of it was found.
- **Gear slots** as a registry waited for C1's fitting (`12 §3` built note); no later record of the registry was found.
- **Content still named in code:** the dev staging helpers for the bench and screenshots, and tests (`07 D54`).
- **Species and empire skins** (panel frames as moddable data) are part of the presentation pass, designed only (see [Presentation and UI](Design-Presentation-and-ui)).

## Discrepancies noted

- **Commodity count in the schema doc.** `06 §1` says "Exactly the 10 ids in `01 §7.2` for MVP", while `commodities.json` holds 17 records and `DESIGN.md` describes 17 commodities from E1. `06 §1`'s P2 note raises the cap to 32 but does not restate the count.
- **M21's scope against P2.** The 1.0 table in `05` describes M21 "Modding data packs" as "pack manifest, load order, override rules, in-game pack manager, mod validation UI" at 1.2k lines, while `07 D52` records the manifest, load order and override rules as built in P2. It is not recorded whether M21 is reduced to the pack manager and validation UI.
- **Which saves load.** `07 D49` and `D50` keep only the previous save version loading until 1.0; T1's record says "a v28–v37 save loads" (`07 D278`), and the 0.2.0 notes say the build "loads saves back to 0.1.0's". No decision found that widened the rule.
- **Migrations dropped.** `12 §1` says "The v2–v10 migration chain and its fixtures are dropped, about 1.2k lines"; `07 D50` records the v2–v11 chain and "nine fixtures" deleted. The later record is the more specific.

## Sources

- SPEC.md, scope tiers
- 02 Architecture: content pipeline, save and load, event engine
- 03 Tech stack: content tooling
- 05 Milestones (P2, M13, the 1.0 table)
- 06 Data schemas: modifier keys, event, pack manifest
- 12 Modding: the owner's calls, the mod system, registries, simplicity
- 07 Decisions: `D49`, `D50`, `D52`, `D53`, `D54`, `D76`, `D83`, `D95`
- DESIGN tracker
- `data/content/`, `data/schema/`, `mods/example/mod.json`

## Related pages

- [Overview](Overview)
- [Architecture and performance](Design-Architecture-and-performance)
- [Presentation and UI](Design-Presentation-and-ui)
- [Roadmap and status](Design-Roadmap-and-status)
- [Economy and resources](Design-Economy-and-resources)
- [Events and crises](Design-Events-and-crises)
- [Ships and fleets](Design-Ships-and-fleets)
- [AI and victory](Design-Ai-and-victory)
