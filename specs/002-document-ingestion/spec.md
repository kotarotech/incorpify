# Feature Specification: Document Ingestion Pipeline

**Feature Branch**: `002-document-ingestion`
**Created**: 2026-03-03
**Status**: Draft
**Input**: User description: "Build a document ingestion pipeline for the Incorpify knowledge base that parses, chunks, embeds, and stores documents in Qdrant for semantic search by the AI chatbot."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - AI Chatbot Returns Accurate Answers from Knowledge Base (Priority: P1)

A user asks the AI chatbot a question about UAE business incorporation (e.g., "What documents do I need to set up a company in a Dubai free zone?"). The chatbot searches the knowledge base, finds the relevant regulation chunks, and returns an accurate, sourced answer. Today this is broken because the search function uses fake embeddings.

**Why this priority**: The chatbot is the core product differentiator. Without working semantic search, it cannot reference the 109 knowledge base documents that contain UAE/KSA regulations, tax guidance, and incorporation procedures. Fixing this unblocks the entire AI consultation experience.

**Independent Test**: Can be tested by asking the chatbot a domain-specific question and verifying the response references real content from the knowledge base documents, not hallucinated information.

**Acceptance Scenarios**:

1. **Given** a knowledge base with indexed documents, **When** a user asks the chatbot about UAE VAT rules, **Then** the chatbot retrieves relevant chunks from the knowledge base and provides an accurate answer citing specific regulations.
2. **Given** the chatbot receives a question about KSA business setup, **When** the system performs semantic search, **Then** the top results are relevant to the query topic, jurisdiction, and category.
3. **Given** a question that spans multiple documents, **When** the system searches, **Then** it returns chunks from multiple relevant sources ranked by relevance.
4. **Given** a question in Arabic, **When** the system performs semantic search, **Then** it returns relevant results regardless of the query language.

---

### User Story 2 - Administrator Ingests New Knowledge Base Documents (Priority: P1)

An administrator (Alejandro, Kai, or operations team) has new or updated government regulations, internal guides, or compliance documents. They trigger the ingestion pipeline to process these documents — extracting text, splitting into chunks, generating embeddings, and storing in the vector database with appropriate metadata (jurisdiction, category, source type). The new content becomes searchable by the chatbot immediately after ingestion.

**Why this priority**: The knowledge base must stay current. Government regulations change, new jurisdictions are added (KSA expansion), and Incorpify's own guides are updated. Without an ingestion pipeline, new content cannot enter the system — which is the exact problem today ("we cannot update or upload new things").

**Independent Test**: Can be tested by adding a new document to the knowledge base, running the ingestion pipeline, and verifying the chatbot can find and reference the new content.

**Acceptance Scenarios**:

1. **Given** a new Word document about KSA tax compliance, **When** the administrator triggers ingestion, **Then** the document is parsed, chunked, embedded, and stored with correct metadata (jurisdiction: KSA_MAINLAND, category: TAX_COMPLIANCE).
2. **Given** an updated version of an existing document, **When** the administrator re-ingests it, **Then** the old chunks are replaced with new ones and the version number increments.
3. **Given** a batch of 20 new documents, **When** the administrator triggers a full corpus re-ingestion, **Then** all documents are processed and the total indexed count increases accordingly.
4. **Given** ingestion is in progress, **When** the administrator checks status, **Then** they see progress information (documents processed, chunks created, errors encountered).

---

### User Story 3 - Administrator Manages Knowledge Base Content (Priority: P2)

An administrator wants to see what's currently in the knowledge base — which documents are indexed, when they were last updated, how many chunks each produced, and whether any failed during ingestion. They can also remove outdated documents from the index.

**Why this priority**: Visibility into the knowledge base is essential for maintaining quality. Without it, the team cannot verify that content is correctly indexed or troubleshoot search quality issues.

**Independent Test**: Can be tested by listing indexed documents, checking metadata for a specific document, and removing a document from the index.

**Acceptance Scenarios**:

1. **Given** an administrator, **When** they request a list of indexed documents, **Then** they see document names, chunk counts, last indexed date, version, and jurisdiction for each entry.
2. **Given** an outdated document in the index, **When** the administrator removes it, **Then** all associated chunks are deleted from the vector database.
3. **Given** a document that failed during ingestion, **When** the administrator views failed documents for that job, **Then** they see each document's error message and the specific point of failure.

---

### User Story 4 - Client Documents Become Searchable by Dashboard Chatbot (Priority: P3)

