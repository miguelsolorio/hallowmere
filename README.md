<div align="center">

  <h1>Hallowmere</h1>

  <img src="dist/assets/icons/app-icon.png" alt="Rounded Hallowmere app icon with the title below the shadowed Warden in cold steel, a dark cloak, and blue-green fog" width="160" height="160">

  <p>A dark fantasy action RPG built with Three.js. Travel between two villages, speak to four characters, fight randomized wilderness packs, collect and equip loot, silence the Bellkeeper in Hallowmere, then explore three harder regions connected by hidden underground passages.</p>

  <img src="screenshots/gameplay.jpg" alt="Hallowmere gameplay in Ashwick, with the Warden, torchlit village, quest tracker, minimap, and combat abilities">

  <p><em>Captured from the running game in Ashwick Village.</em></p>

</div>

## Run

Coding tasks use isolated worktrees and merge completed, validated changes into local `main`. See [the worktree workflow](docs/worktrees.md) for setup, development ports, and conflict recovery.

Requires Node.js 22 or newer. Run `npm ci` once to install the multiplayer server dependency. The pinned Three.js 0.180.0 runtime is included locally.

```sh
npm run dev
```

Open **http://127.0.0.1:5182**, or the printed Network URL to play from another device on the same network. In a task worktree, `npm run dev` uses that worktree's reserved port instead of 5182. The frontend is in `dist/`. Gameplay requires the shared WebSocket backend, included in the development server; a static host must point to a deployed backend. A WebGL2-capable browser is required. Sound begins after the first interaction.

The dev server restarts when its code or imported gameplay modules change. A restart creates a new vigil, and open game tabs reconnect automatically. `npm start` runs the production server without watching files.

If the game stays on the connection screen with a WebSocket error, check **http://localhost:5182/health**. It should return JSON with `ok: true` and `version: 3`. A 404 means an older or static-only server is still using port 5182: stop that process and restart with `npm run dev`. Reloading the browser alone cannot update a running server.

## GitHub Pages

The [Pages workflow](.github/workflows/pages.yml) tests and validates the game on pull requests and pushes to `main`. Pushes to `main` also deploy the checked-in `dist/` directory when a multiplayer backend is configured. You can redeploy from **Actions → Deploy game to GitHub Pages → Run workflow** on `main`.

For the first deployment, select **GitHub Actions** under **Settings → Pages → Build and deployment → Source**, then push `main`. The game will be available at **https://miguelsolorio.github.io/hallowmere/** after the deployment succeeds. Under **Settings → Secrets and variables → Actions → Variables**, set `MULTIPLAYER_SERVER_URL` to the deployed backend’s `wss://<host>/multiplayer` URL. A variable in the `github-pages` environment also works; a secret with the same name is accepted when no variable is set. This endpoint is public and is written into the frontend artifact.

You can also supply the public backend URL in **Run workflow** to override the configured value for one deployment. Save `MULTIPLAYER_SERVER_URL` as a variable for automatic deployments on future pushes; the manual input is not saved.

Code validation runs independently of deployment setup. If the URL is missing or blank, **Prepare Pages deployment** fails with an error and setup instructions in the run summary, so a deployment run cannot report success when configuration is missing. GitHub Pages cannot run the Node/WebSocket backend; that server must be deployed separately before publishing this multiplayer game. Invalid URLs, failed backend health checks, and incompatible protocols also fail preparation before uploading the frontend. The existing public game stays in place on failure. Deployment uses the workflow’s built-in GitHub token.

Keep local asset URLs relative so the game works both at a domain root and under a repository path such as `/hallowmere/`. `npm run build` checks these URLs before deployment. The workflow uses GitHub's [custom Pages deployment actions](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages).

## Controls

| Action | Control |
| --- | --- |
| Move | WASD / arrow keys / click ground |
| Target and cleave | Click a creature; hold to chase and attack |
| Stand and use primary attack | Shift + left click |
| Secondary class ability | Right click |
| Evade | 1 |
| Class skill | 2 |
| Healing draught | 3 |
| Speak / collect loot / harvest plants | F or click its label |
| Inventory, equipment, and food pouch | I |
| Eat forage | Eat button in the Inventory pouch |
| Change character (anywhere in solo; in a sanctuary in multiplayer) | C, your character name, or the pause menu |
| Reveal loot and plant labels during combat | Hold Alt |
| Journal / map / pause | J / M / Escape |

