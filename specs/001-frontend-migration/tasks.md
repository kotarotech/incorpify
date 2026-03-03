# Tasks: Frontend Migration to Monorepo

**Input**: Design documents from `/specs/001-frontend-migration/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ (api-client.md, auth-flow.md, route-map.md), quickstart.md

**Tests**: Not explicitly requested in the feature specification. Test tasks are excluded.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Monorepo package**: `packages/frontend/` at repository root
- **Server code**: `packages/frontend/src/` (Fastify routes, plugins, config)
- **Client code**: `packages/frontend/src/client/` (React app, components, hooks)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization, package scaffolding, and build tooling

- [ ] T001 Create `packages/frontend/` directory structure per plan.md project structure
- [ ] T002 Create `packages/frontend/package.json` with name `@incorpify/frontend`, port 3004, and all dependencies (fastify v5, @fastify/vite, react 19, vite 6, tanstack router, tanstack react-query v5, radix-ui, tailwindcss 4, react-hook-form, zod v4, @fastify/secure-session, @fastify/oauth2, paraglide-js, @fastify/static)
- [ ] T003 [P] Create `packages/frontend/tsconfig.json` extending shared tsconfig with strict mode, JSX react-jsx, path aliases for `@/` → `src/client/`
- [ ] T004 [P] Create `packages/frontend/vite.config.ts` with React plugin, SSR config, TanStack Router file-based routing, Paraglide.js plugin, path aliases, dev server on port 3004
- [ ] T005 [P] Create `packages/frontend/tailwind.config.ts` with Tailwind CSS 4 config, RTL plugin (`tailwindcss-rtl`), design tokens matching existing frontend
- [ ] T006 [P] Create `packages/frontend/postcss.config.ts` with Tailwind CSS and autoprefixer
- [ ] T007 [P] Create `packages/frontend/index.html` as Vite HTML entry with `<div id="root">` and script tag pointing to `src/client/entry-client.tsx`
- [ ] T008 Update root `package.json` workspace config to include `packages/frontend` in Bun workspace

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Server Foundation

- [ ] T009 Create Zod-validated frontend config in `packages/frontend/src/config/index.ts` covering FRONTEND_PORT, APP_URL, AUTH0_DOMAIN, AUTH0_CLIENT_ID, AUTH0_CLIENT_SECRET, AUTH0_AUDIENCE, SESSION_SECRET, CRM_SERVICE_URL, AGENTS_SERVICE_URL, CHECKOUT_SERVICE_URL, NOTIFICATION_SERVICE_URL, STRIPE_PUBLISHABLE_KEY
- [ ] T010 Create Fastify app factory `buildApp()` in `packages/frontend/src/app.ts` with plugin registration order: config → secure-session → oauth2 → vite → static → routes
- [ ] T011 Create Fastify server entry in `packages/frontend/src/index.ts` that calls `buildApp()` and listens on configured port
- [ ] T012 Create Fastify type augmentation in `packages/frontend/src/types/fastify.d.ts` declaring session properties (public + private cookie payloads) on FastifyRequest

### Auth Plugin & Routes

- [ ] T013 Implement auth plugin in `packages/frontend/src/plugins/auth.ts` registering @fastify/secure-session with dual cookie pattern (public `incorpify.session` + private `incorpify.session.private`) and @fastify/oauth2 for Auth0 per auth-flow.md
- [ ] T014 Implement login route in `packages/frontend/src/routes/auth/login.ts` — GET /auth/login with returnTo, referral, invite query params; redirects to Auth0 authorize URL with scopes `openid profile email offline_access`
- [ ] T015 Implement callback route in `packages/frontend/src/routes/auth/callback.ts` — GET /auth/callback; exchanges code for tokens, stores in encrypted cookies, redirects to returnTo or /organization
- [ ] T016 Implement logout route in `packages/frontend/src/routes/auth/logout.ts` — GET/POST /auth/logout; clears all session cookies, redirects to Auth0 /v2/logout
- [ ] T017 [P] Implement session route in `packages/frontend/src/routes/auth/session.ts` — GET /auth/session; returns public session JSON from cookie
- [ ] T018 [P] Implement refresh route in `packages/frontend/src/routes/auth/refresh.ts` — POST /auth/refresh; uses refresh_token to get new access_token from Auth0, updates cookies
- [ ] T019 [P] Implement health route in `packages/frontend/src/routes/health/index.ts` — GET /health returning status, version, uptime

### Vite & Static Plugins

- [ ] T020 [P] Implement Vite plugin in `packages/frontend/src/plugins/vite.ts` registering @fastify/vite with React SSR renderer
- [ ] T021 [P] Implement static plugin in `packages/frontend/src/plugins/static.ts` registering @fastify/static for built assets with cache headers

### Client Foundation

- [ ] T022 Create client entry for hydration in `packages/frontend/src/client/entry-client.tsx` — React hydration with Router and QueryClient providers
- [ ] T023 Create SSR render entry in `packages/frontend/src/client/entry-server.tsx` — server-side render function for @fastify/vite
- [ ] T024 Create root layout component in `packages/frontend/src/client/root.tsx` — HTML shell with `<html lang dir>`, meta tags, Tailwind stylesheet
- [ ] T025 Create TanStack Router configuration in `packages/frontend/src/client/router.ts` — router instance with file-based route tree

### Shared Client Libraries

- [ ] T026 Create fetch-based API client factory in `packages/frontend/src/client/lib/apiClient.ts` — `createApiClient(baseUrl)` injecting Authorization, X-Space-Id, X-Organization-Id, X-Request-Id headers; response envelope unwrapping; error handling per api-client.md (401 → refresh, 403 → access denied, etc.)
- [ ] T027 Create React Query client config in `packages/frontend/src/client/lib/queryClient.ts` — QueryClient with stale times, retry policy, error handler
- [ ] T028 Create hierarchical query key factory in `packages/frontend/src/client/lib/queryKeys.ts` — keys for profiles, spaces, companies, members, invitations, workflows, documents, people, payrolls, insurance, tax, services, countries, jurisdictions, businessAreas, conversations, notifications, checkouts

### Context Providers

- [ ] T029 Create AuthContext in `packages/frontend/src/client/context/authContext.tsx` — provides auth state (user, isLoggedIn, isLoading), login/logout actions, session fetch from /auth/session, auto-refresh on token expiry
- [ ] T030 Create CompanyContext in `packages/frontend/src/client/context/companyContext.tsx` — provides current spaceId, companyId, role, permissions, org/company switch functions, RBAC helper `hasPermission(permission: string): boolean`

### Root Route & Providers

- [ ] T031 Create root route in `packages/frontend/src/client/routes/__root.tsx` — wraps app with QueryClientProvider, AuthContext, CompanyContext, Paraglide locale provider; includes Outlet for child routes

### i18n Setup

- [ ] T032 [P] Create Paraglide.js config in `packages/frontend/src/client/i18n/index.ts` — supported locales (en, ar), default locale (en), locale detection (cookie → Accept-Language → default)
- [ ] T033 [P] Create English translations in `packages/frontend/src/client/i18n/locales/en.ts` — common UI strings (nav, buttons, errors, form labels)
- [ ] T034 [P] Create Arabic translations in `packages/frontend/src/client/i18n/locales/ar.ts` — matching keys from en.ts with Arabic translations

### Hooks

- [ ] T035 [P] Create useAuth hook in `packages/frontend/src/client/hooks/useAuth.ts` — thin wrapper consuming AuthContext
- [ ] T036 [P] Create useMobile hook in `packages/frontend/src/client/hooks/useMobile.ts` — responsive breakpoint detection (mobile < 768px)

### API Response Types

- [ ] T037 Create shared API response types in `packages/frontend/src/client/access-layer/types/index.ts` — ApiResponse<T>, ApiError, PaginatedResponse<T> envelopes per api-client.md
- [ ] T038 [P] Create User, Space, Company, CompanyMember, CompanyMemberPermission types in `packages/frontend/src/client/access-layer/types/entities.ts` per data-model.md
- [ ] T039 [P] Create Conversation, Message, CompanyWorkflow, WorkflowStep types in `packages/frontend/src/client/access-layer/types/workflow.ts` per data-model.md
- [ ] T040 [P] Create CompanyCheckout, CheckoutItem, Notification types in `packages/frontend/src/client/access-layer/types/checkout.ts` per data-model.md
- [ ] T041 [P] Create Country, Jurisdiction, BusinessArea, Service, ServicePrice, SubscriptionPlan reference types in `packages/frontend/src/client/access-layer/types/reference.ts` per data-model.md
- [ ] T042 [P] Create CompanyInvitation type in `packages/frontend/src/client/access-layer/types/invitation.ts` per data-model.md

**Checkpoint**: Foundation ready — user story implementation can now begin in parallel

---

## Phase 3: User Story 1 — Authenticated User Accesses Dashboard (Priority: P1) 🎯 MVP

**Goal**: User logs in via Auth0, selects organization/company, and views the main dashboard with status, workflows, notifications, and service navigation cards.

**Independent Test**: Log in with Auth0 credentials → verify dashboard loads with correct org/company context → confirm service navigation cards visible and functional → test org/company switcher → test session expiry redirect.

### API Access Layer for US1

- [ ] T043 [P] [US1] Create profile getters in `packages/frontend/src/client/access-layer/getters/profiles.ts` — `useProfile()` calling GET /profiles/me/companies, `useUserProfile(userId)` calling GET /profiles/:userId
- [ ] T044 [P] [US1] Create space getters in `packages/frontend/src/client/access-layer/getters/spaces.ts` — `useSpace(spaceId)` calling GET /spaces/:spaceId
- [ ] T045 [P] [US1] Create company getters in `packages/frontend/src/client/access-layer/getters/companies.ts` — `useCompanies(spaceId)` calling GET /spaces/:spaceId/companies, `useCompany(spaceId, companyId)` calling GET /spaces/:spaceId/companies/:companyId
- [ ] T046 [P] [US1] Create workflow getters in `packages/frontend/src/client/access-layer/getters/workflows.ts` — `useWorkflows(spaceId, companyId)` calling GET /spaces/:spaceId/companies/:companyId/workflows, `useIncorporationStatus(spaceId, companyId)` calling GET .../workflows/incorporation-status
- [ ] T047 [P] [US1] Create notification getters in `packages/frontend/src/client/access-layer/getters/notifications.ts` — placeholder for notifications (polling every 30s)
- [ ] T048 [P] [US1] Create profile mutations in `packages/frontend/src/client/access-layer/mutations/profiles.ts` — `useUpdateProfile(userId)` calling PATCH /profiles/:userId

### Atom Components for US1

- [ ] T049 [P] [US1] Create Button atom in `packages/frontend/src/client/components/atoms/button.tsx` — Radix UI + Tailwind, variants (primary, secondary, destructive, outline, ghost), sizes, loading state
- [ ] T050 [P] [US1] Create Card atom in `packages/frontend/src/client/components/atoms/card.tsx` — container with header, content, footer slots
- [ ] T051 [P] [US1] Create Badge atom in `packages/frontend/src/client/components/atoms/badge.tsx` — status indicator with color variants
- [ ] T052 [P] [US1] Create Avatar atom in `packages/frontend/src/client/components/atoms/avatar.tsx` — user avatar with fallback initials
- [ ] T053 [P] [US1] Create Skeleton atom in `packages/frontend/src/client/components/atoms/skeleton.tsx` — loading placeholder
- [ ] T054 [P] [US1] Create DropdownMenu atom in `packages/frontend/src/client/components/atoms/dropdown-menu.tsx` — Radix UI dropdown for org/company switcher
- [ ] T055 [P] [US1] Create Separator atom in `packages/frontend/src/client/components/atoms/separator.tsx` — horizontal/vertical divider
- [ ] T056 [P] [US1] Create ScrollArea atom in `packages/frontend/src/client/components/atoms/scroll-area.tsx` — Radix UI scroll container
- [ ] T057 [P] [US1] Create Tooltip atom in `packages/frontend/src/client/components/atoms/tooltip.tsx` — Radix UI tooltip
- [ ] T058 [P] [US1] Create Sheet atom in `packages/frontend/src/client/components/atoms/sheet.tsx` — Radix UI sheet for mobile sidebar

### Molecule Components for US1

- [ ] T059 [US1] Create SidebarNav molecule in `packages/frontend/src/client/components/molecules/sidebar-nav.tsx` — main navigation with service links (dashboard, payroll, tax, banking, insurance, visa, compliance, documents, people, members, workflows, settings), org/company switcher, user menu; uses Sheet on mobile
- [ ] T060 [US1] Create TopBar molecule in `packages/frontend/src/client/components/molecules/top-bar.tsx` — top navigation with breadcrumbs, notification bell, locale switcher, user dropdown
- [ ] T061 [P] [US1] Create ServiceCard molecule in `packages/frontend/src/client/components/molecules/service-card.tsx` — dashboard quick-access card for each service (icon, title, description, link)
- [ ] T062 [P] [US1] Create WorkflowStatusCard molecule in `packages/frontend/src/client/components/molecules/workflow-status-card.tsx` — active workflow progress display
- [ ] T063 [P] [US1] Create NotificationBell molecule in `packages/frontend/src/client/components/molecules/notification-bell.tsx` — unread count badge, dropdown with recent notifications
- [ ] T064 [P] [US1] Create OrgCompanySwitcher molecule in `packages/frontend/src/client/components/molecules/org-company-switcher.tsx` — dropdown for switching active organization and company, updates CompanyContext

### Protected Route Layout

- [ ] T065 [US1] Create protected layout route in `packages/frontend/src/client/routes/organization/index.tsx` — organization selector page; redirects to default space if only one; shows list of spaces if multiple
- [ ] T066 [US1] Create company layout wrapper in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/` — layout with SidebarNav + TopBar, loads company context, enforces auth guard

