# pak

> Impacchetta un intero folder per analizzarlo con AI.

## Concetti
[[LLM]] [[CLI]] [[Python]]

## Panoramica
CLI che comprime semanticamente più file in un formato adatto agli LLM e riestrae il codice modificato nella struttura giusta. Funzioni: raccolta file con esclusioni (.git, venv, node_modules), compressione semantica (AST e anche LLM), diff a livello di metodo (`PAKDIFF.md`), prioritizzazione, gestione del budget di token, multi-linguaggio.

File: `pak.py`, `pak_compressor.py`, `pak_differ.py`, `semantic_compressor.py`, `pak_analyzer.py`, `ast_helper.py`.

## Tech Stack
Python (Poetry, tiktoken). Repo: https://github.com/pakkio/pak

## Collegamenti
[[pakagent]], [[rlm]]
