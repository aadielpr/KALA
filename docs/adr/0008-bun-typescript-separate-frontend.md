# Bun and TypeScript API with an independently served frontend

**Status:** Accepted, 2026-10-05

## Decision

- Use TypeScript on Bun for the backend API.
- Keep the frontend as a SolidJS SPA built with Vite. Use Bun to install dependencies, run scripts, and build it.
- Treat the API and frontend as separate applications and services. Run them as separate processes in development and build/deploy them independently. The API owns JSON API and health endpoints only; it never serves the SPA files or an `index.html` fallback.
- In development, Vite serves the SPA and proxies `/api/*` to the API. In production, Caddy serves the standalone frontend build and proxies `/api/*` to the API. They share a public browser origin by default, preserving the existing cookie-based admin and guest sessions.
- Do not include Docker, Docker Compose, MinIO, or a Makefile in the fresh scaffold. Use Bun scripts, developer-provided Postgres, and a configured R2/test storage endpoint.
- Keep Postgres, object storage, and the product behavior unchanged. Select the Bun-compatible HTTP framework and migration runner in the refreshed scaffold issue (#7).

## Context

The product decisions and MVP stories stay the same, but the original Go/Echo scaffold has made the API language and frontend serving model part of several tickets. The target is a TypeScript codebase using Bun across the API and frontend toolchain. The frontend must be independently runnable and deployable rather than a route or static-file fallback owned by the API.

The current authentication decisions use cookies. Separate services can still share one public origin through path routing, so browser requests remain same-origin while the API and frontend have independent processes, builds, and deploys. Exposing the frontend and API on different public origins would require a separate decision about cookie scope, CORS, and credential handling.

## Consequences

- The previous implementation scaffold has been removed. Rebuild from the open scaffold ticket (#7) and data-layer ticket (#19), both written for Bun/TypeScript.
- Do not restore the removed Go/Echo sources, `air`, `goose`, container files, local object-store emulator, or Makefile as part of the target implementation.
- Keep API contracts over HTTP; the frontend does not import backend implementation modules.
- Vite's development proxy and Caddy's production routes must both send `/api/*` to the API. The API must not handle browser page routes.
- Bun version, workspace layout, HTTP framework, migration tool, and dev scripts are finalized in issue #7.
