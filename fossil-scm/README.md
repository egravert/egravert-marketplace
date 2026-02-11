# Fossil SCM Plugin for Claude Code

A skill-based plugin that teaches Claude Code how to work with Fossil SCM repositories.

## Features

- Comprehensive Fossil SCM command reference (v2.27)
- Core workflows: commits, branching, merging, sync
- Ticketing system integration
- Git-to-Fossil mental model mapping

## Installation

### Local Testing

```bash
cc --plugin-dir /path/to/fossil-scm
```

### Permanent Installation

Copy to your Claude Code plugins directory or add to your project's `.claude-plugin/` folder.

## Usage

The skill activates automatically when you:

- Ask about Fossil commands ("How do I commit in Fossil?")
- Work with Fossil repositories (detected by `_FOSSIL_` or `.fslckout`)
- Request Fossil-specific operations ("fossil branch", "fossil merge")
- Ask about Fossil vs Git differences

## Examples

```
"How do I create a branch in Fossil?"
"Show me the Fossil stash commands"
"What's the difference between fossil sync and fossil push?"
"Help me merge this feature branch"
"How do I use the Fossil ticketing system?"
```

## Skill Contents

```
skills/fossil-usage/
├── SKILL.md                    # Core concepts and quick reference
└── references/
    └── commands.md             # Complete command documentation
```

## Version

Based on Fossil SCM version 2.27.
