# FMM

> Fast Multipole Method: simulazione gravitazionale 2D con 1M di particelle, ASCII nel terminale.

## Concetti
[[C++]] [[CLI]]

## Panoramica
Implementazione C++17 del **Fast Multipole Method** O(N): quadtree con espansioni multipolari (serie di Taylor, numeri complessi), criterio MAC per bilanciare precisione/velocità, softening per le singolarità. Visualizzazione real-time in ASCII con statistiche live. Zero dipendenze, `-O3 -march=native -pthread`.

## Struttura
`include/{particle,quadtree,fmm}.h`, `src/{main,fmm,quadtree}.cpp`, `CMakeLists.txt`, `Makefile`, `ALGORITHM.md`, `GUIDE.md`, `render_video.py`, `run_demo.sh`.

## Tech Stack
C++17, CMake. Repo: https://github.com/pakkio/FMM