### Dashboard Page

- [ ] T067 [US1] Create dashboard page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/dashboard.tsx` — service navigation cards grid, active workflows list, recent notifications, company status summary; uses useCompany, useWorkflows, useNotifications getters

**Checkpoint**: User Story 1 complete — user can log in, see org selector, pick company, view dashboard with all service cards

---

## Phase 4: User Story 2 — User Navigates Company Management Services (Priority: P1)

**Goal**: User navigates between service sections (payroll, tax, banking, insurance, visas, compliance) with role-based access and data tables.

**Independent Test**: Navigate to each service section → verify data loads → confirm role-based restrictions (ADMIN sees CRUD, VIEWER sees read-only, unauthorized sees access denied).

### API Access Layer for US2

- [ ] T068 [P] [US2] Create payroll getters/mutations in `packages/frontend/src/client/access-layer/getters/payrolls.ts` and `packages/frontend/src/client/access-layer/mutations/payrolls.ts` — CRUD for GET/POST/PATCH/DELETE /spaces/:spaceId/companies/:companyId/payrolls
- [ ] T069 [P] [US2] Create insurance getters/mutations in `packages/frontend/src/client/access-layer/getters/insurance.ts` and `packages/frontend/src/client/access-layer/mutations/insurance.ts` — CRUD for insurance-policies endpoints
- [ ] T070 [P] [US2] Create tax getters/mutations in `packages/frontend/src/client/access-layer/getters/tax.ts` and `packages/frontend/src/client/access-layer/mutations/tax.ts` — CRUD for tax-records endpoints
- [ ] T071 [P] [US2] Create people getters/mutations in `packages/frontend/src/client/access-layer/getters/people.ts` and `packages/frontend/src/client/access-layer/mutations/people.ts` — CRUD for company people endpoints
- [ ] T072 [P] [US2] Create document getters/mutations in `packages/frontend/src/client/access-layer/getters/documents.ts` and `packages/frontend/src/client/access-layer/mutations/documents.ts` — CRUD + upload-url, confirm-upload, download-url for documents endpoints
- [ ] T073 [P] [US2] Create reference data getters in `packages/frontend/src/client/access-layer/getters/reference.ts` — useCountries, useJurisdictions, useBusinessAreas, useServices, useServicePrices (long cache, 1 hour stale time)

### Shared Table & Form Atoms for US2

- [ ] T074 [P] [US2] Create Table atom in `packages/frontend/src/client/components/atoms/table.tsx` — data table with Radix UI styling, sortable columns
- [ ] T075 [P] [US2] Create Input atom in `packages/frontend/src/client/components/atoms/input.tsx` — text input with label, error state, RTL support
- [ ] T076 [P] [US2] Create Select atom in `packages/frontend/src/client/components/atoms/select.tsx` — Radix UI select dropdown
- [ ] T077 [P] [US2] Create Dialog atom in `packages/frontend/src/client/components/atoms/dialog.tsx` — Radix UI modal dialog
- [ ] T078 [P] [US2] Create Label atom in `packages/frontend/src/client/components/atoms/label.tsx` — form label
- [ ] T079 [P] [US2] Create Textarea atom in `packages/frontend/src/client/components/atoms/textarea.tsx` — multiline text input
- [ ] T080 [P] [US2] Create Tabs atom in `packages/frontend/src/client/components/atoms/tabs.tsx` — Radix UI tab navigation
- [ ] T081 [P] [US2] Create Pagination molecule in `packages/frontend/src/client/components/molecules/pagination.tsx` — page navigation with offset/limit, total count

### Shared Molecules for US2

- [ ] T082 [P] [US2] Create DataTable molecule in `packages/frontend/src/client/components/molecules/data-table.tsx` — reusable table with sorting, pagination, empty states, loading skeletons; wraps Table atom + Pagination
- [ ] T083 [P] [US2] Create FormField molecule in `packages/frontend/src/client/components/molecules/form-field.tsx` — React Hook Form integration with Zod resolver, error display
- [ ] T084 [P] [US2] Create PermissionGate molecule in `packages/frontend/src/client/components/molecules/permission-gate.tsx` — renders children only if `hasPermission(required)` from CompanyContext; shows access denied otherwise
- [ ] T085 [P] [US2] Create EmptyState molecule in `packages/frontend/src/client/components/molecules/empty-state.tsx` — placeholder for empty lists with icon, message, CTA
- [ ] T086 [P] [US2] Create ErrorBoundary molecule in `packages/frontend/src/client/components/molecules/error-boundary.tsx` — graceful error display with retry button per FR-017

### Service Pages

- [ ] T087 [US2] Create payroll page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/payroll.tsx` — DataTable of payroll records, create/edit dialog with React Hook Form + Zod, delete confirmation; guarded by view:company
- [ ] T088 [P] [US2] Create tax page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/tax.tsx` — DataTable of tax records, CRUD operations; guarded by view:company
- [ ] T089 [P] [US2] Create banking page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/banking.tsx` — banking overview with account details; guarded by view:company
- [ ] T090 [P] [US2] Create insurance page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/insurance.tsx` — DataTable of insurance policies, CRUD operations; guarded by view:company
- [ ] T091 [P] [US2] Create visa management page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/visa-management.tsx` — visa applications with status tracking, document uploads, appointment scheduling; guarded by view:company
- [ ] T092 [P] [US2] Create compliance page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/compliance.tsx` — compliance status overview; guarded by view:company
- [ ] T093 [P] [US2] Create documents page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/documents.tsx` — DataTable of documents, upload via presigned URL, download; guarded by view:documents
- [ ] T094 [P] [US2] Create people page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/people.tsx` — DataTable of directors/shareholders, CRUD; guarded by view:company
- [ ] T095 [US2] Create workflows list page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/workflows/index.tsx` — DataTable of workflows with status; guarded by view:company
- [ ] T096 [US2] Create workflow detail page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/workflows/$workflowId.tsx` — workflow steps timeline, step start/submit/complete actions; guarded by view:company
- [ ] T097 [P] [US2] Create accounting page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/accounting.tsx` — accounting overview; guarded by view:company
- [ ] T098 [P] [US2] Create company settings page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/settings.tsx` — company profile edit form; guarded by update:company

**Checkpoint**: User Story 2 complete — all service sections navigable with data and RBAC enforcement

---

## Phase 5: User Story 3 — User Interacts with AI Chatbot (Priority: P2)

**Goal**: User opens the AI chatbot from the dashboard, sends messages, and receives contextual streaming responses. Anonymous users can also interact via the pre-incorporation landing chat.

**Independent Test**: Open chatbot → send message → verify streaming response appears incrementally → test anonymous pre-incorporation flow → test action confirmations.

### API Access Layer for US3

- [ ] T099 [P] [US3] Create conversation getters in `packages/frontend/src/client/access-layer/getters/conversations.ts` — `useConversation(id)` calling GET /conversations/:id, `useMessages(id)` calling GET /conversations/:id/messages
- [ ] T100 [P] [US3] Create conversation mutations in `packages/frontend/src/client/access-layer/mutations/conversations.ts` — `useCreateConversation()` calling POST /conversations, `useSendMessage(id)` calling POST /conversations/:id/messages, `useStreamMessage(id)` handling SSE via POST /conversations/:id/messages?stream=true

### Chatbot Components

- [ ] T101 [P] [US3] Create ChatMessage atom in `packages/frontend/src/client/components/atoms/chat-message.tsx` — message bubble with user/assistant styling, markdown rendering, tool call display
- [ ] T102 [P] [US3] Create ChatInput atom in `packages/frontend/src/client/components/atoms/chat-input.tsx` — text input with send button, disabled during streaming
- [ ] T103 [US3] Create ChatPanel molecule in `packages/frontend/src/client/components/molecules/chat-panel.tsx` — full chatbot panel with message list (ScrollArea), ChatInput, streaming indicator, conversation header; manages SSE connection for streaming responses
- [ ] T104 [US3] Create ChatFAB molecule in `packages/frontend/src/client/components/molecules/chat-fab.tsx` — floating action button on dashboard to open/close ChatPanel; includes unread indicator

### Chatbot Integration in Dashboard

- [ ] T105 [US3] Integrate ChatFAB + ChatPanel into the company layout in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/` — chatbot available on all protected pages with current company context passed to conversation

