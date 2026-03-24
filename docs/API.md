# API Reference — BFF (SPA-Facing)

## Overview

- **Base URL (local):** `http://localhost:<BFF_PORT>/api/v1` (exact port from `.env` / compose).
- **API version:** `v1` (path prefix).
- **Style:** JSON over HTTPS in production; structured error envelope on 4xx/5xx.
- **Authentication:** Session **cookie** (`Set-Cookie` on login) named per `auth.json` (e.g. `session_id`) — **HTTP-only, Secure, SameSite=Lax** in production. CSRF protection strategy (double-submit cookie or SameSite-only) must match SPA deployment; document final choice in backend security notes. Some diagnostic tools may use `Cookie:` header instead of `Authorization`.

The SPA **must not** call the backend service directly. Internal backend routes are documented only in `packages/api-contracts` / team wiki; this file is the **stable BFF contract**.

**Related:** [Architecture.md](Architecture.md), feature FRDs under [Features/](Features/).

## Authentication

### Sign up

Creates a subscriber account (role `subscriber`). Email must be unique.

```bash
curl -X POST "http://localhost:8080/api/v1/auth/signup" \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"Str0ng!pass","display_name":"River Camper"}'
```

```python
import requests

r = requests.post(
    "http://localhost:8080/api/v1/auth/signup",
    json={
        "email": "user@example.com",
        "password": "Str0ng!pass",
        "display_name": "River Camper",
    },
)
data = r.json()
```

```javascript
const r = await fetch("http://localhost:8080/api/v1/auth/signup", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    email: "user@example.com",
    password: "Str0ng!pass",
    display_name: "River Camper",
  }),
});
const data = await r.json();
```

**Response `201 Created`:**

```json
{
  "user": {
    "id": "usr_01hq...",
    "email": "user@example.com",
    "role": "subscriber",
    "display_name": "River Camper",
    "status": "active"
  }
}
```

`Set-Cookie` may be omitted if `require_email_verification` is true and login is blocked until verified.

### Log in

```bash
curl -X POST "http://localhost:8080/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"email":"user@example.com","password":"Str0ng!pass"}'
```

**Response `200 OK`:** user object as above; includes `Set-Cookie` for session when login succeeds.

### Log out

```bash
curl -X POST "http://localhost:8080/api/v1/auth/logout" \
  -b cookies.txt
```

**Response `204 No Content`** — session invalidated server-side and cookie cleared.

### Current user

```bash
curl -X GET "http://localhost:8080/api/v1/auth/me" -b cookies.txt
```

**Response `200 OK`:** same user shape as signup. **`401`** if not authenticated.

---

## Endpoints

### GET /api/v1/campsites

**Description:** Paginated campsite catalogue for authenticated users. Each item includes derived `review_count` and `percent_positive` (0–100).

**Authentication:** Required (session cookie).

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| limit | integer | No | Page size (default 20, max 100) |
| cursor | string | No | Opaque pagination cursor |

**Response `200 OK`:**

```json
{
  "data": [
    {
      "id": "csp_01hq...",
      "name": "Pine Ridge Camp",
      "location": "VT, USA",
      "pricing": "$35/night",
      "general_info": "Quiet sites near the brook.",
      "image_url": "/media/csp_01hq.../primary.jpg",
      "review_count": 12,
      "percent_positive": 75,
      "created_at": "2026-03-01T12:00:00Z"
    }
  ],
  "meta": {
    "next_cursor": "eyJpZCI6...",
    "limit": 20
  }
}
```

```bash
curl -G "http://localhost:8080/api/v1/campsites" \
  -b cookies.txt \
  --data-urlencode "limit=10"
```

```python
r = requests.get(
    "http://localhost:8080/api/v1/campsites",
    cookies=cookies,
    params={"limit": 10},
)
payload = r.json()
```

```javascript
const u = new URL("/api/v1/campsites", window.location.origin);
u.searchParams.set("limit", "10");
const r = await fetch(u, { credentials: "include" });
const payload = await r.json();
```

---

### GET /api/v1/campsites/{campsite_id}

**Description:** Full campsite detail and aggregate summary.

**Authentication:** Required.

**Path parameters:** `campsite_id` (string).

**Response `200 OK`:** single campsite object (same fields as list plus optional `created_by_display` if policy allows).

**Errors:** `404` `NOT_FOUND`.

---

### POST /api/v1/campsites

**Description:** Create a campsite with **multipart** body: JSON or text fields plus one image file.

**Authentication:** Required (subscriber or admin).

**Content-Type:** `multipart/form-data`

**Parts:**

| Part | Type | Required |
|------|------|----------|
| payload | application/json | Yes — `{"name","location","pricing","general_info"}` |
| image | file | Yes — `image/jpeg`, `image/png`, or `image/webp` per `storage.json` |

