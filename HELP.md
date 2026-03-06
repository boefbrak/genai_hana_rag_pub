# SAP CAP RAG Application - Help Guide

A Retrieval-Augmented Generation (RAG) application built with SAP Cloud Application Programming Model (CAP), SAP HANA Cloud Vector Engine, and SAP Generative AI Hub.

## Overview

This application allows users to:
- Upload documents (PDF, TXT, CSV) up to 10 MB
- Automatically extract text, chunk it, and generate embeddings
- Store embeddings in SAP HANA Cloud Vector Engine
- Chat with uploaded documents using GPT-4o via SAP Generative AI Hub
- Maintain conversation history with document-linked sessions
- Delete documents with cascade deletion of all related data
- View and delete failed (ERROR) documents

## Application Architecture

### High-Level Component Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SAP Business Technology Platform                    │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                          Cloud Foundry Environment                         │  │
│  │                                                                            │  │
│  │  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐    │  │
│  │  │   Web App       │      │   CAP Service   │      │   DB Deployer   │    │  │
│  │  │   (UI5/Fiori)   │─────▶│   (Node.js)     │      │   (HDI Tasks)   │    │  │
│  │  │                 │      │                 │      │                 │    │  │
│  │  │ staticfile_     │      │ nodejs_         │      │ nodejs_         │    │  │
│  │  │ buildpack       │      │ buildpack       │      │ buildpack       │    │  │
│  │  └─────────────────┘      └────────┬────────┘      └────────┬────────┘    │  │
│  │                                    │                        │             │  │
│  └────────────────────────────────────│────────────────────────│─────────────┘  │
│                                       │                        │                │
│  ┌────────────────────────────────────│────────────────────────│─────────────┐  │
│  │                              Services                       │             │  │
│  │                                    │                        │             │  │
│  │  ┌─────────────────┐      ┌───────▼─────────┐      ┌───────▼─────────┐   │  │
│  │  │   SAP AI Core   │      │   HANA Cloud    │      │  HDI Container  │   │  │
│  │  │   (Shared)      │◀────▶│   Vector Engine │◀─────│  (Per User)     │   │  │
│  │  │                 │      │                 │      │                 │   │  │
│  │  │ • GPT-4o        │      │ • Vector(3072)  │      │ • Tables        │   │  │
│  │  │ • text-embed-3  │      │ • COSINE_SIM    │      │ • Views         │   │  │
│  │  └─────────────────┘      └─────────────────┘      └─────────────────┘   │  │
│  │                                                                           │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Multi-User Deployment Architecture

Each user gets an isolated deployment in a shared Cloud Foundry space using the MTA namespace feature:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Shared Cloud Foundry Space                               │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                    User A Namespace (--namespace usera)                  │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │    │
│  │  │usera-genai- │  │usera-genai- │  │usera-genai- │  │usera-hana-  │     │    │
│  │  │hana-rag-app │  │hana-rag-srv │  │hana-rag-db- │  │hdi-rag      │     │    │
│  │  │             │  │             │  │deployer     │  │(HDI)        │     │    │
│  │  └─────────────┘  └──────┬──────┘  └─────────────┘  └─────────────┘     │    │
│  └──────────────────────────│──────────────────────────────────────────────┘    │
│                             │                                                    │
│  ┌──────────────────────────│──────────────────────────────────────────────┐    │
│  │                    User B│Namespace (--namespace userb)                  │    │
│  │  ┌─────────────┐  ┌──────▼──────┐  ┌─────────────┐  ┌─────────────┐     │    │
│  │  │userb-genai- │  │userb-genai- │  │userb-genai- │  │userb-hana-  │     │    │
│  │  │hana-rag-app │  │hana-rag-srv │  │hana-rag-db- │  │hdi-rag      │     │    │
│  │  │             │  │             │  │deployer     │  │(HDI)        │     │    │
│  │  └─────────────┘  └──────┬──────┘  └─────────────┘  └─────────────┘     │    │
│  └──────────────────────────│──────────────────────────────────────────────┘    │
│                             │                                                    │
│  ┌──────────────────────────▼──────────────────────────────────────────────┐    │
│  │                    Shared Services (No Namespace)                        │    │
│  │  ┌─────────────────────────────┐  ┌─────────────────────────────────┐   │    │
│  │  │      SAP AI Core            │  │       SAP HANA Cloud            │   │    │
│  │  │      (ch-sbb-aicore)        │  │       (Database Instance)       │   │    │
│  │  │                             │  │                                 │   │    │
│  │  │  • GPT-4o model             │  │  • Hosts all HDI containers     │   │    │
│  │  │  • text-embedding-3-large   │  │  • Vector Engine enabled        │   │    │
│  │  └─────────────────────────────┘  └─────────────────────────────────┘   │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### RAG Processing Flow

