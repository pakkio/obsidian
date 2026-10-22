# nexus2

> Evoluzione di Nexus: NPC con memoria a wiki e conversazioni su database.

## Concetti
[[AI Agents]] [[LLM]] [[Python]] [[Database]]

## Panoramica
Motore di GDR testuale AI ambientato in Eldoria: wiki markdown persistente letta/scritta dagli NPC via tool calling, agenti NPC LLM (DeepSeek V4-Flash o OpenRouter), profilo psicologico del giocatore, pannello HTTP sulla porta 6001. Dettagli in `TECHNICAL.md`; README aggiunto il 2026-10-18. Moduli: `main.py`, `llm.py`, `db_manager.py` + `database_schema.sql`, `wiki.py` e cartella `wiki/` (memoria come pagine wiki), `NPCs/`, `generate_conversation.py`, script di backfill/migrazione (`backfill_conversation_pages.py`, `migrate_titles.py`, `sort_links.py`), misure (`measure_wiki_tool_latency.py`) e `simulate_jorin_prompt.py`.

## Tech Stack
Python, DB SQL. Repo: https://github.com/pakkio/nexus2 (2026-05)

## Collegamenti
[[nexus]], [[ppd]], [[nexus-narrative-weaver]], [[llmwiki]]
