# Beads Architecture — Developer Reference

> This is a developer-oriented architectural map for navigating and understanding the beads codebase. For user-facing architecture, see [docs/ARCHITECTURE.md](../../docs/ARCHITECTURE.md). For internal concurrency details, see [docs/INTERNALS.md](../../docs/INTERNALS.md).

## System Overview

**beads** (`bd`) is a distributed, Dolt-powered issue tracker designed for AI-supervised coding workflows. Key design goals:

- **Distributed-first** — Collision-free hash-based IDs; no central coordination needed
- **Offline-capable** — Local Dolt database works without network; sync when ready
- **AI-optimized** — JSON output, `bd ready` for unblocked work, dependency graphs
- **ACID transactions** — Atomic multi-operation sequences via Dolt
- **Audit-everything** — Every mutation tracked with actor and timestamp
- **Zero-config** — Auto-discover `.beads/` directory, auto-migrate schemas

## Package Map

### Entrypoints

| Package | Purpose |
|---------|---------|
| `cmd/bd/` | CLI entrypoint — Cobra commands, lifecycle hooks, signal handling |
| `beads.go` | Public Go API — minimal surface for programmatic extensions |

### Internal Packages by Layer

**Domain Model:**

| Package | Purpose |
|---------|---------|
| `internal/types/` | Core data types: Issue, Dependency, Comment, Event, Label, Status enums |
| `internal/validation/` | Input validation rules for issues, dependencies, configs |

**Storage:**

| Package | Purpose |
|---------|---------|
| `internal/storage/` | Storage interface definitions and composition (Storage, DoltStorage, Transaction) |
| `internal/storage/dolt/` | Dolt SQL server backend — primary implementation |
| `internal/storage/embeddeddolt/` | Embedded (in-process) Dolt mode — no external server |
| `internal/storage/schema/` | Database schema definitions, migrations, table creation |
| `internal/storage/doltutil/` | Dolt connection utilities, query helpers |
| `internal/storage/versioncontrolops/` | Git-like operations: commit, push, pull, branch |
| `internal/storage/issueops/` | High-level issue operations composing lower-level storage calls |

**Configuration & Discovery:**

| Package | Purpose |
|---------|---------|
| `internal/config/` | Multi-level config loading via Viper (yaml files, env vars, defaults) |
| `internal/configfile/` | Config file read/write operations |
| `internal/beads/` | Workspace discovery — find `.beads/` dir, database path, redirects |

**Query & Filtering:**

| Package | Purpose |
|---------|---------|
| `internal/query/` | Custom query DSL — lexer, parser, evaluator for `--filter` expressions |

**Subsystems:**

| Package | Purpose |
|---------|---------|
| `internal/idgen/` | Hash-based collision-free ID generation |
| `internal/compact/` | Issue compaction — semantic memory decay for old closed issues |
| `internal/molecules/` | Issue templates — hierarchical loading, wisp lifecycle |
| `internal/hooks/` | Event hooks — `.beads/hooks/on_create`, `on_update`, `on_close` |
| `internal/routing/` | Multi-repo routing — contributor namespace isolation |
| `internal/audit/` | Audit trail recording for all mutations |
| `internal/git/` | Git integration — repo discovery, identity, hooks installation |
| `internal/telemetry/` | OpenTelemetry traces and metrics |
| `internal/debug/` | Debug logging utilities |
| `internal/lockfile/` | File locking for single-writer enforcement (embedded mode) |
| `internal/remotecache/` | Cache layer for remote operations |
| `internal/timeparsing/` | Natural language time parsing |
| `internal/templates/` | Output templates for CLI rendering |
| `internal/ui/` | TUI components (Charm/lipgloss) |
| `internal/utils/` | Shared utility functions |
| `internal/testutil/` | Test helpers and fixtures |

**External Integrations:**

| Package | Purpose |
|---------|---------|
| `internal/github/` | GitHub issue sync |
| `internal/gitlab/` | GitLab issue sync |
| `internal/jira/` | Jira issue sync |
| `internal/linear/` | Linear issue sync |
| `internal/notion/` | Notion integration |
| `internal/ado/` | Azure DevOps integration |
| `internal/tracker/` | Generic external tracker abstraction |
| `internal/formula/` | Homebrew formula generation |
| `internal/recipes/` | Built-in workflow recipes |
| `internal/doltserver/` | Dolt server lifecycle management |

