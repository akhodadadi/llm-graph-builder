# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**LLM Graph Builder** is a full-stack application for transforming unstructured data (PDFs, documents, YouTube videos, web pages, etc.) into structured Knowledge Graphs stored in Neo4j using Large Language Models (LLMs) and the LangChain framework.

**Architecture:**
- **Backend:** FastAPI (Python 3.12+) handling graph extraction, LLM orchestration, and Neo4j interactions
- **Frontend:** React 18 + TypeScript with Vite, styled with TailwindCSS and Neo4j NDL components
- **Database:** Neo4j (5.23+) with APOC for graph storage and queries
- **Infrastructure:** Docker Compose for local deployment, Google Cloud Run support

## Setup & Development

### Environment Configuration

**Backend (.env):**
```bash
# Required
NEO4J_URI=bolt://127.0.0.1:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=password
NEO4J_DATABASE=neo4j

# LLM Models (format: model_name,api_key or model_name,endpoint,key)
LLM_MODEL_CONFIG_OPENAI_GPT_4O_MINI="gpt-4o-mini,sk-..."
LLM_MODEL_CONFIG_ANTHROPIC_CLAUDE_4_5_HAIKU="claude-haiku-4-5-20250929,sk-..."
LLM_MODEL_CONFIG_GEMINI_2_5_FLASH="gemini-2.5-flash"

# Optional
OPENAI_API_KEY=
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_PROVIDER=openai
IS_EMBEDDING=True
GCS_FILE_CACHE=False
ENTITY_EMBEDDING=False
YOUTUBE_TRANSCRIPT_PROXY=
```

**Frontend (.env):**
```bash
VITE_BACKEND_API_URL=http://localhost:8000
VITE_REACT_APP_SOURCES=local,youtube,wiki,web,s3
VITE_LLM_MODELS=openai_gpt_4o_mini,openai_gpt_4o
VITE_LLM_MODELS_PROD=gemini_2.5_flash,openai_gpt_5_mini,anthropic_claude_4.5_haiku
VITE_CHAT_MODES=vector,graph_vector,graph,fulltext
VITE_ENV=DEV
VITE_SKIP_AUTH=true
VITE_CHUNK_SIZE=5242880
VITE_CHUNK_OVERLAP=20
VITE_TOKENS_PER_CHUNK=100
```

### Running Locally

**Using Docker Compose (recommended):**
```bash
docker-compose up
# Frontend: http://localhost:8080
# Backend: http://localhost:8000
# Docs: http://localhost:8000/docs (Swagger UI)
```

**Backend Only (development):**
```bash
cd backend
python3.12 -m venv venv
source venv/bin/activate
pip install -r requirements.txt -c constraints.txt
uvicorn score:app --reload
```

**Frontend Only (development):**
```bash
cd frontend
yarn install
yarn run dev
# http://localhost:5173
```

### Code Quality

**Frontend (TypeScript + React):**
```bash
cd frontend
yarn lint                   # Run ESLint
yarn format               # Format with Prettier
yarn lint-staged          # Run lint-staged hooks manually
```

ESLint config: `.eslintrc.json` (strict rules, no `any` types in new code)
Prettier config: `.prettierrc.json` (120 char width, 2 spaces, single quotes)
Pre-commit hook: Runs lint-staged on .ts/.tsx files automatically

**Backend (Python):**
- No automated linting configured; follow PEP 8 conventions
- Type hints recommended but not enforced

## Architecture & Data Flow

### Backend Structure (`backend/src/`)

**Core Modules:**
- **`score.py`** — FastAPI application entry point; defines ~25 REST endpoints
- **`main.py`** — Core graph extraction logic; handles file ingestion from multiple sources (local, GCS, S3, YouTube, Wikipedia, web)
- **`llm.py`** — LLM abstraction layer supporting 10+ models (OpenAI, Gemini, Anthropic, Groq, Bedrock, Ollama, etc.)
- **`graphDB_dataAccess.py`** — Neo4j CRUD operations; manages Document nodes, source tracking, processing status
- **`QA_integration.py`** — RAG (Retrieval-Augmented Generation) chat interface with multiple search modes
- **`post_processing.py`** — Graph cleanup, duplicate node merging, community detection, entity embeddings
- **`make_relationships.py`** — Creates relationships between chunks and entities; manages vector indexes

**Document Source Handlers (`document_sources/`):**
- `local_file.py` — Local file processing (PDF, DOCX, TXT)
- `gcs_bucket.py` — Google Cloud Storage integration
- `s3_bucket.py` — AWS S3 integration
- `youtube.py` — YouTube transcript extraction
- `wikipedia.py` — Wikipedia page scraping
- `web_pages.py` — Web page content extraction

