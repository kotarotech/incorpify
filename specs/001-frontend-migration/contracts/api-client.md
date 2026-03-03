# API Client Contract

**Branch**: `001-frontend-migration` | **Date**: 2026-03-02

The frontend communicates with four backend services via HTTP.
This document defines the API client contract — how the frontend
constructs, authenticates, and handles requests/responses.

## Service Endpoints

| Service | Base URL (dev) | Base URL (prod) |
|---------|---------------|-----------------|
| CRM | `http://localhost:3002/api/v1` | `{API_GATEWAY}/api/v1` |
| Agents | `http://localhost:3000/agents/v1` | `{API_GATEWAY}/agents/v1` |
| Checkout | `http://localhost:3001/api/v1` | `{API_GATEWAY}/api/v1` |
| Notification | `http://localhost:3003/api/v1` | `{API_GATEWAY}/api/v1` |

In production, the API Gateway routes by path prefix. In
development, the frontend proxies to individual service ports.

## Request Headers

Every authenticated request MUST include:

```
Authorization: Bearer {access_token}
X-Space-Id: {active_space_id}
X-Organization-Id: {active_space_id}  (legacy alias)
X-Request-Id: {uuid}                  (generated per request)
Content-Type: application/json
```

## Response Envelope

All services return the standard envelope:

```typescript
interface ApiResponse<T> {
  success: boolean;
  data: T | null;
  error: ApiError | null;
}

interface ApiError {
  code: string;
  message: string;
}
```

## Error Handling

| HTTP Status | Client Behavior |
|-------------|----------------|
| 400 | Display validation errors from `error.message` |
| 401 | Attempt token refresh; if fails, redirect to login |
| 403 | Display "Access Denied" with role context |
| 404 | Display "Not Found" with navigation back |
| 409 | Display conflict message (e.g., duplicate invite) |
| 429 | Display rate limit message; retry after header |
| 500+ | Display generic error with retry button |

## Pagination

All list endpoints support:

```
GET /resource?limit=20&offset=0
```

Response includes pagination metadata in `data`:

```typescript
interface PaginatedResponse<T> {
  items: T[];
  total: number;
  limit: number;
  offset: number;
  hasMore: boolean;
}
```

## CRM Service Endpoints

### Spaces
- `GET /spaces/:spaceId`
- `POST /spaces`
- `PATCH /spaces/:spaceId`

### Profiles
- `GET /profiles/me/companies`
- `GET /profiles/:userId`
- `PATCH /profiles/:userId`
- `GET /profiles/search?email={email}`

### Companies
- `GET /spaces/:spaceId/companies`
- `GET /spaces/:spaceId/companies/:companyId`
- `POST /spaces/:spaceId/companies`
- `PATCH /spaces/:spaceId/companies/:companyId`

### Company Members
- `GET /spaces/:spaceId/companies/:companyId/members`
- `PATCH /spaces/:spaceId/companies/:companyId/members/:memberId/role`
- `DELETE /spaces/:spaceId/companies/:companyId/members/:memberId`

### Company Invitations
- `GET /spaces/:spaceId/companies/:companyId/invitations`
- `POST /spaces/:spaceId/companies/:companyId/invitations`
- `POST /spaces/:spaceId/companies/:companyId/invitations/:id/resend`
- `POST /spaces/:spaceId/companies/:companyId/invitations/:id/cancel`
- `POST /spaces/:spaceId/invitations/:id/accept`

### Company People
- `GET /spaces/:spaceId/companies/:companyId/people`
- `POST /spaces/:spaceId/companies/:companyId/people`
- `PATCH /spaces/:spaceId/companies/:companyId/people/:personId`
- `DELETE /spaces/:spaceId/companies/:companyId/people/:personId`

### Company Documents
- `GET /spaces/:spaceId/companies/:companyId/documents`
- `POST /spaces/:spaceId/companies/:companyId/documents`
- `PATCH /spaces/:spaceId/companies/:companyId/documents/:documentId`
- `DELETE /spaces/:spaceId/companies/:companyId/documents/:documentId`
- `POST /spaces/:spaceId/companies/:companyId/documents/upload-url`
- `POST /spaces/:spaceId/companies/:companyId/documents/confirm-upload`
- `GET /spaces/:spaceId/companies/:companyId/documents/:id/download-url`

### Company Workflows
- `GET /spaces/:spaceId/companies/:companyId/workflows`
- `GET /spaces/:spaceId/companies/:companyId/workflows/incorporation-status`
- `POST /spaces/:spaceId/companies/:companyId/workflows`
- `GET /spaces/:spaceId/companies/:companyId/workflows/:workflowId`
- `PATCH /spaces/:spaceId/companies/:companyId/workflows/:workflowId`
- `GET /spaces/:spaceId/companies/:companyId/workflows/:workflowId/steps`
- `POST /spaces/:spaceId/companies/:companyId/workflows/:workflowId/steps/:stepId/start`
- `POST /spaces/:spaceId/companies/:companyId/workflows/:workflowId/steps/:stepId/submit`
- `POST /spaces/:spaceId/companies/:companyId/workflows/:workflowId/steps/:stepId/complete`

### Payroll
- `GET /spaces/:spaceId/companies/:companyId/payrolls`
- `POST /spaces/:spaceId/companies/:companyId/payrolls`
- `PATCH /spaces/:spaceId/companies/:companyId/payrolls/:id`
- `DELETE /spaces/:spaceId/companies/:companyId/payrolls/:id`

### Insurance
- `GET /spaces/:spaceId/companies/:companyId/insurance-policies`
- `POST /spaces/:spaceId/companies/:companyId/insurance-policies`
- `PATCH /spaces/:spaceId/companies/:companyId/insurance-policies/:id`
- `DELETE /spaces/:spaceId/companies/:companyId/insurance-policies/:id`

### Tax Records
- `GET /spaces/:spaceId/companies/:companyId/tax-records`
- `POST /spaces/:spaceId/companies/:companyId/tax-records`
- `PATCH /spaces/:spaceId/companies/:companyId/tax-records/:id`
- `DELETE /spaces/:spaceId/companies/:companyId/tax-records/:id`

### Reference Data
- `GET /countries`
- `GET /jurisdictions`
- `GET /business-areas`
- `GET /business-areas/:id/visa-prices`
- `GET /services`
- `GET /services/:id/prices`

### Checkouts
- `POST /spaces/:spaceId/checkouts/sessions`
- `POST /spaces/:spaceId/checkouts/sessions/sku`
- `GET /spaces/:spaceId/checkouts`
- `GET /spaces/:spaceId/checkouts/:checkoutId`
- `GET /spaces/:spaceId/checkouts/:checkoutId/items`

## Agents Service Endpoints

### Conversations
- `POST /conversations` — Create conversation
- `POST /conversations/:id/messages` — Send message
- `POST /conversations/:id/messages?stream=true` — Stream message (SSE)
- `GET /conversations/:id` — Get full conversation
- `GET /conversations/:id/messages` — Get messages (paginated)

### Checkout (via conversation)
- `GET /conversations/:id/checkout` — Get/create checkout session
- `PATCH /conversations/:id/checkout` — Switch currency
- `POST /conversations/:id/checkout/proceed` — Redirect to Stripe
- `POST /conversations/:id/checkout/cancel` — Cancel checkout

## Notification Service Endpoints

### Notifications (read via CRM, send via notification service)
- Notifications are read through the CRM service endpoints
- Send operations are internal (M2M from CRM → notification service)

## Stripe Webhook

- `POST /stripe-webhooks/stripe` — Stripe webhook (no auth, signature verified)
