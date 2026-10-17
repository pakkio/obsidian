---
tags:
  - craftstorm
  - cpp
  - firestorm
  - indra
  - opensim
  - viewer
aliases:
  - CraftStorm C++ Viewer
  - craftstorm-bin
---

# CraftStorm Native C++ Viewer (Firestorm Fork)

← Back to [[CraftStorm]] | Related: [[CraftSim]], [[Avatar, NPC & Region Modules]], [[CraftStorm Web - Architecture, Rendering & Batching]]

> **Verified against code (commit `bba1a8b8f6`, 2026-10-16).** Everything below was re-read in `C:\Users\pakkio\w\craftstorm\indra\` and in `git log`/`git show` of the fork-specific commits. Nothing was compiled or run for this pass; behaviour claims come from reading source. Marked "(unverified)" where only inferred.

`indra/` is the Firestorm viewer tree (itself derived from the Second Life viewer; merge commits from `upstream/master` and `upstream/release/2025.07` are in history, e.g. `46cfda8762`). The fork's own changes are small, tagged in code with `<FS> Craftstorm`, `<CS:Craftstorm>` or `CraftStorm:` comments, and live in the commits below.

## 0. What is fork-specific (from git history)

Commits by Claudio Pacchiega touching `indra/`, newest first:

| Commit | Date | Touches (evidence from `git show --stat`) |
| :--- | :--- | :--- |
| `7565a209ea` | 2026-10-14 | `llselectmgr.cpp/.h`, `pipeline.cpp` - thin blue contour for clickable objects |
| `9ddb8f58b4` | 2026-10-13 | `skins/craftstorm/` colors.xml, container/widget PNGs, textures.xml |
| `9885a1f27e` | 2026-10-13 | `llavatarnamecache.cpp/.h`, `fsradar.cpp`, `lltoolpie.cpp`, `llviewerwindow.cpp`, `pipeline.cpp`, `llprogressview.cpp`, `settings.xml` |
| `8a3a8ddbb6` | 2026-08-01 | branding, avatar bake fixes, script editor (`fspanellogin.cpp`, `llpreviewscript.cpp`, `llscripteditor.*`, `llinventorybridge.cpp`, `llvoavatar*.cpp`, `lltoolpie.cpp`, full `skins/craftstorm/` tree + `CraftstormLocalSplash` setting) |
| `3e53e53c13` | 2026-08-01 | inherited CI workflows moved to `.github/workflows-disabled/`; quieter greyness diagnostic |
| `a3f27bcf12` | 2026-08-01 | `lltexturefetch.cpp`, `llimagegl.cpp`, `llvoavatar.cpp/.h` - grey-texture fixes |
| `5be9d28d1a` | 2026-07-25 | rebrand (`Firestorm` -> `Craftstorm`: `ROOT_PROJECT_NAME`, `APP_NAME`, `VIEWER_BINARY_NAME`, manifests), `fsgridhandler.cpp` main-coroutine dispatch, `llperfstats.cpp` `FSSmartPerformanceMode`, `llfloaterinspect.cpp`, outfits save button |
| `5c7d59b690` / `bb9ddf9989` | 2026-07-18 | inventory "Copy UUID", allow linking scripts; clickable UUID links for `#include` in the LSL editor (`llpreviewscript.cpp`, `llscripteditor.cpp`, `fslslpreproc.cpp`) |

Zen Mode (section 6, tag `<CS:Craftstorm>`) first appears in `e522fa3e01` (2026-08-30, titled "feat(web): add 3D LLSD mesh slice decoder, avatar auto-clothing, and 512m draw distance" - a mixed commit whose message does not mention it; found via `git log -S applyZenMode`).

---

## 1. Build system

