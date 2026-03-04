# Research: Document Ingestion Pipeline

**Feature**: 002-document-ingestion | **Date**: 2026-03-03

## R1: Document Parsing Libraries (Bun-Compatible)

**Decision**: Use `officeparser` for DOCX and `unpdf` for PDF extraction.

**Rationale**:
- `officeparser` — Pure JS, no native deps, supports .docx/.pptx/.odt. Returns plain text. Works in Bun without compatibility issues. Actively maintained.
- `unpdf` — Built on Mozilla's pdf.js (via `pdfjs-dist`). Pure JS, Bun-native. Extracts text per page. Used by the Vercel AI SDK team (`ai` package references it).
- Both libraries are lightweight and require no system-level dependencies (no LibreOffice, no poppler).

**Alternatives Considered**:
| Library | Why Rejected |
|---------|-------------|
| `mammoth` | Converts DOCX to HTML/markdown — adds unnecessary overhead when we only need plain text. Also has some Bun compatibility issues with xmldom. |
| `pdf-parse` | Wrapper around older pdfjs. Unmaintained (last publish 2021). `unpdf` is the modern replacement. |
| `docx` | Creates/modifies DOCX — not a parser. |
| `libreoffice-convert` | Requires LibreOffice installed. Heavy system dependency. |

---

## R2: Text Chunking Strategy

**Decision**: Use `gpt-tokenizer` for token counting + custom recursive text splitter.

**Rationale**:
- `gpt-tokenizer` — Pure JS BPE tokenizer compatible with OpenAI's `text-embedding-3-large` model. Accurate token counting without API calls.
- Custom recursive splitter — Split on paragraph breaks (`\n\n`), then sentences, then at token limit. Simpler than importing a heavy chunking library. Preserves semantic boundaries.
- Target: ~500 tokens per chunk with ~50 token overlap (10%).
- Chunk ID format: `{document_id}_{chunk_index}` for deterministic IDs enabling idempotent upserts.

**Alternatives Considered**:
| Library | Why Rejected |
|---------|-------------|
| `llm-splitter` | Not widely adopted, limited documentation. Custom splitter is ~50 lines and fully controllable. |
| `langchain` text splitters | Massive dependency tree. Violates Principle X (Simplicity). |
| `tiktoken` (OpenAI) | WASM-based, Bun compatibility uncertain. `gpt-tokenizer` is pure JS and proven in Bun. |

---

## R3: Embedding Generation

**Decision**: Use Vercel AI SDK v6 `embed()` / `embedMany()` with OpenAI `text-embedding-3-large`.

**Rationale**:
- Vercel AI SDK v6 (`ai@^6.0.0`) is already installed and is the mandated AI integration layer (Constitution Principle VII).
- `embed()` for single query embedding (search), `embedMany()` for batch chunk embedding (ingestion).
- Model: `text-embedding-3-large` produces 3072-dimension vectors matching existing Qdrant data.
- Provider: OpenAI (via `@ai-sdk/openai` already in dependencies). Azure OpenAI also supported.
- Batch size: `embedMany()` handles batching internally. OpenAI allows up to 2048 inputs per request for embedding models.

**Alternatives Considered**:
| Approach | Why Rejected |
|----------|-------------|
| Direct OpenAI SDK (`openai` npm) | Bypasses Vercel AI SDK, violates Constitution Principle VII. |
| Azure OpenAI embedding endpoint | Possible but OpenAI direct is simpler for embeddings. Azure can be added later via provider config. |
| Custom embedding service | Over-engineering. YAGNI (Principle X). |

---

## R4: Qdrant Integration — Existing Infrastructure

**Decision**: Extend existing Qdrant infrastructure in `@incorpify/shared`. No new dependencies needed.

**Rationale**:
- `@qdrant/js-client-rest` is already installed and fully integrated.
- Existing functions cover core needs:
  - `upsertSemanticDocuments()` — batch upsert with `QdrantDocument` input type
  - `deleteSemanticDocuments()` — batch delete by IDs
  - `initializeSemanticDatabase()` — collection creation with indexes
  - `semanticHealthCheck()` — connection and collection verification
- Payload schema (`QdrantPayload`) already defines all 20+ metadata fields matching the Qdrant backup data.
- Payload indexes already exist for: `jurisdiction_code`, `country_code`, `zone_type`, `category`, `source_type`, `source_authority`, `tags`, `business_area_codes`.
- Collection name defaults to `knowledge_base` (single collection, aligning with FR-005).

**What needs to change**:
1. `VECTOR_DIMENSION` default: `1536` → `3072`
2. `generateQueryVector()`: Replace hash-based placeholder with real `embed()` call
3. Add `document_id` payload index for efficient document-level deletion during re-ingestion

---

## R5: Collection Migration Strategy (kb_1 + kb_3 → knowledge_base)

**Decision**: Merge legacy collections into single `knowledge_base` collection during initial setup. Use metadata-based filtering and source authority weighting at query time.

