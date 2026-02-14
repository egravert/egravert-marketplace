## Agentic Engineering Practices (Go)

Apply these practices when writing, reviewing, or modifying Go code. These rules optimize code for AI agent comprehension and collaboration. Follow them unless the project's CLAUDE.md explicitly overrides a specific rule.

### Comments & Documentation

**Intent over description.** Never write comments that describe what code does — agents can read the code. Write comments that explain:
- WHY this approach was chosen
- WHAT constraints it operates under (performance budgets, invariants)
- WHAT alternatives were rejected and why
- WHAT assumptions are safe to make

```go
// Bad:
// GetOrder retrieves an order by ID from the database

// Good:
// GetOrder retrieves an order and its full item tree. The item tree is
// denormalized at placement time because menu item IDs are unstable
// across published versions — we cannot look them up later.
// Constraint: called from the reorder flow which has a 500ms budget.
```

**Document decisions at the time they are made.** Do not infer "why" retroactively — record the reasoning when the decision happens. Include what was chosen, what was rejected, and the reason.

**Document constraints at the point they'll be violated.** Performance budgets go on the handler, not in a wiki. Invariants go on the interface, not in a design doc.

**Mark intentional duplication.** When code is deliberately duplicated rather than shared, add a comment explaining why, or agents will refactor it into a shared abstraction:
```go
// Intentionally duplicated from GameDetailHandler — these handlers need
// different fields and the shared version grew too many parameters.
```

### Package & Module Documentation

**Every package must have a `doc.go`** explaining its purpose, key concepts, and important design decisions.

**Domain packages must include a glossary** of key business concepts — the terms your team uses that an outsider (or agent) wouldn't know.

### Testing

**Test at the feature boundary.** Prefer handler-level or API-level tests that exercise the full request path: input parsing, service coordination, and response formatting. Do not write separate unit tests for each layer unless the business logic is complex enough to warrant isolation.

**Mock at the service interface, not at the repository level.** The handler's test should fake the service, not the database. This tests more code per test and catches wiring bugs between layers.

**Define fakes in the test file.** Do not create shared `testutil/` or `testing/` packages for fakes. Each test file should be self-contained. A fake's behavior should be visible in the same file as the test that uses it.

**Interfaces at point of consumption.** Handlers should define their own interfaces for the services they depend on. This keeps the dependency surface explicit, makes fakes trivial to write, and avoids importing shared interface packages:
```go
// Defined in the handler file, not in a shared package
type BoardGameService interface {
    ListGames(context.Context) ([]domain.Boardgame, error)
    FindGame(context.Context, int64) (domain.Boardgame, error)
}
```

**Test names must describe business rules**, not function names:
```go
// Good:
TestCreatePlay_returns_404_when_game_not_found
TestUpdateGame_preserves_existing_image_when_no_new_image_provided

// Bad:
TestCreatePlayError
TestUpdateGame
```

**Failure messages must describe the violated invariant:**
```go
// Good:
assert.Equal(t, http.StatusNotFound, rec.Code,
    "a missing game should return 404, not 500 — the client needs to distinguish 'not found' from 'server error'")

// Bad:
assert.Equal(t, http.StatusNotFound, rec.Code)
```

**Tests must run in isolation.** No shared mutable state, no required ordering, no implicit setup from other test files.

### Code Design

**Greppability.** Use unique, specific names that an agent can find with a single search. `OrderMatchScore` not `Score`. `GameStatusUnplayed` not `Unplayed`. If a function is called, there must be a greppable call site — avoid string-based dispatch and dynamic method resolution.

**Make the implicit explicit.** Document what middleware does at the registration point. Add `//go:generate` directives for code generation. Document environment variables at the point of use with their defaults and per-environment behavior.

**Share rules, duplicate assembly.** Business logic with a single correct definition should be shared. Data fetching, response assembly, and transformation should be duplicated per caller — they diverge over time.

**Rich error messages.** Wrap errors with enough context to trace causation:
```go
// Good: an agent can trace this immediately
return fmt.Errorf("menu publish failed for location %s: version %d conflicts with active version %d: %w",
    locationID, version, activeVersion, ErrVersionConflict)

// Bad: agent must search the entire codebase for all return sites
return ErrVersionConflict
```

### Schema & Data

**Document columns** at the point of definition — what they store, why they exist, and what constraints they enforce.

**Document migrations** with the motivation for the change, not just the DDL.

**Document data format choices** (e.g., "dates stored as RFC3339 TEXT because SQLite has no native datetime type").

### Agent Instruction Files

**If it's not in the instruction file, the agent won't do it.** Agents default to generic patterns from their training data. Project-specific conventions must be in CLAUDE.md (or equivalent) to take effect.

**Treat instruction files as code.** Update them in the same PR that changes the thing they describe. Stale instructions are worse than no instructions.

Required contents:
- Build/test/lint commands
- Architectural constraints (what never to do)
- Testing conventions (granularity, mock strategy, naming format)
- Domain glossary
- Directory structure (keep it current)