```
┌──────────────┐     ┌───────────────────────────────────────────────────────────────┐
│   Document   │     │                    CAP Service Layer                          │
│   Upload     │     │                                                               │
│              │     │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  PDF/TXT/CSV │────▶│  │ file-parser │───▶│   chunker   │───▶│  embedder   │       │
│              │     │  │             │    │             │    │             │       │
└──────────────┘     │  │ Extract     │    │ Split into  │    │ Generate    │       │
                     │  │ raw text    │    │ ~1000 token │    │ 3072-dim    │       │
                     │  │             │    │ chunks      │    │ vectors     │       │
                     │  └─────────────┘    └─────────────┘    └──────┬──────┘       │
                     │                                               │              │
                     └───────────────────────────────────────────────│──────────────┘
                                                                     │
                                                                     ▼
┌──────────────┐     ┌───────────────────────────────────────────────────────────────┐
│   User       │     │                    Chat Flow                                  │
│   Question   │     │                                                               │
│              │     │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  "What is    │────▶│  │  embedder   │───▶│vector-search│───▶│ rag-engine  │       │
│   this about?"│    │  │             │    │             │    │             │       │
│              │     │  │ Embed       │    │ Find top-10 │    │ Build prompt│       │
└──────────────┘     │  │ question    │    │ similar     │    │ + call GPT  │       │
                     │  │             │    │ chunks      │    │             │       │
                     │  └─────────────┘    └─────────────┘    └──────┬──────┘       │
                     │                                               │              │
                     │                           ┌───────────────────┘              │
                     │                           ▼                                  │
                     │                    ┌─────────────┐                           │
                     │                    │  Response   │──────▶ "This document...  │
                     │                    │  + Sources  │        covers topic X..." │
                     │                    └─────────────┘                           │
                     │                                                               │
                     └───────────────────────────────────────────────────────────────┘
```

## User Interface

### Layout

The application uses a side-by-side layout:

```
┌────────────────────────────────────────────────────────────────────┐
│                        RAG Chat Assistant                          │
├──────────────────┬─────────────────────────────────────────────────┤
│                  │                                                 │
│   Documents      │              Chat Panel                         │
│   ┌──────────┐   │  ┌─────────────────────────────────────────┐   │
│   │ doc1.pdf │   │  │ Sessions │     Messages                 │   │
│   │ Ready ✓  │   │  │ ──────── │ ─────────────────────────── │   │
│   ├──────────┤   │  │ Chat 1   │  User: What is this about?  │   │
│   │ doc2.csv │   │  │ Chat 2   │  AI: This document covers... │   │
│   │ Ready ✓  │   │  │          │                              │   │
│   ├──────────┤   │  │  [+New]  │                              │   │
│   │ doc3.txt │   │  │          │ ─────────────────────────── │   │
│   │ Error ✗  │   │  │          │  [Ask a question...]  [Send] │   │
│   └──────────┘   │  └─────────────────────────────────────────┘   │
│   [+ Upload]     │                                                 │
└──────────────────┴─────────────────────────────────────────────────┘
```

### Document States

| State | Description | UI Behavior |
|-------|-------------|-------------|
| **PROCESSING** | Document being processed (chunking, embedding) | Cannot be selected, shows warning status |
| **READY** | Document ready for chat | Can be selected, shows chat interface |
| **ERROR** | Processing failed | Can be selected, shows error panel with delete option |

### User Workflow