When a customer uploads a document through the dashboard (passport, trade license, insurance policy), the system processes it and makes its content searchable by the dashboard chatbot. The chatbot can then reference specific details from the customer's own documents when answering questions (e.g., "When does my trade license expire?").

**Why this priority**: This extends the pipeline to client-uploaded documents, which is a separate use case from the knowledge base. It requires OCR for scanned documents and tenant-scoped access control. It builds on the same pipeline infrastructure but adds complexity.

**Independent Test**: Can be tested by uploading a PDF document through the dashboard, then asking the chatbot a question that can only be answered from that document's content.

**Acceptance Scenarios**:

1. **Given** a customer uploads a trade license PDF, **When** the document is processed, **Then** the extracted text is embedded and stored with the customer's organization and company scope.
2. **Given** a customer asks the chatbot "What is my license number?", **When** the system searches, **Then** it finds the answer from the customer's own uploaded document, not from another customer's documents.
3. **Given** a scanned passport image, **When** the system processes it, **Then** it extracts text via OCR and indexes it with the correct document type metadata.
4. **Given** a customer in Organization A, **When** the chatbot searches for documents, **Then** it never returns results from Organization B's documents.

---

### Edge Cases

- What happens when a document is corrupted or cannot be parsed (e.g., password-protected PDF, malformed DOCX)?
- How does the system handle documents with mixed languages (English and Arabic in the same document)?
- What happens when the embedding API is temporarily unavailable during ingestion?
- How does the system handle duplicate documents (same content, different filenames)?
- What happens when a document is extremely large (100+ pages)?
- How does the system handle ingestion of a document that already exists with a higher version number?
- What happens when Qdrant is unreachable during ingestion?
- How does the system handle documents with tables, charts, or images embedded in text?

## Clarifications

### Session 2026-03-03

- Q: Should new documents go into the existing separate collections (kb_1, kb_3) or a single merged collection? → A: Single collection with metadata-based filtering (Option B). The kb_1/kb_3 split is a legacy artifact from Azure Search migration, not a meaningful boundary. Source authority weighting (source_type: LAW ranks higher than INTERNAL) prevents internal guides from overriding official regulations in search results — this is handled at query time via scoring, not collection separation.
- Q: How should duplicate/versioned documents be handled? → A: **DEFERRED — needs team input.** The knowledge base contains multiple versions of the same document (e.g., `1.KSA_BusinessSetup.docx`, `1.3_KSA_BusinessSetup.docx`) and apparent copies (`Copy of 3.KSA_1.3_Accounting&Banking.docx`). Options: (A) ingest only the highest version, skip older and "Copy of" files; (B) ingest all versions, tag with version number, let search prefer latest; (C) ingest all but only make latest searchable, archive older for audit. Recommend discussing with Kai (who maintains the knowledge base) and Alejandro before deciding.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST extract text content from Word documents (.docx) and PDF files (.pdf).
- **FR-002**: System MUST split extracted text into chunks of approximately 500 tokens with overlapping context between adjacent chunks to preserve semantic continuity.
- **FR-003**: System MUST generate vector embeddings for each chunk using the same embedding model and dimensions (3072) as the existing indexed data.
- **FR-004**: System MUST store embedded chunks in the vector database with metadata including: document ID, chunk index, title, jurisdiction code, country code, category, tags, source type, source authority, confidence score, and version.
- **FR-005**: System MUST store all documents (knowledge base and client) in a single vector collection, using metadata fields (jurisdiction_code, source_type, country_code, category) for filtered and scoped search. The legacy kb_1/kb_3 collection split MUST be merged during initial setup.
- **FR-005a**: System MUST apply source authority weighting in search results so that official sources (source_type: LAW) rank higher than internal guides (source_type: INTERNAL) when both match a query. This prevents internal documentation from overriding authoritative regulatory content.
- **FR-006**: System MUST replace the current non-functional search query mechanism with real embedding-based vector search, so that the AI chatbot can retrieve relevant knowledge base content.
- **FR-007**: System MUST support incremental updates — ingesting new documents without re-processing the entire corpus.
- **FR-008**: System MUST support version-aware updates — when a document is re-ingested, old chunks are replaced and the version number increments.
- **FR-009**: System MUST provide a way to trigger ingestion for a single document, a directory of documents, or the entire corpus.
- **FR-010**: System MUST report ingestion progress and errors, including which documents succeeded, which failed, and why.
- **FR-011**: System MUST support listing all indexed documents with their metadata, chunk counts, and last-indexed timestamps.
- **FR-012**: System MUST support removing a document and all its associated chunks from the index.
- **FR-013**: System MUST automatically derive metadata (jurisdiction, country, category) from the document's folder path and filename when not explicitly provided.
- **FR-014**: System MUST scope client-uploaded documents to the owning organization and company, ensuring tenant isolation during search.
- **FR-015**: System MUST handle ingestion failures gracefully — a single document failure must not abort the entire batch.

