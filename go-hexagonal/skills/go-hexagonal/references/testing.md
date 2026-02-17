# Testing Strategy

Test at the feature boundary. Handler-level tests are the primary layer — they exercise the full request path through inline fakes of the service interface. Storage integration tests verify SQL correctness against real SQLite. Service-layer unit tests are reserved for complex business logic that warrants isolation.

## Table of Contents

- [Test Tooling](#test-tooling)
- [Handler Tests — Primary](#handler-tests)
- [Storage Integration Tests](#storage-integration-tests)
- [Service Unit Tests — When Warranted](#service-unit-tests)
- [Naming & Messages](#naming--messages)
- [Guidelines](#guidelines)

---

## Test Tooling

| Tool | Purpose |
|------|---------|
| `testify/assert` | Non-fatal assertions (test continues on failure) |
| `testify/require` | Fatal assertions (test stops immediately on failure) |
| `testify/suite` | Test suites with setup/teardown for integration tests |

---

## Handler Tests

**Location:** `internal/cli/*_test.go`, `internal/web/*_test.go`
**Dependencies:** Hand-written inline fakes for service interfaces
**Focus:** Input parsing, service coordination, response formatting, HTTP status codes, error handling

Handler tests are the primary testing layer. They exercise the full feature path — from raw input to formatted output — by faking the service interface.

### Inline Fakes

Define fakes in the test file, not in a shared package. With narrow, consumer-owned interfaces (2-4 methods), a fake is 10-15 lines. Keeping it in the same file means a reader sees both the fake's behavior and the test that uses it in one place.

```go
// internal/web/handlers/boardgame_handler_test.go

type stubBoardGameService struct {
    games []domain.Boardgame
    err   error
}

func (s *stubBoardGameService) ListGames(ctx context.Context) ([]domain.Boardgame, error) {
    return s.games, s.err
}

func (s *stubBoardGameService) FindGame(ctx context.Context, id int64) (domain.Boardgame, error) {
    for _, g := range s.games {
        if g.ID == id {
            return g, nil
        }
    }
    return domain.Boardgame{}, s.err
}

func (s *stubBoardGameService) AddGame(ctx context.Context, bg domain.Boardgame) (domain.Boardgame, error) {
    bg.ID = int64(len(s.games) + 1)
    return bg, s.err
}
```

### Web Handler Example

```go
func TestListGames_returns_games_sorted_by_name(t *testing.T) {
    svc := &stubBoardGameService{
        games: []domain.Boardgame{
            {ID: 1, Name: "Catan"},
            {ID: 2, Name: "Ark Nova"},
        },
    }

    handler := web.NewGameHandler(svc, slog.Default())

    req := httptest.NewRequest(http.MethodGet, "/games", nil)
    rec := httptest.NewRecorder()

    handler.ListGames(rec, req)

    require.Equal(t, http.StatusOK, rec.Code,
        "listing games should return 200 even when the list is non-empty")
}

func TestShowGame_returns_404_when_game_not_found(t *testing.T) {
    svc := &stubBoardGameService{
        err: domain.ErrNotFound,
    }

    handler := web.NewGameHandler(svc, slog.Default())

    req := httptest.NewRequest(http.MethodGet, "/games/999", nil)
    rctx := chi.NewRouteContext()
    rctx.URLParams.Add("id", "999")
    req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))

    rec := httptest.NewRecorder()
    handler.ShowGame(rec, req)

    assert.Equal(t, http.StatusNotFound, rec.Code,
        "a missing game should return 404, not 500 — the client needs to distinguish 'not found' from 'server error'")
}

func TestShowGame_returns_400_for_non_numeric_id(t *testing.T) {
    svc := &stubBoardGameService{}

    handler := web.NewGameHandler(svc, slog.Default())

    req := httptest.NewRequest(http.MethodGet, "/games/abc", nil)
    rctx := chi.NewRouteContext()
    rctx.URLParams.Add("id", "abc")
    req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))

    rec := httptest.NewRecorder()
    handler.ShowGame(rec, req)

    assert.Equal(t, http.StatusBadRequest, rec.Code,
        "non-numeric game ID should be rejected before reaching the service layer")
}
```

### Narrow Service Interfaces

Define interfaces in the handler (or handler test) file — not in a shared package. This keeps the dependency surface explicit and makes fakes trivial to write:

```go
// internal/web/handlers/boardgame_handler.go
type BoardGameService interface {
    ListGames(context.Context) ([]domain.Boardgame, error)
    FindGame(context.Context, int64) (domain.Boardgame, error)
    AddGame(ctx context.Context, bg domain.Boardgame) (domain.Boardgame, error)
}
```

### What Handler Tests Cover

- Request parsing (URL params, query strings, form data, JSON bodies)
- Service coordination (correct service method called with correct arguments)
- Response formatting (status codes, response body structure)
- Error mapping (domain errors → HTTP status codes)
- Edge cases (missing params, invalid input, empty results)

---

## Storage Integration Tests

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

func (s *BoardgameRepoSuite) TestCreate_assigns_server_generated_id() {
    game := domain.Boardgame{
        Name:    "Catan",
        Players: domain.Some(mustRange(3, 4)),
    }

    created, err := s.repo.Create(context.Background(), game)

    s.Require().NoError(err, "creating a valid game should succeed")
    s.NotZero(created.ID, "server should assign a non-zero ID")
    s.Equal("Catan", created.Name)
}

func (s *BoardgameRepoSuite) TestGetByID_returns_error_for_missing_game() {
    _, err := s.repo.GetByID(context.Background(), 999)
    s.Error(err, "fetching a non-existent ID should return an error")
}

func (s *BoardgameRepoSuite) TestOptionalFields_roundtrip_correctly() {
    game := domain.Boardgame{
        Name:       "Wingspan",
        Players:    domain.Some(mustRange(1, 5)),
        Complexity: domain.Some(uint8(3)),
    }

    created, _ := s.repo.Create(context.Background(), game)
    fetched, err := s.repo.GetByID(context.Background(), created.ID)

    s.Require().NoError(err)
    s.True(fetched.Players.IsSet(), "players range should survive the round-trip")
    s.True(fetched.Complexity.IsSet(), "complexity should survive the round-trip")
    s.Equal(uint16(1), fetched.Players.Value().Min())
    s.Equal(uint16(5), fetched.Players.Value().Max())
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

- sqlc-generated code itself (it's already tested by sqlc)
- SQL syntax (that's the migration's job)

---

## Service Unit Tests

**When warranted:** Only write isolated service tests when the service contains complex business logic — non-trivial validation, calculations, multi-step workflows, or branching that's hard to exercise through handler tests alone. Simple CRUD delegation does not need its own test layer; handler tests cover it.

**Location:** `internal/service/*_test.go`
**Dependencies:** Hand-written inline fakes for repository interfaces
**Focus:** Business logic, validation, edge cases

### When to Write Service Tests

- Validation with multiple rules and edge cases
- Multi-repository coordination with complex failure modes
- Calculations or transformations that have many input combinations
- Business rules that are hard to observe from the handler level

### When NOT to Write Service Tests

- The service just delegates to a repository (CRUD pass-through)
- The handler test already exercises the relevant code path
- You'd be testing that the service calls the repository (that's a wiring test, not a logic test)

### Validation Example

```go
func TestAddGame_rejects_empty_name(t *testing.T) {
    repo := &stubBoardgameRepo{}
    svc := service.NewBoardGameService(repo, slog.Default())

    _, err := svc.AddGame(context.Background(), domain.Boardgame{Name: ""})

    assert.EqualError(t, err, "game name is required",
        "empty name should be rejected before reaching the repository")
    assert.False(t, repo.createCalled,
        "repository should not be called when validation fails")
}
```

### Inline Fakes for Repositories

```go
// Defined in the test file — not shared across packages
type stubBoardgameRepo struct {
    createCalled bool
    created      domain.Boardgame
    err          error
}

func (s *stubBoardgameRepo) Create(ctx context.Context, game domain.Boardgame) (domain.Boardgame, error) {
    s.createCalled = true
    s.created = game
    game.ID = 1
    return game, s.err
}

func (s *stubBoardgameRepo) GetByID(ctx context.Context, id int64) (domain.Boardgame, error) {
    return domain.Boardgame{}, s.err
}

func (s *stubBoardgameRepo) List(ctx context.Context) ([]domain.Boardgame, error) {
    return nil, s.err
}
```

### Table-Driven Validation Tests

```go
func TestAddGame_validation(t *testing.T) {
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
            repo := &stubBoardgameRepo{}
            svc := service.NewBoardGameService(repo, slog.Default())

            _, err := svc.AddGame(context.Background(), tt.game)

            assert.EqualError(t, err, tt.wantErr)
            assert.False(t, repo.createCalled,
                "repository should not be called when validation fails")
        })
    }
}
```

### Multi-Repository Coordination

```go
func TestAddSession_returns_error_when_game_not_found(t *testing.T) {
    games := &stubGameLookup{
        games: []domain.Boardgame{}, // no matches
    }
    sessions := &stubSessionRepo{}

    svc := service.NewSessionService(sessions, games)

    _, err := svc.AddSession(context.Background(), "Nonexistent", []string{"Alice"}, time.Now())

    assert.ErrorContains(t, err, "no games found",
        "searching for a non-existent game should fail with a descriptive error")
    assert.False(t, sessions.createCalled,
        "session should not be created when the game lookup fails")
}
```

---

## Naming & Messages

### Test Names

Test names describe business rules, not function names. Use the format `Test<Action>_<expected_outcome>` or `Test<Action>_<condition>`:

```go
// Good — describes the business rule
TestCreatePlay_returns_404_when_game_not_found
TestAddGame_rejects_empty_name
TestUpdateGame_preserves_existing_image_when_no_new_image_provided
TestListGames_returns_empty_slice_for_new_database

// Bad — describes the function, not the behavior
TestCreatePlayError
TestUpdateGame
TestBoardGameService_AddGame
```

### Failure Messages

Every assertion should include a message describing the violated invariant. A failing test should tell you what went wrong without reading the test body:

```go
// Good — explains what should have happened and why it matters
assert.Equal(t, http.StatusNotFound, rec.Code,
    "a missing game should return 404, not 500 — the client needs to distinguish 'not found' from 'server error'")

require.NoError(t, err,
    "creating a game with valid fields should always succeed")

assert.True(t, fetched.Players.IsSet(),
    "players range should survive the database round-trip")

// Bad — no message, or just restates the assertion
assert.Equal(t, http.StatusNotFound, rec.Code)
assert.Equal(t, http.StatusNotFound, rec.Code, "expected 404")
```

---

## Guidelines

1. **Use `require` for preconditions** — stop the test immediately when a prerequisite fails (e.g., "no error creating the test fixture")
2. **Use `assert` for verifications** — let the test continue to report all failures at once
3. **Test names describe business rules** — `TestCreatePlay_returns_404_when_game_not_found`, not `TestCreatePlayError`
4. **Failure messages describe invariants** — explain what *should* be true and why it matters
5. **Table-driven tests for variations** — use when testing multiple inputs against the same logic
6. **Never test sqlc-generated code** — it's already tested by sqlc
7. **Integration tests use `:memory:` SQLite** — fast, isolated, no cleanup needed
8. **Fakes live in the test file** — a reader should see both the fake's behavior and the test in one place
9. **Tests run in isolation** — no shared mutable state, no required ordering, no implicit setup from other test files
10. **Mock at the service interface** — handler tests fake the service, not the repository. Only fake repositories when testing service logic in isolation.
