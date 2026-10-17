# REST API Reference (plugin side)

Namespace: `/wp-json/seosaas/v1/`. This is the API **exposed by the WordPress plugin**.
The SaaS-side API (pairing exchange, tasks, report-back, licensing) is a separate project and will
be captured as an OpenAPI contract consumed by `SaasClientInterface`.

## Principles

- Resource-oriented, plural kebab-case nouns, standard verbs, JSON only.
- Every route declares: **controller class, JSON schema for args, permission callback, rate-limit bucket**. A route missing any of the four fails the CI route-audit (`bin/check-routes.php`).
- Pagination: `page`, `per_page` (max 100); totals in `X-WP-Total` / `X-WP-TotalPages` headers.
- Timestamps ISO 8601 UTC. WordPress object IDs are integers; SaaS object IDs are opaque strings.
- Breaking changes require a new namespace (`v2`); `v1` is supported for at least two minor versions after a deprecation notice.

## Authentication modes

| Mode | Used by | Mechanism | Checks |
|---|---|---|---|
| Admin session | React admin UI | Cookie auth + `X-WP-Nonce` | Nonce valid **and** `current_user_can(<capability>)` |
| SaaS signed request | SaaS push calls | Headers `X-SeoSaas-Key-Id`, `X-SeoSaas-Timestamp`, `X-SeoSaas-Nonce`, `X-SeoSaas-Signature` (Ed25519 over method + path + timestamp + nonce + SHA-256 of body) | Verifies against pinned SaaS public key; timestamp ±5 min; nonce unseen (transient, 10 min); site is paired |
| Public | — | **Not allowed** | No route may use `__return_true` (CI rule) |

Outbound (plugin → SaaS): `Authorization: Bearer <site token>` plus `X-SeoSaas-Signature` HMAC over the body, HTTPS only, certificate verification always on.

## Response envelope

```jsonc
// Success (200 / 201)
{ "success": true, "data": { }, "meta": { "page": 1, "per_page": 20, "total": 134 }, "error": null }

// Failure (4xx / 5xx)
{
  "success": false, "data": null,
  "error": {
    "code": "seosaas_content_conflict",
    "message": "The page changed after this suggestion was created.",
    "details": { "post_id": 42 },
    "request_id": "req_8f2c1a"
  }
}
```

## Error codes

| HTTP | Code | When | Exception |
|---|---|---|---|
| 400 | `seosaas_bad_request` | Malformed JSON or missing body | `ValidationException` |
| 401 | `seosaas_unauthenticated` | Missing/invalid signature or nonce | `AuthException` |
| 403 | `seosaas_forbidden` | Capability missing or site not paired | `AuthException` |
| 404 | `seosaas_not_found` | Resource does not exist | `NotFoundException` |
| 409 | `seosaas_content_conflict` | Source content changed; rollback conflict; duplicate state | `ConflictException` |
| 422 | `seosaas_validation_failed` | Schema or business-rule validation failed | `ValidationException` |
| 429 | `seosaas_rate_limited` | Limit exceeded (includes `Retry-After`) | `RateLimitException` |
| 500 | `seosaas_internal_error` | Unexpected; generic message, details only in logs | `Throwable` |
| 502 | `seosaas_upstream_error` | SaaS returned error or invalid response | `ApiException` |
| 503 | `seosaas_unavailable` | Circuit breaker open or queue unavailable | `ApiException` |

## Rate limits (defaults, configurable within hard limits)

| Bucket | Limit | Applies to |
|---|---|---|
| `admin-read` | 120 / min / user | GET routes from UI |
| `admin-write` | 30 / min / user | POST/PUT/DELETE from UI |
| `saas-inbound` | 60 / min / site | SaaS push endpoints |
| `scan-trigger` | 1 / min | `POST /inventory/scans` |

Backed by `RateLimiterInterface` (transients/object-cache by default; Redis implementation can replace it).

## Endpoint catalogue