### Integration Layer

| Directory | Purpose |
|-----------|---------|
| `integrations/beads-mcp/` | MCP server (Python) — exposes bd operations to Claude and other AI assistants |
| `integrations/claude-code/` | Claude Code IDE integration |
| `format/` | Output formatting — JSON serialization, human-readable display |

## Layered Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                         CLI Layer                              │
│  cmd/bd/ — Cobra commands, flags, signal handling              │
│  All commands support --json for programmatic use              │
└──────────────────────────┬─────────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────────┐
│                    Business Logic                              │
│  internal/storage/issueops/  — composed issue operations       │
│  internal/query/             — filter DSL evaluation           │
│  internal/compact/           — compaction engine               │
│  internal/molecules/         — template instantiation          │
│  internal/hooks/             — event hook dispatch             │
└──────────────────────────┬─────────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────────┐
│                    Storage Interface                           │
│  internal/storage/storage.go — Storage, DoltStorage,           │
│                                Transaction interfaces          │
└──────┬───────────────────────────────────────────┬─────────────┘
       │                                           │
┌──────▼──────────────────┐  ┌─────────────────────▼─────────────┐
│   Embedded Dolt Mode    │  │        Server Mode                │
│  internal/storage/      │  │  internal/storage/dolt/           │
│    embeddeddolt/        │  │  Connects via MySQL protocol      │
│  In-process, single-    │  │  to dolt sql-server               │
│  writer, file locking   │  │  Multi-writer, concurrent         │
└──────┬──────────────────┘  └─────────────────────┬─────────────┘
       │                                           │
       └────────────────────┬──────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────┐
│                      Dolt Database                             │
│  .beads/embeddeddolt/ or .beads/dolt/                          │
│  Version-controlled SQL — cell-level merge, branching          │
└───────────────────────────┬────────────────────────────────────┘
                            │  bd dolt push / pull
┌───────────────────────────▼────────────────────────────────────┐
│                      Remote Storage                            │
│  DoltHub, S3, GCS, filesystem backup                           │
└────────────────────────────────────────────────────────────────┘
```

## Command Lifecycle

Every `bd` command follows this lifecycle managed by Cobra's hooks in `cmd/bd/main.go`:

### Phase 1: PersistentPreRun

1. **Config initialization** — `config.Initialize()` loads merged yaml configs
2. **No-DB bypass check** — Commands like `version`, `help` skip DB entirely
3. **Workspace discovery** — Walk up from CWD to find `.beads/` directory
4. **Store creation** — Open DoltStore (embedded or server mode based on metadata.json)
5. **Schema migration** — Auto-upgrade database schema if version mismatch detected
6. **Read-only optimization** — Read commands (`list`, `ready`, `show`) open store in RO mode
7. **Signal handler setup** — SIGTERM/SIGHUP wired for graceful shutdown with final Dolt commit
8. **Hook runner initialization** — Load hook scripts from `.beads/hooks/`
9. **Telemetry setup** — Initialize OpenTelemetry spans if configured
10. **Molecule loading** — Load template molecules from built-in + user + project paths

### Phase 2: RunE

- Execute the command's business logic using the initialized store
- All write operations mark the store as dirty for auto-commit
- Commands produce output (human-readable or `--json`)

### Phase 3: PersistentPostRun

1. **FlushManager processing** — Debounced Dolt commit if store is dirty (embedded mode only)
2. **Auto-push** — Push to remote if `dolt.auto-push` is configured
3. **Store close** — Release connections, file locks
4. **Telemetry flush** — Export accumulated spans/metrics
5. **Cleanup** — Cancel root context, release resources

### Read-Only Commands

These commands open the store in read-only mode for performance:
`list`, `ready`, `show`, `stats`, `blocked`, `count`, `search`, `graph`, `duplicates`, `comments`, `backup`, `export`

### No-DB Commands

These commands skip database initialization entirely:
`version`, `help`, `completion`, `config` (some subcommands)

## Interface Composition

The storage layer uses interface composition to separate capabilities:

```go
// Core interface — all storage implementations satisfy this
type Storage interface {
    CreateIssue(ctx, issue, actor) error
    GetIssue(ctx, id) (*Issue, error)
    UpdateIssue(ctx, id, updates, actor) error
    SearchIssues(ctx, query, filter) ([]*Issue, error)
    AddDependency(ctx, dep, actor) error
    GetReadyWork(ctx, filter) ([]*Issue, error)
    GetBlockedIssues(ctx, filter) ([]*BlockedIssue, error)
    RunInTransaction(ctx, commitMsg, fn) error
    SetConfig(ctx, key, value) error
    Close() error
    // ... ~40 methods total
}

