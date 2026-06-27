<div align="center">
  
# 🧠 InferaDoc
**High-Performance AI Document Assistant**

[![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/react-%2320232a.svg?style=for-the-badge&logo=react&logoColor=%2361DAFB)](https://reactjs.org/)
[![PostgreSQL](https://img.shields.io/badge/postgresql-4169e1?style=for-the-badge&logo=postgresql&logoColor=white)](https://postgresql.org)
[![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?style=for-the-badge&logo=langchain)](https://langchain.com)

InferaDoc is an advanced agentic document analysis platform. Upload your documents, ask complex questions, and let the AI instantly search, summarize, and retrieve precise information. 
</div>

---

## ✨ Features

- 📄 **Universal Document Support**: Effortlessly parses and processes PDFs, Word Documents (`.docx`), and standard Text files (`.txt`).
- 🤖 **Agentic AI Architecture**: Powered by a ReAct (Reason+Act) architecture using LangGraph, allowing the model to decide whether to search the document, summarize a topic, or search the live web.
- 💬 **Live Conversational Streaming**: Features a robust FastAPI backend streaming responses to a dynamic React frontend in real-time.
- ⚡ **Lightning Fast Vector Storage**: Utilizes PostgreSQL configured with `pgvector` for hyper-efficient local and cloud-ready ANN embedding search.
- 🌍 **Web Search Fallback**: Seamlessly bridges facts—if the answer isn't in your document, the agent integrates DuckDuckGo search to prevent hallucinations.
- 🧮 **Native Math/LaTeX Rendering**: Built-in support to parse, render, and display complex equations using KaTeX and Markdown.

## 🛠️ Technology Stack

InferaDoc is split into a high-performance Python server and a modern React client.

### Backend (`/` & `/server.py`)
* **Core Framework:** [FastAPI](https://fastapi.tiangolo.com/) (ASGI, fast, type-safe)
* **Agent logic:** [LangChain](https://www.langchain.com/) & [LangGraph](https://www.langchain.com/langgraph) (ReAct execution, memory checkpointing)
* **LLM Engine:** [Ollama](https://ollama.com/) — local `llama3.1` via `langchain-ollama` (default `http://localhost:11434`)
* **Database & Vector Store:** [PostgreSQL](https://www.postgresql.org/) 16 with `pgvector` via `psycopg2` (local Docker)
* **File Processing:** `PyPDF`, `python-docx`
* **Embeddings:** [Nomic Atlas API](https://www.nomic.ai/) (`nomic-embed-text-v1.5`, 768 dimensions)

### Frontend (`/frontend`)
* **Framework:** React 19 (via [Vite](https://vitejs.dev/))
* **UI & Styling:** Modern custom CSS, Lucide React icons
* **Markdown Parsing:** `react-markdown` augmented with `remark-math` and `rehype-katex` for crisp rendering.
* **Networking:** Axios for REST API and SSE (Server-Sent Events) for real-time streaming.

## 🚀 Getting Started

### 1. Prerequisites
- Python 3.10+
- Node.js 18+
- [Docker](https://www.docker.com/) (for local PostgreSQL with pgvector)
- [Ollama](https://ollama.com/) with the `llama3.1` model pulled locally

### 2. Database Setup (Docker)

Start PostgreSQL 16 with the pgvector extension:

```bash
docker compose up -d
```

### 3. Backend Setup
Clone the repository and install the Python dependencies:

```bash
# Clone the repository
git clone https://github.com/yourusername/InferaDoc.git
cd InferaDoc

# (Optional) Create a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows use `venv\Scripts\activate`

# Install dependencies
pip install -r requirements.txt
```

Copy the environment template and fill in your API keys:

```bash
cp .env.example .env
```

Required variables:

```env
DATABASE_URL=postgresql://inferadoc:inferadoc_secret@localhost:5433/inferadoc
NOMIC_API_KEY=your_nomic_api_key
JWT_SECRET=change-me-in-production
GROQ_API_KEY=your_groq_api_key        # for audio uploads only
```

Start Ollama and pull the local LLM (no API key needed):

```bash
ollama serve          # if not already running
ollama pull llama3.1
```

Launch the FastAPI backend server:
```bash
python server.py
# The API will be available at http://localhost:8000
```

> See [MIGRATION.md](MIGRATION.md) for the full Supabase → local Postgres migration checklist.

### 4. Frontend Setup
Open a new terminal window in the `frontend` folder:

```bash
cd frontend

# Install Node dependencies
npm install

# Start the Vite development server
npm run dev
# The website will be available at http://localhost:5173
```

## 🧠 How it Works

1. **Document Upload**: When you upload a `.pdf` or `.docx`, the backend extracts the text, chunks it intelligently, and creates 768-dimensional embeddings, inserting them directly into PostgreSQL.
2. **ReAct Loop**: Upon asking a question, LangGraph's ReAct agent intercepts it. The AI chooses one of several tools (e.g., `document_search`, `topic_summarizer`, `full_document_summary`, `web_search`) to contextually gather the right data before answering.
3. **Streaming**: As the model generates its final thought, words are streamed chunk-by-chunk over HTTP via Server-Sent Events straight to the React frontend UI.

---
*Built from scratch to redefine context-aware chat experiences.*
