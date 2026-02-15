# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Install dependencies
uv sync

# Run the app (from repo root)
./run.sh
# OR manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# App runs at http://localhost:8000, API docs at http://localhost:8000/docs
```

Requires a `.env` file in the repo root with `ANTHROPIC_API_KEY=...`.

**Always use `uv` to manage dependencies and run commands. Never use `pip` directly.** Use `uv add <package>` to add dependencies, `uv remove <package>` to remove them, and prefix all Python commands with `uv run`.

There are no tests or linting configured in this project.

## Architecture

This is a RAG chatbot that answers questions about educational course materials. It has a FastAPI backend serving a static HTML/JS/CSS frontend.

### Query Flow (two-call pattern to Anthropic API)

1. Frontend sends `POST /api/query` with `{query, session_id}` → `app.py`
2. `RAGSystem.query()` wraps the query in a prompt and calls `AIGenerator.generate_response()` with a `search_course_content` tool definition
3. **API Call #1**: Claude receives the query + tool. It either answers directly or calls the search tool
4. If tool is used: `ToolManager` routes to `CourseSearchTool` → `VectorStore.search()` queries ChromaDB
5. **API Call #2**: Tool results are sent back to Claude (without tools enabled) for final answer synthesis
6. Response + sources returned to frontend, conversation saved to `SessionManager`

### Key Design Decisions

- **Two ChromaDB collections**: `course_catalog` (one entry per course, for fuzzy name resolution) and `course_content` (chunked lesson text, for semantic search)
- **Tool-based RAG**: Instead of always searching, Claude decides via tool-use whether a search is needed. The tool supports optional `course_name` and `lesson_number` filters
- **Fuzzy course matching**: When a user references a course by partial name, `VectorStore._resolve_course_name()` does a semantic search against `course_catalog` to find the exact title
- **Session management**: In-memory, keyed by session_id. Keeps last 2 exchanges (4 messages). History is injected into the system prompt as plaintext
- **Document format**: Course docs in `docs/` follow a specific format: `Course Title:`, `Course Link:`, `Course Instructor:` headers, then `Lesson N: Title` markers with content. Parsed by `DocumentProcessor.process_course_document()`
- **Chunking**: Sentence-boundary splitting at 800 chars with 100 char overlap. First chunk per lesson gets a `"Lesson N content: "` prefix

### Backend Module Responsibilities

- `app.py` — FastAPI routes + static file serving (frontend mounted at `/`)
- `rag_system.py` — Orchestrator that wires all components together
- `ai_generator.py` — Anthropic API client; handles the two-call tool-use pattern
- `vector_store.py` — ChromaDB wrapper with search, course resolution, and metadata storage
- `document_processor.py` — Parses course document format, extracts metadata, chunks text
- `search_tools.py` — `Tool` ABC, `CourseSearchTool` implementation, and `ToolManager` registry
- `session_manager.py` — In-memory conversation history per session
- `models.py` — Pydantic models: `Course`, `Lesson`, `CourseChunk`
- `config.py` — Dataclass with settings loaded from env/defaults

### Frontend

Vanilla HTML/JS/CSS. Uses `marked.js` for markdown rendering. No build step. Served as static files by FastAPI.
