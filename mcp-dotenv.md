# mcp-dotenv

> Server MCP per leggere/gestire file `.env` senza esporre i segreti in chat.

## Concetti
[[MCP]] [[JavaScript]] [[CLI]]

## Panoramica
Server MCP per leggere/gestire `.env` e scansionare file in cerca di segreti **senza esporre i valori** (output mascherato). Tool: `list_env_keys`, `check_env_exists`, `get_env_masked`, `describe_env_value`, `set_env_value`, `delete_env_key`, `scan_file`, `scan_directory`, `redact_file`. Detector regex (AWS, GitHub, Slack, Stripe, JWT, chiavi private...) + fallback a entropia di Shannon. README aggiunto il 2026-10-18. Nella skill `verbosity-control` ([[skills]]) è il server MCP per "secret-handling" locale: i valori restano nel sandbox invece che nel contesto.

## Tech Stack
- JavaScript / Node.js
- Repo: https://github.com/pakkio/mcp-dotenv (creato 2026-08-02)

## Collegamenti
[[mcp-crawler]], [[mcp-local_llm]], [[skills]]
