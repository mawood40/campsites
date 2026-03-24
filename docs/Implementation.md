# Implementation Plan: Campsite Reviews Membership Platform

## Feature Analysis

### Identified Features

| Feature | Description | FRD |
|---------|-------------|-----|
| Authentication & authorization | Signup, login, logout, password hashing, session/token handling, protected routes, role-aware access (subscriber vs admin) | [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md) |
| Campsite listing & detail | Paginated catalog with `review_count` and canonical average summary; detail view with full fields | [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md) |
| Campsite creation & image upload | Create campsite with required fields; multipart upload; storage adapter; validation | [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md) |
| Reviews & ratings | One review per user action with thumbs up/down + comments; list reviews for campsite; transactional aggregate updates | [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md) |
| Admin subscriber removal & audit | Admin removes/deactivates subscribers with confirmation; `AdminAction` audit trail; referential integrity | [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md) |
| Platform bootstrap, config & observability | Repo layout, Docker Compose, config precedence, health/readiness, structured logs, metrics hooks, correlation IDs | [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md) |

### README / Existing Product Features

The root [README.md](../README.md) currently contains only the project title. **No implemented product features are documented yet.** This plan assumes a greenfield implementation aligned with [PRD.md](../PRD.md).

### Feature Categorization

- **Must-Have Features:** Authentication & authorization; campsite listing & detail; campsite creation & image upload; reviews & ratings; admin subscriber removal & audit; platform bootstrap (Compose, migrations, BFF/backend split, edge routing); structured errors; contract + E2E test paths per PRD §17.
- **Should-Have Features:** SSH deployment scripts with immutable image tags; runbooks (startup, deploy, rollback, migration); OpenAPI for BFF + internal backend contract; seed data scripts; optional local S3-compatible storage (e.g. MinIO) behind storage adapter.
- **Nice-to-Have Features:** Contract tests in CI for every PR; Playwright visual regression on key screens; Prometheus scrape endpoints wired in compose for local demos.

### Technical Requirements & Constraints (from PRD)

- **Topology:** React SPA → edge/gateway → FastAPI BFF → FastAPI backend → PostgreSQL, Redis, externalized media storage.
- **SPA may call only the BFF** (stable client contract). Backend is internal; schemas shared via `packages/api-contracts` / `shared-schemas`.
- **No environment-specific code branches:** behavior switches only via `.env`, JSON config under `infra/config/`, and secrets injection.
- **Security:** RBAC enforced in backend (and edge/BFF as trust boundaries); password hashing with modern adaptive algorithm; safe auth error messages; upload validation.
- **AI implementability:** Script-first commands, machine-readable contracts, fail-fast config validation.

### Integrations & Dependencies

- PostgreSQL (system of record), Redis (cache / session or rate limits / ephemeral state — not durable business source of truth), object store or mounted volume for images (adapter pattern), nginx (or equivalent) as edge.

---

## Feature Requirements Documents

Individual FRDs live in `/docs/Features/`:

- [ ] [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md)
- [ ] [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md)
- [ ] [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md)
- [ ] [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md)
- [ ] [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md)
- [ ] [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)

---

## Recommended Tech Stack

Approved versions and packages: **`.cursor/rules/tech-stack.mdc`** (index), **`.cursor/rules/tech-stack-python.mdc`**, **`.cursor/rules/tech-stack-node.mdc`**, **`.cursor/rules/tech-stack-infra.mdc`**.

### Frontend

- **Framework:** React + TypeScript, Vite — SPA routing, forms, API client to BFF only.
- **Documentation:** [React](https://react.dev/), [Vite](https://vitejs.dev/), [React Router](https://reactrouter.com/), [Vitest](https://vitest.dev/), [Playwright](https://playwright.dev/)

### Backend

- **Framework:** FastAPI (BFF + backend services), Pydantic v2, SQLAlchemy 2 / SQLModel, Alembic — per tech-stack-python.
- **Documentation:** [FastAPI](https://fastapi.tiangolo.com/), [Pydantic](https://docs.pydantic.dev/), [SQLAlchemy](https://docs.sqlalchemy.org/), [Alembic](https://alembic.sqlalchemy.org/)

### Datastores & Infra

- **PostgreSQL 16+**, **Redis 7.x**, **Docker / Compose 2.20+**, **nginx 1.27+** as edge.
- **Documentation:** [PostgreSQL](https://www.postgresql.org/docs/), [Redis](https://redis.io/docs/), [Docker Compose](https://docs.docker.com/compose/), [nginx](https://nginx.org/en/docs/)

### Additional Tools

- **HTTP client (SPA):** axios (versions per tech-stack-node).
- **Structured logging (Python):** structlog — see tech-stack-python; align with repo logging standard when added.
- **Metrics:** prometheus-client hooks per PRD §7.8.
- **E2E:** Playwright for acceptance flows in PRD §17.1.

---

## Implementation Stages

### Stage 1: Foundation & Setup

**Duration:** ~2–3 weeks (single engineer; includes test phases)  
**Dependencies:** None

#### Phase 1a: Development

**Goal:** Monorepo skeleton, local orchestration, shared contracts, auth foundation, mocks.

**Sub-steps:**

- [ ] Scaffold PRD directory layout under repo root (`apps/frontend-spa`, `apps/bff-service`, `apps/backend-service`, `packages/api-contracts`, `infra/`, `scripts/`) — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Add root `.env.example`, document config precedence (defaults → JSON base → env JSON → `.env` → secrets) — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Dockerfiles for SPA (build stage + nginx static or serve via gateway), BFF, backend; Compose stack: gateway, BFF, backend, Postgres, Redis, storage volume or MinIO — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] PostgreSQL migrations tooling (Alembic) and initial schema for `users`, `campsites`, `reviews`, `admin_actions` — **FRDs:** [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md), [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md), [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md), [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md)
- [ ] Backend: settings validation (fail fast), structured logging, request/correlation ID middleware — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] BFF: routing to backend via httpx client; internal error normalization for SPA — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Implement authentication domain (signup, login, logout, session/JWT decision documented in Architecture) — **FRD:** [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md)
- [ ] **Mock services:** mock backend responses for BFF unit tests; WireMock or respx fixtures for BFF→backend — enable CI without full stack where practical — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)

