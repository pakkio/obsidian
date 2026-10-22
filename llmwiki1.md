# llmwiki1

> llmwiki1

## Concetti
[[LLM]] [[TypeScript]]

## Panoramica
Riscrittura TypeScript del wiki LLM su OpenRouter: per ora solo scaffolding (`src/storage.ts` con test, design e piano in `docs/plans/`); `wiki.ts` e i comandi non esistono ancora. Vedi [[llmwiki]] per la versione funzionante. README.md e RELEASE-NOTES.md aggiunti il 2026-10-18. Progetto con 11 file.

## Tech Stack
- Linguaggi: TypeScript (2)
- File totali (scan): 11
- Estensioni principali: .json=4, (noext)=3, .md=2, .ts=2

## Struttura (top-level)
```
docs
node_modules
package-lock.json
package.json
pages
src
tsconfig.json
```

## Documentazione presente nel repo
- `docs/plans/2026-04-25-llm-wiki-design.md`
- `docs/plans/2026-04-25-llm-wiki-implementation.md`

## Manifest
### package.json
```
{
  "name": "llmwiki",
  "version": "1.0.0",
  "description": "LLM-powered wiki CLI",
  "scripts": {
    "wiki": "tsx wiki.ts"
  },
  "license": "ISC",
  "type": "module",
  "dependencies": {
    "commander": "^14.0.3",
    "dotenv": "^17.4.2"
  },
  "devDependencies": {
    "@types/node": "^25.6.0",
    "tsx": "^4.21.0",
    "typescript": "^6.0.3"
  }
}
```

## Progetti correlati
- [[agent]]
- [[cck]]
- [[chatgroup]]
- [[gemmafunction]]
- [[llmwiki]]
- [[local_llm]]
- [[mcp-unity]]
- [[mragent]]
- [[nispa-ocr]]
- [[opensim-cpp]]
- [[rlm]]

## Link
- Path locale: `/home/pakkio/w/llmwiki1`
