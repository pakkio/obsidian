# gemmafunction

> A Python project demonstrating function calling capabilities with Google's Gemma/Gemini models using Poetry for dependency management.

## Concetti
[[LLM]] [[Python]]

## Panoramica
# Gemma Function Calling Demo

A Python project demonstrating function calling capabilities with Google's Gemma/Gemini models using Poetry for dependency management.

**Quick Start**: See [QUICK_REFERENCE.md](QUICK_REFERENCE.md) for setup and usage commands.

## Features

This demo showcases three example functions that an AI model can call:

1. **get_weather**: Get weather information for a location
2. **calculate**: Perform mathematical operations (add, subtract, multiply, divide)
3. **search_database**: Search a mock database

## Prerequisites

- Python 3.9 or higher
- Poetry (for dependency management)
- Google AI API key

## Installation

### 1. Install Poetry (if not already installed)

```bash
# On Windows (PowerShell)
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | py -

# On Linux/macOS
curl -sSL https://install.python-poetry.org | python3 -
```

### 2. Install dependencies

```bash
poetry install
```

### 3. Set up your API key

1. Get your Google AI API key from: https://makersuite.google.com/app/apikey
2. Copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```
3. Edit `.env` and add your API key:
   ```
   GOOGLE_API_KEY=your_actual_api_key_here
   ```

### 4. Configure models (optional)

1. Copy the config example:
   ```bash
   cp config.ini.example config.ini
   ```
2. Edit `config.ini` to customize models and parameters
3. See [CONFIG.md](CONFIG.md) for detailed configuration options

## Usage

### Interactive Chat (Recommended)

Start an interactive chat session with function calling:

```bash
# Using Google Gemini API
python -m gemma_demo.chat

# Using OpenRouter (Gemma 2 9B and other models) ⭐ NEW
python -m gemma_demo.chat_openrouter

# With poetry
poetry run python -m gemma_demo.chat
poetry run python -m gemma_demo.chat_openrouter
```

This provides a real-time chat interface where you can:
- Ask questions and get responses
- Have the model automatically call functions as needed
- See function calls and results in real-time
- Type 'exit' to quit, 'clear' to reset, 'help' for examples

**New: OpenRouter Support** 🚀
- Access Gemini and other models through OpenRouter API
- **Important**: Free tier Gemma models do NOT support function calling on OpenRouter
- Uses Gemini 2.5 Flash which supports function calling (paid tier, very affordable)
- See [OPENROUTER_SETUP.md](OPENROUTER_SETUP.md) for setup instructions

### Run Batch Demos

Run the pre-defined demo examples:

```bash
# Function calling demo
python -m gemma_demo.main

# Gemma model demo (no function calling)
python -m gemma_demo.gemma_model_demo

# With poetry
poetry run python -m gemma_demo.main
```

## How It Works

The demo:

1. Configures the Google Generative AI client with your API key
2. Defines example functions with their schemas (automatically extracted from Python functions)
3. Sends prompts to the model that trigger function calls
4. Executes the requested functions locally
5. Returns the results to the model for a final response

## Example Prompts

Try these prompts in the interactive chat:

- "What's the weather like in Paris?" → Calls `get_weather`
- "Can you multiply 25 by 4?" → Calls `calculate`
- "Search the database for information about Python programming" → Calls `search_database`
- "What time is it?" → Calls `get_current_time`
- "Remind me to buy groceries tomorrow" → Calls `create_reminder`
- "Compare the weather in Tokyo and London" → Calls `get_weather` twice

## Project Structure

```
.
├── gemma_demo/
│   ├── __init__.py
│   ├── main.py                # Batch demo with predefined prompts
│   ├── chat.py                # Interactive chat session
│   └── gemma_model_demo.py    # Demo using actual Gemma models
├── pyproject.toml             # Poetry configuration
├── requirements.txt           # Pip requirements
├── .env.example               # Example environment variables
├── .gitignore
├── README.md
└── DEMO_RESULTS.md            # Demo test results
```

## Adding Your Ow

## Tech Stack
- Linguaggi: Python (18)
- File totali (scan): 53
- Estensioni principali: .md=21, .py=18, .txt=3, (noext)=3, .example=2, .exe=1, .toml=1, .lock=1
- Remote: `git@github.com:pakkio/gemmafunction.git`

## Struttura (top-level)
```
100mb
BUGFIX.md
CHANGES_SUMMARY.md
CLAUDE.MD
CONFIG.md
CONFIG_EXTERNALIZATION.md
DEMO_RESULTS.md
ES_UNDERSTANDING_FIX.md
EVERYTHING_SETUP.md
FILE_SEARCH_INTEGRATION.md
FIXES_APPLIED.md
FUNCTION_SCHEMA_EXPLAINED.md
LOCAL_SETUP.md
MODEL_COMPARISON.md
OPENROUTER_FIX.md
OPENROUTER_INTEGRATION.md
OPENROUTER_SETUP.md
PROJECT_SUMMARY.md
QUICKSTART.md
QUICK_REFERENCE.md
README.md
WEB_FUNCTIONS.md
add_system_message.patch
config.ini.example
es-cli.zip
es.exe
gemma_demo
list_models.py
list_openrouter_models.py
poetry.lock
pyproject.toml
requirements-huggingface.txt
requirements-local.txt
requirements.txt
test_chat.py
test_chat_function.py
test_config.py
test_es_path.py
test_inline_chat.py
test_openrouter_function.py
```

## Documentazione presente nel repo
- `BUGFIX.md`
- `CHANGES_SUMMARY.md`
- `CLAUDE.MD`
- `CONFIG.md`
- `CONFIG_EXTERNALIZATION.md`
- `DEMO_RESULTS.md`
- `ES_UNDERSTANDING_FIX.md`
- `EVERYTHING_SETUP.md`
- `FILE_SEARCH_INTEGRATION.md`
- `FIXES_APPLIED.md`
- `FUNCTION_SCHEMA_EXPLAINED.md`
- `LOCAL_SETUP.md`
- `MODEL_COMPARISON.md`
- `OPENROUTER_FIX.md`
- `OPENROUTER_INTEGRATION.md`
- `OPENROUTER_SETUP.md`
- `PROJECT_SUMMARY.md`
- `QUICKSTART.md`
- `QUICK_REFERENCE.md`
- `README.md`
- `WEB_FUNCTIONS.md`

## Manifest
### pyproject.toml
```
[tool.poetry]
name = "gemma-demo"
version = "0.1.0"
description = "Demo project for Gemma function calling capabilities"
authors = ["pakkio"]
readme = "README.md"
packages = [{include = "gemma_demo"}]

[tool.poetry.dependencies]
python = "^3.9"
google-generativeai = "^0.3.0"
python-dotenv = "^1.0.0"
requests = "^2.31.0"
openai = "^1.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```
### requirements.txt
```
google-generativeai>=0.3.0
python-dotenv>=1.0.0
requests>=2.31.0
openai>=1.0.0
```

## Progetti correlati
- [[agent]]
- [[chatgroup]]
- [[llmwiki]]
- [[llmwiki1]]
- [[local_llm]]
- [[mragent]]
- [[rlm]]

## Link
- Repository: git@github.com:pakkio/gemmafunction.git
- Path locale: `/home/pakkio/w/gemmafunction`
