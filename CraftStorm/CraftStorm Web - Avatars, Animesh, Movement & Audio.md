---
tags:
  - craftstorm
  - avatars
  - animesh
  - animation
  - voice
  - webrtc
  - audio
  - physics
aliases:
  - CraftStorm Avatars
  - CraftStorm Animesh
  - CraftStorm Audio & Voice
---

# CraftStorm Web - Avatars, Animesh, Movement & Audio

← Back to [[CraftStorm]] | Related: [[CraftStorm Web - Architecture, Rendering & Batching]], [[CraftStorm Web - Performance, Budgets & Diagnostics]], [[Voice (WebRTC, Janus & SynthVoice)]], [[Avatar, NPC & Region Modules]]

How the browser viewer in `C:\Users\pakkio\w\craftstorm\web\` builds avatar skeletons, rigged-mesh skinning, Bakes-on-Mesh (BoM), Animesh, priority-blended animation, client-side movement prediction, WebRTC voice and 3D audio. Everything below was checked against source; paths are relative to `web/` unless noted and line numbers refer to `app.js` (12,778 lines).

> **Verified against code (commit `bba1a8b8f6`, 2026-10-16)**, build `v2.4.2 (2026-10-15)`. Test coverage: `tests/anim_frame.test.mjs`, `anim_rest.test.mjs`, `anim_root.test.mjs`, `fly_speed.test.mjs`, `tests/test_anim.py`, `tests/test_animesh_flag.py`.

---

## 1. Avatar body, skeleton, floor placement

### Source data and extractor (`scripts/content_tools/extract_avatar_web.py` -> `web/avatar_data/`)
- The extractor reads the three Linden Binary Mesh files `avatar_head.llm`, `avatar_upper_body.llm`, `avatar_lower_body.llm` plus `avatar_skeleton.xml` (`extract_avatar_web.py:80-82`; header documents the .llm layout and the joint "filler ancestor" rule from `LLPolyMeshSharedData::setupJointNames`).
- Output in `avatar_data/`: `head.json` (248 KB), `upper_body.json` (508 KB), `lower_body.json` (223 KB), `skeleton.json` (19.7 KB), `joints.json` (3 joint pivots), UV wireframes, and default skin PNGs (`default_head/upper/lower[_f|_m|_u].png`).
- `skeleton.json` has **160 entries** (`order` + `joints`), including **26 collision volumes** (`collision: true`: PELVIS, BUTT, BELLY, LEFT/RIGHT_HANDLE, LOWER_BACK, CHEST, LEFT/RIGHT_PEC, UPPER_BACK, NECK, HEAD, L/R_CLAVICLE, L/R_UPPER_ARM, L/R_LOWER_ARM, L/R_HAND, L/R_UPPER_LEG, L/R_LOWER_LEG, L/R_FOOT).
- **Hair, eyes, skirt, eyelash `.llm` meshes are NOT extracted.** Hair is a procedural sphere cap on `mSkull` (radius 0.114/0.103/0.106 for F/M/U) and eyes are three small spheres per eye joint (`makeAvatarMesh`, `app.js:2775-2900`).
- Client loading: `loadAvatarPartGeometry(part)` (`:2258`, fetches `/avatar_data/<part>.json`), `loadSkeletonData()` (`:2292`). `makeAvatarMesh(skinColor, sex, onBodyReady)` builds head/upper/lower `SkinnedMesh`es bound to one `THREE.Skeleton`, applies small sex-specific bone scaling for F/M, sets `ud.ready = true` or `ud.bodyError`. Body group is rotated `PI/2` about Y so the mesh's local +X face direction matches the camera's -Z forward; `AVATAR_FACING_FIX` = -PI/2 about Y (`:1156`) is multiplied onto sim rotations.

### Default skins
Because a server bake may be absent, `bakeAvatarTextureCanvas(type, sex)` (`:2536`) paints **512x512 canvas** skins for `head|upper|lower` (cached per `type_sex` in `defaultAvatarTextures`), so avatars are clothed on frame 0. This (not "clientBakeHead/Upper/Lower" / `scheduleClientBake`, which do not exist) is the local-bake fallback. Head/upper/lower materials use `alphaTest: 0.5` so a server bake's alpha layer cuts away system body parts hidden under mesh bodies.

### Placement and the v2.4.2 floor fix (commit `bba1a8b8f6`)
`placeAvatarOrPrim(mesh, isAvatar, simPos, simRot, seat)` (`app.js:3032-3045`): `mesh.position.copy(simToThree(simPos))` with **no height offset**; the old `pos.y -= 0.8` for other avatars/NPCs was removed because the self avatar is drawn at the very same sim position (`playerPos = server pos`) and looked right, so others sank. The previous doc's "feet at y = -0.75 inside `createAvatarMesh()`" is unverified - there is no `createAvatarMesh` function; the factory is `makeAvatarMesh`.

### Seated avatars
`resolveSeatRelative(seat, simRot, outPos, outQuat)` (`:1138`) converts seat-relative pos/rot to world once: position by the seat's world quaternion + position, rotation `seatWorldQ * slQuat(simRot)`. For the self avatar `selfSitQuat` (`:1150`) is computed from the self ObjectUpdate (`:6331`) and `*= AVATAR_FACING_FIX`; the frame loop copies it to the mesh while `selfSitting && haveSelfSitQuat` (`:12494`). Instanced seat proxies are first un-instanced (`prepareParentForChildren`, `:3026`) so a child is not scaled by the proxy's scale. Commit `3bb5b5614c`.

### Bakes (server) and Bakes-on-Mesh
- `applyAvatarTextures(msg)` (`:6779`) caches the message per avatar in `avatarTexByFullId` (and under key `'self'`), skips default-guest bare-skin bakes (`isDefaultGuestAvatar`, `:6801`) and calls `applyBakedTextures(mesh, tex)` (`:2993`), which loads **only `tex.head`, `tex.upper`, `tex.lower`** onto the three materials (priority 0 for self, 1 otherwise). The slot numbers (8, 9, 10, 11, 19, 20) quoted in the old doc are not in the web client (unverified; the bridge sends named fields).
- BoM: `BOM_PLACEHOLDERS` (`:5351`) maps six real SL placeholder UUIDs to slots `head, upper, lower, eyes, skirt, hair` (e.g. `5a9f4a74-...` -> head). `makePrimMaterial` swaps the placeholder for `selfBakeId(slot)` and `refreshBomMaterials()` re-points built materials when the bake arrives. `hideSystemPartsForBom()` (`:5387`, re-run at most every 1500 ms from the cull pass) hides `headMat/upperMat/lowerMat` of the **self** avatar when `selfWearsPlaceholder()` finds a worn face with that placeholder, and hides the stand-in hair and eyes with the head.
- **Documented limitation (code comment `:5351`)**: materials are shared by texture id, so every avatar's BoM faces show the **self** bake; other avatars' BoM parts need a per-wearer material (not implemented). `hideSystemPartsForBom` only handles the self avatar.

---

## 2. Animation pipeline (`assets/anim_*.js`, `app.js:7385-8040`)

### Data flow
sim `AvatarAnimation` / `ObjectAnimation` lists `[{id, seq}]` -> `reconcileTracks(target, anims, nowMs, failedAt)` -> clips fetched as JSON from `GET /anim/<id>` (`loadAnimClip`, `app.js:7404`; decoded bridge-side, cached in `animClipCache`, failures remembered in `failedAnimClips` and retried after **60 s**, `anim_state.js:55`) -> per-frame `applyTargetToBones(target, bones, nowMs, positions)` (`:7485`) -> bones. Targets live in `animTargets` (avatars: `findAnimTargetForLid`, `:7469`; animesh: `target.kind === 'animesh'`).

### `anim_state.js` (140 lines, pure logic)
- `USE_MOTION_PRIORITY = -1` joint priority inherits `clip.priority`.
- `cubicStep(x) = x*x*(3-2x)` (clamped 0..1) for eases.
- `reconcileTracks`: new id -> track `{seq, t0, clip:null, stopping:null, lastW:0}`; changed `seq` restarts clock and clears ease-out; vanished tracks with a loaded clip, `easeOut > 0` and `lastW > 0` become `stopping = {stopT0, residualW}` (ease-out contributors); others are deleted.
- `trackStatus(track, nowMs)`: phases `active` / `easeout` / `done`; ease-in weight `cubicStep(elapsed/easeIn)`; ease-out `residualW * cubicStep(1-u)`; non-looping clips start their own ease-out at `duration - easeOut`.
- `jointContributors(target, joint, nowMs)`: sorts by joint priority (desc), ties by `seq` (asc, activation order).
- `targetJoints(target)`: union of joint names over loaded clips.

### `anim_clip.js` (179 lines)
`slQuatToThree` = `(x, z, -y, w)` normalised; `slPosToThree` = `(x, z, -y)`; `clipTime` wraps looping clips into `[loopIn, loopOut]`; `sampleRotTrack`/`samplePosTrack` binary-search keyframes `[t, x, y, z, w]` and slerp/lerp; `sampleJoint`, `sampleClip`.
- **`blendRotations(rots, current, out)` / `blendPositions`**: start from the live pose with leftover weight `accW = 1 - maxW`; for each contributor (priority order) either take it outright when `accW <= 0` or slerp/lerp with factor `accW / min(1, accW + w)`. At full weights the highest priority wins outright; during eases lower priorities show and a lone contributor with no base blends from the live pose rather than snapping (comment: deliberate difference from the viewer).
- Avatars: rotations from all joints but **positions only for `mPelvis`** (`positions === 'pelvis'`); animesh: `'all'`.

### Procedural fallback pose
`poseAvatar(bones, phase, 'walk'|'idle')` (`:2935`) drives swing/idle when no clip data is playable; other avatars' walk phase advances `dt*8*every` using `moved > 0.005`.

### Rigged mesh and Animesh
- Flag source: the simulator's `ExtendedMeshParams` (`PARAMS_EXTENDED_MESH` type `0x70`) bit 0 (`ANIMATED_MESH_ENABLED_FLAG`) parsed in `bridge.py:3778-3783` -> `animated` on the object message (`app.js:6677`). The previous doc's `OBJECT_ANIMATED_MESH = 0x80000` constant does not appear in the web code (unverified as a protocol value; bit 0 of the extended-mesh param is what the bridge reads).
- `buildAnimRig(meshData, skelData)` (`app.js:7676`) builds a `THREE.Bone` tree rooted at `_animRoot` from `skin.jointNames` + `inverseBindMatrix`; `ensureAnimRig` (`:7908`) marks `_animRigPending`; `stepAnimeshTargets` (`:7983`) re-resolves rig-capable parts **every 3000 ms** (2000 ms while none are found) via `resolveAnimeshParts` (`:7956`), so parts that stream in after the first (commit `81220f2cc8`) get rigged; it deliberately tolerates a dropped script root (children keep animating). 
- Mounting: rig root hangs under the linkset **root** with identity local transform (`animeshRigMount`, `:7537`; `homeAnimRig`, `:7642`), because native SL ignores a rigged part's own link offset - parenting under the part scattered multi-part animesh.
- **Asymmetric skinning pair** (`assets/anim_frame.js`, 52 lines): three skins as `boneWorld * boneInverse * bindShape * v`; `slInverseBindToThree` (`app.js:4668`) = `C*M`, `slBindShapeToThree` (`:4664`) = `M*C^-1`, so the frames cancel. Only the **rest pose** has no partner, so `buildAnimRig` uses `slAffineToThree(THREE, arr16)` = `C*M*C^-1` (C is the permutation `(x,y,z)->(x,z,-y)`) and then inverts it. The file header records the v2.0.12 regression where the cure was over-applied to `boneInverse`.
- **Per-joint scale** (`assets/anim_rest.js`, 49 lines): `restFromBind(THREE, bindWorld, parentOf)` decomposes `inverse(parentBindWorld) * bindWorld` into position, quaternion **and scale** (real rigs: 16 of 26 joints on bird rigs, 8 of 50 on larger critters carry scale). Invariant `bone.matrixWorld * boneInverse = identity`; tests assert error `< 1e-6` (`tests/anim_rest.test.mjs:46,63,64`), not 1e-5 as previously written.
- **Root bind rotation** (`assets/anim_root.js`, 94 lines): `bindShapeRootQuaternion(arr16)` ports `LLControlAvatar::matchVolumeTransform` (MAINT-8671): normalise rows of the root part's `bind_shape_matrix` 3x3, invert, quaternion, then `[x, z, -y, w]`; returns `null` for unusable input. Identity for worn animesh (attachments use `obj_rot * joint_rot`), plain-prim roots with no skin, and until the root asset caches (`syncAnimRootRotation`, `:7631`, applies it per frame; cached in `animRootQCache`). Header explicitly says earlier "cow"/asset-name attributions were wrong.
- Rigged **avatar** attachments: `buildRiggedSkinnedMesh(geo, mats, meshData, avatarMesh)` (`:4823`) with 'attached' bind; static fallback bakes bind pose via `riggedBindBake` (`:4686`) / `mesh_worker.js`.

---

## 3. Movement, prediction and camera

| Constant | Value | Line | Note |
| :--- | :--- | :--- | :--- |
| `MOVE_SPEED` | 2.4576 m/s | `app.js:11157` | = OpenSim `AgentControlMidVel` (0.6 x 4.096) |
| `RUN_MULTIPLIER` | 5/3 (= 4.096 m/s) | `:11234` | = `AgentControlNormalVel` sent with `CTRL_FAST_AT` |
| `RUN_HOLD_MS` | 300 | `:11233` | hold a direction 300 ms, or Shift on ground (Shift = descend while flying) |
| `GRAVITY` / `JUMP_SPEED` | 20 / 6 | `:11229-11230` | local prediction |
| `EYE_HEIGHT` | 1.62 | `:11161` | name tags at `+0.55` above |
| `CAMERA_FOCUS_ABOVE_CENTER` | 0.75 | `:11163` | |
| `camDistance` | 3.34 default, clamped 2..30 | `:11204`, wheel handler | = `CameraOffsetRearView` magnitude x `CameraOffsetScale` 1.1 |
| `SERVER_SNAP_DISTANCE` | 2 m | `:1167` | below this the sim position is ignored |
| `SERVER_SNAP_TELEPORT_DISTANCE` | 10 m | `:1171` | always snap beyond this |
| frame `dt` cap | 0.25 s | `:12201` | `Math.min(clock.getDelta(), 0.25)` |
| move heartbeat | 100 ms (`setInterval`) | `:12114-12165` | `{type:'move', keys, yaw, cameraPos}` |

- **Control flags**: `currentMoveKeys()` (`:12135`): `CTRL_AT_POS/NEG`, `CTRL_FAST_AT` (running), `CTRL_UP_POS` (Space - ascend when flying, jump when grounded), `CTRL_FLY`, `CTRL_UP_NEG` (Shift/PageDown while flying); left+right mouse = forward.
- **`maybeSnapToServer(serverPos)`** (`:1190`) is the reconciliation policy: ignore drift <= 2 m; tolerate along-track lag while moving (and 1.2 s after stopping) up to `moveSpeedNow*1.2 + 1` m; a fresh collision block (`lastCollisionBlockT` < 500 ms) vetoes snaps up to 10 m unless the sim is ahead of us by >= 1 m in the pushed direction, in which case the blocking object is marked non-solid (`markNonSolid`) and a `walkThrough` diag is sent; otherwise a `serverSnap` diag is sent and the position is pulled (halfway for moderate drift - continuation not read here, unverified).
- **Live fly speed** (`assets/fly_speed.js`, 70 lines): `createFlySpeedEstimator(MOVE_SPEED)` seeds both axes with the walk speed; `observeFlySample` ignores samples with `dt` outside 60..600 ms, speeds below base or above `MAX_PLAUSIBLE_SPEED = 60`, only updates an axis whose key was held (`movingH`/`movingV`), EMA `alpha = 0.35`; `resetFlySpeed` on region change. Fed from `sampleFlySpeed(p)` (`app.js:11178`) before `serverPlayerPos` is overwritten; exposed in diag `player.flySpeed {h, v, samples}`.
- **Idle freeze and display damping** (`:12464-12495`): `idleGrounded = !isMoving && !flying && !selfSitting && verticalVel === 0 && |groundY - playerPos.y| < 1.0` skips gravity entirely. Display position follows raw while flying/airborne, else x/z exact and y damped with `1 - exp(-dt*12)`; teleports (>5 m, squared > 25) snap. Camera look anchor `smoothLook.y` damped with `1 - exp(-dt*8)` (`:12536`).
- **Camera floor**: when the boom would go under `worldFloorY + 0.4` the boom is shortened (`t` along `camOffset`, min `MIN_BOOM = 0.6` m, `:12556`) instead of lifting the camera, preserving the pitch (`:12550-12566`).
- Collision itself (`collision.js`) uses `intersectObjectsAccelerated` for wall/floor probes; see [[CraftStorm Web - Architecture, Rendering & Batching]].

---

## 4. Voice (`voice/voice_client.js`, 544 lines)
Same service as Firestorm (wolfvoice / WebRTC; see [[Voice (WebRTC, Janus & SynthVoice)]]).
- Constants (`:15-23`): `DATA_CHANNEL_LABEL = 'SLData'`, `POSITION_INTERVAL_MS = 100`, `POSITION_SCALE = 100` (centimetres), `PEER_GAIN_CONVERSION_FACTOR = 220`, `LEVEL_SCALE = 128`, `SPEAKING_LEVEL = 0.30`, `ICE_BATCH_MS = 200`, `CAP_TIMEOUT_MS = 20000`.
- Signalling goes through the bridge: `callCap(cap, bodyXml)` (`:141`) posts `{type:'voice_cap', id, cap, body}`; `onCapResponse` (`:126`) resolves `voice_cap_response`. Caps actually used: `ProvisionVoiceAccountRequest` (`:294`, `:521`) and `VoiceSignalingRequest` (`:400`). **`ParcelVoiceInfoRequest` is not referenced** in the client or `app.js` (the old doc listed it). Replies are parsed with a small LLSD-XML parser. Media and the `SLData` data channel (`createDataChannel`, ordered, created *before* the offer) go directly to wolfvoice; no ICE servers configured.
- App side (`app.js:8792-8830`): `makeVoiceClient()` wires position/yaw/camera/region handle/display names; UI button states `off|joining|on|failed`; **auto-join once per page** after login when both caps exist (`voiceAutoTried`, `:9157-9161`), re-armed on next login (`:7287`). Visualisers: floating dot + wave ring textures `assets/voice_meter_dot.png`, `voice_meter_rings.png` (`:8903-8904`), mic meter (`getMicLevel`), per-peer mute (`setPeerMuted`), `setMicMuted`.

## 5. World audio (`assets/world_audio.js`, 376 lines)
- Levels: `DEFAULT_LEVELS = {master 1.0, sfx 0.5, ambient 0.5, music 0.3, voice 0.7}` (`:21`), each with mute (`eff(k)`); `setLevel`, `setMuted`. Voice gain node is shared via `deps.voiceGain()`.
- `MAX_SOURCES = 48` concurrent object sounds (`:23`), `TRIGGER_MAX_DIST = 100` m for one-shots (`:24`). Flags `SOUND_FLAG_LOOP 0x01`, `QUEUE 0x10`, `STOP 0x20`.
- `makePanner()` (`:89`): `PannerNode` `panningModel 'equalpower'`, `distanceModel 'inverse'`, `refDistance 1`, `rolloffFactor 1`. `attached`, `objectSound`, `trigger`, `preload`, `gainChange`, `removeObject`. `llSetSoundRadius` is a hard cut-off in `update()` (distance > `st.radius`).
- Sound bytes: `/sound/<uuid>` (Ogg Vorbis). Parcel music: `<audio>` with `/music?url=<encoded>[&agent_id=]` through the bridge (mixed-content workaround), `preload='none'`, retried on user gesture by `resume()`.
- Wind (`ensureWind` `:256`, `updateWind` `:~278`): 4 s of white noise looped through a bandpass (`Q 0.8`); gust = `0.5 + 0.3 sin(0.21 t) + 0.2 sin(0.057 t + 1.3)`; altitude factor `clamp(height/40, 0, 1)`, `speed = gust*(0.35 + 0.65*lift)`; frequency `180 + speed*520` Hz (180..700).
- `uiSound(kind)` (`:~320`): `'send'` 880 Hz blip; `'tp'` bandpass-swept noise (`Q 2.5`, 250 -> 2600 Hz exponential over 0.55 s); `'fail'` two blips 196 Hz then 147 Hz at +0.16 s. (Waveform type/duration details beyond these were not re-read: unverified.)
- Hooked in the frame loop: `worldAudio.update(frameStamp, camera.position, fwd, up, heightAboveGround)` (`app.js:12643`).

---

## 6. Known gaps
- BoM for other avatars (shared material per texture id) - documented in code, not implemented.
- Server bake slots other than head/upper/lower are not applied to the system body.
- No hair/eye/skirt/eyelash system meshes - only stand-ins.
- Whether `ParcelVoiceInfoRequest` is handled elsewhere (e.g. bridge-side) is unverified.

---

## Corrections vs previous doc
- Extractor/data: only **3** `.llm` meshes (head, upper, lower) are extracted - not hair/eye/skirt/eyelash; skeleton has **160** joints (26 collision volumes), not "130+".
- No `scheduleClientBake` / `clientBakeHead|Upper|Lower`: the fallback is `bakeAvatarTextureCanvas` (512x512) default skins; slot ids 8/9/10/11/19/20 not found in the client; server bake application covers head/upper/lower only; BoM is self-only.
- Floor fix: correct (`placeAvatarOrPrim`, no `-0.8`), but the "createAvatarMesh at y = -0.75" statement could not be verified; `app.js:37` is just the build string.
- Seat fix lines (`12448-12450, 12494`) were off; real places are `:1138` (`resolveSeatRelative`), `:6331`, `:12494`.
- Animation docs were largely right (`USE_MOTION_PRIORITY`, `cubicStep`, takeover chain). Added: 60 s clip-retry, pelvis-only positions for avatars, rig part re-resolve cadence (3 s), root-mount rule.
- `OBJECT_ANIMATED_MESH = 0x80000` is not in the web code; the animesh flag is bit 0 of the extended-mesh param (`bridge.py:3778`).
- `anim_rest` identity tolerance is `1e-6` in tests (not `1e-5`).
- Movement: constants confirmed; the dt cap is **0.25 s** (older comments in code still mention 0.05/0.1); added snap distances (2 m / 10 m), the full `maybeSnapToServer` policy and fly-speed estimator limits (60..600 ms, EMA 0.35, 60 m/s cap).
- Voice: `ParcelVoiceInfoRequest` not used; auto-join is once per page; other constants confirmed.
- Audio: all constants confirmed; wind uses a 4 s buffer and bandpass `180 + speed*520` Hz.
