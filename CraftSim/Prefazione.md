---
tags:
  - craftsim
  - preface
  - overview
aliases:
  - Prefazione
  - Preface
---

# Prefazione

← [[CraftSim]] | Poi: [[ZEngine]], [[ZEXT Language Extensions]], [[Added Functions Summary]], [[Known and Unresolved Problems]]

## Da dove si parte

OpenSimulator è un server open source per mondi 3D persistenti. Quello che rende vivo un mondo sono gli script: piccoli programmi in LSL, il Linden Scripting Language, attaccati agli oggetti. In una regione piena di contenuti ce ne sono migliaia, e devono rispondere insieme a fisica, utenti e rete.

Il whitepaper di ZEngine (`zengine_whitepaper.tex`, scritto ad agosto 2026) indica due colli di bottiglia storici. Il primo è l'interprete: i motori storici XEngine e YEngine eseguono istruzioni intermedie su una macchina virtuale software, con un dispatch continuo e allocazioni (*boxing*) per ogni valore generico. Il secondo è la fame di thread: una chiamata bloccante come `llSleep` teneva occupato un thread del sistema operativo, e i pool di worker si saturavano.

CraftSim è il fork che ha provato a togliere entrambi i problemi alla radice. Gira su .NET 10 e sostituisce l'interprete con **ZEngine**, un compilatore che traduce LSL in CIL con `System.Reflection.Emit` e lascia il codice macchina al JIT. I tipi LSL diventano tipi nativi (`integer` → `Int32`, `vector` → `Vector3`, `key` → `UUID`), e le attese si risolvono cedendo il thread con un flag invece di lanciare eccezioni (il whitepaper misura il controllo del flag in circa 6 ns).

## Che cosa è diventato

Dal whitepaper a oggi il progetto è cresciuto in direzioni che il testo di agosto non poteva prevedere. Le principali:

- **Un linguaggio più comodo sopra LSL (ZEXT).** Ternario, `??`, pipeline, slice e indici, mappe `=>`, f-string e stringhe raw, `switch`, cicli strutturati e soprattutto il legame asincrono `->`, che permette di scrivere in modo lineare codice che aspetta una risposta di rete. Tutto viene riportato a LSL semplice, e `zengine desugar` lo mostra.
- **Servizi per gli script.** Archivi di righe su notecard con ricerca vettoriale (`osDb*`), code di messaggi (`osQueue*`), cache con scadenza (`osCache*`), alberi JSON, client HTTP, OCR, grafici, modelli di machine learning e embedding on-sim (`osML*`), chat e immagini con LLM (`osAI*`).
- **Decisioni tipizzate (`osJev*`).** Domande a scelta, sì/no e punteggio, con risposte validate. Oggi lo stesso strato parla con più modelli (Jev, Microsoft, GPT-6 Luna, Perplexity, SubQ), e il confronto misurato è in [[Decision Models Benchmark]].
- **Deduplicazione su due piani.** Gli asset uguali occupano una sola copia nel database, gli oggetti identici condividono un modello, gli script identici condividono il codice compilato. Il whitepaper descrive la sola copia dei contenuti: il resto è in [[Asset Deduplication & Database]].
- **Pilotaggio da fuori.** Un server MCP con 76 strumenti, comandi di console equivalenti e un login a chiave pubblica via SSH, per far lavorare insieme il simulatore e un assistente come Claude Code.
- **Contorno di produzione.** Inventario AISv3, voce WebRTC, server-side baking, pathfinding, fisica ubODE ottimizzata, backup e controlli all'avvio.

## Che cosa si può dire sulle prestazioni

Il whitepaper confronta, con 1.000 script e 100 NPC, la versione ufficiale su Mono, .NET 8 con Server GC e .NET 10 con Workstation GC. Le cifre, copiate dalla sua tabella:

| | Ufficiale (Mono) | .NET 8, Server GC | .NET 10, Workstation GC |
|---|---|---|---|
| Caricamento + 100 NPC | 66 s | 66 s | **44 s** |
| Eventi di script al secondo | 333 | 485 | **496** |
| Memoria di picco (RSS) | **208,6 MB** | 1.671,7 MB | 465,9 MB |

Due precisazioni, perché il testo si può leggere male. Il «−72%» del sommario è calcolato rispetto a .NET 8 con Server GC (1.671,7 → 465,9 MB), non rispetto alla versione ufficiale: contro questa, il picco di memoria è più alto. E sono misure di agosto su una macchina, non ripetute in questa revisione. Le altre misure che vale la pena tenere sotto mano sono in [[Performance & Benchmarks]].

## Come è fatta questa documentazione

Le note di questo archivio sono state rilette contro il codice. Ogni nota porta, quando serve, il commit con cui è stata controllata, e dove un'affermazione non si poteva provare dal codice lo dice (*non verificato*). Quattro regole valgono per tutte:

1. Il codice batte la documentazione. Dove `CLAUDE.md`, `README.md` o il whitepaper dicono altro, la nota riporta il codice e segnala la differenza.
2. I numeri hanno una data e una fonte. Quelli ricopiati da altre fonti sono indicati come tali.
3. Un problema aperto va scritto come aperto. [[Known and Unresolved Problems]] raccoglie quelli ancora in piedi, quelli che sembravano tali e quelli mai misurati.
4. Un pezzo di documentazione non è una prova che il pezzo di codice funzioni: per alcune parti, come la voce con un visore vero o le query PostgreSQL, i test non sono mai stati eseguiti.

Il whitepaper in LaTeX resta una fonte preziosa per l'impianto e le motivazioni, ma è anche una fotografia di agosto: le sezioni su ZEXT, OsDb, OsCache, OsQ, osAI e osML sono state superate da modifiche successive, e la parte sulla deduplicazione descrive solo il primo passo.

## Come leggerla

| Se vuoi… | Parti da |
|---|---|
| capire il progetto in dieci minuti | questa pagina e [[CraftSim]] |
| scrivere uno script | [[ZEXT Language Extensions]], [[OSSL Extension Examples - Complete Coverage]], [[End-to-End Multi-Module LSL Architectures]] |
| sapere che cosa c'è di nuovo, una riga per funzione | [[Added Functions Summary]] |
| usare modelli di IA e decisioni | [[Module - osAI & osJev]], [[Decision Models Benchmark]] |
| gestire un server | [[Build, Configuration & Self-Hosting]], [[MCP Server & Live Diagnostics]], [[Asset Deduplication & Database]] |
| sapere che cosa non funziona o non è provato | [[Known and Unresolved Problems]] |

*Ultimo controllo: 17 ottobre 2026, simulatore al commit `47fb914f1c`, craftstorm al commit `bba1a8b8f6`.*
