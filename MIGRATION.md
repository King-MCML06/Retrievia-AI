# RAG Database Migration: Supabase → Local PostgreSQL

This guide walks through running InferaDoc on a local PostgreSQL 16 instance with `pgvector`, using filesystem storage instead of Supabase buckets.

> **Status:** The application code is fully migrated. There is no Supabase client SDK in this codebase. Database access uses `psycopg2`, vector search uses native pgvector SQL (`<=>`), and files are stored under `./uploads`.

---

## Architecture After Migration

| Layer | Before (Supabase) | After (Local) |
|-------|-------------------|---------------|
| Database | Supabase Postgres + RPC `match_documents` | Local Postgres 16 + pgvector, direct SQL |
| ORM / Client | `@supabase/supabase-js` / Supabase Python SDK | `psycopg2` (native driver) |
| Vector search | `supabase.rpc('match_documents', ...)` | `ORDER BY embedding <=> %s::vector` |
| File storage | Supabase Storage buckets | `./uploads/` directory |
| Auth | Supabase Auth | Custom JWT (`auth.py`) + `app_users` table |
| Config | `SUPABASE_URL`, `SUPABASE_KEY`, etc. | Single `DATABASE_URL` |

---

## Step-by-Step Migration Checklist

### 1. Start local PostgreSQL with pgvector

```bash
cd Document_Summarizer_ChatBot-main
docker compose up -d
```

Wait until the container is healthy:

```bash
docker compose ps
```

Default credentials (from `docker-compose.yml`):

- **User:** `inferadoc`
- **Password:** `inferadoc_secret`
- **Database:** `inferadoc`
- **Port:** `5433` (host) → `5432` (container). Port 5433 avoids conflicts if you already have PostgreSQL installed locally on 5432.

### 2. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and set your API keys:

```env
DATABASE_URL=postgresql://inferadoc:inferadoc_secret@localhost:5433/inferadoc
NOMIC_API_KEY=...
JWT_SECRET=...
GROQ_API_KEY=...   # only needed for audio uploads
```

### 3. Install Python dependencies

```bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

pip install -r requirements.txt
```

### 4. Initialize the database schema

Schema creation runs automatically on API startup via `init_db()` in `database.py`. It:

1. Creates the `app_users` table
2. Runs `CREATE EXTENSION IF NOT EXISTS vector;`
3. Creates `conversations`, `messages`, and `document_chunks` tables
4. Adds an IVFFlat cosine index on embeddings

To initialize manually:

```bash
python -c "from database import init_db; init_db()"
```

### 5. Start the backend

```bash
python server.py
```

API available at `http://localhost:8000`. Health check: `GET /status`.

### 6. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

UI available at `http://localhost:5173`.

### 7. (Optional) Migrate existing Supabase data

If you have data in a Supabase project to move over:

#### 7a. Export from Supabase

```bash
# Conversations, messages, users — adjust table names to match your Supabase schema
pg_dump "postgresql://postgres:[PASSWORD]@db.[PROJECT].supabase.co:5432/postgres" \
  --data-only \
  --table=conversations \
  --table=messages \
  --table=document_chunks \
  > supabase_data.sql
```

#### 7b. Transform Supabase-specific types

Common changes when importing into standard Postgres:

| Supabase | Local Postgres |
|----------|----------------|
| `uuid` with `auth.users` FK | `uuid` FK to `app_users(id)` |
| `vector` via extension | Same — `vector(768)` |
| `jsonb` metadata columns | Unchanged |
| Storage `publicUrl` in metadata | Replace with `uploads/<filename>` relative path in `doc_info.path` |

#### 7c. Download files from Supabase Storage

```python
# Example: download all files from a bucket to ./uploads/
from supabase import create_client
import os

client = create_client(SUPABASE_URL, SUPABASE_KEY)
os.makedirs("uploads", exist_ok=True)

for obj in client.storage.from_("documents").list():
    data = client.storage.from_("documents").download(obj["name"])
    with open(f"uploads/{obj['name']}", "wb") as f:
        f.write(data)
```

Update each conversation's `doc_info` JSON to include `"path": "uploads/<filename>"`.

#### 7d. Import into local Postgres

```bash
psql "postgresql://inferadoc:inferadoc_secret@localhost:5433/inferadoc" -f supabase_data.sql
```

### 8. Verify vector search

```bash
python -c "
from database import search_chunks
# Pass a real conversation UUID and a 768-dim embedding to test
print('search_chunks function ready')
"
```

Or upload a document through the UI and ask a question in chat — the agent's `document_search` tool exercises the full RAG pipeline.

---

## Database Schema Reference

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE app_users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username      VARCHAR(100) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE conversations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
    title       VARCHAR(255) NOT NULL DEFAULT 'New Chat',
    doc_info    JSONB,          -- {name, size, type, chunks, path}
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL,
    content         TEXT NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE document_chunks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    chunk_text      TEXT NOT NULL,
    embedding       vector(768),
    source          VARCHAR(255),
    chunk_index     INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_chunks_embedding
    ON document_chunks
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

### Vector search query (replaces `match_documents` RPC)

```sql
SELECT
    chunk_text,
    source,
    chunk_index,
    1 - (embedding <=> $1::vector) AS similarity
FROM document_chunks
WHERE conversation_id = $2
ORDER BY embedding <=> $1::vector
LIMIT $3;
```

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `extension "vector" does not exist` | Use the `pgvector/pgvector:pg16` Docker image, not plain `postgres:16` |
| `connection refused` on port 5433 | Run `docker compose up -d` and check `docker compose ps` |
| `password authentication failed for user "inferadoc"` | Another PostgreSQL may be bound to the same port. This project uses host port **5433** to avoid clashing with a local install on 5432. Ensure `DATABASE_URL` uses port `5433`. |
| `NOMIC_API_KEY is not set` | Add key to `.env` — required for embeddings |
| IVFFlat index error on empty table | Normal on first run; index builds after data is inserted |
| Uploaded file not found | Check `doc_info.path` in the database points to `uploads/<conv_id>.<ext>` |

---

## Files Changed in Migration

| File | Role |
|------|------|
| `database.py` | psycopg2 connection, schema, CRUD, pgvector search |
| `auth.py` | JWT auth, `app_users` table |
| `server.py` | Local `./uploads` storage, relative paths in `doc_info` |
| `vector_store.py` | Nomic embeddings → `store_chunks` / `search_chunks` |
| `docker-compose.yml` | Local Postgres 16 + pgvector container |
| `.env.example` | Configuration template (`DATABASE_URL` + API keys) |
