# Function-calling notification agent

A small **Gradio** chat demo where an **OpenAI** model answers questions in character using local context (a short bio summary and text extracted from a PDF). When the model uses **function calling** to record lead details or unanswered questions, the app sends **Pushover** mobile notifications so you see activity in real time.

## What it does

- Serves a browser chat UI via **Gradio** `ChatInterface` (OpenAI-style message history: `role` + `content`).
- Builds a **system prompt** from:
  - `me/summary.txt` — narrative background you maintain.
  - `me/Profile.pdf` — extra text (e.g. exported profile) merged into the prompt as context.
- Runs a **tool-use loop**: the model may call tools; the server executes them, appends tool results to the conversation, and calls the model again until it returns a normal assistant message (no more tool calls).

## LLM stack

| Piece | Choice | Role |
|--------|--------|------|
| **API** | [OpenAI Chat Completions](https://platform.openai.com/docs/api-reference/chat) | Hosted inference and native tool calling. |
| **Model** | `gpt-4o-mini` | Fast, cost-efficient multimodal family member; strong enough for grounded Q&A and reliable JSON tool arguments. |
| **Tools** | OpenAI `"function"` tools | Two tools: record contact interest (email + optional name/notes) and record questions the model cannot answer. |
| **Client** | `openai` Python SDK | `openai.chat.completions.create(..., messages=..., tools=tools)`; `finish_reason == "tool_calls"` drives the loop. |

The model is not fine-tuned here: behavior comes from the **system prompt** plus the injected summary and PDF text. Tool definitions are JSON Schema–style objects passed in the `tools` parameter so the model decides when to invoke them.

## Notifications (Pushover)

Tool handlers call a small `push()` helper that POSTs to [Pushover’s messages API](https://pushover.net/api). Successful delivery and API errors are logged to the console. Credentials are read from environment variables (see below).

## Requirements

- Python **3.14+** (see `pyproject.toml`).
- [uv](https://github.com/astral-sh/uv) (or your own way to install deps from `pyproject.toml`).

## Setup

1. Clone the repo and install dependencies:

   ```bash
   uv sync
   ```

2. Copy or create a `.env` file next to `main.py` with:

   - `OPENAI_API_KEY` — required for the chat API.
   - `PUSHOVER_USER` — your Pushover user key.
   - `PUSHOVER_TOKEN` — your Pushover application API token.

3. Add context files:

   - `me/summary.txt` — plain text used in the system prompt.
   - `me/Profile.pdf` — PDF whose text is extracted with **PyPDF** and appended to the prompt.

## Run

```bash
uv run main.py
```

Open the local URL Gradio prints in the terminal (typically `http://127.0.0.1:7860`).

## Project layout

| Path | Purpose |
|------|---------|
| `main.py` | Loads env, defines tools, Pushover helper, PDF/summary ingestion, chat loop, Gradio launch. |
| `me/summary.txt` | Editable bio/context for the persona. |
| `me/Profile.pdf` | Longer profile text source for the prompt. |
| `.env` | Secrets (not committed); loaded from the directory containing `main.py`. |

## Notes

- The persona label and instructions are configured inside `main.py` (system prompt). Swap the summary/PDF files to repurpose the same stack for another site or topic.
- On startup, the app sends a test Pushover message so you can confirm credentials quickly.
