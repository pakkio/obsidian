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

← Back to [[CraftStorm]] | Related: [[CraftStorm Web - Architecture, Rendering & Batching]], [[MCP Server & Live Diagnostics]], [[Voice (WebRTC, Janus & SynthVoice)]], [[CraftSim]]

> **Verified against code (commit `bba1a8b8f6`, 2026-10-16).** Every claim below was re-checked against `web/bridge.py` (6,508 lines), `web/bridge.js`, `web/desktop.py`, `web/j2k_lenient.py` and `web/tests/`. Anything not confirmed is marked "(unverified)". Line numbers are for this commit and will drift.

Browsers cannot open raw UDP sockets (LLUDP) or decode Second Life JPEG2000 textures, so **`web/bridge.py`** is a single-process `asyncio` protocol translator: XML-RPC login, one UDP circuit per browser tab, CAPS/HTTP asset fetching and transcoding, versioned disk caches, a hand-rolled HTTP/WebSocket server and an opt-in MCP endpoint. HTTP and WebSocket code is stdlib, but the process needs third-party packages (section 8).

---

## 1. Process model and entry points

| What | Where | Detail |
| :--- | :--- | :--- |
| Run | `python3 web/bridge.py` | `asyncio.run(main())` (`bridge.py:6507`). Page at `http://localhost:8088/` |
| Desktop shell | `web/desktop.py` (48 lines) | Runs `bridge.main()` in a daemon thread, waits for `bridge.PORT`, opens a `pywebview` window "CraftStorm" (`1440x900`) on `http://127.0.0.1:8088/`. With no console, stdout/stderr go to `%LOCALAPPDATA%\CraftStorm\craftstorm.log` |
| Node port | `web/bridge.js` (2,277 lines) | **Experimental** (package.json description; `HANDOVER_TODO.md #15`). HTTP+WS on `8088` only; HTTP routes `/texture/` and `/mesh/` only (`bridge.js:2151-2156`). No `/anim`, `/sound`, `/music`, no MCP, no TLS, no disk cache (0 hits for `craftstorm-cache`/`homedir`). WASM OpenJPEG (`@cornerstonejs/codec-openjpeg`) + hand-written `png.js`. Supported runtime is `bridge.py` |
| Logging | `_log()` `bridge.py:67` | stdout only, timestamped |
| Listeners | `main()` `bridge.py:6427` | HTTP `PORT = 8088` (hard-coded, `bridge.py:76`, bound `0.0.0.0`); optional HTTPS on `BRIDGE_TLS_PORT` (default `8443`) |
| Multi-user | `ACTIVE_SESSIONS`, `_active_session(agent_id)` `bridge.py:365` | One `Session` per WebSocket; asset endpoints accept `?agent_id=` to pick whose CAPS to use. `LAST_LOGGED_IN_SESSION` is what MCP acts on |

### HTTP routing (`handle_conn`, `bridge.py:6192`)
Non-WebSocket requests: `POST /mcp` -> `serve_mcp`; `/texture/<uuid>`, `/anim/<uuid>`, `/sound/<uuid>`, `/mesh/<uuid>` (UUID validated, `400` otherwise); `/music?url=`; anything else -> `serve_static` (files under `web/`). A WebSocket upgrade goes to `handle_ws_client` (`:6271`), which JSON-decodes each text frame and calls `dispatch()` (`:6302`).

### Environment variables actually read
`MCP_TOKEN`, `BRIDGE_TLS_CERT`, `BRIDGE_TLS_KEY`, `BRIDGE_TLS_PORT`, `CRAFTSTORM_CACHE_DIR`, `CRAFTSTORM_CACHE_MAX_AGE_DAYS` (7), `CRAFTSTORM_TEXTURE_SIZE` (512), `CRAFTSTORM_TEXTURE_QUALITY` (88), `CRAFTSTORM_WEBP_QUALITY` (85), `CRAFTSTORM_TEX_MEM_CACHE` (512), `CRAFTSTORM_MESH_MEM_CACHE` (256), `CRAFTSTORM_ANIM_MEM_CACHE` (128), `CRAFTSTORM_DRAW_DISTANCE` (100), and `CRAFTSTORM_OPENJP2` (in `j2k_lenient.py`).

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

