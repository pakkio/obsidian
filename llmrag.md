# llmrag

> Sistema RAG su PDF con ricerca ibrida, reranking e confidence.

## Concetti
[[RAG]] [[LLM]] [[Database]] [[Python]]

## Panoramica
Ricerca ibrida semantica (OpenAI text-embedding-3-large) + keyword (SQLite FTS5/BM25), pesi 60/40 configurabili, reranking LLM (Gemini 2.0 Flash Lite), query enhancement adattivo, confidence con semafori, rilevamento lingua e risposte multilingua. Interfacce: CLI `query.py` e web `gradio_browser.py`; ingest in `ingest.py`.

## Tech Stack
Python, SQLite FTS5, Gradio. Repo: https://github.com/pakkio/llmrag

## Collegamenti
[[jev-rag]], [[soup-pakkio]], [[mragent]]
