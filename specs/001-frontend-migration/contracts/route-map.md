# Route Map Contract

**Branch**: `001-frontend-migration` | **Date**: 2026-03-02

## URL Structure

All routes use locale prefix for public pages. Authenticated routes
do not use locale prefix.

### Public Routes (SSR, SEO-indexed)

| Route | Page | SSR |
|-------|------|-----|
| `/:locale` | Landing page | Yes |
| `/:locale/services` | Services overview | Yes |
| `/:locale/services/incorporation` | Incorporation details | Yes |
| `/:locale/services/accounting` | Accounting details | Yes |
| `/:locale/services/payroll` | Payroll details | Yes |
| `/:locale/services/insurance` | Insurance details | Yes |
| `/:locale/services/legal` | Legal services details | Yes |
| `/:locale/services/residency` | Residency details | Yes |
| `/:locale/services/banking` | Banking details | Yes |
| `/:locale/pricing` | Pricing plans | Yes |
| `/:locale/blog` | Blog listing | Yes |
| `/:locale/blog/:slug` | Blog post | Yes |
| `/:locale/contact-us` | Contact form | Yes |
| `/:locale/privacy-policy` | Privacy policy | Yes |
| `/:locale/terms-and-conditions` | Terms | Yes |
| `/:locale/cookie-policy` | Cookie policy | Yes |
| `/:locale/pre-incorporation` | Pre-incorporation chat | Yes |
| `/:locale/pre-incorporation/:conversationId` | Chat session | Partial |

### Auth Routes (Server-side, no UI)

| Route | Description |
|-------|-------------|
| `/auth/login` | Initiate Auth0 flow |
| `/auth/callback` | Auth0 callback handler |
| `/auth/logout` | Clear session, redirect |
| `/auth/session` | Return session JSON |
| `/auth/refresh` | Refresh access token |

### Protected Routes (SPA, client-side routing)

| Route | Page | RBAC |
|-------|------|------|
| `/organization` | Organization selector | Authenticated |
| `/organization/:spaceId/create-company` | Create company form | manage:companies |
| `/organization/:spaceId/company/:companyId/dashboard` | Company dashboard | view:company |
| `/organization/:spaceId/company/:companyId/accounting` | Accounting | view:company |
| `/organization/:spaceId/company/:companyId/payroll` | Payroll management | view:company |
| `/organization/:spaceId/company/:companyId/tax` | Tax records | view:company |
| `/organization/:spaceId/company/:companyId/banking` | Banking | view:company |
| `/organization/:spaceId/company/:companyId/insurance` | Insurance | view:company |
| `/organization/:spaceId/company/:companyId/visa-management` | Visa management | view:company |
| `/organization/:spaceId/company/:companyId/compliance` | Compliance | view:company |
| `/organization/:spaceId/company/:companyId/documents` | Documents | view:documents |
| `/organization/:spaceId/company/:companyId/people` | People (directors, etc.) | view:company |
| `/organization/:spaceId/company/:companyId/members` | Team members | view:users |
| `/organization/:spaceId/company/:companyId/workflows` | Workflows | view:company |
| `/organization/:spaceId/company/:companyId/workflows/:workflowId` | Workflow detail | view:company |
| `/organization/:spaceId/company/:companyId/settings` | Company settings | update:company |
| `/organization/:spaceId/settings` | Organization settings | manage:organizations |
| `/organization/:spaceId/billing` | Billing & subscription | manage:billing |
| `/account/profile-completion` | Onboarding profile | Authenticated |
| `/account/email-verification` | Email verification | Authenticated |
| `/account/settings` | User settings | Authenticated |

### Locale Support

- Supported locales: `en`, `ar`
- Default locale: `en`
- Locale detection: Cookie → Browser Accept-Language → Default
- Arabic locale triggers `dir="rtl"` on `<html>` element

### Route Guards

```
Public routes:      No guard (accessible to all)
Auth routes:        Server-side only (Fastify handlers)
Protected routes:   Require valid session cookie
  └── RBAC routes:  Additionally check user permissions via CompanyContext
```

### Redirects (SEO preservation)

| Old URL Pattern | New URL Pattern | Status |
|----------------|-----------------|--------|
| `/en/services/*` | `/:locale/services/*` | 301 (if locale differs) |
| `/organization` (no space) | `/organization/:defaultSpaceId` | 302 |
| `/organization/:id/company/:id` (no page) | `.../dashboard` | 302 |
