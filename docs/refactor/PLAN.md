# Hallowmere refactor design (multi-agent, wave-gated)

> Repo copy of the approved plan. The live task tracker is `docs/refactor/TASKS.md`. Agents are pointed at numbered sections of this file.


## 1. Context

Hallowmere is a Three.js action RPG served as plain ES modules from `dist/` (no bundler; `dist/` is uploaded to GitHub Pages as-is). It was built in ~160 commits over two days, mostly by parallel Codex agents merging feature branches. It works and has 269 passing tests, but it grew by accretion:

- `dist/main.js` is 97 KB in 593 lines (avg 162 chars/line, longest 4,339), touched in 45 of 160 commits. It wires the whole DOM at import time, so tests cannot import it; nine tests read it as text and `vm`-eval string slices bounded by function names.
- ~85 modules in `dist/`, 41 with no test, including every scene builder and character model.
- The same model-primitive kit is copied 10×, `dispose` 11×, a private LCG 6×, portrait renderers 3×.
- Undocumented variation pages ship to production alongside the game.
- Test helpers are copy-pasted across files.

Intended outcome: a `main.js` that is a ~70-line composition root over ~20 importable, individually tested modules; one copy of each shared helper; dev-only pages out of the deploy; then a measured performance pass on the cleaned seams. Zero observable change to gameplay or visuals.

## 2. Decisions (confirmed with the user)

| Topic | Decision |
|---|---|
| Priority | Structure first, then performance |
| Tooling | No bundler. `dist/` stays directly servable. No new runtime deps. Dev-only tooling OK. |
| Behavior | Strict: no observable gameplay/visual change. Perf wins only from removing waste. |
| Tracker | `docs/refactor/TASKS.md` in the repo, orchestrator-maintained, single source of truth |
| Worktrees | One registered worktree per task via `scripts/worktree.mjs start` (branches from `main`) |
| Integration | **Gate per wave.** Each wave = one `main.js` step + up to two disjoint parallel tasks. At each gate the user authorizes once; the orchestrator runs `worktree.mjs finish` for every task in the wave. Phase 3 (no `main.js`) runs as batches with a single gate. |
| main.js steps | Each extraction step is its own task/worktree/merge (independently revertible). |
| Engine | Claude subagents launched by this session (Agent tool), ≤3 concurrent, orchestrator reviews every diff |
| Dev pages | Undocumented pages move out of `dist/` to `docs/variations/` |
| Assets | **Out of scope.** No audio/image recompression. Only the unreferenced `app-icon.png` is removed as dead code. |

## 3. Hard constraints (every agent prompt includes these)

1. **`dist/` top-level JS is flat.** `scripts/validate.mjs` globs one level. Do not create `dist/<subdir>/*.js` (T0-3 makes the glob recursive, but stay flat anyway; the import map and relative-URL rules are simplest flat).
2. **`$('id')` validator contract.** Today `validate.mjs:39-43` checks only `main.js`. T0-3 replaces it with a per-page transitive-closure check. Until T0-3 merges, no `$()` call may leave `main.js`. After T0-3, keep `$('literal')` form everywhere (never `$(variable)`).
3. **DOM-free core.** `server/{world,class-combat,region-campaign,region-combat}.mjs` are `export * from '../dist/...'` shims. The 19-module import closure of `dist/world.js` (`buildings, campaign, cave-expansion, cave-scenery-layout, caves, class-combat, classes, combat, enemy-encounters, expansion-layout, foraging, multiplayer-protocol, region-campaign, region-combat, regions, road-layout, world-layout, world-random, world`) must never import `three`, `./vendor/`, or touch `document/window/navigator/location/matchMedia`. New `random.js` joins this closure via `expansion-layout.js`.
4. **Relative URLs only** for assets/imports (`/assets/`, `/vendor/` are rejected).
5. **Bare `three` resolves only via the import map** in `index.html`. Tests import `dist/vendor/three.core.js` directly or use the helper shim.
6. **`tests/pages-deployment.test.mjs` regex-extracts the first `run: |` block of `.github/workflows/pages.yml`.** Do not add an earlier block or change its indentation.
7. **`.openai/hosting.json` keeps `static.directory: "dist"`.**
8. **Never merge to `main` or run `worktree.mjs finish` without the user's explicit authorization** (AGENTS.md). Agents never run `finish`; only the orchestrator does, after a gate authorization.
9. **`PROTOCOL_VERSION = 3`** and the wire shape are frozen.
10. **`window.hallowmere` and `document.modelContext.registerTool` (main.js:584-587)** are an external automation API. Key set, tool names, and `inputSchema` must remain identical.
11. **Code style:** match the existing dense style of the file being edited. Do not reformat lines you are not changing (keeps diffs reviewable).
12. **Node:** prefix commands with `bash scripts/node22.sh` (shell has Node 25; CI/.nvmrc use 22).

## 4. Baseline (branch `claude/game-refactor-plan-bfc672` @ main `ff9a17f`, 2026-09-14)

| Metric | Value |
|---|---|
| Tests | 269 pass / 0 fail / 47 files |
| Suite wall time | ~84 s (worktree integration tests ≈25 s) |
| `npm run build` | passes |
| `dist/*.js` | 758 KB, ~8.7k lines, 85 files |
| `dist/main.js` | 97 KB, 593 lines, 141 `$()` calls over 72 ids |
| `dist/*.css` | 176 KB (109 KB loaded by index.html) |
| `dist/assets` | 58 MB (untouched by this plan) |