**Response `201 Created`:**

```json
{
  "id": "csp_01hq...",
  "name": "Pine Ridge Camp",
  "location": "VT, USA",
  "pricing": "$35/night",
  "general_info": "Quiet sites near the brook.",
  "image_url": "/media/csp_01hq.../primary.jpg",
  "review_count": 0,
  "percent_positive": 0,
  "created_at": "2026-03-20T15:00:00Z"
}
```

**Errors:** `413` `PAYLOAD_TOO_LARGE`; `415` `UNSUPPORTED_MEDIA_TYPE`; `422` `VALIDATION_ERROR` with field details.

```bash
curl -X POST "http://localhost:8080/api/v1/campsites" \
  -b cookies.txt \
  -F 'payload={"name":"Pine","location":"VT","pricing":"$35","general_info":"Nice"};type=application/json' \
  -F "image=@./photo.jpg;type=image/jpeg"
```

---

### POST /api/v1/campsites/{campsite_id}/reviews

**Description:** Submit **one** review per authenticated user per campsite (unique constraint). Body contains binary rating and comments.

**Authentication:** Required.

**Request body (application/json):**

```json
{
  "rating": "thumbs_up",
  "comments": "Great swimming hole."
}
```

`rating` enum: `thumbs_up` | `thumbs_down`.

**Response `201 Created`:**

```json
{
  "id": "rev_01hq...",
  "campsite_id": "csp_01hq...",
  "rating": "thumbs_up",
  "comments": "Great swimming hole.",
  "author_display_name": "River Camper",
  "created_at": "2026-03-20T16:00:00Z"
}
```

**Errors:** `409` `DUPLICATE_REVIEW` if user already reviewed this campsite; `404` if campsite missing.

---

### GET /api/v1/campsites/{campsite_id}/reviews

**Description:** List reviews for a campsite (default **newest first**).

**Authentication:** Required.

**Query parameters:** `limit` (optional), `cursor` (optional).

**Response `200 OK`:**

```json
{
  "data": [
    {
      "id": "rev_01hq...",
      "rating": "thumbs_up",
      "comments": "Great swimming hole.",
      "author_display_name": "River Camper",
      "created_at": "2026-03-20T16:00:00Z"
    }
  ],
  "meta": { "next_cursor": null, "limit": 50 }
}
```

---

### GET /api/v1/admin/subscribers

**Description:** List subscribers for management UI (admin only).

**Authentication:** Required, role `admin`.

**Response `200 OK`:**

```json
{
  "data": [
    {
      "id": "usr_01hq...",
      "email": "user@example.com",
      "display_name": "River Camper",
      "status": "active",
      "created_at": "2026-03-01T10:00:00Z"
    }
  ]
}
```

**Errors:** `403` `FORBIDDEN` for non-admin.

---

### POST /api/v1/admin/subscribers/{user_id}/deactivate

**Description:** Deactivate a subscriber (soft delete default). Requires confirmation in UI; server accepts optional `reason` for audit.

**Authentication:** Required, role `admin`.

**Request body:**

```json
{
  "reason": "Terms violation",
  "confirm": true
}
```

**Response `200 OK`:**

```json
{
  "target_user_id": "usr_01hq...",
  "status": "inactive",
  "admin_action_id": "adm_01hq..."
}
```

**Errors:** `403` if not admin; `404` if user not found; `400` if attempting to deactivate self or another admin (policy).

---

## Error envelope

All error responses use:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": { "field": "email", "issue": "already_registered" }
  }
}
```

| HTTP | Code | When |
|------|------|------|
| 400 | `BAD_REQUEST` | Malformed request |
| 401 | `UNAUTHORIZED` | Missing/invalid session |
| 403 | `FORBIDDEN` | Authenticated but not allowed |
| 404 | `NOT_FOUND` | Resource missing |
| 409 | `CONFLICT` / `DUPLICATE_REVIEW` | Unique constraint / conflict |
| 413 | `PAYLOAD_TOO_LARGE` | Upload too large |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | Bad file type |
| 422 | `VALIDATION_ERROR` | Schema validation |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Unexpected server error (no stack in client) |

## Rate limiting

- Edge or BFF enforces per-IP and/or per-session limits for auth and upload routes stricter than read routes.
- Responses may include: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
- **`429`** with `error.code` = `RATE_LIMITED`.

## Health (operational)

Not part of the SPA contract but required for orchestration:

- `GET /health/live` — process up.
- `GET /health/ready` — DB and critical dependencies reachable.

Same paths on BFF and backend (implementation detail); document ports in README.

## Versioning and compatibility

- Breaking changes require `/api/v2` or negotiated deprecation window.
- OpenAPI YAML for this surface should be generated in CI and committed or published as a build artifact under `packages/api-contracts`.
