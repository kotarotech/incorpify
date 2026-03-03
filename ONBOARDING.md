# Incorpify Engineering Onboarding Guide

## Table of Contents
1. [Company & Product Overview](#1-company--product-overview)
2. [System Architecture](#2-system-architecture)
3. [Repository Map](#3-repository-map)
4. [Timeline & Trajectory](#4-timeline--trajectory)
5. [Development Setup & Workflow](#5-development-setup--workflow)
6. [Critical Codebase Knowledge](#6-critical-codebase-knowledge)
7. [What Needs to Happen Next (Ship to Customers)](#7-what-needs-to-happen-next)
8. [Highest Impact Things to Build](#8-highest-impact-things-to-build)
9. [Known Issues & Tech Debt](#9-known-issues--tech-debt)
10. [Key People & External Services](#10-key-people--external-services)

---

## 1. Company & Product Overview

**Incorpify** is a B2B SaaS platform for **business incorporation and company management** in the **UAE and Saudi Arabia (KSA)**. It helps entrepreneurs incorporate companies, then manage ongoing operations: payroll, taxes, banking, insurance, visas, compliance, and legal documents.

### Core Value Proposition
- AI-powered consultation for business setup decisions
- End-to-end incorporation workflow (choose jurisdiction → file paperwork → get trade license)
- Post-incorporation management dashboard (accounting, payroll, tax, visas, banking, insurance)
- Multi-tenant organizations with role-based access

### Revenue Model
- Subscription plans via Stripe (billing/plan management in-app)
- Service-based pricing for incorporation and compliance
- AI-guided upselling of services during pre-incorporation consultation

### Target Markets
- **UAE**: Free zones, mainland, offshore company types
- **KSA**: Saudi Arabia business formation

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USERS (Browser)                             │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CloudFront CDN → ALB (HTTPS :443) → Traefik Reverse Proxy        │
│  Domain: prod.incorpify.ai                                        │
└──────┬─────────┬──────────┬──────────┬──────────┬──────────────────┘
       │         │          │          │          │
       ▼         ▼          ▼          ▼          ▼
┌──────────┐┌──────────┐┌──────────┐┌──────────┐┌──────────┐
│ Frontend ││Management││Multi-    ││Knowledge ││Notifi-   │
│ Next.js  ││ Service  ││Agent     ││Base      ││cation    │
│ :3000    ││ :8080    ││Service   ││Service   ││Service   │
│          ││ (Java)   ││:8081/:3k ││:8082     ││:8083     │
└────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘
     │           │           │           │           │
     │           ▼           ▼           ▼           │
     │      ┌─────────────────────────────────┐      │
     │      │  PostgreSQL 18 (RDS Multi-AZ)   │      │
     │      │  + pgvector (semantic search)    │      │
     │      └─────────────────────────────────┘      │
     │           │                                    │
     │           ▼                                    │
     │      ┌──────────────┐                          │
     │      │ ElastiCache  │                          │
     │      │ Redis 7.1    │                          │
     │      └──────────────┘                          │
     │                                                │
     ▼                                                ▼
┌──────────────────────┐    ┌──────────────────────────────┐
│  Auth0 (OAuth2/OIDC) │    │  External Services           │
│  dev-xvppy7hh2okya.. │    │  - Stripe (payments)         │
│                      │    │  - Zendesk (support)         │
│                      │    │  - ShuftiPro (KYC/identity)  │
│                      │    │  - Azure OpenAI (LLM)        │
│                      │    │  - Azure Blob (doc storage)  │
│                      │    │  - Azure Comms (email)       │
│                      │    │  - Airtable (forms/CRM)      │
└──────────────────────┘    └──────────────────────────────┘
```

### Key Architecture Decisions
- **ECS Fargate** on AWS for all services (containerized, serverless compute)
- **Traefik** as internal reverse proxy for service routing
- **ECS Service Connect** for inter-service communication (private DNS)
- **Auth0** for identity/auth (OAuth2 + PKCE, RBAC)
- **Single shared PostgreSQL database** across services (separate schemas via Liquibase/Drizzle migrations)
- **Azure OpenAI** still used for LLM despite AWS migration (provider-agnostic in new agents service)
- **Region**: `me-central-1` (Middle East/Bahrain) as primary

### The Azure → AWS Migration Story
The company was originally on Azure (Azure Container Registry, Azure DevOps pipelines, Azure Kubernetes). A managed services firm called **Cloud Softway** built the AWS infra in Terraform. The migration happened in **Jan 2026**. The backend repos still carry Azure naming/config as legacy artifacts. The `backend-agents-service` is a ground-up TypeScript rewrite that replaces the Java `incorpify-multi-agent-service`.

---

## 3. Repository Map

| Repo | Tech | Purpose | Status | Lines of Code |
|------|------|---------|--------|---------------|
| **azure-backup-frontend** | Next.js 15 / React 19 / TypeScript | Customer-facing web app | Production, 1001 commits | 867 TSX files |
| **azure-backup-backend** | Java 17 / Spring Boot 3.4 / Maven | Core API (management, notifications, knowledge) | Production, 500+ commits | 1,112 Java files |
| **backend-agents-service** | Bun / TypeScript / Fastify 5 | AI agent orchestration (rewrite of Java multi-agent) | Near-production, 120 commits | 19,430 lines |
| **aws-infra** | Terraform | AWS infrastructure-as-code | Complete, 2 commits | 33 modules, 22 .tf files |

### Repo Relationships
```
azure-backup-frontend  ──HTTP──►  azure-backup-backend (Management Service :8080)
                       ──HTTP──►  backend-agents-service (Multi-Agent :8081/3000)

backend-agents-service ──HTTP──►  azure-backup-backend (knowledge-base :8082)
                       ──HTTP──►  azure-backup-backend (management :8080 for company data)

azure-backup-backend   ──Kafka──► notification-service (email delivery)
                       ──HTTP──►  Stripe, Zendesk, ShuftiPro, Auth0

aws-infra              ──provisions──► all AWS resources
```

---

## 4. Timeline & Trajectory

### Development Timeline

| Period | Key Activity |
|--------|-------------|
| **Pre-2025** | Initial product build on Azure (Java Spring Boot backend, Next.js frontend) |
| **2025 Q1-Q3** | Feature development: incorporation workflows, payroll, tax, insurance, banking, visa management, compliance, AI agents |
| **2025-10-01** | Backend v0.0.8 release (latest tagged release) |
| **2025-10-06** | Bug fixes: invitation flow, recent activity logging |
| **2025-11-30** | AWS migration begins; backend CI/CD rewrite for GitHub Actions |
| **2025-12-14** | GitHub Actions workflows for all 4 backend services |
| **2025-12-28** | CI/CD pipeline optimization (latest backend commit) |
| **2026-01-23** | AWS Terraform infra uploaded (Cloud Softway) |
| **2026-01-28** | Frontend: SEO improvements, production sitemap |
| **2026-02-10** | `backend-agents-service` rewrite begins (Bun/TypeScript) |
| **2026-02-20** | Agent service: 15 agents, 26 tools, 379 tests all passing |
| **2026-02-25** | Agent service: Gemini provider added, Docker workflow enhanced |
| **2026-02-26** | Frontend: latest sync from Azure to GitHub |
| **2026-02-27** | Agent service: latest backup (current state) |

### Trajectory Analysis
- **Backend (Java)**: Stable, feature-complete for current scope. No new features since Dec 2025 — team focus shifted to infra migration and agent rewrite.
- **Frontend**: Active SEO work, but core app features appear complete. Mostly maintenance + polish.
- **Agent Service (Bun)**: Extremely rapid development (120 commits in ~17 days). Core architecture complete but needs wiring to real services.
- **Infrastructure**: Professionally built by Cloud Softway, ~85-90% production ready.

### Team Signals
- **Alejandro F. Carrera** (`afc-incorpify`) — Primary backend/infra engineer, handles Azure↔GitHub syncs
- **grey** — Frontend engineer, SEO improvements
- **Cloud Softway team** (Abdelhamed, Anwar, Mohammad) — AWS infrastructure
- Small team (3-5 engineers visible in commits)

---

## 5. Development Setup & Workflow

### Prerequisites
- **Java 17** (via `brew install openjdk@17`) + **Maven 3.9** (via `brew install maven`)
- **Node.js 20.15.1** + npm 10.7.0 (frontend)
- **Bun >= 1.2** (agents service — `bun:sql` requires 1.2+; upgrade with `bun upgrade`)
- **Docker + Docker Compose** (for PostgreSQL)

> **Important**: If you have a local PostgreSQL installed (e.g. via Homebrew), stop it first with `brew services stop postgresql@14` (or whichever version). It will conflict with the Docker PostgreSQL on port 5432.

### Quick Start: Full Stack Local

```bash
# 1. Start shared infrastructure (PostgreSQL via Docker)
cd /Users/omidahourai/Development/incorpify/azure-backup-backend
docker-compose up -d postgres   # PostgreSQL on :5432 (user: incorpify, pass: incorpify, db: incorpify)

# 2. Build & start management service (Java) — in its own terminal
cd /Users/omidahourai/Development/incorpify/azure-backup-backend
mvn clean install -DskipTests
cd incorpify-management-service
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=local"
# Management API on :8080

# 3. Start agents service (Bun/TypeScript) — in its own terminal
cd /Users/omidahourai/Development/incorpify/backend-agents-service
bun install
# Edit .env: set DATABASE_URL=postgresql://incorpify:incorpify@localhost:5432/incorpify
# Edit .env: set PORT=3001 (to avoid conflict with frontend on :3000)
bun run db:migrate
bun run dev             # Agents on :3001

# 4. Start frontend — in its own terminal
cd /Users/omidahourai/Development/incorpify/azure-backup-frontend
npm install -f
cp .env.example .env.development  # Fill in AUTH0 and API URLs
npm run dev             # Frontend on :3000
```

### Gotchas We Hit During Setup
- **No `mvnw` wrapper** — use system `mvn` instead of `./mvnw`
- **`bun:sql` requires Bun >= 1.2** — if you get `SQL is not a constructor`, run `bun upgrade`
- **Port 3000 conflict** — frontend and agents service both default to 3000. Set `PORT=3001` in agents `.env`
- **Local PostgreSQL steals port 5432** — stop any Homebrew postgres before starting Docker
- **`migrate.ts` missing `initializeDatabase()` call** — must call `initializeDatabase()` before `getDatabase()` in the migration script

### Environment Variables Needed

**Critical secrets you'll need from the team:**
- `AUTH0_CLIENT_ID`, `AUTH0_CLIENT_SECRET`, `AUTH0_DOMAIN` — Auth0 tenant credentials
- `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT` — LLM access
- `STRIPE_API_SECRET_KEY`, `STRIPE_WEBHOOK_KEY` — Payment processing
- `DATABASE_URL` or `SPRING_DATASOURCE_*` — Database connection

**Auth0 Dev Tenant (from config files):**
- Domain: `dev-xvppy7hh2okyawyy.us.auth0.com`
- Client ID: `DVPwCTU5VlHaojbgVA73ylt8eZBB7wJO` (backend)
- Frontend Client ID: `p32MQQ5Mm6Yk3uEbam84yEizjYLBwQ80`

### API Documentation (verified working)
- **Management API Swagger UI**: http://localhost:8080/swagger-ui/index.html
- **Management API docs (JSON)**: http://localhost:8080/api-docs
- **Agents Service health**: http://localhost:3001/health
- **Agents Service info**: http://localhost:3001/api/v1/info
- **Knowledge Base Swagger**: http://localhost:8082/swagger-ui/index.html (requires starting knowledge-base service separately)
- **Notification Swagger**: http://localhost:8083/swagger-ui/index.html (requires starting notification service separately)

> **Note**: The `/swagger-ui.html` path returns 401 — use `/swagger-ui/index.html` instead. Only the management service (:8080) runs from the quick start above. The knowledge-base (:8082) and notification (:8083) services need separate `mvn spring-boot:run` commands in their respective directories. The legacy Java multi-agent service (:8081) is being replaced by the agents service.

### Git Workflow
- **Branching**: GitFlow (`develop`, `master`, `feature/*`, `bugfix/*`, `release/*`, `hotfix/*`)
- **Commits**: Conventional commits (`feat:`, `fix:`, `refactor:`, etc.)
- **CI/CD**: GitHub Actions (AWS) + Azure Pipelines (legacy)

### Key Commands by Repo

| Action | Frontend | Backend (Java) | Agents Service |
|--------|----------|----------------|----------------|
| Install | `npm install -f` | `mvn clean install -DskipTests` | `bun install` |
| Dev server | `npm run dev` | `mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=local"` | `bun run dev` |
| Build | `npm run build` | `mvn package` | `bun run build` |
| Test | `npm run lint` | `mvn test` | `bun run test` |
| Lint | `npm run lint` | — | `bun run lint` |
| Format | `npm run format` | — | `bun run format` |
| DB migrate | — | auto (Liquibase) | `bun run db:migrate` |
| Docker | `docker-compose up` | `docker-compose up` | `cd .docker && ./docker-dev.sh quickstart` |

---

## 6. Critical Codebase Knowledge

### Frontend Architecture (Next.js 15)

**Routing**: App Router with locale prefix (`/en/...`). Three route groups:
- `(public)` — Landing, services, blog, legal pages (no auth)
- `(private)` — Profile completion, email verification (partially protected)
- `organization/[orgId]/company/[companyId]/...` — Main app (fully protected)

**Auth Flow**: Auth0 OAuth2 → encrypted cookies (iron-session) → Axios interceptor auto-attaches Bearer token → auto-refreshes on 401.

**Data Layer**: React Query for server state. Access layer pattern:
- `src/access-layer/getters/` — Query definitions (100+ endpoints)
- `src/access-layer/mutations/` — Mutation definitions
- `src/access-layer/types/` — TypeScript interfaces
- `src/shared/query-keys.ts` — Query key factory

**Component System**: Atomic design:
- `src/components/atoms/` — Radix UI wrappers, form controls (60+ files)
- `src/components/molecules/` — Business-specific composite components
- `src/components/pages/` — Full page templates

**Critical Files**:
- `src/middleware.ts` — 237 lines routing all requests (auth, i18n, redirects)
- `src/lib/axios/axios-service.ts` — HTTP client with auth interceptors
- `src/lib/auth/session/manager.ts` — Session encryption/management
- `src/context/AuthContext.tsx` — Auth state for all components

### Backend Architecture (Java Spring Boot)

**4 microservices** in a Maven multi-module project:
1. **Management Service** (:8080) — 42 controllers, core business logic
2. **Multi-Agent Service** (:8081) — AI chat (being replaced by agents-service)
3. **Knowledge Base Service** (:8082) — Semantic search with Azure AI
4. **Notification Service** (:8083) — Templated email delivery

**Auth**: Auth0 JWT validation via Spring Security OAuth2 resource server. Custom annotations: `@RequiresRolePermission`, `@RequiresOrganizationAccess`.

**Database**: 350+ Liquibase migration files. 193+ domain entities. 119 repository classes.

**Key Patterns**:
- OpenFeign for inter-service HTTP calls
- Spring State Machine for multi-step workflows (incorporation, compliance)
- MapStruct for entity/DTO mapping
- ShedLock for distributed job scheduling

**Critical Files**:
- `incorpify-management-service/` — The big one. 42 controllers handle everything.
- `incorpify-common/` — Shared auth, exceptions, DTOs
- `docker-compose.yml` — Local dev orchestration

### Agents Service Architecture (Bun/TypeScript)

**Complete rewrite** of the Java multi-agent service with massive performance improvements.

**15 AI Agents** organized by domain:
- Pre-incorporation (5): General, Supervisor, KSA, UAE, Services
- Incorporation (2): Main workflow, Progress tracking
- Company management (5): Supervisor, Employee, Insurance, Payroll, UBO
- Specialized (3): Banking, Tax, Compliance

**26 Tools** for agent function-calling (Vercel AI SDK):
- `preIncorporationTools.ts` — Country selection, routing
- `incorporationTools.ts` — State management, service selection
- `companyTools.ts` — CRUD operations
- `bankingTools.ts`, `taxTools.ts`, `complianceTools.ts` — Domain tools
- `semanticSearchTool.ts` — Knowledge base integration

**3 LLM Providers**: Azure OpenAI, OpenAI, Anthropic Claude (swappable via config)

**Critical Files**:
- `src/agents/base/` — BaseAgent abstract class (all agents extend this)
- `src/agents/prompts/` — 31 TypeScript prompt modules (extensive domain knowledge)
- `src/infrastructure/ai/AIProviderFactory.ts` — Provider switching logic
- `src/services/conversationService.ts` — Core conversation state management
- `src/config/index.ts` — Zod-validated configuration (40+ env vars)

### Infrastructure (Terraform)

**AWS services provisioned**: VPC, ECS Fargate (5 services), RDS PostgreSQL (Multi-AZ), ElastiCache Redis, ALB, CloudFront, WAF, S3, ECR, CloudWatch, CloudTrail, Security Hub, GuardDuty, SNS budgets.

**Monthly budget**: ~$5,000 (RDS $2,900 is the biggest cost)

**Critical Files**:
- `prod/terraform/environment/prod.tfvars` — 1,454 lines of production config
- `prod/terraform/SSM_PARAMETERS_SETUP.md` — All required secrets for SSM Parameter Store

---

## 7. What Needs to Happen Next

### To Get This in Front of Customers

#### P0: Critical Blockers (Must Do)

1. **Wire Up the Agents Service to Real Services**
   - The `backend-agents-service` has 26 tools and 15 agents but they call mock/stubbed endpoints
   - Need to connect to real Management Service (:8080) for company data, Knowledge Base (:8082) for semantic search
   - Status: Config variables exist, HTTP clients are stubbed

2. **Complete Auth0 Token Validation in Agents Service**
   - Currently accepts any Bearer token without validating against Auth0 JWKS
   - Must validate JWT signature, expiration, audience claims
   - Critical for production security

3. **Run Database Migrations in Production**
   - Agents service Drizzle migrations need to be applied to the production RDS
   - Backend Liquibase migrations should already be running (auto on startup)

4. **SSM Parameter Store Population**
   - 50+ secrets need to be loaded into AWS SSM (documented in `SSM_PARAMETERS_SETUP.md`)
   - Auth0, Stripe, Azure OpenAI, Zendesk, ShuftiPro credentials
   - Currently a hardcoded DB password in `prod.tfvars` needs to be rotated

5. **Deploy Agents Service to ECS**
   - Docker image needs to be pushed to ECR
   - ECS task definition needs to be created/updated
   - Service Connect needs to register the new service

#### P1: High Priority (Should Do Before Launch)

6. **Frontend Port Conflict Resolution**
   - Both frontend and agents service default to port 3000
   - Need to standardize ports across docker-compose files

7. **End-to-End Testing of Core Flows**
   - Incorporation flow: signup → AI consultation → choose jurisdiction → submit documents → get license
   - Payment flow: select plan → Stripe checkout → subscription active
   - No integration/E2E test suites exist currently

8. **Fix the Hardcoded Database Password**
   - `prod.tfvars` line 115 has `password = "InCorpify12345*"` — needs to be moved to SSM/secrets

9. **AI Agent Streaming Integration**
   - SSE endpoints exist but use mock data
   - Need real LLM streaming through the agent pipeline

10. **Frontend Advanced AI Module Completion**
    - Routes exist but UI may be incomplete
    - Need to ensure conversation UI works with new agents service

#### P2: Important (Near-Term)

11. **Monitoring & Observability**
    - CloudWatch dashboards are provisioned but need app-level metrics
    - No distributed tracing across services
    - Agents service has no metrics endpoint yet

12. **KSA Flow Completion**
    - Feature branches exist (`feat/ksa-flow-tmp`) but appear unmerged
    - KSA-specific agents and prompts are built in agents service

13. **Redis Caching**
    - Branch `feature/introduce-redis-cacheable` exists but unmerged
    - ElastiCache Redis is provisioned and paying $1,120/month but likely unused

---

## 8. Highest Impact Things to Build

### Ranked by Customer Impact × Effort

#### 1. **Complete the AI-to-Action Pipeline** ⭐⭐⭐⭐⭐
**Impact**: This IS the product differentiator. Currently, AI agents can chat but can't execute real actions.
**What**: Wire agent tools to real backend APIs so the AI can actually:
- Look up company information and documents
- Submit incorporation applications
- Check compliance status
- Search the knowledge base for regulations
**Effort**: Medium (architecture exists, needs HTTP client implementation)
**Files**: `backend-agents-service/src/tools/*.ts`, `src/infrastructure/clients/baseClient.ts`

#### 2. **Self-Service Onboarding Flow** ⭐⭐⭐⭐⭐
**Impact**: The company is "struggling to put the product in front of users and avoid manual labor." A frictionless signup → AI consultation → plan selection → payment → dashboard flow eliminates manual sales.
**What**: Ensure the full journey works end-to-end:
- Landing page → "Get Started" → Auth0 signup
- Profile completion → AI pre-incorporation agent conversation
- Agent recommends jurisdiction + services → user selects plan
- Stripe checkout → subscription activated → dashboard populated
**Effort**: Medium-High (most pieces exist, need integration testing and gap-filling)
**Where gaps are**: Payment flow completion, agent-to-checkout handoff, post-payment provisioning

#### 3. **Automated Document Processing** ⭐⭐⭐⭐
**Impact**: Document handling is likely a huge source of manual labor (trade licenses, visa docs, compliance docs, financial statements).
**What**: Build automated document intake → validation → status tracking:
- OCR/parsing of uploaded documents
- AI-powered document classification
- Automated completeness checking
- Status notifications to users
**Effort**: High (new feature, but Azure Blob storage and notification service exist)

#### 4. **Customer Dashboard Improvements** ⭐⭐⭐⭐
**Impact**: Users need to see their company status at a glance without contacting support.
**What**:
- Real-time incorporation progress tracker
- Document completion percentage (recently fixed a precision bug — indicates this matters)
- Upcoming deadlines (tax filing, visa renewals, license expiry)
- Action items / tasks requiring user input
**Effort**: Medium (frontend work, backend endpoints mostly exist)

#### 5. **Notification & Communication Automation** ⭐⭐⭐⭐
**Impact**: Reduces manual follow-up. Users get proactive updates instead of having to check.
**What**:
- Email notifications for key events (document approved, payment due, visa expiring)
- In-app notification center (partially built)
- WhatsApp integration (critical for Middle East market)
- Scheduled reminders for upcoming deadlines
**Effort**: Medium (notification service exists, needs more triggers and channels)

#### 6. **Multi-Language Support (Arabic)** ⭐⭐⭐
**Impact**: UAE and KSA markets require Arabic. i18n infrastructure exists (`next-international`) but only English is implemented.
**What**: Translate all UI strings, add RTL support, Arabic AI agent prompts
**Effort**: High (translation + RTL CSS + prompt engineering)

#### 7. **Admin Dashboard / Internal Tools** ⭐⭐⭐
**Impact**: Reduces manual labor for the Incorpify team managing customers.
**What**: Internal dashboard for operations team:
- View all customer companies and their status
- Manage incorporation applications
- Assign tasks to team members
- Track revenue and subscriptions
- Agent conversation monitoring
**Effort**: Medium-High (new frontend, backend admin endpoints partially exist)

---

## 9. Known Issues & Tech Debt

### Security Issues
- [ ] **Hardcoded DB password** in `aws-infra/prod/terraform/environment/prod.tfvars` (`InCorpify12345*`)
- [ ] **Jump server SSH open to 0.0.0.0/0** (should be VPN-only)
- [ ] **Auth0 credentials in frontend docker-compose.yml** and `.env.production`
- [ ] **Agents service accepts any Bearer token** without JWKS validation
- [ ] **EKS public API endpoint** enabled (should restrict)

### Code Quality
- [ ] Frontend `icons.tsx` is 70KB — should be split/tree-shaken
- [ ] Backend has unmerged feature branches (KSA flow, Redis caching, MFA bugs)
- [ ] Agent service repositories use mock implementations
- [ ] No integration or E2E test suites anywhere
- [ ] Frontend uses `npm install -f` (force flag suggests dependency conflicts)

### Infrastructure
- [ ] ElastiCache Redis provisioned at $1,120/month but likely unused
- [ ] EKS cluster provisioned but minimal usage (single node)
- [ ] DocumentDB, Lambda, DynamoDB budgeted but not deployed
- [ ] No secret rotation automation

### Architecture
- [ ] Still using Azure services (OpenAI, Blob, Comms) despite AWS migration
- [ ] Two multi-agent implementations exist (Java :8081 and Bun :3000) — need to deprecate Java one
- [ ] Frontend and agents service both default to port 3000
- [ ] No service mesh or distributed tracing
- [ ] Kafka dependency in notification service but unclear if Kafka is provisioned on AWS

---

## 10. Key People & External Services

### Team (from git history)
| Name | Role | Active In |
|------|------|-----------|
| Alejandro F. Carrera | Backend/Infra Lead | All repos |
| grey | Frontend Engineer | Frontend |
| Cloud Softway (Abdelhamed, Anwar, Mohammad) | AWS Infrastructure | aws-infra |

### External Services & Accounts
| Service | Purpose | Config Location |
|---------|---------|-----------------|
| **Auth0** | Authentication (OAuth2/OIDC) | `dev-xvppy7hh2okyawyy.us.auth0.com` |
| **Stripe** | Payments/Subscriptions | Backend env vars |
| **Azure OpenAI** | LLM (GPT-4o-mini) | `openai-local-incorpify-swedencentral.openai.azure.com` |
| **Azure Blob Storage** | Document storage | Backend config |
| **Azure Communication Services** | Email delivery | Notification service |
| **Azure Cognitive Search** | Vector store for knowledge base | Knowledge base service |
| **Zendesk** | Customer support tickets | Management service |
| **ShuftiPro** | KYC/Identity verification | Management service |
| **Airtable** | CDD forms, contact forms | Frontend env vars |
| **Google Analytics** | Website analytics | `G-GLMQY0DV8D` |
| **Meta Pixel** | Facebook ad tracking | `2844693682382339` |

### Domain & Hosting
- **Production domain**: `prod.incorpify.ai`
- **AWS Region**: `me-central-1` (Middle East/Bahrain)
- **Container Registry**: AWS ECR (migrated from Azure ACR)
- **CI/CD**: GitHub Actions (primary), Azure Pipelines (legacy)

---

## Quick Reference: What's Where

| If you need to... | Go to... |
|---|---|
| Change the landing page | `azure-backup-frontend/src/components/landing-v2/` |
| Add a new API endpoint | `azure-backup-backend/incorpify-management-service/src/main/java/.../controller/` |
| Modify an AI agent prompt | `backend-agents-service/src/agents/prompts/` |
| Add a new AI tool | `backend-agents-service/src/tools/` |
| Change auth behavior | `azure-backup-frontend/src/lib/auth/` |
| Add a database table | Backend: Liquibase migrations / Agents: `backend-agents-service/src/infrastructure/db/schema/` |
| Modify infrastructure | `aws-infra/prod/terraform/` |
| Add a frontend page | `azure-backup-frontend/src/app/[locale]/organization/[organizationId]/company/[companyId]/` |
| Debug API calls | `azure-backup-frontend/src/lib/axios/axios-service.ts` |
| View all routes | `azure-backup-frontend/src/shared/routes.ts` (280+ route functions) |
| View all query keys | `azure-backup-frontend/src/shared/query-keys.ts` |
| Run all agents service tests | `cd backend-agents-service && bun run test` |
| View Swagger docs | `http://localhost:8080/swagger-ui.html` |
