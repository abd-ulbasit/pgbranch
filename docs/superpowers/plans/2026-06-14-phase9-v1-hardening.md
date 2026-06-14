# Phase 9 — Road to v1.0 GA: review hardening

> Source: pre-v1.0 multi-agent review (2026-06-14). Build order = priority. TDD,
> commit per task, keep CI green (`go build ./... && go test ./...`, `helm lint`).
> User approved scope: **Everything** (correctness + app-security + Helm/k8s + audit).

**Goal:** Close every confirmed bug and v1-blocking gap from the review so `v1.0.0`
is defensible to a platform-engineering reviewer.

## Group A — Correctness (engine/registry) — DO FIRST

- [ ] **A1 (🔴 data loss): interrupted branch-from-branch freeze deletes parent's live volume.**
  `engine/reconcile.go:219-241` (`ActionFailStuck`→`removeBranchLayer`), `freeze.go:101` (parent `ready→resetting`), `CommitFreeze` swaps rw only at end.
  Crash mid-freeze → reconcile sees parent "stuck", removes its live `rw_volume` = permanent data loss.
  Fix: a freeze-parent must NOT be `removeBranchLayer`'d on stuck-fail. Distinguish freeze-parents
  (recover via `restoreParent` / restart on original rw) from branches stuck building their own layer.
  Approach: mark freeze-parent with a distinct transient state (e.g. `freezing`) OR a flag column; in
  `ActionFailStuck`, skip `removeBranchLayer` when another live branch references this branch's rw_volume
  as its base. Test: simulate interrupted freeze (parent `resetting`, child `creating`, no CommitFreeze) →
  reconcile must NOT delete parent rw_volume; parent recoverable.
  **ROOT CAUSE (independent-agent pass, sharper):** the real trigger is that `updated_at` is NOT bumped
  during long sagas — `SetBranchContainer` (registry.go:351) deliberately doesn't touch it, and the
  `ListStuckBranches` comment (registry.go:401) *claims* progress resets the timer but the code doesn't.
  So a legit-but-slow freeze (cold image pull, big WAL replay) exceeds `stuck-timeout` (10m) and looks
  abandoned. **Fix must include BOTH:** (1) bump `updated_at` on every saga progress checkpoint
  (`SetBranchContainer` + wait-ready loop) so a progressing op keeps resetting the timer; (2) guard
  `ActionFailStuck` with `CountLiveBranchesByRWVolume` (the guard `gcSourceVolume` already uses,
  saga.go:512) so it never removes a volume that is still some live branch's rw_volume. Same shape on the
  CSI path (`provisionCSI` also parks the parent in `resetting`) — cover both.

- [ ] **A2: `TransitionBranch` TOCTOU** (`registry.go:318-329`, `setState` :246-258). SELECT→Go check→UPDATE
  not atomic; concurrent destroy/reset double-execute. Fix: conditional CAS
  `UPDATE branches SET state=? WHERE id=? AND state=?`; `RowsAffected()==0` ⇒ lost race / illegal.
  Model: `CommitFreeze` already does in-tx state check. Test: two concurrent transitions from same start state — exactly one wins.

- [ ] **A3: force-destroy stuck branches** (`registry.go` legalBranch, `saga.go:458`). A branch wedged in
  `creating`/`resetting` can't be destroyed until stuck-timeout. Fix: `DestroyBranch` first transitions a
  transient-state branch to `failed` (or allow `creating/resetting→failed→destroying` force path). Test: destroy a `creating` branch succeeds.

- [ ] **A4: stop swallowing compensation/failure-transition errors** (`saga.go:78,368`; `freeze.go:77,109,118,209`;
  `csi.go`, `engine.go`, `diff.go:125`). Log (slog.Warn) every ignored `TransitionBranch(...,Failed)` and every
  `undo` closure error; add a `pgbranch_compensation_failures_total` metric. No control-flow change.

## Group B — v1 lifecycle features

- [ ] **B1: `--max-branches` quota** (spec-promised, missing). Registry live-branch count check in `CreateBranch`/`CreateBranchFrom`;
  exceed ⇒ engine error mapped to HTTP 403. Daemon flag + env. Test: N+1th create rejected; destroy frees a slot.
- [ ] **B2: disk-free visibility.** `pgbranch_disk_bytes_free` gauge (statfs on the storage root) in `internal/metrics`;
  document ENOSPC failure mode + recommended alert in `docs/observability.md`.
- [ ] **B3: `--default-ttl` / `--max-ttl`** daemon flags. ghook/API branches with no explicit TTL get default; requested TTL capped at max. Test: default applied, over-max capped.

## Group C — App security

