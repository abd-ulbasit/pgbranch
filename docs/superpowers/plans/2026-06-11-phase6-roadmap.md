# pgbranch Phase 6 — GitHub story, test SDK, durability, diff

> **For agentic workers:** TDD task-by-task. Decisions binding; named tests are the contract. Commit per task (conventional commits); do NOT push — the orchestrator reviews and pushes.

**Goal (user-approved order):** Stage 2 (GitHub DX) → Stage 3 (test-suite SDK) → Stage 1 (durability) → Stage 4 (pgb diff).

**Context:** all infra is torn down; every stage must be verifiable with unit tests + local docker ITs (`PGBRANCH_IT=1`) and, where K8s matters, the kind cluster (`PGBRANCH_K8S_IT=1`, cluster `pgbranch-test`, `hack/kind-up.sh`).

---

## Stage 2 — GitHub story (ghook)

**Architecture decisions (locked):**

1. **Commit statuses.** ghook sets a commit status (context `pgbranch/branch`) on the PR head SHA for every handled event: `pending` ("creating branch <name>" / "resetting…") posted BEFORE the detached operation starts, then `success` ("branch <name> ready — connect via <proxyHost>") or `failure` (error string, truncated 140 chars) when it finishes. Closed events: no status (branch is gone; statuses on a closed PR don't matter). API: `POST /repos/{repo}/statuses/{sha}`. CI consumers can then gate on the status instead of psql retry loops. New GitHub client method `SetStatus(ctx, repo, sha, state, desc)`; failures logged, never fatal (same policy as comments).
2. **GitHub App auth.** Alternative to the PAT: `GHOOK_APP_ID` + `GHOOK_APP_PRIVATE_KEY` (PEM, or `GHOOK_APP_PRIVATE_KEY_FILE`). When set (mutually exclusive with `GHOOK_GITHUB_TOKEN`), ghook mints installation tokens: RS256 app JWT (iss=app id, 9 min expiry) → `POST /app/installations/{id}/access_tokens`; installation id read per-delivery from the webhook payload's `installation.id` field (add to payload struct). Cache the installation token until 5 min before expiry (mutex-guarded). **No new dependencies**: hand-roll the JWT (crypto/rsa + crypto/sha256 + base64url of header/claims — ~40 lines) in `internal/ghook/githubapp.go`. The GitHub client gains a token-provider func instead of a static token string (PAT mode = constant provider).
3. **Live comment.** Replace EnsureComment-once with upsert: find the marker comment (existing pagination logic), PATCH it if present else POST. Body now includes a status line table: branch name, state (creating/ready/reset @ short-sha/destroyed), psql connect string, TTL/expiry if set. On `closed`, the comment is updated to "branch destroyed when the PR closed" (connect string removed). Marker stays the same.
4. **Chart/env:** `ghook.appId`, `ghook.appPrivateKey` (Secret), wired as GHOOK_APP_ID / GHOOK_APP_PRIVATE_KEY; values.yaml comments explain PAT vs App. docs/github-app.md rewritten around App-first setup (manifest URL flow described manually), PAT as the quick path.

**Tests (contract):** unit — JWT shape (header/claims decode, RS256 verifies with the public key), installation-token caching (fake API counts mint calls), SetStatus request shape, comment upsert (POST when absent / PATCH when marker present), payload installation.id parse, statuses around ensure/destroy (fake API records: pending→success order; failure path on branchd 500). Env validation (App vars XOR token). Helm template test in internal/deploy for the new env wiring. Run full `go test ./...`.

## Stage 3 — test-suite SDK

**Architecture decisions (locked):**

1. **Go SDK**: new public package `pgbranchtest/` (top-level, importable as `github.com/abd-ulbasit/pgbranch/pgbranchtest`). API:
   ```go
   func Acquire(t testing.TB, opts ...Option) *Branch
   type Branch struct { Name, Host string; Port int; User, Password, Database, DSN, ProxyDSN string }
   ```
   Options: `WithSource(string)` (default env PGBRANCH_TEST_SOURCE, else "main"), `WithTTL(time.Duration)` (default 1h — safety net; explicit destroy in t.Cleanup is primary). Server/token from PGBRANCH_SERVER/PGBRANCH_TOKEN (skip the test with t.Skip if unset — SDK is integration-only by nature). Branch name: `t-<sanitized test name>-<6 hex crypto/rand>`, ≤41 chars (sanitize like ghook; truncate test name from the left to keep the random suffix). Acquire registers t.Cleanup(destroy), is parallel-safe. Uses internal/apiclient via a thin re-export (apiclient is internal — either move needed types or have pgbranchtest do its own minimal HTTP calls; **decision: minimal self-contained HTTP client inside pgbranchtest, no internal imports** — keeps the public package dependency-free and the internal API free to change).
2. **JS SDK**: `sdk/js/` — zero-dependency npm package `pgbranch-test` (package.json, index.mjs + index.d.ts): `await acquire({source, ttlSeconds, server, token})` → `{branch, host, port, user, password, database, proxyDsn, destroy()}` using global fetch (Node 18+). Tests with `node --test` against a stub HTTP server (no network). Not published to npm this phase — `npm pack` must succeed; publishing documented as a manual step.
3. **GitHub Action**: `action/` (composite, `action.yml` + `entrypoint.sh`): inputs server/token/source/name/ttl, outputs branch/host/port/database (password is NOT an output — workflows already hold it as a secret); creates the branch via curl against the REST API, waits for ready state by polling GET (60×5s). A companion `action/destroy/action.yml` deletes by name. Both usable as `uses: abd-ulbasit/pgbranch/action@main`. Shell logic factored into entrypoint.sh testable by a bash-run unit script invoked from a Go test (exec the script against a httptest stub via env-provided server URL).
4. **Docs**: docs/testing.md ("a real database for every test, in seconds": Go quickstart, JS quickstart, Action example, TTL/parallelism/cleanup semantics, naming); README section "Branches in your test suite" linking it; mkdocs nav.

**Tests (contract):** Go unit (name sanitize/truncate, option defaults, skip-without-env) + **real IT** `PGBRANCH_IT=1` in pgbranchtest: spin local branchd? No — branchd daemon isn't started by ITs; instead the IT starts the API server in-process (internal/api srv on httptest listener backed by a real engine+docker driver, as api ITs already do), points PGBRANCH_SERVER at it, runs Acquire from a subtest, verifies connect+write+cleanup destroys the branch. JS: node --test with stub server asserting request shapes + destroy. Action: script test via stub as described. Full suite green.

## Stage 1 — durability

**Architecture decisions (locked):**

1. **Registry on a PVC.** Chart gains `persistence: {enabled, size (default 1Gi), storageClass}`. When enabled, branchd's state dir is a PVC mount instead of hostPath `<dataRoot>/state`; when `storage.mode=csi`, templates DEFAULT persistence.enabled=true (overridable). branchd code unchanged.
2. **Per-branch credentials.** Engine option `RotateCredentials bool` (branchd flag `--rotate-branch-credentials`, chart `rotateBranchCredentials`, ghook needs nothing — the comment never shows passwords). When on: after masking, run in-branch `ALTER ROLE <user> WITH PASSWORD '<32-hex crypto/rand>'` via the existing psql-exec path; store the password on the branch row (registry v7: `branches.password TEXT NOT NULL DEFAULT ''`); API Branch gains `password` (omitempty, only populated in rotate mode); `pgb connect` includes it in the printed DSN; reset re-rotates (new password — documented). Default OFF (static-credential preview flows like Vercel depend on inherit mode; trade-off documented in docs/architecture.md).
3. **Docs re-lead.** docs/kubernetes.md restructured to recommend CSI mode + persistence for production-ish use, hostpath positioned as the single-node/dev mode (with the node-rollover data-loss warning from docs/eks.md repeated); values.yaml comments updated to match.

**Tests (contract):** registry v7 migration + round-trip; engine fake-driver: rotate mode execs ALTER ROLE after masking and stores pw, inherit mode doesn't; API includes password only in rotate mode; helm template tests: persistence PVC rendering (on/off, csi default-on); docker IT: create with rotation on → connect with NEW password works, OLD source password fails; reset → old branch pw stops working, new one works.

## Stage 4 — `pgb diff`

**Architecture decisions (locked):**

1. **Mechanism.** `DiffBranch(ctx, name)`: create an internal throwaway branch `diff-<6hex>` from the SAME source generation the target branch was cloned from (its recorded source volume/layer chain base — NOT the current generation), wait ready, then in helper containers run `pg_dump --schema-only --no-owner --no-acl` against both instances and capture per-table row estimates (`SELECT relname, reltuples::bigint FROM pg_class JOIN pg_namespace ... WHERE relkind='r' AND nspname NOT IN ('pg_catalog','information_schema')`); destroy the throwaway (saga-compensated). Diff computed host-side: unified diff of the two schema dumps (use a small pure-Go diff: implement simple LCS line diff in internal/diffutil — no new deps) + table list with `{table, base_rows, branch_rows, delta}` (estimates; say so in output).
2. **Surface.** Engine `DiffBranch` returns `{SchemaDiff string, Tables []TableDelta}`; API `GET /v1/branches/{name}/diff` (long request — document ~5-10s); CLI `pgb diff NAME` prints the schema diff (colorless) then the table-delta table, "(row counts are planner estimates)" footer. No ghook integration this phase.
3. **Scope guards.** Refuse while branch not ready. zfs/csi work identically through the same engine paths (throwaway branch is just a branch); no special-casing.

**Tests (contract):** diffutil unit tests (insert/delete/change hunks, identical→empty); engine fake-driver: throwaway created from the branch's OWN base generation and destroyed even when dump fails; API route + CLI rendering golden test; docker IT: branch → apply DDL+rows in branch → diff shows CREATE TABLE/ALTER lines and positive delta; source untouched.

---

### Sequencing for workers
One stage per worker, in order 2 → 3 → 1 → 4. Each stage: full `go build ./... && go vet ./... && gofmt -l .` clean and `go test ./...` green, plus that stage's docker ITs run for real (`PGBRANCH_IT=1 go test ./<pkg> -run <New> -v`). Update README/docs within the stage. Commit per task; no pushes.
