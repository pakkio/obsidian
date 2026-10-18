---
tags:
  - craftstorm
  - viewer
  - webgl
  - threejs
  - firestorm
  - opensim
  - moc
  - index
aliases:
  - CraftStorm Overview
  - pakkio/craftstorm
---

# CraftStorm

← Back to [[Benvenuto]] | Companion Server: [[CraftSim]]

> **Updated 2026-10-18 (HEAD `7cbfa9609b`).** Web viewer and native notes were verified at commit `bba1a8b8f6` (2026-10-16); since then only `web/` bridge code changed (sharding, C extension, asset cache - see [[CraftStorm Bridge - Scaling, Cluster & csfast]]). `app.js` is untouched (12,778 lines, same build id). Working checkout is now `C:\Users\claudio.pacchiega\w\craftstorm`.
>
> Originally verified (`bba1a8b8f6`, 2026-10-16). Repo `git@github.com:pakkio/craftstorm.git` (remote `origin`), local `C:\Users\pakkio\w\craftstorm`, branch `main` (remote also has `origin/master`). Hub notes below were each re-checked against source; this page only summarises and links.

**CraftStorm** is pakkio's two-track viewer for OpenSim / [[CraftSim]] / Second Life grids:

1. **Web viewer (`web/`)** - a zero-install browser viewer (also wrapped as a `pywebview` desktop app), `APP_BUILD = 'v2.4.2 (2026-10-15)'` (`web/app.js:37`), built on **three.js `0.186.0`** and **three-mesh-bvh `0.9.15`** (`web/package.json`; both vendored in `web/vendor/`), paired with a Python `asyncio` bridge (`web/bridge.py`, now 7,036 lines) that speaks LLUDP/XML-RPC/CAPS to the grid.
2. **Native C++ viewer (`indra/`)** - a Firestorm fork (target `craftstorm-bin`) with a small set of OpenSim fixes and branding (see the C++ note). History contains merges from `upstream/master` and `upstream/release/2025.07`.

---

## Map of content

### 1. Web viewer
- **[[CraftStorm Web - Architecture, Rendering & Batching]]** - three.js pipeline in `web/app.js` (12,778 lines), SL Z-up to three.js Y-up conversion, instancing and static batching (`web/assets/static_batch.js`, region-sized `BATCH_CELL_SIZE = 256`, `THREE.BatchedMesh` via `WEBGL_multi_draw`), prim sweeps (`volume_build.js`), sculpts (`sculpt_build.js`), `MeshBVH` (`bvh.js`, `bvh_worker.js`), cutout alpha.
- **[[CraftStorm Web - Performance, Budgets & Diagnostics]]** - frame budgets (`MSG_BUDGET_MS = 6`, `CULL_BUDGET_MS = 3`, `app.js:7319,7379`), `WEAK_GPU` detection (`:109`), load shedding (`shedLevel`, `autoReduceLoad` `:11385-11399`), GPU timer (`assets/gpu_timer.js`), `P` probe, `F9` benchmark (`assets/benchmark.js`), `Diag` JSON.
- **[[CraftStorm Web - Avatars, Animesh, Movement & Audio]]** - avatar skeleton data (`web/avatar_data/skeleton.json`: 160 joints), bakes-on-mesh, animesh and keyframe animation (`assets/anim_clip.js`, `anim_state.js`, `anim_frame.js`, `anim_rest.js`, `anim_root.js`), `collision.js`, `assets/fly_speed.js`, spatial voice (`voice/voice_client.js` talking to [[Voice (WebRTC, Janus & SynthVoice)|wolfvoice]]), `assets/world_audio.js`.
- **[[CraftStorm Web - Controls, UI, Build Tools & Shortcuts]]** - real key map, the 10 implemented slash commands (plus aliases), window manager, inventory (Rez/Wear/View, notecard save), numeric object editor. Note it has **no 3D gizmo and no LSL editing**.

