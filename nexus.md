# nexus

> Sistema completo per NPCs intelligenti in OpenSim utilizzando il motore AI Nexus/Eldoria.

## Concetti
[[AI Agents]] [[JavaScript]] [[Python]] [[Virtual Worlds]]

## Panoramica
# **NPC AI Brain per OpenSim - Documentazione Completa**

Sistema completo per NPCs intelligenti in OpenSim utilizzando il motore AI Nexus/Eldoria.

## **Panoramica Sistema**

Il **NPC AI Brain** trasforma gli NPCs statici di OpenSim in personaggi intelligenti capaci di:

- 🤖 **Conversazioni naturali** alimentate da LLM
- 🎭 **Personalità uniche** basate su lore dettagliata
- 📊 **Profiling psicologico** dei giocatori
- 🎮 **Generazione comandi SL** per animazioni/emotes
- 💾 **Memoria persistente** delle conversazioni
- 🌍 **Integrazione mondo di gioco** completa

### **Flusso Funzionale**
```
Avatar si avvicina → Sensor rileva → API /sense → LLM genera saluto → 
Chat continua → API /chat → Risposte AI → Avatar si allontana → API /leave
```

## **Quick Start**

### **1. Setup Server Nexus**

```bash
# Clone del repository
git clone <nexus-repo>
cd nexus

# Installazione dipendenze
poetry install

# Configurazione ambiente
cp .env.example .env
# Modifica .env con la tua OpenRouter API key

# Avvio server
python app.py
```

### **2. Configurazione OpenSim**

Aggiungi in `OpenSim.ini`:

```ini
[Network]
    OutboundDisallowForUserScriptsExcept = 172.25.249.74:5000|127.0.0.1:5000|localhost:5000

[HttpRequestModule]
    HttpRequestsEnabled = true
    HttpTimeout = 45000
    HttpRequestsAllowLocalhost = true
    HttpRequestsAllowedDomains = localhost,127.0.0.1,172.25.249.74
    HttpRequestsMaxResponseSize = 8192
```

### **3. Deploy Script NPC**

Copia lo script `NPC_AI_Brain_Complete.lsl` in un prim OpenSim e modifica:

```lsl
string API_BASE = "http://172.25.249.74:5000";  // IP del tuo server
string NPC_NAME = "Garin";                      // Nome NPC desiderato
string AREA_NAME = "Village";                   // Area NPC
```

## **Architettura**

```
┌─────────────────┐    HTTP     ┌──────────────────┐    OpenRouter    ┌─────────────┐
│   OpenSim LSL   │ ◄─────────► │   Nexus Server   │ ◄──────────────► │     LLM     │
│   NPC Script    │             │   (Flask API)    │                  │  (Gemini/   │
└─────────────────┘             └──────────────────┘                  │  Llama/etc) │
                                          │                            └─────────────┘
                                          ▼
                                ┌──────────────────┐
                                │    Database      │
                                │  (File/MySQL)    │
                                │ - NPCs           │
                                │ - Conversazioni  │
                                │ - Profili Player │
                                └──────────────────┘
```

## **NPCs Disponibili**

| Nome | Area | Ruolo | Specialità |
|------|------|-------|------------|
| **Garin** | Village | Fabbro della Memoria | Forgiatura, strumenti |
| **Mara** | Village | Erborista della Memoria | Erbe, guarigione |
| **Jorin** | Tavern | Custode di Sogni Perduti | Storie, leggende |
| **Lyra** | Sanctum of Whispers | Tessitrice Suprema | Saggezza antica, guida |
| **Elira** | Forest | Guardiana del Nodo Naturale | Natura, sussurri |
| **Boros** | Mountain | Primo Monaco dell'Equilibrio | Meditazione, equilibrio |
| **Theron** | City | Alto Giudice | Giustizia, politica |
| **Cassian** | City | Burocrate Opportunista | Commercio, influenze |
| **Irenna** | City | Marionettista della Resistenza | Arte, memoria |
| **Syra** | Ancient Ruins | Eco Residuo | Antichità, rovine |
| **Erasmus** | Liminal Void | Sussurro Cosciente | Filosofia, oblio |
| **Meridia** | Nexus of Paths | Tessitrice del Destino | Destino, scelte |

## **API Endpoints**

| Endpoint | Metodo | Descrizione | Payload |
|----------|--------|-------------|---------|
| `/health` | GET | Health check server | - |
| `/sense` | POST | Notifica arrivo avatar | `{name, npcname, area}` |
| `/api/chat` | POST | Chat con NPC | `{player_name, message, npc_name, area}` |
| `/leave` | POST | Notifica partenza avatar | `{name}` |
| `/api/game/npcs` | GET | Lista NPCs disponibili |

