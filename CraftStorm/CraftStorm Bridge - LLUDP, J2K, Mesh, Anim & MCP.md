---
tags:
  - craftstorm
  - bridge
  - python
  - lludp
  - jpeg2000
  - mcp
  - tls
aliases:
  - CraftStorm Bridge
  - bridge.py
---

# CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP

> Hub for the bridge: process model, TLS, login and LLUDP circuit live here; assets, MCP and scaling have their own notes.

← Back to [[CraftStorm]] | Related: [[CraftStorm Web - Architecture, Rendering & Batching]], [[MCP Server & Live Diagnostics]], [[Voice (WebRTC, Janus & SynthVoice)]], [[CraftSim]]

> **Updated 2026-10-18 for commit `7cbfa9609b`.** Sections 1-8 were verified at `bba1a8b8f6` (2026-10-16) when `bridge.py` was 6,508 lines; it is now **7,036 lines**, so every line number quoted below has drifted (use the function names). What changed since is summarised in section 9 and detailed in [[CraftStorm Bridge - Scaling, Cluster & csfast]]. Original check was against `web/bridge.py`, `web/bridge.js`, `web/desktop.py`, `web/j2k_lenient.py` and `web/tests/`. Anything not confirmed is marked "(unverified)". Line numbers are for this commit and will drift.

Browsers cannot open raw UDP sockets (LLUDP) or decode Second Life JPEG2000 textures, so **`web/bridge.py`** is an `asyncio` protocol translator (one process per shard; `cluster.py` runs several, see section 9): XML-RPC login, one UDP circuit per browser tab, CAPS/HTTP asset fetching and transcoding, versioned disk caches, a hand-rolled HTTP/WebSocket server and an opt-in MCP endpoint. HTTP and WebSocket code is stdlib, but the process needs third-party packages (section 8).

---

## 1. Process model and entry points

| What | Where | Detail |
| :--- | :--- | :--- |
| Run | `python3 web/bridge.py` | `asyncio.run(main())` (`bridge.py:6507`). Page at `http://localhost:8088/` |
| Desktop shell | `web/desktop.py` (48 lines) | Runs `bridge.main()` in a daemon thread, waits for `bridge.PORT`, opens a `pywebview` window "CraftStorm" (`1440x900`) on `http://127.0.0.1:8088/`. With no console, stdout/stderr go to `%LOCALAPPDATA%\CraftStorm\craftstorm.log` |
| Node port | `web/bridge.js` (2,277 lines) | **Experimental** (package.json description; `HANDOVER_TODO.md #15`). HTTP+WS on `8088` only; HTTP routes `/texture/` and `/mesh/` only (`bridge.js:2151-2156`). No `/anim`, `/sound`, `/music`, no MCP, no TLS, no disk cache (0 hits for `craftstorm-cache`/`homedir`). WASM OpenJPEG (`@cornerstonejs/codec-openjpeg`) + hand-written `png.js`. Supported runtime is `bridge.py` |
| Logging | `_log()` `bridge.py:67` | stdout only, timestamped |
| Listeners | `main()` `bridge.py:6427` | HTTP `PORT = 8088` (default, now `BRIDGE_PORT`; bound to `BRIDGE_HOST`); optional HTTPS on `BRIDGE_TLS_PORT` (default `8443`) |
| Multi-user | `ACTIVE_SESSIONS`, `_active_session(agent_id)` `bridge.py:365` | One `Session` per WebSocket; asset endpoints accept `?agent_id=` to pick whose CAPS to use. `LAST_LOGGED_IN_SESSION` is what MCP acts on |

### HTTP routing (`handle_conn`, `bridge.py:6192`)
Non-WebSocket requests: `POST /mcp` -> `serve_mcp`; `/texture/<uuid>`, `/anim/<uuid>`, `/sound/<uuid>`, `/mesh/<uuid>` (UUID validated, `400` otherwise); `/music?url=`; anything else -> `serve_static` (files under `web/`). A WebSocket upgrade goes to `handle_ws_client` (`:6271`), which JSON-decodes each text frame and calls `dispatch()` (`:6302`).

