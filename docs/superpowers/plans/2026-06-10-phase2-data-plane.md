# pgbranch Phase 2 — Data Plane & API Implementation Plan

> **For agentic workers:** Execute task-by-task with TDD. This plan locks design decisions and contracts; reference code is intentionally lighter than Phase 1's — the tests named per task are the contract.

**Goal:** `branchd` daemon = REST API + embedded Postgres wire-protocol router (`pgproxy`) + TTL reaper, plus branch reset and generation-based source refresh.

**Architecture decisions (locked):**

1. **One daemon, two listeners.** `branchd` (new `cmd/branchd`) serves REST on `:7070` and the Postgres router on `:6432` in one process, sharing one engine/registry. Run via `pgb serve` is NOT added; `branchd` is its own binary.
2. **Routing key = database-name suffix.** Clients connect with `dbname@branch` (e.g. `psql "host=... port=6432 dbname=postgres@pr-42 user=postgres"`). The router reads the startup message, splits the database param on the **last** `@`, looks up the branch's host port, rewrites the database param back to the real dbname, dials `127.0.0.1:<branchPort>`, replays the (rewritten) startup message, then does transparent bidirectional copy — SCRAM auth flows untouched between client and backend. `SSLRequest` (8-byte magic 80877103) is answered with `'N'` (no TLS in P2; client proceeds plaintext or disconnects). `CancelRequest` (80877102) connections are closed silently. A database param without `@` gets a Postgres-style ErrorResponse (`3D000` invalid_catalog_name, "pgbranch: connect with dbname@branch"). Unknown branch / branch not ready → ErrorResponse `3D000` with a clear message. Use `github.com/jackc/pgx/v5/pgproto3` for message codecs.
3. **CLI gains server mode.** If `PGBRANCH_SERVER` env or `--server` flag is set (e.g. `http://localhost:7070`), CLI commands call the REST API (new thin client in `internal/apiclient`); otherwise they embed the engine as today. SQLite is single-writer — docs state: don't run local-mode CLI while branchd is running; server mode is the supported combo.
4. **Auth = single bearer token.** `PGBRANCH_TOKEN` env on branchd (required, refuse to start without it); REST requests need `Authorization: Bearer <token>`. CLI server mode reads the same env.
5. **TTL.** `branches.expires_at TEXT NOT NULL DEFAULT ''` (RFC3339). `pgb branch create --ttl 24h` → engine computes expiry. Reaper goroutine in branchd ticks every 30s: `DestroyBranch` every ready/failed branch with `expires_at != '' AND expires_at < now`. Registry gains `ListExpiredBranches(now)`.
6. **Reset.** `Engine.ResetBranch(ctx, name)`: transition ready→resetting→ready. Stop+remove container, remove rw volume, then re-run the create saga's resource steps on the SAME registry row (new container id/port). New states: `resetting` legal from `ready`, to `ready`/`failed`.
7. **Source generations (refresh).** `sources` gains `generation INTEGER NOT NULL DEFAULT 1`; volume naming becomes `pgbranch-src-<name>-g<N>` (`cow.SourceVolumeName(name, gen)`). `branches` gains `source_volume TEXT` recording the volume they were created from (existing branches keep their base). `Engine.RefreshSource(ctx, name, password)`: seed gen N+1 volume; on success bump `sources.generation` + volume; old generation volumes are GC'd when no live branch references them (checked on refresh and on branch destroy). Existing P1 rows: migration backfills `source_volume` from the source's current volume, and renames nothing (gen-1 volume name = legacy `pgbranch-src-<name>` is kept as-is for backward compat: `SourceVolumeName(name, 1)` returns the legacy name).
8. **Schema migrations.** `PRAGMA user_version` versioned migrations in `internal/registry/schema.go`; v1 = P1 schema, v2 = the columns above + sources name partial-unique fix below.
9. **Rough-edge fix from P1:** `sources.name` UNIQUE constraint blocks retry after failed seed. v2 migration: drop table-level UNIQUE (SQLite: recreate table) → partial unique index `WHERE state != 'failed'`; add `Engine.RemoveSource(ctx, name)` (refuses while live branches exist) + `pgb source rm`.

**Out of scope (P3/P4):** TLS on the router, K8s, GitHub App, masking, UI, ZFS, metrics endpoint (defer Prometheus to P4 polish).

---

