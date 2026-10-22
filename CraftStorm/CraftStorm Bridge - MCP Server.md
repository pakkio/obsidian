---
tags:
  - craftstorm
  - bridge
  - mcp
aliases:
  - Bridge MCP
---

# CraftStorm Bridge - MCP Server

← Back to [[CraftStorm]] | Parent: [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]] | See also [[MCP Server & Live Diagnostics]]

> **Verified at `bba1a8b8f6` (2026-10-16); `bridge.py` has grown since (now 7,036 lines), so line numbers have drifted - use function names.** Split from [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]].

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
