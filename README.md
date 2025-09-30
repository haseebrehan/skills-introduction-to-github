# RAG Application Detailed Flow Diagrams

## 1. Document Upload Flow

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Frontend UI
    participant API as API Service
    participant BE as FastAPI Backend
    participant DP as Document Processor
    participant TE as Text Extractor
    participant TS as Text Splitter
    participant EMB as OpenAI Embeddings
    participant VDB as ChromaDB
    participant FS as File System

    U->>UI: Drag & Drop PDF file
    UI->>UI: Validate file (type, size)
    UI->>API: POST /documents/upload
    API->>BE: Upload file with multipart/form-data
    
    BE->>BE: Generate UUID for document
    BE->>FS: Save file as UUID_filename.pdf
    FS-->>BE: File saved
    
    BE->>DP: Process document
    DP->>TE: Extract text based on file type
    
    alt PDF File
        TE->>TE: Use PyPDF2 to extract text
    else DOCX File
        TE->>TE: Use python-docx to extract text
    else TXT File
        TE->>TE: Read text directly
    end
    
    TE-->>DP: Return extracted text
    
    DP->>TS: Split text into chunks
    TS->>TS: Apply RecursiveCharacterTextSplitter<br/>(chunk_size=1000, overlap=200)
    TS-->>DP: Return document chunks
    
    DP-->>BE: Return processed chunks
    
    BE->>EMB: Generate embeddings for all chunks
    EMB->>EMB: Call OpenAI API for each chunk
    EMB-->>BE: Return vector embeddings (1536 dims)
    
    BE->>VDB: Store chunks + embeddings + metadata
    VDB->>VDB: Add to collection "documents"
    VDB-->>BE: Confirm storage
    
    BE->>BE: Store metadata in memory<br/>(document_id, filename, date, chunk_count)
    
    BE-->>API: Return success response
    API-->>UI: Update document list
    UI-->>U: Show success notification<br/>"Document uploaded: 15 chunks"
```

## 2. Query/Chat Flow

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Chat Interface
    participant API as API Service
    participant BE as FastAPI Backend
    participant EMB as OpenAI Embeddings
    participant VDB as ChromaDB
    participant RAG as RAG Chain
    participant LLM as OpenAI GPT

    U->>UI: Type question: "What is RAG?"
    UI->>UI: Show typing indicator
    UI->>API: POST /query<br/>{question: "What is RAG?", k: 5}
    API->>BE: Send query request
    
    BE->>EMB: Generate embedding for query
    EMB->>EMB: Call OpenAI Embeddings API
    EMB-->>BE: Return query vector (1536 dims)
    
    BE->>VDB: Similarity search with query vector
    VDB->>VDB: Calculate cosine similarity<br/>with all stored vectors
    VDB->>VDB: Rank and select top K=5 chunks
    VDB-->>BE: Return top 5 relevant chunks + metadata
    
    BE->>RAG: Build context from retrieved chunks
    RAG->>RAG: Concatenate chunk texts
    RAG->>RAG: Format prompt template:<br/>"Context: [chunks]<br/>Question: [query]<br/>Answer:"
    
    RAG->>LLM: Send prompt to GPT-3.5/4
    LLM->>LLM: Generate response based on context
    LLM-->>RAG: Return generated answer
    
    RAG->>RAG: Format response:<br/>- answer text<br/>- source citations<br/>- metadata
    RAG-->>BE: Return formatted response
    
    BE-->>API: JSON response with answer + sources
    API-->>UI: Display response
    UI->>UI: Render answer in chat bubble
    UI->>UI: Add expandable "Sources" section
    UI-->>U: Show complete answer with citations
```

