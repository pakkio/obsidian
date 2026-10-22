# pakagent

> Workflow sicuro a più programmi per sviluppo assistito da LLM con pak/pakdiff.

## Concetti
[[AI Agents]] [[LLM]] [[CLI]] [[Python]]

## Panoramica
Automatizza il copia-incolla con l'LLM:
- `prepare.py` – impacchetta i file in `/tmp/archive.txt`
- `pakmod.py` – chiede all'LLM le modifiche (`/tmp/answer`, `/tmp/fix`)
- `pakdiff.py` – revisione in terminale a 3 finestre
- `pakapply.py` – applica le modifiche con conferma
- `pakrestore.py`, `pakview.py`

Sicurezza: sanitizzazione input, protezione API key, validazione codice; logging e integrazione git.

## Tech Stack
Python (Poetry, PyInstaller specs). Repo: https://github.com/pakkio/pakagent

## Collegamenti
[[pak]], [[agent]]
