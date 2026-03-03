# Quickstart: Frontend Service

**Branch**: `001-frontend-migration` | **Date**: 2026-03-02

## Prerequisites

- Bun >= 1.3
- PostgreSQL 18 running locally (or Docker)
- Auth0 tenant credentials
- Stripe test API keys (for checkout flow)

## Setup

```bash
# From monorepo root
cd /path/to/backend-agents-service

# Install all workspace dependencies
bun install

# Copy environment file
cp .env.example .env
# Edit .env with Auth0 and Stripe credentials

# Run database migrations
bun run db:migrate

# Seed reference data (countries, jurisdictions, services)
bun run db:seed
```

## Run Frontend in Development

```bash
# Start frontend only (port 3004)
bun run --filter @incorpify/frontend dev

# Or start all services together
bun run dev
```

## Run All Services

```bash
# Terminal 1: All backend services
bun run dev

# Services available at:
# - Frontend:     http://localhost:3004
# - Agents:       http://localhost:3000
# - Checkout:     http://localhost:3001
# - CRM:          http://localhost:3002
# - Notification:  http://localhost:3003
```

## Project Structure

```
packages/frontend/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── src/
│   ├── index.ts                 # Fastify server entry
│   ├── app.ts                   # buildApp() function
│   ├── config/
│   │   └── index.ts             # Frontend-specific config
│   ├── routes/
│   │   ├── auth/                # Auth routes (login, callback, logout)
│   │   └── health/              # Health check
│   ├── client/                  # Client-side React app
│   │   ├── entry-client.tsx     # Client entry (hydration)
│   │   ├── entry-server.tsx     # Server entry (SSR)
│   │   ├── root.tsx             # Root layout
│   │   ├── router.ts            # TanStack Router config
│   │   ├── routes/              # Page routes (file-based)
│   │   │   ├── __root.tsx       # Root route with providers
│   │   │   ├── _locale/         # Locale-prefixed public routes
│   │   │   │   ├── index.tsx    # Landing page
│   │   │   │   ├── pricing.tsx
│   │   │   │   ├── blog/
│   │   │   │   └── services/
│   │   │   └── organization/    # Protected routes
│   │   │       ├── index.tsx    # Org selector
│   │   │       └── $spaceId/
│   │   │           └── company/
│   │   │               └── $companyId/
│   │   │                   ├── dashboard.tsx
│   │   │                   ├── payroll.tsx
│   │   │                   ├── tax.tsx
│   │   │                   └── ...
│   │   ├── components/
│   │   │   ├── atoms/           # Base UI (button, input, etc.)
│   │   │   ├── molecules/       # Composite (cards, forms, etc.)
│   │   │   └── pages/           # Page-level layouts
│   │   ├── context/
│   │   │   ├── authContext.tsx
│   │   │   └── companyContext.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   └── useMobile.ts
│   │   ├── lib/
│   │   │   ├── apiClient.ts     # Fetch-based API client
│   │   │   ├── queryClient.ts   # React Query client
│   │   │   └── queryKeys.ts     # Query key factory
│   │   ├── access-layer/
│   │   │   ├── getters/         # Query definitions
│   │   │   ├── mutations/       # Mutation definitions
│   │   │   └── types/           # API response types
│   │   └── i18n/
│   │       ├── index.ts         # i18n config
│   │       └── locales/
│   │           ├── en.ts
│   │           └── ar.ts
│   └── types/
│       └── fastify.d.ts         # Fastify type augmentation
└── tests/
    └── (co-located with source)
```

## Key Commands

```bash
# Development
bun run --filter @incorpify/frontend dev     # Dev server with HMR

# Build
bun run --filter @incorpify/frontend build   # Production build

# Test
bun run --filter @incorpify/frontend test    # Run tests

# Lint
bun run --filter @incorpify/frontend lint    # ESLint

# Type check
bun run --filter @incorpify/frontend typecheck  # tsc --noEmit
```

## Environment Variables

```bash
# Frontend-specific
FRONTEND_PORT=3004
APP_URL=http://localhost:3004

# Auth0
AUTH0_DOMAIN=dev-xvppy7hh2okyawyy.us.auth0.com
AUTH0_CLIENT_ID=...
AUTH0_CLIENT_SECRET=...
AUTH0_AUDIENCE=...

# Session encryption
SESSION_SECRET=... (32+ bytes, hex or base64)

# Backend service URLs (dev only; prod uses API Gateway)
CRM_SERVICE_URL=http://localhost:3002
AGENTS_SERVICE_URL=http://localhost:3000
CHECKOUT_SERVICE_URL=http://localhost:3001
NOTIFICATION_SERVICE_URL=http://localhost:3003

# Stripe (for checkout)
STRIPE_PUBLISHABLE_KEY=pk_test_...
```

## Verification

After setup, verify:

1. Open http://localhost:3004 — landing page renders
2. Click Login — redirects to Auth0
3. Complete login — redirects to dashboard
4. Navigate services — data loads from CRM
5. Open chatbot — conversation works with agents service