1. **Upload Document**: Click [+] button in Documents panel, select PDF/TXT/CSV file
2. **Wait for Processing**: Status shows "Processing...", auto-refreshes when complete
3. **Select Document**: Click on READY document in left panel
4. **Start Chat Session**: Click [+New] to create a session for the selected document
5. **Ask Questions**: Type questions in the input bar, responses use only that document's content
6. **Delete Document**: Click delete button in document header (shows confirmation with counts)
7. **Delete Failed Document**: Select ERROR document, click delete in the error panel

## Data Model

The application uses four CDS entities organized into two functional domains.

### Entity Relationship Diagram (ERD)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  DATABASE SCHEMA                                     │
│                              namespace: genai.rag                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌────────────────────────────────┐           ┌────────────────────────────────┐    │
│  │          DOCUMENTS             │           │       DOCUMENT_CHUNKS          │    │
│  │     (GENAI_RAG_DOCUMENTS)      │           │   (GENAI_RAG_DOCUMENTCHUNKS)   │    │
│  ├────────────────────────────────┤           ├────────────────────────────────┤    │
│  │ PK  ID          UUID           │           │ PK  ID          UUID           │    │
│  │     ─────────────────────────  │    1:N    │     ─────────────────────────  │    │
│  │     fileName    VARCHAR(255)   │◀──────────│ FK  DOCUMENT_ID UUID           │    │
│  │     fileType    VARCHAR(10)    │ Composition│    ─────────────────────────  │    │
│  │     fileSize    INTEGER        │           │     content     NCLOB          │    │
│  │     status      VARCHAR(20)    │           │     chunkIndex  INTEGER        │    │
│  │     chunkCount  INTEGER        │           │     tokenCount  INTEGER        │    │
│  │     errorMsg    VARCHAR(1000)  │           │     embedding   REAL_VECTOR    │    │
│  │     createdAt   TIMESTAMP      │           │                 (3072)         │    │
│  │     createdBy   VARCHAR(255)   │           │                                │    │
│  │     modifiedAt  TIMESTAMP      │           └────────────────────────────────┘    │
│  │     modifiedBy  VARCHAR(255)   │                                                 │
│  └────────────────────────────────┘                                                 │
│              ▲                                                                       │
│              │ 1:N Association                                                       │
│              │                                                                       │
│  ┌───────────┴────────────────────┐           ┌────────────────────────────────┐    │
│  │        CHAT_SESSIONS           │           │        CHAT_MESSAGES           │    │
│  │    (GENAI_RAG_CHATSESSIONS)    │           │    (GENAI_RAG_CHATMESSAGES)    │    │
│  ├────────────────────────────────┤           ├────────────────────────────────┤    │
│  │ PK  ID          UUID           │           │ PK  ID          UUID           │    │
│  │     ─────────────────────────  │    1:N    │     ─────────────────────────  │    │
│  │     title       VARCHAR(200)   │◀──────────│ FK  SESSION_ID  UUID           │    │
│  │ FK  DOCUMENT_ID UUID           │ Composition│    ─────────────────────────  │    │
│  │     createdAt   TIMESTAMP      │           │     role        VARCHAR(20)    │    │
│  │     createdBy   VARCHAR(255)   │           │     content     NCLOB          │    │
│  │     modifiedAt  TIMESTAMP      │           │     timestamp   TIMESTAMP      │    │
│  │     modifiedBy  VARCHAR(255)   │           │     sources     NCLOB (JSON)   │    │
│  └────────────────────────────────┘           └────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

Legend:
  PK = Primary Key (auto-generated UUID)
  FK = Foreign Key
  1:N = One-to-Many relationship
  Composition = Parent owns children (cascade delete)
  Association = Reference relationship (handled in service layer)
```

### Simplified ERD (Crow's Foot Notation)

```
┌──────────────────┐         ┌──────────────────┐
│    Documents     │         │  DocumentChunks  │
├──────────────────┤         ├──────────────────┤
│ ID          (PK) │───────┬▶│ ID          (PK) │
│ fileName         │       │ │ document_ID (FK) │
│ fileType         │       │ │ content          │
│ fileSize         │       │ │ chunkIndex       │
│ status           │       │ │ tokenCount       │
│ chunkCount       │       │ │ embedding        │
│ errorMsg         │       │ └──────────────────┘
│ createdAt        │       │
│ modifiedAt       │       │
└──────────────────┘       │         ┌──────────────────┐
         │                 │         │   ChatMessages   │
         │ 1               │         ├──────────────────┤
         │                 │   ┌────▶│ ID          (PK) │
         ▼ *               │   │     │ session_ID  (FK) │
