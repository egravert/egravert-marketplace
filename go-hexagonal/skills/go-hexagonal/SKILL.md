---
name: Go Layered Architecture
description: Use this skill when working on Go projects, creating new Go services, adding features to Go applications, designing Go package structure, implementing repository patterns, structuring layered Go architectures, or discussing Go application design. Applies to any Go development following a pragmatic layered architecture with clean separation between domain, service, storage, and interface layers.
version: 1.2.0
---

# Go Layered Architecture

Opinionated, pragmatic guide for building Go applications using a layered architecture inspired by hexagonal (ports and adapters) principles. This is not a pure hexagonal implementation — it intentionally flattens the ports-and-adapters model into a straightforward layered structure where dependencies flow inward toward the domain core. The goal is clarity and navigability over architectural purity: each layer has clear responsibilities and boundaries without the extra indirection that strict hexagonal architecture introduces.

## Dependency Flow

```
Interface (CLI/TUI/Web) → Service → Domain ← Storage
```

- **Domain** defines interfaces (ports). Storage implements them (adapters).
- Services depend on domain interfaces, never on concrete storage.
- Interface layers depend on services, never on storage or domain internals.

## Directory Structure

```
project/
├── cmd/
│   ├── cli/          # urfave/cli entry point
│   ├── tui/          # bubbletea entry point
│   └── web/          # chi router entry point
├── internal/
│   ├── domain/       # Entities, value objects, repository interfaces
│   ├── service/      # Business logic, validation, orchestration
│   ├── storage/      # Repository implementations (wraps sqlc)
│   │   └── db/       # sqlc-generated code (do not edit)
│   ├── cli/          # CLI command handlers
│   ├── tui/          # Bubbletea models and views
│   └── web/          # Chi handlers, middleware, templates
├── db/
│   ├── migrations/   # goose SQL migrations
│   └── queries/      # sqlc SQL query files
├── sqlc.yml
└── go.mod
```

## Domain Layer

The domain layer is the innermost ring. It has **zero external dependencies** — no framework imports, no struct tags, no database concerns. Include a `doc.go` with a glossary of key business concepts so that new contributors (human or agent) can understand the domain language.

### Entities

Pure Go structs representing business concepts:

```go
type Boardgame struct {
    ID          int64
    Name        string
    Players     Optional[Range]
    Complexity  Optional[uint8]
    Description string
}
```

- No `json`, `db`, or `yaml` struct tags
- Use `Optional[T]` for nullable fields instead of pointers or sql.Null types
- Fields use domain-appropriate types, not database types

### Value Objects

Enforce invariants through construction:

```go
type Range struct {
    min uint16  // private fields
    max uint16
}

func Between(min, max uint16) (Range, error) {
    if min == 0 || max == 0 {
        return Range{}, errors.New("min and max must be greater than zero")
    }
    if max < min {
        return Range{}, errors.New("max must be >= min")
    }
    return Range{min: min, max: max}, nil
}
```

- Private fields enforce invariants — values are always valid after construction
- Accessor methods expose data: `Min()`, `Max()`, `IsSingle()`
- Constructor functions return `(T, error)` when validation is needed

### Optional Type

Generic wrapper replacing `*T` and `sql.NullXxx` in the domain:

```go
type Optional[T any] struct {
    value T
    ok    bool
}

func Some[T any](v T) Optional[T] { return Optional[T]{value: v, ok: true} }
func None[T any]() Optional[T]    { return Optional[T]{} }
func (o Optional[T]) IsSet() bool { return o.ok }
func (o Optional[T]) Value() T    { return o.value }
func (o Optional[T]) OrElse(v T) T { if o.ok { return o.value }; return v }
```

### Repository Interfaces

Defined in domain — pure interfaces with no generation directives:

```go
type BoardgameRepository interface {
    Create(ctx context.Context, game Boardgame) (Boardgame, error)
    GetByID(ctx context.Context, id int64) (Boardgame, error)
    List(ctx context.Context) ([]Boardgame, error)
}
```

- `Create` returns the entity with server-generated fields (ID) populated
- Keep interfaces narrow (2-4 methods) — this makes hand-written test fakes trivial
- Test doubles are hand-written inline in test files, not generated (see `references/testing.md`)

