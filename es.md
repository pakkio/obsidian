# es

> Wrapper Python per Everything Search (`es.exe` di voidtools) con ricerca file istantanea su Windows, server MCP, app Gradio e documentazione Sphinx.

## Concetti
[[Python]] [[MCP]] [[CLI]]

## Panoramica
Interfaccia Pythonica (repo: https://github.com/pakkio/es) sopra `es.exe` per query istantanee sul filesystem NTFS Windows (filtri regex, dimensioni, date, attributi, export CSV), completa di:
- **`mcp_server.py` / `start_mcp_server.py`**: espone la ricerca file locale agli agenti AI via MCP.
- **`gradio_app.py`**: interfaccia web locale.
- Suite di test (`test_fixed_search.py`, `test_mcp.py`, `test_timeout_fix.py`) e documentazione Sphinx (`docs/`).

```mermaid
flowchart LR
  Caller["CLI / Gradio / Client MCP"] --> Wrap["everything_search.py<br>+ mcp_server.py"]
  Wrap --> ES["es.exe (voidtools Everything)"]
  ES --> NTFS["Indice NTFS istantaneo"]
```

## Tech Stack
- Python 3.8+ (Poetry), Gradio, MCP, Sphinx.
- Repository: https://github.com/pakkio/es (pubblico, verificato 2026-11-27)

## Collegamenti
[[search]], [[gemmafunction]], [[MCP]]
