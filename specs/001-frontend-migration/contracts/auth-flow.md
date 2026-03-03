# Authentication Flow Contract

**Branch**: `001-frontend-migration` | **Date**: 2026-03-02

## OAuth2 Authorization Code Flow

```
User                Frontend (Fastify)         Auth0              Backend Services
 │                       │                       │                       │
 │  GET /auth/login      │                       │                       │
 │──────────────────────>│                       │                       │
 │                       │  302 → /authorize     │                       │
 │                       │──────────────────────>│                       │
 │  Auth0 login page     │                       │                       │
 │<──────────────────────────────────────────────│                       │
 │                       │                       │                       │
 │  Submit credentials   │                       │                       │
 │──────────────────────────────────────────────>│                       │
 │                       │                       │                       │
 │                       │  302 + code           │                       │
 │                       │<──────────────────────│                       │
 │                       │                       │                       │
 │                       │  POST /oauth/token    │                       │
 │                       │──────────────────────>│                       │
 │                       │  {access, refresh, id}│                       │
 │                       │<──────────────────────│                       │
 │                       │                       │                       │
 │                       │  Set-Cookie (public + private)                │
 │  302 → /dashboard     │                       │                       │
 │<──────────────────────│                       │                       │
 │                       │                       │                       │
 │  GET /dashboard       │                       │                       │
 │──────────────────────>│                       │                       │
 │                       │  GET /api/v1/profiles/me                     │
 │                       │  Authorization: Bearer {token}               │
 │                       │──────────────────────────────────────────────>│
 │                       │  {user, companies}    │                       │
 │                       │<──────────────────────────────────────────────│
 │  Dashboard HTML       │                       │                       │
 │<──────────────────────│                       │                       │
```

## Frontend Auth Routes

| Route | Method | Description |
|-------|--------|-------------|
| `/auth/login` | GET | Redirect to Auth0 authorize URL |
| `/auth/login?returnTo=/path` | GET | Login with return URL |
| `/auth/login?referral=code` | GET | Login with referral code |
| `/auth/login?invite=id` | GET | Login to accept invitation |
| `/auth/callback` | GET | Auth0 callback; exchange code for tokens |
| `/auth/logout` | GET/POST | Clear session cookies; redirect to Auth0 logout |
| `/auth/session` | GET | Return current public session as JSON |
| `/auth/refresh` | POST | Refresh access token using refresh token |

## Cookie Schema

### Public Session Cookie

```
Name: incorpify.session
HttpOnly: false
Secure: true (production)
SameSite: Lax
MaxAge: 7 days
Path: /
Encrypted: yes (libsodium)

Payload:
{
  isLoggedIn: boolean
  expires_at: string (ISO datetime)
  user: {
    id: string
    name: string
    email: string
    image: string | null
    isSuperAdmin: boolean
  }
  spaceId: string | null
  companyId: string | null
}
```

### Private Session Cookie

```
Name: incorpify.session.private
HttpOnly: true
Secure: true (production)
SameSite: Lax
MaxAge: 7 days
Path: /
Encrypted: yes (libsodium)

Payload:
{
  access_token: string
  refresh_token: string
  id_token: string
  token_type: "Bearer"
  expires_in: number (seconds)
}
```

## Token Refresh Flow

```
Client                  Frontend (Fastify)         Auth0
 │                            │                       │
 │  API call returns 401      │                       │
 │  (or token near expiry)    │                       │
 │                            │                       │
 │  POST /auth/refresh        │                       │
 │───────────────────────────>│                       │
 │                            │  POST /oauth/token    │
 │                            │  grant_type=refresh   │
 │                            │──────────────────────>│
 │                            │  {new access_token}   │
 │                            │<──────────────────────│
 │                            │                       │
 │  Set-Cookie (updated)      │                       │
 │  200 {success: true}       │                       │
 │<───────────────────────────│                       │
 │                            │                       │
 │  Retry original request    │                       │
 │  with new token            │                       │
```

## Logout Flow

```
Client                  Frontend (Fastify)         Auth0
 │                            │                       │
 │  GET /auth/logout          │                       │
 │───────────────────────────>│                       │
 │                            │  Clear all cookies    │
 │                            │                       │
 │  302 → Auth0 /v2/logout    │                       │
 │<───────────────────────────│                       │
 │                            │                       │
 │  GET Auth0 logout          │                       │
 │───────────────────────────────────────────────────>│
 │                            │                       │
 │  302 → /                   │                       │
 │<───────────────────────────────────────────────────│
```

## Auth0 Configuration

```
Domain: dev-xvppy7hh2okyawyy.us.auth0.com
Client ID: (from environment)
Client Secret: (from secret manager)
Callback URL: {APP_URL}/auth/callback
Logout URL: {APP_URL}
Audience: (API identifier from Auth0)
Scopes: openid profile email offline_access
```