**Deliverables:** `docker compose up` runs gateway + BFF + backend + DB + Redis; health endpoints respond; one admin + seed subscribers scriptable.

#### Phase 1b: Testing

**Goal:** Validate infrastructure and security baselines.

**Sub-steps:**

- [ ] Smoke tests: each service liveness/readiness — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Validate configuration loading and failure on missing required vars — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Integration test: DB connectivity, migration apply on clean DB — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Auth integration tests: signup, duplicate email, login failure messages safe — **FRD:** [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md)
- [ ] Document local developer workflow in README (bootstrap, migrate, test, lint) — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)

**Exit Criteria:** Stack starts cleanly; health checks pass; migrations reproducible; auth happy path proven in tests.

---

### Stage 2: Core Features

**Duration:** ~3–4 weeks  
**Dependencies:** Stage 1 complete (both phases)

#### Phase 2a: Development

**Goal:** Subscriber-facing catalog, create campsite + upload, reviews, SPA integration.

**Sub-steps:**

- [ ] Backend + BFF: campsite list (pagination) with derived `review_count` and `percent_positive` (canonical aggregate) — **FRDs:** [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md), [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md)
- [ ] Backend + BFF: campsite detail by id — **FRD:** [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md)
- [ ] Backend + BFF: create campsite + storage adapter upload pipeline — **FRD:** [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md)
- [ ] Backend + BFF: submit review (binary rating + comments); enforce one review per user per campsite (per PRD intent) — **FRD:** [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md)
- [ ] Backend + BFF: list reviews for campsite (newest first default) — **FRD:** [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md)
- [ ] SPA: login/signup pages, authenticated layout, campsite grid, detail, add form, review UI — **FRDs:** [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md), [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md), [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md), [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md); **UI:** [UI_UX_doc.md](UI_UX_doc.md)
- [ ] Expand mocks for campsite and review payloads for BFF tests — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)

**Deliverables:** End-to-end manual flow: signup → list → create campsite with image → submit review → see updated aggregates.

#### Phase 2b: Testing

**Goal:** Core feature validation with automated coverage.

**Sub-steps:**

- [ ] Unit tests: validation schemas, aggregate math, RBAC guards — **FRDs:** [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md), [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md)
- [ ] Integration tests: API handlers + real DB (test container or compose profile) — all BFF-facing behaviors — **FRD:** [API.md](API.md)
- [ ] SPA component tests (Vitest + Testing Library) for forms and error states — **FRD:** [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md)
- [ ] Playwright E2E: PRD §17.1 subscriber flows (signup, login, add campsite, review, list shows aggregates, review history) — **FRD:** [FRD_ReviewsAndRatings.md](Features/FRD_ReviewsAndRatings.md)
- [ ] Document test scenarios in `docs/` or `tests/README.md`

**Exit Criteria:** E2E acceptance subset green locally and in CI; no critical open defects on core flows.

---

### Stage 3: Advanced Features

**Duration:** ~2 weeks  
**Dependencies:** Stage 2 complete (both phases)

#### Phase 3a: Development

**Goal:** Admin flows, audit, contract hardening, performance-related patterns.

**Sub-steps:**

- [ ] Backend + BFF: admin list subscribers (minimal fields for management UI) and remove/deactivate with configurable policy — **FRD:** [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md)
- [ ] Persist `AdminAction` records for removals — **FRD:** [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md)
- [ ] SPA: admin subscriber management + confirmation modal — **FRD:** [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md); **UI:** [UI_UX_doc.md](UI_UX_doc.md)
- [ ] OpenAPI export for BFF; internal backend OpenAPI or schema package for contract tests — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Redis: caching for read-heavy list/detail where safe (invalidation on write) — **FRD:** [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md)
- [ ] Extend mocks for admin endpoints — **FRD:** [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md)

