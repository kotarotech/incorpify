# Data Model: Document Ingestion Pipeline

**Feature**: 002-document-ingestion | **Date**: 2026-03-03

## Entities

### 1. KnowledgeDocument (New — PostgreSQL, table: `knowledge_documents`)

Tracks which documents are currently indexed in Qdrant. One row per source document. Includes file storage metadata (mirroring `company_documents` pattern) for future file management capabilities.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | uuid | PK, auto-generated | Unique record identifier |
| knowledge_document_job_id | uuid | FK → knowledge_document_jobs.id, NULL | The ingestion job that last processed this document |
| document_id | varchar(255) | UNIQUE, NOT NULL | Deterministic ID derived from file path (e.g., `uae_business-setup_1.3_companyTypes`) |
| file_name | varchar(255) | NOT NULL | Original filename (e.g., `1.3_UAE_CompanyTypes.docx`) |
| file_path | text | NOT NULL | Relative path from knowledge base root |
| file_hash | varchar(64) | NOT NULL | SHA-256 hash of file content for change detection |
| file_size | bigint | NULL | File size in bytes |
| file_path_storage | text | NULL | Cloud storage path (for future cloud-hosted KB files) |
| file_storage_kind | varchar(50) | NULL | Storage provider (e.g., 'local', 's3', 'azure-blob') |
| file_bucket | varchar(255) | NULL | Storage bucket name (for cloud storage) |
| mime_type | varchar(100) | NULL | MIME type (e.g., 'application/vnd.openxmlformats-officedocument.wordprocessingml.document') |
| document_type | varchar(50) | NOT NULL | Document type (e.g., 'REGULATION', 'GUIDE', 'FAQ') |
| title | varchar(500) | NOT NULL | Extracted or derived document title |
| jurisdiction_code | varchar(50) | NULL | Derived: UAE_MAINLAND, FREE_ZONE, KSA_MAINLAND |
| country_code | varchar(5) | NULL | Derived: AE, SA |
| category | varchar(50) | NOT NULL | RagDocumentCategory enum value |
| source_type | varchar(20) | NOT NULL, DEFAULT 'INTERNAL' | RagSourceType enum value |
| source_authority | varchar(50) | NOT NULL, DEFAULT 'INCORPIFY' | Authority identifier |
| tags | text[] | DEFAULT '{}' | Array of tag strings |
| version | integer | NOT NULL, DEFAULT 1 | Document version number |
| chunk_count | integer | NOT NULL, DEFAULT 0 | Number of chunks in Qdrant |
| status | varchar(20) | NOT NULL, DEFAULT 'PENDING' | PENDING, INDEXED, FAILED, REMOVED |
| error_message | text | NULL | Error details if status = FAILED |
| last_indexed_at | timestamp | NULL | When document was last successfully indexed |
| created_at | timestamp | NOT NULL, DEFAULT now() | Record creation time |
| updated_at | timestamp | NOT NULL, DEFAULT now() | Last update time |

**Indexes**: document_id (unique), knowledge_document_job_id, status, category, country_code, jurisdiction_code

**Relationships**: Belongs to KnowledgeDocumentJob (via knowledge_document_job_id FK)

**Note on cost tracking**: Embedding token costs (prompt_tokens, total_tokens, cost_usd) are tracked in the existing `conversation_costs` table with `call_type = 'EMBEDDING'` rather than duplicated here. This follows the established pattern for all LLM cost tracking and allows unified cost reporting across chat and embedding operations.

---

### 2. KnowledgeDocumentJob (New — PostgreSQL, table: `knowledge_document_jobs`)

Records each ingestion run for audit and progress tracking. Individual document errors are tracked via each KnowledgeDocument's `error_message` and `status` fields (no separate error_log — avoids data duplication).

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | uuid | PK, auto-generated | Unique job identifier |
| job_type | varchar(20) | NOT NULL | SINGLE, BATCH, FULL_CORPUS |
| status | varchar(20) | NOT NULL, DEFAULT 'PENDING' | PENDING, PROCESSING, COMPLETED, FAILED |
| total_documents | integer | NOT NULL, DEFAULT 0 | Documents submitted for processing |
| processed_documents | integer | NOT NULL, DEFAULT 0 | Documents successfully processed |
| failed_documents | integer | NOT NULL, DEFAULT 0 | Documents that failed processing |
| skipped_documents | integer | NOT NULL, DEFAULT 0 | Documents skipped (unchanged hash, already indexed at same version) |
| total_chunks_created | integer | NOT NULL, DEFAULT 0 | Total chunks upserted to Qdrant |
| started_at | timestamp | NULL | Processing start time |
| completed_at | timestamp | NULL | Processing end time |
| triggered_by | varchar(100) | NOT NULL, DEFAULT 'CLI' | CLI, API, SYSTEM |
| created_at | timestamp | NOT NULL, DEFAULT now() | Record creation time |

