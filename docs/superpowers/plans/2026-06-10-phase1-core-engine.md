# pgbranch Phase 1 — Core Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the pgbranch core engine: seed a source data dir from any Postgres via `pg_basebackup`, create instant OverlayFS copy-on-write branches as disposable Postgres containers, manage them via the `pgb` CLI.

**Architecture:** Container-native. All data lives in named Docker volumes (so the same code works on Colima/macOS and bare Linux — volumes sit on ext4 inside the VM, avoiding virtiofs-as-overlay-upper problems). Each branch container assembles its own overlay mount (lower = source volume read-only, upper/work = branch volume) in a small entrypoint script with CAP_SYS_ADMIN, then execs the stock postgres entrypoint, which performs normal WAL crash recovery. Host-side Go code is pure control plane: SQLite registry + state machine, saga-orchestrated create/destroy, Docker driver. No daemon in Phase 1 — the CLI embeds the engine; `branchd` (Phase 2) will reuse the same `internal/engine` package.

**Phase 1 scope cut (documented):** branches are created from *sources* only (`--from <source>`). Branch-from-branch (layer DAG with frozen uppers) is Phase 2 — copying overlay uppers requires whiteout/xattr-preserving copies that deserve their own design.

**Tech Stack:** Go 1.26+, `modernc.org/sqlite` (CGO-free), `github.com/docker/docker` SDK, `github.com/spf13/cobra`, `github.com/jackc/pgx/v5`, `github.com/testcontainers/testcontainers-go`. Docker via Colima on macOS dev.

**Conventions:**
- Module: `github.com/abd-ulbasit/pgbranch`
- Integration tests live next to unit tests, guarded by `PGBRANCH_IT=1` env (skip otherwise). Run with `make it`.
- Every container/volume pgbranch creates carries label `pgbranch.managed=true` plus `pgbranch.branch.id` / `pgbranch.source.name`.
- Commit after every green test. Commit messages end with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

---

### Task 1: Scaffolding

**Files:**
- Create: `go.mod`, `Makefile`, `.gitignore`, `cmd/pgb/main.go` (stub)

- [ ] **Step 1: Init module and layout**

```bash
cd /Users/basit/projects/pgbranch
go mod init github.com/abd-ulbasit/pgbranch
mkdir -p cmd/pgb internal/{config,registry,cow,runtime,pgctl,engine,cli}
```

- [ ] **Step 2: Write `.gitignore` and `Makefile`**

`.gitignore`:
```
/bin/
*.db
.DS_Store
```

`Makefile`:
```makefile
.PHONY: build test it lint

build:
	go build -o bin/pgb ./cmd/pgb

test:
	go test ./...

it:
	PGBRANCH_IT=1 go test ./... -count=1 -timeout 20m

lint:
	go vet ./...
```

- [ ] **Step 3: Stub `cmd/pgb/main.go`**

```go
package main

import "fmt"

func main() {
	fmt.Println("pgb: not yet implemented")
}
```

- [ ] **Step 4: Verify build, commit**

Run: `go build ./... && go vet ./...` — expect no output (success).

```bash
git add -A && git commit -m "chore: scaffold pgbranch module"
```

---

### Task 2: Config (home dir + registry path)

**Files:**
- Create: `internal/config/config.go`
- Test: `internal/config/config_test.go`

- [ ] **Step 1: Write failing test**

```go
package config

import (
	"path/filepath"
	"testing"
)

func TestDefaultHomeUnderUserHome(t *testing.T) {
	t.Setenv("PGBRANCH_HOME", "")
	c, err := Load()
	if err != nil {
		t.Fatal(err)
	}
	if filepath.Base(c.Home) != ".pgbranch" {
		t.Fatalf("Home = %q, want ~/.pgbranch", c.Home)
	}
	if c.RegistryPath != filepath.Join(c.Home, "pgbranch.db") {
		t.Fatalf("RegistryPath = %q", c.RegistryPath)
	}
}

func TestHomeOverride(t *testing.T) {
	t.Setenv("PGBRANCH_HOME", "/tmp/pgbtest")
	c, err := Load()
	if err != nil {
		t.Fatal(err)
	}
	if c.Home != "/tmp/pgbtest" {
		t.Fatalf("Home = %q", c.Home)
	}
}
```

- [ ] **Step 2: Run `go test ./internal/config/` — expect FAIL (Load undefined)**

- [ ] **Step 3: Implement**

```go
package config

import (
	"os"
	"path/filepath"
)

type Config struct {
	Home         string // state directory, default ~/.pgbranch
	RegistryPath string // SQLite file
	PostgresImage string // default image for helpers/branches when source has no version
}

func Load() (*Config, error) {
	home := os.Getenv("PGBRANCH_HOME")
	if home == "" {
		uh, err := os.UserHomeDir()
		if err != nil {
			return nil, err
		}
		home = filepath.Join(uh, ".pgbranch")
	}
	return &Config{
		Home:          home,
		RegistryPath:  filepath.Join(home, "pgbranch.db"),
		PostgresImage: "postgres:17",
	}, nil
}

// EnsureHome creates the state directory.
func (c *Config) EnsureHome() error { return os.MkdirAll(c.Home, 0o755) }
```

- [ ] **Step 4: Run `go test ./internal/config/` — expect PASS. Commit `feat: config with PGBRANCH_HOME override`**

---

### Task 3: Registry — schema, sources

**Files:**
- Create: `internal/registry/registry.go`, `internal/registry/schema.go`
- Test: `internal/registry/registry_test.go`

The registry is the single source of truth. SQLite via `modernc.org/sqlite` (driver name `"sqlite"`). All state changes journaled into `transitions`.

- [ ] **Step 1: Add dependency**

```bash
go get modernc.org/sqlite@latest
```

- [ ] **Step 2: Write failing test for Open + source CRUD**

```go
package registry

import (
	"path/filepath"
	"testing"
)

func openTest(t *testing.T) *Registry {
	t.Helper()
	r, err := Open(filepath.Join(t.TempDir(), "test.db"))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { r.Close() })
	return r
}

func TestSourceLifecycle(t *testing.T) {
	r := openTest(t)
	s := &Source{
		Name: "main", PGVersion: "17", Volume: "pgbranch-src-main",
		ConnHost: "db.example.com", ConnPort: 5432, ConnUser: "postgres", ConnDB: "app",
		Network: "",
	}
	if err := r.CreateSource(s); err != nil {
		t.Fatal(err)
	}
	if s.ID == "" || s.State != SourceSeeding {
		t.Fatalf("ID=%q State=%q", s.ID, s.State)
	}
	if err := r.SetSourceState(s.ID, SourceReady, "seed complete"); err != nil {
		t.Fatal(err)
	}
	got, err := r.GetSourceByName("main")
	if err != nil {
		t.Fatal(err)
	}
	if got.State != SourceReady || got.Volume != "pgbranch-src-main" {
		t.Fatalf("got %+v", got)
	}
	// duplicate name rejected
	if err := r.CreateSource(&Source{Name: "main", PGVersion: "17", Volume: "x"}); err == nil {
		t.Fatal("want duplicate-name error")
	}
	list, err := r.ListSources()
	if err != nil || len(list) != 1 {
		t.Fatalf("list=%v err=%v", list, err)
	}
}
```

- [ ] **Step 3: Run `go test ./internal/registry/` — expect FAIL**

- [ ] **Step 4: Implement schema + sources**

`internal/registry/schema.go`:
```go
package registry

const schema = `
CREATE TABLE IF NOT EXISTS sources (
  id TEXT PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  pg_version TEXT NOT NULL,
  volume TEXT NOT NULL,
  conn_host TEXT NOT NULL DEFAULT '',
  conn_port INTEGER NOT NULL DEFAULT 0,
  conn_user TEXT NOT NULL DEFAULT '',
  conn_db   TEXT NOT NULL DEFAULT '',
  network   TEXT NOT NULL DEFAULT '',
  state TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);
CREATE TABLE IF NOT EXISTS branches (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  source_id TEXT NOT NULL REFERENCES sources(id),
  state TEXT NOT NULL,
  container_id TEXT NOT NULL DEFAULT '',
  rw_volume TEXT NOT NULL,
  port INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);
-- name unique among live branches only (destroyed rows kept for history)
CREATE UNIQUE INDEX IF NOT EXISTS branches_live_name
  ON branches(name) WHERE state != 'destroyed';
CREATE TABLE IF NOT EXISTS transitions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  entity TEXT NOT NULL,        -- 'source' | 'branch'
  entity_id TEXT NOT NULL,
  from_state TEXT NOT NULL,
  to_state TEXT NOT NULL,
  reason TEXT NOT NULL DEFAULT '',
  at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);
`
```