### Pre-Incorporation Chat (Public)

- [ ] T106 [US3] Create pre-incorporation chat page in `packages/frontend/src/client/routes/_locale/pre-incorporation/index.tsx` — anonymous chatbot for incorporation guidance; creates conversation of type ANONYMOUS_LANDING
- [ ] T107 [US3] Create pre-incorporation conversation page in `packages/frontend/src/client/routes/_locale/pre-incorporation/$conversationId.tsx` — continues existing anonymous conversation; shows upgrade prompt to sign up

**Checkpoint**: User Story 3 complete — authenticated and anonymous chatbot interactions work with streaming

---

## Phase 6: User Story 4 — User Completes Stripe Checkout (Priority: P2)

**Goal**: User selects a service, reviews order with currency selection, completes Stripe payment, and returns with service activated.

**Independent Test**: Initiate checkout → verify order summary → switch currency → complete Stripe test payment → confirm service activation on return.

### API Access Layer for US4

- [ ] T108 [P] [US4] Create checkout getters in `packages/frontend/src/client/access-layer/getters/checkouts.ts` — `useCheckouts(spaceId)` calling GET /spaces/:spaceId/checkouts, `useCheckout(spaceId, checkoutId)` calling GET /spaces/:spaceId/checkouts/:checkoutId, `useCheckoutItems(spaceId, checkoutId)` calling GET .../items
- [ ] T109 [P] [US4] Create checkout mutations in `packages/frontend/src/client/access-layer/mutations/checkouts.ts` — `useCreateCheckoutSession(spaceId)` calling POST /spaces/:spaceId/checkouts/sessions, `useCreateSkuSession(spaceId)` calling POST .../sessions/sku
- [ ] T110 [P] [US4] Create conversation checkout getters/mutations in `packages/frontend/src/client/access-layer/getters/conversation-checkout.ts` and `packages/frontend/src/client/access-layer/mutations/conversation-checkout.ts` — GET/PATCH/POST for /conversations/:id/checkout endpoints

