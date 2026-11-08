# Brain / Out — Technical Overview & Game Summary

[Brain / Out](https://brainout.org/) is a 2D multiplayer shooter with a nostalgic post-Soviet atmosphere. Players fight across destructible, tile-based maps using a large arsenal of modern weapons, across competitive team modes and a large open-world **Free Play** experience.

This document summarizes what the game is, how the repository is organized, and how the major technical systems fit together.

---

## Game summary

**Genre:** 2D top-down / side-view multiplayer shooter  
**Tone:** Post-Soviet nostalgia — factories, hospitals, bunkers, forests, stations  
**Core fantasy:** Fast sessions with guns, upgrades, cosmetics, and progression; plus a persistent Free Play sandbox with loot, quests, and exploration

### What you do

- Queue into short competitive matches (team objective, deathmatch, assault, etc.)
- Unlock and customize weapons, skins, badges, and other ownable content
- Play **Free Play** for longer sessions with inventory, day/night, generators, real estate, and quests
- Host or join lobbies; use the in-game map editor for custom content
- Play via desktop (LWJGL) or Steam builds; Android sources exist but are not the primary ship path in this tree

### Competitive modes (`GameMode.ID`)

| Mode ID | Class | Role in play |
|---------|--------|--------------|
| `normal` | `GameModeNormal` | Team ticket / points attrition (default competitive mode) |
| `domination` | `GameModeDomination` | Capture and hold flags for score |
| `assault` | `GameModeAssault` | Round-based attack/defend with alive counts and team score |
| `deathmatch` | `GameModeDeathmatch` | Free-for-all ticket pool |
| `foxhunt` | `GameModeFoxHunt` | Ticket-based hunt variant |
| `gungame` | `GameModeGunGame` | Progressive weapon ladder on kills |
| `ctf` | `GameModeCTF` | Capture-the-flag style scoring with “chips” |
| `duel` | `GameModeDuel` | 1v1 state machine (`waiting` → `spawn` → `steady` → `active` → …) |
| `zombie` | `GameModeZombie` | Team-vs-team zombie variant |
| `free` | `GameModeFree` | Free Play open world (day/night, no classic team enmity) |
| `lobby` | `GameModeLobby` | Main-menu / social lobby world |
| `editor` / `editor2` | `GameModeEditor` / `GameModeEditor2` | Legacy and newer map editors |

Matchmaking allow-lists live in `Constants.Matchmaking`: approved maps (factory, station, hospital, forest, bunker, canyon, …) and approved modes (`normal`, `deathmatch`, `domination`, `assault`, `foxhunt`, `gungame`), with a tighter competitive subset.

### Match lifecycle

Modes share a common phase machine:

`none` → `warmUp` → `game` → `aboutToEnd` → `finished`

Shared play-state wrappers: `PlayStateEmpty`, `PlayStateGame`, `PlayStateEndGame`.

---

## High-level architecture

Brain / Out is a **LibGDX** Java game with a classic **shared-core + client + dedicated server** split.

```
┌─────────────────┐     KryoNet (TCP/UDP)      ┌─────────────────┐
│  Client builds  │◄──────────────────────────►│  Game server    │
│  desktop/steam  │     + HTTP content port    │  server module  │
└────────┬────────┘                            └────────┬────────┘
         │                                              │
         └──────────────────┬───────────────────────────┘
                            ▼
                    ┌───────────────┐
                    │  core module  │  shared content, modes,
                    │               │  map, physics, messages
                    └───────┬───────┘
                            ▼
                    ┌───────────────┐
                    │  data/        │  packages, textures,
                    │  content pkgs │  weapons, maps assets
                    └───────────────┘
                            ▼
                    ┌───────────────┐
                    │ Anthill online│  login, matchmaking,
                    │ platform      │  profile, store, market…
                    └───────────────┘
```

### Gradle modules

| Module | Purpose |
|--------|---------|
| `core` | Shared game logic: content model, maps, physics helpers, game modes (shared state), networking message types, Kryo setup |
| `client` | Rendering, UI/menus, input, client mode realizations, voice chat, game states |
| `server` | Authoritative simulation, bots, plugins, HTTP content serving, server mode realizations |
| `desktop` | LWJGL3 desktop launcher |
| `steam` | Steam-integrated client launcher |
| `onlinelib-steam` | Steam online helper library |
| `android` | Android backend sources (present; not in active `settings.gradle` include set the same way as desktop/steam) |
| `data/` | Content packages (not a Java module; built via `make_data`) |
| `tools/` | Texture tools, ProGuard, particle editor, Steam builder helpers |

Entry points:

- Client: `BrainOutClient` (implements LibGDX `ApplicationListener`)
- Server: `BrainOutServer` (headless dedicated loop)
- Shared base: `BrainOut`

Version stamp: `Version.VERSION` / `TAG` (e.g. `valpha2` / `master`); env discovery at `https://env.brainout.org`.

---

## Runtime stack

| Layer | Technology |
|-------|------------|
| Language / JDK | Java, OpenJDK **11** |
| Game framework | **LibGDX** ~1.11 (forked `com.github.desertkun.libgdx`) |
| Physics | **Box2D** (+ Box2DLights on client) |
| Animation | **Spine** (bundled under `core`) |
| Networking | **KryoNet** + **Kryo 5** (TCP + UDP) |
| Online services | **Anthill Platform** runtime (`anthill-runtime-java`) |
| Build | Gradle Wrapper, ProGuard for release obfuscation |
| Content authoring | HJSON “smart” docs → packaged binary content |

Default local ports (`Constants.Connection`):

| Port | Role |
|------|------|
| `36555` | TCP game |
| `36556` | UDP game |
| `36557` | HTTP (map/content serving) |

---

## Client / server responsibilities

### Shared (`core`)

- Content definitions and serialization
- Map representation: block grid, actives, bullets, Box2D world
- Game mode *state* classes and phase timing
- Network message catalog (`common.msg.*`) and Kryo type registration
- User profile / rewards / room settings models

### Client

- Predicts and renders; owns menus (`menu`), game states (`gs`), controllers
- Per-mode `Client*Realization` classes
- Package verification (signed content); `--unsafe` skips signature checks for local builds
- `--offline` avoids Alpha network participation
- Voice chat (`VoiceChatManager`), social controller, Steam/desktop shells

### Server

- Authoritative world simulation and mode rules (`Server*Realization`)
- Bot AI task stack (`server/.../bot/Task*.java`)
- Plugins (rewards, leaderboards, team-kill kick, global conflict, …)
- Map sources and room lifecycle via Anthill `GameServerController`
- Roles: lobby (“main menu” world), active match, editor host

Mode logic uses a **realization pattern**: shared `GameMode` holds serializable state; `ClientRealization` / `ServerRealization` implement side-specific behavior while staying in sync over the network.

---

## World model

### Maps

Maps are chunked, block-based worlds (`Map` in `core`):

- **Blocks** — 16×16 tile sprites (`Constants.Graphics.BLOCK_SIZE`), destructible terrain
- **Actives** — players, flags, items, doors, spawners, lights, generators, safes, portals, …
- **Bullets** — projectile simulation
- **Physics** — Box2D world stepped at a fixed 1/60s accumulator
- **Dimensions** — named map dimensions registered globally for multi-map / Free Play layouts

Map files live under `bin/server/maps/` (dozens of `.map` assets: factory, bunker, CTF variants, Free Play loop/part maps, apartments, etc.). Client map upload path defaults to `~/brainout-maps`; override with `BRAINOUT_MAPS`.

### Content system

Almost everything gameplay-related is **data-driven `Content`**:

- Loaded through `ContentManager` / `PackageManager` from content packages
- Component-based (`ContentComponent` on definitions → runtime `Component` on instances)
- IDs like `weapon-ar15`, `bullet-5.56x45`
- Ownable progression: skins, medals, badges, battle pass, shop items, containers, quests

Major content families under `core/.../content/`: instruments (weapons, grenades), bullets, blocks, actives, shop, quests, battle pass, upgrades, consumables, teams.

---

## Content packages (`data/`)

Authoritative description: [`data/README.md`](../data/README.md).

| Package | Role |
|---------|------|
| `base` | Root strings, weapons, skins, blocks, shop, quests, … |
| `sounds` | SFX and music |
| `mainmenu` | Menu / GUI art |
| `freeplay` | Free Play environment, real estate, cards, intro |
| `lobby` | Lobby content |
| `halloween` / `custom` | Seasonal / custom packs |
| `map_*` | Per-map packages (bridge, canyon, factory, forest, hospital, siber, station, …) |

**Smart folders** author content as HJSON (JSON-like) documents. Example weapon fields: damage, fire-rate, recoil, clip size, allowed ammo, shoot modes. Build with:

```bash
./gradlew make_data
```

Built packages are **digitally signed**; local unsigned data requires the client `--unsafe` flag.

---

## Networking & online

### Transport

- **KryoNet** server/client with separate TCP and UDP channels
- Message types under `core/.../common/msg/client` and `.../server` (100+ message classes: movement/sync, shooting, chat, inventory, Free Play, editor, clans, store, …)
- Reliable UDP helpers (`ReliableUdpMessage`, `ReliableManager`)
- Proximity voice (`VoiceChatMsg`; distance constant 32 units)

### Online platform (Anthill)

Clients and servers discover services via Anthill (`Login`, `Profile`, `Game`, `Store`, `Market`, `Social`, `Leaderboard`, `Event`, `Promo`, `Blog`, `Report`, … — see `Constants.Connection.DISCOVER`).

Without keys in this repository:

- Use `--offline` on client and server for local play
- Use `--unsafe` when packages are locally rebuilt and unsigned
- IDEA run configs `Offline` / `Offline Unsafe` already pass these flags

Direct connect example (base64 of `localhost;36555;36556;36557`):

```text
--connect brainout://bG9jYWxob3N0OzM2NTU1OzM2NTU2OzM2NTU3
```

---

## Free Play

Free Play (`GameModeFree`) is a distinct product surface:

- No classic team enmity / warm-up; magazine management enabled
- Day/night cycle (`timeOfDay` / update speed)
- Large multi-part maps (`fr-loop-*`, `fr-part-*`, apartments, bunkers, …)
- Content: generators, exit doors, personal containers, real estate, quests, exchange menu
- Party join via `--free-play-join`

Server bots and Free Play-specific tasks live under `server/.../bot/` and `mode/freeplay/`.

---

## Progression & meta

- **User profiles** — `UserProfile`, levels, stats
- **Shop / store / market** — purchasable and tradable items
- **Containers / cases** — randomized rewards
- **Battle pass & events** — timed progression
- **Contracts, achievements, medals, profile badges**
- **Clans / social** — clan creation and friend list messages
- **Weapons** — large smart-folder catalog (AK family, AR platforms, SMGs, snipers, explosives, …) with skins and upgrades

---

## Tools & workflows

| Task | Command / note |
|------|----------------|
| IDEA project | `./gradlew idea` → open `brainout.ipr` (do not “import Gradle” in IDE) |
| Build assets | `./gradlew make_data` |
| Desktop client | `./gradlew desktop:dist` |
| Steam client | `./gradlew steam:dist` |
| Server | `./gradlew server:dist` |
| Particle editor | `particle_editor.sh` / `.bat` |
| Localization merge | `merge_localization.sh` |

Common local run configurations (from root README):

- `Server Lobby` — main menu world
- `Server` — active match
- `Server Editor` — editor host
- `Desktop Local Offline Unsafe` — client → localhost
- `Desktop Local 2 *` — second local user instance
- `Desktop Alpha` — Alpha network client

Troubleshooting: [`docs/Troubleshooting.md`](Troubleshooting.md).

---

## Repository layout (quick map)

```text
brainout/
├── core/          Shared simulation & content model
├── client/        Rendering, UI, client realizations
├── server/        Authoritative server, bots, plugins
├── desktop/       Desktop launcher
├── steam/         Steam launcher + deploy helpers
├── android/       Android sources
├── data/          Content packages (HJSON, textures, sounds)
├── bin/           Scripts, maps, launcher bits
├── tools/         Editors, ProGuard, GDX tools jars
├── docs/          Developer documentation (this file)
├── scripts/       Utility scripts
└── build.gradle   Multi-module Gradle build
```

Approx. scale: ~1,800+ Java source files across `core` / `client` / `server`.

---

## Browser / Web feasibility

**Short answer:** a browser *client* is possible in principle (same path as recent LibGDX → TeaVM / Wasm ports), but Brain / Out is **not** close to a drop-in web build today. Full multiplayer in-tab would require a networking rewrite and a large platform-abstraction pass.

### What already exists

Legacy **GWT** stubs remain (`core/src/BrainOut.gwt.xml`, `client/src/BrainOutClient.gwt.xml`) and the root buildscript still pulls `gwt-gradle-plugin` — leftovers from LibGDX’s old HTML5 path. There is **no** active `html` / TeaVM module, and shipping launchers are LWJGL3 (`desktop`, `steam`) plus Android stubs.

### Why this is harder than recent single-player browser ports

Recent breakthroughs (e.g. LibGDX games compiled with [gdx-teavm](https://github.com/xpenatan/gdx-teavm) to JS/Wasm) mostly prove: *render loop + assets + input in WebGL*. Brain / Out adds several hard constraints:

| Area | Desktop today | Browser reality |
|------|---------------|-----------------|
| Transport | **KryoNet** raw TCP + UDP (`36555`/`36556`) | Browsers expose **WebSocket** / WebRTC only — no arbitrary TCP/UDP sockets |
| Serialization | Kryo + heavy reflection | Must work under TeaVM/Wasm limits; often needs explicit registration / narrower reflection |
| Natives | Box2D, FreeType, controllers natives | Need web backends (`gdx-box2d` JS/Wasm, `gdx-freetype-web`, etc.) |
| Online HTTP | Anthill + Unirest-style JVM HTTP | Needs `fetch`/XHR-compatible client; CORS on all Anthill services |
| Files / packages | Signed zip packages on disk (`--unsafe` for local) | Preload into virtual FS / IndexedDB; rethink signing & updates |
| Voice chat | Custom UDP voice path | WebRTC or disable on web |
| Steam / editors / map publish | Desktop file + Steam APIs | Out of scope for first web slice |
| Custom LibGDX fork | `com.github.desertkun.libgdx` | Must align with whatever TeaVM backend expects |

The **dedicated server should stay a JVM process**. Only the client would target the browser; the server would need a WebSocket (or WebRTC) front door that speaks the same game messages (or a gateway that bridges WS ↔ existing KryoNet rooms).

### Realistic slices (least → most work)

1. **Offline / spectator-style web demo** — boot `BrainOutClient` via TeaVM, load a trimmed package set, maybe a local bot match or replay. Proves rendering, Spine, Box2D, UI.
2. **Web client ↔ gateway ↔ existing rooms** — replace KryoNet on the client with WebSockets; run a thin gateway that translates to the live TCP/UDP protocol (or dual-stack the server).
3. **First-class web platform** — asset CDN, login/CORS, voice, progressive download of `data/` packages, feature flags for editor/Steam-only flows.

### Verdict

| Goal | Feasible? |
|------|-----------|
| “Open a URL and see the game render” | Yes, with a new TeaVM (or revived GWT) module and asset pipeline work |
| “Play full Alpha multiplayer in Chrome like desktop” | Yes only after **transport + online + packaging** rework — not a weekend port |
| “Server in the browser” | No — keep authoritative servers on JVM |

Closest analogue among recent ports: single-player LibGDX → TeaVM titles that **disabled** multiplayer/editors for web. Brain / Out’s identity is multiplayer, so the networking rewrite is the real project.

---

## License & community

- License: MIT (Copyright 2022 desertkun) — see [`LICENSE`](../LICENSE)
- Site: [brainout.org](https://brainout.org/)
- Dev support Telegram and Discord links are in the root [`Readme.md`](../Readme.md)

---

## Related docs

- [Readme.md](../Readme.md) — build, run configs, CI environments (`main` / `test` / `dev`)
- [data/README.md](../data/README.md) — content packages and smart folders
- [Troubleshooting.md](Troubleshooting.md) — common setup failures
