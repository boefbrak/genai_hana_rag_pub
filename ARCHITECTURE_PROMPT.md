# SAP CAP RAG Application - Reusable Architecture Prompt

Use this prompt template to create similar RAG (Retrieval-Augmented Generation) applications using SAP Cloud Application Programming Model (CAP), SAP HANA Cloud Vector Engine, and SAP AI Core / Generative AI Hub.

---

## PROMPT TEMPLATE

```
Create a RAG (Retrieval-Augmented Generation) application with the following architecture and specifications:

## Technology Stack

### Backend
- **SAP Cloud Application Programming Model (CAP)** with Node.js runtime (@sap/cds ^8)
- **SAP HANA Cloud** with Vector Engine for vector storage (@cap-js/hana ^1)
- **SAP AI Core / Generative AI Hub** via @sap-ai-sdk/foundation-models ^2
  - Embedding model: `text-embedding-3-large` (3072 dimensions) via AzureOpenAiEmbeddingClient
  - Chat model: `gpt-4o` via AzureOpenAiChatClient
- **Express.js** for custom REST endpoints (file upload with multer)
- **File parsing**: pdf-parse for PDFs, csv-parse for CSVs, native Buffer for TXT

### Frontend
- **SAPUI5 / Fiori** with XML views and JavaScript controllers
- Single-page app served as static files
- JSONModel for client-side state management

### Deployment
- **SAP BTP Cloud Foundry** via MTA (Multi-Target Application)
- Three modules: srv (Node.js), db-deployer (HDB), app (staticfile)
- HDI Container for HANA artifacts
- Existing AI Core service binding

---

## Data Model (CDS Schema)

Create a data model with these entities:

### Documents
- ID (UUID, auto-generated)
- fileName (String 255, mandatory)
- fileType (String 10, mandatory) - pdf, txt, csv
- fileSize (Integer)
- status (String 20, default 'UPLOADED') - UPLOADED → PROCESSING → READY | ERROR
- chunkCount (Integer, default 0)
- errorMsg (String 1000)
- Composition to DocumentChunks

### DocumentChunks
- ID (UUID, auto-generated)
- Association to Documents (mandatory)
- content (LargeString, mandatory)
- chunkIndex (Integer, mandatory)
- tokenCount (Integer)
- embedding (Vector 3072) - HANA vector type

### ChatSessions
- ID (UUID, auto-generated)
- title (String 200)
- Association to Documents (link session to specific document)
- Composition to ChatMessages
- Managed aspect (createdAt, modifiedAt)

### ChatMessages
- ID (UUID, auto-generated)
- Association to ChatSessions (mandatory)
- role (String 20, mandatory) - 'user' | 'assistant'
- content (LargeString, mandatory)
- timestamp (auto-set on insert)
- sources (LargeString) - JSON of source references

---

## Services (CDS)

### DocumentService (path: /api/documents)
Exposed entities:
- Documents (excluding chunks composition for security)

Actions:
- `deleteDocument(documentId: UUID)` → Boolean
  - Cascades: delete messages → sessions → chunks → document

Functions:
- `getStatus(documentId: UUID)` → { status, chunkCount, errorMsg }
- `getDeletePreview(documentId: UUID)` → { sessionCount, messageCount, chunkCount }

### ChatService (path: /api/chat)
Exposed entities:
- ChatSessions (excluding messages)
- ChatMessages

Actions:
- `sendMessage(sessionId: UUID, message: String)` → { reply, messageId, sources[] }
- `createSession(documentId: UUID, title: String)` → ChatSessions
- `updateSession(sessionId: UUID, documentId: UUID, title: String)` → ChatSessions

Functions:
- `getSessionMessages(sessionId: UUID)` → ChatMessages[]
- `getDocumentSessions(documentId: UUID)` → ChatSessions[]

---

## Backend Implementation Patterns

### Server Bootstrap (server.js)
- Use `cds.on('bootstrap', app => {...})` to add custom Express middleware
- Enable CORS for cross-origin frontend
- Add custom POST endpoint `/api/documents/upload` using multer for file handling
- Memory storage with 10MB file size limit

### Upload Processing Pipeline
1. **Receive file** → Validate extension (pdf, txt, csv)
2. **Create document record** with status='PROCESSING'
3. **Process asynchronously** (don't block response):
   - Parse file content using appropriate parser
   - Chunk text with overlap (default: 1000 tokens, 200 overlap)
   - Generate embeddings in batches (20 at a time)
   - Store chunks with vectors using raw SQL (for Vector type support)
   - Update document status to 'READY' or 'ERROR'
4. **Return immediately** with document ID for status polling

### Text Chunking Strategy
- Character-based approximation (4 chars ≈ 1 token)
- Max chunk size: ~4000 characters (1000 tokens)
- Overlap: ~800 characters (200 tokens)
- Sentence boundary detection for natural breaks
- Return array of { content, tokenCount }

### Embedding Generation
- Use singleton client pattern for AzureOpenAiEmbeddingClient
- Model: 'text-embedding-3-large' (3072 dimensions)
- Batch processing with Promise.all for parallel requests
- Return array of embedding vectors

### Vector Search (Raw SQL for HANA)
```sql
SELECT TOP {topK}
  c."ID", c."CONTENT", c."CHUNKINDEX", c."DOCUMENT_ID",
  d."FILENAME" AS "documentName",
  COSINE_SIMILARITY(c."EMBEDDING", TO_REAL_VECTOR('{embeddingJson}')) AS "similarity"
