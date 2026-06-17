# F.R.I.D.A.Y. — Stark Voice Agent

> *"Fully Responsive Intelligent Digital Assistant for You"*

An advanced, Tony Stark-inspired real-time AI voice assistant built using LiveKit Agents and FastMCP. The project is split into two cooperating components:

| Component | Description |
|-----------|-------------|
| **MCP Server** (`uv run friday`) | A [FastMCP](https://github.com/jlowin/fastmcp) server exposing system tools, web search, and real-time feeds over SSE. |
| **Voice Agent** (`uv run friday_voice`) | A [LiveKit Agents](https://github.com/livekit/agents) voice pipeline that transcribes voice, reasons via LLM, and streams speech using high-performance providers. |

---

## How it works

```
 Microphone (User)
       │
       ▼
 [STT] Sarvam Saaras v3 (Indian-English optimized)
       │
       ▼
 [LLM] Groq Llama 3.3 / Google Gemini ◄──(SSE)──► [MCP Server] (FastMCP)
       │                                            ├── News & Finance Feeds
       ▼                                            ├── Web Search & Scraping
 [TTS] Cartesia / OpenAI / ElevenLabs               └── OS & System Diagnostics
       │
       ▼
 Speaker / LiveKit Room (Output)
```

The LiveKit voice agent communicates with the FastMCP server over a Server-Sent Events (SSE) connection (default: `http://127.0.0.1:8000/sse`). When the user asks a question that requires external information or system actions, the LLM calls the appropriate tool from the MCP server, receives the result, and synthesizes a natural voice response.

---

## Project structure

```
friday-stark-voice-agent/
├── server.py           # Starts the FastMCP server (SSE on port 8000)
├── agent_friday.py     # Starts the LiveKit Voice Agent
├── pyproject.toml      # Project dependencies and packaging settings
├── .env.example        # Template for API keys and configuration
└── friday/             # FastMCP server package
    ├── config.py       # Configuration & environment variables loading
    ├── prompts/        # MCP system prompts & template guidelines
    ├── resources/      # Static resources exposed to the agent client
    └── tools/          # Exposed tool definitions
        ├── web.py      # news, finance, web search, & dashboard controls
        ├── system.py   # system diagnostic tools
        └── utils.py    # formatting & utility functions
```

---

## Quick start

### 1. Prerequisites

- Python ≥ 3.11
- [`uv`](https://github.com/astral-sh/uv) — `pip install uv` or `curl -Lsf https://astral.sh/uv/install.sh | sh`
- A [LiveKit Cloud](https://cloud.livekit.io) project (free tier works)

### 2. Clone & install

```bash
git clone https://github.com/BhagathPranav/friday-stark-voice-agent.git
cd friday-stark-voice-agent
uv sync          # creates .venv and installs all dependencies
```

### 3. Set up environment

```bash
cp .env.example .env
# Open .env and fill in your API keys (see the section below)
```

### 4. Run — two terminals

**Terminal 1 — MCP server** (must start first)

```bash
uv run friday
```

Starts the FastMCP server on `http://127.0.0.1:8000/sse`. The voice agent connects here to fetch its tools.

**Terminal 2 — Voice agent**

```bash
uv run friday_voice
```

Starts the LiveKit voice agent in **dev mode** — it joins a LiveKit room and begins listening. Open the [LiveKit Agents Playground](https://agents-playground.livekit.io) and connect to your room to talk to FRIDAY.

---

## `uv run friday` vs `uv run friday_voice`

| Command | Entry point | What it does |
|---------|------------|--------------|
| `uv run friday` | `server.py → main()` | Launches the **FastMCP server** over SSE transport on port 8000. This is the "brain backend" — it registers all tools, prompts, and resources that the LLM can call. |
| `uv run friday_voice` | `agent_friday.py → dev()` | Launches the **LiveKit voice agent**. It builds the STT / LLM / TTS pipeline, connects to your LiveKit room, and wires up the MCP server as a tool source. The `dev()` wrapper auto-injects the `dev` CLI flag so you don't have to type it manually. |

> Both processes must run **simultaneously**. The voice agent calls the MCP server in real time whenever it needs a tool (e.g. fetching news).

---

## Environment variables

Copy `.env.example` to `.env` and set the following keys:

| Variable | Required | Source / Description |
|:---|:---:|:---|
| `LIVEKIT_URL` | ✅ | [LiveKit Cloud Console](https://cloud.livekit.io) (Project Connection URL) |
| `LIVEKIT_API_KEY` | ✅ | LiveKit API Key |
| `LIVEKIT_API_SECRET` | ✅ | LiveKit API Secret |
| `GROQ_API_KEY` | ✅ | [Groq Developer Console](https://console.groq.com) (Default LLM provider) |
| `CARTESIA_API_KEY` | ✅ | [Cartesia Playberry Console](https://cartesia.ai) (Default TTS provider) |
| `SARVAM_API_KEY` | ✅ | [Sarvam AI Dashboard](https://dashboard.sarvam.ai) (Default STT provider) |
| `OPENAI_API_KEY` | optional | [OpenAI Platform](https://platform.openai.com/api-keys) |
| `ELEVEN_API_KEY` | optional | [ElevenLabs Dashboard](https://elevenlabs.io) (For optional ElevenLabs TTS) |
| `DEEPGRAM_API_KEY` | optional | [Deepgram Console](https://console.deepgram.com) |
| `GOOGLE_API_KEY` | optional | [Google AI Studio](https://aistudio.google.com/projects) (For Gemini LLM) |
| `GOOGLE_APPLICATION_CREDENTIALS` | optional | Path to GCP service account JSON (For Google STT) |
| `SUPABASE_URL` | optional | [Supabase Project Dashboard](https://supabase.com) |
| `SUPABASE_API_KEY` | optional | Supabase Public / Anon Key |

---

## Switching providers

Open `agent_friday.py` and change the provider constants at the top:

```python
STT_PROVIDER = "sarvam"   # "sarvam" | "whisper"
LLM_PROVIDER = "groq"     # "groq" | "gemini" | "openai"
TTS_PROVIDER = "cartesia" # "cartesia" | "openai" | "sarvam"
```

---

## Adding a new tool

1. Create or open a file in `friday/tools/`
2. Define a `register(mcp)` function and decorate tools with `@mcp.tool()`
3. Import and call `register(mcp)` inside `friday/tools/__init__.py`

The MCP server will pick it up on next start.

---

## Tech stack

- **[FastMCP](https://github.com/jlowin/fastmcp)** — MCP server framework
- **[LiveKit Agents](https://github.com/livekit/agents)** — real-time voice pipeline
- **Sarvam Saaras v3** — STT (Indian-English optimised)
- **Groq (Llama-3.3-70b-versatile)** — LLM
- **Cartesia TTS** / **OpenAI TTS** / **ElevenLabs TTS** — TTS
- **[uv](https://github.com/astral-sh/uv)** — fast Python package manager

---

## License

MIT
