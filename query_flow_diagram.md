# RAG System Query Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Frontend<br/>(HTML/JS)
    participant FastAPI as FastAPI<br/>(app.py)
    participant RAG as RAG System<br/>(rag_system.py)
    participant AI as AI Generator<br/>(ai_generator.py)
    participant Claude as Anthropic<br/>Claude API
    participant Tools as Search Tools<br/>(search_tools.py)
    participant Vector as Vector Store<br/>(vector_store.py)
    participant Chroma as ChromaDB<br/>(Database)

    User->>Frontend: Types query & clicks Send
    Frontend->>Frontend: Disable input, show loading
    Frontend->>FastAPI: POST /api/query<br/>{query, session_id}
    
    FastAPI->>RAG: rag_system.query(query, session_id)
    RAG->>RAG: Get conversation history
    RAG->>AI: generate_response(query, history, tools)
    
    AI->>Claude: API call with system prompt & tools
    Claude->>AI: Response with tool_use request
    
    alt Claude wants to search
        AI->>Tools: execute_tool("search_course_content", params)
        Tools->>Vector: search(query, course_name, lesson_number)
        Vector->>Vector: Resolve course name (partial match)
        Vector->>Vector: Build metadata filters
        Vector->>Chroma: Similarity search with embeddings
        Chroma->>Vector: Ranked results with metadata
        Vector->>Tools: SearchResults object
        Tools->>Tools: Format results with context headers
        Tools->>AI: Formatted search results
        AI->>Claude: Send tool results back
        Claude->>AI: Final response using search data
    else Direct response
        Claude->>AI: Direct answer (no search needed)
    end
    
    AI->>RAG: Generated response text
    RAG->>Tools: Get last sources from search
    RAG->>RAG: Update conversation history
    RAG->>FastAPI: (response, sources)
    
    FastAPI->>Frontend: JSON response<br/>{answer, sources, session_id}
    Frontend->>Frontend: Remove loading animation
    Frontend->>Frontend: Display response with sources
    Frontend->>User: Show formatted answer
```

## Component Responsibilities

### Frontend Layer
- **User Interface**: Input handling, message display, loading states
- **API Communication**: HTTP requests to backend endpoints
- **Session Management**: Tracks conversation continuity

### Backend API Layer  
- **FastAPI Router**: Request validation, error handling, response formatting
- **Session Management**: Creates/manages user sessions

### RAG Orchestration Layer
- **RAG System**: Coordinates all components, manages conversation flow
- **Tool Management**: Registers and executes available tools

### AI Layer
- **AI Generator**: Interfaces with Claude API, handles tool execution flow
- **Anthropic Claude**: Decides when to search, generates final responses

### Search Layer
- **Search Tools**: Formats queries, processes results for AI consumption  
- **Vector Store**: Semantic search, course name resolution, filtering

### Data Layer
- **ChromaDB**: Vector embeddings, similarity search, metadata storage

## Key Data Transformations

1. **User Query** → HTTP JSON request
2. **RAG Prompt** → "Answer this question about course materials: {query}"
3. **Tool Call** → `search_course_content(query, course_name, lesson_number)`
4. **Vector Search** → ChromaDB similarity query with filters
5. **Search Results** → Formatted context with headers: `[Course Title - Lesson X] content`
6. **AI Response** → JSON with answer, sources, and session_id
7. **Frontend Display** → Rendered markdown with collapsible sources

## Flow Characteristics

- **Async Processing**: Frontend shows loading while backend processes
- **Tool-Based Search**: AI decides when to search based on query type
- **Context Preservation**: Conversation history maintained across queries  
- **Source Tracking**: Search results tracked and displayed to user
- **Error Handling**: Graceful fallbacks at each layer