# Research: Frontend Migration to Monorepo

**Branch**: `001-frontend-migration` | **Date**: 2026-03-02

## R1: Frontend Framework for Bun Monorepo

### Decision: @fastify/vite with React 19 + Vite

### Rationale

The frontend will be a Fastify-based service using `@fastify/vite`
to serve a React 19 application with server-side rendering. This
approach was chosen because:

1. **Fastify alignment**: All backend services use Fastify v5. The
   frontend follows the same plugin-based architecture, reusing
   shared plugins (auth, logging, CORS, helmet, rate-limit).
2. **Bun workspace compatibility**: Vite is natively Bun-compatible.
   No Node.js-specific build pipeline is needed.
3. **SSR capability**: @fastify/vite handles server-side rendering
   for SEO-critical public pages (landing, pricing, blog) while
   serving the authenticated SPA for dashboard routes.
4. **Monorepo fit**: Registers as `packages/frontend` alongside
   existing services. Uses `workspace:*` for `@incorpify/shared`.
5. **Port allocation**: Runs on port 3004 per constitution.

### Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| Next.js 15 in monorepo | Known Bun workspace issues; brings own Express-like server incompatible with Fastify; separate build pipeline |
| Vite SPA (no SSR) | Fails SEO requirement for public pages (SC-006) |
| vite-plugin-ssr standalone | More manual setup; no native Fastify integration |

---

## R2: Client-Side Routing

### Decision: TanStack Router

### Rationale

TanStack Router provides type-safe file-based routing with built-in
SSR support, data loading via loaders, and search param validation
with Zod. It integrates naturally with TanStack React Query for
data prefetching.

### Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| React Router v7 | Less type-safe; no native Zod integration; SSR setup more manual |
| Next.js App Router | Coupled to Next.js framework (rejected in R1) |
| Wouter | Too minimal; no SSR, no data loading |

---

## R3: Authentication in Fastify Frontend

### Decision: Auth0 OAuth2 via @fastify/oauth2 + encrypted cookie sessions

### Rationale

The frontend Fastify server handles the OAuth2 authorization code
flow directly:

1. User clicks "Login" → Fastify redirects to Auth0 `/authorize`
2. Auth0 redirects back to Fastify callback route
3. Fastify exchanges code for tokens, stores in encrypted cookie
4. Subsequent requests read session from cookie, inject Bearer token
   into API calls to backend services

This replaces the old Next.js iron-session approach with Fastify's
native `@fastify/secure-session` or `@fastify/cookie` + `iron-session`
equivalent. The auth plugin from `@incorpify/shared` handles Bearer
token extraction for backend API calls.