### 2. Bridge and MCP
- **[[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]]** - HTTP/WS on `8088`, optional SNI HTTPS on `8443`, CAPS list, asset endpoints (`/texture /mesh /anim /sound /music`), disk caches under `~/.craftstorm-cache`, and the opt-in **15-tool** `POST /mcp` endpoint (`MCP_TOKEN`). Split notes: [[CraftStorm Bridge - Assets & Caches]], [[CraftStorm Bridge - MCP Server]]; see also [[MCP Server & Live Diagnostics]]. **New:** `web/cluster.py` runs N bridge shards behind a sticky front with a shared asset cache, plus an optional `csfast` C extension for hot paths ([[CraftStorm Bridge - Scaling, Cluster & csfast]]).

### 3. Native viewer
- **[[CraftStorm Native C++ Viewer (Firestorm Fork)]]** - evidence-based list of fork changes: UDP texture-fetch fallback and NPOT fix, 10 s avatar-name expiry, blue clickable contour, Land-Impact tooltip, Zen Mode (Ctrl-Z), 8 px blue frame, `craftstorm` skin, rebrand.

---

## Verified repo layout (top level, commit `bba1a8b8f6`)

| Path | What it is |
| :--- | :--- |
| `web/` | The web viewer + bridge (below) |
| `indra/` | Firestorm/SL viewer C++ tree (`newview`, `llmessage`, `llrender`, `llui`, `llcharacter`, `llappearance`, `llimage`, `llcorehttp`, `llwebrtc`, ...) |
| `scripts/` | Stock viewer tooling plus fork additions: `configure_firestorm.sh` (build entry point), `content_tools/` (`anim_tool.py`, `skel_tool.py`, `dae_tool.py`, `arche_tool.py`, `extract_avatar_web.py` which regenerates `web/avatar_data`), `perf/`, `messages/` (message template), `testing/lsl` |
| `build.sh`, `build_target.sh`, `buildscripts_support_functions`, `autobuild.xml`, `requirements.txt` | Upstream build-farm scripts and package manifest (`build.sh` is not a custom CraftStorm script) |
| `build-vc170-64/` | Local Visual Studio 2022 build tree (pre-rebrand cache; see C++ note) |
| `doc/` | Upstream Firestorm docs: `building_windows.md`, `building_macos.md`, `building_linux.md`, licences, test plans |
| `AGENTS.md`, `GEMINI.md`, `CONTRIBUTING.md`, `README.md` | `README.md` is still the **upstream Firestorm README**; `AGENTS.md` grants an agent broad autonomy for build/test/file ops (do not treat as spec) |
| `tasks/todo.md` | Working notes (currently an animesh-horse-offset handoff plan) |
| `graphify-out/`, `web/graphify-out/` | Generated knowledge-graph output |
| `.github/workflows-disabled/` | Inherited upstream CI, disabled by the fork |

### `web/`
| Path | Role |
| :--- | :--- |
| `bridge.py` | Bridge + MCP server (supported runtime; one process) |
| `cluster.py`, `nginx.cluster.conf.sample` | N bridge shards behind a sticky front with response cache |
| `csfast.c`, `setup_fast.py`, `Makefile`, `justfile` | Optional C hot paths (`make`/`just build-fast`); Cython build via `build_bridge_c.sh` + `bridge_compiled.py` (no measured gain); `requirements.txt` |
| `bridge.js`, `httpClient.js`, `llsd.js`, `png.js`, `xml.js`, `openjpeg.js` | Experimental Node port of the bridge (no MCP/TLS/anim/sound) |
| `desktop.py` | pywebview desktop shell around `bridge.py` |
| `j2k_lenient.py` | ctypes OpenJPEG lenient J2K decoder |
| `index.html`, `app.js` | Page/HUD/CSS and the 12.8k-line viewer |
| `assets/` | ES modules: batching, BVH + workers, mesh/geo workers, volume + sculpt builders, anim modules, audio, GPU timer, texture LOD/format, benchmark; plus `terrain_detail_*.png` |
| `runtime/` | `bridge_client.js` (WebSocket client), `message_queue.js` |
| `scene/` | `object_registry.js` |
| `input/` | `chat_commands.js` |
| `ui/` | `windows.js`, `inventory.js`, `editor.js`, `im_panel.js` |
| `voice/` | `voice_client.js` |
| `avatar_data/` | Skeleton/joint JSON + default bake PNGs |
| `vendor/` | three.js r186 + three-mesh-bvh |
| `collision.js`, `spatial_hash.js`, `wireframe.js` | Client-side collision and helpers |
| `tests/` | 11 `*.test.mjs` (118 tests as of 10-16) + 16 `test_*.py` (144 tests on 2026-10-18; on Windows 2 POSIX-only tests fail: flock, selector) + `loadtest/` rig |
| `HANDOVER_TODO.md` | 2026-09-08 review; partly fixed already - treat as history |