## 3. Document Delete Flow

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Document List
    participant API as API Service
    participant BE as FastAPI Backend
    participant VDB as ChromaDB
    participant FS as File System
    participant MEM as In-Memory Store

    U->>UI: Click delete icon on document
    UI->>UI: Show confirmation dialog
    U->>UI: Confirm deletion
    
    UI->>API: DELETE /documents/{document_id}
    API->>BE: Send delete request
    
    BE->>MEM: Get document metadata
    MEM-->>BE: Return filename and path
    
    BE->>VDB: Delete all chunks with document_id
    VDB->>VDB: Query by metadata filter
    VDB->>VDB: Remove matching vectors
    VDB-->>BE: Confirm deletion
    
    BE->>FS: Delete physical file
    FS->>FS: Remove UUID_filename.pdf
    FS-->>BE: File deleted
    
    BE->>MEM: Remove metadata entry
    MEM-->>BE: Metadata removed
    
    BE-->>API: Success response
    API-->>UI: Update document list
    UI->>UI: Remove document from UI
    UI-->>U: Show success notification
```

## 4. Application Initialization Flow

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant ENV as .env File
    participant APP as FastAPI App
    participant VDB as ChromaDB
    participant FS as File System
    participant API as API Routes

    DEV->>ENV: Set OPENAI_API_KEY
    DEV->>DEV: Run: python main.py
    
    DEV->>APP: Start FastAPI application
    APP->>ENV: Load environment variables
    ENV-->>APP: Return API keys & configs
    
    APP->>FS: Check/Create ./documents folder
    FS-->>APP: Folder ready
    
    APP->>FS: Check/Create ./chroma_db folder
    FS-->>APP: Folder ready
    
    APP->>VDB: Initialize ChromaDB client
    VDB->>VDB: Create/Load "documents" collection
    VDB-->>APP: Client ready
    
    APP->>API: Register route handlers
    API->>API: Mount /documents routes
    API->>API: Mount /query routes
    API-->>APP: Routes registered
    
    APP->>APP: Setup CORS middleware
    APP->>APP: Start uvicorn server on port 8000
    APP-->>DEV: Server running at http://localhost:8000
```

## 5. Frontend-Backend Integration Flow

```mermaid
graph LR
    subgraph "Frontend Components"
        A[App.tsx] --> B[DocumentUpload.tsx]
        A --> C[ChatInterface.tsx]
        A --> D[Settings.tsx]
        
        B --> E[useDocuments hook]
        C --> F[useChat hook]
        D --> G[useSettings hook]
        
        E --> H[api.ts]
        F --> H
        G --> H
    end
    
    subgraph "API Layer"
        H -->|axios/fetch| I[HTTP Requests]
        I -->|POST /documents/upload| J[FormData]
        I -->|POST /query| K[JSON Body]
        I -->|GET /documents| L[Query Params]
        I -->|DELETE /documents/:id| M[Path Params]
    end
    
    subgraph "Backend Endpoints"
        J --> N[documents.upload]
        K --> O[query.query_documents]
        L --> P[documents.list_documents]
        M --> Q[documents.delete_document]
    end
    
    subgraph "Response Handling"
        N --> R[Success/Error Response]
        O --> R
        P --> R
        Q --> R
        
        R --> S[React Query Cache]
        S --> T[UI State Update]
        T --> U[Re-render Components]
    end
    
    style H fill:#6366F1,stroke:#4F46E5,stroke-width:2px,color:#fff
    style I fill:#10B981,stroke:#059669,stroke-width:2px,color:#fff
    style R fill:#F59E0B,stroke:#D97706,stroke-width:2px,color:#fff
```

## 6. Data Flow Architecture