**Indexes**: status, job_type, created_at

---

### 3. QdrantPayload (Existing — Qdrant Vector DB)

Already defined in `packages/shared/src/types/database/qdrant/payload.ts`. No schema changes needed.

| Field | Type | Description |
|-------|------|-------------|
| content | string | Chunk text content |
| title | string | Document/section title |
| document_id | string | Parent document ID (matches KnowledgeDocument.document_id) |
| chunk_index | number | Position within document (0-based) |
| chunk_id | string | Unique: `{document_id}_{chunk_index}` |
| jurisdiction_code | string \| null | UAE_MAINLAND, FREE_ZONE, KSA_MAINLAND |
| country_code | string \| null | AE, SA |
| business_area_codes | string[] \| null | DUBAI_MAINLAND, JAFZA, DMCC, etc. |
| zone_type | string \| null | MAINLAND, FREEZONE |
| category | RagDocumentCategory | BUSINESS_SETUP, TAX_COMPLIANCE, etc. |
| tags | string[] | Flexible tagging |
| effective_from | string \| null | ISO date |
| last_verified | string | ISO date |
| source_type | RagSourceType | LAW, CIRCULAR, FAQ, INTERNAL, THIRD_PARTY |
| source_authority | string | INCORPIFY, AE, SA |
| source_url | string \| null | Original source URL |
| confidence_score | number | 0-1 reliability score |
| priority_boost | number | Manual boost (default 0) |
| version | number | Schema version |
| indexed_at | string | ISO timestamp |

**Payload Indexes** (existing): jurisdiction_code, country_code, zone_type, category, source_type, source_authority, tags, business_area_codes

**New Index Needed**: `document_id` (keyword) — for efficient deletion of all chunks belonging to a document during re-ingestion.

---

### 4. QdrantDocument (Existing — Input Type)

Already defined in `packages/shared/src/types/database/qdrant/payload.ts`. Used as input to `upsertSemanticDocuments()`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | string | Yes | Point ID (= chunk_id) |
| vector | number[] | Yes | 3072-dimension embedding vector |
| content | string | Yes | Chunk text |
| title | string | Yes | Document title |
| documentId | string | Yes | Parent document ID |
| chunkIndex | number | Yes | Chunk position |
| chunkId | string | Yes | `{documentId}_{chunkIndex}` |
| jurisdictionCode | string \| null | No | Derived from path |
| countryCode | string \| null | No | Derived from path |
| businessAreaCodes | string[] \| null | No | Derived from content/path |
| zoneType | string \| null | No | Derived from path |
| category | RagDocumentCategory | Yes | Derived from path |
| tags | string[] | No | Derived from path/content |
| sourceType | RagSourceType | Yes | From metadata or default |
| sourceAuthority | string | Yes | From metadata or default |
| confidenceScore | number | No | Default 1.0 |
| priorityBoost | number | No | Default 0 |
| version | number | No | Default 1 |

---

## State Transitions

### KnowledgeDocument.status

```
PENDING → INDEXED    (successful ingestion)
PENDING → FAILED     (ingestion error)
INDEXED → PENDING    (re-ingestion triggered)
INDEXED → REMOVED    (manual removal)
FAILED  → PENDING    (retry triggered)
```

### KnowledgeDocumentJob.status

```
PENDING → PROCESSING  (job starts)
PROCESSING → COMPLETED (all documents processed)
PROCESSING → FAILED    (unrecoverable error)
```

Note: Individual document failures within a batch do NOT fail the job — they increment `failed_documents` and the failing document's `error_message` is set on its KnowledgeDocument record. The job completes with partial success. Unchanged documents increment `skipped_documents`.