**Deliverables:** Admin acceptance criteria from PRD §17.1 satisfied; contract artifacts generated in CI.

#### Phase 3b: Testing

**Goal:** E2E admin path, security checks, contract and performance smoke.

**Sub-steps:**

- [ ] Playwright: admin removes subscriber; verify audited action and subscriber cannot log in (if deactivated) — **FRD:** [FRD_AdminSubscriberRemovalAndAudit.md](Features/FRD_AdminSubscriberRemovalAndAudit.md)
- [ ] Automated checks: subscriber denied admin APIs (403) — **FRD:** [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md)
- [ ] Contract tests: BFF request/response shapes vs OpenAPI — **FRD:** [API.md](API.md)
- [ ] Performance smoke: p95 targets documented for list/detail under seed load (optional k6 or locust stub) — **FRD:** [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md)
- [ ] Security review checklist: upload limits, CSRF/session cookie settings, headers — **FRDs:** [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md), [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md)

**Exit Criteria:** Admin + audit E2E green; contract tests in CI; documented p95 smoke results.

---

### Stage 4: Polish & Production Readiness

**Duration:** ~2 weeks  
**Dependencies:** Stage 3 complete (both phases)

#### Phase 4a: Development

**Goal:** SSH deploy scripts, runbooks, UX polish, metrics endpoints, CI pipeline.

**Sub-steps:**

- [ ] `infra/ssh-deploy/` scripts: copy config, pull immutable tags, migrate, restart, rollback path — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] Runbooks: startup, deploy, rollback, migration failure, incident — under `docs/runbooks/` (see PRD §18.4)
- [ ] UI polish: validation messages, loading states, accessible labels — **FRD:** [UI_UX_doc.md](UI_UX_doc.md)
- [ ] Metrics endpoints / hooks wired per service — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] GitHub Actions (or chosen CI): lint, typecheck, unit, integration, contract, E2E — **FRD:** [FRD_PlatformBootstrapObservability.md](Features/FRD_PlatformBootstrapObservability.md)
- [ ] README: exact commands for setup, test, build, migrate, deploy — PRD §19.1

**Deliverables:** Production checklist satisfied per PRD §21; images promotable without code change.

#### Phase 4b: Testing

**Goal:** Production readiness validation.

**Sub-steps:**

- [ ] Full automated suite green on main; coverage thresholds enforced (tune to project reality) — **Implementation** test philosophy
- [ ] Load test optional: campsite list under concurrent users — **FRD:** [FRD_CampsiteListingAndDetail.md](Features/FRD_CampsiteListingAndDetail.md)
- [ ] Dry-run deploy to staging-like host using same images as CI build
- [ ] Documentation review: API.md, Architecture.md, Implementation.md aligned with code
- [ ] Security pass: dependency scan, secret scan, upload abuse cases — **FRD:** [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md)

**Exit Criteria:** PRD release acceptance criteria §21 met; runbooks and README validated by independent walkthrough.

---

## Testing Infrastructure Requirements

### Mock Services to Create

- [ ] BFF→backend: respx/httpx mocks or test container for backend in BFF tests
- [ ] Object storage: local filesystem adapter in tests; optional MinIO for integration tests — **FRD:** [FRD_CampsiteCreationAndImageUpload.md](Features/FRD_CampsiteCreationAndImageUpload.md)
- [ ] Email provider: stub when verification is enabled (no real SMTP in unit tests)
- [ ] Test data factories/fixtures for users, campsites, reviews (deterministic seeds)

### Testing Tools and Frameworks

- **Unit:** pytest, Vitest — [pytest](https://docs.pytest.org/), [Vitest](https://vitest.dev/)
- **Integration:** pytest + httpx AsyncClient; test DB — [FastAPI testing](https://fastapi.tiangolo.com/tutorial/testing/)
- **Contract:** schemathesis or openapi-spec-validator (choose one; document in repo)
- **E2E:** Playwright — [Playwright](https://playwright.dev/)
- **Mocking:** pytest-mock, respx — tech-stack-python

### Testing Documentation

- [ ] Testing strategy (layering, what runs in CI vs local)
- [ ] Test environment setup (compose profile `test`)
- [ ] Mock service / fixture documentation
- [ ] Per-stage checklist (mirror Implementation stages)
- [ ] Performance benchmark placeholders (fill after Stage 3)

---

## Resource Links

- [FastAPI](https://fastapi.tiangolo.com/)
- [React](https://react.dev/)
- [Vite](https://vitejs.dev/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Redis Documentation](https://redis.io/docs/)
- [Docker Compose](https://docs.docker.com/compose/)
- [nginx documentation](https://nginx.org/en/docs/)
- [Playwright](https://playwright.dev/)
- [Alembic](https://alembic.sqlalchemy.org/)

---

## Roles & Estimates Note

**Default assumption:** 1 full-stack engineer familiar with React and Python; adjust durations for team parallelism. Estimates include development and embedded testing phases per stage.

**Dependencies between tasks:** Authentication before protected campsite/review routes; schema before features; BFF client before SPA integration; admin APIs after core RBAC proven.