### Checkout Components

- [ ] T111 [P] [US4] Create CurrencySelector atom in `packages/frontend/src/client/components/atoms/currency-selector.tsx` — dropdown for currency selection (USD, AED, etc.)
- [ ] T112 [P] [US4] Create PriceDisplay atom in `packages/frontend/src/client/components/atoms/price-display.tsx` — formatted price with currency symbol
- [ ] T113 [US4] Create CheckoutSummary molecule in `packages/frontend/src/client/components/molecules/checkout-summary.tsx` — order items table, currency switcher, total, proceed-to-pay button
- [ ] T114 [US4] Create CheckoutPage page component in `packages/frontend/src/client/components/pages/checkout-page.tsx` — full checkout flow: order review → currency selection → Stripe redirect → return handling (success/cancel)

### Billing Page

- [ ] T115 [US4] Create billing page in `packages/frontend/src/client/routes/organization/$spaceId/billing.tsx` — subscription plan display, checkout history DataTable, upgrade/downgrade actions; guarded by manage:billing

**Checkpoint**: User Story 4 complete — end-to-end checkout flow functional with Stripe

---

## Phase 7: User Story 5 — New User Completes Onboarding Profile (Priority: P3)

**Goal**: New user sees profile completion form after first login, fills personal details, or skips for later. Incomplete profile shows reminder on subsequent visits.

