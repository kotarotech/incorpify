# Tasks: Document Ingestion Pipeline

**Input**: Design documents from `/specs/002-document-ingestion/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/cli.md, contracts/admin-api.md, quickstart.md

**Tests**: Included — Constitution Principle V mandates TDD (NON-NEGOTIABLE).

**Organization**: Tasks grouped by user story. US1 and US2 are both P1 but structurally sequential (US1 fixes search, US2 builds ingestion pipeline that populates what US1 searches). US3 (P2) adds admin management. US4 (P3) deferred — client OCR is out of current scope.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

All paths relative to `backend-agents-service/`:
- **Shared package**: `packages/shared/src/`
- **Agents service**: `packages/agents-service/src/`
- **Scripts**: `packages/agents-service/scripts/`
- **Migrations**: `migrations/drizzle/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Install dependencies, update config, create database schemas

- [ ] T001 Install new dependencies: `officeparser`, `unpdf`, `gpt-tokenizer` in `packages/agents-service/package.json`
- [ ] T002 Update VECTOR_DIMENSION default from 1536 to 3072 in `packages/shared/src/constants/database/config.ts` (line 19) and `packages/shared/src/validations/config/database.ts` (default value)
- [ ] T003 [P] Add `KB_SOURCE_DIR` and `OPENAI_API_KEY` to environment config validation schema in `packages/shared/src/validations/config/database.ts` and export through `packages/shared/src/config/index.ts`
- [ ] T004 [P] Create Drizzle schema for `knowledge_documents` table in `packages/shared/src/infrastructure/database/schema/knowledge-documents.ts` per data-model.md (26 fields incl. file storage fields and knowledge_document_job_id FK, indexes on document_id unique, knowledge_document_job_id, status, category, country_code, jurisdiction_code)
- [ ] T005 [P] Create Drizzle schema for `knowledge_document_jobs` table in `packages/shared/src/infrastructure/database/schema/knowledge-document-jobs.ts` per data-model.md (11 fields incl. skipped_documents, no error_log, indexes on status, job_type, created_at)
- [ ] T006 Export new schemas from `packages/shared/src/infrastructure/database/schema/index.ts` and add type re-exports to `packages/shared/src/types/database/schema/index.ts`
- [ ] T007 Generate and apply Drizzle migrations for `knowledge_document_jobs` (0041, created first since it's referenced by FK) and `knowledge_documents` (0042, has FK to knowledge_document_jobs) in `migrations/drizzle/`
- [ ] T008 Add `document_id` keyword payload index to the `PAYLOAD_INDEXES` array in `packages/shared/src/types/database/qdrant/infrastructure.ts` for efficient chunk deletion during re-ingestion
- [ ] T009 Update `restore-qdrant.ts` script in `scripts/remote-restore-qdrant.ts` to merge kb_1 and kb_3 backup collections into the single `knowledge_base` collection (load both point sets, deduplicate by ID, upsert to unified collection)

**Checkpoint**: Infrastructure ready — VECTOR_DIMENSION=3072, new DB tables exist, Qdrant has document_id index, dependencies installed.

---

## Phase 2: Foundational (Core Pipeline Services)

**Purpose**: Build the reusable pipeline components that both US1 and US2 depend on. TDD: write tests first, ensure they fail, then implement.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Tests (write FIRST, must FAIL)

- [ ] T010 [P] Write unit tests for document parser in `packages/agents-service/src/services/ingestion/documentParser.test.ts` — test DOCX extraction (mock officeparser), PDF extraction (mock unpdf), error handling for corrupt files, unsupported formats, empty files
- [ ] T011 [P] Write unit tests for text chunker in `packages/agents-service/src/services/ingestion/textChunker.test.ts` — test ~500 token chunks with ~50 token overlap, paragraph boundary splitting, minimum chunk size (50 tokens), maximum chunk size (600 tokens), deterministic chunk IDs, empty input handling
- [ ] T012 [P] Write unit tests for metadata deriver in `packages/agents-service/src/services/ingestion/metadataDeriver.test.ts` — test folder path parsing for UAE→AE/UAE_MAINLAND, KSA→SA/KSA_MAINLAND, category derivation from folder names (all 8 mappings from research.md R6), version extraction from filenames (e.g., `1.3_` → version 3), document_id generation from path, document_type derivation from source_type (LAW→REGULATION, INTERNAL→GUIDE, FAQ→FAQ)
- [ ] T013 [P] Write unit tests for embedding service in `packages/agents-service/src/services/ingestion/embeddingService.test.ts` — test single embedding via `embed()`, batch embedding via `embedMany()`, 3072-dimension output validation, error handling for API failures, retry with backoff, empty input rejection

### Implementation

- [ ] T014 [P] Implement document parser in `packages/agents-service/src/services/ingestion/documentParser.ts` — `parseDocument(filePath: string): Promise<{ text: string, metadata: { pageCount?: number, charCount: number } }>`. Use officeparser for .docx, unpdf for .pdf. Throw typed errors for unsupported formats and corrupt files. (FR-001)
- [ ] T015 [P] Implement text chunker in `packages/agents-service/src/services/ingestion/textChunker.ts` — `chunkText(text: string, options: { targetTokens: 500, overlap: 50, maxTokens: 600, minTokens: 50 }): Chunk[]`. Recursive split on `\n\n` → `.` → token limit. Use gpt-tokenizer for counting. Return `{ content, chunkIndex, tokenCount }[]`. (FR-002)
- [ ] T016 [P] Implement metadata deriver in `packages/agents-service/src/services/ingestion/metadataDeriver.ts` — `deriveMetadata(filePath: string): DocumentMetadata`. Parse folder hierarchy for country_code, jurisdiction_code, category. Extract version from filename. Generate deterministic document_id. Map folder names to RagDocumentCategory enum. Derive document_type from source_type and category context (e.g., LAW→'REGULATION', INTERNAL→'GUIDE', FAQ→'FAQ', default→'GUIDE'). (FR-013)
- [ ] T017 Implement embedding service in `packages/agents-service/src/services/ingestion/embeddingService.ts` — `embedText(text: string): Promise<number[]>` and `embedTexts(texts: string[]): Promise<number[][]>`. Use Vercel AI SDK v6 `embed()`/`embedMany()` with OpenAI `text-embedding-3-large`. Include retry with exponential backoff (max 2 retries per Constitution VII). Validate 3072-dimension output. (FR-003)

**Checkpoint**: All four pipeline services pass their unit tests. Ready for orchestration.

---

## Phase 3: User Story 1 — AI Chatbot Returns Accurate Answers (Priority: P1) 🎯 MVP

**Goal**: Fix the broken search path so the AI chatbot can retrieve relevant knowledge base content using real embeddings instead of placeholder hashes.

**Independent Test**: Ask the chatbot a domain-specific question (e.g., "What documents do I need for UAE company setup?") and verify the response references real KB content, not hallucinated information.

### Tests (write FIRST, must FAIL)

- [ ] T018 [P] [US1] Write unit test for generateQueryVector replacement in `packages/shared/src/infrastructure/qdrant/repositories/base.test.ts` — test that `generateQueryVector(query, embedFn)` calls the injected `embedFn` and returns a 3072-dimension vector, not a hash-based vector. Test that existing `existingVector` passthrough still works. Test that missing `embedFn` throws a clear error.
- [ ] T019 [P] [US1] Write integration test for end-to-end search in `packages/agents-service/src/services/ingestion/searchIntegration.test.ts` — test that searching with real embeddings against a disposable Qdrant collection with known test documents returns relevant results ranked by relevance score. (FR-006)
- [ ] T019a [P] [US1] Write unit test for source authority re-ranking in `packages/shared/src/infrastructure/qdrant/repositories/base.test.ts` — test that when two results have similar scores but different source_type (LAW vs INTERNAL), the LAW result is ranked higher after re-ranking. Test edge cases: same source_type, large score gap (no re-ranking needed), priority_boost override. (FR-005a)

### Implementation

- [ ] T020 [US1] Refactor `generateQueryVector()` in `packages/shared/src/infrastructure/qdrant/repositories/base.ts` (lines 104-120) to accept an injected `embedFn: (text: string) => Promise<number[]>` parameter instead of importing embeddingService directly. Remove the hash-based placeholder. Make the method async. Update `search()` to accept `embedFn` as a parameter. This keeps AI SDK dependencies out of `@incorpify/shared` (Constitution I). (FR-006)
- [ ] T021 [US1] Update all repository search callers to pass `embedFn` — in RagService.ts (`packages/agents-service/src/services/chat/RagService.ts`), search-knowledge.ts (`packages/agents-service/src/tools/knowledge/search-knowledge.ts`), and any other callers: import `embeddingService.embedText` from `packages/agents-service/src/services/ingestion/embeddingService.ts` and pass it as the `embedFn` argument to repository `.search()` calls. Verify `countrySemanticRepository`, `jurisdictionSemanticRepository`, `businessAreaSemanticRepository` all work with the updated signature.
- [ ] T022 [US1] Implement source authority weighting in search results — add post-search re-ranking logic in `packages/shared/src/infrastructure/qdrant/repositories/base.ts` so results with `source_type: LAW` rank higher than `INTERNAL` when vector similarity scores are within 0.05 of each other. Use `confidence_score` and `priority_boost` payload fields as tiebreakers. (FR-005a)
- [ ] T023 [US1] Verify existing RagService.ts works with updated search — run the RagService flow (`retrieveDocuments` → `answer`) against a Qdrant instance with the existing 798 points and confirm answers reference real document content. Verify embedFn injection works end-to-end.

**Checkpoint**: Chatbot search returns real results from the knowledge base. SC-001 (90% relevant answers) and SC-004 (85% top-5 accuracy) can be validated.

---

## Phase 4: User Story 2 — Administrator Ingests New Documents (Priority: P1)

**Goal**: Build the full ingestion pipeline (parse → chunk → embed → upsert) and CLI to process knowledge base documents into Qdrant.

**Independent Test**: Add a new .docx document to the knowledge base directory, run `bun run ingest -- file <path>`, then search for content from that document via the chatbot.

### Tests (write FIRST, must FAIL)

- [ ] T024 [P] [US2] Write unit tests for ingestion tracker in `packages/agents-service/src/services/ingestion/ingestionTracker.test.ts` — test job creation (SINGLE/BATCH/FULL_CORPUS), status transitions (PENDING→PROCESSING→COMPLETED), document registration and status updates, progress tracking (processed/failed/skipped counts), embedding cost recording via conversation_costs. Mock PostgreSQL queries.
- [ ] T025 [P] [US2] Write unit tests for ingestion pipeline orchestrator in `packages/agents-service/src/services/ingestion/ingestionPipeline.test.ts` — test single document flow (parse→chunk→embed→upsert→track), batch processing with mixed success/failure (FR-015), version-aware re-ingestion replacing old chunks (FR-008), incremental updates without re-processing existing (FR-007), file hash change detection
- [ ] T026 [P] [US2] Write integration test for full pipeline in `packages/agents-service/src/services/ingestion/ingestionPipeline.integration.test.ts` — test ingesting a real .docx test fixture, verifying chunks appear in disposable Qdrant collection with correct metadata, verifying knowledge_documents row is created in test database

### Implementation

- [ ] T027 [US2] Create repository types for KnowledgeDocument in `packages/shared/src/types/database/repositories/knowledge-document.ts` — interfaces for `IKnowledgeDocumentRepository` with methods: create, getByDocumentId, update, delete, findByStatus, findByCategory, findByJobId, countByStatus, list with pagination
- [ ] T028 [US2] Create repository types for KnowledgeDocumentJob in `packages/shared/src/types/database/repositories/knowledge-document-job.ts` — interfaces for `IKnowledgeDocumentJobRepository` with methods: create, getById, updateStatus, updateProgress (processed/failed/skipped counts), getLatest, list with pagination
- [ ] T029 [US2] Implement ingestion tracker in `packages/agents-service/src/services/ingestion/ingestionTracker.ts` — manages job lifecycle and document status in PostgreSQL. Methods: `createJob()`, `startJob()`, `completeJob()`, `failJob()`, `registerDocument()`, `updateDocumentStatus()`, `skipDocument()`, `recordEmbeddingCost()`, `getProgress()`. Track skipped documents (unchanged hash). Record embedding costs to conversation_costs table with call_type='EMBEDDING' — add 'EMBEDDING' as a valid call_type value in the conversation_costs validation/schema if not already present. Uses Drizzle ORM with bun:sql tagged templates. (FR-010)
- [ ] T030 [US2] Implement ingestion pipeline orchestrator in `packages/agents-service/src/services/ingestion/ingestionPipeline.ts` — `ingestDocument(filePath: string, jobId: string): Promise<IngestResult>` and `ingestBatch(filePaths: string[], jobType: string): Promise<BatchResult>`. Orchestrates: parse → chunk → derive metadata → embed → upsert to Qdrant → track in PostgreSQL. Handles: file hash comparison for change detection (FR-007), deletion of old chunks before re-upsert (FR-008), graceful failure per document (FR-015). Uses structured logging with documentId, chunkCount, duration context.
- [ ] T031 [US2] Implement CLI entry point in `packages/agents-service/scripts/ingest.ts` per contracts/cli.md — parse CLI args for subcommands: `file <path>`, `dir <path>`, `all`. Register `status` and `remove` as recognized commands but output "Not yet implemented — see Phase 5 (US3)" stubs. Wire ingestion subcommands to ingestionPipeline and ingestionTracker. Add `--verbose`, `--dry-run`, `--force`, `--clean`, `--source-dir` flags. Add `"ingest"` script to `packages/agents-service/package.json`. (FR-009)
- [ ] T032 [US2] Implement file discovery for batch/directory ingestion in `packages/agents-service/scripts/ingest.ts` — recursive directory scan for .docx and .pdf files, sorted by path. Used by `dir` and `all` commands. Filter out hidden files (dotfiles). Skip `Copy of` prefixed files by default but include them with `--include-copies` flag (duplicate handling strategy is deferred — see spec.md Clarifications).

**Checkpoint**: Full ingestion pipeline works end-to-end. `bun run ingest -- all` processes all 109 documents. SC-002 (searchable in 5 min), SC-003 (full corpus < 30 min), SC-005 (single doc < 30s), SC-008 (109 docs without errors) can be validated.

---

## Phase 5: User Story 3 — Administrator Manages Knowledge Base Content (Priority: P2)

**Goal**: Add status/listing/removal capabilities via CLI and prepare the admin API endpoints.

**Independent Test**: Run `bun run ingest -- status` to see indexed documents. Run `bun run ingest -- remove <id>` to remove a document, then verify it's no longer returned in search.

### Tests (write FIRST, must FAIL)

- [ ] T033 [P] [US3] Write unit tests for CLI status command in `packages/agents-service/scripts/ingest.test.ts` — test `status` output formatting (document list, jurisdiction breakdown, category breakdown, recent jobs), test `remove` command (deletes chunks from Qdrant + updates PostgreSQL status to REMOVED)
- [ ] T034 [P] [US3] Write contract tests for admin API endpoints in `packages/agents-service/src/routes/admin/knowledge-base.test.ts` — test GET /documents (pagination, filters), GET /stats, DELETE /documents/:documentId, POST /ingest, GET /jobs/:jobId per contracts/admin-api.md. Use app.inject() per Constitution V.

### Implementation

- [ ] T035 [US3] Implement `status` CLI command in `packages/agents-service/scripts/ingest.ts` — query knowledge_documents and knowledge_document_jobs tables, format output per contracts/cli.md status section. Include document counts by jurisdiction and category, recent jobs, last indexed timestamp. (FR-011)
- [ ] T036 [US3] Implement `remove` CLI command in `packages/agents-service/scripts/ingest.ts` — look up document by document_id, delete all associated chunks from Qdrant via `deleteSemanticDocuments()`, update knowledge_documents status to REMOVED. (FR-012)
- [ ] T037 [US3] Create Fastify route plugin for admin knowledge base endpoints in `packages/agents-service/src/routes/admin/knowledge-base.ts` — register routes: GET /documents, GET /stats, DELETE /documents/:documentId, POST /ingest, GET /jobs/:jobId. Use Zod schemas for all params, querystring, body, and response per Constitution II. Use standard response envelope `{ success, data, error }`. Handlers MUST verify admin/super-admin role from gateway headers (`X-User-Id` + role check) before executing mutations per Constitution IV. Rate limiting is delegated to API Gateway per Constitution III — document this in a code comment. (FR-011, FR-012)
- [ ] T038 [US3] Implement Zod validation schemas for admin API in `packages/agents-service/src/validations/admin/knowledge-base.ts` — request/response schemas for all 5 endpoints per contracts/admin-api.md. Use z.uuid(), z.enum() for category/status filters, pagination with limit/offset defaults.
- [ ] T039 [US3] Register admin routes in the agents service Fastify app — add the knowledge-base route plugin to the main app registration in `packages/agents-service/src/app.ts` or equivalent entry point

**Checkpoint**: Administrators can list, inspect, and remove indexed documents. Admin API endpoints respond correctly. SC-006 (95% correct metadata derivation) can be validated by inspecting status output.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Error handling edge cases, performance, documentation

- [ ] T040 [P] Add retry with exponential backoff for Qdrant operations (upsert, delete) in `packages/agents-service/src/services/ingestion/ingestionPipeline.ts` — handle temporary Qdrant unavailability per edge case in spec.md
- [ ] T041 [P] Add structured logging throughout pipeline — ensure all log calls use `log()` from `@incorpify/shared/utils/requests` with context objects `{ documentId, jobId, chunkCount, duration }` per Constitution VIII
- [ ] T042 [P] Add large document handling — if a document exceeds 100 pages or 50,000 tokens, process chunks in batches of 100 to avoid embedding API limits. Log warning for unusually large documents.
- [ ] T043 Handle mixed-language documents (English + Arabic) — add bilingual test fixtures to `packages/agents-service/src/services/ingestion/textChunker.test.ts` verifying gpt-tokenizer handles Arabic text correctly (token counts, chunk boundaries). Test with real bilingual samples from the knowledge base.
- [ ] T044 Run full corpus ingestion validation — execute `bun run ingest -- all` against all 109 knowledge base documents, verify SC-008 (no errors), capture timing for SC-003 (< 30 min), spot-check metadata derivation accuracy for SC-006 (95%)
- [ ] T045 Run quickstart.md end-to-end validation — follow every step in quickstart.md from scratch (Docker up, env config, migrations, restore, ingest, verify search) to confirm the guide is accurate and complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 completion (T001-T009)
- **Phase 3 (US1)**: Depends on Phase 2 (needs embeddingService from T017)
- **Phase 4 (US2)**: Depends on Phase 2 (needs all four pipeline services) + Phase 3 T020 (needs real embeddings for ingested docs to be searchable)
- **Phase 5 (US3)**: Depends on Phase 4 (needs ingestion tracker and pipeline to exist)
- **Phase 6 (Polish)**: Depends on Phase 4 completion minimum

### User Story Dependencies

- **US1 (Fix Search)**: Depends on Phase 2 only — can start as soon as embeddingService exists
- **US2 (Ingestion Pipeline)**: Depends on Phase 2 + US1 T020 (generateQueryVector replacement) — ingested documents need real search to be useful
- **US3 (Admin Management)**: Depends on US2 — management features operate on data created by ingestion pipeline

### Within Each User Story

1. Tests written FIRST → must FAIL (Red)
2. Implementation → tests pass (Green)
3. Refactor if needed
4. Commit at each checkpoint

### Parallel Opportunities

**Phase 1**: T003, T004, T005, T008 can run in parallel (different files)
**Phase 2 Tests**: T010, T011, T012, T013 can all run in parallel (different test files)
**Phase 2 Impl**: T014, T015, T016 can run in parallel (different service files); T017 depends only on T013's test
**Phase 4 Tests**: T024, T025, T026 can all run in parallel
**Phase 4 Impl**: T027, T028 can run in parallel (different type files)
**Phase 5 Tests**: T033, T034 can run in parallel
**Phase 6**: T040, T041, T042 can all run in parallel

---

## Parallel Example: Phase 2 (Foundational)

```bash
# Launch all tests in parallel (must all FAIL first):
Task: T010 "Write unit tests for documentParser in documentParser.test.ts"
Task: T011 "Write unit tests for textChunker in textChunker.test.ts"
Task: T012 "Write unit tests for metadataDeriver in metadataDeriver.test.ts"
Task: T013 "Write unit tests for embeddingService in embeddingService.test.ts"

# Then launch parallel implementations:
Task: T014 "Implement documentParser in documentParser.ts"
Task: T015 "Implement textChunker in textChunker.ts"
Task: T016 "Implement metadataDeriver in metadataDeriver.ts"
# T017 (embeddingService) after — it needs async patterns established
```

## Parallel Example: Phase 4 (User Story 2)

```bash
# Launch all tests in parallel:
Task: T024 "Write unit tests for ingestionTracker"
Task: T025 "Write unit tests for ingestionPipeline"
Task: T026 "Write integration test for full pipeline"

# Then launch type definitions in parallel:
Task: T027 "Create repository types for KnowledgeDocument"
Task: T028 "Create repository types for KnowledgeDocumentJob"

# Then sequential implementation:
Task: T029 "Implement ingestionTracker" (needs T027, T028)
Task: T030 "Implement ingestionPipeline" (needs T029)
Task: T031 "Implement CLI entry point" (needs T030)
Task: T032 "Implement file discovery" (needs T031)
```

---

## Implementation Strategy

### MVP First (User Stories 1 + 2)

1. Complete Phase 1: Setup (~30 min)
2. Complete Phase 2: Foundational pipeline services (~2 hours)
3. Complete Phase 3: US1 — Fix search (~1 hour)
4. **VALIDATE**: Chatbot returns real answers from existing 798 Qdrant points
5. Complete Phase 4: US2 — Ingestion pipeline + CLI (~3 hours)
6. **VALIDATE**: Full corpus ingested, chatbot answers from new + existing content
7. Deploy/demo MVP

### Incremental Delivery

1. Setup + Foundational → Core pipeline services ready
2. US1 → Search works → **Immediate value** (chatbot functional)
3. US2 → Ingestion works → **Content updatable** (KB maintainable)
4. US3 → Admin management → **Operational visibility** (monitoring + cleanup)
5. Each increment is independently testable and deployable

---

## Summary

| Phase | Tasks | Parallel Opportunities |
|-------|-------|----------------------|
| Phase 1: Setup | T001–T009 (9 tasks) | T003, T004, T005, T008 |
| Phase 2: Foundational | T010–T017 (8 tasks) | T010-T013, T014-T016 |
| Phase 3: US1 (Fix Search) | T018–T023 + T019a (7 tasks) | T018, T019, T019a |
| Phase 4: US2 (Ingestion) | T024–T032 (9 tasks) | T024-T026, T027-T028 |
| Phase 5: US3 (Management) | T033–T039 (7 tasks) | T033-T034 |
| Phase 6: Polish | T040–T045 (6 tasks) | T040-T042 |
| **Total** | **46 tasks** | |

**Tasks per user story**: US1: 7, US2: 9, US3: 7
**MVP scope**: Phase 1–4 (33 tasks) — delivers working search + ingestion pipeline
**User Story 4 (Client OCR)**: Deferred to future iteration — not included in task list per spec P3 scope
