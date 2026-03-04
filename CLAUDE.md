# incorpify Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-03-02

## Active Technologies
- TypeScript (Bun runtime, native execution) + officeparser (DOCX), unpdf (PDF), gpt-tokenizer (token counting), Vercel AI SDK v6 (`embed`/`embedMany`), @qdrant/js-client-rest (already installed), Drizzle ORM (already installed) (002-document-ingestion)
- PostgreSQL (ingestion tracking) + Qdrant (vector search, existing) (002-document-ingestion)

- TypeScript (strict mode) on Bun >= 1.3 + Fastify v5, @fastify/vite, React 19, Vite 6, (001-frontend-migration)

## Project Structure

```text
src/
tests/
```

## Commands

npm test && npm run lint

## Code Style

TypeScript (strict mode) on Bun >= 1.3: Follow standard conventions

## Recent Changes
- 002-document-ingestion: Added TypeScript (Bun runtime, native execution) + officeparser (DOCX), unpdf (PDF), gpt-tokenizer (token counting), Vercel AI SDK v6 (`embed`/`embedMany`), @qdrant/js-client-rest (already installed), Drizzle ORM (already installed)

- 001-frontend-migration: Added TypeScript (strict mode) on Bun >= 1.3 + Fastify v5, @fastify/vite, React 19, Vite 6,

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