## Service Layer

Services contain business logic, validation, and multi-repository orchestration. They depend only on domain interfaces.

```go
type BoardGameService struct {
    logger *slog.Logger
    repo   domain.BoardgameRepository
}

func NewBoardGameService(repo domain.BoardgameRepository, logger *slog.Logger) *BoardGameService {
    return &BoardGameService{repo: repo, logger: logger}
}
```

### Responsibilities

1. **Input validation** — verify business rules before delegating to storage
2. **Cross-repository coordination** — look up related entities, enforce consistency
3. **Error context** — return meaningful errors, not raw database errors
4. **Logging** — structured logging with slog for observability

### What services must NOT do

- Import any framework package (urfave, chi, bubbletea)
- Know about HTTP, CLI flags, or UI concerns
- Import sqlc-generated types or database/sql
- Handle serialization (JSON, HTML, etc.)

## Storage Layer

Implements domain repository interfaces by wrapping sqlc-generated code. This is the **only layer** that knows about database types.

### Translation Pattern

Every repository method translates between `domain.Entity` and `db.Entity`:

```go
func (r *BoardgameRepository) GetByID(ctx context.Context, id int64) (domain.Boardgame, error) {
    result, err := r.queries.GetBoardgameByID(ctx, id)
    if err != nil {
        return domain.Boardgame{}, err
    }
    return gameFromResult(result), nil
}
```

Helper functions handle type mapping:
- `sql.NullInt64` ↔ `domain.Optional[T]`
- `sql.NullString` ↔ `domain.Optional[string]`
- Paired nullable columns ↔ `domain.Optional[domain.Range]`

See `references/database.md` for complete sqlc workflow and translation patterns.

## Interface Layer

Each interface (CLI, TUI, Web) is a thin adapter. It parses input, calls services, and formats output. It never contains business logic.

### Interface Segregation

Handlers define **narrow interfaces** matching only what they need:

```go
// In web handler — NOT the full service interface
type BoardGameService interface {
    ListGames(context.Context) ([]domain.Boardgame, error)
    FindGame(context.Context, int64) (domain.Boardgame, error)
    AddGame(ctx context.Context, bg domain.Boardgame) (domain.Boardgame, err error)
}
```

This enables focused testing — fakes only need to implement the methods the handler actually uses.

### Error Handling by Layer

| Layer | Pattern |
|-------|---------|
| Domain | Return `error` from constructors when invariants fail |
| Service | Validate inputs, wrap errors with identifying context: `fmt.Errorf("adding session for game %d: %w", gameID, err)` |
| Storage | Wrap database errors with operation and entity context: `fmt.Errorf("getting game %d: %w", id, err)` |
| Web | Map errors to HTTP status codes (400/404/500), log with `slog.Error` |
| CLI | Map errors to user-facing messages and exit codes |

## Go Conventions

Standards that apply across all layers for consistency and correctness.

### Comments

Write comments that explain **why**, not **what**. Never describe what code does — agents and humans can read the code. Instead, document:

- Why this approach was chosen over alternatives
- What constraints or invariants the code operates under
- What assumptions are safe to make
- Intentional duplication (explain why, or agents will refactor it into a shared abstraction)

```go
// Bad — describes what the code does
// GetOrder retrieves an order by ID from the database

// Good — explains the design decision
// GetOrder retrieves an order and its full item tree. Items are denormalized
// at placement time because menu item IDs are unstable across published
// versions — we cannot look them up later.
```

### Package Documentation

Every package should have a `doc.go` explaining its purpose, key concepts, and important design decisions. Domain packages should include a glossary of business terms:

```go
// Package domain defines the core business entities for a boardgame tracker.
//
// Glossary:
//   - Boardgame: a tabletop game with metadata (player count, complexity)
//   - Session: a recorded play of a boardgame with players and date
//   - Play: alias for Session in user-facing contexts
package domain
```

### Error Handling

Always wrap errors with enough context to trace causation — include relevant IDs and values, not just the operation name:

```go
// Good — an agent or operator can trace this immediately
return domain.Session{}, fmt.Errorf(
    "creating session for game %d with %d players: %w", gameID, len(players), err)

// Good — includes the conflicting value
return domain.Boardgame{}, fmt.Errorf(
    "game name %q already exists: %w", game.Name, err)

// Bad — bare error with no context for debugging
return domain.Review{}, err

// Bad — operation name only, no identifying information
return domain.Review{}, fmt.Errorf("adding review: %w", err)
```

Each layer adds its own context. A fully wrapped error reads as a chain of operations:

```
creating session for game 42 with 3 players: inserting session: UNIQUE constraint failed
└── service                                  └── repository       └── database
```

### Greppability

Use unique, specific names that can be found with a single search. Prefer `GameStatusUnplayed` over `Unplayed`, `OrderMatchScore` over `Score`. If a function is called, there must be a greppable call site — avoid string-based dispatch and dynamic method resolution.

### Slice Return Conventions

**Known size:** Use `make([]T, len(results))` with index assignment — avoids repeated grow/copy from append:

```go
games := make([]domain.Boardgame, len(results))
for i, r := range results {
    games[i] = gameFromResult(r)
}
return games, nil
```

**Unknown size:** Use `make([]T, 0)` with append — signals intent to build up incrementally:

```go
reviews := make([]domain.Review, 0)
for _, r := range results {
    if r.Rating > 3 {
        reviews = append(reviews, reviewFromResult(r))
    }
}
return reviews, nil
```

**On error:** Return `nil` for the slice — signals the result is meaningless and should not be used:

```go
if err != nil {
    return nil, fmt.Errorf("listing reviews for game %d: %w", gameID, err)
}
```

| Scenario | Slice value | Error value |
|----------|------------|-------------|
| Success, known size | `make([]T, len(n))` | `nil` |
| Success, unknown size | `make([]T, 0)` + append | `nil` |
| Error | `nil` | `fmt.Errorf("context: %w", err)` |
| Struct + error | zero value `T{}` | `fmt.Errorf("context: %w", err)` |

## Configuration

Use **koanf** with layered precedence (lowest to highest):

1. Hardcoded defaults
2. Config file (`~/.config/<app>/config.yaml`)
3. Environment variables (`APPNAME_*`)
4. CLI flags (CLI interface only)

```go
k := koanf.New(".")
k.Load(confmap.Provider(defaults, "."), nil)
k.Load(env.Provider("ROLLCALL_", ".", transformKey), nil)

return Config{Addr: k.String("addr"), DBPath: k.String("db_path")}
```

**Critical rule:** Load config in `cmd/*/main.go`, extract into a plain struct, pass values to internal layers. Never pass the koanf instance deeper than main.

## Logging

Use **slog** (standard library) throughout:

- Create logger in `cmd/*/main.go`
- Inject via constructor: `NewService(repo, logger)`
- Use structured key-value pairs: `slog.Warn("parse failed", "id", id, "error", err)`
- Configure format (text vs JSON) per interface

## Adding a New Feature

1. Define domain entities in `internal/domain/` (pure Go structs)
2. Create migration in `db/migrations/`
3. Write SQL queries in `db/queries/` with sqlc annotations
4. Add repository interface to `internal/domain/`
5. Run `go generate ./...` (generates sqlc code)
6. Implement repository in `internal/storage/` (wrap sqlc, translate types)
7. Write repository integration tests with `:memory:` SQLite
8. Implement business logic in `internal/service/`
9. Add handlers in `internal/cli/`, `internal/tui/`, or `internal/web/`
10. Define narrow service interface at point of consumption (handler or handler test file)
11. Write handler tests with hand-written inline fakes for the service interface
12. Write service unit tests only if business logic is complex enough to warrant isolation from handler tests
13. Wire dependencies in `cmd/*/main.go`

## Code Generation

```bash
go generate ./...    # Runs sqlc
sqlc generate        # SQL queries only
```

All generated code lives in `internal/storage/db/` and must never be manually edited.

## Reference Files

- **`references/packages.md`** — Detailed usage patterns for each package (urfave/cli, chi, bubbletea, koanf, slog, templ, HTMX)
- **`references/testing.md`** — Testing strategy, hand-written fakes, testify usage, handler-first test structure
- **`references/database.md`** — sqlc workflow, goose migrations, repository translation patterns, SQL conventions

Consult reference files for package-specific syntax and detailed examples.