### Environment variables actually read
`MCP_TOKEN`, `BRIDGE_TLS_CERT`, `BRIDGE_TLS_KEY`, `BRIDGE_TLS_PORT`, `CRAFTSTORM_CACHE_DIR`, `CRAFTSTORM_CACHE_MAX_AGE_DAYS` (7), `CRAFTSTORM_TEXTURE_SIZE` (512), `CRAFTSTORM_TEXTURE_QUALITY` (88), `CRAFTSTORM_WEBP_QUALITY` (85), `CRAFTSTORM_TEX_MEM_CACHE` (512), `CRAFTSTORM_MESH_MEM_CACHE` (256), `CRAFTSTORM_ANIM_MEM_CACHE` (128), `CRAFTSTORM_DRAW_DISTANCE` (100), and `CRAFTSTORM_OPENJP2` (in `j2k_lenient.py`). Added since: `BRIDGE_PORT` (8088), `BRIDGE_HOST` (`0.0.0.0`), `CRAFTSTORM_LOG_PACKETS` (0), `CRAFTSTORM_DECODE_WORKERS`, `CRAFTSTORM_PACKET_YIELD_EVERY` (8), `CRAFTSTORM_PREFETCH` (1), `CRAFTSTORM_PREFETCH_CONCURRENCY` (3); plus the cluster variables in [[CraftStorm Bridge - Scaling, Cluster & csfast]]. So the port is **no longer hard-coded** to 8088.

---

## 2. TLS listener with SNI and hot reload (`bridge.py:76-92, 6444-6505`)

- Needed because browsers only grant the microphone (WebRTC voice, [[Voice (WebRTC, Janus & SynthVoice)]]) to secure origins.
- `BRIDGE_TLS_CERT` / `BRIDGE_TLS_KEY` are **comma-separated lists** in the same order, zipped into `TLS_PAIRS`. One `ssl.SSLContext` per pair; the first is the default and, with more than one, `default_ctx.sni_callback = _pick` swaps `sock.context` to the context whose `subjectAltName` DNS names (`_cert_names`) match the SNI host.
- `_tls_reload_loop` sleeps `3600 s`, compares `(mtime(cert), mtime(key))` and re-runs `load_cert_chain` in place on change (open connections keep the old cert). Added in commit `8149100e2a`.
- HTTPS start failure is logged and does not stop HTTP; `fatal: no ports could be bound` only if nothing bound.

---

## 3. Login, capabilities and the LLUDP circuit

