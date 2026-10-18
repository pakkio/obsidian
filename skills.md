# skills

> Skill per agenti Claude Code / Freebuff.

## Concetti
[[AI Agents]] [[LLM]] [[Python]] [[CLI]]

## Panoramica
Raccolta di skill:
- **verbosity-control** (`/terse`): impedisce ai modelli di ragionamento di sovra-spiegare con *output contract* espliciti: direttiva terse in 3 istruzioni, budget di parole, vincoli di formato (solo codice/diff), controllo del thinking a livello API, output JSON, regole nel loop agentico. Test A/B via OpenRouter (5 prove/braccio): deepseek-v4-flash output 1846→88 token (−95%, costo −93%), claude-sonnet-5 1142→215 (−81%), qualità intatta.
  - Nota: su OpenRouter `thinking: {type: disabled}` per DeepSeek è ignorato; serve `reasoning: {effort:"none", enabled:false}`.
- **llm-wiki** (`/wiki`): wiki personale mantenuta dall'LLM (raw/ → pages/ → schema), invece del RAG puro. Vedi [[llmwiki]].
- `benchmarks/` con `ab_terse_test.py` e `ab_sonnet5.py`.

## Tech Stack
Markdown + Python. Repo: https://github.com/pakkio/skills

## Collegamenti
[[llmwiki]], [[mcp-crawler]], [[mcp-dotenv]], [[mcp-local_llm]]