- [ ] **C1 (H1): validate source names** — mirror `branchNameRe ^[a-z0-9][a-z0-9-]{0,40}$` in `registry.CreateSource`/`engine.AddSource`. Test: bad name rejected at API layer.
- [ ] **C2: generic 500s (A-2)** — `writeEngineError` 500 branch logs full error server-side, returns `"internal server error"`. **Uniform proxy error (P-2)** — `pgproxy` collapses not-found/not-ready to one `3D000` message (no branch-state leak to unauth clients).
- [ ] **C3: DoS limits** — `io.LimitReader(r.Body, 1<<20)` on ghook webhook (`service.go:97`); `SetDeadline(30s)` on proxy conn before `readStartupFrame` (`proxy.go:82`).
- [ ] **C4: token transport** — `apiclient` refuse/warn bearer token over non-loopback `http://` (M2); add `PGBRANCH_CA_CERT` CA-bundle option, keep `TLS_SKIP_VERIFY` as last-resort w/ stderr warning (M3/S-3).
- [ ] **C5: secret-at-rest** — hash or encrypt stored branch password (S-1; key from `PGBRANCH_TOKEN`); pass rotated password to `ALTER ROLE` via stdin not argv (S-2, keeps it out of exec audit logs).
- [ ] **C6: `token_hash` unique index** (A-1) — add index in a new migration (v10); keep constant-time-safe lookup (hash discriminator reveals nothing).

## Group D — Deploy/k8s + audit

- [ ] **D1: branchd securityContext** (C-1/K-2) — `allowPrivilegeEscalation:false`, `readOnlyRootFilesystem:true` (+ emptyDir for state/tmp), `seccompProfile:RuntimeDefault`, `capabilities.drop:[ALL]`. Document branch-pod `CAP_SYS_ADMIN` (overlay) + recommend CSI for prod.
- [ ] **D2: NetworkPolicy templates** (K-1) — ingress to branchd API only from ghook + labeled clients; ingress to branch pods only from branchd/proxy; egress from branch pods to source + DNS. Gated by `networkPolicy.enabled`.
- [ ] **D3: proxy TLS posture (P-1) + base-image digest pins (C-2)** — startup warning when `--pg-tls-cert` unset and `--pg-addr` non-loopback; chart `NOTES.txt`; pin `golang:1.26-alpine@sha256` + runtime base in both Dockerfiles.
- [ ] **D4: token hygiene (K-3)** — encourage `existingSecret` (token out of Helm release history); drop/scope `secrets: get` in preview-deployer RBAC to `resourceNames`.
- [ ] **D5: audit log (#4)** — record actor (token name/role from request context) on branch create/destroy/reset in the `transitions` table; expose via API/CLI.

## Group E — Deltas surfaced by the installed-agent pass (2026-06-14) — NEW, not in original review

- [ ] **E1: ZFS source-name path injection (sharpens C1).** Source name flows UNQUOTED into the ZFS dataset
  path (`engine/zfs.go` `shellSafeRe` allowlists `/` and `.`), so a name like `../../rpool/ROOT` is
  shell-safe and yields `zfs create -p tank/pgbranch/src-../../rpool/ROOT-g1` = dataset-namespace traversal.
  This makes source-name validation a real injection fix, not cosmetic. The `validateSourceName` regex in C1 closes it; keep this note so the test covers the traversal vector.
- [ ] **E2: ghook branch-name collision / cross-PR reset (G3).** In `git-branch` naming mode an untrusted PR
  head ref, after `sanitizeBranchName`, can collide with another PR's or a human's branch; with `ResetOnPush`
  that **resets someone else's branch (data loss)**. Fix: namespace App-created branches (e.g. `gh-pr-<n>`)
  and forbid git-branch mode from colliding with that namespace. `internal/ghook/service.go`.
- [ ] **E3: ghook deployment has NO securityContext** (it's the internet-facing pod). Add hardened context
  (runAsNonRoot, drop ALL caps, readOnlyRootFilesystem, seccomp RuntimeDefault) in `ghook-deployment.yaml`. Fold into D1.
- [ ] **E4: proxy connection cap + idle timeout** (beyond C3's startup read-deadline). Add a concurrency
  semaphore in `pgproxy.Serve` and an idle timeout on the relay copy loop — slowloris/half-open floods else exhaust goroutines/fds. Fold into C3.
- [ ] **E5: `automountServiceAccountToken: false`** on branch + helper pods (they need no kube API). Fold into D1.
- [ ] **E6: ghook startup guards** — refuse empty/short `GHOOK_WEBHOOK_SECRET`; warn loudly when `GHOOK_REPOS`
  is empty (allow-all). `internal/ghook/env.go`. Fold into D-docs / small code guard.
- [ ] **E7 (docs/default): inherit-mode credential reuse (S2).** Default `rotateBranchCredentials=false`
  means every branch shares the source password ⇒ cross-branch reach via the unauth proxy. Document that
  network-exposed proxies must enable rotation; consider flipping the chart default. Also: don't post the
  proxy connect endpoint in PR comments on **public** repos (config flag, default off for public).
- [ ] **E8 (optional): `/metrics` exposure** — aggregate only (no secrets) but unauthenticated on a
  cluster-reachable port; rely on the D2 NetworkPolicy, or move to a separate scrape port/token.

## Finish
- Tag the GitHub Action `v1`; cut `v1.0.0` (or `-rc.2` first) with release notes summarizing Phase 9.
