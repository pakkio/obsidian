# fastobsidian

> Explorer ed editor rapido per una vault Obsidian, senza dipendenze.

## Concetti
[[CLI]] [[JavaScript]] [[Python]]

## Panoramica
Uno script Python (solo libreria standard) più un template HTML. Nessuna build, nessuna chiamata di rete. Legge la vault e offre indice, grafo, ricerca, vista note con backlink ed editor markdown. Repo: https://github.com/pakkio/fastobsidian

## Funzioni
- **Indice**: statistiche, tag cloud, note più collegate e modificate di recente, orfane, link non risolti, tabella A–Z ordinabile
- **Grafo**: canvas force-directed con pan, zoom, drag, focus a 1–3 salti, colori per cluster o cartella, filtro per tag, link non risolti come nodi fantasma
- **Ricerca**: albero cartelle e full-text con snippet; click su un tag per filtrare
- **Note**: markdown renderizzato, `[[wikilink]]` (alias e `#heading`) e link `[testo](nota.md)`, TOC cliccabile, link in entrata e in uscita
- **Tema**: interfaccia glass con toggle chiaro/scuro (segue il sistema, la scelta viene ricordata)
- **Vault grandi** (modalità server): indice incrementale che rilegge solo i file cambiati, ricerca lato server
- **Editing** (modalità server): crea, modifica, rinomina, elimina note con editor split e anteprima live

## Uso
```bash
# Live: naviga e modifica su http://127.0.0.1:8765/
python obsidian_explorer.py /percorso/vault --serve --open

# Sola lettura: un unico explorer.html autonomo
python obsidian_explorer.py /percorso/vault -o explorer.html --open
```
Opzioni: `--port N` (default 8765), `-o FILE`, `--open`. Richiede Python 3.9+.

## Comportamento da ricordare
- L'eliminazione è recuperabile: i file vanno in `<vault>/.trash/`
- Il rename aggiorna i `[[wikilink]]` nelle altre note, non i link `[testo](x.md)`
- Il salvataggio è rifiutato se il file è cambiato su disco dopo l'apertura
- Il server ascolta solo su `127.0.0.1`, non ha autenticazione: non esporlo in rete
- Cartelle che iniziano con `.` sono ignorate
- `explorer.html` incorpora il contenuto della vault ed è nel gitignore: non condividerlo per errore

## Struttura (top-level)
```
obsidian_explorer.py      parser della vault, generatore statico e server locale
explorer_template.html    UI (indice, grafo, vista nota, editor)
sample-vault/             note di esempio
README.md
```

## Posizione
`C:\Users\claudio.pacchiega\w\obsidian`
