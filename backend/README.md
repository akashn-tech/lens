# XYMP Lens Backend

> High-performance, asynchronous AI data intelligence service transforming natural language questions into safe, read-only SQL queries and synthesized analytical insights across PostgreSQL, MongoDB, and Excel (DuckDB).

---

## 📖 Overview

**XYMP Lens Backend** is an API-first FastAPI service built to bridge business inquiries and complex analytical databases. It acts as an intelligent data layer that enables non-technical and technical users alike to query heterogeneous data sources using conversational English without compromising database security or exposing raw database credentials to LLMs.

### The Core Problem
Traditional Text-to-SQL solutions suffer from major drawbacks:
1. **Security Vulnerabilities**: Providing language models direct execution tools or database connection strings creates SQL-injection and destructive mutation risks (`DROP`, `DELETE`, `UPDATE`).
2. **Schema Hallucinations**: Large database schemas exceed LLM context windows or cause hallucinated joins and incorrect column mappings.
3. **Engine Heterogeneity**: Teams maintain data across relational databases (PostgreSQL), document stores (MongoDB), and spreadsheet exports (Excel `.xlsx`), requiring fragmented tooling.
4. **Lack of Conversational Context**: Isolated queries fail when users ask follow-up questions referencing previous calculations or pronouns ("how did that change last quarter?", "what about his department?").

### The Lens Philosophy
- **Zero-Access Generation**: The LLM *never* receives direct database access or execution tools. Query generation operates exclusively against an isolated schema catalog stored in Supabase.
- **Two-Tier Storage Architecture**: Internal app state (projects, schema metadata, conversation memory, learning rules) is isolated in Supabase via an `asyncpg` connection pool, while analytical queries execute against ephemeral, short-lived customer database connections or local DuckDB files.
- **Human-in-the-Loop Safeguards**: Queries are proposed for inspection before execution, validated through an AST and keyword safety filter, and executed with strict timeout and row limits.
- **Multi-Engine Execution**: Unified querying experience across PostgreSQL, MongoDB schema exploration, and high-performance Excel-to-DuckDB streaming ingestion.

---

## ✨ Core Features

* **Natural Language to SQL Generation**: Schema-aware prompt engineering powered by OpenAI and Agno, generating dialect-accurate SQL.
* **Two-Phase Interactive Analysis**:
  * **Phase 1 (`/start`)**: Proposes validated SQL for user confirmation with conversational pronoun resolution.
  * **Phase 2 (`/run`)**: Executes confirmed SQL, handles automated multi-query recovery if initial results are empty, and synthesizes key takeaway answers with bullet points.
* **Multi-Engine Data Support**:
  * **PostgreSQL**: Native asynchronous schema extraction and query execution via `asyncpg`.
  * **MongoDB**: Atlas and standalone metadata inspection and document sample previews.
  * **Excel / XLSX**: Chunk-based streaming ingest into local embedded **DuckDB** databases with automatic relationship and datatype inference.
* **Token-Efficient Conversational Memory**: Multi-turn chat persistence in `playground_messages` injecting slim `Question + SQL + Answer` context to resolve pronouns without context bloat.
* **Dynamic Self-Learning System**: Correct and incorrect user feedback on generated SQL is compiled into project-specific `confirmed_learnings` and automatically injected into subsequent generation prompts.
* **Strict Security Guardrails**:
  * Enforces read-only statements (`SELECT`, `WITH`).
  * Blocks write operations (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, `EXEC`).
  * Prevents semicolon statement chaining.
  * Enforces 30-second statement timeouts and response row caps.
* **Real-Time Progress Streaming**: Server-Sent Events (SSE) stream live status updates (`executing`, `synthesizing`, `complete`) directly to the frontend.

---

## 🧩 Key Components

