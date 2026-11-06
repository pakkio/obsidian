---
tags:
  - craftstorm
  - webgl
  - threejs
  - media
  - dialogs
  - diagnostics
aliases:
  - CraftStorm media and dialogs
---

# CraftStorm Web - Interactions, Media & Diagnostics

← Back to [[CraftStorm]] | Related: [[CraftStorm Web - Controls, UI, Build Tools & Shortcuts]], [[CraftStorm Web - Avatars, Animesh, Movement & Audio]], [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]]

> Written 2026-11-06 from `craftstorm` `main` at `4399c11c74`, after the work of 2026-11-04 and 2026-11-05. Source of truth in the repo: `web/HANDOVER_TODO.md` (section "Viewer interactions, media and avatar smoothing"). Line numbers are deliberately not given: `app.js` has grown and shifted since the older notes were written.

## 1. Media on prims (MOAP), pictures and panels

Code: `web/ui/media.js`, the mouse-up handler in `web/app.js`, tests in `web/tests/media_picture.test.mjs`.

- **Click opens a lightbox.** Clicking a media surface shows it full screen (`showFull(src, pageUrl, onClick)`): the source is a canvas or a URL. **Esc closes it**, a click on the margin closes it, and clicks in the first 400 ms are ignored (`armedAt`) so the opening click cannot close it. Pointer lock is released while it is open. A click on the image either calls `onClick` (it closes the lightbox, then touches the object) or opens `pageUrl` in a new tab ("Click the page to open it").
- **Pictures** are drawn with `PICTURE_BRIGHTNESS = 0.45`, a cap that stops a screen, which is unlit, from looking brighter than daylight on white. The picture quads have a no-op `mesh.raycast`, so `pictureHit()` calls `THREE.Mesh.prototype.raycast` directly. The ray is built from the **click position**, not from the last hover position (an earlier version used the stale hover ray and missed by metres).
- **Panels with a texture** (a flat object whose face carries a map) open in the same lightbox through `viewTexture(texture, onClick)`. The image is drawn through a pattern with a `DOMMatrix` taken from `texture.matrix`, so flip, repeat, offset and rotation match what the prim shows. Textures are uploaded as `ImageBitmap`s with `imageOrientation: 'flipY'` and `flipY = false`, so the view must undo that orientation.
- **Opening the panel view is not a touch**: only a click on the image inside the lightbox touches the object, which fixed the double `llLoadURL` cards.
- **Avatar draw order and lighting** is in section 4.

## 2. Script dialogs and `llLoadURL`

Code: `web/ui/script_dialogs.js` (`createScriptDialogs({send, notify, doc, open})`), handlers in `web/bridge.py`, wiring in `app.js` (`script_dialog`, `script_question`, `load_url`, `alert`), tests in `web/tests/script_dialogs.test.mjs`.

| Script call | LLUDP message (from `message_template.msg`) | Bridge handler | Viewer UI |
|---|---|---|---|
| `llDialog`, `llTextBox` | `ScriptDialog` (Low 190, zero-coded) -> reply `ScriptDialogReply` (Low 191) | `_handle_script_dialog`, sender `send_script_dialog_reply` | buttons, or a text box for `llTextBox` |
| `llRequestPermissions` | `ScriptQuestion` (Low 188) -> answer `ScriptAnswerYes` (Low 132) | `_handle_script_question`, `send_script_answer` | Allow / Deny with the permission names (`PERMISSION_NAMES`, `describePermissions`) |
| `llLoadURL` | `LoadURL` (Low 194: ObjectName, ObjectID, OwnerID, OwnerIsGroup, Message, URL) | `_handle_load_url` | a card; a repeated offer of the same URL replaces the old card; only `http` and `https` open |
| alert messages | `AlertMessage` (Low 134) | `_handle_alert_message` | notice |