### Key Entities

- **Source Document**: A file (DOCX or PDF) containing knowledge base content or client-uploaded material. Has a path, filename, jurisdiction, category, version, and processing status.
- **Chunk**: A segment of extracted text from a source document, approximately 500 tokens, with overlap from adjacent chunks. Each chunk has a unique ID, the parent document ID, a chunk index, and the extracted text content.
- **Embedding**: A vector representation of a chunk, generated by the embedding model. Has 3072 dimensions and is stored alongside the chunk's metadata in the vector database.
- **Ingestion Job**: A record of a processing run, tracking which documents were submitted, their processing status (pending, processing, completed, failed), error messages, and timestamps.
- **Indexed Document**: A record tracking which documents are currently in the vector database, their version, chunk count, last indexed timestamp, and metadata.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The AI chatbot returns relevant, sourced answers for 90% of questions about topics covered in the knowledge base, compared to 0% today.
- **SC-002**: New documents added to the knowledge base become searchable by the chatbot within 5 minutes of ingestion.
- **SC-003**: The full corpus of 109 source documents can be re-ingested from scratch in under 30 minutes.
- **SC-004**: Search results for a given query return relevant chunks in the top 5 results at least 85% of the time.
- **SC-005**: Ingestion of a single document completes in under 30 seconds.
- **SC-006**: The system correctly derives jurisdiction and category metadata for at least 95% of documents in the existing knowledge base without manual input.
- **SC-007**: Client-uploaded documents are never returned in search results for users outside the owning organization (100% tenant isolation).
- **SC-008**: The pipeline handles all 109 existing documents (108 DOCX + 1 PDF) without errors.

## Assumptions

- The existing 798 Qdrant points (771 in kb_1, 27 in kb_3) were embedded using a 3072-dimension model and will remain compatible with new embeddings generated by the same model family.
- The embedding API (OpenAI or equivalent) will be available and the team has API keys with sufficient quota for the knowledge base volume.
- The knowledge base folder structure (country/category/subcategory) is stable and can be used to derive metadata automatically.
- Document versioning follows the existing pattern where filenames include version numbers (e.g., `1.3_KSA_BusinessSetup.docx` vs `1.4_KSA_BusinessSetup.docx`).
- Client document OCR (User Story 4) will use a cloud OCR service; the choice of provider will be determined during planning.
- The Qdrant instance is running locally during development and will be deployed alongside the backend services in production.

## Dependencies

- Access to an embedding API with a model that produces 3072-dimension vectors compatible with the existing Qdrant data.
- Qdrant instance accessible from the backend services (local for dev, deployed for staging/production).
- The existing Qdrant backup data (kb_1 and kb_3 collections) must be loadable into the development Qdrant instance.
- The source knowledge base documents (109 files) must be accessible to the ingestion pipeline.
- For client document OCR (P3), access to a cloud OCR service API.

## Scope Boundaries

### In Scope

- Text extraction from DOCX and PDF files.
- Chunking with configurable size and overlap.
- Embedding generation using a 3072-dimension model.
- Qdrant upsert with the existing metadata schema.
- Fixing the broken query path (replacing placeholder embeddings with real ones).
- Updating the vector dimension configuration.
- CLI or admin interface for triggering and monitoring ingestion.
- Incremental and version-aware document updates.
- Metadata derivation from folder structure and filenames.
- Tenant-scoped storage for client documents.

### Out of Scope

- Changes to the AI chatbot prompts or agent logic (consumed as-is).
- Real-time streaming ingestion (webhook-triggered processing on upload is future work).
- Web scraping or RSS feed ingestion for government sources (Kai's external sources are a separate initiative).
- Full-text search (the system provides semantic/vector search only).
- UI for knowledge base management (admin manages via CLI or API endpoint; a dashboard UI is future work).
- Re-embedding, payload modification, or deletion of the existing 798 Qdrant points (their vectors and metadata are preserved as-is; structural reorganization into a single collection is in scope per FR-005).
