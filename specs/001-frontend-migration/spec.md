# Feature Specification: Frontend Migration to Monorepo

**Feature Branch**: `001-frontend-migration`
**Created**: 2026-03-02
**Status**: Draft
**Input**: User description: "Migrate the azure-backup-frontend (Next.js 15/React 19) to a new frontend package within the backend-agents-service Bun monorepo. The new frontend should consume the Fastify-based CRM, agents, checkout, and notification services."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Authenticated User Accesses Dashboard (Priority: P1)

A registered business owner logs in via Auth0, selects their organization and company, and lands on the main dashboard. They can see their company status, active workflows, recent notifications, and quick-access cards for all services (payroll, tax, banking, insurance, visas, compliance).

**Why this priority**: Authentication and the dashboard are the foundation of the entire application. Without secure login and a functional dashboard, no other feature can be accessed or tested.

**Independent Test**: Can be fully tested by logging in with Auth0 credentials, verifying the dashboard loads with correct organization/company context, and confirming all service navigation cards are visible and functional.

**Acceptance Scenarios**:

1. **Given** a registered user with valid Auth0 credentials, **When** they log in, **Then** they are redirected to the dashboard with their default organization pre-selected.
2. **Given** a logged-in user belonging to multiple organizations, **When** they use the organization switcher, **Then** the dashboard updates to reflect the selected organization's data.
3. **Given** a logged-in user with multiple companies under an organization, **When** they use the company switcher, **Then** the dashboard displays data specific to the selected company.
4. **Given** a user whose session has expired, **When** they attempt to access any protected page, **Then** they are redirected to the login page with a return URL preserved.

---

### User Story 2 - User Navigates Company Management Services (Priority: P1)

A company administrator navigates between the core service sections (payroll, tax, banking, insurance, visas, compliance) from the dashboard. Each section displays relevant data tables, forms, and action buttons appropriate to the user's role and permissions.

**Why this priority**: Service navigation is the core value proposition. Users must be able to access and manage their company services to derive any business value from the platform.

**Independent Test**: Can be tested by navigating to each service section, verifying data loads correctly, and confirming role-based access restrictions are enforced.

**Acceptance Scenarios**:

1. **Given** a user with ADMIN role, **When** they navigate to the payroll section, **Then** they see payroll records, can create entries, and manage payroll reports.
2. **Given** a user with VIEWER role, **When** they navigate to any service section, **Then** they can view data but cannot create, edit, or delete records.
3. **Given** a user accessing the visa management section, **When** they view visa applications, **Then** they see status tracking, document uploads, and appointment scheduling.
4. **Given** a user with no access to a specific service, **When** they attempt to navigate to that service, **Then** they see an appropriate access denied message.

---

### User Story 3 - User Interacts with AI Chatbot (Priority: P2)

A user opens the AI-powered chatbot from the dashboard to ask questions about incorporation, get guidance on next steps, or request information about their company. The chatbot uses contextual awareness (current company, active workflows) to provide relevant responses.

**Why this priority**: The AI chatbot is a key differentiator for the platform, providing guided consultation. However, users can still operate the platform without it via direct navigation.

**Independent Test**: Can be tested by opening the chatbot, sending messages, and verifying contextual responses are returned. Streaming responses should appear incrementally.

**Acceptance Scenarios**:

1. **Given** a logged-in user on the dashboard, **When** they open the chatbot, **Then** a conversation panel appears with the ability to type messages.
2. **Given** an active conversation, **When** the user sends a question about incorporation, **Then** the chatbot responds with relevant guidance, potentially using tools to fetch company-specific data.
3. **Given** a user in the pre-incorporation phase (not yet a customer), **When** they interact with the landing chatbot, **Then** they receive general incorporation guidance without requiring login.
4. **Given** a chatbot response that includes an action (e.g., "start incorporation workflow"), **When** the user confirms the action, **Then** the system initiates the appropriate workflow.

---

### User Story 4 - User Completes Stripe Checkout (Priority: P2)

A user selects a service or subscription plan and proceeds through the checkout flow. They review their order, select a currency, and are redirected to Stripe for payment. Upon successful payment, they return to the platform with the service activated.

**Why this priority**: Checkout is essential for revenue generation but depends on the dashboard and service navigation being functional first.

**Independent Test**: Can be tested by initiating a checkout for a service, verifying the order summary, completing payment via Stripe test mode, and confirming the service is activated post-payment.

**Acceptance Scenarios**:

1. **Given** a user has selected a service to purchase, **When** they proceed to checkout, **Then** they see an order summary with itemized pricing.
2. **Given** a user reviewing their checkout, **When** they switch currency, **Then** prices update to reflect the selected currency.
3. **Given** a user who completes Stripe payment, **When** they return to the platform, **Then** the purchased service is active and accessible.
4. **Given** a user who cancels payment on Stripe, **When** they return, **Then** their checkout session is preserved for later completion.

