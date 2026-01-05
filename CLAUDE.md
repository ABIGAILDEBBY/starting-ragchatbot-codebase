# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Retrieval-Augmented Generation (RAG) system for querying course materials. It uses ChromaDB for vector storage, Anthropic's Claude for AI generation with tool calling, and provides a web interface. The system allows Claude AI to autonomously decide when to search the knowledge base using tool-based function calling.

## Environment Setup

### Required Environment Variables

Create a `.env` file in the root directory:
```
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

### Installation and Dependency Management

**IMPORTANT: Always use `uv` to manage all dependencies. Do not use `pip` directly.**

```bash
# Install uv package manager (if not installed)
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Install dependencies
uv sync

# Note: NumPy compatibility issue
# If you encounter NumPy 2.x errors, downgrade to 1.x:
uv pip install "numpy<2"
```

### Python Version

The project uses Python 3.12 (not 3.13 as README states) due to PyTorch compatibility. The `.python-version` file has been modified to `3.12`.

## Running the Application

```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000

# Access points
# - Web Interface: http://localhost:8000
# - API Documentation: http://localhost:8000/docs
```

## Architecture Overview

### Core Request Flow (Tool-Based RAG)

The system implements a two-phase AI interaction pattern:

1. **Phase 1 - Decision**: Claude receives a query and available tools, decides whether to search
2. **Phase 2 - Synthesis**: Claude receives search results and generates final answer

```
User Query → FastAPI → RAG System → AI Generator (Phase 1)
    ↓
Tool Manager → Search Tool → Vector Store (ChromaDB)
    ↓
AI Generator (Phase 2) → Response with Sources → User
```

### Component Responsibilities

#### Frontend Layer (`frontend/`)
- **script.js**: Handles user input, HTTP requests via Fetch API, Markdown rendering with marked.js
- **index.html**: Single-page interface with chat area and sidebar
- Session management is client-side; stores `session_id` for conversation continuity

#### Backend Layer (`backend/`)

**Entry Point: `app.py`**
- FastAPI application with CORS middleware
- Two main endpoints:
  - `POST /api/query`: Process user queries (requires QueryRequest with query + optional session_id)
  - `GET /api/courses`: Return course statistics
- On startup, automatically loads documents from `../docs` directory
- Serves frontend static files from `../frontend`

**Orchestrator: `rag_system.py`**
- Coordinates all components (document processor, vector store, AI generator, session manager, tool manager)
- `add_course_document()`: Process single course file
- `add_course_folder()`: Batch process entire directories, skips duplicates based on course titles
- `query()`: Main query processing - retrieves history, calls AI with tools, extracts sources

**AI Integration: `ai_generator.py`**
- Manages Claude API interactions using Anthropic SDK
- Model: `claude-sonnet-4-20250514` (configurable in config.py)
- Temperature: 0 (deterministic responses)
- Max tokens: 800
- Key method: `generate_response()` handles two-phase tool calling
  - Phase 1: Send query + tools → Claude returns `tool_use` request
  - Phase 2: Send tool results → Claude synthesizes final answer
- System prompt enforces: "One search per query maximum", no meta-commentary, concise responses

**Tool System: `search_tools.py`**
- `Tool` abstract base class defines interface: `get_tool_definition()` and `execute()`
- `CourseSearchTool`: Executes semantic search with optional course_name and lesson_number filters
- `ToolManager`: Registers tools, provides definitions to Claude, executes tool calls, tracks sources
- Tool results are formatted as: `[Course Title - Lesson N]\nContent...`

**Vector Storage: `vector_store.py`**
- Uses ChromaDB for persistent vector storage (stored in `backend/chroma_db/`)
- Two collections:
  - `course_metadata`: Stores course titles and metadata
  - `course_content`: Stores text chunks with embeddings
- Embedding model: `all-MiniLM-L6-v2` (SentenceTransformers)
- `search()`: Main search interface with course/lesson filtering
- `add_course_metadata()`: Stores course info with course title as document ID
- `add_course_content()`: Stores chunks with course_title and lesson_number metadata

**Document Processing: `document_processor.py`**
- Parses PDF, DOCX, and TXT files
- Extracts structured course data (title, instructor, lessons with numbers and titles)
- Chunking: 800 characters with 100 character overlap (configurable)
- Returns `Course` object and list of `CourseChunk` objects

**Session Management: `session_manager.py`**
- In-memory conversation history storage (not persisted)
- `create_session()`: Generates UUID session IDs
- `add_exchange()`: Stores user query + AI response pairs
- `get_conversation_history()`: Returns formatted history string
- MAX_HISTORY: 2 (last 2 exchanges kept per session)

**Data Models: `models.py`**
- `Lesson`: lesson_number (int), title (str), lesson_link (optional)
- `Course`: title (str, unique identifier), course_link, instructor, lessons (List[Lesson])
- `CourseChunk`: content (str), course_title (str), lesson_number (optional int), chunk_index (int)

**Configuration: `config.py`**
- Centralized settings using dataclass
- Key parameters:
  - CHUNK_SIZE: 800 characters
  - CHUNK_OVERLAP: 100 characters
  - MAX_RESULTS: 5 search results
  - MAX_HISTORY: 2 conversation turns
  - EMBEDDING_MODEL: "all-MiniLM-L6-v2"
  - CHROMA_PATH: "./chroma_db"

### Data Flow Example

**User Input**: "What is RAG?"

1. **Frontend** sends POST to `/api/query` with `{query: "What is RAG?", session_id: null}`
2. **app.py** creates session, calls `rag_system.query()`
3. **rag_system.py** builds prompt, retrieves empty history (new session)
4. **ai_generator.py** calls Claude with tools
5. **Claude Phase 1** returns: `tool_use: search_course_content(query="RAG retrieval")`
6. **ToolManager** routes to CourseSearchTool
7. **vector_store.py** embeds query, searches ChromaDB, returns chunks with metadata
8. **CourseSearchTool** formats: `[Intro to RAG - Lesson 2]\nRAG is a technique...`
9. **ai_generator.py** sends results to Claude Phase 2
10. **Claude Phase 2** synthesizes: `"RAG (Retrieval-Augmented Generation) is..."`
11. **rag_system.py** extracts sources from tool, updates session history
12. **app.py** returns JSON: `{answer: "...", sources: ["Intro to RAG - Lesson 2"], session_id: "abc123"}`
13. **Frontend** renders Markdown answer, shows collapsible sources dropdown

### Important Implementation Details

**Tool Calling Pattern**:
- Claude is given tool definitions via `tools` parameter
- `tool_choice: auto` lets Claude decide when to search
- Tool execution happens in `_handle_tool_execution()` which constructs a two-turn conversation
- Tool results are sent as `role: user` with `type: tool_result`

**Search Optimization**:
- System prompt enforces "one search per query maximum" to prevent excessive API calls
- Search tool supports optional filters: `course_name` (partial match) and `lesson_number` (exact)
- Vector store uses semantic similarity with configurable `MAX_RESULTS=5`

**Session Continuity**:
- Session IDs generated on first query if not provided
- Frontend stores session_id and includes it in subsequent requests
- History formatted as: `User: {query}\nAssistant: {response}\n\n`
- Only last 2 exchanges retained to control context size

**Document Loading**:
- `app.py` startup event loads all files from `../docs` on server start
- Duplicate detection uses course titles from vector store
- Supported formats: PDF, DOCX, TXT
- Documents are automatically chunked and embedded

**ChromaDB Persistence**:
- Database stored in `backend/chroma_db/` directory
- Collections persist between server restarts
- `clear_existing=False` by default to preserve data
- Use `clear_existing=True` in `add_course_folder()` to rebuild from scratch

## Common Development Tasks

### Adding New Course Materials

```python
# Single file
from rag_system import RAGSystem
from config import config

