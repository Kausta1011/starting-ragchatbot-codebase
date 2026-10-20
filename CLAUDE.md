# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A RAG chatbot that answers questions about course transcripts in `docs/`. FastAPI backend (`backend/`), vanilla HTML/JS frontend (`frontend/`, served by the backend), ChromaDB vector store, Anthropic Claude with tool calling. Python 3.13, managed with `uv`.

## Commands

```bash
uv sync                                          # install dependencies
./run.sh                                         # start server (cd backend && uv run uvicorn app:app --reload --port 8000)
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

- Web UI: http://localhost:8000 — API docs: http://localhost:8000/docs
- Requires `ANTHROPIC_API_KEY` in `.env` at the repo root (see `.env.example`).
- The server must be started from `backend/`: paths are relative (`../docs`, `../frontend`, `./chroma_db`).
- There are no tests, linter, or formatter configured. `main.py` at the root is an unused stub.
- Always use `uv` to run Python and manage dependencies (`uv run ...`, `uv add ...`); don't use `pip` directly.

## Architecture

**Query path** (`POST /api/query`): `app.py` → `RAGSystem.query()` → `AIGenerator.generate_response()` makes Claude call #1 with the `search_course_content` tool. If Claude returns `tool_use`, `_handle_tool_execution` runs the tool via `ToolManager`, then makes call #2 **without tools** — so Claude gets exactly one search round. Sources for the UI are not returned by the AI; `CourseSearchTool` stores them in `last_sources` as a side effect, and `RAGSystem` reads and resets them via `ToolManager` after each query.

**Retrieval** (`vector_store.py`): two Chroma collections, both embedded with `all-MiniLM-L6-v2`:
- `course_catalog` — one doc per course; the embedded text is only the title, used to fuzzy-resolve a `course_name` like "MCP" to an exact title (top-1 match, no threshold). Lessons are stored as a JSON string in `lessons_json` metadata.
- `course_content` — chunks with `course_title` / `lesson_number` / `chunk_index` metadata used for `where` filters.

Course **title is the unique ID** across the system (catalog ID, chunk ID prefix, dedup key).

**Ingestion** (`document_processor.py`, run on startup from `app.py`): parses a fixed text format — `Course Title:`, `Course Link:`, `Course Instructor:` header lines, then `Lesson N: <title>` markers each optionally followed by `Lesson Link:`. Lesson text is split into sentence-based chunks (`CHUNK_SIZE`/`CHUNK_OVERLAP` in `config.py`). Courses whose title already exists in `chroma_db` are skipped, so edits to an existing transcript require clearing `backend/chroma_db/`. `.pdf`/`.docx` pass the file filter but are read as plain UTF-8 text (no real parser).

**Sessions** (`session_manager.py`): in-memory only, sequential IDs (`session_1`, ...), keeps `MAX_HISTORY` exchanges. History is injected into the **system prompt** as text, not as message turns.

**Adding tools**: subclass `Tool` in `search_tools.py` (implement `get_tool_definition()` returning an Anthropic tool schema and `execute(**kwargs)`), then register it with `tool_manager.register_tool()` in `RAGSystem.__init__`. Note the system prompt in `ai_generator.py` currently instructs "one search per query maximum".

**Config** lives in `backend/config.py` (model ID, embedding model, chunking, `MAX_RESULTS`, `MAX_HISTORY`, Chroma path).