---

### User Story 5 - New User Completes Onboarding Profile (Priority: P3)

After signing up via Auth0, a new user is presented with a profile completion screen where they provide personal details (name, phone, date of birth, nationality, address, preferred language). They can either complete the profile or skip it for later.

**Why this priority**: Onboarding improves data quality and user experience, but users can still access the platform without completing their profile.

**Independent Test**: Can be tested by signing up a new user, verifying the profile form appears, completing all fields, and confirming data is saved correctly.

**Acceptance Scenarios**:

1. **Given** a newly registered user, **When** they first access the platform, **Then** they are shown the profile completion screen.
2. **Given** the profile form, **When** the user fills all required fields and submits, **Then** their profile is saved and they proceed to the dashboard.
3. **Given** the profile form, **When** the user clicks "Skip for now", **Then** they are taken to the dashboard with a reminder to complete their profile later.
4. **Given** an incomplete profile, **When** the user returns to the platform, **Then** they see a subtle prompt to complete their profile but are not blocked.

---

### User Story 6 - User Manages Organization Members and Roles (Priority: P3)

An organization owner or super administrator invites team members, assigns roles (Owner, Admin, Manager, Viewer), and manages permissions across their organization's companies.

**Why this priority**: Multi-user collaboration is important for business operations but depends on the core platform being functional first.

**Independent Test**: Can be tested by inviting a new member via email, assigning them a role, logging in as that member, and verifying their access matches the assigned permissions.

**Acceptance Scenarios**:

1. **Given** an organization owner, **When** they navigate to member management, **Then** they see a list of all members with their roles and status.
2. **Given** an owner inviting a new member, **When** they enter an email and select a role, **Then** an invitation email is sent and the member appears as "Pending" in the list.
3. **Given** a member with MANAGER role, **When** they attempt to invite another member, **Then** the system allows it only if they have the `manage:organizations` permission.
4. **Given** a SUPER_ADMINISTRATOR, **When** they change a member's role, **Then** the member's permissions update immediately across all sessions.

---

### User Story 7 - Public Visitor Browses Landing and Service Pages (Priority: P3)

A visitor (not logged in) browses the public-facing pages: landing page, service descriptions, pricing, blog, and contact form. They can switch languages and view content in their preferred locale.

**Why this priority**: Public pages drive user acquisition but are independent of the authenticated application. They can be implemented as a separate slice.

**Independent Test**: Can be tested by visiting each public page, switching locales, verifying content renders correctly, and confirming SEO metadata is present.

**Acceptance Scenarios**:

1. **Given** a visitor on the landing page, **When** they browse service descriptions, **Then** they see detailed information about incorporation, payroll, tax, and other services.
2. **Given** a visitor, **When** they switch the language to Arabic, **Then** all content updates to Arabic with RTL layout applied.
3. **Given** a visitor on the pricing page, **When** they view plans, **Then** they see tier comparisons with clear calls-to-action to sign up.
4. **Given** a visitor, **When** they submit the contact form, **Then** the form validates inputs and confirms submission.

---

### Edge Cases

- What happens when a user belongs to an organization but has no companies assigned?
- How does the system handle simultaneous sessions across multiple browser tabs with different company contexts?
- What happens when the backend services (CRM, agents, checkout, notification) are temporarily unavailable?
- How does the system behave when Auth0 is unreachable during token refresh?
- What happens when a user's role is changed while they have an active session?
- How does the system handle switching between organizations with different subscription tiers?
- What happens when a chatbot conversation references a company the user no longer has access to?
- How does the checkout flow handle currency conversion when the user's preferred currency is unavailable for a service?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST authenticate users via Auth0 OAuth2 and maintain secure sessions with token refresh capability.
- **FR-002**: System MUST support multi-tenant isolation where each request is scoped to the user's active organization and company.
- **FR-003**: System MUST enforce role-based access control with five roles (Super Administrator, Owner, Admin, Manager, Viewer) and eleven or more granular permissions.
- **FR-004**: System MUST provide a responsive dashboard displaying company status, active workflows, notifications, and service navigation.
- **FR-005**: System MUST support navigation to and interaction with company management services: payroll, tax, banking, insurance, visa management, and compliance.
- **FR-006**: System MUST integrate with the AI chatbot for guided consultation, supporting both anonymous (landing) and authenticated (dashboard) conversation modes.
- **FR-007**: System MUST support streaming responses from the AI chatbot for real-time conversation feedback.
- **FR-008**: System MUST integrate with Stripe checkout for service and subscription purchases, including currency selection and order review.
- **FR-009**: System MUST support internationalization with at minimum English and Arabic locales, including right-to-left layout support.
- **FR-010**: System MUST provide organization and company switching for users belonging to multiple entities.
- **FR-011**: System MUST support member invitation and role management within organizations.
- **FR-012**: System MUST display data tables with pagination, sorting, and filtering for all list views (employees, documents, invoices, etc.).
- **FR-013**: System MUST support file uploads for documents, profile images, and KYC verification materials.
- **FR-014**: System MUST display real-time notifications for workflow updates, payment confirmations, and system alerts.
- **FR-015**: System MUST render public-facing pages (landing, services, pricing, blog, contact) accessible without authentication.
- **FR-016**: System MUST preserve the existing URL structure and routing patterns to maintain SEO value during migration.
- **FR-017**: System MUST gracefully handle backend service unavailability with user-friendly error messages and retry mechanisms.
- **FR-018**: System MUST support the onboarding profile completion flow for new users.