- `llLoadURL` was confirmed live (screenshot); `llDialog`, `llTextBox`, permissions and alerts are live but **not yet tested against a simulator**.
- A missing texture is answered by the bridge with **HTTP 204** (`serve_texture`); the viewer treats 204 as a quiet 404 so the console is not flooded.
- **Not handled yet**: instant messages from friends and groups, and sending or receiving inventory items. `_handle_im` needs the session from `AgentData.SessionID`; group chat needs `ChatterBoxInvitation` from the event queue; the offer dialogs (inventory, lure, friendship) need accept and decline replies (`TeleportLureRequest` Low 71, `AcceptFriendship` Low 297, `DeclineFriendship` Low 298). The dialog codes are in `indra/llmessage/llinstantmessage.h`.

## 3. Walking and running

- Holding a direction **walks**; running needs `Shift` on the ground or a double-tap of `W` (2026-11-05). See [[CraftStorm Web - Controls, UI, Build Tools & Shortcuts]].
- The frame clock, the shared avatar and camera height and the loop-seam fix are described in [[CraftStorm Web - Avatars, Animesh, Movement & Audio]]. Measured on 2026-11-05: avatar-versus-camera error 0 mm after the shared height; the foot jitter dropped from about 0.58 to 0.38 of frames; the left toe error from 24 to 10 mm rms (max 158 to 50 mm) after `loopSeamU`. The stock walk clip has no key at t=0, so a looping clip jumped from its last key to its first at the wrap; the fix interpolates across that gap (`assets/anim_clip.js`, test `anim_loop_seam.test.mjs`).

## 4. Avatar lighting and draw order

- `AVATAR_FILL = 2.2` adds an emissive fill to avatar materials (`liftAvatarMaterial`); `liftAvatarTrees()` re-applies it every 30 frames because worn items arrive late. The same sweep gives every avatar mesh a fixed `renderOrder` in the range around -80, so the transparent worn layers stop swapping draw order (three.js sorts by distance to each object's origin, and all worn layers share almost the same origin). A value of -1,000,000 was tried and rejected: it let the sun sprite (order -90) paint over see-through parts.
- 2.2 may be too bright; not tuned since.
- The shadow `normalBias = 0.05` did **not** help the darkness at the tester's position (the shadow frustum is +-300 m around the origin).

## 5. Console tools (`?debug` URL parameter)

Without `?debug` the viewer keeps the console quiet (`dbg()` and the voice log are gated by it). In the console:

- `await craftJitter(5)` waits until you move, then measures for 5 s. It reports the shakiest joints and altitude fraction, bone scale changes, body moves, track counts, geometry swaps and local-transform moves of worn prims, a frame-time histogram, avatar-versus-camera error and visibility flips.
- `craftBodies()` lists every skinned surface on your avatar (system body versus worn, vertex counts, posed bounds, alpha share, transparency) plus a summary of what you wear.
- `craftBodyToggle()` hides or shows the baked system body, to test whether it is a second body.

## 6. Open problems

- **Straps on the head and arms flash while walking**, and some see "the body painted twice in different positions". Measured so far: no visibility flips, texture changes, scale changes or animation fights; `craftBodies()` shows only the system body (3 skinned surfaces, 403 worn prims, 0 mesh, 0 rigged). Untested explanations: a 144 Hz display showing held frames (median frame 20.8 ms = three refreshes); draw distance (try about 30 m) and the lowest detail setting. Waiting for the tester's `await craftJitter(5)` result and frame rate.
- Password-manager prompts still react to the start-position number fields.

## 7. Operations

- Client files are served `no-store`: **Ctrl+F5** is enough for viewer changes.
- A **bridge change needs a restart, which signs everyone out**. In tmux session `bridge`, end every process holding the bridge's output pipe (the main process, its multiprocessing workers and the resource tracker), then start `web/.venv/bin/python web/bridge.py` again with its output piped to `web/bridge.log`. Killing only the main process leaves orphans that keep the pipe open, and the site answers 503. `MCP endpoint disabled` in the log is normal.
- Tests: `npm test` in `web/` (163 tests in 17 files, about a minute because card timers keep handles open) and `python -m pytest tests` (199 collected on 2026-11-06).