1. **`Session.do_login`** (`bridge.py:1109`): browser sends `{type:"login", loginUri, first, last, passwd, startType, region, startX/Y/Z}`. `dispatch` maps `startType` to `"home"`, `"last"` or `uri:<region>&x&y&z` (custom defaults 128,128,25). XML-RPC runs in an executor (`xmlrpc_login`). On success it stores agent/session ids, `circuit_code`, sim ip/port, inventory root and `region_size_x/y` (varregions), connects a non-blocking UDP socket, wipes scene state, then emits `scene_reset` and `login_ok {firstName,lastName,agentId,region,startPos,regionSize}`.
2. **Handshake** (`_handshake`, `:1446`): reliable `UseCircuitCode`, 0.2 s pause, reliable `CompleteAgentMovement`. `_recv_loop`, `_agent_update_loop`, `_visibility_loop` start.
3. **Capabilities** (`CAPS_WANTED`, `:1198`): the seed cap is POSTed with exactly: `GetTexture, SetDisplayName, GetDisplayNames, GetMesh, GetMesh2, ViewerAsset, ProvisionVoiceAccountRequest, VoiceSignalingRequest, ObjectAnimation, FetchInventoryDescendents2, UpdateAvatarAppearance, UploadBakedTexture, UpdateNotecardAgentInventory, UpdateNotecardTaskInventory, EventQueueGet`. Merely *requesting* `ObjectAnimation` makes OpenSim enable its ObjectAnim cap flag (otherwise no `ObjectAnimation` packets). Up to 6 attempts honouring `Retry-After` on `503`. `get_mesh_url = GetMesh2 or GetMesh or ViewerAsset`. `EventQueueGet` starts `_event_queue_loop` (OpenSim delivers `ParcelProperties` - parcel name, music URL - only that way).
4. **Wire format**: `FLAG_ZEROCODE 0x80`, `FLAG_RELIABLE 0x40`, `FLAG_RESENT 0x20`, `FLAG_ACK 0x10`; the `MSG` table maps (frequency, number) for every handled message. `_handle_packet` (`:2299`) strips appended acks, zero-decodes, ACKs reliable packets, dispatches by name.
5. **Received and handled**: `StartPingCheck`, `ChatFromSimulator`, `AvatarAnimation`, `ObjectAnimation`, `SoundTrigger`/`AttachedSound`/`AttachedSoundGainChange`/`PreloadSound`, `ImprovedInstantMessage`, `ObjectUpdate`, `ObjectUpdateCompressed`, `ObjectUpdateCached`, `ImprovedTerseObjectUpdate`, `KillObject`, `AgentMovementComplete`, `TeleportLocal`, `TeleportFailed`, `MapBlockReply`, `RegionHandshake`, `AvatarAppearance`, `LayerData`, `ObjectPropertiesFamily`, `ObjectProperties`, `ReplyTaskInventory`, `SendXferPacket`, `SimulatorViewerTimeMessage`.
6. **Sent**: `UseCircuitCode`, `CompleteAgentMovement`, `AgentUpdate`, `AgentThrottle`, `RegionHandshakeReply` (**Flags = 0**, `:2635`), `ChatFromViewer`, `ImprovedInstantMessage`, `ObjectGrab`/`ObjectDeGrab`, `AgentRequestSit`/`AgentSit`, `TeleportLocationRequest`, `MapNameRequest`, `RequestMultipleObjects`, `RequestObjectPropertiesFamily`, `LogoutRequest`, `RezObject`, `RezSingleAttachmentFromInv`, editor set (`MultipleObjectUpdate`, `ObjectName`, `ObjectDescription`, `ObjectSelect`/`ObjectDeselect`, `ObjectFlagUpdate`, `ObjectExtraParams`), `RequestTaskInventory`, `RequestXfer`, `ConfirmXferPacket`, `PacketAck`, `CompletePingCheck`.
7. **AgentUpdate**: `_agent_update_loop` sends one every `0.1 s` and survives send failures (logs the 1st and every 50th). `Far` = the session's `DROP_DISTANCE` so the sim culls upstream; `CameraCenter` = browser `move.cameraPos` or the avatar. `AgentThrottle` is sent after `RegionHandshake` with 7 floats (Resend 150k, Land 150k, Wind 20k, Cloud 20k, Task 400k, Texture 400k, Asset 200k bps) so terrain gets bandwidth.
8. **Terrain**: `LayerData` types `L` (16x16) and `M` (32x32 var-region) are decoded by `_BitReader` plus a numpy IDCT (`_idct_patch`) and emitted as `terrain_patch {pi,pj,size,heights}`; wind/cloud/water layers are ignored. Grid default terrain detail textures are served from bundled `web/assets/terrain_detail_*.png` (`TERRAIN_DETAIL_LOCAL_FILES`) and never hit the grid.
9. **Streaming** (`_visibility_loop`, `:3486`): 64 m spatial cells (`VISIBILITY_CELL_SIZE`), polled every `0.25 s` (`VISIBILITY_INTERVAL`), at most `60` objects released per tick (`VISIBILITY_BUDGET`), nearest-first within `DRAW_DISTANCE`; objects beyond `DROP_DISTANCE = 1.5 x DRAW_DISTANCE` get `kill`. Default `DRAW_DISTANCE` 100 m; browser `set_draw_distance {meters}` clamps to `10..256` (`:6350`). The HUD slider's initial value in `index.html` is `20`, not 100.
10. **Browser batching** (`_flush_browser_updates`, `:1027`): waits 1/60 s then sends `objects` (full dicts) and `object_transforms` (flat arrays via `pack_transform`); loops/reschedules so updates arriving mid-send are not stranded.

### Browser <-> bridge JSON messages
- **Browser -> bridge (`dispatch`)**: `login, chat, im, set_display_name, voice_cap, move (keys,yaw,cameraPos), set_draw_distance, touch, sit, stand, diag, map_search, teleport (x,y,z), obj_select, obj_deselect, obj_edit, obj_contents, obj_item_text, save_notecard, obj_flags, obj_extra, inv_fetch, inv_rez, inv_asset, inv_wear, request_object_properties`. `voice_cap` runs as a session background task so slow voice HTTP never blocks movement.
- **Bridge -> browser** (`"type"` values emitted): `login_ok, scene_reset, objects, object_transforms, kill, terrain_patch, region, region_handle, sun, parcel, chat, im, display_name, display_name_result, avatar_anim, object_anim, avatar_tex, attached_sound, sound_trigger, sound_preload, sound_gain, object_properties, obj_contents, inv_folder, inv_asset, inv_result, inv_error, notecard_saved, map_blocks, teleport_result, self_pos, voice_caps, voice_cap_response, clear_cache, error`.

---

## 4-5. Asset endpoints and on-disk caches

Moved to [[CraftStorm Bridge - Assets & Caches]]: `/texture /mesh /anim /sound /music`, the J2K decode chain, transcoding, `~/.craftstorm-cache` layout, memory LRUs, `ObjectUpdateCached`.

---

## 6. Built-in MCP server

Moved to [[CraftStorm Bridge - MCP Server]]: opt-in `POST /mcp` (`MCP_TOKEN`), protocol, and the 15 tools.

---

## 7. Tests (`web/tests/`)

