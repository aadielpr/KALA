# Infrastructure is backend-agnostic (Docker optional, local supported)

> **Historical note:** Docker was removed from the fresh implementation reset on 2026-10-05. This ADR is kept as a record of the earlier Docker-optional decision. Environment-driven configuration and developer-provided Postgres remain valid; references below to Docker Compose and Make targets are historical.

## Original decision (historical)

The two infra dependencies — Postgres and object storage — are reached only through env vars (`DATABASE_URL`, `STORAGE_*`), never through a hard-coded Docker address. Docker Compose (`make infra`) stays the recommended default for a fast setup, but running the services natively (e.g. a Homebrew Postgres and a standalone MinIO binary, or a cloud R2 bucket) is an equally supported path: the app must start fine with no Docker daemon running, as long as the env points at reachable services.

## Context

Issue #7 ships the MVP scaffold and lists Docker Compose for local dev. On macOS, Docker Desktop consumes a meaningful chunk of CPU and memory even at idle, and the `5433` port mapping in `docker-compose.yml` avoids collisions with a developer's locally-installed Postgres. Decoupling the app from Docker lets a developer who already runs Postgres natively skip Docker entirely.

## Considered Options

- **Docker-only** — `make dev` required to start the app; env hard-codes the Docker addresses. Rejected: forces the Docker daemon on macOS for everyone, even those with native Postgres; also the `5433` dodge is a symptom of coupling, not a solution.
- **Native-only** — drop Docker, document manual install for Postgres + MinIO. Rejected: raises the onboarding floor for a new contributor who lacks either service and wants one command. Docker remains the easiest first-run path.
- **Backend-agnostic (chosen)** — env-driven config; Docker is the recommended default, native is supported.

## Consequences

- Application configuration is the single entry point for infrastructure addresses — no code path may hard-code `localhost:5433` or the MinIO Docker port.
- `docker-compose.yml` keeps its `5433` mapping for the Docker path; local Postgres users set `DATABASE_URL` to their native port (typically `5432`). Both are correct for their path; do not "fix" one to match the other.
- `make infra` is documented as *recommended, not required*. `make dev` must not assume Docker is running — it only starts the Bun API and the separate Vite dev server.
- The README documents both paths (Docker and native) so the "documented equivalent" clause in issue #7's AC #1 is satisfied either way.
