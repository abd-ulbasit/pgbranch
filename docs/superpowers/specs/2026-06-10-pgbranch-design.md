# pgbranch — Design Spec

**Date:** 2026-06-10
**Status:** Approved
**One-liner:** `git branch` for Postgres — instant copy-on-write branches of any-size databases, self-hosted, branch-per-PR.

## Problem

Every backend team needs fresh, production-like Postgres data for development, testing, and PR review — but copying a multi-GB database takes minutes-to-hours and multiplies storage cost. Neon and Supabase solved this with cloud-only branching; the self-hosted option (DBLab/postgres.ai) is operationally clunky and ZFS-bound. There is no lightweight, container-native, bring-your-own-infrastructure Postgres branching engine.

## Scope boundary

pgbranch is a **dev/test data tool**. Branches are disposable, writable, single-writer Postgres instances sourced from a snapshot of a source database (prod replica, staging, seed). It is explicitly **not**:

- A production HA/failover system
- A multi-writer or merge-back system (no merging data between branches)
- A storage engine (we orchestrate copy-on-write at the filesystem layer; we do not implement a pageserver like Neon)
- Multi-engine (Postgres only; no MySQL)

## Architecture

Five components, one binary where possible.

### 1. Control plane — `branchd`

Go daemon. Owns:

- **Branch registry**: metadata for the branch tree (id, name, parent, created-at, TTL, state), persisted in embedded SQLite (no external dependency for single-host mode).
- **Branch state machine**: `creating → ready → (resetting|expiring) → destroyed`, with failure states. Every transition is journaled.
- **API**: REST (primary) + gRPC. Auth via API tokens (Phase 2).
- **Lifecycle policies**: TTL reaper, max-branch and disk-quota enforcement.

### 2. CoW snapshot engine (pluggable)

One interface, three backends:

| Backend | Use case | Mechanism |
|---|---|---|
| **OverlayFS** (default) | Any Linux host/container | Source data dir = lower layer; each branch gets its own upper layer + workdir |
| **ZFS** (optional) | Hosts already running ZFS | `zfs snapshot` + `zfs clone` |
| **Copy** (fallback) | Tests, macOS dev, correctness baseline | `cp --reflink=auto` or plain copy |

**Consistency model:** branch creation issues `CHECKPOINT` on the source instance, then snapshots the live data directory. The snapshot is crash-consistent; the branch's Postgres performs standard WAL recovery on first boot. This is the same well-understood mechanism as filesystem-snapshot backups (and how DBLab works).

### 3. Runtime drivers (pluggable)

Where branch instances run. Each branch = one disposable Postgres container on its CoW layer.

- **Docker driver** (Phase 1): single host, containers managed via Docker API.
- **Kubernetes driver** (Phase 3): branch pods, node-local CoW volumes, Helm chart for the whole system.

### 4. Wire-protocol router — `pgproxy`

Single stable endpoint for all branches. Postgres wire-protocol aware:

- Routing key: branch name extracted from the startup message (database name suffix `mydb@pr-42`, or SNI host `pr-42.db.internal` when TLS).
- Auth passthrough to the target instance; TLS termination optional.
- Per-branch connection counts and metrics (Prometheus).

### 5. CLI — `pgb`

`pgb source add`, `pgb branch create pr-42 --from main`, `pgb branch list` (tree view), `pgb branch reset`, `pgb branch destroy`, `pgb connect pr-42` (prints/launches psql connection string).

### Supporting subsystems

- **Source sync**: seed the "main" data dir from an existing Postgres via `pg_basebackup`; scheduled refresh re-syncs and rebases nothing (existing branches keep their old base; new branches use the new base).
- **Masking hooks**: ordered SQL scripts executed against a branch immediately after creation, before it is marked `ready` (PII scrubbing).
- **GitHub App** (Phase 3): PR opened → create branch + comment connection details; PR closed/merged → destroy.
- **Web UI** (Phase 4): branch tree visualization, disk usage per branch, activity log.

## Data flow

```
source Postgres ──pg_basebackup──► main data dir (lower layer)
                                        │ CHECKPOINT + snapshot
                                        ▼
                              branch CoW layer (upper)
                                        │ boot disposable Postgres (WAL recovery)
                                        ▼
                              masking hooks → state: ready
                                        │ register route
                                        ▼
                       pgproxy ◄── client connects to pr-42.db.internal
```

## Error handling

- **Branch creation is a saga**: each step (snapshot, container boot, recovery wait, masking, route registration) has a compensating action. Any failure unwinds completely — no orphaned FS layers, containers, or registry rows.
- **Integrity**: Postgres data checksums enabled on sources we create; `pg_controldata` state verified after recovery; a branch that fails verification is marked `failed`, never `ready`.
- **Crash recovery of branchd itself**: on startup, reconcile registry against actual containers/FS layers; orphans are adopted or garbage-collected.

## Testing strategy

- **TDD** throughout; unit tests for registry, state machine, policy logic.
- **Integration**: testcontainers-based — real Postgres, real OverlayFS (inside a Linux container/Colima VM on macOS dev machines).
- **Correctness harness**: branch while the source is under sustained write load (pgbench), then run full-table checksums on the branch; verify zero corruption across N iterations.
- **Chaos**: kill branchd/container mid-snapshot and mid-boot; verify saga unwinding and startup reconciliation.
- **Benchmarks** (published in README): branch-creation latency and disk overhead at 1 GB / 50 GB / 500 GB.

## Phases

- **P1 — Core engine**: OverlayFS backend, Docker driver, SQLite registry, CLI (source add / branch create / list / destroy / connect). Demo: branch a 10 GB database in < 3 s.
- **P2 — Data plane**: pgproxy router, REST API + token auth, TTL reaper, branch reset, source refresh.
- **P3 — Platform**: Kubernetes driver, Helm chart, GitHub App.
- **P4 — Polish**: masking hooks, web UI, ZFS backend, benchmark publication, docs site.

## Tech stack

Go 1.26+, embedded SQLite (modernc or mattn), Docker SDK, controller-free Kubernetes client (client-go, no CRDs/operators by design), Prometheus client, testcontainers-go, pgx for Postgres control connections. License: Apache-2.0.
