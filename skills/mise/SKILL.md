---
name: mise
description: Use when working with mise - creating/editing mise.toml or .mise.toml files, defining tasks with usage field arguments, managing dev tools and environments, configuring hooks, or when user mentions mise configuration.
---

# Mise Comprehensive Skill

## Table of Contents

- [STRICT ENFORCEMENT: Usage Field Required](#-strict-enforcement-usage-field-required)
- [Overview](#overview)
- [Task Definition Methods](#task-definition-methods)
  - [TOML-Based Tasks](#toml-based-tasks-in-misetoml)
  - [File-Based Tasks](#file-based-tasks-executable-scripts)
  - [Task Grouping (Namespaces)](#task-grouping-namespaces)
  - [Remote Tasks](#remote-tasks)
- [Task Arguments - Usage Spec Reference](#task-arguments---usage-spec-reference)
  - [Positional Arguments (arg)](#positional-arguments-arg)
  - [Flags (flag)](#flags-flag)
  - [Custom Completions (complete)](#custom-completions-complete)
  - [Accessing Arguments in Scripts](#accessing-arguments-in-scripts)
  - [File Task Headers](#file-task-headers)
- [Task Configuration Reference](#task-configuration-reference)
  - [All Task Fields](#all-task-fields)
  - [Structured run Array](#structured-run-array)
  - [Structured depends with Args/Env](#structured-depends-with-argsenv)
  - [Task Inheritance (extends)](#task-inheritance-extends)
  - [Global Task Configuration](#global-task-configuration)
  - [Variables (vars)](#variables-vars)
- [Running Tasks (CLI)](#running-tasks-cli)
- [Task Dependencies and Caching](#task-dependencies-and-caching)
- [Dev Tools Management](#dev-tools-management)
  - [Backends Overview](#backends-overview)
  - [TOML Syntax for Tools](#toml-syntax-for-tools)
  - [Per-Tool Options](#per-tool-options)
  - [Backend-Specific Configuration](#backend-specific-configuration)
  - [Shims and Aliases](#shims-and-aliases)
- [Environment Configuration](#environment-configuration)
  - [Basic Variables](#basic-variables)
  - [Special Directives (env._)](#special-directives-env_)
  - [Profiles (MISE_ENV)](#profiles-mise_env)
  - [Required and Redacted Variables](#required-and-redacted-variables)
  - [Secrets (SOPS and age)](#secrets-sops-and-age)
  - [Templates (Tera)](#templates-tera)
- [Hooks and Watchers](#hooks-and-watchers)
- [Configuration and Settings](#configuration-and-settings)
  - [File Hierarchy](#file-hierarchy)
  - [Key Settings Reference](#key-settings-reference)
  - [Minimum Version](#minimum-version)
  - [Automatic Environment Variables](#automatic-environment-variables)
- [Prepare Feature (Experimental)](#prepare-feature-experimental)
- [Monorepo Tasks (Experimental)](#monorepo-tasks-experimental)
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

mise is an all-in-one developer environment tool that manages:

- **Dev tools** — install and manage language runtimes, CLIs, and build tools (18+ backends)
- **Tasks** — project-specific commands with argument handling, dependencies, caching
- **Environments** — manage env vars, profiles, dotenv files, secrets, age/sops-encrypted values
- **Hooks** — run commands on directory changes, project enter/leave, tool install

**Key Features:**
- Parallel dependency building (concurrent by default, up to `jobs` setting)
- Last-modified and content-hash (blake3) checking (avoid unnecessary rebuilds)
- File watching (`mise watch` rebuilds on changes)
- Cross-platform argument handling via `usage` spec
- Hierarchical configuration with profile support
- 18+ tool backends (aqua, github, gitlab, forgejo, npm, cargo, pipx, etc.)
- Security verification (cosign, SLSA, GitHub Attestations, minisign)
- Secret management (sops, age encryption)

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

#### Multiline Scripts

```toml
[tasks.build]
run = '''
#!/usr/bin/env bash
set -euo pipefail
cargo clippy
cargo build --release
'''
```

#### Shebang Support in TOML

Execute scripts in multiple languages via shebang:

```toml
[tasks.script]
run = '''
#!/usr/bin/env python
for i in range(10):
    print(i)
'''
```

Supports: Python, Node, Bun, Deno, Ruby, Bash, PowerShell. Use `-S` flag for multiple interpreter arguments.

### File-Based Tasks (executable scripts)

Place in task directories. Files **must** be executable (`chmod +x`).

```bash
#!/usr/bin/env bash
#MISE description="Build the CLI"
#MISE depends=["lint"]
cargo build
```

Supported directories (searched by default):
- `mise-tasks/`
- `.mise-tasks/`
- `mise/tasks/`
- `.mise/tasks/`
- `.config/mise/tasks/`

If `task_config.includes` is set, it **replaces** these defaults — list them explicitly to keep them.

File tasks use `#MISE` or `#USAGE` comment headers for configuration.

**Alternative header syntax** (if formatters modify `#MISE`):
```bash
# [MISE] description="Build"
# [MISE] depends=["lint"]
```

**Non-bash languages** use language-appropriate comment syntax:
```javascript
#!/usr/bin/env node
//USAGE flag "-v --verbose" help="Enable verbose output"
//MISE description="Run node script"
```

```python
#!/usr/bin/env python
#MISE description="Hello from Python"
```

```powershell
#!/usr/bin/env pwsh
#MISE description="Hello from PowerShell"
```

**Direct script execution** bypasses discovery (path must start with `/`, `./`, `C:\`, or `.\`):
```bash
mise run ./path/to/script.sh
```

### Task Grouping (Namespaces)

Subdirectories create namespaced tasks (colon separator):

```
mise-tasks/
├── build              → build
├── test/
│   ├── _default       → test (default task)
│   ├── unit           → test:unit
│   ├── integration    → test:integration
│   └── e2e            → test:e2e
└── lint/
    ├── eslint         → lint:eslint
    └── prettier       → lint:prettier
```

The file named `_default` executes when invoking the directory name without a subtask.

### Remote Tasks

Fetch tasks from external sources:

```toml
# HTTP
[tasks.build]
file = "https://example.com/build.sh"

# Git (experimental) — SSH
[tasks.release]
file = "git::ssh://git@github.com/org/repo.git//scripts/release.sh?ref=v1.0.0"

# Git (experimental) — HTTPS
[tasks.build]
file = "git::https://github.com/org/repo.git//path?ref=main"
```

Format: `git::<protocol>://<url>//<path>?<ref>` — ref is optional (defaults to repo's default branch).

Remote files cached in `$MISE_CACHE_DIR` (git specifically in `$MISE_CACHE_DIR/remote-git-tasks-cache`). Clear with `mise cache clear`. Disable cache with `MISE_TASK_REMOTE_NO_CACHE=true` or `--no-cache`.

Remote git includes (experimental) in `[task_config]`:
```toml
[task_config]
includes = ["git::https://github.com/myorg/shared-tasks.git//tasks?ref=main"]
```

---

## Task Arguments - Usage Spec Reference

**REMINDER:** Always use the `usage` field. Never use `$1`, `$@`, or shell-native argument handling.

The `usage` field uses [KDL-inspired syntax](https://usage.jdx.dev/) to define arguments, flags, and completions.

### Positional Arguments (`arg`)

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (name) | string | *required* | `"<name>"` = required, `"[name]"` = optional. `"<file>"` triggers file completion, `"<dir>"` triggers dir completion. |
| `help` | string | none | Short help text shown with `-h` |
| `long_help` | string | none | Extended help text shown with `--help` |
| `default` | string | none | Default value if not provided. `default=""` sets to empty string (different from unset). |
| `env` | string | none | Environment variable that can provide this arg's value. Priority: CLI > env > default. |
| `var` | boolean | `#false` | Variadic mode (accept multiple values). `"<name>"` requires 1+, `"[name]"` accepts 0+. Shorthand: `"<name>..."` |
| `var_min` | integer | none | Minimum values when variadic |
| `var_max` | integer | none | Maximum values when variadic |
| `choices` | values | none | Restrict to enumerated set. Also supports `choices env="VAR_NAME"` |
| `double_dash` | string | none | `"required"`, `"optional"`, `"automatic"`, `"preserve"` |
| `hide` | boolean | `#false` | Exclude from help output |
| `parse` | string | none | Parse arg value with external command (template: `"mycli parse {}"`) |

**Examples:**

```
arg "<file>" help="Input file to process"
arg "[output]" default="out.txt" help="Output file"
arg "<files>" var=#true var_min=1 help="One or more files"
arg "<files>..." help="Shorthand variadic syntax"
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
arg "<env>" { choices env="DEPLOY_ENVS" }
arg "<args>..." double_dash="automatic" help="Pass-through arguments"
arg "<file>" env="MY_FILE" help="Input file (or set MY_FILE)"
```

**Variadic args in bash:**
```bash
# Values are shell-escaped string in usage_files
eval "files=($usage_files)"
for f in "${files[@]}"; do
  echo "Processing: $f"
done
```

### Flags (`flag`)

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (definition) | string | *required* | `"-s --long"` (boolean) or `"-s --long <value>"` (with value). Short flag optional. |
| `alias` | string | none | Alternative short form (block-format flags only) |
| `help` | string | none | Short help text |
| `long_help` | string | none | Extended help for `--help` |
| `default` | string/bool | none | Default value. Boolean flags use `#true`/`#false`. |
| `env` | string | none | Environment variable backing the flag |
| `config` | string | none | Config key (e.g. `"ui.color"`) backing the flag |
| `global` | boolean | `#false` | Available on all subcommands |
| `count` | boolean | `#false` | Value = number of times used (e.g., `-vvv` = 3) |
| `var` | boolean | `#false` | Flag repeatable, collecting values |
| `var_min` | integer | none | Min values when `var=#true` |
| `var_max` | integer | none | Max values when `var=#true` |
| `negate` | string | none | Negative form (e.g., `"--no-color"`). Sets env var to `false`. |
| `choices` | block | none | Restrict values to enumerated set |
| `hide` | boolean | `#false` | Exclude from docs/completions |
| `required_if` | string | none | Required if named flag is set |
| `required_unless` | string | none | Required unless named flag is present |
| `overrides` | string | none | If set, named flag's value is ignored |

**Examples:**

```
flag "-v --verbose" help="Enable verbose output"
flag "-f --force" help="Skip confirmation prompts"
flag "--port <port>" default="8080" help="Server port"
flag "--color" negate="--no-color" default=#true help="Enable colors"
flag "-d --debug" count=#true help="Debug level (-ddd for max)"
flag "--include <pattern>" var=#true help="Include patterns (repeatable)"
flag "--include... <pattern>" help="Ellipsis notation for variadic"
flag "--shell <shell>" { choices "bash" "zsh" "fish" }
flag "--env <env>" { choices env="DEPLOY_ENVS" }
flag "--file <file>" required_if="--dir" help="Required if --dir is set"
flag "--file <file>" required_unless="--stdin"
flag "--file <file>" overrides="--stdin"
flag "--color" env="MYCLI_COLOR" help="Backed by env var"
flag "--color" config="ui.color" help="Backed by config key"
```

**Count flags:** `-vvv` sets `$usage_verbose` to `3`. Short flags chain: `-abc` = `-a -b -c`.

**Negate flags:** `flag "--color" negate="--no-color" default=#true` — `--no-color` sets `$usage_color` to `false`.

### Custom Completions (`complete`)

Provide **dynamic tab-completion** for arguments. Preferred over `choices` when values change.

```
complete "<arg_name>" run="<shell command outputting one value per line>"
```

**Key rules:**
- Must appear **after** the `arg` it applies to
- Names must match exactly
- Do **not** combine with `choices` on the same `arg`

**Examples:**

```toml
usage = '''
arg "<service>" help="Service name"
complete "service" run="ls -d infrastructure/*/application 2>/dev/null | sed 's|infrastructure/||;s|/application||'"
arg "<environment>" help="Target environment"
complete "environment" run="ls infrastructure/${usage_service}/application/env/ 2>/dev/null | sed 's/.tfvars//'"
'''
```

**With descriptions:**
```
complete "plugin" run="mise plugins ls" descriptions=#true
```
Output format: `value:description` per line (colons within descriptions escaped with backslash).

**Tera template variables in `run`:**
- `words` — array of all prompt words; access via `words[index]`
- `CURRENT` — index of word being typed
- `PREV` — index of previous word (`CURRENT-1`)

```
complete "controller" run="ls modules/{{words[PREV]}}/controllers"
```

**`choices` vs `complete`:**

| Feature | `choices` | `complete` |
|---------|-----------|------------|
| Values | Hardcoded in spec | Dynamic from command |
| Validation | Rejects invalid input | Tab-completion only |
| Maintenance | Must edit to add options | Auto-discovers new options |
| Best for | Stable enums (yes/no, log levels) | File/directory-derived values |

### Accessing Arguments in Scripts

**Environment variables** (prefixed with `usage_`):

| Definition | Environment Variable |
|------------|---------------------|
| `arg "<file>"` | `$usage_file` |
| `arg "[output]"` | `$usage_output` |
| `flag "-v --verbose"` | `$usage_verbose` |
| `flag "--dry-run"` | `$usage_dry_run` |
| `flag "-o --output <file>"` | `$usage_output` |

**Naming rules:** `usage_` prefix, hyphens → underscores, lowercase, long name used.

**Bash variable expansion patterns:**

| Pattern | Use Case |
|---------|----------|
| `${var?}` | Required args — fail if missing |
| `${var:-default}` | Optional with fallback |
| `${var:?msg}` | Required with custom error |
| `${var:+value}` | Conditional (if set, use value) |

**Values by type:**

| Type | Present | Absent |
|------|---------|--------|
| Boolean flag | `"true"` | unset (or `"false"` with `default=#false`) |
| Count flag | `"1"`, `"2"`, etc. | `"0"` |
| Value flag/arg | the string value | unset or default |
| Variadic | shell-escaped space-separated | unset or empty |

**Critical distinction:** `default=""` makes variable **SET** to empty string. No default + not provided = **UNSET**.

```bash
# Required arg — error if not provided
echo "Deploying to ${usage_environment?}"

# Optional with default
clean="${usage_clean:-false}"

# Conditional flag forwarding
docker build ${usage_verbose:+--verbose} .

# Boolean flag check
if [ -n "${usage_dry_run:-}" ]; then
  echo "Dry run mode"
fi

# Variadic to array
eval "files=($usage_files)"
```

**Precedence:** CLI args > env vars > defaults.

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

**Supported shebangs:** `bash`, `node`, `python`, `deno`, `pwsh`, `fish`, `zsh`, `ruby`.

Non-hash comment languages (Node/JS/TypeScript/Deno) use `//MISE` and `//USAGE`.

---

## Task Configuration Reference

### All Task Fields

#### Core Execution

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `run` | `string \| string[] \| ({task: string, args?: string[], env?: object} \| {tasks: string[]} \| string)[]` | — | Command(s) to execute. Supports structured array (see below). |
| `run_windows` | same as `run` | — | Windows-specific override |
| `file` | `string` | — | External script path (local, HTTP, or Git URL) |
| `shell` | `string` | `sh -c -o errexit` (Unix), `cmd /c` (Windows) | Interpreter. TOML-tasks only. E.g., `"bash -c"`, `"node -e"`. |
| `usage` | `string` | — | Usage spec for arguments/flags. TOML-tasks only. |

#### Metadata

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `description` | `string` | — | Task help text |
| `alias` | `string \| string[]` | — | Alternative name(s) |
| `hide` | `bool` | `false` | Exclude from listings |

#### Dependencies

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `depends` | `string \| string[] \| object[]` | — | Tasks to run BEFORE (supports structured objects with args/env). Parallel by default. |
| `depends_post` | same as `depends` | — | Tasks to run AFTER this task and its deps complete |
| `wait_for` | same as `depends` | — | Wait for tasks without adding as deps (optional coordination) |

#### Environment & Tools

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `env` | `table` | — | Task-specific env vars (**NOT passed to depends**). Supports sops/age-encrypted values and `_.file` directive. |
| `tools` | `table` | — | Tools to install before running |

#### Execution Context

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dir` | `string` | `{{config_root}}` | Working directory. Use `{{cwd}}` for user's cwd. |
| `raw` | `bool` | `false` | Direct stdin/stdout connection (disables parallel execution; bypasses redactions) |
| `raw_args` | `bool` | `false` | Pass all args verbatim including `--help`/`-h` to underlying command |
| `interactive` | `bool` | `false` | Exclusive lock on stdin/stdout/stderr; only one interactive task at a time |

#### Output Control

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `quiet` | `bool` | `false` | Suppress mise's command-display output |
| `silent` | `bool \| "stdout" \| "stderr"` | `false` | Suppress all/specific output |

#### Caching

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sources` | `string \| string[]` | — | Input files (glob patterns) |
| `outputs` | `string \| string[] \| {auto: true}` | `{auto: true}` (if `sources` set) | Generated files. `{auto = true}` writes to `~/.local/state/mise/task-outputs/<hash>`. |

#### Safety

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `confirm` | `string \| {message: string, default: string}` | — | Prompt before running. Supports Tera templates with `usage.*`. **Guards only `run`, not `depends`.** |

#### Timeout & Inheritance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `timeout` | `string` | — | Max execution duration (e.g., `"30s"`, `"5m"`). **Shorter of per-task and global `task.timeout` wins.** |
| `extends` | `string` | — | Template task to inherit configuration from |
| `vars` | `table` | — | Task-local vars that override `[vars]` |
| `redactions` | `string[]` | — | **Experimental.** Env var names (globs allowed, e.g. `SECRETS_*`) redacted from output as `[redacted]` |

### Structured `run` Array

Mix inline scripts with task references, including parallel sub-task execution:

```toml
[tasks.pipeline]
run = [
  { task = "lint" },                 # run lint (with its dependencies)
  { task = "build", args = ["--release"], env = { RUSTFLAGS = "-C opt-level=3" } },
  { tasks = ["test", "typecheck"] }, # run test and typecheck in parallel
  "echo 'All checks passed!'",       # then run a script
]
```

### Structured `depends` with Args/Env

Pass arguments and environment variables to dependencies:

```toml
[tasks.deploy]
depends = [
  { task = "build", args = ["--release"], env = { RUSTFLAGS = "-C opt-level=3" } }
]
run = "./deploy.sh"
```

Shell-style inline env form also supported:
```toml
depends = ["NODE_ENV=test setup"]
```

Forward parent args via Tera `usage.*`:
```toml
[tasks.deploy]
usage = 'arg "<app>"'
depends = [{ task = "build", args = ["{{usage.app}}"] }]
run = 'echo "deploying {{usage.app}}"'
```

**`wait_for` matching:** by name alone matches regardless of args/env; with explicit args/env must match exactly.

Env vars passed to dependencies are scoped to that dependency only.

### Task Inheritance (`extends`)

A task can inherit configuration from another task:

```toml
[tasks._base_deploy]
hide = true
depends = ["build", "test"]
env = { DEPLOY_TIMESTAMP = "{{now()}}" }
tools = { node = "22" }

[tasks.deploy-staging]
extends = "_base_deploy"
env = { DEPLOY_ENV = "staging" }
run = "./scripts/deploy.sh staging"

[tasks.deploy-prod]
extends = "_base_deploy"
confirm = "Deploy to production?"
env = { DEPLOY_ENV = "production" }
run = "./scripts/deploy.sh production"
```

### Global Task Configuration

#### [task_config] Section

```toml
[task_config]
dir = "{{cwd}}"  # Default working directory for all tasks
includes = [
  "tasks/*.toml",                              # Local task files
  ".mise/tasks/",                              # Task directory
  "git::https://github.com/org/tasks?ref=v1"   # Remote tasks (experimental)
]
```

Setting `includes` **replaces** default search paths — list defaults explicitly to keep them:
```toml
includes = [
  "mise-tasks", ".mise-tasks", ".mise/tasks",
  ".config/mise/tasks", "mise/tasks",
  "mytasks", "tasks.toml",
]
```

Included task-file format (short form, not full mise.toml):
```toml
task1 = "echo task1"
task2 = "echo task2"

[task4]
run = "echo task4"
vars = { target = "linux" }
```

#### [vars] Section

Shared variables between tasks (NOT environment variables):

```toml
[vars]
project_name = "myapp"
version = "1.0.0"
registry = "ghcr.io/myorg"

[vars._.file]
path = "vars.toml"  # Load vars from external file

[tasks.build]
run = "echo Building {{vars.project_name}} v{{vars.version}}"
```

Vars accessed via `{{vars.key_name}}` Tera templates. **Scope precedence:** global (`~/.config/mise/config.toml`) < project `mise.toml` < `mise.local.toml` < task-local `vars`. Currently TOML-tasks only.

#### Global Task Settings

| Setting | Type | Default | Env Var | Description |
|---------|------|---------|---------|-------------|
| `task.output` | string | none | `MISE_TASK_OUTPUT` | Output mode: prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `task.timeout` | string | none | `MISE_TASK_TIMEOUT` | Default timeout. Per-task cannot exceed this. |
| `task.timings` | bool | none | `MISE_TASK_TIMINGS` | Show elapsed time per task |
| `task.skip` | string[] | `[]` | `MISE_TASK_SKIP` | Tasks to skip by default |
| `task.skip_depends` | bool | `false` | `MISE_TASK_SKIP_DEPENDS` | Skip dependencies |
| `task.run_auto_install` | bool | `true` | `MISE_TASK_RUN_AUTO_INSTALL` | Auto-install missing tools |
| `task.show_full_cmd` | bool | `false` | `MISE_TASK_SHOW_CMD_NO_TRUNC` | Disable command truncation in output |
| `task.disable_paths` | string[] | `[]` | `MISE_TASK_DISABLE_PATHS` | Paths to exclude from task discovery |
| `task.remote_no_cache` | bool | none | `MISE_TASK_REMOTE_NO_CACHE` | Always fetch latest remote tasks |
| `task.source_freshness_hash_contents` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_HASH_CONTENTS` | Use blake3 content hashing instead of mtime |
| `task.source_freshness_equal_mtime_is_fresh` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_EQUAL_MTIME_IS_FRESH` | Equal mtime = fresh |
| `task.disable_spec_from_run_scripts` | bool | `false` | `MISE_TASK_DISABLE_SPEC_FROM_RUN_SCRIPTS` | Exclude template functions from usage spec (early opt-out before 2026.11.0 removal) |
| `task.monorepo_depth` | int | `5` | `MISE_TASK_MONOREPO_DEPTH` | Subdirectory depth for monorepo discovery |
| `task.monorepo_exclude_dirs` | string[] | `[]` | `MISE_TASK_MONOREPO_EXCLUDE_DIRS` | Custom replaces defaults (`node_modules`, `target`, `dist`, `build`) |
| `task.monorepo_respect_gitignore` | bool | `true` | `MISE_TASK_MONOREPO_RESPECT_GITIGNORE` | Honor `.gitignore` in monorepo discovery |
| `jobs` | int | `8` | `MISE_JOBS` | Max concurrent task execution |

---

## Running Tasks (CLI)

### Commands

| Command | Description |
|---------|-------------|
| `mise run <task>` / `mise r <task>` | Execute task |
| `mise <task>` | Shorthand (if no command conflict) |
| `mise tasks` / `mise tasks ls` | List all tasks |
| `mise tasks --hidden` | Include hidden tasks |
| `mise tasks deps` | Show dependency tree |
| `mise tasks deps --dot` | DOT format for Graphviz |
| `mise tasks info <task>` | Show task details |
| `mise tasks add <name> -- <cmd>` | Create task via CLI |
| `mise tasks edit <task>` | Edit/create task in $EDITOR |
| `mise tasks validate` | Validate task definitions |
| `mise watch <task>` / `mise w <task>` | Watch and re-run on file changes (requires watchexec) |

### Execution Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--jobs <N>` | `-j` | Parallel job limit (default: 4) |
| `--force` | `-f` | Ignore source/output caching |
| `--dry-run` | `-n` | Preview without executing |
| `--output MODE` | `-o` | prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `--raw` | `-r` | Direct stdin/stdout/stderr (forces `--jobs=1`, bypasses redactions) |
| `--quiet` | `-q` | Suppress extra output |
| `--silent` | `-S` | Hide all output except errors |
| `--continue-on-error` | `-c` | Continue running tasks even if one fails |
| `--cd <DIR>` | `-C` | Change working directory before execution |
| `--shell SHELL` | `-s` | Shell spec (default: `sh -c -o errexit -o pipefail`) |
| `--tool TOOL@VERSION` | `-t` | Additional tools beyond mise.toml |
| `--timings` | — | Show elapsed time per task |
| `--no-timings` | — | Hide task completion duration |
| `--timeout DURATION` | — | Task timeout (e.g., `30s`, `5m`) |
| `--fresh-env` | — | Bypass environment cache |
| `--skip-deps` | — | Run only specified tasks, skip dependencies |
| `--no-deps` | — | Skip automatic dependency preparation |
| `--skip-tools` | — | Skip installing tools before running tasks |
| `--no-cache` | — | Skip cache on remote tasks |

**Experimental sandbox flags:**

| Flag | Description |
|------|-------------|
| `--allow-env <VAR>` | Allow specific env var through (supports wildcards) |
| `--allow-net <HOST>` | Allow network access to specific host |
| `--allow-read <PATH>` | Allow filesystem reads from specific path |
| `--allow-write <PATH>` | Allow filesystem writes to specific path |
| `--deny-all` | Block reads, writes, network, and env vars |
| `--deny-env` | Block env var inheritance |
| `--deny-net` | Block all network access |
| `--deny-read` | Block filesystem reads |
| `--deny-write` | Block all filesystem writes |

### Output Modes

- `prefix` — Default when `jobs > 1`; each line prefixed with task name
- `interleave` — Default when `jobs == 1`; print as output arrives
- `keep-order` — Stream one task's output live; buffer others; print in definition order
- `replacing` — Replace stdout on each new line (similar to `mise install`)
- `timed` — Show only stdout lines that took >1s
- `quiet` — Print only task stdout/stderr, nothing from mise itself
- `silent` — Print nothing from tasks or mise

### Parallel Tasks and Wildcards

```bash
mise run test:*         # All test:* tasks
mise run lint:**        # All nested lint tasks
mise run {build,test}   # Multiple specific tasks
mise run lint ::: test ::: check  # Parallel task groups with :::
mise run cmd1 arg1 ::: cmd2 arg2  # Parallel with separate args
```

### Default Task

```toml
[tasks.default]
depends = ["build", "test"]
run = "echo 'Ready!'"
```

---

## Task Dependencies and Caching

### Dependencies

```toml
[tasks.deploy]
depends = ["build", "test", "lint"]  # Run before (parallel by default)
depends_post = ["notify"]             # Run after
wait_for = ["db:migrate"]             # Wait if running, don't add
```

If a dependency fails, the dependent task skips execution. Shared dependencies run once.

### Caching with sources/outputs

```toml
[tasks.build]
sources = ["Cargo.toml", "src/**/*.rs"]
outputs = ["target/release/myapp"]
run = "cargo build --release"
# Skips if sources unchanged and outputs exist
```

**Auto outputs:**
```toml
[tasks.build]
sources = ["src/**/*.rs"]
outputs = { auto = true }  # Implicit tracking via task hash
run = "cargo build"
```

Enable content-hash checking (more accurate, slower) via `task.source_freshness_hash_contents = true`.

### Redactions (Experimental)

Hide sensitive values from task output:

```toml
redactions = ["API_KEY", "PASSWORD", "SECRETS_*"]
```

Redactions intercept task output line-by-line; tasks with `raw = true` bypass them.

**CI integration (GitHub Actions):**
```bash
for value in $(mise env --redacted --values); do
  echo "::add-mask::$value"
done
```

`jdx/mise-action@v3` handles this automatically.

### Error Handling

Tasks run with `set -e` by default. Disable locally:
```toml
run = '''
set +e
cd /nonexistent
echo "This will not fail the task"
'''
```

---

## Dev Tools Management

### Backends Overview

mise supports 18+ backend types. Recommended priority order (per official registry):

| Priority | Backend | Description |
|----------|---------|-------------|
| 1 | **aqua** | Most features, best security (cosign/SLSA/attestation/minisign). No plugins needed. |
| 2 | **github** | GitHub releases with auto OS/arch detection |
| 3 | **gitlab** | GitLab releases |
| 4 | **forgejo** | Forgejo/Codeberg releases |
| 5 | **http** | Direct HTTP/HTTPS downloads with URL templating |
| 6 | **s3** | S3/MinIO (experimental) |
| 7 | **pipx** | Python CLIs in isolated environments (uses uvx by default) |
| 8 | **npm** | Node packages (auto-detects `aube`, `bun`, `pnpm`, `npm`) |
| 9 | **go** | Go packages (requires compilation) |
| 10 | **cargo** | Rust packages (uses binstall by default) |
| 11 | **gem** | Ruby gems |
| 12 | **ubi** | Universal Binary Installer (**DEPRECATED** — migrate to `github`) |
| 13 | **dotnet** | .NET tools (experimental) |
| 14 | **conda** | Conda packages (experimental) |
| 15 | **spm** | Swift packages (experimental) |
| 16 | **vfox** | vfox plugins (cross-platform, Windows-supported) |
| 17 | **asdf** | asdf plugins (legacy, no Windows) |

> New vfox/asdf tools are rarely accepted into the registry for supply-chain reasons.

**Registry override** per tool:
```bash
export MISE_BACKENDS_PHP='vfox:mise-plugins/vfox-php'
mise install php@latest
```

**Disable backends:**
```bash
mise settings disable_backends=asdf
```

**Discovery:**
```bash
mise registry          # list everything
mise use               # interactive selector
mise search <query>    # find tools in registry
```

### TOML Syntax for Tools

```toml
[tools]
# Simple version
node = "22"
python = "3.12"
ruby = "latest"

# Multiple versions
python = ["3.12", "3.11"]

# With options
node = { version = "22", postinstall = "corepack enable" }
python = { version = "3.11", os = ["linux", "macos"] }
ripgrep = { version = "latest", os = ["linux", "macos"] }
hk = { version = "latest", os = ["linux", "macos/arm64"] }  # OS/arch combos

# Explicit backend
"aqua:BurntSushi/ripgrep" = "latest"
"github:cli/cli" = "latest"
"npm:prettier" = "latest"
"pipx:psf/black" = "latest"
"cargo:eza" = "latest"
"go:github.com/DarthSim/hivemind" = "latest"
```

**Version formats:**

| Format | Example | Description |
|--------|---------|-------------|
| Exact | `"20.0.0"` | Specific version |
| Prefix | `"20"` | Latest matching prefix |
| Latest | `"latest"` | Most recent stable |
| `ref:<SHA>` | `"ref:abc123"` | Compile from git ref |
| `path:<PATH>` | `"path:/opt/node"` | Use custom binary |
| `sub-<N>:<BASE>` | `"sub-1:latest"` | N versions behind base |
| `tag:<TAG>` | `"tag:v1.0.0"` | Cargo git tag |
| `branch:<BRANCH>` | `"branch:main"` | Cargo git branch |
| `rev:<SHA>` | `"rev:abc1234"` | Cargo git rev |

### Per-Tool Options

Universal options supported by every backend:

| Option | Type | Description |
|--------|------|-------------|
| `version` | string | Tool version |
| `os` | string[] | Restrict to OS/arch: `"linux"`, `"macos"`, `"darwin"`, `"windows"`, `"win"`. Combos: `"linux/x64"`, `"macos/arm64"`. Arches: `arm64`/`aarch64`, `x64`/`x86_64`/`amd64`. |
| `install_env` | table | Environment variables injected during install |
| `postinstall` | string | Command after install. `MISE_TOOL_INSTALL_PATH` available. |
| `depends` | string \| string[] | Tool installation dependencies |

Example with dependencies:
```toml
[tools]
python = "3.12.11"
"pipx:ruff" = { version = "latest", depends = ["python"] }
"cargo:usage-cli" = {
    version = "latest",
    os = ["linux", "macos"],
    install_env = { RUST_BACKTRACE = "1" }
}
```

### Backend-Specific Configuration

#### Aqua Backend (Preferred)

```toml
[tools]
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:cli/cli" = { version = "latest", symlink_bins = true }  # Filtered .mise-bins directory
"aqua:flutter/flutter" = { version = "3.32.8", channel = "stable" }
"aqua:scenarigo/scenarigo" = { version = "0.21.0", vars = { go_version = "1.24" } }
```

**Tool options:** `symlink_bins` (bool), `vars` (table, aqua registry template variables), `channel`.

**Security (all default `true`):**

| Setting | Description |
|---------|-------------|
| `aqua.cosign` | Verify cosign signatures |
| `aqua.slsa` | Verify SLSA provenance |
| `aqua.github_attestations` | Verify GitHub Artifact Attestations |
| `aqua.minisign` | Verify minisign signatures |
| `aqua.baked_registry` | Use built-in aqua registry |
| `aqua.registry_url` | Custom aqua registry URL |
| `aqua.cosign_extra_args` | Additional cosign arguments |

Aqua verification is native (no external CLIs needed) covering: GitHub attestations, cosign, SLSA, minisign, and SHA256/512/1/MD5.

**Limitation:** Aqua tools can't set env vars or do more than download binaries.

#### GitHub Backend

```toml
[tools]
"github:cli/cli" = "latest"

[tools."github:cli/cli"]
version = "latest"
asset_pattern = "gh_*_linux_x64.tar.gz"
bin = "gh"
filter_bins = "gh"
no_app = true
bin_path = "cli-{{ version }}/bin"
rename_exe = "gh"
version_prefix = "release-"
checksum = "sha256:..."
size = "12345678"
api_url = "https://github.mycompany.com/api/v3"
strip_components = 1

[tools."github:cli/cli".platforms]
linux-x64 = { asset_pattern = "gh_*_linux_x64.tar.gz" }
macos-arm64 = { asset_pattern = "gh_*_macOS_arm64.tar.gz" }
```

**Tool options:** `asset_pattern`, `version_prefix`, `platforms` (per-platform `asset_pattern`/`checksum`/`size`), `strip_components`, `bin`, `rename_exe`, `bin_path` (Tera templating `{{version}}`/`{{os}}`/`{{arch}}`), `filter_bins`, `checksum`, `size`, `no_app`, `api_url`.

**Auth settings:** `github.credential_command`, `github.gh_cli_tokens` (default `true`), `github.github_attestations`, `github.slsa`, `github.use_git_credentials`. Env: `MISE_GITHUB_TOKEN`, `MISE_GITHUB_USE_GIT_CREDENTIALS`.

#### GitLab / Forgejo Backends

Same option surface as GitHub (`asset_pattern`, `platforms`, `bin`, `bin_path`, `rename_exe`, `filter_bins`, `checksum`, `size`, `strip_components`, `api_url`).

```toml
"gitlab:gitlab-org/gitlab-runner" = "16.8.0"
"forgejo:user/repo" = "latest"  # Defaults to Codeberg
```

**Forgejo auth order:** `MISE_FORGEJO_ENTERPRISE_TOKEN` → `MISE_FORGEJO_TOKEN` → `FORGEJO_TOKEN` → credential_command → forgejo_tokens.toml → `fj` CLI → git credential fill.

#### HTTP Backend

```toml
[tools."http:my-tool"]
version = "1.0.0"
url = "https://example.com/releases/my-tool-v{{version}}.tar.gz"
bin_path = "bin"
format = "tar.gz"  # Explicit override
strip_components = 1

[tools."http:my-tool".platforms]
macos-x64 = { url = "https://example.com/tool-macos-x64.tar.gz", checksum = "sha256:..." }
linux-x64 = { url = "https://example.com/tool-linux-x64.tar.gz", checksum = "sha256:..." }
```

**URL template variables:** `{{ version }}`, `{{ os() }}`, `{{ arch() }}`, `{{ os_family() }}`. Remapping: `{{ os(macos="darwin") }}`, `{{ arch(x64="amd64") }}`.

**Platform keys:** `macos-x64`, `macos-arm64`, `linux-x64`, `linux-arm64`, `windows-x64`, `windows-arm64`.

**Version discovery options:**
- `version_list_url` — URL returning version list (text, JSON array, or JSON objects)
- `version_regex` — Regex to extract versions (first capturing group)
- `version_json_path` — jq-like path (e.g., `".[].tag_name"`)
- `version_expr` — expr-lang expression with access to response `body`

**Caching:** `$MISE_CACHE_DIR/http-tarballs/` (Blake3 keys); auto-pruned after 30 days.

#### S3 Backend (Experimental)

Requires `experimental = true`.

```toml
[tools."s3:my-internal-tool"]
version = "latest"
url = "s3://tools-bucket/releases/my-tool-{{ version }}.tar.gz"
endpoint = "http://minio.internal:9000"  # MinIO / S3-compat
region = "us-east-1"
version_list_url = "s3://tools-bucket/releases/versions.json"
bin_path = "bin"
```

**Options:** `url`, `endpoint`, `region`, `checksum`, `bin_path`, `format`, `platforms`, `version_list_url`, `version_json_path`, `version_expr`, `version_prefix`, `version_regex`.

**Auth:** AWS SDK default chain (env vars → `~/.aws/credentials` → IAM roles).

#### Cargo Backend

```toml
[tools]
"cargo:eza" = "latest"
"cargo:cargo-edit" = { version = "latest", features = "add" }
"cargo:demo" = { version = "latest", default-features = false }
"cargo:demo" = { version = "latest", bin = "demo", crate = "demo", locked = false }
```

**Git-based installation:**
```bash
mise use cargo:https://github.com/user/repo@tag:v1.0
mise use cargo:https://github.com/user/repo@branch:main
mise use cargo:https://github.com/user/repo@rev:abc123
```

**Settings:** `cargo.binstall` (default `true`), `cargo.binstall_only` (default `false`), `cargo.registry_name`.

#### Pipx Backend

```toml
[tools]
"pipx:black" = "latest"
"pipx:harlequin" = { version = "latest", extras = "postgres,s3" }
"pipx:ansible" = { version = "latest", uvx = false }  # Disable uv
"pipx:ansible-core" = { version = "latest", uvx_args = "--with ansible" }
"pipx:black" = { version = "latest", pipx_args = "--preinstall" }
```

**Tool options:** `extras`, `pipx_args`, `uvx` (per-tool toggle), `uvx_args`.

**Settings:** `pipx.uvx` (default `true`), `pipx.registry_url`.

**Install sources:** PyPI, GitHub (`pipx:psf/black`), Git (`pipx:git+https://...`), HTTP ZIP.

Reinstall after Python upgrade: `mise install -f pipx:psf/black`

#### NPM Backend

```toml
[tools]
"npm:prettier" = "latest"
```

**Auto-detects package manager:** prefers `aube` if present, else `npm`, with `bun`/`pnpm` as alternatives. Override with `npm.package_manager = "auto|npm|aube|bun|pnpm"` (env: `MISE_NPM_PACKAGE_MANAGER`).

Supports minimum-release-age protection (transitive supply chain) via compatible package managers (`npm >= 11.10.0`, `aube`, `bun >= 1.3.0`, `pnpm >= 10.16.0`).

#### Go Backend

```toml
[tools]
"go:github.com/DarthSim/hivemind" = "latest"
"go:github.com/golang-migrate/migrate/v4/cmd/migrate" = { version = "latest", tags = "postgres" }
```

#### Gem Backend

```toml
[tools]
"gem:rubocop" = "latest"
```

Reinstall after Ruby upgrade: `mise install -f gem:rubocop` or `mise install -f "gem:*"`.

#### Conda Backend (Experimental)

```toml
[tools]
"conda:ruff" = "latest"
"conda:bioconductor-deseq2" = { version = "latest", channel = "bioconda" }
```

Direct anaconda.org API — no conda/mamba required. **Single packages only** (not full environments).

#### SPM Backend (Experimental)

```toml
[tools]
"spm:tuist/tuist" = "latest"
"spm:swiftlang/swiftly" = { version = "latest", filter_bins = ["swiftly"] }
```

#### Dotnet Backend (Experimental)

```toml
[tools]
"dotnet:GitVersion.Tool" = "5.12.0"
```

**Settings:** `dotnet.registry_url`, `dotnet.package_flags` (supports `prerelease`), `dotnet.isolated`, `dotnet.cli_telemetry_optout`, `dotnet.dotnet_root`.

#### vfox & asdf Plugins

**vfox (recommended):** cross-platform (Win/macOS/Linux), HTTP/JSON/HTML modules, attestation verification, lock files.

**asdf (legacy):** Unix-only, bash scripts, no Windows. New asdf tools rarely accepted for supply-chain reasons.

```bash
mise plugin install my-plugin https://github.com/username/my-plugin
mise install my-plugin:some-tool@1.0.0
```

### Shims and Aliases

**Shim location:**
- Linux/macOS: `~/.local/share/mise/shims`
- Windows: `%LOCALAPPDATA%\mise\shims`

Three activation methods:
1. **PATH activation** (`mise activate`) — updates PATH per prompt
2. **Shims** (`mise activate --shims`) — tiny intercepting executables
3. **Explicit** (`mise exec`, `mise run`, `mise en`)

```bash
# Shim mode (non-interactive shells — .bash_profile/.zprofile)
eval "$(mise activate zsh --shims)"

# PATH mode (interactive shells — .bashrc/.zshrc)
eval "$(mise activate zsh)"
```

**Best practice:** Use both — shims in profile for non-interactive, PATH activation in rc for interactive. `mise activate` strips the shims dir from PATH, so the combo is safe.

**Shims vs PATH:**
- **Shims**: env vars only load when shim is called; only `preinstall`/`postinstall` hooks work; `which` points to shim; use `mise which` for real path
- **PATH**: full environment, all hooks (`cd`, `enter`, `leave`, `watch_files`), `which` shows actual binary
- **Recommendation**: PATH for interactive shells; shims for IDEs/non-interactive

**`mise reshim`:** Regenerates shim directory. Runs automatically on install/update/remove. Never manually drop binaries in the shims dir — they get deleted.

**Tool aliases (remap to different backend):**
```toml
[tool_alias]
node = 'asdf:company/our-custom-node'
erlang = 'asdf:https://github.com/company/our-custom-erlang'

[tool_alias.node.versions]
lts = '22'
my_custom_20 = '20'
```

> Note: `[alias]` is deprecated in favor of `[tool_alias]`.

**Template-driven version aliases:**
```toml
[tool_alias.node.versions]
current = "{{exec(command='node --version')}}"
```

**Shell aliases:**
```toml
[shell_alias]
ll = "ls -la"
gs = "git status"
```

### Auto-Install Controls

- `auto_install` (default `true`) — master switch
- `exec_auto_install` (default `true`) — `mise x`/`mise r`
- `task_auto_install` (default `true`)
- `not_found_auto_install` (default `true`) — requires at least one existing version
- `auto_install_disable_tools = ["..."]` — per-tool skip list

### CLI Commands for Tools

```bash
mise use node@22            # Install + activate + write to mise.toml
mise use -g node@22         # Write to global config
mise install node@20        # Install without activation
mise install                # Install all configured tools
mise install -f "gem:*"     # Force reinstall pattern
mise ls                     # List installed
mise ls-remote node         # List available versions
mise which node             # Show real binary path
mise x python@3.12 -- script.py  # Run with specific tool
mise reshim                 # Rebuild shims
mise registry               # List all available tools
mise outdated               # Check for updates
mise upgrade                # Update versions
mise search <query>         # Search registry
mise cache clear            # Clear cached downloads
```

---

## Environment Configuration

### Basic Variables

```toml
[env]
NODE_ENV = 'production'
DEBUG = 'app:*'
PORT = 3000

# Unset a variable
UNWANTED_VAR = false
```

```bash
mise set NODE_ENV=development   # Set via CLI
mise set                        # View all
mise set -E staging NODE_ENV=staging  # Write to profile file
mise set --prompt PASSWORD      # Hidden interactive prompt
cat private.key | mise set --stdin MY_KEY  # From stdin
mise unset NODE_ENV             # Remove
mise env                        # Export all
mise env --json                 # Export as JSON
mise env --dotenv               # Export as dotenv
mise env --redacted             # Show only redacted variables
mise env --values               # Show only values
mise env -s bash                # Shell-specific output
```

### Special Directives (`env._`)

The reserved key `_` is used as a TOML table for configuration since nested env vars don't make sense.

#### `_.path` — Prepend to PATH

```toml
[env]
_.path = ["tools/bin", "{{config_root}}/scripts"]
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }  # Lazy eval after tools
```

Relative paths resolve against `{{config_root}}`.

#### `_.file` — Load from .env/json/yaml/toml files

```toml
[env]
_.file = '.env'
_.file = ['.env', '.env.local', '.env.{{env.MISE_ENV}}']
_.file = { path = ".secrets.yaml", redact = true }
```

Supported formats: `.env`, `.env.json`, `.env.yaml`, `.env.toml`. Auto-load: `MISE_ENV_FILE=.env`.

Options: `path` (string), `redact` (boolean), `tools` (boolean — resolve after tools).

#### `_.source` — Source shell scripts

```toml
[env]
_.source = "./setup-env.sh"
_.source = { path = "my/env.sh", redact = true }
_.source = { path = "my/env.sh", tools = true }
```

Scripts execute with `source` semantics; shebangs are ignored.

#### Multiple Identical Directives

TOML doesn't allow duplicate keys, so use array-of-tables:

```toml
[[env]]
_.source = "./script_1.sh"
[[env]]
_.source = "./script_2.sh"
```

#### Lazy Evaluation (`tools = true`)

Resolve variables after tools configure their environment:

```toml
[env]
NODE_VERSION = { value = "{{ tools.node.version }}", tools = true }
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }
```

### Profiles (`MISE_ENV`)

Profiles enable environment-specific config files. Three ways to set `MISE_ENV`:
1. CLI: `-E development` / `--env development`
2. Env var: `MISE_ENV=development`
3. `.miserc.toml`: `env = ["development"]`

**`MISE_ENV` cannot be set in `mise.toml`** — it must be known before mise.toml is discovered.

```bash
MISE_ENV=staging mise run deploy
```

This loads `mise.staging.toml` in addition to `mise.toml`. Config file **precedence** (highest first):

1. `mise.{MISE_ENV}.local.toml`
2. `mise.local.toml`
3. `mise.{MISE_ENV}.toml`
4. `mise.toml`

Comma-separated supports multiple profiles: `MISE_ENV=ci,test` (rightmost wins).

`.miserc.toml` has limited Tera context — only OS-level (`env.*`, `cwd`, `xdg_*`, `arch()`, `os()`); `mise_env`, `exec()`, `read_file()` are not available.

### Required and Redacted Variables

```toml
[env]
# Required — error if not set (warns during shell activation)
DATABASE_URL = { required = true }
DATABASE_URL = { required = "Set postgres connection string" }

# Redacted — hidden from output
API_KEY = { value = "secret_key_here", redact = true }

# Pattern-based redactions (top-level, not under [env])
redactions = ["*_TOKEN", "SECRET_*", "API_*"]
```

### Secrets (SOPS and age)

**SOPS (experimental, age-backed):**

```bash
mise use -g sops age
age-keygen -o ~/.config/mise/age.txt
sops encrypt -i --age "<public key>" .env.json
```

```toml
[env]
_.file = ".env.json"
# or with redaction
_.file = { path = ".env.json", redact = true }
```

**Key resolution:** `MISE_SOPS_AGE_KEY` → `MISE_SOPS_AGE_KEY_FILE` → `SOPS_AGE_KEY_FILE` → `SOPS_AGE_KEY` → `~/.config/mise/age.txt`.

**Settings:** `sops.age_key`, `sops.age_key_file` (default `~/.config/mise/age.txt`), `sops.age_recipients`, `sops.rops` (default `true`, native Rust impl), `sops.strict` (default `true`).

**Direct age encryption (experimental):**

```bash
mise set --age-encrypt DB_PASSWORD=supersecret
mise set --age-encrypt --prompt DB_PASSWORD
```

Flags: `--age-encrypt`, `--age-recipient <KEY>`, `--age-ssh-recipient <PATH|KEY>`, `--age-key-file <PATH>`.

Stored in mise.toml:
```toml
[env]
DB_PASSWORD = { age = { value = "<base64>" } }
```

**Settings:** `age.identity_files`, `age.key_file`, `age.ssh_identity_files`, `age.strict` (default `true`).

Decrypted values are automatically marked for redaction.

### Templates (Tera)

mise.toml values support Tera templates:

```toml
[env]
PROJECT_DIR = "{{config_root}}"
LOG_FILE = "{{config_root}}/logs/{{now() | date(format='%Y-%m-%d')}}.log"
NODE_PATH = "{{env.npm_config_prefix}}/lib/node_modules"
PROJECT_NAME = "{{ cwd | basename }}"
```

**Available template variables:**

| Variable | Type | Description |
|----------|------|-------------|
| `env.*` | HashMap | Current environment variables |
| `config_root` | PathBuf | Directory containing mise.toml |
| `cwd` | PathBuf | Current working directory |
| `mise_bin` | String | Path to mise binary |
| `mise_pid` | String | Process ID |
| `mise_env` | Vec | Configuration environments from `MISE_ENV` |
| `tools` | HashMap | Installed tool info (`.version`, `.path`). Requires `tools = true` on task templates. |
| `usage` | HashMap | Task arguments/flags (task run scripts only). Hyphenated: `usage["dry-run"]`. |
| `xdg_cache_home` | PathBuf | XDG cache directory |
| `xdg_config_home` | PathBuf | XDG config directory |
| `xdg_data_home` | PathBuf | XDG data directory |
| `xdg_state_home` | PathBuf | XDG state directory |

**Key Tera functions:**

| Function | Description |
|----------|-------------|
| `exec(command, [cache_key], [cache_duration])` | Execute shell command, return stdout |
| `arch()` | System architecture (e.g., `x64`, `arm64`) |
| `os()` | Operating system (linux, macos, windows) |
| `os_family()` | OS family (unix/windows) |
| `num_cpus()` | CPU count |
| `now([timestamp], [utc])` | Current datetime |
| `get_env(name, [default])` | Get env var (throws if missing, no default) |
| `choice(n, alphabet)` | Random n-char string |
| `read_file(path)` | Read file contents |
| `task_source_files()` | Resolved source file paths (task scripts only) |
| `range(end, [start], [step_by])` | Integer array |
| `get_random(end, [start])` | Random integer |
| `throw(message)` | Raise error |

**Key Tera filters:**

| Filter | Description |
|--------|-------------|
| `lower`, `upper`, `capitalize`, `title` | Case transforms |
| `kebabcase`, `snakecase`, `shoutysnakecase`, `lowercamelcase`, `uppercamelcase` | Case conversion |
| `trim`, `trim_start`, `trim_end` | Whitespace removal |
| `replace(from, to)` | String substitution |
| `quote` | Escape and quote string |
| `split(pat)`, `join(sep)` | Array operations |
| `first`, `last`, `length`, `reverse` | Collection operations |
| `concat(with)` | Append values to array |
| `map(attribute)` | Extract attribute from objects |
| `basename`, `dirname`, `extname`, `file_stem` | Path operations |
| `file_size`, `last_modified` | File metadata |
| `absolute`, `canonicalize` | Path resolution (canonicalize throws if missing) |
| `join_path` | Join array of path segments |
| `hash([algorithm], [len])` | SHA256 (default) or BLAKE3 hashing |
| `hash_file([len])` | File BLAKE3 hash |
| `date(format)` | Format datetime (chrono strftime) |
| `default(value)` | Fallback for undefined/empty |
| `abs` | Absolute value |
| `filesizeformat` | Human-readable file size |
| `urlencode` | URL-safe encoding |
| `truncate([length])` | Shorten string |

**Tera tests:**

| Test | Description |
|------|-------------|
| `defined` | Variable exists |
| `string` | Is string type |
| `number` | Is numeric type |
| `starting_with(arg)` | Starts with text |
| `ending_with(arg)` | Ends with text |
| `containing(arg)` | Contains text |
| `matching(regex)` | Matches regex |
| `dir` | Path is directory |
| `file` | Path is file |
| `exists` | Path exists |
| `odd`, `even` | Numeric parity |

**Template syntax:**
- `{{ }}` — Expressions
- `{% %}` — Statements
- `{# #}` — Comments
- `{% raw %} {% endraw %}` — Skip rendering
- Operators: `+`, `-`, `/`, `*`, `%`, `==`, `!=`, `>=`, `<=`, `and`, `or`, `not`, `~` (concat), `in`

**Shell-style variable expansion** (requires `env_shell_expand = true`):
```toml
[settings]
env_shell_expand = true

[env]
LD_LIBRARY_PATH = "$MY_LIB:$LD_LIBRARY_PATH"
PATH_SAFE = "${VAR:-default}"      # With default
CLEAN = "${UNSET_VAR:-}"           # Empty if unset (no warning)
```

---

## Hooks and Watchers

### Hook Types (Experimental)

| Hook | Trigger | Requires `mise activate`? |
|------|---------|--------------------------|
| `cd` | Directory changes | Yes |
| `enter` | Enter a project (once per entry) | Yes |
| `leave` | Leave a project (once) | Yes |
| `preinstall` | Before tool installation | No |
| `postinstall` | After tool installation | No |

### Syntax

```toml
[hooks]
cd = "echo 'changed directory'"
enter = "echo 'entered project'"
leave = "echo 'left project'"
preinstall = "echo 'about to install'"
postinstall = "echo 'installed'"

# Multiple hooks
enter = ["echo 'first'", "echo 'second'"]

# Shell hooks (execute in current shell context)
[hooks.enter]
shell = "bash"
script = "source completions.sh"

# Task hooks
[hooks]
enter = { task = "setup" }
enter = ["echo 'entering'", { task = "setup" }]  # Mixed syntax

# Array-of-tables form
[[hooks.cd]]
script = "echo 'I changed directories'"
[[hooks.cd]]
script = "echo 'I also changed directories'"
```

**Important:** Shell hooks don't auto-cleanup on directory exit like `[env]` does. Exported vars, aliases, sourced files persist — manually reverse in a corresponding `leave` hook.

### Watch Files

```toml
[[watch_files]]
patterns = ["src/**/*.rs"]
run = "cargo fmt"

# Task reference
[[watch_files]]
patterns = ["uv.lock"]
task = "sync-deps"
```

Each entry requires **either** `run` or `task`, not both. Sets `MISE_WATCH_FILES_MODIFIED` env var (colon-separated, literal colons backslash-escaped). Requires watchexec (`mise use -g watchexec@latest`).

### Hook Environment Variables

All hooks receive:
- `MISE_ORIGINAL_CWD` — user's working directory at hook fire
- `MISE_PROJECT_ROOT` — detected project root

CD hooks additionally receive:
- `MISE_PREVIOUS_DIR` — previous directory

Preinstall/postinstall hooks receive:
- `MISE_INSTALLED_TOOLS` — JSON array of installed tools
- `MISE_TOOL_NAME` — tool identifier (e.g., `node`)
- `MISE_TOOL_VERSION` — installed version
- `MISE_TOOL_INSTALL_PATH` — installation directory

### Per-Tool postinstall (not a hook)

```toml
[tools]
node = { version = "22", postinstall = "corepack enable" }
python = { version = "3.12", postinstall = "pip install pipx" }
```

---

## Configuration and Settings

### File Hierarchy

Config files in precedence order (highest first):

1. `mise.local.toml` (gitignored)
2. `mise.toml`
3. `mise/config.toml`
4. `.mise/config.toml`
5. `.config/mise.toml`
6. `.config/mise/config.toml`
7. `.config/mise/conf.d/*.toml` (alphabetical)

Any can also appear as dotfiles (`.mise.toml`, etc.).

**Global:** `~/.config/mise/config.toml` (+ `conf.d/` subdirectory)
**System:** `/etc/mise/config.toml` (+ `conf.d/`)
**Legacy:** `.tool-versions` (asdf-compatible)

**With `MISE_ENV`:** `mise.<env>.toml`, `mise.<env>.local.toml` layer on top.

**Schema validation:**
- `https://mise.jdx.dev/schema/mise.json`
- `https://mise.jdx.dev/schema/mise-task.json`

mise searches upward from cwd to root (stops at `MISE_CEILING_PATHS`). Merge behavior:
- **Tools:** Additive with overrides
- **Env vars:** Additive with overrides
- **Tasks:** Completely replaced per task name (closest wins)
- **Settings:** Additive with overrides

**Write targeting:** `mise use`, `mise set`, `mise unuse` write to the lowest-precedence file in the highest-precedence directory.

**Useful commands:**
```bash
mise cfg / mise config     # Show loaded files in precedence order
mise ls --current          # Active versions with overrides
mise doctor                # Diagnose setup issues
```

### Idiomatic Version Files

Disabled by default. Enable per-tool:
```bash
mise settings add idiomatic_version_file_enable_tools python
```

Supported files include `.nvmrc`, `.node-version`, `package.json`, `.python-version`, `.ruby-version`, `Gemfile`, `.go-version`, `rust-toolchain.toml`, `.java-version`, `.sdkmanrc`, `global.json`, `.terraform-version`, `.bun-version`, `.deno-version`.

### Key Settings Reference

```toml
[settings]
# Execution
jobs = 8                    # Concurrent jobs (MISE_JOBS)
experimental = false        # Enable experimental features
yes = false                 # Auto-answer prompts (MISE_YES)

# Task defaults
task.output = "prefix"      # prefix|interleave|keep-order|replacing|timed|quiet|silent
task.timeout = "10m"        # Default task timeout
task.timings = true         # Show elapsed time
task.skip = ["slow-task"]   # Tasks to skip
task.skip_depends = false   # Skip dependencies
task.source_freshness_hash_contents = false  # blake3 content check

# Environment
env_shell_expand = true     # Enable $VAR expansion
env_cache = false           # Cache computed environment
env_cache_ttl = "1h"        # Cache TTL
env_file = ""               # MISE_ENV_FILE

# Tool management
auto_install = true         # Auto-install missing tools
exec_auto_install = true    # Auto-install on mise x/run
not_found_auto_install = true
auto_install_disable_tools = []
disable_backends = ["asdf"] # Disable backends
enable_tools = []           # Allowlist (restricts tools)
disable_tools = []
pin = false                 # Default --pin for mise use
lockfile = true             # Read/update lockfiles
locked = false              # Fail if no pre-resolved URLs

# Security
paranoid = false            # Extra-secure behavior
gpg_verify = false          # Verify GPG signatures
slsa = true                 # SLSA provenance verification
github_attestations = true  # GitHub Artifact Attestations

# Performance
fetch_remote_versions_cache = "1h"  # Version cache
fetch_remote_versions_timeout = "20s"
http_timeout = "30s"                # HTTP timeout
http_retries = 0                    # HTTP retries with exponential backoff

# Network
offline = false             # Block all HTTP requests
prefer_offline = false      # Prefer cached data

# Status line
status.missing_tools = "if_other_versions_installed"
status.show_env = false
status.show_tools = false
status.show_deps_stale = true

# Node-specific
[settings.node]
corepack = false            # Enable corepack
compile = false             # Compile from source
verify = true

# Python-specific
[settings.python]
uv_venv_auto = false        # Auto-create venv with uv (false|source|"create|source"|true)
compile = false             # Compile from source

# Ruby-specific
[settings.ruby]
compile = false             # Compile from source
ruby_install = false        # Use ruby-install instead of ruby-build

# Aqua security
[settings.aqua]
cosign = true
slsa = true
github_attestations = true
minisign = true
baked_registry = true

# Cargo
[settings.cargo]
binstall = true             # Use precompiled binaries
binstall_only = false

# Pipx
[settings.pipx]
uvx = true                  # Use uvx instead of pipx

# NPM
[settings.npm]
package_manager = "auto"    # auto|npm|aube|bun|pnpm

# Conda
[settings.conda]
channel = "conda-forge"

# Age encryption (experimental)
[settings.age]
key_file = "~/.config/mise/age.txt"
strict = true

# Sops encryption
[settings.sops]
rops = true                 # Use native Rust implementation
strict = true               # Fail on decryption errors
age_key_file = "~/.config/mise/age.txt"

# Hook environment
[settings.hook_env]
cache_ttl = "0s"
chpwd_only = false
```

**All settings** support environment variable overrides using `MISE_` prefix (e.g., `MISE_JOBS=4`, `MISE_TASK_OUTPUT=interleave`).

### Minimum Version

```toml
min_version = '2024.11.1'                               # Hard (errors)
min_version = { soft = '2024.11.1' }                    # Soft (warns)
min_version = { hard = '2024.11.1', soft = '2024.9.0' } # Both
```

### Automatic Environment Variables

Tasks automatically receive:

| Variable | Description |
|----------|-------------|
| `MISE_ORIGINAL_CWD` | Original working directory |
| `MISE_CONFIG_ROOT` | Directory containing mise.toml |
| `MISE_PROJECT_ROOT` | Project root directory |
| `MISE_TASK_NAME` | Current task name |
| `MISE_TASK_DIR` | Task script directory |
| `MISE_TASK_FILE` | Full path to task script |

### CI/CD Integration

**GitHub Actions (`jdx/mise-action@v3`):**

```yaml
- uses: jdx/mise-action@v3
  with:
    version: 2024.12.14
    install: true
    cache: true
```

Automatically redacts values flagged with `redact = true` or matching `redactions` patterns.

**GitLab CI:**

```yaml
variables:
  MISE_DATA_DIR: $CI_PROJECT_DIR/.mise/mise-data
cache:
  - key:
      prefix: mise-
      files: ["mise.toml", "mise.lock"]
    paths:
      - $MISE_DATA_DIR
```

**Generic CI bootstrap:**
```bash
curl https://mise.run | sh
mise install
mise x -- <cmd>
```

Or: `mise generate bootstrap -l -w` produces a self-contained install script.

**Useful CI env vars:**
- `MISE_YES=1` — auto-answer prompts
- `MISE_DATA_DIR` — install/cache root
- `MISE_EXPERIMENTAL=1` — unlock experimental features
- `MISE_OFFLINE=1` / `MISE_PREFER_OFFLINE=1` — network policy

### Key Environment Variables

- `MISE_DATA_DIR` (default `~/.local/share/mise`)
- `MISE_CACHE_DIR` (default `~/.cache/mise`)
- `MISE_TMP_DIR` (default system temp)
- `MISE_SYSTEM_CONFIG_DIR` (default `/etc/mise`)
- `MISE_GLOBAL_CONFIG_FILE` (default `~/.config/mise/config.toml`)
- `MISE_GLOBAL_CONFIG_ROOT` (default `$HOME`)
- `MISE_ENV_FILE` (e.g., `.env`)
- `MISE_${PLUGIN}_VERSION` (e.g., `MISE_NODE_VERSION=20`)
- `MISE_TRUSTED_CONFIG_PATHS` / `MISE_CEILING_PATHS` (`:` Unix, `;` Windows)
- `MISE_LOG_LEVEL` (trace|debug|info|warn|error)
- `MISE_QUIET` (= `MISE_LOG_LEVEL=warn`)
- `MISE_HTTP_TIMEOUT` (default 30s)
- `MISE_RAW` (pipes directly; forces `MISE_JOBS=1`)

---

## Prepare Feature (Experimental)

Ensures project dependencies are installed before task execution. Requires `MISE_EXPERIMENTAL=1`.

```bash
mise prepare
```

```toml
[prepare.npm]
auto = true

[prepare.custom]
sources = ["schema/*.graphql"]
outputs = ["src/generated/"]
run = "npm run codegen"
```

Built-in providers: npm, yarn, pnpm, bun, go, pip, poetry, uv, bundler, composer.

Disable auto-run: `mise run --no-prepare <task>`.

---

## Monorepo Tasks (Experimental)

```toml
# Root mise.toml
experimental_monorepo_root = true
```

Requires `MISE_EXPERIMENTAL=1`. Implicit trust for descendants; lazy task loading; subdirs inherit parent tools.

```bash
mise //projects/frontend:build    # Absolute path from root
mise :build                       # Task in current config_root
mise '//projects/frontend:*'      # All tasks in frontend
mise //...:test                   # Test task in all projects
```

Settings:
```toml
[settings.task]
monorepo_depth = 5
monorepo_exclude_dirs = ["dist", "node_modules"]
monorepo_respect_gitignore = true
```

---

## Best Practices

### DO ✅

- **Always use `usage` field** for task arguments
- Use `${var?}` for required args to fail early
- Set `description` for discoverability
- Use `sources`/`outputs` for cacheable tasks
- Use `depends` for task ordering; structured `depends` to pass args/env
- Use `confirm` for destructive operations
- Use `choices` for stable enums, `complete` for dynamic/filesystem-derived values
- Group related tasks with namespaces (e.g., `test:unit`, `test:e2e`)
- Use `mise.local.toml` for personal overrides (gitignored)
- Prefer aqua backend for security (cosign/SLSA/attestation verification)
- Migrate from `ubi:` backend to `github:` (ubi deprecated)
- Use `env._.file` for dotenv loading instead of `MISE_ENV_FILE`
- Redact sensitive values with `redact = true`; use SOPS or direct-age for secrets
- Use templates for dynamic values instead of hardcoding paths
- Use `extends` to share config between similar tasks
- Use shims in `.zprofile`/`.bash_profile` and PATH activation in `.zshrc`/`.bashrc`
- Use `[tool_alias]` (not deprecated `[alias]`)
- Use `jdx/mise-action@v3` in GitHub Actions — it handles masking automatically

### DON'T ❌

- Use `$1`, `$2`, `$@`, `$*` for arguments
- Use `$args` in PowerShell
- Use inline `{{arg(name="x")}}` syntax (deprecated, removed 2026.11)
- Forget to quote glob patterns in sources
- Set env vars in `env` that deps need (they don't inherit — use structured `depends` with `env`)
- Use `raw = true` unless interactive input needed (forces single-threaded, bypasses redactions)
- Set `MISE_ENV` in `mise.toml` (it determines which files to load — use `.miserc.toml`)
- Manually add executables to shims directory (`mise reshim` deletes them)
- Use `MISE_RAW=1` without knowing it sets `MISE_JOBS=1`
- Install new `asdf:` or `vfox:` plugins when aqua/github alternatives exist

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
tools = { node = "22" }
sources = ["dist/**/*"]
timeout = "5m"
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

### Complete Dev Tools + Env Example

```toml
min_version = "2024.11.1"

[settings]
jobs = 8
task.output = "interleave"
task.timings = true
env_shell_expand = true

[tools]
node = "22"
python = { version = "3.12", postinstall = "pip install -r requirements.txt" }
"aqua:BurntSushi/ripgrep" = "latest"
"npm:prettier" = "latest"

[env]
NODE_ENV = "development"
DATABASE_URL = { required = "Set postgres connection string" }
API_KEY = { value = "{{env.API_KEY}}", redact = true }
_.path = ["./node_modules/.bin", "{{config_root}}/scripts"]
_.file = [".env", ".env.local"]

[hooks]
enter = "echo 'Welcome to {{vars.project_name}}'"

[vars]
project_name = "myapp"

[tasks.dev]
description = "Start development server"
depends = ["build"]
tools = { node = "22" }
run = "npm run dev"

[tasks.build]
description = "Build project"
sources = ["src/**/*.ts", "tsconfig.json"]
outputs = ["dist/**/*"]
run = "tsc --build"

[tasks.test]
description = "Run tests"
depends = ["build"]
run = "vitest run"
```