┌──────────────────┐       │   │     │ role             │
│   ChatSessions   │       │   │     │ content          │
├──────────────────┤       │   │     │ timestamp        │
│ ID          (PK) │───────┘   │     │ sources          │
│ document_ID (FK) │───────────┘     └──────────────────┘
│ title            │         1 : *
│ createdAt        │
│ modifiedAt       │
└──────────────────┘
```

### Cascade Delete Flow

When a document is deleted, all related data is removed in this order:

```
Delete Document
    │
    ├──▶ 1. Find all ChatSessions linked to this document
    │
    ├──▶ 2. Delete all ChatMessages from those sessions
    │
    ├──▶ 3. Delete all ChatSessions linked to document
    │
    ├──▶ 4. Delete all DocumentChunks for this document
    │
    └──▶ 5. Delete the Document itself
```

### Entity Descriptions

| Entity | Purpose | Key Fields |
|--------|---------|------------|
| **Documents** | Stores metadata for uploaded files | `status` tracks processing state (UPLOADED → PROCESSING → READY/ERROR) |
| **DocumentChunks** | Stores text chunks with vector embeddings | `embedding` is a 3072-dimension vector for similarity search |
| **ChatSessions** | Groups chat messages into conversations, linked to a document | `document_ID` links session to a specific document for scoped chat |
| **ChatMessages** | Individual messages in a chat session | `role` is "user" or "assistant", `sources` contains JSON of retrieved chunks |

### Relationships

- **Documents → DocumentChunks**: One-to-Many composition. When a document is deleted, all its chunks are cascade deleted.
- **Documents → ChatSessions**: One-to-Many association. Each session is linked to exactly one document. Cascade delete handled in service layer.
- **ChatSessions → ChatMessages**: One-to-Many composition. Deleting a session removes all its messages.

### Status Flow (Documents)

```
UPLOADED ──▶ PROCESSING ──▶ READY
                │
                └──▶ ERROR (with errorMsg)
