---
tags:
  - craftsim
  - osjev
  - decision-models
  - benchmark
  - jev
  - subq
  - operations
aliases:
  - Jev-like benchmark
  - Decision models benchmark
---

# Decision Models Benchmark (Jev-like models via `osJevAskExt`)

← Back to [[CraftSim]] | Related: [[Module - osAI & osJev]], [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[MCP Server & Live Diagnostics]], [[Performance & Benchmarks]]

Measured **2026-10-17** on the live sim (region `PKC`, host `craft`), by running an LSL script through the MCP tool `zengine_run` (`mcp-craftsim`). The script calls `osJevAskExt(state, questionsJson, model, "")` once per ticket and model, scores the answers against hand-written labels, and prints one line per model. Script and ticket set: `tasks/jev_decision_eval.lsl` in the repo (doc copy: `doc/DECISION-MODELS-BENCHMARK.md`).

> [!WARNING] Read the numbers as indicative
> 30 tickets labelled by one person, one pass per model, one network path. A gap of one ticket (3 points) is within noise. Timings are single samples and exclude retry waits. SubQ's cost is computed from its list price, not reported by its API.

## Models tested

| Model id (as passed to `osJevAskExt`) | Resolved version | Route |
|---|---|---|
| `typesafe/jev-1.14` (`~typesafe/jev-latest` resolves to the same build) | `jev-1.14-20261014` | OpenRouter `/api/alpha/decisions` |
| `typesafe/jev-1.13` | `jev-1.13-20260917` | OpenRouter |
| `openai/gpt-6-luna-decisions` | `...-20261006` | OpenRouter |
| `microsoft/microsoft-decision-1` | `...-20261009` | OpenRouter |
| `perplexity/pplx-decider-v1.2` | `...-20261016` | OpenRouter |
| `perplexity/pplx-decider-flash-v1.2` | `...-20261016` | OpenRouter |
| `subq-snap-preview` | `subq-snap-preview` | `api.subq.ai/v1/systemone` (own key, text only) |

`perplexity/pplx-decider-v1.1-27b`, the old default, now fails with `HTTP 404: No endpoints found`. The default became `pplx-decider-v1.2` on 2026-10-17 (commit `47fb914f1c`).

## Method

- Prompt: `"Support ticket: <text>"` with three typed questions in one request: `intent` (choice: `refund`, `replace`, `info`, `other`), `angry` (noul, yes/no probability), `urgency` (score over `low, medium, high, critical`).
- 30 support tickets (English and a few Italian), each with a hand-written label for intent, angry yes/no and urgency level.
- Scoring: intent correct if the chosen id equals the label; angry correct if `noul >= 0.5` matches the label; urgency level = `floor(score * 4)` capped at 3, reported exact and within one level (±1).
- `pplx-decider-*` calls are spaced by 0.4 s and a `429` is retried up to 4 times with a 2 s, 4 s, 6 s, 8 s back-off; the other models run back to back.

## Results (30 tickets)

| Model | Intent | Angry | Urgency exact | Urgency ±1 | Avg time | Cost / call | Cost / 1000 tickets |
|---|---|---|---|---|---|---|---|
| **subq-snap-preview** | **30/30** | **30/30** | **24/30** | 30/30 | 0.27 s | $0.0000112 * | ~$11.20 |
| openai/gpt-6-luna-decisions | 30/30 | 29/30 | 23/30 | 30/30 | 0.35 s | $0.0000417 | $41.70 |
| perplexity/pplx-decider-v1.2 | 29/30 | 30/30 | 22/30 | 30/30 | 0.36 s | $0.0000038 | $3.80 |
| perplexity/pplx-decider-flash-v1.2 | 28/30 | 29/30 | 21/30 | 30/30 | **0.26 s** | **$0.0000019** | **$1.90** |
| typesafe/jev-1.14 | 30/30 | 29/30 | 20/30 | 30/30 | 0.35 s | $0.0000176 | $17.60 |
| typesafe/jev-1.13 | 30/30 | 29/30 | 20/30 | 30/30 | 0.34 s | $0.0000176 | $17.60 |
| microsoft/microsoft-decision-1 | 30/30 | 29/30 | 19/30 | 30/30 | 0.38 s | $0.0000049 | $4.90 |

