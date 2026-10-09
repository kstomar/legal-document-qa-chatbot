# Legal Document Q&A Assistant

A production-ready, grounded legal document question-and-answer assistant built with FastAPI. It enables users to upload legal PDFs and contracts, parse document structure (articles, sections, clauses), and ask grounded questions with strict citations referencing exact pages, clauses, sections, and source text snippets.

---

## Key Highlights & Production Architecture

* **High-Precision Hybrid Retrieval**: Combines dense semantic search (Gemini Embedding / FastEmbed) with BM25 lexical ranking fused via Reciprocal Rank Fusion (RRF).
* **Durable Vector Storage (Neon PostgreSQL + pgvector)**: Production vector embeddings and chunks are persisted in Neon PostgreSQL using `pgvector` as the durable single source of truth, backed by an in-memory Qdrant acceleration cache (eliminating ephemeral disk data loss on hosts like Render).
* **Object Storage (Supabase Storage)**: Pluggable storage abstraction supporting local storage and private Supabase Storage buckets for uploaded documents.
* **LLM Engine**: Primary support for Groq (`llama-3.3-70b-versatile`) and Google Gemini (`gemini-2.5-flash`), with fallback/testing adapters for OpenAI and local mocks.
* **Legal Grounding & Anti-Hallucination**: Enforces strict evidence grounding, citation verification, and explicit confidence gating.
* **Multi-Turn Context & Dialogue**: Tracks multi-turn conversational history with session isolation (HMAC-signed session management).
* **Cross-Platform Access**: Full REST API (`/api/v1`), interactive web frontend dashboard, and Slack Bolt bot integration.

---

## Architecture Overview

```text
User Question / API / Slack Request
                │
                ▼
      [FastAPI Routes: /api & /api/v1]
  - Security safeguards & rate limiting (app/core/security.py)
  - Validation (Query length, Top-K bounds, File isolation)
                │
                ▼
      [Agent Orchestrator (app/services/agent_orchestrator.py)]
  - Contextualization & conversational follow-ups
  - Intent routing (Metadata, Profile, Navigation, Evidence)
  - Deterministic tool execution branch
                │
                ▼
      [Hybrid Retrieval (app/services/retrieval_service.py)]
  - Dense semantic search + BM25 keyword search
  - Reciprocal Rank Fusion (RRF)
  - Source: Neon PostgreSQL (pgvector) / In-memory Qdrant cache
                │
                ▼
      [Grounded Legal Reasoning (app/services/legal_reasoning.py)]
  - Strict evidence verification & confidence thresholding
  - Formatted answer with exact page, clause & snippet citations
```

---

## Tech Stack & Dependencies

* **Runtime**: Python 3.12 / 3.14 compatible
* **Web Framework**: FastAPI, Uvicorn (ASGI)
* **Database & Persistence**: SQLAlchemy 2.x, `psycopg` (psycopg 3 binary driver), `pgvector`, SQLite (local fallback)
* **Cloud Storage**: Boto3 (S3-compatible / Supabase Storage)
* **Search & Vectors**: `pgvector`, `qdrant-client`, `rank-bm25`, `fastembed`
* **Parsing & Extraction**: PyMuPDF (`pymupdf`), Pillow
* **LLM & Embeddings**: Groq API, `google-genai`
* **Messaging & Integrations**: `slack-bolt`, `slack-sdk`
* **Testing**: Pytest, Pytest-Asyncio, Pytest-Cov

---

## Project Structure

```text
.
├── app/
│   ├── api/             # FastAPI endpoints (/api and versioned /api/v1)
│   ├── core/            # Configuration, logging, security, telemetry
│   ├── integrations/    # Slack Bolt adapter and service coordinators
│   ├── models/          # Pydantic schemas and domain models
│   ├── persistence/     # SQLAlchemy models, Neon pgvector repository, DB manager
│   ├── providers/       # LLM and RAG provider interfaces and adapters
│   ├── services/        # Orchestrator, chunking, retrieval, citation, reasoning
│   └── main.py          # Application entry point and lifespan lifecycle
├── data/
│   ├── uploads/         # Local upload directory (when STORAGE_BACKEND=local)
│   └── vector_store/    # Local vector fallback directory
├── docs/                # Architecture, PRD, and development guides
├── frontend/            # Vanilla JS/CSS web dashboard
├── scripts/             # Integration tests and synthetic contract generator
├── tests/               # Unit, integration, and evaluation benchmark tests
├── Dockerfile           # Multi-stage production container build
├── requirements.txt     # Python production runtime dependencies
└── README.md
```

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/kstomar/legal-document-qa-chatbot.git
cd legal-document-qa-chatbot
```

### 2. Set up virtual environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

### 3. Configure environment variables

Create a `.env` file from `.env.example`:

```bash
cp .env.example .env
```

Key environment configurations:

```ini
# Application
APP_ENV=development
PORT=8000
HOST=0.0.0.0

# Database (PostgreSQL / Neon)
DATABASE_URL=postgresql://user:password@host/neondb?sslmode=require

# Storage Backend ('local' or 'supabase')
STORAGE_BACKEND=supabase
SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_SERVICE_KEY=your-supabase-service-role-key
SUPABASE_BUCKET_NAME=legal-documents

# LLM & Embeddings
LLM_PROVIDER=groq
GROQ_API_KEY=your-groq-api-key
LLM_MODEL=llama-3.3-70b-versatile

EMBEDDING_PROVIDER=gemini
GEMINI_API_KEY=your-gemini-api-key
EMBEDDING_MODEL=text-embedding-004

# Slack Integration (Optional)
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_APP_TOKEN=xapp-your-app-token
```

### 4. Run the application

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Open your browser at `http://localhost:8000` to interact with the web dashboard.

---

## API Usage

### Health Check

```bash
curl http://localhost:8000/api/health
```

### Upload a Document

```bash
curl -X POST "http://localhost:8000/api/documents/upload" \
  -F "file=@documents/synthetic_contract.pdf"
```

### Ask a Grounded Question

```bash
curl -X POST "http://localhost:8000/api/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "document_id": "<DOCUMENT_ID>",
    "question": "What is the governing law and jurisdiction of this agreement?",
    "top_k": 4
  }'
```

---

## Testing & Verification

Run the test suite locally:

```bash
# Focused unit tests
PYTHONPATH=. pytest tests/unit/test_persistence_and_storage.py tests/unit/test_security_and_hardening.py -v

# Full unit test suite
PYTHONPATH=. pytest tests/unit/ -v

# Offline RAG evaluation benchmark
PYTHONPATH=. python tests/evaluation/run_benchmark.py
```

---

## Production Deployment

### Render Deployment

The service is configured for direct deployment on Render:

* **Build Command**: `pip install -r requirements.txt`
* **Start Command**: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
* **Environment**: Ensure `DATABASE_URL` (Neon PostgreSQL), `GROQ_API_KEY`, `GEMINI_API_KEY`, and storage credentials are configured in the Render Dashboard.

### Docker Deployment

```bash
docker build -t legal-document-qa-chatbot:latest .
docker run -p 8000:8000 --env-file .env legal-document-qa-chatbot:latest
```