```

### HANA Table Names

CDS entities are deployed to HANA with uppercase table names:

| CDS Entity | HANA Table |
|------------|------------|
| `genai.rag.Documents` | `GENAI_RAG_DOCUMENTS` |
| `genai.rag.DocumentChunks` | `GENAI_RAG_DOCUMENTCHUNKS` |
| `genai.rag.ChatSessions` | `GENAI_RAG_CHATSESSIONS` |
| `genai.rag.ChatMessages` | `GENAI_RAG_CHATMESSAGES` |

---

## Project Structure

```
genai_hana_rag/
├── app/                          # Frontend application
│   ├── index.html                # Root redirect to webapp
│   └── webapp/                   # UI5 Fiori application
│       ├── index.html            # Main HTML entry point
│       ├── manifest.json         # UI5 application descriptor
│       ├── Component.js          # UI5 Component definition
│       ├── config.js             # API configuration (srv URL) [auto-generated]
│       ├── Staticfile            # Cloud Foundry static file config
│       ├── css/
│       │   └── style.css         # Custom styles (panels, messages, error states)
│       ├── model/
│       │   └── formatter.js      # UI formatters (file size, etc.)
│       ├── fragment/
│       │   └── UploadDialog.fragment.xml  # File upload dialog
│       ├── view/
│       │   ├── App.view.xml      # Root view with App container
│       │   └── Main.view.xml     # Side-by-side layout (documents + chat)
│       └── controller/
│           ├── App.controller.js # Root controller
│           └── Main.controller.js # Combined document & chat logic
│
├── db/                           # Database layer
│   └── schema.cds                # CDS data model with Vector type
│
├── srv/                          # Service layer
│   ├── document-service.cds      # Document service definition
│   ├── document-service.js       # Document service handler (cascade delete)
│   ├── chat-service.cds          # Chat service definition
│   ├── chat-service.js           # Chat service handler (document-scoped)
│   ├── server.js                 # Custom Express server (multer, CORS)
│   └── lib/                      # Utility libraries
│       ├── file-parser.js        # PDF/TXT/CSV text extraction
│       ├── chunker.js            # Text chunking with overlap
│       ├── embedder.js           # SAP AI Hub embedding client
│       ├── vector-search.js      # HANA vector similarity search
│       ├── rag-engine.js         # RAG prompt + GPT-4o client
│       └── upload-processor.js   # Async document processing pipeline
│
├── gen/                          # Generated build artifacts (auto-created)
│   ├── db/                       # HANA deployment artifacts
│   └── srv/                      # Service deployment artifacts
│
├── mta_archives/                 # MTA build output (auto-created)
│   └── genai-hana-rag_1.0.0.mtar # Deployable archive
│
├── setup-deployment.sh           # Multi-user deployment script
├── user-config.json              # User configuration (edit before deploy)
├── user-config.mtaext            # MTA extension template
├── my-deployment.mtaext          # Generated MTA extension [auto-generated]
├── DEPLOYMENT_GUIDE.md           # Multi-user deployment guide
├── HELP.md                       # This file
│
├── package.json                  # Node.js dependencies & CDS config
├── mta.yaml                      # MTA deployment descriptor (base)
├── .cdsrc.json                   # CDS build configuration
└── .npmrc                        # npm configuration (cache path)
```

---

## File Descriptions

### Database Layer (`db/`)

| File | Description |
|------|-------------|
| `schema.cds` | CDS data model defining four entities: `Documents` (uploaded files metadata), `DocumentChunks` (text chunks with Vector(3072) embeddings), `ChatSessions` (conversation sessions), `ChatMessages` (individual messages with sources) |

### Service Layer (`srv/`)

| File | Description |
|------|-------------|
| `document-service.cds` | OData service definition exposing Documents entity and actions: `upload`, `getStatus`, `deleteDocument`, `getDeletePreview` |
| `document-service.js` | Handler implementing document CRUD, cascade delete (chunks → sessions → messages), and delete preview |
| `chat-service.cds` | OData service definition exposing ChatSessions/ChatMessages and actions: `createSession`, `updateSession`, `sendMessage`, `getSessionMessages`, `getDocumentSessions` |
| `chat-service.js` | Handler implementing document-scoped chat with RAG pipeline integration |
| `server.js` | Custom Express middleware for multipart file upload (multer) and CORS headers |

### Library Files (`srv/lib/`)

| File | Description |
|------|-------------|
| `file-parser.js` | Extracts text from uploaded files: `pdf-parse` for PDFs, direct buffer conversion for TXT, `csv-parse` for CSV (converts to "Column: Value" format) |
| `chunker.js` | Splits text into chunks of ~1000 tokens with ~200 token overlap, breaking at sentence boundaries for better context preservation |
| `embedder.js` | Wraps SAP AI SDK's `AzureOpenAiEmbeddingClient` for `text-embedding-3-large` model, processes in batches of 20 texts |
| `vector-search.js` | Executes HANA SQL with `COSINE_SIMILARITY()` function to find top-K similar chunks to a query embedding |
| `rag-engine.js` | Wraps SAP AI SDK's `AzureOpenAiChatClient` for `gpt-4o`, constructs RAG prompts with document context and chat history |
| `upload-processor.js` | Orchestrates async document processing: parse → chunk → embed → store with vector, updates document status |

### Frontend (`app/webapp/`)

| File | Description |
|------|-------------|
| `index.html` | Entry point loading config.js and UI5 bootstrap |
| `manifest.json` | UI5 app descriptor with routing to Main view |
| `Component.js` | UI5 component with router initialization and app model |
| `config.js` | Runtime configuration with API base URL for srv app |
| `App.view.xml` | Root view containing the App container |
| `Main.view.xml` | Side-by-side layout: documents panel (left) + chat panel (right) with empty/error states |
| `Main.controller.js` | Combined controller: document selection, upload, delete, session management, chat |
| `UploadDialog.fragment.xml` | File upload dialog fragment |
| `formatter.js` | UI formatters for file sizes and other display values |
| `style.css` | Custom styles for panels, messages, error states |

### Configuration Files

| File | Description |
|------|-------------|
| `package.json` | Dependencies (`@sap/cds`, `@sap-ai-sdk/foundation-models`, `multer`, `pdf-parse`, `csv-parse`) and CDS configuration |
| `mta.yaml` | Base MTA descriptor defining modules (srv, db-deployer, app) and resources (hana-hdi-rag, aicore) |
| `.cdsrc.json` | CDS build settings for HANA hdbtable format |
| `.npmrc` | npm cache directory configuration |

### Deployment Files

| File | Description |
|------|-------------|
| `setup-deployment.sh` | Automated deployment script with `--config` (generate files) and `--deploy` (full deployment) modes |
| `user-config.json` | User configuration: USERNAME, REGION, DATABASE_ID, AICORE_SERVICE_NAME |
| `user-config.mtaext` | MTA extension template with placeholders |
| `my-deployment.mtaext` | Auto-generated MTA extension with user-specific routes and config |
| `DEPLOYMENT_GUIDE.md` | Detailed multi-user deployment instructions |

---

## Prerequisites

Before building and deploying, ensure you have:

1. **Development Tools**
   - Node.js 18+ installed
   - npm or yarn package manager
   - SAP Cloud Application Programming Model CLI (`@sap/cds-dk`)
   - Cloud Foundry CLI (`cf`)
   - MTA Build Tool (`mbt`)

2. **SAP BTP Account**
   - Cloud Foundry environment enabled
   - SAP HANA Cloud instance provisioned
   - SAP AI Core service with extended plan

3. **AI Core Configuration**
   - `text-embedding-3-large` model deployed
   - `gpt-4o` model deployed
   - Service key created

---

## Step-by-Step Deployment Guide

This application supports **multi-user deployment** where each user gets isolated apps and HDI container in a shared CF space. See `DEPLOYMENT_GUIDE.md` for detailed multi-user instructions.

### Quick Start (Automated Deployment)

```bash
# 1. Edit configuration file with your values
vi user-config.json

