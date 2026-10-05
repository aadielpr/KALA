# Admin token rides in the admin_link query string; logs must not capture it

The admin link carries the event id and the plaintext admin token as a query parameter: `/admin/<event_id>?token=<admin_token>`. This keeps the one-tap UX from DECISIONS.md #3 and PRD user story 2 — the organizer bookmarks the link and is authenticated to their dashboard with no extra step.

Putting a secret bearer credential in a URL query string is normally an anti-pattern: query strings land in access logs, browser history, and `Referer` headers. We accept the history and `Referer` surfaces as documented consequences (the admin page links nowhere external in the MVP, and the organizer's own phone history is their own device). The one surface we control server-side is the **access log**: the server uses `middleware.RequestLogger()`, which by default logs the full request URI including the query. That would write the live admin token to logs on every admin hit, defeating ADR-0004's "a DB leak should not grant admin power" guarantee from the log side.

So this ADR records two coupled decisions:

1. The admin link shape is `/admin/<event_id>?token=<admin_token>` (host-relative), and the `/portal` paste flow (ticket #8) is the recovery path for organizers who lose it.
2. The request logger **must not capture the query string**. Concretely: log `c.Path()` (the matched route, e.g. `/admin/:id`) or otherwise drop the raw query, not `c.Request().URL.RequestURI()`. This applies at minimum to `/admin/` paths; a global "don't log query strings" rule is simpler and safer.

## Considered Options

- **Token in a path segment (`/admin/<id>/<token>`).** Still in the URL (same history concern), marginally better against naive loggers that log path-without-query. Same bookmark UX. Rejected: a logger that logs the full URI still captures it; the real fix is the logger, not the URL shape, and the query form is easier to parse client-side and matches the `/portal?token=` paste flow.
- **No token in the URL; rely on `/portal` paste flow only.** Cleanest — the secret never lives in a URL, so no log/history/Referer surface exists at all. Rejected: it kills the "click the bookmark and you're in" UX the PRD wants (user story 2), and pushes all admin auth into ticket #8 (the `/portal` page), which is a later slice. Bigger scope change for no MVP benefit; the log fix already closes the one surface we control.

## Consequences

- The admin link is a bookmark; the organizer is authenticated by clizing it. No cookie is set at creation time — the query param itself is the credential for the admin dashboard slice. The dashboard slice (ticket #8) may exchange it for a session cookie, but that's its decision, not this slice's.
- The API request logger must be configured so the query string is never emitted on any path. A global redaction is preferred over a `/admin/`-only one — simpler, and it protects any future secret query param too.
- The plaintext token still exists in exactly the places ADR-0004 enumerates (the `POST /api/events` 201 response, and now: the admin link URL the organizer holds). Adding it to the URL does **not** add a server-side storage location — the server never logs it, never persists it, never echoes it after the 201.
- Changing the admin link shape later (e.g. to a path segment or a cookie) means every previously-issued bookmark breaks. This is the hard-to-reverse part.
