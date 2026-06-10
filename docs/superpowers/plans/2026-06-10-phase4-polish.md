# pgbranch Phase 4 — Polish Implementation Plan

> **For agentic workers:** TDD task-by-task. Decisions binding; named tests are the contract.

**Goal:** data-masking hooks, embedded web UI with disk usage, published benchmarks, experimental ZFS backend, docs curation.

**Architecture decisions (locked):**

1. **Masking hooks** are per-source ordered SQL, stored in the registry (v4 migration: `mask_scripts(source_id TEXT, ord INTEGER, name TEXT, sql TEXT, PRIMARY KEY(source_id, ord))`). Set via `PUT /v1/sources/{name}/mask` (JSON array of {name, sql}) and `pgb source set-mask NAME file1.sql [file2.sql...]` (CLI reads files; order = argv). Applied by the engine inside the create/reset saga AFTER readiness, BEFORE MarkBranchReady: engine connects with pgx to `info.Host:port` using the source's recorded user + a password the engine doesn't have… **decision:** masking runs in-container via `Exec` with `psql -v ON_ERROR_STOP=1 -U <user> -d <db> -c <stmt>` per script (peer/local auth inside the container avoids password handling). Scripts that error → branch `failed` (unwound). New IT: branch of a seeded source with an `UPDATE` masking script; verify masked data in branch, original in source.
2. **Web UI**: embedded (go:embed) static SPA served by branchd at `/ui/` — **no build toolchain**: one `index.html` + vanilla JS + minimal CSS (dark, monospace, decent). Features: sources list (gen, state), branch table (name, source, state, host:port, expires-in countdown, disk usage), create-branch form (name, source, ttl), destroy/reset buttons, auto-refresh 5s. Talks to the existing REST API with the token pasted once into a field (stored in localStorage; UI is same-origin so no CORS work).
3. **Disk usage**: `runtime.Driver.RunHelper` signature changes to return `(output string, err error)` (captured stdout/stderr, both drivers; all call sites updated — engine ignores output except where needed). New driver-level helper in engine: `BranchUsage(ctx, name)` runs `du -sb` on the branch rw volume via helper (docker: alpine du; kube: helper pod du). Exposed as `GET /v1/branches/{name}/usage` → `{"bytes": N}`; UI shows MB/GiB; CLI `pgb branch ls` gains SIZE column **only with `--usage`** flag (it's a helper-pod roundtrip — not free).
4. **Benchmarks**: `hack/benchmark.sh` — sizes via pgbench scale factors targeting ~1 GiB, ~5 GiB, ~10 GiB on the Colima VM; per size: seed time (`pg_basebackup`), branch create p50 of 5 runs (CLI timing), branch rw overhead after creation (usage API), write-amplification probe (UPDATE 1% rows, usage delta). Results → `docs/benchmarks.md` (table + methodology + hardware: Apple Silicon, Colima VM specs) + README excerpt with the headline number. Script must check free disk and skip sizes that don't fit.
5. **ZFS backend (experimental, honest)**: `internal/cow` gains a `Backend` enum surfaced as branchd `--cow overlay|zfs` (default overlay). ZFS mode: volumes are ZFS datasets under a configured pool/prefix (`--zfs-dataset tank/pgbranch`); source seed targets the dataset mountpoint; branch create = `zfs snapshot` + `zfs clone` (instant, no overlay entrypoint — branch entrypoint reduces to perms+pid-cleanup+exec); destroy = `zfs destroy`. All zfs commands run through the runtime driver as privileged helpers (docker: `--privileged` + `/dev/zfs`; kube: privileged pod) — add `Privileged bool` + `HostDevices []string` to HelperSpec (docker) / privileged securityContext (kube). Tests: command-construction unit tests + an IT gated `PGBRANCH_ZFS_IT=1` (SKIPPED on this machine — no ZFS; document manual verification steps in docs/zfs.md). README labels it **experimental: unit-tested, manual-verification instructions provided**.
6. **Docs curation**: `mkdocs.yml` (material theme) over docs/: index (from README), quickstart, kubernetes, github-app, benchmarks, zfs, architecture (move the spec's architecture section into docs/architecture.md, updated to as-built). No CI, no hosting — buildable with `pip install mkdocs-material && mkdocs serve`. Roadmap in README: P4 ✅; add "Future" line (branch-from-branch layer DAG, multi-node CSI storage, TLS).

---

### Task A: masking hooks
Registry v4 + CRUD (`SetMaskScripts`, `GetMaskScripts`), engine saga step `applyMasking` (Exec psql per script; reset path re-applies), API PUT + GET mask endpoints (+ api_test), apiclient + CLI `source set-mask` (+ tests), Docker IT: masked branch vs untouched source (extend engine IT or new test). 
Commit: `feat(masking): per-source SQL masking hooks on branch creation`

### Task B: RunHelper output, usage endpoint, web UI
Driver signature change + both drivers capture output (docker: existing logs path; kube: pod logs on success too) + all call-site updates; engine.BranchUsage; API usage endpoint + test; embedded UI (decision 2) served at /ui/ with httptest smoke test (HTML served, no auth on static assets; API calls carry token); CLI `branch ls --usage`. Docker IT asserts usage > 0 for a branch with writes.
Commit: `feat(ui): embedded web UI and branch disk-usage endpoint`

### Task C: benchmarks
`hack/benchmark.sh` (set -euo pipefail, disk-check, JSON intermediate, markdown emit), run it for real at 1 GiB and 5 GiB (10 GiB only if disk allows), write docs/benchmarks.md with the REAL measured numbers + methodology + hardware, README headline update.
Commit: `docs(bench): measured branching benchmarks`

### Task D: ZFS backend (experimental) + docs site
Decision 5 implementation + unit tests + docs/zfs.md; mkdocs.yml + docs/architecture.md (as-built); README final pass (P4 ✅, Future line, docs links). Full suites: unit, PGBRANCH_IT=1, PGBRANCH_K8S_IT=1 — all green, tree clean.
Commit: `feat(cow): experimental zfs backend` + `docs: phase 4 complete — docs site, architecture`