## 5. Findings (evidence; condensed)

### 5.1 What is already good (preserve)
- Simulation core is clean, DI-styled, and shared with the server. `local-session.js` and `multiplayer-client.js` are interchangeable transports consumed by `applySnapshot`.
- GLTF prefabs loaded once, merged per material (`optimizeModel`), cloned with shared geometry. `InstancedMesh` for scenery. `dispose()` on most scene modules; `switchMap` disposes correctly.
- DI is the house testing style: `AudioEngine({contextFactory,...})`, `bindPageActivity({windowTarget,documentTarget,...})`, `createGameServer({world,...})`.

### 5.2 main.js section map
| Lines | Responsibility |
|---|---|
| 1–52 | imports, `$`, icon atlas (`:48` runs `querySelectorAll('[data-icon]')` at import) |
| 53–69 | ~90 module-scope mutable bindings; `:63 worldBounds`, `:65 safeHere`, `:66 bossType`, `:67 enemyModelType`; `:68 tmp` unused |
| 70–86 | `init()`: renderer/scene/lights/GLTF load; `setAnimationLoop(frame)` |
| 87–187 | mode choice, journey lifecycle, session lifecycle, main menu, pagehide/pageshow |
| 188–198 | `optimizeModel`, `cloneModel`, `getRig`, `spawnEnemy`, `updateEnemyBar`, `resize`, `awaken` |
| 199–259 | pointer/raycast, interaction, `regionInteractions`, region travel, `renderRegionLabels`, `renderHazards`, `nearestEnemy` |
| 260–324 | all input bindings |
| 325–357 | `toast`, `perform`, effect factories, `removeObject`, `animateRig`, `updatePlayer`, `updateEffects`, `floatText`, `updateFloaters` |
| 358–393 | class HUD, roster, `updateUI`, `drawMap`, audio positioning |
| 394–397 | `frame()` |
| 398–471 | inventory render, modal system, pause menu, restart, connection UI |
| 472–582 | `applySnapshot`, `networkEvent`, `renderSharedWorld` |
| 584–593 | automation surface, boot |

### 5.3 Tests that read main.js as text (must be migrated at the named step)
| Test | Slice markers | Migrates at |
|---|---|---|
| character-integration, npc-models, oathkeeper-appearance, ranger-appearance | `function optimizeModel` → `function spawnEnemy` | M2 |
| nightblade-appearance | same + `function networkEvent(event)` → `if(skill?.kind==='projectile')` | M2, M10 |
| mouse-targeting | `function attacksFromHere(` → `function stopAttackMovement(`; `function updatePlayer(` → `function updateEffects(` | M5, M6 |
| loot-pickup | `function pointerAction(` → `function updateMouseTarget(`; `function collectClickedLoot(` → `function regionInteractions(`; `$('world').addEventListener('pointerdown'` → `window.addEventListener('pointerup'` | M4, M5 |
| game-mode-choice | `function setModeChoiceInert` → `$('connection-back').onclick`; `function connectionStatus(` → `$('connection-retry').onclick` | M9 |
| audio | regex `(?:audio\.play\|audioAt)\('…'` over main.js | T0-1 (widen to all `dist/*.js`) |

### 5.4 Per-frame hot spots (Phase 3 targets)
| Location | Waste |
|---|---|
| `main.js:397` | 2 `Vector3` allocs for camera target; `safeHere()` inside `enemies.some` (O(enemies×checkpoints)) |
| `mouse-targeting.js:34` | `getBoundingClientRect()` every frame = forced layout |
| `main.js:202-209` | raycast + `life.pickLoot` (filter+flatMap+intersectObjects) + `pickDoor` every frame |
| `main.js:234 regionInteractions()` | `Object.entries().flatMap(spread).filter` rebuilt 2–3×/frame (`:208,:240,:380,:386`) |
| `main.js:221-223` | 3 unconditional `classList.toggle`/frame |
| `main.js:239-257` | labels: `new Vector3().project`, `querySelector`, `textContent`, `style` writes with no change check; hazards: `new Set` + material writes |
| `main.js:355` + `combat-effects.js:239-241` | 5 array allocs, object spread per bolt, `sort()` every frame |
| `world-actors.js:83-84` | spread + filter + `localeCompare` sort + 2 allocs/item + unconditional style writes |
| `main.js:561-566,578` | `new Vector3` per enemy; `updateEnemyBar` redraws 256×48 canvas + re-uploads `CanvasTexture` while bar animates; `new Set(...)` for projectiles/zones |
| `local-session.js advance→publish` | `structuredClone(world.snapshot())` up to 20×/s; `world.js:161` 6 filter/map/rest chains |
| `main.js:378-383 updateUI` (11 Hz) | ~22 unconditional DOM writes; `querySelector('[data-action=…]')` in ability loop; `questSummary(state)` recomputed |
| `exploration-map.js:81-114` (11 Hz) | full minimap redraw with `ctx.filter=blur()`; per-enemy `hasLineOfSight` |
| `main.js:340-345` | new `RingGeometry(…,64)`, materials, `Float32Array`s per combat event; no pooling |
| `resource-orbs.js` | no `dispose()` |