Touch devices have a movement stick and ability buttons with automatic enemy aiming. Red ground indicators show enemy attacks before they resolve. Evade grants brief invulnerability. Essence regenerates. Every defeated monster drops crowns; draughts and equipment can also drop. Walk over crowns, or use F/click a label to collect loot. Equip weapons and charms in the inventory for real damage and vitality bonuses. Brief browser focus changes do not open a menu. Menus and background tabs release movement input. Single player pauses; in multiplayer, the shared world continues. In multiplayer, enemies can still attack an idle character outside a sanctuary. A manually opened menu, map, or conversation stays open on return. Interrupted touch gestures reset the movement stick.

## Foraging

Harvest 6 outdoor patches with **F**, the interaction button, or a plant label. Clicking a distant label walks to it. Ashwick has one patch, Mourning Road three, and Hallowmere two, with two patches of each food across the world. Each plant gives one food, independently for each adventurer, and returns after three minutes. A full stack leaves the plant available.

Open **Inventory (I)** to use the pouch above your equipment satchel. It holds **five of each food**:

| Food | Effect |
| --- | --- |
| Crimson mushroom | Restore 35 current health |
| Moonleaf herb | Restore 40 extra essence over five seconds, alongside natural regeneration |
| Bramble berries | Restore 15 current health and 15 essence |

Foods share a two-second cooldown, separate from belt draughts. They never exceed maximum resources or raise your stats. Moonleaf cannot stack or refresh while active. Food is not consumed when all its affected resources are full; berries work if either resource is depleted. Inventory shows resources, food availability, and moonleaf duration. In multiplayer, the shared world keeps moving while inventory is open.

Your pouch and personal harvest timers survive death, respawn, and the normal 60-second reconnect window. Death clears active moonleaf regeneration; disconnecting pauses personal food effects and cooldowns. Plants keep regrowing with the world. A new vigil empties the pouch and restores all plants. These rules and range/visibility checks are authoritative on the server. Protocol 3 retains personal `forage` snapshots and `forage {id}` / `consume {itemId}` commands; update the backend and frontend together.

## Chosen roster

Every journey begins as the Sorcerer; switch to Ranger, Reaver, Nightblade,
Oathkeeper, Plague Alchemist, or Geralt from the pause menu whenever you like.
Each has a distinct model, weapon, combat kit, vitality,
essence, and movement speed. Sorcerer offers its crystal staff and spellbook,
plus the chosen Mire Witch, Bone Oracle, and Storm Hermit appearances.

The character wheel explains every skill. Press **C** or use the pause menu to
change character while preserving progress and upgrades; multiplayer requires a sanctuary. Health and essence percentages
and existing cooldowns carry across. Models, abilities, and support effects are
shared with other players; healing and protection can help nearby allies.
The [roster notes](docs/character-selection.md) list the live skills. The original
[character studies](dist/character-studies.html) remain available separately.

## Buildings

All twelve buildings have walkable rooms. Approach a front door with the movement stick or WASD, tap the door, or use the nearby Enter button. Roofs lift away inside; Leave guides you back through the doorway. Side and rear walls remain solid, including on rotated houses and during an evade. Hallowmere Chapel unlocks when the Bellkeeper is defeated.

## Campaign

Start in the safe village of **Ashwick**. Elder Rowan offers *The Last Toll*. Sister Edda restores vitality, essence, and a minimum of three draughts for free. Brann sells draughts and hones any equipped weapon for crowns.

Follow the cobbled **Mourning Road** east through three randomized packs (six to nine monsters). A new run creates a fresh seed, which changes the creatures, positions, and loot. Watchman Rook waits in a protected lantern ward at Hallowmere’s gate and shares supplies once.

Defeat the twelve afflicted in **Hallowmere** to summon **the Bellkeeper**, a 720-vitality boss with telegraphed ground attacks and radial projectiles. Road kills do not count toward the village objective. The boss always drops 60 crowns, two draughts, and **Bellkeeper’s Requiem**, a legendary class weapon with +18 primary attack damage. Collect the relic, equip it, and return to Rowan for an 80-crown quest reward. Exploration and trading continue after victory.

