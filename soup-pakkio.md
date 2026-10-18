# soup-pakkio

> A small demo of [Soup](https://github.com/MakazhanAlpamys/Soup) (a CLI for fine-tuning LLMs)

## Concetti
[[Python]] [[RAG]]

## Panoramica
# soup-pakkio

A small demo of [Soup](https://github.com/MakazhanAlpamys/Soup) (a CLI for fine-tuning LLMs)
on a GTX 1080 (8 GB), plus a retrieval (RAG) alternative for answering questions about Soup.

## What's here

| File | Purpose |
|---|---|
| `soup.yaml` | LoRA SFT config for `google/gemma-3-1b-it` (4-bit, r=32, 5 epochs) |
| `make_data.py` | Builds `data/train.jsonl`: Soup Q&A from the README/commands docs (each phrasing x3) plus 150 `alpaca-cleaned` rows |
| `compare.sh`, `test_v2.sh` | Scripted `soup chat` tests of the trained adapters |
| `rag_demo.py`, `run_rag.sh` | BM25 retrieval over Soup's `README.md` and `docs/commands.md`, answered by the base model |

Adapters, checkpoints and logs are gitignored.

## Setup

```bash
git clone https://github.com/MakazhanAlpamys/Soup ~/w/Soup
cd ~/w/Soup && uv venv .venv && . .venv/bin/activate
uv pip install -e ".[train]"
```

The scripts expect the Soup checkout at `~/w/Soup` (with its `.venv`) and an optional
`~/w/.env` holding `HF_TOKEN`.

On a GPU whose driver supports only CUDA 12.6 (as here), install torch's cu126 build:

```bash
uv pip install torch==2.14.1+cu126 --index-url https://download.pytorch.org/whl/cu126 \
  --extra-index-url https://pypi.org/simple --index-strategy unsafe-best-match
```

Without a C compiler, Triton fails at step 0 ("Failed to find C compiler"). Either install
`build-essential` and `python3-dev`, or run with `TORCH_DISABLE_NATIVE_JIT=1`.

## Fine-tune

```bash
python make_data.py
TORCH_DISABLE_NATIVE_JIT=1 soup train --config soup.yaml --yes
soup chat --model output_v2
```

## RAG demo

```bash
bash run_rag.sh
```

Questions about commands and versions get a short, exact answer. Conceptual questions get a
3-5 sentence answer with the README overview pinned into the prompt. Questions the docs don't
cover get "The documentation does not cover that."

## Results

- **Gemma 4 E2B did not fit.** `device_map="auto"` placed only 24 of 46 modules on the GPU
  (about 1.5 GB), spilling the rest, including the large per-layer embedding table, to CPU.
  Soup rejects CPU offload, so training aborted.
- **Gemma 3 1B trains fine:** about 1.5 s/step at about 5 GB peak VRAM.
- **Fine-tuning taught the style, not the facts.** After the Soup-specific retrain the model
  describes Soup as a CLI fine-tuning tool, but it still invents commands and flags (for
  example a nonexistent `soup export --output gggf`) and gets the Python versions wrong.
- **Retrieval gave correct commands and versions** (`soup train --config soup.yaml`,
  `soup export ... --format gguf`, Python 3.10-3.12), and is easy to keep current.

Retrieval is a hand-written BM25 with keyword routing, so it is a demo rather than a robust
system, and the answers come from a 1B model, so they can still add small inaccuracies.

## Tech Stack
- Linguaggi: Python (2)
- File totali (scan): 9
- Estensioni principali: .sh=3, .py=2, .yaml=1, (noext)=1, .md=1, .jsonl=1
- Remote: `git@github.com:pakkio/soup-pakkio.git`

## Struttura (top-level)
```
README.md
compare.sh
data
make_data.py
rag_demo.py
run_rag.sh
soup.yaml
test_v2.sh
```

## Documentazione presente nel repo
- `README.md`

## Progetti correlati
- [[jev-biblio]]
- [[jev-news]]
- [[jev-pakkio]]
- [[jev-rag]]
- [[mragent]]
- [[mragent-backup]]
- [[search]]

## Link
- Repository: git@github.com:pakkio/soup-pakkio.git
- Path locale: `/home/pakkio/w/soup-pakkio`
