# search

> Small CLI scripts for web search via third-party APIs, and a Jev-based news digest

## Concetti
[[CLI]] [[Python]]

## Panoramica
# search

Small CLI scripts for web search via third-party APIs, and a Jev-based news digest
built on top of them.

## Weekly digest on any topic

`just run "<topic>"` builds the same weekly page as the LGBT+ digest, for any
topic. A model lays out 4-5 thematic areas and the Serper queries for that topic;
then the script collects the last 7 days, merges outlets reporting the same event,
rates and translates everything into Italian, and writes a self-contained HTML
page (photos inlined) to `topics/<topic>.html`.

```bash
just run "auto elettriche"            # italiano (default)
just run "auto elettriche" en         # inglese
just run "auto elettriche" --max-credits 20
uv run python topic_news.py "auto elettriche"
```

The page is always in Italian — story titles and snippets, thread titles and
synopses, and the area labels — while proper nouns stay as written and Jev's
criteria stay in English (it judges meaning better there). The topic plan (areas,
queries, topic sentence) is cached in `topics/<topic>.areas.json`, so a rerun and
`--reuse` cost nothing. Needs `SERPER_API_KEY`, `TYPESAFE_API_KEY` and
`OPENROUTER_API_KEY` (or `OPENCODEGO_API_KEY`), from the environment or `../.env`.

Flags: `--max-credits N` (Serper cap, default 8), `--reuse` (rebuild from the
saved stories), `--no-summaries`, `--no-images`, `--model M`, `--lang it|en`. A
bare `it` or `en` after the topic is shorthand for `--lang`: `just run "auto
elettriche" en`.
Output and caches live in the gitignored `topics/` directory.

## Scripts

### `main.py` — Serper + Jina search

Searches with the [Serper](https://serper.dev) API and fetches readable
page content for each result via [Jina Reader](https://jina.ai/reader/).

```bash
export SERPER_API_KEY="your-serper-api-key"
python3 main.py "your search query"
```

| Flag | Effect |
| --- | --- |
| `--news` | use Serper's News endpoint instead of web search (adds `Via: <source> · <date>`) |
| `--days N` | restrict to the last `N` days — `1` → `qdr:d`, `7` → `qdr:w`, `30` → `qdr:m` |
| `--num N` | number of results (default 10) |

```bash
python3 main.py --news --days 7 "artificial intelligence"
```

### `collect_news.py` — balanced collection

A single generic sweep skews hard towards model launches: one area can swallow
a third of the page while another gets two cards. This walks a per-area query
list instead, stopping early for areas that already have enough coverage and
spending extra queries only on the ones that are still short.

```bash
python3 collect_news.py --target 20        # 20 stories per area
python3 collect_news.py --target 25 --max-credits 60
```

Writes `ai-news.json` (the default of `format_news.py`; `news.py` topics use `provs/<slug>.json`), and prints a per-area count with `OK`/`LOW` so an
unbalanced run is visible immediately.

### `format_news.py` — digest renderer

Reads a stories JSON file and writes three views: a styled HTML page, a
markdown digest, and a colourised terminal digest.

```bash
python3 format_news.py --lang it --intro "..." [--main 4] [--more 6] [--folded]
```

| Flag | Effect |
| --- | --- |
| `--lang it\|en` | output language, including relative dates (`2 ore fa`) |
| `--intro` | lead paragraph under the title |
| `--main N` | lead cards shown per area (default 4) |
| `--more N` | secondary cards shown per area (default 6) |
| `--sim F` | same-event merge threshold (default 0.62) |
| `--folded` | list every merge, with its folded titles - check the clustering |
| `--translate` | translate titles and snippets into Italian (see below) |
| `--model M` | model for translation, summaries and threads (default `$LLM_MODEL`, else `openrouter/free`; `opencodego:<model>` uses OpenCode Go, key `OPENCODEGO_API_KEY`) |
| `--hero N` | banner image: `earth` (default), `circuit`, `code`, `robot`, `laptop`, `off`, or any URL |
| `--images M` | story images: `all` (default), `main` (lead cards only), `off` |

Grouped into five thematic areas (Politica e sicurezza, Modelli e ricerca,
Chip e infr

## Tech Stack
- Linguaggi: Python (19)
- File totali (scan): 74
- Estensioni principali: .json=44, .py=19, .md=4, .html=3, (noext)=2, .csv=2
- Remote: `git@github.com:pakkio/search.git`

## Struttura (top-level)
```
README.md
__pycache__
collect_news.py
costs.py
decide.py
demo.py
diff_runs.py
embed_html.py
format_news.py
images.py
jev.py
justfile
labels.py
main.py
meter-history.json
meter.py
news.py
provs
rate.py
sim_tokens.py
summarize.py
topic_news.py
topics
translate.py
```

## Documentazione presente nel repo
- `README.md`
- `topics/ai-it.md`
- `topics/ai.md`
- `topics/gay.md`

## Progetti correlati
- [[assegna]]
- [[jev-biblio]]
- [[jev-news]]
- [[jev-pakkio]]
- [[jev-rag]]
- [[rlm]]
- [[soup-pakkio]]
- [[spirale]]
- [[subs]]
- [[tbi]]

## Link
- Repository: git@github.com:pakkio/search.git
- Path locale: `/home/pakkio/w/search`
