# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Use `uv` for ALL dependency management and Python execution.** Never use `pip`, `pip-tools`, `poetry`, `requirements.txt`, or invoke `python` / `uvicorn` directly. **Every Python file is run as `uv run python <file>` — never bare `python <file>`.** The project is uv-pinned (`pyproject.toml` + `uv.lock` + `.python-version`) and the lockfile must stay the single source of truth. Do not hand-edit `pyproject.toml` to add a dependency — use `uv add` so the lockfile updates atomically.

```bash
# Environment
uv sync                          # install / sync everything
uv add <pkg>                     # add a dependency
uv add --dev <pkg>               # add a dev-only dependency
uv remove <pkg>                  # remove a dependency
uv lock --upgrade-package <pkg>  # upgrade one package
uv lock --upgrade                # upgrade all

# Running the app (FastAPI + frontend, single process)
./run.sh
#   or, equivalently:
cd backend && uv run uvicorn app:app --reload --port 8000

# Any one-off Python command
uv run python <script>
uv run <command>
```

The app expects `ANTHROPIC_API_KEY` in a `.env` file at the repo root (see `.env.example`).

The server serves both the API and the frontend at `http://localhost:8000`; Swagger docs at `/docs`.

There is **no test suite, no linter, and no type checker configured** in this repo. `pyproject.toml` only declares runtime dependencies.

`main.py` at the repo root is a leftover stub (`print("Hello from starting-codebase!")`) — the real entrypoint is `backend/app.py`.

## Architecture

This is a tool-using RAG chatbot. The architecture only makes sense if you understand two things that span multiple files:

### 1. Claude decides when to search (tool use)

The system does **not** always retrieve before generating. Instead, every `/api/query` call hands Claude a `search_course_content` tool and lets the model decide whether to call it. General-knowledge questions skip the database entirely.

This is enforced by two cooperating pieces:

- `ai_generator.py` makes the **first** Claude call with `tool_choice={"type": "auto"}`. If `stop_reason == "tool_use"`, it calls `_handle_tool_execution`, which runs the tool, appends a `tool_result` block as a user message, and makes a **second** Claude call — this time *without* tools — to force a text answer.
- `search_tools.py` defines the `Tool` ABC, `CourseSearchTool` (the only registered tool), and `ToolManager`. `RAGSystem.__init__` wires these together: `ToolManager.register_tool(CourseSearchTool(vector_store))`.

The `SYSTEM_PROMPT` in `ai_generator.py` restricts Claude to **one search per query**. Multi-hop questions ("compare lesson 3 of A with lesson 5 of B") will not chain searches.

### 2. Sources are returned via a side-channel, not the LLM response

When `CourseSearchTool.execute()` runs, it writes a human-readable list of sources to `self.last_sources` as a side effect. After the LLM call returns, `RAGSystem.query()` reads `tool_manager.get_last_sources()`, then calls `reset_sources()`. The sources are returned to the frontend alongside the answer — they never appear in the model's text output.

If you add a new tool that should populate sources, give it a `last_sources` attribute; `ToolManager.get_last_sources` iterates tools looking for that attribute.

### Component map

```
app.py            FastAPI surface — 2 endpoints + startup ingestion + static mount
  └─ rag_system.py    Orchestrator; owns one instance of each component below
       ├─ document_processor.py   Parses course .txt files → Course + List[CourseChunk]
       ├─ vector_store.py         ChromaDB wrapper (2 collections, see below)
       ├─ ai_generator.py         Claude client + tool-use loop
       ├─ session_manager.py      In-memory chat history (lost on restart)
       └─ search_tools.py         Tool ABC, CourseSearchTool, ToolManager
```

### Two ChromaDB collections with different jobs

`VectorStore` (`vector_store.py`) maintains two collections — they are **not** interchangeable:

- `course_catalog` — one document per course (the course title). Used by `_resolve_course_name()` to turn a partial name like `"MCP"` into the full canonical title via semantic search before content search runs.
- `course_content` — the actual text chunks. Filtered by `course_title` and/or `lesson_number` after name resolution.

**Course title doubles as the Chroma document ID** in `course_catalog`. `add_course_folder()` uses this to deduplicate: if a course title already exists, the file is skipped (not re-ingested).

Lessons are stored as a JSON-serialized string in the catalog metadata (`lessons_json`), then parsed back out by `get_all_courses_metadata()` and `get_lesson_link()`.

### Course document format

`document_processor.process_course_document()` expects this exact format — changing it will silently produce wrong courses:

```
Course Title: <title>           ← becomes the unique ID
Course Link: <url>
Course Instructor: <name>

Lesson 0: <title>
Lesson Link: <url>               ← optional, consumed if present right after marker
<content...>

Lesson 1: <title>
<content...>
```

Chunking is sentence-aware (regex handles common abbreviations) and greedy up to `CHUNK_SIZE` with `CHUNK_OVERLAP` carry-back. The first chunk of each lesson is prefixed with `"Lesson N content: "` so embeddings carry lesson context; the final lesson's chunks use a slightly different prefix (`"Course {title} Lesson {N} content: "`) — this is an inconsistency in the code, not intentional.

### Configuration

All tunables live in `backend/config.py` (`@dataclass Config`, exported as `config`). Notable defaults: `CHUNK_SIZE=800`, `CHUNK_OVERLAP=100`, `MAX_RESULTS=5`, `MAX_HISTORY=2` (only the last 2 exchanges survive), `temperature=0`, `max_tokens=800`. The Chroma DB persists at `./chroma_db` **relative to the backend/ directory** (because `run.sh` cd's into it).

### Ingestion on startup

`app.py`'s `@app.on_event("startup")` calls `rag_system.add_course_folder("../docs", clear_existing=False)`. Adding a new file to `docs/` and restarting the server is the standard way to load new content. Pass `clear_existing=True` only when you want to wipe and rebuild.