### Task A: Registry v2 migration + new queries
Files: `internal/registry/schema.go`, `registry.go`, `registry_test.go`
- Versioned migrations via `PRAGMA user_version`; fresh DB applies v1+v2; existing P1 DB upgrades in place (test: open a DB created with v1-only schema fixture, assert upgrade preserves rows and backfills `source_volume`).
- New: `Branch.ExpiresAt`, `Branch.SourceVolume`, `Source.Generation`; `ListExpiredBranches(now string)`, `BumpSourceGeneration(id, newVolume string)`, `CountLiveBranchesBySource(sourceID)`, `CountLiveBranchesByVolume(volume string)`, `DeleteSource(id)`; `resetting` state + legal transitions (`ready→resetting`, `resetting→{ready,failed}`).
- Sources name partial-unique (failed rows don't block reuse); test create-fail-recreate.
Commit: `feat(registry): v2 schema — TTL, source generations, resetting state`

### Task B: cow + engine — generations, reset, TTL, remove
Files: `internal/cow/plan.go`, `internal/engine/{engine.go,saga.go}`, tests
- `cow.SourceVolumeName(name string, gen int)`: gen 1 → `pgbranch-src-<name>` (legacy), gen N>1 → `pgbranch-src-<name>-g<N>`. `PlanBranch` takes the source volume explicitly (it already does).
- `CreateBranch(ctx, name, source string, ttl time.Duration)` — records `source_volume` + `expires_at` (ttl 0 = never).
- `ResetBranch(ctx, name)` — per decision 6; unit tests: happy path, container-start failure → `failed` + resources unwound.
- `RefreshSource(ctx, name, password)` — per decision 7; unit tests: generation bump, old-volume GC when unreferenced, old volume KEPT while a live branch references it; branch destroy GCs an orphaned old-generation volume.
- `RemoveSource(ctx, name)` — refuses with live branches; removes volume + row.
- `ReapExpired(ctx, now time.Time) (destroyed []string, err error)` — used by branchd loop; unit test with fake clock string.
Commit: `feat(engine): branch reset, TTL reaping, source refresh with generations`

### Task C: pgproxy router
Files: `internal/pgproxy/{proxy.go,startup.go}`, `pgproxy_test.go` (unit: startup parsing/rewrite against recorded bytes), `pgproxy_it_test.go` (integration)
- `go get github.com/jackc/pgx/v5` (pgproto3 included).
- `type BranchResolver interface { ResolveBranch(name string) (hostPort int, err error) }` — implemented by a small adapter over the registry (ready branches only). Proxy depends on the interface (unit-testable with a fake).
- `Proxy.Serve(ctx, lis net.Listener)`; per-conn goroutine: handle SSLRequest→'N', parse StartupMessage, split dbname on last `@`, resolve, rewrite, dial backend, write rewritten startup, then `io.Copy` both directions (`errgroup`, half-close aware via `CloseWrite` on *net.TCPConn). ErrorResponse helper for refusals (encode via pgproto3.ErrorResponse, then close).
- Integration test: spin a real branch via engine (reuse `pgctl.StartSourcePG`), run proxy on an ephemeral port, `pgx.Connect` with `database=postgres@pr-1` through the proxy, query rows; also assert: missing `@` → server error mentioning pgbranch; unknown branch name → error; SCRAM auth (password) works through the relay.
Commit: `feat(pgproxy): postgres wire-protocol router with dbname@branch routing`

### Task D: branchd daemon — REST API + reaper + router wiring
Files: `internal/api/{server.go,handlers.go,middleware.go}`, `cmd/branchd/main.go`, tests `internal/api/api_test.go` (httptest against engine with fake driver)
- stdlib `net/http` + `http.ServeMux` (Go 1.22+ patterns, no framework). JSON errors `{"error": "..."}`.
- Endpoints (all under `/v1`, bearer-token middleware, `GET /healthz` unauthenticated):
  - `POST /v1/sources` {name,host,port,user,database,network,pg_version,password} → 201 source JSON (password accepted in body, never stored/returned)
  - `GET /v1/sources` | `DELETE /v1/sources/{name}` | `POST /v1/sources/{name}/refresh` {password}
  - `POST /v1/branches` {name,source,ttl_seconds} → 201 branch JSON (incl. port, proxy hint `dbname@branch`)
  - `GET /v1/branches` | `GET /v1/branches/{name}` | `DELETE /v1/branches/{name}` | `POST /v1/branches/{name}/reset`
- `cmd/branchd/main.go`: flags `--api-addr :7070 --pg-addr :6432`; requires `PGBRANCH_TOKEN`; runs `engine.Reconcile` at startup, then API server + pgproxy + 30s reaper ticker; graceful shutdown on SIGINT/SIGTERM (stop listeners, let in-flight finish, leave branch containers running — they're durable state, restart-policy unless-stopped).
- api_test.go covers: auth rejection (401 without/with-wrong token), source+branch happy path with fake driver, ttl propagation, reset endpoint, 404s, 409 on duplicate branch name.
Commit: `feat(branchd): REST API, embedded router, TTL reaper`

### Task E: CLI server mode + new commands
Files: `internal/apiclient/client.go` (+test against httptest server), `internal/cli/*` updates
- `apiclient.Client{BaseURL, Token}` with methods mirroring the API.
- CLI: global `--server` flag / `PGBRANCH_SERVER` env + `PGBRANCH_TOKEN`; when set, commands go through apiclient (engine path unchanged otherwise). New commands: `pgb branch create --ttl 24h`, `pgb branch reset NAME`, `pgb source rm NAME`, `pgb source refresh NAME`. `pgb connect` in server mode prints BOTH the direct port URL and the proxy form `postgres://user@<server-host>:6432/db@branch`.
Commit: `feat(cli): server mode, ttl/reset/refresh/rm commands`

### Task F: Phase 2 e2e + README update
Files: `internal/engine/phase2_it_test.go` (or `internal/api/api_it_test.go`), README
- Integration: start branchd in-process (api+proxy on ephemeral ports), seed source, create branch with 2s TTL via REST, connect through proxy, verify reaper destroys it (<45s wait); reset flow keeps name/port-changes documented.
- Full suites green: `go test ./...` and `PGBRANCH_IT=1 go test ./... -count=1 -timeout 25m`. No leaked managed containers/volumes.
- README: add branchd section (run, env vars, REST examples via curl, proxy connect example), update roadmap checkboxes.
Commit: `feat: phase 2 complete — branchd data plane` + `docs: README for branchd and proxy`
