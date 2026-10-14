# IT Support Agentic RAG Copilot

An IT support assistant that answers employee questions from a private company knowledge base, falls back to web search only when internal evidence is weak, and shows every decision it makes.

## Architecture

```text
                   ┌─────────────────────┐
                   │   Employee / User   │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ HTML/CSS/JavaScript │
                   │      Web UI         │
                   └──────────┬──────────┘
                              │ POST /api/chat
                              ▼
                   ┌─────────────────────┐
                   │       FastAPI       │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │     LangGraph       │
                   │ Agentic RAG Control │
                   └──────────┬──────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
       ┌───────────────┐              ┌──────────────┐
       │ Private KB    │              │ Tavily Web   │
       │ Pinecone      │              │ Search       │
       └───────┬───────┘              └──────┬───────┘
               │                             │
               └──────────────┬──────────────┘
                              ▼
                     ┌────────────────┐
                     │ OpenAI LLM     │
                     │ Grounded Answer│
                     └────────────────┘
```

## Agentic RAG Workflow

```text
Question
   ↓
[1] Route Question
   ├── Greeting / simple chat ─────────────→ Direct Answer
   │
   └── IT support question
                ↓
[2] Retrieve from Private Pinecone KB
                ↓
[3] Grade Private Evidence
       ┌────────┴────────┐
       │                 │
     GOOD               WEAK
       │                 │
       ▼                 ▼
Generate from KB    [4] Tavily Web Search
                         ↓
                  [5] Grade Web Evidence
                    ┌────┴─────┐
                    │          │
                  GOOD        WEAK
                    │          │
                    ▼          ▼
              Generate Web  [6] Rewrite Query
                               ↓
                    Retry from [2] (max 1 retry)
                               ↓
              Still weak after retry → Insufficient Evidence
```

```text
Simple RAG:   Question → Retrieve → Generate
This project: Question → Route → Retrieve → Grade → KB / Web / Rewrite & retry → Grounded answer
```

Each step is a LangGraph node. The decision trace is returned with every answer, shown in the UI, and logged to SQLite.

## Tech Stack

| Layer | Technology |
|---|---|
| Agent workflow | LangGraph |
| LLM | OpenAI `gpt-4o-mini` |
| Embeddings | OpenAI `text-embedding-3-small` |
| Vector DB | Pinecone |
| Web search | Tavily |
| API | FastAPI |
| Frontend | HTML, CSS, JavaScript |
| Audit log | SQLite |
| Packaging | Docker |

## Setup

Requires Python 3.10+ and API keys for OpenAI, Pinecone and Tavily.

```bash
python -m venv venv
venv\Scripts\activate            # macOS/Linux: source venv/bin/activate
pip install -r requirements.txt
```

Create a `.env` file in the project root. `ADMIN_API_KEY` protects document uploads.

```env
OPENAI_API_KEY=...
PINECONE_API_KEY=...
TAVILY_API_KEY=...
ADMIN_API_KEY=...
```

Optional: `OPENAI_MODEL`, `EMBEDDING_MODEL`, `PINECONE_INDEX_NAME`, `PINECONE_NAMESPACE`, `TOP_K` (default 4), `MAX_RETRIES` (default 1).

Index the sample knowledge base (run once; it creates the Pinecone index if needed), then start the app:

```bash
python ingest_sample_kb.py
python run.py
```

Open http://127.0.0.1:8080. API docs are at `/docs`.

## Document Ingestion

The sample-KB script and the UI upload (sidebar → **Add Company Document**) use the same pipeline:

```text
PDF / TXT / MD / DOCX → Load → Chunk (900 chars, 120 overlap) → OpenAI embeddings → Pinecone
```

## Example Questions

| Question | Path |
|---|---|
| How do I connect to the company VPN? | Private KB |
| Can IT support ask me to share my MFA code? | Private KB |
| What is the latest Microsoft Teams outage guidance? | KB weak → web search |
| Hello! | Direct answer |

## API

| Endpoint | Description |
|---|---|
| `GET /api/health` | Health check |
| `POST /api/chat` | `{"question": "..."}` → `answer`, `source_used`, `citations`, `trace`, `rewritten_query` |
| `POST /api/ingest` | Multipart upload (`file`) with `X-Admin-Key` header; indexes the document into Pinecone |

## Project Structure

```text
app/
├── api/routes.py         REST endpoints
├── core/config.py        settings loaded from .env
├── rag/workflow.py       LangGraph agent
├── rag/state.py          shared agent state
├── rag/vectorstore.py    Pinecone + embeddings
└── services/             document ingestion, audit log
templates/, static/       web UI
data/sample_kb/           sample IT documents
Agentic-RAG-demo/         notebook version of the workflow
```

## License

Apache License 2.0. See [LICENSE](LICENSE).
