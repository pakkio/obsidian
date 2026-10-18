# logprob

> Analisi di autocertezza per frase sui modelli OpenAI.

## Concetti
[[LLM]] [[Python]] [[Web]]

## Panoramica
Mostra con colori (verde→rosso) quanto il modello è sicuro di ogni token/parola/frase, usando `logprobs` dell'API OpenAI e la formula `exp(logprob)*100`. Serve a individuare possibili allucinazioni. Supporta GPT-4.1, 4o, 4o-mini, 4-turbo, 3.5-turbo. Due versioni: app Gradio e pagina HTML standalone.

## Tech Stack
Python (Gradio), HTML. Repo: https://github.com/pakkio/logprob
