# tbi

> Interpreter BASIC minimale per uso didattico, ispirato al Dr. Dobb's Journal (1976).

## Concetti
[[CLI]] [[Python]]

## Panoramica
# TinyBASIC

Interpreter BASIC minimale per uso didattico, ispirato al Dr. Dobb's Journal (1976).

## Caratteristiche

- **Variabili**: variabili numeriche (A-Z) e stringa (A$-Z$)
- **Controllo flusso**: IF...THEN, FOR...TO...[STEP], GOTO, GOSUB/RETURN
- **I/O**: PRINT, INPUT, LET
- **Funzioni matematiche**: SIN, COS, TAN, ATN, SQR, ABS, EXP, LN, LOG, PI, INT
- **Funzioni stringa**: LEFT$, RIGHT$, MID$, CHR$, STR$, LEN, ASC, VAL, INSTR
- **Plotting Braille**: CANVAS, AXES, XTICK, YTICK, PLOTXY, PSET, RENDER, PLOT$
- **Operatori**: AND, OR, ~ (circa uguale)

## Utilizzo

```bash
# Modalità interattiva
python3 tinybasic.py

# Eseguire un file .bas
python3 tinybasic.py examples/sin_braille.bas
```

## Comandi Principali

| Comando | Descrizione |
|---------|-------------|
| `LET X=10` | Assegna valore a variabile |
| `PRINT "Hello"` | Stampa a schermo |
| `INPUT X` | Richiede input da tastiera |
| `GOTO 100` | Salta alla riga specificata |
| `GOSUB 500` | Chiamata a subroutine (con RETURN) |
| `FOR I=1 TO 10` | Ciclo numerico |
| `IF X>5 THEN GOTO 200` | Condizione |
| `REM commento` | Commento |
| `END` | Termina programma |

## Plotting Braille (Comandi Dichiarativi)

TinyBASIC include un sistema di plotting Braille ad alta risoluzione basato su
**framebuffer a punti**. Si dichiara la tela, l'intervallo dei dati, i tick, e
si disegna con PLOTXY — RENDER compone automaticamente la figura completa con
assi e label.

| Comando | Descrizione |
|---------|-------------|
| `CANVAS W,H` | Alloca una griglia W×H pixel |
| `AXES X0,X1,Y0,Y1` | Imposta l'intervallo dati e disegna gli assi |
| `XTICK x [,l$]` | Registra un tick sull'asse X in posizione x |
| `YTICK y [,l$]` | Registra un tick sull'asse Y in posizione y |
| `PLOTXY x,y` | Disegna un punto alle coordinate dati (x,y) |
| `PSET x,y` | Disegna un punto alle coordinate pixel (x,y) |
| `RENDER` | Stampa la figura completa con label, righello e legenda |

```basic
10 REM Y=SIN(X) -- plot braille dichiarativo
20 CANVAS 160,64
30 AXES 0,360,-1,1
40 XTICK 0 : XTICK 90 : XTICK 180 : XTICK 270 : XTICK 360
50 YTICK -1 : YTICK 0 : YTICK 1
60 FOR G=0 TO 360 STEP 0.5
70 PLOTXY G, SIN(G)
80 NEXT G
90 RENDER
```

Documentazione completa: `docs/PLOTTING_MANUAL.md`

## Esempi

Tutti gli esempi in `examples/` sono stati migrati ai nuovi comandi dichiarativi.

### FizzBuzz
```basic
10 LET I=1
20 IF I>20 THEN GOTO 200
30 LET M=I-(I/15)*15
40 IF M=0 THEN GOTO 100
50 LET M=I-(I/3)*3
60 IF M=0 THEN GOTO 110
70 LET M=I-(I/5)*5
80 IF M=0 THEN GOTO 120
90 PRINT I
95 GOTO 130
100 PRINT "FIZZBUZZ"
105 GOTO 130
110 PRINT "FIZZ"
115 GOTO 130
120 PRINT "BUZZ"
130 LET I=I+1
140 GOTO 20
200 END
```

### Sin² (Braille con assi etichettati)
```basic
10 CANVAS 160,64
20 AXES 0,360,0,1
30 XTICK 0 : XTICK 90 : XTICK 180 : XTICK 270 : XTICK 360
40 YTICK 0 : YTICK 0.5 : YTICK 1
50 FOR G=0 TO 360 STEP 0.5
60 PLOTXY G, SIN(G)*SIN(G)
70 NEXT G
80 RENDER
```

### Due curve (EXP e EXP(-X) sovrapposte)
```basic
10 CANVAS 160,64
20 AXES -3,3,0,20
30 XTICK -3 : XTICK 0 : XTICK 3
40 YTICK 0 : YTICK 10 : YTICK 20
50 FOR X=-3 TO 3 STEP 0.05
60 V=EXP(X) : IF V<=20 THEN PLOTXY X,V
70 W=EXP(-X) : IF W<=20 THEN PLOTXY X,W
80 NEXT X
90 RENDER
```

## File inclusi in `examples/`

| File | Funzione |
|------|----------|
| `sin_braille.bas` | SIN(X) 0..360° |
| `cos_braille.bas` | COS(X) 0..360° |
| `sinsq_braille.bas`, `sinsq_canvas.bas`, `sinsq_axes.bas` | SIN²(X) |
| `tan_braille.bas` | TAN(X) clampata a ±5 |
| `sincos_braille.bas` | SIN(X)COS(X) |
| `gauss_braille.bas` | e^(−x²) gaussiana |
| `exp_hires.bas` | EXP(X) |
| `ln_hires.bas` | LN(X) |
| `parabola.bas`, `parabola_hires.bas` | X² |
| `exp_pair_axes.bas` | EXP(X) & EXP(-X) |
| `exp_prod_axes.bas` | EXP(X)×EXP(-X) (=1) |
| `lnexp_axes.bas` | LN(EXP(X)) (=X) |
| `plot_1overx.bas` | 1/X |
| `plot_lib.bas`, `plot_lib_braille.bas`, `plot_lib_lines.bas` | 1/X e 1/X² |
| `fizzbuzz.bas` | FizzBuzz classico |
| `for.bas` | Cicli FOR |
| `gosub.bas` | Subroutine |
| `mathfuncs.bas`

## Tech Stack
- Linguaggi: BASIC (32), Python (1)
- File totali (scan): 37
- Estensioni principali: .bas=32, .md=2, .json=2, .py=1

## Struttura (top-level)
```
README.md
__pycache__
debug.bas
debug2.bas
docs
examples
lib
test_and.bas
test_if.bas
test_or.bas
tinybasic.py
```

## Documentazione presente nel repo
- `README.md`
- `docs/PLOTTING_MANUAL.md`

## Progetti correlati
- [[assegna]]
- [[rlm]]
- [[search]]
- [[spirale]]
- [[subs]]

## Link
- Path locale: `/home/pakkio/w/tbi`