In multiplayer, each vigil is a shared session. “Vote for a new vigil” resets progress and rerolls the road only when all connected players agree. Votes expire after 30 seconds and cancel when membership changes. An empty world resets after ten minutes.

## Beyond Hallowmere

Defeating the Bellkeeper opens the eastern road at Hallowmere’s edge. His original relic and Rowan’s reward remain available independently of the new chapters.

| Region | Landmarks | Guardian | Guaranteed weapon |
| --- | --- | --- | --- |
| Drowned Wood | Three corrupted shrines | The Rootbound | Rootbound class weapon · +24 damage |
| Blackvein Quarry | Two broken lifts | The Quarry Warden | Blackvein class weapon · +30 damage |
| Crownfall Keep | Two ward anchors | The Ash Regent | Crownfall class weapon · +36 damage |

Clear a landmark’s original guards, then interact to begin two defense waves. The next wave arrives after the first is defeated. Interact again after the final wave to restore the landmark. Completing the region’s landmarks summons its guardian; defeating it opens the next region. Cleared enemies and completed landmarks stay cleared for the vigil. An unfinished boss resets when no living adventurer remains near its arena.

Hunters flank, rootlings snare, miners charge, quarry mages call down rocks, sentinels protect their front, and pyromancers create flame lanes. Ground warnings precede damage; move into the gaps or evade. Bosses expose themselves during recovery. The Quarry Warden can destroy arena cover, which returns when his encounter resets. The Ash Regent combines hazards in three health-based phases. New enemies scale health once at engagement for nearby teammates; distant players do not increase encounter health or damage.

Activate each region’s sanctuary lantern with **F**, click, or the touch interaction button to set your personal checkpoint, restore health and essence, and replenish at least three draughts. Class selection is available there. XP thresholds remain unchanged through level eight; subsequent levels require progressively more souls to keep later encounters threatening. Regional bosses also guarantee charms with +35 / +50 / +65 vitality.

### The Underways

Look for faint light, displaced stones, and overgrown rock openings in Hallowmere and each new region. Cave entrances appear on the map after proximity discovery, shared with all adventurers. Use the entrance to travel underground and its paired exit to return. Three connected cave chambers contain elite guards and personal caches; later passages remain sealed until the preceding surface guardian is defeated. Cave rewards are optional: each cache gives a regional charm and 25 / 50 / 75 crowns once per adventurer per vigil.

Travel, checkpoints, cave discoveries, landmark waves, and rewards are server-authoritative. Protocol 3 adds map identity, regional progress, nearby-map snapshots, hazard warnings, interactions, and broken-cover state, with `travel`, `objective`, `checkpoint`, and `cache` commands accepting an `id`. Update server and client together. Map travel cancels pending attacks and movement; combat and support effects cannot cross map boundaries. There are no durable saves; a new vigil or server restart resets the expansion along with the original campaign.

Regional playtime is a design target rather than a measured guarantee. Deterministic tests verify combat and traversal rules; a full player playthrough is still needed to calibrate the 15–20 minute first-visit target.

## Entering the game

After assets load, the game goes straight into single player: it continues your most recently saved journey, or on a first visit creates a new journey as the Sorcerer. There is no mode or character screen; change character from the pause menu (Escape) at any time. The game never connects to multiplayer unless the page is opened with `?mode=multiplayer`.

Single player runs the shared game simulation locally in your browser, with no multiplayer backend connection. Menus, character selection, and a hidden tab pause the local world. Progress autosaves to the browser as a journey; **Save & exit** in the pause menu opens **Your journeys**, where you can continue, rename, delete, or start another journey. Page assets must still load from the website (there is no offline installation).

Opening `/?mode=multiplayer` joins the existing shared world and retains its reconnection behavior. While the initial connection is pending or fails, **Play solo instead** returns to your journey. To change modes after entering a world, reload the page with or without the parameter.

## Cooperative multiplayer