**Rationale**:
- The kb_1/kb_3 split is a legacy artifact from Azure Search migration, not a meaningful boundary.
- The codebase already uses a single `knowledge_base` collection name.
- All 798 existing points share identical vector dimensions (3072) and payload schemas.
- Source authority weighting (FR-005a) prevents `INTERNAL` documents from overriding `LAW` documents — handled at query time via score boosting on `source_type` and `confidence_score`.
- The existing `restore-qdrant.ts` script can be adapted to load both collections into one.

**Migration steps**:
1. Load kb_1 snapshot (771 points) into `knowledge_base` collection
2. Load kb_3 snapshot (27 points) into same collection
3. Verify total: 798 points with correct dimensions and payload
4. Create payload indexes

---

## R6: Metadata Derivation from Folder Structure

**Decision**: Parse the knowledge base folder hierarchy to auto-derive `jurisdiction_code`, `country_code`, `category`, and `tags`.

**Rationale**:
- Source documents follow a consistent structure: `{country}/{topic}/{subtopic}/{version}_{name}.docx`
- Example: `UAE/1-Business Setup/1.1-Company Types/1.3_UAE_CompanyTypes.docx`
- Mapping rules:
  - `UAE/` → `country_code: 'AE'`, `jurisdiction_code: 'UAE_MAINLAND'`
  - `KSA/` → `country_code: 'SA'`, `jurisdiction_code: 'KSA_MAINLAND'`
  - Topic folders map to `RagDocumentCategory` enum values
  - Subtopic folders become `tags`
  - Version numbers extracted from filename prefix (e.g., `1.3_` → version 3)

**Folder-to-Category Mapping**:
| Folder Pattern | Category |
|---------------|----------|
| `Business Setup` | BUSINESS_SETUP |
| `Tax Compliance` | TAX_COMPLIANCE |
| `Accounting*Banking` | ACCOUNTING_BANKING |
| `Trademark*Legal` | TRADEMARK_LEGAL |
| `Visa*Immigration` | VISA_IMMIGRATION |
| `Labor*Law` | LABOR_LAW |
| `Licensing` | LICENSING |
| `Corporate*Governance` | CORPORATE_GOVERNANCE |

---

## R7: Ingestion Tracking — PostgreSQL vs In-Memory

**Decision**: Track ingestion jobs and indexed documents in PostgreSQL using new Drizzle tables.

**Rationale**:
- The existing `company_documents` table tracks client-uploaded files (with KYC, versioning, storage metadata) but not knowledge base ingestion state.
- Ingestion tracking needs persistence across service restarts for: job history, error logs, document versions, chunk counts.
- Two new tables:
  - `knowledge_document_jobs` — tracks batch processing runs (status, progress, skipped)
  - `knowledge_documents` — tracks what's currently in Qdrant (document_id, chunk_count, version, last_indexed_at, metadata, file storage fields)
- Aligns with Principle I (shared PostgreSQL for structured data, Qdrant for vectors only).

**Alternatives Considered**:
| Approach | Why Rejected |
|----------|-------------|
| In-memory tracking | Lost on restart. No audit trail. |
| Qdrant metadata only | Can't query Qdrant for "all documents" or "failed jobs" efficiently. |
| File-based tracking | Fragile, not queryable, doesn't support concurrent access. |

---

## R8: CLI vs API Endpoint for Triggering Ingestion

**Decision**: CLI script (Bun executable) for initial implementation; admin API endpoint as a follow-up.

**Rationale**:
- CLI is simpler to implement and aligns with the current workflow (admins SSH into servers or run locally).
- Constitution Principle II mandates Fastify endpoints have full Zod validation — an API endpoint is more work and not needed for P1.
- The CLI can be a standalone script in `packages/agents-service/scripts/` that imports shared infrastructure.
- An admin API endpoint can be added later (P2, User Story 3) when the admin dashboard is built.

**CLI interface**:
```
bun run ingest -- --path <file-or-dir>     # Ingest specific file or directory
bun run ingest -- --all                     # Re-ingest entire corpus
bun run ingest -- --status                  # Show indexed documents
bun run ingest -- --remove <document-id>    # Remove document from index
```

---

## R9: Client Document OCR (P3 — Future)

**Decision**: Defer OCR implementation. Design pipeline with extension point for OCR step.

**Rationale**:
- P3 priority per spec. Current focus is knowledge base documents (all .docx/.pdf with extractable text).
- Pipeline architecture: `parse → chunk → embed → store` — OCR would be an alternative `parse` step.
- When implemented, OCR provider options: Azure AI Document Intelligence, Google Cloud Vision, Tesseract.js (local).
- The `company_documents` table already has an `ocrStatus` field, confirming OCR was planned.

---

## Open Questions (Deferred)

### Duplicate Document Handling
- **Status**: Deferred per user decision — needs team consultation (Kai, Alejandro).
- **Context**: Knowledge base contains versioned files (`1.KSA_BusinessSetup.docx`, `1.3_KSA_BusinessSetup.docx`) and apparent copies (`Copy of 3.KSA_1.3_Accounting&Banking.docx`).
- **Options documented in spec.md** Clarifications section.
- **Current approach**: Ingest all files, tag with version, revisit after team input.