```
backend/
├── app/
│   ├── main.py                  # FastAPI application entry point, lifespan, CORS, and routing
│   ├── config.py                # Environment configuration, DSN normalization, Pydantic settings
│   ├── api/                     # REST API controllers
│   │   ├── projects.py          # Project CRUD, XLSX upload, preview, and health checks
│   │   ├── analysis_routes.py   # Playground two-phase analysis endpoints (/start & /run with SSE)
│   │   ├── conversation_routes.py # Playground chat thread history and message retrieval
│   │   └── sql_routes.py        # Standalone SQL generation and execution endpoints
│   ├── agent/                   # AI and prompt orchestration layer
│   │   ├── analysis_agent.py    # Two-phase analysis agent, conversation formatting, answer synthesis
│   │   ├── sql_generator.py     # Standalone NL-to-SQL generation agent
│   │   ├── introspect_schema.py # Batch schema catalog reader tool for Agno agents
│   │   └── agno_db.py           # Agno session telemetry and connection manager
│   ├── db/                      # Database connectors, validation, and storage repositories
│   │   ├── app_db.py            # Global asyncpg connection pool management for Supabase
│   │   ├── query_runner.py      # Multi-engine execution router (Postgres vs DuckDB)
│   │   ├── sql_validation.py    # Read-only AST and keyword SQL safety validator
│   │   ├── conversations.py     # Chat thread and message persistence repository
│   │   ├── learnings.py         # Query attempts and confirmed learnings repository
│   │   └── connectors/          # Ephemeral database connectors (Postgres, Mongo, DuckDB)
│   ├── storage/                 # File storage and streaming ingestion
│   │   ├── xlsx_ingest.py       # Chunked streaming Excel ingest into DuckDB
│   │   ├── xlsx_storage.py      # Supabase Storage client for raw workbook persistence
│   │   └── duckdb_file.py       # Local DuckDB connection and lifecycle manager
│   └── schemas/                 # Pydantic data transfer objects (DTOs)
│       ├── analysis.py          # Start/Run request & response schemas
│       ├── conversation.py      # Chat thread and message models
│       ├── project.py           # Project creation, preview, and metadata models
│       └── sql.py               # SQL generation and execution schemas
└── supabase/
    └── migrations/              # Plain SQL migrations applied via Supabase CLI
```

---

## 🛠️ Local Development Setup

### Prerequisites
* **Python 3.11+**
* **Supabase CLI** (installed and authenticated)
* **Node.js 18+** (for frontend integration)
* **OpenAI API Key**

### 1. Clone & Navigate
```bash
cd c:/smartproject/lens/backend
```

### 2. Create Virtual Environment & Install Dependencies
```bash
# Create virtual environment
python -m venv .venv

# Activate virtual environment
# Windows (PowerShell):
.venv\Scripts\Activate.ps1
# Windows (cmd):
.venv\Scripts\activate.bat
# Linux / macOS:
source .venv/bin/activate

# Install requirements
pip install -r requirements.txt
```

