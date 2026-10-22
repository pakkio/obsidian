# googlelivetranslate

> Traduttore vocale in tempo reale: ascolta il microfono, traduce in inglese e legge la traduzione ad alta voce.

## Concetti
[[Python]]

## Panoramica
# Google Live Translate

Traduttore vocale in tempo reale: ascolta il microfono, traduce in inglese e legge la traduzione ad alta voce.

## Come funziona

1. Il microfono viene campionato continuamente a 16 kHz
2. Un VAD (Voice Activity Detection) basato su RMS rileva quando stai parlando
3. Il segmento audio viene inviato a Gemini via OpenRouter per trascrizione + traduzione in inglese in un solo passo
4. La traduzione viene letta ad alta voce con Gemini TTS (voce Zephyr)
5. Tutta la sessione viene salvata su file WAV

Tre worker paralleli gestiscono la traduzione in modo che frasi consecutive non si blocchino a vicenda.

## Requisiti

```
pip install sounddevice numpy
```

Serve anche `mpv` per la riproduzione audio:

```bash
sudo apt install mpv
```

Su WSL2 serve PulseAudio attivo (disponibile tramite WSLg).

## Configurazione

Crea un file `.env` nella directory del progetto:

```
OPENROUTER_KEY=sk-or-...
```

Ottieni una chiave su [openrouter.ai](https://openrouter.ai).

## Utilizzo

### Traduzione live (microfono → voce)

```bash
python live_translate.py
```

Parla in qualsiasi lingua: la traduzione in inglese apparirà a schermo e verrà letta ad alta voce. Premi `Ctrl+C` per fermare — la registrazione viene salvata automaticamente.

### Traduzione testo

```bash
echo "Ciao mondo" | python translate.py English
python translate.py Italian < file.txt
```

Lingua target opzionale come primo argomento (default: `Italian`).

### Text-to-speech

```bash
echo "Hello world" | python tts.py Zephyr output.wav
```

Argomenti opzionali: voce (default: `Zephyr`) e file di output (default: `output.wav`).

## Modelli

| Componente | Modello |
|---|---|
| STT + Traduzione | `google/gemini-3.5-flash` |
| TTS | `google/gemini-3.1-flash-tts-preview` |

## Parametri principali (`live_translate.py`)

| Parametro | Default | Descrizione |
|---|---|---|
| `SILENCE_THRESHOLD` | 300 | Soglia RMS per rilevare la voce |
| `SILENCE_CHUNKS` | 15 | Chunk di silenzio (~1.5s) per chiudere un'utterance |
| `MIN_SPEECH_CHUNKS` | 5 | Durata minima (~0.5s) per inviare un segmento |
| `TRANSLATE_WORKERS` | 3 | Thread paralleli per la traduzione |
| `TTS_VOICE` | Zephyr | Voce Gemini TTS |
| `SPEAK` | True | Abilita/disabilita la riproduzione audio |

## Tech Stack
- Linguaggi: Python (3)
- File totali (scan): 11
- Estensioni principali: .wav=4, .py=3, .md=2, .mp3=1, (noext)=1

## Struttura (top-level)
```
ARTICOLO.md
README.md
live_translate.py
output.mp3
output.wav
recording_20260613_164244.wav
recording_20260613_164436.wav
recording_20260613_165035.wav
translate.py
tts.py
```

## Documentazione presente nel repo
- `ARTICOLO.md`
- `README.md`

## Progetti correlati
_nessuno_

## Link
- Path locale: `/home/pakkio/w/googlelivetranslate`
