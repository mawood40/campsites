# FRD: Campsite Creation and Image Upload

## Feature Overview

**Purpose:** Authenticated members create a new **Campsite** with required fields and **exactly one primary image** stored outside container filesystem. PRD §6.6–6.7, §15.3.

**Scope:** Create-only (no edit/delete v1); multipart upload through BFF; pluggable storage adapter. **Out of scope:** Multiple images per campsite, image editing, virus scanning beyond config hook placeholder.

**Business Value:** Community-generated catalogue growth with trustworthy metadata and visuals.

**Priority:** Must-Have

## User stories

- **As a subscriber**, I want to submit a new campsite with name, location, pricing, description, and a photo so that others can discover it.
- **As the system**, I want to reject oversized or wrong-type files so that storage and bandwidth stay safe.

### Edge cases

- Upload succeeds but DB insert fails → **orphan file cleanup** job or transactional outbox pattern; minimum requirement: document manual cleanup + best-effort delete in request handler.
- Concurrent creates by same user → both allowed (no uniqueness constraint on name required by PRD).

## Functional requirements

1. **POST /api/v1/campsites** accepts `multipart/form-data` with JSON `payload` part and `image` file part.
2. Required fields: `name`, `location`, `pricing` (text or structured per schema), `general_info`, `image`.
3. Validate **MIME** allowlist (`image/jpeg`, `image/png`, `image/webp`) and **max size** from `storage.json`.
4. Sanitize filename; generate stored object name with campsite id or UUID to avoid collisions.
5. Persist `image_url` (or path) on `campsites` row; set `created_by` to current user id.
6. Return **201** with full campsite DTO including `review_count: 0`, `percent_positive: 0`.

## Technical requirements

- Storage adapter interface: `put_stream(key, content_type, body) -> url`; implementations: `LocalFilesystemStorage`, `S3CompatibleStorage`.
- BFF may stream upload to backend or proxy to pre-signed URL flow **v2**; v1: single POST multipart to BFF → backend.

### Dependencies

- Authentication; `campsites` table; gateway or static route to serve public media URLs.

## API specifications

Primary reference: [API.md](../API.md) **POST /api/v1/campsites**.

**Error JSON examples:**

```json
{
  "error": {
    "code": "PAYLOAD_TOO_LARGE",
    "message": "Image exceeds maximum size",
    "details": { "max_bytes": 5242880 }
  }
}
```

```bash
curl -X POST "http://localhost:8080/api/v1/campsites" \
  -b cookies.txt \
  -F 'payload={"name":"Pine","location":"VT","pricing":"$35","general_info":"Quiet"};type=application/json' \
  -F "image=@./photo.jpg;type=image/jpeg"
```

## Data models

### `campsites`

| Column | Notes |
|--------|-------|
| id | PK |
| name, location, pricing, general_info | text |
| image_url | text |
| created_by | FK users |
| created_at | timestamptz |
| review_count, percent_positive | defaults 0 |

### Data flow (ASCII)

```
SPA--multipart-->BFF-->Backend-->validate-->StorageAdapter.put
                              |                |
                              +--INSERT camp---+
```

## Security requirements

- Authenticated users only; optional future: rate limit creates per user.
- Do not execute uploaded content as code; serve images with `Content-Type` from sniffed/declared type.
- Path traversal forbidden in local adapter.

## Performance criteria

- Accept **up to 5 MB** default image; streaming upload without loading full file into memory twice where possible.

## Testing requirements

### Unit

- Filename sanitization; MIME allowlist; adapter contract with fake adapter.

### Integration

- Full multipart request; assert file on disk or MinIO bucket; DB row correct.

### Acceptance

- PRD §17.1: subscriber adds campsite with image and sees it in list (E2E).

## Implementation notes

- Configure `infra/config/base/storage.json`: `adapter`, `local_path`, `max_bytes`, `allowed_mime_types`, `public_base_url`.
