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
- **Environments** — manage env vars, profiles, dotenv files, secrets, age-encrypted values
- **Hooks** — run commands on directory changes, project enter/leave, tool install

**Key Features:**
- Parallel dependency building (concurrent by default, up to `jobs` setting)
- Last-modified and content-hash checking (avoid unnecessary rebuilds)
- File watching (`mise watch` rebuilds on changes)
- Cross-platform argument handling via `usage` spec
- Hierarchical configuration with profile support
- 18+ tool backends (aqua, github, npm, cargo, pipx, go, etc.)
- Security verification (cosign, SLSA, GitHub Attestations, minisign)

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

Supports: Python, Node, Bun, Deno, Ruby, Bash. Use `-S` flag for multiple interpreter arguments.

### File-Based Tasks (executable scripts)

Place in task directories. Files **must** be executable (`chmod +x`).

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

**Non-bash languages** use language-appropriate comment syntax:
```javascript
#!/usr/bin/env node
//USAGE flag "-v --verbose" help="Enable verbose output"
//MISE description="Run node script"
```

### Task Grouping (Namespaces)

Subdirectories create namespaced tasks:

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

### Remote Tasks

Fetch tasks from external sources:

```toml
[tasks.build]
file = "https://example.com/build.sh"

# Git (experimental)
[tasks.release]
file = "git::https://github.com/org/repo.git//scripts/release.sh?ref=v1.0.0"
file = "git::ssh://git@github.com/org/repo.git//path?ref=main"
```

Format: `git::<protocol>://<url>//<path>?<ref>`

Remote files are cached in `MISE_CACHE_DIR`. Clear with `mise cache clear`.

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
| `choices` | values | none | Restrict to enumerated set |
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
| `help` | string | none | Short help text |
| `long_help` | string | none | Extended help for `--help` |
| `default` | string/bool | none | Default value. Boolean flags use `#true`/`#false`. |
| `env` | string | none | Environment variable backing the flag |
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
| `config` | string | none | Config file key backing the flag |

**Examples:**

```
flag "-v --verbose" help="Enable verbose output"
flag "-f --force" help="Skip confirmation prompts"
flag "--port <port>" default="8080" help="Server port"
flag "--color" negate="--no-color" default=#true help="Enable colors"
flag "-d --debug" count=#true help="Debug level (-ddd for max)"
flag "--include <pattern>" var=#true help="Include patterns (repeatable)"
flag "--shell <shell>" { choices "bash" "zsh" "fish" }
flag "--file <file>" required_if="--dir" help="Required if --dir is set"
flag "--color" env="MYCLI_COLOR" help="Backed by env var"
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

**Supported shebangs:** `bash`, `node`, `python`, `deno`, `pwsh`, `fish`, `zsh`.

---

## Task Configuration Reference

### All Task Fields

#### Core Execution

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `run` | `string \| string[] \| (string \| {task: string} \| {tasks: string[]})[]` | — | Command(s) to execute. Supports structured array (see below). |
| `run_windows` | same as `run` | — | Windows-specific override |
| `file` | `string` | — | External script path (local, HTTP, or Git URL) |
| `shell` | `string` | `sh -c` (Unix), `cmd /c` (Windows) | Interpreter (e.g., `"bash -c"`, `"node -e"`) |
| `usage` | `string` | — | Usage spec for arguments/flags |

#### Metadata

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `description` | `string` | — | Task help text |
| `alias` | `string \| string[]` | — | Alternative name(s) |
| `hide` | `bool` | `false` | Exclude from listings |

#### Dependencies

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `depends` | `string \| string[] \| object[]` | — | Tasks to run BEFORE (supports structured objects) |
| `depends_post` | `string \| string[] \| object[]` | — | Tasks to run AFTER |
| `wait_for` | `string \| string[] \| object[]` | — | Wait for tasks without adding as deps |

#### Environment & Tools

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `env` | `table` | — | Task-specific env vars (**NOT passed to depends**). Supports age-encrypted values and `_.file` directive. |
| `tools` | `table` | — | Tools to install before running |

#### Execution Context

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dir` | `string` | `{{config_root}}` | Working directory. Use `{{cwd}}` for user's cwd. |
| `raw` | `bool` | `false` | Direct stdin/stdout connection (disables parallel execution) |
| `interactive` | `bool` | `false` | Like `raw` but acquires exclusive global write lock instead of forcing single-threaded |