`internal/registry/registry.go`:
```go
package registry

import (
	"crypto/rand"
	"database/sql"
	"encoding/hex"
	"errors"
	"fmt"

	_ "modernc.org/sqlite"
)

type SourceState string
type BranchState string

const (
	SourceSeeding SourceState = "seeding"
	SourceReady   SourceState = "ready"
	SourceFailed  SourceState = "failed"

	BranchCreating   BranchState = "creating"
	BranchReady      BranchState = "ready"
	BranchFailed     BranchState = "failed"
	BranchDestroying BranchState = "destroying"
	BranchDestroyed  BranchState = "destroyed"
)

var ErrNotFound = errors.New("not found")

type Source struct {
	ID, Name, PGVersion, Volume        string
	ConnHost, ConnUser, ConnDB, Network string
	ConnPort                            int
	State                               SourceState
	CreatedAt                           string
}

type Branch struct {
	ID, Name, SourceID, ContainerID, RWVolume string
	Port                                      int
	State                                     BranchState
	CreatedAt                                 string
}

type Registry struct{ db *sql.DB }

func Open(path string) (*Registry, error) {
	db, err := sql.Open("sqlite", path+"?_pragma=busy_timeout(5000)&_pragma=journal_mode(WAL)&_pragma=foreign_keys(1)")
	if err != nil {
		return nil, err
	}
	db.SetMaxOpenConns(1) // SQLite single-writer; keep it simple
	if _, err := db.Exec(schema); err != nil {
		return nil, fmt.Errorf("apply schema: %w", err)
	}
	return &Registry{db: db}, nil
}

func (r *Registry) Close() error { return r.db.Close() }

func newID() string {
	b := make([]byte, 8)
	rand.Read(b)
	return hex.EncodeToString(b)
}

func (r *Registry) CreateSource(s *Source) error {
	s.ID, s.State = newID(), SourceSeeding
	_, err := r.db.Exec(`INSERT INTO sources
		(id,name,pg_version,volume,conn_host,conn_port,conn_user,conn_db,network,state)
		VALUES (?,?,?,?,?,?,?,?,?,?)`,
		s.ID, s.Name, s.PGVersion, s.Volume, s.ConnHost, s.ConnPort, s.ConnUser, s.ConnDB, s.Network, s.State)
	if err != nil {
		return fmt.Errorf("create source %q: %w", s.Name, err)
	}
	return r.journal("source", s.ID, "", string(SourceSeeding), "created")
}

func (r *Registry) SetSourceState(id string, to SourceState, reason string) error {
	return r.setState("sources", "source", id, string(to), reason)
}

func (r *Registry) setState(table, entity, id, to, reason string) error {
	var from string
	if err := r.db.QueryRow(`SELECT state FROM `+table+` WHERE id=?`, id).Scan(&from); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return ErrNotFound
		}
		return err
	}
	if _, err := r.db.Exec(`UPDATE `+table+` SET state=?, updated_at=strftime('%Y-%m-%dT%H:%M:%fZ','now') WHERE id=?`, to, id); err != nil {
		return err
	}
	return r.journal(entity, id, from, to, reason)
}

func (r *Registry) journal(entity, id, from, to, reason string) error {
	_, err := r.db.Exec(`INSERT INTO transitions (entity,entity_id,from_state,to_state,reason) VALUES (?,?,?,?,?)`,
		entity, id, from, to, reason)
	return err
}

func scanSource(row interface{ Scan(...any) error }) (*Source, error) {
	s := &Source{}
	err := row.Scan(&s.ID, &s.Name, &s.PGVersion, &s.Volume, &s.ConnHost, &s.ConnPort,
		&s.ConnUser, &s.ConnDB, &s.Network, &s.State, &s.CreatedAt)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, ErrNotFound
	}
	return s, err
}

const sourceCols = `id,name,pg_version,volume,conn_host,conn_port,conn_user,conn_db,network,state,created_at`

func (r *Registry) GetSourceByName(name string) (*Source, error) {
	return scanSource(r.db.QueryRow(`SELECT `+sourceCols+` FROM sources WHERE name=?`, name))
}

func (r *Registry) ListSources() ([]*Source, error) {
	rows, err := r.db.Query(`SELECT ` + sourceCols + ` FROM sources ORDER BY created_at`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var out []*Source
	for rows.Next() {
		s, err := scanSource(rows)
		if err != nil {
			return nil, err
		}
		out = append(out, s)
	}
	return out, rows.Err()
}
```

- [ ] **Step 5: Run `go test ./internal/registry/` — expect PASS. Commit `feat(registry): sqlite registry with sources`**

---

### Task 4: Registry — branches + state machine rules

**Files:**
- Modify: `internal/registry/registry.go`
- Test: `internal/registry/registry_test.go` (append)

Legal transitions: `creating→{ready,failed}`, `ready→destroying`, `failed→destroying`, `destroying→destroyed`. Anything else errors.

- [ ] **Step 1: Write failing tests**

```go
func TestBranchLifecycleAndTransitions(t *testing.T) {
	r := openTest(t)
	s := &Source{Name: "main", PGVersion: "17", Volume: "v"}
	if err := r.CreateSource(s); err != nil {
		t.Fatal(err)
	}
	b := &Branch{Name: "pr-1", SourceID: s.ID, RWVolume: "pgbranch-br-pr-1-rw"}
	if err := r.CreateBranch(b); err != nil {
		t.Fatal(err)
	}
	if b.State != BranchCreating {
		t.Fatalf("state=%q", b.State)
	}
	// illegal: creating -> destroyed
	if err := r.TransitionBranch(b.ID, BranchDestroyed, ""); err == nil {
		t.Fatal("want illegal transition error")
	}
	if err := r.MarkBranchReady(b.ID, "cid123", 54321); err != nil {
		t.Fatal(err)
	}
	got, _ := r.GetBranchByName("pr-1")
	if got.State != BranchReady || got.ContainerID != "cid123" || got.Port != 54321 {
		t.Fatalf("got %+v", got)
	}
	if err := r.TransitionBranch(b.ID, BranchDestroying, "user destroy"); err != nil {
		t.Fatal(err)
	}
	if err := r.TransitionBranch(b.ID, BranchDestroyed, ""); err != nil {
		t.Fatal(err)
	}
	// name reusable after destroy
	if err := r.CreateBranch(&Branch{Name: "pr-1", SourceID: s.ID, RWVolume: "v2"}); err != nil {
		t.Fatalf("name not reusable: %v", err)
	}
	// live list excludes destroyed
	live, err := r.ListLiveBranches()
	if err != nil || len(live) != 1 {
		t.Fatalf("live=%v err=%v", live, err)
	}
}
```

- [ ] **Step 2: Run — expect FAIL. Step 3: Implement**

Append to `registry.go`:
```go
var legalBranch = map[BranchState][]BranchState{
	BranchCreating:   {BranchReady, BranchFailed},
	BranchReady:      {BranchDestroying},
	BranchFailed:     {BranchDestroying},
	BranchDestroying: {BranchDestroyed},
}

func (r *Registry) CreateBranch(b *Branch) error {
	b.ID, b.State = newID(), BranchCreating
	_, err := r.db.Exec(`INSERT INTO branches (id,name,source_id,state,rw_volume) VALUES (?,?,?,?,?)`,
		b.ID, b.Name, b.SourceID, b.State, b.RWVolume)
	if err != nil {
		return fmt.Errorf("create branch %q: %w", b.Name, err)
	}
	return r.journal("branch", b.ID, "", string(BranchCreating), "created")
}

func (r *Registry) TransitionBranch(id string, to BranchState, reason string) error {
	b, err := r.getBranch(`id=?`, id)
	if err != nil {
		return err
	}
	for _, ok := range legalBranch[b.State] {
		if ok == to {
			return r.setState("branches", "branch", id, string(to), reason)
		}
	}
	return fmt.Errorf("illegal branch transition %s -> %s", b.State, to)
}

func (r *Registry) MarkBranchReady(id, containerID string, port int) error {
	if _, err := r.db.Exec(`UPDATE branches SET container_id=?, port=? WHERE id=?`, containerID, port, id); err != nil {
		return err
	}
	return r.TransitionBranch(id, BranchReady, "instance running")
}

const branchCols = `id,name,source_id,state,container_id,rw_volume,port,created_at`

func (r *Registry) getBranch(where string, args ...any) (*Branch, error) {
	b := &Branch{}
	err := r.db.QueryRow(`SELECT `+branchCols+` FROM branches WHERE `+where, args...).
		Scan(&b.ID, &b.Name, &b.SourceID, &b.State, &b.ContainerID, &b.RWVolume, &b.Port, &b.CreatedAt)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, ErrNotFound
	}
	return b, err
}

func (r *Registry) GetBranchByName(name string) (*Branch, error) {
	return r.getBranch(`name=? AND state!='destroyed'`, name)
}

func (r *Registry) ListLiveBranches() ([]*Branch, error) {
	rows, err := r.db.Query(`SELECT ` + branchCols + ` FROM branches WHERE state!='destroyed' ORDER BY created_at`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var out []*Branch
	for rows.Next() {
		b := &Branch{}
		if err := rows.Scan(&b.ID, &b.Name, &b.SourceID, &b.State, &b.ContainerID, &b.RWVolume, &b.Port, &b.CreatedAt); err != nil {
			return nil, err
		}
		out = append(out, b)
	}
	return out, rows.Err()
}
```