// Full Dolt capability — composes all sub-interfaces
type DoltStorage interface {
    Storage
    VersionControl       // branch, checkout, merge, commit
    HistoryViewer        // log, diff, blame
    RemoteStore          // push, pull, remote management
    SyncStore            // high-level sync with peers
    FederationStore      // multi-repo federation
    BulkIssueStore       // batch import/export
    DependencyQueryStore // advanced dependency queries
    AnnotationStore      // issue annotations
    ConfigMetadataStore  // config metadata CRUD
    CompactionStore      // issue compaction
    AdvancedQueryStore   // complex queries
}
```

### Optional Capabilities via Type Assertion

Some capabilities are exposed via type assertion rather than the main interface:

```go
// Raw database access (diagnostics, migrations)
type RawDBAccessor interface { DB() *sql.DB; UnderlyingDB() *sql.DB }

// Filesystem location
type StoreLocator interface { Path() string; CLIDir() string }

// Maintenance operations
type GarbageCollector interface { DoltGC(ctx) error }
type Flattener interface { Flatten(ctx) error }
type Compactor interface { Compact(ctx, ...) error }

// Lifecycle inspection
type LifecycleManager interface { IsClosed() bool }

// Auto-commit support
type PendingCommitter interface { CommitPending(ctx, actor) (bool, error) }

