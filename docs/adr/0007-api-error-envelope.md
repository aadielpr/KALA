# API error envelope: message + reason + optional error bag

Every API error response from every endpoint uses one envelope:

```json
{
  "message": "<status-code name>",
  "reason": "<specific phrase, optional>",
  "error":  { "<field>": ["<msg>", ...] }
}
```

- **`message`** — always present. The HTTP status-code name in lowercase (`"bad request"`, `"conflict"`, `"forbidden"`, `"unauthorized"`, `"not found"`, `"internal server error"`).
- **`reason`** — optional. A short specific phrase when the status name alone is not enough (`"slug already taken"`, `"event not found"`, `"malformed body"`, `"validation failed"`). Absent when `message` already says enough (`"forbidden"`, `"unauthorized"`, `"internal server error"`).
- **`error`** — optional. An object whose keys are field names and whose values are always arrays of message strings. Present **only** for 400 request-body validation, where every field's messages are collected across all body checks and returned together (not fail-fast). Absent for every other status.

## Accumulate vs fail-fast by tier

- **400 request-body validation** runs *all* body checks, collects every field's messages, and returns once. Not fail-fast. Carries `reason: "validation failed"` plus the `error` bag.
- **Everything else** (401, 403, 409, 500, entity-not-found, etc.) returns immediately (fail-fast) with `message` and, where useful, `reason`. No `error` bag.

## Status-code conventions

- **Entity-not-found → 400**, with `reason: "event not found"` (or the relevant entity). The `error` bag is not used here — `reason` carries the specificity.
- **404 is reserved for a missing HTTP route only** — an unmatched path, nothing about a found-but-absent entity. This overrides the #9 comment's `GET /api/events/:slug` 404 `event_not_found`; entity-not-found is 400.

## Why

The original Echo implementation defaulted to a single `{"message": "..."}` string. That can't carry two things the frontend needs at once: *which* status class failed, and *what specifically* failed. The status-name `message` gives the class (so the frontend can branch on `message === "conflict"` …); `reason` gives the specificity (`"slug already taken"` vs some other future 409); the `error` bag gives per-field detail so the phone form can highlight the bad fields in one round-trip. Accumulating the bag (not fail-fast) means an organizer tapping Submit on a three-field phone form gets all the format errors at once, not one at a time. This API contract remains in force across the Bun/TypeScript rewrite and is independent of the HTTP framework.

## Considered Options

- **Echo default `{"message": "..."}`.** Rejected: one string can't carry class + specificity + per-field at once; the frontend can't highlight the bad fields from it.
- **RFC 7807 Problem Details (`application/problem+json` with `type`/`title`/`detail`/`invalid_params`).** Rejected: heavier than this MVP needs, the fields overlap with our `message`/`reason`/`error`, and the Content-Type negotiation is overhead for a SolidJS SPA that already reads JSON. We borrow the spirit (a small structured envelope) without the wire format.
- **Top-level magic error codes (`{"error": "slug_invalid"}`, `{"error": "slug_taken"}`, etc.) as the #9 comment and #20 AC originally named them.** Rejected: they collide if you ever need to report two at once, they encode the status class opaquely (the frontend has to learn each code), and they don't carry per-field detail. `message` (class) + `reason` (specific phrase) + `error` bag (per-field) generalizes to every endpoint this slice and later slices will add.

## Consequences

- #20's AC named codes are retired: `invalid_request` → 400 `{ "message": "bad request", "reason": "validation failed", "error": { ... } }`; `slug_invalid` → a field message inside the `error` bag under `slug`; `slug_taken` → 409 `{ "message": "conflict", "reason": "slug already taken" }`. Integration tests assert on status + `message` + `reason`/field-keys, not on magic codes.
- #21's `GET /api/events/:slug` entity-not-found is 400 `{ "message": "bad request", "reason": "event not found" }`, not 404. Carried forward when #21 is grilled.
- Every handler this slice adds and every later slice's handlers must use the envelope. A small helper (e.g. `handlers ValidationError(msgs)`, `handlers Error(status, reason)`) renders it — the one seam to get right.
- The `error` bag's field names are the JSON request field names (`title`, `starts_at`, `slug`), not DB column names. Payload casing is snake_case (per #9 comment), so the keys are snake_case.
- Adding `reason` later to a response that currently only has `message` is additive and safe. Removing the envelope later is not — once the frontend branches on `message`/`reason`, every handler is locked in.