```mermaid
graph TB
    subgraph "User Actions"
        A1[Upload Document]
        A2[Ask Question]
        A3[View Sources]
        A4[Delete Document]
    end
    
    subgraph "Frontend State Management"
        B1[Documents State]
        B2[Chat History State]
        B3[Settings State]
        B4[Loading State]
    end
    
    subgraph "API Calls"
        C1[POST /documents/upload]
        C2[POST /query]
        C3[GET /documents]
        C4[DELETE /documents/:id]
    end
    
    subgraph "Backend Processing"
        D1[File Processing]
        D2[RAG Pipeline]
        D3[Metadata Management]
        D4[Vector Operations]
    end
    
    subgraph "Data Storage"
        E1[(File System)]
        E2[(ChromaDB)]
        E3[Memory Cache]
    end
    
    subgraph "External APIs"
        F1[OpenAI Embeddings]
        F2[OpenAI Chat]
    end
    
    A1 --> B1 --> C1 --> D1 --> E1
    D1 --> F1 --> D4 --> E2
    
    A2 --> B2 --> C2 --> D2 --> D4
    D4 --> E2
    D2 --> F2 --> D2
    D2 --> B2
    
    A3 --> B2 --> B2
    
    A4 --> B1 --> C4 --> D3 --> E1
    D3 --> D4 --> E2
    D3 --> E3
    
    style A1 fill:#7C3AED,stroke:#6D28D9,stroke-width:2px,color:#fff
    style A2 fill:#7C3AED,stroke:#6D28D9,stroke-width:2px,color:#fff
    style E1 fill:#F59E0B,stroke:#D97706,stroke-width:2px,color:#fff
    style E2 fill:#F59E0B,stroke:#D97706,stroke-width:2px,color:#fff
    style F1 fill:#EF4444,stroke:#DC2626,stroke-width:2px,color:#fff
    style F2 fill:#EF4444,stroke:#DC2626,stroke-width:2px,color:#fff
```

## 7. Component Hierarchy

```
RAG Application
│
├── Frontend (React/Loveable)
│   ├── App Component
│   │   ├── Layout
│   │   │   ├── Sidebar
│   │   │   │   ├── DocumentUpload
│   │   │   │   │   ├── FileDropzone
│   │   │   │   │   └── UploadProgress
│   │   │   │   ├── DocumentList
│   │   │   │   │   └── DocumentItem
│   │   │   │   └── ActionButtons
│   │   │   └── MainContent
│   │   │       ├── ChatHeader
│   │   │       ├── MessageList
│   │   │       │   ├── UserMessage
│   │   │       │   └── AIMessage
│   │   │       │       └── SourcesCitation
│   │   │       └── MessageInput
│   │   └── Modals
│   │       └── SettingsModal
│   ├── Hooks
│   │   ├── useDocuments
│   │   ├── useChat
│   │   └── useSettings
│   └── Services
│       └── api.ts
│
└── Backend (FastAPI)
    ├── main.py
    ├── app/
    │   ├── routes/
    │   │   ├── documents.py
    │   │   └── query.py
    │   ├── services/
    │   │   ├── document_processor.py
    │   │   ├── vector_store.py
    │   │   └── rag_chain.py
    │   ├── models/
    │   │   └── schemas.py
    │   └── config.py
    └── Storage
        ├── documents/ (File System)
        └── chroma_db/ (Vector Database)
```

## Key Technologies Used

### Frontend Stack:
- **React 18+**: UI framework
- **TypeScript**: Type safety
- **Tailwind CSS**: Styling
- **React Query**: API state management
- **Axios**: HTTP client
- **Lucide React**: Icons
- **Loveable**: AI-powered generation

### Backend Stack:
- **FastAPI**: Web framework
- **LangChain**: RAG orchestration
- **ChromaDB**: Vector database
- **OpenAI API**: Embeddings + LLM
- **PyPDF2**: PDF processing
- **python-docx**: Word processing
- **Pydantic**: Data validation

### Communication:
- **REST API**: HTTP/JSON
- **CORS**: Cross-origin requests
- **Multipart Form Data**: File uploads
- **WebSocket** (optional): Streaming responses

## Performance Considerations

1. **Chunking Strategy**: 1000 chars with 200 overlap
2. **Embedding Dimension**: 1536 (OpenAI)
3. **Similarity Search**: Cosine similarity
4. **Top K Results**: Default 5 chunks
5. **API Rate Limits**: Consider OpenAI limits
6. **Caching**: React Query for frontend
7. **Async Processing**: FastAPI async handlers


