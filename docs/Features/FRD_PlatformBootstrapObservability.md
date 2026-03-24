# FRD: Platform Bootstrap, Configuration, and Observability

## Feature Overview

**Purpose:** Provide the **repository skeleton**, **local Docker orchestration**, **deterministic configuration**, **health/readiness**, **structured logging**, **metrics hooks**, and **correlation IDs** across edge, BFF, and backend so the product meets PRD §7, §12–13, §18, §19 and is AI-implementable.

**Scope:** Compose stacks, `.env.example`, JSON config precedence, startup validation, `/health/*`, logging/metrics conventions, CI script entrypoints, OpenAPI artifact generation. **Out of scope:** Full Grafana/Loki deployment (optional local compose profile).

**Business Value:** Safe operations, debuggable failures, promotable artifacts across environments.

**Priority:** Must-Have

## User stories

- **As a developer**, I want one command to run all services locally so that I can implement features without manual wiring.
- **As an operator**, I want health checks so that orchestrators can restart bad instances.
- **As an incident responder**, I want request IDs in logs so that I can trace a user report across services.

### Edge cases

- Missing required env var → process **exits non-zero** before binding port (fail fast).
- Migration version mismatch on **ready** check → `/health/ready` returns **503** until migrated.

## Functional requirements

1. **Config precedence:** defaults → `infra/config/base/*.json` → `infra/config/environments/<env>/*.json` → environment variables → injected secrets (document exact merge in README).
2. **No secrets in JSON** committed to git; only placeholders in `.env.example`.
3. **docker compose** profile brings up: gateway, SPA static (or build target), BFF, backend, postgres, redis, storage volume (and optional MinIO).
4. Each service exposes **GET /health/live** (always 200 if process up) and **GET /health/ready** (DB + Redis + storage reachability as applicable).
5. **Structured logs:** JSON or key=value with fields: `timestamp`, `level`, `service`, `message`, `request_id`, `user_id` (when authenticated).
6. **Metrics:** prometheus-client registry exposed on configurable port or `/metrics` (document port conflicts with BFF).
7. **Scripts:** `scripts/local/up.sh`, `scripts/local/migrate.sh`, `scripts/local/seed.sh`, `scripts/ci/test.sh` (exact names flexible but must exist as **documented** commands).

## Technical requirements

- Versions: `.cursor/rules/tech-stack-infra.mdc`, `tech-stack-python.mdc`, `tech-stack-node.mdc`.
- nginx gateway routes `/api/` to BFF, `/media/` to storage or backend static handler, `/` to SPA.

### Dependencies

- None upstream; all feature FRDs depend on this.

## API specifications

Health endpoints are **not** part of the SPA product API but must be documented for operators:

| Method | Path | Service | Auth |
|--------|------|---------|------|
| GET | /health/live | BFF, backend | No |
| GET | /health/ready | BFF, backend | No |

Internal metrics scrape path TBD (e.g. `:9100/metrics` sidecar pattern) — document chosen approach in README.

**Example:**

```bash
curl -sf "http://localhost:8001/health/ready" && echo OK
```

## Data models

N/A for platform FRD (operational only). Migrations owned by backend service.

### Data flow (ASCII)

```
Request->Edge--X-Request-ID-->BFF--forward ID-->Backend--log JSON-->
```

## Security requirements

- Debug endpoints must not leak secrets; `/metrics` bind to internal network in production.
- Rate limiting at edge per PRD §5.2.

## Performance criteria

- Ready check completes < **2 s** when dependencies healthy.
- Local `compose up` reaches ready state < **60 s** on typical laptop after images pulled.

## Testing requirements

### Integration

- CI job: `docker compose -f infra/compose/test.yml up --wait` then run pytest + vitest + contract suite.

### Acceptance

- PRD §21: same images used in local and prod promotion story documented; config validation test fails CI when required keys missing.

## Implementation notes

- Add `docs/runbooks/` stubs: `startup.md`, `deploy.md`, `rollback.md`, `migrations.md` during Stage 4 of [Implementation.md](../Implementation.md).
- Generate OpenAPI from FastAPI apps into `packages/api-contracts/dist/` in CI.
