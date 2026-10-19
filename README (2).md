# Copilot

**A full-stack AI assistant with Chat, Voice, and Image understanding.**

Copilot runs on a **local model first** (Ollama, `gemma3:4b` by default) and automatically falls back to **cloud AI** (Anthropic Claude, OpenAI, Google Gemini, xAI Grok, Meta Llama) when no local model is available or a local model fails.

- **GitHub repo:** `<add your public repo link here>`
- **Live demo:** `<optional: add your Vercel / Render / Railway link here>`
- **Word documentation:** [`docs/Copilot-Documentation.docx`](docs/Copilot-Documentation.docx)

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Screenshots](#screenshots)
4. [My Contributions](#my-contributions)
5. [Tech Stack](#tech-stack)
6. [Architecture](#architecture)
7. [Project Structure](#project-structure)
8. [Setup & Installation](#setup--installation)
9. [Environment Variables](#environment-variables)
10. [How It Works](#how-it-works)
11. [API Reference](#api-reference)
12. [Testing](#testing)
13. [Author](#author)

---

## Overview

Copilot has three services that run side by side:

| Service | Folder | Role |
|---|---|---|
| **Frontend** | `frontend/` | Next.js chat UI: history, model picker, photos, dictation, voice mode |
| **Backend** | `backend/` | FastAPI: model providers, streaming chat, fallback logic, image checks, speech-to-text, voice sessions |
| **Voice agent** | `agent/` | LiveKit agent: listens, transcribes, answers through the backend, speaks |

Design goals: **privacy** (local models and local photos by default), **resilience** (automatic fallback so a reply is rarely lost), and **simple, tested code**.

---

## Features

The three required features:

### 1. Chat / Text
- Streaming replies (server-sent events) with **Stop** and **Regenerate**.
- Markdown with tables and **copyable code blocks**.
- Conversation history with **search**, date grouping, and **delete with undo** (saved in the browser).
- Quote a reply, copy it, give thumbs up / down.
- Keyboard shortcuts: `Ctrl/Cmd + Shift + O` new chat, `Ctrl/Cmd + K` search, `Esc` stop.
- Model picker grouped as **Local · Ollama** and **Cloud (fallback)**, with live model lists.
- Light and dark themes (follows your system), mobile-friendly layout.

### 2. Voice (speech input + voice output)
- **Voice mode:** real-time spoken conversation over LiveKit. Talk any time, even over the assistant.
- **Live captions**, mute button, and a 20-second timeout with a clear message if the agent is not running.
- Voice answers use **the same model picker and fallback** as text chat, and the earlier text chat is carried into the call.
- The spoken conversation is saved into the chat, tagged **Spoken**.
- **Dictation:** the mic button records your voice with a live level meter and timer, transcribes it with Groq Whisper, and puts the text in the message box.

### 3. Image understanding
- Add up to **5 photos per message** (JPEG, PNG, WebP, GIF) with the paperclip, camera, paste, or drag and drop.
- The browser **resizes photos to 2048 px and re-encodes them**, which also removes metadata such as GPS location.
- **Vision-aware routing:** with photos attached, Auto only picks a model that can see images. A text-only model is rejected with a one-click "Switch to Auto".
- Models that can see images are tagged **"Sees images"**.
- A privacy line tells you whether photos stay on your computer or which provider receives them.
- Server checks: the real file type is read from the file signature (not the claimed type), plus per-image, per-message and per-request limits and a request-size cap (HTTP 413).

---

## Screenshots

> Add your own screenshots to `docs/screenshots/` using these names, then they will show up here.

| Feature | Screenshot |
|---|---|
| Chat | ![Chat](docs/screenshots/chat.png) |
| Voice | ![Voice](docs/screenshots/voice.png) |
| Image understanding | ![Image](docs/screenshots/image.png) |

_Optional demo GIF:_ `![Demo](docs/screenshots/demo.gif)`

---

## My Contributions

> This project was built with AI assistance (Claude Code) guided by my `AGENTS.md` rules, and it builds on the provided agent skills for the three core features (chat, voice, image). This section says what I did myself.

**What I did**
- Defined the project: the stack, the folder layout (`frontend/`, `backend/`, `agent/`) and the working rules in `AGENTS.md` (uv + Ruff + pytest, no secrets in git, minimal dependencies).
- Chose the design: **local-first with Ollama (`gemma3:4b`) and a cloud fallback** across five providers, instead of a single cloud API.
- Created and configured my own LiveKit Cloud project and API keys for voice mode.
- Reviewed, ran and tested the app.

**What I customized or added myself** _(fill this in with your real work before submitting)_
- [ ] e.g. changed the system prompt / assistant personality
- [ ] e.g. customized the UI theme, colors or layout
- [ ] e.g. added a feature of my own (for example: a new voice, saved prompts, a new provider)
- [ ] e.g. wrote extra tests or documentation

---

## Tech Stack

### Frontend (`frontend/`)
| Area | Technology |
|---|---|
| Framework | Next.js 16, React 19, TypeScript |
| Styling | Tailwind CSS 4, lucide-react icons |
| Content | react-markdown, remark-gfm |
| Voice | `livekit-client` |
| Storage | `localStorage` (chat text); photos stay in memory only |

### Backend (`backend/`)
| Area | Technology |
|---|---|
| Framework | FastAPI, Pydantic v2, pydantic-settings |
| Tooling | Python 3.13, uv, Uvicorn, Ruff, pytest |
| Local AI | Ollama Python SDK |
| Cloud AI | Anthropic SDK; OpenAI SDK for OpenAI, Gemini, Grok, Meta (OpenAI-compatible endpoints) |
| Speech-to-text | Groq Whisper (`whisper-large-v3-turbo`) |
| Voice | `livekit-api` (room tokens and agent dispatch) |
| Streaming | Server-sent events |

### Voice agent (`agent/`)
| Area | Technology |
|---|---|
| Framework | LiveKit Agents |
| Speech-to-text | `assemblyai/universal-3-5-pro` (LiveKit Inference) |
| Text-to-speech | `fishaudio/s2.1-pro` (LiveKit Inference) |
| Voice activity | Silero VAD |
| Noise cancellation | LiveKit BVC |
| LLM | The Copilot backend (`VOICE_LLM=app`), or any LiveKit Inference model id |

### AI models and APIs
| Provider | Type | Enabled by |
|---|---|---|
| Ollama (`gemma3:4b`, Llama, Qwen, LLaVA...) | Local | `OLLAMA_ENABLED=true` (default) |
| Anthropic Claude | Cloud | `ANTHROPIC_API_KEY` |
| OpenAI | Cloud | `OPENAI_API_KEY` |
| Google Gemini | Cloud | `GEMINI_API_KEY` |
| xAI Grok | Cloud | `GROK_API_KEY` |
| Meta Llama API | Cloud | `META_API_KEY` |
| Groq Whisper | Dictation | `GROQ_API_KEY` |
| LiveKit Cloud + Inference | Voice mode | `LIVEKIT_*` |

---

## Architecture

```
┌──────────────────── Browser (Next.js) ─────────────────────┐
│ Chat UI · Model picker · Photos · Dictation · Voice panel   │
└──────┬──────────────────────────────────┬──────────────────┘
       │ /api/*  (proxied by Next.js)     │ WebRTC (mic + speaker)
       ▼                                  ▼
┌─────────────────┐   POST /api/voice/chat   ┌────────────────┐
│ FastAPI backend │◄─────────────────────────│  LiveKit Cloud │
│  /api/chat      │                          └───────┬────────┘
│  /api/models    │                                  ▼
│  /api/stt       │                          ┌────────────────┐
│  /api/voice/*   │◄─────────────────────────│  Voice agent   │
└───┬─────────┬───┘                          │ STT → LLM → TTS│
    ▼         ▼                              └────────────────┘
 Ollama   Cloud providers
 (local)  Anthropic · OpenAI · Gemini · Grok · Meta
```

**Voice session flow**
1. The browser calls `POST /api/voice/session`. The backend creates a room, remembers the chosen model and earlier chat, and returns a token that also dispatches the agent.
2. The browser joins the room with WebRTC. The agent joins, greets the user and listens.
3. After each spoken turn, the agent posts the conversation to `POST /api/voice/chat`. The backend picks the model (with fallback), streams the reply, and the agent speaks it.
4. When the call ends, the spoken transcript is added to the chat.

---

## Project Structure

```
copilot/
├── AGENTS.md
├── backend/                 FastAPI backend
│   ├── app/
│   │   ├── main.py          App factory, CORS, request-size limit
│   │   ├── config.py        Settings from environment variables
│   │   ├── schemas.py       Request models, image validation
│   │   ├── images.py        Image format detection and decoding
│   │   ├── vision.py        Which models can see images
│   │   ├── providers/       ollama, anthropic, openai_compat, registry
│   │   ├── services/chat.py Streaming + fallback logic
│   │   └── routes/          chat, models, health, stt, voice
│   └── tests/               pytest suite (fake providers, no network)
├── agent/                   LiveKit voice agent
│   ├── voice_agent.py       Entrypoint
│   ├── app_llm.py           LLM adapter that answers through the backend
│   ├── settings.py
│   └── tests/
└── frontend/                Next.js app
    ├── app/                 layout, page, styles
    ├── components/          chat-app, composer, messages, sidebar, voice-panel...
    ├── hooks/               use-chat, use-models, use-recorder, use-voice
    └── lib/                 api client, types, image helpers, storage
```

---

## Setup & Installation

### Prerequisites
- Python 3.13 and [uv](https://docs.astral.sh/uv/)
- Node.js 20.9+ and npm
- [Ollama](https://ollama.com) (recommended, for local models)
- A [LiveKit Cloud](https://cloud.livekit.io) project (voice mode only)
- A [Groq](https://console.groq.com) API key (dictation only, free tier available)
- Cloud API keys (optional fallback)

### 1. Clone
```bash
git clone <your-repo-url>
cd copilot
```

### 2. Pull local models
```bash
ollama pull gemma3:4b        # chat + image understanding
```

### 3. Run the backend → http://localhost:8000/docs
```bash
cd backend
uv sync
cp .env.example .env         # add keys if you have them
uv run fastapi dev
```

### 4. Run the frontend → http://localhost:3000
In a new terminal:
```bash
cd frontend
npm install
cp .env.example .env.local   # API_URL=http://localhost:8000
npm run dev
```
The browser calls `/api/*` on the Next.js app, which forwards the request to the FastAPI server, so no CORS setup is needed.

### 5. Enable voice mode (optional)
Put the same LiveKit values in **both** `backend/.env` and `agent/.env`:
```bash
LIVEKIT_URL=wss://<your-project>.livekit.cloud
LIVEKIT_API_KEY=...
LIVEKIT_API_SECRET=...
```
Then start the agent (the backend must already be running):
```bash
cd agent
uv sync
cp .env.example .env
uv run python voice_agent.py dev
```
The **Voice** button in the composer turns on once the backend has LiveKit credentials.

### 6. Enable dictation (optional)
```bash
# backend/.env
GROQ_API_KEY=your-groq-key
```

---

## Environment Variables

> **Never commit real API keys.** `.env` and `.env.local` are in `.gitignore`. Only the `.env.example` files (with empty values) are committed.

### Backend (`backend/.env`)

| Variable | Default | Purpose |
|---|---|---|
| `ENVIRONMENT` | `development` | Set `production` to hide `/docs` |
| `CORS_ORIGINS` | `["http://localhost:3000"]` | Allowed browser origins (JSON list) |
| `SYSTEM_PROMPT` | helpful assistant | System prompt for every chat |
| `ALLOW_CLOUD_FALLBACK` | `true` | Try other models when one fails |
| `OLLAMA_ENABLED` | `true` | Use local models |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama address |
| `OLLAMA_MODEL` | `gemma3:4b` | Preferred local model |
| `CLOUD_PRIORITY` | `["anthropic","openai","gemini","grok","meta"]` | Cloud fallback order |
| `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `GROK_API_KEY`, `META_API_KEY` | empty | A provider is enabled when its key is set |
| `ANTHROPIC_MODEL`, `OPENAI_MODEL`, ... | empty | Preferred model per provider (empty = first listed) |
| `GROQ_API_KEY` | empty | Dictation (speech-to-text) |
| `MAX_IMAGES_PER_MESSAGE` | `5` | Photos per message |
| `MAX_IMAGE_BYTES` | `10485760` | Max size of one photo |
| `VISION_MODELS` | `[]` | Extra patterns for models that can see images |
| `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | empty | Enable voice mode |
| `VOICE_AGENT_NAME` | `copilot-agent` | Must match the agent |
| `VOICE_AGENT_TOKEN` | empty | Shared secret protecting `/api/voice/chat` |

### Voice agent (`agent/.env`)

| Variable | Default | Purpose |
|---|---|---|
| `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | empty | Same LiveKit project as the backend |
| `VOICE_AGENT_NAME` | `copilot-agent` | Must match the backend |
| `VOICE_LLM` | `app` | `app` = answer through the backend |
| `API_URL` | `http://localhost:8000` | Where the agent reaches the backend |
| `VOICE_AGENT_TOKEN` | empty | Same value as in the backend |
| `VOICE_STT_MODEL`, `VOICE_STT_LANGUAGE` | `assemblyai/universal-3-5-pro`, `en` | Speech-to-text |
| `VOICE_TTS_MODEL`, `VOICE_TTS_VOICE` | `fishaudio/s2.1-pro`, empty | Text-to-speech |
| `VOICE_NOISE_CANCELLATION` | `true` | Set `false` on self-hosted LiveKit |
| `VOICE_GREETING` | friendly greeting | First thing the agent says |

### Frontend (`frontend/.env.local`)

| Variable | Default | Purpose |
|---|---|---|
| `API_URL` | `http://localhost:8000` | Where Next.js forwards `/api/*` |

---

## How It Works

### Model selection and fallback
1. With **Auto**, the backend uses your preferred local model (`OLLAMA_MODEL`) if it is installed, otherwise the first local model, otherwise the first cloud provider with a key.
2. If the chosen model fails **before** sending any text, the backend tries the next one (local first, then `CLOUD_PRIORITY` order).
3. The reply starts with a `meta` event that says which model answered and, if a fallback happened, why.
4. If a model fails **in the middle** of an answer, an `error` event is sent and no other model takes over, so one reply never mixes two models.

### Photos and vision
When any message has photos, Auto and fallback only consider models that can see images. If you pick a text-only model, the request is rejected right away with a clear message. Photos go to Ollama as raw bytes, to Anthropic as image blocks, and to OpenAI-compatible APIs as `image_url` parts. Nothing is written to disk on the server.

### Dictation
The composer records audio with `MediaRecorder`, uploads it to `POST /api/stt`, and the backend sends it to Groq Whisper. The text is added to the message box.

---

## API Reference

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/health` | Liveness check |
| `GET` | `/api/models?refresh=true` | Models grouped by provider, with vision flags and limits |
| `POST` | `/api/chat` | Stream a reply as server-sent events |
| `POST` | `/api/stt` | Transcribe audio (max 25 MB) |
| `GET` | `/api/voice/config` | Whether voice mode is enabled |
| `POST` | `/api/voice/session` | Create a room and a token that dispatches the agent |
| `POST` | `/api/voice/chat` | Called by the agent for a spoken-turn reply |

**Chat request**
```json
{
  "messages": [
    { "role": "user", "content": "What is in this photo?",
      "images": [{ "media_type": "image/jpeg", "data": "<base64>" }] }
  ],
  "provider": "ollama",
  "model": "gemma3:4b"
}
```
`provider` and `model` are optional. Leave them out for Auto.

**Stream events**
```
event: meta
data: {"provider":"ollama","model":"gemma3:4b","local":true,"fallback":false,"notice":null}

data: {"delta":"Hello"}

event: done        (or)   event: error
data: {}                  data: {"message":"..."}
```

---

## Testing

```bash
# Backend
cd backend && uv run ruff check . && uv run ruff format --check . && uv run pytest

# Voice agent
cd agent && uv run ruff check . && uv run ruff format --check . && uv run pytest

# Frontend
cd frontend && npm run lint && npm run typecheck && npm run build
```

The backend and agent tests use fake providers and mocked HTTP, so they run without Ollama, API keys or network access.

---

## Author

**Azeefa Tahir Malhi**

| | |
|---|---|
| GitHub | [github.com/malhiazeefa6-collab](https://github.com/malhiazeefa6-collab) |
| LinkedIn | [linkedin.com/in/azeefa-malhi-05173628b](https://www.linkedin.com/in/azeefa-malhi-05173628b/) |
| Email | [malhiazeefa6@gmail.com](mailto:malhiazeefa6@gmail.com) |

<p align="center">Built with FastAPI, Next.js, Ollama, and LiveKit.</p>
