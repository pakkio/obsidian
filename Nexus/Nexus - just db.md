---
tags: [nexus, just, database]
---

# Nexus - just db e just db-kill

> Esplorare un database SQLite dal browser, senza toccarlo.

Ricette del `justfile` di [[nexus]]. Richiedono `sqlite_web` e `tmux`.

## `just db [path] [args...]`
Avvia [sqlite_web](https://github.com/coleifer/sqlite-web) in **sola lettura** su una sessione tmux chiamata `sqlite_web`.

| | |
|---|---|
| Default | `database/nexus.db` (l'archivio di Nexus) |
| Porta | `8081`, in ascolto su tutte le interfacce (`0.0.0.0`) |
| Modo | read-only (`-r`): niente scritture accidentali |
| Sessione | tmux `sqlite_web`; se già attiva viene **riavviata** |
| Argomenti extra | passati a `sqlite_web` così come sono |

Dopo l'avvio la ricetta interroga `localhost:8081` per circa 10 secondi; stampa `sqlite_web up on :8081 (<path>, tmux session sqlite_web)` oppure `sqlite_web did not come up` (exit 1).

```bash
just db                          # database/nexus.db
just db .opencode/opencode.db    # un altro database SQLite
```

Poi apri `http://localhost:8081/`. Le tabelle sono una per tipo di documento (vedi [[Nexus - Lore]] per ciò che contengono gli NPC).

## `just db-kill`
Ferma il server: termina la sessione tmux `sqlite_web`. Stampa `sqlite_web stopped` oppure `sqlite_web not running`.

```bash
just db-kill
```

## Attenzione
Con `0.0.0.0` il browser è raggiungibile da chiunque sia sulla stessa rete: spegnilo con `just db-kill` quando hai finito.