Key difference from backend services: The frontend service DOES
validate tokens (it's the auth originator), while backend services
trust the gateway per constitution Principle III.

### Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| Auth0 SPA SDK (client-side only) | No server-side token; can't pre-render authenticated content; tokens exposed to client JS |
| Proxy through CRM service | Adds latency; CRM is not an auth service; violates service boundaries |

---

## R4: State Management

### Decision: TanStack React Query + React Context (no Redux/Zustand)

### Rationale

The existing frontend uses this exact pattern successfully:

- **Server state**: TanStack React Query v5 with query key factory
- **Auth state**: React Context (`AuthContext`)
- **Tenant state**: React Context (`CompanyContext` with RBAC)
- **Form state**: React Hook Form with Zod resolvers

No global state manager is needed. This matches the existing codebase
and avoids unnecessary complexity (Principle X).

### Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| Zustand | Unnecessary; React Query + Context covers all needs |
| Redux Toolkit | Over-engineered for this use case; adds bundle size |

---

## R5: Component Library Strategy

### Decision: Migrate Radix UI + Tailwind CSS atoms/molecules directly

### Rationale

The existing frontend has 63 atom components and 30+ molecule
components built on Radix UI primitives with Tailwind CSS. These
are framework-agnostic React components that work identically in
Vite. The migration strategy:

1. Copy atom components (button, input, dialog, etc.) into
   `packages/frontend/src/components/atoms/`
2. Copy molecule components into `molecules/`
3. Adapt any Next.js-specific patterns (e.g., `next/image` →
   standard `<img>` or `@unpic/react`, `next/link` → TanStack
   Router `<Link>`)
4. Preserve Tailwind CSS configuration and design tokens

No component library rebuild is required. The shadcn/ui-style
approach (copy + own) means components are already decoupled
from Next.js.

---

## R6: Internationalization

### Decision: Paraglide.js (i18next alternative optimized for Vite)

### Rationale

The existing frontend uses `next-international` which is
Next.js-specific. For the Vite-based frontend:

- Paraglide.js compiles translations at build time (tree-shakeable)
- Supports locale-prefixed URLs (`/en/...`, `/ar/...`)
- Works with SSR (cookies for server-side locale detection)
- Supports RTL via HTML `dir` attribute + Tailwind CSS `rtl:` prefix
- Tiny runtime (~1KB) vs i18next (~40KB)

### Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| next-international | Next.js-specific; incompatible with Vite |
| i18next + react-i18next | Large bundle; runtime translation lookup not needed with Vite |
| FormatJS/react-intl | Heavier; ICU syntax overkill for this use case |

---

## R7: API Client for Backend Services

### Decision: Fetch-based client with typed wrappers (replace Axios)

### Rationale

The existing frontend uses Axios with a custom interceptor
architecture. For the new frontend:

- Bun and modern browsers have native `fetch` — no library needed
- Type-safe wrappers infer response types from Zod schemas
- Request interceptor pattern replaced by a `createApiClient()`
  factory that injects auth headers and tenant context
- Aligns with backend's approach (no external HTTP libraries)
- Reduces bundle size (~30KB saved by dropping Axios)

The client factory accepts `baseUrl`, reads session cookie for
auth token, and injects `Authorization`, `X-Space-Id`, and
`X-Organization-Id` headers automatically.

### Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| Axios (keep existing) | Unnecessary dependency; fetch is sufficient |
| ky | Extra dependency for minimal benefit over fetch |
| openapi-fetch | Requires OpenAPI spec generation; premature |

---

## R8: Build and Deployment

### Decision: Vite build → Fastify server in Docker (Bun image)

### Rationale

The build pipeline mirrors existing backend services:

1. `vite build` produces client bundle + SSR server entry
2. Fastify server imports SSR entry and serves it
3. Docker multi-stage build with `oven/bun` image
4. Deploy via ECS Fargate on port 3004
5. ALB routes `/` traffic to frontend service
6. API routes (`/api/v1/*`, `/agents/v1/*`) route to backend services

Static assets served with `@fastify/static` and aggressive caching
headers. SSR pages rendered on-demand with optional caching.

---

## R9: CRM API Endpoint Mapping

### Decision: Map existing frontend access layer to new CRM routes

### Rationale

The old frontend calls Java Spring Boot at `localhost:8080/api/v1`.
The new CRM service exposes the same data via Fastify at port 3002
with different URL patterns. Key mappings:

| Old Frontend Endpoint | New CRM Endpoint |
|----------------------|------------------|
| `GET /companies` | `GET /api/v1/spaces/:spaceId/companies` |
| `GET /companies/:id` | `GET /api/v1/spaces/:spaceId/companies/:companyId` |
| `POST /companies/:id/invitations/send` | `POST /api/v1/spaces/:spaceId/companies/:companyId/invitations` |
| `GET /companies/:id/members` | `GET /api/v1/spaces/:spaceId/companies/:companyId/members` |
| `GET /users/profile` | `GET /api/v1/profiles/me/companies` |

Critical change: All company-scoped endpoints now require
`spaceId` in the URL path (multi-tenant enforcement per
Principle IV). The old `incorpify-org` header is replaced by
the URL parameter.

---

## R10: Session Management in Fastify

### Decision: @fastify/secure-session with dual cookie pattern

### Rationale

Replaces iron-session from Next.js. The pattern remains:

- **Public cookie** (`incorpify.session`): User info, org/company
  selection, expiry. Readable by client JS for auth state.
- **Private cookie** (`incorpify.session.private`): access_token,
  refresh_token, id_token. httpOnly, not accessible to client JS.
- **CSRF protection**: SameSite=Lax + CSRF token for mutations

@fastify/secure-session uses libsodium for encryption (same
security guarantees as iron-session). Fastify plugin registration
follows existing patterns.