## Tech Stack
- Linguaggi: Python (75), LSL (OpenSim) (5), JavaScript (1)
- File totali (scan): 200
- Estensioni principali: .py=75, .json=40, .txt=38, .md=15, .log=5, .lsl=5, (noext)=4, .xml=4
- Remote: `git@github.com:pakkio/nexus.git`

## Struttura (top-level)
```
ANTEFATTO.txt
ARCHITETTURA_NARRATIVA_COMPLETA.md
BVH_IMPORT_TROUBLESHOOTING_GUIDE.md
CLAUDE.md
DEPLOYMENT.md
DIALOG_EXAMPLE.md
GENERATION_GUIDE.md
GUIDA_RAPIDA_SISTEMA.md
Location.asteroide_b612.txt
Location.nodo_centrale.txt
Location.pianeta_algoritmi.txt
Location.pianeta_contabile.txt
Location.pianeta_emozioni.txt
Location.pianeta_geografo.txt
Location.pianeta_influencer.txt
Location.pianeta_lampionaio.txt
Location.pianeta_programmatore.txt
Location.pianeta_ricordi.txt
NARRATIVE_FRAMEWORK.txt
NPC.asteroide_b612.piccolo_principe.txt
NPC.asteroide_b612.rosa_di_luce.txt
NPC.nodo_centrale.presenza_nodo.txt
NPC.pianeta_algoritmi.re_algoritmi.txt
NPC.pianeta_contabile.contabile_dati.txt
NPC.pianeta_emozioni.curatrice_emozioni.txt
NPC.pianeta_geografo.geografo_digitale.txt
NPC.pianeta_influencer.influencer_solitario.txt
NPC.pianeta_lampionaio.lampionaio_sistema.txt
NPC.pianeta_programmatore.programmatore_errante.txt
NPC.pianeta_ricordi.custode_ricordi.txt
NPC.testarea.teleporter.txt
NPC_FACIAL_EXPRESSIONS_GUIDE.md
NPC_PREFIX.asteroide_b612.piccoloprincipe.txt
NPC_PREFIX.asteroide_b612.rosadiluce.txt
NPC_PREFIX.nodo_centrale.presenzadelnodo.txt
NPC_PREFIX.pianeta_algoritmi.redeglialgoritmi.txt
NPC_PREFIX.pianeta_contabile.contabiledidati.txt
NPC_PREFIX.pianeta_emozioni.curatriceemozioni.txt
NPC_PREFIX.pianeta_geografo.geografodigitale.txt
NPC_PREFIX.pianeta_influencer.influencersolitario.txt
```

## Documentazione presente nel repo
- `ARCHITETTURA_NARRATIVA_COMPLETA.md`
- `BVH_IMPORT_TROUBLESHOOTING_GUIDE.md`
- `CLAUDE.md`
- `DEPLOYMENT.md`
- `DIALOG_EXAMPLE.md`
- `GENERATION_GUIDE.md`
- `GUIDA_RAPIDA_SISTEMA.md`
- `NPC_FACIAL_EXPRESSIONS_GUIDE.md`
- `NPC_SPAWN_DEBUG_GUIDE.md`
- `OSSL_NPC_FUNCTIONS_REFERENCE.md`
- `README-NPC-AI-BRAIN.md`
- `README-SL-INTEGRATION.md`
- `README.md`
- `README_NPC_PREFIXES.md`
- `TESTING-SERVER.md`

## Manifest
### pyproject.toml
```
[tool.poetry]
name = "nexus"
version = "0.1.0"
description = ""
authors = ["Claudio Pacchiega <claudio.pacchiega@gmail.com>"]
package-mode = false
packages = [
    { include = "*.py" }
]



[tool.poetry.dependencies]
python = "^3.12"
requests = "^2.32.3"
python-dotenv = "^1.0.0"
mysql-connector-python = "^9.3.0"
flask = "^3.1.1"
flask-cors = "^6.0.1"
gradio = "^5.32.0"
tree-sitter = "^0.24.0"
tree-sitter-languages = "^1.10.2"
tree-sitter-python = "^0.23.6"
pytest = "^7.4.0"
pytest-mock = "^3.10.0"
black = "^23.7.0"
mcp = "^2.1.1"
uvicorn = "^0.52.4"
starlette = "^1.6.0"
asgiref = "^3.12.1"

# Dipendenze di sviluppo
[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
black = "^23.7.0"
python = "^3.12"
dotenv = "^0.9.9"
mysql-connector-python = "^9.3.0"
requests = "^2.32.3"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

## Progetti correlati
- [[agent]]
- [[browser]]
- [[craftsim]]
- [[craftstorm]]
- [[libopenmetaverse]]
- [[libremetaverse]]
- [[mragent]]
- [[mragent-backup]]
- [[opensim-baseline]]
- [[opensim-cpp]]
- [[phoenix-firestorm]]
- [[pk]]

## Link
- Repository: git@github.com:pakkio/nexus.git
- Path locale: `/home/pakkio/w/nexus`
