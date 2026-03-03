# Implementation Plan: Frontend Migration to Monorepo

**Branch**: `001-frontend-migration` | **Date**: 2026-03-02 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-frontend-migration/spec.md`

## Summary

Migrate the existing Next.js 15/React 19 frontend (`azure-backup-frontend`)
into a new `packages/frontend` package within the Bun monorepo
(`backend-agents-service`). The new frontend is a Fastify v5 service using
`@fastify/vite` for server-side rendering of public pages and client-side
SPA routing for authenticated dashboard routes. It consumes four backend
services (CRM, agents, checkout, notification) via typed fetch-based API
clients, authenticates via Auth0 OAuth2 with encrypted cookie sessions,
and preserves the existing Radix UI + Tailwind CSS component library with
Atomic Design. TanStack Router provides type-safe routing; TanStack React
Query v5 manages server state; Paraglide.js handles i18n with RTL support.

## Technical Context

**Language/Version**: TypeScript (strict mode) on Bun >= 1.3
**Primary Dependencies**: Fastify v5, @fastify/vite, React 19, Vite 6,
  TanStack Router, TanStack React Query v5, Radix UI, Tailwind CSS 4,
  React Hook Form, Zod v4, @fastify/secure-session, @fastify/oauth2,
  Paraglide.js, @fastify/static
**Storage**: PostgreSQL 18 (via shared Drizzle ORM repositories, no
  direct DB access from frontend), Qdrant (accessed only via agents service)
**Testing**: bun test (co-located `.test.ts` files, 80%+ coverage target,
  70/20/10 unit/integration/E2E pyramid)
**Target Platform**: Web browser (desktop + mobile responsive), served by
  Fastify on Bun runtime, deployed via Docker (oven/bun image) on ECS Fargate
**Project Type**: Web service (SSR + SPA hybrid)
**Performance Goals**: Dashboard load < 2s, SSR pages < 500ms TTFB,
  chatbot first token < 1s, org/company switch < 2s, 500 concurrent users
**Constraints**: Auth0 OAuth2 only (no custom auth), Stripe only (no
  alternative payment), single shared PostgreSQL database, services MUST
  NOT call each other over HTTP, Zod v4 top-level methods only
**Scale/Scope**: ~30 protected routes, ~15 public routes, 5 auth routes,
  63 atom components, 30+ molecule components, 2 locales (en/ar with RTL),
  5 roles, 13 permissions, 4 backend service integrations

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| # | Principle | Status | Notes |
|---|-----------|--------|-------|
| I | Monorepo-First | PASS | Frontend is `packages/frontend` with own `package.json`, `tsconfig.json`, entrypoint. Imports shared code via `@incorpify/shared/...`. No cross-service imports. |
| II | Fastify-First API Design | PASS | Frontend is a Fastify v5 service on port 3004. Auth routes define Zod schemas. Response envelope used for all API client responses. |
| III | Security by Design | PASS | Auth0 OAuth2 with encrypted cookies. Bearer tokens stored in httpOnly cookie, never exposed to client JS. All inputs validated via Zod. No JWT validation in backend services (gateway trust model). Frontend validates tokens as auth originator. |
| IV | Multi-Tenant Isolation | PASS | Every API call includes `X-Space-Id` and `X-Organization-Id` headers. CompanyContext enforces RBAC with 5 roles and 13 permissions. Space-scoped URL parameters in CRM API calls. |
| V | Test-Driven Development | PASS | TDD mandatory. bun test sole runner. Co-located test files. 80%+ coverage. Unit tests use mocks (no network/DB). Integration tests use `app.inject()`. |
| VI | Type Safety and Validation | PASS | TypeScript strict mode. Zod v4 top-level methods. Types inferred from schemas via `z.infer<>`. Shared types in `@incorpify/shared/types/`. `import type` for type-only imports. |
| VII | AI Agent Architecture | PASS | Frontend consumes agents service via SSE streaming. No direct AI SDK usage in frontend — agents service handles all LLM interactions. Conversations scoped to user's space context. |
| VIII | Structured Logging | PASS | Uses `log()` from `@incorpify/shared`. Structured data objects. No string interpolation. Context includes userId, requestId. |
| IX | Frontend Architecture | PASS | React + Radix UI + Tailwind CSS. Atomic Design (atoms/molecules/pages). TanStack React Query with query key factory. React Hook Form + Zod resolvers. i18n with en/ar + RTL. Auth0 encrypted cookies with auto-refresh. SSR for public pages. Graceful degradation on service unavailability. |
| X | Simplicity and Pragmatism | PASS | No Redux/Zustand (React Query + Context sufficient). Fetch-based client (no Axios). No premature abstractions. YAGNI applied to future features. Configuration validated with Zod at startup. |

**Gate result**: ALL PASS — no violations detected.

**Post-Phase 1 re-check**: All principles remain satisfied. The design
artifacts (data-model.md, contracts/, quickstart.md) align with every
constitution principle. The @fastify/vite choice reinforces Principle II
(Fastify-first). The dual-cookie pattern satisfies Principle III (security).
The space-scoped API endpoints satisfy Principle IV (multi-tenant isolation).

## Project Structure

### Documentation (this feature)

```text
specs/001-frontend-migration/
├── plan.md              # This file
├── research.md          # Phase 0: 10 research decisions
├── data-model.md        # Phase 1: Entity definitions + state ownership
├── quickstart.md        # Phase 1: Developer setup guide
├── contracts/
│   ├── api-client.md    # Phase 1: Full API endpoint catalog
│   ├── auth-flow.md     # Phase 1: OAuth2 flow + cookie schema
│   └── route-map.md     # Phase 1: URL structure + route guards
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2: Generated by /speckit.tasks
```

### Source Code (repository root)

```text
packages/frontend/
├── package.json                    # @incorpify/frontend, port 3004
├── tsconfig.json                   # Extends shared tsconfig
├── vite.config.ts                  # Vite 6 + React + SSR config
├── tailwind.config.ts              # Tailwind CSS 4 with RTL plugin
├── postcss.config.ts               # PostCSS with Tailwind
├── index.html                      # Vite HTML entry
├── src/
│   ├── index.ts                    # Fastify server entry (production)
│   ├── app.ts                      # buildApp() — Fastify app factory
│   ├── config/
│   │   └── index.ts                # Zod-validated frontend config
│   ├── plugins/
│   │   ├── auth.ts                 # @fastify/oauth2 + session plugin
│   │   ├── vite.ts                 # @fastify/vite registration
│   │   └── static.ts               # @fastify/static for assets
│   ├── routes/
│   │   ├── auth/
│   │   │   ├── login.ts            # GET /auth/login
│   │   │   ├── callback.ts         # GET /auth/callback
│   │   │   ├── logout.ts           # GET|POST /auth/logout
│   │   │   ├── session.ts          # GET /auth/session
│   │   │   └── refresh.ts          # POST /auth/refresh
│   │   └── health/
│   │       └── index.ts            # GET /health
│   ├── client/
│   │   ├── entry-client.tsx        # Client hydration entry
│   │   ├── entry-server.tsx        # SSR render entry
│   │   ├── root.tsx                # Root layout component
│   │   ├── router.ts              # TanStack Router configuration
│   │   ├── routes/
│   │   │   ├── __root.tsx          # Root route with providers
│   │   │   ├── _locale/
│   │   │   │   ├── index.tsx       # Landing page (SSR)
│   │   │   │   ├── pricing.tsx     # Pricing (SSR)
│   │   │   │   ├── contact-us.tsx  # Contact form (SSR)
│   │   │   │   ├── privacy-policy.tsx
│   │   │   │   ├── terms-and-conditions.tsx
│   │   │   │   ├── cookie-policy.tsx
│   │   │   │   ├── pre-incorporation/
│   │   │   │   │   ├── index.tsx
│   │   │   │   │   └── $conversationId.tsx
│   │   │   │   ├── blog/
│   │   │   │   │   ├── index.tsx
│   │   │   │   │   └── $slug.tsx
│   │   │   │   └── services/
│   │   │   │       ├── index.tsx
│   │   │   │       ├── incorporation.tsx
│   │   │   │       ├── accounting.tsx
│   │   │   │       ├── payroll.tsx
│   │   │   │       ├── insurance.tsx
│   │   │   │       ├── legal.tsx
│   │   │   │       ├── residency.tsx
│   │   │   │       └── banking.tsx
│   │   │   ├── organization/
│   │   │   │   ├── index.tsx       # Org selector
│   │   │   │   └── $spaceId/
│   │   │   │       ├── create-company.tsx
│   │   │   │       ├── settings.tsx
│   │   │   │       ├── billing.tsx
│   │   │   │       └── company/
│   │   │   │           └── $companyId/
│   │   │   │               ├── dashboard.tsx
│   │   │   │               ├── accounting.tsx
│   │   │   │               ├── payroll.tsx
│   │   │   │               ├── tax.tsx
│   │   │   │               ├── banking.tsx
│   │   │   │               ├── insurance.tsx
│   │   │   │               ├── visa-management.tsx
│   │   │   │               ├── compliance.tsx
│   │   │   │               ├── documents.tsx
│   │   │   │               ├── people.tsx
│   │   │   │               ├── members.tsx
│   │   │   │               ├── settings.tsx
│   │   │   │               └── workflows/
│   │   │   │                   ├── index.tsx
│   │   │   │                   └── $workflowId.tsx
│   │   │   └── account/
│   │   │       ├── profile-completion.tsx
│   │   │       ├── email-verification.tsx
│   │   │       └── settings.tsx
│   │   ├── components/
│   │   │   ├── atoms/              # 63 base components (button, input, etc.)
│   │   │   ├── molecules/          # 30+ composites (cards, forms, tables)
│   │   │   └── pages/              # Page-level containers
│   │   ├── context/
│   │   │   ├── authContext.tsx      # Auth state + login/logout actions
│   │   │   └── companyContext.tsx   # Tenant state + RBAC helpers
│   │   ├── hooks/
│   │   │   ├── useAuth.ts          # Auth context consumer
│   │   │   └── useMobile.ts        # Responsive breakpoint hook
│   │   ├── lib/
│   │   │   ├── apiClient.ts        # Fetch-based API client factory
│   │   │   ├── queryClient.ts      # React Query client config
│   │   │   └── queryKeys.ts        # Hierarchical query key factory
│   │   ├── access-layer/
│   │   │   ├── getters/            # React Query query definitions
│   │   │   ├── mutations/          # React Query mutation definitions
│   │   │   └── types/              # API response type definitions
│   │   └── i18n/
│   │       ├── index.ts            # Paraglide.js config
│   │       └── locales/
│   │           ├── en.ts           # English translations
│   │           └── ar.ts           # Arabic translations
│   └── types/
│       └── fastify.d.ts            # Fastify type augmentation
└── tests/                          # Co-located with source files
```

**Structure Decision**: Frontend is a dedicated package (`packages/frontend`)
within the existing Bun monorepo, following the same layout as other services
(agents, checkout, crm, notification). The Fastify server layer lives in
`src/` (routes, plugins, config) while the React client app lives in
`src/client/`. This separation mirrors the SSR/SPA split — Fastify handles
auth routes and SSR, the client handles SPA navigation after hydration.

## Complexity Tracking

> No constitution violations detected. No complexity justifications required.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| (none)    | —          | —                                    |
