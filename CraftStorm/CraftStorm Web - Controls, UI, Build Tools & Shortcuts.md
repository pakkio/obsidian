---
tags:
  - craftstorm
  - controls
  - shortcuts
  - ui
  - editor
  - inventory
aliases:
  - CraftStorm Controls
  - CraftStorm Shortcuts
---

# CraftStorm Web - Controls, UI, Build Tools & Shortcuts

← Back to [[CraftStorm]] | Related: [[CraftStorm Web - Architecture, Rendering & Batching]], [[CraftStorm Web - Performance, Budgets & Diagnostics]], [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]]

> **Verified against code (commit `bba1a8b8f6`, 2026-10-16, `APP_BUILD = 'v2.4.2 (2026-10-15)'` at `web/app.js:37`).** Key handlers were read in `web/app.js` (12,778 lines), `web/input/chat_commands.js` (54), `web/ui/windows.js` (221), `ui/inventory.js` (335), `ui/editor.js` (322), `ui/im_panel.js` (46) and `web/index.html`. Not run in a browser in this pass; behaviour is from source. "(unverified)" marks inference.

---

## 1. Keyboard and mouse (`web/app.js:10112-10189`, `keyup` `:10178`)

Letter keys use `e.code` (physical key). All of the below are ignored while a text field has focus (only `Escape` blurs it), except the two Ctrl chords which fire first.

