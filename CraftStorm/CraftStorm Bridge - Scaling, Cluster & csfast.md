---
tags:
  - craftstorm
  - bridge
  - performance
  - scaling
aliases:
  - CraftStorm Cluster
  - csfast
---

# CraftStorm Bridge - Scaling, Cluster & csfast

← Back to [[CraftStorm]] | Parent: [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]] | See also [[CraftStorm Web - Performance, Budgets & Diagnostics]]

> **Verified against code at HEAD `7cbfa9609b` (2026-10-18)**, commits `3fcbdf4ad9`..`7cbfa9609b`.

One `bridge.py` is one Python process, so every user's UDP circuit, WebSocket and decode work shared one core. Ten commits on 2026-10-17/18 attack that.

### 1 `web/cluster.py` - sharded bridge behind a sticky front
- Starts `CRAFTSTORM_SHARDS` copies of the bridge on loopback (`CRAFTSTORM_SHARD_BASE_PORT`, default `18100`; `BRIDGE_HOST=127.0.0.1`) and a byte-level front on `BRIDGE_PORT`. **Default shards: `min(8, cpu_count-1)`** (source comment: 8 measured best for 200 users on a 32-core host, 12 was slower).
- **Stickiness**: first page load is redirected once with a `cs_shard` cookie naming the least-loaded shard (`pick_shard`, by live WebSocket count); the WebSocket and `/texture /mesh /anim /sound` fetches (which need the session's caps) carry the cookie. Cookie-less clients (curl, MCP) go to the least-loaded shard.
- **Front asset cache** (`ResponseCache`): assets are addressed by UUID, so the front caches full HTTP responses (key ignores `?agent_id=`, meshes split by gzip) with request **coalescing** (`inflight`). `CRAFTSTORM_FRONT_CACHE_MB` (1024, 0 disables); responses over a quarter of the cache or 64 MiB are not stored.
- A supervisor restarts dead shards; on Linux shards `prctl(PR_SET_PDEATHSIG)` so a killed front leaves no orphans. HTTPS: the front terminates TLS (`BRIDGE_TLS_CERT/KEY`) and speaks plain HTTP to shards.
- Bigger deployments: `nginx.cluster.conf.sample` (`hash $cookie_cs_shard`). `CRAFTSTORM_COMPILED=1` makes shards run `bridge_compiled.py` (the Cython build, see Measurements).

### 2 Bridge changes that make shards work together
- **Decode process pool**: J2K/mesh decode runs in a `ProcessPoolExecutor` (spawn context, `CRAFTSTORM_DECODE_WORKERS`, default `min(4, cpu-1)`, `0` = old in-thread path); the fetch stays on the thread that owns the session's caps. A broken pool is rebuilt.
- **Shared cache + cross-process fetch lock**: all shards share `CRAFTSTORM_CACHE_DIR`; an `flock` on one of `LOCK_SLOTS` lock files in `<cache>/locks/` (slot = hash of the asset key) stops two shards fetching the same asset. The lock dies with its process; without `fcntl` (Windows) it silently does not coordinate. Fixed cache temp-file collisions and orphaned shards (`681db7d306`); single object-cache writer (`2cc59ce191`).
- **Prefetch / gzip / orjson**: asset prefetch (`CRAFTSTORM_PREFETCH`, concurrency 3), gzip'd mesh responses when the client accepts it, `orjson` for the per-user stream when installed.
- **Fixes**: hung connections after logout (the persistent sim-socket reader is released on logout), throttled visibility rebuild, quieter packet log (`CRAFTSTORM_LOG_PACKETS=1` to see it), slower drop scan.
- **Animesh flag**: compressed object updates (`ObjectUpdateCompressed`) now carry the animesh flag too (`7cbfa9609b`, test `test_animesh_flag.py`).

### 3 `web/csfast.c` - C hot paths
- Optional CPython extension (`make build-fast` / `just build-fast`, `clean-fast`; sources `csfast.c`, `setup_fast.py`; the `.so` is git-ignored, so **build it on every deploy host**). `bridge.py` does `import csfast` and falls back to Python twins when absent, or when the C code returns `NotImplemented` on odd input.
- Functions: `bytes_to_uuid`, `terse_parse`, `pack_transform`, `te_extract` (texture ids, face colours, face texture transforms in one pass), `volume_params`, and the visibility scans `abs_pos`, `rebuild_cells`, `drop_scan`, `release_scan`. Rounding is exact.
- `tests/test_csfast.py` fuzzes C vs Python (damaged packets, NaN, rounding ties). Distance scans use `sqrt` of the squared sum, so the last bit can differ from `math.dist`.
- The recv loop drains queued datagrams with non-blocking `recv` after each `sock_recv`; terse updates mutate the object dict in place.

### 4 Measurements and verdicts (from `HANDOVER_TODO.md`, 200 simulated users)
| Step | Result |
| :--- | :--- |
| 8 shards, then csfast | bridge CPU 5.1-5.25 -> 4.2-4.3 cores |
| more C + cheaper recv (10 shards, cold cache) | 5.27-5.39 -> 4.71-4.84 cores (~-10%), median RTT 2.0-2.2 -> 1.5-1.7 ms, p95 ~155 ms unchanged |
| Cython-compile all of `bridge.py` (`make build-bridge-c`) | **no gain**: time is in asyncio, dict churn and socket calls. Rewriting `Session` in C judged not worth it |

Remaining hot spots (py-spy): per-self-update `ensure_future(to_browser)`, JSON encode + websocket writes (~5%), `_extract_compressed_extras`, `_parse_compressed_blob` dicts.

### 5 Load-test rig (`web/tests/loadtest/`)
`fake_sim.py` (simulated grid), `clients.py` (simulated browsers), `run.py` (`--mode cluster --users 200 --shards 10`), `sweep.py` (settings sweep). Profile with `py-spy record -d 20 -r 100 -f raw -p <shard pid>`.