# 2. Run the automated deployment script
./setup-deployment.sh --deploy
```

The script will automatically:
- Validate your configuration
- Generate the MTA extension file
- Update the frontend config
- Build the application
- Deploy with namespace isolation
- Bind the shared AI Core service
- Restage the service app

### Manual Deployment Steps

#### Step 1: Install Development Tools

```bash
# Install CDS development kit globally
npm install -g @sap/cds-dk

# Install MTA build tool
npm install -g mbt

# Verify installations
cds --version
mbt --version
cf --version
```

#### Step 2: Configure Your Deployment

Edit `user-config.json` with your values:

```json
{
  "USERNAME": "your-username",
  "REGION": "eu10-004",
  "DATABASE_ID": "your-hana-database-uuid",
  "AICORE_SERVICE_NAME": "ch-sbb-aicore"
}
```

Run the setup script to generate configuration files:

```bash
./setup-deployment.sh --config
```

This generates:
- `my-deployment.mtaext` - MTA extension with your routes
- `app/webapp/config.js` - Frontend API URL

#### Step 3: Login to Cloud Foundry

```bash
# Login to CF
cf login -a https://api.cf.REGION.hana.ondemand.com

# Select your org and space when prompted
# Or specify directly:
cf target -o YOUR_ORG -s YOUR_SPACE
```

#### Step 4: Build the Application

```bash
# Build MTA archive (includes npm install and cds build)
mbt build
```

This creates `mta_archives/genai-hana-rag_1.0.0.mtar`.

#### Step 5: Deploy with Namespace

**IMPORTANT**: Always use the `--namespace` flag to isolate your deployment:

```bash
# Deploy with your username as namespace
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar \
  -e my-deployment.mtaext \
  --namespace YOUR_USERNAME
```

#### Step 6: Bind AI Core Service

The AI Core service is shared and must be bound manually after deployment:

```bash
# Bind the shared AI Core service
cf bind-service YOUR_USERNAME-genai-hana-rag-srv ch-sbb-aicore