### 3. Configure Environment Variables
Copy `.env.example` to `.env`:
```bash
cp .env.example .env
```
Fill in your `SUPABASE_DB_URL`, `OPENAI_API_KEY`, and Supabase storage credentials (see [Configuration](#-configuration) below).

### 4. Database Setup & Migrations
Apply the plain SQL database migrations to your Supabase project:
```bash
# Option A: Link your remote Supabase project and push
supabase login
supabase link --project-ref YOUR_PROJECT_REF
supabase db push

# Option B: Push directly using connection string from .env
supabase db push --db-url "YOUR_SUPABASE_DB_URL"
```

### 5. Start the Development Server
```bash
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

### 6. Verify Health & OpenAPI Docs
* **Health Check**: `http://127.0.0.1:8000/health` → `{"status": "ok"}`
* **Interactive API Documentation (Swagger)**: `http://127.0.0.1:8000/docs`
* **Alternative API Documentation (ReDoc)**: `http://127.0.0.1:8000/redoc`

---

## ⚙️ Configuration

All configuration is managed via environment variables defined in `.env` and validated through Pydantic's `BaseSettings` in `app/config.py`.

| Variable | Required | Default | Description |
| :--- | :---: | :---: | :--- |
| `APP_NAME` | No | `xymp_lens_backend` | Identifier displayed in logs and OpenAPI docs. |
| `SUPABASE_DB_URL` | **Yes** | — | Supabase PostgreSQL connection string. **Use the Session/Transaction Pooler URI** to avoid IPv6 issues on Windows. |
| `SUPABASE_URL` | For XLSX | `""` | Supabase Project REST URL (e.g. `https://xxx.supabase.co`). |
| `SUPABASE_SERVICE_ROLE_KEY`| For XLSX | `""` | Supabase Service Role Key for workbook storage access. |
| `SUPABASE_STORAGE_BUCKET` | No | `xlsx-projects` | Storage bucket name for uploaded Excel files. |
| `OPENAI_API_KEY` | **Yes** | `""` | OpenAI API key for Agno SQL generation & synthesis agents. |
| `MODEL_ID` | No | `gpt-4o` | Primary model for SQL generation and analysis synthesis. |
| `RELATIONSHIP_VERIFY_MODEL_ID`| No | `gpt-4o-mini` | Fast model used to infer and verify foreign key relationships in Excel sheets. |
| `XLSX_INGEST_CHUNK_SIZE` | No | `3000` | Number of Excel rows read per chunk during streaming DuckDB ingestion. |
| `DUCKDB_MEMORY_LIMIT` | No | `1GB` | Maximum RAM threshold allocated to DuckDB query execution. |
| `DUCKDB_THREADS` | No | `4` | Maximum parallel worker threads for DuckDB execution. |

> [!TIP]
> **Windows Pooler Notice**: Direct connections to `db.[REF].supabase.co:5432` frequently fail on Windows due to IPv6 routing. Always use the Supabase connection pooler URI format:
> `postgresql://postgres.[REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres`

---

## 🏛️ Architecture & Data Flow

```mermaid
flowchart TD
    subgraph Client["Frontend (React / Vite :8080)"]
        UI["Playground & Schema UI"]
    end

    subgraph Backend["Lens Backend (FastAPI :8000)"]
        Router["API Routers\n(/analysis, /projects, /sql)"]
        Validator["SQL Safety Validator\n(Read-Only AST & Keyword Filter)"]
        
        subgraph AgentLayer["Agent & Reasoning Layer"]
            AnalysisAgent["Analysis Agent\n(Agno + OpenAI)"]
            SQLGen["SQL Generator"]
            PromptContext["Context Injector\n(Learnings + Slim Chat Turns)"]
        end

        subgraph StorageLayer["Storage & Ingest Pipeline"]
            DuckDBEngine["Local DuckDB Engine\n(Embedded File DB)"]
            XLSXIngest["Streaming XLSX Ingest\n(Chunked Processing)"]
        end
    end

    subgraph AppDB["App Database (Supabase)"]
        Catalog["table_schema_catalog"]
        Learnings["confirmed_learnings\n& query_attempts"]
        Conversations["playground_conversations\n& playground_messages"]
        SupaStorage["Supabase Storage\n(xlsx-projects bucket)"]
    end

    subgraph ExternalData["Customer Data Sources"]
        CustomerPG["Customer PostgreSQL"]
        CustomerMongo["Customer MongoDB Atlas"]
    end

    UI -->|HTTP / SSE| Router
    Router --> AgentLayer
    AgentLayer -->|Read Schema & Learnings| Catalog & Learnings
    AgentLayer -->|Propose SQL| Router
    Router --> Validator
    Validator -->|Execute Validated SQL| CustomerPG
    Validator -->|Execute Validated SQL| DuckDBEngine
    XLSXIngest -->|Stream Chunks| DuckDBEngine
    XLSXIngest -->|Upload File| SupaStorage
    XLSXIngest -->|Sync Metadata| Catalog
    Router -->|Persist Chat & Messages| Conversations
```

### Two-Phase Interactive Analysis Flow

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Frontend
    participant API as FastAPI Backend
    participant Agent as Agno Analysis Agent
    participant Catalog as Schema Catalog (Supabase)
    participant DB as Target DB (Postgres / DuckDB)
    participant Store as Chat Store (Supabase)

    Note over User,Store: Phase 1: Propose SQL (/analysis/start)
    User->>API: POST /analysis/start { question, conversation_id }
    API->>Store: Fetch recent 5 turns (Question + SQL + Answer)
    API->>Catalog: Fetch relevant table schemas & confirmed learnings
    API->>Agent: Prompt with schema + learnings + slim history
    Agent-->>API: Propose validated SQL query
    API-->>User: Return proposed_sql & analysis_id (Confirm Dialog)

    Note over User,Store: Phase 2: Execute & Synthesize (/analysis/{id}/run)
    User->>API: POST /analysis/{id}/run?stream=1&conversation_id=...
    API-->>User: SSE: event: progress {"stage": "executing"}
    API->>DB: Execute validated read-only SQL
    DB-->>API: Return query rows & columns
    API-->>User: SSE: event: progress {"stage": "synthesizing"}
    API->>Agent: Synthesize takeaway + bullet points from result preview
    Agent-->>API: Final structured answer
    API->>Store: Persist turn in playground_messages (Q + SQL + Answer)
    API-->>User: SSE: event: complete { answer, queries_used }
```

---

## 📡 API Endpoints Reference

### 1. Projects Management

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/projects` | Create a Postgres or MongoDB project, test external connection, and sync initial schema. |
| `POST` | `/projects/upload-xlsx` | Multipart upload: Ingest `.xlsx` into DuckDB, sync schema catalog, and store raw workbook in Supabase Storage. |
| `GET` | `/projects` | List all registered projects (passwords omitted). |
| `GET` | `/projects/{project_id}` | Retrieve specific project metadata and connection details. |
| `DELETE` | `/projects/{project_id}` | Delete project, drop cached schemas, and remove associated DuckDB files. |
| `GET` | `/projects/{project_id}/preview` | Fetch sample preview rows across project tables/collections. |
| `GET` | `/health` | Health check endpoint returning `{ "status": "ok" }`. |

### 2. Schema Catalog & Annotations

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/projects/{project_id}/schema` | Fetch stored schema catalog (tables, columns, types, descriptions). |
| `POST` | `/projects/{project_id}/schema/sync` | Re-scan external database/DuckDB and update schema catalog. |
| `PATCH` | `/projects/{project_id}/schema/{table_name}` | Update table-level description or business annotations. |
| `PATCH` | `/projects/{project_id}/schema/{table_name}/columns/{column_name}` | Update column-level description or semantic annotations. |

### 3. Playground Analysis

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/projects/{project_id}/analysis/start` | Initiates analysis. Proposes read-only SQL for confirmation and handles conversational pronoun resolution. |
| `POST` | `/projects/{project_id}/analysis/{analysis_id}/run` | Executes confirmed query on customer DB or DuckDB. Supports SSE streaming via `stream=1`. |

#### Example: Start Analysis (`POST /analysis/start`)
```json
// Request
{
  "question": "What were the top 5 selling products last month?",
  "conversation_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
}

// Response
{
  "analysis_id": "4e1a0b5c-6789-4ef0-bc12-9876543210ab",
  "attempt_id": "7f8e9d0a-1234-4567-89ab-cdef01234567",
  "conversation_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "proposed_sql": "SELECT product_name, SUM(quantity * unit_price) AS total_revenue FROM orders JOIN order_items ON orders.id = order_items.order_id WHERE order_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month') GROUP BY product_name ORDER BY total_revenue DESC LIMIT 5",
  "message": "Running this analysis requires executing queries against your data. Proceed?"
}
```

#### Example: Run Analysis (`POST /analysis/{id}/run?stream=1`)
```http
event: progress
data: {"stage": "executing", "message": "Running query 1…"}

event: progress
data: {"stage": "synthesizing", "message": "Writing answer…"}

event: complete
data: {
  "analysis_id": "4e1a0b5c-6789-4ef0-bc12-9876543210ab",
  "answer": "Top 5 products generated $124,500 in revenue last month:\n- Enterprise Plan led sales at $54,000 across 18 transactions.\n- Pro Tier followed with $32,500 in volume.\n- Starter Pack accounted for $18,000 across 360 units.",
  "queries_used": [
    {
      "attempt_id": "7f8e9d0a-1234-4567-89ab-cdef01234567",
      "sql": "SELECT product_name, ... LIMIT 5",
      "result_summary": {
        "columns": ["product_name", "total_revenue"],
        "row_count": 5,
        "rows": [{"product_name": "Enterprise Plan", "total_revenue": 54000}]
      }
    }
  ]
}
```

### 4. Playground Conversations

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/projects/{project_id}/conversations` | List conversation threads for the project sidebar. |
| `GET` | `/projects/{project_id}/conversations/{conversation_id}` | Fetch full conversation thread including Q, SQL, Answer, and `queries_used`. |
| `DELETE` | `/projects/{project_id}/conversations/{conversation_id}` | Delete a conversation thread and cascade-delete all messages (204 No Content). |

### 5. Learnings & Feedback

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `PATCH` | `/projects/{project_id}/attempts/{attempt_id}/feedback` | Record user feedback (`"correct"` \| `"incorrect"`). |
| `POST` | `/projects/{project_id}/attempts/{attempt_id}/confirm` | Save verified SQL as a permanent project learning rule (`confirmed_learnings`). |

### 6. Standalone SQL Engine

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/projects/{project_id}/sql/generate` | Generates SQL from a natural language question without creating an analysis session. |
| `POST` | `/projects/{project_id}/sql/execute` | Executes arbitrary read-only SQL against the project's target database or DuckDB. |

---

## 🖥️ Frontend Integration

The backend is configured to pair with the Vite + React frontend located in `frontend/`.

### Vite Development Proxy
The frontend development server (`http://localhost:8080`) automatically proxies API requests to the backend (`http://127.0.0.1:8000`) via `vite.config.ts`:

```typescript
server: {
  port: 8080,
  proxy: {
    "/projects": {
      target: "http://127.0.0.1:8000",
      changeOrigin: true,
      timeout: 300000,
    },
    "/health": {
      target: "http://127.0.0.1:8000",
      changeOrigin: true,
    },
  },
}
```

### Handling Real-Time SSE Streams in React
When calling `/projects/{project_id}/analysis/{analysis_id}/run?stream=1`, the client consumes the stream using standard `fetch` with `ReadableStream` reader or an EventSource parser:

```typescript
const response = await fetch(
  `/projects/${projectId}/analysis/${analysisId}/run?stream=1&conversation_id=${conversationId}`,
  {
    method: "POST",
    headers: { "Accept": "text/event-stream" },
  }
);

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  const chunk = decoder.decode(value);
  // Parse event: progress and event: complete payloads
}
```

### UI Tab Data Consumption
The response payload from `/run` or stored conversation messages populates all four playground tabs:
1. **Analysis Tab**: Renders `payload.answer` (executive takeaway + key bullet points).
2. **Data Tab**: Renders tabular grid from `payload.queries_used[0].result_summary.rows` and `columns`.
3. **Query Tab**: Displays the executable SQL query from `payload.queries_used[0].sql` with syntax highlighting and feedback controls.
4. **Graph Tab**: Automatically maps columns from `result_summary` into chart axes (Bar, Line, Pie) for instant visualization.

---

## 🔒 Security & Best Practices

1. **Read-Only Enforcement**: Every query executed through `/analysis/run` or `/sql/execute` passes through `app/db/sql_validation.py` to prevent data modification or schema alteration.
2. **Connection Isolation**: Customer database connections and DuckDB file handles are opened on-demand and closed immediately after query execution. They are never pooled with the app's Supabase connection pool.
3. **No Direct LLM Access**: LLMs generate queries strictly from metadata stored in `table_schema_catalog`. They never receive credentials, hostnames, or direct execution privileges.
4. **Token Cost Optimization**: Conversation memory is compressed into slim `Question + SQL + Answer` turns, avoiding redundant system prompt re-evaluations and saving up to 70% in token overhead per turn.