- **Binary name**: `indra/newview/CMakeLists.txt:948` -> `set(VIEWER_BINARY_NAME "craftstorm-bin" CACHE STRING ...)`; `ROOT_PROJECT_NAME "Craftstorm"` in `indra/CMakeLists.txt`; `APP_NAME = "Craftstorm"` in `indra/llcommon/indra_constants.h`.
- **Stale local tree**: the existing `build-vc170-64/CMakeCache.txt` still has `VIEWER_BINARY_NAME=firestorm-bin`, `ROOT_PROJECT_NAME=Firestorm`, `VIEWER_CHANNEL=Firestorm-MyViewer`, `OPENSIM=ON`, `HAVOK_TPV=OFF`, generator `Visual Studio 17 2022`, and contains `Release/firestorm-bin.exe` + `FirestormOS-MyViewer.exe`. Cached values win over the new CMake defaults, so that tree predates the rebrand; reconfigure to get `craftstorm-bin` (unverified that nothing else needs changing).
- **How to configure/build**: `scripts/configure_firestorm.sh` (Firestorm's stock script). Flags seen in `showUsage`: `--clean --config --build --package --velopack --no-package --fmodstudio --opensim --no-opensim --singlegrid <login_uri> --havok --avx --avx2 --tracy --lto --crashreporting --jobs --platform --kdu --ninja --vscode --compiler-cache --chan --btype`. **`WANTS_OPENSIM` defaults to TRUE** in this script, i.e. `-DOPENSIM:BOOL=ON` (Havok features disabled) unless `--no-opensim`. Upstream build docs: `doc/building_windows.md`, `doc/building_macos.md`, `doc/building_linux.md` (README).
- **`build.sh` (556 lines) is the stock Linden/Firestorm build-farm script**, last changed by upstream merges (`5c03f284af`, `bbc2ad8fc5`). It has no `--opensim`/`--sl` options of its own (only an internal `HAVOK=ON/OFF` toggle at `build.sh:150-157`) and says it must run inside the LL shared-buildscript environment. The earlier claim of a custom VS2022 "build.sh --opensim/--sl" script with FMOD-stub generation was **not found**.
- **`build_target.sh` (71 lines)** is upstream's "Add simple script to allow parallel build of OS/SL" (`cef7c02325`). Usage `./build_target.sh <OS|SL>` (uppercase) or `test`/`?`. It renames `./build-vc150-64` <-> `./build-vc150-64-OS` / `-SL`; it does not write a `.current_build_target` file. The repo's actual build dir is `build-vc170-64`, so this script's hard-coded `vc150` path does not match it here.
- **CI**: all inherited GitHub workflows were moved to `.github/workflows-disabled/` (`3e53e53c13`).
- **Default skin**: `SkinCurrent` default is **`craftstorm`** (`app_settings/settings.xml:15319-15328`).

---

## 2. OpenSim grey-texture fixes (commit `a3f27bcf12`)

**Symptom** (commit message): with `ImagePipelineUseHTTP=FALSE` object textures never loaded on OpenSim; geometry rendered but flat grey (709 textures stuck, 0 HTTP requests).

1. `indra/newview/lltexturefetch.cpp:1399-1403`, in `LLTextureFetchWorker::doWork()` (`LOAD_FROM_NETWORK`). When `use_http_textures()` is false the URL-acquisition block is skipped (`mUrl` empty) but `mCanUseHTTP` stayed true, and the UDP fallback is gated on `!mCanUseHTTP`, so neither path ran:
   ```cpp
   if (!use_http_textures() && mCanUseHTTP && mCanUseNET && mUrl.empty())
   {
       LL_INFOS_ONCE(LOG_TXT) << "HTTP texture fetch disabled; falling back to simulator (UDP) texture fetch." << LL_ENDL;
       mCanUseHTTP = false;
   }
   ```
   The `mCanUseNET` guard means a stall is never converted into an abort.
2. `indra/llrender/llimagegl.cpp` `LLImageGL::checkSize()`: the power-of-two restriction was **removed** (non-POT textures are legal since GL 2.0 and the viewer uses a core context). Previously already-fetched, decoded textures (26 per session observed) were silently discarded because OpenSim does not enforce POT on upload.
3. `indra/newview/llvoavatar.cpp` `updateRuthTimer()`: throttled (every 10 s while `getRezzedStatus() < 4`) `GREYNESS` log line per baked texture (`defined/hasGL/missing/discard/loaded`), including self. `3e53e53c13` quietened it.

---

## 3. Avatar name cache timeout and radar colour

- **`indra/llmessage/llavatarnamecache.cpp`**: new `LLAvatarNameCache::expireStalePendingNames()` (`:614`), called from `idle()` (`:608`). Upstream keeps unanswered ids in `mPendingQueue` for `PENDING_TIMEOUT_SECS = 5*60` (`:663`), so OpenSim NPCs (which omit unknown ids from both `agents` and `bad_ids` in `GetDisplayNames`) show `(Loading...)` for minutes. The fork expires anything pending longer than **`STALE_PENDING_SECS = 10.0`** (`:616`): fires waiting callbacks with the cached entry if one exists (extending expiry by `TEMP_CACHE_ENTRY_LIFETIME = 60.0`, `:57`), otherwise synthesizes a temporary name `"Unknown " + id[:8]` (`:654`) marked temporary. Whether CraftSim's server-side fix (see [[Avatar, NPC & Region Modules]]) is in place does not change this client behaviour.
- **`indra/newview/fsradar.cpp:606`**: `colortable.getColor("FSRadarNameColor", LLColor4::white)` - radar name text has its own colour entry so the status-icon tint stays white. `FSRadarNameColor` is defined at the top of `skins/craftstorm/colors.xml` (`0.23 0.15 0.08 1`).

---

## 4. Clickable-object highlight (commit `7565a209ea`)

Replaces painting scripted/touchable faces red (the beacon path) with a thin blue silhouette.

- Setting **`CraftStormHighlightClickable`** (`app_settings/settings.xml:18758`, Boolean, default `1`, comment "Always highlight clickable (touch-handler) objects").
- `pipeline.cpp:3630-3636`: `renderHighlightClickable(LLDrawable*)` collects non-avatar **root** objects with `flagScripted() && flagHandleTouch()` into `sClickableRoots`. In `LLPipeline::postSort` (`:3941-3949`) it runs `forAllVisibleDrawables(renderHighlightClickable)` when the setting is on and `sRenderHighlight`, then `LLSelectMgr::getInstance()->setClickableOutlines(sClickableRoots)`.
- `llselectmgr.cpp:6532` `setClickableOutlines()` rebuilds its `LLSelectNode` list (roots + non-avatar children) **only when the visible set changes**. `renderSilhouettes()` (`:7084-7102`) draws every node not already selected with `clickable_color(0.25f, 0.6f, 1.f, 1.f)` via `renderMeshSelection_f` (mesh) or `node->renderOneSilhouette` (prim).
- The web viewer has a parallel feature: cyan `0x00e5ff` hover contour (`web/app.js:10896`) - see [[CraftStorm Web - Controls, UI, Build Tools & Shortcuts]].

---

## 5. Tooltips: Land Impact, worn attachments, task inventory (`indra/newview/lltoolpie.cpp`)

1. **Local Land Impact** (`craftstormEstimateLandImpact`, `:97`; used at `:1570`). Comment: vanilla OpenSim answers `GetObjectCost` with a flat 1. It sums the root and non-avatar children: prim = `1`, mesh = `max(vol->getStreamingCost(), 1)`, total `ceilf`, minimum 1. **It is now always used for the tooltip** (`:1568` "always computed locally, never 'loading'") - not only when the server reports `<= 1` as the old note said.
2. **Worn attachments** (`:1326-1331`): the stock redirect `if (hover_object->isAttachment())` -> wearer's tooltip becomes `isAttachment() && !advancedToolTip`, so with advanced tooltips the attachment reports its own info.
3. **Framed "Craftstorm" block** (`:1471-1483`, closed at `:1658` with `CS_RULE`): a rule line then `"\n Craftstorm"`, then the usual lines plus: task inventory (`getInventoryContents`; up to **10** items, each `name - <LLAssetType::lookupHumanReadable>, <x.y KB | "not cached">`; "... and N more"; `Inventory: empty`; or `Inventory: loading...` after `requestInventory()`), and object creation date (`TooltipCreationDate`, `:1645`).

---

## 6. Zen Mode (Ctrl-Z), 8 px frame, skin

- **Zen Mode** (tag `<CS:Craftstorm>`):
  - Setting `CraftstormZenMode` in `app_settings/settings_firestorm.xml:5-16`, Boolean, Persist, default **1**.
  - `LLViewerWindow::applyZenMode()` (`llviewerwindow.cpp:7494`): no-op if the whole UI is hidden (Alt-Shift-U); otherwise `show = !getZenMode()` and applies it to `gToolBarView->setToolBarsVisible`, `FSInternalShowNavbarNavigationPanel`, `FSInternalShowNavbarFavoritesPanel` (without touching the user's own preferences), `mStatusBarContainer`, `gMenuBarView->setVisible(show)` (kept **enabled** so accelerators still fire) and `LLChicletBar`. Chat bar and floaters are untouched. It is re-applied after `setUIVisibility`, when leaving mouselook (`llagent.cpp:2743`) and on entering `STATE_RUNNING` (`llviewerwindow.cpp:2574-2580`, which also shows toast `CraftstormZenModeEntered`: "Craftstorm Zen mode. Press Ctrl-Z for the full viewer.", `skins/default/xui/en/notifications.xml:439`).
  - Toggle: menu **View > Zen Mode**, accelerator `control|Z` (`skins/default/xui/en/menu_viewer.xml:680-691`), listeners `View.ToggleZenMode/CheckZenMode/EnableZenMode` (`llviewermenu.cpp:6091-6120`, registered `:12928`). Ctrl-Z is shared with Undo: `CSViewEnableZenMode` returns false while `gEditMenuHandler->canUndo()`, so the toggle stands down while editing.
  - The earlier note's description (`setShowUI(false)`, menu bar moved to `(0,-1000)`) is **not** what the code does.
- **8 px blue frame** (`llviewerwindow.cpp:3186-3197`): at the end of `LLViewerWindow::draw()`, four `gl_rect_2d` fills with `LLColor4(0.f, 0.3f, 1.f, 1.f)`, `t = 8`, drawn on top of all UI. Unconditional (no setting).
- **Skin**: `indra/newview/skins/craftstorm/` (full copy: `colors.xml` plus `xui/<lang>/` and `textures/`). `craftstorm/colors.xml` is a **blue/slate** palette: `AccentColor 0.365 0.678 0.886`, `EmphasisColor 0.204 0.596 0.859`, `SchemeLightest 0.204 0.596 0.859`, `SchemeDark 0.173 0.243 0.314` and a comment saying it is "deliberately distinct from the default skin, which is purple/teal". The **teal/indigo** values the old note quoted (`AccentColor 0.30 0.88 0.82`, `EmphasisColor 0.49 0.36 0.92`, `SchemeLightest 0.55 0.95 0.90`, `SchemeDark 0.20 0.15 0.38`) are in `skins/default/colors.xml:20-36`, not in the active skin.
- **Other**: `CraftstormLocalSplash` setting (`settings.xml:7405`) shows a bundled splash page instead of the grid's login page (`fspanellogin.cpp:883-900`); `llprogressview.cpp` tidied (3rd-party logo vars initialised, Vivox logo drawing removed); `fsgridhandler.cpp` posts grid-list HTTP callbacks to the main coroutine to avoid `LLMutex::on_main_coro()` asserts; `FSSmartPerformanceMode` setting adaptively changes shadow/LOD to hold target FPS (`llperfstats.cpp`); inventory "Copy UUID" (`llinventorybridge.cpp`, `menu_gallery_inventory.xml`); LSL editor `#include` UUID links.

---

## Known gaps and caveats
- Nothing built or run in this pass; the local build tree is pre-rebrand (section 1).
- `build_target.sh` assumes `build-vc150-64`; repo has `build-vc170-64`.
- Zen Mode landed inside a mixed web commit (`e522fa3e01`), so history for it is hard to follow.
- C++ unit/integration tests under `indra/integration_tests` and `tests/` were not run.

---

## Corrections vs previous doc
- `build.sh --opensim/--sl` and a "VS2022 build script that auto-generates a stub `3p-fmodstudio`" do **not** exist; `build.sh` is the stock LL build-farm script. Real entry point: `scripts/configure_firestorm.sh` (`--opensim` default on).
- `build_target.sh` is `OS|SL` (upper-case), uses `build-vc150-64`, 71 lines, and records no `.current_build_target`.
- Texture-fetch fix line numbers are `1399-1403` (not `1390-1404`) and the commit also removed the POT texture restriction and added a greyness diagnostic.
- Land Impact: the local estimate is always used (not only when the server value is `<= 1`).
- Zen Mode: implemented by `applyZenMode()`; menu bar is hidden with `setVisible(false)` (not moved off-screen); bound via View > Zen Mode with `control|Z`, yielding to Undo.
- Palette: the active skin is `skins/craftstorm` with a **blue** palette; teal/indigo is `skins/default`. The old file path `skins/default/colors.xml:19-37` for "Commit `9ddb8f58b4`" was wrong (that commit edited `skins/craftstorm/colors.xml`).
- Added missing fork changes: rebrand, `CraftstormLocalSplash`, `FSSmartPerformanceMode`, name-cache and radar details, LSL editor and inventory tweaks, disabled CI.
