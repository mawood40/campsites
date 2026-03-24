# FRD: Authentication and Authorization

## Feature Overview

**Purpose:** Allow users to sign up, log in, and log out; issue and validate sessions; enforce **subscriber** vs **admin** roles on every protected operation. Satisfies PRD §3, §6.1–6.3, §15.1–15.2.

**Scope:** Email + password credentials, password hashing, configurable signup confirmation / email verification, safe authentication error messages, SPA protected routes backed by server session validation. **Out of scope:** OAuth social login, passwordless magic links (unless added later), payment.

**Business Value:** Membership boundary for catalogue and reviews; admins can manage subscribers without sharing elevated credentials insecurely.

**Priority:** Must-Have

## User Stories

### Primary use cases

- **As a visitor**, I want to create an account with email and password so that I can access campsite content.
- **As a subscriber**, I want to log in and stay authenticated across SPA navigations so that I can use the app without re-entering credentials constantly.
- **As a subscriber**, I want to log out so that my session ends on shared devices.
- **As an admin**, I want the system to recognize my role so that I can open subscriber management features.
- **As the product**, we must reject protected API calls without a valid session and return **403** when the user is authenticated but not authorized.

### Edge cases

- Duplicate signup email → **409** or **422** with safe, non-enumerating messaging strategy documented in API.
- Login with wrong password → **401** generic failure.
- Deactivated subscriber attempts login → **401** or **403** with policy-defined message.
- Admin attempts to use subscriber-only rate limits vs stricter admin audit — same session model.

## Functional requirements

### Core functionality

1. Signup persists `User` with `role` default `subscriber`, `status` active or pending per `auth.json`.
2. Passwords stored **only** as salted hash (e.g. Argon2 or bcrypt) — never plaintext or reversible encoding.
3. Login creates **server-side session** in Redis (session id in HTTP-only cookie) per [Architecture.md](../Architecture.md); TTL configurable.
4. Logout deletes server session and clears cookie.
5. Backend validates session on each mutation and sensitive read; BFF forwards session cookie or internal token per internal contract.
6. **Authorization:** `admin` routes return **403** for `subscriber`; subscribers cannot call admin APIs even by URL guessing.

### Input / output specifications

- **Inputs:** email (RFC-like validation), password (rules from `auth.json`), optional `display_name`.
- **Outputs:** User DTO without password hash; optional `Set-Cookie`.
- **Validation:** Central Pydantic models in `packages/shared-schemas`; shared between BFF and backend where possible.

## Technical requirements

### Implementation constraints

- Python 3.13+ / FastAPI / Redis / structlog per `.cursor/rules/tech-stack-python.mdc`.
- SPA: React + Vite; store **no long-lived tokens in localStorage** if using cookie sessions (preferred).

### Dependencies

- **Internal:** Platform health, Redis, PostgreSQL `users` table.
- **External:** Optional SMTP for verification emails.

## API specifications

### BFF (SPA-facing)

Full request/response shapes: [API.md](../API.md) §Authentication and error envelope.

**POST /api/v1/auth/signup** — body: `{ "email", "password", "display_name" }`; response `201` + user DTO.

**POST /api/v1/auth/login** — body: `{ "email", "password" }`; response `200` + `Set-Cookie`.

**POST /api/v1/auth/logout** — session required; `204`.

**GET /api/v1/auth/me** — session required; `200` user DTO.

**Example (`curl` login):**

```bash
curl -X POST "http://localhost:8080/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"email":"user@example.com","password":"Str0ng!pass"}'
```

### Internal (backend only; not for browser)

Document in `packages/api-contracts` — illustrative only:

- `POST /internal/v1/auth/validate-session` — BFF passes session id; backend returns user id + role or 401.

## Data models

### Database (`users`)

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| email | unique text | citext optional |
| password_hash | text | |
| role | enum | `subscriber`, `admin` |
| status | enum | `active`, `inactive`, `pending_verification` |
| display_name | text | |
| created_at | timestamptz | |

### Data flow (ASCII)

```
SPA--POST/login-->BFF--internal-->Backend--verify-->PostgreSQL
  |                  |               |
  |                  |               +--session-->Redis
  |                  +--Set-Cookie---+
  +<-----200---------+
```

## Security requirements

### Authentication

- HTTPS in production; **Secure** cookies; **SameSite** appropriate for SPA host alignment.
- Rate limit login and signup per IP + account lockout policy optional via `auth.json`.

### Authorization

- Backend decorators / dependencies: `require_user`, `require_admin`.
- Never trust client-sent `role` field.

### Data protection

- Logs must not contain passwords or session ids in full; redact email in public logs if policy requires.

## Performance criteria

- Login p95 < **300 ms** excluding network under dev seed data; session validation p95 < **50 ms** with Redis warm.

## Testing requirements

### Unit

- Password hashing and verification; JWT/session parsing if applicable.

### Integration

- Signup → login → `me`; duplicate email; wrong password message safe; admin vs subscriber 403 on admin route.

### Acceptance

- PRD §17.1: unauthenticated users cannot access protected SPA routes (E2E); admin APIs reject subscribers.

## Implementation notes

### Development approach

- Implement backend auth domain first; BFF thin mapping; SPA `credentials: "include"` on fetch.

### Potential challenges

- CORS + cookies: BFF and SPA origins must be configured explicitly in dev and prod.

### Configuration

- `infra/config/base/auth.json`: `require_email_verification`, `password_min_length`, `session_ttl_seconds`, `cookie_name`.
