# jev-biblio

> > Harness Jev per verificare se le affermazioni di un testo sono sostenute dalle sue fonti.

## Concetti
[[LLM]] [[Python]]

## Panoramica
# 📚 Skepsis

> Harness Jev per verificare se le affermazioni di un testo sono sostenute dalle sue fonti.

Dashboard web locale che controlla se un articolo è davvero sostenuto dalle sue fonti:

- **verifica i link** della bibliografia e **scarica il testo delle fonti**;
- **classifica l'autorevolezza di ogni fonte** con Jev in 4 fasce — 🟢 autorevole, 🟡 semi-autorevole, 🟠 opinione, 🔴 inaffidabile (o inesistente) — indipendentemente dal voto dell'articolo;
- **estrae le affermazioni** dell'articolo con un LLM, separando fatti, conclusioni dell'autore, opinioni ed esperienze personali;
- **verifica ogni affermazione sul testo delle fonti con [Jev](https://typesafe.ai)** (TypeSafe *System One*) e ne stima la plausibilità secondo la conoscenza generale;
- **passa a un LLM che ragiona solo i casi in cui Jev è incerto**;
- **assegna da 1 a 5 stelle con regole esplicite** sui verdetti, mostrando il motivo, e **scrive una giustificazione in italiano**;
- **mostra tempi e costi** di ogni passaggio, con il totale in centesimi.

Una verifica completa costa in media **0,1-0,2 ¢** e richiede **10-40 secondi**, a seconda della lunghezza dell'articolo. Ultima esecuzione (settembre 2026): 100% sul set di test, 95% sul set di sviluppo (vedi [Valutazione](#valutazione) per i numeri completi, per la regressione del 27 settembre e per cosa significa ormai "test" — non è più un set indipendente).

## Avvio rapido

Serve [uv](https://docs.astral.sh/uv/). Le dipendenze (`typesafe-sdk`, `python-dotenv`, `pypdf`, `curl-cffi`) sono dichiarate in testa a `main.py` e uv le installa da solo.

```bash
uv run main.py                      # apre http://127.0.0.1:8000/
uv run main.py --port 9000 --no-browser
```

Skepsis ascolta solo su localhost. Per esporla intenzionalmente a una rete bisogna aggiungere `--host … --allow-remote` e metterla dietro un proxy con autenticazione e rate limiting.

Incolla il testo dell'articolo e la bibliografia con gli URL, poi premi **Avvia analisi Jev e controllo link**. Nella bibliografia i riferimenti possono essere numerati (`[1] Titolo https://…`) e citati nel testo con `[1]`. Se l'articolo non usa i numeri, è il LLM ad associare ogni affermazione alle fonti pertinenti.

Se hai copiato una pagina intera (articolo e bibliografia già insieme, come capita incollando una pagina web), puoi lasciare vuoto il campo della bibliografia: Jev prova a separare le due parti da solo (vedi il passaggio 1 in [Come funziona](#come-funziona)). Se non trova un elenco di fonti separato, il testo resta tutto nell'articolo e la dashboard chiede comunque una bibliografia.

## Chiavi API

Il programma legge le chiavi dall'ambiente oppure da `./.env` o `../.env`:

```dotenv
TYPESAFE_API_KEY=...      # obbligatoria: verifiche e voto di Jev
OPENROUTER_API_KEY=...    # consigliata: estrazione delle affermazioni, casi incerti, giustificazione
TAVILY_API_KEY=...        # facoltativa: ricerca esterna di fonti autorevoli
```

Senza `OPENROUTER_API_KEY` la dashboard funziona in modalità ridotta: divide l'articolo per frasi invece di estrarre le affermazioni, non passa a un LLM i casi incerti e mostra solo il riepilogo di Jev.

## Come funziona

| Passaggio | Chi lo fa | Cosa succede |
|---|---|---|
| 1. Separazione (solo se manca la bibliografia) | Jev, una chiamata con una domanda sì/no per paragrafo (riserva: euristica sui titoli di sezione) | Se l'utente incolla una pagina intera (articolo e bibliografia insieme) senza compilare il campo bibliografia, Jev classifica ogni paragrafo come "articolo" o "elenco di fonti" e taglia al primo paragrafo di bibliografia confermato da una maggioranza dei successivi. Senza questo, o se Jev non decide nulla, il testo resta tutto nell'articolo. |
| 2. Link e fonti | Python | Scarica in parallelo fino a 15 URL pubblici, con un tetto complessivo di 8 s; legge HTML, testo e PDF entro limiti di dimensione. Usa `curl_cffi` per replicare l'handshake TLS di Chrome (alcuni siti dietro Cloudflare bloccano il fingerprint TLS di

## Tech Stack
- Linguaggi: Python (8)
- File totali (scan): 29
- Estensioni principali: .json=10, .py=8, .txt=7, .jsonl=2, (noext)=1, .md=1
- Remote: `git@github.com:pakkio/jev-biblio.git`

## Struttura (top-level)
```
README.md
claims.py
colab_main.py
discovery.py
eval
llm.py
main.py
samples
tests
```

## Documentazione presente nel repo
- `README.md`

## Progetti correlati
- [[jev-news]]
- [[jev-pakkio]]
- [[jev-rag]]
- [[search]]
- [[soup-pakkio]]

## Link
- Repository: git@github.com:pakkio/jev-biblio.git
- Path locale: `/home/pakkio/w/jev-biblio`
