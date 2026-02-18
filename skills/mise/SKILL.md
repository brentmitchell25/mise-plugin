---
name: mise
description: Use when creating or editing mise, .mise.toml, or mise.toml task sections, creating mise task files in mise-tasks/ or .mise/tasks/ directories, configuring task arguments with usage field, setting task dependencies, or when user mentions mise task configuration.
---

# Mise Tasks Skill

## Table of Contents

- [🔴 STRICT ENFORCEMENT: Usage Field Required](#-strict-enforcement-usage-field-required)
- [Overview](#overview)
- [Task Definition Methods](#task-definition-methods)
- [Task Arguments (Usage Field)](#task-arguments-usage-field)
- [Task Configuration Reference](#task-configuration-reference)
  - [Global Task Configuration](#global-task-configuration)
- [Running Tasks (CLI)](#running-tasks-cli)
- [Task Dependencies & Caching](#task-dependencies--caching)
- [Environment Configuration](#environment-configuration)
- [Prepare Feature (Experimental)](#prepare-feature-experimental)
- [Monorepo Tasks (Experimental)](#monorepo-tasks-experimental)
- [Tool Requirements](#tool-requirements)
- [Automatic Environment Variables](#automatic-environment-variables)
- [Best Practices](#best-practices)

---

## 🔴 STRICT ENFORCEMENT: Usage Field Required

**This skill WILL NOT generate tasks with shell-native argument handling.**

All task arguments MUST be defined using the `usage` field. This is non-negotiable.

### ✅ REQUIRED Pattern

```toml
[tasks.deploy]
description = "Deploy to environment"
usage = 'arg "<env>" help="Target environment" choices "dev" "staging" "prod"'
run = 'deploy.sh ${usage_env?}'
```

```bash
#!/usr/bin/env bash
#MISE description="Process files"
#USAGE arg "<input>" help="Input file"
#USAGE arg "[output]" default="out.txt" help="Output file"
echo "Processing ${usage_input?} -> ${usage_output:-out.txt}"
```

### ❌ BLOCKED Patterns

```toml
# BLOCKED: Bash positional arguments
[tasks.bad]
run = 'deploy.sh $1 $2'

# BLOCKED: Bash special variables
[tasks.also_bad]
run = 'process.sh "$@"'

# BLOCKED: PowerShell args
[tasks.nope]
run = 'script.ps1 $args'

# BLOCKED: Inline Tera templates (deprecated, removed 2026.11)
[tasks.deprecated]
run = 'echo {{arg(name="x")}}'
```

### Why This is Enforced

| Benefit | Description |
|---------|-------------|
| **Type Safety** | Arguments validated before execution |
| **Auto-completion** | Shell completions generated automatically |
| **Cross-platform** | Works on bash, zsh, fish, PowerShell |
| **Self-documenting** | `mise run --help` shows all options |
| **No Parsing Bugs** | Eliminates shell quoting/escaping issues |
| **Choices Validation** | Invalid values rejected immediately |

---

## Overview

mise tasks are project-specific commands defined in `mise.toml` or as executable scripts. They automatically include the mise environment with configured tools and environment variables.

**Key Features:**

- Parallel dependency building (concurrent by default)
- Last-modified checking (avoid unnecessary rebuilds)
- File watching (`mise watch` rebuilds on changes)
- Script files with proper syntax highlighting
- Cross-platform argument handling via `usage` field

---

## Task Definition Methods

### TOML-Based Tasks (in mise.toml)

#### Simple Tasks

```toml
[tasks]
build = "cargo build"
test = "cargo test"
lint = "cargo clippy"
```

#### Detailed Tasks

```toml
[tasks.build]
description = "Build the CLI"
run = "cargo build"

[tasks.test]
description = "Run tests"
depends = ["build"]
run = "cargo test"
```

### File-Based Tasks (executable scripts)

Place in `mise-tasks/` or `.mise/tasks/` directory:

```bash
#!/usr/bin/env bash
#MISE description="Build the CLI"
#MISE depends=["lint"]
cargo build
```

Supported directories:
- `mise-tasks/`
- `.mise-tasks/`
- `mise/tasks/`
- `.mise/tasks/`
- `.config/mise/tasks/`

File tasks use `#MISE` or `#USAGE` comment headers for configuration.

**Alternative header syntax** (if formatters modify `#MISE`):
```bash
# [MISE] description="Build"
# [MISE] depends=["lint"]
```

### Task Grouping (Namespaces)

Subdirectories create namespaced tasks:

```
mise-tasks/
├── test/
│   ├── unit          → test:unit
│   ├── integration   → test:integration
│   └── e2e           → test:e2e
└── lint/
    ├── eslint        → lint:eslint
    └── prettier      → lint:prettier
```

### Remote Tasks

Fetch tasks from external sources:

```toml
[tasks.build]
file = "https://example.com/build.sh"

[tasks.release]
file = "git::https://github.com/org/repo.git//scripts/release.sh?ref=v1.0.0"
```

Remote files are cached in `MISE_CACHE_DIR`.

---

## Task Arguments (Usage Field)

**REMINDER:** Always use the `usage` field. Never use `$1`, `$@`, or shell-native argument handling.

### Syntax Overview

```toml
[tasks.example]
usage = '''
arg "<required>" help="Required argument"
arg "[optional]" default="value" help="Optional argument"
flag "-v --verbose" help="Enable verbose output"
flag "--output <file>" help="Output file path"
'''
run = 'process ${usage_required?} ${usage_optional:-default} ${usage_verbose:+--verbose}'
```

### Positional Arguments (`arg`)

| Syntax | Purpose |
|--------|---------|
| `arg "<name>"` | Required argument (angle brackets) |
| `arg "[name]"` | Optional argument (square brackets) |
| `arg "<file>"` | File path with shell completion |
| `arg "<dir>"` | Directory path with shell completion |
| `var=#true` | Variadic (accepts multiple values) |
| `var_min=N var_max=M` | Constrain count |
| `default="value"` | Default if not provided |
| `choices "a" "b" "c"` | Enumerated options |
| `help="description"` | Help text |
| `long_help="extended text"` | Extended help for `--help` |
| `env="VAR_NAME"` | Backed by environment variable |
| `double_dash="required"` | Control `--` behavior (`required`, `optional`, `automatic`) |

**Examples:**

```
arg "<file>" help="Input file to process"
arg "[output]" default="out.txt" help="Output file"
arg "<files>" var=#true var_min=1 help="One or more files"
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
```

### Flags (`flag`)

| Syntax | Purpose |
|--------|---------|
| `flag "-f --force"` | Boolean flag |
| `flag "--count" count=#true` | Countable (-vvv = 3) |
| `flag "--output <file>"` | Flag with value |
| `flag "--region <r>" default="us-east-1"` | Flag with default |
| `negate="--no-color"` | Creates negation flag |
| `hide=#true` | Hide from help |
| `global=#true` | Flag available to all subcommands |
| `long_help="extended text"` | Extended help for `--help` |

**Examples:**

```
flag "-v --verbose" help="Enable verbose output"
flag "-f --force" help="Skip confirmation prompts"
flag "--port <port>" default="8080" help="Server port"
flag "--color" negate="--no-color" help="Enable colors"
flag "-d --debug" count=#true help="Debug level (-ddd for max)"
```

### Custom Completions (`complete`)

Use `complete` to provide **dynamic tab-completion** for arguments. This is preferred over hardcoded `choices` when values may change over time (e.g., directories, services, environments derived from files).

**Docs:** https://usage.jdx.dev/spec/complete

**Syntax:**

```
complete "<arg_name>" run="<shell command that outputs one value per line>"
```

The `run` command is executed at completion time and should output one option per line to stdout.

**Key rules:**
- The `complete` directive must appear **after** the `arg` it applies to
- The `arg` name and `complete` name must match exactly
- When using `complete`, do **not** also use `choices` on the same `arg` — they conflict
- The `run` command runs from the project root directory

**Basic examples:**

```
# Static list via echo
complete "environment" run="echo 'dev\nstaging\nprod'"

# Dynamic from mise
complete "plugin" run="mise plugins ls"

# Dynamic from filesystem with descriptions
complete "file" run="fd --type f" descriptions=#true
```

**Common patterns for dynamic completions:**

```toml
# List subdirectories matching a pattern (e.g., services with an application/ dir)
usage = '''
arg "<service>" help="Service name"
complete "service" run="ls -d infrastructure/*/application 2>/dev/null | sed 's|infrastructure/||;s|/application||'"
'''

# Derive values from filenames (e.g., environments from .tfvars files)
usage = '''
arg "<environment>" help="Target environment"
complete "environment" run="ls infrastructure/ecom-service/application/env/ 2>/dev/null | sed 's/.tfvars//'"
'''

# Multiple args each with their own completions
usage = '''
arg "<environment>" help="Target environment"
complete "environment" run="ls infrastructure/ecom-service/application/env/ 2>/dev/null | sed 's/.tfvars//'"
arg "<service>" help="Service name"
complete "service" run="ls -d infrastructure/*/resources 2>/dev/null | sed 's|infrastructure/||;s|/resources||'"
'''
```

**`choices` vs `complete` — when to use which:**

| Feature | `choices` | `complete` |
|---------|-----------|------------|
| Values | Hardcoded in toml | Dynamic from command |
| Validation | Rejects invalid input | Tab-completion only (no validation) |
| Maintenance | Must edit toml to add options | Auto-discovers new options |
| Best for | Stable enums (yes/no, log levels) | File-derived, directory-derived values |

**Tip:** If you need both validation AND dynamic values, you can combine `choices` with known values and keep `complete` for discoverability, but typically you pick one or the other.

### Accessing Arguments in Scripts

**Environment variables** (prefixed with `usage_`):

```bash
# Required arg - error if missing
${usage_environment?}

# Optional with default
${usage_output:-default.txt}

# Boolean flag as conditional
${usage_verbose:+--verbose}

# Check if flag set
if [ -n "$usage_dry_run" ]; then
  echo "Dry run mode"
fi

# Variadic args (space-separated)
for file in $usage_files; do
  process "$file"
done
```

**Bash Variable Access Patterns:**

| Pattern | Use Case |
|---------|----------|
| `${var?}` | Required args - fail if missing |
| `${var:-default}` | Optional with fallback |
| `${var:?msg}` | Required with custom error |
| `${var:+value}` | Conditional (if set, use value) |

**In Tera templates:**

```toml
run = "deploy {{ usage.environment }} {% if usage.verbose %}--verbose{% endif %}"
```

### File Task Headers

```bash
#!/usr/bin/env bash
#MISE description="Deploy application"
#USAGE arg "<environment>" help="Environment" {
#USAGE   choices "dev" "prod"
#USAGE }
#USAGE flag "-f --force" help="Skip confirmation"

echo "Deploying to ${usage_environment?}"
```

---

## Task Configuration Reference

### Required

| Key | Type | Description |
|-----|------|-------------|
| `run` | `string \| array` | Command(s) to execute |

### Metadata

| Key | Type | Description |
|-----|------|-------------|
| `description` | `string` | Task description for help/completions |
| `alias` | `string \| string[]` | Alternative names for the task |
| `hide` | `bool` | Exclude from listings (default: false) |

### Dependencies

| Key | Type | Description |
|-----|------|-------------|
| `depends` | `string \| string[]` | Tasks to run BEFORE this task |
| `depends_post` | `string \| string[]` | Tasks to run AFTER this task |
| `wait_for` | `string \| string[]` | Wait for tasks without adding as deps |

**With arguments:**

```toml
depends = ["build --release", "lint"]
```

### Environment & Tools

| Key | Type | Description |
|-----|------|-------------|
| `env` | `{key: value}` | Task-specific env vars (NOT passed to depends) |
| `tools` | `{key: version}` | Tools to install before running |

### Execution Context

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dir` | `string` | `{{config_root}}` | Working directory |
| `shell` | `string` | system default | Shell interpreter |
| `raw` | `bool` | false | Direct stdin/stdout connection |

### Output Control

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `quiet` | `bool` | false | Suppress mise output |
| `silent` | `bool \| "stdout" \| "stderr"` | false | Suppress all/specific output |

### File Dependencies (Caching)

| Key | Type | Description |
|-----|------|-------------|
| `sources` | `string \| string[]` | Input files (glob patterns supported) |
| `outputs` | `string \| string[] \| {auto: true}` | Generated files |

### Safety

| Key | Type | Description |
|-----|------|-------------|
| `confirm` | `string` | Prompt user before running |

### Platform-Specific

| Key | Type | Description |
|-----|------|-------------|
| `run_windows` | same as run | Windows-specific command |

### Global Task Configuration

#### [task_config] Section

Configure defaults and include external tasks:

```toml
[task_config]
dir = "{{cwd}}"  # Default working directory for all tasks
includes = [
  "tasks/*.toml",                              # Local task files
  ".mise/tasks/",                              # Task directory
  "git::https://github.com/org/tasks?ref=v1"  # Remote tasks
]
```

#### [vars] Section

Shared variables between tasks (NOT environment variables):

```toml
[vars]
project_name = "myapp"
version = "1.0.0"
registry = "ghcr.io/myorg"

[vars._.file]
# Load vars from external file
path = "vars.toml"

[tasks.build]
run = "echo Building {{vars.project_name}} v{{vars.version}}"

[tasks.push]
run = "docker push {{vars.registry}}/{{vars.project_name}}:{{vars.version}}"
```

#### Global Settings

Configure in `~/.config/mise/config.toml` or project `mise.toml`:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `task_output` | string | `"prefix"` | Output mode (prefix, interleave, keep-order, replacing, timed, quiet, silent) |
| `task_timeout` | string | none | Default timeout (e.g., `"30m"`, `"1h"`) |
| `task_timings` | bool | `false` | Show elapsed time per task |
| `task_skip` | string[] | `[]` | Tasks to skip by default |
| `task_skip_depends` | bool | `false` | Run only specified tasks, skip dependencies |
| `task_run_auto_install` | bool | `true` | Auto-install missing tools before task |
| `jobs` | int | `4` | Max concurrent task execution |

```toml
# Example global settings
[settings]
task_output = "interleave"
task_timings = true
task_timeout = "10m"
jobs = 8
```

---

## Running Tasks (CLI)

### Commands

| Command | Description |
|---------|-------------|
| `mise run <task>` | Execute task |
| `mise r <task>` | Short form |
| `mise tasks` | List all tasks |
| `mise tasks --hidden` | Include hidden tasks |
| `mise tasks deps` | Show dependency tree |
| `mise tasks add <name> -- <cmd>` | Create task via CLI |

### Execution Flags

| Flag | Description |
|------|-------------|
| `--jobs N` | Parallel job limit (default: 4) |
| `--interleave` | Direct stdout/stderr |
| `--dry-run` | Preview without executing |
| `--force` | Ignore source/output caching |
| `--timings` | Show elapsed time per task |

### Wildcards

```bash
mise run test:*         # All test:* tasks
mise run lint:**        # All lint:** nested tasks
mise run {build,test}   # Multiple specific tasks
mise run test:unit*     # Pattern matching
```

### Output Modes

Set via `--output` flag or `task_output` setting:

| Mode | Description |
|------|-------------|
| `prefix` | Label each line (default) |
| `interleave` | Direct output, mixed |
| `keep-order` | Buffer output, ordered |
| `replacing` | Single line, replaced |
| `timed` | With timestamps |
| `quiet` | Suppress mise output |
| `silent` | Suppress all output |

### Default Task

Running `mise run` without arguments executes the "default" task if defined:

```toml
[tasks.default]
depends = ["build", "test"]
run = "echo 'Ready!'"
```

---

## Task Dependencies & Caching

### Dependencies

```toml
[tasks.deploy]
depends = ["build", "test", "lint"]  # Run before
depends_post = ["notify"]             # Run after
wait_for = ["db:migrate"]             # Wait if running
```

Dependencies execute in **parallel** by default.

### Caching with sources/outputs

```toml
[tasks.build]
sources = ["Cargo.toml", "src/**/*.rs"]
outputs = ["target/release/myapp"]
run = "cargo build --release"
# Skips if sources unchanged and outputs exist
```

### Force Execution

```bash
mise run build --force  # Ignore cache
```

---

## Environment Configuration

### Basic Variables

```toml
[env]
NODE_ENV = 'production'
DEBUG = 'app:*'
PORT = 3000
```

### Unset Variables

```toml
[env]
UNWANTED_VAR = false
```

### CLI Management

```bash
mise set NODE_ENV=development
mise set                    # View all
mise unset NODE_ENV
```

### Special Directives (env._)

| Directive | Purpose |
|-----------|---------|
| `_.path` | Prepend to PATH |
| `_.file` | Load from .env files |
| `_.source` | Source shell script |

```toml
[env]
_.path = ["tools/bin", "{{config_root}}/scripts"]
_.file = ['.env', '.env.local', '.env.{{env.MISE_ENV}}']
_.source = "./setup-env.sh"
```

### Required Variables

```toml
[env]
DATABASE_URL = { required = "Set postgres connection string (e.g., postgres://user:pass@localhost/db)" }
```

### Redaction (Hide Sensitive Values)

```toml
[env]
API_KEY = { value = "secret_key_here", redact = true }
redactions = ["*_TOKEN", "SECRET_*", "API_*"]
```

### Lazy Evaluation (Access Tool Paths)

```toml
[env]
PYTHON_BIN = { value = "{{env.VIRTUAL_ENV}}/bin", tools = true }
NODE_PATH = { value = "{{env.npm_config_prefix}}/lib/node_modules", tools = true }
```

### Templates

```toml
[env]
PROJECT_DIR = "{{config_root}}"
LOG_FILE = "{{config_root}}/logs/{{now() | date(format='%Y-%m-%d')}}.log"
```

---

## Prepare Feature (Experimental)

Ensures dependencies are installed before task execution.

### Enable

```bash
export MISE_EXPERIMENTAL=1
mise prepare  # or: mise prep
```

### Configuration

```toml
[prepare.npm]
auto = true  # Auto-run before mise x/run

[prepare.pnpm]
auto = true

[prepare.custom]
sources = ["schema/*.graphql"]
outputs = ["src/generated/"]
run = "npm run codegen"
```

### Built-in Providers

npm, yarn, pnpm, bun, go, pip, poetry, uv, bundler, composer

Each provider knows its lockfile (e.g., `package-lock.json`) and output directory (e.g., `node_modules/`).

### CLI

```bash
mise prepare --dry-run      # Preview what would run
mise prepare --force        # Force execution
mise prepare --list         # List providers
mise prepare --only npm     # Run specific providers
```

### How It Works

1. Compares lockfile mtime against output directory mtime
2. If lockfile newer → runs install command
3. If outputs newer → skips (already up to date)

---

## Monorepo Tasks (Experimental)

Organize tasks across multiple projects in a single repository.

### Enable

```toml
# Root mise.toml
experimental_monorepo_root = true

[tools]
node = "20"
```

Requires: `MISE_EXPERIMENTAL=1` environment variable.

### Task Path Syntax

Monorepo tasks use special prefixes with `//` and `:`:

| Syntax | Description |
|--------|-------------|
| `mise //projects/frontend:build` | Absolute path from monorepo root |
| `mise :build` | Task in current config_root |
| `mise '//projects/frontend:*'` | All tasks in frontend |
| `mise //...:test` | Test task in all projects |

The `//` prefix indicates monorepo root. Tasks never conflict with core mise commands when using these prefixes.

### Wildcards

**Path wildcards** (`...`) match any directory depth:

```bash
mise //...:test              # All test tasks across all projects
mise '//projects/...:build'  # All build tasks in projects tree
```

**Task name wildcards** (`*`) target specific tasks:

```bash
mise '//projects/frontend:*'  # All tasks in frontend
mise '//...:test*'            # All test-prefixed tasks everywhere
```

### Tool & Environment Inheritance

Subdirectories automatically inherit parent configurations but can override:

```
root/
├── mise.toml              # node = "20" (default)
├── frontend/
│   └── mise.toml          # node = "18" (override)
└── backend/
    └── mise.toml          # inherits node = "20"
```

Precedence: global configs → parent overrides → task-specific definitions.

### Configuration

```toml
[task]
monorepo_depth = 5                              # Search depth (default: 5)
monorepo_exclude_dirs = ["dist", "node_modules", "target", "build"]
monorepo_respect_gitignore = true               # Skip .gitignore'd directories
```

### Automatic Exclusions

Task discovery automatically skips:
- Hidden directories (`.git`, `.cache`, etc.)
- `node_modules`, `target`, `dist`, `build`

### Listing Tasks

```bash
mise tasks          # Shows current config_root hierarchy
mise tasks --all    # Shows all monorepo tasks
```

---

## Tool Requirements

Specify tools that must be available for a task:

```toml
[tasks.test]
tools = { node = "20", pnpm = "latest" }
run = "pnpm test"

[tasks.build]
tools = { rust = "1.75", cargo-watch = "latest" }
run = "cargo build --release"
```

Tools are installed automatically before task execution.

---

## Automatic Environment Variables

Tasks automatically receive these variables:

| Variable | Description |
|----------|-------------|
| `MISE_ORIGINAL_CWD` | Original working directory |
| `MISE_CONFIG_ROOT` | Directory containing mise.toml |
| `MISE_PROJECT_ROOT` | Project root directory |
| `MISE_TASK_NAME` | Current task name |
| `MISE_TASK_DIR` | Task script directory |
| `MISE_TASK_FILE` | Full path to task script |

---

## Best Practices

### DO ✅

- **Always use `usage` field** for arguments
- Use `${var?}` for required args to fail early
- Set `description` for discoverability
- Use `sources`/`outputs` for cacheable tasks
- Use `depends` for task ordering
- Use `confirm` for destructive operations
- Use `choices` for stable enums, `complete` for dynamic/filesystem-derived values
- Group related tasks with namespaces (e.g., `test:unit`, `test:e2e`)

### DON'T ❌

- Use `$1`, `$2`, `$@`, `$*` for arguments
- Use `$args` in PowerShell
- Use inline `{{arg(name="x")}}` syntax (deprecated, removed 2026.11)
- Forget to quote glob patterns in sources
- Set env vars in `env` that deps need (they don't inherit)
- Use `raw = true` unless interactive input needed

### Complete Task Example

```toml
[tasks.deploy]
description = "Deploy application to environment"
alias = "d"
depends = ["build", "test"]
usage = '''
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
flag "-f --force" help="Skip confirmation"
flag "--dry-run" help="Preview only"
'''
env = { DEPLOY_TIMESTAMP = "{{now()}}" }
tools = { node = "20" }
sources = ["dist/**/*"]
confirm = "Deploy to {{usage.env}}?"
run = '''
#!/usr/bin/env bash
set -euo pipefail

if [ -n "${usage_dry_run:-}" ]; then
  echo "DRY RUN: Would deploy to ${usage_env?}"
  exit 0
fi

./scripts/deploy.sh "${usage_env?}"
'''
```

### Complete File Task Example

```bash
#!/usr/bin/env bash
#MISE description="Run database migrations"
#MISE alias="migrate"
#MISE depends=["db:check"]
#MISE tools={postgresql="16"}
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
