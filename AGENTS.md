# AGENTS.md

Repository instructions for AI coding agents. Keep this file short, concrete, and tied to commands that actually work in this
repo. If a fact here conflicts with source files or CI config, trust the source file and update this guide.

## Project Snapshot

Memos is an open-source, self-hosted note-taking app: short Markdown memos on a chronological timeline, with tags, spaces,
attachments, sharing, and webhooks. Zero telemetry, MIT license.

- Backend: Go 1.27.0, Echo v5 HTTP server, Connect RPC + gRPC-Gateway (REST/JSON via the same proto services), Protocol Buffers, `buf` for codegen.
- Frontend: React 19 (with the React Compiler Babel preset), TypeScript, Vite 8 (rolldown), Tailwind CSS v4, Base UI primitives, React Query v5, react-router v7, CodeMirror editor.
- Storage: SQLite (default, `modernc.org/sqlite`, pure Go), MySQL, PostgreSQL; object storage via S3-compatible drivers (`internal/storage`).
- AI integrations: `internal/ai` resolves providers and does speech-to-text / audio LLM calls (OpenAI, Google GenAI).
- MCP: `server/router/mcp` serves a Model Context Protocol endpoint at `/mcp` (Streamable HTTP), derived from the generated OpenAPI document and executing in-process against the REST API.
- Generated API outputs: `proto/gen/` (Go + `openapi.yaml`), `web/src/types/proto/` (TypeScript). Never hand-edit generated files.

## Working Rules

- Read relevant code before editing; prefer local patterns over new abstractions.
- Keep diffs scoped. Do not do repo-wide cleanup, dependency churn, or generated-file rewrites unless the task requires it.
- Do not hand-edit generated proto outputs. Change `.proto` files, then run `buf generate`.
- Schema changes require migrations for all three DB drivers under `store/migration/` and an update to each driver's `LATEST.sql`.
- Add public API endpoints to `server/router/api/v1/acl_config.go`.
- Ask before adding heavy dependencies, changing auth/token behavior, or altering Docker/release workflows.
- Match the domain vocabulary in `CONTEXT.md` and `docs/glossary.md` (e.g. "Memo UID", "Memo resource name", "Space", "Space membership"). Read `docs/adr/README.md` before contradicting an accepted ADR.

## Commands

Run from the repository root unless a command starts with `cd`.

```bash
# Backend
go run ./cmd/memos --port 8081    # Start backend dev server (sqlite, data dir default)
go test ./...                      # Run all Go tests
go test -v ./store/...             # Store tests: all DB drivers via TestContainers (mysql/postgres/minio)
go test -v -race ./server/...      # Server tests with race detector (includes real-server smoke via server/test)
go test -v -race ./internal/...    # Internal package tests with race detector
go mod tidy -go=1.27.0             # Match CI tidy check
golangci-lint run                  # Go lint, config: .golangci.yaml
golangci-lint run --fix            # Auto-fix lint, including goimports
./scripts/build.sh                 # Simple local binary build into ./build/

# Frontend
cd web && pnpm install             # Install dependencies (Node >= 24, pnpm 11.0.1)
cd web && pnpm dev                 # Dev server on :3001, proxying API to :8081 (override with DEV_PROXY_SERVER)
cd web && pnpm lint                # Type check + Biome lint
cd web && pnpm test                # Vitest unit tests (tests in web/tests/, jsdom + Testing Library)
cd web && pnpm build               # Production build
cd web && pnpm release             # Build SPA into server/router/frontend/dist (embedded by the Go binary)

# Protocol Buffers
cd proto && buf generate           # Regenerate Go + TypeScript + OpenAPI
cd proto && buf lint               # Lint proto files
cd proto && buf format -w          # Format proto files
```

## Code Map

| Path | Purpose |
| --- | --- |
| `cmd/memos/main.go` | Cobra/Viper CLI, flags/env (`MEMOS_*`), profile validation, server startup |
| `server/server.go` | Echo HTTP server, router wiring (API, file, frontend, MCP), graceful shutdown |
| `server/router/api/v1/` | Connect/gRPC-Gateway services, ACL config, SSE hub |
| `server/router/mcp/` | OpenAPI-derived MCP server at `/mcp` |
| `server/router/frontend/` | Static SPA serving from `dist` |
| `server/router/fileserver/` | Native HTTP file serving, thumbnails, range requests |
| `server/auth/` | JWT access tokens, refresh tokens, PAT handling |
| `server/access/` | Memo access/visibility checks |
| `server/runner/memopayload/` | Memo payload rebuilding |
| `server/notification/` | Email notifications |
| `store/store.go` | Store facade with in-memory caches, deployment config snapshot |
| `store/db/{sqlite,mysql,postgres}/` | Database-specific drivers and SQL |
| `store/migration/{sqlite,mysql,postgres}/` | Versioned migrations (`<version>/NN__name.sql`) plus `LATEST.sql` |
| `store/seed/sqlite/` | Demo-mode seed data |
| `internal/ai/` | AI provider resolution, STT, audio LLM |
| `internal/filter/` | CEL-based memo filtering |
| `internal/{idp,storage,webhook,scheduler,email,markdown,motionphoto,...}/` | App-private packages |
| `proto/api/v1/` | Public API service definitions (source of truth for REST + MCP) |
| `proto/store/` | Internal storage proto messages |
| `web/src/connect.ts` | Connect RPC clients, auth interceptor, access-token refresh |
| `web/src/auth-state.ts` | Token storage and cross-tab sync |
| `web/src/hooks/` | React Query hooks for server state |
| `web/src/components/ui/` | shadcn-style kit on Base UI — single source of styling truth (see its README) |
| `web/src/components/` | Feature components (editor, memo views, spaces, settings) |
| `web/src/themes/` | OKLch color tokens (`COLOR_GUIDE.md`, light/dark themes) |

