# Phase 7 — Road to v1: operational trust

## Why

pgbranch v0.3 is feature-rich (CoW branching, branch-from-branch, wire
router, TTL, masking, dump/basebackup seeding, K8s hostpath+CSI, ghook with
statuses/App-auth/live-comment, test SDK, `pgb diff`). What stands between it
and a **1.0** people run for a team is not more features — it is **trust**:
no single point of failure, no resource leaks, visibility into what it's
doing, and a defensible security posture.

This phase delivers the four "trust bar" items. Tier 2/3 features (managed-
cloud snapshot seeding, data diff, credential helper, Marketplace Action,
API-compat guarantees) come in a later phase; they make pgbranch *adopted*,
but these four are what make it *trustworthy*.

Each item below was motivated by something that actually broke or was blind
during the v0.3 build/deploy work (registry SPOF, leaked branches/volumes/
LBs, pod-IP and webhook-cancellation races, zero metrics, cluster-admin
kubeconfig handed to CI).

**Explicitly out of scope (do not build):** an operator/CRDs (pgbranch's
services+Helm shape is intentional); new storage backends beyond
overlay/zfs/csi; distributed/sharded Postgres.

## Components

### 1. Observability (metrics, logs, real readiness)

A Prometheus `/metrics` endpoint on branchd (separate from the REST API, or
the same mux behind the same listener) exposing operational gauges/counters/
histograms: branches by state, sources by state, branch create/reset/destroy
latency, masking duration, reaper runs and reaped count, diff duration, layer
count and refcount, in-flight operations, and error counters by operation.
Readiness reflects real backend health (registry reachable + driver reachable),
not just "process up." Structured logs already exist (slog); ensure every
saga step and reconcile decision logs at a consistent level with branch/source
context. No new heavy deps — use `prometheus/client_golang` (already common)
or a tiny hand-rolled text exposition if avoiding the dep is preferred.

### 2. Reconcile loop + leak-proof GC

Today `Reconcile` runs once at startup. v1 makes it a **periodic authoritative
control loop** (configurable interval) that converges actual state to the
registry: fail+clean stuck `creating`/`resetting` rows past a deadline, remove
managed containers/pods/volumes with no registry row, GC dangling frozen
layers (refcount 0) and orphaned rw volumes, and enforce TTL (the reaper folds
into this loop). Add operator-facing commands: `pgb doctor` (report drift:
orphans, leaks, stuck rows — read-only) and `pgb gc` (converge: clean them).
Every reconcile pass emits metrics (item 1) and structured logs. This directly
addresses the branch/volume/LB leaks seen repeatedly during v0.3 work.

### 3. Security: authz model + scoped credentials

Today the REST API has a single bearer token (all-or-nothing) and deployment
recipes handed cluster-admin to CI. v1 adds:
- **Scoped API tokens**: named tokens with roles (`admin` full; `operator`
  create/reset/destroy/list; `viewer` read-only). Registry-backed token table,
  `pgb token create/ls/revoke`. The existing single `PGBRANCH_TOKEN` maps to
  `admin` for backward compatibility.
- **Proxy TLS posture**: document and default to TLS for the wire router in
  cloud (the cert plumbing exists from Phase 5; make it first-class in the
  chart with cert-manager guidance).
- **Scoped-SA deployment recipe**: ship a namespaced ServiceAccount + Role
  (not cluster-admin) for CI/preview integrations, as the documented default.

### 4. HA: eliminate the registry SPOF

Today branchd is single-replica because SQLite is single-writer; if branchd's
node dies, control is lost (and on hostpath, so is the data — already mitigated
by CSI + the Phase 1 PVC). v1: run branchd as a Deployment with **leader
election** (k8s Lease) — N replicas, one active writer, automatic failover.
The registry stays SQLite on a shared PVC (RWO, follows the leader) — no new
datastore. Non-leaders serve read-only `/healthz`/`/metrics` and stand by.
This is the riskiest item (touches startup/lifecycle) and is sequenced last.

## Testing

- **Unit** (fake driver, in-memory/temp registry): metric emission on each op;
  reconcile decisions (stuck→failed, orphan removal, layer GC, TTL); authz
  (role gating per endpoint, legacy-token→admin); leader-election state machine
  with a fake Lease/clock.
- **Docker IT** (`PGBRANCH_IT=1`): GC actually removes a leaked container +
  volume; TTL reaped via the loop; scoped token forbidden on a privileged
  endpoint (403) and allowed on its own.
- **kind IT** (`PGBRANCH_K8S_IT=1`): two branchd replicas, one becomes leader;
  kill the leader → the other takes over and keeps serving; `/metrics` scrapeable.
- All existing suites stay green; CI unchanged gates plus the new unit tests.

## Sequencing

Build one item at a time, lowest-risk-highest-value first, review between:
**1) metrics → 2) reconcile+GC → 3) authz/security → 4) HA.** Each ships
independently (own commits, docs, tests) and leaves the tree green.
