# ADR 0012 — API conventions: typed error codes in problem details, ETag concurrency, idempotency keys, keyset paging

- **Status:** Proposed, 2026-11-06 (recommendation; decide before phase 4)
- **Scope:** `Api` (every endpoint), `ApiClient`, `Contracts`, `Core` (`ErrorCode`)

## Context

Every client — web, scraper, admin tool, tests — talks to one API (ADR 0002). Errors, optimistic
concurrency, retries and paging need one answer each, decided once, or every endpoint will invent
its own. The scraper in particular retries over a flaky home connection and must never double-write
a lookup's results (ADR 0003); the web UI edits rows that another tab or the Admin may also edit;
Results and flight options are long lists read on phones.

`Chris-Wolfgang/wms` has a settled `docs/API-CONVENTIONS.md`; this ADR adopts its rules, adapted to
a per-row `Version` instead of a global row version. The full conventions document is written in
phase 0 (DESIGN §13); this ADR records the decisions it will contain.

## Decision (recommended)

### 1. Errors are problem details with a typed code

- Every error response is `application/problem+json` carrying `code` (a typed `ErrorCode` from
  `Core`: lower-case dotted name, HTTP status, message template, docs anchor, severity),
  `severity` and `traceId`. Templates use positional placeholders and are validated at startup.
- Unhandled exceptions become `500 internal.error` problem details; the detail is the trace id,
  never the exception.
- Readiness (ADR 0001) answers `503` with `database.waking`, `database.quota_exhausted`,
  `database.missing_migration` or `database.unavailable`; `Retry-After` is set when known.
- The API never returns a `3xx`; authentication failures are `401 auth.not_signed_in` and
  `403 auth.forbidden` (ADR 0005).

### 2. Status codes

| Status | When |
|---|---|
| 200 | read, or an update that returns the row |
| 201 + `Location` | create |
| 202 | accepted for later (`digests/send`) |
| 204 | delete, or an update with no body |
| 400 | malformed request, invalid cursor, unknown sort |
| 404 | not found **or not visible to the caller** (never 403 for another user's row — nothing reveals another user) |
| 409 | uniqueness or state conflict: duplicate search name, a run already in progress, a `result` for a superseded attempt |
| 412 | `If-Match` did not match the row's `Version` (stale edit), or the row changed between check and save |
| 422 | the request is well-formed but the rules reject it (validation); an `Idempotency-Key` reused with a different body |
| 428 | a change or delete without `If-Match`, or with `*` |
| 503 | readiness (above) |

### 3. Concurrency: `Version` is the ETag

- Reads of a single row return `ETag: "<Version>"`; `If-None-Match` answers `304`.
- Every `PUT`/`PATCH`/`DELETE` on a versioned row requires `If-Match` with that strong tag; missing
  or `*` is `428`; stale is `412`. Comparison is strong (RFC 9110). `ApiClient` carries the tag
  automatically from the DTO it last read.
- `Cache-Control: private, no-cache` on data; never `no-store`, never `max-age`.

### 4. Idempotency

- `POST`s that create or record (the scraper's `result` and `complete`, `digests/send`, search
  creation) accept an `Idempotency-Key` header (16–128 visible ASCII; the scraper uses its
  `attempt` token, ADR 0003). The API stores the key, a SHA-256 fingerprint of the body and the
  response for 24 h: a repeat with the same body returns the stored response; a different body is
  `422`. The check and the write share one transaction.

### 5. Paging: keyset, opaque cursors

- List endpoints take `after` / `before` (opaque cursors), `sort` (field name, `-` prefix for
  descending; a cursor is bound to its sort and refused under another with `400`), and `size`
  (default 25, cap 200; the flight-options endpoint fixes 10, price ascending). They return
  `items`, `nextCursor`, `previousCursor` and `totalCount`.
- Every sortable field is backed by an index ending in the row id. The UI shows "Showing 1–25 of
  62" with Previous/Next; no page numbers.

### 6. Compression, time, naming

- Brotli/gzip on data responses; **never** on authentication endpoints (BREACH).
- Every instant is `DateTimeOffset` in UTC, ISO 8601; the client renders in the user's zone.
- JSON is camelCase; DTOs are records in `Contracts`; enums are strings.

## Not chosen

- *409 for every conflict.* Conflates "someone else changed it" (412, reload and retry) with "that
  name is taken" (409, choose another); the client's recovery differs.
- *Offset paging with page numbers.* Simple, but unstable under inserts and expensive at depth;
  keyset gives stable Previous/Next and the mockups never show "page N of M".
- *Retries without idempotency keys.* The scraper's home connection drops mid-request often enough
  that a duplicated `result` is a matter of when.

## Consequences

- DESIGN §8 changes: stale edits are `412`/`428`, not `409`; `409` is reserved for uniqueness and
  state conflicts.
- `ErrorCode`, `Permission` and `SettingKey<T>` share one typed-key base in `Core`.
- `ApiClient` is the only place that knows about ETags, idempotency keys and cursors; screens see
  typed results and `ConflictException` / `PreconditionFailedException`.
- The conventions document, the error-code catalogue (`docs/api/errors.md`) and the OpenAPI
  document are reviewed together in phase 4.
