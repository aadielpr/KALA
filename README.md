# KALA

A web app for collecting photos from event guests into one shared gallery.

## Target stack

- **Backend:** TypeScript API running on Bun
- **Frontend:** SolidJS SPA written in TypeScript, built with Vite and Bun
- **Database:** Postgres
- **Object storage:** Cloudflare R2 through its S3-compatible API
- **Migrations:** SQL migrations through a Bun-compatible runner (to be selected in issue #7)
- **Package manager and runtime:** Bun

The API and frontend are separate applications. In development, Bun runs the API and Vite serves the frontend in separate processes. In production, Caddy serves the built frontend and routes `/api/*` to the API. The API never serves frontend files or provides an SPA fallback. Both services share one public origin by default so the existing cookie-based sessions remain same-origin.

See [ADR 0008](docs/adr/0008-bun-typescript-separate-frontend.md) for the architecture decision and [the PRD](docs/PRD.md) for the MVP scope.

## Current implementation status

The previous Go backend, frontend scaffold, migrations, Docker files, and Makefile have been removed so implementation can restart from a clean slate. Product decisions and feature scope remain documented in `DECISIONS.md`, `CONTEXT.md`, the PRD, and ADRs. Issue #7 defines the new Bun/TypeScript scaffold.

## Local services

No Docker or MinIO setup is included. Developers provide a reachable Postgres and configure the API through environment variables. Object storage uses a configured R2 bucket; the test bucket and credentials are established with the upload implementation. See [ADR 0008](docs/adr/0008-bun-typescript-separate-frontend.md).

## Target development flow

The refreshed scaffold will provide Bun scripts to install dependencies, start the API and Vite frontend independently (and together), build the frontend, run API integration tests, and apply or roll back SQL migrations. There is no Makefile; the exact scripts and required Bun version are part of issue #7.

The API and frontend run as separate processes. In production, Caddy serves the frontend build and routes `/api/*` to the API under one public origin.

## Repository layout

The repository currently contains product and architecture documentation only. Issue #7 establishes the Bun workspace layout and the separate API and frontend applications.
