# Agentic Practices Plugin for Claude Code

Installs agentic engineering practices directly into a project's CLAUDE.md as permanent coding standards. Rather than activating as a background skill, this plugin is invoked explicitly and writes language-specific practices into the project so they apply to every future session.

## How It Works

1. Invoke `/agentic-practices` in your project
2. The skill asks which language to install (currently: Go)
3. It appends a version-marked block to your project's CLAUDE.md (creates one if needed)
4. From that point on, the practices are always active for that project

Re-running the skill auto-updates the practices block if a newer version is available.

## What Gets Installed

The practices cover:

- **Intent-first comments** — explains why, not what; documents constraints, rejected alternatives, and safe assumptions
- **Decision documentation** — captures reasoning at decision time, not retroactively
- **Feature-boundary testing** — handler-level tests with business-rule naming, rich failure messages, and self-contained fakes
- **Greppable code design** — unique names, explicit wiring, no string-based dispatch
- **Agent instruction hygiene** — treats CLAUDE.md as code, keeps it current with the codebase

## Supported Languages

| Language | Practices File |
|----------|---------------|
| Go | `references/agentic-go-practices.md` |

More languages can be added by creating `references/agentic-<language>-practices.md`.

## Skill Contents

```
skills/agentic-practices/
├── SKILL.md                              # Installation procedure
└── references/
    └── agentic-go-practices.md           # Go-specific practices
```

## Version

1.0.0
