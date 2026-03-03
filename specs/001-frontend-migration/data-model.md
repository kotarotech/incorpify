# Data Model: Frontend Migration

**Branch**: `001-frontend-migration` | **Date**: 2026-03-02

This document defines the frontend data model — the TypeScript types
and state shapes consumed by React components. These map to the
backend CRM/agents/checkout API responses but are owned by the
frontend as view models.

## Entity Definitions

### User

Represents the authenticated individual. Sourced from Auth0 +
CRM profiles endpoint.

```
User
├── id: UUID
├── iamId: string (Auth0 subject)
├── email: string
├── name: string
├── phonePrefix: string (e.g., "+971")
├── phoneNumber: string
├── birthDate: date (ISO string)
├── countryId: UUID (→ Country)
├── preferredLanguage: string (e.g., "en", "ar")
├── profileImageUrl: string (nullable)
├── isSuperAdmin: boolean
├── notifyInternal: boolean
├── notifyEmail: boolean
├── createdAt: datetime
└── updatedAt: datetime
```

### Space (Organization)

Top-level tenant entity. Maps to X-Space-Id header.

```
Space
├── id: UUID
├── name: string
├── ownerId: UUID (→ User)
├── subscriptionPlanId: UUID (→ SubscriptionPlan, nullable)
├── stripeCustomerId: string (nullable)
├── isActive: boolean
├── createdAt: datetime
└── updatedAt: datetime
```

### Company

Business entity within a space. Primary context for all service
operations.

```
Company
├── id: UUID
├── spaceId: UUID (→ Space)
├── name: string
├── tradeName: string (nullable)
├── jurisdictionId: UUID (→ Jurisdiction)
├── businessAreaId: UUID (→ BusinessArea)
├── status: enum (DRAFT, ACTIVE, SUSPENDED, DISSOLVED)
├── incorporationDate: date (nullable)
├── licenseNumber: string (nullable)
├── countryCode: string
├── currencyCode: string
├── createdAt: datetime
└── updatedAt: datetime
```

### CompanyMember

Junction between User and Company with role/permissions.

```
CompanyMember
├── id: UUID
├── companyId: UUID (→ Company)
├── userId: UUID (→ User)
├── role: enum (SUPER_ADMINISTRATOR, OWNER, ADMIN, MANAGER, VIEWER)
├── permissions: CompanyMemberPermission[]
├── joinedAt: datetime
└── updatedAt: datetime

CompanyMemberPermission
├── id: UUID
├── memberId: UUID (→ CompanyMember)
└── permission: enum (
│     manage:organizations, manage:billing, view:users,
│     remove:users, manage:invitations, manage:companies,
│     create:company, view:company, update:company,
│     manage:incorporation, view:documents, upload:documents,
│     download:documents
│   )
```

### CompanyInvitation

Pending invitation to join a company.

```
CompanyInvitation
├── id: UUID
├── companyId: UUID (→ Company)
├── email: string
├── role: enum (ADMIN, MANAGER, VIEWER)
├── status: enum (PENDING, ACCEPTED, CANCELLED, EXPIRED)
├── invitedBy: UUID (→ User)
├── acceptedAt: datetime (nullable)
├── expiresAt: datetime
├── createdAt: datetime
└── updatedAt: datetime
```

### Conversation

AI chatbot session. Sourced from agents service.

```
Conversation
├── id: UUID
├── userId: UUID (→ User, nullable for anonymous)
├── spaceId: UUID (→ Space, nullable)
├── companyId: UUID (→ Company, nullable)
├── type: enum (ANONYMOUS_LANDING, LANDING_LOGGED, DASHBOARD)
├── status: enum (ACTIVE, ARCHIVED, COMPLETED)
├── createdAt: datetime
└── updatedAt: datetime

Message
├── id: UUID
├── conversationId: UUID (→ Conversation)
├── role: enum (USER, ASSISTANT, SYSTEM, TOOL)
├── content: string
├── toolCalls: ToolCall[] (nullable)
├── toolResults: ToolResult[] (nullable)
├── createdAt: datetime
└── updatedAt: datetime
```

### CompanyWorkflow

Multi-step business process instance.

