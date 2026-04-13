# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

Install dependencies and start the server (requires Python 3.13+ and `uv`):

```bash
uv sync
./run.sh
```

Or start manually:

```bash
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app will be available at `http://localhost:8000` and API docs at `http://localhost:8000/docs`.

**Environment:** Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`.

## Architecture Overview

This is a full-stack **Retrieval-Augmented Generation (RAG)** system for querying course materials.

### Stack
- **Backend:** FastAPI (Python), served by Uvicorn on port 8000.
- **Frontend:** Vanilla HTML/CSS/JS in `frontend/`, served as static files by FastAPI from `backend/app.py`.
- **Vector DB:** ChromaDB with `sentence-transformers` (`all-MiniLM-L6-v2`) for embeddings.
- **LLM:** Anthropic Claude (`claude-sonnet-4-20250514`) via the Anthropic SDK.

### Core RAG Flow

The system uses a **tool-based RAG architecture** where Claude decides whether to search course content:

1. `app.py` receives `POST /api/query` and delegates to `RAGSystem.query()` in `rag_system.py`.
2. `RAGSystem` builds a prompt, fetches in-memory conversation history from `SessionManager`, and calls `AIGenerator.generate_response()`.
3. `AIGenerator` sends the request to Claude with a system prompt and a single tool definition: `search_course_content`.
4. If Claude decides to search, `AIGenerator._handle_tool_execution()` is triggered:
   - It routes to `CourseSearchTool.execute()` inside `search_tools.py`.
   - `CourseSearchTool` calls `VectorStore.search()` in `vector_store.py`.
5. `VectorStore` performs semantic search across the `course_content` collection. If a `course_name` filter is provided, it first resolves the exact title via the `course_catalog` collection.
6. Search results are formatted with course/lesson headers and tracked as `last_sources` for the UI.
7. The tool result is sent back to Claude in a second API call for the final natural-language answer.
8. `RAGSystem` retrieves the tracked sources, resets them, updates the session history, and returns the response to the frontend.

### Data Model

ChromaDB uses **two collections**:
- `course_catalog` — stores course metadata (title, instructor, link, serialized lessons JSON). Course title is the unique ID.
- `course_content` — stores chunked lesson text with metadata (`course_title`, `lesson_number`, `chunk_index`).

On startup (`app.py` `startup_event`), the app scans `../docs` and indexes any new `.pdf`, `.docx`, or `.txt` files. Existing courses (matched by title) are skipped to avoid duplicates.

### Document Processing

`DocumentProcessor` in `document_processor.py` expects course documents to follow a specific plain-text format:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <lesson title>
Lesson Link: <url>
<content...>

Lesson 2: <lesson title>
...
```

Text is split into sentence-based chunks of ~800 characters with ~100 characters of overlap. Each chunk is prefixed with context: `Course <title> Lesson <N> content: <chunk>`.

### Frontend Behavior

The UI is a single-page chat interface in `frontend/index.html` and `script.js`. It:
- Sends queries to `/api/query` and maintains `session_id` across turns.
- Renders assistant responses as Markdown via `marked.parse()`.
- Displays collapsible **Sources** when the backend returns them.
- Loads course statistics from `/api/courses` on page load.
