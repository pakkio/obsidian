# ChessChallenge

> Esercizio TryCatchUs: posizionare pezzi di scacchi senza attacchi, ottimizzato di 690.000×.

## Concetti
[[CLI]]

## Panoramica
Solver Scala che trova tutte le disposizioni di pezzi su scacchiera M×N senza minacce reciproche.
- 6×6: da 24+ ore a **124 ms**; 7×7: da 40+ min a **1,1 s** (382.990 soluzioni canoniche)
- Tappe: funzionale (20 s) → parallelo (8,7 s) → eliminazione oggetti (1,9 s) → simmetrie gruppo diedrale D₄ (353 ms) → bitboard (124 ms)
- Verifica: 23.752 soluzioni = 2.969 canoniche × 8 simmetrie
- Documenti: `article.pdf/.tex`, `PERFORMANCE_ANALYSIS.md`, `enhancement-ai.md` (piano SAT/AI)

## Tech Stack
Scala (sbt), con esperimenti Python (CNF/SAT). Repo: https://github.com/pakkio/ChessChallenge (dal 2015)