```
CompanyWorkflow
├── id: UUID
├── companyId: UUID (→ Company)
├── workflowKey: string (e.g., "UAE_INCORPORATION")
├── status: enum (PENDING, IN_PROGRESS, COMPLETED, FAILED, CANCELLED)
├── progress: number (0-100)
├── startedAt: datetime (nullable)
├── completedAt: datetime (nullable)
├── createdAt: datetime
└── updatedAt: datetime

WorkflowStep
├── id: UUID
├── companyWorkflowId: UUID (→ CompanyWorkflow)
├── stepKey: string
├── title: string
├── description: string
├── status: enum (PENDING, IN_PROGRESS, COMPLETED, FAILED,
│          WAITING_FOR_DEPENDENCIES, CANCELLED)
├── assignedTo: enum (CUSTOMER, ADMIN, SYSTEM)
├── assignedUserId: UUID (nullable)
├── orderNumber: number
├── dependencies: string[] (step keys)
├── stepData: object (form inputs)
├── result: object (nullable)
├── createdAt: datetime
└── updatedAt: datetime
```

### Checkout

Stripe checkout session.

```
CompanyCheckout
├── id: UUID
├── spaceId: UUID (→ Space)
├── companyId: UUID (→ Company, nullable)
├── conversationId: UUID (→ Conversation, nullable)
├── stripeSessionId: string (nullable)
├── stripePaymentIntentId: string (nullable)
├── status: enum (DRAFT, PENDING, PAID, FAILED, CANCELLED, REFUNDED)
├── currency: string (e.g., "USD", "AED")
├── totalAmountCents: number
├── createdAt: datetime
└── updatedAt: datetime

CheckoutItem
├── id: UUID
├── checkoutId: UUID (→ CompanyCheckout)
├── serviceId: UUID (→ Service)
├── description: string
├── quantity: number
├── unitPriceCents: number
├── totalPriceCents: number
└── createdAt: datetime
```

### Notification

System-generated alerts.

```
Notification
├── id: UUID
├── userId: UUID (→ User)
├── spaceId: UUID (→ Space)
├── type: enum (WORKFLOW_UPDATE, PAYMENT, INVITATION, SYSTEM)
├── title: string
├── message: string
├── isRead: boolean
├── data: object (nullable, contextual payload)
├── createdAt: datetime
└── updatedAt: datetime
```

### Reference Entities

Static/semi-static reference data.

```
Country
├── id: UUID
├── name: string
├── code: string (ISO 3166-1 alpha-2)
├── phonePrefix: string
└── currencyCode: string

Jurisdiction
├── id: UUID
├── name: string
├── countryId: UUID (→ Country)
├── code: string
└── description: string

BusinessArea
├── id: UUID
├── name: string
├── jurisdictionId: UUID (→ Jurisdiction)
├── code: string
└── description: string

Service
├── id: UUID
├── name: string
├── code: string
├── description: string
├── serviceType: string
├── isActive: boolean
└── prices: ServicePrice[]

ServicePrice
├── id: UUID
├── serviceId: UUID (→ Service)
├── currency: string
├── amountCents: number
├── billingPeriod: enum (ONE_TIME, MONTHLY, ANNUALLY)
└── isActive: boolean

SubscriptionPlan
├── id: UUID
├── name: string
├── description: string
├── priceCents: number
├── currency: string
├── billingPeriod: enum (MONTHLY, ANNUALLY)
├── features: string[]
└── isActive: boolean
```

## State Relationships

```
User ──< CompanyMember >── Company ──< Space
  │                          │
  │                          ├── CompanyWorkflow ──< WorkflowStep
  │                          ├── CompanyCheckout ──< CheckoutItem
  │                          ├── CompanyDocument
  │                          ├── CompanyInvitation
  │                          ├── InsurancePolicy
  │                          ├── Payroll ──< PayrollEntry
  │                          └── TaxRecord
  │
  ├── Conversation ──< Message
  └── Notification

Company ──> Jurisdiction ──> Country
Company ──> BusinessArea ──> Jurisdiction
CheckoutItem ──> Service ──> ServicePrice
Space ──> SubscriptionPlan
```

## Frontend State Ownership

| State Domain | Storage | Refresh Strategy |
|-------------|---------|-----------------|
| Auth session | Encrypted cookie | On mount + token refresh |
| Current space | React Context | On org switch |
| Current company | React Context | On company switch |
| User profile | React Query (cached) | Stale-while-revalidate |
| Company list | React Query (cached) | Stale-while-revalidate |
| Service data | React Query (cached) | Stale-while-revalidate |
| Conversation | React Query (streaming) | Real-time via SSE |
| Notifications | React Query (polling) | Poll every 30s |
| Reference data | React Query (long cache) | Cache for 1 hour |