**Independent Test**: Sign up new user → verify profile form appears → complete all fields and submit → verify data saved → test skip flow → verify reminder on return.

### Onboarding Components

- [ ] T116 [P] [US5] Create DatePicker atom in `packages/frontend/src/client/components/atoms/date-picker.tsx` — date input for birth date
- [ ] T117 [P] [US5] Create PhoneInput atom in `packages/frontend/src/client/components/atoms/phone-input.tsx` — phone number input with country prefix selector
- [ ] T118 [P] [US5] Create CountrySelector atom in `packages/frontend/src/client/components/atoms/country-selector.tsx` — searchable country dropdown using reference data

### Onboarding Pages

- [ ] T119 [US5] Create profile completion page in `packages/frontend/src/client/routes/account/profile-completion.tsx` — multi-field form (name, phone prefix + number, birth date, country, preferred language, profile image upload) with React Hook Form + Zod validation; "Complete" and "Skip for now" buttons; redirects to /organization on completion
- [ ] T120 [P] [US5] Create email verification page in `packages/frontend/src/client/routes/account/email-verification.tsx` — email verification status and resend
- [ ] T121 [P] [US5] Create user settings page in `packages/frontend/src/client/routes/account/settings.tsx` — edit profile, change preferences, notification settings

### Profile Reminder

