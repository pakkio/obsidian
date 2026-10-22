# mcp-crawler

> Server MCP che crawla un URL e lo indicizza in SQLite FTS5.

## Concetti
[[MCP]] [[JavaScript]] [[Web]] [[Database]]

## Panoramica
Server MCP che scarica una pagina, ne estrae il testo (strip HTML), lo indicizza in SQLite FTS5 e lo cerca con regex o testo semplice, senza AI nel retrieval. Tool: `crawl_url`, `search_text`, `search_regex`, `list_indexed_pages`, `crawler_stats`, `clear_index`. Moduli: `crawler.js`, `db.js`, `search.js`, `summarize.js` (node-llama-cpp, con test). Nella skill `verbosity-control` ([[skills]]) è l'alternativa locale a WebFetch/WebSearch per tenere i dati fuori dalla conversazione. README aggiunto il 2026-10-18.

## Tech Stack
- JavaScript / Node.js, better-sqlite3 (WAL), `@modelcontextprotocol/sdk`
- Repo: https://github.com/pakkio/mcp-crawler

## Collegamenti
[[mcp-dotenv]], [[mcp-local_llm]], [[skills]]
