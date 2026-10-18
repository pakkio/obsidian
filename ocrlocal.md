# ocrlocal

> Pipeline locale PDF → Markdown con descrizione automatica delle figure via LLM vision.

## Concetti
[[OCR]] [[Python]]

## Panoramica
# marker

Pipeline locale PDF → Markdown con descrizione automatica delle figure via LLM vision.

**Stack:** [Marker](https://github.com/VikParuchuri/marker) + [Surya OCR](https://github.com/VikParuchuri/surya) + GPU (CUDA) + [OpenRouter](https://openrouter.ai) per vision.

## Funzionalità

- Routing automatico: pdfplumber per PDF digitali semplici (sub-secondo/pagina), Marker+OCR per PDF complessi o scansionati
- Converte PDF in Markdown strutturato (testo, tabelle, heading, formule LaTeX)
- Estrae figure/fotografie come immagini JPEG
- Descrive ogni figura in italiano tramite un modello vision (Gemini 2.5 Flash via OpenRouter o minicpm-v via Ollama)
- Ricostruisce i simboli matematici Unicode persi dall'OCR (ℝⁿ, λᵢ, ωₙ) tramite LLM testuale locale
- Modalità completamente offline: nessun dato esce dalla macchina (Ollama per vision + testo)

## Requisiti

- Python 3.12+
- [uv](https://github.com/astral-sh/uv)
- GPU NVIDIA con CUDA 12.x e **almeno 6 GB VRAM** (facoltativo ma consigliato — ~4x più veloce; su CPU funziona ma lento)
- [Ollama](https://ollama.com) con `minicpm-v` per descrizione figure offline — oppure API key OpenRouter per modalità cloud

## Installazione

```bash
git clone ...
cd marker
uv sync
```

Al primo avvio `uv sync` scarica PyTorch+CUDA e `marker-pdf`. I modelli Surya (~1-2 GB) vengono scaricati da HuggingFace alla prima conversione.

## Configurazione

```bash
cp .env.example .env
# modifica .env con la tua chiave
```

`.env` — modalità offline (consigliata per privacy):
```
OLLAMA_MODEL=minicpm-v
```

`.env` — modalità cloud (OpenRouter):
```
OPENROUTER_API_KEY=sk-or-...
VISION_MODEL=google/gemini-2.5-flash
```

Se `OLLAMA_MODEL` è definito ha la precedenza su OpenRouter. `VISION_MODEL` può essere qualsiasi modello vision disponibile su OpenRouter.

## Utilizzo

### 1. Converti PDF → Markdown

```bash
uv run convert.py <file.pdf> [cartella_output] [lingue]
```

```bash
uv run convert.py documento.pdf output it
uv run convert.py tesi.pdf output it,en
```

Output in `output/`:
- `documento.md` — testo convertito
- `documento/` — immagini estratte (`_page_N_Figure_M.jpeg`)

### 2. Descrivi le figure con LLM vision

```bash
uv run describe_figures.py output/documento.md
```

Produce `output/documento_described.md`: ogni `![]()` viene sostituito con alt-text descrittivo generato dal modello vision + blocco citazione.

### 3. Correggi i simboli matematici

```bash
uv run fix_symbols.py output/documento.md
```

Produce `output/documento_fixed.md`: ogni `■` (simbolo Unicode perso dall'OCR) viene ricostruito tramite LLM testuale in base al contesto matematico. Richiede Ollama con un modello testuale (es. `gemma4` o `gemma3:4b`).

### Workflow completo

**Con figure e simboli matematici:**
```bash
uv run convert.py documento.pdf output it
uv run describe_figures.py output/documento.md          # → documento_described.md
uv run fix_symbols.py output/documento_described.md     # → documento_described_fixed.md
```

**Solo simboli matematici (nessuna figura):**
```bash
uv run convert.py documento.pdf output it
uv run fix_symbols.py output/documento.md               # → documento_fixed.md
```

`fix_symbols.py` esce subito se non trova `■`. `describe_figures.py` salta le immagini non trovate.

## Esempio reale: tesi di ingegneria scansionata

Scenario: tesi di 120 pagine in italiano, scansionata (no testo selezionabile), con grafici matplotlib, tabelle di dati e formule con ℝⁿ, λᵢ, ωₙ.

```bash
# 1. Converti — Marker rileva la scansione e avvia OCR su GPU (~35 min)
uv run convert.py tesi_scansionata.pdf output it
# → output/tesi_scansionata.md  (testo + tabelle in Markdown)
# → output/tesi_scansionata/    (23 figure estratte come JPEG)

# 2. Descrivi le 23 figure in italiano (~6 min con minicpm-v offline)
uv run describe_figures.py output/tesi_scansionata.md
# → output/tesi_scansionata_described.md
#   Ogni ![]() diventa:
#   ![Grafico della risposta al gradino con ζ=0.5 e ωₙ=10 rad/s...](fig.jpeg)
#   > **Figura:**

## Tech Stack
- Linguaggi: Python (12)
- File totali (scan): 108
- Estensioni principali: (noext)=60, .py=12, .md=6, .jpeg=5, .jpg=4, .pdf=4, .bat=3, .png=2
- Remote: `git@github.com:pakkio/ocrlocal.git`

## Struttura (top-level)
```
README.md
__pycache__
convert.py
crea_pdf.py
crea_pdf_complesso.py
crea_pdf_grafici.py
crea_pdf_reale.py
crea_solo_testo.py
describe_figures.py
fix_symbols.py
img_architettura.jpg
img_chart.jpg
img_grafico.jpg
img_grafico.png
img_paesaggio.jpg
img_tabella.png
output
pdf_complesso_italiano.pdf
pyproject.toml
server.py
solo_testo_italiano.pdf
test_grafici.pdf
test_italiano.pdf
uv.lock
```

## Documentazione presente nel repo
- `README.md`
- `output/pdf_complesso_italiano.md`
- `output/pdf_complesso_italiano_described.md`
- `output/pdf_complesso_italiano_described_fixed.md`
- `output/solo_testo_italiano.md`
- `output/test_italiano.md`

## Manifest
### pyproject.toml
```
[project]
name = "marker"
version = "0.1.0"
description = "Add your description here"
requires-python = ">=3.12"
dependencies = [
    "marker-pdf>=1.10.2",
    "pdfplumber>=0.11.0",
    "matplotlib>=3.10.9",
    "reportlab>=4.5.0",
    "torch>=2.6.0",
    "torchvision>=0.21.0",
    "fastapi>=0.136.1",
    "uvicorn[standard]>=0.46.0",
    "python-multipart>=0.0.27",
]

[tool.uv.sources]
torch = [
    { index = "pytorch-cu126", marker = "sys_platform == 'win32'" },
]
torchvision = [
    { index = "pytorch-cu126", marker = "sys_platform == 'win32'" },
]

[[tool.uv.index]]
name = "pytorch-cu126"
url = "https://download.pytorch.org/whl/cu126"
explicit = true
```

## Progetti correlati
- [[marker1]]
- [[nispa-ocr]]
- [[ocr]]

## Link
- Repository: git@github.com:pakkio/ocrlocal.git
- Path locale: `/home/pakkio/w/ocrlocal`
