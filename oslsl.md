# oslsl

> Server MCP in TypeScript per LSL (Linden Scripting Language) e OSSL (OpenSimulator) con motore di analisi semantica, sicurezza e similarità vettoriale a 128 dimensioni.

## Concetti
[[MCP]] [[TypeScript]] [[OpenSim]]

## Panoramica
Server MCP specializzato (repo: https://github.com/pakkio/oslsl) per lo sviluppo su Second Life e OpenSimulator:
- **Documentazione completa**: lookup funzioni LSL, eventi LSL, estensioni OSSL, ricerca esempi e best practice.
- **Analisi semantica enterprise**: validazione con confidence score, metriche (memoria e complessità ciclomatica), scansione sicurezza/permessi OSSL, riconoscimento di pattern, estrazione codice pulito dai wiki e similarità tra funzioni tramite embedding a $128$ dimensioni.

## Architettura

```mermaid
flowchart LR
  Agent["Client MCP (Claude / IDE)"] -->|"stdio JSON-RPC"| Srv["LSL MCP Server (TypeScript)"]
  Srv --> Docs["Lookup LSL / OSSL<br>Funzioni · Eventi · Esempi"]
  Srv --> Sem["Motore Analisi Semantica<br>Validazione · Complessità · Sicurezza"]
  Srv --> Sim["Ricerca Similarità<br>(embedding 128-dim)"]
```

## Tech Stack
- TypeScript / Node.js (`npm run build`, test Jest), script wrapper WSL/Windows (`start-server.bat`, `run-server.sh`).
- Repository: https://github.com/pakkio/oslsl (pubblico, verificato 2026-11-27)

## Collegamenti
[[lsl_simulator]], [[CraftSim]] (estensioni OSSL)