- [ ] T122 [US5] Create ProfileReminder molecule in `packages/frontend/src/client/components/molecules/profile-reminder.tsx` — subtle banner shown on dashboard when profile is incomplete; dismissable but reappears on next visit

**Checkpoint**: User Story 5 complete — onboarding flow and profile management functional

---

## Phase 8: User Story 6 — User Manages Organization Members and Roles (Priority: P3)

**Goal**: Organization owner/admin invites members, assigns roles, manages permissions across companies.

**Independent Test**: Navigate to members → invite new member via email → assign role → log in as invited member → verify permissions match role.

### API Access Layer for US6

- [ ] T123 [P] [US6] Create member getters in `packages/frontend/src/client/access-layer/getters/members.ts` — `useMembers(spaceId, companyId)` calling GET /spaces/:spaceId/companies/:companyId/members
- [ ] T124 [P] [US6] Create member mutations in `packages/frontend/src/client/access-layer/mutations/members.ts` — `useUpdateMemberRole(spaceId, companyId, memberId)` calling PATCH .../role, `useRemoveMember(spaceId, companyId, memberId)` calling DELETE
- [ ] T125 [P] [US6] Create invitation getters in `packages/frontend/src/client/access-layer/getters/invitations.ts` — `useInvitations(spaceId, companyId)` calling GET .../invitations
- [ ] T126 [P] [US6] Create invitation mutations in `packages/frontend/src/client/access-layer/mutations/invitations.ts` — `useCreateInvitation`, `useResendInvitation`, `useCancelInvitation`, `useAcceptInvitation`

### Member Management Components

- [ ] T127 [P] [US6] Create RoleSelect atom in `packages/frontend/src/client/components/atoms/role-select.tsx` — role picker (SUPER_ADMINISTRATOR, OWNER, ADMIN, MANAGER, VIEWER) with descriptions
- [ ] T128 [US6] Create MembersList molecule in `packages/frontend/src/client/components/molecules/members-list.tsx` — DataTable of members with role badges, status, actions (edit role, remove); respects permissions
- [ ] T129 [US6] Create InviteForm molecule in `packages/frontend/src/client/components/molecules/invite-form.tsx` — email + role selection dialog for inviting new members
- [ ] T130 [US6] Create InvitationsList molecule in `packages/frontend/src/client/components/molecules/invitations-list.tsx` — DataTable of pending invitations with resend/cancel actions

### Member Pages

- [ ] T131 [US6] Create members page in `packages/frontend/src/client/routes/organization/$spaceId/company/$companyId/members.tsx` — MembersList + InviteForm + InvitationsList; guarded by view:users, invite requires manage:invitations
- [ ] T132 [US6] Create organization settings page in `packages/frontend/src/client/routes/organization/$spaceId/settings.tsx` — org profile, settings; guarded by manage:organizations

**Checkpoint**: User Story 6 complete — member invitation and role management functional

---

## Phase 9: User Story 7 — Public Visitor Browses Landing and Service Pages (Priority: P3)