# Restage to pick up the binding
cf restage YOUR_USERNAME-genai-hana-rag-srv
```

#### Step 7: Verify Deployment

```bash
# Check running apps (should see your namespace-prefixed apps)
cf apps

# Check services
cf services

# Check MTA deployments
cf mtas

# View srv logs
cf logs YOUR_USERNAME-genai-hana-rag-srv --recent
```

#### Step 8: Access the Application

Your deployment URLs follow this pattern:
- **UI App**: `https://YOUR_USERNAME-genai-hana-rag-app.cfapps.REGION.hana.ondemand.com`
- **Srv API**: `https://YOUR_USERNAME-genai-hana-rag-srv.cfapps.REGION.hana.ondemand.com`

### Naming Convention

| Component | Pattern | Example |
|-----------|---------|---------|
| Web App | `{username}-genai-hana-rag-app` | `jsmith-genai-hana-rag-app` |
| Service App | `{username}-genai-hana-rag-srv` | `jsmith-genai-hana-rag-srv` |
| DB Deployer | `{username}-genai-hana-rag-db-deployer` | `jsmith-genai-hana-rag-db-deployer` |
| HDI Container | `{username}-hana-hdi-rag` | `jsmith-hana-hdi-rag` |
| AI Core | Shared (not namespaced) | `ch-sbb-aicore` |

### Undeploying

