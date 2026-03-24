# FRD: Reviews and Ratings

## Feature Overview

**Purpose:** Authenticated members submit **binary reviews** (thumbs up / thumbs down) with **comments** for a campsite; view **all reviews** for a campsite ordered **newest first**; keep **review_count** and **percent_positive** on the campsite consistent. PRD §6.8–6.10, §9.1–9.2.

**Scope:** Single review per user per campsite; transactional aggregate update. **Out of scope:** Star ratings, editing/deleting reviews in v1, moderation queues.

**Business Value:** Community feedback loop; aggregates guide browsing decisions.

**Priority:** Must-Have

## User stories

- **As a subscriber**, I want to rate a campsite up or down and leave comments so that I can share my experience.
- **As a subscriber**, I want to read others’ reviews with author display name and time so that I can judge consensus.
- **As the system**, I must prevent duplicate reviews from the same user for the same campsite.

### Edge cases

- User submits twice → **409 DUPLICATE_REVIEW** or return existing review (pick **409** for clarity).
- Review with empty comments if policy allows → configurable `min_comment_length` in `app.json` (0 allowed default).
- Aggregate update must be **atomic** with insert (single transaction) or use serializable isolation.

## Functional requirements

1. **POST** review: body `{ "rating": "thumbs_up"|"thumbs_down", "comments": "..." }`.
2. Persist `reviews` row: `campsite_id`, `user_id`, `rating_value` (enum or bool), `comments`, `created_at`.
3. **Unique constraint** `(campsite_id, user_id)`.
4. On success, recompute or increment **`review_count`** and **`percent_positive`** on parent campsite in same transaction.
5. **GET** reviews: paginated, default sort `created_at DESC`.
6. BFF returns `author_display_name` from user profile; never expose internal user id to other subscribers unless product allows (default **hide** other users’ ids).

## Technical requirements

- Backend: SQL transaction wrapping INSERT + UPDATE campsite aggregates.
- Optional: trigger-based alternative — still must be tested for concurrency.

### Dependencies

- Campsite must exist; user authenticated; listing/detail FRD for aggregate display.

## API specifications

See [API.md](../API.md):

- **POST /api/v1/campsites/{campsite_id}/reviews**
- **GET /api/v1/campsites/{campsite_id}/reviews**

**Response `201` example:**

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

**JavaScript example:**

```javascript
await fetch(`/api/v1/campsites/${campsiteId}/reviews`, {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ rating: "thumbs_up", comments: "Great spot." }),
});
```

## Data models

### `reviews`

| Column | Type |
|--------|------|
| id | UUID PK |
| campsite_id | FK |
| user_id | FK |
| rating_value | smallint or enum (up/down) |
| comments | text |
| created_at | timestamptz |

Unique: `(campsite_id, user_id)`.

### Data flow (ASCII)

```
SPA--POST/review-->BFF-->Backend--BEGIN txn-->
                              INSERT review
                              UPDATE campsite agg
                         COMMIT-->response
```

## Security requirements

- Only authenticated **active** users; cannot review as another user.
- CSRF protection if using cookies (match auth FRD).

## Performance criteria

- Submit review p95 < **400 ms** under normal load; read list p95 < **300 ms** for first page.

## Testing requirements

### Unit

- Aggregate math: 0 reviews → add thumbs_up → 100%; mixed counts.

### Integration

- Concurrent two posts same user same campsite → one succeeds, one 409.

### Acceptance

- PRD §17.1: submit review; listing shows updated count and percent; review history visible (E2E).

## Implementation notes

- Invalidate Redis cache keys for campsite list/detail when reviews change if caching enabled.