#### Output Control

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `quiet` | `bool` | `false` | Suppress mise output |
| `silent` | `bool \| "stdout" \| "stderr"` | `false` | Suppress all/specific output |

#### Caching

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sources` | `string \| string[]` | — | Input files (glob patterns) |
| `outputs` | `string \| string[] \| {auto: true}` | `{auto: true}` | Generated files. Use `{auto = true}` for implicit tracking. |

#### Safety

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `confirm` | `string` | — | Prompt before running. Supports Tera templates with `usage.*`. |

#### Timeout & Inheritance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `timeout` | `string` | — | Max execution duration (e.g., `"30s"`, `"5m"`). Overrides global `task.timeout`. |
| `extends` | `string` | — | Template task to inherit configuration from. |

### Structured `run` Array

Mix inline scripts with task references, including parallel sub-task execution:

```toml
[tasks.pipeline]
run = [
  { task = "lint" },                 # run lint (with its dependencies)
  { tasks = ["test", "typecheck"] }, # run test and typecheck in parallel
  "echo 'All checks passed!'",      # then run a script
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
  "git::https://github.com/org/tasks?ref=v1"  # Remote tasks (experimental)
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
path = "vars.toml"  # Load vars from external file

[tasks.build]
run = "echo Building {{vars.project_name}} v{{vars.version}}"
```

Vars are accessed via `{{vars.key_name}}` Tera templates. They are **not** passed as environment variables. Task-level `vars` override config-level declarations.

#### Global Task Settings

| Setting | Type | Default | Env Var | Description |
|---------|------|---------|---------|-------------|
| `task.output` | string | none | `MISE_TASK_OUTPUT` | Output mode: prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `task.timeout` | string | none | `MISE_TASK_TIMEOUT` | Default timeout (e.g., `"30m"`) |
| `task.timings` | bool | none | `MISE_TASK_TIMINGS` | Show elapsed time per task |
| `task.skip` | string[] | `[]` | `MISE_TASK_SKIP` | Tasks to skip by default |
| `task.skip_depends` | bool | `false` | `MISE_TASK_SKIP_DEPENDS` | Skip dependencies |
| `task.run_auto_install` | bool | `true` | `MISE_TASK_RUN_AUTO_INSTALL` | Auto-install missing tools |
| `task.show_full_cmd` | bool | `false` | `MISE_TASK_SHOW_CMD_NO_TRUNC` | Disable command truncation in output |
| `task.disable_paths` | string[] | `[]` | `MISE_TASK_DISABLE_PATHS` | Paths to exclude from task discovery |
| `task.remote_no_cache` | bool | none | `MISE_TASK_REMOTE_NO_CACHE` | Always fetch latest remote tasks |
| `task.source_freshness_hash_contents` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_HASH_CONTENTS` | Use blake3 content hashing instead of mtime |
| `task.source_freshness_equal_mtime_is_fresh` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_EQUAL_MTIME_IS_FRESH` | Equal mtime = fresh |
| `task.disable_spec_from_run_scripts` | bool | `false` | `MISE_TASK_DISABLE_SPEC_FROM_RUN_SCRIPTS` | Exclude template functions from usage spec |
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
| `mise watch <task>` / `mise w <task>` | Watch and re-run on file changes |

### Execution Flags

| Flag | Description |
|------|-------------|
| `-j, --jobs N` | Parallel job limit (default: 4) |
| `-f, --force` | Ignore source/output caching |
| `-n, --dry-run` | Preview without executing |
| `-o, --output MODE` | prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `-r, --raw` | Direct stdin/stdout/stderr (forces `--jobs=1`) |
| `-q, --quiet` | Suppress extra output |
| `-S, --silent` | Hide all output except errors |
| `-c, --continue-on-error` | Continue running tasks even if one fails |
| `-C, --cd <DIR>` | Change working directory before execution |
| `-s, --shell SHELL` | Shell spec (default: `sh -c -o errexit -o pipefail`) |
| `-t, --tool TOOL@VERSION` | Additional tools beyond mise.toml |
| `--timings` | Show elapsed time per task |
| `--no-timings` | Hide task completion duration |
| `--timeout DURATION` | Task timeout (e.g., `30s`, `5m`) |
| `--fresh-env` | Bypass environment cache |
| `--skip-deps` | Run only specified tasks, skip dependencies |
| `--no-cache` | Skip cache on remote tasks |
| `--no-prepare` | Skip automatic dependency preparation |

### Parallel Tasks and Wildcards

```bash
mise run test:*         # All test:* tasks
mise run lint:**        # All nested lint tasks
mise run {build,test}   # Multiple specific tasks
mise run lint ::: test ::: check  # Parallel task groups with :::
mise run cmd1 arg1 ::: cmd2 arg2 # Parallel with separate args
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

If a dependency fails, the dependent task skips execution.

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

### Redactions (Experimental)

Hide sensitive values from task output:

```toml
redactions = ["API_KEY", "PASSWORD", "SECRETS_*"]
```

Redactions work by intercepting task output line-by-line, requiring non-`raw` output mode. Tasks with `raw = true` bypass redactions.

**CI integration example (GitHub Actions):**
```bash
for value in $(mise env --redacted --values); do
  echo "::add-mask::$value"
done
```

---

## Dev Tools Management

### Backends Overview

mise supports 18+ backend types. Recommended priority order:

| Priority | Backend | Description |
|----------|---------|-------------|
| 1 | **aqua** | Most features, best security (cosign/SLSA/attestation/minisign). No plugins needed. |
| 2 | **github** | GitHub releases with auto OS/arch detection |
| 3 | **gitlab** | GitLab releases |
| 4 | **forgejo** | Forgejo/Codeberg releases |
| 5 | **http** | Direct HTTP/HTTPS downloads with URL templating |
| 6 | **s3** | S3/MinIO (experimental) |
| 7 | **pipx** | Python CLIs in isolated environments (uses uvx by default) |
| 8 | **npm** | Node packages |
| 9 | **go** | Go packages (requires compilation) |
| 10 | **cargo** | Rust packages (uses binstall by default) |
| 11 | **gem** | Ruby gems |
| 12 | **ubi** | Universal Binary Installer |
| 13 | **dotnet** | .NET tools (experimental) |
| 14 | **conda** | Conda packages (experimental) |
| 15 | **spm** | Swift packages (experimental) |
| 16 | **vfox** | vfox plugins (cross-platform, Windows) |
| 17 | **asdf** | asdf plugins (legacy, no Windows) |

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

### Per-Tool Options

Every tool supports these options regardless of backend:

| Option | Type | Description |
|--------|------|-------------|
| `version` | string | Tool version |
| `os` | string[] | Restrict to OS: `"linux"`, `"macos"`, `"windows"` |
| `install_env` | table | Environment variables during installation |
| `postinstall` | string | Command after installation. `MISE_TOOL_INSTALL_PATH` available. |

### Backend-Specific Configuration

#### Aqua Backend

```toml
[tools]
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:cli/cli" = { version = "latest", symlink_bins = true }  # Filtered .mise-bins directory
```

**Security settings:**

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `aqua.cosign` | bool | `true` | Verify cosign signatures |
| `aqua.slsa` | bool | `true` | Verify SLSA provenance |
| `aqua.github_attestations` | bool | `true` | Verify GitHub Artifact Attestations |
| `aqua.minisign` | bool | `true` | Verify minisign signatures |
| `aqua.baked_registry` | bool | `true` | Use built-in aqua registry |
| `aqua.registry_url` | string | none | Custom aqua registry URL |
| `aqua.cosign_extra_args` | string[] | none | Additional cosign arguments |

**Limitation:** Aqua tools can't set environment variables or do more than download binaries.

#### GitHub Backend Options

```toml
[tools."github:cli/cli"]
version = "latest"
asset_pattern = "gh_*_linux_x64.tar.gz"  # Match specific release asset
bin = "gh"                                 # Rename binary
filter_bins = "gh"                         # Only expose specific bins
no_app = true                              # Skip macOS .app bundles
bin_path = "cli-{{ version }}/bin"         # Binary directory (Tera templating)
rename_exe = "gh"                          # Rename executable after extraction
version_prefix = "release-"               # Custom version tag prefix
checksum = "sha256:..."                    # SHA256 verification
size = "12345678"                          # File size verification
api_url = "https://github.mycompany.com/api/v3"  # GitHub Enterprise

[tools."github:cli/cli".platforms]
linux-x64 = { asset_pattern = "gh_*_linux_x64.tar.gz" }
macos-arm64 = { asset_pattern = "gh_*_macOS_arm64.tar.gz" }
```

#### HTTP Backend

```toml
[tools."http:my-tool"]
version = "1.0.0"
url = "https://example.com/releases/my-tool-v{{version}}.tar.gz"

[tools."http:my-tool".platforms]
macos-x64 = { url = "https://example.com/tool-macos-x64.tar.gz", checksum = "sha256:..." }
linux-x64 = { url = "https://example.com/tool-linux-x64.tar.gz", checksum = "sha256:..." }
```

URL template variables: `{{ version }}`, `{{ os() }}`, `{{ arch() }}`, `{{ os_family() }}`
OS/arch remapping: `{{ os(macos="darwin") }}`, `{{ arch(x64="amd64") }}`

**Version discovery options:**
- `version_list_url` — URL returning version list (text, JSON array, or JSON objects)
- `version_regex` — Regex to extract versions (first capturing group)
- `version_json_path` — jq-like path with filter support (e.g., `".[].tag_name"`)
- `version_expr` — expr-lang expression with access to response `body`

#### Cargo Backend

```toml
[tools]
"cargo:eza" = "latest"
"cargo:cargo-edit" = { version = "latest", features = "add" }
"cargo:demo" = { version = "latest", default-features = false }
"cargo:demo" = { version = "latest", bin = "demo", crate = "demo", locked = false }
```

Settings: `cargo.binstall = true` (use precompiled binaries, default), `cargo.binstall_only = false`, `cargo.registry_name` (custom registry).

**Git-based installation:**
```bash
mise use cargo:https://github.com/user/repo@tag:v1.0
mise use cargo:https://github.com/user/repo@branch:main
mise use cargo:https://github.com/user/repo@rev:abc123
```

#### Pipx Backend

```toml
[tools]
"pipx:black" = "latest"
"pipx:harlequin" = { version = "latest", extras = "postgres,s3" }
"pipx:ansible" = { version = "latest", uvx = false }  # Disable uv
"pipx:ansible-core" = { version = "latest", uvx_args = "--with ansible" }
"pipx:black" = { version = "latest", pipx_args = "--preinstall" }
```

Settings: `pipx.uvx = true` (use uvx by default), `pipx.registry_url`.

**Install sources:** PyPI, GitHub (`pipx:psf/black`), Git (`pipx:git+https://...`), HTTP ZIP.

### Shims and Aliases

**Shims location:** `~/.local/share/mise/shims`

```bash
# Shim mode (non-interactive shells — .bash_profile/.zprofile)
eval "$(mise activate zsh --shims)"

# PATH mode (interactive shells — .bashrc/.zshrc)
eval "$(mise activate zsh)"
```

**Best practice:** Use both — shims in profile for non-interactive, PATH activation in rc for interactive.

**Shims vs PATH:**
- Shims: env vars only load when shim is called, most hooks don't trigger, `which` points to shim
- PATH: full environment, hooks work, `which` shows actual binary
- Recommendation: PATH (`mise activate`) for interactive, shims for non-interactive/IDE

**Tool aliases (remap to different backend):**
```toml
[tool_alias]
node = 'asdf:company/our-custom-node'

[tool_alias.node.versions]
lts = '22'
```

**Shell aliases:**
```toml
[shell_alias]
ll = "ls -la"
gs = "git status"
```

### CLI Commands for Tools

```bash
mise use node@22            # Install + activate + write to mise.toml
mise use -g node@22         # Write to global config
mise install node@20        # Install without activation
mise install                # Install all configured tools
mise ls                     # List installed
mise ls-remote node         # List available versions
mise which node             # Show real binary path
mise x python@3.12 -- script.py  # Run with specific tool
mise reshim                 # Rebuild shims
mise registry               # List all available tools
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
mise unset NODE_ENV             # Remove
mise env                        # Export all
mise env --json                 # Export as JSON
mise env --dotenv               # Export as dotenv
mise env --redacted             # Show only redacted variables
mise env --values               # Show only values
```

### Special Directives (`env._`)

The reserved key `_` is used as a TOML table for configuration since nested environment variables don't make sense.

#### `_.path` — Prepend to PATH

```toml
[env]
_.path = ["tools/bin", "{{config_root}}/scripts"]
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }  # Lazy eval after tools
```

Relative paths resolve against `{{config_root}}`.

#### `_.file` — Load from .env/json/yaml files

```toml
[env]
_.file = '.env'
_.file = ['.env', '.env.local', '.env.{{env.MISE_ENV}}']
_.file = { path = ".secrets.yaml", redact = true }
```

Supported formats: dotenv, JSON, YAML. Auto-loading: set `MISE_ENV_FILE=.env`.

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

Profiles enable environment-specific config files:

```bash
MISE_ENV=staging mise run deploy
```

This loads `mise.staging.toml` in addition to `mise.toml`. Config file order:
- `mise.toml` (base)
- `mise.staging.toml` (profile overlay)
- `mise.staging.local.toml` (local overrides, gitignored)

**Important:** `MISE_ENV` cannot be set in `mise.toml` — it must be in `.miserc.toml`, an env var, or CLI flag because it determines which config files to load. Supports comma-separated values: `MISE_ENV=ci,test`.

### Required and Redacted Variables

```toml
[env]
# Required — error if not set (warns during shell activation)
DATABASE_URL = { required = true }
DATABASE_URL = { required = "Set postgres connection string" }

# Redacted — hidden from output
API_KEY = { value = "secret_key_here", redact = true }

# Pattern-based redactions
redactions = ["*_TOKEN", "SECRET_*", "API_*"]
```

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
| `tools` | HashMap | Installed tool info (name → version/path) |
| `usage` | HashMap | Task arguments/flags (task run scripts only) |
| `xdg_cache_home` | PathBuf | XDG cache directory |
| `xdg_config_home` | PathBuf | XDG config directory |
| `xdg_data_home` | PathBuf | XDG data directory |
| `xdg_state_home` | PathBuf | XDG state directory |

**Key Tera functions:**

| Function | Description |
|----------|-------------|
| `exec(command="cmd")` | Execute shell command, return stdout. Supports `cache_key` and `cache_duration` params. |
| `arch()` | System architecture (e.g., `x64`, `arm64`) |
| `os()` | Operating system (linux, macos, windows) |
| `os_family()` | OS family (unix/windows) |
| `num_cpus()` | CPU count |
| `now()` | Current datetime. Params: `timestamp` (bool), `utc` (bool). |
| `get_env(name, default)` | Get env var with fallback |
| `choice(n, alphabet)` | Generate random string of n characters |
| `read_file(path)` | Read file contents |
| `task_source_files()` | Resolved source file paths (task scripts only) |

**Key Tera filters:**

| Filter | Description |
|--------|-------------|
| `lower`, `upper`, `capitalize`, `title` | Case transforms |
| `trim`, `trim_start`, `trim_end` | Whitespace removal |
| `replace(from, to)` | String substitution |
| `quote` | Escape and quote string |
| `split(pat)`, `join(sep)` | Array operations |
| `first`, `last`, `length`, `reverse` | Collection operations |
| `concat(with)` | Append values to array |
| `map(attribute)` | Extract attribute from objects |
| `basename`, `dirname`, `extname`, `file_stem` | Path operations |
| `file_size`, `last_modified` | File metadata |
| `absolute`, `canonicalize` | Path resolution |
| `join_path` | Join array of path segments |
| `hash`, `hash(algorithm="blake3")` | SHA256/BLAKE3 hashing (supports `len` truncation) |
| `hash_file` | File BLAKE3 hash |
| `kebabcase`, `snakecase`, `lowercamelcase`, `uppercamelcase` | Case conversion |
| `date(format)` | Format datetime (chrono strftime syntax) |
| `default(value)` | Fallback for undefined/empty |
| `abs` | Absolute value (numeric) |
| `filesizeformat` | Human-readable file size |
| `urlencode` | URL-safe encoding |
| `truncate` | Shorten string |

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

### Hook Types

| Hook | Trigger | Requires `mise activate`? |
|------|---------|--------------------------|
| `cd` | Directory changes | Yes |
| `enter` | Enter a project (once) | Yes |
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
```

**Note:** Shell hooks don't perform cleanup when leaving directories like `[env]` declarations do.

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

Sets `MISE_WATCH_FILES_MODIFIED` env var (colon-separated, colons escaped with backslash). Requires watchexec.

### Hook Environment Variables

All hooks receive:
- `MISE_ORIGINAL_CWD` — user's current directory
- `MISE_PROJECT_ROOT` — project root

CD hooks additionally receive:
- `MISE_PREVIOUS_DIR` — previous directory

Postinstall hooks receive:
- `MISE_INSTALLED_TOOLS` — JSON array of installed tools
- `MISE_TOOL_NAME` — tool identifier
- `MISE_TOOL_VERSION` — installed version
- `MISE_TOOL_INSTALL_PATH` — installation directory

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

**Global:** `~/.config/mise/config.toml` (+ `conf.d/` subdirectory)
**System:** `/etc/mise/config.toml` (+ `conf.d/`)
**Legacy:** `.tool-versions` (asdf-compatible)

**Schema validation:** `https://mise.jdx.dev/schema/mise.json` and `https://mise.jdx.dev/schema/mise-task.json`

mise searches upward from cwd to root (or `MISE_CEILING_PATHS`). Merge behavior:
- **Tools:** Additive with overrides
- **Env vars:** Additive with overrides
- **Tasks:** Completely replaced per task name
- **Settings:** Additive with overrides

**Write targeting:** `mise use`, `mise set`, `mise unuse` write to the lowest precedence file in the highest precedence directory.

### Key Settings Reference

```toml
[settings]
# Execution
jobs = 8                    # Concurrent jobs
experimental = false        # Enable experimental features

# Task defaults
task.output = "prefix"      # prefix|interleave|keep-order|replacing|timed|quiet|silent
task.timeout = "10m"        # Default task timeout
task.timings = true         # Show elapsed time
task.skip = ["slow-task"]   # Tasks to skip
task.skip_depends = false   # Skip dependencies

# Environment
env_shell_expand = true     # Enable $VAR expansion
env_cache = false           # Cache computed environment
env_cache_ttl = "1h"        # Cache TTL

# Tool management
auto_install = true         # Auto-install missing tools
disable_backends = ["asdf"] # Disable backends
pin = false                 # Default --pin for mise use
lockfile = true             # Read/update lockfiles

# Security
paranoid = false            # Extra-secure behavior
gpg_verify = false          # Verify GPG signatures
slsa = true                 # SLSA provenance verification
github_attestations = true  # GitHub Artifact Attestations

# Performance
fetch_remote_versions_cache = "1h"  # Version cache
http_timeout = "30s"                # HTTP timeout
http_retries = 0                    # HTTP retries with exponential backoff

# Network
offline = false             # Block all HTTP requests
prefer_offline = false      # Prefer cached data

# Node-specific
[settings.node]
corepack = false            # Enable corepack
compile = false             # Compile from source

# Python-specific
[settings.python]
uv_venv_auto = false        # Auto-create venv with uv (false|source|"create|source"|true)
compile = false             # Compile from source

# Ruby-specific
[settings.ruby]
compile = false             # Compile from source

# Aqua security
[settings.aqua]
cosign = true               # Verify Cosign signatures
slsa = true                 # Verify SLSA provenance
github_attestations = true  # Verify GitHub Attestations
minisign = true             # Verify minisign signatures

# Cargo
[settings.cargo]
binstall = true             # Use precompiled binaries
binstall_only = false       # Require binstall

# Pipx
[settings.pipx]
uvx = true                  # Use uvx instead of pipx

# Age encryption (experimental)
[settings.age]
key_file = "~/.config/mise/age.txt"

# Sops encryption
[settings.sops]
rops = true                 # Use native Rust implementation
strict = true               # Fail on decryption errors
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

---

## Prepare Feature (Experimental)

Ensures dependencies are installed before task execution.

```bash
export MISE_EXPERIMENTAL=1
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

---

## Monorepo Tasks (Experimental)

```toml
# Root mise.toml
experimental_monorepo_root = true
```

Requires `MISE_EXPERIMENTAL=1`.

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
- Use `depends` for task ordering
- Use `confirm` for destructive operations
- Use `choices` for stable enums, `complete` for dynamic/filesystem-derived values
- Group related tasks with namespaces (e.g., `test:unit`, `test:e2e`)
- Use `mise.local.toml` for personal overrides (gitignored)
- Prefer aqua backend for security (cosign/SLSA/attestation verification)
- Use `env._.file` for dotenv loading instead of `MISE_ENV_FILE`
- Redact sensitive values with `redact = true`
- Use templates for dynamic values instead of hardcoding paths
- Use `extends` to share config between similar tasks
- Use shims in `.zprofile`/`.bash_profile` and PATH activation in `.zshrc`/`.bashrc`

### DON'T ❌

- Use `$1`, `$2`, `$@`, `$*` for arguments
- Use `$args` in PowerShell
- Use inline `{{arg(name="x")}}` syntax (deprecated, removed 2026.11)
- Forget to quote glob patterns in sources
- Set env vars in `env` that deps need (they don't inherit — use structured `depends` with `env`)
- Use `raw = true` unless interactive input needed (forces single-threaded)
- Set `MISE_ENV` in `mise.toml` (it determines which files to load)
- Manually add executables to shims directory (`mise reshim` removes them)
- Use `MISE_RAW=1` without knowing it sets `MISE_JOBS=1`

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