### 5.5 Duplication
- **Model primitives** (`group/mesh/ball/box/rod/path/plate/material`, ~11.9 KB) in geralt-, nightblade-, oathkeeper-, ranger-, reaver-character-model.js, hunt-king-model.js, predator-model.js, npc-models.js, character-study-models.js, `scripts/generate-assets.mjs`. Second tier `cloak/ribbon/edgedPlate/wrap/makeHead/makeFace/makeHand` in 2–4 files.
- **`dispose` traverse-and-release** 11× + inline variant in enemy-visuals.
- **Portrait camera-fit** duplicated in inventory-portraits (`traverse`, pad 1.05) and npc-portraits (`traverseVisible`, pad 1.07). Helper must be parameterized, not unified.
- **LCG `rand`** 6× (cave-entrance-scenery, cave-scenery, environment, expansion-layout, outland-scenery, region-environment). `environment.js:11` uses `seed*1664525` instead of `Math.imul`; **proven bit-identical** for all uint32 seeds (max product 7.15e15 < 2^53; 0 diffs over 2M sequential + 200k random seeds). Safe to unify; ship the proof as a test.
- **Merge-batch idiom** 6×, **palette idiom** 5×, **value-noise** 3× (+ `makeNoiseTexture`).
- **Small helpers:** `$` 5× (`selection-explorations.js` defines `$` as `querySelector`), `escape` 3×, `clamp` 2× (different signatures), `range` 2×, `project` 2×, `glow` 3×.
- **Dead:** `campaign.js:11 VILLAGES`; 10 exports with no external consumer; `dist/npc-models.js` is build-time only; `dist/assets/icons/app-icon.png` (1.9 MB) unreferenced.
- **Three import spelled 3 ways:** `'three'` (17), `'./vendor/three.core.js'` (17), `'./vendor/three.module.js'` (inventory-portraits, npc-portraits).

### 5.6 Dev pages
| Move to `docs/variations/` | Keep in `dist/` |
|---|---|
| `geralt-character-study.*`, `hunt-king-study.*` + `hunt-king-model.js`, `predator-study.*`, `selection-explorations.*` + `selection-directions.js`, `orb-variations.html`, `quest-proposals.html` | `character-studies.*` (+ `character-study-selection.js`), `sound-audition.html` (tested), `route-atlas.html`, `menu-directions.html`, `predator-world-preview.*` (live), `cave-preview.js`, `enemy-preview.js` |
`dist/npc-models.js` → `scripts/npc-models.mjs` (used only by `scripts/generate-assets.mjs` and its test).

### 5.7 CSS
- Four `:root` blocks + four font `@import`s; `--muted` has 4 values, `--line`/`--bright` drifted.
- `chronicle.css` (24.8 KB) redeclares 80 selectors from inventory/roster/dialogue/title-screen/style with different bodies and wins by load order (`index.html:42`). Deliberate layering; the shadowed base rules are mostly dead weight. `#connection-overlay`/`.connection-card`/`.connection-spinner` defined 3×.

## 6. Design

### 6.1 State sharing: one `ctx` object
`dist/game-context.js` exports `createGameContext({audio, gameSettings, exploration, resourceOrbs, journeyStore, previewMode, reducedMotion, coarse})` returning **one flat object literal with every field pre-declared** (data fields carry the `main.js:53-68` initial values; action slots are `null`), plus `assertWired(ctx)` that throws at boot if any slot is still `null`. Extracted modules are factories `createX(ctx)` that close over `ctx` and return the functions `main.js` assigns back onto it.

Why: the `vm` tests already build hand-rolled context objects (`tests/mouse-targeting.test.mjs:13-20`, `tests/loot-pickup.test.mjs:83-87`), so migration is near-mechanical; after one rename pass (`X` → `ctx.X`) every later extraction is cut-and-paste; ES live bindings fail on the ~60 bindings written from multiple sites; strict-mode modules make a missed rename a loud `ReferenceError` (no binding name collides with a `window` global, verified). One pre-declared shape keeps property loads monomorphic; the pre-split perf baseline (T0-4) and P6a prove the split cost nothing.

