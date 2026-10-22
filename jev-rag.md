# jev-rag

> A small, readable Retrieval-Augmented Generation (RAG) web app. You upload documents, ask

## Concetti
[[Database]] [[LLM]] [[Python]] [[RAG]]

## Panoramica
# Tiny RAG + Jev

A small, readable Retrieval-Augmented Generation (RAG) web app. You upload documents, ask
questions, and get answers with citations. The right-hand panel shows each pipeline step: its
timing, inputs and outputs.

It compares four ways to produce the final answer:

| Engine | What it does | Runs on | Cost |
|---|---|---|---|
| **Local LLM** (default: `qwen3:14b`) | Writes an answer with `[n]` citations | Your GPU, via [Ollama](https://ollama.com) | Free |
| **Jev** ([TypeSafe](https://docs.typesafe.ai)) | *Selects* the sentence that answers the question, or says "not in the documents" — and, in parallel, has a drafted detailed answer **verified sentence by sentence** | TypeSafe + OpenRouter | Per token |
| **Flash** (`OPENROUTER_MODEL`: default `google/gemini-2.5-flash-lite`, here `deepseek-v4-flash-0731`; via [OpenRouter](https://openrouter.ai)) | Writes an answer with `[n]` citations | OpenRouter API | Per token |
| **Claude** (optional, off by default) | Writes an answer with `[n]` citations | Anthropic API | Per token, prepaid credits |

**Stack:** FastAPI · fastembed (local ONNX embeddings, on the GPU via CUDA when available) · a numpy vector store · Ollama · TypeSafe SDK · Anthropic SDK · OpenRouter (Flash) · vanilla HTML/JS. Requires **Python 3.10+**. A [`justfile`](justfile) wraps the common commands (see below).

## The RAG workflow

RAG has two pipelines. **Ingest** runs once per document. **Query** runs on every question.

```
 INGEST (per document)
 ┌────────┐   ┌─────────┐   ┌──────────┐   ┌──────────────┐
 │  Load  │──▶│  Chunk  │──▶│  Embed   │──▶│    Store     │
 │ pdf/md │   │ 800 chr │   │ 768-dim  │   │ vectors.npy  │
 │ → text │   │ +overlap│   │ vectors  │   │ chunks.json  │
 └────────┘   └─────────┘   └──────────┘   └──────┬───────┘
                                                  │ similarity search
 QUERY (per question)                             ▼
 "question?" ─▶ Embed ─▶ Retrieve top-k ─┬─▶ Augment ─▶ Generate   Local LLM, Claude or Flash
                                         │                          write an answer citing [1]..[k]
                                         └─▶ Select ─┬─▶ quote      Jev picks one sentence (~0.5 s)
                                                      └─▶ Draft ─▶ Verify
                                                       the Flash model drafts a detailed
                                                       answer, Jev checks each sentence
```

| Step | File | What happens | Knob to experiment with |
|---|---|---|---|
| Load | `rag/loader.py` | PDF/TXT/MD → plain text; drops the second copy of pages some PDFs draw twice | – |
| Chunk | `rag/chunker.py` | Split into overlapping pieces, cut at paragraph or sentence breaks | chunk size, overlap |
| Embed | `rag/embedder.py` | Text → normalized 768-dim vector (`paraphrase-multilingual-mpnet-base-v2`, multilingual; GPU if CUDA loads, else CPU) | embedding model |
| Store | `rag/store.py` | Append vectors + metadata, persist to `data/` | – |
| Retrieve | `rag/store.py` | `scores = vectors @ query_vec`, take top-k | top-k |
| Augment | `rag/generator.py` | Number the chunks `[1]..[k]` and wrap them in `<context>` | system prompt |
| Generate | `rag/local_llm.py`, `rag/gemini_llm.py`, `rag/generator.py` | The LLM answers only from that context and cites `[n]` | model |
| Select | `rag/jev.py` | Code splits chunks into sentences; in **one** request Jev judges "is the answer here?" (Noul) and "which sentence?" (Choice). With the Flash engine also on, an LLM drafts a detailed cited answer in parallel and Jev verifies each sentence as it completes: supported facts kept, partly supported or a character's guesses kept inside `[[ ]]`, unsupported sentences dropped | confidence thresholds |

`rag/pipeline.py` connects the steps and records the trace. `app.py` is the HTTP API.
Each retrieved source also names the **chapter** its chunk belongs to (see below).

### Beyond top-k: book cards, chapters and librar

## Tech Stack
- Linguaggi: Python (17)
- File totali (scan): 243
- Estensioni principali: (noext)=79, .lock=48, .json=35, .pdf=26, .py=17, .refs=14, .md=3, .npy=3
- Remote: `git@github.com:pakkio/gev-rag.git`

## Struttura (top-level)
```
README.md
__pycache__
app.py
bench.py
book_bench.py
book_questions
data
data.bak_20260929_2240
data.bak_20261003_1605
docs
justfile
logs
models
rag
requirements.txt
sample_docs
static
uploads
```

## Documentazione presente nel repo
- `README.md`
- `docs/predictions.md`
- `sample_docs/acme_handbook.md`

## Manifest
### requirements.txt
```
fastapi
uvicorn[standard]
python-multipart
pypdf
numpy
fastembed-gpu
onnxruntime-gpu[cuda,cudnn]
# The WSL NVIDIA driver (CUDA UMD 13.3) segfaults in cudaSetDevice with the 13.4 runtime
# that onnxruntime-gpu pulls by default; 13.0 is what Ollama ships and works.
nvidia-cuda-runtime==13.0.*
nvidia-cublas==13.0.*
nvidia-cuda-nvrtc==13.0.*
nvidia-nvjitlink==13.0.*
anthropic
typesafe-sdk
ollama
requests
```

## Progetti correlati
- [[jev-biblio]]
- [[jev-news]]
- [[jev-pakkio]]
- [[mragent]]
- [[mragent-backup]]
- [[search]]
- [[soup-pakkio]]

## Link
- Repository: git@github.com:pakkio/gev-rag.git
- Path locale: `/home/pakkio/w/jev-rag`