### Key Entities

- **User**: Represents an authenticated individual with profile details, preferences, and role assignments across one or more organizations.
- **Organization (Space)**: A top-level tenant entity that groups companies and members under a single billing and management scope.
- **Company**: A business entity within an organization, representing an incorporated or in-progress business with its own workflows, documents, and services.
- **Conversation**: An AI chatbot session tied to a user and optionally a company, containing a sequence of messages and tool interactions.
- **Workflow**: A multi-step business process (e.g., incorporation, visa application) with status tracking, task dependencies, and document requirements.
- **Checkout Session**: A transactional entity representing a pending or completed purchase, linked to Stripe for payment processing.
- **Notification**: A system-generated alert or update delivered to users about workflow progress, payments, or administrative actions.
- **Member Invitation**: A pending or accepted invitation for a user to join an organization with a specified role.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can log in and reach the dashboard within 5 seconds of entering valid credentials.
- **SC-002**: All existing functionality from the previous frontend is accessible in the new frontend with no feature regression.
- **SC-003**: Users can switch between organizations and companies in under 2 seconds with the dashboard updating accordingly.
- **SC-004**: The AI chatbot displays streaming responses with the first token appearing within 1 second of sending a message.
- **SC-005**: 95% of users can complete a Stripe checkout flow (from service selection to payment confirmation) in under 3 minutes.
- **SC-006**: Public-facing pages achieve equivalent or better search engine indexing scores compared to the previous frontend.
- **SC-007**: The application supports at least 500 concurrent authenticated users without degradation in response times.
- **SC-008**: Language switching between English and Arabic (including RTL layout) completes in under 1 second with no visual glitches.
- **SC-009**: Role-based access restrictions are enforced consistently, with 100% of restricted actions blocked for unauthorized users.
- **SC-010**: The new frontend integrates as a package within the existing monorepo without breaking existing service builds or tests.

## Assumptions

- Auth0 will remain the identity provider; no migration to a different auth system is planned.
- The existing Radix UI + Tailwind CSS component design language will be preserved in the new frontend.
- The Fastify backend services (CRM, agents, checkout, notification) already expose all necessary API endpoints or will be extended as needed during frontend implementation.
- The monorepo's Bun workspace configuration will be extended to include the new frontend package alongside existing services.
- The port conflict (agents service and frontend both using port 3000) will be resolved by assigning a dedicated port to the frontend service.
- Existing blog content stored in MongoDB will continue to be accessible via the same or a compatible API.
- SEO-critical pages (landing, services, pricing, blog) require server-side rendering or static generation for search engine crawlability.
- The migration will be incremental, allowing both the old and new frontends to coexist during the transition period.

## Dependencies

- Auth0 tenant configuration and credentials must be available for the new frontend.
- Stripe API keys and webhook configuration must be set up for the checkout flow.
- The CRM service must expose all CRUD endpoints used by the current frontend's access layer.
- The agents service must support the conversation API with streaming for chatbot integration.
- The notification service must provide endpoints for fetching and managing user notifications.
- Internationalization content (translation files) must be migrated from the existing frontend.
- The monorepo CI/CD pipeline must be updated to include frontend build, test, and deployment steps.

## Scope Boundaries

### In Scope

- Full migration of all authenticated dashboard features (company management, workflows, services).
- Migration of public-facing pages (landing, services, pricing, blog, contact).
- Auth0 integration with session management and token refresh.
- AI chatbot integration (anonymous and authenticated modes).
- Stripe checkout flow.
- Internationalization (English and Arabic with RTL support).
- Role-based access control enforcement.
- Organization and company context switching.
- Member invitation and management.
- File upload capabilities.
- Responsive design for desktop and mobile viewports.

### Out of Scope

- Backend API changes (services are consumed as-is; if new endpoints are needed, they are tracked separately).
- Migration of the admin blog management interface (low priority, can be deferred).
- Implementation of future features documented in `docs/future/` (analytics dashboard, billing management, medical appointments, etc.) -- these are tracked separately but the frontend architecture must support their future addition.
- Changes to the Auth0 tenant configuration or user migration.
- Mobile native application development.
- Automated end-to-end testing of the complete migration (manual verification is acceptable for the initial release).