FROM "NAMESPACE_DOCUMENTCHUNKS" c
INNER JOIN "NAMESPACE_DOCUMENTS" d ON c."DOCUMENT_ID" = d."ID"
WHERE d."STATUS" = 'READY'
  AND c."DOCUMENT_ID" IN ({documentIdList})  -- optional filter
ORDER BY "similarity" DESC
```

### RAG Response Generation
1. Embed user query
2. Search for similar chunks (top 10)
3. Build context block from chunks with source attribution
4. Construct messages array:
   - System prompt with rules + context block
   - Chat history (last 10 messages)
   - Current user query
5. Call GPT-4o with temperature=0.3, max_tokens=2000
6. Store assistant response with sources JSON

### System Prompt Template
```
You are a helpful AI assistant that answers questions based on the provided document context.

Rules:
1. Answer ONLY based on the provided context. If the context doesn't contain enough information, say so clearly.
2. Cite which document(s) your answer is based on when possible.
3. Be concise but thorough.
4. If the user's question is a greeting or general conversation, respond naturally.
5. Maintain a professional and helpful tone.

--- DOCUMENT CONTEXT ---
[Source 1: "filename.pdf", relevance: 95.2%]
{chunk content}

---

[Source 2: "filename.pdf", relevance: 87.1%]
{chunk content}
--- END CONTEXT ---
```

---

## Frontend Architecture

### Component Structure
```
app/webapp/
├── Component.js          # UI5 Component with router
├── manifest.json         # App descriptor with routing config
├── config.js            # Runtime config (API base URL)
├── index.html           # Bootstrap page
├── css/style.css        # Custom styles
├── model/
│   └── formatter.js     # Display formatters
├── view/
│   ├── App.view.xml     # Root view with App control
│   ├── Main.view.xml    # Split-pane layout (docs + chat)
│   ├── Upload.view.xml  # Alternative upload view
│   └── Chat.view.xml    # Alternative chat view
├── controller/
│   ├── App.controller.js
│   ├── Main.controller.js
│   ├── Upload.controller.js
│   └── Chat.controller.js
└── fragment/
    └── UploadDialog.fragment.xml
```

### Main View Layout (Recommended)
- Horizontal Splitter with two panels:
  - **Left (350px)**: Document list with upload button
    - ObjectListItem showing fileName, fileType, fileSize, status, chunkCount
    - Status indicator: READY=Success, ERROR=Error, PROCESSING=Warning
  - **Right (flex)**: Chat interface
    - Empty state when no document selected
    - Chat area with message list + input bar when document selected

### Controller Patterns
- Use `window.RAG_CONFIG.apiBaseUrl` for configurable API endpoint
- JSONModel for documents, messages, app state
- Fetch API for all backend calls (no OData binding complexity)
- Status polling after upload (setInterval, 3 seconds)
- Auto-create/select session when document is selected

### Chat Message Display
- User messages: blue background, right-aligned
- Assistant messages: white background with border, left-aligned
- Icons to differentiate sender (person-placeholder vs da-2)
- "Thinking..." indicator while waiting for response
- ScrollContainer with auto-scroll to bottom

---

## Deployment Configuration

### MTA Structure (mta.yaml)
```yaml
_schema-version: "3.1"
ID: {app-name}
version: 1.0.0

parameters:
  enable-parallel-deployments: true

build-parameters:
  before-all:
    - builder: custom
      commands:
        - npm ci
        - npx cds build --production

modules:
  # Node.js service
  - name: {app-name}-srv
    type: nodejs
    path: gen/srv
    parameters:
      buildpack: nodejs_buildpack
      memory: 512M
      disk-quota: 1024M
    requires:
      - name: hana-hdi
      - name: aicore
    provides:
      - name: srv-api
        properties:
          srv-url: ${default-url}

  # HANA DB deployer
  - name: {app-name}-db-deployer
    type: hdb
    path: gen/db
    requires:
      - name: hana-hdi

  # Static file frontend
  - name: {app-name}-app
    type: staticfile
    path: app/webapp
    parameters:
      buildpack: staticfile_buildpack
      memory: 64M
    requires:
      - name: srv-api
        group: destinations
        properties:
          name: srv-api
          url: ~{srv-url}
          forwardAuthToken: true

