---
name: mise
description: Use when working with mise - creating/editing mise.toml or .mise.toml files, defining tasks with usage field arguments, managing dev tools and environments, configuring hooks, sandboxing, machine bootstrap, or when user mentions mise configuration.
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
  - [Subcommands (cmd block)](#subcommands-cmd-block)
  - [Spec-Level Metadata](#spec-level-metadata)
  - [Documented but NOT Implemented](#documented-but-not-implemented-do-not-use)
- [Task Configuration Reference](#task-configuration-reference)
  - [All Task Fields](#all-task-fields)
  - [Structured run Array](#structured-run-array)
  - [Structured depends with Args/Env](#structured-depends-with-argsenv)
  - [Task Templates (extends)](#task-templates-extends)
  - [Global Task Configuration](#global-task-configuration)
  - [Variables (vars)](#vars-section)
- [Running Tasks (CLI)](#running-tasks-cli)
- [Task Dependencies and Caching](#task-dependencies-and-caching)
- [Dev Tools Management](#dev-tools-management)
  - [Backends Overview](#backends-overview)
  - [TOML Syntax for Tools](#toml-syntax-for-tools)
  - [Per-Tool Options](#per-tool-options)
  - [Backend-Specific Configuration](#backend-specific-configuration)
  - [Shims and Aliases](#shims-and-aliases)
  - [Shell Completion (per-directory tab-complete)](#shell-completion-per-directory-tab-complete)
  - [Lockfiles (mise.lock)](#lockfiles-miselock)
  - [Auto-Install Controls](#auto-install-controls)
  - [CLI Commands for Tools](#cli-commands-for-tools)
- [Environment Configuration](#environment-configuration)
  - [Basic Variables](#basic-variables)
  - [Special Directives (env._)](#special-directives-env_)
  - [Profiles (MISE_ENV)](#profiles-mise_env)
  - [Required and Redacted Variables](#required-and-redacted-variables)
  - [Secrets (fnox, SOPS, age)](#secrets-fnox-sops-age)
  - [Templates (Tera)](#templates-tera)
- [Hooks and Watchers](#hooks-and-watchers)
- [Sandboxing and Safe Mode](#sandboxing-and-safe-mode)
- [Machine Bootstrap (Developer Setup)](#machine-bootstrap-developer-setup)
  - [`[bootstrap]` Configuration](#bootstrap-configuration)
  - [Declarative Dotfiles (`[dotfiles]`)](#declarative-dotfiles-dotfiles)
- [OCI Container Images](#oci-container-images)
- [Configuration and Settings](#configuration-and-settings)
  - [File Hierarchy](#file-hierarchy)
  - [Key Settings Reference](#key-settings-reference)
  - [Minimum Version](#minimum-version)
  - [Automatic Environment Variables](#automatic-environment-variables)
- [Dependency Preparation (`[deps]`)](#dependency-preparation-deps)
- [Monorepo Tasks](#monorepo-tasks)
- [Deprecation Calendar](#deprecation-calendar)
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

- **Dev tools** — install and manage language runtimes, CLIs, and build tools (18 backends)
- **Tasks** — project-specific commands with argument handling, dependencies, caching
- **Environments** — manage env vars, profiles, dotenv files, secrets, age/sops-encrypted values
- **Hooks** — run commands on directory changes, project enter/leave, tool install
- **Sandboxing** — restrict a task's filesystem/network/env access; `safe` mode for untrusted configs
- **Machine bootstrap** — provision a whole dev machine (system packages, git repos, dotfiles, macOS defaults, services, login shell) via `mise bootstrap`
- **OCI images** — build/push container images containing mise-managed tools

**Key Features:**
- Parallel dependency building (concurrent by default, up to `jobs` setting)
- Last-modified and content-hash (blake3) checking (avoid unnecessary rebuilds)
- File watching (`mise watch` rebuilds on changes)
- Cross-platform argument handling via `usage` spec
- Hierarchical configuration with profile support
- 18 tool backends (aqua, github, gitlab, forgejo, npm, cargo, pipx, etc.)
- Security verification (cosign, SLSA, GitHub Attestations, minisign)
- Secret management (sops, age encryption)

**Top-level `mise.toml` keys** (authoritative, from `https://mise.jdx.dev/schema/mise.json`):
`_`, `alias`, `bootstrap`, `deps`, `dotenv`, `dotfiles`, `env`, `env_file`, `env_path`, `hooks`,
`min_version`, `monorepo`, `monorepo_root`, `oci`, `plugins`, `redactions`, `settings`,
`shell_alias`, `task_config`, `task_templates`, `tasks`, `tool_alias`, `tools`, `vars`, `watch_files`.

> There is **no `[prepare]` key** — that feature is now `[deps]`. See [Dependency Preparation](#dependency-preparation-deps).

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

Supports: Python, Node, Bun, Deno, Ruby, Bash, PowerShell. Use `-S` flag for multiple interpreter arguments (`#!/usr/bin/env -S deno run --allow-env`).

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

**Supported `#MISE` directives:** `description`, `alias`, `sources`, `outputs`, `env`, `depends`, `wait_for`, `tools`, `dir`, `hide`, `run`, `quiet`, `raw`, `shell`, `confirm`.
**Supported `#USAGE` directives:** `arg`, `flag`, `complete`, `env`, and root-level `mount`.

**Alternative header syntax** (if formatters modify `#MISE`):
```bash
# [MISE] description="Build"
# [MISE] depends=["lint"]
```

**Accepted comment prefixes** for the usage block — `#`, `//`, and `::` — each in three forms (`#USAGE`, `# [USAGE]`, `#[USAGE]`):

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

> Header extraction only happens when the file's first bytes are `#!`. Blank comment lines continue the block; the **first non-blank, non-USAGE line ends it** — later `#USAGE` lines are ignored.

**Root-level `mount`** (mise 2026.7.0+) hoists a spec produced by another command onto the task:
```bash
#!/usr/bin/env bash
#MISE description="Wrapper around an external CLI"
#USAGE mount "mise run run-release -- --usage-spec"
```

**Direct script execution** bypasses discovery (path must start with `/`, `./`, `C:\`, or `.\`):
```bash
mise run ./path/to/script.sh
```

> By default mise executes these files directly. Setting `use_file_shell_for_executable_tasks = true` (default `false`) routes them through `unix_default_file_shell_args` (`sh`) / `windows_default_file_shell_args` (`cmd /c`) instead.

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

# Git — SSH
[tasks.release]
file = "git::ssh://git@github.com/org/repo.git//scripts/release.sh?ref=v1.0.0"

# Git — HTTPS
[tasks.build]
file = "git::https://github.com/org/repo.git//path?ref=main"
```

Format: `git::<protocol>://<url>//<path>?ref=<ref>` — ref is optional (defaults to repo's default branch).

Remote files cached in `$MISE_CACHE_DIR` (git specifically in `$MISE_CACHE_DIR/remote-git-tasks-cache`). Clear with `mise cache clear`. Disable cache with `MISE_TASK_REMOTE_NO_CACHE=true` or `--no-cache`.

> Since 2026.7.11, remote HTTP and Git-backed task files have their `#MISE` headers parsed (`tools`, `description`, `hide`, inline TOML overrides), and git-backed files are made executable after clone and on cache hits.

Remote git includes in `[task_config]`:
```toml
[task_config]
includes = ["git::https://github.com/myorg/shared-tasks.git//tasks?ref=main"]
```

---

## Task Arguments - Usage Spec Reference

**REMINDER:** Always use the `usage` field. Never use `$1`, `$@`, or shell-native argument handling.

The `usage` field uses [KDL-inspired syntax](https://usage.jdx.dev/) to define arguments, flags, and completions. Reference below verified against **usage v3.5.6**.

### Positional Arguments (`arg`)

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (name) | string | *required* | `"<name>"` = required, `"[name]"` = optional. `"<file>"`/`"<path>"` triggers file completion, `"<dir>"` triggers dir completion. |
| `help` | string | none | Short help text shown with `-h` |
| `long_help` / `help_long` | string | none | Extended help text shown with `--help` |
| `help_md` | string | none | Markdown-only help (docs generation) |
| `required` | boolean | `#true` for `<x>`, `#false` for `[x]` | **Forced to `#false` whenever `default` is set** |
| `default` | string | none | Default value if not provided. `default=""` sets to empty string (different from unset). |
| `env` | string | none | Environment variable that can provide this arg's value. Priority: CLI > env > default. |
| `var` | boolean | `#false` | Variadic mode (accept multiple values). Shorthand: `"<name>..."` |
| `var_min` | integer | none | Minimum values when variadic |
| `var_max` | integer | none | Maximum values when variadic |
| `choices` | child node | none | Restrict to enumerated set (see below) |
| `double_dash` | enum | `optional` | `"required"`, `"optional"`, `"automatic"`, `"preserve"` |
| `hide` | boolean | `#false` | Exclude from help output |

**Examples:**

```
arg "<file>" help="Input file to process"
arg "[output]" default="out.txt" help="Output file"
arg "<files>" var=#true var_min=1 help="One or more files"
arg "<files>..." help="Shorthand variadic syntax"
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
arg "<env>" { choices env="DEPLOY_ENVS" }
arg "<env>" { choices "local" env="DEPLOY_ENVS" }   # literal + env-backed, deduped
arg "<args>..." double_dash="automatic" help="Pass-through arguments"
arg "<file>" env="MY_FILE" help="Input file (or set MY_FILE)"
arg "<mode>" { default { "fast"; "safe" } }         # multi-value default block
```

**Variadic args in bash:**
```bash
# Values are a shell-escaped string in usage_files
eval "files=($usage_files)"
for f in "${files[@]}"; do
  echo "Processing: $f"
done
```

### Flags (`flag`)

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (definition) | string | *required* | `"-s --long"` (boolean) or `"-s --long <value>"` (with value). Short flag optional. Short flags must be exactly 1 char. |
| `help` | string | none | Short help text |
| `long_help` / `help_long` | string | none | Extended help for `--help` |
| `help_md` | string | none | Markdown-only help |
| `required` | boolean | `#false` | **Forced `#false` when `default` set** |
| `default` | string/bool | none | Default value. Boolean flags use `#true`/`#false`. Multi-value block form also accepted. |
| `env` | string | none | Environment variable backing the flag |
| `global` | boolean | `#false` | Available on all subcommands |
| `count` | boolean | `#false` | Value = number of times used (e.g., `-vvv` = 3) |
| `var` | boolean | `#false` | Flag repeatable, collecting values |
| `var_min` | integer | none | Min values when `var=#true` |
| `var_max` | integer | none | Max values when `var=#true` |
| `negate` | string | none | Negative form (e.g., `"--no-color"`). Sets env var to `false`. |
| `allow_hyphen_values` | boolean | `#false` | **usage 3.5.5+.** Let a value-taking flag consume a following `-…` token as its value (pass-through to wrapped CLIs). Errors if the flag takes no value. |
| `deprecated` | bool \| string | none | `#true` → literal `"deprecated"`; a string is used as the message |
| `choices` | child node | none | Restrict values to enumerated set (flag must take a value) |
| `hide` | boolean | `#false` | Exclude from docs/completions |

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
flag "--color" env="MYCLI_COLOR" help="Backed by env var"
flag "-a --args <ARGS>" allow_hyphen_values=#true help="Pass-through args (e.g. -a -destroy)"
flag "--old-flag" deprecated="use --new-flag instead"
```

**Count flags:** `-vvv` sets `$usage_verbose` to `3`. Short flags chain: `-abc` = `-a -b -c`.

**Negate flags:** `flag "--color" negate="--no-color" default=#true` — `--no-color` sets `$usage_color` to `false`. Help renders these as `--color / --no-color` (usage 3.5.3+).

> **No `alias` child node.** To give a flag a short form, put it in the definition string: `flag "-u --user <user>"`.

### Custom Completions (`complete`)

Provide **dynamic tab-completion** for arguments. Preferred over `choices` when values change.

```
complete "<arg_name>" run="<shell command outputting one value per line>"
complete "<arg_name>" type="dir"     # built-in completer: path | file | dir
```

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `run` | string (Tera) | none | Shell command; one candidate per line |
| `type` | string | none | Built-in completer: `path`/`file` (cwd entries) or `dir`. **Mutually exclusive with `run`.** |
| `descriptions` | boolean | `#false` | Parse `value:description` output |

**Key rules:**
- Node name is lowercased and must match the arg name exactly
- Must appear **after** the `arg` it applies to
- Do **not** combine with `choices` on the same `arg`
- When `type` is unset, **the arg's own name is used as the type** — which is why `arg "<file>"`, `arg "<dir>"`, and `arg "<path>"` auto-complete with no `complete` block at all

**Resolution order per arg:** built-in type (or arg name) → `choices` → `run`.

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
Output format: `value:description` per line — split on the **first unescaped colon**; escape a literal colon with `\:`; both sides are trimmed.

**Tera template variables in `run`:**
- `words` — array of all prompt words including the one being typed; access via `words[index]`
- `CURRENT` — index of word being typed
- `PREV` — index of previous word (`CURRENT-1`). **Only defined when `CURRENT > 0`.**

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

**Naming rules:** `usage_` prefix + snake_case of the long name (hyphens → underscores, lowercased).

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

**Critical distinction:** `default=""` makes the variable **SET** to an empty string. No default + not provided = **UNSET** (not empty) — test with `[ -n "${usage_x:-}" ]`, not `= "false"`.

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

**Precedence:** CLI args > env vars > defaults. This applies to **both TOML tasks and file tasks** (documented for file tasks since 2026.7.12).

> **Breaking change (2026.7.6):** `usage_*` variables are now **invocation-local** — they no longer leak into nested tasks as implicit inputs. If a dependency needs a value, declare it with `env=` or pass it via structured `depends`.

Arguments are also readable as Tera values in TOML tasks: `{{ usage.env }}`, `{{ usage["dry-run"] }}`.

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

**Supported shebangs:** `bash`, `node`, `python`, `deno`, `pwsh`, `fish`, `zsh`, `ruby`, `bun`.

### Subcommands (`cmd` block)

The `usage` spec supports nested subcommands via `cmd` blocks. Each subcommand can declare its own args/flags/completions:

```toml
[tasks.deploy]
usage = '''
flag "-v --verbose" global=#true help="Verbose output"

cmd "staging" help="Deploy to staging" {
  arg "<service>"
  flag "--force"
}

cmd "production" help="Deploy to production" {
  arg "<service>"
  flag "--canary"
  before_help "Production deploys require approval."
}
'''
```

| `cmd` attribute | Type | Notes |
|-----------------|------|-------|
| `help`, `long_help`, `before_help`, `after_help`, `before_long_help`, `after_long_help` | string | Help text (all valid as props *or* child nodes) |
| `help_md`, `before_help_md`, `after_help_md`, `help_long`, `before_help_long`, `after_help_long` | string | **Props only** — these fail as child nodes |
| `subcommand_required` | boolean | Require a subcommand |
| `hide` | boolean | Hide from help |
| `restart_token` | string | Resets arg parsing for repeated invocations (this is how `mise run a ::: b` works) |
| `deprecated` | bool \| string | Mark deprecated |
| `alias "x"` | child node | Repeatable; supports `hide=#true` |
| `example "code"` | child node | **Exactly one** positional arg; use `header=` / `help=` / `lang=` props |
| `mount run="<cmd>"` | child node | Dynamic subcommand mounting; `run` is required |

```
example "mycli list --all" header="Basic usage" help="Lists everything"
```

### Spec-Level Metadata

Top-level usage keywords (rarely needed for mise tasks but supported):

```
name "..."              # display name
bin "..."               # binary name (defaults to filename)
version "..."           # version string
min_usage_version "..." # minimum usage spec version required (put first; warns, does not fail)
author "..."            # CLI author
license "..."           # SPDX license identifier
about "..."             # short help
long_about "..."        # long help
before_help "..."       # text before help body
after_help "..."        # text after help body
before_long_help "..."  # before-help shown with --help
after_long_help "..."   # after-help shown with --help
usage "..."             # override the generated usage line
disable_help #true      # suppress the automatic help flag
default_subcommand "..." # "naked" invocation target
source_code_link_template "..."  # Tera template for doc source links
example "code" header="..." help="..." lang="..."
include file="./other.usage.kdl"   # merge another spec (file= is required)
```

**`config` block** — declares config-file-backed properties (metadata for docs/SDK generation):

```
config {
  prop "color"   default=#true   env="COLOR"   help="Enable color output"
  prop "jobs"    default=4       env="JOBS"    help="Number of jobs" data_type="integer"
  prop "timeout" default=1.5     env="TIMEOUT" help="Timeout" data_type="float"
}
```

`prop` attributes: `default`, `default_note`, `data_type` (`null`/`string`/`integer`/`float`/`boolean`), `env`, `help`, `long_help`.

### Documented but NOT Implemented (do not use)

These appear on usage.jdx.dev but **hard-error** in usage v3.5.6 (verified empirically against the installed binary). Avoid them:

| Syntax | Error | Use instead |
|--------|-------|-------------|
| `arg "<f>" parse="cmd {}"` | `unsupported arg key parse` | — (no equivalent) |
| `flag "--user <u>" { alias "-u" }` | `unsupported flag key alias` | `flag "-u --user <u>"` |
| `flag "--color" config="ui.color"` | `unsupported flag key config` | `env="..."` |
| `flag "--f <x>" required_if="--dir"` | `unsupported flag key` | Validate inside the script |
| `flag "--f <x>" required_unless="--stdin"` | `unsupported flag key` | Validate inside the script |
| `flag "--f <x>" overrides="--stdin"` | `unsupported flag key` | Validate inside the script |
| `config { file "..." findup=#true format="ini" }` | `unsupported` | `config { prop "..." }` |
| `config { default "k" "v" }` / `config { alias "a" "b" }` | `unsupported` | `config { prop "..." default=... }` |
| `cmd "x" { example "Header" "code" }` | needs exactly 1 arg | `example "code" header="Header"` |

> Consequently the "CLI flag > env var > config file > default" chain described in the usage docs is **not wired up for config files** in this release — the `config` block is metadata only. Env-var backing (`env=`) does work.

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
| `vars` | `table` | — | Task-local vars that override `[vars]` |

#### Execution Context

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dir` | `string` | `{{config_root}}` | Working directory. Use `{{cwd}}` for user's cwd. |
| `raw` | `bool` | `false` | Direct stdin/stdout connection (disables parallel execution; bypasses redactions) |
| `raw_args` | `bool` | `false` | Pass all args verbatim including `--help`/`-h` to underlying command |
| `interactive` | `bool` | `false` | Exclusive lock on stdin/stdout/stderr; other non-interactive tasks still run in parallel (narrower than `raw`) |

#### Output Control

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `output` | `"prefix" \| "interleave" \| "keep-order" \| "replacing" \| "timed" \| "quiet" \| "silent"` | inherits `task.output` | **Per-task output style** (2026.7.6+). Orthogonal to `quiet`/`silent`. |
| `quiet` | `bool` | `false` | Suppress mise's own chatter (echoed command). **No longer changes the output style.** |
| `silent` | `bool \| "stdout" \| "stderr"` | `false` | Suppress all/specific task output |

#### Caching

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sources` | `string \| string[]` | — | Input files (glob patterns) |
| `outputs` | `string \| string[] \| {auto: true}` | `{auto: true}` (if `sources` set) | Generated files. `{auto = true}` writes to `~/.local/state/mise/task-outputs/<hash>`. |

#### Safety & Sandboxing

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `confirm` | `string \| {message: string, default: string}` | — | Prompt before running. Supports Tera with `usage.*`. **Guards only `run`, not `depends`** (deps run before the prompt). |
| `deny_all` | `bool` | `false` | Block reads, writes, network, and env inheritance |
| `deny_read` / `deny_write` / `deny_net` / `deny_env` | `bool` | `false` | Individual sandbox blocks |
| `allow_read` / `allow_write` | `string[]` | — | Path allowlists |
| `allow_net` | `string[]` | — | Host allowlist |
| `allow_env` | `string[]` | — | Env var allowlist |
| `redactions` | `string[]` | — | **Experimental.** Env var names (globs allowed, e.g. `SECRETS_*`) redacted from output as `[redacted]` |

#### Timeout & Inheritance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `timeout` | `string` | — | Max execution duration (e.g., `"30s"`, `"5m"`). **Shorter of per-task and global `task.timeout` wins**; `--timeout` overrides. Tera templates supported (2026.7.6+). |
| `extends` | `string` | — | Name of a `[task_templates.*]` entry to inherit from |

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

Wildcards work too: `depends = ["lint:*"]`.

**Conditional dependencies (2026.7.8+):** a dependency whose **templated name renders empty is skipped**, and empty templated dependency *args* are omitted.

**`wait_for` matching:** by name alone matches regardless of args/env; with explicit args/env it must match exactly.

Env vars passed to dependencies are scoped to that dependency only.

### Task Templates (`extends`)

Reusable task definitions live in `[task_templates.*]`; tasks inherit them with `extends`. Template names may be namespaced with `:`.

```toml
[task_templates."python:build"]
description = "Build a Python project"
run = "uv build"
tools = { python = "3.12", uv = "latest" }
env = { PYTHONPATH = "src" }

[tasks.build]
extends = "python:build"

[tasks.test]
extends = "python:test"
run = "pytest --cov"       # overrides the template's run
```

**Merge rules:**

| Behavior | Fields |
|----------|--------|
| **Full override** (task wins) | `run`, `run_windows`, `depends`, `depends_post`, `wait_for`, `sources`, `outputs`, `output`, `description`, `shell`, `timeout` |
| **Deep merge** (per key; task wins) | `tools`, `env` |
| **Compose / combine** | sandbox `deny_*` compose; `allow_*` combine |
| **Not allowed on templates** | `hide`, `interactive`, `quiet`, `raw`, `raw_args` — set these per task |

Templates are Tera-rendered in the **consuming** project's context (`{{config_root}}`, `{{env.VAR}}`, `{{cwd}}`, `{{vars.*}}`).

### Global Task Configuration

#### [task_config] Section

Only two keys exist (`unevaluatedProperties: false`):

```toml
[task_config]
dir = "{{cwd}}"  # Default working directory for all tasks in this file
includes = [
  "tasks/*.toml",                              # Local task files
  ".mise/tasks/",                              # Task directory
  "git::https://github.com/org/tasks?ref=v1"   # Remote tasks
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
e2e_args = { default = "--headless" }                     # only if unset
api_token = { required = "Set api_token in mise.local.toml" }
secret_arg = { value = "--token=abc123", redact = true }
_.file = "vars.toml"    # load vars from an external file (dotenv/json/yaml/toml)

[tasks.build]
run = "echo Building {{vars.project_name}} v{{vars.version}}"

[tasks.test]
vars = { e2e_args = "--headed" }   # task-local override
run = './scripts/test-e2e.sh {{vars.e2e_args}}'
```

Value forms: plain scalar; `{ value, redact? }`; `{ default, redact? }`; `{ required = true|"msg", redact? }`; `{ age = ... }` (experimental). Plus the `_` module: `_.file` and `_.source`.

Vars accessed via `{{vars.key_name}}` Tera templates. **Scope precedence:** global (`~/.config/mise/config.toml`) < project `mise.toml` < `mise.local.toml` < task-local `vars`.

> The `vars.mise` namespace is **rejected at parse time** (no deprecation period). Use `vars._`.

#### Global Task Settings

| Setting | Type | Default | Env Var | Description |
|---------|------|---------|---------|-------------|
| `task.output` | string | unset | `MISE_TASK_OUTPUT` | prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `task.timeout` | string | unset | `MISE_TASK_TIMEOUT` | Default timeout. Per-task cannot exceed this. |
| `task.timings` | bool | unset | `MISE_TASK_TIMINGS` | Show elapsed time per task |
| `task.skip` | string[] | `[]` | `MISE_TASK_SKIP` | Tasks to skip by default |
| `task.skip_depends` | bool | `false` | `MISE_TASK_SKIP_DEPENDS` | Skip dependencies |
| `task.run_auto_install` | bool | `true` | `MISE_TASK_RUN_AUTO_INSTALL` | Auto-install missing tools |
| `task.show_full_cmd` | bool | `false` | `MISE_TASK_SHOW_FULL_CMD` | Disable command truncation in output |
| `task.disable_paths` | string[] | `[]` | `MISE_TASK_DISABLE_PATHS` | Paths to exclude from task discovery |
| `task.remote_no_cache` | bool | unset | `MISE_TASK_REMOTE_NO_CACHE` | Always fetch latest remote tasks |
| `task.source_freshness_hash_contents` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_HASH_CONTENTS` | Use blake3 content hashing instead of mtime |
| `task.source_freshness_equal_mtime_is_fresh` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_EQUAL_MTIME_IS_FRESH` | Equal mtime = fresh |
| `task.disable_spec_from_run_scripts` | bool | `false` | `MISE_TASK_DISABLE_SPEC_FROM_RUN_SCRIPTS` | Exclude template functions from usage spec (early opt-out before removal) |
| `task.monorepo_depth` | int | `5` | `MISE_TASK_MONOREPO_DEPTH` | Subdirectory depth for monorepo discovery |
| `task.monorepo_exclude_dirs` | string[] | `[]` | `MISE_TASK_MONOREPO_EXCLUDE_DIRS` | Empty ⇒ built-ins (`node_modules`, `target`, `dist`, `build`); any custom value **replaces** them |
| `task.monorepo_respect_gitignore` | bool | `true` | `MISE_TASK_MONOREPO_RESPECT_GITIGNORE` | Honor `.gitignore` in monorepo discovery |
| `jobs` | int | `8` | `MISE_JOBS` | Max concurrent task execution |

> Deprecated flat aliases still parse: `task_output`, `task_timeout`, `task_timings`, `task_skip`, `task_skip_depends`, `task_disable_paths`, `task_remote_no_cache`, `task_run_auto_install`, `task_show_full_cmd`.
>
> The `jobs` **default is documented inconsistently** — the settings reference says `8`, while the `mise run`/`mise install` CLI pages say `4`. Set it explicitly if it matters.

---

## Running Tasks (CLI)

### Commands

| Command | Description |
|---------|-------------|
| `mise run <task>` / `mise r <task>` | Execute task |
| `mise <task>` | Shorthand (if no command conflict) |
| `mise run` (no args) | Runs the `default` task |
| `mise tasks` / `mise tasks ls` | List all tasks |
| `mise tasks --hidden` | Include hidden tasks |
| `mise tasks --global` / `--local` | Filter by config scope |
| `mise tasks deps` | Show dependency tree |
| `mise tasks deps --dot` | DOT format for Graphviz |
| `mise tasks info <task>` | Show task details (JSON output includes `config_sources`) |
| `mise tasks add <name> -- <cmd>` | Create task via CLI (`--depends`, `--run-windows`) |
| `mise tasks edit <task>` | Edit/create task in `$EDITOR` (respects configured `includes`) |
| `mise tasks validate` | Validate task definitions |
| `mise watch <task>` / `mise w <task>` | Watch and re-run on file changes (requires watchexec) |
| `mise run a ::: b ::: c` | Run multiple tasks in parallel (with separate args) |

### Execution Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--jobs <N>` | `-j` | Parallel job limit |
| `--force` | `-f` | Ignore source/output caching |
| `--dry-run` | `-n` | Preview without executing |
| `--output MODE` | `-o` | prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `--raw` | `-r` | Direct stdin/stdout/stderr (forces `--jobs=1`, bypasses redactions) |
| `--quiet` | `-q` | Suppress mise's extra output |
| `--silent` | `-S` | Hide all output except errors |
| `--continue-on-error` | `-c` | Continue running tasks even if one fails |
| `--cd <DIR>` | `-C` | Change working directory before execution |
| `--shell SHELL` | `-s` | Shell spec for TOML tasks |
| `--tool TOOL@VERSION` | `-t` | Additional tools beyond mise.toml |
| `--timings` / `--no-timings` | — | Show / hide elapsed time per task |
| `--timeout DURATION` | — | Task timeout (e.g., `30s`, `5m`) |
| `--fresh-env` | — | Bypass environment cache |
| `--skip-deps` | — | Run only specified tasks, skip dependencies |
| `--no-deps` | — | Skip automatic dependency preparation |
| `--skip-tools` | — | Skip installing tools before running tasks |
| `--no-cache` | — | Skip cache on remote tasks |

**Sandbox / permission flags** (stable since v2026.6.6; also available on `mise exec`):

| Flag | Description |
|------|-------------|
| `--allow-env <VAR>` | Allow specific env var through (supports wildcards; implies deny for everything else) |
| `--allow-net <HOST>` | Allow network access to specific host |
| `--allow-read <PATH>` | Allow filesystem reads from specific path |
| `--allow-write <PATH>` | Allow filesystem writes to specific path |
| `--deny-all` | Block reads, writes, network, and env vars |
| `--deny-env` | Block env var inheritance (keeps PATH/HOME/USER/SHELL/TERM/LANG) |
| `--deny-net` | Block all network access |
| `--deny-read` | Block filesystem reads |
| `--deny-write` | Block all filesystem writes |

> **Breaking change (2026.7.6):** `--quiet` / `quiet = true` / `MISE_QUIET=1` **no longer collapse task output** to un-prefixed interleave — they preserve the resolved style. Use `--output quiet` or `-o interleave` for the old behavior.

### Output Modes

- `prefix` — Default when `jobs > 1`; each line prefixed with task name
- `interleave` — Default when `jobs == 1`; print as output arrives
- `keep-order` — Stream one task's output live; buffer others; print in definition order
- `replacing` — Replace stdout on each new line (similar to `mise install`)
- `timed` — Show only stdout lines that took >1s
- `quiet` — Print only task stdout/stderr, nothing from mise itself
- `silent` — Print nothing from tasks or mise

Set via `--output`, `task.output`, `MISE_TASK_OUTPUT`, or the per-task `output` field.

### `mise watch` Flags

Wrapper around `watchexec`. By default watches the task's `sources` glob set.

| Flag | Short | Description |
|------|-------|-------------|
| `--watch <PATH>` | `-w` | Watch path recursively (repeatable) |
| `--watch-non-recursive <PATH>` | `-W` | Non-recursive watch |
| `--watch-file <PATH>` | `-F` | File listing paths to watch |
| `--exts <EXTS>` | `-e` | Comma-separated extension filter |
| `--filter <PATTERN>` | `-f` | Include glob |
| `--ignore <PATTERN>` | `-i` | Exclude glob |
| `--poll <INTERVAL>` | — | Polling instead of native fs events |
| `--clear <MODE>` | `-c` | `clear` or `reset` between runs |
| `--restart` | `-r` | Shorthand for `--on-busy-update=restart` |
| `--on-busy-update <MODE>` | `-o` | `queue`, `do-nothing` (default), `restart`, `signal` |
| `--signal <SIGNAL>` | `-s` | Signal to running process |
| `--stop-signal <SIGNAL>` | — | Default SIGTERM (Unix) |
| `--stop-timeout <DUR>` | — | Grace period before kill (default `10s`) |
| `--debounce <DUR>` | `-d` | Debounce window (default `50ms`) |
| `--postpone` | `-p` | Wait for first change before initial run |
| `--delay-run <DUR>` | — | Sleep before each execution |
| `--notify` | `-N` | Desktop notifications |
| `--bell` | — | Terminal bell on completion |
| `--print-events` | — | Human-readable event output |
| `--fs-events <KINDS>` | — | Default `create,remove,rename,modify,metadata` |
| `--wrap-process <MODE>` | — | `group` (default), `session`, `none` |
| `--skip-deps` | — | Run only specified tasks; skip deps |

### Parallel Tasks and Wildcards

```bash
mise run test:*         # All test:* tasks
mise run lint:**        # All nested lint tasks
mise run {build,test}   # Multiple specific tasks
mise run lint ::: test ::: check  # Parallel task groups with :::
mise run cmd1 arg1 ::: cmd2 arg2  # Parallel with separate args
```

Glob patterns accepted in task selectors:

| Pattern | Matches |
|---------|---------|
| `?` | Single character |
| `*` | Zero or more characters (within one namespace segment) |
| `**` | Zero or more nested namespace segments |
| `{a,b,c}` | Comma-separated alternatives |
| `[abc]` | Character set/range |
| `[!abc]` | Negated character set |

Extra args go to the **last** command: `mise run build --release`.

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

**Exclusions** use gitignore-style `!` prefixes (escape a literal `!` path with `\!`); later entries override earlier ones:
```toml
sources = ["src/**/*.ts", "!src/**/*.test.ts", "!src/**/*.spec.ts"]
```

**Auto outputs:**
```toml
[tasks.build]
sources = ["src/**/*.rs"]
outputs = { auto = true }  # Implicit tracking via task hash (state in ~/.local/state/mise/task-outputs/<hash>)
run = "cargo build"
```

**Dependency invalidation:** when a depended-on task re-runs because *its* sources changed, the dependent task also re-runs even if its own sources are unchanged.

Enable content-hash (blake3) checking instead of mtime via `task.source_freshness_hash_contents = true` (more accurate, slower). `task.source_freshness_equal_mtime_is_fresh = true` treats equal source/output mtime as fresh.

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

`jdx/mise-action@v4` handles this automatically.

### Error Handling

Tasks run with `set -e` semantics by default (the default inline shell is `sh -c -o errexit`). Disable locally:
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

mise supports 18 backends. The registry assigns tools to backends by an **acceptance-tier** preference order — prefer the highest tier available for a given tool:

| Tier | Backends | When chosen |
|------|----------|-------------|
| **1 — Preferred** | **aqua**, **github**, **gitlab** | aqua offers the most features + security (cosign/SLSA/attestation/minisign, native Windows, no plugin). github/gitlab for releases not yet in aqua. |
| **2 — High bar** | **conda** | mise's conda backend needs no conda/mamba/micromamba installed. |
| **3 — Very high bar** | **pipx**, **npm**, **gem**, **go**, **cargo**, **dotnet** | Language-specific; silently bind tools to whichever runtime was available at install time. |
| **Other** | **forgejo**, **http**, **s3**, **spm**, **pkgx** | forgejo (Codeberg default); http for direct URLs; s3 for private buckets; spm for Swift; pkgx (experimental) for the pkgx pantry. |
| **Not accepted for new registry entries** | **vfox**, **asdf**, **ubi** | vfox/asdf rejected for supply-chain reasons; ubi deprecated. |

| Backend | Status | Description |
|---------|--------|-------------|
| **aqua** | stable | Most features, best security (cosign/SLSA/attestation/minisign). No plugins needed. Native Windows. |
| **github** | stable | GitHub releases with auto OS/arch/libc asset detection |
| **gitlab** | stable | GitLab releases |
| **forgejo** | stable | Forgejo/Codeberg releases |
| **http** | stable | Direct HTTP/HTTPS downloads with URL templating |
| **s3** | stable | S3/MinIO/S3-compatible private buckets (AWS SDK credential chain) |
| **pipx** | stable | Python CLIs in isolated environments (uses uvx by default) |
| **npm** | stable | Node packages (auto-detects `aube`, `npm`, `bun`, `pnpm`) |
| **go** | stable | Go packages (requires compilation) |
| **cargo** | stable | Rust packages (uses binstall by default) |
| **gem** | stable | Ruby gems |
| **conda** | stable | Single conda packages direct from anaconda.org (no conda/mamba needed) |
| **dotnet** | stable | .NET tools |
| **spm** | stable | Swift packages |
| **pkgx** | **experimental** | pkgx pantry packages (bottles from dist.pkgx.dev); requires `experimental = true` |
| **ubi** | **DEPRECATED** | Universal Binary Installer — migrate to `github` |
| **vfox** | stable | vfox plugins (cross-platform, Windows-supported) |
| **asdf** | stable (legacy) | asdf plugins (no Windows) |

**Backend capability matrix:**

| Feature | Core | Lang PMs | aqua | github | Backend Plugins | Tool Plugins | asdf |
|---------|------|----------|------|--------|-----------------|--------------|------|
| Speed | ✅ | ⚠️ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| Security | ✅ | ⚠️ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Windows | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Env Vars | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Custom Scripts | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |

**Backend selection priority:** explicit spec (`aqua:owner/repo`) → `MISE_BACKENDS_<TOOL>` env override → registry lookup → core tools (native Rust: node, python, ruby, go, java) → fallback.

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
mise registry                # list everything
mise registry --backend aqua # filter by backend
mise registry --json --security  # per-backend security info (slower)
mise use                     # interactive selector
mise search <query>          # find tools in registry
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
hk = { version = "latest", os = ["linux", "macos/arm64"] }  # OS/arch combos

# Explicit backend
"aqua:BurntSushi/ripgrep" = "latest"
"github:cli/cli" = "latest"
"npm:prettier" = "latest"
"pipx:psf/black" = "latest"
"cargo:eza" = "latest"
"go:github.com/DarthSim/hivemind" = "latest"

# Table form (best for nested options)
[tools."http:my-tool"]
version = "1.0.0"
platforms.macos-x64.url = "https://example.com/my-tool-macos-x64.tar.gz"
```

CLI tool-option syntax uses brackets: `mise use "conda:ruff[channel=bioconda]"`, `mise use "s3:my-tool[url=s3://bucket/x.tar.gz]@1.0.0"`.

**Version formats:**

| Format | Example | Description |
|--------|---------|-------------|
| Exact | `"20.0.0"` | Specific version |
| Prefix | `"20"` | Latest matching prefix |
| Latest | `"latest"` | Most recent stable |
| `lts` | `"lts"` | LTS release (node) |
| `prefix:<P>` | `"prefix:1.19"` | Latest matching prefix (explicit form) |
| `ref:<SHA>` | `"ref:master"` | Compile from git ref |
| `path:<PATH>` | `"path:./shfmt"` | Use custom binary |
| `sub-<N>:<BASE>` | `"sub-2:lts"` | N versions behind base (`sub-0.1:latest` = 1 minor back) |
| `tag:<TAG>` | `"tag:v1.0.0"` | Cargo git tag |
| `branch:<BRANCH>` | `"branch:main"` | Cargo git branch |
| `rev:<SHA>` | `"rev:abc1234"` | Cargo git rev |

> **Resolution nuance:** in config files, `node@20` means the latest *installed* 20.x. But `mise install node@20`, `mise latest node@20`, and `mise upgrade node@20` resolve to the latest *available* 20.x.

**Structured tool selectors (2026.7.11+)** are uniform across root `[tools]`, inline task defs, task templates, and file-task headers. Exactly one selector is required; conflicting, missing, or non-string selectors are a hard config error:

```toml
node       = { version = "20" }
go         = { prefix = "1.22" }
python     = { ref = "main" }
shellcheck = { path = "/opt/shellcheck" }
```

### Per-Tool Options

Universal options supported by every backend:

| Option | Type | Description |
|--------|------|-------------|
| `version` | string | Tool version |
| `os` | string[] | Restrict to OS/arch: `"linux"`, `"macos"`, `"darwin"`, `"windows"`, `"win"`. Combos: `"linux/x64"`, `"macos/arm64"`. Arches: `arm64`/`aarch64`, `x64`/`x86_64`/`amd64`. |
| `install_env` | table | Environment variables injected during install (and tool-level `postinstall`) |
| `postinstall` | string | Command after install. `MISE_TOOL_INSTALL_PATH` available; the tool's bin dir is on PATH. |
| `depends` | string \| string[] | Tool installation dependencies |
| `github_attestations` | bool | Per-tool toggle for GitHub Artifact Attestation verification (2026.7.0+) |

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

Tool option values support template expansion referencing `env.*` and `vars.*`; nested option arrays/tables are Tera-rendered too (2026.7.7+).

### Backend-Specific Configuration

#### Aqua Backend (Preferred)

```toml
[tools]
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:cli/cli" = { version = "latest", symlink_bins = true }  # Filtered .mise-bins directory
"aqua:flutter/flutter" = { version = "3.32.8", channel = "stable" }
"aqua:scenarigo/scenarigo" = { version = "0.21.0", vars = { go_version = "1.24" } }
```

**Tool options:** `symlink_bins` (bool, filters bundled bins into `.mise-bins`), `vars` (table, aqua registry template variables), `channel`, `prerelease` (bool, include pre-releases in `ls-remote`/`latest`).

**Security (all default `true`):**

| Setting | Default | Description |
|---------|---------|-------------|
| `aqua.cosign` | `true` | Verify cosign signatures |
| `aqua.slsa` | `true` | Verify SLSA provenance |
| `aqua.github_attestations` | `true` | Verify GitHub Artifact Attestations |
| `aqua.minisign` | `true` | Verify minisign signatures |
| `aqua.baked_registry` | `true` | Use built-in (compiled-in) aqua registry |
| `aqua.registries` | none | Extra registry repos (string[], `MISE_AQUA_REGISTRIES`), fetched before the baked-in registry; supports `file://` |
| `aqua.registry_url` | none | **Deprecated** (use `aqua.registries`) |
| `aqua.registry_cache_ttl` | `1w` | Registry source freshness TTL (`0s` = always refresh) |
| `aqua.cosign_extra_args` | none | Additional cosign arguments (string[]) |

Aqua verification is native (no external CLIs needed) covering GitHub attestations, cosign, SLSA, minisign, and SHA256/512/1/MD5 checksums (always on).

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

**Tool options:** `asset_pattern`, `matching` (substring pre-filter; preserves OS/arch autodetection), `matching_regex` (case-sensitive — prefix `(?i)` for insensitive), `version_prefix` (default `v`), `platforms` (per-platform `asset_pattern`/`checksum`/`size`), `strip_components`, `bin`, `rename_exe`, `bin_path` (Tera: `{{version}}`, `{{os}}`, `{{arch}}`, `{{darwin_os}}`, `{{amd64_arch}}`, `{{x86_64_arch}}`, `{{gnu_arch}}`), `filter_bins`, `checksum`, `size`, `no_app`, `api_url`, `prerelease`, `github_attestations`.

> **Multi-binary releases:** the same backend reused with a different `matching` overwrites the prior entry — give each binary a distinct `[tool_alias]` mapping to the repo, then set `matching` + `rename_exe` per tool (e.g. `dhall-json`/`dhall-lsp` both → `github:dhall-lang/dhall-haskell`).

**Auth settings:** `github.credential_command`, `github.gh_cli_tokens` (default `true`), `github.github_attestations`, `github.slsa`, `github.use_git_credentials` (default `false`). OAuth device-flow: `github.oauth_client_id`, `github.oauth_api_url`, `github.oauth_auth_url`, `github.oauth_export_env` (default `GITHUB_TOKEN`), `github.oauth_open_browser`, `github.oauth_scopes`. Env: `MISE_GITHUB_TOKEN`, `MISE_GITHUB_USE_GIT_CREDENTIALS`. Debug a resolved token with `mise token github`.

#### GitLab / Forgejo Backends

Same option surface as GitHub (`asset_pattern`, `matching`, `matching_regex`, `platforms`, `bin`, `bin_path`, `rename_exe`, `filter_bins`, `checksum`, `size`, `strip_components`, `no_app`, `api_url`). Forgejo also supports `prerelease` and defaults `api_url` to `https://codeberg.org/api/v1`.

```toml
"gitlab:gitlab-org/gitlab-runner" = "16.8.0"
"forgejo:user/repo" = "latest"  # Defaults to Codeberg
```

**GitLab auth order:** `MISE_GITLAB_ENTERPRISE_TOKEN` → `MISE_GITLAB_TOKEN` → `GITLAB_TOKEN` → credential_command → gitlab_tokens.toml → `glab` CLI → git credential fill. Settings: `gitlab.glab_cli_tokens` (default `true`), `gitlab.use_git_credentials` (default `false`). Debug with `mise token gitlab [--unmask]`.

**Forgejo auth order:** `MISE_FORGEJO_ENTERPRISE_TOKEN` → `MISE_FORGEJO_TOKEN` → `FORGEJO_TOKEN` → credential_command → forgejo_tokens.toml → `fj` CLI → git credential fill. Setting: `forgejo.fj_cli_tokens` (default `true`).

A `credential_command` receives `MISE_CREDENTIAL_HOST` and `MISE_CREDENTIAL_PROVIDER` env vars (the legacy single positional argument is deprecated after mise 2026.11.0). For security, `*.credential_command` is **global-config-only** — it is stripped from project/local `mise.toml` (with a warning).

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

**Platform keys:** `macos-x64`, `macos-arm64`, `linux-x64`, `linux-arm64`, `windows-x64`, `windows-arm64` (`darwin`/`amd64` variants also accepted).

**Checksums:** `checksum`, plus `checksum_url` (algorithm auto-detected from filename) and `checksum_expr` (expr-lang returning `algo:hash`) for cross-platform lockfile population.

**Version discovery options:**
- `version_list_url` — URL returning version list (text, JSON array, or JSON objects)
- `version_regex` — Regex to extract versions (first capturing group)
- `version_json_path` — jq-like path (e.g., `".[].tag_name"`)
- `version_expr` — expr-lang expression with access to response `body`. **Takes precedence** over regex/json_path.

**Caching:** `$MISE_CACHE_DIR/http-tarballs/` (Blake3 keys); auto-pruned after 30 days.

#### S3 Backend

Install from Amazon S3 / S3-compatible storage (MinIO, DigitalOcean Spaces) — useful for private/internal tools.

```toml
[tools."s3:my-internal-tool"]
version = "latest"
url = "s3://tools-bucket/releases/my-tool-{{ version }}.tar.gz"
endpoint = "http://minio.internal:9000"  # MinIO / S3-compat
region = "us-east-1"
version_list_url = "s3://tools-bucket/releases/versions.json"
bin_path = "bin"
```

**Options:** `url` (required), `endpoint`, `region`, `checksum`, `size`, `bin`, `rename_exe`, `bin_path`, `format`, `strip_components`, `platforms`, `version_list_url`, `version_json_path`, `version_expr`, `version_prefix`, `version_regex`.

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

**Tool options:** `features` (**skips binstall**), `default-features` (default `true`; `false` **skips binstall**), `bin`, `crate` (monorepo select), `locked` (default `true`), `install_env`.

**Settings:** `cargo.binstall` (default `true`), `cargo.binstall_only` (default `false`), `cargo.binstall_quickinstall` (**default `false` since 2026.7.6**), `cargo.binstall_native` (experimental, requires `experimental = true` — native fast path reading `package.metadata.binstall`), `cargo.registry_name`. Exit code 94 from binstall falls back to `cargo install`.

#### Pipx Backend

```toml
[tools]
"pipx:black" = "latest"
"pipx:harlequin" = { version = "latest", extras = "postgres,s3" }
"pipx:ansible" = { version = "latest", uvx = false }  # Disable uv
"pipx:ansible-core" = { version = "latest", uvx_args = "--with ansible" }
"pipx:black" = { version = "latest", pipx_args = "--preinstall" }
```

**Tool options:** `extras`, `pipx_args`, `uvx` (per-tool toggle), `uvx_args`, `install_env`.

**Settings:** `pipx.uvx` (default `true`), `pipx.registry_url` (default `https://pypi.org/pypi/{}/json`).

**Install sources:** PyPI, GitHub (`pipx:psf/black`), Git (`pipx:git+https://...`), HTTP ZIP.

Honors `minimum_release_age` via uv `--exclude-newer` or pipx `--uploaded-prior-to`. Reinstall after Python upgrade: `mise install -f pipx:psf/black`.

#### NPM Backend

```toml
[tools]
"npm:prettier" = "latest"
"npm:some-cli" = { version = "latest", allow_builds = ["esbuild"] }  # approve build scripts
```

**Auto-detects package manager:** prefers the embedded `aube` installer, else `npm`, with `bun`/`pnpm` as alternatives. Override with `npm.package_manager = "auto|npm|aube|bun|pnpm"` (env: `MISE_NPM_PACKAGE_MANAGER`). Because `aube` is embedded, **Node is only required to run the tool, not to install it**.

**Tool options:** `allow_builds` (array of pkgs, or `true` for all — lifecycle build scripts are **denied by default**), `trust_policy_excludes` (exempt reviewed packages from trust-policy downgrade checks), `aube_args`, `npm_args`, `pnpm_args`, `bun_args`, `install_env`.

**Setting `npm.shell_out`** (default `false`, `MISE_NPM_SHELL_OUT`) routes metadata lookups and installs through the npm CLI instead.

**Lifecycle-script defaults:** npm passes `--ignore-scripts=true` (override `npm_args = "--ignore-scripts=false"`); bun runs no dependency scripts unless `bun_args = "--trust"`; aube/pnpm need `allow_builds`.

Supports minimum-release-age protection (transitive supply chain) via compatible package managers (`npm >= 11.10.0`, `aube`, `bun >= 1.3.0`, `pnpm >= 10.16.0`). Honors `~/.npmrc` and `NPM_CONFIG_*`.

#### Go Backend

```toml
[tools]
"go:github.com/DarthSim/hivemind" = "latest"
"go:github.com/golang-migrate/migrate/v4/cmd/migrate" = { version = "latest", tags = "postgres" }
```

**Tool options:** `tags` (string or array → `-tags`), `install_env`. mise sets `GOBIN` to the tool install dir *after* applying `install_env`; use `install_env = { GOPROXY = "direct" }` for unreleased revisions.

#### Gem Backend

```toml
[tools]
"gem:rubocop" = "latest"
```

**Tool option:** `install_env` only. Requires `gem` (Ruby) on PATH. Reinstall after Ruby upgrade: `mise install -f gem:rubocop` or `mise install -f "gem:*"`.

#### Conda Backend

```toml
[tools]
"conda:ruff" = "latest"
"conda:bioconductor-deseq2" = { version = "latest", channel = "bioconda" }
```

Direct anaconda.org API — no conda/mamba/micromamba required. **Single packages only** (not full environments or dependency trees). **Tool option:** `channel` (overrides `conda.channel`, default `conda-forge`). Platforms auto-detected (linux-64, linux-aarch64, osx-64, osx-arm64, win-64) with `noarch` fallback.

#### SPM Backend

```toml
[tools]
"spm:tuist/tuist" = "latest"
"spm:swiftlang/swiftly" = { version = "latest", filter_bins = ["swiftly"] }
```

**Tool options:** `filter_bins`, `artifactbundle` (`true` = require prebuilt bundle, `false` = always build from source, unset = try bundle then source), `artifactbundle_asset`, `provider` (`github`/`gitlab`), `api_url`, `install_env`. **Setting:** `spm.artifactbundle_only`.

#### Dotnet Backend

```toml
[tools]
"dotnet:GitVersion.Tool" = "5.12.0"
"dotnet:GitVersion.Tool" = { version = "latest", prerelease = true }
```

**Tool options:** `prerelease`, `install_env`.

**Settings:** `dotnet.registry_url` (default `https://api.nuget.org/v3/index.json`), `dotnet.isolated` (default `false`), `dotnet.cli_telemetry_optout`, `dotnet.dotnet_root`. `dotnet.package_flags` is **deprecated** — use the `prerelease` tool option instead.

#### Pkgx Backend (Experimental)

Installs packages from the pkgx pantry without shelling out to the pkgx CLI (bottles fetched from `dist.pkgx.dev`, checksums verified). Requires `experimental = true`.

```toml
[tools]
"pkgx:stedolan.github.io/jq" = "1.7.1"
```

Supports npm-style semver ranges; runtime env comes from pantry manifests via generated wrappers. Lockfile entries are recorded under `[pkgx-packages]`; use `mise lock` / `mise install --locked`.

#### UBI Backend (Deprecated)

**Options:** `exe`, `rename_exe`, `matching`, `matching_regex`, `provider`, `api_url`, `extract_all`, `bin_path`, `tag_regex`.

> **Migration gotcha:** ubi includes `matching` in the install path (so one repo can supply several binaries), but the `github` backend keys install paths by tool name + version only — different `matching` values would collide. Give each binary its own `[tool_alias]` entry.

#### vfox & asdf Plugins

**vfox (recommended):** cross-platform (Win/macOS/Linux), built-in Lua interpreter with HTTP/JSON/archive modules, attestation verification, lock files. **Tool option:** `install_env` (applies to `cmd.exec` during install hooks only).

**asdf (legacy):** Unix-only, bash scripts, needs curl/jq, no Windows. New asdf tools rarely accepted for supply-chain reasons. **Tool option:** `install_env`.

> Tool plugins support attestation verification; **backend plugins do not** — a known security gap.

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
- **Shims**: env vars only load when a shim is called; `[env]` and `watch_files` unsupported; only `preinstall`/`postinstall` hooks work; `which` points to the shim (use `mise which` for the real path)
- **PATH**: full environment, all hooks (`cd`, `enter`, `leave`, `watch_files`), `which` shows the actual binary
- **Recommendation**: PATH for interactive shells; shims for IDEs/cron/CI/non-interactive

**`mise reshim`** regenerates the shim directory (`-f/--force` removes all shims first). Runs automatically on install/update/remove. Never manually drop binaries in the shims dir — they get deleted.

### Shell Completion (per-directory tab-complete)

Combined with `mise activate`, shell completions make `mise <TAB>` and `mise run <TAB>` automatically list the tasks/tools defined in the current directory's `mise.toml`.

```bash
# bash
mise completion bash > /etc/bash_completion.d/mise
# zsh (ensure the dir is on $fpath)
mise completion zsh > /usr/local/share/zsh/site-functions/_mise
# fish
mise completion fish > ~/.config/fish/completions/mise.fish
# powershell
mise completion powershell | Out-String | Invoke-Expression
```

Flag: `--include-bash-completion-lib` bundles bash-completion helpers into the script.

> **Standalone usage scripts** need a separate one-time opt-in for completion: `source <(usage g completion-init bash)` in `~/.bashrc` (zsh: same in `~/.zshrc`; fish: `usage g completion-init fish | source`). Since usage 3.5.6 the completion spec cache moved to `${XDG_CACHE_HOME:-$HOME/.cache}/usage` — **regenerate completion scripts** to pick it up.

**Tool aliases (remap to different backend):**
```toml
[tool_alias]
node = 'github:company/our-custom-node'
erlang = 'aqua:company/our-custom-erlang'

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

Manage from the CLI with `mise shell-alias get|set|unset|ls` and `mise tool-alias get|set|unset|ls`.

**Custom plugin repos (`[plugins]`):**
```toml
[plugins]
node = "https://github.com/myorg/asdf-node#v2"  # optional #GITREF suffix
my-tool = "vfox:myorg/vfox-my-tool"             # asdf:/vfox:/vfox-backend: prefixes
```

> The `[_]` table holds arbitrary user data that mise ignores during parsing — useful for sharing values with external tooling.

### Lockfiles (`mise.lock`)

mise can pin tool URLs/checksums in a `mise.lock` file for reproducibility. Enable with `lockfile = true`, then `touch mise.lock && mise install`.

```toml
[[tools.node]]
version = "20.11.0"
backend = "core:node"

[tools.node.platforms.linux-x64]
checksum = "sha256:a6c2..."
size = 23456789
url = "https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-x64.tar.xz"

[[tools.ripgrep]]
version = "14.1.1"
backend = "aqua:BurntSushi/ripgrep"
options = { exe = "rg" }
```

```bash
mise lock                       # Update lockfile from current installs
mise lock node python           # Update specific tools only
mise lock --bump                # Advance fuzzy selectors (latest/lts/"20") without installing
mise lock --bump --dry-run --json
mise lock -p linux-x64          # Add/update entries for specific platform(s)
mise lock --local               # Update mise.local.lock instead of mise.lock
mise lock -g                    # Target global config lockfiles
mise lock --minimum-release-age "30d"
```

> `mise lock --bump` re-resolves fuzzy version selectors **without installing or touching `mise.toml`**; exact pins are left alone.

Per-config lockfiles pair with their config: `mise.toml`→`mise.lock`, `mise.test.toml`→`mise.test.lock`, `mise.local.toml`→`mise.local.lock` (gitignore the local one).

**Backend lockfile support:** full (version+checksum+size+URL+provenance) — `aqua`, `http`, `github`, `gitlab`; partial — `vfox` (version+URL+provenance), `ubi` (version+checksum+size); basic — `core` (version+checksum); version only — `asdf`, `npm`, `cargo`, `pipx`.

Settings: `lockfile` (read/update lockfiles), `locked = true` (require lockfile-resolved URLs; blocks API calls — good for CI), `locked_verify_provenance = true` (re-verify SLSA/cosign/minisign/attestations at install; auto-on with `paranoid`), `lockfile_platforms = ["linux-x64", "macos-arm64"]` (current platform always included).

`minimum_release_age` setting (env `MISE_MINIMUM_RELEASE_AGE`, **default `24h`**, e.g. `"7d"`, `"6mo"`, `"2026-01-01"`) filters fuzzy version requests by release date — supply-chain delay protection. Exempt specific tools with `minimum_release_age_excludes = ["node", "..."]`. Supported backends: aqua, cargo, npm, pipx. Already-installed versions stay eligible. Note durations use jiff format — months are `6mo`, not `6m`.

CI flag: `--locked` (global flag on any command) errors if any version isn't resolved via the lockfile.

### Auto-Install Controls

- `auto_install` (default `true`) — master switch
- `exec_auto_install` (default `true`) — `mise x`/`mise r`
- `task.run_auto_install` (default `true`)
- `not_found_auto_install` (default `true`) — requires at least one existing version
- `auto_install_disable_tools = ["..."]` — per-tool skip list

### CLI Commands for Tools

```bash
mise use node@22            # Install + activate + write to mise.toml
mise use -g node@22         # Write to global config
mise use --pin node@22      # Pin exact resolved version (e.g. 22.5.1)
mise use -E staging node@22 # Write to mise.staging.toml
mise use --remove python    # Remove a tool from config
mise use --dry-run-code     # Exit 1 if changes would be made
mise install node@20        # Install without activation
mise install                # Install all configured tools
mise install -f "gem:*"     # Force reinstall pattern
mise install --monorepo     # Install across all monorepo config roots
mise ls                     # List installed
mise ls --current           # Active versions only
mise ls --prunable          # Tools eligible for prune
mise ls --outdated          # Tools with newer versions
mise ls --monorepo          # Across monorepo config roots
mise ls-remote node         # List available versions
mise ls-remote --prerelease # Include pre-releases
mise latest node            # Latest available version (no install)
mise which node             # Show real binary path
mise where node@22          # Show install directory
mise bin-paths              # List active runtime bin paths
mise uninstall node@20      # Remove an installed tool version
mise link node@custom ./dir # Symlink an external install into mise
mise x python@3.12 -- script.py  # Run with specific tool
mise reshim                 # Rebuild shims
mise registry               # List all available tools
mise backends ls            # List available backends
mise fmt                    # Format mise.toml (sort keys, clean whitespace)
mise outdated               # Check for updates
mise upgrade                # Update versions (respects mise.toml ranges)
mise upgrade --bump         # Bump mise.toml to absolute latest
mise upgrade -i             # Interactive selection
mise prune                  # Remove unused versions
mise lock                   # Update lockfile checksums/URLs
mise search <query>         # Search registry (-m equal|contains|fuzzy)
mise cache clear            # Clear cached downloads
mise sync node --nvm        # Import versions from nvm
mise sync python --uv       # Import versions from uv
mise sync ruby --brew       # Import versions from brew
mise token github           # Show resolved host token (--unmask to reveal)
mise generate bootstrap -w ./bin/mise   # Self-contained install script for CI
mise generate task-stubs    # Generate task stub wrappers (bin/)
mise generate task-docs     # Generate markdown docs for tasks
mise generate config        # Generate sample config
mise generate github-action # Generate sample GitHub Action workflow
mise generate devcontainer  # Generate devcontainer spec
mise generate git-pre-commit # Generate pre-commit hook
mise generate tool-stub     # Generate a standalone tool stub
mise doctor                 # Diagnose installation issues (mise doctor path)
mise self-update            # Update mise binary
mise mcp                    # Run mise as a Model Context Protocol (MCP) server
mise dotfiles               # Manage dotfile synchronization
mise bootstrap              # Provision a whole machine
mise oci build|push|run     # Build/inspect OCI container images with mise tools
mise deps add|install|remove  # Project dependency preparation
mise en                     # Spawn a sub-shell with the project env loaded
mise test-tool <tool>       # Verify a tool installs/builds correctly
```

---

## Environment Configuration

### Basic Variables

```toml
[env]
NODE_ENV = 'production'
DEBUG = 'app:*'
PORT = 3000

# Default — applied only when the var is unset/empty (existing non-empty values win).
EDITOR = { default = 'vim' }

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
mise env --json-extended        # JSON with source + tool attribution
mise env --dotenv               # Export as dotenv
mise env --redacted             # Show only redacted variables
mise env --values               # Show only values
mise env -s bash                # Shell-specific output
```

Env vars resolve **before** tools. Use `tools = true` on a value to defer evaluation until tool paths/versions exist.

### Special Directives (`env._`)

The reserved key `_` is used as a TOML table for configuration since nested env vars don't make sense.

> **Deprecation:** the older `env.mise.*` spelling is deprecated (removal **2026.12.0**) — use `env._.*`. Likewise, the `value`/`values` keys inside `_.file`/`_.path`/`_.source` objects are deprecated in favor of `path` (which accepts a string *or* an array); removal **2026.12.0**.

#### `_.path` — Prepend to PATH

```toml
[env]
_.path = ["tools/bin", "{{config_root}}/scripts"]
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }  # Lazy eval after tools
```

Relative paths resolve against `{{config_root}}`. Options: `path`, `tools`.

#### `_.file` — Load from .env/json/yaml/toml files

```toml
[env]
_.file = '.env'
_.file = ['.env', '.env.local', '.env.{{env.MISE_ENV}}']
_.file = { path = ".secrets.yaml", redact = true }
```

Supported formats: `.env`, `.env.json`, `.env.yaml`, `.env.toml` (plus sops/age-encrypted variants). Auto-load a single dotenv with `MISE_ENV_FILE=.env` (or the `env_file` setting).

Options: `path`, `redact`, `tools`.

> **Deprecation:** the top-level `env_file`/`dotenv` and `env_path` keys are deprecated (removal **mise 2027.4.0**). Migrate to `_.file` and `_.path`.

#### `_.source` — Source shell scripts

```toml
[env]
_.source = "./setup-env.sh"
_.source = { path = "my/env.sh", redact = true }
_.source = { path = "my/env.sh", tools = true }
```

Scripts execute with `source` semantics; shebangs are ignored. On Windows this requires a POSIX bash (Git for Windows / MSYS2).

#### `_.python.venv` — Auto-activate Python venv

```toml
[env]
_.python.venv = ".venv"                                  # string shorthand
_.python.venv = { path = ".venv", create = true }
_.python.venv = { path = ".venv", python = "3.12" }      # specify python version
_.python.venv = { path = ".venv", create = true, uv_create_args = ["--seed"] }  # uv venv with pip
```

| Option | Type | Purpose |
|--------|------|---------|
| `path` | string | venv location (required; templates supported) |
| `create` | bool | Auto-create if missing (default `false`) |
| `python` | string | Python version used to create the venv |
| `python_create_args` | string[] | Args passed to `python -m venv` |
| `uv_create_args` | string[] | Args passed to `uv venv` |

When `uv` is installed via mise it is used automatically (uv omits pip by default — add `uv_create_args = ["--seed"]`). Activation needs `mise activate`/`mise exec`; shims alone don't add the venv `bin/` to PATH.

> This is a **separate codepath** from the `python.uv_venv_auto` setting — `uv_create_args` here is not used by `uv_venv_auto`. For uv-managed projects (with `uv.lock`), prefer `python.uv_venv_auto` (`"source"` or `"create|source"`). The legacy `virtualenv` tool option is deprecated in favor of `_.python.venv`.

#### Plugin-Provided Directives

Plugins can register custom `_.<name>` directives (options land in the plugin's `ctx.options`):

```toml
[env]
_.my-plugin = {}
_.my-plugin = { option1 = "value1", option2 = "value2" }
_.vault-secrets = { vault_url = "https://vault.example.com", secrets_path = "secret/myapp" }
```

#### Multiple Identical Directives

TOML doesn't allow duplicate keys, so use array-of-tables:

```toml
[[env]]
_.source = "./script_1.sh"
[[env]]
_.source = "./script_2.sh"
```

> Array parsing was tightened in 2026.7.12: accidental array forms for `vars` and task `env`/`vars` are rejected. Intentional `[[env]]` and directive-level arrays (`_.source`, `_.file`, `_.path`) remain supported.

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

Comma-separated supports multiple profiles: `MISE_ENV=ci,test` (rightmost wins). Also recognized: `mise/config.{MISE_ENV}.toml`, `.config/mise.{MISE_ENV}.toml`.

`.miserc.toml` lookup order: cwd + parents → `~/.config/mise/miserc.toml` → `/etc/mise/miserc.toml`. It has limited Tera context — `env.*`, `config_root`, `cwd`, `xdg_*`, all filters/tests, and all functions **except** `exec()` and `read_file()`; `mise_env`, `mise_bin`, and `mise_pid` are unavailable.

**Platform auto-envs** (`auto_env` setting): `unix`; `linux`/`macos`/`windows`; `linux-x64`/`macos-arm64`/`windows-x64`. Files like `mise.windows.toml` and `mise.macos-arm64.toml` load automatically with matching lockfiles (`mise.windows.lock`). Precedence: `unix` < `{os}` < `{os}-{arch}` < explicit `MISE_ENV` entries. **Disabled by default today; defaults to enabled in mise 2027.6.0.**

### Required and Redacted Variables

```toml
[env]
# Required — error if not set (warns during shell activation, doesn't block)
DATABASE_URL = { required = true }
DATABASE_URL = { required = "Set postgres connection string" }

# Redacted — hidden from output
API_KEY = { value = "secret_key_here", redact = true }

# Pattern-based redactions (top-level, not under [env])
redactions = ["*_TOKEN", "SECRET_*", "API_*"]
```

Redaction requires a non-`raw` output mode — tasks with `raw = true` bypass interception.

### Secrets (fnox, SOPS, age)

mise documents three approaches to secrets:

1. **fnox (recommended)** — a separate [@jdx](https://github.com/jdx) project: a full secret manager with remote storage (1Password, AWS Secrets Manager) and remote encryption (AWS KMS). Works standalone; no mise-specific integration required.
2. **SOPS** — encrypt entire files, load via `env._.file`.
3. **Direct age** — encrypt individual env vars inline in `mise.toml`.

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

**Key resolution:** `MISE_SOPS_AGE_KEY` → `MISE_SOPS_AGE_KEY_FILE`/`sops.age_key_file` → `SOPS_AGE_KEY_FILE` → `SOPS_AGE_KEY` → `~/.config/mise/age.txt`.

**Settings:** `sops.age_key`, `sops.age_key_file` (default `~/.config/mise/age.txt`), `sops.age_recipients`, `sops.rops` (default `true`, native Rust impl), `sops.strict` (default `true`).

> The external `sops` CLI does not support TOML I/O — use JSON/YAML with the CLI, or set `sops.rops = true` for encrypted TOML. age is currently the only supported sops encryption method.

**Direct age encryption (experimental):**

```bash
mise settings set experimental=true
mise set --age-encrypt DB_PASSWORD=supersecret
mise set --age-encrypt --prompt DB_PASSWORD
```

Flags: `--age-encrypt`, `--age-recipient <KEY>` (repeatable), `--age-ssh-recipient <PATH|KEY>` (repeatable), `--age-key-file <PATH>`, `--no-redact`.

Stored in mise.toml (an optional `format` of `"raw"` or `"zstd"` may appear — zstd is used automatically for values larger than ~1KB):
```toml
[env]
DB_PASSWORD = { age = { value = "<base64>" } }
```

**Settings:** `age.identity_files`, `age.key_file` (default `~/.config/mise/age.txt`), `age.ssh_identity_files`, `age.strict` (default `true`). **Decryption key resolution:** `MISE_AGE_KEY` → `age.identity_files` → `age.key_file` → `~/.config/mise/age.txt` → SSH identities.

Decrypted values are always marked for redaction.

### Templates (Tera)

mise.toml values support Tera templates. **mise 2026.7.1+ uses Tera v2 by default.**

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
| `mise_env` | Vec | Configuration environments from `MISE_ENV` (undefined if unset) |
| `tools` | HashMap | Installed tool info (`.version`, `.path`). Requires `tools = true` on task templates. |
| `usage` | HashMap | Task arguments/flags (task run scripts only). Hyphenated: `usage["dry-run"]`. |
| `vars` | HashMap | Values from `[vars]` |
| `xdg_cache_home` / `xdg_config_home` / `xdg_data_home` / `xdg_state_home` | PathBuf | XDG directories |

**Key Tera functions:**

| Function | Description |
|----------|-------------|
| `exec(command, [cache_key], [cache_duration])` | Execute shell command, return stdout (runs even during `--dry-run`) |
| `get_env(name, [default])` | Get env var from the original process env |
| `arch()` | System architecture (`x64`, `arm64`) |
| `os()` | Operating system (linux, macos, windows) |
| `os_family()` | OS family (unix/windows) |
| `num_cpus()` | CPU count |
| `now([timezone])` | Current datetime (IANA tz names) |
| `choice(n, alphabet)` | Random n-char string |
| `read_file(path)` | Read file contents |
| `range(end, [start], [step_by])` | Integer array |
| `get_random(start, end, [seed])` | Random integer |
| `task_source_files()` | Resolved source file paths (task scripts only) |
| `throw(message)` | Raise error |

**Key Tera filters:**

| Filter | Description |
|--------|-------------|
| `lower`, `upper`, `capitalize`, `title` | Case transforms |
| `kebabcase`, `snakecase`, `shoutysnakecase`, `lowercamelcase`, `uppercamelcase` | Case conversion |
| `slug` / `slugify`, `striptags`, `spaceless` | Text normalization |
| `trim`, `trim_start`, `trim_end`, `truncate([length])` | Whitespace / shortening |
| `replace(from, to)`, `regex_replace(pattern, rep)` | Substitution |
| `quote` | Escape and quote string |
| `split(pat)`, `join(sep)`, `concat(with)`, `shuffle([seed])` | Array operations |
| `first`, `last`, `length`, `reverse` | Collection operations |
| `map(attribute)` | Extract attribute from objects |
| `basename`, `dirname`, `extname`, `file_stem`, `join_path` | Path operations |
| `absolute`, `canonicalize` | Path resolution (canonicalize throws if missing) |
| `file_size`, `last_modified` | File metadata |
| `hash([algorithm], [len])` | SHA256 (default) or BLAKE3 hashing |
| `hash_file([len])` | File BLAKE3 hash |
| `b64_encode`, `b64_decode`, `json_encode([pretty])` | Encoding |
| `date(format, [timezone])` | Format datetime (chrono strftime) |
| `default(value)` | Fallback for undefined/empty |
| `abs`, `filesize_format`, `format(spec)` | Numeric formatting |
| `urlencode`, `urlencode_strict` | URL-safe encoding |

**Tera tests:**

| Test | Description |
|------|-------------|
| `defined` | Variable exists |
| `string`, `number` | Type checks |
| `starting_with(arg)`, `ending_with(arg)`, `containing(arg)`, `matching(regex)` | String checks |
| `dir`, `file`, `exists` | Path checks (mise custom) |
| `odd`, `even`, `divisible_by(n)` | Numeric checks |

**Template syntax:**
- `{{ }}` — Expressions · `{% %}` — Statements · `{# #}` — Comments · `{% raw %} {% endraw %}` — Skip rendering
- Operators: `+`, `-`, `/`, `*`, `%`, `==`, `!=`, `>=`, `<=`, `and`, `or`, `not`, `~` (concat), `in`

**Tera v1 → v2 migration:** `trim_start_matches()` → `trim_start()`; array slicing `items[0:2]`; comprehensions `[item.name for item in items if item.active]`; optional chaining `env?.NODE_ENV`; ternaries `"prod" if release else "dev"`.

```toml
[settings]
tera_v1 = true    # escape hatch — deprecated; warns 2026.10.0, removed 2027.4.0
```

**Shell-style variable expansion** — `env_shell_expand` **now defaults to `true`** (since mise 2026.7.0):
```toml
[env]
LD_LIBRARY_PATH = "$MY_LIB:$LD_LIBRARY_PATH"
PATH_SAFE = "${VAR:-default}"      # With default
CLEAN = "${UNSET_VAR:-}"           # Empty if unset (no warning)
```

Supported forms: `$VAR`, `${VAR}`, `${VAR:-default}`, `${VAR:-}`. Opt out with `env_shell_expand = false`.

---

## Hooks and Watchers

### Hook Types

| Hook | Trigger | Requires `mise activate`? |
|------|---------|--------------------------|
| `cd` | Any directory change (including within the project) | Yes |
| `enter` | Enter a project (once per entry; not re-fired for subdirs) | Yes |
| `leave` | Leave a project (once) | Yes |
| `preinstall` | Before tool installation | No |
| `postinstall` | After tool installation (fires even on no-op installs) | No |

### Syntax

```toml
[hooks]
cd = "echo 'changed directory'"
enter = "echo 'entered project'"
leave = "echo 'left project'"
preinstall = "echo 'about to install'"
postinstall = "echo 'installed'"

# Inline object form ({ run = "..." } is equivalent to the string shorthand)
enter = { run = "echo 'entered project'" }
enter = { run = "echo unix", run_windows = "echo win" }  # OS-specific variant
postinstall = { run = "echo installed", shell = "bash -c" }  # explicit inline shell

# Multiple hooks (each `run` spawns its own subprocess)
enter = ["echo 'first'", { run = "echo 'second'" }]

# Multiline run = ONE subprocess
[hooks.enter]
run = """
echo one
echo two
"""

# Shell hooks (execute IN the current shell; shell = selector bash|zsh|fish)
[hooks.enter]
shell = "bash"
script = "source completions.sh"   # `scripts = [...]` for multiple

# Task hooks
[hooks]
enter = { task = "setup" }
enter = ["echo 'entering'", { task = "setup" }]  # Mixed syntax

# Array-of-tables form
[[hooks.cd]]
run = "echo 'I changed directories'"
[[hooks.cd]]
run = "echo 'I also changed directories'"
```

`run`/`run_windows` must be **strings** — arrays are unsupported.

**Important:** Shell hooks don't auto-cleanup on directory exit like `[env]` does. mise executes literal shell code without tracking it, so exported vars, aliases, and sourced files persist — reverse them manually in a corresponding `leave` hook.

> **Deprecation:** the *spawned* `script`/`scripts` table form is deprecated in favor of `run` — warns **2026.9.0**, removed **2027.3.0**. Current-shell `shell` + `script`/`scripts` for `cd`/`enter`/`leave` is unaffected. For `preinstall`/`postinstall`, always use `run`; a `shell` selector is ignored there.
>
> Since 2026.7.8, non-string `postinstall` hooks and unknown table fields in hook definitions are **rejected at parse time**.

### Watch Files

```toml
[[watch_files]]
patterns = ["src/**/*.rs"]
run = "cargo fmt"

[[watch_files]]
patterns = ["*.js"]
run = "eslint --fix ."
shell = "bash -c"

# Task reference
[[watch_files]]
patterns = ["uv.lock"]
task = "sync-deps"
```

Fields: `patterns` (glob array), `run`, `task`, `shell` (applies to `run` only). Each entry requires **either** `run` or `task`, not both. Sets `MISE_WATCH_FILES_MODIFIED` (colon-separated, literal colons backslash-escaped). Requires watchexec (`mise use -g watchexec@latest`) and `mise activate`.

### Hook Environment Variables

All hooks receive:
- `MISE_ORIGINAL_CWD` — user's working directory at hook fire
- `MISE_PROJECT_ROOT` — detected project root

CD/enter/leave hooks additionally receive:
- `MISE_PREVIOUS_DIR` — previous directory

Config-level `postinstall` receives:
- `MISE_INSTALLED_TOOLS` — JSON array of installed tools

Tool-level `postinstall` additionally receives:
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

## Sandboxing and Safe Mode

Two independent mechanisms: **sandboxing** restricts what a task/exec can touch; **safe mode** makes mise itself refuse to execute anything a config asks for.

### Sandboxing (`[settings.sandbox]`)

Set deny-by-default policy globally, then re-open specific access per task or per invocation:

```toml
[settings.sandbox]
deny_all = true      # or individually:
deny_read = true
deny_write = true
deny_net = true
deny_env = true
```

Env vars: `MISE_SANDBOX_DENY_ALL`, `MISE_SANDBOX_DENY_READ`, `MISE_SANDBOX_DENY_WRITE`, `MISE_SANDBOX_DENY_NET`, `MISE_SANDBOX_DENY_ENV`. Applies to `mise run` and `mise exec`.

Per-task fields (see [Task Configuration](#safety--sandboxing)) and the matching `mise run`/`mise exec` flags compose with these:

```toml
[tasks.fetch-deps]
deny_all = true
allow_net = ["registry.npmjs.org"]
allow_read = ["{{config_root}}"]
allow_write = ["{{config_root}}/node_modules"]
run = "npm ci"
```

```bash
mise run build --deny-net --allow-read /src
mise exec -- --deny-all ./untrusted-script.sh
```

`--deny-env` still passes through `PATH`, `HOME`, `USER`, `SHELL`, `TERM`, and `LANG`. `--allow-env <VAR>` implies deny for everything else and supports wildcards.

### Safe Mode (`safe` / `MISE_SAFE=1`)

**Global-config-only** setting (mise 2026.7.12+) that turns mise into an inert config reader — useful for untrusted repos, fork PRs, and CI that only needs to read versions.

```bash
MISE_SAFE=1 mise lock --bump --dry-run --json
```

In safe mode mise **errors** (never silently falls back) on: template `exec()`/`read_file()`, `_.source`, hooks, tasks, asdf plugin scripts, and plugin installs. It **ignores** project `[env]`, `_.path`, `_.file`, `[shell_alias]`, and `[settings]`. Because the config cannot do anything, safe mode also **skips the trust requirement entirely**.

CI docs also reference `MISE_SAFE=1` as the recommended posture for untrusted configs.

### Trust

Outside CI, untrusted configs must be approved with `mise trust` (`mise trust --all`, `--untrust`, `--show`).

- mise **auto-trusts** configs when it detects a CI environment — unless `MISE_PARANOID=1`.
- Since v2026.6.6, **safe** `mise.toml` files (no templates; only `min_version` and plain `[tools]`/`[tasks]` string values) auto-load without a trust prompt; anything with templates or richer constructs still requires trust.
- Since 2026.7.5, a config in a linked **git worktree** is auto-trusted if the equivalent path in the main checkout is trusted (one-way; `--ignore` still wins; excluded under paranoid mode). `mise trust --all` walks subdirectories, respecting `.gitignore` and skipping hidden dirs, `node_modules`, `vendor`, `target`, `dist`, `build`.
- Trust-sensitive keys (`ci`, `paranoid`, `trusted_config_paths`, `yes`) are ignored when set from project/local config — only global config, CLI flags, and env vars apply.

---

## Machine Bootstrap (Developer Setup)

`mise bootstrap` provisions an entire machine from declarative config — system packages, git repos, dotfiles, shell activation, macOS defaults, services, login shell, tools, and a custom `bootstrap` task. **Stable since v2026.7.4** (no longer requires `MISE_EXPERIMENTAL`).

```bash
mise bootstrap                    # run the full setup
mise bootstrap -n                 # --dry-run: preview without changing anything
mise bootstrap -y                 # --yes: skip confirmation prompts
mise bootstrap --update           # refresh package-manager metadata (and repos) first
mise bootstrap --force-dotfiles   # overwrite conflicting whole-file dotfiles
mise bootstrap --only packages,dotfiles
mise bootstrap --skip macos-defaults
mise bootstrap status [--json] [--missing]   # non-zero exit when out of sync
```

`--only` / `--skip` parts: `plugins`, `packages`, `repos`, `dotfiles`, `mise-shell-activate`, `macos-defaults`, `macos-launchd-agents`, `linux-systemd-units`, `user`, `tools`, `task`, `final-hook`.

**Execution order (13 steps):**

1. `[bootstrap.plugins]` — vfox plugins acting as package managers
2. `[bootstrap.packages]` — system packages
3. `[bootstrap.repos]` — git checkouts
4. `[dotfiles]` — `mise dotfiles apply`
5. `[bootstrap.mise_shell_activate]` — shell rc activation snippets
6. macOS defaults
7. macOS LaunchAgents
8. Linux systemd user units
9. `[bootstrap.user]` — login shell
10. `[tools]` — `mise install`
11. Plugin package managers (post-tools)
12. `mise run bootstrap` task (if defined)
13. `[bootstrap.hooks.final]`

Each phase is also runnable on its own: `mise bootstrap packages apply`, `mise bootstrap macos defaults`, `mise bootstrap linux systemd-units`, `mise bootstrap repos update`, `mise bootstrap user apply`, etc. `mise bootstrap packages brew tap|untap` manages third-party Homebrew taps; `mise bootstrap packages import|prune` syncs brew formulae.

> System-package installs are gated by `[settings.system_packages]` (`sudo`, `managers`). The related `system_deps` setting (`prompt` default, `auto`, `warn`, `ignore`) controls how vfox `PLUGIN.systemDependencies` are surfaced — detection never fails an install.

### `[bootstrap]` Configuration

```toml
# System packages — prefix with the manager: apk: apt: dnf: pacman: brew: brew-cask: flatpak: mas:
[bootstrap.packages]
"apt:build-essential" = "latest"
"apk:curl" = "*"                    # Alpine apk (version: "@2.45.2-r0" form)
"brew:postgresql@17" = "latest"
"brew-cask:firefox" = "latest"      # app-bundle casks, no local Homebrew required
"flatpak:org.gimp.GIMP" = "latest"
"mas:497799835" = "latest"          # Mac App Store apps by ADAM ID

# Declarative git checkouts (keys may be relative paths, resolved against the project root)
[bootstrap.repos]
"~/src/dotfiles" = { url = "git@github.com:jdx/dotfiles.git", ref = "main" }

# Shell activation snippets written into rc files (marker-delimited)
[bootstrap.mise_shell_activate]
zprofile = "shims"
zshrc = "activate"
fish = "activate"

# macOS — high-level convenience blocks…
[bootstrap.macos.dock]
autohide = true
orientation = "left"
tilesize = 48

[bootstrap.macos.finder]
show_pathbar = true

[bootstrap.macos.keyboard]
key_repeat = 2
initial_key_repeat = 15

[bootstrap.macos.trackpad]
tap_to_click = true

# …or raw user defaults by domain
[bootstrap.macos.defaults]
"com.apple.finder" = { AppleShowAllFiles = true }

# Services (launchd on macOS, systemd on Linux)
[bootstrap.macos.launchd.agents.my-sync]
program = "~/.local/bin/my-sync"
args = ["--watch"]
run_at_load = true
start_calendar_interval = { hour = 3, minute = 30 }

[bootstrap.linux.systemd.units.my-sync]
description = "sync files"
exec_start = "~/.local/bin/my-sync --watch"
restart = "on-failure"
type = "simple"
remain_after_exit = false
private_tmp = true

# systemd user TIMERS (2026.7.7+) — rendered as .timer units
[bootstrap.linux.systemd.units.dotfiles-maintain-timer]
unit = "dotfiles-maintain"   # bare key → dev.mise.dotfiles-maintain.service
on_calendar = "daily"
persistent = true

# Converge the login shell (runs chsh, updates /etc/shells when needed)
[bootstrap.user]
login_shell = "/bin/zsh"

# Lifecycle hooks — run may be a string or an array; all honor --dry-run
# Phases: pre-packages, post-packages, pre-repos, post-repos, pre-dotfiles, post-dotfiles,
#         pre-defaults, post-defaults, pre-user, post-user, pre-tools, post-tools, final
[bootstrap.hooks.pre-packages]
run = "softwareupdate --install-rosetta --agree-to-license"
[bootstrap.hooks.post-tools]
run = [
  "mise exec -- corepack enable",
  "mise exec -- rustup component add rustfmt clippy",
]
[bootstrap.hooks.post-defaults]
run = "killall Dock || true"

[tools]
node = "lts"
python = "3.12"

# Custom task — the last app-level step, after tools install
[tasks.bootstrap]
run = "gh auth status || gh auth login"
```

Declarative steps converge idempotently; the `bootstrap` **task runs every invocation** (write it to be idempotent). Unpinned `[bootstrap.repos]` entries are never pulled by a plain apply — use `mise bootstrap repos update` for the explicit fetch + fast-forward. Switching a systemd unit name between service and timer stops/disables/removes the stale sibling.

### Declarative Dotfiles (`[dotfiles]`)

Manage dotfiles declaratively; applied during `mise bootstrap` (step 4) or standalone via `mise dotfiles apply`. **Stable since v2026.7.4.** Entries are keyed by target path.

```toml
[settings]
dotfiles.root = "~/.dotfiles"      # default source root
dotfiles.default_mode = "symlink"  # symlink|symlink-each|copy|template

[dotfiles]
"~/.zshrc" = {}                                   # source mirrors target under dotfiles.root
"~/.gitconfig" = "dotfiles/gitconfig"             # string shorthand = source path
"~/.config/alacritty.toml" = { mode = "copy" }
"~/.ssh/config" = { source = "dotfiles/ssh_config.tmpl", mode = "template" }
"~/.local/bin" = { source = "dotfiles/bin", mode = "symlink-each" }
"~/.config/*.toml" = "dotfiles/config/*.toml"     # glob: * ** ? [ab] (target must match)
# Block / line edits to files mise does NOT own (key = target/edit-id)
"~/.zshrc/activate" = { block = 'eval "$(mise activate zsh)"' }
"/etc/hosts/dev" = { line = "127.0.0.1 dev.local" }
"~/.gitconfig/identity" = { source = "snippets/git-identity.tmpl", template = "tera" }
```

**Modes:** `symlink` (default; one link for a file or whole directory) · `symlink-each` (directory source → per-file links, so the target dir can also hold unmanaged files) · `copy` (a real file/dir, for tools that rewrite their config in place) · `template` (render the source through mise's Tera engine with `env`, `vars`, `exec()`; permissions mirror the source).

**Source resolution:** omitting `source` mirrors the home-relative target path under `dotfiles.root` (`~/.zshrc` → `~/.dotfiles/.zshrc`). Relative explicit sources resolve against the config file's directory. Targets outside `$HOME` require an explicit `source`.

**Block edits** wrap content in marker comments (`# >>> mise:id >>>` … `# <<< mise:id <<<`); the comment style is inferred per file type (`#`, `--`, `//`, `;`, `"`) and can be overridden with `comment`. Strict JSON/XML cannot use blocks. **Line edits** append a single line if absent and never modify other content. Edit IDs allow letters, digits, `_`, `-`, `.`.

```bash
mise dotfiles status [--missing] [--json]   # applied/missing/differs/source missing
mise dotfiles apply [--dry-run] [--verbose] [--yes] [--force]
mise dotfiles add ~/.zshrc         # capture a live file into dotfiles.root
mise dotfiles edit [--apply] ~/.zshrc
```

Dotfiles are **manual-only** — never applied implicitly, only by `mise dotfiles apply` or `mise bootstrap`. A regular file whose content already matches its source converges to a symlink without `--force`; a genuine conflict needs `--force`. On Windows, file symlinks fall back to copies (directories use junctions). Removing a config entry leaves files in place — cleanup is manual.

---

## OCI Container Images

Build and publish container images containing mise-managed tools. `oci` is a top-level config key.

```bash
mise oci build -o ./img                  # build an image
mise oci build --copy ./dist:/app/dist   # reproducible host-path copy layer
mise oci push myregistry.io/myimg:tag    # built-in registry client (no skopeo/crane)
mise oci push --cache-from myimg:prev    # reuse layers
mise oci push --no-cache
mise oci push --update-index             # upsert into a multi-arch image index
mise oci run --engine docker
```

```toml
[[oci.copy]]
source = "./dist"
dest = "/app/dist"

[settings.oci]
default_from = "debian:bookworm-slim"     # base image
default_mount_point = "/mise"
insecure_registries = ["registry.lan:5000", "10.0.0.8:5000"]   # plain HTTP
```

Since v2026.7.12 the registry client is built in — `docker login` / `podman login` is the only setup needed. `mise oci build` also bakes `[dotfiles]` and `apt:` `[bootstrap.packages]` into images as dedicated, annotated OCI layers.

> `mise oci push --tool` was **removed** in 2026.7.12. Use `mise oci build -o ./img` + `skopeo copy` instead.

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

**`.tool-versions` format:**
```text
node        20.0.0       # comments are allowed
ruby        3            # fuzzy version
erlang      ref:master   # compile from vcs ref
go          prefix:1.19  # latest 1.19.x
shfmt       path:./shfmt # custom runtime
node        sub-2:lts    # 2 major versions behind lts
```

### Idiomatic Version Files

Disabled by default. Enable per-tool:
```bash
mise settings add idiomatic_version_file_enable_tools python
```

Supported files include `.nvmrc`, `.node-version`, `package.json`, `.python-version`, `.python-versions`, `.ruby-version`, `Gemfile`, `.go-version`, `go.mod`, `rust-toolchain.toml`, `.java-version`, `.sdkmanrc`, `global.json`, `.terraform-version`, `.bun-version`, `.deno-version`.

### Key Settings Reference

```toml
[settings]
# Execution
jobs = 8                    # Concurrent jobs (MISE_JOBS)
experimental = false        # Enable experimental features
yes = false                 # Auto-answer prompts (MISE_YES)
safe = false                # Inert config-reader mode (global config only)

# Task defaults
task.output = "prefix"      # prefix|interleave|keep-order|replacing|timed|quiet|silent
task.timeout = "10m"        # Default task timeout
task.timings = true         # Show elapsed time
task.skip = ["slow-task"]   # Tasks to skip
task.skip_depends = false   # Skip dependencies
task.source_freshness_hash_contents = false  # blake3 content check
use_file_shell_for_executable_tasks = false  # Run file tasks through a shell

# Shells
unix_default_inline_shell_args = "sh -c -o errexit"
unix_default_file_shell_args = "sh"
windows_default_inline_shell_args = "cmd /c"
windows_default_file_shell_args = "cmd /c"
windows_powershell_no_profile = true

# Environment
env_shell_expand = true     # Shell-style expansion — DEFAULT TRUE since 2026.7.0
env_cache = false           # Cache computed environment
env_cache_ttl = "1h"        # Cache TTL
env_file = ""               # MISE_ENV_FILE
auto_env = false            # Auto-load platform/profile env files (default-on in 2027.6.0)

# Tool management
auto_install = true         # Auto-install missing tools
exec_auto_install = true    # Auto-install on mise x/run
not_found_auto_install = true
auto_install_disable_tools = []
disable_backends = ["asdf"] # Disable backends
disable_default_registry = false  # Ignore the built-in registry
enable_tools = []           # Allowlist (restricts tools)
disable_tools = []
pin = false                 # Default --pin for mise use
lockfile = true             # Read/update lockfiles
lockfile_platforms = []     # Extra platforms to resolve in the lockfile
locked = false              # Fail if no pre-resolved URLs
prereleases = false         # Allow pre-release versions for fuzzy requests
registry_floating = false   # Fetch current registry data instead of release-pinned snapshots
system_deps = "prompt"      # prompt|auto|warn|ignore — vfox systemDependencies handling

# Security
paranoid = false            # Extra-secure behavior (auto-enables locked_verify_provenance)
gpg_verify = false          # See note below — verification now ALWAYS runs when enabled
slsa = true                 # SLSA provenance verification
github_attestations = true  # GitHub Artifact Attestations
provenance_api_failures_fatal = true  # Treat provenance API failures as install errors
netrc = true                # Honor ~/.netrc for HTTP auth (netrc_file overrides path)
minimum_release_age = "24h" # Filter fuzzy versions by release age (e.g. "7d", "6mo", "2026-01-01")
minimum_release_age_excludes = []  # Tools exempt from the release-age delay

# Sandbox (deny-by-default policy)
[settings.sandbox]
deny_all = false
deny_read = false
deny_write = false
deny_net = false
deny_env = false

# Performance / network
[settings]
fetch_remote_versions_cache = "1h"
fetch_remote_versions_timeout = "20s"
http_timeout = "30s"                # Per-request HTTP timeout
http_download_timeout = "30m"       # Total download wall-clock (separate setting)
http_retries = 3                    # HTTP retries with exponential backoff (0 = none)
cache_prune_age = "30d"             # Age before cached downloads are pruned
registry_cache_ttl = "1h"
use_versions_host = true            # Use mise-versions shared version cache
offline = false                     # Block all HTTP requests
prefer_offline = false              # Prefer cached data

# UI / shell
color = true                # Colorized output
color_theme = "default"     # auto|default|charm|base16|catppuccin|dracula
terminal_progress = true    # OSC 9;4 terminal progress indicators
verbose = false             # Verbose install output

# Windows
windows_shim_mode = "exe"   # exe|file|hardlink|symlink
windows_executable_extensions = ["exe", "bat", "cmd", "com", "ps1", "vbs"]

# Status line
status.missing_tools = "if_other_versions_installed"
status.show_env = false
status.show_tools = false
status.show_deps_stale = true
status.truncate = true

# Node-specific
[settings.node]
corepack = false            # Enable corepack
compile = false             # Compile from source
verify = true
npm_shim = true
flavor = ""                 # Alternate distribution flavor

# NPM backend
[settings.npm]
package_manager = "auto"    # auto|npm|aube|bun|pnpm
shell_out = false           # Route metadata + installs through the npm CLI

# Python-specific
[settings.python]
uv_venv_auto = false        # false | "source" | "create|source" | true (legacy true form deprecated)
uv_venv_create_args = []
venv_create_args = []
compile = false             # Compile from source
venv_stdlib = false         # Prefer stdlib venv module
precompiled_flavor = "install_only_stripped"

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
registry_cache_ttl = "1w"
# registries = ["myorg/aqua-registry"]  # replaces deprecated registry_url

# Cargo
[settings.cargo]
binstall = true             # Use precompiled binaries
binstall_only = false
binstall_quickinstall = false   # DEFAULT FLIPPED TO false in 2026.7.6
# binstall_native = true    # experimental native binstall fast path

# Pipx
[settings.pipx]
uvx = true                  # Use uvx instead of pipx

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

# OCI images
[settings.oci]
default_from = "debian:bookworm-slim"
default_mount_point = "/mise"
insecure_registries = []

# Dotfiles (mise dotfiles)
[settings.dotfiles]
default_mode = "symlink"    # symlink|symlink-each|copy|template
root = "~/.dotfiles"

# System packages
[settings.system_packages]
sudo = true                 # set managers = [...] to pick package managers
```

> **`gpg_verify` behavior change (2026.7.12):** GPG verification now **always runs** when enabled, with Node/Swift signatures verified in-process via rPGP (no external `gpg` binary needed). Previously a missing `gpg` silently skipped verification — you must now set `gpg_verify = false` explicitly to opt out.

**All settings** support environment variable overrides using the `MISE_` prefix (e.g., `MISE_JOBS=4`, `MISE_TASK_OUTPUT=interleave`). The above is a representative subset — run `mise settings --all` (or `mise settings ls`) for the complete list, and `mise settings set <key> <value>` / `mise settings get <key>` to manage them. Language backends each have their own namespace (`[settings.node]`, `[settings.python]`, `[settings.ruby]`, `[settings.go]`, `[settings.rust]`, `[settings.java]`, `[settings.erlang]`, `[settings.swift]`, `[settings.zig]`, etc.).

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
| `MISE_PROJECT_ROOT` | Project root directory (subproject dir in monorepos; stable regardless of cwd) |
| `MISE_MONOREPO_ROOT` | Monorepo root (set inside a monorepo with `monorepo_root = true`) |
| `MISE_TASK_NAME` | Current task name |
| `MISE_TASK_DIR` | Task script directory |
| `MISE_TASK_FILE` | Full path to task script |

### CI/CD Integration

**GitHub Actions (`jdx/mise-action@v4`):**

```yaml
- uses: jdx/mise-action@v4   # v4 current (latest v4.2.2); mise docs still show v3 — v4 is correct
  with:
    version: 2026.7.12    # mise version (default: latest)
    install: true         # run `mise install`
    cache: true           # cache via GitHub cache
    experimental: false   # enable experimental features
    # tool_versions: |    # optionally inline .tool-versions content
    # mise_toml: |        # optionally inline a mise.toml
```

Automatically redacts values flagged with `redact = true` or matching `redactions` patterns. When a `mise.lock` is present, `mise-action@v4` auto-applies `--locked` so CI cannot mutate the lockfile.

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
script:
  - mise install
  - mise exec --command 'npm build'
```

**Generic CI bootstrap:**
```bash
curl https://mise.run | sh
mise install
mise x -- <cmd>

# Skip reinstall if already present (Docker layer caching)
curl https://mise.run | MISE_INSTALL_SKIP_IF_EXISTS=1 sh
```

Or: `mise generate bootstrap -l -w` produces a self-contained `./bin/mise` you can commit, so jobs don't re-download mise.

**Useful CI env vars:**
- `MISE_YES=1` — auto-answer prompts
- `MISE_SAFE=1` — inert config reader for untrusted/fork-PR configs
- `MISE_DATA_DIR` — install/cache root
- `MISE_EXPERIMENTAL=1` — unlock experimental features
- `MISE_OFFLINE=1` / `MISE_PREFER_OFFLINE=1` — network policy

### IDE Integration

IDEs inherit the environment from their launch shell and do not reload mise config changes. Because arbitrary `[env]` vars only load when a shim is executed, activate **shims** in your login profile so GUI-launched editors see mise tools:

```bash
# ~/.zprofile / ~/.bash_profile (login, non-interactive)
eval "$(mise activate zsh --shims)"
```

```fish
# ~/.config/fish/config.fish
if status is-interactive
  mise activate fish | source
else
  mise activate fish --shims | source
end
```

```lua
-- Neovim: prepend the shim dir to PATH
vim.env.PATH = vim.env.HOME .. "/.local/share/mise/shims:" .. vim.env.PATH
```

- **VS Code:** extension `hverlin.mise-vscode` (tools, tasks, env, `mise.toml` completion); or set `terminal.integrated.automationProfile.*` to a login shell; or use `runtimeExecutable: "mise"` with `runtimeArgs: ["exec", "--", "node"]` in launch configs.
- **JetBrains:** plugin `intellij-mise` (tools + run-configuration env), or the asdf-compat symlink `ln -s ~/.local/share/mise ~/.asdf`.
- **Xcode:** add `$(SRCROOT)/mise.toml` to build-phase input files, then `eval "$($HOME/.local/bin/mise activate -C $SRCROOT bash --shims)"`. Xcode Cloud: run the install + `mise activate bash --shims` in a `ci_post_clone.sh` script.
- **Emacs:** package `mise.el` (`liuyinz/mise.el`) — `(global-mise-mode)`; or add the shims dir to both `PATH` and `exec-path`.
- **Vim:** `let $PATH = $HOME . '/.local/share/mise/shims:' . $PATH`.

### Key Environment Variables

- `MISE_DATA_DIR` (default `~/.local/share/mise`)
- `MISE_CACHE_DIR` (default `~/.cache/mise`)
- `MISE_TMP_DIR` (default system temp)
- `MISE_SYSTEM_CONFIG_DIR` (default `/etc/mise`)
- `MISE_GLOBAL_CONFIG_FILE` (default `~/.config/mise/config.toml`)
- `MISE_GLOBAL_CONFIG_ROOT` (default `$HOME`)
- `MISE_ENV_FILE` (e.g., `.env`)
- `MISE_${TOOL}_VERSION` (e.g., `MISE_NODE_VERSION=20`)
- `MISE_TRUSTED_CONFIG_PATHS` / `MISE_CEILING_PATHS` / `MISE_IGNORED_CONFIG_PATHS` (`:` Unix, `;` Windows)
- `MISE_OVERRIDE_CONFIG_FILENAMES` / `MISE_DEFAULT_CONFIG_FILENAME`
- `MISE_LOG_LEVEL` (trace|debug|info|warn|error)
- `MISE_QUIET` (= `MISE_LOG_LEVEL=warn`)
- `MISE_HTTP_TIMEOUT` (default 30s) / `MISE_HTTP_DOWNLOAD_TIMEOUT` (default 30m)
- `MISE_TERM_WIDTH` — terminal width override for `mise ls`/`registry`/`settings` (takes precedence over `COLUMNS`; env-only, not a setting)
- `MISE_RAW` (pipes directly; forces `MISE_JOBS=1`)

**Global CLI flags:** `-C/--cd <DIR>`, `-E/--env <ENV>`, `-j/--jobs <N>`, `-q/--quiet`, `-v/--verbose`, `-y/--yes`, `--raw`, `--locked`, `--silent`, `--no-config`, `--no-env`, `--no-hooks`, `--output <MODE>`.

---

## Dependency Preparation (`[deps]`)

Ensures project dependencies (npm packages, Python venvs, Go modules, …) are installed before task execution. This replaces the old `[prepare]` section — **there is no `prepare` key in the schema.**

```bash
mise deps            # install project deps
mise deps add <pkg>
mise deps remove <pkg>
mise deps install
mise deps --monorepo                    # experimental; across monorepo config roots
mise deps --monorepo --only //apps/api:uv
```

```toml
[deps.npm]
auto = true

[deps.custom]
sources = ["schema/*.graphql"]
outputs = ["src/generated/"]
run = "npm run codegen"
```

**Built-in providers:** `npm`, `yarn`, `pnpm`, `bun`, `deno`, `aube`, `go`, `pip`, `poetry`, `uv`, `bundler`, `composer`.

Built-in providers activate only when **explicitly configured in `mise.toml` AND their lockfile exists**. Disable auto-run for a single invocation with `mise run --no-prepare <task>` / `--no-deps`.

In monorepos, provider IDs are qualified by config root (`//apps/api:uv`) so repeated provider names don't collide. Deps **task providers** are gated behind `experimental = true` (2026.7.12+).

---

## Monorepo Tasks

```toml
# Root mise.toml
monorepo_root = true

[monorepo]
config_roots = ["packages/frontend", "packages/backend", "services/*"]
lockfile = true    # true = single root lockfile; false = per-subproject; unset = current behavior
```

Stable since v2026.6.6 — no longer requires `MISE_EXPERIMENTAL=1`. Provides implicit trust for descendants, lazy task loading, and tool inheritance from parent configs.

> `experimental_monorepo_root` is **deprecated and now emits a warning** (it is no longer a silent alias). Removal: **2027.12.0**. Use `monorepo_root`.

**Path syntax:**
```bash
mise //projects/frontend:build    # Absolute path from root
mise :build                       # Task in current config_root
mise '//projects/frontend:*'      # All tasks in frontend
mise //...:test                   # Test task in all projects
mise '//...:test*'                # Wildcard task names across all projects
```

`...` matches directory depth; `*` matches task names. mise never defines commands starting with `//` or `:`.

**Listing & install:**
```bash
mise tasks                          # current root + parents
mise tasks --all
mise tasks '//projects/frontend:*'
MISE_ENV=ci mise install --monorepo
mise install --monorepo node
mise ls --monorepo
```

`[monorepo].config_roots` supports single-level `*` globs only (`**` unsupported) and skips filesystem walking. **Automatic filesystem discovery is deprecated** — declare `config_roots` explicitly.

Discovery tuning when relying on walking:
```toml
[settings.task]
monorepo_depth = 5
monorepo_exclude_dirs = ["dist", "node_modules"]
monorepo_respect_gitignore = true
```

> `[monorepo].lockfile` rollout: unset keeps per-subproject lockfiles today; **warns in 2026.12.0, defaults to root lockfiles in 2027.6.0**. Pin `lockfile = false` for mixed-version teams. Old subproject lockfiles auto-migrate into the root lockfile (root entries win).

---

## Deprecation Calendar

| Removal | What | Migrate to |
|---------|------|------------|
| **2026.9.0** (warn) | Hook `script`/`scripts` *spawned* table form | `run` (removal 2027.3.0) |
| **2026.10.0** (warn) | `tera_v1` setting | Tera v2 syntax (removal 2027.4.0) |
| **2026.11.0** | `credential_command` legacy single positional argument | `MISE_CREDENTIAL_HOST` / `MISE_CREDENTIAL_PROVIDER` env vars |
| **2026.12.0** | `env.mise.*` namespace | `env._.*` |
| **2026.12.0** | `value`/`values` keys in `_.file`/`_.path`/`_.source` | `path` (string or array) |
| **2026.12.0** (warn) | `[monorepo].lockfile` unset behavior | Set it explicitly |
| **2027.3.0** | Hook `script`/`scripts` spawned form | `run` |
| **2027.4.0** | `tera_v1` / `MISE_TERA_V1` | Tera v2 |
| **2027.4.0** | Top-level `env_file` / `dotenv` / `env_path` | `_.file` / `_.path` |
| **2027.5.0** | Tera task-arg functions `{{arg()}}`, `{{option()}}`, `{{flag()}}` in run scripts | `usage` spec + `$usage_*` |
| **2027.12.0** | `experimental_monorepo_root` | `monorepo_root` |

**Default flips ahead:** `auto_env` → `true` in **2027.6.0** · `[monorepo].lockfile` → root lockfiles in **2027.6.0** · `cargo.binstall_native` warns 2027.1.0 and defaults on **2027.7.0**.

**Already removed (no deprecation period):** `vars.mise` namespace · non-string `postinstall` hooks · unknown table fields in hook definitions · `mise oci push --tool`.

> **Doc inconsistency to be aware of:** on the Tera task-arg removal, `/tasks/toml-tasks.html` says **2027.5.0** while `/tasks/task-arguments.html` and `/tasks/task-configuration.html` say **2026.11.0**. Either way, do not use them. Opt out early with `task.disable_spec_from_run_scripts = true`.

---

## Best Practices

### DO ✅

- **Always use `usage` field** for task arguments
- Use `${var?}` for required args to fail early; test optional values with `[ -n "${usage_x:-}" ]` (unset ≠ `"false"`)
- Set `description` for discoverability
- Use `sources`/`outputs` for cacheable tasks
- Use `depends` for task ordering; structured `depends` to pass args/env
- Use `confirm` for destructive operations
- Use `choices` for stable enums, `complete` for dynamic/filesystem-derived values
- Group related tasks with namespaces (e.g., `test:unit`, `test:e2e`)
- Share task config via `[task_templates]` + `extends`
- Use the per-task `output` field for style; `quiet` only to silence mise's own chatter
- Use `mise.local.toml` for personal overrides (gitignored)
- Prefer aqua backend for security (cosign/SLSA/attestation verification)
- Migrate from `ubi:` backend to `github:` (ubi deprecated), giving each binary its own `[tool_alias]`
- Use `env._.file`/`env._.path` instead of the deprecated top-level `env_file`/`dotenv`/`env_path`
- Redact sensitive values with `redact = true`; use fnox, SOPS, or direct-age for secrets
- Use templates for dynamic values instead of hardcoding paths
- Use shims in `.zprofile`/`.bash_profile` and PATH activation in `.zshrc`/`.bashrc`
- Use `[tool_alias]` (not deprecated `[alias]`)
- Pin tool versions with `mise.lock` + `locked = true` in CI; use `minimum_release_age` for supply-chain delay
- Use `mise lock --bump` to advance fuzzy selectors without touching `mise.toml`
- Use `jdx/mise-action@v4` in GitHub Actions — it handles masking and `--locked` automatically
- Use `MISE_SAFE=1` when reading configs you don't control (fork PRs, untrusted repos)
- Sandbox risky tasks with `deny_*` + narrow `allow_*` lists
- Declare `[monorepo].config_roots` explicitly instead of relying on filesystem walking
- Use `mise bootstrap` with `[bootstrap]`/`[dotfiles]` to onboard developers and provision fresh machines in one command

### DON'T ❌

- Use `$1`, `$2`, `$@`, `$*` for arguments
- Use `$args` in PowerShell
- Use inline template functions `{{arg()}}`/`{{option()}}`/`{{flag()}}` in run scripts (deprecated)
- Use usage attributes that don't exist: `parse`, flag `alias` child, `config=`, `required_if`, `required_unless`, `overrides`, or the `config { file … }` block — they hard-error
- Rely on `usage_*` leaking into nested tasks (invocation-local since 2026.7.6 — pass via `env=` or structured `depends`)
- Assume `--quiet` changes output style (it no longer does — use `--output`)
- Forget to quote glob patterns in sources
- Set env vars in `env` that deps need (they don't inherit — use structured `depends` with `env`)
- Use `raw = true` unless interactive input is needed (forces single-threaded, bypasses redactions)
- Set `MISE_ENV` in `mise.toml` (it determines which files to load — use `.miserc.toml`)
- Manually add executables to shims directory (`mise reshim` deletes them)
- Use `MISE_RAW=1` without knowing it sets `MISE_JOBS=1`
- Install new `asdf:` or `vfox:` plugins when aqua/github alternatives exist
- Use `[prepare.*]` — it no longer exists; use `[deps.*]`
- Use `vars.mise` or `env.mise.*` — rejected / deprecated in favor of `vars._` and `env._`

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
output = "keep-order"
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
min_version = "2026.7.0"

[settings]
jobs = 8
task.output = "interleave"
task.timings = true
lockfile = true

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

[task_templates.node-script]
tools = { node = "22" }
env = { NODE_OPTIONS = "--enable-source-maps" }

[tasks.dev]
description = "Start development server"
extends = "node-script"
depends = ["build"]
run = "npm run dev"

[tasks.build]
description = "Build project"
extends = "node-script"
sources = ["src/**/*.ts", "tsconfig.json"]
outputs = ["dist/**/*"]
run = "tsc --build"

[tasks.test]
description = "Run tests"
depends = ["build"]
run = "vitest run"
```