**Goal**: Public visitors browse SSR-rendered landing, service, pricing, blog, and contact pages with locale switching and RTL support.

**Independent Test**: Visit each public page → switch locale to Arabic → verify RTL layout → check SEO metadata → submit contact form.

### Public Page Atoms

- [ ] T133 [P] [US7] Create LocaleSwitcher atom in `packages/frontend/src/client/components/atoms/locale-switcher.tsx` — language toggle (EN/AR) updating Paraglide locale and URL prefix
- [ ] T134 [P] [US7] Create CTAButton atom in `packages/frontend/src/client/components/atoms/cta-button.tsx` — call-to-action button linking to signup/login
- [ ] T135 [P] [US7] Create Footer molecule in `packages/frontend/src/client/components/molecules/footer.tsx` — site footer with links, locale switcher, copyright
- [ ] T136 [P] [US7] Create PublicNav molecule in `packages/frontend/src/client/components/molecules/public-nav.tsx` — public page navigation with services dropdown, pricing, blog, contact, login/signup buttons

### Locale Layout

- [ ] T137 [US7] Create locale layout route in `packages/frontend/src/client/routes/_locale/` — wraps public pages with PublicNav + Footer, sets HTML dir attribute based on locale

### Public Pages

- [ ] T138 [US7] Create landing page in `packages/frontend/src/client/routes/_locale/index.tsx` — hero section, service highlights, testimonials, CTA; SSR rendered
- [ ] T139 [P] [US7] Create services overview page in `packages/frontend/src/client/routes/_locale/services/index.tsx` — grid of service cards with descriptions; SSR
- [ ] T140 [P] [US7] Create incorporation page in `packages/frontend/src/client/routes/_locale/services/incorporation.tsx` — detailed incorporation service info; SSR
- [ ] T141 [P] [US7] Create accounting page in `packages/frontend/src/client/routes/_locale/services/accounting.tsx` — accounting service details; SSR
- [ ] T142 [P] [US7] Create payroll page in `packages/frontend/src/client/routes/_locale/services/payroll.tsx` — payroll service details; SSR
- [ ] T143 [P] [US7] Create insurance page in `packages/frontend/src/client/routes/_locale/services/insurance.tsx` — insurance service details; SSR
- [ ] T144 [P] [US7] Create legal page in `packages/frontend/src/client/routes/_locale/services/legal.tsx` — legal service details; SSR
- [ ] T145 [P] [US7] Create residency page in `packages/frontend/src/client/routes/_locale/services/residency.tsx` — residency service details; SSR
- [ ] T146 [P] [US7] Create banking page in `packages/frontend/src/client/routes/_locale/services/banking.tsx` — banking service details; SSR
- [ ] T147 [US7] Create pricing page in `packages/frontend/src/client/routes/_locale/pricing.tsx` — tier comparison table with CTA buttons; SSR
- [ ] T148 [P] [US7] Create blog listing page in `packages/frontend/src/client/routes/_locale/blog/index.tsx` — blog post cards with pagination; SSR
- [ ] T149 [P] [US7] Create blog post page in `packages/frontend/src/client/routes/_locale/blog/$slug.tsx` — individual blog post with markdown rendering; SSR
- [ ] T150 [US7] Create contact-us page in `packages/frontend/src/client/routes/_locale/contact-us.tsx` — contact form with Zod validation, submit handler; SSR
- [ ] T151 [P] [US7] Create privacy policy page in `packages/frontend/src/client/routes/_locale/privacy-policy.tsx` — static content; SSR
- [ ] T152 [P] [US7] Create terms and conditions page in `packages/frontend/src/client/routes/_locale/terms-and-conditions.tsx` — static content; SSR
- [ ] T153 [P] [US7] Create cookie policy page in `packages/frontend/src/client/routes/_locale/cookie-policy.tsx` — static content; SSR

### SEO Redirects

- [ ] T154 [US7] Implement SEO redirect routes in `packages/frontend/src/routes/redirects.ts` — 301 redirects for old URL patterns per route-map.md (preserve SEO value)

**Checkpoint**: User Story 7 complete — all public pages SSR rendered with i18n and SEO

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T155 [P] Create company creation page in `packages/frontend/src/client/routes/organization/$spaceId/create-company.tsx` — multi-step form for creating a new company (name, jurisdiction, business area); guarded by manage:companies
- [ ] T156 [P] Add error boundary wrapping in `packages/frontend/src/client/routes/__root.tsx` — global ErrorBoundary with graceful degradation per FR-017
- [ ] T157 [P] Add service unavailability handling in `packages/frontend/src/client/lib/apiClient.ts` — retry logic with exponential backoff for 5xx errors, user-friendly offline/error states
- [ ] T158 [P] Implement file upload utility in `packages/frontend/src/client/lib/fileUpload.ts` — presigned URL upload flow (request URL → upload to blob storage → confirm upload) for documents, profile images, KYC materials per FR-013
- [ ] T159 [P] Add SEO meta tags utility in `packages/frontend/src/client/lib/seo.ts` — helper for setting page title, description, og:tags in SSR-rendered pages
- [ ] T160 [P] Create Dockerfile in `packages/frontend/Dockerfile` — multi-stage build with `oven/bun` base image, vite build step, fastify production server
- [ ] T161 Run quickstart.md validation — verify all setup steps work, dev server starts, auth flow completes, dashboard loads

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — BLOCKS all user stories
- **User Stories (Phase 3–9)**: All depend on Phase 2 completion
  - US1 (Phase 3) and US7 (Phase 9) have no inter-story dependencies — can start in parallel
  - US2 (Phase 4) benefits from US1 layout components but can start independently
  - US3 (Phase 5) is independent of other stories
  - US4 (Phase 6) is independent of other stories
  - US5 (Phase 7) is independent of other stories
  - US6 (Phase 8) is independent of other stories
