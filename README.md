# mise Plugin for Claude Code

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Use Cases](#use-cases)
- [Requirements](#requirements)
- [How It Works](#how-it-works)
- [License](#license)

---

## Overview

A Claude Code plugin that teaches Claude how to create and configure [mise](https://mise.jdx.dev/) tasks correctly. When installed, Claude will use the `usage` field for all task arguments (never shell-native `$1`/`$@` patterns), generate proper TOML-based and file-based tasks, configure dependencies and caching, and follow mise best practices.

This plugin is a pure skill - no commands, hooks, or MCP servers. It activates automatically whenever you ask Claude about mise task configuration.

## Features

- **Strict `usage` field enforcement** - all task arguments use mise's cross-platform `usage` spec, never shell-native `$1`/`$@`
- **TOML-based and file-based tasks** - supports both `mise.toml` inline tasks and executable script files with `#MISE`/`#USAGE` headers
- **Argument handling** - positional args, flags, choices validation, custom completions, variadic args, defaults
- **Task dependencies** - `depends`, `depends_post`, `wait_for` with parallel execution
- **Caching** - `sources`/`outputs` for last-modified checking and rebuild avoidance
- **Environment configuration** - env vars, `.env` file loading, PATH manipulation, required vars, redaction
- **Task namespacing** - subdirectory-based grouping (e.g., `test:unit`, `test:e2e`)
- **Remote tasks** - fetch tasks from URLs or git repos
- **Monorepo support** - experimental cross-project task orchestration
- **Tool requirements** - automatic tool installation before task execution
- **Global configuration** - `task_config`, `vars`, output modes, timeouts

## Installation

Install from the official Claude Code plugin directory:

```bash
claude plugin install mise
```

Or install directly from this repository:

```bash
claude plugin install --from https://github.com/brentmitchell25/mise-plugin.git
```

## Use Cases

### 1. Deploy with environment selection and flags

Ask Claude: *"Create a mise task that deploys to dev/staging/prod with a force flag"*

Claude generates tasks using the `usage` field with `choices` validation and proper `${usage_*}` variable access:

```toml
[tasks.deploy]
description = "Deploy application to environment"
depends = ["build", "test"]
usage = '''
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
flag "-f --force" help="Skip confirmation"
'''
confirm = "Deploy to {{usage.env}}?"
run = './scripts/deploy.sh ${usage_env?} ${usage_force:+--force}'
```

### 2. File-based tasks for database migrations

Ask Claude: *"Set up file-based mise tasks for database migrations"*

Claude creates executable scripts with `#MISE`/`#USAGE` headers and `${usage_*}` env vars:

```bash
#!/usr/bin/env bash
#MISE description="Run database migrations"
#MISE depends=["db:check"]
#USAGE arg "<direction>" choices "up" "down" help="Migration direction"
#USAGE flag "-n --count <n>" default="1" help="Number of migrations"
#USAGE flag "--dry-run" help="Preview SQL only"

set -euo pipefail
direction="${usage_direction?}"
count="${usage_count:-1}"

if [ -n "${usage_dry_run:-}" ]; then
  echo "Would run $count migration(s) $direction"
  exit 0
fi

migrate "$direction" -n "$count"
```

### 3. Build pipeline with caching and dependencies

Ask Claude: *"Configure a build pipeline with lint, test, build using caching"*

Claude creates tasks with `depends` for ordering and `sources`/`outputs` for cache-based rebuild avoidance:

```toml
[tasks.lint]
description = "Run linter"
sources = ["src/**/*.ts", ".eslintrc.*"]
run = "eslint src/"

[tasks.test]
description = "Run tests"
depends = ["lint"]
sources = ["src/**/*.ts", "tests/**/*.ts"]
run = "vitest run"

[tasks.build]
description = "Build for production"
depends = ["test"]
sources = ["src/**/*.ts", "tsconfig.json"]
outputs = ["dist/**/*"]
run = "tsc --build"
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [mise](https://mise.jdx.dev/) installed in your development environment

## How It Works

This plugin provides a single skill file (`SKILL.md`) that Claude reads when it detects mise-related context in your conversation. The skill contains:

1. **Strict enforcement rules** - Claude will never generate tasks with shell-native argument patterns
2. **Complete syntax reference** - all `usage` field syntax for args, flags, completions, and choices
3. **Configuration reference** - every task configuration key with types and descriptions
4. **Best practices** - DO/DON'T patterns for reliable task definitions
5. **Full examples** - complete TOML and file-based task examples ready to adapt

The skill activates automatically when you mention mise tasks, edit `mise.toml` files, or work with `mise-tasks/` directories.

## License

[MIT](LICENSE)
