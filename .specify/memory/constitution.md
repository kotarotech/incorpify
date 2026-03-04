<!--
  Sync Impact Report
  ==================
  Version change: 0.0.0 (template) → 1.0.0 (initial ratification)

  Modified principles: N/A (first version)

  Added sections:
    - 10 Core Principles (I through X)
    - Architecture Constraints section
    - Development Workflow section
    - Governance section

  Removed sections: None

  Templates requiring updates:
    - .specify/templates/plan-template.md — ✅ No changes needed
      (Constitution Check section already references this file dynamically)
    - .specify/templates/spec-template.md — ✅ No changes needed
      (spec template is technology-agnostic by design)
    - .specify/templates/tasks-template.md — ✅ No changes needed
      (task phases and parallel markers align with principles)
    - .specify/templates/commands/*.md — ✅ No files found (no updates needed)

  Follow-up TODOs: None
-->

# Incorpify Constitution

## Core Principles

### I. Monorepo-First

All code lives in a single Bun-workspace monorepo. Each concern
is a dedicated package under `packages/`.

- Every service (agents, checkout, crm, notification, frontend) MUST
  be its own package with an independent `package.json`, `tsconfig.json`,
  and entrypoint.
- Shared logic (types, repositories, infrastructure, utilities) MUST
  reside in `@incorpify/shared` and MUST use relative imports internally.
- Services MUST import shared code via `@incorpify/shared/...` and local
  code via the `@/` alias. Services MUST NOT import from other services.
- Database migrations MUST live at monorepo root (`migrations/drizzle/`),
  never inside individual packages.
- A single PostgreSQL database is shared across all services. Qdrant is
  used exclusively for vector/semantic search.

### II. Fastify-First API Design

All HTTP services are built on Fastify v5 with strict schema validation.

- Every route MUST define Zod v4 schemas for params, querystring, body,
  and response. No route may skip validation.
- Response envelope MUST follow: `{ success: boolean, data: T | null,
  error: { code: string, message: string } | null }`.
- Handlers MUST be thin: extract request data, call service layer,
  return response. Business logic MUST NOT live in handlers.
- Services run on dedicated ports: agents 3000, checkout 3001, crm 3002,
  notification 3003, frontend 3004.
- Pagination MUST use `?limit=N&offset=N` (default 20, max 100) on all
  list endpoints.
- URLs MUST use kebab-case, plural nouns, and resource-named params
  (e.g., `/api/v1/business-areas/:businessAreaId`).

### III. Security by Design

The platform handles business incorporation data and financial
transactions. Security is non-negotiable at every layer.

- Services MUST deploy in a private VPC, never exposed directly to the
  public internet. An API Gateway handles TLS termination and JWT
  validation.
- Services MUST NOT validate JWTs themselves. Bearer tokens and custom
  headers (`X-User-Id`, `X-Space-Id`, `X-Organization-Id`,
  `X-Request-Id`) are trusted when originating from the gateway.
- All database queries MUST be parameterized (bun:sql tagged templates).
  String concatenation in SQL is forbidden.
- All inputs MUST be validated at the boundary using allowlist schemas.
  Reject on first error; fail closed.
- Secrets MUST be stored in a dedicated secret manager or secure
  environment system, never in source control.
- Bearer tokens, API keys, passwords, and PII MUST NOT appear in logs,
  analytics, or crash payloads.
- Rate limiting and quota enforcement MUST be applied; return HTTP 429
  consistently.

### IV. Multi-Tenant Isolation

Incorpify is a multi-tenant B2B SaaS. Every data operation MUST be
scoped to the user's active space (organization) and company.

- `X-Space-Id` is the primary tenant discriminator. `X-Organization-Id`
  is supported as a legacy alias.
- Every repository query that touches tenant-scoped data MUST include
  the space filter. Omitting it is a security defect.
- Role-based access control enforces five roles (Super Administrator,
  Owner, Admin, Manager, Viewer) with eleven or more granular
  permissions. Handlers MUST verify permissions before executing
  mutations.
- Auth0 is the identity provider. Session management uses OAuth2 with
  token refresh. The frontend stores sessions in encrypted cookies.

### V. Test-Driven Development (NON-NEGOTIABLE)

All new features and bug fixes MUST follow TDD: write a failing test,
implement the minimum code to pass, then refactor.

- The Red-Green-Refactor cycle is mandatory. No implementation code
  may be written before a corresponding test exists and fails.
- Bun test (`bun test`) is the sole test runner. Do NOT mix frameworks.
- Tests MUST be co-located with source: `userService.ts` next to
  `userService.test.ts`.
- Test pyramid target: 70% unit, 20% integration, 10% E2E.
- Coverage target: 80% or higher for lines and branches on changed
  modules. CI MUST fail on coverage regression.
- Unit tests MUST NOT call real network, filesystem, or database.
  Use in-memory fakes, mocks, and fixtures.
- Integration tests MUST use transactional PostgreSQL with rollback
  and disposable Qdrant collections.
- HTTP tests MUST use `app.inject()` (Fastify), not real sockets.
- Tests MUST be deterministic, isolated, and share no mutable state.
  Each test has a single reason to fail.

### VI. Type Safety and Validation

TypeScript strict mode is mandatory. Types are inferred from schemas,
never duplicated.

- Zod v4 top-level methods MUST be used: `z.uuid()`, `z.email()`,
  `z.url()`, `z.iso.datetime()`. Deprecated v3 chained methods
  (`z.string().uuid()`) are forbidden.
- TypeScript types MUST be inferred from Zod schemas via `z.infer<>`.
  Do NOT define parallel type definitions.
- Shared types live in `packages/shared/src/types/`. Service-specific
  types live in `packages/<service>/src/types/`. Types MUST NOT be
  duplicated across locations.
- Use `import type` for type-only imports.
- Database schemas define table structure only. Validation logic is
  separate and lives in validation schema files.

### VII. AI Agent Architecture

The AI chatbot is a core differentiator. Agent design follows
strict patterns for reliability and safety.

- Vercel AI SDK v6 is the sole AI integration layer. Use `streamText`
  and `generateText` with `Output.object()` for structured output.
  Deprecated v5 methods (`generateObject`, `streamObject`) are
  forbidden.
- Each agent has exactly one immutable system prompt. User content
  MUST NOT be concatenated into system prompts without explicit
  boundaries.
- System prompts MUST include anti-hallucination directives: "If
  missing data, ask or decline."
- Tools MUST declare parameters and outputs with Zod schemas. Tool
  calls MUST time out and retry with exponential backoff (max 2
  retries).
- `maxSteps` MUST be set to prevent infinite agent loops.
- All agent operations MUST respect the user's space context from
  request headers.
- Sensitive fields MUST be masked in tool logs. Bearer tokens MUST
  NOT appear in AI context or logs.

### VIII. Structured Logging

Observability requires consistent, structured, and safe logging.

- Application code MUST use `log()` from `@incorpify/shared/utils/
  requests`. The `logger` export is reserved for Fastify instance
  configuration only.
- Log calls MUST use structured data objects, never string
  interpolation: `log('Processing order', { orderId, userId })`.
- Errors MUST be extracted as messages:
  `error instanceof Error ? error.message : String(error)`.
  Do NOT log raw error objects.
- Every log entry MUST include relevant context: userId, requestId,
  operationName. Duration MUST be logged for timed operations.
- Log messages MUST use present tense, active voice, and be specific:
  "Database connection failed" not "Something went wrong."

### IX. Frontend Architecture

The frontend is a package within the monorepo, preserving the
existing design system while consuming Fastify backend services.

- The frontend MUST use React with Radix UI and Tailwind CSS,
  preserving the existing component design language.
- Components MUST follow Atomic Design: atoms (primitives), molecules
  (composites), pages (route-level containers).
- Data fetching MUST use TanStack React Query with a centralized
  query key factory and access layer (getters for reads, mutations
  for writes).
- Forms MUST use React Hook Form with Zod schema resolvers for
  validation.
- Internationalization MUST support at minimum English and Arabic
  locales with full RTL layout support.
- Auth0 OAuth2 integration MUST use encrypted cookie sessions with
  automatic token refresh.
- The frontend MUST pass `Authorization`, `X-Space-Id`, and
  `X-Organization-Id` headers on every authenticated API call.
- SEO-critical public pages (landing, services, pricing, blog) MUST
  support server-side rendering or static generation.
- The frontend MUST gracefully degrade when backend services are
  unavailable, showing user-friendly error states with retry options.

### X. Simplicity and Pragmatism

Complexity is a cost. Every abstraction must earn its place.

- Start with the simplest solution that meets requirements. Do NOT
  design for hypothetical future needs (YAGNI).
- Three similar lines of code are preferable to a premature
  abstraction.
- Services MUST NOT call each other via HTTP. Use shared repositories
  backed by the common PostgreSQL database instead.
- Configuration MUST be validated with Zod schemas at startup. Never
  read `process.env` directly.
- Every deviation from these principles MUST be documented in a
  Complexity Tracking table (see plan template) with justification
  and a rejected simpler alternative.

## Architecture Constraints

### Technology Stack

- **Runtime**: Bun (native TypeScript execution, no transpilation step)
- **Backend Framework**: Fastify v5 with Zod v4 type provider
- **Frontend Framework**: React 19 with Radix UI + Tailwind CSS
- **AI Integration**: Vercel AI SDK v6 (multi-provider: Azure OpenAI,
  OpenAI, Anthropic, Google Gemini)
- **Database**: PostgreSQL (structured data) + Qdrant (vector search)
- **ORM**: Drizzle ORM with bun:sql driver
- **Authentication**: Auth0 OAuth2
- **Payments**: Stripe
- **Package Manager**: Bun (workspaces for monorepo)
- **Test Runner**: Bun test (sole runner, no mixing)

### Service Boundaries

| Service        | Port | Responsibility                          |
|----------------|------|-----------------------------------------|
| agents-service | 3000 | AI chat, conversations, tool execution  |
| checkout       | 3001 | Stripe payment flows                    |
| crm-service    | 3002 | RESTful CRUD for all business entities  |
| notification   | 3003 | Email, SMS, push notifications          |
| frontend       | 3004 | Web application serving                 |

- Services share a single PostgreSQL database via shared repositories.
- Services MUST NOT call each other over HTTP.
- The CRM service is a pure CRUD API: no AI, no streaming, no Qdrant.
- The agents service handles all AI/LLM interactions exclusively.

### Naming Conventions

| Context    | Convention  | Example                           |
|------------|-------------|-----------------------------------|
| Database   | snake_case  | `company_workflows`, `created_at` |
| API URLs   | kebab-case  | `/api/v1/business-areas`          |
| Code       | camelCase   | `getCompanyById`, `spaceId`       |
| Types      | PascalCase  | `CompanyWorkflow`, `UserRole`     |
| Constants  | UPPER_SNAKE | `DEFAULT_PAGE_SIZE`, `MAX_RETRIES`|
| Files      | camelCase   | `companyService.ts`               |
| Test files | camelCase   | `companyService.test.ts`          |

## Development Workflow

### Code Quality Gates

Every pull request MUST pass all gates before merge:

1. **Typecheck**: `tsc --noEmit` passes with zero errors.
2. **Lint**: ESLint passes with zero warnings treated as errors.
3. **Format**: Prettier formatting is consistent (enforced by CI).
4. **Test**: All tests pass via `bun test`.
5. **Coverage**: No coverage regression below 80% threshold.
6. **Constitution Compliance**: Reviewer verifies adherence to these
   principles using the plan template's Constitution Check section.

### Commit and Branch Strategy

- Feature branches follow the pattern `NNN-short-name` (e.g.,
  `001-frontend-migration`).
- Commits MUST be atomic and descriptive.
- Specifications live in `specs/NNN-short-name/` with plan, spec,
  tasks, and supporting artifacts.

### Review Expectations

- Every PR MUST be reviewed by at least one team member.
- Reviewers MUST verify: test coverage, type safety, security
  implications, naming conventions, and constitution compliance.
- Complexity additions MUST be justified in the PR description
  with reference to the Complexity Tracking table.

## Governance

This constitution is the authoritative source of engineering
standards for the Incorpify project. It supersedes all other
practice documents, README guidance, and ad-hoc conventions.

- **Amendments** require: (1) a written proposal documenting the
  change and rationale, (2) review and approval by at least one
  team lead, and (3) a migration plan if existing code must change.
- **Versioning** follows semantic versioning: MAJOR for
  backward-incompatible principle changes or removals, MINOR for
  new principles or material expansions, PATCH for clarifications
  and wording fixes.
- **Compliance reviews** MUST occur at every PR via the Constitution
  Check section in plan documents. Quarterly audits of codebase
  adherence are recommended.
- **Exceptions** to any principle MUST be documented in the
  Complexity Tracking table with justification and a rejected
  simpler alternative.

Refer to the Windsurf rules in `backend-agents-service/.windsurf/
rules/` for detailed implementation-level guidance that supports
these principles.

**Version**: 1.0.0 | **Ratified**: 2026-03-02 | **Last Amended**: 2026-03-02
