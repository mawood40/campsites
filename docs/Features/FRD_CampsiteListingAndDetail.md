# FRD: Campsite Listing and Detail

## Feature Overview

**Purpose:** Let authenticated members browse all campsites with **pagination**, see **review_count** and **percent_positive** (canonical aggregate), and open a **detail** view with full metadata. PRD §6.4–6.5, §6.10, §9.2.

**Scope:** Read-only catalogue APIs and SPA views. **Out of scope:** Editing or deleting campsites in v1 (PRD §2.1 campsite CRUD limited to create + read).

**Business Value:** Core discovery experience; aggregates communicate community sentiment at a glance.

**Priority:** Must-Have

## User stories

- **As a subscriber**, I want to see a list of campsites with image, location, price, and rating summary so that I can choose where to explore.
- **As a subscriber**, I want to open a campsite’s detail page so that I can read full information before reviewing.
- **As the system**, I must compute aggregates from stored reviews consistently across list and detail.

### Edge cases

- Campsite with **zero reviews** → `review_count = 0`, `percent_positive = 0` (or `null` if product prefers “N/A” — **use 0** for simpler UI; document in API).
- Very long `general_info` → truncate on card with “read more” on detail.
- Missing or broken image URL → placeholder image in SPA.

## Functional requirements

1. **GET list** requires authenticated session; supports `limit` + opaque `cursor` (keyset pagination by `created_at`, `id`).
2. Each list item includes: `id`, `name`, `location`, `pricing`, `general_info` (or truncated), `image_url`, `review_count`, `percent_positive`, `created_at`.
3. **GET detail** by id returns full fields; **404** if not found.
4. Aggregates are **read from campsite row** updated transactionally on review write (see [FRD_ReviewsAndRatings.md](FRD_ReviewsAndRatings.md)) or equivalent controlled recomputation — **no drift** between list and detail.
5. Optional **Redis cache** for list/detail keys invalidated on review create or campsite create (Stage 3).

## Technical requirements

- Backend: SQLAlchemy queries; indexes on `(created_at DESC, id)` for pagination.
- BFF: maps backend models to stable SPA DTOs.

### Dependencies

- **Internal:** Authentication; `campsites` and `reviews` tables; aggregate columns on `campsites`.

## API specifications

See [API.md](../API.md):

- **GET /api/v1/campsites**
- **GET /api/v1/campsites/{campsite_id}**

**Example (Python):**

```python
import requests

r = requests.get(
    "http://localhost:8080/api/v1/campsites",
    cookies=cookies,
    params={"limit": 20},
)
r.raise_for_status()
page = r.json()
```

**Error responses:** `401 UNAUTHORIZED`; `404 NOT_FOUND` (detail).

## Data models

### `campsites` (relevant columns)

| Column | Notes |
|--------|-------|
| review_count | int, maintained |
| percent_positive | int 0–100, maintained |
| image_url | public URL or path served via gateway |

### Data flow (ASCII)

```
SPA--GET/list-->BFF-->Backend-->PostgreSQL(rows)
                    |
                    +-->Redis(optional cache)-->return DTO
```

## Security requirements

- Only **active** authenticated users; no public anonymous catalogue in v1 per PRD entry experience.

## Performance criteria

- List p95 < **500 ms** for 20 rows with cold DB on dev hardware; use pagination to cap payload.
- Cursor pagination avoids large `OFFSET` for scale.

## Testing requirements

### Unit

- DTO mapping; pagination cursor encode/decode.

### Integration

- Seed N campsites; verify order; verify aggregates match synthetic reviews.

### Acceptance

- PRD §17.1: listing shows correct average summary and count after reviews submitted (E2E).

## Implementation notes

- **percent_positive** = `round(100 * thumbs_up_count / NULLIF(review_count,0))` with 0 when no reviews.
- Align card layout with [UI_UX_doc.md](../UI_UX_doc.md).
