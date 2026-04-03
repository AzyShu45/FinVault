# Security & Architecture Fixes — FinVault

This document explains every security improvement I made to the FinVault backend, why each one was needed, and how it was implemented.

***

## P0 — Critical Fixes

### 1. Public Registration Allowed Self-Assigning the ADMIN Role

**Problem:** The `POST /api/auth/register` endpoint accepted a `role` field from the client. Any anonymous user could send `"role": "ADMIN"` in the request body and immediately gain full system access — creating records, managing other users, and reading all data. This is a textbook OWASP A01: Broken Access Control violation.

**Fix:** Removed the `role` field from `RegisterRequest` entirely. The backend now always assigns `Role.VIEWER` on public registration. If a user needs a higher role, an admin must explicitly update it via `PUT /api/users/{id}`.

**Files changed:**
- `RegisterRequest.java` — removed `role` field
- `AuthService.register()` — hardcoded `Role.VIEWER`
- `index.html` — removed role dropdown from the registration form
- `AccessControlTest.java` — added `publicRegistration_alwaysAssignsViewer` test

This follows the Principle of Least Privilege: users start with minimum access and get elevated only through an intentional admin action.

***

### 2. Password Reset Token Was Leaking in the API Response

**Problem:** The `POST /api/auth/forgot-password` endpoint returned the raw reset token directly in the JSON response body. This means the token showed up in:
- The browser's DevTools Network tab
- Reverse proxy and load balancer access logs
- Any APM tool capturing HTTP payloads
- Network traffic if served over plain HTTP

This is OWASP A07: Identification and Authentication Failures.

**Fix:** The reset token is never included in the API response. Instead:
- The endpoint returns a generic message: *"If an account exists with this email, a password reset link has been sent."*
- The token is written to the server console with a `SIMULATED EMAIL` prefix (in production, this becomes `JavaMailSender` / SendGrid)
- The generic message also prevents email enumeration — an attacker can't tell whether an email address is registered

**Files changed:**
- `AuthService.forgotPassword()` — token removed from response, added `log.info()` for simulated delivery
- `PasswordResetResponse.java` — removed `resetToken` field
- `index.html` — updated forgot password UI to show a manual token entry prompt

The simulated email approach means the API contract is already production-correct — the only change for real deployment is wiring in an email sender.

***

### 3. JWT Secret Was Hardcoded in Source Code

**Problem:** The JWT signing secret was a plain Base64 string committed directly to `application.yml`. Anyone with repo access could decode it and forge valid JWT tokens for any user — including ADMIN. Even rotating the secret later doesn't fully help because the old value stays in Git history.

**Fix:**
- The secret is now read from the `APP_JWT_SECRET` environment variable, with a fallback default for local dev only
- `JwtService` has a `@PostConstruct` method that logs a clear **WARNING** at startup if the default insecure key is still in use
- All environment-specific values (secret, port, expiration) use `${ENV_VAR:default}` syntax throughout `application.yml`

**Files changed:**
- `application.yml` — `secret: ${APP_JWT_SECRET:...}`, `port: ${PORT:8080}`
- `JwtService.java` — added `@PostConstruct validateSecret()`
- `render.yaml` / `Dockerfile` — production deployments inject `APP_JWT_SECRET` via env

This follows the 12-Factor App methodology: configuration belongs in the environment, not in code.

***

## P1 — Operational Fixes

### 4. No Rate Limiting on Authentication Endpoints

**Problem:** `/api/auth/login` and `/api/auth/forgot-password` had no throttling. An attacker could run unlimited brute-force attempts, enumerate emails, or just exhaust server resources with automated requests.

**Fix:** Added `RateLimitFilter` — a servlet filter that allows each client IP a maximum of **10 requests per minute** on all `/api/auth/**` endpoints. Requests beyond the limit return HTTP 429 Too Many Requests.

Implementation details:
- Uses a `ConcurrentHashMap` with per-IP sliding time windows
- Reads `X-Forwarded-For` to get the real IP when running behind a reverse proxy
- The limit is configurable via `app.rate-limit.max-requests` — test environments set this to 1000 so integration tests don't get throttled

**Files changed:**
- `RateLimitFilter.java` — new filter component
- `application.yml` — added `max-requests` property
- `src/test/resources/application.yml` — overridden to 1000 for tests

***

### 5. Dev Tools Exposed in Production

**Problem:** A single `application.yml` served all environments. In production this meant:
- H2 console accessible at `/h2-console` with no password — full database access to anyone
- Swagger UI publicly exposing all API contracts
- `ddl-auto: update` potentially modifying the production schema silently