| Input | Action | Code detail |
| :--- | :--- | :--- |
| `W` / `ArrowUp` | Forward | `setMoveKey('w')`; `keyLatchUntil` keeps a tap alive `MIN_PRESS_MS = 150` (`:12120`) |
| `S` / `ArrowDown` | Backward | same latch |
| `A` `D` / `ArrowLeft` `ArrowRight` | **Turn** (not strafe) | `TURN_RATE = PI/2` (90 deg/s, `:9624`) with a start ramp `YAW_NUDGE_RATE = 0.05` over `NUDGE_TIME = 0.25 s` |
| Left+Right mouse together | Forward | `mouseForward = left && right` (`:12138`) |
| Right mouse alone | Backward | `mouseBackward = right && !left` |
| Hold forward/back `> RUN_HOLD_MS = 300` or hold `Shift` (on the ground) | Run | Sends `CTRL_FAST_AT 0x400`; `RUN_MULTIPLIER = 5/3`, walk `MOVE_SPEED = 2.4576` m/s -> run `4.096` m/s (matches the sim's `AgentControlNormalVel`) |
| `Space` | Jump (ground) / ascend (flying) | Sends `CTRL_UP_POS 0x10`; held `Space` re-jumps on landing (`jumpRequested`) |
| `PageUp` / `Numpad9` | Fly + ascend | `setFlying(true)` if grounded, `CTRL_UP_POS` |
| `PageDown` / `Numpad3`; `Shift` while flying | Descend | `CTRL_UP_NEG 0x20` (`:12139-12165`); flying vertical speed from `flySpeed.vertical` |
| `F` | Toggle flying | `setFlying(!flying)`, `CTRL_FLY 0x2000` |
| `Y` | **Teleport to the surface under the pointer** (crosshair in mouselook) | `teleportToCursor(true, null)` (`:9785`); works on objects too; ignores key repeat; sends `{type:'teleport', x, y:-p.z, z:p.y+0.5}` through the sim (never sets position locally) |
| Double-click | Touch if the hovered object is clickable (cyan outline), else **teleport onto bare ground/plain object** (when single-click teleport is off) | `dblclick` handler `:9937`; shows notice "Shift+double-click to teleport onto it" |
| `Shift` + double-click | Always teleport | `teleportToCursor(true, e)` |
| Single click on ground | Teleport, only when `/click on` | `clickTeleport` persisted in `localStorage['craftstorm.clickTeleport']`; requires a click that did not rotate the camera (`rotDist < 0.02`) and was not the click that entered pointer lock |
| `T` or `E` | Touch what is under the pointer | `touchUnderPointer()` (`:11045`); `TOUCH_RANGE = 40` m. Right-click (unlocked) and the toolbar `Touch` button also touch |
| `G` | Sit on the selected/hovered object | `send({type:'sit', fullId})` |
| `Escape` | Stand up + release pointer lock + **reset view** | `send({type:'stand'})`, `resetToDefaults()` (`:10336`): camera behind avatar (`CAM_DISTANCE_DEFAULT = 3.34`), resolution `RESET_RESOLUTION = '1'`, draw distance `RESET_DRAW_DISTANCE = 70` (pushed to the bridge), wireframe off, downloads resumed, then a notice and chat line |
| `Alt` + drag | Orbit free camera around the pointed object (or the avatar if sky) | `pickFocusPoint()`; avatar does not move, no pointer lock |
| `Ctrl+Alt` + vertical drag | Dolly | `camDistance *= exp(movementY*0.003)` clamped `[2, 30]` m |
| Mouse wheel | Zoom (only in `rear` camera mode) | `camDistance += deltaY*0.005`, clamped `[2, 30]` |
| `Home` / `Numpad7` | Front view | `cameraMode='front'`, `FRONT_DISTANCE = 2.2` (mirrors SL `Front.xml`) |
| `End` / `Numpad1` | Rear view | `cameraMode='rear'` |
| Mouse move (locked or button held) | Look | yaw/pitch `0.0025` rad per px |
| `Enter` | Focus chat input | `#chat-input`, `maxlength=250` |
| `X` | Cycle avatar sex preset | `cycleSex()` (`:3421`); selector in login form: Androgynous (default) / Male / Female |
| `C` | Toggle wireframe | `toggleWireframe(scene)` |
| `N` | Display-name prompt | `showNamePrompt()` -> bridge `set_display_name` |
| `H` | Toggle the HUD "Controls" `<details id="hud-help">` | |
| `R` | Re-download all textures of the hovered object (whole linkset) bypassing cache | `refreshObjectTextures()` (`:10528`); notice if nothing hovered |
| `I` (no Ctrl) | Copy hovered-object report (JSON) to clipboard | `copyHintReport()` (`:10855`) |
| `Shift+A` | Inspect own avatar (skeleton, attachments, bakes) | `inspectSelfAvatar()` (`:10575`) |
| `P` / `Shift+P` | Performance probe / copy last probe | `runPerfProbe()` (`:10629`) / `copyLastPerfProbe()` |
| `F9` | Benchmark orbit | `toggleBenchmark()` (`:12070`, `BENCH_DEFAULT_SECONDS`) |
| `Ctrl+I` | Toggle Inventory window | `window.craftWindows.toggle('inventory')` |
| `Ctrl+E` | Open the object editor on the hovered object | `window.craftEditor.open(selectedLocalId)` |

Window blur clears every held key/button and the latches and re-sends movement state (`:10190`).

---

## 2. Indicators and HUD

- **Clickable outline**: electric cyan `0x00e5ff`, `CLICKABLE_SILHOUETTE_WIDTH = 0.008` (8 mm inverted hull) at `app.js:10896`. **Animesh outline**: purple `0xc9a0ff`, `uOutline = 0.025` (25 mm), `:7818-7834`. Target hint (`updateTargetHint`, `:10467`) shows name, distance, click-action text ("Click to Sit/Buy/Pay/Open..."), estimated Land Impact (`estimateLandImpact`) and prim count. The native viewer has the same idea in blue, see [[CraftStorm Native C++ Viewer (Firestorm Fork)]].
- **`[Y] teleport here` ring**: `makeTpHint` (`:9816`), mint `0x7fe0a8`, `TP_HINT_RANGE = 150` m, refreshed every 100 ms (250 ms when `shedLevel >= 1`, `:9904`). Persisted `localStorage['craftstorm.tpHint']` (`/tphint off` disables). A hit on a steep face (`normal.y < 0.6`) backs off 0.6 m along the ray and drops onto a floor (search <= 30 m).
- **Stats panel `#hud-stats`**: four rows `WHERE / WORLD / FRAME / STREAM` plus a hidden `NOTE` row (the old "5-row" description counted the note row), `#hud-clock` canvas, `#compass-tape` canvas + heading text.
- **Selectors (`web/index.html:487-523`)**: Draw distance slider `10..256 m` (initial HTML value **20**; bridge default is 100; `Esc` resets to 70); Resolution `Auto / 0.75x / 1x / Native` (**no 0.5x option**); Detail `Low 0.5 / Medium 1 / High 1.5 / Very high 2 / Ultra 3 / Max 4` (**up to 4, not 2**); Time of day `3pm / Noon / Night` (local only, "this grid has no day/night cycle").
- **Toolbar (`#toolbar`)**: Touch, Sit, `Stand`, `Clear Cache`, `Downloads` pause/resume, `Diag` (copy JSON), `Detta` (Italian speech dictation), `Leggi` (read chat aloud via TTS), `Voice`, `Mic`, `Sound` (mixer: master / sfx / ambient / music / voice, defaults `1.0 / 0.5 / 0.5 / 0.3 / 0.7`, `assets/world_audio.js:21`). On-screen camera pad `#campad` (arrows + centre) for trackpad/touch.
- **Login form** (`#login`): default URI `http://<grid-host>:9000/`, start choices Custom Position / Last Location / Home, region datalist.

---

## 3. Chat commands (`web/input/chat_commands.js`, handlers `app.js:9480-9610`)

Parsed by `parseChatCommand`; anything unmatched is sent verbatim as public chat on channel 0.

| Command | Effect |
| :--- | :--- |
| `/tp` or `/goto` (bare) | Lists `LANDMARKS` (hard-coded story NPC home spots, `app.js:9207`) |
| `/tp X Y [Z]` | Teleport inside the region (clamped `0.5..size-0.5`; Z defaults to terrain+1.5) |
| `/tp <name>` | Landmark / NPC / visible avatar by name (`resolveTpName`); otherwise "Unknown place or person" |
| `/t <region> [x y z]`, also `Region/x/y/z`, `Region[x,y,z]` | Hop to another region (`parseRegionSpec`) |
| `/r` or `/regions` [query] | Region search (`map_search` -> `map_blocks`) |
| `/im <name> <message>` | IM to a visible avatar (two-word names first); refuses NPCs and self; reuses a known IM session with `dialog 17` |
| `/click [on\|off]` | Single-click ground teleport |
| `/tphint [on\|off]` | Landing-ring hint |
| `/near [radius]` (default 6) | Copies a JSON of nearby objects |
| `/att` or `/attachments` | Attachment report (flags rigged parts not skinned to the avatar) |
| `/flex` | Report all flexi prims held |
| `/hide <localId>` | Toggle one object's visibility locally (debug) |

**Not implemented** (previously documented): `/help`, `/pos`, `/who`, `/sit`, `/stand`, `/s`, `/w`, `/me`, `/<channel> msg`. They are not parsed in `chat_commands.js` or `app.js` (grep), so they go out as plain channel-0 chat text.

---

## 4. Floating windows

- **Window manager** (`ui/windows.js`): panels registered in `WINDOWS`: `hud` (Info), `chatlog` (Chat), `implog` (IMs), `voice-roster` (Voice), `toolbar` (Tools), `campad` (Camera), `inventory`, `editor` (Edit). All start closed; a dock `#win-dock` at the top reopens them (visible once logged in). Draggable by the header strip, position + open state persisted in `localStorage['craftstorm_windows']` (the old key `nexus_win_*` does not exist); z-order via a rising `zTop`; clamped into view on resize. API `window.craftWindows.toggle(id)` / `isOpen(id)`. No resize handles found in this file (unverified for other CSS).
- **Inventory** (`ui/inventory.js`, `Ctrl+I`): lazy folder load (`inv_fetch {folderId}`), background-loads the rest (two folders at a time) while the name filter box is non-empty. Buttons: **Rez** (object, `invType 6`; `inv_rez` at `window.craftRezPoint()` ~2 m in front), **Wear** (objects and wearables `6|18`; `inv_wear`), **View** (double-click also). View opens a modal: notecards (`invType 7`) and scripts (`10`) read via `inv_asset` (or `obj_item_text` for in-prim items); script text needs the modify bit (`ownerMask & 0x4000`); textures/snapshots show an image from `/texture/<assetId>`; permissions, asset id, item id. **Notecards you may modify become editable with Save / `Ctrl+S`** (`save_notecard`). **Scripts are read-only** - there is no LSL editing in the web UI. No right-click context menu and no landmark teleport exist in this file.
- **Object editor** (`ui/editor.js`, `Ctrl+E` on hovered object, or "Edit hovered"): **numeric form, no 3D transform gizmo** (`TransformControls` appears nowhere in `web/`). Selects the object on the sim (`obj_select`/`obj_deselect`), shows owner/permissions from `ObjectProperties` ("You can modify this object" / not), and edits Name (62), Description (126), Position, Rotation (degrees, X-Y-Z), Size (this prim only, clamp 0.01-256) on Enter / blur / **Apply all** -> `obj_edit {pos, rot, scale, linked, name, desc}`. Flags (physical / temporary / phantom -> `obj_flags`), flexible and point-light parameters (`obj_extra`), and a Contents list (`obj_contents`, opens notecards/scripts through the same viewer modal).
- **IMs** (`ui/im_panel.js`): a single `#implog` list plus a 6-second `#imtoast` ("IM from X"); the log only shows while an avatar is within 50 m. It is **not tabbed** and has no group-chat tabs; session state is kept in `imSessions` in `app.js` (`dialog 17` replies).

---

## Corrections vs previous doc
- **Slash commands**: `/help /pos /who /sit /stand /s /w /me /<channel>` are not implemented; real set is `/tp /goto /t /r /regions /im /click /tphint /near /att /attachments /flex /hide`.
- **Editor**: no Three.js `TransformControls` gizmo; numeric fields only. No in-world **LSL script editor** in the web UI (scripts are read-only; only notecards save).
- **Windows persistence** key is `craftstorm_windows`, not `nexus_win_*`; no resize behaviour confirmed.
- **Inventory**: no context menu, no "teleport to landmark"; actions are Rez / Wear / View.
- **IM panel** is a log + toast shown only near avatars, not a tabbed purple group-chat panel.
- Resolution choices are `Auto / 0.75x / 1x / Native`; Detail goes `0.5..4`; stats panel has 4 visible rows + a hidden note row.
- Wheel zoom works only in `rear` camera mode; `Ctrl+Alt+drag` is the dolly gesture; `Space` also sends `CTRL_UP_POS` when grounded (jump).
- Shift+double-click behaviour and bare-ground double-click teleport clarified; `Y`, `T/E` ranges: `TP_HINT_RANGE 150`, `TOUCH_RANGE 40`.
- Line ranges refreshed (key handlers now `app.js:10112-10189`).