// Disaster recovery
type BackupStore interface { BackupDatabase(ctx, dir) error; RestoreDatabase(ctx, dir, force) error }
```

### Transaction Interface

Transactions provide atomic multi-operation support:

```go
store.RunInTransaction(ctx, "create parent+child", func(tx Transaction) error {
    tx.CreateIssue(ctx, parent, actor)
    tx.AddDependency(ctx, dep, actor)
    tx.CreateIssue(ctx, child, actor)
    return nil  // commit on success, rollback on error
})
```

## Storage Backends

### Embedded Mode (default)

- Dolt runs in-process via `dolthub/driver`
- Data in `.beads/embeddeddolt/`
- Single-writer enforced via file locking (`internal/lockfile/`)
- FlushManager handles debounced auto-commit (see [docs/INTERNALS.md](../../docs/INTERNALS.md))
- No external server process needed

### Server Mode

- Connects to external `dolt sql-server` via MySQL protocol
- Data in `.beads/dolt/`
- Multi-writer capable with cell-level merge
- Connection managed via circuit breaker for resilience
- PID file at `.beads/dolt-server.pid`
- Auto-starts server if not running, reference-counted for test lifecycle

### Mode Selection

Mode is determined by `.beads/metadata.json`:
- `dolt_mode: "embedded"` → in-process
- `dolt_mode: "server"` → external server

## Dependency Graph Engine

### Dependency Types

| Type | Semantic | Affects `bd ready`? |
|------|----------|---------------------|
| `blocks` | X must close before Y starts | Yes |
| `parent-child` | Hierarchical decomposition | Yes (transitive) |
| `related` | Soft reference link | No |
| `discovered-from` | Origin tracking | No |
| `conditional-blocks` | Y runs only if X fails | Yes (conditional) |

### Blocked Cache

Performance-critical materialized cache for `bd ready`:

- `blocked_issues_cache` table stores all currently-blocked issue IDs
- Rebuilt (DELETE + INSERT via recursive CTE) on every dependency/status change
- Queries use `NOT EXISTS` against cache — 29ms vs 752ms with live CTE
- Invalidated only by `blocks`/`parent-child` changes, not `related`/`discovered-from`
- Full rebuild is <50ms even on 10K issues

See [docs/INTERNALS.md](../../docs/INTERNALS.md#blocked-issues-cache-bd-5qim) for implementation details.

### Cycle Detection

Circular dependencies are detected and prevented at insertion time. Dependency tree traversal is depth-limited (max 50) for safety.

## ID Generation

Hash-based collision-free IDs enable distributed operation without coordination:

1. Generate random UUID
2. SHA256 hash → truncate to short prefix (4-6 chars)
3. Result: `bd-a1b2` format
4. Progressive scaling: IDs grow from 4 → 5 → 6 chars as database grows
5. Content hash (SHA256 of issue fields) enables deduplication during merge

Collision probability < 1% at realistic scales. See [docs/COLLISION_MATH.md](../../docs/COLLISION_MATH.md).

## Configuration Hierarchy

Config is loaded by `internal/config/config.go` using Viper with multi-file merging.

**Precedence (highest → lowest):**

1. **CLI flags** — `--actor`, `--db`, `--readonly`, etc. (bound to Viper via Cobra)
2. **Environment variables** — `BD_*` prefix (via `v.AutomaticEnv()`), plus specific bindings like `BEADS_IDENTITY`, `BEADS_DIR`
3. **Merged config files** (each layer overrides the previous):
   - `$BEADS_DIR/config.yaml` (highest file priority)
   - Project `.beads/config.yaml` (walk up from CWD)
   - `~/.config/bd/config.yaml` (user-level)
   - `~/.beads/config.yaml` (legacy)
4. **Viper defaults** — hardcoded in `Initialize()`

**Key config keys:** `issue_prefix`, `dolt.auto-commit`, `dolt_mode`, `routing.mode`, `federation.remote`, `sync.require_confirmation_on_mass_delete`

Separate from Viper config: **DB-stored config** in the `config` table (issue_prefix, custom_statuses, custom_types) — managed via `bd config` commands and `Storage.SetConfig()`.

## Query DSL

The `internal/query/` package implements a custom filter language for `bd list --filter`:

```
bd list --filter "priority < 2 AND status != closed"
bd list --filter "assignee = alice OR label = urgent"
```

**Pipeline:** Input string → Lexer → Parser → AST → Evaluator

- **Simple queries** (AND-only, no OR): compiled to `IssueFilter` struct for SQL-level filtering
- **Complex queries** (with OR): compiled to Go predicate functions for in-memory evaluation
- **Supported operators:** `=`, `!=`, `<`, `>`, `<=`, `>=`, `AND`, `OR`, `NOT`
- **Queryable fields:** `status`, `priority`, `type`, `assignee`, `owner`, `label`, `created`, `updated`, etc.

## Schema & Migrations

Database schema lives in `internal/storage/schema/`:

**Core tables:**
- `issues` — work items with all fields
- `dependencies` — relationship graph (from_id, to_id, dep_type)
- `labels` — issue tagging
- `comments` — threaded discussion
- `events` — audit trail
- `config` — key-value configuration
- `custom_statuses`, `custom_types` — user-defined enums
- `wisps` — ephemeral/transient messages
- `blocked_issues_cache` — materialized blocking computation

**Dolt system tables:** `dolt_log`, `dolt_branches`, `dolt_status`, etc.

Schema migrations run automatically in `PersistentPreRun` when a version mismatch is detected.

## Extension Points

### Event Hooks

Scripts in `.beads/hooks/` executed asynchronously on mutations:
- `on_create` — fired after issue creation
- `on_update` — fired after issue update
- `on_close` — fired after issue closure
- 10-second timeout per hook; fire-and-forget

### Molecules (Templates)

Hierarchical template loading from multiple sources:
1. Built-in molecules (embedded in binary)
2. Town-level: `$GT_ROOT/.beads/molecules.jsonl`
3. User-level: `~/.beads/molecules.jsonl`
4. Project-level: `.beads/molecules.jsonl`

Templates can be instantiated as work items. Wisps (ephemeral child issues) track execution steps and are squashed into permanent digests.

### Public Go API

The root `beads.go` package exports:
- `Open()` / `OpenFromConfig()` — open a database
- `FindDatabasePath()` / `FindBeadsDir()` — workspace discovery
- Type aliases for all core types (Issue, Dependency, Status, etc.)
- Constant exports for Status, IssueType, DependencyType, EventType, SortPolicy

### MCP Server

`integrations/beads-mcp/` — Python MCP server exposing bd operations to AI assistants (Claude, etc.) via the Model Context Protocol.

## Architectural Invariants & Gotchas

1. **Embedded vs Server mode** — FlushManager only operates in embedded mode; server mode has its own commit coordination
2. **Read-only store opening** — Read commands skip write-path initialization for performance
3. **No-DB commands** — Some commands (`version`, `help`) bypass all database initialization
4. **Test database firewalls** — Pattern-based prefixes (`testdb_*`, `beads_test*`, etc.) prevent accidental production writes
5. **CGO requirement** — Dolt's embedded mode requires CGO; Windows ARM64 and Android use pure-Go regex fallback
6. **Single-writer locking** — Embedded mode uses file locks; only one `bd` process can write at a time
7. **Content hashing** — SHA256 of issue fields; used for dedup during merge, not for ID generation (IDs use random UUID hashing)
8. **Auto-commit policy** — Every write operation triggers a Dolt commit (configurable via `dolt.auto-commit`)
9. **Signal handling** — SIGTERM/SIGHUP trigger a final Dolt commit before exit to prevent data loss

## Build & Release

- **Build:** `make build` — CGO-enabled Go build
- **Test:** `make test` or `./scripts/test.sh` — standard test suite with skip list
- **Full CGO tests:** `make test-full-cgo` or `./scripts/test-cgo.sh`
- **Lint:** `golangci-lint run ./...`
- **Release:** GoReleaser with zig cross-compilation (Linux, macOS, Windows, FreeBSD, Android)
- **Parallelism:** `--parallelism 1` to avoid zig race conditions
- **Code signing:** Authenticode (Windows), codesign (macOS)
- **Distribution:** GitHub Releases, Homebrew, npm (`@beads/bd`), PyPI (`beads-mcp`)

## Testing Patterns

- **`t.TempDir()`** — All tests use temporary directories; never touch production DB
- **Database firewalls** — Test DB name prefixes prevent accidental production writes
- **Race detection** — `go test -race` for concurrency tests (FlushManager, store lifecycle)
- **Dolt availability** — Tests skip gracefully if Dolt is not installed
- **Test server lifecycle** — Reference-counted auto-start/stop of dolt sql-server for tests
- **Table-driven tests** — Standard Go pattern used throughout

## Related Documentation

| Document | Covers |
|----------|--------|
| [docs/ARCHITECTURE.md](../../docs/ARCHITECTURE.md) | User-facing architecture, two-layer model, write/read paths, collision math |
| [docs/INTERNALS.md](../../docs/INTERNALS.md) | FlushManager concurrency, blocked cache implementation |
| [docs/CONFIG.md](../../docs/CONFIG.md) | Configuration reference |
| [docs/DOLT.md](../../docs/DOLT.md) | Dolt backend details, migration between modes |
| [docs/DOLT-BACKEND.md](../../docs/DOLT-BACKEND.md) | Dolt backend internals |
| [docs/OBSERVABILITY.md](../../docs/OBSERVABILITY.md) | OpenTelemetry integration |
| [docs/TESTING.md](../../docs/TESTING.md) | Test infrastructure and patterns |
| [docs/MOLECULES.md](../../docs/MOLECULES.md) | Template system and wisp lifecycle |
| [docs/PLUGIN.md](../../docs/PLUGIN.md) | Plugin/extension architecture |
| [docs/DEPENDENCIES.md](../../docs/DEPENDENCIES.md) | Dependency semantics |
| [docs/COLLISION_MATH.md](../../docs/COLLISION_MATH.md) | Birthday paradox analysis for ID length |
| [docs/ROUTING.md](../../docs/ROUTING.md) | Multi-repo routing |