\* SubQ Snap: $0.05 per 1M input tokens, output free (subq.ai/pricing); 224 input tokens for the test prompt. The API response carries no `cost` field, so this is list price × tokens. The same page advertises 100M free tokens for the first 10 000 users. SubQ is in public preview; the price can change.

Costs for the others are the `usage.cost` the endpoint returned. Token counts differ per model for the same prompt (Jev 419, Luna 417, pplx 189, Microsoft 117, SubQ 224).

## Reading the results

- **Fastest**: `pplx-decider-flash-v1.2` (0.20-0.23 s on repeated runs), SubQ a close second. The rest sit at 0.28-0.38 s; differences between them are inside the noise of a single sample.
- **Cheapest**: flash ($1.90 per 1000), then pplx v1.2 and Microsoft. Luna costs about 22× flash and was not better.
- **Most precise**: SubQ (intent and anger perfect, 80 % exact urgency). Everything is within one level on urgency (±1 = 100 %).
- **Anger misses** are all ticket 9 (a polite but urgent double-charge message labelled "not angry"). Five models score it as angry (0.59-0.77); only SubQ and pplx v1.2 follow the label. The label is debatable; excluding ticket 9, every model has 29/29.
- **Intent misses**: flash on ticket 20 ("the refund arrived, you can close the ticket" → `refund`) and ticket 29 ("can I pay by bank transfer instead of card?" → `other`); pplx v1.2 on ticket 20. Ticket 29's label (`info`) is arguable.
- **Rate limit on `pplx-decider-v1.2`**: `HTTP 429 ... Rate limit exceeded for model 'pplx-decider-v1.2'; retry after 1s`. 12 of 30 calls needed a retry in the 30-ticket run; a 1.2 s gap between calls did not prevent it, so it is not only the script's pacing. All calls succeeded after back-off. `DecisionClient` itself does **not** retry yet: a script calling the default model can get an `ERROR: HTTP 429` now and then.
- **Repeatability**: with a unique ticket number in every prompt (to rule out provider-side caching) the answers moved by 0.01-0.06 on anger and urgency between rounds. There is no cache in `DecisionClient`.
- **Luna's 1.66 s** in the first run was a one-off; it measured 0.2-0.4 s afterwards.

## Recommendation

| Goal | Model |
|---|---|
| Best accuracy | `subq-snap-preview` (text only, preview, own key) |
| Best value | `perplexity/pplx-decider-flash-v1.2` |
| Near-top accuracy, low cost | `perplexity/pplx-decider-v1.2` (needs 429 handling) |
| Avoid for now | `openai/gpt-6-luna-decisions` (22× flash, no gain), `microsoft/microsoft-decision-1` (weakest urgency) |

## Reproducing

```text
# the script lists its models in MODELS; zengine_run waits at most 60 s,
# so run a few models at a time (3 models x 30 tickets ~ 30-40 s)
mcp-craftsim.zengine_run scriptFile=/root/pakkio/craftsim/tasks/jev_decision_eval.lsl waitMilliseconds=60000
```

Console equivalent: `zengine run <file> --wait 60000` in the `craftsim` console. A `zengine_run` of a script that calls the network needs the keys in `Secrets.ini` (`OpenRouterKey`, `TypeSafeApiKey`, `SubqApiKey`).

## Pitfall found while building it

A variable declared **without an initialiser inside a loop** (`integer ok;` in a `for` body) was observed not to be reset on the next iteration in ZEngine: counters kept accumulating across models. Declare with `= 0`. Strings and floats that were initialised were fine.
