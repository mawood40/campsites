# System Architecture: Campsite Reviews Membership Platform

## Overview

The product is a membership web application: subscribers authenticate, browse a shared catalogue of campsites (with community ratings), add new campsites with a primary photo, and submit thumbs-up or thumbs-down reviews with comments. Administrators retain all subscriber capabilities and may remove or deactivate subscriber accounts in a confirmed, audited way.

The runtime follows a **three-tier application pattern** aligned with the PRD: a **React SPA** talks only to a **FastAPI BFF** behind an **edge reverse proxy**. The BFF shapes responses for the UI and forwards domain operations to a **FastAPI backend** that owns business rules, authorization, and persistence. **PostgreSQL** holds durable entities; **Redis** supports cache, rate limiting, session or ephemeral coordination, and must not replace PostgreSQL as the system of record. **Image binaries** live outside container writable layers via a **storage adapter** (local filesystem, mounted volume, or S3-compatible API), selected by configuration.

The same container images and application code are promoted across environments; only `.env`, JSON configuration, secrets, and infrastructure endpoints differ.

## Architecture Diagram

```
                    (TLS)                           internal net
Browser (React)      |         Edge (nginx)         |    BFF (FastAPI)
+-------------+     |     +------------------+       |    +------------------+
|  SPA static |     |     | TLS, routes,     |       |    | Client DTOs,     |
|  + JS app   |---->|---->| rate limit,      |------>|--->| upload broker,   |
+-------------+     |     | coarse auth opt  |       |    | error normalize  |
                    |     +------------------+       |    +------------------+
                    |              |                 |            |
                    |              v                 |            v
                    |     +------------------+       |    +------------------+
                    |     | Backend (FastAPI)|<------|----| httpx / internal   |
                    |     | domain, RBAC,    |       |    | API + schemas      |
                    |     | aggregates       |       |    +------------------+
                    |     +---------+--------+       |
                    |               |                |
                    |        +------v------+   +-----v-----+
                    |        | PostgreSQL  |   | Redis     |
                    |        | (SoT)     |   | cache/sess|
                    |        +-----------+   +-----------+
                    |               |
                    |        +------v------+
                    |        | Media store|
                    |        | (adapter)  |
                    |        +------------+
```

## Components

### SPA (`apps/frontend-spa`)

- **Technology:** React, TypeScript, Vite, React Router — versions per `.cursor/rules/tech-stack-node.mdc`.
- **Responsibility:** Login/signup views, authenticated routing, campsite list/detail, create form with upload, review UI, admin subscriber UI (role-gated). Calls **only** the BFF base URL.
- **Communicates with:** Edge (same origin or configured API host) using cookies or `Authorization` as defined in [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md).

### Edge / API Gateway (`infra/gateway` or compose service)

- **Technology:** nginx (1.27+ alpine per tech-stack-infra).
- **Responsibility:** TLS termination (deployed envs), routing to SPA static assets and BFF upstream, rate limiting, optional static asset caching. Coarse auth boundary as required by PRD (e.g. reject unauthenticated paths at edge only when compatible with chosen session model).
- **Communicates with:** SPA (static), BFF (HTTP upstream).

### BFF (`apps/bff-service`)

- **Technology:** FastAPI, httpx, Pydantic — per tech-stack-python.
- **Responsibility:** Stable **versioned JSON API** for the SPA; request aggregation if needed; multipart upload brokering; mapping backend models to client DTOs; propagating identity context to backend via trusted internal mechanism (header or network trust); normalizing errors to machine-readable SPA payloads.
- **Communicates with:** Backend over private network; must not expose backend URLs to the browser.

### Backend (`apps/backend-service`)

- **Technology:** FastAPI, SQLAlchemy/SQLModel, Alembic, redis-py, structlog — per tech-stack-python.
- **Responsibility:** Authoritative **RBAC**, validation, transactions, review aggregates, user lifecycle (including soft deactivate), audit rows, structured logging with correlation IDs, metrics hooks, health/readiness.
- **Communicates with:** PostgreSQL, Redis, storage adapter.

### PostgreSQL

- **Technology:** PostgreSQL 16+ (alpine image in dev per tech-stack-infra).
- **Responsibility:** Users, campsites, reviews, admin actions; foreign keys; migrations versioned in repo.

### Redis

- **Technology:** Redis 7.x.
- **Responsibility:** Cache for list/detail (optional), rate limit counters, session/token backing if chosen, short-lived job state. **Not** durable source of truth for business entities.

### Media storage

- **Technology:** Pluggable adapter (local path, volume mount, or S3-compatible SDK).
- **Responsibility:** Persist campsite images; return URL or key stored in `campsites.image_url` (or equivalent).

## Data Flow

### Authenticated read: campsite list

```
SPA          Edge        BFF           Backend        PostgreSQL     Redis
 |            |           |               |               |            |
 |-- GET ---->|---------->|-- GET list -->|               |            |
 |            |           |               |-- query ----->|            |
 |            |           |               |<- rows -------|            |
 |            |           |               |-- cache opt->|            |
 |            |           |<- JSON -------|<--------------|            |
 |<- JSON ----|<----------|<--------------|               |            |
```

### Write: create review (aggregate updated in same transaction)

```
SPA     BFF      Backend    PostgreSQL
 |       |          |            |
 | POST review      |            |
 |------>|--------->|            |
 |       |          | BEGIN      |
 |       |          | INSERT rev |
 |       |          | UPDATE agg |
 |       |          | COMMIT     |
 |       |<---------|            |
 |<------|          |            |
```

## External Integrations

- **SMTP or transactional email provider (optional):** Used only if `auth.json` enables email verification or signup confirmation; must be mockable in tests.
- **Object storage (deployed):** S3-compatible or server-local volume; no vendor lock-in in domain code — adapter only.

## Key Design Decisions

1. **SPA ↔ BFF only:** The browser never calls the backend service URL; contracts are versioned (`/api/v1/...`) and documented in [API.md](API.md).
2. **Backend is authorization authority:** Edge/BFF may assist with routing and ergonomics; **role and ownership checks** execute in backend handlers before mutations.
3. **Resolved open items from PRD §20 (defaults for implementation):**
   - **Session model:** Server-side session stored in **Redis** with **HTTP-only, Secure, SameSite** session cookie issued by BFF after backend validates credentials (alternative: JWT access token documented in FRD if team switches — pick one and delete ambiguity in code).
   - **Image persistence:** **Filesystem adapter** for local dev; **S3-compatible or mounted volume** for deployment via `storage.json`.
   - **Canonical rating summary:** **`percent_positive`** (0–100 integer) plus **`review_count`** on list/detail DTOs.
   - **Subscriber removal:** **Soft deactivate** by default (`status = inactive`); hard delete behind config flag if explicitly enabled.
   - **Email verification:** **Configurable** — `auth.json` flag `require_email_verification`; when false, user may use account immediately after signup.
4. **Idempotency:** Review creation should be idempotent per `(user_id, campsite_id)` if PRD “exactly one rating payload per review action” is interpreted as one review per user per campsite — enforce with unique constraint and return 409 or existing review per API.md.
5. **Observability:** Request ID middleware propagates `X-Request-ID` from edge through BFF to backend; structured JSON logs; `/health/live` and `/health/ready` on BFF and backend.

## Related Documentation

- [Implementation.md](Implementation.md) — staged delivery plan.
- [API.md](API.md) — BFF contract.
- [Project_structure.md](Project_structure.md) — repository layout.
- Feature FRDs under [Features/](Features/).
