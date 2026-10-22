# jev-pakkio

> One-pass option scoring with a local Gemma 3 4B, on Apple silicon via MLX or on

## Concetti
[[LLM]] [[Python]]

## Panoramica
# jev_pakkio

One-pass option scoring with a local Gemma 3 4B, on Apple silicon via MLX or on
NVIDIA/CPU via PyTorch (`--backend torch`).
Design notes: [docs/design/one-pass-option-scoring.md](docs/design/one-pass-option-scoring.md);
per-task training: [docs/design/per-task-finetuning-with-gemma.md](docs/design/per-task-finetuning-with-gemma.md).

Given a context and a list of pre-written options, the model prefills the
context once, expands that KV cache across the option batch, and scores every
option in a single padded forward pass. No decoding. The score is the
log-probability of the option tokens given the context; a softmax over the
option scores gives a probability per option, like `jevlike-predict`.

## Demo: Doom in the terminal

`demo/doom/` runs ViZDoom headless, describes each frame in a line of text, and
lets the server rank the action menu with one `/score` call (or one System One
`choice` question). Start `make serve`, then `make doom`. Details and keys in
[`demo/doom/README.md`](demo/doom/README.md).

![jev_pakkio playing Doom in the terminal: the model ranks the action menu each step](docs/media/doom-recording.gif)

Full-resolution recording: [docs/media/doom-recording.mov](docs/media/doom-recording.mov).

## Setup

### Windows and Linux

Install Python 3.12+ and [uv](https://docs.astral.sh/uv/getting-started/installation/), then run these commands from the repository in PowerShell or a Linux shell:

```sh
uv sync
uv run hf auth login
uv run hf download google/gemma-3-4b-it --local-dir models/gemma-3-4b-it
uv run jev_pakkio serve --backend torch --device auto --port 8000
```

Accept the Gemma license on Hugging Face before downloading. You can also pass
`--model google/gemma-3-4b-it` to load directly from the Hugging Face cache.
No Make, Xcode, shell activation, or `.venv/bin` paths are needed here.

`--backend auto` (the default) selects MLX on Apple silicon and PyTorch elsewhere.
For PyTorch, `--device auto` selects CUDA when available, then MPS, then CPU.
Use `--device cuda` or `--device cuda:1` to require a specific GPU (fails clearly
if CUDA is unavailable), or `--device cpu` to force CPU inference.

For NVIDIA acceleration, install the GPU driver and a matching PyTorch build using
the [official PyTorch installer](https://pytorch.org/get-started/locally/).
Run its pip command in this repository's virtual environment (use `uv pip install`
in place of `pip3 install`). After a custom PyTorch install, use `uv run --no-sync`
so uv does not replace your selected build. On supported Linux AMD systems,
PyTorch ROCm builds also use the `cuda` device name.

```sh
uv run --no-sync python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
uv run --no-sync jev_pakkio serve --backend torch --device cuda --batch-size 2
uv run --no-sync jev_pakkio check --backend torch --device cuda
uv run --no-sync jev_pakkio score --backend torch --device cuda --context "The capital of France is" --option " Paris" --option " Berlin"
```

The base 4B weights need roughly 8 GB just for GPU model weights at 16-bit precision,
plus memory for activations and the option KV caches. Reduce `--batch-size` and
context length if you run out of GPU memory. CPU runs use float32 and need more RAM.
Use original Hugging Face weights with PyTorch; MLX quantized weights and MLX LoRA
adapters are not supported by this backend.

Scoring, evaluation, benchmarks, correctness checks, and both HTTP scoring endpoints
support PyTorch. Frozen-feature extraction, head training/evaluation, and the chess
LoRA training workflow still require MLX on Apple silicon. The Doom terminal UI
also has its own platform dependencies; cross-platform support here covers the
scoring CLI and HTTP server.

### Apple silicon / existing Make workflow

```sh
make setup       # uv sync (arm64 Python 3.12 venv) + download google/gemma-3-4b-it into models/ (gated; needs HF login)
make serve       # start the HTTP server on :8000
make health      # curl /health
make re

## Tech Stack
- Linguaggi: Python (69)
- File totali (scan): 350
- Estensioni principali: .py=69, .jsonl=62, (noext)=52, .json=45, .md=34, .log=22, .safetensors=18, .sh=14
- Remote: `git@github.com:pakkio/jev-pakkio.git`

## Struttura (top-level)
```
LICENSE
Makefile
README.md
__pycache__
bench_bbc_jev_pakkio.py
bench_classif.py
bench_mem.py
bench_mixed.py
benchmark_bbc.py
benchmark_speed.py
chess_finetune
compare_jev.py
data
demo
docs
examples
finetune
gen_ambig.py
gen_badges.py
gen_movie_tasks.py
gen_movies.py
gen_movies_general.py
gen_movies_hard.py
gen_news.py
gen_noul_categorize.py
jev_pakkio
jevmem-laya.json
justfile
mkdocs.yml
pyproject.toml
runs
tasks
tests
uv.lock
```

## Documentazione presente nel repo
- `.pytest_cache/README.md`
- `README.md`
- `chess_finetune/README.md`
- `demo/doom/README.md`
- `docs/api.md`
- `docs/cli.md`
- `docs/design/one-pass-option-scoring.md`
- `docs/design/per-task-finetuning-with-gemma.md`
- `docs/doom-demo.md`
- `docs/getting-started.md`
- `docs/index.md`
- `docs/jev-research-2026-09-20.md`
- `docs/lora.md`
- `docs/mem.md`
- `docs/training.md`
- `finetune/README.md`
- `runs/bbc/2026-09-26.md`
- `runs/lora-categorize_general_chat-r/README.md`
- `runs/lora-categorize_general_chat/README.md`
- `runs/lora-claims_attrib-1000/README.md`
- `runs/lora-claims_attrib/README.md`
- `runs/lora-claims_attrib_keys-1500/README.md`
- `runs/lora-claims_attrib_keys/README.md`
- `runs/lora-langid/README.md`
- `runs/lora-movies_general-2ep/README.md`
- `runs/lora-movies_general/README.md`
- `runs/lora-movies_general_v2/README.md`
- `runs/lora-movies_hard2k/README.md`
- `runs/lora-news_general/README.md`
- `runs/lora-news_general_chat/README.md`

## Manifest
### pyproject.toml
```
[project]
name = "jev_pakkio"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.141.1",
    "huggingface-hub>=1.31.0",
    "numpy>=2.5.3",
    "safetensors>=0.8.0",
    "transformers>=5.17.0",
    "uvicorn[standard]>=0.53.0",
]

