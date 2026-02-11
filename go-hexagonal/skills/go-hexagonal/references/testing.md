# Testing Strategy

Three-tier testing approach aligned with the layered architecture. Each layer tests different concerns with different dependencies.

## Table of Contents

- [Test Tooling](#test-tooling)
- [Unit Tests — Service Layer](#unit-tests)
- [Integration Tests — Storage Layer](#integration-tests)
- [Interface Tests — CLI/TUI/Web Handlers](#interface-tests)
- [Code Generation](#code-generation)
- [Guidelines](#guidelines)

---

## Test Tooling

| Tool | Purpose |
|------|---------|
| `moq` | Generate test doubles from interfaces via `go generate` |
| `testify/assert` | Non-fatal assertions (test continues on failure) |
| `testify/require` | Fatal assertions (test stops immediately on failure) |
| `testify/suite` | Test suites with setup/teardown for integration tests |

Install moq:
```bash
go install github.com/matryer/moq@latest
```

---

## Unit Tests

**Location:** `internal/service/*_test.go`, `internal/domain/*_test.go`
**Dependencies:** moq-generated doubles only — no database, no frameworks
**Focus:** Business logic, validation, edge cases

### Pattern

```go
func TestBoardGameService_AddGame(t *testing.T) {
    repo := &domaintesting.BoardgameRepositoryMock{
        CreateFunc: func(ctx context.Context, game domain.Boardgame) (domain.Boardgame, error) {
            game.ID = 1
            return game, nil
        },
    }

    svc := service.NewBoardGameService(repo, slog.Default())

    game, err := svc.AddGame(context.Background(), domain.Boardgame{
        Name:       "Catan",
        Complexity: domain.Some(uint8(3)),
    })

    require.NoError(t, err)
    assert.Equal(t, "Catan", game.Name)
    assert.Equal(t, int64(1), game.ID)

    // Verify the mock was called correctly
    require.Len(t, repo.CreateCalls(), 1)
    assert.Equal(t, "Catan", repo.CreateCalls()[0].Game.Name)
}
```

### Testing Validation

```go
func TestBoardGameService_AddGame_EmptyName(t *testing.T) {
    repo := &domaintesting.BoardgameRepositoryMock{}

    svc := service.NewBoardGameService(repo, slog.Default())

    _, err := svc.AddGame(context.Background(), domain.Boardgame{Name: ""})

    assert.EqualError(t, err, "game name is required")
    assert.Empty(t, repo.CreateCalls())  // repo never called
}
```

### Table-Driven Tests

```go
func TestBoardGameService_AddGame_Validation(t *testing.T) {
    tests := []struct {
        name    string
        game    domain.Boardgame
        wantErr string
    }{
        {
            name:    "empty name",
            game:    domain.Boardgame{Name: ""},
            wantErr: "game name is required",
        },
        {
            name:    "complexity too high",
            game:    domain.Boardgame{Name: "Catan", Complexity: domain.Some(uint8(6))},
            wantErr: "complexity must be between 1 and 5",
        },
        {
            name:    "whitespace only name",
            game:    domain.Boardgame{Name: "   "},
            wantErr: "game name is required",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := &domaintesting.BoardgameRepositoryMock{}
            svc := service.NewBoardGameService(repo, slog.Default())

            _, err := svc.AddGame(context.Background(), tt.game)

            assert.EqualError(t, err, tt.wantErr)
            assert.Empty(t, repo.CreateCalls())
        })
    }
}
```

### Multi-Repository Coordination

```go
func TestSessionService_AddSession_GameNotFound(t *testing.T) {
    games := &domaintesting.BoardgameRepositoryMock{
        SearchGamesFunc: func(ctx context.Context, name string) ([]domain.Boardgame, error) {
            return []domain.Boardgame{}, nil  // no matches
        },
    }
    sessions := &domaintesting.SessionRepositoryMock{}

    svc := service.NewSessionService(sessions, games)

    _, err := svc.AddSession(context.Background(), "Nonexistent", []string{"Alice"}, time.Now())

    assert.ErrorContains(t, err, "no games found")
    assert.Empty(t, sessions.CreateCalls())  // session never created
}
```

---

## Integration Tests

**Location:** `internal/storage/*_test.go`
**Dependencies:** In-memory SQLite (`:memory:`) with goose migrations
**Focus:** SQL correctness, repository implementations, type translation

### Suite Pattern

```go
type BoardgameRepoSuite struct {
    suite.Suite
    db   *sql.DB
    repo *storage.BoardgameRepository
}

func (s *BoardgameRepoSuite) SetupTest() {
    db, err := sql.Open("sqlite3", ":memory:")
    s.Require().NoError(err)

    goose.SetDialect("sqlite3")
    err = goose.Up(db, "../../db/migrations")
    s.Require().NoError(err)

    s.db = db
    s.repo = storage.NewBoardgameRepository(dbq.New(db))
}

func (s *BoardgameRepoSuite) TearDownTest() {
    s.db.Close()
}

func (s *BoardgameRepoSuite) TestCreate() {
    game := domain.Boardgame{
        Name:    "Catan",
        Players: domain.Some(mustRange(3, 4)),
    }

    created, err := s.repo.Create(context.Background(), game)

    s.NoError(err)
    s.NotZero(created.ID)
    s.Equal("Catan", created.Name)
}

func (s *BoardgameRepoSuite) TestGetByID_NotFound() {
    _, err := s.repo.GetByID(context.Background(), 999)
    s.Error(err)
}

func TestBoardgameRepoSuite(t *testing.T) {
    suite.Run(t, new(BoardgameRepoSuite))
}
```

### What to Test

- CRUD operations return correct domain types
- Optional fields round-trip correctly (Some → DB → Some, None → DB → None)
- Range values translate correctly through sql.NullInt64 pairs
- Database constraints are enforced (CHECK, NOT NULL, UNIQUE)
- List operations return expected ordering

### What NOT to Test

- sqlc-generated code itself (it's tested by sqlc)
- SQL syntax (that's the migration's job)

---

## Interface Tests

**Location:** `internal/cli/*_test.go`, `internal/web/*_test.go`
**Dependencies:** moq doubles for service interfaces
**Focus:** Input parsing, output formatting, HTTP status codes, error handling

### Web Handler Tests

```go
func TestGameHandler_ShowGame(t *testing.T) {
    svc := &domaintesting.GameServiceMock{
        FindGameFunc: func(ctx context.Context, id int64) (domain.Boardgame, error) {
            return domain.Boardgame{ID: id, Name: "Catan"}, nil
        },
    }

    handler := web.NewGameHandler(svc, nil, slog.Default())

    req := httptest.NewRequest("GET", "/games/1", nil)
    rctx := chi.NewRouteContext()
    rctx.URLParams.Add("id", "1")
    req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))

    w := httptest.NewRecorder()
    handler.ShowGame(w, req)

    assert.Equal(t, http.StatusOK, w.Code)
    require.Len(t, svc.FindGameCalls(), 1)
    assert.Equal(t, int64(1), svc.FindGameCalls()[0].ID)
}

func TestGameHandler_ShowGame_InvalidID(t *testing.T) {
    svc := &domaintesting.GameServiceMock{}

    handler := web.NewGameHandler(svc, nil, slog.Default())

    req := httptest.NewRequest("GET", "/games/abc", nil)
    rctx := chi.NewRouteContext()
    rctx.URLParams.Add("id", "abc")
    req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))

    w := httptest.NewRecorder()
    handler.ShowGame(w, req)

    assert.Equal(t, http.StatusBadRequest, w.Code)
    assert.Empty(t, svc.FindGameCalls())  // service never called
}
```

### Narrow Service Interfaces for Handlers

Define interfaces in the handler file, not in domain:

```go
// internal/web/handlers/games.go
type BoardGameService interface {
    ListGames(context.Context) ([]domain.Boardgame, error)
    FindGame(context.Context, int64) (domain.Boardgame, error)
    AddGame(ctx context.Context, bg domain.Boardgame) (domain.Boardgame, err error)
}
```

Generate mocks for these narrow interfaces separately from domain mocks.

---

## Code Generation

### Adding go:generate Directives

Every interface that needs test doubles gets a directive:

```go
// internal/domain/boardgame_repository.go
//go:generate moq -out testing/boardgame_repository_moq.go . BoardgameRepository

type BoardgameRepository interface { ... }
```

### Running Generation

```bash
go generate ./...    # All generators (sqlc + moq)
```

### Generated Mock Structure

moq generates a struct with:
- `XxxFunc` fields — set these to stub behavior
- `XxxCalls()` methods — inspect call history

```go
type BoardgameRepositoryMock struct {
    CreateFunc      func(ctx context.Context, game Boardgame) (Boardgame, error)
    GetByIDFunc     func(ctx context.Context, id int64) (Boardgame, error)
    ListFunc        func(ctx context.Context) ([]Boardgame, error)
    SearchGamesFunc func(ctx context.Context, name string) ([]Boardgame, error)
}
```

---

## Guidelines

1. Use `require` for preconditions that must hold (stop test immediately on failure)
2. Use `assert` for verifications where the test should continue
3. One concern per test function — test name describes the scenario
4. Table-driven tests for multiple input variations of the same logic
5. Verify mock calls with `Calls()` — confirm the right methods were called with the right arguments
6. Never test sqlc-generated code directly
7. Integration tests use `:memory:` SQLite — fast, isolated, no cleanup needed
8. Keep test doubles in `domain/testing/` for repository mocks, handler-local for service mocks