## 4. Asset endpoints and transcoders

| Endpoint | Handler | Behaviour (verified) |
| :--- | :--- | :--- |
| `/texture/<uuid>?edge=<px>&format=webp\|sculpt&agent_id=` | `serve_texture` `:5520`, `_fetch_and_decode_texture` `:4284` | Order: bundled terrain defaults -> memory LRU (`TEXTURE_PNG_CACHE`, key `(uuid, edge, format)`) -> disk (**only `edge=512`, `format=auto` is persisted**) -> network. Fetch via `GetTexture` (`?texture_id=`), falling back to `ViewerAsset` on non-200 (commit `bba193f99a`). `edge` clamped `128..512`. 25 s timeout -> `504`; failure -> `404`; no CAPS yet -> `503`. `Cache-Control: max-age=86400`. 16-thread `TEXTURE_EXECUTOR` |
| J2K decode | `:4284-4400` | 1) `imagecodecs.jpeg2k_decode`, keeping the first 4 components (5th mask channel dropped). 2) On failure of a real J2K stream: `j2k_lenient.decode` (ctypes onto OpenJPEG with `opj_decoder_set_strict_mode(codec, 0)`; library search: `CRAFTSTORM_OPENJP2`, `../build-vc170-64/newview/{Release,RelWithDebInfo}/openjp2.dll`, `openjp2.dll`, `libopenjp2.so.7`, `libopenjp2.dylib`). 3) Last resort `opj_decompress -allow-partial` subprocess (`apt install libopenjp2-tools`). 4) Non-J2K bytes tried as a Pillow image |
| Encode | `_encode_for_browser` `:4434` | LANCZOS downscale of the long edge. Alpha present -> PNG, with tEXt `craftstorm-alpha=mask` when `_alpha_is_mask` (<=0.5% pixels with mid alpha, histogram bins 9..246). Opaque -> JPEG q88, or WebP q85 for `format=webp`. `format=sculpt` -> RGB PNG, alpha stripped (lossless sculpt maps). Content-Type is sniffed: `image/png; alpha=mask`, `image/png; alpha=blend`, `image/webp`, `image/jpeg` |
| `/mesh/<uuid>?lod=` | `serve_mesh` `:5325`, `_decode_mesh_asset` `:4492` | **Only `low` is distinguished; any other value = `high`.** `low` prefers `low_lod, lowest_lod, medium_lod, high_lod`; high prefers `high_lod, medium_lod, low_lod, lowest_lod`. Fetch tries `GetMesh2/GetMesh/ViewerAsset ?mesh_id=`, then `GetTexture ?texture_id=` and `?mesh_id=`. Header is LLSD; slices are `{offset,size}` zlib blocks, un-quantized from uint16 domains. `skin` -> `jointNames, bindShapeMatrix, inverseBindMatrix`, optional `pelvisOffset`, `altInverseBindMatrix` (joint overrides); weights via `_decode_mesh_weights`. Response JSON. 16-thread `MESH_EXECUTOR` |
| `/anim/<uuid>` | `serve_anim` `:5290`, `_decode_anim_asset` `:4902`, `_fetch_anim_asset` `:5014` | Tries `get_mesh_url` then `get_texture_url` with params `asset_id`, `mesh_id`, `texture_id`; the first working combo is memoized on the session. Supports `.anim` v1.0 and legacy 0.1 (float times, Euler keys, vectors clamped +-5 m). Limits: 216 joints, 8192 keys/curve, 200,000 keys total, 60 s duration. `mScreen`/`mRoot` tracks dropped; ease and hand-pose parsed then discarded. **No built-in animation table exists** - every clip comes from the grid |
| `/sound/<uuid>` | `serve_sound` `:5171` | Tries `viewer_asset_url`, `get_mesh_url`, `get_texture_url` with `sound_id` then `asset_id`; accepts only bytes beginning `OggS`; memory LRU 64 + disk `sounds/<id[:2]>/<id>.ogg`; served as `audio/ogg` |
| `/music?url=` | `serve_music` `:5206` | Relays Icecast/Shoutcast streams (up to 4 redirects, 15 s connect). **Allow-listed**: only URLs the sim sent as a parcel `music_urls` entry (else `403`) |

