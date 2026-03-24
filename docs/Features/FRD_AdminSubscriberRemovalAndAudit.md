# FRD: Admin Subscriber Removal and Audit

## Feature Overview

**Purpose:** **Admin** users remove or deactivate **subscriber** accounts with explicit confirmation; action is **auditable**; campsite and review data remain referentially intact per PRD §3.2, §6.11–6.12, §16.1–16.2.

**Scope:** Soft deactivate default; optional hard delete behind config; prevent self-lockout and protecting other admins. **Out of scope:** Bulk ban, reason codes beyond free-text, GDPR export.

**Business Value:** Operational control over abusive accounts without losing community content history.

**Priority:** Must-Have

## User stories

- **As an admin**, I want to see subscribers in a management view so that I can choose whom to deactivate.
- **As an admin**, I want a confirmation step so that I do not remove accounts accidentally.
- **As compliance**, I want each removal logged with actor, target, timestamp, and result.

### Edge cases

- Target already inactive → idempotent **200** or **409** (document choice; recommend **200** with `status: inactive`).
- Admin targets another admin → **403 FORBIDDEN**.
- Admin targets self → **400** or **403**.

## Functional requirements

1. **GET /api/v1/admin/subscribers** — admin only; returns non-admin users or all subscribers per policy (recommend **subscribers only**, never list other admins’ emails broadly if privacy-sensitive — product may adjust).
2. **POST /api/v1/admin/subscribers/{user_id}/deactivate** — body includes `confirm: true` and optional `reason`.
3. On success: set user `status = inactive` (soft delete); sessions for that user revoked.
4. Insert **`admin_actions`** row: `actor_user_id`, `target_user_id`, `action_type` (e.g. `subscriber_deactivate`), `reason`, `result` (`success`/`failure`), `created_at`.
5. Campsites and reviews **remain**; display names on old reviews may show “Former member” if configured in `app.json`.

## Technical requirements

- Backend-only enforcement of admin role; BFF forwards and maps errors.
- Audit table append-only from application perspective.

### Dependencies

- Authentication; `users`, `admin_actions` tables.

## API specifications

See [API.md](../API.md) §Admin endpoints.

**POST deactivate example:**

```bash
curl -X POST "http://localhost:8080/api/v1/admin/subscribers/usr_01hq.../deactivate" \
  -b admin_cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"reason":"Terms violation","confirm":true}'
```

**Response `200`:**

```json
{
  "target_user_id": "usr_01hq...",
  "status": "inactive",
  "admin_action_id": "adm_01hq..."
}
```

**403 example:**

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Admin action not allowed for this target",
    "details": null
  }
}
```

## Data models

### `admin_actions`

| Column | Type |
|--------|------|
| id | UUID PK |
| actor_user_id | FK users |
| target_user_id | FK users |
| action_type | text |
| reason | text nullable |
| result | text / enum |
| created_at | timestamptz |

### Data flow (ASCII)

```
SPA--POST/deact-->BFF-->Backend--check admin role
                         |
                         +-->UPDATE user status
                         +-->INSERT admin_actions
                         +-->DELETE redis sessions(user)
```

## Security requirements

- **403** for subscribers hitting admin routes.
- Audit log must not contain passwords; reason text length-capped.

## Performance criteria

- Deactivate operation p95 < **300 ms**; session purge best-effort async acceptable if documented.

## Testing requirements

### Integration

- Admin deactivates subscriber; subscriber login fails; audits row exists.
- Subscriber gets **403** on admin API.

### Acceptance

- PRD §17.1: admin removal audited (E2E assert DB or API audit tail if exposed to tests).

## Implementation notes

- `infra/config/base/app.json`: `subscriber_removal_mode`: `soft` | `hard` (hard only if legal/product approves).