**Fix:** Added a separate `application-prod.yml` that:
- Disables H2 console
- Disables Swagger / OpenAPI endpoints
- Sets `ddl-auto: validate` — Hibernate checks schema matches entities but never alters it

Activated via `SPRING_PROFILES_ACTIVE=prod` environment variable on the server.

**Files changed:**
- `application.yml` — added `profiles.active: ${SPRING_PROFILES_ACTIVE:dev}`
- `application-prod.yml` — new production profile

***

### 6. Forgot Password Revealed Whether an Email Was Registered

**Problem:** When you submitted an unregistered email to `/api/auth/forgot-password`, the original endpoint returned a different error message than for registered emails. An attacker could use this to enumerate which emails have accounts.

**Fix:** The endpoint now always returns the same generic message regardless of whether the email exists in the system. Account existence is never revealed to the caller.

This is the same behavior you see in production systems like GitHub and Google's password reset flows.

***

## P2 — Performance & Architecture Improvements

### 7. Dashboard Queries Running on Every Request

**Problem:** `DashboardService.getSummary()` executed 4–5 database queries on each request. For a read-heavy endpoint accessed by every analyst and viewer on page load, this was unnecessary load.

**Fix:** Added Spring Cache with Caffeine:
- `@Cacheable("dashboardSummary")` on `getSummary()` — results cached for 60 seconds
- `@CacheEvict("dashboardSummary")` on create, update, and delete operations — cache invalidates whenever data changes

The 60-second TTL is a deliberate tradeoff: dashboard analytics don't need real-time freshness, but should reflect recent changes within a minute.

**Files changed:**
- `pom.xml` — added `spring-boot-starter-cache` and `caffeine`
- `FinanceApplication.java` — added `@EnableCaching`
- `DashboardService.java` — added `@Cacheable`
- `FinancialRecordService.java` — added `@CacheEvict` on all write methods
- `application.yml` — Caffeine spec configuration

***

### 8. Docker Build Was Skipping Tests

**Problem:** The Dockerfile ran `./mvnw clean package -DskipTests`, which meant the Docker image could be built from broken code.

**Fix:** Removed `-DskipTests`. The Docker build now runs all 15 tests before packaging, so a failing test stops the image from being built and deployed.

**Files changed:**
- `Dockerfile` — removed `-DskipTests` flag

***

### 9. No Regression Test for Role Enforcement

**Problem:** The original test suite didn't verify that the role-enforcement fix was working. If someone accidentally removed the server-side role hardcoding, no test would catch it.

**Fix:** Added `publicRegistration_alwaysAssignsViewer` — a test that sends `"role":"ADMIN"` in the registration payload and asserts the response still contains `"role":"VIEWER"`. It will fail loudly if the enforcement is ever removed.

**Files changed:**
- `AccessControlTest.java` — added test, refactored to use seeded users

***

## Summary

| # | Severity | Fix | OWASP Category |
|---|----------|-----|----------------|
| 1 | **P0** | Registration locked to VIEWER | A01: Broken Access Control |
| 2 | **P0** | Reset token removed from response | A07: Auth Failures |
| 3 | **P0** | JWT secret via environment variable | A02: Cryptographic Failures |
| 4 | **P1** | Rate limiting on auth endpoints | A07: Auth Failures |
| 5 | **P1** | Dev/prod profile separation | A05: Security Misconfiguration |
| 6 | **P1** | Generic forgot-password response | A07: Auth Failures |
| 7 | **P2** | Dashboard caching (Caffeine, 60s TTL) | Performance |
| 8 | **P2** | Docker build runs tests | Infrastructure |
| 9 | **P2** | Role enforcement regression test | Testing |

## Notes & Tradeoffs

**Email is simulated via console logging.** In production, swapping `log.info()` for `JavaMailSender` + SendGrid gives you the exact same API contract — the response never contains the token in either case.

**H2 stays as the database** for easy local setup. The `application-prod.yml` profile is already configured for a real database — just set the datasource env vars and add Flyway migrations.

**Rate limiting is in-memory**, which is fine for a single instance. For horizontal scaling, the right approach is Redis-backed rate limiting with bucket4j. The current implementation is easy to replace because it's isolated in a single filter class.

**The default JWT secret still works in dev** so local development isn't blocked. The `@PostConstruct` warning makes sure developers notice. In production, `APP_JWT_SECRET` is required.