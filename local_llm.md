# local_llm

> A minimal local implementation of Google's **Gemma 4 E2B** ("Effective 2B") for

## Concetti
[[C++]] [[LLM]] [[Python]]

## Panoramica
# Gemma 4 E2B — Local Chat Demo (llama.cpp, no Ollama)

A minimal local implementation of Google's **Gemma 4 E2B** ("Effective 2B") for
a simple terminal chat. Uses **llama.cpp** directly for inference (GPU-accelerated
with CUDA on NVIDIA), plus a dependency-free Python chat client that talks to
llama.cpp's OpenAI-compatible server over HTTP.

## Reality check on the "800 MB text-only" model

The claim in the video ("~800 MB, text-only") does **not** exist as a real
downloadable GGUF. Verified file sizes on Hugging Face (Aug 2026):

| Build | Format | Size |
|---|---|---|
| Google official QAT q4_0 (`gemma-4-E2B_q4_0-it.gguf`) | GGUF — **this project's default** | **3.35 GB** |
| unsloth UD-IQ2_M (`gemma-4-E2B-it-UD-IQ2_M.gguf`) | GGUF — smallest llama.cpp-runnable | 2.29 GB |
| LiteRT "web" (`gemma-4-E2B-it-web.litertlm`) | Mobile format, **not** llama.cpp-runnable | 2.0 GB |
| BF16 (full) | GGUF | 9.3 GB |
| Google official QAT q4_0, **12B** (`gemma-4-12b-it-qat-q4_0.gguf`) | GGUF — dense 12B variant | 6.98 GB |
| bartowski Q3_K_S, **12B** (`gemma-4-12B-it-Q3_K_S.gguf`) | GGUF — fits fully on an 8GB GPU | 5.72 GB |

The official Q4_0 12B build is too large to fully offload on an 8GB consumer
GPU under WSL2 — partial CPU/GPU offload measured only ~4.5-6.5 tok/s. The
bartowski Q3_K_S build fits entirely in VRAM (`-ngl 99`) and measured
~20-24 tok/s with no accuracy loss on a 10-question spot check versus the
official Q4_0 build (2/10 errors either way).

The "800 MB" figure was almost certainly a marketing/approximation of the mobile
build. For llama.cpp the practical minimum is ~2.3 GB. This project defaults to
Google's official QAT q4_0 (best quality per MB); swap in the smaller one by
overriding `MODEL_URL` (see below).

## Prerequisites

- Linux (WSL2 works great), Python 3.9+, `curl`, `cmake`, `gcc`/`g++`
- For GPU acceleration: NVIDIA GPU + CUDA toolkit (`nvcc`) — otherwise it falls
  back to a CPU-only build automatically

## Quick start

```bash
./setup.sh          # downloads the model (~3.3 GB) + builds llama.cpp
python3 chat.py     # start chatting
```

First run takes a while (model download + compile). Everything is cached, so
subsequent runs are instant.

## Usage

```bash
# Default: auto-start llama.cpp server (CUDA, all layers on GPU), then chat
python3 chat.py

# Force CPU only
python3 chat.py --ngl 0

# Connect to an already-running llama.cpp server instead
python3 chat.py --url http://localhost:8080
```

Chat commands inside the demo:

- `/reset` — clear conversation history
- `/help` — show commands
- `/quit` (or Ctrl+C at the prompt) — exit
- Ctrl+C while Gemma is generating — stop the current reply

## Options / configuration

| Env var | Purpose | Default |
|---|---|---|
| `MODEL_URL` | GGUF download URL | Google QAT q4_0 |
| `LLAMA_TAG` | llama.cpp release tag to build | `b10218` |

Smaller-model example:

```bash
MODEL_URL=https://huggingface.co/unsloth/gemma-4-E2B-it-GGUF/resolve/main/gemma-4-E2B-it-UD-IQ2_M.gguf ./setup.sh
```

Bigger-model example (dense 12B, better quality, fits fully on an 8GB GPU):

```bash
MODEL_URL=https://huggingface.co/bartowski/gemma-4-12B-it-GGUF/resolve/main/gemma-4-12B-it-Q3_K_S.gguf ./setup.sh
```

(The official Q4_0 12B build also exists at
`https://huggingface.co/google/gemma-4-12B-it-qat-q4_0-gguf/resolve/main/gemma-4-12b-it-qat-q4_0.gguf`
but at 6.98 GB it won't fully fit in 8GB of VRAM — expect ~4.5-6.5 tok/s from
partial CPU/GPU offload instead of ~20-24 tok/s.)

`chat.py` auto-picks whichever `.gguf` sorts first in `models/` when `--model`
isn't given, so if you have multiple weights downloaded, pass `--model` to pick
one explicitly, e.g. `python3 chat.py --model models/gemma-4-12B-it-Q3_K_S.gguf`.

## How it works

1. `setup.sh` downloads the GGUF weights and builds llama.cpp (`llama-server`,
   `llama-cli`) from source — with CUDA if `nvcc` is found, else CPU.
2. `chat.py` starts `llama-server` with the model (alias `gemma-4-e2b`),
   waits for it to be ready, t

## Tech Stack
- Linguaggi: Python (111), C++ (103), C/C++ header (61)
- File totali (scan): 499
- Estensioni principali: .py=111, .cpp=103, .h=61, .sh=28, .md=27, .txt=23, .gguf=22, (noext)=21

## Struttura (top-level)
```
README.md
__pycache__
bin
chat.py
llama-server.log
models
setup.sh
vendor
```

## Documentazione presente nel repo
- `README.md`
- `vendor/llama.cpp/AGENTS.md`
- `vendor/llama.cpp/CLAUDE.md`
- `vendor/llama.cpp/CONTRIBUTING.md`
- `vendor/llama.cpp/README.md`
- `vendor/llama.cpp/SECURITY.md`

## Progetti correlati
- [[MegaFuse]]
- [[agent]]
- [[chatgroup]]
- [[craftstorm]]
- [[gemmafunction]]
- [[libopenmetaverse]]
- [[llmwiki]]
- [[llmwiki1]]
- [[mragent]]
- [[opensim-cpp]]
- [[rlm]]
- [[spirale]]

## Link
- Path locale: `/home/pakkio/w/local_llm`