- [ ] **Step 4: Run `go test ./internal/registry/` — PASS. Commit `feat(registry): branch state machine with journaled transitions`**

---

### Task 5: CoW layer planning (pure logic)

**Files:**
- Create: `internal/cow/plan.go`, `internal/cow/entrypoint.sh`
- Test: `internal/cow/plan_test.go`

Pure host-side planning — no syscalls. Produces volume names, container mount specs, and the overlay entrypoint script (embedded with `go:embed`). The actual overlay mount happens *inside* the branch container.

- [ ] **Step 1: Write failing test**

```go
package cow

import (
	"strings"
	"testing"
)

func TestPlanBranch(t *testing.T) {
	p := PlanBranch("pr-1", "pgbranch-src-main")
	if p.RWVolume != "pgbranch-br-pr-1-rw" {
		t.Fatalf("RWVolume=%q", p.RWVolume)
	}
	if p.Lowers[0] != "/pgbranch/lower0" {
		t.Fatalf("Lowers=%v", p.Lowers)
	}
	if p.LowerEnv() != "/pgbranch/lower0" {
		t.Fatalf("LowerEnv=%q", p.LowerEnv())
	}
	if p.SourceVolume != "pgbranch-src-main" {
		t.Fatalf("SourceVolume=%q", p.SourceVolume)
	}
}

func TestEntrypointScriptContent(t *testing.T) {
	for _, want := range []string{
		"mount -t overlay overlay",
		"lowerdir=${PGBRANCH_LOWERS}",
		"upperdir=/pgbranch/rw/upper",
		"workdir=/pgbranch/rw/work",
		"rm -f \"$PGDATA/postmaster.pid\"",
		"exec docker-entrypoint.sh postgres",
	} {
		if !strings.Contains(EntrypointScript, want) {
			t.Fatalf("entrypoint script missing %q", want)
		}
	}
}
```

- [ ] **Step 2: Run — FAIL. Step 3: Implement**

`internal/cow/entrypoint.sh`:
```sh
#!/bin/sh
# pgbranch branch entrypoint: assemble overlay CoW view of the source data
# dir, then hand off to the stock postgres entrypoint (WAL recovery runs there).
set -eu
: "${PGBRANCH_LOWERS:?}" "${PGDATA:?}"
mkdir -p /pgbranch/rw/upper /pgbranch/rw/work "$PGDATA"
mount -t overlay overlay \
  -o "lowerdir=${PGBRANCH_LOWERS},upperdir=/pgbranch/rw/upper,workdir=/pgbranch/rw/work" \
  "$PGDATA"
chown postgres:postgres "$PGDATA"
chmod 0700 "$PGDATA"
rm -f "$PGDATA/postmaster.pid"
exec docker-entrypoint.sh postgres
```

`internal/cow/plan.go`:
```go
// Package cow plans copy-on-write layer layouts for branch containers.
// The overlay mount itself is performed inside the branch container by
// EntrypointScript; host code only decides volume names and mount paths.
package cow

import (
	_ "embed"
	"strings"
)

//go:embed entrypoint.sh
var EntrypointScript string

const (
	MergedPath = "/pgbranch/merged" // PGDATA inside branch container
	RWPath     = "/pgbranch/rw"     // branch rw volume mountpoint
)

type Plan struct {
	SourceVolume string   // mounted ro at Lowers[0]
	RWVolume     string   // upper+work live here
	Lowers       []string // in overlay order, topmost first
}

func SourceVolumeName(source string) string { return "pgbranch-src-" + source }
func BranchRWVolumeName(branch string) string { return "pgbranch-br-" + branch + "-rw" }

func PlanBranch(branchName, sourceVolume string) Plan {
	return Plan{
		SourceVolume: sourceVolume,
		RWVolume:     BranchRWVolumeName(branchName),
		Lowers:       []string{"/pgbranch/lower0"},
	}
}

// LowerEnv renders PGBRANCH_LOWERS for the entrypoint (colon-separated).
func (p Plan) LowerEnv() string { return strings.Join(p.Lowers, ":") }
```

- [ ] **Step 4: Run `go test ./internal/cow/` — PASS. Commit `feat(cow): layer planning and overlay entrypoint script`**

---

### Task 6: Docker runtime driver

**Files:**
- Create: `internal/runtime/runtime.go` (interface + specs), `internal/runtime/docker.go`
- Test: `internal/runtime/docker_test.go` (integration, `PGBRANCH_IT=1` gated)

The driver is deliberately thin: volumes, helper runs (one-shot containers for data ops), branch containers, exec, inspect, remove. Everything else is engine logic.

- [ ] **Step 1: Add dependencies**

```bash
go get github.com/docker/docker@latest github.com/docker/go-connections@latest
```

- [ ] **Step 2: Define the interface**

`internal/runtime/runtime.go`:
```go
// Package runtime abstracts where branch instances run (Docker now, K8s in P3).
package runtime

import "context"

type Mount struct {
	Volume   string
	Target   string
	ReadOnly bool
}

// HelperSpec is a one-shot container performing a data operation
// (seeding, file fixes). Run blocks until exit; non-zero exit = error
// including captured output.
type HelperSpec struct {
	Image   string
	Cmd     []string
	Env     []string
	Mounts  []Mount
	Network string
	User    string // e.g. "postgres" for pg_basebackup so file ownership is uid 999
}

// BranchSpec is a long-running branch Postgres container.
type BranchSpec struct {
	Name       string // container name, e.g. pgbranch-br-pr-1
	Image      string
	Env        []string
	Mounts     []Mount
	Entrypoint []string // overrides image entrypoint
	Labels     map[string]string
	Network    string
}

type ContainerInfo struct {
	ID      string
	Running bool
	Port    int // host port mapped to 5432, 0 if none
	Labels  map[string]string
}

type Driver interface {
	EnsureImage(ctx context.Context, image string) error
	CreateVolume(ctx context.Context, name string, labels map[string]string) error
	RemoveVolume(ctx context.Context, name string) error
	RunHelper(ctx context.Context, spec HelperSpec) error
	StartBranch(ctx context.Context, spec BranchSpec) (id string, err error)
	Exec(ctx context.Context, containerID string, cmd []string) error // error on non-zero exit
	Inspect(ctx context.Context, containerID string) (ContainerInfo, error)
	StopRemove(ctx context.Context, containerID string) error
	ListManaged(ctx context.Context) ([]ContainerInfo, error) // label pgbranch.managed=true
}
```

- [ ] **Step 3: Write failing integration test**

`internal/runtime/docker_test.go`:
```go
package runtime

import (
	"context"
	"os"
	"testing"
	"time"
)

func itDriver(t *testing.T) Driver {
	t.Helper()
	if os.Getenv("PGBRANCH_IT") != "1" {
		t.Skip("set PGBRANCH_IT=1 to run integration tests")
	}
	d, err := NewDockerDriver()
	if err != nil {
		t.Fatal(err)
	}
	return d
}

func TestVolumeAndHelperRoundtrip(t *testing.T) {
	d := itDriver(t)
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()

	vol := "pgbranch-test-vol"
	if err := d.CreateVolume(ctx, vol, map[string]string{"pgbranch.managed": "true"}); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { d.RemoveVolume(context.Background(), vol) })

	if err := d.EnsureImage(ctx, "alpine:3.21"); err != nil {
		t.Fatal(err)
	}
	// write a file via one helper, verify via another
	if err := d.RunHelper(ctx, HelperSpec{
		Image:  "alpine:3.21",
		Cmd:    []string{"sh", "-c", "echo hello > /data/probe"},
		Mounts: []Mount{{Volume: vol, Target: "/data"}},
	}); err != nil {
		t.Fatal(err)
	}
	if err := d.RunHelper(ctx, HelperSpec{
		Image:  "alpine:3.21",
		Cmd:    []string{"sh", "-c", "grep -q hello /data/probe"},
		Mounts: []Mount{{Volume: vol, Target: "/data", ReadOnly: true}},
	}); err != nil {
		t.Fatal(err)
	}
	// failing helper surfaces output in error
	err := d.RunHelper(ctx, HelperSpec{Image: "alpine:3.21", Cmd: []string{"sh", "-c", "echo boom >&2; exit 3"}})
	if err == nil {
		t.Fatal("want error from non-zero helper exit")
	}
}
```

- [ ] **Step 4: Run `PGBRANCH_IT=1 go test ./internal/runtime/ -run Volume -v` — FAIL (NewDockerDriver undefined)**

- [ ] **Step 5: Implement the Docker driver**