---

## Quick reference (checked against source)

| Component | Source | Value |
| :--- | :--- | :--- |
| Web build id | `web/app.js:37` | `v2.4.2 (2026-10-15) - other avatars and NPCs stand on the floor ...` (matches HEAD commit `bba1a8b8f6`) |
| three / BVH | `web/package.json` | `three 0.186.0`, `three-mesh-bvh 0.9.15`, `@cornerstonejs/codec-openjpeg ^1.3.3` (Node port) |
| Run web viewer | `web/bridge.py` | `python3 web/bridge.py` then `http://localhost:8088/` (`python web/desktop.py` for the desktop shell). Multi-user: `python3 web/cluster.py` (`CRAFTSTORM_SHARDS`, default `min(8,cpus-1)`) |
| Ports | `BRIDGE_PORT`, `BRIDGE_TLS_PORT` | HTTP `8088` (default, env-overridable; cluster shards on `18100+`), HTTPS `BRIDGE_TLS_PORT` default `8443` |
| MCP | `bridge.py:447` | off unless `MCP_TOKEN` set; Bearer auth; `POST /mcp`; 15 tools |
| Cache | `CACHE_DIR` in `bridge.py` (shared by shards; `locks/` added) | `~/.craftstorm-cache`: `textures/v3-512q88`, `meshes/v3`, `anims/v1`, `objects/v4`, `sounds`; 7-day idle sweep |
| Tests | `web/package.json`, `web/tests/` | `cd web && npm test`; `python -m unittest discover -s tests -p 'test_*.py'` |
| Native binary | `indra/newview/CMakeLists.txt:948` | `craftstorm-bin`; configure with `scripts/configure_firestorm.sh` (OpenSim default on) |
| Active skin | `indra/newview/app_settings/settings.xml:15319` | `craftstorm` |

---

## Corrections vs previous doc
- Bridge MCP has **15** tools (not 13); `/mesh` supports only low/high; no built-in animation table; `bridge.js` lacks MCP/TLS/anim/sound.
- Controls: several documented slash commands (`/help /pos /who /sit /stand /s /w /me /<ch>`) do not exist; editor has no gizmo; no LSL script editing; windows key is `craftstorm_windows`.
- C++: `build.sh --opensim/--sl` is not a CraftStorm script (it is stock LL; use `scripts/configure_firestorm.sh`); Zen Mode implementation and Land-Impact behaviour were mis-described; the active skin `craftstorm` is blue, teal/indigo belongs to `skins/default`.
- Index wording "130+ bone rig / 26 collision volumes": the skeleton data has **160 joints**; the "26 fitted-mesh collision volumes" figure is not re-verified here (unverified - see the Avatars note).
- "`OBJECT_ANIMATED_MESH = 0x80000`" could not be found as a named constant in `web/app.js` or `bridge.py` (the animesh flag is read from extended params; see the Avatars and Bridge notes) - removed from this page.
- "`origin/main` at `bba1a8b8f6`" confirmed; line counts refreshed (`app.js` 12,778; `bridge.py` 6,508).
- 2026-10-18: the bridge is no longer single-process-only or hard-coded to port 8088; `bridge.py` grew 6,508 -> 7,036 lines, so line numbers in the bridge note are stale. Python test count refreshed (144).