---

## 5. On-disk caches (`~/.craftstorm-cache`, override `CRAFTSTORM_CACHE_DIR`)

| Path | Version constant | Layout / key |
| :--- | :--- | :--- |
| `textures/v3-512q88/` | `TEXTURE_CACHE_VERSION = f"v3-{MAX_TEXTURE_EDGE}q{JPEG_QUALITY}"` | `<uuid[:2]>/<uuid>.jpg\|.png`. v2 entries migrated on first read (`_texture_cache_migrate`, PNGs re-marked cutout) |
| `meshes/v3/` | `MESH_CACHE_VERSION = "v3"` (joint overrides) | `<id[:2]>/<id>.json.gz`; low LOD stored as `<id>_low` |
| `anims/v1/` | `ANIM_CACHE_VERSION = "v1"` | `<id[:2]>/<id>.json.gz` (compact clip JSON) |
| `objects/v4/` | `OBJECT_CACHE_VERSION = "v4"` (v3 animesh flag, v4 attached sound) | `<region_key>.json.gz`, `region_key = f"{sim_ip}_{region_name}"` (set at `RegionHandshake`, `:2619`); entries `{localId: {crc, obj}}`; avatars and CRC-less objects are not stored; flushed every `60 s` when dirty |
| `sounds/` | none | `<id[:2]>/<id>.ogg` |

- Writes are atomic (`.tmp` + `os.replace`). `_cache_touch` refreshes mtime on every hit; `_cache_sweep` (startup, in an executor) deletes files unused for `CRAFTSTORM_CACHE_MAX_AGE_DAYS` (7; `<=0` disables) across textures, meshes, objects, anims and sounds and prunes empty shard dirs.
- In-memory LRUs: textures 512, meshes 256, anims 128, sounds 64.
- **`ObjectUpdateCached`** (`:3222`): announced objects whose CRC matches the disk cache are replayed; the rest are queued to `_cache_miss_loop` and requested with `RequestMultipleObjects` in batches (not one datagram each). A source comment cites ~11k ids saved on a large region reconnect (figure not re-measured).

---

## 6. Built-in MCP server (`POST /mcp`, `bridge.py:5676-6190`)

- **Opt-in**: set `MCP_TOKEN`. Without it `POST /mcp` returns `404` and startup logs "MCP endpoint disabled". There is deliberately no default credential.
- **Auth**: `Authorization: Bearer <token>` via `hmac.compare_digest`; mismatch -> `403`. Body limit `MCP_MAX_BODY = 1 MiB` (else `400`).
- **Protocol**: stateless JSON-RPC 2.0, one message per POST, no session ids, batches rejected (`400`). `protocolVersion "2024-11-05"`, `serverInfo {name:"craftstorm-bridge", version:"0.4"}`. Methods: `initialize`, `notifications/initialized` (`202`), `tools/list`, `tools/call`, `ping`, `resources/list`, `prompts/list`; others -> JSON-RPC error `-32601`. Tool failures return `result.isError = true`.
- Tools act on `LAST_LOGGED_IN_SESSION`; most return an error "not logged in..." when no live session.

**15 tools:**