`internal/runtime/docker.go`:
```go
package runtime

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"strconv"

	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/api/types/image"
	"github.com/docker/docker/api/types/mount"
	"github.com/docker/docker/api/types/volume"
	"github.com/docker/docker/client"
	"github.com/docker/docker/pkg/stdcopy"
	"github.com/docker/go-connections/nat"
)

type DockerDriver struct{ cli *client.Client }

func NewDockerDriver() (*DockerDriver, error) {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		return nil, fmt.Errorf("docker client: %w", err)
	}
	return &DockerDriver{cli: cli}, nil
}

func (d *DockerDriver) EnsureImage(ctx context.Context, ref string) error {
	if _, err := d.cli.ImageInspect(ctx, ref); err == nil {
		return nil
	}
	rc, err := d.cli.ImagePull(ctx, ref, image.PullOptions{})
	if err != nil {
		return fmt.Errorf("pull %s: %w", ref, err)
	}
	defer rc.Close()
	_, err = io.Copy(io.Discard, rc)
	return err
}

func (d *DockerDriver) CreateVolume(ctx context.Context, name string, labels map[string]string) error {
	_, err := d.cli.VolumeCreate(ctx, volume.CreateOptions{Name: name, Labels: labels})
	return err
}

func (d *DockerDriver) RemoveVolume(ctx context.Context, name string) error {
	return d.cli.VolumeRemove(ctx, name, true)
}

func toMounts(ms []Mount) []mount.Mount {
	out := make([]mount.Mount, 0, len(ms))
	for _, m := range ms {
		out = append(out, mount.Mount{Type: mount.TypeVolume, Source: m.Volume, Target: m.Target, ReadOnly: m.ReadOnly})
	}
	return out
}

func (d *DockerDriver) RunHelper(ctx context.Context, spec HelperSpec) error {
	if err := d.EnsureImage(ctx, spec.Image); err != nil {
		return err
	}
	cfg := &container.Config{Image: spec.Image, Cmd: spec.Cmd, Env: spec.Env, User: spec.User,
		Labels: map[string]string{"pgbranch.managed": "true", "pgbranch.role": "helper"}}
	host := &container.HostConfig{Mounts: toMounts(spec.Mounts), NetworkMode: container.NetworkMode(spec.Network)}
	cr, err := d.cli.ContainerCreate(ctx, cfg, host, nil, nil, "")
	if err != nil {
		return fmt.Errorf("create helper: %w", err)
	}
	defer d.cli.ContainerRemove(context.WithoutCancel(ctx), cr.ID, container.RemoveOptions{Force: true})
	if err := d.cli.ContainerStart(ctx, cr.ID, container.StartOptions{}); err != nil {
		return fmt.Errorf("start helper: %w", err)
	}
	waitC, errC := d.cli.ContainerWait(ctx, cr.ID, container.WaitConditionNotRunning)
	select {
	case err := <-errC:
		return err
	case st := <-waitC:
		if st.StatusCode != 0 {
			return fmt.Errorf("helper exited %d: %s", st.StatusCode, d.logs(ctx, cr.ID))
		}
		return nil
	}
}

func (d *DockerDriver) logs(ctx context.Context, id string) string {
	rc, err := d.cli.ContainerLogs(ctx, id, container.LogsOptions{ShowStdout: true, ShowStderr: true, Tail: "20"})
	if err != nil {
		return ""
	}
	defer rc.Close()
	var buf bytes.Buffer
	stdcopy.StdCopy(&buf, &buf, rc)
	return buf.String()
}

func (d *DockerDriver) StartBranch(ctx context.Context, spec BranchSpec) (string, error) {
	if err := d.EnsureImage(ctx, spec.Image); err != nil {
		return "", err
	}
	cfg := &container.Config{
		Image: spec.Image, Env: spec.Env, Entrypoint: spec.Entrypoint, Labels: spec.Labels,
		ExposedPorts: nat.PortSet{"5432/tcp": struct{}{}},
	}
	host := &container.HostConfig{
		Mounts:       toMounts(spec.Mounts),
		CapAdd:       []string{"SYS_ADMIN"},                  // overlay mount inside container
		SecurityOpt:  []string{"apparmor=unconfined"},        // no-op where AppArmor absent
		PortBindings: nat.PortMap{"5432/tcp": {{HostIP: "127.0.0.1", HostPort: ""}}}, // random host port
		NetworkMode:  container.NetworkMode(spec.Network),
		RestartPolicy: container.RestartPolicy{Name: container.RestartPolicyUnlessStopped},
	}
	cr, err := d.cli.ContainerCreate(ctx, cfg, host, nil, nil, spec.Name)
	if err != nil {
		return "", fmt.Errorf("create branch container: %w", err)
	}
	if err := d.cli.ContainerStart(ctx, cr.ID, container.StartOptions{}); err != nil {
		d.cli.ContainerRemove(context.WithoutCancel(ctx), cr.ID, container.RemoveOptions{Force: true})
		return "", fmt.Errorf("start branch container: %w", err)
	}
	return cr.ID, nil
}

func (d *DockerDriver) Exec(ctx context.Context, id string, cmd []string) error {
	ex, err := d.cli.ContainerExecCreate(ctx, id, container.ExecOptions{Cmd: cmd, AttachStdout: true, AttachStderr: true})
	if err != nil {
		return err
	}
	att, err := d.cli.ContainerExecAttach(ctx, ex.ID, container.ExecStartOptions{})
	if err != nil {
		return err
	}
	defer att.Close()
	var buf bytes.Buffer
	stdcopy.StdCopy(&buf, &buf, att.Reader)
	insp, err := d.cli.ContainerExecInspect(ctx, ex.ID)
	if err != nil {
		return err
	}
	if insp.ExitCode != 0 {
		return fmt.Errorf("exec %v exited %d: %s", cmd, insp.ExitCode, buf.String())
	}
	return nil
}

func (d *DockerDriver) Inspect(ctx context.Context, id string) (ContainerInfo, error) {
	j, err := d.cli.ContainerInspect(ctx, id)
	if err != nil {
		return ContainerInfo{}, err
	}
	info := ContainerInfo{ID: j.ID, Running: j.State != nil && j.State.Running, Labels: j.Config.Labels}
	if b, ok := j.NetworkSettings.Ports["5432/tcp"]; ok && len(b) > 0 {
		info.Port, _ = strconv.Atoi(b[0].HostPort)
	}
	return info, nil
}

func (d *DockerDriver) StopRemove(ctx context.Context, id string) error {
	timeout := 30
	_ = d.cli.ContainerStop(ctx, id, container.StopOptions{Timeout: &timeout})
	err := d.cli.ContainerRemove(ctx, id, container.RemoveOptions{Force: true})
	if client.IsErrNotFound(err) {
		return nil
	}
	return err
}

func (d *DockerDriver) ListManaged(ctx context.Context) ([]ContainerInfo, error) {
	f := filters.NewArgs(filters.Arg("label", "pgbranch.managed=true"), filters.Arg("label", "pgbranch.role=branch"))
	cs, err := d.cli.ContainerList(ctx, container.ListOptions{All: true, Filters: f})
	if err != nil {
		return nil, err
	}
	out := make([]ContainerInfo, 0, len(cs))
	for _, c := range cs {
		out = append(out, ContainerInfo{ID: c.ID, Running: c.State == "running", Labels: c.Labels})
	}
	return out, nil
}
```

Note: exact SDK type names move between Docker SDK versions (e.g. `ImageInspect` signature, `container.ExecOptions`). If compilation fails, check the installed version's godoc and adjust — the *behavior* in the test is the contract, not these exact calls.

- [ ] **Step 6: Run `PGBRANCH_IT=1 go test ./internal/runtime/ -v` — PASS. Commit `feat(runtime): docker driver for volumes, helpers, branch containers`**

---

### Task 7: pgctl — seeding and Postgres control

**Files:**
- Create: `internal/pgctl/seed.go`
- Test: `internal/pgctl/seed_test.go` (integration)

Seeding = `pg_basebackup` from the user's source DB into the source volume, run as a helper container under user `postgres` (uid 999) so ownership matches the branch containers.

- [ ] **Step 1: Write failing integration test**

Uses testcontainers to stand up a "production" Postgres, then seeds from it.

```bash
go get github.com/testcontainers/testcontainers-go@latest github.com/jackc/pgx/v5@latest
```

