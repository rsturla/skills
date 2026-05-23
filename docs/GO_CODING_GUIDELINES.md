# Go Coding Guidelines

## Formatting & Tooling

- `gofmt` is non-negotiable — all code must be formatted
- Use `go vet` and `staticcheck` for linting
- Run `go mod tidy` before committing

## Naming

- Follow [Effective Go](https://go.dev/doc/effective_go) naming conventions
- Unexported by default — only export what the package API requires
- Interfaces: single-method interfaces named by the method + `er` suffix (`Reader`, `Writer`, `Closer`)
- Avoid stuttering: `http.Client` not `http.HTTPClient`
- Acronyms are all-caps: `HTTPServer`, `XMLParser`, `ID` not `Id`
- Package names: short, lowercase, singular (`user` not `users`)

## Error Handling

- Always handle errors — never use `_` for error returns
- Wrap errors with context: `fmt.Errorf("loading config: %w", err)`
- Use `errors.Is` and `errors.As` for comparison, not string matching
- Sentinel errors as package-level vars: `var ErrNotFound = errors.New("not found")`
- Return early on error — avoid deep nesting

## Package Layout

```text
cmd/
  myapp/
    main.go             # minimal — wires dependencies, calls run()
internal/
  server/               # HTTP/gRPC server setup
  store/                # data access
  domain/               # core types and business logic
pkg/                    # only if genuinely reusable outside this repo
```

- `internal/` for everything that isn't a public API
- `cmd/` entry points should be thin — parse flags, build dependencies, call `run()`
- Avoid `utils/`, `helpers/`, `common/` packages — find a real name

## Registry Pattern (init)

Use `init()` for self-registering implementations — keeps the wiring declarative and avoids manual import lists.

```go
// internal/store/postgres/postgres.go
package postgres

import "myapp/internal/store"

func init() {
    store.Register("postgres", New)
}

func New(dsn string) (store.Store, error) { ... }
```

```go
// internal/store/registry.go
package store

var registry = map[string]Factory{}

type Factory func(dsn string) (Store, error)

func Register(name string, f Factory) {
    registry[name] = f
}

func New(name, dsn string) (Store, error) {
    f, ok := registry[name]
    if !ok {
        return nil, fmt.Errorf("unknown store: %s", name)
    }
    return f(dsn)
}
```

```go
// cmd/myapp/main.go
import (
    _ "myapp/internal/store/postgres"  // register via init()
    _ "myapp/internal/store/sqlite"
)
```

## Dependencies

- Minimal third-party dependencies — prefer stdlib
- Justify every external dependency: does stdlib really not cover this?
- Pin dependencies with exact versions in `go.mod`

### Recommended Libraries

Prefer stdlib. When stdlib doesn't cover it, use these:

| Category | Package | Notes |
| -------- | ------- | ----- |
| HTTP server/router | `net/http` (stdlib) | Enhanced pattern matching in Go 1.22+ |
| Logging | `log/slog` (stdlib) | Default for all projects. JSON in prod, text in dev. |
| YAML | `go.yaml.in/yaml/v4` | `gopkg.in/yaml.v3` is deprecated. v4 for new projects. |
| JSON | `encoding/json` (stdlib) | Use `json/v2` when it lands in stdlib |
| SQL | `database/sql` (stdlib) | With driver packages (`pgx`, `go-sqlite3`) |
| CLI (subcommands) | `github.com/spf13/cobra` | Only when you need subcommands. Use stdlib `flag` for single-command tools. |
| Errors | `fmt.Errorf` with `%w` (stdlib) | No error libraries needed |
| Config (multi-source) | `github.com/knadh/koanf` | Only when merging env + file + defaults. For single YAML file, just unmarshal directly. |

### Avoid

| Package | Why | Replacement |
| ------- | --- | ----------- |
| `gopkg.in/yaml.v3` | Deprecated (April 2025) | `go.yaml.in/yaml/v4` |
| `gopkg.in/yaml.v2` | Deprecated | `go.yaml.in/yaml/v4` |
| `logrus` | Maintenance mode, no new features | `log/slog` |
| `gin` / `echo` / `fiber` | Usually unnecessary — stdlib router is sufficient | `net/http` |

## Testing

- Table-driven tests as default pattern
- Test files alongside source: `foo.go` / `foo_test.go`
- Use `testdata/` for fixtures
- `t.Helper()` in test helper functions
- Subtests with `t.Run("descriptive name", ...)`
- No test frameworks — stdlib `testing` package is sufficient
- Use `t.Parallel()` where safe
- No benchmarks — focus on correctness, not micro-optimization

### Unit Tests

- Test one function/method per test function
- Mock external dependencies via interfaces
- Fast — no network, no disk, no database

### Integration Tests

- Use build tag `//go:build integration`
- Run separately: `go test -tags=integration ./...`
- OK to hit real databases, APIs, filesystems
- Use `t.Cleanup()` for teardown

### Fuzz Testing

- Write fuzz tests for parsers, validators, serializers, and any function that handles untrusted input
- Naming: `FuzzParseConfig`, `FuzzDecodeToken`
- Seed corpus in `testdata/fuzz/`
- Run: `go test -fuzz=FuzzParseConfig -fuzztime=30s`

## Concurrency

- Prefer channels for coordination, mutexes for state protection
- Never start goroutines without a way to stop them — use `context.Context`
- `errgroup` for fan-out/fan-in patterns
- Document goroutine ownership: who starts it, who stops it

## Interfaces

Interfaces are a primary extensibility mechanism. Use them for pluggable backends, not just testing.

- Accept interfaces, return structs
- Define interfaces at the consumer, not the provider
- Keep interfaces small — 1-3 methods (single-method preferred)
- Extract interfaces at every infrastructure boundary — storage, messaging, external APIs, clocks, filesystems

### Design for Pluggability

The same interface should support production backends, local development, and testing — all as first-class
implementations, not afterthoughts.

```go
// internal/storage/storage.go — defined by the consumer
type BlobStore interface {
    Put(ctx context.Context, key string, data io.Reader) error
    Get(ctx context.Context, key string) (io.ReadCloser, error)
    Delete(ctx context.Context, key string) error
}

// internal/storage/s3/s3.go — production
type S3Store struct { client *s3.Client; bucket string }

// internal/storage/pvc/pvc.go — on-prem / PVC
type PVCStore struct { basePath string }

// internal/storage/memory/memory.go — dev / testing
type MemoryStore struct { data map[string][]byte }
```

```go
// internal/db/db.go
type Repository interface {
    FindByID(ctx context.Context, id string) (Order, error)
    Save(ctx context.Context, order Order) error
}

// internal/db/postgres/postgres.go — production
// internal/db/sqlite/sqlite.go — local dev, CI
// internal/db/memory/memory.go — unit tests
```

Each implementation is a real, standalone package — not a mock. Tests use the in-memory backend. Local dev uses SQLite
or filesystem. Production uses Postgres or S3. Same interface, same service code.

### Composition via Embedding

Compose larger interfaces from smaller ones when needed:

```go
type Reader interface { Read(ctx context.Context, id string) (Item, error) }
type Writer interface { Write(ctx context.Context, item Item) error }
type ReadWriter interface {
    Reader
    Writer
}
```

### Wire via Constructor

Pass interfaces through constructors — makes dependencies explicit and testable:

```go
func NewService(store Store, clock Clock) *Service {
    return &Service{store: store, clock: clock}
}
```

### Functional Options for Optional Deps

Use functional options when some interface deps are optional:

```go
type Option func(*Service)

func WithLogger(l Logger) Option {
    return func(s *Service) { s.logger = l }
}

func NewService(store Store, opts ...Option) *Service {
    s := &Service{store: store, logger: nopLogger{}}
    for _, o := range opts { o(s) }
    return s
}
```

## Comments

- Package comments on every exported package
- Exported symbols get doc comments starting with the symbol name
- No comments on unexported code unless the why is non-obvious