To remove your deployment (won't affect other users):

```bash
cf undeploy genai-hana-rag --namespace YOUR_USERNAME --delete-services --delete-service-keys
```

---

## Local Development

### Run Locally with SQLite

```bash
# Start local development server
cds watch
```

This starts the app at `http://localhost:4004` with SQLite for local testing.

**Note**: Vector operations require HANA Cloud. Local development with SQLite won't support embedding/search features.

### Run with Hybrid Mode (Local + Remote HANA)

```bash
# Bind to remote HANA service
cds bind -2 genai-hana-rag-db

# Run in hybrid mode
cds watch --profile hybrid
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| `npm cache` permission errors | Create `.npmrc` with `cache=/tmp/npm-cache` |
| "Multiple databases" error | Add `database_id` to user-config.json |
| 401 Unauthorized on CF | Add `"auth": { "[production]": { "kind": "dummy" } }` to package.json cds config |
| Tables not found in HANA | Delete HDI container and redeploy (see undeploy command) |
| AI model not found | Ensure models are deployed in AI Core with correct names |
| CORS errors in browser | Verify CORS middleware in server.js, check srv logs |
| "Select file first" on upload | Hard refresh browser (Cmd+Shift+R) to clear cache |
| Cannot select PROCESSING document | Wait for processing to complete (auto-refreshes) |
| ERROR document won't delete | Select it first, then click delete in error panel |
| Chat not showing after document select | Document may be ERROR status; check the error panel |
| Old sessions showing | Sessions are now linked to documents; select the correct document |
| "Route already exists" error | Another user has this route; use a unique username |
| AI Core binding fails | AI Core is shared; run `cf bind-service` manually after deploy |
| Deployment overwrites other users | Always use `--namespace YOUR_USERNAME` flag |

### Viewing Logs

```bash
# Real-time logs (use your namespace-prefixed app name)
cf logs YOUR_USERNAME-genai-hana-rag-srv

# Recent logs
cf logs YOUR_USERNAME-genai-hana-rag-srv --recent

# DB deployer logs
cf logs YOUR_USERNAME-genai-hana-rag-db-deployer --recent
```

### Redeploying After Changes

```bash
# Option 1: Full automated redeploy
./setup-deployment.sh --deploy

# Option 2: Manual redeploy
mbt build
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace YOUR_USERNAME
```

### Undeploying (Cleanup)

```bash
# Remove your deployment only (won't affect other users)
cf undeploy genai-hana-rag --namespace YOUR_USERNAME --delete-services --delete-service-keys
```

---

## API Reference

### Document Service (`/api/documents`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/Documents` | GET | List all documents |
| `/Documents('{id}')` | GET | Get document by ID |
| `/upload` | POST | Upload file (multipart/form-data) |
| `/getStatus(documentId='{id}')` | GET | Get document processing status |
| `/deleteDocument` | POST | Delete document with cascade (chunks, sessions, messages) |
| `/getDeletePreview(documentId='{id}')` | GET | Get counts of related data before delete |

### Chat Service (`/api/chat`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/ChatSessions` | GET | List all sessions |
| `/createSession` | POST | Create session linked to a document |
| `/updateSession` | POST | Update session (reassign document, change title) |
| `/getDocumentSessions(documentId='{id}')` | GET | Get all sessions for a document |
| `/getSessionMessages(sessionId='{id}')` | GET | Get messages for session |
| `/sendMessage` | POST | Send message and get RAG response (scoped to session's document) |

### Example: Upload a Document

```bash
curl -X POST "https://YOUR-SRV-URL/api/documents/upload" \
  -F "file=@document.pdf"
```

### Example: Create Chat Session for Document

```bash
curl -X POST "https://YOUR-SRV-URL/api/chat/createSession" \
  -H "Content-Type: application/json" \
  -d '{"documentId": "DOC_UUID", "title": "My Chat Session"}'
```

### Example: Send Chat Message

```bash
curl -X POST "https://YOUR-SRV-URL/api/chat/sendMessage" \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "SESSION_ID", "message": "What is this document about?"}'
```

### Example: Get Delete Preview

```bash
curl "https://YOUR-SRV-URL/api/documents/getDeletePreview(documentId='DOC_UUID')"
# Returns: { sessionCount: 2, messageCount: 15, chunkCount: 42 }
```

### Example: Delete Document (Cascade)

```bash
curl -X POST "https://YOUR-SRV-URL/api/documents/deleteDocument" \
  -H "Content-Type: application/json" \
  -d '{"documentId": "DOC_UUID"}'
```

---

## Technical Details

### Embedding Configuration
- **Model**: `text-embedding-3-large`
- **Dimensions**: 3072
- **Batch Size**: 20 texts per API call

### Chunking Configuration
- **Chunk Size**: ~1000 tokens (~4000 characters)
- **Overlap**: ~200 tokens (~800 characters)
- **Boundary**: Sentence-aware splitting

### RAG Configuration
- **Model**: `gpt-4o`
- **Temperature**: 0.3
- **Max Tokens**: 2000
- **Context**: Top 10 similar chunks
- **History**: Last 10 messages

### Vector Search
- **Algorithm**: Cosine Similarity
- **Function**: `COSINE_SIMILARITY()` in HANA SQL
- **Top-K**: 10 chunks returned

---

## Changelog

### Version 1.1.0 - Multi-User Deployment (February 2026)

**New Features:**
- Multi-user deployment support using CF MTA namespace feature
- Each user gets isolated apps and HDI container in shared CF space
- Automated deployment script (`setup-deployment.sh`) with `--deploy` flag
- User configuration file (`user-config.json`) for easy setup
- Automatic AI Core service binding after deployment

**Architecture Changes:**
- Renamed HDI container resource from `genai-hana-rag-db` to `hana-hdi-rag`
- HDI containers now named `{username}-hana-hdi-rag`
- AI Core service is shared (not namespaced) and bound via script
- Added MTA extension file generation for user-specific routes

**Deployment Changes:**
- Deploy command now requires `--namespace USERNAME` flag
- AI Core marked as `active: false` in extension, bound after deployment
- Added undeploy command that only removes specific user's deployment

**Files Added:**
- `setup-deployment.sh` - Automated multi-user deployment script
- `user-config.json` - User configuration template
- `user-config.mtaext` - MTA extension template
- `DEPLOYMENT_GUIDE.md` - Detailed multi-user deployment guide

### Version 1.0.0 - Initial Release

**Features:**
- Document upload (PDF, TXT, CSV) with automatic text extraction
- Text chunking with sentence-aware boundaries
- Vector embeddings using `text-embedding-3-large` (3072 dimensions)
- HANA Cloud Vector Engine for similarity search
- RAG chat using GPT-4o via SAP Generative AI Hub
- Document-linked chat sessions
- Cascade delete for documents (chunks, sessions, messages)
- Side-by-side UI layout with document and chat panels

---

## License

Internal SAP project - refer to your organization's licensing policies.
