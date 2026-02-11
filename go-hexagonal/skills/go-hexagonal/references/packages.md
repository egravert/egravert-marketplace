# Package Usage Patterns

Opinionated package selections and their usage patterns for consistency across projects.

## Table of Contents

- [urfave/cli — CLI Framework](#urfavecli)
- [bubbletea — TUI Framework](#bubbletea)
- [chi — HTTP Router](#chi)
- [templ — HTML Templating](#templ)
- [HTMX — Interactive Web](#htmx)
- [koanf — Configuration](#koanf)
- [slog — Structured Logging](#slog)
- [SCS — Session Management](#scs)

---

## urfave/cli

Package: `github.com/urfave/cli/v3`

### Command Structure

Each domain area gets its own handler struct and `Command()` method:

```go
type gameHandler struct {
    svc service.BoardGameService
}

func NewGameHandler(svc *service.BoardGameService) *gameHandler {
    return &gameHandler{svc: svc}
}

func (h *gameHandler) Command() *cli.Command {
    return &cli.Command{
        Name:  "game",
        Usage: "manage boardgames",
        Commands: []*cli.Command{
            {
                Name:      "add",
                Aliases:   []string{"a"},
                Arguments: []cli.Argument{&cli.StringArg{Name: "name"}},
                Flags: []cli.Flag{
                    &cli.Uint8Flag{Name: "complexity", Value: 2},
                    &cli.GenericFlag{
                        Name:  "players",
                        Value: &RangeValue{Range: domain.None[domain.Range]()},
                    },
                },
                Action: h.AddGame,
            },
        },
    }
}
```

### Custom Flag Types

Implement the `cli.Generic` interface for domain-specific parsing:

```go
type RangeValue struct {
    Range domain.Optional[domain.Range]
}

func (r *RangeValue) Set(s string) error {
    // Parse "2-4" or "2" into domain.Range
}

func (r *RangeValue) String() string {
    if !r.Range.IsSet() { return "" }
    return r.Range.Value().String()
}
```

### Wiring in main.go

```go
func main() {
    db := openDatabase(dbPath)
    repo := storage.NewBoardgameRepository(dbq.New(db))
    svc := service.NewBoardGameService(repo, logger)

    gameCmd := cli.NewGameHandler(svc)

    app := &cli.Command{
        Name:  "rollcall",
        Usage: "boardgame tracker",
        Commands: []*cli.Command{
            gameCmd.Command(),
        },
    }
    app.Run(context.Background(), os.Args)
}
```

### Conventions

- Handlers receive services, never repositories
- Action methods map CLI inputs to domain types, call service, format output
- Use `tabwriter` for tabular output
- Exit with meaningful error messages, not stack traces

---

## bubbletea

Package: `github.com/charmbracelet/bubbletea`

### Elm Architecture

Every component implements `tea.Model` with `Init()`, `Update()`, `View()`:

```go
type Tab struct {
    games   *service.BoardGameService
    list    list.Model
    detail  *detail.Model
    viewMode viewMode
    keys    keyMap
}

func (t *Tab) Init() tea.Cmd {
    return t.loadGames  // async data fetch
}

func (t *Tab) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        t.list.SetSize(msg.Width, msg.Height-4)
    case gamesLoadedMsg:
        t.list.SetItems(toListItems(msg.games))
    case tea.KeyMsg:
        if key.Matches(msg, t.keys.Select) {
            return t, t.showDetail()
        }
    }
    // delegate to child components
    var cmd tea.Cmd
    t.list, cmd = t.list.Update(msg)
    return t, cmd
}
```

### Typed Messages

Use private message types for component communication:

```go
type gamesLoadedMsg struct {
    games []domain.Boardgame
}

type errMsg struct {
    err error
}
```

### Commands for Async Work

```go
func (t *Tab) loadGames() tea.Msg {
    games, err := t.games.ListGames(context.Background())
    if err != nil {
        return errMsg{err}
    }
    return gamesLoadedMsg{games}
}
```

### Conventions

- Use `bubbles` components (list, textinput, table) as building blocks
- Keep models composable — parent delegates to children
- All I/O happens through commands, never in Update/View
- Use key bindings with `key.Binding` for consistent keyboard handling

---

## chi

Package: `github.com/go-chi/chi/v5`

### Router Setup

```go
func NewServer(bs *service.BoardGameService, ss *service.SessionService, logger *slog.Logger) http.Handler {
    r := chi.NewRouter()

    r.Use(middleware.RequestID)
    r.Use(middleware.Logger)
    r.Use(sm.LoadAndSave)  // session middleware

    // Public routes
    r.Get("/login", authHandler.LoginPage)
    r.Post("/login", authHandler.Login)

    // Protected group
    r.Group(func(r chi.Router) {
        r.Use(RequireAuth(sm))
        r.Get("/games", gameHandler.ListGames)
        r.Post("/games", gameHandler.CreateGame)
        r.Get("/games/{id}", gameHandler.ShowGame)
    })

    return r
}
```

### Handler Pattern

Handlers are structs with narrow service interfaces:

```go
type GameHandler struct {
    games    BoardGameService
    sessions SessionService
    logger   *slog.Logger
}

func (h *GameHandler) ShowGame(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
    if err != nil {
        http.Error(w, "Invalid game ID", http.StatusBadRequest)
        return
    }

    game, err := h.games.FindGame(r.Context(), id)
    if err != nil {
        http.Error(w, "Game Not Found", http.StatusNotFound)
        return
    }

    templates.GameShow(game).Render(r.Context(), w)
}
```

### Error Mapping

```
Parse error     → 400 Bad Request
Not found       → 404 Not Found
Internal error  → 500 Internal Server Error + slog.Error()
```

### Conventions

- Use `r.Group()` for middleware scoping (auth, CORS, etc.)
- URL params via `chi.URLParam(r, "id")`
- Handlers call services, never repositories
- Return early on errors (guard clause pattern)

---

## templ

Package: `github.com/a-h/templ`

### Component Structure

```templ
templ GameCard(game domain.Boardgame) {
    <div class="game-card">
        <h3>{ game.Name }</h3>
        if game.Players.IsSet() {
            <span>{ game.Players.Value().String() } players</span>
        }
    </div>
}

templ GamesIndex(games []domain.Boardgame) {
    @Layout("Games") {
        <div class="game-grid">
            for _, g := range games {
                @GameCard(g)
            }
        </div>
    }
}
```

### Layout Pattern

```templ
templ Layout(title string) {
    <!DOCTYPE html>
    <html>
        <head><title>{ title }</title></head>
        <body>
            @Nav()
            <main>
                { children... }
            </main>
        </body>
    </html>
}
```

### Rendering from Handlers

```go
templates.GamesIndex(games).Render(r.Context(), w)
```

### Conventions

- Components accept domain types directly (templ is in the interface layer)
- Use `{ children... }` for layout composition
- Conditional rendering with `if`/`for` in templ syntax
- Keep styles in the layout template or external CSS

---

## HTMX

### Partial vs Full Page Rendering

Detect HTMX requests and return partials:

```go
func IsHTMX(r *http.Request) bool {
    return r.Header.Get("HX-Request") == "true"
}

func (h *GameHandler) ListGames(w http.ResponseWriter, r *http.Request) {
    games, _ := h.games.ListGames(r.Context())

    if IsHTMX(r) {
        templates.GameList(games).Render(r.Context(), w)  // partial
    } else {
        templates.GamesIndex(games).Render(r.Context(), w)  // full page
    }
}
```

### Common Patterns in Templates

```templ
// Search with HTMX
<input type="search"
    name="q"
    hx-get="/games/search"
    hx-trigger="input changed delay:300ms"
    hx-target="#results" />

// Form submission
<form hx-post="/games" hx-target="#game-list" hx-swap="beforeend">
    ...
</form>
```

---

## koanf

Package: `github.com/knadh/koanf`

### Layered Configuration

```go
func loadConfig() (Config, error) {
    k := koanf.New(".")

    // Layer 1: Defaults
    defaults := map[string]interface{}{
        "addr":    ":8080",
        "db_path": "db/rollcall_dev.db",
    }
    k.Load(confmap.Provider(defaults, "."), nil)

    // Layer 2: Environment variables
    cb := func(s string) string {
        return strings.ToLower(strings.TrimPrefix(s, "ROLLCALL_"))
    }
    k.Load(env.Provider("ROLLCALL_", ".", cb), nil)

    return Config{
        Addr:   k.String("addr"),
        DBPath: k.String("db_path"),
    }, nil
}
```

### Per-Interface Strategy

| Interface | Config Sources |
|-----------|---------------|
| CLI | defaults + config file + env vars + CLI flags |
| TUI | defaults + config file + env vars |
| Web | defaults + env vars (12-factor style) |

### Critical Rule

Load config in `cmd/*/main.go`. Extract values into a plain `Config` struct. Pass individual values to internal layers. **Never pass the koanf instance** into services or storage.

---

## slog

Package: `log/slog` (standard library)

### Setup in main.go

```go
// JSON for production (web)
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

// Text for development (CLI)
logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))
```

### Injection Pattern

```go
type BoardGameService struct {
    logger *slog.Logger
    repo   domain.BoardgameRepository
}

func NewBoardGameService(repo domain.BoardgameRepository, logger *slog.Logger) *BoardGameService {
    return &BoardGameService{logger: logger, repo: repo}
}
```

### Usage

```go
slog.Warn("could not parse date", "session_id", id, "error", err)
slog.Info("game created", "game_id", game.ID, "name", game.Name)
slog.Error("database failure", "error", err)
```

### Conventions

- Always use structured key-value pairs, not string interpolation
- `Debug` for development tracing
- `Info` for significant operations
- `Warn` for recoverable issues
- `Error` for failures that affect the response

---

## SCS — Session Management

Package: `github.com/alexedwards/scs/v2`

### Setup

```go
sm := scs.New()
sm.Store = sqlite3store.New(db)
sm.Lifetime = 24 * time.Hour

r.Use(sm.LoadAndSave)
```

### Auth Middleware

```go
func RequireAuth(sm *scs.SessionManager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            userID := sm.GetInt64(r.Context(), "user_id")
            if userID == 0 {
                http.Redirect(w, r, "/login", http.StatusSeeOther)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```
