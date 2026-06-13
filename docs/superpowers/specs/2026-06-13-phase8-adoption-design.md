# Phase 8 — adoption: review tooling, credentials, CI trust, packaging

## Why

v1.0-rc.1 (Phase 7) made pgbranch *trustworthy* — no SPOF, no leaks, metrics,
authz. Phase 8 makes it *adopted*: the features and polish that turn "I'd run
it" into "my team uses it daily." These are the Tier 2/3 items from the v1
roadmap that are buildable and testable without standing cloud infra.

Sequenced lowest-risk-highest-value first; each ships independently, leaves
the tree green, and is verified by unit + docker/kind ITs (no cloud spend).

## Components

### A. `pgb diff` on the pull request (+ bounded data sample)

`pgb diff` is the review-tool differentiator vs Neon/Supabase, but today it's
CLI-only. Make it show up where review happens:
- **ghook diff comment**: opt-in `GHOOK_DIFF_ON_PUSH` — after a branch is
  ready/reset, ghook calls branchd's diff endpoint and upserts a *separate*
  marker comment (`<!-- pgbranch-diff -->`) with the schema diff + per-table
  row deltas. Failures are non-fatal (the branch is the point). Diff is
  ~seconds and opt-in, so it never blocks the webhook ack.
- **bounded data sample**: `pgb diff --data` (and an API flag) adds, for each
  table with a row-count change, up to N (default 20) rows present in the
  branch but not the base, keyed by primary key (`branch EXCEPT base LIMIT N`
  over the PK columns) — a real but bounded data view, never a full dump.

### B. Credential helper (rotation ⇄ static-config)

Phase 1's per-branch rotation and static-env consumers (Vercel) are mutually
exclusive today — the wall we hit live. Resolve it: the *static* config a
consumer holds becomes the API endpoint + a scoped token, not the DB password.
- A tiny **connect helper** (Go package `pgbranchconnect` + a zero-dep JS
  `pgbranch-connect`): given `PGBRANCH_API` + a scoped token + a branch name
  (or git ref), it fetches the branch's current credentials from the REST API
  and returns a ready DSN. So an app keeps static config *and* per-branch
  rotation. Wire it into the demo app's `_db.js` as the reference.
- A `viewer`-role token is sufficient (read-only branch lookup), so the
  consumer never holds an admin credential.

### C. CI trust — graduate the kind / CSI / HA-failover ITs

The CSI, matrix, and HA-failover ITs are gated and never run in CI (CI does
the docker IT only). Stand up a **kind job** in GitHub Actions that runs the
`PGBRANCH_K8S_IT=1` suite (hostpath + CSI via the existing
`hack/kind-csi-up.sh`) and the HA-failover IT, so multi-node storage and
leader-election failover are continuously verified, not just written. Also
fix the flaky docker-driver host-port allocation surfaced in Phase 7 (publish
to an ephemeral port and read it back; retry on the rare collision) so the
integration job is deterministic under concurrency.

### D. Reusable Action + API-compat policy

- **Action**: harden and version the composite Action (`action/` + `action/
  destroy/`) — branding/metadata in `action.yml`, input validation, a `v1`
  moving tag, and a usage section so consumers pin `abd-ulbasit/pgbranch/
  action@v1`. (Marketplace listing needs a root `action.yml` or a split repo;
  documented as a follow-up, not required — `@v1` path refs already work.)
- **API-compat policy**: the REST surface is `/v1`. Add a compatibility test
  that locks the wire shape of the documented endpoints (golden JSON for the
  request/response of create/get/list/diff/reconcile/tokens) so a breaking
  change fails CI, plus a short `docs/api.md` stating the v1 stability promise.

## Deferred (needs cloud, not in Phase 8)

Snapshot-based seeding for RDS/Aurora and incremental source refresh require a
live managed Postgres to test honestly (cost). Captured as a design note in
docs; build when there's a cloud target. `--via dump` remains the
managed-cloud path until then.

## Testing

- A: engine/ghook unit (diff-comment upsert with a fake API; data-sample SQL
  shape) + docker IT (`pgb diff --data` shows branch-only rows).
- B: helper unit (fetches creds, builds DSN; viewer-token sufficient) + a JS
  `node --test`; demo-app wiring compiles.
- C: the kind job IS the test — it must go green on CSI + HA failover; plus a
  unit test for the docker ephemeral-port read-back/retry.
- D: action entrypoint tests (already exist) extended for validation; the
  golden API-compat test; `helm`/build unaffected.

## Sequencing

**A → B → C → D.** No operator. No new storage backends. CI stays green at
every step.
