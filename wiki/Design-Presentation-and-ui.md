# Presentation and UI

**Status:** built for the current look (G1–G4 and M14's UI pass); designed for the immersive presentation pass (`07 D231`, milestones R1–R4, U1–U2, A1, not placed), as of 2026-11-23.

The presentation is everything the player sees and hears: the galaxy and system maps, ships, planets and surfaces, the egui screens, sound and music. The built look pays for quality offline or once (Blender-rendered lit sprites, GPU-generated planets baked on first view) so each frame costs a few texture fetches per pixel, and a paused, idle game draws nothing. On 2026-11-16 the owner asked for a more immersive, release-like look in the spirit of the reference games, spending performance headroom; that pass is scoped and comes after the MVP.

## How it works

### Renderer and backend

- **Shell.** `eframe` provides the window, input, graphics context and frame loop on native and web (`02 §11.1`).
- **Backend.** `glow` (OpenGL 3.3 / WebGL2), chosen by the M0 measurement: 76 MB resident and a 0.30 s cold start, against 165–215 MB and 0.78–1.26 s for `wgpu`, which also failed the idle test (`03 §3`). The `wgpu` implementation was removed on 2026-11-02 at the owner's call (`02 §11.1`).
- **The `Backend` trait** in `render` began with six calls (`upload_instances`, `draw_layer`, `upload_texture`, `set_viewport`, `begin`, `end`). G1 added `bake_background` (`07 D42`) and G3 added `bake_planet` (`07 D83`), so a backend is still one file.
- **WebGL2 limits.** All pipelines use only vertex and instance buffers, uniforms of 16 KiB or less and textures, the intersection of OpenGL 3.3 and WebGL2 (`02 §11.2`, `10 §5`).
- **Repaint on demand.** The UI requests a frame when a new snapshot arrives, on input, and during animations; otherwise no frames are produced. A paused game with no pointer movement renders nothing (`02 §11.1`), proved by `app::frames` (`00-status`, G1).
- **Shaders.** The GLSL sources live in `crates/render/src/shaders`; `tools shaders` validates them for `#version 330 core` and `300 es` in the gate, printing a notice and passing where the validator is missing (the host) (`07 D42`).

### Graphics by element

| Element | As built | Source |
|---|---|---|
| Galaxy background | fBm nebula and star field baked once per galaxy on the GPU into a tileable texture; parallax 0.3 px per world unit of camera travel; every look parameter a field of `Bake` | `07 D42` |
| Stars | Star shader: limb-darkened disc, glow and corona by spectral colour, additive. Glow size `30 + 26 × (blue − ½ red)` pixels, clamped 24–56, ×1.3 for home systems; a colour under 0.5 luminance draws a black disc with a bright ring (the black hole); the corona turns only on High | `07 D42` |
| Territory | The centroid dual of a Delaunay triangulation over the systems plus unowned fill points; borders drawn from an edge distance in the fragment shader; re-meshed only when a claim the player sees changes | `07 D42` |
| Hyperspace layer | One texel per cell on a `TEX` quad, re-uploaded when the month or a corridor changes: clouds violet, storms amber, corridors cyan, the deep darker; fleet routes as lines | `02 §11.2` |
| Ships, stations, structures | Blender Cycles renders each design from straight above into albedo and normal-with-emissive atlases; the shader lights the quad from the system's star (upper left on the galaxy map). Two instances per ship: a `GLYPH` (the class arrowhead below 18 px on screen) and the lit `SPRITE`; LOD decided in the shader. 136 sprites in six lineages | `07 D76`, `00-status` (G2) |
| Planets | A procedural equirectangular texture per body, baked on the GPU on first view from its seed and class, drawn as a lit rotating sphere with a Fresnel atmosphere rim. Sixteen looks (continental, ocean, tropical, arid, desert, savanna, alpine, tundra, arctic, barren, molten, toxic, frozen, gas giant, ice giant, asteroid). One turn takes 40–120 s by seed; Low does not turn | `07 D83` |
| Planet cache | One RGBA8 texture array of 64 layers, each 256×128 with its mip chain: 128 KB a layer, about 10.7 MB in all; LRU by last view; the planet screen has its own 512×256 texture | `07 D83` |
| Sol | The 15 named major bodies use NASA, JPL and USGS public-domain maps at 512×256 WebP (0.17 MB) | `07 D41`, `D83` |
| Hex surface | One egui textured mesh: 12 terrains and 10 features rendered by Blender into one 1024×416 atlas (40 KB lossy WebP); hidden hexes as flat fog | `07 D84` |
| Gigastructures | Builders for the Matrioshka Brain, Dyson Sphere, Ringworld, Star Lifter and Nicoll-Dyson Lance; drawn around the star, 110 units across | `07 D84` |
| Motion | Hyperspace storms blend toward next month by the month's fraction; corridors shimmer; fleets trail six days of route in hyperspace and the last quarter of an in-system leg; a jump in or out gets a 1.6 s glint | `07 D87` |
| Battles | Instanced ships, shots, beams, impacts and particles on a second backend path (C1) | `00-status` (C1) |

**No post-processing** in the built look: glow is drawn into the sprites and the star shader with additive blending, so there is no bloom pass and no off-screen framebuffer (`07 D37`, `10 §2`).

**Presets.** Low, Standard and High (`10 §2`, `00-status` G1). The animation clock advances at 30 frames a second only while the map shows and the game runs. High animates throughout (corona, corridor shimmer); Standard animates only while something moves; Low never animates (`07 D87`).

**Asset budget.** ≤ 3 MB for the download (`03 §10`). To stay inside it, only the 2× sprite atlases ship and the UI box-filters lower sizes at load (`07 D84`); the normal atlas ships at 1×; the 2× albedo atlas ships as lossy WebP (quality 92). After G4 the download is **2.22 MB of 3 MB** (`07 D88`).

**GPU budgets** (`10 §3`, `08 §1`), measured on the host's GTX 1060 3 GB through `space-sim --gpu-bench`:

| Row | Budget native / web | Measured (host GTX 1060) |
|---|---|---|
| Map frame, Huge, all layers, 1080p | ≤ 2 ms / ≤ 4 ms | 0.329 ms p50 (G1); 0.431 ms with G4's weather in the shader |
| System view, 30 bodies and 40 ships, 1080p | ≤ 2 ms / ≤ 4 ms | 0.066 ms (G1); 0.147 ms with G3's lit spheres and sprites |
| Same at 4K (2 pixels per point) | ≤ 2 ms | map 0.952 ms p50, system 0.213 ms |
| Battle frame, 2,000 ships with effects, 1080p | ≤ 4 ms / ≤ 8 ms | 0.071 ms p50 (C2); 0.075 ms as sprites (G2) |
| Texture memory | ≤ 64 MB | 42.5 MB at 1080p, 48.4 MB at 4K (G3, mip chains and full planet cache counted) |
| First planet view | ≤ 5 ms per body / ≤ 15 ms | 0.34–1.40 ms over the 16 looks |
| Map frame CPU, Huge | ≤ 1.5 ms / ≤ 4 ms | not restated |
| UI frame, busiest screen | ≤ 2 ms / ≤ 5 ms | not restated |

Web GPU rows are not measured (`08 §1`).

### Audio

- **Engine:** `kira` 0.12 on both targets; on the web it drives Web Audio through `cpal`. It starts on the first click or key, meeting the browser's gesture gate. Vorbis in Ogg (`07 D88`).
- **Buses:** interface, effects, ambience and music under the master, each a slider on a 40 dB taper, plus a mute. Each sound plays at most every 70 ms. A paused, idle game queues nothing; the audio thread keeps the music going (`07 D88`).
- **Cues:** clicks; settings and planet view open and close; a rejected command errs; completions chime; research, eras, colonies and events notify; stalls, lost fleets, recessions, intercepts and conquests warn; the player's fleets jumping whoosh; weapon fire, hits and explosions in battle (`07 D88`).
- **Effects:** 19 CC0 effects (Kenney packs and OpenGameArt), mono, 32 kHz, 157 KB (`07 D88`).
- **Music as built:** three CC0 beds from OpenGameArt, one per era, each a 56 s loop at 48 kbps mono, 759 KB together. A procedural synthesis (`ui::audio::Synth`) plays where no bed fits: a battle, a crisis, an era without a bed, and the ambience bus always. Music crossfades over 3 s when the mood changes (`07 D88`).
- **Music direction (owner, 2026-11-17):** by mood, leaning orchestral. Exploration, trade and peace get atmospheric beds with orchestral colour, not piano; war, battles, crises and the late eras get orchestral percussion, brass and choir. The procedural synthesis stays as the fallback. The chosen beds will replace the current three under the same budget (about 1 MB of Opus on the web) (`07 D238`).

### The egui UI

- **Architecture.** One `UiStore` holds the latest `SimView`, selection, open screens, pending turn-based commands and per-panel caches keyed by snapshot version, so a frame with an unchanged snapshot touches no sim data. Large tables use `egui_extras::TableBuilder` with uniform-height rows, which skips rows off screen. Tooltips are one overlay fed by the query channel. Screens are tabs in a left rail with the map underneath (`02 §11.5`).
- **Theme.** egui theme through tokens, with Chakra Petch (OFL) as the display face (G1, `00-status`). The tokens gained a spacing scale (2, 4, 8, 16), icon sizes (14, 18, 24) and two commodity tints (`07 D182`).
- **Icons.** One atlas rasterised by `resvg` at start: 15 game-icons.net glyphs (CC BY 3.0, credited) and 26 generated CC0 glyphs (9 for panels and the HUD, 17 for the commodities), 41 in all (`07 D180`). Each commodity names its glyph in `commodities.json` (`icon`), so mods name one too (`07 D181`).
- **Interaction rules.** Every number has a tooltip with its modifier-stack breakdown; every list is sortable and filterable; keyboard: space pauses or ends the turn, 1–5 set speed, F-keys open screens, Esc closes (`01 §16`). Keys are rebindable on the settings screen and saved with the settings (`07 D154`).
- **Notices** (load, save, import) show as 6-second toasts wrapped to the window (`07 D106`). The top bar's buttons wrap to a row of their own at narrow widths (`07 D109`). Empty states were added in M14's UI pass (`07 D153`). An action that cannot be taken is greyed out with the reason (`docs/release/0.1.0-notes.md`).
- **Settings** are saved per user as `key=value` lines: `settings.txt` beside the saves natively, `localStorage` on the web (`07 D88`).

### Screens

From `01 §16`, with the milestone that built each where the sources record it:

| Screen | Contents | Built |
|---|---|---|
| Main menu | new game (galaxy settings, empire creator), load, settings, credits | M14 front end (`07 D97`) |
| Galaxy map | systems, the hyperspace layer, territories, fleets and routes; layers: political, trade (goods in transit), piracy, resources, diplomacy, surveyed | M1, G1, M8c |
| System view | bodies and moons in orbit rings, stations, habitats and domes, fleets, starbase; the primary screen of Era I | M1 onward |
| Planet view | surface hex map; tabs Overview, Build, Market, Expedition | M4a, M4b, M6 |
| Empire | ledger (30-day rolling income and expense), policies, government, civics, factions (1.0), edicts, eras | M4b, M7 |
| Markets | commodities × markets matrix: stock, inbound, price, 30-day trend; flows; shortages; freight utilisation | M5 |
| Research | the tree view: the Kardashev header, the three fields' projects, the goals, search and filters, a card per tech | T4 (built 2026-11-24) |
| Traditions | trees and ascension perks | M7 |
| Ship designer | hull, slots, stats preview, auto-design | M8 |
| Fleet manager | fleets, composition, stance, orders, upgrade, fuel, readiness, supply | M8 |
| Diplomacy | empires, opinion breakdown, treaties, claims, war status | M10 |
| Situation log | anomalies, projects, archaeology, situations, expeditions | M3, M11 |
| Leaders | the "Ldr" tab | M11 (`DESIGN.md`) |
| Captain (persona modes) | persona, reputation, contracts, cargo, purse, holding; rank and assignments | M12a; rank and assignments M16 |
| Battle screen | play, watch, simulate, auto-resolve | C1, C2 |
| Flight view | the persona's fleet in hyperspace | F1 (see [Flight view](Design-Flight-view)) |
| Outliner | planets with alerts, fleets, construction | M4a |
| Notifications | feed with filters; event popups with options | M3 |
| Turn report (TB) | playback scrubber, digest, pending choices | M13 |
| Settings | graphics, audio, autosave, automation defaults, keybinds | G4, M14 |
| Game end | scoreboard and victories | M14 |

### The immersive presentation pass (designed)

The owner's request (2026-11-16): something more immersive, similar to the reference games, spending performance gains on higher-quality, more release-like UX/UI with a strong sci-fi feel (`18`). Her calls (`07 D231`, `18 §0`):

| # | Call | The owner's answer |
|---|---|---|
| 1 | World view | A 3D camera with seamless zoom over the 2D sim |
| 2 | Ships | High-detail 2D sprites in battles, with 3D models on the map |
| 3 | Planets | Full 3D |
| 4 | Backend | `wgpu` (reverses M0's `glow`) |
| 5 | Post-processing | Glow (bloom) on by default (reverses `D37`'s "none") |
| 6 | GPU target | 60 fps at 1440p on the GTX 1060 3 GB |
| 7 | Characters | Animated portraits rendered from 3D, with a plugin hook for live 3D characters |
| 8 | Download | The web streams HD packs |
| 9 | Design language | Command deck |
| 10 | Skins | Frames as moddable data |
| 11 | UI technology | Keep `egui` with a skin layer |
| 12 | Layout | "Rails and outliner" for the empire (28 px status strip, 44 px icon rail with the signal rail, collapsible outliner on the right, selection dossier at bottom-left); the captain's console (ledger header, four mode tabs, dense sortable tables, an inspector, a course strip); type: Chakra Petch for display, IBM Plex Sans and Condensed for body, Plex Mono for numbers; motion: finite pulses that settle, nothing animating while paused and idle; an Okabe–Ito-based status palette with sign and icon redundancy; species skins pass a contrast check in `datacheck` |
| 13 | Timing | Finish the MVP, then the presentation pass |
| 14 | Where | Everything on the test VM (the test host, the GTX 1060 3 GB) |
| 15 | Install | Blender 5.2.3 LTS on the test VM: done 2026-11-16, library capped at 40 GB |

The "command deck" language (`18 §3.5`): dark, slightly translucent glass panels over a blurred backdrop, thin luminous edges and cut corners, two accent colours per empire, panels sliding in under 150 ms, numbers that tick when they change. Readability stays first: UI scale, text size and colour-blind palettes stay, and the 1.0 accessibility pass keeps its place. The proposal drops the 3 MB web budget for a first download of ≤ 25 MB plus streamed HD packs; native ships everything (`18 §3.4`).

**Proposed milestones** (`18 §4`; placement is the spec session's):

| # | Milestone | Contents | Est. LOC |
|---|---|---|---|
| R1 | Renderer foundation | `wgpu` backend behind the trait, WebGL2 fallback, 3D camera, glTF, PBR, instancing, HDR, bloom, tone mapping, new GPU rows | ≈ 2.5k |
| R2 | 2.5D galaxy and system | tilt and orbit camera, seamless zoom galaxy to system to planet, nebula and hyperspace layers, 3D planets with clouds, scattering and lights | ≈ 2.0k |
| R3 | Ship models | Blender → glTF pipeline per lineage and species set, LODs, engine and weapon effects, battles in 3D with the tactical camera | ≈ 1.5k + assets |
| R4 | Portraits and the creator | animated portrait loops from 3D busts, the persona composed from parts, gear shown | ≈ 1.0k + assets |
| U1 | UI design system | skin layer, tokens, type, motion, sounds, species skins as data | ≈ 1.5k |
| U2 | Screen-by-screen pass | every screen in the new language, captain's and state consoles, notification rail, dense tables | ≈ 2.0k |
| A1 | Asset pipeline and environment | the 3D modelling environment of `18 §5` | ≈ 0.5k (scripts) |

Order: A1 and R1 first, together; U1 beside R1; then R2 → R3 → R4 → U2 (`18 §4`). Nothing in the pass changes the simulation, its determinism or its CPU budgets (`18`).

## Key decisions

| Decision | What was decided | Date | Source |
|---|---|---|---|
| UI framework | egui, to be polished into a high-quality, easy-to-navigate interface | 2026-11-01; restated 2026-11-13 | `07 D2` |
| Art direction | Pre-rendered 3D sprites from scripted Blender, flat vector UI, procedural nebula and planet shaders, aiming at Starsector's look diversity | 2026-11-01; restated 2026-11-13 | `07 D8` |
| Map presentation | Top-down in strategic mode, plus a Starsector-style flight view for travel | 2026-11-13 | `07 D3` |
| Backend | `glow`, by the M0 measurement; `wgpu` removed 2026-11-02 | 2026-11-01 | `07 D19`, `03 §3` |
| Ship lighting | Rendered once from above into atlases, lit by the shader from the system's star | 2026-11-05 | `07 D37` |
| Planets | Procedural GPU textures on first view, lit rotating spheres; Sol's bodies use fixed public-domain textures | 2026-11-11 | `07 D37`, `D41` |
| Post-processing | None for the built look | 2026-11-11 | `07 D37` |
| Art timing | A G-track of four milestones interleaved with the sim; G1 after M8 | 2026-11-08 | `07 D37` |
| GPU budgets | Four GPU rows in `08 §1`, measured on the host's GTX 1060 | 2026-11-11 | `07 D37` |
| Music (first call) | A CC0 bed where one fits, procedural synthesis where none does; a commission or licensed score is a 1.0 call | 2026-11-11 | `07 D37` |
| Planet cache size | 128 KB per layer (256×128 with mips) instead of `10 §2`'s 256 KB per body, to stay under 64 MB | 2026-11-14 | `07 D83` |
| Two atlases | Normal map carries emissive strength in blue; two atlases from three maps, two fetches per pixel | 2026-11-14 | `07 D76` |
| Asset budget room | Only the 2× atlases ship; the normal atlas at 1×; the albedo atlas as lossy WebP | 2026-11-14 | `07 D84`, `D88` |
| Immersive presentation | The fifteen calls of `18 §0`, reversing the `glow` choice and `D37`'s "none" for the coming presentation pass | 2026-11-16 | `07 D231` |
| Music by mood | Leaning orchestral; replaces the soft piano bed; procedural synthesis stays as fallback. Replaces `D37`'s per-era CC0 beds | 2026-11-17 | `07 D238` |

## Not yet built or still open

- **The presentation pass** (R1–R4, U1, U2, A1): designed, not placed in the milestone order (`18 §4`). Timing: after the MVP (`18 §0` call 13).
- **New music beds:** track C sources 6–8 CC0 candidates for the owner to hear before anything is downloaded or changed (`07 D238`).
- **Web GPU rows** are not measured (`08 §1`).
- **Species art:** the portrait ids are a hook left by M14 (`05 M14`).
- **Free-text names:** names come from each species' list; no free-text names yet (`DESIGN.md`, `07 D97`).
- **Accessibility pass:** 1.0, in M23 (`05`).
- **Owner review:** the `07` call sets still open for review include `D42` (G1's calls) (`DESIGN.md`).

## Discrepancies noted

- **Has the owner heard the audio?** The `DESIGN.md` pillar row says "G1–G4 built (audio not yet heard by the owner)", while `07 D238` (2026-11-17) says "The owner heard the G4 music for the first time" and chose new music direction.
- **The tracker's graphics pillar against `D231`.** `DESIGN.md`'s pillar row describes the design as "Lit Blender sprites ..., GPU-generated planets ..., no post-processing", while `07 D231` (2026-11-16) chose bloom on by default, `wgpu`, 3D on the map and full 3D planets. The row may describe the built state rather than the decided design; it is not marked either way.
- **What "0.2" means.** `18 §3.6`'s call 13 describes "finish the MVP as built (an internal 0.1); then a presentation pass (0.2, the milestones below)", and the owner chose "Finish the MVP, then the presentation pass". Release 0.2.0 was deployed on 2026-11-23 with gameplay features (rebellion, employee modes, intrigue, trade fleets) and none of R1–U2 (`docs/release/0.2.0-notes.md`, `00-status`).
- **Galaxy map layers.** `01 §16` lists the galaxy map's layers as "political, trade (with goods in transit), piracy, resources, diplomacy, surveyed", while `02 §11.2` lists draw layers (background, territory, hyper, trade, systems, fleets, transit, overlays, labels). Which of `01 §16`'s toggle layers are built is not recorded.

## Sources

- 01 Game design: UI screens
- 02 Architecture: rendering and UI
- 03 Tech stack: graphics and UI stack, assets
- 05 Milestones (G1–G4, M14)
- 08 CPU budget: budgets
- 10 Graphics scoping: approach, budgets, milestone track
- 18 Presentation and UI: the owner's calls, milestones
- 07 Decisions: `D2`, `D3`, `D8`, `D19`, `D37`, `D41`, `D42`, `D76`, `D83`, `D84`, `D87`, `D88`, `D97`, `D106`, `D153`, `D154`, `D180`–`D182`, `D231`, `D238`
- 00 Status
- DESIGN tracker
- Release notes 0.1.0, 0.2.0

## Related pages

- [Overview](Overview)
- [Architecture and performance](Design-Architecture-and-performance)
- [Modding and data](Design-Modding-and-data)
- [Roadmap and status](Design-Roadmap-and-status)
- [Flight view](Design-Flight-view)
- [Combat](Design-Combat)
- [Colonies and surfaces](Design-Colonies-and-surfaces)
- [Galaxy and hyperspace](Design-Galaxy-and-hyperspace)
- [Time and modes](Design-Time-and-modes)
