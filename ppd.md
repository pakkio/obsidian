# ppd

> NPC AI Brain per OpenSim: NPC intelligenti pilotati dal motore Nexus/Eldoria.

## Concetti
[[OpenSim]] [[LLM]] [[AI Agents]] [[Python]]

## Panoramica
Documentazione e contenuti per NPC OpenSim guidati da LLM: conversazioni naturali, personalità dalla lore, profiling psicologico dei giocatori, comandi SL per animazioni/emote, memoria persistente. Flusso: avatar si avvicina → sensor → API `/sense` → saluto LLM → `/chat` → `/leave`. Server [[nexus]] (Python, OpenRouter), script LSL `NPC_AI_Brain_Complete.lsl` in un prim.

Contenuti: set di location/NPC a tema *Piccolo Principe* (asteroide B612, pianeti algoritmi, contabile, emozioni, geografo, influencer, lampionaio, programmatore, ricordi, nodo centrale), `ARCHITETTURA_NARRATIVA_COMPLETA.md`, `NARRATIVE_FRAMEWORK.txt`, guide su deploy, generazione dialoghi e import BVH.

## Tech Stack
Python, LSL, OpenSim.ini (HttpRequestModule). Repo: https://github.com/pakkio/ppd

## Collegamenti
[[nexus]], [[nexus2]], [[nexus-narrative-weaver]], [[lsl_simulator]]