```go
package pgctl

import (
	"context"
	"fmt"
	"os"
	"testing"
	"time"

	"github.com/jackc/pgx/v5"
	tc "github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"

	"github.com/abd-ulbasit/pgbranch/internal/runtime"
)

// StartSourcePG starts a "production" postgres on a dedicated docker network
// and returns its container, network name, and a host connection string.
func StartSourcePG(t *testing.T, ctx context.Context) (host string, port int, network string, hostConn string) {
	t.Helper()
	net, err := tc.GenericNetwork(ctx, tc.GenericNetworkRequest{NetworkRequest: tc.NetworkRequest{Name: "pgbranch-it-net", CheckDuplicate: true}})
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { net.Remove(context.Background()) })
	req := tc.ContainerRequest{
		Image:        "postgres:17",
		Env:          map[string]string{"POSTGRES_PASSWORD": "secret"},
		Networks:     []string{"pgbranch-it-net"},
		NetworkAliases: map[string][]string{"pgbranch-it-net": {"sourcedb"}},
		ExposedPorts: []string{"5432/tcp"},
		Cmd:          []string{"-c", "wal_level=replica", "-c", "max_wal_senders=4"},
		WaitingFor:   wait.ForListeningPort("5432/tcp").WithStartupTimeout(60 * time.Second),
	}
	c, err := tc.GenericContainer(ctx, tc.GenericContainerRequest{ContainerRequest: req, Started: true})
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { c.Terminate(context.Background()) })
	mp, err := c.MappedPort(ctx, "5432")
	if err != nil {
		t.Fatal(err)
	}
	return "sourcedb", 5432, "pgbranch-it-net", fmt.Sprintf("postgres://postgres:secret@localhost:%d/postgres", mp.Int())
}

func TestSeedFromRunningPostgres(t *testing.T) {
	if os.Getenv("PGBRANCH_IT") != "1" {
		t.Skip("set PGBRANCH_IT=1")
	}
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
	defer cancel()
	host, port, network, hostConn := StartSourcePG(t, ctx)

	conn, err := pgx.Connect(ctx, hostConn)
	if err != nil {
		t.Fatal(err)
	}
	if _, err := conn.Exec(ctx, `CREATE TABLE t(i int); INSERT INTO t SELECT generate_series(1,1000)`); err != nil {
		t.Fatal(err)
	}
	conn.Close(ctx)

	d, err := runtime.NewDockerDriver()
	if err != nil {
		t.Fatal(err)
	}
	vol := "pgbranch-test-seed"
	if err := d.CreateVolume(ctx, vol, nil); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { d.RemoveVolume(context.Background(), vol) })

	err = Seed(ctx, d, SeedSpec{
		Image: "postgres:17", Volume: vol, Network: network,
		Host: host, Port: port, User: "postgres", Password: "secret",
	})
	if err != nil {
		t.Fatal(err)
	}
	// PG_VERSION present => valid cluster layout
	if err := d.RunHelper(ctx, runtime.HelperSpec{
		Image: "alpine:3.21", Cmd: []string{"test", "-f", "/seed/PG_VERSION"},
		Mounts: []runtime.Mount{{Volume: vol, Target: "/seed", ReadOnly: true}},
	}); err != nil {
		t.Fatalf("seed volume missing PG_VERSION: %v", err)
	}
}
```

- [ ] **Step 2: Run — FAIL. Step 3: Implement**

```go
// Package pgctl runs Postgres-side operations (seeding, readiness) through
// the runtime driver — pgbranch never touches data files from the host.
package pgctl

import (
	"context"
	"fmt"
	"strconv"

	"github.com/abd-ulbasit/pgbranch/internal/runtime"
)

type SeedSpec struct {
	Image   string // postgres image matching the source's major version
	Volume  string // target source volume
	Network string // docker network from which the source is reachable ("" = bridge)
	Host    string
	Port    int
	User    string
	Password string
}

// Seed runs pg_basebackup into the source volume. The helper runs as the
// in-image postgres user (uid 999) so file ownership matches branch containers.
// Requires REPLICATION privilege on the source (superuser works).
func Seed(ctx context.Context, d runtime.Driver, s SeedSpec) error {
	err := d.RunHelper(ctx, runtime.HelperSpec{
		Image: s.Image,
		User:  "postgres",
		Cmd: []string{"pg_basebackup",
			"-h", s.Host, "-p", strconv.Itoa(s.Port), "-U", s.User,
			"-D", "/seed/data", "-X", "stream", "--checkpoint=fast", "--no-password"},
		Env:     []string{"PGPASSWORD=" + s.Password},
		Mounts:  []runtime.Mount{{Volume: s.Volume, Target: "/seed"}},
		Network: s.Network,
	})
	if err != nil {
		return fmt.Errorf("pg_basebackup: %w", err)
	}
	return nil
}
```

Note the data lands in `/seed/data` (subdir, because the volume root is owned by root and `pg_basebackup` wants to create the dir itself with 0700). A pre-step is needed: volume root must be writable by uid 999. Add this as the first helper inside `Seed`, before basebackup:

```go
	if err := d.RunHelper(ctx, runtime.HelperSpec{
		Image:  "alpine:3.21",
		Cmd:    []string{"sh", "-c", "mkdir -p /seed && chown 999:999 /seed"},
		Mounts: []runtime.Mount{{Volume: s.Volume, Target: "/seed"}},
	}); err != nil {
		return fmt.Errorf("prepare seed volume: %w", err)
	}
```

The branch overlay lower path therefore is `<source volume>/data` — Task 8 mounts the source volume at `/pgbranch/lower0` and sets `PGBRANCH_LOWERS=/pgbranch/lower0/data`. Update `internal/cow/plan.go`: `PlanBranch` returns `Lowers: []string{"/pgbranch/lower0/data"}` and the cow unit test accordingly.

- [ ] **Step 4: Run `PGBRANCH_IT=1 go test ./internal/pgctl/ -v` — PASS (first run pulls postgres:17, allow time). Commit `feat(pgctl): source seeding via pg_basebackup helper`**

---

### Task 8: Engine — branch create/destroy saga + reconcile

**Files:**
- Create: `internal/engine/engine.go`, `internal/engine/saga.go`
- Test: `internal/engine/engine_test.go` (unit, fake driver), `internal/engine/engine_it_test.go` (integration e2e)

- [ ] **Step 1: Write failing unit test with a fake driver**

```go
package engine

import (
	"context"
	"errors"
	"path/filepath"
	"testing"

	"github.com/abd-ulbasit/pgbranch/internal/registry"
	"github.com/abd-ulbasit/pgbranch/internal/runtime"
)

type fakeDriver struct {
	volumes    map[string]bool
	containers map[string]bool
	failStart  bool
	execErr    error
}

func newFake() *fakeDriver {
	return &fakeDriver{volumes: map[string]bool{}, containers: map[string]bool{}}
}
func (f *fakeDriver) EnsureImage(ctx context.Context, image string) error { return nil }
func (f *fakeDriver) CreateVolume(ctx context.Context, name string, l map[string]string) error {
	f.volumes[name] = true
	return nil
}
func (f *fakeDriver) RemoveVolume(ctx context.Context, name string) error {
	delete(f.volumes, name)
	return nil
}
func (f *fakeDriver) RunHelper(ctx context.Context, s runtime.HelperSpec) error { return nil }
func (f *fakeDriver) StartBranch(ctx context.Context, s runtime.BranchSpec) (string, error) {
	if f.failStart {
		return "", errors.New("boom")
	}
	f.containers["cid-"+s.Name] = true
	return "cid-" + s.Name, nil
}
func (f *fakeDriver) Exec(ctx context.Context, id string, cmd []string) error { return f.execErr }
func (f *fakeDriver) Inspect(ctx context.Context, id string) (runtime.ContainerInfo, error) {
	return runtime.ContainerInfo{ID: id, Running: f.containers[id], Port: 54321}, nil
}
func (f *fakeDriver) StopRemove(ctx context.Context, id string) error {
	delete(f.containers, id)
	return nil
}
func (f *fakeDriver) ListManaged(ctx context.Context) ([]runtime.ContainerInfo, error) {
	var out []runtime.ContainerInfo
	for id := range f.containers {
		out = append(out, runtime.ContainerInfo{ID: id, Running: true})
	}
	return out, nil
}

func testEngine(t *testing.T, d runtime.Driver) (*Engine, *registry.Registry) {
	t.Helper()
	r, err := registry.Open(filepath.Join(t.TempDir(), "t.db"))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { r.Close() })
	return New(r, d, "postgres:17"), r
}

func readySource(t *testing.T, r *registry.Registry) *registry.Source {
	t.Helper()
	s := &registry.Source{Name: "main", PGVersion: "17", Volume: "pgbranch-src-main"}
	if err := r.CreateSource(s); err != nil {
		t.Fatal(err)
	}
	if err := r.SetSourceState(s.ID, registry.SourceReady, "test"); err != nil {
		t.Fatal(err)
	}
	return s
}

func TestCreateBranchHappyPath(t *testing.T) {
	d := newFake()
	e, r := testEngine(t, d)
	readySource(t, r)

	b, err := e.CreateBranch(context.Background(), "pr-1", "main")
	if err != nil {
		t.Fatal(err)
	}
	if b.State != registry.BranchReady || b.Port != 54321 {
		t.Fatalf("branch %+v", b)
	}
	if !d.volumes["pgbranch-br-pr-1-rw"] {
		t.Fatal("rw volume not created")
	}
	if !d.containers["cid-pgbranch-br-pr-1"] {
		t.Fatal("container not started")
	}
}

func TestCreateBranchUnwindsOnStartFailure(t *testing.T) {
	d := newFake()
	d.failStart = true
	e, r := testEngine(t, d)
	readySource(t, r)

	if _, err := e.CreateBranch(context.Background(), "pr-1", "main"); err == nil {
		t.Fatal("want error")
	}
	if len(d.volumes) != 0 {
		t.Fatalf("rw volume leaked: %v", d.volumes)
	}
	b, err := r.GetBranchByName("pr-1")
	if err != nil {
		t.Fatal(err)
	}
	if b.State != registry.BranchFailed {
		t.Fatalf("state=%q want failed", b.State)
	}
}

func TestDestroyBranch(t *testing.T) {
	d := newFake()
	e, r := testEngine(t, d)
	readySource(t, r)
	if _, err := e.CreateBranch(context.Background(), "pr-1", "main"); err != nil {
		t.Fatal(err)
	}
	if err := e.DestroyBranch(context.Background(), "pr-1"); err != nil {
		t.Fatal(err)
	}
	if len(d.containers) != 0 || len(d.volumes) != 0 {
		t.Fatalf("leaked: c=%v v=%v", d.containers, d.volumes)
	}
	if _, err := r.GetBranchByName("pr-1"); !errors.Is(err, registry.ErrNotFound) {
		t.Fatalf("want gone, got %v", err)
	}
}
```