- Python (stdlib unittest): `cd web && python -m unittest discover -s tests -p 'test_*.py'` -> **152 tests** (2026-10-19: all pass on Linux; on Windows all pass with 13 skipped - `FetchLockTest` is skipped there because the lock is a no-op without `flock`, and `test_hot_paths` checks selector entries only on selector loops. The two failures seen on 2026-10-18 were a real Windows regression, see [[CraftStorm Bridge - Scaling, Cluster & csfast]] section 3). Files: `test_agent_loops, test_anim, test_animesh_flag, test_asset_serving, test_csfast, test_decode_pool, test_handover_fixes, test_hot_paths, test_im_sessions, test_mcp_tools, test_mesh_vertex_decode, test_object_omega, test_region_handshake, test_sun, test_terrain_idct, test_texture_decode`.
- JS: `cd web && npm test` (= `node --test tests/*.test.mjs`) -> **118 tests, all pass**. Files: `anim_frame, anim_rest, anim_root, app_internals, browser_flows, fly_speed, gpu_timer, handover_fixes, sculpt_build, static_batch, volume_build`.

---

## 8. Dependencies, limits and known gaps

- `bridge.py` imports `Pillow`, `imagecodecs`, `numpy`, `llsd`, and optionally `orjson` and `csfast`. `web/requirements.txt` declares `Pillow>=11.3.0`, `imagecodecs>=2026.8.16`, `numpy>=2.0` `llsd>=1.2.4` and optional `orjson>=3.10` (`llsd` was imported at start-up but missing from the file until `50f88d876f`, 2026-10-19: a clean install gave a bridge that would not start). Optional: `pywebview` (desktop), `opj_decompress`.
- Node port deps (package.json): `@cornerstonejs/codec-openjpeg`, `three 0.186.0`, `three-mesh-bvh 0.9.15`.
- Strict-J2K failures need the OpenJPEG DLL/so (for `j2k_lenient`) or `opj_decompress`; otherwise the texture `404`s. On Linux `j2k_lenient` also finds the `libopenjp2` bundled in the `imagecodecs`/`Pillow` wheels (since `4cbfc8ca33`), which also powers the reduced-resolution decode.
- Mesh LOD selection is binary despite four LOD blocks in the asset. Disk texture cache stores only the 512 px auto variant (graded/`webp` variants are RAM only).
- MCP has no inventory, parcel or continuous-movement tools and only targets the most recent login.
- `web/HANDOVER_TODO.md` is a 2026-09-08 review list (now with a 2026-10-18 section on the csfast work); at least three of its items (voice_cap blocking the reader loop, flush rescheduling, stale WebSocket callbacks) appear already addressed in code (`dispatch` voice task, `_flush_browser_updates`); treat it as history, not status.

---

## Corrections vs previous doc
- MCP tool count is **15, not 13**; `clear_cache` does **not** delete `sounds/`; `bridge_status` returns cache entry counts plus session summary.
- `/mesh` LOD is **low vs high only**, not four selectable LODs.
- **No `BUILT_IN_ANIMS` / "36 built-in animations"** exists; anim assets are always fetched from grid caps (`asset_id|mesh_id|texture_id`).
- CAPS list was wrong: `ParcelVoiceInfoRequest` and `UpdateScriptTask` are **not** requested; actual list in section 3 (adds `ObjectAnimation`, `FetchInventoryDescendents2`, `UpdateAvatarAppearance`, `UploadBakedTexture`, `UpdateNotecardAgentInventory`, `EventQueueGet`).
- `RegionHandshakeReply` is sent with **Flags = 0**; `FLAGS_SUPPORTS_SELF_APPEARANCE = 4` does not appear in the code. The flag constants are `FLAG_ZEROCODE/RELIABLE/RESENT/ACK`, not `MSG_*`.
- Object cache key is sim IP + region name (not "grid_host"); disk textures hold only the 512 px auto encode.
- `/music` is an allow-listed relay, not a general proxy; `/sound` sniffs the `OggS` magic.
- `web/bridge.js` has no MCP, TLS, anim, sound or disk-cache support (previous text implied parity).
- The docstring says the server is stdlib-only for HTTP/WS, but the process also needs Pillow, imagecodecs, numpy and llsd.
- Line numbers refreshed (file is 6,508 lines; MCP block `5676-6190`; `set_draw_distance` at `6350`).

---

## 9. Scaling (2026-10-17/18)

`web/cluster.py` now runs N bridge shards behind a sticky front with a shared asset cache; an optional C extension (`csfast`) speeds the hot paths; decode moved to a process pool. Details, env vars and measurements: [[CraftStorm Bridge - Scaling, Cluster & csfast]].
