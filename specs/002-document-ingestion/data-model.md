# Data Model: Document Ingestion Pipeline

**Feature**: 002-document-ingestion | **Date**: 2026-03-03

## Entities

### 1. KbIndexedDocument (New — PostgreSQL)

Tracks which documents are currently indexed in Qdrant. One row per source document.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | uuid | PK, auto-generated | Unique record identifier |
| document_id | varchar(255) | UNIQUE, NOT NULL | Deterministic ID derived from file path (e.g., `uae_business-setup_1.3_companyTypes`) |
| file_name | varchar(500) | NOT NULL | Original filename (e.g., `1.3_UAE_CompanyTypes.docx`) |
| file_path | text | NOT NULL | Relative path from knowledge base root |
| file_hash | varchar(64) | NOT NULL | SHA-256 hash of file content for change detection |
| title | varchar(500) | NOT NULL | Extracted or derived document title |
| jurisdiction_code | varchar(50) | NULL | Derived: UAE_MAINLAND, FREE_ZONE, KSA_MAINLAND |
| country_code | varchar(5) | NULL | Derived: AE, SA |
| category | varchar(50) | NOT NULL | RagDocumentCategory enum value |
| source_type | varchar(20) | NOT NULL, DEFAULT 'INTERNAL' | RagSourceType enum value |
| source_authority | varchar(50) | NOT NULL, DEFAULT 'INCORPIFY' | Authority identifier |
| tags | text[] | DEFAULT '{}' | Array of tag strings |
| version | integer | NOT NULL, DEFAULT 1 | Document version number |
| chunk_count | integer | NOT NULL, DEFAULT 0 | Number of chunks in Qdrant |
| total_tokens | integer | NULL | Total token count across all chunks |
| status | varchar(20) | NOT NULL, DEFAULT 'PENDING' | PENDING, INDEXED, FAILED, REMOVED |
| error_message | text | NULL | Error details if status = FAILED |
| last_indexed_at | timestamp | NULL | When document was last successfully indexed |
| created_at | timestamp | NOT NULL, DEFAULT now() | Record creation time |
| updated_at | timestamp | NOT NULL, DEFAULT now() | Last update time |

**Indexes**: document_id (unique), status, category, country_code, jurisdiction_code

**Relationships**: None (standalone tracking table)

---

### 2. KbIngestionJob (New — PostgreSQL)

Records each ingestion run for audit and progress tracking.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | uuid | PK, auto-generated | Unique job identifier |
| job_type | varchar(20) | NOT NULL | SINGLE, BATCH, FULL_CORPUS |
| status | varchar(20) | NOT NULL, DEFAULT 'PENDING' | PENDING, PROCESSING, COMPLETED, FAILED |
| total_documents | integer | NOT NULL, DEFAULT 0 | Documents submitted for processing |
| processed_documents | integer | NOT NULL, DEFAULT 0 | Documents successfully processed |
| failed_documents | integer | NOT NULL, DEFAULT 0 | Documents that failed processing |
| total_chunks_created | integer | NOT NULL, DEFAULT 0 | Total chunks upserted to Qdrant |
| error_log | jsonb | DEFAULT '[]' | Array of `{ documentId, error, timestamp }` |
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
| document_id | string | Parent document ID (matches KbIndexedDocument.document_id) |
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

### KbIndexedDocument.status

```
PENDING → INDEXED    (successful ingestion)
PENDING → FAILED     (ingestion error)
INDEXED → PENDING    (re-ingestion triggered)
INDEXED → REMOVED    (manual removal)
FAILED  → PENDING    (retry triggered)
```

### KbIngestionJob.status

```
PENDING → PROCESSING  (job starts)
PROCESSING → COMPLETED (all documents processed)
PROCESSING → FAILED    (unrecoverable error)
```

Note: Individual document failures within a batch do NOT fail the job — they increment `failed_documents` and log to `error_log`. The job completes with partial success.

---

## Validation Rules

### KbIndexedDocument
- `document_id` must be unique — derived deterministically from file path
- `file_hash` must be SHA-256 (64 hex characters)
- `category` must be a valid `RagDocumentCategory` enum value
- `source_type` must be a valid `RagSourceType` enum value
- `version` must be >= 1
- `chunk_count` must be >= 0
- `status` must be one of: PENDING, INDEXED, FAILED, REMOVED

### KbIngestionJob
- `processed_documents + failed_documents <= total_documents`
- `completed_at` must be NULL when status is PENDING or PROCESSING
- `started_at` must be set when status transitions to PROCESSING
- `error_log` entries must have shape: `{ documentId: string, error: string, timestamp: string }`

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
KbIngestionJob 1──→ N KbIndexedDocument (via job processing)
KbIndexedDocument 1──→ N QdrantPayload (via document_id)

CompanyDocument (existing) ──future──→ QdrantPayload (P3: client doc OCR)
```

The `KbIndexedDocument` table is separate from the existing `company_documents` table:
- `company_documents` tracks client-uploaded files (KYC, legal, people documents) with company/organization scoping
- `kb_indexed_documents` tracks knowledge base content managed by administrators
- Both will eventually produce `QdrantPayload` entries in Qdrant, but with different `source_type` values and scoping rules