Choosing Multiplayer joins one shared world with up to eight adventurers. Ground rings, numbered labels, and map markers identify each character. Movement and travel are independent: a player can enter the Mourning Road, Hallowmere, or an unlocked building while teammates remain elsewhere. After Hallowmere, adventurers can independently travel to Drowned Wood, Blackvein Quarry, Crownfall Keep, and the Underways. No party gathering is required at boundaries.

Enemies, quest acceptance, objectives, the boss, relic recovery, and chapel access are shared. Each connected player on the enemy’s map gets personal enemy loot and experience, while health, mana, currency, equipment, services, and reward claims remain individual. Late joiners inherit the world’s current quest state without historical enemy drops. There is no friendly fire or player collision. Death offers a return to the latest personally activated checkpoint (Ashwick by default) with health and mana restored, retaining gear and currency.

The Node server owns all gameplay outcomes. It simulates at 20 Hz and broadcasts at 10 Hz; the browser predicts walking, reconciles acknowledgments, and interpolates other actors. Messages carry a protocol version, world ID, and sequence number. The server enforces collisions, costs, cooldowns, line of sight, ownership, and interaction distances. `/health` reports protocol and connected-player count. Allowed origins, 4 KiB messages, per-connection rate limits, a join timeout, heartbeat checks, and bounded outbound buffers protect the connection layer.

A per-tab resume token restores the same character within 60 seconds of disconnection. Disconnected characters stop participating in combat and return safely to their activated checkpoint when resumed. Connection loss blocks controls until a fresh snapshot arrives. World and character state are held in memory: deployment or server restart starts a new vigil, with an explicit in-game notice. There are no accounts or durable saves.

### Public backend deployment

