# pk

> pk

## Concetti
[[JavaScript]] [[Web]]

## Panoramica
Partita a scacchi contro un LLM via OpenRouter, da terminale (`chess-ai.js`) o dal browser (Express + `public/index.html`); `chess.js` valida le mosse. README.md e RELEASE-NOTES.md aggiunti il 2026-10-18. Progetto con 7 file.

## Tech Stack
- Linguaggi: JavaScript (3)
- File totali (scan): 7
- Estensioni principali: .js=3, .json=2, (noext)=1, .html=1

## Struttura (top-level)
```
chess-ai.js
node_modules
openrouter-call.js
package-lock.json
package.json
public
server.js
```

## Manifest
### package.json
```
{
  "name": "pk",
  "version": "1.0.0",
  "description": "",
  "main": "chess-ai.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "type": "module",
  "dependencies": {
    "@openrouter/sdk": "^0.8.0",
    "chess.js": "^1.4.0",
    "dotenv": "^17.3.1",
    "express": "^5.2.1"
  }
}
```

## Progetti correlati
- [[browser]]
- [[nexus]]

## Link
- Path locale: `/home/pakkio/w/pk`
