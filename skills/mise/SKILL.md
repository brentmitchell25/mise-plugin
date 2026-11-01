---
name: mise
description: Use when working with mise - creating or editing mise.toml, .mise.toml, mise.local.toml, or file tasks; defining tasks and their usage-spec arguments; managing dev tools, backends, and lockfiles; configuring environments, secrets, hooks, daemons, sandboxing, or settings; machine bootstrap and dotfiles; monorepo workspaces; or running any mise CLI command.
---

# Mise Comprehensive Skill

## Table of Contents

- [🔴 STRICT ENFORCEMENT: Usage Field Required](#-strict-enforcement-usage-field-required)
- [Overview](#overview)
- [Reference Files](#reference-files)
- [Best Practices](#best-practices)
  - [DO ✅](#do-)
  - [DON'T ❌](#dont-)
  - [Complete Task Example](#complete-task-example)
  - [Complete File Task Example](#complete-file-task-example)
  - [Complete Dev Tools + Env Example](#complete-dev-tools--env-example)

---

## 🔴 STRICT ENFORCEMENT: Usage Field Required

Define every task argument with the `usage` field (or `#USAGE` lines in a file task) and read it from the `usage_*` environment variables. Don't read task arguments through shell-native positional parameters (`$1`, `"$@"`) or inline Tera functions; the table below gives the reasons. The rule covers how a task receives its arguments — `$1` inside an awk program or a shell function in the script body is unrelated.

### ✅ REQUIRED Pattern

```toml
[tasks.deploy]
description = "Deploy to environment"
usage = 'arg "<env>" help="Target environment" { choices "dev" "staging" "prod" }'
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

# BLOCKED: Inline Tera templates (deprecated, removed 2027.5.0)
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

> **Verified against mise 2026.10.25** (released 2026-10-31) and **usage-lib 7.1.4** (the version that mise bundles; usage 7.1.5 is not in a mise release yet), using the live binary (`mise settings`, `mise --help`, `mise backends ls`, real `mise run` probes), the published JSON schema (`https://mise.jdx.dev/schema/mise.json`), the docs source at the `v2026.10.25` tag, mise's Rust source for deprecation dates, and the `jdx/mise` release notes for every release from 2026.10.4 to 2026.10.25. Where docs and release notes disagree, release notes win; where both disagree with the binary, the binary wins.

mise is an all-in-one developer environment tool that manages:

- **Dev tools** — install and manage language runtimes, CLIs, and build tools (20 backends)
- **Tasks** — project-specific commands with argument handling, dependencies, freshness checks, and output caching
- **Environments** — manage env vars, configuration environments, dotenv files (with `find_up`), per-machine `[vars]` prompts, age/sops-encrypted values, and task-scoped fnox secrets
- **Hooks** — run commands on directory changes, project enter/leave, tool install
- **Daemons** — long-lived project services (databases, brokers, dev servers, Docker/Podman containers) supervised by pitchfork, startable as a task prerequisite
- **Diagnostics** — project-defined `mise doctor project` checks for requirements tool versions can't express
- **Wrappers** — intercept a command name before it reaches the active toolset
- **Sandboxing** — restrict a task's filesystem/network/env access; `safe` mode for untrusted configs
- **Machine bootstrap** — provision a whole dev machine, locally or over SSH (system packages, users/groups, privileged files, systemd services, Docker Compose, firewall, git repos, dotfiles, macOS defaults, login shell) via `mise bootstrap`
- **Dotfiles history** — Git-backed checkpoints of tracked config files, with rollback, undo, and optional sharing
- **Shared config** — pull `[tools]`/`[env]`/`[hooks]` fragments from a git repo or OCI artifact with top-level `include`, and task catalogs with `task_config.includes`
- **Tracing** — export `mise run` traces and task logs over OpenTelemetry (experimental)
- **Agent skills** — version-matched `SKILL.md` bundles published by `packslip:` tools, linked into an agent's skills directory
- **Editor support** — `mise lsp` language server for mise config files; `mise config explain` for key-by-key explanations
- **Automatic tool updates** — keep global tools (e.g. coding agents) current within their request
- **Monorepos** — workspace project graph inference across Cargo/uv/Go/Node, affected-task selection
- **OCI images** — build/push container images containing mise-managed tools

**Key Features:**
- Parallel dependency building (concurrent by default, up to `jobs` setting)
- Last-modified and content-hash (blake3) freshness checking
- Task output artifact caching, local and remote (experimental); `skip_if` probes
- File watching (`mise watch` rebuilds on changes)
- Cross-platform argument handling via `usage` spec
- Hierarchical configuration with configuration-environment support
- 20 tool backends (aqua, packslip, github, gitlab, forgejo, npm, cargo, pypi, etc.)
- Security verification (cosign, SLSA, GitHub Attestations, minisign, packslip signer pinning) — all native, no external CLIs
- Secret management (fnox, sops, age encryption)

**Top-level `mise.toml` keys** (37; authoritative, from `https://mise.jdx.dev/schema/mise.json`):
`_`, `alias` (deprecated), `bootstrap`, `daemon_groups`, `daemon_providers`, `daemons`, `daemons_settings`, `deps`, `doctor`,
`dotenv` (deprecated), `dotfile_groups`, `dotfiles`, `env`, `env_file` (deprecated), `env_path` (deprecated),
`experimental_monorepo_root` (deprecated), `history`, `hooks`, `include`, `min_version`, `monorepo`, `monorepo_root`, `oci`,
`plugins`, `redactions`, `secrets`, `settings`, `shell_alias`, `task_config`, `task_templates`, `tasks`, `tool_alias`, `tool_config`,
`tools`, `vars`, `watch_files`, `wrappers`.

> There is **no `[prepare]` key** — that feature is `[deps]` (the CLI accepts `mise prepare` as an alias for `mise deps`). See [Dependency Preparation](references/deps.md#dependency-preparation-deps).

---

## Reference Files

The full reference lives in `references/`, one file per topic. Before writing or reviewing config for a topic, read its file — each one opens with a contents list, so you can jump to the subsection you need. Facts in these files were verified against the mise version named in the Overview.

| File | Read when | Covers |
|------|-----------|--------|
| [`references/task-definition.md`](references/task-definition.md) | Writing or moving a task (TOML task or file task) | TOML-Based Tasks (in mise.toml), File-Based Tasks (executable scripts), Configuring File Tasks from TOML, Task Grouping (Namespaces), Remote Tasks |
| [`references/usage-spec.md`](references/usage-spec.md) | Adding arguments, flags, choices, validation, or completions to a task | Positional Arguments (arg), Flags (flag), Custom Completions (complete), Accessing Arguments in Scripts, File Task Headers… |
| [`references/task-configuration.md`](references/task-configuration.md) | Setting any task field: depends, env, tools, timeout, extends, vars, secrets | All Task Fields, Structured run Array, Structured depends with Args/Env, Task Templates (extends), Global Task Configuration… |
| [`references/running-tasks.md`](references/running-tasks.md) | Running tasks, or explaining `mise run` / `mise watch` flags and output modes | Commands, Execution Flags, Output Modes, mise watch Flags, Parallel Tasks and Wildcards… |
| [`references/task-dependencies.md`](references/task-dependencies.md) | Ordering tasks, sources/outputs freshness, `skip_if`, redactions | Dependencies, Freshness with sources/outputs, Conditional Skip (skip_if), Redactions (Experimental), Error Handling |
| [`references/task-caching.md`](references/task-caching.md) | Caching task outputs locally or remotely | Local Artifact Cache, Cache Inputs, Remote Task Cache, Rust Compiler Action Cache — REMOVED (rust_cache is a no-op) |
| [`references/task-tracing.md`](references/task-tracing.md) | Exporting task traces or logs over OpenTelemetry | Task Tracing with OpenTelemetry (Experimental) |
| [`references/dev-tools.md`](references/dev-tools.md) | Adding or pinning tools, choosing a backend, lockfiles, shims, tool CLI commands | Backends Overview, TOML Syntax for Tools, Per-Tool Options, Backend-Specific Configuration, Lazy Tools… |
| [`references/environments.md`](references/environments.md) | Env vars, dotenv files, `MISE_ENV` profiles, secrets, Tera templates | Basic Variables, Special Directives (env._), Profiles / Configuration Environments (MISE_ENV), Required and Redacted Variables, Secrets (fnox, SOPS, age)… |
| [`references/hooks.md`](references/hooks.md) | Running commands on cd/enter/leave/install, or `watch_files` | Hook Types, Syntax, Watch Files, Hook Environment Variables, Per-Tool postinstall (not a hook) |
| [`references/daemons.md`](references/daemons.md) | Declaring long-lived services or making tasks depend on them | Declaring Daemons, Tasks That Require Daemons, Groups, Namespaces, and Cross-Project Daemons, Shared Server Providers ([daemon_providers]), Service Presets… |
| [`references/doctor.md`](references/doctor.md) | Adding project checks for `mise doctor project` | Project Diagnostics ([doctor]) |
| [`references/wrappers.md`](references/wrappers.md) | Intercepting a command name before it reaches a tool | Command Wrappers ([wrappers]) |
| [`references/sandboxing.md`](references/sandboxing.md) | Restricting a task's filesystem/network/env access, or handling untrusted configs | Sandboxing ([settings.sandbox]), Safe Mode (safe / MISE_SAFE=1), Paranoid Mode (paranoid / MISE_PARANOID), Trust |
| [`references/bootstrap.md`](references/bootstrap.md) | Provisioning a machine, dotfiles, dotfile groups, dotfiles history | [bootstrap] Configuration, Remote Bootstrap over SSH ([bootstrap.remote]), Declarative Dotfiles ([dotfiles]), Dotfile Groups ([dotfile_groups]), Dotfiles History (mise dot / [history]) |
| [`references/oci.md`](references/oci.md) | Building or pushing container images with mise tools | OCI Container Images |
| [`references/settings.md`](references/settings.md) | Config file hierarchy, includes, settings, `min_version`, env vars, CI, editors and `mise lsp`, MCP | File Hierarchy, Remote Config Includes (include), Idiomatic Version Files, Key Settings Reference, Minimum Version… |
| [`references/deps.md`](references/deps.md) | Project dependency installation with `[deps]` | Dependency Preparation ([deps]) |
| [`references/monorepo.md`](references/monorepo.md) | Workspace project graph, `--affected`, monorepo task layouts | Workspace Project Graph, Affected Tasks |
| [`references/deprecations.md`](references/deprecations.md) | Checking whether a key, setting, or backend is deprecated or scheduled for removal | Deprecation Calendar |

---

## Best Practices

### DO ✅

- **Always use `usage` field** for task arguments
- Use `${var?}` for required args to fail early; test **boolean** flags with `[ "${usage_x:-false}" = "true" ]` (a `-n` test treats the string `"false"` as on) and **value** args with `[ -n "${usage_x:-}" ]`
- Put `choices` in a child block — `arg "<env>" { choices "dev" "prod" }`; the inline form is silently ignored
- Use `${usage_count_flag:-0}` for count flags — never put `default` on a `count` flag
- Set `description` for discoverability
- Use `sources`/`outputs` for freshness; add `cache = { enabled = true }` (experimental) to restore outputs and replay logs
- Use `outputs = []` for lint/test/typecheck tasks that produce no files
- Declare `pass_through_env` for tokens so credential rotation doesn't bust the cache
- Use `depends` for task ordering; structured `depends` to pass args/env; `optional = true` for globs that may match nothing
- Use `confirm` for destructive operations
- Use literal `choices` for stable enums, `choices run="cmd"` / `choices env="VAR"` for derivable closed sets (they **validate**), and `complete` only for open-ended or cascading values
- Use `validate=` + `validate_error=` for format/range checks (ports, names) — it works in mise and rejects bad input before dependencies run
- Nest `complete` inside its `arg` (`arg "<svc>" { complete run="…" }`) — it keeps the completer beside its arg and wins over a same-named top-level `complete`
- Give a relationship several targets with the child-node form — `flag "--a" { conflicts "--b" "--c" }`, never `conflicts="--b --c"`
- Wrap usage-time templates in TOML `usage` strings with `{% raw %}…{% endraw %}` (`{{ words[PREV] }}`), and branch on `$usage_cmd` for subcommands
- Run `mise tasks validate` in CI — one broken spec silently breaks tab-completion for every task (it also flags `secrets` without `[secrets.fnox]`, secret/sandbox/env conflicts, and missing `daemons`; it does **not** catch `clause` or inline `choices`)
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
- Remember `minimum_release_age` is **already on at 24h** by default for **every** backend (2026.10.15); raise it (e.g. `"7d"`) to harden, or set `"0s"` to opt out — don't assume unset means off
- Use `group` to express "exactly one of these flags" instead of hand-rolling the check in the script
- Use `version_order = "semver"` on aqua/github/gitlab/forgejo/http tools whose releases include backport lines
- Use `mise lock --bump` to advance fuzzy selectors without touching `mise.toml`
- Use `jdx/mise-action@v5` in GitHub Actions — it handles masking and `--locked` automatically; pin `version:` (or set `minimum_release_age`) knowing v5 waits 24h for new mise releases, and set `persist_github_token: true` only if later steps need the token
- Use `MISE_SAFE=1` when reading configs you don't control (fork PRs, untrusted repos, lockfile bots)
- Sandbox risky tasks with `deny_*` + narrow `allow_*` lists
- Declare `[monorepo].config_roots` explicitly instead of relying on filesystem walking
- Use `mise run --affected` in monorepo CI to skip untouched projects (stable since 2026.10.6; add `--affected-committed` in CI if stray working-tree files might widen the set, and clone with `fetch-depth: 0`)
- Use `mise bootstrap` with `[bootstrap]`/`[dotfiles]` to onboard developers and provision fresh machines in one command
- Use `mise dot …` for dotfiles and history — `mise bootstrap dotfiles …` is a hidden alias since 2026.10.5
- Prefer the **`packslip:`** backend when the publisher ships signed manifests — best provenance, plus version-matched completions, man pages, and agent skills
- Prefer **`pypi:`** over `pipx:` in new configs (they are separate tool identities — don't switch an existing pin casually)
- Use `lazy = true` for machine-wide tools you rarely invoke; declare `lazy_bins` for non-registry backends
- Declare long-lived services in `[daemons]` and make tasks depend on them with `daemons = [...]` instead of hand-rolled "start background process, poll for readiness" prerequisite tasks
- Use `port = "auto"` so the same stack runs in several git worktrees without hand-assigned ports
- Encode non-version requirements (native libraries, reachable services) as `[doctor.checks]` so `mise doctor project` explains them with a `hint`
- Replace the deprecated `output = "quiet"` mode with an explicit style plus `task.quiet = true`
- Commit the `.mise/locks/` sidecar directory alongside a revision-2+ `mise.lock` (`mise lock --sidecars` lists it)
- Put any task field except `run`/`run_windows`/`file`/`usage` straight into `#MISE` headers (2026.10.14+); use a metadata-only `[tasks.<name>]` block only when older mise must also honor `timeout`/`vars`/sandbox fields
- Share tools/env/hooks across repos with a top-level `include = ["git::…?ref=<sha>"]` and tasks with `task_config.includes = ["oci::…@sha256:…"]` — pin by SHA/digest
- Retire a managed dotfile with `mode = "absent"` rather than deleting its entry; organize Stow-style trees as `[dotfile_groups]` selected per machine
- Use a global `[daemon_providers]` server when many checkouts each need a database
- Replace `[bootstrap].config_roots` bundles with `conf.d` folder fragments
- Use `task_config.excludes` to keep stray executables out of file-task discovery
- Put shared task flags in a `[task_templates]` `usage` and let extending tasks add their own — since 2026.9.11 the two **merge**, so stop copying flags into every task
- Use `#MISE extends="<template>"` in file tasks to inherit tools/env/vars/usage (the template's `run` is ignored there — the script is the command)
- Use `skip_if` for "already done?" probes (workspace exists, image pushed) instead of hand-rolled early exits in the script
- Grant credentials per task with `[secrets.fnox]` + `secrets = [...]` (experimental) instead of exporting them into `[env]` for every task
- Attach `mise lsp` to TOML files in your editor and run `mise config explain` to catch unknown/deprecated keys — unknown tool options now **fail installs**
- Put personal overrides that must beat every project (`AWS_PROFILE`, a pinned tool) in `~/.config/mise/config.override.toml` (`--override`)
- Collect per-developer values with `[vars.x] prompt = "…"` + `mise vars prompt` rather than asking people to edit `mise.local.toml`
- Keep global CLIs current with `auto_update` / `tool_update.global_auto` in **global** config (never in a project)
- In CI, cache `~/.cache/mise/downloads-cas` (verified) rather than installed tools, and `mise lock --platform …` every CI platform
- Use `[doctor.checks.*] depends` to order diagnostics so dependent checks report `blocked` instead of noisy failures
- Run tasks named `lsp`, `vars`, `secrets`, or `installs` with `mise run <name>` — the bare form now hits built-in commands (likewise `mise tasks set`/`rm`/`remove` are now subcommands)
- Script task edits with `mise tasks add --force`, `mise tasks set`, and `mise tasks rm --yes` (preview with `--dry-run --json`); `--env local` keeps a personal task in `mise.local.toml`
- Block until a file changes with `mise watch <task> --once` (exits with the task's status) instead of a sleep/poll loop
- Add personal bin dirs with `_.path = { path = ["~/bin"], position = "append" }` so they can't shadow mise-managed tools
- Use `--no-global` (or `--project`) on `ls`, `outdated`, `upgrade`, and `tasks ls` to leave out global config
- Write `mise use <tool>@latest` to install the newest release — a bare `mise use <tool>` reuses the newest installed version
- Anchor `version_regex` (`'^v\d+\.\d+\.\d+$'`) on github/gitlab/forgejo tools — it matches anywhere in the raw tag
- Set `registry_index = "priority"` on `pypi:` tools whose `registry_url` index hosts only your own packages
- Behind a TLS-inspecting proxy, set `extra_ca_certs` (global config or `MISE_EXTRA_CA_CERTS`) instead of replacing `SSL_CERT_FILE`
- Run databases and brokers as container daemons (`image = "postgres:18"`, `data_mount`, `proxy = false`) when you'd rather not install them as tools

### DON'T ❌

- Use shell positional parameters or `"$@"`-style expansion for arguments
- Use `$args` in PowerShell
- Use inline template functions `{{arg()}}`/`{{option()}}`/`{{flag()}}` in run scripts (deprecated)
- Use usage attributes that still don't exist: `parse`, flag `config=`, `config_alias`, or a 2-positional `example` — they hard-error
- Use `clause`, `external_subcommand`, `multicall`, `version`, or `long_version` in task specs — they misbehave in mise (and `clause` breaks completion for every task)
- Write `choices` inline (`arg "<env>" choices "a" "b"`) — it parses but validates nothing; use `{ choices "a" "b" }`
- Test boolean flags with `-n` or `${usage_x:+…}` — `default=#false`, `--no-x`, and env-backed flags set the string `"false"`
- Put bare `{{ words[PREV] }}` in a TOML `usage` string — mise renders it first and the task fails to load
- Read `${usage_x}` inside a `complete run=` — usage vars aren't set during completion; use `{{ words[…] }}`
- Use the `slice` filter in completion templates — it isn't available, and the arg silently falls back to file completion
- Assume the old usage 4.x limits still apply — `flag { alias }`, `required_if`, `required_unless`, `overrides`, `conflicts`, and `requires` all work under v6, and `double_dash="required"` is now enforced
- Expect `arg "<start> <end>"` fixed arity to produce two env vars — both values land in the first
- Write a `usage` spec that is only a `mount` — with no `arg`, `flag`, or `cmd`, mise ignores the spec at run time (TOML and file tasks); add an arg/flag or put the mount in a `cmd`
- Give a relationship property several targets (`conflicts="--b --c"`) or repeat it — mise 2026.10.25 silently mis-enforces it and usage 7.1.5 rejects it; use the child-node form
- Put `run`, `run_windows`, `file`, or `usage` in a `#MISE` header — they are ignored with a warning (use `#USAGE` lines for args)
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
- Use tool options — inline (`"tool[opt=…]"`) or a tool table with any option — in shared configs expecting them to load untrusted; they require `mise trust`
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
- Assume `minimum_release_age` being "unset" means no delay — a **built-in 24h cutoff** applies on every backend; use `"0s"` to truly disable
- Declare the same flag in both a task template's `usage` and the extending task's — it gets listed **twice**
- Expect `depends = []` on a task to cancel an inherited `depends` — it still inherits the template's
- Assume `mise upgrade` frees disk immediately — replaced versions are kept for `upgrade.prune_after` (24h); use `--prune` to force
- Rely on the old HTTP dedup/symlink layout — own-directory extraction is the default since 2026.9.6; set `shared_extraction = true` to opt back in
- Use `mise bootstrap --from-git` — renamed to `--adopt`, **removed in 2026.10.0** (now an unknown-flag error)
- Assume `curl https://mise.run | sh` or `mise self-update` gets the newest release — both wait 24h unless `MISE_VERSION` is pinned
- Name multi-word `conf.d` fragments with dots (`node.tools.toml`) — they become environment selectors in 2027.8.10; use hyphens
- Expect `otel.enabled` alone to export traces — `experimental = true` and an `OTEL_EXPORTER_OTLP_*ENDPOINT` must also be set (and `otel.logs` turns task TTYs into pipes)
- Leave typos or other backends' options in tool tables — since 2026.10.14 unknown options **fail the install**
- Use `git::`/`oci::` task includes or `file = "git::…"` without `experimental = true` (gated since 2026.10.5)
- Put secrets in task `env`, `cache.env`/`global_env` values, `vars`, or `command_inputs` when a remote task cache is configured — the action description is uploaded verbatim (use `pass_through_env`)
- Set `PATH` in `[env]` — it's ignored; use `_.path`
- Name a structured env file `.env.yml` — only `.json`/`.yaml`/`.toml` are structured; `.yml` is parsed as dotenv
- Use `state = "absent"` on `apt:`/`brew:`/`choco:`/… packages — only pacman, scoop, snap, zypper (and dnf on rpm-ostree with `ostree = "layer"`) remove
- Put `encrypt = true` on symlink/copy/template dotfile entries — it applies to `mode = "track"` only
- Use the old `auto_update`/`auto_update_check_duration` settings — they're `self_update.auto`/`self_update.check_duration` now (the per-tool `auto_update` option is a different feature)
- Rely on a `.config/mise.toml`-relative `exec()`/`read_file()` resolving from the project root — they run in the config file's directory
- Assume a hook's `mise run`, a shim, `--help`, or an MCP call will trust a project for you (2026.10.19)
- Set `lockfile_mode = "merge"` — deprecated (warns from 2027.4.0, removed 2027.10.0); delete the line
- Set `system_packages.elevate_with`, `extra_ca_certs`, or `ruby.apply_patches` in a project `mise.toml` — they are global-only and ignored with a warning
- Expect a project task to receive a key from your **personal** fnox config (`~/.config/fnox/config.toml`, or a `fnox.toml` in/above `$HOME`) — it's refused; define it in the project's `fnox.toml` or pass `--secrets NAME`
- Use `--local` / `-l` on `ls`, `outdated`, `upgrade`, or `tasks ls` — deprecated in favor of `--no-global` (removed 2027.7.0)
- Re-run `mise tasks add` on an existing name without `--force` — replacing a different task that way is deprecated (refused from 2028.5.0)
- Confuse `mise tasks add -e/--env <ENV>` (writes `mise.<ENV>.toml`, loaded only with `MISE_ENV`) with `--env-var KEY=VALUE` (the task's `env`)
- Mix `env`/`vars` and `version` in one tool-option Tera expression (`{{ env.D ~ version }}`) — it renders at install time with `env`/`vars` empty; use separate `{{ }}` blocks
- Expect project `[env]` values inside a container daemon — only the daemon's own `env` table is passed in

### Complete Task Example

```toml
[tasks.deploy]
description = "Deploy application to environment"
alias = "d"
depends = ["build", "test"]
usage = '''
arg "<env>" help="Target environment" { choices "dev" "staging" "prod" }
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
skip_if = "./scripts/is-deployed.sh {{usage.env}}"   # 2026.10.10: exit 0 ⇒ skip (counts as success)
run = '''
#!/usr/bin/env bash
set -euo pipefail

if [ "${usage_dry_run:-false}" = "true" ]; then
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
#MISE timeout="10m"
#USAGE arg "<direction>" help="Migration direction" {
#USAGE   choices "up" "down"
#USAGE }
#USAGE flag "-n --count <n>" default="1" help="Number of migrations"
#USAGE flag "--dry-run" help="Preview SQL only"

set -euo pipefail

direction="${usage_direction?}"
count="${usage_count:-1}"

if [ "${usage_dry_run:-false}" = "true" ]; then
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
