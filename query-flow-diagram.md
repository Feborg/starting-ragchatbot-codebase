# Query Processing Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                 FRONTEND                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
│
│ 1. User types query in chat input
│ 2. script.js:sendMessage() triggered
│ 3. POST /api/query {query, session_id}
│
▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              FASTAPI SERVER                                    │
│                                app.py:56                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
│
│ 4. Endpoint receives query
│ 5. Creates session if needed
│ 6. Calls rag_system.query()
│
▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               RAG SYSTEM                                       │
│                             rag_system.py:102                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
│
│ 7. Builds prompt with query
│ 8. Gets conversation history
│ 9. Calls ai_generator.generate_response()
│
▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AI GENERATOR                                      │
│                            ai_generator.py:43                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
│
│ 10. Sends to Claude API with tools enabled
│ 11. Claude decides: Use search tool? ──── No ────┐
│                                │                  │
│                               Yes                 │
│                                │                  │
│                                ▼                  │
│         ┌─────────────────────────────────────────────────────┐                │
│         │              TOOL EXECUTION                        │                │
│         │            search_tools.py:52                      │                │
│         └─────────────────────────────────────────────────────┘                │
│                                │                              │                │
│         12. CourseSearchTool.execute() called                 │                │
│         13. Calls vector_store.search()                       │                │
│                                │                              │                │
│                                ▼                              │                │
│         ┌─────────────────────────────────────────────────────┐                │
│         │              VECTOR STORE                          │                │
│         │             vector_store.py:61                     │                │
│         └─────────────────────────────────────────────────────┘                │
│                                │                              │                │
│         14. Course name resolution (if provided)              │                │
│             ├─ Semantic search on course_catalog             │                │
│             └─ Find exact course title match                 │                │
│         15. Build ChromaDB filters                            │                │
│         16. Semantic search on course_content                 │                │
│                                │                              │                │
│                                ▼                              │                │
│         ┌─────────────────────────────────────────────────────┐                │
│         │               CHROMADB                             │                │
│         │            chroma_db/ folder                       │                │
│         └─────────────────────────────────────────────────────┘                │
│                                │                              │                │
│         17. Vector similarity search                          │                │
│         18. Returns documents + metadata                      │                │
│                                │                              │                │
│                                ▼                              │                │
│         19. Format results with course/lesson context         │                │
│         20. Return to AI Generator                            │                │
│                                │                              │                │
│                                ▼                              │                │
│         21. Send tool results back to Claude ─────────────────┘                │
│         22. Claude generates final answer                                      │
│                                                                                 │
│                                ▼                                               │
│         23. Return response text ◄─────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            RESPONSE ASSEMBLY                                   │
│                             rag_system.py:129                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
│
│ 24. Get sources from tool_manager
│ 25. Update conversation history
│ 26. Return {answer, sources}
│
▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              API RESPONSE                                      │
│                                app.py:68                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
│
│ 27. Return JSON: {answer, sources, session_id}
│
▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               FRONTEND                                         │
│                              script.js:76                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
│
│ 28. Parse JSON response
│ 29. Update session ID
│ 30. Display answer (markdown → HTML)
│ 31. Show sources in collapsible section

```

## Key Decision Points

- **Tool Usage**: Claude autonomously decides whether to search based on query type
- **Course Resolution**: Vector search finds best course match for partial names
- **Context Preservation**: Session management maintains conversation history
- **Source Tracking**: Search results provide UI source references