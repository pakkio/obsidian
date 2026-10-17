---
tags:
  - craftsim
  - mcp
  - diagnostics
  - debugging
  - console
---

# MCP Server & Live Diagnostics

← Back to [[CraftSim]] | Related: [[ZEngine]], [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[Asset Deduplication & Database]], [[Build, Configuration & Self-Hosting]]

CraftSim embeds a **Model Context Protocol server** (`mcp-craftsim`) as an OpenSim application plugin: `OpenSim/ApplicationPlugins/McpServer/` (Kestrel/ASP.NET Core + the official C# `ModelContextProtocol 2.2.0` SDK). It exposes **76 `[McpServerTool]` tools in 13 tool classes** (counted from source; the old "70+" was loosely right) and rides on the simulator's ordinary console commands.

> [!NOTE] Verified against code (commit `9fea6a0332`, 2026-10-16)
> Tool names/line numbers, config keys, auth logic and bridge env vars were read from the source. I did not start a simulator or call the endpoint; runtime behaviour beyond what code shows is **(unverified)**.

---

## 1. Architecture

| Piece | File | Role |
|---|---|---|
| Plugin | `McpCraftsimPlugin.cs` | `IApplicationPlugin` (`[Extension(Path="/OpenSim/Startup")]`); `PostInitialise()` at line 87 reads `[Mcp]`, starts Kestrel in a background `Task` (must not block `OpenSim.Startup()`); registers console `jev choose`, `jev status` |
| Endpoint | `McpEndpointConfigurator.cs` | `EndpointPath = "/mcp-craftsim"`; maps HTTP POST, SSE and WebSocket; `IsAuthorized` auth gate |
| Transports | `HttpServerTransport.cs`, `WebSocketServerTransport.cs` | custom `ITransport`s (single-request HTTP, SSE, WS); the SDK's own HTTP/SSE mapping is not used |
| JWT | `McpJwtAuthenticator.cs` | EdDSA verification against `authorized_keys`-style file |
| Console bridge | `ConsoleCommandRunner.cs` | runs a console command and captures output via `MainConsole.OnOutput` under one global lock; async work scheduled by a command and finished after return is not captured (documented limitation) |
| Tools | `Tools/*.cs` | one class per area, discovered by `WithToolsFromAssembly` |

Routes (`McpEndpointConfigurator.cs` `MapEndpoint`): `/mcp-craftsim` (WS upgrade, or `Accept: text/event-stream` -> SSE, or POST -> JSON-RPC, GET -> JSON info `{server, status, transports:[http_post,sse,websocket], endpoints}`), `/mcp-craftsim/sse`, `/mcp-craftsim/message?sessionId=...`, `/mcp-craftsim/rpc`. Server info name `mcp-craftsim`, version = plugin `Version` (`"0.1"`).

### `[Mcp]` config (`McpCraftsimPlugin.cs:87-128`)
| Key | Default in code | Notes |
|---|---|---|
| `Enabled` | `false` | tracked `bin/OpenSim.ini:1624` sets `true` |
| `Port` | `80` | tracked ini: `80` (CLAUDE.md says the live host uses 9200) |
| `Token` | empty | static bearer; tracked ini: `"${Secrets|McpToken}"` |
| `AuthorizedKeysFile` | `"mcp_authorized_keys"` (full-path'd relative to cwd) | `ssh-ed25519` lines only |
| `JwtMaxLifetimeMinutes` | `1440` (min 1) | max `exp - iat` |
| `ListenAddress` | **empty = all interfaces** (`0.0.0.0`) | `localhost` -> `ListenLocalhost`; any other value parsed as `IPAddress`; `*`/`0.0.0.0` = any. Not set in tracked ini |

If `Token` is empty **and** no valid keys are loaded, a random 32-byte token is generated and printed in the startup line (`GenerateToken`), so the endpoint is never left open. With keys present and empty `Token`, JWT is the only way in. Startup log: `MCP server 'mcp-craftsim' listening on http://<host>:<port>/mcp-craftsim and ws://... (JWT: N key(s) in <file>; static token: disabled|configured|generated <token>)`.

---

## 2. Authentication (`McpEndpointConfigurator.IsAuthorized`, ~line 298)

- Token taken from `Authorization: Bearer <t>` **or** the `?token=` query parameter (so tokens can leak into URLs/logs).
- If it looks like a JWT (`McpJwtAuthenticator.LooksLikeJwt`) it is validated as EdDSA; failure is logged `[MCP CRAFTSIM]: JWT refused from <ip>: <reason>` and rejected. Otherwise it is compared with the static token using `CryptographicOperations.FixedTimeEquals` (length-checked first).
- **Rejection returns HTTP 403** (not 401) on every route.
- `McpJwtAuthenticator`: `Audience = "mcp-craftsim"` (line 46), `Leeway = 60 s` (49), `alg` must be `EdDSA`, signature must be 64 bytes and verify with BouncyCastle `Ed25519.Verify` over `header.payload`; optional `kid` must equal the key fingerprint **`"SHA256:" + standard Base64 (padding stripped) of SHA256(ssh blob)`** (same string `ssh-keygen -lf` prints; it is *not* Base64Url); `exp + leeway` not passed, `iat` not in the future, `nbf` honoured, `exp - iat <= JwtMaxLifetimeMinutes`, `aud` string or array containing `mcp-craftsim`. The key file is re-read when its `LastWriteTimeUtc` changes, so revoking = deleting the line. Only `ssh-ed25519` entries are used. Six tests in `tests/OpenSim.Tests/McpJwtAuthenticatorTests.cs`.
- Traffic is still plain HTTP/WS: a captured JWT or token can be replayed until expiry (CLAUDE.md says so; use the SSH tunnel below).

### Client bridge `scripts/mcp-craftsim-bridge.js` (stdio <-> endpoint)
- URL: `MCP_URL` (fallback `CRAFTSIM_MCP_URL`), default `http://127.0.0.1:80/mcp-craftsim`. `http(s)://` uses POST; `ws(s)://` opens a WebSocket with the bearer header.
- Token precedence: `MCP_SSH_KEY` (mint JWTs) -> `MCP_TOKEN`/`CRAFTSIM_MCP_TOKEN` -> `Secrets.ini` key `MCP_SECRET` (default `McpToken`) searched in `$CRAFTSIM_SECRETS`, `../bin/Secrets.ini`, `~/.craftsim/Secrets.ini`. No token -> it exits with a message instead of guessing.
- JWT minting: `node scripts/mcp-craftsim-bridge.js --jwt [-k <key>] [-m <minutes, default 70>] [--sub <name>]`; the live bridge uses `MCP_JWT_MINUTES` (default 70) and re-mints when less than 300 s remain (`bearer()`, line ~243). Key default `~/.ssh/id_ed25519`. Passphrase-protected keys are signed by `ssh-agent` (`SSH_AGENTC_SIGN_REQUEST = 13`, via `SSH_AUTH_SOCK`); no npm dependencies.
- SSH tunnel: `MCP_SSH_TARGET` (e.g. `<user>@<host>`) + `MCP_SSH_PORT` (e.g. <ssh-port>) -> spawns `ssh -N -L 127.0.0.1:<free>:<MCP_URL host>:<port>` with `BatchMode=yes`, `ExitOnForwardFailure=yes`, `StrictHostKeyChecking=accept-new`, `ServerAliveInterval=30`, `ServerAliveCountMax=3` (lines 294-297); `MCP_URL` is then the address *as seen from the server*. The bridge exits if the tunnel drops; the MCP client restarts it. `MCP_SSH_KEY` doubles as `ssh -i`.
- Repo client configs: `.mcp.json` (stdio -> bridge, `MCP_URL=http://127.0.0.1:9200/mcp-craftsim`), `mcp_config.json` (old paths under `/root/craftsim`, `http://<grid-host>:80/mcp-craftsim`), `opencode.json`. No tokens are stored in them (since 2026-10-10).

---

## 3. Tool catalog (76 tools, by source class; line = `[McpServerTool]` line)

| Class (`Tools/`) | Count | Tools |
|---|---|---|
| `ConsoleTools.cs` | 15 | `run_console_command`(53), `server_version`, `list_console_commands`, `show_uptime`, `show_server_info`, `show_threads`, `show_estates`, `login_status`, `login_enable`, `login_disable`, `force_gc`(101), `console_tail`(113), `asset_dedup_stats`(163), `inspect_asset`(193), `get_craftsim_skill`(253) |
| `AdminTools.cs` | 6 | `assets_sweep`(60, needs `confirm=true`), `assets_backfill`(72), `zengine_run`(78), `db_backup`(105), `create_object`(145), `delete_object`(183, needs `confirm=true`) |
| `DbTools.cs` | 18 | SQL: `db_info`(67), `db_tables`, `db_describe`, `db_count`, `db_query`, `db_exec`(114); OsDb: `osdb_list`(130), `osdb_info`, `osdb_rows`, `osdb_find`, `osdb_count`, `osdb_insert`, `osdb_update`, `osdb_delete`, `osdb_export`, `osdb_import`, `osdb_attach`, `osdb_detach`(376) |
| `ScriptDebugTools.cs` | 7 | `set_breakpoint`(89), `remove_breakpoint`, `is_script_paused`, `get_script_globals`, `get_script_locals`, `resume_script`, `current_script_line` |
| `AisTools.cs` | 5 | `check_ais`(51), `use_ais`, `dont_use_ais`, `trace_ais_on`, `trace_ais_off` |
| `ObjectTools.cs` | 5 | `list_objects`(48), `get_object_details`, `update_inventory_item_asset`, `set_notecard_content`, `get_notecard_content` |
| `NpcTools.cs` | 4 | `create_npc`(48), `npc_move_to`, `delete_npc`, `npc_list_with_state` |
| `RegionTools.cs` | 4 | `list_regions`(52), `region_status`, `region_broadcast`, `region_restart` |
| `UserTools.cs` | 4 | `get_display_name`(51), `set_display_name`, `list_users`, `inventory_stats` |
| `ScriptEngineTools.cs` | 3 | `script_engine_status`(49), `reset_script`, `reset_object_scripts` |
| `AvatarTools.cs` | 2 | `list_avatars`(47), `teleport_avatar` |
| `PathfindingTools.cs` | 2 | `pathfinding_rebuild`(45), `pathfinding_status` |
| `JevTools.cs` | 1 | `choose_best_tool`(167) |

Behaviours worth knowing:
- **`db_*`** shells through the console `db *` commands (`SqlConsoleModule.cs:140-179`); `db_exec` is refused unless `[DbConsole] AllowWrite = true` (default `false`), and is logged at WARN. `db_query` caps output at `[DbConsole] MaxRows` / `MaxCellChars`; blob columns print as byte length. Console parsing strips double quotes: use single-quoted SQL literals.
- **`osdb_*`** call `IOsDbModule` directly (no console splitter), lock the store and persist to the backing notecard. See [[Module - OsDb & Vector Search]].
- **`console_tail`** reads only a bounded window from the end of `OpenSim.log` (default 8 MB, clamped 64 KB-64 MB) so multi-GB logs cannot OOM; the filter only searches that window and the result says so.
- **`zengine_run`** wraps `zengine run <file> [--touch] [--say ch "msg"] --wait <0-60000> --copies <1-500>`; path must contain no spaces/quotes. Headless test of a script on a throwaway prim; see [[ZEngine]].
- **`db_backup`** runs `sh bin/backup-db.sh` (MySQL only) and returns `exit=<n>` + output; killed after 10 min.
- **`get_craftsim_skill`** returns `.agents/skills/craftsim-mcp/SKILL.md` searched under cwd, parent, and `AppDomain.BaseDirectory` (that file is not in this repo checkout: **unverified present on the host**).
- **`choose_best_tool(query, category)`**: `JevTools.Catalog` has 69 entries (of the 76 tools). Uses `DecisionClient` (`OpenSim/Framework/DecisionClient.cs:62`, `DefaultModel = "perplexity/pplx-decider-v1.2"` via OpenRouter; `subq-snap-preview` also works, see [[Module - osAI & osJev]] §6.4); overrides `OPENROUTER_API_KEY`, `OPENROUTER_DECISION_MODEL`; no key -> keyword heuristic fallback (logged). Console: `jev choose <query>`, `jev status`.

---

## 4. Console commands behind the tools (grouped; file where registered)

- **ZEngine** (`ZEngine.cs:330-384`): `zengine run <file> [--touch] [--say <ch> <msg>] [--wait <ms>] [--copies <n>]`, `zengine desugar <file> [--out <f>] | zengine desugar --heredoc [--out <f>]` (384), `zengine desugarcheck <file> [--touch] [--say ...] [--wait ms] [--verbose] | --all [dir ...]` (358; default dirs `tasks` and `scripts`, `--all` waits 3000 ms), `zengine include show <asset-uuid | prim-uuid item-name>` (330), plus the variable inspector reachable as `zengine vars [filter] [--all]`, `zengine show vars`, `show script vars` (argument-matched at `ZEngine.cs:572-586`).
- **Objects** (`ObjectCommandsModule.cs:211-259`): `reset script <id> <script>`, `reset object scripts id|name [--regex]`, `set script source id <id> <script> @<file>`, `set notecard content id ... @<file>`, `show notecard content id ...`, `set object desc id ...`.
- **Assets** (`AssetService.cs:96,108` and `assets sweep|backfill`): `assets dedup stats [<type>]`, `assets inspect <id>`, `assets sweep`, `assets backfill`. See [[Asset Deduplication & Database]].
- **Database** (`SqlConsoleModule.cs`): `db info|tables [filter]|describe <t>|count <t> [where]|query <sql>|exec <sql>`. **OsDb** (`OsDbConsole.cs:81-180`): `osdb list|info|rows|find|count|insert|update|delete|export|import|attach|detach|drop`.
- **AISv3** (`AISv3Module.cs:195`): `check_ais [<region-name>]` (underscore form, as registered). The `trace_ais_on/off`, `use_ais`, `dont_use_ais` names were not individually re-grepped as console commands **(unverified)**; the MCP tools exist (`AisTools.cs:51-90`).
- **SSB** (`UpdateAvatarAppearanceModule.cs:87-138`): `ssb bake|app|http|cof|test`.
- **Experience** (`ExperienceConsole.cs`): `experience create|list|show|delete|state|grant|revoke|associate|disassociate`.
- **Pathfinding** (`PathfindingModule.cs:517-560`): `pathfinding rebuild|test-character|enable|disable|status|rebuild-radius|clear`. **Region**: `region reset [--yes]` (`RegionCommandsModule.cs:122`).
- **Server base** (`ServerBase.cs`): `force gc` (316), `log history [<count>]` (334), `log browse [<level>] [from <dt>] [to <dt>] [latest <N> min|hour] [<count>] [<keyword>]` (341); `debug threadpool set worker|iocp min|max <n>`.
- **Voice**: `janus info|show|list|list rooms|list sessions|room` (module "Webrtc"); see [[Voice (WebRTC, Janus & SynthVoice)]].
- Quoting gotcha: the console `Parser` strips `"`; use `'...'` or `@<file>`.

---

## Corrections vs previous doc
- **`AdminTools.cs` was missing entirely**: `assets_sweep`, `assets_backfill`, `zengine_run`, `db_backup`, `create_object`, `delete_object` live there, not in `ConsoleTools`/`ScriptEngineTools`/`DbTools`/`ObjectTools` as the old table claimed. Real total: 76 tools, 13 classes.
- Old claim "`ListenAddress` default `127.0.0.1`" is wrong: code default is **empty = all interfaces**. Loopback binding is opt-in.
- Fingerprint is standard Base64 (`SHA256:...`, ssh-keygen style), not Base64Url. Old "`McpJwtAuthenticator.cs:38-285`" / "`McpCraftsimPlugin.cs:89-128`" ranges were approximate; use lines above.
- Missing in old doc: `?token=` query auth, HTTP 403 on failure, routes `/sse` `/message` `/rpc`, random-token fallback, `JwtMaxLifetimeMinutes` min 1, bridge WS mode and token precedence, `MCP_JWT_MINUTES`.
- "`Console` categories `General & Bots`: `dialog <prim-uuid>`, `touch <prim-uuid> [<bot-number>]`" are **pCampBot** (`OpenSim/Tools/pCampBot/BotManager.cs`) commands, not simulator console commands. `jev choose/status` are the plugin's own console commands (omitted before).
- `choose_best_tool` model: default `perplexity/pplx-decider-v1.2` (since 2026-10-17; `OPENROUTER_DECISION_MODEL` overrides), and the key source is env `OPENROUTER_API_KEY` (no `[Mcp]` key).
- Port/SSH values `<grid-host>`, `<ssh-port>`, `9200` come from CLAUDE.md/`.mcp.json` host config, not from the tracked `bin/OpenSim.ini` (`Port = 80`).
