---
tags:
  - craftsim
  - flowing-water
  - water
  - modules
  - mcp
aliases:
  - Flowing water
  - Minecraft-style water
---

# Module - Flowing Water

← Back to [[CraftSim]] | Related: [[MCP Server & Live Diagnostics]], [[Configuration Reference (ini files)]], [[Avatar, NPC & Region Modules]]

Source of truth: `doc/FLOWING-WATER.md` in the repo (written 2026-11-12 from the code, `OpenSim/Region/OptionalModules/World/FlowingWater/`).

Minecraft-style water made of prims. It is a region module (`[FlowingWater]`), off by default, enabled on craft. Code:
`OpenSim/Region/OptionalModules/World/FlowingWater/` (`WaterFlow.cs` rules, `FlowingWaterModule.cs` scene side,
`WaterMeshBuilder.cs` mesh assets). Tests: `FlowingWaterTests` (rules, no scene) and `FlowingWaterModuleTests` (a real Scene).

## What it does

Put a prim named **`Water Source`** (the name is configurable, compared without regard to case) in a region. Water then:

- **falls first** from the source, one block per wave;
- **spreads sideways** over whatever it lands on, up to `MaxDistance` blocks, getting shallower (level 1, 2, ...) with distance;
- **stops** at terrain and at non-phantom prims (phantom prims are not solid, so water passes through them);
- **drains** when the source prim is deleted, again one wave at a time.

A source placed on a cell that already has flowing water takes it over. Sources are found again every `RescanSeconds`, so moving
or deleting one is noticed within that time. Blocks are `BlockSize` metres on a side (1 m by default), on a grid aligned to the
region origin, the same grid as the CraftStorm viewer's block mode, so water and built blocks line up.

### The rules (`WaterFlow`)

A cell holds nothing, a source (level 0, never changed by the flow), falling water, or flowing water at level 1..`MaxDistance`.
Every non-source cell is a function of its neighbours:

1. water directly above it -> falling;
2. otherwise a horizontal neighbour that can spread (it rests on a solid cell or on non-falling water) at distance d -> level d + 1,
   the smallest such, while it stays within `MaxDistance`;
3. otherwise empty.

Sources and falling water count as distance 0. `Step()` re-evaluates at most `MaxChangesPerTick` dirty cells per tick; a change
dirties the six neighbours, so the water advances (and drains) one wave per tick, which is what looks like flow.
`MaxBlocks` caps the number of water cells per region (sources not counted).

## Two ways to draw it

| | `Mesh = true` (default in code and on craft) | `Mesh = false` (block mode) |
|---|---|---|
| Representation | one **mesh prim per tile** of 16 x 16 cells (any height), named `Water` | one phantom, translucent **box prim per wet cell** |
| Description tag | `craftstorm-water:mesh:<tx>,<ty>` | `craftstorm-water:<level>` |
| Cost | few prims; a tile is rebuilt at most once a second, at most 4 tiles per tick | one persisted prim per block (`MaxBlocks` of them) |
| After a restart | tagged prims are deleted at startup and the sources flow again, so the water is recomputed | water prims are found by their description and restored with their level |

Mesh details: the mesh has two faces (top of the water, and the sides) so the texture animation treats them differently, is
translucent blue, and carries a scrolling texture animation. The corners of a cell are shared with its neighbours, so a change
redraws the tiles around it as well. Each rebuild creates a new mesh asset; the last 3 assets of a tile are kept and older ones
deleted. A tile prim deleted by hand is drawn again. In block mode a water prim deleted by hand comes back if the rules say so.

Water prims are owned by whoever owns the source. Do not reuse the description prefix `craftstorm-water:` for other prims:
`water purge` deletes everything that starts with it.

## Configuration (`[FlowingWater]` in `bin/OpenSim.ini`; needs a simulator restart)

| Key | Default | Notes |
|---|---|---|
| `Enabled` | `false` | everything below is ignored when off, and the MCP tools answer "flowing water is not enabled for this region" |
| `BlockSize` | `1.0` | metres, clamped to 0.25 - 10 |
| `SourceName` | `Water Source` | prim name that makes a source |
| `TickMs` | `500` | one wave of flow per tick, at least 50 |
| `MaxChangesPerTick` | `64` | cells changed per tick, at least 1 |
| `MaxDistance` | `7` | how far water spreads, **clamped to 1 - 15**: a larger value is silently lowered to 15 |
| `MaxBlocks` | `2000` | water cells per region, sources not counted |
| `RescanSeconds` | `5` | how often sources and solid prims are looked at again, at least 1 |
| `Mesh` | `true` | mesh tiles instead of one prim per block, see above |

craft (2026-11-12): `Enabled = true`, `MaxDistance = 50` (acts as 15 because of the clamp), `MaxBlocks = 10000`, `Mesh = true`,
`MaxChangesPerTick = 400`.

## Commands

Console, in the selected region (also on `IFlowingWaterModule`, which the MCP tools use):

| Command | What it does |
|---|---|
| `water status` | sources, water blocks, prims, cells still waiting to settle (0 = at rest) |
| `water sources` | lists the sources as block-centre positions |
| `water source <x> <y> <z>` | rezzes a source in the block containing that region position, owned by the estate owner |
| `water drain` | deletes all flowing water; sources stay and start flowing again, so delete the source prims to stop the water for good |
| `water purge` | drains, then deletes **every** prim whose description starts with `craftstorm-water:`, including water the module lost track of (for example left over from before a restart), in one batch: the objects are deleted without a message each and viewers get a few large kill messages (250 ids each), which is much faster than deleting one by one |

MCP tools (`WaterTools.cs`): `water_status(regionName)`, `water_sources(regionName)`,
`water_add_source(regionName, x, y, z, ownerId)` (returns the new prim's UUID; the owner defaults to the estate owner) and
`water_drain(regionName, confirm)` (`confirm` must be true; returns how many blocks were removed). There is no MCP tool for
`water purge`; use the console.

## Practical notes

- A source needs a few seconds before water shows; check that `water status` shows 0 cells waiting to know it has settled.
- Large areas are limited by `MaxBlocks` and by `MaxChangesPerTick` per tick: a big flood takes many waves to fill.
- Anything solid (terrain or a non-phantom prim) in the way stops the water; make a prim phantom to let water through.
- If water prims pile up (for example after switching `Mesh` on or off), run `water purge` and let the sources flow again.