## Change Routing

| Change | Update | Verify |
| --- | --- | --- |
| Go service or router behavior | Service code under `server/`, tests near package | `go test -v -race ./server/...` |
| Store or migration behavior | `store/`, all three driver migrations, `LATEST.sql` | `go test -v ./store/...` |
| Internal package logic | Relevant `internal/` package tests | `go test -v -race ./internal/...` |
| Frontend behavior | Components/hooks/contexts under `web/src/` | `cd web && pnpm lint && pnpm test` |
| Frontend production output | Vite config or release-sensitive UI | `cd web && pnpm build` or `pnpm release` |
| Proto API | `.proto` source plus generated outputs | `cd proto && buf generate && buf lint` |
| Public unauthenticated route | `server/router/api/v1/acl_config.go` | Targeted server test or manual route check |

## Go Conventions

- Wrap errors with `errors.Wrap`/`errors.Wrapf` from `github.com/pkg/errors`; `fmt.Errorf` is forbidden by golangci-lint (forbidigo), as is `ioutil.ReadDir`.
- Return service errors with `status.Errorf(codes.X, "message")`.
- Keep imports grouped as stdlib, third-party, then `github.com/usememos/memos`; goimports (with local prefix) runs via golangci-lint.
- Add doc comments for exported identifiers; godot enforces exported comment punctuation.
- Avoid package-level mutable state unless the surrounding package already uses that pattern.

## Frontend Conventions

- Use `@/` for absolute imports.
- Biome formatting: 2-space indent, double quotes, semicolons, trailing commas, 140-character line width. Lint forbids `any` (noExplicitAny), enforces `const`/`no var`.
- Style through the `components/ui` kit: use `variant`/`size` props, not `className`; colors come from semantic OKLch tokens, never hardcoded palette classes. Read `web/src/components/ui/README.md` first.
- Put server data in React Query hooks under `web/src/hooks/`; keep UI-only state in contexts or component state.
- Keep generated proto TypeScript under `web/src/types/proto/` out of manual edits and Biome rewrites.
- Unit tests live in `web/tests/` (colocated naming: `*.test.ts(x)`), running in jsdom with Testing Library; mocks reset between tests.

## Database And Proto Rules

- Schema changes require SQLite, MySQL, and PostgreSQL migrations plus `LATEST.sql` updates; fresh-install SQL and incremental migrations must stay equivalent.
- Proto field changes must preserve compatibility (buf `FILE` breaking check) unless the task explicitly allows a breaking API change.
- Regenerate after proto edits and include both Go/OpenAPI and TypeScript generated outputs; `proto/gen/openapi.yaml` feeds the MCP server.

## Runtime Configuration

Deployment-supplied configuration (identity providers, instance settings) can be provisioned as protobuf-JSON files scanned from `/etc/secrets` after migration/seed; it shadows database state for the process lifetime. See `docs/configuration-provisioning.md`. Outbound webhooks have private-network (SSRF) protection; allowlist destinations via `--webhook-private-network-allowlist`.

## Verification Policy

- Run the narrowest relevant checks while iterating.
- Before finishing, run the checks that match the changed surface from "Change Routing".
- For docs-only changes, `git diff --check` is sufficient unless the docs include runnable examples that should be tested.
- If a required check cannot run locally (e.g. TestContainers needs Docker), report the reason and the exact command that remains.

## CI Reference

- Backend CI: Go 1.27.0, `go mod tidy -go=1.27.0`, golangci-lint v2.13.1, test groups `store`, `server`, `internal`, `other` (codecov on main).
- Frontend CI: Node 24, pnpm 11.0.1, `pnpm lint`, `pnpm test`, `pnpm build`.
- Proto CI: `buf lint` and `buf format` check.
- Upgrade smoke: Docker-based fresh-install + previous-stable upgrade tests on relevant PRs (`scripts/release_smoke_test.sh`).
- Releases: release-please (Go, prerelease `rc.1`), Docker images `neosmemo/memos` (Docker Hub) and `ghcr.io/usememos/memos`.
- Docker: `scripts/Dockerfile`, Alpine 3.21 runtime, non-root user via `entrypoint.sh`, port 5230, multi-arch amd64/arm64/arm/v7.
