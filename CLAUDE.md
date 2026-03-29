# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SQLBot is a ChatBI (conversational BI) system built on LLMs and RAG. It converts natural language questions into SQL queries, executes them against user-configured data sources, and returns results with visualizations. Developed by the DataEase open-source team.

## Build & Development Commands

### Backend (Python / FastAPI)
```bash
cd backend
uv sync                                    # Install dependencies
uv run python main.py                      # Run dev server (port 8000)
uv run pytest                              # Run all tests
uv run pytest tests/test_foo.py            # Run a single test file
uv run pytest tests/test_foo.py::test_bar  # Run a single test
uv run ruff check .                        # Lint
uv run ruff format .                       # Format
uv run mypy .                              # Type check
```

Requires Python 3.11, PostgreSQL (with pgvector), and optionally Redis for caching.
Optional deps: `uv sync --extra cpu` or `uv sync --extra cu128` for PyTorch embeddings.

### Frontend (Vue 3 / Vite)
```bash
cd frontend
npm install         # Install dependencies
npm run dev         # Dev server with type-checking (port 5173)
npm run build       # Production build
npm run lint        # ESLint with auto-fix
```

### G2-SSR (Chart server-side rendering)
Node.js service using AntV G2 + node-canvas, managed by PM2. Located in `g2-ssr/`.

### Docker
```bash
docker compose up -d    # Full stack (ports 8000 API, 8001 MCP, 5432 PostgreSQL)
```

Default credentials: admin / SQLBot@123456

## Architecture

### Multi-Service Layout
- **`backend/`** — FastAPI application (Python 3.11)
- **`frontend/`** — Vue 3 SPA (TypeScript + Vite)
- **`g2-ssr/`** — Chart rendering micro-service (Node.js)
- **`installer/`** — Deployment scripts (install.sh, sctl CLI)

### Backend Structure (`backend/`)

Entry point: `main.py` — creates the FastAPI app, registers middleware (CORS, auth token, response), runs Alembic migrations on startup, initializes embedding data.

**`apps/`** — Domain modules, each following the pattern: `api.py` (routes) + `crud/` (DB ops) + `schemas/` (Pydantic models) + `models/` (SQLModel/SQLAlchemy models):

| Module | Purpose |
|--------|---------|
| `chat/` | Core Q&A: NL question → SQL → execution → answer |
| `datasource/` | Data source management (13 types: MySQL, PostgreSQL, Oracle, SQL Server, ClickHouse, Hive, Doris, StarRocks, Redshift, Kingbase, DM, Elasticsearch, Excel) |
| `ai_model/` | LLM provider configuration (OpenAI-compatible APIs) |
| `dashboard/` | Dashboard/visualization assembly |
| `data_training/` | Training data for improving SQL generation accuracy |
| `terminology/` | Domain-specific terminology for RAG |
| `template/` | SQL templates |
| `mcp/` | Model Context Protocol server (port 8001) |
| `settings/` | System settings |
| `system/` | Users, workspaces, auth, audit logs, assistants |
| `swagger/` | API docs with i18n (zh/en) |

**`common/`** — Shared infrastructure:
- `common/core/` — Config (`config.py` reads env vars + `.env`), DB session, security (JWT), pagination, caching (Redis/memory)
- `common/utils/` — AES encryption, embedding helpers, Excel I/O, HTTP utilities, locale

**Database**: PostgreSQL with pgvector. Migrations via Alembic (`backend/alembic/versions/`).

**AI Pipeline**: LangChain + LangGraph for orchestrating LLM calls. Sentence Transformers for local embeddings. RAG is used to match terminology and training examples to improve Text-to-SQL accuracy.

### Datasource Connection Architecture (`backend/apps/db/`)

The system supports two connection strategies, defined by `ConnectType` in `constant.py`:

1. **`ConnectType.sqlalchemy`** — MySQL, PostgreSQL, Oracle, SQL Server, ClickHouse. Uses `create_engine()` with a URI built in `db.py:get_uri()`. Metadata queries use parameterized SQL via SQLAlchemy `text()`.
2. **`ConnectType.py_driver`** — Hive, Doris, StarRocks, Redshift, Kingbase, DM, Elasticsearch. Uses native Python drivers directly. Each type has its own `if/elif` branch in `db.py` for `check_connection`, `get_schema`, `get_tables`, `get_fields`, and `exec_sql`.

