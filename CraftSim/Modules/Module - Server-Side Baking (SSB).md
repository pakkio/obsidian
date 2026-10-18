---
tags:
  - craftsim
  - ssb
  - avatar
  - viewer
  - caps
aliases:
  - SSB
  - Server-Side Baking
  - UpdateAvatarAppearance
---

# Module - Server-Side Baking (SSB)

← Back to [[CraftSim]] | Related: [[Module - AISv3 Inventory API]], [[Upstream OpenSim vs CraftSim - Divergence]], [[Avatar, NPC & Region Modules]], [[Known and Unresolved Problems]]

Server-Side Baking (SSB, "Sunshine") is the Second Life protocol in which the **simulator side** composites the avatar's baked textures (head, body, eyes, hair...) instead of each viewer doing it and uploading the result. **This is fork-only code**, added 2026-07-27/28. Read from `origin/master` at `181fb0ae86` (2026-10-18); the local checkout at that time was older and did not contain the later commits.

> [!INFO] What "aligned to SL" means on this page
> Each section separates **what the Second Life viewer expects** (taken from the viewer source, quoted in the code comments and in `tasks/ssb_investigation_todo.md`) from **what CraftSim does**. No Second Life server was available to compare against, so the SL column is the *viewer's contract*, not an observed SL server trace. Where CraftSim is known to differ, it says so.

## 1. Upstream OpenSimulator and SSB

Upstream does not implement SSB. Checked against the tag `baseline-upstream-202507` (upstream, June 2025):

- no `UpdateAvatarAppearance` capability anywhere in the C# source;
- no baking code (`ServerBaking` does not exist);
- `LLClientView.cs` (`RegionHandshake`) contains this comment and then does the opposite: `// bit 0 signals server side texture baking` / `// bit 63 signals more than 6 baked textures support`, followed by `zc.AddUInt64(1UL << 63);`. Upstream advertises **only bit 63**. Bit 0 is documented in a comment and never set;
- `SimulatorFeaturesModule` already sets `BakesOnMeshEnabled`, which is the unrelated Bakes-on-Mesh feature.

So upstream's model is: **the viewer bakes and uploads**, the simulator only stores and relays the result. The upstream wiki and an open, unassigned issue in a modern OpenSim fork (Tranquillity) say the same (cited in `tasks/ssb_investigation_todo.md`). The same search on a fresh clone of upstream master (HEAD `58c2118`, 2026-10-16) finds the same thing, so upstream still does not support SSB as of that date (master branch tip only; see [[Upstream OpenSim vs CraftSim - Divergence]] for the limits of the check).

