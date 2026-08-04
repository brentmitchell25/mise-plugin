---
name: mise
description: Use when working with mise - creating/editing mise.toml or .mise.toml files, defining tasks with usage field arguments, managing dev tools and environments, configuring hooks, task caching, sandboxing, machine bootstrap, monorepo workspaces, or when user mentions mise configuration.
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
  - [Command Effects (effect=)](#command-effects-effect)
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
- [Task Dependencies and Freshness](#task-dependencies-and-freshness)
- [Task Output Caching (Experimental)](#task-output-caching-experimental)
  - [Local Artifact Cache](#local-artifact-cache)
  - [Cache Inputs](#cache-inputs)
  - [Remote Task Cache](#remote-task-cache)
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
  - [Profiles / Configuration Environments (MISE_ENV)](#profiles--configuration-environments-mise_env)
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
- [Monorepo Tasks and Workspace Graph](#monorepo-tasks-and-workspace-graph)
  - [Workspace Project Graph (Experimental)](#workspace-project-graph-experimental)
  - [Affected Tasks (Experimental)](#affected-tasks-experimental)
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
- **Tasks** — project-specific commands with argument handling, dependencies, freshness checks, and output caching
- **Environments** — manage env vars, configuration environments, dotenv files, secrets, age/sops-encrypted values
- **Hooks** — run commands on directory changes, project enter/leave, tool install
- **Sandboxing** — restrict a task's filesystem/network/env access; `safe` mode for untrusted configs
- **Machine bootstrap** — provision a whole dev machine (system packages, users, files, services, firewall, git repos, dotfiles, macOS defaults, login shell) via `mise bootstrap`
- **Monorepos** — workspace project graph inference across Cargo/uv/Go/Node, affected-task selection
- **OCI images** — build/push container images containing mise-managed tools

**Key Features:**
- Parallel dependency building (concurrent by default, up to `jobs` setting)
- Last-modified and content-hash (blake3) freshness checking
- Task output artifact caching, local and remote (experimental)
- File watching (`mise watch` rebuilds on changes)
- Cross-platform argument handling via `usage` spec
- Hierarchical configuration with configuration-environment support
- 18 tool backends (aqua, github, gitlab, forgejo, npm, cargo, pipx, etc.)
- Security verification (cosign, SLSA, GitHub Attestations, minisign) — all native, no external CLIs
- Secret management (fnox, sops, age encryption)

**Top-level `mise.toml` keys** (authoritative, from `https://mise.jdx.dev/schema/mise.json`):
`_`, `alias` (deprecated), `bootstrap`, `deps`, `dotenv` (deprecated), `dotfiles`, `env`, `env_file` (deprecated),
`env_path` (deprecated), `hooks`, `min_version`, `monorepo`, `monorepo_root`, `oci`, `plugins`, `redactions`,
`settings`, `shell_alias`, `task_config`, `task_templates`, `tasks`, `tool_alias`, `tools`, `vars`, `watch_files`.

> There is **no `[prepare]` key** — that feature is `[deps]`. See [Dependency Preparation](#dependency-preparation-deps).

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

Supports: Python, Node, Bun, Deno, Ruby, Bash, PowerShell. Use `-S` for multiple interpreter arguments (`#!/usr/bin/env -S deno run --allow-env`).

> PowerShell tasks run with `-NoProfile` by default since **2026.7.13** (`windows_powershell_no_profile = true`), so a profile that mutates PATH can no longer shadow the task's own tools. Opt out with `MISE_WINDOWS_POWERSHELL_NO_PROFILE=false`.

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

If `task_config.includes` is set, it **replaces** these defaults — list them explicitly to keep them. On name collision across includes, the **last entry wins**.

**Supported `#MISE` directives:** `description`, `alias`, `sources`, `outputs`, `env`, `depends`, `depends_post`, `wait_for`, `tools`, `dir`, `hide`, `run`, `quiet`, `silent`, `raw`, `shell`, `confirm`, `timeout`.
**Supported `#USAGE` directives:** `arg`, `flag`, `choices`, `complete`, `env`, and root-level `mount`.

**Each `#MISE` line is TOML.** Arrays and inline tables may span lines as long as every line keeps the prefix, and dotted keys build tables without braces:

```bash
#MISE depends=[
#MISE   "lint",
#MISE   "test",
#MISE ]
#MISE tools.node="20"
```

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

> **File tasks only.** mise hoists root-level `mount` nodes into a synthetic command, and that hoisting runs only for file-task headers. A root-level `mount` inside a TOML `usage = '''…'''` field fails with `Invalid usage config` — put it inside a `cmd` block there. Mounts resolve at **completion** time, not on `--help`.

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

The file named `_default` executes when invoking the directory name without a subtask. In TOML, quote the key: `[tasks."test:unit"]`.

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

Remote git includes in `[task_config]` (**experimental**):
```toml
[task_config]
includes = ["git::https://github.com/myorg/shared-tasks.git//tasks?ref=main"]
```

---

## Task Arguments - Usage Spec Reference

**REMINDER:** Always use the `usage` field. Never use `$1`, shell positional parameters, or other shell-native argument handling.

The `usage` field uses [KDL-inspired syntax](https://usage.jdx.dev/) to define arguments, flags, and completions.

> **Version context (2026-08-04):** mise pins **usage-lib 4.1.0**. The standalone `usage` CLI's latest release is **5.0.0**. Anything marked "usage 5.0+" below is **not yet available inside mise**. mise emits `min_usage_version "4.0"`.

### Positional Arguments (`arg`)

Every attribute is accepted both as a prop (`arg "<f>" key=value`) and as a child node (`arg "<f>" { key value }`), except `choices` which is child-node only.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (name) | string | *required* | `"<name>"` = required, `"[name]"` = optional. `"<file>"`/`"<path>"` triggers file completion, `"<dir>"` triggers dir completion. |
| `help` | string | none | Short help text shown with `-h` |
| `long_help` / `help_long` | string | none | Extended help text shown with `--help` |
| `help_md` | string | none | Markdown-only help (docs generation) |
| `required` | boolean | `#true` for `<x>`, `#false` for `[x]` | **Forced to `#false` whenever `default` is set** |
| `default` | string (prop) / string-or-list (child) | none | Default value if not provided. `default=""` sets to empty string (different from unset). |
| `env` | string | none | Environment variable that can provide this arg's value. Priority: CLI > env > default. |
| `var` | boolean | `#false` | Variadic mode (accept multiple values). Shorthand: `"<name>..."` |
| `var_min` | integer | none | Minimum values when variadic |
| `var_max` | integer | none | Maximum values when variadic |
| `choices` | child node | none | Restrict to enumerated set (literal values only — see caveat below) |
| `effect` | enum | none | `read` \| `write` \| `destructive` — raises the command's effect (usage 4.0+) |
| `double_dash` | enum | `optional` | `"required"`, `"optional"`, `"automatic"`, `"preserve"` |
| `hide` | boolean | `#false` | Exclude from help output |

**Shorthands:** `<f>` required · `[f]` optional · `<f>...` ⇒ `var=#true` · `<-- f>` / `[-- f]` ⇒ `double_dash="required"`.

**Examples:**

```
arg "<file>" help="Input file to process"
arg "[output]" default="out.txt" help="Output file"
arg "<files>" var=#true var_min=1 help="One or more files"
arg "<files>..." help="Shorthand variadic syntax"
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
arg "<args>..." double_dash="automatic" help="Pass-through arguments"
arg "<file>" env="MY_FILE" help="Input file (or set MY_FILE)"
arg "<mode>" { default { "fast"; "safe" } }         # multi-value default block
```

> ⚠️ **`choices env="VAR"` does NOT work in mise.** Env-backed choices sit behind the `unstable_choices_env` Cargo feature, which mise does not enable. Verified on mise 2026.8.0: both `choices env="DEPLOY_ENVS"` and the mixed `choices "local" env="DEPLOY_ENVS"` form fail with `Invalid usage config`. Use literal `choices` or a `complete … run=` block instead.

> ⚠️ **`double_dash="required"` parses but is silently ignored** under usage 4.1.0 (what mise ships). It is only enforced from usage **5.0.0**, where offering a value before `--` errors with `ArgRequiresDoubleDash`. Do not rely on it for validation in mise today, and expect specs that "happened to work" to start erroring once mise upgrades.

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
| `global` | boolean | `#false` | Available on all subcommands; also passed to `mount` |
| `count` | boolean | `#false` | Value = number of times used (e.g., `-vvv` = 3) |
| `var` | boolean | `#false` | Flag repeatable, collecting values |
| `var_min` / `var_max` | integer | none | Min/max values when `var=#true` |
| `negate` | string | none | Negative form (e.g., `"--no-color"`). Sets env var to `false`. |
| `effect` | enum | none | `read` \| `write` \| `destructive` (usage 4.0+) |
| `allow_hyphen_values` | boolean | `#false` | **usage 3.5.5+.** Let a value-taking flag consume a following `-…` token as its value. Errors if the flag takes no value. |
| `deprecated` | bool \| string | none | `#true` → literal `"deprecated"`; a string is used as the message |
| `arg` | child node | none | Names the flag's value: `flag "--user" { arg "<user>" }` |
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
flag "--color" env="MYCLI_COLOR" help="Backed by env var"
flag "-a --args <ARGS>" allow_hyphen_values=#true help="Pass-through args (e.g. -a -destroy)"
flag "--old-flag" deprecated="use --new-flag instead"
flag "--clear" effect="destructive" help="Delete stored logs"
```

**Count flags:** `-vvv` sets `$usage_verbose` to `3`. Short flags chain: `-abc` = `-a -b -c`.

> ⚠️ **Never put `default` on a `count` flag.** A bare integer (`default 0`) fails to parse (`expected string`), and `default="0"` is coerced through the boolean path, yielding `usage_verbose=false` when the flag is absent. Use `${usage_verbose:-0}` in the script instead. (mise's own `task-arguments` doc contains this broken example.)

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
| `type` | string | none | Built-in completer: `path`/`file` (currently identical) or `dir`. **Mutually exclusive with `run`** — setting both errors. |
| `descriptions` | boolean | `#false` | Parse `value:description` output |

**Key rules:**
- Node name is lowercased and must match the arg name exactly
- Must appear **after** the `arg` it applies to
- Do **not** combine with `choices` on the same `arg`
- When `type` is unset, **the arg's own name is used as the type** — which is why `arg "<file>"`, `arg "<dir>"`, and `arg "<path>"` auto-complete with no `complete` block at all
- A **spec-level** `complete` shadows a same-named `complete` inside a `cmd` block

**Resolution order per arg:** built-in type (or arg name) → `choices` → `run`.

> Consequence: `arg "<file>" { choices "a" "b" }` silently ignores the choices during *completion*, because the name `file` triggers builtin path completion first. Validation still applies.

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
Output format: `value:description` per line — split on the **first unescaped colon** (regex requires a preceding non-backslash char, so a line *starting* with `:` is not split); escape a literal colon with `\:`; both sides are trimmed.

**Tera template variables in `run`:**
- `words` — array of all prompt words including the one being typed; access via `words[index]`
- `CURRENT` — index of word being typed
- `PREV` — index of previous word (`CURRENT-1`). **Only defined when `CURRENT > 0`.**

```
complete "controller" run="ls modules/{{words[PREV]}}/controllers"
complete "four" run="echo {{ words | slice(start=-4) | join(sep='\n') }}"
```

Execution: `sh -c` (`cmd /c` on Windows when `sh` is absent), stdin closed, `__USAGE` set to the usage version. On Windows `cmd` cannot run pipelines or builtins — keep `run` to a single command.

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
| Boolean flag | `"true"` | **variable absent entirely** (or `"false"` with `default=#false`/`negate`) |
| Count flag | `"1"`, `"2"`, etc. | **absent** |
| Value flag/arg | the string value | absent unless `default=` or `env=` supplied one |
| Variadic | shell-escaped space-separated | **absent** |

**Critical distinction:** absent means **UNSET**, not empty string. `default=""` makes the variable SET to an empty string. Test with `[ -n "${usage_x:-}" ]`, never `= "false"`.

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

# Count flag
level="${usage_verbose:-0}"

# Variadic to array
eval "files=($usage_files)"
```

**Precedence:** CLI args > env vars > defaults. This applies to **both TOML tasks and file tasks** (documented for file tasks since 2026.7.12). The usage docs describe a four-tier chain including config files, but the config tier is unimplemented (see [`config` block](#spec-level-metadata)).

> **Breaking change (2026.7.6):** `usage_*` variables are **invocation-local** — they no longer leak into nested tasks as implicit inputs. Values are cleared for normal task execution; `raw_args = true` tasks retain them. If a dependency needs a value, declare it with `env=` (under a different name) or pass it via structured `depends`.

Arguments are also readable as Tera values in TOML tasks: `{{ usage.env }}`, `{{ usage["dry-run"] }}`. Variadics arrive as arrays (usable with `for` / `length`). Values are **not** shell-escaped or quoted. Valid in `depends`, `depends_post`, and `wait_for` too.

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

`#MISE key=value` lines are task **config**, not spec — mise routes anything matching `^[a-z0-9_.-]+\s*=` away from the spec parser.

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

cmd "production" effect="destructive" help="Deploy to production" {
  arg "<service>"
  flag "--canary"
  before_help "Production deploys require approval."
}
'''
```

| `cmd` attribute | Type | Notes |
|-----------------|------|-------|
| `help`, `long_help`, `before_help`, `after_help`, `before_long_help`, `after_long_help` | string | Help text (valid as props *or* child nodes) |
| `help_md`, `before_help_md`, `after_help_md` | string | Also valid as child nodes since usage 3.6.0 |
| `subcommand_required` | boolean | Require a subcommand |
| `hide` | boolean | Hide from help |
| `effect` | enum | `read` \| `write` \| `destructive` |
| `restart_token` | string | Resets arg parsing for repeated invocations (this is how `mise run a ::: b` works) |
| `deprecated` | bool \| string | Mark deprecated |
| `alias "x"` | child node | Repeatable; supports `hide=#true` |
| `example "code"` | child node | **Exactly one** positional arg; use `header=` / `help=` / `lang=` props |
| `mount run="<cmd>"` | child node | Dynamic subcommand mounting; `run` is required |

```
example "mycli list --all" header="Basic usage" help="Lists everything"
```

**Child-node-only** (never props): `flag`, `arg`, `mount`, `cmd`, `alias`, `example`, `complete`. Every prop also has a child-node form.

Mounted commands do **not** inherit the mounting CLI's global flags (usage 3.5.7+), so `mise run mytask --<TAB>` no longer offers mise's own `--cd`/`--jobs`/etc., and a task flag that collides with a global is no longer shadowed.

### Command Effects (`effect=`)

Declares what a command does to the world. Values, ordered: `read` < `write` < `destructive`.

| Effect | Meaning |
|--------|---------|
| `read` | Only inspects state. Idempotent. |
| `write` | Creates/modifies state; removes nothing the user can't recreate. |
| `destructive` | May delete or irreversibly overwrite. Deserves a confirmation prompt. |

- Available on `cmd` (usage 3.6.0+) and on `flag`/`arg` (usage 4.0+).
- The effect of an invocation is the **maximum** of the command's effect and every flag/arg actually supplied. Effects only ever **raise**, never lower.
- **Not inherited by subcommands.** Absent means *unknown*, not safe — consumers should treat it as "ask".
- **Most flags should declare nothing** — reserve it for the few that change what a command does.
- `--dry-run` lowering is deliberately unsupported.

```
cmd "settings" effect="read" {
  arg "[setting]"
  arg "[value]" effect="write"   # `settings foo` reads, `settings foo=bar` writes
}
```

Consumed by `mise mcp`'s `list_commands` tool and `usage mcp` (4.1.0+), which tag every command with its declared effect.

### Spec-Level Metadata

Top-level usage keywords (rarely needed for mise tasks but supported):

```
name "..."              # display name
bin "..."               # binary name (defaults to filename)
version "..."           # version string
min_usage_version "..." # minimum usage spec version required (put first; warns, does not fail)
author "..."            # CLI author
license "..."           # SPDX license identifier
repository "..."        # plain project URL (usage 4.1.0+)
about "..."             # short help
long_about "..."        # long help
before_help "..."       # text before help body
after_help "..."        # text after help body
before_long_help "..."  # before-help shown with --help
after_long_help "..."   # after-help shown with --help
usage "..."             # override the generated usage line
disable_help #true      # suppress the automatic help flag
default_subcommand "..." # "naked" invocation target
source_code_link_template "..."  # Tera template with {{path}} / {{cmd}}
example "code" header="..." help="..." lang="..."
include file="./other.usage.kdl"   # merge another spec (file= is required)
```

`repository` (a plain URL, like `Cargo.toml`'s) is distinct from `source_code_link_template` (a per-command deep link) — neither implies the other. There is **no `mount` at spec root** in usage itself; see the file-task hoisting note above.

**`config` block** — declares config-file-backed properties (metadata for docs/SDK generation only):

```
config {
  prop "color"   default=#true   env="COLOR"   help="Enable color output"
  prop "jobs"    default=4       env="JOBS"    help="Number of jobs" data_type="integer"
  prop "timeout" default=1.5     env="TIMEOUT" help="Timeout" data_type="float"
}
```

`prop` attributes: `default`, `default_note`, `data_type` (`null`/`string`/`integer`/`float`/`boolean`), `env`, `help`, `long_help`.

> **usage reads no config files at all.** The `config` block is documentation/metadata; the "CLI flag > env var > config file > default" chain in the usage docs is aspirational. Env-var backing (`env=`) does work.

### Documented but NOT Implemented (do not use)

These appear on usage.jdx.dev but **hard-error**. Verified empirically against usage 4.1.0 (what mise ships) and mise 2026.8.0:

| Syntax | Error | Use instead |
|--------|-------|-------------|
| `arg "<f>" parse="cmd {}"` | `unsupported arg key parse` | — (no equivalent) |
| `arg "<e>" { choices env="VAR" }` | `Invalid usage config` (mise) | Literal `choices`, or `complete … run=` |
| `flag "--user <u>" { alias "-u" }` | `unsupported flag key alias` | `flag "-u --user <u>"` |
| `flag "--color" config="ui.color"` | `unsupported flag key config` | `env="..."` |
| `flag "--f <x>" required_if="--dir"` | `unsupported flag key` | Validate inside the script |
| `flag "--f <x>" required_unless="--stdin"` | `unsupported flag key` | Validate inside the script |
| `flag "--f <x>" overrides="--stdin"` | `unsupported flag key` | Validate inside the script |
| `config { file "..." findup=#true format="ini" }` | `unsupported config key file` | `config { prop "..." }` |
| `config_file "..."` / `config_alias "a" "b"` | `unsupported spec key` | `config { prop "..." }` |
| `cmd "x" { example "Header" "code" }` | needs exactly 1 arg | `example "code" header="Header"` |
| root-level `mount` in a TOML `usage` field | `Invalid usage config` | File-task header, or a `cmd` block |

---

## Task Configuration Reference

### All Task Fields

#### Core Execution

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `run` | `string \| string[] \| ({task: string, args?: string[], env?: object} \| {tasks: string[]} \| string)[]` | — | Command(s) to execute. The only required property. |
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
| `depends` | `string \| string[] \| object[]` | — | Tasks to run BEFORE (supports structured objects with `args`/`env`/`optional`). Parallel by default. |
| `depends_post` | same as `depends` | — | Tasks to run AFTER this task and its deps complete (runs even on failure) |
| `wait_for` | same as `depends` | — | Wait for tasks without adding as deps (optional coordination) |

#### Environment & Tools

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `env` | `table` | — | Task-specific env vars (**NOT passed to depends**). Supports sops/age-encrypted values and `_.file`. |
| `pass_through_env` | `string[]` | — | Ambient env vars passed through **without** affecting the cache key (tokens, CI vars). Survives `deny_env`. |
| `tools` | `table` | — | Tools to install before running |
| `vars` | `table` | — | Task-local vars that override `[vars]` |

#### Execution Context

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dir` | `string` | `{{config_root}}` | Working directory. Use `{{cwd}}` for user's cwd. |
| `raw` | `bool` | `false` | Direct stdin/stdout connection (forces `--jobs=1`; **bypasses redactions**) |
| `raw_args` | `bool` | `false` | Pass all args verbatim including `--help`/`-h` to underlying command |
| `interactive` | `bool` | `false` | Exclusive lock on stdin/stdout/stderr; other non-interactive tasks still run in parallel (narrower than `raw`) |

#### Output Control

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `output` | `"prefix" \| "interleave" \| "keep-order" \| "replacing" \| "timed" \| "quiet" \| "silent"` | inherits `task.output` | **Per-task output style** (2026.7.6+). Orthogonal to `quiet`/`silent`. |
| `quiet` | `bool` | `false` | Suppress mise's own chatter (echoed command). **No longer changes the output style.** |
| `silent` | `bool \| "stdout" \| "stderr"` | `false` | Suppress all/specific task output |

#### Freshness & Caching

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sources` | `string \| string[]` | — | Input files (globs, `{a,b}` braces, `!` exclusions, `@group:name` refs) |
| `outputs` | `string \| string[] \| {auto: true}` | `{auto: true}` (if `sources` set) | Generated files. Supports ordered `!` exclusions. `outputs = []` enables result-only caching. |
| `cache` | `{enabled, audit, env, command_inputs}` | `{enabled=false, audit=false, env=[], command_inputs=[]}` | **Experimental.** Task output artifact cache — see [Task Output Caching](#task-output-caching-experimental). |
| `watch` | `{no_vcs_ignore: bool}` | — | `no_vcs_ignore = true` watches sources excluded by `.gitignore` (2026.8.0+) |

#### Safety & Sandboxing

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `confirm` | `string \| {message: string, default: string}` | — | Prompt before running. Supports Tera with `usage.*`. **Guards only `run`, not `depends`** (deps run before the prompt). |
| `deny_all` | `bool` | `false` | Block reads, writes, network, and env inheritance |
| `deny_read` / `deny_write` / `deny_net` / `deny_env` | `bool` | `false` | Individual sandbox blocks |
| `allow_read` / `allow_write` | `string[]` | — | Path allowlists |
| `allow_net` | `string[]` | — | Host allowlist |
| `allow_env` | `string[]` | — | Env var allowlist (wildcards: `NODE_*`) |
| `redactions` | `string[]` | — | **Experimental.** Env var names (globs allowed, e.g. `SECRETS_*`) redacted from output as `[redacted]` |

These are **flat top-level task fields**, not a nested `sandbox` table.

#### Timeout & Inheritance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `timeout` | `string` | — | Max execution duration (e.g., `"30s"`, `"5m"`). **Shorter of per-task and global `task.timeout` wins** — a per-task value cannot extend beyond the global one; `--timeout` overrides. Tera supported (2026.7.6+). |
| `extends` | `string` | — | Name of a `[task_templates.*]` entry to inherit from (single string; no multiple inheritance) |

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

**Optional dependencies (2026.7.17+):** `optional = true` runs all matches when present and is silently omitted when nothing matches. Invalid selectors still error. Works across `depends`, `depends_post`, and `wait_for`.
```toml
depends = [{ task = "lint:*", optional = true }]
```

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
| **Full override** (task wins, no merging) | `run`, `run_windows`, `depends`, `depends_post`, `wait_for`, `sources`, `outputs`, `cache`, `output` |
| **Override if set locally** | `dir`, `description`, `shell`, `timeout` |
| **Deep merge** (per key; task wins) | `tools`, `env` |
| **Compose / combine** | sandbox `deny_*` compose; `allow_*` combine |
| **Not allowed on templates** | `hide`, `interactive`, `quiet`, `raw`, `raw_args` — set these per task |

> `depends` **replaces** rather than merges: template `["lint","typecheck"]` + task `["build"]` = `["build"]`.

Templates are Tera-rendered in the **consuming** project's context (`{{config_root}}`, `{{env.VAR}}`, `{{cwd}}`, `{{vars.*}}`). Tasks loaded via `task_config.includes` now receive the collected `task_templates` map.

### Global Task Configuration

#### [task_config] Section

```toml
[task_config]
dir = "{{cwd}}"      # Default working directory for all tasks in this file
shell = "bash -c"    # Project-scoped default shell for tasks (2026.7.15+)
cascade = false      # Cascade dir/shell/cache/includes to descendant config roots
includes = [
  "tasks/*.toml",                              # Local task files
  ".mise/tasks/",                              # Task directory
  "git::https://github.com/org/tasks?ref=v1"   # Remote tasks (experimental)
]
```

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `dir` | string | — | Default dir for all tasks in scope |
| `shell` | string | — | Project-scoped default shell; task-level `shell` wins |
| `cascade` | bool | `false` | Cascade `dir`, `shell`, `cache`, `includes` to descendant config roots |
| `includes` | string[] | the five default task dirs | **Last entry wins** on name collision |
| `cache` | table | — | **Experimental**; inherits only to tasks with sources+outputs |
| `global_env` | string[] | — | **Experimental**; env names folded into every task's cache key |
| `global_pass_through_env` | string[] | — | **Experimental**; passed through without affecting cache keys |
| `global_inputs` | string[] | — | **Experimental**; config-rooted patterns applied to every task; may use `@group:` refs |
| `input_groups` | `{name: string[]}` | — | **Experimental**; referenced as `@group:name`, nestable |

> 🔴 **Breaking change (2026.7.14):** `unix_default_inline_shell_args`, `unix_default_file_shell_args`, `windows_default_inline_shell_args`, and `windows_default_file_shell_args` are now **global-config-only**. Project-local values are ignored, because local config is loaded before trust evaluation and an untrusted repo could otherwise influence how trusted commands execute. **Migration:** set `shell` per task, or use `task_config.shell` (added in 2026.7.15 precisely as the sanctioned replacement). Global config and `MISE_*` env vars still apply.

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

Shared variables between tasks (NOT environment variables — never exported to processes):

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

Value forms: plain scalar; `{ value, redact? }`; `{ default, redact? }`; `{ required = true|"msg", redact? }`; `{ age = ... }` (experimental). Plus the `_` module: `vars._.file` and `vars._.source` (strings or arrays only — no object form).

Vars accessed via `{{vars.key_name}}` Tera templates. **Scope precedence:** global (`~/.config/mise/config.toml`) < project `mise.toml` < `mise.local.toml` < task-local `vars`.

> The `vars.mise` namespace is **rejected at parse time** (no deprecation period). Use `vars._`.

#### Global Task Settings

| Setting | Type | Default | Env Var | Description |
|---------|------|---------|---------|-------------|
| `task.output` | string | unset | `MISE_TASK_OUTPUT` | prefix, interleave, keep-order, replacing, timed, quiet, silent |
| `task.timeout` | duration | unset | `MISE_TASK_TIMEOUT` | Default timeout. Per-task cannot exceed this. |
| `task.timings` | bool | unset | `MISE_TASK_TIMINGS` | Show elapsed time per task (shown by default with `prefix` output) |
| `task.skip` | string[] | `[]` | `MISE_TASK_SKIP` | Tasks to skip by default |
| `task.skip_depends` | bool | `false` | `MISE_TASK_SKIP_DEPENDS` | Skip dependencies |
| `task.run_auto_install` | bool | `true` | `MISE_TASK_RUN_AUTO_INSTALL` | Auto-install missing tools |
| `task.show_full_cmd` | bool | `false` | `MISE_TASK_SHOW_FULL_CMD` | Disable command truncation in output |
| `task.disable_paths` | string[] | `[]` | `MISE_TASK_DISABLE_PATHS` | Paths to exclude from task discovery |
| `task.remote_no_cache` | bool | unset | `MISE_TASK_REMOTE_NO_CACHE` | Always fetch latest remote tasks |
| `task.source_freshness_hash_contents` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_HASH_CONTENTS` | Use blake3 content hashing instead of mtime |
| `task.source_freshness_equal_mtime_is_fresh` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_EQUAL_MTIME_IS_FRESH` | Equal mtime = fresh |
| `task.disable_spec_from_run_scripts` | bool | `false` | `MISE_TASK_DISABLE_SPEC_FROM_RUN_SCRIPTS` | Derive the usage spec only from the `usage` field (early opt-out before Tera arg functions are removed) |
| `task.auto_infer` | string[] | `[]` | `MISE_TASK_AUTO_INFER` | **Experimental.** Ecosystems whose scripts become tasks (e.g. `["node"]`) |
| `task.cache_dir` | path | `$MISE_CACHE_DIR/task-artifacts` | `MISE_TASK_CACHE_DIR` | **Experimental.** Task artifact cache location |
| `task.cache_max_size` | string | unset | `MISE_TASK_CACHE_MAX_SIZE` | **Experimental.** SI/IEC units (`500MB`, `2GiB`); LRU eviction |
| `task.cache_max_age` | duration | unset | `MISE_TASK_CACHE_MAX_AGE` | **Experimental.** `0s`/unset disables |
| `task.cache_remote_*` | — | — | — | **Experimental.** See [Remote Task Cache](#remote-task-cache) |
| `task.monorepo_depth` | int | `5` | `MISE_TASK_MONOREPO_DEPTH` | Subdirectory depth for monorepo discovery |
| `task.monorepo_exclude_dirs` | string[] | `[]` | `MISE_TASK_MONOREPO_EXCLUDE_DIRS` | Empty ⇒ built-ins (`node_modules`, `target`, `dist`, `build`); any custom value **replaces** them |
| `task.monorepo_respect_gitignore` | bool | `true` | `MISE_TASK_MONOREPO_RESPECT_GITIGNORE` | Honor `.gitignore` in monorepo discovery |
| `jobs` | int | `8` | `MISE_JOBS` | Max concurrent task execution |

> **Deprecated flat aliases** (`task_output`, `task_timeout`, `task_timings`, `task_skip`, `task_skip_depends`, `task_disable_paths`, `task_remote_no_cache`, `task_run_auto_install`, `task_show_full_cmd`) **began warning in 2026.8.0** and are **removed in 2027.2.0**. Use the dotted `task.*` forms.
>
> **`jobs` default:** the global `jobs` setting is **8** (verified: `mise settings get jobs` → `8`), but the per-command `-j/--jobs` flag on `run`/`exec`/`install`/`use` documents a default of **4**. Set it explicitly if it matters.

---

## Running Tasks (CLI)

### Commands

| Command | Description |
|---------|-------------|
| `mise run <task>` / `mise r <task>` | Execute task |
| `mise <task>` | Shorthand (discouraged in scripts — future mise versions may add conflicting commands) |
| `mise run` (no args) | Runs the `default` task |
| `mise tasks` / `mise tasks ls` | List all tasks |
| `mise tasks --hidden` | Include hidden tasks |
| `mise tasks --global` / `--local` | Filter by config scope |
| `mise tasks --all` | Entire monorepo, including siblings |
| `mise tasks --name-only` | One name per line (fzf-friendly) |
| `mise tasks --sort <name\|alias\|description\|source>` | Sort order (`--sort-order asc\|desc`) |
| `mise tasks deps` | Show dependency tree |
| `mise tasks deps --compact` | Expand each shared subtree once, mark repeats `(already shown)` (2026.7.18+) |
| `mise tasks deps --dot` | DOT format for Graphviz |
| `mise tasks graph [--explain] [--json]` | **Experimental.** Inspect the workspace project graph |
| `mise tasks info <task>` | Show task details (JSON output includes `config_sources`) |
| `mise tasks add <name> -- <cmd>` | Create task via CLI (`--depends`, `--run-windows`) |
| `mise tasks edit <task>` | Edit/create task in `$EDITOR` (respects configured `includes`) |
| `mise tasks validate [--errors-only] [--json]` | Validate task definitions |
| `mise watch <task>` / `mise w <task>` | Watch and re-run on file changes (requires watchexec) |
| `mise run a ::: b ::: c` | Run multiple tasks in parallel (with separate args) |

### Execution Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--jobs <N>` | `-j` | Parallel job limit (default 4 on this command) |
| `--force` | `-f` | Ignore source/output freshness |
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

**Task output cache flags** (experimental): `--task-cache <read-write|read-only|write-only|off|local-only>` (default `read-write`), `--task-cache-explain`, `--task-cache-explain-json`, `--task-cache-stats`.

**Affected-set flags** (experimental): `--affected`, `--affected-base <REV>`, `--affected-head <REV>`, `--affected-explain`, `--affected-json`.

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
| `--deny-write` | Block all filesystem writes (except `/tmp`) |

> **mise's own flags must precede the task name:** `mise run --silent build`, not `mise run build --silent`. Extra args after the task go to the **last** command.

> **Breaking change (2026.7.6):** `--quiet` / `quiet = true` / `MISE_QUIET=1` **no longer collapse task output** to un-prefixed interleave — they preserve the resolved style. Use `--output quiet` or `-o interleave` for the old behavior.

> **Ctrl-C is an interruption, not a failure (2026.8.0+).** `mise run` stops starting new work and **exits 130**, without printing `no exit status` / `task failed`, while still allowing post-dependency cleanup.

### Output Modes

- `prefix` — Default when `jobs > 1`; each line prefixed with task name
- `interleave` — Default when `jobs == 1`; print as output arrives
- `keep-order` — Stream one task's output live; buffer others; print in definition order
- `replacing` — Replace stdout on each new line (similar to `mise install`)
- `timed` — Show only stdout lines that took >1s
- `quiet` — Print only task stdout/stderr, nothing from mise itself
- `silent` — Print nothing from tasks or mise

Set via `--output`, `task.output`, `MISE_TASK_OUTPUT`, or the per-task `output` field. Style is **orthogonal** to verbosity — `MISE_TASK_OUTPUT=prefix --quiet` keeps prefixes while silencing mise's own messages.

### `mise watch` Flags

Wrapper around `watchexec` (install separately: `mise use -g watchexec@latest`). By default watches the task's `sources` glob set.

| Flag | Short | Description |
|------|-------|-------------|
| `--watch <PATH>` | `-w` | Watch path recursively (repeatable; `/dev/null` disables path watching) |
| `--watch-non-recursive <PATH>` | `-W` | Non-recursive watch |
| `--watch-file <PATH>` | `-F` | File listing paths to watch (`-` = stdin) |
| `--exts <EXTS>` | `-e` | Comma-separated extension filter |
| `--filter <PATTERN>` | `-f` | Include glob |
| `--filter-prog <EXPR>` | `-J` | **Experimental** jaq-based filter |
| `--ignore <PATTERN>` | `-i` | Exclude glob |
| `--no-vcs-ignore` | — | Include `.gitignore`d files |
| `--poll <INTERVAL>` | — | Polling instead of native fs events (default `30s`) |
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
| `--wrap-process <MODE>` | — | `group` (default), `session`, `none` — honored on macOS since 2026.7.18 |
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

`mise run 'test:*:local'` matches only `test:units:local`; `mise run 'test:**:local'` also matches `test:e2e:happy:local`.

### Default Task

```toml
[tasks.default]
depends = ["build", "test"]
run = "echo 'Ready!'"
```

---

## Task Dependencies and Freshness

### Dependencies

```toml
[tasks.deploy]
depends = ["build", "test", "lint"]  # Run before (parallel by default)
depends_post = ["notify"]             # Run after (even on failure)
wait_for = ["db:migrate"]             # Wait if running, don't add
```

If a dependency fails, the dependent task skips execution. Shared dependencies run once.

### Freshness with sources/outputs

```toml
[tasks.build]
sources = ["Cargo.toml", "src/**/*.rs"]
outputs = ["target/release/myapp"]
run = "cargo build --release"
# Skips if sources unchanged and outputs exist
```

A task is fresh when output mtime is newer than the newest source. `sources` respect `.gitignore` by default.

**Exclusions** use gitignore-style `!` prefixes (escape a literal `!` path with `\!`); later entries override earlier ones. Since 2026.7.15, `outputs` supports the same ordered exclusions and re-inclusions:
```toml
sources = ["src/**/*.ts", "!src/**/*.test.ts", "!src/**/*.spec.ts"]
outputs = ["dist/**", "!dist/*.map"]
```
Excluded output paths are omitted from freshness hashes and cache archives; existing excluded files are preserved on restore.

Brace globs work in both (2026.8.0+): `sources = ["{src,lib}/**/*.ts"]`.

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

Redactions intercept task output line-by-line; tasks with `raw = true` bypass them. A variable marked `redact = false` opts **out** of matching `redactions` patterns (2026.8.0+), so a short non-sensitive value no longer pollutes the global scrubber.

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

## Task Output Caching (Experimental)

Introduced in **v2026.7.15** and expanded through **v2026.8.1**. Distinct from freshness checking: instead of merely *skipping* a task, mise **restores its declared outputs and replays its stdout/stderr** from a content-addressed archive. Requires `experimental = true`.

> There is no dedicated upstream doc page — this is folded into `https://mise.jdx.dev/tasks/task-configuration.html`.

### Local Artifact Cache

Requires `sources` plus either explicit `outputs` or `outputs = []`.

```toml
[tasks.build]
run = "cargo build --release"
sources = ["src/**/*.rs", "Cargo.toml"]
outputs = ["target/release/myapp"]
cache = { enabled = true }

# Result-only caching for checks that produce no files (lint/test/typecheck)
[tasks.lint]
run = "cargo clippy"
sources = ["src/**/*.rs"]
outputs = []                 # caches the successful result + replayable logs, no archive
cache = { enabled = true }
```

| `cache` key | Type | Default | Description |
|-------------|------|---------|-------------|
| `enabled` | bool | `false` | Opt in |
| `env` | string[] | `[]` | Ambient env var names folded into the cache key |
| `command_inputs` | string[] | `[]` | Commands whose text + stdout/stderr hash into the key (2026.7.16+) |
| `audit` | bool | `false` | Report undeclared reads/writes (strace; Linux only) |

**Cache key** = source contents + task config + args + declared/allowlisted ambient env + resolved tools + OS + arch, plus the artifact keys of upstream dependencies. Cache failures degrade to misses or warnings — never a task failure. Artifacts are checksum-verified on restore.

```toml
[tasks.build]
run = "cargo build --release"
sources = ["src/**/*.rs"]
outputs = ["dist/app"]
cache = { enabled = true, env = ["CI"], command_inputs = ["rustc --version"] }
```

**Per-run control and inspection:**
```bash
mise run --task-cache off build          # read-write (default) | read-only | write-only | off | local-only
mise run --task-cache-explain build      # structural breakdown of the key; works with --dry-run
mise run --task-cache-explain-json build # JSON Lines, no run
mise run --task-cache-stats build
mise cache task build                    # stored size, restorable bytes, saved time, last access, outputs
mise cache task build --json
mise cache clear --task build            # leaves working-tree outputs untouched
```

`--task-cache-explain` reports input categories, counts, env/var names and presence, and platform — and deliberately emits **no secret-derived hashes**.

**Size and age caps** (2026.8.1+), independent of `cache_prune_age`:
```toml
[settings]
task.cache_dir = "~/.cache/mise/task-artifacts"
task.cache_max_size = "2GiB"    # LRU eviction after writes
task.cache_max_age = "30d"      # expired entries rejected on restore
```

### Cache Inputs

Reusable and global input declarations live in `[task_config]`:

```toml
[task_config]
global_inputs = ["mise.toml", "@group:lockfiles"]
global_env = ["CI", "NODE_ENV"]
global_pass_through_env = ["GITHUB_TOKEN"]   # available under deny_env, NOT part of the key

[task_config.input_groups]
lockfiles = ["package-lock.json", "pnpm-lock.yaml"]
rust = ["Cargo.toml", "Cargo.lock", "@group:lockfiles"]   # groups nest

[tasks.build]
sources = ["src/**/*.rs", "@group:rust"]
outputs = ["dist/app"]
cache = { enabled = true }
```

`pass_through_env` is also a per-task field. Use it for tokens: they stay available to the command (even under `deny_env`) without making every token rotation a cache miss.

### Remote Task Cache

Added **v2026.8.1**. A composite store: reads local first, promotes remote hits, and commits locally **then** mirrors writes so a remote failure never loses a local hit. Requests are hardened, verified, and streamed. **HTTPS is enforced** except for loopback dev endpoints.

| Setting | Env | Notes |
|---------|-----|-------|
| `task.cache_remote_url` | `MISE_TASK_CACHE_REMOTE_URL` | Endpoint |
| `task.cache_remote_namespace` | `MISE_TASK_CACHE_REMOTE_NAMESPACE` | **Required when `cache_remote_url` is set**; opaque org/repo namespace isolating entries |
| `task.cache_remote_mode` | `MISE_TASK_CACHE_REMOTE_MODE` | `read-write` (default), `read-only`, `write-only` |
| `task.cache_remote_token` | `MISE_TASK_CACHE_REMOTE_TOKEN` | `Authorization: Bearer`. **Global-config only.** |
| `task.cache_remote_token_file` | `MISE_TASK_CACHE_REMOTE_TOKEN_FILE` | Re-read before each request (rotating creds, K8s projected SA tokens). **Global-config only.** |
| `task.cache_remote_oidc_audience` | `MISE_TASK_CACHE_REMOTE_OIDC_AUDIENCE` | GitHub Actions OIDC. **Global-config only.** |

**Credential precedence:** explicit bearer token → global-only token file → GitHub Actions OIDC. The credential settings are global-only so a shared project config cannot supply them.

Protocol reference: `https://mise.jdx.dev/tasks/remote-cache-protocol.html` (protocol version 1). Upstream notes remote caching is experimental and "not yet configurable" beyond these settings.

---

## Dev Tools Management

### Backends Overview

mise supports 18 backends plus custom backend plugins. The registry assigns tools to backends by an **acceptance-tier** preference order — prefer the highest tier available for a given tool:

| Tier | Backends | When chosen |
|------|----------|-------------|
| **1 — Preferred** | **aqua**, **github**, **gitlab** | aqua offers the most features + security (cosign/SLSA/attestation/minisign, native Windows, no plugin). github/gitlab for releases not yet in aqua. |
| **2 — High bar** | **conda** | Lower bar than tier 3 because mise's conda backend needs no separately-installed package manager. |
| **3 — Very high bar** | **pipx**, **npm**, **gem**, **go**, **cargo**, **dotnet** | Depend on a separately-installed runtime on PATH; silently bind tools to whichever runtime was available at install time. |
| **Other** | **forgejo**, **http**, **s3**, **spm**, **pkgx** | forgejo (Codeberg default); http for direct URLs; s3 for private buckets; spm for Swift; pkgx (experimental) for the pkgx pantry. |
| **Not accepted for new registry entries** | **vfox**, **asdf**, **ubi** | vfox/asdf rejected for supply-chain reasons; ubi deprecated. |

| Backend | Status | Description |
|---------|--------|-------------|
| **aqua** | stable | Most features, best security (cosign/SLSA/attestation/minisign). No plugins needed. Native Windows. Registry compiled into the mise binary; the aqua CLI is never used. |
| **github** | stable | GitHub releases with auto OS/arch/libc asset detection |
| **gitlab** | stable | GitLab releases |
| **forgejo** | stable | Forgejo/Codeberg releases |
| **http** | stable | Direct HTTP/HTTPS downloads with URL templating |
| **s3** | stable | S3/MinIO/S3-compatible private buckets (AWS SDK credential chain) |
| **pipx** | stable | Python CLIs in isolated environments (uses uvx by default) |
| **npm** | stable | Node packages via the embedded `aube` installer — **node/npm not required to install** |
| **go** | stable | Go packages (requires compilation) |
| **cargo** | stable | Rust packages (uses binstall by default) |
| **gem** | stable | Ruby gems |
| **conda** | stable | Single conda packages direct from anaconda.org (no conda/mamba needed) |
| **dotnet** | stable | .NET tools |
| **spm** | stable | Swift packages |
| **pkgx** | **experimental** | pkgx pantry packages (bottles from dist.pkgx.dev); requires `experimental = true` |
| **ubi** | **DEPRECATED** | Universal Binary Installer — migrate to `github` |
| **vfox** | stable | **The recommended plugin system**; cross-platform, Windows-supported; default plugin backend on Windows |
| **asdf** | **legacy** | asdf plugins (no Windows; disabled by default on Windows) |
| **core** | stable | Built into the binary: bun, deno, elixir, erlang, go, java, node, python, ruby, rust, swift, zig |

**Backend capability matrix:**

| Feature | Core | Lang PMs | aqua | github | Backend Plugins | Tool Plugins | asdf |
|---------|------|----------|------|--------|-----------------|--------------|------|
| Speed | ✅ | ⚠️ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| Security | ✅ | ⚠️ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Windows | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Env Vars | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Custom Scripts | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |

**Backend selection priority:** explicit spec (`aqua:owner/repo`) → `MISE_BACKENDS_<TOOL>` env override (highest — overrides registry *and* alias) → registry lookup → core tools → fallback.

**Registry override** per tool (SHOUTY_SNAKE_CASE; `my-tool` → `MISE_BACKENDS_MY_TOOL`):
```bash
export MISE_BACKENDS_PHP='vfox:mise-plugins/vfox-php'
mise install php@latest
```

**Disable backends** (affects new installs only; does not uninstall existing):
```bash
mise settings disable_backends=asdf
```

**Discovery:**
```bash
mise registry                    # list everything
mise registry --backend aqua     # filter by backend
mise registry --json --security  # per-backend security info (slower)
mise registry --hide-aliased
mise use                         # interactive selector
mise search <query>              # -m equal|contains|fuzzy (default fuzzy)
```

**Floating registries** (`registry_floating = true`, default `false`) fetch the shorthand registry published with the latest mise release plus the current official aqua registry, instead of the release-pinned snapshots. Bundled snapshots remain the fallback, and fast/offline commands never refresh. Opt-in, because a floating registry may contain changes made after your installed mise was tested.

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

# Arbitrary nested options work for any backend
[tools."custom:my-backend".cache.redis]
host = "redis.example.com"
port = 6379
```

Nested options are internally flattened to dot notation (`platforms.macos-x64.url`, `cache.redis.port`).

CLI tool-option syntax uses brackets: `mise use "conda:ruff[channel=bioconda]"`, `mise use "github:oxc-project/oxc[matching=oxlint,rename_exe=oxlint]@apps_v1.69.0"`.

**Version formats:**

| Format | Example | Description |
|--------|---------|-------------|
| Exact | `"20.0.0"` | Specific version |
| Prefix | `"20"` | Latest matching prefix |
| Latest | `"latest"` | Most recent stable |
| `lts` | `"lts"` | LTS release (node); also `lts-iron`, `lts-jod`, `lts-krypton` |
| `prefix:<P>` | `"prefix:1.19"` | Latest matching prefix (explicit form — needed where a bare `1.20` would be exact, e.g. Go) |
| `ref:<SHA>` | `"ref:master"` | Compile from git ref |
| `path:<PATH>` | `"path:./shfmt"` | Use custom binary |
| `sub-<PARTIAL>:<BASE>` | `"sub-2:lts"` | **Numeric subtraction**, not "Nth previous release": resolve BASE, subtract PARTIAL's components, resolve as a prefix. `sub-2:lts` → 20 becomes 18; `sub-0.1:latest` → 3.11 becomes 3.10. |
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

Tool versions may now contain a colon (2026.8.1+), so templated versions like `{{ exec(...) | split(pat=': ') | last }}` no longer break config loading.

### Per-Tool Options

Universal options supported by every backend:

| Option | Type | Description |
|--------|------|-------------|
| `version` / `prefix` / `ref` / `path` | string | The selector — exactly one required in table form |
| `os` | string \| string[] | Restrict to OS/arch: `"linux"`, `"macos"`/`"darwin"`, `"windows"`/`"win"`. Combos: `"linux/x64"`, `"macos/arm64"`. Arches: `arm64`/`aarch64`, `x64`/`x86_64`/`amd64`. A bare OS matches any arch; an entry with `/` requires both to match. Non-matching ⇒ mise skips installing **and using** the tool. |
| `install_env` | table | Environment variables injected during install (and tool-level `postinstall`) |
| `postinstall` | string | Command after successful install. `MISE_TOOL_INSTALL_PATH` available; the tool's bin dir is on PATH. |
| `depends` | string \| string[] | **Install-graph ordering only**, and only for tools already in the current install set. Does not add tools to the PATH used by vfox install hooks — declare those in the plugin's `metadata.lua`. (Exception: since 2026.7.18 the **asdf** backend does put `depends` tools on the PATH for `bin/download`/`bin/install`.) |

> `github_attestations` is **not** universal — it is a `github:` backend tool option. The separate *global* `github_attestations` setting (default `true`) applies to supported tools.

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

Tool option values support template expansion referencing `env.*` and `vars.*` (including values produced by `_.source`/`_.file`); nested option arrays/tables are Tera-rendered too (2026.7.7+).

### Backend-Specific Configuration

#### Aqua Backend (Preferred)

```toml
[tools]
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:cli/cli" = { version = "latest", symlink_bins = true }  # Filtered .mise-bins directory
"aqua:flutter/flutter" = { version = "3.32.8", channel = "stable" }
"aqua:scenarigo/scenarigo" = { version = "0.21.0", vars = { go_version = "1.24" } }
```

**Tool options:** `symlink_bins` (bool — filters bundled bins into `.mise-bins`, using the registry's `files` field when defined; solves e.g. `aws-cli` bundling its own Python), `vars` (table, aqua registry template variables), `channel`, `prerelease` (bool — no effect when the package uses the `github_tag` version source, since git tags carry no prerelease flag; drafts always excluded).

**Security (all default `true`):**

| Setting | Default | Description |
|---------|---------|-------------|
| `aqua.cosign` | `true` | Verify cosign signatures |
| `aqua.slsa` | `true` | Verify SLSA provenance |
| `aqua.github_attestations` | `true` | Verify GitHub Artifact Attestations |
| `aqua.minisign` | `true` | Verify minisign signatures |
| `aqua.baked_registry` | `true` | Use built-in (compiled-in) aqua registry |
| `aqua.registries` | none | Extra registry sources (string[], `MISE_AQUA_REGISTRIES`), loaded before the baked-in registry |
| `aqua.registry_url` | none | **Deprecated** — warns 2026.12.0, removed 2027.12.0. Use `aqua.registries`. |
| `aqua.registry_cache_ttl` | `1w` | Registry source freshness TTL (`0s` = always refresh) |
| `aqua.cosign_extra_args` | none | Additional cosign arguments (string[]) |

`aqua.registries` accepts a repository URL, a direct `registry.yaml`/`.yml` URL, or an absolute `file://` directory (local sources bypass the download cache, so edits are read on the next load). Packages resolve by checking configured registries in order, then the baked registry.

> **Aqua registry aliases are local to the registry that defines them** — use `[tool_alias]` to point a mise shorthand at a package from another registry.

Aqua verification is native Rust (no cosign/slsa-verifier/gh CLIs) covering GitHub attestations, cosign, SLSA, minisign, and SHA256/512/1/MD5 checksums (always on). Verification failure aborts the install.

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

**Tool options:** `asset_pattern`, `additional_asset_patterns`, `matching`, `matching_regex`, `version_prefix` (default `v`), `platforms` (per-platform `asset_pattern`/`checksum`/`size`), `strip_components`, `bin`, `rename_exe`, `bin_path`, `filter_bins`, `checksum`, `size`, `no_app`, `api_url`, `prerelease`, `github_attestations`.

**Precedence and interaction rules:**
- `asset_pattern` **takes precedence** over `matching`/`matching_regex`, which are then silently ignored — an invalid `matching_regex` is never consulted and never reported.
- `matching` + `matching_regex` together = logical **AND**. `matching_regex` is case-sensitive; prefix `(?i)` for insensitive.
- `matching` also **scopes verification**: checksum lookup and SLSA provenance discovery narrow to the selected asset, so a multi-binary release can't verify one binary against another's provenance.
- **`matching` is NOT part of the install path** — paths are keyed by tool name + version only. Two `github:owner/repo` entries with different `matching` collide and the second overwrites the first. Give each binary its own `[tool_alias]`.

**`rename_exe` table form (2026.7.13+)** exposes several executables from one archive:
```toml
[tools."github:DanielGavin/ols"]
version = "latest"
rename_exe = { "ols-*" = "ols", "odinfmt-*" = "odinfmt" }
```

**`additional_asset_patterns` (2026.7.14+)** overlays multiple release archives from the same tag into one install dir. Each pattern must select exactly one archive; supplemental assets must be archives (bare binaries unsupported) and are extracted **without** the primary asset's `strip_components`/`bin`/`rename_exe`; on path collision the later archive wins. Each supplemental artifact is independently locked and verified, and `--locked` fails if the recorded set no longer matches.
```toml
[tools."github:ollama/ollama"]
version = "latest"
additional_asset_patterns = ["ollama-linux-amd64-rocm.tgz"]
```

> Use `[tool_alias]` for **independent** binaries (each gets its own install dir); use `additional_asset_patterns` when several archives must compose **one** runnable tool.

**`bin_path` templating:** `{{ version }}` plus the **functions** `{{ os() }}` and `{{ arch() }}`, which take remap kwargs — `{{ arch(x64="x86_64", arm64="aarch64") }}`. There are **no bare `{{ os }}` / `{{ arch }}` variables and no `{{ x86_64_arch }}`-style aliases.** Use a single-quoted TOML string when the template contains double quotes.

**Binary path lookup order (github/gitlab/forgejo):** `bin_path` → `bin/` in install path → install-path root if it holds an executable → subdirectories containing `bin/` → immediate subdirs with any executable → root of extracted dir.

**Asset autodetection** scores OS compatibility, arch compatibility, libc variant (gnu/musl on Linux, msvc on Windows), archive-format preference, and build type (avoiding debug/test builds). `.exe` assets are preferred on Windows; OS package/installer assets (`.apk`, `.deb`, `.rpm`, DMG/PKG, MSI/MSIX/AppX) are skipped; `linux-musl` is a fallback for Android/Termux.

**Auth settings:** `github.credential_command`, `github.gh_cli_tokens` (default `true`), `github.github_attestations`, `github.slsa`, `github.use_git_credentials` (default `false`). OAuth device-flow: `github.oauth_client_id`, `github.oauth_api_url`, `github.oauth_auth_url`, `github.oauth_export_env` (default `GITHUB_TOKEN`), `github.oauth_open_browser`, `github.oauth_scopes`. Env: `MISE_GITHUB_TOKEN`. Debug with `mise token github [--unmask]`.

#### GitLab / Forgejo Backends

Same option surface as GitHub (`asset_pattern`, `matching`, `matching_regex`, `platforms`, `bin`, `bin_path`, `rename_exe`, `filter_bins`, `checksum`, `size`, `strip_components`, `no_app`, `api_url`). Forgejo also supports `prerelease` and defaults `api_url` to `https://codeberg.org/api/v1`. `prerelease` has **no effect on GitLab**.

```toml
"gitlab:gitlab-org/gitlab-runner" = "16.8.0"
"forgejo:user/repo" = "latest"  # Defaults to Codeberg
```

**GitLab auth order:** `MISE_GITLAB_ENTERPRISE_TOKEN` → `MISE_GITLAB_TOKEN` → `GITLAB_TOKEN` → credential_command → `gitlab_tokens.toml` → `glab` CLI → git credential fill. Settings: `gitlab.glab_cli_tokens` (default `true`), `gitlab.use_git_credentials` (default `false`).

**Forgejo auth order:** `MISE_FORGEJO_ENTERPRISE_TOKEN` → `MISE_FORGEJO_TOKEN` → `FORGEJO_TOKEN` → credential_command → `forgejo_tokens.toml` → `fj` CLI → git credential fill. Setting: `forgejo.fj_cli_tokens` (default `true`).

A `credential_command` runs in the configured default inline shell and receives `MISE_CREDENTIAL_HOST` and `MISE_CREDENTIAL_PROVIDER`. The legacy single positional hostname argument warns from **2026.11.0** and is removed in **2027.11.0**. For security, `*.credential_command` is **global-config-only** — it is stripped from project/local `mise.toml`.

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

**URL template functions:** `{{ version }}`, `{{ os() }}`, `{{ arch() }}`, `{{ os_family() }}`. Remapping: `{{ os(macos="darwin") }}`, `{{ arch(x64="amd64") }}`.

**Platform keys:** `macos-x64`, `macos-arm64`, `linux-x64`, `linux-arm64`, `windows-x64`, `windows-arm64` (`darwin`/`amd64` variants also accepted; mise will also make sense of near-misses like `darwin-aarch64`).

**Checksums:** `checksum`, plus `checksum_url` and `checksum_expr`. `checksum_url` resolves checksums for **all** target platforms without downloading the artifacts, so one machine can produce a complete cross-platform lockfile; it accepts an individual checksum file, a SHASUMS-style file, or a manifest, with the algorithm detected from the filename (`*.sha512`, `SHA512SUMS`, `*.md5`, `*.b3`; default sha256). `checksum_expr` is an expr-lang expression over `body`, `version`, `os`, `arch`, `url`, `filename` returning a qualified `algo:hash`.

**Version discovery options:**
- `version_list_url` — plain text, line-separated, JSON array of strings, JSON array of objects (`version`/`tag_name`), or `{"versions":[…]}`; `v` prefixes auto-stripped
- `version_regex` — first capturing group (whole match if no group)
- `version_json_path` — jq-like path (`.[]`, `.field[]`, `.data.versions[]`, `.[?field=value]`)
- `version_expr` — expr-lang over `body`. **Takes precedence** over regex/json_path.

> **expr-lang gotchas:** write the predicate placeholder as `{ #... }` **with a space** after `{`, because `{#` is the Tera comment delimiter. Index a map by runtime value with `[version + ""]` — a bare `[version]` is read as the literal key `"version"`.

**Caching:** `$MISE_CACHE_DIR/http-tarballs/`, keyed by Blake3 hash of file content plus `strip_components`; installs are symlinks into the cache, so identical tarballs are shared across tools. Each entry carries `metadata.json`. Auto-pruned after 30 days.

**http bin path lookup** has only 4 steps (no install-root-executable step): `bin_path` → `bin/` → subdirs containing `bin/` → root.

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

**Options:** `url` (required), `endpoint`, `region`, `checksum`, `size`, `bin`, `rename_exe`, `bin_path`, `format`, `strip_components`, `platforms`, `version_list_url`, `version_json_path`, `version_expr`, `version_prefix` (also usable as an S3 key prefix for object-listing discovery), `version_regex`.

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

**Tool options:** `features` (**skips binstall**), `default-features` (default `true`; `false` **skips binstall**), `bin`, `crate` (monorepo select), `locked` (default `true`), `install_env`. `bin`/`crate`/`locked` are passed through and do not skip binstall.

Fallback to `cargo install` happens **only** on binstall exit code **94** (no prebuilt artifact) — other errors do not trigger it. With `cargo.binstall_only = true` there is no fallback. Explicit Git sources always use `cargo install --git`.

Since **2026.7.18**, installs record the effective `features`/`default-features`/`bin`/`crate`/`locked` and auto-reinstall the same version when any change. Feature names are normalized, so reordering or switching string↔array does not force a needless reinstall.

**Settings:** `cargo.binstall` (default `true`), `cargo.binstall_only` (default `false`), `cargo.binstall_quickinstall` (default `false` — mise passes `--disable-strategies compile,quick-install`; `true` → `--disable-strategies compile`), `cargo.binstall_native` (**graduated from experimental in 2026.7.16**; now also discovers conventionally named GitHub release artifacts from a crate's linked repo when `package.metadata.binstall` is absent, and works under the default `locked = true` path — warns 2027.1.0, becomes the default 2027.7.0), `cargo.registry_name`.

#### Pipx Backend

```toml
[tools]
"pipx:black" = "latest"
"pipx:harlequin" = { version = "latest", extras = "postgres,s3" }
"pipx:ansible" = { version = "latest", uvx = false }  # Disable uv
"pipx:ansible-core" = { version = "latest", uvx_args = "--with ansible" }
"pipx:black" = { version = "latest", pipx_args = "--preinstall" }
```

**Tool options:** `extras` (string or array; now applies to git-based installs too), `package_name` (for Git repos whose name differs from the Python distribution name), `pipx_args`, `uvx` (per-tool toggle), `uvx_args`, `install_env`.

**Settings:** `pipx.uvx` (default `true`), `pipx.registry_url` (default `https://pypi.org/pypi/{}/json`).

**Install sources:** PyPI, GitHub (`pipx:psf/black`), Git (`pipx:git+https://...@main`), HTTP ZIP.

Honors `minimum_release_age` via uv `--exclude-newer` (needs `uv >= 0.2.22`) or pip `--uploaded-prior-to`. Reinstall after a Python upgrade: `mise install -f "pipx:*"`.

#### NPM Backend

```toml
[tools]
"npm:prettier" = "latest"
"npm:some-cli" = { version = "latest", allow_builds = ["esbuild"] }  # approve build scripts
"npm:bibtex-tidy" = { version = "latest", allow_low_downloads = true }
```

**Auto-detects package manager:** prefers the embedded `aube` installer, else `npm`, with `aube_cli`/`bun`/`pnpm` as alternatives. Override with `npm.package_manager = "auto|npm|aube|aube_cli|bun|pnpm"` (env: `MISE_NPM_PACKAGE_MANAGER`). Because `aube` is embedded, **Node is only required to run the tool, not to install it**. `aube_cli` installs through a separately installed standalone Aube executable, avoiding Aube's npm compatibility shim.

**Tool options:**

| Option | Applies to | Description |
|--------|-----------|-------------|
| `allow_builds` | aube, aube_cli, pnpm ≥10.4, npm ≥11.16 | Array of package names, or `true` for all. Lifecycle build scripts are **denied by default**. |
| `trust_policy_excludes` | aube / aube_cli | Exempt reviewed packages from trust-policy downgrade checks; supports ranges (`"undici@^5 \|\| >=6 <7"`) |
| `allow_low_downloads` | aube | Bypasses aube's weekly-download popularity gate (**1000** downloads) for the requested package only — transitive deps stay gated |
| `aube_args` / `npm_args` / `pnpm_args` / `bun_args` | respective PM | Extra args (`aube_args` is ignored for embedded aube, which installs in-process) |
| `install_env` | all | Env during install |

**Lifecycle-script defaults:** npm passes `--ignore-scripts=true` (dropped when `allow_builds` is used with npm ≥11.16.0); bun runs no dependency scripts unless `bun_args = "--trust"`; aube/pnpm need `allow_builds`.

**Setting `npm.shell_out`** (default `false`) routes metadata lookups and installs through the npm CLI instead of the built-in aube implementation.

npm tools resolved from a `mise.lock` pin are auto-trusted through the low-download gate — reproducing an existing lockfile no longer needs `allow_low_downloads`.

Supports minimum-release-age protection for transitive dependencies via compatible package managers (`npm >= 11.10.0`, `aube`, `bun >= 1.3.0`, `pnpm >= 10.16.0`). Honors `~/.npmrc` and `NPM_CONFIG_*`.

#### Go Backend

```toml
[tools]
"go:github.com/DarthSim/hivemind" = "latest"
"go:github.com/golang-migrate/migrate/v4/cmd/migrate" = { version = "latest", tags = "postgres" }
"go:github.com/grafana/oats" = "v0.7.1-0.20260703092802-96201f1b8136"  # module pseudo-version
```

**Tool options:** `tags` (string or array → `-tags`), `install_env`. mise sets `GOBIN` to the tool install dir *after* applying `install_env`; use `install_env = { GOPROXY = "direct" }` for unreleased revisions.

#### Gem Backend

```toml
[tools]
"gem:rubocop" = "latest"
```

**Tool option:** `install_env` only. Requires `gem` (Ruby) on PATH. Reinstall after a Ruby upgrade: `mise install -f "gem:*"`.

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

**Tool options:** `filter_bins` (array or comma-string; filtering happens before `swift build` and fails if a listed name isn't an executable product), `artifactbundle` (tri-state: unset = try prebuilt bundle then source; `true` = require a bundle; `false` = always build from source), `artifactbundle_asset` (required when a release has multiple bundles), `provider` (`github`/`gitlab`, default `github`), `api_url`, `install_command` (**2026.7.16+**; source installs only, cannot combine with `filter_bins`, sets `PREFIX` and `MISE_TOOL_INSTALL_PATH`, fails if nothing lands in `bin/`), `install_env`. **Setting:** `spm.artifactbundle_only` (default `false`).

#### Dotnet Backend

```toml
[tools]
"dotnet:GitVersion.Tool" = "5.12.0"
"dotnet:GitVersion.Tool" = { version = "latest", prerelease = true }
```

**Tool options:** `prerelease`, `install_env`.

**Settings:** `dotnet.registry_url` (default `https://api.nuget.org/v3/index.json`), `dotnet.isolated` (default `false`), `dotnet.cli_telemetry_optout`, `dotnet.dotnet_root`. `dotnet.package_flags` is **deprecated** (warns 2026.11.0, removed 2027.11.0) — use the `prerelease` tool option or the global `prereleases` setting.

#### Pkgx Backend (Experimental)

Installs packages from the pkgx pantry without shelling out to the pkgx CLI (bottles fetched from `dist.pkgx.dev`, checksums verified). Requires `experimental = true`.

```toml
[tools]
"pkgx:stedolan.github.io/jq" = "1.7.1"
```

Tool ID is the pantry project name. Supports npm-style semver ranges; runtime env comes from pantry manifests via generated wrappers. Lockfile entries record the main bottle URL + checksum, with transitive deps under `[pkgx-packages]`; under `--locked` it requires a lockfile URL for the current platform rather than doing live pantry resolution.

#### UBI Backend (Deprecated)

**Options:** `exe`, `rename_exe`, `matching`, `matching_regex`, `provider`, `api_url`, `extract_all` (incompatible with `exe`/`rename_exe`), `bin_path` (only meaningful with `extract_all`), `tag_regex`.

> **Migration gotchas:** (1) ubi folds `matching` into the install path so one repo can supply several binaries; `github` keys paths by tool name + version, so different `matching` values collide — give each binary its own `[tool_alias]`. (2) ubi applies substring `matching` only as a *tiebreaker* among assets already matching your OS/arch and skips it when a single asset matches, whereas `github` applies it as a *pre-filter* before autodetection — so you get the named binary or a clear error.

#### vfox & asdf Plugins

**vfox (recommended plugin system):** cross-platform (Win/macOS/Linux), built-in Lua interpreter with HTTP/JSON/archive modules, attestation verification, lock files. **Tool option:** `install_env` (applies to `cmd.exec` during install hooks only — vfox's built-in Lua helpers do not use it). Since 2026.8.1 Lua plugins gain `strip_components = 1` on `archiver.decompress`, plus sorted `file.list`, `file.glob`, and `file.move`.

**asdf (legacy):** Unix-only, bash scripts, needs curl/jq, no Windows, disabled by default on Windows. New asdf tools rarely accepted for supply-chain reasons. **Tool option:** `install_env`. Since 2026.7.18, `depends` tools are on the PATH given to `bin/download` and `bin/install`, and `bin/list-all`/`bin/latest-stable` receive resolved `[env]` values and `_.path` additions.

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
1. **PATH activation** (`mise activate`) — updates PATH per prompt via `mise hook-env`
2. **Shims** (`mise activate --shims`) — tiny intercepting executables
3. **Explicit** (`mise exec`, `mise run`, `mise en`)

```bash
# Shim mode (non-interactive shells — .bash_profile/.zprofile)
eval "$(mise activate zsh --shims)"

# PATH mode (interactive shells — .bashrc/.zshrc)
eval "$(mise activate zsh)"
```

**Best practice:** Use both — shims in your login profile for non-interactive/GUI processes, PATH activation in your rc file for interactive shells.

> How the combination behaves depends on `not_found_auto_install`. **Enabled (the default):** `mise activate` *keeps* the shims dir in PATH, behind the tool paths it manages, so resolved tools win and shims remain an auto-install fallback; `mise doctor` does not flag this. **Disabled:** `mise activate` removes the shims dir from PATH.

**Shims vs PATH:**
- **Shims**: `[env]` vars only load when a shim is called; `watch_files` unsupported; only `preinstall`/`postinstall` hooks work; `which` points to the shim (use `mise which` for the real path)
- **PATH**: full environment, all hooks (`cd`, `enter`, `leave`, `watch_files`), `which` shows the actual binary. When a fuzzy version is active, the PATH entry may use the requested-version symlink (`installs/python/3.15/bin`) rather than the fully resolved patch dir.
- **Recommendation**: PATH for interactive shells; shims for IDEs/cron/CI/non-interactive

Shells with a cd hook: `bash` (`PROMPT_COMMAND`), `zsh` (`chpwd`), `fish` (`fish_prompt`), `xonsh` (`on_chdir`). Without one, `cd a && node -v` on a single line uses the *original* directory's tools — shims always work there.

**`mise reshim`** regenerates shims for **all installed** tools, not just active ones (`-f/--force` removes all shims first). Runs automatically on install/update/remove. Never manually drop binaries in the shims dir — they get deleted.

**`windows_shim_mode`** (default `exe`): `exe` (copies native `mise-shim.exe`; recommended — works with all shells, package managers, and `where.exe`), `file` (`.cmd` batch + extensionless bash script for Git Bash/Cygwin), `hardlink` (NTFS, same filesystem; needs `mise reshim --force` after upgrading mise), `symlink` (needs admin or Developer Mode).

> Windows (2026.8.1+): mise **warns when the generated PATH exceeds ~8191 characters**, the limit at which `cmd.exe` silently drops the variable and every command appears unrecognized. Shims are the documented workaround.

**Tool aliases (remap to different backend):**
```toml
[tool_alias]
node = 'github:company/our-custom-node'
erlang = 'aqua:company/our-custom-erlang'

# Install multiple independent binaries from the same GitHub release
dhall-json = 'github:dhall-lang/dhall-haskell'
dhall-lsp  = 'github:dhall-lang/dhall-haskell'

[tool_alias.node.versions]
lts = '22'
my_custom_20 = '20'
```

```toml
[tools]
dhall-json = { version = "v1.42.2", matching = "dhall-json" }
dhall-lsp  = { version = "latest",  matching = "dhall-lsp-server" }
```

> **Aliases are not an overlay mechanism** — each alias creates a separate install directory. Adding a version alias also creates a symlink (`installs/node/20 -> ./20.x.x`).

> `[alias]` is **deprecated** in favor of `[tool_alias]` (no removal version announced).

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

**Custom plugin repos (`[plugins]`)** — affects new installations only:
```toml
[plugins]
node = "https://github.com/myorg/asdf-node#v2"  # optional #GITREF suffix
my-tool = "vfox:myorg/vfox-my-tool"             # asdf:/vfox:/vfox-backend: prefixes
example = "./plugins/mise-example"              # local path (2026.7.18+)
```

The type prefix is optional — if omitted, mise clones first and detects the type. **Local filesystem paths** (absolute, `~/`, or explicit `./`/`../` relative to the config root) install as **symlinks**, so source edits take effect immediately; use `mise plugins install --force <NAME>` to replace an existing plugin with a local source. `[plugins]` replaces the deprecated `shorthands_file` setting (**removed 2026.12.0**).

> The `[_]` table holds arbitrary user data that mise never parses — useful for sharing values with external tooling.

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

> **Standalone usage scripts** need a separate one-time opt-in: `source <(usage g completion-init bash)` in `~/.bashrc` (zsh: same in `~/.zshrc`; fish: `usage g completion-init fish | source`). Since usage 3.5.6 the completion spec cache lives in `${XDG_CACHE_HOME:-$HOME/.cache}/usage` — **regenerate completion scripts** to pick it up.

### Lockfiles (`mise.lock`)

**Lockfiles are not created automatically.** Enable with the `lockfile` setting, then `touch mise.lock && mise install` (or run `mise lock`). Once one exists, mise keeps it updated as tools are installed or upgraded.

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

Tool entry fields: `version` (required), `backend`, `options` (backend-specific artifact identity), `platforms`. Platform sub-fields: `checksum` (SHA256 or Blake3), `size`, `url`.

**Multiple entries per version** occur when artifact identity depends on more than the platform key — e.g. Swift's per-distro Linux tarballs. Entries match on options **exactly**, so a machine only verifies against its own distro's entry:
```toml
[[tools.swift]]
version = "6.3.1"
backend = "core:swift"
options = { swift_platform = "ubuntu24.04" }
```

```bash
mise lock                       # Update lockfile from current config (installs nothing)
mise lock node python           # Update specific tools only
mise lock --bump                # Advance fuzzy selectors (latest/lts/"20") without installing
mise lock --bump --dry-run --json
mise lock -p linux-x64,macos-arm64   # Add/update entries for specific platform(s)
mise lock --local               # Update mise.local.lock instead of mise.lock
mise lock -g                    # Target global config lockfiles
mise lock --minimum-release-age "30d"
mise lock node@22.15.0          # Pin a version in the lockfile without reinstalling
```

> `mise lock --bump` re-resolves fuzzy version selectors **without installing or touching `mise.toml`**; exact pins resolve to themselves. Use `mise upgrade --bump` to rewrite config pins. `--json` reports only *version-level* changes, so a plain `mise lock --json` typically prints `[]` while still refreshing checksums and URLs.

Per-config lockfiles pair with their config: `mise.toml`→`mise.lock`, `mise.test.toml`→`mise.test.lock`, `mise.local.toml`→`mise.local.lock` (gitignore the local ones). Scoping is strict, so CI that doesn't set `MISE_ENV` depends only on `mise.lock` and dev-tool bumps won't invalidate CI caches.

**Command × lockfile behavior:**

| Command | Installs | Updates `mise.toml` | Updates `mise.lock` |
|---------|----------|---------------------|---------------------|
| `mise use node@22` | Yes | Yes | Yes |
| `mise install` | Yes | No | Yes |
| `mise install node@22.15.0` | Yes | No | **No** (one-off, not config-driven) |
| `mise upgrade` | Yes | No | Yes |
| `mise upgrade --bump` | Yes | Yes | Yes |
| `mise lock` | No | No | Yes |
| `mise lock --bump` | No | No | Yes |

**Backend lockfile support:** full (version+checksum+size+URL) — `aqua`, `http`, `github`, `gitlab`; partial — `vfox` (version+URL+provenance, tool plugins only), `ubi` (version+checksum+size); basic — `core` (version+checksum); version only — `asdf`, `npm`, `cargo`, `pipx`. `pkgx` records bottle URL + checksum plus a `[pkgx-packages]` section. **Provenance support:** `aqua`, `github`, `core:python`, `core:ruby`, `core:zig`.

For the current platform, `mise lock` **downloads the artifact and performs full cryptographic verification at lock time**, so the entry is backed by real verification rather than registry metadata. Cross-platform entries record detected provenance without verifying. `github_attestations = "unavailable"` is a **negative cache entry, not provenance** — SLSA/cosign/minisign/checksum verification still runs, and a later `mise lock` can discover attestations added after release.

Settings: `lockfile` (unset behaves as enabled without conflicting with `locked`), `locked = true` (require lockfile-resolved URLs; blocks API calls — good for CI), `locked_verify_provenance` (default `false`; auto-on with `paranoid`), `lockfile_platforms`.

> ⚠️ **All mise settings are global in scope.** `locked = true` in a project's `mise.toml` applies to *all* tool resolution, including tools from `~/.config/mise/config.toml`. Fix warnings about global tools with `mise lock -g`.

`minimum_release_age` (env `MISE_MINIMUM_RELEASE_AGE`, **default `24h`**) filters fuzzy version requests by release date — supply-chain delay protection. Accepts relative durations (`7d`, `90d`, `6mo`, `1y`) and absolute dates (`2024-06-01`, `2024-06-01T12:00:00Z`); `"0s"` effectively disables it. Versions without release timestamps are included. Explicit pins are never filtered. Only **`npm:` and `pipx:`** forward the cutoff into transitive dependency resolution. Exempt tools with `minimum_release_age_excludes = ["trivy", "npm:*"]` (backend wildcards, registry shorthands, or full IDs), or per-tool:
```toml
[tools.trivy]
minimum_release_age = "1d"
```
Note durations use jiff format — months are `6mo`, not `6m`. The deprecated `install_before` setting maps to this (warns 2026.10.0, removed 2027.10.0).

CI flag: `--locked` (global flag on any command) errors if any version isn't resolved via the lockfile.

### Auto-Install Controls

All are enabled by default and all require the master `auto_install`:

- `auto_install` (default `true`) — master switch
- `exec_auto_install` (default `true`) — `mise x`/`mise r`
- `task.run_auto_install` (default `true`) — the dev-tools doc page calls this `task_auto_install`, which does not exist; use the dotted form
- `not_found_auto_install` (default `true`) — requires at least one existing version, since mise otherwise can't know which tool provides a binary
- `auto_install_disable_tools = ["..."]` — per-tool skip list

### CLI Commands for Tools

```bash
mise use node@22            # Install + activate + write to mise.toml
mise use -g node@22         # Write to global config
mise use --pin node@22      # Pin exact resolved version (e.g. 22.5.1)
mise use -E staging node@22 # Write to mise.staging.toml
mise use --file mise.local.toml node@22   # --file and --path are interchangeable (2026.8.1+)
mise use --remove python    # Remove a tool from config
mise use --dry-run-code     # Exit 1 if changes would be made
mise install node@20        # Install without activation
mise install                # Install all configured tools
mise install -f "gem:*"     # Force reinstall pattern
mise install --monorepo     # Install across all monorepo config roots
mise install --system       # Install to the system data dir (may need sudo)
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
mise tool <TOOL>            # Backend/description/config-source/tool-options for one tool
mise uninstall node@20      # Remove an installed tool version
mise unuse node@20          # Remove one version from config (keeps siblings)
mise link node@custom ./dir # Symlink an external install into mise
mise x python@3.12 -- script.py  # Run with specific tool
mise reshim                 # Rebuild shims
mise registry               # List all available tools
mise backends ls            # List available backends
mise fmt                    # Format mise.toml (sort keys, clean whitespace)
mise outdated               # Check for updates
mise upgrade                # Update versions (respects mise.toml ranges)
mise upgrade --bump         # Bump mise.toml to absolute latest (keeps precision)
mise upgrade --no-prune     # Keep the replaced version (2026.8.1+)
mise upgrade -i             # Interactive selection
mise prune                  # Remove unused versions (destructive; --configs also prunes stale links)
mise lock                   # Update lockfile checksums/URLs
mise search <query>         # Search registry (-m equal|contains|fuzzy)
mise cache clear            # Clear cached downloads
mise cache task <task>      # Inspect a task's cached artifacts
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
mise generate git-pre-commit # Generate pre-commit hook (staged files exposed as STAGED)
mise generate tool-stub     # Generate a standalone tool stub
mise install-into <tool> <path>  # Install a tool into a specific path
mise doctor                 # Diagnose installation issues (mise doctor path)
mise self-update            # Update mise binary
mise mcp                    # Run mise as a Model Context Protocol (MCP) server
mise bootstrap              # Provision a whole machine (alias: bs)
mise oci build|push|run     # Build/inspect OCI container images with mise tools
mise deps add|install|remove  # Project dependency preparation
mise en                     # Spawn a sub-shell with the project env loaded
mise test-tool <tool>       # Verify a tool installs/builds correctly
mise implode                # Remove mise entirely
```

`mise use` writes to the **lowest-precedence file in the highest-precedence directory** (so `mise.toml`, not `mise.local.toml`). Target order: `--global` → `--path`/`--file` → `--env` → `MISE_DEFAULT_CONFIG_FILENAME` → first of `MISE_OVERRIDE_CONFIG_FILENAMES` → `mise.toml`.

> Since 2026.8.0, `--path <dir>` correctly targets a config *inside that directory* for `use`, `unuse`, `set`, `unset`, `dotfiles add`, and the `system` subcommands — previously it was silently discarded when the cwd already had a config in scope, so `mise unuse --path ../other` could remove a tool from the wrong file.

> Since 2026.8.0, `mise unuse node@20` on `node = ["20", "22"]` removes **only the matching version**, preserving the rest (including structured options and `.tool-versions` entries). The whole key is removed only for an unversioned request or after the last version is unused.

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

Value forms (schema `oneOf`, all with `unevaluatedProperties: false`): scalar (string/int/bool; `false` unsets) · `{ value, tools?, redact?, required? }` · `{ default, tools?, redact? }` (**no `required`**) · `{ required, tools?, redact? }` · `{ age = … }` (experimental).

| Sub-key | Type | Meaning |
|---------|------|---------|
| `value` | string\|int\|bool | The value; `false` removes the variable |
| `default` | string\|int | Fallback used only when the var is unset **or empty** |
| `required` | bool \| string | Must be defined pre-mise or in a **later** config file; string form is the error help text |
| `redact` | bool | Redact from logs. `redact = false` opts **out** of matching global `redactions` patterns |
| `tools` | bool | Defer resolution until after tools set their environment |
| `age` | string \| `{value, format}` | Experimental; `format` is `raw` or `zstd` |

```bash
mise set NODE_ENV=development   # Set via CLI
mise set                        # View all (key / value / source table)
mise set -E staging NODE_ENV=staging  # Write to a config environment file
mise set --prompt PASSWORD      # Hidden interactive prompt
cat private.key | mise set --stdin MY_KEY  # From stdin (single key, read to EOF)
mise unset NODE_ENV             # Remove
mise env                        # Export all
mise env --json                 # Export as JSON
mise env --json-extended        # JSON with source + tool attribution
mise env --dotenv               # Export as dotenv
mise env --redacted             # Show only redacted variables
mise env --values               # Show only values
mise env -s bash                # Shell-specific output
```

Env vars resolve **before** tools, so they can configure tool-install subprocesses. Use `tools = true` on a value to defer evaluation until tool paths/versions exist.

> Variables that configure mise itself (`MISE_DATA_DIR`, `MISE_INSTALLS_DIR`, …) are read at process start and **cannot** be set from `[env]` — set them in the shell or CI environment.

### Special Directives (`env._`)

The reserved key `_` is a TOML table for configuration, since nested env vars make no sense.

> **Deprecation:** the older `env.mise.*` spelling is deprecated (removal **2026.12.0**) — use `env._.*`. Likewise, the `value`/`values` keys inside `_.file`/`_.path`/`_.source` objects are deprecated in favor of `path` (a string *or* an array); removal **2026.12.0**.

#### `_.path` — Prepend to PATH

Options: `path` (string or array) / `paths` (array), `tools`. **No `redact`/`expand`.**

```toml
[env]
_.path = ["tools/bin", "{{config_root}}/scripts"]
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }  # Lazy eval after tools
```

Relative paths resolve against `{{config_root}}`.

#### `_.file` — Load from .env/json/yaml/toml files

Options: `path` (string or array), `tools`, `expand`, `redact`.

```toml
[env]
_.file = '.env'
_.file = ['.env', '.env.local', '.env.{{env.MISE_ENV}}']
_.file = { path = ".secrets.yaml", redact = true }
_.file = { path = ".env.json", expand = true }
```

Supported formats: `.env`, `.env.json`, `.env.yaml`, `.env.toml` (plus sops/age-encrypted variants). Auto-load a single dotenv with `MISE_ENV_FILE=.env` (or the `env_file` setting) — note that one searches cwd **and parent directories**, while `_.file` relative paths resolve against `config_root`.

> **Changed behavior (2026.7.14):** values in **structured** files (JSON/YAML/TOML) are **literal by default** again. Previously, with `env_shell_expand` on, every structured value was shell-expanded — corrupting literals like bcrypt-style hashes and potentially pulling in matching process-environment values. Opt back in per file with `expand = true`, which also lets values reference vars defined earlier in the same file, an earlier file, or an earlier `[env]` block. Dotenv files keep dotenvy's same-file expansion regardless. A global `env_shell_expand = false` overrides `expand = true`.

> **Deprecation:** the top-level `env_file`/`dotenv` and `env_path` keys are deprecated (removal **2027.4.0**). Migrate to `_.file` and `_.path`.

#### `_.source` — Source shell scripts

Options: `path` (string or array), `tools`, `redact`.

```toml
[env]
_.source = "./setup-env.sh"
_.source = { path = "my/env.sh", redact = true }
_.source = ["./script_1.sh", "./script_2.sh"]   # ordered
```

Scripts must be sourceable by **bash**; shebangs are ignored. On Windows this requires a real POSIX bash (Git for Windows / MSYS2) — common install locations are probed even if bash isn't on PATH, `MISE_BASH_PATH` overrides, and WSL's `bash.exe` is never auto-selected. Ignored entirely under `safe = true`.

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

Uses `uv venv` when `uv` is on PATH, else `python -m venv`. uv omits pip by default — add `uv_create_args = ["--seed"]`. Activation needs `mise activate`/`mise exec`; **shims alone don't add the venv `bin/` to PATH**.

> This is a **separate codepath** from the `python.uv_venv_auto` setting — `uv_create_args` here is not used by `uv_venv_auto`. For uv-managed projects (with `uv.lock`), prefer `python.uv_venv_auto` (`"source"` or `"create|source"`); the legacy `true` value is being phased out. The `[tools] python = { virtualenv = ".venv" }` tool option is **deprecated** (warned since 2026.7.13, removal ~2027.7.0) in favor of `_.python.venv`.

#### Plugin-Provided Directives

Plugins can register custom `_.<name>` directives (the TOML table lands in the plugin's `ctx.options`):

```toml
[env]
_.my-plugin = {}
_.my-plugin = { option1 = "value1", option2 = "value2" }
_.vault-secrets = { vault_url = "https://vault.example.com", secrets_path = "secret/myapp" }
```

mise loads the plugin, calls its `MiseEnv` hook for env vars and `MisePath` hook for PATH entries, and applies them on `mise env` / shell integration.

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

```toml
[env]
NODE_VERSION = { value = "{{ tools.node.version }}", tools = true }
_.path = { path = ["{{env.GEM_HOME}}/bin"], tools = true }
```

### Profiles / Configuration Environments (`MISE_ENV`)

"Profiles" were renamed **configuration environments**; the `profile` setting / `MISE_PROFILE` is deprecated in favor of `MISE_ENV`.

Three ways to set `MISE_ENV`:
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

Comma-separated supports multiple environments: `MISE_ENV=ci,test` (rightmost wins). Also recognized: `mise/config.{MISE_ENV}.toml`, `.config/mise.{MISE_ENV}.toml`. `MISE_OVERRIDE_CONFIG_FILENAMES` bypasses all of it.

**`.miserc.toml`** is loaded very early — before config discovery and before Settings. Lookup order: cwd + parents (`.miserc.toml`, `.config/miserc.toml`) → `~/.config/mise/miserc.toml` → `/etc/mise/miserc.toml`.

Only **six** settings are `.miserc`-settable:

| Key | Env var | Default |
|-----|---------|---------|
| `env` | `MISE_ENV` | `[]` |
| `auto_env` | `MISE_AUTO_ENV` | unset |
| `ceiling_paths` | `MISE_CEILING_PATHS` | `[]` |
| `ignored_config_paths` | `MISE_IGNORED_CONFIG_PATHS` | `[]` |
| `override_config_filenames` | `MISE_OVERRIDE_CONFIG_FILENAMES` | `[]` |
| `override_tool_versions_filenames` | `MISE_OVERRIDE_TOOL_VERSIONS_FILENAMES` | `[]` |

Its Tera context is limited — `env.*`, `config_root`, `cwd`, `xdg_*`, all filters/tests, and all functions **except** `exec()` and `read_file()`; `mise_env`, `mise_bin`, and `mise_pid` are unavailable. Render failure logs a warning and falls back to raw content.

**Platform auto-envs** (`auto_env` setting): `unix` (**not defined on Windows**); `linux`/`macos`/`windows`; `linux-x64`/`macos-arm64`/`windows-x64`. Files like `mise.windows.toml` and `mise.macos-arm64.toml` load automatically with matching lockfiles (`mise.windows.lock`). Precedence: `unix` < `{os}` < `{os}-{arch}` < explicit `MISE_ENV` entries. Affects **config discovery and lockfile selection only** — platform envs are not added to `{{ mise_env }}` or the `MISE_ENV` passed to tasks. **Disabled by default today; warns from 2026.12.0 when a platform file would newly load; defaults to enabled in 2027.6.0.** Setting it in `mise.toml` does nothing (early-init).

### Required and Redacted Variables

```toml
[env]
# Required — error if not set
DATABASE_URL = { required = true }
DATABASE_URL = { required = "Set postgres connection string" }

# Redacted — hidden from output
API_KEY = { value = "secret_key_here", redact = true }

# Opt a value OUT of a matching redactions pattern
TEST_TOKEN = { value = "not-sensitive", redact = false }

# Pattern-based redactions (top-level, not under [env])
redactions = ["*_TOKEN", "SECRET_*", "API_*"]
```

Required vars are satisfied by a pre-existing environment value or by a config file processed **later** (e.g. `mise.local.toml`). Regular commands (`mise env`) **fail** with the help text; `mise hook-env` (shell activation) **warns and continues** so shell setup isn't broken.

Redaction requires a non-`raw` output mode — tasks with `raw = true` bypass interception. In CI set `MISE_TASK_OUTPUT=prefix` to see full logs *with* redaction applied.

### Secrets (fnox, SOPS, age)

mise documents three approaches:

1. **fnox (recommended, not experimental)** — a separate [@jdx](https://github.com/jdx) project: a full secret manager with remote storage (1Password, AWS Secrets Manager) and remote encryption (AWS KMS). Use `fnox exec -- mise ...` to populate mise's environment.
2. **SOPS (experimental)** — encrypt entire files, load via `env._.file`.
3. **Direct age (experimental)** — encrypt individual env vars inline in `mise.toml`.

**SOPS (age-backed):**

```bash
mise use -g sops age
age-keygen -o ~/.config/mise/age.txt
sops encrypt -i --age "<public key>" .env.json
```

```toml
[env]
_.file = ".env.json"
_.file = { path = ".env.json", redact = true }   # with redaction
```

**Key resolution:** `MISE_SOPS_AGE_KEY` → `MISE_SOPS_AGE_KEY_FILE`/`sops.age_key_file` → `SOPS_AGE_KEY_FILE` → `SOPS_AGE_KEY` → `~/.config/mise/age.txt`.

**Settings:** `sops.age_key`, `sops.age_key_file` (default `~/.config/mise/age.txt`), `sops.age_recipients`, `sops.rops` (default `true`, native Rust impl), `sops.strict` (default `true`).

> The external `sops` CLI has no TOML support. mise decrypts SOPS `.env.toml` only with the default `sops.rops = true`; setting `sops.rops = false` shells out and encrypted TOML fails. age is currently the only supported sops encryption method.

**Direct age encryption:**

The `age` tool is **not** required — support is built into mise. Defaults to your SSH key (`~/.ssh/id_ed25519` or `~/.ssh/id_rsa`) when present.

```bash
mise settings set experimental=true
mise set --age-encrypt DB_PASSWORD=supersecret
mise set --age-encrypt --prompt DB_PASSWORD   # keeps it out of shell history
```

Flags: `--age-encrypt`, `--age-recipient <KEY>` (repeatable), `--age-ssh-recipient <PATH|KEY>` (repeatable), `--age-key-file <PATH>`, `--no-redact`.

```toml
[env]
DB_PASSWORD = { age = { value = "<base64>", format = "zstd" } }
```

`format` is `raw` or `zstd` (zstd used automatically above ~1KB).

**Decryption identity order:** `MISE_AGE_KEY` → `age.identity_files` → `age.key_file` → `~/.config/mise/age.txt` → SSH identities (`age.ssh_identity_files` plus common defaults).

**Encryption recipient defaults** (when `--age-encrypt` is used with no explicit recipients): public keys derived from identities in `~/.config/mise/age.txt`, plus keys inferred from SSH private keys with a matching `.pub`. If none are found, the command errors.

**Settings:** `age.identity_files`, `age.key_file`, `age.ssh_identity_files`, `age.strict` (default `true` — no identity, no decryptable identity, or an invalid payload is a **hard failure** rather than a partially-resolved environment).

Decrypted values are always marked for redaction.

### Templates (Tera)

mise.toml values support Tera templates. **mise 2026.7.1+ uses Tera v2 by default.** The TOML structure itself is not templated and must remain valid TOML.

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
| `config_root` | PathBuf | Project root for the config file (`~/src/foo/.config/mise/config.toml` → `~/src/foo`) |
| `cwd` | PathBuf | Current working directory |
| `mise_bin` | String | Path to mise binary |
| `mise_pid` | String | Process ID |
| `mise_env` | Vec | Configuration environments from `MISE_ENV` (**undefined** if unset; excludes platform envs) |
| `tools` | HashMap | Installed tool info (`.version`, `.path`). Becomes an array with multiple versions (`tools.node[0].version`). Requires `tools = true` in env directives. |
| `usage` | HashMap | Task arguments/flags (task run scripts only). Hyphenated: `usage["dry-run"]`. Not shell-escaped. |
| `vars` | HashMap | Values from `[vars]` |
| `xdg_cache_home` / `xdg_config_home` / `xdg_data_home` / `xdg_state_home` | PathBuf | XDG directories |

**Key Tera functions:**

| Function | Description |
|----------|-------------|
| `exec(command, [cache_key], [cache_duration])` | Execute shell command, return stdout. `cache_duration="1d"`. **Runs whenever the template renders, including during `--dry-run`** — keep it side-effect free. Blocked under `safe = true`. |
| `get_env(name, [default])` | Get env var from the **original process** env (compat helper; prefer `env.*`). `default` applies only when absent, not when empty. |
| `arch()` | System architecture (`x64`, `arm64`) |
| `os()` | Operating system (linux, macos, windows) |
| `os_family()` | OS family (unix/windows) |
| `num_cpus()` | CPU count |
| `now([timezone])` | Current datetime — **Tera v2 signature**; defaults to UTC, accepts IANA names |
| `choice(n, alphabet)` | Random n-char string |
| `read_file(path)` | Read file contents. Blocked under `safe = true`. |
| `range(end, [start], [step_by])` | Integer array |
| `get_random(start, end, [seed])` | Random integer (`seed` makes it reproducible) |
| `task_source_files()` | Resolved source file paths (task scripts only); unmatched patterns omitted |
| `throw(message)` | Raise error |

**Key Tera filters:**

| Filter | Description |
|--------|-------------|
| `lower`, `upper`, `capitalize`, `title` | Case transforms |
| `kebabcase`, `snakecase`, `shoutysnakecase`, `lowercamelcase`, `uppercamelcase` | Case conversion |
| `slug` / `slugify`, `striptags`, `spaceless` | Text normalization |
| `trim`, `trim_start`, `trim_end`, `truncate(length)` | Whitespace / shortening |
| `replace(from, to)`, `regex_replace(pattern, rep)` | Substitution |
| `quote` | Escape and quote string |
| `split(pat)`, `join(sep)`, `shuffle([seed])` | Array operations |
| `first`, `last`, `length`, `reverse` | Collection operations |
| `basename`, `dirname`, `extname`, `file_stem`, `join_path` | Path operations |
| `absolute`, `canonicalize` | Path resolution (`canonicalize` throws if missing; `absolute` doesn't require existence) |
| `file_size`, `last_modified` | File metadata |
| `hash([algorithm], [len])` | SHA256 (default) or BLAKE3 hashing |
| `hash_file([len])` | File BLAKE3 hash |
| `b64_encode`, `b64_decode`, `json_encode([pretty])` | Encoding |
| `date(format, [timezone])` | Format datetime (jiff crate) |
| `default(value)` | Fallback for undefined/empty |
| `abs`, `filesize_format`, `format(spec)` | Numeric formatting |
| `urlencode`, `urlencode_strict` | URL-safe encoding |
| `map(attribute)`, `concat(with)` | **Deprecated v1 compat** — use comprehensions / spread |

**Tera tests:**

| Test | Description |
|------|-------------|
| `defined` | Variable exists |
| `string`, `number`, `map` | Type checks |
| `starting_with(arg)`, `ending_with(arg)`, `containing(arg)`, `matching(regex)` | String checks |
| `before(other, [inclusive])`, `after(other, [inclusive])` | Date comparison |
| `dir`, `file`, `exists` | Path checks (mise custom) |
| `odd`, `even`, `divisible_by(n)` | Numeric checks |

**Template syntax:**
- `{{ }}` — Expressions · `{% %}` — Statements · `{# #}` — Comments · `{% raw %} {% endraw %}` — Skip rendering
- Operators: `+`, `-`, `/`, `*`, `%`, `==`, `!=`, `>=`, `<=`, `and`, `or`, `not`, `~` (concat), `in`

**Tera v1 → v2 migration:**

| v1 | v2 |
|----|-----|
| `value \| trim_start_matches(pat="v")` | `value \| trim_start(pat="v")` |
| `value \| trim_end_matches(pat="-beta")` | `value \| trim_end(pat="-beta")` |
| `items \| slice(start=0, end=2)` | `items[0:2]` |
| `[base] \| concat(with="file.txt")` | `[base, "file.txt"]` |
| `items \| map(attribute="name")` | `[item.name for item in items]` |
| `items \| filter(attribute="active")` | `[item for item in items if item.active]` |
| `value \| as_str` | `value \| str` |
| `value \| escape` | `value \| escape_html` |
| `value \| linebreaksbr` | `value \| newlines_to_br` |
| `value is divisibleby(divisor=3)` | `value is divisible_by(divisor=3)` |
| `value is object` | `value is map` |
| `value \| indent(prefix=">")` | `value \| indent(width=1)` (spaces only) |
| `value \| truncate` | `value \| truncate(length=255)` |

New v2 syntax: slices (`parts[0:2]`, `parts[-1]`, `name[::-1]`), spread (`[first, ...rest]`, `{...base, key: value}`), comprehensions, optional chaining (`env?.NODE_ENV or "development"`), ternaries (`"prod" if release else "dev"`). Undefined-variable access is **stricter** in v2, and **Tera v1 macros are unsupported**.

```toml
[settings]
tera_v1 = true    # escape hatch — deprecated on arrival; warns 2026.10.0, removed 2027.4.0
```

> In a shared `mise.toml`, prefer the env form `[env] MISE_TERA_V1 = true` — older mise versions treat it as a normal env var rather than erroring on an unknown setting.

**Shell-style variable expansion** — `env_shell_expand` **defaults to `true`** (verified on 2026.8.0):
```toml
[env]
LD_LIBRARY_PATH = "$MY_LIB:$LD_LIBRARY_PATH"
PATH_SAFE = "${VAR:-default}"      # With default
CLEAN = "${UNSET_VAR:-}"           # Empty if unset (no warning)
```

Supported forms: `$VAR`, `${VAR}`, `${VAR:-default}`, `${VAR:-}`. Expansion runs **after** Tera rendering, so both can be mixed. Undefined vars without a default are left unexpanded and produce a warning. Opt out with `env_shell_expand = false`.

---

## Hooks and Watchers

### Hook Types

| Hook | Trigger | Requires `mise activate`? |
|------|---------|--------------------------|
| `cd` | Any directory change (including within the project) | Yes |
| `enter` | Enter a project (once per entry; not re-fired for subdirs) | Yes |
| `leave` | Leave a project (once) | Yes |
| `preinstall` | Before tool installation | No |
| `postinstall` | After tool installation (**fires even on no-op installs**) | No |

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

# Task hooks — executed via `mise run`, so the full task system applies
[hooks]
enter = { task = "setup" }
enter = ["echo 'entering'", { task = "setup" }]  # Mixed syntax

# Array-of-tables form
[[hooks.cd]]
run = "echo 'I changed directories'"
[[hooks.cd]]
run = "echo 'I also changed directories'"
```

`run`/`run_windows` must be **strings** — arrays are unsupported. On Windows mise uses `run_windows` when set; on other platforms a hook with only `run_windows` is skipped.

`shell` means different things depending on context: on a `run` hook it's an **inline shell command** (include the eval arg: `"bash -c"`, `"pwsh -Command"`); on a `script`/`scripts` hook it's a **shell-name selector** (`bash`, `zsh`, `fish`) and mise only emits the script when the active `mise activate` shell matches.

**Important:** Shell hooks don't auto-cleanup on directory exit like `[env]` does. mise executes literal shell code without tracking it, so exported vars, aliases, and sourced files persist — reverse them manually in a corresponding `leave` hook.

> **Deprecation:** the *spawned* `script`/`scripts` table form is deprecated in favor of `run` — warns **2026.9.0**, removed **2027.3.0**. Current-shell `shell` + `script`/`scripts` for `cd`/`enter`/`leave` is unaffected. For `preinstall`/`postinstall`, `script`/`scripts` are legacy aliases for `run`, and a `shell` set alongside them is **ignored with a warning**.
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

Fields: `patterns` (required glob array), `run`, `task`, `shell` (applies to `run` only). Each entry requires **either** `run` or `task`, not both. Sets `MISE_WATCH_FILES_MODIFIED` (colon-separated, literal colons backslash-escaped). Requires watchexec (`mise use -g watchexec@latest`) and `mise activate`.

### Hook Environment Variables

All hooks receive:
- `MISE_ORIGINAL_CWD` — user's working directory at hook fire
- `MISE_PROJECT_ROOT` — detected project root

CD/enter/leave hooks additionally receive:
- `MISE_PREVIOUS_DIR` — previous directory (only when a directory change occurred)

Config-level `postinstall` receives:
- `MISE_INSTALLED_TOOLS` — JSON array, e.g. `[{"name":"node","version":"20.10.0"}]`

> A `mise install` that finds nothing to install **still runs `postinstall`**, with `MISE_INSTALLED_TOOLS` set to `[]`. Guard accordingly.

Tool-level `postinstall` additionally receives:
- `MISE_TOOL_NAME` — tool identifier (e.g., `node`)
- `MISE_TOOL_VERSION` — installed version
- `MISE_TOOL_INSTALL_PATH` — installation directory
- plus that tool's `install_env` values

### Per-Tool postinstall (not a hook)

Runs immediately after each tool is installed, before other tools in the same session:

```toml
[tools]
node = { version = "22", postinstall = "corepack enable" }
python = { version = "3.12", postinstall = "pip install pipx" }
```

---

## Sandboxing and Safe Mode

Two independent mechanisms: **sandboxing** restricts what a task/exec can touch; **safe mode** makes mise itself refuse to execute anything a config asks for.

### Sandboxing (`[settings.sandbox]`)

Any `--deny-*`/`--allow-*` flag implicitly enables sandboxing. Applies to `mise run` **and** `mise exec`.

```toml
[settings.sandbox]
deny_all = true      # or individually:
deny_read = true
deny_write = true
deny_net = true
deny_env = true
```

Env vars: `MISE_SANDBOX_DENY_ALL`, `MISE_SANDBOX_DENY_READ`, `MISE_SANDBOX_DENY_WRITE`, `MISE_SANDBOX_DENY_NET`, `MISE_SANDBOX_DENY_ENV`.

Per-task fields (flat, not nested) and the matching CLI flags compose with these:

```toml
[tasks.fetch-deps]
deny_all = true
allow_net = ["registry.npmjs.org"]
allow_read = ["{{config_root}}"]
allow_write = ["{{config_root}}/node_modules"]
allow_env = ["NODE_*", "npm_*"]
run = "npm ci"
```

```bash
mise run build --deny-net --allow-read /src
mise x --deny-all --allow-read=. --allow-write=./dist -- npm install
```

**Implicit access:** system paths stay readable (Linux `/usr /lib /bin /etc /dev /proc /sys /tmp /nix`; macOS `/System /Library /usr /bin /dev /etc /private /opt/homebrew /nix`) along with mise tool dirs. `/tmp` and `/dev` stay writable. `--allow-write` paths are implicitly readable. `--deny-env` still passes `PATH`, `HOME`, `USER`, `SHELL`, `TERM`, `LANG`; `--allow-env <VAR>` implies deny for everything else and supports wildcards.

**Platform matrix:**

| Feature | Linux | macOS | Windows |
|---------|-------|-------|---------|
| Deny/allow reads & writes | Landlock (kernel ≥5.13) | Seatbelt | ❌ |
| Deny all network | seccomp-bpf | Seatbelt | ❌ |
| Per-host network (`allow_net`) | **Not supported (v1)** — falls back to allowing all network | ✅ | ❌ |
| Env filtering | Built-in | Built-in | ❌ |

If Landlock is unavailable or cannot apply filesystem restrictions, the command **fails**. On Windows a warning is printed and the command runs unsandboxed. Sandboxing is **not** marked experimental.

### Safe Mode (`safe` / `MISE_SAFE=1`)

**Global-config-only** setting (mise 2026.7.12+) that turns mise into an inert config reader — useful for untrusted repos, fork PRs, and bots that only need to read or bump versions.

```bash
MISE_SAFE=1 mise lock --bump --dry-run --json
```

In safe mode mise **errors** (never silently falls back) on: template `exec()`/`read_file()`, `_.source`, hooks, tasks, asdf plugin scripts, and plugin installs. It **ignores** project `[env]`, `_.path`, `_.file`, `[shell_alias]`, and `[settings]`. Version resolution still works for HTTP-based backends and Go. Because the config cannot do anything, safe mode also **skips the trust requirement entirely**.

### Paranoid Mode (`paranoid` / `MISE_PARANOID=1`)

- Requires **all** config files trusted before loading, including formats normally exempt.
- **Re-verifies the hash** on modification → re-approval after every change.
- Community plugins must specify the full git repo (no shorthand names).
- Forces HTTPS on all endpoints.
- Always re-verifies provenance (SLSA, cosign, minisign, GitHub attestations) at install time.
- Disables cross-worktree trust propagation.

### Trust

Outside CI, untrusted configs must be approved with `mise trust` (`--all`, `--ignore`, `--show`, `--untrust`; plus the separate `mise untrust`).

- mise **auto-trusts** configs when it detects a CI environment — unless `MISE_PARANOID=1`.
- Since v2026.6.6, **safe** `mise.toml` files (no templates; only `min_version` and plain `[tools]`/`[tasks]` string values) auto-load without a trust prompt; anything with templates or richer constructs still requires trust.
- Since 2026.7.5, a config in a linked **git worktree** is auto-trusted if the equivalent path in the main checkout is trusted (one-way; `--ignore` still wins; excluded under paranoid mode). `mise trust --all` walks subdirectories, respecting `.gitignore` and skipping hidden dirs, `node_modules`, `vendor`, `target`, `dist`, `build`.
- When a monorepo root is trusted, **all descendant configs are automatically trusted.**
- Trust-sensitive keys (`ci`, `paranoid`, `trusted_config_paths`, `yes`) are ignored when set from project/local config — only global config, CLI flags, and env vars apply.

---

## Machine Bootstrap (Developer Setup)

`mise bootstrap` (alias **`bs`**) provisions an entire machine from declarative config. **Stable since v2026.7.4** (no longer requires `MISE_EXPERIMENTAL`). Its declared effect is **destructive**.

```bash
mise bootstrap                    # run the full setup
mise bootstrap -n                 # --dry-run: preview without changing anything
mise bootstrap -y                 # --yes: skip confirmation prompts
mise bootstrap --update           # refresh package-manager metadata (and repos) first
mise bootstrap --force-dotfiles   # overwrite conflicting whole-file dotfiles
mise bootstrap --prompt-secrets   # prompt for [bootstrap.secrets] inputs
mise bootstrap --only packages,dotfiles
mise bootstrap --skip macos-defaults
mise bootstrap plan [--json] [--detailed-exitcode]  # 0 = no changes, 2 = changes, 1 = failure
mise bootstrap status [--json] [--missing]          # non-zero exit when out of sync
mise bootstrap remote [TARGET]…   # apply config to inventory hosts / SSH destinations
```

`--only` / `--skip` are mutually exclusive, repeatable or comma-separated. Parts: `plugins`, `packages`, `accounts`, `files`, `services`, `firewall`, `compose`, `repos`, `dotfiles`, `mise-shell-activate` (alias `shell`), `macos-defaults` (alias `defaults`), `macos-launchd-agents` (alias `launchd`), `linux-systemd-units` (alias `systemd`), `user`, `tools`, `task`, `final-hook`.

**Execution order (17 steps):**

0. `[bootstrap.users]` / `[bootstrap.groups]` — Linux accounts
1. `[bootstrap.plugins]` — vfox plugins acting as package managers → then `[bootstrap.hooks.pre-packages]`
2. `[bootstrap.packages]` — built-in-manager entries
3. `[bootstrap.files]` / `[bootstrap.directories]` — privileged files and dirs
4. systemd **system** services (Linux)
5. `[bootstrap.linux.firewall]`
6. Docker Compose projects
7. `[bootstrap.repos]` — git checkouts
8. `[dotfiles]` — via `mise bootstrap dotfiles apply`
9. `[bootstrap.mise_shell_activate]` — shell rc activation snippets
10. macOS defaults
11. macOS LaunchAgents
12. Linux systemd user units
13. `[bootstrap.user]` — login shell
14. `[tools]` — `mise install`, then package-plugin entries, then `[bootstrap.hooks.post-packages]`
15. `mise run bootstrap` task (if defined)
16. `[bootstrap.hooks.final]`

Each phase is runnable on its own: `mise bootstrap packages apply`, `mise bootstrap macos defaults`, `mise bootstrap linux systemd-units`, `mise bootstrap repos update`, `mise bootstrap user apply`, etc. `mise bootstrap packages brew tap|untap` manages third-party Homebrew taps; `mise bootstrap packages import|prune|upgrade|use` syncs brew formulae; `mise bootstrap repos exec` runs a command across checkouts.

> System-package installs are gated by `[settings.system_packages]` (`sudo`, `managers`). The related `system_deps` setting (`prompt` default, `auto`, `warn`, `ignore`) controls how vfox `PLUGIN.systemDependencies` are surfaced — `prompt` falls back to `warn` non-interactively, and detection never fails an install.

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

# Secret inputs — stable logical names; fnox owns providers/auth
[bootstrap.secrets]
service_token = "EXAMPLE_SERVICE_TOKEN"

# Privileged files and directories
[bootstrap.files."/etc/example.conf"]
content = 'token={{ secret(name="service_token") }}'
template = true
owner = "root"
group = "root"
mode = "0644"
notify = ["example"]

# Linux firewall
[bootstrap.linux.firewall]
backend = "auto"
state = "enabled"
default_incoming = "deny"
default_outgoing = "allow"

[[bootstrap.linux.firewall.rules]]
name = "https"
port = 443
protocol = "tcp"
action = "allow"

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

Declarative steps converge idempotently; the `bootstrap` **task runs every invocation** (write it to be idempotent). Hooks stop bootstrap on failure and run in the current process environment. Unpinned `[bootstrap.repos]` entries are never pulled by a plain apply — use `mise bootstrap repos update` for the explicit fetch + fast-forward. Switching a systemd unit name between service and timer stops/disables/removes the stale sibling.

### Declarative Dotfiles (`[dotfiles]`)

Manage dotfiles declaratively; applied during `mise bootstrap` (step 8) or standalone via `mise bootstrap dotfiles apply`. **Stable since v2026.7.4.** Entries are keyed by target path.

> 🔴 **Renamed (2026.7.16):** the top-level `mise dotfiles` command is **deprecated and hidden from help**. Use **`mise bootstrap dotfiles`** — only that form runs the `pre-dotfiles`/`post-dotfiles` hooks correctly. The old command persists as a hidden compatibility alias: **warns from 2027.2.0, removed 2028.2.0**.

```toml
[settings]
dotfiles.root = "~/.dotfiles"      # default source root
dotfiles.default_mode = "symlink"  # symlink|symlink-each|copy|template

[dotfiles]
"~/.zshrc" = {}                                   # source mirrors target under dotfiles.root
"~/.gitconfig" = "dotfiles/gitconfig"             # string shorthand = source path
"~/.config/alacritty.toml" = { mode = "copy" }
"~/.ssh/config" = { source = "dotfiles/ssh_config.tmpl", mode = "template" }
"~/.local/bin" = { source = "dotfiles/bin", mode = "symlink-each", exclude = ["*.bak"] }
"~/.config/*.toml" = "dotfiles/config/*.toml"     # glob: * ** ? [ab] (target must match)
# Block / line edits to files mise does NOT own (key = target/edit-id)
"~/.zshrc/activate" = { block = 'eval "$(mise activate zsh)"' }
"/etc/hosts/dev" = { line = "127.0.0.1 dev.local" }
"~/.gitconfig/identity" = { source = "snippets/git-identity.tmpl", template = "tera" }
```

**Modes:** `symlink` (default; one link for a file or whole directory) · `symlink-each` (directory source → per-file links, so the target dir can also hold unmanaged files; supports `exclude` globs, prunes stale mise-managed links, and records exact source→target pairs in a manifest under `$MISE_STATE_DIR/dotfiles` rather than recursively walking shared targets like `~`) · `copy` (a real file/dir; additive for directories and **never pruned**) · `template` (render the source through mise's Tera engine with `env`, `vars`, `exec()`; permissions mirror the source and are repaired on drift).

**Source resolution:** omitting `source` mirrors the home-relative target path under `dotfiles.root` (`~/.zshrc` → `~/.dotfiles/.zshrc`). Relative explicit sources resolve against the declaring config file's directory. Targets outside `$HOME` require an explicit `source`.

**Block edits** wrap content in marker comments (`# >>> mise:id >>>` … `# <<< mise:id <<<`); the comment style is inferred per file type (`#`, `--`, `//`, `;`, `"`) and can be overridden with `comment`. Strict JSON/XML cannot use blocks. **Line edits** append a single line if absent and never modify other content. Edit IDs allow letters, digits, `_`, `-`, `.`. A table with `source` + `template = "tera"` is unambiguously an edit; a table with only `source` is a whole-file entry.

```bash
mise bootstrap dotfiles status [--missing] [--json]   # applied/missing/differs/source missing
mise bootstrap dotfiles apply [--dry-run] [--verbose] [--yes] [--force]
mise bootstrap dotfiles add ~/.zshrc [--no-apply]     # capture a live file (applies by default)
mise bootstrap dotfiles edit [--apply] ~/.zshrc
mise bootstrap dotfiles unapply                       # remove managed links/copies/templates/blocks
```

Dotfiles are **manual-only** — never applied implicitly by `mise install` or `mise bootstrap packages`. A regular file whose content already matches its source converges to a symlink without `--force`; a genuine conflict needs `--force`. `--dry-run` promises to execute nothing, so it **skips template rendering** and lists those entries as `(if changed)`. On Windows, file symlinks fall back to copies (directories use junctions). Removing a config entry leaves files in place — cleanup is manual (or use `unapply`).

---

## OCI Container Images

Build and publish container images containing mise-managed tools. **Experimental** — requires `experimental = true`. `oci` is a top-level config key.

```bash
mise oci build -o ./img                  # build an image (default output ./mise-oci)
mise oci build --copy ./dist:/app/dist   # reproducible host-path copy layer
mise oci build --from ubuntu:24.04 --tag myorg/dev:latest
mise oci build --include-global          # include ~/.config/mise tools (default is project-only)
mise oci build --no-mise                 # don't embed the mise binary
mise oci push myregistry.io/myimg:tag    # built-in registry client (no skopeo/crane)
mise oci push --cache-from myimg:prev    # reuse layers
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

Each tool version becomes its own content-addressable OCI layer, so bumping one tool invalidates only that layer. Output conforms to the OCI image-layout spec (consumable by `skopeo`, `crane`, `podman load`). Since v2026.7.12 the registry client is built in — `docker login` / `podman login` is the only setup needed. `mise oci build` also bakes `[dotfiles]` and `apt:` `[bootstrap.packages]` into images as dedicated, annotated layers.

**Limits:** asdf and vfox plugins are not supported in v1 — use core, aqua, github, cargo, npm, go, pipx, spm, or http backends. Build on the same OS/arch as the target image or pass `--no-mise`.

> `mise oci push --tool` was **removed** in 2026.7.12. Use `mise oci build -o ./img` + `skopeo copy` instead.

---

## Configuration and Settings

### File Hierarchy

Config files in per-directory precedence order (highest first):

1. `mise.local.toml` (gitignored)
2. `mise.toml`
3. `mise/config.toml`
4. `.mise/config.toml`
5. `.config/mise.toml`
6. `.config/mise/config.toml`
7. `.config/mise/conf.d/*.toml` (alphabetical)

Any can also appear as dotfiles (`.mise.toml`, etc.).

**Full stack, lowest → highest:**
```
/etc/mise/conf.d/*.toml, /etc/mise/config.toml, /etc/mise/config.<env>.toml
~/.config/mise/conf.d/*.toml, config.toml, config.<env>.toml, config.local.toml, config.<env>.local.toml
<ancestor dirs>/mise.toml …
<project>/mise.toml, mise.<env>.toml, mise.local.toml, mise.<env>.local.toml
<project>/<subdir>/mise.toml   ← highest
```

**Legacy:** `.tool-versions` (asdf-compatible)

**Schema validation:**
- `https://mise.jdx.dev/schema/mise.json`
- `https://mise.jdx.dev/schema/mise-task.json`

mise searches upward from cwd to root (stops at `MISE_CEILING_PATHS`). Merge behavior:
- **Tools:** Additive with overrides
- **Env vars:** Additive with overrides
- **Tasks:** Completely replaced per task name (closest wins)
- **Settings:** Additive with overrides

**Write targeting:** `mise use`, `mise set`, `mise unuse` write to the lowest-precedence file in the highest-precedence directory. With both present, writes go to `mise.toml`, not `mise.local.toml`.

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
node        sub-2:lts    # numeric subtraction from lts
python      sub-0.1:latest
```

### Idiomatic Version Files

Disabled by default. Enable per-tool:
```bash
mise settings add idiomatic_version_file_enable_tools python
mise settings add idiomatic_version_file_enable_tools go        # go.mod support
mise settings add idiomatic_version_file_enable_tools dagger task lefthook
```

Supported files include `.nvmrc`, `.node-version`, `package.json`, `.python-version`, `.python-versions`, `.ruby-version`, `Gemfile`, `.go-version`, `go.mod`, `rust-toolchain.toml`, `.java-version`, `.sdkmanrc`, `global.json`, `.terraform-version`, `.bun-version`, `.deno-version`.

- **`go.mod` (2026.7.13+):** a `toolchain goX.Y.Z` directive is an exact pin; a `go X.Y` minimum resolves to the latest matching patch.
- **Structured parsers (2026.7.15+):** registry `idiomatic_files` entries support `version_regex`, `version_json_path`, and `version_expr`, giving **in-process parsing with no plugin or shell execution** for 11 tools — Dagger, Task, chezmoi, CMake, Earthly, golangci-lint, GoReleaser, Lefthook, Pixi, pre-commit, Ruff (`dagger.json`, `Taskfile.yml`, `.chezmoiversion`, …).
- **Disable individual files per tool (2026.7.17+)** with `tool:filename` pairs — e.g. keep `.nvmrc` selecting Node while stopping node from reading `devEngines.runtime` in `package.json`, with pnpm still reading it:
  ```bash
  mise settings add idiomatic_version_file_disable_files node:package.json
  ```
- Since 2026.7.18, `idiomatic_version_file_enable_tools` set in a config root's own `[settings]` is honored by monorepo-wide commands (`mise ls --monorepo`, `mise install --monorepo`).

### Key Settings Reference

mise ships **272 settings**. This is a representative subset — run `mise settings --all` (or `mise settings ls`) for the complete list, and `mise settings set <key> <value>` / `mise settings get <key>` to manage them.

```toml
[settings]
# Execution
jobs = 8                    # Concurrent jobs (MISE_JOBS)
experimental = false        # Enable experimental features
yes = false                 # Auto-answer prompts (MISE_YES) — global only
safe = false                # Inert config-reader mode — global only

# Task defaults
task.output = "prefix"      # prefix|interleave|keep-order|replacing|timed|quiet|silent
task.timeout = "10m"        # Default task timeout
task.timings = true         # Show elapsed time
task.skip = ["slow-task"]   # Tasks to skip
task.skip_depends = false   # Skip dependencies
task.source_freshness_hash_contents = false  # blake3 content check
task.auto_infer = []        # experimental — e.g. ["node"]
task.cache_max_size = "2GiB"  # experimental
use_file_shell_for_executable_tasks = false  # Run file tasks through a shell

# Shells — ALL FOUR ARE GLOBAL-CONFIG-ONLY since 2026.7.14
unix_default_inline_shell_args = "sh -c -o errexit"
unix_default_file_shell_args = "sh"
windows_default_inline_shell_args = "cmd /c"
windows_default_file_shell_args = "cmd /c"
windows_powershell_no_profile = true   # -NoProfile for pwsh tasks (default true)

# Environment
env_shell_expand = true     # Shell-style expansion — DEFAULT TRUE
env_cache = false           # experimental — cache computed environment
env_cache_ttl = "1h"        # Cache TTL
env_file = ""               # MISE_ENV_FILE
auto_env = false            # Auto-load platform config files (default-on in 2027.6.0)

# Tool management
auto_install = true         # Auto-install missing tools
exec_auto_install = true    # Auto-install on mise x/run
not_found_auto_install = true
auto_install_disable_tools = []
disable_backends = ["asdf"] # Disable backends (new installs only)
disable_default_registry = false  # Only affects vfox and asdf shorthands
enable_tools = []           # Allowlist (unset = all; empty = none)
disable_tools = []
pin = false                 # Default --pin for mise use
lockfile = true             # Read/update lockfiles (unset behaves as enabled)
lockfile_platforms = []     # Extra platforms to resolve in the lockfile
locked = false              # Fail if no pre-resolved URLs
prereleases = false         # Allow pre-release versions for fuzzy requests
registry_floating = false   # Fetch current registry data instead of release-pinned snapshots
registry_cache_ttl = "1h"
shared_install_dirs = []    # Read-only dirs searched for installed versions
system_deps = "prompt"      # prompt|auto|warn|ignore — vfox systemDependencies handling

# Security
paranoid = false            # Extra-secure behavior — global only
gpg_verify = true           # Built-in OpenPGP verification (no external gpg binary)
slsa = true                 # SLSA provenance verification
github_attestations = true  # GitHub Artifact Attestations
provenance_api_failures_fatal = true  # Treat provenance API failures as install errors
netrc = true                # Honor ~/.netrc for HTTP auth (netrc_file overrides path)
minimum_release_age = "24h" # Filter fuzzy versions by release age
minimum_release_age_excludes = []  # Tools exempt from the release-age delay
locked_verify_provenance = false   # Re-verify at install (auto-on with paranoid)

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
http_timeout = "30s"                # Connect / between-reads timeout
http_download_timeout = "30m"       # Total download wall-clock including retries
http_retries = 3                    # HTTP retries with exponential backoff (0 = none)
cache_prune_age = "30d"             # Age before cached downloads are pruned
use_versions_host = true            # Use mise-versions shared version cache
offline = false                     # Block all HTTP requests
prefer_offline = false              # Prefer cached data

# UI / shell
color = true                # Colorized output
color_theme = "default"     # auto|default|charm|base16|catppuccin|dracula
terminal_progress = true    # OSC 9;4 terminal progress indicators
verbose = false             # Verbose install output
activate_aggressive = false # Push tool bin-paths to the front of PATH

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
npm_shim = true             # bash wrapper at bin/npm that reshims after `npm install -g`
flavor = ""                 # Alternate distribution flavor

# NPM backend
[settings.npm]
package_manager = "auto"    # auto|npm|aube|aube_cli|bun|pnpm
shell_out = false           # Route metadata + installs through the npm CLI

# Python-specific
[settings.python]
uv_venv_auto = false        # false | "source" | "create|source" | true (legacy form deprecated)
uv_venv_create_args = []
venv_create_args = []
compile = false             # Compile from source
venv_stdlib = false         # Prefer stdlib venv module
precompiled_flavor = "install_only_stripped"

# Ruby-specific
[settings.ruby]
compile = false             # PRECOMPILED IS NOW THE DEFAULT (2026.8.0) — set true to force source
ruby_install = false        # Use ruby-install instead of ruby-build
precompiled_url = "jdx/ruby"

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
binstall_quickinstall = false
# binstall_native = true    # graduated from experimental in 2026.7.16

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
rops = true                 # Use native Rust implementation (required for TOML)
strict = true               # Fail on decryption errors
age_key_file = "~/.config/mise/age.txt"

# Hook environment
[settings.hook_env]
cache_ttl = "0s"            # Cache hook-env dir checks (useful on NFS)
chpwd_only = false          # Only run on directory change, not every prompt

# OCI images
[settings.oci]
default_from = "debian:bookworm-slim"
default_mount_point = "/mise"
insecure_registries = []

# Dotfiles
[settings.dotfiles]
default_mode = "symlink"    # symlink|symlink-each|copy|template
root = "~/.dotfiles"

# System packages
[settings.system_packages]
sudo = true                 # set managers = [...] to pick package managers
```

**Nested namespaces (28):** `age`, `aqua`, `cargo`, `conda`, `dotfiles`, `dotnet`, `erlang`, `forgejo`, `github`, `gitlab`, `go`, `hook_env`, `java`, `node`, `npm`, `oci`, `pipx`, `python`, `ruby`, `rust`, `sandbox`, `sops`, `spm`, `status`, `swift`, `system_packages`, `task`, `zig`.

**Global-config-only settings (15)** — ignored when set from project config: `ci`, `forgejo.credential_command`, `github.credential_command`, `gitlab.credential_command`, `paranoid`, `safe`, `task.cache_remote_oidc_audience`, `task.cache_remote_token`, `task.cache_remote_token_file`, `trusted_config_paths`, `unix_default_file_shell_args`, `unix_default_inline_shell_args`, `windows_default_file_shell_args`, `windows_default_inline_shell_args`, `yes`.

> **`gpg_verify` behavior (2026.7.12+):** GPG verification **always runs** when enabled, with Node/Swift signatures verified in-process via rPGP (no external `gpg` binary). Previously a missing `gpg` silently skipped verification — you must now set `gpg_verify = false` explicitly to opt out.

**All settings** support environment variable overrides using the `MISE_` prefix (e.g., `MISE_JOBS=4`, `MISE_TASK_OUTPUT=interleave`).

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
- uses: actions/checkout@v7
- uses: jdx/mise-action@v4      # latest v4.2.4 (2026-08-01); v4 is the current major
  with:
    version: 2026.8.1     # mise version (default: latest)
    install: true         # run `mise install`
    install_args: "bun"   # extra args to `mise install`
    bootstrap: false      # run `mise bootstrap` instead of `mise install`
    bootstrap_skip: "tools,task"
    bootstrap_args: "--yes"
    cache: true           # cache via GitHub cache
    cache_key: "mise-v1-{{platform}}-{{install_args_hash}}-{{file_hash}}"
    experimental: false   # enable experimental features
    log_level: info
    working_directory: .
    reshim: false         # run `mise reshim -f`
    env: true             # export mise environment variables
    export_path: true     # add mise PATH entries to subsequent steps
    github_token: ${{ secrets.GITHUB_TOKEN }}
    # tool_versions: |    # optionally inline .tool-versions content
    # mise_toml: |        # optionally inline a mise.toml
```

> mise's own CI docs page still shows `@v3` — it is stale. **`@v4` is correct.**

Behavior worth knowing: PATH entries are added individually via `GITHUB_PATH` (the runner's complete PATH is not copied into `GITHUB_ENV`). When a `mise.lock` exists in the working directory or a parent, the action **automatically appends `--locked`** — unless you supply `mise_toml`/`tool_versions` inputs. `install_args` cannot be combined with `bootstrap: true`. Values flagged `redact = true` or matching `redactions` are masked automatically. Cache-key templating supports `{{version}}`, `{{platform}}`, `{{file_hash}}`, `{{mise_env}}`, `{{install_args_hash}}`, `{{bootstrap_hash}}`, `{{default}}`.

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

Or `mise generate bootstrap -l -w` produces a self-contained `./bin/mise` you can commit, so jobs don't re-download mise. `-l/--localize` sandboxes `MISE_DATA_DIR`/`MISE_CACHE_DIR` into a `.mise` directory in the project. The generated script honors `MISE_VERSION` and `MISE_INSTALL_PATH`.

**Untrusted config in CI:**
```yaml
script: |
  MISE_SAFE=1 mise lock --bump --json
```
Recommended whenever a job resolves tool versions from configuration it doesn't control — most commonly a bot refreshing `mise.lock` on PR branches.

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

- **VS Code:** extension `hverlin/mise-vscode` (tools, tasks, env, `mise.toml` completion). macOS doesn't read the login profile, so set an automation profile: `"terminal.integrated.automationProfile.osx": { "path": "/usr/bin/zsh", "args": ["--login"] }`. Or use `runtimeExecutable: "mise"` with `runtimeArgs: ["exec", "--", "node"]` in launch configs.
- **JetBrains:** plugin `intellij-mise` (tools + run-configuration env), or the asdf-compat symlink `ln -s ~/.local/share/mise ~/.asdf`.
- **Xcode:** script phases run under `sandbox-exec`, so add `$(SRCROOT)/mise.toml` to the build-phase **Input files**, then `eval "$($HOME/.local/bin/mise activate -C $SRCROOT bash --shims)"`. Xcode Cloud: do the install + activate in `ci_post_clone.sh`.
- **Emacs:** package `mise.el` — `(add-hook 'after-init-hook #'global-mise-mode)`; or add the shims dir to both `PATH` and `exec-path`.
- **Vim:** `let $PATH = $HOME . '/.local/share/mise/shims:' . $PATH`.

> Docs warn against `/bin/bash` on macOS ("decades old"; prefer zsh). On Linux the login profile is read at login, so logout/login is required after editing it.

### MCP Server

```bash
mise mcp    # JSON-RPC over stdin/stdout
```

```json
{"mcpServers": {"mise": {"command": "mise", "args": ["mcp"], "env": {}}}}
```

**Resources:** `mise://tools` (`?include_inactive=true`), `mise://tasks`, `mise://env`, `mise://config`.
**Tools:** `list_commands` (2026.7.16+ — the full command tree with each command's declared effect: `read`, `write`, `destructive`, or unclassified, plus help text and hidden status; `include_hidden` opts in), `run_task`, `install_tool` (**not yet implemented**).

> **Unclassified is explicitly "unknown", not "safe."** Consumers should treat a missing effect as "ask".

### Key Environment Variables

- `MISE_DATA_DIR` (default `~/.local/share/mise`)
- `MISE_CACHE_DIR` (default `~/.cache/mise`; macOS `~/Library/Caches/mise`; Windows `%TEMP%\mise`)
- `MISE_TMP_DIR` (default system temp)
- `MISE_SYSTEM_CONFIG_DIR` (default `/etc/mise`)
- `MISE_GLOBAL_CONFIG_FILE` (default `~/.config/mise/config.toml`)
- `MISE_GLOBAL_CONFIG_ROOT` (default `$HOME`; used as `{{config_root}}` for global config)
- `MISE_ENV_FILE` (e.g., `.env`)
- `MISE_${TOOL}_VERSION` (e.g., `MISE_NODE_VERSION=20`)
- `MISE_TRUSTED_CONFIG_PATHS` / `MISE_CEILING_PATHS` / `MISE_IGNORED_CONFIG_PATHS` (`:` Unix, `;` Windows)
- `MISE_OVERRIDE_CONFIG_FILENAMES` / `MISE_DEFAULT_CONFIG_FILENAME`
- `MISE_LOG_LEVEL` (trace|debug|info|warn|error), `MISE_LOG_FILE`, `MISE_LOG_FILE_LEVEL`, `MISE_LOG_HTTP`
- `MISE_LOG_VERBOSE_DEPS` — the only way to see h2/hyper/reqwest/rustls logs, even under `-vv`
- `MISE_QUIET` (= `MISE_LOG_LEVEL=warn`)
- `MISE_HTTP_TIMEOUT` (30s) / `MISE_HTTP_DOWNLOAD_TIMEOUT` (30m)
- `MISE_TERM_WIDTH` — terminal width override (takes precedence over `COLUMNS`; env-only, not a setting)
- `MISE_BASH_PATH` — bash used for `_.source` on Windows
- `MISE_FISH_AUTO_ACTIVATE` (default on; `0` disables)
- `MISE_RAW` (pipes directly; forces `MISE_JOBS=1`)

**Global CLI flags:** `-C/--cd <DIR>`, `-E/--env <ENV>`, `-j/--jobs <N>`, `-q/--quiet`, `-v/--verbose`, `-y/--yes`, `--raw`, `--locked`, `--silent`, `--no-config`, `--no-env`, `--no-hooks`, `--output <MODE>`.

---

## Dependency Preparation (`[deps]`)

**Experimental.** Ensures project dependencies (npm packages, Python venvs, Go modules, …) are installed before task execution. This replaces the old `[prepare]` section — **there is no `prepare` key in the schema.**

```bash
mise deps            # install project deps
mise deps --list     # show active/inactive providers with a reason
mise deps <provider> --explain
mise deps add <pkg>  # -D/--dev for dev dependencies
mise deps remove <pkg>
mise deps install --force
mise deps --monorepo                    # across monorepo config roots
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

Built-in providers activate only when **explicitly configured in `mise.toml` AND their lockfile exists**. `mise deps --list` reports inactive providers with the reason (e.g. `inactive (missing package-lock.json)`) instead of silently omitting them. Providers with `auto = true` run automatically before `mise x` and `mise run`; disable for one invocation with `--no-deps`.

In monorepos, provider IDs are qualified by config root (`//apps/api:uv`) so repeated provider names don't collide. Deps **task providers** are gated behind `experimental = true` (2026.7.12+).

---

## Monorepo Tasks and Workspace Graph

```toml
# Root mise.toml
monorepo_root = true

[monorepo]
config_roots = ["packages/frontend", "packages/backend", "services/*"]
lockfile = true    # true = single root lockfile; false = per-subproject; unset = current behavior
```

Stable since v2026.6.6 — no longer requires `MISE_EXPERIMENTAL=1`. Provides implicit trust for descendants, lazy task loading, and tool/env/vars inheritance from parent configs.

> `experimental_monorepo_root` is **deprecated and emits a warning**. Removal: **2027.12.0**. Use `monorepo_root`.

**Path syntax:**
```bash
mise //projects/frontend:build    # Absolute path from root
mise :build                       # Task in current config_root (leading : recommended)
mise '//projects/frontend:*'      # All tasks in frontend
mise //...:test                   # Test task in all projects
mise //projects/.../api:build     # ... matches any directory depth
mise '//...:test*'                # Wildcard task names across all projects
```

`...` matches directory depth (bazel/buck2 style); `*` matches task names. Path globs (`*`/`**` in the path portion) are **not** supported yet. mise never defines commands starting with `//` or `:`, so direct invocation is safe here.

**Listing & install:**
```bash
mise tasks                          # current config_root hierarchy
mise tasks --all                    # whole monorepo, including siblings
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

**Nested roots:** the **nearest** `monorepo_root = true` wins (e.g. git worktrees inside the main checkout). Tasks from the enclosing monorepo are not loaded, but the enclosing config remains an ancestor for tools, env vars, and vars.

> `[monorepo].lockfile` rollout: unset keeps per-subproject lockfiles today; **warns in 2026.12.0, defaults to root lockfiles in 2027.6.0**. Pin `lockfile = false` for mixed-version teams. Old subproject lockfiles auto-migrate into the root lockfile (root entries win).

### Workspace Project Graph (Experimental)

Requires `experimental = true` + `monorepo_root = true`. mise discovers projects and their internal dependency edges **without the underlying toolchain installed** — a project does not need its own `mise.toml` to appear in the graph.

| Ecosystem | Project ID | Discovery |
|-----------|-----------|-----------|
| **Cargo** | `cargo:<package>` | `[workspace]` members/exclude + root `[package]`; edges from local `path` deps across normal/dev/build/target tables, renamed deps, and `workspace = true` inheritance. No `cargo` binary. |
| **uv (Python)** | `uv:<package>` | `[tool.uv.workspace]` member globs/exclusions; edges only when `[tool.uv.sources]` selects a workspace member (`workspace = true`) or an in-repo `path`. Covers main, optional, `[dependency-groups]`, and legacy dev deps. No `uv`/Python invoked. |
| **Go** | `go:<module-path>` | `use` directives in `go.work` plus each `go.mod`. **Does not infer edges** from `require`/`replace` — declare them via overrides. |
| **Node** | `node:<pkg>` | npm/pnpm/Yarn/Bun via `pnpm-workspace.yaml` or the `workspaces` field (pnpm wins when both exist); edges from `dependencies`, `devDependencies`, `optionalDependencies`, `peerDependencies` (version strings are opaque — `workspace:*`, `catalog:`, ranges all yield the same edge). |

```bash
mise tasks graph              # inspect the graph
mise tasks graph --explain    # provider + metadata-source provenance per project/edge/task
mise tasks graph --json
```

Cycles are **reported**, not silently dropped. Config overrides are labeled `configuration` rather than misattributed to inference.

**Explicit overrides:**
```toml
[monorepo.projects."go:example.com/acme/api"]
depends_add = ["go:example.com/acme/lib"]   # depends replaces; depends_add/depends_remove adjust
```

**Node package scripts as tasks** (opt-in) — `package.json` scripts become first-class tasks with no per-package `mise.toml`:
```toml
[settings]
experimental = true
task.auto_infer = ["node"]
```
Task ID `node:@acme/web#build`, with the monorepo path `//apps/web:build` as an alias. Runs in the package directory through the detected workspace package manager with raw arg passthrough. An explicit mise task at that path takes precedence, and both names resolve to it. The Node provider also imports `inputs`, `outputs`, `cache`, and `dependsOn` from matching **`turbo.json`** entries (tracking `turbo.json` as a task-definition source); Turbo expressions mise can't preserve (e.g. `$TURBO_ROOT$`) are left unset. mise never guesses outputs or cacheability from a command string.

**Root task defaults** (applied by task name to inferred and explicit workspace tasks):
```toml
[monorepo.task_defaults.build]
sources = ["src/**", "package.json"]
outputs = ["dist/**"]
cache = { enabled = true }
depends = ["^build"]

[monorepo.task_defaults.test]
env = { NODE_ENV = "test" }
```

**Precedence** (high → low): the task's own fields → the `extends` template → `[monorepo.task_defaults.<name>]`. Map fields (`env`, `vars`, `tools`) merge across layers; collection fields (`depends`, `sources`, `outputs`) take the complete value from the highest-precedence layer that defines them — **not** concatenated.

**Upstream dependencies (`^`)** — runs the same task in every upstream project first, expanding transitively and skipping projects that lack it. Supported in **`depends` only**, not `depends_post` or `wait_for`.
```toml
depends = ["^build"]
```

**Relative (`./`) dependencies** resolve from the declaring task's own monorepo location, so one aggregate declaration works at root, in nested apps, and in leaves. A trailing `...` includes the base project itself as well as its descendants:
```toml
[tasks.test]
depends = [{ task = "./...:groups:tests:*", optional = true }]
```

### Affected Tasks (Experimental)

`mise run --affected <task>` selects only the projects owning changed paths, then follows reverse project dependencies so downstream projects are included. Workspace-global paths and `task_config.global_inputs` select the whole workspace.

```bash
mise run --affected test
mise run --affected --affected-base origin/main --affected-head HEAD test
mise run --affected --affected-explain --dry-run build
mise run --affected --affected-json build
```

Defaults are `HEAD~1` / `HEAD`; `MISE_AFFECTED_BASE` / `MISE_AFFECTED_HEAD` override them; **GitHub Actions and GitLab merge-request metadata provide CI defaults**; explicit CLI options win. Selection combines the workspace dependency graph, `global_inputs`, and provider lockfile diffs.

---

## Deprecation Calendar

| Removal | What | Migrate to |
|---------|------|------------|
| **2026.9.0** (warn) | Hook `script`/`scripts` *spawned* table form | `run` (removal 2027.3.0) |
| **2026.10.0** (warn) | `tera_v1` setting; Tera v1 compat helpers; `install_before` | Tera v2 syntax; `minimum_release_age` |
| **2026.11.0** (warn) | `credential_command` legacy single positional argument; `*.default_packages_file`; `dotnet.package_flags` | `MISE_CREDENTIAL_HOST`/`MISE_CREDENTIAL_PROVIDER`; tool-level `postinstall`; `prerelease` option |
| **2026.12.0** | `shorthands_file` · `env.mise.*` namespace · `value`/`values` keys in `_.file`/`_.path`/`_.source` | `[plugins]` · `env._.*` · `path` (string or array) |
| **2026.12.0** (warn) | `[monorepo].lockfile` unset behavior; `aqua.registry_url`; `auto_env` platform-config warning | Set them explicitly; `aqua.registries` |
| **2027.2.0** | Flat `task_*` settings (`task_output`, `task_timeout`, `task_skip`, …) — **warnings live since 2026.8.0** | Dotted `task.*` |
| **2027.2.0** (warn) | Top-level `mise dotfiles` command | `mise bootstrap dotfiles` (removal 2028.2.0) |
| **2027.3.0** | Hook `script`/`scripts` spawned form | `run` |
| **2027.4.0** | `tera_v1` / `MISE_TERA_V1`; Tera v1 helpers; top-level `env_file` / `dotenv` / `env_path` | Tera v2; `_.file` / `_.path` |
| **2027.5.0** | Tera task-arg functions `{{arg()}}`, `{{option()}}`, `{{flag()}}` in run scripts | `usage` spec + `$usage_*` |
| **~2027.7.0** | `[tools] python = { virtualenv = … }` tool option | `env._.python.venv` |
| **2027.10.0** | `install_before` | `minimum_release_age` |
| **2027.11.0** | `credential_command` positional arg; `*.default_packages_file`; `dotnet.package_flags` | see above |
| **2027.12.0** | `experimental_monorepo_root`; `aqua.registry_url` | `monorepo_root`; `aqua.registries` |
| **2028.2.0** | Top-level `mise dotfiles` | `mise bootstrap dotfiles` |

**Default flips ahead:** `auto_env` → `true` in **2027.6.0** (warns from 2026.12.0) · `[monorepo].lockfile` → root lockfiles in **2027.6.0** · `cargo.binstall_native` warns 2027.1.0 and defaults on **2027.7.0**.

**Recent default changes:** `minimum_release_age` unset → **`24h`** · `ruby.compile` now defaults to **precompiled binaries** (2026.8.0) · `windows_powershell_no_profile` → `true` (2026.7.13) · structured env-file values are **literal by default** again (2026.7.14) · `cargo.binstall_quickinstall` = `false`.

**Undated deprecations:** `[alias]` → `[tool_alias]` · `ubi` backend → `github` · `asdf_compat` (no longer supported) · `go.set_gopath` · `idiomatic_version_file` / `idiomatic_version_file_disable_tools` → `..._enable_tools` · `legacy_version_file*` → `idiomatic_version_file*` · `npm.bun` → `npm.package_manager` · `profile`/`MISE_PROFILE` → `MISE_ENV` · `python.uv_venv_auto = true` (the legacy `true` value only) · monorepo automatic filesystem discovery → explicit `config_roots`.

**Already removed (no deprecation period):** `vars.mise` namespace · non-string `postinstall` hooks · unknown table fields in hook definitions · `mise oci push --tool` · project-local `*_default_*_shell_args` (2026.7.14).

> **On the Tera task-arg removal date:** `/tasks/task-arguments.html` and `/tasks/task-configuration.html` say **2026.11.0** while `/tasks/toml-tasks.html` says **2027.5.0**. The **source is authoritative**: `src/task/task_script_parser.rs` calls `deprecated_at!(…, "2027.5.0", …)`. Either way, do not use them — opt out early with `task.disable_spec_from_run_scripts = true`.

---

## Best Practices

### DO ✅

- **Always use `usage` field** for task arguments
- Use `${var?}` for required args to fail early; test optional values with `[ -n "${usage_x:-}" ]` (absent ≠ `"false"`)
- Use `${usage_count_flag:-0}` for count flags — never put `default` on a `count` flag
- Set `description` for discoverability
- Use `sources`/`outputs` for freshness; add `cache = { enabled = true }` (experimental) to restore outputs and replay logs
- Use `outputs = []` for lint/test/typecheck tasks that produce no files
- Declare `pass_through_env` for tokens so credential rotation doesn't bust the cache
- Use `depends` for task ordering; structured `depends` to pass args/env; `optional = true` for globs that may match nothing
- Use `confirm` for destructive operations
- Use literal `choices` for stable enums, `complete` for dynamic/filesystem-derived values
- Group related tasks with namespaces (e.g., `test:unit`, `test:e2e`)
- Share task config via `[task_templates]` + `extends`
- Set a project-wide default shell with `task_config.shell` (project-local `*_default_*_shell_args` are ignored since 2026.7.14)
- Use the per-task `output` field for style; `quiet` only to silence mise's own chatter
- Use `mise.local.toml` for personal overrides (gitignored)
- Prefer aqua backend for security (cosign/SLSA/attestation verification, all native)
- Migrate from `ubi:` backend to `github:` (ubi deprecated), giving each binary its own `[tool_alias]`
- Use `additional_asset_patterns` when several archives compose **one** tool; `[tool_alias]` when they are **independent** tools
- Use `env._.file`/`env._.path` instead of the deprecated top-level `env_file`/`dotenv`/`env_path`
- Set `expand = true` on a structured env file only when you actually want shell expansion
- Redact sensitive values with `redact = true`; use fnox, SOPS, or direct-age for secrets
- Use templates for dynamic values instead of hardcoding paths
- Use shims in `.zprofile`/`.bash_profile` and PATH activation in `.zshrc`/`.bashrc`
- Use `[tool_alias]` (not deprecated `[alias]`)
- Pin tool versions with `mise.lock` + `locked = true` in CI; use `minimum_release_age` for supply-chain delay
- Use `mise lock --bump` to advance fuzzy selectors without touching `mise.toml`
- Use `jdx/mise-action@v4` in GitHub Actions — it handles masking and `--locked` automatically
- Use `MISE_SAFE=1` when reading configs you don't control (fork PRs, untrusted repos, lockfile bots)
- Sandbox risky tasks with `deny_*` + narrow `allow_*` lists
- Declare `[monorepo].config_roots` explicitly instead of relying on filesystem walking
- Use `mise run --affected` in monorepo CI to skip untouched projects
- Use `mise bootstrap` with `[bootstrap]`/`[dotfiles]` to onboard developers and provision fresh machines in one command
- Use `mise bootstrap dotfiles …` (not the deprecated top-level `mise dotfiles`)

### DON'T ❌

- Use shell positional parameters or `"$@"`-style expansion for arguments
- Use `$args` in PowerShell
- Use inline template functions `{{arg()}}`/`{{option()}}`/`{{flag()}}` in run scripts (deprecated)
- Use `choices env="VAR"` — it is feature-gated out of mise and hard-errors with `Invalid usage config`
- Rely on `double_dash="required"` for validation in mise — it parses but is unenforced until usage 5.0
- Use usage attributes that don't exist: `parse`, flag `alias` child, `config=`, `required_if`, `required_unless`, `overrides`, or the `config { file … }` block — they hard-error
- Put a root-level `mount` in a TOML `usage` field (file-task headers only)
- Rely on `usage_*` leaking into nested tasks (invocation-local since 2026.7.6 — pass via `env=` or structured `depends`)
- Assume `--quiet` changes output style (it no longer does — use `--output`)
- Put mise's own flags after the task name (`mise run --silent build`, not `mise run build --silent`)
- Forget to quote glob patterns in sources
- Set env vars in `env` that deps need (they don't inherit — use structured `depends` with `env`)
- Use `raw = true` unless interactive input is needed (forces single-threaded, bypasses redactions)
- Set `MISE_ENV` in `mise.toml` (it determines which files to load — use `.miserc.toml`)
- Set `auto_env` in `mise.toml` — it is read during early init and has no effect there
- Set `locked = true` in a project config expecting project scope — all settings are global in scope
- Manually add executables to shims directory (`mise reshim` deletes them)
- Use `MISE_RAW=1` without knowing it sets `MISE_JOBS=1`
- Install new `asdf:` or `vfox:` plugins when aqua/github alternatives exist
- Use `[prepare.*]` — it no longer exists; use `[deps.*]`
- Use `vars.mise` or `env.mise.*` — rejected / deprecated in favor of `vars._` and `env._`
- Expect `bin_path` to support bare `{{os}}`/`{{arch}}` — use the `os()` / `arch()` functions

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
pass_through_env = ["DEPLOY_TOKEN"]
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
#MISE tools.postgresql="16"
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
min_version = "2026.8.0"

[settings]
jobs = 8
task.output = "interleave"
task.timings = true
lockfile = true
minimum_release_age = "7d"

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
_.python.venv = { path = ".venv", create = true, uv_create_args = ["--seed"] }

[hooks]
enter = "echo 'Welcome to {{vars.project_name}}'"

[vars]
project_name = "myapp"

[task_config]
shell = "bash -c"

[task_config.input_groups]
lockfiles = ["package-lock.json", "uv.lock"]

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
sources = ["src/**/*.ts", "tsconfig.json", "@group:lockfiles"]
outputs = ["dist/**"]
run = "tsc --build"

[tasks.test]
description = "Run tests"
depends = ["build"]
sources = ["src/**/*.ts", "test/**/*.ts"]
outputs = []
run = "vitest run"
```