rag = RAGSystem(config)
course, chunks = rag.add_course_document("path/to/course.pdf")

# Entire folder
courses, chunks = rag.add_course_folder("docs/", clear_existing=False)
```

### Clearing and Rebuilding Vector Database

```python
# Complete rebuild
rag.add_course_folder("docs/", clear_existing=True)

# Or manually
from vector_store import VectorStore
store = VectorStore(config.CHROMA_PATH, config.EMBEDDING_MODEL, config.MAX_RESULTS)
store.clear_all_data()
```

### Modifying AI Behavior

Edit `ai_generator.py` SYSTEM_PROMPT:
- Control search frequency
- Adjust response style
- Add constraints (e.g., "cite specific lessons")

### Adjusting Search Parameters

Edit `config.py`:
- `CHUNK_SIZE`: Larger = more context per chunk, fewer chunks
- `MAX_RESULTS`: More results = more context to Claude, higher API costs
- `MAX_HISTORY`: More history = better context continuity, larger prompts

### Testing API Endpoints

```bash
# Query endpoint
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is RAG?", "session_id": null}'

# Course stats
curl http://localhost:8000/api/courses
```

## Technical Constraints and Gotchas

1. **NumPy Version**: Must use NumPy 1.x due to PyTorch 2.2.2 compatibility (Intel Mac limitation)
2. **Python Version**: Project uses 3.12, not 3.13 as stated in README
3. **Tool Calling Limitations**: System prompt enforces one search per query to control costs
4. **Session Persistence**: Sessions are in-memory only; restarting server clears all sessions
5. **ChromaDB Threading**: ChromaDB uses multiprocessing; startup warnings about resource trackers are normal
6. **CORS**: Configured for `allow_origins=["*"]` for development; tighten for production
7. **Frontend Caching**: DevStaticFiles class adds no-cache headers for development hot-reloading
8. **API Key Required**: Application will start but fail on queries without valid ANTHROPIC_API_KEY

## Debugging Tips

- Check `backend/chroma_db/` exists and has data if searches return no results
- Monitor terminal output for ChromaDB loading messages on startup
- Use `/docs` endpoint to inspect FastAPI's auto-generated API documentation
- Verify `.env` file is in root directory (not `backend/`), FastAPI loads it via `python-dotenv`
- If Claude doesn't search when expected, check system prompt constraints in `ai_generator.py`