### 6.2 Target module map (all flat in `dist/`)
| Module | From main.js | Exports |
|---|---|---|
| `game-context.js` | `:53-68` bindings, `:63,:65,:66,:67` helpers | `createGameContext`, `assertWired` |
| `util.js` | `:43 $` | `$`, `escapeHtml`, `clamp`, `clamp01` |
| `icon-atlas.js` | `:44-48` | `icons`, `classIconNames`, `icon`, `paintIcons(root)` (explicit call replaces import-time `querySelectorAll`) |
| `model-kit.js` | `:188-190` | `optimizeModel`, `getRig` (pure, standalone), `createModelCache(ctx)`→`{cloneModel}` |
| `scene-setup.js` | `:72-79`, `:196 resize` | `createSceneSetup(ctx)`→`{buildRenderer, buildWorld, buildHero, resize}` |
| `enemy-spawner.js` | `:191-194` | `createEnemySpawner(ctx)` |
| `effects-factory.js` | `:340-347`, `:356 floatText` | `createEffects(ctx)` |
| `pointer-targeting.js` | `:199-224`, `:259` | `createPointerTargeting(ctx)` |
| `interaction.js` | `:225-237` | `createInteraction(ctx)` |
| `region-travel.js` | `:238-257` | `createRegionTravel(ctx)` |
| `input-bindings.js` | `:260-324` | `bindInput(ctx,{windowTarget=window,documentTarget=document})` |
| `player-motion.js` | `:326,:339,:348-355,:357` | `createPlayerMotion(ctx)` |
| `game-audio.js` | `:88,:198,:390-393` | `createGameAudio(ctx)` |
| `hud.js` | `:325,:358-388` | `createHud(ctx)` |
| `modals.js` | `:398-399,:439-463,:508` | `createModals(ctx)` |
| `inventory-ui.js` | `:400-438` | `createInventoryUi(ctx)` |
| `connection-ui.js` | `:464-471` | `createConnectionUi(ctx)` |
| `session-lifecycle.js` | `:87,:89-187` | `createSessionLifecycle(ctx,{windowTarget=window})` (registers `pagehide` #1) |
| `snapshot-apply.js` | `:472-507` **verbatim** | `createSnapshotApply(ctx)` |
| `network-events.js` | `:509-559` | `createNetworkEvents(ctx)` |
| `shared-world-render.js` | `:560-582` | `createSharedWorldRender(ctx)` |
| `render-loop.js` | `:394-397` | `createRenderLoop(ctx)` |
| `automation-surface.js` | `:584-587` | `installAutomationSurface(ctx,{windowTarget,documentTarget})` (registers `pagehide` #2) |

`main.js` keeps: imports, `createGameContext(...)`, `init()` (async load orchestration with `titleScreen.setProgress`), the wiring block (`Object.assign(ctx, createX(ctx), ...)` in a fixed order: session-lifecycle before automation-surface, `worldPreview` pagehide last), `assertWired(ctx)`, audio-unlock capture listeners, `awaken(); init();`. Target ~70 lines. Import-time side effects (`:185`, `:401 bindInventoryPreviews`, `:452 bindPauseMenu`, listeners) move inside factory bodies so registration order equals the visible order of the wiring block.

### 6.3 Validator upgrade (T0-3)
Replace `validate.mjs:17-27` glob with a recursive walk of `dist/**` excluding `dist/vendor/`, and replace the `main.js` special case (`:39-43`) with: for each `dist/*.html`, collect `id="…"`, walk the transitive local-module closure of its `<script type=module src>` (and inline module imports), and require every `$('literal')` in that closure to be an id on that page. Verified to pass on the current tree for every page once `selection-explorations.js` (which defines `$` as `querySelector`) has moved out (T0-5 precedes T0-3). Strictly stronger than today (also covers `character-studies.html`).

### 6.4 Test safety net (T0-1, T0-2)
- `tests/helpers/source.mjs`: `sliceBetween(source,start,end)` **throws on a missing marker** (today `indexOf` → `-1` yields a plausible substring and a silently green test). All slicing tests adopt it before any extraction.
- `tests/helpers/dom.mjs` (one save/restore stub with `t.after`), `tests/helpers/three-shim.mjs` (the `mergeGeometries` data-URL shim pasted in 5 tests), `tests/helpers/sim.mjs` (`tick`, `cast`).
- `tests/core-purity.test.mjs`: walks the `world.js` closure, asserts no `three`/`./vendor/`/DOM identifiers, imports all four `server/*.mjs` shims. Run on **every** task.
- `tests/automation-surface.test.mjs` (source-level snapshot in T0-1: `window.hallowmere` key list + tool names + action enum; upgraded to a real contract test at M11).
- `tests/audio.test.mjs` cue regex widened to all `dist/*.js` (only 15 literal cues exist).

### 6.5 Perf and visual baselines (T0-4)
- `scripts/perf-smoke.mjs` (reporting script, not a CI test): seeded `World`; medians-of-5 for `world.step`, `world.snapshot`, `structuredClone(snapshot)`, `LocalSession.advance`; later `regionInteractions` (after M4) and `VillageLife.renderLabels` (P2). Prints ms/op and bytes/op (heap delta with `--expose-gc`).
- `docs/refactor/PERF.md`: browser protocol using the existing `window.hallowmere.getState().drawCalls` and a 900-frame `requestAnimationFrame` sampler → p50/p95/p99 frame time, draw calls, heap delta; fixed scenario driven through `hallowmere` `control_warden` (solo → Sorcerer → 300 frames idle → 3 waypoints → 600 frames); 3 runs, medians.
- `docs/refactor/VISUAL.md` + `screenshots/refactor-baseline/`: the **13-shot set** (title/mode choice, journeys menu, roster picker, main menu, connection overlay, HUD at spawn, inventory, journal, help, pause, NPC dialogue, death, victory/map expanded) at 1440 wide; CSS tasks add 1024 and 390.

## 7. Task DAG

Legend: `V` = `bash scripts/node22.sh npm test && bash scripts/node22.sh npm run build` + core-purity. Size S/M/L. Model = haiku/sonnet/opus (escalation ladder: haiku → sonnet → opus → fable on a second failed review). One `main.js`-owning task per wave, marked **M**.

### Phase 0 — safety net (2 waves, 2 gates)
| ID | Wave | Model | Size | Title | Owns | Depends | Extra verification |
|---|---|---|---|---|---|---|---|
| T0-0 | 0A | orchestrator | S | Tracker + design doc: `docs/refactor/{TASKS,PLAN}.md` | those two files (tracker worktree `hm-tracker`) | — | — |
| T0-2 | 0A | opus | S | DOM-free core guard test | `tests/core-purity.test.mjs` | — | green on current main; prove it bites by temporarily adding a `three` import |
| T0-4 | 0A | sonnet | L | Perf + visual baseline | `scripts/perf-smoke.mjs`, `docs/refactor/{PERF,VISUAL}.md`, `screenshots/refactor-baseline/` | — | numbers + 13 shots recorded from unmodified main |
| T0-5 | 0A | haiku | M | Move dev pages to `docs/variations/`; `npc-models.js` → `scripts/npc-models.mjs` | the files in §5.6, `scripts/generate-assets.mjs`, `tests/npc-models.test.mjs`, `README.md`, `docs/selection-explorations.md` | — | `V`; remaining pages load with no console errors; `npm run generate -- --npcs` output byte-identical |
| T0-1 | 0B | sonnet | M | `tests/helpers/*`, slice guard, audio widening, automation snapshot test | `tests/helpers/`, the 9 main.js-reading tests, ~6 helper-pasting tests, `tests/automation-surface.test.mjs` | T0-5 | prove the guard: rename a marker → red |
| T0-3 | 0B | opus | M | Validator: recursive glob + per-page `$()` closure | `scripts/validate.mjs` | T0-5 | prove it bites: bogus `$('nope')` → build fails naming the page |
| T0-7 | 0B | sonnet | M | New shared modules with zero consumers + tests | `dist/{model-primitives,util,dispose,random}.js`, `tests/{model-primitives,util,dispose,random}.test.mjs` | — | `random.test.mjs` carries the LCG equivalence proof (1e6 iterations from the 4 real seeds); `random.js` Three-free+DOM-free; `model-primitives.js` DOM-free |

### Phase 1 — main.js M1–M5 + dedup (5 waves, 5 gates)
| ID | Wave | Model | Size | Title | Owns | Depends | Extra verification |
|---|---|---|---|---|---|---|---|
| **M1** | 1.1 | opus | L | ctx rename pass (no extraction): adopt `game-context.js`, `util.js`; every binding → `ctx.X`; move `:63,:65-67` helpers | `dist/main.js`, `dist/game-context.js`, tests game-mode-choice/loot-pickup/mouse-targeting/nightblade (vm contexts → `{ctx}`) | 0B | `V`; `main.js` has zero top-level `let`; play 60 s; 13 shots identical; browser perf within noise of T0-4 |
| T1-1 | 1.1 | sonnet | M | model-primitives → geralt, nightblade | `dist/{geralt,nightblade}-character-model.js` + golden tests | T0-7 | golden geometry hash (vertex count + bbox + material colors) captured on main first, identical after |
| T1-2 | 1.1 | sonnet | M | model-primitives → oathkeeper, ranger | `dist/{oathkeeper,ranger}-character-model.js` + golden tests | T0-7 | same |
| **M2** | 1.2 | sonnet | S | `model-kit.js`, `icon-atlas.js` | `dist/main.js`, 2 new, 5 appearance/model tests (drop `vm`, import `optimizeModel`) | M1 | `V` |
| T1-3 | 1.2 | sonnet | M | model-primitives → reaver, predator-model | `dist/{reaver-character-model,predator-model}.js` + golden tests | T0-7 | golden |
| T1-4 | 1.2 | sonnet | M | model-primitives → character-study-models, generate-assets | `dist/character-study-models.js`, `scripts/generate-assets.mjs`, `scripts/npc-models.mjs` | T0-7, T0-5 | `npm run generate` output byte-identical |
| **M3** | 1.3 | sonnet | M | `effects-factory.js`, `enemy-spawner.js` | `dist/main.js`, 2 new, `tests/effects-factory.test.mjs` | M2 | `V` |
| T1-5 | 1.3 | haiku | M | `dispose.js` adoption, effects tier | `dist/{combat-effects,class-effects,multiplayer-view,enemy-visuals}.js` | T0-7 | existing tests cover disposal counts |
| T1-6 | 1.3 | sonnet | L | `random.js` + `dispose.js` adoption, scenery tier | `dist/{environment,cave-entrance-scenery,cave-scenery,expansion-layout,outland-scenery,region-environment}.js` | T0-7 | core-purity green; overworld obstacle-position hash identical before/after |
| **M4** | 1.4 | sonnet | M | `interaction.js`, `region-travel.js`, `pointer-targeting.js` | `dist/main.js`, 3 new, `tests/loot-pickup.test.mjs` (slices 1–2 → imports); add `regionInteractions` to perf-smoke | M3 | `V` |
| T1-7 | 1.4 | haiku | S | `util.js` adoption | `dist/{dialogue,journeys-menu,inventory,resource-orbs}.js` | T0-7 | `escapeHtml` identical for `&<>"'` |
| T1-8 | 1.4 | sonnet | M | Portrait camera-fit helper (parameterized) | `dist/{portrait-fit,character-portraits,inventory-portraits,npc-portraits}.js` | T0-7 | `toDataURL` hashes of 7 class + 5 NPC portraits identical |
| **M5** | 1.5 | opus | L | `input-bindings.js` (DI `windowTarget/documentTarget`) | `dist/main.js`, `dist/input-bindings.js`, tests loot-pickup (slice 3), mouse-targeting (slice 1), new `input-bindings.test.mjs` (listener registration order) | M4 | `V`; manual pass over every binding |
| T1-9 | 1.5 | haiku | S | Dead code | `dist/campaign.js` (`VILLAGES`), 10 internal-only `export`s, `dist/assets/icons/app-icon.png`, `docs/app-icon.md` + `docs/social-previews.md` references | — | grep proof of zero references per removal |
| T1-10 | 1.5 | sonnet | M | Geometry batcher + palette helper | `dist/{geometry-batch,palette}.js`, `dist/{environment,cave-entrance-scenery,treasure-chests,outland-scenery}.js` | T1-6 | `hallowmere.getState().drawCalls` identical at spawn |

### Phase 2 — import normalization, M6–M11, CSS (7 waves, 7 gates)
| ID | Wave | Model | Size | Title | Owns | Depends | Extra verification |
|---|---|---|---|---|---|---|---|
| T2-1 | 2.0 (solo) | haiku | S | Three import normalization to bare `'three'` where the renderer is needed, `'./vendor/three.core.js'` elsewhere; fix the two `three.module.js` relative imports | all `dist/*.js` | Gate 1.5 | `V`; node test asserts single `T.REVISION` instance across both entrypoints; draw calls unchanged |
| **M6** | 2.1 | sonnet | M | `player-motion.js` | `dist/main.js`, 1 new, `tests/mouse-targeting.test.mjs` (slice 2) | T2-1 | `V` |
| T2-2 | 2.1 | sonnet | M | `tokens.css` (shared `:root`, one font `@import`, button/link resets) | `dist/tokens.css`, `:root` blocks + `@import`s in `dist/{style,title-screen,roster,inventory,journeys,dialogue}.css`, `dist/index.html`, `dist/character-studies.css` | — | 13 shots × 3 viewports identical; each drifted token resolved to the value that wins the cascade today, documented |
| T2-3 | 2.1 | sonnet | M | value-noise dedup | `dist/{noise,class-effect-materials,loot-effects,map-fog}.js` | T1-6 | `map-fog` mask hash identical |
| **M7** | 2.2 | sonnet | M | `hud.js`, `game-audio.js` | `dist/main.js`, 2 new | M6 | `V`; HUD shots identical |
| T2-4 | 2.2 | opus | M | chronicle.css layering map (docs only, zero CSS edits) | `docs/refactor/CSS-LAYERS.md` | T2-2 | enumerates all 80 redeclared selectors and whether each base rule is reachable on any page/state |
| T2-5 | 2.2 | sonnet | S | `#connection-overlay` triplicate → one definition | `dist/{style,chronicle,title-screen}.css` | T2-2 | overlay shots in loading/reconnecting/failed + over main menu |
| **M8** | 2.3 | sonnet | M | `modals.js`, `inventory-ui.js` | `dist/main.js`, 2 new, `tests/modals.test.mjs` | M7 | `V`; modal shots identical |
| T2-6 | 2.3 | opus | L | chronicle.css dead-rule deletion | `dist/chronicle.css` + the base files it shadows | T2-4, T2-5 | **39 shot pairs** (13 × 3 viewports) byte-compared; any diff → revert that rule; if pairs cannot be made identical, abandon and keep file as-is |
| **M9** | 2.4 | opus | L | `session-lifecycle.js`, `connection-ui.js` | `dist/main.js`, 2 new, `tests/game-mode-choice.test.mjs` | M8 | `V`; session matrix: solo→journey→save&exit→continue; multiplayer→disconnect→retry; main menu→roster→cancel; pagehide order test |
| **M10** | 2.5 | opus | L | `snapshot-apply.js` (verbatim), `network-events.js`, `shared-world-render.js` | `dist/main.js`, 3 new, `tests/{nightblade-appearance,snapshot-apply}.test.mjs` | M9 | `V`; two-client multiplayer session against `scripts/serve.mjs`; `PROTOCOL_VERSION` untouched; statement-by-statement diff vs `git show main:dist/main.js` |
| **M11** | 2.6 | opus | L | `scene-setup.js`, `render-loop.js`, `automation-surface.js`; `main.js` → composition root; `assertWired`; delete `tmp` | `dist/main.js`, 3 new, `tests/automation-surface.test.mjs` (contract test) | M10 | `V`; `window.hallowmere` keys + both tool `inputSchema`s identical to T0-1 snapshot; 13 shots; browser perf vs T0-4 |

### Phase 3 — performance on the cleaned seams (4 batches of 3, 1 gate)
All branch from post-M11 `main`; files are disjoint by construction, so batches are execution order only.
| ID | Batch | Model | Size | Title | Owns | Fix | Measurement |
|---|---|---|---|---|---|---|---|
| P1 | 3.1 | sonnet | M | pointer rect cache | `dist/mouse-targeting.js`, new `tests/mouse-targeting-pick.test.mjs` | cache `getBoundingClientRect()`, invalidate on resize/scroll | browser p95/p99 |
| P2 | 3.1 | sonnet | L | village-life per-frame allocs | `dist/world-actors.js` | reuse arrays; sort behind dirty flag; guard style writes | perf-smoke `renderLabels` + browser p95 |
| P3 | 3.1 | sonnet | M | light-source array | `dist/combat-effects.js` | scratch array; skip `sort()` when not needed | perf-smoke; bolt lighting shots identical |
| P4 | 3.2 | sonnet | M | minimap | `dist/exploration-map.js` | cache blurred fog layer; cache LOS between 11 Hz redraws | browser p95; minimap pixel-identical |
| P5 | 3.2 | opus | L | snapshot + clone | `dist/world.js`, `dist/local-session.js`, new `tests/snapshot-aliasing.test.mjs` | aliasing test **first** (world mutation after `snapshot()` never changes a returned snapshot), then emit detached objects and drop `structuredClone` | perf-smoke `snapshot`/`advance` ms+bytes; core-purity green |
| P7 | 3.2 | haiku | S | orb disposal | `dist/resource-orbs.js`, `tests/resource-orbs.test.mjs` | add `dispose()` | test |
| P6a | 3.3 | opus | M | render loop | `dist/render-loop.js` | scratch camera vectors; hoist `safeHere()` | browser p50/p95/p99 vs T0-4 pre-split baseline (proves the split cost nothing) |
| P6b | 3.3 | sonnet | M | region labels + hazards | `dist/region-travel.js` | cache `querySelector`; write on change only; reuse id `Set` | browser p95; label shots identical |
| P6c | 3.3 | sonnet | M | HUD writes | `dist/hud.js` | guard ~22 writes; ability lookup built once; memo `questSummary` | browser p95; HUD shots identical |
| P6d | 3.4 | sonnet | L | enemy sync + bars | `dist/shared-world-render.js`, `dist/enemy-spawner.js` | reuse `Vector3`; redraw bar canvas only on hp/name change; reuse id `Set`s | browser p95 during boss fight; bar pixels identical |
| P6e | 3.4 | sonnet | M | interaction memo | `dist/interaction.js`, `dist/pointer-targeting.js` | memoize `regionInteractions()` keyed on `(lastSnapshot, discoveries.length)`; invalidate in `applySnapshot` | perf-smoke bytes/op |
| P8 | 3.4 | sonnet | L | effect pooling | `dist/effects-factory.js`, `tests/effects-factory.test.mjs` | pool ring/slash geometries, materials, particle buffers | browser heap delta over 600 combat frames |

Totals: 40 tasks, 18 waves/batches, **15 gate authorizations** (2 + 5 + 7 + 1).

## 8. Operating procedure

### 8.1 Starting a task (orchestrator)
1. Ensure the previous gate for this wave's dependencies is merged (`git -C /Users/miguelsolorio/Developer/hallowmere log -1 main`).
2. Create the worktree from the **primary checkout** (the helper refuses non-`codex/` cwd; this is the one sanctioned exception to staying in this session's worktree):
   ```bash
   (cd /Users/miguelsolorio/Developer/hallowmere && bash scripts/node22.sh node scripts/worktree.mjs start --task hm-<ID> --name <slug>)
   ```
   Read `path`, `branch`, `port` from the JSON. Run `npm ci` in `path`.
3. Update `docs/refactor/TASKS.md` (status `running`, branch, path) in the `hm-tracker` worktree; commit.
4. Launch the agent (`Agent` tool, `model` per the DAG, `run_in_background: true`, ≤3 concurrent) with the prompt template in 8.2.

### 8.2 Agent prompt template (self-contained; agents never read this plan or other agents' output)
```
TASK <ID>: <title>
Worktree: <path> (branch <branch>, dev port <port>). Work ONLY here. Never cd to the primary checkout. Never run `worktree.mjs finish`, never merge, never push.
Files you may change: <exact list>. If you need to touch anything else, STOP and report why.
Constraints: <§3 block verbatim>.
Context: <the §5 rows relevant to this task, with file:line refs>.
Steps: <numbered, concrete>.
Behavior rule: no observable change. Extract verbatim; no drive-by cleanups; keep the dense style; do not reformat untouched lines.
Done checklist (paste evidence for each):
 1. `bash scripts/node22.sh npm test` summary line (expect ≥269 pass, 0 fail; state new total if you added tests)
 2. `bash scripts/node22.sh npm run build` final line
 3. `bash scripts/node22.sh node --test tests/core-purity.test.mjs` green
 4. `git status --porcelain` empty; `git diff --stat main...HEAD` lists only your files
 5. <task-specific verification from the DAG row>
 6. One commit per logical step, message starts with "<ID>:"; end with the Co-Authored-By line
Report (≤300 words): branch, commit(s), files, what you verified, anything you could not verify. No transcript.
```

### 8.3 Orchestrator review (per task, read the diff not the summary)
- `git -C <path> diff main...HEAD` — every path in the declared set; no duplicate paths across the wave; exactly one task touched `main.js`.
- Scan for semantic drift in "mechanical" diffs: `??`→`||`, `.find`→`.some`, `===`→`==`, moved `await`, dropped `?.`, reordered side effects, `let`→`const` on a reassigned binding.
- Core purity: no `three`/DOM import in the 19-module closure.
- Test strength: a migrated test asserts the same things; shrunken assertions are rejected.
- For M9/M10/M11: diff `window.hallowmere` keys and tool schemas against the T0-1 snapshot.
- Outcome → TASKS.md status `review` → `ready-to-merge` or `changes-requested` (send the agent a follow-up via SendMessage; on a second failure escalate one model tier in the same worktree).

### 8.4 Gate (once per wave)
1. All wave tasks `ready-to-merge`. Ask the user once: "Gate <wave>: merge <IDs>?"
2. On yes, from each task worktree in dependency order: `bash scripts/node22.sh node scripts/worktree.mjs finish --task hm-<ID>` (helper merges into `main` in a temp worktree, runs `npm ci && npm test && npm run build`, fast-forwards `main`). Tracker worktree last.
3. Record merge commits in TASKS.md; set status `merged`. On `VALIDATION_FAILED`, keep the retained integration path in TASKS.md, fix in the task worktree, rerun.
4. Post-gate smoke: `npm run dev` from the tracker worktree (merged with main), load `index.html`, check console clean, take the 13 shots when the wave touched UI.

### 8.5 `docs/refactor/TASKS.md` format (orchestrator-only edits)
```
# Refactor tracker  (updated <date>; plan: docs/refactor/PLAN.md)
Status legend: todo · running · review · changes-requested · ready-to-merge · merged · blocked · dropped
## Gates
| Gate | Tasks | Authorized | Merged commits |
## Phase 0
| ID | Wave | Model | Title | Owns | Depends | Status | Branch | Path | Port | Commit | Notes |
...
## Log
- 2026-09-14 — created; baseline 269/269, 84 s
```
Every status change is one commit in the `hm-tracker` worktree. `docs/refactor/PLAN.md` is §3, §5, §6 of this document so agents can be pointed at specific sections by line.

### 8.6 Using agents and limits well
- ≤3 agents concurrently; one of them is the `main.js` step. Haiku for file moves, adoption of an existing tested helper, dead-code removal, import spelling; sonnet for single-file extraction/judgment; opus for the seams where a green suite can hide a semantic change (M1, M5, M9, M10, M11, T0-2, T0-3, T2-4, T2-6, P5, P6a).
- Prompts carry only the §5 rows the task needs, not the whole plan. Agents report ≤300 words plus evidence; the orchestrator reads diffs.
- Agents run the full suite at most twice per task (it is ~84 s); core-purity and the task's own test file are cheap and run freely.
- Each worktree needs its own `npm ci` (only `ws`; seconds). Never share `node_modules`.
- Escalation instead of retry: a failed review goes back to the same agent once with the specific finding; a second failure escalates one tier in the same worktree with the diff so far.
- Long-running visual tasks (T0-4, T2-6, M9, M11) drive the in-app browser themselves; the orchestrator only re-checks the shots it flags.

## 9. Verification (end to end)
- **Every task:** `V` (tests + build + core-purity), diff-set check, task-specific evidence from the DAG row.
- **Every gate:** helper-run `npm ci && npm test && npm run build` on the merged result; post-gate boot with clean console; 13 shots when UI was touched.
- **End of Phase 2:** `main.js` ≤ ~80 lines; every extracted module has an importing test; no test reads `main.js` as text except the automation snapshot; `tests/core-purity` green; 13 shots identical to `screenshots/refactor-baseline/`; `window.hallowmere` contract test green; perf within noise of T0-4.
- **End of Phase 3:** `scripts/perf-smoke.mjs` and the `PERF.md` browser table show before/after medians for every P-task; all visual comparisons identical; suite green; final gate merged.

## 10. Risks and mitigations
| Risk | Mitigation | Task |
|---|---|---|
| String-slice tests go silently green after extraction (`indexOf`→`-1`) | `sliceBetween` throws on missing marker; adopted before any extraction; §5.3 migration map checked at every M review | T0-1 |
| `$()` validator silently stops covering moved code | per-page closure validator, strictly stronger; sequenced after the dev-page move | T0-3 (after T0-5) |
| DOM-free core contamination breaks the server | core-purity test on every task; `random.js` gets its own purity assertion | T0-2, T0-7, T1-6, P5 |
| Unifying `environment.js` LCG shifts the overworld | proven bit-identical; proof test + obstacle-position golden hash | T0-7, T1-6 |
| chronicle.css deletion changes a menu state invisible to tests | docs-only map first; 39 byte-compared shot pairs; abandon on any unresolvable diff | T2-4/5/6 |
| Automation API drift | source snapshot test from Phase 0; contract test at M11; key/schema diff at each M review | T0-1, M11 |
| `pagehide` ordering (autosave → automation abort → preview dispose) | fixed wiring order in the composition block; registration-order tests with stub `windowTarget` | M5, M9, M11 |
| `applySnapshot` reordering (`!me` early return, first-snapshot key deletion, `switchMap` before `Object.assign`, event watermark) | extract verbatim; five-scenario test; statement-by-statement review | M10 |
| The split itself costs frame time | pre-split baseline; P6a re-measures | T0-4, P6a |
| Dropping `structuredClone` aliases live world objects | aliasing test lands first and gates the change | P5 |
| A "mechanical" adoption shifts a vertex | golden geometry hash per model captured on main before edits | T1-1..T1-4 |
| Parallel worktree contention on the helper lock | `start`/`finish` serialize on `workflow.lock`; the orchestrator runs them sequentially, never concurrently | 8.1, 8.4 |

## 11. Out of scope (explicit)
Asset recompression (WAV rate, og.png, inventory WebP), renderer quality settings (antialias, shadow map 2048², pixel ratio 1.7), any gameplay tuning, protocol changes, moving `dist/` source into subdirectories, adding a bundler or linter to CI.