| Tool | Parameters | Behaviour |
| :--- | :--- | :--- |
| `clear_cache` | none | `rmtree` of texture, mesh, object and anim cache dirs (**not** `sounds/`), clears the three in-memory caches, sends `{"type":"clear_cache"}` to the browser. Returns `{cleared:true}` |
| `bridge_status` | none | `{server, version, uptime_s, port, texture_cache, mesh_cache, anim_cache, session}` (counts = in-memory entries) |
| `agent_info` | none | `{session:{logged_in, first_name, last_name, display_name, agent_id, region_key, object_count, self_pos}}` or `session:null` |
| `scene_summary` | none | `{logged_in, region_key, object_count, avatar_count, self_pos}` |
| `list_objects` | `max` int (default 50, cap 500), `avatars_only` bool, `name_filter` str (case-insensitive; names exist only after an object was touched/selected), `max_scale` number (largest axis <= value) | `{count, objects:[{localId, fullId, name, shape, scale, pos, isAvatar, isSelf, isMesh, sculptId, parentId, spinning, texAnim, playingAnims}]}` |
| `object_details` | `localId` int (required) | `{localId, object:<full cached dict>, anims?:[{id,seq}]}` |
| `object_properties` | `fullId` uuid (required), `timeout` s (default 10, clamp 1-30) | Sends `RequestObjectPropertiesFamily`, awaits the reply; timeout -> error |
| `list_animations` | `fullId` (optional) | `{fullId, localId, isAvatar, anims:[{id,seq}]}` for one target, else `{count, targets}` |
| `anim_clip_info` | `animId` uuid (required) | Reads `ANIM_CLIP_CACHE` only: `{id, cached, duration, loop, loopIn, loopOut, priority, jointCount, joints, rotKeys, posKeys}`; error if never fetched |
| `send_chat` | `message` (required, non-blank), `channel` int (default 0) | `ChatFromViewer`; `{sent, channel}` |
| `send_im` | `targetId` (required), `message` (required), `sessionId`, `dialog` (0 or 17) | `ImprovedInstantMessage`; NPCs have no IM inbox |
| `teleport_to` | `x`,`y`,`z` (required) | Same-region `TeleportLocationRequest`; x/y clamped to `0.5..size-0.5` using login `region_size`; needs `region_handle` |
| `touch_object` | `localId` int or `fullId` uuid | `ObjectGrab` + `ObjectDeGrab` |
| `sit_on` | `fullId` uuid (required) | `AgentRequestSit` + `AgentSit` |
| `stand_up` | none | `CTRL_STAND_UP` agent update |

```bash
MCP_TOKEN=secret python3 web/bridge.py
curl -s localhost:8088/mcp -H "Authorization: Bearer secret" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```
This is the *bridge's* catalog. The simulator has its own MCP catalog (`/mcp-craftsim` on :80 per a comment at `bridge.py:437`) - see [[MCP Server & Live Diagnostics]].

---

## 7. Tests (`web/tests/`)

- Python (stdlib unittest): `cd web && python -m unittest discover -s tests -p 'test_*.py'` -> **105 tests, 1 skipped, OK** (run 2026-10-16). Files: `test_agent_loops, test_anim, test_animesh_flag, test_handover_fixes, test_im_sessions, test_mcp_tools, test_mesh_vertex_decode, test_object_omega, test_region_handshake, test_sun, test_terrain_idct, test_texture_decode`.
- JS: `cd web && npm test` (= `node --test tests/*.test.mjs`) -> **118 tests, all pass**. Files: `anim_frame, anim_rest, anim_root, app_internals, browser_flows, fly_speed, gpu_timer, handover_fixes, sculpt_build, static_batch, volume_build`.

---

## 8. Dependencies, limits and known gaps

- `bridge.py` imports `Pillow`, `imagecodecs`, `numpy`, `llsd`. `web/requirements.txt` declares only `Pillow>=11.3.0`, `imagecodecs>=2026.8.16`, `numpy>=2.0`; **`llsd` is imported by the mesh decoder but not listed** (tests pass here because it is installed). Optional: `pywebview` (desktop), `opj_decompress`.
- Node port deps (package.json): `@cornerstonejs/codec-openjpeg`, `three 0.186.0`, `three-mesh-bvh 0.9.15`.
- Strict-J2K failures need the OpenJPEG DLL/so (for `j2k_lenient`) or `opj_decompress`; otherwise the texture `404`s.
- Mesh LOD selection is binary despite four LOD blocks in the asset. Disk texture cache stores only the 512 px auto variant (graded/`webp` variants are RAM only).
- MCP has no inventory, parcel or continuous-movement tools and only targets the most recent login.
- `web/HANDOVER_TODO.md` is a 2026-09-08 review list; at least three of its items (voice_cap blocking the reader loop, flush rescheduling, stale WebSocket callbacks) appear already addressed in code (`dispatch` voice task, `_flush_browser_updates`); treat it as history, not status.

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
