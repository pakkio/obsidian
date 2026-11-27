# gemma3-270M-finetuning-ner

> Framework di fine-tuning QLoRA (4-bit) e confronto tra Gemma3 (270M & 1B) e CodeS-1B su NER, Text-to-SQL italiano e Intent Classification su GPU consumer (4–8 GB VRAM).

## Concetti
[[LLM]] [[Python]]

## Panoramica
Framework completo (repo: https://github.com/pakkio/gemma3-270M-finetuning-ner) per addestrare e confrontare **`google/gemma-3-270m`**, **`google/gemma-3-1b-it`** (32k context window, 140+ lingue) e **CodeS-1B** su più task NLP usando quantizzazione 4-bit e adattatori LoRA su GPU da 4–8 GB VRAM.

### Risultati misurati
| Task | Modello | Risultato | Note |
|---|---|---:|---|
| **NER** | Gemma3-270M (LoRA) | $F_1 = 98{,}3\%$ | 7 min training; pari a spaCy fine-tuned ($98{,}4\%$), vs $53{,}3\%$ spaCy generico |
| **Text-to-SQL (IT)** | CodeS-1B | $62{,}5\%$ EM | Exact match da italiano |
| **Intent classification** | CodeS-1B (fine-tuned) | $100\%$ | Miglioramento completo sul benchmark locale |

## Pipeline QLoRA

```mermaid
flowchart LR
  Data["Dataset (NER / SQL / Intent)"] --> Quant["Modello base 4-bit NF4<br>(Gemma3 270M/1B, CodeS-1B)"]
  Quant --> LoRA["Training LoRA (PEFT)<br>GPU 4-8 GB VRAM"]
  LoRA --> Eval["Valutazione F1 / Exact Match"]
  Eval --> Cmp["Confronto vs spaCy & Baseline"]
```

## Formulazione matematica
LoRA congela i pesi quantizzati $W_0 \in \mathbb{R}^{d \times k}$ e apprende un aggiornamento a basso rango $r \ll \min(d, k)$:

$$W' = W_0 + \Delta W = W_0 + \frac{\alpha}{r}\,B A, \qquad B \in \mathbb{R}^{d \times r},\; A \in \mathbb{R}^{r \times k}$$

La qualità su NER è valutata tramite punteggio $F_1$ armonico tra precisione $P$ e richiamo $R$:

$$F_1 = \frac{2\,P\,R}{P + R}$$

## Struttura
`configs/`, `data/`, `docs/`, `results/`, `scripts/`, `tests/`, `quick_start.py`, `run_training.sh`, `pyproject.toml`.

## Tech Stack
- Python (Poetry), PyTorch, Transformers, PEFT/LoRA, bitsandbytes 4-bit.
- Repository: https://github.com/pakkio/gemma3-270M-finetuning-ner (pubblico, verificato 2026-11-27)

## Collegamenti
[[gemmafunction]], [[jev-pakkio]], [[soup-pakkio]]
