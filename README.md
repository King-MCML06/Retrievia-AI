<div align="center">

# 🧠 Retrievia
**AI Document & Audio Summarizer**

[![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/react-%2320232a.svg?style=for-the-badge&logo=react&logoColor=%2361DAFB)](https://reactjs.org/)
[![PostgreSQL](https://img.shields.io/badge/postgresql-4169e1?style=for-the-badge&logo=postgresql&logoColor=white)](https://postgresql.org)
[![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?style=for-the-badge&logo=langchain)](https://langchain.com)
[![pgvector](https://img.shields.io/badge/pgvector-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://github.com/pgvector/pgvector)

**Upload PDF, Word, text, or audio files — get instant summaries, ask questions, and chat with your content.**

Retrievia is a full-stack Retrieval-Augmented Generation (RAG) platform that turns dense documents and recordings into conversational intelligence. It extracts text from multiple file formats, embeds content into a vector database, and powers an agentic AI assistant that can search, summarize topics, produce full-document summaries, and fall back to live web search when needed.

</div>

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Supported File Formats](#supported-file-formats)
- [Technology Stack](#technology-stack)
- [Architecture](#architecture)
- [How It Works](#how-it-works)
- [AI Agent & Tools](#ai-agent--tools)
- [Audio Processing Pipeline](#audio-processing-pipeline)
- [Frontend Application](#frontend-application)
- [Backend API Reference](#backend-api-reference)
- [Database Schema](#database-schema)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Troubleshooting](#troubleshooting)
- [Migration Notes](#migration-notes)

---

## Overview

Retrievia is designed as an **intelligent document summarizer and Q&A assistant**. Whether you upload a research paper (PDF), a business report (DOCX), plain notes (TXT), or a meeting recording (MP3/WAV), the platform:

1. **Extracts** readable text (or transcribes audio)
2. **Chunks** content into overlapping segments for context preservation
3. **Embeds** each chunk as a 768-dimensional vector via Nomic Atlas
4. **Stores** vectors in PostgreSQL with pgvector for fast similarity search
5. **Summarizes** and **answers questions** through a LangGraph ReAct agent powered by local Ollama (Llama 3.1)

The result is a split-pane workspace: your source material on one side, a streaming AI chat on the other — with specialized views for documents vs. audio files.

---

## Features

### Document Summarization

| Capability | Description |
|------------|-------------|
| **Full document summary** | Ask for a comprehensive overview covering main topic, key points, important details, and conclusion via the `full_document_summary` agent tool |
| **Topic-level summary** | Request summaries of specific subjects (e.g., "Summarize the methodology section") via `topic_summarizer` |
| **Semantic search** | Retrieve the most relevant passages before answering via `document_search` |
| **Conversational follow-ups** | Multi-turn chat with conversation memory (LangGraph checkpointing per conversation) |

### Multi-Format Ingestion

| Format | Processing |
|--------|------------|
| **PDF** (`.pdf`) | Text extraction via `PyPDF` |
| **Word** (`.docx`) | Paragraph extraction via `python-docx` |
| **Plain text** (`.txt`) | Direct UTF-8 read |
| **Audio** (`.mp3`, `.wav`, `.m4a`, `.mpeg`, `.flac`, `.ogg`, `.webm`, `.mp4`, `.mpga`) | Groq Whisper transcription + structured Minutes-of-Meeting JSON extraction |

### Audio-Specific Features

- **Cloud transcription** via Groq Whisper (`whisper-large-v3`) — no local GPU required
- **Structured MoM extraction** — automatically generates JSON with overview, discussion points, decisions, action items (with owner/deadline), insights, and summary
- **Raw transcript storage** — saved to disk and served via API for display in the UI
- **WaveSurfer.js player** — interactive waveform, play/pause, ±10s skip, playback speed (1.0x–2.0x)
- **Audio-optimized chat prompts** — quick-action buttons for "Summarize main discussion points" and "List action items"

### Agentic AI (ReAct Architecture)

The AI agent uses a **Reason + Act** loop (LangGraph) with four tools:

1. `document_search` — vector similarity search over uploaded content
2. `topic_summarizer` — focused summary of a specific topic
3. `full_document_summary` — comprehensive document overview
4. `web_search` — DuckDuckGo fallback when the document lacks the answer

The agent is constrained to call **exactly one tool** per turn, then produce a final answer — preventing infinite tool loops and keeping responses fast.

### Real-Time Streaming

- **Upload progress** — Server-Sent Events (SSE) during document embedding (4 steps: save → embed → metadata → agent ready)
- **Chat streaming** — Token-by-token SSE delivery from Ollama to the React frontend
- **Live markdown rendering** — Responses render as they stream with GFM and LaTeX support

### User Authentication & Workspace

- **JWT-based auth** — Register/login with bcrypt password hashing
- **Per-user conversations** — Each user owns isolated conversation threads
- **Conversation management** — Create, rename, delete conversations from the sidebar
- **Auto-titling** — Conversations auto-named from filename or first chat message
- **Persistent message history** — All user/agent messages stored in PostgreSQL

### Rich UI Experience

- **Split-pane document view** — 65% document preview (iframe) + 35% AI chat panel
- **Dedicated audio view** — Waveform player, transcript panel, and insights sidebar
- **Drag-and-drop upload** — Dashboard drop zone with real-time processing feedback
- **Recent conversations grid** — Bento-style cards with file type icons (document vs. microphone)
- **KaTeX math rendering** — Inline `$...$` and block `$$...$$` LaTeX in chat responses
- **Material Design 3 theming** — Custom Tailwind CSS with Manrope/Inter typography

### Vector Search & RAG

- **768-dimensional embeddings** — Nomic Atlas `nomic-embed-text-v1.5`
- **Cosine similarity search** — pgvector `<=>` operator with IVFFlat index
- **Conversation-scoped retrieval** — Each upload is isolated to its conversation ID
- **Overlapping chunks** — 500-word chunks with 50-word overlap to preserve boundary context
- **Batch embedding** — Processes up to 100 chunks per Nomic API call

### Web Search Fallback

When uploaded content does not contain the answer, the agent uses DuckDuckGo search and prefixes responses with `🌍 **Web Search Result:**` to clearly distinguish external sources from document content.

---

## Supported File Formats

```
Documents          Audio / Video (transcribed)
─────────          ────────────────────────────
.pdf               .mp3
.docx              .wav
.txt               .m4a
                   .mpeg / .mpga
                   .flac
                   .ogg
                   .webm
                   .mp4
```

> **Note:** Legacy `.doc` (binary Word) is not supported — only `.docx`. PPT/PPTX mentioned in the dashboard UI is not yet implemented in the backend loader.

---

## Technology Stack

### Backend

| Component | Technology | Purpose |
|-----------|------------|---------|
| **API Framework** | [FastAPI](https://fastapi.tiangolo.com/) + Uvicorn | REST API, SSE streaming, file uploads |
| **Agent Framework** | [LangChain](https://www.langchain.com/) + [LangGraph](https://www.langchain.com/langgraph) | ReAct agent, tool orchestration, memory checkpointing |
| **Local LLM** | [Ollama](https://ollama.com/) — `llama3.1` via `langchain-ollama` | Chat, summarization, Q&A (temperature 0.3) |
| **Embeddings** | [Nomic Atlas API](https://www.nomic.ai/) — `nomic-embed-text-v1.5` | 768-dim vector embeddings |
| **Audio Transcription** | [Groq Cloud](https://groq.com/) — `whisper-large-v3` | GPU-free speech-to-text |
| **Audio MoM Extraction** | Groq — `llama-3.1-8b-instant` | Structured meeting minutes JSON |
| **Database** | PostgreSQL 16 + [pgvector](https://github.com/pgvector/pgvector) | Conversations, messages, vector storage |
| **DB Driver** | `psycopg2-binary` | Native SQL, no ORM |
| **Auth** | `bcrypt` + `PyJWT` (HS256) | Password hashing, JWT tokens |
| **PDF Parsing** | `pypdf` | PDF text extraction |
| **Word Parsing** | `python-docx` | DOCX paragraph extraction |
| **Web Search** | `duckduckgo-search` | Fallback internet search |
| **File Storage** | Local filesystem (`./uploads/`) | Uploaded documents and transcripts |

### Frontend

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Framework** | React 19 + [Vite 8](https://vitejs.dev/) | SPA with HMR |
| **Routing** | React Router DOM 7 | `/`, `/chat/:convId` |
| **Styling** | Tailwind CSS 3 + custom Material Design 3 tokens | Editorial UI theme |
| **HTTP Client** | Axios | REST API calls, auth headers |
| **Streaming** | Native `fetch` + ReadableStream | SSE consumption for upload/chat |
| **Markdown** | `react-markdown` + `remark-gfm` | GitHub-flavored markdown in chat |
| **Math Rendering** | `remark-math` + `rehype-katex` + KaTeX CSS | LaTeX equations in responses |
| **Audio Player** | [WaveSurfer.js 7](https://wavesurfer.xyz/) | Waveform visualization and playback |
| **Icons** | Google Material Symbols + Lucide React | UI iconography |
| **Fonts** | Manrope (headlines) + Inter (body) | Typography |

### Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Database Container** | Docker — `pgvector/pgvector:pg16` | PostgreSQL 16 with pgvector pre-installed |
| **Port Mapping** | Host `5433` → Container `5432` | Avoids conflict with local PostgreSQL |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         React Frontend (Vite :5173)                     │
│  ┌──────────┐  ┌─────────────┐  ┌────────────┐  ┌──────────────────┐   │
│  │ AuthPage │  │  Dashboard  │  │ DocumentView│  │    AudioView    │   │
│  └────┬─────┘  └──────┬──────┘  └─────┬──────┘  └────────┬─────────┘   │
│       │               │               │                   │            │
│       └───────────────┴───────────────┴───────────────────┘            │
│                              │ Axios + SSE                              │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
┌──────────────────────────────┼──────────────────────────────────────────┐
│                    FastAPI Backend (:8000)                              │
│  ┌──────────┐  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │  auth.py │  │  server.py   │  │  agent.py   │  │ vector_store.py│  │
│  │ JWT/bcrypt│  │ REST + SSE  │  │ LangGraph   │  │ Nomic embed    │  │
│  └────┬─────┘  └──────┬───────┘  │ ReAct Agent │  └───────┬────────┘  │
│       │               │          └──────┬──────┘          │            │
│       │               │                 │                 │            │
│  ┌────┴───────────────┴─────────────────┴─────────────────┴────────┐  │
│  │                        database.py (psycopg2)                    │  │
│  └──────────────────────────────┬───────────────────────────────────┘  │
│                                 │                                       │
│  ┌──────────────────────────────┴───────────────────────────────────┐  │
│  │              document_loader.py                                   │  │
│  │   PDF │ DOCX │ TXT │ Audio → Groq Whisper → MoM JSON             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
    ┌────┴────┐          ┌─────┴─────┐         ┌─────┴─────┐
    │ Ollama  │          │ PostgreSQL│         │  Nomic    │
    │ llama3.1│          │ + pgvector│         │  Atlas    │
    │ :11434  │          │  :5433    │         │  API      │
    └─────────┘          └───────────┘         └───────────┘
                              │
                         ┌────┴────┐
                         │  Groq   │
                         │  Cloud  │
                         │ (audio) │
                         └─────────┘
```

---

## How It Works

### Document Upload Pipeline

```
User uploads file
       │
       ▼
┌──────────────────┐
│ 1. Save to disk  │  uploads/{conv_id}.{ext}
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 2. Extract text  │  document_loader.py
│    (or transcribe│  PDF/DOCX/TXT → plain text
│     for audio)   │  Audio → Groq Whisper → MoM JSON
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 3. Chunk text    │  500 words/chunk, 50-word overlap
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 4. Embed chunks  │  Nomic Atlas (768-dim vectors)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 5. Store vectors │  PostgreSQL document_chunks table
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 6. Build agent   │  LangGraph ReAct agent scoped to conv_id
└────────┬─────────┘
         ▼
    Ready to chat
```

### Chat / Summarization Flow

```
User sends message
       │
       ▼
┌──────────────────┐
│ Persist to DB    │  messages table (role: user)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ ReAct Agent      │  Ollama llama3.1 decides which tool to call
│ (LangGraph)      │
└────────┬─────────┘
         │
    ┌────┴────┬──────────────┬─────────────────┐
    ▼         ▼              ▼                 ▼
 document  topic         full_document    web_search
 _search   _summarizer   _summary         (DuckDuckGo)
    │         │              │                 │
    └────┬────┴──────────────┴─────────────────┘
         ▼
┌──────────────────┐
│ Generate answer  │  Stream tokens via SSE
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Persist to DB    │  messages table (role: agent)
└──────────────────┘
```

---

## AI Agent & Tools

The agent (`agent.py`) is built with LangGraph's `create_react_agent` and uses Ollama's `llama3.1` model at temperature 0.3. Each conversation gets its own agent instance with tools scoped to that conversation's vector store.

### Tool Reference

| Tool | Input | Behavior |
|------|-------|----------|
| `document_search` | Query string | Embeds query, retrieves top-4 similar chunks with cosine scores, returns ranked excerpts |
| `topic_summarizer` | Topic string | Retrieves top-5 relevant chunks, sends to LLM with summarization prompt |
| `full_document_summary` | (none) | Retrieves top-8 overview chunks, generates structured summary (topic, key points, details, conclusion) |
| `web_search` | Query string | Queries DuckDuckGo (max 3 results), returns source URLs and snippets |

### Agent System Prompt Rules

1. Call exactly **one** tool to gather context
2. Immediately write the final answer — no additional tool loops
3. Use `web_search` only when the document lacks the answer
4. Prefix web search responses with `🌍 **Web Search Result:**`
5. Render all math in strict LaTeX (`$...$` inline, `$$...$$` block)

### Memory

- **LangGraph MemorySaver** checkpointing keyed by `thread_id = conv_id`
- Enables multi-turn conversation context within a single upload session

---

## Audio Processing Pipeline

When an audio file is uploaded, the `load_mp3()` function in `document_loader.py` runs a two-stage cloud pipeline:

### Stage 1: Transcription (Groq Whisper)

```
Audio file → Groq whisper-large-v3 → Raw transcript text
                                   → Saved as {conv_id}_transcript.txt
```

### Stage 2: Structured MoM Extraction (Groq LLM)

The raw transcript is sent to `llama-3.1-8b-instant` with a JSON schema prompt. The output is structured as:

```json
{
  "overview": "Meeting purpose and context",
  "discussion_points": ["Point 1", "Point 2"],
  "decisions": ["Decision 1"],
  "action_items": [
    {"task": "...", "owner": "...", "deadline": "..."}
  ],
  "insights": ["Insight 1"],
  "summary": "Executive summary paragraph"
}
```

This JSON is what gets chunked, embedded, and stored — making action items, decisions, and discussion points **queryable via RAG**.

> **Requires:** `GROQ_API_KEY` in `.env`. Without it, audio uploads will fail.

---

## Frontend Application

### Pages & Routes

| Route | Component | Purpose |
|-------|-----------|---------|
| `/` (unauthenticated) | `AuthPage` | Login / Register |
| `/` (authenticated) | `Dashboard` | Upload zone + recent conversations |
| `/chat/:convId` | `ChatRouter` | Routes to DocumentView or AudioView |

### DocumentView

- **Left panel (65%)** — Inline document preview via iframe (`/conversations/{id}/file`)
- **Right panel (35%)** — Streaming AI chat with markdown + KaTeX rendering
- **Empty state** — Prompts user to upload from dashboard
- **Streaming chat** — Real-time markdown + KaTeX rendering in the chat panel

### AudioView

- **Center panel** — WaveSurfer waveform player with play/pause, skip ±10s, speed control
- **Transcript section** — Scrollable raw transcript fetched from `/conversations/{id}/transcript`
- **Right sidebar** — AI chat with quick-action summary prompts
- **Export button** — Download original audio file

### Layout Components

- **TopNav** — Brand header with link to dashboard
- **SideNav** — Conversation list with inline rename, delete confirmation modal, user profile, sign out

### Auth Flow

1. User registers or logs in → receives JWT token
2. Token stored in `localStorage` and attached to all Axios requests
3. On app load, token verified via `GET /auth/me`
4. Network failures gracefully fall back to local JWT payload parsing

---

## Backend API Reference

Base URL: `http://localhost:8000`

### System

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/status` | No | Health check — returns `{"status": "ready"}` |

### Authentication

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/auth/register` | No | Register new user (`username`, `password`) |
| `POST` | `/auth/login` | No | Login and receive JWT token |
| `GET` | `/auth/me` | Yes | Verify token, return current user |

### Conversations

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/conversations` | Yes | List user's conversations (most recent first) |
| `POST` | `/conversations` | Yes | Create new conversation |
| `GET` | `/conversations/{id}` | Yes | Get conversation metadata + message history |
| `PATCH` | `/conversations/{id}` | Yes | Rename conversation (`title`) |
| `DELETE` | `/conversations/{id}` | Yes | Delete conversation, messages, chunks, and files |

### Documents & Files

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/conversations/{id}/upload` | Yes | Upload file — returns SSE progress stream |
| `GET` | `/conversations/{id}/file` | Yes | Serve uploaded file (inline preview/download) |
| `GET` | `/conversations/{id}/transcript` | Yes | Return raw audio transcript text |

### Chat

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/conversations/{id}/chat` | Yes | Send message — returns SSE token stream |

**Auth header:** `Authorization: Bearer <token>` or query param `?token=<token>` (used for iframe file preview).

### Upload SSE Events

```json
{"step": 1, "message": "File saved. Extracting content..."}
{"step": 2, "message": "Embedded N knowledge chunks into vector store."}
{"step": 3, "message": "Metadata saved. Building AI agent..."}
{"step": 4, "message": "Done"}
```

### Chat SSE Events

```json
{"chunk": "partial response text"}
{"chunk": "[DONE]"}
```

---

## Database Schema

PostgreSQL 16 with the `vector` extension enabled.

### Tables

**`app_users`** — User accounts
```sql
id            UUID PRIMARY KEY
username      VARCHAR(100) UNIQUE NOT NULL
password_hash TEXT NOT NULL
created_at    TIMESTAMPTZ
```

**`conversations`** — Per-user chat threads
```sql
id          UUID PRIMARY KEY
user_id     UUID → app_users(id) ON DELETE CASCADE
title       VARCHAR(255) DEFAULT 'New Chat'
doc_info    JSONB   -- {name, size, type, chunks, path}
created_at  TIMESTAMPTZ
updated_at  TIMESTAMPTZ
```

**`messages`** — Chat history
```sql
id              UUID PRIMARY KEY
conversation_id UUID → conversations(id) ON DELETE CASCADE
role            VARCHAR(20)   -- 'user' | 'agent'
content         TEXT
created_at      TIMESTAMPTZ
```

**`document_chunks`** — Vector embeddings
```sql
id              UUID PRIMARY KEY
conversation_id UUID → conversations(id) ON DELETE CASCADE
chunk_text      TEXT
embedding       vector(768)
source          VARCHAR(255)
chunk_index     INTEGER
created_at      TIMESTAMPTZ
```

**Index:** IVFFlat cosine index on `document_chunks.embedding`

### Vector Search Query

```sql
SELECT chunk_text, 1 - (embedding <=> $query_vector) AS similarity
FROM document_chunks
WHERE conversation_id = $conv_id
ORDER BY embedding <=> $query_vector
LIMIT $top_k;
```

---

## Project Structure

```
Retrievia-AI/
├── server.py              # FastAPI app — routes, SSE, file serving
├── agent.py               # LangGraph ReAct agent + streaming chat
├── document_loader.py     # PDF/DOCX/TXT/audio loading + chunking
├── vector_store.py        # Nomic embeddings + pgvector storage/search
├── database.py            # PostgreSQL schema, CRUD, vector search
├── auth.py                # JWT auth, user registration/login
├── requirements.txt       # Python dependencies
├── docker-compose.yml     # PostgreSQL 16 + pgvector container
├── .env.example           # Environment variable template
├── MIGRATION.md           # Supabase → local Postgres migration guide
├── uploads/               # Local file storage (created at runtime)
│
└── frontend/
    ├── src/
    │   ├── App.jsx                    # Auth gate + routing
    │   ├── pages/
    │   │   ├── AuthPage.jsx           # Login / Register
    │   │   ├── Dashboard.jsx          # Upload + recent conversations
    │   │   ├── ChatRouter.jsx         # Document vs Audio view router
    │   │   ├── DocumentView.jsx       # PDF/DOCX/TXT split-pane chat
    │   │   └── AudioView.jsx          # Audio player + transcript + chat
    │   └── components/
    │       └── Layout.jsx             # TopNav, SideNav, delete modal
    ├── package.json
    ├── tailwind.config.js
    └── vite.config.js
```

---

## Getting Started

### Prerequisites

- **Python** 3.10+
- **Node.js** 18+
- **Docker** (for PostgreSQL with pgvector)
- **Ollama** with `llama3.1` model pulled locally
- **API Keys:** Nomic Atlas (required), Groq (required for audio)

### 1. Start PostgreSQL

```bash
docker compose up -d
```

Verify the container is healthy:

```bash
docker compose ps
```

Default connection: `postgresql://Retrievia:Retrievia_secret@localhost:5433/Retrievia`

### 2. Configure Environment

```bash
cp .env.example .env
```

Edit `.env` with your keys (see [Environment Variables](#environment-variables)).

### 3. Start Ollama

```bash
ollama serve          # if not already running
ollama pull llama3.1
```

### 4. Install & Run Backend

```bash
python -m venv venv

# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

pip install -r requirements.txt
python server.py
```

API available at **http://localhost:8000**

### 5. Install & Run Frontend

```bash
cd frontend
npm install
npm run dev
```

UI available at **http://localhost:5173**

### 6. Use the App

1. **Register** an account on the login page
2. **Upload** a PDF, DOCX, TXT, or audio file via drag-and-drop on the dashboard
3. Wait for processing (embedding progress shown in real time)
4. **Chat** with your document — ask for summaries, specific topics, or detailed questions
5. For audio files, view the **transcript** and use quick-action prompts for meeting summaries

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string (default: `postgresql://Retrievia:Retrievia_secret@127.0.0.1:5433/Retrievia`) |
| `NOMIC_API_KEY` | Yes | Nomic Atlas API key for text embeddings |
| `JWT_SECRET` | Yes | Secret for signing JWT tokens — change in production |
| `GROQ_API_KEY` | For audio | Groq Cloud API key for Whisper transcription and MoM extraction |

Ollama runs locally and requires **no API key**. Ensure it is serving at `http://localhost:11434`.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `extension "vector" does not exist` | Use the `pgvector/pgvector:pg16` Docker image, not plain `postgres:16` |
| `connection refused` on port 5433 | Run `docker compose up -d` and verify with `docker compose ps` |
| `NOMIC_API_KEY is not set` | Add your Nomic Atlas key to `.env` |
| `GROQ_API_KEY is missing` | Required for audio uploads — add to `.env` |
| Ollama connection errors | Ensure `ollama serve` is running and `llama3.1` is pulled |
| IVFFlat index error on empty table | Normal on first run — index builds after data is inserted |
| Uploaded file not found | Check `doc_info.path` in database points to `uploads/{conv_id}.{ext}` |
| Empty PDF extraction | Scanned PDFs without OCR text layers will produce empty content |

---

## Migration Notes

This project was migrated from Supabase to local PostgreSQL. The codebase no longer uses the Supabase SDK — all database access is via `psycopg2`, vector search uses native pgvector SQL, and files are stored in `./uploads/`.

For the full migration checklist, schema reference, and Supabase data export instructions, see **[MIGRATION.md](MIGRATION.md)**.

---

<div align="center">

*Built to turn documents and recordings into actionable, conversational intelligence.*

</div>
