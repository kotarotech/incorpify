# Quickstart: Document Ingestion Pipeline

**Feature**: 002-document-ingestion | **Date**: 2026-03-03

## Prerequisites

1. **Bun** installed (runtime)
2. **Docker** running (for Qdrant and PostgreSQL)
3. **OpenAI API key** with access to `text-embedding-3-large`
4. **Knowledge base documents** extracted to a local directory

## Setup

### 1. Start infrastructure

From the monorepo root:

```bash
cd backend-agents-service
docker compose -f docker/docker-compose.yml up -d
```

This starts:
- PostgreSQL on `localhost:5432`
- Qdrant on `localhost:6333` (REST) / `localhost:6334` (gRPC)

### 2. Configure environment

Copy the `.env.example` and set required variables:

```bash
cp .env.example .env
```

Key variables for ingestion:

```env
DATABASE_URL=postgresql://incorpify:incorpify@localhost:5432/incorpify
QDRANT_URL=http://localhost:6333
QDRANT_COLLECTION_NAME=knowledge_base
VECTOR_DIMENSION=3072
OPENAI_API_KEY=sk-...
KB_SOURCE_DIR=./knowledge-base
```

### 3. Run database migrations

```bash
bun run migrate
```

This applies all Drizzle migrations including the new `kb_indexed_documents` and `kb_ingestion_jobs` tables.

### 4. Restore existing Qdrant data (optional)

If you have the Qdrant backup with existing kb_1/kb_3 data:

```bash
bun run scripts/restore-qdrant.ts --source .docker/data/qdrant/
```

This merges kb_1 (771 points) and kb_3 (27 points) into the unified `knowledge_base` collection.

### 5. Place knowledge base documents

Extract the knowledge base zip to the configured source directory:

```bash
mkdir -p ./knowledge-base
unzip ~/Downloads/Knowledge\ Base.zip -d ./knowledge-base/
```

Expected structure:
```
knowledge-base/
├── UAE/
│   ├── 1-Business Setup/
│   ├── 2-Tax Compliance/
│   ├── 3-Accounting & Banking/
│   └── 4-Trademark & Legal/
└── KSA/
    ├── 1-Business Setup/
    ├── 2-Tax Compliance/
    ├── 3-Accounting & Banking/
    └── 4-Trademark & Legal/
```

## Usage

### Ingest the full corpus

```bash
bun run ingest -- all
```

### Ingest a single document

```bash
bun run ingest -- file ./knowledge-base/UAE/1-Business\ Setup/1.3_UAE_CompanyTypes.docx
```

### Ingest a country directory

```bash
bun run ingest -- dir ./knowledge-base/KSA/
```

### Check index status

```bash
bun run ingest -- status
```

### Remove a document

```bash
bun run ingest -- remove uae_business-setup_1.3_companyTypes
```

### Dry run (parse + chunk without embedding)

```bash
bun run ingest -- all --dry-run
```

## Verify Search Works

After ingestion, test that the chatbot can find content:

```bash
# Start the agents service
bun run dev --filter agents-service

# In another terminal, test search via the API
curl -X POST http://localhost:3000/api/v1/chat/message \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What documents do I need to set up a company in a Dubai free zone?",
    "conversationId": "test-123"
  }'
```

The response should reference specific content from the knowledge base documents.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Qdrant client not initialized` | Qdrant not running | `docker compose up -d` |
| `OPENAI_API_KEY not set` | Missing env variable | Add key to `.env` |
| `Vector dimension mismatch` | VECTOR_DIMENSION != 3072 | Set `VECTOR_DIMENSION=3072` in `.env` |
| `Unable to parse DOCX` | Corrupt file | Check file can be opened in Word; skip with `--verbose` |
| `Rate limit exceeded` | Too many embedding requests | Pipeline auto-retries with backoff |
| Search returns irrelevant results | Fake embeddings still active | Verify `generateQueryVector` uses real embeddings |

## Development

### Run tests

```bash
bun test --filter ingestion
```

### Run with verbose output

```bash
bun run ingest -- all --verbose
```

### Architecture overview

```
CLI (ingest.ts)
  → DocumentParser (officeparser / unpdf)
    → TextChunker (gpt-tokenizer + recursive splitter)
      → EmbeddingService (Vercel AI SDK → text-embedding-3-large)
        → QdrantUpserter (existing upsertSemanticDocuments)
          → IngestionTracker (PostgreSQL: kb_indexed_documents + kb_ingestion_jobs)
```
