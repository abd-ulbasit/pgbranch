# Phase 7 — Road to v1 Implementation Plan

> **For agentic workers:** TDD task-by-task. Decisions below are binding; named
> tests are the contract. Commit per logical chunk (conventional commits); do
> NOT push — the orchestrator reviews and pushes. Build tasks in order A→B→C→D.

**Goal:** ship the four operational-trust items that earn pgbranch a 1.0:
observability, a leak-proof reconcile loop, an authz model, and HA via leader
election. Spec: `docs/superpowers/specs/2026-06-13-phase7-road-to-v1-design.md`.

**Grounding (current code):** registry latest migration `migrateV7` (new ⇒
v8); API auth is a single bearer token in `internal/api/middleware.go`
(`Server.auth`, `api.New(eng, reg, token)`); branchd runs `eng.Reconcile(ctx)`
once at startup and `eng.RunReaper(ctx, interval, …)` in `cmd/branchd/main.go`;
API served via `http.Server{Handler: api.New(...).Handler()}` on `apiAddr`.

---

## Task A — Observability (metrics + real readiness)

**Architecture decisions (locked):**
1. **Dependency:** use `github.com/prometheus/client_golang/prometheus` +
   `promhttp` (add to go.mod). It's the standard; no hand-rolling.
2. **Endpoint:** `GET /metrics` on the **API mux**, NOT behind the bearer
   token (Prometheus scrapers don't auth; metrics are non-sensitive). Add it in
   `internal/api/server.go` routing next to `/healthz`.
3. **Registry of metrics** in a new `internal/metrics` package (so engine +
   api + proxy can share without import cycles): 
   - `branches_total{state}` gauge, `sources_total{state}` gauge (refreshed by
     a collector that queries the registry on scrape — implement a
     `prometheus.Collector` backed by `registry` count queries).
   - `branch_op_duration_seconds{op}` histogram (op = create|reset|destroy|
     from_branch|diff), observed in the engine sagas.
   - `branch_op_errors_total{op}` counter.
   - `masking_duration_seconds` histogram; `reaper_runs_total`,
     `reaper_reaped_total` counters; `reconcile_runs_total`,
     `reconcile_actions_total{action}` counters (action = fail_stuck|
     remove_orphan_container|gc_layer|gc_volume).
   - `inflight_ops` gauge (inc/dec around saga entry/exit).
4. **Real readiness:** `GET /readyz` (new) returns 200 only when the registry
   is reachable (a trivial query succeeds) and the driver responds (a cheap
   `ListManaged`/ping); `/healthz` stays liveness (process up). Chart probes:
   liveness→/healthz, readiness→/readyz.
5. Engine gets a `*metrics.Metrics` (nil-safe: a nil receiver no-ops so unit
   tests without metrics keep working) wired from branchd; saga code calls
   `m.ObserveOp("create", dur)` etc.

**Tests (contract):** `internal/metrics` unit — collector reports correct
gauges from a seeded registry; histogram/counter helpers are nil-safe.
`internal/api` — `/metrics` returns 200 without auth and includes
`pgbranch_branches_total`; `/readyz` 503 when registry closed, 200 when open.
Engine unit — a create increments `branch_op_duration_seconds` count (use a
test registry). Full `go test ./...` green.

Commits: `feat(metrics): prometheus registry + collector`,
`feat(api): /metrics and /readyz`, `feat(engine): observe saga ops`,
`feat(chart,docs): scrape config + readiness probe`.

## Task B — Reconcile loop + leak-proof GC

**Architecture decisions (locked):**
1. **Periodic reconcile:** branchd runs `eng.Reconcile` on a ticker
   (`--reconcile-interval`, default 60s) in addition to startup. Fold the TTL
   reaper INTO reconcile (one loop): each pass (a) reaps expired branches,
   (b) fails `creating`/`resetting` rows older than a deadline
   (`--stuck-timeout`, default 10m) and cleans their resources, (c) removes
   managed containers/pods with no `ready`/`creating` registry row, (d) GCs
   frozen layers with refcount 0 and rw volumes with no owning branch. Keep
   `RunReaper` as a thin alias that calls the unified loop (back-comp) or
   remove its separate goroutine in branchd — pick one, don't run both.
2. **GC safety:** only ever touch resources carrying the `pgbranch.managed`
   label / name prefix; never delete a volume referenced by any live branch's
   layer chain (re-check `LayerChain` + rw-volume ownership before delete).
   Dry-runnable: the core returns a `ReconcilePlan` (list of intended actions)
   that callers can either apply or just report.
3. **CLI:** `pgb doctor` → prints the `ReconcilePlan` (drift: stuck rows,
   orphan containers, dangling layers/volumes) read-only, non-zero exit if
   drift found; `pgb gc` → applies it. Both work in server mode (new REST:
   `GET /v1/reconcile/plan`, `POST /v1/reconcile`) and local mode.
4. Emits the Task-A reconcile metrics + structured logs per action.

**Tests (contract):** engine fake-driver — stuck `creating` row past deadline
→ failed + volume removed; managed container with no row → removed; frozen
layer refcount 0 → GC'd, but a layer still in a live branch's chain → kept; rw
volume with no branch → removed, in-use → kept; `ReconcilePlan` lists actions
without applying when dry. Docker IT (`PGBRANCH_IT=1`, prefix `gc-`): create a
branch, manually `docker rm` its registry row’s container? — instead: leave a
stray managed volume + a stuck row, run reconcile, assert gone; TTL branch
reaped by the loop. CLI: `pgb doctor` exit code + output on a drifted fake.

Commits: `feat(engine): unified periodic reconcile + GC plan`,
`feat(api,cli): pgb doctor / pgb gc`, `feat(branchd): reconcile ticker`.

## Task C — Security: authz model + scoped credentials

**Architecture decisions (locked):**
1. **Registry v8:** `api_tokens(id, name, token_hash, role, created_at)` —
   store only a SHA-256 hash of the token, never the plaintext. Roles:
   `admin` (all), `operator` (branch create/reset/destroy + all reads),
   `viewer` (reads only). 
2. **Auth middleware** (`internal/api/middleware.go`): look the presented
   bearer up by hash; attach its role to the request context; per-route
   required-role check (a small table mapping method+path → min role).
   **Back-compat:** the `PGBRANCH_TOKEN` env still works and is treated as a
   built-in `admin` (not stored; checked first by constant-time compare).
3. **CLI:** `pgb token create <name> --role operator` (prints the token once),
   `pgb token ls` (names/roles, never the token), `pgb token revoke <name>`.
   REST: `POST/GET/DELETE /v1/tokens` (admin-only).
4. **Proxy TLS + scoped SA (docs/chart):** make `--pg-tls-*` first-class in
   the chart (`proxy.tls.certSecret`), document cert-manager issuance; ship a
   namespaced `Role` (not cluster-admin) example for CI/preview deployers in
   `deploy/` + docs/usage.md (supersedes the cluster-admin kubeconfig recipe).

**Tests (contract):** registry v8 migration + token hash round-trip + lookup.
api unit — viewer token: GET allowed, POST/DELETE 403; operator: branch ops
allowed, token endpoints 403; admin + legacy env token: all allowed; unknown
token 401. CLI token create/ls/revoke against a fake server. Helm template
test: proxy TLS secret wired; scoped Role renders. Full suite green.

Commits: `feat(registry): v8 api_tokens`, `feat(api): role-based authz`,
`feat(cli): pgb token`, `feat(chart,docs): proxy TLS + scoped deployer role`.

## Task D — HA: leader election (do last; riskiest)

**Architecture decisions (locked):**
1. **Leader election** via `k8s.io/client-go/tools/leaderelection` using a
   coordination.k8s.io **Lease** in the pod's namespace (name
   `pgbranch-branchd`, identity = pod name). Only the leader runs the
   reconcile loop and accepts **mutating** `/v1` requests; non-leaders return
   `503` with a `Location`-style hint (or just 503 + "not leader") on mutating
   routes and still serve `/healthz`, `/readyz`, `/metrics`, and read-only
   GETs from their own registry handle (SQLite on the shared RWO PVC — reads
   are safe; the leader is the only writer).
