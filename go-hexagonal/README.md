# Go Hexagonal Architecture Plugin for Claude Code

An opinionated guide for building Go applications using hexagonal (ports and adapters) architecture with clean separation between domain, service, storage, and interface layers.

## Features

- Layered architecture patterns with clear dependency flow
- Domain-driven design: entities, value objects, repository interfaces
- sqlc + goose database workflow with type translation patterns
- Three-tier testing strategy using moq and testify
- Package usage patterns for urfave/cli, chi, bubbletea, koanf, templ, and HTMX

## Usage

The skill activates automatically when you:

- Work on Go projects following hexagonal architecture
- Create new Go services or add features to existing ones
- Design package structure or implement repository patterns
- Ask about layered architecture, ports and adapters, or clean architecture in Go

## Skill Contents

```
skills/go-hexagonal/
├── SKILL.md                    # Architecture patterns and conventions
└── references/
    ├── database.md             # sqlc workflow, migrations, type translation
    ├── packages.md             # Package-specific usage patterns
    └── testing.md              # Testing strategy, moq, testify patterns
```

## Version

1.2.0
