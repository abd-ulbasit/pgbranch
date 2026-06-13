# Phase 8 — adoption Implementation Plan

> **For agentic workers:** TDD task-by-task. Decisions binding; named tests are
> the contract. Commit per logical chunk (conventional commits); do NOT push —
> the orchestrator reviews, runs ITs, and pushes. Build A→B→C→D.

**Goal:** review-on-PR, a credential helper that reconciles rotation with
static config, CI coverage of kind/CSI/HA, and a versioned Action + API-compat
promise. Spec: docs/superpowers/specs/2026-06-13-phase8-adoption-design.md.

**Grounding:** `apiclient.DiffBranch(ctx,name) (*engine.DiffResult,error)`
exists. ghook `GitHub.UpsertComment(repo,number,body)` finds/updates ONE marker
`<!-- pgbranch -->` (internal/ghook/github.go); a second comment type needs a
marker param. Composite Action at `action/` + `action/destroy/`. CI
(.github/workflows/ci.yml) has unit/integration/helm; kind is deliberately
excluded. Docker driver already publishes an ephemeral host port (HostPort "")
and reads it back (internal/runtime/docker.go) — the Phase-7 "address already
in use" flake is a rare ephemeral-bind race to retry.

---

## Task A — `pgb diff` on the PR + bounded data sample

**Locked decisions:**
1. **Generalize ghook comments to a marker param.** Refactor `UpsertComment`/
   the finder to take a `marker string` (keep the connect comment on
   `<!-- pgbranch -->`). Add a second marker `<!-- pgbranch-diff -->`.
