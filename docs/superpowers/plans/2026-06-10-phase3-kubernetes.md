# pgbranch Phase 3 — Kubernetes Implementation Plan

> **For agentic workers:** TDD task-by-task. Decisions here are binding; named tests are the contract.

**Goal:** run pgbranch on Kubernetes: a `runtime.Driver` backed by pods, a Helm chart for the whole system, and a GitHub webhook service for branch-per-PR.

**Architecture decisions (locked):**

1. **Storage model: one designated storage node.** All CoW data lives under `/var/lib/pgbranch` (configurable `--data-root`) on a single node selected by name (`--node`). "Volumes" map to subdirectories (`<data-root>/<volume-name>`); helpers and branch pods mount them via `hostPath` and are pinned with `nodeName`. This is the honest dev/test-tool scope (documented limitation; multi-node via CSI is future work). Branch pods get CAP_SYS_ADMIN (overlay mount in-container — same entrypoint script as Docker).
2. **Address, not port.** Registry v3 migration: `branches.host TEXT NOT NULL DEFAULT '127.0.0.1'`. `runtime.ContainerInfo` gains `Host string`. Docker driver: `127.0.0.1` + mapped port (unchanged behavior). K8s driver: pod IP + 5432. `registry.MarkBranchReady(id, cid, host, port)` signature updated; pgproxy `RegistryResolver` returns `host:port` (proxy `BackendHost` override stays for tests). `pgb connect` prints host accordingly. All existing callers/tests updated.
3. **K8s driver mapping** (`internal/runtime/kube.go`, package-level parity with DockerDriver):
   - `CreateVolume` → mkdir via a short-lived helper pod (busybox/alpine, hostPath `<data-root>`, `mkdir -p`); volume labels tracked in an annotation file `<data-root>/<vol>/.pgbranch-labels.json` written by the helper (no etcd objects for volumes). `RemoveVolume` → helper pod `rm -rf`.
   - `RunHelper` → Pod with `restartPolicy: Never`, hostPath mounts translated from `Mount.Volume` to `<data-root>/<volume>`; wait for Succeeded/Failed (watch); on Failed, error includes last 20 log lines; always delete pod (background propagation). `HelperSpec.Network` is ignored on K8s (pod network reaches cluster + external hosts) — document.
   - `StartBranch` → Pod (not Deployment; branches are disposable, engine reconciles) with labels from spec, `nodeName` pin, SYS_ADMIN security context, postgres image, entrypoint override. Readiness = engine's existing `pg_isready` Exec path. `Inspect` returns pod IP/5432/labels/Running(phase==Running). `StopRemove` → delete pod (grace 30s). `ListManaged` → pods by label selector `pgbranch.managed=true,pgbranch.role=branch`.
   - `Exec` → client-go `remotecommand` SPDY executor.
   - `EnsureImage` → no-op (kubelet pulls).
   - Constructor `NewKubeDriver(kubeconfig string, namespace, nodeName, dataRoot string)`; in-cluster config when kubeconfig=="" and in-cluster env present.
   - Label name compliance: docker labels with dots are fine as K8s labels (`pgbranch.managed` is a valid label key). Branch **names** become part of pod names — sanitize: pod name `pgbranch-br-<name>` must be RFC1123; validate branch names at engine level NOW (new: reject names not matching `[a-z0-9][a-z0-9-]{0,40}`) — applies to both drivers, with unit test + API 400 test.