1. Create a Render Blueprint from this repository using [render.yaml](render.yaml), which selects a single **Free** Node instance. You can also create a Web Service from the public repository URL using the same build/start commands, environment variables, and `/health` check. Select **Free**, Node 22, and turn Auto-Deploy off. Keep the service at one instance, since multiple instances would create separate worlds. The declaration follows the [Render Blueprint reference](https://render.com/docs/blueprint-spec).
2. `ALLOWED_ORIGINS` defaults to `https://miguelsolorio.github.io`. Add any Sites frontend origin as a comma-separated value when serving the game there. Origins contain no paths. Keep automatic backend deployment off so a frontend push cannot unexpectedly reset an active adventure.
3. Deploy the backend and verify `https://<backend-host>/health`. Its response must include `ok: true` and `version: 3`. Render provides secure [WebSocket connections](https://render.com/docs/websocket).
4. Set the GitHub repository variable `MULTIPLAYER_SERVER_URL` to `wss://<backend-host>/multiplayer`, then run the Pages workflow. The workflow installs dependencies, tests, validates, checks the backend, and writes the public endpoint into the deployment artifact. Health checks retry transient failures up to eight times, with a 15-second timeout per request and five seconds between attempts, so a sleeping Free instance has time to wake. An incompatible protocol still fails immediately.
5. For another static host, run `MULTIPLAYER_SERVER_URL=wss://<backend-host>/multiplayer npm run configure:multiplayer` before uploading `dist/`. An empty endpoint uses the current origin, which is suitable for the included local server. Do not publish an empty endpoint to a static-only host.

Render's [Free service limits](https://render.com/docs/free) include sleeping after 15 minutes without inbound traffic, around a minute to wake, monthly usage caps, and possible restarts. Players may need to wait at the connection screen while it wakes. A restart starts a new vigil because this game stores world state in memory; a paid instance avoids idle sleep but does not add durable saves.

Adding this configuration alone does not provision a backend. For rollout, deploy a compatible backend first and the frontend second. Retain the previous frontend artifact for rollback; a protocol-incompatible backend requires restoring its matching server revision too. Expect a new vigil whenever the backend restarts. Monitor `/health` and logs for connection counts, simulation delays, and server errors; resume tokens are never logged.

## Generated assets

Eight original articulated GLB models are generated by `scripts/generate-assets.mjs`: Warden, Hollow, Grave Hound, Cinder Revenant, Bellkeeper, Elder, Healer, and Smith. The audio workshop produces 79 original synthesized PCM WAV assets in **Witchglass**, the user-selected sound direction from the ten-set audition. It covers combat, distinct creature voices, three footstep surfaces, doors, inventory, healing, bells, and evolving stereo ambience. Regenerate everything with `npm run generate`, or only sound with `npm run generate:audio`; the synthesis recipes are checked in and regeneration requires only Node.js.

The [Witchglass implementation notes](docs/witchglass-audio.md) explain the selected sound palette, variations, mix decisions, and verification. Hear the [29-second game-mixer preview](docs/witchglass-preview.wav), or revisit all ten directions on the [audition page](dist/sound-audition.html). Combat cues load first, followed by other effects and ambience; the full PCM bank is 24.8 MiB. Sound is positioned relative to the player, muffled through walls, and ducked beneath important attack cues. Ambience crossfades between regions, grows tense near enemies, and quiets after victory. Background tabs fade to silence and suspend audio.

The background score adds approximately 20 minutes of dark music by Scott Buckley: **Balefire**, **Descent**, **The Old Ones**, and **Incantation**. The playlist crossfades and repeats, with a separate Music switch in the game menu. Music pauses on mute or when the tab is hidden, resuming in place. Tracks stream from local compressed assets; [soundtrack notes](docs/soundtrack.md) cover playback and validation, and [soundtrack credits](dist/assets/music/CREDITS.txt) list the CC BY 4.0 attribution and source links.

The two villages and connecting forest use merged scenery geometry, instanced cobbles and grass, shadowed moonlight, animated torchlight, drifting ash, and a shader portal. Creature parts are merged by material while retaining animated limb groups.

The [1024 × 1024 app icon](dist/assets/icons/app-icon.png) was created with built-in ImageGen using the Warden artwork and village screenshot as references, then revised to match the game's cold blue-green shadows, weathered steel, and haunted atmosphere. It includes the Hallowmere title and rounded corners. Browser and home-screen exports live in `dist/assets/icons/`; the game uses them as its favicon and Apple touch icon. The [artwork notes](docs/app-icon.md) include the revision prompt. The gameplay screenshot above is a direct browser capture.

Social links use a [landscape Hallowmere card](dist/og.png) based on the existing Warden and village artwork. Open Graph also offers the [actual gameplay screenshot](dist/assets/social/gameplay.jpg) as an alternate image; Twitter uses the branded card. The metadata is included directly in `dist/index.html` and uses the public GitHub Pages URL. See [social preview notes](docs/social-previews.md) for asset sources, the generation prompt, and updating the public URL.

## Validation

```sh
npm test
npm run build
```

The checks cover combat geometry, cooldown and resource rules, health and death, collision and pathfinding, progression, seeded encounters and safe zones, loot collection and equipment, NPC services and quest rewards, GLB structure and numeric buffers, audio integrity, seamless loops, variation, spatial mixing, voice budgets, mute/pause behavior, failed-download recovery, script syntax, and asset/DOM references. Multiplayer tests additionally cover eight live WebSocket clients, private state, capacity, reconnects, simultaneous damage, quest completion, personal loot, independent travel, respawning, restart votes, stale input, and malformed connections.

Optional browser WebMCP integration exposes `get_vigil_state` and `control_warden` through the same game rules and controls. Browsers without it run the game normally.

## Attribution

Game code, environment, and creature meshes are original. The active Witchglass audio palette is original synthesis. Historical CC0 recordings from [Kenney Impact Sounds](https://kenney.nl/assets/impact-sounds) and [Kenney RPG Audio](https://kenney.nl/assets/rpg-audio) remain in the source archive but are not used by the active effects. Music by [Scott Buckley](https://www.scottbuckley.com.au) is licensed under CC BY 4.0; track attribution is available in the game menu and `dist/assets/music/CREDITS.txt`. Publisher license text and a file-by-file effects source manifest are in `scripts/audio-sources/`; runtime credits are in `dist/assets/audio/CREDITS.txt`. [Three.js](https://threejs.org/) is MIT licensed; its license is included at `dist/vendor/LICENSE.three`. Fonts are Cinzel and Inter from Google Fonts. The GLB loading setup follows the [official Three.js GLTFLoader documentation](https://threejs.org/docs/pages/GLTFLoader.html).