2. **Enablement:** `--leader-elect` flag (off by default → single-instance
   behavior unchanged, so docker/local is untouched). Chart: `replicaCount`
   (default 1) and `leaderElection.enabled`; when replicas>1 the chart sets
   `--leader-elect`, adds the RBAC for leases (coordination.k8s.io
   leases: get/create/update in the namespace), and keeps the registry PVC
   (RWO) — note pods co-schedule to the PVC's node (or use RWX/CSI).
3. **Failover behavior:** losing leadership cancels the reconcile loop and
   flips the mutating-gate closed within the lease's renew deadline; gaining it
   runs a reconcile pass immediately (converge any drift from the gap).

**Tests (contract):** unit — a leader-state gate (atomic bool) gates mutating
handlers (mutating route returns 503 when not leader, 200 when leader; GET/
healthz/metrics unaffected); the election callbacks flip the gate and
start/stop the reconcile loop (drive with a fake elector). kind IT
(`PGBRANCH_K8S_IT=1`): deploy 2 replicas with `--leader-elect`; assert exactly
one holds the Lease and serves mutations; delete the leader pod; assert the
other acquires the Lease and a branch create succeeds against the service
within the renew deadline. `/metrics` scrapeable on both.

Commits: `feat(branchd): leader election`, `feat(api): mutating-route leader
gate`, `feat(chart): replicas + leader-election RBAC`, `docs: HA guide`.

---

### Final (after D)
Update README (Operations/Observability/HA/Security sections), docs nav,
benchmarks unaffected; tag `v1.0.0-rc.1` and a GitHub pre-release summarizing
the trust bar. Tier 2/3 (snapshot seeding, data diff, credential helper,
Marketplace Action, API-compat policy) become Phase 8.
