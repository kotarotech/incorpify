# CLI Contract: Ingestion Pipeline

**Feature**: 002-document-ingestion | **Date**: 2026-03-03

## Command: `bun run ingest`

Script location: `packages/agents-service/scripts/ingest.ts`

### Synopsis

```
bun run ingest -- <command> [options]
```

### Commands

#### `ingest file <path>`
Ingest a single document file.

```
bun run ingest -- file ./.kb_data/UAE/1-Business\ Setup/1.3_UAE_CompanyTypes.docx
```

**Input**: Absolute or relative path to a .docx or .pdf file.
**Output**:
```
Ingesting: 1.3_UAE_CompanyTypes.docx
  Parsed: 4,231 characters extracted
  Chunked: 12 chunks (avg 487 tokens, overlap 50)
  Embedded: 12 vectors (3072d)
  Upserted: 12 points to knowledge_base
  Metadata: country=AE, jurisdiction=UAE_MAINLAND, category=BUSINESS_SETUP

Done. Document indexed as uae_business-setup_1.3_companyTypes (v1, 12 chunks)
```

**Exit codes**: 0 = success, 1 = parse error, 2 = embedding error, 3 = Qdrant error

---

#### `ingest dir <path>`
Ingest all documents in a directory (recursive).

```
bun run ingest -- dir ./.kb_data/UAE/
```

**Output**:
```
Scanning: ./.kb_data/UAE/
  Found: 54 documents (.docx: 53, .pdf: 1)

Processing [1/54] 1.3_UAE_CompanyTypes.docx... OK (12 chunks)
Processing [2/54] 1.UAE_BusinessSetup.docx... OK (8 chunks)
...
Processing [54/54] 4.2_UAE_DataProtection.docx... FAILED (corrupt file)

Summary:
  Processed: 53/54 documents
  Failed: 1 document
  Chunks created: 612
  Time: 2m 34s

Failed documents:
  - 4.2_UAE_DataProtection.docx: Error: Unable to parse DOCX (corrupt ZIP header)
```

**Exit codes**: 0 = all succeeded, 1 = partial failure (some docs failed), 2 = total failure

---

#### `ingest all`
Re-ingest the entire knowledge base corpus from the configured source directory.

```
bun run ingest -- all
```

**Behavior**: Same as `ingest dir` pointed at the root knowledge base directory. Prompts for confirmation before proceeding if documents already exist in the index.

**Options**:
- `--force` — Skip confirmation prompt
- `--clean` — Delete all existing points before re-ingesting

---

#### `ingest status`
Show the current state of the knowledge base index.

```
bun run ingest -- status
```

**Output**:
```
Knowledge Base Index Status
===========================
Collection: knowledge_base
Vector dimension: 3072
Total points: 798
Indexed documents: 67

Documents by jurisdiction:
  UAE_MAINLAND: 45 documents (612 chunks)
  KSA_MAINLAND: 22 documents (186 chunks)

Documents by category:
  BUSINESS_SETUP: 18 documents
  TAX_COMPLIANCE: 12 documents
  ACCOUNTING_BANKING: 10 documents
  ...

Recent ingestion jobs:
  2026-03-03 14:30 | FULL_CORPUS | COMPLETED | 67 docs, 798 chunks | 4m 12s
  2026-03-02 09:15 | SINGLE     | COMPLETED | 1 doc, 12 chunks    | 8s

Last indexed: 2026-03-03 14:30:00 UTC
```

---

#### `ingest remove <document-id>`
Remove a document and all its chunks from the index.

```
bun run ingest -- remove uae_business-setup_1.3_companyTypes
```

**Output**:
```
Removing: uae_business-setup_1.3_companyTypes
  Deleted: 12 chunks from knowledge_base collection
  Updated: knowledge_documents status → REMOVED

Done. Document removed from index.
```

---

### Global Options

| Option | Description |
|--------|-------------|
| `--verbose` | Show detailed processing output |
| `--dry-run` | Parse and chunk without embedding or upserting |
| `--source-dir <path>` | Override knowledge base source directory |
| `--help` | Show command help |

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | Yes | — | PostgreSQL connection string |
| `QDRANT_URL` | No | `http://localhost:6333` | Qdrant instance URL |
| `QDRANT_COLLECTION_NAME` | No | `knowledge_base` | Target collection |
| `VECTOR_DIMENSION` | No | `3072` | Embedding vector dimension |
| `OPENAI_API_KEY` | Yes | — | For text-embedding-3-large |
| `KB_SOURCE_DIR` | No | `./.kb_data` | Knowledge base root directory |
