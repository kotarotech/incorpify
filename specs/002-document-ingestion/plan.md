# Implementation Plan: Document Ingestion Pipeline

**Branch**: `002-document-ingestion` | **Date**: 2026-03-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/002-document-ingestion/spec.md`

## Summary

Build a document ingestion pipeline that parses DOCX/PDF knowledge base documents, chunks text into ~500-token segments, generates 3072-dimension embeddings via OpenAI `text-embedding-3-large`, and upserts them to Qdrant with rich metadata. Simultaneously fix the broken search path by replacing the placeholder `generateQueryVector()` with real embedding calls and correcting the vector dimension config from 1536 to 3072. This unblocks the AI chatbot's semantic search over the 109-document knowledge base.

## Technical Context

**Language/Version**: TypeScript (Bun runtime, native execution)
**Primary Dependencies**: officeparser (DOCX), unpdf (PDF), gpt-tokenizer (token counting), Vercel AI SDK v6 (`embed`/`embedMany`), @qdrant/js-client-rest (already installed), Drizzle ORM (already installed)
**Storage**: PostgreSQL (ingestion tracking) + Qdrant (vector search, existing)
**Testing**: Bun test (sole runner, TDD mandatory per Constitution)
**Target Platform**: Linux server (ECS Fargate) / macOS development
**Project Type**: CLI tool + service extension within Bun monorepo
**Performance Goals**: Single document < 30s, full corpus (109 docs) < 30min, search top-5 relevance > 85%
**Constraints**: Must produce 3072-dimension vectors to match existing Qdrant data; must use Vercel AI SDK for embeddings; existing 798 points preserved as-is
**Scale/Scope**: 109 source documents, ~800-1200 expected chunks, single Qdrant collection

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Status | Notes |
|---|-----------|--------|-------|
| I | Monorepo-First | PASS | Pipeline lives in `packages/agents-service/` (scripts + services). New DB tables in shared schema. Shared types in `@incorpify/shared`. |
| II | Fastify-First API Design | PASS | Admin API endpoints (P2) will use Fastify + Zod validation. CLI (P1) is a standalone script, not an HTTP route. |
| III | Security by Design | PASS | No user-facing input surface on CLI. Admin API behind gateway. Secrets in env vars validated by Zod. No SQL concatenation (Drizzle parameterized). |
| IV | Multi-Tenant Isolation | PASS | Knowledge base documents are global (no tenant scope). Client documents (P3 future) will be scoped by organization_id + company_id via existing company_documents table. |
| V | Test-Driven Development | PASS | TDD for all pipeline components. Unit tests for parser, chunker, metadata derivation. Integration tests for Qdrant upsert/search with disposable collections. |
| VI | Type Safety and Validation | PASS | Existing `QdrantPayload`, `QdrantDocument`, and enum types in shared package. New Drizzle schemas for ingestion tables. Zod validation for CLI args and API inputs. |
| VII | AI Agent Architecture | PASS | Embeddings via Vercel AI SDK v6 `embed()`/`embedMany()`. No deprecated v5 methods. Provider factory already supports OpenAI. |
| VIII | Structured Logging | PASS | Use `log()` from shared utils. Structured context: documentId, chunkCount, jobId, duration. No raw error objects. |
| IX | Frontend Architecture | N/A | No frontend changes in this feature. |
| X | Simplicity and Pragmatism | PASS | CLI-first approach (no API until needed). Custom chunker (~50 lines) over heavy library. Extends existing Qdrant infrastructure. No new abstractions beyond necessary service layer. |

**Gate result**: ALL PASS — proceed to implementation.

## Project Structure

### Documentation (this feature)

```text
specs/002-document-ingestion/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0: technology research and decisions
├── data-model.md        # Phase 1: entity definitions and validation rules
├── quickstart.md        # Phase 1: developer setup guide
├── contracts/
│   ├── cli.md           # Phase 1: CLI command interface
│   └── admin-api.md     # Phase 1: Admin REST API (P2)
├── checklists/
│   └── requirements.md  # Spec quality validation
└── tasks.md             # Phase 2: implementation tasks (NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
backend-agents-service/
├── packages/
│   ├── agents-service/
│   │   ├── scripts/
│   │   │   └── ingest.ts                    # CLI entry point (NEW)
│   │   └── src/
│   │       ├── services/
│   │       │   ├── ingestion/
│   │       │   │   ├── documentParser.ts     # DOCX/PDF text extraction (NEW)
│   │       │   │   ├── documentParser.test.ts
│   │       │   │   ├── textChunker.ts        # Token-aware chunking (NEW)
│   │       │   │   ├── textChunker.test.ts
│   │       │   │   ├── embeddingService.ts   # Vercel AI SDK embed/embedMany (NEW)
│   │       │   │   ├── embeddingService.test.ts
│   │       │   │   ├── metadataDeriver.ts    # Folder path → metadata (NEW)
│   │       │   │   ├── metadataDeriver.test.ts
│   │       │   │   ├── ingestionPipeline.ts  # Orchestrator: parse→chunk→embed→upsert (NEW)
│   │       │   │   ├── ingestionPipeline.test.ts
│   │       │   │   ├── ingestionTracker.ts   # PostgreSQL job/document tracking (NEW)
│   │       │   │   └── ingestionTracker.test.ts
│   │       │   └── chat/
│   │       │       └── RagService.ts         # Existing (no changes needed)
│   │       └── tools/
│   │           └── knowledge/
│   │               └── search-knowledge.ts   # Existing (no changes needed)
│   └── shared/
│       └── src/
│           ├── constants/
│           │   └── database/
│           │       └── config.ts             # MODIFY: VECTOR_DIMENSION 1536→3072
│           ├── infrastructure/
│           │   ├── database/
│           │   │   └── schema/
│           │   │       ├── kb-indexed-documents.ts   # NEW table
│           │   │       ├── kb-ingestion-jobs.ts       # NEW table
│           │   │       └── index.ts                   # MODIFY: add exports
│           │   └── qdrant/
│           │       ├── connection.ts                  # MODIFY: add document_id index
│           │       └── repositories/
│           │           └── base.ts                    # MODIFY: refactor generateQueryVector to accept injected embedFn
│           ├── types/
│           │   └── database/
│           │       ├── schema/
│           │       │   └── index.ts                   # MODIFY: add type exports
│           │       └── repositories/
│           │           ├── kb-indexed-document.ts     # NEW repository types
│           │           └── kb-ingestion-job.ts        # NEW repository types
│           └── validations/
│               └── config/
│                   └── database.ts                    # MODIFY: default 3072
├── migrations/
│   └── drizzle/
│       ├── 0041_kb_indexed_documents.sql              # NEW migration
│       └── 0042_kb_ingestion_jobs.sql                 # NEW migration
└── knowledge-base/                                     # Source documents (gitignored)
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

**Structure Decision**: Pipeline code lives within the existing `packages/agents-service/` package since the agents service owns all AI/LLM interactions (Constitution: Service Boundaries). New ingestion service modules are co-located in `src/services/ingestion/`. Shared infrastructure (DB schema, types, Qdrant config) is in `@incorpify/shared`. No new packages — follows Principle X (Simplicity).

## Complexity Tracking

> No Constitution violations. All changes fit within existing architecture.

| Decision | Rationale | Simpler Alternative Rejected Because |
|----------|-----------|-------------------------------------|
| New PostgreSQL tables for tracking | Need persistent ingestion state across restarts | In-memory tracking lost on restart; Qdrant metadata can't track jobs |
| Custom text chunker | ~50 lines, token-aware, controllable overlap | `langchain` splitters: massive dependency. `llm-splitter`: not widely adopted |
| CLI before API | P1 requires admin-triggered ingestion, not user-facing | API requires Zod route schemas, auth middleware, error handling — deferred to P2 |
