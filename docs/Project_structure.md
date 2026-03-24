# Project Structure

This document mirrors the **required repository layout** from [PRD.md](../PRD.md) §11 and refines it for implementers. Paths are relative to the repository root.

## Directory Tree (Target State)

```
repo-root/
├── apps/
│   ├── frontend-spa/
│   │   ├── src/
│   │   ├── public/
│   │   ├── tests/
│   │   └── package.json
│   ├── bff-service/
│   │   ├── app/
│   │   ├── tests/
│   │   └── pyproject.toml
│   └── backend-service/
│       ├── app/
│       ├── tests/
│       └── pyproject.toml
├── packages/
│   ├── api-contracts/          # OpenAPI / JSON Schema shared artifacts
│   ├── shared-schemas/         # Pydantic models shared BFF<->backend (private)
│   └── shared-config/          # Typed config loaders if needed
├── infra/
│   ├── docker/
│   ├── compose/              # docker-compose*.yml profiles: dev, test
│   ├── ssh-deploy/
│   ├── gateway/
│   ├── db/
│   └── config/               # Policy JSON only; no secrets (PRD §11)
│       ├── base/
│       │   ├── app.json
│       │   ├── auth.json
│       │   └── storage.json
│       └── environments/
│           ├── dev/
│           ├── test/
│           └── prod/
├── scripts/
│   ├── local/                # bootstrap, seed, dev helpers
│   ├── ci/                   # CI entrypoints
│   └── deploy/               # Wrappers calling infra/ssh-deploy
├── docs/
│   ├── Implementation.md
│   ├── Architecture.md
│   ├── API.md
│   ├── Project_structure.md  # (this file)
│   ├── UI_UX_doc.md
│   ├── Bug_tracking.md
│   ├── Features/
│   ├── prd/                  # Optional: linked copies or index to root PRD
│   ├── runbooks/
│   └── ai-implementation-guides/
├── migrations/               # Optional top-level alias; prefer Alembic inside backend app
├── .env.example
├── README.md
└── PRD.md
```

**Note:** All policy JSON lives under **`infra/config/`** per PRD; secrets only via `.env` / deployment stores.

## Structure Rules

1. **Deployable units:** `frontend-spa`, `bff-service`, and `backend-service` build to **separate Docker images** and may deploy to different hosts.
2. **No copy-paste schemas:** Shared types and OpenAPI fragments live under `packages/`; services depend on them as local path dependencies.
3. **Infrastructure as code:** Everything under `infra/` and `scripts/` must be executable in CI and by automation without interactive prompts (secrets excepted).
4. **Configuration:** Feature flags, storage adapter selection, auth policy, and environment endpoints live in JSON under config trees; **secrets never committed**.

## Discoverability for AI / New Developers

| Need | Where to look |
|------|----------------|
| Product scope | [PRD.md](../PRD.md) |
| Delivery plan | [Implementation.md](Implementation.md) |
| BFF endpoints | [API.md](API.md) |
| Runtime topology | [Architecture.md](Architecture.md) |
| Screen flows | [UI_UX_doc.md](UI_UX_doc.md) |
| Feature depth | [Features/](Features/) |
| Local commands | [README.md](../README.md) (to be expanded) |

## Migrations

Alembic revision files should live with the **backend** service (e.g. `apps/backend-service/alembic/`) with a single migration chain for the application database. CI and deploy scripts invoke the same `alembic upgrade head` against the target DSN.