Key files:
- `constant.py` — `DB` enum defining type identifier, quoting chars, connection type, template name, illegal JDBC params
- `db.py` — Connection management, SQL execution, metadata extraction for all database types
- `db_sql.py` — Database-specific SQL for version queries, table listing (`get_table_sql`), column listing (`get_field_sql`)

Datasource credentials are AES-encrypted before storage (`apps/datasource/utils/utils.py`, static key `SQLBot1234567890`). Configuration is stored in `CoreDatasource.configuration` as an encrypted JSON string of `DatasourceConf` fields.

### SQL Dialect Template System (`backend/templates/sql_examples/`)

Each database type has a YAML template (e.g., `MySQL.yaml`, `Apache_Hive.yaml`) that guides LLM SQL generation. Templates include:
- `quot_rule` — Identifier quoting style (backticks, double quotes, square brackets)
- `limit_rule` — Pagination syntax (LIMIT, ROWNUM, FETCH FIRST)
- `other_rule` — Database-specific constraints (keyword conflicts, string concatenation, etc.)
- `basic_example` — Good/bad SQL examples for the LLM
- `example_answer_*` — Sample JSON responses for few-shot prompting

SQL read-safety is enforced by `check_sql_read()` in `db.py`, which uses `sqlglot` with database-specific dialects to block INSERT/UPDATE/DELETE/CREATE/DROP/ALTER.

### Frontend Structure (`frontend/`)

Vue 3 + TypeScript + Vite. Uses hash-based routing (`createWebHashHistory`).

- **`src/views/`** — Page components organized by feature: `chat/`, `dashboard/`, `ds/` (datasource), `system/` (admin), `embedded/`
- **`src/api/`** — Axios-based API clients matching backend modules
- **`src/stores/`** — Pinia stores (user, appearance, assistant, chatConfig, dashboard)
- **`src/components/layout/`** — App shell layouts (LayoutDsl, SinglePage, Menu)
- **`src/router/`** — Route definitions in `index.ts`, dynamic route loading in `dynamic.ts`
- **`src/i18n/`** — Internationalization (Chinese primary)

UI framework: Element Plus with auto-import. Charts: AntV G2 (interactive), S2 (pivot tables), X6 (diagrams). Rich text: TinyMCE. Markdown rendering: markdown-it.

Key frontend routes: `/chat` (main Q&A), `/dashboard` (dashboards), `/canvas` (dashboard editor), `/ds/:dsId/:dsName` (table explorer), `/system/*` (admin).

### Configuration

- Backend config: `backend/common/core/config.py` — reads from environment variables and `../.env`
- Frontend env: `.env.development` / `.env.production` in `frontend/`
- Key env vars: `POSTGRES_SERVER`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `SECRET_KEY`, `BACKEND_CORS_ORIGINS`, `LOG_LEVEL`, `CACHE_TYPE` (redis/memory/None)

### MCP Server

Separate FastAPI app (`mcp_app` in `main.py`) using `fastapi-mcp` to expose SQLBot operations (data source listing, model listing, question answering, workspace management) as MCP tools on port 8001.

## Conventions

- Python backend uses `uv` as package manager, not pip or poetry
- Frontend uses npm (not pnpm/yarn)
- Backend linting: ruff with rules E, W, F, I, B, C4, UP, ARG001
- Database migrations are auto-run on app startup via Alembic
- The `sqlbot-xpack` package (from testpypi) provides enterprise/extended features
- The project is primarily documented in Chinese; code comments and UI strings are Chinese-first
- When adding a new database type, register it in `constant.py` DB enum, add branches in `db.py` and `db_sql.py`, create a YAML template in `templates/sql_examples/`, add the type to `ds-type.ts` (frontend), and add a PNG icon to `frontend/src/assets/datasource/`
- The `common.utils.utils:equals_ignore_case()` helper is used throughout for case-insensitive string comparison instead of direct string matching