# One backend extra per platform; install exactly the one you can run.
# mlx must not be a base dependency: it installs on Linux but its libmlx.so is
# absent, and transformers' _is_mlx_available() sees the metadata and then dies
# importing it -- breaking the torch backend on any non-Apple host.
[project.optional-dependencies]
mlx = ["mlx>=0.24", "mlx-lm>=0.24"]                          # Apple silicon
torch = [
    "torch==2.9.1; sys_platform == 'linux'",  # the cu128 build verified with bitsandbytes + peft
    "torch>=2.9.1; sys_platform != 'linux'",
    "accelerate>=1.0",
    "bitsandbytes>=0.50.2",
    "peft>=0.21.0",
]  # CUDA / CPU
mem = ["sentence-transformers>=3.0"]                  # jev_pakkio.mem embedder (Jev-Mem)
laya = ["laya"]                                   # engine laya (convaiinnovations/laya)
doom = ["vizdoom>=1.2.4"]                         # demo/doom: play Doom in the terminal with the server choosing actions
mcp = [
    "mcp[cli]>=2.3.0",
    "httpx>=0.27",  # also used by the jev engine
]
typesafe = [
    "typesafe-sdk>=0.7.0",
]        # mcp_server.py's jev tool: hosted api.typesafe.ai


[project.scripts]
jev_pakkio = "jev_pakkio.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = 
```

## Progetti correlati
- [[jev-biblio]]
- [[jev-news]]
- [[jev-rag]]
- [[search]]
- [[soup-pakkio]]

## Link
- Repository: git@github.com:pakkio/jev-pakkio.git
- Path locale: `/home/pakkio/w/jev-pakkio`