- [ ] **Step 2: Run `go test ./internal/engine/` — FAIL. Step 3: Implement**

`internal/engine/engine.go`:
```go
// Package engine orchestrates branch lifecycle as sagas over the registry,
// cow planner, and runtime driver. The CLI (P1) and branchd (P2) both embed it.
package engine

import (
	"context"
	"fmt"
	"time"

	"github.com/abd-ulbasit/pgbranch/internal/cow"
	"github.com/abd-ulbasit/pgbranch/internal/pgctl"
	"github.com/abd-ulbasit/pgbranch/internal/registry"
	"github.com/abd-ulbasit/pgbranch/internal/runtime"
)

type Engine struct {
	reg          *registry.Registry
	drv          runtime.Driver
	defaultImage string
}

func New(reg *registry.Registry, drv runtime.Driver, defaultImage string) *Engine {
	return &Engine{reg: reg, drv: drv, defaultImage: defaultImage}
}

func (e *Engine) image(pgVersion string) string {
	if pgVersion == "" {
		return e.defaultImage
	}
	return "postgres:" + pgVersion
}

// AddSource registers a source and seeds it from the given live Postgres.
func (e *Engine) AddSource(ctx context.Context, s *registry.Source, password string) error {
	s.Volume = cow.SourceVolumeName(s.Name)
	if err := e.reg.CreateSource(s); err != nil {
		return err
	}
	if err := e.drv.CreateVolume(ctx, s.Volume, map[string]string{"pgbranch.managed": "true", "pgbranch.source.name": s.Name}); err != nil {
		e.reg.SetSourceState(s.ID, registry.SourceFailed, "volume create failed")
		return err
	}
	err := pgctl.Seed(ctx, e.drv, pgctl.SeedSpec{
		Image: e.image(s.PGVersion), Volume: s.Volume, Network: s.Network,
		Host: s.ConnHost, Port: s.ConnPort, User: s.ConnUser, Password: password,
	})
	if err != nil {
		e.drv.RemoveVolume(context.WithoutCancel(ctx), s.Volume)
		e.reg.SetSourceState(s.ID, registry.SourceFailed, err.Error())
		return fmt.Errorf("seed source %q: %w", s.Name, err)
	}
	return e.reg.SetSourceState(s.ID, registry.SourceReady, "seed complete")
}
```

`internal/engine/saga.go`:
```go
package engine

import (
	"context"
	"fmt"
	"time"

	"github.com/abd-ulbasit/pgbranch/internal/cow"
	"github.com/abd-ulbasit/pgbranch/internal/registry"
	"github.com/abd-ulbasit/pgbranch/internal/runtime"
)

// CreateBranch is a saga: every step registers a compensation that runs
// (in reverse order) if a later step fails. No orphans, ever.
func (e *Engine) CreateBranch(ctx context.Context, name, sourceName string) (*registry.Branch, error) {
	src, err := e.reg.GetSourceByName(sourceName)
	if err != nil {
		return nil, fmt.Errorf("source %q: %w", sourceName, err)
	}
	if src.State != registry.SourceReady {
		return nil, fmt.Errorf("source %q is %s, not ready", sourceName, src.State)
	}
	plan := cow.PlanBranch(name, src.Volume)
	b := &registry.Branch{Name: name, SourceID: src.ID, RWVolume: plan.RWVolume}
	if err := e.reg.CreateBranch(b); err != nil {
		return nil, err
	}

	var undo []func()
	fail := func(stepErr error) (*registry.Branch, error) {
		for i := len(undo) - 1; i >= 0; i-- {
			undo[i]()
		}
		e.reg.TransitionBranch(b.ID, registry.BranchFailed, stepErr.Error())
		return nil, stepErr
	}
	bg := context.WithoutCancel(ctx)

	// 1. rw volume (upper/work + entrypoint script live here)
	if err := e.drv.CreateVolume(ctx, plan.RWVolume, map[string]string{"pgbranch.managed": "true", "pgbranch.branch.id": b.ID}); err != nil {
		return fail(fmt.Errorf("create rw volume: %w", err))
	}
	undo = append(undo, func() { e.drv.RemoveVolume(bg, plan.RWVolume) })

	// 2. write entrypoint into the rw volume
	if err := e.drv.RunHelper(ctx, runtime.HelperSpec{
		Image:  "alpine:3.21",
		Cmd:    []string{"sh", "-c", `printf '%s' "$PGBRANCH_ENTRYPOINT" > /pgbranch/rw/entrypoint.sh && chmod 0755 /pgbranch/rw/entrypoint.sh && mkdir -p /pgbranch/rw/upper /pgbranch/rw/work`},
		Env:    []string{"PGBRANCH_ENTRYPOINT=" + cow.EntrypointScript},
		Mounts: []runtime.Mount{{Volume: plan.RWVolume, Target: cow.RWPath}},
	}); err != nil {
		return fail(fmt.Errorf("install entrypoint: %w", err))
	}

	// 3. branch container
	cid, err := e.drv.StartBranch(ctx, runtime.BranchSpec{
		Name:  "pgbranch-br-" + name,
		Image: e.image(src.PGVersion),
		Env: []string{
			"PGDATA=" + cow.MergedPath,
			"PGBRANCH_LOWERS=" + plan.LowerEnv(),
		},
		Mounts: []runtime.Mount{
			{Volume: src.Volume, Target: "/pgbranch/lower0", ReadOnly: true},
			{Volume: plan.RWVolume, Target: cow.RWPath},
		},
		Entrypoint: []string{"/bin/sh", cow.RWPath + "/entrypoint.sh"},
		Labels: map[string]string{
			"pgbranch.managed": "true", "pgbranch.role": "branch",
			"pgbranch.branch.id": b.ID, "pgbranch.branch.name": name,
		},
	})
	if err != nil {
		return fail(fmt.Errorf("start instance: %w", err))
	}
	undo = append(undo, func() { e.drv.StopRemove(bg, cid) })

	// 4. wait for postgres readiness (covers WAL recovery time)
	if err := e.waitReady(ctx, cid, 90*time.Second); err != nil {
		return fail(fmt.Errorf("instance never became ready: %w", err))
	}

	// 5. record container + host port, mark ready
	info, err := e.drv.Inspect(ctx, cid)
	if err != nil {
		return fail(err)
	}
	if err := e.reg.MarkBranchReady(b.ID, cid, info.Port); err != nil {
		return fail(err)
	}
	return e.reg.GetBranchByName(name)
}

func (e *Engine) waitReady(ctx context.Context, cid string, timeout time.Duration) error {
	deadline := time.Now().Add(timeout)
	var lastErr error
	for time.Now().Before(deadline) {
		lastErr = e.drv.Exec(ctx, cid, []string{"pg_isready", "-U", "postgres", "-h", "/var/run/postgresql"})
		if lastErr == nil {
			return nil
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(time.Second):
		}
	}
	return lastErr
}

func (e *Engine) DestroyBranch(ctx context.Context, name string) error {
	b, err := e.reg.GetBranchByName(name)
	if err != nil {
		return err
	}
	if err := e.reg.TransitionBranch(b.ID, registry.BranchDestroying, "destroy requested"); err != nil {
		return err
	}
	if b.ContainerID != "" {
		if err := e.drv.StopRemove(ctx, b.ContainerID); err != nil {
			return fmt.Errorf("remove container: %w", err)
		}
	}
	if err := e.drv.RemoveVolume(ctx, b.RWVolume); err != nil {
		return fmt.Errorf("remove rw volume: %w", err)
	}
	return e.reg.TransitionBranch(b.ID, registry.BranchDestroyed, "")
}
```

(The `time` import in engine.go is only needed if used there — keep imports tidy per file; `goimports` settles it.)

- [ ] **Step 4: Run `go test ./internal/engine/` — PASS. Commit `feat(engine): branch create/destroy sagas`**

- [ ] **Step 5: Add startup reconciliation — failing test first**

```go
func TestReconcileCleansOrphans(t *testing.T) {
	d := newFake()
	e, r := testEngine(t, d)
	readySource(t, r)
	// registry says creating, but no container exists -> failed + cleaned
	b := &registry.Branch{Name: "stuck", SourceID: mustSource(t, r).ID, RWVolume: "pgbranch-br-stuck-rw"}
	if err := r.CreateBranch(b); err != nil {
		t.Fatal(err)
	}
	d.volumes["pgbranch-br-stuck-rw"] = true
	// container exists but registry has no row -> removed
	d.containers["cid-ghost"] = true

	if err := e.Reconcile(context.Background()); err != nil {
		t.Fatal(err)
	}
	got, _ := r.GetBranchByName("stuck")
	if got.State != registry.BranchFailed {
		t.Fatalf("state=%q", got.State)
	}
	if d.containers["cid-ghost"] {
		t.Fatal("ghost container not removed")
	}
}
```

