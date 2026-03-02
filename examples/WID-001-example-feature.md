---
id: WID-001
title: "Implement Google OAuth2 Authentication"
type: feature
status: in-progress
created: "2026-03-01"
author: "@ehammond"
parent: WID-000
depends_on:
  - WID-003
blocks:
  - WID-005
  - WID-006
priority: high
labels:
  - auth
  - backend
  - security
assignee: "@claude-agent"
estimate: 4h
due: "2026-03-15"
branch: "wid/WID-001-oauth-google"
pr: "https://gitea.example.com/org/app/pulls/42"
context:
  files:
    - path: "src/auth/oauth.py"
      role: "OAuth2 provider integration — Google-specific flow"
      glob: "src/auth/oauth*.py"
    - path: "src/auth/middleware.py"
      role: "Authentication middleware for protected routes"
    - path: "config/auth.yaml"
      role: "OAuth provider configuration (client IDs, scopes, redirect URIs)"
  modules:
    - name: "auth"
      description: "Authentication and authorization subsystem"
    - name: "users"
      description: "User model and profile management"
  apis:
    - endpoint: "POST /api/v1/auth/login"
      description: "Initiates OAuth2 flow, returns redirect URL"
    - endpoint: "GET /api/v1/auth/callback"
      description: "Handles OAuth2 callback, creates session"
    - endpoint: "POST /api/v1/auth/logout"
      description: "Invalidates session and refresh token"
x-team: platform
x-sprint: "2026-Q1-S3"
---

## Problem Statement

Users currently authenticate via email/password only. We need Google OAuth2
as a second authentication method to reduce signup friction and support
organizations that mandate SSO. Our user research shows 68% of target users
prefer OAuth over password-based signup.

## Design Discussion

**Why Google first (not GitHub or generic OIDC)?**
Our target users are business teams — Google Workspace is near-universal.
GitHub OAuth appeals to developers but not to the broader user base we're
targeting. We'll add generic OIDC as a follow-up (WID-012) once the
provider abstraction is solid.

**Session strategy: JWT vs server-side sessions.**
We chose server-side sessions (Redis-backed) over pure JWTs because:
- Token revocation is immediate (critical for security incidents)
- Session data doesn't bloat every request
- Refresh token rotation is simpler server-side

JWTs are still used as the transport token, but they're short-lived (15min)
and validated against the session store on each request.

**Redirect URI handling.**
The OAuth callback URL must be registered with Google. For local dev we use
`http://localhost:3000/auth/callback`; production uses the configured domain.
The `config/auth.yaml` file holds per-environment settings.

## Approach

1. Add `google-auth-oauthlib` dependency and configure the OAuth client
2. Implement the `/auth/login` endpoint to generate the authorization URL
   with appropriate scopes (`openid`, `email`, `profile`)
3. Implement the `/auth/callback` endpoint to exchange the authorization
   code for tokens, extract user info, and create/link the user account
4. Add session creation with Redis-backed storage
5. Implement token refresh logic with rotation
6. Add logout endpoint that invalidates both the session and the refresh token
7. Update the authentication middleware to accept both password and OAuth sessions
8. Add rate limiting on the login endpoint (5 attempts/min per IP)

## Acceptance Criteria

- [ ] User can authenticate via Google OAuth2
  ```yaml
  verify: test
  ref: tests/auth/test_oauth.py::test_full_flow
  ```
- [ ] New users are created automatically on first OAuth login
  ```yaml
  verify: test
  ref: tests/auth/test_oauth.py::test_new_user_creation
  ```
- [ ] Existing users can link their Google account
  ```yaml
  verify: test
  ref: tests/auth/test_oauth.py::test_account_linking
  ```
- [ ] Failed authentication attempts are rate-limited to 5/min per IP
  ```yaml
  verify: test
  ref: tests/auth/test_rate_limit.py::test_login_rate_limit
  ```
- [ ] Tokens expire after 15 minutes; refresh tokens rotate on use
  ```yaml
  verify: test
  ref: tests/auth/test_oauth.py::test_token_expiry_and_refresh
  ```
- [ ] Logout invalidates both session and refresh token
  ```yaml
  verify: test
  ref: tests/auth/test_oauth.py::test_logout_invalidation
  ```
- [ ] OAuth configuration is environment-aware (dev/staging/prod)
  ```yaml
  verify: manual
  ref: config/auth.yaml
  ```

## Codebase Context

The `auth` module (`src/auth/`) currently handles password-based login only.
The middleware (`src/auth/middleware.py`) extracts a bearer token and validates
it — this needs to be extended to also accept OAuth-issued session tokens.

The user model (`src/users/models.py`) has an `email` field but no
`oauth_provider` or `oauth_id` fields yet — these need to be added via
migration.

Redis is already configured for caching (`src/cache/redis.py`) — we'll
reuse the same connection pool for session storage.

## Changelog

- **2026-03-01** (author: @ehammond) -- Created from design session discussing auth strategy
- **2026-03-02** (author: @claude-agent) -- Added rate-limiting requirement after reviewing OWASP guidelines
- **2026-03-02** (author: @ehammond) -- Clarified JWT vs server-side session decision in Design Discussion

## Execution Log

| Session | Agent | Started | Duration | Status | Log |
|---------|-------|---------|----------|--------|-----|
| exec-001 | claude-code | 2026-03-02 | 12m | completed | [log](../logs/WID-001/exec-001.md) |
| exec-002 | claude-code | 2026-03-02 | 23m | in-progress | [log](../logs/WID-001/exec-002.md) |

## Notes

- Google Cloud project credentials are stored in 1Password vault "Engineering"
  under "Google OAuth - Production". Do **not** commit client secrets to the repo.
- The generic OIDC follow-up (WID-012) should reuse the provider abstraction
  we're building here. Design the interface with that in mind.