**The divergence.** CraftSim sets bit 0, serves the capability and composites on the server. The reason is practical: a viewer on a region without bit 0 logs `Region does not support baking`, recomposes from its own COF on **every login** and discards what the server held. That broke `osLoadAppearance` and NPC appearances loaded from notecards (an NPC's beard rendered grey), and it cannot be fixed from script. Seen from the other side, the cost of the divergence is that CraftSim now owns a compositor that upstream's code never has to keep correct.

## 2. Why it exists (the viewer side)

Without SSB support the Firestorm log shows, on every login:

```text
WARNING #Avatar# LLAppearanceMgr::serverAppearanceUpdateCoro : Region does not support baking
```

The viewer-source finding behind it: `serverAppearanceUpdateCoro` returns early when `getCentralBakeVersion() == 0`, and only otherwise looks up the `UpdateAvatarAppearance` capability.

## 3. The four moving parts

| Part | File | Job |
| :--- | :--- | :--- |
| Advertisement | `LLClientView.cs:~985-993` | In `RegionHandshake`, `RegionProtocols` bit **0** is set when the shim is enabled; bit **63** is always set. |
| Capability | `UpdateAvatarAppearanceModule.cs` (`ISharedRegionModule`) | Registers the per-agent `UpdateAvatarAppearance` cap and drives bake plus echo. |
| Compositor | `ServerBaking/AvatarBakeService.cs` | Decodes worn wearables and textures, composites, stores. |
| COF echo | `OpenSim.Framework/IServerSideBakingClient.cs`, `LLClientView.SendAppearanceWithCOF` (`:~4431`) | Sends an `AvatarAppearance` packet that carries the COF version. |

`SimulatorFeaturesModule.cs:142` separately sets `BakesOnMeshEnabled = true` unconditionally. **Bakes on Mesh is a different feature** from SSB; it was a red herring during the investigation, do not chase it again.

### Configuration

| Place | Value |
| :--- | :--- |
| `[ClientStack.LindenCaps] EnableServerSideBakingShim` in `bin/OpenSimDefaults.ini` | `false` |
| same key in `bin/OpenSim.ini` | `true` (this is what the running grid uses) |
| default in code (module and `LLClientView`) | `true` |

Both the module and the handshake read the same key, so turning it off removes the capability **and** the `RegionProtocols` bit together.

## 4. The request flow

1. The viewer sees bit 0 in `RegionProtocols`, so its `centralBakeVersion` is nonzero.
2. After a COF change it POSTs LLSD `{ "cof_version": <int> }` to the `UpdateAvatarAppearance` capability URL (`"/" + random UUID` per agent).
3. `HandleRequest`: non-POST gives **404**; a body that is not an LLSD map, or has no `cof_version`, gives **400**; otherwise it calls `ProcessAppearanceUpdate` and answers **200** with `{ "success": true }`.
4. `ProcessAppearanceUpdate` ignores child agents, runs `AvatarBakeService.TryBake`, and **only if the bake produced something** echoes the COF version with `SendAppearanceWithCOF(visualParams, bakes, hoverHeight, cofVersion)`.
5. The viewer treats the update as authoritative only after it sees that COF version come back on the `AvatarAppearance` packet; the HTTP `success` alone is not enough. Echoing after a failed bake would tell the viewer to trust a stale texture, which is why the echo is gated.

`SendAppearanceWithCOF` exists only on a real UDP `LLClientView`. NPCs and other synthetic clients do not implement `IServerSideBakingClient`, so for them the console reports "not sent". That is correct, not an error.

## 5. The compositor (`AvatarBakeService.TryBake`)

- **Inputs**: `sp.Appearance.Wearables` (the wearable assets the server already knows) and the textures those reference. It does **not** read the COF from inventory.
- **Wearables**: each worn wearable asset is parsed as `AssetBodypart` (Shape, Skin, Hair, Eyes) or `AssetClothing` and folded into the per-slot texture table (`AppearanceManager.DecodeWearableParams`). A bad or missing wearable is reported and skipped; it does not abort the other slots.
- **Textures**: J2K-decoded with `OpenJPEG.DecodeToImage` directly (the `System.Drawing` overload is unusable on this Linux build).
- **Bake types**: Head, UpperBody, LowerBody, Eyes, Hair, and Skirt **only if a skirt is worn**.
- **Skin guard**: Head, UpperBody and LowerBody take their base tone from the Skin tint. If the Skin did not decode, those three bakes are **skipped** (they would come out near black); the existing texture stays.
- **Output**: one `ServerBaker` per bake type; each result is a new random asset UUID, cached, then **stored non-temporary** in the asset service so the UUID still resolves after a region restart (a cache-only asset made relogs render as a cloud). The previous bake of that type is deleted, so rebakes do not leak rows.
- **Apply**: the new UUIDs go into a *copy* of the `TextureEntry`, then `IAvatarFactoryModule.SetAppearance(...)`. Working on a copy matters: mutating the live object made the diff report "unchanged", so the bake was broadcast but never persisted.
- **Concurrency**: one bake in flight per agent; a request that arrives meanwhile sets a re-bake flag and the bake runs again afterwards against the settled state. `AvatarFactoryModule` also throttles rebake requests.
- Returns `false` (appearance untouched) if nothing was produced, so the viewer can fall back to baking itself.

History worth knowing: the first version (`b6c52535ab`) only advertised SSB and served existing bakes; a compositor was removed (`29e96a00e2`) because it tried to decode wearable *asset IDs* as J2K images (they are text definitions, not pixels) and its stale-bake fallback made the viewer render a cloud; the real compositor came back in `9b1d9a6a69` and was hardened in `623c5aaceb`.

## 6. Console commands (no viewer needed)

Registered once, however many regions load. Each takes `<avatar-uuid>` or `<first> <last>`; the target is searched in every loaded region.

| Command | Exercises |
| :--- | :--- |
| `ssb bake <target>` | the compositor only, one line per decode/bake step |
| `ssb app <target> [cof_version]` | the whole `ProcessAppearanceUpdate` path including the COF-echo gate |
| `ssb http <target> [cof_version]` | registers a throwaway real handler on the HTTP listener and POSTs real LLSD to it (method check, parsing, response serialisation, which `ssb app` skips) |
| `ssb cof <target> [cof_version]` | just the `SendAppearanceWithCOF` packet, no bake |
| `ssb test [target]` | `app` + `http` per target, or every avatar and NPC if none given; PASS / PARTIAL / FAIL. PARTIAL is expected for NPCs; only FAIL is a problem |

## 7. Alignment with the Second Life server

| Aspect | Second Life viewer's contract | CraftSim |
| :--- | :--- | :--- |
| Capability name and verb | `UpdateAvatarAppearance`, HTTP POST | same |
| Request body | LLSD `{cof_version: S32}` | same; missing or malformed gives 400 |
| Response | viewer checks `result["success"]` as boolean | `{success: true}` |
| Region advertises support | `RegionProtocols` bit 0 in `RegionHandshake` | set when the shim is enabled (**upstream: never set**) |
| More than 6 baked textures | bit 63 | always set (as upstream) |
| COF version echoed | `AvatarAppearance` packet with `AppearanceVersion` and `CofVersion` | sent after a successful bake, UDP clients only |
| Source of the bake | the **Current Outfit Folder** held in inventory (hence `cof_version`) | the wearables already on `sp.Appearance`; **`cof_version` is only echoed, never compared** with the COF |
| Failure reporting | SL can answer an error to the POST | **always `success: true`** with HTTP 200, even if the bake was skipped or failed; the only signal is the missing COF echo |
| Bake set | the standard bake types | Head, UpperBody, LowerBody, Eyes, Hair, Skirt (if worn); no per-layer extras were verified |
| Bakes on Mesh | separate feature | `BakesOnMeshEnabled` always advertised, independent of SSB (as upstream) |

> [!WARNING] Unverified or open
> - **No viewer end-to-end result is recorded** in the repo for a full login with the shim on; the `ssb` commands were run against live avatars and NPCs, which proves the pieces, not the viewer behaviour.
> - **Because the bake ignores the COF, a viewer whose COF differs from what the server thinks is worn gets a bake of the server's idea.** The most likely source of "wrong outfit after relog" reports.
> - `m_lastBakedAssetID` (previous-bake cleanup) is an in-memory static dictionary, so after a restart the first rebake cannot delete the previous asset: one orphan per avatar and bake type. Inferred from the code, not measured.
> - `tasks/ssb_investigation_todo.md` is the **pre-implementation handoff** and still says nothing was implemented; the code has overtaken it.
> - `CLAUDE.md` § "Server-Side Baking shim" is the short repo-side summary of this page.
