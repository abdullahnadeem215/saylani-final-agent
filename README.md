# SMIT Multi-Agent Workspace

A local AI workspace for chatting with documents, searching GitHub, working with Google Calendar, and drafting or scheduling email. The backend is a FastAPI service that routes requests through a LangGraph supervisor. The frontend is a React + Vite app with a document-aware chat, MCP integrations, Human-in-the-Loop confirmations, and a persistent scheduled-email view.

## What It Does

- **Document RAG**: upload PDFs, index them in Supabase PostgreSQL with pgvector, and ask page-aware questions.
- **Universal agent chat**: route a request to the RAG, GitHub, Calendar, or Email agent through one endpoint.
- **GitHub explorer**: search repositories, code, commits, and issues through the GitHub MCP integration.
- **Calendar tools**: list events, detect conflicts, and create events using Google Calendar or the local fallback store.
- **Email tools**: draft email, require confirmation before sending, and persist scheduled jobs with APScheduler and SQLite.
- **Frontend workspace**: switch between Chat, Documents, GitHub, Agents, and Settings without losing document-specific chat history.

## Architecture

```text
React + Vite (frontend/:5173)
          |
          | /api proxy
          v
FastAPI (backend/:8000)
          |
          v
LangGraph supervisor
   |       |        |        |
  RAG   GitHub  Calendar  Email
   |       |        |        |
Supabase  MCP     Google    SMTP/IMAP
pgvector  stdio   Calendar  APScheduler + SQLite
```

## Repository Layout

```text
.
├── agents/              # RAG, GitHub, Calendar, and Email workers
├── backend/             # FastAPI app and API routers
├── frontend/            # React + Vite workspace UI
├── mcp_tools/           # MCP clients and local FastMCP servers
├── rag/                 # PDF ingestion, embeddings, and retrieval
├── data/                # Sample knowledge content
├── supervisor.py        # LangGraph routing and orchestration
├── state.py             # Shared agent state
├── config.py            # Environment-backed configuration
├── requirements.txt     # Python dependencies
├── test_*.py            # Module and end-to-end tests
├── run_backend.bat      # Windows backend launcher
└── run_frontend.bat     # Windows frontend launcher
```

## Requirements

- Python 3.11 or newer
- Node.js 18 or newer and npm
- A Supabase project with the `vector` extension for document RAG
- A Gemini or OpenAI API key for model calls
- A GitHub token for GitHub MCP searches
- Google Calendar credentials when using live Calendar integration
- An SMTP/IMAP account when using live email integration

The application can still start without every integration configured. The Settings view and `/api/health` endpoint expose the current configuration state; Calendar and Email also contain local development fallbacks where supported.

## Configuration

Create a `.env` file in the repository root. Keep it out of source control.

```env
GEMINI_API_KEY=your_gemini_key
# OPENAI_API_KEY=optional_openai_key

SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your_supabase_key
# SUPABASE_SERVICE_ROLE_KEY=optional_service_role_key
SUPABASE_DB_URL=postgresql://...

GITHUB_PERSONAL_ACCESS_TOKEN=your_github_token

# Google Calendar
GOOGLE_CALENDAR_CREDENTIALS_PATH=credentials.json
GOOGLE_CALENDAR_TOKEN_PATH=token.json
TIMEZONE=Asia/Karachi

# Email, if enabled
EMAIL_ADDRESS=you@example.com
EMAIL_APP_PASSWORD=your_app_password
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_SMTP_PORT=587
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_IMAP_PORT=993
```

Optional RAG settings include `EMBEDDING_MODEL`, `EMBEDDING_DIMENSION` (default `768`), `CHUNK_SIZE` (default `1000`), `CHUNK_OVERLAP` (default `200`), `TOP_K` (default `5`), `MIN_SIMILARITY` (default `0.3`), and `MAX_UPLOAD_MB` (default `20`).

Before using document search, apply [`rag/schema.sql`](rag/schema.sql) to Supabase. It creates the document/chunk storage and retrieval function expected by the ingestion pipeline.

## Run Locally on Windows

### 1. Create the Python environment

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

### 2. Start the backend

From the repository root:

```powershell
.\.venv\Scripts\Activate.ps1
uvicorn backend.main:app --reload --port 8000
```

Or run `run_backend.bat`.

- API: <http://127.0.0.1:8000>
- Swagger docs: <http://127.0.0.1:8000/docs>
- Health: <http://127.0.0.1:8000/api/health>

### 3. Start the frontend

```powershell
cd frontend
npm install
npm run dev
```

Or run `run_frontend.bat` from the repository root. Open <http://localhost:5173>. Vite proxies `/api` requests to the FastAPI server on port 8000.

For a production frontend build:

```powershell
cd frontend
npm run build
```

## API Surface

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/health` | Report service and provider configuration |
| `POST` | `/api/agent` | Route a prompt through the supervisor |
| `POST` | `/api/upload` | Index a PDF into the RAG store |
| `GET` | `/api/documents` | List indexed documents |
| `DELETE` | `/api/documents/{document_id}` | Delete one document |
| `DELETE` | `/api/documents/all` | Delete all documents and chunks |
| `POST` | `/api/chat` | Direct document chat endpoint |
| `POST` | `/api/github/search` | Search through the GitHub agent |
| `GET` | `/api/scheduled-emails` | List persisted email jobs |
| `DELETE` | `/api/scheduled-emails/{job_id}` | Cancel an email job |

`POST /api/agent` accepts `query` (or `message`/`question`), an optional `document_id`, conversation `history`, and `confirmed` plus `pending_action` for write-action confirmation.

## Testing

Run the focused module tests or the complete suite from the repository root:

```powershell
.\.venv\Scripts\Activate.ps1
python -m pytest -q
```

The end-to-end suite may require live provider credentials and running services. Tests that use Supabase, GitHub MCP, Calendar, or Email should be run with the corresponding `.env` configuration available.

## Security Notes

- Never commit `.env`, OAuth tokens, service-role keys, app passwords, or generated local database files.
- Keep GitHub and email write operations behind the frontend confirmation flow.
- Use a dedicated email account or sandbox outbox during development.
- Restrict Supabase service-role credentials to the backend only.
