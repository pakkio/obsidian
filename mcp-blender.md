# mcp-blender

> Exposes Blender to MCP clients (Claude Code, Claude Desktop, Antigravity, and others) through a

## Concetti
[[Blender]] [[MCP]] [[Python]]

## Panoramica
# mcp-blender (v2.3.5)

Exposes Blender to MCP clients (Claude Code, Claude Desktop, Antigravity, and others) through a
high-performance two-process bridge, mirroring [mcp-unity](https://github.com/claudiopacchiega/mcp-unity)'s
architecture but built in pure Python for Blender's native runtime:

```
MCP client (Claude / Gemini / GPT-4) <--stdio--> mcp_server (pip package) <--WebSocket--> extension (Blender 4.2+ / 5.x)
```

- **`extension/`** -- a Blender 4.2+ & 5.x Native Extension (`blender_manifest.toml`). Runs a
  localhost WebSocket server inside Blender and executes tool calls safely on Blender's
  main thread with non-modal viewport HUD rendering.
- **`mcp_server/`** -- a pip-installable Python package that is the actual MCP
  server (stdio transport, via FastMCP). It communicates over WebSockets directly to Blender.

---

## 🌟 Why `mcp-blender` is Fundamentally Different & Superior

Existing open-source Blender MCP implementations (e.g. `RFingAdam/mcp-blender`, `ahujasid/blender-mcp`) are typically minimal proofs-of-concept providing only 5 to 15 basic tools and rely heavily on raw `exec(python_code)` strings. 

`mcp-blender` was engineered from the ground up as a **complete 3D production pipeline suite**:

| Capability | Generic Blender MCPs | `mcp-blender` (v2.3.5) |
| :--- | :--- | :--- |
| **Total Tool Count** | ~5 to 15 basic tools | **146 Native Tools / 10 Unified Low-Context Domain Facades** |
| **Context Overhead** | Heavy per-tool bloat | **Ultra-Low Context Mode (90% token reduction) with on-demand `blender_docs`** |
| **Architecture** | Legacy Blender 2.8/3.x zip addons | **Blender 4.2+ & 5.2+ Native Extension System** |
| **Transactional Safety** | ❌ None (scene corrupts on fail) | **`execute_batch` (with automatic snapshot rollback on failure)** + **`create_scene_checkpoint`** / **`restore_scene_checkpoint`** |
| **Async Background Jobs** | ❌ UI freezes indefinitely | **`get_job_status`**, **`cancel_job`**, **`list_jobs`** |
| **Multimodal Vision Feedback** | ❌ None (blind execution) | **`capture_multiview_audit`** (4-angle contact sheets returned as real MCP image content) + **`inspect_focus_shot`** + **`evaluate_scene_visually`** (Gemini 2.5 Flash VLM fallback) |
| **Online & AI 3D Sourcing** | ❌ None | **Poly Haven, Sketchfab, ambientCG, Meshy AI, Tripo3D, Trellis 3D** |
| **IK Rigging & Metarigs** | ❌ Basic single bone only | **`setup_humanoid_rig_preset`** (biped metarig), **`setup_ik_constraint`** (pole targets/angles), **`setup_spline_ik_constraint`** |
| **Blender 4.2+ Hair Curves** | ❌ Legacy particle only | **`create_hair_curves`**, **`apply_hair_groom_modifier`** (Frizz, Clump, Noise, Braid), **`convert_legacy_hair_to_curves`** |
| **Grease Pencil & VFX Matchmoving** | ❌ None | **GPv3 layers & strokes**, **`setup_camera_tracking`**, **`setup_vfx_shadow_catcher`** |
| **Batch Latency Optimization** | ❌ Slow roundtrips (1 tool/turn) | **`execute_batch`** (Executes 50+ actions in a single roundtrip with stop-on-error/optional-step control and per-step reporting) |
| **Non-Modal Progress HUD** | ❌ None / Freezes UI | **`update_progress_hud`** (Real-time GPU 2D floating glass card with progress % and live task logs) |
| **Geometry Nodes Studio** | ❌ None | **Procedural graphs, Point scattering, Curve profiling, VDB volume remeshing, Proximity effectors** |
| **Shader & Material Studio** | ❌ Flat single colors | **Procedural edge-wear grunge masks, Triplanar box mapping, Auto-PBR texture folder loader, Specialty shaders** |
| **Game Engine Pipelines** | ❌ Raw unoptimized exports | **`export_unity_fbx`** (fixes Unity $-90^\circ$ X-axis bug), **`generate_lods`** (LOD0..LODn), Draco GLTF/GLB, Animation Keyframe Baking |

---

## ⚡ Low-Context Unified Domain Controllers (`MCP_BLENDER_TOOL_MODE=AGGREGATED`)

By default, `mcp-blender` exposes **10 unified domain facade tools** that cut LLM context consumption by **90%** while retaining 100% of underlying pipeline features:

1. **`blender_docs`**: Query multi

## Tech Stack
- Linguaggi: Python (193)
- File totali (scan): 262
- Estensioni principali: .py=193, (noext)=22, .md=14, .whl=12, .json=4, .yaml=2, .toml=2, .12=2
- Remote: `git@github.com:pakkio/mcp-blender.git`

## Struttura (top-level)
```
LICENSE
README.md
TODO.md
extension
mcp_server
opencode.json
scripts
tasks
tests_live
tools
```

## Documentazione presente nel repo
- `README.md`
- `TODO.md`
- `tasks/lessons.md`
- `tasks/query_log.md`
- `tasks/todo.md`
- `tools/mcp-blender-pp-cli/AGENTS.md`
- `tools/mcp-blender-pp-cli/BUILD_LOG.md`
- `tools/mcp-blender-pp-cli/CHANGELOG.md`
- `tools/mcp-blender-pp-cli/CLAUDE.md`
- `tools/mcp-blender-pp-cli/README.md`
- `tools/mcp-blender-pp-cli/SKILL.md`

## Progetti correlati
- [[cck]]
- [[mcp-unity]]

## Link
- Repository: git@github.com:pakkio/mcp-blender.git
- Path locale: `/home/pakkio/w/mcp-blender`