(`mustSource` helper: `func mustSource(t *testing.T, r *registry.Registry) *registry.Source { s, err := r.GetSourceByName("main"); if err != nil { t.Fatal(err) }; return s }`)

- [ ] **Step 6: Implement `Reconcile` in `engine.go`**

```go
// Reconcile aligns the registry with reality at startup: stuck 'creating'
// branches are failed and their resources cleaned; managed containers with
// no registry row are removed.
func (e *Engine) Reconcile(ctx context.Context) error {
	branches, err := e.reg.ListLiveBranches()
	if err != nil {
		return err
	}
	known := map[string]bool{}
	for _, b := range branches {
		if b.ContainerID != "" {
			known[b.ContainerID] = true
		}
		if b.State == registry.BranchCreating {
			if b.ContainerID != "" {
				e.drv.StopRemove(ctx, b.ContainerID)
			}
			e.drv.RemoveVolume(ctx, b.RWVolume)
			e.reg.TransitionBranch(b.ID, registry.BranchFailed, "reconcile: interrupted create")
		}
	}
	managed, err := e.drv.ListManaged(ctx)
	if err != nil {
		return err
	}
	for _, c := range managed {
		if !known[c.ID] {
			e.drv.StopRemove(ctx, c.ID)
		}
	}
	return nil
}
```

- [ ] **Step 7: Run `go test ./internal/engine/` — PASS. Commit `feat(engine): startup reconciliation`**

---

### Task 9: End-to-end integration test (the money test)

**Files:**
- Create: `internal/engine/engine_it_test.go`

This is the test that proves the product: seed from a live DB, branch it, verify data, verify isolation between branches and from the source.

- [ ] **Step 1: Write the test**

```go
package engine

import (
	"context"
	"fmt"
	"os"
	"testing"
	"time"

	"github.com/jackc/pgx/v5"

	"github.com/abd-ulbasit/pgbranch/internal/pgctl"
	"github.com/abd-ulbasit/pgbranch/internal/registry"
	"github.com/abd-ulbasit/pgbranch/internal/runtime"
)

func branchConn(b *registry.Branch) string {
	return fmt.Sprintf("postgres://postgres:secret@localhost:%d/postgres", b.Port)
}

func mustQueryInt(t *testing.T, ctx context.Context, conn string, q string) int {
	t.Helper()
	c, err := pgx.Connect(ctx, conn)
	if err != nil {
		t.Fatal(err)
	}
	defer c.Close(ctx)
	var n int
	if err := c.QueryRow(ctx, q).Scan(&n); err != nil {
		t.Fatal(err)
	}
	return n
}

func mustExec(t *testing.T, ctx context.Context, conn string, q string) {
	t.Helper()
	c, err := pgx.Connect(ctx, conn)
	if err != nil {
		t.Fatal(err)
	}
	defer c.Close(ctx)
	if _, err := c.Exec(ctx, q); err != nil {
		t.Fatal(err)
	}
}

func TestEndToEndBranching(t *testing.T) {
	if os.Getenv("PGBRANCH_IT") != "1" {
		t.Skip("set PGBRANCH_IT=1")
	}
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Minute)
	defer cancel()

	host, port, network, hostConn := pgctl.StartSourcePG(t, ctx)
	mustExec(t, ctx, hostConn, `CREATE TABLE accounts(id int primary key, balance int);
		INSERT INTO accounts SELECT i, 100 FROM generate_series(1,10000) i`)

	d, err := runtime.NewDockerDriver()
	if err != nil {
		t.Fatal(err)
	}
	r, err := registry.Open(t.TempDir() + "/it.db")
	if err != nil {
		t.Fatal(err)
	}
	defer r.Close()
	e := New(r, d, "postgres:17")

	src := &registry.Source{Name: "main", PGVersion: "17", ConnHost: host, ConnPort: port, ConnUser: "postgres", Network: network}
	if err := e.AddSource(ctx, src, "secret"); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { d.RemoveVolume(context.Background(), src.Volume) })

	start := time.Now()
	b1, err := e.CreateBranch(ctx, "pr-1", "main")
	if err != nil {
		t.Fatal(err)
	}
	t.Logf("branch pr-1 created in %s", time.Since(start))
	t.Cleanup(func() { e.DestroyBranch(context.Background(), "pr-1") })

	// branch sees source data
	if n := mustQueryInt(t, ctx, branchConn(b1), `SELECT count(*) FROM accounts`); n != 10000 {
		t.Fatalf("branch rows = %d", n)
	}
	// writes to branch do not affect source
	mustExec(t, ctx, branchConn(b1), `UPDATE accounts SET balance = 0`)
	if n := mustQueryInt(t, ctx, hostConn, `SELECT sum(balance) FROM accounts`); n != 1000000 {
		t.Fatalf("source mutated! sum=%d", n)
	}
	// second branch is isolated from first
	b2, err := e.CreateBranch(ctx, "pr-2", "main")
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { e.DestroyBranch(context.Background(), "pr-2") })
	if n := mustQueryInt(t, ctx, branchConn(b2), `SELECT sum(balance) FROM accounts`); n != 1000000 {
		t.Fatalf("pr-2 saw pr-1 writes, sum=%d", n)
	}
}
```

Note: branch containers reuse the source cluster's credentials — the seeded cluster from `StartSourcePG` has superuser `postgres`/`secret` with `trust`-or-md5 hba as produced by the official image. If auth fails when connecting to the branch, inspect `docker logs pgbranch-br-pr-1`; the seeded `pg_hba.conf` is authoritative.

- [ ] **Step 2: Run `PGBRANCH_IT=1 go test ./internal/engine/ -run EndToEnd -v` — expect PASS; debug overlay/entrypoint issues here (this is the riskiest task; budget time). Common failure modes: AppArmor blocking mount (check `docker logs`), `PGBRANCH_LOWERS` path wrong (`/pgbranch/lower0/data`), upper/work perms.**

- [ ] **Step 3: Commit `test(engine): end-to-end branching with isolation verification`**

---

### Task 10: CLI

**Files:**
- Create: `internal/cli/root.go`, `internal/cli/source.go`, `internal/cli/branch.go`
- Modify: `cmd/pgb/main.go`
- Test: `internal/cli/cli_test.go`

```bash
go get github.com/spf13/cobra@latest
```

- [ ] **Step 1: Write failing test (command wiring, not Docker)**

```go
package cli

import (
	"bytes"
	"testing"
)

func TestCommandTree(t *testing.T) {
	root := NewRootCmd()
	for _, path := range [][]string{
		{"source", "add"}, {"source", "ls"},
		{"branch", "create"}, {"branch", "ls"}, {"branch", "destroy"},
		{"connect"},
	} {
		cmd, _, err := root.Find(path)
		if err != nil || cmd.Name() != path[len(path)-1] {
			t.Fatalf("command %v not found: %v", path, err)
		}
	}
	// help renders without side effects
	var buf bytes.Buffer
	root.SetOut(&buf)
	root.SetArgs([]string{"--help"})
	if err := root.Execute(); err != nil {
		t.Fatal(err)
	}
}
```

- [ ] **Step 2: Run — FAIL. Step 3: Implement**

`internal/cli/root.go`:
```go
// Package cli wires cobra commands to the engine. Engine construction is
// lazy (inside RunE) so --help and tests never touch Docker.
package cli

import (
	"github.com/spf13/cobra"

	"github.com/abd-ulbasit/pgbranch/internal/config"
	"github.com/abd-ulbasit/pgbranch/internal/engine"
	"github.com/abd-ulbasit/pgbranch/internal/registry"
	"github.com/abd-ulbasit/pgbranch/internal/runtime"
)

func NewRootCmd() *cobra.Command {
	root := &cobra.Command{
		Use:           "pgb",
		Short:         "pgbranch — git branch for Postgres",
		SilenceUsage:  true,
		SilenceErrors: false,
	}
	root.AddCommand(newSourceCmd(), newBranchCmd(), newConnectCmd())
	return root
}

// open builds the engine; callers must Close the returned registry.
func open() (*engine.Engine, *registry.Registry, error) {
	cfg, err := config.Load()
	if err != nil {
		return nil, nil, err
	}
	if err := cfg.EnsureHome(); err != nil {
		return nil, nil, err
	}
	reg, err := registry.Open(cfg.RegistryPath)
	if err != nil {
		return nil, nil, err
	}
	drv, err := runtime.NewDockerDriver()
	if err != nil {
		reg.Close()
		return nil, nil, err
	}
	return engine.New(reg, drv, cfg.PostgresImage), reg, nil
}
```

