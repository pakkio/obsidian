# gemma3-270M-finetuning-ner

> Framework di fine-tuning e confronto Gemma3 (270M/1B) vs CodeS-1B.

## Concetti
[[LLM]] [[Python]]

## Panoramica
Training su GPU consumer (4-8 GB VRAM) con quantizzazione 4-bit e LoRA. Risultati:
- **NER**: Gemma3-270M F1 98,3% (7 min), pari a spaCy fine-tuned (98,4%), contro 53,3% di spaCy generico
- **Text-to-SQL** (da italiano): Codes-1B 62,5% exact match
- **Intent classification**: Codes-1B migliorato 100%
- Hashtag generation, Gemma3-1B-it (32k contesto, 140+ lingue)

Cartelle: `configs`, `data`, `models`, `scripts`, `tests`, `quick_start.py`, `run_training.sh`.

## Tech Stack
Python (Poetry), PEFT/LoRA. Repo: https://github.com/pakkio/gemma3-270M-finetuning-ner

## Collegamenti
[[gemmafunction]], [[jev-pakkio]]
