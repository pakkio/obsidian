# nispa-ocr

> > Convert PDF documents to Markdown — locally, accurately, with no data leaving your machine.

## Concetti
[[JavaScript]] [[OCR]] [[Python]] [[TypeScript]]

## Panoramica
# nispa-ocr

> Convert PDF documents to Markdown — locally, accurately, with no data leaving your machine.

Built on [dots.mocr](https://huggingface.co/rednote-hilab/dots.mocr), a state-of-the-art 3B vision-language model that understands document layout, tables, multi-column text, and mathematical formulas.

---

## Why not just use the official dots.mocr demo?

The [official demo](https://github.com/rednote-hilab/dots.mocr) is a research script — great for testing a single image, not for converting real documents.

| | Official demo | nispa-ocr |
|---|---|---|
| Input | Single image | Multi-page PDF |
| Output | Printed to terminal | Downloadable `.md` file |
| Progress | None | Page-by-page progress bar |
| Interface | Command line | Web UI (works on any device on your network) |
| Concurrent users | 1 | Multiple (vLLM batching) |
| Crash recovery | Lost | Resumes after browser refresh or server restart |
| Integration | Manual | REST API ready |

## Why not just use Microsoft MarkItDown?

[MarkItDown](https://github.com/microsoft/markitdown) is a great tool — fast, zero dependencies, no GPU needed. Use it when your PDFs have selectable text and no complex layout.

nispa-ocr is better when:

| | MarkItDown | nispa-ocr |
|---|---|---|
| **Approach** | Text extraction from PDF structure | Vision model — reads the page as an image |
| **Scanned PDFs** | ❌ Empty output | ✅ Full OCR |
| **Complex layout** | ⚠️ Often loses structure | ✅ Preserved |
| **Multi-column text** | ❌ | ✅ |
| **Mathematical formulas** | ❌ | ✅ |
| **Embedded images** | ❌ Silently ignored | ✅ Extracted to `assets/` |
| **Tables** | ✅ Via pdfplumber | ✅ Via vision model |
| **Speed** | ⚡ Instant | 🐢 Seconds per page (GPU) |
| **Requirements** | Python only | Docker + NVIDIA GPU |

If your PDF has selectable text and no images → use MarkItDown.  
If your PDF is scanned, has complex layout, formulas, or embedded images → use nispa-ocr.

## Why Docker instead of a plain Python script?

A plain script works fine for personal use. Docker makes sense when:

- **You share it with a team** — one `docker compose up` and everyone has the same environment, no "works on my machine"
- **You want isolation** — the model, the API server, and the web UI each run in their own container, don't interfere with your system Python, and can be updated independently
- **You want it always available** — `restart: unless-stopped` means the service survives reboots without manual intervention
- **You expose it on a local network** — colleagues can use it from their browser without installing anything

If you are the only user and just want to convert a PDF once, the official script is enough. nispa-ocr is for when you need it as a persistent, shared service.

---

## Requirements

- Docker + Docker Compose
- NVIDIA GPU with ≥ 16 GB VRAM
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)

## Quick start

```bash
git clone https://github.com/nispa/nispa-ocr.git
cd nispa-ocr

# Optional but recommended: set your GPU index and HF token
cp .env.example .env

# Start (first run downloads ~6 GB of model weights — grab a coffee)
docker compose up -d

# Open the UI
open http://localhost:3000
```

Upload a PDF. Watch it convert page by page. Download the `.md`.

---

## How it works

```
Browser → nginx → FastAPI backend → vLLM (dots.mocr)
                       ↓
               renders each PDF page as PNG
               sends it to the model
               streams Markdown back
               saves job state to disk
```

Each PDF page is rendered at 150 DPI, sent to dots.mocr as a vision input, and converted to Markdown independently. The results are joined into a single file with page separators.

Job state is persisted to disk — if you refresh the browser mid-conversion, or restart the backend, progress is not lost.

---

## Configuration

Copy `.env.example` to `.env`:

| Variable | Default | Description |
|---|---|---|
| `NVI

## Tech Stack
- Linguaggi: Python (5), JavaScript (2), TypeScript (2)
- File totali (scan): 37
- Estensioni principali: (noext)=6, .tsx=6, .py=5, .json=4, .pdf=2, .js=2, .ts=2, .svg=1
- Remote: `git@github.com:nispa/nispa-ocr.git`

## Struttura (top-level)
```
LICENSE
README.md
backend
docker-compose.yml
frontend
models
pk
shared
social-cover.svg
start.bat
start.ps1
```

## Documentazione presente nel repo
- `README.md`

## Progetti correlati
- [[cck]]
- [[llmwiki]]
- [[llmwiki1]]
- [[marker1]]
- [[mcp-unity]]
- [[ocr]]
- [[ocrlocal]]
- [[opensim-cpp]]

## Link
- Repository: git@github.com:nispa/nispa-ocr.git
- Path locale: `/home/pakkio/w/nispa-ocr`