| Method | Route | Purpose | Auth / capability |
|---|---|---|---|
| GET | `/health` | Plugin version, schema version, connection and queue state | Admin: `seosaas_view` |
| POST | `/pairing` | Start pairing with a code | Admin: `seosaas_manage_connection` |
| GET | `/pairing` | Current pairing status (never returns the token) | Admin: `seosaas_view` |
| DELETE | `/pairing` | Disconnect and revoke | Admin: `seosaas_manage_connection` |
| GET | `/inventory` | List inventoried items (filters: orphan, broken, type, search) | Admin: `seosaas_view` |
| GET | `/inventory/summary` | Counts: total, orphan, broken, over-linked | Admin: `seosaas_view` |
| POST | `/inventory/scans` | Queue a scan (full or incremental) | Admin: `seosaas_run_scan` |
| GET | `/inventory/scans/{id}` | Scan progress and result | Admin: `seosaas_view` |
| POST | `/drafts` | Receive a draft from SaaS (idempotent) | SaaS signed |
| GET | `/drafts` | List drafts by status | Admin: `seosaas_view` |
| GET | `/drafts/{id}` | Draft detail and preview data | Admin: `seosaas_view` |
| POST | `/drafts/{id}/approve` | Approve `{mode: publish\|schedule, publish_at?}` | Admin: `seosaas_approve_content` |
| POST | `/drafts/{id}/reject` | Reject with reason | Admin: `seosaas_approve_content` |
| POST | `/drafts/{id}/request-changes` | Ask SaaS to revise | Admin: `seosaas_approve_content` |
| POST | `/drafts/{id}/unpublish` | Return published post to draft | Admin: `seosaas_approve_content` |
| POST | `/links/suggestions` | Receive suggestions from SaaS (batch, idempotent) | SaaS signed |
| GET | `/links/suggestions` | List suggestions by status | Admin: `seosaas_view` |
| GET | `/links/suggestions/{id}/preview` | Before/after preview | Admin: `seosaas_view` |
| POST | `/links/suggestions/{id}/apply` | Apply one suggestion | Admin: `seosaas_manage_links` |
| POST | `/links/suggestions/{id}/reject` | Reject suggestion | Admin: `seosaas_manage_links` |
| POST | `/links/bulk-apply` | Queue many approved suggestions | Admin: `seosaas_manage_links` |
| GET | `/links/changes` | History of applied changes | Admin: `seosaas_view` |
| POST | `/links/changes/{id}/rollback` | Roll back one applied change | Admin: `seosaas_manage_links` |
| GET | `/audit` | Query audit log with filters | Admin: `seosaas_view_audit` |
| GET | `/audit/export` | CSV export (streamed) | Admin: `seosaas_view_audit` |
| GET / PUT | `/settings` | Read / update settings (validated) | Admin: `seosaas_manage_settings` |
| GET | `/jobs` | Background job list and status | Admin: `seosaas_view` |
| POST | `/jobs/{id}/retry` | Retry a failed job | Admin: `seosaas_manage_settings` |
| GET | `/diagnostics` | System checks and redacted support bundle | Admin: `seosaas_manage_settings` |

### Draft payload (`POST /drafts`)

```json
{
  "saas_draft_id": "dr_01H...",
  "idempotency_key": "site123:dr_01H...:v1",
  "title": "Hidden Beaches in Neelum Valley",
  "slug": "hidden-beaches-neelum-valley",
  "content_html": "<h2>...</h2><p>...</p>",
  "excerpt": "...",
  "meta": { "title": "...", "description": "...", "focus_keyword": "..." },
  "category_names": ["Travel Guides"],
  "featured_image": { "url": "https://...", "alt": "..." },
  "proposed_publish_at": "2026-11-02T09:00:00Z",
  "internal_links": [ { "target_post_id": 42, "anchor": "Neelum tour packages" } ]
}
```

Unknown fields are rejected (422), bodies over 2 MB are rejected, and replaying the same `idempotency_key` returns the existing mapping with HTTP 200.

### Link suggestion payload (`POST /links/suggestions`)

```json
{ "suggestions": [ {
  "suggestion_id": "ls_01H...", "source_post_id": 12, "target_post_id": 42,
  "anchor_text": "Neelum tour packages", "context_snippet": "…explore the valley with our…"
} ] }
```

External targets are rejected with 422 (FR-LNK-011).