- **Polish (Phase 10)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: After Phase 2 — creates layout + dashboard (no dependencies on other stories)
- **US2 (P1)**: After Phase 2 — reuses layout from US1 but can create its own if needed
- **US3 (P2)**: After Phase 2 — chatbot is standalone; integrates into US1 layout
- **US4 (P2)**: After Phase 2 — checkout is standalone; billing page is standalone
- **US5 (P3)**: After Phase 2 — onboarding is standalone
- **US6 (P3)**: After Phase 2 — members page is standalone
- **US7 (P3)**: After Phase 2 — public pages are fully independent (SSR, no auth)

### Within Each User Story

- API types/getters/mutations before components that use them
- Atom components before molecule components
- Molecule components before page components
- Layout routes before page routes

### Parallel Opportunities

- All Setup tasks T003–T007 can run in parallel
- All Foundational type definitions T038–T042 can run in parallel
- All Foundational i18n tasks T032–T034 can run in parallel
- All Foundational hooks T035–T036 can run in parallel
- Within US1: all atom components T049–T058 can run in parallel
- Within US2: all service pages T088–T098 can run in parallel
- Within US7: all service detail pages T140–T146 and static pages T151–T153 can run in parallel
- US1 and US7 can be developed simultaneously by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all US1 getters in parallel:
Task: "Create profile getters in packages/frontend/src/client/access-layer/getters/profiles.ts"
Task: "Create space getters in packages/frontend/src/client/access-layer/getters/spaces.ts"
Task: "Create company getters in packages/frontend/src/client/access-layer/getters/companies.ts"
Task: "Create workflow getters in packages/frontend/src/client/access-layer/getters/workflows.ts"
Task: "Create notification getters in packages/frontend/src/client/access-layer/getters/notifications.ts"

# Launch all US1 atom components in parallel:
Task: "Create Button atom in packages/frontend/src/client/components/atoms/button.tsx"
Task: "Create Card atom in packages/frontend/src/client/components/atoms/card.tsx"
Task: "Create Badge atom in packages/frontend/src/client/components/atoms/badge.tsx"
Task: "Create Avatar atom in packages/frontend/src/client/components/atoms/avatar.tsx"
Task: "Create Skeleton atom in packages/frontend/src/client/components/atoms/skeleton.tsx"
Task: "Create DropdownMenu atom in packages/frontend/src/client/components/atoms/dropdown-menu.tsx"
```

## Parallel Example: User Story 7

```bash
# Launch all US7 service pages in parallel:
Task: "Create incorporation page in packages/frontend/src/client/routes/_locale/services/incorporation.tsx"
Task: "Create accounting page in packages/frontend/src/client/routes/_locale/services/accounting.tsx"
Task: "Create payroll page in packages/frontend/src/client/routes/_locale/services/payroll.tsx"
Task: "Create insurance page in packages/frontend/src/client/routes/_locale/services/insurance.tsx"
Task: "Create legal page in packages/frontend/src/client/routes/_locale/services/legal.tsx"
Task: "Create residency page in packages/frontend/src/client/routes/_locale/services/residency.tsx"
Task: "Create banking page in packages/frontend/src/client/routes/_locale/services/banking.tsx"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1 (Dashboard)
4. **STOP and VALIDATE**: Log in → org selector → company dashboard → all service cards visible
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. Add US1 (Dashboard) → Test → Deploy/Demo (**MVP!**)
3. Add US2 (Service Pages) → Test → Deploy/Demo
4. Add US3 (Chatbot) → Test → Deploy/Demo
5. Add US4 (Checkout) → Test → Deploy/Demo
6. Add US5 (Onboarding) → Test → Deploy/Demo
7. Add US6 (Members) → Test → Deploy/Demo
8. Add US7 (Public Pages) → Test → Deploy/Demo
9. Polish → Final validation → Production release

### Parallel Team Strategy

With multiple developers after Phase 2:

- **Developer A**: US1 (Dashboard) → US2 (Service Pages)
- **Developer B**: US7 (Public Pages) → US3 (Chatbot)
- **Developer C**: US4 (Checkout) → US5 (Onboarding) → US6 (Members)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- All paths are relative to repository root
- Tests not included as not explicitly requested — add with /speckit.tasks if TDD desired