resources:
  # HDI Container
  - name: hana-hdi
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared
      config:
        database_id: {database-guid}

  # AI Core (existing shared service)
  - name: aicore
    type: org.cloudfoundry.existing-service
    parameters:
      service-name: {aicore-service-name}
```

### Package.json CDS Configuration
```json
{
  "cds": {
    "requires": {
      "db": {
        "kind": "sql",
        "[production]": {
          "kind": "hana-cloud",
          "deploy-format": "hdbtable"
        },
        "[hybrid]": {
          "kind": "hana-cloud",
          "deploy-format": "hdbtable"
        }
      },
      "auth": {
        "[production]": {
          "kind": "dummy"
        }
      }
    },
    "hana": {
      "deploy-format": "hdbtable"
    }
  }
}
```

---

## File Structure Summary

```
{project-root}/
├── package.json           # Dependencies and CDS config
├── mta.yaml              # Multi-target app descriptor
├── .cdsrc.json           # CDS runtime config
├── db/
│   └── schema.cds        # Data model definitions
├── srv/
│   ├── server.js         # Custom server bootstrap
│   ├── chat-service.cds  # Chat service definition
│   ├── chat-service.js   # Chat service implementation
│   ├── document-service.cds
│   ├── document-service.js
│   └── lib/
│       ├── file-parser.js    # PDF/TXT/CSV parsing
│       ├── chunker.js        # Text chunking logic
│       ├── embedder.js       # Embedding generation
│       ├── vector-search.js  # HANA vector search
│       ├── rag-engine.js     # RAG response generation
│       └── upload-processor.js  # Upload orchestration
└── app/
    └── webapp/
        ├── Component.js
        ├── manifest.json
        ├── config.js
        ├── index.html
        ├── css/style.css
        ├── model/formatter.js
        ├── view/*.xml
        ├── controller/*.js
        └── fragment/*.xml
```

---

## Key Implementation Notes

1. **Vector Storage**: Use raw SQL for INSERT/SELECT with HANA vectors since CDS doesn't natively support Vector type
2. **Async Processing**: File processing runs asynchronously to avoid timeout on large files
3. **Error Handling**: Document status tracks processing errors for user feedback
4. **Session Scope**: Each chat session is linked to one document for focused RAG retrieval
5. **CORS**: Enable in server.js for cross-origin frontend deployment
6. **Staticfile Buildpack**: Use Staticfile in webapp root to configure routing for Cloud Foundry
7. **Config Injection**: Use window global for runtime API URL configuration

---

## Customization Points

Replace these values when creating a new application:
- `{app-name}` - Your application identifier
- `{database-guid}` - Your HANA Cloud database GUID
- `{aicore-service-name}` - Your AI Core service instance name
- Embedding model and dimension (currently text-embedding-3-large, 3072)
- Chat model (currently gpt-4o)
- Chunk size and overlap parameters
- System prompt for your specific use case
- File type restrictions (currently pdf, txt, csv)
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SAP BTP Cloud Foundry                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────┐    ┌──────────────────┐    ┌───────────────┐ │
│  │   Frontend App   │    │   CAP Service    │    │   HANA Cloud  │ │
│  │   (staticfile)   │───►│   (Node.js)      │───►│   (HDI)       │ │
│  │                  │    │                  │    │               │ │
│  │  - SAPUI5/Fiori  │    │  - OData/REST    │    │  - Documents  │ │
│  │  - XML Views     │    │  - File Upload   │    │  - Chunks     │ │
│  │  - Controllers   │    │  - RAG Pipeline  │    │  - Vectors    │ │
│  └──────────────────┘    └────────┬─────────┘    │  - Sessions   │ │
│                                   │              │  - Messages   │ │
│                                   │              └───────────────┘ │
│                                   │                                 │
│                          ┌────────▼─────────┐                      │
│                          │   AI Core        │                      │
│                          │   (Gen AI Hub)   │                      │
│                          │                  │                      │
│                          │  - Embeddings    │                      │
│                          │  - Chat (GPT-4o) │                      │
│                          └──────────────────┘                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

RAG Pipeline Flow:
1. Upload → Parse → Chunk → Embed → Store (with vectors)
2. Query → Embed → Vector Search → Build Context → LLM → Response
```

---

This template captures the complete architecture of a production-ready RAG application on SAP BTP. Modify the customization points and system prompt to adapt for your specific use case.
