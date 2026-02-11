# egravert-marketplace

A personal Claude Code plugin marketplace.

## Plugins

| Plugin | Description |
|--------|-------------|
| [fossil-scm](./fossil-scm) | Fossil SCM command reference and workflow guidance |
| [go-hexagonal](./go-hexagonal) | Opinionated guide for hexagonal architecture in Go |

## Installation

Add this marketplace to Claude Code:

```
claude plugins:add-marketplace /path/to/egravert-marketplace
```

Then install individual plugins:

```
claude plugins:install fossil-scm@egravert-marketplace
claude plugins:install go-hexagonal@egravert-marketplace
```
