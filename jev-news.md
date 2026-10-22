# jev-news

> Small CLI scripts for web search via third-party APIs, and a Jev-based news digest

## Concetti
[[LLM]] [[Python]]

## Panoramica
# search

Small CLI scripts for web search via third-party APIs, and a Jev-based news digest
built on top of them.

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
Chip e infrastrutture, Business, Settore), each with its own accent colour and
a note explaining what belongs there.

### How stories get unified

Two headlines about the same event become one card with several sources. Two
rules decide it:

- **token containment** for the same story told twice;
- **shared rare entities** for the case containment misses. The NYT writes
  *"DeepSeek and Huawei Target a Key Source of Nvidia's A.I. Dominance"*, the
  wire writes *"DeepSeek partners with Huawei to develop chip programming
  tools"* - two words overlap and nothing else, but both name the same
  companies.

An entity is a term that is ALLCAPS (`HBM`) or PascalCase (`DeepSeek`) in the
headline, or capitalised inside one and never written in lowercase inside one.
That last part is what stops `'faces'` and `'lawsuit'` - ordinary words that
are merely rare in a few hundred headlines - from merging two unrelated
lawsuits.

A headline reduced to one meaningful word (`AI and education do not mix` keeps
only *education*) would otherwise score 1.00 against anything else mentioning
education, so below three tokens and two shared words the score falls back to
Jaccard, which a short title cannot game.

Run `--folded` to see every merge and its sources before trusting it.

### Italian titles and snippets

### The hero image

A 236px banner from Unsplash, hotlinked free, masked so it dissolves into the
page

## Tech Stack
- Linguaggi: Python (18)
- File totali (scan): 44
- Estensioni principali: .py=18, .json=17, .csv=3, (noext)=2, .md=2, .html=1, .lock=1
- Remote: `git@github.com:pakkio/jev-news.git`

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
translate.py
```

## Documentazione presente nel repo
- `README.md`
- `provs/ai-news.md`

## Progetti correlati
- [[jev-biblio]]
- [[jev-pakkio]]
- [[jev-rag]]
- [[search]]
- [[soup-pakkio]]

## Link
- Repository: git@github.com:pakkio/jev-news.git
- Path locale: `/home/pakkio/w/jev-news`
