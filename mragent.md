# mragent

> A lightweight, multi-role LLM agent with a persistent graph-based memory system and hybrid retrieval (FTS5 BM25 + embeddings + rerank). Designed for CLI-first workflows where you want an AI that can r

## Concetti
[[AI Agents]] [[Database]] [[LLM]] [[Python]] [[RAG]]

## Panoramica
# mragent

A lightweight, multi-role LLM agent with a persistent graph-based memory system and hybrid retrieval (FTS5 BM25 + embeddings + rerank). Designed for CLI-first workflows where you want an AI that can remember, retrieve, and reason over accumulated knowledge — everything lives in a single SQLite file plus an optional FAISS index, no heavyweight framework.

---

## Why mragent?

Most LLM tools treat every session as stateless. You repeat context, re-explain background, or paste in documents every time. mragent fixes this by giving the agent a persistent, structured memory it can write to and read from across sessions.

The backbone is a graph of **cues** (specific keywords), **tags** (broad topics), and **content** (the actual text) — traversed either locally or by the LLM itself. On top of that, recall is hybrid: FTS5 BM25 keyword search and embedding similarity run in parallel, are fused with reciprocal rank fusion, and optionally re-ranked with Cohere Rerank. Each layer degrades gracefully — no FAISS installed means brute-force cosine, no embeddings means BM25 only, no FTS5 means an in-memory BM25 with Snowball stemming and diacritic folding.

---

## Architecture

```
main.py          — CLI entry point, command dispatch
llmwrapper.py    — thin HTTP wrapper over OpenRouter (no SDK dependency):
                   chat, tools, embeddings, Cohere rerank, OCR
agent/
  agent.py       — MemoryAgent: store, recall (fast + deep), PDF ingest, RRF fusion
memory/
  graph.py       — Cue-Tag-Content graph on SQLite: FTS5 BM25, FAISS vector
                   search (optional), in-memory BM25 fallback, query expansion
  controller.py  — LLM tool schemas (search_memory, get_by_tag, …)
memory.db        — SQLite database: nodes, edges, embeddings, FTS5 index
                   (auto-created on first write)
memory.faiss     — persisted FAISS index (auto-rebuilt from memory.db)
.ocr_cache/      — per-batch OCR text checkpoints, keyed by docname
                   (gitignored; lets an interrupted --batch-pages ingest
                   resume without re-paying for already-OCR'd pages)
```

### The Cue-Tag-Content Graph

Every piece of stored knowledge is represented as three node types connected by edges:

```
cue:"python"  ──►  tag:"programming"  ──►  content:"Python is a …"
cue:"GIL"     ──►  tag:"concurrency"  ──►  content:"The Global …"
```

- **Cue nodes** are specific keywords or named entities (3–6 per item). They point to tag nodes.
- **Tag nodes** are broad topic categories (2–3 per item). They point to content nodes.
- **Content nodes** hold the raw text of a stored memory chunk.

This two-level indirection lets the system answer both narrow queries ("find everything about GIL") and broad queries ("what do I know about concurrency?") without embedding arithmetic.

Nodes are deduplicated: storing two items with the same tag reuses the existing tag node and fans out its edges. The graph lives in a single SQLite database (`memory.db`, WAL mode) committed per-write, protected by a threading lock for concurrent ingestion. A legacy `memory.json` is migrated automatically on first run.

---

## Commands

### One-shot / stateless

```bash
# Ask a single question (no memory)
python main.py ask What is the capital of France?

# Interactive multi-turn chat
python main.py chat --system "You are a concise tutor."

# Summarize a file or stdin (3–5 bullet points)
python main.py summarize report.txt
cat report.txt | python main.py summarize

# Multi-step research pipeline (brainstorm → distill → briefing)
python main.py research "Byzantine fault tolerance"
```

### Memory-aware

```bash
# Store text with LLM-extracted cues and tags
python main.py remember "The GIL prevents true thread parallelism in CPython."

# Store with manually specified metadata (skips LLM extraction call)
python main.py remember --cues "GIL,CPython,threads" --tags "python,concurrency" \
    "The GIL prevents true thread parallelism in CPython."

# Store from a file
pyt

## Tech Stack
- Linguaggi: Python (9)
- File totali (scan): 82
- Estensioni principali: .txt=42, (noext)=13, .py=9, .json=2, .md=2, .12=2, .faiss=1, .db=1
- Remote: `git@github.com:pakkio/mragent.git`

## Struttura (top-level)
```
Advanced_Physics_Nick_Lucid_July_2023.pdf
README.md
__pycache__
agent
config.json
llmwrapper.py
main.py
memory
memory.db
memory.faiss
memory.faiss.ids.npy
physics.txt
poetry.lock
pyproject.toml
test_llmwrapper.py
test_ranking.py
```

## Documentazione presente nel repo
- `.pytest_cache/README.md`
- `README.md`

## Manifest
### pyproject.toml
```
[project]
name = "mragent"
version = "0.1.0"
description = ""
authors = [
    {name = "Claudio Pacchiega",email = "claudio.pacchiega@valuepartners.com"}
]
requires-python = ">=3.12"
dependencies = [
    "python-dotenv (>=1.2.2,<2.0.0)",
    "snowballstemmer (>=3.1.0,<4.0.0)"
]

[project.optional-dependencies]
# FAISS vector search — memory/graph.py degrades gracefully to brute-force cosine without it
vector = ["faiss-cpu (>=1.9.0,<2.0.0)", "numpy (>=2.0,<3.0)"]
# --batch-pages parallel PDF OCR
pdf = ["pypdf (>=5.0,<7.0)"]


[build-system]
requires = ["poetry-core>=2.0.0,<3.0.0"]
build-backend = "poetry.core.masonry.api"

[dependency-groups]
dev = [
    "pytest (>=9.1.1,<10.0.0)"
]

[tool.pytest.ini_options]
markers = [
    "live: calls the real OpenRouter API (requires OPENROUTER_APIKEY)",
]
```

## Progetti correlati
- [[agent]]
- [[chatgroup]]
- [[gemmafunction]]
- [[jev-rag]]
- [[llmwiki]]
- [[llmwiki1]]
- [[local_llm]]
- [[mragent-backup]]
- [[nexus]]
- [[rlm]]
- [[soup-pakkio]]

## Link
- Repository: git@github.com:pakkio/mragent.git
- Path locale: `/home/pakkio/w/mragent`