2. **ghook diff comment** (opt-in `GHOOK_DIFF_ON_PUSH=true`, env + chart
   `ghook.diffOnPush`, Config.DiffOnPush): in the detached worker, after the
   branch is ready (opened/synchronize), call `apiclient.DiffBranch`; render a
   comment (schema diff in a ```diff fence, truncated to ~3000 chars with a
   "(truncated)" note; the table-delta table) and upsert it under the diff
   marker. On `closed`, leave the diff comment (or update to "branch
   destroyed") — keep simple: skip diff handling on closed. Non-fatal on error
   (log, don't fail the op). Only when a GitHub client is configured.
3. **Bounded data sample**: `DiffResult` gains `Tables[i].SampleRows []map[string]any`
   (omitempty) populated ONLY when diff is asked with data sampling. Engine
   `DiffBranch(ctx, name, opts...)` gains a `WithDataSample(n int)` option
   (default off). For each table whose branch row-estimate > base, run inside
   the branch instance: `SELECT to_jsonb(t) FROM (<branch rows> EXCEPT <base
   rows>) ...` — implement as: get PK columns from the branch, then
   `SELECT to_jsonb(t.*) FROM <table> t WHERE (pk...) NOT IN (SELECT pk... FROM
   <base table via dblink? NO — base is a different instance)`. Since base and
   branch are separate instances, do it host-side: pull up to N rows by PK from
   each and diff in Go (branch PKs not in base, capped at N). Keep it bounded
   and read-only. API `GET /v1/branches/{name}/diff?data=N`; CLI `pgb diff NAME
   --data [--sample N]` renders the schema diff, table deltas, then up to N
   sample rows per changed table. Tables with no PK: skip sampling (note in
   output).
4. Strip-dump-noise (existing) still applies to the schema diff.

**Tests:** ghook unit — diff-comment upserted under its own marker when
DiffOnPush + GitHub configured, absent otherwise, connect comment unaffected;
fake API returns a DiffResult. engine unit — WithDataSample off → no SampleRows;
PK extraction + branch-only row selection logic (fake ExecOutput returning rows).
docker IT (`PGBRANCH_IT=1`, `diffd-` prefix): branch → INSERT rows with PKs →
`DiffBranch(WithDataSample(20))` returns the new rows as samples; a no-PK table
is skipped. Full `go test ./...` green.

Commits: `feat(ghook): diff comment on push (opt-in)`, `feat(engine,api,cli):
bounded data sample in pgb diff`.

## Task B — credential helper (`pgbranchconnect` + JS)

**Locked decisions:**
1. **Go package** `pgbranchconnect` (top-level, public): `Resolve(ctx, opts)
   (DSN string, err error)` and `ResolvePool`-friendly fields. opts: Server
   (PGBRANCH_API), Token (PGBRANCH_TOKEN, viewer role suffices), Branch (name)
   OR Ref (git ref → sanitized like ghook), ProxyHost (host:port the app dials)
   optional (defaults to the API host + 6432). It GETs `/v1/branches/<name>`,
   reads host/port/user/database/password (password present in rotate mode;
   empty → inherit, the helper then requires a Password opt or PGPASSWORD),
   builds and returns the DSN. Self-contained HTTP (no internal imports), like
   pgbranchtest.
2. **JS package** `sdk/js-connect` (or extend `sdk/js`): zero-dep `resolve({
   server, token, branch|ref, proxyHost })` → `{ dsn, host, port, user,
   database }` via global fetch.
3. **Wire the demo app**: pgbranch-demo `_db.js` uses the JS helper (fetch the
   branch's creds from PGBRANCH_API with a viewer token) instead of a static
   PGPASSWORD — so the demo works with rotation ON. (Do this in the pgbranch-
   demo repo; if not accessible, ship the helper + a docs example only.)
4. Docs: docs/usage.md credential-modes section gains the helper as the
   "rotation + static config" answer; testing.md/preview sections reference it.

**Tests:** Go unit against an httptest stub (resolves DSN incl. rotated
password; ref sanitization; viewer-token path; inherit-mode requires Password).
JS `node --test` against a stub. `go build ./...`, `npm pack --dry-run` for the
JS package.

Commits: `feat(pgbranchconnect): credential-resolving connect helper`,
`feat(sdk/js): connect helper`, `docs: credential helper`.

## Task C — CI: kind / CSI / HA-failover + flaky-port fix

**Locked decisions:**
1. **New CI job `kube`** in .github/workflows/ci.yml: on a runner, install kind
   (helmfile/kind action or `hack/kind-up.sh` + `hack/kind-csi-up.sh`), build
   + `kind load` the branchd/ghook images, then run
   `PGBRANCH_K8S_IT=1 PGBRANCH_CSI_IT=1 go test ./internal/runtime/ ./internal/deploy/ ./internal/engine/ -timeout 30m` (the kind/CSI/HA-failover suites). Allow it
   to be slower; keep unit/integration/helm jobs unchanged. If a full CSI
   install proves flaky on the runner, scope to hostpath + HA-failover first
   and note CSI as best-effort — but ATTEMPT CSI.
2. **Flaky host-port fix** (internal/runtime/docker.go): the ephemeral bind can
   rarely race ("address already in use"); on StartBranch container-start
   failure matching that string, retry up to 3x (new container name suffix is
   not needed; recreate/start). Keep it minimal and unit-test the retry via a
   fake docker client returning the error once then succeeding — if the docker
   client isn't easily fakeable, gate the retry logic behind a small helper
   func and unit-test that helper.
3. README CI badge stays; document the kube job in docs.

**Tests:** the kube CI job going green IS the contract (orchestrator watches
it). Unit test for the port-retry helper. Existing jobs unaffected.

Commits: `ci: kind job (k8s/CSI/HA-failover ITs)`, `fix(runtime): retry branch
start on ephemeral-port race`.

## Task D — versioned Action + API-compat policy

**Locked decisions:**
1. **Action hardening**: `action/action.yml` + `action/destroy/action.yml` gain
   `branding` (icon/color), clear input descriptions + required flags, and
   entrypoint input validation (fail fast on missing server/token/source).
   Add a `v1` lightweight tag after merge (orchestrator tags). Usage section in
   docs/testing.md: `uses: abd-ulbasit/pgbranch/action@v1`.
2. **API-compat test**: `internal/api/compat_test.go` — golden JSON locking the
   wire shape (field names/types) of: POST/GET/list branches, GET diff, GET
   reconcile/plan, token responses. Marshal representative structs (api.Branch,
   engine.DiffResult, engine.ReconcilePlan, token list item) and compare to
   committed golden files in internal/api/testdata/; a field rename/removal
   fails the test. `docs/api.md`: the `/v1` stability promise (additive-only;
   breaking changes ⇒ `/v2`), and the policy. mkdocs nav + README docs list.

**Tests:** the golden compat test; action entrypoint validation tests
(extend internal/actiontest). Full suite green.

Commits: `feat(action): branding + input validation`, `test(api): /v1 wire
compatibility golden`, `docs: API stability policy`.

---

### Final (after D)
README/docs refreshed; tag `v1.0.0-rc.2` (or `v1.0.0` if you're satisfied),
release notes summarizing Phase 8. Cloud snapshot seeding remains a documented
future item.
