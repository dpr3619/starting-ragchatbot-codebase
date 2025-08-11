# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start using the provided script
chmod +x run.sh
./run.sh

# Manual start from backend directory
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Environment Setup
```bash
# Install dependencies
uv sync

# Create required environment file
# Create .env file with: ANTHROPIC_API_KEY=your_key_here
```

The application serves at `http://localhost:8000` with API docs at `http://localhost:8000/docs`.

## Architecture Overview

This is a **Retrieval-Augmented Generation (RAG) system** for querying course materials using semantic search and AI-powered responses.

### Core Architecture Pattern
The system follows a **tool-based RAG architecture** where Claude AI decides when to search course content:

1. **Query Flow**: User → FastAPI → RAG System → AI Generator → Claude API → Search Tools → Vector Store → ChromaDB
2. **Tool-Based Search**: Claude autonomously decides whether to use the search tool based on query content
3. **Two-Phase AI**: Initial Claude call → Tool execution → Follow-up Claude call with results

### Key Components

**Backend (`/backend/`)**:
- `app.py` - FastAPI server with `/api/query` and `/api/courses` endpoints
- `rag_system.py` - **Main orchestrator** that coordinates all components
- `ai_generator.py` - Handles Anthropic Claude API integration with tool support
- `search_tools.py` - Defines the search tool interface and execution logic
- `vector_store.py` - ChromaDB wrapper with semantic search and course name resolution
- `document_processor.py` - Parses structured course documents and creates text chunks
- `session_manager.py` - Manages conversation history per user session
- `config.py` - Configuration management with environment variable support

**Data Models (`models.py`)**:
- `Course` - Contains course metadata and lessons list
- `Lesson` - Individual lesson with title and optional link
- `CourseChunk` - Text chunks for vector storage with course/lesson metadata

### Document Processing Pipeline

Course documents follow a specific format:
```
Course Title: [title]
Course Link: [optional url]  
Course Instructor: [name]

Lesson 0: Introduction
[lesson content...]

Lesson 1: Next Topic
[lesson content...]
```

The system:
1. Parses structured headers for course metadata
2. Segments content by lesson markers (`Lesson X:`)
3. Creates sentence-based text chunks with configurable overlap
4. Adds contextual headers to chunks (`Course Title Lesson X content: [chunk]`)
5. Stores in ChromaDB with metadata for filtering

### Search Architecture

**Two-Collection Design**:
- `course_catalog` - Course titles and metadata for name resolution
- `course_content` - Text chunks with course/lesson metadata for semantic search

**Smart Course Name Resolution**:
- Partial course name matching (e.g., "MCP" matches "MCP: Build Rich-Context AI Apps")
- Fuzzy matching for common abbreviations and keywords

**Search Tool Parameters**:
- `query` (required) - What to search for
- `course_name` (optional) - Course filter with partial matching
- `lesson_number` (optional) - Specific lesson filter

### AI Integration

**System Prompt Strategy**:
- Instructs Claude to use search tool only for course-specific questions
- One search per query maximum to prevent over-searching
- Direct responses for general knowledge questions

**Tool Execution Flow**:
1. Claude receives query with available tools
2. If course-specific, Claude calls `search_course_content` tool
3. Tool results sent back to Claude for final response synthesis
4. Sources tracked and returned to frontend for display

### Configuration

**Key Settings** (`config.py`):
- `CHUNK_SIZE: 800` - Text chunk size for vector storage
- `CHUNK_OVERLAP: 100` - Overlap between chunks for context preservation
- `MAX_RESULTS: 5` - Search results limit
- `MAX_HISTORY: 2` - Conversation memory depth
- `ANTHROPIC_MODEL: "claude-sonnet-4-20250514"` - Claude model version
- `EMBEDDING_MODEL: "all-MiniLM-L6-v2"` - Sentence transformer for embeddings

### Session Management

Conversation context is maintained through:
- UUID-based session identifiers
- Rolling conversation history (configurable depth)
- Automatic session creation for new users
- History integration in AI prompts for coherent conversations

### Data Storage

**ChromaDB Structure**:
- Persistent storage in `./chroma_db/`
- Sentence transformer embeddings for semantic similarity
- Metadata filtering for course/lesson constraints
- Automatic duplicate prevention during document loading

### Error Handling Patterns

- Graceful degradation at each layer (FastAPI → RAG → AI → Tools)
- UTF-8 encoding fallback for document reading
- Course name resolution errors return helpful messages
- Search errors don't crash the system, return empty results with error context
- always use uv to run the server
- make sure to use uv to manage all dpendencies
- use uv to run python files
- don't run the server using ./run.sh I will start it myself