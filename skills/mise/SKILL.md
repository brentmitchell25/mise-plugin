---
name: mise
description: Use when working with mise - creating/editing mise.toml or .mise.toml files, defining tasks with usage field arguments, managing dev tools and environments, configuring hooks, task caching, OpenTelemetry task tracing, sandboxing, project daemons, project diagnostics, command wrappers, shared remote config includes, machine bootstrap, dotfiles and dotfile groups, dotfiles history, agent skills, monorepo workspaces, or when user mentions mise configuration.
---

# Mise Comprehensive Skill

## Table of Contents

- [STRICT ENFORCEMENT: Usage Field Required](#-strict-enforcement-usage-field-required)
- [Overview](#overview)
- [Task Definition Methods](#task-definition-methods)
  - [TOML-Based Tasks](#toml-based-tasks-in-misetoml)
  - [File-Based Tasks](#file-based-tasks-executable-scripts)
  - [Configuring File Tasks from TOML](#configuring-file-tasks-from-toml)
  - [Task Grouping (Namespaces)](#task-grouping-namespaces)
  - [Remote Tasks](#remote-tasks)
- [Task Arguments - Usage Spec Reference](#task-arguments---usage-spec-reference)
  - [Positional Arguments (arg)](#positional-arguments-arg)
  - [Dynamic Choices (env= and run=)](#dynamic-choices-env-and-run)
  - [Flags (flag)](#flags-flag)
  - [Custom Completions (complete)](#custom-completions-complete)
  - [Tera Rendering of TOML usage Strings](#tera-rendering-of-toml-usage-strings)
  - [Accessing Arguments in Scripts](#accessing-arguments-in-scripts)
  - [File Task Headers](#file-task-headers)
  - [Subcommands (cmd block)](#subcommands-cmd-block)
  - [Command Effects (effect=)](#command-effects-effect)
  - [Spec-Level Metadata](#spec-level-metadata)
  - [Formerly Feature-Gated — Now Working](#formerly-feature-gated--now-working)
  - [Still Not Implemented](#still-not-implemented-do-not-use)
  - [Flag Groups, Flagsets, and Outputs (v6)](#flag-groups-flagsets-and-outputs-v6)
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
  - [Rust Compiler Action Cache — REMOVED](#rust-compiler-action-cache--removed-rust_cache-is-a-no-op)
- [Task Tracing with OpenTelemetry (Experimental)](#task-tracing-with-opentelemetry-experimental)
- [Dev Tools Management](#dev-tools-management)
  - [Backends Overview](#backends-overview)
  - [TOML Syntax for Tools](#toml-syntax-for-tools)
  - [Per-Tool Options](#per-tool-options)
  - [Backend-Specific Configuration](#backend-specific-configuration)
  - [Lazy Tools](#lazy-tools)
  - [Tool Stubs](#tool-stubs)
  - [Shims and Aliases](#shims-and-aliases)
  - [Shell Completion (per-directory tab-complete)](#shell-completion-per-directory-tab-complete)
  - [Packslip, Man Pages, and Agent Skills](#packslip-man-pages-and-agent-skills)
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
- [Project Daemons (`[daemons]`)](#project-daemons-daemons)
  - [Declaring Daemons](#declaring-daemons)
  - [Tasks That Require Daemons](#tasks-that-require-daemons)
  - [Groups, Namespaces, and Cross-Project Daemons](#groups-namespaces-and-cross-project-daemons)
  - [Shared Server Providers (`[daemon_providers]`)](#shared-server-providers-daemon_providers)
  - [Service Presets](#service-presets)
  - [Ports and Worktrees](#ports-and-worktrees)
- [Project Diagnostics (`[doctor]`)](#project-diagnostics-doctor)
- [Command Wrappers (`[wrappers]`)](#command-wrappers-wrappers)
- [Sandboxing and Safe Mode](#sandboxing-and-safe-mode)
- [Machine Bootstrap (Developer Setup)](#machine-bootstrap-developer-setup)
  - [`[bootstrap]` Configuration](#bootstrap-configuration)
  - [Remote Bootstrap over SSH (`[bootstrap.remote]`)](#remote-bootstrap-over-ssh-bootstrapremote)
  - [Declarative Dotfiles (`[dotfiles]`)](#declarative-dotfiles-dotfiles)
  - [Dotfile Groups (`[dotfile_groups]`)](#dotfile-groups-dotfile_groups)
  - [Dotfiles History (`mise dot` / `[history]`)](#dotfiles-history-mise-dot--history)
- [OCI Container Images](#oci-container-images)
- [Configuration and Settings](#configuration-and-settings)
  - [File Hierarchy](#file-hierarchy)
  - [Remote Config Includes (`include`)](#remote-config-includes-include)
  - [Idiomatic Version Files](#idiomatic-version-files)
  - [Key Settings Reference](#key-settings-reference)
  - [Minimum Version](#minimum-version)
  - [Automatic Environment Variables](#automatic-environment-variables)
  - [CI/CD Integration](#cicd-integration)
  - [IDE Integration](#ide-integration)
  - [MCP Server](#mcp-server)
  - [Key Environment Variables](#key-environment-variables)
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

> **Verified against mise 2026.10.3** (released 2026-10-05) and **usage 6.12.0** (the usage-lib version that mise bundles), using the live binary (`mise settings`, `mise --help`, `mise backends ls`, real `mise run` probes), the published JSON schema (`https://mise.jdx.dev/schema/mise.json`), the docs source at the `v2026.10.3` tag, mise's Rust source for deprecation dates, and the `jdx/mise` release notes. Where docs and release notes disagree, release notes win; where both disagree with the binary, the binary wins.

mise is an all-in-one developer environment tool that manages:

- **Dev tools** — install and manage language runtimes, CLIs, and build tools (20 backends)
- **Tasks** — project-specific commands with argument handling, dependencies, freshness checks, and output caching
- **Environments** — manage env vars, configuration environments, dotenv files, secrets, age/sops-encrypted values
- **Hooks** — run commands on directory changes, project enter/leave, tool install
- **Daemons** — long-lived project services (databases, brokers, dev servers) supervised by pitchfork, startable as a task prerequisite
- **Diagnostics** — project-defined `mise doctor project` checks for requirements tool versions can't express
- **Wrappers** — intercept a command name before it reaches the active toolset
- **Sandboxing** — restrict a task's filesystem/network/env access; `safe` mode for untrusted configs
- **Machine bootstrap** — provision a whole dev machine, locally or over SSH (system packages, users/groups, privileged files, systemd services, Docker Compose, firewall, git repos, dotfiles, macOS defaults, login shell) via `mise bootstrap`
- **Dotfiles history** — Git-backed checkpoints of tracked config files, with rollback, undo, and optional sharing
- **Shared config** — pull `[tools]`/`[env]`/`[hooks]` fragments from a git repo or OCI artifact with top-level `include`, and task catalogs with `task_config.includes`
- **Tracing** — export `mise run` traces and task logs over OpenTelemetry (experimental)
- **Agent skills** — version-matched `SKILL.md` bundles published by `packslip:` tools, linked into an agent's skills directory
- **Monorepos** — workspace project graph inference across Cargo/uv/Go/Node, affected-task selection
- **OCI images** — build/push container images containing mise-managed tools

**Key Features:**
- Parallel dependency building (concurrent by default, up to `jobs` setting)
- Last-modified and content-hash (blake3) freshness checking
- Task output artifact caching, local and remote (experimental)
- File watching (`mise watch` rebuilds on changes)
- Cross-platform argument handling via `usage` spec
- Hierarchical configuration with configuration-environment support
- 20 tool backends (aqua, packslip, github, gitlab, forgejo, npm, cargo, pypi, etc.)
- Security verification (cosign, SLSA, GitHub Attestations, minisign, packslip signer pinning) — all native, no external CLIs
- Secret management (fnox, sops, age encryption)

**Top-level `mise.toml` keys** (36; authoritative, from `https://mise.jdx.dev/schema/mise.json`):
`_`, `alias` (deprecated), `bootstrap`, `daemon_groups`, `daemon_providers`, `daemons`, `daemons_settings`, `deps`, `doctor`,
`dotenv` (deprecated), `dotfile_groups`, `dotfiles`, `env`, `env_file` (deprecated), `env_path` (deprecated),
`experimental_monorepo_root` (deprecated), `history`, `hooks`, `include`, `min_version`, `monorepo`, `monorepo_root`, `oci`,
`plugins`, `redactions`, `settings`, `shell_alias`, `task_config`, `task_templates`, `tasks`, `tool_alias`, `tool_config`,
`tools`, `vars`, `watch_files`, `wrappers`.

> **New since 2026.9.12:** `include` ([Remote Config Includes](#remote-config-includes-include)),
> `daemon_providers` ([Shared Server Providers](#shared-server-providers-daemon_providers)), and
> `dotfile_groups` ([Dotfile Groups](#dotfile-groups-dotfile_groups)). `experimental_monorepo_root` is not new — it is a
> deprecated alias of `monorepo_root` that the schema now types.

> There is **no `[prepare]` key** — that feature is `[deps]` (the CLI accepts `mise prepare` as an alias for `mise deps`). See [Dependency Preparation](#dependency-preparation-deps).

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

**Supported `#MISE` directives** (from mise's source; verified on 2026.10.3): `description`, `extends` (2026.9.11+), `alias`/`aliases`, `confirm`, `depends`, `depends_post`, `wait_for`, `daemons`, `env`, `dir`, `hide`, `raw`, `raw_args`, `interactive`, `sources`, `outputs`, `watch`, `cache`, `shell`, `quiet`, `silent`, `output`, `pass_through_env`, `tools`.
**Supported `#USAGE` directives:** `arg`, `flag`, `choices`, `complete`, `env`, and root-level `mount`.

> ⚠️ **`timeout`, `vars`, `run`, `run_windows`, `file`, and every `deny_*`/`allow_*` sandbox field are NOT accepted in `#MISE` headers.** mise prints `WARN unknown field(s) ["timeout", "vars"] in task file header, ignoring` and runs the task without them. The upstream docs claim every task property works in headers — they don't. Set these in a metadata-only TOML block instead (see [Configuring File Tasks from TOML](#configuring-file-tasks-from-toml)).

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

> Header lines are recognized by the regex `^(?:#|//|::)\s*(?:(USAGE|MISE)|\[(USAGE|MISE)\])(.*)$`. An executable file **without a shebang** still has its `#MISE`/`#USAGE` lines parsed (a UTF-8 BOM is stripped first). Blank comment lines continue the block; the **first non-comment line ends it** — later `#USAGE` lines are ignored. File-task `#USAGE` lines are **not** Tera-rendered (TOML `usage` strings are — see [Tera in TOML usage](#tera-rendering-of-toml-usage-strings)).

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

### Configuring File Tasks from TOML

🔴 **Breaking change in 2026.9.13.** A `[tasks.<name>]` block whose name matches a file task now either *configures* or *replaces* that script:

| TOML block | Effect |
|------------|--------|
| **No** `run` / `run_windows` / `file` | **Metadata overlay.** Description, env, depends, timeout, sandbox fields, etc. apply to `mise-tasks/<name>.*`; the script stays the command. |
| **Has** `run` / `run_windows` / `file` | **Replaces** the script. `[tasks.hello] run = "…"` makes `mise-tasks/hello.sh` disappear as a separate task. |
| `[tasks."hello.sh"]` | Targets only that one script (when `hello.sh` and `hello.js` share the stem `hello`). |

```toml
# mise-tasks/deploy is a file task; give it fields its #MISE header can't carry
[tasks.deploy]
description = "Deploy (overrides the header description)"
timeout = "10m"                      # verified: the file task is killed after 10m
env = { REGION = "us-east-1" }
```

- **Precedence:** a command replaces a script only if its block comes from the config whose `task_config.includes` selected the script dir, or a higher-precedence config. Lower-precedence `run` blocks are ignored (their metadata still applies).
- **Layered definitions:** a metadata-only `[tasks.check]` in `mise.local.toml` overlays `[tasks.check]` from `mise.toml` — so you can add `depends` or env locally without copying the command. Only blocks *above* the highest-precedence command definition contribute.
- **Windows pairs:** `build.sh` + `build.ps1` form one task `build` (Windows picks the native one); `[tasks."build.ps1"]` defines a separate task.

> Before 2026.9.13, `[tasks."hello.sh"] run = …` was ignored and `[tasks.hello] run = …` left `hello.sh` as a second task. Configs relying on that now behave differently.

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

Remote files cached in `$MISE_CACHE_DIR` (git in `$MISE_CACHE_DIR/remote-git-tasks-cache`, OCI catalogs in `$MISE_CACHE_DIR/remote-oci-tasks-cache`). Clear with `mise cache clear`. Disable cache with `MISE_TASK_REMOTE_NO_CACHE=true` or `--no-cache` (which, since 2026.9.16, also re-clones `git::` includes and refetches remote dependency tasks).

> Since 2026.7.11, remote HTTP and Git-backed task files have their `#MISE` headers parsed (`tools`, `description`, `hide`, inline TOML overrides), and git-backed files are made executable after clone and on cache hits.

Remote git includes in `[task_config]` (**experimental**):
```toml
[task_config]
includes = ["git::https://github.com/myorg/shared-tasks.git//tasks?ref=main"]
```

**OCI task catalogs (2026.9.18)** — publish a task directory as an OCI artifact and include it by `docker pull`-style reference (no experimental badge):
```toml
[task_config]
includes = [
  "oci::ghcr.io/myorg/shared-tasks:1.0.0",
  "oci::registry.example.com/platform/tasks@sha256:0f1e2d3c...",   # pin a digest for immutability
]
```
```bash
oras push ghcr.io/myorg/shared-tasks:1.0.0 build.toml scripts/deploy   # publish
```
- The artifact unpacks as a task directory (executable file tasks + `.toml` task files). Files starting with `#!` are made executable. Titled layers become files at that path; untitled tar/tar+gzip/tar+zstd layers extract at the root. Symlinks and device files are rejected.
- Every blob is digest-verified, but there is **no signature verification** — pin `@sha256:` for anything you don't control.
- Credentials come from `docker login` / `podman login`, else anonymous. Loopback registries use plain HTTP; others need `oci.insecure_registries`.
- Cached in `$MISE_CACHE_DIR/remote-oci-tasks-cache` **per reference — a moved tag is not re-pulled**. Refresh with a new tag/digest, by deleting the dir, or with `MISE_TASK_REMOTE_NO_CACHE=true`.
- Not supported: single-file `oci::` includes and a `//subpath` selector (the artifact is always a directory).

> `includes` entries are Tera-rendered (`config_root`, `env`, `vars`). The top-level [`include`](#remote-config-includes-include) key is a different feature and **cannot carry tasks** — share tasks through `task_config.includes`.

---

## Task Arguments - Usage Spec Reference

**REMINDER:** Always use the `usage` field. Never use `$1`, shell positional parameters, or other shell-native argument handling.

The `usage` field uses [KDL-inspired syntax](https://usage.jdx.dev/) to define arguments, flags, and completions.

> 🔴 **Version context (verified 2026-10-05).** mise 2026.10.3 bundles **usage-lib 6.12.0** (every usage crate in its `Cargo.lock` is pinned to 6.12.0; mise's own CLI spec declares `min_usage_version "6.11"`). In **2026.8.11** mise migrated its *own* CLI parser, help output, and shell completions from clap to usage-rs.
>
> **Nothing is compiled out any more.** Since **2026.8.13** mise depends on `usage-cli`, whose manifest enables usage-lib's `validation` and `unstable_choices_env` features; Cargo unifies features across the build, so mise's single usage-lib gets them. `validate=` and `choices env=` — previously hard errors in this skill — **work and are enforced** (see [Formerly Feature-Gated](#formerly-feature-gated--now-working)).
>
> **usage 6.11–6.12 change how specs are written:** `choices run="cmd"` (command-backed choices that validate), `complete` nested inside an `arg`/`flag`, `complete … delegate="tool"`, `$usage_cmd` for the chosen subcommand, `default_subcommand_on_empty`, and `logo`. All verified on mise 2026.10.3 below.
>
> 🔴 **TOML `usage` strings are Tera-rendered by mise before usage parses them.** Any `{{ … }}` meant for usage (e.g. `{{ words[PREV] }}` in `complete run=`) must be wrapped in `{% raw %}…{% endraw %}` — see [Tera Rendering of TOML usage Strings](#tera-rendering-of-toml-usage-strings). File-task `#USAGE` headers are not rendered.

### Positional Arguments (`arg`)

Every attribute is accepted both as a prop (`arg "<f>" key=value`) and as a child node (`arg "<f>" { key value }`), except `choices` which is child-node only.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| (name) | string | *required* | `"<name>"` = required, `"[name]"` = optional. `"<file>"`/`"<path>"` triggers file completion, `"<dir>"` triggers dir completion. |
| `help` | string | none | Short help text shown with `-h` |
| `long_help` / `help_long` | string | none | Extended help text shown with `--help` |
| `help_md` | string | none | Markdown-only help (docs generation) |
| `required` | boolean | `#true` for `<x>`, `#false` for `[x]` | **Forced to `#false` whenever `default` is set** |
| `default` | string (prop) / string-or-list (child) | none | Default value if not provided. `default=""` sets to empty string (different from unset). A multi-value default (`default { "a"; "b" }`) only takes effect on **variadic** args — on a single-value arg only the first value is used. |
| `env` | string | none | Environment variable that can provide this arg's value. Priority: CLI > env > default. |
| `var` | boolean | `#false` | Variadic mode (accept multiple values). Shorthand: `"<name>..."` |
| `var_min` | integer | none | Minimum values when variadic |
| `var_max` | integer | none | Maximum values when variadic |
| `choices` | child node | none | Restrict to enumerated set. `choices strict=#false "a" "b"` **suggests** without rejecting other values (v6). `choices env="VAR"` reads the set from an env var; `choices run="cmd"` from a command's output (6.12) — see [Dynamic Choices](#dynamic-choices-env-and-run). |
| `validate` / `validate_error` | string (expr) | none | Expression checked against `value` (e.g. `int(value) >= 1`, `value matches '^[a-z]+$'`); `validate_error` is the message. **Works in mise** (see [Formerly Feature-Gated](#formerly-feature-gated--now-working)). |
| `complete` | child node | none | **6.12.** Completion attached directly to this arg — `arg "<x>" { complete run="…" }`. Beats a same-named top-level `complete`. |
| `effect` | enum / child node | none | `read` \| `write` \| `destructive` — raises the command's effect |
| `double_dash` | enum | `optional` | `"required"`, `"optional"`, `"automatic"`, `"preserve"`. **Now enforced** under v6. |
| `hide` | boolean | `#false` | Exclude from help output |
| `display_order` | integer | none | **v6.** Explicit ordering in help; parse order is unchanged |
| `delimiter` | string | none | **v6.** Split one word into several values (`--tags a,b,c`). Variadic args only; applied *before* choices/validation. |
| `allow_negative_numbers` | boolean | `#false` | **v6.** Accept a leading-minus number as a value; `--force` stays flag-like |
| `value_terminator` | string | none | **v6.** Token that ends a variadic arg without being stored |
| `hide_default_value`, `hide_env`, `hide_env_values`, `hide_possible_values`, `hide_short_help`, `hide_long_help` | boolean | `#false` | **v6.** Presentation-only; defaults, env fallback, and validation stay active |
| `value_names` | child node | none | **v6.** Relabel values / declare fixed arity |
| `env_fallback` | child node | none | **v6.** Additional env vars in precedence order |
| `deprecated_env` | child node | none | **v6.** Compatibility env aliases, consulted last, reported as deprecated |
| `note` / `warning` | child node | none | **v6.** Semantic callouts in long help and generated Markdown |

**Shorthands:** `<f>` required · `[f]` optional · `<f>...` ⇒ `var=#true` · `<-- f>` / `[-- f]` ⇒ `double_dash="required"`.

**Relationship child nodes (v6, valid on `arg` and `flag`)** — bare selectors name positionals, dashed selectors name flags:
`conflicts`, `requires`, `requires_if`, `required_if`, `required_if_eq`, `required_if_eq_all`, `required_unless`, `required_unless_all`.

```
arg "[request]" { requires "--mode" "--scope" }
arg "[token]"   { required_if_eq "--mode" "remote" }
arg "[sum]"     { required_unless_all "--stdin" "--file" }
```

**Environment precedence:** CLI argv → `env` → `env_fallback` (left to right) → `deprecated_env` → `default`.

**Examples:**

```
arg "<file>" help="Input file to process"
arg "[output]" default="out.txt" help="Output file"
arg "<files>" var=#true var_min=1 help="One or more files"
arg "<files>..." help="Shorthand variadic syntax"
arg "<env>" choices "dev" "staging" "prod" help="Target environment"
arg "<args>..." double_dash="automatic" help="Pass-through arguments"
arg "<file>" env="MY_FILE" help="Input file (or set MY_FILE)"
arg "<modes>..." { default { "fast"; "safe" } }     # multi-value default — variadic args only
arg "<port>" validate="int(value) >= 1 && int(value) <= 65535" validate_error="must be a valid port"
```

#### Dynamic Choices (`env=` and `run=`)

Both forms **validate** (unlike `complete`, which only suggests) and both feed tab-completion. Verified on mise 2026.10.3:

```
arg "<env>" { choices env="DEPLOY_ENVS" }                    # DEPLOY_ENVS="dev,staging prod"
arg "<svc>" { choices "all" run="ls services" }              # literal values may sit beside run= (6.12)
flag "--svc <svc>" { choices run="printf 'app\ndb\n'" }      # also on a flag's value
```

| Situation | Result |
|-----------|--------|
| `choices env=` — value in the list | accepted; the var is split on **commas and whitespace** |
| `choices env=` — value not in the list | `Invalid choice for arg env: qa, expected one of dev, staging, prod` |
| `choices env=` — var unset | `Invalid choice for arg env: dev, no choices resolved from env DEPLOY_ENVS` |
| `choices run=` — value not in output | `Invalid choice for arg svc: nope, expected one of all, app, db` |
| `choices run=` — command fails | `Could not check arg svc: x against its choices: exited with code 3` (task fails) |
| `--help` | does **not** run the command; prints ``[possible values: output of `ls services`]`` |

> Prefer `choices run=` over `complete run=` whenever the set is derivable *and* invalid values should be rejected. Inside a TOML `usage` string, any `{{ }}` in the command must be `{% raw %}`-wrapped.

> ✅ **`double_dash="required"` is now enforced** (usage v6). A value offered before `--` errors with `Argument <args> can only be set after a '--' separator`. Specs that "happened to work" under the old unenforced behavior will now error — re-verified on mise 2026.10.3.

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
| `negate` | string | none | Negative form (e.g., `"--no-color"`). Sets env var to `false`. Without a `default`, an absent flag leaves the var **unset** — only `--no-x` yields `false`. |
| `effect` | enum | none | `read` \| `write` \| `destructive` |
| `allow_hyphen_values` | boolean | `#false` | Let a value-taking flag consume a following `-…` token as its value. Errors if the flag takes no value. |
| `deprecated` | bool \| string | none | `#true` → literal `"deprecated"`; a string is used as the message |
| `arg` | child node | none | Names the flag's value: `flag "--user" { arg "<user>" }` |
| `choices` | child node | none | Enumerated set. `choices strict=#false` suggests without rejecting. `env=` / `run=` forms work (see [Dynamic Choices](#dynamic-choices-env-and-run)). |
| `validate` / `validate_error` | string | none | Expression check on the flag's value — put it on the value's `arg` child: `flag "--port <p>" { arg "<p>" validate="…" }` |
| `complete` | child node | none | **6.12.** `flag "--out <path>" { complete type="dir" }` |
| `hide` | boolean | `#false` | Exclude from docs/completions |
| `alias` | child node | none | **v6 — now works.** Short/extra form: `flag "--user" { alias "-u" }`; supports `hide=#true` |
| `required_if` / `required_unless` | string | none | **v6 — now works and is enforced** |
| `required_if_eq` / `required_if_eq_all` / `required_unless_all` | child node | none | **v6.** Value-conditional requirements |
| `conflicts` / `requires` / `requires_if` | string \| child node | none | **v6.** Mutual exclusion / co-requirements |
| `overrides` | string | none | **v6.** Two flags override each other; last one wins |
| `exclusive` | boolean | `#false` | **v6.** Flag must be given on its own |
| `require_equals` | boolean | `#false` | **v6.** `--inspect=9229` accepted, `--inspect 9229` rejected |
| `default_missing` | string | none | **v6.** Value used for a bare `--color` vs an explicit `--color=never` |
| `value_optional` | boolean | `#false` | **v6.** Absent / bare / valued become three distinct states |
| `bool_value` | boolean | `#false` | **v6.** Allow explicit `--color=false` |
| `default_if` | child node | none | **v6.** Conditional default: `default_if "--json" "true"` |
| `delimiter` | string | none | **v6.** `--tags a,b,c` → three values |
| `allow_negative_numbers` | boolean | `#false` | **v6.** `--jobs -1` binds `-1` |
| `value_terminator` | string | none | **v6.** Token ending one variadic occurrence |
| `display_order` | integer | none | **v6.** Explicit order within its help section |
| `help_heading` | string | none | **v6.** Group this flag under a heading in help |
| `action` | enum | none | **v6.** `help` \| `help_short` \| `help_long` \| `help_all` \| `version`; pair with `builtin=#true` for parser-supplied flags |
| `env_fallback` / `deprecated_env` | child node | none | **v6.** Extra / legacy env sources |
| `note` / `warning` | child node | none | **v6.** Semantic callouts |
| `hide_default_value`, `hide_env`, `hide_env_values`, `hide_possible_values`, `hide_short_help`, `hide_long_help` | boolean | `#false` | **v6.** Presentation-only |

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

**Negate flags:** `flag "--color" negate="--no-color" default=#true` — `--no-color` sets `$usage_color` to `false`. Help renders these as `--color / --no-color`.

**v6 examples (all re-verified working in mise 2026.10.3):**

```
flag "--user" { alias "-u" }                      # short form as a child node
flag "--file <f>" required_unless="--stdin"       # enforced
flag "--dump" exclusive=#true                     # must be given alone
flag "--inspect <p>" require_equals=#true         # --inspect=9229 only
flag "--color <w>" default_missing="always"       # bare --color ⇒ "always"
flag "--tags <t>" var=#true delimiter=","         # --tags a,b,c ⇒ 3 values
flag "--backend <b>" { choices strict=#false "core" "git" }   # suggest, don't reject
```

### Custom Completions (`complete`)

Provide **dynamic tab-completion** for arguments. `complete` only *suggests*; to also *reject* bad values use [`choices run=`](#dynamic-choices-env-and-run).

```
arg "<svc>" { complete run="ls services" }        # 6.12: nested inside the arg (preferred in mise tasks)
flag "--out <path>" { complete type="dir" }       # 6.12: nested on a flag
complete "<arg_name>" run="<cmd, one value per line>"   # top-level, matched by arg name
complete "<arg_name>" type="dir"
complete "<arg_name>" delegate="terraform"       # 6.12: hand off to another tool's completion
```

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `run` | string (Tera) | none | Shell command; one candidate per line |
| `type` | string | none | Built-in completer: `path`/`file` (identical), `dir`, `path:toml,yaml` (extension filter), `command` (PATH executables), `hostname` (`/etc/hosts`), `none` (no file fallback) |
| `delegate` | string | none | **6.12.** Complete the remaining words with *that* command's own shell completion (zsh verified; bash/fish need the tool's completion installed, else fall back to files). Completion-only — the task runs normally. |
| `descriptions` | boolean | `#false` | Parse `value:description` output |

`run`, `type`, and `delegate` are **mutually exclusive** — combining them errors with `can set only one of run, type or delegate`.

**Key rules (verified on mise 2026.10.3):**
- **Nest `complete` inside the `arg` in mise tasks.** mise mounts every task as a `cmd` under `run`, so a *top-level* `complete "<name>"` is looked up after **mise's own root completers** — and loses to them for these names: `plugin`, `task`, `tool`, `dir`, `alias`, `setting`, `env_var`, `env_key`, `backend`, `bin_name`, `prefix`, `config_file`, `new_plugin`, `installed_tool`, `tool@version`, `installed_tool@version`. E.g. `arg "<plugin>"` + `complete "plugin" run=…` completes mise's plugin list, not yours. A nested `complete` (or a different arg name) avoids this.
- A top-level `complete` may now appear before or after its `arg`; the node name is lowercased and must match the arg name.
- **`usage_*` variables are NOT set during completion** — `${usage_service}` in a `complete run=` is empty. Read earlier words with `{{ words[PREV] }}` instead.
- When no completer is given, the arg's own name is used as the type: `arg "<file>"`, `<dir>`, `<path>`, `<command>`, `<hostname>` complete with no block.

**Resolution order per arg:** `choices` (literal/`env=`/`run=`) → nested `complete` → top-level named `complete` → arg name as built-in type → file fallback. (Older versions let the `<file>` name win over `choices`; it no longer does.)

> 🔴 **One broken spec breaks completion for EVERY task.** If any task in scope has an invalid `usage` (or uses `clause`, or a TOML `usage` with an un-escaped `{{ }}`), `mise run <TAB>` falls back to file names for all tasks. Running the other tasks still works. Run `mise tasks validate` (which fails on usage parse errors since 2026.9.15) to find it.

**Example — values derived from the filesystem, the second depending on the first:**

```toml
[tasks.deploy]
usage = '''
arg "<service>" help="Service name" {
  complete run="ls -d infrastructure/*/application 2>/dev/null | sed 's|infrastructure/||;s|/application||'"
}
arg "<environment>" help="Target environment" {
  complete run="ls infrastructure/{% raw %}{{ words[PREV] | shell_quote }}{% endraw %}/application/env/ 2>/dev/null | sed 's/.tfvars//'"
}
'''
run = 'terraform -chdir=infrastructure/${usage_service?}/application apply -var-file=env/${usage_environment?}.tfvars'
```

**With descriptions:**
```
arg "<plug>" { complete run="printf 'alpha:First one\nbeta:Second one\n'" descriptions=#true }
```
Output format: `value:description` per line — split on the **first unescaped colon** (a line *starting* with `:` is not split); escape a literal colon with `\:`; both sides are trimmed.

**Tera variables and filters in `run`** (usage's own Tera, evaluated at completion time):
- `words` — array of all prompt words including the one being typed; access via `words[index]`. In a mise task this is the **whole line** — `["mise", "run", "<task>", …]`.
- `CURRENT` — index of word being typed
- `PREV` — index of previous word (`CURRENT-1`). **Only defined when `CURRENT > 0`.**
- Filters `shell_quote` (one value) and `shell_join` (array → space-separated, each quoted) (6.12) — quote words before splicing them into the command
- ⚠️ `slice` is **not** available in usage's completion Tera — a `{{ words | slice(…) }}` template fails and the arg silently falls back to file completion. Index with `words[…]` instead.

```
# file-task #USAGE header (not mise-rendered) — write the template directly
#USAGE complete "controller" run="ls modules/{{ words[PREV] | shell_quote }}/controllers"
# TOML usage string — wrap it so mise leaves it for usage
arg "<b>" { complete run="printf '%s\n' {% raw %}{{ words | shell_join }}{% endraw %}" }
```

Execution: `sh -c` (`cmd /c` on Windows when `sh` is absent), stdin closed, `__USAGE` set to the usage version. On Windows `cmd` cannot run pipelines or builtins — keep `run` to a single command.

**`choices` vs `complete`:**

| Feature | `choices` (literal / `env=` / `run=`) | `complete` |
|---------|-----------|------------|
| Values | Literal, env var, or command output | Command output, built-in type, or delegated |
| Validation | **Rejects** invalid input | Tab-completion only |
| `--help` | Lists literal values; shows ``output of `cmd` `` for `run=` | Not shown |
| Best for | Any closed set (static or derivable) | Open-ended values, paths, cascading lookups |

#### Tera Rendering of TOML `usage` Strings

mise renders a TOML task's `usage` string with **its own Tera context** (`config_root`, `cwd`, `env`, `vars`, `tools`, `mise_bin`, `xdg_*` …) **before** usage parses it. Consequences, verified on 2026.10.3:

- `{{ words[PREV] }}` / `{{ usage }}` meant for usage fail at **task load** — `ERROR Failed to render task usage … Cannot index into an undefined value` — breaking `mise run <task>`, its `--help`, **and completion for every other task**.
- Fix: wrap usage-time templates in `{% raw %}…{% endraw %}`.
- Upside: mise values work in defaults — `arg "[dir]" default="{{ config_root }}/sub"`, `flag "--home <h>" default="{{ env.HOME }}"`.
- File-task `#USAGE` lines are **not** rendered by mise; write `{{ words[PREV] }}` directly there.

### Accessing Arguments in Scripts

**Environment variables** (prefixed with `usage_`):

| Definition | Environment Variable |
|------------|---------------------|
| `arg "<file>"` | `$usage_file` |
| `arg "[output]"` | `$usage_output` |
| `flag "-v --verbose"` | `$usage_verbose` |
| `flag "--dry-run"` | `$usage_dry_run` |
| `flag "-o --output <file>"` | `$usage_output` |
| `cmd "migrate" { … }` (chosen subcommand) | `$usage_cmd` → `migrate`; nested → `db migrate` (6.12) |

**Naming rules:** `usage_` prefix + snake_case of the long name (hyphens → underscores, lowercased — `--Mixed-Case` → `usage_mixed_case`). An arg with `value_names "START" "END"` exports `$usage_start` (named after the **first** value name), not `$usage_<argname>`.

**`$usage_cmd` (usage 6.12, verified):** holds the subcommand path actually chosen, with aliases canonicalised (`dep` → `deploy`). When a spec has `cmd` children but none was chosen, mise sets it **to an empty string** (usage's own docs say "unset"). Branch on it instead of re-parsing words:

```bash
case "${usage_cmd:-}" in
  migrate) ./db migrate "${usage_dir?}" ;;
  seed)    ./db seed ;;
  *)       echo "pick a subcommand: migrate | seed" >&2; exit 1 ;;
esac
```

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
| Boolean flag | `"true"` | **variable absent entirely** (or `"false"` with `default=#false`, or after `--no-x` for a `negate` flag) |
| Count flag | `"1"`, `"2"`, etc. | **absent** |
| Value flag/arg | the string value (an explicit `--opt ''` is SET-empty) | absent unless `default=` or `env=` supplied one |
| `value_optional` flag | bare `--bump` → SET-empty; `--bump=major` → `major` | **absent** |
| Variadic | shell-escaped space-separated (`'y z' w`) | **absent** |

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
default_subcommand "..." # target for an unmatched word (`task src` → `task run src`)
default_subcommand_on_empty #true  # 6.11: a BARE invocation also runs the default subcommand
logo #"""…ascii art…"""# style="cyan+bold"   # 6.11: shown on the root -h/--help page only
source_code_link_template "..."  # Tera template with {{path}} / {{cmd}}
example "code" header="..." help="..." lang="..."
include file="./other.usage.kdl"   # merge another spec (file= is required)
                                   # In FILE TASKS (2026.9.12+) the path may be relative to the
                                   # task file, or use $MISE_CONFIG_ROOT / $MISE_TASK_DIR /
                                   # $MISE_PROJECT_ROOT — so shared flagsets need no absolute path.
```

`repository` (a plain URL, like `Cargo.toml`'s) is distinct from `source_code_link_template` (a per-command deep link) — neither implies the other. `bin "x"` does **not** rename the usage line in mise — help always uses the task name.

**`default_subcommand` vs `default_subcommand_on_empty` (verified):** with only `default_subcommand "run"`, a bare `mise run t` runs the *root* (`$usage_cmd` empty) while `mise run t src` routes to `run`. Adding `default_subcommand_on_empty #true` makes the bare call run `run` too (its arg defaults apply). `--help` marks the default `(default)`.

**`config` block** — declares a CLI's settings for docs, JSON-schema, and SDK generation:

```
config {
  file "~/.config/mycli/config.toml" scope="global"
  file "mycli.toml" findup=#true
  prop "jobs" type="uint" default=0 help="Number of parallel jobs" {
    cli "--jobs" "-j"
    env "MYCLI_JOBS"
  }
}
```

`prop` keys are dotted paths (`prop "status.missing_tools"`; props do not nest) with a `type=` (scalars plus list/set/map/option/union forms), `default`, `help`, and child `cli` / `env` / `deprecated_env` / `source` bindings. `source "git" name="git config"` declares an external kind for docs. The older `data_type=` / `default_note` spellings still parse.

> **mise resolves nothing from a `config` block.** Verified: with `prop "jobs" … { env "MYCLI_JOBS" }`, `MYCLI_JOBS=5 mise run t` leaves the value unset. It is metadata only — the "CLI > env > config file > default" chain does not apply to tasks. Use `env=` on the `arg`/`flag` itself for env backing.

### Formerly Feature-Gated — Now Working

Earlier versions of this skill said `validate=` and `choices env=` were compiled out of mise and hard-errored. **That is no longer true** — both work and are enforced on mise 2026.10.3 (the fix traces to 2026.8.13, when mise began depending on `usage-cli`, whose manifest turns on usage-lib's `validation` and `unstable_choices_env` features):

| Syntax | Valid value | Invalid value |
|--------|-------------|---------------|
| `flag "--port <p>" { arg "<p>" validate="int(value) >= 1 && int(value) <= 65535" validate_error="port must be 1-65535" }` | `--port 8080` → runs | `--port 99999` → `Invalid value for port: 99999: port must be 1-65535`; `--port abc` → `validation expression failed: invalid operation: int(abc)` |
| `arg "<name>" validate="value matches '^[a-z]+$'" validate_error="lowercase only"` | `abc` → runs | `ABC` → `Invalid value for name: ABC: lowercase only` |
| `arg "<env>" { choices env="DEPLOY_ENVS" }` | in the list → runs | `Invalid choice for arg env: qa, expected one of dev, prod` |

A `validate` rule is skipped when the flag is absent. Prefer `validate=` / `choices` over hand-rolled checks in the script body — mise rejects bad input before any dependency runs.

### Still Not Implemented (do not use)

These appear in usage docs but hard-error (or silently do nothing) in mise 2026.10.3. Errors now carry a precise span, e.g. `unsupported arg key parse`:

| Syntax | Result in mise | Use instead |
|--------|----------------|-------------|
| `arg "<f>" parse="cmd {}"` | `unsupported arg key parse` | — (no equivalent) |
| `flag "--color" config="ui.color"` | `unsupported flag key config` | `env="..."` |
| `config_alias "a" "b"` | `unsupported spec key config_alias` | — |
| `cmd "x" { example "Header" "code" }` | `expected 1..=1 arguments, got 2` | `example "code" header="Header"` |
| root-level `mount` in a TOML `usage` field | **no longer errors, but is ignored at run time** — the task reports no arguments and extra args pass through raw; only completion runs the mount | File-task `#USAGE mount`, or a `cmd` block |
| `clause "x" separator=":::"` | values never reach the script; `:::` collides with mise's own task separator; **breaks completion for every task** | Separate tasks, or `arg "<args>..."` |
| `external_subcommand #true` | runs, but nothing is exported | `raw_args = true` |
| `multicall #true` | `unexpected word` | — |
| `dont_delimit_trailing_values #true` | ignored | — |
| `long_version "…"` | `-V` prints the version as `mise ERROR …` and exits 1 | `version "…"` |

> **Working v6 policies** (verified): `unknown_flags "error"`, `subcommand_negates_reqs #true`, `allow_missing_positional #true`, `args_override_self #false`, `arg_required_else_help #true` (bare call prints usage and exits 1), `default_subcommand_flags`/`default_subcommand_help`, `help_template` (`{% raw %}`-wrapped in TOML), `heading` + `help_heading`, `flatten_help`, and `deprecated` / `deprecated_warn_at` / `deprecated_remove_at` (rendered in `--help` only — mise prints **no runtime warning**). Rich `choice "always" { alias "yes" }` entries validate, but the value is **not normalised** (`ALWAYS`/`yes` reach the script as typed).

**Sigil args (usage 6.5, verified)** — a variadic positional whose items start with a sigil, ahead of the normal args:

```
arg "[tool]..." sigil="+" { choices "node@22" "node@24" "python@3.14" }
arg "<command>"
arg "[args]..."
```
`mise run t +node@22 +python@3.14 python x y` → `usage_tool="node@22 python@3.14"`, `usage_command=python`, `usage_args="x y"`; `+n<TAB>` completes with the sigil restored.

### Flag Groups, Flagsets, and Outputs (v6)

**`group`** — declare how a set of flags relate. Fully enforced in mise:

```toml
[tasks.fetch]
usage = '''
flag "--file <f>"
flag "--url <u>"
flag "--stdin"
group "input" "--file" "--url" "--stdin" required=#true
'''
```

| `required` | `multiple` | Meaning |
|-----------|-----------|---------|
| — | — | At most one |
| `#true` | — | Exactly one |
| — | `#true` | Nothing enforced |
| `#true` | `#true` | At least one |

Errors read `Missing one of the required flags in group input: --file, --url` and `Invalid flag '--url': cannot be used with --file in group input`.

**`flagset` + `use`** — a named set of flags any `cmd` can pull in. Declared at spec top level only; resolved while the spec is read, so nothing downstream sees a new concept:

```toml
usage = '''
flagset "output" {
  flag "-v --verbose"
  flag "--json"
}
cmd "build" { use "output" }
cmd "test"  { use "output" }
'''
```

**`output` / `exit_code` / `select`** — declare what a command writes and what its statuses mean. `output`/`exit_code` are metadata for docs and MCP consumers, **but `select` is enforced**: it turns the named flag into a choice over the declared outputs (`--format bogus` → `Invalid choice for option format: bogus, expected one of human, json`), and `--help` lists them as possible values. A boolean flag can select one output with `output "json" select="--json"`.

```
cmd "check" {
  output "human" default=#true help="Human-readable report"
  output "json" media_type="application/json" framing="json"
  select "--format"
  exit_code 0 "all checks passed"
  exit_code 1 "a check failed"
}
```

`framing` is `text` (default), `json` (one document), or `jsonl` (one per line).

> ⚠️ **Multi-placeholder fixed arity does not map to separate env vars in mise.** `arg "<start> <end>" var_min=2 var_max=2` parses, but both values land in `$usage_start` (`"1 2"`) and `$usage_end` is empty. Use two separate `arg` nodes when you need two variables.

---

## Task Configuration Reference

### All Task Fields

#### Core Execution

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `run` | `string \| string[] \| ({task: string, args?: string[], env?: object} \| {tasks: string[]} \| string)[]` | — | Command(s) to execute. The only required property. |
| `run_windows` | same as `run` | — | Windows-specific override |
| `file` | `string` | — | External script path (local, HTTP, or Git URL) |
| `shell` | `string` | `task_config.shell`, else `sh -o errexit -c` (Unix) / `cmd /c` (Windows) | Interpreter. TOML-tasks only. E.g., `"bash -c"`, `"node -e"`. Since 2026.9.3, *simple* inline commands on Unix (`node build.js`) run **directly without `sh`** unless a `shell` is set explicitly, or the command uses shell syntax, builtins, or sandboxing. |
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
| `depends` | `string \| (string \| string[] \| {task, args?, env?, optional?})[]` | — | Tasks to run BEFORE. An inner `string[]` is `[task, ...args]`. Parallel by default. |
| `depends_post` | same as `depends` | — | Tasks to run AFTER this task. Runs if the parent **started** (even if it failed), but is **skipped when a regular dependency fails** before the parent starts. |
| `wait_for` | same as `depends` | — | Wait for tasks without adding them as deps. A `wait_for` naming a **nonexistent task errors** unless `optional = true`. |

#### Environment & Tools

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `env` | `table` | — | Task-specific env vars (**NOT passed to depends**). Supports sops/age-encrypted values and `_.file`. |
| `pass_through_env` | `string[]` | — | **Experimental.** Ambient env vars (wildcards allowed) passed through **without** affecting the cache key (tokens, CI vars). Only matters under env sandboxing (`deny_env`/`deny_all`/`allow_env`) — without it every ambient var passes anyway. |
| `tools` | `table` | — | Tools to install before running |
| `vars` | `table` | — | Task-local vars that override `[vars]` |
| `daemons` | `bool \| string \| string[]` | — | **Experimental (2026.9.12+).** Project daemons that must be running and ready before the task body runs. `true` = every daemon in this project's `[daemons]`. See [Tasks That Require Daemons](#tasks-that-require-daemons). |

#### Execution Context

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dir` | `string` | `{{config_root}}` | Working directory. Use `{{cwd}}` for user's cwd. |
| `raw` | `bool` | `false` | Direct stdin/stdout connection. Takes an **exclusive lock per command** (nothing else runs alongside each of its commands; other tasks can run between them). Does **not** set jobs=1 — that is `mise run --raw`. Bypasses redactions and the artifact cache; not stopped by the whole-run timeout. |
| `raw_args` | `bool` | `false` | Pass all args verbatim including `--help`/`-h` to underlying command |
| `interactive` | `bool` | `false` | Exclusive lock held for the **whole task** — every other task waits until it ends (verified; the schema description saying others "still run in parallel" is wrong). Bypasses redactions and the artifact cache. |

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
| `rust_cache` | `bool \| {enabled: bool}` | — | 🔴 **Deprecated NO-OP** (removal 2027.8.14). Does nothing. Use [mbx](#rust-compiler-action-cache--removed-rust_cache-is-a-no-op). |
| `watch` | `{no_vcs_ignore: bool}` | — | `no_vcs_ignore = true` watches sources excluded by `.gitignore` (2026.8.0+) |

#### Safety & Sandboxing

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `confirm` | `string \| {message, default?, yes?, no?}` | — | Prompt before running. `default` (optional since 2026.10.3, defaults to yes) accepts `yes`/`no`/`y`/`n`/`true`/`false`; `yes`/`no` are display labels (2026.10.3). All support Tera with `usage.*`. **Guards only `run`, not `depends`.** `-y/--yes` skips it; with **no TTY at all** (CI, agent shells) it fails: `task requires confirmation but there was nobody to ask; pass --yes to accept`. |
| `deny_all` | `bool` | `false` | Block reads, writes, network, and env inheritance |
| `deny_read` / `deny_write` / `deny_net` / `deny_env` | `bool` | `false` | Individual sandbox blocks |
| `allow_read` / `allow_write` | `string[]` | — | Path allowlists |
| `allow_net` | `string[]` | — | Host allowlist |
| `allow_env` | `string[]` | — | Env var allowlist (wildcards: `NODE_*`) |

These are **flat top-level task fields**, not a nested `sandbox` table.

> 🔴 **`redactions` is NOT a task field.** `[tasks.x] redactions = [...]` is a hard parse error (`unknown field 'redactions'`). It is a **top-level** key only — see [Redactions](#redactions-experimental).

#### Timeout & Inheritance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `timeout` | `string` | — | Max execution duration (e.g., `"30s"`, `"5m"`). **Shorter of per-task and global `task.timeout` wins**; `--timeout` overrides. Tera supported. On expiry: SIGTERM, then SIGKILL after 5s (Windows: Ctrl+C, 5s grace, then tree kill). A timed-out task **fails even if it exits 0** (2026.10.2). **Not accepted in `#MISE` headers** — set it in a TOML block. |
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

> Prefer this structured form for templated args. The string form `depends = ["build {{usage.app}}"]` runs fine, but `mise tasks validate` (2026.10.3) reports it as `missing-dependency` because it checks the unrendered name — a false failure if you validate in CI.

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
| **Deep merge** (per key; task wins) | `tools`, `env`, `vars` |
| **Concatenated** | `usage` — **template's flags first, then the task's** (2026.9.11+) |
| **Compose / combine** | sandbox `deny_*` compose; `allow_*` combine |
| **Not allowed on templates** | `hide`, `interactive`, `quiet`, `raw`, `raw_args` — set these per task |

> `depends` **replaces** rather than merges: template `["lint","typecheck"]` + task `["build"]` = `["build"]`.

> ⚠️ **An empty local list does NOT cancel an inherited one.** `depends = []` (likewise `run`, `run_windows`, `depends_post`, `wait_for`, `sources`) on the task still inherits the template's value. To genuinely clear it, extend a different template that has none. Exceptions: `outputs = []` *is* an explicit declaration (result-only caching), and `cache = { enabled = false }` explicitly disables inherited caching.

**`usage` now merges (changed 2026.9.11).** A task that `extends` a template *and* declares its own `usage` gets the template's flags **too**, listed first in `--help`. Previously the task's spec replaced the template's entirely, so shared flags had to be copied into every task.

```toml
[task_templates.deploy]
usage = 'flag "--env <env>" help="Target environment"'

[tasks."deploy:api"]
extends = "deploy"
usage = 'flag "--canary" help="Canary rollout"'   # --env AND --canary both available
run = "./deploy.sh"
```

> A flag declared in **both** places is listed **twice** — declare each flag in exactly one of the two. Workspace-root `[monorepo.task_defaults]` still only fills in `usage` when the task has none.

**File tasks can extend too (2026.9.11+):**
```bash
#!/usr/bin/env bash
#MISE extends="python:build"
```
For a **file task the template's `run` is ignored** (the script itself is the command), but `extends` still inherits tools, env, vars, args, and the rest. A template's `vars` can also read values the extending task supplies.

Templates are Tera-rendered in the **consuming** project's context (`{{config_root}}`, `{{env.VAR}}`, `{{cwd}}`, `{{vars.*}}`). Tasks loaded via `task_config.includes` now receive the collected `task_templates` map.

### Global Task Configuration

#### [task_config] Section

```toml
[task_config]
dir = "{{cwd}}"      # Default working directory for all tasks in this file
shell = "bash -c"    # Project-scoped default shell for tasks (2026.7.15+)
cascade = false      # Cascade dir/shell/cache/inputs/includes to descendant config roots
includes = [
  "tasks/*.toml",                              # Local task files
  ".mise/tasks/",                              # Task directory
  "git::https://github.com/org/tasks?ref=v1",  # Remote git tasks (experimental)
  "oci::ghcr.io/org/tasks:1.0.0",              # OCI task catalog (2026.9.18)
]
```

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `dir` | string | — | Default dir for all tasks in scope |
| `shell` | string | — | Project-scoped default shell; task-level `shell` wins |
| `cascade` | bool | `false` | Cascade `dir`, `shell`, `cache`, `global_inputs`, `input_groups`, and `includes` to descendant config roots. A descendant's non-empty `global_inputs` replaces; `input_groups` merge by name (nearest wins); a descendant `cascade = false` stops inheriting. |
| `includes` | string[] | the five default task dirs | Local paths, `git::` (experimental), or `oci::` refs; Tera-rendered. **Last entry wins** on name collision |
| `excludes` | string[] | `[]` | Config-root-relative paths or globs excluded from **file-task discovery**. Task dirs are searched recursively and every non-mise `.toml` in them loads as a task file — exclude `pyproject.toml`/`Cargo.toml` copies this way. The closest config defining `excludes` replaces inherited ones; `excludes = []` clears them. |
| `cache` | table | — | **Experimental**; inherits only to tasks with sources+outputs |
| `rust_cache` | bool \| table | — | 🔴 **Deprecated NO-OP** (removal 2027.8.14). Does nothing. |
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
| `task.output` | string | unset (→ `prefix` if jobs>1, else `interleave`) | `MISE_TASK_OUTPUT` | prefix, interleave, keep-order, replacing, timed, quiet (deprecated), silent |
| `task.quiet` | bool | `false` | `MISE_TASK_QUIET` | Suppress mise's own task messages/prefix headers without hiding task output (replaces `output = "quiet"`) |
| `raw` | bool | unset | `MISE_RAW` | Global `--raw` (forces jobs=1, bypasses redactions) |
| `task.timeout` | duration | unset | `MISE_TASK_TIMEOUT` | Default timeout. Per-task cannot exceed this. |
| `task.timings` | bool | unset | `MISE_TASK_TIMINGS` | Show elapsed time per task (shown by default with `prefix` output) |
| `task.skip` | string[] | `[]` | `MISE_TASK_SKIP` | Tasks to skip by default |
| `task.skip_depends` | bool | unset | `MISE_TASK_SKIP_DEPENDS` | Skip dependencies |
| `task.run_auto_install` | bool | `true` | `MISE_TASK_RUN_AUTO_INSTALL` | Auto-install missing tools |
| `task.show_full_cmd` | bool | unset | `MISE_TASK_SHOW_FULL_CMD` | Disable command truncation in output |
| `task.disable_paths` | string[] | `[]` | `MISE_TASK_DISABLE_PATHS` | Paths to exclude from task discovery |
| `task.remote_no_cache` | bool | unset | `MISE_TASK_REMOTE_NO_CACHE` | Always fetch latest remote tasks |
| `task.source_freshness_hash_contents` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_HASH_CONTENTS` | Use blake3 content hashing instead of mtime |
| `task.source_freshness_equal_mtime_is_fresh` | bool | `false` | `MISE_TASK_SOURCE_FRESHNESS_EQUAL_MTIME_IS_FRESH` | Equal mtime = fresh |
| `task.disable_spec_from_run_scripts` | bool | `false` | `MISE_TASK_DISABLE_SPEC_FROM_RUN_SCRIPTS` | Derive the usage spec only from the `usage` field (early opt-out before Tera arg functions are removed) |
| `task.auto_infer` | string[] | `[]` | `MISE_TASK_AUTO_INFER` | **Experimental.** Ecosystems whose scripts become tasks (e.g. `["node"]`) |
| `task.cache_dir` | path | `$MISE_CACHE_DIR/task-artifacts` | `MISE_TASK_CACHE_DIR` | **Experimental.** Task artifact cache location |
| `task.cache_max_size` | string | unset | `MISE_TASK_CACHE_MAX_SIZE` | **Experimental.** SI/IEC units (`500MB`, `2GiB`); LRU eviction |
| `task.cache_max_age` | duration | unset | `MISE_TASK_CACHE_MAX_AGE` | **Experimental.** `0s`/unset disables |
| `task.cache.remote_*` | — | — | — | **Experimental.** See [Remote Task Cache](#remote-task-cache). Flat `task.cache_remote_*` is deprecated. |
| `task.cache.audit_report` | path | unset | `MISE_TASK_CACHE_AUDIT_REPORT` | **Experimental.** JSON Lines audit of undeclared reads/writes |
| `task.cache.stats_report` | path | unset | `MISE_TASK_CACHE_STATS_REPORT` | **Experimental.** Action-cache session statistics as JSON |
| `task.monorepo_depth` | int | `5` | `MISE_TASK_MONOREPO_DEPTH` | Subdirectory depth for monorepo discovery |
| `task.monorepo_exclude_dirs` | string[] | `[]` | `MISE_TASK_MONOREPO_EXCLUDE_DIRS` | Empty ⇒ built-ins (`node_modules`, `target`, `dist`, `build`); any custom value **replaces** them |
| `task.monorepo_respect_gitignore` | bool | `true` | `MISE_TASK_MONOREPO_RESPECT_GITIGNORE` | Honor `.gitignore` in monorepo discovery |
| `jobs` | int | `8` | `MISE_JOBS` | Max concurrent task execution |
| `otel.enabled` | bool | `false` | `MISE_OTEL_ENABLED` | **Experimental (2026.9.13).** Export `mise run` traces — see [OpenTelemetry](#task-tracing-with-opentelemetry-experimental) |
| `otel.logs` | bool | `false` | `MISE_OTEL_LOGS` | **Experimental.** Also export task stdout/stderr as OTLP log records |

> **Deprecated flat aliases** (`task_output`, `task_timeout`, `task_timings`, `task_skip`, `task_skip_depends`, `task_disable_paths`, `task_remote_no_cache`, `task_run_auto_install`, `task_show_full_cmd`) **began warning in 2026.8.0** and are **removed in 2027.2.0**. Use the dotted `task.*` forms.
>
> **`jobs` default is 8.** The old 8-vs-4 discrepancy is gone: on 2026.8.12 both `mise settings get jobs` and the global `-j/--jobs` help report **8**, and `mise run --help` no longer documents a separate default. Values below 1 are treated as 1.

---

## Running Tasks (CLI)

### Commands

| Command | Description |
|---------|-------------|
| `mise run <task>` / `mise r <task>` | Execute task |
| `mise <task>` | Shorthand (discouraged in scripts — future mise versions may add conflicting commands) |
| `mise run` (no args) | Runs the `default` task; with no `default`, opens the interactive task selector (TTY only) |
| `mise tasks` / `mise tasks ls` | List all tasks (`-J/--json`, `-x/--extended`, `--no-header`) |
| `mise tasks --hidden` | Include hidden tasks |
| `mise tasks --global` / `--local` | Filter by config scope |
| `mise tasks --all` | Entire monorepo, including siblings |
| `mise tasks --name-only` | One name per line (fzf-friendly) |
| `mise tasks --sort <name\|alias\|description\|source>` | Sort order (`--sort-order asc\|desc`) |
| `mise tasks deps` | Show dependency tree |
| `mise tasks deps --compact` | Expand each shared subtree once, mark repeats `(already shown)` (2026.7.18+) |
| `mise tasks deps --dot` | DOT format for Graphviz |
| `mise tasks graph [--explain] [--json]` | **Experimental.** Inspect the workspace project graph |
| `mise tasks info <task> [-J]` | Show task details (JSON output includes `config_sources`) |
| `mise tasks add <name> -- <cmd>` | Create task via CLI: `-a/--alias`, `-d/--depends`, `--depends-post`, `-w/--wait-for`, `-D/--dir`, `-f/--file` (file task), `-H/--hide`, `-q/--quiet`, `--silent`, `-r/--raw`, `-s/--sources`, `--outputs`, `--description`, `--run-windows`, `--shell` |
| `mise tasks edit <task> [-p]` | Edit/create task in `$EDITOR` (respects configured `includes`; `-p` prints the path) |
| `mise tasks validate [--errors-only] [--json]` | Validate task definitions. Since **2026.9.15** an unparseable `usage` spec is an **error** (`usage-parse-error`, exit 1) — CI that ran it may start failing |
| `mise watch <task>` / `mise w <task>` | Watch and re-run on file changes (requires watchexec) |
| `mise run a ::: b ::: c` | Run multiple tasks in parallel (with separate args) |

### Execution Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--jobs <N>` | `-j` | Parallel job limit (default 8; values below 1 treated as 1) |
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
| `--no-timings` | — | Hide per-task elapsed time; overrides `MISE_TASK_TIMINGS=1` (`--timings` still works but is hidden from help) |
| `--timeout DURATION` | — | Task timeout (e.g., `30s`, `5m`) |
| `--fresh-env` | — | Bypass environment cache |
| `--skip-deps` | — | Run only specified tasks, skip dependencies |
| `--no-deps` | — | Skip automatic dependency preparation |
| `--skip-tools` | — | Skip installing tools before running tasks |
| `--no-cache` | — | Skip cache on remote tasks |

**Task output cache flags** (experimental): `--task-cache <read-write|read-only|write-only|off|local-only>` (default `read-write`, env `MISE_TASK_CACHE`), `--task-cache-explain`, `--task-cache-explain-json` (**requires `--dry-run`**), `--task-cache-stats` (conflicts with `--dry-run`).

**Affected-set flags** (experimental): `--affected`, `--affected-base <REV>`, `--affected-head <REV>`, `--affected-explain`, `--affected-json`.

**Sandbox / permission flags** (stable since v2026.6.6; also available on `mise exec`):

| Flag | Description |
|------|-------------|
| `--allow-env <VAR>` | Allow specific env var through (supports wildcards; implies deny for everything else) |
| `--allow-net <HOST>` | Allow network access to specific host |
| `--allow-read <PATH>` | Allow filesystem reads from specific path |
| `--allow-write <PATH>` | Allow filesystem writes to specific path |
| `--deny-all` | Block reads, writes, network, and env vars |
| `--deny-env` | Block env var inheritance (keeps PATH/HOME/USER/SHELL/TERM/LANG/COLORTERM) |
| `--deny-net` | Block all network access |
| `--deny-read` | Block filesystem reads |
| `--deny-write` | Block filesystem writes (except implicitly writable system paths such as the temp dir) |

> `--allow-net <HOST>` per-host filtering is unsupported on Linux (errors); on Windows sandboxing is unavailable (warns and runs unfiltered).

> **mise's own flags must precede the task name:** `mise run --silent build`, not `mise run build --silent`. Extra args after the task go to the **last** command.

> **Breaking change (2026.7.6):** `--quiet` / `quiet = true` / `MISE_QUIET=1` **no longer collapse task output** to un-prefixed interleave — they preserve the resolved style. Use `--output quiet` or `-o interleave` for the old behavior.

> **Ctrl-C is an interruption, not a failure.** Since **2026.10.1** a single Ctrl-C stops new work, **waits for running tasks to clean up** (no duplicate SIGINT, no SIGTERM to siblings), then exits **130**; a second Ctrl-C force-quits. Since 2026.10.0 every mise command (`install`, `exec`, …) exits 130 on Ctrl-C, not 1.

> **Timeouts really stop tasks (2026.10.1+).** The whole-run `--timeout` / `task.timeout` sends SIGTERM then SIGKILL after 5s (Windows: immediate `taskkill /F /T`); it does not stop `raw = true` tasks. A timed-out task is reported failed even if it exits 0.

### Output Modes

- `prefix` — Default when `jobs > 1`; each line prefixed with task name
- `interleave` — Default when `jobs == 1`; print as output arrives
- `keep-order` — Stream one task's output live; buffer others; print in definition order
- `replacing` — Replace stdout on each new line (similar to `mise install`)
- `timed` — Show only stdout lines that took >1s
- `quiet` — Print only task stdout/stderr, nothing from mise itself. **Deprecated (warns since 2026.9.3), removed 2027.9.3** — use `output = "interleave"` plus `task.quiet = true` (`MISE_TASK_QUIET`) or a per-task `quiet = true`, or `--output interleave --quiet`.
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

A task is fresh when output mtime is newer than the newest source. The task definition itself is an implicit source, missing declared outputs force a run, and relative entries resolve from the task `dir` (`..` allowed).

> ⚠️ **Freshness does NOT respect `.gitignore`.** A change to a gitignored file matched by `sources` re-runs the task (verified). Only `mise watch` honors VCS ignores by default (override per task with `watch = { no_vcs_ignore = true }`).

**Exclusions** use gitignore-style `!` prefixes (escape a literal `!` path with `\!`); later entries override earlier ones. Since 2026.7.15, `outputs` supports the same ordered exclusions and re-inclusions:
```toml
sources = ["src/**/*.ts", "!src/**/*.test.ts", "!src/**/*.spec.ts"]
outputs = ["dist/**", "!dist/*.map"]
```
Excluded output paths are omitted from freshness hashes and cache archives; existing excluded files are preserved on restore.

Brace globs work in both (2026.8.0+): `sources = ["{src,lib}/**/*.ts"]`.

**Templated sources/outputs (2026.9.5+):** both accept `{{usage.*}}`, so freshness can be scoped per invocation:
```toml
[tasks.build]
usage = 'arg "<target>"'
sources = ["src/{{usage.target}}/**/*.ts"]
outputs = ["dist/{{usage.target}}/**"]
run = "tsc --build {{usage.target}}"
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

Redactions intercept task output line-by-line; tasks with `raw = true` bypass them. A variable marked `redact = false` opts **out** of matching `redactions` patterns (2026.8.0+), so a short non-sensitive value no longer pollutes the global scrubber.

**CI integration (GitHub Actions)** — mask redacted values safely (never `for v in $(…)`, which splits on whitespace):
```bash
mise env --redacted --json | jq -r '.[] | select(length > 0)
  | "::add-mask::" + (gsub("%"; "%25") | gsub("\r"; "%0D") | gsub("\n"; "%0A"))'
```

`jdx/mise-action@v5` handles this automatically. Note `mise env` itself prints **plaintext** — `--redacted` only filters *which* vars are shown. The default `prefix`/`interleave` output styles print full logs with redactions applied; only `replacing`/`timed` hide lines.

### Error Handling

Tasks run with `set -e` semantics by default (the default inline shell is `sh -o errexit -c`; simple commands may run without a shell at all). Disable locally:
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

> Docs: `https://mise.jdx.dev/tasks/caching.html`. Artifacts live in `$MISE_CACHE_DIR/task-artifacts/v2`. Unsupported outputs: `{ auto = true }`, absolute paths, or paths escaping the task dir. `raw`/`interactive` tasks bypass the artifact cache. `command_inputs` must exit 0 with non-empty output (≤16 MiB), inherit the task timeout (or 30s), and don't run during dry runs.

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
mise run --dry-run --task-cache-explain-json build # JSON Lines; --dry-run is required
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

Added **v2026.8.1**. A composite store: reads local first, promotes remote hits, and commits locally **then** mirrors writes so a remote failure never loses a local hit. Requests are hardened, verified, and streamed. Requests **carrying credentials** require HTTPS outside loopback; an unauthenticated HTTP endpoint is permitted with a warning.

> 🔴 **Renamed:** these settings moved from `task.cache_remote_*` to nested **`task.cache.remote_*`**. The flat spellings are hidden compatibility aliases — they start warning in **2027.2.0** and are removed in **2027.8.0**. Use the nested form.

| Setting | Env | Notes |
|---------|-----|-------|
| `task.cache.remote_url` | `MISE_TASK_CACHE_REMOTE_URL` | Endpoint |
| `task.cache.remote_namespace` | `MISE_TASK_CACHE_REMOTE_NAMESPACE` | **Required when `remote_url` is set**; opaque org/repo namespace isolating entries |
| `task.cache.remote_mode` | `MISE_TASK_CACHE_REMOTE_MODE` | `read-write` (default), `read-only`, `write-only` |
| `task.cache.remote_token` | `MISE_TASK_CACHE_REMOTE_TOKEN` | `Authorization: Bearer`. **Global-config only.** |
| `task.cache.remote_token_file` | `MISE_TASK_CACHE_REMOTE_TOKEN_FILE` | Re-read before each request (rotating creds, K8s projected SA tokens). **Global-config only.** |
| `task.cache.remote_oidc_audience` | `MISE_TASK_CACHE_REMOTE_OIDC_AUDIENCE` | GitHub Actions OIDC. **Global-config only.** |
| `task.cache.audit_report` | `MISE_TASK_CACHE_AUDIT_REPORT` | **2026.8.6+.** JSON Lines file capturing *every* undeclared read/write (`{"task","kind","path"}`). Console still caps at 20 paths/task. Truncated once per invocation, appended by later audited tasks. |
| `task.cache.stats_report` | `MISE_TASK_CACHE_STATS_REPORT` | Versioned JSON report of action-cache activity, transfer volume, restored outputs, phase timings (nanoseconds). Replaced atomically per `mise run`. |

**Credential precedence:** explicit bearer token → global-only token file → GitHub Actions OIDC. The credential settings are global-only so a shared project config cannot supply them.

> **Remote writes are policy-restricted:** only protected-branch push pipelines in GitHub Actions and GitLab CI may write. Pull requests, tags, unprotected branches, other CI systems, and local developer runs are read-only (a `write-only` remote mode disables the remote entirely in those contexts). The server is expected to enforce the same policy independently from verified OIDC claims.

Protocol reference: `https://mise.jdx.dev/tasks/remote-cache-protocol.html` (protocol version 1).

### Rust Compiler Action Cache — REMOVED (`rust_cache` is a no-op)

> 🔴 **`rust_cache` is a deprecated NO-OP. Do not use it.** The JSON schema marks the field `"deprecated": true` with the description *"deprecated no-op; use mbx for Rust action caching instead."* Setting `rust_cache = true` on a task or in `[task_config]` does **nothing** — it neither errors nor caches. Removal is scheduled for **2027.8.14**.
>
> Earlier versions of this skill documented it as a working experimental rustc action cache (concurrent blob prefetch, shared remote backend). That is no longer accurate.

**Migrate to [mbx](https://mr-boxington.jdx.dev/getting-started)** (Mr. Boxington), the separate `jdx` project that now owns Rust action caching:

```toml
# ❌ no longer does anything
[tasks.build]
rust_cache = true

# ✅ use mbx instead — e.g. as a command wrapper
[wrappers.cargo]
command = "mbx"
env = { MBX_CARGO_SHIM_MODE = "1" }
```

The task **output** cache ([Local Artifact Cache](#local-artifact-cache)) is unaffected and still works — it caches whole task results, not individual rustc invocations.

---

## Task Tracing with OpenTelemetry (Experimental)

Added **2026.9.13** (docs: `https://mise.jdx.dev/tasks/opentelemetry.html`). `mise run` can export one trace per run — a root span, setup spans (fetch remote tasks, resolve tasks, install tools, deps, start daemons), monorepo group spans, and a span per task — plus, optionally, every task output line as an OTLP log record.

```toml
[settings]
otel.enabled = true   # traces
otel.logs = true      # optional: task stdout (INFO) / stderr (WARN) as log records
```
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
mise run build ::: test
```

| Setting | Env | Default | Exports only when… |
|---------|-----|---------|--------------------|
| `otel.enabled` | `MISE_OTEL_ENABLED` | `false` | `OTEL_EXPORTER_OTLP_ENDPOINT` or `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` is also set |
| `otel.logs` | `MISE_OTEL_LOGS` | `false` | `OTEL_EXPORTER_OTLP_ENDPOINT` or `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` is also set |

The setting gate exists so mise doesn't emit spans just because other tools set `OTEL_EXPORTER_OTLP_*`. It does **not** require `experimental = true`.

- **Standard env honored:** `OTEL_EXPORTER_OTLP_{ENDPOINT,TRACES_ENDPOINT,LOGS_ENDPOINT,HEADERS,TRACES_HEADERS,LOGS_HEADERS,TIMEOUT,…}`, `OTEL_EXPORTER_OTLP_PROTOCOL` (`http/protobuf` default, or `http/json` — **no gRPC**), `OTEL_SERVICE_NAME` (default `mise`), `OTEL_RESOURCE_ATTRIBUTES`.
- **Task span attributes:** `mise.task.name`, `mise.task.args`, `mise.task.source`, `mise.task.config_root`, `mise.task.skipped`, `mise.task.cancelled`, `process.command_args`, `process.exit.code`. Only the task that actually failed is marked `Error`; siblings mise stopped get `mise.task.cancelled = true`.
- **Propagation:** `TRACEPARENT` / `TRACESTATE` are passed to tasks, so nested `mise run` calls and OTel-instrumented tools join the same trace. Nested runs coordinate log export via `MISE_TASK_OTEL_LOG_CLAIM` (each line exported once, by the innermost run).
- **Redactions** apply to args in span names/attributes, error messages, and exported log lines.
- **Failure handling:** 3s export timeout; failures are logged at debug and never fail the run. Offline mode disables export. On `--timeout` expiry the root span ends as `Error`; still-running tasks aren't exported.

> ⚠️ **`otel.logs` changes how tasks see their terminal.** In `interleave`/`quiet` output modes the child's stdio becomes a **pipe, not a TTY** — affecting `isatty()`, colours, progress bars, and prompts. Use `--raw` to keep a TTY (that output is then not exported). It is also a separate trust boundary: anything a task prints, including secrets that escape redaction, leaves the machine.

---

## Dev Tools Management

### Backends Overview

mise supports **20 backends** plus custom backend plugins (`mise backends ls` on 2026.10.3: aqua, asdf, cargo, conda, core, dotnet, forgejo, gem, github, gitlab, go, http, npm, **packslip**, **pypi**, s3, **spinel**, spm, ubi, vfox). The count is unchanged from 2026.9.12 because **pkgx was removed (2026.9.13)** and **spinel was added (2026.10.2, experimental)**. The registry assigns tools to backends by an **acceptance-tier** preference order — prefer the highest tier available for a given tool:

| Tier | Backends | When chosen |
|------|----------|-------------|
| **1 — Preferred** | **packslip**, **aqua**, **github**, **gitlab** | packslip when the publisher ships signed release manifests (best provenance + version-matched completions/man pages/agent skills). aqua otherwise offers the most features + security (cosign/SLSA/attestation/minisign, native Windows, no plugin). github/gitlab for releases not yet in aqua. |
| **2 — High bar** | **conda** | Lower bar than tier 3 because mise's conda backend needs no separately-installed package manager. |
| **3 — Very high bar** | **pypi**, **npm**, **gem**, **go**, **cargo**, **dotnet** | Depend on a separately-installed runtime on PATH; silently bind tools to whichever runtime was available at install time. |
| **Other** | **forgejo**, **http**, **s3**, **spm**, **spinel** | forgejo (Codeberg default); http for direct URLs; s3 for private buckets; spm for Swift; spinel (experimental) compiles a Ruby CLI to a native binary. |
| **Not accepted for new registry entries** | **vfox**, **asdf**, **ubi** | vfox/asdf rejected for supply-chain reasons; ubi deprecated. |

> 🔴 **`pipx:` → `pypi:` (2026.9.7).** `pypi:` is the preferred name for the Python CLI backend. `pipx:` remains **fully supported with no warnings**, and settings accept both `pypi.*` and `pipx.*` spellings. But the two are **distinct tool identities** (`pypi-black` vs `pipx-black` install dirs and lock entries), so switching spelling creates a separate installation. mise preserves explicit `pipx:` names in output and lockfiles.

> 🔴 **`pkgx:` backend REMOVED (2026.9.13, breaking).** `"pkgx:…"` tool keys no longer resolve — switch to a registry shorthand or an `aqua:`/`github:` spec. Lockfile `[pkgx-packages]` sections still load and are dropped on the next write. (The `pkgx` *registry entry*, i.e. the pkgx CLI itself, still exists.)

| Backend | Status | Description |
|---------|--------|-------------|
| **packslip** | stable | Signed release manifests published by maintainers; signer pinning (SSH-style), plus version-matched completions, man pages, and agent skills. No Packslip CLI needed. |
| **aqua** | stable | Most features, best security (cosign/SLSA/attestation/minisign). No plugins needed. Native Windows. Registry compiled into the mise binary; the aqua CLI is never used. |
| **github** | stable | GitHub releases with auto OS/arch/libc asset detection |
| **gitlab** | stable | GitLab releases |
| **forgejo** | stable | Forgejo/Codeberg releases |
| **http** | stable | Direct HTTP/HTTPS downloads with URL templating |
| **s3** | stable | S3/MinIO/S3-compatible private buckets (AWS SDK credential chain) |
| **pypi** | stable | Python CLIs in isolated environments (uses uv by default, pipx fallback). `pipx:` is a supported alias with a **separate tool identity**. |
| **npm** | stable | Node packages via the embedded `aube` installer — **node/npm not required to install** |
| **go** | stable | Go packages (requires compilation) |
| **cargo** | stable | Rust packages (uses binstall by default) |
| **gem** | stable | Ruby gems |
| **conda** | stable | Single conda packages direct from anaconda.org (no conda/mamba needed) |
| **dotnet** | stable | .NET tools |
| **spm** | stable | Swift packages |
| **spinel** | **experimental** | Ruby CLI from a GitHub repo compiled to a native binary by Matz's Spinel compiler (2026.10.2); requires `experimental = true` |
| **ubi** | **DEPRECATED** (warns; removal 2027.1.0) | Universal Binary Installer — migrate to `github` |
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

**Backend selection priority:** explicit spec (`aqua:owner/repo`) → `MISE_BACKENDS_<TOOL>` env override (highest — overrides registry *and* alias) → **a matching `mise.lock` entry keeps its recorded backend** → registry lookup (may depend on version via registry `min_version`/`max_version`, and on platform) → core tools → fallback. `[tool_alias]`/`[plugins]` and installed external plugins can override a shorthand; disabled backends are excluded from resolution.

**Locked backend wins over registry moves (2026.9.13, breaking).** When the registry moves a tool to a new backend, a locked tool keeps installing from the old one and mise warns `` `<tool>` is locked to `<old>`, but the registry now installs it from `<new>`. Run `mise backends switch <tool>`. `` Switch explicitly:

```bash
mise backends switch            # every configured tool whose locked backend was replaced
mise backends switch hk@1.58.1  # one tool/version
mise backends switch -n         # dry run;  -g = the global config's lockfile
```
`switch` moves the lock entries to the new backend at the **same versions**, relocks every platform (new URLs/checksums), and reinstalls; if any relock fails, every changed lockfile is restored.

**Registry override** per tool (SHOUTY_SNAKE_CASE; `my-tool` → `MISE_BACKENDS_MY_TOOL`):
```bash
export MISE_BACKENDS_PHP='vfox:jdx/vfox-php'
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
mise search npm:typescript-lang  # 2026.9.13: prefix npm:/cargo:/gem:/dotnet: to search that package registry
mise search -a <query>           # --all: registry, aqua, installed backend plugins, AND npm/cargo/gem/dotnet
mise tool <name> --url           # registry project URL (2026.9.13)
```

> **mise-versions for any public GitHub repo (2026.9.14).** `github:`/`aqua:`/`packslip:` tools outside the registry get version lists, release lookups, and attestation lookups from mise-versions (avoiding GitHub rate limits). mise treats it as an untrusted mirror (URLs must match repo/tag/asset), falls back to api.github.com on non-404 errors, skips it when `url_replacements` reroute the GitHub API, and paranoid mode re-checks "no attestations" answers against GitHub. `MISE_USE_VERSIONS_HOST=0` fetches from the source.

> **Registry default-backend changes (2026.9.6):** `postgres`, `redis`, and `mongodb` shorthands now prefer **`conda:`**. `mise use <tool>` therefore resolves differently than in earlier versions — pin the backend explicitly (`aqua:…`, `github:…`) if you depend on the old resolution.

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
| `os` | string \| string[] | Restrict to OS/arch: `"linux"`, `"macos"`/`"darwin"`, `"windows"`/`"win"`, and **`"unix"`** (2026.9.12+ — every non-Windows platform). Combos: `"linux/x64"`, `"macos/arm64"`. Arches: `arm64`/`aarch64`, `x64`/`x86_64`/`amd64`. A bare OS matches any arch; an entry with `/` requires both to match. A **concrete OS variant wins over a `unix` one**. Non-matching ⇒ mise skips installing **and using** the tool. The same selector syntax applies in `[bootstrap.packages]`, `[doctor.checks]`, and `[dotfiles]` variants. |
| `install_env` | table | Environment variables injected during install (and tool-level `postinstall`) |
| `postinstall` | string \| `{ run, when? }` | Command after successful install. `MISE_TOOL_INSTALL_PATH` available; the tool's bin dir is on PATH. **Table form (2026.9.17):** `postinstall = { run = "corepack enable", when = "always" }` — `when = "install"` (default) runs only on a fresh install or repair; `"always"` runs on **every** `mise install` that selects the tool, even when already installed (skipped on dry runs). `run` is required; other keys and invalid `when` values are parse errors. |
| `depends` | string \| string[] | Install-graph ordering for tools in the current install set. For vfox, `[tools].depends` is merged with the plugin's `PLUGIN.depends` into one install-dependency context — it affects ordering, the PATH seen by `os.execute`/`cmd.exec` in hooks (not `io.popen`), and `tools = true` env values. The asdf backend also puts `depends` tools on PATH for `bin/download`/`bin/install`, and Ruby source builds see them (2026.10.3). An unconfigured dependency may be satisfied from the system PATH. |
| `version_order` | `"source"` \| `"semver"` | **2026.8.4+.** Make `latest` and prefix resolution follow semantic precedence rather than source/chronological order. Supported on **aqua, github, gitlab, forgejo, http**. Fixes releases where a backport line outranked a newer version (neo4j, victoria-metrics, talosctl, tealdeer…). `mise ls-remote` still shows upstream source order. |
| `lazy` | bool | **2026.9.0+.** Defer installation until one of the tool's bootstrap shims is invoked. See [Lazy Tools](#lazy-tools). Default `false`. |
| `lazy_bins` | string \| string[] | **2026.9.0+.** Command names (no `/` or `\`) for lazy tools whose backend has no registry `bins` metadata. Single string accepted since 2026.10.2. |
| `minimum_release_age` | string | Per-tool override of the supply-chain delay (e.g. `"1d"`, `"0s"` to disable). A **built-in 24h default applies when unset** on timestamp-reporting backends — see [Lockfiles](#lockfiles-miselock). |
| `install_before` | string | **Deprecated** — maps to `minimum_release_age` (**warning live since 2026.10.0**, removed 2027.10.0). |
| `prerelease` | bool | Include prereleases where the backend supports it |
| `platforms` / `platform` | table | Per-`<os>-<arch>` overrides (`platforms.linux-x64.url`); `platform` is an alias |

> **Typed tool options in the JSON schema (2026.10.2).** Editors using `https://mise.jdx.dev/schema/mise.json` now validate and autocomplete options per backend prefix (github, gitlab, forgejo, ubi, http, s3, aqua, cargo, npm, pypi/pipx, gem, go, conda, spm, packslip, spinel, core python/java/rust/dotnet, `platforms.<os>-<arch>`, and `[tasks.*.tools]`) — existing configs with typos may start showing editor errors. Boolean options accept `true`/`false`, `"true"`/`"false"`, or `1`/`0`. Unknown options on generic tools are still allowed.

> 🔴 **Inline options require trust (security, 2026.9.18 / 2026.10.0).** Any tool **key** containing `[` — `"github:cli/cli[api_url=https://ghe.example.com/api/v3]" = "latest"` — makes a `mise.toml` require `mise trust`, and since 2026.10.0 the same applies to `.tool-versions` entries (GHSA-wcqh-j26q-g44x). Inline options can redirect downloads, so they are no longer treated as "safe" config. Prefer the table form, which is reviewed like any other config.

> `github_attestations` is **not** universal — it is a `github:` backend tool option. The separate *global* `github_attestations` setting (default `true`) applies to supported tools.

**Core-tool options** (typed in the schema):

| Tool | Option | Type / default | Notes |
|------|--------|----------------|-------|
| `rust` | `profile` | string | rustup profile (`minimal`, `default`, `complete`) |
| `rust` | `components`, `targets` | string \| string[] | rustup components / cross targets |
| `rust` | `mr_boxington` | bool, `false` | Wrap cargo with mbx (needs `mr-boxington` in `[tools]`): `mise use --tool-option mr_boxington=true rust mr-boxington` |
| `python` | `patch_sysconfig` | bool, `true` (unix) | Patch sysconfig of precompiled builds |
| `python` | `virtualenv` | string | **Deprecated** — use `env._.python.venv` |
| `java` | `release_type` | `ga` (default) \| `ea` | Early-access builds |
| `dotnet` | `runtime` | `dotnet` \| `aspnetcore` \| `windowsdesktop` | Install a shared runtime instead of the SDK |

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

#### Packslip Backend (Preferred where available)

Installs from **signed release manifests published by maintainers**. mise verifies the release, picks the build for your platform, and installs its executables — plus any completions, man pages, and agent skills the release declares. No Packslip CLI needed.

```toml
[tools]
"packslip:github.com/jdx/hk" = "latest"
"packslip:jdx/hk" = "latest"              # GitHub host may be omitted
"packslip:tool.example.com" = { version = "latest", pubkey = "/path/to/vendor.pub" }
```

| Identifier | Source |
|------------|--------|
| `packslip:github.com/owner/repo` | A GitHub repository's releases |
| `packslip:github.com/owner/repo/tools/mytool` | One tool in a GitHub monorepo |
| `packslip:tool.example.com` | A signed release list hosted by the publisher |
| `packslip:example.com/tools/mytool` | One tool on a publisher's domain |

The project **must publish Packslip manifests** — this backend never infers install instructions from arbitrary release filenames. GitHub projects get built-in release discovery and signer identity rules; other hosts need a signed release list plus explicit signer configuration.

**Tool options:**

| Option | Default | Purpose |
|--------|---------|---------|
| `variant` | none | Select a publisher-declared alternative build (`fips`, `baseline`). Without it, only artifacts with **no** variant are considered. |
| `pubkey` | unset | Pin a minisign-format public key, or the path to its `.pub` file |
| `identity` / `identity_prefix` / `issuer` | derived from a recognized forge | Expected keyless signer + OIDC issuer. Keep the trailing slash in a repository prefix. |
| `list_identity_prefix` | release signer policy | Pin a different workflow for the **vendor release list** only. Requires `issuer`; cannot combine with `pubkey`. |
| `prerelease` | `false` | Include prerelease versions |
| `trust` | configured stampers | `"vendor"` exempts this tool from stamp requirements |
| `allow_unlogged` | `false` | Accept key-signed bundles without transparency-log evidence |
| `ignore_requirements` | `false` | Install despite confirmed host requirement failures |

**Settings:** `packslip.exec` (default `false` — run a tool's own command at install time to produce a resource its packslip offers only as an `exec` entry; does **not** disable on-demand completion generation), `packslip.stampers` (hosts whose signed stamp lists say which releases may be installed, each with its pin).

**Version selection.** Only releases with Packslip manifests are visible. Semantic versions, including compatible date versions like `2026.9.1`. Prereleases excluded unless opted in. `latest` follows the publisher's `latest` pointer in a signed release list → GitHub's latest release → the highest eligible semver; a publisher may recommend an older supported release over a newer major.

> **The 24-hour `minimum_release_age` applies here**, so a just-published release may be absent until it cools. That 24h is mise's **built-in default for timestamp-reporting backends** (packslip among them) whenever the setting is unset — see [Lockfiles](#lockfiles-miselock).

**Private GitHub repos** work with the same credentials as `github:` (`MISE_GITHUB_TOKEN`, `GITHUB_API_TOKEN`, `GITHUB_TOKEN`) and need nothing in config. The token is transport-only: signature, project identity, signer continuity, and digest checks are unchanged, and no credential is read from the signed manifest.

**Signer pinning.** mise remembers which signer it accepted, the way SSH remembers hosts. A later release from another signer, a weaker scheme, a repackager where the vendor signed before, or one that drops build provenance is **refused until a person says so**:

```bash
mise packslip pins [--json]              # list accepted signers
mise packslip forget github.com/jdx/hk   # reset the pin so the next accepted release sets it again
```

`forget` also resets vendor release-list continuity. It does **not** change explicit signer options, erase stamper-list state, or remove a signer commitment from `mise.lock` — configure the new signer policy first, and if a lockfile entry conflicts, remove and regenerate it with `mise install`.

mise retains the verified manifest as `.mise-packslip.json` in the install directory.

> Verification authenticates the **publisher and bytes**, not that the software is safe. mise records whether build provenance links are present but does not fetch and verify that linked provenance.

#### PyPI Backend (formerly pipx)

```toml
[tools]
python = "3.14"
uv = "latest"
"pypi:black" = "latest"
"pypi:harlequin" = { version = "latest", extras = ["postgres", "s3"] }
"pypi:azure-cli" = { version = "latest", with = ["pip"] }
"pypi:ansible" = { version = "latest", expose = ["ansible-core"] }
"pypi:ansible" = { version = "latest", uvx = false, expose = [], pipx_args = "--include-deps" }
```

mise uses **uv**: with dependency locking it runs `uv sync --frozen`; version-only installs use `uv tool install`, falling back to `pipx install` when uv is unavailable.

**Package sources:** `pypi:black` (PyPI) · `pypi:psf/black` (GitHub) · `pypi:git+https://github.com/psf/black.git[@main]` (Git; the `.git` suffix is optional since 2026.9.15) · `pypi:git+ssh://git@github.com/org/repo` (2026.9.14). For **GitHub** sources `latest` is the **newest GitHub release**, falling back to the default branch only when there are no releases (2026.9.15); for other Git URLs `latest` resolves default-branch HEAD to a concrete commit. `latest` skips PEP 440 `.devN` releases (2026.9.14). Direct HTTPS archive URLs are unsupported.

**Monorepo subdirectories (2026.9.15)** — each subdirectory is its own tool:
```bash
mise use 'pypi:git+https://github.com/runpantheon/ltui#subdirectory=ltui@main'
mise use 'pypi:runpantheon/ltui#subdirectory=jtui@main'
```
Use the `extras` option (not `#egg=`) and set `package_name` if the guessed distribution name is wrong.

**Tool options:**

| Option | Notes |
|--------|-------|
| `extras` | String or array of Python package extras. Works with Git sources. Inline: `mise use 'pypi:psf/black[extras=jupyter]@latest'` |
| `package_name` | For Git repos whose name differs from the Python distribution name |
| `with` | Extra requirements installed **into** the tool environment. Requires uv; participates in locking. |
| `expose` | Like `with`, but also exposes their entry points. Requires **uv ≥ 0.8.5**; participates in locking. |
| `dependency_prereleases` | uv prerelease policy: `disallow` \| `allow` \| `if-necessary` \| `explicit` |
| `registry_url` | Per-tool index (include a `{}` placeholder for the package name); overrides `pypi.registry_url` |
| `uvx` | `false` selects the pipx installer (version-only locking) |
| `uvx_args` / `pipx_args` | Free-form installer args. **Version-only installs only** — rejected by explicit dependency locking. |
| `install_env` | Env during install |

> A `with`/`expose` requirement pinned to an exact version **narrows the locked Python range** to what that release supports, so the tool may need a newer interpreter than it declares. Guard with a marker: `"legacy==1.0.0; python_version < '3.12'"`.

**Settings:** `pypi.registry_url`, `pypi.uvx` (`pipx.registry_url` / `pipx.uvx` are compatibility aliases). The legacy names `uvx`/`uvx_args` control **uv** installation — they do not mean mise runs the `uvx` command.

Honors `minimum_release_age` via uv `--exclude-newer` (needs uv ≥ 0.2.22) or pip `--uploaded-prior-to`. Reinstall after a Python upgrade: `mise install --force pypi:black`.

#### Aqua Backend (Preferred)

```toml
[tools]
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:cli/cli" = { version = "latest", symlink_bins = true }  # Filtered .mise-bins directory
"aqua:flutter/flutter" = { version = "3.32.8", channel = "stable" }
"aqua:scenarigo/scenarigo" = { version = "0.21.0", vars = { go_version = "1.24" } }
```

**Tool options:** `symlink_bins` (bool — filters bundled bins into `.mise-bins`, using the registry's `files` field when defined; solves e.g. `aws-cli` bundling its own Python), `vars` (table, aqua registry template variables), `channel`, `prerelease` (bool — no effect when the package uses the `github_tag` version source, since git tags carry no prerelease flag; drafts always excluded), and:

| Option | Since | Notes |
|--------|-------|-------|
| `libc` | 2026.9.16 | `"glibc"` \| `"gnu"` \| `"musl"` — **strict** asset selection on Linux (no fallback to the other libc). Overrides the global `libc` setting, but not a musl host or a `linux-*-musl` lock platform. Recorded in the lockfile; already-installed versions keep their build until `mise install --force`. A registry template var literally named `libc` must be passed as `vars.libc`. |
| `slsa_signer_identity` + `slsa_signer_issuer` | 2026.10.0 | Replace the registry's SLSA signer (both required, non-empty; identity supports aqua's `{{.Version}}`). Don't enable SLSA where the registry has no `slsa_provenance`. |

```toml
[tools]
"aqua:domcyrus/rustnet" = { version = "latest", libc = "musl" }
# signer = the BUILDER workflow in the certificate (here the SLSA generator), not the repo's own workflow
"aqua:google/osv-scanner" = { version = "2.6.0", slsa_signer_identity = "https://github.com/slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@refs/tags/v2.1.0", slsa_signer_issuer = "https://token.actions.githubusercontent.com" }
```
Look the signer up in the project's release provenance rather than copying it from an unreviewed source.

> 🔴 **musl hosts changed (2026.10.0, breaking).** On Alpine/musl with no explicit request, aqua now installs **whatever asset the registry names** — a gnu build then needs glibc/gcompat. Set `libc = "musl"` per tool to keep musl builds.

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

`aqua.registries` also accepts a `file://` registry **file**; `MISE_AQUA_REGISTRIES` is comma-separated; `aqua.registry_url` is ignored when `aqua.registries` is set.

Aqua verification is native Rust (no cosign/slsa-verifier/gh CLIs) covering GitHub attestations, cosign, SLSA, minisign, and SHA256/512/1/MD5 checksums (always on). Verification failure aborts the install.

**Stricter signer checks (2026.9.16 – 2026.10.0, security):**
- **SLSA** passes only with a matching certificate identity **and** OIDC issuer (from the registry's `slsa_provenance.signer_identity`/`signer_issuer`, the tool options above, or a vfox `PreInstall`). With no expected signer, the SLSA check is **skipped** (not passed). DSSE bundles must carry a matching SHA-256 subject.
- **Keyless cosign** requires a pinned identity — mise applies the registry's `--certificate-identity[-regexp]`, `--certificate-oidc-issuer[-regexp]`, and `--certificate-github-workflow-*`; unknown or empty `--certificate-*` options are errors.
- **GitHub attestation** `signer_workflow` uses an anchored whole-segment match; an empty value fails.
- A lock entry with a checksum and recorded provenance is trusted digest-only; `locked_verify_provenance` / paranoid mode re-verify and require a signer.
- **42 registry tools** (aube, aqua, pixi, ty, pandoc, fnox, doppler, syncthing, …) declare `attestations_since = "<semver>"` (2026.9.14): from that version a missing GitHub attestation is a **hard error** unless you disabled `github_attestations`.

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

**Tool options:** `asset_pattern`, `additional_asset_patterns`, `matching`, `matching_regex`, `version_prefix` (default `v`), `platforms` (per-platform `asset_pattern`/`additional_asset_patterns`/`url`/`checksum`/`size`/`bin`/`bin_path`/…), `strip_components`, `bin`, `rename_exe`, `bin_path`, `filter_bins`, `format` (force archive type: `tar.gz`, `zip`, `7z`, `raw`, …), `checksum`, `size`, `no_app`, `api_url`, `prerelease`, `github_attestations`, and **`slsa_signer_identity`** (Tera — `{{ version }}`) + **`slsa_signer_issuer`** (2026.9.16; without both, SLSA is skipped; with `locked_verify_provenance` a missing signer is an error).

```toml
"github:myorg/mytool" = { version = "latest", slsa_signer_identity = "https://github.com/myorg/mytool/.github/workflows/release.yml@refs/tags/v{{version}}", slsa_signer_issuer = "https://token.actions.githubusercontent.com" }
```

**Direct per-platform URL** — skip asset selection entirely: `platforms.linux-x64.url = "https://…/tool-{{ version }}-linux.tar.gz"` (or flat `platform_linux_x64_url`). `{{ version }}` is the resolved version even for `latest`. A **top-level** `url` is ignored by github.

**Listing:** releases with **no assets** are hidden from `mise ls-remote` (unless every platform has a `url`); `MISE_LIST_ALL_VERSIONS=1` reads every page. Autodetection skips SBOM/signature/checksum sidecar assets (2026.10.1). On a checksum mismatch mise hints that the maintainer probably re-uploaded the asset.

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

Same forge option surface as GitHub (`asset_pattern`, `additional_asset_patterns`, `matching`, `matching_regex`, `version_prefix`, `platforms` incl. per-platform `url`, `bin`, `bin_path`, `rename_exe`, `filter_bins`, `format`, `checksum`, `size`, `strip_components`, `no_app`, `api_url`). `slsa_*` and `github_attestations` are **github-only**. Forgejo also supports `prerelease` and defaults `api_url` to `https://codeberg.org/api/v1`. `prerelease` has **no effect on GitLab**.

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
url = "https://example.com/releases/my-tool-v{{version}}.tar.gz"   # also file:///abs/path.tar.gz (2026.9.13)
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
- `version_expr` — expr-lang over `body` **and** `versions` (the output of `version_regex`/`version_json_path`); it is the **final post-processing step**, not a precedence switch. `sortVersions(array)` is available.

**Other http options:** `file://` URLs (2026.9.13) copy a local archive, still verify `checksum`, and work offline (the URL is locked exactly as written). `windows_script_interpreter = "python"` (Windows only) writes a `.cmd` launcher that runs a raw script with that interpreter. `bin` names a downloaded raw/compressed single binary (ignored for archives; OS/arch suffixes are auto-stripped). Setting `bin_path` disables automatic root stripping.

> **expr-lang gotchas:** write the predicate placeholder as `{ #... }` **with a space** after `{`, because `{#` is the Tera comment delimiter. Index a map by runtime value with `[version + ""]` — a bare `[version]` is read as the literal key `"version"`.

> 🔴 **Changed default (2026.9.6): HTTP tools now extract into their own install directory.** Previously installs were **symlinks into a deduplicated store** (`http-tarballs/`), which meant `uninstall`/`prune` could not reclaim the disk. Own-directory extraction is now the default so that space is actually freed.
>
> Opt back into deduplication per tool with **`shared_extraction = true`**. Existing symlinked installs are unaffected until `mise install --force`, and **legacy `http-tarballs` entries are not reclaimed automatically**.

```toml
[tools."http:my-tool"]
version = "1.0.0"
url = "https://example.com/my-tool.tar.gz"
shared_extraction = true    # opt back into the old shared/dedup cache layout
```

**Shared-store behavior (when `shared_extraction = true`):** entries live in **`$MISE_DATA_DIR/http-tarballs/`** — deliberately *outside* the cache dir so `mise cache clear` can't break the links — keyed by Blake3 of the content plus extraction options (effective filename, root stripping, renaming, format/launcher). Installs are symlinks into it, so identical artifacts are shared across tools. **Neither `mise prune` nor `mise cache prune` reclaims entries** (there is no auto-prune). `--system`/`--shared`/`install-into` always get their own files.

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

#### Pipx Backend (alias of `pypi:`)

`pipx:` is a fully supported alias for [`pypi:`](#pypi-backend-formerly-pipx) — no warnings, same options, and settings accept both `pipx.*` and `pypi.*` spellings. Prefer `pypi:` in new configs.

```toml
[tools]
"pipx:black" = "latest"                                    # still valid
"pipx:harlequin" = { version = "latest", extras = "postgres,s3" }
"pipx:ansible" = { version = "latest", uvx = false }       # use the pipx installer
```

> ⚠️ **`pypi:black` and `pipx:black` are distinct tool identities** — separate install directories and separate lock entries. Switching the prefix creates a **new installation**; it is not a rename. mise preserves explicit `pipx:` names in output and lockfiles.

Reinstall after a Python upgrade: `mise install -f "pipx:*"`.

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
| `allow_exotic_deps` | aube | **2026.9.10+.** Array of package names, or `true` for all — permits git/file/tarball dependencies that aube blocks by default |
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

**Tool options:** `install_env`, and **`source`** (2026.9.18) — a per-gem registry URL, added alongside the default sources rather than replacing them:
```toml
"gem:internal-tool" = { version = "1.4.2", source = "https://rubygems.pkg.github.com/myorg" }
```
Credentials go in the URL as basic auth (redacted in output; https required except on localhost). A `https://rubygems.pkg.github.com/<org>` source **without** credentials uses mise's GitHub token (needs `read:packages`) and requires an **exact version pin**, because GitHub Packages can't list versions.

Requires `gem` (Ruby) on PATH. Reinstall after a Ruby upgrade: `mise install -f "gem:*"`.

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
"spm:org/tool" = "rev:abc1234"   # 2026.8.4+ — pin to a commit, builds from source
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

#### Spinel Backend (Experimental)

Added **2026.10.2** (docs: `https://mise.jdx.dev/dev-tools/backends/spinel.html`). Compiles a Ruby CLI from a GitHub repo into a **native binary** with Matz's Spinel compiler — the result needs neither Ruby nor Spinel at runtime. Requires `experimental = true`; may be removed in a future release.

```toml
[settings]
experimental = true

[tools."spinel:tobi/try"]
version = "1.10.1"
entrypoint = "try.rb"
bin = "try"
tag_prefix = "v"
```

| Option | Default | Notes |
|--------|---------|-------|
| `entrypoint` | `main.rb` | The single file compiled |
| `bin` | repo name | Output binary name |
| `tag_prefix` | none | Only tags with this prefix are listed as versions |
| `source_ref` | none | Full 40-char commit SHA; the version becomes a label only |
| `spinel` | `spinel` on PATH | Path to the compiler (mise does **not** install it) |

**Requirements:** macOS or Linux only, `git`, a C compiler, and the `spinel` compiler on PATH. Versions come from `git ls-remote` tags (no GitHub API). Compiles **one** entrypoint — no gems, data files, or submodules, and Spinel handles only part of Ruby.

> **`pkgx:` was removed in 2026.9.13** — see the note under [Backends Overview](#backends-overview).

#### UBI Backend (Deprecated)

**Options:** `exe`, `rename_exe`, `matching`, `matching_regex`, `provider`, `api_url`, `extract_all` (incompatible with `exe`/`rename_exe`), `bin_path` (only meaningful with `extract_all`), `tag_regex`.

> **Migration gotchas:** (1) ubi folds `matching` into the install path so one repo can supply several binaries; `github` keys paths by tool name + version, so different `matching` values collide — give each binary its own `[tool_alias]`. (2) ubi applies substring `matching` only as a *tiebreaker* among assets already matching your OS/arch and skips it when a single asset matches, whereas `github` applies it as a *pre-filter* before autodetection — so you get the named binary or a clear error.

#### vfox & asdf Plugins

**vfox (recommended plugin system):** cross-platform (Win/macOS/Linux), built-in Lua interpreter with HTTP/JSON/archive modules, attestation verification, lock files. **Tool option:** `install_env` (applies to `cmd.exec` during install hooks only — vfox's built-in Lua helpers do not use it); any other options reach hooks as **`ctx.options`** (`MISE_TOOL_OPTS__*` env is legacy). Since 2026.8.1 Lua plugins gain `strip_components = 1` on `archiver.decompress`, plus sorted `file.list`, `file.glob`, and `file.move`. vfox honors `url_replacements` and `netrc`.

- **New hooks:** `hooks/backend_uninstall.lua` (backend plugins; runs before removal on uninstall/upgrade/prune, keeps the install dir if it errors — 2026.9.13) and `hooks/mise_install_satisfied.lua` (re-runs `PostInstall` plus the tool's `postinstall` when options change, e.g. gcloud `components` — 2026.9.15).
- **Signer fields (security):** a `PreInstall` returning SLSA provenance must also return `slsa_signer_identity`/`slsa_signer_issuer` (else SLSA is skipped); keyless cosign must set `cosign_certificate_identity` or `cosign_certificate_identity_regexp` (optional `cosign_certificate_oidc_issuer`) — **breaking in 2026.10.0**.
- **Signed plugin distribution:** `mise plugins install vfox:NAME 'packslip:OWNER/REPO#PLUGIN_VERSION'` or `[plugins] "vfox:NAME" = "packslip:OWNER/REPO#VER"`.
- Registry **vfox** plugins live under the `jdx` org (e.g. `vfox:jdx/vfox-php`); asdf plugins remain under `mise-plugins` (`asdf:mise-plugins/asdf-php`).

**asdf (legacy):** Unix-only, bash scripts, needs curl/jq, no Windows, disabled by default on Windows. New asdf tools rarely accepted for supply-chain reasons. **Tool option:** `install_env`. Since 2026.7.18, `depends` tools are on the PATH given to `bin/download` and `bin/install`, and `bin/list-all`/`bin/latest-stable` receive resolved `[env]` values and `_.path` additions.

> Tool plugins support attestation verification; **backend plugins do not** — a known security gap.

```bash
mise plugin install my-plugin https://github.com/username/my-plugin
mise install my-plugin:some-tool@1.0.0
```

### Lazy Tools

Added **2026.9.0**. `lazy = true` makes mise generate **bootstrap shims** into its normal user/system shim farms without installing the tool. The provider is installed only when one of its commands is first called, then executes immediately; later calls run the real binary with no further mise dispatch.

```toml
[tools]
node = { version = "24", lazy = true }
"github:example/acme" = { version = "1.2.3", lazy = true, lazy_bins = ["acme", "acmectl"] }
```

- **Registry tools** derive their command names from the registry's `bins` metadata. **Explicit or non-registry backends must declare them** with `lazy_bins`.
- A bare `mise install` **skips** lazy declarations — use `mise install --include-lazy` to provision them all.
- Lazy tools also install when their command is invoked from a `mise run` task or `mise x` (2026.9.1+), matching activated-shell behavior. mise inserts the shim farms *after* real tool paths for lazy toolsets.
- Lazy tools are **not** reported as `missing: <tool>` on project entry or a bare `mise install`, regardless of `status.missing_tools` (2026.9.8+). Ordinary missing tools still are.
- Invoking a lazy shim also installs that provider's configured `depends`. `mise install <lazy-tool>` installs just that one. Run `mise reshim` after hand-editing a lazy declaration.

**Related path settings (2026.9.0+):** `shims_dir` (`MISE_SHIMS_DIR`), `system_installs_dir` (`MISE_SYSTEM_INSTALLS_DIR`), `system_shims_dir` (`MISE_SYSTEM_SHIMS_DIR`), plus `mise reshim --system` — for system-scoped and collocated layouts.

> For a machine-wide catalogue of ordinary tools, prefer `lazy = true` over standalone [tool stubs](#tool-stubs). Tool stubs remain useful when the executable file itself should carry a portable, self-contained definition.

### Tool Stubs

An executable file that records how to obtain and run one tool — commit it so `./bin/py` selects the intended runtime. mise installs the tool on first execution; a normal project `mise install` does **not** discover and install arbitrary stub files.

```toml
#!/usr/bin/env -S mise tool-stub

tool = "python"
version = "3.14"
bin = "python"
```

```bash
chmod +x ./bin/py && ./bin/py --version
```

Fields sit at the **top level** — a stub is a tool declaration, not a whole `mise.toml`, so do **not** wrap them in `[tools]`. `tool`, `version` (default `latest`), `bin` (default the stub filename), `os`, and `install_env` control the stub; other keys pass to the selected backend. Omitting `tool` makes a top-level or platform-specific `url` select the HTTP backend; otherwise the **filename** is used as the tool name. A stub requires `mise` on PATH unless generated with the optional bootstrap wrapper (`--bootstrap`). A Windows `.cmd` launcher is generated beside it, and executed stubs are tracked in `~/.local/state/mise/tracked-stubs` so `mise prune` keeps their versions.

> 🔴 **Stub locking moved to `mise.lock` (2026.9.13, breaking).** An embedded `[lock]` section in a stub is now **ignored and removed**. `mise generate tool-stub ./bin/node --lock [--version 26]` writes lock data to the `mise.lock` of the **nearest project config above the stub** (it fails if there is none) and lists the stub under a top-level `tool-stubs = ["bin/node"]` array. The stub keeps its fuzzy request; `mise lock --bump` re-resolves stubs and prunes deleted ones. In locked mode a stub with no entry for the current platform is rejected.

`mise generate tool-stub` flags: `--lock`, `--version`, `--bootstrap`/`--bootstrap-version`, `--platform-url [PLATFORM:]URL`, `--platform-bin PLATFORM:PATH`, `--checksum-algorithm blake3|sha256` (default blake3; not with `--lock`/`--skip-download`), `--fetch`, `--skip-download`, `--http`, `-u/--url`, `-b/--bin`.

> `env -S` is required because Unix shebangs traditionally allow only one argument after the interpreter — it splits the line into `env` → `mise` → `tool-stub`.

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

Shells with a cd hook: `bash` (chpwd emulation that wraps `cd`/`pushd`/`popd`, plus `PROMPT_COMMAND`), `zsh` (`chpwd`), `fish` (`fish_prompt`), `xonsh` (`on_chdir`). Without one, `cd a && node -v` on a single line uses the *original* directory's tools — shims always work there.

**`mise reshim`** regenerates shims for **all installed** tools, not just active ones. It only replaces or removes entries it recognizes as mise shims (`-f/--force` rebuilds mise-owned shims; it does not adopt unrelated files), so a shared `shims_dir` like `~/.local/bin` works for reshim — though not for `mise activate`/hook-env. Runs automatically on install/update/remove. Exclude tools from shim generation with `[settings.shims] exclude = [...]` (2026.9.10); `activate_shims` and `not_found_system_fallback` both default `true`.

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
project-lts = "{{ env.PROJECT_NODE_VERSION | default(value='24') }}"
```

> Don't compute a tool's version by **invoking that same tool** (e.g. `exec(command='node --version')`) — the docs now explicitly discourage it: resolution can happen before the tool exists, or re-enter mise through a shim.

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

`mise completion --install` writes self-contained scripts directly.

> 🔴 **2026.8.11:** mise's own CLI moved from clap to usage-rs, and completions/help are now generated from compiled usage metadata rather than an external `usage` CLI. As a result **`--include-bash-completion-lib` and `--usage` are now no-ops.** Command behavior, flags, and aliases are otherwise preserved.

> **Standalone usage scripts** need a separate one-time opt-in: `source <(usage g completion-init bash)` in `~/.bashrc` (zsh: same in `~/.zshrc`; fish: `usage g completion-init fish | source`). Since usage 3.5.6 the completion spec cache lives in `${XDG_CACHE_HOME:-$HOME/.cache}/usage` — **regenerate completion scripts** to pick it up.

### Packslip, Man Pages, and Agent Skills

Tools installed through the [`packslip:` backend](#packslip-backend-preferred-where-available) can ship **man pages, shell completions, and agent skills that match the version active in your project**. The publisher declares them in the release manifest. An existing installation from another backend does **not** acquire them.

**Completions** (zsh, bash, fish, PowerShell) become available automatically with `mise activate` — the installed completion follows the tool version active in each project, so changing versions needs no reinstall. mise registers a loader and reads the publisher's script only when you complete a command; leaving the project removes the registration.

```bash
mise completion zsh --tool hk --install    # manual setup without shell activation
mise completion zsh --tool hk              # print without installing
```

`--tool` takes the **command name**, not a backend identifier. Without `--tool`, `mise completion` generates completions for mise itself. mise writes the completion file but never edits your shell config, and preserves an existing file it did not create unless you pass `--force`.

**Man pages** declared as a static `man` resource are added to `MANPATH` while that tool version is active (in an activated shell and under `mise exec`/`run`/`env`). They live under `.mise-packslip/man` in the tool's installation; a tool installed with an older mise must be reinstalled once. Generated `exec` resources and pages derived only from a `cli-spec` are **not** installed automatically.

**Agent skills** — a directory holding `SKILL.md` and its supporting files, in the Agent Skills format. mise knows which tool version is active, so it hands an agent the skill for exactly that version.

```bash
mise skills ls [--json]                    # skills the active tools declare
mise skills sync --dir .agents/skills      # link them where your agent looks
```

| Setting | Default | Description |
|---------|---------|-------------|
| `skills.fetch` | `true` | Fetch the agent skills a tool's packslip declares when installing it |
| `skills.auto_sync` | `false` | Link active tools' skills into the project after `mise install` and `mise use` |
| `skills.dir` | `.claude/skills` | Where `mise skills sync` links skills — under the project root, or under `$HOME` with `--global` |
| `skills.prune` | `false` | Remove links mise made for skills that are no longer active when syncing |

> If a completion needs the publisher's generator command, mise runs it **on demand** and caches successful output per installed version/executable/shell. `packslip.exec` controls resource generation **at install time**; setting it to `false` does **not** disable on-demand completion generation.

> Since **2026.10.1** a packslip install **fails** if a declared skill can't be fetched; the next install fetches only the missing skills. Packslip pins by forge **repository ID** (lockfile revision 3), so a renamed (2026.9.16) or transferred (2026.10.2) repo keeps installing with a one-time warning, while a different repo under the old name is refused — the error names what to clear (`mise packslip forget`, the `mise.lock` entries, or both). The registry has moved timoni, worktrunk, helmfile, and dagu (newer versions) to packslip; older versions remain reachable via an explicit `aqua:` spec. **usage 6.12** itself ships a version-matched skill: `mise use usage && mise skills sync`.

### Lockfiles (`mise.lock`)

**Lockfiles are not created automatically.** With `lockfile` unset, mise **updates existing lockfiles but doesn't create new ones** — create one with `mise lock` (or `touch mise.lock && mise install`). `MISE_LOCKFILE=1` keeps that update-only behavior and is *not* the same as `lockfile = true`. Global lockfiles are only created by `mise lock --global`.

```toml
lockfile_version = 3

[[tools.node]]
version = "20.11.0"
backend = "core:node"
specifiers = ["20"]

[tools.node."platforms.linux-x64"]
checksum = "sha256:a6c2..."
url = "https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-x64.tar.xz"

[[tools.ripgrep]]
version = "14.1.1"
backend = "aqua:BurntSushi/ripgrep"
options = { exe = "rg" }
```

The writer uses a **quoted** platform key — `[tools.node."platforms.linux-x64"]` — and omits `size`.

Tool entry fields: `version` (required), `backend`, `specifiers` (the original requests that resolved here), `options` (backend-specific artifact identity), `platforms`, plus `aube` / `uv` (revision 2 — sidecar path + digest for npm / Python dependency graphs). A top-level `tool-stubs = [...]` array lists locked [tool stubs](#tool-stubs) (2026.9.13).
Platform sub-fields: `checksum` (SHA256 or Blake3), `url`, `url_api` (authenticated asset requests), `provenance` (which method verified it — SLSA / cosign / minisign / GitHub attestations), `signer` / `attested_by` (packslip identity), **`repository_ids = { repository = "<id>" }`** (revision 3 — the forge repository ID packslip pins), `size` (**legacy, read-only now**). `provenance_verified` is inert compatibility metadata — mise neither reads nor writes it.

**Backends exempt from strict URL-locking** (`--locked` does not require a resolved URL): `asdf`, `cargo`, `gem`, `go`, `npm`, `pypi`/`pipx`, `ubi`, vfox **backend** plugins, `core:dotnet`, `core:rust`, `core:swift`. URL-lockable — `aqua`, `github`, `gitlab`, `http`, `s3`, `packslip`, and vfox **tool** plugins — must have a resolved URL for the current platform. Tool stubs follow the same rules via the project `mise.lock`.

> A `provenance` field alone is **not proof the bytes were verified**. The lockfile is a trust input requiring review, not an automatic guarantee.

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
mise lock --sidecars [--json]   # 2026.9.18: list the native dependency sidecar dirs to commit (writes nothing)
mise lock --upgrade             # migrate to the newest lockfile revision (transactional)
```

`mise lock --bump` checks remote versions for every selector and **fails** when a list can't be fetched (2026.9.13 — it used to exit 0 with stale data). For bots, `MISE_SAFE=1 mise lock --bump --json` is recommended. `mise lock` also resolves **task tools** into the lockfile.

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

**Backend lockfile support by family:** download backends (aqua, github, gitlab, forgejo, http, s3) record per-platform artifact metadata (URL + checksum + provenance); `packslip` records signed artifact info plus signer and repository commitments; `npm` (embedded aube) and `pypi` (uv) add revision-2 dependency-graph sidecars; other language package managers (cargo, gem, go, dotnet) lock top-level versions only; vfox tool plugins are URL-lockable; asdf and vfox backend plugins have no strict URL requirement. **Provenance support:** `aqua`, `github`, `core:python`, `core:ruby`, `core:zig`.

For the current platform, `mise lock` **downloads the artifact and performs full cryptographic verification at lock time**, so the entry is backed by real verification rather than registry metadata. Cross-platform entries record detected provenance without verifying. `github_attestations = "unavailable"` is a **negative cache entry, not provenance** — SLSA/cosign/minisign/checksum verification still runs, and a later `mise lock` can discover attestations added after release.

Settings: `lockfile` (unset ⇒ update existing lockfiles, never create new ones), `locked = true` (require lockfile-resolved URLs; blocks API calls — good for CI), `locked_verify_provenance` (default `false`; auto-on with `paranoid`), `lockfile_platforms`, `lockfile_mode` (`merge` | `generate`), `locked_scopes` (default `["global","project","system"]`).

**Lockfile revisions** (`lockfile_version`):

| Revision | Since | Adds |
|----------|-------|------|
| 0 | — | Unversioned legacy format |
| 1 | 2026.8.11 | Binds each original request to the entry it resolved, so `"1"` and `"1.0.0"` can lock different versions |
| 2 | 2026.9.7 | npm/pypi dependency-graph sidecars (see below) |
| **3** | **2026.9.16** | `repository_ids` — packslip pins by forge repository ID, so a **renamed** (2026.9.16) or **transferred** (2026.10.2) repo keeps installing with a one-time warning, while a *different* repo re-created under the same name is refused |

🔴 **New and empty lockfiles are written as revision 3, which older mise rejects** — upgrade collaborators and CI to **2026.9.16+** before committing one. Existing lockfiles keep their revision during ordinary `lock`/`install`/`upgrade`; run **`mise lock --upgrade`** (transactional; rolls back on failure) to migrate. Since 2026.9.17 mise warns when a lockfile's format was superseded more than 6 months ago.

#### Dependency-Graph Sidecars (revision 2+, 2026.9.7)

Revision 2 records the **full transitive dependency graph** of `npm:` tools (via embedded aube) and `pypi:` tools (via uv), then replays it with a strict frozen install — so two projects on the same top-level version can each keep their own reviewed graph.

Graphs live in **native sidecar files** (`uv.lock` / `aube-lock.yaml` plus a manifest) under `.mise/locks/<backend-tool>/<version>/`, referenced from `mise.lock` by relative path and SHA-256 digest, so the lockfile itself stays small.

```bash
mise lock --upgrade            # move an existing lockfile to the newest revision and resolve graphs
mise install --locked          # replay the recorded graphs
mise lock --bump pypi:black    # refresh Black's transitive deps without changing its version
uv tree --project .mise/locks/pypi-black/24.10.0    # inspect a sidecar directly
```

- **Commit the sidecar directory alongside `mise.lock`.** If you gitignore `mise.local.lock`, also ignore its sidecar subdirectory (e.g. `.mise/locks/mise.local/`).
- Sidecar locations follow the lockfile (`.mise/locks/`, `.config/mise/locks/`) or a symlink target; digests normalize CRLF. `mise lock --sidecars` lists what to commit.
- Ordinary `mise install` validates and accepts hand-edited sidecars; `mise install --locked` **rejects digest mismatches** until you run `mise lock`.
- Python graph locking needs **uv ≥ 0.12.10** and published **wheels** for the target platform — explicit `mise lock` and locked installs never build sdists. Git sources, standalone pipx installs, and free-form `uvx_args`/`pipx_args` stay **version-only**.
- Lock generation needs an interpreter discoverable by uv, though it need not match the tool's configured Python version.

> 🔴 **Revision 2+ is not readable by older mise versions.** Revision 2+ `--locked` installs fail if a recorded graph is missing or its digest doesn't match.

> ✅ **Fixed 2026.9.7:** installing from a committed `mise.lock` no longer fails when the locked release is younger than `minimum_release_age`. The cutoff still applies to unlocked fuzzy requests and to generating/bumping a lockfile, so a reviewed lock entry reproduces immediately in CI.

> ⚠️ **Settings are global in scope.** `locked = true` in a project's `mise.toml` applies to *all* tool resolution, including tools from `~/.config/mise/config.toml`. Fix warnings about global tools with `mise lock -g`.
>
> ✅ **Use `[tool_config]` for project-scoped strictness instead (2026.8.6+).** This is a top-level config key — not a setting — and applies only to tools declared by configs sharing that config root, without forcing global or parent-root tools into strict mode:
> ```toml
> [tool_config]
> locked = true
> ```
> `[settings] locked`, the `--locked` flag, and `MISE_LOCKED` remain invocation-wide.

`minimum_release_age` (env `MISE_MINIMUM_RELEASE_AGE`) filters fuzzy version requests by release date — supply-chain delay protection. Accepts relative durations (`7d`, `90d`, `6mo`, `1y`) and absolute dates (`2024-06-01`, `2024-06-01T12:00:00Z`); `"0s"` disables it. Versions without release timestamps are included. Only **`npm:` and `pypi:`/`pipx:`** forward the cutoff into transitive dependency resolution. Exempt tools with `minimum_release_age_excludes = ["trivy", "npm:*"]` (backend wildcards, registry shorthands, or full IDs), or per-tool:
```toml
[tools.trivy]
minimum_release_age = "1d"
```
Note durations use jiff format — months are `6mo`, not `6m`. The deprecated `install_before` setting maps to this (**warning live since 2026.10.0**, removed 2027.10.0). The cutoff is also overridable per command with `--minimum-release-age` on `use`, `install`, `upgrade`, `ls-remote`, `latest`, and `lock`.

> **mise itself now honors a release age (2026.9.17).** Unpinned `mise self-update`, automatic updates, update notifications, and the `curl https://mise.run | sh` installer pick the newest stable release **at least 24h old**. Precedence: `self-update --minimum-release-age <DUR>` → `self_update.minimum_release_age` (`MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE`; optional, no default of its own) → `minimum_release_age` → `24h`. Because it inherits `minimum_release_age`, a `7d` tool cutoff also delays mise upgrades by 7 days. Explicit versions bypass it (`--force` does not). The installer reads env vars only (`MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE` → `MISE_MINIMUM_RELEASE_AGE` → `24h`); set `MISE_VERSION` for reproducible installs. Self-update also requires a **signed packslip** for releases ≥ 2026.9.3.

> 🔴 **There IS a built-in 24h default — the setting being "unset" does not mean "off."** This is a subtle trap. `mise settings get minimum_release_age` reports *not set* and the JSON schema carries **no `default`**, because `settings.toml` declares `default_docs = "24h"` (a docs-display value) rather than `default`. But mise's source defines `DEFAULT_MINIMUM_RELEASE_AGE = "24h"` and applies it whenever the setting is unset, for backends that report release timestamps:
>
> **aqua, cargo, core, forgejo, gem, github, gitlab, go, npm, packslip, pipx/pypi, spm, ubi.**
>
> It does **not** apply to backends without release timestamps (http, s3, conda, dotnet, asdf, vfox). The `go` backend dates only the newest 100 (proxy) / 10 (`go list`) versions, checks undated ones individually, and **warns and allows** a version if the date lookup fails. Set `minimum_release_age = "0s"` to genuinely disable it, or a longer duration to harden. mise distinguishes the built-in default from an explicit value internally, so a fresh release being invisible for 24h is expected behavior, not a bug.
>
> **Never filtered regardless:** an explicit pin (`hk = "2.0.1"`) and a lockfile-resolved version — both are decisions already taken and committed. Since 2026.9.7, installing from a committed `mise.lock` no longer fails when the locked release is younger than the cutoff.

CI flag: `--locked` (global flag on any command) errors if any version isn't resolved via the lockfile.

### Auto-Install Controls

All are enabled by default and all require the master `auto_install`:

- `auto_install` (default `true`) — master switch
- `exec_auto_install` (default `true`) — `mise x`/`mise r`
- `task.run_auto_install` (default `true`) — the dev-tools doc page calls this `task_auto_install`, which does not exist; use the dotted form
- `not_found_auto_install` (default `true`) — maps command → tool via registry `bins` metadata, so it covers **configured tools even if never installed**; not raw backend specs (`cargo:x`, `ubi:x`)
- `not_found_auto_install_registry` (default **`false`**, 2026.9.17) — for an **unconfigured** command: if **exactly one** enabled registry tool provides it, install its latest version and add it to the **global** config (like `mise use --global`). Ambiguous matches are skipped
- `auto_install_disable_tools = ["..."]` — per-tool skip list

### CLI Commands for Tools

```bash
mise use node@22            # Install + activate + write to mise.toml
mise use -g node@22         # Write to global config
mise use --pin node@22      # Pin exact resolved version (e.g. 22.5.1)
mise use -e staging node@22 # Write to mise.staging.toml (subcommand -e/--env; the GLOBAL -E only selects
                            # the env for loading, so `mise -E staging use` still writes mise.toml)
mise use --fuzzy node@22    # Keep the fuzzy request (default unless MISE_PIN=1)
mise use --file mise.local.toml node@22   # --file and --path are interchangeable (2026.8.1+)
mise use --postinstall 'corepack enable' node@22   # 2026.8.15+ set a per-tool postinstall
mise use --tool-option matching=oxlint github:oxc-project/oxc  # 2026.9.2+, repeatable
mise use --remove python    # Remove a tool from config
mise use --dry-run-code     # Exit 1 if changes would be made
mise install node@20        # Install without activation
mise install                # Install all configured tools
mise install -f "gem:*"     # Force reinstall pattern
mise install --monorepo     # Install across all monorepo config roots
mise install --include-task-tools   # Install every task's [tools] without running tasks (+ --monorepo)
mise install --shared <DIR> # Install into a shared directory
mise install --system       # Install to the system data dir. On Unix (2026.9.12+) mise downloads,
                            # verifies, and unpacks AS THE INVOKING USER, then uses sudo only to
                            # publish into the system install/shim dirs. Supports relocatable tools
                            # from aqua, github, gitlab, forgejo, http, s3 without a tool postinstall.
                            # system_packages.sudo = false disables elevation.
mise ls                     # List installed
mise ls --current           # Active versions only
mise ls --prunable          # Tools eligible for prune
mise ls --outdated          # Tools with newer versions
mise ls --monorepo          # Across monorepo config roots
mise ls -b aqua --grouped   # 2026.9.13: filter by backend (repeatable; works with --json) /
                            # one section per backend (not with --json)
mise ls-remote node         # List available versions
mise ls-remote --prerelease # Include pre-releases
mise ls-remote --all node   # Every version, ignoring filters; -J includes created_at
mise latest node            # Latest available version (no install)
mise which node             # Show real binary path (-t/--tool <TOOL@VER>, --plugin, --version)
mise where node@22          # Show install directory
mise bin-paths              # List active runtime bin paths
mise tool <TOOL>            # Backend/description/config-source/tool-options for one tool (--url: registry URL)
mise uninstall node@20      # Remove an installed tool version
mise unuse node@20          # Remove one version from config (keeps siblings)
mise link node@custom ./dir # Symlink an external install into mise
mise x python@3.12 -- script.py  # Run with specific tool
mise reshim                 # Rebuild shims
mise registry               # List all available tools
mise backends ls            # List available backends (`mise b` alias deprecated, removed 2027.4.0)
mise backends switch [-n] [-g] [TOOL@VER]  # 2026.9.13: move locked tools to the registry's new backend
mise fmt                    # Format mise.toml (sort keys, clean whitespace)
mise outdated               # Check for updates
mise upgrade                # Update versions (respects mise.toml ranges)
mise upgrade --bump         # Bump mise.toml to absolute latest (keeps precision)
mise upgrade -b             # 2026.8.6+ shorthand for --bump (old -l deprecated, removed 2027.8.5)
mise upgrade --no-prune     # Keep the replaced version indefinitely (2026.8.1+)
mise upgrade --prune        # Force IMMEDIATE removal, skipping the upgrade.prune_after grace period
mise upgrade -i             # Interactive selection
mise upgrade -x node        # --exclude a tool; also --inactive, --local
mise upgrade node@latest --bump   # 2026.9.13: persists the selector `latest`, not a concrete version
mise install --force        # 2026.8.4+ works with no tool args: reinstall every configured tool
mise lock --upgrade         # Migrate a lockfile to the newest revision (3)
mise prune --dry-run        # Explains WHY each version is prunable (2026.8.11+)
mise run --all              # Interactive picker across the whole monorepo (2026.8.6+)
mise prune                  # Remove unused versions (destructive; --configs also prunes stale links)
mise lock                   # Update lockfile checksums/URLs
mise search <query>         # Search registry (-m equal|contains|fuzzy; -a/--all; npm:/cargo:/gem:/dotnet: prefixes)
mise cache clear            # Clear cached downloads
mise cache task <task>      # Inspect a task's cached artifacts
mise sync node --nvm        # Import versions from nvm
mise sync python --uv       # Import versions from uv
mise sync ruby --brew       # Import versions from brew
mise token github           # Show resolved host token (--unmask to reveal)
mise generate install-script -w ./bin/mise  # Self-contained install script for CI
                            # renamed from `mise generate bootstrap` in 2026.8.11;
                            # old spelling is a hidden alias, removed 2027.9.0
mise generate task-stubs    # Generate task stub wrappers (bin/)
mise generate task-docs     # Generate markdown docs for tasks
mise generate config        # Generate sample config
mise generate github-action # Generate sample GitHub Action workflow
mise generate devcontainer  # Generate devcontainer spec
mise generate git-pre-commit # Generate pre-commit hook (staged files exposed as STAGED)
mise generate tool-stub     # Generate a standalone tool stub
mise install-into <tool> <path>  # Install a tool into a specific path
mise doctor                 # Diagnose installation issues (mise doctor path)
mise doctor project         # Run the project's [doctor.checks] diagnostics (--json)
mise self-update            # Update mise — newest stable release ≥24h old (--minimum-release-age <DUR>)
mise mcp                    # Run mise as a Model Context Protocol (MCP) server
mise bootstrap              # Provision a whole machine (alias: bs)
mise oci build|push|run     # Build/inspect OCI container images with mise tools
mise deps add|install|remove  # Project dependency preparation (alias: prepare)
mise en                     # Spawn a sub-shell with the project env loaded
mise test-tool <tool>       # Verify a tool installs/builds correctly
mise implode                # Remove mise entirely

# New command groups (2026.9.x)
mise daemons                # [experimental] Manage project daemons via pitchfork (alias: daemon)
mise dotfiles | mise dot    # Manage [dotfiles] + the whole history system
mise skills ls|sync         # Agent skills the active tools ship, from their packslips
mise packslip pins|forget   # Signers mise accepts packslips from
mise ssh [DEST] [-- CMD]    # SSH session, optionally borrowing read-only GitHub access
mise tool-stub <file>       # Execute a tool stub
mise install --include-lazy # Also provision tools declared lazy = true
mise reshim --system        # Rebuild system-scoped shims
mise lock --sidecars        # List dependency-graph sidecar dirs to commit (2026.9.18)
mise bootstrap packages export --format nix   # Emit a NixOS module from nix: declarations
mise bootstrap packages use --no-install      # Record declarations without touching managers
mise patrons | mise sponsors  # Project supporters
```

**`mise ssh` GitHub relay flags:** `--github-relay-read-only` (borrow read-only GitHub access for this session only), `--github-relay-repo <OWNER/REPO>` (repeatable allowlist), `--github-relay-all-repos`, `--github-relay-log-requests` / `--github-relay-no-log-requests`, `--github-relay-log-format <text|jsonl>`, `--github-relay-max-duration <DUR>` (`0s` = session lifetime). Tuned by the `github_relay.*` settings.

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

> **Changed behavior (2026.7.14):** values in **structured** files (JSON/YAML/TOML) are **literal by default** again. Previously, with `env_shell_expand` on, every structured value was shell-expanded — corrupting literals like bcrypt-style hashes and potentially pulling in matching process-environment values. Opt back in per file with `expand = true`, which also lets values reference vars defined earlier in the same file, an earlier file, or an earlier `[env]` block. A global `env_shell_expand = false` overrides `expand = true`.

> 🔴 **Dotenv parsing changed (2026.10.3).** mise replaced dotenvy with its own `mise-dotenv` parser (from dotenv-ng), used by both `_.file` and the `env_file` setting / `MISE_ENV_FILE`. Same-file references always expand, and **`${VAR}` now resolves against the file's own earlier assignments first**, then values loaded so far (with `expand = true`) or the process env — previously an ambient value (e.g. one `mise activate` exported from another `.env`) won. New operators: `${VAR:-default}`, `${VAR:+alt}`, `${VAR:?message}` (aborts with `required variable 'X' is not set: <message>`); `${VAR:=x}` stays **literal**. Multiline quoted values work. `_.file` still fails on a syntax error; the `env_file` setting keeps the assignments read before the error and warns once.

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

**PATH changes from a sourced script are prepend-only:** `export PATH="/new/bin:$PATH"` works (the original PATH must remain an exact suffix); appending, removing, reordering, or replacing PATH entries is **silently ignored**. Relative prepended entries resolve against `config_root`; empty entries are dropped.

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
3. `.miserc.toml` / `.miserc.local.toml`: `env = ["development"]`

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

**`.miserc.toml`** is loaded very early — before config discovery and before Settings. Lookup order (highest first): **in each directory from cwd upward** — `.miserc.local.toml`, `.miserc.toml`, `.config/miserc.toml` → `~/.config/mise/miserc.local.toml` → `~/.config/mise/miserc.toml` → `/etc/mise/miserc.toml`. `--no-config` skips miserc discovery too (2026.10.2).

**Personal env selection without editing a shared file** (2026.9.13 / 2026.9.17):
```toml
# .miserc.local.toml — per checkout (add to core.excludesFile; create per worktree)
env = ["native"]
```
```toml
# ~/.config/mise/miserc.local.toml — per machine, loads regardless of cwd
env = ["work"]
```
Explicitly set fields override the shared file at the same level; omitted fields inherit. `env` **replaces** the inherited list (`env = []` clears it). CLI `-E` and `MISE_ENV` still win over both.

Only **seven** settings are `.miserc`-settable:

| Key | Env var | Default |
|-----|---------|---------|
| `env` | `MISE_ENV` | `[]` |
| `auto_env` | `MISE_AUTO_ENV` | unset |
| `ceiling_paths` | `MISE_CEILING_PATHS` | `[]` |
| `ignored_config_paths` | `MISE_IGNORED_CONFIG_PATHS` | `[]` — **2026.8.9+** supports relative entries and globs (incl. recursive `**`). Entries in `.miserc.toml` resolve against the declaring file; `MISE_IGNORED_CONFIG_PATHS` resolves against the invocation directory. |
| `override_config_filenames` | `MISE_OVERRIDE_CONFIG_FILENAMES` | `[]` |
| `override_tool_versions_filenames` | `MISE_OVERRIDE_TOOL_VERSIONS_FILENAMES` | `[]` |
| `env_conf_d` | `MISE_ENV_CONF_D` | unset — opt into environment-specific dotted `conf.d` filenames now (see [File Hierarchy](#file-hierarchy)) |

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

Redaction requires a non-`raw` output mode — tasks with `raw = true` bypass interception. The default `prefix` (jobs > 1) and `interleave` (jobs = 1) styles already print full logs with redaction applied; only `replacing`/`timed` hide lines. A value supplied by the caller for a `{ required = true, redact = true }` var (or one overriding a redacted `default`) is still redacted.

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

> The external `sops` CLI has no TOML support. mise decrypts SOPS `.env.toml` only with the default `sops.rops = true` (the built-in engine, which supports **age only**). With `sops.rops = false` mise shells out to the `sops` CLI, which supports AWS KMS, GCP KMS, Azure Key Vault, Vault, and PGP — but encrypted TOML then fails.

**Secret hygiene (2026.10.3, security):**
- `__MISE_DIFF` / `__MISE_SESSION` — inherited by every child of an activated shell, task, and `mise x` — now store a `blake3:<hex>` **digest** per managed value instead of plaintext (the value itself is still in the child's env; low-entropy values are brute-forceable).
- **Secret-bearing environments are never written to the env cache:** any config with an `age` value, a sops-encrypted `_.file`, any directive with `redact = true`, or a plugin returning `redact = true`/`cacheable = false`. Settings-level `redactions` patterns alone do **not** mark the env uncacheable.
- `_.my-plugin = { redact = false }` now **overrides** the plugin's redact preference (and isn't passed to the plugin); a non-boolean `redact` is a config error.
- `mise x -- fish` no longer passes env values in fish's argv (visible in `ps`).

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
| `config_source` | String | **2026.8.15+.** Absolute path of the config file the template is written in (vs `config_root`, its directory) |
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
| `haiku([words], [separator], [digits])` | Random readable name — defaults `words=2`, `separator="-"`, `digits=2` ⇒ `fragrant-hummingbird-32`. **Undocumented upstream**; verified on 2026.8.12. Useful for ephemeral branch/preview/container names. |
| `read_file(path)` | Read file contents. Blocked under `safe = true`. |
| `range(end, [start], [step_by])` | Integer array |
| `get_random(start, end, [seed])` | Random integer (`seed` makes it reproducible) |
| `task_source_files([only_changed])` | Resolved source file paths (task scripts only); unmatched patterns omitted. **2026.8.15+:** `only_changed=true` narrows to files changed since the last successful run. |
| `throw(message)` | Raise error |

**Key Tera filters:**

| Filter | Description |
|--------|-------------|
| `lower`, `upper`, `capitalize`, `title` | Case transforms |
| `kebabcase`, `shoutykebabcase`, `snakecase`, `shoutysnakecase`, `lowercamelcase`, `uppercamelcase` | Case conversion (`shoutykebabcase` ⇒ `HELLO-WORLD`) |
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
tera_v1 = true    # escape hatch — WARNS NOW (since 2026.10.0), removed 2027.4.0
```

> In a shared `mise.toml`, prefer the env form `[env] MISE_TERA_V1 = true` — older mise versions treat it as a normal env var rather than erroring on an unknown setting.

> **v1 compatibility now warns (2026.10.x, verified).** Using a v1 helper prints e.g. `mise WARN deprecated [tera-v1-trim-start-matches]: … Use trim_start(pat=...) instead. This will be removed in mise 2027.4.0.` (likewise `[tera-v1-concat]`, `map`, …), and `tera_v1 = true` itself emits `deprecated [setting.tera_v1]`. Migrate now: `items | concat(with=extra)` → `[...items, ...extra]`.

**Tera components (v2 only)** replace v1 macros: define `{% component wrap(opt="none") %}…{% endcomponent %}` and call `{{ <wrap opt={vars.opt} /> }}`. Definitions must live in the **same** `run`/template string — there is no shared component library.

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

> **Hooks do not override across config files — they all run.** When the same hook type is defined in several loaded configs, mise runs **every** one, from highest-precedence config to lowest. Within one file, array-form hooks run in listed order. So hooks in `conf.d/a.toml`, `b.toml`, `c.toml` execute as **c, b, a** (later alphabetical fragments have higher precedence).

> `enter` does not re-fire when you `cd` **within** the same project, and `leave` does not fire on internal `cd` either.

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

# Task hooks — executed via `mise run`, so the full task system applies.
# Works with ALL hook types (cd, enter, leave, preinstall, postinstall).
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

> ⚠️ **A task used as a `preinstall` hook does NOT auto-install missing tools** — that would defeat the point of running ahead of the install it is preparing. Its commands must already be on PATH. Every other task-backed hook type keeps normal task tool-installation behavior.

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
- `MISE_CONFIG_ROOT` — root of the **config file that defines the hook** (distinct from the project root)

> For a hook in **global** config, both `MISE_PROJECT_ROOT` and `MISE_CONFIG_ROOT` are the global config root, and project hooks don't run.

CD/enter/leave hooks additionally receive:
- `MISE_PREVIOUS_DIR` — previous directory (only when a directory change occurred)

Config-level `postinstall` receives:
- `MISE_INSTALLED_TOOLS` — JSON array of `{name, version, requested_version, backend, install_path}` (the last two since **2026.9.13**), e.g.
  `[{"name":"node","version":"20.10.0","requested_version":"lts","backend":"core:node","install_path":"/home/u/.local/share/mise/installs/node/20.10.0"}]`.
  - `requested_version` is the **canonical pre-resolution selector** (`latest`, `20`, `lts`, `ref:main`), so a hook can branch without re-reading config.
  - `backend` is the canonical backend id **with options and URL credentials stripped** (they may carry registry secrets).
  - `install_path` is the concrete version dir — never a floating link like `latest` or `20`.
  - After a **partial failure** the array still lists the tools that did install.
  - It means "installation completed", not "config now selects it" — tool-level hooks can run before floating links are refreshed. **A postinstall hook is not an application-activation notification.**

> A `mise install` that finds nothing to install **still runs `postinstall`**, with `MISE_INSTALLED_TOOLS` set to `[]`. Guard accordingly.

> `preinstall`/`postinstall` run with the **project root as CWD** even when `mise install` was invoked from a subdirectory — the invocation dir is still in `MISE_ORIGINAL_CWD`.

Tool-level `postinstall` additionally receives:
- `MISE_TOOL_NAME` — tool identifier (e.g., `node`)
- `MISE_TOOL_VERSION` — installed version
- `MISE_TOOL_INSTALL_PATH` — installation directory
- `MISE_CONFIG_FILE` — the exact config file that declared the tool
- `MISE_CONFIG_ROOT` / `MISE_PROJECT_ROOT`
- plus that tool's `install_env` values

### Per-Tool postinstall (not a hook)

Runs immediately after each tool is installed, before other tools in the same session:

```toml
[tools]
node = { version = "22", postinstall = "corepack enable" }
python = { version = "3.12", postinstall = "pip install pipx" }

# 2026.9.17: table form — run on EVERY `mise install` that selects the tool, even if already installed
go = { version = "1.24", postinstall = { run = "go install golang.org/x/tools/gopls@latest", when = "always" } }
```

`when = "install"` (default, also the string form) runs only on a fresh install or repair; `when = "always"` re-runs on every explicit install (once per tool request; skipped on dry runs).

> **Hooks from a remote [`include`](#remote-config-includes-include) fragment** run **before** the including file's hooks of the same type.

---

## Project Daemons (`[daemons]`)

**Experimental** (requires `experimental = true`) and requires **pitchfork** for process supervision. Added in **2026.9.6** and substantially expanded in **2026.9.12**.

Daemons are processes that keep running *between* task invocations — a database, message broker, or dev server. mise supplies the project configuration and tool environment; pitchfork owns the processes and readiness checks.

```toml
[settings]
experimental = true

[tools]
node = "24"

[daemons]
postgres = "18"

[tasks.dev]
daemons = "postgres"
run = "npm run dev"
```

`mise run dev` installs missing tools, starts PostgreSQL, waits for readiness, then runs the task. The preset exports `DATABASE_URL` and friends and keeps data between runs. The daemon stays up after the task exits.

### Declaring Daemons

Exactly one declaration style per entry:

| Declaration | Meaning |
|-------------|---------|
| `postgres = "18"` | Service preset whose **name matches the key** |
| `preset = "postgres"` + `version = "18"` | Named instance of a preset, with overrides |
| `run = "exec npm run dev"` | Custom shell command |
| `task = "dev:core"` | An existing mise task (optionally with `args`) |
| `project = "../other"` (+ `name`) | A daemon declared by **another project** |
| `provider = "local-postgres"` (+ `resource`) | A database/account on a **shared server** from global `[daemon_providers]` (2026.9.13) — see [Shared Server Providers](#shared-server-providers-daemon_providers) |

```toml
[daemons.api]
run = "exec npm run dev"
ready_port = 3000

[daemons.analytics]
preset = "postgres"
version = "18"
port = 5433
```

| Field | Type | Notes |
|-------|------|-------|
| `run` | string | Long-running command. Use `exec` so it receives stop signals directly. Mutually exclusive with `task`/`preset`. Since **2026.10.1** it may use `{{ env.X }}` / `{{ vars.X }}` (rendered by `mise x` at launch; `[env]` values are never written into the generated pitchfork file; needs pitchfork > 2.29.0). Pitchfork's own `{{ port }}`/`{{ url }}` still work. |
| `task` | string | mise task to supervise. `args` requires it. mise is already the entry point, so it is not wrapped again unless `init` is also set. |
| `args` | string[] | Passed to the task after `--`. Requires `task`. |
| `preset` | enum | `cockroachdb` \| `nats` \| `postgres` \| `redis` \| `spicedb`. Requires `version`. |
| `version` | string | Version request for the preset's tool. |
| `init` | string \| string[] | Idempotent setup run **before** the long-running process, on every start and restart. Shares a shell with the main command. |
| `mise` | bool | `false` disables the mise tool-environment wrapper for both setup and the main command. |
| `port` | int \| `"auto"` \| table | Fixed port, or per-worktree allocation. Custom daemons also accept pitchfork's structured port table. |
| `ports` | table | Overrides for a preset's additional named ports. |
| `options` | table | Preset-specific options. |
| `data_dir` | string | Persistent preset data directory, relative to the declaring project root or absolute. Defaults to mise state storage. |
| `proxy` | string \| bool | Reverse-proxy hostname label; `false` opts out, `true` uses the daemon's name. Only a daemon with a port is routed. |
| `proxy_tls` | enum | `terminate` \| `passthrough`. |
| `proxy_idle_timeout` | duration \| `false` | **2026.9.18** (pitchfork ≥ 2.27.0). Stop a **proxy-started** daemon after this long without traffic (`"30m"`, `"1h"`). Explicit starts (`mise daemons start`, shell hook, `boot_start`) and their deps are exempt; dependencies without their own value inherit the requester's; `false` (or `"0"`) exempts even from a global default (`PITCHFORK_PROXY_IDLE_TIMEOUT`). Open connections count as activity; direct-port traffic does not. |

Readiness is configured with pitchfork's own fields (`ready_port`, `ready_cmd`, `auto`, …), which pass through verbatim. Readiness checks apply **after** `init`, so anything waiting on the daemon also waits for setup.

### Tasks That Require Daemons

```toml
[daemons]
postgres = "18"
redis = "8"

[tasks.test]
daemons = ["postgres", "redis"]
run = "npm test"
```

Use a string for one, a list for several, or `true` for every daemon in the task's project configuration. Already-running daemons are reused. This replaces prerequisite tasks that launch background processes and poll for readiness.

- Startup is part of **dependency handling**: `--skip-deps` and `task.skip_depends` skip it.
- `--dry-run` validates names and the experimental setting, and reports what would start without starting anything.
- **Safe mode blocks task daemon startup.**
- In a monorepo each task resolves daemon names in its own project's hierarchy, including inherited declarations. `true` covers only the project's own daemons — it never reaches into a referenced project.
- Inspect with `mise tasks info <task>`.

> ⚠️ **Subtasks do not start daemons.** The requirement is honored for tasks a run resolves up front (including their `depends`). A subtask reached through a `run = [{ task = "…" }]` entry is resolved once the run is already executing, so its own `daemons` are **not** started. Declare the requirement on the task you invoke.

> A daemon's `task` is invoked via `mise run`, so that task's `depends` run first — but its own `daemons` are skipped, since starting those would restart this daemon.

### Groups, Namespaces, and Cross-Project Daemons

```toml
[daemon_groups]
default = ["postgres", "nats", "core"]
two-cluster = ["default", "core2"]        # groups expand in place

# equivalent table form
[daemon_groups.two-cluster]
daemons = ["default", "core2"]
```

```bash
mise daemons start two-cluster
mise daemons stop --group two-cluster
```

Members must be daemons or groups **in the same project** — a group can never select outside it. A group name may not repeat a daemon name in the same project, and needs at least one member. `mise daemons start` with no names starts the `default` group when one exists, otherwise every project daemon; `stop` without names always covers every project daemon. A bare positional name selects that project's group first, then a daemon of that name; `--group` only ever selects a group.

```toml
[daemons_settings]
namespace = "services"            # fixed pitchfork namespace, else derived from dir name + path hash
namespace_per_worktree = true     # default; appends a 16-hex suffix in a linked git worktree
```

A daemon's full ID is `<namespace>/<name>`. Namespace resolution order: `[daemons_settings].namespace` → the project's `pitchfork.toml` → the generated default. Stop the project's daemons before changing its namespace. `[daemons_settings]` **merges per key** across config files (unlike `[daemons.<name>]`, where a higher-precedence declaration replaces the whole definition). **Global and system config cannot set `[daemons_settings]`** — mise ignores those tables with a warning.

```toml
# run a daemon defined in another checkout
[daemons.pipeline]
project = "../mirror-pipeline"
name = "worker"

[daemons.api]
run = "npm run dev"
depends = ["pipeline"]
```

The referenced checkout must already be **trusted** — `mise run` and `mise daemons start` never trust it for you. A reference table accepts only `project` and `name`, and cannot point at another reference. The imported daemon runs in *its own* project's environment, so a preset's `DATABASE_URL` does **not** join your application's environment. A missing or untrusted checkout drops the import rather than breaking the rest of your config; naming it explicitly fails with the reason.

### Shared Server Providers (`[daemon_providers]`)

**Experimental, 2026.9.13.** Instead of one PostgreSQL per checkout, run **one shared server per machine** and give each project its own database (or NATS account) on it. Providers are declared **only in global config**:

```toml
# ~/.config/mise/config.toml
[daemon_providers.local-postgres]
preset = "postgres"        # required: postgres | cockroachdb | nats  (no redis / spicedb)
version = "18"             # required
port = "auto"              # optional; also ports, options, data_dir, tool
```

Projects consume a provider — a provider reference accepts **only** `provider` and `resource`:

```toml
# project mise.toml
[daemons.db]
provider = "local-postgres"   # gets a checkout-specific database (derived from checkout path + daemon name)
# resource = "shared_app"     # ^[a-z][a-z0-9_]{0,62}$ — same name in several consumers = one shared DB
```

- Starting a consumer (`mise daemons start`, a task with `daemons = ["db"]`, or `depends = ["db"]`) waits for the server **and** the database to be provisioned. mise never runs migrations or deletes databases. The usual preset vars (`DATABASE_URL`, …) point at the selected database; explicit `[env]` wins.
- Providers run with their own tools and a **minimal env** (no project env/tools), are never idle-stopped, and aren't part of project groups. Data lives in `$MISE_STATE_DIR/daemon-providers/<name>/data`; pruning projects never deletes it. Change settings in global config, then `mise daemons providers restart <name>`.
- **NATS** providers give each resource its own **account** (separate subjects and JetStream). `NATS_URL` then embeds that account's username/password — treat it as a credential. Custom NATS config files and TLS are rejected for providers.
- A missing provider is an error (never auto-selected). Local superuser auth — **not a security boundary**.

### Service Presets

Presets supply the command, required tool, readiness check, data directory, and connection variables. Preset tools install on the first `mise daemons start` (2026.9.18). NATS and SpiceDB also need `curl` on PATH for their HTTP readiness checks.

> **Windows (2026.10.2):** every preset **except `redis`** (no Windows build) now runs on Windows under pitchfork's default `cmd /C` (a custom `windows_shell` is unsupported). PostgreSQL stops cleanly only with pitchfork ≥ 2.29.0 and refuses to run elevated. Hyper-V/WSL reserved port ranges (`netsh interface ipv4 show excludedportrange protocol=tcp`) may force an explicit `port`. Task daemons start **without a shell** since 2026.10.1 (pitchfork ≥ 2.28.0) so args arrive exactly — except a task daemon with `init`, which shares one shell (`cmd /C` on Windows).

> **PostgreSQL refuses to run as root (2026.9.13)** — starting a postgres daemon or provider as root fails before installing anything. In containers, run mise as a regular `USER`.

| Preset | Tool | Default port | Additional listeners | Exports |
|--------|------|--------------|----------------------|---------|
| `postgres` | `postgres` | 5432 | — | `PGHOST`, `PGPORT`, `PGUSER`, `PGDATABASE`, `DATABASE_URL` |
| `redis` | `redis` | 6379 | — | `REDIS_URL` |
| `cockroachdb` | `cockroach` | 26257 | `http_port` 8080 | `COCKROACH_HOST`, `COCKROACH_URL`, `DATABASE_URL` |
| `nats` | `nats-server` | 4222 | `monitor_port` 8222 | `NATS_URL`, `NATS_MONITORING_URL` |
| `spicedb` | `spicedb` | 50051 | `http_port` 8443, `metrics_port` 9090 | `SPICEDB_ENDPOINT`, `SPICEDB_PRESHARED_KEY` |

**Preset options** (`options.*`; file paths resolve relative to the project root, `~/` expands):

- **postgres** — `database` (created on first init; defaults to `postgres`). Uses the `postgres` user with local trust auth.
- **redis** — append-only persistence; no options.
- **cockroachdb** — `database` (`"defaultdb"`, names the connection default, does **not** create), `databases` (`[]`, created on first init; `name=region` sets a primary region), `settings` (`[]`, each an assignment like `"sql.defaults.vectorize = 'off'"`, prefixed with `SET CLUSTER SETTING`), `locality`, `max_offset`.
- **nats** — `jetstream` (`true`), `config` (a config file takes over JetStream/storage and voids `jetstream`), `tls_cert`/`tls_key` (both required to enable TLS; `NATS_URL` then uses `tls://`), `tls_ca` (requires the other two).
- **spicedb** — `datastore_engine` (`"memory"`), `datastore_uri`, `datastore_daemon` (startup order only — it does **not** make `datastore_uri` follow an automatic port), `preshared_key` (`"mise-dev-key"`). Setting only one of engine/uri is an error. `spicedb migrate head` runs before every start on a persistent datastore.

> 🔴 **Local development only.** Preset defaults use loopback addresses with no authentication or a fixed development key. For shared or untrusted environments, write a custom daemon with real auth and network restrictions.

Options used during **first initialization** (database names, cluster settings) do not modify existing data when changed. An explicit `[tools]` version must satisfy the preset's request (`postgres = "18.1"` satisfies a preset asking for `"18"`); multiple instances sharing a tool must use the same request. Explicit `[env]` values override preset defaults, and when several instances export the same variable the **last declaration wins**.

### Ports and Worktrees

```toml
[daemons.postgres]
preset = "postgres"
version = "18"
port = "auto"                                  # primary checkout keeps 5432

[daemons.api]
run = "exec npm run dev -- --port $API_PORT"
port = { auto = true, base = 3000 }            # base is REQUIRED for custom daemons
```

| Option | Meaning | Default |
|--------|---------|---------|
| `auto` | Enables automatic allocation. Must be `true`. | Required in table form |
| `base` | Port used by the primary checkout | The preset's default; **required for custom daemons** |
| `stride` | Spacing between allocation slots | `1` |

There are **511 worktree offsets**. With defaults, PostgreSQL uses 5433–5943 in linked worktrees and Redis 6380–6890. A preset's named ports move with its primary port; a named port you set yourself is used exactly as written. All configured ports must be free — **mise never searches for a free port** — and two daemons in one project cannot share a port (reported at config load). Resolved ports are saved in the project's generated `state.json` and reused; changing `base`/`stride` re-resolves, so restart the daemon.

Custom daemons with a port export `<NAME>_PORT` (uppercased, punctuation → underscores): `[daemons.api]` → `API_PORT`, `[daemons.web-ui]` → `WEB_UI_PORT`. Names that would collide (`web-ui` vs `web_ui`) or start with a digit get no variable and a warning — the daemon still runs, and pitchfork still injects `$PORT`. Independent clones and non-Git projects keep the base port; worktrees of a bare repo all get offsets.

**Preset named ports (2026.9.18)** export as `<NAME>_<PORT_NAME>`: a `cockroachdb` daemon named `crdb` exports `CRDB_HTTP_PORT`; a `spicedb` named `authz` exports `AUTHZ_HTTP_PORT` and `AUTHZ_METRICS_PORT` — the port actually used (default, worktree offset, or `ports.<name>`).

**Stable URLs per worktree:** every proxied daemon (has a `port`, not `proxy = false`) also exports **`<NAME>_URL`** — its hostname URL, which never changes when a port moves — so services can point at each other without port arithmetic:

```toml
[daemons.api]
run = "exec npm run dev -- --port $API_PORT"
port = { auto = true, base = 3000 }

[env]
APP_BASE_URL = "{{ env.API_URL }}"
```

Set `proxy = false` on any daemon that doesn't speak HTTP (the `postgres`/`redis` presets opt out already and keep exporting `PGPORT`/`DATABASE_URL`/`REDIS_URL`). When a port is taken, mise's warning names the daemon, the port, and whether it is the base or a worktree offset.

```bash
mise daemons                 # list configured + previously managed daemons
mise daemons ls --json       # includes resolved `port` and `port_auto`
mise daemons start [NAME…]   # installs missing tools
mise daemons start --all     # 2026.9.18: every project daemon (incl. ones a `default` group omits);
                             # never reaches other projects; not combinable with names/--group
mise daemons stop | restart | status | logs | tui
mise daemons providers ls [--json] | start | stop | restart [NAMES]…   # shared servers (2026.9.13)
mise daemons register        # prepare for on-demand startup without starting
mise daemons urls            # each daemon's port and proxy hostname URL
mise daemons prune           # drop state from deleted project directories
```

Hostnames are `<proxy>.<project>.<tld>` in the primary checkout and `<proxy>.<worktree>.<project>.<tld>` in a linked worktree.

---

## Project Diagnostics (`[doctor]`)

`mise doctor project` runs named checks from the project's configuration — for requirements tool versions alone cannot establish, like whether a compiler can find a native library or a database accepts connections.

```toml
[doctor.checks.openssl]
description = "OpenSSL development files are discoverable"
run = "pkg-config --exists openssl"
hint = "Run `mise bootstrap packages apply` to install the declared build dependencies."
timeout = "5s"
os = ["linux", "macos"]

[doctor.checks.database]
description = "PostgreSQL accepts connections"
run = "pg_isready --quiet"
```

```bash
mise doctor project
mise doctor project --json
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `run` | string | *required* | Diagnostic command. **Exit zero passes**; any other code fails. |
| `description` | string | the check name | Human-readable requirement |
| `hint` | string | — | Remediation shown on failure. **Never executed.** |
| `timeout` | duration | `10s` | Positive duration (`500ms`, `5s`) |
| `dir` | string | the declaring config's root | Relative paths resolve from that root; `~/` and absolute used as given |
| `shell` | string | mise's inline task shell | Includes the command flag, e.g. `"bash -c"`, `"pwsh -Command"` |
| `os` | string \| string[] | run everywhere | Same selectors/aliases as tool filters (`darwin`, `win`, `amd64`, `linux/arm64`). **`os = []` is rejected.** |

- Ordinary `mise doctor` diagnoses mise itself and does **not** run these. Checks never run on directory entry or before a task — there are no implicit checks.
- Checks run **concurrently**, bounded by `jobs` (`MISE_JOBS`); set `jobs = 1` for sequential. Results stay in name order and a failure doesn't stop the others.
- A higher-precedence definition **replaces the entire check** of the same name, including its hint and dir.
- Commands get the project's environment and tool paths. Diagnostics do **not** install tools or run task dependencies/hooks. Checks are **not sandboxed**.
- Each command has a deadline and a combined **64 KiB** stdout/stderr cap; output is captured and **discarded** so a probe can't print a credential into the report. Use `description`/`hint` to explain; run the command directly for detail.
- Templates are **not rendered** in `dir`. Checks that apply to the current platform report errors in safe mode. Empty or fully platform-excluded check lists still succeed.
- Unknown `[doctor]` container options are ignored for forward compatibility; **unknown check fields are rejected** to catch typos.
- JSON report has `checks` and `errors` arrays; config-loading errors populate top-level `errors` with `checks` empty.

> Plain `mise doctor` gained checks too: it warns when an installed plugin differs from its `[plugins]` URL/ref (2026.9.15) and when a dotfiles-history watcher is stale or outdated (2026.9.18 / 2026.10.0), with recovery advice.

---

## Command Wrappers (`[wrappers]`)

Added **2026.8.16**. Intercept an ordinary command name with a different command, arguments, and environment. Wrappers take precedence over mise-managed tools, and mise **strips its own dispatch directories before delegating**, so the underlying tool still resolves from mise or the system.

```toml
[wrappers]
# string shorthand
rg = "rg-wrapper"

# table form
[wrappers.cargo]
command = "mbx"
args = ["--shim"]                        # inserted BEFORE the intercepted command's arguments
env = { MBX_CARGO_SHIM_MODE = "1" }
```

| Field | Type | Description |
|-------|------|-------------|
| `command` | string | *Required* in table form. The command to run instead. |
| `args` | string[] | Arguments inserted **before** the intercepted command's own arguments |
| `env` | table | Environment variables set for the wrapper |

Works with normal activation, `mise activate --shims`, and `mise exec`, and (since 2026.9.15) through Windows exe/file shims. Managed wrapper shims are refreshed by `mise reshim`. Since **2026.9.13** a wrapper **installs a missing provider tool before dispatching** (lazy providers always; others when `not_found_auto_install` is on, the default). A remote [`include`](#remote-config-includes-include) fragment may carry `[wrappers]`; the including file's entry wins. There is **no dedicated doc page** — this key exists in the JSON schema and release notes only.

> Related: `activate_shims = false` (`MISE_ACTIVATE_SHIMS`, default `true`, added 2026.9.2) keeps tool shim directories off PATH during activation and hooks without changing auto-install or lazy-tool settings — which lets external command wrappers keep working.

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
- Always re-verifies provenance (SLSA, cosign, minisign, GitHub attestations) at install time, and confirms mise-versions "no attestations" answers against GitHub.
- Disables cross-worktree trust propagation.
- `--yes`, `MISE_YES=1`, and CI auto-confirmation **never approve config trust** (2026.9.17).
- A remote [`include`](#remote-config-includes-include) must be pinned by full commit SHA or OCI digest.

### Trust

Outside CI, untrusted configs must be approved with `mise trust` (`--all`, `--ignore`, `--show`, `--untrust`; plus the separate `mise untrust`).

- mise **auto-trusts** configs when it detects a CI environment — unless `MISE_PARANOID=1`.
- Since v2026.6.6, **safe** `mise.toml` files (no templates; only `min_version` and plain `[tools]`/`[tasks]` string values) auto-load without a trust prompt; anything with templates or richer constructs still requires trust. **Exception (2026.9.18):** a tool **key** containing `[` (inline options, e.g. `"github:cli/cli[api_url=…]"`) makes the file require trust — and since 2026.10.0 so does a `.tool-versions` entry with inline options (GHSA-wcqh-j26q-g44x).
- Since 2026.10.0, **declining** a trust prompt skips that config for the run instead of failing; since 2026.10.2, an untrusted project config no longer breaks shell activation — other trusted configs keep loading, with a warning.
- Since 2026.7.5, a config in a linked **git worktree** is auto-trusted if the equivalent path in the main checkout is trusted (one-way; `--ignore` still wins; excluded under paranoid mode). `mise trust --all` walks subdirectories, respecting `.gitignore` and skipping hidden dirs, `node_modules`, `vendor`, `target`, `dist`, `build`.
- Since 2026.8.9, `mise run`, naked `mise <task>`, `mise install`, `mise exec`, and `mise watch` **implicitly trust and persist** the active config in normal mode, removing a redundant prompt. Automatic `hook-env`/inspection commands still require explicit trust; paranoid and safe modes are unchanged.
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
mise bootstrap --skip-dirty        # 2026.8.13+ skip repos with uncommitted changes
mise bootstrap --adopt             # adopt an existing setup repo (`--from-git` was REMOVED in 2026.10.0)
mise bootstrap --from 'git::https://github.com/me/dotfiles.git?ref=v1'   # 2026.9.18: ?ref= pins a
                                   # branch/tag/commit; `git::` optional; --update re-resolves the ref
mise bootstrap unapply ssh --dry-run   # 2026.9.13: remove what a deselected module set up (destructive)
mise bootstrap plan [--json] [--detailed-exitcode]  # 0 = no changes, 2 = changes, 1 = failure
mise bootstrap status [--json] [--missing]          # non-zero exit when out of sync
mise bootstrap remote [TARGET]…   # apply config to inventory hosts / SSH destinations
```

**Declarative resource model (2026.8.2+).** Bootstrap converged into a Terraform-style plan/apply system. Resources have stable identities and a dependency graph, and are validated for duplicates, missing dependencies, and cycles. Each resource type also has its own `apply`/`status` pair that converges only when something actually differs:

```bash
mise bootstrap plan --json --detailed-exitcode   # 0 = no changes, 2 = changes, 1 = error
mise bootstrap files apply
mise bootstrap accounts status
mise bootstrap services apply
mise bootstrap compose apply
mise bootstrap firewall apply
mise bootstrap secrets status      # reports availability without revealing values
mise bootstrap packages prune --manager brew-cask
```

Subcommands: `accounts`, `compose`, `dotfiles`, `files`, `firewall`, `linux`, `macos`, `mise-shell-activate` (alias `shell`), `packages` (incl. `where`), `plan`, `plugins`, `remote`, `repos`, `secrets`, `services` (incl. `remove`), `status` (alias `ls`), **`unapply`** (2026.9.13), `user`.

**Modules and `mise bootstrap unapply` (2026.9.13).** A *module* is a configuration environment — `~/.config/mise/config.<name>.toml` (or `mise.<name>.toml` in a bootstrap project) — selected per machine via `miserc.toml` `env = ["ssh", "gpg"]` or `mise -E ssh,gpg bootstrap` (per-host `mise_env` for remotes). Later envs in the list win for the same key. To retire one: remove it from `env`, **keep its file on disk**, then:

```bash
mise bootstrap unapply ssh --dry-run
mise bootstrap unapply ssh gpg --yes
```

Removal is planned from the current config *with vs. without* the named envs (not from run history): managed files, directories, user services, and dotfile entries/edits it contributed are removed, unless still declared elsewhere (an `absent` declaration does not protect). Changed targets are skipped unless `--force`; dirs are removed only if empty. Packages, repos, and Compose projects need separate cleanup (the output says how); system services are out of scope. `mise bootstrap plan --json` reports `origin.config` / `origin.environment` per resource.

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
# ⚠️ config_roots is DEPRECATED (2026.9.4; hidden from help, removal 2027.3.3) — emits a warning.
# Migrate each bundle to a conf.d FOLDER fragment (~/.config/mise/conf.d/<name>/mise.toml, 2026.9.14),
# which is its own config root and keeps the module's files beside it.
[bootstrap]
config_roots = ["bundles/*"]
dotfile_groups = ["home", "zsh"]   # 2026.10.3: which [dotfile_groups] apply on THIS machine
                                   # (unset = all groups; a more-local list replaces others)

# System packages — managers: apk: apt: aur: brew: brew-cask: dnf: flatpak: flatpak-user:
#   macos-app: mas: nix: pacman: scoop: winget: zypper:
[bootstrap.packages]
"apt:build-essential" = "latest"
"apk:curl" = "*"                    # Alpine apk (version: "@2.45.2-r0" form)
"brew:postgresql@17" = "latest"     # resolves formula aliases such as `openssl`
"brew-cask:firefox" = "latest"      # app-bundle casks, no local Homebrew required
"brew-cask:1password" = { version = "latest", appdir = "/Applications" }   # 2026.10.0 per-cask appdir
                                    # (overrides MISE_BREW_CASK_OPT_APPDIR; install/upgrade only — never
                                    #  moves existing apps; first install won't replace an app unless adopt = true)
"flatpak:org.gimp.GIMP" = "latest"  # system-scoped
"flatpak-user:org.gimp.GIMP" = "latest"   # 2026.8.3+ per-user scope; both may coexist
"mas:497799835" = "latest"          # Mac App Store apps by ADAM ID
"aur:yay-bin" = "latest"            # 2026.9.2+ Arch User Repository via yay (preferred) or paru
"winget:Microsoft.PowerShell" = "latest"  # 2026.9.3+ Windows, exact package ID
"nix:ripgrep" = "latest"            # 2026.9.4+ Linux/macOS, applied through your Nix profile
"scoop:extras/vscode" = "latest"    # 2026.9.12+ Windows, adds the bucket if missing
"zypper:gcc" = "latest"             # 2026.9.12+ openSUSE / SLE

# 2026.9.11+ macOS .app from a vendor/internal download.
# version, url, sha256, artifact are ALL required — "latest" is rejected.
[bootstrap.packages."macos-app:example"]
version = "1.2.3"
url = "https://example.com/Example-{{version}}.dmg"
sha256 = "…"
artifact = "Example.app"

# Table form with version + [tools]-style os selectors (2026.8.4+).
# Platform-incompatible packages surface as unavailable instead of aborting the run.
"brew-cask:font-hack-nerd-font" = { version = "latest", os = ["macos", "linux"] }

# 2026.9.4+ `env` selector gates an entry on the active mise environment
"apt:postgresql-client" = { version = "latest", env = ["ci", "staging"] }
"apt:build-essential" = { version = "latest", state = "absent" }   # declare removal

[bootstrap.brew]
adopt = true                        # adopt existing app bundles globally (per-cask: adopt = true)
taps = { "homebrew/cask-fonts" = "https://github.com/homebrew/homebrew-cask-fonts" }

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
phase = "pre-packages"   # 2026.9.5+ — converge this file BEFORE packages install
                         # (also valid on [bootstrap.directories])

# 2026.9.13+: targets may start with ~/ (a file you own in $HOME needs no sudo)
[bootstrap.files."~/.config/app/conf.ini"]
source = "files/app.ini.tmpl"
template = true
remove_empty = true      # an empty/whitespace render REMOVES the target (only with template = true)

# Permissions-only: omit source/content, declare mode/owner/group — manages just those, in place
[bootstrap.files."/etc/ssl/private/site.key"]
mode = "0600"
owner = "root"

[bootstrap.files."~/.oldrc"]
state = "absent"

# Linux accounts — converge BEFORE the files that reference them; UID/GID collisions fail closed
[bootstrap.users.deploy]
groups = ["docker", "www-data"]
state = "present"                   # or "absent"

[bootstrap.groups.deploy]
state = "present"

# Services — running/stopped, enabled/disabled, masked
[bootstrap.services.nginx]
state = "running"
enabled = true
# full field set: builtin, command, description, enabled, environment, masked,
#                 on_change, requires_tools, restart, scope, state, working_directory

# `builtin` selects a mise-provided service — e.g. the dotfiles history watcher
[bootstrap.services.mise-history]
builtin = "history-watch"

# Docker Compose projects (Compose v2 only) — convergence compares live container
# runtime and health against the rendered Compose model.
# 2026.10.3+: text fields are Tera-rendered in the declaring config's context
# ({{ config_root }}, {{ env.X }}); enum fields must be literal; exec() is unavailable.
[bootstrap.compose.observability]
project_dir = "{{ config_root }}/compose"   # REQUIRED, absolute after rendering (there is no `path` key)
files = ["docker/observability.yml"]
env_files = ["{{ config_root }}/compose/.env"]
state = "running"                   # running | stopped | absent
pull = "missing"                    # always | missing | never
wait = true
# also: project_name, profiles, services, oneshot, build (auto|always|never), recreate,
#       wait_timeout, timeout, remove_orphans (true), renew_anonymous_volumes, down_volumes,
#       down_images (local|all), sudo, command, engine_command, depends_on

# Linux firewall — nftables / firewalld / UFW
[bootstrap.linux.firewall]
backend = "auto"
state = "enabled"
default_incoming = "deny"
default_outgoing = "allow"
# allow_lockout = true              # required to set default-deny with no covering allow rule
# exclusive = true                  # remove undeclared rules (default: preserve them)

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
apps = ["/Applications/Ghostty.app", "/Applications/Firefox.app"]   # 2026.9.6+

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

# 2026.9.5+ superset of [bootstrap.macos.defaults] — adds host/path targeting
[[bootstrap.macos.defaults_entries]]
domain = "com.apple.finder"
key = "AppleShowAllFiles"
value = true
# host = "currentHost"
# path = "~/Library/Preferences/com.example.plist"

# Services (launchd on macOS, systemd on Linux)
[bootstrap.macos.launchd.agents.my-sync]
program = "~/.local/bin/my-sync"
args = ["--watch"]
run_at_load = true
start_calendar_interval = { hour = 3, minute = 30 }
process_type = "Background"   # 2026.9.12+ → launchd ProcessType:
                              # Background | Standard | Adaptive | Interactive
                              # misspellings are REJECTED at config time

[bootstrap.linux.systemd.units.my-sync]
description = "sync files"
exec_start = "~/.local/bin/my-sync --watch"
exec_start_pre = ["-~/bin/check"]    # 2026.9.13: also exec_start_post, exec_stop_post (service-only lists;
                                     # ~ expansion keeps systemd prefixes like "-")
part_of = ["graphical-session.target"]   # 2026.9.13: before, binds_to, part_of, conflicts ([Unit] lists)
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

> **macOS sandboxed apps (2026.9.15):** if `~/Library/Containers/<domain>` exists, defaults are read/written in the container plist (and its `ByHost`). Launch the app once first; the terminal may need Full Disk Access.

> **Shared setup repos:** `mise bootstrap --adopt` runs the shared `bootstrap` task after restoring files, but later `mise dot pull` / the history watcher restore files **without** running setup — run `mise bootstrap` again (`mise dot status` reminds you until a complete bootstrap has run).

Declarative steps converge idempotently; the `bootstrap` **task runs every invocation** (write it to be idempotent). Hooks stop bootstrap on failure and run in the current process environment. Unpinned `[bootstrap.repos]` entries are never pulled by a plain apply — use `mise bootstrap repos update` for the explicit fetch + fast-forward. Switching a systemd unit name between service and timer stops/disables/removes the stale sibling.

### Remote Bootstrap over SSH (`[bootstrap.remote]`)

Added **2026.8.2**, extended through 2026.8.11. mise archives and stages your project on each target, provisions a compatible mise binary, runs bootstrap with forwarded flags, then cleans up staging.

```toml
[bootstrap.remote]
source = "."                        # local directory archived and sent to each target
mise_env = ["production"]           # config environments to load remotely (2026.8.10+)
install_mise = true                 # persist mise on the host (default ~/.local/bin/mise)
copy_links = false                  # dereference every symlink in the archive
copy_link = ["config/secrets"]      # or dereference selected source-relative links
exclude = ["node_modules", ".git"]

[bootstrap.remote.hosts.web1]
host = "web1.internal"
user = "deploy"
port = 22
identity_file = "~/.ssh/id_ed25519"
tags = ["web", "prod"]
ssh_options = ["StrictHostKeyChecking=yes"]
# per-host overrides: source, mise_env, copy_links, copy_link, exclude,
#                     install_mise, mise_bin, remote_mise, bootstrap_command
```

```bash
mise bootstrap remote web1 db1          # named inventory hosts
mise bootstrap remote --all
mise bootstrap remote --tag prod        # select by tag (repeatable, matches any)
mise bootstrap remote --host deploy@10.0.0.5   # ad-hoc target
mise bootstrap remote -n                # dry-run
mise bootstrap remote --remote-env ci,dotfiles
mise bootstrap remote --install-mise=/usr/local/bin/mise
mise bootstrap remote --only packages,dotfiles --fail-fast
mise bootstrap remote --prompt-secrets --keep-staging
```

**Binary provisioning is fail-closed.** mise detects each target's OS, architecture, and Linux libc (glibc vs musl); when the local binary is incompatible it downloads the matching raw executable for the *same release* from GitHub with **minisign-verified checksums**. Custom or debug builds refuse to guess and require an explicit `mise_bin`, `remote_mise`, or `bootstrap_command`.

`--only` / `--skip` accept the same part names as local bootstrap. Other flags: `--connect-timeout`, `--force-dotfiles`, `--update`, `-y/--yes`, `-i/--identity-file`, `--port`, `--ssh-option`, `--bootstrap-command`.

### Declarative Dotfiles (`[dotfiles]`)

Manage dotfiles declaratively; applied during `mise bootstrap` (step 8) or standalone via `mise bootstrap dotfiles apply`. **Stable since v2026.7.4.** Entries are keyed by target path.

> 🔴 **Un-deprecated (2026.9.8) — reversal of the 2026.7.16 note.** The top-level **`mise dotfiles` command is first-class again**, with the short alias **`mise dot`**. Re-verified on 2026.10.3: it is visible in `mise --help`, has no `deprecated_at!` in the source, emits **no deprecation warning**, and exposes the same subcommand set as `mise bootstrap dotfiles` — plus the whole history system (`save`, `history`, `rollback`, `undo`, `track`, `sync`, `origin`, `watch`, `conflicts`, `capture`, `recover`, `paths`, `pull`, `exclude`, `include`). mise's own output now recommends `mise dot save` and `mise dot origin set`. Both spellings work; **prefer `mise dot`** for history operations and either for apply/status.

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
"~/.config/small.conf" = { content = "key = value\n" }   # 2026.8.6+ inline body;
                                                  # cannot combine with source/mode/exclude
# Block / line edits to files mise does NOT own (key = target/edit-id)
"~/.zshrc/activate" = { block = 'eval "$(mise activate zsh)"' }
"/etc/hosts/dev" = { line = "127.0.0.1 dev.local", position = "prepend" }  # append (default) | prepend
"~/.gitconfig/identity" = { source = "snippets/git-identity.tmpl", template = "tera" }

# Encrypt tracked contents in Git; the LIVE file stays plaintext
"~/.config/app/token" = { source = "dotfiles/token", encrypt = true }

# Select managed files from a directory via its git manifest (copy / symlink-each only)
"~/.config/nvim" = { source = "dotfiles/nvim", mode = "symlink-each", manifest = "git" }

# 2026.9.13+: delete a file on every machine that applied the config (file or symlink only, no globs)
"~/.oldrc" = { mode = "absent" }
# 2026.9.13+: permissions on copies/templates/inline content — or ALONE, managing only an existing target's mode
"~/.netrc" = { source = "netrc", mode = "copy", permissions = "0600" }
"~/.ssh" = { permissions = "0700" }
# 2026.9.13+: remove the target when the template renders empty (only what mise last wrote)
"~/.config/work.env" = { source = "work.env.tmpl", mode = "template", remove_empty = true }
# 2026.9.14+: GNU Stow style — dot-<name> sources deploy as .<name>; relative symlinks
"~" = { source = "home", mode = "symlink-each", dot_prefix = true, relative = true, exclude = ["README.md"] }
# Track mode: select what is captured (2026.9.13) — explicit exclude always wins
"~/.codex" = { mode = "track", include = ["config.toml", "rules/**"], exclude = ["rules/tmp/**"] }

# Per-OS / per-profile variants
[[dotfiles."~/.gitconfig".variants]]
os = ["macos"]
target = "~/.gitconfig-mac"
[[dotfiles."~/.gitconfig".variants]]
profile = "work"
default = true
```

**Entry fields:** `source` · `content` · `mode` (`symlink` \| `symlink-each` \| `copy` \| `template` \| `track` \| **`absent`**) · `exclude` · `manifest` (`"git"`) · `block` · `line` · `position` (`append` \| `prepend`) · `template` (`"tera"`) · `comment` · `encrypt` · `variants` (each `{ os?, profile?, default?, target? }`) · **`permissions`** (octal string `"0600"`) · **`remove_empty`** (template mode) · **`relative`** (symlink modes) · **`dot_prefix`** (directory source + `symlink-each`/`copy`) · **`include`** / **`allow_plaintext`** (track only).

| New field (2026.9.13–9.16) | Rules |
|----------------------------|-------|
| `mode = "absent"` | Deletes a regular file or symlink (never follows the link; never removes a dir/socket/FIFO — errors even with `--force`). No `source`/`content`/edit keys, no globs. Works with destination `variants` (e.g. remove only on macOS). Status `absent` once gone, `would remove` while present. `mise dot unapply` skips it; `mise oci build` adds a whiteout. **This is the way to clean up a file you stopped managing** — merely deleting the entry leaves it in place. |
| `permissions` | Works with `copy`, `template` (file source), inline `content`. **On its own** it manages only an existing target's permissions (missing target → warning, status `applied` with reason `target absent; permissions not applied`). Not with `symlink`/`symlink-each`/`track`/directory sources; ignored on Windows. |
| `remove_empty` | Template mode only. An empty/whitespace render removes the target if it still holds what mise last wrote (else conflict; `--force` overrides), plus now-empty parent dirs mise created inside `$HOME`. |
| `relative` | Relative symlink targets; requires `symlink`/`symlink-each`; overrides the `dotfiles.relative_symlinks` setting (default `false`). Ignored on Windows (junctions). |
| `dot_prefix` | Stow `--dotfiles`: source components `dot-<name>` deploy as `.<name>`. `exclude`/`manifest` match **source** names. A `dot-bashrc` + `.bashrc` collision fails. `mise dot add` refuses to capture into a `dot_prefix` entry (groups accept). |
| `include` (track) | Only matching paths are captured; `include = []` selects nothing. Selects credential-named files too (plaintext unless `encrypt = true`). New checkpoints use history schema v2, so **upgrade every machine** sharing the history first. |
| `allow_plaintext` (track) | Written by `mise dot track --allow-plaintext` — lets a directly tracked credential-named file be stored in plaintext. `--yes` does **not** imply it. |

**Pattern matching (2026.9.15):** a leading `/` anchors a pattern to the entry root like `.gitignore` (`"/cache"` = top level only); `**` crosses directories; in `include` patterns `*` **never** crosses `/`. In `exclude` patterns containing `/`, `*` still crosses `/` but **warns** (deprecated 2026.9.13, removal **2027.9.13**) — write `**` instead.

- `encrypt = true` stores encrypted contents in Git while the live file stays plaintext. Not combinable with `content`/`block`/`line`/`template`. Shared recipients come from `[history].encryption.recipients`.
- `variants` are history **streams** in track mode, or optional **destinations** in deployment modes. A variant `target` needs an explicit `source` (or a safe relative entry key when every variant has a target) and is **not supported in track mode**.
- Templates (`mode = "template"`) can reference `[bootstrap.secrets]` via `{{ secret(name="…") }}` (2026.9.7+). Commands that render templates (`add`, `apply`, `diff`, `edit`, `status`, `unapply`) accept `--prompt-secrets`; without a value, rendering **fails closed**. Textual diffs redact resolved secret values.

> 🔒 `mise oci build` renders dotfile templates with a **restricted engine**: `secret()` is rejected and the `env` context, `get_env()`, `exec()`, and `read_file()` are unavailable, so ambient credentials cannot be baked into a publishable image layer.

**Modes:** `symlink` (default; one link for a file or whole directory) · `symlink-each` (directory source → per-file links, so the target dir can also hold unmanaged files; supports `exclude` globs, prunes stale mise-managed links, and records exact source→target pairs in a manifest under `$MISE_STATE_DIR/dotfiles` rather than recursively walking shared targets like `~`) · `copy` (a real file/dir; additive for directories and **never pruned**) · `template` (render the source through mise's Tera engine with `env`, `vars`, `exec()`; permissions mirror the source and are repaired on drift).

**Source resolution:** omitting `source` mirrors the home-relative target path under `dotfiles.root` (`~/.zshrc` → `~/.dotfiles/.zshrc`). Relative explicit sources resolve against the declaring config file's directory. Targets outside `$HOME` require an explicit `source`.

**Block edits** wrap content in marker comments (`# >>> mise:id >>>` … `# <<< mise:id <<<`); the comment style is inferred per file type (`#`, `--`, `//`, `;`, `"`) and can be overridden with `comment`. Strict JSON/XML cannot use blocks. **Line edits** append a single line if absent and never modify other content. Edit IDs allow letters, digits, `_`, `-`, `.`. A table with `source` + `template = "tera"` is unambiguously an edit; a table with only `source` is a whole-file entry.

```bash
mise dot status [--missing] [--json]        # applied/missing/differs/absent/orphaned (alias: ls); JSON
                                            # adds `reason` on differs + permission-only entries
mise dot apply [--dry-run] [--verbose] [--yes] [--force]
mise dot apply --prune                      # 2026.10.3: also remove files from deselected/deleted groups
mise dot add ~/.zshrc [--no-apply]          # capture a live file (applies by default)
mise dot add ~/.config/starship.toml --group home   # 2026.10.3: capture into a dotfile group
mise dot diff                               # changes needed to apply
mise dot edit [--apply] [--group NAME] ~/.zshrc
mise dot unapply [--group NAME]             # remove managed links/copies/templates/blocks (one group)
mise dot conflicts [PATH…] [--difftool]     # 2026.9.7+ inspect local vs remote before resolving
mise dot recover                            # recover an interrupted dotfile operation
mise dot capture                            # capture live changes back into sources
```

`mise dot apply` and the bootstrap dotfiles phase also run `[history.reload]` commands (2026.9.13).

`mise bootstrap dotfiles <same subcommand>` is equivalent; both run the `pre-dotfiles`/`post-dotfiles` hooks.

`conflicts` is read-only — a unified diff including file-mode changes by default, `--difftool` opens the configured Git `diff.tool` (falling back to `merge.tool`), `--tool <name>` picks one. Encrypted contents decrypt only into private temporary files, and inspection never modifies either side or marks the conflict resolved. Resolve with `--take-remote` or `--keep-local`.

Dotfiles are **manual-only** — never applied implicitly by `mise install` or `mise bootstrap packages`. A regular file whose content already matches its source converges to a symlink without `--force`; a genuine conflict needs `--force`. `--dry-run` promises to execute nothing, so it **skips template rendering** and lists those entries as `(if changed)`. On Windows, file symlinks fall back to copies (directories use junctions). Removing a config entry leaves files in place — replace it with `mode = "absent"` (or `[bootstrap.files."~/x"] state = "absent"`) to delete it on every machine, or run `unapply` locally.

### Dotfile Groups (`[dotfile_groups]`)

Added **2026.10.3**. Stow-package-style **named directory trees** under `dotfiles.root`, each deployed as a unit and selectable per machine:

```toml
[dotfile_groups.zsh]
root = "zsh"                 # REQUIRED; relative = under dotfiles.root (~/.dotfiles), NOT the config dir

[dotfile_groups.home]
root = "home"
target = "~"                 # default "~"; absolute or ~/
mode = "symlink-each"        # default; symlink-each | copy | symlink (whole tree)
dot_prefix = true            # default false
exclude = ["README.md"]
# manifest = "git"           # only files in Git's index
# relative = true            # overrides dotfiles.relative_symlinks

[dotfile_groups.home.entries]               # whole-file entries cut out of the walk
"~/.config/kitty" = { mode = "symlink" }    # link a directory as a whole
"~/.ssh/config" = { mode = "copy", permissions = "0600" }
"~/.gitconfig" = { source = "git/config.tmpl", mode = "template" }   # relative source starts at the group root
"~/.kitty-old.conf" = { mode = "absent" }

# Per machine, e.g. ~/.config/mise/config.local.toml
[bootstrap]
dotfile_groups = ["home", "zsh"]   # unset = ALL groups apply; [dotfiles] entries always apply
```

- Group names match `^[A-Za-z0-9_.-]+$`; a more-local config with the same group name **replaces** the whole group table.
- Two selected groups may not deploy the same file, nor may one link a directory whole while another places files in it — the conflict is reported before anything is written.
- Entries without `source` find it under the group root at the same relative path; entries without `mode` deploy like the group; walking entries inherit `dot_prefix`, `manifest`, and slash-less `exclude`s. Every entry must lie inside the group's `target`.
- **Deselecting or deleting a group leaves its files** — mise records deployments under `$MISE_STATE_DIR/dotfiles/groups` and `mise dot status` lists them as **`orphaned`**. Clean up with `mise dot apply --prune` (every group; asks unless `--yes`) or `mise dot unapply --group <name>`. Both only remove links still pointing at the source / copies still holding what mise wrote (`--force` for changed copies), and never remove through linked dirs or inside `dotfiles.root`.
- `mise dot add <file>` captures into the deepest containing group (choose with `--group` for a new file); `manifest = "git"` groups refuse `add`.
- `mise oci build` does **not** include dotfile groups.

### Dotfiles History (`mise dot` / `[history]`)

mise saves versions of tracked configuration files as **ordinary Git commits** ("checkpoints") so you can inspect changes and restore an earlier version. History stays **local** until you connect a remote setup repository.

```bash
mise dot track ~/.zshrc                       # tracks and saves the first version immediately
mise dot track ~/.config/app/state.json --no-autosave
mise dot save ~/.zshrc                        # save one file
mise dot save --description "before theme change"   # save all tracked files
mise dot untrack ~/.zshrc
mise dot paths [--noisy]                      # what history tracks, and under which policies
mise dot track --dry-run ~/.config/app        # 2026.9.13: preview files/size/what is left out
mise dot paths --preview ~/.config/app        # same preview; warns above 5,000 files or 256 MiB
mise dot track --allow-plaintext ~/.netrc     # 2026.9.16: store a credential-named file unencrypted
```

**Credential filtering is by filename, not contents:** `.netrc`, `*.age`, `*.key`, `*.pem`, `*.gpg`, `*.kdbx`, `id_*` (including `id_ed25519.pub`), `*token*`, `*secret*`, `credentials*`, `oauth*`, and under the mise config dir `github_tokens.toml`, `hosts.yml`, `age.txt`. `*.local.toml` is **always** omitted, even with encryption. `mise dot paths` lists omissions with reasons.

**Nested repositories (2026.9.13):** a directory containing `.git` inside a tracked directory is skipped and reported — track its root explicitly to save its working files (`.git` is always excluded).

An ordinary `save` with no changes creates no commit; supplying `--description`, `--label`, or `--task` creates a checkpoint anyway. `save` **fails** if it can't save anything (Git missing, history disabled, path untracked) — use `--best-effort` in scripts to warn and continue.

**The watcher** is a background service saving changes made by any editor or command — a systemd user service on Linux, a LaunchAgent on macOS, a Scheduled Task on Windows:

```toml
[bootstrap.services.mise-history]
builtin = "history-watch"
```

```bash
mise bootstrap services apply
mise dot status
mise dot watch --once
```

**Comparing and restoring:**

```bash
mise dot history [--path ~/.zshrc]            # checkpoints, newest first
mise dot history show 12 [--files] [--json]
mise dot history diff 12 --patch --path ~/.zshrc
mise dot history diff 11 12 --patch           # between two checkpoints
mise dot history diff --path ~/.zshrc         # current file vs latest saved (add --exit-code)
mise dot rollback ~/.zshrc [--dry-run]
mise dot rollback --to latest~3 --all --dry-run
mise dot undo                                 # reverse the tracked-file changes of an operation
```

| Reference | Meaning |
|-----------|---------|
| `12` | Checkpoint with local ID 12 |
| `latest` | Most recent checkpoint |
| `latest~N` | N checkpoints back (with `--path`, counts only checkpoints where that path changed) |
| `commit:<sha>` | Git commit by full hash or unambiguous prefix |

Numeric IDs are **local and may change** if mise rebuilds its index — use a Git commit hash for a stable reference. Rollback saves current contents first, so `undo` can reverse it; both create new commits, so earlier versions stay available. `--all` selects everything in the chosen checkpoint, and a checkpoint that recorded a file as **absent** makes rollback **delete** it. Preview actions are `write` (replace with different saved contents) and `delete`.

**Sharing** connects a setup repository:

```bash
mise dot origin set <url>
mise dot sync                                  # publish, fetch, record what is pending
mise dot pull                                  # pull incoming shared changes into live files
```

**`[history]` configuration:**

```toml
[history]
exclude = ["**/*.log", "!important.log"]       # last matching rule wins; a !glob re-includes inside an
                                               # excluded dir — but cannot override a per-entry `exclude`
git_email = "mise@{hostname}"                  # 2026.9.17: author/committer for saves, autosaves, merges
                                               # ({hostname} expands per machine; default mise@localhost)

[history.encryption]
recipients = ["age1…", "ssh-ed25519 AAAA…"]    # shared age/SSH/age-plugin recipients

[history.reload]
"~/.zshrc" = "exec zsh"                        # run once after a rollback/undo writes a matching path
                                               # (trusted GLOBAL or SYSTEM config only)

[history.origin]                               # machine-local; written by `mise dot origin set`
url = "git@github.com:me/setup.git"
branch = "main"
```

**`[settings.history]`:**

| Setting | Default | Description |
|---------|---------|-------------|
| `enabled` | `true` | Record Git commits for tracked files around bootstrap operations and on explicit saves |
| `sync` | `"sync"` | What the watcher does with a connected setup repo on its own |
| `sync_interval` | `"5m"` | How soon after a save the watcher publishes, at most this often |
| `fetch_interval` | `"15m"` | How often the watcher fetches the origin branch (values below 1s use 1s) |
| `notify` | `true` | Desktop notification when conflicts pause sharing (retries stay silent) |
| `allow_plaintext_history` | `false` | Allow publishing history containing unencrypted versions of files now marked for encryption |
| `describe_command` | `""` | Command that names commits the watcher saves (an agent, say); gets one JSON object on stdin |
| `watch.debounce` | `"2s"` | Quiet period before the watcher saves a changed file |
| `watch.max_interval` | `"24h"` | Longest autosave interval a constantly-changing file is stretched to |
| `watch.reconcile` | `"10m"` | Full rescan for missed changes (`0` disables periodic reconciliation) |

Ordinary edits are saved after ~2s of quiet; constantly-changing files are stretched on their own schedule (doubling up to `max_interval`) and never delay the others. Explicit `save` is always immediate.

> 🔒 **`history.describe_command` is global-only (2026.9.7 security fix).** An implicitly trusted project could previously set it and have a later checkpoint execute the project-controlled command with **unencrypted tracked-file diffs**. It is now honored only from system/global config or `MISE_HISTORY_DESCRIBE_COMMAND`; project values are ignored with a warning.

> **Watcher self-restart (2026.9.18):** the history watcher checks every minute whether its executable was replaced; it saves pending work and exits non-zero so the service manager restarts it on the new version. Pre-safeguard watchers need one `mise bootstrap services apply` (`mise doctor`/`mise dot status` report a "stale watcher schema"). Warnings from background captures are stored and shown at the next `mise dot` command or `mise bootstrap`.

> Logs, caches, databases, and session state usually belong **outside** tracked files. For a file you want saved only on request, track it with `--no-autosave` and use explicit `mise dot save`.

---

## OCI Container Images

Build and publish container images containing mise-managed tools. **Experimental** — requires `experimental = true`. `oci` is a top-level config key.

```bash
mise oci build -o ./img                  # build an image (default output ./mise-oci)
mise oci build --copy ./dist:/app/dist   # reproducible host-path copy layer
mise oci build --from ubuntu:24.04 --tag myorg/dev:latest
mise oci build --include-global          # include ~/.config/mise tools (default is project-only)
mise oci build --no-mise                 # don't embed the mise binary
mise oci build --owner 1000:1000 --no-cache   # file ownership in layers; skip the layer cache
mise oci push myregistry.io/myimg:tag    # built-in registry client (no skopeo/crane)
mise oci push --cache-from myimg:prev    # reuse layers
mise oci push --update-index             # upsert into a multi-arch image index
mise oci run --engine docker
```

```toml
[oci]
from = "debian:bookworm-slim"
workdir = "/app"
entrypoint = ["/app/dist/server"]
env = { API_TOKEN = "build-placeholder" }   # also satisfies [env] `required` vars during mise oci commands (2026.9.18)
# also: tag, cmd, user, user_id, group_id, mount_point, labels

[[oci.copy]]
host = "./dist"        # relative to the config file
image = "/app/dist"    # absolute; no . or .. — `source`/`dest` are NOT valid keys (schema rejects them)

[settings.oci]
default_from = "debian:bookworm-slim"     # base image
default_mount_point = "/mise"
insecure_registries = ["registry.lan:5000", "10.0.0.8:5000"]   # plain HTTP
```

Each tool version becomes its own content-addressable OCI layer, so bumping one tool invalidates only that layer. Output conforms to the OCI image-layout spec (consumable by `skopeo`, `crane`, `podman load`). Since v2026.7.12 the registry client is built in — `docker login` / `podman login` is the only setup needed. `mise oci build` also bakes `[dotfiles]` and `apt:` `[bootstrap.packages]` into images as dedicated, annotated layers.

**Limits:** only **asdf** plugins are rejected. Since **2026.9.15** vfox plugin tools (including custom backend plugins) are packaged — one layer per tool plus one per plugin at `/mise/plugins/<name>/`; the plugin's env hook runs on the build host. Build on a **Linux host** with the target arch (binaries are host-native) or pass `--no-mise`. Image `Env` order: base → `[env]` → tool exec env → `[oci].env` → PATH → `MISE_DATA_DIR=/mise`/`MISE_CONFIG_DIR=/etc/mise`. `[env]` secrets **are baked into the image** (mise warns). Dotfile groups are not included.

> `mise oci push --tool` was **removed** in 2026.7.12. Use `mise oci build -o ./img` + `skopeo copy` instead.

---

## Configuration and Settings

### File Hierarchy

Config files in per-directory precedence order (highest first):

1. `mise.local.toml` (gitignored)
2. `mise.toml`
3. `mise/config.toml`
4. `mise/conf.d/*.toml` (visible form, 2026.8.13+)
5. `.mise/config.toml`
6. `.mise/conf.d/*.toml`
7. `.config/mise.toml`
8. `.config/mise/config.toml`
9. `.config/mise/conf.d/*.toml` (alphabetical)

Any can also appear as dotfiles (`.mise.toml`, etc.).

**`conf.d` fragments (2026.8.7+)** work in project, global (`~/.config/mise/conf.d`), and system (`/etc/mise/conf.d`) directories. With `env_conf_d = true`, fragments named `*.<env>.toml` (plus `.local` variants) load only when that configuration environment is active.

> ⚠️ **Dotted fragment names (`node.tools.toml`) — reverted breaking change.** 2026.8.9 briefly read the extra dot as an environment selector; **2026.8.11 reverted it**. Today dotted names still load **unconditionally**, but warn as **deprecated** (since 2026.8.10): in **2027.8.10** the suffix becomes an environment selector by default. Opt in now with `env_conf_d = true` in a **miserc** file (or `MISE_ENV_CONF_D`); `env_conf_d = false` keeps legacy behavior and silences the warning. Either way, **use hyphens** — `node-tools.toml`.

**`conf.d` folder fragments (2026.9.14)** — a *folder* inside any `conf.d` directory is its own config root:

```text
~/.config/mise/conf.d/
├── git.toml                  # single-file fragment
└── git-tools/                # folder fragment (may be a symlink to a dotfiles checkout)
    ├── mise.toml             # always loaded
    ├── mise.local.toml       # always loaded, usually gitignored
    ├── mise.linux.toml       # loaded when the `linux` environment is active
    ├── mise.linux.local.toml
    └── gitconfig             # a dotfile source resolved relative to the folder
```

- Only `mise.toml`, `mise.local.toml`, `mise.<env>.toml`, `mise.<env>.local.toml` are read; **not recursive**; folders starting with `.` are ignored. `mise.<env>.toml` is unaffected by the `env_conf_d` migration.
- The folder is the **config root**: relative paths resolve inside it, `{{ config_root }}` is the folder, and **tasks run in the folder**. Its `[task_config]` applies only to its tasks, and its `includes` resolve inside it.
- Order: folder fragments load **after** single-file fragments in the same `conf.d` (alphabetically by folder) and **before** that directory's regular config. Verified on 2026.10.3: in a project, `.mise/conf.d/<folder>/mise.toml` ranks below the project `mise.toml`, but `.mise/conf.d/<folder>/mise.dev.toml` (with `dev` active) ranks **above** it.
- This is the migration target for the deprecated `[bootstrap].config_roots`.

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
- **Tasks:** A higher-precedence definition **with a command** replaces the task; a **metadata-only** block overlays it (see [Configuring File Tasks from TOML](#configuring-file-tasks-from-toml))
- **Settings:** Additive with overrides
- **`[tool_config]`:** config-root-scoped, not merged invocation-wide

**Write targeting:** `mise use`, `mise set`, `mise unset` write to the lowest-precedence file in the highest-precedence directory — with both present, writes go to `mise.toml`, not `mise.local.toml` (`mise use --env local node@20` targets `mise.local.toml`). `mise unuse` targets the **first loaded config that declares the tool** (`--path` to choose); `mise config get/set` default to the **highest-precedence loaded** file (`--file` to choose).

**Useful commands:**
```bash
mise cfg / mise config     # Show loaded files in precedence order
mise config get|ls|set     # Read/write individual config keys
mise config set --append  redactions '*_TOKEN'   # 2026.8.15+ list-key append
mise config set --remove  redactions '*_TOKEN'   # 2026.8.15+ list-key remove
mise config set --global|--system <key> <value>  # 2026.8.15+ explicit scope targeting
mise ls --current          # Active versions with overrides
mise doctor                # Diagnose setup issues
mise doctor project        # Run this project's [doctor.checks]
mise fmt                   # Format mise.toml — 2026.9.6+ also SORTS certain list values
                           # (redactions, task sources/outputs, task_config.global_inputs,
                           #  input_groups) by defined rules
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

### Remote Config Includes (`include`)

Added **2026.9.18** (not experimental; docs `https://mise.jdx.dev/configuration.html#include`). Pull shared config fragments from a git repo or OCI artifact:

```toml
include = [
  "git::https://github.com/myorg/platform.git//mise.toml?ref=main",
  "oci::ghcr.io/myorg/platform-config@sha256:0f1e2d3c...",   # artifact with mise.toml at its root
]

[tools]
node = "22"   # this file's own entries override the included ones
```

- **Forms:** `git::<url>//<path>.toml?ref=<ref>` or `oci::<registry>/<repo>[:tag|@sha256:<digest>]`; a string or an array.
- **Precedence:** each fragment is merged **into** the including file (never a config file of its own) and ranks **just below it**; later entries override earlier ones. Hooks of the same type from both run (shared first); aliases merge per name; the including file's PATH entries come first.
- **Allowed in a fragment:** `[tools]`, `[tool_alias]`, `[env]`, `[vars]`, `[hooks]`, `[alias]`, `[shell_alias]`, `[plugins]`, `[wrappers]`, `min_version` (enforced).
- **Errors (not silent):** a nested `include`; `[settings]` and monorepo keys; `[tasks]`, `task_config`, `task_templates` (share tasks via [`task_config.includes`](#remote-tasks) instead); per-file sections like `[dotfiles]`, `[daemons]`, `redactions`.
- Relative paths (`_.file`, …) and `{{ config_root }}` resolve against the **including** file; tools lock into the including file's lockfile.
- **Trust:** inherits the including file's trust (an untrusted repo can't trigger a fetch). Safe mode never fetches for project config. **Paranoid mode requires a full 40-hex commit SHA (`?ref=<sha>`) or `@sha256:` digest** — a branch/tag is an error.
- **Cache:** `$MISE_CACHE_DIR/config-includes`. SHA/digest refs are never refetched. Branch/tag refs refresh when older than `fetch_remote_versions_cache` (1h), **only** from commands that look at remote versions (`install`, `up`, `use`, `config`); hook-env, `mise ls`, `mise exec`, shims, and offline mode use the cache. A failed refresh keeps the cached copy with a warning; a cold cache with no network is an error. `mise cache clear` forces a refetch.

### Idiomatic Version Files

Disabled by default. Enable per-tool:
```bash
mise settings add idiomatic_version_file_enable_tools python
mise settings add idiomatic_version_file_enable_tools go        # go.mod support
mise settings add idiomatic_version_file_enable_tools dagger task lefthook
```

Supported files include `.nvmrc`, `.node-version`, `package.json`, `.python-version`, `.python-versions`, `.ruby-version`, `Gemfile`, `.go-version`, `go.mod`, `rust-toolchain.toml`, `.java-version`, `.sdkmanrc`, `global.json`, `.terraform-version`, `.bun-version`, `.deno-version`.

- **`go.mod` (2026.7.13+):** a `toolchain goX.Y.Z` directive is an exact pin. Reading a bare `go X.Y` minimum (and `CMakeLists.txt` `cmake_minimum_required`) as a version floor **warns and is removed in 2026.11.0** — use `toolchain goX.Y.Z`, `.go-version`, or `mise.toml`. `idiomatic_version_file_ignore_minimum_versions` goes away at the same time.
- **`.nim-version`** is an idiomatic file for `nim` (2026.9.13).
- **`go.work` (2026.9.12+):** with `go` enabled, an active `go.work`'s `toolchain` line selects the Go version and — as with the `go` command — member `go.mod` files are **ignored in workspace mode**. `GOWORK` (`auto`, `off`, or an absolute path) is honored.
- **`.bazelversion` (2026.9.12+):** an idiomatic version file for `bazel` when enabled. Only **concrete releases** are read — `latest`, `last_green`, `8.x`, and commit hashes select nothing rather than failing.
- **`idiomatic_version_file_ignore_minimum_versions`** (bool, default `false`) ignores idiomatic fields that declare only a *minimum* compatible version.
- **Structured parsers (2026.7.15+):** registry `idiomatic_files` entries support `version_regex`, `version_json_path`, and `version_expr`, giving **in-process parsing with no plugin or shell execution** for 11 tools — Dagger, Task, chezmoi, CMake, Earthly, golangci-lint, GoReleaser, Lefthook, Pixi, pre-commit, Ruff (`dagger.json`, `Taskfile.yml`, `.chezmoiversion`, …).
- **Disable individual files per tool (2026.7.17+)** with `tool:filename` pairs — e.g. keep `.nvmrc` selecting Node while stopping node from reading `devEngines.runtime` in `package.json`, with pnpm still reading it:
  ```bash
  mise settings add idiomatic_version_file_disable_files node:package.json
  ```
- Since 2026.7.18, `idiomatic_version_file_enable_tools` set in a config root's own `[settings]` is honored by monorepo-wide commands (`mise ls --monorepo`, `mise install --monorepo`).

### Key Settings Reference

mise ships **329 settings** (leaf count from `https://mise.jdx.dev/schema/mise.json` on 2026.10.3: 169 top-level keys, 37 of which are nested namespaces — note `mise settings --all` prints only the ~202 that resolve to a value, omitting unset optional ones). New since 2026.9.12: `otel.enabled`, `otel.logs`, `not_found_auto_install_registry`, `self_update.minimum_release_age`, `dotfiles.relative_symlinks` — none removed, no defaults changed. This is a representative subset — run `mise settings --all` (or `mise settings ls`) for the live list, and `mise settings set <key> <value>` / `mise settings get <key>` to manage them.

```toml
[settings]
# Execution
jobs = 8                    # Concurrent jobs (MISE_JOBS)
experimental = false        # Enable experimental features
yes = false                 # Auto-answer prompts (MISE_YES) — global only
safe = false                # Inert config-reader mode — global only

# Task defaults (example values — task.output/timeout/timings are UNSET by default)
task.output = "prefix"      # prefix|interleave|keep-order|replacing|timed|silent (unset → prefix if jobs>1)
task.timeout = "10m"        # Default task timeout
task.timings = true         # Show elapsed time
task.quiet = false          # Suppress mise's own task chatter (replaces output = "quiet")
task.skip = ["slow-task"]   # Tasks to skip
task.skip_depends = false   # Skip dependencies
task.source_freshness_hash_contents = false  # blake3 content check
task.auto_infer = []        # experimental — e.g. ["node"]
task.cache_max_size = "2GiB"  # experimental
use_file_shell_for_executable_tasks = false  # Run file tasks through a shell

# Shells — ALL FOUR ARE GLOBAL-CONFIG-ONLY since 2026.7.14
unix_default_inline_shell_args = "sh -o errexit -c"
unix_default_file_shell_args = "sh"
windows_default_inline_shell_args = "cmd /c"
windows_default_file_shell_args = "cmd /c"
windows_powershell_no_profile = true   # -NoProfile for pwsh tasks (default true)

# Environment
env_shell_expand = true     # Shell-style expansion — DEFAULT TRUE
env_cache = false           # experimental — cache computed environment (never caches secret-bearing envs, 2026.10.3)
env_cache_ttl = "1h"        # Cache TTL
env_file = ""               # MISE_ENV_FILE
auto_env = false            # Auto-load platform config files (default-on in 2027.6.0)

# Tool management
auto_install = true         # Auto-install missing tools
exec_auto_install = true    # Auto-install on mise x/run
not_found_auto_install = true
not_found_auto_install_registry = false  # 2026.9.17: install UNCONFIGURED tools into global config
auto_install_disable_tools = []
disable_backends = ["asdf"] # Disable backends (new installs only)
disable_default_registry = false  # Only affects vfox and asdf shorthands
enable_tools = []           # Allowlist (unset = all; empty = none)
disable_tools = []
pin = false                 # Default --pin for mise use
lockfile = true             # Create + update lockfiles (unset = update existing ones only)
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
minimum_release_age = "7d"  # built-in default is 24h when unset (timestamp-reporting
                            # backends only); "0s" disables
minimum_release_age_excludes = []  # Tools exempt from the release-age delay
locked_verify_provenance = false   # Re-verify at install (auto-on with paranoid)
not_found_system_fallback = true   # false = shim for a missing tool fails loudly
                                   # instead of falling back to a PATH binary
all_compile = false         # Never use precompiled binaries for any tool
url_replacements = {}       # Map of URL patterns → replacements for all requests
use_versions_host_track = true     # Anonymous download statistics
gix = true                  # Use gix for git operations (false = shell out to git)
libgit2 = true              # Use libgit2 for git operations
auto_update = false         # Opt-in self-update before interactive commands — global only
auto_update_check_duration = "7d"
self_update.minimum_release_age = "24h"  # 2026.9.17; unset → minimum_release_age → 24h; "0s" = immediate
upgrade.auto_prune = true   # 2026.8.15+: removal is now DEFERRED, not immediate —
                            # the replaced version is kept for upgrade.prune_after
upgrade.prune_after = "24h" # grace period before a replaced version is auto-pruned

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
relative_symlinks = false   # 2026.9.14: Stow-style relative links (per-entry `relative` overrides)

# OpenTelemetry (experimental, 2026.9.13)
[settings.otel]
enabled = false             # export mise run traces (needs OTEL_EXPORTER_OTLP_*ENDPOINT)
logs = false                # also export task output lines

# System packages
[settings.system_packages]
sudo = true                 # set managers = [...] to pick package managers
```

**Nested namespaces (37):** `age`, `aqua`, `cargo`, `conda`, `dotfiles`, `dotnet`, `erlang`, `forgejo`, `github`, `github_relay`, `gitlab`, `go`, `history`, `hook_env`, `java`, `node`, `npm`, `oci`, **`otel`**, `packslip`, `pipx`, `pypi`, `python`, `ruby`, `rust`, `sandbox`, `self_update`, `shims`, `skills`, `sops`, `spm`, `status`, `swift`, `system_packages`, `task`, `upgrade`, `zig`. (`task.cache` and `history.watch` nest one level deeper.) **Bold = new since 2026.9.12.**

**Notable new top-level settings:**

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `activate_shims` | bool | `true` | `false` keeps tool shim dirs off PATH during activation and hooks (2026.9.2+), so external [command wrappers](#command-wrappers-wrappers) keep working |
| `shims_dir` / `system_shims_dir` / `system_installs_dir` | path | — | Shim/install locations for system-scoped and collocated layouts (2026.9.0+) |
| `shims.exclude` | string[] | `[]` | Command names mise should **never** create shims for, leaving them to the system (2026.9.10+) |
| `lockfile_mode` | enum | `merge` | `merge` (incremental) or `generate` (**experimental** — complete lockfile generation). In `generate` mode `provenance_verified` is **no longer treated as a trust signal**. |
| `locked_scopes` | string[] | `["global","project","system"]` | Config scopes where invocation-wide `locked` mode is enforced |
| `no_env` | bool | unset | Do not load env vars from config files (matches `--no-env`) |
| `no_hooks` | bool | unset | Do not execute hooks from config files (matches `--no-hooks`) |
| `task.quiet` | bool | `false` | Suppress mise's own output while executing tasks |
| `upgrade.prune_after` | duration | `"24h"` | Grace period before versions replaced by `mise upgrade` are auto-pruned |
| `idiomatic_version_file_ignore_minimum_versions` | bool | `false` | Ignore idiomatic version-file fields that only declare a **minimum** compatible version (now hidden; removed with the floors in 2026.11.0) |
| `not_found_auto_install_registry` | bool | `false` | **2026.9.17.** Command-not-found installs an *unconfigured* registry tool when exactly one provides the command, adding it to global config |
| `self_update.minimum_release_age` | string | unset (→ `minimum_release_age` → `24h`) | **2026.9.17.** Release-age delay for self-update, auto-update, and update notices. **Not** global-only |
| `dotfiles.relative_symlinks` | bool | `false` | **2026.9.14.** Relative symlink targets for `symlink`/`symlink-each` |
| `truncate` | bool | `true` | Terminal-width truncation; `--truncate`/`--no-truncate` override. **Auto-disabled when a coding agent is detected** (2026.9.6+) |
| `disable_update_warning` | bool | `false` | Suppress the "a newer mise is available" warning (2026.9.5+) |
| `disable_hints` | string[] | `[]` | Silence specific hint messages by name |
| `self_update.repository` / `self_update.api_url` | string | `jdx/mise` / `https://api.github.com` | Source used by `mise self-update` |
| `github_relay.*` | — | — | `concurrency` (`8`, 1–32, excess fails closed), `request_timeout` (`5m`), `max_duration` (`0s` = session lifetime), `log_requests` (`false`), `log_format` (`text`\|`jsonl`) — for [`mise ssh`](#cli-commands-for-tools) borrowed GitHub access |
| `packslip.exec` / `packslip.stampers` | bool / array | `false` / unset | See [Packslip Backend](#packslip-backend-preferred-where-available) |
| `skills.*` | — | — | See [Agent Skills](#packslip-man-pages-and-agent-skills) |
| `history.*` | — | — | See [Dotfiles History](#dotfiles-history-mise-dot--history) |

**Global-config-only settings (33 in `settings.toml`, 30 excluding deprecated aliases)** — ignored when set from project config: `auto_update`, `auto_update_check_duration`, `ci`, `forgejo.credential_command`, `github.credential_command`, `gitlab.credential_command`, `github_relay.{concurrency,log_format,log_requests,max_duration,request_timeout}`, `history.allow_plaintext_history`, **`history.describe_command`** (global-only since 2026.9.7, a security fix), `locked_scopes`, `paranoid`, `safe`, `self_update.api_url`, `self_update.repository`, `shims_dir`, `system_installs_dir`, `system_shims_dir`, `task.cache.remote_oidc_audience`, `task.cache.remote_token`, `task.cache.remote_token_file` (plus their deprecated flat `task.cache_remote_*` aliases), `trusted_config_paths`, `unix_default_file_shell_args`, `unix_default_inline_shell_args`, `windows_default_file_shell_args`, `windows_default_inline_shell_args`, `yes`. `[daemons_settings]` is the inverse — **ignored from global/system config** — and `[daemon_providers]` is allowed **only** in global config.

> **`gpg_verify` behavior (2026.7.12+):** GPG verification **always runs** when enabled, with Node/Swift signatures verified in-process via rPGP (no external `gpg` binary). Previously a missing `gpg` silently skipped verification — you must now set `gpg_verify = false` explicitly to opt out.

**Settings** generally map to `MISE_` + the upper-snake path (`MISE_JOBS=4`, `MISE_TASK_OUTPUT=interleave`), but ~20 don't follow the rule: `status.*` → `MISE_STATUS_MESSAGE_*`; `python.venv_stdlib` → `MISE_VENV_STDLIB`; `python.pyenv_repo` → `MISE_PYENV_REPO`; `ruby.ruby_build_*`/`ruby.ruby_install*` → `MISE_RUBY_BUILD_*`/`MISE_RUBY_INSTALL*`; `rust.cargo_home` → `MISE_CARGO_HOME`; `rust.rustup_home` → `MISE_RUSTUP_HOME`; `dotnet.dotnet_root` → `MISE_DOTNET_ROOT`; `node.nvm_dir` → `NVM_DIR`; `node.nodenv_root` → `NODENV_ROOT`; `ci` → `CI`. About 22 legacy/deprecated settings have no env var. Five settings are **env-only** (must be env vars): `default_config_filename`, `default_tool_versions_filename`, `global_config_file`, `global_config_root`, `system_config_file`.

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
| `MISE_TASK_COLOR` | ANSI start sequence of the task's prefix colour (empty when colours are off or there's no prefix) |
| `TRACEPARENT` / `TRACESTATE` | With [OpenTelemetry](#task-tracing-with-opentelemetry-experimental) enabled — nested `mise run` and OTel-aware tools join the trace |

### CI/CD Integration

**GitHub Actions (`jdx/mise-action@v5`):**

```yaml
- uses: actions/checkout@v7
- id: mise
  uses: jdx/mise-action@v5      # latest v5.1.1 (2026-10-04); v5 is the current major
  with:
    version: 2026.10.3    # pin mise (default: newest release ≥ minimum_release_age old)
    minimum_release_age: 24h   # v5 DEFAULT; "0s" restores v4's "newest release" behavior
    # sha256: "…"         # verify the mise binary
    install: true         # run `mise install`
    install_args: "bun"   # extra args to `mise install`
    plugins: |            # v5.1: newline list of `name` or `name url`
      my-plugin https://github.com/me/vfox-my-plugin
    bootstrap: false      # run `mise bootstrap` instead of `mise install`
    bootstrap_skip: "tools,task"
    bootstrap_args: "--yes"
    cache: true           # cache via GitHub cache
    cache_save_post: false   # v5.1: save the cache in the post step (catches later-installed tools)
    cache_key: "{{cache_key_prefix}}-{{platform}}-{{install_args_hash}}-{{plugins_hash}}-{{file_hash}}"
    auto_update: false    # v5.1: keep the cached mise until the cache key changes
    experimental: false   # enable experimental features
    log_level: info
    working_directory: .
    reshim: false         # run `mise reshim --all`
    env: true             # export mise environment variables
    export_path: true     # add mise PATH entries to subsequent steps
    github_token: ${{ secrets.GITHUB_TOKEN }}
    persist_github_token: false   # v5.1.1: token is NO LONGER exported to later steps by default
    # tool_versions: |    # optionally inline .tool-versions content
    # mise_toml: |        # optionally inline a mise.toml
- run: echo "node is ${{ steps.mise.outputs.node }}"   # v5.1 per-tool outputs (+ `versions` JSON)
```

> 🔴 **mise-action v5 breaking changes** (verified against the GitHub API, 2026-10-05 — mise's own CI docs page still shows `@v4`, which is stale):
> - **v5.0.0:** `minimum_release_age` defaults to **`24h`** — an unpinned run installs the newest stable mise **at least a day old** (release list from a CDN index, no API quota). Set `minimum_release_age: 0s` for v4 behavior.
> - **v5.0.1:** a cached/existing mise binary is integrity-verified (signed checksums or the `sha256` input) before running; version switches do a full install.
> - **v5.1.1:** the GitHub token is **no longer exported** as `MISE_GITHUB_TOKEN` to later steps. `persist_github_token: true` restores it, or pass a different (e.g. read-only) token value.
> - New outputs: `cache-hit`, `versions` (JSON of `{version, requested_version, install_path, source}` per tool), and one output per tool (`steps.mise.outputs.node`).

Behavior worth knowing: PATH entries are added individually via `GITHUB_PATH` (the runner's complete PATH is not copied into `GITHUB_ENV`). When a `mise.lock` exists in the working directory or a parent, the action **automatically appends `--locked`** — unless you supply `mise_toml`/`tool_versions` inputs. `install_args` cannot be combined with `bootstrap: true`. Values flagged `redact = true` or matching `redactions` are masked automatically. Cache-key templating supports `{{version}}`, `{{cache_key_prefix}}` (default `mise-v1`), `{{platform}}`, `{{file_hash}}`, `{{mise_env}}`, `{{install_args_hash}}`, `{{bootstrap_hash}}`, `{{plugins_hash}}`, `{{env.VAR_NAME}}`, `{{default}}`, and `{{#if …}}…{{/if}}` conditionals. `wings_enabled` (experimental mise-wings asset cache via OIDC; needs `permissions: id-token: write`).

**GitLab CI** — either commit a wrapper (`mise generate install-script -l -w ./bin/mise`) or use the official image:

```yaml
# Committed wrapper (the localized wrapper keeps installs in .mise/ inside the project)
build-job:
  image: debian:13-slim
  cache:
    key:
      prefix: mise-debian13-amd64      # distinct prefix per architecture
      files: [bin/mise, mise.toml, mise.lock]
    paths: [.mise/installs/, .mise/cache/]
  before_script:
    - apt-get update && apt-get install -y --no-install-recommends curl ca-certificates tar
  script:
    - ./bin/mise install --locked
    - ./bin/mise exec -- npm run build
```

```yaml
# Official -debian image (mise, curl, git, CA certs preinstalled; no wrapper needed)
build-job:
  image: ghcr.io/jdx/mise:2026.10.3-debian
  variables:
    MISE_DATA_DIR: $CI_PROJECT_DIR/.mise
    MISE_CACHE_DIR: $CI_PROJECT_DIR/.mise/cache
  cache:
    key:
      prefix: mise-image-amd64
      files: [mise.toml, mise.lock]
    paths: [.mise/installs/, .mise/cache/]
  script:
    - mise install --locked
    - mise exec -- npm run build
```

**Official Docker images (2026.9.12+)** — `ghcr.io/jdx/mise` and `jdxcode/mise`, for `linux/amd64` and `linux/arm64`, built from the **minisign-verified release binaries**:

| Tag | Contents |
|-----|----------|
| `2026.10.3`, `2026.10`, `latest` | **scratch** image (static binary + CA certificates, **no shell**) — intended for `COPY --from=` |
| `debian`, `*-debian` | Debian slim base with `curl` and `git` |
| `dev` | Unsupported source-built image |

```Dockerfile
FROM debian:13-slim
COPY --from=ghcr.io/jdx/mise:2026.10.3 /usr/local/bin/mise /usr/local/bin/mise
```

> 🔴 **Breaking (2026.9.12):** `:latest` is now the **scratch** image, not a usable base. CI and dev-container users should switch to the `debian` tag and install tools explicitly.

**Generic CI bootstrap:**
```bash
curl https://mise.run | sh
mise install
mise x -- <cmd>

# Skip reinstall if already present (Docker layer caching)
curl https://mise.run | MISE_INSTALL_SKIP_IF_EXISTS=1 sh

# Reproducible: pin the version (bypasses the release-age delay)
curl https://mise.run | MISE_VERSION=2026.10.3 sh
```

> **The installer now waits 24h too (2026.9.17/9.18).** `curl https://mise.run | sh` picks the newest stable release **at least 24h old at run time**. Override with `MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE` (→ `MISE_MINIMUM_RELEASE_AGE` → `24h`; integer + `s|m|h|d|w`, `0s` = immediate); `MISE_VERSION` pins and bypasses it.

Or `mise generate install-script -l -w ./bin/mise` produces a self-contained `./bin/mise` you can commit, so jobs don't re-download mise (the old `mise generate bootstrap` spelling is a hidden alias until 2027.9.0). `-l/--localize` sandboxes `MISE_DATA_DIR`/`MISE_CACHE_DIR` into a `.mise` directory in the project. Without `--version`, the generated wrapper is pinned to the age-eligible release. The script honors `MISE_VERSION` and `MISE_INSTALL_PATH`.

**packslip install (CI/Docker, 2026.10.x docs):** `packslip install github.com/jdx/mise --pin ps1_nlhmwtfeufglxv5myvwvronk7a [--version 2026.10.3]` (packslip ≥ 1.5.1; Linux x64/arm64, macOS arm64, Windows) installs mise with signer verification pinned to mise's repository.

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

- **VS Code:** extension `hverlin/mise-vscode` (tools, tasks, env, `mise.toml` completion; auto-configuring other language extensions is **off by default** — `mise.configureExtensionsAutomatically`). For task/debug terminals set an automation profile: `"terminal.integrated.automationProfile.osx": { "path": "/bin/zsh", "args": ["--login"] }` (it doesn't affect the extension host or LSPs). Or use `runtimeExecutable: "mise"` with `runtimeArgs: ["exec", "--", "node"]` in launch configs. Remote contexts (SSH/WSL/devcontainer) need mise on that side. Since 2026.10.2 the JSON schema validates per-backend tool options, so typos in `[tools]` now show as editor errors.
- **JetBrains:** plugin `intellij-mise` (tools + run-configuration env), or the asdf-compat symlink `ln -s ~/.local/share/mise ~/.asdf`.
- **Xcode:** run tools with `"$HOME/.local/bin/mise" --cd "$SRCROOT" exec -- swiftlint lint`. With User Script Sandboxing, listing `$(SRCROOT)/mise.toml` as an input file is not enough for every tool (it also needs the installed executables/data dirs). Xcode Cloud: commit a wrapper and in `ci_scripts/ci_post_clone.sh` run `cd "$CI_PRIMARY_REPOSITORY_PATH"; ./bin/mise install`, then `./bin/mise exec -- swiftlint lint`.
- **Neovim:** plugin `miser.nvim`, or the shim-PATH snippet above.
- **Emacs:** package `mise.el` — `(add-hook 'after-init-hook #'global-mise-mode)`; or add the shims dir to both `PATH` and `exec-path`.
- **Vim:** `let $PATH = $HOME . '/.local/share/mise/shims:' . $PATH`.

> For Bash, edit the **first existing** of `~/.bash_profile`, `~/.bash_login`, `~/.profile` — creating a new `~/.bash_profile` can stop `~/.profile` from being read. On Linux the login profile is read at login, so logout/login is required after editing it. For a fixed SDK path an IDE can't re-resolve, point it at `mise which <bin>` / `mise where <tool>`. JetBrains' asdf-compat symlink works only if `~/.asdf` doesn't already exist.

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
- `MISE_CONFIG_DIR` (default `~/.config/mise`; holds global `config.toml`, `conf.d/`, `miserc[.local].toml`)
- `MISE_SYSTEM_CONFIG_FILE` (default `/etc/mise/config.toml`)
- `MISE_ENV_FILE` (e.g., `.env`; parsed by `mise-dotenv` since 2026.10.3)
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
- `MISE_DEBUG=1` / `MISE_TRACE=1` — debug / trace logging
- `MISE_SELF_UPDATE_MINIMUM_RELEASE_AGE` — release-age delay for self-update and the installer
- `MISE_NOT_FOUND_AUTO_INSTALL_REGISTRY`, `MISE_DOTFILES_RELATIVE_SYMLINKS`, `MISE_OTEL_ENABLED`, `MISE_OTEL_LOGS` — settings added in 2026.9.13 through 2026.9.17

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

`...` matches directory depth (bazel/buck2 style); `*` matches task names. Path globs (`*`/`**` in the path portion) are **not** supported yet. mise never defines commands starting with `//` or `:`, so direct invocation is safe here. `mise run :<TAB>` completes the `:task` shorthand (2026.10.1).

**Short names for deep project paths (`[monorepo.path_aliases]`, 2026.9.16):**
```toml
monorepo_root = true

[monorepo]
config_roots = ["foo/bar/baz/abc/123"]

[monorepo.path_aliases]
"123" = "foo/bar/baz/abc/123"
```
`mise run //123:build` then runs `//foo/bar/baz/abc/123:build` — also in `depends`, patterns (`//123:*`), and child paths (`//123/sub:build`). An alias must be a **single path segment**, must point at a root in `config_roots`, can't contain `...`, and can't overlap an existing root. The full path stays the canonical name in listings and output prefixes.

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

Dates below come from mise's own `deprecated_at!` macros and `settings.toml` (`deprecated_warn_at` / `deprecated_remove_at`) at v2026.10.3 — the docs disagree with the source in places, and the source wins.

| Removal | What | Migrate to |
|---------|------|------------|
| **2026.11.0** ⚠️ **soonest** | `go.mod` `go X.Y` and `CMakeLists.txt` `cmake_minimum_required` version **floors** stop being read (warn since 2026.8.10); `idiomatic_version_file_ignore_minimum_versions` goes with them. `toolchain goX.Y.Z` unaffected. | Pin an exact version |
| **2026.11.0** (warn) | `credential_command` legacy single positional argument; `*.default_packages_file`; `dotnet.package_flags` | `MISE_CREDENTIAL_HOST`/`MISE_CREDENTIAL_PROVIDER`; tool-level `postinstall`; `prerelease` option |
| **2026.12.0** | `shorthands_file` · `env.mise.*` namespace · `value`/`values` keys in `_.file`/`_.path`/`_.source` | `[plugins]` · `env._.*` · `path` (string or array) |
| **2026.12.0** (warn) | `[monorepo].lockfile` unset behavior; `aqua.registry_url`; `auto_env` platform-config warning | Set them explicitly; `aqua.registries` |
| **2027.1.0** | **`ubi:` backend** (warns since 2026.4.0) | `github:` (give each binary its own `[tool_alias]`) |
| **2027.2.0** | Flat `task_*` settings (`task_output`, `task_timeout`, `task_skip`, …) — **warnings live since 2026.8.0** | Dotted `task.*` |
| **2027.2.0** (warn) | Flat `task.cache_remote_*` settings | Nested `task.cache.remote_*` (removal 2027.8.0) |
| **2027.3.0** | Hook `script`/`scripts` spawned table form (warns since 2026.9.0); legacy `{version}` template syntax | `run`; `{{version}}` |
| **2027.3.3** | `[bootstrap].config_roots` and `mise bootstrap config-roots` (warns since **2026.9.3**, hidden from help) | `conf.d` folder fragments (the warning gives the exact path) |
| **2027.4.0** | `tera_v1` / `MISE_TERA_V1` and Tera v1 helpers (**warnings live since 2026.10.0**); top-level `env_file` / `dotenv` / `env_path` (warn since 2026.4.17); `mise b` alias | Tera v2; `_.file` / `_.path`; `mise backends` |
| **2027.5.0** | Tera task-arg functions `{{arg()}}`, `{{option()}}`, `{{flag()}}` in run scripts; `mise github …` | `usage` spec + `$usage_*`; `mise token github` |
| **2027.7.0** | `[tools] python = { virtualenv = … }` tool option; `python.uv_venv_auto = true` (legacy value) | `env._.python.venv`; `"source"` / `"create\|source"` |
| **2027.8.0** | Automatic `all_compile = true` distro defaults on **NixOS** and **Alpine**; flat `task.cache_remote_*` | Enable `nix-ld`, or set `all_compile = true` explicitly; `task.cache.remote_*` |
| **2027.8.5** | `-l` as the `--bump` shorthand on `upgrade`/`outdated` (so `-l` can later mean `--local`) | `-b` / `--bump` |
| **2027.8.10** | Dotted **unconditional** `conf.d` filenames (`node.tools.toml`) start acting as environment selectors (warns since 2026.8.10) | Hyphens (`node-tools.toml`), or opt in now with `env_conf_d = true` |
| **2027.8.14** | Task/`[task_config]` `rust_cache` — already a **no-op** | [mbx](https://mr-boxington.jdx.dev/getting-started) |
| **2027.9.0** | `mise generate bootstrap` | `mise generate install-script` |
| **2027.9.3** | Task `output = "quiet"` mode (warns since **2026.9.3**) | `output = "interleave"` + `task.quiet = true` |
| **2027.9.13** | `[dotfiles]` `exclude` patterns where `*` crosses `/` (warns since 2026.9.13) | `**` |
| **2027.10.0** | `install_before` (**warning live since 2026.10.0**) | `minimum_release_age` |
| **2027.11.0** | `credential_command` positional arg; `*.default_packages_file`; `dotnet.package_flags` | see above |
| **2027.12.0** | `experimental_monorepo_root` (warns since 2026.7.7); `aqua.registry_url` | `monorepo_root`; `aqua.registries` |

> **Not deprecated:** the top-level `mise dotfiles` / `mise dot` command. A 2026.7.16 note announced a deprecation (removal 2028.2.0), but it was **reversed in 2026.9.8** — re-verified on 2026.10.3 (no `deprecated_at!`, no warning).

**Default flips ahead:** `auto_env` → `true` in **2027.6.0** (warns from 2026.12.0) · `[monorepo].lockfile` → root lockfiles in **2027.6.0** · `cargo.binstall_native` warns 2027.1.0 and defaults on **2027.7.0**.

**Recent default changes:** `ruby.compile` now defaults to **precompiled binaries** (2026.8.0), and since **2026.8.2** `ruby.compile = false` is a *strict precompiled-only* mode — installs error with `no precompiled ruby found` instead of falling back to ruby-build, and version listings are filtered to versions with a precompiled binary for your platform (unset and `true` unchanged; Windows unaffected) · `windows_powershell_no_profile` → `true` (2026.7.13) · structured env-file values are **literal by default** again (2026.7.14) · `cargo.binstall_quickinstall` = `false`.

> **`minimum_release_age` has a built-in `24h` default** applied when the setting is unset, on timestamp-reporting backends (aqua, cargo, core, forgejo, gem, github, gitlab, go, npm, packslip, pipx/pypi, spm, ubi). `mise settings get` reporting it "not set" and the schema carrying no `default` are both true and both misleading — `settings.toml` uses `default_docs = "24h"`, and the constant `DEFAULT_MINIMUM_RELEASE_AGE = "24h"` lives in mise's source. Use `"0s"` to actually disable it.

**Other breaking changes in the 2026.8.x line:**
- **2026.8.5** — `disable_tools = ["python"]` no longer activates a `_.python.venv`. The venv stays on disk and is restored when Python is re-enabled.
- **2026.8.9** — legacy `RTX_*` environment variables (incl. `RTX_TOOL_OPTS__*`, `RTX_ADD_PATH`) **removed** from asdf/vfox plugin hooks; use `MISE_*`. Standard `ASDF_*` remain.
- **2026.8.9** — `conf.d` fragments with an extra dot before `.toml` became environment-specific — **reverted in 2026.8.11**; now a deprecation (selectors by default in 2027.8.10). Use hyphens.
- **2026.8.9** — `vlang` configs pinning `2026.x`-style versions must move to a real upstream version (`0.5.2`, `weekly.*`).
- **2026.8.6** — `mise use --global` alongside a path is now rejected rather than silently ignored.

**Breaking changes in the 2026.9.13 – 2026.10.3 line:**
- **2026.9.13** — `pkgx:` backend removed · a TOML `[tasks.x]` with `run` now **replaces** a same-named file task (metadata-only blocks configure it) · locked tools stay on their locked backend when the registry moves them (`mise backends switch`) · tool-stub lock data moved into the project `mise.lock`.
- **2026.9.15** — `mise tasks validate` exits 1 on an unparseable `usage` spec.
- **2026.9.16** — new lockfiles are **revision 3** (unreadable by older mise) · SLSA provenance requires a matching signer identity or is skipped.
- **2026.9.17** — `mise self-update` and the installer wait for a 24h release age · paranoid mode ignores `--yes`/CI auto-confirm for trust.
- **2026.9.18** — tool keys with inline `[options]` require trust in `mise.toml`.
- **2026.10.0** — same for `.tool-versions` (GHSA-wcqh-j26q-g44x) · keyless cosign needs a pinned identity (vfox plugins must set `cosign_certificate_identity`) · aqua on musl hosts installs the registry-named asset unless `libc = "musl"` · Ctrl-C exits 130 everywhere · `--from-git` removed.
- **2026.10.1/10.2** — timeouts actually stop tasks, and a timed-out task fails even on exit 0.
- **2026.10.3** — dotenv files parsed by `mise-dotenv` (a file's own values beat ambient env) · `__MISE_DIFF`/`__MISE_SESSION` hold digests, not values.

**Undated deprecations:** `[alias]` → `[tool_alias]` · `asdf_compat` (no longer supported) · `go.set_gopath` · `idiomatic_version_file` / `idiomatic_version_file_disable_tools` → `..._enable_tools` · `legacy_version_file*` → `idiomatic_version_file*` · `npm.bun` → `npm.package_manager` · `profile`/`MISE_PROFILE` → `MISE_ENV` · monorepo automatic filesystem discovery → explicit `config_roots` · registry shorthands `localstack` → `lstk`, `actionlint` → `jactionlint` (and `mbx` now means `mr-boxington`).

**Already removed:** `pkgx:` backend (**2026.9.13**, no deprecation period — lockfile `[pkgx-packages]` sections still load and are dropped on the next write) · `mise bootstrap --from-git` / `mise bootstrap remote --from-git` (**2026.10.0**; use `--adopt`) · embedded `[lock]` sections in tool stubs (2026.9.13; ignored and removed — use `mise generate tool-stub --lock`) · `vars.mise` namespace · non-string `postinstall` hooks · unknown table fields in hook definitions · `mise oci push --tool` · project-local `*_default_*_shell_args` (2026.7.14).

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
- Use literal `choices` for stable enums, `choices run="cmd"` / `choices env="VAR"` for derivable closed sets (they **validate**), and `complete` only for open-ended or cascading values
- Use `validate=` + `validate_error=` for format/range checks (ports, names) — it works in mise and rejects bad input before dependencies run
- Nest `complete` inside its `arg` (`arg "<svc>" { complete run="…" }`) — a top-level `complete "plugin"`/`"task"`/`"tool"`… is shadowed by mise's own completers
- Wrap usage-time templates in TOML `usage` strings with `{% raw %}…{% endraw %}` (`{{ words[PREV] }}`), and branch on `$usage_cmd` for subcommands
- Run `mise tasks validate` in CI — one broken spec silently breaks tab-completion for every task
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
- Pin tool versions with `mise.lock`; prefer **`[tool_config] locked = true`** for config-root scope over the invocation-wide `locked` setting
- Remember `minimum_release_age` is **already on at 24h** by default for most backends; raise it (e.g. `"7d"`) to harden, or set `"0s"` to opt out — don't assume unset means off
- Use `group` to express "exactly one of these flags" instead of hand-rolling the check in the script
- Use `version_order = "semver"` on aqua/github/gitlab/forgejo/http tools whose releases include backport lines
- Use `mise lock --bump` to advance fuzzy selectors without touching `mise.toml`
- Use `jdx/mise-action@v5` in GitHub Actions — it handles masking and `--locked` automatically; pin `version:` (or set `minimum_release_age`) knowing v5 waits 24h for new mise releases, and set `persist_github_token: true` only if later steps need the token
- Use `MISE_SAFE=1` when reading configs you don't control (fork PRs, untrusted repos, lockfile bots)
- Sandbox risky tasks with `deny_*` + narrow `allow_*` lists
- Declare `[monorepo].config_roots` explicitly instead of relying on filesystem walking
- Use `mise run --affected` in monorepo CI to skip untouched projects
- Use `mise bootstrap` with `[bootstrap]`/`[dotfiles]` to onboard developers and provision fresh machines in one command
- Use `mise dot …` for dotfiles and history (the top-level command is first-class again since 2026.9.8); `mise bootstrap dotfiles …` is equivalent
- Prefer the **`packslip:`** backend when the publisher ships signed manifests — best provenance, plus version-matched completions, man pages, and agent skills
- Prefer **`pypi:`** over `pipx:` in new configs (they are separate tool identities — don't switch an existing pin casually)
- Use `lazy = true` for machine-wide tools you rarely invoke; declare `lazy_bins` for non-registry backends
- Declare long-lived services in `[daemons]` and make tasks depend on them with `daemons = [...]` instead of hand-rolled "start background process, poll for readiness" prerequisite tasks
- Use `port = "auto"` so the same stack runs in several git worktrees without hand-assigned ports
- Encode non-version requirements (native libraries, reachable services) as `[doctor.checks]` so `mise doctor project` explains them with a `hint`
- Replace the deprecated `output = "quiet"` mode with an explicit style plus `task.quiet = true`
- Commit the `.mise/locks/` sidecar directory alongside a revision-2+ `mise.lock` (`mise lock --sidecars` lists it)
- Give file tasks their `timeout`, `vars`, and sandbox fields through a metadata-only `[tasks.<name>]` TOML block — `#MISE` headers ignore them
- Share tools/env/hooks across repos with a top-level `include = ["git::…?ref=<sha>"]` and tasks with `task_config.includes = ["oci::…@sha256:…"]` — pin by SHA/digest
- Retire a managed dotfile with `mode = "absent"` rather than deleting its entry; organize Stow-style trees as `[dotfile_groups]` selected per machine
- Use a global `[daemon_providers]` server when many checkouts each need a database
- Replace `[bootstrap].config_roots` bundles with `conf.d` folder fragments
- Use `task_config.excludes` to keep stray executables out of file-task discovery
- Put shared task flags in a `[task_templates]` `usage` and let extending tasks add their own — since 2026.9.11 the two **merge**, so stop copying flags into every task
- Use `#MISE extends="<template>"` in file tasks to inherit tools/env/vars (the template's `run` is ignored there — the script is the command)

### DON'T ❌

- Use shell positional parameters or `"$@"`-style expansion for arguments
- Use `$args` in PowerShell
- Use inline template functions `{{arg()}}`/`{{option()}}`/`{{flag()}}` in run scripts (deprecated)
- Believe older advice that `choices env=` / `validate=` are compiled out of mise — both **work** since 2026.8.13
- Use usage attributes that still don't exist: `parse`, flag `config=`, `config_alias`, or a 2-positional `example` — they hard-error
- Use `clause`, `external_subcommand`, `multicall`, or `long_version` in task specs — they misbehave in mise (and `clause` breaks completion for every task)
- Put bare `{{ words[PREV] }}` in a TOML `usage` string — mise renders it first and the task fails to load
- Read `${usage_x}` inside a `complete run=` — usage vars aren't set during completion; use `{{ words[…] }}`
- Use the `slice` filter in completion templates — it isn't available, and the arg silently falls back to file completion
- Assume the old usage 4.x limits still apply — `flag { alias }`, `required_if`, `required_unless`, `overrides`, `conflicts`, and `requires` all work under v6, and `double_dash="required"` is now enforced
- Expect `arg "<start> <end>"` fixed arity to produce two env vars — both values land in the first
- Put a root-level `mount` in a TOML `usage` field — it no longer errors, but it is ignored at run time (file-task headers only)
- Put `timeout`, `vars`, `run`, or `deny_*`/`allow_*` in a `#MISE` header — they are ignored with a warning
- Put `redactions` inside `[tasks.x]` — it's a top-level key only; in a task it is a parse error
- Assume `sources` freshness ignores gitignored files — it doesn't (only `mise watch` does)
- Assume `interactive = true` lets other tasks keep running — it blocks them all
- Expect a `confirm` prompt to work in a TTY-less agent shell or CI — pass `--yes`
- Rely on `usage_*` leaking into nested tasks (invocation-local since 2026.7.6 — pass via `env=` or structured `depends`)
- Assume `--quiet` changes output style (it no longer does — use `--output`)
- Put mise's own flags after the task name (`mise run --silent build`, not `mise run build --silent`)
- Forget to quote glob patterns in sources
- Set env vars in `env` that deps need (they don't inherit — use structured `depends` with `env`)
- Use `raw = true` unless interactive input is needed (serializes each of its commands, bypasses redactions and the artifact cache)
- Set `MISE_ENV` in `mise.toml` (it determines which files to load — use `.miserc.toml`)
- Set `auto_env` in `mise.toml` — it is read during early init and has no effect there
- Set `locked = true` in a project config expecting project scope — all settings are global in scope
- Hand-place mise-looking shims in the shims directory — `mise reshim` replaces/removes entries it recognizes as its own (unrelated files are left alone)
- Use `pkgx:` tool specs — the backend was removed in 2026.9.13
- Use `mise use -E staging` to write `mise.staging.toml` — the global `-E` only selects the load env; use `mise use -e staging`
- Use `[[oci.copy]] source/dest` — the keys are `host`/`image`
- Use `[bootstrap.compose.*] path =` — Compose needs `project_dir` (absolute) + `files`
- Use inline tool options (`"tool[opt=…]"`) in shared configs expecting them to load untrusted — since 2026.9.18 they require `mise trust`
- Compute a version alias by running the same tool (`exec(command='node --version')`)
- Use `MISE_RAW=1` without knowing it sets `MISE_JOBS=1`
- Install new `asdf:` or `vfox:` plugins when aqua/github alternatives exist
- Use `[prepare.*]` — it no longer exists; use `[deps.*]`
- Use `vars.mise` or `env.mise.*` — rejected / deprecated in favor of `vars._` and `env._`
- Expect `bin_path` to support bare `{{os}}`/`{{arch}}` — use the `os()` / `arch()` functions
- Expect a **subtask** reached via `run = [{ task = "…" }]` to start its own `daemons` — declare the requirement on the task you actually invoke
- Assume switching `pipx:` → `pypi:` is a rename — it creates a **separate installation and lock entry**
- Commit a revision-3 `mise.lock` before collaborators and CI are on mise **2026.9.16+** (revision 2 needs 2026.9.7+) — older versions cannot read it
- Use `ghcr.io/jdx/mise:latest` as a base image — since 2026.9.12 it is a **scratch** image with no shell; use the `debian` tag or `COPY --from=`
- Set `[daemons_settings]` in global or system config — it is ignored there with a warning
- Put a `[doctor.checks]` secret or credential in command output — output is captured and discarded; explain via `description`/`hint` instead
- Rely on `[bootstrap].config_roots` — deprecated 2026.9.3, removed 2027.3.3
- Use `rust_cache` — it is a **deprecated no-op** that silently does nothing; use [mbx](https://mr-boxington.jdx.dev/getting-started)
- Assume `minimum_release_age` being "unset" means no delay — a **built-in 24h cutoff** applies on most backends; use `"0s"` to truly disable
- Declare the same flag in both a task template's `usage` and the extending task's — it gets listed **twice**
- Expect `depends = []` on a task to cancel an inherited `depends` — it still inherits the template's
- Assume `mise upgrade` frees disk immediately — replaced versions are kept for `upgrade.prune_after` (24h); use `--prune` to force
- Rely on the old HTTP dedup/symlink layout — own-directory extraction is the default since 2026.9.6; set `shared_extraction = true` to opt back in
- Use `mise bootstrap --from-git` — renamed to `--adopt`, **removed in 2026.10.0** (now an unknown-flag error)
- Assume `curl https://mise.run | sh` or `mise self-update` gets the newest release — both wait 24h unless `MISE_VERSION` is pinned
- Name multi-word `conf.d` fragments with dots (`node.tools.toml`) — they become environment selectors in 2027.8.10; use hyphens
- Expect `otel.enabled` alone to export traces — an `OTEL_EXPORTER_OTLP_*ENDPOINT` must also be set (and `otel.logs` turns task TTYs into pipes)

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
confirm = { message = "Deploy to {{usage.env}}?", yes = "Deploy", no = "Cancel", default = "no" }
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
