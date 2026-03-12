---
name: restful-api-design
description: Use when designing or reviewing REST API endpoints, resource naming, HTTP methods, status codes, pagination, versioning, or error response structures
---

# RESTful API Design

Design APIs around **resources (nouns)**, not operations or database tables. Requests must be **stateless**. Model business concepts, not your schema. Internal refactors should never force API changes.

## URLs

- Plural nouns for collections: `/users`, `/users/{id}`, `/customers/{id}/orders`
- Never verbs in paths: ~~`/getUsers`~~ ~~`/createUser`~~ — use HTTP methods
- Max 2 levels of nesting; flatten deeper hierarchies to top-level resources
- Identifiers must be stable and opaque — no internal keys or table names (`/user_role_map`)

## HTTP Methods

| Method | Semantics | Idempotent |
|--------|-----------|------------|
| GET | Read (must be side-effect-free) | Yes |
| POST | Create / execute action | No |
| PUT | Full replace | Yes |
| PATCH | Partial update | No |
| DELETE | Remove | Yes |

- Don't use POST as catch-all — breaks caching, retries, idempotency
- Don't overload endpoints with multiple behaviors via body shape or magic headers

## Status Codes

Use standard codes. Non-obvious guidance:

| Code | Note |
|------|------|
| 201 Created | Include `Location` header |
| 202 Accepted | Long-running ops — return status URL |
| 204 No Content | DELETE or PUT with no body |
| 400 | Unparseable request (malformed JSON, wrong content type) |
| 401 | Missing or invalid authentication |
| 403 | Authenticated but insufficient permissions |
| 404 | Also use to hide 403 for security |
| 422 | Parseable but semantically invalid (missing required field, bad value) |
| 429 | Include `Retry-After` header |

- Never return 200 on errors
- Always `Content-Type: application/json`; support `Accept` for content negotiation

## Error Envelope

Consistent across **all** endpoints:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email address is not valid",
    "details": [{ "field": "email", "reason": "must be a valid email format" }],
    "correlationId": "abc-123-def"
  }
}
```

- `code`: `SCREAMING_SNAKE_CASE` — `VALIDATION_ERROR`, `RATE_LIMIT_EXCEEDED`, `INTERNAL_ERROR`
- `details`: validation → `{ "field", "reason" }`, rate limiting → `{ "retryAfterSeconds" }`, general → omit
- Never leak stack traces, SQL errors, or hostnames

## Filtering, Sorting, Pagination

```
GET /orders?status=shipped&sort=-createdAt&limit=25&offset=0
GET /products?fields=id,name,price
```

- Query params only — never filters in the path
- Default page size 25, max 100; never unbounded result sets
- `-` prefix for descending sort; `?fields=` for field selection
- Wrap collections in an object (`{ "data": [...], "total": N }`) — never return bare arrays

## Versioning

- One strategy: URL segment (`/v1/`) or headers
- Additive changes (new fields) allowed within major version
- Never silently change existing version behavior
- Deprecate with timeline and migration path

## Security

- HTTPS only
- Auth in headers (OAuth 2.0/OIDC/JWT), not query strings
- Resource-level authorization with least privilege
- Validate all inputs; rate limit; consider 404 vs 403 for existence leaks

## Performance

- Cache headers (`ETag`, `Last-Modified`, `Cache-Control`), compression (gzip/brotli)
- Idempotent retries on writes; avoid chatty APIs — use better resource shapes or batches

## Async Operations

For long-running work, return `202 Accepted` with a pollable status resource:

```json
// POST /orders/batch-import → 202
{ "id": "job-123", "status": "pending", "statusUrl": "/jobs/job-123" }

// GET /jobs/job-123 → 200 (on completion)
{ "id": "job-123", "status": "completed", "progress": 100,
  "result": { "imported": 847, "failed": 3,
    "errors": [{ "line": 12, "reason": "invalid SKU" }] } }
```

Status values: `pending` | `processing` | `completed` | `failed`. Optionally support webhooks.

## Batch and Action Endpoints

Use sparingly:

```
POST /users/batch-delete { "ids": ["id1", "id2"] }   # batch — per-item error reporting
POST /orders/{id}/cancel                               # action — only if not a state change
```

Don't turn the API into RPC (`/runReport`, `/doThing`).