4. **branchd flags** select driver: `--runtime docker|kube` (+ `--kube-namespace`, `--kube-node`, `--kube-data-root`, `--kubeconfig`). CLI local mode stays Docker-only (K8s is a server deployment concern).
5. **Helm chart** at `deploy/helm/pgbranch`: Deployment (branchd, `--runtime kube`, sqlite state on a PVC or hostPath on the storage node — default: hostPath `<data-root>/state` + nodeName pin so state and data co-locate), ServiceAccount + Role/RoleBinding (pods create/delete/get/list/watch, pods/exec, pods/log in release namespace), two Services (api ClusterIP :7070, proxy ClusterIP/NodePort :6432), Secret for PGBRANCH_TOKEN (existingSecret support), values: image, node, dataRoot, sourceDefaults. `helm lint` + `helm template` golden test in CI script form (`make helm-test`).
6. **GitHub integration = separate small binary** `cmd/pgbranch-github` (`internal/ghook`): verifies `X-Hub-Signature-256` (HMAC-SHA256, env `GHOOK_WEBHOOK_SECRET`); on `pull_request` events for configured repos: opened/reopened/synchronize→ensure branch `pr-<number>` exists (create via apiclient if missing; synchronize does NOT reset by default, `GHOOK_RESET_ON_PUSH=true` opts in), closed→destroy. Optional PR comment with connect info when `GHOOK_GITHUB_TOKEN` set (plain REST call to GitHub API, no SDK dep; comment once per PR using a marker string search). Config via env: `GHOOK_PGBRANCH_SERVER`, `GHOOK_PGBRANCH_TOKEN`, `GHOOK_SOURCE`, `GHOOK_TTL`, `GHOOK_PROXY_HOST` (for the comment text). Tests: recorded webhook payload fixtures (hand-written minimal JSON), signature negative/positive, httptest fake pgbranch API asserting calls, fake GitHub API asserting comment. Helm chart gets an optional `ghook` sub-deployment (enabled flag). Docs: `docs/github-app.md` — registering the webhook/app, permissions (PRs read, issues write for comments), secret setup.

**Out of scope:** multi-node storage/CSI, TLS anywhere, GitHub Checks API.

---

### Task A: registry v3 (host column) + ContainerInfo.Host + branch-name validation
`internal/registry` (migration v3, MarkBranchReady signature), `internal/runtime/runtime.go` (Host field; DockerDriver Inspect sets Host "127.0.0.1"), `internal/pgproxy` resolver returns host:port, `internal/engine` (name validation `^[a-z0-9][a-z0-9-]{0,40}$` with clear error; saga passes info.Host), `internal/cli` connect uses branch.Host, `internal/api` 400 on invalid name (handler maps validation error). All existing unit+IT tests updated and green.
Commit: `feat: branch host tracking and name validation (registry v3)`

### Task B: Kubernetes runtime driver
`internal/runtime/kube.go` + `kube_test.go` (unit: volume→hostPath translation, pod spec construction golden-checks, name sanitation) + `kube_it_test.go` gated `PGBRANCH_K8S_IT=1`: creates/uses a kind cluster (helper script `hack/kind-up.sh`: `kind create cluster --name pgbranch-test` if absent; storage node = the single control-plane node; data-root /var/lib/pgbranch), runs: CreateVolume→RunHelper write/read→StartBranch with a REAL seeded source (seed via RunHelper pg_basebackup from a postgres pod started by the test… simplify: run a source postgres AS a plain pod via RunHelper-style spec or reuse engine.AddSource with the kube driver end-to-end) → engine e2e on kind: AddSource + CreateBranch + SQL through pod IP requires in-cluster access — from the host use a `kubectl port-forward`-equivalent via client-go portforward to the branch pod for the test's pgx connection. DestroyBranch. Full cleanup; cluster left running (cheap to keep; `hack/kind-down.sh` provided).
Deps: `k8s.io/client-go@latest` + apimachinery.
Commit: `feat(runtime): kubernetes driver — branch pods on a storage node`

### Task C: branchd --runtime flag + Helm chart
`cmd/branchd/main.go` flags per decision 4; `deploy/helm/pgbranch` per decision 5; `make helm-test` (lint + template with default and custom values, grep-assert critical fields: SYS_ADMIN, nodeName, hostPath, token secret ref). IT (gated PGBRANCH_K8S_IT=1): `helm install` onto the kind cluster, port-forward api Service, REST healthz + create source from an in-cluster postgres pod + branch + destroy. Document in README (K8s section).
Commit: `feat(deploy): helm chart for branchd on kubernetes`

### Task D: GitHub webhook service
Per decision 6: `internal/ghook/{service.go,github.go}` + tests, `cmd/pgbranch-github/main.go`, Helm optional sub-deployment, `docs/github-app.md`, README section. Unit/httptest only (no live GitHub).
Commit: `feat(ghook): branch-per-PR github webhook service`

### Task E: Phase 3 wrap
Full suites: unit, `PGBRANCH_IT=1` (docker), `PGBRANCH_K8S_IT=1` (kind). README roadmap update. 
Commit: `docs: phase 3 complete`