**Data Models (`entities/`):**
- `user_credential.py` — Neo4j connection credentials
- `source_extract_params.py` — Extraction parameters from frontend
- `source_node.py` — Document metadata object

**Shared Utilities (`shared/`):**
- `common_fn.py` — File operations, embedding logic, token tracking, LLM orchestration helpers
- `constants.py` — Cypher queries, configuration constants, instruction prompts
- `schema_extraction.py` — Graph schema inference from text

### API Endpoints (FastAPI)

**Graph Operations:**
- `POST /extract` — Extract graph from uploaded file with selected LLM
- `POST /url/scan` — Process URL/S3/GCS sources
- `POST /post_processing` — Run graph cleanup (deduplication, communities, embeddings)
- `POST /schema` — Infer/validate node and relationship labels
- `POST /populate_graph_schema` — Apply custom schema to extraction

**Chat & Query:**
- `POST /chat_bot` — RAG-based chat interface with multiple search modes
- `POST /graph_query` — Execute Cypher queries or semantic search
- `POST /chunk_entities` — Get entities within specific file chunks
- `GET /fetch_chunktext` — Retrieve original chunk text for sources

**File & Source Management:**
- `POST /upload` — Chunk and upload files (supports parallel chunks)
- `POST /sources_list` — List all processed sources/documents
- `POST /delete_document_and_entities` — Delete source and related data
- `GET /document_status/{file_name}` — Poll processing status

**Graph Optimization:**
- `POST /get_unconnected_nodes_list` — Find orphaned nodes
- `POST /delete_unconnected_nodes` — Remove orphaned nodes
- `POST /get_duplicate_nodes` — Identify similar nodes
- `POST /merge_duplicate_nodes` — Merge duplicate entities

**Configuration:**
- `POST /connect` — Test Neo4j connection
- `POST /backend_connection_configuration` — Store connection details
- `POST /fetch_embedding_model` — Get user's embedding config
- `POST /change_embedding_model` — Switch embedding model

**Metrics:**
- `POST /metric` — RAGAS evaluation metrics
- `POST /additional_metrics` — Extra quality metrics

### Frontend Structure (`frontend/src/`)

**Entry Points:**
- `main.tsx` — Vite app bootstrap; Auth0 conditional wrapper
- `App.tsx` — Router setup; `/` (main), `/chat-only` (chat-only mode), `/readonly`

**Core Components:**
- `Content.tsx` — Main UI container; file upload, table, graph preview
- `FileTable.tsx` — Table of uploaded files with status, actions, metrics
- `Home.tsx` — Root component with auth flow

**Feature Modules:**
- **`ChatBot/`** — Chat interface with Q&A, history, source attribution
- **`Graph/`** — Neo4j graph visualization (Neo4j NVL), schema viewer
- **`DataSources/`** — File upload, URL/bucket inputs, source selectors
- **`Popups/`** — Dialogs for settings, schema customization, confirmation flows
- **`UI/`** — Reusable components (buttons, dropdowns, loaders)
- **`Auth/`** — Auth0 integration (optional, controlled by `VITE_SKIP_AUTH`)

**Context Providers (`context/`):**
- `UserCredentials.tsx` — Neo4j connection state
- `UsersFiles.tsx` — Uploaded files/sources state
- `UserMessages.tsx` — Toast notifications
- `Alert.tsx` — Alert dialogs

**Services (`services/`):**
Each service calls backend endpoints via Axios:
- `PostProcessing.ts` — Trigger post-processing
- `DeleteFiles.ts` — Delete documents
- `GraphQuery.ts` — Execute graph queries
- `ChunkEntitiesInfo.ts` — Fetch entities in chunks
- `ChangeEmbeddingModel.ts` — Update embedding config
- `GetRagasMetric.ts` — Fetch evaluation metrics

**State Management:**
- React Context (no Redux/Zustand)
- Axios interceptors inject Neo4j credentials into all requests automatically
- Server-Sent Events (SSE) for streaming extraction progress

**Styling:**
- TailwindCSS (4.x) for utility-first CSS
- Neo4j NDL Design System components
- App.css for custom styles

### Data Flow: File to Graph

1. **Upload:** User selects file → chunks at configurable size (5MB default) → parallel upload with `chunkNumber/totalChunks`
2. **Processing:** Backend receives chunks → merges locally/GCS → extracts text (PDF, DOCX, TXT via LangChain)
3. **LLM Extraction:** Text chunked by tokens (10k default) → LLM extracts nodes/relationships (using LLMGraphTransformer or DiffBot)
4. **Graph Creation:** Entities and relationships saved to Neo4j with source attribution
5. **Post-Processing:** Deduplication, community detection, embeddings (optional)
6. **Chat:** User queries → vector search + graph traversal (multiple modes) → LLM generates answers