`internal/cli/source.go`:
```go
package cli

import (
	"fmt"
	"os"
	"text/tabwriter"

	"github.com/spf13/cobra"

	"github.com/abd-ulbasit/pgbranch/internal/registry"
)

func newSourceCmd() *cobra.Command {
	cmd := &cobra.Command{Use: "source", Short: "Manage branch sources (seeded data dirs)"}
	cmd.AddCommand(newSourceAddCmd(), newSourceLsCmd())
	return cmd
}

func newSourceAddCmd() *cobra.Command {
	var host, user, db, network, pgVersion, passwordEnv string
	var port int
	cmd := &cobra.Command{
		Use:   "add NAME",
		Short: "Register a source and seed it from a running Postgres (needs REPLICATION privilege)",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			password := os.Getenv(passwordEnv)
			if password == "" {
				return fmt.Errorf("password env %q is empty", passwordEnv)
			}
			e, reg, err := open()
			if err != nil {
				return err
			}
			defer reg.Close()
			s := &registry.Source{Name: args[0], PGVersion: pgVersion,
				ConnHost: host, ConnPort: port, ConnUser: user, ConnDB: db, Network: network}
			if err := e.AddSource(cmd.Context(), s, password); err != nil {
				return err
			}
			fmt.Fprintf(cmd.OutOrStdout(), "source %q seeded and ready\n", s.Name)
			return nil
		},
	}
	cmd.Flags().StringVar(&host, "host", "", "source Postgres host (as reachable from containers; use host.docker.internal for a host-local DB)")
	cmd.Flags().IntVar(&port, "port", 5432, "source Postgres port")
	cmd.Flags().StringVar(&user, "user", "postgres", "user with REPLICATION privilege")
	cmd.Flags().StringVar(&db, "database", "postgres", "database name recorded for connection strings")
	cmd.Flags().StringVar(&network, "network", "", "docker network from which the source is reachable")
	cmd.Flags().StringVar(&pgVersion, "pg-version", "17", "source Postgres major version (branch image must match)")
	cmd.Flags().StringVar(&passwordEnv, "password-env", "PGPASSWORD", "env var holding the source password")
	cmd.MarkFlagRequired("host")
	return cmd
}

func newSourceLsCmd() *cobra.Command {
	return &cobra.Command{
		Use: "ls", Short: "List sources",
		RunE: func(cmd *cobra.Command, args []string) error {
			_, reg, err := open()
			if err != nil {
				return err
			}
			defer reg.Close()
			sources, err := reg.ListSources()
			if err != nil {
				return err
			}
			w := tabwriter.NewWriter(cmd.OutOrStdout(), 2, 4, 2, ' ', 0)
			fmt.Fprintln(w, "NAME\tPG\tSTATE\tCREATED")
			for _, s := range sources {
				fmt.Fprintf(w, "%s\t%s\t%s\t%s\n", s.Name, s.PGVersion, s.State, s.CreatedAt)
			}
			return w.Flush()
		},
	}
}
```

`internal/cli/branch.go`:
```go
package cli

import (
	"fmt"
	"text/tabwriter"
	"time"

	"github.com/spf13/cobra"
)

func newBranchCmd() *cobra.Command {
	cmd := &cobra.Command{Use: "branch", Short: "Manage branches"}
	cmd.AddCommand(newBranchCreateCmd(), newBranchLsCmd(), newBranchDestroyCmd())
	return cmd
}

func newBranchCreateCmd() *cobra.Command {
	var from string
	cmd := &cobra.Command{
		Use:   "create NAME",
		Short: "Create an instant copy-on-write branch",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			e, reg, err := open()
			if err != nil {
				return err
			}
			defer reg.Close()
			start := time.Now()
			b, err := e.CreateBranch(cmd.Context(), args[0], from)
			if err != nil {
				return err
			}
			fmt.Fprintf(cmd.OutOrStdout(), "branch %q ready in %s (port %d)\n", b.Name, time.Since(start).Round(time.Millisecond), b.Port)
			return nil
		},
	}
	cmd.Flags().StringVar(&from, "from", "", "source to branch from")
	cmd.MarkFlagRequired("from")
	return cmd
}

func newBranchLsCmd() *cobra.Command {
	return &cobra.Command{
		Use: "ls", Short: "List branches",
		RunE: func(cmd *cobra.Command, args []string) error {
			_, reg, err := open()
			if err != nil {
				return err
			}
			defer reg.Close()
			branches, err := reg.ListLiveBranches()
			if err != nil {
				return err
			}
			w := tabwriter.NewWriter(cmd.OutOrStdout(), 2, 4, 2, ' ', 0)
			fmt.Fprintln(w, "NAME\tSTATE\tPORT\tCREATED")
			for _, b := range branches {
				fmt.Fprintf(w, "%s\t%s\t%d\t%s\n", b.Name, b.State, b.Port, b.CreatedAt)
			}
			return w.Flush()
		},
	}
}

func newBranchDestroyCmd() *cobra.Command {
	return &cobra.Command{
		Use:   "destroy NAME",
		Short: "Destroy a branch (container + CoW layer)",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			e, reg, err := open()
			if err != nil {
				return err
			}
			defer reg.Close()
			if err := e.DestroyBranch(cmd.Context(), args[0]); err != nil {
				return err
			}
			fmt.Fprintf(cmd.OutOrStdout(), "branch %q destroyed\n", args[0])
			return nil
		},
	}
}

func newConnectCmd() *cobra.Command {
	return &cobra.Command{
		Use:   "connect NAME",
		Short: "Print the connection string for a branch",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			_, reg, err := open()
			if err != nil {
				return err
			}
			defer reg.Close()
			b, err := reg.GetBranchByName(args[0])
			if err != nil {
				return err
			}
			src, err := reg.GetSourceByName(args[0]) // wrong on purpose? no — look up via b.SourceID:
			_ = src
			s, err := reg.GetSourceByID(b.SourceID)
			if err != nil {
				return err
			}
			fmt.Fprintf(cmd.OutOrStdout(), "postgres://%s@localhost:%d/%s\n", s.ConnUser, b.Port, s.ConnDB)
			return nil
		},
	}
}
```

**Fix while implementing:** the `connect` command above must use only `GetSourceByID` (delete the stray `GetSourceByName(args[0])` lookup — it's shown crossed out to make the point: branch → source lookup goes through `b.SourceID`). Add `GetSourceByID` to the registry:

```go
func (r *Registry) GetSourceByID(id string) (*Source, error) {
	return scanSource(r.db.QueryRow(`SELECT `+sourceCols+` FROM sources WHERE id=?`, id))
}
```

`cmd/pgb/main.go`:
```go
package main

import (
	"os"

	"github.com/abd-ulbasit/pgbranch/internal/cli"
)

func main() {
	if err := cli.NewRootCmd().Execute(); err != nil {
		os.Exit(1)
	}
}
```

- [ ] **Step 4: Run `go test ./internal/cli/` and `make build` — PASS. Commit `feat(cli): pgb source/branch/connect commands`**

---

### Task 11: Manual smoke test + README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Manual smoke test on Colima**

```bash
colima status || colima start
docker run -d --name demo-src --network bridge -e POSTGRES_PASSWORD=secret postgres:17 \
  -c wal_level=replica -c max_wal_senders=4
docker exec demo-src sh -c 'until pg_isready -U postgres; do sleep 1; done'
docker exec demo-src psql -U postgres -c "CREATE TABLE t(i int); INSERT INTO t SELECT generate_series(1,100000);"
SRC_IP=$(docker inspect -f '{{.NetworkSettings.IPAddress}}' demo-src)

make build
PGPASSWORD=secret ./bin/pgb source add main --host "$SRC_IP" --user postgres
./bin/pgb branch create pr-1 --from main     # expect: ready in ~2-4s
./bin/pgb branch ls
psql "$(./bin/pgb connect pr-1)" -c "SELECT count(*) FROM t"   # expect 100000
./bin/pgb branch destroy pr-1
docker rm -f demo-src
```

Record the actual `branch create` timing for the README.

- [ ] **Step 2: Write `README.md`** — one-liner, the problem, quickstart (the smoke test above, cleaned up), architecture sketch (overlay-in-container diagram from the spec), scope boundary ("dev/test tool, not HA"), Phase roadmap, "why not Neon/DBLab" comparison table.

- [ ] **Step 3: Run full suites: `make test` then `make it` — all PASS. Commit `docs: README with quickstart and architecture`**

---

## Self-review notes

- **Spec coverage:** P1 spec items all mapped: OverlayFS backend (Task 5+6, in-container assembly), Docker driver (6), SQLite registry + state machine (3+4), pg_basebackup seeding (7), saga + reconcile (8), CLI (10), correctness-by-test (9). ZFS/copy backends, pgproxy, TTL, masking, K8s = later phases per spec.
- **Known risk concentrations:** Task 9 (overlay mount inside container on Colima) is the highest-risk step; Task 6's Docker SDK type names drift between versions. Both flagged inline.
- **Type consistency:** `runtime.Driver` interface (Task 6) matches the fake in Task 8 method-for-method; `cow.PlanBranch` lower path updated to `/pgbranch/lower0/data` in Task 7's note — apply that change when doing Task 7.
