# youtube-news

> Rassegna stampa settimanale in italiano dai canali YouTube AI/tech.

## Concetti
[[LLM]] [[Python]]

## Panoramica
Monitora 15+ canali (AI Explained, Matthew Berman, Two Minute Papers, e canali italiani come Simone Rizzo, Raffaele Gaito...), analizza i video con LLM (Gemini 2.5 Flash, GPT-5 Mini via OpenRouter) e genera riassunti per video più una rassegna unificata. Caching su file, retry e rate limiting. Script: `run.py`, `rss_analyzer.py`, `generate_italian_weekly.py`, `llm_wrapper.py`, tool per cercare channel ID.

## Tech Stack
Python (Poetry), OpenRouter, YouTube Data API. Repo: https://github.com/pakkio/youtube-news

## Collegamenti
[[jev-news]], [[ainews]]