---

## Validation Rules

### KnowledgeDocument
- `document_id` must be unique — derived deterministically from file path
- `file_hash` must be SHA-256 (64 hex characters)
- `category` must be a valid `RagDocumentCategory` enum value
- `source_type` must be a valid `RagSourceType` enum value
- `document_type` must be a non-empty string
- `version` must be >= 1
- `chunk_count` must be >= 0
- `status` must be one of: PENDING, INDEXED, FAILED, REMOVED
- `knowledge_document_job_id` must reference an existing job when set

### KnowledgeDocumentJob
- `processed_documents + failed_documents + skipped_documents <= total_documents`
- `completed_at` must be NULL when status is PENDING or PROCESSING
- `started_at` must be set when status transitions to PROCESSING

### Chunk Generation
- Chunk size: ~500 tokens (± 10%)
- Overlap: ~50 tokens between adjacent chunks
- Minimum chunk size: 50 tokens (skip very small remnants)
- Maximum chunk size: 600 tokens (hard limit)
- chunk_index: 0-based, sequential per document
- chunk_id format: `{document_id}_{chunk_index}` (deterministic)

### Vector Embedding
- Dimension: 3072 (text-embedding-3-large)
- All vectors must be normalized (unit length)
- Empty chunks must not be embedded

---

## Relationship to Existing Entities

```
KnowledgeDocumentJob 1──→ N KnowledgeDocument (via knowledge_document_job_id FK)
KnowledgeDocument 1──→ N QdrantPayload (via document_id)
EmbeddingService ──→ conversation_costs (via call_type='EMBEDDING')

CompanyDocument (existing) ──future──→ QdrantCompanyPayload (P3: client doc OCR)
```

The `knowledge_documents` table is separate from the existing `company_documents` table:
- `company_documents` tracks client-uploaded files (KYC, legal, people documents) with company/organization scoping
- `knowledge_documents` tracks knowledge base content managed by administrators
- Both will eventually produce Qdrant entries, but with different payload types, `source_type` values, and scoping rules

### Embedding Cost Tracking

Embedding costs are tracked in the existing `conversation_costs` table (not duplicated on KnowledgeDocument):
- `call_type = 'EMBEDDING'` (new enum value alongside existing 'CHAT', 'TOOL')
- `model_provider = 'openai'`, `model_name = 'text-embedding-3-large'`
- `prompt_tokens` = total tokens embedded, `completion_tokens` = 0
- `cost_usd` = calculated from token count × per-token rate
- `conversation_id` = NULL (no conversation context), `user_id` = 'SYSTEM'

---

## Future Entities (P3 — Client Document OCR)

### QdrantCompanyDocument (Future — Input Type)

When client-uploaded documents (passports, trade licenses, insurance policies) are processed via OCR and embedded, they will use a dedicated Qdrant input type that extends QdrantDocument with tenant scoping fields.

| Field | Type | Description |
|-------|------|-------------|
| *(all QdrantDocument fields)* | | Inherited from base type |
| organizationId | string | Owning organization (tenant isolation) |
| companyId | string | Owning company within organization |
| companyDocumentId | string | FK to company_documents table |

### QdrantCompanyPayload (Future — Qdrant Payload)

Extends QdrantPayload with tenant-scoping fields for client document isolation.

| Field | Type | Description |
|-------|------|-------------|
| *(all QdrantPayload fields)* | | Inherited from base type |
| organization_id | string | Owning organization |
| company_id | string | Owning company |
| company_document_id | string | FK to company_documents |

### RagService Dual-Scope Search (Future)

When client document search is implemented, the RagService will support two search scopes:
- **Dashboard chatbot** (authenticated user): Searches KB documents + company documents scoped to the user's organization/company
- **Landing page chatbot** (anonymous): Searches KB documents only (no company documents)

The scope is determined by the presence of `X-Space-Id` / `X-Organization-Id` headers in the request.

---

## Reference: Cloudflare Markdown for Agents

For future document ingestion from web sources, [Cloudflare's Markdown for Agents](https://developers.cloudflare.com/speed/other/agents/) feature converts HTML to clean markdown when an AI agent sends an `Accept: text/markdown` header. This could simplify web scraping for government regulation pages in future iterations.
