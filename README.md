# FinVault — Finance Data Processing and Access Control Backend

A full-stack finance dashboard built with **Java 17, Spring Boot 3.2, and H2 Database**. It supports JWT authentication, role-based access control, financial records management, dashboard analytics with caching, rate limiting, and a custom HTML/CSS/JS frontend — all bundled into a single runnable JAR.

> **Quick Start:** Clone the repo, run `./mvnw spring-boot:run`, and open `http://localhost:8080`. No database setup needed.

***

## Table of Contents

- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
- [Default Users](#default-users)
- [API Endpoints](#api-endpoints)
- [Access Control Matrix](#access-control-matrix)
- [Features Implemented](#features-implemented)
- [API Usage Examples](#api-usage-examples)
- [Frontend Pages](#frontend-pages)
- [Security Measures](#security-measures)
- [Design Decisions & Assumptions](#design-decisions--assumptions)
- [What I Would Do Differently in Production](#what-i-would-do-differently-in-production)
- [Project Structure](#project-structure)
- [Running Tests](#running-tests)

***

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 17 |
| Framework | Spring Boot 3.2.5 |
| Security | Spring Security + JWT (jjwt 0.12.5) |
| Database | H2 (embedded, file-based, zero-setup) |
| ORM | Spring Data JPA / Hibernate |
| Validation | Jakarta Bean Validation |
| Caching | Spring Cache + Caffeine (in-memory) |
| API Docs | Springdoc OpenAPI / Swagger UI |
| Build | Maven + Maven Wrapper |
| CI/CD | GitHub Actions |
| Frontend | Vanilla HTML5 + CSS3 + JavaScript + Chart.js |

***

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Client (Browser)                     │
│              index.html  (FinVault Dashboard)            │
└────────────────────────┬─────────────────────────────────┘
                         │  HTTP + JWT Bearer Token
                         ▼
┌──────────────────────────────────────────────────────────┐
│                   Spring Boot Application                │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ Rate Limit  │→ │  JWT Auth    │→ │  Security      │  │
│  │ Filter      │  │  Filter      │  │  Config (RBAC) │  │
│  └─────────────┘  └──────────────┘  └────────────────┘  │
│                         │                                │
│  ┌──────────────────────▼───────────────────────────┐    │
│  │              REST Controllers                     │    │
│  │  AuthController │ RecordController │ Dashboard   │    │
│  │  UserController │                  │ Controller  │    │
│  └──────────────────────┬───────────────────────────┘    │
│                         │                                │
│  ┌──────────────────────▼───────────────────────────┐    │
│  │              Service Layer                        │    │
│  │  AuthService │ RecordService │ DashboardService   │    │
│  │  UserService │               │ (@Cacheable 60s)   │    │
│  └──────────────────────┬───────────────────────────┘    │
│                         │                                │
│  ┌──────────────────────▼───────────────────────────┐    │
│  │           Spring Data JPA Repositories            │    │
│  │    UserRepo │ FinancialRecordRepo │ ResetTokenRepo│    │
│  └──────────────────────┬───────────────────────────┘    │
│                         │                                │
│  ┌──────────────────────▼───────────────────────────┐    │
│  │            H2 Database (file-based)               │    │
│  │     users │ financial_records │ reset_tokens      │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

**Layered separation:**
- **Controllers** — handle HTTP only, no business logic
- **Services** — all business rules, validation, and caching live here
- **Repositories** — database queries via Spring Data JPA (JPQL)
- **DTOs** — request/response objects are separate from entities
- **Filters** — rate limiting and JWT auth as cross-cutting concerns

***

## Getting Started

### Prerequisites

- **Java 17+** — check with `java -version`
- **Maven 3.8+** — or just use the included `./mvnw` wrapper

No external database, Docker, or extra services needed.

### Run the Application

```bash
git clone https://github.com/AzyShu45/FinVault.git
cd FinVault
./mvnw spring-boot:run
```

The app starts on **http://localhost:8080** in about 6 seconds.

### Available URLs

| URL | Description |
|-----|-------------|
| http://localhost:8080 | FinVault Dashboard (Login page) |
| http://localhost:8080/swagger-ui.html | Interactive API documentation |
| http://localhost:8080/h2-console | Database browser (JDBC URL: `jdbc:h2:file:./data/finance_db`, user: `sa`, no password) |

***

## Default Users

Three users are seeded on first startup so you can start testing immediately:

| Username | Password | Role | What they can do |
|----------|----------|------|-----------------|
| `admin` | `admin123` | ADMIN | Full access — records, users, dashboard |
| `analyst` | `analyst123` | ANALYST | View records and dashboard summaries |
| `viewer` | `viewer123` | VIEWER | Read records only |

***

## API Endpoints

### Authentication (`/api/auth`) — Public, Rate Limited (10 req/min)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/register` | Register a new user (always assigned VIEWER role) |
| POST | `/api/auth/login` | Login and receive a JWT token |
| POST | `/api/auth/forgot-password` | Request a password reset token |
| POST | `/api/auth/reset-password` | Reset password using the token |

### Financial Records (`/api/records`)

| Method | Endpoint | Allowed Roles | Description |
|--------|----------|---------------|-------------|
| GET | `/api/records` | VIEWER, ANALYST, ADMIN | List records with filters and pagination |
| GET | `/api/records/{id}` | VIEWER, ANALYST, ADMIN | Get a single record |
| POST | `/api/records` | ADMIN | Create a new record |
| PUT | `/api/records/{id}` | ADMIN | Update an existing record |
| DELETE | `/api/records/{id}` | ADMIN | Soft-delete a record |

**Filter params:** `type` (INCOME/EXPENSE), `category`, `startDate`, `endDate`, `page`, `size`, `sort`

### Dashboard (`/api/dashboard`) — Analyst & Admin

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/dashboard/summary` | Totals, category breakdown, monthly trends, recent activity |

Response includes: `totalIncome`, `totalExpenses`, `netBalance`, `categoryWiseTotals`, `recentActivity` (last 10), `monthlyTrends` (last 6 months)

### User Management (`/api/users`) — Admin Only

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/users` | List all users |
| GET | `/api/users/{id}` | Get a user by ID |
| PUT | `/api/users/{id}` | Update role, status, name, or email |
| DELETE | `/api/users/{id}` | Deactivate a user (soft delete) |

***

## Access Control Matrix

| Action | VIEWER | ANALYST | ADMIN |
|--------|--------|---------|-------|
| View financial records | ✅ | ✅ | ✅ |
| Filter and paginate records | ✅ | ✅ | ✅ |
| View dashboard summary | ❌ | ✅ | ✅ |
| Create / update / delete records | ❌ | ❌ | ✅ |
| Manage users | ❌ | ❌ | ✅ |
| Public registration | VIEWER only | — | — |
| Elevate user roles | ❌ | ❌ | ✅ |

***

## Features Implemented

### Core Requirements

| # | Requirement | Status | Notes |
|---|------------|--------|-------|
| 1 | User and Role Management | ✅ Done | Three roles, user CRUD, status management, role elevation by admin only |
| 2 | Financial Records Management | ✅ Done | Full CRUD with type/category/date filtering, pagination, soft delete |
| 3 | Dashboard Summary APIs | ✅ Done | Totals, category breakdown, recent activity, 6-month trends |
| 4 | Access Control | ✅ Done | Spring Security filter chain + `@PreAuthorize` method guards |
| 5 | Validation and Error Handling | ✅ Done | Bean Validation on all DTOs, global exception handler with proper HTTP status codes |
| 6 | Data Persistence | ✅ Done | H2 file-based DB, JPA entities with indexes, BigDecimal for money |

### Optional Enhancements

| Enhancement | Status | Details |
|------------|--------|---------|
| JWT Authentication | ✅ Done | 24-hour expiry, Bearer token in Authorization header |
| Pagination | ✅ Done | `page`, `size`, `sort` query params via Spring Data Pageable |
| Soft Delete | ✅ Done | `deleted` flag on records; `active` flag on users |
| Rate Limiting | ✅ Done | 10 requests/minute per IP on auth endpoints (returns HTTP 429) |
| Tests | ✅ Done | 15 integration tests covering auth, RBAC, and role enforcement |
| API Documentation | ✅ Done | Swagger UI with JWT auth support |
| Password Reset | ✅ Done | Token-based flow, 15-minute expiry, single-use tokens |
| Caching | ✅ Done | Caffeine cache on dashboard (60s TTL), auto-evict on data changes |
| CI/CD | ✅ Done | GitHub Actions — build + test + Docker on every push |
| Frontend | ✅ Done | FinVault dashboard — login, records, analytics, user management |
| Security Hardening | ✅ Done | See [Security Measures](#security-measures) |

***

## API Usage Examples

### 1. Login

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

Response:
```json
{
  "token": "eyJhbGciOiJIUzM4NCJ9...",
  "username": "admin",
  "role": "ADMIN"
}
```

### 2. Register a New User

New registrations always get the VIEWER role. Role changes require an admin.

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"azad","email":"azad@example.com","password":"pass123","fullName":"Azad Shukla"}'
```

### 3. Create a Financial Record

```bash
curl -X POST http://localhost:8080/api/records \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"amount":5000,"type":"INCOME","category":"Salary","date":"2026-04-01","description":"April salary"}'
```

### 4. Filter Records with Pagination

```bash
curl "http://localhost:8080/api/records?type=EXPENSE&category=Food&startDate=2026-01-01&endDate=2026-12-31&page=0&size=10&sort=date,desc" \
  -H "Authorization: Bearer <token>"
```

### 5. Dashboard Summary

```bash
curl http://localhost:8080/api/dashboard/summary \
  -H "Authorization: Bearer <token>"
```

Response:
```json
{
  "totalIncome": 8000.00,
  "totalExpenses": 2200.00,
  "netBalance": 5800.00,
  "categoryWiseTotals": {
    "Salary": 5000,
    "Freelance": 3000,
    "Rent": 1200,
    "Food": 800
  },
  "recentActivity": [ "..." ],
  "monthlyTrends": [
    { "month": "2026-03", "income": 3000, "expenses": 800, "net": 2200 },
    { "month": "2026-04", "income": 5000, "expenses": 1400, "net": 3600 }
  ]
}
```

### 6. Forgot Password

The reset token is printed to the server console (simulating email delivery). The API response never contains the token.

```bash
curl -X POST http://localhost:8080/api/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@finance.com"}'
```

***

## Frontend Pages

The frontend is served directly by Spring Boot from `src/main/resources/static/`. No separate build step needed.

| Page | Description |
|------|-------------|
| Login / Register | Authentication with forgot password flow |
| Dashboard | KPI cards, monthly trend chart, category donut, recent activity |
| Records | CRUD table with type/category/date filters, pagination, add/edit modals |
| Analytics | Bar chart comparing income vs expense, category breakdowns |
| Users | Admin-only — edit roles, status, deactivate users |

Built with Vanilla HTML/CSS/JS + Chart.js. Navigation items and action buttons adapt based on the logged-in user's role.

***

## Security Measures

I did a full security review of the codebase and fixed the issues below. Full explanation is in [`SECURITY_FIXES.md`](SECURITY_FIXES.md).

| # | Severity | Fix | OWASP Category |
|---|----------|-----|----------------|
| 1 | **P0** | Registration always assigns VIEWER — prevents privilege escalation | A01: Broken Access Control |
| 2 | **P0** | Password reset token removed from API response | A07: Auth Failures |
| 3 | **P0** | JWT secret externalized via `APP_JWT_SECRET` env var | A02: Cryptographic Failures |
| 4 | **P1** | Rate limiting on auth endpoints (10 req/min per IP) | A07: Auth Failures |
| 5 | **P1** | Dev/prod Spring profiles — H2 console and Swagger off in production | A05: Security Misconfiguration |
| 6 | **P1** | Forgot password returns a generic message to prevent email enumeration | A07: Auth Failures |

***

## Design Decisions & Assumptions

| Decision | Why |
|----------|-----|
| **H2 file-based database** | Zero setup for running locally. The prod profile is ready to swap in PostgreSQL via environment variables. |
| **Single role per user** | Three well-defined roles don't need a many-to-many table. Easy to extend if requirements grow. |
| **Public registration → always VIEWER** | Principle of Least Privilege. Role elevation is an admin-only action via `PUT /api/users/{id}`. |
| **Soft delete on records and users** | Financial data has audit implications — hard deletes are a one-way door. |
| **BigDecimal for monetary amounts** | IEEE 754 floating-point causes rounding errors with money. `BigDecimal(15,2)` is precise. |
| **JWT stateless auth** | No server-side sessions needed. Tokens expire in 24 hours. Scales horizontally. |
| **Password reset token via console logs** | No email server in local dev. The API contract matches production — response never contains the token. |
| **Dashboard cached for 60 seconds** | It's a read-heavy endpoint. Cache evicts automatically on any record write. Up to 60s staleness is acceptable for analytics. |
| **Rate limiting (10/min on auth)** | Prevents brute-force without impacting normal usage. Configurable via `app.rate-limit.max-requests`. |
| **Frontend served from Spring Boot** | Single deployable JAR. For a larger team, the frontend would be a separate React/Next.js app. |

***

## What I Would Do Differently in Production

| Area | This Project | Production Approach |
|------|-------------|---------------------|
| Database | H2 embedded | PostgreSQL on RDS |
| Schema migrations | `ddl-auto: update` | Flyway with versioned SQL scripts |
| Email delivery | Console logging | Spring Mail + SendGrid / SES |
| Auth tokens | In-memory | HttpOnly Secure cookies (SameSite=Strict) |
| Caching | Caffeine (local) | Redis (distributed, shared across instances) |
| Rate limiting | ConcurrentHashMap | Redis-backed (bucket4j) |
| Frontend | Vanilla JS | React / Next.js as separate deployment |
| Monitoring | None | Micrometer + Prometheus + Grafana |
| Audit logging | `createdBy` field | Full audit trail (who changed what, when) |
| API versioning | None | `/api/v1/` prefix with backward compatibility |

***

## Project Structure

```
FinVault/
├── .github/workflows/ci.yml             # CI/CD — build + test + Docker on every push
├── Dockerfile                            # Multi-stage build (tests run inside Docker)
├── pom.xml                               # Maven dependencies
├── render.yaml                           # Render.com deployment config
├── SECURITY_FIXES.md                     # Full security review documentation
├── src/
│   ├── main/
│   │   ├── java/com/finance/
│   │   │   ├── FinanceApplication.java
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java         # Filter chain, RBAC rules
│   │   │   │   ├── JwtAuthFilter.java          # Reads and validates JWT on each request
│   │   │   │   ├── RateLimitFilter.java        # IP-based request throttling
│   │   │   │   ├── PasswordEncoderConfig.java  # BCrypt bean
│   │   │   │   ├── OpenApiConfig.java          # Swagger + JWT auth button
│   │   │   │   ├── WebConfig.java              # Root URL → index.html redirect
│   │   │   │   └── DataSeeder.java             # Seeds 3 default users on startup
│   │   │   ├── controller/
│   │   │   │   ├── AuthController.java         # Register, Login, Forgot/Reset Password
│   │   │   │   ├── FinancialRecordController.java  # CRUD + filtering + pagination
│   │   │   │   ├── DashboardController.java    # Aggregated analytics
│   │   │   │   └── UserController.java         # Admin user management
│   │   │   ├── dto/
│   │   │   │   ├── request/                    # LoginRequest, RegisterRequest, etc.
│   │   │   │   └── response/                   # AuthResponse, DashboardSummary, etc.
│   │   │   ├── entity/
│   │   │   │   ├── User.java                   # Implements UserDetails
│   │   │   │   ├── FinancialRecord.java        # Soft-deletable, indexed
│   │   │   │   └── PasswordResetToken.java     # Single-use, 15-min expiry
│   │   │   ├── enums/                          # Role, RecordType
│   │   │   ├── exception/                      # GlobalExceptionHandler + custom exceptions
│   │   │   ├── repository/                     # JPA repos with JPQL queries
│   │   │   ├── security/
│   │   │   │   └── JwtService.java             # Token generation, validation, secret check
│   │   │   └── service/                        # Business logic — interface + impl pattern
│   │   └── resources/
│   │       ├── application.yml                 # Main config (env-var driven)
│   │       ├── application-prod.yml            # Production overrides
│   │       └── static/                         # FinVault frontend (HTML + CSS + JS)
│   └── test/
│       └── java/com/finance/
│           ├── FinanceApplicationTests.java    # Context load
│           ├── AuthControllerTest.java         # Auth flow (5 tests)
│           └── AccessControlTest.java          # RBAC enforcement (9 tests)
```

***

## Running Tests

```bash
./mvnw test
```

**15 tests** covering:
- Registration: success, duplicate username, validation errors
- Login: success, wrong credentials
- Role-based access for all 3 roles across all endpoint types
- Regression test: sending `"role":"ADMIN"` during registration still results in `VIEWER`

All tests use an H2 in-memory database with elevated rate limits — no external dependencies.

***

## Links

- **Repository:** https://github.com/AzyShu45/FinVault
- **API Docs (local):** http://localhost:8080/swagger-ui.html
- **Security Review:** [`SECURITY_FIXES.md`](SECURITY_FIXES.md)