### Neo4j Schema

**Document Node:**
```cypher
Document {
  fileName, fileSize, fileType, status, url, model,
  createdAt, updatedAt, processingTime,
  nodeCount, relationshipCount, errorMessage,
  gcsBucket, gcsBucketFolder, chunkNodeCount, chunkRelCount,
  entityNodeCount, entityEntityRelCount, communityNodeCount, communityRelCount
}
```

**Relationships:**
- `Document -[:CONTAINS]-> Chunk` — Document to text chunks
- `Chunk -[:HAS_ENTITY]-> Entity` — Chunk to extracted entities
- `Entity -[:RELATES_TO]-> Entity` — Entity relationships
- `Entity -[:HAS_EMBEDDING]-> Vector` — Vector embeddings (if enabled)
- `Entity -[:IN_COMMUNITY]-> Community` — Community grouping

## Key Development Patterns

### Adding a New Data Source

1. Create handler in `backend/src/document_sources/{source_name}.py`
2. Implement function returning `List[Document]`
3. Call from `main.py` extraction functions
4. Add to `APP_SOURCES` in `frontend/src/utils/Constants.ts`
5. Create input component in `frontend/src/components/DataSources/`

### Adding a New LLM Model

1. Update `backend/.env` with model config: `LLM_MODEL_CONFIG_{MODEL_NAME}="model_id,api_key,..."`
2. Add condition in `backend/src/llm.py` `get_llm()` function
3. Add to `VITE_LLM_MODELS` or `VITE_LLM_MODELS_PROD` in frontend `.env`
4. Update `frontend/src/utils/Constants.ts` `llms` or `prodllms` list

### Modifying Graph Schema

- Graph extraction uses `LLMGraphTransformer` (LangChain) or `DiffbotGraphTransformer`
- Schema can be constrained in `frontend/src/components/Popups/GraphEnhancementDialog`
- Constraints sent to backend in `allowedNodes` and `allowedRelationship` of `ExtractParams`
- Backend filters/merges extracted nodes/relationships in `main.py`

### Adding Backend Endpoints

1. Import dependencies at top of `score.py`
2. Define request/response models (Pydantic)
3. Use `@app.post()` or `@app.get()` decorator
4. Inject `Neo4jCredentials` via `Depends()` for DB access
5. Use `create_graph_database_connection()` to get Neo4j driver
6. Return `create_api_response()` for consistency

### Frontend API Integration

1. Create service file in `frontend/src/services/{Name}.ts`
2. Use Axios instance from `frontend/src/API/Index.ts` (credentials auto-injected)
3. Handle both sync and async responses (check for SSE streams)
4. Update related Context if state management needed

## Testing

- **Backend:** Test files in `backend/` (`test_integrationqa.py`, `test_commutiesqa.py`, `Performance_test.py`) but not integrated into CI
- **Frontend:** No automated tests configured; manual testing via Vite dev server

## Deployment

**Google Cloud Run (via Cloud Build):**
- `cloudbuild.yaml` orchestrates build and deploy for `main`, `staging`, `dev` branches
- Backend and frontend deploy separately
- Substitutions for secrets (API keys) injected at build time
- Frontend must be manually uncommented in `cloudbuild.yaml` to deploy

**Environment Variables for Cloud:**
- All `LLM_MODEL_CONFIG_*` vars must be set as Cloud Build substitutions
- GCS bucket credentials via `GOOGLE_APPLICATION_CREDENTIALS`
- Set `VITE_ENV=PROD` for production frontend builds

## Important Constraints & Gotchas

1. **Token Tracking:** If `TRACK_USER_USAGE=true`, requires separate Neo4j instance for token tracking (not main graph DB)
2. **File Chunking:** Frontend chunks large files (5MB default); backend must reassemble before processing
3. **LLM Rate Limits:** Extract operations are sequential; consider retry logic for transient failures
4. **Async Processing:** Frontend uses SSE for extraction progress; WebSocket not supported (use polling for status)
5. **Neo4j Version:** Requires 5.23+ with APOC; some queries use `apoc.algo.pagerank`
6. **CORS:** Backend allows all origins (`*`); add restrictions for production
7. **Embedding Models:** Sentence Transformers model pre-downloaded in Docker build for offline use

## File Organization Quick Reference

- **Backend entry:** `backend/score.py`
- **Main extraction logic:** `backend/src/main.py`
- **LLM config:** `backend/src/llm.py`
- **Neo4j queries:** `backend/src/shared/constants.py`
- **Frontend entry:** `frontend/src/main.tsx`
- **Main UI:** `frontend/src/components/Content.tsx`
- **API calls:** `frontend/src/API/Index.ts`
- **Constants/models:** `frontend/src/utils/Constants.ts`, `frontend/src/types.ts`
