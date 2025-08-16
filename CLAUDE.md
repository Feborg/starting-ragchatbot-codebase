# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Package Management
```bash
# Install dependencies
uv sync

# Run Python commands in environment
cd backend && uv run python [script.py]
```

### Environment Setup
- Requires `ANTHROPIC_API_KEY` in `.env` file
- Uses Python 3.13+ with UV package manager
- Application runs on `http://localhost:8000`

## Architecture Overview

### RAG Pipeline Architecture
This is a **Retrieval-Augmented Generation (RAG) system** with a tool-calling architecture where Claude autonomously decides when to search course content.

**Core Flow:**
1. `RAGSystem` orchestrates all components
2. `AIGenerator` sends queries to Claude with tool definitions
3. Claude decides whether to call `search_course_content` tool
4. `CourseSearchTool` performs semantic search via `VectorStore`
5. Claude synthesizes final response using retrieved context

### Component Relationships

**Data Models (`models.py`):**
- `Course` → `Lesson[]` → `CourseChunk[]` (hierarchical structure)
- Course title serves as unique identifier across all components

**Document Processing Pipeline:**
- `DocumentProcessor` parses structured course files with lesson markers
- Text chunking uses sentence-based splitting with configurable overlap
- Chunks get context-enhanced: `"Course [title] Lesson [N] content: [chunk]"`

**Vector Storage (`vector_store.py`):**
- **Dual Collections**: `course_catalog` (metadata) + `course_content` (chunks)
- **Course Resolution**: Semantic search on titles enables partial name matching
- **Smart Filtering**: Supports course + lesson filtering for precise retrieval

**Session Management:**
- `SessionManager` maintains conversation history (max 2 exchanges)
- Session IDs persist across frontend interactions

### Key Implementation Details

**Tool-Calling Pattern:**
- `ToolManager` registers tools and provides definitions to Claude
- Tools implement abstract `Tool` interface with `get_tool_definition()` and `execute()`
- Sources tracked in tools and retrieved after execution for UI display

**Frontend Integration:**
- Static file serving with development no-cache headers
- Session-based conversation state
- Real-time course statistics from `/api/courses` endpoint

**Configuration (`config.py`):**
- Centralized settings for chunk size (800), overlap (100), max results (5)
- Uses Claude Sonnet 4 and all-MiniLM-L6-v2 embeddings
- ChromaDB persistence in `./chroma_db`

### Document Format Expected
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 1: [lesson title]
Lesson Link: [lesson url]
[lesson content...]

Lesson 2: [lesson title]
[lesson content...]
```