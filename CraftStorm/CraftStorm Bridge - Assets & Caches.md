---
tags:
  - craftstorm
  - bridge
  - jpeg2000
  - cache
aliases:
  - Bridge Assets
---

# CraftStorm Bridge - Assets & Caches

← Back to [[CraftStorm]] | Parent: [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]] | See also [[CraftStorm Bridge - Scaling, Cluster & csfast]]

> **Verified at `bba1a8b8f6` (2026-10-16); `bridge.py` has grown since (now 7,036 lines), so line numbers have drifted - use function names.** Split from [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]].

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
