# MCP

Model Context Protocol: server e client standardizzati per collegare agenti AI (Claude, Gemini, GPT) a editor 3D, vault, database, browser, file locali e simulatori di mondi virtuali.

```mermaid
flowchart LR
  Client["Client AI (Claude / Gemini / Codex)"] -->|"stdio / HTTP JSON-RPC"| Hub["Server MCP"]
  Hub --> V["Vault & Docs: fastobsidian, es, mcp-crawler, mcp-dotenv"]
  Hub --> W3D["3D & Virtual Worlds: mcp-blender, mcp-cck, mcp-unity, oslsl, CraftSim, CraftStorm"]
  Hub --> AI["LLM & Web: mcp-local_llm, jev-pakkio, wordpress, mcpweb"]
```

## Progetti collegati

- [[fastobsidian]]
- [[mcp-blender]]
- [[mcp-cck]]
- [[mcp-unity]]
- [[mcp-crawler]]
- [[mcp-dotenv]]
- [[mcp-local_llm]]
- [[mcp-wolfvoice]]
- [[mcpweb]]
- [[oslsl]]
- [[es]]
- [[wordpress]]
- [[gemini-desktop]]
- [[jev-pakkio]]
- [[cck]]
- [[CraftSim]]
- [[CraftStorm]]
